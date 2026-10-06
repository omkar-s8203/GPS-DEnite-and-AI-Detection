# Research Notes — VIO and SLAM

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |
| Nature | Background research. Not a design document. |

## 1. Terminology

| Term | Meaning |
|---|---|
| VO | Visual odometry: incremental pose from images only |
| VIO | Visual-inertial odometry: VO fused with an IMU |
| SLAM | Simultaneous localisation and mapping: odometry plus a persistent map and loop closure |
| Loosely coupled | Vision produces a pose; a separate filter fuses pose with IMU |
| Tightly coupled | Raw feature measurements and IMU are fused in one estimator |
| Filter-based | EKF variants (MSCKF, ROVIO) |
| Optimisation-based | Sliding-window or full bundle adjustment (VINS, OKVIS, ORB-SLAM3, Basalt) |
| Direct / indirect | Uses pixel intensities / uses extracted features |

## 2. Estimator families

| Family | Examples | Cost | Accuracy | Notes |
|---|---|---|---|---|
| MSCKF (filter, sliding window of poses) | OpenVINS, MSCKF-VIO, S-MSCKF | Low | Good | Features marginalised immediately; cost independent of map size |
| EKF with features in state | ROVIO | Low–medium | Good | Direct photometric updates |
| Sliding-window optimisation | VINS-Mono/Fusion, OKVIS, Basalt | Medium–high | Very good | Relinearisation improves accuracy; pre-integration of IMU |
| Full SLAM with map reuse | ORB-SLAM3, Kimera, RTAB-Map | High | Best on revisits | Loop closure, relocalisation, multi-session |
| Semi-direct | SVO | Very low | Good at high frame rates | Sensitive to photometric conditions |

## 3. Notes on the evaluated systems

### OpenVINS [P][L]

- Research platform from the University of Delaware (Geneva et al., ICRA 2020). MSCKF core with optional SLAM features, first-estimate Jacobians for consistency.
- Online calibration of camera intrinsics, camera–IMU extrinsics and camera–IMU time offset; inertial intrinsics added in release 2.7 together with improved stereo KLT tracking.
- Monocular, stereo and multi-camera configurations; expects synchronised cameras.
- ROS 1 and ROS 2 support in the main repository; a contribution adds ROS 2 Jazzy / Ubuntu 24.04 support (addresses a Ceres-related crash and header changes); community forks for Jazzy exist.
- Mostly single-threaded estimator: per-core speed matters more than core count.
- Community guidance on cameras: low resolution (≈ 640×480), fixed 20–30 Hz, monochrome, short exposure, **global shutter**; reports that rolling-shutter Pi cameras with long exposure perform poorly [S].
- Licence GPL-3.0.

### VINS-Fusion [L][S]

- HKUST (Qin et al.). Optimisation-based; mono+IMU, stereo, stereo+IMU; optional GNSS global fusion; online temporal calibration.
- Official code targets ROS 1; several community ROS 2 ports including Jazzy forks.
- A 2025 paper comparing odometry methods reports VINS-Fusion at about 36 ms per frame on a Raspberry Pi 5 on EuRoC, i.e. near real time at 20 Hz [S].
- VINS-Mono includes a rolling-shutter option (readout-time parameter); applicability to the stereo pipeline to be checked [U].

### ORB-SLAM3 [L]

- University of Zaragoza (Campos et al., IEEE T-RO 2021). Mono/stereo/RGB-D, with or without IMU; multi-map; loop closing.
- Three parallel threads plus loop closing; heavy for a Pi sharing CPU with other tasks.
- No official ROS 2 wrapper; community wrappers of varying maintenance.
- Inertial initialisation needs sufficient motion and good timing.
- Licence GPL-3.0.

### RTAB-Map [P][L]

- Université de Sherbrooke (Labbé & Michaud, Journal of Field Robotics 2019). Graph-based SLAM with memory management; separate odometry nodes (stereo, RGB-D, ICP).
- ROS 2 Jazzy binaries available (`ros-jazzy-rtabmap-ros`; version 0.23.x seen).
- Visual odometry is not tightly inertial; IMU can supply gravity/orientation.
- A study using a Jetson Nano reports about 7 Hz visual odometry against a recommended 15 Hz for 3D mapping [S]: indicates that settings must be reduced on small computers.
- Licence BSD.

### Stereo VO with OpenCV [L]

