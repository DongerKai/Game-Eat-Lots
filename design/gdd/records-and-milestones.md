# 纪录与里程碑

> **Status**: In Design
> **Author**: game-designer + systems-designer, qa-lead
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 数字看得见地涨 · 没有白吃的菜

## Summary

纪录与里程碑把每一局结算变成生涯账本上的一行：它按档口保存最佳一局、累计局数与终身净利，用稳定的里程碑 id 记录玩家达成的目标，并在全部必需里程碑完成时判定通关。它不重算任何数值，也不写文件——只消费结算产出的 `RunResult`，把结果整理成 `records` 命名空间交给存档。没有它，玩家每局结束只看到一张孤立收据，看不到自己比上次强在哪里。

> **Quick reference** — Layer: `Feature` · Priority: `Vertical Slice` · Key deps: `结算与入场费`, `存档`, `数据配置`

## Overview

纪录与里程碑是《回本》的长期记忆。一局结束时，「结算与入场费」把吃下的菜算成一张收据，这张收据在下一局开始后就消失了；本系统负责把其中值得留下的部分变成不会被后来的数值调整改写的记录：这个档口最好的一局赚了多少、总共打过多少局、累计赚了多少、哪些目标已经达成。

它有三件事。第一是**最佳局**：每个档口各存一条，按净利比较，冻结当时的入场费、总值、回本率、结束原因与升级快照——这些是「那一局发生了什么」的历史事实，只用于展示与比较，不参与任何运行期计算。第二是**生涯合计**：局数、终身净利、终身吃进总值，如实累加，亏损也计入。第三是**里程碑与完成条件**：里程碑是带稳定 id 的一次性成就，定义成数据，判定只读当前这一局与生涯合计；当所有标记为「完成必需」的里程碑都达成时，生涯通关。

本系统不发明数字，也不评价玩家。它不计算净利（结算拥有）、不决定钱包与升级（局外升级拥有）、不决定哪些档口可用（解锁与档口拥有）、不写盘（存档拥有）、不画界面（界面流程拥有）。它只回答一个问题：**和过去比，这一局算不算数。**

## Detailed Design

### Core Rules

1. **职责边界**：本系统拥有按档口的最佳局纪录、生涯合计、里程碑判定、完成条件与这些事实的时间戳。它不拥有净利与回本率的计算（结算与入场费）、钱包与升级（局外升级）、档口与菜品的解锁（解锁与档口）、持久化写入（存档）、界面绘制（界面流程）或排行榜（在线挑战）。它保存结算算完的值，不替结算重算。
2. **一局一次，只进不出**：本系统只在收到一份已结算的 `RunResult` 时更新一次。同一会话内 `run_id` 与上一次已应用的相同时，返回 `ALREADY_APPLIED`，不重复累计、不重复发信号。放弃、进入失败与局中退出从不产生 `RunResult`，因此从不产生纪录（结算规则 8）。
3. **输入只读，不重算**：净利、总值、回本率、每行明细一律取自 `RunResult`，不读菜品价格、不重放腻度、不重新求和。收到结果时做一次一致性断言：`net_profit = total_value − entry_fee`；`entry_fee > 0` 时 `return_percent_x10 = floor(1000 × total_value / entry_fee)` 且 `return_ratio` 非空；`entry_fee = 0` 时两者都为 `null`。任一不成立返回 `INCONSISTENT_RESULT`，纪录不变。这条断言防的是上游状态损坏被固化成一条永久纪录。该错误与 `OVERFLOW` 都是**结算事务的致命错误**：提交编排者必须整笔放弃本次结算的 `progression` / `unlocks` 更新，不能只让纪录失败、钱包照常入账——同一份结算结果不能在两个命名空间里变成两种事实。
4. **纪录按档口，不按全局**：每个档口各有一条最佳局纪录，以 `station_id` 为键。不同档口的入场费不同，绝对净利不可直接比较；按档口比较让每个档口都有自己的目标，也让「吃遍所有档口」的长期目标成立。
5. **排名指标是净利**：同一档口内，最佳局由 `net_profit` 决定（有符号 int64）。总值、入场费、回本率、结束原因与升级快照随纪录一起冻结，只用于展示与复盘。选净利而不是回本率，因为净利是玩家在收据上看到的数、也是未来钱包的来源（结算 Open Question 3）；在单一档口内两者单调一致，而净利能直接回答「这一局赚了多少」。
6. **严格更优才刷新，平局保留旧纪录**：新一局的 `net_profit` 严格大于旧纪录时才替换；恰好相等时旧纪录保留。平局不比较时间、不比较总值——比较一旦读时钟，纪录就依赖玩家机器的时钟回拨，历史也不再稳定。第一次结算没有旧纪录，无论净利正负都建立纪录。
7. **纪录冻结历史事实**：`entry_fee`、`total_value`、`return_ratio`、`end_reason` 与 `upgrade_snapshot` 在达成那一刻冻结，之后数据配置怎么调都不改写。这些字段只用于展示与比较，不参与任何运行期计算（存档规则 3）。档口从数据配置中移除后，它的纪录仍按键保留，用冻结值显示。
8. **负纪录不设下限**：最佳局可以是负数。亏损的「最佳」如实显示负号，不钳制为 0，不隐藏该档口尚未回本的事实。第一次就打亏的档口，纪录就是那次亏损。
9. **生涯合计如实累加**：`runs_settled`、`lifetime_net_profit`、`lifetime_value` 对每一份已结算结果累加，包含亏损局与空局。三个值都是有符号 int64，累加用 checked 运算；任何一步溢出返回 `OVERFLOW`，整次更新不生效，绝不回绕。合计是事实，不是最佳值。
10. **里程碑用稳定 id，不用序号**：里程碑 id 形如 `ms_*`，由数据定义，发布后不得改名。用序号会在插入新里程碑时改变历史含义（存档规则 291 行的前置契约）。已达成里程碑是单调的：一旦达成，永不撤销、永不删除，即使阈值后来被调高、定义被移除。
11. **判定时机与顺序**：一次结算内的处理顺序固定为：断言输入 → 更新档口纪录 → 累加生涯合计 → 按 id 升序逐条判定未达成里程碑 → 锁存完成条件 → 发信号。里程碑用更新后的生涯合计（第 N 局计入「累计 N 局」），用当前这一局的单局指标。任何一步失败，整次更新不生效，不产生部分记录。
12. **四种里程碑指标**：单局净利、单局回本率（以 0.1% 为单位的整数）、终身净利、累计结算局数。单局类指标可以限定档口；生涯类指标不能限定档口，限定即为数据错误。没有隐藏里程碑，没有随机里程碑——玩家看到的进度必须和判定用的一致（支柱 1）。
13. **不追溯授予**：之后新增的里程碑只从下一次结算起可达成，不回扫已有纪录。单局阈值无法从聚合数据忠实重建，用加载时刻补一个时间戳等于伪造达成日期；生涯类阈值最多晚一局达成，代价可接受。
14. **完成条件是数据驱动的派生态**：完成条件 = 「目录中至少存在一条必需里程碑，且全部 `required_for_completion = true` 的里程碑都已达成」。达成时把 `completed_at` 锁存为当时时间；此后新增必需里程碑**不会**撤销已完成状态——完成是历史事实，不是当前配置的函数。没有必需里程碑时，生涯永远不完成（公式里的存在量词为假），数据校验给出 `NO_COMPLETION_PATH` 警告。**锁存前**完成条件跟随当前目录：删除一条必需里程碑会放宽条件、可能让生涯立即达成，这类编辑属于「会改变完成难度」的数据改动，必须与通关条件一起评审；锁存后不再受目录改动影响。
15. **时间戳只做展示**：`achieved_at`、`earned_at`、`completed_at` 由本系统在接收结果时按 UTC ISO-8601 添加，三者共用同一次采样。任何逻辑（比较、排序、冲突、迁移、过期）都不得读取时间戳；时钟回拨不会重排、撤销或重发任何纪录与里程碑（存档规则 22 同口径）。
16. **只经存档命名空间持久化**：本系统不写文件、不碰存档信封、不自行提交事务。启动载入后通过 `get_namespace("records")` 读一次；每次更新返回一份完整的新 `records` 值，由结算提交编排者并入当次事务，与其他命名空间一起提交。写盘失败时纪录**不回滚**——一局已经结束，回滚纪录等于丢弃这一局的事实；保留内存值并标记 dirty，由存档在下一个写入时点重试（存档规则 18）。
17. **载入时做语义校验**：存档只保证结构合法，`records` 的语义由本系统校验。负数计数钳为 0 并写警告；缺少 `net_profit` / `total_value` / `entry_fee` 等语义字段的单档口纪录整条丢弃并写警告；只缺 `achieved_at` 时保留纪录、时间戳置 null 并写警告——时间戳不参与任何逻辑，不能为一个展示字段丢掉一条真实纪录；不认识的里程碑 id、档口 id 与字段原样保留。任何情况下不静默删除已达成里程碑。警告通过存档的 `LoadReport.warnings` 交出去。
18. **只读对外**：界面流程与音效反馈读取纪录快照、订阅信号；只有本系统修改 `records` 命名空间。其他系统不得写、不得补、不得修正本命名空间。
19. **不存逐局历史**：只存聚合与每档口最佳，不存「最近 N 局」列表。收据屏就是那一局的视图，再存一份历史只会在 `save_bytes` 上线性增长，而 MVP/Vertical Slice 没有任何消费者（存档 Open Question 7 的答案）。
20. **单机信任模型**：本地纪录不防篡改，也不假装可信。将来在线挑战需要可信数据时，由服务端解决，不得把本地 `records` 当作排行榜数据（存档 Open Question 5、在线挑战）。
21. **确定性与可测试性**：给定同一份起始 `records` 与同一份 `RunResult`，两次应用除时间戳外逐字段相同；不依赖帧率、随机数或 wall clock。时间戳是唯一允许变化的字段，测试必须把它排除在外。

