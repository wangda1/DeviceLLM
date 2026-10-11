# PhotoCritic 设计摘要（交接文档）

> 日期：2026-10-11 · 对应 SPEC v0.3 · 仓库目前**只有文档，没有代码**
> 用途：在 Mac 上继续开发时的唯一入口。人和 AI agent 读完这份文档就能接着干活；需要细节时再查 [SPEC.md](SPEC.md) 和 [tickets/](tickets/README.md)。
> 标了 **（待核实）** 的地方是设计时没能验证的事实，实现前要先确认。

---

## 0. 一页速览

- **做什么**：一个离线的手机"摄影点评" App。用户从相册选一张照片，App 输出 4 个维度的分数（曝光、构图与主体、色彩、清晰度）、问题标签和可以照着做的中文建议，并在图上叠加主体框、水平线和建议裁剪框。
- **学什么**：端侧模型的完整链路，包括选型、评测、量化、转换、跨平台部署和微调。
- **怎么做**：
  1. 先用确定性的信号指标和 ONNX 小模型**查出问题**：快、稳定、可以评测。
  2. 再用 1B 以下的小 VLM（llama.cpp）**把问题说清楚**；VLM 不可用时用模板兜底。
  3. 全部逻辑写在一个 **C++ 跨平台核心**里，iOS 和以后的 Android 只做很薄的外壳。
- **下一步**：在 Mac 和 iPhone 16 Pro 上做 **T-000 真机可行性 spike**（§12），用实测数据定下模型档位和性能预算，然后按 §13 的顺序开工。

---

## 1. 决策记录（为什么是现在这个样子）

| # | 决策 | 理由 / 来源 |
|---|---|---|
| D1 | 输入：从相册选单张图，拍完再分析；不做实时取景和批量扫描 | 用户选择；MVP 最简单 |
| D2 | 4 个维度全部覆盖：曝光、主体与构图、色彩、清晰度 | 用户选择 |
| D3 | 输出：维度分数 + 带置信度的问题标签 + 中文建议 + 叠加层（JSON） | 用户选择 |
| D4 | **混合路线**：信号指标和专用小模型负责判断，VLM 负责表达 | 用户选择。判断结果稳定、可评测；VLM 拿到已有结论后再说话，幻觉更少 |
| D5 | 先零样本建立基线，再微调 | 用户选择 |
| D6 | 目标机型：iPhone 15 Pro 及以上、iOS 26+；**基准测试机：iPhone 16 Pro** | 用户选择 |
| D7 | **尽量不依赖平台原生能力**：去掉 Apple Vision 和 Foundation Models，平台层只负责 UI、图片解码、相册和文件 | 用户要求：以后要扩展到 Android 等平台 |
| D8 | **C++17 核心 + 稳定的 C API，跨语言边界用 JSON 字符串传数据** | llama.cpp 和 ONNX Runtime 都是 C/C++；C ABI 是 Swift、Kotlin/JNI、Python 都能接的最小公约数；JSON 让 FFI 最简单，报告很小，序列化开销可以忽略 |
| D9 | 小模型用 **ONNX Runtime**；VLM 用 **llama.cpp (GGUF)** | ORT 各平台都有官方包，通过 Execution Provider 适配硬件加速；Python 评测用的是同一个模型文件。备选方案（ExecuTorch、TFLite、MNN、ncnn）记录在 ADR 里 |
| D10 | **VLM 分档，默认档用 1B 以下的模型** | 粗算显示 3B 模型在手机上的内存约 3.2GB、decode 约 20 tok/s，达不到预算；VLM 只负责表达，小模型加上蒸馏 LoRA 应该够用；对 Android 只有 CPU 的情况也很关键 |
| D11 | **VLM 输出最多 200 tokens**：短 key 的 JSON；数值参数（EV、色温）由模板根据指标计算后填入 | decode 速度受内存带宽限制，输出长度直接决定总耗时 |
| D12 | **T-000 真机 spike 最先做** | 最大的未知数是端侧 VLM 的内存、延迟和发热；在 C++ 核心上投入之前先拿到实测数据 |
| D13 | 文档和 tickets 都放在仓库内（Markdown） | 用户选择；方便按 ticket 分发给 agent |
| D14 | 实现主要在 **Mac** 上进行；Linux（2×RTX 3090）以后主要用于训练（T-021、T-022） | 用户要求。Linux 机器的 g++ 7.5 和 cmake 3.16 太老，**不要在 Linux 上装环境** |

---

## 2. 架构

```
┌──────────── 平台外壳（薄）────────────┐      ┌──── Android 外壳（后期）────┐
│ iOS: SwiftUI + PhotosPicker            │      │ Compose + ImageDecoder       │
│ ImageIO 解码 → sRGB RGBA8 + EXIF       │      │ Kotlin/JNI                   │
│ PhotoCriticKit (Swift 封装)            │      │                              │
└───────────────┬────────────────────────┘      └──────────────┬───────────────┘
                │ C API (photocritic.h)，数据以 JSON 字符串传递  │
┌───────────────▼────────────────────────────────────────────────▼─────────────┐
│                         core/  C++17（跨平台）                               │
│  S0 预处理 ──┬─► S1 检测（ORT）：显著性 U²-Netp、人脸 YuNet ───┐              │
│  area 缩放   ├─► S2 信号指标（纯 C++）：曝光/清晰度/色彩/水平线 ─┼─► S4 融合规则 │
│              └─► S3 MobileCLIP（ORT）：标签分/场景/截图过滤 ───┘   (rules.json) │
│                                                                     │        │
│                     ┌───────────── findings (分数 + 标签) ◄─────────┘        │
│                     ▼                                                        │
│  S5 VLM（llama.cpp + mtmd，GBNF 约束 JSON，流式回调，超时 / 取消）            │
│      └─ 失败 / 超时 / 模型未加载 ─► S5' 模板兜底 (templates_zh)               │
│                     ▼                                                        │
│               QualityReport JSON                                             │
└──────────────────────────────────────────────────────────────────────────────┘
      ▲ nanobind
┌─────┴──────── ml/  Python（研究、评测、训练、导出）─────────────┐
│ 原型 → 规范 → 评测 harness 同时跑 "python" 和 "cpp" 两条 pipeline  │
└─────────────────────────────────────────────────────────────────┘
```

