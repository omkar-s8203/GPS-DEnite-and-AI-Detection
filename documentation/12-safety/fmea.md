# Failure Mode and Effects Analysis (FMEA)

| Field | Value |
|---|---|
| Document ID | GDN-SAF-002 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | Baseline — design-stage FMEA. Ratings are engineering judgement and are revised after bench and SITL testing. |

## 1. Rating scales

| Score | Severity (S) | Occurrence (O) | Detection (D) |
|---|---|---|---|
| 1–2 | No effect on flight; data or convenience loss | Unlikely during the project | Detected automatically and immediately |
| 3–4 | Test aborted; no damage | Occasional (once in tens of flights) | Detected automatically within about a second |
| 5–6 | Uncontrolled drift or hard landing; minor damage | Expected several times during the project | Detected by the crew through telemetry |
| 7–8 | Crash; significant vehicle damage | Frequent (most test days) | Detected only after effects appear |
| 9–10 | Risk of injury to people | Almost every flight | Not detectable |

RPN = S × O × D. Items with RPN ≥ 100 or S ≥ 9 require a stated mitigation and a verification test before flight.

## 2. FMEA table

| ID | Item | Failure mode | Cause | Local effect | System effect | S | O | D | RPN | Detection | Mitigation / response | Verified by |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| F01 | RC link | Loss of signal | Range, interference, ground-unit battery | No pilot input | Pilot cannot override | 8 | 2 | 1 | 16 | FC RC failsafe | `FS_THR_ENABLE` → RTL/LAND; short range; charged controller | L7, L8 |
| F02 | RC link | Wrong mode-switch mapping | Configuration error | Wrong mode selected | Unexpected behaviour on take-over | 8 | 3 | 4 | 96 | Pre-flight mode-switch check on GCS | Checklist item: cycle every switch and read the mode before each session | L7 |
| F03 | Flight controller | Reset / lock-up in flight | Brown-out, firmware fault, hardware defect | Loss of control | Crash | 9 | 1 | 9 | 81 | None in time | Genuine hardware; PM02 supply; stable firmware; 30 min bench soak; fly low over soft ground initially | L7 |
| F04 | Flight controller | Wrong EKF source parameters | Configuration error | EKF ignores or mis-uses external nav | Fly-away or drift at source switch | 8 | 4 | 3 | 96 | Pre-flight parameter check by the companion; SITL with the same file | Version-controlled param files; companion verifies key parameters; first switch in tethered hover | L5, L7 |
| F05 | Flight controller | Excess vibration | Unbalanced props, loose mount | Accelerometer clipping | Altitude and position errors; EKF failsafe | 7 | 4 | 3 | 84 | `VIBE` log; vibration telemetry | Balance props; check mounts; vibration limit as a flight-test gate | L8 (manual flights) |
| F06 | Companion | Total hang / kernel panic | Software fault, memory, kernel | No heartbeat, no external nav, no setpoints | In GUIDED on vision: loss of position aiding | 7 | 3 | 2 | 42 | FC Lua watchdog (2 s) | BRAKE → LOITER/ALT_HOLD; source set 3; EKF failsafe; pilot | L5, L7 |
| F07 | Companion | Power loss / brown-out | BEC fault, connector, battery sag | Same as F06 | Same as F06 | 7 | 3 | 2 | 42 | Same | Separate BEC with margin; locking connectors; bulk capacitor; bench load test | L7 |
| F08 | Companion | Single class-A node crash | Software fault | Missing data stream | Tier change; restart | 5 | 4 | 2 | 40 | Supervisor heartbeat (1 s) | Respawn; state machine leaves VIO tier; restart limit | L3, L5 |
| F09 | Companion | Navigator sends wrong setpoints | Logic or frame error | Vehicle moves unexpectedly | Possible collision | 8 | 3 | 5 | 120 | Pilot observation; fence | Frame checks CF-1…8; speed and tilt limits on the FC; geofence; soft fence; SITL regression; first autonomous motion is a 1 m step at 0.3 m/s; pilot ready | L4, L5, L8 |
| F10 | Companion | Thermal throttling | Poor cooling, hot day | CPU slows; latency rises | VIO degraded | 5 | 4 | 2 | 40 | `system_monitor` | Active cooler; load shedding; pre-flight NO-GO | L7 |
| F11 | Companion | CPU overload | Too many nodes, logging | Latency, dropped frames | VIO degraded | 5 | 5 | 2 | 50 | Latency and rate diagnostics | CPU budget; priorities; load shedding | L7 |
| F12 | Companion | Storage full or slow | Long recording | Bag drops messages | Data loss only | 2 | 4 | 2 | 16 | Disk check; recorder stats | Pre-flight check; recording profiles | L3 |
| F13 | Camera | One/both streams stop | Ribbon loose, driver fault | No images | VIO lost; no obstacle data | 6 | 4 | 1 | 24 | Driver timeout; `vio_monitor` | Tier 3; hold; strain relief; pre-flight tug test | L2, L5 |
| F14 | Camera | L/R desynchronised | Software sync unlock, CPU stall | Skewed pairs | Depth and VIO scale errors during rotation | 6 | 5 | 2 | 60 | Per-pair skew measurement | Drop bad pairs; confidence term; gate G2 | L2 |
| F15 | Camera | Motion blur / rolling-shutter distortion | Fast motion, long exposure, vibration | Poor features | VIO drift or loss | 6 | 6 | 3 | 108 | Feature count; cross-checks | Short fixed exposure; speed and yaw-rate limits; damped mount; camera upgrade path | L2, L7, G2 |
| F16 | Camera | Low light / low texture | Environment | Few features, depth holes | VIO lost; obstacles unknown | 6 | 5 | 2 | 60 | Gain at limit; feature count; valid fraction | Confidence; unknown ≠ clear; site selection; daylight only | L2, L4 |
| F17 | Camera | Calibration invalid | Knock, thermal, re-mount | Wrong geometry | Depth bias; VIO drift | 6 | 4 | 5 | 120 | Quick verification procedure (not automatic) | Verification before each test day; recalibration triggers; calibration ID in logs | L2 |
| F18 | VIO IMU | Dropouts / jitter | I²C errors, CPU | Bad inertial data | VIO degraded or diverges | 6 | 4 | 2 | 48 | Gap and rate diagnostics | Interrupt-driven reads if available; priority; cross-checks | L2 |
| F19 | VIO IMU | Saturation / vibration noise | Hard mount, prop imbalance | Clipped data | VIO diverges | 6 | 4 | 3 | 72 | Saturation counter; noise level check | Range ±8 g / ±500 °/s; damped mount; vibration bench test | L7 |
| F20 | VIO | Tracking loss | F15, F16 | No output or frozen pose | Loss of tier 2 | 6 | 5 | 1 | 30 | Rate/gap monitor | T11 → flow tier | L5 |
| F21 | VIO | Divergence with plausible output | Bad initialisation, wrong calibration, timing | Wrong pose, small covariance | **FC follows wrong position: fly-away tendency** | 8 | 4 | 4 | 128 | Cross-checks vs FC attitude, gyro, baro; EKF innovation gate; pilot | Confidence minimum-of-checks; EKF failsafe; speed limit; fence; tethered first flights | L4, L5, L8 |
| F22 | VIO | Re-initialisation in flight | Recovery after loss | New origin and yaw | Position jump if fed directly | 8 | 3 | 2 | 48 | State change + step detector | Stream withheld until re-aligned; reset counter | L1, L5 |
| F23 | VIO | Slow drift | Inherent | Position error grows | Vehicle wanders from intended place | 5 | 9 | 7 | 315 | Only against a reference | **Accepted and bounded**: limit GPS-denied duration (≤ 3 min) and distance (≤ 60 m) per test; open area; measure and report drift; pilot supervision | L8 |
| F24 | Alignment | Wrong `map → odom` | Poor GNSS during alignment, compass error, too little motion | External nav offset/rotated relative to EKF | Position step and wrong direction of motion at switch | 8 | 4 | 3 | 96 | Alignment residual; `dv` check | `aligned` gate; freeze with look-back; first switch in hover; step limit check | L1, L5, L8 |
| F25 | Frame conventions | ENU/NED or FLU/FRD error | Implementation mistake | Axis flipped | Vehicle accelerates the wrong way on vision: diverging | 9 | 3 | 2 | 54 | Bench checks CF-1…CF-8; SITL | Single conversion point (MAVROS); mandatory bench gate; SITL first | L3, L5, L7 |
| F26 | Time sync | Wrong delay / offset | Uncalibrated `VISO_DELAY_MS`, unstable timesync | Vision fused at wrong time | Oscillation or poor hold | 6 | 4 | 4 | 96 | EKF innovation analysis (post-flight); timesync monitor | Delay calibration procedure; conservative gains; hover test | L6, L8 |
| F27 | GNSS monitor | Missed degradation | Thresholds too loose; spoof-like slow drift | Stays on bad GNSS | EKF follows wrong GNSS | 7 | 3 | 5 | 105 | `dv` check; EKF variance | Tune on site logs; velocity-consistency check; pilot source switch | L5 |
| F28 | GNSS monitor | False denial | Thresholds too tight | Unnecessary switch to vision | Reduced accuracy; no hazard | 3 | 5 | 2 | 30 | Event log | Hysteresis and dwell; tuning | L5 |
| F29 | Source switching | Command not executed | Link loss, rejected command | Stays on failed source | EKF failsafe | 7 | 2 | 2 | 28 | ACK timeout | Retries; announce; pilot RC source switch | L5 |
| F30 | Source switching | Oscillation between sources | Marginal GNSS | Repeated position steps | Unstable hold | 6 | 3 | 2 | 36 | Event rate | 10 s validation; auto-return disabled by default | L5 |
| F31 | Source switching | Large step on return to GNSS | Accumulated vision drift | Position jump | Sudden manoeuvre | 6 | 6 | 2 | 72 | Offset computed before switching | Switch only when slow; report; pilot-initiated by default | L5, L8 |
| F32 | Optical flow / range | Invalid over poor surface or above 8 m | Environment | Tier 3 unavailable | Tier 4 on VIO loss | 6 | 4 | 2 | 48 | Flow quality; range validity | Altitude limit; site selection; state machine checks `flow_ok` before relying on it | L5, L8 |
| F33 | Obstacle detection | Missed obstacle | Thin, texture-less, outside FOV, dark | No stop | Collision at ≤ 2 m/s | 7 | 5 | 6 | 210 | Pilot | Low speed; nose-first rule; unknown ≠ clear; open test site; pilot supervision; **documented limitation** | L4, L8 |
| F34 | Obstacle detection | False obstacle | Depth noise, ground in band | Unnecessary stop | Mission delay | 2 | 6 | 2 | 24 | Event log | Percentile + pixel count; band levelling | L4 |
| F35 | AI detector | False negative (person not detected) | Model limits | No label | Mission rule not triggered | 7 | 6 | 7 | 294 | None automatic | **AI is never the safety barrier**: depth-based stop still applies; site rules keep people away; never treat "no detection" as "clear" | Procedure |
| F36 | AI detector | False positive | Model limits | Phantom object | Unnecessary hold | 2 | 6 | 3 | 36 | Review | N-consecutive rule | L2 |
| F37 | AI detector | Crash / overload | Software, CPU | No detections | None on flight | 1 | 4 | 1 | 4 | Supervisor | Advisory class; shed first | L3 |
| F38 | MAVLink link | Serial errors / disconnect | Connector, EMI, baud mismatch | Data loss | As F06 if total | 7 | 3 | 2 | 42 | MAVROS diagnostics; FC watchdog | Locking connector; twisted wiring; bench error-rate test | L6 |
| F39 | Battery | Low voltage | Long test, old pack, cold | Reduced thrust | Forced landing | 6 | 4 | 1 | 24 | FC battery failsafe; alarm | Calibrated monitor; conservative thresholds; flight timer | L7 |
| F40 | Power | BEC output fault (high) | Component failure | Pi and camera destroyed | As F07 for flight; hardware loss | 7 | 1 | 2 | 14 | FC watchdog | Quality BEC; optional over-voltage clamp | — |
| F41 | Compass | Interference | Currents, structures, indoors | Yaw error | Toilet-bowling; wrong heading for alignment | 7 | 4 | 3 | 84 | EKF mag innovations; pre-arm check | Mast; `COMPASS_MOT`; vision yaw option indoors | L8 |
| F42 | EMI | Pi/HDMI/CSI noise into GNSS | Layout | Fewer satellites | False GNSS degradation in normal flight | 4 | 5 | 2 | 40 | Bench satellite/C/N0 comparison | Mast; shielding; no USB 3 | L7 |
| F43 | Mechanical | Camera mount shifts | Vibration, landing | Extrinsics change | F17 | 6 | 3 | 5 | 90 | Quick verification | Rigid bar; thread-lock; inspection | L7 |
| F44 | Human | Pilot slow to take over | Surprise, inexperience | Delayed recovery | Drift or collision | 8 | 4 | 6 | 192 | — | Pilot proficiency in ALT_HOLD/STABILIZE; briefing; observer call-outs; simulator practice; tether/net early | Procedure |
| F45 | Human | Wrong parameter file / software version flown | Process error | Unknown configuration | Unpredictable | 7 | 3 | 3 | 63 | Pre-flight parameter check; version in status text | Version-controlled configs; run ID procedure | L7 |
| F46 | Lua watchdog | Script not running or faulty | Not loaded, error | No reaction to companion loss | Reliance on native failsafes only | 6 | 2 | 2 | 24 | "Alive" value checked pre-flight | SITL test; pre-flight check | L5, L7 |
| F47 | Geofence | Ineffective without position | Tier 4 | No containment | Drift out of area | 7 | 2 | 3 | 42 | State known | Pilot takeover procedure; auto-LAND timer | L5 |

