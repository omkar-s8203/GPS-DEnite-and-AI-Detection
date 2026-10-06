# 5. Radio and Telemetry Diagrams

| Field | Value |
|---|---|
| Document ID | GDN-DIA-005 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

Design text: [mavlink-integration.md](../10-communication/mavlink-integration.md), [telemetry-and-links.md](../10-communication/telemetry-and-links.md), [siyi-mk15.md](../03-hardware/siyi-mk15.md).

## D5.1 All links in the system

```mermaid
flowchart LR
    subgraph GND["Ground"]
        ST["Sticks"]
        QGC["QGroundControl"]
        APP["GDN Ground app"]
    end
    subgraph RAD["SIYI MK15 radio link, 2.4 GHz"]
        K1["K1 RC"]
        K2["K2 serial telemetry"]
        K7["K7 IP network"]
    end
    subgraph AIRC["Aircraft"]
        FC["Flight controller"]
        PI["Raspberry Pi"]
        SEN["Cameras, IMU"]
        GPS["GPS, compass, flow"]
    end
    ST --> K1 --> FC
    QGC <--> K2 <--> FC
    APP <--> K7 <--> PI
    PI <-- "K4 on-board UART, MAVLink 2" --> FC
    SEN -- "K6 CSI, USB, I2C" --> PI
    GPS -- "K6 UART, I2C" --> FC
```

| Link | Carries | Importance |
|---|---|---|
| K1 RC | Sticks, flight mode, safety switches | Flight-critical |
| K2 serial telemetry | Standard telemetry to QGroundControl | Important |
| K4 Pi to flight controller | Position, obstacles, setpoints; state back | Needed for autonomy, not for flight |
| K7 IP network | App: video, commands, findings | Advisory |
| K6 sensors | Images, inertial data, GPS | Per sensor |

## D5.2 RC path

```mermaid
flowchart LR
    P(["Pilot"]) --> S["Sticks and switches"]
    S --> GU["MK15 ground unit radio"]
    GU -. "2.4 GHz" .-> AU["MK15 air unit"]
    AU -- "S.Bus, 16 channels" --> RC["Pixhawk RC IN"]
    RC --> AP["ArduPilot RC input"]
    AP --> C14["Ch 1 to 4: roll, pitch, throttle, yaw"]
    AP --> C5["Ch 5: flight mode"]
    AP --> C6["Ch 6: position source 1, 2, 3"]
    AP --> C7["Ch 7: motor emergency stop"]
    AP --> C8["Ch 8: GPS disable, for tests"]
    AP --> C9["Ch 9: autonomy enable, read by the Pi"]
    AU -. "link lost" .-> FSAFE["RC failsafe in the flight controller"]
```

The RC path never passes through the Raspberry Pi or the app.

## D5.3 Telemetry path to QGroundControl

```mermaid
flowchart LR
    AP["ArduPilot"] -- "TELEM1, MAVLink, 57600 baud" --> AU["MK15 air unit UART"]
    AU -. "radio" .-> GU["MK15 ground unit"]
    GU -- "UART, USB COM, Bluetooth or UDP, chosen in SIYI TX" --> QGC["QGroundControl"]
    QGC --> SHOW["Attitude, position, battery, GPS, mode, messages"]
    QGC -- "parameters, commands" --> GU
    PI["Raspberry Pi"] -- "status text and named values" --> AP
    AP -- "forwarded by ArduPilot's MAVLink router" --> AU
```

## D5.4 IP path for the app

```mermaid
flowchart LR
    subgraph NET["Network 192.168.144.0/24"]
        A["Android system .20"]
        G["Ground unit .12"]
        U["Air unit .11"]
        P["Raspberry Pi .50"]
    end
    APP["GDN Ground app"] --- A
    A --- G
    G -. "radio" .- U
    U -- "Ethernet cable" --- P
    P --> PORT1["8765 control: WebSocket, JSON"]
    P --> PORT2["8766 video: WebSocket, JPEG frames"]
    P --> PORT3["8767 map tiles: HTTP, read-only"]
```

## D5.5 What the Raspberry Pi and the flight controller say to each other

