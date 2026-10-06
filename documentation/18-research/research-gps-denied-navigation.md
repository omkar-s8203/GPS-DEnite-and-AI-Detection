# Research Notes — GPS-Denied Navigation

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |
| Nature | Background research. Not a design document. |

## 1. Why GNSS fails

| Mode | Mechanism | Signature seen by a receiver / EKF |
|---|---|---|
| Blockage | Indoors, under dense canopy, tunnels, urban canyons | Satellite count falls; fix lost |
| Multipath | Reflections from buildings, water, vehicles | Position wanders by metres; reported accuracy may stay optimistic |
| Geometry | Few satellites in a narrow part of the sky | HDOP rises |
| Interference (unintentional) | Nearby electronics, USB 3, HDMI, video transmitters | C/N0 falls on all satellites together |
| Jamming | Deliberate broadband noise | Sudden loss of all satellites |
| Spoofing | Counterfeit signals | Plausible fix that drifts away from truth; good reported accuracy |
| Receiver cold start / antenna faults | — | No fix |

Observation: reported accuracy and HDOP are not sufficient by themselves, particularly for multipath and spoofing. An independent motion estimate (vision or inertial) compared with GNSS velocity is a stronger test. This motivated the velocity-consistency check (`dv`) in the state machine.

## 2. Families of GPS-denied localisation

| Family | Principle | Strengths | Weaknesses |
|---|---|---|---|
| Inertial dead reckoning | Integrate IMU | Self-contained | Unusable beyond seconds with MEMS sensors |
| Optical flow + range | Ground-relative velocity from a downward imager | Cheap, light, runs in the FC | Needs textured ground, low height; drifts |
| Visual odometry (mono/stereo) | Track features between frames | No infrastructure | Texture, light; mono has no scale |
| Visual-inertial odometry | Fuse features with IMU | Metric, robust to short visual gaps | Calibration and timing sensitive |
| Visual SLAM | VO/VIO + map + loop closure | Bounded drift on revisits | Compute; discontinuous corrections |
| LiDAR odometry / SLAM | Scan matching | Works in darkness; accurate | Mass, cost, power |
| Radio ranging (UWB, Wi-Fi RTT, beacons) | Ranges to known anchors | Absolute, drift-free | Infrastructure |
| Motion capture | External cameras | Millimetre accuracy | Laboratory only |
| Terrain / map matching, visual place recognition against satellite imagery | Compare onboard imagery with a prior map | Absolute without GNSS | Needs prior data and higher altitude; research-grade |
| Magnetic / signals-of-opportunity | Environmental fingerprints | No added infrastructure | Requires surveys; coarse |

For a sub-2 kg vehicle with a Pi-class computer and a camera, the practical set is: optical flow, stereo VO, stereo VIO.

## 3. ArduPilot's support for non-GNSS navigation [P]

Findings from the ArduPilot documentation consulted:

- **EKF3 source sets.** Up to three sets of sources for horizontal position, horizontal velocity, vertical position, vertical velocity and yaw (`EK3_SRC1_*`, `EK3_SRC2_*`, `EK3_SRC3_*`). Documented example for GPS/non-GPS transitions: set 1 = GPS (POSXY 3, VELXY 3, POSZ 1 baro, VELZ 3, YAW 1 compass); set 2 = ExternalNav (POSXY 6, VELXY 6, POSZ 1, VELZ 6, YAW 6); `EK3_SRC_OPTIONS = 0`.
- **Switching.** By a three-position RC switch (`RCx_OPTION = 90`), by Lua scripts (examples: `ahrs-source.lua` for GPS/T265/optical flow, and GPS/optical-flow and GPS/wheel-encoder variants) that switch on sensor health or EKF innovations, or by `MAV_CMD_SET_EKF_SOURCE_SET` (42007) from a GCS or companion (param1 = 1–3).
- **Cautions in the documentation.** Bench-test switching; wait and verify EKF health after a switch; expect a position jump when moving from non-GPS back to GPS; test at low speed with the pilot ready to take manual control; source changes are logged (events and an `XKFS` log message). Non-GPS navigation is stated to be unsuitable for fast or high-flying vehicles.
- **Visual odometry inputs.** For the Intel T265 integration: `VISO_TYPE = 2`, external-nav sources for position/velocity, barometer recommended for vertical position because of the camera's sensitivity to vibration, serial at 921 600 baud MAVLink 2, messages `VISION_POSITION_ESTIMATE` at 30 Hz and `VISION_POSITION_DELTA` for confidence at 1 Hz. An RC option exists for re-aligning vision yaw.
- **Depth-camera obstacle input.** Companion sends `OBSTACLE_DISTANCE`; `PRX1_TYPE = 2`; avoidance parameters (`AVOID_ENABLE = 7`, `AVOID_MARGIN`, `AVOID_BEHAVE`, `AVOID_DIST_MAX`, `AVOID_ANG_MAX`); ≥ 10 Hz; documented for Loiter and AltHold; forward-facing camera only; warnings about limited FOV.
- **MicoAir MTF-01.** `FLOW_TYPE = 5`, `RNGFND1_TYPE = 10`, range 0.01–8 m, serial MAVLink 1 at 115 200; on 4.5+ the sensor's MAVLink ID must be changed and the port set to not forward.

Implication: the autopilot side of this project is configuration plus a small script; the engineering effort belongs on the companion.

## 4. PX4's approach [P]

- EKF2 fuses external vision through `VISION_POSITION_ESTIMATE` or `ODOMETRY` (preferred, carries velocity), expected at 30–50 Hz, frame `MAV_FRAME_LOCAL_FRD`.
- `EKF2_EV_CTRL` selects which quantities are fused; `EKF2_EV_DELAY` compensates latency; sensor position offsets are parameters.
- GNSS and vision can be fused together; origin can be set without GNSS.

Implication: equally capable for external vision; less explicit about externally commanded source selection. See ADR-002.

## 5. How commercial and research systems achieve robustness (general observations) [L]

| Practice | Purpose |
|---|---|
| Global-shutter cameras with hardware-triggered IMU | Removes the timing problems this project must work around |
| Multiple cameras with wide/fisheye lenses | Features always in view; all-direction obstacle sensing |
| Tightly coupled optimisation with relocalisation | Accuracy and recovery |
| Factory calibration, online refinement | Stability over temperature and shock |
| Dedicated vision processors | Compute isolation |
| Redundant estimators and consistency monitors | Fault detection |

The gap between such systems and a commodity-part build is mainly in sensors and calibration, not in algorithms. This is why the camera suitability question (ADR-011) dominates the risk list.

## 6. Testing GPS denial legally

Radiating on GNSS frequencies is prohibited without authorisation in essentially every jurisdiction. Standard practice in research is to (a) disable GNSS use in software, (b) inject faults in simulation, or (c) fly into naturally denied areas. ArduPilot provides an RC auxiliary function to disable GPS for testing and simulator parameters for loss, noise and glitches.

## 7. Takeaways used in the design

| Finding | Used in |
|---|---|
| ArduPilot has a documented, commandable source-set mechanism | ADR-002, ADR-009 |
| Barometer for vertical position even in vision mode | State-estimation §4.2 |
| Position jumps on return to GNSS are expected | State machine T14 |
| Reported GNSS accuracy is not a complete health signal | `dv` consistency check |
| A flow sensor is cheap and FC-native | Tier 3 |
| Depth → `OBSTACLE_DISTANCE` is a documented pattern, forward-facing only | ADR-012 |
| Sensor quality, not algorithm choice, is the main differentiator | ADR-011 |
