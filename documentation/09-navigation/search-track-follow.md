# Search, Track and Follow Missions

| Field | Value |
|---|---|
| Document ID | GDN-NAV-005 |
| Version | 1.0 (introduced in design baseline DB-3.0) |
| Date | 2026-10-06 |
| Status | Baseline |
| Decision | [ADR-018](../17-decisions/ADR-018-search-track-follow.md) |
| Requirements | FR-120 – FR-135; NFR-085 – NFR-092 |
| Implemented by | `search_planner`, `finding_manager` (`gdn_mission`), `target_tracker` (`gdn_tracking`), follow behaviour in `navigator`, `detector` in aerial mode |

## 1. What is added

Three operator-commanded behaviours, all usable with GPS denied because position comes from satellite map matching ([visual-geolocalization.md](visual-geolocalization.md)):

| Behaviour | In one sentence | Priority |
|---|---|---|
| **Grid search** | The operator draws an area; the drone flies it line by line, detects objects on the ground, and pins each find on the map with coordinates and a picture | Must |
| **Track** | The operator taps an object in the video; the drone keeps estimating where that object is | Should |
| **Follow** | The drone stays above the tracked object as it moves | Should (last to be built) |

They are requested from the Android app ([ground-app.md](../10-communication/ground-app.md)) and executed by the mission layer under the same safety gates as any mission.

## 2. One camera does the looking: the downward camera

From above, the useful view of the ground is straight down. The search and follow behaviours therefore run the detector on the **downward** camera, not the forward stereo camera.

| | Forward stereo camera (unchanged) | Downward camera (new use) |
|---|---|---|
| Used in | Low regime, 1–10 m | Search and follow, 20–30 m |
| Detector model | Ground-view model (people, vehicles from the side) | **Aerial-view model** (people and vehicles from above) |
| Range to object | Stereo depth | Not needed: position comes from projecting the pixel onto the ground |
| Output | Class + distance ahead | Class + map position (latitude, longitude) |

There is still one detector process. `nav_mode_manager` switches its input and model by flight profile, so the CPU cost does not double.

### Object position without a range sensor

For a detection at pixel (u, v) in the downward image, the same projection used for map matching gives the ground point:

```
ray_map   = R_map_base · R_base_cam · K⁻¹ · [u, v, 1]ᵀ
ground pt = camera position + ray_map · ( h / −ray_map.z )
```

The drone's own position comes from map matching, so the object's coordinates are real coordinates even with GPS denied. Position error of a find ≈ drone position error (target ≤ 5 m) plus projection error (`h·tan δ`, about 0.5 m per degree of attitude error at 25–30 m).

## 3. The height conflict, and how it is resolved

| Need | Wants |
|---|---|
| Map matching against ordinary satellite imagery (0.3–0.5 m/px) | High: 40–60 m, so the footprint holds enough map pixels |
| Seeing a person from above | Low: a person is about 0.5 m across from above |

Pixels on a person from above (width of a standing person ≈ 0.5 m; length of a lying person ≈ 1.7 m), 100° lens:

| Height | 1280 px wide image | 1920 px wide image |
|---|---|---|
| 25 m | 4.7 cm/px → ≈ 11 px standing, 36 px lying | 3.1 cm/px → ≈ 16 px standing, 55 px lying |
| 30 m | 5.6 cm/px → ≈ 9 px, 30 px | 3.7 cm/px → ≈ 13 px, 46 px |
| 50 m | 9.3 cm/px → ≈ 5 px, 18 px | 6.2 cm/px → ≈ 8 px, 27 px |

A nano detector needs roughly 12–16 pixels on an object to have a fair chance `[ESTIMATE]`. At 50 m a standing person is not detectable with this camera class. Vehicles (≈ 4.5 m) are detectable at every height listed.

**Resolution:**

1. Search and follow are flown at **25–30 m**.
2. The downward camera requirement rises to **≥ 1920×1080 delivered** for detection (the 640×480 stream for map matching and odometry is produced by down-scaling the same frames).
3. Map matching at 25–30 m needs a **sharper reference**. The minimum matching height follows from the reference resolution:

   `h_min = 250 · GSD / (2·tan(HFOV/2))` → 0.5 m/px: 52 m; 0.3 m/px: 31 m; 0.25 m/px: 26 m; 0.10 m/px: 10 m.

   So search needs reference imagery of about 0.25 m/px or better: high-resolution satellite imagery where available, otherwise the team's own orthomosaic of the site ([ADR-016](../17-decisions/ADR-016-reference-imagery-and-downward-camera.md)). `match_min_height` is now computed from the map pack's resolution instead of being a fixed 35 m.
4. Where only 0.5 m/px imagery exists, the fallback is to **climb for a fix and descend to search**: matching at 50 m at the start of each search line pair, odometry in between. Slower, and position error grows along each line; recorded as the fallback, not the baseline.

This is the least certain part of the DB-3.0 design and is tested on recordings before flight (gate G2 sensitivity test, extended).

