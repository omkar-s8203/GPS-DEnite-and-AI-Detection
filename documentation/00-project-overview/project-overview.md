# Project Overview

| Field | Value |
|---|---|
| Document ID | GDN-OVR-001 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | Baseline |

## 1. Problem

Small multirotors depend on GNSS for position hold and waypoint flight. GNSS is unavailable or unreliable indoors, under canopy, near tall structures, and under interference. When it fails, a typical hobby or college drone falls back to altitude-hold and drifts with the wind until a pilot intervenes.

## 2. Objective

Build a research prototype that:

1. Flies normally on GNSS when GNSS is healthy.
2. Detects that GNSS is degrading before the autopilot loses its position estimate.
3. Switches to onboard visual localisation: it matches images of the ground against a satellite image stored on board to obtain its position, and keeps holding position and flying waypoint routes inside the mapped area.
4. Uses a neural network plus stereo depth to report what objects are in view and how far away they are.
5. Always leaves the safety pilot and the flight controller in charge of flight safety.

Success is defined by the measurable targets in [system-requirements.md](../01-requirements/system-requirements.md) and demonstrated through the staged tests in [testing-strategy.md](../13-testing/testing-strategy.md).

## 3. What this project is and is not

| It is | It is not |
|---|---|
| A low-cost, CPU-only, open-source reference design | A product, or a system for unsupervised operation |
| A study of how far inexpensive stereo hardware can be pushed for VIO, with measured limits | A claim of matching commercial autonomous drones |
| A ROS 2 architecture that a later team can extend | A full SLAM / exploration / path-planning system |
| Slow, low-altitude, line-of-sight flight with a safety pilot | High-speed, long-range or night flight |

## 4. System in one paragraph (DB-2.0)

When GPS is lost, the drone photographs the ground with a downward camera, finds where that photograph fits on a satellite image stored on board, and reads its position from the match. A Raspberry Pi 5 running Ubuntu 24.04 and ROS 2 Jazzy does this about once a second (`map_matcher`), tracks motion between matches from the same camera (`ground_vo`), and fuses the two into a continuous position. A navigation-mode manager watches GNSS health and tells the flight controller (Pixhawk 6C running ArduPilot Copter) to switch its EKF from GNSS to that position when GNSS is denied. This is flown at 40–60 m. Near the ground (1–10 m) a forward stereo camera provides depth for obstacle stopping, range for a small object detector (YOLO26n on NCNN), and stereo visual-inertial odometry. The Pi sends pose estimates, obstacle distances and slow setpoints over MAVLink; the flight controller does all stabilisation, estimation fusion and failsafe handling. A SIYI MK15 provides RC control, telemetry and video.

**DB-3.0 additions.** The operator works from a native Android app on the MK15, connected to the Raspberry Pi over the MK15's IP link. From the app they draw an area and start a **grid search**: the drone flies it line by line at 25–30 m, an aerial-view detector looks at the ground through the downward camera, and each find is pinned on the map with coordinates and a picture. They can also tap an object to **track** it and have the drone **follow it from above**. All of this works with GPS denied because position comes from map matching. The motivating use is search and rescue; the demonstration is on a mapped, undamaged site with dummy targets ([ADR-017](../17-decisions/ADR-017-ground-app.md), [ADR-018](../17-decisions/ADR-018-search-track-follow.md)).

DB-1.0 used stereo visual-inertial odometry alone as the GPS-denied source. The change to satellite matching is recorded in [ADR-015](../17-decisions/ADR-015-visual-geolocalization.md).

## 5. Key design decisions (summary)

