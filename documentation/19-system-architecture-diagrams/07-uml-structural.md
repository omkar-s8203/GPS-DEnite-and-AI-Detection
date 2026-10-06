# 7. UML Structural Diagrams

| Field | Value |
|---|---|
| Document ID | GDN-DIA-007 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

UML 2 defines seven structural diagram types. All seven are given here. Mermaid draws class diagrams natively; the other types are drawn with its flowchart notation using UML conventions (stereotypes in « », nodes, components, ports).

| # | UML diagram | In this document |
|---|---|---|
| 1 | Class | U1 drone software, U2 Android app, U3 messages |
| 2 | Object | U4 |
| 3 | Component | U5 |
| 4 | Composite structure | U6 |
| 5 | Deployment | U7 |
| 6 | Package | U8 |
| 7 | Profile | U9 |

## U1 Class diagram: drone software (main classes)

```mermaid
classDiagram
    class NavModeManager {
        -NavModeState state
        -GnssHealth gnss
        -int ekfSourceSet
        -FlightProfile profile
        +classifyGnss(GpsRaw, EkfStatus) GnssHealth
        +step() NavModeState
        +requestSourceSet(int) bool
        +speedLimit() float
    }
    class LocalizationManager {
        -OffsetFilter filter
        -float confidence
        -bool aligned
        +onOdometry(Odometry)
        +onGeoFix(GeoFix)
        +onFcPose(Pose)
        +publishExternalNav()
        +confidenceLevel() Level
    }
    class OffsetFilter {
        -Vector2 offset
        -Matrix2 covariance
        -float slewRate
        +predict(float distance)
        +update(Vector2 z, Matrix2 r) bool
        +appliedOffset() Vector2
        +sigma() float
    }
    class MapMatcher {
        -MapPack map
        -MatchMethod method
        -float minHeight
        +match(Image, Attitude, float height, Pose prediction) GeoFix
        -orthorectify(Image) Image
        -gate(MatchResult) bool
    }
    class MapPack {
        +string id
        +float gsd
        +Bounds bounds
        +string licence
        +tilesIn(Window) Tile[]
        +featuresIn(Window) Feature[]
        +minMatchHeight(float hfov) float
    }
    class GroundVO {
        -Image previous
        +update(Image, Gyro, float height) Odometry
    }
    class VioMonitor {
        -OdomSource source
        +selectSource(float height)
        +health() VioStatus
    }
    class Detector {
        -Model activeModel
        +setProfile(FlightProfile)
        +detect(Image) Detection[]
    }
    class FindingManager {
        -Finding[] findings
        +onDetections(Detection[], Pose)
        -projectToGround(Detection, Pose) Point
        -confirm(Candidate) bool
    }
    class Finding {
        +int id
        +string className
        +float score
        +double latitude
        +double longitude
        +float sigma
        +Review review
    }
    class SearchPlanner {
        +plan(Polygon, float height, float overlap) SearchPlan
        +validate(Polygon) Refusal
        +execute(SearchPlan)
    }
    class SearchPlan {
        +Line[] lines
        +float areaM2
        +float timeS
    }
    class TargetTracker {
        -KalmanFilter kf
        -TrackState state
        +select(int frameId, Point2 tap) bool
        +onDetections(Detection[])
        +target() TrackedTarget
    }
    class Navigator {
        -NavigatorState state
        +goTo(Pose goal)
        +followTarget(TrackedTarget)
        +hold()
        -limitSpeed() float
    }
    class MissionManager {
        -Behaviour active
        +startSearch(Polygon)
        +startFollow(int trackId)
        +gotoFinding(int id)
        +hold()
        +onAppLinkLost()
    }
    class SafetySupervisor {
        -SafetyLevel level
        +checkHeartbeats()
        +shedLoad(int level)
        +runPreflight() PreflightResult
        +setpointsAllowed() bool
    }
    class AppGateway {
        -bool clientConnected
        +onRequest(Request) Reply
        -validate(Request) bool
        +sendStatus()
        +sendFinding(Finding)
    }

    LocalizationManager *-- OffsetFilter
    LocalizationManager --> VioMonitor : odometry
    LocalizationManager --> MapMatcher : fixes
    MapMatcher *-- MapPack
    VioMonitor --> GroundVO : above 12 m
    NavModeManager --> LocalizationManager : confidence
    NavModeManager --> Detector : selects model
    FindingManager --> Detector : detections
    FindingManager "1" o-- "0..*" Finding
    SearchPlanner --> SearchPlan : creates
    SearchPlanner --> Navigator : GoTo
    TargetTracker --> Detector : detections
    Navigator --> TargetTracker : target
    MissionManager --> SearchPlanner
    MissionManager --> Navigator
    MissionManager --> TargetTracker
    SafetySupervisor --> Navigator : allows setpoints
    SafetySupervisor --> MissionManager : hold
    AppGateway --> MissionManager : requests
    AppGateway --> FindingManager : findings
    Navigator --> NavModeManager : speed limit
```

