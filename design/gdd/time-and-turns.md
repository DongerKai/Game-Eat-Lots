# 时间与回合（Time and Turns）

> **Status**: In Design
> **Author**: user + Claude（autonomous 模式；Lean 模式不派生子代理）
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 时间在走，手不用快 · 每口都是算术题

## Summary

时间与回合拥有《回本》一局的“营业时段”：开局从档口取的回合预算，以及每一次吃菜或付费跳波扣掉的一格。它把时间做成有限、可数的资源，让“还能吃几口”和“胃还能装几格”同时约束选择；它没有真实时钟，不按秒走，也不惩罚思考。玩家看到的剩余时间格、结算收据上的回合明细，以及 `TIME_EXHAUSTED` 结束判据都由它提供。

> **Quick reference** — Layer: `Core` · Priority: `MVP` · Key deps: `餐局流程`（协调的下游：`结算与入场费, 局内 HUD`）

## Overview

时间与回合是《回本》里的一只**离散的钟**。一局开始时它拿到本局的回合预算 `turns_total`，把剩余格数设为满值；玩家每提交一次花时间的动作，它就恰好扣掉一格。它只回答三个问题：这一局原本有几格、现在还剩几格、是不是已经归零。它不决定哪些动作花时间，也不决定一局什么时候结束——前者属于餐局流程，后者属于餐局流程与菜单波次；它只保证“格子只减不增、不会透支、归零即报”。

它不拥有任何一个动作语义。吃一道菜花 1 回合、付费跳波花 1 回合、最后一波“结束取餐”与波次自动推进花 0 回合，这些规则分别写在餐局流程与菜单波次里；本系统只负责按调用扣减。它也不拥有界面：剩余时间格的画法、位置与动效属于局内 HUD，本系统只保证扣减在同一帧可读。它更不是实时系统——没有倒计时、没有按秒扣格、没有“思考太久会损失时间”。

没有它，每个动作都要各自判断“这一口该不该扣时间”，扣减会散落在吃菜、跳波和以后的任何新动作里，回合总数也会变成各系统各自记住的一份副本。它主要服务支柱 3「时间在走，手不用快」——压力来自还剩几个格，不来自反应速度；也服务支柱 1「每口都是算术题」——回合与胃容量是同一道题里的两个上限，而不是两个模糊的“资源”。

**与胃容量的分工**：两者都是每局预算，但约束的对象不同。回合限制**动作次数**（吃与跳共用一份），容量限制**一次能吃多少**；回合在中途耗尽，容量在吃饱后放不进下一道菜。MVP 的 10 回合与 20 格初始容量被刻意调到同一点咬合：最优餐恰好 10 道菜、20 格，任意一个更宽都会让另一个不再承重（见 `dish-values.md` 的 `opt_value` / `calibration_ratio`）。

## Detailed Design

### Core Rules

1. **职责边界**：本系统拥有**每局的回合预算、扣减、剩余/已用读数与耗尽判据**。它不拥有：哪个动作花回合（餐局流程）、跳波与最后一波“结束取餐”的语义（菜单波次）、结束时机与结束原因的优先级（餐局流程）、回合数字的绘制（局内 HUD）、跨局持久化（存档）。它没有自己的平衡数值——唯一的可调值是档口的 `turn_budget`，存在数据配置里。

2. **回合是离散单位，不是时间**：1 回合 = 1 次已提交且计入时间的动作。没有秒、没有小数、没有“每秒扣一格”。**等待、思考、暂停、失焦、窗口最小化、结算动画都不消耗回合**；唯一的消耗来源是对 `spend_turn()` 的调用。这是支柱 3 的硬边界：任何按真实时间记账的实现都算违反本 GDD。

3. **预算来自档口数据**：`turns_total = StationData.turn_budget`。`turn_budget` 是 `StationData` 的必填字段（整数，≥ 1），由数据配置在加载期校验。**餐局流程解析该字段后把整数传给本系统；本系统不查询数据目录**，因此不新增一条指向数据配置的运行时依赖。

4. **初始化即重置**：`initialize(turn_budget)` 把 `turns_total` 设为该值、`turns_remaining` 设为 `turns_total`、`turns_used` 设为 0，状态进入 `Active`。它是本系统唯一的入口；每一局恰好调用一次。`turn_budget < 1` 时返回结构化错误 `INVALID_TURN_BUDGET`，保持原状态，**不钳到 1、不生成 0 回合的局**。

5. **一次消耗恰好一格**：`spend_turn()` 的代价恒为 1，没有多回合动作、没有折扣、没有返还。本系统不提供“加回合”或“退回合”的接口；未来的时间类升级必须在开局冻结进预算，而不是局中修改（见 Open Question 3）。

