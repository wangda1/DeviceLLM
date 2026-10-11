# T-023 量化消融：质量与延迟、内存的权衡
**环境**: 🐧 + 🍎 + Android · **依赖**: T-019, T-020, T-022 · **规模**: M · **里程碑**: M3

## 交付物
- VLM 量化矩阵：LLM 用 Q3_K_M / Q4_0 / Q4_K_M / Q5_K_M / Q8_0；mmproj 用 f16 / q8_0；可选尝试 imatrix 量化。注意 Q4_0 在 ARM CPU 上有专门的重排加速，对 Android 可能更有利
- 小模型量化：ONNX 的 FP32 / FP16 / INT8（动态量化和静态量化）
- 每个组合：在 Linux 上测质量（judge 分数、F1），在 iOS 和 Android 真机上测首 token 耗时、tok/s、峰值内存（用 T-019 和 T-020 的 benchmark 流程）
- 画出 Pareto 图（质量 vs 延迟，气泡大小代表内存），**每个平台一张**
- 更新各平台的默认配置（可以不同）

## 验收标准
- [ ] 满足 SPEC §7.3 M3 的通过标准
- [ ] 把量化格式的原理和取舍写入 LEARN.md
