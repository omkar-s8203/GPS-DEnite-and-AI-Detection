# Engineering Decision Records

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |

Each record has: Context, Options, Evaluation, Decision, Reason, Consequences, Status.

Status values: **Accepted** (in force), **Accepted — conditional** (in force until a named gate decides), **Proposed**, **Superseded by ADR-xxx**.

To change a decision, add a new ADR that supersedes the old one. Do not edit the decision of an accepted ADR.

| ADR | Title | Status |
|---|---|---|
| [ADR-001](ADR-001-ros2-distribution.md) | ROS 2 distribution and operating system | Accepted |
| [ADR-002](ADR-002-autopilot-firmware.md) | Autopilot firmware: ArduPilot | Accepted |
| [ADR-003](ADR-003-flight-controller.md) | Flight-controller hardware: Pixhawk 6C | Accepted |
| [ADR-004](ADR-004-vio-solution.md) | VIO solution: OpenVINS, with backup and simplified options | Accepted — conditional on gate G2 |
| [ADR-005](ADR-005-slam-solution.md) | SLAM: not in the flight loop | Accepted |
| [ADR-006](ADR-006-ai-framework.md) | AI model and runtime: YOLO26n on NCNN | Accepted |
| [ADR-007](ADR-007-camera-interface.md) | Camera interface: dual CSI-2, libcamera, single-process stereo driver | Accepted |
| [ADR-008](ADR-008-mavlink-architecture.md) | MAVLink architecture: MAVROS over UART | Accepted |
| [ADR-009](ADR-009-gps-denied-transition.md) | GNSS ↔ vision transition: EKF3 source sets with companion-side alignment | Accepted |
| [ADR-010](ADR-010-ai-depth-fusion.md) | AI + stereo depth: parallel pipelines, late fusion | Accepted |
| [ADR-011](ADR-011-stereo-camera-suitability.md) | Stereo camera suitability and upgrade path | Accepted — conditional on gate G2 |
| [ADR-012](ADR-012-obstacle-avoidance.md) | Obstacle handling: stop-and-hold plus FC avoidance; no Nav2 | Accepted |
| [ADR-013](ADR-013-simulation.md) | Simulation: ArduPilot SITL + Gazebo Harmonic | Accepted |
| [ADR-014](ADR-014-vio-imu-source.md) | IMU source for VIO: camera-board ICM-20948 | Accepted — conditional on gate G2b |
| [ADR-015](ADR-015-visual-geolocalization.md) | **Primary GPS-denied localisation: satellite image matching + ground visual odometry** (DB-2.0) | Accepted |
| [ADR-016](ADR-016-reference-imagery-and-downward-camera.md) | Reference imagery policy and downward USB camera (DB-2.0) | Accepted — source and model to be confirmed |

| [ADR-017](ADR-017-ground-app.md) | **Ground app: native Android on the MK15 over the IP link** (DB-3.0) | Accepted |
| [ADR-018](ADR-018-search-track-follow.md) | **Search, track and follow using the downward camera** (DB-3.0) | Accepted |

## DB-3.0 effect on earlier records

| ADR | Effect |
|---|---|
| ADR-006 | Extended: a second, aerial-view YOLO26n model for the downward camera; one model active at a time |
| ADR-008 | Unchanged. The app does not use MAVLink; QGroundControl keeps the datalink |
| ADR-015 | Extended: minimum matching height is computed from the reference resolution; a search profile at 25–30 m is added |
| ADR-016 | Tightened: downward camera ≥ 1920×1080; reference of ≈ 0.25 m/px or better for the search profile; imagery is also displayed on the ground unit |
| Video topology (siyi-mk15.md §6) | Reversed: topology B (Pi Ethernet to the air unit) is the baseline; HDMI converter removed |

## DB-2.0 effect on earlier records

| ADR | Effect |
|---|---|
| ADR-004 | Still valid for the **low regime**. OpenVINS is no longer the primary GPS-denied source; its priority is Should. Its gate is now called G2b |
| ADR-005 | Unchanged and reinforced: drift is bounded by a prior satellite map, not by SLAM |
| ADR-009 | Still valid. The alignment filter now also accepts map fixes when GNSS is denied |
| ADR-011 | Still valid, but the stereo camera's weakness for VIO no longer threatens the core result. An upgrade is justified only if the low regime needs it |
| ADR-012 | Still valid for the low regime. In the cruise regime no obstacle sensing is active; the site must be clear |
| ADR-014 | Applies to low-regime VIO only |