## U2 Class diagram: Android app

```mermaid
classDiagram
    class MainActivity {
        +onCreate()
        +setContent()
    }
    class LiveViewModel {
        +StateFlow~LiveUiState~ ui
        +onTap(float x, float y)
        +onTrack()
        +onFollow()
        +onStop()
    }
    class MapViewModel {
        +StateFlow~MapUiState~ ui
        +onAreaDrawn(List~LatLon~ polygon)
        +onStartSearch(SearchSettings)
        +onPause()
        +onAbort()
    }
    class FindingsViewModel {
        +StateFlow~List~Finding~~ findings
        +onReview(int id, bool confirmed)
        +onGoThere(int id)
        +onExport()
    }
    class StatusViewModel {
        +StateFlow~StatusUiState~ ui
        +onRunPreflight()
    }
    class DroneRepository {
        +StateFlow~DroneStatus~ status
        +StateFlow~ConnectionState~ connection
        +send(Request) Reply
        +observeDetections() Flow
        +observeFindings() Flow
    }
    class ControlClient {
        -WebSocket socket
        +connect(String host, int port, String key)
        +send(String json)
        +incoming() Flow
    }
    class VideoClient {
        +frames() Flow~VideoFrame~
    }
    class TileCache {
        +tile(int z, int x, int y) Bitmap
    }
    class FindingsDao {
        +insert(Finding)
        +all() Flow
        +setReview(int id, Review r)
    }
    class DroneStatus {
        +String navMode
        +float confidence
        +float fixAgeS
        +double lat
        +double lon
        +float heightM
        +float batteryPct
        +bool autonomyEnabled
    }
    class Finding {
        +int id
        +String className
        +float score
        +double lat
        +double lon
        +float sigmaM
        +Review review
    }
    class VideoFrame {
        +int frameId
        +long stampMs
        +Bitmap image
    }
    class Request {
        <<interface>>
        +int id
        +String type
    }
    class SelectTarget
    class SearchStart
    class FollowStart
    class Hold

    MainActivity --> LiveViewModel
    MainActivity --> MapViewModel
    MainActivity --> FindingsViewModel
    MainActivity --> StatusViewModel
    LiveViewModel --> DroneRepository
    MapViewModel --> DroneRepository
    FindingsViewModel --> DroneRepository
    StatusViewModel --> DroneRepository
    DroneRepository *-- ControlClient
    DroneRepository *-- VideoClient
    DroneRepository *-- TileCache
    DroneRepository *-- FindingsDao
    DroneRepository ..> DroneStatus
    DroneRepository ..> Finding
    VideoClient ..> VideoFrame
    Request <|.. SelectTarget
    Request <|.. SearchStart
    Request <|.. FollowStart
    Request <|.. Hold
```

## U3 Class diagram: main data messages