6. **唯一调用者是餐局流程**：餐局流程完成动作校验后，在同一提交内调用 `spend_turn()` **恰好一次**，然后才广播 `action_committed` 并写入 `RunLog`。菜单波次只发出“想跳波”的意图，不直接扣减；局内 HUD 与音效反馈只读，不调用。多个调用者会让“谁扣过”无法审计。

7. **不可透支**：`turns_remaining = 0` 时，`spend_turn()` 返回 `REJECTED_EXHAUSTED` 且**不改变任何状态**；剩余格数永远不会是负数。本系统不做钳制式兜底——调用方必须先问 `can_spend_turn()`，再用 `spend_turn()` 提交；两者分开是为了让“能不能”成为可测试的纯读取，而不是靠副作用试探。

8. **耗尽判据只报告，不裁决**：`is_exhausted() = initialized ∧ (turns_remaining = 0)`。本系统不保存 `end_reason`，也不排序结束原因；`TIME_EXHAUSTED` 优先于 `NO_EATABLE_DISH` 的规则写在餐局流程里。餐局流程在初始化后判定一次、每次成功提交动作后判定一次，本系统只保证判据在同一帧为真。

9. **预算在本局内冻结**：开局后修改 `StationData.turn_budget`（热重载、调试、未来的升级）不影响当前局；下一局 `initialize()` 才读取新值。局中预算漂移会让玩家刚看到的剩余格数变成谎话。

10. **MVP 没有时间类升级**：`RunConfig.upgrade_snapshot` 只有 `capacity_bonus`、`cloying_resistance`、`menu_vision`，没有时间字段。若未来加入，必须作为 `upgrade_snapshot` 的新字段并在开局冻结，不能由本系统自行读升级表。

11. **不发信号**：本系统没有 `turn_*` 信号。状态变化通过餐局流程的 `action_committed` / `run_ended` 之后的只读快照被观察；多一条信号就多一个可能漂移的事实来源。局内 HUD 在快照里读 `turns_remaining` / `turns_total`，与读 `capacity_used` 是同一个时机。

12. **结束后只读**：`run_ended` 之后数值冻结，供 `RunLog` 与结算读取；直到下一次 `initialize()` 才重置。局外界面（菜单、升级、结算页）不得直接读取本系统的当前读数——结算只使用 `RunLog` 里冻结的值，也不能据此显示“上一局还剩几格”。

### States and Transitions

| 状态 | 含义 | 进入条件 | 离开 |
|---|---|---|---|
| `Uninitialized` | 没有进行中的局；`turns_total = turns_remaining = turns_used = 0`，`initialized = false` | 系统加载；从未初始化 | `initialize()` 成功 → `Active`；失败 → 停留 |
| `Active` | 本局进行中，`1 ≤ turns_remaining ≤ turns_total` | `initialize()` 成功 | 一次成功消耗把 `turns_remaining` 减到 0 → `Exhausted`；新的 `initialize()` → `Active`（整局重置） |
| `Exhausted` | `turns_remaining = 0`，`is_exhausted() = true` | 消耗把剩余格数减到 0 | 本局内没有离开路径；下一次 `initialize()` → `Active` |

**转移规则：**

- `initialize()` 可以从任意状态调用，效果都是**完整替换**上一局：旧的 `turns_total` / `turns_remaining` / `turns_used` 不延续。餐局流程只在开局调用它；局中重复调用属于生命周期 bug，本系统无法区分“新的一局”和“误调用”，因此用集成判据（`run_id` 变化）覆盖，而不是加一个 `run_id` 参数。
- `Exhausted` 是终态而不是可恢复态：本局内不提供“补时”，下一次 `initialize()` 才是唯一出口。餐局流程在观察到耗尽后同一帧进入 `Ending`，因此 `Exhausted` 通常只存在一帧，但必须可观测、可断言。
- **读数的有效窗口**：`turns_remaining` / `turns_used` 只在 `Active` 与 `Exhausted` 内有定义。`Uninitialized` 下读取返回 0 且 `initialized = false`；`run_ended` 之后数值冻结但不再属于“当前局”的读数。局内 HUD 只在 `run_state ∈ {Active, Ending}` 时读取。

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 餐局流程 | 上游（调用者） | `initialize(turn_budget) → turns_total \| 结构化错误`；`can_spend_turn() → bool`；`spend_turn() → SPENT \| REJECTED_EXHAUSTED \| NOT_INITIALIZED`；`turns_remaining`、`turns_used`、`turns_total`；`is_exhausted()` | 硬 |
| 结算与入场费 | 下游 | 经 `RunLog` 取得 `turns_total` / `turns_used`；本系统保证 `turns_used = 吃菜数 + 付费跳波数` 且 `≤ turns_total` | 硬 |
| 局内 HUD | 下游 | 从本局只读快照读 `turns_remaining` / `turns_total`；同一帧更新；不得自行推算耗尽 | 硬 |
| 菜单波次 | 间接 | 从餐局流程收到 `turns_total` 作为参数；付费跳波的 1 回合由餐局流程调用本系统扣除，菜单波次不直接调用 | 硬（经餐局流程） |
| 菜品数值 | 下游（设计期，软） | `T` 作为 `opt_value(M, C, T)` 与 `calibration_ratio` 的审计参数；改档口预算必须重跑这两项 | 软 |
| 数据配置 | 数据所有者，非运行时依赖 | `StationData.turn_budget` 字段由它定义与校验；本系统只接收已解析的整数 | 软（间接） |
| 局外升级 | 无直接交互 | MVP 无时间类升级；`upgrade_snapshot` 不含时间字段 | — |
| 存档 | 无直接交互 | MVP 无中途存档；回合不持久化 | — |
| 输入 | 无直接交互 | 动作输入由餐局流程读取并路由 | — |
| 音效反馈 | 无直接交互 | 本系统不发信号；回合音效由 `action_committed` / `run_ended` 派生 | — |

