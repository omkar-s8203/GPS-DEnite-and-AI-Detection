# ADR-001 — ROS 2 Distribution and Operating System

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

ROS 2 is mandatory. The companion is a Raspberry Pi 5 (arm64). The choice of distribution fixes the OS, the Gazebo version, and which third-party packages are available as binaries. At the time of writing (October 2026) two LTS distributions are current: Jazzy Jalisco (May 2024, supported to May 2029, Tier 1 on Ubuntu 24.04) and Lyrical Luth (May 2026, supported to 2031, Tier 1 on Ubuntu 26.04). The Pi 5 is not supported by Ubuntu 22.04, which rules out Humble on a native install.

A complication: the Pi 5 camera pipeline is supported by Raspberry Pi's fork of libcamera. Ubuntu's packaged libcamera has lacked the Pi 5 pipeline handler, so on Ubuntu it is built from source.

## Options

| # | Option |
|---|---|
| A | Ubuntu Server 24.04 + ROS 2 Jazzy (binaries) |
| B | Ubuntu 26.04 + ROS 2 Lyrical (binaries) |
| C | Raspberry Pi OS (Bookworm/Trixie) + ROS 2 built from source or in Docker |
| D | Ubuntu 22.04 + Humble in Docker on a newer host |

## Evaluation

| Criterion | A: 24.04 + Jazzy | B: 26.04 + Lyrical | C: Pi OS + source/Docker | D: Humble in Docker |
|---|---|---|---|---|
| ROS 2 support tier on arm64 | Tier 1 | Tier 1 | Tier 3 (source) | Tier 1 inside container |
| Maturity (time in the field) | > 2 years | ≈ 5 months | — | Mature but ageing (EOL 2027) |
| MAVROS binaries | Yes (2.14.x seen) | Not verified | Build from source | Yes |
| RTAB-Map binaries | Yes | Not verified | Build | Yes |
| OpenVINS | Jazzy/24.04 fixes exist | Not verified | Build | Supported |
| Gazebo pairing | Harmonic (LTS) — supported by `ardupilot_gazebo` | Newer Gazebo; ArduPilot plugin support to be verified | — | Fortress/Harmonic |
| Camera stack | Build Raspberry Pi libcamera fork (documented by several community guides) | Possibly improved packaging; not verified | **Native, works out of the box** | Hard: device access from container |
| Build time on the Pi | Low (binaries) | Low | High (hours) | Low |
| Effort for a student team | Low–medium | Unknown risk | High | Medium–high |

## Decision

**Option A: Ubuntu Server 24.04 LTS (arm64) with ROS 2 Jazzy Jalisco**, using the default Fast DDS middleware. Gazebo Harmonic on the workstation.

## Reason

1. Jazzy has more than two years of ecosystem maturity; every third-party package this design needs is known to exist for it.
2. It is Tier 1 on the Pi 5's architecture with binary packages, so the Pi does not spend hours compiling ROS.
3. Gazebo Harmonic pairs with Jazzy and is supported by the ArduPilot Gazebo plugin.
4. Support to May 2029 covers the project and its successors.
5. Lyrical is the newer LTS, but five months after release the availability of MAVROS, OpenVINS fixes and ArduPilot simulation tooling could not be confirmed. A college project should not be the integration tester for a new distribution.

## Consequences

- The Raspberry Pi fork of libcamera must be built from source on the Pi and kept in the dependency workspace. This is the main cost of the decision. Documented in [ADR-007](ADR-007-camera-interface.md).
- If the libcamera build proves unworkable, the fallback is **Option C variant**: Raspberry Pi OS with ROS 2 Jazzy in a Docker container, capturing images natively on the host and passing them into the container over a shared-memory or local socket bridge. This fallback is recorded, not planned.
- Migration to Lyrical is a future task: project code avoids deprecated APIs to keep it cheap.
- Workstation must run Ubuntu 24.04 (dual boot or WSL 2); see [simulation-strategy.md](../11-simulation/simulation-strategy.md) §6.

## Review trigger

Revisit if, at the start of implementation, MAVROS, OpenVINS and `ardupilot_gazebo` are all confirmed working on Lyrical/26.04 **and** 26.04 ships a libcamera that supports the Pi 5 pipeline natively.
