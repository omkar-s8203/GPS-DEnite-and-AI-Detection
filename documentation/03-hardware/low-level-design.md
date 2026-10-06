# Low-Level Hardware Design

| Field | Value |
|---|---|
| Document ID | GDN-HW-002 |
| Version | 1.0 |
| Date | 2026-10-05 |
| Status | Baseline — pin-level items tagged `[VERIFY]` must be checked against the actual boards before wiring |

## 1. Component-level architecture

| Ref | Component | Key devices | Role |
|---|---|---|---|
| U1 | Raspberry Pi 5 (8 GB) | BCM2712, RP1 I/O controller, PMIC | Companion computer |
| U2 | Waveshare IMX219-83 | 2 × Sony IMX219, ICM-20948 | Stereo camera + VIO IMU |
| U3 | Holybro Pixhawk 6C | STM32H743 (FMU), STM32F103 (IO), ICM-42688-P, BMI088, IST8310, MS5611 | Flight controller |
| U4 | Holybro PM02 | Shunt + 5.2 V regulator | FC power and battery sensing |
| U5 | Holybro M10 | u-blox M10 GNSS, IST8310 compass, safety switch, LED, buzzer `[VERIFY]` | GNSS + compass |
| U6 | MicoAir MTF-01 | Optical-flow sensor + ToF | Flow + height |
| U7 | SIYI MK15 air unit | RF transceiver, router | RC, telemetry, video link |
| U8 | SIYI HDMI input converter | H.265 encoder | HDMI → Ethernet |
| U9 | 5 V BEC | Buck converter ≥ 5 A | Pi supply |
| U10 | 12 V regulator | Buck converter ≥ 0.5 A | HDMI converter supply |
| U11–14 | ESCs | BLHeli-class, 4S | Motor drive |

## 2. Voltage domains

| Domain | Nominal | Range | Source | Loads |
|---|---|---|---|---|
| VBAT | 14.8 V (4S) | 12.8–16.8 V | LiPo | ESCs, PM02, BEC, 12 V regulator, MK15 air unit |
| V5_FC | 5.2 V | 4.9–5.5 V | PM02 | Pixhawk and its port-powered peripherals |
| V5_PI | 5.1 V | 4.95–5.25 V at the Pi | BEC U9 | Raspberry Pi 5 |
| V12 | 12 V | 11.5–12.5 V | U10 | HDMI converter |
| V3V3_PI | 3.3 V | — | Pi PMIC | GPIO logic, camera board, ICM-20948 |
| V3V3_FC | 3.3 V | — | Pixhawk | FC logic, UART signalling |
| V5_PERIPH | 5 V | — | Pixhawk ports | GNSS, MTF-01 |

Rules:

- **One ground reference.** All supplies return to the PDB ground. Signal grounds are connected wherever a signal crosses between devices.
- **Never back-power.** The 5 V pin of the Pixhawk TELEM2 connector is **not** connected to the Pi. The Pi and the FC have separate 5 V supplies and share only ground and signals.
- **All inter-device logic is 3.3 V.** Pi GPIO is not 5 V tolerant.

## 3. Interface table (master)

