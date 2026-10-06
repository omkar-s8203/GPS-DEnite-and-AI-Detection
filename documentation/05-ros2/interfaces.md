# ROS 2 Interfaces — Topics, Services, Actions, Messages

| Field | Value |
|---|---|
| Document ID | GDN-ROS-003 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | Baseline |

QoS profile names refer to [ros2-architecture.md](ros2-architecture.md) §5. Frame IDs follow [coordinate-frames.md](../02-system-architecture/coordinate-frames.md).

## 1. Topics

### 1.1 Sensors

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/stereo/left/image_raw` | `sensor_msgs/Image` (mono8, 640×480) | `stereo_camera` | `open_vins`, `rectify_left`, recorder | 20 Hz | SENSOR |
| `/stereo/right/image_raw` | `sensor_msgs/Image` (mono8) | `stereo_camera` | `open_vins`, `rectify_right`, recorder | 20 Hz | SENSOR |
| `/stereo/left/camera_info` | `sensor_msgs/CameraInfo` | `stereo_camera` | rectification, depth, object localiser | 20 Hz | SENSOR |
| `/stereo/right/camera_info` | `sensor_msgs/CameraInfo` | `stereo_camera` | rectification, depth | 20 Hz | SENSOR |
| `/stereo/left/image_color` | `sensor_msgs/Image` (bgr8, 320×240) | `stereo_camera` | `detector`, `hud_node` | 10 Hz | SENSOR |
| `/stereo/sync_status` | `gdn_interfaces/StereoSyncStatus` | `stereo_camera` | `safety_supervisor`, `vio_monitor`, recorder | 20 Hz | SENSOR |
| `/imu/data_raw` | `sensor_msgs/Imu` (no orientation) | `imu_driver` | `open_vins`, recorder | 200–225 Hz | SENSOR |

### 1.2 Stereo and obstacles

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/stereo/left/image_rect` | `sensor_msgs/Image` | `rectify_left` | `stereo_depth` | 10 Hz | SENSOR (intra-process) |
| `/stereo/right/image_rect` | `sensor_msgs/Image` | `rectify_right` | `stereo_depth` | 10 Hz | SENSOR (intra-process) |
| `/stereo/disparity` | `stereo_msgs/DisparityImage` | `stereo_depth` | recorder (optional) | 10 Hz | SENSOR |
| `/stereo/depth/image` | `sensor_msgs/Image` (32FC1, metres, NaN invalid) | `stereo_depth` | `obstacle_sectors`, `object_localizer` | 10 Hz | SENSOR |
| `/stereo/depth/camera_info` | `sensor_msgs/CameraInfo` | `stereo_depth` | same | 10 Hz | SENSOR |
| `/obstacle/sectors` | `gdn_interfaces/ObstacleSectors` | `obstacle_sectors` | `navigator`, `hud_node` | 10 Hz | ESTIMATE |
| `/mavros/obstacle/send` | `sensor_msgs/LaserScan` (72 bins, 5°) | `obstacle_sectors` | `mavros` | 10 Hz | SENSOR |

