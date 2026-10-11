# T-001 仓库骨架：Python 环境 + C++ 核心骨架
**环境**: 🐧 Linux · **依赖**: — · **规模**: S · **里程碑**: M0

## 背景
按照 SPEC §8 建好目录。Python 用于研究和评测，C++ 核心是最终跨平台交付的代码，两边从第一天起就要能构建和测试。

## 交付物
- 按 SPEC §8 建好目录结构，加上 `README.md`（项目简介、"一个核心，多个外壳"的分层原则、各平台的构建入口）
- Python：`ml/pyproject.toml`（uv），基础依赖：numpy、pillow、pillow-heif、opencv-python-headless（只在研究侧使用）、onnxruntime、pydantic v2、jsonschema、pytest、ruff
- C++：`core/CMakeLists.txt`（C++17）、GoogleTest、`include/photocritic.h` 里的空 API（`pc_version()`）、`bindings/python`（nanobind）把 `pc_version` 暴露给 Python；`scripts/build-linux.sh`
- `.clang-format`；`.gitignore` 忽略数据集、模型权重（`*.gguf`、`*.onnx`、`*.safetensors`）和构建目录
- `docs/decisions/0000-template.md`：ADR 模板

## 验收标准
- [ ] `cd ml && uv sync && uv run pytest` 通过
- [ ] `scripts/build-linux.sh && ctest` 通过
- [ ] Python 里 `import photocritic_core; photocritic_core.version()` 能运行
- [ ] 第一个 commit 只包含骨架
