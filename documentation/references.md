# References

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |

Sources are grouped by reliability following the project's source policy. "Consulted" means the page or a search summary of it was read during the research pass on 2026-10-05. Web content changes; re-check before citing in a formal report.

## 1. Manufacturer and vendor documentation (consulted)

| # | Source | Used for |
|---|---|---|
| V1 | Waveshare — IMX219-83 Stereo Camera wiki: https://www.waveshare.com/wiki/IMX219-83_Stereo_Camera | Sensor, resolution, FOV, baseline, IMU, Pi 5 overlays, statement that the camera has no hardware synchronisation |
| V2 | Raspberry Pi — Raspberry Pi 5 product page: https://www.raspberrypi.com/products/raspberry-pi-5/ | SoC, GPU, RAM, interfaces, power |
| V3 | Raspberry Pi — Camera software documentation (software camera synchronisation, `rpicam-apps` multi-camera): https://www.raspberrypi.com/documentation/computers/camera_software.html | Software sync mechanism and requirements |
| V4 | SIYI — MK15 Enterprise specifications: https://siyi.biz/en/product/hand-gcs/mk15-industry/spec/ | Range, channels, video, air-unit voltage/power/mass, converter |
| V5 | SIYI — MK32/HM30/MK15 Air Unit listing: https://shop.siyi.biz/products/mk32-hm30-mk15-air-unit | Air-unit interfaces, voltage range and lot note, power, mass |
| V6 | SIYI — Air Unit Recording HDMI Input Converter user manual: https://siyi.biz/siyi_file/Sky%20end%20card%20HDMI/Air_Unit_Recording_HDMI_Input_Converter_User_Manual_v1.1.pdf | Converter supply, IP address, RTSP |
| V7 | SIYI — MK15 user manual: https://siyi.biz/siyi_file/MK15/MK15%20User%20Manual%20v1.7.pdf | Network addresses, set-up |
| V8 | Holybro — Pixhawk 6C: https://holybro.com/products/pixhawk-6c | Processors, sensors, dimensions, mass, port current limits |
| V9 | Luxonis — OAK-D Lite documentation: https://docs.luxonis.com/hardware/products/OAK-D%20Lite | Sensors, baseline, IMU, mass |
| V10 | MicoAir — MTF-01: https://micoair.com/optical_range_sensor_mtf-01/ | Range, rate, power |
| V11 | Ultralytics — Raspberry Pi guide and benchmarks: https://docs.ultralytics.com/guides/raspberry-pi/ | YOLO26n format benchmarks on Raspberry Pi 5 |

## 2. Reseller listings (consulted, for product identification and indicative Indian prices)

| # | Source | Used for |
|---|---|---|
| R1 | Robu.in — Waveshare IMX219-83 (project reference link; page returned HTTP 403 to automated access; price taken from search listing): https://robu.in/product/waveshare-imx219-83-stereo-camera-8mp-binocular-camera-module-depth-vision/ | Product identity; price ≈ ₹4,799 |
| R2 | ElectroPi — SIYI MK15 HDMI Combo (project reference link): https://www.electropi.in/siyi-mk15-hdmi-combo-rc-transmitter | Package, specifications, price ₹53,430 excl. GST |
| R3 | Robu.in — Holybro Pixhawk 6C Mini + PM02 + M10 combo: https://robu.in/product/holybro-pixhawk-6c-mini-and-pm02-12s-and-m10-gps-combo/ | Indicative price ≈ ₹24,486 |
| R4 | Other Indian listings seen in search results (Zbotic, ElectronicsComp, ThinkRobotics, CrazyPi, TheEngineerStore, Robocraze, IndiaMART, Amazon.in, TheRoboMart, RCProduct) | Price ranges only |

## 3. Official ROS, ArduPilot, PX4 and MAVLink documentation (consulted)