### States and Transitions

本系统没有跨帧状态机，只有一次结算内的处理管线与一个持久化脏状态（脏状态归存档，见上）。

| 阶段 | 触发 | 下一个阶段 | 说明 |
|---|---|---|---|
| `Idle` | 等待结算结果 | `Validating` | 已载入 `records`，无待处理结果 |
| `Validating` | 收到 `RunResult` | `Applying` / `Idle` | 断言失败返回 `INCONSISTENT_RESULT`；重复 `run_id` 返回 `ALREADY_APPLIED` 并留在 `Idle` |
| `Applying` | 断言通过 | `Evaluating` / `Idle` | 计算新档口纪录与生涯合计；溢出返回 `OVERFLOW`，整次不生效 |
| `Evaluating` | 纪录与合计计算成功 | `Committing` | 按 id 升序判定里程碑，锁存完成条件 |
| `Committing` | 新命名空间就绪 | `Idle` | 返回 `RecordsChange`；信号在内存应用成功后发出，不等写盘 |
| `Committing` | 写盘失败 | `Dirty` | 保留内存值，由存档重试；不发第二条「已保存」 |
| `Dirty` | 下一次成功写入 | `Idle` | 与后续结算的更新合并，不重复发已发过的信号 |

**`records` 命名空间结构（设计级 schema）**

| 字段 | 类型 | 默认值 | 含义 |
|---|---|---|---|
| `stations` | map<StringName, StationRecord> | 空表 | 每个已结算过的档口一条最佳局纪录 |
| `career.runs_settled` | int64 | 0 | 已结算局数，含亏损与空局 |
| `career.lifetime_net_profit` | int64 | 0 | 终身净利，有符号 |
| `career.lifetime_value` | int64 | 0 | 终身吃进总值 |
| `milestones` | map<StringName, MilestoneEntry> | 空表 | 仅存已达成里程碑；未达成不占空间 |
| `completed_at` | String \| null | null | 完成条件首次达成的时间；锁存后不再改写 |

**`StationRecord`**

| 字段 | 类型 | 含义 |
|---|---|---|
| `best.net_profit` | int64 | 该档口最佳局的净利，可为负 |
| `best.total_value` | int64 | 冻结的总值 |
| `best.entry_fee` | int | 冻结的入场费 |
| `best.return_ratio` | float64 \| null | 冻结的回本率；`entry_fee = 0` 时为 null |
| `best.return_percent_x10` | int \| null | 冻结的展示值（0.1% 单位） |
| `best.end_reason` | enum | `TIME_EXHAUSTED` / `NO_EATABLE_DISH` |
| `best.upgrade_snapshot` | Dictionary | 原样冻结的 `RunResult.upgrade_snapshot`（已解析的本局修正值，语义归局外升级；本系统不解释、不重算） |
| `best.achieved_at` | String (UTC ISO-8601) | 该纪录达成时间，仅展示 |

**`MilestoneEntry`**

| 字段 | 类型 | 含义 |
|---|---|---|
| `earned_at` | String \| null | 达成时间；补记或结构缺失时为 null |
| `threshold_at_earn` | int64 | 达成时生效的阈值，冻结；阈值之后被调高也不改写 |

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 结算与入场费 | 上游 | 一份 `RunResult`：`run_id`、`station_id`、`end_reason`、`entry_fee`、`total_value`、`net_profit`、`return_ratio` / `return_percent_x10`、`upgrade_snapshot` | 硬 |
| 存档 | 上游 | `get_namespace("records")`；`commit({records: …})` / `mark_dirty("records")`；`LoadReport.warnings` 回传语义警告 | 硬 |
| 数据配置 | 上游 | 里程碑目录（`MilestoneData`：id、指标、档口过滤器、阈值、是否完成必需）；档口 id 与 `display_name`；前提 `Ready` | 硬（需扩展） |
| 餐局流程 | 无直接调用（间接上游） | 保证放弃 / 进入失败 / 退出不产生 `RunResult`；在 `Settling` 广播一次 `run_settled(RunResult)` | 软（边界） |
| 界面流程 | 下游 | 只读 `RecordsSnapshot`；纪录页、里程碑进度、完成横幅；收据屏的「新纪录」标记 | 硬（UI） |
| 音效反馈 | 下游 | `record_broken`、`milestone_earned`、`career_completed` 三个信号 | 软 |
| 在线挑战 | 下游（未来） | 可读取本纪录用于展示；不得把本地纪录当作可信排行榜数据 | 软 |

