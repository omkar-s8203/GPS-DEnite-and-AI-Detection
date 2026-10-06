# ADR-005 — SLAM: Not in the Flight Loop

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

"GPS-denied drone" projects often assume SLAM is required. SLAM adds a persistent map and loop closure to odometry, which removes accumulated drift when places are revisited. It costs CPU and complexity, and its corrections are discontinuous.

## Options

| # | Option |
|---|---|
| A | Odometry only (VIO) in flight; no SLAM |
| B | Full visual SLAM in flight (ORB-SLAM3 or RTAB-Map) feeding the FC |
| C | VIO in flight feeding the FC; SLAM running in parallel at low rate for mapping only (not fed back) |
| D | VIO in flight; SLAM off-board on recorded data |

## Evaluation

| Criterion | A | B | C | D |
|---|---|---|---|---|
| CPU on the Pi 5 | Fits | Does not fit alongside depth + AI | Marginal | Fits |
| Smoothness of data given to EKF3 | Smooth | Jumps at loop closure / relocalisation | Smooth | Smooth |
| Drift | Unbounded but small over short flights | Bounded on revisits | Unbounded for control | Unbounded for control |
| Benefit at mission scale (minutes, tens of metres) | — | Small | Map for display | Map and drift estimate for the report |
| Implementation risk | Low | High | Medium–high | Low |

## Decision

**Option A for flight, with option D as an analysis tool.** No SLAM system runs in the control path. RTAB-Map may be run off-board on recorded bags to build a 3D map of the test site and to quantify drift through loop closure.

## Reason

1. The flight controller's EKF wants a smooth, continuous position measurement. Loop-closure corrections are steps; ArduPilot's own documentation warns of position jumps when sources disagree.
2. At 1–2 % drift, a 60 m GPS-denied flight accumulates about a metre of error: acceptable for the demonstration and honestly reportable.
3. The CPU budget is already close to its limit.
4. Reliable VIO plus a clean GNSS transition is a complete project. Adding SLAM would spread effort across a third hard problem.

## Consequences

- DB-2.0 note: in the cruise regime drift is now bounded by matching against a prior satellite map ([ADR-015](ADR-015-visual-geolocalization.md)). That is localisation in a given map, not SLAM; this decision stands. The limits below apply to the low regime.
- The system cannot correct drift by recognising places. GPS-denied flight duration and distance are limited by policy (≤ 3 min, ≤ 60 m per test).
- "SLAM" must not appear in claims about the flying system. The accurate term is visual-inertial odometry.
- Off-line mapping remains available as a visual result for the report.
- Future work: option C, with pose-graph corrections applied to `map → odom` slowly rather than as steps. The frame design already provides the place for that correction.
