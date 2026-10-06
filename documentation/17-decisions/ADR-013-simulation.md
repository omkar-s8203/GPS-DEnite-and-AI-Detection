# ADR-013 — Simulation Architecture

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

Development is simulation-first. The simulator must run the real autopilot code (so that EKF source switching, failsafes and the Lua watchdog are exercised), provide stereo camera and IMU data to the same ROS 2 graph that runs on the vehicle, and work with ROS 2 Jazzy.

## Options

| # | Option |
|---|---|
| A | ArduPilot SITL + Gazebo Harmonic via `ardupilot_gazebo`, sensors bridged with `ros_gz`, MAVROS over UDP |
| B | ArduPilot SITL with ArduPilot's own ROS 2 packages (`ardupilot_gz`, native DDS) |
| C | PX4 SITL + Gazebo |
| D | AirSim-family photorealistic simulator |
| E | ArduPilot SITL only (no 3D world), with a synthetic VIO source |
| F | Recorded-data replay only |

## Evaluation

| Criterion | A | B | C | D | E | F |
|---|---|---|---|---|---|---|
| Real ArduPilot code incl. EKF3 and Lua | Yes | Yes | No | Yes | Yes | No |
| Camera + IMU simulation | Yes | Yes | Yes | Yes (best images) | No | Real data |
| Matches the flight software interface (MAVROS) | **Yes** | No (DDS path; docs target Humble) | n/a | Yes | **Yes** | Partial |
| Jazzy compatibility | Harmonic pairs with Jazzy; plugin supports Harmonic | Humble-targeted | Yes | Extra work | Yes | Yes |
| Hardware needed | GPU recommended | GPU recommended | — | Strong GPU | None | None |
| Speed / CI suitability | Moderate | Moderate | — | Low | **High** | High |
| Closed loop | Yes | Yes | — | Yes | Yes | No |
| Sensor realism (sync, rolling shutter, vibration) | Low | Low | — | Medium | None | **Real** |

## Decision

Three complementary configurations:

| Name | Composition | Purpose |
|---|---|---|
| **SIM-A** | Option E: ArduPilot SITL + MAVROS + `fake_vio` | Fast, headless, automatable tests of the state machine, alignment, navigator, failsafes, watchdog |
| **SIM-B** | Option A: + Gazebo Harmonic, `ardupilot_gazebo`, `ros_gz` | Full closed loop with simulated stereo and IMU: VIO, depth, obstacles, detector, missions |
| **SIM-C** | Option F: rosbag replay of real sensor data | VIO and perception quality; regression |

Design: [simulation-strategy.md](../11-simulation/simulation-strategy.md).

## Reason

1. Using MAVROS over UDP in simulation keeps the simulated interface identical to the vehicle's (only the URL changes), which option B would not.
2. No single simulator covers everything. Logic needs speed and determinism (SIM-A); integration needs sensors in the loop (SIM-B); perception quality needs real data (SIM-C).
3. Gazebo Harmonic is the LTS Gazebo paired with Jazzy and is supported by the ArduPilot plugin.
4. SIM-A needs no GPU, so every team member and CI can run it.

## Consequences

- A vehicle SDF with stereo camera and IMU, several worlds and bridge configuration must be authored (`gdn_sim`).
- Gazebo does not reproduce this camera's worst properties (rolling shutter, sync error). Simulation success does not predict VIO quality on hardware; gate G2 uses real data.
- The development workstation needs Ubuntu 24.04; dual boot is recommended over WSL 2 for SIM-B.
- SITL parameter names for fault injection must be confirmed against Copter 4.7.
- Photorealistic simulation is left as optional future work.