### DB-2.0 additions (satellite map matching, cruise regime)

| ID | Item | Failure mode | Cause | Local effect | System effect | S | O | D | RPN | Detection | Mitigation / response | Verified by |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| F48 | Map matcher | Wrong fix accepted | Repetitive scene, changed scene, shadows | Position pulled toward a wrong place | Vehicle moves off its intended track by up to the gate size | 8 | 4 | 4 | 128 | Innovation gate; consistency with previous fix; shadow-mode statistics | Geometric check, gates, slew-limited application, fence, spotter, GPS-restore rule | L1, L3, L8-B |
| F49 | Map matcher | No fixes | Featureless or changed terrain, season, haze, low sun | Position carried by odometry; uncertainty grows | Hold, then pilot or land | 5 | 6 | 2 | 60 | Fix age, acceptance ratio | Site with distinct features; recent imagery; own orthomosaic as alternative; degrade steps T20/T22 | L3, L8-B |
| F50 | Reference map | Georeference offset, wrong or outdated map pack | Provider error, wrong file, old image | Constant bias or no matches | Navigation offset from true coordinates | 6 | 4 | 3 | 72 | Shadow-mode mean error vs GNSS; map ID check | Site registration procedure; pre-flight map check; NO-GO above 5 m mean error | L7, L8-B |
| F51 | Reference map | Vehicle leaves coverage | Mission or drift toward the map edge | No reference to match | Loss of fixes | 6 | 3 | 1 | 18 | Distance-to-edge | Goals outside coverage refused; 60 m margin; turn back | L1, L4 |
| F52 | Downward camera | Stream lost, lens dirty, over- or under-exposed | Cable, USB fault, sun, dust | No ground VO and no fixes | **No horizontal source at height** | 8 | 3 | 2 | 48 | Driver timeout; exposure monitor | Shielded short cable, strain relief, lens hood, pre-flight image check; T22; GPS-restore rule | L2, L7 |
| F53 | Height estimate | Wrong height above ground | Barometer drift, terrain relief | Wrong image scale; ground VO velocity scaled wrongly | Odometry drift; fewer accepted fixes | 5 | 4 | 3 | 60 | Matcher scale estimate vs expected | Scale gate; slow bias correction; flat site | L3, L8-B |
| F54 | Cruise regime | All vision lost at 40–60 m with flow fallback out of range | F49 + F52, companion crash | No position estimate at height | Drift during a 30–40 s descent; possible exit from the test area | 9 | 3 | 4 | 108 | State machine; FC watchdog; EKF failsafe | GPS-restore rule (denial is simulated); large clear site; wind limit; pilot practised in ALT_HOLD from height; spotter | L5, L8 |
| F55 | Attitude / heading | Compass or attitude error at image time | Interference, timing | Orthorectified image rotated or shifted | Match rejected or biased by `h·tan δ` | 5 | 4 | 3 | 60 | Matcher residual rotation; rejection rate | Compass on mast; attitude interpolated to image time; rotation tolerance ± 15° | L3, L7 |
| F56 | Compliance | Reference imagery used against its licence | Convenience | — | Result cannot be published or distributed | 4 | 3 | 2 | 24 | Map pack metadata review | Licence recorded per map pack; own orthomosaic as a clean alternative | Review |
| F57 | CPU | Stereo and geo-localisation nodes active together | Regime switching fault | Overload | Latency; missed fixes | 5 | 3 | 2 | 30 | CPU and latency diagnostics | Height-based lifecycle activation with hysteresis; load shedding | L3 |

