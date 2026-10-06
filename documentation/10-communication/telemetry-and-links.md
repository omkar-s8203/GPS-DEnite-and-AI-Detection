# Telemetry, Video and Link Budget

| Field | Value |
|---|---|
| Document ID | GDN-COM-002 |
| Version | 1.0 |
| Date | 2026-10-05 |

## 1. Links in the system

| # | Link | Medium | Protocol | Direction | Criticality |
|---|---|---|---|---|---|
| K1 | RC | MK15 RF → S.Bus | S.Bus | Ground → air | **Flight-critical** |
| K2 | GCS telemetry | MK15 RF → UART | MAVLink 2 | Both | Important (monitoring, failsafe option) |
| K3 | Video | MK15 RF ← Ethernet ← HDMI | RTSP / H.265 | Air → ground | Advisory |
| K4 | Companion ↔ FC | UART | MAVLink 2 | Both | Critical for autonomy; not for flight |
| K5 | Development | Wi-Fi 5 GHz | SSH, DDS | Both | Bench only |
| K6 | Sensors | CSI-2, I²C, UART | — | Sensor → host | Per sensor |

> **DB-3.0 change.** A seventh link is added: **K7, app link** — MK15 Ethernet/IP bridge between the Android app and the Raspberry Pi (WebSocket control, JPEG video, map tiles; advisory criticality). K3 (HDMI video) is retired: the HDMI converter is removed and the app shows the video and draws the overlay itself. QGroundControl continues on K2 unchanged and still receives the status texts and named values below. Design: [ground-app.md](ground-app.md).

## 2. What the operator sees

### On the MK15 (QGroundControl)

| Item | Source |
|---|---|
| Attitude, altitude, speed, battery, GNSS status, flight mode | FC |
| EKF status | FC |
| Proximity (obstacle) display | FC, from companion data |
| Text messages: `CC READY`, `NAV: GPS`, `NAV: GPS DEGRADED`, `NAV: VIO`, `NAV: FLOW`, `LOC LOST - TAKE CONTROL`, `CC FAULT: <reason>`, `PREFLIGHT NO-GO: <reason>` | Companion via FC |
| Named values: `loc_conf` (0–1), `nav_mode` (enum), `vio_feat`, `skew_ms`, `cc_temp` (°C), `obst_m` | Companion via FC |
| Video with overlay | Companion via HDMI converter |

### HUD overlay content

| Element | Purpose |
|---|---|
| Navigation mode banner, colour-coded (green GPS, blue VIO, amber degraded/flow, red lost) | The single most important item for the safety pilot's observer |
| Confidence bar | Trend at a glance |
| Detection boxes with class and range | AI demonstration |
| Nearest obstacle range and bearing marker | Obstacle awareness |
| Battery voltage, flight time | Convenience |
| "AUTONOMY ON/OFF", "PILOT" | Who is commanding |

The pilot flies line-of-sight. A second team member (observer) watches the GCS and calls out mode changes. The HUD is not a piloting aid.

## 3. Status text catalogue

Status texts are limited to 50 characters and to one per second from the companion. Severity follows MAVLink `MAV_SEVERITY`.

| Text | Severity | When |
|---|---|---|
| `CC READY` | INFO | SENSOR_CHECK passed |
| `PREFLIGHT NO-GO: <item>` | WARNING | A pre-flight item failed |
| `NAV: GPS` | INFO | Entered GPS_NAV |
| `NAV: GPS DEGRADED (<reason>)` | NOTICE | Entered GPS_DEGRADED |
| `NAV: VIO (src2)` | NOTICE | Source switch to vision confirmed |
| `NAV: VIO DEGRADED C=<x.xx>` | WARNING | Confidence below 0.7 |
| `NAV: FLOW (src3)` | WARNING | Fell to flow tier |
| `NAV: GPS RECOVERY` | INFO | GNSS good again, validating |
| `NAV: BACK TO GPS step=<x.x>m` | INFO | Returned to source set 1, with observed step |
| `LOC LOST - TAKE CONTROL` | CRITICAL | No horizontal source |
| `OBSTACLE STOP <x.x>m` | NOTICE | Navigator blocked |
| `CC FAULT: <node>` | ERROR | Class-A node failed |
| `CC HOT <T>C - AI OFF` | WARNING | Load shedding |
| `SRC CMD FAILED` | ERROR | EKF source command not acknowledged |
| `MISSION START / DONE / ABORT: <reason>` | INFO / NOTICE | Mission events |

## 4. Bandwidth

### K2 (57 600 baud ≈ 5.7 kB/s)

| Stream | Rate | Approx. load |
|---|---|---|
| Attitude | 4 Hz | 0.15 kB/s |
| Position | 2 Hz | 0.08 kB/s |
| GPS raw, status, battery, EKF | 1–2 Hz | 0.3 kB/s |
| RC channels | 1 Hz | 0.05 kB/s |
| Companion status text | ≤ 1 Hz | ≤ 0.07 kB/s |
| Companion named values | 6 Hz total | 0.2 kB/s |
| Proximity (`DISTANCE_SENSOR`/`OBSTACLE_DISTANCE` forwarded by FC) | 1–2 Hz | 0.2–0.35 kB/s |
| **Total** | | **≈ 1.1–1.3 kB/s** (≈ 22 % of capacity) |

Headroom is needed for parameter downloads and commands. If the MK15 datalink supports a higher baud rate reliably, 115 200 is preferred `[VERIFY]`.

### K4 (921 600 baud ≈ 92 kB/s)

≈ 11 kB/s up, ≈ 10 kB/s down: about 12 % utilisation in each direction ([mavlink-integration.md](mavlink-integration.md) §3–4).

### K3 (video)

Converter output ≈ 12 Mbit/s H.265 `[VENDOR]`. Within the MK15's video capacity at short range. If the RF link degrades, video suffers first; RC and telemetry have priority in the SIYI link `[VERIFY]`.

## 5. Link-loss behaviour summary

| Link lost | Immediate effect | System response |
|---|---|---|
| K1 RC | Pilot cannot command | FC RC failsafe |
| K2 telemetry | Operator blind to data | FC GCS failsafe (optional); flight continues under RC |
| K3 video | No HUD | None |
| K4 companion link | No vision aiding, no setpoints | FC watchdog; EKF source fallback |
| K5 Wi-Fi | — | Not used in flight |
| K1 + K2 together (whole MK15 link) | No pilot control | RC failsafe; tier-dependent action |

## 6. Ground-side data handling

| Data | Captured by | Retrieved |
|---|---|---|
| `.tlog` | QGroundControl on the MK15 | After the session |
| Video recording | HDMI converter microSD (optional) or QGC | After the session |
| Dataflash `.bin` | FC | MAVFTP or SD card |
| rosbag | Pi | Wi-Fi after landing |

All four are stored together per flight under one run ID ([testing-strategy.md](../13-testing/testing-strategy.md) §8).

## 7. Open items

| # | Item |
|---|---|
| TL-1 | Confirm that ArduPilot routes companion `STATUSTEXT` and `NAMED_VALUE_FLOAT` from SERIAL2 to SERIAL1 with the chosen IDs and options |
| TL-2 | Confirm QGC on the MK15 displays named values (MAVLink inspector availability on Android) |
| TL-3 | Measure video latency |
| TL-4 | Decide whether to render the HUD at 720p or 1080p after measuring CPU cost |
