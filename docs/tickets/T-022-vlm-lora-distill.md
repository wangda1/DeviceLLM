# T-022 VLM 建议生成：蒸馏数据与 LoRA 微调
**环境**: 🐧 Linux · **依赖**: T-016 · **规模**: L · **里程碑**: M3

## 背景
默认档用的是 1B 以下的小 VLM，中文摄影建议的质量是主要短板。小模型从针对任务的微调中收益最大，这个 ticket 决定默认档能不能用。用大模型当 teacher 生成高质量数据，再给小模型做 LoRA 微调。

## 交付物
- 蒸馏数据：用 teacher 模型（云端大模型，或者在本机 GPU 上跑的大号开源 VLM）对每张图加上 findings，按 T-010 的 prompt 生成建议 JSON；过滤掉 schema 不合法、没有命中问题或有幻觉的结果；目标 5k–20k 条
- 训练：在 T-009 选出的**默认档模型**上做 LoRA SFT（如果时间允许，也对高质量档做一次，比较微调后两档的差距是否缩小）；只训练 LLM 部分，冻结视觉塔，框架可以选 LLaMA-Factory、ms-swift 或 HF PEFT
- 合并 LoRA 权重 → 转成 GGUF → 量化成 Q4_K_M；验证 llama.cpp 能正常加载
- 评测：judge 分数、幻觉率、schema 合法率，和 M1 的零样本 VLM 对比

## 验收标准
- [ ] judge 分数相对 M1 有提升
- [ ] 量化后质量下降 < 5%（相对 FP16 的 LoRA 合并模型）
- [ ] 记录在 `docs/decisions/0007-vlm-lora.md`，并把蒸馏和 LoRA 的踩坑写入 LEARN.md
