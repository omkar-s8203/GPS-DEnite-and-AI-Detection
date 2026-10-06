# Data

**Nothing in this folder is committed except this file.** Datasets, satellite images, recordings and model weights are large and carry their own licences, so they stay out of git.

## Where the data lives

On the development laptop the data folder is outside the repository and outside OneDrive:

```text
C:\Users\IT Tech\gdn-data\          (in WSL: /mnt/c/Users/IT Tech/gdn-data)
├── datasets/
│   ├── visdrone/         aerial images of people and vehicles, for training the detector
│   ├── uav-visloc/       real drone photos with known positions, for testing map matching
│   ├── heridal/          search-and-rescue images of people
│   └── sard/             search-and-rescue images of people
├── imagery/
│   └── naip/<terrain>/   two image years per simulated terrain
├── map_packs/<map_id>/   on-board maps built from the imagery
├── models/               trained detector weights
├── recordings/           simulation and replay logs
└── logs/                 download logs
```

Each team member keeps their own copy. Set the environment variable `GDN_DATA` to its location; the software reads data from there.

What each dataset is, its size, its licence and how to get it: [datasets-and-terrains.md](../documentation/11-simulation/datasets-and-terrains.md).