| # | Source | Destination | Interface | Protocol | Data | Rate | Voltage | Notes |
|---|---|---|---|---|---|---|---|---|
| I1 | IMX219 left | Pi CAM/DISP 0 | MIPI CSI-2, 2 lanes | CSI-2 RAW10 | Left image | 20 fps at 1640×1232 binned (downscaled to 640×480) | LVDS, 3.3 V supply | 15-pin (camera) to 22-pin (Pi 5) cable |
| I2 | IMX219 right | Pi CAM/DISP 1 | MIPI CSI-2, 2 lanes | CSI-2 RAW10 | Right image | Same | Same | Same |
| I3 | Pi | IMX219 ×2 | I²C in CSI cable | I²C (sensor control) | Exposure, gain, frame length | On demand | 3.3 V | Kernel driver `imx219` |
| I4 | ICM-20948 | Pi I²C-1 (GPIO2/3) | I²C | Register read | Gyro + accel | ~225 Hz burst reads, 400 kHz bus | 3.3 V | Address 0x68 `[VERIFY]`; separate jumper wires from camera board pads |
| I5 | Pi UART0 TX (GPIO14, pin 8) | Pixhawk TELEM2 RX | UART | MAVLink 2 | External nav, setpoints, obstacles, heartbeat | 921 600 baud 8N1 | 3.3 V | Cross TX↔RX |
| I6 | Pixhawk TELEM2 TX | Pi UART0 RX (GPIO15, pin 10) | UART | MAVLink 2 | State, GNSS, EKF status, RC, battery | 921 600 baud 8N1 | 3.3 V | |
| I7 | Pi micro-HDMI 0 | HDMI converter | HDMI | TMDS | HUD video | 1080p30 or 720p30 | — | Connector type on converter `[VERIFY]` (micro vs mini) |
| I8 | HDMI converter | MK15 air unit | Ethernet (8-pin GH) | RTSP / H.265 | Video | ≈ 12 Mbit/s `[VENDOR]` | — | Converter default IP 192.168.144.25 `[VENDOR]` |
| I9 | MK15 air unit | Pixhawk RC IN | S.Bus | Inverted serial 100 kbaud | 16 RC channels | ~70 Hz `[VERIFY]` | 3.3 V | 3-pin GH on air unit |
| I10 | MK15 air unit | Pixhawk TELEM1 | UART (4-pin GH) | MAVLink | GCS telemetry and commands | 57 600 baud (configurable) | 3.3 V | |
| I11 | M10 GNSS | Pixhawk GPS1 | UART + I²C + switch/LED | UBX / I²C | Position, velocity, compass | 5–10 Hz GNSS, 115 200+ baud | 3.3 V logic, 5 V power | 10-pin GH |
| I12 | MTF-01 | Pixhawk TELEM3 (or GPS2) | UART | MAVLink 1 | `OPTICAL_FLOW`, `DISTANCE_SENSOR` | 115 200 baud, 100 Hz output `[VENDOR]` | 3.3 V logic, 5 V power | `SERIALx_OPTIONS = 1024`; sensor `mav_id ≠ 1` `[VENDOR]` |
| I13 | Pixhawk MAIN OUT 1–4 | ESCs | PWM / DShot | DShot300/600 or PWM | Motor commands | 400 Hz+ | 3.3 V | DShot availability depends on output group `[VERIFY]` |
| I14 | PM02 | Pixhawk POWER1 | 6-pin | Analog | 5.2 V supply, voltage sense, current sense | Continuous | 5.2 V; sense 0–3.3 V | |
| I15 | Pi | microSD / NVMe | SDIO / PCIe 2.0 ×1 | — | OS, rosbag2 | ≥ 30 MB/s sustained write needed for raw image logging | — | |
| I16 | Pi | Dev laptop | Wi-Fi 5 GHz | SSH, DDS (bench only) | Debug | — | — | Radio disabled or 5 GHz only in flight |
| I17 | Pi fan header | Active cooler | 4-pin PWM | — | Cooling | — | 5 V | |
| I19 | Pi Ethernet (DB-3.0) | MK15 air unit Ethernet (8-pin GH) | 100BASE-TX, 4 wires | IP: WebSocket control, JPEG video, HTTP tiles | App link | ≈ 2–3 Mbit/s | — | Static 192.168.144.50; custom GH-to-RJ45 cable `[VERIFY pin-out]`. **Replaces I7 and I8: the HDMI converter and regulator U8, U10 are removed in DB-3.0** |
| I18 | Downward camera (DB-2.0) | Pi USB 2.0 port | USB 2.0 | UVC (MJPEG/YUYV) | Ground image 640×480 | 15 fps; ≈ 5–40 Mbit/s depending on format | 5 V from the port, ≤ 300 mA `[VERIFY]` | Short shielded cable; never a USB 3 port/cable. See [downward-camera.md](downward-camera.md) |

## 4. Bus usage summary

