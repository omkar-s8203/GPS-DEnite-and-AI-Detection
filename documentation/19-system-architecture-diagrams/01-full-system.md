# 1. Full System Diagrams

| Field | Value |
|---|---|
| Document ID | GDN-DIA-001 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

Everything in one view first, then the same system seen from four angles.

## D1.1 Full system: hardware, software, Raspberry Pi and Android app

```mermaid
flowchart LR
    subgraph GROUND["Ground"]
        direction TB
        PILOT(["Safety pilot"])
        OPER(["Operator"])
        subgraph MK15G["SIYI MK15 ground unit - Android 9"]
            direction TB
            STICKS["Sticks and switches"]
            QGC["QGroundControl"]
            APP["GDN Ground app - Kotlin"]
        end
        PILOT --> STICKS
        OPER --> APP
        OPER --> QGC
    end

    subgraph AIR["Aircraft"]
        direction TB
        AU["MK15 air unit"]

        subgraph FCB["Pixhawk 6C - ArduPilot Copter"]
            direction TB
            EKF["EKF3 state estimator"]
            CTRL["Position and attitude control"]
            FS["Failsafes, geofence, Lua watchdog"]
        end

        subgraph PIB["Raspberry Pi 5 - Ubuntu 24.04 - ROS 2 Jazzy"]
            direction TB
            DRV["Camera and IMU drivers"]
            GEO["Map matcher + ground odometry"]
            VIO["Stereo odometry + depth"]
            AI["Object detector"]
            LOC["Localisation manager"]
            NMM["Navigation-mode manager"]
            MIS["Mission: search, track, follow"]
            NAV["Navigator"]
            SAF["Safety supervisor"]
            GW["App gateway + video streamer"]
            MAV["MAVROS bridge"]
        end

        DCAM["Downward camera"]
        SCAM["Stereo camera + IMU"]
        GPS["GPS + compass"]
        FLOW["Optical flow + range"]
        MAP[("Satellite map pack")]
        ESC["4 x ESC"]
        MOT["4 x motor"]
        BAT["4S battery + power module"]
    end

    STICKS -. "RC link" .-> AU
    QGC <-. "telemetry link, MAVLink" .-> AU
    APP <-. "IP link: video, commands, findings" .-> AU

    AU -- "S.Bus" --> FCB
    AU <-- "UART, MAVLink" --> FCB
    AU <-- "Ethernet" --> GW

    DCAM -- "USB 2.0" --> DRV
    SCAM -- "2 x CSI + I2C" --> DRV
    MAP --> GEO
    DRV --> GEO
    DRV --> VIO
    DRV --> AI
    GEO --> LOC
    VIO --> LOC
    LOC --> NMM
    AI --> MIS
    NMM --> NAV
    MIS --> NAV
    SAF --> NAV
    GW <--> MIS
    LOC --> MAV
    NMM --> MAV
    NAV --> MAV
    MAV <-- "UART 921600, MAVLink 2" --> FCB

    GPS --> FCB
    FLOW --> FCB
    FCB -- "PWM / DShot" --> ESC --> MOT
    BAT --> FCB
    BAT --> PIB
    BAT --> ESC
    BAT --> AU
```

How to read it: the left side is what people hold; the right side is what flies. Three separate radio paths join them. On the aircraft, the Pixhawk flies and the Raspberry Pi thinks.

## D1.2 The system in five blocks

```mermaid
flowchart LR
    SEE["SEE<br/>cameras, sensors"] --> LOCATE["LOCATE<br/>where am I?<br/>GPS or satellite map"]
    SEE --> UNDERSTAND["UNDERSTAND<br/>what is there?<br/>AI detection"]
    LOCATE --> DECIDE["DECIDE<br/>mode, mission,<br/>safety"]
    UNDERSTAND --> DECIDE
    DECIDE --> ACT["ACT<br/>flight controller,<br/>motors"]
    HUMAN["PEOPLE<br/>pilot, operator, app"] <--> DECIDE
    HUMAN --> ACT
```

