# T-002 输出契约：report schema 与标签体系
**环境**: 🐧 Linux · **依赖**: T-001 · **规模**: S · **里程碑**: M0

## 背景
Python、C++ 核心和各平台外壳共用的唯一数据契约，见 SPEC §5。这里一旦定下来，后面的修改都要升 `schema_version`。

## 交付物
- `schema/report.schema.json`（JSON Schema draft 2020-12），覆盖 SPEC §5 的全部字段；`tag` 和 `dimension` 用 enum 约束
- `schema/taxonomy.yaml`：每个标签的中文名、所属维度、一句话定义、典型的"有意为之"场景
- `schema/fixtures/`：至少 3 个合法样例（正常、多问题、`skip_reason=utility`）和 2 个非法样例
- `ml/photocritic/report.py`：对应 schema 的 pydantic 模型（C++ 端直接按 JSON 处理，不另写结构体），再加一个测试检查它和 JSON Schema 是否一致

## 验收标准
- [ ] 合法样例全部通过校验，非法样例全部被拒绝
- [ ] pydantic 模型导出的 schema 和 `report.schema.json` 在语义上等价（有测试覆盖）
- [ ] taxonomy 覆盖 SPEC §5 表格里的全部标签
