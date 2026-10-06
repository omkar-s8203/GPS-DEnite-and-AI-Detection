# 2. Hardware Diagrams

| Field | Value |
|---|---|
| Document ID | GDN-DIA-002 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |

First the whole hardware, then the full wiring, then one diagram per part. Pin-level tables are in [low-level-design.md](../03-hardware/low-level-design.md).

## D2.1 Hardware block diagram

```mermaid
flowchart TB
    subgraph POWER["Power"]
        BAT["4S LiPo battery"]
        PM["PM02 power module"]
        PDB["Power distribution board"]
        BEC["5 V 5 A BEC"]
    end
    subgraph COMPUTE["Computers"]
        FC["Pixhawk 6C flight controller"]
        PI["Raspberry Pi 5, 8 GB"]
    end
    subgraph SENSE["Sensors"]
        DCAM["Downward camera, USB, 1080p"]
        SCAM["Stereo camera, 2 x IMX219"]
        IMU["ICM-20948 IMU on the stereo board"]
        GPS["M10 GPS + compass"]
        FLOW["MTF-01 optical flow + range"]
    end
    subgraph LINK["Radio"]
        AU["MK15 air unit"]
        GU["MK15 ground unit"]
    end
    subgraph PROP["Propulsion"]
        ESC["4 x ESC"]
        MOT["4 x motor + propeller"]
    end
    STORE[("microSD / NVMe: OS, logs, map pack")]

    BAT --> PM --> PDB
    PDB --> ESC --> MOT
    PDB --> BEC --> PI
    PDB --> AU
    PM --> FC
    DCAM --> PI
    SCAM --> PI
    IMU --> PI
    GPS --> FC
    FLOW --> FC
    PI <--> FC
    AU <--> FC
    AU <--> PI
    AU <-.-> GU
    FC --> ESC
    PI --- STORE
```

## D2.2 Full hardware connection diagram

Every cable, with its interface.

```mermaid
flowchart LR
    BAT["4S LiPo<br/>12.8 to 16.8 V"]
    PM["PM02"]
    PDB["PDB"]
    BEC["BEC 5.1 V 5 A"]
    FC["Pixhawk 6C"]
    PI["Raspberry Pi 5"]
    AU["MK15 air unit"]
    DCAM["Downward camera"]
    SL["IMX219 left"]
    SR["IMX219 right"]
    IMU["ICM-20948"]
    GPS["M10 GPS + compass"]
    FLOW["MTF-01"]
    E1["ESC 1"] --> M1["Motor 1"]
    E2["ESC 2"] --> M2["Motor 2"]
    E3["ESC 3"] --> M3["Motor 3"]
    E4["ESC 4"] --> M4["Motor 4"]
    FAN["Active cooler"]

    BAT -- "XT60" --> PM
    PM -- "battery voltage" --> PDB
    PM -- "POWER1: 5.2 V + V/I sense" --> FC
    PDB -- "fuse 3 A" --> BEC
    BEC -- "GPIO pins 2,4 and 6,9" --> PI
    PDB -- "fuse 2 A, battery voltage" --> AU
    PDB --> E1
    PDB --> E2
    PDB --> E3
    PDB --> E4

    SL -- "CSI-2 to CAM0" --> PI
    SR -- "CSI-2 to CAM1" --> PI
    IMU -- "I2C-1: GPIO2, GPIO3" --> PI
    DCAM -- "USB 2.0, UVC" --> PI
    PI -- "fan header" --> FAN

    PI -- "GPIO14 TX to TELEM2 RX" --> FC
    FC -- "TELEM2 TX to GPIO15 RX" --> PI
    PI <-- "Ethernet, 100BASE-TX" --> AU

    AU -- "S.Bus to RC IN" --> FC
    AU <-- "UART to TELEM1, 57600" --> FC
    GPS -- "GPS1: UART + I2C" --> FC
    FLOW -- "TELEM3: UART, MAVLink" --> FC
    FC -- "MAIN OUT 1" --> E1
    FC -- "MAIN OUT 2" --> E2
    FC -- "MAIN OUT 3" --> E3
    FC -- "MAIN OUT 4" --> E4
```

Rules shown by this diagram: the Pi and the flight controller have separate 5 V supplies and share only ground and signals; the 5 V pin of TELEM2 is not connected; nothing connects the Pi to the motors.

## D2.3 Power distribution

