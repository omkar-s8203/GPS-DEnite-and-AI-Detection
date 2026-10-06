# ADR-009 — GNSS ↔ Vision Transition

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted.** Extended by [ADR-015](ADR-015-visual-geolocalization.md): the `map → odom` filter that is driven by GNSS while GNSS is good is driven by satellite-map fixes when GNSS is denied in the cruise regime, instead of being frozen |

## Context

The vehicle must move from GNSS-based to vision-based position hold in flight, and back. Three questions must be answered: where the fusion happens, who decides to switch, and how the vision frame is related to the autopilot's frame so that the switch does not cause a jump.

## Options

**Fusion location**

| # | Option |
|---|---|
| F1 | ArduPilot EKF3 with switchable source sets (GNSS / ExternalNav / OpticalFlow) |
| F2 | Companion-side filter fusing GNSS + VIO, sending one blended pose to the FC as external nav at all times |
| F3 | Tightly coupled GNSS-visual-inertial estimator on the companion |

**Decision authority**

| # | Option |
|---|---|
| D1 | Pilot only (RC switch) |
| D2 | FC Lua script (in the style of ArduPilot's `ahrs-source.lua` examples) |
| D3 | Companion state machine via `MAV_CMD_SET_EKF_SOURCE_SET`, pilot can override |

**Frame alignment**

| # | Option |
|---|---|
| A1 | Let ArduPilot align (T265-style `VISO_TYPE = 2` behaviour) |
| A2 | Companion aligns continuously while GNSS is good and sends pose already in the FC frame (`VISO_TYPE = 1`) |
| A3 | No alignment: reset EKF origin at the switch |

## Evaluation

| Option | Pros | Cons |
|---|---|---|
| F1 | FC remains the authority; documented mechanism; logs each source; survives companion loss with a third source | Loose coupling; a step is possible at each switch |
| F2 | Smooth blending | Companion becomes flight-critical in all modes, even with perfect GNSS; second filter to tune |
| F3 | Best theoretical accuracy | Highest complexity; same criticality problem |
| D1 | Simplest, safest | Not autonomous; slow |
| D2 | Decision inside the FC | Lua has no access to companion-side health (features, sync, cross-checks) except through a few numbers; logic split across two places |
| D3 | Uses all available health information; testable in ROS 2; pilot retains a hardware override | Depends on the companion being alive (covered by the watchdog) |
| A1 | Less companion code | Tied to a specific device type's assumptions; less visibility; alignment happens at the switch, not before |
| A2 | Alignment quality is known *before* it is needed; switch can be refused if not aligned; external nav can be validated in shadow mode | More companion logic |
| A3 | Trivial | Loses the relation to the GNSS frame; breaks missions and home |

## Decision

- **F1:** EKF3 source sets — set 1 GNSS, set 2 ExternalNav (vision), set 3 optical flow.
- **D3 with staged introduction:** D1 (pilot switch) in the first flight tests; D3 (companion state machine) once validated. The RC source switch always overrides. A small FC Lua watchdog (not the decision-maker) handles companion loss.
- **A2:** the companion estimates a 4-DoF `map → odom` transform while GNSS is good, freezes it (with a 5 s look-back) when GNSS degrades, and always streams aligned pose to the FC, including while GNSS is in use.

State machine: [gps-denied-state-machine.md](../02-system-architecture/gps-denied-state-machine.md). Alignment: [coordinate-frames.md](../02-system-architecture/coordinate-frames.md) §8.

## Reason

1. Keeping fusion in EKF3 preserves the principle that the vehicle flies without the companion.
2. The companion has far more information about vision health than the FC does. It should make the recommendation; the FC executes it; the pilot can overrule it.
3. Streaming aligned external nav continuously means (a) the FC logs it during GNSS flight, giving a free in-flight evaluation of VIO against GNSS, and (b) the EKF already has consistent data at the instant of switching, minimising the step.
4. Freezing the alignment with look-back prevents degraded GNSS from corrupting the frame relation just before it is needed.

## Consequences

- The companion needs a correct, well-tested alignment estimator and must never apply the camera lever arm twice (`VISO_POS_* = 0`).
- Yaw alignment before take-off relies on the compass; position-based refinement needs a short translation in GNSS mode. The test procedure includes it.
- Returning to GNSS produces a step equal to accumulated vision drift. It is done only at low speed, reported, and by default left to the pilot.
- If the companion dies in vision mode, the FC watchdog selects the flow source or the EKF failsafe acts.
- The transition logic, alignment and confidence scoring are the project's main original software; see [academic-contribution.md](../00-project-overview/academic-contribution.md).
