# Flight controller configuration (planned)

Empty for now. Filled in step 2 of the [development checklist](../documentation/16-development-roadmap/development-checklist.md). The same files are used in simulation and, later, on the real Pixhawk.

| Folder | Content |
|---|---|
| `params/` | ArduPilot parameter files: base set, position-source sets, failsafes |
| `scripts/` | Lua scripts that run on the flight controller, such as the companion watchdog |

Design: [flight-controller.md](../documentation/03-hardware/flight-controller.md) and [mavlink-integration.md](../documentation/10-communication/mavlink-integration.md).
