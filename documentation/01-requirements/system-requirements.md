# System Requirements Specification

| Field | Value |
|---|---|
| Document ID | GDN-REQ-001 |
| Version | 3.0 (DB-2.0: satellite image matching, §4.10, §5.7. DB-3.0: Android ground app, grid search, track and follow, §4.11, §4.12, §5.8) |
| Date | 2026-10-05 |
| Status | Baseline (design phase) |
| Applies to | AI-Integrated GPS-Denied Autonomous Drone |

## 1. Purpose and scope

This document defines what the system must do (functional requirements, `FR-xxx`) and how well it must do it (non-functional requirements, `NFR-xxx`). Every requirement is traceable to a design document and to a verification method. Numeric thresholds are **design targets**, not measured results; see [performance-requirements.md](../14-performance/performance-requirements.md) for the TARGET / ESTIMATE / MEASURED / VALIDATED convention.

**In scope:** a multirotor research prototype that flies with GNSS when it is healthy, detects GNSS degradation, and continues to navigate by matching images from a downward camera against a satellite image stored on board (absolute position), with visual odometry between matches, stereo vision at low altitude, and AI perception.

**DB-3.0 addition to scope:** an Android app on the MK15 from which the operator runs grid-search missions that pin detected objects on the map, and tracks and follows a selected object from above, with GNSS or with GNSS denied.

**Explicitly out of scope:** real search-and-rescue or disaster operations, flight over uninvolved people, and following people at low height. The rescue use is the motivation; the project demonstrates the capability on a mapped, undamaged test site with dummy targets.

**Out of scope:** long-range BVLOS flight, high-speed flight, operation in darkness or featureless environments, swarm operation, payload delivery, any use of real GNSS jamming (illegal; denial is always simulated — see FR-007).

## 2. Conventions

- **Shall** = mandatory. **Should** = desired, may be dropped with a recorded decision. **May** = optional.
- **Priority:** M = Must (needed for final demonstration), S = Should, C = Could (stretch).
- **Verification:** T = Test, A = Analysis, I = Inspection, D = Demonstration. The level refers to the testing pyramid in [testing-strategy.md](../13-testing/testing-strategy.md) (L1–L8).
- **Actors:** FC = flight controller (ArduPilot), CC = companion computer (Raspberry Pi 5), GCS = ground control station (SIYI MK15 ground unit), Pilot = safety pilot holding the RC.

## 3. Operating envelope (assumed)

These bound every requirement below. They are assumptions `[ASSUMPTION]` until confirmed by flight test.

| Parameter | Value | Reason |
|---|---|---|
| Vehicle class | Quadrotor, 450–500 mm class, all-up weight < 2 kg | Payload of ≈ 0.45–0.52 kg avionics; stays in the Indian "micro" category |
| Altitude above ground, **low regime** | 1–10 m | Stereo depth range, stereo VIO, optical-flow/ToF range; obstacle stop and AI at best range |
| Altitude above ground, **search profile** (DB-3.0: grid search, track, follow) | 25–30 m | Low enough to detect a person from above, high enough to match a reference of ≈ 0.25 m/px or better ([search-track-follow](../09-navigation/search-track-follow.md) §3) |
| Altitude above ground, **cruise regime** (satellite matching) | 40–60 m, nominal 50 m | Ground footprint must cover ≥ ≈ 250 reference pixels at 0.3–0.5 m/px ([visual-geolocalization](../09-navigation/visual-geolocalization.md) §3) |
| Ground speed (GPS-denied, autonomous) | Low regime: ≤ 1.5 m/s initially, ≤ 2.0 m/s max. Cruise regime: ≤ 3 m/s | Stopping distance vs. stereo range (low); fix rate and footprint overlap (cruise) |
| Mission area | Inside the onboard map pack, ≥ 60 m from its edge; ≈ 1 km × 1 km | Matching needs reference imagery |
| Terrain | Approximately flat; distinct, stable ground features (roads, buildings, field boundaries) | Planar-ground assumption; feature matching |
| Yaw rate (GPS-denied) | ≤ 45 °/s | Rolling-shutter distortion limit |
| Lighting | Daylight or well-lit indoor, > ~100 lux | Passive cameras, no illuminator |
| Scene | Textured, mostly static | VIO and stereo matching need texture |
| Wind | ≤ 5 m/s | Prototype airframe |
| Flight duration | ≥ 8 min usable | Battery estimate, see power budget |
| Line of sight | Visual line of sight at all times, safety pilot present | Safety and regulation |

