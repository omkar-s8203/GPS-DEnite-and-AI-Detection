# Research Notes — AI Perception on the Edge

| Field | Value |
|---|---|
| Version | 1.0 |
| Date | 2026-10-05 |
| Nature | Background research. Not a design document. |

## 1. What a Raspberry Pi 5 can run

| Fact | Source |
|---|---|
| Four Cortex-A76 cores at 2.4 GHz with NEON; no NPU; VideoCore VII GPU without mainstream NN framework support | [P] Raspberry Pi |
| Roughly 2–3× the CPU performance of a Pi 4 | [L] |
| PCIe 2.0 ×1 allows an external accelerator | [P] |

Practical classes of model on the CPU alone:

| Class | Example | Feasibility |
|---|---|---|
| Nano detectors (2–3 M parameters) | YOLO26n, YOLO11n, YOLOv8n, NanoDet | Yes: ≈ 10–15 FPS at 640 px on an idle Pi; more at 320 px |
| Small detectors (9–11 M) | YOLO "s" models | ≈ 3–5 FPS at 640 px idle; not alongside VIO |
| Classification backbones | MobileNetV2/V3 | Easily real-time |
| Segmentation, pose | Nano variants | A few FPS |
| Learned depth / learned stereo / learned VIO | — | Not real-time |
| Transformers, VLMs | — | Not real-time |

## 2. Ultralytics benchmark on Raspberry Pi 5 [P]

From the Ultralytics Raspberry Pi guide (Ultralytics 8.4.x), YOLO26n, 640 px, FP32:

| Format | Size (MB) | mAP50-95 | ms / image |
|---|---|---|---|
| PyTorch | 5.3 | 0.4760 | 299.09 |
| TorchScript | 9.8 | 0.4734 | 353.20 |
| ONNX | 9.5 | 0.4734 | 125.99 |
| OpenVINO | 9.6 | 0.4734 | 104.55 |
| LiteRT | 9.8 | 0.4730 | 123.30 |
| MNN | 9.4 | 0.4749 | 91.87 |
| ExecuTorch | 9.4 | 0.4772 | 144.83 |
| **NCNN** | 9.4 | 0.4784 | **67.03** |

The guide benchmarks only the n and s models on the Pi 5, describing larger models as impractical there, and recommends NCNN for Raspberry Pi devices because it is optimised for Arm mobile/embedded CPUs.

Caveats:

- Idle board, all cores available. In this project VIO and stereo occupy most of two to three cores.
- The mAP figures are from a benchmark subset and are not the model's headline COCO score.
- 640 px. Cost scales roughly with pixel count; 320 px is expected at about a quarter [ESTIMATE].

## 3. Model notes

| Model | Notes |
|---|---|
| YOLO26 | Ultralytics' current generation at the time of writing: end-to-end (NMS-free) head and design changes aimed at simpler export and faster CPU inference [P/S]. Exact parameter counts and COCO scores to be taken from the official model card before citing. |
| YOLO11 | Previous generation; mature exports; needs NMS |
| YOLOv8 | Widely documented; many tutorials for NCNN on Raspberry Pi |
| MobileNet-SSD | Historically the default for TFLite on Raspberry Pi; clearly lower accuracy, particularly on small objects |
| NanoDet-Plus | Anchor-free, about 1 M parameters, designed around NCNN; Apache-2.0 |
| EfficientDet-Lite | TFLite model family; moderate speed and accuracy |

## 4. Runtime notes

| Runtime | Notes |
|---|---|
| NCNN (Tencent) | C++ library, no third-party dependencies, NEON-optimised, optional Vulkan; BSD-3. Fastest in the benchmark above. |
| MNN (Alibaba) | Similar niche; second fastest |
| OpenVINO | Arm CPU plugin exists; third |
| ONNX Runtime | General-purpose; XNNPACK/Arm optimisations; slower here |
| LiteRT / TensorFlow Lite | XNNPACK delegate; INT8 quantisation is its strength |
| OpenCV DNN | Convenient; lags on new operators; generally slower than dedicated mobile runtimes on Arm |
| PyTorch | For training and export; unsuitable for deployment on this board |

## 5. Quantisation

INT8 post-training quantisation typically gives 1.5–3× speed-up on Arm CPUs with a small accuracy loss, more for nano models whose capacity is already limited. Requires a calibration image set representative of the deployment domain. Operator coverage for newer heads should be verified. Treated as an optimisation experiment.

## 6. Accelerators for the Pi 5 [S][U]

| Device | Interface | Notes |
|---|---|---|
| Hailo-8L / Hailo-8 AI HAT | PCIe | Officially supported on Raspberry Pi OS; tens to hundreds of FPS for YOLO-class models. Use on Ubuntu 24.04 requires building the driver and runtime; maturity to be verified. Occupies the single PCIe lane. |
| OAK-D cameras | USB | Inference runs inside the camera on its vision processor; model must be converted to the device's format |
| Coral USB / M.2 TPU | USB / PCIe | Ageing software stack; restricted to TFLite INT8 models |

## 7. Datasets

| Dataset | Use | Note |
|---|---|---|
| COCO | Pretraining; 80 everyday classes | Ground-level viewpoints |
| VisDrone | Aerial viewpoints: pedestrians, vehicles | Small objects; check terms for academic use |
| Self-collected | Fine-tuning and honest evaluation | The only data matching this camera, lens, altitude and environment |

Evaluation practice: split by recording session; report precision/recall by range bin; include empty scenes; report failures.

## 8. Fusing detections with depth

Common patterns in the literature and in practice [L]:

| Pattern | Summary |
|---|---|
| Late fusion (box → depth ROI statistic) | Simple, robust, model-agnostic; chosen |
| Frustum methods | Extract the point-cloud frustum behind a 2D box and fit a 3D box |
| RGB-D networks | Depth as an input channel; needs matched training data |
| Stereo 3D detection networks | Heavy |

Robust ROI statistics (median or low percentile of the central region) are standard for coping with background pixels.

## 9. Role of AI in small-UAV autonomy

In fielded systems, learned models are used for semantic tasks (detection, segmentation, tracking) and increasingly for depth and obstacle perception on platforms with dedicated accelerators. Geometric estimation remains the backbone of state estimation in safety-relevant loops because its failure modes are better understood. On a CPU-only companion, limiting AI to semantic perception is both the feasible and the conservative choice.

## 10. Licensing

Ultralytics code and pretrained weights are AGPL-3.0 with a commercial licence available. Derived models used in a distributed system carry the AGPL obligation. NanoDet (Apache-2.0) and MobileNet-SSD (Apache-2.0) are permissive alternatives.

## 11. Takeaways used in the design

| Finding | Used in |
|---|---|
| NCNN fastest on Pi 5 by a clear margin | ADR-006 |
| 640 px nano inference ≈ 67 ms on an idle Pi | 320 px at 5 Hz |
| Only nano models are practical beside VIO | Model choice |
| PyTorch is 4–5× slower than NCNN on the board | PyTorch off-board only |
| Late fusion is model-agnostic | ADR-010 |
| Domain mismatch of public datasets | Self-collected fine-tuning set |
| AGPL obligations | Licence register |
