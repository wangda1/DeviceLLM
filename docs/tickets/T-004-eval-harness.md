# T-004 评测 harness
**环境**: 🐧 Linux · **依赖**: T-002, T-003 · **规模**: M · **里程碑**: M0

## 背景
统一评测入口。任何 pipeline 变体（不同模型、阈值或 prompt）都通过它出数字，结果可以对比。见 SPEC §7.2。

## 交付物
- `ml/eval/run.py --pipeline <name> --split dev|test --dataset synthetic|spaq|para|golden`
- Pipeline 接口：`analyze(image_path) -> QualityReport`；harness 负责并行执行、缓存中间特征（按 image hash 和阶段版本）
- 指标：
  - 每个标签和宏平均的 P/R/F1，外加混淆分析（最常见的误报和漏报）
  - 维度分数和数据集属性分之间的 SRCC/PLCC
  - schema 合法率、每个阶段的耗时
  - 建议质量：LLM-as-judge 接口先留好（评分细则见 SPEC §7.2），实现可以放到 T-010
- 输出 `ml/eval/reports/<date>-<pipeline>.md` 和 `.json`；`compare.py A B` 生成两个报告的 diff 表
- 一个 `dummy` pipeline（输出随机结果）作为 harness 自测

## 验收标准
- [ ] dummy pipeline 在 synthetic 的 dev 集上跑完并生成报告
- [ ] 重复运行时命中特征缓存
- [ ] `compare.py` 能输出 Markdown 格式的对比表