### DB-3.0 additions (ground app, search, track, follow)

| ID | Item | Failure mode | Cause | Local effect | System effect | S | O | D | RPN | Detection | Mitigation / response | Verified by |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| F58 | Aerial detector | Person or object in the search area not detected | Small target, viewpoint, clutter, light, model limits | No finding | A search reports nothing where something is present | 7 | 7 | 7 | 343 | None automatic | Measure and display recall; label results "covered", never "clear"; low height, high-resolution camera, confirmation over frames; human review; thermal camera named as the real remedy | L2, L8 |
| F59 | Aerial detector | False finding | Clutter, shadows | Extra pin | Operator time wasted | 2 | 7 | 2 | 28 | Operator review | Multi-frame confirmation; reject button | L2 |
| F60 | Finding position | Coordinates wrong by more than the stated error | Drone position error, attitude error, height error | Pin in the wrong place | Responders sent to the wrong spot | 6 | 4 | 4 | 96 | Uncertainty shown with each finding | Findings suppressed when position uncertainty > 10 m; uncertainty displayed; thumbnail with surroundings | L8 |
| F61 | Target tracker | Follows the wrong object | Similar objects cross; detection gap | Target identity switches | Drone follows something else | 5 | 5 | 4 | 100 | Operator sees the box in the app | Gate on predicted position; hold when ambiguous; operator Stop; follow is from above so the consequence is benign | L1, L4 |
| F62 | Follow | Vehicle leaves the intended area after a moving target | Target walks out | Approaches boundary | Fence or map-edge event | 5 | 4 | 1 | 20 | Boundary check | Hold at geofence and map margin; time limit | L5 |
| F63 | App link | Lost during a mission | Range, interference, app crash | No operator commands or video | Mission continues unobserved | 4 | 5 | 1 | 20 | Heartbeat | Link-loss rules per behaviour; pilot and QGroundControl unaffected | L5, L7 |
| F64 | App | Operator commands something unintended | Mis-tap, misunderstanding | Unwanted behaviour starts | Vehicle moves unexpectedly | 6 | 4 | 3 | 72 | Operator, pilot | Two-step start for follow and search; plan preview before start; Stop on every screen; all gates still apply; pilot override | L7 |
| F65 | App gateway | Malformed or hostile input | Bug, interference | Bad request | Could command an invalid area | 6 | 2 | 2 | 24 | Validation counters | All validation on the Pi; ranges, polygon checks, rate limit, single client, shared key | L1, L3 |
| F66 | Search profile | Map matching weak at 25–30 m with coarse imagery | Reference resolution too low | Few fixes | Search pauses repeatedly or positions drift | 5 | 5 | 2 | 50 | Fix age, uncertainty | Reference of ≈ 0.25 m/px or better; own orthomosaic; climb-for-fix fallback | L3 (gate G2) |
| F67 | CPU | Aerial detector starves localisation | Tiled inference load | Late fixes, odometry gaps | Navigation degraded | 6 | 4 | 2 | 48 | Latency diagnostics | Detector at low priority and limited threads; rate drops before localisation is affected | L3, L7 |
| F68 | Scene change (disaster) | Ground differs from the stored image | Damage, flooding | Matches fail or mislead | No reliable position where it is most needed | 7 | 8 (in a real disaster) | 3 | 168 | Fix acceptance rate | **Not addressed by this project**; stated as a limitation; odometry carries for a limited time; learned matcher may help | Stated |
| F69 | People | Vehicle above or near uninvolved people during search or follow | Site control failure | — | Injury risk | 9 | 2 | 3 | 54 | Spotter | Site rules; consenting participants only; height ≥ 20 m; no automatic descent | Procedure |