| Topic | Decision | Record |
|---|---|---|
| OS / middleware | Ubuntu Server 24.04 LTS (arm64) + ROS 2 Jazzy Jalisco | [ADR-001](../17-decisions/ADR-001-ros2-distribution.md) |
| Autopilot firmware | ArduPilot Copter 4.7.x | [ADR-002](../17-decisions/ADR-002-autopilot-firmware.md) |
| Flight controller | Holybro Pixhawk 6C (+ PM02, M10 GPS) | [ADR-003](../17-decisions/ADR-003-flight-controller.md) |
| VIO | OpenVINS (primary), RTAB-Map stereo odometry (backup), OpenCV stereo VO (simplified) | [ADR-004](../17-decisions/ADR-004-vio-solution.md) |
| SLAM | Not in the flight loop; RTAB-Map optional for mapping on recorded data | [ADR-005](../17-decisions/ADR-005-slam-solution.md) |
| AI | YOLO26n exported to NCNN, 320 px input, CPU | [ADR-006](../17-decisions/ADR-006-ai-framework.md) |
| Camera interface | Dual CSI-2 through the Raspberry Pi libcamera fork, single-process stereo driver | [ADR-007](../17-decisions/ADR-007-camera-interface.md) |
| FC link | MAVROS over 921 600 baud UART, MAVLink 2 | [ADR-008](../17-decisions/ADR-008-mavlink-architecture.md) |
| GNSS ↔ vision transition | ArduPilot EKF3 source sets, companion-commanded, pilot override | [ADR-009](../17-decisions/ADR-009-gps-denied-transition.md) |
| AI + depth | Parallel detection and depth, fused per detection | [ADR-010](../17-decisions/ADR-010-ai-depth-fusion.md) |
| Stereo camera suitability | Keep the Waveshare camera as baseline behind a measurable gate; named upgrade if the gate fails | [ADR-011](../17-decisions/ADR-011-stereo-camera-suitability.md) |
| Obstacle avoidance | FC-side avoidance fed by the companion, plus companion stop-and-hold; no Nav2 | [ADR-012](../17-decisions/ADR-012-obstacle-avoidance.md) |
| Simulation | ArduPilot SITL + Gazebo Harmonic + ros_gz | [ADR-013](../17-decisions/ADR-013-simulation.md) |
| VIO IMU source | Camera-board ICM-20948 over I²C | [ADR-014](../17-decisions/ADR-014-vio-imu-source.md) |
| **Primary GPS-denied localisation (DB-2.0)** | **Satellite image matching (SIFT + RANSAC; XFeat compared) + ground visual odometry, at 40–60 m** | [ADR-015](../17-decisions/ADR-015-visual-geolocalization.md) |
| **Ground app (DB-3.0)** | Native Android app on the MK15 over the IP link to the Pi; QGroundControl unchanged | [ADR-017](../17-decisions/ADR-017-ground-app.md) |
| **Search, track, follow (DB-3.0)** | Downward camera with an aerial-view model at 25–30 m; follow from above | [ADR-018](../17-decisions/ADR-018-search-track-follow.md) |
| Reference imagery and downward camera (DB-2.0) | Source-agnostic map pack with recorded licence; USB 2.0 wide-angle downward camera | [ADR-016](../17-decisions/ADR-016-reference-imagery-and-downward-camera.md) |

## 6. Stakeholders

| Role | Interest |
|---|---|
| Project team | Build, test, report |
| Faculty guide / evaluators | Technical soundness, measurable results, honest claims |
| Safety pilot | Clear override, predictable failure behaviour |
| Future student teams | Documentation good enough to continue the work |

## 7. Top risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R-A (DB-3.0) | Scope: app, search and follow add about 8–10 weeks to an already full plan | High | High | Fixed build order with follow last; delivery levels; app developed in parallel against a mock from week 4 |
| R-B (DB-3.0) | People are hard to detect from above with a small camera and a nano model | High | Medium | 25–30 m search height, 1080p camera, own aerial training data; recall measured and reported; results labelled "covered", never "clear" |
| R-C (DB-3.0) | The MK15's IP path does not behave as needed for the app | Low | High | Checked first (gate G0); MAVLink-plus-RTSP fallback defined in ADR-017 |
| R-D (DB-3.0) | Rescue framing invites claims the project cannot support | Medium | Medium | Limits stated in every document and on the slides; tests on an undamaged site with dummies |
| R0 (DB-2.0) | Satellite matching fails or gives wrong fixes over the test site (featureless or changed terrain, poor reference image) | Medium | High | Offline feasibility gate G2 on recorded GNSS-tagged images before anything depends on it; own orthomosaic as alternative reference; learned matcher; site choice |
| R0b (DB-2.0) | Flight at 40–60 m raises the consequences of any failure; no FC-native position fallback at that height | Medium | High | GPS denial is simulated and reversible; shadow mode first; large clear site; spotter; pilot practice at height ([safety-architecture](../12-safety/safety-architecture.md) §8a) |
| R0c (DB-2.0) | No reference imagery that may legally be stored offline at sufficient resolution | Medium | Medium | Source-agnostic map pack; own orthomosaic ([ADR-016](../17-decisions/ADR-016-reference-imagery-and-downward-camera.md)) |
| R1 | Stereo camera (rolling shutter, no hardware sync) gives VIO too poor to fly on. **Lower impact since DB-2.0: affects the low regime only** | High | Medium | Gate G2b with numeric criteria; loosely-coupled backup; named camera upgrade ([ADR-011](../17-decisions/ADR-011-stereo-camera-suitability.md)) |
| R2 | Pi 5 CPU cannot run VIO + depth + AI together | Medium | Medium | CPU budget, load shedding, AI at 5 Hz / 320 px, optional accelerator |
| R3 | Camera stack on Ubuntu needs a source-built libcamera | High | Low | Documented build; fallback OS option in ADR-001 |
| R4 | Vibration corrupts IMU and blurs images | Medium | High | Isolated camera/IMU mount, short exposure, vibration test on bench |
| R5 | Position jump at source switch upsets the vehicle | Medium | High | Continuous pre-alignment, switch only when consistent, SITL first, hover-only first flights |
| R6 | MK15 air unit lot does not support 4S | Low | Medium | Check label before wiring; boost converter or 6S if needed |
| R7 | Crash during testing | Medium | High | Testing pyramid, tethered/netted first flights, spare props, pilot always on sticks |
| R8 | Schedule: integration takes longer than a semester | High | Medium | Simulation-first; each phase has a demonstrable output |

## 8. Document map

Start at [documentation/README.md](../README.md).