| Bus | Used | Devices | Not used / reserved |
|---|---|---|---|
| **UART** | Pi UART0 ↔ FC TELEM2; FC TELEM1 ↔ MK15; FC GPS1 ↔ GNSS; FC TELEM3 ↔ MTF-01 | — | Pi UART2–4 spare |
| **USB** | Pi USB 2.0 ↔ downward camera (DB-2.0) | UVC camera | FC USB for bench configuration only; Pi USB 3 ports left unused in flight (2.4 GHz noise) |
| **I²C** | Pi I²C-1 ↔ ICM-20948; FC I²C ↔ compass (in GPS cable) | — | FC I²C port spare |
| **SPI** | Internal to FC (IMUs, baro) only | — | Pi SPI0 spare |
| **CAN** | Not used | — | FC CAN1/CAN2 spare for a future DroneCAN GNSS or range sensor |
| **CSI-2** | Pi CAM0, CAM1 | Both camera sensors | — |
| **Ethernet** | MK15 air unit ↔ HDMI converter | — | Pi Ethernet spare (bench; alternative link, see [siyi-mk15](siyi-mk15.md) §6) |
| **GPIO** | Pi GPIO14/15 UART, GPIO2/3 I²C, 5 V/GND power pins | — | Optional: GPIO for IMU data-ready interrupt if the board exposes INT `[VERIFY]` |

## 5. Raspberry Pi 5 40-pin header allocation

| Pin | GPIO | Function | Connects to |
|---|---|---|---|
| 2, 4 | 5V | Power input | BEC + (two pins in parallel) |
| 6, 9, 14 | GND | Ground | BEC −, FC TELEM2 GND, IMU GND |
| 3 | GPIO2 | I²C1 SDA | ICM-20948 SDA |
| 5 | GPIO3 | I²C1 SCL | ICM-20948 SCL |
| 8 | GPIO14 | UART0 TXD | TELEM2 pin 3 (RX) |
| 10 | GPIO15 | UART0 RXD | TELEM2 pin 2 (TX) |
| 11 | GPIO17 | Optional: IMU INT input | ICM-20948 INT `[VERIFY board exposes it]` |
| 36 / 11 | GPIO16 / GPIO17 | Optional UART0 CTS / RTS | TELEM2 RTS / CTS — only if flow control is enabled `[VERIFY alt-function mapping on Pi 5]` |

Pi configuration items (`/boot/firmware/config.txt`), to be confirmed at bring-up `[VERIFY]`:

| Setting | Purpose |
|---|---|
| `dtoverlay=imx219,cam0` and `dtoverlay=imx219,cam1` | Enable both sensors `[VENDOR]` |
| `camera_auto_detect=0` | Use explicit overlays |
| `dtparam=uart0=on` | PL011 UART on GPIO14/15 (`/dev/ttyAMA0`) |
| `dtparam=i2c_arm=on,i2c_arm_baudrate=400000` | I²C-1 at 400 kHz |
| `usb_max_current_enable=1` | Allow full USB current when powered through GPIO |
| Disable serial console on the UART | The UART is for MAVLink, not a login shell |

## 6. Pixhawk 6C port allocation

| Port | ArduPilot serial | Protocol | Baud | Peer |
|---|---|---|---|---|
| TELEM1 | `SERIAL1` | MAVLink 2 (`SERIAL1_PROTOCOL = 2`) | 57 (57 600) | MK15 air unit datalink |
| TELEM2 | `SERIAL2` | MAVLink 2 (`SERIAL2_PROTOCOL = 2`) | 921 (921 600) | Raspberry Pi 5 |
| GPS1 | `SERIAL3` | GPS (`SERIAL3_PROTOCOL = 5`) | Auto | M10 |
| GPS2 | `SERIAL4` | Spare | — | — |
| TELEM3 | `SERIAL5` `[VERIFY index]` | MAVLink 1 (`= 1`), options 1024 | 115 | MTF-01 |
| RC IN | — | S.Bus | — | MK15 air unit |
| POWER1 | — | Analog | — | PM02 |
| MAIN OUT 1–4 | — | PWM/DShot | — | ESCs |
| USB | `SERIAL0` | MAVLink | — | Bench only |

