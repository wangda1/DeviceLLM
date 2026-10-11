# PhotoCritic 端侧图片质量分析 — Spec v0.4

> 状态：Draft，2026-10-11 · 交接与实现细节见 [DESIGN_SUMMARY.md](DESIGN_SUMMARY.md)
> 定位：一个**学习端侧模型**的项目。业务目标是做一个能在手机上离线运行的"摄影建议"App，先做 iOS，后期扩展到 Android 等其他端侧平台；学习目标是走一遍从模型选型、评测、量化、转换、跨平台部署到微调的完整链路。
>
> v0.4 变更：数据契约落成实际文件（`schema/`、`assets/rules.yaml`），§5 改为文件索引；C API 以 DESIGN_SUMMARY §4 为准；VLM 改为短 key 输出，grammar 在运行时生成。
>
> v0.3 变更：VLM 改为分档，默认用 1B 以下的小模型（§4 S5）；性能预算按档位重设，以 iPhone 16 Pro 为基准机型（§6）；新增 T-000 真机可行性 spike。
>
> v0.2 变更：**尽量减少对平台原生能力的依赖**。去掉 Apple Vision 和 Foundation Models；推理逻辑全部放进一个跨平台的 C++ 核心（`core/`），平台层只负责 UI、图片解码和相册访问。

---

## 1. 目标与非目标

### 1.1 目标
1. 用户从相册里选一张照片，App **完全离线**地输出：
   - 4 个维度的分数（0–100）：曝光、构图与主体、色彩、清晰度
   - 问题标签（带置信度和严重程度），例如"过曝"、"主体不突出"、"水平线倾斜"、"偏暖"
   - 可以照着做的**中文摄影建议**（例如"曝光补偿 −0.7EV"、"把主体放到右侧三分线"、"白平衡调冷约 500K"）
   - 可视化叠加层：显著性热区、水平线、建议裁剪框
2. **可移植**：分析能力全部在 C++ 核心里实现，只依赖可移植的开源组件（llama.cpp、ONNX Runtime）。核心可以在 Linux、iOS、Android 上编译运行，结果一致。
3. 在 Linux 上通过 Python 绑定直接调用**同一份 C++ 核心**，做离线评测和回归；Python 原型只用于研究和调参。
4. 有可以量化的评测体系，每次改动都能拿到数字（P/R/F1、SRCC、延迟、内存）。

### 1.2 非目标（v1 不做）
- 实时取景器提示和视频分析
- 批量扫描相册
- 自动修图并导出（只给建议和裁剪框预览，不改原图）
- 云端推理和账号体系
- **完整的 Android App**（v1 只验证核心能在 Android 上编译和运行，见 T-020）
- 使用平台专有的 AI 能力（Apple Vision、Apple Intelligence / Foundation Models、ML Kit 等）

---

## 2. 约束

| 项 | 约束 |
|---|---|
| 首发设备 | iPhone 15 Pro 及以上（A17 Pro+，8GB RAM），iOS 26+；**基准测试机为 iPhone 16 Pro** |
| 后续平台 | Android（arm64-v8a，8GB+ 旗舰机，具体机型待定） |
| 开发环境 | **Linux + GPU**：数据、评测、训练、量化、转换，以及 **C++ 核心的开发和测试**、Android NDK 交叉编译。**Mac + Xcode**：只负责 iOS 外壳和真机调试 |
| 平台依赖原则 | 核心只依赖：C++17 标准库、llama.cpp（ggml）、ONNX Runtime、header-only 的小库（JSON、YAML 解析）。**不依赖 OpenCV**（控制包体积）。平台原生能力只能用在平台外壳层：UI、图片解码、相册、文件 |
| 网络 | 推理全程离线；模型文件可以首次启动时下载（开发期直接打进 bundle） |
| 延迟 / 内存 | 见 §6 |
| 输入 | 单张图片。平台层负责解码成 sRGB RGBA8 缓冲区并提取 EXIF，再交给核心 |
| 输出语言 | 中文 |

---

## 3. 总体架构

采用**混合路线**：快速、确定性的信号指标和专用小模型负责"判断有没有问题"，VLM 负责"看图并说出怎么改"。

