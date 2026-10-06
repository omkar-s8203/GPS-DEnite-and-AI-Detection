# Weight Budget

| Field | Value |
|---|---|
| Document ID | GDN-HW-010 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | **ESTIMATE.** Weigh every item on a 1 g scale at procurement and replace these values. |

## 1. Avionics and payload

| Item | Qty | Mass each (g) | Total (g) | Basis |
|---|---|---|---|---|
| Raspberry Pi 5 (board) | 1 | 46 | 46 | `[VERIFY]` commonly quoted |
| Active cooler | 1 | 30 | 30 | `[ESTIMATE]` |
| microSD card | 1 | 1 | 1 | |
| Waveshare IMX219-83 board | 1 | 25 | 25 | `[ESTIMATE]`; not given by vendor |
| CSI cables | 2 | 3 | 6 | `[ESTIMATE]` |
| Camera + Pi mounting plate, dampers, standoffs | 1 | 60 | 60 | `[ESTIMATE]` (3D-printed / carbon plate) |
| 5 V BEC | 1 | 20 | 20 | `[ESTIMATE]` |
| 12 V regulator | 1 | 8 | 8 | `[ESTIMATE]` |
| MK15 air unit | 1 | 116 | 116 | `[VENDOR]` upper figure (range 74–116 g) |
| MK15 antennas + cables | 2 | 12 | 24 | `[ESTIMATE]` |
| HDMI input converter | 1 | 35 | 35 | `[ESTIMATE]` |
| HDMI cable (thin, short) | 1 | 12 | 12 | `[ESTIMATE]` |
| MTF-01 | 1 | 5 | 5 | `[ESTIMATE]` |
| Downward USB camera with lens, cable and mount (DB-2.0) | 1 | 30 | 30 | `[ESTIMATE]` |
| Wiring, connectors, fuses, ties | — | — | 50 | `[ESTIMATE]` |
| **Autonomy + link payload** | | | **≈ 468** | |

## 2. Flight-control set (present on any ArduPilot quad)

| Item | Mass (g) | Basis |
|---|---|---|
| Pixhawk 6C (plastic case) | 35 | `[VENDOR]` 34.6 g |
| PM02 + leads | 25 | `[ESTIMATE]` |
| M10 GNSS + mast | 50 | `[ESTIMATE]` |
| **Subtotal** | **≈ 110** | |

## 3. Total against requirement

| Group | Mass (g) |
|---|---|
| Autonomy + link payload | 468 |
| Flight-control set | 110 |
| **Total avionics** | **≈ 578** |
| Requirement NFR-040 | ≤ 550 |

**Over the limit by about 28 g since the downward camera was added (DB-2.0), with ±15 % uncertainty.** Removing the HDMI converter path (−55 g with its cable and regulator) brings it back inside. The all-up-weight estimate in §5 rises to ≈ 1,920 g, still under 2.0 kg but with only 80 g of margin. Candidates for reduction, in order: drop the HDMI converter and cable (−47 g, −3 W; lose pilot video overlay), lighter mounting plate, shorter antenna leads.

**DB-3.0 update:** the HDMI converter, its cable and the 12 V regulator are removed (−55 g) and an Ethernet cable to the air unit is added (+10 g). Total avionics ≈ **533 g**, inside the 550 g limit. All-up weight ≈ 1,875 g.

## 4. Added payload attributable to this project

Compared with a plain GNSS quad that already has an RC receiver and a telemetry radio (≈ 30 g together):

| | Mass (g) |
|---|---|
| Vision/compute payload (Pi, cooler, camera, mount, BEC, cables, MTF-01) | ≈ 243 |
| MK15 link in place of a small receiver and radio (≈ 187 − 30) | ≈ 157 |
| **Net added payload** | **≈ 400** |

## 5. Vehicle-level estimate (airframe assumed)

| Group | Mass (g) | Basis |
|---|---|---|
| Frame, 450–500 mm, with landing gear | 450 | `[ASSUMPTION]` |
| Motors ×4 (2212–2216 class) | 260 | `[ASSUMPTION]` |
| ESCs ×4 (or 4-in-1) | 100 | `[ASSUMPTION]` |
| Propellers ×4 (10-inch) | 50 | `[ASSUMPTION]` |
| Battery 4S 5200 mAh | 480 | `[ASSUMPTION]` |
| Avionics (§3) | 548 | |
| **All-up weight** | **≈ 1890** | |
| Requirement NFR-043 | < 2000 | 110 g margin |

Thrust check: for controllable flight the vehicle should have a thrust-to-weight ratio ≥ 2. At 1.9 kg that needs ≥ 950 g of thrust per motor at full throttle on 4S. Many 2212-920 KV motor / 10-inch prop sets deliver roughly 800–1000 g on 4S `[VERIFY from the motor's thrust table]`, which is borderline. **The airframe decision must include a thrust table check**; 2216-class motors or 11-inch props may be needed.

## 6. Centre of gravity

The camera, Pi and mount (≈ 190 g) sit at the front. Balance with the battery position and by placing the MK15 air unit at the rear. Verify by suspending the vehicle at the geometric centre: it must hang level within a few degrees.

## 7. Actions

| # | Action |
|---|---|
| WB-1 | Weigh all owned items now (Pi, camera, MK15 air unit, converter, antennas) and update §1 |
| WB-2 | Select the airframe with a thrust table that gives thrust-to-weight ≥ 2 at 2.0 kg |
| WB-3 | Re-run this budget after procurement; record the measured all-up weight before first flight |
