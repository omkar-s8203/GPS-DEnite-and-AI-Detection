# ADR-017 — Ground App: Native Android on the MK15, over the IP Link

| Field | Value |
|---|---|
| Date | 2026-10-06 |
| Status | **Accepted** (design baseline DB-3.0) |
| Amends | Video topology in [siyi-mk15.md](../03-hardware/siyi-mk15.md) (A → B); retires `hud_node` |

## Context

The project owner requires an app on the SIYI MK15 from which the operator tracks and follows objects and starts missions such as a grid search, and has decided that it shall be a **native Android app** designed around the MK15's architecture. (A web app served by the Raspberry Pi was proposed and declined.)

Research into the MK15 established:

- The telemetry datalink can be presented to Android as UART, USB COM, Bluetooth or UDP (chosen in the SIYI TX app). The built-in UART path needs SIYI's SDK for third-party ground stations.
- QGroundControl's MAVLink forwarding to another endpoint is **one-way**: a second app cannot send commands through it.
- The datalink is a 57 600 baud serial stream.
- The air unit's Ethernet port carries IP traffic from devices on the aircraft to the ground unit; SIYI uses it for cameras and documents third-party IP devices on that network.

## Options

| # | Option |
|---|---|
| A | Native app as the **only** ground station: takes the MAVLink datalink, replaces QGroundControl, receives video by RTSP from the HDMI converter |
| B | Native app sharing the MAVLink datalink with QGroundControl through a MAVLink router built into the app |
| C | Native app talking to the **Raspberry Pi over the IP link**; QGroundControl keeps the MAVLink datalink |
| D | Native app using both: MAVLink for commands, IP for video and data |

## Evaluation

| Criterion | A | B | C | D |
|---|---|---|---|---|
| QGroundControl still usable in flight | No | Yes, if the router works | **Yes, untouched** | Depends |
| Work to build | Very high: a full ground station | High: router + custom MAVLink commands | **Medium**: one protocol to the Pi | High: two protocols |
| Rich data (thumbnails, map tiles, detection boxes per frame) | Poor over 57 600 baud | Poor | **Good** | Good |
| Custom MAVLink messages or commands needed | Yes | Yes | **No** | Yes |
| Tap-on-video accuracy | Tap refers to an encoded HUD picture | Same | **Tap refers to a frame id from the Pi** | Either |
| Video encoding load on the Pi | None (HDMI converter) | None | JPEG encoding, a few ms per frame | Either |
| Mass and power | Unchanged | Unchanged | **−55 g, −3 W** (converter removed) | Unchanged |
| Risk | Safety-relevant GCS functions re-implemented by students | Router bugs affect telemetry | Depends on the IP path working as documented | Most complex |

## Decision

**Option C.** A native Kotlin app on the MK15 connects to an `app_gateway` node on the Raspberry Pi through the MK15's IP bridge (Pi Ethernet ↔ air unit Ethernet). Control uses a JSON protocol over WebSocket; video is a stream of JPEG frames; map tiles come from the drone's own map pack. QGroundControl continues to use the serial MAVLink datalink unchanged. The HDMI converter is removed and the `hud_node` is retired; the app draws the overlay.

Design: [ground-app.md](../10-communication/ground-app.md).

## Reason

1. It follows the MK15's actual architecture: three independent paths (RC, serial datalink, IP), each used for what it suits.
2. It keeps the standard ground station and all its safety-relevant functions exactly as they are. The new app adds capability without replacing anything flight-critical.
3. It needs no MAVLink customisation and no SIYI SDK.
4. The IP path has the bandwidth for video, per-frame detections and finding thumbnails.
5. Removing the HDMI converter brings the mass and power budgets back inside their limits.
6. The app cannot arm, change mode or bypass the pilot, by construction: the protocol has no such messages, and the Pi applies the same gates to app requests as to any mission.

## Consequences

- A team member must own Android development. The app can be developed against a mock gateway from the first week.
- The Pi encodes video in software (JPEG). Estimated cost is small; to be measured.
- Dependence on the IP path through the MK15, which must be verified on the owned unit before development is committed (open point APP-1). If it does not work as expected, the fallback is option D restricted to essentials: commands as MAVLink `COMMAND_LONG` user commands through an in-app router, video by RTSP from the HDMI converter.
- A custom cable from the air unit's 8-pin connector to the Pi's Ethernet port.
- The reference imagery is displayed on the ground unit too; its licence must allow that.
- One more piece of software to test; the app is advisory class and its failure does not affect flight.
- `hud_node` and the HDMI path are removed from the baseline; the converter stays in the kit as a spare way to get video.
