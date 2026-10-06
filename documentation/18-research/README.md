# Research Notes — Index

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |

These notes record what was learned during the research pass. They are **background, not decisions**. Decisions are in [17-decisions](../17-decisions/README.md); designs are in sections 02–16.

| Note | Topics |
|---|---|
| [research-gps-denied-navigation.md](research-gps-denied-navigation.md) | GPS-denied navigation methods; GNSS failure modes; ArduPilot non-GPS navigation |
| [research-visual-geolocalization.md](research-visual-geolocalization.md) | **DB-2.0:** matching UAV images to satellite imagery; methods, datasets, failure cases, imagery licensing |
| [research-vio-slam.md](research-vio-slam.md) | VIO and SLAM families; estimators; sensor requirements; benchmarks |
| [research-stereo-vision.md](research-stereo-vision.md) | Stereo geometry, matching, synchronisation, rolling shutter, calibration |
| [research-ai-perception.md](research-ai-perception.md) | Edge detection models and runtimes on Raspberry Pi 5 |
| [research-uav-autonomy-ros2-mavlink.md](research-uav-autonomy-ros2-mavlink.md) | UAV autonomy architectures; ROS 2 on Pi 5; MAVLink; companion-computer integration |
| [research-state-estimation.md](research-state-estimation.md) | Filtering, loose vs tight coupling, EKF3, frame alignment, consistency |

## Source reliability key

| Mark | Meaning |
|---|---|
| **[P]** | Primary source consulted in this pass (manufacturer, official project documentation) |
| **[S]** | Secondary source consulted in this pass (search summary, reseller, community guide) |
| **[L]** | Literature known to the author of these notes; bibliographic details to be re-checked before citing in a report |
| **[U]** | Unverified; treat as a lead |

Full URL list: [references.md](../references.md).

## Open research questions carried into implementation

| # | Question | Resolved by |
|---|---|---|
| R-1 | Residual L/R skew achievable with libcamera software sync on two IMX219 sensors on one Pi 5 | Phase P04 measurement |
| R-2 | IMX219 readout time in the 1640×1232 binned mode under the Pi 5 driver | P04 |
| R-3 | Whether the pinned OpenVINS version models rolling-shutter readout or exposes feature counts | P07 |
| R-4 | Whether ArduPilot 4.7 uses `ODOMETRY` covariance and timestamp, and which frame IDs it accepts | P03 (SITL), P08 |
| R-5 | Whether simple avoidance limits GUIDED velocity commands in ArduPilot 4.7 | P03/P10 (SITL) |
| R-6 | Lua API available for observing a companion heartbeat in 4.7 | P05 |
| R-7 | State of MAVROS, OpenVINS and `ardupilot_gazebo` on ROS 2 Lyrical / Ubuntu 26.04 | Review trigger in ADR-001 |
| R-8 | Whether Ubuntu 24.04 point releases now package a libcamera with the Pi 5 pipeline handler | P04 |
| R-9 | MK15 air-unit lot and 4S support; HDMI converter connector and supply | Inspection of owned unit |
| R-10 | Indian price and IMU presence for the OAK-D Lite variant on sale | Before any purchase |
| R-11 | Which imagery sources cover the test site at ≤ 0.5 m/px under terms that allow offline academic use | Before P06G |
| R-12 | Acceptance rate and accuracy of SIFT matching on the site's terrain; the minimum useful height | Gate G2 |
| R-13 | XFeat export to NCNN/ONNX and its speed on the Pi 5 | P06G |
| R-14 | Applicable altitude limit and permissions for 50 m flight at the site with this vehicle category | Before stage A2 |
| R-15 | Timestamp jitter and manual-exposure support of the chosen UVC camera on Ubuntu 24.04 | P04 |
