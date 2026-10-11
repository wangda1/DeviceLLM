# T-002 输出契约：自动化校验
**环境**: 🍎 macOS · **依赖**: T-001 · **规模**: S · **里程碑**: M0 · **状态**: 待开始

## 背景
契约文件**已经写好**（2026-10-11），本 ticket 的工作是把它们变成自动化测试，保证以后任何改动都不会让契约悄悄失配。已有的文件：
- `schema/report.schema.json`、`schema/vlm_output.schema.json`
- `schema/taxonomy.yaml`、`schema/features.md`、`schema/rules_dsl.md`
- `schema/fixtures/report/{valid,invalid}/`、`schema/fixtures/vlm_output/{valid,invalid}/`
- `assets/rules.yaml`

发现契约本身有问题时，**先记录到 `docs/decisions/` 并询问用户**，不要直接修改；如果确实要修改，必须升 `schema_version`。

## 交付物
- `ml/photocritic/report.py`：和 `report.schema.json` 对应的 pydantic 模型
- `ml/tests/test_contract.py`，至少检查：
  1. 两个 JSON Schema 本身合法（draft 2020-12）
  2. `valid/` 下的示例全部通过校验，`invalid/` 下的全部失败
  3. pydantic 模型能对 `valid/` 示例做 decode/encode 往返，结果不变
  4. `taxonomy.yaml` 的标签列表（**包括顺序**）和 schema 的 `$defs.tag` 完全一致；场景列表同理
  5. `rules.yaml` 恰好为每个标签提供一条规则，`dimension` 和 taxonomy 一致
  6. `rules.yaml` 里引用的特征（`f:`、`evidence`）全部出现在 `features.md` 中（`clip_<tag>`、`scene_<name>` 按 taxonomy 展开）
  7. `piecewise` 的 x 严格单调；`thresholds` 的顺序和 `dir` 一致；`scores.weights` 覆盖所有标签；`suppress` 和 `intentional` 只引用存在的标签
  8. `valid/` 示例里 issue 的 tag 和 dimension 对应正确，evidence 的键都在 `features.md` 中
- 这些检查加入 CI（或 `scripts/check.sh`），每次提交都要跑

## 验收标准
- [ ] 以上 8 项检查全部实现并通过
- [ ] 故意改坏任一契约文件（例如删除一个标签），至少有一项测试会失败
