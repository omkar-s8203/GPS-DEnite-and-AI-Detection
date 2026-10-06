# High-Level Hardware Architecture

| Field | Value |
|---|---|
| Document ID | GDN-HW-001 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | Baseline |

## 1. Full hardware block diagram

```mermaid
flowchart LR
    subgraph AIR[Airborne]
        direction LR
        BAT[4S LiPo battery]
        PM[PM02 power module<br/>V/I sense + 5.2 V]
        PDB[Power distribution]
        BEC[5 V 5 A BEC]
        ESC[4 x ESC]
        MOT[4 x Motor]

        FC[Pixhawk 6C<br/>ArduPilot Copter]
        GPS[M10 GNSS + compass]
        FLOW[MTF-01<br/>optical flow + ToF]

        PI[Raspberry Pi 5<br/>Ubuntu 24.04 + ROS 2 Jazzy]
        CAM[IMX219-83 stereo camera<br/>+ ICM-20948 IMU]
        DCAM[Downward camera<br/>USB 2.0, wide lens]
        SD[(microSD / NVMe<br/>rosbag2 + satellite map pack)]

        AU[MK15 air unit]
        CONV[HDMI input converter]
    end

    subgraph GND[Ground]
        GU[MK15 ground unit<br/>Android + QGroundControl]
        PC[Dev laptop<br/>RViz2 / logs]
    end

    BAT --> PM --> PDB
    PDB --> ESC --> MOT
    PDB --> BEC --> PI
    PDB --> AU
    PM -- 5.2 V + V/I sense --> FC

    CAM -- "2 x CSI-2" --> PI
    CAM -- "I2C (IMU)" --> PI
    DCAM -- "USB 2.0 (UVC)" --> PI
    PI --- SD
    PI <-- "UART 921600, MAVLink 2" --> FC
    PI -- micro-HDMI --> CONV -- Ethernet --> AU

    GPS -- "UART + I2C" --> FC
    FLOW -- "UART, MAVLink" --> FC
    FC -- "PWM / DShot" --> ESC
    AU -- S.Bus --> FC
    AU <-- "UART 57600, MAVLink" --> FC

    AU <-. "2.4 GHz: RC + telemetry + video" .-> GU
    PI <-. "Wi-Fi, bench only" .-> PC
```

## 2. Perception-to-actuation chain

```mermaid
flowchart TD
    A[Stereo camera + IMU] -->|CSI-2, I2C| B[Raspberry Pi 5]
    B --> C[ROS 2 Jazzy graph]
    C --> D[VIO + localisation manager<br/>navigation-mode manager<br/>navigator]
    D -->|"MAVLink 2: ODOMETRY, OBSTACLE_DISTANCE,<br/>SET_POSITION_TARGET_LOCAL_NED"| E[Flight controller<br/>EKF3 + position/attitude control]
    E -->|PWM / DShot| F[ESCs]
    F -->|3-phase| G[Motors]
    E -->|"MAVLink 2: state, GPS, EKF status"| D
```

The Pi never drives ESCs. There is no electrical path from the Pi to the motors.

## 3. RC path

```mermaid
flowchart LR
    P([Pilot]) --> GU[MK15 ground unit] -. "2.4 GHz" .-> AU[MK15 air unit] -- "S.Bus, 16 ch" --> RCIN[Pixhawk RC IN] --> AP[ArduPilot RC input<br/>modes, sticks, aux switches]
```

- The RC path does not pass through the Pi. Pilot authority is independent of all companion hardware and software.
- Loss of this path triggers the FC throttle/RC failsafe.

Planned channel map (`[ASSUMPTION]`, to be fixed at FC bring-up):

| Channel | Function | ArduPilot parameter |
|---|---|---|
| 1–4 | Roll, pitch, throttle, yaw | `RCMAP_*` |
| 5 | Flight mode (6 positions) | `FLTMODE_CH = 5` |
| 6 | EKF source set (3-position) | `RC6_OPTION = 90` |
| 7 | Motor emergency stop | `RC7_OPTION = 31` |
| 8 | GPS disable (test aid) | `RC8_OPTION = 65` |
| 9 | Companion autonomy enable (read by companion via `RC_CHANNELS`) | `RC9_OPTION = 0` (scripting/none) |

Flight modes on channel 5: STABILIZE, ALT_HOLD, LOITER, GUIDED, LAND, RTL.

## 4. Telemetry path

```mermaid
flowchart LR
    FC[Pixhawk TELEM1] <-- "UART 57600, MAVLink" --> AU[MK15 air unit] <-. "2.4 GHz" .-> GU[MK15 ground unit] --> QGC[QGroundControl]
    PI[Raspberry Pi 5 / MAVROS] <-- "UART 921600, MAVLink 2" --> T2[Pixhawk TELEM2]
    T2 -. "MAVLink routing inside ArduPilot<br/>STATUSTEXT, NAMED_VALUE_FLOAT" .-> FC
```

- The GCS talks to the FC directly. It works with the Pi off.
- Companion status reaches the GCS by MAVLink routing through the FC. High-rate companion-to-FC traffic is kept off the radio with `SERIAL2_OPTIONS` bit 10 ("don't forward") where needed; see [mavlink-integration](../10-communication/mavlink-integration.md).

