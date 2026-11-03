# 结算与入场费

> **Status**: In Design
> **Author**: game-designer + economy-designer, systems-designer, qa-lead
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 每口都是算术题 · 没有白吃的菜 · 数字看得见地涨

## Summary

结算与入场费在每局结束时把吃下的菜价求和、扣掉档口入场费，产出一份可以为负的净利与一张逐行可核对的收据。它把支柱 1 的算术题在结算页验证一次：玩家局内的每一次心算都要在这里得到同一个答案。它不碰钱包、纪录与最优解——只负责把这一局算清。

> **Quick reference** — Layer: `Feature` · Priority: `MVP` · Key deps: `数据配置`, `菜品数值`, `腻度与口味`, `胃容量`, `时间与回合`, `餐局流程`

## Overview

结算与入场费把一局的经营动作收束成一张收据。餐局结束的那一刻，它读取本局吃下的每一道菜，按「菜品数值」提供的 `price` 求和，扣掉档口在「数据配置」里定义的入场费，得到一个可以为负的净利，并把完整明细交给「界面流程」「局外升级」与「纪录与里程碑」。它不解释、不建议、不比较最优解——只回答玩家走进自助餐厅前就在问的那句话：这一局，回本了没有。

系统自己不做饭、不判定能不能吃、不发明价格。它的价值在于**把已经发生的事实算清并摊开**：每一口值多少、重复腻度多占了几格、离回本差多少。它是支柱 1「每口都是算术题」的验证环节——玩家在局内做的每一次心算，都要在这里得到同一个答案。

## Detailed Design

### Core Rules

1. **职责边界**：本系统拥有入场费的套用、本局总值的求和、净利与回本判定、以及结算明细 `RunResult` 的组装。它不拥有菜品价格与饱腹（数据配置 / 菜品数值）、腻度惩罚（腻度与口味）、可吃性与回合推进（胃容量 / 时间与回合）、持久化钱包与升级（局外升级 / 存档）、最佳纪录比较（纪录与里程碑）或界面绘制（界面流程）。
2. **入场费是数据**：入场费取自 `StationData.entry_fee`（int ≥ 0）。MVP 档口 `station_canteen.entry_fee = 85`，本 GDD 确认该值（回应菜品数值 Open Question 2）。运行期只读；结算不得写回记录。
3. **价格即得分，没有额外加成**：本局总值 `total_value` 是吃下的每道菜 `price` 之和，不计算时间奖励、容量奖励、连击倍率、口味加成或随机修正。任何玩家在局内看不到的加成都违反支柱 1——算术题必须能在局内算完。
4. **一次交易，一张收据**：入场费在收据上是一条扣款项。叙事上它在进店前支付，机制上只在结算时发生一次；开局不做钱包扣款，也不存在"付不起就不能进"的门槛。MVP 不设钱包门槛（见 Open Questions 1）。
5. **净利可以为负，且不被钳制**：`net_profit = total_value − entry_fee`，有符号 int64。亏损就是负值，如实显示；不改成 0，不转成下局抵扣，不静默换算成"经验"。
6. **回本是一个明确状态**：`broke_even = net_profit ≥ 0`。收据必须用文字给出「回本 / 未回本」，不要求玩家自己比较两个数。净利恰好为 0 计为回本。
7. **每局恰好结算一次**：结算由「餐局流程」在本局结束、发起 `Run → Menu` 之前调用，结果随 `RunResult` 在同一帧交给「界面流程」（场景与流程管理规则 6）。同一 `run_id` 的第二次调用被拒绝并返回错误，不产生第二份结果、不重复触发下游。
8. **放弃、进入失败与退出不是结算**：玩家放弃一局、`RunConfig` 解析失败或直接退出时，不产生 `RunResult`、不算净利、不写纪录（场景与流程管理规则 6 / 8 / 10）。
9. **收据按吃下的顺序**：明细行严格按本局吃下的先后排列，不按价格、饱腹或单位价值重排。顺序本身是复盘信息：玩家要能指出"从第三盘开始就走错了"。
10. **每一行都是完整的事实**：每行至少包含菜名、`price`、基础 `satiety`、腻度追加 `extra_satiety`、实际占用 `effective_satiety`、静态单位价值、实际单位价值（`price / effective_satiety`）。缺任何一项，玩家就无法把某一盘和它真正占的容量对上。
11. **合计行覆盖容量与回合**：除总值、入场费、净利外，收据还必须给出已用容量 / 总容量、浪费容量、已用回合 / 总回合、腻度额外占用合计。浪费容量回答"为什么没吃满"，不是装饰。
12. **不展示最优解**：结算不得显示 `opt_value`、理论最高净利或"你本可以多赚 X"。结算只陈述发生了什么；展示上限会把静态菜单变成背下来的常数（菜品数值规则 11），并把失败从"我的取舍"变成"抄答案"。
13. **纯函数、可回放**：给定同一份 `RunLog`、同一档口与同一份数据，输出逐字段相同；不使用随机数、不读系统时钟、不做文件或网络 I/O。记录需要的时间戳由「纪录与里程碑」或「存档」在接收结果后自行添加。
14. **int64 求和并检查溢出**：`price` 与 `satiety` 的上限是 INT32_MAX，所有累加用 int64；任何一步溢出都返回错误并终止结算，绝不回绕。正常玩法受回合数约束不可能溢出，检查是为了让非法数据变成可见错误。
15. **零入场费合法**：`entry_fee = 0` 时 `net_profit = total_value`、`broke_even` 恒为真，`return_ratio` 无定义；收据显示「免单」，不出现除零、`∞` 或 `NaN`。
16. **重复出现的菜按次计价**：同一道菜被吃两次时，收据出现两行，各自计一次 `price` 与各自的实际饱腹，不合并、不取最后一次。MVP 假设不重复抽取；本条为菜单波次的未来改动预留。
17. **数据未就绪或记录缺失是错误，不是零分**：若数据配置不在 `Ready`、或吃菜记录中的 id 无法解析，返回结构化错误（调用者、dish id、原因），不生成部分收据、不用占位菜、不按 0 元计入。
18. **结算不碰钱包，也不比较纪录**：它只产出 `RunResult`。净利如何进入钱包（是否允许为负、是否有下限）、是否刷新最佳纪录、是否解锁档口，分别属于「局外升级」「纪录与里程碑」「解锁与档口」。这条边界让结算可独立测试，也避免与未设计的局外系统形成环。
19. **局内净利与本系统同一套求和**：「局内 HUD」显示的实时净利必须调用本系统公开的 `running_net` 与整数累加器，不得自行再写一份求和。两处一旦漂移，玩家在局内和结算页会看到两个不同的世界。

