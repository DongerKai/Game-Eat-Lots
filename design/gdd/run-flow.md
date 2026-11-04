# 餐局流程（Run Flow）

> **Status**: In Design
> **Author**: user + Claude（autonomous 模式；systems-designer 与 qa-lead 按 Lean 模式仅复核 Formulas 与 Acceptance Criteria）
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 没有白吃的菜 · 数字看得见地涨

## Summary

餐局流程是一局自助餐的骨架状态机：从菜单带着 `RunConfig` 进入 `Run`，到初始化各子系统、接受吃菜与跳波动作、判定结束条件、组装 `RunLog` 交给结算，再把 `RunResult` 交回界面流程，最后丢弃本局状态等待重开。它不计算价格、饱腹、回合或腻度，只决定这些系统在一局里的生命周期与交接顺序。它服务支柱 2「没有白吃的菜」与支柱 4「数字看得见地涨」：一局必须完整地开始、结束、结账，且中途放弃不产生任何收益。

> **Quick reference** — Layer: `Core` · Priority: `MVP` · Key deps: `场景与流程管理, 数据配置, 输入`（协调的下游：`时间与回合, 胃容量, 菜单波次, 腻度与口味, 结算与入场费`）

## Overview

餐局流程是《回本》里“一局”的骨架。它定义一局从哪一刻开始、按什么顺序初始化各子系统、玩家的一次输入如何变成一次已提交的吃菜或跳波、什么时候判定本局结束、由谁把这一局的事实拼成一份完整的日志交给结算，以及结算结果如何回到菜单并在重开时彻底清零。

它不拥有任何一个数字。回合预算属于时间与回合，容量属于胃容量，可见菜与波次属于菜单波次，重复惩罚属于腻度与口味，价格与净利属于策略层与结算；餐局流程只拥有**顺序与边界**：谁先初始化、一次动作里谁的判定先于谁、结束时哪些输入必须被拒绝、哪些事实必须原样记录而不能重算。它把“一局”变成一个可重复、可测试、可复盘的对象。

没有它，每个子系统都要自己判断“现在是不是在局内”“这一口该不该扣回合”“结算有没有发生过”，重开与放弃的清理就会散落在各处。它主要服务支柱 4「数字看得见地涨」——玩家按下确认到数字变化之间没有任何流程等待；也服务支柱 2「没有白吃的菜」——放弃一局不产生任何收益，且不允许通过半途退出绕过结算。

它与场景与流程管理是分工而非重复：后者拥有**应用级**顶层状态（`Boot / Error / Menu / Run / Paused / Quit`）与切换合法性，本系统只拥有 `Run` 内部的**一局级**生命周期，并通过 `request_transition` 发起进入结算后的返回。暂停由场景与流程管理切换顶层状态并通知本系统冻结，本系统不自行判定暂停。

## Detailed Design

### Core Rules

1. **职责边界**：本系统拥有**一局的生命周期、动作仲裁、结束判定、`RunLog` 组装与交接顺序**。它不拥有菜品价格与饱腹（数据配置 / 菜品数值）、回合预算（时间与回合）、容量（胃容量）、波次划分（菜单波次）、腻度惩罚（腻度与口味）、净利计算（结算与入场费）、钱包与纪录（局外升级 / 纪录与里程碑）或任何界面绘制（界面流程 / 局内 HUD）。
2. **一局恰好一次**：每一局分配唯一的 `run_id`，经历恰好一次开始、一次结束、至多一次结算、一次交接。同一 `run_id` 不得出现第二份 `RunLog` 或 `RunResult`；重复结算由结算系统拒绝，本系统不依赖该拒绝来保证正确性。
3. **进入即初始化**：顶层 `Menu → Run` 生效时，本系统取得 `RunConfig`，校验后在同一帧内完成：分配 `run_id`、确定 `run_seed`、依次初始化时间与回合、胃容量、菜单波次，然后广播 `run_started`（腻度与口味以此重置计数）并进入 `Active`。初始化不消耗回合与容量。
4. **初始化失败不是一局**：`RunConfig` 无法解析、`station_id` 不存在、升级快照字段非法或任一子系统初始化失败时，本系统立即请求 `Run → Menu`，携带 `init_failed = true`；不产生 `RunLog`、不产生 `RunResult`、不广播 `run_started`、不写纪录。**若该返回请求因同一帧内 `Menu → Run` 已先生效而被拒绝**（场景与流程管理按帧初状态校验），本系统留在 `Failed` 并在下一帧重发，不重复初始化。不得停在半初始化的 `Run`。
5. **一次输入只提交一个动作**：`Active` 中每个输入事件至多提交一个动作，动作集只有三种——吃一道菜（`ui_accept` 作用于当前焦点菜品）、主动跳波（`game_skip_wave`，非最后一波）、最后一波「结束取餐」（同一动作，0 回合）。焦点菜品由局内 HUD 按当前键盘/手柄焦点或鼠标悬停解析后随动作传入；本系统只校验它仍在当前可见集，不自行猜测焦点。动作入口为 `submit_action(action, dish_id) → COMMITTED | REJECTED(code)`，`code` 至少覆盖 `STALE_FOCUS` / `CAPACITY` / `NO_TURNS` / `FROZEN`；动作先完整校验再原子提交；被拒绝的尝试不改变任何状态，HUD 直接渲染该码对应文案。
6. **吃菜的提交顺序固定**：校验顺序为“菜仍在当前可见集 → 向腻度取得 `effective_satiety(d)` → 胃容量 `can_fit(effective_satiety)` → 回合仍可用”；通过后原子提交：按吃下顺序追加 `EatenEntry(dish_id, extra_satiety, effective_satiety)`、更新腻度计数、调用菜单波次的 `consume(dish_id)`（返回 `ADVANCED` 或 `NO_EATABLE`）把该菜从当前波移除、消耗 1 回合，然后执行一次结束判定。任何一步之前失败都整体回滚，不扣回合、不写日志。
7. **跳波由菜单波次拥有，回合由时间与回合拥有**：非最后一波的 `game_skip_wave` 丢弃当前波剩余菜、推进一波、消耗 1 回合；最后一波该动作为「结束取餐」，不消耗回合、丢弃剩余菜并向本系统发出“无可吃菜”信号。本系统只负责路由输入、确认代价、执行结束判定，不自行计算跳过代价。跳波校验顺序固定为“波次是否允许推进 → 回合是否可用 → 提交”。
8. **只有两个结束原因**：`TIME_EXHAUSTED` 与 `NO_EATABLE_DISH`。**没有“吃撑”结束原因**：胃容量只决定一道菜能不能吃，不直接结束本局；玩家可能仍有回合却暂时放不下任何可见菜，此时可以付费跳波或等下一波。菜单波次的 `no_eatable` 信号只覆盖“最后一波被吃完 / 玩家结束取餐 / 本局已无剩余菜”，**不覆盖“可见菜都放不进剩余容量”**；后者不结束本局。MVP 不提供主动提前结账（回答结算与入场费 Open Question 2）；最后一波「结束取餐」记为 `NO_EATABLE_DISH`，不新增第三枚举。
9. **结束判定时机固定**：每次成功提交动作之后判定一次；初始化完成后判定一次。被拒绝的动作、暂停中的帧、以及纯展示帧都不判定。判定命中后同一帧进入 `Ending`，此后不再接受任何动作。
10. **结束原因优先级固定**：若同一帧同时满足回合耗尽与无可吃菜，取 `TIME_EXHAUSTED`。顺序写死在判定函数里，不依赖事件到达顺序，也不随机。
11. **结束即冻结**：进入 `Ending` 后，回合消耗、容量变化、波次推进、菜品选择全部停止；局内画面保持最终状态只读显示。剩余工作只有组装 `RunLog`、调用结算、交接结果。
12. **`RunLog` 由本系统组装，值来自动作发生时刻**：字段包含 `run_id`、`station_id`、`entry_fee`、`end_reason`、`eaten[]`（按提交顺序，每条含 `dish_id`、`extra_satiety`、`effective_satiety`）、`capacity_total`、`capacity_used`、`turns_total`、`turns_used`、`upgrade_snapshot`。`extra_satiety` 与 `effective_satiety` 是吃下那一刻由腻度系统给出的快照，**结算时不得按菜名重放**。
13. **结算恰好一次**：在 `Ending → Settling` 中调用结算的 `settle(RunLog)` 一次。成功则持有 `RunResult` 进入 `Handoff`；返回结构化错误则进入 `Faulted`，不重试结算、不生成部分结果、不把错误当 0 分。顶层处于 `Paused` 时停在 `Ending` 等待，恢复后才调用。
14. **交接与重试**：在 `Handoff` 中请求 `Run → Menu`，携带同一份 `RunResult`。若请求被拒绝（例如同一帧内暂停先到），保留该 `RunResult`，在回到 `Run` 顶层状态后重发；**不得重新结算、不得重建结果、不得重复交付**。暂停期间等待，不请求。
15. **暂停是冻结，不是一个局内状态**：本系统只持有 `frozen` 标志，由场景与流程管理的冻结/恢复通知驱动。冻结期间不提交动作、不判定结束、不调用结算、不请求交接；恢复后从暂停前的精确状态继续。暂停与恢复不消耗回合、不改变 `run_id` 或日志。
16. **放弃与退出不是结算**：顶层状态在不由本系统发起交接的情况下离开 `Run` / `Paused` 进入 `Menu` 或 `Quit`（暂停菜单放弃、直接退出、窗口关闭确认），本系统丢弃本局全部状态并广播 `run_discarded`；不产生 `RunResult`、不结算、不写纪录。下一局从全新的 `RunConfig` 开始。
17. **重开是全新一局**：结算屏“再来一局”是一次新的 `Menu → Run`，分配新的 `run_id`、新的 `run_seed`（`RunConfig.seed` 为空时），所有子系统重置；上一局的任何字段都不延续。“重开”不是独立生命周期事件——每一次开始都是重置。
18. **失败不是卡死**：容量不一致、结算错误、日志含未知菜等不变式失败时进入 `Faulted`，以 ERROR 记录 `run_id` 与违反的字段，并通过当前唯一可用的无结果返回（`Run → Menu`，`init_failed = true` 附带 `fault` 明细）离开 `Run`。若返回请求因同帧 `Menu → Run` 已生效而被拒绝，留在 `Faulted` 并在下一帧重发。绝不把玩家留在 `Run` 里，也绝不伪造 `RunResult`。该返回通道复用 `init_failed` 是暂定方案，见 Open Questions。
19. **确定性**：给定同一 `RunConfig`（含显式种子）与同一动作序列，`run_seed`、`end_reason` 与 `RunLog` 必须逐字段相同——**`run_id` 除外**，它按局递增。本系统不使用系统时钟或未播种随机数；种子在 `run_started` 中记录，供复盘与复现。
20. **交接尽量单帧、不做等待态**：动作提交、结束判定与结算都在触发帧内完成；返回请求在切换合法的第一帧发出，最迟下一帧——因同帧 `Menu → Run` 已生效而被拒绝时顺延一帧。不插入加载、过场或“结算中”状态。淡入淡出属于表现层，且不得推迟状态生效（场景与流程管理规则 12）。
21. **首局钩子**：`RunConfig` 可以携带 `is_first_run` 与可选的 `pool_override`，本系统把它们原样转交首局引导与菜单波次，除此之外对首局不做任何特殊处理。首局引导可以压入 Blocked 输入上下文，但不得改变本局状态、结束条件或生命周期。
22. **数据前提**：本系统只在数据配置 `Ready` 时开始一局（由 `Boot` 保证），并只校验 `station_id` 可解析；解析成功后读取 `StationData.turn_budget`，作为 `initialize(turn_budget)` 的入参传给时间与回合。`pool_override` 的子集校验只读取 `StationData.dish_pool` 的 id 集合。它不读取任何菜品价格或饱腹；所有数字来自各子系统。

