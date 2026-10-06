# Sensor Suite

| Field | Value |
|---|---|
| Document ID | GDN-HW-007 |
| Version | 1.0 |
| Date | 2026-10-05 |

## 1. Principle

A sensor is added only if a requirement cannot be met without it. Each entry below states the requirement it serves and what happens if it is absent.

## 2. Required sensors

| Sensor | Part | Location | Serves | Without it |
|---|---|---|---|---|
| IMU (flight) | ICM-42688-P + BMI088 | Inside Pixhawk 6C | Attitude control, EKF3 prediction | Cannot fly |
| Barometer | MS5611 | Inside Pixhawk 6C | Altitude (EKF `POSZ` source in every set) | No altitude hold |
| Magnetometer | IST8310 on the GNSS mast (external, primary); internal as backup | GPS1 cable | Heading for GNSS flight and for outdoor vision flight | Yaw must come from GNSS motion or vision |
| GNSS | Holybro M10 | Mast | Tier-1 navigation (FR-001); the reference against which vision is aligned (FR-022); ground truth for evaluating VIO drift outdoors | No GNSS-assisted mode, no transition to demonstrate, no outdoor ground truth |
| Stereo camera | 2 × IMX219 | Front | Low-regime VIO, depth, AI | No obstacle sensing, no object ranging, no low-altitude vision odometry |
| **Downward camera (DB-2.0)** | USB 2.0 UVC, wide lens | Underside | Satellite map matching (FR-092) and ground visual odometry (FR-094) | No absolute position without GPS: the core of the DB-2.0 concept is impossible |
| IMU (VIO) | ICM-20948 on the camera board | Camera board | VIO inertial input, rigidly attached to the cameras, on the Pi clock | VIO must use FC IMU over MAVLink (see [ADR-014](../17-decisions/ADR-014-vio-imu-source.md)) |
| Downward range sensor | ToF in MicoAir MTF-01 | Underside | Height above ground for take-off/landing and terrain following (FR-016); scale for optical flow | Baro-only height near the ground (drifts, ground effect); no optical-flow tier |

## 3. Recommended sensor (safety tier)

| Sensor | Part | Serves | Justification |
|---|---|---|---|
| Optical flow | PMW3901-class sensor in the MTF-01 (same module as the range sensor) | FR-025: a horizontal velocity source that does **not** depend on the Pi | Without it, a companion crash while GNSS is denied leaves only ALT_HOLD, in which the vehicle drifts with wind until the pilot reacts. With it, the FC holds position by itself. One small module (≈ 0.5 W, 100 Hz output, ToF to 8 m per vendor) covers both range and flow. |

The MTF-01 connects by MAVLink serial. ArduPilot setup per its guide: `FLOW_TYPE = 5`, `RNGFND1_TYPE = 10`, `RNGFND1_MAX = 8`, `RNGFND1_MIN = 0.01`, `SERIALx_PROTOCOL = 1`, `SERIALx_BAUD = 115`. On ArduPilot 4.5 and later, the sensor's MAVLink ID must be changed from 1 and the port set to not forward (`SERIALx_OPTIONS = 1024`).

Limits: needs a textured, lit surface below; range ≤ 8 m (less in sunlight: 5 m at 60 klux per vendor); not valid over water or uniform floors.

## 4. Sensors considered and not included

| Sensor | Reason not included |
|---|---|
| 2D / 3D LiDAR | Would give robust localisation and obstacle data, but mass (≥ 150 g for useful units), cost and compute change the project into a LiDAR project. Outside the "low-cost vision" contribution. Listed as a future upgrade. |
| UWB anchors/tags | Needs installed infrastructure; contradicts the goal of infrastructure-free navigation. Useful only as an indoor ground-truth aid (optional). |
| Second GNSS / RTK | Not needed for flight. RTK would be valuable as centimetre ground truth for evaluating VIO; listed as optional. |
| Separate high-grade IMU for VIO | The ICM-20948 is sufficient to start. Revisit only if calibration shows it is the limiting factor. |
| Forward ToF / sonar | Stereo depth covers the forward sector; a single-beam sensor adds little. |
| Airspeed sensor | Not relevant to a multirotor. |
| Thermal / event cameras | Out of scope and budget. |
| Motion-capture system | Would be the ideal indoor ground truth; use it if the institute has one. Not a project purchase. |

## 5. IMU comparison (flight vs VIO)

| | FC IMU (ICM-42688-P) | Camera-board IMU (ICM-20948) |
|---|---|---|
| Grade | Consumer, low-noise, current generation | Consumer, previous generation |
| Bus | SPI, kHz sampling inside the FC | I²C from the Pi, ≈ 225 Hz |
| Mechanical | Isolated and temperature-controlled inside the FC | Rigid on the camera PCB |
| Clock | FC clock | Pi clock (same as images) |
| Rigid to camera | No (FC and camera are on different dampers) | **Yes** |
| Used for | Flight control and EKF3 | VIO |

For VIO the two properties that matter most are a rigid camera–IMU transform and a common clock. The camera-board IMU has both; the FC IMU has neither. That outweighs its better noise figures.

## 6. Ground-truth and evaluation aids (not flight sensors)

| Aid | Use | Cost |
|---|---|---|
| GNSS position (when GOOD) | Outdoor reference for VIO drift: run VIO in shadow during GNSS flight and compare | None |
| Surveyed ground markers / tape-measured course | Loop-closure error: return to the take-off marker and measure the miss distance | Negligible |
| AprilTag board at known positions | Spot checks of absolute pose from the camera | Printing |
| RTK GNSS (optional) | Centimetre reference outdoors | High |
| Motion capture (if available at the institute) | Indoor reference | None to the project |

## 7. Summary

Additional purchases for sensing: **two items** beyond what comes with the flight-controller combo: the MTF-01 and, since DB-2.0, a downward USB camera. Everything else is either inside the Pixhawk, bundled with it, or already owned.

Note on the MTF-01 in DB-2.0: its 8 m range means the optical-flow fallback tier exists only in the low regime. At cruise height (40–60 m) there is no flight-controller-native horizontal fallback. A longer-range rangefinder would not change this, because the flow sensor itself needs to resolve ground texture. This limitation is carried into the safety architecture.
