# Raspberry Pi 5 (8 GB) — Companion Computer Analysis

| Field | Value |
|---|---|
| Document ID | GDN-HW-003 |
| Version | 1.0 |
| Date | 2026-10-05 |

## 1. Specification

| Item | Value | Source |
|---|---|---|
| SoC | Broadcom BCM2712, quad-core 64-bit Arm Cortex-A76 at 2.4 GHz, 512 KB L2 per core, 2 MB shared L3 | `[VENDOR]` |
| GPU | VideoCore VII, OpenGL ES 3.1, Vulkan 1.3 | `[VENDOR]` |
| RAM | 8 GB LPDDR4X-4267 | `[VENDOR]` |
| Camera/display | 2 × 4-lane MIPI transceivers (22-pin, 0.5 mm) | `[VENDOR]` |
| USB | 2 × USB 3.0 (5 Gbit/s), 2 × USB 2.0 | `[VENDOR]` |
| Network | Gigabit Ethernet; dual-band 802.11ac; Bluetooth 5.0 / BLE | `[VENDOR]` |
| Expansion | PCIe 2.0 ×1 (FFC); 40-pin GPIO header | `[VENDOR]` |
| Power | 5 V / 5 A via USB-C with Power Delivery | `[VENDOR]` |
| Other | Real-time clock (external battery), power button, fan header, 2 × micro-HDMI | `[VENDOR]` |
| Power draw | ≈ 3 W idle; ≈ 7–9 W four-core CPU load; ≈ 11–12 W sustained worst case; higher transients reported | Third-party measurements; `[MEASURE]` on our build |
| Mass | ≈ 46 g board only | `[VERIFY]`, `[MEASURE]` |

## 2. Suitability by subsystem

### 2.1 CPU

Four A76 cores are the entire compute budget. There is no usable GPU compute path for neural networks or stereo matching in mainstream frameworks, so every algorithm is CPU-bound.

Estimated steady-state CPU budget (`[ESTIMATE]`, 400 % = four cores):

| Workload | Configuration | Estimate |
|---|---|---|
| Stereo capture + ISP | 2 × 640×480 at 20 Hz | 30–40 % |
| IMU driver | 225 Hz I²C | 2–4 % |
| VIO (OpenVINS) | Stereo, 150–200 features, 20 Hz | 80–120 % |
| Rectification + disparity | 640×480 block matching at 10 Hz | 50–70 % |
| Object detector | YOLO26n, NCNN, 320 px, 5 Hz, 2 threads | 30–50 % |
| MAVROS | Reduced plugin set | 10–15 % |
| Localisation, nav-mode, navigator, safety, telemetry | — | 10–15 % |
| HUD rendering to HDMI | 720p, 15 Hz | 10–20 % |
| rosbag2 recording | MCAP, images raw | 10–20 % |
| **Total** | | **≈ 230–350 %** |

Target is ≤ 300 % average (NFR-013). The upper estimate exceeds it, so the design includes load shedding in this order: HUD rate → bag image decimation → detector rate → depth rate. VIO and MAVROS are never shed.

Core assignment (planned): VIO pinned to cores 2–3; camera driver and MAVROS on core 0–1; detector limited to 2 threads on cores 0–1 at lower priority. `isolcpus` is not used initially.

### 2.2 GPU

Used only for display composition (HUD). Vulkan compute through NCNN is possible in principle on VideoCore VII but is not relied on: reported gains for small CNNs are inconsistent and driver maturity is uncertain. Treated as an optional experiment in the optimisation phase.

### 2.3 RAM

8 GB is ample. Expected resident use is 2–3 GB `[ESTIMATE]`. The margin lets rosbag2 buffer in RAM (cache size raised) to ride out slow storage writes. Swap is disabled to avoid latency spikes.

### 2.4 Camera interfaces

Two 4-lane CSI-2 ports accept both IMX219 sensors directly, which is why no multiplexer or USB bridge is needed. Points to note:

- Pi 5 uses 22-pin 0.5 mm connectors; the camera board uses 15-pin 1.0 mm. Adapter cables are required.
- The Pi 5 camera pipeline (PiSP front end and back end) is supported by the **Raspberry Pi fork of libcamera**. Upstream libcamera as packaged for Ubuntu 24.04 has lacked the Pi 5 pipeline handler, so the fork is built from source. See [ADR-007](../17-decisions/ADR-007-camera-interface.md).
- Frame timestamps come from the CSI-2 frame-start event on the kernel monotonic clock, which is what makes software stereo pairing measurable.

