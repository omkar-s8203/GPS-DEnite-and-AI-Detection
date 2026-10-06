# 8. UML Behavioural Diagrams

| Field | Value |
|---|---|
| Document ID | GDN-DIA-008 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

UML 2 defines seven behavioural diagram types. All seven are given here. Mermaid draws sequence and state diagrams natively; use case, activity, communication and interaction-overview diagrams are drawn with its flowchart notation using UML conventions. UML timing diagrams have no Mermaid equivalent, so U24 uses a time-axis bar chart to show the same information.

| # | UML diagram | In this document |
|---|---|---|
| 1 | Use case | U10, U11 |
| 2 | Activity | U12, U13, U14 |
| 3 | State machine | U15, U16, U17, U18 |
| 4 | Sequence | U19, U20, U21, U22, U22b |
| 5 | Communication | U23 |
| 6 | Interaction overview | U25 |
| 7 | Timing | U24 |

## U10 Use case diagram: whole system

```mermaid
flowchart LR
    PILOT(("Safety pilot"))
    OPER(("Operator"))
    MAINT(("Maintainer"))
    FCA(("Flight controller"))

    subgraph SYS["GPS-denied drone system"]
        direction TB
        UC1(["Fly manually and take over"])
        UC2(["Arm and select flight mode"])
        UC3(["Simulate GPS loss"])
        UC4(["Monitor flight in the app"])
        UC5(["Run pre-flight check"])
        UC6(["Run a grid search"])
        UC7(["Review findings"])
        UC8(["Track an object"])
        UC9(["Follow an object from above"])
        UC10(["Stop / hold"])
        UC11(["Navigate without GPS"])
        UC12(["Detect GPS loss"])
        UC13(["Find position from the satellite map"])
        UC14(["Detect objects"])
        UC15(["Prepare a map pack"])
        UC16(["Calibrate cameras"])
        UC17(["Analyse flight logs"])
    end

    PILOT --- UC1
    PILOT --- UC2
    PILOT --- UC3
    OPER --- UC4
    OPER --- UC5
    OPER --- UC6
    OPER --- UC7
    OPER --- UC8
    OPER --- UC9
    OPER --- UC10
    MAINT --- UC15
    MAINT --- UC16
    MAINT --- UC17
    UC11 --- FCA
    UC1 --- FCA

    UC6 -. "include" .-> UC11
    UC9 -. "include" .-> UC8
    UC9 -. "include" .-> UC11
    UC11 -. "include" .-> UC12
    UC11 -. "include" .-> UC13
    UC6 -. "include" .-> UC14
    UC8 -. "include" .-> UC14
    UC7 -. "extend" .-> UC6
    UC1 -. "extend: interrupts" .-> UC6
    UC1 -. "extend: interrupts" .-> UC9
```

## U11 Use case diagram: the Android app

```mermaid
flowchart LR
    OPER(("Operator"))
    GW(("app_gateway on the Pi"))
    subgraph APP["GDN Ground app"]
        direction TB
        A1(["Connect to the drone"])
        A2(["Watch live video with detections"])
        A3(["See drone on the map"])
        A4(["Draw a search area"])
        A5(["Start, pause, resume, abort search"])
        A6(["Tap an object to track"])
        A7(["Start follow"])
        A8(["Press Stop"])
        A9(["Review a finding"])
        A10(["Go to a finding"])
        A11(["Export findings"])
        A12(["Run pre-flight check"])
        A13(["Set start position, no GPS"])
    end
    OPER --- A1
    OPER --- A2
    OPER --- A3
    OPER --- A4
    OPER --- A5
    OPER --- A6
    OPER --- A7
    OPER --- A8
    OPER --- A9
    OPER --- A10
    OPER --- A11
    OPER --- A12
    OPER --- A13
    A1 --- GW
    A5 --- GW
    A6 --- GW
    A7 --- GW
    A8 --- GW
    A10 --- GW
    A12 --- GW
    A5 -. "include" .-> A4
    A7 -. "include" .-> A6
    A10 -. "extend" .-> A9
```

## U12 Activity diagram: a mission that loses GPS