```
┌─────────── 平台外壳（薄）───────────┐
│ iOS: SwiftUI + ImageIO 解码          │   Android（后期）: Compose + ImageDecoder
│ Swift 封装 ◄── C API ──┐            │   Kotlin/JNI 封装 ◄── C API ──┐
└────────────────────────┼────────────┘                              │
                         ▼                                           ▼
┌──────────────────────── core/  (C++17, 跨平台) ───────────────────────────┐
│ 输入: RGBA8 sRGB 缓冲区 + EXIF 结构体                                       │
│                                                                           │
│ S0 预处理 ──► S1 检测模型 (ONNX Runtime) ──┐                                │
│   缩放/裁切     显著性 · 人脸               │                                │
│          ──► S2 信号指标 (纯 C++) ─────────┼─► S4 融合规则 ─► 分数+标签 ─┐    │
│                曝光 · 清晰度 · 色彩 · 水平线 │   (读 rules.json)          │    │
│          ──► S3 MobileCLIP (ONNX Runtime) ┘                            ▼    │
│                维度打分 · 场景 · 截图过滤          S5 VLM 建议 (llama.cpp mtmd) │
│                                                   GBNF 约束 JSON，流式回调     │
│                                                     │ 超时/失败 ↓             │
│                                                   S5' 模板兜底                │
│                                                     ▼                        │
│                                             QualityReport (JSON)            │
└───────────────────────────────────────────────────────────────────────────┘
        ▲ nanobind Python 绑定
┌───────┴──────── Linux 研究与评测 (Python) ────────┐
│ 原型（numpy/PyTorch）→ 调参 → 规范                │
│ 评测 harness 同时跑"Python 原型"和"C++ 核心"两条 pipeline │
│ 数据集 / 训练 / ONNX 与 GGUF 导出、量化             │
└────────────────────────────────────────────────┘
```

**设计原则**
- **判断和表达分开**：问题标签由 S1–S4 决定（快、稳定、可评测）；VLM 只负责解释和给建议，并且被告知已经检测出的问题，降低幻觉。VLM 可以补充 S4 漏掉的问题，但要单独标记 `source: "vlm"`。
- **一个核心，多个外壳**：所有分析逻辑都在 `core/` 里，对外只暴露一个稳定的 **C API**（`photocritic.h`）。Swift、Kotlin/JNI、Python 都只是这层 C API 的薄封装。平台外壳里不允许出现分析逻辑。
- **模型格式统一**：小模型统一用 **ONNX**（ONNX Runtime 通过 Execution Provider 在不同平台上加速：iOS 用 CoreML EP，Android 用 XNNPACK/NNAPI/QNN EP，Linux 用 CPU/CUDA）；VLM 统一用 **GGUF**（llama.cpp：iOS 用 Metal，Android 用 CPU 或 Vulkan/OpenCL）。Linux 评测跑的就是同一份模型文件。
- **配置即数据**：融合规则、prompt、grammar、建议模板都是数据文件（JSON/YAML/GBNF），核心在运行时加载，Python 原型也读同一份。
- **可降级**：S5 不可用（内存不足、超时、模型未下载）时，用模板生成建议，App 仍然可用。

---

## 4. 各阶段详细设计

### S0 预处理（核心）
- 平台层输入：`RGBA8` 缓冲区（已转换到 sRGB）、宽高、stride，以及 EXIF 结构体（ISO、快门、光圈、焦距、曝光补偿、白平衡模式、设备型号）
- 核心生成 `analysis`（长边 1024）和各模型需要的输入尺寸；缩放用自己实现的 area 插值（精确定义见 DESIGN_SUMMARY §3 S0），保证各平台结果一致
- 平台层的职责（唯一允许依赖原生能力的地方）：HEIC/JPEG 解码、Display P3 转 sRGB、应用 EXIF 方向

### S1 检测模型（核心，ONNX Runtime）
代替 Apple Vision 提供的前置识别，全部使用可移植的开源小模型（T-007 中确定具体选型）：
| 能力 | 候选模型 | 用途 |
|---|---|---|
| 显著性 / 主体分割 | U²-Netp（约 4.7MB）或其他轻量显著目标检测模型 | 主体区域、主体占比、主体位置（三分线距离）、主体是否贴边被裁切 |
| 人脸检测 | YuNet（约 0.3MB）或 BlazeFace | 人像场景：人脸位置、大小、是否被裁切、人脸区域的曝光和清晰度 |

