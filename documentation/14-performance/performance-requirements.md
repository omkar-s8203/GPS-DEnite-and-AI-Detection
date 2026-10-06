# Performance Requirements and Tracking

| Field | Value |
|---|---|
| Document ID | GDN-PRF-001 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | Baseline — **no project measurement exists yet.** Every MEASURED and VALIDATED cell is empty by design. |

## 1. Status vocabulary

| Status | Meaning | Who may set it |
|---|---|---|
| **TARGET** | The value the design must achieve. From requirements. | Design |
| **ESTIMATE** | Engineering prediction from analysis, vendor data or literature. Not evidence. | Design |
| **MEASURED** | Observed on project hardware or in project simulation, with a run ID. A single measurement. | Test |
| **VALIDATED** | Measured repeatedly (≥ 3 runs) under the stated conditions, meets the target, reviewed. | Test + review |

Rules:

- A number without one of these labels must not appear in any report or presentation.
- Vendor benchmarks are ESTIMATE for this project, even though the vendor measured them.
- Simulation results are MEASURED (sim), never VALIDATED for real-world claims.
- If a MEASURED value misses its TARGET, either the design changes or the target is changed by a recorded decision. Targets are not edited silently.

## 2. Sensing

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-01 | Stereo frame rate | 20 Hz ± 1 | 20 Hz | — | — | T2-01 |
| P-02 | Dropped stereo pairs | < 1 % | < 1 % | — | — | T2-01 |
| P-03 | L/R timestamp skew (99th percentile) | ≤ 1.0 ms | Unknown: 0.1–5 ms plausible | — | — | T2-02 |
| P-04 | Exposure time outdoors | ≤ 4 ms | 1–4 ms | — | — | T2-04 |
| P-05 | IMU sample rate | ≥ 200 Hz | 225 Hz | — | — | T2-05 |
| P-06 | IMU max sample gap | ≤ 20 ms | 5–15 ms | — | — | T2-05 |
| P-07 | Camera–IMU time-offset stability | ± 2 ms | ± 1–3 ms | — | — | T2-06 |
| P-08 | Stereo reprojection error | ≤ 0.5 px RMS | 0.2–0.5 px | — | — | T2-07 |

## 3. Depth and obstacles

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-10 | Depth rate | ≥ 10 Hz | 10 Hz (BM) | — | — | T3-04 |
| P-11 | Depth error at 2 m (static) | ≤ 5 % | 2–4 % | — | — | T2-09 |
| P-12 | Depth error at 5 m (static) | ≤ 10 % | 5–10 % | — | — | T2-09 |
| P-13 | Minimum range | ≤ 0.5 m | 0.40 m | — | — | T2-09 |
| P-14 | Obstacle latency (capture → MAVLink) | ≤ 200 ms | ≈ 100 ms | — | — | T3-06 |
| P-15 | Detection of 0.3 m post at 4 m | Detected | Likely | — | — | T2-11 |

## 4. VIO and localisation

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-20 | VIO output rate | ≥ 20 Hz | 20 Hz | — | — | T3-04 |
| P-21 | Latency capture → external nav sent | ≤ 80 ms (p95) | 50–80 ms | — | — | T3-06 |
| P-22 | VIO drift, handheld loop | ≤ 2 % of path | 1–3 % | — | — | T2-15 |
| P-23 | VIO drift, flight (vs GNSS reference) | ≤ 2 % of path | 1.5–4 % | — | — | L8-B |
| P-24 | Stationary drift, motors running | ≤ 0.3 m in 3 min | 0.1–0.5 m | — | — | T7-15 |
| P-25 | Yaw-rate limit before tracking degrades | ≥ 45 °/s | 30–60 °/s | — | — | T2-16 |
| P-26 | Alignment residual | ≤ 0.3 m, ≤ 3° | 0.2 m, 2° | — | — | S-02, T7-09 |
| P-27 | Position hold on vision, 60 s | ≤ 0.5 m RMS | 0.2–0.6 m | — | — | L8-C/D |
| P-28 | Confidence drops below MEDIUM before 1 m error (induced faults) | 100 % of cases | Unknown | — | — | S-08, T2-16 |