### 分层红线（每次 review 都要检查）
1. 分析逻辑只能写在 `core/`（C++）或 `ml/`（Python 原型）里。**Swift / Kotlin 里不能出现任何阈值、规则或算法。**
2. `core/` 里不能 include 任何平台头文件（`<TargetConditionals.h>`、`<android/...>`、Objective-C 等）。平台差异只能通过 CMake 选项和 config JSON 来区分。
3. 规则、prompt、grammar、模板、文本 embedding 全部是 `assets/` 里的**数据文件**，Python 和 C++ 读同一份。
4. 核心不依赖 OpenCV（包体积太大），缩放、卷积、Hough 变换都自己实现。

### 两条主要流程
- `pc_analyze`（同步，目标 ≤ 500ms）：S0 → 并行执行 {S1, S2（需要 S1 的显著性 mask，所以显著性先跑）, S3} → S4 → 返回不含 suggestions 的报告。UI 立即展示分数和标签。
- `pc_suggest`（异步）：输入上一步的报告 → S5 流式输出 → 结束时返回完整报告。失败、超时或被取消时走 S5'。

实际的依赖顺序：`S0 → S1(显著性, 人脸) → S2(需要 mask)`；`S3` 可以和 `S1` 并行。

---

## 3. 各阶段设计（实现规范）

### S0 预处理
- **平台层的职责**：把 HEIC/JPEG 解码成 **sRGB、RGBA8、已应用 EXIF 方向**的图像，长边限制在 2048 以内（省内存）；提取 EXIF。
- **核心的职责**：
  - `analysis` 图：长边缩放到 1024，用 area（box）插值。规范是：每个目标像素等于覆盖它的源像素按面积加权的平均值，在 8-bit sRGB 值上计算，四舍五入。Python 和 C++ 必须逐像素一致（允许 ±1）。
  - 各模型的输入：从 `analysis` 图再缩放，每个模型自己的预处理写在模型说明里（§6）。
  - 小于 256px 的图直接返回 `skip_reason: "too_small"`。

### S1 检测模型（ONNX Runtime）
| 模型 | 来源 | 输入 | 输出与后处理 |
|---|---|---|---|
| 显著性：U²-Netp（约 4.7MB） | github.com/xuebinqin/U-2-Net（Apache-2.0）**（待核实许可证）** | 320×320，RGB/255，再按 ImageNet mean/std 归一化 | 取 d0 输出（sigmoid）→ min-max 归一化 → 以 0.5 为阈值得到 mask → 最大连通域的 bbox；另外压缩成 16×16 网格（均值）给叠加层用 |
| 人脸：YuNet（约 0.3MB） | opencv/opencv_zoo `face_detection_yunet`（MIT） | 动态尺寸（建议 320 长边） | 按官方实现解码 → score 阈值 0.6 → NMS（IoU 0.3）→ bbox + 5 个关键点 |

前处理和后处理的细节以官方实现为准，T-007 中固化成 Python 代码和文档，C++ 照着实现。

### S2 信号指标（纯算法；Python 原型是规范，C++ 的误差 ≤ 1%）
输入：`analysis` 图（sRGB uint8）和可选的显著性 mask。

| 组 | 指标 | 精确定义（初稿） |
|---|---|---|
| 曝光 | `luma_hist[256]` | `Y = round(0.2126R + 0.7152G + 0.0722B)`，在 sRGB 编码值上计算（luma，不是线性亮度） |
| | `clip_high` / `clip_low` | `Y ≥ 250` / `Y ≤ 5` 的像素占比 |
| | `mean_L`、`p5`、`p95`、`dyn_range = p95 − p5` | 在 Y 上统计 |
| | `*_salient` | 同一组指标只在 mask 内计算；`backlit_delta = mean_L_salient − mean_L_background` |
| 清晰度 | `lap_var_global` | 灰度 Y/255 → 3×3 拉普拉斯核 `[0,1,0;1,-4,1;0,1,0]`，边界用 replicate → 方差 |
| | `lap_var_salient`、`focus_ratio = lap_var_salient / lap_var_background` | 主体糊、背景清楚 → 比值小 → 判断为 `subject_out_of_focus` |
| | `grad_anisotropy` | Sobel 得到 gx、gy；在梯度幅值前 20% 的像素上计算结构张量 J；`(λ1−λ2)/(λ1+λ2)`，越接近 1，方向性越强（运动模糊）；同时输出主方向角 |
| | `noise_sigma` | 分成 8×8 块，选梯度能量最低的 10% 块，在块内计算拉普拉斯残差的 `MAD/0.6745`，取中位数 |
| 色彩 | `lab_a_neutral`、`lab_b_neutral` | sRGB → 线性 → XYZ(D65) → Lab；中性像素指 `C* < 15 且 20 < L* < 90`；对它们求 a*、b* 的均值（色偏方向和强度） |
| | `cct` | 中性像素的线性 RGB 平均值 → XYZ → xy → McCamy 公式 |
| | `sat_mean`、`sat_clip` | HSV 的 S 通道均值；`S > 0.95 且 V > 0.2` 的像素占比 |
| 水平线 | `horizon_angle_deg`、`horizon_conf` | Sobel 幅值取前 10% 的边缘点 → Hough 变换，θ 限制在 ±15°（以水平为 0°），步长 0.1°，按幅值加权投票 → 峰值角；`conf = 峰值票数 / 总票数`，再用一个阈值做归一化 |

