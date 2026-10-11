# T-021 在 MobileCLIP 特征上训练多头打分器
**环境**: 🐧 Linux · **依赖**: T-016 · **规模**: L · **里程碑**: M3

## 背景
用监督学习替换零样本打分，见 SPEC §4 S3 阶段二。

## 交付物
- 特征缓存：把全部训练图片的 MobileCLIP 图像 embedding 预先算好（冻结编码器）
- 模型：在 embedding 上接多头 MLP，包括 4 个维度的回归头（用 SPAQ/PARA 的属性分监督）和 N 个标签的多标签分类头（用 synthetic 标签监督）
- 实验：只训练头 vs 用 LoRA 微调编码器的最后几层；不同数据配比；评估在真实照片上的泛化能力（synthetic → golden）
- 导出 ONNX：把头部和编码器合并成一个模型，或者把头部单独导出（在 iOS CoreML EP 和 Android XNNPACK 上各测一遍，选效率更好的）
- 核心中 `ClipScorer` 支持"零样本"和"训练头"两种模式（通过配置切换）；融合规则改为使用新分数，重新调阈值

## 验收标准
- [ ] 宏平均 F1 和 SRCC 相对 M1 基线有提升（test 集，用 `cpp` pipeline 评测）
- [ ] 端侧 S3 的耗时不退化
- [ ] 训练配置和结果记录在 `docs/decisions/0006-scorer-training.md`