### S2 信号指标（核心，纯 C++）
- **曝光**：亮度直方图，高光溢出比例（L>250）、暗部死黑比例（L<5）、平均亮度、5/95 分位动态范围；显著区域单独算一遍（用来识别逆光主体）
- **清晰度**：拉普拉斯方差（全图 + 显著区域），两者之比用来判断"背景清楚、主体糊了"（失焦）；方向性梯度能量比用来区分运动模糊；用平坦区域的 MAD 估噪声
- **色彩**：Lab 空间下中性像素的 a*/b* 均值（色偏方向和强度），灰世界估计的 CCT；平均饱和度和高饱和溢出比例
- **水平线**（代替 Vision 的水平线检测）：边缘图上做 Hough 直线检测，取较长的近水平线算出倾斜角，并给出置信度
- 算法规范以 Python 原型 `ml/photocritic/signals.py` 的 docstring 为准；C++ 实现必须通过 golden 对齐测试

### S3 专用小模型（核心，ONNX Runtime）
- **阶段一（零样本）**：MobileCLIP（S0 或 S2）的图像编码器导出为 ONNX；每个问题标签配一对正负 prompt（"an overexposed photo" vs "a well-exposed photo"），用 softmax 差值作为分数。文本 embedding 预先算好，作为数据文件打包。
- 同一份图像 embedding 还用来零样本判断：**场景类别**（风景、人像、美食、夜景……）和**截图/文档过滤**（代替 Vision 的 `isUtility`）。
- **阶段二（微调）**：冻结图像编码器，训练多头 MLP（每个维度一个回归头 + 多标签分类头），数据用 SPAQ 属性分、PARA 属性分和合成退化数据；头部同样导出为 ONNX。

### S4 融合规则（核心）
- 输入：S1 + S2 + S3 的特征向量
- 输出：4 个维度分数，以及标签列表 `{tag, confidence, severity, evidence}`
- 规则写在 `assets/rules.yaml`（构建时生成 `rules.json`），包含阈值、权重、场景修正、压制关系和"有意为之"的判断；Python 原型和 C++ 核心执行同一份规则。语法见 `schema/rules_dsl.md`，可用的特征见 `schema/features.md`；23 个标签的起始规则已经写好，阈值是占位值，要在 T-008 中调参
- 规则受场景影响：比如夜景允许更多暗部，高调/低调风格的照片要放宽曝光阈值
- 阶段二可以把部分规则换成小模型（逻辑回归或 GBDT 导出为 ONNX）

### S5 VLM 建议生成（核心，llama.cpp）
- 推理框架：**llama.cpp**（`libmtmd` 多模态），作为 `core/` 的子模块一起编译
- **模型分档**（T-000 真机实测，T-009 质量评测，最后定稿）。因为 VLM 只负责"表达"，问题已经由 S1–S4 查出来了，所以优先用 **1B 以下的小模型**，再用 T-022 的蒸馏 LoRA 补质量：

| 档位 | 候选 | 文件大小（LLM Q4 + mmproj） | 中文 | 许可证 |
|---|---|---|---|---|
| **默认档（< 1B）** | LFM2.5-VL-450M（LFM2.5-350M + SigLIP2 86M，原生 512px，图像 token 数 32–256 可调） | 229MB + 103MB(q8) / 189MB(f16) | 官方支持 | LFM Open License v1.0（商用有条件，需确认） |
| | Qwen3.5-0.8B（原生多模态） | 533MB + 205MB(f16) | 强 | Apache-2.0 |
| | InternVL3-1B（Qwen2.5-0.5B + InternViT） | 675MB(Q8) + 333MB(q8) | 强 | Apache-2.0 |
| **高质量档（约 2B）** | Qwen3-VL-2B | 1107MB + 445MB(q8) | 强 | Apache-2.0 |
| | Qwen3.5-2B | 1281MB + 668MB(f16) | 强 | Apache-2.0 |
| 上限参照（不上端） | Qwen2.5-VL-3B | 约 1.9GB + 约 1.3GB(f16) | 强 | Qwen 许可证 |

  SmolVLM / SmolVLM2（256M、500M）体积最小，但基座以英文为主，中文输出不合格，只作为速度参照。
