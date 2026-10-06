# ADR-002 — Autopilot Firmware: ArduPilot

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

The flight controller must fuse vision-based position from a companion computer, switch between GNSS and vision in flight, fall back to a third source, and keep all safety functions. Two open-source autopilots are candidates. The SIYI MK15 supports both.

## Options

| # | Option |
|---|---|
| A | ArduPilot Copter (4.7.x stable) |
| B | PX4 Autopilot (current stable) |

## Evaluation

| Criterion | ArduPilot | PX4 | Weight |
|---|---|---|---|
| External vision fusion | EKF3 ExternalNav; `VISION_POSITION_ESTIMATE`, `VISION_SPEED_ESTIMATE`, `ODOMETRY` | EKF2; `VISION_POSITION_ESTIMATE`, `ODOMETRY` | Equal |
| **Explicit GNSS ↔ non-GNSS switching** | Three configurable EKF source sets; switch by RC, Lua, or `MAV_CMD_SET_EKF_SOURCE_SET`; documented "GPS / Non-GPS transitions" procedure | Concurrent fusion with internal handling; less external control over which source is active | **High — favours ArduPilot** |
| Independent third source | Source set 3 (optical flow), same mechanism | Flow fused concurrently | Favours ArduPilot for testability |
| FC-side scripting | Lua | None (C++ modules) | Favours ArduPilot (watchdog) |
| Obstacle input from companion | `OBSTACLE_DISTANCE` → proximity/avoidance, documented for depth cameras | `OBSTACLE_DISTANCE` → collision prevention | Equal |
| Native ROS 2 | `AP_DDS` exists; docs target Humble; smaller interface | uXRCE-DDS first-class | **Favours PX4** |
| MAVROS support | Mature | Mature | Equal |
| In-flight GNSS-loss test aid | RC auxiliary "GPS Disable" | Failure injection | Slightly favours ArduPilot |
| Simulation | SITL + Gazebo Harmonic plugin | SITL + Gazebo | Equal |
| Local community, GCS familiarity | Strong in Indian colleges (Mission Planner, Pixhawk) | Smaller locally | Favours ArduPilot |
| Licence | GPLv3 | BSD-3 | Irrelevant for an academic project that does not modify firmware |

## Decision

**ArduPilot Copter 4.7.x (stable).**

## Reason

The project's central feature is a controlled, observable transition between position sources. ArduPilot exposes exactly that as a first-class, documented mechanism that the companion can command and the pilot can override with a switch. The mapping between the design's tiers and ArduPilot's source sets is one-to-one, which makes the behaviour easy to explain, test in SITL and log. Lua scripting lets the companion watchdog live on the flight controller. PX4's stronger native ROS 2 bridge is a real loss, but MAVROS provides everything this design needs.

## Consequences

- MAVROS becomes the bridge ([ADR-008](ADR-008-mavlink-architecture.md)).
- With `VISO_TYPE = 1`, ArduPilot does not align the vision frame itself; the companion must do it ([ADR-009](ADR-009-gps-denied-transition.md)).
- Parameter names and enumerations in this documentation must be validated against Copter 4.7 at bring-up.
- A flight controller with ≥ 2 MB flash is required for scripting and visual-odometry features ([ADR-003](ADR-003-flight-controller.md)).
- **Fallback:** if a blocking ArduPilot issue appears, PX4 on the same Pixhawk 6C is viable. The ROS 2 architecture above MAVROS would remain; `nav_mode_manager` would lose explicit source commands and rely on EKF2's own handling, and the watchdog would need a different implementation.
