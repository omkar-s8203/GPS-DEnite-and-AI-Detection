# ADR-004 — VIO Solution

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted for the low regime — conditional on gate G2b.** Amended by [ADR-015](ADR-015-visual-geolocalization.md): satellite image matching is now the primary GPS-denied source; this record governs odometry below about 12 m only, and its priority is Should |

## Context

The companion must produce metric 6-DoF odometry at ≥ 20 Hz on a Raspberry Pi 5 from a rolling-shutter, software-synchronised stereo camera and an I²C IMU, while leaving CPU for depth and a detector. The brief requires a primary, a backup and a simplified implementation, and warns against choosing the most complex option.

## Options

OpenVINS, VINS-Fusion, ORB-SLAM3, RTAB-Map (stereo odometry), custom OpenCV stereo VO. (Basalt, Kimera, SVO noted and not evaluated in depth.)

## Evaluation

Full analysis: [vio-slam-evaluation.md](../07-vio-slam/vio-slam-evaluation.md).

| Candidate | Pi 5 real-time with headroom | ROS 2 Jazzy | Handles missing cam–IMU trigger | Reports covariance | Complexity | Role |
|---|---|---|---|---|---|---|
| OpenVINS | Yes (filter) | First-party ROS 2; Jazzy fixes available | Online time-offset calibration | Yes | Medium | **Primary** |
| VINS-Fusion | Feasible (≈ 36 ms/frame reported on Pi 5), heavier | Community ports only | Online time-offset | Limited | Medium–high | Second alternative |
| ORB-SLAM3 | Doubtful with other loads | No maintained official wrapper | Sensitive | No | High | Rejected |
| RTAB-Map stereo odometry | Yes at reduced settings | Binary package | Not needed (no IMU coupling) | Yes | Low | **Backup** |
| OpenCV stereo VO | Yes | Own node | Not needed | No (heuristic) | Low | **Simplified** |

## Decision

| Role | Choice |
|---|---|
| **Primary** | **OpenVINS**, stereo + IMU, MSCKF |
| **Backup** | **RTAB-Map stereo odometry**, visual-only, with inertial fusion left to the FC's EKF3 |
| **Simplified** | **Custom OpenCV stereo visual odometry** (`gdn_vo_simple`) for learning, validation and bench fallback |

All three publish through the same adapter (`vio_monitor`), so switching is a launch argument.

## Reason

- OpenVINS is the lightest tightly coupled estimator with official ROS 2 support. Its online camera–IMU time-offset estimation targets this hardware's main weakness, and its covariance output feeds the confidence metric.
- The backup is chosen to fail *differently*: it does not depend on camera–IMU timing at all. If G2 shows that timing is the limiting factor, the backup sidesteps it.
- The simplified implementation gives the team an estimator they fully understand and an independent check of calibration and frames.
- ORB-SLAM3 is rejected despite its reputation: heaviest, least ROS 2-ready, most sensitive to this sensor's flaws, and its SLAM features are not needed ([ADR-005](ADR-005-slam-solution.md)).

## Consequences

- OpenVINS is GPL-3.0; used unmodified as a separate process. Any modification is published.
- OpenVINS assumes simultaneous stereo and global shutter. Pixel noise is inflated and speed/yaw-rate are limited to compensate.
- A pinned OpenVINS commit is recorded at first successful build.
- Gate G2 decides whether the primary stands, the backup takes over with a reduced envelope, or the camera is upgraded ([ADR-011](ADR-011-stereo-camera-suitability.md)).
- Comparing three estimators on identical recorded data becomes a deliverable of the project.
