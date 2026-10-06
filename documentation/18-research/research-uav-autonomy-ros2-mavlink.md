# Research Notes — UAV Autonomy, ROS 2 and MAVLink

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |
| Nature | Background research. Not a design document. |

## 1. UAV autonomy architecture patterns

| Pattern | Description | Used by |
|---|---|---|
| Autopilot only | All functions on the flight controller MCU | Hobby GNSS drones |
| **Autopilot + companion computer** | FC does estimation and control; a Linux computer does perception and high-level decisions; linked by MAVLink or DDS | Most research and developer platforms |
| Integrated flight computer | One board runs both real-time flight code and perception | Some commercial and developer boards |
| Off-board control | Ground computer closes loops over a radio link | Laboratory work with motion capture |

The companion pattern separates hard-real-time safety functions from best-effort perception. Its defining rule is that the FC must remain safe when the companion fails. Offboard/GUIDED interfaces in both major autopilots include timeouts for exactly this reason.

## 2. Levels of command a companion can issue

| Level | Interface | Risk | Chosen? |
|---|---|---|---|
| Mission upload | Waypoint protocol | Lowest; needs global position | No (local frame needed) |
| Position / velocity setpoints | `SET_POSITION_TARGET_LOCAL_NED` in GUIDED | Low; FC position controller in the loop | **Yes** |
| Attitude / thrust setpoints | `SET_ATTITUDE_TARGET` | Higher; companion closes the position loop | No |
| Rate / actuator | — | Highest | No |

## 3. ROS 2 on the Raspberry Pi 5

| Finding | Source |
|---|---|
| Jazzy Jalisco: LTS, May 2024 – May 2029; Tier 1 on Ubuntu 24.04 amd64 and arm64 | [S] ROS documentation summaries |
| Lyrical Luth: LTS released May 2026, supported to 2031; Tier 1 on Ubuntu 26.04 amd64/arm64; Ubuntu 24.04 is Tier 3 for Lyrical | [S] |
| Pi 5 is supported by Ubuntu 24.04 and later (not 22.04) | [S] |
| A community real-time Raspberry Pi image exists for Jazzy / 24.04 covering Pi 3–5 | [S] |
| MAVROS 2.14.x binaries exist for Jazzy on arm64 | [S] package index |
| RTAB-Map 0.23.x binaries exist for Jazzy | [S] |
| OpenVINS has Jazzy/24.04 support work | [S] |
| Camera access on Pi 5 under Ubuntu needs the Raspberry Pi libcamera fork | [S] several guides |

ROS 2 features relevant to this design:

| Feature | Use |
|---|---|
| Composable nodes + intra-process transport | Zero-copy image pipeline |
| Managed (lifecycle) nodes | Ordered bring-up, clean failure states |
| QoS policies (reliability, durability, deadline, lifespan, liveliness) | Sensor vs command vs latched-state semantics; rate monitoring; stale-command rejection |
| Actions | Cancellable navigation goals |
| rosbag2 with MCAP | Crash-tolerant logging |
| `use_sim_time` and `/clock` | Simulation parity |
| Discovery range controls | Localhost-only in flight |

Common pitfalls noted in community sources: QoS mismatches with MAVROS sensor topics; DDS discovery traffic over Wi-Fi; Python node jitter under CPU load; large messages copied between processes.

## 4. ROS standards

| REP | Content | Use |
|---|---|---|
| REP-103 | Units and axis conventions (ENU, FLU, right-handed, SI) | All ROS-side data |
| REP-105 | Frames `map`, `odom`, `base_link`; `map → odom` carries discontinuous corrections, `odom → base_link` is continuous | TF design |
| REP-147 | Conventions for aerial vehicles | Reference for message choices |
| REP-2000 | Release and platform support tiers | Distribution choice |

## 5. MAVLink

| Topic | Notes |
|---|---|
| Protocol | Lightweight binary messaging for drones; v2 adds signing, larger ID space, payload truncation |
| Addressing | System ID + component ID; the companion conventionally uses the vehicle's system ID with an onboard-computer component ID |
| Routing | Autopilots forward messages between links, enabling companion → GCS status through the FC |
| Microservices | Parameters, missions, commands (`COMMAND_LONG` / `COMMAND_INT` with `COMMAND_ACK`), time sync, heartbeat |
| Frames | Local frames are NED; body frames FRD; frame IDs matter and are a frequent integration error |
| Bandwidth | Sized for low-rate radios; stream rates must be requested deliberately |

