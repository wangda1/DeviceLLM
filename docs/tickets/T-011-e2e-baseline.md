# T-011 端到端参考 pipeline 与 M1 基线报告
**环境**: 🐧 Linux · **依赖**: T-010 · **规模**: S · **里程碑**: M1（出口）

## 交付物
- `ml/photocritic/pipeline.py` 和 CLI：`uv run photocritic analyze img.jpg` 输出完整的 QualityReport JSON 和一份可读的终端摘要
- 在全部数据集上跑完整评测，生成 `ml/eval/reports/M1-baseline.md`
- 分析报告：每个标签哪里做得好、哪里差，失败样例画廊（HTML，每个标签挑 5 张），给 M3 的改进方向

## 验收标准
- [ ] 满足 SPEC §7.3 M1 的通过标准
- [ ] 报告列出 M3 要优先攻克的 3 个问题