Serial index-to-port mapping varies by board definition. Confirm against the ArduPilot hardware page for Pixhawk 6C before setting parameters.

### TELEM2 ↔ Pi cable (JST-GH 6-pin to Dupont)

| TELEM pin | Signal (FC side) | Wire to Pi |
|---|---|---|
| 1 | VCC 5 V | **Not connected** |
| 2 | TX (out) | Pin 10 (GPIO15 RXD) |
| 3 | RX (in) | Pin 8 (GPIO14 TXD) |
| 4 | CTS | Not connected (optional) |
| 5 | RTS | Not connected (optional) |
| 6 | GND | Pin 6 (GND) |

Pin order follows the Pixhawk connector standard `[VERIFY against Holybro pinout drawing]`.

## 7. Expected data rates

| Link | Payload | Estimate | Capacity | Margin |
|---|---|---|---|---|
| CSI-2 (each) | 1640×1232 RAW10 at 20 fps | ≈ 0.4 Gbit/s | 2 lanes, ample | OK |
| I²C (IMU) | 12–14 bytes at 225 Hz + addressing | ≈ 40 kbit/s | 400 kbit/s | ~10× |
| UART Pi → FC | ODOMETRY 30 Hz, setpoint 20 Hz, OBSTACLE_DISTANCE 10 Hz, misc | ≈ 15–18 kB/s | 92 kB/s at 921 600 | ~5× |
| UART FC → Pi | Streams per §4 of mavlink-integration | ≈ 10–20 kB/s | 92 kB/s | ~4× |
| UART FC ↔ MK15 | Standard GCS streams | ≈ 3–5 kB/s | 5.7 kB/s at 57 600 | Tight: keep stream rates low |
| Storage (bag) | 2 × 640×480 mono at 20 Hz raw + IMU + state | ≈ 12.5 MB/s | Card dependent | Needs A2-class card or NVMe; else record compressed or decimated |

## 8. Connectors and cables

| Cable | Ends | Length | Notes |
|---|---|---|---|
| CSI ×2 | 15-pin 1.0 mm ↔ 22-pin 0.5 mm FFC | ≤ 200 mm | Equal length for both; shielded type preferred; strain-relieved |
| IMU I²C | Pads/header on camera board ↔ Dupont | ≤ 200 mm | 4 wires (SDA, SCL, GND, and 3V3 if the IMU is not powered through the CSI connector `[VERIFY]`); twisted with ground |
| TELEM2 | JST-GH 6 ↔ Dupont 3 | ≤ 250 mm | Twisted TX/GND and RX/GND |
| Pi power | BEC ↔ Dupont 2×2 or soldered | ≤ 150 mm, ≥ 20 AWG | Two 5 V and two GND pins in parallel; 5 A through a single header pin is beyond its rating |
| HDMI | micro-HDMI ↔ converter | Short, flexible, ultra-thin type | Rigid HDMI cables transmit vibration and stress the Pi connector |
| MK15 power | XT30/JST from PDB | — | Check air unit voltage range label before connecting |
| All flight wiring | — | — | Locking connectors (JST-GH, XT30/XT60). Dupont connectors on the Pi are secured with a header clamp or hot-melt and inspected pre-flight |

## 9. Grounding

- Star ground at the PDB. BEC, 12 V regulator, PM02 and MK15 negative leads all return there.
- Signal ground accompanies every signal cable (UART, I²C).
- Avoid ground loops through HDMI: the HDMI shield connects Pi ground to converter ground; both already share PDB ground. Keep the HDMI cable short and route it alongside the supply leads to minimise loop area.
- High-current ESC leads are kept short and twisted, routed away from the compass, the FC and the CSI ribbons.

## 10. EMI considerations