## 4. Functional requirements

### 4.1 GNSS-assisted flight and GNSS monitoring

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-001 | The system shall support stabilised, position-hold and waypoint flight using GNSS as the horizontal position source when GNSS is healthy. | M | D, L8 | [mavlink-integration](../10-communication/mavlink-integration.md) |
| FR-002 | The CC shall monitor GNSS availability at ≥ 5 Hz using fix type, satellite count, HDOP, reported horizontal accuracy and FC EKF status. | M | T, L3/L5 | [gps-denied-state-machine](../02-system-architecture/gps-denied-state-machine.md) |
| FR-003 | The system shall classify GNSS as GOOD, DEGRADED or DENIED using configurable thresholds with hysteresis and minimum dwell times, and shall publish the classification. | M | T, L3/L5 | same |
| FR-004 | The system shall declare GNSS DENIED within 3 s of the denial condition becoming true. | M | T, L5 | same |
| FR-005 | On GNSS DENIED with healthy visual localisation, the system shall switch the FC's horizontal position/velocity source from GNSS to external navigation without requiring a landing or reboot. | M | T, L5/L8 | [state-estimation](../09-navigation/state-estimation.md) |
| FR-006 | When GNSS has been GOOD continuously for a configurable validation period (default 10 s), the system shall be able to return to GNSS as the position source, and shall report the position discontinuity observed at the switch. | S | T, L5/L8 | same |
| FR-007 | The system shall provide a way to simulate GNSS denial in flight and in simulation without radio interference (FC parameter / RC auxiliary function / simulator fault injection). | M | D, L5/L8 | [testing-strategy](../13-testing/testing-strategy.md) |
| FR-008 | The system should detect gross inconsistency between GNSS velocity and vision-derived velocity (possible GNSS glitch or spoofing) and treat it as DEGRADED. | C | T, L5 | state machine |

### 4.2 Sensing

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-010 | The CC shall capture left and right images from the stereo camera at a fixed, configurable rate (default 20 Hz) and resolution (default 640×480 monochrome). | M | T, L2 | [stereo-camera](../03-hardware/stereo-camera.md) |
| FR-011 | Each image shall carry a timestamp derived from the sensor frame-start time on the CC monotonic clock; the measured left/right timestamp skew shall be published for every pair. | M | T, L2 | [stereo-vision-pipeline](../06-computer-vision/stereo-vision-pipeline.md) |
| FR-012 | The system shall rectify stereo images using stored intrinsic and extrinsic calibration. | M | T, L2 | same |
| FR-013 | The system shall produce a metric depth image at ≥ 10 Hz from rectified stereo pairs, with invalid pixels explicitly marked. | M | T, L2/L4 | same |
| FR-014 | The CC shall acquire inertial measurements (3-axis gyro, 3-axis accelerometer) at ≥ 200 Hz with timestamps on the same clock as the images. | M | T, L2 | [sensors](../03-hardware/sensors.md) |
| FR-015 | Calibration data (camera intrinsics, stereo extrinsics, camera–IMU extrinsics and time offset, IMU noise parameters) shall be stored in version-controlled files and loaded at start-up; the system shall refuse to enter READY if calibration is missing. | M | I, T L3 | [calibration](../06-computer-vision/calibration.md) |
| FR-016 | The FC shall measure height above ground with a downward range sensor at ≥ 20 Hz within 0.05–8 m. | M | T, L2/L7 | [sensors](../03-hardware/sensors.md) |

