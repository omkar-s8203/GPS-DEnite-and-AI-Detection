# ROS 2 Node Reference

| Field | Value |
|---|---|
| Document ID | GDN-ROS-004 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | Baseline — specification for implementation |

Message types, rates and QoS for every topic are tabulated in [interfaces.md](interfaces.md); they are repeated here only where needed to understand a node. Criticality classes (A/B/C) are defined in [software-architecture.md](../04-software/software-architecture.md) §5.

## Node index

| # | Node | Package | Lang. | Class | Lifecycle |
|---|---|---|---|---|---|
| 1 | `stereo_camera` | `gdn_camera` | C++ | A | Yes |
| 2 | `imu_driver` | `gdn_imu` | C++ | A | Yes |
| 3 | `rectify_left`, `rectify_right` | `image_proc` | C++ | B | No |
| 4 | `stereo_depth` | `gdn_stereo` | C++ | B | Yes |
| 5 | `obstacle_sectors` | `gdn_obstacle` | C++ | B | Yes |
| 6 | `open_vins` | third-party | C++ | A | No |
| 7 | `vio_monitor` | `gdn_vio` | C++ | A | Yes |
| 8 | `stereo_vo` | `gdn_vo_simple` | C++ | — (alternative to 6) | Yes |
| 9 | `localization_manager` | `gdn_localization` | C++ | A | Yes |
| 10 | `nav_mode_manager` | `gdn_nav_mode` | Python | A | Yes |
| 11 | `detector` | `gdn_perception` | C++ | C | Yes |
| 12 | `object_localizer` | `gdn_perception` | C++ | C | Yes |
| 13 | `navigator` | `gdn_navigation` | Python | B | Yes |
| 14 | `mission_manager` | `gdn_mission` | Python | B | Yes |
| 15 | `safety_supervisor` | `gdn_safety` | Python | A | No |
| 16 | `telemetry_node` | `gdn_telemetry` | Python | C | No |
| 17 | `hud_node` | `gdn_telemetry` | C++ | C | No |
| 18 | `system_monitor` | `gdn_diagnostics` | Python | C | No |
| 19 | `lifecycle_manager` | `gdn_bringup` | Python | A | No |
| 20 | `mavros` | third-party | C++ | A | No |
| 21 | `robot_state_publisher` | standard | C++ | A | No |
| 22 | `diagnostic_aggregator` | standard | C++ | C | No |

---

## 1. `stereo_camera`

| Item | Specification |
|---|---|
| Responsibility | Open both IMX219 sensors in one process through libcamera; run software synchronisation; pair frames by sensor timestamp; publish left and right with identical header stamps; report skew |
| Inputs | CSI-2 cameras 0 and 1; calibration files |
| Outputs | `/stereo/{left,right}/image_raw`, `/stereo/{left,right}/camera_info`, `/stereo/left/image_color`, `/stereo/sync_status`, heartbeat, diagnostics |
| Services / actions | Lifecycle only |
| Frequency | 20 Hz (parameter); colour stream 10 Hz |
| QoS | SENSOR |
| Dependencies | libcamera (Raspberry Pi fork), `camera_info_manager`, `image_transport` |
| Key parameters | `frame_rate` (20), `width` (640), `height` (480), `sensor_mode` (1640×1232), `exposure_us` (3000), `analog_gain_max` (8.0), `auto_gain` (true), `sync_enabled` (true), `max_skew_ms` (1.0), `drop_unsynced` (true), `left_camera_index`, `right_camera_index`, `calibration_id` |
| Timestamp rule | `header.stamp` = left sensor frame-start time + half the exposure, converted from the kernel boot clock to ROS time with a fixed offset measured at start-up. Right image carries the same stamp; the true difference is in `sync_status.skew_ms`. |
| Failure behaviour | Camera not found at configure → lifecycle error, NO-GO. Frame timeout > 200 ms → diagnostics ERROR; after 1 s → restart the capture pipeline once; if it fails again → lifecycle error. Skew above threshold → pair dropped and counted; > 5 % dropped → `synchronized = false`, which lowers localisation confidence. |

## 2. `imu_driver`