**对外接口（设计级）**

| 接口 | 输入 | 输出 | 说明 |
|---|---|---|---|
| `load(saved: Dictionary)` | 存档命名空间 | `RecordsLoadReport` | 载入后调用一次；做语义校验并产出警告列表 |
| `apply_run(result: RunResult)` | 已结算结果 | `RecordsChange` / 错误 | 纯更新：返回新命名空间与本次新达成的里程碑；不写盘 |
| `snapshot()` | — | `RecordsSnapshot` | 只读视图，供界面与音效读取 |
| `milestone_progress(id)` | 里程碑 id | `{current, threshold, unit, earned}` | 只读；用于展示「340 / 500」这类进度 |
| `record_broken(station_id, new_best, previous_best)` | — | 信号 | 新纪录替换旧纪录时发一次；首次建立纪录时 `previous_best = null` |
| `milestone_earned(milestone_id)` | — | 信号 | 每条新达成里程碑发一次，按 id 升序 |
| `career_completed()` | — | 信号 | 完成条件首次锁存时发一次；载入时补齐状态不发 |

**接口所有权**：本系统拥有 `records` 命名空间的全部字段、里程碑与完成条件的语义、比较规则与时间戳。存档只提供存放位置与原子性；界面流程只读；结算不得解释纪录。`RunResult` 的字段与求和规则归结算，本系统不复制一份求和逻辑。

**结算提交编排者的契约**：本系统不决定事务边界。`apply_run` 返回的新 `records` 值必须由同一个编排者与其他命名空间（局外升级的 `progression`、解锁的 `unlocks`）合并为**一次**提交，且该事务在写盘失败时整体延后重试、不逐命名空间回滚。`apply_run` 返回任何错误（`INCONSISTENT_RESULT` / `OVERFLOW` / `DATA_NOT_READY`）时，编排者必须中止整笔结算事务，不得提交其他命名空间。编排者身份尚未指定（见 Open Question 1）。

## Formulas

本系统定义四个可判定的规则：纪录比较、生涯累加、里程碑判定与完成条件。里程碑阈值本身是数据（`MilestoneData.threshold`），不是公式里的字面量；公式只定义「怎么比」。

### 纪录比较 `record_is_better`

The `record_is_better` formula is defined as:

`record_is_better(r_new, r_old) = (r_old = ⊥) ∨ (r_new.net_profit > r_old.net_profit)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `r_new.net_profit` | int64 | −entry_fee–(Σprice − entry_fee) | 结算 `RunResult.net_profit` | 本局净利，有符号 |
| `r_old.net_profit` | int64 \| ⊥ | 同上 | `records.stations[station_id].best` | 该档口旧纪录；从未结算过为 ⊥ |
| `record_is_better` | bool | {true, false} | calculated | 是否用本局替换旧纪录 |

**Output Range:** `{true, false}`。只有严格更大才为 true；相等为 false。**不比较时间、总值或回本率**——平局保留旧纪录，历史才稳定。

**Example:** 旧 `+19`，新 `+19` → `false`（保留旧纪录）；旧 `+19`，新 `+20` → `true`；旧 `⊥`，新 `−85` → `true`（第一次打亏也建立纪录）。

### 生涯累加 `career_accumulate`

The `career_accumulate` formula is defined as:

`runs_settled′ = runs_settled + 1`

`lifetime_net_profit′ = lifetime_net_profit + net_profit`

`lifetime_value′ = lifetime_value + total_value`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `runs_settled` | int64 | 0–INT64_MAX | `records.career` | 已结算局数 |
| `lifetime_net_profit` | int64 | −INT64_MAX–INT64_MAX | `records.career` | 终身净利，可负 |
| `lifetime_value` | int64 | 0–INT64_MAX | `records.career` | 终身吃进总值 |
| `net_profit` / `total_value` | int64 | 同上 | 结算 `RunResult` | 本局净利与总值 |

**Output Range:** 三个值各自用 checked int64 累加。任一步溢出返回 `OVERFLOW`，本次更新整体不生效（三道值都不变），不回绕、不饱和。正常玩法受局数与单价约束不可能溢出；检查是为了让非法数据变成可见错误，而不是悄悄写坏账本。

**Example:** 初始 `(0, 0, 0)`，结算 F1 `(+19, 104)` → `(1, +19, 104)`；再结算 F2 `(−31, 54)` → `(2, −12, 158)`；再结算空局 `(−85, 0)` → `(3, −97, 158)`。

### 里程碑判定 `milestone_met`

The `milestone_met` formula is defined as:

`milestone_met(m, u, c) = station_ok(m, u) ∧ metric_met(m.metric, m.threshold, u, c)`

`station_ok(m, u) = (m.station_id = "") ∨ (m.station_id = u.station_id)`

`metric_met(NET_PROFIT, τ, u, c) = u.net_profit ≥ τ`

`metric_met(RETURN_RATIO_X10, τ, u, c) = (u.return_percent_x10 ≠ ⊥) ∧ (u.return_percent_x10 ≥ τ)`

`metric_met(LIFETIME_NET_PROFIT, τ, u, c) = c.lifetime_net_profit ≥ τ`

`metric_met(RUNS_SETTLED, τ, u, c) = c.runs_settled ≥ τ`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `m.id` | StringName | `^ms_[a-z0-9_]+$` | `MilestoneData` | 稳定里程碑 id |
| `m.metric` | enum | 4 值 | `MilestoneData.metric` | `NET_PROFIT` / `RETURN_RATIO_X10` / `LIFETIME_NET_PROFIT` / `RUNS_SETTLED` |
| `m.threshold` | int64 | 见下 | `MilestoneData.threshold` | 阈值；单位随指标变化 |
| `m.station_id` | StringName | 空或已存在档口 | `MilestoneData.station_id` | 仅单局类指标可用；空表示不限档口 |
| `u` | RunResult | — | 结算 | 当前这一局 |
| `c` | Career | — | 本系统累加后的值 | 生涯合计 |

**Threshold units:** `NET_PROFIT` 与 `LIFETIME_NET_PROFIT` 是元（int64，可为负）；`RETURN_RATIO_X10` 是 0.1%（1000 = 100.0%）；`RUNS_SETTLED` 是局数。`RETURN_RATIO_X10` 在 `entry_fee = 0` 的档口永远不成立——数据校验必须在加载期拒绝「回本率指标 + 零入场费档口」的组合（`INVALID_MILESTONE_FOR_ZERO_FEE`），而不是让玩家永久追一个不可能的目标。生涯类指标带 `station_id` 同样在加载期拒绝（`INVALID_MILESTONE_SCOPE`）。

**Output Range:** `{true, false}`。判定用整数或未格式化的 float64，不使用任何展示格式。

**Example:** 目录含 `ms_first_break_even`（`NET_PROFIT`，无档口，τ = 0）；F1 `net_profit = +19` → true，F2 `−31` → false。`ms_canteen_return_1100`（`RETURN_RATIO_X10`，`station_canteen`，τ = 1100）：F1 展示 `122.3%` → `1223 ≥ 1100` → true；F2 `63.5%` → false。`ms_runs_10`（`RUNS_SETTLED`，τ = 10）：第 10 局结算后 `c.runs_settled = 10` → true。

**进度展示 `milestone_progress().current` 的取值来源**（只读，不参与判定）：

| 指标 | 当前值来源 | 无可用纪录时 |
|---|---|---|
| `RUNS_SETTLED` | `career.runs_settled` | `0` |
| `LIFETIME_NET_PROFIT` | `career.lifetime_net_profit` | `0` |
| `NET_PROFIT`，带档口过滤器 | 该档口 `best.net_profit` | `null`（显示「未开始」） |
| `NET_PROFIT`，无过滤器 | 所有档口 `best.net_profit` 的最大值 | `null` |
| `RETURN_RATIO_X10`，带档口过滤器 | 该档口 `best.return_percent_x10`（免单档口恒为 `null`） | `null` |
| `RETURN_RATIO_X10`，无过滤器 | 所有档口 `best.return_percent_x10` 的最大值 | `null` |

当前值只用于显示「340 / 500」这类进度，不参与达成判定；判定永远用当前这一局与生涯合计（见上）。`current = null` 时界面显示「未开始」，不显示 `0`——0 是一个打出来的成绩，`null` 是还没打过。

### 完成条件 `career_complete`

The `career_complete` formula is defined as:

`all_required_earned(M, E) = (∃ m ∈ M : m.required_for_completion) ∧ (∀ m ∈ M : (¬m.required_for_completion) ∨ (m.id ∈ E))`

`completed_at′ = completed_at if completed_at ≠ ⊥ else (now if all_required_earned else ⊥)`

`career_complete = (completed_at ≠ ⊥)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `M` | MilestoneData[] | ≥ 0 条 | 数据配置里程碑目录 | 当前全部里程碑定义 |
| `E` | set<StringName> | ⊆ M.id | `records.milestones` | 已达成 id 集合 |
| `m.required_for_completion` | bool | {true,false} | `MilestoneData` | 是否属于通关条件 |
| `completed_at` | String \| null | — | `records.completed_at` | 首次达成时间，锁存 |
| `career_complete` | bool | {true,false} | calculated（派生） | 是否已通关 |