### RunConfig 与 RunResult 的归属（回答场景与流程管理 Open Question）

场景与流程管理只转交 payload，字段 schema 需要由拥有者定义。本 GDD 明确：

- **`RunConfig` 由界面流程组装**，局外升级提供本局生效的升级快照；本系统只校验与消费，不组装、不读存档。
- **`RunResult` 由结算与入场费拥有**，本系统原样转交，不增删字段、不解释、不缓存到下一帧之外。

`RunConfig` 的最小字段：

| 字段 | 类型 | 必需 | 来源 | 本系统校验 |
|---|---|---|---|---|
| `station_id` | `StringName` | 是 | 界面流程 | `get_station(id)` 必须存在 |
| `entry_fee_override` | `int \| null` | 否（默认 `null`） | 界面流程 / 首局引导 | `null` = 用 `StationData.entry_fee`；非 `null` 时必须 ≥ 0 |
| `upgrade_snapshot` | `Dictionary` | 是 | 局外升级（经界面流程组装） | 至少含 `capacity_bonus: int ≥ 0`、`cloying_resistance: float ≥ 0`、`menu_vision: int ≥ 0`（已解析值，不是等级） |
| `seed` | `int64 \| null` | 否 | 界面流程 / 首局引导 / 在线挑战 | 任意 int64；`0` 是合法种子，不当作缺省 |
| `is_first_run` | `bool` | 否（默认 `false`） | 界面流程 / 存档 | — |
| `pool_override` | `StringName[] \| null` | 否（暂定） | 首局引导 | 非空，且必须是该档口 `dish_pool` 的子集 |

`upgrade_snapshot` 是 `Dictionary`，由局外升级定义并填充，结算只转交、不解释。它携带的是**已解析的本局修正值**，不是等级：`capacity_bonus` 加到容量上，`cloying_resistance` 交给腻度，`menu_vision` 交给菜单波次（作为总预览额度，不是等级）。等级到数值的换算属于局外升级，本系统不重算。结算 GDD 用“等级快照”概括该字段；两者以“结算不解释、只转交”为前提兼容，若纪录需要按等级解释，快照应同时保留等级与解析值（见 Open Questions）。

`is_first_run` 与 `pool_override` 相互独立：`pool_override` 也可用于挑战或调试，不要求 `is_first_run = true`。

### States and Transitions

本系统的状态只在顶层 `Run` 内存在；`Paused` 是覆盖其上的冻结标志，不是局内状态。

| 状态 | 进入条件 | 允许的动作 | 离开 |
|---|---|---|---|
| `Idle` | 应用启动、上一局已交接或已丢弃 | 无（只在 `Menu`） | `begin(RunConfig)` → `Initializing` |
| `Initializing` | 收到 `RunConfig` | 仅校验与初始化 | 成功 → `Active`；配置失败 → `Failed`；`run_id` 溢出 → `Faulted` |
| `Active` | 初始化成功且广播 `run_started` | 吃菜、跳波、结束取餐 | 结束判定命中 → `Ending`；顶层离开 → `Idle`（丢弃） |
| `Ending` | 结束判定命中 | 只组装 `RunLog` | 组装成功 → `Settling`；不变式失败 → `Faulted` |
| `Settling` | `RunLog` 就绪且顶层为 `Run` | 只调用 `settle` 一次 | 成功 → `Handoff`；错误 → `Faulted`；顶层暂停 → 等待 |
| `Handoff` | 持有 `RunResult` | 重发 `Run → Menu` | 接受 → `Idle`；拒绝 → 留在 `Handoff` 等待；暂停 → 等待 |
| `Failed` | 初始化失败 | 请求 `Run → Menu (init_failed)` | 接受 → `Idle` |
| `Faulted` | 不变式或结算失败 | 记录 ERROR、请求 `Run → Menu (init_failed + fault)` | 接受 → `Idle` |