### 2.5 USB

Not used in flight. USB 3.0 radiates broadband noise around 2.4 GHz that is known to degrade GNSS reception and 2.4 GHz links. A USB camera upgrade (see [stereo-camera](stereo-camera.md) §9) would need a shielded cable, USB 2.0 mode if sufficient, and distance from the GNSS antenna.

### 2.6 Networking

| Interface | Use |
|---|---|
| Wi-Fi 5 GHz | Bench: SSH, RViz2 on the laptop, log download. Off in flight by default. |
| Ethernet | Bench; optional direct connection to the MK15 air unit instead of the HDMI converter |
| Bluetooth | Disabled |

ROS 2 discovery is restricted to the local host in flight (`ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST`).

### 2.7 GPIO

3.3 V logic, not 5 V tolerant. On the Pi 5 the header is driven by the RP1 controller; older GPIO libraries that touch SoC registers directly do not work. The design needs only kernel-supported functions (PL011 UART, I²C), accessed through `/dev/ttyAMA0` and `/dev/i2c-1`.

### 2.8 Power

- The Pi 5 expects 5.1 V at up to 5 A. A supply that cannot deliver 5 A makes the firmware limit USB current; this is harmless here because USB is unused.
- Powering through the GPIO 5 V pins bypasses USB-C PD negotiation and the input protection. The BEC must therefore be clean and correctly set **before** it is connected. Set `usb_max_current_enable=1` and the EEPROM `PSU_MAX_CURRENT=5000` so the firmware does not assume a weak supply `[VERIFY]`.
- An alternative is a USB-C PD trigger BEC. It adds a connector that can shake loose; GPIO feed with a clamped header or soldered leads is preferred.
- Under-voltage events (below about 4.8 V) are logged by the firmware and are a pre-flight NO-GO if they appear during the bench load test.

### 2.9 Thermal management

The Pi 5 throttles without active cooling under sustained four-core load. The official Active Cooler (heatsink + PWM fan) is mandatory. Requirement: no throttling flag after 20 minutes of full-stack operation at the worst-case ambient temperature expected at the test site (NFR-015).

### 2.10 Storage

| Option | For | Against |
|---|---|---|
| microSD (A2, U3) | Light, no extra hardware | Sustained write speed varies; wear; raw image logging may drop messages |
| NVMe SSD on M.2 HAT | Fast, reliable logging | ≈ 30–40 g, cost, occupies the PCIe port |
| USB 3 SSD | Fast | RF noise; rejected for flight |

Baseline: A2 microSD with compressed or decimated image logging. NVMe is the recommended optional upgrade for dataset-collection flights (see BOM).

## 3. Real-time behaviour

Standard Ubuntu kernel (PREEMPT_DYNAMIC). No hard real-time requirement exists on the Pi because the FC closes all fast loops. The tightest companion deadline is the 20–30 Hz external-nav stream; jitter of a few milliseconds is acceptable. A PREEMPT_RT kernel is not adopted in the baseline; it can be revisited if IMU timestamp jitter proves to be a problem.

## 4. AI suitability

| Fact | Consequence |
|---|---|
| Vendor benchmark: YOLO26n at 640 px, NCNN, ≈ 67 ms per image on Pi 5; ONNX ≈ 126 ms; PyTorch ≈ 299 ms (Ultralytics, version 8.4.x) | NCNN is the runtime. 640 px at full rate would consume the whole CPU; the design uses 320 px at 5 Hz. |
| No NPU | Small models only: "nano" class detectors |
| PCIe port available | A Hailo-based AI HAT can be added later; driver support on Ubuntu must be proven first |

Conclusion: the Pi 5 can run a nano-class detector at a few hertz alongside VIO. It cannot run medium or large models in real time, and it cannot run learned depth or learned VIO.

## 5. Verdict

**Suitable** as the companion computer for this project's envelope, with three conditions:

1. Active cooling and a dedicated 5 A supply.
2. Disciplined CPU budget and load shedding.
3. Acceptance that perception is "nano model at ≈ 5 Hz", not high-rate dense AI.

Alternatives considered:

| Platform | Why not chosen |
|---|---|
| NVIDIA Jetson Orin Nano | Better for AI (GPU) and has mature VIO options, but several times the cost; the Pi 5 is already owned |
| Raspberry Pi 4 | About half the CPU performance; VIO + depth + AI together is not feasible |
| Raspberry Pi CM5 + carrier | Same compute, lighter; needs a carrier board; no benefit at prototype stage |
