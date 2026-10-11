# T-016 C++ 核心端到端评测
**环境**: 🍎 macOS（也可以在 Linux 上做） · **依赖**: T-011, T-013, T-014, T-015 · **规模**: S · **里程碑**: M2a（出口） · **状态**: 待开始

## 交付物
- 在 T-004 的 harness 中加入 `cpp` pipeline（通过 nanobind 调用核心）
- 在全部数据集上对比 `cpp` 和 Python 原型，生成 `ml/eval/reports/M2a-core-parity.md`
- 把 `cpp` pipeline 设为之后的**默认评测对象**，Python 原型只用于研究新特性

## 验收标准
- [ ] 满足 SPEC §7.3 M2a 的通过标准（S4 标签一致率 ≥ 98%）
- [ ] 不一致的样例都有原因分析
