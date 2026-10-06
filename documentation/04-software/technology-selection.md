# Technology Selection

| Field | Value |
|---|---|
| Document ID | GDN-SW-002 |
| Version | 1.0 |
| Date | 2026-10-05 |

The project brief lists candidate technologies and asks that they not all be used automatically. This table gives the verdict on each, plus the additional technologies the design needs.

## 1. Technologies named in the brief

| Technology | Verdict | Role / reason |
|---|---|---|
| **ROS 2** | **Adopted** (Jazzy) | Mandatory middleware. [ADR-001](../17-decisions/ADR-001-ros2-distribution.md) |
| **OpenCV** | **Adopted** | Stereo block matching, rectification maps, image conversion, drawing the HUD; the simplified VO fallback. Version from Ubuntu 24.04 (4.6). |
| **MAVLink** | **Adopted** (v2) | The only protocol between companion and FC and between FC and GCS. |
| **MAVSDK** | **Rejected** | Designed primarily around PX4; its ArduPilot coverage is partial, and it would duplicate MAVROS. |
| **ArduPilot** | **Adopted** | Autopilot firmware. [ADR-002](../17-decisions/ADR-002-autopilot-firmware.md) |
| **PX4** | **Rejected** (as firmware) | Good alternative; loses on explicit EKF source switching for this use case. Kept as the documented fallback in ADR-002. |
| **OpenVINS** | **Adopted** (primary VIO) | Filter-based stereo-inertial odometry, efficient on CPU, online calibration of extrinsics and time offset. [ADR-004](../17-decisions/ADR-004-vio-solution.md) |
| **ORB-SLAM3** | **Rejected** | Heaviest option; no maintained official ROS 2 wrapper; stereo-inertial mode is sensitive to exactly the timing weaknesses of this camera. |
| **RTAB-Map** | **Adopted in two limited roles** | (1) Backup odometry: its stereo odometry node (visual-only, binary package for Jazzy). (2) Optional mapping on recorded bags. Not used as in-flight SLAM. [ADR-005](../17-decisions/ADR-005-slam-solution.md) |
| **Eigen** | **Adopted** | Linear algebra in C++ nodes (transforms, alignment). Already a dependency of OpenVINS and tf2. |
| **NumPy** | **Adopted** | Python nodes and analysis tools. |
| **TensorFlow Lite (LiteRT)** | **Rejected as runtime** | Slower than NCNN on Pi 5 in the vendor benchmark (≈ 123 ms vs ≈ 67 ms for YOLO26n at 640 px). Kept only as a comparison point in the AI benchmark. |
| **ONNX Runtime** | **Deferred** | Slower than NCNN on this CPU (≈ 126 ms). ONNX is used as an interchange format during export. |
| **PyTorch** | **Adopted off-board only** | Training and export on a workstation or cloud GPU. **Not installed on the drone**: ≈ 299 ms per frame and a large memory footprint. |
| **Lightweight YOLO** | **Adopted** | YOLO26n (fallback YOLO11n), 320 px. [ADR-006](../17-decisions/ADR-006-ai-framework.md) |
| **Gazebo** | **Adopted** (Harmonic) | Physics and sensor simulation. [ADR-013](../17-decisions/ADR-013-simulation.md) |
| **RViz2** | **Adopted** (workstation only) | Visualisation of TF, images, odometry, detections. Not run on the drone. |

## 2. Additional technologies required by the design

