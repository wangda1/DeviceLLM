# 融合规则 DSL 规范（rules v1）

`assets/rules.yaml` 是规则的唯一来源，构建时转换成 `assets/rules.json`（内容等价）。Python（`ml/photocritic/fusion.py`）和 C++（`core/src/fusion/`）各实现一个解释器，**行为必须完全一致**。DSL 刻意保持很小：不允许任意表达式，不允许调用函数，也不允许引用其他规则。

## 1. 顶层结构

```yaml
version: 1                 # DSL 版本
features_version: 1        # 对应 schema/features.md
min_confidence: 0.5        # 置信度低于它的 issue 丢弃
max_issues: 6
rules: [Rule, ...]         # 每个标签恰好一条
suppress: [Suppress, ...]  # 互斥 / 压制关系
intentional: [Intentional, ...]
scores: Scores
crop: Crop
```

## 2. 条件 `Cond`

```yaml
{all: [Cond, ...]}             # 全部为真；空列表为真
{any: [Cond, ...]}             # 任一为真；空列表为假
{not: Cond}
{f: <feature>, gt: <number>}   # 比较运算符：gt, gte, lt, lte, eq, ne（每个节点只能有一个）
{scene: <scene_name>}          # 该场景在 report.scene 中为真（top-2，且概率 > 0.3）
```
- **NaN 规则**：特征为 NaN 时，任何比较（包括 `ne`）都为 **false**。所以 `{not: {f: x, gt: 1}}` 在 x 为 NaN 时为 true，写规则时要注意。
- 特征名不存在时视为配置错误：加载阶段就要报错，不能在运行时静默处理。

## 3. 数值函数 `Value`

```yaml
{const: <number>}
{piecewise: {f: <feature>, points: [[x1, y1], [x2, y2], ...]}}
{max: [Value, ...]}
{min: [Value, ...]}
```
- `piecewise`：在相邻点之间线性插值，超出两端时取端点的 y 值。`x` 必须严格单调（递增或递减都可以，加载时检查）；`x` 为 NaN 时结果为 0。
- `confidence` 的结果最后截断到 [0, 1]。

## 4. 严重程度 `Severity`

```yaml
{f: <feature>, thresholds: [t1, t2], dir: asc}   # asc：v < t1 → low；v < t2 → medium；否则 high
{f: <feature>, thresholds: [t1, t2], dir: desc}  # desc：v > t1 → low；v > t2 → medium；否则 high
{const: low|medium|high}
```
特征为 NaN 时取 `low`。

## 5. 规则 `Rule`

```yaml
- tag: highlights_clipped          # 必须是 taxonomy.yaml 中的标签；每个标签恰好一条规则
  dimension: exposure              # 必须和 taxonomy 一致
  when: Cond                       # 为 false 时不产生 issue
  confidence: Value
  severity: Severity
  evidence: [feature, ...]         # 写入 issues[].evidence 的特征（值为 NaN 的特征不写入）
  scene_overrides:                 # 可选；按列表顺序检查，第一个命中的场景生效
    - scene: night
      when: Cond                   # 可选，覆盖同名字段
      confidence: Value            # 可选
      severity: Severity           # 可选
```

## 6. 压制 `Suppress`

```yaml
- if: motion_blur          # 如果这个标签存在
  drop: [out_of_focus]     # 就删除这些标签
```
在所有规则执行完之后，**按列表顺序**执行一遍（不迭代到收敛）。

## 7. 有意为之 `Intentional`

```yaml
- name: silhouette
  when: Cond
  marks: [underexposed, shadows_crushed, backlit_subject]   # 这些 issue 的 intentional_possible 设为 true
```
issue 仍然保留；被标记的 issue 的扣分在计算维度分数时乘以 `scores.intentional_factor`。

## 8. 分数 `Scores`

```yaml
scores:
  sev_w: {low: 0.5, medium: 1.0, high: 1.6}
  intentional_factor: 0.3
  weights: {<tag>: <扣分基数>, ...}    # 23 个标签都要有
  overall: {exposure: 0.3, composition: 0.3, color: 0.2, sharpness: 0.2}
```
- `dim_score = round(clamp(100 − Σ_{该维度的 issue} weights[tag] × sev_w[severity] × confidence × (intentional ? intentional_factor : 1), 0, 100))`
- `overall = round(Σ overall[d] × dim_score[d])`
- 只计算经过 `min_confidence` 过滤和 `suppress` 处理之后的 issue；`max_issues` 截断只影响报告输出，**不影响分数**。

## 9. 裁剪 `Crop`

```yaml
crop:
  enabled_when: Cond            # 例如 has_subject 且（unbalanced 或 excess_empty_space 或 subject_too_small 的特征条件）
  min_area: 0.7                 # 裁剪后面积不小于原图的 70%
  keep_aspect: true
```
算法：保持原图比例，在面积 ≥ `min_area` 的候选裁剪框中，选使主体质心离最近的三分线交点最近的那一个（面积步长 0.05，位置在可行范围内按 1% 网格搜索），不能超出图像边界。条件不满足时输出 null。

## 10. 执行顺序

1. 对每条规则：先确定生效的 scene override → 计算 `when` → 计算 confidence 和 severity → 置信度 ≥ `min_confidence` 的产生 issue（`source: rules`）
2. 执行 `suppress`
3. 执行 `intentional` 标记
4. 计算 scores
5. 排序：按 `sev_w[severity] × confidence` 降序；相同时按标签在 taxonomy 中的顺序
6. 截断到 `max_issues`
7. 计算 crop