- **对照组：纯文本小模型**。只把 findings、场景标签和 EXIF 交给一个小语言模型（例如 LFM2.5-350M 或 Qwen3.5-0.8B 的纯文本模式），不输入图片。用来衡量"看图"到底带来多少质量提升；如果提升不大，就可以省掉视觉编码的开销。
- 输入：缩放后的图片（默认 512px，用 `--image-max-tokens` 控制图像 token 数）+ system prompt + S4 的结构化 findings + 场景标签 + EXIF 摘要
- 输出长度上限 200 tokens：JSON 字段名用短 key；数值参数（例如 EV、色温）由模板根据 S2 指标填写，VLM 只写简短文字
- 输出：VLM 只输出**短 key JSON**（`schema/vlm_output.schema.json`）。GBNF grammar 由核心**在运行时生成**：标签枚举 = 本次检测到的标签 ∪ {"other"}。核心再把它展开成 `report.suggestions[]`，`params` 由模板计算。生成过程通过 C 回调流式返回
- 超时：8s 硬超时；支持取消
- **S5' 兜底**：按标签选模板生成建议（参数从 S2 的指标里计算，比如 EV 补偿量），模板在 `assets/templates_zh.yaml`

### C API
完整定义（错误码、线程模型、内存所有权、回调规则）见 [DESIGN_SUMMARY.md](DESIGN_SUMMARY.md) §4，以那里为准。要点：`pc_engine_create` / `pc_analyze`（同步执行 S0–S4）/ `pc_suggest`（异步执行 S5，流式回调）/ `pc_cancel` / `pc_engine_release_vlm`；数据全部用 JSON 字符串跨语言边界传递。好处是 FFI 很简单，代价是多一次序列化（报告很小，可以忽略）。

---

## 5. 数据契约

契约以**文件**为准，本节只做索引：

| 文件 | 内容 |
|---|---|
| `schema/report.schema.json` | QualityReport 的 JSON Schema（draft 2020-12），`schema_version: "1.0"` |
| `schema/vlm_output.schema.json` | VLM 的短 key 输出格式 |
| `schema/taxonomy.yaml` | 23 个问题标签（中文名、定义、有意为之的场景、主要信号）和 11 个场景类别 |
| `schema/features.md` | 规则和 `evidence` 可以使用的特征名、单位和定义 |
| `schema/rules_dsl.md` | 融合规则 DSL 的语法和执行语义 |
| `schema/fixtures/report/{valid,invalid}/` | 示例报告；`valid/01_full_vlm.json` 的分数和置信度都是按 `assets/rules.yaml` 的公式算出来的 |
| `schema/fixtures/vlm_output/{valid,invalid}/` | VLM 输出示例 |

关键约定：
- 一份报告有两种状态：`pc_analyze` 返回时 `suggestions = []`、`generator = "none"`；`pc_suggest` 完成后填充 suggestions，`generator` 为 `vlm:<model>` 或 `template`
- `skip_reason` 不为 null 时（截图/文档、图太小），`scores` 为 null，`issues` 和 `suggestions` 为空
- 坐标全部归一化到 [0,1]，bbox 为 `[x, y, w, h]`，原点在图像左上角
- 被判断为"有意为之"的 issue 保留，但 `intentional_possible: true`，计算分数时扣分打折，建议里要说明这一点
- 修改契约必须升 `schema_version`，并同步 fixtures、taxonomy、rules 和两端的实现

---

## 6. 性能预算（基准机型 iPhone 16 Pro，冷启动除外）

> v0.3：按模型分档设定预算，数值是**初稿，T-000 真机实测后定稿**。v0.2 里"完整输出 ≤ 6s"对 3B 模型不现实（受内存带宽限制，3B Q4 约 20 tok/s），已废弃。

