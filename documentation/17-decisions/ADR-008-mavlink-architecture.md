# ADR-008 — MAVLink Architecture: MAVROS over UART

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

The ROS 2 graph on the Pi must exchange data with ArduPilot: external navigation, obstacle distances, setpoints and commands upward; state, GNSS quality and EKF status downward. Coordinate conventions differ (ENU/FLU vs NED/FRD) and clocks differ.

## Options

| # | Option |
|---|---|
| A | MAVROS (ROS 2) |
| B | ArduPilot native DDS (`AP_DDS`) with the Micro XRCE-DDS agent |
| C | Custom bridge node using pymavlink |
| D | MAVSDK with a custom ROS 2 wrapper |

Physical link options: UART (TELEM2), USB, Ethernet.

## Evaluation

| Criterion | A: MAVROS | B: AP_DDS | C: pymavlink bridge | D: MAVSDK |
|---|---|---|---|---|
| Jazzy availability | Binary (arm64) | ArduPilot docs target Humble | Own code | Own wrapper |
| External odometry input | `odometry`, `vision_pose`, `vision_speed` plugins | `/ap/tf` (odom → base_link) only | Write it | Limited for ArduPilot |
| Obstacle distance | Plugin | Not listed | Write it | — |
| EKF source-set command | Generic command service | Not listed | Write it | — |
| Frame conversion ENU/NED | Built in, well tested | Built in (REP-147 style) | Must write — classic source of errors | Partial |
| Time sync | Built in | Built in | Must write | Built in |
| Access to arbitrary MAVLink messages | Yes | No (fixed topic set) | Yes | Limited |
| CPU cost | Moderate (reduced by plugin allow-list) | Low | Low–moderate (Python) | Moderate |
| Maturity with ArduPilot | High | Growing; some interfaces marked experimental | — | Lower (PX4-centric) |

Physical link:

| Link | Assessment |
|---|---|
| **UART, 921 600 baud, 3-wire** | Standard, documented by ArduPilot for companions, locking connector on the FC side, 3.3 V on both ends. ≈ 12 % utilisation. **Selected.** |
| USB | Not recommended for flight (connector retention; noise) |
| Ethernet | Not available on the Pixhawk 6C |

## Decision

**MAVROS** with a reduced plugin allow-list, connected by **UART (Pi GPIO14/15 ↔ Pixhawk TELEM2) at 921 600 baud, MAVLink 2**. External navigation is sent as the MAVLink `ODOMETRY` message through the MAVROS `odometry` plugin; `VISION_POSITION_ESTIMATE` + `VISION_SPEED_ESTIMATE` is the fallback.

## Reason

1. MAVROS already implements every message this design needs, with tested frame conversion and time synchronisation. Re-implementing these would add the highest-severity class of bug in the FMEA (frame errors) for no gain.
2. It is available as a binary for Jazzy on arm64.
3. It gives access to ArduPilot-specific messages and commands that the native DDS interface does not expose (source-set selection, obstacle distance, EKF status).
4. `ODOMETRY` carries pose, velocity, both covariances, a reset counter and a quality field in one message.

## Consequences

- MAVROS is the single point where ENU/FLU ↔ NED/FRD conversion occurs. No project node performs that conversion.
- MAVROS subscriptions use sensor-data QoS; project nodes must match (checked in integration tests).
- MAVROS must be configured not to publish the `map → base_link` / `odom → base_link` transforms.
- A few items need bench confirmation: that ArduPilot 4.7 accepts `ODOMETRY` with the frame IDs MAVROS emits; that EKF variance values are exposed; and the component-ID convention for the watchdog. Each has a stated fallback.
- The companion never arms, never sends RC overrides, and never selects GUIDED ([mavlink-integration.md](../10-communication/mavlink-integration.md) §2). These prohibitions are enforced by not loading the corresponding plugins or not calling the services, and are verified by code review and tests.
- `AP_DDS` is a candidate for a later revision once it is documented for Jazzy and covers the needed interfaces.