Messages central to this project:

| Message | Role |
|---|---|
| `ODOMETRY` | Pose + velocity + covariances + reset counter + estimator type + quality |
| `VISION_POSITION_ESTIMATE`, `VISION_SPEED_ESTIMATE` | Alternative external-nav inputs |
| `OBSTACLE_DISTANCE` | 72-sector distance array around the vehicle |
| `SET_POSITION_TARGET_LOCAL_NED` | Position/velocity/acceleration/yaw setpoints with a type mask |
| `MAV_CMD_SET_EKF_SOURCE_SET` (42007) | ArduPilot: select EKF source set 1–3 |
| `GPS_RAW_INT`, `EKF_STATUS_REPORT` | GNSS and estimator health |
| `STATUSTEXT`, `NAMED_VALUE_FLOAT` | Operator-visible companion status |
| `TIMESYNC`, `SYSTEM_TIME` | Clock relation |
| `SET_GPS_GLOBAL_ORIGIN` | EKF origin without GNSS |

## 6. Bridges between ROS 2 and autopilots

| Bridge | Autopilot | Notes |
|---|---|---|
| MAVROS | ArduPilot and PX4 | Plugin architecture; frame conversion; time sync; long history |
| uXRCE-DDS | PX4 | Native uORB ↔ ROS 2 topics; first-class in PX4 |
| `AP_DDS` | ArduPilot | Native topics and services: pose, twist, GNSS, battery, IMU (experimental), clock; inputs `ap/cmd_vel`, `ap/cmd_gps_pose`, `ap/joy`, and `ap/tf` for external odometry (only `odom → base_link`); services for arming, mode switch, take-off, pre-arm check, parameters. Documentation specifies ROS 2 Humble. Rates bounded by the scheduler loop rate. [P] |
| MAVSDK | Primarily PX4 | C++/Python API; ArduPilot coverage partial |
| pymavlink | Any | Low-level; all conventions are the user's responsibility |

For ArduPilot on Jazzy with a need for source-set commands and obstacle distances, MAVROS is the only bridge covering everything today.

## 7. ArduPilot simulation with ROS 2 [P][S]

- `ardupilot_gazebo`: official plugin and models for Gazebo Sim; supports Garden, Harmonic (LTS), Ionic and Jetty.
- `ardupilot_gz`: ROS 2 launch integration aimed at Humble with native DDS; prerequisites list Humble with Harmonic or Jetty.
- Recommended pairing by Gazebo for new users: Ubuntu 24.04 + Jazzy + Harmonic.
- ArduPilot SITL runs Lua scripts and the complete EKF3, so source switching and watchdog logic can be tested unchanged.

## 8. SIYI MK15 as a link [P][S]

- Air unit provides S.Bus RC output, a UART MAVLink datalink and an Ethernet port for video/network; ground unit runs Android with QGroundControl.
- Fixed network addresses in the 192.168.144.x range; HDMI converter streams RTSP H.265 at about 12 Mbit/s.
- It is a serial telemetry link plus a video link, not a general IP network for ROS traffic.

## 9. Regulatory context (India) [U]

General knowledge to be verified against current official rules before flying: drones are categorised by all-up weight (nano below 250 g, micro up to 2 kg, small up to 25 kg, and larger classes); registration and operation are administered through a national digital platform; airspace is zoned with permission requirements varying by zone; altitude limits apply; educational and R&D institutions have had specific provisions. The 2 kg all-up-weight target in this project keeps the vehicle in the micro category. This note is not legal advice.

## 10. Takeaways used in the design

| Finding | Used in |
|---|---|
| Companion pattern with FC-side timeouts | Architecture principles; safety layers |
| Position/velocity setpoints are the lowest-risk control level that works in a local frame | FR-046 |
| Jazzy has the mature ecosystem today | ADR-001 |
| MAVROS covers all needed messages | ADR-008 |
| `AP_DDS` targets Humble and lacks needed interfaces | Deferred |
| REP-105 separates smooth and corrected frames | TF design |
| QoS mismatches are the common failure | L3 test T3-03 |
| MK15 is serial telemetry + video | Communication design |
