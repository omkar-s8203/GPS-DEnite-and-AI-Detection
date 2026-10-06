# ADR-012 — Obstacle Handling and Navigation Stack

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

The vehicle must not fly into obstacles ahead of it during slow autonomous flight. Sensing is a forward stereo camera with ≈ 73° horizontal FOV and 0.5–6 m useful range. The Pi's CPU is nearly fully allocated. ROS 2's standard navigation stack (Nav2) is the obvious candidate to consider.

## Options

| # | Option |
|---|---|
| A | Nav2 (costmaps, planner, controller) adapted to a multirotor |
| B | Custom 3D local planner on the companion (for example a voxel map with a sampling planner) |
| C | Companion stop-and-hold only |
| D | FC-side avoidance only, fed with `OBSTACLE_DISTANCE` |
| E | C + D together: two independent layers |
| F | E plus ArduPilot BendyRuler path planning (`OA_TYPE`) |

## Evaluation

| Criterion | A | B | C | D | E | F |
|---|---|---|---|---|---|---|
| Fit to multirotor in 3D | Poor (2D ground-robot assumptions) | Good | Adequate | Adequate | Adequate | Good |
| CPU cost | High | High | Negligible | Negligible | Negligible | Low (on FC) |
| Works when the pilot is flying (LOITER) | No | No | No | **Yes** | **Yes** | Yes |
| Works when the companion navigator is faulty | No | No | No | **Yes** | **Yes** | Yes |
| Development effort | High | Very high | Low | Low | Low | Medium (tuning, SITL) |
| Goes around obstacles | Yes | Yes | No | No | No | Yes |
| Appropriate to the sensor (narrow forward FOV) | Over-built | Over-built | Yes | Yes | Yes | Marginal: planning around what it cannot see |

## Decision

**Option E.**

- Layer 1 (companion): the navigator slows and stops its own velocity setpoints from `/obstacle/sectors`; unknown depth is not treated as clear.
- Layer 2 (flight controller): ArduPilot proximity avoidance fed by `OBSTACLE_DISTANCE` (`PRX1_TYPE = 2`, stop behaviour), following ArduPilot's documented companion depth-camera arrangement.
- No Nav2. No path planning around obstacles in the baseline. A simple sidestep and BendyRuler are stretch items to be tried in simulation first.

Details: [autonomous-navigation.md](../09-navigation/autonomous-navigation.md) §5.

## Reason

1. With a forward-only, 6 m sensor, the honest capability is "stop before hitting what is ahead". Planning around obstacles would require knowledge of space the sensor has not observed.
2. Two independent layers mean a navigator defect cannot by itself drive the vehicle into a detected obstacle, and the pilot gets protection in LOITER too.
3. Nav2 would consume CPU the system does not have and would need significant adaptation for no demonstrable benefit at this scope.
4. The project's contribution lies in localisation and transition, not planning.

## Consequences

- The vehicle halts at obstacles and waits, then aborts on timeout. It does not find a way around. This is stated as a limitation.
- Flight is nose-first only so that the camera covers the direction of travel.
- Obstacles outside the FOV, thin obstacles and texture-less surfaces are not reliably sensed (FMEA F33). Test sites are chosen accordingly and the pilot supervises.
- Whether ArduPilot's simple avoidance also limits GUIDED-mode velocity commands in 4.7 must be confirmed in SITL; layer 1 covers GUIDED regardless.
