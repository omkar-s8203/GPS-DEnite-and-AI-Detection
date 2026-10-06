# Research Notes — State Estimation

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |
| Nature | Background research. Not a design document. |

## 1. The estimation problem on a multirotor

A multirotor is unstable and needs attitude at hundreds of hertz and position/velocity at tens of hertz. IMU integration supplies the high rate but drifts; every other sensor exists to correct that drift.

| Quantity | Corrected by |
|---|---|
| Roll, pitch | Gravity (accelerometer) through the filter |
| Yaw | Magnetometer, GNSS heading/motion, or vision |
| Vertical position | Barometer, range sensor, GNSS |
| Horizontal velocity | GNSS, optical flow, vision |
| Horizontal position | GNSS, vision, beacons |
| IMU biases | Observed indirectly through all of the above |

Without a horizontal velocity or position source, position error grows within seconds. That is what "GPS-denied" means in practice.

## 2. Extended Kalman filtering in autopilots

| Aspect | Notes |
|---|---|
| Structure | IMU drives prediction; other sensors update; states include biases |
| Delayed measurements | Measurements arrive late (GNSS ≈ 100–200 ms, vision ≈ 50–150 ms). Filters buffer IMU/state history and fuse at the measurement's time horizon; an output predictor brings the state to the present. A wrong delay parameter appears as lag or oscillation. |
| Innovation gating | A measurement far from prediction (in units of expected standard deviation) is rejected. Protects against outliers; can also reject a correct source after the filter has drifted. |
| Health reporting | Normalised innovation "variances" per source; flags for which aiding is active |
| Multiple cores / lanes | Parallel filters on different IMUs with lane switching |

ArduPilot EKF3: 24 states (attitude quaternion, velocity NED, position NED, gyro bias, accel bias, earth field, body field, wind NE). Source selection per axis group through `EK3_SRCn_*`. External-navigation noise floors through `VISO_*_M_NSE`; delay through `VISO_DELAY_MS`.

## 3. Loose vs tight coupling

| | Loose | Tight |
|---|---|---|
| Interface | Pose/velocity from a separate estimator | Raw measurements (pixels, pseudoranges) |
| Optimality | Sub-optimal: correlated errors treated as white | Better use of information |
| Modularity | High | Low |
| Failure isolation | Good: the upstream estimator can fail without corrupting the downstream IMU states | A bad measurement model affects everything |
| Typical use | Vision pose into an autopilot EKF | Inside the VIO itself |

This project is tight inside the VIO and loose between VIO and EKF3. The two stages use different IMUs, so inertial information is not counted twice. The remaining theoretical weakness is that VIO drift is not white noise; it is accepted.

## 4. Visual-inertial filtering essentials [L]

| Concept | Summary | Reference |
|---|---|---|
| MSCKF | Keep a window of past camera/IMU poses; use each feature track to form a constraint among them, then discard the feature | Mourikis & Roumeliotis, ICRA 2007 |
| IMU pre-integration | Summarise IMU between keyframes for optimisation | Forster et al., IEEE T-RO 2017 |
| Observability | Global position and yaw are unobservable in VIO; estimators must not gain spurious information in those directions (first-estimate Jacobians, observability constraints) | Li & Mourikis; Hesch et al. |
| Online temporal calibration | Estimate camera–IMU time offset as a state | Li & Mourikis, IJRR 2014; Qin & Shen, IROS 2018 |
| Consistency | Reported covariance should match actual error (NEES); inconsistent filters become over-confident | — |

Practical consequence of unobservability: VIO position and yaw drift without bound, and its reported covariance grows with time. Feeding that growing covariance straight to an autopilot would make it trust vision less and less. Hence the covariance policy in the design (incremental uncertainty with floors).

## 5. Aligning a drifting local frame with a global one

