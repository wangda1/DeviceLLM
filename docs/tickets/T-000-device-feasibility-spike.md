# T-000 真机可行性 spike：iPhone 16 Pro 上的小 VLM
**环境**: 🍎 Mac + iPhone 16 Pro（可以在 Linux 上先跑一遍做准备） · **依赖**: — · **规模**: M（1–2 天） · **里程碑**: M-1
**⚠️ 最先做：结论决定 SPEC §6 的预算和 S5 的模型档位**

## 背景
最大的未知数是端侧 VLM 的内存、延迟和发热。在投入 C++ 核心之前，先用最少的代码在真机上拿到数据。**这是一次性代码，不进入 `core/`**，放在 `spikes/t000/`。

## 范围
- 不写业务逻辑；prompt 使用一份固定的"findings → 中文建议 JSON"示例（约 150 个输入 token，要求输出约 150–200 个 token）
- 测试图片：5 张不同的照片，统一缩放到 512px 和 768px 两档

## 交付物
1. **Linux 准备**（可选，0.5 天）：用 `llama-mtmd-cli` 在 Linux CPU 上跑通下面全部候选模型，确认 GGUF 和 mmproj 能加载、中文输出基本可读，淘汰明显不行的模型
2. **最小 iOS demo**：
   - 用 llama.cpp 的 `build-xcframework.sh` 构建（Metal 开启），**确认 xcframework 里包含多模态模块（mtmd）**；如果没有，记录需要怎么改构建脚本
   - 一个按钮：加载模型 → 编码图片 → 生成，把各项指标打印到屏幕上并导出 CSV
3. **测试矩阵**（每个组合跑 3 次，取中位数）：

| 档位 | 模型 | 量化 |
|---|---|---|
| 默认档 | LFM2.5-VL-450M | Q4_0 / Q8_0；mmproj q8_0 / f16 |
| 默认档 | Qwen3.5-0.8B | Q4_K_M；mmproj f16 |
| 默认档 | InternVL3-1B | Q8_0；mmproj q8_0 |
| 高质量档 | Qwen3-VL-2B | Q4_K_M；mmproj q8_0 |
| 上限参照 | Qwen2.5-VL-3B | Q4_K_M；mmproj q8_0 |
| 对照组 | LFM2.5-350M 纯文本（不输入图片） | Q4_0 |

4. **每个组合要测的指标**：
   - 模型加载时间（冷启动和热启动）
   - 图像 token 数、图像编码耗时、prefill 耗时、首 token 耗时
   - decode tok/s；生成 200 tokens 的总耗时
   - 内存：`os_proc_available_memory()` 在加载前后和推理中的变化、`phys_footprint` 峰值；**分别在申请和不申请 increased-memory-limit 权限的情况下**测出 App 可用内存的上限
   - 温控：每种模型连续跑 20 次，记录 tok/s 曲线和 `ProcessInfo.thermalState` 的变化
   - 推理过程中切到后台：观察 Metal 报什么错，App 能不能恢复
5. **报告** `docs/decisions/0000-device-feasibility.md`：
   - 测试数据表；和 SPEC §6 的预算逐项对比
   - 结论：默认档和高质量档分别选哪个模型；§6 的预算定稿（直接提交对 SPEC 的修改）
   - 顺带记录：开发者账号类型对真机调试和 increased-memory-limit 权限的限制

## 验收标准
- [ ] 至少 1 个默认档候选在 16 Pro 上达到 §6 的默认档预算；如果没有一个达标，报告里要给出调整方案（降分辨率、缩短输出、改用纯文本模型等）
- [ ] SPEC §6 已按实测数据更新
- [ ] 把真机实测数据和估算的差距写进 LEARN.md（修正估算方法）
