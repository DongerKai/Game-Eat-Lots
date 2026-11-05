# ADR-0013: 局内动作提交、结束判定与冻结

## Status

Proposed

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。其他 agent 与任何 skill 都不能自行设置。**

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

user + agents（lean 模式；本 ADR 由编排会话起草，待用户接受）

## Summary

本 ADR 固定 `run-flow.md` 里**一次动作如何变成一次提交**：`submit_action(action, dish_id)` 是唯一入口，动作只有 `eat` / `skip_wave` / `end_pickup` 三种；每种动作有写死的校验顺序，全部校验通过后才原子提交，任何一步失败都不改变状态、不扣回合、不写日志。结束判定只在一局初始化后与每次成功提交后各执行一次，只产生 `TIME_EXHAUSTED` 或 `NO_EATABLE_DISH` 两种原因，且 `TIME_EXHAUSTED` 优先；命中后同帧进入 `Ending` 并冻结所有后续动作。本 ADR 不重定义局生命周期与结算事务（ADR-0007），只补齐动作管线与结束裁决。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Core |
| **Layer** | Core |
| **Knowledge Risk** | LOW — 纯 GDScript 状态机与整数运算，不依赖 4.5–4.7 新 API |
| **References Consulted** | `design/gdd/run-flow.md`、`design/gdd/time-and-turns.md`、`design/gdd/stomach-capacity.md`、`design/gdd/menu-waves.md`、`design/gdd/cloying-and-taste.md`、`design/gdd/settlement-and-entry-fee.md`、`docs/architecture/architecture.md`、`docs/engine-reference/godot/VERSION.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① 一次 `eat` 提交后 `turns_used` 恰好 +1、`capacity_used` 恰好 +`effective_satiety`、`eaten` 恰好 +1、`action_seq` 恰好 +1；② 任一校验失败时上述四项与 `running_net` 全部不变；③ `turns_remaining = 1` 的非最后一波 `skip_wave` 返回 `NO_TURNS`，且 `advance()` 与 `spend_turn()` 的调用计数都为 0；④ 同一帧注入两个动作时第二个不产生任何写入（由输入层单动作契约 + 集成夹具验证）；⑤ `Ending` 后任意 `submit_action` 返回 `NOT_ACTIVE` 且不再产生信号 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述五条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0004（服务组合与信号）— `RunFlow` 的构造、注入与信号派发；ADR-0007（局生命周期与结算事务）— 状态枚举、`run_id`、`RunLog` / `RunResult` 与冻结语义；ADR-0010（菜品数值）— 吃菜提交时的 `price`；ADR-0011（时间与回合）— `can_spend_turn` / `spend_turn` / 耗尽判据；ADR-0012（胃容量）— `can_fit` / `consume` |
| **Enables** | ADR-0014（腻度与口味）的吃菜记录点；ADR-0015（结算）的 `RunLog` 来源；ADR-0019（HUD）的动作结果与快照；ADR-0020（首局引导）的 Blocked 输入 |
| **Blocks** | `run-flow` 的全部动作实现 story；`TR-run-flow-003` / `TR-run-flow-004` 的验收；任何「吃一口 / 跳一波 / 结束取餐」的端到端测试 |
| **Ordering Note** | 本 ADR 不定义初始化顺序（`TR-run-flow-002`，归 ADR-0007）、不定义 `RunLog` 组装与结算交接（`TR-run-flow-005`，归 ADR-0007）、不定义信号全表（`TR-run-flow-006`，归 ADR-0007）、不定义 HUD 快照字段（`TR-run-flow-007`，归 ADR-0019）。它只定义动作校验/提交与结束裁决 |

## Context

### Problem Statement

`run-flow.md` 已经写死了三种动作与两种结束原因，但没有回答实现层最容易出错的三件事：

1. **校验顺序与原子性。** 吃一道菜要经过可见集、腻度、容量、回合四道闸门；如果边校验边修改，任何一步失败都会留下半提交（扣了回合没写日志、写了日志没扣容量）。顺序还必须固定，否则「容量不足但回合可用」会被误报成 `NO_TURNS`。
2. **跳波的最后一回合。** `can_spend_turn()` 在剩 1 回合时为 `true`，但非最后一波付费跳波在剩 1 回合时必须拒绝——否则玩家花掉最后一回合进入一波，却没有任何回合可吃。这个 +1 余量是动作语义，不属于回合服务的通用判定。
3. **结束裁决的唯一性与时机。** 结束判定只能在一处、两个固定时机执行；如果每个动作各自判断，同一帧会出现两次 `run_ended`，或者 `NO_EATABLE_DISH` 抢先于 `TIME_EXHAUSTED`。

### Current State

`src/` 只有占位文件。`architecture.md` 已给出 `RunFlow.submit_action(action, dish_id) -> ActionResult` 与 `run_state` 枚举，ADR-0007 已固定生命周期、`run_id` 与结算事务。缺的是动作枚举、拒绝码词表、逐动作校验顺序、提交顺序与结束判定函数。

### Constraints

- **一次输入至多一个动作**：动作由输入层按事件投递；`RunFlow` 不轮询输入、不读 `ui_accept` / `game_skip_wave`。
- **先校验后提交**：任何写入之前完成全部校验；被拒绝的动作零副作用。
- **只有两种结束原因**：没有 `STUFFED`、没有 `PLAYER_CASH_OUT`；容量满不结束本局。
- **固定优先级**：同帧同时满足回合耗尽与无可吃菜时取 `TIME_EXHAUSTED`。
- **结束即冻结**：`Ending` 之后不再接受动作、不再消耗、不再推进波次。
- **值来自动作发生时刻**：`eaten` 里的 `extra_satiety` / `effective_satiety` 是提交那一刻的读数，结算不得重放。
- **不拥有数值**：价格来自 `DishValues`，回合来自 `TimeTurns`，容量来自 `StomachCapacity`，波次来自 `MenuWaves`，腻度来自 `CloyingAndTaste`。
- **单人单线程**：无并发；校验与提交之间没有其他写入者，不需要锁或预留。

### Requirements

- 必须提供 `submit_action(action, dish_id) → COMMITTED | REJECTED(code)`，`code` 为封闭词表。
- 必须为 `eat` / `skip_wave` / `end_pickup` 各自固定校验顺序。
- 必须让吃菜提交按固定顺序写入 `EatenEntry`、腻度计数、波次、回合与 `running_net`，并保证全有或全无。
- 必须让非最后一波在 `turns_remaining = 1` 时返回 `NO_TURNS`，且在任何 `spend_turn()` / `advance()` 之前返回。
- 必须只在一局初始化后与每次成功提交后判定一次结束。
- 必须只产生两种 `end_reason`，并固定 `TIME_EXHAUSTED` 优先。
- 必须让 `Ending` 冻结所有动作与波次推进。
- 必须让 `action_seq` 每次成功提交 +1，且被拒绝的动作不增加。

## Decision

### 1. 动作与结果词表

```gdscript
enum RunAction { EAT, SKIP_WAVE, END_PICKUP }