单元测试要覆盖：纯色图、棋盘格、已知的高斯/运动模糊核、已知的旋转角、已知的通道增益（色偏）。

### S3 MobileCLIP（ONNX Runtime）
- 模型：MobileCLIP-S0（首选，最快）和 S2（对比）；来源是 apple/ml-mobileclip **（预处理细节和许可证待核实，以官方 preprocess 为准）**；只导出**图像编码器**。
- 输出的图像 embedding 经过 L2 归一化后用于：
  1. **标签分**：每个标签有 K 对 prompt（pos 和 neg），`p_tag = mean_k softmax(100·[cos(e, pos_k), cos(e, neg_k)])[0]`
  2. **场景分类**：landscape / portrait / food / night / architecture / street / pet / document / screenshot / other，取 softmax 的 top-2
  3. **截图和文档过滤**：如果 `screenshot + document` 的概率 > 0.6，就返回 `skip_reason: "utility"`
- 文本 embedding 预先算好：`assets/clip_text_emb.bin`，格式是 `magic "PCTE" | u32 version | u32 count | u32 dim | float16[count*dim]`，另有一个 `clip_text_emb.json` 记录每一行对应的标签或场景以及 pos/neg。
- 阶段二（T-021）：在 embedding 上接多头 MLP，导出成独立的 ONNX，通过 config 切换。

### S4 融合规则（`assets/rules.yaml` → 构建时生成 `rules.json`）
**规则 DSL（初稿）**：只允许比较、`all` / `any` / `not`、分段线性函数和阈值分档，Python 和 C++ 各写一个解释器。
```yaml
version: 1
features_doc: "特征名与 S1/S2/S3 输出字段一一对应，见 signals.py / detectors.py / clip_scorer.py"
rules:
  - tag: highlights_clipped
    dimension: exposure
    when: {all: [{f: clip_high, gt: 0.02}]}
    confidence: {piecewise: {f: clip_high, points: [[0.02, 0.5], [0.08, 0.95]]}}
    severity: {f: clip_high, thresholds: [0.03, 0.06]}     # <0.03 low, <0.06 medium, 其余 high
    evidence: [clip_high, clip_high_salient]
    scene_overrides:
      night: {when: {all: [{f: clip_high, gt: 0.05}]}}      # 夜景放宽
  - tag: subject_out_of_focus
    dimension: sharpness
    when: {all: [{f: has_subject, eq: 1}, {f: focus_ratio, lt: 0.35}, {f: lap_var_background, gt: 0.002}]}
    ...
intentional:
  - name: silhouette
    when: {all: [{f: mean_L_salient, lt: 30}, {f: clip_low_salient, gt: 0.5}, {f: lap_var_salient, gt: 0.001}]}
    marks: [underexposed, shadows_crushed, backlit_subject]
scores:
  # 维度分 = clamp(100 − Σ(该维度的 issue: weight[tag] × sev_w[severity] × confidence), 0, 100)
  sev_w: {low: 0.5, medium: 1.0, high: 1.6}
  weights: {highlights_clipped: 20, overexposed: 30, ...}
  overall: {exposure: 0.3, composition: 0.3, color: 0.2, sharpness: 0.2}
```
- `suggested_crop`：如果有主体，就把主体中心对齐到最近的三分线交点，保持原图比例，裁剪面积不少于原图的 70%，在边界内做 clamp；没有主体时不输出。
- 输出的 issues 按 `severity × confidence` 降序排列，最多 6 个。

### S5 VLM 建议生成（llama.cpp + mtmd）
**输入 prompt**（system 放在 `assets/prompts/vlm_system.md`，user 消息由核心拼装）：
```
<image>
场景：风景，户外
EXIF：ISO 800，1/30s，f/1.8，EV 0
主体：画面中央偏左，约占 9%
已检测到的问题：
1. highlights_clipped｜高光溢出｜严重｜高光溢出 8.3%，主要在画面上部
2. warm_cast｜偏暖｜中等｜b* +12
请针对以上问题给出拍摄或后期建议。
```
- 位置描述由核心把 bbox 映射成九宫格的中文描述（"左上"、"中央偏左"……），不把原始坐标交给模型。
- **输出格式（短 key，用 GBNF 约束）**：
```json
{"s":[{"t":["highlights_clipped"],"h":"天空过曝","a":"点按天空测光或降低曝光补偿，必要时开启 HDR。"}]}
```
  - `t` 只能取本次检测到的标签，或者 `"other"`（grammar 根据本次的标签**动态生成枚举**）
  - `h` 不超过 12 字；`a` 不超过 60 字；最多 3 条
