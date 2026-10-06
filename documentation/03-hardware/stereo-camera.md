# Stereo Camera — Waveshare IMX219-83

| Field | Value |
|---|---|
| Document ID | GDN-HW-004 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Verdict | **Usable for stereo depth and AI. Marginal for visual-inertial odometry.** Kept as the baseline sensor behind gate G2; see §8 and [ADR-011](../17-decisions/ADR-011-stereo-camera-suitability.md). |

## 1. Verified specification

Source: Waveshare product wiki for the IMX219-83 Stereo Camera (see [references](../references.md)).

| Parameter | Value |
|---|---|
| Image sensors | 2 × Sony IMX219 |
| Resolution (each) | 3280 × 2464 (8 MP) |
| Optical format | 1/4 inch |
| Focal length | 2.6 mm |
| Field of view | 83° diagonal / 73° horizontal / 50° vertical |
| Lens distortion | < 1 % |
| Stereo baseline | 60 mm |
| Interface | 2 × MIPI CSI-2 (one per sensor) |
| Onboard IMU | ICM-20948, 9-axis: 16-bit accelerometer (±2/4/8/16 g), 16-bit gyroscope (±250/500/1000/2000 °/s), magnetometer (±4900 µT); 3.3 V logic; I²C |
| Board size | 24 mm × 85 mm |
| Platform support | Jetson Nano; Raspberry Pi (Bullseye/Bookworm); Compute Module; Raspberry Pi 5 with 22-pin cable |
| Pi 5 configuration | `dtoverlay=imx219,cam0` and `dtoverlay=imx219,cam1` in `/boot/firmware/config.txt` |
| **Synchronisation** | **Vendor statement: "The IMX219-83 doesn't feature hardware synchronization."** |

Not stated by the vendor, known from the sensor: the IMX219 is a **rolling-shutter** colour (Bayer) sensor with fixed-focus lens. Pixel pitch is 1.12 µm `[VERIFY from Sony datasheet]`.

Not available from the vendor and to be measured: mass, current draw, IMU I²C address and interrupt pin availability, exact supported frame rates per mode under the Pi 5 driver.

## 2. Operating mode selected

| Setting | Value | Reason |
|---|---|---|
| Sensor mode | 1640 × 1232, 2×2 binned, full field of view | Full FOV; four times the light per pixel; shorter readout than full resolution |
| Output to ROS | 640 × 480, 8-bit monochrome (luma), downscaled by the ISP | VIO and block matching work on grey images; small images keep CPU load down |
| Frame rate | 20 Hz, fixed frame duration | Sync requires a fixed rate; 20 Hz matches the standard VIO benchmark rate and leaves CPU headroom |
| Exposure | Manual, ≤ 4 ms outdoors (≤ 8 ms indoors), fixed per flight; analogue gain adjusted by a slow controller | Avoid motion blur and exposure "pumping" that breaks feature tracking |
| White balance / colour | Fixed; a colour stream is produced only for the detector if needed | Deterministic images |
| Detector input | Left image, colour, 320 × 240 from the same ISP (second stream) or mono replicated to 3 channels | See [ai-architecture](../08-ai/ai-architecture.md) |

The mode table of the `imx219` driver (frame rates per mode) is confirmed at bring-up `[MEASURE]`.

## 3. Derived optical quantities

Focal length in pixels from the horizontal FOV: `f = (W/2) / tan(HFOV/2)`.

| Image width | f (px) | f·B (px·m), B = 0.06 m |
|---|---|---|
| 3280 (full) | ≈ 2216 | 133 |
| 1640 (binned) | ≈ 1108 | 66.5 |
| 1280 | ≈ 865 | 51.9 |
| **640 (baseline)** | **≈ 432** | **25.9** |

(From 2.6 mm / 1.12 µm the full-resolution value is ≈ 2320 px. The two figures differ by 5 %; calibration decides.)

## 4. Depth capability

Depth: `Z = f·B / d`. Depth uncertainty: `ΔZ ≈ Z² / (f·B) · Δd`, where Δd is the disparity error.

At 640 px width, f·B = 25.9:

| Range Z | Disparity d | ΔZ at Δd = 0.25 px | ΔZ at Δd = 0.5 px |
|---|---|---|---|
| 0.5 m | 51.8 px | 0.002 m | 0.005 m |
| 1 m | 25.9 px | 0.010 m | 0.019 m |
| 2 m | 13.0 px | 0.039 m (1.9 %) | 0.077 m (3.9 %) |
| 3 m | 8.6 px | 0.087 m (2.9 %) | 0.17 m (5.8 %) |
| 5 m | 5.2 px | 0.24 m (4.8 %) | 0.48 m (9.7 %) |
| 6 m | 4.3 px | 0.35 m (5.8 %) | 0.70 m (12 %) |
| 10 m | 2.6 px | 0.97 m (9.7 %) | 1.9 m (19 %) |