### 4.3 Localisation and state estimation

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-020 | The CC shall estimate 6-DoF pose and linear velocity from stereo images and inertial data (visual-inertial odometry) at ≥ 20 Hz. | M | T, L4/L7 | [vio-design](../07-vio-slam/vio-design.md) |
| FR-021 | The CC shall continuously assess VIO health (tracked feature count, estimator covariance, output rate, latency, resets) and publish it. | M | T, L3 | same |
| FR-022 | While GNSS is GOOD, the CC shall estimate the 4-DoF transform (x, y, z, yaw) between the VIO frame and the FC local frame, so that vision-derived pose is expressed in the FC frame before GNSS is lost. | M | T, L4/L5 | [coordinate-frames](../02-system-architecture/coordinate-frames.md) |
| FR-023 | The CC shall send aligned external-navigation pose and velocity to the FC at 20–30 Hz with covariance, a reset counter and a quality indicator. | M | T, L5/L6 | [mavlink-integration](../10-communication/mavlink-integration.md) |
| FR-024 | The CC shall compute a scalar localisation confidence in [0, 1] and a discrete level (HIGH / MEDIUM / LOW / LOST). | M | T, L3/L4 | [state-estimation](../09-navigation/state-estimation.md) |
| FR-025 | The FC shall have a position/velocity source that does not depend on the CC (optical flow + range sensor) as a third navigation tier. | S | T, L7/L8 | [sensors](../03-hardware/sensors.md) |
| FR-026 | After VIO failure the system shall attempt re-initialisation only while the vehicle is held by another source or by the pilot, and shall not feed the FC from VIO until health is restored for a configurable period. | M | T, L4/L5 | state machine |
| FR-027 | The authoritative vehicle state for control shall be the FC's EKF output; the CC shall not run a competing attitude or position controller. | M | I, A | [state-estimation](../09-navigation/state-estimation.md) |

### 4.4 Perception

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-030 | The system shall detect obstacles within the stereo field of view between 0.5 m and 6 m and compute the nearest obstacle distance per angular sector. | M | T, L4/L7 | [stereo-vision-pipeline](../06-computer-vision/stereo-vision-pipeline.md) |
| FR-031 | The CC shall send sector obstacle distances to the FC at ≥ 10 Hz so that FC-side avoidance can act on them. | S | T, L5 | [mavlink-integration](../10-communication/mavlink-integration.md) |
| FR-032 | The system shall detect objects of a configured class set in the left camera image using a neural network, at ≥ 5 Hz. | M | T, L2/L4 | [ai-architecture](../08-ai/ai-architecture.md) |
| FR-033 | For each detected object the system shall estimate range from the depth image and report it with a validity flag. | M | T, L4/L7 | [ai-stereo-fusion](../08-ai/ai-stereo-fusion.md) |
| FR-034 | The system shall express detected object positions in the `map` frame. | S | T, L4 | same |
| FR-035 | AI perception shall not be in the path that keeps the vehicle stable or localised; loss of the AI pipeline shall not degrade flight safety. | M | A, T L4 | [ai-architecture](../08-ai/ai-architecture.md) |

### 4.5 Navigation and mission

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-040 | The vehicle shall hold position in GNSS-denied mode using external navigation. | M | D, L8 | [autonomous-navigation](../09-navigation/autonomous-navigation.md) |
| FR-041 | The vehicle shall fly a short sequence of local waypoints (≥ 3 waypoints, ≥ 10 m total path) in GNSS-denied mode. | M | D, L8 | same |
| FR-042 | The CC shall limit commanded speed as a function of localisation confidence and shall command a hold when confidence is LOW. | M | T, L4/L5 | same |
| FR-043 | The CC shall stop forward motion when an obstacle is closer than a configurable stop distance (default 2.0 m) along the direction of travel. | M | T, L4/L8 | same |
| FR-044 | The system shall execute a mission defined as a list of local-frame waypoints and actions (takeoff, goto, hold, land) through a single mission interface. | S | T, L4 | [node-reference](../05-ros2/node-reference.md) |
| FR-045 | On mission completion, abort, or unrecoverable fault the system shall hand the vehicle to an FC-native behaviour (LOITER, LAND or RTL as appropriate to the active position source). | M | T, L5 | [safety-architecture](../12-safety/safety-architecture.md) |
| FR-046 | The CC shall command motion only through position/velocity setpoints in the FC's GUIDED mode. It shall not command attitude, rates or motor outputs. | M | I | [mavlink-integration](../10-communication/mavlink-integration.md) |