## D1.3 Technology stack, bottom to top

```mermaid
flowchart TB
    subgraph L5["Layer 5 - People and apps"]
        A1["Pilot with RC"]
        A2["GDN Ground app"]
        A3["QGroundControl"]
    end
    subgraph L4["Layer 4 - Mission software on the Pi"]
        B1["Search, track, follow, navigator, safety"]
    end
    subgraph L3["Layer 3 - Perception and localisation on the Pi"]
        C1["Map matching, odometry, AI detection, fusion"]
    end
    subgraph L2["Layer 2 - Middleware"]
        D1["ROS 2 Jazzy"]
        D2["MAVROS, MAVLink 2"]
        D3["WebSocket server"]
    end
    subgraph L1["Layer 1 - Operating systems and firmware"]
        E1["Ubuntu 24.04 on the Pi"]
        E2["ArduPilot Copter on the Pixhawk"]
        E3["Android 9 on the MK15"]
    end
    subgraph L0["Layer 0 - Hardware"]
        F1["Raspberry Pi 5, cameras"]
        F2["Pixhawk 6C, GPS, flow sensor, ESCs, motors"]
        F3["SIYI MK15 ground and air units"]
    end
    L5 --> L4 --> L3 --> L2 --> L1 --> L0
```

## D1.4 Who is allowed to do what

```mermaid
flowchart TB
    PILOT(["Pilot"]) -- "always wins" --> FC["Flight controller"]
    FC -- "owns" --> O1["Stabilisation and control"]
    FC -- "owns" --> O2["Failsafes and geofence"]
    FC -- "owns" --> O3["The state used for flying"]
    PI["Raspberry Pi"] -- "supplies position, obstacles" --> FC
    PI -- "requests setpoints, only in GUIDED" --> FC
    PI -- "requests position source 1, 2 or 3" --> FC
    APP["Android app"] -- "requests missions" --> PI
    APP -. "cannot arm, cannot change mode" .-x FC
    PI -. "never arms, never takes control" .-x PILOT
```

## D1.5 The three radio paths

```mermaid
flowchart LR
    subgraph G["MK15 ground unit"]
        S["Sticks"]
        Q["QGroundControl"]
        A["GDN Ground app"]
    end
    subgraph D["Aircraft"]
        F["Flight controller"]
        P["Raspberry Pi"]
    end
    S -- "Path 1: RC, S.Bus" --> F
    Q <-- "Path 2: serial telemetry, MAVLink, 57600 baud" --> F
    A <-- "Path 3: IP network, WebSocket + JPEG video" --> P
    P <-- "On board: UART, MAVLink 2" --> F
```

## D1.6 What happens when GPS is lost (end to end)

```mermaid
sequenceDiagram
    participant GPS as GPS receiver
    participant FC as Flight controller
    participant PI as Raspberry Pi
    participant CAM as Downward camera
    participant APP as Android app
    GPS->>FC: weak or no fix
    FC->>PI: GPS quality report
    PI->>PI: classify GPS as DENIED after 2 s
    CAM->>PI: picture of the ground
    PI->>PI: match picture to the stored satellite map
    PI->>FC: position from the map, 20 to 30 times a second
    PI->>FC: request position source 2
    FC-->>PI: accepted
    FC->>FC: fly using the map position
    PI->>APP: status, GPS LOST, POSITION FROM MAP
    Note over FC,PI: Mission continues at reduced speed
```

## D1.7 Flight profiles by height

```mermaid
flowchart TB
    P3["CRUISE 40 to 60 m<br/>map matching with ordinary satellite imagery<br/>transit flights"]
    P2["SEARCH 25 to 30 m<br/>map matching with a sharp reference<br/>grid search, track, follow, aerial AI"]
    P1["LOW 1 to 10 m<br/>stereo odometry and depth<br/>obstacle stop, ground-view AI, take-off, landing"]
    P1 -- "climb" --> P2 -- "climb" --> P3
    P3 -- "descend" --> P2 -- "descend" --> P1
```
