# ADR-016 — Reference Imagery and Downward Camera

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted — imagery source and camera model to be confirmed** (open decisions OD-11, OD-12) |

## Context

Satellite image matching ([ADR-015](ADR-015-visual-geolocalization.md)) needs two things the project does not yet have: a reference image of the flight area stored on the drone, and a camera looking down. The Raspberry Pi 5 has two CSI camera ports and the stereo camera occupies both.

## Part 1 — Reference imagery

### Options

| Source | Typical resolution | Offline storage allowed? | Notes |
|---|---|---|---|
| Google Maps / Google Earth imagery | ≈ 0.3 m/px in many areas | **No** under the Maps Platform terms (caching and offline use are prohibited except narrow technical caches) | Used by published datasets, but not something to build a distributed project on |
| Other commercial web maps (Bing, Esri, Mapbox) | ≈ 0.3–0.6 m/px | Depends on each provider's terms; generally restricted `[VERIFY]` | Read the terms before use |
| Sentinel-2 (Copernicus) | 10 m/px | Yes, free and open with attribution | **Too coarse** for a small drone at 50 m |
| OpenAerialMap | 0.1–0.3 m/px where it exists | Yes (CC-BY 4.0) | Coverage is sparse; check the test site |
| National / state geoportals (for example ISRO Bhuvan) | Varies | Per their terms `[VERIFY]` | Check resolution and terms for the site |
| Purchased high-resolution satellite scene | 0.3–0.5 m/px | Per licence | Cost |
| **Own orthomosaic** from a prior GPS mapping flight | 0.05–0.10 m/px | Yes, own data | Not a satellite image, but the same pipeline; highest quality; needs one mapping flight and photogrammetry software |

### Decision

1. The system is **source-agnostic**: it consumes a georeferenced GeoTIFF converted to a "map pack". Nothing in the software depends on a provider.
2. Every map pack records its source, date and licence. Imagery whose terms forbid offline storage is not used.
3. For the project: obtain a satellite image of the test site from a source whose terms permit offline academic use (to be identified: OD-11). In parallel, make an own orthomosaic of the same site.
4. Report results for both. Accuracy and availability as a function of reference resolution and age is a useful experimental result.

### Reason

A localisation method that only works with imagery it is not allowed to carry is not a valid result. Making the map pack format independent of the source keeps the decision reversible and the experiment honest.

## Part 2 — Downward camera

### Options

| # | Option | Assessment |
|---|---|---|
| A | Point the existing stereo camera down | Loses forward obstacle sensing and forward AI view; contradicts "AI vision stays as decided" |
| B | CSI camera through a multiplexer board | Driver risk on Pi 5 + Ubuntu; cameras cannot run simultaneously on most multiplexers |
| C | **USB (UVC) camera, USB 2.0, wide lens** | Works with the standard Linux driver; independent of the CSI ports; modest bandwidth at 640×480 |
| D | Replace the stereo camera with a USB stereo/depth camera and use a CSI port for the downward camera | Larger change; tied to the stereo upgrade decision |

### Decision

**Option C.** Requirements for the camera:

| Property | Requirement | Reason |
|---|---|---|
| Interface | USB 2.0 UVC | No custom driver; avoids USB 3 radio noise near GPS |
| Shutter | Global shutter preferred; rolling shutter acceptable with short exposure | Less critical than for VIO: matching uses one image at a time |
| Resolution | ≥ 640×480 delivered; 1 MP class sensor | Scaled down to map resolution anyway |
| Lens | 90–120° horizontal field of view, fixed focus at infinity | Large footprint at 40–60 m |
| Frame rate | ≥ 15 Hz | Ground visual odometry |
| Exposure | Manual control available through UVC | Outdoor brightness; no blur |
| Mass / power | ≤ 30 g, ≤ 1.5 W | Budgets are already at their limits |
| Mount | Rigid, on the underside, clear of landing gear, on the damped plate | Sharp images; known boresight |

A specific model is chosen at procurement (OD-12). Candidates are 1 MP global-shutter UVC modules with M12 wide lenses `[VERIFY availability and price in India]`.

### Reason

USB is the only path that keeps both stereo sensors and adds a third camera without driver risk. At one match per second and 15 Hz odometry, UVC timestamp accuracy is sufficient: attitude is interpolated to the image time, and 20 ms of error at cruise yaw rates is a fraction of a degree.

## Consequences

- One more camera to calibrate (intrinsics; mounting angle relative to the flight controller).
- Roughly +25 g and +1 W: the avionics budgets, already at their limits, are exceeded on paper unless something is removed. The HDMI converter path is the first candidate ([weight-budget.md](../03-hardware/weight-budget.md)).
- A USB cable near the GPS antenna: use a short shielded cable and check GPS reception on the bench.
- A map-preparation tool and a site-registration procedure become part of the project.
- A licence statement for the reference imagery must appear in the report.