| Source | Victim | Mitigation |
|---|---|---|
| Pi 5 SoC, HDMI, CSI ribbons | GNSS L1 reception | GNSS on a mast ≥ 10 cm above; copper/aluminium foil ground plane under the GNSS puck if `sats`/SNR drops when the Pi is on; shielded CSI cables |
| USB 3.0 devices | GNSS and 2.4 GHz RC link | No USB 3 devices in flight; storage on microSD or NVMe |
| Pi Wi-Fi / Bluetooth 2.4 GHz | MK15 2.4 GHz link `[VERIFY band]` | Disable Bluetooth; Wi-Fi off in flight or locked to 5 GHz |
| MK15 air unit transmitter | GNSS, compass | Antennas at the rear, pointing down, ≥ 15 cm from GNSS |
| ESC / motor currents | Compass | Compass on mast; `COMPASS_MOT` compensation; twisted power leads |
| BEC switching noise | Camera analog supply, I²C | BEC away from camera; LC filter or ≥ 470 µF low-ESR capacitor at the Pi 5 V pins |
| HDMI | 2.4 GHz link | Short cable, ferrite if link quality drops with HDMI active |

Bench test: record GNSS satellite count and C/N0 with (a) Pi off, (b) Pi on idle, (c) Pi full stack with HDMI. A drop of more than 3 dB or 3 satellites requires shielding rework before flight.

## 11. Vibration considerations

| Element | Requirement | Method |
|---|---|---|
| FC IMU | Clip-free accelerometer data; ArduPilot `VIBE` < 30 m/s² (ideally < 15) | Pixhawk 6C internal isolation plus balanced props |
| Camera + ICM-20948 | No visible rolling-shutter "jello"; IMU not saturating; gyro noise in hover within 3× bench value | Camera and Pi on a common plate with 4 silicone dampers (≈ 30A–50A Shore); stiff stereo bar |
| Stereo geometry | L/R relative pose stable to < 0.1° | Both lenses on one PCB (as supplied); no flex in the mount |
| Camera–IMU extrinsics | Constant | IMU is on the camera PCB; do not relocate |
| FC ↔ camera relative pose | Moves slightly on dampers | Acceptable: the EKF treats VIO as a position/velocity source; lever arm error of millimetres is negligible |
| Exposure | Motion blur < 1 px | Exposure ≤ 2–4 ms outdoors; fixed exposure with gain, not auto-exposure hunting |
| Connectors | No intermittent contact | Locking connectors; ribbon strain relief; pre-flight tug test |

## 12. Thermal considerations

| Item | Limit | Design |
|---|---|---|
| Pi 5 SoC | Firmware throttles from about 80 °C, harder at 85 °C `[VERIFY]` | Active cooler mandatory; target ≤ 75 °C; inlet not blocked by the mounting plate |
| Pi in prop-wash | — | Helps in flight; worst case is hover in still air or ground idle before take-off. Test with props off and a full CPU load for 20 min. |
| BEC | Derates with temperature | Choose ≥ 5 A continuous rating; mount in airflow |
| MK15 air unit | −10 to 50 °C `[VENDOR]` | Built-in fan; do not enclose |
| LiPo | Do not fly below ~10 °C or above ~50 °C cell temperature | Standard practice |
| Camera sensors | Dark noise rises with temperature | No action |

The safety supervisor reads SoC temperature and throttle flags (`vcgencmd get_throttled` equivalent via sysfs) and sheds AI and HUD load first. See [safety-architecture](../12-safety/safety-architecture.md).

## 13. Items to verify at hardware bring-up

| # | Item |
|---|---|
| LL-1 | ICM-20948 I²C address, supply source, and whether INT is brought out on the Waveshare board |
| LL-2 | CSI cable type supplied with the camera and whether both 22-pin cables are included |
| LL-3 | Pixhawk 6C TELEM pin order and serial index mapping in ArduPilot 4.7 |
| LL-4 | MK15 air unit input voltage label (4S support) and connector type |
| LL-5 | HDMI converter input connector, supply voltage and whether it is powered from the air unit cable |
| LL-6 | Pi 5 UART overlay name and device node under Ubuntu 24.04 |
| LL-7 | MK15 operating band and whether Pi Wi-Fi must be disabled in flight |
| LL-8 | S.Bus frame rate from the MK15 air unit |
