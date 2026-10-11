# T-005 S2 信号指标（Python 原型 + 规范）
**环境**: 🐧 Linux · **依赖**: T-002 · **规模**: M · **里程碑**: M1 · **状态**: 待开始

## 背景
不依赖模型的确定性指标，见 SPEC §4 S2。这份 Python 实现同时是 T-013 C++ 移植的**规范**，所以算法要简单、清楚、能复现，**不能用 OpenCV 里不好复现的函数**（研究时可以拿 OpenCV 做对照）。

## 交付物
- `ml/photocritic/signals.py`，输入为 sRGB uint8、长边 1024 的图像，以及可选的显著区域 mask。**输出的特征名、单位和定义以 `schema/features.md` 为准**（S2 部分），下面是概要：
  - 曝光：亮度直方图（256 bins，Rec.709 luma）、`clip_high`、`clip_low`、`mean_L`、`p5`、`p95`，显著区域内的同一组指标
  - 清晰度：`lap_var_global`、`lap_var_salient`、`grad_anisotropy`（运动模糊方向性）、`noise_sigma`（平坦块的 MAD）
  - 色彩：中性像素的 Lab a*/b* 均值、灰世界 CCT 估计、`sat_mean`、`sat_clip`
  - 水平线：Sobel 边缘 → Hough 直线 → 取长度加权的近水平主方向，输出 `horizon_angle_deg` 和 `horizon_conf`
  - 缩放：自己实现 area 插值（C++ 端照样实现），不用 PIL 的 resize
- 每个函数的 docstring 写清楚精确公式、颜色空间转换、卷积核和边界处理
- `schema/fixtures/signals_golden/`：10 张图（输入用 PNG，避免不同 JPEG 解码器的差异）及其指标输出 JSON，作为跨端对齐的基准
- 单元测试：合成纯色图、棋盘格、已知模糊核、已知旋转角度等，指标结果应该符合预期

## 验收标准
- [ ] 在 synthetic dev 集上，每个指标和对应的退化强度单调相关（Spearman |ρ| > 0.8）；水平线角度误差中位数 ≤ 0.5°
- [ ] golden 输出已提交
