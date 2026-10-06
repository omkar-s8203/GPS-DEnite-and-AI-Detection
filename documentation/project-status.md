# Project Status

| Field | Value |
|---|---|
| Last updated | 2026-10-05 |
| Baseline | Design baseline **DB-3.0** (DB-2.0: satellite image matching. DB-3.0: native Android app on the MK15, grid search, track and follow) |
| Current phase | P01 — Research and design baseline (revised, pending review) |
| Next gate | G1 — design baseline accepted |
| Coding | **Not started. Must not start until G1 is passed.** |

## Completed

| Item | Evidence |
|---|---|
| Repository inspection | [repository-inspection.md](00-project-overview/repository-inspection.md): directory was empty |
| Technology and hardware research | [18-research](18-research/README.md), [references.md](references.md) |
| System requirements (FR/NFR) | [system-requirements.md](01-requirements/system-requirements.md) |
| System, hardware and software architecture | Sections 02, 03, 04 |
| ROS 2 architecture: packages, nodes, interfaces, QoS, TF, lifecycle, launch | Section 05 |
| Coordinate-frame design | [coordinate-frames.md](02-system-architecture/coordinate-frames.md) |
| GPS-denied state machine | [gps-denied-state-machine.md](02-system-architecture/gps-denied-state-machine.md) |
| Stereo, calibration, VIO, AI, navigation, state-estimation designs | Sections 06–09 |
| MAVLink / FC integration design | Section 10 |
| Simulation strategy | Section 11 |
| Safety architecture and FMEA | Section 12 |
| Testing strategy | Section 13 |
| Performance targets (no measurements yet) | Section 14 |
| BOM, power and weight budgets (estimates) | Sections 15, 03 |
| Roadmap | Section 16 |
| 16 decision records | Section 17 |
| DB-2.0 revision: satellite map matching design, downward camera, amended requirements, state machine, frames, ROS 2 interfaces, safety, FMEA, tests, performance, BOM, roadmap | [visual-geolocalization.md](09-navigation/visual-geolocalization.md), [ADR-015](17-decisions/ADR-015-visual-geolocalization.md), [ADR-016](17-decisions/ADR-016-reference-imagery-and-downward-camera.md) and the "DB-2.0" sections in the amended documents |
| Flight-controller selection | ADR-002, ADR-003 |

## In Progress

| Item | Owner | Note |
|---|---|---|
| Review of the design baseline by team and guide (gate G1) | Team | Read [README.md](README.md) first |
| Closing open decisions OD-1 … OD-4 | Team | Needed for G1 |

## Not Started

| Item | Phase |
|---|---|
| Git repository, package skeletons, CI | P02 |
| SITL environment (SIM-A) | P03 |
| Raspberry Pi provisioning; camera and IMU drivers | P04 |
| Navigation-mode manager, localisation manager, safety supervisor, Lua watchdog | P05 |
| Calibration; depth; obstacle sectors; simplified VO | P06 |
| OpenVINS integration; VIO evaluation; gate G2 | P07 |
| FC bring-up; MAVLink HIL | P08 |
| Detector, object localiser, dataset | P09 |
| Gazebo simulation (SIM-B); navigator; mission manager | P10 |
| Airframe build and manual flight | P11 |
| Vehicle integration and bench tests | P12 |
| All flight testing | P13–P15 |
| Optimisation, evaluation, report | P16–P17 |

## Blocked

| Item | Blocked by |
|---|---|
| FC bring-up, HIL, any vehicle work | Flight controller not yet procured (HP-1) |
| Airframe build, weight/power validation, flight time | Airframe not selected (OD-1) |
| MK15 wiring | Verification of air-unit voltage range and converter connector (HP-5) |
| SIM-B work | No Ubuntu 24.04 workstation with a GPU confirmed (SP-1) |
| Stereo VIO go/no-go (low regime) | Requires P04–P07 measurements (gate G2b) |
| **Map-matching go/no-go (gate G2)** | Needs the downward camera (HP-11), reference imagery (HP-12) and a GNSS-tagged image set (HP-13) |
| Cruise-regime flights | Test site and permission for 40–60 m (OD-14) |

## Decisions Pending