**Output Range:** `{true, false}`。一旦 `completed_at` 非空，答案恒为 true：之后新增必需里程碑不会把已通关的生涯改回未通关。目录中没有 `required_for_completion = true` 的条目时，公式里的存在量词为假，`all_required_earned` **恒为 false**——不能依赖「空集合恒真」的数学约定，否则空目录会在第一局就宣布通关；数据校验同时给出 `NO_COMPLETION_PATH` 警告，但警告只是提示，真正拦住自动通关的是公式里的存在量词。

**Example:** 目录 6 条中 3 条必需，已达成 2 条 → false；第 3 条在本次结算达成 → `completed_at = now`，此后永久 true。之后再加入第 4 条必需里程碑 → 仍为 true，新条目作为通关后目标展示。

**Vertical Slice 基线目录（设计起点，阈值需在 `/prototype` 后校准）**

| id | 指标 | 档口 | 阈值 | 完成必需 | 说明 |
|---|---|---|---|---|---|
| `ms_first_settlement` | `RUNS_SETTLED` | — | 1 | 否 | 第一局结算完成 |
| `ms_first_break_even` | `NET_PROFIT` | — | 0 | 否 | 任意档口首次回本 |
| `ms_canteen_return_1000` | `RETURN_RATIO_X10` | `station_canteen` | 1000 | 是 | 食堂回本 |
| `ms_canteen_return_1100` | `RETURN_RATIO_X10` | `station_canteen` | 1100 | 否 | 超过回本线 10% |
| `ms_canteen_return_1200` | `RETURN_RATIO_X10` | `station_canteen` | 1200 | 否 | 接近静态最优（校准上限 1223） |
| `ms_runs_10` | `RUNS_SETTLED` | — | 10 | 否 | 累计 10 局 |
| `ms_runs_25` | `RUNS_SETTLED` | — | 25 | 否 | 累计 25 局 |
| `ms_lifetime_500` | `LIFETIME_NET_PROFIT` | — | 500 | 否 | 终身净利 500 |
| `ms_lifetime_2000` | `LIFETIME_NET_PROFIT` | — | 2000 | 否 | 终身净利 2000 |

阈值不是本 GDD 的公式常量，而是 `MilestoneData` 的数据；新增档口时，每个档口至少一条 `*_return_1000` 并标记为完成必需，通关条件才覆盖全部档口。**当前基线在 Vertical Slice 只有 2 个档口时才成立**；档口数量变化时必须重算完成条件的覆盖面，否则「吃遍所有档口」会名不副实。

## Edge Cases