enum RejectCode {
    FROZEN,            # 顶层 Paused
    NOT_ACTIVE,        # 不在 Active（Ending / Settling / Handoff / Failed / Faulted / Idle）
    STALE_FOCUS,       # dish_id 不在当前可见集
    CAPACITY,          # effective_satiety 放不进剩余容量
    NO_TURNS,          # 无可用回合支付该动作
    NOT_FINAL_WAVE,    # end_pickup 只允许在最后一波
    NO_EATABLE,        # 菜单波次已无任何可吃菜
}

class ActionResult:
    var committed: bool
    var action: RunAction
    var reject_code: RejectCode   # committed = false 时有效
```

`submit_action(action: RunAction, dish_id: StringName = &"") -> ActionResult` 是唯一动作入口。`RunFlow` 不读输入；`InputRouter` / 局内 HUD 经 `AppRoot` 调用它。`dish_id` 只对 `eat` 有意义，其余动作忽略。

**必须**：一次输入事件最多调用一次 `submit_action`；同一帧的第二个动作由 `InputRouter` 拦下（ADR-0005 的「首个消费者获胜」）。`RunFlow` 不读帧计数器、不维护「本帧已提交」标志——那是输入层的契约，不是流程逻辑。

### 2. 公共前置条件

任何动作先检查：

1. `frozen == true` → `REJECTED(FROZEN)`；
2. `run_state != ACTIVE` → `REJECTED(NOT_ACTIVE)`。

两项都在动作语义校验之前，且不改变任何字段。暂停中、`Ending` 之后、初始化失败或故障中的调用全部在此返回。

### 3. 吃菜 `eat`：四道闸门、一次原子提交

校验顺序**固定**（任何一步失败立即返回，后续步骤不执行）：

| # | 校验 | 失败码 | 说明 |
|---|---|---|---|
| 1 | `menu_waves.is_visible(dish_id)` | `STALE_FOCUS` | 菜必须仍在当前可见集；焦点由 HUD 解析，本系统不猜 |
| 2 | `cloying.effective_satiety(dish) → {extra, effective}` | — | 读取，不改计数；`effective ≥ 1` 由腻度保证 |
| 3 | `dish_values.price(dish)` | — | 读取；失败视为数据不变式错误 → `Faulted` |
| 4 | `capacity.can_fit(effective)` | `CAPACITY` | 等号允许；边界归 ADR-0012 |
| 5 | `turns.can_spend_turn()` | `NO_TURNS` | 纯读取；`false` 只可能是 `turns_remaining = 0` |

全部通过后**原子提交**（顺序固定，任何一步出现「不应失败」的错误都进入 `Faulted`，不尝试部分回滚）：

1. 追加 `EatenEntry {dish_id, extra_satiety, effective_satiety}`（按提交顺序）；
2. `cloying.record_eat(dish)` —— 更新组计数一次（ADR-0014）；
3. `menu_waves.consume(dish_id) → ADVANCED | NO_EATABLE | NONE`；`NONE` 视为不变式错误 → `Faulted`；`NO_EATABLE` 置 `no_eatable = true`；
4. `turns.spend_turn()` —— 必须返回 `SPENT`，否则 `Faulted`；
5. `capacity.consume(effective)` —— 必须返回 `CONSUMED`，否则 `Faulted`；
6. `running_net += price`；`shortfall = max(0, −running_net)`；
7. `action_seq += 1`；
8. 广播 `action_committed(EAT, {dish_id, effective_satiety, extra_satiety})`；
9. 执行一次结束判定（决策 6）。

**必须**：`extra_satiety` / `effective_satiety` 使用第 2 步读到的值，不重算、不按菜名重放。第 3 步的 `consume` 与第 5 步的 `capacity.consume` 在单线程下不会因并发失败；它们的失败只可能来自本系统自己的重复调用，因此按 `Faulted` 处理而不是回滚。

### 4. 跳波 `skip_wave`：先问波次，再问回合

`skip_wave` 在最后一波等价于 `end_pickup`：`menu_waves.is_final_wave() == true` 时直接转入决策 5，不消耗回合。

非最后一波的校验顺序**固定**：

| # | 校验 | 失败码 | 说明 |
|---|---|---|---|
| 1 | `menu_waves.can_advance()` | `NO_EATABLE` | 当前波必须还能推进 |
| 2 | `turns.get_turns_remaining() ≥ 2` | `NO_TURNS` | 付费跳波要支付自己，并留至少 1 回合在新波吃菜 |

**必须**：第 2 步不能用 `can_spend_turn()` 代替。`can_spend_turn()` 在剩 1 回合时为 `true`，但允许该跳波会把玩家送进一波 0 回合的局；`≥ 2` 这条 +1 余量是 `skip_wave` 的动作语义，归 `RunFlow`，不归 `TimeTurns`（ADR-0011 决策 6）。

提交顺序固定：`menu_waves.advance()` → `turns.spend_turn()`（必须 `SPENT`）→ `action_seq += 1` → 广播 `action_committed(SKIP_WAVE, {})` → 结束判定。任一步「不应失败」的错误 → `Faulted`。

**必须**：拒绝发生在 `advance()` 与 `spend_turn()` **之前**——波次不推进、当前波剩余菜不丢弃、回合不变、日志不写、不播成功反馈。

### 5. 结束取餐 `end_pickup`：0 回合、只在最后一波

校验顺序：

| # | 校验 | 失败码 | 说明 |
|---|---|---|---|
| 1 | `menu_waves.is_final_wave()` | `NOT_FINAL_WAVE` | 非最后一波应走 `skip_wave` |
| 2 | `menu_waves.can_end_pickup()` | `NO_EATABLE` | 已无剩余菜时不得重复结束 |

提交：`menu_waves.end_pickup()` 必须返回 `ENDED` → `no_eatable = true` → `action_seq += 1` → 广播 `action_committed(END_PICKUP, {})` → 结束判定。**不调用** `spend_turn()`、**不改变**容量。

### 6. 结束判定：一个函数、两个时机、两种原因

```gdscript
func _judge_end() -> EndReason:
    if turns.get_turns_remaining() == 0:
        return EndReason.TIME_EXHAUSTED
    if no_eatable:
        return EndReason.NO_EATABLE_DISH
    return EndReason.NONE