```mermaid
flowchart TB
    S(("start")) --> A1["Power on; nodes start"]
    A1 --> D1{"Pre-flight check GO?"}
    D1 -- "no" --> A2["Show reasons; fix"] --> D1
    D1 -- "yes" --> A3["Pilot arms and takes off on GPS"]
    A3 --> A4["Climb to mission height"]
    A4 --> A5["Fly mission on GPS; map matching runs in shadow"]
    A5 --> D2{"GPS still good?"}
    D2 -- "yes" --> D5{"Mission finished?"}
    D5 -- "no" --> A5
    D2 -- "no" --> A6["Declare GPS denied after 2 s"]
    A6 --> D3{"Map fix or odometry healthy?"}
    D3 -- "yes" --> A7["Switch flight controller to source set 2"]
    A7 --> A8["Continue mission on map position, reduced speed"]
    A8 --> D4{"Fixes still arriving?"}
    D4 -- "yes" --> D6{"GPS back for 10 s?"}
    D6 -- "no" --> D7{"Mission finished?"}
    D7 -- "no" --> A8
    D6 -- "yes" --> A9["Slow down; switch back to source set 1"] --> A5
    D4 -- "no, 20 s" --> A10["Hold position"]
    A10 --> D8{"Fix returns within 60 s?"}
    D8 -- "yes" --> A8
    D8 -- "no" --> A11["Alert pilot: take control"]
    D3 -- "no" --> A11
    A11 --> A12["Pilot flies down, or automatic landing"]
    D5 -- "yes" --> A13["Return and land"]
    D7 -- "yes" --> A13
    A12 --> E(("end"))
    A13 --> E
```

## U13 Activity diagram: grid search, with swimlanes

```mermaid
flowchart LR
    subgraph OPL["Operator and app"]
        direction TB
        O1["Draw area, set height"] --> O2["Press Start search"]
        O5["See plan preview"] --> O6["Confirm"]
        O9["See pin and thumbnail"] --> O10["Confirm or reject"]
        O12["See area covered"]
    end
    subgraph PIL["Raspberry Pi"]
        direction TB
        P3{"Area valid and gates open?"}
        P4["Plan lines"]
        P7["Fly a line"]
        P8{"Object confirmed in 3 frames?"}
        P8b["Create finding with coordinates"]
        P11{"More lines?"}
        P13["Hold at the end"]
    end
    subgraph FCL["Flight controller"]
        direction TB
        F1["Follow setpoints in GUIDED"]
        F2["Hold position"]
    end
    O2 --> P3
    P3 -- "no: reason" --> O1
    P3 -- "yes" --> P4 --> O5
    O6 --> P7
    P7 --> F1
    F1 --> P8
    P8 -- "yes" --> P8b --> O9
    P8 -- "no" --> P11
    O10 --> P11
    P11 -- "yes" --> P7
    P11 -- "no" --> P13 --> F2 --> O12
```

## U14 Activity diagram: one map-matching cycle

```mermaid
flowchart TB
    S(("start")) --> A["Take the newest downward image"]
    A --> B{"Height and tilt within limits?"}
    B -- "no" --> R1["Reject: height or tilt"] --> E(("end"))
    B -- "yes" --> C["Undistort, level, north-up, scale"]
    C --> D["Extract features"]
    D --> F{"Predicted position inside the map?"}
    F -- "no" --> R2["Reject: out of coverage"] --> E
    F -- "yes" --> G["Match against map features in the window"]
    G --> H["RANSAC geometric check"]
    H --> I{"Enough inliers?"}
    I -- "no" --> R3["Reject: few inliers"] --> E
    I -- "yes" --> J{"Near the prediction?"}
    J -- "no" --> R4["Reject: gate"] --> E
    J -- "yes" --> K["Publish accepted fix"]
    K --> L["Offset filter update, applied slowly"]
    L --> E
```

## U15 State machine diagram: navigation mode

