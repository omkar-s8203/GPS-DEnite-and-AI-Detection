# ADR-014 — IMU Source for VIO

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted — conditional on gate G2b** (applies to low-regime stereo VIO only since DB-2.0) |

## Context

Visual-inertial odometry needs an IMU that is (1) rigidly attached to the cameras, (2) timestamped on the same clock as the images, (3) sampled at ≥ 200 Hz, and (4) of adequate noise quality. Two IMUs exist in the system: the ICM-20948 on the camera board and the IMUs inside the Pixhawk.

## Options

| # | Option |
|---|---|
| A | Camera-board ICM-20948, read by the Pi over I²C |
| B | Pixhawk IMU, streamed to the Pi over MAVLink (`RAW_IMU` / `SCALED_IMU` / `HIGHRES_IMU`) |
| C | Add a separate, better IMU on the camera mount (SPI, with data-ready interrupt) |
| D | No IMU in the visual estimator: stereo visual odometry only, inertial fusion in EKF3 |

## Evaluation

| Criterion | A: ICM-20948 on camera board | B: FC IMU via MAVLink | C: Added IMU | D: None |
|---|---|---|---|---|
| Rigid to cameras | **Yes** (same PCB) | **No** (FC and camera on separate vibration mounts) | Yes | n/a |
| Same clock as images | **Yes** (Pi monotonic) | No (FC clock; needs time sync; serial transport jitter) | Yes | n/a |
| Rate | ≈ 225 Hz | Limited by serial bandwidth and FC scheduling; high rates load the link | High | n/a |
| Timestamp jitter | I²C polling ± 1–2 ms `[ESTIMATE]`; better with interrupt | Transport and scheduling jitter of several ms `[ESTIMATE]` | Low | n/a |
| Noise quality | Consumer, older generation | Better | Better | n/a |
| Extra hardware | None | None | Yes | None |
| Extrinsic calibration stability | Fixed | Changes as dampers flex | Fixed | n/a |

## Decision

**Option A** for the primary estimator. **Option D** is the designated backup (it is the RTAB-Map stereo-odometry configuration of [ADR-004](ADR-004-vio-solution.md)).

Driver settings: gyro ±500 °/s, accelerometer ±8 g, ≈ 225 Hz output, burst reads so that gyro and accelerometer share a timestamp, interrupt-driven if the board exposes the INT pin. The magnetometer is not used.

## Reason

For VIO, a rigid camera–IMU transform and a common clock matter more than noise density. The camera-board IMU has both properties by construction; the flight-controller IMU has neither, because it sits on its own isolation and its samples arrive through a serial protocol with a different time base. An IMU that moves relative to the camera invalidates the extrinsic calibration the estimator depends on.

## Consequences

- An I²C driver node must be written and its timing characterised (rate, gaps, jitter).
- IMU noise parameters come from an Allan-variance run and are inflated for flight vibration.
- The camera/IMU assembly needs its own vibration damping; otherwise motor vibration aliases into a 225 Hz stream. A vibration test on the vehicle is part of gate G2.
- If gate G2 identifies IMU timing or noise as the limiting factor, the order of remedies is: interrupt-driven sampling → option D → option C (or a camera with an integrated, hardware-synchronised IMU per [ADR-011](ADR-011-stereo-camera-suitability.md)).
- The FC IMU is still used by `vio_monitor` as an independent cross-check of VIO attitude and rotation rate.