| # | Source | Used for |
|---|---|---|
| D1 | ArduPilot — GPS / Non-GPS Transitions: https://ardupilot.org/copter/docs/common-non-gps-to-gps.html | EKF3 source sets, switching methods, cautions |
| D2 | ArduPilot — VIO tracking camera (Intel T265) set-up: https://ardupilot.org/copter/docs/common-vio-tracking-camera.html | `VISO_*`, source parameters, serial settings, message rates |
| D3 | ArduPilot — Depth camera for obstacle avoidance: https://ardupilot.org/copter/docs/common-realsense-depth-camera.html | `OBSTACLE_DISTANCE`, proximity and avoidance parameters, limitations |
| D4 | ArduPilot — EKF Source Selection and Switching: https://ardupilot.org/copter/docs/common-ekf-sources.html | `MAV_CMD_SET_EKF_SOURCE_SET` |
| D5 | ArduPilot — MicoAir MTF-01: https://ardupilot.org/copter/docs/common-mtf-01.html | Flow/range parameters |
| D6 | ArduPilot — ROS 2 and ROS 2 interfaces: https://ardupilot.org/dev/docs/ros2.html , https://ardupilot.org/dev/docs/ros2-interfaces.html | `AP_DDS` topics, services, Humble targeting |
| D7 | ArduPilot — ROS 2 with Gazebo; SITL with Gazebo: https://ardupilot.org/dev/docs/ros2-gazebo.html , https://ardupilot.org/dev/docs/sitl-with-gazebo.html | Simulation |
| D8 | ArduPilot — Non-GPS position estimation (MAVLink): https://ardupilot.org/dev/docs/mavlink-nongps-position-estimation.html | External-nav messages |
| D9 | ArduPilot — Setting home and EKF origin: https://ardupilot.org/dev/docs/mavlink-get-set-home-and-origin.html | `SET_GPS_GLOBAL_ORIGIN` |
| D10 | ArduPilot Discourse — Copter 4.7.0 / 4.7.1 release announcements: https://discuss.ardupilot.org/t/copter-4-7-1-released/145385 | Current stable version |
| D11 | PX4 — External position estimation: https://docs.px4.io/main/en/ros/external_position_estimation.html | EKF2 external vision |
| D12 | MAVLink — message definitions (common and ArduPilot dialect): https://mavlink.io/en/messages/common.html , https://mavlink.io/en/messages/ardupilotmega.html | Message semantics |
| D13 | ROS 2 documentation — Jazzy: https://docs.ros.org/en/jazzy/ | Distribution, concepts |
| D14 | ROS Enhancement Proposals — REP-103, REP-105, REP-147, REP-2000: https://www.ros.org/reps/rep-0105.html | Conventions, platform tiers |
| D15 | Gazebo — Installing Gazebo with ROS: https://gazebosim.org/docs/latest/ros_installation/ | Jazzy ↔ Harmonic pairing |
| D16 | OpenVINS documentation: https://docs.openvins.com/ | Estimator, configuration, installation |
| D17 | ROS package index / documentation — rtabmap, mavros (Jazzy): https://docs.ros.org/en/jazzy/p/rtabmap/ , https://index.ros.org/p/rtabmap_ros/ | Binary availability |

## 4. Source repositories (consulted or identified)

| # | Repository | Used for |
|---|---|---|
| G1 | rpng/open_vins (incl. pull request adding ROS 2 Jazzy / Ubuntu 24.04 support): https://github.com/rpng/open_vins | Primary VIO |
| G2 | introlab/rtabmap_ros (jazzy-devel): https://github.com/introlab/rtabmap_ros | Backup odometry |
| G3 | ArduPilot/ardupilot_gazebo: https://github.com/ArduPilot/ardupilot_gazebo | Simulation plugin, supported Gazebo versions |
| G4 | ArduPilot/ardupilot_gz: https://github.com/ArduPilot/ardupilot_gz | ROS 2 simulation packages (Humble-targeted) |
| G5 | mavlink/mavros: https://github.com/mavlink/mavros | Bridge |
| G6 | raspberrypi/libcamera: https://github.com/raspberrypi/libcamera | Camera stack for Pi 5 |
| G7 | christianrauch/camera_ros: https://github.com/christianrauch/camera_ros | Bring-up camera node |
| G8 | erykpawelek/libcamera_ros2_setup (community guide: Pi 5 + Ubuntu 24.04 + Jazzy camera): https://github.com/erykpawelek/libcamera_ros2_setup | libcamera build procedure |
| G9 | Community VINS-Fusion ROS 2 Jazzy ports (e.g. KJaebye/VINS-Fusion-ROS2-jazzy) | Alternative VIO availability |
| G10 | Tencent/ncnn: https://github.com/Tencent/ncnn | Inference runtime |
| G11 | ethz-asl/kalibr: https://github.com/ethz-asl/kalibr | Calibration |
| G12 | ArduPilot/ardupilot (releases; Lua example scripts `ahrs-source*.lua`): https://github.com/ArduPilot/ardupilot | Firmware; source-switching examples |
| G13 | stephendade/ros_stereo_ardupilot (stereo visual navigation with ArduPilot, example project): https://github.com/stephendade/ros_stereo_ardupilot | Prior art |

## 5. Peer-reviewed and technical literature

Bibliographic details are from memory and must be verified before formal citation.