- **If 生涯第一次结算**：没有旧纪录，无论净利正负都建立第一条档口纪录；`previous_best = null`，发出一次 `record_broken`。首次就打亏时纪录为负，如实显示，界面上不盖「新纪录」印章（净利 ≤ 0 不庆祝）。
- **If 空局（0 道菜）**：仍是一次完整结算。`runs_settled +1`、`lifetime_net_profit + (−entry_fee)`、`lifetime_value + 0`；若为首次，建立一条负纪录。单局类里程碑不会因此达成。
- **If 新一局净利与旧纪录恰好相等**：旧纪录保留，不发 `record_broken`；生涯合计照常累加。不使用权时间或总值做二次比较。
- **If 最佳局为负**：如实显示符号，不钳制为 0、不显示「尚无纪录」。该档口「最佳」仍是玩家真实打出的最好一局。
- **If `entry_fee = 0`**：`return_ratio` 与 `return_percent_x10` 为 null，回本率类里程碑不成立；净利类里程碑照常判定。数据校验必须在加载期拒绝「回本率里程碑 + 零费用档口」的组合。
- **If 新一局净利恰好为 0**：`≥ 0` 的回本类里程碑达成；纪录比较仍按严格大于，与旧负纪录比较时替换，与旧 0 纪录比较时保留。
- **If 同一 `RunResult` 被投递两次**：第二次返回 `ALREADY_APPLIED`，`runs_settled` 不增加、信号不重发。去重键是会话内 `run_id`，与结算的 `last_settled_run_id` 同口径。
- **If `RunResult` 字段自相矛盾**（`net_profit ≠ total_value − entry_fee`、回本率与费用不符）：返回 `INCONSISTENT_RESULT`，整次更新不生效、不发信号。不「就地修正」——修正会把损坏数据写进永久纪录。
- **If 同一帧收到两份不同 `run_id` 的结算**：按到达顺序逐份处理，每份独立判定与发信号；不允许交错处理（单线程主循环保证）。
- **If 纪录已应用到内存但写盘失败**：保留内存值并标记 dirty，界面仍显示新纪录；存档在下一个写入时点重试。不撤回 `record_broken`，不把已经发生的这一局从账本上抹掉。
- **If 载入时 `career` 计数为负**：钳为 0 并写 `CLAMPED_NEGATIVE_COUNTER` 警告；其余字段照常载入。负数说明文件被改坏或旧版本缺陷，钳制是恢复可玩状态的最小动作。
- **If 载入时某条 `StationRecord` 缺语义字段**（没有 `net_profit` / `total_value` / `entry_fee`）：整条丢弃并写 `STATION_RECORD_DROPPED` 警告，其他档口纪录与生涯合计不受影响。残缺的纪录无法可靠修复，保留半条会让展示与比较出错。
- **If 载入时某条 `StationRecord` 只缺 `achieved_at`**：保留整条纪录，`achieved_at = null`，写 `RECORD_TIMESTAMP_MISSING` 警告。时间戳不参与逻辑，不能为一个展示字段丢掉一条真实的净利纪录。
- **If 载入时某条 `MilestoneEntry` 缺 `earned_at`**：保留该 id，`earned_at = null`，写 `MILESTONE_TIMESTAMP_MISSING` 警告；仍算已达成。已达成的里程碑不能因为时间戳缺失而被撤销。
- **If 载入时遇到不认识的里程碑 id**：原样保留，不删除、不参与完成判定（当前目录没有它），在纪录页标为「已归档」。定义恢复后自然重新显示。
- **If 里程碑定义在达成后被移除**：已达成条目保留，不再出现在活动列表。若它是完成必需项且 `completed_at` 已锁存，通关状态不变；若尚未锁存，移除它会**放宽**完成条件（下一次结算可能立即通关），这是会改变完成难度的数据编辑，必须与通关条件一起评审。
- **If 通关后新增必需里程碑**：`completed_at` 已锁存，生涯仍是已通关；新里程碑作为通关后目标，从下一次结算起可达成。完成状态从不回退。
- **If 通关前新增里程碑**：只从下一次结算起可判定，不回扫旧纪录。生涯类阈值会在下一次结算自然满足；单局类需要再打一局达标。
- **If 里程碑目录没有任何完成必需项**：`NO_COMPLETION_PATH` 警告，`career_complete` 恒为 false；不按「空集合恒真」宣布通关。
- **If 里程碑目录存在重复 id、非法 id 或 `RUNS_SETTLED` 带档口过滤器**：数据校验失败，数据配置不进入 `Ready`，游戏停在 Error 屏（数据配置规则 7）。本系统拒绝在非法目录上判定。
- **If 必需里程碑指向已被移除的档口**：该里程碑永远无法达成，完成条件永久不可达。数据校验给出 `REQUIRED_MILESTONE_UNREACHABLE` 警告；是否阻断启动由数据配置决定，纪录系统只报告。
- **If 档口从数据配置中移除但仍有纪录**：纪录按键保留，纪录页用冻结值显示该档口；不删除、不改写。恢复档口定义后自然重新关联。
- **If 系统时钟回拨或被修改**：时间戳可能早于前一条纪录，但比较、排序、判定都不读时间。界面上按存储顺序显示即可；不因时间戳异常重建纪录。
- **If 终身合计溢出 int64**：返回 `OVERFLOW`，本次更新的三条合计与档口纪录都不生效；不饱和、不回绕。正常玩法不可达，测试用注入夹具覆盖。
- **If 载入时 `records` 命名空间整体缺失**（旧档迁移未补）：按空纪录继续，迁移负责补默认值（存档规则：旧档缺 `records` 字段补默认，不视为错误）；不影响启动。
- **If 玩家删除生涯**：存档隔离旧档并建立新生涯；内存中的 `records` 由本系统重置为空，`completed_at = null`。不做「纪念性保留」——删除生涯就是删除纪录。
- **If 玩家手工编辑存档**：单机信任模型，结构合法的新纪录值被接受（存档规则 25）。本系统只做类型与语义范围校验，不做防篡改。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 结算与入场费 | 硬 | 一份 `RunResult`：`run_id`、`station_id`、`end_reason`、`entry_fee`、`total_value`、`net_profit`、`return_ratio` / `return_percent_x10`、`upgrade_snapshot`；每局恰好一次 | GDD 已写（`design/gdd/settlement-and-entry-fee.md`） |
| 存档 | 硬 | `records` 命名空间的读写；`commit` / `mark_dirty`；`LoadReport.warnings`；载入前提是数据 `Ready` 之后 | GDD 已写（`design/gdd/save-system.md`） |
| 数据配置 | 硬 | 里程碑目录 `MilestoneData`（id、指标、档口过滤器、阈值、完成必需）；`get_station(id)` → `display_name`、是否存在；前提 `Ready` | GDD 已写，**需扩展第四类记录**（见下） |

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 界面流程 | 硬（UI） | 只读 `RecordsSnapshot` 与 `milestone_progress`；纪录页、里程碑列表、完成横幅；收据屏「新纪录」标记 | GDD 已写（`design/gdd/ui-flow.md`） |
| 音效反馈 | 软 | `record_broken`、`milestone_earned`、`career_completed` | GDD 已写（`design/gdd/audio-feedback.md`） |
| 在线挑战 | 软（未来） | 只读展示；不得信任本地纪录作为排行榜数据 | GDD 未写 |

**不属于依赖的关系：**

- **局外升级 / 解锁与档口**：与本系统同属结算的下游，但互不读写。钱包、升级等级与解锁集合各有自己的命名空间；本系统只读 `RunResult.upgrade_snapshot` 做冻结展示，不读取局外升级的当前等级。
- **餐局流程**：不直接调用本系统。它只保证「放弃 / 进入失败 / 退出不结算」，从而保证本系统不会收到这些结果。
- **局内 HUD**：不显示纪录，也不在局内比较历史最佳；本系统不向 HUD 提供接口。

**双向一致要求**：以下文档在编写或修订时必须列出「纪录与里程碑」并写明接口：

- `settlement-and-entry-fee.md`：下游表「纪录与里程碑」状态改为「GDD 已写」；Open Question 3（按哪个指标排名）由本 GDD 回答：**按净利排名**，同时冻结总值、回本率、入场费与升级快照。
- `save-system.md`：下游表「纪录与里程碑」状态改为「GDD 已写」；Open Question 7（是否要逐局历史）由本 GDD 回答：**不要**，MVP/Vertical Slice 只存聚合与每档口最佳。
- `data-config.md`：下游表新增「纪录与里程碑」一行——本系统需要里程碑目录与档口 `display_name`。这是本 GDD 对数据配置提出的唯一结构性扩展。

**对数据配置的扩展契约（provisional）**：里程碑阈值是可调的玩法数值，按编码标准必须数据驱动；数据配置是唯一数值来源（数据配置规则 1）。因此本系统要求新增第四类记录 `MilestoneData`：

| 字段 | 类型 | 校验 |
|---|---|---|
| `id: StringName` | 稳定唯一 | `^ms_[a-z0-9_]+$`；发布后不得改名 |
| `display_name: String` | 展示名 | 非空 |
| `description: String` | 达成条件文案 | 非空 |
| `metric: enum` | `NET_PROFIT` / `RETURN_RATIO_X10` / `LIFETIME_NET_PROFIT` / `RUNS_SETTLED` | 固定四值 |
| `threshold: int64` | 阈值 | 单位随指标；`RUNS_SETTLED ≥ 1`、`RETURN_RATIO_X10 ≥ 1`；两个净利指标可为任意 int64（含负） |
| `station_id: StringName` | 档口过滤器 | 空或已存在档口；仅单局类指标可非空 |
| `required_for_completion: bool` | 是否通关必需 | 至少一条为 true，否则 `NO_COMPLETION_PATH` 警告 |

