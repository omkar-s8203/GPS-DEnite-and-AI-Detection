# Novelty and Academic Contribution

| Field | Value |
|---|---|
| Document ID | GDN-OVR-004 |
| Version | 1.0 |
| Date | 2026-10-05 |

## 1. What must not be claimed

| Claim | Why it is false or indefensible |
|---|---|
| "World's first GPS-denied AI drone" | GPS-denied flight with onboard vision has been demonstrated in research for well over a decade and is sold commercially |
| "Novel VIO / SLAM algorithm" | The project uses existing open-source estimators |
| "AI-based navigation" | Navigation is geometric; the neural network only labels objects |
| "Outperforms commercial drones" | It will not, and no evidence could support it |
| "Fully autonomous" | A safety pilot supervises every flight; arming and mode selection are human actions |
| "Works in any environment" | It needs light, texture and slow motion |
| Any performance number not marked MEASURED or VALIDATED | See [performance-requirements.md](../14-performance/performance-requirements.md) §11 |

## 2. Candidate contribution areas — assessed

| Area | Is it new? | Is it defensible as a contribution? | Verdict |
|---|---|---|---|
| Low-cost architecture | Low-cost vision drones exist | Yes, **if quantified**: a costed BOM and measured performance per rupee on commodity parts | **Supporting contribution** |
| Stereo vision | Standard | Not by itself | Enabler, not a contribution |
| ROS 2 integration | ROS 2 + ArduPilot integrations exist, but complete, documented Jazzy-era reference designs with simulation and tests are uncommon in student work | Yes, as an engineering artefact | **Supporting contribution** |
| VIO | Existing algorithms | The *evaluation on unsynchronised rolling-shutter hardware* is the contribution, not the algorithm | **Primary contribution (C2)** |
| AI perception | Standard detector | Not by itself | Enabler |
| Edge AI co-execution | Known that nano models run on a Pi | Measured co-execution budget (VIO + depth + detector on four cores, with interference quantified) is useful and rarely reported | **Supporting contribution** |
| Modular architecture | Good practice | As part of the reference design | Supporting |
| GPS/VIO transition | The mechanism exists in ArduPilot; example scripts exist | A companion-side manager with continuous pre-alignment, multi-check confidence gating, a third tier, and systematic fault-injection evaluation goes beyond the stock examples | **Primary contribution (C1)** |
| Open-source implementation | Many exist | Value depends on documentation and reproducibility | Supporting |
| AI + depth object localisation | Common technique | Measured range accuracy vs distance on this hardware | Minor |

## 2a. DB-2.0: revised contribution

With satellite image matching as the primary GPS-denied method ([ADR-015](../17-decisions/ADR-015-visual-geolocalization.md)), the contribution list changes as follows. The technique itself is established in the literature and is **not** claimed as new.

| Rank | Contribution | What is defensible |
|---|---|---|
| **C-A (new, primary)** | **Onboard satellite-image geo-localisation on a CPU-only Raspberry Pi 5.** A complete pipeline (orthorectify with autopilot attitude and height → match against precomputed map features → gate → fuse with ground visual odometry → feed ArduPilot) with measured accuracy, availability and wrong-fix rate against GNSS | Published systems usually assume a GPU or offline processing. A quantified result on a ₹-thousands computer and camera, with the design choices that made it fit (precomputed map features, 2-D search from known attitude), is useful and honest whether the numbers are good or modest |
| **C-B** | **Hand-over manager** (was C1): GNSS health classification, one offset filter that takes GNSS while it is good and map fixes when it is not, confidence from independent cross-checks, tiered fallback, pilot override | Original software with testable behaviour; evaluated by fault injection in simulation and by simulated denial in flight |
| **C-C (new)** | **Comparison of matching methods on identical data**: hand-crafted (SIFT), lightweight learned (XFeat) and correlation, including the effect of height, reference resolution and reference age (satellite image vs own orthomosaic) | An experiment a student team can actually complete from recorded data |
| C-D | Characterisation of the low-cost stereo module for VIO (was C2) | Now secondary; still a valid smaller result for the low regime |
| C-E | Open, documented, tested ROS 2 reference architecture (was C3) | Unchanged |

