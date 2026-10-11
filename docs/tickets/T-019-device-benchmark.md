# T-019 iOS 真机基准测试与跨平台对齐验收
**环境**: 🍎 Mac · **依赖**: T-018 · **规模**: M · **里程碑**: M2b

## 交付物
- App 内置一个隐藏的 benchmark 模式：批量处理打包进去的 golden 子集（PNG），导出 JSONL（报告加上每个阶段的耗时和内存）
- 把导出的 JSONL 拷回 Linux，用 harness 计算指标，和 Linux 上 `cpp` pipeline 的结果对比
- ORT EP 对比：CPU vs CoreML EP，记录每个模型实际有多少节点跑在 CoreML 上，以及延迟差异
- llama.cpp：Metal 下的首 token 耗时、decode tok/s、峰值内存
- 温控测试：连续分析 20 张图，记录每张的耗时变化
- `ml/eval/reports/M2b-ios.md`

## 验收标准
- [ ] 达到 SPEC §6 的预算，Linux 和 iOS 的对齐性达标
- [ ] 不达标的项要写清楚原因和对策