### 1.3 Estimation

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/ov_msckf/odomimu` | `nav_msgs/Odometry` | `open_vins` | `vio_monitor` | 20 Hz | ESTIMATE |
| `/ov_msckf/trackhist` (debug) | `sensor_msgs/Image` | `open_vins` | bench only | — | SENSOR |
| `/vio/odometry` | `nav_msgs/Odometry` (`odom` → `base_link`) | `vio_monitor` | `localization_manager`, recorder | 20–30 Hz | ESTIMATE |
| `/vio/status` | `gdn_interfaces/VioStatus` | `vio_monitor` | `localization_manager`, `safety_supervisor` | 10 Hz | ESTIMATE |
| `/localization/odometry` | `nav_msgs/Odometry` (`map` → `base_link`) | `localization_manager` | `navigator`, `object_localizer`, recorder | 20–30 Hz | ESTIMATE |
| `/localization/status` | `gdn_interfaces/LocalizationStatus` | `localization_manager` | `nav_mode_manager`, `navigator`, `telemetry_node`, `hud_node` | 10 Hz | ESTIMATE |
| `/mavros/odometry/out` | `nav_msgs/Odometry` | `localization_manager` | `mavros` (→ MAVLink `ODOMETRY`) | 20–30 Hz | ESTIMATE |
| `/tf` | `tf2_msgs/TFMessage` | `vio_monitor`, `localization_manager` | all | 20–30 Hz | default TF |
| `/tf_static` | `tf2_msgs/TFMessage` | `robot_state_publisher`, `mavros` | all | latched | default static TF |

### 1.3a Visual geo-localisation (DB-2.0)

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/down/image_raw` | `sensor_msgs/Image` (mono8, 640×480) | `down_camera` | `ground_vo`, `map_matcher`, recorder | 15 Hz | SENSOR |
| `/down/camera_info` | `sensor_msgs/CameraInfo` | `down_camera` | `ground_vo`, `map_matcher` | 15 Hz | SENSOR |
| `/ground_vo/odometry` | `nav_msgs/Odometry` | `ground_vo` | `vio_monitor` | 15 Hz | ESTIMATE |
| `/geoloc/fix` | `gdn_interfaces/GeoFix` | `map_matcher` | `localization_manager`, recorder | ≈ 1 Hz | ESTIMATE |
| `/geoloc/status` | `gdn_interfaces/GeoLocStatus` | `map_matcher` | `nav_mode_manager`, `localization_manager`, `telemetry_node`, `hud_node` | 2 Hz | ESTIMATE |
| `/geoloc/debug_image` | `sensor_msgs/Image` | `map_matcher` | bench only | ≈ 1 Hz | SENSOR |

Additional MAVROS topics consumed: `/mavros/global_position/gp_origin` (EKF origin, for geodetic conversion) and `/mavros/global_position/global` (GNSS position for shadow-mode comparison).

Services: `/geoloc/set_start_position` (`gdn_interfaces/srv/SetStartPosition`: latitude, longitude, radius) and `/geoloc/reload_map` (`std_srvs/srv/Trigger`), both served by `map_matcher`.

Custom messages added:

```text
# GeoFix
std_msgs/Header header            # stamp = image acquisition time; frame: map
bool accepted
uint8 REJ_NONE=0  REJ_FEW_INLIERS=1  REJ_RATIO=2  REJ_GEOMETRY=3
uint8 REJ_GATE=4  REJ_INCONSISTENT=5  REJ_TILT=6  REJ_HEIGHT=7  REJ_COVERAGE=8
uint8 reject_reason
float64 latitude                  # WGS-84
float64 longitude
geometry_msgs/Point position_map  # base_link in map (ENU), z unused
float32[4] covariance             # east/north 2x2, row-major, m^2
uint16 inliers
float32 inlier_ratio
float32 rotation_deg              # estimated residual rotation
float32 scale                     # estimated scale relative to expected
float32 window_m                  # half-size of the search window used
float32 latency_ms
float32 gnss_error_m              # shadow mode: distance to GNSS position; NaN if unavailable
string map_id

# GeoLocStatus
std_msgs/Header header
uint8 INACTIVE=0  SEARCHING=1  TRACKING=2  COASTING=3  LOST=4  OUT_OF_COVERAGE=5
uint8 state
float32 fix_age_s
float32 accept_ratio              # accepted / attempted, last 30 s
uint16 median_inliers
float32 distance_to_edge_m
string map_id
string method                     # sift, xfeat, ncc
```

`LocalizationStatus` gains: `float32 pos_sigma_m`, `float32 fix_age_s`, `uint8 position_submode` (GEO=1, ODOM=0), `uint8 odometry_source` (STEREO_VIO=0, GROUND_VO=1).