```mermaid
classDiagram
    class GeoFix {
        +Time stamp
        +bool accepted
        +int rejectReason
        +double latitude
        +double longitude
        +Point positionMap
        +float[4] covariance
        +int inliers
        +float inlierRatio
        +float scale
    }
    class LocalizationStatus {
        +float confidence
        +Level level
        +bool aligned
        +float posSigmaM
        +float fixAgeS
        +SubMode positionSubmode
        +OdomSource odometrySource
    }
    class NavModeState {
        +State state
        +int ekfSourceSet
        +bool manualOverride
        +float speedLimit
        +string reason
    }
    class TrackedTarget {
        +int trackId
        +TrackState state
        +string className
        +Point positionMap
        +Vector3 velocityMap
        +float timeSinceSeenS
    }
    class Finding {
        +int id
        +string className
        +float score
        +double latitude
        +double longitude
        +float positionSigmaM
        +Review review
    }
    class SearchStatus {
        +SearchState state
        +float progress
        +int lineIndex
        +int linesTotal
        +float areaCoveredM2
    }
    class SafetyState {
        +SafetyLevel level
        +bool autonomyEnabled
        +bool setpointsAllowed
        +int loadShedLevel
    }
    class State {
        <<enumeration>>
        BOOT
        SENSOR_CHECK
        READY
        GPS_NAV
        GPS_DEGRADED
        VISION_NAV
        VISION_DEGRADED
        FLOW_FALLBACK
        GPS_RECOVERY
        LOCALIZATION_LOST
        FAULT
    }
    class Level {
        <<enumeration>>
        HIGH
        MEDIUM
        LOW
        LOST
    }
    NavModeState --> State
    LocalizationStatus --> Level
    LocalizationStatus ..> GeoFix : built from
```

## U4 Object diagram: a moment during a grid search with GPS denied

```mermaid
flowchart TB
    nmm["<u>nmm : NavModeManager</u><br/>state = VISION_NAV<br/>ekfSourceSet = 2<br/>profile = SEARCH"]
    lm["<u>lm : LocalizationManager</u><br/>confidence = 0.86<br/>aligned = true"]
    of["<u>filter : OffsetFilter</u><br/>offset = (1.8, -0.9) m<br/>sigma = 2.4 m"]
    mp["<u>site : MapPack</u><br/>id = campus_2026_10<br/>gsd = 0.10 m/px"]
    fix["<u>fix412 : GeoFix</u><br/>accepted = true<br/>inliers = 41"]
    plan["<u>plan3 : SearchPlan</u><br/>lines = 6<br/>areaM2 = 21000"]
    sp["<u>sp : SearchPlanner</u><br/>lineIndex = 3"]
    f1["<u>f1 : Finding</u><br/>className = person<br/>score = 0.62<br/>review = UNREVIEWED"]
    f2["<u>f2 : Finding</u><br/>className = vehicle<br/>score = 0.88<br/>review = CONFIRMED"]
    fm["<u>fm : FindingManager</u>"]
    mm["<u>mm : MissionManager</u><br/>active = SEARCH"]
    nmm --- lm
    lm --- of
    lm --- fix
    fix --- mp
    mm --- sp
    sp --- plan
    fm --- f1
    fm --- f2
    mm --- fm
```

The values are an illustration of one possible instant, not measurements.

## U5 Component diagram

```mermaid
flowchart LR
    subgraph COMP["«system» Companion software on the Raspberry Pi"]
        direction TB
        C1["«component»<br/>Sensing"]
        C2["«component»<br/>Geo-localisation"]
        C3["«component»<br/>Low-height vision"]
        C4["«component»<br/>Localisation"]
        C5["«component»<br/>Perception"]
        C6["«component»<br/>Navigation mode"]
        C7["«component»<br/>Mission and navigation"]
        C8["«component»<br/>Safety"]
        C9["«component»<br/>App gateway"]
        C10["«component»<br/>FC bridge: mavros"]
    end
    X1["«component»<br/>ArduPilot"]
    X2["«component»<br/>GDN Ground app"]
    X3["«component»<br/>QGroundControl"]
    X4["«artifact»<br/>Map pack"]

    C1 -- "Images, Imu" --> C2
    C1 -- "Images, Imu" --> C3
    C1 -- "Images" --> C5
    X4 -- "tiles, features" --> C2
    C2 -- "GeoFix, Odometry" --> C4
    C3 -- "Odometry, Depth" --> C4
    C4 -- "LocalizationStatus" --> C6
    C5 -- "Detections" --> C7
    C6 -- "NavModeState" --> C7
    C8 -- "SafetyState" --> C7
    C9 -- "SearchArea, FollowTarget" --> C7
    C7 -- "Findings, Status" --> C9
    C4 -- "external navigation" --> C10
    C6 -- "source set command" --> C10
    C7 -- "setpoints" --> C10
    C10 -- "MAVLink 2" --> X1
    C9 -- "WebSocket" --> X2
    X1 -- "MAVLink" --> X3
```