```mermaid
stateDiagram-v2
    [*] --> BOOT
    BOOT --> SENSOR_CHECK: nodes configured
    SENSOR_CHECK --> READY: checks pass
    SENSOR_CHECK --> FAULT: check fails
    READY --> GPS_NAV: armed [GPS good]
    READY --> VISION_NAV: armed [GPS denied, start position set]
    state "On GPS" as ONGPS {
        GPS_NAV --> GPS_DEGRADED: GPS degraded / freeze alignment
        GPS_DEGRADED --> GPS_NAV: GPS good 10 s
    }
    GPS_DEGRADED --> VISION_NAV: GPS denied [vision healthy] / set source 2
    GPS_NAV --> VISION_NAV: GPS denied suddenly / set source 2
    state "On vision" as ONVIS {
        VISION_NAV --> VISION_DEGRADED: fix age over 20 s or confidence low / hold
        VISION_DEGRADED --> VISION_NAV: fixes back 5 s
    }
    VISION_DEGRADED --> FLOW_FALLBACK: vision lost [below 8 m] / set source 3
    VISION_DEGRADED --> LOCALIZATION_LOST: fix age over 60 s
    VISION_NAV --> GPS_RECOVERY: GPS good 10 s
    VISION_DEGRADED --> GPS_RECOVERY: GPS good 10 s
    GPS_RECOVERY --> GPS_NAV: slow / set source 1
    GPS_RECOVERY --> VISION_NAV: GPS bad again
    FLOW_FALLBACK --> LOCALIZATION_LOST: flow lost
    FLOW_FALLBACK --> GPS_RECOVERY: GPS good 10 s
    LOCALIZATION_LOST --> GPS_RECOVERY: GPS good 10 s
    LOCALIZATION_LOST --> READY: landed and disarmed
    FAULT --> SENSOR_CHECK: reset while disarmed
```

## U16 State machine diagram: search mission

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> PLANNED: search_start [area valid]
    IDLE --> IDLE: search_start [area invalid] / nack
    PLANNED --> RUNNING: operator confirms
    PLANNED --> IDLE: cancel
    RUNNING --> PAUSED: operator pause
    RUNNING --> PAUSED: localisation degraded
    PAUSED --> RUNNING: resume [localisation good]
    RUNNING --> DONE: last line finished
    RUNNING --> ABORTED: pilot changes mode
    RUNNING --> ABORTED: localisation lost
    RUNNING --> ABORTED: battery reserve reached
    PAUSED --> ABORTED: operator abort
    DONE --> IDLE: acknowledged
    ABORTED --> IDLE: acknowledged
```

## U17 State machine diagram: tracked target

```mermaid
stateDiagram-v2
    [*] --> NONE
    NONE --> TRACKING: select [detection under tap]
    NONE --> NONE: select [nothing under tap] / nack
    TRACKING --> TRACKING: detection associated / update filter
    TRACKING --> LOST: unseen for 3 s
    LOST --> TRACKING: detection inside gate
    LOST --> ENDED: unseen for 15 s / report last position
    TRACKING --> ENDED: clear_target
    ENDED --> NONE
    state TRACKING {
        [*] --> NOT_FOLLOWING
        NOT_FOLLOWING --> FOLLOWING: follow_start [gates open]
        FOLLOWING --> NOT_FOLLOWING: hold or boundary reached
    }
```

## U18 State machine diagram: a finding

```mermaid
stateDiagram-v2
    [*] --> CANDIDATE: first detection
    CANDIDATE --> CANDIDATE: seen again at the same place
    CANDIDATE --> DISCARDED: not seen again in time
    CANDIDATE --> UNREVIEWED: seen in 3 frames / create pin, send to app
    UNREVIEWED --> MERGED: same class within 4 m of an existing finding
    UNREVIEWED --> CONFIRMED: operator confirms
    UNREVIEWED --> REJECTED: operator rejects
    CONFIRMED --> [*]: exported
    REJECTED --> [*]
    DISCARDED --> [*]
    MERGED --> [*]