| Technology | Role | Reason |
|---|---|---|
| **MAVROS** | ROS 2 ↔ MAVLink bridge | Mature, binary for Jazzy on arm64, handles ENU/NED conversion and time sync, has plugins for odometry, obstacle distance, setpoints and commands. [ADR-008](../17-decisions/ADR-008-mavlink-architecture.md) |
| **libcamera (Raspberry Pi fork)** | Camera access on Pi 5 | Required for the Pi 5 ISP pipeline; provides software camera synchronisation. [ADR-007](../17-decisions/ADR-007-camera-interface.md) |
| **NCNN** | Neural-network inference | Fastest CPU runtime on Pi 5 in the vendor benchmark; small C++ library without Python dependency. |
| **Ultralytics** (off-board) | Training and export of the detector | Standard tooling for YOLO26/YOLO11. AGPL-3.0 licence noted. |
| **Kalibr** | Camera, stereo and camera-IMU calibration | Reference tool; outputs convert directly to OpenVINS configuration. Run on a workstation (Docker). |
| **allan_variance_ros** or equivalent | IMU noise identification | Supplies IMU noise parameters to Kalibr and OpenVINS. |
| **`image_proc` / `stereo_image_proc`** | Rectification and (initially) disparity | Standard ROS 2 components; reused instead of re-implemented. |
| **`vision_msgs`** | Detection message types | Standard messages; avoids custom types. |
| **`robot_state_publisher`, tf2** | Static transforms | Standard. |
| **`diagnostic_updater` / `diagnostic_aggregator`** | Health reporting | Standard. |
| **rosbag2 + MCAP** | Recording | Default in Jazzy; crash-tolerant chunked format. |
| **ArduPilot SITL + `ardupilot_gazebo`** | Software-in-the-loop | Runs the real ArduPilot code, including EKF3 and source switching. |
| **`ros_gz`** | Gazebo ↔ ROS 2 bridge | Simulated camera, IMU and clock into the same topics as hardware. |
| **ArduPilot Lua scripting** | FC-side companion watchdog | Keeps the reaction to companion loss inside the FC. |
| **evo** | Trajectory evaluation | ATE/RPE between VIO and reference. Off-board. |
| **pytest, GoogleTest, launch_testing** | Tests | Standard ROS 2 test stack. |
| **systemd** | Autostart and restart | Headless boot to READY. |

### DB-2.0 additions (satellite map matching)

| Technology | Role | Reason |
|---|---|---|
| **OpenCV SIFT, FLANN matcher, RANSAC (`estimateAffinePartial2D`)** | Primary map-matching method | No training; geometric verification; in the OpenCV already selected |
| **OpenCV FAST + KLT** | Ground visual odometry | Standard, cheap |
| **XFeat** (learned lightweight features) | Matching method to compare against SIFT; second use of AI in the project | Published as real-time on a laptop CPU; run through NCNN or ONNX Runtime if export succeeds; adopted only if measurably better within budget |
| **GDAL / rasterio, pyproj** (off-board; `map_prepare`) | Reading GeoTIFF, reprojection to UTM, tiling | Standard geospatial tooling |
| **GeographicLib or PROJ** (onboard) | WGS-84 ↔ UTM ↔ local ENU | Small, exact, well tested |
| **`usb_cam` / `v4l2_camera`** | Downward UVC camera driver | Standard ROS 2 nodes |
| **OpenDroneMap** (off-board, optional) | Own orthomosaic of the test site | Open source; gives a licence-clean, high-resolution reference |
| **QGIS** (off-board) | Inspecting and georeferencing the reference image, checking the map pack | Standard |

Rejected for onboard use: SuperPoint + LightGlue and LoFTR (need a GPU for real time); image-retrieval networks (unnecessary for a 1 km² map with a position prior).

Licence additions: SIFT (patent expired; in OpenCV main), XFeat (Apache-2.0 per its repository `[VERIFY]`), GDAL (MIT/X), PROJ (MIT), OpenDroneMap (AGPL-3.0, used as a separate tool). **Reference imagery carries its own licence**, recorded in each map pack ([ADR-016](../17-decisions/ADR-016-reference-imagery-and-downward-camera.md)).

### DB-3.0 additions (Android app, search, track, follow)

| Technology | Role | Reason |
|---|---|---|
| **Kotlin, Android SDK (API 28+), Android Studio** | Native ground app for the MK15 | Owner's decision ([ADR-017](../17-decisions/ADR-017-ground-app.md)); standard Android stack |
| **Jetpack Compose** (XML views as fallback) | App user interface | Fast to build; performance on the MK15's 2 GB RAM to be checked |
| **OkHttp (WebSocket), kotlinx.serialization** | App networking and JSON | Mature and small |
| **osmdroid** (or MapLibre) with an offline custom tile source | Map screen | Works without internet; shows the drone's own map pack |
| **Room** | Findings and events stored on the device | Standard |
| **Python `websockets` / asyncio** | `app_gateway` on the Pi | Simple; fits the low message rates |
| **libjpeg-turbo** (through OpenCV) | Video frames for the app | Cheap software encoding; no hardware encoder on the Pi 5 |
| **Aerial-view YOLO26n** (second model, NCNN) | Detection from the downward camera | Same runtime as the ground-view model; fine-tuned on aerial data |
| **VisDrone** (and own data) | Aerial training data | Public aerial dataset; licence terms to be checked for the use made of it |
| **OpenCV Kalman filter** | Target tracking in ground coordinates | Sufficient; nothing heavier is justified |

