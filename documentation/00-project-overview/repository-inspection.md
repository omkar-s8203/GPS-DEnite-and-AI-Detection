# Repository Inspection Report

| Field | Value |
|---|---|
| Document ID | GDN-OVR-002 |
| Version | 1.0 |
| Date | 2026-10-05 |

## 1. Method

The project directory `GPS DEnite and AI Detection/` was listed recursively, including hidden files, before any documentation was written.

## 2. Findings

| Item checked | Result |
|---|---|
| Files of any kind | None. The directory was empty. |
| Source code | None |
| ROS 2 packages (`package.xml`, `setup.py`, `CMakeLists.txt`) | None |
| Configuration files | None |
| Existing documentation | None |
| Simulation assets (worlds, models, URDF/SDF) | None |
| Hardware files (schematics, CAD, wiring) | None |
| Dependency manifests | None |
| Version control | Not a git repository |

## 3. Consequences

- This is a greenfield design. Nothing was overwritten, and there was no earlier work to merge.
- All structure in `documentation/` was created in this pass.
- The implementation layout proposed in [package-structure.md](../05-ros2/package-structure.md) does not exist yet and must not be created until the design baseline is accepted.

## 4. Recommendations before implementation starts

1. Initialise a git repository at the project root and commit `documentation/` as the first commit.
2. Rename the project directory to a name without spaces (for example `gps-denied-drone`). ROS 2 build tools and many shell scripts handle spaces in paths poorly.
3. The directory sits inside OneDrive. Move the working copy outside any cloud-synced folder before building: sync clients lock files and corrupt build trees. Use git for backup instead.
4. Development of ROS 2 code needs Ubuntu 24.04. The current workstation runs Windows 11; see [simulation-strategy.md](../11-simulation/simulation-strategy.md) §6 for the workstation options.
