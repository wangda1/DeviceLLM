# T-007 S1 检测模型：显著性与人脸（ONNX）
**环境**: 🐧 Linux · **依赖**: T-002, T-004 · **规模**: M · **里程碑**: M1 · **状态**: 待开始

## 背景
代替 Apple Vision 的前置识别，见 SPEC §4 S1。只用能导出为 ONNX、许可证允许使用的开源小模型。

## 交付物
- 显著性：至少比较 2 个候选（例如 U²-Netp 和另一个轻量 SOD 模型），在 golden 集上人工标注约 50 张主体 bbox 后评估 IoU；同时比较体积和 CPU 耗时
- 人脸：YuNet 或 BlazeFace，在 golden 集的人像子集上评估召回率
- `ml/photocritic/detectors.py`：通过 onnxruntime 推理；后处理（阈值、NMS、mask 转 bbox、显著性压缩成 16×16 网格）写得足够简单，方便 C++ 照着实现
- 每个模型的来源、许可证和 ONNX 文件的 SHA256 记录在 `docs/decisions/0002-detectors.md`
- 输出 `schema/features.md` 中 S1 部分的全部特征；mask 也交给 T-005 计算显著区域指标（`*_salient`）

## 验收标准
- [ ] 显著性 bbox 在 golden 标注上的平均 IoU 有基线数字；所选模型在 CPU 上 1 张图 ≤ 50 ms
- [ ] 人脸召回率 ≥ 95%（正脸、脸宽 ≥ 图宽 5% 的情况）
- [ ] 所选模型的许可证允许在 App 中使用
