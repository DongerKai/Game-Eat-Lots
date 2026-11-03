# 菜品数值（Dish Values）

> **Status**: Approved (2026-11-03)
> **Author**: user + agents
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 每口都是算术题；没有白吃的菜；数字看得见地涨

## Summary

菜品数值定义《回本》每道菜的价格与饱腹值，并把两者合成唯一可用于跨菜比较的单位价值（元/格）。局内只把价格与饱腹交给 HUD，比值留给玩家心算；结算页才展示单位价值，验证这一局算得对不对。它同时提供设计期与 CI 的菜单平衡审查：单位价值跨度、腻度组内的严格支配、以及容量与回合约束下的最优餐价值。没有它，玩家无法在同一把尺子上比较菜品，支柱「每口都是算术题」就没有对象。

> **Quick reference** — Layer: `Core` · Priority: `MVP` · Key deps: `数据配置`

## Overview

菜品数值是《回本》里每道菜的数值定义与派生计算层。它定义 MVP 的 12 道菜——价格、饱腹值、口味标签、腻度分组——并从价格与饱腹这两个原始数字推出本作唯一的跨菜比较指标：单位价值（元/格）。

对玩家，它提供的是"这口值不值"的原料：局内菜品卡只露出价格与饱腹，比值要玩家自己算；结算页才把单位价值摊开，验证这一局算得对不对。对下游系统，它是一组只读查询与纯函数：给 dish id 返回原始值与派生值，并回答"这个菜单里有没有被严格支配的菜"。

没有这一层，菜只是名字和图标，玩家无从比较，支柱「每口都是算术题」就没有对象；没有单位价值这个统一刻度，价格与饱腹也无法放在同一把尺子上量。

## Detailed Design

### Core Rules

1. **职责边界**：数据配置拥有 `DishData` 的 schema 与合法性校验；本系统拥有菜品的**数值内容**（12 道菜的具体价格、饱腹）与**派生公式**。合法不等于平衡——记录能加载，不代表这组数字好玩。
2. **原始值只读、不缓存**：`price` 与 `satiety` 一律通过 `数据配置.get_dish(id)` 读取。本系统不修改记录，也不持有副本或缓存；每次查询都委托给数据配置。可观测行为：连续两次查询返回同一个只读实例（`is_same()` 为 true），且对返回记录的任何赋值都被拒绝——与数据配置 AC 19 一致，本系统不提供绕过该不可变性的路径。
3. **单位价值是唯一比较刻度**：对每道菜定义 `unit_value(d) = price(d) / satiety(d)`，单位「元/格」。玩家跨菜比较时只需要这一个数；价格与饱腹单独看都不构成跨菜比较。
4. **比较不看表现层**：排序与取舍只使用 `unit_value`、`cloying_group`、`taste_tags`。`display_name`、`icon`、`station_id` 不参与任何数值判定。
5. **局内不显示单位价值**：菜品卡在局内只显示价格与饱腹两个原始数字。若系统替玩家算出比值，算术题会退化成排序题，违反支柱 1。单位价值在结算页才展示，作为"这局算得对不对"的验证。
6. **首局例外**：第一局的结算页额外展示每道菜的单位价值与一处替换建议。该教学流程由「首局引导」细化，本系统只负责提供数据。
7. **数值带**：`satiety` 取 1–6，且 MVP 档口菜池必须同时覆盖 1 格、2–4 格、≥5 格三档。只有小份、常规、大份并存，容量约束才会真正咬合；如果所有菜都是 1 格，最优解退化成"挑最贵的 10 道"，除法就成了装饰。`price` 为 int。MVP 菜单内单位价值跨度必须 ≥ 1.5 元/格。
8. **严格支配审查**：同一 `cloying_group` 内，若存在菜 B 满足 `price(B) ≥ price(A)` 且 `satiety(B) ≤ satiety(A)`（至少一项严格），则 A 被严格支配；同组腻度对两者的惩罚相同，A 永远不该被端上桌，这是设计错误。**两个空 `cloying_group` 视为同组**——它们都没有腻度惩罚，支配关系真实存在。跨组允许支配，但必须能为 A 写出一条用途，且用途必须落在三类之一：`腻度错峰` / `口味路线` / `容量精度`；写不出或不属于三类，即标记待复核。
9. **内容冗余**：两道菜 `price` 与 `satiety` 完全相同是允许的，但它们必须在 `cloying_group` 或 `taste_tags` 上至少有一项不同；否则审查报告记 warning——两道完全等价的菜只会让菜单变长，不会让决策变多。`taste_tags` 按**集合**比较：顺序不同或重复标签不算差异。
10. **菜单健康度**：一个档口的 `dish_pool` 只有同时满足「UV 跨度 ≥ 1.5」「同组严格支配对数 = 0」「同时覆盖 1 格 / 2–4 格 / ≥5 格三档」才算可发布菜单。**`menu_healthy` 只作用于档口菜池，不作用于单次刷新菜单**——要求每次刷新都覆盖全部档位会迫使大份菜每波必出；单次刷新的门槛是 `opt_value` 偏差（见 Formulas 末节）。该审查是设计期与 CI 期的检查，不阻断游戏启动——平衡问题不应该让玩家看到错误页。
11. **不聚合**：本系统在运行时不计算一局的总价值、净利或最优解。它只提供单菜的 `price` 与 `unit_value`；求和由「结算与入场费」在 int64 中完成。`opt_value`（见 Formulas）只用于设计期与 CI 的平衡审查，不进入游戏运行时——把最优解交给玩家会把静态菜单变成一个背下来的常数。
12. **确定性**：给定同一份记录集，所有派生值与审查结果恒定。任何需要排序的输出以 `uv_cmp` 降序为主键、`id` 字典序为次键，不依赖哈希、字典遍历顺序或浮点比较。

