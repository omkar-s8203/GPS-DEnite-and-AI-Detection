# Research Notes — Stereo Vision

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |
| Nature | Background research. Not a design document. |

## 1. Geometry

For a rectified pair with focal length f (pixels) and baseline B (metres):

| Quantity | Relation |
|---|---|
| Depth | Z = f·B / d |
| Depth resolution | ΔZ ≈ Z² / (f·B) · Δd |
| Minimum depth | Z_min = f·B / d_max |
| Lateral position | X = (u − c_x)·Z / f |

Consequences:

- Depth error grows with the **square** of range. Doubling the baseline or the focal length halves the error.
- A short baseline (60 mm) suits close range. For comparison, wider-baseline devices (75–120 mm) and higher resolution extend useful range proportionally.
- Sub-pixel disparity precision (typically 0.1–0.5 px depending on texture and algorithm) is the practical limit.

## 2. The Waveshare IMX219-83 as a stereo head [P]

From the vendor wiki: 2 × IMX219, 3280×2464, 2.6 mm focal length, FOV 83° D / 73° H / 50° V, distortion < 1 %, 60 mm baseline, ICM-20948 on board, CSI per sensor, **no hardware synchronisation**. Supported on Jetson Nano, Raspberry Pi (Bullseye/Bookworm), Compute Module and Raspberry Pi 5 (22-pin cable; `dtoverlay=imx219,cam0` and `cam1`).

The board originated as a Jetson Nano accessory, where the two sensors are also free-running. Its "depth vision" label refers to the possibility of computing depth, not to a synchronised depth product.

Indian listing prices seen: about ₹4,800–6,600 [S].

## 3. Matching algorithms

| Algorithm | Principle | Cost | Quality | Notes |
|---|---|---|---|---|
| Block matching (BM) | SAD over a window along the epipolar line | Low | Noisy; holes on weak texture | OpenCV `StereoBM`; NEON-optimised paths exist |
| Semi-global matching (SGM/SGBM) | Aggregates matching cost along several image paths with smoothness penalties (Hirschmüller, PAMI 2008) | Medium–high | Dense, smoother | OpenCV `StereoSGBM`; 3-way mode is faster |
| Learned stereo | CNN cost volumes | Very high | Best | Not real-time on a Pi CPU |
| Hardware stereo (in-camera) | FPGA/ASIC/VPU | None on host | Good | OAK-D, RealSense |

For obstacle sensing, sparse but trustworthy depth is preferable to dense but smoothed depth: smoothing can bridge gaps and invent surfaces.

## 4. Failure modes of passive stereo

| Condition | Effect |
|---|---|
| Texture-less surfaces | No match; holes or wrong depth |
| Repetitive patterns (fences, tiles) | Aliased matches |
| Specular surfaces, glass, water | Wrong or missing depth |
| Thin structures | Missed at low resolution |
| Occlusion boundaries | Fattening of foreground |
| Low light | Noise; long exposure → blur |
| Sun in view | Saturation, flare |
| Mis-calibration | Systematic depth bias; row misalignment reduces match rate |
| L/R timing error | Motion-dependent disparity error |

Active stereo (projected IR pattern) solves the texture problem indoors but adds power and fails in sunlight.

## 5. Synchronisation

### 5.1 Why it matters

A time offset Δt between exposures with image motion of ω rad/s produces a disparity error of about f·ω·Δt pixels. At f = 432 px (640-wide image) and ω = 1 rad/s: 0.43 px per millisecond. At 3 m range the disparity is only 8.6 px, so 5 ms of offset gives a 25 % depth error.

### 5.2 Mechanisms

| Mechanism | Accuracy | Availability here |
|---|---|---|
| Shared external trigger (sensor in slave/trigger mode) | Microseconds | Not provided by this board |
| Sensors sharing one clock and frame-sync line | Microseconds | Not provided |
| Stereo bridge chip presenting one double-wide image | Line-accurate | Not this product |
| **Software frame-timing servo (libcamera on Raspberry Pi)** | To be measured | Available |
| Timestamp pairing only | Up to half a frame period | Fallback |

### 5.3 Raspberry Pi software camera synchronisation [P]

From Raspberry Pi's documentation: the libcamera implementation can synchronise the frames of different cameras in software. One camera acts as **server** and broadcasts timing messages; **client** cameras lengthen or shorten frame times slightly to pull into alignment. It is stated to work with all Raspberry Pi camera modules and with third-party sensors whose drivers implement frame-duration control correctly; clients may be on the same Pi or on other Pis on the network; all cameras must run at the same fixed frame rate. Exposed in `rpicam-apps` through `--sync server` / `--sync client`.

The IMX219 driver is the one used by Camera Module v2, so the feature should apply. The residual error for two IMX219 sensors on one Pi 5 is not stated and is the key unknown (R-1).

### 5.4 Hardware-synchronised alternatives [P][S]

- Raspberry Pi Global Shutter Camera (IMX296): supports an external trigger input, allowing true hardware synchronisation of two units.
- Luxonis OAK-D Lite: OV7251 global-shutter mono stereo pair, 75 mm baseline, BMI270 IMU listed in vendor documentation, 61 g, 91 × 28 × 17.5 mm; depth computed on-device.
- Intel RealSense D435i: global-shutter IR stereo with IMU; USB 3.

## 6. Rolling shutter in stereo

If both sensors are identical and start together, corresponding rows are exposed at the same instant, so disparity (a same-row quantity) is nearly unaffected even though each image is skewed. With a start offset, rows no longer correspond in time. Hence sync matters *more* with rolling shutter, and good sync recovers most of the depth quality.

## 7. Calibration

| Item | Notes |
|---|---|
| Model | Pinhole + radial-tangential is adequate for 73° HFOV with < 1 % distortion |
| Target | AprilGrid preferred (partial views usable, unambiguous orientation) |
| Accuracy drivers | Target flatness and measured size; coverage of image corners; sharp images; no motion blur; fixed focus |
| Quality indicators | Reprojection RMS (< 0.5 px), epipolar error, recovered baseline against the mechanical value |
| Stability | Board-mounted lenses are reasonably stable; plastic lens holders shift with temperature and shock |

## 8. Pi 5 camera stack notes [P][S]

- The Pi 5 processes raw Bayer through its ISP (PiSP); the pipeline handler lives in Raspberry Pi's libcamera fork.
- On Ubuntu 24.04, community guides report that the distribution's libcamera lacks the Raspberry Pi pipeline handlers, giving "no cameras available"; the remedy is to build the Raspberry Pi fork and build the ROS camera node against it. Guides exist specifically for IMX219 on Pi 5 with Ubuntu 24.04 and Jazzy.
- Pi 5 CSI connectors are 22-pin 0.5 mm pitch; older cameras use 15-pin 1 mm.
- The Pi 5 has no hardware H.264/H.265 encoder: video encoding is in software.

## 9. Takeaways used in the design

| Finding | Used in |
|---|---|
| Vendor confirms no hardware sync | ADR-011 |
| Software sync exists in the Pi libcamera stack | ADR-007 |
| 0.43 px/ms disparity error at 1 rad/s | NFR-002 (≤ 1 ms) |
| Depth error ∝ Z² with f·B = 25.9 at 640 px | Range limit 0.5–6 m; speed limit |
| BM first, SGBM as option | stereo-vision-pipeline §4 |
| Unknown ≠ clear | obstacle design |
| libcamera fork needed on Ubuntu | ADR-001, ADR-007 |
| No hardware encoder on Pi 5 | MK15 topology A |