- Classic pipeline (Scaramuzza & Fraundorfer tutorial, IEEE RAM 2011): detect → stereo match → triangulate → temporal track → PnP + RANSAC.
- Drift of a few percent; fails on blur and low texture; no gravity reference.

## 4. Sensor requirements for VIO [L][S]

| Requirement | Reason | This project's camera |
|---|---|---|
| Global shutter | One pose per image | **No** (rolling) |
| Hardware-synchronised stereo | Simultaneous views | **No** (software sync) |
| Hardware-triggered or timestamped IMU | Known camera–IMU offset | **No** (same clock; offset estimated) |
| IMU ≥ 200 Hz, low noise | Pre-integration quality | 225 Hz, consumer grade |
| Short exposure | Low blur | Achievable in daylight |
| Wide FOV | Features stay in view during rotation | 73° H: moderate |
| Rigid camera–IMU mount | Constant extrinsics | **Yes** (same PCB) |
| Stable calibration | — | To be established |

Three of the first four properties are missing. That is the technical core of ADR-011.

## 5. Rolling shutter [L]

- Each row is exposed at a different time; over a readout time of tens of milliseconds, motion skews the image.
- Effects: geometric bias in feature positions proportional to angular rate × readout time × focal length; "jello" from vibration.
- Handling in estimators: (a) ignore and inflate measurement noise; (b) model per-row pose by interpolating between states (some VIO systems and Kalibr support this); (c) correct features with gyro before use.
- Phone-based VIO works with rolling shutter because readout time is calibrated and modelled, and camera and IMU are hardware-timestamped.

## 6. Benchmarks and what they imply [L][S]

| Source | Finding | Implication |
|---|---|---|
| EuRoC MAV dataset (Burri et al., IJRR 2016) | Standard stereo + IMU dataset, 20 Hz, global shutter, hardware sync | The baseline on which published accuracies are obtained; our sensor is worse in every respect |
| Delmerico & Scaramuzza, ICRA 2018 | Benchmark of VIO on computers from embedded to laptop: accuracy/CPU trade-offs differ strongly between algorithms | Filter-based methods are attractive on small CPUs |
| "Run Your Visual-Inertial Odometry on NVIDIA Jetson" (Jeon et al., 2021) | VIO benchmarks on Jetson boards on a MAV | Embedded feasibility; resource usage tables |
| 2025 comparison including Raspberry Pi 5 | VINS-Fusion ≈ 36 ms/frame on a Pi 5 | A Pi 5 can run optimisation-based stereo VIO near real time when nothing else is running |

Typical published drift for good stereo-inertial systems on EuRoC-class data is well under 1 % of path length. Expect several times worse with this hardware.

## 7. Calibration tools [L]

- **Kalibr** (Furgale, Rehder, Siegwart, IROS 2013 and later): multi-camera, camera–IMU spatial and temporal calibration; rolling-shutter camera calibration; AprilGrid targets. De-facto standard; outputs are directly convertible to OpenVINS and VINS configurations.
- **Allan variance** tools for IMU noise identification.
- OpenVINS documentation includes a calibration guide recommending inflated IMU noise values relative to Allan-variance results.

## 8. SLAM vs odometry for feeding an autopilot

Observations from ArduPilot/PX4 practice [P][L]:

- Autopilot EKFs expect continuous position measurements. Tracking cameras that relocalise report "jumps", and the MAVLink messages carry a reset counter for that reason.
- For control, smoothness matters more than global accuracy. Global corrections belong in a slowly varying `map → odom` transform (REP-105), not in the measurement stream.

## 9. Takeaways used in the design

| Finding | Used in |
|---|---|
| Filter-based VIO is the best fit for a shared Pi 5 | ADR-004 |
| OpenVINS has first-party ROS 2 and Jazzy fixes | ADR-004 |
| ORB-SLAM3 is heavy and lacks ROS 2 support | ADR-004 |
| RTAB-Map odometry is a binary install and IMU-timing independent | ADR-004 (backup) |
| The camera lacks three core VIO properties | ADR-011, gate G2 |
| Inflate noise and limit motion for rolling shutter | vio-design §4–5 |
| Corrections belong in `map → odom` | coordinate-frames §8, ADR-005 |
| EuRoC on the Pi separates compute limits from sensor limits | vio-slam-evaluation §7 |
