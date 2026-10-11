# T-012 C++ 核心：架构、C API 与 ONNX Runtime 集成
**环境**: 🐧 Linux · **依赖**: T-002 · **规模**: M · **里程碑**: M2a
**可以在 M1 期间并行开始**

## 背景
跨平台的地基，见 SPEC §3 和 §4 的 C API 草案。这一步定下模块边界和对外接口，具体算法在后续 ticket 里填充。

## 交付物
- `core/include/photocritic.h`：按 SPEC §4 定义的 C API，写清楚线程安全、内存所有权（谁分配、谁释放）和错误码
- 内部模块接口：`Preprocess`、`Signals`、`Detectors`、`ClipScorer`、`Fusion`、`Vlm`、`Templates`，先用 stub 实现
- `pc_image` / `pc_exif` 结构体；`config_json` 包含模型和 assets 路径、ONNX EP 选择、线程数、S5 超时
- 第三方依赖：ONNX Runtime（Linux 用预编译包；记录 iOS 和 Android 包的获取方式）、nlohmann/json、一个 YAML 或直接用 JSON（规则在构建时已转成 JSON，核心里只需要解析 JSON）
- `core/src/ort_session.*`：ORT 会话的封装，EP 可以配置（cpu / cuda / coreml / xnnpack / nnapi），在 report 的 `runtime` 字段里记录实际使用的 EP
- S0 预处理：area 缩放，和 T-005 的 Python 实现一致
- `tools/cli/photocritic-cli`：用 stb_image 解码，调用 C API，输出 JSON
- nanobind 绑定：`photocritic_core.Engine(config).analyze(np_array, exif_dict) -> dict`
- ADR `docs/decisions/0004-core-architecture.md`：为什么选 C++ + C API + JSON 边界，为什么选 ORT 而不是 ExecuTorch / TFLite / MNN

## 验收标准
- [ ] CLI 和 Python 绑定都能端到端跑通 stub pipeline，输出符合 schema 的 JSON
- [ ] 用 AddressSanitizer 跑测试没有报错
- [ ] 核心代码不出现任何平台相关的头文件（CI 中用 grep 检查）
