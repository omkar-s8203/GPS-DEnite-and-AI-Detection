# 3. Software Diagrams (Raspberry Pi and Flight Controller)

| Field | Value |
|---|---|
| Document ID | GDN-DIA-003 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

One simple picture first, then the layers, then one diagram per software part. Node details are in [node-reference.md](../05-ros2/node-reference.md).

## D3.1 Software in one simple picture

```mermaid
flowchart LR
    IN["Pictures and sensor data"] --> WHERE["Where am I?<br/>localisation"]
    IN --> WHAT["What is there?<br/>AI perception"]
    WHERE --> DECIDE["What should I do?<br/>mode, mission, safety"]
    WHAT --> DECIDE
    APP["Operator's app"] <--> DECIDE
    DECIDE --> TELL["Tell the flight controller<br/>MAVROS"]
    TELL --> FLY["ArduPilot flies"]
    FLY --> WHERE
```

## D3.2 Software layers on the Raspberry Pi

```mermaid
flowchart TB
    subgraph L6["L6 Application"]
        MM["mission_manager"]
        SP["search_planner"]
        FM["finding_manager"]
        AG["app_gateway"]
        VS["video_streamer"]
        TN["telemetry_node"]
    end
    subgraph L5["L5 Decision and supervision"]
        NMM["nav_mode_manager"]
        NAV["navigator"]
        SS["safety_supervisor"]
    end
    subgraph L4["L4 State estimation"]
        MMA["map_matcher"]
        GVO["ground_vo"]
        OV["open_vins"]
        VM["vio_monitor"]
        LM["localization_manager"]
    end
    subgraph L3["L3 Perception"]
        DET["detector"]
        OL["object_localizer"]
        TT["target_tracker"]
        SD["stereo_depth"]
        OS["obstacle_sectors"]
    end
    subgraph L1["L1 Drivers"]
        DC["down_camera"]
        SC["stereo_camera"]
        IMU["imu_driver"]
        SYS["system_monitor"]
    end
    subgraph L0["L0 Flight-controller bridge"]
        MAV["mavros"]
    end
    FCU[["ArduPilot on the Pixhawk"]]
    L1 --> L3
    L1 --> L4
    L3 --> L5
    L4 --> L5
    L5 --> L6
    L4 --> L0
    L5 --> L0
    L0 <--> FCU
```

## D3.3 Complete node graph

```mermaid
flowchart LR
    DC["down_camera"] --> GVO["ground_vo"]
    DC --> MMA["map_matcher"]
    DC --> DET["detector"]
    MAP[("map pack")] --> MMA
    SC["stereo_camera"] --> OV["open_vins"]
    IMU["imu_driver"] --> OV
    SC --> SD["stereo_depth"]
    SC --> DET
    SD --> OS["obstacle_sectors"]
    SD --> OL["object_localizer"]
    DET --> OL
    DET --> FM["finding_manager"]
    DET --> TT["target_tracker"]
    GVO --> VM["vio_monitor"]
    OV --> VM
    VM --> LM["localization_manager"]
    MMA --> LM
    MAV["mavros"] --> LM
    MAV --> NMM["nav_mode_manager"]
    LM --> NMM
    LM --> MAV
    NMM --> NAV["navigator"]
    NMM --> MAV
    OS --> NAV
    OS --> MAV
    TT --> NAV
    SP["search_planner"] --> NAV
    FM --> AG["app_gateway"]
    TT --> AG
    MM["mission_manager"] --> SP
    MM --> NAV
    AG <--> MM
    SS["safety_supervisor"] --> NAV
    SS --> MM
    SYS["system_monitor"] --> SS
    NAV --> MAV
    VS["video_streamer"] --> APPL(["to the app"])
    AG <--> APPL
    DC --> VS
    SC --> VS
    MAV <--> FC[("ArduPilot")]
```

## D3.4 Part: sensing (drivers)

```mermaid
flowchart LR
    subgraph HW["Hardware"]
        D["Downward camera"]
        L["Left sensor"]
        R["Right sensor"]
        I["ICM-20948"]
    end
    D -- "UVC, MJPEG" --> DCN["down_camera node"]
    L --> SCN["stereo_camera node"]
    R --> SCN
    I -- "I2C" --> IN["imu_driver node"]
    DCN --> T1["/down/image_raw, 15 Hz"]
    DCN --> T2["/down/image_full, 1080p"]
    SCN --> T3["/stereo/left and right image_raw, 20 Hz, same stamp"]
    SCN --> T4["/stereo/sync_status: left-right skew"]
    SCN --> T5["/stereo/left/image_color, 10 Hz"]
    IN --> T6["/imu/data_raw, 225 Hz"]
```