Additional things that must not be claimed: that the system works under real jamming or spoofing (denial is simulated and GNSS is restored as the first recovery action); that it works over any terrain or in any season; centimetre accuracy; "AI navigation" (the learned matcher, if adopted, is one component of a geometric pipeline).

Research questions added:

| # | Question | Metric |
|---|---|---|
| RQ6 | What position accuracy and availability does satellite matching achieve on a Pi 5 over the test site? | RMS error vs GNSS; share of attempts accepted; wrong-fix rate |
| RQ7 | How do height, reference resolution and reference age change that? | Same metrics across conditions |
| RQ8 | Does a lightweight learned matcher justify its CPU cost over SIFT here? | Accepted fixes per CPU-second; error |
| RQ9 | How long can the fused estimate stay within 5 m without GNSS? | Error over time and distance |

Suggested report title for DB-2.0:

> **GNSS-Denied Navigation of a Small Multirotor by Onboard Satellite-Image Matching on a Raspberry Pi 5: Design and Evaluation**

The sections below are the DB-1.0 text; C1–C3 there correspond to C-B, C-D and C-E above.

## 2b. DB-3.0: effect of the app, search and follow on the contribution

| Addition | Is it a research contribution? | How to present it |
|---|---|---|
| Android ground app | No: it is engineering. It makes the system usable and demonstrable | As part of the reference implementation (C-E); describe the MK15 three-path architecture it relies on |
| Grid search with geolocated findings **without GNSS** | **Yes, a supporting contribution (C-F)**: object coordinates obtained from map-matched position plus ground projection, with measured position error and detection rate | Report finding-position error on GNSS vs GNSS-denied, and recall/precision against surveyed dummy targets |
| Tracking and follow from above | Minor; established techniques | As a demonstration of the localisation's usefulness |

Added research question:

| # | Question | Metric |
|---|---|---|
| RQ10 | How accurately can a low-cost drone report the coordinates of objects it finds when GNSS is denied? | Finding position error vs surveyed truth; detection recall and precision from 25–30 m |

Further claims to avoid: "search and rescue drone", "works in disaster areas", "finds survivors", "area cleared". Accurate wording: *"a grid-search mission that marks candidate objects with coordinates for a human to check, demonstrated on a mapped test site with dummy targets, with GNSS disabled."* The detection rate must be quoted whenever search is mentioned.

Ethics additions: participants in follow tests give informed consent; no imagery of uninvolved people is collected; the limits of detection are disclosed to anyone who might rely on a search result.

## 3. Contribution statement

### C1 — A confidence-gated GNSS ↔ vision navigation hand-over for ArduPilot

**What:** a ROS 2 navigation-mode manager and localisation manager that (a) classify GNSS health, (b) continuously align a drifting vision frame to the autopilot frame while GNSS is trusted, (c) score localisation confidence from independent cross-checks against the flight controller's own inertial and barometric sensing, and (d) command ArduPilot's EKF source sets through three tiers (GNSS → vision → optical flow), with pilot override and an FC-side watchdog.

**Evidence to produce:** transition latency; position step at hand-over; hold accuracy after hand-over; behaviour under injected faults (GNSS glitch, VIO dropout, VIO divergence, companion crash) in SITL across many runs, and in flight for the nominal cases; ablation showing what happens without pre-alignment and without the cross-checks.

**Why defensible:** it is original software with a clear function, testable hypotheses and measurable outcomes. The underlying EKF mechanism is acknowledged as ArduPilot's.

### C2 — Characterisation of a low-cost, unsynchronised, rolling-shutter stereo module for visual-inertial odometry

