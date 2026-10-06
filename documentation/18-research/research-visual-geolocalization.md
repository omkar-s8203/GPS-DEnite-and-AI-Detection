# Research Notes — Visual Geo-Localisation with Satellite Imagery

| Field | Value |
|---|---|
| Version | 1.0 (added with DB-2.0) |
| Date | 2026-10-05 |
| Nature | Background research. Not a design document. Source marks as in [README](README.md). |

## 1. The problem

Absolute visual localisation: estimate a UAV's position in a global frame by relating what its camera sees to a georeferenced reference (satellite image, orthophoto, map). It complements relative methods (VO, VIO), which drift.

Why it is hard: the reference and the live image differ in sensor, resolution, viewing angle, season, time of day, shadows and scene content (vehicles, crops, construction). The task is cross-domain image matching.

## 2. Families of methods [L][S]

| Family | Idea | Strength | Weakness |
|---|---|---|---|
| Template / correlation matching | Slide the live image over the map; maximise correlation or mutual information | Simple; no features | Sensitive to appearance change, rotation and scale; needs good prior attitude and height |
| Hand-crafted local features (SIFT, ORB, AKAZE) + RANSAC | Match keypoints; fit a geometric transform | No training; geometric verification; cheap | Degrades with seasonal/illumination change and low texture |
| Learned local features and matchers (SuperPoint, LightGlue, LoFTR, XFeat) | CNN/transformer features | More robust across domains | Compute; most need a GPU |
| Image retrieval (place recognition) then matching | Global descriptor finds candidate tiles; local matching refines | Scales to large maps | Two stages; training |
| Semantic matching | Match roads, buildings, land-use masks | Invariant to appearance | Needs a segmentation network |
| Probabilistic localisation | Particle filter / Monte-Carlo over map position, with a learned or correlation likelihood | Handles ambiguity; integrates motion | Compute; tuning |

For a small prior area (about 1 km²) with a good position prior from odometry, retrieval is unnecessary: the search window is already small.

## 3. Sources consulted in this pass

| Source | Finding | Mark |
|---|---|---|
| Kinnari et al., "GNSS-denied geolocalization of UAVs by visual matching of onboard camera images with orthophotos" (arXiv 2103.14381) | States that the typical approach for small UAVs is a downward camera matched to a map; proposes Monte-Carlo localisation using inertial measurements, a camera and an orthoimage map | [S] |
| Kinnari et al., "Season-invariant GNSS-denied visual localization for UAVs" (arXiv 2110.01967) | Addresses seasonal appearance change with learned similarity | [S] |
| "GNSS-denied UAV localization with satellite and aerial image matching" (ScienceDirect, 2025) | Recent work combining satellite and aerial references | [S] |
| Retrieval-and-matching works for UAV absolute visual localisation | Retrieval selects candidate satellite patches; matching refines; only a downward camera and public satellite imagery needed | [S] |
| UAV-VisLoc dataset (arXiv 2405.11936) | 6,742 drone images with latitude, longitude, altitude, date and heading; 11 satellite maps at about 0.3 m resolution extracted from Google Earth; fixed-wing and multirotor, several altitudes | [S] |
| AerialVL dataset | 11 flight sequences of 3.7–11 km over varied terrain, altitude and lighting | [S] |
| XFeat (Potje et al., CVPR 2024) | Lightweight learned features; reported up to 5× faster than other learned features with comparable accuracy; real-time on a laptop i5 CPU at VGA without special optimisation; 64-dimensional descriptors; sparse and semi-dense modes; code and weights public; a C++ implementation exists | [S] |
| ArduPilot `GPS_INPUT` (GPS type 14, MAVLink) | A companion can feed latitude/longitude/velocity as a MAVLink GPS; used by visual-navigation integrations | [S] |
| Google Maps Platform terms | Caching/offline storage of map content is prohibited except limited temporary technical caches | [S] |
| Copernicus Sentinel-2 | Free and open; 10 m best resolution | [S] |
| OpenAerialMap | Openly licensed (CC-BY 4.0) satellite and UAV imagery; high resolution where available | [S] |
| India Drone Rules 2021 summaries | Green zone up to 400 ft / 120 m; micro category 250 g – 2 kg; visual line of sight; daytime | [S], verify against the official text |