| ID | Decision | Options / recommendation | Needed by | Decided by |
|---|---|---|---|---|
| OD-1 | Airframe and propulsion | 450–500 mm quad, 4S, thrust-to-weight ≥ 2 at 2.0 kg (for example an S500/X500-class kit); or an existing institute airframe | G1 | Team + guide |
| OD-2 | AI class set and demonstration scenario | Default: person, vehicle, custom marker | G1 (affects data collection) | Team + guide |
| OD-3 | Indoor flight required? | Default: outdoor only, with simulated denial. Indoor adds vision-yaw configuration and a netted space. | G1 | Team + guide |
| OD-4 | One-semester or two-semester scope | Bronze / Silver / Gold levels in the roadmap | G1 | Guide |
| OD-5 | Keep the Waveshare camera or upgrade | Decided by measurement at gate G2 ([ADR-011](17-decisions/ADR-011-stereo-camera-suitability.md)) | Week ≈ 13 | Team, on evidence |
| OD-6 | Automatic return to GNSS in the final demonstration, or pilot-initiated | Default: pilot-initiated | G5 | Team |
| OD-7 | Licence for project code | Recommendation: GPL-3.0 (compatible with OpenVINS and ArduPilot; AGPL considerations for the detector package) | P02 | Team + institute |
| OD-8 | Video topology: HDMI converter (A) or direct Ethernet (B) | Default: A | P08 | Team |
| OD-9 | Stereo matcher: BM or SGBM | Decide from benchmark | P06 | Team, on evidence |
| OD-10 | Storage for raw-image logging: microSD or NVMe | Decide from write-rate test | P04 | Team, on evidence |
| **OD-11** | **Reference imagery source for the test site** | Must be ≤ 0.5 m/px and licensed for offline academic use. Google Maps imagery may not be stored offline. Options: a permissive provider or geoportal, purchased scene, OpenAerialMap if covered, own orthomosaic | Before P06G | Team + guide |
| **OD-12** | **Downward camera model and lens** | USB 2.0 UVC, ≈ 1 MP, global shutter preferred, 100° or 120° lens | P02 | Team |
| OD-13 | Run the detector on the downward camera at cruise height? | Default: no (AI vision unchanged, forward camera) | P09 | Team + guide |
| **OD-14** | **Test site for 40–60 m flight** | Open, feature-rich ground (roads, buildings, field edges), clear to 200 m+, permission and altitude limit confirmed | Before stage A2 | Team + guide + institute |
| OD-15 | Matching method: SIFT or XFeat | Decide from the comparison on recorded data | Gate G2 | Team, on evidence |
| OD-16 | Drop the HDMI converter to recover mass and power? | **Closed by DB-3.0: yes.** The app uses the Ethernet link instead ([ADR-017](17-decisions/ADR-017-ground-app.md)) | — | — |
| OD-13 | (see above) | **Closed by DB-3.0: yes**, in the search profile, with an aerial-view model ([ADR-018](17-decisions/ADR-018-search-track-follow.md)) | — | — |
| **OD-17** | **Who builds the Android app** | One team member owning it from week 4; needs Kotlin/Android experience or time to learn | G1 | Team |
| **OD-18** | **Is follow in scope for the final demonstration?** | Build order puts it last; decide at week ≈ 28 from progress | Week 28 | Team + guide |
| OD-19 | Search targets and classes for the demonstration | Person-sized dummies and vehicles by default (extends OD-2) | P19 | Team + guide |
| OD-20 | Participant rules for follow tests | A consenting, briefed team member only; institute approval | Before stage N | Team + guide + institute |
| **OD-22** | **Confirm the radio and the stereo camera before purchase.** The team owns no hardware (corrected 2026-10-06). The MK15 (≈ ₹63,000) and the Waveshare stereo camera were chosen partly because they were believed to be owned | Keep the MK15 if the budget allows, since the app design is built on it; otherwise a cheaper RC link plus a separate IP link and an ordinary Android phone or tablet, recorded in a new ADR. For the stereo camera, weigh a hardware-synchronised unit against the Waveshare board (ADR-011) | Before parts are ordered | Team + guide |
| OD-21 | Reference imagery at ≈ 0.25 m/px or own orthomosaic for the search profile (tightens OD-11) | Own orthomosaic recommended | Before P19 | Team |

## Hardware Pending