`frozen` 标志与状态正交：`Active`、`Ending`、`Settling`、`Handoff` 在 `frozen = true` 时都不得推进。冻结不清空任何字段，恢复后原状态继续。

| 从 | 触发 | 到 | 副作用 |
|---|---|---|---|
| `Idle` | `Menu → Run` 生效，`RunConfig` 合法 | `Initializing` | 分配 `run_id`、确定 `run_seed` |
| `Initializing` | 时间与回合、胃容量、菜单波次初始化成功 | `Active` | 广播 `run_started`（腻度计数重置）；执行一次初始结束判定 |
| `Initializing` | 任一步失败 | `Failed` | 丢弃部分初始化；请求 `Run → Menu, init_failed` |
| `Active` | 成功吃菜 / 跳波 / 结束取餐后判定命中 | `Ending` | 广播 `run_ended(end_reason)`；冻结动作 |
| `Ending` | `RunLog` 组装成功且顶层为 `Run` | `Settling` | — |
| `Ending` | `RunLog` 就绪但顶层暂停 | `Ending` | 等待恢复；不调用 `settle` |
| `Ending` | 不变式失败 | `Faulted` | 记录违反字段 |
| `Settling` | `settle` 成功 | `Handoff` | 广播 `run_settled(RunResult)` |
| `Settling` | `settle` 返回结构化错误 | `Faulted` | 记录错误码 |
| `Handoff` | `request_transition(Run → Menu, RunResult)` 接受 | `Idle` | 清空本局全部状态；交给界面流程 |
| `Handoff` | 请求被拒绝 | `Handoff` | 保留结果，下一可用帧重发 |
| `任何局内状态` | 顶层离开 `Run` / `Paused` 且非本系统交接 | `Idle` | 广播 `run_discarded`；不结算、不写纪录 |
| `Initializing` | `run_id` 分配溢出 | `Faulted` | 记录 `RUN_ID_OVERFLOW`；请求 `Run → Menu (init_failed + fault)` |
| `Failed` | 返回请求被拒绝 | `Failed` | 下一帧重发；不重复初始化 |
| `Faulted` | 返回请求被拒绝 | `Faulted` | 下一帧重发；不重试结算 |

**同帧冲突**：同一帧内动作与暂停同时到达时，先按场景与流程管理的顶层状态判定处理暂停；本系统看到 `frozen` 后不再提交动作。动作已提交并进入 `Ending` 后暂停到达，则结算等待恢复；若玩家在暂停中放弃，按“放弃不是结算”丢弃，已算出的 `RunResult` 不交付。

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 场景与流程管理 | 上游 | `Menu → Run` 时取得 `RunConfig`；接收冻结/恢复通知；调用 `request_transition(Run → Menu, RunResult \| init_failed)`；观察当前顶层状态以识别放弃/退出 | 硬 |
| 数据配置 | 上游 | 前提 `Ready`；`get_station(station_id)` 必须存在，并读取 `turn_budget`；不读取菜品数值 | 硬 |
| 输入 | 上游 | 每次动作读取 `ui_accept`、`game_skip_wave`；**不读取 `ui_pause`**（由场景与流程管理消费） | 硬 |
| 时间与回合 | 协调（下游） | `initialize(turn_budget) → turns_total`、`spend_turn()`、`turns_remaining`、`turns_used`；提供 `TIME_EXHAUSTED` 判据 | 硬 |
| 胃容量 | 协调（下游） | `initialize(upgrade_snapshot) → capacity_total`、`capacity_used`、`can_fit(effective_satiety)`、`consume(e)` | 硬 |
| 菜单波次 | 协调（下游） | `initialize(station_id, seed, turn_budget, menu_vision[, pool_override])`；`consume(dish_id) → {ADVANCED, NO_EATABLE}`；`visible_dishes`；`advance()` 只承载付费跳波；无菜可吃信号 | 硬 |
| 腻度与口味 | 协调（下游） | `run_started` / `run_ended` 生命周期；吃下前返回 `extra_satiety`、`effective_satiety` 供容量判定，吃下时记录同一组值；组计数 | 硬 |
| 结算与入场费 | 协调（下游） | `settle(RunLog) → RunResult \| 结构化错误`；每局恰好一次 | 硬 |
| 局内 HUD | 下游 | 只读本局快照与 `run_*` 信号；渲染 `submit_action` 的 `REJECTED(code)` 文案；不得自行求和或推算结束条件 | 硬 |
| 界面流程 | 下游 | 经场景与流程管理发出 `RunConfig`、接收 `RunResult`；重开时组装新的 `RunConfig` | 硬 |
| 首局引导 | 下游（软） | 读取 `is_first_run`、`pool_override`；可压入 Blocked 上下文，不得改变本局状态 | 软 |
| 音效反馈 | 下游（软） | 订阅 `run_started` / `run_ended` / `run_settled` / `run_discarded` / `run_faulted` | 软 |
| 存档 | 无直接调用 | MVP 无中途存档；局外升级与纪录由下游持久化 | 软（间接） |

**接口所有权**：本系统拥有 `run_id`、`run_seed` 的选择、动作提交顺序、结束判定与 `RunLog` 组装；各子系统拥有自己的数值与判定。下游不得自行开始、结束或结算一局，也不得改写 `run_state`。

**信号（过去式命名）**：`run_started(run_id, station_id, seed)`、`action_committed(action, payload)`、`run_ended(run_id, end_reason)`、`run_settled(run_id, RunResult)`、`run_faulted(run_id, code, detail)`、`run_discarded(run_id)`。每个信号对同一 `run_id` 至多发一次，`run_discarded` 在从未开始的局上不发。

**本局只读快照（局内 HUD 读取，本系统不绘制）**：

| 字段 | 类型 | 含义 |
|---|---|---|
| `run_id` / `station_id` | int64 / StringName | 本局标识与档口 |
| `run_state` | enum | `Idle` / `Initializing` / `Active` / `Ending` / `Settling` / `Handoff` / `Failed` / `Faulted` |
| `frozen` | bool | 顶层是否处于 `Paused` |
| `turns_remaining` / `turns_total` | int | 来自时间与回合 |
| `capacity_used` / `capacity_total` | int | 来自胃容量 |
| `eaten_count` | int | 已提交的吃菜条数 |
| `end_reason` | enum | 未结束为 `NONE` |
| `run_seed` | int64 | 仅 debug 构建显示 |

快照只读；HUD 不得由快照自行推算结束条件或净利，净利必须走结算的 `running_net`。

## Formulas

本系统只定义四个数值/判定规则：`run_id` 分配、`run_seed` 选择、结束原因判定、状态转移成本。它们都不进数据配置——它们是流程不变量，不是平衡数值；任何一项被改动都意味着流程语义变了，而不是数值调优。

### `run_id` 分配

The `run_id_alloc` formula is defined as:

`run_id = max_seen_run_id + 1`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `max_seen_run_id` | int64 | 0–INT64_MAX | session state | 本会话已分配的最大 `run_id`；启动时为 0 |
| `run_id` | int64 | 1–INT64_MAX | calculated | 本局唯一标识；结算的去重键 |

