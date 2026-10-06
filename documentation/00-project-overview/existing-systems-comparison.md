# Comparison with Existing GPS-Denied UAV Systems

| Field | Value |
|---|---|
| Document ID | GDN-OVR-003 |
| Version | 1.0 |
| Date | 2026-10-05 |

## 1. Purpose and caution

This document places the proposed system among existing approaches. It makes no claim of superiority. Descriptions of commercial products are general, drawn from public information, and may be out of date; they are included to show the *class* of capability, not to benchmark against specific models.

## 2. Classes of existing systems

| Class | Typical examples | Sensors | How position is obtained | Typical capability |
|---|---|---|---|---|
| GPS-only hobby/college drones | Pixhawk or similar FC + GNSS + compass | GNSS, baro, IMU | GNSS in an EKF | Good outdoor hold and missions; on GNSS loss falls to altitude-hold or lands |
| Optical-flow systems | ArduPilot/PX4 with PMW3901-class flow + ToF/LiDAR; small indoor drones | Downward flow, range sensor, IMU | Flow velocity integrated by the FC | Stable indoor hover and slow flight over textured floors; no forward perception; position drifts |
| VIO systems (tracking-camera style) | FC + self-contained tracking camera (e.g. the discontinued Intel RealSense T265), or companion computers with VIO such as ModalAI VOXL-class boards | Global-shutter fisheye/stereo cameras with factory-synchronised IMU | VIO on dedicated hardware → FC external navigation | Robust GPS-denied position hold and waypoint flight; mature ArduPilot/PX4 integration |
| SLAM / mapping research platforms | University platforms using VINS-Fusion, ORB-SLAM3, LiDAR-inertial odometry | Synchronised global-shutter stereo or LiDAR, high-grade IMU, x86 or GPU computers | Optimisation-based VIO/LIO with loop closure and mapping | Agile flight, exploration, dense mapping, planning in clutter |
| Commercial autonomous drones | Skydio-class and enterprise DJI-class aircraft | Multiple wide-angle navigation cameras covering all directions, dedicated vision processors, sometimes LiDAR/radar | Proprietary multi-camera VIO + learned perception + 3D mapping | 360° obstacle avoidance, GPS-denied flight, autonomous tracking, operation near structures |

### DB-2.0 addition: map-matching ("scene matching") systems

| Class | Typical examples | Sensors | How position is obtained | Typical capability |
|---|---|---|---|---|
| Absolute visual localisation against satellite or aerial imagery | Research systems matching a downward camera to orthophotos or satellite maps; an old idea in guided-weapon and aircraft navigation; now appearing in commercial GNSS-free navigation modules | Downward or gimballed camera, IMU, barometer; usually a GPU or strong onboard computer | Image-to-map matching (correlation, features, learned descriptors, particle filters) fused with inertial or visual odometry | Bounded position error of metres to tens of metres over long flights, at heights of tens to hundreds of metres, over terrain with distinct features |

With DB-2.0 this is the class the project belongs to for its cruise regime. Relative to that class:

| Aspect | Typical published system | This project |
|---|---|---|
| Onboard computer | GPU-class or offline processing | Raspberry Pi 5 CPU |
| Matching | Increasingly learned, robust to season | Classical features first; lightweight learned matcher compared |
| Area | Kilometres to tens of kilometres | About 1 km² |
| Height | 100 m and above is common | 40–60 m |
| Evaluation | Datasets and long flights | Small site, simulated denial, GNSS as truth |
| Robustness to season / terrain | Studied explicitly | Limited to the test site and season; stated as a limitation |

In the capability matrix below, read the "This project" column with these changes: position hold and waypoint flight without GNSS become **bounded-error (metres) in the cruise regime**; "Mapping / loop closure" stays ○, but drift is now bounded by a prior map; the sensor-quality row about the stereo camera applies to the low regime only.

## 3. Capability matrix

Legend: ● yes / strong, ◐ partial / limited, ○ no.