| ID | Item | Status | Action |
|---|---|---|---|
| HP-1 | Holybro Pixhawk 6C + PM02 + M10 GPS | To buy | Order from an authorised reseller |
| HP-2 | MicoAir MTF-01 | To buy | Order with HP-1 |
| HP-3 | Raspberry Pi Active Cooler, A2 microSD, 15→22-pin CSI cables | To buy | Order now: unblocks P04 |
| HP-4 | 5 V ≥ 5 A BEC, wiring, connectors, fuses | To buy | Order with HP-1 |
| HP-0 | Raspberry Pi 5, stereo camera, SIYI MK15 | **To buy** (earlier recorded as owned; corrected 2026-10-06) | Confirm choices first (OD-22) |
| HP-5 | MK15 checks: air-unit voltage label (4S support), converter connector and supply, operating band | To inspect | Inspect the unit on arrival; record in [siyi-mk15.md](03-hardware/siyi-mk15.md) §7 |
| HP-6 | Airframe, motors, ESCs, props, batteries, charger | Decision pending (OD-1) | — |
| HP-7 | Calibration target | To make | Print and mount |
| HP-8 | Weigh all components as they arrive | To do | Update [weight-budget.md](03-hardware/weight-budget.md) |
| HP-9 | Camera upgrade (OAK-D Lite or equivalent) | Conditional | Only after gate G2; hold budget |
| HP-10 | Waveshare board checks: IMU address, INT pin, supplied cables | To inspect | Record in [low-level-design.md](03-hardware/low-level-design.md) §13 |
| **HP-11** | **Downward USB camera** | To buy (OD-12) | Order now: needed for the gate G2 data collection |
| **HP-12** | **Reference image of the test site** | To obtain (OD-11) | Needed before P06G |
| **HP-14** | **MK15 IP-path check** (MK-9, MK-10, MK-11): third-party device on the air unit's Ethernet reachable from an Android app; cable pin-out; APK install | To do first: **gate G0** | A laptop on the air unit's Ethernet and a test app or browser on the MK15 are enough |
| HP-15 | Downward camera now ≥ 1920×1080 (replaces the 640×480 requirement in HP-11) | To buy (OD-12) | — |
| HP-16 | Search targets (dummies, markers); protective equipment for the follow participant | To make / buy | Before stages L and N |
| HP-13 | GNSS-tagged downward image set of the test site at 25–60 m, including dummy targets for the aerial detector | To record | Any camera drone flown manually on GPS is sufficient for the first feasibility test |

## Software Pending

| ID | Item | Phase |
|---|---|---|
| SP-1 | Ubuntu 24.04 development workstation (dual boot recommended) | P02 |
| SP-2 | Repository, `gdn_interfaces`, package skeletons, CI | P02 |
| SP-3 | Pi image: Ubuntu Server 24.04, ROS 2 Jazzy, Raspberry Pi libcamera fork, provisioning script | P02/P04 |
| SP-4 | ArduPilot SITL + MAVROS + `fake_vio`; parameter files | P03 |
| SP-5 | `gdn_camera`, `gdn_imu`, `gdn_diagnostics` | P04 |
| SP-6 | `gdn_nav_mode`, `gdn_localization`, `gdn_safety`, Lua watchdog | P05 |
| SP-7 | `gdn_stereo`, `gdn_obstacle`, `gdn_vo_simple`; calibration set | P06 |
| SP-8 | OpenVINS build and configuration; `gdn_vio`; evaluation tools | P07 |
| SP-9 | `gdn_perception`; NCNN build; model training and export | P09 |
| SP-10 | `gdn_sim` (Gazebo), `gdn_navigation`, `gdn_mission`, `gdn_telemetry` | P10 |
| SP-11 | `gdn_bringup`: launch, lifecycle manager, systemd, bag profiles | P02 → P12 |
| **SP-12** | `map_prepare` tool and map pack for the site; `gdn_geoloc` (`map_matcher`, `ground_vo`, geodetic library) | P06G, P07G |
| **SP-14** | Android app `gdn-ground` (Kotlin) and mock gateway | P18 |
| **SP-15** | `gdn_app_gateway` (`app_gateway`, `video_streamer`, tile server) | P18 |
| **SP-16** | Aerial dataset, aerial-view model, tiled inference; `search_planner`, `finding_manager` | P19 |
| SP-17 | `gdn_tracking` (`target_tracker`); follow controller and `FollowTarget` in `gdn_navigation` | P20 |
| SP-13 | Offset filter for map fixes in `gdn_localization`; regime switching and T19–T23 in `gdn_nav_mode`; fake map-fix source in `gdn_sim` | P03, P05, P07G |