**Output Range:** 1 到 INT64_MAX。到达 INT64_MAX 后下一次分配返回溢出错误并进入 `Faulted`，绝不回绕到 0 或负数。

**Example:** 会话开始 `max_seen_run_id = 0` → 第一局 `run_id = 1`；第二局 `run_id = 2`。

> **会话作用域**：MVP 的 `run_id` 每会话从 1 开始，不落盘。若纪录需要跨会话全局唯一，必须改为持久化计数器；见 Open Questions。

### `run_seed` 选择

The `run_seed` formula is defined as:

`run_seed = RunConfig.seed` if `RunConfig.seed ≠ null`，否则 `run_seed = int64(next_u64(session_rng))`（64 位按二补码解释）

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `RunConfig.seed` | int64 \| null | 任意 int64 | RunConfig | 调用方指定的种子；`0` 是合法值 |
| `session_rng` | 随机源 | — | 应用会话 | 未指定种子时使用；播种方式属于 ADR |
| `run_seed` | int64 | INT64_MIN–INT64_MAX | calculated | 本局种子；写入 `run_started` 日志 |

**Output Range:** 完整 int64 范围，无排除值。

**Example:** `RunConfig.seed = 0` → `run_seed = 0`（不重新生成）；`RunConfig.seed = null` → 取 `session_rng` 的下一个 64 位值并记录。

> **确定性边界**：同一 `run_seed` 加同一动作序列必须产生同一 `RunLog`；未指定种子时，跨会话不保证相同。实现使用哪种随机源由 ADR 决定，本 GDD 只要求结果可记录、可复现。

### 结束原因判定

The `end_reason` formula is defined as:

