# ADR-007 — Camera Interface

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

The Waveshare IMX219-83 has two independent CSI-2 sensors with no hardware synchronisation (vendor-stated). The Raspberry Pi 5 has two CSI-2 ports. Stereo depth and stereo VIO need left/right frames exposed at the same time and stamped accurately. On Ubuntu 24.04, the packaged libcamera has lacked the Pi 5 pipeline handler.

## Options

| # | Option |
|---|---|
| A | Two instances of the community `camera_ros` node |
| B | `v4l2_camera` |
| C | GStreamer `libcamerasrc` pipelines bridged to ROS |
| D | Custom single-process stereo driver on libcamera (Raspberry Pi fork) with software synchronisation |
| E | Replace the camera with a USB device that synchronises internally |

## Evaluation

| Criterion | A | B | C | D | E |
|---|---|---|---|---|---|
| Works with Pi 5 ISP pipeline | Yes (with RPi libcamera) | No for raw Bayer sensors | Yes | Yes | n/a |
| Control over L/R synchronisation | None across processes | None | Limited | **Full** (libcamera software sync, server/client) | Built in |
| Timestamp from sensor frame start | Yes | — | Uncertain through the bridge | **Yes, explicit** | Device-dependent |
| Identical stamps per pair + skew published | No | No | No | **Yes** | Yes |
| Zero-copy to rectification/depth | No | — | No | Yes (component) | Driver-dependent |
| Effort | Lowest | — | Medium | Highest | Purchase + integration |

## Decision

**Option D:** a single C++ component (`gdn_camera/stereo_camera`) that opens both sensors through the Raspberry Pi fork of libcamera, enables libcamera's software camera synchronisation, pairs frames by sensor timestamp, and publishes left/right with the same header stamp together with the measured skew.

**Option A** is used only as a bring-up shortcut to obtain first images.

The topic contract is fixed ([package-structure.md](../05-ros2/package-structure.md) §5) so that **option E** can replace the driver without downstream changes if [ADR-011](ADR-011-stereo-camera-suitability.md) triggers a camera upgrade.

## Reason

The analysis in [stereo-camera.md](../03-hardware/stereo-camera.md) §5 shows that inter-camera timing error translates directly into disparity error during rotation (about 2 px per 5 ms at 1 rad/s). Synchronisation and its measurement are therefore the most important properties of the driver, and only a single-process design controls them. Raspberry Pi documents software synchronisation as working with its camera modules and with third-party sensors whose drivers implement frame-duration control; the IMX219 driver is the same one used by the official Camera Module v2.

## Consequences

- libcamera (Raspberry Pi fork) is built from source on Ubuntu 24.04 and pinned.
- The achievable residual skew is unknown until measured; it is the first measurement of the camera phase and an input to gate G2.
- More driver code to write and test than with an off-the-shelf node.
- Rolling shutter is not addressed by this decision; see ADR-011.
- Sensor mode, output size and rate: 1640 × 1232 binned → 640 × 480 mono at 20 Hz, plus a 320 × 240 colour stream from the left camera for the detector.