| Item | Specification |
|---|---|
| Responsibility | Configure the ICM-20948 and publish gyro and accelerometer samples with low-jitter timestamps |
| Inputs | `/dev/i2c-1`; optional data-ready GPIO |
| Outputs | `/imu/data_raw` (`imu_link`; orientation covariance[0] = −1), heartbeat, diagnostics |
| Frequency | ≈ 225 Hz (ODR divider 4) |
| QoS | SENSOR |
| Key parameters | `i2c_bus` (1), `address` (0x68), `gyro_range_dps` (500), `accel_range_g` (8), `dlpf_gyro_hz` (≈ 51–119, selected in calibration), `dlpf_accel_hz`, `odr_divider` (4), `use_drdy_interrupt` (false), `gyro_noise_density`, `accel_noise_density` (for covariance fields) |
| Timestamp rule | Interrupt mode: time of the data-ready edge. Polling mode: read time minus half the sample period. Burst-read gyro and accel in one transaction so that they share a stamp. |
| Notes | The magnetometer is not read (near motors and the Pi; unused). No bias removal or filtering beyond the sensor's own low-pass filter: VIO estimates biases. |
| Failure behaviour | Wrong `WHO_AM_I` → configure error. I²C error → retry; 3 consecutive failures → lifecycle error. Sample gap > 20 ms → counted; > 10 gaps per minute → diagnostics WARN. Saturation → counted and WARN. |

## 3. `rectify_left`, `rectify_right`

| Item | Specification |
|---|---|
| Responsibility | Undistort and rectify images for the depth pipeline (not for VIO, which uses raw images with its own camera model) |
| Implementation | `image_proc::RectifyNode` components in `sensor_container`, intra-process |
| Inputs / outputs | `image_raw` + `camera_info` → `image_rect` |
| Frequency | 10 Hz (every second pair; a decimating subscription or the `image_proc` crop/decimate node) |
| Failure behaviour | Stateless; restarts with the container |

## 4. `stereo_depth`

| Item | Specification |
|---|---|
| Responsibility | Compute disparity and metric depth from a rectified pair |
| Inputs | `/stereo/{left,right}/image_rect`, `camera_info` (exact-time synchroniser: stamps are identical by construction) |
| Outputs | `/stereo/depth/image` (32FC1, metres, NaN invalid, registered to the left rectified image), `/stereo/depth/camera_info`, `/stereo/disparity` (optional) |
| Frequency | 10 Hz |
| QoS | SENSOR |
| Algorithm | OpenCV `StereoBM` by default; `StereoSGBM` (3-way) selectable. Post-filters: uniqueness ratio, speckle filter, left-right consistency if enabled. |
| Key parameters | `algorithm` (bm), `num_disparities` (64), `block_size` (15), `uniqueness_ratio` (10), `speckle_window` (100), `speckle_range` (2), `min_depth` (0.4), `max_depth` (8.0), `process_scale` (1.0; 0.5 halves the resolution for speed) |
| Failure behaviour | No valid input for 500 ms → diagnostics STALE; downstream treats obstacles as unknown and the navigator holds. Processing time > 100 ms sustained → node self-reduces `process_scale` and reports WARN. |

## 5. `obstacle_sectors`

| Item | Specification |
|---|---|
| Responsibility | Reduce the depth image to nearest-obstacle distances per horizontal sector, for the navigator and for the FC |
| Inputs | `/stereo/depth/image`, `camera_info`, TF `base_link ← left_camera_optical_frame`, `/mavros/imu/data` (to level the band using roll/pitch) |
| Outputs | `/obstacle/sectors`; `/mavros/obstacle/send` (`LaserScan`, 72 × 5°, only the bins inside the camera FOV are finite; the rest report "unknown" as MAVROS/ArduPilot expect) |
| Frequency | 10 Hz |
| Method | Take a horizontal band of the depth image corresponding to the vehicle's flight corridor (± 0.5 m vertically at the stop distance); project to `base_link`; per 5° sector take a low percentile (5th) of valid depths, requiring a minimum pixel count to reject speckle. |
| Key parameters | `sector_width_deg` (5), `band_half_height_m` (0.5), `percentile` (5), `min_pixels` (30), `min_range` (0.5), `max_range` (6.0), `min_valid_fraction` (0.3) |
| Failure behaviour | `valid_fraction` below threshold (dark, texture-less) → sectors published as unknown (NaN). Unknown is **not** treated as clear by the navigator. |

## 6. `open_vins`

