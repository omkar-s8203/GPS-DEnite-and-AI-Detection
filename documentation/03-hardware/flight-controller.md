# Flight Controller Selection

| Field | Value |
|---|---|
| Document ID | GDN-HW-005 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Decision | **Holybro Pixhawk 6C running ArduPilot Copter 4.7.x** — [ADR-002](../17-decisions/ADR-002-autopilot-firmware.md), [ADR-003](../17-decisions/ADR-003-flight-controller.md) |

## 1. What the flight controller must provide

| Need | Why |
|---|---|
| External-navigation fusion (vision pose/velocity into the EKF) | Core of GPS-denied flight |
| Runtime switching between GNSS and external navigation | GNSS ↔ vision transition without landing |
| Optical-flow and range-sensor support | Tier-3 fallback |
| Onboard scripting or equivalent | Companion watchdog on the FC side |
| ≥ 2 MB flash MCU (STM32H7 or F7 with 2 MB) | ArduPilot removes features (including scripting and some visual-odometry support) on 1 MB boards |
| ≥ 4 free UARTs: GCS radio, companion, GNSS, flow sensor | Interface count |
| 921 600 baud capable companion UART at 3.3 V | Pi link |
| Redundant, vibration-isolated IMUs; barometer | Estimation quality on a vibrating prototype |
| S.Bus input | MK15 RC |
| Analog power-module input | Battery monitoring and failsafe |
| Good documentation; available in India | College project practicality |

## 2. Firmware: ArduPilot vs PX4

Both are mature and both support external vision. The comparison below is limited to what matters for this project.

| Criterion | ArduPilot Copter | PX4 |
|---|---|---|
| External vision fusion | EKF3 "ExternalNav" source; accepts `VISION_POSITION_ESTIMATE`, `VISION_SPEED_ESTIMATE`, `ODOMETRY`, `VISION_POSITION_DELTA` | EKF2 fuses `VISION_POSITION_ESTIMATE` / `ODOMETRY` (30–50 Hz); selected by `EKF2_EV_CTRL` |
| GNSS ↔ non-GNSS transition | **Explicit source sets** (`EK3_SRC1/2/3_*`), switchable by RC switch, Lua script, or `MAV_CMD_SET_EKF_SOURCE_SET` from a companion. Documented procedure for GPS/non-GPS transitions. | EKF2 fuses GNSS and vision together and handles loss of either internally; less explicit external control of "which source is active" |
| Third-tier fallback | Source set 3 = optical flow, selectable the same way | Optical flow fuses concurrently |
| Companion-side obstacle input | `OBSTACLE_DISTANCE` → proximity library → simple avoidance (documented for depth cameras) | `OBSTACLE_DISTANCE` → collision prevention |
| Onboard scripting | Lua scripts on H7 boards | No user scripting; modules in C++ |
| ROS 2 integration | MAVROS (mature, Jazzy binaries); native DDS (`AP_DDS`) exists but documentation targets ROS 2 Humble and the interface set is smaller | **Native uXRCE-DDS** bridge is first-class; MAVROS also works |
| Simulation | SITL + Gazebo (Harmonic supported by `ardupilot_gazebo`) | SITL + Gazebo (first-class) |
| GCS on the MK15 | QGroundControl works; Mission Planner on a laptop for tuning | QGroundControl is native |
| Testing aid | RC auxiliary function "GPS Disable" simulates GNSS loss in flight | GNSS loss injection through parameters/failure injection |
| Local familiarity | Very common in Indian colleges and hobby community with Pixhawk hardware | Less common locally |
| Licence | GPLv3 | BSD-3 |

**Decision: ArduPilot.** The deciding factor is the explicit, externally commandable EKF source-set mechanism, which maps one-to-one onto this project's central feature (GNSS → vision → flow tiers) and makes the transition observable and testable. PX4's better native ROS 2 bridge is a real advantage that is given up; MAVROS covers the need.

Firmware version: Copter 4.7.x stable (4.7.1 was released as stable in September 2026). All parameter names in this documentation must be checked against the 4.7 parameter list at FC bring-up.

## 3. Hardware candidates

| | Pixhawk 6C | Pixhawk 6C Mini | Pixhawk 6X | Cube Orange+ | Matek H743 (Slim/Wing) | Pixhawk 2.4.8 (clone) | F405-class FPV boards |
|---|---|---|---|---|---|---|---|
| MCU | STM32H743, 480 MHz, 2 MB flash, 1 MB RAM | STM32H743 | STM32H753 | STM32H757 | STM32H743 | STM32F427 (some clones have the 1 MB flash defect) | STM32F405, 1 MB |
| IMUs | ICM-42688-P + BMI088 | ICM-42688-P + BMI088 `[VERIFY]` | 3 IMUs | 3 IMUs | 2 IMUs (board-dependent) | Older MPU6000-class; clone quality varies | 1 IMU |
| IMU isolation / heating | Yes / yes | Yes / `[VERIFY]` | Yes / yes | Yes / yes | No / no | Foam / no | No / no |
| Baro / mag | MS5611 / IST8310 | Yes / yes `[VERIFY]` | 2 baro / mag | Yes / yes | Baro; no mag | Yes / yes | Baro; no mag |
| Telemetry UARTs | TELEM1–3 + GPS1–2 | Fewer (TELEM1–2 + GPS) `[VERIFY]` | More + Ethernet | TELEM1–2 + GPS1–2 | 7 UARTs (solder pads) | TELEM1–2 + GPS + SERIAL4/5 | 4–6 (pads) |
| CAN | 2 | 2 `[VERIFY]` | 2 | 2 | 1–2 | 1 | 0 |
| Connectors | JST-GH, Pixhawk standard | JST-GH | JST-GH | Carrier board | Solder pads | DF13 | Solder pads |
| ArduPilot | Supported | Supported | Supported | Supported | Supported | Supported, legacy | Supported with features removed |
| PX4 | Supported | Supported | Supported | Supported | Not an official target | Legacy | Limited |
| Lua scripting / external nav | Yes | Yes | Yes | Yes | Yes | Marginal/No on 1 MB | No |
| Mass | 34.6 g (plastic case) | ≈ 39 g `[VERIFY]` | Higher | Higher with carrier | ≈ 10 g | ≈ 38 g | ≈ 10 g |
| Approx. price in India (FC + power module + GNSS combo) | ≈ ₹25,000–37,000 `[VERIFY]` | ≈ ₹24,500–37,000 (listings seen) | Higher | Highest | Low (FC only; GNSS and PM extra) | Lowest | Lowest |
| Assessment | **Selected** | Acceptable alternate | More than needed | More than needed | Viable low-cost option for a team comfortable with soldering; no isolation | **Rejected** | **Rejected** |