```

## U19 Sequence diagram: hand-over from GPS to the map

```mermaid
sequenceDiagram
    participant FC as ArduPilot
    participant M as mavros
    participant NM as nav_mode_manager
    participant LM as localization_manager
    participant MM as map_matcher
    participant GV as ground_vo
    Note over FC,GV: GPS good. Map matching runs in shadow.
    loop every second
        MM->>LM: GeoFix (compared with GPS, logged)
    end
    GV->>LM: odometry 15 Hz
    LM->>M: aligned pose 20 to 30 Hz
    M->>FC: ODOMETRY (logged, not used)
    FC->>M: GPS_RAW_INT, fix lost
    M->>NM: GPS quality
    NM->>NM: DEGRADED after 1 s
    NM->>LM: freeze alignment, 5 s look-back
    NM->>NM: DENIED after 2 s
    NM->>LM: vision healthy?
    LM-->>NM: confidence HIGH, fix age 0.8 s
    NM->>M: SET_EKF_SOURCE_SET 2
    M->>FC: COMMAND_INT
    FC-->>M: ACK
    Note over FC: EKF now uses external navigation
    MM->>LM: GeoFix now updates the offset filter
    LM->>M: pose continues
    NM->>M: STATUSTEXT NAV VIO
```

## U20 Sequence diagram: grid search from the app

```mermaid
sequenceDiagram
    actor OP as Operator
    participant APP as App
    participant GW as app_gateway
    participant SP as search_planner
    participant NAV as navigator
    participant DET as detector
    participant FM as finding_manager
    participant FC as ArduPilot
    OP->>APP: draw area, press Start
    APP->>GW: search_start (polygon, height, overlap)
    GW->>GW: validate, check gates
    GW->>SP: SearchArea goal
    SP->>SP: plan lines, estimate time
    SP-->>GW: accepted, plan
    GW-->>APP: ack with plan
    OP->>APP: confirm
    loop each line
        SP->>NAV: GoTo end of line
        NAV->>FC: velocity setpoints 20 Hz
        DET->>FM: aerial detections
        FM->>FM: project to ground, confirm over 3 frames
        opt object confirmed
            FM->>GW: Finding
            GW->>APP: finding (class, coordinates, thumbnail)
        end
        SP->>GW: progress
        GW->>APP: search status
    end
    SP->>NAV: hold
    SP-->>GW: completed, area covered
    GW->>APP: search done
```

## U21 Sequence diagram: follow from above

```mermaid
sequenceDiagram
    actor OP as Operator
    participant APP as App
    participant GW as app_gateway
    participant TT as target_tracker
    participant MM as mission_manager
    participant NAV as navigator
    participant FC as ArduPilot
    OP->>APP: tap object, press Track
    APP->>GW: select_target (frame id, x, y)
    GW->>TT: select
    TT-->>GW: track id
    GW-->>APP: ack
    OP->>APP: press Follow
    APP->>GW: follow_start
    GW->>MM: request follow
    MM->>MM: gates open? height at least 20 m?
    MM->>NAV: FollowTarget goal
    loop while following
        TT->>NAV: target position and speed
        NAV->>NAV: velocity = target speed + gain x offset
        NAV->>FC: velocity setpoint
        NAV->>GW: offset, target visible
        GW->>APP: target message
    end
    alt target lost for 3 s
        TT->>NAV: LOST
        NAV->>FC: zero velocity
        NAV-->>MM: result TARGET_LOST after 15 s
        MM->>GW: event
        GW->>APP: target lost, last position
    else operator presses Stop
        APP->>GW: hold
        GW->>MM: hold
        MM->>NAV: cancel
        NAV->>FC: zero velocity
    end
```

## U22 Sequence diagram: start-up to READY

```mermaid
sequenceDiagram
    participant SD as systemd
    participant L as launch
    participant LC as lifecycle_manager
    participant N as managed nodes
    participant SS as safety_supervisor
    participant NM as nav_mode_manager
    participant FC as ArduPilot
    participant APP as App
    SD->>L: start gdn.service
    L->>N: spawn processes
    L->>LC: start
    LC->>N: configure in order
    N-->>LC: inactive
    LC->>N: activate drivers, estimation, decision
    N-->>SS: heartbeats
    NM->>FC: wait for HEARTBEAT
    FC-->>NM: connected
    NM->>FC: request stream rates
    NM->>NM: SENSOR_CHECK
    SS->>SS: pre-flight checks
    alt all pass
        NM->>FC: STATUSTEXT CC READY
        SS->>APP: GO (through app_gateway)
    else something fails
        NM->>FC: STATUSTEXT PREFLIGHT NO-GO with reason
        SS->>APP: NO-GO with reasons
    end