| 指标 | 默认档（< 1B） | 高质量档（约 2B） |
|---|---|---|
| S0–S4 合计 | ≤ 500 ms | ≤ 500 ms |
| S5 首 token（从 S4 完成起算） | ≤ 1.0 s | ≤ 2.5 s |
| 第一条建议完整出现 | ≤ 2 s | ≤ 4 s |
| 全部建议（≤ 200 tokens） | ≤ 4 s | ≤ 10 s |
| 峰值内存（整个 App） | ≤ 1.5 GB | ≤ 3.0 GB（需要申请 `com.apple.developer.kernel.increased-memory-limit`） |
| 模型总体积 | ≤ 1.0 GB | ≤ 2.5 GB |
| 核心库体积（不含模型） | ≤ 30 MB（ONNX Runtime 用精简构建） | 同左 |
| 温控 | 连续分析 20 张，最后 5 张的平均耗时不超过前 5 张的 1.5 倍 | 同左 |

- App 默认用默认档。高质量档只在设备可用内存足够时提供（运行时用 `os_proc_available_memory()` 判断），并且只有 T-009 证明它的质量明显更好时才提供。
- UI 上先展示 S4 的结果（分数和标签），S5 的建议流式追加，这样用户不需要等 VLM。
- 默认档对 Android 尤其重要：在只有 CPU 的情况下，1B 以下的模型才有可能流畅运行。Android 的预算在 T-020 实测后另定。

---

## 7. 评测体系

### 7.1 数据集
| 集合 | 来源 | 用途 |
|---|---|---|
| `synthetic` | 从质量好的原图上做可控退化（曝光 ±EV、色温偏移、高斯/运动模糊、噪声、旋转、裁切），**标签由退化参数精确给出** | 标签 P/R/F1，阈值调参 |
| `spaq` | SPAQ（手机照片，带亮度/色彩/对比/噪声/清晰度属性分） | 维度分数的 SRCC/PLCC |
| `para` | PARA（带构图、色彩、光线、主体突出等审美属性） | 构图和主体维度的 SRCC |
| `golden` | 人工挑选并标注的约 100 张真实照片 | 端到端主观评审，建议质量评分 |

数据集许可证需要在 T-003 中逐个确认，只用于研究和学习。

### 7.2 指标
- 标签：每个标签的 precision / recall / F1，以及宏平均 F1
- 分数：SRCC / PLCC
- 建议质量：用 LLM-as-judge 按评分细则打分（是否命中问题、是否可以照着做、是否有幻觉、中文是否通顺，每项 1–5 分），golden 集再人工抽查
- 结构：JSON schema 合法率（目标 100%，靠 grammar 保证）
- 端侧：各阶段延迟（P50/P90）、峰值内存、温控表现（连续分析 20 张）
- **对齐性**，分三层检查：
  1. Python 原型 vs C++ 核心（Linux）：S2 指标相对误差 ≤ 1%，S4 标签一致率 ≥ 98%
  2. C++ 核心 Linux vs iOS（相同输入缓冲区）：S4 标签一致率 ≥ 98%（不同 EP 和浮点实现会带来微小差异）
  3. Linux vs Android：同上（T-020）

### 7.3 里程碑的通过标准
| 里程碑 | 通过标准 |
|---|---|
| M-1 真机可行性（T-000） | 拿到各候选模型在 iPhone 16 Pro 上的内存、延迟和温控数据；§6 预算定稿 |
| M1 零样本基线（Python 原型） | 端到端跑通；synthetic 宏平均 F1 有基线数字；VLM 选型结论写入 `docs/decisions/` |
| M2a C++ 核心（Linux） | 核心通过 Python 绑定跑完评测，和原型的对齐性达标 |
| M2b iOS MVP | 真机端到端跑通并达到 §6 预算；Linux 和 iOS 的对齐性达标；核心能交叉编译到 Android 并在命令行跑通 |
| M3 微调 | 宏平均 F1 和 SRCC 相对 M1 有提升；建议质量的 judge 分数相对 M1 有提升，且端侧预算不退化 |

---

## 8. 仓库结构

