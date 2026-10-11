# T-014 C++ 核心：检测模型与 MobileCLIP 推理
**环境**: 🍎 macOS（也可以在 Linux 上做） · **依赖**: T-006, T-007, T-012 · **规模**: M · **里程碑**: M2a · **状态**: 待开始

## 交付物
- `core/src/detectors/`：显著性模型和人脸模型的前处理、ORT 推理、后处理（和 `detectors.py` 一致）
- `core/src/clip/`：图像编码器推理，读取 `clip_text_emb.bin`，计算标签分数、场景类别和截图过滤
- 截图或文档时提前返回 `skip_reason: "utility"`
- S1 和 S3 可以并行（线程池），ORT 会话在 engine 生命周期里复用
- 对齐测试：和 Python（onnxruntime）的输出对比，分数误差 ≤ 1e-3

## 验收标准
- [ ] 对齐测试通过
- [ ] 记录 Linux CPU 上每个模型的耗时