## Items to verify at bring-up

Collected `[VERIFY]` items: [references.md](references.md) §7; research questions R-1 … R-10 in [18-research/README.md](18-research/README.md); hardware checks in [low-level-design.md](03-hardware/low-level-design.md) §13; MK15 checks in [siyi-mk15.md](03-hardware/siyi-mk15.md) §7.

## Documentation audit (2026-10-05)

| Check | Result | Where |
|---|---|---|
| Every major subsystem documented | Yes | Sections 02–13 |
| Hardware interfaces defined | Yes, to pin level; some pins tagged `[VERIFY]` | [low-level-design.md](03-hardware/low-level-design.md) |
| ROS 2 architecture defined | Yes: 19 packages, 22 nodes, all topics/services/actions, QoS, TF, lifecycle | Section 05 |
| FC integration defined | Yes: every message in both directions, parameters, failsafe communication | [mavlink-integration.md](10-communication/mavlink-integration.md) |
| VIO defined | Yes: primary, backup, simplified; configuration; health monitor | Section 07 |
| AI defined | Yes: role, model, runtime, training, deployment | Section 08 |
| Stereo vision defined | Yes: capture, sync, rectification, depth, obstacles, calibration | Section 06 |
| State estimation defined | Yes: two-stage fusion, state vectors, covariance, confidence | [state-estimation.md](09-navigation/state-estimation.md) |
| GPS-denied transition defined | Yes: states, guards, transition table, alignment | [gps-denied-state-machine.md](02-system-architecture/gps-denied-state-machine.md) |
| Safety defined | Yes: six layers, supervisor, watchdog, FMEA with 47 items | Section 12 |
| Simulation defined | Yes: three configurations, 20 scenarios, fault injection | Section 11 |
| Testing defined | Yes: eight levels with exit criteria | Section 13 |
| BOM defined | Yes: core, optional, future; prices as ranges | Section 15 |
| Decisions documented | Yes: 14 ADRs | Section 17 |
| Assumptions clearly marked | Yes: `[VENDOR]`, `[ESTIMATE]`, `[ASSUMPTION]`, `[MEASURE]`, `[VERIFY]`; collected in [references.md](references.md) §7 | Throughout |
| Unresolved questions identified | Yes: OD-1…OD-10, R-1…R-10, HP/SP lists | This file; [18-research](18-research/README.md) |
| Internal links | 68 files; all relative links resolve (script-checked) | — |
| Code created | None | — |

Known gaps accepted in this baseline:

- No measurement of any kind exists; every performance figure is a target or an estimate.
- Airframe-dependent values (thrust, flight time, mounting dimensions) are placeholders until OD-1 is closed.
- Mermaid diagrams were checked for syntax by inspection, not rendered; fix any that fail to render in the viewer used.
- ArduPilot parameter values are design intent and have not been run in SITL.

## Change log

| Date | Version | Change |
|---|---|---|
| 2026-10-05 | DB-1.0 | Initial design baseline created |
| 2026-10-05 | DB-2.0 | Concept change by the project owner: GPS-denied position from matching ground images to an onboard satellite image; AI vision unchanged. Added: visual-geolocalization.md, downward-camera.md, ADR-015, ADR-016, research note. Amended: requirements (FR-090–100, NFR-070–077, envelope), state machine (states renamed `VISION_*`, §8a), coordinate frames (§11), system architecture, hardware documents and budgets, ROS 2 packages/nodes/interfaces, state estimation (§10a), navigation, safety (§8a), FMEA (F48–F57), testing (§9a), performance (§4a), BOM, roadmap (§1a), simulation, contribution, comparison, technology selection, references. Earlier text that describes stereo VIO as the primary GPS-denied source now applies to the low regime only |

| 2026-10-06 | DB-3.0 | Added by the project owner: a native Android app on the MK15 (web app declined), grid search, track and follow. Added: ground-app.md, search-track-follow.md, ADR-017, ADR-018. Amended: requirements (FR-110–131, NFR-080–092, search profile), ROS 2 packages/nodes/interfaces, MK15 topology (B is now baseline; HDMI converter removed; `hud_node` retired), downward camera (1080p), visual geo-localisation (matching height from reference resolution), AI architecture (aerial model), safety (§8b), FMEA (F58–F69), testing (§9b), performance (§4b), roadmap (§1b, ≈ 42 weeks), BOM, budgets (back inside limits), technology selection, README |

