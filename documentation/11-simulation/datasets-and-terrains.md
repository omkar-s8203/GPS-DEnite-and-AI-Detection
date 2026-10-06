# Datasets and Terrain Coverage

| Field | Value |
|---|---|
| Document ID | GDN-SIM-002 |
| Baseline | DB-3.0 |
| Date | 2026-10-06 |
| Status | Plan. No result has been measured. Download state is in §6 |

The project is developed in simulation on a laptop until 15 January 2027. This document lists the data that work needs, where it comes from, what its licence allows, and how the system is tested over every terrain type.

## 1. What data is needed

| Need | Used for | Checklist step | Source |
|---|---|---|---|
| Real drone photos with known positions, plus satellite maps of the same places | Proving map matching on real data (gate G2-sim) | 5, 6 | UAV-VisLoc |
| Labelled aerial images of people and vehicles | Training the aerial-view detector | 8 | VisDrone2019-DET |
| Labelled aerial images of people in open country | Search-and-rescue recall figures | 8, 10 | HERIDAL, SARD |
| Aerial images of several terrain types, two dates each | Ground texture and on-board map in the simulator | 4, 5, 12 | NAIP |
| Pretrained detector weights | Forward-view detector; starting point for training | 8 | Ultralytics YOLO26n |

## 2. Datasets