### MVP 菜品表（食堂档口）

`station_canteen` 的 12 道菜。所有数值进入 `DishData` 后不得在本表之外重复定义。

| id | 菜名 | price | satiety | unit_value | taste_tags | cloying_group | 设计作用 |
|---|---|---|---|---|---|---|---|
| `dish_white_rice` | 白米饭 | 4 | 1 | 4.00 | — | 主食 | 容量精度 |
| `dish_cucumber` | 拍黄瓜 | 5 | 1 | 5.00 | 酸 | 蔬菜 | 腻度错峰 |
| `dish_cola` | 可乐 | 6 | 1 | 6.00 | 甜, 冰 | 饮料 | 腻度错峰 |
| `dish_ice_cream` | 冰淇淋 | 7 | 1 | 7.00 | 甜, 冰 | 甜 | 容量精度 |
| `dish_greens` | 炒青菜 | 6 | 2 | 3.00 | 鲜 | 蔬菜 | 腻度错峰 |
| `dish_tomato_egg` | 番茄炒蛋 | 8 | 2 | 4.00 | 酸, 鲜 | 蛋 | 口味路线 |
| `dish_mapo_tofu` | 麻婆豆腐 | 12 | 2 | 6.00 | 辣, 鲜, 油 | 辣 | 口味路线 |
| `dish_fried_chicken` | 炸鸡腿 | 13 | 2 | 6.50 | 油, 鲜 | 油 | 容量精度 |
| `dish_steamed_fish` | 清蒸鱼 | 15 | 3 | 5.00 | 鲜 | 鲜 | 口味路线 |
| `dish_braised_pork` | 红烧肉 | 16 | 3 | 5.33 | 甜, 油 | 肉 | 容量精度 |
| `dish_sweet_sour_pork` | 糖醋里脊 | 18 | 4 | 4.50 | 甜, 酸, 油 | 肉 | 容量精度 |
| `dish_seafood_pot` | 豪华海鲜锅 | 20 | 6 | 3.33 | 鲜, 油 | 鲜 | 高价低比值陷阱 |

**表内约定**：

- **价格是得分，不是成本。** 自助餐已付入场费，局内没有预算约束；因此低价本身不是优点，低单位价值才是缺点。任何"便宜"的菜必须靠打包精度或腻度错峰证明自己的位置。
- **同组无严格支配**：每组内没有任何菜同时更贵且更省容量（已逐组核对）。**跨组支配**存在，且每一条都能在「设计作用」列写出用途；用途必须属于 `腻度错峰` / `口味路线` / `容量精度` 三类。
- **比值最优的 10 盘组合**：冰淇淋、炸鸡腿、可乐、麻婆豆腐、红烧肉、拍黄瓜、清蒸鱼、糖醋里脊、番茄炒蛋、白米饭——饱腹合计 20 格，价格合计 104 元。这是校准入场费与初始胃容量的基准（初始容量按 20 格、10 回合计算）。
- **按价格排序的贪心只有 94 元**，比最优低 9.6%，且只吃 6 盘。除法在这个容量下是必要的，不是装饰。
- **跨组支配的前提是菜单波次会限制可见集合。** 如果一次刷新就把 12 道菜全部给玩家，同饱腹档里最弱的那道就没有存在理由（例如白米饭永远输给可乐和冰淇淋）。菜单波次必须保证每波只暴露子集；否则本表的支配审查结论不成立。
- `cloying_group` 的归属是本表的决策，惩罚曲线与叠加规则由「腻度与口味」拥有；本表不定义任何腻度系数。
- **≥5 格档只有豪华海鲜锅一道。** 删除或重调它会静默让所有菜池失去档位覆盖、`menu_healthy` 全变 false；内容作者改动它之前必须先补一道同档菜。
- 数值为第一版。平衡是否成立由 Formulas 中的审查工具与 `/prototype` 决定，不以"看起来合理"结案。

### States and Transitions

本系统没有内部状态机。它不加载、不缓存、不持有可变状态；每次查询都是对数据配置只读记录的纯函数计算。

| 条件 | 行为 |
|---|---|
| 数据配置尚未 `Ready` | 查询返回错误，并带调用者名与 dish id；不返回 0、空值或占位菜 |
| 数据配置 `Ready` | 查询恒可用，派生值只读 |
| 记录通过合法性校验但未通过平衡审查 | 不影响可用性；只写审查报告，不阻断启动 |
| 游戏退出 | 无清理动作 |

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 数据配置 | 上游 | `get_dish(id)` → `price`、`satiety`、`taste_tags`、`cloying_group`；查询前提为 `Ready` | 硬 |
| 腻度与口味 | 下游 | `unit_value(d)`、`satiety(d)`、`cloying_group(d)` | 硬 |
| 菜单波次 | 下游 | 菜池审查：`uv_span(M)`、`same_group_dominated_pairs(M)`；单次刷新审查：`opt_value(M, C, T)` | 硬（Vertical Slice 期） |
| 结算与入场费 | 下游 | 单菜 `price(d)` 与 `unit_value(d)`；求和由结算在 int64 中完成 | 硬 |
| 局内 HUD | 下游 | `price(d)`、`satiety(d)`；局内不请求 `unit_value` | 硬 |
| 首局引导 | 下游 | 结算期的 `unit_value` 讲解数据 | 软 |

