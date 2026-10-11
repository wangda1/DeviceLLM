# T-006 S3 MobileCLIP 零样本打分与 ONNX 导出
**环境**: 🐧 Linux · **依赖**: T-002, T-004 · **规模**: M · **里程碑**: M1

## 背景
"专用小模型"的第一版，见 SPEC §4 S3。主要学习 CLIP 零样本 IQA（CLIP-IQA 思路）的做法和它的局限，同时打通 PyTorch → ONNX 的导出流程。

## 交付物
- `ml/export/clip_to_onnx.py`：只导出图像编码器（固定输入尺寸，opset ≥ 17），分别导出 FP32、FP16 和 INT8 动态量化三个版本；和 PyTorch 输出做数值对比
- `ml/photocritic/clip_scorer.py`：**通过 onnxruntime 推理**（和端侧用的是同一个文件），对每个标签用正负 prompt 对算出分数
- `assets/prompts/clip_prompts.yaml`：每个标签至少 3 组 prompt 做 ensemble；另外加上场景类别的 prompt 和截图/文档过滤的 prompt
- 文本 embedding 预先算好，导出为 `assets/clip_text_emb.bin`，附带格式说明（端侧只需要跑图像编码器）
- 实验记录：S0 vs S2、prompt ensemble vs 单个 prompt、INT8 的精度损失、哪些标签 CLIP 擅长，哪些不擅长（预期：构图和主体相关的标签较弱，色彩和清晰度较强）

## 验收标准
- [ ] harness 中以 `clip_only` pipeline 跑出 synthetic、SPAQ、PARA 的指标
- [ ] ONNX 和 PyTorch 的 embedding 余弦相似度 ≥ 0.999（FP32/FP16）
- [ ] 截图过滤在一个小测试集（截图、文档、正常照片各 50 张）上准确率 ≥ 95%
- [ ] 实验结论写入 `docs/decisions/0001-clip-variant.md`
