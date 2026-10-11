# T-015 C++ 核心：llama.cpp 多模态集成
**环境**: 🍎 macOS（也可以在 Linux 上做） · **依赖**: T-009, T-012 · **规模**: L · **里程碑**: M2a · **状态**: 待开始

## 背景
项目里最核心的端侧推理学习点。在 Linux 上把 llama.cpp 的 C API 用熟，iOS 和 Android 只需要换编译后端（Metal / CPU / Vulkan）。

## 交付物
- llama.cpp 作为 git 子模块放在 `core/third_party/`，固定 commit；CMake 选项可以开关 CUDA、Metal、Vulkan
- `core/src/vlm/`：
  - 模型和 mmproj 的加载与卸载（使用 mmap）、上下文、采样器；grammar 按 T-010 的规则**在运行时生成**，和 Python 版本的 golden 文件对齐
  - **先读固定 commit 的 `llama.h`、`mtmd.h`、`mtmd-helper.h` 和 `tools/mtmd/mtmd-cli.cpp`，再写代码**；不要凭记忆使用 API
  - 多模态输入：通过 mtmd API 把图像编码成 embedding，再和文本交错拼接；findings 按 T-010 的规范序列化
  - 实现 `pc_suggest`：在后台线程运行，通过回调流式输出 token；支持取消和超时；超时或出错时调用模板兜底，并把 `generator` 设为 `template`
  - 内存：提供 `pc_engine_release_vlm()`，平台层收到内存警告时可以调用
- 对齐：在 Linux 上用同一个 GGUF、同样的 seed 和 greedy 采样，输出和 llama-server 一致

## 验收标准
- [ ] CLI 能输出流式建议；在 golden 集上 schema 合法率 100%
- [ ] 连续推理 50 次，用 ASan/LSan 或 valgrind 检查没有内存泄漏
- [ ] 取消和超时在 100 ms 内生效