**接口所有权**：本系统拥有派生公式、数值带与平衡审查；数据配置拥有原始记录与合法性校验。下游不得自行重算 `unit_value`——重算会让两处公式漂移。

**查询错误契约**：公开查询（`unit_value(id, caller)`、`uv_cmp(a, b, caller)`、`get_dish(id, caller)`）都接受调用者名。失败时返回 `QueryError { caller: StringName, dish_id: StringName, reason: enum }`，`reason ∈ {DATA_NOT_READY, UNKNOWN_DISH, INVALID_SATIETY}`；不返回 0、空记录或占位菜。返回类型是 Result 风格，不抛未捕获异常。

## Formulas

### 单位价值 `unit_value`

The `unit_value` formula is defined as:

`unit_value(d) = float64(price(d)) / float64(satiety(d))`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `d` | DishData | — | 数据配置 `get_dish(id)` | 被评估的菜 |
| `price(d)` | int | 0–INT32_MAX | `DishData.price` | 吃下即计入得分 |
| `satiety(d)` | int | 1–INT32_MAX | `DishData.satiety` | 占用的容量格；0 在上游非法 |
| `unit_value` | float64 | 0–+∞ | calculated | 每格得分（元/格） |

**Output Range:** `[0, price_max]`；MVP 菜单的常态区间是 3.0–7.0。`price = 0` 时恒为 `0.0`。展示值由 `display_uv(uv) = floor(uv × 100 + 0.5) / 100` 得出，固定两位小数；**展示值不参与任何玩法判定**。

**Example:** 红烧肉 `16 / 3 = 5.333…`，结算页显示 `5.33`；清蒸鱼 `15 / 3 = 5.00`，显示 `5.00`。

### 精确比较 `uv_cmp`

浮点除法不能用来判定两菜相等。排序、支配与审查一律用整数交叉相乘：

The `uv_cmp` formula is defined as:

`uv_cmp(a, b) = sign( price(a) × satiety(b) − price(b) × satiety(a) )`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `a`, `b` | DishData | 不同 id | 菜单集合 | 待比较的两道菜 |
| `uv_cmp` | int | {-1, 0, 1} | calculated | -1 表示 a 更低，0 表示精确相等 |

**Output Range:** `{-1, 0, 1}`。返回 0 是精确相等，不是四舍五入造成的并列；并列时以 `id` 字典序升序打破。`INT32_MAX² ≈ 4.6×10¹⁸`，小于 int64 上限 `9.22×10¹⁸`，乘积不会溢出。

**Example:** 红烧肉 vs 清蒸鱼：`16×3 − 15×3 = +3` → 红烧肉更高；清蒸鱼 vs 另一道 `20/4`：`15×4 − 20×3 = 0` → 精确并列。

### 菜单跨度与健康度 `uv_span` / `menu_healthy`

The `uv_span` formula is defined as:

`uv_span(M) = uv_max(M) − uv_min(M)`

其中 `uv_max` / `uv_min` 由 `uv_cmp` 选出，跨度用两者的精确分数在 float64 中相减。

The `menu_healthy` formula is defined as:

`menu_healthy(M) = (uv_span(M) ≥ 1.5 − 1e-9) ∧ (same_group_dominated_pairs(M) = 0) ∧ tier_coverage(M)`