```

- **时机 1**：初始化成功、广播 `run_started` 后立即执行一次（防御 `turns_total = 0`，正常数据不可达；命中则生成零行 `RunLog` 并正常结算）。
- **时机 2**：每次成功提交后执行一次。被拒绝的动作、暂停帧、纯展示帧都不判定。
- 命中后同一帧：`run_state = ENDING`、冻结动作、广播 `run_ended(run_id, reason)`；此后 `submit_action` 一律返回 `NOT_ACTIVE`。
- `TIME_EXHAUSTED` 优先于 `NO_EATABLE_DISH`，顺序写死在函数里，不依赖事件到达顺序。
- 本系统**不**产生第三种原因；容量满、可见菜放不下都不触发结束。

`no_eatable` 的来源只有三个：`menu_waves.consume(dish_id) == NO_EATABLE`、`menu_waves.end_pickup() == ENDED`、菜单波次在最后一波被吃空后发出的“无剩余菜”通知。它一旦为 `true` 不回落；可见菜因容量放不下**不**设置它。

### 7. 冻结与 `Ending` 之后

- `Ending`、`Settling`、`Handoff`、`Failed`、`Faulted` 与 `frozen = true` 下，`submit_action` 全部被拒绝；吃菜、跳波、结束取餐都不再有副作用。
- `Ending` 后不再调用 `spend_turn()` / `consume()` / `advance()` / `end_pickup()`；剩余工作只有组装 `RunLog` 与交接（ADR-0007）。
- `Ending` 不撤销已提交的动作；最后一帧的读数就是 `RunLog` 的读数。

### 8. 动作管线图

```
InputRouter / HUD
      │  submit_action(action, dish_id)
      ▼
 ┌──────────────────────────────────────────────┐
 │ 公共前置：frozen? → FROZEN                    │
 │           state != ACTIVE? → NOT_ACTIVE      │
 ├──────────────┬───────────────┬───────────────┤
 │ eat          │ skip_wave     │ end_pickup    │
 │ 可见集        │ can_advance   │ is_final      │
 │ 腻度 e/extra │ remaining ≥ 2 │ can_end       │
 │ price        │               │               │
 │ can_fit      │               │               │
 │ can_spend    │               │               │
 ├──────────────┴───────────────┴───────────────┤
 │ 原子提交 → action_seq++ → action_committed    │
 │           → 结束判定 → Ending? run_ended      │
 └──────────────────────────────────────────────┘
