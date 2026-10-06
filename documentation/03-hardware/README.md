# Hardware Engineering — Index

| Field | Value |
|---|---|
| Document ID | GDN-HW-000 |
| Version | 1.0 |
| Date | 2026-10-05 |

## Documents

| Document | Content |
|---|---|
| [high-level-architecture.md](high-level-architecture.md) | Block diagrams: data, RC, telemetry, power, GNSS, logging paths |
| [low-level-design.md](low-level-design.md) | Interfaces, pins, voltages, buses, connectors, wiring, EMI, vibration, thermal |
| [raspberry-pi-5.md](raspberry-pi-5.md) | Companion computer analysis |
| [stereo-camera.md](stereo-camera.md) | Waveshare IMX219-83 analysis, limits, alternatives |
| [flight-controller.md](flight-controller.md) | Flight controller comparison and selection |
| [siyi-mk15.md](siyi-mk15.md) | RC / datalink / video analysis |
| [downward-camera.md](downward-camera.md) | Downward camera for satellite map matching (DB-2.0) |
| [sensors.md](sensors.md) | Which additional sensors are required and which are not |
| [power-architecture.md](power-architecture.md) | Power distribution design |
| [power-budget.md](power-budget.md) | Electrical load estimate |
| [weight-budget.md](weight-budget.md) | Mass estimate |

## Hardware baseline (HB-1.0)

| Role | Component | Status |
|---|---|---|
| Companion computer | Raspberry Pi 5, 8 GB, with active cooler | Owned (cooler to buy) |
| Stereo camera + VIO IMU | Waveshare IMX219-83 (dual IMX219, ICM-20948) | Owned — **conditional**, see gate G2 |
| Downward camera (DB-2.0) | USB 2.0 UVC, ≈ 1 MP, 90–120° lens, global shutter preferred | **To procure** — model open (OD-12) |
| Reference imagery (DB-2.0) | Georeferenced image of the test site, ≤ 0.5 m/px, licence permitting offline use | **To obtain** (OD-11) |
| RC / telemetry / video | SIYI MK15 HDMI combo | Owned |
| Flight controller | Holybro Pixhawk 6C | **To procure** |
| Power module | Holybro PM02 (analog, 5.2 V 3 A) | To procure (combo) |
| GNSS + compass | Holybro M10 | To procure (combo) |
| Optical flow + range | MicoAir MTF-01 | To procure |
| Pi regulator | 5.1–5.25 V, ≥ 5 A continuous BEC | To procure |
| Airframe + propulsion | 450–500 mm quad, 4S | **Not specified — open decision** |

## Reading the specification tables

| Tag | Meaning |
|---|---|
| `[VENDOR]` | Taken from manufacturer or reseller documentation; source listed in [references](../references.md) |
| `[ESTIMATE]` | Engineering estimate; must be replaced by a measurement |
| `[ASSUMPTION]` | Assumed for design; must be confirmed |
| `[MEASURE]` | To be measured on the actual hardware |
| `[VERIFY]` | Believed correct from general knowledge but not confirmed against a primary source in this pass |