```mermaid
flowchart TD
    BAT["4S LiPo 5000 to 5200 mAh"] --> PM["PM02: shunt + 5.2 V 3 A regulator"]
    PM --> PDB["Power distribution board"]
    PM -- "5.2 V" --> FC["Pixhawk 6C"]
    FC -- "5 V" --> GPS["GPS + compass"]
    FC -- "5 V" --> FLOW["MTF-01"]
    PDB -- "battery voltage, 18 to 22 A in hover" --> ESC["4 x ESC and motors"]
    PDB -- "battery voltage" --> AU["MK15 air unit, 3 W average"]
    PDB --> BEC["BEC 5.1 V 5 A"]
    BEC -- "5.1 V, about 8 W" --> PI["Raspberry Pi 5"]
    PI -- "3.3 V over CSI" --> SCAM["Stereo camera + IMU"]
    PI -- "5 V USB" --> DCAM["Downward camera"]
    PI -- "5 V" --> FAN["Active cooler"]
```

## D2.4 Raspberry Pi 5: what is plugged where

```mermaid
flowchart LR
    subgraph PI["Raspberry Pi 5"]
        direction TB
        CSI0["CAM/DISP 0"]
        CSI1["CAM/DISP 1"]
        USB2["USB 2.0 port"]
        ETH["Ethernet"]
        UART["GPIO14 TX / GPIO15 RX"]
        I2C["GPIO2 SDA / GPIO3 SCL"]
        PWR["5 V pins 2, 4 and GND 6, 9"]
        FANH["Fan header"]
        SD["microSD / PCIe NVMe"]
        USB3["USB 3.0 ports"]
        HDMI["micro-HDMI"]
        WIFI["Wi-Fi / Bluetooth"]
    end
    SL["Stereo left sensor"] --> CSI0
    SR["Stereo right sensor"] --> CSI1
    DCAM["Downward camera"] --> USB2
    ETH <--> AU["MK15 air unit"]
    UART <--> FC["Pixhawk TELEM2"]
    IMU["ICM-20948"] --> I2C
    BEC["BEC 5.1 V"] --> PWR
    FANH --> FAN["Active cooler"]
    SD --- STORE[("OS, rosbag, map pack")]
    USB3 -.- N1["not used in flight: radio noise"]
    HDMI -.- N2["not used since DB-3.0"]
    WIFI -.- N3["bench only, off in flight"]
```

## D2.5 Pixhawk 6C: what is plugged where

```mermaid
flowchart LR
    subgraph FC["Pixhawk 6C"]
        direction TB
        T1["TELEM1 - SERIAL1"]
        T2["TELEM2 - SERIAL2"]
        T3["TELEM3"]
        G1["GPS1"]
        RC["RC IN"]
        P1["POWER1"]
        MO["MAIN OUT 1 to 4"]
        USB["USB"]
        CAN["CAN1, CAN2, GPS2, I2C"]
        subgraph INT["Inside the case"]
            I1["IMU ICM-42688-P"]
            I2["IMU BMI088"]
            I3["Barometer MS5611"]
            I4["Compass IST8310"]
            I5["microSD: flight log"]
        end
    end
    AU["MK15 air unit datalink"] <-- "MAVLink 57600" --> T1
    PI["Raspberry Pi 5"] <-- "MAVLink 2, 921600" --> T2
    FLOW["MTF-01"] -- "MAVLink 1, 115200" --> T3
    GPS["M10 GPS + compass"] --> G1
    SB["MK15 air unit S.Bus"] --> RC
    PM["PM02"] --> P1
    MO --> ESC["4 x ESC"]
    USB -.- B["bench set-up only"]
    CAN -.- S["spare"]
```

## D2.6 Downward camera

```mermaid
flowchart LR
    GROUND["Ground below the drone"] -. "light" .-> LENS["Wide lens, 100 to 120 degrees"]
    LENS --> SENSOR["2 MP sensor, short exposure"]
    SENSOR --> UVC["UVC interface, MJPEG 1920x1080"]
    UVC -- "USB 2.0, short shielded cable" --> PI["Raspberry Pi 5"]
    PI --> F1["Full resolution: aerial object detector"]
    PI --> F2["Scaled to 640x480: map matcher"]
    PI --> F3["Scaled to 640x480: ground odometry"]
    PI --> F4["640x480 JPEG: video to the app"]
```

## D2.7 Stereo camera and its IMU