`tier_coverage(M) = (∃d∈M: satiety(d) = 1) ∧ (∃d∈M: 2 ≤ satiety(d) ≤ 4) ∧ (∃d∈M: satiety(d) ≥ 5)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `M` | id 集合 | 0–N 道菜 | 档口 `dish_pool`（**不含单次刷新菜单**） | 被审查的菜池 |
| `uv_span` | float64 | 0–+∞ | calculated | 菜池内最高与最低单位价值之差 |
| `tier_coverage` | bool | {true, false} | calculated | 是否同时覆盖 1 格、2–4 格、≥5 格三档 |
| `menu_healthy` | bool | {true, false} | calculated | 菜池是否可发布 |

**Output Range:** `uv_span ≥ 0`；`menu_healthy ∈ {true, false}`。`|M| = 0` 返回 `uv_span = 0`、`menu_healthy = false`，不抛错。跨度低于 1.5 意味着菜单里最好的菜和最差的菜看起来差不多，玩家做的除法不改变答案——支柱 1 就没有可验证的对象。`1e-9` 的容差用于吸收形如 `16/3 − 23/6` 的精确 1.5 在 float64 中算成 `1.4999999999999996` 的误差；容差是固定常量，不是可调参数。

**Example:** 当前 12 道菜池：最高 7.00（冰淇淋），最低 3.00（炒青菜），跨度 4.00，同组支配 0 对；satiety 覆盖 1、2–4、6 三档 → `menu_healthy = true`。

### 严格支配 `dominates`

The `dominates` formula is defined as:

`dominates(B, A) = same_group(A, B) ∧ price(B) ≥ price(A) ∧ satiety(B) ≤ satiety(A) ∧ (price(B) > price(A) ∨ satiety(B) < satiety(A))`

其中 `same_group(A, B) = (cloying_group(A) = cloying_group(B))`，**两个空串视为同组**。

The `unjustified_dominated` formula is defined as:

`unjustified_dominated(M) = |{ (A, B) ∈ M² : A ≠ B ∧ ¬same_group(A, B) ∧ dominates_raw(B, A) ∧ purpose(A) ∉ {腻度错峰, 口味路线, 容量精度} }|`

`same_group_dominated_pairs(M) = |{ (A,B) ∈ M² : A ≠ B ∧ dominates(B,A) }|`。支配是反对称的（严格性使两个方向不可能同时为真），所以有向计数与无向对数相同；这里采用有向定义，便于逐对报告。

`dominates_raw` 是去掉 `same_group` 条件后的同一谓词；`purpose(A)` 取自菜品表的「设计作用」列。该列是设计评审清单，不是 `DishData` 字段——CI 能自动算 `same_group_dominated_pairs` 与 `cross_dominated_pairs`，`unjustified_dominated` 需要评审时人工核对用途（见 Open Question 7）。

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `A`, `B` | DishData | 不同 id | 菜单集合 | 有向对：B 是否使 A 失去存在理由 |
| `same_group_dominated_pairs(M)` | int | 0–n(n−1) | calculated | 同组严格支配对数；必须为 0 |
| `cross_dominated_pairs(M)` | int | 0–n(n−1) | calculated | 跨组严格支配对数；允许非 0 |
| `unjustified_dominated(M)` | int | 0–n(n−1) | calculated | 跨组且写不出用途的对数；必须为 0 |

表中后三行是**审查计数**，不是 `dominates` 的输入；`dominates` 只消费 A、B 的 `price`、`satiety`、`cloying_group`。

**Output Range:** 三个计数均为 `[0, n(n−1)]`；`n = 12` 时最多 132 个有向对，O(n²)，可每次 CI 全量扫描。完全相同的 `price` 与 `satiety` 不构成支配（严格性使两个方向都为 false），由 Core Rule 9 的冗余检查处理。

**Example:** 炸鸡腿 `13/2` vs 麻婆豆腐 `12/2`：`13 ≥ 12`、`2 ≤ 2`、价格严格 → 炸鸡腿支配麻婆豆腐。两者跨组（油 / 辣），麻婆豆腐的用途是「口味路线」→ `unjustified_dominated` 不计入。若把麻婆豆腐的用途清空，审查报告标记待复核。

### 最优餐价值 `opt_value`（仅设计期 / CI）

The `opt_value` formula is defined as:

`opt_value(M, C, T) = max { Σ_{d∈S} price(d) : S ⊆ M, |S| ≤ T, Σ_{d∈S} satiety(d) ≤ C }`

递推：`V[i][c][t] = max( V[i−1][c][t], price_i + V[i−1][c−satiety_i][t−1] )`，其中 `c ≥ satiety_i`、`t ≥ 1`；边界 `V[·][c][0] = 0`、`V[0][c][t] = 0`、`V[i][c<0][t] = −∞`。

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `M` | id 集合 | ≤ 档口菜池 | data file | 参与计算的菜单 |
| `C` | int | ≥ 0 | 胃容量系统 / 平衡配置 | 容量格数 |
| `T` | int | ≥ 0 | 时间与回合系统 | 可用回合数 |
| `opt_value` | int64 | 0–Σprice | calculated | 该菜单在容量与回合限制下的最高得分 |

**Output Range:** `[0, Σ_{d∈M} price(d)]`。`M = ∅`、`C = 0` 或 `T = 0` 时返回 0。按 12 道菜、`C = 20`、`T = 10` 计算，状态数 2,400，滚动数组 O(C·T) 空间。**不进入运行时**——只在设计审查与 CI 中调用。

**假设**：每道菜至多选一次（0/1 背包），与「菜单波次」不重复抽取的规则一致。若未来允许重复选取，改为有界背包，递推需增加每道菜的重数维度。

**Example:** 当前 12 道菜池、`C = 20`、`T = 10` → `opt_value = 104`（冰淇淋、炸鸡腿、可乐、麻婆豆腐、红烧肉、拍黄瓜、清蒸鱼、糖醋里脊、番茄炒蛋、白米饭）。同条件按价格降序贪心只有 94，差 9.6%。

### 入场费校准比 `calibration_ratio`

The `calibration_ratio` formula is defined as:

`calibration_ratio = opt_value(M, C0, T) / entry_fee`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `C0` | int | ≥ 1 | 胃容量系统（初始值） | 初始容量格数 |
| `T` | int | ≥ 1 | 时间与回合系统 | 每局回合数 |
| `entry_fee` | int | ≥ 0 | `StationData.entry_fee` | 档口入场费 |
| `calibration_ratio` | float64 | (0, +∞) | calculated | 最优餐相对入场费的倍数 |

**Output Range:** 目标区间 `[1.15, 1.35]`。低于 1.15，一次约 10% 的心算失误就会抹掉全部利润，局外成长停滞，玩家会觉得自己因为尝试而被惩罚；高于 1.35，随便吃都能赚，除法不再承重。**「腻度与口味」已落地（2026-11-03）**：当前 12 道菜池存在零惩罚顺序（红烧肉 → 拍黄瓜 → 糖醋里脊 → 其余），`opt_value = 104` 仍然可达，因此区间与入场费不变。若未来菜单不存在零惩罚顺序，校准必须改用实际可达上限。

**Example:** `opt_value = 104`、`entry_fee = 85` → `calibration_ratio = 1.224`；最优玩家净利 +19，按价格贪心的玩家净利 +9。

> **满级带（2026-11-03）**：`[1.15, 1.35]` 是**未升级**校准带；满级容量 25 的上限 120 / 85 = 1.412 属于「局外升级」定义的满级带 `[1.35, 1.45]`。两者同时成立才算该档口校准通过。

### 审查产出的使用方式

- **CI**：每次改 `.tres` 后跑 `menu_healthy`、`same_group_dominated_pairs` 与 `opt_value`；这三项越界即失败。`cross_dominated_pairs` 只记录、不设门槛——跨组支配本来就允许，只要有用途。`unjustified_dominated` 不在 CI 中：`purpose` 不是数据字段，它是人工评审步骤（见 Open Question 7）。
- **设计期**：改入场费或初始容量后重算 `calibration_ratio`；菜单波次生成单次菜单时，其 `opt_value` 相对全池校准值的偏差不得超过 ±10%，否则该次刷新重掷。

## Edge Cases

- **If `price = 0`**：合法，`unit_value = 0.0`，显示 `0.00`。它通常只会给菜单增加一张零分卡，审查报告标 warning；除非它已在同组内被严格支配（那属于支配错误，不是零价格问题）。回合宽裕时它可能仍进入 `opt_value`（它只占容量与回合），这是允许的，但报告必须列出。
- **If `satiety = 0`**：上游已拒绝，但公式仍须断言并返回错误，绝不执行除法。
- **If 两个不同比值的展示值相同**（例如 5.331 与 5.334 都显示 `5.33`）：排序与支配一律走 `uv_cmp`，不得解析或比较展示字符串。展示值只用于告知，不用于决策。
- **If 两个比值精确相等**（例如 `15/3` 与 `20/4`）：`uv_cmp = 0`，以 `id` 字典序升序打破并列。这是并列，不是四舍五入的假象，报告中记为 tie。
- **If 两道菜的 `cloying_group` 都为空**：视为同组。若其中一道同时更贵且更省容量，即为严格支配，必须改数值或改分组。
- **If 两道菜 `price` 与 `satiety` 完全相同**：不构成支配；若 `cloying_group` 与 `taste_tags` 也完全相同，记 warning（内容冗余）；只要有一项不同，允许存在。
- **If 菜单为空 `|M| = 0`**：`uv_span = 0`、`menu_healthy = false`，不抛错、不返回默认菜。空菜单是数据错误，由数据配置在加载期拦截。
- **If 菜单只有 1 道菜**：`uv_span = 0`、`menu_healthy = false`。可以开始游戏（数据合法），但没有任何算术可做；这是可玩性缺陷，不是崩溃。
- **If 所有菜 `satiety = 1`**：容量永不成为约束，最优解退化为"挑最贵的 T 道"，除法变成装饰。`menu_healthy = false`（违反 Core Rule 7 的档位覆盖），禁止作为可发布菜单。
- **If 菜单内所有菜单位价值相同**：`uv_span = 0` → `menu_healthy = false`。这是"看起来很多选择、实际只有一个答案"的典型退化。
- **If 单次刷新菜单的 `opt_value` 偏离全池校准值超过 ±10%**：「菜单波次」应重掷该次刷新。本条是跨系统契约，不归本系统执行；本系统只提供计算。
- **If 玩家在局内连续选择同一 `cloying_group` 的菜**：本系统不做任何处理。惩罚系数、叠加顺序与上限全部属于「腻度与口味」；本系统只提供 `cloying_group` 与 `satiety`。
- **If `uv_cmp` 的乘积超出 int64**：仅当上游放宽 `price` / `satiety` 到 int64 全范围时才会发生。当前 schema 上限为 INT32_MAX，乘积约 `4.6×10¹⁸ < 9.22×10¹⁸`，安全；若未来放宽，必须改用 128 位整数或在数据配置加界，不得静默截断。
- **If 平衡审查失败但数据合法**：游戏照常启动，只写审查报告。把平衡问题变成启动错误，会让内容作者无法进入游戏调试。
- **If 菜单波次允许重复抽取同一道菜**：`opt_value` 的 0/1 背包假设失效，公式必须改为有界背包并更新本 GDD。当前假设与「不重复抽取」一致，跨系统改动必须先改这里。
- **If 数据配置尚未 `Ready` 或 dish id 不存在**：查询返回错误并带调用者与 id，不返回 0、空记录或占位菜。静默兜底会让数值 bug 变成平衡 bug。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 数据配置 | 硬 | `get_dish(id)` → `price`、`satiety`、`taste_tags`、`cloying_group`；查询前提为 `Ready` | GDD 已写 |
| 胃容量 | 软（仅设计期） | 初始容量 `C0` 作为 `opt_value` / `calibration_ratio` 的审计参数 | GDD 已写（`design/gdd/stomach-capacity.md`） |
| 时间与回合 | 软（仅设计期） | 每局回合数 `T` 作为 `opt_value` 的审计参数（来自 `StationData.turn_budget`） | GDD 已写（`design/gdd/time-and-turns.md`） |

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 腻度与口味 | 硬 | `unit_value(d)`、`satiety(d)`、`cloying_group(d)` | GDD 已写（`design/gdd/cloying-and-taste.md`） |
| 菜单波次 | 硬（VS） | 菜池：`uv_span(M)`、`same_group_dominated_pairs(M)`；单次刷新：`opt_value(M, C, T)` | GDD 已写（`design/gdd/menu-waves.md`） |
| 结算与入场费 | 硬 | `price(d)`、`unit_value(d)`；一局求和由结算在 int64 中完成 | GDD 已写（`design/gdd/settlement-and-entry-fee.md`） |
| 局内 HUD | 硬 | `price(d)`、`satiety(d)`；局内不请求 `unit_value` | GDD 已写（`design/gdd/in-game-hud.md`） |
| 首局引导 | 软 | 结算期 `unit_value` 讲解数据 | GDD 已写（`design/gdd/first-run-onboarding.md`） |

**双向一致要求**：以上每个系统在编写自己的 GDD 时，必须在 Dependencies 中列出「菜品数值」为上游依赖。数据配置的 GDD 已在上游侧列出本系统；其余系统仍待设计，此要求记录在此供后续核对。

**本系统对「菜单波次」的前置假设**：部分跨组支配菜（白米饭、炒青菜、可乐等）之所以存在，是因为单次刷新只暴露菜池的一个子集，它们在受限菜单里承担填充与错峰；如果菜单波次改为全池常显，这些菜会立即变成死选项，本表的支配审查结论失效。这是一条软依赖，需要在菜单波次 GDD 中确认。

**对外契约的稳定性**：`unit_value` 是跨系统契约，任何重定义（改分母、改取整规则）都必须先改本 GDD，再改下游；下游不得自行实现一份。

## Visual/Audio Requirements

本系统的全部可见输出都是数字，视觉要求围绕「读得清」而不是「好看」。

- **数字优先**：菜品卡上价格与饱腹的字号必须大于食物图标与菜名。测试沿用概念中的规则——一眼看不清数字就改布局。
- **等宽数字**：价格、饱腹、单位价值一律使用等宽字体，避免数字跳动时列宽变化。
- **单位价值颜色**：局内不出现；结算页用回本红标出单位价值与净利，用橄榄绿标出容量相关数字。不使用大面积暖黄——它会压低数字对比度。
- **零特效延迟**：数值更新不做缩放、弹跳、延迟入场。唯一允许的动画是结算页的计数滚动（≤ 0.6 秒，可跳过）。
- **音频**：本系统不直接触发音效。选菜音与结算音由「音效反馈」拥有；本系统只保证数值在同一帧可读，不要求音画同步的额外延迟。

> `art-director` not consulted — Lean mode（本系统是计算层，视觉呈现由局内 HUD 拥有；概念文档的视觉锚点已覆盖数字优先原则）。进入生产前需人工复核。

## Game Feel

- **同一帧结算**：玩家确认一道菜后，价格、饱腹与剩余容量的变化必须在同一帧内完成。心算被验证的快感依赖于"我按下按钮，数字立刻同意我"。
- **读数节奏由玩家控制**：没有倒计时、没有自动推进。界面不得用动画催促决策——支柱 3 明确压力来自剩余格数，不来自时间。
- **展示精度与实际精度一致**：结算页固定两位小数，显示精确比值的前两位，不做取整美化，也不截断成整数。玩家复盘时看到的数必须能和价格、饱腹对上。
- **差距可复盘**：结算页用净利与入场费的差说明本局是赚是亏，让失败归因到选菜而不是运气。不展示最优解上限——那会把菜单变成背下来的常数（Core Rule 11）。
- **失败要可归因**：玩家输掉一局后，应能指着某一道菜说出"这盘吃错了"。如果复盘时说不出错在哪，是数值空间或展示的问题，不是玩家的问题。

## UI Requirements

- **菜品卡字段**：名称、价格、饱腹、图标。价格与饱腹必须同屏可见，不得折叠进详情页或悬浮提示。
- **默认排序**：按档口陈列顺序（`dish_pool` 顺序）。不得默认按单位价值排序——那等于把答案直接交给玩家。
- **可选排序**：允许按价格、饱腹排序；不提供"按单位价值"排序或高亮筛选。
- **单位价值展示**：只在结算页出现，格式 `x.xx 元/格`，等宽字体；同一屏内不重复出现。
- **容量反馈**：选菜时剩余容量必须实时可见（由局内 HUD 提供，本系统只提供 `satiety`）。
- **无障碍**：数字与背景对比度 ≥ 4.5:1；字号放大到 150% 时菜品卡不换行错位。完整要求见未来的 `design/accessibility-requirements.md`。

## Cross-References

| 文档 | 关系 | 本系统引用/依据 |
|---|---|---|
| `design/gdd/data-config.md` | 上游 | `DishData` 字段定义、`Ready` 门、int64 求和规则、合法性/平衡职责划分 |
| `design/gdd/game-concept.md` | 来源 | MVP 12 道菜、四支柱、视觉锚点「食堂价目板」 |
| `design/gdd/systems-index.md` | 索引 | 优先级 MVP、层 Core、下游系统清单与设计顺序 |
| `design/gdd/input.md` | 无直接接口 | 选菜输入由场景/餐局流程转译，本系统不读取输入 |

**待回指**：所有列出本系统的下游 GDD 已完成并已回指。

## Acceptance Criteria

测试夹具放在 `tests/fixtures/dishes/` 与 `tests/fixtures/menus/`。`mvp` 指 `tests/fixtures/dishes/mvp_canteen_12.tres`（本 GDD 的 12 道菜表）。每条用例使用独立夹具。

**运行时（玩家可见行为）**

1. **GIVEN** `mvp`，**WHEN** 调用 `unit_value("dish_braised_pork")`，**THEN** 返回 `5.333333333333333 ± 1e-12`，不是取整后的字符串；`unit_value("dish_steamed_fish")` 返回精确 `5.0`，`unit_value("dish_white_rice")` 返回精确 `4.0`。
2. **GIVEN** `zero_price.tres`（price 0、satiety 2），**WHEN** 查询，**THEN** `unit_value = 0.0`。**GIVEN** 手工构造的 `DishData`（satiety 0，绕过数据配置目录、直接作为夹具传入），**WHEN** 调用 `unit_value`，**THEN** 返回错误对象，不发生除法、不产生 INF/NaN、不崩溃。这一条必须绕过目录，因为数据配置在加载期就会拒绝 satiety 0。
3. **GIVEN** `tie_pair.tres`（`dish_alpha` 15/3、`dish_beta` 20/4），**WHEN** 调用 `uv_cmp(alpha, beta)` 与 `uv_cmp(beta, alpha)`，**THEN** 两次都返回 `0`。
4. **GIVEN** 两个 float64 商在末位不同、但精确比值有明确大小关系的夹具，**WHEN** 调用 `uv_cmp`，**THEN** 返回精确的 `+1` 或 `−1`，等于 `sign(price_a × satiety_b − price_b × satiety_a)`——证明比较没有走浮点。
5. **GIVEN** `tie_pair.tres` 按单位价值排序，**WHEN** 连续调用两次，**THEN** 两次顺序完全一致，且 `dish_alpha` 在 `dish_beta` 前（id 升序打破并列）。
6. **GIVEN** 数据配置尚未 `Ready`，**WHEN** 以调用者 `"hud"` 查询任意 dish id，**THEN** 返回含调用者 `"hud"` 与 dish id 的结构化错误，绝不返回 `0`、`null` 或占位菜。
7. **GIVEN** id `"dish_missing"`，**WHEN** 查询，**THEN** 返回与第 6 条相同的错误结构，不伪造占位记录。
8. **WHEN** 逐项检查局内 HUD 的菜品卡查询结果，**THEN** 只包含 `price` 与 `satiety`；不存在 `unit_value` 字段，且用 spy 断言局内调用链从未调用 `unit_value`。
9. **GIVEN** 渲染局内菜品卡，**WHEN** 截图，**THEN** `production/qa/evidence/` 中留存的截图上只有价格与饱腹，卡片任何位置都不出现 `元/格`。
10. **GIVEN** 渲染结算页，**WHEN** 截图，**THEN** 红烧肉显示 `5.33`、清蒸鱼显示 `5.00`、豪华海鲜锅显示 `3.33`、冰淇淋显示 `7.00`，全部固定两位小数；截图留存。
11. **GIVEN** 对同一 dish id 连续两次查询，**WHEN** 比较两次返回的记录，**THEN** `is_same()` 为 true；**GIVEN** 尝试对返回记录的 `price` 赋值，**THEN** 操作被拒绝或抛错，再次读取仍为原值（与数据配置 AC 19 一致）。

**设计期 / CI（不进入运行时）**

12. **GIVEN** `mvp`，**WHEN** 计算 12 道菜的单位价值，**THEN** 依次为：白米饭 4.00、拍黄瓜 5.00、可乐 6.00、冰淇淋 7.00、炒青菜 3.00、番茄炒蛋 4.00、麻婆豆腐 6.00、炸鸡腿 6.50、清蒸鱼 5.00、红烧肉 5.333333…、糖醋里脊 4.50、豪华海鲜锅 3.333333…；`price`、`satiety` 均为 int，`satiety ∈ 1–6`。
13. **GIVEN** `mvp`，**WHEN** 计算 `uv_span`，**THEN** 结果为 `4.00`（7.00 − 3.00）。
14. **GIVEN** `mvp`，**WHEN** 运行审查，**THEN** `menu_healthy = true`、`same_group_dominated_pairs = 0`、`uv_span = 4.00 ≥ 1.5`、`tier_coverage = true`。
15. **GIVEN** 空菜单夹具，**WHEN** 审查，**THEN** `uv_span = 0`、`menu_healthy = false`、不抛异常；**GIVEN** 单菜菜单，**THEN** `menu_healthy = false`。
16. **GIVEN** `span_1_0.tres`（菜池三道菜 6/1、16/3、30/6，UV 为 6.0、5.333、5.0，档位覆盖完整），**WHEN** 审查，**THEN** `uv_span = 1.0`、`menu_healthy = false`，且失败原因仅为跨度不足。
17. **GIVEN** `span_boundary_fractional.tres`（三道菜：16/3、23/6，外加 4/1 补齐 satiety=1 档；最高与最低仍为 16/3 与 23/6，精确差 1.5），**WHEN** 审查，**THEN** `menu_healthy = true`——确认 `1e-9` 容差生效，float64 的 `1.4999999999999996` 不会误判。
18. **GIVEN** `all_satiety_1.tres`（四道菜 satiety 均为 1、分组各不相同），**WHEN** 审查，**THEN** `menu_healthy = false`（`tier_coverage = false`）。
19. **GIVEN** `all_uv_equal.tres`（所有菜 UV = 3.0），**WHEN** 审查，**THEN** `uv_span = 0`、`menu_healthy = false`。
20. **GIVEN** 同组夹具：B 为 13/2、A 为 12/2，**WHEN** 审查，**THEN** `dominates(B, A) = true`；**GIVEN** 两条完全相同的 13/2 记录，**THEN** 两个方向的 `dominates` 都为 `false`（严格性）。
21. **GIVEN** 两条菜的 `cloying_group` 都为空，B 13/2、A 12/2，**WHEN** 审查，**THEN** `dominates(B, A) = true`、`same_group_dominated_pairs = 1`，菜单被判定不可发布。
22. **GIVEN** 炸鸡腿 13/2 与麻婆豆腐 12/2、不同分组、麻婆豆腐用途为「口味路线」，**WHEN** 审查，**THEN** `same_group_dominated_pairs = 0`；`cross_dominated_pairs ≥ 1`（原始跨组支配存在，由人工核对用途）。
23. **GIVEN** 两道菜 `price`、`satiety`、`cloying_group`、`taste_tags` 完全相同，**WHEN** 审查，**THEN** 恰好产生 1 条冗余 warning；**GIVEN** 仅 `taste_tags` 顺序不同（集合相同），**THEN** 0 条 warning。
24. **GIVEN** `mvp`，**WHEN** 只修改 `display_name` 与 `icon`，**THEN** 所有审查输出（单位价值、跨度、支配对数、`opt_value`）逐字节不变。`station_id` 不在此列——它是结构字段，单独改动会破坏档口双向引用。
25. **GIVEN** `mvp`、`C = 20`、`T = 10`，**WHEN** 计算 `opt_value`，**THEN** 结果为 `104`，使用的 10 道菜合计 20 格；按价格降序贪心为 `94`。
26. **GIVEN** `M = ∅`、`C = 0`、`T = 0` 三种情况，**WHEN** 计算 `opt_value`，**THEN** 都返回 `0`；**GIVEN** 单菜 10/1、`C = 2`、`T = 2`，**THEN** 返回 `10` 而不是 `20`（0/1，不重复选）。
27. **GIVEN** `opt_value = 104`、`entry_fee = 85`，**WHEN** 计算 `calibration_ratio`，**THEN** 结果为 `1.223529411…`，落在 `[1.15, 1.35]`，报告取 `1.224`。
28. **WHEN** 静态检查运行时 API 表面，**THEN** 没有任何导出函数对菜集合求和、返回净利或调用 `opt_value`；`opt_value` 只能从 CI/设计审查入口到达。
29. **GIVEN** 12 道菜、`C = 20`、`T = 10`，**WHEN** 跑完整审查（跨度 + 支配 + 冗余 + `opt_value`），**THEN** 全量支配扫描（132 个有向对）≤ 1 ms（硬上限 5 ms），`opt_value` ≤ 5 ms（硬上限 20 ms），整轮 ≤ 50 ms。这些是设计期/CI 预算，不是帧时间预算。

**无法直接断言的规则**：

- **Core Rule 1（合法 ≠ 平衡）**：这是职责划分，不是可执行判定；由第 14、20–23 条间接覆盖。
- **Core Rule 6（首局替换建议）**：建议文案与触发属于「首局引导」，本系统只保证结算期能取到单位价值数据。
- **`unjustified_dominated`**：用途列不是 `DishData` 字段，CI 只能算原始跨组支配对；用途核对是人工评审步骤（Open Question 7）。
- **±10% 菜单重掷**：属于「菜单波次」契约，本系统只提供 `opt_value`，不在本 GDD 内断言。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | 初始胃容量 20 格是暂定值（胃容量 GDD 未写）。若最终值不同，`opt_value`、`calibration_ratio` 与入场费必须重算。**已回答（2026-11-03）**：胃容量 GDD 确认 `C0 = 20`；数据归属见其 Open Question 1。 | game-designer（胃容量） | 胃容量 GDD 完成时 |
| 2 | MVP 档口入场费建议 85 元（`calibration_ratio = 1.224`）。数据配置 GDD 公式示例里的 `entry_fee = 30` 只是示例，不是锁定值；最终值由「结算与入场费」确认。 | economy-designer（结算） | 结算 GDD 完成时 |
| 3 | 9 个 `cloying_group` 的归属尚未与「腻度与口味」核对；惩罚曲线可能要求合并或拆分分组，任何调整都要重跑支配审查。 | systems-designer（腻度） | 腻度 GDD 完成时 |
| 4 | 12 道菜的首版数值未经原型验证。"被算穿"的风险由 `/prototype 点菜取舍与腻度` 判定；若原型显示比值排序仍然近似最优，需要扩大价格/饱腹分布或依赖腻度制造第二轴。 | game-designer + prototyper | Vertical Slice 前 |
| 5 | 菜单波次是否允许重复抽取同一道菜。若允许，`opt_value` 的 0/1 背包假设失效，需改为有界背包。 | game-designer（菜单波次） | 菜单波次 GDD 完成时 |
| 6 | 局内是否在任何情况下展示单位价值（例如教学后的可选开关）。当前定为完全不展示；若试玩证明心算负担过高，再讨论"延迟到第 N 局解锁"。 | ux-designer + playtest | 首轮试玩后 |
| 7 | 「设计作用」（`腻度错峰` / `口味路线` / `容量精度`）目前是 GDD 内的评审清单，不是 `DishData` 字段。是否要进入数据层由数据配置决定；在进入之前，`unjustified_dominated` 由人工评审执行，CI 只能算原始支配对。 | lead-programmer + 数据配置 | 技术设计阶段 |