### States and Transitions

本系统不持有餐局进行中的状态——吃菜记录由「餐局流程」持有并在结算时以 `RunLog` 一次性传入。本系统只保留一个 `last_settled_run_id`，用于阻止同一局重复结算。

| 状态 | 触发 | 下一个状态 | 说明 |
|---|---|---|---|
| `Idle` | 应用启动 / 上一局已结算 | `Idle` / `Settled` | 没有待结算的局 |
| `Idle` / `Settled` | `settle(RunLog)`，`run_id` 未结算过且数据 `Ready` | `Settled` | 计算并返回 `RunResult`；写入 `last_settled_run_id` |
| `Idle` / `Settled` | `settle(RunLog)`，`run_id` 等于 `last_settled_run_id` | 不变 | 拒绝，返回 `ALREADY_SETTLED`，不返回第二份结果 |
| `Idle` / `Settled` | 结算期间数据未 `Ready` / dish id 不存在 / 溢出 | 不变 | 拒绝，返回结构化错误，不产生部分结果 |
| `Settled` | 新一局开始（新 `run_id`） | `Settled` | 无需清理；上一局结果已交给界面流程且不缓存 |
| 任意 | 放弃 / 进入失败 / 退出 | 不变 | 不调用结算，无结果、无纪录 |

`RunLog` 的字段是结算的完整输入，由「餐局流程」组装：

| 字段 | 类型 | 来源 | 说明 |
|---|---|---|---|
| `run_id` | int64 | 餐局流程 | 每局唯一；结算的去重键 |
| `station_id` | StringName | 场景与流程管理 `RunConfig` | 用于读取 `entry_fee` |
| `end_reason` | enum | 时间与回合 / 胃容量 | MVP 只有 `TIME_EXHAUSTED` / `NO_EATABLE_DISH`；主动结账见 Open Question 2 |
| `eaten` | EatenEntry[] | 腻度与口味 + 餐局流程 | 按吃下顺序；每条含 `dish_id`、`extra_satiety`、`effective_satiety` |
| `capacity_total` | int | 胃容量 | 本局总容量 |
| `capacity_used` | int | 胃容量 | 本局实际占用；必须等于各行 `effective_satiety` 之和（不变式 1） |
| `turns_total` / `turns_used` | int | 时间与回合 | 总回合数与实际消耗 |
| `upgrade_snapshot` | Dictionary | 场景与流程管理 `RunConfig` | 本局的局外升级等级快照；结算不解释，只随结果转交纪录 |

**行值在吃下时记录，结算不重算**：`extra_satiety` 与 `effective_satiety` 由腻度系统在吃下那一刻给出并随 `RunLog` 传入；结算不得根据菜名重放腻度序列。重放会复制一套腻度规则，一旦两边漂移，玩家付掉的容量和收据上的容量就会对不上。

`RunResult` 的字段是结算的完整输出，交给界面流程、局外升级与纪录：

| 字段 | 类型 | 说明 |
|---|---|---|
| `run_id` / `station_id` / `end_reason` | 同 RunLog | 原样转交，用于去重与纪录归属 |
| `entry_fee` | int | 本局套用的入场费；纪录必须冻结它，否则重调后历史不可比 |
| `total_value` | int64 | 本局吃进总值 |
| `net_profit` | int64 | 有符号净利 |
| `return_ratio` / `return_percent_x10` | float64 \| null / int \| null | 回本率；`entry_fee = 0` 时两者都为 `null` |
| `rows` | EatenRow[] | 按吃下顺序；含 `dish_id`、`price`、`satiety`、`extra_satiety`、`effective_satiety`、`unit_value`、`effective_unit_value` |
| `capacity_total` / `capacity_used` / `wasted_capacity` | int | 容量明细 |
| `cloy_extra_total` | int | 腻度额外占用合计 |
| `turns_total` / `turns_used` | int | 回合明细 |
| `upgrade_snapshot` | Dictionary | 从 RunLog 原样转交 |

`broke_even` **不是存储字段**：它是 `net_profit ≥ 0` 的派生谓词，读取时计算。`rows[].unit_value` 直接取自「菜品数值」的 `unit_value(d)`，不重算；`rows[].effective_unit_value` 由本系统按 Formulas 计算。

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 数据配置 | 上游 | `get_station(id)` → `entry_fee`；`get_dish(id)` → `price`、`satiety`；调用前提 `Ready` | 硬 |
| 菜品数值 | 上游 | `price(d)`、`unit_value(d)`；本系统不重算单位价值 | 硬 |
| 腻度与口味 | 上游 | 每个 `EatenEntry` 的 `extra_satiety`、`effective_satiety`（吃下时记录） | 硬 |
| 胃容量 | 上游 | `capacity_total`、本局实际已用容量 | 硬 |
| 时间与回合 | 上游 | `turns_total`、`turns_used`、`end_reason` | 硬 |
| 餐局流程 | 上游 | 本局结束时调用 `settle(RunLog)`；保证每局一次、顺序正确 | 硬 |
| 界面流程 | 下游 | `RunResult` → 收据屏；在 `Run → Menu` 同帧接收（场景与流程管理规则 6） | 硬 |
| 局外升级 | 下游 | `net_profit`（有符号）、`broke_even`；钱包策略由它拥有 | 硬 |
| 纪录与里程碑 | 下游 | `net_profit`、`total_value`、`return_ratio`、`station_id`、`end_reason`；时间戳由它添加 | 硬 |
| 存档 | 下游（间接） | 只持久化局外升级与纪录保留下来的字段；结算本身无 I/O | 软 |
| 局内 HUD | 下游 | `running_net(k)`：局内实时总值减入场费 | 硬 |
| 首局引导 | 下游 | 每行的静态 / 实际单位价值 | 软 |
| 音效反馈 | 下游 | `settlement_started`、`broke_even`、`loss` 三个结算事件 | 软 |

