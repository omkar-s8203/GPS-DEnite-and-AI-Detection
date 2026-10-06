# ADR-010 — AI + Stereo Depth: Parallel Pipelines, Late Fusion

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

The brief sketches a serial pipeline: left + right → stereo → depth → AI detection → object + distance → navigation, and invites a better architecture if one exists.

## Options

| # | Option |
|---|---|
| A | Serial: depth first, then detection on depth-filtered or RGB-D input |
| B | Parallel: detection on the left image and stereo depth computed independently; fused per detection |
| C | Detection only, range from monocular cues (box size) |
| D | 3D detection on the stereo point cloud |

## Evaluation

| Criterion | A | B | C | D |
|---|---|---|---|---|
| Works with pretrained RGB weights | No (RGB-D needs custom training) / partial | **Yes** | Yes | No |
| Failure isolation | Depth failure stops detection | **Independent** | n/a | Coupled |
| Metric range | Yes | **Yes** | Poor (needs known object size) | Yes |
| Latency | Sum of both | **Max of both** | Detector only | High |
| CPU | Same as B | Same as A | Lowest | Highest |
| Obstacle safety independent of AI | Risk of coupling obstacles to detections | **Yes: obstacles from depth only** | No depth at all | Coupled |

## Decision

**Option B.** The detector runs on the left colour image at 5 Hz; stereo depth runs at 10 Hz; `object_localizer` samples a robust depth statistic (30th percentile of the central half of each box) and back-projects to `map`. Obstacle sectors are derived from depth alone and never depend on the detector.

Algorithm: [ai-stereo-fusion.md](../08-ai/ai-stereo-fusion.md).

## Reason

1. Pretrained detectors expect ordinary images; option B uses them unchanged.
2. Each pipeline runs at its natural rate and fails independently.
3. Geometry answers "how far"; the network answers "what". Each is used for what it is reliable at.
4. An unrecognised object must still stop the vehicle. Separating obstacle sensing from recognition guarantees this.

## Consequences

- Range is only as good as stereo depth: useful to about 6–8 m, while detection reaches further. Objects beyond depth range are reported with class and bearing and an explicit "range invalid" flag.
- Time alignment between detection and depth matters during rotation; a 150 ms gate and TF lookup at image time are required.
- Boxes include background; shrink + percentile mitigates it. Segmentation masks would be better and are deferred to an accelerator upgrade.
