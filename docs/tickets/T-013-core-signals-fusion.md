# T-013 C++ 核心：信号指标、融合规则与模板
**环境**: 🍎 macOS（也可以在 Linux 上做） · **依赖**: T-005, T-008, T-010, T-012 · **规模**: L · **里程碑**: M2a · **状态**: 待开始
**信号指标部分在 T-005 和 T-012 完成后就可以开始**

## 背景
把 Python 原型中确定性的部分（S2、S4、S5'）移植到 C++，以 Python 实现作为规范。

## 交付物
- `core/src/signals/`：按 `signals.py` 的 docstring 逐个实现；热点循环写成编译器容易自动向量化的形式（之后可以考虑 NEON 和 SSE 的 intrinsics，但不是这个 ticket 的要求）
- `core/src/fusion/`：规则 DSL 解释器（按 `schema/rules_dsl.md`），读取 `assets/rules.json`；NaN 语义必须和 Python 版本一致
- `core/src/templates/`：读取 `templates_zh` 并做占位符替换
- 对齐测试（GoogleTest + Python 两边）：
  - `schema/fixtures/signals_golden/` 每个指标的相对误差 ≤ 1%
  - 规则和模板：输入相同的特征，输出完全一致

## 验收标准
- [ ] 对齐测试全部通过
- [ ] Linux 单核上 1024px 输入的 S2 总耗时 ≤ 40 ms（作为端侧的参考）
