# T-001 仓库骨架：Python 环境 + C++ 核心骨架
**环境**: 🍎 macOS（Apple Silicon） · **依赖**: — · **规模**: S · **里程碑**: M0 · **状态**: 待开始

## 背景
按照 SPEC §8 和 DESIGN_SUMMARY §8 建好目录。Python 用于研究和评测，C++ 核心是最终跨平台交付的代码，两边从第一天起就要能构建和测试。`schema/`、`assets/rules.yaml` 和 `docs/` 已经存在，**不要改动它们的内容**。

## 交付物
- 补齐 DESIGN_SUMMARY §8 的目录结构，加上根目录 `README.md`（项目简介、"一个核心，多个外壳"的分层原则、各平台的构建入口、文档索引）
- Python：`ml/pyproject.toml`（uv），基础依赖：numpy、pillow、pillow-heif、onnxruntime、pydantic v2、jsonschema、pyyaml、pytest、ruff；opencv-python-headless 只能用在研究和对照代码里，不能出现在 `ml/photocritic/` 的规范实现中；**提交 `uv.lock`**
- C++：`core/CMakeLists.txt`（C++17，CMake ≥ 3.24）、GoogleTest（FetchContent，固定 tag）、`include/photocritic.h` 先只放 `pc_version()`、`bindings/python`（nanobind，固定版本）把它暴露给 Python；`scripts/build-macos.sh`
- 一个构建步骤：`assets/rules.yaml` → `assets/rules.json`（Python 脚本，CMake 和 Python 测试都调用它；生成的 JSON 不提交）
- `.clang-format`；`.gitignore` 忽略数据集、模型权重（`*.gguf`、`*.onnx`、`*.safetensors`）、构建目录、`spikes/*/models/`、`rules.json`
- `docs/decisions/0000-template.md`：ADR 模板
- `docs/VERSIONS.md`：填写本 ticket 引入的工具和库的版本（见验收标准）

## 验收标准
- [ ] `cd ml && uv sync && uv run pytest` 通过
- [ ] `scripts/build-macos.sh && ctest --test-dir build` 通过
- [ ] Python 里 `import photocritic_core; photocritic_core.version()` 能运行
- [ ] `docs/VERSIONS.md` 中填好：macOS、Xcode、Apple Clang、CMake、Python、uv、GoogleTest、nanobind 的版本
- [ ] 只提交骨架，没有任何业务逻辑
