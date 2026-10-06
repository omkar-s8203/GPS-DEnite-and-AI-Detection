# ADR-006 — AI Model and Runtime: YOLO26n on NCNN

| Field | Value |
|---|---|
| Date | 2026-10-05 |
| Status | **Accepted** |

## Context

The system needs object detection on a Raspberry Pi 5 CPU that is simultaneously running VIO and stereo matching. No GPU or NPU is available in the baseline. The detector is advisory (FR-035).

## Options

Runtime: PyTorch, ONNX Runtime, LiteRT (TensorFlow Lite), OpenVINO, MNN, NCNN, OpenCV DNN.
Model: YOLO26n, YOLO11n, YOLOv8n, MobileNet-SSD, NanoDet-Plus, EfficientDet-Lite0; larger YOLO variants.

## Evaluation

Runtime — vendor benchmark on Raspberry Pi 5, YOLO26n at 640 px (Ultralytics):

| Runtime | ms / image |
|---|---|
| NCNN | 67 |
| MNN | 92 |
| OpenVINO | 105 |
| LiteRT | 123 |
| ONNX Runtime | 126 |
| PyTorch | 299 |

OpenCV DNN was not in the benchmark; it is generally slower than dedicated mobile runtimes on Arm and supports new operators later.

Model:

| Model | Accuracy class | Speed class on Pi 5 | Licence | Note |
|---|---|---|---|---|
| YOLO26n | Best of the nano group | Fast; end-to-end head, no NMS | AGPL-3.0 | Current generation |
| YOLO11n | Slightly lower | Similar | AGPL-3.0 | Mature exports |
| YOLOv8n | Lower | Similar | AGPL-3.0 | Very well documented |
| MobileNet-SSD | Clearly lower | Fast | Apache-2.0 | Weak on small objects |
| NanoDet-Plus | Lower than YOLO nano | Very fast | Apache-2.0 | Permissive licence |
| "s" and larger | Higher | ≈ 3× slower or worse | — | Does not fit the budget |

## Decision

- **Model:** YOLO26n, COCO-pretrained then fine-tuned on project data.
- **Input:** 320 × 320.
- **Runtime:** NCNN (C++), FP32, 2 threads.
- **Rate:** 5 Hz, newest-frame-only.
- **Training/export:** Ultralytics + PyTorch off-board only.
- **Fallbacks:** YOLO11n if YOLO26n export to NCNN misbehaves; NanoDet-Plus if the AGPL licence becomes unacceptable.

## Reason

1. NCNN is the fastest measured CPU runtime on this exact board, by a wide margin over ONNX Runtime and LiteRT.
2. A nano model at 320 px is the largest configuration that fits beside VIO: about a quarter of the 640 px cost.
3. YOLO26n's NMS-free head simplifies C++ post-processing and removes a CPU step.
4. PyTorch on the drone is several times slower and adds a heavy dependency for no benefit.
5. Detection at 5 Hz is sufficient for an advisory function at ≤ 2 m/s.

## Consequences

- Detection range for people is limited to roughly 10–12 m at 320 px; small objects are missed.
- Ultralytics models are AGPL-3.0: the project's detector code and derived weights must be released under AGPL if distributed. Acceptable for an open academic project; stated in the report.
- The vendor benchmark is an ESTIMATE for this project. The team must measure NCNN, ONNX Runtime and LiteRT at 320 px on its own Pi, with the full stack running, and record the result ([performance-requirements.md](../14-performance/performance-requirements.md) P-40).
- INT8 quantisation and hardware accelerators are deferred to the optimisation phase.