Considered and not chosen for the app's link: MAVLink libraries for Android (mavlink-kotlin, dronefleet) and MAVSDK-Java. They are the fallback if the MK15's IP path cannot be used ([ADR-017](../17-decisions/ADR-017-ground-app.md)). Not used: SIYI's SDK for the built-in UART (the app does not touch the telemetry datalink); learned re-identification trackers; a web app (declined by the owner).

## 3. Technologies deliberately not used

| Technology | Reason |
|---|---|
| **Nav2** | Built for 2D ground robots (costmaps, planners, controllers assuming planar motion). Heavy for a Pi already at its CPU limit. The navigation need here is a short waypoint follower with a stop condition. |
| **`robot_localization`** | A second EKF on the companion would compete with EKF3 and add tuning burden for no benefit ([state-estimation](../09-navigation/state-estimation.md) §2). |
| **micro-ROS / `AP_DDS`** | ArduPilot's native DDS interface is documented against ROS 2 Humble and exposes fewer functions than MAVROS. Revisit in a later revision. |
| **Docker on the drone** | Adds complexity to camera and device access for no benefit on a single-purpose computer. Used on the workstation for Kalibr only. |
| **PREEMPT_RT kernel** | No hard real-time need on the companion. |
| **Learned depth / learned VIO** | Not real-time on this CPU. |
| **Behaviour trees (BehaviorTree.CPP)** | The mission logic is a short linear sequence; a plain state machine is easier to verify. |
| **Cyclone DDS** | No identified problem with the default. Switch only if Fast DDS shows issues. |

## 4. Licence register

| Component | Licence | Obligation relevant to a college project |
|---|---|---|
| ROS 2 core | Apache-2.0 | Attribution |
| ArduPilot | GPL-3.0 | Source of any modified firmware must be made available if distributed. The design does not modify firmware. |
| MAVROS | Triple-licensed (GPL-3.0 / LGPL-3.0 / BSD) `[VERIFY]` | Used unmodified as a separate process |
| OpenVINS | GPL-3.0 | Used unmodified as a separate process; any modifications published under GPL |
| RTAB-Map | BSD-3 | Attribution |
| OpenCV | Apache-2.0 | Attribution |
| NCNN | BSD-3 | Attribution |
| Ultralytics YOLO (code and pretrained weights) | AGPL-3.0 | Derived models and the code that uses them must be open-sourced under AGPL if distributed or offered as a service. Acceptable for an open academic project; **state this in the report.** A commercial follow-on would need an Ultralytics licence or a differently licensed model. |
| Kalibr | BSD | Attribution |
| Gazebo | Apache-2.0 | Attribution |

The project's own packages are intended to be released under a licence compatible with the above (GPL-3.0 is the simplest consistent choice; decision recorded as open item OD-7 in [project-status](../project-status.md)).

## 5. Version baseline

| Component | Version | Note |
|---|---|---|
| Ubuntu | 24.04.x LTS arm64 (Pi), amd64 (workstation) | |
| ROS 2 | Jazzy Jalisco | |
| Gazebo | Harmonic | Paired with Jazzy |
| ArduPilot | Copter 4.7.x stable | 4.7.1 current at time of writing |
| MAVROS | 2.14.x (Jazzy apt) | Version seen in the Jazzy repository |
| OpenVINS | Latest master with the Jazzy/Ubuntu 24.04 fixes | Pin a commit hash at first successful build |
| RTAB-Map | 0.23.x (Jazzy apt) | |
| OpenCV | 4.6 (Ubuntu 24.04) | |
| NCNN | Latest release; pin tag | Build from source with NEON |
| YOLO | YOLO26n; Ultralytics 8.4.x for export | |
| Python | 3.12 | |
| C++ | C++17 | |

Versions are pinned in a `repos` file (vcstool) and an apt package list at the start of implementation.
