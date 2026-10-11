# AGENTS.md — 给 coding agent 的工作规则

本仓库是 **PhotoCritic**：一个离线的手机"摄影点评" App，同时也是学习端侧模型的项目。C++ 跨平台核心（ONNX Runtime + llama.cpp）+ 很薄的平台外壳（先做 iOS，以后做 Android）。

## 1. 开工前按顺序阅读

1. [`docs/DESIGN_SUMMARY.md`](docs/DESIGN_SUMMARY.md)：设计全貌、决策记录、接口和执行清单（**必读**）
2. [`docs/tickets/README.md`](docs/tickets/README.md)：ticket 列表、依赖关系和状态
3. 你要做的那个 ticket 文件，以及它引用的契约文件（`schema/`、`assets/rules.yaml`）
4. [`docs/SPEC.md`](docs/SPEC.md)：背景和目标（细节以前面几份为准）
5. [`docs/VERSIONS.md`](docs/VERSIONS.md)：固定的工具、库和模型版本

**文档优先级**：契约文件（`schema/`、`assets/rules.yaml`）> `DESIGN_SUMMARY.md` > `tickets/` > `SPEC.md`。发现冲突时以优先级高的为准，并在同一次提交里修正优先级低的文档。

## 2. 工作方式

1. **一次只做一个 ticket。** 开始前确认它的依赖都是"已完成"状态；没有完成就停下来告诉用户。
2. **满足验收标准就停下。** 把 ticket 的状态改成"已完成"，用几句话汇报做了什么、验收标准每一项的结果（包括没通过的），然后等用户决定下一步。不要顺手开始下一个 ticket，也不要做 ticket 范围以外的"改进"。
3. **不确定的事实不要猜。** 文档里标了"（待核实）"的地方，以及所有第三方 API（尤其是 llama.cpp 的 `llama.h`、`mtmd.h`、`mtmd-helper.h`，以及 ONNX Runtime 的 C/C++ API），都要先读固定版本的头文件、源码或官方文档再写代码。核实结果写进 `docs/decisions/` 或 `docs/VERSIONS.md`。无法核实时问用户。
4. **测试是交付物的一部分。** 每个 ticket 都要带上测试；宣布完成前在本地跑通。测试失败就如实说明，不要删测试或放宽断言来让它通过。
5. **选型和取舍写 ADR**：`docs/decisions/NNNN-<主题>.md`（背景、选项、决定、后果），模板见 `0000-template.md`。
6. **设计有变化时同步文档**：改了接口、格式或流程，就在同一次提交里更新 DESIGN_SUMMARY 和相关 ticket。

## 3. 红线（违反任何一条都要先问用户）

1. **分层**：分析逻辑（阈值、规则、算法、prompt 拼装）只能写在 `core/`（C++）或 `ml/`（Python）里。Swift 和 Kotlin 只负责 UI、图片解码、相册、文件和生命周期。
2. **核心可移植**：`core/` 里不能 include 平台头文件，不能依赖 OpenCV，不能用 Apple Vision、Core Image、Foundation Models、ML Kit 等平台专有的 AI 能力。
3. **契约不能悄悄改**：修改 `schema/` 下的任何文件，都必须升对应的版本号、同步 fixtures 和两端实现，并事先征得用户同意。
4. **规则阈值不能拍脑袋改**：`assets/rules.yaml` 里的数值只能通过调参（T-008、T-021）修改，提交信息里要注明对应的评测报告。
5. **C API 以 DESIGN_SUMMARY §4 为准**：要修改时先改文档并说明理由。
6. **大文件不进 git**：模型、数据集、构建产物都不提交。模型的来源、文件名和 SHA256 记在 `docs/VERSIONS.md`，下载写进 `scripts/fetch-models.sh`。
7. **数据外发要先问**：把图片或数据发给任何云端服务（LLM-as-judge、teacher 模型等）之前，先征得用户同意。

## 4. 需要用户本人完成的步骤

agent 负责写好代码、脚本和操作说明，下面这些由用户执行，再把结果（CSV、JSON、日志、截图）交回给 agent：
- 在 iPhone 真机上运行和操作（T-000、T-017~T-019），以及 Android 真机（T-020）
- Apple 开发者账号、签名、provisioning profile、entitlement（例如 `increased-memory-limit`）的配置
- 接受模型或数据集的许可证、需要登录的下载
- 在 Linux 训练机上的任何操作（那台机器上**不要安装工具链**）

## 5. 约定

- **语言**：文档用中文；代码里的标识符和注释用英文；面向用户的文案（建议模板、prompt）用中文。
- **提交信息**：英文，加前缀 `docs:` / `schema:` / `core:` / `ml:` / `ios:` / `android:` / `spike:` / `build:`。只在用户要求时 push。
- **构建和测试命令**：由 T-001 补充到这里。
- **知识沉淀**：对话中产生的可复用知识（原理、用法、踩坑、选型结论）蒸馏后追加到用户的 `LEARN.md`（Linux 上是 `~/CX/LEARN.md`；Mac 上的位置问用户）。写之前先读，已有同类条目就更新，不要重复追加。格式：
  ```markdown
  ## [YYYY-MM-DD] 主题
  **标签**: tag1, tag2
  一句话核心结论。
  - 要点
  ```