```mermaid
flowchart LR
    subgraph BOARD["Waveshare IMX219-83 board"]
        direction TB
        L["IMX219 left"]
        R["IMX219 right"]
        I["ICM-20948 IMU"]
        B["60 mm baseline, no hardware sync"]
    end
    L -- "CSI-2, 2 lanes" --> C0["Pi CAM0"]
    R -- "CSI-2, 2 lanes" --> C1["Pi CAM1"]
    I -- "I2C, 400 kHz, about 225 Hz" --> I2C["Pi I2C-1"]
    C0 --> DRV["stereo_camera node: software sync, same timestamp per pair"]
    C1 --> DRV
    I2C --> IMUD["imu_driver node"]
    DRV --> USE1["Stereo depth: distance ahead"]
    DRV --> USE2["Stereo odometry, low height"]
    DRV --> USE3["Ground-view object detector"]
    IMUD --> USE2
```

## D2.8 SIYI MK15: ground unit and air unit

```mermaid
flowchart LR
    subgraph GU["Ground unit"]
        direction TB
        ST["Sticks, mode switch, aux switches"]
        AND["Android 9: QGroundControl, GDN Ground app, SIYI TX"]
        SCR["5.5 inch touchscreen"]
        RFG["Radio"]
        ST --> RFG
        AND <--> RFG
        AND --> SCR
    end
    subgraph AUN["Air unit"]
        direction TB
        RFA["Radio"]
        SB["S.Bus out, 16 channels"]
        UA["UART datalink"]
        ET["Ethernet port"]
        PW["Power in, battery voltage"]
        RFA --> SB
        RFA <--> UA
        RFA <--> ET
    end
    RFG <-. "2.4 GHz link" .-> RFA
    SB --> FC1["Pixhawk RC IN"]
    UA <--> FC2["Pixhawk TELEM1"]
    ET <--> PI["Raspberry Pi Ethernet"]
    PDB["PDB"] --> PW
```

## D2.9 GPS, compass and flow sensor

```mermaid
flowchart LR
    SAT(["GNSS satellites"]) -. "radio signals" .-> ANT["M10 receiver on a mast"]
    ANT -- "UART: position, velocity" --> FC["Pixhawk GPS1"]
    MAG["IST8310 compass in the same puck"] -- "I2C: heading" --> FC
    subgraph MTF["MTF-01 on the underside"]
        OF["Optical flow imager"]
        TOF["ToF range, up to 8 m"]
    end
    OF -- "MAVLink OPTICAL_FLOW" --> FC3["Pixhawk TELEM3"]
    TOF -- "MAVLink DISTANCE_SENSOR" --> FC3
    FC --> U1["Source set 1: GPS navigation"]
    FC3 --> U3["Source set 3: flow hold, below 8 m only"]
```

## D2.10 Propulsion

```mermaid
flowchart LR
    FC["Pixhawk MAIN OUT 1 to 4"] -- "PWM / DShot" --> ESCS["4 x ESC"]
    PDB["PDB, battery voltage"] --> ESCS
    ESCS -- "3-phase" --> M["4 x brushless motor"]
    M --> PR["4 x propeller, 10 to 11 inch"]
    PR --> TH["Thrust: at least 2 x all-up weight"]
    SW(["Pilot emergency-stop switch"]) -. "RC" .-> FC
```

## D2.11 Where things sit on the airframe (top view)

```mermaid
flowchart TB
    subgraph FRONT["Front"]
        SCAM["Stereo camera, facing forward"]
        PIF["Raspberry Pi + cooler, on a damped plate"]
    end
    subgraph CENTRE["Centre"]
        FCC["Pixhawk, on its damping"]
        GPSM["GPS + compass on a mast, above everything"]
        BATC["Battery, slid to balance"]
    end
    subgraph UNDER["Underside"]
        DC["Downward camera, near the centre"]
        MT["MTF-01 flow sensor"]
    end
    subgraph REAR["Rear"]
        AUR["MK15 air unit, antennas pointing down"]
        BECR["BEC, away from compass"]
    end
    FRONT --- CENTRE --- REAR
    CENTRE --- UNDER
```

## D2.12 Signal types used

```mermaid
flowchart LR
    A["Camera images"] --- A1["CSI-2 and USB 2.0"]
    B["Inertial data for vision"] --- B1["I2C, 400 kHz"]
    C["Drone to flight controller data"] --- C1["UART, MAVLink 2, 921600 baud"]
    D["Pilot commands"] --- D1["S.Bus, 16 channels"]
    E["Ground telemetry"] --- E1["UART, MAVLink, 57600 baud"]
    F["App data and video"] --- F1["Ethernet, IP, WebSocket"]
    G["Motor commands"] --- G1["PWM or DShot"]
    H["Power"] --- H1["Battery voltage, 5.2 V, 5.1 V, 3.3 V"]
```
