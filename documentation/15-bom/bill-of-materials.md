# Bill of Materials

| Field | Value |
|---|---|
| Document ID | GDN-BOM-001 |
| Version | 1.0 |
| Date | 2026-10-05 |

## 1. Price policy

Prices are **approximate Indian retail ranges** for budgeting only. They are not quotations.

| Tag | Meaning |
|---|---|
| **Seen** | A listing with this price was found during research in October 2026 (source named). Prices change; GST and shipping may or may not be included as noted. |
| **Est.** | No listing verified in this pass. Range based on typical market levels. Confirm before purchase. |

Component prices in India, especially Raspberry Pi boards, were volatile in 2026: listings for the same board differed by a factor of three between sellers. Always check several authorised resellers.

Status values: **Owned**, **To buy**, **Decision pending**, **Optional**.

## 2. Core BOM

Required for the baseline design.

| # | Component | Qty | Purpose | Key specification | Approx. price (₹) | Source / basis | Status |
|---|---|---|---|---|---|---|---|
| C1 | Raspberry Pi 5, 8 GB | 1 | Companion computer | BCM2712, 8 GB | 7,000–23,000 | Seen: listings from ≈ ₹7,000 to ≈ ₹22,800 across Indian sellers (wide spread; verify) | Owned |
| C2 | Raspberry Pi Active Cooler | 1 | Mandatory cooling | Official heatsink + PWM fan | 450–800 | Seen: ≈ ₹450–670 | To buy |
| C3 | microSD card, 64–128 GB, A2/U3 | 1 | OS and logs | High-endurance preferred | 800–1,800 | Est. | To buy |
| C4 | Waveshare IMX219-83 stereo camera | 1 | Stereo vision + VIO IMU | 2 × IMX219, 60 mm baseline, ICM-20948 | 4,800–6,600 | Seen: ≈ ₹4,799 (Robu, incl. GST) to ≈ ₹6,600 | Owned |
| C5 | CSI cables, 15-pin to 22-pin (Pi 5), ≤ 200 mm | 2 | Camera connection | Check what is supplied with C4 | 150–400 each | Est. | To buy if not supplied |
| C6 | SIYI MK15 HDMI combo | 1 | RC, telemetry, video | Ground unit, air unit, HDMI converter | ≈ 53,400 + 18 % GST (≈ 63,000) | Seen: ElectroPi listing ₹53,430 excl. GST | Owned |
| C7 | Holybro Pixhawk 6C + PM02 + M10 GPS (combo) | 1 | Flight controller, power module, GNSS/compass | STM32H743; ICM-42688-P + BMI088 | 24,500–37,000 | Seen: 6C Mini combos ≈ ₹24,500–37,000; standard 6C combo price to be confirmed | **To buy** |
| C8 | MicoAir MTF-01 | 1 | Optical flow + ToF range | 8 m range, 100 Hz, MAVLink serial | 2,500–5,000 | Est. | To buy |
| C9 | BEC 5.1–5.25 V, ≥ 5 A continuous, 3S–6S input | 1 | Pi supply | Low ripple; adjustable or fixed 5.2 V | 500–1,500 | Est. | To buy |
| C10 | 12 V regulator (buck-boost or low-dropout), ≥ 0.5 A | 1 | HDMI converter supply | See power architecture §3 | 200–600 | Est. | To buy (after verifying converter supply) |
| C11 | Micro-HDMI cable, thin/flexible, short | 1 | Pi → HDMI converter | Connector at converter end to be verified | 300–900 | Est. | To buy |
| C12 | Airframe kit, 450–500 mm quad, with landing gear | 1 | Vehicle | Thrust-to-weight ≥ 2 at 2.0 kg | 3,000–15,000 (frame only) | Est. | **Decision pending** |
| C13 | Motors ×4 + ESCs ×4 (or 4-in-1) + propellers (10–11 in, with spares) | 1 set | Propulsion | 4S; thrust ≥ 950 g per motor | 8,000–20,000 | Est. | Decision pending |
| C14 | LiPo 4S 5000–5200 mAh | 2 | Flight battery (two for test days) | ≥ 30C, XT60 | 4,000–7,500 each | Est. | To buy |
| C15 | LiPo balance charger + LiPo-safe bag | 1 | Charging | 4S capable | 3,000–6,000 | Est. | To buy / institute |
| C16 | Power distribution board / harness, XT60/XT30 connectors, inline fuses | 1 set | Power wiring | — | 500–1,500 | Est. | To buy |
| C17 | Wiring: JST-GH pigtails, Dupont, silicone wire, heat-shrink, ferrites | 1 set | Interconnect | — | 500–1,500 | Est. | To buy |
| C18 | Camera/Pi mounting plate, vibration dampers, standoffs, GNSS mast | 1 set | Mechanical | 3D-printed or cut plate | 500–2,000 | Est. | To make |
| C19 | Calibration target (AprilGrid print on rigid board) | 1 | Calibration | Flat within 1 mm | 300–1,000 | Est. | To make |
| C20 | LiPo voltage alarm | 1 | Independent battery warning | — | 100–300 | Est. | To buy |
| C21 | Spare propellers | 4 sets | Testing | — | 600–1,500 | Est. | To buy |