| # | Reference | Relevance |
|---|---|---|
| L1 | A. I. Mourikis, S. I. Roumeliotis, "A Multi-State Constraint Kalman Filter for Vision-aided Inertial Navigation," ICRA 2007 | MSCKF |
| L2 | P. Geneva, K. Eckenhoff, W. Lee, Y. Yang, G. Huang, "OpenVINS: A Research Platform for Visual-Inertial Estimation," ICRA 2020 | Primary estimator |
| L3 | T. Qin, P. Li, S. Shen, "VINS-Mono: A Robust and Versatile Monocular Visual-Inertial State Estimator," IEEE T-RO 2018 | VINS family |
| L4 | T. Qin, J. Pan, S. Cao, S. Shen, "A General Optimization-based Framework for Local Odometry Estimation with Multiple Sensors" (VINS-Fusion), arXiv 2019 | Alternative estimator |
| L5 | C. Campos et al., "ORB-SLAM3: An Accurate Open-Source Library for Visual, Visual-Inertial and Multi-Map SLAM," IEEE T-RO 2021 | Evaluated SLAM |
| L6 | M. Labbé, F. Michaud, "RTAB-Map as an Open-Source Lidar and Visual SLAM Library for Large-Scale and Long-Term Online Operation," Journal of Field Robotics 2019 | Backup odometry / mapping |
| L7 | M. Burri et al., "The EuRoC Micro Aerial Vehicle Datasets," IJRR 2016 | Benchmark data |
| L8 | J. Delmerico, D. Scaramuzza, "A Benchmark Comparison of Monocular Visual-Inertial Odometry Algorithms for Flying Robots," ICRA 2018 | Embedded VIO trade-offs |
| L9 | J. Jeon et al., "Run Your Visual-Inertial Odometry on NVIDIA Jetson: Benchmark Tests on a Micro Aerial Vehicle," 2021 (arXiv:2103.01655) | Embedded VIO |
| L10 | D. Scaramuzza, F. Fraundorfer, "Visual Odometry: Part I — The First 30 Years and Fundamentals," IEEE Robotics & Automation Magazine 2011 | VO fundamentals |
| L11 | H. Hirschmüller, "Stereo Processing by Semiglobal Matching and Mutual Information," IEEE T-PAMI 2008 | SGM |
| L12 | P. Furgale, J. Rehder, R. Siegwart, "Unified Temporal and Spatial Calibration for Multi-Sensor Systems," IROS 2013 | Kalibr |
| L13 | C. Forster, L. Carlone, F. Dellaert, D. Scaramuzza, "On-Manifold Preintegration for Real-Time Visual-Inertial Odometry," IEEE T-RO 2017 | IMU pre-integration |
| L14 | M. Li, A. I. Mourikis, "Online Temporal Calibration for Camera-IMU Systems," IJRR 2014 | Time-offset estimation |
| L15 | "SMF-VO: Direct Ego-Motion Estimation via Sparse Motion Fields," arXiv:2511.09072 (2025) | Reports VINS-Fusion timing on Raspberry Pi 5 (via search summary) |
| L16 | Sensor-fusion study for RTAB-Map indoor mapping, arXiv:2305.04594 | Embedded RTAB-Map odometry rate (via search summary) |

## 5a. DB-2.0: satellite image matching (consulted through search summaries on 2026-10-05 unless marked)

| # | Source | Used for |
|---|---|---|
| G-1 | Kinnari et al., "GNSS-denied geolocalization of UAVs by visual matching of onboard camera images with orthophotos": https://arxiv.org/abs/2103.14381 | Standard approach (downward camera matched to a map); Monte-Carlo localisation |
| G-2 | Kinnari et al., "Season-invariant GNSS-denied visual localization for UAVs": https://arxiv.org/pdf/2110.01967 | Seasonal appearance change |
| G-3 | "GNSS-denied UAV localization with satellite and aerial image matching" (2025): https://www.sciencedirect.com/science/article/pii/S2590123025041787 | Recent method context |
| G-4 | UAV-VisLoc dataset: https://arxiv.org/abs/2405.11936 , https://github.com/IntelliSensing/UAV-VisLoc | Evaluation data: 6,742 drone images, 11 satellite maps at about 0.3 m |
| G-5 | "Hierarchical Image Matching for UAV Absolute Visual Localization via Semantic and Structural Constraints": https://arxiv.org/pdf/2506.09748 | AerialVL and UAV-VisLoc usage; retrieval + matching |
| G-6 | List of aerial localisation datasets: https://github.com/michaelschleiss/awesome-aerial-localization-datasets | Dataset options |
| G-7 | XFeat, Potje et al., CVPR 2024: https://arxiv.org/pdf/2404.19174 , https://github.com/verlab/accelerated_features | Lightweight learned matcher; CPU real-time claim |
| G-8 | ArduPilot / MAVProxy GPS Input module and non-GPS position estimation: https://ardupilot.org/mavproxy/docs/modules/GPSInput.html , https://ardupilot.org/dev/docs/mavlink-nongps-position-estimation.html | `GPS_INPUT` fallback path |
| G-9 | Google Maps Platform terms and Map Tiles API policies: https://developers.google.com/maps/documentation/tile/policies | Prohibition on caching / offline use |
| G-10 | Copernicus Sentinel-2 on AWS: https://registry.opendata.aws/sentinel-2/ | Open licence; 10 m resolution |
| G-11 | OpenAerialMap: https://openaerialmap.org/about/ | CC-BY 4.0 imagery |
| G-12 | Summaries of India's Drone Rules 2021 (several sites) | Green zone to 400 ft / 120 m; micro category; VLOS. **Verify against the official rules** |
| G-13 | From memory, to verify before citing: Goforth & Lucey (ICRA 2019); Patel et al. (ICRA 2020); Bianchi & Barfoot (RA-L 2021); Couturier & Akhloufi, review of absolute visual localisation for UAVs (2021); Lowe, SIFT (IJCV 2004) | Literature survey |