All values are `[ESTIMATE]` from geometry for a static, well-textured scene.

| Limit | Value | Cause |
|---|---|---|
| Minimum range | ≈ 0.40 m with 64 disparities; ≈ 0.27 m with 96 | Disparity search range |
| Useful maximum range | ≈ 5–6 m | Error grows with Z² |
| Horizontal coverage | ± 36.5° about the optical axis | Lens FOV |
| Fails on | Texture-less surfaces, repetitive patterns, glare, thin wires, low light | Passive stereo |

**Consequence for flight:** obstacle sensing is reliable only to about 5 m, forward only. With 0.3 s total latency and 2 m/s² braking, stopping from speed v needs `0.3·v + v²/4` metres. Keeping a 1.5 m margin inside a 5 m detection range gives v ≈ 3.2 m/s as the theoretical ceiling; the design limit is 2.0 m/s and initial tests use ≤ 1.0 m/s.

## 5. Synchronisation analysis

The two sensors free-run. Three effects matter.

### 5.1 Left/right frame offset

If the two exposures start Δt apart and the image moves at angular rate ω, a point shifts by `f·ω·Δt` pixels between the two images. That shift adds directly to (or subtracts from) disparity.

At 640 px (f = 432 px):

| ω | Δt = 0.5 ms | Δt = 1 ms | Δt = 5 ms | Δt = 25 ms (worst case, unsynchronised at 20 Hz) |
|---|---|---|---|---|
| 0.25 rad/s (14 °/s) | 0.05 px | 0.11 px | 0.54 px | 2.7 px |
| 0.5 rad/s (29 °/s) | 0.11 px | 0.22 px | 1.1 px | 5.4 px |
| 1.0 rad/s (57 °/s) | 0.22 px | 0.43 px | 2.2 px | 10.8 px |

At 3 m the true disparity is only 8.6 px. An uncontrolled offset makes depth and stereo-VIO scale meaningless during rotation. **The offset must be held below about 1 ms** (NFR-002).

Mitigation: the Raspberry Pi libcamera stack provides **software camera synchronisation**: one camera acts as server, the other as client and stretches or shrinks its frame duration until frame starts coincide. Raspberry Pi documents it as working with all Raspberry Pi camera modules and with third-party sensors whose drivers implement frame-duration control; it requires a fixed, equal frame rate. The achievable residual offset with two IMX219 sensors on one Pi 5 is **not documented and must be measured** `[MEASURE]`. The driver publishes the per-pair offset so that every downstream consumer can reject bad pairs.

### 5.2 Rolling shutter

Rows are exposed sequentially over the readout time t_r (tens of milliseconds for a full frame; exact value for the chosen mode `[MEASURE]`). Under rotation, the top and bottom rows see the scene at different orientations: shift ≈ `f·ω·t_r`. With t_r ≈ 20 ms and ω = 0.5 rad/s that is ≈ 4 px of skew across the image.

- For stereo depth: if both sensors are synchronised and identical, both images are skewed alike and row correspondence survives. Depth degrades gracefully.
- For VIO: features are assumed to be observed at one instant. Unmodelled rolling shutter adds a motion-dependent bias. This is the main reason the camera is "marginal" for VIO. Mitigations: limit yaw rate to ≤ 45 °/s, keep exposure short, fly slowly, and use an estimator that tolerates or models readout time if available.
- Vibration at motor frequency produces "jello" that no algorithm fixes. Mechanical isolation is mandatory.

### 5.3 Camera–IMU timing

The ICM-20948 is read over I²C by a user-space driver; there is no hardware trigger between IMU and cameras. Both are stamped on the same monotonic clock, so the offset is roughly constant and OpenVINS estimates it online (`calib_cam_timeoffset`). Jitter of I²C polling (±1–2 ms `[ESTIMATE]`) remains as noise. Using the data-ready interrupt, if the board exposes it, reduces this.

## 6. Driver support and Raspberry Pi compatibility

| Layer | Status |
|---|---|
| Kernel sensor driver | `imx219` is in the mainline and Raspberry Pi kernels; enabled by device-tree overlay per port |
| Pi 5 dual-camera | Supported: one sensor per CSI port |
| Raspberry Pi OS | Works with the stock `rpicam-apps` / libcamera |
| Ubuntu 24.04 | Kernel driver present; **user-space libcamera must be the Raspberry Pi fork built from source** for the Pi 5 pipeline handler `[VERIFY at bring-up; later point releases may package it]` |
| ICM-20948 | Kernel IIO driver (`inv_icm20948`) availability on the Ubuntu Pi kernel `[VERIFY]`; baseline is a user-space I²C driver node, which works regardless |

## 7. ROS 2 integration options

