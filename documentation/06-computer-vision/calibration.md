# Calibration Plan

| Field | Value |
|---|---|
| Document ID | GDN-CV-002 |
| Version | 1.0 |
| Date | 2026-10-05 |

Calibration quality bounds VIO and depth quality. With a rolling-shutter, software-synchronised camera it matters more than usual. This plan is a required deliverable, not an optional refinement.

## 1. What is calibrated

| # | Calibration | Parameters | Tool | Consumer |
|---|---|---|---|---|
| C1 | IMU noise | Gyro and accel noise density, bias random walk | Allan-variance tool on a static log | Kalibr, OpenVINS |
| C2 | Camera intrinsics ×2 | fx, fy, cx, cy; radial-tangential k1, k2, p1, p2 | Kalibr (multi-camera) | OpenVINS, rectification |
| C3 | Stereo extrinsics | T_right_left (rotation, translation ≈ 60 mm) | Kalibr (same run as C2) | OpenVINS, rectification, depth |
| C4 | Camera–IMU | T_cam_imu, time offset t_d | Kalibr (camera-IMU) | OpenVINS |
| C5 | Rolling-shutter readout time | t_r per mode | Sensor timing from the driver; cross-check with Kalibr's rolling-shutter calibration if used | VIO settings, analysis |
| C6 | Vehicle mounting | T_base_stereo | Ruler/CAD + hover check | URDF |
| C7 | FC sensors | Accelerometer, compass, RC, ESC, battery monitor | Mission Planner / QGroundControl | ArduPilot |
| C8 | External-nav delay | `VISO_DELAY_MS` | FC log analysis | ArduPilot |
| C9 | Compass motor interference | `COMPASS_MOT` | ArduPilot procedure | ArduPilot |

## 2. Equipment

| Item | Specification |
|---|---|
| Target | AprilGrid (6×6, tag size ≈ 55–88 mm) printed and bonded flat to a rigid board (foam board or aluminium composite). Flatness within 1 mm. Measure the printed tag size with a caliper; do not trust the print scale. |
| Lighting | Bright, diffuse; allows ≤ 2 ms exposure indoors |
| Workstation | Ubuntu PC with Kalibr in Docker |
| Recording | `vio_dataset` bag profile on the Pi (raw images at 20 Hz for C4; 4 Hz is sufficient for C2/C3) |

## 3. Procedure

### C1 — IMU noise (once)

1. Mount the camera board on a heavy, vibration-free surface. Let it reach thermal equilibrium (20 min).
2. Record `/imu/data_raw` for ≥ 3 hours.
3. Compute Allan deviation; read noise density (slope −½ at τ = 1 s) and bias random walk (slope +½).
4. Inflate the results by ×5–×10 for use in Kalibr and OpenVINS. Datasheet-level values understate noise on a vibrating vehicle.

### C2 + C3 — Intrinsics and stereo extrinsics

1. Fix exposure short enough to avoid blur.
2. Move the **target** slowly in front of the fixed camera, or the camera in front of the fixed target, covering every image region including corners, at distances 0.3–1.5 m, with tilts up to ± 40°.
3. Record 60–90 s. Keep motion slow: rolling shutter and residual L/R skew contaminate fast sequences.
4. Run Kalibr multi-camera calibration with the pinhole-radtan model.
5. Accept if: reprojection RMS ≤ 0.5 px per camera; baseline 60 ± 1.5 mm; relative rotation < 1°; principal points within ± 5 % of the image centre.
6. Generate `camera_info` YAMLs (including rectification matrices) and the OpenVINS camera chain.

### C4 — Camera–IMU

1. Target fixed, well lit. Hold the vehicle/camera assembly by hand.
2. Excite all six axes: translations along x, y, z and rotations about x, y, z, smoothly, for 60–90 s, keeping the target in view. Begin and end with 2 s stationary.
3. Record raw images at 20 Hz and IMU at full rate.
4. Run Kalibr camera-IMU calibration with the C1 noise values and the C2/C3 camera chain. Enable time-offset estimation.
5. Accept if: reprojection RMS ≤ 1.0 px; gyro and accel residuals within their 3σ bounds; estimated translation camera↔IMU plausible against the PCB layout (a few centimetres at most); t_d stable (± 2 ms) across three independent runs.
6. Repeat three times. Use the median result. Large run-to-run variation in t_d indicates timestamp jitter that must be fixed in the driver before continuing.

### C5 — Readout time

Compute from the sensor mode: line time × active rows (from the driver's reported line length and pixel rate). Record the value. If Kalibr's rolling-shutter camera calibration is used, compare.

### C6 — Mounting

Measure the stereo midpoint relative to the FC IMU centre (x forward, y left, z up) with a ruler; record the camera pitch with an inclinometer. Enter in the URDF. Validate with bench check CF-6 and CF-8 in [coordinate-frames.md](../02-system-architecture/coordinate-frames.md).

### C8 — External-nav delay

With VIO streaming to the FC and the vehicle moved by hand (sharp, distinct motions), compare the FC's IMU-derived motion with the `VISP` log entries; set `VISO_DELAY_MS` to the measured lag. Expected 50–100 ms `[ESTIMATE]`.

## 4. Storage and identification

```text
gdn_description/calibration/<calibration_id>/
    README.md                 # date, operator, target, conditions, results summary
    imu_noise.yaml            # C1
    kalibr_camchain.yaml      # C2 + C3
    kalibr_camchain_imu.yaml  # C4
    left.yaml, right.yaml     # camera_info with rectification
    ov_kalibr_imu_chain.yaml  # OpenVINS format
    ov_kalibr_imucam_chain.yaml
    report/                   # Kalibr PDF reports
```

`calibration_id` = `<camera_serial>_<YYYYMMDD>_<n>`. The active ID is a launch parameter and is written into every bag's metadata.

## 5. Validity and re-calibration triggers

| Trigger | Action |
|---|---|
| Any crash or hard landing | Verify (row alignment + depth at 2 m); recalibrate C2–C4 if out of tolerance |
| Camera or mount removed/refitted | C6; verify C2–C4 |
| Change of sensor mode or resolution | C2–C5 |
| Depth error or VIO drift worse than the last accepted baseline | Full recalibration |
| Every month during the test campaign | Verification only |

## 6. Quick verification (5 minutes, before each test day)

| Check | Pass |
|---|---|
| Rectified checkerboard corners: row difference L/R | ≤ 0.5 px mean |
| Depth of a flat target at a taped 2.00 m | Within 5 % |
| Static VIO for 60 s | Drift ≤ 5 cm |
| Rotate by hand 90° and back | Returns to start within 5° and 10 cm |

## 7. Risks specific to this camera

| Risk | Effect | Mitigation |
|---|---|---|
| Rolling shutter not modelled in the standard Kalibr camera-IMU run | Biased t_d and extrinsics | Slow, smooth calibration motion; cross-check t_d stability; consider Kalibr's rolling-shutter mode |
| L/R skew during calibration | Biased stereo extrinsics | Slow motion; reject pairs above the skew threshold |
| Fixed-focus lens with close target | Blur at < 0.3 m | Keep target ≥ 0.3 m |
| PCB flex between lenses | Extrinsics change with mounting stress | Mount the board at its designed holes without bending; recalibrate after final mounting |
| Temperature | Focal length drift | Calibrate at a temperature similar to operating conditions; note temperature in the README |