### 1.3b Ground app, search, track and follow (DB-3.0)

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/perception/aerial_detections` | `vision_msgs/Detection2DArray` (downward camera, full-frame pixel coordinates) | `detector` | `finding_manager`, `target_tracker`, `app_gateway` | ≈ 1 Hz | ESTIMATE |
| `/tracking/target` | `gdn_interfaces/TrackedTarget` | `target_tracker` | `navigator`, `app_gateway`, recorder | 5 Hz while active | ESTIMATE |
| `/search/status` | `gdn_interfaces/SearchStatus` | `search_planner` | `app_gateway`, `telemetry_node` | 1 Hz | STATE |
| `/search/coverage` | `nav_msgs/OccupancyGrid` (covered cells, `map` frame, 2 m cells) | `search_planner` | `app_gateway`, recorder | 0.5 Hz | STATE |
| `/search/findings` | `gdn_interfaces/FindingArray` | `finding_manager` | `app_gateway`, `mission_manager`, recorder | On change | STATE |
| `/app/link_status` | `gdn_interfaces/AppLinkStatus` | `app_gateway` | `mission_manager`, `safety_supervisor` | 1 Hz | STATE |

Services: `/tracking/select`, `/tracking/clear` (`target_tracker`); `/mission/hold` (`mission_manager`, `std_srvs/Trigger`).

Actions:

```text
# SearchArea — server: search_planner
geometry_msgs/Polygon area_map        # vertices in map (converted from lat/lon by app_gateway)
float32 height                         # m AGL
float32 overlap                        # 0..0.6
float32 speed                          # m/s
string[] classes
---
uint8 COMPLETED=0  ABORTED_PILOT=1  ABORTED_LOCALIZATION=2  ABORTED_BATTERY=3  CANCELED=4  REFUSED=5
uint8 result
string message
float32 area_covered_m2
uint16 findings
---
float32 progress                       # 0..1
uint16 line_index
uint16 lines_total
float32 time_remaining_s

# FollowTarget — server: navigator
uint32 track_id
float32 height                         # 0 = keep current
---
uint8 ENDED_BY_OPERATOR=0  TARGET_LOST=1  BOUNDARY=2  ABORTED_PILOT=3  ABORTED_LOCALIZATION=4
uint8 result
geometry_msgs/Point last_target_position
---
float32 offset_m                       # horizontal distance target - point below drone
float32 target_speed
bool target_visible
```

Messages:

```text
# TrackedTarget
std_msgs/Header header                 # frame: map
uint32 track_id
uint8 TRACKING=1  LOST=2  ENDED=3
uint8 state
string class_name
geometry_msgs/Point position_map
geometry_msgs/Vector3 velocity_map
float32 position_sigma_m
vision_msgs/BoundingBox2D bbox
uint8 camera                           # 0 front, 1 down
float32 time_since_seen_s

# Finding / FindingArray
uint32 id
builtin_interfaces/Time stamp
string class_name
float32 score
float64 latitude
float64 longitude
geometry_msgs/Point position_map
float32 position_sigma_m
uint8 confirming_frames
uint8 UNREVIEWED=0  CONFIRMED=1  REJECTED=2
uint8 review
string thumbnail_path

# SearchStatus: state (IDLE, PLANNED, RUNNING, PAUSED, DONE, ABORTED), progress, line_index,
#               lines_total, area_requested_m2, area_covered_m2, time_remaining_s, pause_reason
# AppLinkStatus: connected, heartbeat_age_s, round_trip_ms, client_id
```

The app's own protocol (JSON over WebSocket) is specified in [ground-app.md](../10-communication/ground-app.md) §6. It is not a ROS interface.

### 1.4 Navigation mode and decision

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/nav_mode/gnss_health` | `gdn_interfaces/GnssHealth` | `nav_mode_manager` | `telemetry_node`, recorder | 5 Hz | ESTIMATE |
| `/nav_mode/state` | `gdn_interfaces/NavModeState` | `nav_mode_manager` | `navigator`, `mission_manager`, `safety_supervisor`, `telemetry_node`, `hud_node` | On change + 2 Hz | STATE |
| `/navigation/state` | `gdn_interfaces/NavigatorState` | `navigator` | `mission_manager`, `telemetry_node` | 5 Hz | STATE |
| `/mavros/setpoint_raw/local` | `mavros_msgs/PositionTarget` | `navigator` | `mavros` | 20 Hz while autonomous | COMMAND |
| `/mission/state` | `gdn_interfaces/MissionState` | `mission_manager` | `telemetry_node`, `hud_node` | On change + 1 Hz | STATE |
| `/safety/state` | `gdn_interfaces/SafetyState` | `safety_supervisor` | `navigator`, `nav_mode_manager`, `mission_manager`, `detector`, `hud_node`, `telemetry_node` | On change + 5 Hz | STATE |
| `/events` | `gdn_interfaces/Event` | any | recorder, `telemetry_node` | Event | EVENT |