F58 becomes the highest-RPN item in the analysis. Like F35 it is a limit of the sensing, controlled by honest reporting and human review, not a defect that engineering in this project removes.

With DB-2.0, F23 (slow drift of vision odometry) no longer dominates in the cruise regime: map fixes bound it. It remains as stated for the low regime.

## 3. Ranking of the highest risks

| Rank | ID | RPN | Summary | Primary control |
|---|---|---|---|---|
| 1 | F23 | 315 | Slow VIO drift | Bound duration and distance; measure and report |
| 2 | F35 | 294 | Person not detected by AI | AI is not a safety barrier; site rules |
| 3 | F33 | 210 | Obstacle missed | Low speed; open site; pilot |
| 4 | F44 | 192 | Pilot slow to take over | Training, briefing, tether/net |
| 5 | F21 | 128 | VIO divergence with plausible output | Independent cross-checks; EKF gate |
| 6 | F09 | 120 | Wrong setpoints | Frame checks; FC limits; staged first motion |
| 6 | F17 | 120 | Calibration invalid | Verification routine |
| 8 | F15 | 108 | Blur / rolling shutter | Envelope limits; gate G2 |
| 9 | F27 | 105 | Missed GNSS degradation | Consistency check; tuning |

DB-2.0 entries that join this list: **F48** wrong map fix accepted (RPN 128, rank 5 equal) and **F54** all vision lost at cruise height (RPN 108, and the only new item with severity 9). Both are controlled mainly by the fact that GPS denial is simulated and can be reversed instantly, and by site and crew rules ([safety-architecture.md](safety-architecture.md) §8a).