### 4.6 Communication and telemetry

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-050 | The CC and FC shall communicate using MAVLink 2 over a dedicated serial link. | M | T, L6 | [mavlink-integration](../10-communication/mavlink-integration.md) |
| FR-051 | FC telemetry shall be available on the GCS through the RC system's datalink, independent of the CC. | M | D, L7 | [telemetry-and-links](../10-communication/telemetry-and-links.md) |
| FR-052 | The CC shall report navigation mode, localisation confidence and faults to the GCS as human-readable status messages and named numeric values. | M | T, L6 | same |
| FR-053 | The system should provide a live video downlink with perception overlay to the GCS. | S | D, L7 | same |
| FR-054 | The CC and FC clocks shall be related by a continuously estimated offset so that timestamps sent to the FC are meaningful. | M | T, L6 | [mavlink-integration](../10-communication/mavlink-integration.md) |

### 4.7 Logging

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-060 | The CC shall record a configurable set of topics to rosbag2 (MCAP) for every armed flight. | M | T, L3 | [ros2-architecture](../05-ros2/ros2-architecture.md) |
| FR-061 | The FC shall record its own onboard log for every armed flight, including external-navigation inputs and EKF source changes. | M | I, L7 | [mavlink-integration](../10-communication/mavlink-integration.md) |
| FR-062 | Every navigation-mode transition and safety action shall be logged with timestamp, cause and the values that triggered it. | M | T, L3 | state machine |
| FR-063 | Each recording shall store run metadata: software commit, parameter snapshot, calibration identifiers, hardware configuration. | S | I | [testing-strategy](../13-testing/testing-strategy.md) |

### 4.8 Safety

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-070 | The pilot shall be able to take manual control at any time by changing flight mode on the RC; no CC action shall be able to prevent or delay this. | M | D, L7/L8 | [safety-architecture](../12-safety/safety-architecture.md) |
| FR-071 | The CC shall not arm the vehicle. Arming is a pilot action. | M | I, T L5 | same |
| FR-072 | Loss of RC link shall trigger the FC's RC failsafe independently of the CC. | M | T, L7 | same |
| FR-073 | Loss of the CC (crash, hang, power loss, link loss) while in GUIDED mode shall be detected by the FC within 2 s and shall result in an FC-native safe behaviour. | M | T, L5/L7 | same |
| FR-074 | Low and critical battery levels shall trigger FC battery failsafes independently of the CC. | M | T, L7 | same |
| FR-075 | The CC shall monitor its own temperature, throttling state, CPU load and memory, shall shed non-critical load (AI, video) before critical load, and shall report thermal faults. | M | T, L2/L7 | same |
| FR-076 | A geofence (maximum radius and altitude) shall be enforced by the FC. | M | T, L5 | same |
| FR-077 | The system shall run an automated pre-flight check and present a single GO / NO-GO result with reasons. | M | D, L7 | same |
| FR-078 | A motor emergency stop shall be available on the RC. | M | D, L7 | same |
| FR-079 | In GNSS-denied mode, loss of all position sources shall result in a defined behaviour that needs no position estimate (altitude-hold for pilot takeover, then land). | M | T, L5/L8 | state machine |