## D3.5 Part: satellite map matching

```mermaid
flowchart TB
    A["Downward image"] --> B["Remove lens distortion"]
    ATT["Roll, pitch, heading, height from the flight controller"] --> C
    B --> C["Level the image, turn it north-up, scale to map resolution"]
    C --> D["Contrast normalisation"]
    D --> E["Extract features: SIFT"]
    PRED["Predicted position and uncertainty"] --> W["Choose search window on the map"]
    MAP[("Map pack: tiles + precomputed features")] --> W
    E --> M["Match features, ratio test"]
    W --> M
    M --> R["RANSAC: translation, small rotation, scale"]
    R --> G{"Inliers >= 12 and ratio >= 0.25<br/>and near prediction<br/>and agrees with last fix?"}
    G -- "yes" --> FIX["Accepted fix: position + covariance"]
    G -- "no" --> REJ["Rejected: logged with reason"]
    FIX --> OUT["/geoloc/fix to localization_manager"]
    REJ --> OUT
```

## D3.6 Part: ground visual odometry

```mermaid
flowchart LR
    A["Frame k"] --> T["Track corners to frame k+1: FAST + KLT"]
    B["Frame k+1"] --> T
    GY["Gyro from flight controller"] --> RC["Remove image motion caused by rotation"]
    T --> RC
    RC --> RS["RANSAC fit, reject outliers"]
    H["Height above ground"] --> SC["Pixels to metres: height x pixels / focal length"]
    RS --> SC
    SC --> V["Horizontal velocity and position, 15 Hz"]
    V --> O["/ground_vo/odometry"]
```

## D3.7 Part: localisation fusion

```mermaid
flowchart TB
    GV["ground_vo, above 12 m"] --> SEL["vio_monitor: choose source by height, check health"]
    SV["Stereo VIO, below 12 m"] --> SEL
    SEL -- "smooth but drifting odometry" --> F["Offset filter in localization_manager"]
    GPSP["GPS-based pose from the flight controller, while GPS is good"] -- "measurement" --> F
    FIXP["Map fixes, when GPS is denied"] -- "measurement, gated" --> F
    F --> SLEW["Apply correction slowly: at most 0.5 m/s"]
    SLEW --> POSE["Continuous pose, 20 to 30 Hz"]
    POSE --> EX["/mavros/odometry/out to the flight controller"]
    F --> SIG["Position uncertainty"]
    CHK["Cross-checks against flight controller attitude, gyro, barometer"] --> CONF["Localisation confidence 0 to 1"]
    SIG --> CONF
    CONF --> NM["to nav_mode_manager"]
```

## D3.8 Part: navigation-mode manager

```mermaid
stateDiagram-v2
    [*] --> BOOT
    BOOT --> SENSOR_CHECK
    SENSOR_CHECK --> READY: all checks pass
    SENSOR_CHECK --> FAULT: a check fails
    READY --> GPS_NAV: armed, GPS good
    GPS_NAV --> GPS_DEGRADED: GPS degraded
    GPS_DEGRADED --> GPS_NAV: GPS good for 10 s
    GPS_DEGRADED --> VISION_NAV: GPS denied, vision healthy
    GPS_NAV --> VISION_NAV: GPS denied suddenly
    VISION_NAV --> VISION_DEGRADED: no map fix 20 s or low confidence
    VISION_DEGRADED --> VISION_NAV: fixes return
    VISION_DEGRADED --> FLOW_FALLBACK: vision lost, below 8 m
    VISION_DEGRADED --> LOCALIZATION_LOST: no fix 60 s
    VISION_NAV --> GPS_RECOVERY: GPS good for 10 s
    GPS_RECOVERY --> GPS_NAV: slow, then switch back
    FLOW_FALLBACK --> LOCALIZATION_LOST: flow lost
    LOCALIZATION_LOST --> READY: landed, disarmed
    GPS_NAV --> READY: disarmed
    VISION_NAV --> READY: disarmed
```

## D3.9 Part: AI perception

```mermaid
flowchart TB
    PROF{"Flight profile?"}
    PROF -- "LOW 1 to 10 m" --> G1["Left stereo image, 320 px"]
    PROF -- "SEARCH 25 to 30 m" --> A1["Downward image 1080p, cut into 6 tiles"]
    G1 --> GM["Ground-view YOLO26n, NCNN, 5 Hz"]
    A1 --> AM["Aerial-view YOLO26n, NCNN, about 1 Hz"]
    GM --> GD["/perception/detections"]
    AM --> AD["/perception/aerial_detections"]
    DEPTH["Stereo depth image"] --> OL["object_localizer: distance from depth"]
    GD --> OL
    OL --> OBJ["Objects with class and distance"]
    AD --> PRJ["Project pixel onto the ground"]
    POSE["Drone position, attitude, height"] --> PRJ
    PRJ --> COORD["Objects with latitude and longitude"]
```