> **DB-3.0 change to the diagrams in this document.** The HDMI converter and its link are removed. The Raspberry Pi's Ethernet port connects directly to the MK15 air unit's Ethernet port, and an Android app on the ground unit talks to the Pi over that IP link (video, detections, map, commands). RC and the serial telemetry datalink to QGroundControl are unchanged. See [ground-app.md](../10-communication/ground-app.md) §2 for the current link diagram.

## 5. Video path

```mermaid
flowchart LR
    CAM[Left camera image] --> HUD[hud node<br/>overlay: detections, nav mode, confidence]
    HUD --> KMS[Pi HDMI output<br/>KMS, no desktop] --> CONV[SIYI HDMI converter<br/>H.265 hardware encode] -- Ethernet --> AU[MK15 air unit] -. "2.4 GHz" .-> GU[Ground unit display]
```

Rationale: the Pi 5 has no hardware H.264/H.265 encoder. Encoding in software would cost about one CPU core. The MK15 HDMI combo already includes a hardware encoder, so the Pi only has to render frames to its HDMI output.

## 6. GNSS path

```mermaid
flowchart LR
    SAT[(GNSS constellations)] -.-> ANT[M10 receiver + IST8310 compass<br/>on mast] -- "UART (GNSS) + I2C (compass)" --> FC[Pixhawk GPS1] --> EKF[EKF3 source set 1]
    EKF -- "GPS_RAW_INT, EKF_STATUS_REPORT" --> PI[Pi: GNSS health monitor]
```

GNSS goes only to the FC. The Pi observes GNSS quality through MAVLink; it has no receiver of its own.

## 7. Sensor allocation

| Sensor | Connected to | Used for |
|---|---|---|
| FC IMUs (ICM-42688-P, BMI088) | FC internal | Attitude, EKF3 prediction |
| FC barometer (MS5611) | FC internal | Altitude |
| GNSS + compass (M10 / IST8310) | FC GPS1 | Tier-1 position, heading |
| Optical flow + ToF (MTF-01) | FC serial | Height above ground; tier-3 velocity |
| Stereo camera (2 × IMX219) | Pi CSI-2 ×2 | Low-regime VIO, depth, AI |
| Downward camera (DB-2.0) | Pi USB 2.0 | Satellite map matching, ground visual odometry |
| ICM-20948 (on camera board) | Pi I²C | VIO inertial input |

## 8. Power path

```mermaid
flowchart LR
    BAT[4S LiPo<br/>12.8 - 16.8 V] --> PM[PM02]
    PM -->|battery voltage| PDB[PDB]
    PM -->|"5.2 V, 3 A + V/I sense"| FC[Pixhawk POWER1]
    PDB --> ESC[ESCs + motors]
    PDB --> BEC[5 V 5 A BEC] -->|"5.1 V"| PI[Pi 5 via GPIO 5V pins]
    PDB -->|battery voltage| AU[MK15 air unit]
    PDB --> B12[12 V regulator] --> CONV[HDMI converter]
    FC -->|5 V port power| GPS[GNSS]
    FC -->|5 V port power| FLOW[MTF-01]
    PI -->|3.3 V via CSI| CAM[Stereo camera + IMU]
```

Details and protection: [power-architecture.md](power-architecture.md).

## 9. Data logging

| Log | Where | Content | Retrieval |
|---|---|---|---|
| ArduPilot dataflash (`.bin`) | FC microSD | IMU, EKF, GNSS, `VISP`/`VISV` external-nav inputs, source changes, RC, battery | MAVFTP or card removal |
| rosbag2 (MCAP) | Pi storage | Images (optionally compressed/decimated), IMU, VIO, alignment, state machine, detections, diagnostics | scp over Wi-Fi after landing |
| GCS telemetry log (`.tlog`) | MK15 ground unit | MAVLink stream as seen on the ground | USB / SD |
| System journal | Pi | Service start/stop, kernel, throttling | `journalctl` export |

## 10. Physical layout guidance

| Item | Placement | Reason |
|---|---|---|
| Flight controller | Vehicle centre, on its damping, arrow forward | IMU at centre of rotation |
| Stereo camera | Front, rigid bar, clear of props in the image, on a separate soft-mounted plate with the Pi **or** rigidly on a damped sub-frame | Rigid stereo geometry; isolation from motor vibration |
| Raspberry Pi | Close to the camera | CSI ribbon length ≤ 200 mm |
| GNSS/compass | Mast, ≥ 10 cm above everything | Away from the Pi, CSI ribbons, power leads and the MK15 antennas |
| MK15 air unit | Rear, antennas pointing down, apart from each other and from GNSS | RF separation |
| MTF-01 | Underside, clear view down, ≥ 5 cm from landing-gear legs in view | Flow and ToF need unobstructed view |
| Downward camera | Underside near the centre, image top towards the nose, landing gear outside its field of view, on the damped plate | Nadir view for map matching; known boresight |
| Battery | Centred, adjusts centre of gravity | Balance |
| BEC and power leads | Away from compass and CSI ribbons; twisted pairs | Magnetic and conducted noise |