## 4a. Satellite map matching and ground odometry (DB-2.0)

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-80 | Horizontal error of accepted fixes vs GNSS | ≤ 5 m RMS | 2–5 m after site registration | — | — | T3-G2, L8-B2 |
| P-81 | Share of match attempts accepted (suitable terrain, 50 m) | ≥ 70 % | 40–90 %, strongly terrain-dependent | — | — | T3-G2 |
| P-82 | Accepted wrong fixes (> 15 m error) | < 1 % | Unknown | — | — | T3-G2, L8-B2 |
| P-83 | Match latency, SIFT path, on the Pi with the full stack | ≤ 500 ms | 100–300 ms | — | — | T3-G3 |
| P-84 | Match attempt rate | ≥ 0.5 Hz | 1 Hz | — | — | T3-G3 |
| P-85 | Time to first accepted fix after denial | ≤ 10 s | 2–6 s | — | — | S-21, L8-D2 |
| P-86 | Ground VO drift | ≤ 3 % of distance | 2–5 % | — | — | T3-G5 |
| P-87 | Fused position error with GNSS withheld (replay) | ≤ 5 m RMS, bounded | 3–6 m | — | — | T3-G6 |
| P-88 | Position hold on map matching, 60 s at 50 m | ≤ 5 m RMS | 3–6 m | — | — | L8-D2 |
| P-89 | Circuit closure, ≈ 300 m with GNSS disabled | ≤ 5 m | 3–8 m | — | — | L8-G2 |
| P-90 | Minimum height for ≥ 70 % acceptance with the site reference | Reported | 35–50 m at 0.3–0.5 m/px | — | — | T3-G4 |
| P-91 | Mean offset of the map pack vs GNSS after registration | ≤ 1 m | ≤ 1 m | — | — | L7, L8-B2 |
| P-92 | CPU, cruise configuration (geo-localisation + detector, stereo nodes off) | ≤ 75 % of 4 cores | 45–75 % | — | — | T3-G3 |
| P-93 | Map pack size, 1 km² | ≤ 500 MB | 50–300 MB | — | — | T2-G2 |

The estimates for P-80 to P-82 are the least certain numbers in the project: published results for this technique range from a few metres to tens of metres depending on terrain and season. They are placeholders until gate G2.

## 4b. Ground app, search, track and follow (DB-3.0)

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-100 | Video latency, camera → app | ≤ 500 ms | 200–500 ms | — | — | T7-A1 |
| P-101 | Command latency, tap → acknowledgement | ≤ 300 ms | 50–250 ms | — | — | T7-A1 |
| P-102 | Video rate to the app at 640×480 | ≥ 8 fps | 10 fps | — | — | T7-A1 |
| P-103 | CPU for JPEG video streaming | ≤ 10 % of one core | 5–10 % | — | — | T3 |
| P-104 | App memory on the MK15 | ≤ 400 MB | 150–350 MB | — | — | TA-1 |
| P-105 | Recall, person-sized dummy from 25–30 m, daylight, open ground | ≥ 60 % | 30–75 %, very uncertain | — | — | T2-A2, L8-L |
| P-106 | Precision, same conditions | ≥ 70 % | 50–85 % | — | — | T2-A2 |
| P-107 | Recall, vehicles from 25–30 m | ≥ 85 % | 80–95 % | — | — | T2-A2 |
| P-108 | Aerial detection rate, full frame tiled, with localisation running | ≥ 0.5 Hz | 0.7–1.5 Hz | — | — | T2-A3 |
| P-109 | Finding position error, GNSS on | ≤ 5 m | 2–5 m | — | — | L8-L |
| P-110 | Finding position error, GNSS denied | ≤ 8 m | 4–9 m | — | — | L8-M |
| P-111 | Search coverage of the requested area | ≥ 95 % | 90–98 % | — | — | S-25, L8-L |
| P-112 | Search area per battery | ≥ 2 ha | 3–5 ha | — | — | L8-L |
| P-113 | Follow: target inside the central half of the image at ≤ 2 m/s | ≥ 90 % of the time | Unknown | — | — | S-26, L8-O |
| P-114 | Tracker re-acquisition | ≤ 3 s | 1–3 s | — | — | T1-A4, S-26 |
| P-115 | Map-fix acceptance at 25–30 m with the site reference | ≥ 70 % | Depends on reference resolution | — | — | T3-G4 |
| P-116 | CPU in the search profile | ≤ 80 % of 4 cores | 60–85 % | — | — | T2-A3 |

P-105 is the number a reviewer will ask about, and the least predictable. It must be reported as measured, with the test conditions, and never rounded up.

## 5. GNSS monitoring and transition

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-30 | Time to declare DENIED | ≤ 3 s | 2–2.5 s (dwell-limited) | — | — | S-03 |
| P-31 | Decision → source set active | ≤ 1 s | 0.1–0.3 s | — | — | S-03 |
| P-32 | Position step at GNSS → vision | ≤ 1.0 m | 0.2–0.8 m | — | — | S-03, L8-D |
| P-33 | Position step at vision → GNSS | Reported (no target; depends on drift) | 0.5–3 m | — | — | S-05, L8 |
| P-34 | False DENIED declarations in open sky | 0 per 10 min | 0 | — | — | L8-B |
| P-35 | VIO loss → flow tier active | ≤ 1 s | 0.6–0.8 s | — | — | S-06 |
| P-36 | Companion loss → FC mode change | ≤ 3 s | 2–2.5 s | — | — | S-09, T7-12 |

