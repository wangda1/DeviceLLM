# T-010 S5 prompt、GBNF grammar 与模板兜底
**环境**: 🐧 Linux · **依赖**: T-008, T-009 · **规模**: M · **里程碑**: M1 · **状态**: 待开始

## 背景
让 VLM 稳定输出符合 schema 的中文建议，同时实现不依赖 VLM 的模板兜底，见 SPEC §4 S5 和 S5'。所有产物都放在 `assets/`，C++ 核心直接复用。

## 交付物
- `assets/prompts/vlm_system.md`：角色设定、输出要求（可以照着做、给出参数、不重复检测结果、对"有意为之"的照片要谨慎）
- findings 序列化格式：按 DESIGN_SUMMARY §3 S5 的示例（中文纯文本，bbox 转成九宫格位置描述），写成规范和 golden 文件，C++ 端照着实现
- **grammar 在运行时生成**：输出格式固定为 `schema/vlm_output.schema.json` 的短 key JSON；`t` 的枚举 = 本次检测到的标签 ∪ {"other"}，`h` ≤ 12 字、`a` ≤ 60 字、最多 3 条。实现 `build_grammar(tags) -> str`（Python 版），生成的 grammar 写成 golden 文件，供 C++ 版本做对齐测试。可以先用 llama.cpp 的 `json_schema_to_grammar.py` 生成一个模板再精简
- `ml/photocritic/vlm.py`：通过 llama-server 的 HTTP API 调用（请求中带上运行时生成的 grammar）；支持流式输出和超时；把短 key 输出展开成 `report.suggestions[]`（`source: vlm`，`params` 由模板计算）
- `assets/templates_zh.yaml` 和 `ml/photocritic/templates.py`：每个标签的建议模板，参数从信号指标计算（例如 `ev_delta` 根据 `mean_L` 推算，色温调整量根据 CCT 推算）；模板只用简单的占位符替换，不依赖任何模板引擎
- 实现 LLM-as-judge 评分（评分细则版本化），接入 T-004 的 harness

## 验收标准
- [ ] golden 集上 VLM 输出的 schema 合法率达到 100%
- [ ] 模板兜底覆盖全部标签
- [ ] judge 分数：VLM 和模板各有一个基线数字
