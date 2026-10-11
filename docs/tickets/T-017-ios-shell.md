# T-017 iOS 外壳：xcframework 构建与 Swift 封装
**环境**: 🍎 Mac · **依赖**: T-012 · **规模**: M · **里程碑**: M2b · **状态**: 待开始
**工程骨架在 T-012 完成后就可以开始；完整验收需要 T-016**

## 背景
iOS 是第一个外壳，**只做**：构建核心、解码图片、调用 C API、展示结果。任何分析逻辑都不能写在 Swift 里。

## 交付物
- `scripts/build-ios-xcframework.sh`：把核心、llama.cpp（Metal）和 ONNX Runtime（iOS 包，带 CoreML EP）打包成 `PhotoCriticCore.xcframework`
- `platforms/ios/PhotoCritic.xcodeproj`：SwiftUI，iOS 26 deployment target；加上 increased-memory-limit entitlement
- Swift 封装 `PhotoCriticKit`：
  - 封装 `pc_engine`；用 `Codable` 解码报告 JSON（和 `report.schema.json` 对应）
  - `pc_suggest` 的回调桥接成 `AsyncThrowingStream`
  - 图片输入：用 ImageIO 解码 HEIC/JPEG，应用 EXIF 方向，Display P3 转 sRGB，转成 RGBA8 缓冲区；提取 EXIF 字段
  - 收到内存警告时调用 `pc_engine_release_vlm()`
- 模型和 assets：开发期打进 bundle 或者用 devicectl 拷到沙盒；正式版按需下载（先写个设计草稿即可）
- App：PhotosPicker 选图 → 分析 → 结果页（先展示原始 JSON）；开发调试页可以选择 ORT 的 EP（CPU / CoreML）

## 验收标准
- [ ] 真机上端到端跑通（S0–S5）
- [ ] 同一张 PNG 输入，iOS 和 Linux 的 S4 结果一致
- [ ] Swift 代码中没有任何分析逻辑（code review 时检查）