**接口所有权**：本系统拥有计数器的值与不变量；调用方拥有“为什么扣”和“什么时候扣”。这个分界让回合规则可以脱离动作语义单独测试：给一组 `initialize` / `spend_turn` 调用，就能验证全部不变量，不需要启动一局。

## Formulas

本系统定义四条数值规则：预算解析、单次消耗、已用回合、耗尽判据。其中**唯一可调的是档口的 `turn_budget`**，它存在数据配置里，由平衡调整；`cost = 1` 与 `turns_used = turns_total − turns_remaining` 是结构性不变量，不是平衡旋钮——改动它们等于改变“回合”这个词的含义，而不是调数值。因此注册表只需要一条跨系统事实：`station_canteen` 的回合预算值（`turn_budget = 10`）；四条规则本身不入注册表。

### 回合预算 `turn_budget`

The `turn_budget` formula is defined as:

`turns_total = turn_budget`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `turn_budget` | int | 1–INT32_MAX | data file | 档口字段 `StationData.turn_budget`，本局允许花时间的动作次数 |
| `turns_total` | int | 1–INT32_MAX | calculated | 本局的总回合数，初始化后不再改变 |

**Output Range:** 正常游戏为 1–INT32_MAX；`< 1` 不进入游戏，`initialize()` 返回 `INVALID_TURN_BUDGET`，不钳制、不默认成 10。
**Example:** `station_canteen` 的 `turn_budget = 10` → `turns_total = 10`。当前 12 道菜池、初始容量 `C0 = 20` 下，最优餐恰好 10 道菜、占满 20 格；回合与容量在同一点咬合。

**可调带（设计期，不进入运行时）**：当前内容下 `turn_budget = 10` 是两条约束同时承重的点，安全带为 **10 ± 1**。

- 低于 10：回合先于容量耗尽，最优餐吃不满 20 格，`opt_value` 下降；若不调整 `entry_fee`，`calibration_ratio` 会跌破 1.15，玩家会因为尝试而亏钱。
- 高于 10：容量先于回合耗尽，回合不再影响任何选择，支柱 3 的“还剩几个格”退化为装饰；此时的 `opt_value` 由容量决定，加回合不增加可达分数。

因此**修改 `turn_budget` 必须同时重跑 `opt_value` 与 `calibration_ratio`**（`dish-values.md`），必要时按结算的校准带调整 `entry_fee`；单改预算会让入场费与内容脱钩。

### 单次消耗 `spend_turn`

The `spend_turn` formula is defined as:

`turns_remaining' = turns_remaining − 1`，当且仅当 `initialized ∧ turns_remaining ≥ 1`；否则保持原值并返回拒绝

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `turns_remaining` | int | 0–`turns_total` | calculated | 当前剩余回合；初始化时为 `turns_total` |
| `cost` | int | `{1}` | constant | 每次成功调用的固定代价；没有其他取值 |
| `spend_turn` | enum | `{SPENT, REJECTED_EXHAUSTED, NOT_INITIALIZED}` | calculated | 提交结果 |

**Output Range:** 成功后 `turns_remaining ∈ [0, turns_total − 1]`；拒绝时不变。任何路径都不会出现负值或超过 `turns_total`。
**Example:** `turns_total = 10`，已吃 4 道、付费跳过 1 次 → 第 5 次成功消耗后 `turns_remaining = 5`；第 11 次调用返回 `REJECTED_EXHAUSTED`，`turns_remaining` 仍为 0。

### 已用回合 `turns_used`

The `turns_used` formula is defined as:

`turns_used = turns_total − turns_remaining`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `turns_total` | int | 1–INT32_MAX | calculated | 本局总回合数 |
| `turns_remaining` | int | 0–`turns_total` | calculated | 当前剩余回合 |
| `turns_used` | int | 0–`turns_total` | calculated | 已消耗回合；结算收据与 `RunLog` 的字段 |

**Output Range:** `[0, turns_total]`。恒等式 `turns_used = 吃菜数 + 付费跳波数`：吃一道菜与跳一次非最后一波各调用一次 `spend_turn()`；自动推进与最后一波「结束取餐」代价为 0，不计入。
**Example:** 吃了 8 道、付费跳过 2 次 → `turns_used = 10`、`turns_remaining = 0` → `is_exhausted() = true`。吃 6 道、跳 1 次、最后一波结束取餐 → `turns_used = 7`、`turns_remaining = 3`，本局以 `NO_EATABLE_DISH` 结束。

> 这条恒等式是结算不变式 4（`n ≤ turns_used ≤ turns_total`，`n` = 吃菜行数）的左边来源：跳过让 `turns_used` 严格大于 `n`，所以结算只能断言下界，不能断言相等。

### 耗尽判据 `is_exhausted`

The `is_exhausted` formula is defined as:

`is_exhausted = initialized ∧ (turns_remaining = 0)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `initialized` | bool | `{true, false}` | calculated | 是否已成功初始化过一局 |
| `turns_remaining` | int | 0–`turns_total` | calculated | 当前剩余回合 |
| `is_exhausted` | bool | `{true, false}` | calculated | `TIME_EXHAUSTED` 的判据；本系统只报告，不据此结束本局 |

**Output Range:** `{true, false}`。`initialized = false` 时恒为 `false`，即使 `turns_remaining = 0`——否则刚启动的系统会被误判成“已耗尽”。
**Example:** 开局 `turns_total = 10` → `false`；第 10 次成功消耗后 → `true`。本局以 `NO_EATABLE_DISH` 结束时剩余 3 格 → 仍为 `false`；下一局 `initialize()` 后重置。

## Edge Cases

- **If `turn_budget < 1`**：`initialize()` 返回 `INVALID_TURN_BUDGET`，保持 `Uninitialized`，不钳到 1、不建立 0 回合的局。餐局流程把它映射为 `init_failed`：不广播 `run_started`、不产生 `RunLog` / `RunResult`。餐局流程的“初始化后 `turns_total = 0` → 立即 `TIME_EXHAUSTED`”只是防御分支，正常数据走不到。
- **If `turn_budget` 缺失或不是整数**：数据配置应在加载期拒绝整个数据集；若仍到达 `initialize()`，本系统断言类型并返回 `INVALID_TURN_BUDGET`。**不存在“缺省 10”的静默兜底**——兜底会让漏配字段的档口看起来正常，却带着错误的节奏上线。
- **If `spend_turn()` 在 `turns_remaining = 0` 时被调用**：返回 `REJECTED_EXHAUSTED`，状态完全不变。正常流程不可达：餐局流程在提交前用 `can_spend_turn()` 校验，并在上一次动作后已经结束本局。拒绝而不是钳制，是为了让调用方的时序错误暴露成测试失败，而不是被悄悄吸收。
- **If `spend_turn()` 在 `Uninitialized` 时被调用**：返回 `NOT_INITIALIZED`，不修改状态。没有“先按 0 记”这种兜底。
- **If 一次动作调用了两次 `spend_turn()`**：扣除两格，本系统无法识别——它没有动作 id。这是调用方不变量（每次提交恰好一次），由餐局流程的判据与本 GDD 的集成判据共同覆盖；不要给本系统加去重逻辑，那会掩盖真正的重复提交 bug。
- **If 局中修改了 `StationData.turn_budget`**（热重载、调试或未来的升级）：当前局不变，`turns_total` 与 `turns_remaining` 保持开局值；下一局 `initialize()` 才使用新值。局中预算变动会让 HUD 已显示的数字变成谎话。
- **If 局中再次调用 `initialize()`**：视为新的一局，完整替换所有值（新 `turns_total`、剩余满格、已用归零）。餐局流程只在开局调用；局中重入属于生命周期 bug，由 `run_id` 的集成判据发现，本系统不额外加锁。
- **If 暂停、失焦、最小化或结算动画期间**：不调用 `spend_turn()`，因此不消耗任何回合。本系统没有内部计时器，不存在“后台偷偷走表”。
- **If 本局以 `NO_EATABLE_DISH` 结束且仍有剩余回合**：数值冻结在结束时刻，`is_exhausted()` 为 `false`；结算按 `turns_total` / `turns_used` 如实记录，不收走剩余回合、不折算成分数。
- **If 本局以 `TIME_EXHAUSTED` 结束**：`turns_remaining = 0`、`turns_used = turns_total`，`is_exhausted()` 保持 `true` 直到下一局 `initialize()`。
- **If 最后一格花在付费跳波上**：餐局流程先确认波次可推进、回合可用，再提交；提交后 `turns_remaining = 0`，本局同一帧以 `TIME_EXHAUSTED` 结束，新波次只作为最终画面出现，玩家无法从中取菜。本系统允许这次消耗（它不判断“值不值”）；是否在剩余 1 格时禁止付费跳波由餐局流程 / 菜单波次决定，见 Open Question 1。
- **If 同一帧同时满足回合耗尽与无可吃菜**：本系统只报告 `is_exhausted() = true`，不保存原因；餐局流程按固定优先级取 `TIME_EXHAUSTED`。本系统不参与排序。
- **If 观测到 `turns_used > turns_total` 或 `turns_remaining < 0`**：内部状态已损坏，返回结构化错误并停止消耗，**不钳回合法区间**。钳制会让越界变成一张看起来正常的收据。
- **If `turn_budget` 远大于菜池规模**（例如 1000 回合、12 道菜）：数据合法，游戏照常开始；但回合永远不是约束，属于可玩性缺陷而非错误。按 `dish-values.md` 的 `opt_value` 审计处理：当最优解的道数少于 `turns_total` 时，回合不再承重，应同时调低预算或调高容量，而不是让本系统拒绝数据。
- **If `turn_budget = 1`**：合法。玩家恰好能做一次花时间的动作，然后 `TIME_EXHAUSTED`。这是一局退化但完整的结算，不是错误；内容评审应把它标为异常档口。
- **If 在 `Uninitialized` 下读取**：`turns_total` / `turns_remaining` / `turns_used` 返回 0，`initialized = false`，`is_exhausted() = false`。局内 HUD 不得在菜单或结算页显示这些读数。
- **If 整数溢出**：不会发生。`turn_budget` 来自 int32 数据字段，全部运算是在 int64 上的减法，`turns_used` 与 `turns_remaining` 都不会越过 `[0, turns_total]`。

## Dependencies

**上游依赖（本系统被它调用）：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 餐局流程 | 硬 | `initialize(turn_budget) → turns_total \| 结构化错误`；`can_spend_turn()`；`spend_turn()`；`turns_remaining` / `turns_used` / `turns_total`；`is_exhausted()` | GDD 已写（Designed） |

**下游依赖方（它们在运行时依赖本系统；依赖方向上它们依赖本系统）：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 结算与入场费 | 硬 | 经 `RunLog` 取得 `turns_total` / `turns_used`；依赖恒等式 `turns_used = 吃菜数 + 付费跳波数` 与 `turns_used ≤ turns_total` | GDD 已写（Designed） |
| 局内 HUD | 硬 | 只读本局快照中的 `turns_remaining` / `turns_total`；不得自行推算耗尽 | GDD 已写（`design/gdd/in-game-hud.md`） |
| 菜单波次 | 硬（经餐局流程） | 从餐局流程收到 `turns_total`；付费跳波由餐局流程扣 1 回合，菜单波次不直接调用本系统 | GDD 已写（Designed） |
| 菜品数值 | 软（仅设计期） | `T` 作为 `opt_value(M, C, T)` 与 `calibration_ratio` 的审计参数 | GDD 已写（Approved） |

**为什么方向是“本系统依赖餐局流程”，而不是反过来**：索引记录 `时间与回合 → 餐局流程`。运行时的调用方向是餐局流程调用本系统，但设计依赖按**生命周期所有权**记录——本系统需要餐局流程定义“一局从哪一刻开始、哪些动作算提交、什么时候结算”，而餐局流程不需要本系统定义“什么时候算一局”。这与餐局流程 GDD 的 §Dependencies 采用同一条理由，避免形成环。

**不属于依赖的关系：**

- **数据配置**：它拥有 `StationData.turn_budget`，但本系统不查询目录。预算由餐局流程在开局解析后以整数传入。若改成由本系统直接读 `StationData`，索引需要新增 `时间与回合 → 数据配置` 这条边，并让数据配置的下游表回指本系统——当前设计刻意避免这条边，因为本系统只需要一个整数，不需要数据层。
- **局外升级**：MVP 没有时间类升级，`upgrade_snapshot` 不含时间字段。未来的时间升级属于接口扩展（Open Question 3），不是当前依赖。
- **存档**：MVP 无中途存档，回合数不持久化；一局的回合明细只活在 `RunLog` 与 `RunResult` 里。
- **输入**：本系统不读任何输入；动作路由归餐局流程。
- **音效反馈**：本系统不发信号；回合相关音效由 `action_committed` / `run_ended` 派生。
- **胃容量**：两者是同级的每局预算，互不调用。唯一的耦合是设计期校准（`opt_value(M, C, T)` 同时使用 `C` 与 `T`），不构成运行时依赖。

**双向一致核对：**

- 已回指本系统的 GDD：餐局流程（下游表，接口逐字一致）、结算与入场费（上游表，`turns_total` / `turns_used`）、菜品数值（软上游，`T` 作为审计参数）。菜单波次从餐局流程接收 `turns_total`，索引未把本系统列为其上游，与本设计一致。
- **索引第 6 行**：`Status` 应由 `Not Started` 改为 `Designed`，`Design Doc` 应指向 `design/gdd/time-and-turns.md`；由编排会话统一处理。
- **数据配置已定义**：`StationData.turn_budget: int`（≥ 1）已在 `design/gdd/data-config.md` 声明并纳入加载期校验；本 GDD 的规则 3 依赖该字段存在。
- **菜单波次措辞**：其 UI Requirements 写“本系统只发起扣除”，本 GDD 规定唯一调用者是餐局流程；行为一致（发起 ≠ 执行），但文档措辞需要在一致性检查时统一，见 Open Question 6。

## Visual/Audio Requirements

**本系统不绘制任何元素、不播放任何声音。** 它只保证一个可观察的事实：每次 `spend_turn()` 成功后，剩余格数在同一帧变化。回合计数器的视觉、动效与音效分别由局内 HUD 与音效反馈实现；本系统不规定槽位形状、颜色或过渡时长，也不拥有任何音效事件。

## Game Feel

- **减法是即时的**：扣格发生在动作提交的同一帧，不做补间、不做延迟 tick。玩家按下确认到数字变化之间没有任何等待——这是支柱 4 在回合上的体现。
- **压力不是焦虑**：没有倒计时、没有脉动、没有红色闪烁、没有“快一点”的提示音。压力来自“还剩几个格”这个可算的数字，不来自时间流逝的表演（支柱 3）。
- **时间和胃是同一笔预算**：吃一道菜时，回合与容量应当被读成同时减少的两栏；玩家要能感到“这一口既花了时间也占了胃”，而不是两个互不相干的资源。
- **可归因**：结算收据已经列出 `turns_total` / `turns_used`，玩家能说出“我还有 3 格没花掉”或“我为了等那道菜跳过了两波”。本系统不额外产出解释文案，但必须让这两个数字始终真实。

## UI Requirements

- **常驻可见**：`Active` 期间，剩余回合与总回合必须同时可读（例如“6 / 10 格”或等价的槽位行）。不能只在悬停、暂停或结算时出现。
- **同帧更新**：吃菜与付费跳波共用同一个计数器；提交的那一帧数字立即变化，不等待动画结束（与菜单波次的“回合数字当场减一”一致）。
- **不表演倒计时**：不显示秒数、不显示预计剩余时间、不做逐秒递减、不做时间条自动缩短。回合只在动作提交时变化。
- **耗尽不是弹窗**：`turns_remaining = 0` 时，本局由餐局流程进入结束流程；HUD 不得自行弹出“时间到”模态、不得播放假倒计时，只把数字置为 0 并进入只读。
- **无障碍**：剩余回合不能只靠颜色或槽位图形表达，必须同时有可读数字；字号 150% 时不换行错位；对比度 ≥ 4.5:1。

> **📌 UX Flag — 时间与回合**：本系统有 UI 需求。Phase 4 写 epic 之前，先用 `/ux-design` 为局内 HUD 的回合计数出规格；相关 story 引用 `design/ux/hud.md`，不直接引用本 GDD。

## Cross-References

| 系统 | GDD | 关系 |
|---|---|---|
| 餐局流程 | design/gdd/run-flow.md | 上游（调用者）：`initialize` / `can_spend_turn` / `spend_turn` / 读数 / `is_exhausted()`；本 GDD 落实其 §Dependencies 中列出的接口 |
| 结算与入场费 | design/gdd/settlement-and-entry-fee.md | 下游：`RunLog.turns_total` / `turns_used` 的来源；满足其不变式 4 |
| 菜单波次 | design/gdd/menu-waves.md | 间接：`turns_total` 经餐局流程传入；付费跳波由餐局流程扣 1 回合；其 `eats + skips ≤ T` 与本系统的 `turns_used` 恒等式一致 |
| 菜品数值 | design/gdd/dish-values.md | 软（设计期）：`T` 是 `opt_value` / `calibration_ratio` 的审计参数；改预算须重跑校准 |
| 数据配置 | design/gdd/data-config.md | 数据所有者：`StationData.turn_budget` 字段；本系统不在运行时查询它 |
| 局内 HUD | design/gdd/in-game-hud.md | 下游：显示剩余 / 总回合；不得自行推算耗尽 |
| 局外升级 | design/gdd/meta-upgrades.md | 无直接交互：MVP 无时间类升级 |
| 存档 | design/gdd/save-system.md | 无直接交互：MVP 无中途存档，回合不持久化 |
| 输入 | design/gdd/input.md | 无直接交互：动作由餐局流程路由 |
| 音效反馈 | design/gdd/audio-feedback.md | 无直接交互：本系统不发信号 |

**待回指（不由本文件修改）**：其余回指均已完成。唯一剩余项：`menu-waves.md` §UI Requirements 的「本系统只发起扣除」建议改为「本系统发出跳波意图，由餐局流程调用时间与回合扣除」。

## Acceptance Criteria

> **Formulas 覆盖**：`turn_budget` 由第 1–3、20、21 条；`spend_turn` 由第 4–9 条；`turns_used` 由第 7、16 条；`is_exhausted` 由第 8、9、10、15、17 条。其余判据来自 Detailed Design。

> **集成判据**：第 2、8、14、16、17、18、19、20 条需要餐局流程、菜单波次或结算的测试替身；只测本系统的判据可用 `initialize` / `spend_turn` 直接驱动，不需要启动一局。

> **术语**：`initialized` = 已成功初始化过一局；`Active` / `Exhausted` / `Uninitialized` = 本系统的状态；`turns_used = 吃菜数 + 付费跳波数` 是恒等式，不含自动推进与最后一波“结束取餐”。

1. **GIVEN** `turn_budget = 10`，**WHEN** `initialize(10)`，**THEN** 返回 `turns_total = 10`，`turns_remaining = 10`、`turns_used = 0`、状态 `Active`、`initialized = true`、`is_exhausted() = false`。
2. **GIVEN** `turn_budget = 0` 与 `turn_budget = -1`（两个夹具），**WHEN** `initialize()`，**THEN** 返回 `INVALID_TURN_BUDGET`，状态保持 `Uninitialized`，读数仍为 0/0/0；**WHEN** 经餐局流程开局，**THEN** `init_failed = true`，不广播 `run_started`、不产生 `RunLog` / `RunResult`。
3. **GIVEN** `turn_budget` 缺失或不是整数，**WHEN** 初始化，**THEN** 返回 `INVALID_TURN_BUDGET`，不套用默认值 10、不开始一局。
4. **GIVEN** `Active` 且 `turns_remaining = 3`，**WHEN** `spend_turn()`，**THEN** 返回 `SPENT`，`turns_remaining = 2`、`turns_used = turns_total − 2`。
5. **GIVEN** `turns_remaining = 0`，**WHEN** `spend_turn()`，**THEN** 返回 `REJECTED_EXHAUSTED`，`turns_remaining` 仍为 0、`turns_used` 仍为 `turns_total`，不出现负值。
6. **GIVEN** `Uninitialized`，**WHEN** `spend_turn()`，**THEN** 返回 `NOT_INITIALIZED`，三个读数仍为 0。
7. **GIVEN** `turns_total = 10`，已提交 4 次吃菜与 1 次付费跳波，**WHEN** 读取，**THEN** `turns_used = 5`、`turns_remaining = 5`，且 `turns_used = 吃菜数 + 付费跳波数`。
8. **GIVEN** `turns_total = 10`，已连续提交 10 次吃菜，**WHEN** 第 10 次提交完成，**THEN** `turns_remaining = 0`、`is_exhausted() = true`，餐局流程同一帧判定 `TIME_EXHAUSTED`；第 11 次动作被 `can_spend_turn() = false` 拒绝，`spend_turn()` 未被调用。
9. **GIVEN** `turn_budget = 1`，**WHEN** 提交一次吃菜，**THEN** `turns_remaining = 0`、`is_exhausted() = true`、本局结束；下一次动作无法通过 `can_spend_turn()`。
10. **GIVEN** `Uninitialized`（读数均为 0），**WHEN** 调用 `is_exhausted()`，**THEN** 返回 `false`。
11. **GIVEN** `Active` 且 `turns_remaining = 6`，**WHEN** 经过 60 秒无输入、连续 3 帧空闲、一次暂停/恢复、一次窗口失焦，**THEN** 三个读数与 `is_exhausted()` 全部不变；测试不依赖真实时钟，使用假时钟或帧步进。
12. **GIVEN** `Active` 且 `turns_total = 10`，**WHEN** `StationData.turn_budget` 在局中被改为 14，**THEN** 当前局仍为 10；**WHEN** 下一局 `initialize()`，**THEN** `turns_total = 14`。
13. **GIVEN** 一局已进行到 `turns_remaining = 4`，**WHEN** 再次 `initialize(8)`，**THEN** `turns_total = 8`、`turns_remaining = 8`、`turns_used = 0`，上一局的任何值都不延续。
14. **GIVEN** 当前波被吃空触发自动推进，或玩家在最后一波选择「结束取餐」，**WHEN** 动作完成，**THEN** `spend_turn()` 未被调用，`turns_used` 不变。
15. **GIVEN** 本局以 `NO_EATABLE_DISH` 结束且 `turns_remaining = 3`，**WHEN** `run_ended` 之后读取，**THEN** 数值冻结为 3，`is_exhausted() = false`；`RunLog.turns_total` / `turns_used` 与冻结值一致。
16. **GIVEN** 任意完成的一局，吃菜行数为 `n`、付费跳波次数为 `s`，**WHEN** 结算校验，**THEN** `n ≤ turns_used ≤ turns_total` 且 `turns_used = n + s`；结算能接收该 `RunLog` 而不返回 `TURN_INCONSISTENT`。
17. **GIVEN** 一次成功动作后同时满足 `turns_remaining = 0` 与“无可吃菜”，**WHEN** 餐局流程判定，**THEN** `end_reason = TIME_EXHAUSTED`；本系统只返回 `is_exhausted() = true`，不保存也不排序原因。
18. **GIVEN** `turns_remaining = 1`，**WHEN** 餐局流程允许并提交一次付费跳波，**THEN** `turns_remaining = 0`、本局同一帧以 `TIME_EXHAUSTED` 结束；新波次只作为最终画面出现，不能从中取菜（当前行为；是否禁止见 Open Question 1）。
19. **GIVEN** 一次吃菜提交后 `action_committed` 广播，**WHEN** 局内 HUD 读取本局快照，**THEN** 快照中的 `turns_remaining` 与本系统的读数在同一帧相等，且比提交前恰好少 1。
20. **GIVEN** 当前 12 道菜池、`C = 20`、`turn_budget = 10` 的 MVP 内容，**WHEN** 玩家完成最优餐（10 道菜、20 格），**THEN** 第 10 次吃菜后 `turns_used = 10`、`turns_remaining = 0`；**GIVEN** 玩家付费跳过 2 波，**THEN** 最多只能吃 8 道菜，`turns_used` 仍为 10。
21. **GIVEN** `turn_budget = 1000` 与 12 道菜的菜池，**WHEN** 开局，**THEN** 初始化成功（数据合法，不钳制）；设计期审计应报告“回合不承重”，而不是让本系统拒绝该档口。
22. **GIVEN** 故障注入：同一动作调用两次 `spend_turn()`，**WHEN** 两次都被处理，**THEN** 扣除两格；本系统不去重、不猜测调用意图；正常路径的集成判据（第 7、8 条）断言每次提交只扣一次。

## Open Questions

1. **最后一格花在付费跳波上是否应被禁止？** 当前本系统允许（它不判断“值不值”），结果是玩家花掉最后一回合看到一波自己吃不到的新菜，本局立即结束。建议由餐局流程 / 菜单波次在 `turns_remaining = 1` 时禁用“下一波”按钮，而不是在本系统加特例。Owner：餐局流程 + 菜单波次。目标：局内 HUD UX 规格之前（2026-11-17）。
2. **`turn_budget` 的归属与命名？** 本 GDD 假定它是 `StationData` 的必填字段（整数 ≥ 1），由数据配置加载期校验；MVP 值 `station_canteen = 10`。若数据配置更愿意把它放进每档口的常量表或单独的时间配置，需要同步修改本 GDD 的变量 Source 与校准说明。Owner：数据配置 + 编排会话。目标：实现前。
3. **是否要有时间类升级？** MVP 明确没有（`upgrade_snapshot` 不含时间字段）。若垂直切片的试玩证明“回合永远不够”是主要挫败来源，可考虑 `upgrade_snapshot.turn_bonus`；加入时必须开局冻结，并重跑 `opt_value` / `calibration_ratio`。Owner：局外升级 + 用户。目标：Vertical Slice 试玩后。
4. **HUD 用槽位还是数字？** 10 格以内槽位可读，更多回合需要数字。当前 MVP 只有 10，两者皆可；若未来档口预算超过 20，槽位行会失去可读性。Owner：`/ux-design`（`design/ux/hud.md`）。目标：局内 HUD epic 之前。
5. **“回合不承重”的审计放在哪里？** 本 GDD 只在设计期说明，未定义运行时校验。`dish-values.md` 已经计算 `opt_value` 与其见证解，最自然的归属是它的平衡审查（当最优解道数 < `turns_total` 时告警），而不是本系统。Owner：菜品数值 + 编排会话。目标：一致性检查。
6. **菜单波次的措辞统一**：其 UI Requirements 写“本系统只发起扣除”，本 GDD 规定唯一调用者是餐局流程。语义兼容（发起意图 vs 执行扣减），但两份文档应使用同一句话，避免实现时出现第二个调用者。Owner：菜单波次。目标：一致性检查。