```mermaid
flowchart LR
    subgraph PI["Raspberry Pi to flight controller"]
        direction TB
        U1["HEARTBEAT, 1 Hz"]
        U2["ODOMETRY: position from vision, 20 to 30 Hz"]
        U3["OBSTACLE_DISTANCE, 10 Hz"]
        U4["SET_POSITION_TARGET_LOCAL_NED: setpoints, 20 Hz"]
        U5["MAV_CMD_SET_EKF_SOURCE_SET: 1, 2 or 3"]
        U6["Mode request: BRAKE, LOITER, LAND only"]
        U7["STATUSTEXT, NAMED_VALUE_FLOAT"]
        U8["TIMESYNC"]
    end
    subgraph FC["Flight controller to Raspberry Pi"]
        direction TB
        D1["HEARTBEAT: armed, mode"]
        D2["LOCAL_POSITION_NED, ATTITUDE, 30 Hz"]
        D3["GPS_RAW_INT, 5 Hz"]
        D4["EKF_STATUS_REPORT, 5 Hz"]
        D5["DISTANCE_SENSOR, OPTICAL_FLOW"]
        D6["RC_CHANNELS"]
        D7["BATTERY_STATUS"]
        D8["COMMAND_ACK, STATUSTEXT"]
    end
    PI == "UART 921600 baud, MAVLink 2" ==> FC
    FC == "same cable, other direction" ==> PI
```

Never sent by the Pi: arm or disarm, attitude or motor commands, RC override, a request to enter GUIDED.

## D5.6 Switching the position source

```mermaid
sequenceDiagram
    participant NM as nav_mode_manager
    participant M as mavros
    participant FC as ArduPilot
    participant Q as QGroundControl
    participant A as App
    NM->>M: command 42007, source set 2
    M->>FC: COMMAND_INT
    FC-->>M: COMMAND_ACK accepted
    FC-->>Q: STATUSTEXT, source changed
    M-->>NM: result
    NM->>M: STATUSTEXT "NAV: VIO (src2)"
    M->>FC: forwarded
    FC-->>Q: shown in the message list
    NM->>A: event through app_gateway
    Note over NM: No ACK in 1 s, retry up to 3 times
```

## D5.7 How information reaches the people

```mermaid
flowchart TB
    subgraph SRC["Sources"]
        S1["Flight controller state"]
        S2["Pi: navigation mode, confidence, faults"]
        S3["Pi: video, detections, findings, map"]
    end
    S1 -- "K2 telemetry" --> QGC["QGroundControl: instruments"]
    S2 -- "K4, then K2: status text" --> QGC
    S2 -- "K7" --> APP["App: banner and status"]
    S3 -- "K7" --> APP
    QGC --> OBS(["Operator / observer"])
    APP --> OBS
    OBS -- "calls out mode changes" --> PIL(["Pilot, eyes on the drone"])
```

## D5.8 What each link loss leads to

```mermaid
flowchart TB
    A["K1 RC lost"] --> A1["Flight-controller RC failsafe: RTL on GPS flights, LAND on GPS-denied flights"]
    B["K2 telemetry lost"] --> B1["Operator loses instruments; flight continues under RC"]
    C["K7 app link lost"] --> C1["Mission rule: follow holds after 10 s, search continues then holds"]
    D["K4 Pi link lost"] --> D1["Lua watchdog: BRAKE, then LOITER or ALT_HOLD; setpoint timeout stops the drone"]
    E["K1, K2 and K7 together: whole MK15 link"] --> E1["RC failsafe decides"]
    F["Downward camera lost"] --> F1["No map fixes: hold, then pilot or land; restore GPS"]
```

## D5.9 Message rates and link capacity

```mermaid
flowchart LR
    subgraph K4["K4 Pi to flight controller: 92 kB/s available"]
        direction TB
        X1["Up: about 11 kB/s"]
        X2["Down: about 10 kB/s"]
        X3["About 12 % used"]
    end
    subgraph K2["K2 telemetry: 5.7 kB/s available"]
        direction TB
        Y1["About 1.2 kB/s"]
        Y2["About 22 % used"]
    end
    subgraph K7["K7 app link"]
        direction TB
        Z1["Video: 2 to 3 Mbit/s"]
        Z2["Control and status: under 50 kbit/s"]
    end
```

All figures are design estimates.

## D5.10 Time synchronisation

```mermaid
flowchart LR
    CAMT["Camera frame time, Pi clock"] --> STAMP["Message timestamps"]
    IMUT["IMU sample time, same Pi clock"] --> STAMP
    STAMP --> MAVR["mavros time sync"]
    FCT["Flight-controller clock"] <-- "TIMESYNC messages" --> MAVR
    MAVR --> OUT["ODOMETRY time in the flight controller's time base"]
    OUT --> DELAY["VISO_DELAY_MS on the flight controller compensates processing delay"]
```