## 4. Detection from above

| Item | Design |
|---|---|
| Model | YOLO26n, NCNN, fine-tuned for the aerial view |
| Training data | Public aerial datasets (for example VisDrone; check licence terms) for pre-training, then the team's own downward images from the test site with dummy targets and consenting team members |
| Classes | person, vehicle (project-specific targets can be added) |
| Input | Full-resolution frame cut into overlapping tiles (1920×1080 → 6 tiles of ≈ 704×608 with overlap), each run at 640 px |
| Rate | ≈ 1 full frame per second `[ESTIMATE]` with map matching and odometry running |
| Sufficiency | At 3 m/s and a 30 m along-track footprint, each ground point is seen in ≈ 10 frames |
| Confirmation | A find is raised only when the same ground position (within 3 m) is detected in ≥ 3 frames |
| Output | Detections with ground position → `finding_manager` |

Honest expectations: recall for a standing person seen from directly above with a nano model is likely to be modest; lying figures, bright clothing and vehicles are easier. Real search-and-rescue aircraft use thermal cameras for this reason. The targets in [performance-requirements.md](../14-performance/performance-requirements.md) are stated for person-sized dummies on open ground in daylight.

## 5. Grid search

### 5.1 Planning (`search_planner`)

| Step | Operation |
|---|---|
| 1 | Receive the polygon (latitude/longitude) and settings from the app |
| 2 | Validate: inside map coverage by ≥ 60 m, inside the geofence, area ≤ limit for the battery, height within the profile |
| 3 | Choose the sweep direction along the polygon's longest side (fewest turns) |
| 4 | Generate parallel lines (back-and-forth, "lawnmower" pattern) spaced `swath × (1 − overlap)` |
| 5 | Add a lead-in from the current position and a hold point at the end |
| 6 | Estimate time; refuse if it exceeds the usable battery with reserve |
| 7 | Return the plan to the app for display; start on confirmation |

Swath (across-track footprint) at height h with a 100° lens: `2·h·tan 50°`.

| Height | Swath | Line spacing at 30 % overlap | Area rate at 3 m/s |
|---|---|---|---|
| 25 m | 60 m | 42 m | ≈ 125 m²/s (0.75 ha/min) |
| 30 m | 72 m | 50 m | ≈ 150 m²/s (0.9 ha/min) |

With about 5–6 minutes available for the search itself in a 10–12 minute flight, the coverable area is roughly **3–5 hectares per battery** `[ESTIMATE]`, before turns and wind.

### 5.2 Execution

- Lines are flown nose-first at constant height and ≤ 3 m/s using the existing `GoTo` action.
- Progress, covered area and remaining time are published at 1 Hz.
- The search **pauses** (hold) when: navigation mode is `VISION_DEGRADED`; position uncertainty exceeds 10 m; the safety supervisor requests a hold; the operator presses Pause.
- The search **aborts** when: the pilot changes mode; localisation is lost; battery reserve is reached. The drone then holds or follows the normal fallback.
- Coverage is recorded from the actual camera footprints, not from the plan, so gaps caused by pauses or drift are visible on the map.

### 5.3 Findings (`finding_manager`)

| Field | Content |
|---|---|
| Id, time | — |
| Class, confidence | From the detector, best of the confirming frames |
| Position | Latitude, longitude, map x/y, estimated uncertainty (drone uncertainty + projection) |
| Evidence | Thumbnail crop; number of confirming frames; height and navigation mode at the time |
| Review | Unreviewed / confirmed / rejected by the operator |

Finds closer than 4 m to an existing find of the same class are merged. Everything is logged and exportable.

`Go there`: fly to a point above the finding at search height and hold, so the operator can look.

## 6. Tracking (`target_tracker`)

| Item | Design |
|---|---|
| Selection | Tap in the app → frame id and normalised position → the detection containing that point in that frame (or the nearest within 40 px) |
| State | Ground position and velocity of the target in `map` (constant-velocity Kalman filter) plus the last image box |
| Association | Each new detection set: nearest detection of the same class to the predicted ground position, within a gate that grows with time since the last update |
| Between detections | Prediction only in the baseline. A lightweight image tracker (correlation filter) to bridge frames is optional if the detector rate proves too low |
| Lost | No association for 3 s → `LOST`; keeps predicting for 15 s to allow re-acquisition; then the track ends and the last known position is reported |
| Works on | Either camera: forward camera at low height (with stereo range), downward camera from above |

Limits: similar-looking objects close together can be confused; the tracker follows a class and a position, not an identity. It is not a person-identification system.

## 7. Follow

### 7.1 Chosen form: follow from above

The drone holds a fixed height (≥ 20 m above the ground) and keeps the target near the centre of the downward image.