## 6. AI perception

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-40 | Inference time, YOLO26n 320 px NCNN, 2 threads, full stack | ≤ 80 ms | 35–60 ms (scaled from vendor 67 ms at 640 px, 4 idle cores) | — | — | T2-12 |
| P-41 | Detection rate | ≥ 5 Hz | 5 Hz | — | — | T2-12 |
| P-42 | Latency capture → detection | ≤ 150 ms | 90–130 ms | — | — | T3-06 |
| P-43 | mAP50, project test set | ≥ 0.6 (person, 3–10 m) | Unknown | — | — | T2-13 |
| P-44 | Person detection range | ≥ 10 m | 10–12 m | — | — | T2-13 |
| P-45 | Object range error, 2–6 m | ≤ 12 % | 5–12 % | — | — | T2-14 |
| P-46 | Object map-position error at ≤ 5 m | ≤ 0.5 m | 0.3–0.6 m | — | — | S-16, T2-14 |
| P-47 | VIO drift increase with detector on | ≤ 10 % | Small | — | — | T7 |

## 7. Compute, thermal, power

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-50 | CPU, full stack, average | ≤ 75 % of 4 cores | 58–88 % | — | — | T3-12, T7-16 |
| P-51 | RAM resident | ≤ 4 GB | 2–3 GB | — | — | T3-12 |
| P-52 | SoC temperature, steady state, 35 °C ambient | ≤ 75 °C | 60–75 °C with active cooler | — | — | T2-17, T7-16 |
| P-53 | Throttling events | 0 | 0 | — | — | T7-16 |
| P-54 | Boot to `CC READY` | ≤ 90 s | 40–60 s | — | — | T3-01 |
| P-55 | Pi power, full stack | ≤ 9 W average | 7–9 W | — | — | T2-18 |
| P-56 | Avionics power from battery | ≤ 20 W average | ≈ 20 W | — | — | PB-4 |
| P-57 | 5 V rail at Pi under load step | ≥ 4.9 V | — | — | — | T7-02 |

## 8. Communication

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-60 | Companion–FC link errors | 0 CRC errors in 10 min | 0 | — | — | ML-1 |
| P-61 | External-nav rate at FC | 20–30 Hz | 30 Hz | — | — | ML-3 |
| P-62 | Setpoint rate jitter | ± 10 ms | ± 5–15 ms (Python) | — | — | T3-04 |
| P-63 | Time-sync jitter | < 5 ms | 1–3 ms | — | — | ML-11 |
| P-64 | GCS telemetry load on TELEM1 | < 50 % of baud capacity | ≈ 22 % | — | — | ML-10 |
| P-65 | Video latency | Informational | 150–400 ms | — | — | TL-3 |

## 9. Vehicle

| ID | Metric | TARGET | ESTIMATE | MEASURED | VALIDATED | Test |
|---|---|---|---|---|---|---|
| P-70 | All-up weight | < 2.0 kg | ≈ 1.89 kg | — | — | T7-19 |
| P-71 | Avionics mass | ≤ 550 g | ≈ 548 g | — | — | WB-1 |
| P-72 | Hover endurance (to 20 % reserve) | ≥ 8 min | 11–13 min | — | — | L8-A |
| P-73 | Vibration (`VIBE`) | < 30 m/s² | 10–25 | — | — | L8-A |
| P-74 | Thrust-to-weight | ≥ 2.0 | Depends on airframe | — | — | Airframe selection |
| P-75 | Max autonomous speed, GPS-denied | 1.5 m/s (limit, not goal) | — | — | — | L8-G |

## 10. Updating this document

1. After each test, enter the value with its run ID in the MEASURED column: for example `0.42 ms (20261114-03)`.
2. After three consistent runs meeting the target, copy to VALIDATED with the run IDs.
3. If a target is missed, open an entry in [project-status.md](../project-status.md) under "Decisions Pending".
4. Never delete a measured value; add a new one with a note.

## 11. What may be claimed

| Evidence available | Permissible statement |
|---|---|
| TARGET only | "The design targets …" |
| ESTIMATE | "We expect approximately … based on …" |
| MEASURED (sim) | "In simulation, we measured …" |
| MEASURED (hardware, one run) | "In one test, we observed …" |
| VALIDATED | "The system achieves … under the following conditions: …" |