**What:** a quantitative study of the Waveshare IMX219-83 (about ₹5,000) as a VIO sensor on a Raspberry Pi 5: achievable software synchronisation; calibration quality; drift versus speed and yaw rate; sensitivity to vibration; comparison of a tightly coupled filter (OpenVINS), a loosely coupled stereo odometry (RTAB-Map) and a minimal OpenCV odometry on identical recorded data.

**Evidence to produce:** skew distributions; drift tables; the operating envelope within which the camera is usable; failure cases; a recommendation (use / use with limits / upgrade).

**Why defensible:** the vendor states the camera has no hardware synchronisation, and guidance for VIO generally says such cameras are unsuitable. A measured answer to "how unsuitable, and under what conditions is it usable?" is useful to every student team with the same budget. **A negative result is still a valid result.**

### C3 — An open, documented, tested reference architecture

**What:** the complete package: requirements, ROS 2 Jazzy architecture, frame conventions, safety architecture with FMEA, simulation environments, test pyramid, and code, for a Pi 5 + Pixhawk + ArduPilot GPS-denied platform.

**Evidence to produce:** the repository itself; reproducible simulation demonstration; CPU/thermal/power budgets measured on the Pi 5 with all subsystems running.

**Why defensible:** engineering contribution of reuse value; modest and true.

## 4. Research questions

| # | Question | Method | Metric |
|---|---|---|---|
| RQ1 | How small a position discontinuity can be achieved when handing over from GNSS to vision using continuous pre-alignment? | SITL Monte-Carlo with fault injection; flight tests | Step size distribution; hold RMS |
| RQ2 | Do cross-checks against FC inertial and barometric data detect VIO failure before position error becomes hazardous? | Induced failures on recorded and simulated data | Detection lead time; missed-detection rate |
| RQ3 | What VIO accuracy is achievable with an unsynchronised rolling-shutter stereo pair, and what limits it? | Bench and flight datasets; three estimators | Drift % vs speed, yaw rate, skew |
| RQ4 | Can VIO, stereo depth and a neural detector share four Cortex-A76 cores without degrading localisation? | Controlled load experiments on the Pi | VIO drift and latency with/without each load; CPU, temperature |
| RQ5 | How accurate is object range from late fusion of a nano detector and 60 mm-baseline stereo? | Targets at surveyed distances | Range error vs distance |

## 5. Comparison baselines for the evaluation

| Baseline | Purpose |
|---|---|
| GNSS position (open sky) | Reference trajectory for VIO drift |
| ArduPilot optical-flow hold (tier 3 alone) | "What a low-cost drone can already do without this project" |
| Hand-over without pre-alignment (ablation) | Shows the value of C1(b) |
| Confidence from covariance only (ablation) | Shows the value of C1(c) |
| EuRoC dataset on the same Pi and estimator | Separates sensor limitations from compute/algorithm limitations |
| Published accuracy of the estimators on standard datasets | Context; clearly labelled as from literature |

## 6. Scope of valid conclusions

Results apply to: this camera model, this computer, this estimator version, the tested environments, and speeds within the envelope. The report must not generalise beyond that.

## 7. Suggested report title

> **Design and Evaluation of a Low-Cost ROS 2 Architecture for GNSS-to-Vision Navigation Hand-over on a Small Multirotor**

Subtitle or abstract may mention stereo visual-inertial odometry on a Raspberry Pi 5 and onboard object detection. Avoid "AI-powered navigation" in the title; "with onboard AI perception" is accurate.

## 8. Ethics and responsible conduct

- All flights supervised, in permitted areas, compliant with applicable rules.
- No GNSS interference of any kind.
- Recorded imagery containing people is handled according to institute policy; faces are not published without consent.
- Limitations are reported with the same prominence as results.
- Third-party software and licences are credited ([technology-selection.md](../04-software/technology-selection.md) §4).