- **后处理**：核心把短 key 展开成 schema 里的 `suggestions[]`；`params`（例如 `ev_delta`、`wb_kelvin_delta`、`rotate_deg`）**由模板根据 S2 的指标计算后填入**，不让 VLM 编数字。
- 采样参数：temperature 0.2，top_p 0.9，固定 seed；`max_new_tokens = 200`；超时 8s（以后由 T-000 的数据调整）。
- **取消**：`pc_cancel` 设置一个原子标志，解码循环每个 token 检查一次；App 进入后台时平台层必须调用（iOS 不允许后台执行 GPU 任务）。
- **S5' 模板兜底**（`assets/templates_zh.yaml`）：
```yaml
highlights_clipped:
  h: "高光溢出"
  a: "高光有 {clip_high_pct}% 溢出。拍摄时点按亮部测光，或者把曝光补偿调到 {ev_delta} EV；也可以开启 HDR。"
  params:
    ev_delta: {expr: piecewise, f: clip_high, points: [[0.02, -0.3], [0.1, -1.0]], round: 0.1}
```
  占位符替换只支持 `{name}`，参数计算复用规则 DSL 里的分段线性函数。

---

## 4. C API（`core/include/photocritic.h`，初稿）

```c
#ifdef __cplusplus
extern "C" {
#endif
#define PC_API_VERSION 1

typedef enum { PC_OK = 0, PC_ERR_INVALID_ARG, PC_ERR_MODEL_LOAD, PC_ERR_INFERENCE,
               PC_ERR_TIMEOUT, PC_ERR_CANCELLED, PC_ERR_OOM, PC_ERR_BUSY } pc_status;

typedef enum { PC_PIXEL_RGBA8 = 0, PC_PIXEL_BGRA8 = 1 } pc_pixel_format;

typedef struct {
  const uint8_t* data; int32_t width, height, stride_bytes;
  pc_pixel_format format;           // 必须是 sRGB、已应用方向
} pc_image;

typedef struct {                    // 缺失的字段：数值填 NaN，字符串填 NULL
  double iso, exposure_time_s, f_number, focal_length_35mm, exposure_bias_ev;
  int32_t white_balance_mode;       // -1 未知，0 自动，1 手动
  const char* device_model;
} pc_exif;

typedef struct pc_engine pc_engine;

// config_json 见 §5；失败时返回 NULL，并通过 err_out 返回错误信息（调用方用 pc_free_string 释放）
pc_engine* pc_engine_create(const char* config_json, char** err_out);
void       pc_engine_destroy(pc_engine*);

// 同步执行 S0–S4。成功时 *report_json_out 是报告（不含 suggestions），调用方负责释放
pc_status  pc_analyze(pc_engine*, const pc_image*, const pc_exif*, char** report_json_out);

// 异步执行 S5。立即返回；on_token 和 on_done 在核心的工作线程上调用
// 回调里传入的字符串只在回调期间有效；同一时间只能有一个 suggest 任务（否则返回 PC_ERR_BUSY）
typedef void (*pc_token_cb)(const char* text_piece, void* user);
typedef void (*pc_done_cb)(pc_status status, const char* final_report_json, void* user);
pc_status  pc_suggest(pc_engine*, const char* report_json, pc_token_cb, pc_done_cb, void* user);
void       pc_cancel(pc_engine*);              // 线程安全；会在 ≤100ms 内生效

void       pc_engine_release_vlm(pc_engine*);  // 收到内存警告时调用；下次 suggest 时再懒加载
pc_status  pc_engine_preload_vlm(pc_engine*);  // 可选：提前加载，减少首次使用的延迟
const char* pc_version(void);                   // 静态字符串，不需要释放
void       pc_free_string(char*);
#ifdef __cplusplus
}
#endif
```
- **线程模型**：一个 engine 同一时间只执行一个 `pc_analyze`（内部会并行执行各阶段）；`pc_suggest` 可以和下一次 `pc_analyze` 同时运行，但 VLM 部分是串行的。Swift 端在回调里要切回 MainActor。
- **为什么 token 流和最终结果分开回调**：`on_token` 用来实时显示进度；最终以 `on_done` 返回的完整 JSON 为准，UI 用它替换之前的流式内容（流式输出是 JSON 片段，UI 可以只显示 `a` 字段的增量，也可以只显示"生成中"）。

---

## 5. 配置与数据契约

### config_json
```json
{
  "assets_dir": "/path/to/assets",
  "models": {
    "saliency": "u2netp.onnx",
    "face": "yunet.onnx",
    "clip_image": "mobileclip_s0_image_fp16.onnx",
    "vlm": "LFM2.5-VL-450M-Q4_0.gguf",
    "mmproj": "mmproj-LFM2.5-VL-450m-Q8_0.gguf"
  },
  "ort": {"ep": "coreml", "intra_threads": 4},
  "vlm": {"enabled": true, "n_ctx": 2048, "n_threads": 4, "n_gpu_layers": 99,
          "image_max_tokens": 128, "max_new_tokens": 200, "timeout_ms": 8000,
          "temperature": 0.2, "seed": 42},
  "lang": "zh"
}
```
`ort.ep` 可选 `cpu | coreml | xnnpack | nnapi | cuda`；EP 不可用时回退到 cpu，并记录在报告的 `runtime` 字段里。

### QualityReport（`schema/report.schema.json`，T-002 中写成正式的 JSON Schema）
| 字段 | 类型 | 说明 |
|---|---|---|
| `schema_version` | `"1.0"` | 修改契约时必须升版本 |
| `image` | `{width, height, exif{...}}` | 原图尺寸 |
| `skip_reason` | `null \| "utility" \| "too_small"` | 不为 null 时，后面的字段可以为空 |
| `scene` | `string[]`（最多 2 个） | 场景枚举见 S3 |
| `scores` | `{exposure, composition, color, sharpness, overall}`，整数 0–100 | |
| `issues[]` | `{tag, dimension, confidence(0–1), severity(low/medium/high), source(rules/vlm), intentional_possible(bool), evidence{特征名: 数值}}` | 按严重程度降序，最多 6 个 |
| `suggestions[]` | `{issue_tags[], title, action, params{}, priority(1..), source(vlm/template)}` | 最多 3 条 |
| `overlays` | `{saliency_bbox[x,y,w,h], saliency_grid[16][16], faces[[x,y,w,h]], horizon_angle_deg, horizon_conf, suggested_crop[x,y,w,h]}` | 坐标都归一化到 0–1，以图像左上角为原点 |
| `timings_ms` | `{s0..s5}` | |
| `runtime` | `{platform, onnx_ep, llama_backend, core_version}` | |
| `generator` | `"vlm:<model-id>" \| "template"` | |