| 2026-10-06 | DB-3.0 (diagram set) | Added [19-system-architecture-diagrams](19-system-architecture-diagrams/README.md): 94 diagrams in eight files covering the full system, hardware (whole, wiring, each part), software (whole, each part), Android app, radio and telemetry, the seven-phase software development life cycle, and all 14 UML diagram types; exported as 93 PNG and 93 SVG images in `rendered/`. **All Mermaid diagrams in the documentation were render-tested** with the Mermaid command-line tool: the 94 new ones and the 40 older ones. One older file, `03-hardware/high-level-architecture.md`, failed to render (unquoted "2.4 GHz" link labels) and was fixed. No design content changed |

| 2026-10-06 | DB-3.0 (build order) | Decided by the project owner: software first, hardware second. All software, including the Android app, is developed and proven in simulation (Part 1, ending at gate G3) before hardware work starts (Part 2). Added: root `README.md`, root `CONTRIBUTING.md`, [development-checklist.md](16-development-roadmap/development-checklist.md) (19 steps), roadmap §1c, interactive diagram `19-system-architecture-diagrams/interactive/system-explorer.html`. New gate G2-sim (map matching on public data and in simulation); gate G2 (own site) and gate G0 (MK15 IP path) move to Part 2. No design content changed |

| 2026-10-06 | DB-3.0 (hardware status) | Correction from the project owner: no hardware is owned; only a development laptop (i5-11300H, 24 GB, RTX 3050, Windows 11 with WSL2). Raspberry Pi 5, stereo camera and MK15 changed from Owned to To buy in the READMEs, hardware overview and BOM; BOM total revised to about ₹132,000–214,000; OD-22 and HP-0 added. Passages elsewhere that justify a choice by "already owned" (raspberry-pi-5.md, gps-denied-navigation.md, ADR-011, ADR-017, siyi-mk15.md, system-requirements NFR-060) are superseded on that point; the technical content is unchanged |

| 2026-10-06 | DB-3.0 (deadline) | Set by the project owner: completion by **15 January 2027**, with simulation over all terrain types. Deliverable for that date is Part 1 (complete system in simulation, gate G3) plus a terrain campaign and results on public datasets; hardware and flight (Part 2) follow afterwards. Stereo VIO with OpenVINS cut; follow, XFeat comparison and snow terrain are stretch items. Added: [datasets-and-terrains.md](11-simulation/datasets-and-terrains.md), checklist v3.0 with a 14-week calendar and checkpoints, roadmap §1d. Dataset downloads started (VisDrone, UAV-VisLoc) into a data folder outside the repository |

## DB-3.0 consistency notes

- Diagrams: every Mermaid block now renders. Earlier notes in this file saying the diagrams were "checked by inspection, not rendered" are superseded.

- Counts: 22 ROS packages plus one Android project, 29 nodes, 18 ADRs, 69 FMEA items, three flight profiles (low 1–10 m, search 25–30 m, cruise 40–60 m).
- Where older text mentions the HDMI converter, the HUD node or "video topology A", read: removed in DB-3.0; the app shows video over the IP link.
- "The app" always means the native Android app on the MK15. QGroundControl remains the standard ground station on the telemetry datalink.
- The rescue scenario is the motivation only. Nothing in the documentation claims performance in a real disaster area.
- Not verified in this revision: the MK15 IP path on the owned unit (gate G0), any Android behaviour on the MK15, aerial detection performance.

## DB-2.0 consistency notes

- Where an older section says "VIO tier" or "tier 2 = VIO", read "vision tier": map matching + ground odometry in the cruise regime, stereo VIO in the low regime.
- "Gate G2" in DB-1.0 text about the stereo camera is now **G2b**. **G2** is the map-matching feasibility gate.
- Counts: 20 packages, 25 nodes, 16 ADRs, 57 FMEA items.
- Mass and power budgets are now slightly over their limits on paper (OD-16).
- Not re-verified in this revision: the Mermaid diagrams that were edited, and all ArduPilot parameter values.