## D3.10 Part: grid search

```mermaid
flowchart TB
    REQ["Request from the app: polygon, height, overlap"] --> V{"Inside map, inside fence,<br/>small enough for the battery?"}
    V -- "no" --> NACK["Refuse, with the reason"]
    V -- "yes" --> PLAN["Plan back-and-forth lines"]
    PLAN --> PREV["Show plan in the app, wait for start"]
    PREV --> LINE["Fly next line with GoTo"]
    LINE --> DETQ{"Detection on the ground?"}
    DETQ -- "yes" --> CONF{"Seen in 3 frames<br/>at the same place?"}
    CONF -- "yes" --> PIN["Create finding: class, coordinates, thumbnail"]
    PIN --> SEND["Send to the app"]
    CONF -- "no" --> LINE
    DETQ -- "no" --> MORE
    SEND --> MORE{"More lines?"}
    MORE -- "yes" --> LINE
    MORE -- "no" --> DONE["Hold at the end, report area covered"]
    LINE --> PAUSE{"Localisation degraded<br/>or operator pause?"}
    PAUSE -- "yes" --> HOLD["Hold until it recovers"]
    HOLD --> LINE
```

## D3.11 Part: track and follow

```mermaid
flowchart TB
    TAP["Tap in the app: frame id + point"] --> SEL["Find the detection under the tap in that frame"]
    SEL --> KF["Kalman filter: target position and speed on the ground"]
    DETS["New detections"] --> ASSOC{"A detection near<br/>the predicted position?"}
    KF --> ASSOC
    ASSOC -- "yes" --> UPD["Update the filter"]
    ASSOC -- "no" --> PRED["Predict only"]
    UPD --> TGT["/tracking/target"]
    PRED --> LOSTQ{"Unseen for 3 s?"}
    LOSTQ -- "no" --> TGT
    LOSTQ -- "yes" --> LOST["Target LOST: drone holds"]
    TGT --> FOLLOWQ{"Follow requested?"}
    FOLLOWQ -- "yes" --> CTL["Velocity = target speed + gain x offset, max 3 m/s"]
    CTL --> LIM{"At fence or map edge?"}
    LIM -- "no" --> SETP["Setpoints to the navigator, height fixed at 20 m or more"]
    LIM -- "yes" --> STOP["Hold at the boundary"]
```

## D3.12 Part: navigation and mission

```mermaid
flowchart LR
    APPR["Requests from the app"] --> MM["mission_manager: one behaviour at a time"]
    MM --> B1["SEARCH"]
    MM --> B2["FOLLOW"]
    MM --> B3["GOTO FINDING"]
    MM --> B4["HOLD"]
    B1 --> NAV["navigator"]
    B2 --> NAV
    B3 --> NAV
    B4 --> NAV
    LIMS["Speed limit = min of: mode limit, confidence limit, obstacle limit"] --> NAV
    NAV --> CHK{"Pilot selected GUIDED,<br/>armed, autonomy on,<br/>safety allows?"}
    CHK -- "yes" --> SP["Velocity or position setpoints, 20 Hz"]
    CHK -- "no" --> NONE["Publish nothing"]
    SP --> MAV["mavros to ArduPilot GUIDED mode"]
```

## D3.13 Part: safety

```mermaid
flowchart TB
    L1["1 Pilot: mode switch, sticks, emergency stop"]
    L2["2 Flight controller: RC, battery, EKF failsafes, geofence, GUIDED timeout"]
    L3["3 Flight-controller script: companion watchdog"]
    L4["4 Safety supervisor on the Pi: heartbeats, temperature, load shedding"]
    L5["5 Each node: stale-input holds, range checks"]
    L6["6 Procedure: pre-flight check, test cards, crew"]
    L1 --> L2 --> L3 --> L4 --> L5 --> L6
    HB["Heartbeats of all nodes"] --> SUP["safety_supervisor"]
    TMP["CPU temperature, load, memory"] --> SUP
    SUP --> ST{"Safety level"}
    ST -- "NOMINAL" --> OK["Setpoints allowed"]
    ST -- "DEGRADED" --> SLOW["Allowed, reduced; shed video and AI first"]
    ST -- "HOLD" --> ZERO["Zero velocity only"]
    ST -- "ABORT or FAULT" --> NOSP["No setpoints; flight controller takes over"]
```

## D3.14 Part: app gateway

