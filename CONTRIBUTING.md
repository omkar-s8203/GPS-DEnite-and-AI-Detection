# Contributing

Thank you for helping with the GPS-denied drone project. This guide applies to team members and outside contributors alike.

## Before you start

1. Read the [README](README.md) for what the project is.
2. Skim the [system architecture diagrams](documentation/19-system-architecture-diagrams/README.md).
3. Open the [development checklist](documentation/16-development-roadmap/development-checklist.md) to see what is being built now.

The project is at the start of the build. Most checklist items are open.

## Ways to contribute

| Contribution | Needs hardware? | Where to look |
|---|---|---|
| Fix or clarify documentation | No | `documentation/` |
| Simulation scenarios and tests | No | Checklist steps 3 and 4 |
| Logic with unit tests (mode switching, planners, filters) | No | Checklist steps 4, 6, 11, 12 |
| Android app against the mock gateway | No, an emulator is enough | Checklist step 9 |
| Map matching on recorded images | No, recordings are enough | Checklist step 5 |
| Camera drivers, bench tests | Yes | Checklist steps 7, 8, 13 |
| Aerial images for training and testing | A camera drone | Checklist steps 5 and 10 |
| Review a pull request | No | Open pull requests |

## How to contribute

1. **Find or open an issue.** Say which checklist item it belongs to. For anything larger than a small fix, wait for a short reply before starting, so two people do not do the same work.
2. **Fork the repository**, or create a branch if you are on the team.
3. **Name the branch** after the work: `feature/map-matcher-gates`, `fix/readme-links`, `docs/app-protocol`.
4. **Make the change** in small commits.
5. **Run the checks** that exist for the part you touched (see below).
6. **Open a pull request** to `main` and fill in what changed, why, and how you tested it.
7. **Respond to review.** One approval from a maintainer is needed to merge.

## Commit messages

One short line in the present tense, then an optional explanation.

```text
Add inlier-ratio gate to map matcher

Rejects matches with fewer than 25 % inliers. Threshold from
visual-geolocalization.md section 6.
```

## Rules for code

These follow from the design and are not optional.

| Rule | Reason |
|---|---|
| Nothing on the companion computer or in the app arms the drone, selects a flight mode, or sends attitude or motor commands | The pilot and flight controller own flight and safety |
| ROS code uses ENU and FLU frames. Conversion to the flight controller's frames happens only inside MAVROS | One place for frame conversion prevents sign errors |
| Third-party sources are pinned in `deps.repos`, never copied into the repository | Reproducible builds and clear licences |
| Follow the package and topic names in [package-structure.md](documentation/05-ros2/package-structure.md) and [interfaces.md](documentation/05-ros2/interfaces.md) | Nodes are replaceable only if the contracts hold |
| New logic comes with a unit test | The switching logic is safety-relevant |
| No secrets, keys or personal data in commits | The repository is public |

Style: C++ follows the ROS 2 style checked by `ament_lint`; Python follows PEP 8; Kotlin follows the official Kotlin style guide. Match the code around your change.

## Rules for data

- **Satellite and map imagery:** do not commit it unless its licence clearly allows redistribution. Map packs stay out of the repository by default.
- **Photos and video:** do not commit images in which people can be identified, unless they agreed.
- **Large files** such as recordings and model weights do not belong in git. Describe where they are stored instead.

## Rules for results

- State every number as one of TARGET, ESTIMATE, MEASURED or VALIDATED, as the [performance document](documentation/14-performance/performance-requirements.md) does.
- Report failed tests as failures. A measured result that misses its target is useful; an invented one is not.
- Do not claim the system is the first or the best of its kind.

## Rules for documentation

- Design changes go into the matching file under `documentation/`, in the same pull request as the code that needs them.
- A change to a recorded decision needs a new decision record in [documentation/17-decisions](documentation/17-decisions/README.md); do not rewrite an old one.
- Diagrams are written in Mermaid so they can be reviewed as text.
- Tick the item in the development checklist when the work that completes it is merged.

## Checks before a pull request

The automated checks are set up in checklist step 2. Until then, check by hand:

- [ ] The change builds.
- [ ] Existing tests still pass and new logic has a test.
- [ ] Documentation links you touched still work.
- [ ] No map imagery, recordings, weights or secrets are included.
- [ ] The pull request says which checklist item it advances.

## Safety when testing

- Propellers off for all bench work.
- No flight test without the safety pilot and the pre-flight check in [safety-architecture.md](documentation/12-safety/safety-architecture.md).
- If you are an outside contributor, do not fly this software on your own aircraft. It is unproven.

## Questions

Open an issue with the label `question`. For anything about a design choice, the decision records usually already hold the answer.

## Licence of contributions

The project has not yet chosen a licence. By opening a pull request you agree that your contribution may be released under the open-source licence the team selects. If that matters to you, ask in an issue before contributing.