| Dataset | Content | Size | Licence | Access |
|---|---|---|---|---|
| **UAV-VisLoc** | 6,742 downward drone photos with latitude, longitude, height, date and heading; 11 satellite maps at about 0.3 m per pixel; cities, towns, villages, farms, rivers, hills | 16.4 GB (a 2.04 GB sample also exists) | Not stated in the repository. One index lists CC0. **To verify before any image is republished** | Google Drive link in the [repository](https://github.com/IntelliSensing/UAV-VisLoc) |
| **VisDrone2019-DET** | 8,629 drone images (6,471 train, 548 validation, 1,610 test) with 10 classes: pedestrian, people, bicycle, car, van, truck, tricycle, awning-tricycle, bus, motor | About 1.9 GB in three archives | CC BY-NC-SA 3.0 as reported by dataset hosts: non-commercial use, credit required | Direct download from the Ultralytics assets release; see [Ultralytics VisDrone page](https://docs.ultralytics.com/datasets/detect/visdrone) |
| **HERIDAL** | About 1,650 high-resolution (4000 × 3000) aerial images of people in Mediterranean wilderness, made for search and rescue | Several GB (not confirmed) | CC BY 3.0 | [IPSAR, University of Split](http://ipsar.fesb.unist.hr/HERIDAL%20database.html) |
| **SARD** | 1,981 labelled frames of actors walking, sitting and lying, filmed from a drone for search and rescue | A few GB (not confirmed) | IEEE DataPort terms; an account is needed | [IEEE DataPort](https://ieee-dataport.org/documents/search-and-rescue-image-dataset-person-detection-sard), or a community mirror on Kaggle |
| **NAIP** | Aerial images of the United States at 0.3 to 1 m per pixel, repeated every two to three years | Only small crops are needed: under 1 GB in total | Public domain (US Department of Agriculture) | [Microsoft Planetary Computer](https://planetarycomputer.microsoft.com/docs), open access |

Not used, and why:

| Dataset | Reason |
|---|---|
| LaDD (Lacmus) | GPL-3.0 and hosted behind a request; HERIDAL and SARD cover the same need |
| University-1652, DenseUAV | Oblique or building-centred views; the project uses a straight-down camera |
| Google Maps or Google Earth imagery | May not be stored offline ([ADR-016](../17-decisions/ADR-016-reference-imagery-and-downward-camera.md)) |

### Rules for using the data

- Datasets are never committed to the repository. They live in a data folder outside it.
- VisDrone is non-commercial. The trained weights inherit that limit; the model card must say so.
- Every dataset is cited in the report and in [references](../references.md).
- Sample images are republished in the report or slides only where the licence clearly allows it: NAIP and HERIDAL yes with credit; UAV-VisLoc only after its licence is confirmed.

## 3. Why NAIP for the simulated ground

The simulator needs two things for each test area: a picture to paint on the ground, and a different picture of the same place to carry on board as the map. If both were the same picture, matching would always succeed and prove nothing.

NAIP fits because the same area is re-photographed every two to three years, so two dates are available; it is public domain, so it can be stored, cropped and shown; and it reaches 0.3 to 0.6 m per pixel, close to the 0.3 to 0.5 m the design assumes.

Limits: it covers only the United States, so the simulated sites are not Indian landscapes. That is acceptable for simulation, where the question is how the method behaves on each terrain type. The real test site in India is a separate matter for Part 2 (OD-11).

## 4. Terrain types

Each terrain is one simulated site of about 1 km × 1 km. Sites are chosen in checklist step 4 by looking at the imagery; coordinates are recorded here when chosen.

| ID | Terrain | Why it is included | Expected for map matching | Site and image years |
|---|---|---|---|---|
| T1 | Dense urban | Roads and roof edges give many features | Should work well | To choose |
| T2 | Suburb or village | The likely real test environment | Should work well | To choose |
| T3 | Farmland | Field edges help, but crops change with season | Should work; sensitive to the date gap | To choose |
| T4 | Forest | Tree canopy looks the same everywhere | Expected to be poor | To choose |
| T5 | Desert or bare ground | Very few features | Expected to be poor | To choose |
| T6 | Coast or river | Water has no usable features | Expected to fail over water and work along the shore | To choose |
| T7 | Hills | Ground height varies, which breaks the flat-ground assumption | Expected to be degraded | To choose |
| T8 | Snow (optional) | Features hidden | Expected to be poor | Only if time allows |

The "expected" column is a prediction from the literature in [research-visual-geolocalization](../18-research/research-visual-geolocalization.md), not a result. The measured answer replaces it.

**A poor result on forest, desert or water is not a project failure.** It is the honest operating envelope, and it is exactly where the design's fallback chain matters: the drone must notice that it has no fix, hold, and hand over. Testing those terrains proves the safety behaviour.

## 5. Test matrix

Run for every terrain in step 12, with GPS switched off in simulation.

| Variable | Values |
|---|---|
| Terrain | T1 to T7 (T8 optional) |
| Height | 25 m, 40 m, 60 m |
| On-board map | Different year from the ground texture |
| Light | Normal; strong shadows |
| Camera faults | None; noise and blur; dropped frames |

Measured for each run:

| Measure | Meaning |
|---|---|
| Acceptance rate | Share of photos that give an accepted fix |
| Position error | RMS and worst case against simulation truth |
| Wrong fixes | Accepted fixes more than 15 m from the truth |
| Longest gap | Longest time without a fix |
| Outcome | Mission continued, held, or handed over |
| Search result | Targets found out of targets placed, and false alarms |

The result is one table: terrain against height, showing where the system works, where it degrades and where it correctly gives up.

## 6. What simulation over terrain does and does not show

| Shows | Does not show |
|---|---|
| How matching behaves as ground features become scarce | Real camera noise, blur and rolling shutter |
| The effect of a date gap between photo and map | Tall buildings and trees seen from the side: the simulated ground is a flat picture |
| That the fallback chain triggers where matching fails | Real weather, haze and lighting |
| Search coverage and findings on each terrain | Detection of real people: simulated people are 3D models |

For this reason the terrain results from simulation are reported **beside** the UAV-VisLoc results, which come from real drone photos over cities, towns, farms, rivers and hills. Where the two disagree, the real-data result is the one to trust.

A detector trained on real photos may detect simulated people poorly. If so, the search results in simulation are reported with a detector fine-tuned on simulator images, and that is stated plainly beside the figures.

## 7. Download state

Data folder on the development laptop: `C:\Users\IT Tech\gdn-data\`, outside the repository and outside OneDrive. Its layout is described in [data/README.md](../../data/README.md).

| Dataset | State on 2026-10-06 | Note |
|---|---|---|
| VisDrone2019-DET | **Downloaded** 2026-10-07, 1.9 GB, sizes match the source | Three archives in `datasets\visdrone\`; not yet unpacked |
| UAV-VisLoc sample (2.04 GB) | Downloading | Into `datasets\uav-visloc\`. Google Drive sometimes refuses automated downloads; if so, download in a browser |
| UAV-VisLoc full (16.4 GB) | Download started after the sample | Same caution |
| HERIDAL | Not started | Download from the university site in step 8 |
| SARD | Not started | Needs a free IEEE DataPort or Kaggle account; a team member must do this |
| NAIP crops | Not started | Fetched in step 4, once the sites are chosen |
| YOLO26n weights | Not started | Fetched automatically by the training tool in step 8 |

Disk space: about 25 GB needed in total; 100 GB was free on the laptop.
