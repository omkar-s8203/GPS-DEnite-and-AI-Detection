# ADR-011 — Stereo Camera Suitability and Upgrade Path

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted — conditional on gate G2b** (renamed in DB-2.0). Since [ADR-015](ADR-015-visual-geolocalization.md) the stereo camera serves depth, AI and low-regime odometry only, so a failed gate reduces the low regime's capability but no longer blocks the project |

## Context

The project owns a Waveshare IMX219-83 stereo camera. Research for this baseline established:

- **Vendor statement:** the camera "doesn't feature hardware synchronization" between its two sensors.
- The IMX219 is a **rolling-shutter** sensor.
- The onboard ICM-20948 is read over I²C with no hardware trigger relating it to image capture.
- Visual-inertial estimators, including the selected OpenVINS, are designed around synchronised, preferably global-shutter cameras; community guidance for OpenVINS explicitly lists global shutter, short exposure and fixed rate as requirements.

The brief asks that unsuitable hardware be identified and a better alternative proposed. The camera is suitable for stereo depth and AI. For VIO it is marginal: usable only if software synchronisation proves tight and motion is kept slow.

## Options

| # | Option |
|---|---|
| A | Proceed with the Waveshare camera unconditionally |
| B | Replace it now with a global-shutter, hardware-synchronised stereo camera with IMU |
| C | Keep it as the baseline, measure against explicit criteria early, and upgrade only if it fails |
| D | Keep it for depth/AI and add a separate sensor for localisation only (for example optical flow as the sole GPS-denied source) |

## Evaluation

| Criterion | A | B | C | D |
|---|---|---|---|---|
| Cost now | None | ≈ ₹18,000–35,000 `[VERIFY]` | None | Small |
| Risk of late failure | **High** | Low | Low (decided by week ≈ 13) | Low |
| Uses owned hardware | Yes | No | Yes, if it passes | Yes |
| Academic value | — | — | **The characterisation of a low-cost unsynchronised rolling-shutter rig for VIO is itself a result** | Lower (no VIO) |
| Schedule impact | None until failure | Procurement delay now | None unless it fails | None |

## Decision

**Option C.**

1. Use the Waveshare IMX219-83 as the baseline sensor.
2. Apply every available mitigation: libcamera software synchronisation with per-pair skew measurement; short fixed exposure; binned mode; inflated vision noise in the estimator; online time-offset calibration; damped rigid mount; speed ≤ 2 m/s and yaw rate ≤ 45 °/s.
3. Evaluate at **gate G2** against numeric criteria:

| Criterion | Threshold |
|---|---|
| L/R timestamp skew | ≤ 1 ms for ≥ 99 % of pairs |
| Stereo reprojection error | ≤ 0.5 px RMS |
| Depth error | ≤ 5 % at 2 m; ≤ 10 % at 5 m |
| Handheld VIO loop (≈ 30 m) | ≤ 2 % end-point error in 4 of 5 runs |
| VIO on a vibrating frame | No divergence in 3 min; stationary drift ≤ 0.3 m |

4. If G2 fails on VIO but passes on sync: switch to the backup estimator (loosely coupled stereo odometry) with a reduced envelope and re-test.
5. If G2 fails on sync, or the backup also fails: **upgrade the camera.** Recommended: Luxonis OAK-D Lite (global-shutter synchronised mono stereo pair, 75 mm baseline, IMU per vendor documentation, on-device depth and neural inference, ROS 2 driver). Alternatives: Intel RealSense D435i; two Raspberry Pi Global Shutter cameras with external trigger on a custom rigid bar.
6. The Waveshare camera then remains useful as a colour camera for the detector.

## Reason

- Replacing a ₹5,000 camera with a ₹20,000–35,000 one before measuring anything would be poor engineering and poor use of a college budget.
- Proceeding without a gate would risk discovering the problem during flight testing, the most expensive place to find it.
- A measured answer to "how well does VIO work on a cheap unsynchronised rolling-shutter stereo module, and where does it break?" is a defensible contribution whichever way it turns out.
- The software architecture is unaffected by the outcome because the camera is behind a topic contract ([ADR-007](ADR-007-camera-interface.md)).

## Consequences

- Phases P04–P07 are front-loaded so that G2 falls around week 13.
- The BOM lists the upgrade as optional item O1; budget should be held in reserve until G2.
- All performance targets for VIO are stated for the baseline camera with explicit envelope limits; they are re-baselined if the camera changes.
- A USB camera upgrade introduces USB noise near GNSS and must be laid out accordingly.
- The presence of an IMU on a specific OAK-D Lite unit, and its Indian price, must be verified before purchase.
- The project report must state the camera's limitations plainly ([stereo-camera.md](../03-hardware/stereo-camera.md) §10).