| Property | Value |
|---|---|
| Height | Fixed at the height where follow was started, 20–30 m; **never descends** on its own |
| Command | Velocity = target velocity estimate + gain × horizontal offset between the target's ground position and the point below the drone; limited to 3 m/s and the usual acceleration limit |
| Yaw | Nose along the direction of travel when moving; unchanged when nearly stationary |
| Target speed supported | ≈ walking to jogging pace, ≤ 2.5 m/s `[ESTIMATE]` |
| GPS denied | Works: the control uses the ground-projected offset and odometry; map matching keeps the absolute position for reporting and for the fence |
| Boundaries | Follow stops and the drone holds at the geofence or within 60 m of the map edge |
| Target lost | Hold position; resume automatically if re-acquired within 15 s; otherwise end and report the last position |
| App link lost | Continue 10 s, then hold |
| Navigation degraded | Hold |

### 7.2 Why not follow behind at low height

| | Follow from above (chosen) | Follow behind at 2–5 m |
|---|---|---|
| Distance from the person | ≥ 20 m | A few metres |
| Obstacles | None at that height on a cleared site | Trees, wires, walls; forward stereo sees only 73° and 6 m |
| Position source without GPS | Map matching + odometry | Drifting stereo odometry only |
| Consequence of a control error | Drifts in open air | Possible contact with the person |

Low-level following of people is excluded from the project. It can be demonstrated in simulation only.

## 8. Using these in a disaster area: what can and cannot be claimed

The motivating use is searching a disaster area. The design must be honest about it.

| Issue | Effect | Position of this project |
|---|---|---|
| Map matching compares the ground with a **pre-disaster** image | Collapsed buildings, flooding, debris and fire change exactly the features being matched; fixes become fewer or wrong | Demonstrated only on an undamaged test site. Robustness to scene change is **not claimed**. The learned matcher comparison and the odometry-carry time are the relevant measurements |
| People are hard to see from above with a small visible-light camera | Missed detections | Reported as measured recall on dummies; thermal imaging is named as the realistic next step |
| Flying over people | Not permitted in tests | Dummies and consenting team members at the edge of the area; no flight over uninvolved people |
| Real operations need authorisation and coordination | — | Out of scope |
| A missed person has real consequences | — | The system is an aid that marks candidates for a human to check. "Area searched" must never be reported as "area clear" |

What **can** be claimed if the tests succeed: that a low-cost drone can fly a search pattern and report object coordinates without GPS over a mapped, undamaged site, with stated accuracy and detection rate.

## 9. Mission layer changes

| Element | Change |
|---|---|
| `mission_manager` | New behaviours `SEARCH`, `FOLLOW`, `GOTO_FINDING` alongside the DB-1.0 step list; one behaviour active at a time; app requests arrive through `app_gateway` |
| `navigator` | New action `FollowTarget`; cruise limits from the active profile |
| Flight profiles | A third profile, **search** (25–30 m), between low (1–10 m) and cruise (40–60 m). Stereo nodes off; geo-localisation on; detector on the downward camera |
| `nav_mode_manager` | `match_min_height` derived from map pack resolution; detector input/model selected by profile |
| Speed limiting | Unchanged rules; search and follow use the cruise limits |

Pre-conditions for any of the three, in addition to the usual ones: navigation mode `GPS_NAV` or `VISION_NAV` with sub-mode `GEO`; height inside the search profile; the detector's aerial model loaded.

## 10. Verification

| Test | Level | Pass |
|---|---|---|
| Lawnmower planner: coverage of convex and concave polygons; spacing; refusal cases | L1 | ≥ 98 % planned coverage; all invalid areas refused |
| Pixel-to-ground projection | L1 | Within 0.2 m at 30 m on synthetic data |
| Tracker association and loss logic | L1 | Synthetic crossings and dropouts |
| Aerial detector on held-out downward images | L2 | Recall and precision per class and per height recorded |
| Finding position: dummies at surveyed GNSS points, drone on GNSS | L8 | ≤ 5 m |
| Grid search in simulation with GNSS disabled | L4/L5 | Area covered ≥ 95 %; all placed targets found or the miss explained |
| Follow in simulation: target at 1, 2, 3 m/s, with turns | L4/L5 | Target stays in the central half of the image up to 2 m/s |
| Link-loss, pause, abort, fence and coverage-edge cases | L5 | As specified |
| Grid search in flight on GNSS, then with GNSS disabled | L8 | Finding position ≤ 8 m; coverage map complete |
| Follow in flight: a team member walking in the open area, on GNSS first | L8 | Holds above the target for 60 s at walking pace |

## 11. Open points

| # | Item |
|---|---|
| STF-1 | Reference imagery at ≈ 0.25 m/px or better for the test site, or own orthomosaic |
| STF-2 | Downward camera with ≥ 1920×1080 over USB 2.0 at a useful frame rate (MJPEG) |
| STF-3 | Measured recall for person-sized targets from 25–30 m |
| STF-4 | Tiled inference rate on the Pi with map matching running |
| STF-5 | Whether an image tracker is needed between detections for follow |
| STF-6 | Classes and dummy targets for the demonstration (decision OD-2) |