### 4.9 System behaviour and configuration

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-080 | Navigation-mode behaviour shall be implemented as an explicit state machine with documented states, transitions and guards. | M | I, T L1 | [gps-denied-state-machine](../02-system-architecture/gps-denied-state-machine.md) |
| FR-081 | Flight-critical nodes shall be managed lifecycle nodes with an ordered bring-up and shut-down. | M | T, L3 | [ros2-architecture](../05-ros2/ros2-architecture.md) |
| FR-082 | All thresholds, rates and topic names shall be parameters loaded from files, not constants in code. | M | I | same |
| FR-083 | The same ROS 2 graph shall run in simulation and on hardware; only driver nodes and parameters differ. | M | D, L4/L5 | [simulation-strategy](../11-simulation/simulation-strategy.md) |

### 4.10 Visual geo-localisation by satellite image matching (DB-2.0)

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-090 | The CC shall store a georeferenced reference image of the mission area ("map pack") with its source, date, resolution and licence. | M | I | [visual-geolocalization](../09-navigation/visual-geolocalization.md) §4 |
| FR-091 | The CC shall capture downward images at ≥ 15 Hz with timestamps on the CC clock. | M | T, L2 | [downward-camera](../03-hardware/downward-camera.md) |
| FR-092 | When GNSS is denied and the vehicle is in the cruise regime, the CC shall estimate absolute horizontal position by matching the downward image against the map pack at ≥ 0.5 Hz. | M | T, L3/L8 | visual-geolocalization §5 |
| FR-093 | Every match result shall carry a covariance and quality measures, and shall be accepted only if it passes geometric and consistency gates. Rejected results shall not influence the position estimate. | M | T, L1/L3 | §5, §9 |
| FR-094 | The CC shall estimate relative motion from the downward camera at ≥ 15 Hz to carry the position between accepted fixes. | M | T, L3 | §6 |
| FR-095 | The CC shall fuse accepted fixes with relative odometry into one continuous pose without steps larger than a configurable limit, and send it to the FC as external navigation. | M | T, L1/L5 | §8 |
| FR-096 | The CC shall publish the time since the last accepted fix and the estimated position uncertainty, and shall include them in localisation confidence. | M | T, L3 | §9 |
| FR-097 | The system shall refuse goals outside map coverage and shall not enter the map-matching mode where no reference imagery exists. | M | T, L1/L4 | §10 |
| FR-098 | An off-board tool shall convert a georeferenced image into a map pack with precomputed features, and a procedure shall register the map against GNSS at the test site. | M | D | §4 |
| FR-099 | Every match attempt (image, window, result, gate decision) shall be logged; while GNSS is good, every fix shall also be compared with GNSS and the error recorded. | M | T, L3 | §9, §14 |
| FR-100 | When fixes are unavailable for longer than a configurable time or the position uncertainty exceeds a limit, the system shall degrade in defined steps (slow, hold, hand to pilot / land). | M | T, L5 | §10 |

### 4.11 Ground app (DB-3.0)

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-110 | A native Android app shall run on the SIYI MK15 ground unit (Android 9) and communicate with the CC over the MK15's IP link. | M | D, L7 | [ground-app](../10-communication/ground-app.md) |
| FR-111 | The app shall show live video with detection boxes, navigation mode, localisation confidence, time since the last map fix, battery and height. | M | D, L7 | same §5 |
| FR-112 | The app shall show the drone's position, uncertainty, path, map coverage and geofence on the stored satellite map, without an internet connection. | M | D, L7 | same |
| FR-113 | The operator shall be able to select an object by tapping it in the video; the selection shall refer to the frame that was tapped. | S | T, L3 | same §6 |
| FR-114 | The operator shall be able to draw a search area, set its parameters, and start, pause, resume and abort a search. | M | D, L5/L7 | same |
| FR-115 | The app shall list findings with class, confidence, coordinates, time and a thumbnail, and allow confirm/reject and export. | M | D, L7 | same |
| FR-116 | A hold ("Stop") command shall be available on every screen. | M | I | same |
| FR-117 | The app shall not be able to arm, disarm, change flight mode or override the pilot. Every app request shall pass the same gates as any mission and shall be answered with an acknowledgement or a refusal with a reason. | M | I, T L3 | same §4 |
| FR-118 | Loss of the app link shall lead to the defined behaviour for the active mission and never to an unsafe action. | M | T, L5 | same §8 |
| FR-119 | QGroundControl shall remain usable on the MK15's telemetry datalink while the app is in use. | M | D, L7 | same §2 |