**接口所有权**：本系统拥有 `RunResult` 的字段与语义、求和规则、净利与回本判定、收据顺序。下游不得自行实现另一套求和或回本判定；HUD 的实时净利必须走 `running_net`。

## Formulas

### 本局总值 `total_value`

The `total_value` formula is defined as:

`total_value(R) = Σ_{i=1}^{n} price(d_i)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `R` | RunLog | — | 餐局流程 | 本局完整输入：`run_id`、`station_id`、`eaten[]`、容量与回合计数 |
| `n` | int | 0–`turns_used` | `R.eaten.length` | 成功吃下的菜数；被拒绝的菜不入日志 |
| `d_i` | DishData | — | `get_dish(R.eaten[i].dish_id)` | 第 i 个吃下的菜 |
| `price(d_i)` | int | 0–INT32_MAX | `DishData.price` | 该菜的得分 |
| `total_value` | int64 | 0–Σprice | calculated | 本局吃进的总价值（元） |

**Output Range:** `[0, Σ price]`；空局为 `0`。累加使用 checked int64，任一步溢出即返回 `OVERFLOW` 并终止，不回绕。

**Example:** F1（10 道、合计 104）→ `total_value = 104`；F2（6 道、合计 54）→ `54`；空局 → `0`。

### 净利 `net_profit`

The `net_profit` formula is defined as:

`net_profit(R) = total_value(R) − entry_fee(station_id)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `entry_fee(station_id)` | int | 0–INT32_MAX | `StationData.entry_fee`（数据配置） | 本档口入场费；MVP `station_canteen = 85` |
| `net_profit` | int64 | −entry_fee–(Σprice − entry_fee) | calculated | 本局净利，有符号；负值表示亏损 |

**Output Range:** 下界 `−entry_fee`（一道没吃），上界由菜单与容量决定。**不做钳制**：负值原样返回，是否进入钱包或按 0 计入由「局外升级」决定。

**Example:** `104 − 85 = +19`；`54 − 85 = −31`；`0 − 85 = −85`。

### 回本判定 `broke_even`

The `broke_even` formula is defined as:

`broke_even(R) = (net_profit(R) ≥ 0)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `broke_even` | bool | {true, false} | calculated（派生，不单独存储） | 本局是否回本；净利恰好为 0 时为 true |

**Output Range:** `{true, false}`。`broke_even` 是 `net_profit` 的派生谓词，不写入 `RunResult` 的存储字段，读取时由 `net_profit` 计算——两处若各存一份，迟早会不一致。

**Example:** `+19 → true`；`−31 → false`；`0 → true`。

### 回本率 `return_ratio`

The `return_ratio` formula is defined as:

`return_ratio(R) = total_value(R) / entry_fee(station_id)`，当 `entry_fee = 0` 时无定义

展示值用整数基点计算，避免浮点格式化漂移：

`return_percent_x10(R) = floor(1000 × total_value(R) / entry_fee)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `return_ratio` | float64 | (0, +∞) | calculated | 实际总值相对入场费的倍数；1.0 即回本 |
| `return_percent_x10` | int | 0–+∞ | calculated | 以 0.1% 为单位的展示值；1000 表示 100.0% |

**Output Range:** `return_ratio > 0`；`entry_fee = 0` 时返回「无定义」，收据显示「免单」，**不执行除法**。1.0 与回本线重合，玩家可以直接读「超过 100% 就是赚了」。

**Example:** `104 / 85 = 1.2235…` → 展示 `122.3%`；`54 / 85 = 0.6353…` → `63.5%`；`85 / 85 = 1.0` → `100.0%`。

> **与 `calibration_ratio` 的区别**：`calibration_ratio = opt_value / entry_fee`（菜品数值拥有，设计期校准上限）；本式是**实际**总值 / 入场费，运行期结果。两者都以 85 为分母、都以 1.0 为回本线，但一个衡量"最好能到多少"，一个衡量"这一局到了多少"。结算不得读取或展示 `opt_value`。

### 容量使用 `capacity_used` / `wasted_capacity`

The `capacity_used` formula is defined as:

`capacity_used(R) = Σ_{i=1}^{n} effective_satiety(d_i)`

The `wasted_capacity` formula is defined as:

`wasted_capacity(R) = capacity_total(R) − capacity_used(R)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `effective_satiety(d_i)` | int | 1–(satiety_max + cloy_cap) | `R.eaten[i].effective_satiety`（吃下时由腻度系统记录） | 该菜实际占用的容量格 |
| `capacity_total(R)` | int | ≥ 1 | 胃容量 | 本局总容量（MVP 初始 20） |
| `capacity_used` | int | 0–capacity_total | calculated | 本局实际占用格数 |
| `wasted_capacity` | int | 0–capacity_total | calculated | 没吃满的格数；因回合耗尽或放不下任何菜而留下 |

**Output Range:** `capacity_used ∈ [0, capacity_total]`、`wasted_capacity ∈ [0, capacity_total]`，两者之和恒等于 `capacity_total`。若 `capacity_used > capacity_total`，返回 `CAPACITY_OVERFLOW` 错误，不钳制；若 `Σ effective_satiety` 与 `RunLog.capacity_used` 不一致，返回 `CAPACITY_MISMATCH`。

**Example:** F1 `20 / 20`、浪费 `0`；F2 `12 / 20`、浪费 `8`。

### 腻度额外占用 `cloy_extra_total`

The `cloy_extra_total` formula is defined as:

`cloy_extra_total(R) = Σ_{i=1}^{n} extra_satiety(d_i)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `extra_satiety(d_i)` | int | 0–`cloy_cap` | `R.eaten[i].extra_satiety` | 该次因同组重复被追加的容量 |

**Output Range:** `[0, n × cloy_cap]`。它把"浪费容量"中的可避免部分单独列出来：玩家能区分"没吃满是因为点错顺序"与"没吃满是因为回合到了"。

**Example:** F4 中 B 第二次触发 1 格额外占用 → `cloy_extra_total = 1`。

### 有效单位价值 `effective_unit_value`

The `effective_unit_value` formula is defined as:

