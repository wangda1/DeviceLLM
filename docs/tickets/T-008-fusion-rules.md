# T-008 S4 融合规则与调参
**环境**: 🐧 Linux · **依赖**: T-005, T-006, T-007 · **规模**: M · **里程碑**: M1 · **状态**: 待开始

## 背景
把 S1 + S2 + S3 的特征转成分数和标签，见 SPEC §4 S4。规则必须可以解释（`evidence` 字段），并且由 C++ 核心直接执行同一份规则文件。

## 交付物
- `assets/rules.yaml` **已有起始版本**（23 个标签的规则、压制关系、有意为之、分数和裁剪配置），阈值都是占位值；本 ticket 负责调参并修正规则逻辑
- DSL 的语法和执行语义**已经定义**在 `schema/rules_dsl.md`，按它实现解释器，不要扩展语法；确实需要扩展时，先改 `rules_dsl.md`（升 `version`）并说明理由
- `ml/photocritic/fusion.py`：规则解释器（Python 版），严格按 `rules_dsl.md` §10 的执行顺序；单元测试要覆盖 NaN 语义、scene override、suppress、intentional、分数公式；用 `schema/fixtures/report/valid/01_full_vlm.json` 中的特征值反算，分数应该和示例一致
- `intentional_possible` 的判断逻辑（场景标签加上特定的特征组合，例如剪影：主体全黑但边缘很清楚）
- 调参脚本：在 synthetic dev 集上网格搜索或贝叶斯优化阈值，以宏平均 F1 为目标，结果写回 `rules.yaml`
- `suggested_crop`：按 `rules_dsl.md` §9 实现

## 验收标准
- [ ] `rules_only` pipeline（S1 + S2 + S3 + S4）在 synthetic test 集上给出基线宏平均 F1
- [ ] 每个标签的 `evidence` 都能指出是哪条规则、哪个特征触发了它
- [ ] 规则文件只使用 DSL 中定义的语法