| Option | Description | Assessment |
|---|---|---|
| A. Two `camera_ros` instances | Community libcamera node, one per camera | Simple, but two processes: no control over start alignment, no shared sync logic, pairs matched only by approximate time |
| B. `v4l2_camera` | Generic V4L2 node | Does not drive the Pi 5 ISP pipeline for raw Bayer sensors; rejected |
| C. GStreamer `libcamerasrc` → ROS bridge | Pipelines per camera | Workable for viewing; timestamp fidelity is harder to guarantee |
| **D. Custom single-process stereo node (`gdn_camera`)** | One C++ node opens both cameras through libcamera, applies software sync, pairs frames by sensor timestamp, publishes left/right with **identical header stamps** plus the measured skew | **Selected.** It is the only option that gives explicit control of synchronisation and timestamps, which the analysis in §5 shows is the critical property. |

Option A is the bring-up shortcut (first images within a day) before option D exists.

## 8. Calibration requirements

| Calibration | Tool | Output | When |
|---|---|---|---|
| Intrinsics, each camera (pinhole + radial-tangential) | Kalibr or ROS `camera_calibration` | fx, fy, cx, cy, k1, k2, p1, p2 | After mounting; repeat after any knock |
| Stereo extrinsics | Kalibr (multi-camera) | T_right_left; expect baseline ≈ 60 mm | Same session |
| IMU noise model | Allan variance from ≥ 3 h static log | Gyro/accel noise density and random walk | Once |
| Camera–IMU extrinsics and time offset | Kalibr (camera-IMU), with rolling-shutter model if supported | T_cam_imu, t_d | After intrinsics |
| Readout time | From sensor mode timing or Kalibr rolling-shutter calibration | t_r | Once per mode |
| Verification | Reprojection error; known-distance depth test; handheld loop closure error | Pass/fail against gate G2 | Before flight |

Full procedure: [calibration.md](../06-computer-vision/calibration.md).

## 9. Suitability gate G2 and alternatives

### 9.1 Gate G2 (end of the stereo and VIO bench phases)

| Criterion | Pass threshold |
|---|---|
| L/R timestamp skew | ≤ 1 ms for ≥ 99 % of pairs over 10 min |
| Stereo calibration reprojection error | ≤ 0.5 px RMS |
| Depth error, static target | ≤ 5 % at 2 m; ≤ 10 % at 5 m |
| VIO handheld loop (≈ 30 m, walking pace, textured outdoor) | Final position error ≤ 2 % of path length, in 4 of 5 runs |
| VIO on a vibrating frame (motors running, props off or tethered) | No divergence in 3 min; drift while stationary ≤ 0.3 m |

If G2 passes: continue with this camera. If it fails on skew or VIO criteria: adopt the backup estimator (loosely coupled stereo odometry, which is less sensitive to IMU timing) and re-test; if it still fails, procure an upgrade.

### 9.2 Upgrade candidates

| Candidate | Shutter / sync | IMU | Interface | Notes | Assessment |
|---|---|---|---|---|---|
| Luxonis OAK-D Lite | Global-shutter mono stereo pair (OV7251), synchronised; 75 mm baseline `[VENDOR]` | BMI270 `[VENDOR]`; presence on a given unit `[VERIFY]` | USB-C | 61 g `[VENDOR]`; on-device stereo depth and neural inference; ROS 2 driver (`depthai-ros`); community reports of use on Pi 5 with Ubuntu 24.04 + Jazzy | **Recommended upgrade.** Fixes sync and shutter and offloads depth. USB noise must be managed. Price in India `[VERIFY]`. |
| Intel RealSense D435i | Global-shutter IR stereo, synchronised | BMI055 | USB 3 | Widely used with ArduPilot and VIO; heavier; costlier | Good alternative if available through the institute |
| 2 × Raspberry Pi Global Shutter Camera (IMX296) with external trigger | Global shutter; hardware trigger supported by the sensor board | None (use the existing ICM-20948 board or FC IMU) | 2 × CSI | Needs lenses, a rigid custom stereo bar and trigger wiring | Lowest-cost true-sync option; most mechanical work |
| Arducam synchronised global-shutter stereo kits | Global shutter, hardware-synchronised | Varies | CSI via bridge | Driver support on Pi 5 + Ubuntu is the risk | Only if driver support is confirmed first |

The software architecture is unchanged by an upgrade: the replacement only has to publish the same topics ([ADR-007](../17-decisions/ADR-007-camera-interface.md)).

## 10. Summary of limitations to carry into the project report

1. No hardware stereo synchronisation (vendor-stated); software sync residual unknown until measured.
2. Rolling shutter; motion- and vibration-dependent distortion.
3. IMU not hardware-synchronised to the cameras; consumer-grade MEMS on I²C.
4. 60 mm baseline and wide lens: useful depth to roughly 5–6 m.
5. Passive: needs light and texture. No night operation.
6. Fixed focus, colour Bayer sensor: lower sensitivity than a mono global-shutter sensor of the same size.
