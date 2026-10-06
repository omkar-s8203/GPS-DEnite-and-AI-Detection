# Power Budget

| Field | Value |
|---|---|
| Document ID | GDN-HW-009 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | **ESTIMATE.** No value here has been measured on project hardware. Replace each row with a measurement during bench testing. |

## 1. Avionics load

| Item | Rail | Average (W) | Peak (W) | Basis |
|---|---|---|---|---|
| Raspberry Pi 5, full stack (≈ 60–75 % CPU) | 5 V | 8.0 | 12.0 | Third-party measurements: ≈ 3 W idle, ≈ 7–9 W four-core load, ≈ 11–12 W sustained maximum |
| Active cooler fan | 5 V | 0.5 | 1.0 | `[ESTIMATE]` |
| Stereo camera (2 × IMX219) + ICM-20948 | 3.3 V from Pi | 0.8 | 1.0 | `[ESTIMATE]`; IMU contribution is negligible (mA-level per vendor) |
| Pixhawk 6C | 5.2 V | 1.5 | 2.5 | `[ESTIMATE]` |
| M10 GNSS + compass | 5 V from FC | 0.3 | 0.4 | `[ESTIMATE]` |
| MTF-01 optical flow + ToF | 5 V from FC | 0.5 | 0.5 | `[VENDOR]` 500 mW |
| MK15 air unit | VBAT | 3.2 | 12.0 | `[VENDOR]` |
| HDMI input converter | 12 V | 3.0 | 3.0 | `[VENDOR]` |
| Downward USB camera (DB-2.0) | 5 V from Pi USB | 1.0 | 1.5 | `[ESTIMATE]` |
| **Subtotal at the loads** | | **18.8** | **33.9** | |
| Regulator losses (≈ 88 % average efficiency on regulated rails) | | 2.1 | 3.2 | `[ESTIMATE]` |
| **Total from battery** | | **≈ 20.9** | **≈ 37.1** | |

Peaks do not all coincide; the simultaneous peak is lower than the sum. The sum is used for regulator and wiring sizing.

Compared with NFR-041 (≤ 20 W average, ≤ 35 W peak): **exceeded by about 1 W average and 2 W peak since the downward camera was added (DB-2.0).** Removing the HDMI converter (3 W) restores the margin; otherwise the requirement is relaxed by a recorded decision. Battery-current figures in §2 rise by about 5 %.

**DB-3.0 update:** the HDMI converter is removed ([ADR-017](../17-decisions/ADR-017-ground-app.md)). Loads become ≈ 15.8 W average and ≈ 30.9 W peak; from the battery ≈ **17.6 W average, ≈ 34 W peak**. NFR-041 is met again. The Pi encodes JPEG video for the app (a few percent of one core) and the aerial detector raises CPU use in the search profile; the Pi figure of 8 W average is kept but must be re-measured in that profile.

Flight time at cruise height: climbing to 50 m and back costs energy that the DB-1.0 low-altitude profile did not. Expect roughly one minute less usable time per flight `[ESTIMATE]`.

## 2. Battery current for avionics

| Battery voltage | Average current | Peak current |
|---|---|---|
| 16.8 V (full) | 1.18 A | 2.1 A |
| 14.8 V (nominal) | 1.34 A | 2.4 A |
| 13.2 V (critical) | 1.50 A | 2.7 A |

## 3. Safety margin

| Rail | Expected peak | Regulator rating | Margin |
|---|---|---|---|
| V5_PI | ≈ 2.8 A (14 W / 5.1 V) | 5 A | 44 % |
| V5_FC | ≈ 0.7 A | 3 A | 77 % |
| V12 | ≈ 0.25 A | 0.5 A | 50 % |

Design rule: ≥ 30 % current margin on every regulator at peak. All rails comply on paper.

## 4. Flight-time estimate

Propulsion power is dominated by the airframe, which is not yet selected. The figures below are an order-of-magnitude check, not a prediction.

| Quantity | Value | Basis |
|---|---|---|
| All-up weight | 1.9 kg | [weight-budget.md](weight-budget.md) |
| Hover power | ≈ 250–320 W | `[ESTIMATE]`: 130–170 W/kg for 10-inch props on a 450–500 mm frame |
| Avionics | ≈ 20 W | §1 |
| Total | ≈ 270–340 W | |
| Battery | 4S 5200 mAh = 77 Wh | |
| Usable (80 % depth of discharge) | 61.6 Wh | |
| **Hover endurance** | **≈ 11–13.5 min** | 61.6 Wh ÷ total power |

Avionics take roughly 6–7 % of the energy; they cost about one minute of flight. The requirement (≥ 8 min usable, §3 of the requirements) is plausible with margin.

## 5. Measurement plan

| # | Measurement | Method | Phase |
|---|---|---|---|
| PB-1 | Pi power: idle, each node added in turn, full stack | Inline USB-C/DC power meter on the 5 V feed | Bench (L7) |
| PB-2 | 5 V rail droop and ripple under load step | Oscilloscope at the Pi pins | Bench |
| PB-3 | MK15 + converter power | Bench supply current reading | Bench |
| PB-4 | Whole-avionics current with motors off | FC current sensor after calibration | Bench |
| PB-5 | Hover current and endurance | FC log (`BAT.Curr`, `BAT.CurrTot`) | First hover flights (L8) |

Results go into [performance-requirements.md](../14-performance/performance-requirements.md) with status MEASURED.