| Capability | GPS-only | Optical flow | VIO (dedicated hardware) | Research SLAM | Commercial autonomous | **This project (target)** |
|---|---|---|---|---|---|---|
| Outdoor GNSS navigation | ● | ◐ | ● | ● | ● | ● |
| Position hold without GNSS | ○ | ● (over texture, low height) | ● | ● | ● | ◐ (slow, textured, daylight) |
| Waypoint flight without GNSS | ○ | ◐ | ● | ● | ● | ◐ (short, ≤ 2 m/s) |
| In-flight GNSS ↔ non-GNSS transition | ○ | ◐ (manual or scripted) | ● (available in ArduPilot/PX4) | ● | ● | ● (automatic, confidence-gated; the project's focus) |
| GNSS degradation detection before loss | ◐ (EKF innovation checks) | ◐ | ◐ | ● | ● | ● (explicit classifier + vision cross-check) |
| Independent fallback if vision fails | — | — | ◐ | ◐ | ● | ● (optical-flow tier) |
| Forward obstacle sensing | ○ | ○ | ◐ (if depth camera fitted) | ● | ● (all directions) | ◐ (forward 73°, 0.5–6 m) |
| Obstacle avoidance (path around) | ○ | ○ | ◐ | ● | ● | ○ (stop-and-hold only) |
| Object recognition | ○ | ○ | ◐ | ◐ | ● | ◐ (nano detector, 5 Hz) |
| Object range / position | ○ | ○ | ◐ | ● | ● | ◐ (to ≈ 6–8 m) |
| Mapping / loop closure | ○ | ○ | ◐ | ● | ● | ○ (off-line only) |
| Low-light / night | ● (GNSS unaffected) | ○ | ◐ | ◐–● (LiDAR) | ◐–● | ○ |
| High-speed / agile flight | ● | ○ | ◐ | ● | ● | ○ |
| Sensor quality for VIO | — | — | Global shutter, hardware sync | Global shutter, hardware sync | Custom | Rolling shutter, software sync (**weakest in class**) |
| Compute | MCU | MCU | Dedicated vision SoC | x86 / GPU | Custom SoC/GPU | Pi 5 CPU only |
| Approximate cost of the autonomy payload | — | Very low | Medium–high | High | Not sold separately | Low (owned Pi + ≈ ₹5,000 camera + ≈ ₹3,000–5,000 flow sensor) |
| Openness | Open | Open | Mixed | Open code, custom hardware | Closed | Fully open, documented |

## 4. Existing capabilities

What already exists and is **not** new in this project:

- External-navigation fusion and source switching in ArduPilot's EKF3.
- Open-source VIO (OpenVINS, VINS-Fusion and others).
- Depth-camera obstacle input to ArduPilot.
- Nano-class object detectors running on a Raspberry Pi.
- ROS 2 ↔ MAVLink bridging through MAVROS.

The project integrates these; it does not invent them.

## 5. Our capabilities (as designed)

- A complete, documented ROS 2 Jazzy architecture joining the above on a Raspberry Pi 5.
- A navigation-mode manager with explicit GNSS-health classification, continuous vision-to-FC frame alignment, a localisation-confidence metric built from independent cross-checks, and three-tier fallback.
- Semantic object reports with metric range from late fusion of a detector and stereo depth.
- A safety architecture in which the companion is never flight-critical.
- A measured evaluation of low-cost, unsynchronised, rolling-shutter stereo hardware for VIO.

## 6. Missing capabilities relative to the state of the art

| Missing | Why |
|---|---|
| All-direction obstacle sensing | One forward stereo pair |
| Planning around obstacles; exploration | Scope and compute |
| Loop closure / drift correction | Scope and compute ([ADR-005](../17-decisions/ADR-005-slam-solution.md)) |
| Robustness to fast motion | Rolling shutter, software sync |
| Night / low-texture operation | Passive visible-light cameras |
| Long GPS-denied endurance | Unbounded drift |
| Redundant flight-critical hardware | Prototype |
| Large or high-rate perception models | CPU only |
| Dynamic-object handling in VIO | Not addressed |
| Certification-grade safety | Research prototype |

## 7. Improvements this project offers within its niche

Stated relative to a **typical college or hobby build**, not relative to commercial or leading research systems:

| Improvement | Compared with |
|---|---|
| Retains position hold and slow navigation when GNSS is lost | GPS-only builds |
| Forward perception and object awareness; position not tied to floor texture alone | Optical-flow-only builds |
| Explicit, logged, testable transition logic with confidence gating and a third tier | Typical "plug in a tracking camera" integrations, where switching is manual or a short script |
| Works with a commodity Pi camera module rather than a discontinued or costly tracking camera | T265-style integrations |
| Complete open documentation, simulation and test plan | Most student projects |

Each of these is a design intent until supported by VALIDATED measurements.

## 8. Limitations to state alongside any comparison

1. Performance on the baseline camera is expected to be clearly below dedicated VIO hardware.
2. All GPS-denied results apply only inside the stated envelope (speed, yaw rate, light, texture, height, duration).
3. Obstacle handling is stop-only and forward-only.
4. The system is supervised at all times by a safety pilot.

## 9. Positioning statement

> A low-cost, open, ROS 2-based reference implementation of GNSS-to-vision navigation hand-over for a small multirotor, built from commodity parts, with an honest quantitative account of what such parts can and cannot do.