### 标签体系 v1（`schema/taxonomy.yaml`）
| 维度 | 标签 |
|---|---|
| exposure | `overexposed` 过曝 · `underexposed` 欠曝 · `highlights_clipped` 高光溢出 · `shadows_crushed` 暗部死黑 · `low_contrast` 发灰 · `backlit_subject` 逆光主体偏暗 |
| composition | `subject_not_prominent` 主体不突出 · `subject_too_small` 主体太小 · `subject_cut_off` 主体被裁切 · `cluttered_background` 背景杂乱 · `tilted_horizon` 水平线倾斜 · `unbalanced` 构图失衡 · `excess_empty_space` 留白过多 |
| color | `warm_cast` 偏暖 · `cool_cast` 偏冷 · `green_tint` 偏绿 · `magenta_tint` 偏品红 · `oversaturated` 过饱和 · `dull_colors` 色彩寡淡 |
| sharpness | `motion_blur` 运动模糊/手抖 · `out_of_focus` 全图失焦 · `subject_out_of_focus` 主体跑焦 · `high_noise` 噪点多 |

每个标签主要由哪些信号检测（规则的起点）：
- 曝光类 → S2 曝光组；`backlit_subject` 需要显著性 mask；`low_contrast` → `dyn_range`
- 构图类 → S1 显著性和人脸（主体占比、位置、是否贴边）+ S2 水平线 + S3 CLIP（`cluttered_background`、`unbalanced` 主要靠 CLIP，零样本效果预计较弱）
- 色彩类 → S2 的 Lab 和 CCT 指标、饱和度 + S3 CLIP 辅助
- 清晰度类 → S2 清晰度组 + 人脸区域的拉普拉斯方差 + S3 CLIP 辅助

---

## 6. 模型清单

### 小模型（ONNX）
| 用途 | 模型 | 体积 | 许可证 |
|---|---|---|---|
| 显著性 | U²-Netp | 约 4.7MB | Apache-2.0（待核实） |
| 人脸 | YuNet（opencv_zoo） | 约 0.3MB | MIT |
| 场景/标签分 | MobileCLIP-S0 图像编码器（FP16） | 约 20–30MB（待实测） | Apple 的许可证（待核实是否允许在 App 中使用） |

### VLM（GGUF，已通过 Hugging Face API 核实文件大小，2026-10-11）
| 档位 | 仓库 | LLM 文件 | mmproj 文件 | 许可证 |
|---|---|---|---|---|
| 默认档 ① | `LiquidAI/LFM2.5-VL-450M-GGUF` | `LFM2.5-VL-450M-Q4_0.gguf` 219MB / `-Q4_K_M` 229MB / `-Q8_0` 379MB | `mmproj-LFM2.5-VL-450m-Q8_0.gguf` 103MB / `-F16` 189MB | LFM Open License v1.0（商用有条件） |
| 默认档 ② | `unsloth/Qwen3.5-0.8B-GGUF` | `Qwen3.5-0.8B-Q4_K_M.gguf` 533MB / `-Q4_0` 507MB | `mmproj-F16.gguf` 205MB | Apache-2.0 |
| 默认档 ③ | `ggml-org/InternVL3-1B-Instruct-GGUF` | `InternVL3-1B-Instruct-Q8_0.gguf` 675MB | `mmproj-InternVL3-1B-Instruct-Q8_0.gguf` 333MB | Apache-2.0 |
| 高质量档 | `Qwen/Qwen3-VL-2B-Instruct-GGUF` | `Qwen3VL-2B-Instruct-Q4_K_M.gguf` 1107MB | `mmproj-Qwen3VL-2B-Instruct-Q8_0.gguf` 445MB | Apache-2.0 |
| 上限参照 | `ggml-org/Qwen2.5-VL-3B-Instruct-GGUF`（文件名待核实） | Q4_K_M 约 1.9GB | f16 约 1.3GB | Qwen 许可证 |
| 纯文本对照组 | `LiquidAI/LFM2.5-350M-GGUF`（待核实） | Q4_0 | — | LFM |
| 速度参照（中文弱） | `ggml-org/SmolVLM2-500M-Video-Instruct-GGUF` | Q8_0 437MB | q8 109MB | Apache-2.0 |

LFM2.5-VL-450M 的结构：LFM2.5-350M + SigLIP2 NaFlex 86M；原生 512×512，大图切成 512 的块再加一张缩略图；图像 token 数可在 32–256 之间调整（llama.cpp 参数 `--image-max-tokens`）。

---

## 7. 性能预算（iPhone 16 Pro，初稿，T-000 后定稿）