Observations:

- The three highest items are **limits of the sensing approach**, not defects to be fixed. They are controlled by operating envelope, site and supervision, and must be reported as limitations.
- F25 (frame error) has a modest RPN only because detection is good **if** the bench checks are actually performed. They are a hard gate.
- No item with S ≥ 9 lacks a control, but F03 (FC failure) has no in-flight mitigation; it is controlled by hardware quality and by keeping people out of the flight area.

## 4. Mandatory verification before first GPS-denied flight

| FMEA items | Required evidence |
|---|---|
| F04, F45 | Companion parameter check passes with the flight parameter file |
| F06, F07, F46 | Watchdog test with props off: kill companion in GUIDED → mode change observed |
| F09, F25 | CF-1…CF-8 passed; SITL scenarios S-03, S-04, S-15 passed with the same software commit |
| F21, F22 | SITL S-08; bench: cover the lens, shake, restart VIO → stream withheld, tier change |
| F24 | SITL S-02; outdoor shadow-mode run showing alignment residuals within limits |
| F26 | Delay calibration recorded |
| F01, F02 | RC failsafe and switch mapping test, props off |
| F44 | Pilot has flown the vehicle manually in ALT_HOLD and STABILIZE for ≥ 5 battery packs |

## 5. Maintenance of this FMEA

Reviewed at each roadmap gate (G1–G5) and after every incident. Occurrence and detection scores are updated from test evidence; new failure modes found in testing are added with the next free ID.