### Core additions in DB-2.0 (satellite map matching)

| # | Component | Qty | Purpose | Key specification | Approx. price (₹) | Source / basis | Status |
|---|---|---|---|---|---|---|---|
| C22 | Downward camera | 1 | Satellite map matching and ground visual odometry | USB 2.0 UVC, ≈ 1 MP, global shutter preferred, M12 lens 90–120° ([downward-camera.md](../03-hardware/downward-camera.md)) | 2,500–8,000 | Est.; model not yet selected | **To buy** (OD-12) |
| C23 | Short shielded USB 2.0 cable and mount | 1 | Camera connection | ≤ 200 mm | 200–600 | Est. | To buy / make |
| C24 | Reference imagery of the test site | 1 | Onboard map pack | Georeferenced, ≤ 0.5 m/px, licence permitting offline academic use | 0 – several thousand, depending on source | Est.; source not yet identified | **To obtain** (OD-11) |
| C25 | Own orthomosaic of the test site (alternative / comparison reference) | 1 | High-resolution reference | One GNSS mapping flight + open-source photogrammetry (for example OpenDroneMap) | 0 (time only) | — | To make |

These add roughly ₹2,700–8,600 to the core spend, excluding any paid imagery.

### DB-3.0 changes (Android app, search, follow)

| # | Component | Change | Approx. price (₹) | Status |
|---|---|---|---|---|
| C22 | Downward camera | Specification raised: USB 2.0 UVC, **≥ 2 MP, 1920×1080 MJPEG**, wide lens | 3,000–9,000 (Est.) | To buy (OD-12) |
| C26 | Ethernet cable, MK15 air unit 8-pin connector to RJ45 | New: Pi ↔ air unit IP link for the app | 200–800 (Est.), or made from a pigtail | To make / buy |
| C10, C11 | 12 V regulator, micro-HDMI cable | **No longer needed** (HDMI converter not used) | −500 to −1,500 | Removed |
| C24 | Reference imagery | Resolution requirement tightened to ≈ 0.25 m/px or better for the search profile, or use the own orthomosaic (C25) | — | To obtain (OD-11) |
| C27 | Search targets: person-sized dummies, bright markers, cones for the test area | New | 500–3,000 (Est.) | To make / buy |
| C28 | Head protection and high-visibility vest for the team member taking part in follow tests | New | 500–1,500 (Est.) | To buy / borrow |
| — | Android development | Android Studio on the development laptop; the MK15 itself is the test device; no purchase | 0 | — |

Net effect on cost is small (roughly ₹0–5,000 more). The main new cost is time.

### Core cost summary

| Group | Approx. range (₹) |
|---|---|
| Already owned (C1, C4, C6) | — |
| Flight controller set (C7) | 24,500–37,000 |
| Sensing and compute accessories (C2, C3, C5, C8) | 4,000–8,400 |
| Power and wiring (C9, C10, C11, C16, C17, C20) | 2,100–6,300 |
| Airframe and propulsion (C12, C13, C21) | 11,600–36,500 |
| Batteries and charging (C14 ×2, C15) | 11,000–21,000 |
| Mechanical and calibration (C18, C19) | 800–3,000 |
| DB-2.0 additions (C22, C23; imagery not counted) | 2,700–8,600 |
| **Additional spend for the core build** | **≈ 57,000–121,000** |

The spread is dominated by the airframe/propulsion choice and by where the Pixhawk is bought. If the institute already has a suitable 450–500 mm quad, batteries and a charger, the additional spend falls to roughly ₹31,000–55,000.

## 3. Optional BOM

Improves the project; not required for the baseline.