| 指标 | 默认档（< 1B） | 高质量档（约 2B） |
|---|---|---|
| S0–S4 | ≤ 500 ms | ≤ 500 ms |
| S5 首 token | ≤ 1.0 s | ≤ 2.5 s |
| 第一条建议完整出现 | ≤ 2 s | ≤ 4 s |
| 全部建议（≤ 200 tokens） | ≤ 4 s | ≤ 10 s |
| 峰值内存（整个 App） | ≤ 1.5 GB | ≤ 3.0 GB（需要 increased-memory-limit） |
| 模型总体积 | ≤ 1.0 GB | ≤ 2.5 GB |
| 核心库体积（不含模型） | ≤ 30 MB | 同左 |
| 温控 | 连续 20 张，最后 5 张的平均耗时 ≤ 前 5 张 × 1.5 | 同左 |

**估算方法**（用于判断数量级，误差约 ±2 倍）：
- decode tok/s ≈ 内存带宽（约 60GB/s）÷ 每个 token 读取的权重字节数 × 50–70%
- prefill 和图像编码的计算量 ≈ 2 × 参数量 × token 数（或 patch 数），按 GPU 峰值算力的 30–50% 估算
- 中文 JSON 约 1–1.5 字/token

---

## 8. 仓库结构（目标）

```
on-device/
├── docs/            SPEC.md · DESIGN_SUMMARY.md（本文件）· decisions/（ADR）· tickets/
├── schema/          report.schema.json · taxonomy.yaml · fixtures/（示例报告、signals_golden/）
├── assets/          rules.yaml→rules.json · templates_zh.yaml · prompts/ · suggestions.gbnf(模板) · clip_text_emb.{bin,json}
├── core/
│   ├── CMakeLists.txt         选项：PC_WITH_LLAMA, PC_WITH_ORT, PC_METAL, PC_CUDA, PC_VULKAN, PC_BUILD_TESTS, PC_BUILD_PYTHON
│   ├── include/photocritic.h
│   ├── src/{preprocess,signals,detectors,clip,fusion,vlm,templates,report,util}/
│   ├── third_party/           llama.cpp（git submodule，固定 commit）· nlohmann/json · stb_image（只给 CLI 用）
│   ├── tests/                 GoogleTest；golden 对齐测试读取 schema/fixtures
│   ├── tools/cli/             photocritic-cli
│   └── bindings/python/       nanobind → photocritic_core
├── ml/              Python（uv）：photocritic/（原型）· data/ · eval/ · train/ · export/
├── platforms/
│   ├── ios/                   PhotoCritic.xcodeproj · PhotoCriticKit（SwiftPM，binaryTarget 引用 xcframework）
│   └── android/               以后再做；目前只有 NDK 构建脚本
├── spikes/t000/               T-000 的一次性代码（不进入 core）
└── scripts/                   build-macos.sh · build-ios-xcframework.sh · build-android.sh · fetch-models.sh
```

---

## 9. 构建方案

### macOS（本地开发核心的主力平台）
- 需要：Xcode 26（iOS 26 SDK）、Homebrew：`cmake ninja uv`，以及 `pip install -U "huggingface_hub[cli]"`（提供 `hf` 命令）
- ONNX Runtime：macOS 开发时用 `brew install onnxruntime`，或者从官方 release 下载 `onnxruntime-osx-arm64-<ver>.tgz`；CMake 通过 `ORT_ROOT` 找到它
- llama.cpp：`add_subdirectory(third_party/llama.cpp)`，开启 `GGML_METAL=ON`；多模态库 `mtmd` 位于 `tools/mtmd`，需要确认它在库模式下作为 target 的导出方式**（待核实，跟着固定的 commit 走）**
- 在 macOS 上就可以用 Metal 跑完整的 S0–S5，CLI 和 Python 绑定都可用，所以 T-012~T-016 可以全部在 Mac 上完成

### iOS xcframework
- 先用 llama.cpp 自带的 `build-xcframework.sh` 验证（T-000），**确认产物里有 mtmd 的头文件和符号**
- 正式方案（`scripts/build-ios-xcframework.sh`）：用 CMake 的 iOS 工具链（`-DCMAKE_SYSTEM_NAME=iOS -DCMAKE_OSX_ARCHITECTURES=arm64`）分别构建 `ios-arm64`、`ios-arm64-simulator`、`macos-arm64` 三个 slice → 用 `libtool -static` 合并 core、ggml、llama、mtmd 的静态库 → `xcodebuild -create-xcframework`
- ONNX Runtime iOS：使用官方的 `onnxruntime-c`（CocoaPods）或 `onnxruntime-swift-package-manager` 提供的 xcframework，作为**单独的** binaryTarget 链接，不合并进我们的库；需要在构建时启用 CoreML EP
- 包体积：之后用 ORT 的精简构建（`--minimal_build` 加上只包含所需算子的配置）把体积降下来（T-019）

### Linux（只用于训练和数据处理，**不在这台机器上装工具链**）
### Android（T-020）：NDK 的 CMake 工具链，arm64-v8a；llama.cpp 用 CPU 后端（可选 Vulkan）；ORT 用 Android AAR 里的 so，开启 XNNPACK EP

---

## 10. iOS 外壳设计（T-017、T-018）

- **PhotoCriticKit（SwiftPM）**
  - `Engine`（actor）：持有 `OpaquePointer`；`init(config:)`、`analyze(_ image: CGImage, exif:) async throws -> Report`（在后台线程调用 `pc_analyze`）、`suggest(_ report:) -> AsyncThrowingStream<SuggestEvent, Error>`（用 `Unmanaged` 把 continuation 传给 C 回调的 `user` 指针）、`cancel()`、`releaseVLM()`
  - `Report: Codable`：和 schema 一一对应，snake_case 通过 `keyDecodingStrategy` 转换
  - `ImageLoader`：`CGImageSourceCreateThumbnailAtIndex`，参数 `kCGImageSourceCreateThumbnailFromImageAlways`、`kCGImageSourceCreateThumbnailWithTransform`（应用方向）、`kCGImageSourceThumbnailMaxPixelSize = 2048` → 用 `CGContext`（sRGB 色彩空间、RGBA8、`noneSkipLast`）画出来，拿到 sRGB 缓冲区（Display P3 在这一步转换）
  - `ExifReader`：从 `CGImageSourceCopyPropertiesAtIndex` 读取 `{Exif}`（ISOSpeedRatings、ExposureTime、FNumber、FocalLenIn35mmFilm、ExposureBiasValue、WhiteBalance）和 `{TIFF}`（Model）