若编排评审决定不扩展数据配置，则本系统必须自带一个只读的里程碑资源加载器，并在启动序列里与数据配置一起校验；两条路只能选一条，不能一边说「唯一数值来源」一边把里程碑阈值写在代码里。

**对存档的契约澄清（provisional）**：存档规则 18 规定「纪录写盘失败保留内存并重试」，规则的状态表又规定「一次结算引起的钱包、纪录与解锁变化合并为同一个事务」。两条同时成立时，混合事务的失败策略必须明确：本系统要求**结算来源的事务整体延后重试**（钱包、纪录、解锁一起留在内存），而不是钱包回滚、纪录保留——否则同一份结算结果会在两个命名空间里出现不同的事实。购买来源的事务仍按存档规则 17 回滚，两者按事务来源区分。若 `apply_run` 返回错误，该事务**不提交**（不是延后重试），其他命名空间也不得写入。

**对未设计系统的前置契约（provisional）：**

| 系统 | 本系统要求 | 若最终实现不同 |
|---|---|---|
| 界面流程 | 在 `Menu` 内提供纪录页；读取 `snapshot()`，不自行计算比较或完成条件；收据屏只在 `record_broken` 后显示「新纪录」 | 若界面自行比较，会出现「界面说破纪录但数据没变」 |
| 音效反馈 | 三个信号各自对应一个短音效，互不叠加；完成音效只在 `career_completed` 时响一次 | 若按界面轮询状态播放，重载后会重复庆祝 |
| 在线挑战 | 只把本地纪录当展示数据；排行榜数据必须来自服务端或可复现校验 | 若上传本地 `records` 当分数，单机信任模型会让排行榜失效 |

## Visual/Audio Requirements

纪录与里程碑本身不绘制画面，但它决定玩家看到的**账本**长什么样。视觉目标与结算一致：像一本可以翻的账本，不像一排成就弹窗。

- **纪录页是账本，不是奖杯柜**：沿用「食堂价目板 / 界面像收据」的视觉锚点，米白纸底、粗分隔线、等宽数字。每个档口一行：档口名、最佳净利（带符号）、回本率、总值、达成日期。
- **新纪录是一枚图章**：收据屏上以一次性印章动效盖在净利旁（≤ 0.4 秒，可跳过），不使用彩带、闪光或全屏特效。破的是自己的纪录，不是打败了谁。
- **里程碑是账本上的勾**：已达成条目用勾选与日期表示，未达成条目显示进度（`340 / 500`），不显示锁形图标或灰色不可见条目。没有隐藏目标。
- **完成是唯一的高光时刻**：通关时允许一次完整的横幅与音效（≤ 1.5 秒），此后进入「通关后目标」状态，不再重复庆祝。
- **亏损纪录不用红色**：回本红只用于正收益；负的最佳局用中性墨色加负号，与结算屏的盈亏语义保持一致。
- **音效事件**（由音效反馈实现，本系统只定义语义）：`record_broken`（轻短图章声）、`milestone_earned`（翻页/勾选声）、`career_completed`（一次性完整收尾音）。三者互斥，不叠加。本系统对每条新达成里程碑**各发一次** `milestone_earned`；同一帧多次到达时，由音效反馈合并为一次播放。

> `art-director` not consulted — Lean mode（本系统是 Progression 数据层，视觉由既有账本隐喻决定）。进入生产前需人工复核。

## Game Feel

- **像翻账本，不像看成就**：玩家打开纪录页的动机是「我上次打到多少」，不是「还差几个成就」。第一屏必须直接给出每档口最佳与差额，成就列表在下方。
- **比较是即时的**：结算页出现「新纪录」时，玩家不点任何东西就能看到旧值和新值并排；只显示「新纪录」而不同时显示旧值，等于让玩家自己去记。
- **进度可读**：未达成里程碑显示当前值 / 阈值 / 单位，玩家能心算出还差多少（支柱 1）。不允许出现「接近达成」这种模糊文案。
- **不制造重复庆祝**：一条里程碑只庆祝一次，重载、切屏、再进菜单都不重播。重复庆祝会让玩家怀疑数据是否真的保存了。
- **失败不羞辱**：负的最佳局照实显示，不加「继续努力」；账本陈述事实。

## UI Requirements

本系统不拥有界面，但对界面流程拥有的屏幕提出约束：

1. **纪录页入口**：放在主菜单的一级位置（与「开始营业」「升级」并列），不埋在设置里。玩家在决定「再来一局」之前应该能查到自己最好打到多少。
2. **每档口一行**：档口名、最佳净利（带 `+` / `−` 符号）、回本率（`122.3%`，免单档口显示「免单」）、总值、达成日期。新档口尚无纪录时显示明确的空状态行，不显示空白或 0。
3. **里程碑列表**：按「未达成」与「已达成」分组；每条显示名称、条件文案、进度 `当前值 / 阈值 单位`、达成日期（有则显示）。不显示隐藏条目。
4. **完成状态**：未通关时显示「距离通关还差 N 项」并列出必需项；已通关时显示一次性横幅与「通关后目标」分区。
5. **收据屏集成**：`record_broken` 后并排显示旧最佳；「新纪录」标记只在新纪录 `net_profit > 0` 时出现——负纪录建立或刷新不加印章。没有破纪录时不显示任何安慰性文案。
6. **空生涯**：一局未打时，纪录页显示「还没有账本」的明确空状态，不显示全 0 表格。
7. **无障碍**：正负不能只靠颜色，必须有符号与文字；字号 150% 时纪录行不换行错位、金额列不重叠；所有列表可用键盘浏览。完整的无障碍要求见未来的 `design/accessibility-requirements.md`。

> **📌 UX Flag — 纪录与里程碑**：本系统有 UI 需求（纪录页、里程碑列表、完成横幅、收据屏「新纪录」标记）。在 Phase 4（Pre-Production）写 epic 之前，运行 `/ux-design` 为纪录页产出 UX 规格；相关 story 引用 `design/ux/records.md` 与 `design/ux/settlement.md`，不要直接引用本 GDD。

## Cross-References

| 文档 | 关系 | 本系统引用/依据 |
|---|---|---|
| `design/gdd/settlement-and-entry-fee.md` | 上游 | `RunResult` 字段、每局一次、时间戳由下游添加、Open Question 3（排名指标） |
| `design/gdd/save-system.md` | 上游 | `records` 命名空间、`commit` / `mark_dirty`、失败重试策略、版本迁移、Open Question 7（逐局历史） |
| `design/gdd/data-config.md` | 上游 | 稳定 id 规则、`Ready` 门、档口 `display_name`；需新增 `MilestoneData` |
| `design/gdd/run-flow.md` | 边界 | 放弃 / 进入失败 / 退出不结算，因此不产生纪录；`run_id` 会话内唯一 |
| `design/gdd/game-concept.md` | 来源 | 长期目标「吃遍所有档口、达成里程碑」、最高净利纪录、完成全部档口挑战即通关 |
| `design/gdd/systems-index.md` | 索引 | 优先级 Vertical Slice、层 Feature、设计顺序 #16 |

**待回指**：在线挑战的 GDD 写好后，必须在上表中列出「纪录与里程碑」。结算与入场费、存档、界面流程、音效反馈已完成回指；数据配置的 `MilestoneData` 仍待其 Open Question 2 定归属。