```mermaid
flowchart LR
    APP(["Android app"]) <-- "WebSocket, JSON" --> AG["app_gateway"]
    AG --> VAL{"Key correct, one client,<br/>numbers in range,<br/>area valid?"}
    VAL -- "no" --> NACK["nack with reason"]
    VAL -- "yes" --> GATE{"Mission gates open?"}
    GATE -- "no" --> NACK
    GATE -- "yes" --> ROS["ROS 2 action or service call"]
    ROS --> ACK["ack"]
    ST["Status, detections, target, search progress, findings, events"] --> AG
    CAM["Selected camera image"] --> VS["video_streamer: 640x480 JPEG, 10 fps"]
    VS -- "WebSocket, binary" --> APP
    TILES[("Map pack tiles")] -- "HTTP, read-only" --> APP
```

## D3.15 Part: flight-controller bridge

```mermaid
flowchart LR
    subgraph PI["Raspberry Pi"]
        LM["localization_manager"] -- "/mavros/odometry/out" --> M["mavros"]
        OS["obstacle_sectors"] -- "/mavros/obstacle/send" --> M
        NAV["navigator"] -- "/mavros/setpoint_raw/local" --> M
        NMM["nav_mode_manager"] -- "/mavros/cmd/command_int" --> M
        TN["telemetry_node"] -- "statustext, named values" --> M
        M -- "state, local position, GPS, EKF status, RC, battery" --> OUT["to all decision nodes"]
    end
    M <-- "UART 921600, MAVLink 2, ENU to NED conversion inside mavros" --> AP["ArduPilot"]
```

## D3.16 Which nodes run in which flight profile

```mermaid
flowchart LR
    subgraph ALWAYS["Always on"]
        A1["mavros, nav_mode_manager, localization_manager, safety_supervisor"]
        A2["navigator, mission_manager, app_gateway, video_streamer"]
    end
    subgraph LOW["LOW profile, below 12 m"]
        B1["stereo_camera, imu_driver, open_vins"]
        B2["stereo_depth, obstacle_sectors"]
        B3["detector with ground-view model"]
    end
    subgraph UPPER["SEARCH and CRUISE profiles, above 12 m"]
        C1["down_camera, ground_vo, map_matcher"]
        C2["detector with aerial-view model, finding_manager, target_tracker"]
    end
    NMM["nav_mode_manager switches by height"] --> LOW
    NMM --> UPPER
```

## D3.17 Processes on the Raspberry Pi

```mermaid
flowchart TB
    SYSD["systemd: gdn.service at boot"] --> LAUNCH["ROS 2 launch: gdn.launch.py profile=flight"]
    LAUNCH --> P1["sensor_container: stereo_camera, rectify, stereo_depth, obstacle_sectors"]
    LAUNCH --> P2["imu_driver"]
    LAUNCH --> P3["open_vins"]
    LAUNCH --> P4["geoloc: down_camera, ground_vo, map_matcher"]
    LAUNCH --> P5["estimation: vio_monitor, localization_manager"]
    LAUNCH --> P6["perception_ai: detector, object_localizer, finding_manager, target_tracker"]
    LAUNCH --> P7["decision: nav_mode_manager, navigator, mission_manager, search_planner"]
    LAUNCH --> P8["safety_supervisor, separate process"]
    LAUNCH --> P9["mavros"]
    LAUNCH --> P10["app: app_gateway, video_streamer"]
    LAUNCH --> P11["support: telemetry_node, system_monitor, diagnostics, rosbag2"]
    LM["lifecycle_manager"] --> P1
    LM --> P4
    LM --> P5
    LM --> P7
```

## D3.18 Software on the flight controller

```mermaid
flowchart TB
    subgraph AP["ArduPilot Copter on the Pixhawk"]
        IN1["GPS"] --> SRC{"EKF source set"}
        IN2["External navigation from the Pi"] --> SRC
        IN3["Optical flow"] --> SRC
        SRC -- "1 GPS, 2 vision, 3 flow" --> EKF["EKF3: 24 states"]
        IMU["IMUs, barometer, compass"] --> EKF
        EKF --> POS["Position controller"]
        GUIDED["GUIDED setpoints from the Pi"] --> POS
        RCIN["Pilot sticks and mode"] --> MODE["Flight mode logic"]
        MODE --> POS
        POS --> ATT["Attitude and rate control"]
        ATT --> MIX["Motor mixer"]
        PRX["Obstacle distances from the Pi"] --> AVD["Simple avoidance: stop"]
        AVD --> POS
        FS["Failsafes: RC, battery, EKF, fence"] --> MODE
        LUA["Lua watchdog: is the Pi alive?"] --> MODE
        LOG["Dataflash log"]
    end
    MIX --> ESC["ESCs"]
```
