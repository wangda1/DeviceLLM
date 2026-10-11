# 固定版本记录

所有 ticket 都基于这里记录的版本。升级任何一项都要**单独提交**，说明原因，并重新跑受影响的测试和基准。"待定"表示还没有 ticket 固定它；负责的 ticket 完成时必须填好。

## 工具链

| 组件 | 版本 | 由哪个 ticket 固定 | 备注 |
|---|---|---|---|
| macOS | 待定 | T-001 | |
| Xcode / iOS SDK | 待定 | T-000 | 需要 iOS 26 SDK |
| Apple Clang | 待定 | T-001 | |
| CMake | 待定（≥ 3.24） | T-001 | |
| Python | 待定（3.12） | T-001 | |
| uv | 待定 | T-001 | 依赖以 `ml/uv.lock` 为准 |
| GoogleTest | 待定 | T-001 | FetchContent 固定 tag |
| nanobind | 待定 | T-001 | |
| iPhone 基准机 | iPhone 16 Pro，iOS 待定 | T-000 | |

## 推理运行时

| 组件 | 版本 | 由哪个 ticket 固定 | 备注 |
|---|---|---|---|
| llama.cpp | commit 待定 | T-000 | T-015、T-016 必须使用同一个 commit |
| ONNX Runtime（macOS） | 待定 | T-012 | |
| ONNX Runtime（iOS，含 CoreML EP） | 待定 | T-017 | |
| ONNX Runtime（Android，含 XNNPACK EP） | 待定 | T-020 | |
| nlohmann/json | 待定 | T-012 | |
| stb_image | 待定 | T-012 | 只给 CLI 使用 |

## 模型文件

SHA256 在第一次下载时计算并填写。

| 用途 | 仓库 | 文件 | SHA256 | 由哪个 ticket 固定 |
|---|---|---|---|---|
| VLM 默认档候选 | `LiquidAI/LFM2.5-VL-450M-GGUF` | `LFM2.5-VL-450M-Q4_0.gguf`、`mmproj-LFM2.5-VL-450m-Q8_0.gguf` | 待定 | T-000 |
| VLM 默认档候选 | `unsloth/Qwen3.5-0.8B-GGUF` | `Qwen3.5-0.8B-Q4_K_M.gguf`、`mmproj-F16.gguf` | 待定 | T-000 |
| VLM 默认档候选 | `ggml-org/InternVL3-1B-Instruct-GGUF` | `InternVL3-1B-Instruct-Q8_0.gguf`、`mmproj-InternVL3-1B-Instruct-Q8_0.gguf` | 待定 | T-000 |
| VLM 高质量档候选 | `Qwen/Qwen3-VL-2B-Instruct-GGUF` | `Qwen3VL-2B-Instruct-Q4_K_M.gguf`、`mmproj-Qwen3VL-2B-Instruct-Q8_0.gguf` | 待定 | T-000 |
| VLM 上限参照 | 待核实 | Qwen2.5-VL-3B Q4_K_M + mmproj | 待定 | T-000 |
| 纯文本对照组 | 待核实（LFM2.5-350M GGUF） | Q4_0 | 待定 | T-000 |
| 显著性 | 待定（U²-Netp 等） | ONNX | 待定 | T-007 |
| 人脸 | opencv_zoo `face_detection_yunet` | ONNX（具体版本待定） | 待定 | T-007 |
| CLIP 图像编码器 | apple/ml-mobileclip（S0/S2） | 导出的 ONNX | 待定 | T-006 |

## 契约版本

| 文件 | 版本 |
|---|---|
| `schema/report.schema.json` | `schema_version` 1.0 |
| `schema/vlm_output.schema.json` | 1.0 |
| `schema/taxonomy.yaml` | 1 |
| `schema/features.md` / `assets/rules.yaml` 的 `features_version` | 1 |
| `schema/rules_dsl.md` / `assets/rules.yaml` 的 `version` | 1 |