## U6 Composite structure diagram: inside the localisation component

```mermaid
flowchart LR
    subgraph LOCC["Localisation «component»"]
        direction LR
        pin1(("odomIn"))
        pin2(("fixIn"))
        pin3(("fcPoseIn"))
        subgraph PARTS["parts"]
            direction TB
            vm["vioMonitor : VioMonitor"]
            flt["filter : OffsetFilter"]
            conf["confidence : ConfidenceScorer"]
            pub["extNav : ExternalNavPublisher"]
        end
        pout1(("extNavOut"))
        pout2(("statusOut"))
        pout3(("tfOut"))
    end
    GVO["ground_vo"] --> pin1
    OVN["open_vins"] --> pin1
    MMA["map_matcher"] --> pin2
    MAVR["mavros"] --> pin3
    pin1 --> vm
    vm --> flt
    pin2 --> flt
    pin3 --> flt
    vm --> conf
    flt --> conf
    pin3 --> conf
    flt --> pub
    pub --> pout1
    conf --> pout2
    flt --> pout3
    pout1 --> MAVO["mavros: ODOMETRY"]
    pout2 --> NMM["nav_mode_manager"]
    pout3 --> TF["tf: map to odom"]
```

## U7 Deployment diagram

```mermaid
flowchart TB
    subgraph N1["«device» SIYI MK15 ground unit"]
        subgraph E1["«executionEnvironment» Android 9"]
            A1["«artifact» gdn-ground.apk"]
            A2["«artifact» QGroundControl"]
            A3["«artifact» cached map tiles, findings database"]
        end
    end
    subgraph N2["«device» SIYI MK15 air unit"]
        A4["Radio, S.Bus, UART, Ethernet bridge"]
    end
    subgraph N3["«device» Raspberry Pi 5"]
        subgraph E3["«executionEnvironment» Ubuntu 24.04 + ROS 2 Jazzy"]
            A5["«artifact» gdn ROS 2 packages"]
            A6["«artifact» open_vins, mavros, ncnn, libcamera"]
            A7["«artifact» map pack"]
            A8["«artifact» AI models: ground-view, aerial-view"]
            A9["«artifact» calibration files, parameter profiles"]
            A10["«artifact» gdn.service"]
        end
    end
    subgraph N4["«device» Pixhawk 6C"]
        subgraph E4["«executionEnvironment» ArduPilot Copter 4.7"]
            A11["«artifact» parameter files"]
            A12["«artifact» companion_watchdog.lua"]
        end
    end
    subgraph N5["«device» Development workstation"]
        subgraph E5["«executionEnvironment» Ubuntu 24.04"]
            A13["«artifact» ArduPilot SITL, Gazebo Harmonic"]
            A14["«artifact» map_prepare, Kalibr, training tools"]
            A15["«artifact» Android Studio, mock gateway"]
        end
    end
    N1 <-. "2.4 GHz: RC, telemetry, IP" .-> N2
    N2 -- "Ethernet, IP" --- N3
    N2 -- "S.Bus + UART MAVLink" --- N4
    N3 -- "UART, MAVLink 2" --- N4
    N5 -. "Wi-Fi / USB, bench only: install and logs" .-> N3
    N5 -. "USB, bench only: parameters" .-> N4
    N5 -. "USB: install APK" .-> N1
```