Specifications for the Pixhawk 6C are from Holybro: STM32H743 FMU, STM32F103 IO processor, ICM-42688-P and BMI088 IMUs, IST8310 magnetometer, MS5611 barometer, integrated vibration isolation, IMU heating, 84.8 × 44 × 12.4 mm, 34.6 g in plastic case, port current limits of 1.5 A for TELEM1 + GPS2 combined and 1.5 A for the remaining ports combined.

### Why not the cheaper options

- **Pixhawk 2.4.8 clones:** obsolete sensors, inconsistent build quality, 1 MB-flash variants that cannot run the needed features. A flight controller with uncertain IMUs is the wrong place to save money on a project whose subject is state estimation.
- **F405 boards:** ArduPilot builds for 1 MB targets omit scripting and other features this design depends on.
- **Matek H743:** technically sufficient and inexpensive. Rejected as the baseline because it lacks vibration isolation and plug-in connectors, which costs time and reliability in a student build. It remains the fallback if budget forces it.

### Why not the more expensive options

Cube Orange+ and Pixhawk 6X add a third IMU and (6X) Ethernet. Neither addresses a risk this project has. The cost difference is better spent on the camera upgrade if gate G2 fails.

## 4. Selected configuration

| Item | Choice |
|---|---|
| Flight controller | Holybro Pixhawk 6C, plastic case (lighter) |
| Power module | Holybro PM02 (analog, up to 6S class) — the 4S build does not need the 12S/14S variants `[VERIFY the variant bundled]` |
| GNSS | Holybro M10 with IST8310 compass |
| Firmware | ArduPilot Copter 4.7.x stable |

If only the 6C Mini combo is available at a reasonable price, it is acceptable provided it exposes three independent MAVLink-capable serial ports besides GPS1 (GCS radio, companion, flow sensor). Otherwise the flow sensor moves to GPS2 or to DroneCAN.

## 5. Required interfaces (summary)

| Interface | Port | Peer |
|---|---|---|
| MAVLink 2, 921 600 baud | TELEM2 | Raspberry Pi 5 |
| MAVLink, 57 600 baud | TELEM1 | MK15 air unit |
| GNSS + compass | GPS1 | M10 |
| MAVLink 1, 115 200 baud | TELEM3 | MTF-01 |
| S.Bus | RC IN | MK15 air unit |
| Analog power | POWER1 | PM02 |
| PWM/DShot | MAIN/FMU outputs | 4 ESCs |

Pin-level detail: [low-level-design.md](low-level-design.md) §6.

## 6. Key ArduPilot configuration (design intent)

Parameter values are design intent to be validated in SITL and on the bench. The full list is in [mavlink-integration.md](../10-communication/mavlink-integration.md) §8.

| Group | Parameters |
|---|---|
| EKF | `AHRS_EKF_TYPE = 3`, `EK3_ENABLE = 1`, `EK3_SRC_OPTIONS = 0` |
| Source set 1 (GNSS) | `POSXY = 3`, `VELXY = 3`, `POSZ = 1`, `VELZ = 3`, `YAW = 1` |
| Source set 2 (vision) | `POSXY = 6`, `VELXY = 6`, `POSZ = 1`, `VELZ = 6`, `YAW = 1` (compass; `6` for indoor) |
| Source set 3 (flow) | `POSXY = 0`, `VELXY = 5`, `POSZ = 1`, `VELZ = 0`, `YAW = 1` |
| Visual odometry | `VISO_TYPE = 1` (MAVLink), `VISO_POS_X/Y/Z = 0`, `VISO_DELAY_MS` tuned |
| Flow and range | `FLOW_TYPE = 5`, `RNGFND1_TYPE = 10`, `RNGFND1_MAX = 8`, `RNGFND1_MIN = 0.01` (per MTF-01 guide) |
| Proximity | `PRX1_TYPE = 2`, `AVOID_ENABLE`, `AVOID_MARGIN`, `AVOID_BEHAVE = 1` (stop) |
| Scripting | `SCR_ENABLE = 1` |
| Failsafes | `FS_THR_ENABLE`, `FS_EKF_ACTION`, `BATT_FS_*`, `FENCE_*`, `GUID_TIMEOUT` |

## 7. Hardware limitations to note

| Limitation | Effect | Handling |
|---|---|---|
| Two IMUs, not three | No majority vote on IMU failure | Acceptable for a prototype under pilot supervision |
| Port power limits (1.5 A groups) | Cannot power the Pi or the MK15 from the FC | Separate BEC; FC powers only GNSS and MTF-01 |
| Single analog power input used | No redundant FC supply | Acceptable; USB is not a flight supply |
| TELEM serial at 3.3 V | Matches the Pi | No level shifter needed |
| Internal compass near power wiring | Interference | Use the external M10 compass as primary |
