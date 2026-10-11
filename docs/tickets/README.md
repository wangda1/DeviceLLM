# Tickets

> 在 Mac 上开工前先读 [../DESIGN_SUMMARY.md](../DESIGN_SUMMARY.md)，特别是 §12（T-000 执行清单）和 §13（Mac 上的开发顺序）。

每个 ticket 是一个可以独立交付的工作单元，开头标明**运行环境**（🐧 Linux / 🍎 Mac / Android 真机）、**依赖**和**规模**（S ≤ 半天，M ≤ 2 天，L ≤ 1 周）。
领取规则：只有依赖全部完成的 ticket 才能开始；完成时要满足"验收标准"里的每一项，并把学到的要点写进 `../../../LEARN.md`。

**分层红线**（每个 ticket 的 review 都要检查）：分析逻辑只能写在 `core/`（C++）或 `ml/`（Python 原型）里；平台外壳只负责 UI、图片解码、相册和文件访问。

## 里程碑总览

### M-1 真机可行性（最先做）
| ID | 标题 | 环境 | 依赖 | 规模 |
|---|---|---|---|---|
| [T-000](T-000-device-feasibility-spike.md) | 真机可行性 spike：iPhone 16 Pro 上的小 VLM | 🍎 + iPhone | — | M |

### M0 基础设施
| ID | 标题 | 环境 | 依赖 | 规模 |
|---|---|---|---|---|
| [T-001](T-001-repo-scaffold.md) | 仓库骨架：Python 环境 + C++ 核心骨架 | 🐧 | — | S |
| [T-002](T-002-report-schema.md) | 输出契约：report schema 与标签体系 | 🐧 | T-001 | S |
| [T-003](T-003-eval-datasets.md) | 评测数据集 v0（合成退化 + SPAQ/PARA + golden） | 🐧 | T-001 | M |
| [T-004](T-004-eval-harness.md) | 评测 harness | 🐧 | T-002, T-003 | M |

### M1 零样本基线（Python 原型）
| ID | 标题 | 环境 | 依赖 | 规模 |
|---|---|---|---|---|
| [T-005](T-005-signal-metrics.md) | S2 信号指标（Python 原型 + 规范） | 🐧 | T-002 | M |
| [T-006](T-006-clip-zeroshot.md) | S3 MobileCLIP 零样本打分与 ONNX 导出 | 🐧 | T-002, T-004 | M |
| [T-007](T-007-detectors.md) | S1 检测模型：显著性与人脸（ONNX） | 🐧 | T-002, T-004 | M |
| [T-008](T-008-fusion-rules.md) | S4 融合规则与调参 | 🐧 | T-005, T-006, T-007 | M |
| [T-009](T-009-vlm-selection.md) | VLM 质量选型（llama.cpp） | 🐧 | T-000, T-004 | M |
| [T-010](T-010-vlm-prompt-grammar.md) | S5 prompt、GBNF grammar 与模板兜底 | 🐧 | T-008, T-009 | M |
| [T-011](T-011-e2e-baseline.md) | 端到端参考 pipeline 与 M1 基线报告 | 🐧 | T-010 | S |

### M2a C++ 跨平台核心（全部在 Linux 上完成）
| ID | 标题 | 环境 | 依赖 | 规模 |
|---|---|---|---|---|
| [T-012](T-012-core-architecture.md) | 核心架构、C API 与 ONNX Runtime 集成 | 🐧 | T-002 | M |
| [T-013](T-013-core-signals-fusion.md) | 核心：信号指标、融合规则与模板 | 🐧 | T-005, T-008, T-010, T-012 | L |
| [T-014](T-014-core-models.md) | 核心：检测模型与 MobileCLIP 推理 | 🐧 | T-006, T-007, T-012 | M |
| [T-015](T-015-core-llama.md) | 核心：llama.cpp 多模态集成 | 🐧 | T-009, T-012 | L |
| [T-016](T-016-core-e2e.md) | 核心端到端评测（和 Python 原型对齐） | 🐧 | T-011, T-013, T-014, T-015 | S |

### M2b 平台外壳
| ID | 标题 | 环境 | 依赖 | 规模 |
|---|---|---|---|---|
| [T-017](T-017-ios-shell.md) | iOS 外壳：xcframework 构建与 Swift 封装 | 🍎 | T-012（完整验收需要 T-016） | M |
| [T-018](T-018-results-ui.md) | iOS 结果页 UI 与叠加层 | 🍎 | T-017 | M |
| [T-019](T-019-device-benchmark.md) | iOS 真机基准测试与跨平台对齐验收 | 🍎 | T-018 | M |
| [T-020](T-020-android-portability.md) | Android 可移植性验证（NDK + 命令行） | 🐧 + Android | T-016 | M |

### M3 微调与量化
| ID | 标题 | 环境 | 依赖 | 规模 |
|---|---|---|---|---|
| [T-021](T-021-train-scorer-heads.md) | 在 MobileCLIP 特征上训练多头打分器 | 🐧 | T-016 | L |
| [T-022](T-022-vlm-lora-distill.md) | VLM 建议生成：蒸馏数据与 LoRA 微调 | 🐧 | T-016 | L |
| [T-023](T-023-quant-ablation.md) | 量化消融（分平台） | 🐧 + 🍎 + Android | T-019, T-020, T-022 | M |

## 依赖图

```
T-001 ─┬─ T-002 ─┬─ T-005 ──────────────────────┐
       │         └─ T-012 ──────────┬───────────┼──────────────┬─ T-017 (🍎, 可提前搭骨架)
       └─ T-003 ── T-004 ─┬─ T-006 ─┤           │              │
                          ├─ T-007 ─┴─ T-008 ───┤              │
                          │      (T-006,T-007)  │              │
                          │          └─ T-014   │              │
  T-000 (🍎 真机) ───────┴─ T-009 ─┬─ T-010 ◄──┘(T-008)       │
                                    │    └─ T-011              │
                                    └─ T-015 ◄── T-012         │
   T-005+T-008+T-010+T-012 ─ T-013                             │
   T-011+T-013+T-014+T-015 ─ T-016 ─┬─ T-017 完整验收 ─ T-018 ─ T-019 ─┐
                                    ├─ T-020 (Android) ────────────────┼─ T-023
                                    ├─ T-021                           │
                                    └─ T-022 ──────────────────────────┘
```

## 并行建议
- **T-000 最先做**（Mac + iPhone 16 Pro，1–2 天），可以和 Linux 上的 T-001~T-003 同时进行。它的结论决定 S5 的模型档位和 SPEC §6 的预算。
- 除了 T-000，绝大部分工作都在 **Linux** 上，Mac 主要在 T-017 之后介入。
- **第一条线（研究）**：T-001 → T-002/T-003 → T-004 → T-005/T-006/T-007/T-009 → T-008 → T-010 → T-011。
- **第二条线（工程）**：T-002 完成后马上开始 T-012，随后 T-015（llama.cpp 集成只依赖 T-009 的选型结果）。
- **T-009（VLM 质量选型）** 在 T-000 和 T-004 都完成后马上开始。
- **T-020（Android）** 是可移植性的"守门员"，T-016 完成后尽早做，越晚发现问题，返工越大。