| # | Component | Qty | Purpose | Approx. price (₹) | Basis | When to buy |
|---|---|---|---|---|---|---|
| O1 | Luxonis OAK-D Lite (or equivalent global-shutter, hardware-synchronised stereo camera with IMU) | 1 | Replaces C4 if gate G2 fails; also offloads depth and neural inference | 18,000–35,000 | Est. (US list price in the range of roughly US$150–270 depending on variant and date; Indian price to be confirmed) | Only after gate G2 |
| O2 | NVMe SSD (128–256 GB) + M.2 HAT for Pi 5 | 1 | Reliable raw-image logging | 3,500–7,000 | Est. | If SD logging drops data |
| O3 | Propeller guards / indoor cage netting | 1 | Safer early flights | 1,000–5,000 | Est. | Before stage C flights |
| O4 | Tether line and ground anchor | 1 | First GPS-denied flights | 300–800 | Est. | Before stage C |
| O5 | USB-C / inline DC power meter | 1 | Power measurements | 500–1,500 | Est. | Bench phase |
| O6 | RTC battery for Pi 5 | 1 | Keeps wall-clock time without network | 300–600 | Est. | Convenience |
| O7 | 5.6 V TVS / over-voltage protection module | 1 | Protect the Pi from BEC failure | 100–400 | Est. | With C9 |
| O8 | Third battery | 1 | Longer test days | 4,000–7,500 | Est. | As needed |
| O9 | Copper foil / shielding tape, shielded CSI cables | — | EMI mitigation | 300–1,200 | Est. | If T7-06 fails |
| O10 | Small Ethernet switch + GH-to-RJ45 cable | 1 | Use HDMI converter and Pi Ethernet together | 800–2,000 | Est. | Only if in-flight SSH is needed |
| O11 | Matek H743 flight controller | 1 | Low-cost alternative to C7 | 7,000–10,000 (FC only) | Est. | Only if budget forces it |

## 4. Future-upgrade BOM

Beyond this project's scope; listed to show the growth path.

| # | Component | Purpose | Approx. price (₹) | Basis |
|---|---|---|---|---|
| U1 | AI accelerator HAT for Pi 5 (Hailo-8L class) | Higher-rate / larger detection and segmentation models | 7,000–12,000 | Est. |
| U2 | NVIDIA Jetson Orin Nano class computer | GPU for learned perception and heavier VIO/SLAM | 30,000–55,000 | Est. |
| U3 | RTK GNSS pair (base + rover) | Centimetre ground truth for evaluating VIO | 30,000–80,000 | Est. |
| U4 | 2D or solid-state 3D LiDAR | Robust obstacle sensing and localisation in poor light | 10,000–100,000+ | Est. |
| U5 | Second camera (downward or rear) | Wider perception coverage | 2,000–20,000 | Est. |
| U6 | Higher-grade IMU for VIO | Lower noise, SPI, hardware trigger | 3,000–15,000 | Est. |
| U7 | Pixhawk 6X or Cube-class flight controller | Triple IMU redundancy, Ethernet | 35,000–60,000 | Est. |
| U8 | UWB kit | Indoor ground truth / absolute reference | 15,000–40,000 | Est. |
| U9 | Compute Module 5 + lightweight carrier | Mass reduction | 8,000–15,000 | Est. |

## 5. Tools and consumables (usually available in a college lab)

| Item | Use |
|---|---|
| Soldering station, multimeter, bench power supply | Assembly and testing |
| Oscilloscope | 5 V rail ripple and droop |
| 1 g digital scale | Weight budget |
| Caliper, ruler, inclinometer app | Calibration target and mounting measurements |
| Hex drivers, thread-lock, cable ties, double-sided foam tape | Assembly |
| Prop balancer | Vibration |
| Laptop with Ubuntu 24.04 | Development, simulation, GCS |
| Fire-safe charging area, sand bucket | LiPo safety |

## 6. Procurement order

| Order | Items | Reason |
|---|---|---|
| 1 (now) | C2, C3, C5, C19, O5, **C22, C23, C24** | Enables camera and calibration work, and the offline map-matching feasibility test (gate G2), with owned hardware |
| 2 | C7, C8, C9, C17 | Enables FC bring-up, HIL and link testing |
| 3 (after airframe decision) | C12, C13, C14, C15, C16, C18, C20, C21 | Vehicle build |
| 4 (after verifying the MK15 units) | C10, C11 | Depends on converter connector and supply |
| Conditional | O1 after gate G2; O2 if logging drops; O3/O4 before stage C | — |

## 7. Notes

- Buy the flight controller from an authorised Holybro reseller. Counterfeit and "2.4.8" clone boards are common and are explicitly rejected ([flight-controller.md](../03-hardware/flight-controller.md)).
- Record the invoice value and date for each purchase in this table (add a column) so that the final project report states actual cost.
- Confirm the MK15 air unit's 4S support (siyi-mk15 MK-1) **before** choosing a 4S airframe.