```

## U22b Sequence diagram: the Raspberry Pi fails in flight

```mermaid
sequenceDiagram
    participant PI as Raspberry Pi
    participant FC as ArduPilot
    participant LUA as Lua watchdog
    participant Q as QGroundControl
    actor P as Pilot
    PI->>FC: HEARTBEAT, ODOMETRY, setpoints
    Note over PI: crash or power loss
    PI--xFC: nothing more arrives
    FC->>FC: setpoint timeout, vehicle stops
    LUA->>LUA: no companion heartbeat for 2 s
    LUA->>FC: set mode BRAKE
    LUA->>Q: STATUSTEXT companion lost
    alt position source still healthy
        LUA->>FC: set mode LOITER
    else on vision, below 8 m
        LUA->>FC: select source set 3, flow
    else no position source
        LUA->>FC: ALT_HOLD
    end
    Q-->>P: observer calls take over
    P->>FC: mode switch, flies manually or restores GPS
```

## U23 Communication diagram: one accepted map fix

Objects and the numbered messages between them.

```mermaid
flowchart LR
    DC["down_camera"]
    MM["map_matcher"]
    MP["site : MapPack"]
    MV["mavros"]
    LM["localization_manager"]
    OF["filter : OffsetFilter"]
    NM["nav_mode_manager"]
    FC["ArduPilot"]
    DC -- "1: image" --> MM
    MV -- "2: attitude, height" --> MM
    LM -- "3: predicted position, sigma" --> MM
    MM -- "4: featuresIn(window)" --> MP
    MP -- "4.1: features" --> MM
    MM -- "5: GeoFix accepted" --> LM
    LM -- "6: update(z, R)" --> OF
    OF -- "6.1: offset, sigma" --> LM
    LM -- "7: LocalizationStatus" --> NM
    LM -- "8: pose" --> MV
    MV -- "9: ODOMETRY" --> FC
```

## U24 Timing diagram: one second of localisation

Time runs left to right in milliseconds. Each row is one participant.

```mermaid
gantt
    title One localisation cycle (design estimate, milliseconds)
    dateFormat x
    axisFormat %L
    section Downward camera
    Frame exposed and delivered          :c1, 0, 30ms
    Next frames at 15 Hz                 :c2, 66, 30ms
    section ground_vo
    Track, fit, scale                    :g1, 30, 25ms
    Odometry published                   :milestone, g2, 55, 0ms
    section map_matcher
    Orthorectify                         :m1, 30, 40ms
    Features                             :m2, 70, 60ms
    Match and RANSAC                     :m3, 130, 50ms
    Gates                                :m4, 180, 10ms
    Fix published                        :milestone, m5, 190, 0ms
    section localization_manager
    Filter update                        :l1, 190, 5ms
    Offset slewed in over the next steps :l2, 195, 400ms
    section mavros and UART
    Pose sent every 33 to 50 ms          :u1, 55, 10ms
    Pose with new fix sent               :u2, 195, 10ms
    section ArduPilot
    Fused after VISO_DELAY_MS            :a1, 205, 80ms
```

All durations are design estimates to be replaced by measurements.

## U25 Interaction overview diagram: a complete demonstration flight

Each box refers to a sequence or activity diagram above.

```mermaid
flowchart TB
    S(("start")) --> I1["ref: U22 Start-up to READY"]
    I1 --> D1{"GO?"}
    D1 -- "no" --> I1
    D1 -- "yes" --> I2["Pilot arms, takes off on GPS, climbs to search height"]
    I2 --> I3["Pilot switches GPS off (simulated)"]
    I3 --> I4["ref: U19 Hand-over from GPS to the map"]
    I4 --> D2{"Holding on the map position?"}
    D2 -- "no" --> I9["ref: U12 fallback branch: pilot takes control"]
    D2 -- "yes" --> I5["ref: U20 Grid search from the app"]
    I5 --> D3{"Finding to inspect?"}
    D3 -- "yes" --> I6["Go to the finding and hold"]
    D3 -- "no" --> I7
    I6 --> I7["ref: U21 Follow from above"]
    I7 --> I8["Pilot switches GPS on; return and land"]
    I9 --> I8
    I8 --> E(("end"))
    X["ref: U22b Raspberry Pi fails"] -. "can interrupt any step" .-> I9
```