## 5b. DB-3.0: MK15 app architecture (consulted through search summaries on 2026-10-06)

| # | Source | Used for |
|---|---|---|
| A-1 | SIYI MK15 user manual (v1.5–v1.9): https://siyi.biz/siyi_file/MK15/MK15%20User%20Manual%20v1.7.pdf | Datalink connection types on the ground unit (UART, USB COM, Bluetooth, upgrade port, UDP); built-in UART needs SIYI's SDK for third-party ground stations; QGroundControl set-up over USB COM or UDP; reserved IP addresses; third-party IP cameras |
| A-2 | ArduPilot Discourse, "SIYI MK15 connection problem": https://discuss.ardupilot.org/t/siyi-mk15-connecti-on-problem/117249 | Practical QGroundControl connection notes |
| A-3 | QGroundControl MAVLink settings and issue "mavlink forwarding is one-way": https://docs.qgroundcontrol.com/Stable_V4.3/en/qgc-user-guide/settings_view/mavlink.html , https://github.com/mavlink/qgroundcontrol/issues/10682 | Forwarding cannot carry commands from a second app |
| A-4 | mavlink-kotlin: https://github.com/divyanshupundir/mavlink-kotlin ; dronefleet MAVLink for Java: https://mvnrepository.com/artifact/io.dronefleet.mavlink ; MAVLink implementations list: https://mavlink.io/en/about/implementations.html | Fallback option if the IP path cannot be used |
| A-5 | From general knowledge, to verify: VisDrone dataset and its terms; Android 9 / API 28 capabilities; osmdroid offline tiles; UVC 1080p MJPEG over USB 2.0 | App and aerial-detection design |

Not found in this pass and therefore marked `[VERIFY]`: explicit confirmation that an Android app on the MK15 can open sockets to an arbitrary device on the air unit's Ethernet, and the bandwidth of that path. This is gate G0.

## 6. Secondary and community sources (used for corroboration only)

| # | Source | Used for |
|---|---|---|
| S1 | Open Robotics Discourse — real-time Raspberry Pi ROS 2 image for Jazzy / 24.04; camera installation thread | Pi 5 + Jazzy feasibility |
| S2 | Community write-ups on IMX219 / Camera Module 3 with Pi 5, Ubuntu 24.04 and Jazzy | libcamera build need |
| S3 | Community example of OpenVINS (ROS-free) noting camera requirements | Sensor guidance for VIO |
| S4 | Blog: OAK-D Lite with ROS 2 Jazzy on Ubuntu 24.04 on Raspberry Pi 5 | Upgrade-path feasibility |
| S5 | Third-party Raspberry Pi 5 power measurements (CNX Software, Jeff Geerling, others) | Power estimates |
| S6 | Articles summarising ROS 2 Lyrical Luth vs Jazzy | Distribution status |
| S7 | ArduPilot Discourse thread "GPS to non-GPS with MAVLink" | Source-set command usage |

## 7. Items stated from general engineering knowledge without a consulted source

These are tagged `[VERIFY]` where they appear and are collected here so that they are checked at bring-up:

- Pixhawk TELEM connector pin order; ArduPilot serial index mapping on Pixhawk 6C.
- Exact enumerated values and names of ArduPilot 4.7 parameters and SITL fault-injection parameters.
- Raspberry Pi 5 UART overlay name, throttle temperatures, GPIO power configuration keys.
- IMX219 pixel pitch and readout time; ICM-20948 I²C address and INT availability on the Waveshare board.
- MK15 operating band; S.Bus frame rate; forwarding options.
- MAVROS default component IDs, plugin topic names in 2.14, licence terms.
- Model parameter counts and COCO scores for the detectors.
- Indian regulatory categories and requirements.
- Masses marked `[ESTIMATE]` in the weight budget.