- **生命周期**：
  - `scenePhase == .background` → `cancel()`
  - `UIApplication.didReceiveMemoryWarningNotification` → `releaseVLM()`
  - App 启动后在后台调用 `pc_engine_preload_vlm`（如果 T-000 证明加载很快，就改为懒加载）
- **模型分发**：开发期把模型放进 bundle 的 folder reference，或者开启 `UIFileSharingEnabled`，通过 Finder 拷进沙盒；正式版首次启动时下载到 Application Support，设置 `isExcludedFromBackup`
- **UI**：PhotosPicker → 分析页（先展示分数卡和标签，建议流式出现）→ 叠加层开关（显著性网格热图、水平线、裁剪框）→ 开发信息（各阶段耗时、EP、模型）；隐藏的 benchmark 模式（T-019）
- **权限**：`com.apple.developer.kernel.increased-memory-limit`（只在高质量档需要；**是否需要付费开发者账号待核实**）；用 PhotosPicker 不需要申请相册权限

---

## 11. 评测设计（概要）

- **数据集**：synthetic（可控退化，标签精确）、SPAQ（属性分，看 SRCC）、PARA（审美属性，看构图和主体）、golden（约 100 张人工标注 + 参考建议）。**SPAQ 和 PARA 的许可证待核实。**
- **Harness**：`ml/eval/run.py --pipeline {dummy,python,clip_only,rules_only,cpp} --dataset ... --split dev|test`；按"图片 hash + 阶段版本"缓存特征；输出 Markdown 和 JSON 报告；`compare.py A B` 生成对比表
- **指标**：每个标签的 P/R/F1 和宏平均；SRCC/PLCC；schema 合法率；LLM-as-judge（是否命中问题、是否可执行、有没有幻觉、中文是否通顺，各 1–5 分）；端侧延迟（P50/P90）、内存、温控
- **跨端对齐**：Python vs C++（S2 误差 ≤ 1%，S4 标签一致率 ≥ 98%）；macOS/Linux vs iOS；以后再加 Android
- **数据外发**：LLM-as-judge 和 T-022 的 teacher 模型如果用云端大模型，会把图片发出去；如果要全程本地，就用 Linux 3090 上跑开源大 VLM 代替

---

## 12. T-000 真机 spike 执行清单（在 Mac 上的第一件事）

**目标**：1–2 天内拿到各候选 VLM 在 iPhone 16 Pro 上的真实数据，定下默认档和高质量档，并修订 SPEC §6。代码放在 `spikes/t000/`，**不进入 core**。

1. **准备环境**
   ```bash
   brew install cmake ninja uv
   pip install -U "huggingface_hub[cli]"
   git clone https://github.com/ggml-org/llama.cpp spikes/t000/llama.cpp
   cd spikes/t000/llama.cpp && git rev-parse HEAD     # 记下这个 commit，写进报告
   ```
2. **下载模型**（放在 `spikes/t000/models/`，不要提交到 git）
   ```bash
   hf download LiquidAI/LFM2.5-VL-450M-GGUF LFM2.5-VL-450M-Q4_0.gguf LFM2.5-VL-450M-Q8_0.gguf \
       mmproj-LFM2.5-VL-450m-Q8_0.gguf mmproj-LFM2.5-VL-450m-F16.gguf --local-dir models/lfm25vl450m
   hf download unsloth/Qwen3.5-0.8B-GGUF Qwen3.5-0.8B-Q4_K_M.gguf mmproj-F16.gguf --local-dir models/qwen35-08b
   hf download ggml-org/InternVL3-1B-Instruct-GGUF InternVL3-1B-Instruct-Q8_0.gguf \
       mmproj-InternVL3-1B-Instruct-Q8_0.gguf --local-dir models/internvl3-1b
   hf download Qwen/Qwen3-VL-2B-Instruct-GGUF Qwen3VL-2B-Instruct-Q4_K_M.gguf \
       mmproj-Qwen3VL-2B-Instruct-Q8_0.gguf --local-dir models/qwen3vl-2b
   # Qwen2.5-VL-3B 和 LFM2.5-350M 的仓库名和文件名需要先在 HF 上确认
   ```
3. **先在 Mac 上跑通**（Metal）：`cmake -B build -DGGML_METAL=ON && cmake --build build -j --target llama-mtmd-cli`，然后对每个模型执行：
   `llama-mtmd-cli -m <llm.gguf> --mmproj <mmproj.gguf> --image test.jpg -p "<固定 prompt>" -n 200 --temp 0.2 --seed 42`
   记录 Mac 上的速度，粗看中文质量；明显不能用的模型就不上手机了。