```

### 9. 只读可用性投影 `action_availability()`

HUD 需要知道跳过/结束取餐**此刻是否可用、为什么不可用**，但不得复制本 ADR 的门控阈值。`RunFlow` 因此发布一个无副作用的只读投影：

```gdscript
class ActionAvailability extends RefCounted
var skip_visible: bool          # 非最后一波且当前波可推进
var skip_enabled: bool
var skip_reason: int            # RejectCode 的只读映射；可用时 NONE
var end_visible: bool           # 最后一波
var end_enabled: bool
var end_reason: int
```

- 判定顺序与本 ADR 决策 4 / 5 的校验表**同源**：`skip_wave` 依次问 `is_final_wave()`、`can_advance()`、`turns_remaining ≥ 2`；`end_pickup` 依次问 `is_final_wave()`、`can_end_pickup()`。
- 它是投影，不提交、不 `advance()`、不 `spend_turn()`、不 `action_seq += 1`、不发信号；调用次数不受限。
- `frozen = true` 时两个 `*_enabled` 均为 `false`，`*_reason` 为 `FROZEN`；`run_state != Active` 时为 `NOT_ACTIVE`。
- HUD 只格式化 `*_reason` 与代价文案，不得自行比较 `turns_remaining ≥ 2` 或 `is_final_wave`（ADR-0019 D3）。

## Alternatives Considered

### Alternative 1: 边校验边提交

- **Description**: 先扣回合，再检查容量；或先移出菜单，再检查回合。
- **Pros**: 代码短。
- **Cons**: 失败会留下半提交；「容量不足」会先扣回合；日志与账本不一致。
- **Rejection Reason**: GDD 要求「先完整校验再原子提交」，被拒绝的动作零副作用。

### Alternative 2: 用 `can_spend_turn()` 直接门控付费跳波

- **Description**: `turns_remaining = 1` 时允许跳波，因为 `can_spend_turn()` 为 `true`。
- **Pros**: 复用回合服务的通用判定。
- **Cons**: 玩家花掉最后一回合进入新波，0 回合可吃，等于零收益丢掉当前波；`TIME_EXHAUSTED` 与跳波成功反馈叠在同一帧。
- **Rejection Reason**: 动作需要 +1 余量；该规则归 `RunFlow`，不污染 `TimeTurns` 的通用语义。

### Alternative 3: 每个动作各自判定结束

- **Description**: 吃菜、跳波、结束取餐分别判断 `three_conditions`。
- **Pros**: 分支直观。
- **Cons**: 三份优先级实现必然漂移；同一帧可能广播两次 `run_ended`；`NO_EATABLE_DISH` 可能抢在 `TIME_EXHAUSTED` 之前。
- **Rejection Reason**: 结束裁决必须只有一个函数、两个时机。

### Alternative 4: 把结束判定做成每帧轮询

- **Description**: 在 `_process` 里每帧检查回合与菜，命中就结束。
- **Pros**: 不依赖动作触发。
- **Cons**: 暂停、结算动画、纯展示帧都会触发；帧率影响行为；与「每次成功提交后判定一次」冲突。
- **Rejection Reason**: 判定时机是事件驱动，不是帧驱动。

### Alternative 5: 在 `Ending` 里撤销最后一次动作

- **Description**: 如果结束判定命中，回滚刚提交的吃菜。
- **Pros**: 收据与「最后一帧」看起来更整齐。
- **Cons**: 玩家吃下的最后一口凭空消失；`turns_used` 与 `capacity_used` 与动作历史不符。
- **Rejection Reason**: 结束即冻结，不撤销已提交的动作。

### Alternative 6: 把 `skip_wave` 与 `end_pickup` 合成一个动作

- **Description**: 只保留 `skip_wave`，最后一波自动解释为结束取餐。
- **Pros**: 动作集更小。
- **Cons**: 输入层与 HUD 无法区分「付费跳波」与「0 回合结束」的反馈；`NO_TURNS` 与结束取餐的按钮禁用条件不同。
- **Rejection Reason**: GDD 明确三种动作与各自代价；本 ADR 保留三个枚举，但在 `skip_wave` 的最后一波分支内部复用 `end_pickup` 语义。

## Consequences

### Positive

- 每种动作只有一条写死的校验链，拒绝原因可归因、可测试。
- 原子提交消除了半扣回合、半写日志这一类最难复现的 bug。
- 结束原因只有一个裁决点，同帧冲突由代码顺序而非事件顺序决定。
- 付费跳波的 +1 余量留在动作层，回合服务保持通用、干净。
- `action_seq` 与 `eaten` 同源，HUD 与 `RunLog` 可以逐条对齐。

### Negative

- 校验与提交步骤较多，`eat` 的提交序列需要严格的单元测试与故障注入。
- `RunFlow` 必须依赖五个子系统的具体返回码；任一子系统改词表都要同步本 ADR。
- 「同帧第二个动作」不在 `RunFlow` 内部拦截，依赖输入层的单动作契约；边界测试必须覆盖。

## Risks

- **提交序列中途失败**：按 `Faulted` 处理，不尝试回滚；这是 ADR-0007 的失败哲学，避免「看起来正常的半局」。
- **`no_eatable` 被容量误设**：契约明确容量满不设置；HUD 的「吃不下」提示不是 `no_eatable`。
- **`skip_wave` 在最后一波被误判**：`is_final_wave()` 由菜单波次拥有；本系统只读，不自己数波。
- **输入层同帧投递两个动作**：由 ADR-0005 的 `consume_action` + 首个消费者获胜保证；集成测试用双投递夹具验证。
- **`price` 读取失败发生在提交中途**：把 `price` 放进校验阶段（第 3 步），失败时不进入提交。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `run-flow.md` | TR-run-flow-003：`submit_action(eat/skip_wave/end_pickup, dish_id)` 固定校验顺序、原子提交；非最后一波在 `turns_remaining = 1` 时于任何回合/波次变更前返回 `NO_TURNS` | 决策 1–5；逐动作校验表与提交序列，`skip_wave` 的 `≥ 2` 门控 |
| `run-flow.md` | TR-run-flow-004：只有 `TIME_EXHAUSTED` / `NO_EATABLE_DISH` 两种结束原因；固定优先级；`Ending` 后冻结 | 决策 6、7；单一判定函数、两个时机、`TIME_EXHAUSTED` 优先、`Ending` 拒绝一切动作 |
| `run-flow.md` | Core Rule 5：一次输入至多一个动作；`REJECTED(code)` 至少覆盖 `STALE_FOCUS` / `CAPACITY` / `NO_TURNS` / `FROZEN` | 决策 1、2；封闭 `RejectCode` 词表 |
| `run-flow.md` | Core Rule 6：吃菜四道闸门与固定提交顺序 | 决策 3；校验表 + 八步提交 |
| `run-flow.md` | Core Rule 7 / Edge Case：非最后一波付费跳波的 `NO_TURNS` 前置于 `advance()` / `spend_turn()` | 决策 4；`turns_remaining ≥ 2` |
| `run-flow.md` | Core Rule 9 / 10：初始化后与每次成功提交后各判定一次；同帧取 `TIME_EXHAUSTED` | 决策 6 |
| `run-flow.md` | Core Rule 11：结束即冻结 | 决策 7 |
| `run-flow.md` | Core Rule 12：`EatenEntry` 的 `extra_satiety` / `effective_satiety` 来自吃下时刻 | 决策 3；第 2 步读取、第 1 步写入，不重算 |
| `run-flow.md` | AC 6–12、36、39、40：吃菜/跳波/结束取餐/过期焦点/容量满不结束 | 决策 3–6 |
| `run-flow.md` | AC 11a：`turns_remaining = 1` 的付费跳波零副作用 | 决策 4；拒绝先于任何变更 |
| `menu-waves.md` | `advance()` 只承载付费跳波；`end_pickup()` 只在最后一波 | 决策 4、5；本系统只调用，不重定义波次 |
| `time-and-turns.md` | Core Rule 13：`turns_remaining = 1` 时拒绝非最后一波付费跳波 | 决策 4；阈值读取回合服务的读数，判断归本系统 |

## Performance Implications

- **CPU**：一次动作 O(1)，无循环、无分配；`eat` 的九步提交在单帧内完成，远低于 1 ms。
- **Memory**：`eaten` 数组按局增长（最多 `turns_total` 条），MVP 上限 10 条；无缓存。
- **Load Time**：不参与启动门。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 定义 `RunAction` / `RejectCode` / `ActionResult` 与前置条件，跑 AC 20（冻结）与 AC 40（过期焦点）。
2. 实现 `eat` 的校验链（先用子系统替身），跑 AC 6–9、34、40。
3. 实现 `eat` 的原子提交与 `running_net`，跑 AC 6、14、26、34。
4. 实现 `skip_wave`（含 `≥ 2` 门控）与 `end_pickup`，跑 AC 10、11、11a、26。
5. 实现 `_judge_end` 与 `Ending` 冻结，跑 AC 12、13、27、36、39。
6. 接入真实的 `TimeTurns` / `StomachCapacity` / `MenuWaves` / `CloyingAndTaste` / `DishValues`（各自 ADR 落地后）。

## Validation Criteria

- GDD AC 6–13、20、26、27、34、36、39、40 全部通过。
- 故障注入：在提交序列的每一步注入「不应失败」的错误，断言进入 `Faulted` 且不产生部分 `RunLog`。
- 零副作用：对每个 `RejectCode` 各一条测试，断言 `turns_used`、`capacity_used`、`eaten`、`running_net`、`action_seq` 与 `run_state` 全部不变，且 `action_committed` 未广播。
- 单动作契约：双投递夹具在输入层被拦截；`RunFlow` 收到两次合法调用时按两次独立动作处理（不静默去重）。
- 结束唯一性：同一局 `run_ended` 恰好一次；同帧双条件时 `end_reason = TIME_EXHAUSTED`。

## Related

- ADR-0004 服务组合与信号 — `RunFlow` 的构造与 `action_committed` 派发
- ADR-0005 输入上下文栈 — 「一次输入一个动作」与首个消费者获胜
- ADR-0007 局生命周期与结算事务 — 状态枚举、`RunLog`、`run_ended` / `run_settled`
- ADR-0010 菜品数值契约 — 提交时的 `price`
- ADR-0011 回合预算与耗尽判定 — `can_spend_turn` / `spend_turn` / `is_exhausted`
- ADR-0012 胃容量账本 — `can_fit` / `consume`
- ADR-0014 腻度与口味 — `effective_satiety` / `record_eat`
- ADR-0015 结算纯函数 — `RunLog` 的消费方
- ADR-0019 HUD 数据绑定 — `ActionResult` 与快照
- `design/gdd/run-flow.md`
- `design/gdd/menu-waves.md`
