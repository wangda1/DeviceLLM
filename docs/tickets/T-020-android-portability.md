# T-020 Android 可移植性验证（NDK 交叉编译 + 命令行）
**环境**: 🐧 Linux + 一台 Android 真机 · **依赖**: T-016 · **规模**: M · **里程碑**: M2b · **状态**: 待开始

## 背景
v1 不做 Android App，但要尽早证明核心确实可以移植，防止 iOS 的假设悄悄混进核心。

## 交付物
- `scripts/build-android.sh`：NDK 交叉编译出 arm64-v8a 的核心库和 `photocritic-cli`（llama.cpp 用 CPU 后端，可选尝试 Vulkan；ORT 用 Android 包，带 XNNPACK EP）
- 通过 adb push 把 CLI、模型和 assets 拷到设备上，跑 golden 子集，结果拷回 Linux 用 harness 比较
- 记录 Android 的延迟和内存（CPU / XNNPACK / NNAPI 各 EP，llama.cpp CPU vs Vulkan），据此确定 Android 的性能预算
- 在 CI（或本地脚本）中加入 Android 交叉编译检查，保证以后的提交不会破坏可移植性
- `docs/decisions/0005-android-runtime.md`：Android 外壳的建议方案（Kotlin + JNI 封装、模型分发方式）

## 验收标准
- [ ] Android 上 S4 结果和 Linux 一致（标签一致率 ≥ 98%）
- [ ] 得到一份 Android 的性能数据和预算建议