```
on-device/
├── docs/
│   ├── SPEC.md
│   ├── decisions/              # ADR：选型和取舍记录
│   └── tickets/
├── schema/
│   ├── report.schema.json · vlm_output.schema.json
│   ├── taxonomy.yaml · features.md · rules_dsl.md
│   └── fixtures/               # 示例报告、VLM 输出示例、signals_golden/
├── assets/                     # 运行时数据文件，所有平台共用
│   ├── rules.yaml → rules.json
│   ├── templates_zh.yaml
│   ├── prompts/
│   ├── (suggestions.gbnf 由核心在运行时生成，不放文件)
│   └── clip_text_emb.bin
├── core/                       # C++17 跨平台核心（CMake）
│   ├── include/photocritic.h   # 唯一对外的 C API
│   ├── src/                    # preprocess, signals, detectors, clip, fusion, vlm, templates
│   ├── third_party/            # llama.cpp（子模块，固定 commit）、nlohmann/json 等
│   ├── tests/                  # GoogleTest，含 golden 对齐测试
│   ├── tools/cli/              # photocritic-cli（stb_image 解码，Linux/Android 命令行使用）
│   └── bindings/python/        # nanobind
├── ml/                         # Python (uv)，在 Linux 上运行
│   ├── photocritic/            # 原型：signals, clip_scorer, detectors, fusion, vlm, pipeline
│   ├── data/                   # 数据集下载和合成脚本（数据本身不入库）
│   ├── eval/                   # 评测 harness 和报告
│   ├── train/
│   └── export/                 # 导出 ONNX、GGUF，量化
├── platforms/
│   ├── ios/                    # 在 Mac 上开发：Xcode 工程 + Swift 封装 + SwiftUI
│   └── android/                # 后期；v1 只有 NDK 构建脚本
└── scripts/                    # 各平台构建脚本：build-macos.sh, build-ios-xcframework.sh, build-android.sh（Linux 只用于训练）
```

---

## 9. 风险与待确认事项

| 风险 | 应对 |
|---|---|
| 端侧 VLM 的内存、延迟和发热不达标 | **T-000 在动工前用真机实测**；默认档用 1B 以下的模型 |
| 1B 以下的模型建议质量不够（中文表达、幻觉） | VLM 只负责表达，问题由 S1–S4 查出；用 T-022 蒸馏 LoRA；保留 2B 高质量档和纯模板兜底 |
| 新架构模型（LFM2.5、Qwen3.5）在 llama.cpp Metal / ARM CPU 后端上的支持和速度 | T-000 实测；固定 llama.cpp 版本 |
| LFM 许可证对商用有条件 | 学习阶段不受影响；如果要商用，改用 Apache-2.0 的候选（Qwen3.5-0.8B、InternVL3-1B） |
| iOS 后台不允许 GPU 推理 | 切到后台时取消 S5，回到前台后重新执行或用模板兜底 |
| Android 上 llama.cpp 没有 Metal，GPU 后端（Vulkan/OpenCL）成熟度因机型而异 | T-020 先测 CPU 的基线；必要时 Android 上用更小的 VLM 或只用模板 |
| ONNX Runtime 在 iOS 上走 CoreML EP 时，可能有部分算子不支持而回退到 CPU | T-019 测量每个 EP 下实际的节点分配；必要时调整导出方式（opset、静态 shape） |
| 开源显著性模型不如 Apple Vision 准确 | T-007 在 golden 集上评估；不够好时可以用 MobileCLIP 的 patch 特征或 VLM 补充 |
| 两个推理运行时（ORT + ggml）让包体积和内存变大 | ORT 用精简构建；长期可以评估把小模型也迁到 ggml，统一成一个运行时（写进 ADR） |
| C++ 核心的开发成本比 Swift 高 | 好处是 Android 版本基本只需要写外壳；大部分 C++ 开发和测试都可以在 Linux 上完成 |
| 图像 token 太多导致首 token 慢 | 限制输入分辨率；先测量图像编码耗时 |
| 小 VLM 的中文摄影建议质量一般 | 把 findings 喂给模型降低难度；M3 用大模型蒸馏做 LoRA |
| 风格化照片被误判（高调、剪影、刻意模糊） | `intentional_possible` 字段加上 VLM 的判断；golden 集里专门放这类样本 |
| 公开数据集许可证 | T-003 中逐个确认，只用于研究和学习 |

---

## 10. 里程碑与 tickets

见 [tickets/README.md](tickets/README.md)。