## U8 Package diagram

```mermaid
flowchart TB
    subgraph PK0["«package» gdn_interfaces"]
        I["msg, srv, action"]
    end
    subgraph PKA["Drivers"]
        P1["gdn_camera"]
        P2["gdn_imu"]
    end
    subgraph PKB["Localisation"]
        P3["gdn_geoloc"]
        P4["gdn_vio"]
        P5["gdn_vo_simple"]
        P6["gdn_localization"]
    end
    subgraph PKC["Perception"]
        P7["gdn_stereo"]
        P8["gdn_obstacle"]
        P9["gdn_perception"]
        P10["gdn_tracking"]
    end
    subgraph PKD["Decision"]
        P11["gdn_nav_mode"]
        P12["gdn_navigation"]
        P13["gdn_mission"]
        P14["gdn_safety"]
    end
    subgraph PKE["Operator interface"]
        P15["gdn_app_gateway"]
        P16["gdn_telemetry"]
    end
    subgraph PKF["Support"]
        P17["gdn_bringup"]
        P18["gdn_description"]
        P19["gdn_diagnostics"]
        P20["gdn_sim"]
        P21["gdn_tools"]
    end
    subgraph EXT["Third party"]
        X1["mavros"]
        X2["open_vins"]
        X3["ncnn, OpenCV"]
    end
    AND["«package» android/gdn-ground (Kotlin, not ROS)"]

    PKA -.-> PK0
    PKB -.-> PK0
    PKC -.-> PK0
    PKD -.-> PK0
    PKE -.-> PK0
    P4 -.-> X2
    P9 -.-> X3
    P3 -.-> X3
    P17 -.-> PKA
    P17 -.-> PKB
    P17 -.-> PKC
    P17 -.-> PKD
    P17 -.-> PKE
    P17 -.-> X1
    P20 -.-> P18
    AND -. "protocol only, no code dependency" .-> P15
```

Dashed arrows mean "depends on". Project packages depend on each other only through `gdn_interfaces`.

## U9 Profile diagram: stereotypes used in this design

```mermaid
classDiagram
    class Node {
        <<metaclass>>
    }
    class LifecycleNode {
        <<stereotype>>
        +configure()
        +activate()
        +deactivate()
    }
    class ClassA {
        <<stereotype>>
        criticality = localisation
        neverShed = true
        heartbeat = 5 Hz
    }
    class ClassB {
        <<stereotype>>
        criticality = autonomy
        onLoss = hold
    }
    class ClassC {
        <<stereotype>>
        criticality = advisory
        shedFirst = true
    }
    class ThirdParty {
        <<stereotype>>
        unmodified = true
        pinnedVersion : string
    }
    class ProfileGated {
        <<stereotype>>
        activeIn : FlightProfile
    }
    class Topic {
        <<metaclass>>
    }
    class QosProfile {
        <<stereotype>>
        name : SENSOR, ESTIMATE, STATE, COMMAND, EVENT
    }
    Node <|-- LifecycleNode : extension
    Node <|-- ClassA : extension
    Node <|-- ClassB : extension
    Node <|-- ClassC : extension
    Node <|-- ThirdParty : extension
    Node <|-- ProfileGated : extension
    Topic <|-- QosProfile : extension
```

How the stereotypes are applied:

| Node | Stereotypes |
|---|---|
| `map_matcher`, `ground_vo` | «LifecycleNode» «ClassA» «ProfileGated: SEARCH, CRUISE» |
| `open_vins` | «ThirdParty» «ClassA» «ProfileGated: LOW» |
| `localization_manager`, `nav_mode_manager` | «LifecycleNode» «ClassA» |
| `navigator`, `mission_manager`, `search_planner`, `target_tracker` | «LifecycleNode» «ClassB» |
| `detector`, `finding_manager`, `app_gateway`, `video_streamer` | «ClassC» |
| `mavros` | «ThirdParty» «ClassA» |
| `safety_supervisor` | «ClassA» (not lifecycle-managed: always running) |
