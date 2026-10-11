# 特征清单（features v1）

S4 融合规则、`issues[].evidence`、模板参数只能引用这里列出的特征名。新增或改名时更新本文件，并在 `assets/rules.yaml` 里升 `features_version`。

**通用约定**
- 所有特征都是 `double`。不适用时取 **NaN**（例如没有主体时的 `*_salient`）。规则 DSL 中任何与 NaN 的比较都为 false，详见 [rules_dsl.md](rules_dsl.md)。
- 布尔特征用 0 / 1 表示。
- 图像量都在 S0 生成的 `analysis` 图（长边 1024，sRGB uint8）上计算。
- "mask"指显著性 mask（U²-Netp 输出经 min-max 归一化后，阈值 0.5，只保留最大连通域）；"背景"指 mask 之外的区域。
- 精确算法见 `ml/photocritic/signals.py` 等文件的 docstring（实现 T-005 / T-007 时写入）；本表只给定义和单位。

## S1 检测模型

| 名称 | 单位 / 值域 | 定义 |
|---|---|---|
| `has_subject` | 0/1 | mask 面积 ≥ 图像面积的 0.5%，且 `saliency_contrast` ≥ 0.15 |
| `subject_area` | 0–1 | mask 面积 / 图像面积 |
| `subject_cx`, `subject_cy` | 0–1 | mask 质心（归一化坐标） |
| `subject_thirds_dist` | 0–1 | 质心到最近三分线交点的距离 / 图像对角线长度 |
| `subject_center_dist` | 0–1 | 质心到图像中心的距离 / 图像对角线长度 |
| `subject_touch_edges` | 0–4 | mask 的 bbox 贴住图像边缘（距离 < 1% 边长）的边数 |
| `saliency_contrast` | 0–1 | mask 内显著性均值 − mask 外显著性均值 |
| `face_count` | 整数 | score ≥ 0.6 的人脸数 |
| `face_max_area` | 0–1 | 最大人脸 bbox 面积 / 图像面积；无人脸时为 0 |
| `face_cut_off` | 0/1 | 任一人脸的 bbox 超出图像边界，或者被边缘截断超过 15% |
| `face_lap_var` | 同 `lap_var_global` | 最大人脸区域内的拉普拉斯方差；无人脸时为 NaN |
| `face_mean_L` | 0–255 | 最大人脸区域内的 luma 均值；无人脸时为 NaN |

## S2 信号指标

| 名称 | 单位 / 值域 | 定义 |
|---|---|---|
| `mean_L` | 0–255 | 全图 luma 均值，`Y = 0.2126R + 0.7152G + 0.0722B`（在 sRGB 编码值上计算） |
| `p5`, `p95` | 0–255 | luma 的 5% / 95% 分位数 |
| `dyn_range` | 0–255 | `p95 − p5` |
| `clip_high` | 0–1 | `Y ≥ 250` 的像素占比 |
| `clip_low` | 0–1 | `Y ≤ 5` 的像素占比 |
| `mean_L_salient`, `clip_high_salient`, `clip_low_salient` | 同上 | 只在 mask 内计算 |
| `mean_L_background` | 0–255 | 只在背景区域计算 |
| `backlit_delta` | −255–255 | `mean_L_salient − mean_L_background` |
| `lap_var_global` | ≥ 0 | luma/255 → 3×3 拉普拉斯核 `[0,1,0;1,-4,1;0,1,0]`（replicate 边界）→ 方差 |
| `lap_var_salient`, `lap_var_background` | ≥ 0 | 分别在 mask 内和背景区域计算 |
| `focus_ratio` | ≥ 0 | `lap_var_salient / max(lap_var_background, 1e-6)` |
| `grad_anisotropy` | 0–1 | 在 Sobel 梯度幅值前 20% 的像素上求结构张量的特征值，`(λ1−λ2)/(λ1+λ2)` |
| `grad_direction_deg` | 0–180 | 结构张量主方向（运动模糊方向） |
| `noise_sigma` | 0–255（8-bit 单位） | 8×8 分块，取梯度能量最低的 10% 块，块内对 luma 做拉普拉斯残差，`MAD/0.6745`，再对各块取中位数 |
| `edge_density_background` | 0–1 | 背景区域内 Sobel 幅值 > 0.1（luma/255 尺度）的像素占比；没有主体时在全图上计算 |
| `flat_ratio` | 0–1 | 8×8 块中梯度能量低于阈值（luma/255 尺度下块内平均幅值 < 0.02）的块所占比例 |
| `lab_a_neutral`, `lab_b_neutral` | Lab 单位 | 中性像素（`C* < 15` 且 `20 < L* < 90`）的 a*、b* 均值；sRGB → 线性 → XYZ(D65) → Lab |
| `neutral_ratio` | 0–1 | 中性像素占比；< 0.03 时 `lab_*_neutral` 和 `cct` 取 NaN（不可靠） |
| `cct` | K | 中性像素的线性 RGB 均值 → XYZ → xy → McCamy 公式 |
| `sat_mean` | 0–1 | HSV 的 S 通道均值 |
| `sat_clip` | 0–1 | `S > 0.95 且 V > 0.2` 的像素占比 |
| `horizon_angle_deg` | −15–15 | Hough 峰值对应的角度；正值表示右端偏高 |
| `horizon_abs_deg` | 0–15 | `|horizon_angle_deg|` |
| `horizon_conf` | 0–1 | 峰值角 ±0.5° 内的票数 / 总票数，再除以 0.3，最后截断到 1 |

## S3 MobileCLIP

| 名称 | 值域 | 定义 |
|---|---|---|
| `clip_<tag>` | 0–1 | 每个标签的零样本分数（23 个，例如 `clip_overexposed`）：K 对 prompt 的 `softmax(100·[cos_pos, cos_neg])[0]` 的平均值。阶段二改为训练头的输出，名称不变 |
| `scene_<name>` | 0–1 | 每个场景类别的 softmax 概率（例如 `scene_night`） |
| `clip_utility` | 0–1 | `scene_screenshot + scene_document` |

## EXIF

| 名称 | 定义 |
|---|---|
| `exif_iso` | ISO；缺失时为 NaN |
| `exif_exposure_time_s` | 快门时间（秒）；缺失时为 NaN |
| `exif_exposure_bias_ev` | 曝光补偿；缺失时为 NaN |
