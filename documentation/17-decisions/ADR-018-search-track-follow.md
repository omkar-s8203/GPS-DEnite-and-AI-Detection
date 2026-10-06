# ADR-018 — Search, Track and Follow

| Field | Value |
|---|---|
| Date | 2026-10-06 |
| Status | **Accepted** (design baseline DB-3.0) |
| Amends | [ADR-006](ADR-006-ai-framework.md) (a second, aerial-view model), [ADR-015](ADR-015-visual-geolocalization.md) (matching height now depends on reference resolution), [ADR-016](ADR-016-reference-imagery-and-downward-camera.md) (camera resolution), [ADR-012](ADR-012-obstacle-avoidance.md) (unchanged) |

## Context

The project owner requires that, in a GPS-denied environment, the operator can track an object, have the drone follow it, and run missions such as a grid search of a disaster area for rescue.

## Options

**Camera for search and follow:** forward stereo camera; downward camera; a gimballed camera.

**Follow geometry:** behind the target at low height; above the target at fixed height.

**Search height with respect to map matching:** search at matching height (40–60 m); search low (25–30 m) with a sharper reference; climb for fixes and descend to search.

**Tracking method:** detection association with a motion filter; image correlation tracker; learned tracker (re-identification).

## Evaluation

| Question | Option | Assessment |
|---|---|---|
| Camera | Forward stereo | Sees the horizon from height; useless for looking at the ground |
| | **Downward** | Already required for map matching; gives object ground position by projection | 
| | Gimbal | Best, but mass, cost and control complexity |
| Follow geometry | Behind, low | Close to people and obstacles; only drifting odometry without GPS |
| | **Above, ≥ 20 m** | Safe separation; uses the absolute position source; simple control |
| Search height | 40–60 m | A person is ≈ 5–8 px: not detectable |
| | **25–30 m with a reference of ≈ 0.25 m/px or better** | Person ≈ 13–16 px at 1920 px width: marginal but possible; matching feasible |
| | Climb/descend | Works with coarse imagery; slow; error grows along lines |
| Tracking | **Detection association + Kalman filter** | Cheap; uses what already runs |
| | Correlation tracker | Useful between detections; optional |
| | Learned re-identification | Too heavy; raises privacy questions |

## Decision

1. **Search and follow use the downward camera**, with the detector switched to an **aerial-view YOLO26n model** in the search profile. The forward stereo camera and its ground-view model are unchanged for the low regime.
2. **Object positions come from projecting the detection onto the ground** using the drone's map-matched position, attitude and height.
3. **Grid search**: operator-drawn polygon, back-and-forth lines, finds confirmed over several frames and pinned with coordinates and a thumbnail. Flown at **25–30 m**.
4. **Matching height is computed from the map pack's resolution.** Search requires a reference of about 0.25 m/px or better; climb-for-fix is the fallback for coarser imagery.
5. **Downward camera requirement raised to ≥ 1920×1080** for detection.
6. **Tracking** by associating detections with a constant-velocity Kalman filter in ground coordinates.
7. **Follow from above** at fixed height, never descending automatically. Low-level following of people is excluded.
8. **Priority and build order:** grid search (Must) → tracking (Should) → follow (Should, last).
9. **Claims are limited** to an undamaged, mapped test site with dummy targets. Performance in a real disaster area is not claimed.

Design: [search-track-follow.md](../09-navigation/search-track-follow.md).

## Reason

- The downward camera and the ground-projection mathematics already exist for map matching; search and follow reuse them, so the additions are a planner, a tracker and a controller rather than a new sensing system.
- Coordinates of finds without GPS are a direct, demonstrable benefit of map matching, and tie the new features to the project's core.
- Following from above keeps the vehicle far from people, which is the only form a student prototype can responsibly fly.
- Building in priority order protects the project: grid search alone is a complete, useful result.

## Consequences

- **Scope grows substantially.** With the app ([ADR-017](ADR-017-ground-app.md)) this adds roughly 8–10 weeks of work. The roadmap is extended and follow is explicitly the first feature to drop.
- A second model to train and evaluate, needing an aerial-view dataset of the team's own.
- A higher-resolution downward camera and a sharper reference image than DB-2.0 required.
- Detection of people from above with a small visible-light camera and a nano model will have limited recall. This is measured and reported, and the system is described as an aid that marks candidates for a human to check.
- **Disaster areas change the ground**, which is what map matching depends on. This limitation is stated wherever the rescue use is mentioned.
- New safety rules: no flight over uninvolved people; follow tested with a consenting team member in the open area; height never reduced automatically.
- New failure modes in the FMEA (missed person, wrong target followed, app misuse).
