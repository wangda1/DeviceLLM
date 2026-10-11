# T-003 评测数据集 v0
**环境**: 🐧 Linux · **依赖**: T-001 · **规模**: M · **里程碑**: M0

## 背景
没有数据就没法评测。见 SPEC §7.1。数据本身不进仓库，只提交脚本和 manifest。

## 交付物
- `ml/data/README.md`：每个数据集的来源、**许可证结论**和下载方式
- `ml/data/synthetic.py`：从一批质量好的原图生成可控退化样本，退化类型和标签的对应关系：
  - 曝光 ±0.7/1.3/2 EV → `overexposed`/`underexposed`/`highlights_clipped`/`shadows_crushed`
  - 对比度压缩 → `low_contrast`
  - 色温/色调偏移（在线性空间里乘通道增益）→ `warm_cast`/`cool_cast`/`green_tint`/`magenta_tint`
  - 饱和度 ×0.4 / ×1.8 → `dull_colors`/`oversaturated`
  - 高斯模糊 / 运动模糊核 / 泊松+高斯噪声 → `out_of_focus`/`motion_blur`/`high_noise`
  - 旋转 2–8° → `tilted_horizon`；偏移裁切 → `subject_cut_off`
  - 每张输出 `labels.jsonl`：`{path, source, degradations:[{type, param}], expected_tags}`
- 干净原图的来源（选一个，许可证要清楚）：例如 SPAQ 或 PARA 中高分的子集，或 Unsplash Lite
- `ml/data/fetch_spaq.py`、`ml/data/fetch_para.py`：下载并转换成统一的 manifest（如果许可证不允许自动下载，就写成手动步骤加校验脚本）
- golden 集：先建好目录和标注格式 `golden/labels.jsonl`（标签 + 人工写的参考建议），初始 ≥ 30 张，后续补到 100 张
- 数据集划分：固定 `dev` / `test`（seed 固定，test 集禁止用来调参）

## 验收标准
- [ ] 一条命令生成 synthetic 数据（≥ 2000 张，dev/test 按 7:3 划分）
- [ ] SPAQ 和 PARA 的 manifest 可以生成，并给出各属性分的分布统计
- [ ] README 里写明许可证结论