4. **构建 iOS 版本**：`./build-xcframework.sh` → 检查 `build-apple/llama.xcframework` 里**是否有 `mtmd.h`**。如果没有，记录需要怎样修改构建脚本（例如把 mtmd 的源文件也编进去）。
5. **最小 iOS demo**（SwiftUI，一个页面）：
   - 选择模型组合 → 加载（`llama_model_load_from_file` + `mtmd_init_from_file`）→ 用 mtmd 的 helper 读取 bitmap、tokenize、eval chunks → 采样循环生成 200 tokens。**函数名以固定 commit 的 `mtmd.h`、`mtmd-helper.h` 和 `llama.h` 为准**；参考 `tools/mtmd/mtmd-cli.cpp` 的流程。
   - 测量：各阶段耗时（用 `mach_absolute_time`）；`os_proc_available_memory()` 在加载前后和推理中的值；`task_info(TASK_VM_INFO).phys_footprint` 的峰值；`ProcessInfo.processInfo.thermalState`
   - 连续跑 20 次的模式；推理中途切到后台的测试
   - 结果导出 CSV（通过分享面板发到 Mac）
6. **固定的测试 prompt**（所有模型都用这一份）：
   - system：`你是一位专业摄影老师。根据图片和已检测到的问题，给出最多 3 条可执行的中文改进建议。只输出 JSON：{"s":[{"t":[标签],"h":"标题(≤12字)","a":"建议(≤60字)"}]}`
   - user：§3 S5 中那段示例的 findings，加上图片
   - 测试图片：5 张（风景过曝、人像逆光、夜景噪点、倾斜的建筑、偏暖的室内），分别缩放到 512 和 768 两档
7. **测试矩阵**：见 [T-000](tickets/T-000-device-feasibility-spike.md)。每个组合跑 3 次取中位数；内存上限要在申请和不申请 increased-memory-limit 两种情况下各测一次。
8. **产出**：`docs/decisions/0000-device-feasibility.md`（数据表、结论、选定的模型、修订后的预算）+ 提交对 SPEC §6 的修改 + 把实测数据和估算的差距写进 LEARN

---

## 13. Mac 上的开发顺序（在 tickets/README 的基础上，按"主要在 Mac 上开发"调整）

| 阶段 | Tickets | 说明 |
|---|---|---|
| A | **T-000** | 真机 spike，决定模型档位 |
| B | T-001 → T-002 → T-003 → T-004 | T-001 改为在 macOS 上搭建：CMake + Apple Clang + uv；ORT 用 macOS 版本 |
| C | T-005、T-006、T-007、T-009 并行 → T-008 → T-010 → T-011 | Python 原型和评测都在 Mac 上跑；T-009 用 llama.cpp Metal；MobileCLIP 导出用 PyTorch（CPU 或 MPS） |
| D | T-012 → T-013、T-014、T-015 → T-016 | C++ 核心在 macOS 上开发和测试（ORT 用 CPU 或 CoreML EP，llama.cpp 用 Metal） |
| E | T-017 → T-018 → T-019 | iOS 外壳；T-017 的骨架在 T-012 完成后就可以开始 |
| F | T-020 | Android NDK 在 Mac 上也能用；需要一台 Android 真机 |
| G | T-021、T-022 → T-023 | 训练放到 Linux 的 3090 上（450M 模型的 LoRA 也可以在 Mac 上用 MPS 先试） |

**说明**：tickets 里写的"🐧 Linux"环境标记，在这个方案下大部分都可以在 Mac 上完成，只有 T-021 和 T-022 的训练建议用 Linux GPU。

---

## 14. 待核实 / 待决定事项

| # | 事项 | 何时解决 |
|---|---|---|
| Q1 | llama.cpp 的 iOS xcframework 是否包含 mtmd；mtmd 作为库的 CMake target 怎么用 | T-000 |
| Q2 | App 实际可用内存；mmap 的权重在 Metal 下怎么计入 footprint | T-000 |
| Q3 | LFM2.5-VL 和 Qwen3.5 在 Metal 后端上的支持和速度 | T-000 |
| Q4 | increased-memory-limit 权限和真机调试对开发者账号的要求 | T-000 |
| Q5 | LFM 许可证的商用条件（学习阶段没有影响；如果要商用，默认档改用 Qwen3.5-0.8B 或 InternVL3-1B） | 产品化之前 |
| Q6 | SPAQ、PARA、U²-Net、MobileCLIP 的许可证 | T-003、T-006、T-007 |
| Q7 | MobileCLIP 的官方预处理（尺寸和归一化） | T-006 |
| Q8 | 云端 judge 和 teacher 是否允许把图片发出去 | T-010、T-022 |
| Q9 | 是否把小模型也迁到 ggml，只保留一个推理运行时（减小包体积） | M3 之后，写 ADR |

---

## 15. 项目约定（换机器后不要丢）

- **知识沉淀**：对话中产生的可复用知识（原理、用法、踩坑、选型结论），蒸馏后追加到上级目录的 `LEARN.md`（Linux 上是 `~/CX/LEARN.md`；Mac 上如果没有，就在仓库外建一个同名文件，以后再合并）。格式：
  ```markdown
  ## [YYYY-MM-DD] 主题
  **标签**: tag1, tag2
  一句话核心结论。
  - 要点
  ```
  写之前先读，已有同类条目就更新，不要重复追加。
- **ADR**：选型和取舍写进 `docs/decisions/NNNN-*.md`（背景、选项、决定、后果）。
- **Tickets**：完成时满足"验收标准"里的每一项；修改 SPEC 时要升版本号，并在开头写清楚变更点。
- **提交**：git 提交信息用英文，前缀用 `docs:`、`core:`、`ml:`、`ios:`、`spike:` 等。
- **大文件**：模型、数据集、构建产物都不进入 git（`.gitignore` 由 T-001 建立）；模型的来源、文件名和 SHA256 记录在 ADR 或 `scripts/fetch-models.sh` 里。