### 4.12 Search, track and follow (DB-3.0)

| ID | Requirement | Pri | Verif. | Design ref. |
|---|---|---|---|---|
| FR-120 | The system shall plan a back-and-forth search pattern covering an operator-defined polygon at a given height and overlap, and shall refuse areas outside map coverage, outside the geofence or too large for the battery. | M | T, L1/L4 | [search-track-follow](../09-navigation/search-track-follow.md) §5 |
| FR-121 | The system shall fly the search pattern with GNSS or with GNSS denied, pausing when localisation is degraded. | M | T, L5/L8 | same |
| FR-122 | During a search the system shall detect objects of the configured classes in the downward camera image using an aerial-view model. | M | T, L2/L8 | same §4 |
| FR-123 | Each detection shall be converted to a ground position (latitude, longitude) by projection, and a finding shall be raised only after confirmation in several frames. | M | T, L1/L8 | same §2, §5.3 |
| FR-124 | Findings shall be stored with class, confidence, position, uncertainty, time and image evidence, and reported to the app. | M | T, L3 | same |
| FR-125 | The system shall record the area actually covered by the camera footprint. | S | T, L4 | same §5.2 |
| FR-126 | On request the vehicle shall fly to a point above a finding and hold. | S | D, L5 | same |
| FR-127 | The system shall track an operator-selected object, estimating its ground position and velocity, and shall report loss of the target. | S | T, L1/L4 | same §6 |
| FR-128 | On request the vehicle shall follow the tracked object from above at a fixed height of at least 20 m, with GNSS or with GNSS denied. | S | T, L5/L8 | same §7 |
| FR-129 | During follow the vehicle shall not reduce its height automatically, shall not leave the geofence or map coverage, and shall hold position when the target is lost. | M (if FR-128 is built) | T, L5 | same |
| FR-130 | Search and follow shall be available only in the search flight profile (25–30 m) with the aerial model loaded and localisation in the absolute (map-fix) sub-mode or on GNSS. | M | T, L3 | same §9 |
| FR-131 | No autonomous behaviour shall reduce separation from people: tests use dummies or consenting team members, and no flight takes place over uninvolved people. | M | I | [safety-architecture](../12-safety/safety-architecture.md) §8b |

## 5. Non-functional requirements

### 5.1 Performance

| ID | Requirement | Target | Verif. |
|---|---|---|---|
| NFR-001 | Stereo capture rate | 20 Hz ± 1 Hz, < 1 % dropped pairs | T, L2 |
| NFR-002 | Left/right timestamp skew | ≤ 1 ms (goal ≤ 0.5 ms) for ≥ 99 % of pairs | T, L2 |
| NFR-003 | VIO output rate | ≥ 20 Hz | T, L4/L7 |
| NFR-004 | Latency, image capture → external-nav message leaving CC | ≤ 80 ms (95th percentile) | T, L6 |
| NFR-005 | VIO drift (slow flight, textured scene) | ≤ 2 % of distance travelled (goal ≤ 1 %) | T, L7/L8 |
| NFR-006 | Position hold in GNSS-denied mode | Low regime (stereo VIO): ≤ 0.5 m horizontal RMS over 60 s. Cruise regime (map matching): see NFR-074 | T, L8 |
| NFR-007 | Depth image rate | ≥ 10 Hz | T, L2 |
| NFR-008 | Depth error | ≤ 5 % at 2 m, ≤ 10 % at 5 m (static scene) | T, L2 |
| NFR-009 | AI detection rate / latency | ≥ 5 Hz; ≤ 150 ms capture → published detection | T, L2 |
| NFR-010 | Obstacle detection latency (capture → sector distances sent) | ≤ 200 ms | T, L4 |
| NFR-011 | GNSS-denied detection time | ≤ 3 s | T, L5 |
| NFR-012 | Position-source switch | Completed ≤ 1 s after decision; position step ≤ 1.0 m | T, L5/L8 |
| NFR-013 | CC CPU utilisation, full stack | ≤ 75 % average across 4 cores | T, L7 |
| NFR-014 | CC memory use, full stack | ≤ 4 GB resident | T, L7 |
| NFR-015 | CC SoC temperature in flight configuration | ≤ 75 °C steady state at 35 °C ambient; never throttled | T, L7 |