| Approach | Notes |
|---|---|
| One-shot alignment at start | Simple; sensitive to the moment chosen |
| Continuous least-squares alignment over a window (4-DoF: translation + yaw, since gravity fixes roll/pitch) | Robust; gives a residual as a quality measure |
| Joint optimisation of VIO + GNSS (global fusion) | Best accuracy; estimator-specific (for example VINS-Fusion's global fusion) |
| REP-105 `map → odom` | The standard place to hold the result |

Yaw between frames is observable from positions only when the vehicle translates; while stationary, attitude sources (compass) must be used.

## 6. External navigation in ArduPilot — practical notes [P][L]

- Barometer is recommended for vertical position even with a vision source, because vision height is sensitive to vibration and scale error.
- Yaw source can be compass or external nav. Indoors the compass is often unusable; outdoors it gives an absolute reference that vision lacks.
- With a tracking-camera type configured, ArduPilot can align yaw and position itself; with generic MAVLink input the data should already be in the autopilot's frame.
- The reset counter in vision messages tells the EKF to re-initialise rather than interpret a jump as motion.
- External-nav inputs are logged (`VISP`, `VISV`), allowing innovation analysis and delay tuning after a flight.
- Switching sources causes the EKF to reset to, or converge on, the new source; a mismatch appears as a step.

## 7. Confidence and integrity monitoring

| Technique | Notes |
|---|---|
| Covariance thresholds | Necessary; insufficient for an inconsistent filter |
| Feature count / track length | Early indicator of visual degradation |
| Innovation monitoring | Standard; depends on the filter not having already followed the fault |
| **Analytical redundancy** (compare with an independent sensor set) | Roll/pitch vs another AHRS; yaw rate vs another gyro; vertical velocity vs barometer; velocity vs GNSS when available |
| Solution separation / multiple estimators | Run two estimators and compare; costly |
| Minimum-of-checks aggregation | Conservative; avoids masking |

In GPS-denied mode the autopilot's horizontal position follows vision, so it cannot serve as a check. The FC's attitude, gyro and barometer remain independent of the camera. This observation shaped the confidence metric.

## 8. Coordinate conventions [L]

| System | World | Body |
|---|---|---|
| ROS (REP-103) | ENU | FLU |
| MAVLink / ArduPilot / PX4 | NED | FRD |
| Camera optical | — | x right, y down, z forward |
| OpenVINS | Gravity-aligned global, z up | IMU frame as mounted |
| Kalibr | — | `T_cam_imu`: IMU → camera |

Quaternion storage order differs between ROS (x, y, z, w) and MAVLink (w, x, y, z). Rotation direction conventions (active/passive, Hamilton/JPL) differ between libraries; OpenVINS uses JPL quaternions internally and publishes standard ROS messages [L]. All of this argues for one conversion point and bench checks rather than reasoning.

## 9. Time

| Issue | Notes |
|---|---|
| Sensor timestamps | Should be the time of measurement, not of arrival |
| Camera exposure | The effective time is mid-exposure (per row for rolling shutter) |
| Clock domains | Companion and FC clocks drift; MAVLink `TIMESYNC` estimates the offset |
| Transport delay | Serial at 921 600 baud adds a few milliseconds; processing adds tens |
| Effect of 50 ms unmodelled delay at 2 m/s | 10 cm apparent position error, growing with speed; destabilising in aggressive flight, tolerable in slow flight if compensated |

## 10. Takeaways used in the design

| Finding | Used in |
|---|---|
| Autopilot EKFs fuse delayed measurements using a delay parameter | `VISO_DELAY_MS` calibration |
| Loose coupling keeps the FC independent | Two-stage architecture |
| VIO covariance grows without bound | Covariance policy |
| Yaw alignment needs translation or a compass | Initialisation procedure |
| Corrections go in `map → odom` | TF design |
| Baro for height in vision mode | Source set 2 `POSZ = 1` |
| Independent-sensor cross-checks stay valid without GNSS | Confidence metric |
| Convention differences are numerous | Single conversion point; CF-1…CF-8 |