`end_reason = TIME_EXHAUSTED if turns_remaining = 0 else (NO_EATABLE_DISH if no_eatable else NONE)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `turns_remaining` | int | 0–`turns_total` | 时间与回合 | 本局剩余回合 |
| `no_eatable` | bool | true / false | 菜单波次信号 | 最后一波吃完、结束取餐或无剩余菜 |
| `end_reason` | enum | `{NONE, TIME_EXHAUSTED, NO_EATABLE_DISH}` | calculated | 本局结束原因 |

**Output Range:** 恰好三者之一。两个条件同时为真时取 `TIME_EXHAUSTED`——回合是玩家持续跟踪的外层资源，收据的浪费容量行能解释剩余差异。

**Example:** 玩家吃下最后一道菜，同时用掉最后一回合 → `turns_remaining = 0`、`no_eatable = true` → `TIME_EXHAUSTED`；玩家在最后一波点「结束取餐」且仍有回合 → `NO_EATABLE_DISH`。

**判定点**：初始化完成后一次；每次成功提交动作后一次。被拒绝的动作与暂停帧不判定。

### 状态转移成本

The `transition_cost` rule is defined as:

`transition_turn_cost = 0` 且 `transition_capacity_cost = 0`，对所有内部转移（`Idle → Initializing → Active`、`Active → Ending`、`Ending → Settling`、`Settling → Handoff`、`Handoff → Idle`、`Active/Paused → Idle` 的丢弃）以及冻结/恢复成立。

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `transition` | enum | 上述内部转移 | calculated | 被检查的转移 |
| `transition_turn_cost` | int | 0 | constant | 该转移消耗的回合 |
| `transition_capacity_cost` | int | 0 | constant | 该转移消耗的容量格 |

**Output Range:** 恒为 0/0。只有玩家的吃菜与跳波动作可以消耗回合或容量，且消耗由时间与回合、菜单波次拥有；初始化、结束、结算、交接、暂停、恢复都不得暗中扣费。

**Example:** 从点击“结束取餐”到进入结算页，`turns_used` 不变；暂停 10 分钟再恢复，`turns_used` 与 `capacity_used` 都不变。

### 运行时不变式（不是公式，是断言）

| 不变式 | 含义 | 违反时 |
|---|---|---|
| `capacity_used = Σ effective_satiety(eaten)` | 已用容量必须等于逐行实际占用之和 | `Faulted`，`CAPACITY_MISMATCH` |
| `eaten` 顺序 = 提交顺序 | 日志不得重排、合并或按价格排序 | 视为实现 bug，`Faulted` |
| `settle` 调用次数 ≤ 1 / `run_id` | 同一局不重复结算 | 第二次调用由结算拒绝；本系统视为 `Faulted` |
| `RunResult` 交付次数 ≤ 1 / `run_id` | 同一结果不重复交付 | 拒绝第二次请求；记录 ERROR |
| `run_id` 严格递增 | 不重用、不回绕 | 分配失败，`Faulted` |

## Edge Cases

- **If `station_id` 不存在**：初始化失败，请求 `Run → Menu, init_failed = true`；不产生 `run_started`、`RunLog`、`RunResult`、纪录。
- **If `upgrade_snapshot` 含负值**（`capacity_bonus < 0`、`cloying_resistance < 0`、`menu_vision < 0`）：初始化失败，`init_failed`；不钳到 0。
- **If `pool_override` 为空、含未知菜或含不属于该档口的菜**：初始化失败，`init_failed`；不静默过滤。
- **If 数据配置不在 `Ready`**：初始化失败，`init_failed`。正常流程不可能发生；失败说明调用时序错误，不按 0 分继续。
- **If `RunConfig.seed = 0`**：合法种子，原样使用，不当作“未指定”。
- **If `RunConfig.seed = null`**：生成并记录种子；同一会话内两局通常不同，复盘依赖记录值。
- **If `run_id` 达到 INT64_MAX**：停止分配并进入 `Faulted`，不回绕、不重用。
- **If 初始化后 `turns_total = 0`**：立即判定为 `TIME_EXHAUSTED`，生成零行 `RunLog`，正常结算（净利 = −入场费）。`turns_total ≥ 1` 由时间与回合保证；此为防御性判定。
- **If 本局一道菜都没吃**：合法失败，`end_reason` 为 `TIME_EXHAUSTED` 或 `NO_EATABLE_DISH`，零行收据，净利为负。
- **If 玩家跳完每一波**：最后一波「结束取餐」触发 `NO_EATABLE_DISH`；0 道菜，正常结算。
- **If 吃一道放不下的菜**：拒绝，不扣回合、不写日志、不改变状态；由 HUD 显示不可选原因。
- **If 焦点菜品已不在当前可见集**（波次已推进或菜已被吃）：拒绝为过期输入，不扣回合。焦点由局内 HUD 解析，本系统不猜测。
- **If 同一帧收到两个动作**：第一个通过完整校验的动作提交，第二个不产生任何效果；不得出现双扣回合或双写日志。
- **If 回合耗尽但仍有可见菜**：`TIME_EXHAUSTED`，剩余菜作废；这是玩家可复盘的失败，不是错误。
- **If 容量已满但仍有可见菜且回合未耗尽**：本局**不**结束；玩家可以付费跳波或等下一波，最终由回合耗尽或无可吃菜结束。不得新增“吃撑”结束原因。
- **If 菜单波次报告 `no_eatable`**：只在最后一波被吃完、玩家结束取餐或本局已无剩余菜时成立；可见菜放不下不触发该信号，不结束本局。
- **If 可见菜都放不进剩余容量**：`no_eatable` 不成立，本局不结束；玩家可以付费跳波或等下一波。
- **If 两个结束条件同帧成立**：`TIME_EXHAUSTED` 优先，写死在判定函数里。
- **If 结束取餐发生在最后一波且仍有回合**：`NO_EATABLE_DISH`，0 回合消耗；这是主动停止，不是错误。
- **If 结算返回 `ALREADY_SETTLED`**：视为本系统重复调用的 bug，进入 `Faulted`；不产生第二份结果。
- **If 结算返回 `UNKNOWN_DISH` / `OVERFLOW` / `CAPACITY_MISMATCH` 等结构化错误**：进入 `Faulted`，不重试、不生成部分收据、不当 0 分。
- **If 初始化失败或故障返回请求因同帧 `Menu → Run` 被拒绝**：留在 `Failed` / `Faulted`，下一帧重发；不得第二次初始化或第二次结算。
- **If 结算已完成但玩家在暂停菜单退出**：结果不交付、不写纪录；`Ending` 之后界面不再提供“放弃本局”（见 UI Requirements），避免“算完但被放弃”的窗口。
- **If 同一帧内暂停与结束动作同时到达**：按场景与流程管理先判定顶层状态；本系统看到 `frozen` 后不提交动作。动作已提交则结算等待恢复；暂停中放弃则按“放弃不是结算”丢弃。
- **If 暂停发生在 `Settling` 之前**：不调用 `settle`，等待恢复；暂停期间的结算请求一律不发起（与结算 GDD 一致）。
- **If 暂停发生在 `Handoff` 中**：保留 `RunResult`，不请求交接；恢复后重发。若玩家在暂停中放弃，结果不交付。
- **If `Run → Menu` 请求被拒绝**：保留同一 `RunResult` 并在回到 `Run` 后重发；不重新结算、不重建结果。
- **If 顶层状态在不由本系统交接的情况下离开 `Run` / `Paused`**（放弃、退出、窗口关闭）：丢弃全部局内状态，广播 `run_discarded`，不结算、不写纪录。
- **If 玩家在暂停菜单放弃一局**：不产生 `RunResult`、不结算、不写纪录；下一局使用全新 `RunConfig`。
- **If 一局在 `Run` / `Paused` 中崩溃、被系统杀掉或断电**：MVP 无中途存档，本局作废；不产生收益或纪录。
- **If 重开**：新 `run_id`、新种子、全部子系统重置；上一局的任何字段都不延续。跨局残留是最容易被漏掉的 bug 类型。
- **If `pool_override` 用于首局引导**：仅替换本局的菜池；价格、饱腹、结算规则完全不变。若菜单波次当前只读 `StationData.dish_pool`，该字段暂不可用，见 Open Questions。
- **If 首局引导压入 Blocked 上下文**：本局状态、结束条件与生命周期不变；指导层只能限制输入，不能改规则。
- **If 窗口失去焦点**：不自动暂停（场景与流程管理规则），本局状态不变。
- **If 中途获得升级**：本局不生效；`upgrade_snapshot` 在开局冻结，下一局才变化。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 场景与流程管理 | 硬 | `Menu → Run` 取得 `RunConfig`；冻结/恢复通知；`request_transition(Run → Menu, RunResult \| init_failed)`；观察顶层状态识别放弃/退出 | GDD 已写（In Review） |
| 数据配置 | 硬 | 前提 `Ready`；`get_station(station_id)` 必须存在 | GDD 已写（In Design） |
| 输入 | 硬 | `ui_accept`、`game_skip_wave`；`ui_pause` 不由本系统读取 | GDD 已写（In Design） |

**下游依赖方（本系统在运行时协调它们；依赖方向上它们依赖本系统）：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 时间与回合 | 硬 | `initialize(turn_budget) → turns_total`、`spend_turn()`、`turns_remaining`、`turns_used`；`TIME_EXHAUSTED` 判据 | GDD 已写（`design/gdd/time-and-turns.md`） |
| 胃容量 | 硬 | `initialize(upgrade_snapshot) → capacity_total`、`capacity_used`、`can_fit(effective_satiety)`、`consume(e)` | GDD 已写（`design/gdd/stomach-capacity.md`） |
| 菜单波次 | 硬 | `initialize(station_id, seed, turn_budget, menu_vision[, pool_override])`；`consume(dish_id) → {ADVANCED, NO_EATABLE}`；`visible_dishes`；`advance()` 只承载付费跳波；无菜可吃信号 | GDD 已写（Designed） |
| 腻度与口味 | 硬 | `run_started` / `run_ended` 生命周期；吃下时 `extra_satiety`、`effective_satiety` | GDD 已写（Designed） |
| 结算与入场费 | 硬（回调） | `settle(RunLog) → RunResult \| 结构化错误`；每局一次 | GDD 已写（Designed） |
| 局内 HUD | 硬 | 只读本局快照与 `run_*` 信号；不得自行求和或判定结束 | GDD 已写（`design/gdd/in-game-hud.md`） |
| 界面流程 | 硬 | 经场景与流程管理发出 `RunConfig`、接收 `RunResult`；重开时组装新 `RunConfig` | GDD 已写（`design/gdd/ui-flow.md`） |
| 首局引导 | 软 | `is_first_run`、`pool_override`；可压入 Blocked 上下文 | GDD 已写（`design/gdd/first-run-onboarding.md`） |
| 音效反馈 | 软 | 订阅 `run_started` / `run_ended` / `run_settled` / `run_discarded` / `run_faulted`；`run_settled` 不产生独立 cue，声音由结算的三个事件承载 | GDD 已写（`design/gdd/audio-feedback.md`） |
| 存档 | 无直接调用 | MVP 无中途存档；局外升级与纪录由下游持久化 | GDD 已写（`design/gdd/save-system.md`） |

**为什么下游是“依赖本系统”而不是“本系统依赖它们”**：本系统在运行时调用它们，但设计依赖按**生命周期所有权**方向记录——它们需要本系统发出开始/结束/结算时机的信号，本系统不需要它们定义“什么时候算一局”。若把它们列为本系统的上游，会与索引中“时间与回合 → 餐局流程”“胃容量 → 餐局流程”“结算 → 餐局流程”的记录形成环。腻度与口味 GDD 已用同一理由避免与局外升级成环，这里沿用。

**不属于依赖的关系**：

- **菜品数值**：本系统不直接读取 `price`、`satiety` 或 `unit_value`；这些由菜单波次、腻度与结算消费。间接关系，不记为依赖。
- **局外升级**：本系统只消费 `RunConfig.upgrade_snapshot` 中的已解析值，不读升级表、不算等级效果。
- **存档**：MVP 不持久化局内状态；放弃、崩溃与退出都不留下可恢复的一局。

**双向一致核对**：

- 已回指本系统的 GDD：场景与流程管理、数据配置、输入、腻度与口味、菜单波次、结算与入场费。
- **数据配置的行归属（不在本文件修改）**：`data-config.md` 的交互表把 `StationData.entry_fee` 记为「餐局流程读取」，但本设计中入场费由结算读取；本系统只解析 `station_id`。需在一致性检查时修正该行。
- **索引缺口（不在本文件修改）**：`systems-index.md` 第 9 行（腻度与口味）与第 10 行（菜单波次）的 `Depends On` 尚未列出「餐局流程」，而两份 GDD 都已把本系统记为硬上游；需由编排会话统一补正。
- **索引第 5 行**：`Status` 应由 `Not Started` 改为 `Designed`，`Design Doc` 应指向 `design/gdd/run-flow.md`；由编排会话统一处理。

## Visual/Audio Requirements

**本系统不绘制任何元素、不播放任何声音。** 它只定义五个交接点，视觉与音效由界面流程、局内 HUD 与音效反馈实现：

1. **开始一局**：`Run` 生效的同一帧显示局内画面；不插入加载屏、过场或“准备中”。首帧必须已经能看见回合、容量与当前波。
2. **动作提交**：吃菜、跳波、结束取餐的数字变化在同一帧完成；不得先播动画再改数（支柱 4）。
3. **结束一局**：进入 `Ending` 后，局内画面保持最终状态只读显示，直到结算结果就绪；不播放“结算中”占位。
4. **结算交接**：收据屏由界面流程打开，数据必须是本系统交出的同一份 `RunResult`；本系统不做任何汇总或格式化。
5. **丢弃一局**：放弃或退出直接回菜单，不出现收据、不庆祝、不给部分奖励——这是支柱 2 在表现层的落点。故障返回同样平静，debug 构建可显示错误码。

**音效事件**（由音效反馈消费）：`run_started`、`run_ended`、`run_settled`、`run_discarded`、`run_faulted`。本系统只发事件，不选音色。

**资产**：本系统不产生任何美术或音频资产。

## Game Feel

**目标手感是“一局像一次真实的用餐”：进店、取餐、结账。** 流程本身不应被感知；玩家只应感觉到数字在变。

- **即时**：一次吃菜或跳波从输入到数字变化不超过 1 帧（60 FPS 下 16.7 ms）。没有确认窗口、没有动画锁。
- **无等待**：开局、结束、结算与重开之间不插入加载或过场；淡入淡出属于表现层，且不推迟状态生效。
- **暂停是冻结，不是中断**：暂停与恢复不丢失画面、不改变数字、不消耗回合；恢复后玩家看到的世界与暂停前完全一致。
- **失败可归因**：结束原因只有两种，且都能在收据上被解释——时间用完看浪费容量，无菜可吃看剩余菜。玩家不应觉得“这局莫名其妙就结束了”。
- **放弃有代价**：中途放弃不产生任何收益、不写纪录；这是支柱 2 的流程层落点，不能做成“提前结账”。
- **不做**：结算前的庆祝动画、开局倒计时、自动暂停、需要反应速度的任何确认。

## UI Requirements

本系统不拥有任何屏幕，但对界面流程与局内 HUD 提出约束：

1. **HUD 只读快照**：局内 HUD 必须读取本系统发布的 `run_state` 快照与 `run_*` 信号，不得自行推算“是否结束”、不得另写一份总值或净利求和（结算与入场费规则 19）。
2. **结束即禁用**：进入 `Ending` 或 `frozen = true` 时，所有吃菜、跳波、结束取餐控件必须禁用；不得出现“点了没反应”。
3. **结算必须用同一份结果**：收据屏显示的数据必须来自本系统交出的 `RunResult`，不得在界面层重新计算净利、总值或回本判定。
4. **重开是新局**：结算屏“再来一局”必须发起新的 `RunConfig`；不得复用上一局的种子、升级快照或局内状态，除非调用方显式指定同一个 `seed`。
5. **无加载态**：`Run → Menu` 与 `Menu → Run` 之间不得插入加载屏；收据屏在切换生效的同一帧打开。
6. **最终画面保留**：从结束判定到收据出现之间，局内最终状态必须可见，不得先清空再显示收据。
7. **首局引导**：指导层可以压入 Blocked 上下文并聚焦指定控件，但不得隐藏回合、容量或净利数字；它改变的是可操作性，不是规则。
8. **无障碍**：不存在限时确认；暂停冻结一切；结束原因必须用文字给出，不能只靠颜色或图标。
9. **结算阶段不提供放弃**：进入 `Ending` 后，暂停菜单只提供「继续」与「退出」，不提供「放弃本局」。放弃语义只属于仍在 `Active` 的一局；否则会出现“已算完但被放弃”的窗口。

> **📌 UX Flag — 餐局流程**：本系统对局内 HUD 的流程态（活动/冻结/结束/交接）与收据屏的打开时机提出约束。Phase 4 写 epic 之前，先用 `/ux-design` 覆盖 `design/ux/hud.md` 与 `design/ux/settlement.md` 的这些状态；相关 story 引用 UX 规格，不直接引用本 GDD。

## Cross-References

| 系统 | GDD | 关系 |
|---|---|---|
| 场景与流程管理 | design/gdd/scene-flow.md | 上游：取得 `RunConfig`、冻结/恢复通知、`request_transition`；本 GDD 回答其 `RunConfig` / `RunResult` schema 归属 Open Question |
| 数据配置 | design/gdd/data-config.md | 上游：`Ready` 前提与 `station_id` 解析 |
| 输入 | design/gdd/input.md | 上游：`ui_accept`、`game_skip_wave`；`ui_pause` 归场景与流程管理 |
| 时间与回合 | 未写 | 下游：回合预算、消耗与耗尽判据 |
| 胃容量 | 未写 | 下游：容量、可吃性判定 |
| 菜品数值 | design/gdd/dish-values.md | 间接：由菜单波次、腻度与结算消费，本系统不直接读取 |
| 腻度与口味 | design/gdd/cloying-and-taste.md | 下游：生命周期事件与吃下时的 `extra_satiety` / `effective_satiety` |
| 菜单波次 | design/gdd/menu-waves.md | 下游：波次初始化、可见集、跳波与无菜可吃信号 |
| 结算与入场费 | design/gdd/settlement-and-entry-fee.md | 下游（回调）：`settle(RunLog) → RunResult`；`RunResult` 字段归它所有 |
| 局内 HUD | 未写 | 下游：只读本局快照与事件 |
| 界面流程 | 未写 | 下游：组装 `RunConfig`、接收 `RunResult`、重开 |
| 首局引导 | 未写 | 下游（软）：`is_first_run`、`pool_override`、Blocked 上下文 |
| 音效反馈 | 未写 | 下游（软）：`run_*` 事件 |
| 存档 | 未写 | 无直接调用：MVP 无中途存档 |

**待回指（不由本文件修改）**：局内 HUD、界面流程、首局引导、音效反馈的 GDD 写好后，需在各自的 Cross-References 与 Dependencies 中列出「餐局流程」。时间与回合、胃容量、菜单波次、腻度与口味、存档已回指。

## Acceptance Criteria

> **Formulas 覆盖**：`run_id_alloc` 由第 3 条；`run_seed` 由第 2 条；`end_reason` 由第 12、13、36 条；`transition_cost` 由第 26 条。运行时不变式由第 3、14、16–19 条覆盖。其余判据来自 Detailed Design。

> **集成判据**：第 1–3、6–13、14–19、26–28、29–31、33–34、36 条需要时间与回合、胃容量、菜单波次、腻度、结算的测试替身；第 4、18、19、32、37、38 条还需要场景与流程管理的顶层状态替身。只测本系统的判据使用 `Idle/Active/Ending` 状态探针即可。

> **术语**：`run_state` = 本系统的局内状态枚举；`frozen` = 顶层 `Paused` 的冻结标志；`no_eatable` = 菜单波次的“无可吃菜”信号；`Faulted` = 不变式或结算失败；`Handoff` = 持有 `RunResult` 等待 `Run → Menu` 生效。

1. **GIVEN** 合法的 `RunConfig`，**WHEN** `Menu → Run` 生效，**THEN** 分配 `run_id`、确定并记录 `run_seed`；时间与回合、胃容量、菜单波次按固定顺序各初始化一次，并分别收到 `turn_budget`（返回 `turns_total`）、`upgrade_snapshot`、`(station_id, seed, turn_budget, menu_vision[, pool_override])` 参数；`RunLog.entry_fee` 等于 `RunConfig.entry_fee_override`（`null` 时取 `StationData.entry_fee`）；腻度在 `run_started` 时重置；`run_started` 恰好广播一次；`turns_used = 0`、`capacity_used = 0`。
2. **GIVEN** `RunConfig.seed = 0`，**WHEN** 开局，**THEN** `run_seed = 0` 且不重新生成；**GIVEN** `seed = null`，**THEN** 生成一个种子并写入 `run_started`。
3. **GIVEN** 会话中已分配 `run_id = 7`，**WHEN** 开始下一局，**THEN** 新 `run_id = 8`；**GIVEN** `max_seen_run_id = INT64_MAX`，**WHEN** 再分配，**THEN** 返回溢出错误并进入 `Faulted`，不回绕。
4. **GIVEN** `station_id` 不存在，**WHEN** 开局，**THEN** 请求 `Run → Menu, init_failed = true`，不广播 `run_started`、不产生 `RunLog` / `RunResult` / 纪录，最终状态为 `Idle`。
5. **GIVEN** `upgrade_snapshot` 的 `capacity_bonus = -1`、`cloying_resistance = -1` 或 `menu_vision = -1`（三个夹具），**WHEN** 开局，**THEN** `init_failed`，不钳到 0、不开始一局。
6. **GIVEN** 当前焦点菜仍在可见集且 `effective_satiety` 不超过剩余容量，**WHEN** 按 `ui_accept`，**THEN** 恰好追加一条 `EatenEntry`（顺序正确、`dish_id` / `extra_satiety` / `effective_satiety` 正确）、腻度计数更新一次、该菜移出当前波、`turns_used` 加 1、`capacity_used` 增加对应格数。
7. **GIVEN** 一道菜放不进剩余容量，**WHEN** 按 `ui_accept`，**THEN** 拒绝；`turns_used`、`capacity_used`、`eaten` 与状态均不变。
8. **GIVEN** 焦点菜已不在可见集（波次已推进或已被吃），**WHEN** 按 `ui_accept`，**THEN** 作为过期输入拒绝，不扣回合、不写日志。
9. **GIVEN** 同一帧收到两个动作输入，**WHEN** 处理该帧，**THEN** 只有第一个通过完整校验的动作提交，第二个无任何效果；不得双扣回合或双写 `eaten`。
10. **GIVEN** 非最后一波，**WHEN** 按 `game_skip_wave`，**THEN** 当前波剩余菜作废、推进一波、`turns_used` 加 1。
11. **GIVEN** 最后一波，**WHEN** 按 `game_skip_wave`（结束取餐），**THEN** 不扣回合、剩余菜作废、`no_eatable` 信号恰好一次、`end_reason = NO_EATABLE_DISH` 并进入 `Ending`。
12. **GIVEN** 一次成功动作后 `turns_remaining = 0`，**WHEN** 结束判定，**THEN** `end_reason = TIME_EXHAUSTED`，此后所有动作被拒绝；**GIVEN** 同时 `no_eatable = true`，**THEN** 仍取 `TIME_EXHAUSTED`。
13. **GIVEN** 初始化完成且 `turns_total = 0`，**WHEN** 初始判定，**THEN** 立即 `TIME_EXHAUSTED`，生成零行 `RunLog` 并正常结算。
14. **GIVEN** `Ending` 中组装 `RunLog`，**WHEN** 检查字段，**THEN** `run_id`、`station_id`、`entry_fee`、`end_reason`、`eaten` 顺序、`capacity_total`、`capacity_used`、`turns_total`、`turns_used`、`upgrade_snapshot` 全部来自动作发生时刻的记录；`capacity_used` 等于各行 `effective_satiety` 之和。**WHEN** 组装后修改数据配置与腻度状态再读取日志，**THEN** 日志仍是旧值（证明不重算）。
15. **GIVEN** `capacity_used` 与逐行之和不等，**WHEN** 组装，**THEN** 进入 `Faulted`，不调用结算、不生成部分 `RunLog`。
16. **GIVEN** 合法 `RunLog`，**WHEN** 进入 `Settling`，**THEN** `settle` 恰好调用一次；成功则进入 `Handoff`，`run_settled` 的 payload 与最终交付的 payload 深比较相等，且 `run_settled` 只广播一次。
17. **GIVEN** 结算返回 `ALREADY_SETTLED`、`UNKNOWN_DISH`、`OVERFLOW` 或 `CAPACITY_MISMATCH`，**WHEN** 返回，**THEN** 进入 `Faulted`，不重试、不生成第二份结果、不按 0 分继续。
18. **GIVEN** `Handoff` 中 `Run → Menu` 请求被拒绝，**WHEN** 下一可用帧仍在 `Run`，**THEN** 重发同一份 `RunResult`；`settle` 未被再次调用，`RunResult` 未被重建。
19. **GIVEN** `Handoff` 中请求被接受，**WHEN** 切换生效，**THEN** `RunResult` 恰好交付一次，本局状态清空为 `Idle`。
20. **GIVEN** `frozen = true`，**WHEN** 尝试吃菜、跳波，或注入一次结束判定、结算、交接调用，**THEN** 全部被拒绝；`turns_used`、`capacity_used`、`eaten` 与 `run_state` 不变。
21. **GIVEN** 暂停后恢复，**WHEN** 恢复，**THEN** 状态与暂停前逐字段一致，`frozen = false`，且整个过程 `turns_used` 与 `capacity_used` 不变。
22. **GIVEN** `Active` 中玩家从暂停菜单放弃，**WHEN** 顶层进入 `Menu`，**THEN** 广播 `run_discarded` 一次，不调用 `settle`、不产生 `RunResult`、不写纪录。
23. **GIVEN** `Active` 中玩家退出或确认关闭窗口，**WHEN** 顶层进入 `Quit`，**THEN** 同样丢弃且不结算。
24. **GIVEN** 已完成一局后再次 `Menu → Run`，**WHEN** 开局，**THEN** `run_id` 加 1；`seed = null` 时使用新种子，显式种子则沿用；时间与回合、胃容量、菜单波次、腻度全部重置；上一局的任何字段都不延续。
25. **GIVEN** 相同 `RunConfig`（含显式种子）与相同动作序列，**WHEN** 重放两局，**THEN** `run_seed`、`end_reason` 与 `RunLog` 除 `run_id` 外逐字段相同。
26. **GIVEN** 任意内部转移（初始化、结束、结算、交接、暂停、恢复），**WHEN** 发生，**THEN** `transition_turn_cost = 0` 且 `transition_capacity_cost = 0`；只有吃菜与跳波能改变回合或容量。
27. **GIVEN** 容量已满但仍有可见菜且回合未耗尽，**WHEN** 连续 3 帧不做任何动作（或下一次输入前），**THEN** `run_state` 仍为 `Active` 且 `end_reason = NONE`；玩家仍可跳波，最终只由回合耗尽或无可吃菜结束。
28. **GIVEN** 本局 0 道菜，**WHEN** 结算，**THEN** `RunLog.eaten` 为空、`capacity_used = 0`、`turns_used` 与本局一致、`entry_fee` 等于本局入场费；结算能接收该 `RunLog` 并生成零行收据。
29. **GIVEN** `is_first_run = true` 且 `pool_override` 是档口菜池的非空子集，**WHEN** 开局，**THEN** 菜单波次收到该子集，价格与结算规则不变。
30. **GIVEN** `pool_override` 含未知菜或不属于该档口的菜，**WHEN** 开局，**THEN** `init_failed`，不静默过滤。
31. **GIVEN** 首局引导压入 Blocked 上下文，**WHEN** 本局继续，**THEN** `run_state` 仍为 `Active`、`run_id` 不变、未额外广播 `run_started` / `run_ended`，结束条件不变。
32. **GIVEN** 注入 `CAPACITY_MISMATCH` 故障，**WHEN** 离开 `Run`，**THEN** 不产生 `RunResult`、不写纪录，ERROR 日志含 `run_id` 与违反字段，`Run → Menu` payload 为 `init_failed = true` 且带 `fault` 明细，顶层最终停在 `Menu` 而不是卡在 `Run`。
33. **GIVEN** 一局正常开始与结束，**WHEN** 检查事件，**THEN** `run_started`、`run_ended`、`run_settled` 各恰好一次；`run_discarded` 对该局为 0 次。
34. **GIVEN** 局内 HUD 显示回合、容量与净利，**WHEN** 每次动作后与快照及结算的 `running_net(k)` 对比，**THEN** 三处数字一致；HUD 的净利来自 `running_net(k)`，不自行为求和。
35. **GIVEN** 数据配置不在 `Ready`，或任一子系统初始化抛出错误，**WHEN** 开局，**THEN** `init_failed`，不广播 `run_started`、不产生 `RunLog` / `RunResult`，最终 `Idle`。
36. **GIVEN** 最后一波的最后一道菜被吃下，**WHEN** 动作提交后判定，**THEN** `no_eatable` 恰好一次、`end_reason = NO_EATABLE_DISH`、进入 `Ending`；若同时回合耗尽则取 `TIME_EXHAUSTED`。
37. **GIVEN** `turns_total = 0` 的局在帧 N 经 `Menu → Run` 生效，**WHEN** 同帧请求 `Run → Menu`，**THEN** 请求被拒绝；本系统不重复初始化或结算，留在 `Handoff`，在帧 N+1 重发并被接受。
38. **GIVEN** 初始化失败发生在 `Menu → Run` 生效的同一帧，**WHEN** `Run → Menu (init_failed)` 被拒绝，**THEN** 留在 `Failed`，下一帧重发；`run_started` 从未广播。
39. **GIVEN** 可见菜都放不进剩余容量但 `no_eatable = false` 且回合未耗尽，**WHEN** 保持 `Active`，**THEN** 本局不结束，`run_state = Active`。
40. **GIVEN** `ui_accept` 携带的焦点菜不在当前可见集，**WHEN** 处理，**THEN** 拒绝，`turns_used` 与 `eaten` 不变。
41. **GIVEN** 本局进入 `Ending`，**WHEN** 打开暂停菜单，**THEN** 不提供「放弃本局」，只提供「继续」与「退出」。

**无法直接断言、已改写或转为开放问题的规则**：

- “不伪造结果”：黑盒只能验证“没有 `RunResult` 交付”，不能验证内部是否构造过；第 17、32 条改为事件与出口断言。
- “确定性”：跨会话的未指定种子不保证相同，第 25 条只在同一 `RunConfig`（含显式种子）或同一会话记录值下重放。
- “失败不是卡死”：需要场景与流程管理的顶层状态替身；第 32 条验证最终停在 `Menu`，不验证玩家主观体验。
- “首局钩子”：`pool_override` 是否能被菜单波次接受取决于未写的首局引导，见 Open Questions。
- “职责边界”（规则 1）：黑盒不可判 → 通过第 34 条的数字来源与第 6 条的日志结构间接验证。
- “吃菜校验顺序”（规则 6）：顺序是内部实现 → 通过注入“容量不足但回合可用”的用例验证整体拒绝，不验证中间步骤。
- “数据前提”（规则 22）：`Ready` 是上游保证 → 第 35 条用未就绪桩验证 `init_failed`。
- “放弃后重置”（规则 16/17）：第 22、24 条验证不结算与全新字段。

## Open Questions

| # | 问题 | 负责人 | 何时解决 |
|---|---|---|---|
| 1 | **结算失败的返回通道**：场景与流程管理只有 `init_failed`，没有 `run_faulted`。本 GDD 暂定复用 `init_failed = true` 并附 `fault` 明细。是否给 #3 增加独立 payload（或允许无 payload 返回）？ | systems-designer + game-designer | 场景与流程管理复审时 |
| 2 | **输入 GDD 把 `ui_pause` 列为本系统的读取项**，但顶层暂停由场景与流程管理消费；本系统不应重复读取。是否修正输入 GDD 的下游行？ | game-designer（输入） | `/consistency-check` 时 |
| 3 | **`game_skip_wave` 的语义与命名**：输入 GDD 描述为“跳过悬停的菜品卡”，菜单波次实现为“跳过当前波 / 最后一波结束取餐”。是否改名为 `game_skip_wave` 或拆成两个动作？ | game-designer（输入 + 菜单波次） | 写 `/ux-design` 局内 HUD 前 |
| 4 | **放弃/退出的清理通知**：场景与流程管理从 `Paused → Menu (abandoned)` 或 `Quit` 时没有显式通知本系统；本 GDD 暂定由本系统观察顶层状态来丢弃。是否增加 `run_discarded` 通知？ | systems-designer | 场景与流程管理复审时 |
| 5 | **首局 6 道菜的菜池注入**：本 GDD 预留 `RunConfig.pool_override`，但菜单波次当前只读 `StationData.dish_pool`。是用专用档口、数据覆盖还是让菜单波次接受覆盖？ | game-designer（首局引导 + 菜单波次） | 首局引导 GDD 编写时 |
| 6 | **`run_id` 是否跨会话持久化**：MVP 每会话从 1 开始；若纪录需要全局唯一或跨会话比较，需要持久化计数器。 | systems-designer（存档 + 纪录） | 存档 GDD 编写时 |
| 7 | **暂停时的结算边界**：本 GDD 规定暂停中不调用 `settle`，恢复后再结算；并已要求 `Ending` 之后不提供「放弃本局」来关闭“算完但被放弃”的窗口。是否在顶层再加一个“已提交”子状态，仍需与场景与流程管理统一；结算 GDD 也应明确“结算已完成后放弃”不属于其规则 8 的放弃。 | game-designer + systems-designer | 场景与流程管理复审时 |
| 8 | **是否需要主动提前结账**：本 GDD 回答结算 Open Question 2 为“MVP 不提供”，最后一波「结束取餐」已覆盖主动停止。若试玩显示玩家需要“收手”，需新增 `PLAYER_CASH_OUT` 结束原因并先改结算枚举。 | game-designer | 垂直切片试玩后 |
| 9 | **“吃撑”是否应成为结束原因**：当前胃容量只阻止继续吃，不结束本局。若试玩认为“吃不下却还有回合”是空转，需要新增第三结束原因并同步结算。 | game-designer | 垂直切片试玩后 |
| 10 | **数据配置把 `entry_fee` 的读取记在餐局流程名下**，但本设计由结算读取；本系统只解析 `station_id`。是否修正 `data-config.md` 的下游行？ | game-designer | `/consistency-check` 时 |
| 11 | **`upgrade_snapshot` 的语义**：结算 GDD 称其为“等级快照”（`Dictionary`），本 GDD 要求已解析修正值。两者以“结算不解释”兼容；若纪录按等级解释，快照需同时保留等级与解析值。 | systems-designer | 局外升级 / 纪录 GDD 编写时 |

**GDD 定义行为，ADR 记录实现路径。** 以下是需要在 ADR 中落地的实现问题：一局状态机的载体（节点脚本 / 自动加载 / 纯对象）、`run_*` 信号与直接调用的选择与重入顺序、`session_rng` 的随机源与种子来源、`run_id` 的会话作用域与持久化、`RunLog` 的不可变构造方式、以及 `Faulted` 的返回通道。封闭的状态表、动作提交顺序、结束优先级与“放弃不是结算”的**行为**已在正文定义，ADR 只决定实现手段。
