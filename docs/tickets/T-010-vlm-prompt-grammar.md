# T-010 S5 prompt、GBNF grammar 与模板兜底
**环境**: 🐧 Linux · **依赖**: T-008, T-009 · **规模**: M · **里程碑**: M1

## 背景
让 VLM 稳定输出符合 schema 的中文建议，同时实现不依赖 VLM 的模板兜底，见 SPEC §4 S5 和 S5'。所有产物都放在 `assets/`，C++ 核心直接复用。

## 交付物
- `assets/prompts/vlm_system.md`：角色设定、输出要求（可以照着做、给出参数、不重复检测结果、对"有意为之"的照片要谨慎）
- findings 序列化格式：精简的文本或 JSON，控制 token 数；格式写成规范，C++ 端照着实现
- `assets/suggestions.gbnf`：只约束 `suggestions[]` 部分，从 JSON Schema 生成（可以用 llama.cpp 自带的 `json_schema_to_grammar.py`）并手动精简
- `ml/photocritic/vlm.py`：通过 llama-server 的 HTTP API 调用；支持流式输出和超时
- `assets/templates_zh.yaml` 和 `ml/photocritic/templates.py`：每个标签的建议模板，参数从信号指标计算（例如 `ev_delta` 根据 `mean_L` 推算，色温调整量根据 CCT 推算）；模板只用简单的占位符替换，不依赖任何模板引擎
- 实现 LLM-as-judge 评分（评分细则版本化），接入 T-004 的 harness

## 验收标准
- [ ] golden 集上 VLM 输出的 schema 合法率达到 100%
- [ ] 模板兜底覆盖全部标签
- [ ] judge 分数：VLM 和模板各有一个基线数字