### 5.2 Reliability and robustness

| ID | Requirement |
|---|---|
| NFR-020 | No single CC software fault shall cause loss of vehicle control; the FC shall remain flyable by the pilot with the CC powered off. |
| NFR-021 | Flight-critical CC nodes shall restart automatically on crash; restart shall not cause an unbounded setpoint or position jump. |
| NFR-022 | The CC shall boot to READY without a network connection, display, keyboard or operator login. |
| NFR-023 | The full CC stack shall run for ≥ 30 min continuously on the bench without a crash, memory growth > 5 %, or thermal throttling. |
| NFR-024 | A CC brown-out shall not corrupt the root filesystem badly enough to prevent the next boot (journaled/overlay strategy, logs on a separate partition). |

### 5.3 Safety

| ID | Requirement |
|---|---|
| NFR-030 | All flight safety functions (stabilisation, failsafes, geofence, arming checks, motor control) shall remain in the FC. |
| NFR-031 | Every autonomous behaviour shall have a defined, tested fallback that requires fewer sensors than the behaviour itself. |
| NFR-032 | No autonomous flight shall take place until every lower level of the testing pyramid has met its exit criteria. |
| NFR-033 | Propellers shall be removed for all bench tests that can arm motors. |

### 5.4 Physical and electrical

| ID | Requirement | Target |
|---|---|---|
| NFR-040 | Added avionics mass (everything except frame, propulsion, battery) | ≤ 550 g |
| NFR-041 | Avionics electrical load | ≤ 20 W average, ≤ 35 W peak |
| NFR-042 | CC supply | 5.1 V ± 0.15 V at the Pi under a 5 A step load |
| NFR-043 | All-up weight | < 2.0 kg |
| NFR-044 | Camera mount | Rigid stereo bar; vibration-isolated from motors; no relative motion between the two lenses and the IMU |

### 5.5 Maintainability, portability, openness

| ID | Requirement |
|---|---|
| NFR-050 | Software shall be organised as independent ROS 2 packages with interfaces defined in a single interface package. |
| NFR-051 | The VIO and camera implementations shall be replaceable without changing downstream nodes (topic-level contract). |
| NFR-052 | All software dependencies shall be open-source with licences recorded; licence obligations (GPL/AGPL) shall be documented. |
| NFR-053 | Builds shall be reproducible from a documented procedure on a clean OS image. |
| NFR-054 | Code shall follow ROS 2 style (ament linters) and include unit tests for all logic nodes. |
| NFR-055 | Every design decision of consequence shall be recorded as an ADR. |

### 5.6 Cost and regulatory

| ID | Requirement |
|---|---|
| NFR-060 | Additional hardware beyond items already owned should stay within a college-project budget; each purchase shall be justified in the BOM. |
| NFR-061 | Operation shall comply with the applicable Indian drone rules and institute policy (category by weight, registration, permitted zones, altitude limits). The team shall confirm current rules before any outdoor flight. |
| NFR-062 | No intentional radio interference with GNSS shall ever be used. |

### 5.7 Visual geo-localisation (DB-2.0)

| ID | Requirement | Target |
|---|---|---|
| NFR-070 | Horizontal error of accepted fixes against GNSS | ≤ 5 m RMS (goal ≤ 3 m) |
| NFR-071 | Fix rate in the cruise regime over suitable terrain | ≥ 0.5 Hz attempted; ≥ 70 % of attempts accepted |
| NFR-072 | Accepted wrong fixes (error > 15 m) | < 1 % of accepted fixes |
| NFR-073 | Time from GNSS denial to first accepted fix | ≤ 10 s |
| NFR-074 | Position hold on map matching, 60 s | ≤ 5 m horizontal RMS |
| NFR-075 | Ground visual odometry drift | ≤ 3 % of distance |
| NFR-076 | Match latency (image → fix published) | ≤ 500 ms |
| NFR-077 | Map pack size for a 1 km × 1 km area | ≤ 500 MB including features |