## Acceptance Criteria

测试夹具放在 `tests/fixtures/records/`，每条用例使用独立的起始 `records` 与 `RunResult`，不共享可变状态。时间戳在断言中排除（只断言非空与格式），其余字段逐项比较。第二档口夹具 S1：`station_noodle`，`entry_fee = 65`、`total_value = 60`、`net_profit = −5`、`return_percent_x10 = 923`。

**运行时（玩家可见行为）**

1. **GIVEN** 一份空纪录、首次结算 F1（`net_profit = +19`），**WHEN** 应用结果并打开纪录页，**THEN** `station_canteen` 出现一条纪录：`+19`、回本率 `122.3%`、总值 104、日期非空；`runs_settled = 1`、`lifetime_net_profit = 19`、`lifetime_value = 104`，`record_broken` 发一次且 `previous_best = null`。**截图留存** `production/qa/evidence/`。
2. **GIVEN** 已有 `+19` 的档口纪录，结算一局 `+20` 的更好结果，**WHEN** 应用并查看收据屏，**THEN** 纪录被替换为新值，收据屏显示「新纪录」并并排显示旧值 `+19`；`record_broken` 恰好发一次。**截图留存**。
3. **GIVEN** 已有 `+19` 的纪录，结算另一局 `+19`，**WHEN** 应用，**THEN** 旧纪录保留（包括其 `achieved_at` 与升级快照），不发 `record_broken`，收据屏不显示「新纪录」；`runs_settled` 仍 +1。
4. **GIVEN** 首次结算为空局 F3（`net_profit = −85`），**WHEN** 打开纪录页，**THEN** 该档口显示 `−85` 而不是空状态或 0，且不显示「无纪录」、不盖「新纪录」印章；**截图留存**。
5. **GIVEN** `station_canteen` 已纪录 F1（`+19` / `122.3%`），随后 `station_noodle` 结算 S1（`−5` / `92.3%`），**WHEN** 打开纪录页，**THEN** 两行分别为 `station_canteen +19 122.3%` 与 `station_noodle −5 92.3%`，互不覆盖。
6. **GIVEN** 完成目录中 3 条必需里程碑中的前 2 条，**WHEN** 第 3 条在本次结算达成，**THEN** 纪录页显示完成横幅，`career_completed` 发一次；重新进入纪录页不重复播放庆祝（观察项，不作为截图断言）。**截图留存**。
7. **GIVEN** 一条未达成里程碑（阈值 500，当前 340），**WHEN** 打开纪录页，**THEN** 显示 `340 / 500` 的进度，而不是「接近达成」或空白。**截图留存**。
8. **GIVEN** 一局未打的全新生涯，**WHEN** 打开纪录页，**THEN** 显示明确的「还没有账本」空状态，不显示全 0 表格，不崩溃。**截图留存**。
9. **GIVEN** 任意已通关生涯，**WHEN** 浏览纪录页，**THEN** 页面不存在「最近 N 局」列表或逐局明细入口，`records` 命名空间中也不存在 history / recent 类字段；只显示聚合与每档口最佳。
10. **GIVEN** 一局净利为负但刷新了该档口纪录（旧纪录更负，例如从 `−20` 到 `−5`），**WHEN** 在纪录页显示，**THEN** 用中性墨色加负号显示，不使用回本红或庆祝特效。**截图留存**。
11. **GIVEN** 免单档口夹具 F6（`entry_fee = 0`、`total_value = 54`、`net_profit = 54`、回本率 `null`），**WHEN** 应用并打开纪录页，**THEN** 该档口纪录净利 `+54`、回本率显示「免单」；`NET_PROFIT` 阈值 0 的里程碑达成，目录中不存在绑定该档口的 `RETURN_RATIO_X10` 里程碑。
12. **GIVEN** 恰好回本夹具 F5（`net_profit = 0`、`return_percent_x10 = 1000`），**WHEN** 应用，**THEN** `NET_PROFIT` 阈值 0 的里程碑达成；**GIVEN** 该档口旧纪录为 `−5`，**THEN** 新纪录替换为 `0`；**GIVEN** 旧纪录为 `0`，**THEN** 保留旧纪录、不发 `record_broken`。

**逻辑 / 单元测试**