### 1.5 Perception

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/perception/detections` | `vision_msgs/Detection2DArray` | `detector` | `object_localizer`, `hud_node` | 5 Hz | ESTIMATE |
| `/perception/objects` | `gdn_interfaces/TrackedObjectArray` | `object_localizer` | `mission_manager`, `navigator`, `hud_node`, `telemetry_node` | 5 Hz | ESTIMATE |

### 1.6 System

| Topic | Type | Publisher | Subscribers | Rate | QoS |
|---|---|---|---|---|---|
| `/system/status` | `gdn_interfaces/SystemStatus` | `system_monitor` | `safety_supervisor`, `telemetry_node` | 1 Hz | ESTIMATE |
| `/diagnostics` | `diagnostic_msgs/DiagnosticArray` | all | `diagnostic_aggregator` | 1 Hz | DIAG |
| `/diagnostics_agg` | `diagnostic_msgs/DiagnosticArray` | `diagnostic_aggregator` | `safety_supervisor`, recorder | 1 Hz | DIAG |
| `/<node>/heartbeat` | `std_msgs/Header` | each class-A/B node | `safety_supervisor` | 5 Hz | HEARTBEAT |

### 1.7 MAVROS topics consumed

| Topic | Type | Used by | Purpose |
|---|---|---|---|
| `/mavros/state` | `mavros_msgs/State` | `nav_mode_manager`, `navigator`, `safety_supervisor`, `mission_manager` | Connected, armed, mode |
| `/mavros/local_position/pose` | `geometry_msgs/PoseStamped` | `localization_manager` | EKF pose (alignment) |
| `/mavros/local_position/velocity_local` | `geometry_msgs/TwistStamped` | `localization_manager` | EKF velocity |
| `/mavros/local_position/odom` | `nav_msgs/Odometry` | `vio_monitor` (fallback `odom → base_link`), recorder | EKF odometry |
| `/mavros/gpsstatus/gps1/raw` | `mavros_msgs/GPSRAW` | `nav_mode_manager` | Fix, satellites, HDOP, accuracy |
| `/mavros/global_position/raw/gps_vel` | `geometry_msgs/TwistStamped` | `localization_manager` | GNSS velocity for the consistency check |
| `/mavros/estimator_status` | `mavros_msgs/EstimatorStatus` | `nav_mode_manager` | EKF flags |
| `/mavros/imu/data` | `sensor_msgs/Imu` | `localization_manager` | FC attitude and rates for cross-checks |
| `/mavros/rangefinder/rangefinder` | `sensor_msgs/Range` | `nav_mode_manager`, `navigator` | Height above ground, flow validity |
| `/mavros/battery` | `sensor_msgs/BatteryState` | `safety_supervisor`, `hud_node` | Display, logging |
| `/mavros/rc/in` | `mavros_msgs/RCIn` | `nav_mode_manager`, `safety_supervisor` | Autonomy-enable switch, source-switch detection |
| `/mavros/statustext/recv` | `mavros_msgs/StatusText` | `nav_mode_manager`, recorder | EKF source-change messages, FC warnings |
| `/mavros/timesync_status` | `mavros_msgs/TimesyncStatus` | `localization_manager` | Clock offset health |
| `/mavros/statustext/send` | `mavros_msgs/StatusText` | from `telemetry_node` | Operator messages |
| `/mavros/debug_value/send` | `mavros_msgs/DebugValue` | from `telemetry_node` | `NAMED_VALUE_FLOAT` values |

EKF variance values (`EKF_STATUS_REPORT`) are needed by the GNSS classifier. Whether the MAVROS `estimator_status` plugin exposes ArduPilot's variance fields must be confirmed `[VERIFY]`; if not, a minimal additional MAVROS plugin or a raw-MAVLink subscription (`/uas1/mavlink_source`) is used for that single message.

## 2. Services

| Service | Type | Server | Clients | Purpose |
|---|---|---|---|---|
| `/nav_mode/set_override` | `gdn_interfaces/srv/SetNavModeOverride` | `nav_mode_manager` | operator tools, tests | Force or release a source set; enable/disable automatic selection. Rejected in flight unless from the RC-mapped path. |
| `/vio/reset` | `gdn_interfaces/srv/ResetVio` | `vio_monitor` | `nav_mode_manager`, operator | Restart the estimator (kills and respawns `open_vins`), increments reset counter |
| `/safety/run_preflight` | `gdn_interfaces/srv/RunPreflightCheck` | `safety_supervisor` | `telemetry_node`, operator | Returns GO/NO-GO with reasons |
| `/safety/set_autonomy_enabled` | `gdn_interfaces/srv/SetAutonomyEnabled` | `safety_supervisor` | `nav_mode_manager` (from RC switch), operator | Master enable for companion setpoints |
| `/lifecycle_manager/startup`, `/shutdown` | `std_srvs/srv/Trigger` | lifecycle manager | systemd wrapper, operator | Ordered bring-up / tear-down |
| `/recorder/start`, `/recorder/stop` | `std_srvs/srv/Trigger` | recorder wrapper in `gdn_bringup` | `nav_mode_manager` (on arm/disarm) | Bag control |
| `/<node>/get_state`, `/change_state` | `lifecycle_msgs` | lifecycle nodes | lifecycle manager | Standard |

MAVROS services called:

| Service | Type | Caller | Purpose |
|---|---|---|---|
| `/mavros/cmd/command_int` | `mavros_msgs/srv/CommandInt` | `nav_mode_manager` | `MAV_CMD_SET_EKF_SOURCE_SET` (42007), param1 = 1/2/3 |
| `/mavros/set_mode` | `mavros_msgs/srv/SetMode` | `navigator`, `nav_mode_manager`, `safety_supervisor` | **Only** to leave GUIDED: BRAKE, LOITER, ALT_HOLD, LAND, RTL |
| `/mavros/cmd/takeoff` | `mavros_msgs/srv/CommandTOL` | `navigator` | Take-off in GUIDED after the pilot has armed (later test phases only) |
| `/mavros/set_message_interval` | `mavros_msgs/srv/MessageInterval` | `nav_mode_manager` at start-up | Request stream rates |
| `/mavros/global_position/set_gp_origin` | topic/service per MAVROS version | `nav_mode_manager` | `SET_GPS_GLOBAL_ORIGIN` for indoor start |

**Never called by any project node:** `/mavros/cmd/arming`.

## 3. Actions

### `GoTo` — server: `navigator`

```text
# Goal
geometry_msgs/PoseStamped target      # frame: map (ENU). Yaw from orientation.
float32 max_speed                     # m/s; clamped by nav-mode limit
float32 acceptance_radius             # m (default 0.5)
bool hold_yaw                         # keep current yaw if true
---
# Result
uint8 SUCCEEDED=0  ABORTED_OBSTACLE=1  ABORTED_LOCALIZATION=2
uint8 ABORTED_OVERRIDE=3  ABORTED_TIMEOUT=4  CANCELED=5
uint8 result
float32 final_distance
---
# Feedback
float32 distance_remaining
float32 commanded_speed
float32 nearest_obstacle
uint8 limiting_factor                 # none, confidence, obstacle, nav_mode
```

### `ExecuteMission` — server: `mission_manager`

```text
# Goal
string mission_file                   # name within gdn_mission/missions
---
# Result
bool success
uint16 steps_completed
string message
---
# Feedback
uint16 current_step
string step_type                      # TAKEOFF, GOTO, HOLD, LAND, WAIT_OBJECT
string status
```

## 4. Custom messages (`gdn_interfaces/msg`)

Field lists are the design contract; exact definitions are written at implementation.

### `StereoSyncStatus`
```text
std_msgs/Header header
float32 skew_ms              # right stamp - left stamp for this pair
float32 skew_mean_ms         # over last 100 pairs
float32 skew_max_ms
uint32 pairs_published
uint32 pairs_dropped         # unmatched or over threshold
float32 exposure_ms
float32 analog_gain
bool synchronized            # |skew| <= threshold
```

### `VioStatus`
```text
std_msgs/Header header
uint8 UNINITIALIZED=0  INITIALIZING=1  TRACKING=2  DEGRADED=3  LOST=4
uint8 state
uint16 tracked_features
float32 position_cov_trace   # m^2
float32 output_rate_hz
float32 latency_ms           # image stamp -> odometry publish
uint16 reset_counter
float32 time_offset_ms       # estimated camera-IMU offset
float32 speed                # m/s
float32 yaw_rate             # rad/s
```

### `LocalizationStatus`
```text
std_msgs/Header header
float32 confidence           # 0..1
uint8 HIGH=3  MEDIUM=2  LOW=1  LOST=0
uint8 level
bool vio_ok
bool aligned
float32 align_residual_m
float32 align_residual_yaw_deg
float32 gnss_vio_velocity_diff   # m/s (NaN if unavailable)
float32 baro_vio_alt_diff        # m
float32 extnav_rate_hz
float32 timesync_offset_ms
uint16 reset_counter
bool extnav_streaming
```

### `GnssHealth`
```text
std_msgs/Header header
uint8 GOOD=2  DEGRADED=1  DENIED=0
uint8 classification
uint8 fix_type
uint8 satellites
float32 hdop
float32 h_acc_m
float32 ekf_pos_variance
float32 time_in_class_s
string reason                # which condition is limiting
```

### `NavModeState`
```text
std_msgs/Header header
uint8 BOOT=0  SENSOR_CHECK=1  READY=2  GPS_NAV=3  GPS_DEGRADED=4
uint8 VISION_NAV=5  VISION_DEGRADED=6  FLOW_FALLBACK=7  GPS_RECOVERY=8
uint8 LOCALIZATION_LOST=9  FAULT=10
uint8 state
uint8 previous_state
uint8 ekf_source_set         # 1..3 as last confirmed
bool manual_override         # FC mode != GUIDED
bool auto_source_select
float32 speed_limit          # m/s for the navigator
string reason
```

### `ObstacleSectors`
```text
std_msgs/Header header       # frame: base_link
float32 angle_min            # rad
float32 angle_increment      # rad
float32[] ranges             # m; inf = clear; nan = unknown
float32 nearest_range
float32 nearest_bearing
float32 valid_fraction       # fraction of depth pixels valid in the band
```

### `TrackedObject` / `TrackedObjectArray`
```text
std_msgs/Header header       # frame: map
uint32 track_id
string class_name
float32 score
geometry_msgs/Point position_map
geometry_msgs/Point position_camera
float32 range_m
float32 range_std_m
bool range_valid
vision_msgs/BoundingBox2D bbox
```

### `SafetyState`
```text
std_msgs/Header header
uint8 NOMINAL=0  DEGRADED=1  HOLD=2  ABORT=3  FAULT=4
uint8 level
bool autonomy_enabled
bool setpoints_allowed
uint8 load_shed_level        # 0 none .. 4 max
string[] active_faults
```

### `SystemStatus`
```text
std_msgs/Header header
float32 soc_temp_c
bool throttled_now
bool undervoltage_now
bool throttled_since_boot
float32[4] cpu_percent
float32 mem_used_mb
float32 disk_free_gb
float32 load_avg_1m
```

### `Event`
```text
std_msgs/Header header
string source                # node name
uint8 INFO=0  WARN=1  ERROR=2
uint8 severity
string code                  # e.g. NAVMODE_TRANSITION, EKF_SRC_SET, VIO_RESET
string message
string[] keys
float64[] values             # numeric context that triggered the event
```

`NavigatorState` and `MissionState` carry a state enum, the active goal and a reason string.

## 5. MAVROS plugin allow-list

Only these plugins are loaded (reduces CPU and serial traffic):

| Plugin | Why |
|---|---|
| `sys_status`, `sys_time` | Heartbeat, state, time sync |
| `command` | EKF source set, take-off, message intervals |
| `local_position` | EKF pose and velocity (TF publishing disabled) |
| `global_position` | Raw GNSS velocity, origin |
| `gps_status` | `GPS_RAW_INT` fields |
| `imu` | FC attitude |
| `odometry` | Outgoing external navigation |
| `setpoint_raw` | Outgoing setpoints |
| `obstacle_distance` | Outgoing obstacle sectors |
| `rangefinder` / `distance_sensor` | Height above ground |
| `rc_io` | RC channels |
| `estimator_status` | EKF flags |
| `debug_value` | Named values to GCS |
| `param` | Read-only checks of FC configuration at start-up |

Alternative external-nav path if `ODOMETRY` proves problematic on the bench: `vision_pose` (`VISION_POSITION_ESTIMATE`) plus `vision_speed` (`VISION_SPEED_ESTIMATE`).