Earlier literature to cite after checking details [L]: Goforth & Lucey, "GPS-denied UAV localization using pre-existing satellite imagery" (ICRA 2019); Patel et al., "Visual localization with Google Earth images for robust global pose estimation of UAVs" (ICRA 2020); Bianchi & Barfoot, "UAV localization using autoencoded satellite images" (RA-L 2021); Couturier & Akhloufi, "A review on absolute visual localization for UAV" (Robotics and Autonomous Systems 2021); Lowe, "Distinctive image features from scale-invariant keypoints" (IJCV 2004); DeTone et al., SuperPoint (2018); Lindenberger et al., LightGlue (ICCV 2023).

## 4. What published systems typically assume

| Assumption | Present in this project? |
|---|---|
| Downward (or gimballed nadir) camera | Yes (to be added) |
| Height of tens to hundreds of metres | Yes: 40–60 m planned |
| Attitude and heading from the autopilot | Yes |
| Barometric or laser height | Barometric |
| Reference at 0.3–1 m/px | To be sourced |
| GPU or strong CPU on board, or offline processing | **No**: Raspberry Pi 5 CPU. Main deviation; drives the choice of a classical matcher and precomputed map features |
| Relative odometry between fixes | Yes (ground VO) |

## 5. Geometry notes

| Quantity | Relation |
|---|---|
| Ground sample distance of the drone image | `GSD_cam = h / f_px` (per pixel at nadir) |
| Footprint width | `2·h·tan(HFOV/2)` |
| Position error from attitude error δ | `h·tan δ` (0.87 m per degree at 50 m) |
| Position error from heading error ψ at offset r from the image centre | `r·ψ` (zero at the centre) |
| Scale error from height error | `Δh / h`; no effect at the image centre |

Using the image centre as the reported point makes the fix insensitive to heading and scale errors to first order; roll and pitch errors remain the main geometric term.

## 6. Feature-matching practice

- Bring both images to the same scale and orientation before extracting features: rotation- and scale-invariance are then spare capacity instead of the thing being relied upon.
- Contrast normalisation (CLAHE) on both.
- Ratio test (≈ 0.75–0.8) and RANSAC with a low-degree-of-freedom model (similarity, not full homography) when the geometry is already approximately known: fewer parameters means fewer false consensus sets.
- Constrain the estimated rotation and scale to plausible ranges.
- A minimum inlier count alone is not sufficient: also check the spatial spread of inliers and consistency with motion.

## 7. Failure cases reported in the literature

| Case | Why |
|---|---|
| Farmland, grassland, forest, water, desert | Few stable distinctive features |
| Season mismatch (crops, foliage, snow) | Appearance change |
| Low sun, long shadows | Strong gradients not in the reference |
| New construction, demolished buildings | Scene change since the reference was taken |
| Repetitive urban grids | Perceptual aliasing: plausible wrong matches |
| Low height relative to reference resolution | Too few reference pixels |
| Oblique views | Perspective mismatch |

## 8. Takeaways used in the design

| Finding | Used in |
|---|---|
| Downward camera + map is the standard approach for small UAVs | ADR-015 |
| Systems combine absolute matching with inertial/odometry | Fusion design |
| Public datasets with ground truth exist | Verification plan before any flight |
| Reference resolution about 0.3 m is typical | Height regime 40–60 m |
| Learned features are more robust; XFeat is the CPU-feasible one | Planned comparison |
| Google imagery cannot be stored offline | ADR-016 |
| Sentinel-2 is open but too coarse | ADR-016 |
| `GPS_INPUT` is an available integration path | Fallback |
| 120 m green-zone ceiling | Cruise height 50 m is inside it; verify for the site |