### 5.8 Ground app, search, track and follow (DB-3.0)

| ID | Requirement | Target |
|---|---|---|
| NFR-080 | Video latency, camera to app screen | ≤ 500 ms |
| NFR-081 | Command latency, tap to acknowledgement | ≤ 300 ms |
| NFR-082 | Video to the app | ≥ 640×480 at ≥ 8 fps |
| NFR-083 | App on the MK15 (Android 9, 2 GB RAM) | No crash or freeze in a 20 min session; memory ≤ 400 MB |
| NFR-084 | Time for a new operator to start a search | ≤ 2 min without help |
| NFR-085 | Search coverage of the requested area | ≥ 95 % by recorded footprint |
| NFR-086 | Detection of person-sized dummy targets from 25–30 m, open ground, daylight | Recall ≥ 60 % (goal 80 %); precision ≥ 70 % |
| NFR-087 | Detection of vehicles from 25–30 m | Recall ≥ 85 % |
| NFR-088 | Position error of a finding | ≤ 5 m on GNSS; ≤ 8 m with GNSS denied |
| NFR-089 | Aerial detection rate (full frame, tiled) | ≥ 0.5 Hz |
| NFR-090 | Search area per battery | ≥ 2 ha (goal 4 ha) |
| NFR-091 | Follow: target kept within the central half of the image | For target speeds ≤ 2 m/s |
| NFR-092 | Tracker re-acquisition after a short loss | ≤ 3 s when the target reappears within the gate |

## 6. Requirement-to-subsystem allocation

| Subsystem | Requirements |
|---|---|
| Camera + IMU drivers | FR-010, 011, 014, 015; NFR-001, 002 |
| Stereo processing | FR-012, 013, 030; NFR-007, 008, 010 |
| VIO | FR-020, 021, 026; NFR-003, 005 |
| Localisation manager | FR-022, 023, 024; NFR-004 |
| Navigation-mode manager | FR-002–008, 080; NFR-011, 012 |
| AI perception | FR-032–035; NFR-009 |
| Obstacle handling | FR-030, 031, 043 |
| Navigation / mission | FR-040–046 |
| MAVLink bridge | FR-050, 054 |
| Telemetry | FR-051–053 |
| Safety supervisor | FR-070–079; NFR-020, 021 |
| Flight controller configuration | FR-001, 016, 025, 061, 072, 073, 074, 076, 078 |
| Logging / diagnostics | FR-060–063 |

## 7. Open requirement issues

| # | Issue | Owner | Needed by |
|---|---|---|---|
| RQ-1 | Airframe is not specified. The envelope in §3 assumes a 450–500 mm quad. | Team | Phase 2 |
| RQ-2 | The AI class set (what must be detected) is not specified. Default assumed: person, vehicle, landing marker. | Team / guide | Phase 9 |
| RQ-3 | Whether indoor flight is required. It changes the yaw source and the test site. | Team | Phase 2 |
| RQ-4 | NFR-002 and NFR-005 may not be achievable with the current stereo camera; see [ADR-011](../17-decisions/ADR-011-stereo-camera-suitability.md). Less critical since DB-2.0: stereo VIO now serves the low regime only. | Team | Gate G2b |
| RQ-5 | Reference imagery for the test site: source, resolution, licence. | Team | Gate G2 |
| RQ-6 | Flight at 40–60 m: site permission, applicable altitude limit, crew procedure. | Team + guide | Before any cruise-regime flight |
| RQ-7 | NFR-070 – NFR-072 depend on the terrain of the chosen site; featureless sites cannot meet them. | Team | Site selection |
