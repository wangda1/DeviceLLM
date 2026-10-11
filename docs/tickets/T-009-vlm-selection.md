# T-009 VLM 选型 spike（llama.cpp）
**环境**: 🐧 Linux · **依赖**: T-000, T-004 · **规模**: M · **里程碑**: M1 · **状态**: 待开始
**⚠️ 风险最高，T-004 完成后优先开始**

## 背景
确定 S5 用哪个模型、什么量化、多大的输入分辨率。见 SPEC §4 S5 和 §9。

## 交付物
- 编译 llama.cpp（CUDA 版本，同时编译一个纯 CPU 版本：用来粗略估计端侧耗时，也作为 Android 只用 CPU 时的参考），学会 `llama-mtmd-cli` 和 `llama-server` 的多模态用法
- 候选模型：以 T-000 真机测试后留下的模型为准（默认档：LFM2.5-VL-450M、Qwen3.5-0.8B、InternVL3-1B；高质量档：Qwen3-VL-2B；参照：Qwen2.5-VL-3B），外加**纯文本小模型对照组**（只输入 findings，不输入图片）
- 重点回答两个问题：① 1B 以下的模型零样本质量差多少，差距能不能靠 T-022 的 LoRA 补回来；② 看图比纯文本好多少（例如能不能说出"天空过曝"、"右下角杂物"这类和画面位置相关的建议）
- 在 golden 集上用同一个 prompt（先给 findings，再让模型输出 JSON 建议）比较：
  - 建议质量（judge 打分加上人工看 10 张）、中文流畅度、幻觉率
  - 图像 token 数、图像编码耗时、prefill 和 decode 速度
  - 估算端侧内存：模型文件 + mmproj + KV cache（上下文 2k 时）
- 输入分辨率和图像 token 实验：384 / 512 / 768；LFM2.5-VL 再测 `--image-max-tokens` 64 / 128 / 256

## 验收标准
- [ ] `docs/decisions/0003-vlm-selection.md`：对比表加结论（首选模型和备选模型）
- [ ] 默认档和高质量档各选出一个模型，内存和延迟以 T-000 的真机数据为准
- [ ] 把 llama.cpp 多模态推理的要点写入 LEARN.md（mmproj 是什么、图像 token 怎么来的）