| Item | Specification |
|---|---|
| Responsibility | Stereo visual-inertial odometry (MSCKF) |
| Executable | `ov_msckf` ROS 2 subscriber node, unmodified |
| Inputs | `/stereo/left/image_raw`, `/stereo/right/image_raw`, `/imu/data_raw` |
| Outputs | `/ov_msckf/odomimu` (pose of the IMU in OpenVINS's global frame, with covariance; twist in the IMU frame), debug topics. Its TF broadcasting is disabled. |
| Frequency | 20 Hz (camera rate) |
| Configuration | `gdn_vio/config/openvins/`: `estimator_config.yaml`, `kalibr_imu_chain.yaml`, `kalibr_imucam_chain.yaml`. Key settings in [vio-design.md](../07-vio-slam/vio-design.md) §4. |
| Dependencies | OpenCV, Eigen, Ceres (initialisation) |
| Failure behaviour | Divergence or initialisation failure is not self-reported reliably; `vio_monitor` detects it. Process crash → launch respawn after 2 s; `vio_monitor` increments the reset counter. |

## 7. `vio_monitor`

| Item | Specification |
|---|---|
| Responsibility | Adapter and health monitor between any VIO implementation and the rest of the system. Converts IMU-frame odometry to `odom → base_link`; publishes TF; detects resets, jumps and divergence; publishes `VioStatus`; owns VIO restart |
| Inputs | `/ov_msckf/odomimu` (or the backup/simple VO odometry, by remap), `/stereo/sync_status`, static TF `imu_link → base_link`, `/mavros/local_position/odom` (fallback source for `odom → base_link` when VIO is unavailable), `/mavros/imu/data` (cross-check) |
| Outputs | `/vio/odometry`, `/vio/status`, TF `odom → base_link`, heartbeat, diagnostics |
| Services | `/vio/reset` |
| Frequency | Pass-through at VIO rate; status at 10 Hz |
| QoS | ESTIMATE |
| Health logic | TRACKING requires: rate ≥ 15 Hz; latency ≤ 100 ms; position covariance trace below `cov_max`; no pose step > `max_step_m` between consecutive outputs; speed < `max_speed`; roll/pitch within `att_tol_deg` of FC attitude. Violations → DEGRADED; persistent (> 1 s) or stream gap > 0.5 s → LOST. |
| Key parameters | `source_topic`, `min_rate_hz` (15), `max_latency_ms` (100), `cov_max` (1.0 m²), `max_step_m` (0.5), `max_speed` (5.0), `att_tol_deg` (5), `min_features` (30, if available from the estimator), `fallback_to_fc_odom` (true) |
| Failure behaviour | On LOST: stop publishing `/vio/odometry`; keep TF alive from the FC odometry so that `map → base_link` remains valid for other nodes; status reflects LOST. Own crash → respawn; `localization_manager` sees the gap and reports LOST. |

## 8. `stereo_vo` (simplified implementation)

| Item | Specification |
|---|---|
| Responsibility | Teaching and fallback stereo visual odometry using only OpenCV |
| Inputs | Rectified left/right images, `camera_info` |
| Outputs | `nav_msgs/Odometry` (camera pose → converted by `vio_monitor`) |
| Method | Detect corners (FAST/Shi-Tomasi) in the left image; match to the right along the epipolar row (KLT) → 3D points; track to the next left image (KLT); solve PnP with RANSAC; reject outliers; keyframe when parallax or feature count requires. No IMU, no optimisation window. |
| Frequency | 10–20 Hz |
| Role | Not flight-qualified. Used to understand the pipeline, to validate calibration and frames, and as a last-resort odometry on the bench. Selected by `vio:=opencv`. |
| Failure behaviour | Too few inliers → publishes nothing for that frame; `vio_monitor` handles gaps. |

## 9. `localization_manager`

| Item | Specification |
|---|---|
| Responsibility | (1) Estimate and publish `map → odom` (4-DoF alignment). (2) Compute localisation confidence. (3) Produce the aligned odometry and stream it to the FC as external navigation. |
| Inputs | `/vio/odometry`, `/vio/status`, `/mavros/local_position/pose`, `/mavros/local_position/velocity_local`, `/mavros/global_position/raw/gps_vel`, `/mavros/imu/data`, `/mavros/timesync_status`, `/nav_mode/state`, `/stereo/sync_status` |
| Outputs | TF `map → odom`; `/localization/odometry`; `/localization/status`; `/mavros/odometry/out`; heartbeat; diagnostics |
| Frequency | Odometry at VIO rate, limited to 30 Hz; status 10 Hz; TF 20 Hz |
| QoS | ESTIMATE |
| Alignment | Sliding-window 4-DoF estimate with ring buffer; frozen or refined according to `/nav_mode/state` ([coordinate-frames.md](../02-system-architecture/coordinate-frames.md) §8) |
| Confidence | [state-estimation.md](../09-navigation/state-estimation.md) §8 |
| External-nav output | Pose and twist in `map`/`base_link` with covariance from VIO inflated by alignment uncertainty; `reset_counter`; quality mapped from confidence. Streaming continues in every armed state **except** when VIO is LOST or a reset has not been re-aligned. |
| Key parameters | `align_window_s` (5), `align_freeze_lookback_s` (5), `align_max_res_m` (0.3), `align_max_res_yaw_deg` (3), `extnav_rate_hz` (30), `extnav_min_cov_pos` (0.01 m²), `conf_weights.*`, `conf_feature_ref` (80), `stream_when_disarmed` (true, for bench checks) |
| Failure behaviour | MAVROS pose missing → alignment not refined, `aligned` keeps its last value with an age; after 5 s without FC pose → `aligned = false`. VIO missing → `extnav_streaming = false`. Own crash → external nav stops; FC EKF loses the source and its failsafe or the state machine handles it. |

## 10. `nav_mode_manager`

| Item | Specification |
|---|---|
| Responsibility | GNSS health classification; navigation-mode state machine; EKF source-set commands; speed limit for the navigator; bag start/stop on arm/disarm; FC stream-rate setup |
| Specification | [gps-denied-state-machine.md](../02-system-architecture/gps-denied-state-machine.md) |
| Inputs | `/mavros/state`, `/mavros/gpsstatus/gps1/raw`, `/mavros/estimator_status` (+ variances), `/mavros/rangefinder/rangefinder`, `/mavros/rc/in`, `/mavros/statustext/recv`, `/localization/status`, `/safety/state` |
| Outputs | `/nav_mode/gnss_health`, `/nav_mode/state`, `/events` |
| Services | Server: `/nav_mode/set_override`. Client: `/mavros/cmd/command_int`, `/mavros/set_mode`, `/mavros/set_message_interval`, `/vio/reset`, recorder start/stop |
| Frequency | 5 Hz evaluation |
| QoS | STATE for `/nav_mode/state` |
| Key parameters | All thresholds and dwell times from the state-machine document; `auto_source_select` (true), `auto_return_to_gps` (false initially), `autonomy_rc_channel` (9), `source_cmd_retries` (3), `source_cmd_timeout_s` (1.0) |
| Failure behaviour | Command not acknowledged after retries → event ERROR, state unchanged, warn pilot. Own crash → respawn into BOOT; reads FC state and re-derives a safe state without commanding a source change (the FC keeps its current source). Missing inputs are treated as worst case (GNSS data stale > 2 s = DENIED). |

## 11. `detector`

| Item | Specification |
|---|---|
| Responsibility | Neural-network object detection on the left image |
| Inputs | `/stereo/left/image_color`, `/safety/state` (load-shed level) |
| Outputs | `/perception/detections` (`Detection2DArray`, stamp = image stamp), diagnostics (inference time) |
| Frequency | 5 Hz target; drops frames rather than queueing (queue depth 1) |
| QoS | Input SENSOR; output ESTIMATE |
| Model | YOLO26n, NCNN, 320 × 320 letterboxed, FP32, 2 threads ([ai-architecture.md](../08-ai/ai-architecture.md)) |
| Key parameters | `model_param`, `model_bin`, `input_size` (320), `num_threads` (2), `score_threshold` (0.4), `class_allow_list`, `max_rate_hz` (5), `nms_iou` (only for models that need NMS) |
| Failure behaviour | Model load failure → configure error; system still reaches READY with AI flagged unavailable (AI is advisory). Inference time > 200 ms sustained → reduces rate; reports WARN. Load-shed level ≥ 2 → rate halved; ≥ 3 → deactivated. |

## 12. `object_localizer`

| Item | Specification |
|---|---|
| Responsibility | Attach range and 3D position to detections using the depth image; simple track association |
| Inputs | `/perception/detections`, `/stereo/depth/image` (nearest in time, tolerance 150 ms), `camera_info`, TF `map ← left_camera_optical_frame` at the image stamp |
| Outputs | `/perception/objects` |
| Frequency | 5 Hz |
| Method | [ai-stereo-fusion.md](../08-ai/ai-stereo-fusion.md) |
| Key parameters | `roi_shrink` (0.5), `depth_percentile` (30), `min_valid_pixels` (25), `max_time_diff_ms` (150), `track_gate_m` (1.0), `track_timeout_s` (2.0) |
| Failure behaviour | No depth within tolerance or too few valid pixels → object published with `range_valid = false`. Never fabricates a range. |

## 13. `navigator`

| Item | Specification |
|---|---|
| Responsibility | Convert a goal pose into a stream of position/velocity setpoints for GUIDED mode, respecting speed limits and obstacles |
| Inputs | `/localization/odometry`, `/nav_mode/state`, `/safety/state`, `/obstacle/sectors`, `/mavros/state`, `/mavros/rangefinder/rangefinder` |
| Outputs | `/mavros/setpoint_raw/local` (20 Hz, only while active and allowed), `/navigation/state` |
| Actions | Server: `GoTo` |
| Services | Client: `/mavros/set_mode` (to leave GUIDED), `/mavros/cmd/takeoff` |
| QoS | COMMAND for setpoints |
| Behaviour | [autonomous-navigation.md](../09-navigation/autonomous-navigation.md). Publishes setpoints **only if** FC mode is GUIDED, armed, `setpoints_allowed`, and a goal is active or a hold is commanded. |
| Key parameters | `setpoint_rate_hz` (20), `cruise_speed` (1.0), `max_accel` (1.0), `stop_distance` (2.0), `slow_distance` (4.0), `corridor_half_width_deg` (20), `acceptance_radius` (0.5), `goal_timeout_s` (120), `min_altitude_agl` (1.0), `max_altitude_agl` (8.0), `max_goal_distance` (30) |
| Failure behaviour | Any input stale (odometry > 300 ms, nav-mode > 1 s, safety > 1 s) → hold setpoint (zero velocity), abort goal. Own crash → setpoints stop → FC `GUID_TIMEOUT` stops the vehicle; Lua watchdog covers full companion loss. |

## 14. `mission_manager`

| Item | Specification |
|---|---|
| Responsibility | Execute a mission file as a sequence of steps; the only client of the `GoTo` action in normal operation |
| Inputs | `/nav_mode/state`, `/safety/state`, `/navigation/state`, `/perception/objects`, `/mavros/state` |
| Outputs | `/mission/state`, `/events` |
| Actions | Server: `ExecuteMission`. Client: `GoTo` |
| Mission steps | `TAKEOFF(alt)`, `GOTO(x, y, z, yaw)` in `map` relative to the take-off point, `HOLD(seconds)`, `WAIT_OBJECT(class, timeout)`, `LAND` |
| Rules | Starts only when armed by the pilot, in GUIDED, autonomy enabled, nav mode in {GPS_NAV, VISION_NAV}. Pauses on HOLD-class conditions; aborts on pilot override (does not resume automatically). |
| Failure behaviour | Abort → navigator hold → after `abort_hold_s` (10) request LOITER (or LAND if in a degraded tier). Own crash → navigator goal times out → hold. |

## 15. `safety_supervisor`

| Item | Specification |
|---|---|
| Responsibility | Companion-side watchdog and arbiter: node heartbeats, resource and thermal monitoring, load shedding, autonomy gating, pre-flight check |
| Inputs | Heartbeats of class-A/B nodes, `/diagnostics_agg`, `/system/status`, `/nav_mode/state`, `/localization/status`, `/stereo/sync_status`, `/mavros/state`, `/mavros/battery`, `/mavros/rc/in` |
| Outputs | `/safety/state`, `/events` |
| Services | Server: `/safety/run_preflight`, `/safety/set_autonomy_enabled`. Client: `/mavros/set_mode` (only GUIDED-exit modes), lifecycle manager |
| Frequency | 10 Hz |
| Logic | [safety-architecture.md](../12-safety/safety-architecture.md) §5 |
| Key parameters | `heartbeat_timeout_s` (1.0), `temp_warn_c` (75), `temp_crit_c` (82), `cpu_warn_pct` (85), `mem_warn_pct` (80), `disk_min_gb` (2), `restart_limit` (3 per 60 s), `shed_order` |
| Design constraints | Separate process; no dependency on any project library beyond `gdn_interfaces`; no blocking calls; if it cannot decide, it disallows setpoints. It never arms, never commands motion, never overrides the pilot. |
| Failure behaviour | Its own heartbeat is watched by `navigator` and `nav_mode_manager`: if `/safety/state` is stale > 1 s, both go to hold. |

## 16. `telemetry_node`

| Item | Specification |
|---|---|
| Responsibility | Summarise companion state for the GCS over MAVLink |
| Inputs | `/nav_mode/state`, `/localization/status`, `/safety/state`, `/mission/state`, `/events`, `/perception/objects` |
| Outputs | `/mavros/statustext/send` (on change and on events; rate-limited to 1 message/s), `/mavros/debug_value/send` (`NAMED_VALUE_FLOAT` at 1 Hz: `loc_conf`, `nav_mode`, `vio_feat`, `skew_ms`, `cc_temp`, `obst_m`) |
| Failure behaviour | None safety-relevant |

## 17. `hud_node`

| Item | Specification |
|---|---|
| Responsibility | Draw the pilot-facing video: left image with detections, ranges, navigation mode, confidence bar, nearest obstacle, battery; output to HDMI |
| Inputs | `/stereo/left/image_color`, `/perception/objects`, `/nav_mode/state`, `/localization/status`, `/obstacle/sectors`, `/mavros/battery`, `/safety/state` |
| Outputs | HDMI (KMS/DRM, 1280×720 at 15 Hz); no ROS outputs |
| Failure behaviour | First node shed under load. Crash → blank video; no other effect. |

## 18. `system_monitor`

| Item | Specification |
|---|---|
| Responsibility | Read SoC temperature, throttle/under-voltage flags, CPU, memory, disk, fan |
| Outputs | `/system/status` (1 Hz), `/diagnostics` |
| Sources | `/sys/class/thermal`, `/proc/stat`, `/proc/meminfo`, firmware throttle status, `statvfs` |

## 19. `lifecycle_manager`

| Item | Specification |
|---|---|
| Responsibility | Ordered configure/activate/deactivate of managed nodes; restart on error; report to supervisor |
| Services | `/lifecycle_manager/startup`, `/shutdown` |
| Failure behaviour | A node failing to activate within its timeout → NO-GO with the node name |

## 20. `mavros`

| Item | Specification |
|---|---|
| Responsibility | MAVLink 2 link to the FC; frame conversion; time sync |
| Configuration | `fcu_url: /dev/ttyAMA0:921600` (hardware) or `udp://127.0.0.1:14550@` (SITL); `tgt_system: 1`; companion system/component ID 1 / 191 (`MAV_COMP_ID_ONBOARD_COMPUTER`) `[VERIFY ID convention with the FC watchdog script]`; plugin allow-list per [interfaces.md](interfaces.md) §5; `local_position.tf.send: false`; `odometry.fcu.odom_parent_id_des: map`, `odom_child_id_des: base_link` |
| Failure behaviour | Serial loss → `connected = false`; all consumers hold; FC watchdog acts. Respawned by launch. |

## 21–22. Standard nodes

`robot_state_publisher` publishes static transforms from the URDF. `diagnostic_aggregator` groups diagnostics per its YAML configuration.

---

## DB-2.0 additions

### 23. `down_camera`

| Item | Specification |
|---|---|
| Responsibility | Publish downward images from the USB camera |
| Implementation | Standard UVC node (`usb_cam` or `v4l2_camera`), configured in `gdn_geoloc`; class A in the cruise regime |
| Outputs | `/down/image_raw` (mono8, 640×480, 15 Hz), `/down/camera_info` |
| Key parameters | `device`, `width`, `height`, `framerate`, `pixel_format`, `exposure_absolute` (manual), `gain`, `camera_info_url` |
| Failure behaviour | Device lost → node exits, respawned; `ground_vo` and `map_matcher` report stale input; state machine degrades (T20/T22) |

### 24. `ground_vo`

| Item | Specification |
|---|---|
| Package / language / class | `gdn_geoloc` / C++ / A (cruise regime), lifecycle |
| Responsibility | Relative horizontal motion from consecutive downward images |
| Inputs | `/down/image_raw`, `/down/camera_info`, `/mavros/imu/data` (attitude, rates), height above ground (`/mavros/local_position/pose` z relative to take-off; rangefinder when valid) |
| Outputs | `/ground_vo/odometry` (`nav_msgs/Odometry`, `odom` → `base_link`, velocity and integrated position, covariance), diagnostics, heartbeat |
| Frequency | 15 Hz |
| Method | FAST + KLT tracking; rotation compensation from FC gyro; RANSAC similarity; metres = height × pixels / focal length ([visual-geolocalization.md](../09-navigation/visual-geolocalization.md) §6) |
| Key parameters | `max_features` (150), `min_inliers` (25), `min_height` (10), `ransac_thresh_px` (2.0), `drift_fraction` (0.03, for covariance) |
| Failure behaviour | Too few inliers → that frame is skipped and covariance inflated; more than 1 s without a valid estimate → output stops, `vio_monitor` reports LOST |

### 25. `map_matcher`

| Item | Specification |
|---|---|
| Package / language / class | `gdn_geoloc` / C++ / A (cruise regime), lifecycle |
| Responsibility | Absolute position by matching the downward image against the onboard satellite map pack |
| Inputs | `/down/image_raw`, `/down/camera_info`, `/mavros/imu/data`, height above ground, `/localization/odometry` (predicted position and uncertainty), `/mavros/global_position/gp_origin` (EKF origin), `/mavros/global_position/global` (for shadow-mode comparison), map pack on disk |
| Outputs | `/geoloc/fix` (`GeoFix`, every attempt, accepted or rejected), `/geoloc/status` (`GeoLocStatus`, 2 Hz), `/geoloc/debug_image` (bench only), diagnostics, heartbeat |
| Services | `/geoloc/set_start_position` (operator-confirmed start for a cold start without GNSS), `/geoloc/reload_map` |
| Frequency | 1 Hz attempts (parameter); newest image only |
| Method | Undistort → orthorectify (level, north-up, map resolution) → CLAHE → features → window from prediction → match against precomputed map features → RANSAC similarity → gates ([visual-geolocalization.md](../09-navigation/visual-geolocalization.md) §5) |
| Key parameters | `map_pack_path`, `method` (sift, xfeat, ncc), `rate_hz` (1.0), `match_min_height` (35), `max_tilt_deg` (15), `max_features` (500), `ratio` (0.78), `min_inliers` (12), `min_inlier_ratio` (0.25), `max_rot_dev_deg` (15), `max_scale_dev` (0.20), `window_min_m` (40), `window_max_m` (200), `gate_sigma` (3.0), `edge_margin_m` (60), `shadow_compare` (true) |
| Failure behaviour | No map pack or wrong CRS → configure error, NO-GO for cruise-regime GPS-denied flight. Match rejected → published with the reason; never forwarded as a position. Processing slower than the period → skips cycles, reports WARN. Outside coverage → `OUT_OF_COVERAGE` |

### Changes to existing nodes

| Node | Change in DB-2.0 |
|---|---|
| `vio_monitor` | Selects the relative-odometry source by height: `/ground_vo/odometry` above `odom_switch_height` (12 m, ± 2 m hysteresis), stereo VIO below. Applies the same health checks to either. At a source change it carries the pose across so that `/vio/odometry` has no step. |
| `localization_manager` | Subscribes to `/geoloc/fix`. In `VISION_NAV` (cruise) it updates `map → odom` from accepted fixes with an offset filter and slew limit; publishes `pos_sigma_m`, `fix_age_s`, geo-localisation sub-scores. New parameters: `fix_offset_process_noise` (drift fraction), `fix_slew_mps` (0.5), `fix_max_step_m` (15). |
| `nav_mode_manager` | Regime selection by height; lifecycle activation of stereo vs geo-localisation nodes; transitions T19–T23; coverage check for goals. |
| `navigator` | Altitude limits become regime-dependent (`max_altitude_agl` 60 m in cruise profile); refuses goals outside map coverage; cruise speed limit 3 m/s. |
| `safety_supervisor` | Pre-flight checks add: map pack present and matches the site; downward camera rate and exposure; shadow-mode fix statistics when available. |
| `hud_node` / `telemetry_node` | Show geo-localisation state, fix age, position uncertainty; named values `fix_age`, `pos_sig`, `geo_inl`. |
| `open_vins`, `stereo_depth`, `obstacle_sectors` | Deactivated above 12 m to free CPU. |

## DB-3.0 additions

### 26. `app_gateway`

| Item | Specification |
|---|---|
| Package / language / class | `gdn_app_gateway` / Python (asyncio) / C |
| Responsibility | Drone side of the Android app's protocol ([ground-app.md](../10-communication/ground-app.md)): WebSocket server, request validation and gating, translation to ROS 2 actions/services, status and findings out, read-only map tile server |
| Inputs | `/nav_mode/state`, `/localization/status`, `/geoloc/status`, `/safety/state`, `/mission/state`, `/navigation/state`, `/mavros/state`, `/mavros/battery`, `/perception/detections`, `/perception/aerial_detections`, `/tracking/target`, `/search/status`, `/search/findings`, `/events` |
| Outputs | `/app/link_status` (connected, last heartbeat age, RTT); protocol messages to the app |
| Actions / services (client) | `SearchArea`, `FollowTarget`, `GoTo`; `/tracking/select`, `/tracking/clear`; `/safety/run_preflight`; `/geoloc/set_start_position`; `/mission/hold` |
| Frequency | Status 2 Hz; detections up to 5 Hz; events as they occur |
| Key parameters | `bind_address` (192.168.144.50), `control_port` (8765), `tile_port` (8767), `shared_key`, `max_clients` (1), `heartbeat_timeout_s` (3), `max_search_area_ha` (5), `rate_limit_per_s` (10) |
| Rules | Never calls MAVROS. Never arms or changes mode. Refuses anything not allowed by `/safety/state` and `/nav_mode/state`, with a reason. Validates every number and polygon |
| Failure behaviour | Crash or link loss → `/app/link_status` stale → `mission_manager` applies the link-loss rule for the active behaviour. Respawned by launch. Flight unaffected |

### 27. `video_streamer`

| Item | Specification |
|---|---|
| Package / language / class | `gdn_app_gateway` / C++ / C |
| Responsibility | Send the selected camera view to the app as JPEG frames with frame id and stamp |
| Inputs | `/stereo/left/image_color` or `/down/image_raw` (selected by profile or by the app) |
| Outputs | WebSocket binary frames on port 8766; `/app/video_stats` |
| Key parameters | `width` (640), `height` (480), `fps` (10), `jpeg_quality` (60) |
| Failure behaviour | Sheds first under load (frame rate halves at shed level 1, stops at level 3). No effect on flight |

### 28. `target_tracker`

| Item | Specification |
|---|---|
| Package / language / class | `gdn_tracking` / C++ / B (when follow is active) |
| Responsibility | Track one operator-selected object: ground position and velocity ([search-track-follow.md](../09-navigation/search-track-follow.md) §6) |
| Inputs | `/perception/detections` (front) or `/perception/aerial_detections` (down), `/localization/odometry`, TF, height |
| Outputs | `/tracking/target` (`TrackedTarget`, 5 Hz while active) |
| Services | `/tracking/select` (frame id + normalised point, or detection id), `/tracking/clear` |
| Key parameters | `select_radius_px` (40), `gate_base_m` (3), `gate_growth_mps` (3), `lost_after_s` (3), `end_after_s` (15), `process_noise` |
| Failure behaviour | No association → LOST → track ends after `end_after_s`, last position reported. Crash → follow holds |

### 29. `search_planner` and 30. `finding_manager` (in `gdn_mission`)

| Item | `search_planner` | `finding_manager` |
|---|---|---|
| Language / class | Python / B | Python / C |
| Responsibility | Validate the area; generate the back-and-forth pattern; sequence the lines through `GoTo`; publish progress and recorded coverage | Project detections to the ground; confirm over frames; merge duplicates; store evidence; publish findings |
| Action | Server: `SearchArea` | — |
| Inputs | Polygon and settings; `/localization/odometry`; `/nav_mode/state`; `/safety/state`; `/geoloc/status` (coverage bounds) | `/perception/aerial_detections`; `/localization/odometry`; attitude; height; image for thumbnails |
| Outputs | `/search/status`, `/search/coverage` | `/search/findings`, files under the run directory |
| Key parameters | `overlap` (0.3), `speed` (3.0), `height` (25), `max_area_ha` (5), `edge_margin_m` (60), `reserve_battery_pct` (40) | `confirm_frames` (3), `confirm_radius_m` (3), `merge_radius_m` (4), `min_score` (0.35) |
| Failure behaviour | Pauses on degraded localisation; aborts on pilot override or lost localisation; refuses invalid areas with a reason | Stops raising findings if position uncertainty > 10 m; never invents a position |

### Changes to existing nodes (DB-3.0)

| Node | Change |
|---|---|
| `detector` | Two models: ground-view (forward camera) and aerial-view (downward camera, tiled). One is active at a time, selected by flight profile. New output `/perception/aerial_detections` |
| `down_camera` | Delivers ≥ 1920×1080 for detection and a down-scaled 640×480 stream for `ground_vo` and `map_matcher` |
| `navigator` | New action server `FollowTarget`; follow-from-above controller; search-profile limits |
| `mission_manager` | Behaviours `SEARCH`, `FOLLOW`, `GOTO_FINDING`; app-link-loss rules; `/mission/hold` service |
| `nav_mode_manager` | Third flight profile **search** (25–30 m); `match_min_height` computed from map pack resolution; selects detector input and model |
| `map_matcher` | Publishes map pack resolution and the derived minimum matching height |
| `safety_supervisor` | Pre-flight adds: app link (advisory), aerial model loaded, search profile limits; gate for follow (never below 20 m) |
| `telemetry_node` | Unchanged role (status to QGroundControl); events are mirrored to the app by `app_gateway` |
| `hud_node` | **Removed.** The app draws the overlay |

## Rate summary

| Stream | Rate |
|---|---|
| IMU | 225 Hz |
| Images | 20 Hz |
| VIO odometry | 20 Hz |
| External nav to FC | 20–30 Hz |
| Setpoints | 20 Hz |
| Depth, obstacle sectors | 10 Hz |
| Detections, objects | 5 Hz |
| State machine | 5 Hz |
| Supervisor | 10 Hz |
| Heartbeats | 5 Hz |
| Diagnostics, system status | 1 Hz |
| Downward images, ground VO (DB-2.0) | 15 Hz |
| Map-match attempts (DB-2.0) | 1 Hz |