`effective_unit_value(d) = price(d) / effective_satiety(d)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `effective_satiety(d)` | int | 1–(satiety_max + cloy_cap) | 腻度与口味 | 吃下时实际占用 |
| `effective_unit_value` | float64 | (0, price_max] | calculated | 这一口真实的元/格；仅结算页展示 |

**Output Range:** `(0, price_max]`。展示固定两位小数（沿用菜品数值的 `display_uv` 规则）。**它不是 `unit_value`**：`unit_value(d) = price(d) / satiety(d)` 是静态刻度、由菜品数值拥有；本式只在结算页作为对照出现，且不参与排序或判定。

**Example:** 红烧肉后再吃糖醋里脊：未解腻时 `18 / 5 = 3.60`；先吃拍黄瓜解掉「肉」后 `18 / 4 = 4.50`。收据两列并列，玩家能看见顺序的代价。

### 局内实时净利 `running_net` / 回本差额 `shortfall`

The `running_net` formula is defined as:

`running_net(k) = Σ_{i=1}^{k} price(d_i) − entry_fee`

The `shortfall` formula is defined as:

`shortfall(k) = max(0, entry_fee − Σ_{i=1}^{k} price(d_i))`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `k` | int | 0–n | 局内 HUD | 当前已吃下的菜数；`k = 0` 时总值为 0 |
| `running_net(k)` | int64 | −entry_fee–+∞ | calculated | 局内实时净利；第一道菜之前就是 `−entry_fee` |
| `shortfall(k)` | int64 | 0–entry_fee | calculated | 距回本还差多少元；回本后恒为 0 |

**Output Range:** `running_net` 从 `−entry_fee` 起步，随每道菜上升；`shortfall` 单调不增，回本后为 0。**HUD 必须从第一道菜之前就显示 `−85`**，否则玩家在结算页第一次看到入场费，会觉得被偷袭。

**Example:** 已吃红烧肉（16）、拍黄瓜（5）、糖醋里脊（18）→ 总值 39、`running_net = −46`、`shortfall = 46`。

### 运行时不变式（不是公式，是断言）

结算在产出结果前必须验证以下四条；任一失败返回结构化错误，不生成收据：

| # | 不变式 | 失败原因 |
|---|---|---|
| 1 | `capacity_used = RunLog.capacity_used` | `CAPACITY_MISMATCH` — 行记录与容量系统对不上 |
| 2 | `capacity_used ≤ capacity_total` | `CAPACITY_OVERFLOW` — 容量系统放行了不该放行的菜 |
| 3 | `effective_satiety(d_i) = satiety(d_i) + extra_satiety(d_i)` | `ROW_INCONSISTENT` — 腻度行快照损坏 |
| 4 | `n ≤ turns_used ≤ turns_total` | `TURN_INCONSISTENT` — 回合记录损坏 |

### 设计期校准带（不进入运行时）

入场费 85 与 `opt_value = 104` 已在「菜品数值」确认为校准基准（`calibration_ratio = 1.224`，落在 `[1.15, 1.35]`）。本 GDD 追加一条**可调目标的验收带**，用于原型与垂直切片的平衡判断，不作为运行时断言：

| 指标 | 目标带 | 超出时的动作 |
|---|---|---|
| 前 3 局总值中位数 | 88–96（净利 +3 – +11） | 中位数 < 88 → 入场费每轮降 3–5 元；中位数 > 96 → 每轮升 2–3 元。每次只改入场费一项并重跑原型，直到落回 88–96 |
| 前 3 局总值 25 分位 | ≥ 75（净利 ≥ −10） | 低于 75 说明惩罚过重：先按上一条降入场费；若入场费已降到 75 仍无改善，再考虑降 `cloy_cap` |
| 熟练玩家总值 | ≥ 99（净利 ≥ +14） | 低于 99 说明最优路径不可读，检查解腻顺序是否在 UI 中可见；不改入场费来掩盖可读性问题 |

**注意：104 不是无条件可达。** 它依赖 `红烧肉 → 拍黄瓜（解腻「肉」）→ 糖醋里脊` 这条零惩罚顺序；没有解腻时两道肉共占 21 格，104 确实不可达。任何重算必须带解腻，否则会得出错误的"上限 93–100"。

**升级会改变校准。** 胃容量或解腻抗性升级生效后，`opt_value` 与入场费的对应关系不再成立；每次调整容量上限或抗性后必须重跑 `calibration_ratio`，不能沿用 104 / 85。

## Edge Cases

- **If `entry_fee = 0`**：`net_profit = total_value`、`broke_even = true`；`return_ratio` 返回「无定义」，收据显示「免单」，不显示 `∞`、`NaN` 或 0。零费用是合法的档口配置，不是错误。
- **If `entry_fee > total_value`**：净利为负，照实显示。不把负值钳到 0，不显示"差一点回本"之类的补偿文案，也不把差额留到下一局。
- **If `net_profit = 0`**：`broke_even = true`。判定用 `≥ 0` 而不是 `> 0`；恰好回本是一局成立的合法结果。
- **If 本局一道菜都没吃**：收据零行，`total_value = 0`、`net_profit = −entry_fee`、`broke_even = false`；不崩溃、不省略收据。空局仍然是一次完整的结算。
- **If 同一道菜出现两次**：收据两行，各自计 `price` 与各自的实际饱腹；不合并。`cloy_extra_total` 累加两次各自记录的 `extra_satiety`。
- **If `effective_satiety ≠ satiety + extra_satiety`**：视为上游状态损坏，返回错误并终止结算，不"就地修正"；修正会让收据与玩家实际付出的容量不一致。
- **If `extra_satiety < 0` 或 `capacity_used < 0`**：返回错误。负占用意味着腻度或容量系统出了 bug，静默取绝对值会掩盖它。
- **If `capacity_used > capacity_total`**：返回错误，不钳到 `capacity_total`。正常流程不可能发生；钳制会让越界变成一张看起来正常的收据。
- **If `turns_used > turns_total`**：返回错误，理由同上。
- **If 数据配置不在 `Ready`**：返回 `DATA_NOT_READY`，不生成收据、不返回 0 分。结算期间数据应始终就绪；失败说明调用时序错误。
- **If `RunLog` 中的 dish id 不存在**：返回 `UNKNOWN_DISH`，列出全部缺失 id；不跳过该行、不用占位菜、不按 0 元计入。
- **If `price < 0`**：数据配置本应在加载期拒绝；结算仍须断言并返回错误，不把负价当折扣。
- **If 求和溢出 int64**：返回 `OVERFLOW`，不输出任何数值；绝不回绕成负数或显示错误总额。
- **If 同一 `run_id` 第二次结算**：返回 `ALREADY_SETTLED`，不产生第二份 `RunResult`、不重复触发下游。不同 `run_id` 视为新的一局，正常结算。
- **If 玩家从暂停菜单放弃一局**：不产生结算；没有收据、没有净利、不写纪录（场景与流程管理规则 8）。放弃不是"提前结账"。
- **If `RunConfig` 解析失败**：不产生结算，返回 `Menu` 并携带 `init_failed`（场景与流程管理规则 6）。
- **If 玩家在局内退出游戏**：本局作废，不产生结算、不写纪录（场景与流程管理规则 10）。
- **If 暂停中收到结算请求**：拒绝并返回错误。暂停期间回合冻结，结算只能发生在 `Run` 状态结束的那一刻。
- **If 回合耗尽但仍有剩余容量**：正常结算，`end_reason = TIME_EXHAUSTED`，收据显示浪费容量。这是玩家可以复盘的失败（"最后两格没用上"），不是错误。
- **If 没有任何合法动作（所有可见菜都放不下且无法等待）**：正常结算，`end_reason = NO_EATABLE_DISH`。该结束条件由「时间与回合」/「餐局流程」判定，结算只记录原因，不自行判断。
- **If 本局只有一道菜**：正常结算；`turns_used` 可能为 1、`wasted_capacity` 可能很大，全部如实显示。
- **If 首局**：结算流程与数值完全不变；「首局引导」只额外使用收据里的单位价值列做讲解，不改变任何合计。
- **If 收据的显示精度与实际精度不一致**：合计与净利为整数元；单位价值固定两位小数（菜品数值规则）；回本率固定一位小数的百分比。展示值不参与任何判定，判定一律用整数或未格式化的 float64。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 数据配置 | 硬 | `get_station(id)` → `entry_fee`；`get_dish(id)` → `price`、`satiety`；前提 `Ready`；int64 求和规则 | GDD 已写 |
| 菜品数值 | 硬 | `price(d)`、`unit_value(d)`（静态单位价值 = `price / satiety`）；本系统不重算 | GDD 已写（Approved 2026-11-03） |
| 腻度与口味 | 硬 | 每个 `EatenEntry` 的 `extra_satiety` 与 `effective_satiety`（吃下时记录，结算不重放） | GDD 已写（Designed 2026-11-03） |
| 胃容量 | 硬 | `capacity_total`；可吃性判定在结算之前完成 | GDD 已写（`design/gdd/stomach-capacity.md`） |
| 时间与回合 | 硬 | `turns_total`、`turns_used`、`end_reason` | GDD 已写（`design/gdd/time-and-turns.md`） |
| 餐局流程 | 硬 | 本局结束时调用一次 `settle(RunLog)`；放弃 / 进入失败 / 退出不调用 | GDD 已写（`design/gdd/run-flow.md`） |

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 局外升级 | 硬 | `net_profit`（有符号）、`broke_even`；钱包策略与是否钳制由它拥有 | GDD 已写（`design/gdd/meta-upgrades.md`） |
| 纪录与里程碑 | 硬 | `net_profit`、`total_value`、`return_ratio`、`station_id`、`end_reason`；时间戳由它添加 | GDD 已写（`design/gdd/records-and-milestones.md`） |
| 界面流程 | 硬 | `RunResult` → 收据屏；在 `Run → Menu` 同帧接收 | GDD 未写 |
| 局内 HUD | 硬 | `running_net(k)` 与共享的 int64 累加器；不得自行求和 | GDD 已写（`design/gdd/in-game-hud.md`） |
| 首局引导 | 软 | 每行的静态 / 实际单位价值，用于讲解与替换建议 | GDD 未写 |
| 音效反馈 | 软 | `settlement_started` / `broke_even` / `loss` 事件；结果 cue 在数字滚动结束时起播 | GDD 已写（`design/gdd/audio-feedback.md`） |
| 存档 | 软（间接） | 只持久化局外升级与纪录保留下来的字段；结算无 I/O | GDD 已写（`design/gdd/save-system.md`） |
| 菜单波次 | 软（未来） | 若允许同一道菜重复出现，结算按次计价 | GDD 未写 |

**双向一致要求**：上表每个系统在编写自己的 GDD 时，必须在 Dependencies 中列出「结算与入场费」。已知需要回指的两处：

- **局内 HUD**：`systems-index.md` 当前把 HUD 的上游列为餐局流程、时间与回合、胃容量、菜品数值、输入，**漏了结算**。HUD 要显示实时净利，就必须依赖本系统的 `running_net`；索引行需要补上这条边。
- **餐局流程 / 界面流程**：两者分别负责调用结算与接收 `RunResult`；它们的 GDD 必须写出"放弃不是结算"的边界（场景与流程管理规则 6 / 8）。

**对未设计系统的前置契约（provisional）**：

| 系统 | 结算要求 | 若最终实现不同 |
|---|---|---|
| 胃容量 | 提供本局 `capacity_total`；已吃菜的 `effective_satiety` 之和不得超过它 | 若容量系统允许透支，结算的 `wasted_capacity` 会变负；本系统将报错而不是钳制 |
| 时间与回合 | 提供 `turns_total` / `turns_used`；`end_reason` 只能取 `TIME_EXHAUSTED` 或 `NO_EATABLE_DISH` | 若新增结束原因，必须先改本 GDD 的枚举，再改下游 |
| 餐局流程 | 每局恰好调用一次 `settle(RunLog)`，`run_id` 唯一，`eaten` 按顺序且只含成功吃下的菜 | 若把被拒绝的菜也写进日志，`capacity_used` 会超过总容量，结算报错 |
| 菜单波次 | MVP 不重复抽取；若改为允许，同一道菜每次出现各占一行 | 结算的按次计价规则已就位，不需要改公式 |

**与菜品数值的校准契约**：本 GDD 确认 `station_canteen.entry_fee = 85`，静态最优 `opt_value = 104`，`calibration_ratio = 104 / 85 = 1.224`。**104 只在存在零惩罚顺序时可达**：`红烧肉 → 拍黄瓜（解腻「肉」）→ 糖醋里脊 → 其余七道`，合计 20 格、104 元（腻度与口味 Formulas 给出的见证顺序）。任何评审若按静态饱腹直接相加，会把这两道肉算成 21 格并错误地宣布 104 不可达；重算时必须带解腻。若未来菜单不存在零惩罚顺序，校准改用实际可达上限，入场费必须重算。

**与腻度与口味的容量契约**：结算只信任 `RunLog` 中记录的 `effective_satiety`，不按菜名重放腻度序列。重放会复制一套解腻规则，一旦两边漂移，收据上的占用就会和玩家实际付出的容量对不上（腻度与口味 Edge Cases 明确要求求和使用实际付出的容量）。

## Visual/Audio Requirements

结算页的视觉目标只有一个：**像一张可以逐行核对的收据**，而不是一个庆祝或安慰的界面。

- **收据纸与等宽数字**：米白纸底、粗分隔线、全部金额与单位价值使用等宽字体。视觉锚点沿用概念文档的「食堂价目板」与「界面像收据」。
- **回本红只给正收益**：正净利、回本率 > 100%、有效单位价值使用回本红。负净利用中性墨色 + 负号 + 「未回本」文字，**不用红色警示**——红色在这套视觉语言里代表"赚到了"，用在亏损上会反转含义。
- **容量用橄榄绿**：已用容量、浪费容量、腻度额外占用使用橄榄绿，与"胃容量与安全区"的既有语义一致。
- **无庆祝特效**：亏损不撒纸屑、不播放上扬音效、不出现"再接再厉"弹窗。回本可以有一次性短促的收银机音效，上限 0.4 秒。
- **一次性呈现**：收据行在同一帧全部出现，不用逐行打字机效果。唯一允许的动画是数字从 0 滚到终值（≤ 0.6 秒，可跳过，且不阻塞按钮）。
- **音效事件**（由「音效反馈」实现，本系统只定义语义）：`settlement_started`（纸张/打印）、`broke_even`（收银机/上扬短音）、`loss`（低闷短音）。三个事件互斥，不叠加。

> `art-director` not consulted — Lean mode（本系统是 Economy 计算层，视觉由收据隐喻与概念文档的视觉锚点直接决定）。进入生产前需人工复核。

## Game Feel

- **同一帧出结果**：本局结束到收据数据可用不超过一帧；`Run → Menu` 的状态切换不等待动画（场景与流程管理规则 12）。
- **一秒内读懂**：玩家进入结算页后一秒内必须能说出"赚了还是亏了、差多少"。净利与回本判定放在视觉层级最高的位置，明细在下方。
- **可归因**：每一行都能让玩家指着说"这一盘吃错了"。如果玩家看完收据仍说不出错在哪，是展示或数值空间的问题，不是玩家的问题。
- **不催促、不拖延**：没有自动跳转、没有倒计时；数字滚动可跳过，跳过不改变结果。
- **无随机**：同一局无论重看多少次，收据逐字段相同。结算页不是抽奖结果的揭晓，是账本的打印。
- **亏损不羞辱**：文案陈述事实（"未回本，差 31 元"），不评价玩家，不给"你本可以"。

## UI Requirements

**结算屏（收据）**

- **行顺序**：严格按吃下顺序；不提供排序控件，不默认按价格或单位价值排序。
- **每行字段**：菜名、价格、基础饱腹、腻度追加（0 时显示 `—`）、实际饱腹、静态单位价值、有效单位价值。金额与单位价值等宽、右对齐。
- **合计区**：总值、入场费（负项，带括号或负号）、净利（带符号）、回本判定（「回本」/「未回本」）、回本率（`122.3%`）、已用容量 / 总容量、浪费容量、已用回合 / 总回合、腻度额外占用合计。
- **空局**：显示明确的空状态（"这一局什么都没吃"），不是一张全 0 的表。
- **免单档口**：回本率显示「免单」，不显示 `∞`、`NaN` 或 100%。
- **按钮**：由「界面流程」拥有；结算只提供数据。MVP 至少需要「再来一局」与「返回菜单」，位置固定，不随净利正负变化。
- **首局**：允许「首局引导」高亮单位价值列并给出一条替换建议；建议的算法与文案属于首局引导，结算不计算、不展示最优解。

**局内 HUD（跨系统要求）**

- 从进入 `Run` 的第一道菜之前，HUD 就显示 `running_net = −entry_fee`（MVP 为 `−85`），并显示「还差 85 元回本」。不得等第一道菜下肚才出现入场费。
- 每吃一道菜，净利与回本差额在同一帧更新；数据来自本系统的 `running_net` / `shortfall`，HUD 不得自行求和。
- 入场费必须在进入一局之前的菜单上可见，玩家不能在不知道价格的情况下走进餐厅。

**无障碍**

- 数字与背景对比度 ≥ 4.5:1；字号放大到 150% 时收据行不换行错位、金额列不重叠。
- 盈亏不能只用颜色表达：必须有符号（`+` / `−`）与文字（「回本」/「未回本」）。
- 完整的无障碍要求见未来的 `design/accessibility-requirements.md`。

> **📌 UX Flag — 结算与入场费**：本系统有 UI 要求（收据屏、HUD 的实时净利与回本差额）。在 Phase 4（Pre-Production）写 epic 之前，运行 `/ux-design` 为收据屏与 HUD 这两块产出 UX 规格；相关 story 引用 `design/ux/settlement.md` 与 `design/ux/hud.md`，不要直接引用本 GDD。

## Cross-References

| 文档 | 关系 | 本系统引用/依据 |
|---|---|---|
| `design/gdd/data-config.md` | 上游 | `StationData.entry_fee`、`DishData.price`、`Ready` 门、int64 求和规则 |
| `design/gdd/dish-values.md` | 上游 | `price`、`unit_value`（静态）、`opt_value` 只供设计期校准，不进结算 |
| `design/gdd/cloying-and-taste.md` | 上游 | `extra_satiety` / `effective_satiety` 的行快照；求和使用实际付出容量 |
| `design/gdd/scene-flow.md` | 上游/下游 | `Run → Menu` 携带 `RunResult`；放弃与进入失败不结算的边界 |
| `design/gdd/systems-index.md` | 索引 | 优先级 MVP、层 Feature、设计顺序 #10 |
| `design/gdd/game-concept.md` | 来源 | 四支柱、核心循环「总价值 − 入场费 = 净利」、视觉锚点 |
| `design/gdd/input.md` | 无直接接口 | 结算屏的导航由输入上下文与界面流程处理 |

**待回指**：局外升级、纪录与里程碑、界面流程、局内 HUD、首局引导、音效反馈的 GDD 写好后，必须在各自的 Dependencies 与 Cross-References 中列出「结算与入场费」。其中局内 HUD 还需要在 `systems-index.md` 的依赖行补上本系统（见 Dependencies 的双向一致要求）。

## Acceptance Criteria

测试夹具放在 `tests/fixtures/settlement/`。以下夹具为合成 `RunLog`，不要求菜来自 MVP 菜池；每条用例使用独立夹具。

| 夹具 | 内容 | 结果 |
|---|---|---|
| F1 | 10 道菜，价格合计 104，容量 20/20，回合 10/10 | 静态最优局的结算 |
| F2 | 6 道菜，价格合计 54，容量 12/20，回合 6/10 | 亏损局 |
| F3 | 空局，0 道菜，容量 0/20，回合 0/10 | 空收据 |
| F4 | A(8/2) + B(10/2) + B(10/2，第二次 `extra_satiety = 1`) | 重复菜与腻度行 |
| F5 | 6 道菜合计恰好 85，容量 18/20，回合 6/10 | 恰好回本 |
| F6 | F2 的日志，档口 `entry_fee = 0` | 免单档口 |

**运行时（玩家可见行为）**

1. **GIVEN** F1，**WHEN** 结算，**THEN** `total_value = 104`、`net_profit = +19`、`broke_even = true`、回本率显示 `122.3%`。
2. **GIVEN** F2，**WHEN** 结算，**THEN** `total_value = 54`、`net_profit = −31`、`broke_even = false`、回本率显示 `63.5%`。
3. **GIVEN** F3，**WHEN** 结算，**THEN** 收据零行、`total_value = 0`、`net_profit = −85`、`broke_even = false`，界面显示明确的空状态而不是全 0 表；不崩溃。**截图留存** `production/qa/evidence/`。
4. **GIVEN** F5，**WHEN** 结算，**THEN** `net_profit = 0`、`broke_even = true`、回本率显示 `100.0%`（不因取整显示成 `99.9%`）。
5. **GIVEN** F6（`entry_fee = 0`），**WHEN** 结算，**THEN** `net_profit = 54`、`broke_even = true`，回本率显示「免单」，界面不出现 `∞`、`NaN` 或 `100%`；不发生除零。**截图留存**。
6. **GIVEN** F4，**WHEN** 生成收据，**THEN** 行顺序为 A、B、B（吃下顺序，不是价格或 id 顺序），且三行 `price` 之和等于合计区的 `total_value`。
7. **GIVEN** F4，**WHEN** 读取第二行 B，**THEN** 基础饱腹 2、腻度追加 1、实际饱腹 3、静态单位价值 `5.00`、有效单位价值 `3.33`；第一行 B 的追加为 0、实际饱腹 2、有效单位价值 `5.00`。
8. **GIVEN** F2，**WHEN** 读取合计区，**THEN** 已用容量 `12` / 总容量 `20`、浪费容量 `8`、已用回合 `6` / 总回合 `10`、腻度额外占用 `0`。
9. **GIVEN** F4，**WHEN** 读取合计区，**THEN** `cloy_extra_total = 1`，且它不计入浪费容量（浪费容量只由 `capacity_total − capacity_used` 决定）。
10. **GIVEN** F2 或 F3，**WHEN** 渲染结算屏，**THEN** 净利显示负号与「未回本」，不显示 0、空白或绝对值；不出现庆祝特效。**截图留存**。
11. **GIVEN** 任意已结算局，**WHEN** 检查 `RunResult` 字段与结算屏文本，**THEN** 不存在 `opt_value`、"最高"、"本可以"、"上限" 等字段或文案；payload 中没有最优值属性。**截图留存**。
12. **GIVEN** 进入 `Run` 且尚未吃任何菜，**WHEN** HUD 渲染，**THEN** 显示 `running_net = −85` 与「还差 85 元回本」；吃下红烧肉、拍黄瓜、糖醋里脊（合计 39）后，HUD 与结算输入同源，显示 `−46` 与「还差 46 元回本」。**截图留存**。
13. **GIVEN** 一局自然结束，**WHEN** `Run → Menu` 发生，**THEN** 收据屏恰好打开一次；返回菜单再进入，上一局收据不再出现、不残留。**截图留存**。
14. **GIVEN** 玩家从暂停菜单放弃一局，**WHEN** 回到菜单，**THEN** 没有收据、没有 `net_profit`、没有纪录写入；结算从未被调用。**截图留存**。
15. **GIVEN** `RunConfig` 解析失败，**WHEN** 返回菜单，**THEN** 携带 `init_failed`，不产生 `RunResult`，结算从未被调用。
16. **GIVEN** 同一 `run_id` 已结算，**WHEN** 再次调用 `settle`，**THEN** 返回 `ALREADY_SETTLED`，只有一份收据、下游只收到一次通知；不出现两个结算屏。
17. **GIVEN** 上一局已结算，**WHEN** 新一局以新 `run_id` 结算，**THEN** 正常产出新收据，不受上一局影响。

**逻辑 / 单元测试**

18. **GIVEN** 同一份 `RunLog` 的两个副本，**WHEN** 分别结算，**THEN** 两份 `RunResult` 逐字段深度相等；改变全局随机种子不改变结果；结果中不含时间戳、种子或读时钟生成的字段。
19. **GIVEN** 一个受控的 checked 累加器夹具（两个价格之和越过 int64 上限），**WHEN** 累加，**THEN** 返回 `OVERFLOW`、不输出任何数值、不回绕成负数；该用例直接测累加器，不构造数十亿行 `RunLog`。
20. **GIVEN** `RunLog.capacity_used = 19` 而各行 `effective_satiety` 之和为 18，**WHEN** 结算，**THEN** 返回 `CAPACITY_MISMATCH`；**GIVEN** 各行之和为 21、`capacity_total = 20`，**THEN** 返回 `CAPACITY_OVERFLOW`，两者都不钳制、不生成收据。
21. **GIVEN** 某行 `satiety = 2`、`extra_satiety = 0`、`effective_satiety = 4`，**WHEN** 结算，**THEN** 返回 `ROW_INCONSISTENT`，不"就地修正"。
22. **GIVEN** 行数 11 而 `turns_used = 10`，或 `turns_used = 11` 而 `turns_total = 10`，**WHEN** 结算，**THEN** 返回 `TURN_INCONSISTENT`。
23. **GIVEN** 数据配置未 `Ready`，**WHEN** 结算，**THEN** 返回 `DATA_NOT_READY`；**GIVEN** 日志含不存在的 dish id，**THEN** 返回 `UNKNOWN_DISH` 并列出全部缺失 id，不生成部分收据、不按 0 元计入。
24. **GIVEN** 修改数据配置中某道菜的 `price`（例如 +2），**WHEN** 重放 F1，**THEN** `total_value` 与 `net_profit` 精确变化 +2，证明价格只来自数据而非硬编码。
25. **GIVEN** 将档口 `entry_fee` 改为 37，**WHEN** 结算 F2，**THEN** `net_profit = 17`；**GIVEN** 将 `entry_fee` 改为 0，**THEN** 按第 5 条处理。
26. **GIVEN** 同一道菜在 `RunLog` 中出现两次，**WHEN** 结算，**THEN** 收据两行、`total_value` 计两次价格，不合并。
27. **GIVEN** 逐道吃下 F1 的十道菜，**WHEN** 每一步读取 `running_net` 与 `shortfall`，**THEN** `running_net` 从 `−85` 起步单调不降，`shortfall` 单调不增，最后分别为 `+19` 与 `0`。

**设计期 / CI（不进入运行时）**

28. **GIVEN** MVP 菜池、`C = 20`、`T = 10`、`entry_fee = 85`，**WHEN** 计算 `calibration_ratio`，**THEN** 结果为 `1.2235…`，落在 `[1.15, 1.35]`；**GIVEN** 见证顺序 `红烧肉 → 拍黄瓜（解腻「肉」）→ 糖醋里脊 → 其余七道`，**THEN** 合计恰好 20 格、104 元，证明 104 可达且不是静态相加得出的。
29. **GIVEN** 静态检查运行时 API，**THEN** 没有任何导出函数求和或计算净利时绕过 `RunLog` 与 checked int64；`opt_value` 只能从 CI / 设计审查入口到达，结算无引用。
30. **GIVEN** 20 行以内的正常 `RunLog`，**WHEN** 结算，**THEN** 单次结算 ≤ 1 ms（硬上限 5 ms）。这是逻辑预算，不是帧时间预算。

**无法直接断言的规则**

- **Core Rule 4（开局不扣款的叙事）**：这是表现层选择，由界面流程决定如何展示「进店前已付」；结算只保证收据上恰有一条扣款项。
- **Core Rule 18（钱包策略）**：净利是否进入钱包、是否允许为负、是否设下限，属于「局外升级」；本 GDD 不断言。
- **设计期校准带**：需要真实玩家数据，由 `/prototype` 与 `/playtest-report` 验证，不是自动化用例。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | **负净利如何进入钱包？** 本 GDD 只保证 `net_profit` 有符号且不钳制。**已回答（2026-11-03）**：局外升级采用全额入账 `wallet += net_profit`，允许负钱包，不按 0 计入——见 `design/gdd/meta-upgrades.md`。不同选择会直接决定"回本"是经济状态还是分数。 | economy-designer（局外升级） | 局外升级 GDD 完成时 |
| 2 | **MVP 是否提供主动提前结账？** 当前设计不提供：一局只在回合耗尽或没有任何合法动作时结束，提前结账不会带来任何优势。若界面流程希望给玩家"收手"的主动权，需要新增 `PLAYER_CASH_OUT` 结束原因，并明确它与"从暂停菜单放弃"的区别（后者不结算）。 | game-designer（餐局流程） | 餐局流程 GDD 完成时 |
| 3 | **纪录按哪个指标排名？** 建议按 `net_profit`（玩家看到的数、未来钱包的来源）。`return_ratio` 在单档口下是净利的单调变换，但跨档口比较更公平；两者都存入纪录，排名规则由纪录系统定。纪录还必须冻结 `station_id`、`entry_fee`、`total_value` 与升级快照，否则重调数值后历史不可比。 | game-designer（纪录与里程碑） | 纪录 GDD 完成时 |
| 4 | **首局引导的"一条替换建议"如何计算？** 结算只提供每行的静态/有效单位价值，不计算最优解。首局引导需要自己选一条建议（例如"把豪华海鲜锅换成炸鸡腿"），但不得展示完整最优组合。 | ux-designer（首局引导） | 首局引导 GDD 完成时 |
| 5 | **入场费 85 是否成立？** 它满足 `calibration_ratio = 1.224`，但未经真实玩家验证。设计期校准带给出重调规则：中位数 < 88 → 每轮降 3–5；> 96 → 每轮升 2–3，直到中位数落回 88–96。 | economy-designer + prototyper | `/prototype 点菜取舍与腻度` 之后 |
| 6 | **升级后如何重算校准？** **已回答（2026-11-03）**：入场费运行期不变；`[1.15, 1.35]` 只约束未升级上限，满级容量 25 的 120/85 = 1.412 必须落在满级带 `[1.35, 1.45]`——见 `design/gdd/meta-upgrades.md`。 | economy-designer（局外升级 + 结算） | 局外升级 GDD 完成时 |
| 7 | **收据是否每行都显示两列单位价值？** 当前要求静态与有效两列，教学价值最高，但信息密度也最高。UX 可以决定只在首局显示两列、常规局只显示有效值；无论哪种，数据都必须由结算提供，UI 不得自行计算。 | ux-designer（界面流程） | `/ux-design` 收据屏时 |
