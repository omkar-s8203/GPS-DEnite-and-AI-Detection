# ADR-015 — Primary GPS-Denied Localisation: Satellite Image Matching

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** (design baseline DB-2.0) |
| Amends | [ADR-004](ADR-004-vio-solution.md) (role of VIO), [ADR-009](ADR-009-gps-denied-transition.md) (alignment), [ADR-011](ADR-011-stereo-camera-suitability.md) (criticality of the stereo camera), [ADR-005](ADR-005-slam-solution.md) (unchanged, reinforced) |

## Context

The project owner changed the concept: when GPS is lost, the drone captures an image of the ground, matches it against a satellite image stored on board, and obtains its current location from the match. AI vision (object detection with stereo range) stays as already designed.

DB-1.0 used stereo visual-inertial odometry as the only GPS-denied position source. Odometry is relative: its error grows without bound, and it needs alignment to GPS before GPS is lost.

## Options

| # | Option |
|---|---|
| A | Keep DB-1.0: stereo VIO only |
| B | Satellite map matching only, fixes sent to the FC as a MAVLink GPS (`GPS_INPUT`) |
| C | Satellite map matching for absolute fixes **plus** a relative visual odometry between fixes, fused on the companion and sent as external navigation |
| D | Full SLAM with a prior map |

Matching method options: classical features (SIFT, ORB, AKAZE) with RANSAC; learned features (XFeat, SuperPoint + LightGlue, LoFTR); template correlation; retrieval-then-match.

## Evaluation

| Criterion | A: VIO only | B: matching as GPS | C: matching + odometry | D: SLAM + prior map |
|---|---|---|---|---|
| Drift | Unbounded | Bounded | **Bounded** | Bounded |
| Smoothness of data to EKF3 | Smooth | ≈ 1 Hz steps, no velocity | **Smooth (20–30 Hz)** | Variable |
| Behaviour when a match fails | n/a | Position input stops | **Odometry carries on** | Complex |
| Fits existing frame design | Yes | Bypasses it | **Yes: fixes update `map → odom`** | No |
| Pi 5 compute | Fits | Fits | Fits with height-based node activation | Does not fit |
| Implementation effort | Done in design | Low | Medium | Very high |
| Matches the owner's concept | No | Yes | **Yes** | Partly |

Matching method:

| Method | Pi 5 CPU | Robust to appearance change | Training needed | Verdict |
|---|---|---|---|---|
| SIFT + RANSAC, map features precomputed | Tens of ms `[ESTIMATE]` | Moderate | None | **Primary** |
| ORB / AKAZE | Faster | Lower | None | Fallback if SIFT is too slow |
| XFeat (learned, lightweight) | Published as real-time on a laptop CPU; on a Pi to be measured | Higher | Pretrained weights | **Compare; adopt if clearly better** |
| SuperPoint + LightGlue, LoFTR | Too heavy without a GPU | Highest | Pretrained | Rejected for onboard use |
| Edge-image cross-correlation | Light | Low | None | **Simplified version** |

## Decision

**Option C.**

1. A downward camera supplies images to two nodes: `map_matcher` (absolute fix, ≈ 1 Hz) and `ground_vo` (relative motion, 15 Hz).
2. `localization_manager` fuses them in the `map → odom` offset and sends a continuous pose to ArduPilot as external navigation (EKF3 source set 2), as in DB-1.0.
3. Matching: SIFT + RANSAC similarity as primary, with roll, pitch, heading and height taken from the flight controller to orthorectify the image first. XFeat is evaluated on the same recordings. Edge cross-correlation is the simplified implementation.
4. GPS-denied cruise is flown at 40–60 m above ground; the low regime (1–10 m) keeps stereo VIO, stereo depth and obstacle stop.
5. Stereo VIO (OpenVINS) is demoted from "primary GPS-denied source" to "low-altitude odometry". Its priority drops from Must to Should.

Design: [visual-geolocalization.md](../09-navigation/visual-geolocalization.md).

## Reason

- An absolute fix removes the fundamental limit of DB-1.0 (unbounded drift) and matches what the owner asked for.
- Keeping an odometry between fixes makes the system tolerant of the matcher's inevitable gaps and gives the flight controller smooth input.
- Using known attitude, heading and height reduces matching to a two-dimensional search, which is what makes a classical matcher feasible on a Raspberry Pi.
- Precomputing map features off-board moves most of the cost out of the flight.
- The architecture above the localisation layer (state machine, MAVLink interface, safety) is unchanged.

## Consequences

- **New hardware:** a downward camera ([ADR-016](ADR-016-reference-imagery-and-downward-camera.md)).
- **New data dependency:** a licensed, georeferenced reference image of the flight area.
- **Higher flight:** 40–60 m instead of 1–8 m. Larger geofence, stricter site requirements, more demanding for the safety pilot, and the optical-flow fallback is out of range at that height. The safety documents are amended.
- **New main risk:** matching fails over featureless or changed terrain, or a wrong match is accepted. A new gate G2 tests feasibility on recorded data before any vehicle work depends on it.
- **Reduced risk:** the stereo camera's weakness for VIO no longer threatens the core result.
- Two more nodes and one tool to write; one more calibration (downward camera intrinsics and mounting).
- The academic contribution shifts towards "low-cost onboard satellite-image geo-localisation with a quantified accuracy and availability", which is a stronger and clearer result than odometry alone.
- `GPS_INPUT` remains a documented fallback way to deliver fixes to ArduPilot.
