# Downward Camera

| Field | Value |
|---|---|
| Document ID | GDN-HW-011 |
| Version | 1.0 (added with DB-2.0) |
| Date | 2026-10-05 |
| Status | Requirement defined; model not yet selected (OD-12) |
| Decision | [ADR-016](../17-decisions/ADR-016-reference-imagery-and-downward-camera.md) |

## 1. Purpose

Supplies ground images for satellite map matching (≈ 1 Hz) and ground visual odometry (15 Hz). See [visual-geolocalization.md](../09-navigation/visual-geolocalization.md).

## 2. Why a third camera

| Fact | Consequence |
|---|---|
| The Raspberry Pi 5 has two CSI ports | Both are used by the stereo pair |
| The stereo pair faces forward for obstacle sensing and AI | It cannot also look down |
| Map matching needs a nadir view with a wide footprint | A separate downward camera on USB |

## 3. Requirements

| Property | Requirement |
|---|---|
| Interface | USB 2.0, UVC (standard Linux `uvcvideo` driver) |
| Sensor | ≈ 1 MP; global shutter preferred; monochrome or colour |
| Output used | 640×480 at 15 Hz (MJPEG or YUYV) |
| Lens | M12, 90–120° horizontal field of view, fixed focus at infinity, low distortion preferred |
| Exposure | Manual exposure and gain over UVC controls |
| Mass | ≤ 30 g with lens and cable |
| Power | ≤ 1.5 W from the Pi USB port |
| Cable | ≤ 200 mm, shielded |

### DB-3.0 change: higher resolution for search

Grid search and follow run the object detector on this camera ([search-track-follow.md](../09-navigation/search-track-follow.md)). To put enough pixels on a person from 25–30 m, the requirement rises:

| Property | DB-2.0 | DB-3.0 |
|---|---|---|
| Delivered resolution | 640×480 | **≥ 1920×1080** (MJPEG over USB 2.0) for detection; down-scaled to 640×480 for map matching and odometry |
| Sensor | ≈ 1 MP | ≥ 2 MP |
| Frame rate | ≥ 15 Hz | ≥ 15 Hz at full resolution preferred; ≥ 10 Hz acceptable |
| Shutter | Global preferred | Global-shutter sensors at 2 MP over UVC are less common; a rolling-shutter sensor with short exposure is acceptable (§7) |

Ground resolution at 1920 px width, 100° lens: 3.1 cm/px at 25 m, 3.7 cm/px at 30 m. A standing person seen from above is then about 13–16 px across; a lying person about 46–55 px long.

USB 2.0 carries 1080p MJPEG at 15–30 fps on typical UVC cameras `[VERIFY for the chosen model]`. Decoding 1080p MJPEG on the Pi costs CPU; measure it.

## 4. Mounting

| Item | Requirement |
|---|---|
| Location | Underside, near the vehicle centre, clear of landing gear and of the optical-flow sensor's view |
| Orientation | Optical axis along body down; image top towards the nose (documented in the URDF as `down_camera_optical_frame`) |
| Isolation | On the same damped plate as the Pi, or its own soft mount |
| Boresight | Measured: fly or hold level over a marked point and record the pixel offset; stored as a mounting rotation |
| Protection | Lens recessed or with a hood: no direct sun or prop shadow flicker in view |

## 5. Footprint

`W = 2·h·tan(HFOV/2)`; native ground resolution at 640 px = `W / 640`.

| Height | Footprint, 100° lens | Native resolution | Footprint, 120° lens |
|---|---|---|---|
| 30 m | 72 m × 54 m | 0.11 m/px | 104 m × 78 m |
| 50 m | 119 m × 89 m | 0.19 m/px | 173 m × 130 m |
| 60 m | 143 m × 107 m | 0.22 m/px | 208 m × 156 m |

The camera always out-resolves the satellite reference (0.3–0.5 m/px), so the live image is scaled **down** to the map resolution. A 640×480 stream is sufficient; higher resolution adds cost without benefit.

## 6. Calibration

| Calibration | Method |
|---|---|
| Intrinsics and distortion | Checkerboard or AprilGrid with OpenCV / Kalibr (wide lenses may need the fisheye/equidistant model) |
| Mounting rotation (boresight) | §4 |
| Position relative to the FC | Ruler; entered in the URDF |
| Time offset | Not calibrated separately; attitude is interpolated to the image timestamp |

## 7. Exposure and blur

Motion blur at the ground: `blur_px = v·t_exp / GSD_native`. At 3 m/s, 2 ms exposure and 0.19 m/px: 0.03 px. Negligible. Rotation matters more: at 20 °/s and 2 ms, a 640 px, 100° image moves 0.2 px. A rolling-shutter sensor is therefore acceptable if exposure stays short, although global shutter removes the question.

## 8. Electrical and EMI

- USB 2.0 only. Do not use a USB 3 port/cable pair for this camera: USB 3 signalling raises the noise floor around GPS L1 and 2.4 GHz.
- Repeat the bench GPS reception test (T7-06) with the camera streaming.
- Current comes from the Pi's USB port; include it in the 5 V budget.

## 9. ROS 2 integration

Standard UVC driver node (`usb_cam` or `v4l2_camera`) configured in `gdn_geoloc`:

| Topic | Type | Rate |
|---|---|---|
| `/down/image_raw` | `sensor_msgs/Image` (mono8, 640×480) | 15 Hz |
| `/down/camera_info` | `sensor_msgs/CameraInfo` | 15 Hz |

Frame: `down_camera_optical_frame`.

## 10. Open items

| # | Item |
|---|---|
| DC-1 | Select a model available in India; confirm mass, power, lens options and price |
| DC-2 | Confirm manual exposure works through the UVC controls on Ubuntu 24.04 |
| DC-3 | Measure timestamp jitter of the UVC stream |
| DC-4 | Decide the lens: 100° (less distortion) or 120° (larger footprint) |