13. **GIVEN** 旧纪录 `+19`，**WHEN** 分别用 `+19`、`+20`、`+18` 比较，**THEN** `record_is_better` 依次为 false、true、false；**GIVEN** 无旧纪录，新值 `−85`，**THEN** 为 true。
14. **GIVEN** 初始 `(0, 0, 0)`，**WHEN** 依次应用 `(+19, 104)`、`(−31, 54)`、`(−85, 0)`，**THEN** 合计依次为 `(1, 19, 104)`、`(2, −12, 158)`、`(3, −97, 158)`。
15. **GIVEN** 四条里程碑夹具，**WHEN** 判定，**THEN**：`NET_PROFIT τ=0` 对 F1 `+19` 为 true、对 F2 `−31` 为 false；`RETURN_RATIO_X10 τ=1100` 对 F1 `1223` 为 true、对 F2 `635` 为 false；`LIFETIME_NET_PROFIT τ=500` 在合计 `(2, −12, 158)` 时为 false；`RUNS_SETTLED τ=10` 在 `runs_settled = 9` 时为 false、`10` 时为 true。
16. **GIVEN** 一条 `RETURN_RATIO_X10` 里程碑绑定 `station_canteen`，**WHEN** 用 `station_noodle` 的 `RunResult`（其回本率达标）判定，**THEN** 不达成；**GIVEN** 无档口过滤器的同阈值里程碑，**THEN** 达成。
17. **GIVEN** 一条里程碑已达成、阈值后来从 1100 调到 1300，**WHEN** 重新载入并再结算一局低分，**THEN** 已达成条目保留、`threshold_at_earn = 1100`、不撤销、不重复发 `milestone_earned`。
18. **GIVEN** 已达成一条旧里程碑，**WHEN** 目录新增一条本应满足的单局里程碑并立即结算一局不达标的结果，**THEN** 新里程碑不追溯授予。
19. **GIVEN** 3 条必需里程碑已达成、`completed_at` 已锁存，**WHEN** 目录新增第 4 条必需项，**THEN** `career_complete` 仍为 true；**GIVEN** 目录没有任何必需项，**THEN** `career_complete` 为 false 并产生 `NO_COMPLETION_PATH` 警告。
20. **GIVEN** 一次结算同时达成 `record_broken` 与两条里程碑，**WHEN** 应用，**THEN** 信号顺序固定为 `record_broken` → 按 id 升序的两次 `milestone_earned` →（若完成）`career_completed`，每条各一次。
21. **GIVEN** 一个已通关的生涯，**WHEN** 提交、退出并重新启动游戏，**THEN** 载入后 `career_complete` 为 true、`completed_at` 与重启前相同、不重发 `career_completed`、纪录页不重播庆祝横幅。
22. **GIVEN** 同一份 `RunResult` 连续应用两次，**WHEN** 第二次调用，**THEN** 返回 `ALREADY_APPLIED`，`runs_settled` 不变、信号不重发。
23. **GIVEN** 一份 `net_profit ≠ total_value − entry_fee` 的 `RunResult`，**WHEN** 应用，**THEN** 返回 `INCONSISTENT_RESULT`，`records` 逐字段不变；**GIVEN** `entry_fee = 0` 但 `return_percent_x10` 非空，**THEN** 同样拒绝。**GIVEN** 同一事务还含钱包变化，**THEN** 编排者中止整笔事务，钱包与解锁逐字段不变（编排者指定后可执行，见 Open Question 1）。
24. **GIVEN** 载入数据含 `runs_settled = −3`、一条缺 `net_profit` 的档口纪录、一条只缺 `achieved_at` 的档口纪录、一条缺 `earned_at` 的里程碑，**WHEN** 载入，**THEN** 计数钳为 0、缺 `net_profit` 的纪录被丢弃、只缺时间戳的纪录保留且 `achieved_at = null`、该里程碑保留；`LoadReport.warnings` 分别含 `CLAMPED_NEGATIVE_COUNTER`、`STATION_RECORD_DROPPED`、`RECORD_TIMESTAMP_MISSING`、`MILESTONE_TIMESTAMP_MISSING`，其余数据不变。
25. **GIVEN** 一个受控累加器夹具（两条 `lifetime_net_profit` 之和越过 int64 上限），**WHEN** 应用，**THEN** 返回 `OVERFLOW`，三条合计与档口纪录均不生效、不回绕。
26. **GIVEN** 同一起始 `records` 与同一 `RunResult` 的两个副本，**WHEN** 分别应用，**THEN** 两份结果除时间戳外逐字段深度相等；改变系统时钟不改变任何判定或排序。
27. **GIVEN** 数据配置不在 `Ready` 或里程碑目录含重复 id / 非法 id / 生涯指标带档口过滤器，**WHEN** 应用或载入，**THEN** 返回结构化错误 `DATA_NOT_READY` / `MILESTONE_CATALOGUE_INVALID`，不产生部分更新。
28. **GIVEN** 一次成功应用后写盘失败，**WHEN** 检查内存与界面，**THEN** 新纪录仍在内存中显示、命名空间标记 dirty、不发第二条信号；下一次成功写入后磁盘与内存一致。**GIVEN** 该事务同时含钱包变化，**THEN** 两者一起延后重试，不出现钱包已回滚而纪录保留的分裂（该子项待 Open Question 1 指定编排者后可执行）。
29. **GIVEN** 载入时遇到不认识的里程碑 id 与档口 id，**WHEN** 保存后再载入，**THEN** 两者原样保留、不被删除、不参与完成判定；**GIVEN** `records` 命名空间整体缺失，**THEN** 按空纪录继续，首局结算正常工作。
30. **GIVEN** 一条已冻结的档口纪录，**WHEN** 数据配置把该档口 `entry_fee` 从 85 改为 99、或把档口 `display_name` 改名，**THEN** 纪录中的 `entry_fee`、`total_value`、回本率与快照逐字段不变；只有展示名跟随数据配置变化。
31. **GIVEN** 玩家删除生涯，**WHEN** 新纪录页打开，**THEN** `records` 为空、`completed_at = null`，不保留旧生涯的任何纪录。

**设计期 / CI（不进入运行时）**

32. **GIVEN** Vertical Slice 里程碑目录，**WHEN** 运行数据校验，**THEN** 所有 id 匹配 `^ms_[a-z0-9_]+$` 且唯一；至少一条 `required_for_completion = true`；`RUNS_SETTLED` 与 `RETURN_RATIO_X10` 的阈值 ≥ 1；没有任何 `RETURN_RATIO_X10` 里程碑绑定零入场费档口；没有生涯指标带 `station_id`；每条必需里程碑的目标档口存在（否则 `REQUIRED_MILESTONE_UNREACHABLE`）。
33. **GIVEN** 4 个档口、每个档口 1 条纪录、目录 24 条里程碑全部达成，**WHEN** 估算 `save_bytes`，**THEN** 体积不超过 `SAVE_BYTES_MAX`；新增里程碑或档口时该估算随目录一起更新。
34. **GIVEN** 20 行以内的正常结算结果，**WHEN** 在 headless 基准中应用 1000 次（丢弃前 100 次预热，取中位数），**THEN** 单次更新中位数 ≤ 1 ms、p99 ≤ 5 ms。这是逻辑预算，不是帧时间预算。

**无法直接断言的规则**

- **Core Rule 20（单机信任）**：本地纪录可被手工编辑，本系统不防篡改；排行榜可信性属于在线挑战的服务端设计。
- **完成条件的叙事意义**：通关横幅是否让玩家感到「吃遍了」，需要 `/playtest-report` 验证，不是自动化断言。
- **阈值校准**：基线目录的 500 / 2000 / 1100 / 1200 是设计起点，必须由 `/prototype` 与真实对局数据校准。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | **谁拥有「结算提交编排者」？** 本系统返回新的 `records` 值，但把 `progression`、`records`、`unlocks` 合并为同一事务的调用者尚未指定。由界面流程、局外升级还是场景流程承担？该决定同时决定混合事务的失败策略（见 Dependencies 的存档契约澄清）。 | lead-programmer + technical-director | 架构阶段 |
| 2 | **`MilestoneData` 放进数据配置，还是纪录系统自带资源加载器？** 数据配置现有四类记录（Dish / Upgrade / Station / Balance）；若选择放进数据配置，`MilestoneData` 将成为第五类记录，否则由纪录系统自带只读加载器。两条路只能选一条。 | technical-director（数据配置 + 纪录） | 架构阶段 |
| 3 | **里程碑阈值如何校准？** 当前基线的 1100 / 1200 依据食堂静态最优 1223；多档口、升级生效后可达上限会变化。需要 economy-designer 在 `/prototype` 后按每档口的 `calibration_ratio` 重算，并保证每条必需里程碑可达。 | economy-designer + prototyper | Vertical Slice 设计前 |
| 4 | **完成条件是否应该解锁内容？** **已回答（2026-11-03）**：MVP/VS 完成条件不发放解锁；若未来发放，必须并入结算事务——见 `design/gdd/unlocks-and-stations.md`。 | game-designer（解锁与档口 + 纪录） | 解锁与档口 GDD 完成时 |
| 5 | **是否需要跨档口的生涯最佳（而非每档口最佳）？** 当前只存每档口最佳，纪录页可派生「所有档口里的最高净利」。若界面要单独展示「生涯最高一局」，需明确跨档口比较是否公平（不同入场费下的绝对净利不可比）。 | ux-designer（界面流程 + 纪录） | 纪录页 UX 规格时 |
| 6 | **逐局历史是否在 Full Vision 重启？** 当前明确不存。若未来要「最近 10 局」或回放，需要 schema 版本升级与单独的容量上限；`run_id` 目前是会话内唯一，跨会话历史需要持久化计数器或全局 id。 | game-designer + lead-programmer | Full Vision 规划时 |
| 7 | **时间戳的展示时区与格式由谁决定？** 存储为 UTC ISO-8601；纪录页显示本地时间还是 UTC、是否显示相对时间（「3 天前」），属于 UX 规格。任何显示格式都不得参与逻辑。 | ux-designer | `/ux-design` 纪录页时 |
