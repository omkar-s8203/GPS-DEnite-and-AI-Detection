# ADR-003 — Flight-Controller Hardware: Holybro Pixhawk 6C

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

The flight controller was not specified. It must run ArduPilot Copter 4.7 with external navigation, optical flow, proximity and Lua scripting; provide four serial links (GCS radio, companion, GNSS, flow sensor); and be reliable enough that the state-estimation experiments are not confounded by poor inertial data.

## Options

| # | Option |
|---|---|
| A | Holybro Pixhawk 6C |
| B | Holybro Pixhawk 6C Mini |
| C | Holybro Pixhawk 6X |
| D | CubePilot Cube Orange+ |
| E | Matek H743 (Slim/Wing) |
| F | Pixhawk 2.4.8 (clone) |
| G | F405-class FPV controller |

## Evaluation

Full table in [flight-controller.md](../03-hardware/flight-controller.md) §3. Summary:

| Option | Meets functional needs | Vibration isolation / IMU heating | Connectors | Cost | Verdict |
|---|---|---|---|---|---|
| A | Yes | Yes / yes | JST-GH | Medium | **Selected** |
| B | Yes, if port count is sufficient | Yes | JST-GH | Medium | Acceptable alternate |
| C | Yes | Yes; 3 IMUs; Ethernet | JST-GH | High | Over-specified |
| D | Yes | Yes; 3 IMUs | Carrier | Highest | Over-specified |
| E | Yes | No | Solder pads | Low | Budget fallback |
| F | Marginal / No | Poor | DF13 | Lowest | Rejected |
| G | No (1 MB flash: features removed) | No | Solder pads | Lowest | Rejected |

## Decision

**Holybro Pixhawk 6C (plastic case) with PM02 power module and M10 GNSS/compass**, bought as a combo from an authorised reseller.

## Reason

1. STM32H743 with 2 MB flash runs the full ArduPilot feature set, including Lua and visual odometry.
2. Two current-generation IMUs (ICM-42688-P, BMI088) with built-in isolation and heating give trustworthy inertial data.
3. Three TELEM ports plus two GPS ports cover every link without adapters.
4. Pixhawk-standard JST-GH connectors remove soldering errors from a student build.
5. It is a first-class target for both ArduPilot and PX4, preserving the fallback in ADR-002.
6. It costs substantially less than Cube/6X-class hardware; the saving is reserved for a possible camera upgrade.

## Consequences

- About ₹25,000–37,000 for the combo (range from listings; confirm at purchase).
- Two IMUs only: no majority voting. Accepted for a supervised prototype.
- The FC cannot power the Pi or the MK15; separate regulation is required ([power-architecture.md](../03-hardware/power-architecture.md)).
- If only the 6C Mini is obtainable, verify the port count; move the flow sensor to GPS2 or DroneCAN if necessary.
- Clone boards are explicitly out of bounds, even as temporary substitutes: results obtained on them would not be comparable.
