# ADR-0011: 回合预算、扣减所有权与耗尽判定

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

《回本》的时间是一叠离散的格子，不是一只走的钟。本 ADR 把 `time-and-turns.md` 固定成一套**唯一的运行时实现契约**：`TimeTurns` 是 `RefCounted` 纯服务，只持有一个开局冻结的整数预算与三个计数器；`initialize(turn_budget)` 每局恰好一次，非法值返回 `INVALID_TURN_BUDGET` 而不钳制、不默认；`spend_turn()` 的代价恒为 1，`can_spend_turn()` 是无副作用的纯读取；`is_exhausted()` 只报告事实，结束原因与优先级归 `RunFlow`。它不读数据目录、不读墙钟、不发信号，也不存在「加回合」「退回合」「多回合动作」的入口。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Core |
| **Layer** | Core |
| **Knowledge Risk** | LOW — 只用 `int` / `bool` / `StringName` 与纯 GDScript 整数运算，不依赖任何 4.5–4.7 新 API |
| **References Consulted** | `design/gdd/time-and-turns.md`、`design/gdd/run-flow.md`、`design/gdd/menu-waves.md`、`docs/architecture/architecture.md`、`docs/engine-reference/godot/VERSION.md`、`current-best-practices.md`、`deprecated-apis.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① `turns_total = INT32_MAX` 时第 N 次扣减不溢出（全部运算在 int64 域）；② `turns_remaining` 从 1 扣到 0 后 `is_exhausted()` 为 `true`，第 11 次调用返回 `REJECTED_EXHAUSTED` 且三读数不变；③ `Uninitialized` 下三个读数均为 0、`is_exhausted()` 为 `false`；④ 连续 60 s 无输入 / 暂停 / 失焦后读数不变（静态扫描确认全项目无 `Time.get_ticks_*` / `OS.get_ticks_*` 参与回合记账）；⑤ 同一帧内 `RunFlow` 提交后读取 `turns_remaining` 与 `HudFrame` 快照值相等 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述五条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0004（服务组合）— `TimeTurns` 由 `AppRoot` 在 CONSTRUCT 阶段构造并按引用注入 `RunFlow`；ADR-0007（局生命周期）— 一局的开始、动作提交与 `run_ended` 由 `RunFlow` 定义，本服务只在被调用时响应 |
| **Enables** | ADR-0013（局内动作提交）的回合校验与结束判定；ADR-0015（结算）的 `turns_total` / `turns_used` 字段；ADR-0019（HUD）的回合只读绑定；ADR-0020（首局引导）的回合节奏演示 |
| **Blocks** | 时间与回合系统的全部实现 story；任何「还能吃几口」相关的验收（`run-flow` AC 12 / 13、`menu-waves` 跳波、`settlement` 收据） |
| **Ordering Note** | 本 ADR 不决定哪些动作花回合、也不决定结束时机——前者是 ADR-0013（`RunFlow`），后者是 `run-flow.md` 的结束判定。本 ADR 只规定「拿到一个预算之后如何扣、如何读、如何报耗尽」 |

## Context

### Problem Statement

`time-and-turns.md` 已经把语义写死，但没有回答实现层必须回答的四件事：

1. **预算是谁的值、什么时候冻。** 如果 `TimeTurns` 每次扣减都去查 `StationData.turn_budget`，热重载或调试改数据会让玩家刚看到的剩余格数变成谎话；如果每个动作各自记一份剩余数，回合总数会出现多个副本。
2. **「能不能扣」与「扣」是否同一个调用。** 如果只提供有副作用的 `spend_turn()`，调用方只能用「试着扣一下」来判断，失败会悄悄消耗或破坏状态；GDD 明确要求 `can_spend_turn()` 是纯读取。
3. **耗尽之后谁说了算。** 回合归零只是事实，不是结局；`TIME_EXHAUSTED` 与 `NO_EATABLE_DISH` 的优先级必须只有一个所有者，否则同一帧两个系统各写一次 `end_reason`。
4. **非法预算怎么失败。** 如果 `turn_budget < 1` 被钳到 1 或默认成 10，漏配字段的档口会带着错误节奏上线，而错误看起来像平衡问题。

### Current State

`src/` 只有占位文件，没有任何实现。`architecture.md` 已把 `TimeTurns` 列为 Core 层的纯 `RefCounted` 服务，并声明数据由 `RunFlow` 解析后以整数传入。缺的是可被 story 直接引用的 API 表面、状态机与错误词表。

### Constraints

- **离散、非实时**：没有秒、没有小数、没有内部计时器、没有按帧推进；唯一扣减来源是 `spend_turn()` 调用。
- **开局冻结**：`turns_total` 在 `initialize()` 之后不再改变；数据变更新局才生效。
- **唯一调用者**：只有 `RunFlow` 调用 `initialize()` 与 `spend_turn()`；`MenuWaves`、HUD、音效只读或只发意图。
- **不可透支、不返还**：`turns_remaining ∈ [0, turns_total]` 恒成立；没有负数、没有补时、没有退款。
- **不发信号**：状态变化经 `RunFlow` 的 `action_committed` / `run_ended` 后可观察，本服务不产生第二事实源。
- **不查数据目录**：本服务只接收已解析的 `int`，不持有 `DataConfig` 引用。
- **单人单线程**：MVP 无并发调用；不需要锁。

### Requirements

- 必须提供 `initialize(turn_budget) → turns_total | INVALID_TURN_BUDGET`，每局恰好一次，非法输入不钳制、不默认。
- 必须提供 `can_spend_turn()` 纯读取与 `spend_turn()` 提交，两者语义分离。
- 必须保证 `spend_turn()` 代价恒为 1，且成功后 `turns_used = turns_total − turns_remaining`。
- 必须只报告 `is_exhausted()`，不保存、不排序结束原因。
- 必须让预算在一局内冻结，下一局重新读取。
- 必须让未初始化与已耗尽状态可区分、可断言，且都不发生隐式写入。
- 必须让 `turns_used` 的恒等式（吃菜数 + 付费跳波数）可在集成测试中核对。

## Decision

### 1. 模块形态：`TimeTurns` 是有界的纯服务

`TimeTurns` 是 `RefCounted`，位于 Core 层，由 `AppRoot` 在 CONSTRUCT 阶段构造，**没有构造参数**。它：

- 只持有 `state`、`turns_total`、`turns_remaining`、`turns_used` 四个字段；
- 不持有 `DataConfig`、`RunFlow`、`MenuWaves` 或 `Save` 的引用；
- 不发射信号、不注册 `_process` / `_input`、不读墙钟、不写存档、不播放音效；
- 不提供任何「加回合」「退回合」「设置剩余」的公开方法。

### 2. 初始化契约：一次、冻结、不兜底

```gdscript
enum State { UNINITIALIZED, ACTIVE, EXHAUSTED }
enum InitStatus { OK, INVALID_TURN_BUDGET }

class InitError:
    var caller: StringName        # 例如 &"run_flow"
    var reason: InitStatus        # 当前只有 INVALID_TURN_BUDGET
    var received: Variant         # 原样带回非法输入，便于日志定位

class InitResult:
    var ok: bool
    var turns_total: int          # ok = true 时为 turns_total
    var error: InitError          # ok = false 时非空
```

`initialize(turn_budget: int) -> InitResult`：

- 成功：`turns_total = turn_budget`、`turns_remaining = turn_budget`、`turns_used = 0`、`state = ACTIVE`、返回 `ok = true`；
- 失败（`turn_budget < 1`）：返回 `ok = false`、`reason = INVALID_TURN_BUDGET`，**状态与三读数完全不变**；如果此前是 `ACTIVE` 或 `EXHAUSTED`，保持原值，不建立 0 回合的局；
- 可以从任意状态调用，效果是**完整替换**上一局：旧值一个都不延续。

**必须**：`turn_budget` 的类型与存在性由 `RunFlow` 在解析 `StationData` 时校验；`TimeTurns` 只接受 `int` 形参，缺字段 / 非整数在到达本服务之前就应被映射成同一个 `INVALID_TURN_BUDGET`。**不存在「缺省 10」或「钳到 1」的静默兜底。**

**必须**：`turn_budget = 1` 合法；`turn_budget = 1000` 也合法（数据合法，可玩性由设计期审计标记，不由运行时拒绝）。

### 3. 消耗契约：读与写分离

```gdscript
enum SpendStatus { SPENT, REJECTED_EXHAUSTED, NOT_INITIALIZED }

func can_spend_turn() -> bool      # 纯读取：state == ACTIVE 且 turns_remaining ≥ 1
func spend_turn() -> SpendStatus
```

- `can_spend_turn()` 无任何副作用，可每帧调用，不改变状态、不写日志。
- `spend_turn()` 的代价**恒为 1**，不接受参数；没有多回合动作、没有折扣、没有返还。
- `state == UNINITIALIZED` → 返回 `NOT_INITIALIZED`，三读数仍为 0。
- `state == ACTIVE` 且 `turns_remaining = 0` → 返回 `REJECTED_EXHAUSTED`，状态与三读数完全不变。
- 成功：`turns_remaining -= 1`；`turns_used = turns_total − turns_remaining`（同一帧重算，不独立自增）；若 `turns_remaining = 0`，`state = EXHAUSTED`；返回 `SPENT`。

**必须**：不做去重。同一动作调用两次就扣两格——这是调用方的时序 bug，必须暴露成测试失败，而不是被服务悄悄吸收（GDD Edge Case「一次动作调用了两次 `spend_turn()`」）。

### 4. 耗尽只报告，不裁决

```gdscript
func is_exhausted() -> bool        # state != UNINITIALIZED 且 turns_remaining == 0
func get_state() -> State
func get_turns_total() -> int      # Uninitialized 下为 0
func get_turns_remaining() -> int  # Uninitialized 下为 0
func get_turns_used() -> int       # Uninitialized 下为 0
```

- `is_exhausted()` 只是 `turns_remaining = 0` 的可读封装；`UNINITIALIZED` 下恒为 `false`。
- 本服务**不保存** `end_reason`，**不排序** `TIME_EXHAUSTED` 与 `NO_EATABLE_DISH`，**不调用**任何结束流程。
- `EXHAUSTED` 是可观测终态：本局内没有离开路径，下一次 `initialize()` 才回到 `ACTIVE`。`RunFlow` 在观察到耗尽后同一帧进入 `Ending`，因此该状态通常只存在一帧，但必须可断言。
- 本服务**不发射** `turn_*` 信号；观察点只有 `RunFlow` 的 `action_committed` / `run_ended` 与随后生成的只读快照。

### 5. 预算冻结与读数窗口

- 开局后 `turns_total` 永久冻结；任何对 `StationData.turn_budget` 的修改（热重载、调试、未来的时间类升级）都不影响当前局，下一局 `initialize()` 才读取新值。
- `turns_remaining` / `turns_used` 只在 `ACTIVE` 与 `EXHAUSTED` 内有意义；`run_ended` 之后数值冻结，只允许 `RunLog` 与结算读取冻结副本。
- 局内 HUD 只在 `run_state ∈ {Active, Ending}` 时通过 `HudFrame` 快照读取，不直接持有本服务引用（见 ADR-0019）。
- 局外界面（菜单、升级、结算页）不得读取本服务的当前读数；结算只用 `RunLog` 里冻结的值。

### 6. 调用者纪律：只有 `RunFlow` 扣回合

| 调用者 | 允许的调用 | 禁止 |
|---|---|---|
| `RunFlow` | 开局 `initialize(turn_budget)` 一次；每个成功提交的 `eat` / 非最后一波 `skip_wave` 各 `spend_turn()` 一次；提交前 `can_spend_turn()` | 在一次提交里调用两次；对自动推进或 `end_pickup` 调用 |
| `MenuWaves` | 只发出跳波意图，由 `RunFlow` 扣除 | 直接调用 `spend_turn()` |
| 局内 HUD | 只读快照中的 `turns_remaining` / `turns_total` | 调用 `spend_turn()`；自行推算耗尽 |
| 音效反馈 | 从 `action_committed` / `run_ended` 派生 | 调用本服务 |

`turns_remaining = 1` 时拒绝非最后一波的付费跳波（`NO_TURNS`）**不是**本服务的规则：`RunFlow` 在调用 `spend_turn()` **之前**执行该判断（ADR-0013），本服务只提供 `turns_remaining` 读数，不加特例。最后一波「结束取餐」代价为 0，不经过 `spend_turn()`。

### 7. 不变量与防御

- 恒等式：`0 ≤ turns_remaining ≤ turns_total`、`turns_used = turns_total − turns_remaining`。
- 若观测到 `turns_used > turns_total` 或 `turns_remaining < 0`，视为内部状态损坏：返回结构化错误并停止扣减，**不钳回合法区间**（钳制会让越界变成一张看起来正常的收据）。
- 全部运算在 int64 域；`turn_budget` 来自 int32 数据字段，减法不会溢出。
- `initialize()` 中途重入视为新的一局并完整替换；`RunFlow` 只在开局调用，重入由 `run_id` 的集成判据发现，本服务不额外加锁。

### 8. 架构图

```
StationData.turn_budget (int)
        │  由 RunFlow 解析并校验
        ▼
RunFlow ── initialize(turn_budget) ──▶ TimeTurns (RefCounted)
   │  ◀── turns_total ────────────────┘
   │
   ├─ can_spend_turn() ──▶ bool（纯读取）
   ├─ spend_turn() ─────▶ SPENT / REJECTED_EXHAUSTED / NOT_INITIALIZED
   ├─ is_exhausted() ───▶ bool（只报告）
   └─ 组装 RunLog（turns_total / turns_used）
              │
              ▼
        HudFrame（同一帧只读快照） ──▶ 局内 HUD
```

## Alternatives Considered

### Alternative 1: 按真实时间扣格

- **Description**: 用 `_process` 或计时器每秒 / 每 N 秒扣一格。
- **Pros**: 「时间在走」的字面直觉更强。
- **Cons**: 直接违反支柱 3——压力来自反应速度而不是算术；暂停、失焦与低帧率会改变玩法；结算无法复现。
- **Rejection Reason**: GDD 把「等待、思考、暂停不消耗回合」写成硬边界；任何按秒记账的实现都算违反。

### Alternative 2: `TimeTurns` 自己查 `StationData.turn_budget`

- **Description**: 开局时由本服务直接读数据目录。
- **Pros**: 调用方少传一个参数。
- **Cons**: 新增一条运行时依赖边；热重载与调试语义要重新定义；数据层与 Core 逻辑耦合。
- **Rejection Reason**: 预算只是一个整数，由 `RunFlow` 解析后传入即可；数据所有权留给 `DataConfig`。

### Alternative 3: 非法预算钳到 1 或默认 10

- **Description**: `turn_budget < 1` 时静默修正。
- **Pros**: 游戏总能开始。
- **Cons**: 漏配字段的档口带着错误节奏上线；错误从「数据错误」伪装成「平衡问题」。
- **Rejection Reason**: GDD 明确 `INVALID_TURN_BUDGET` 且不钳制；`RunFlow` 映射为 `init_failed`。

### Alternative 4: 让 `TimeTurns` 保存 `end_reason` 并排序

- **Description**: 回合归零时由本服务写 `TIME_EXHAUSTED`，并处理与 `NO_EATABLE_DISH` 的优先级。
- **Pros**: 结束逻辑离触发器更近。
- **Cons**: 结束原因是局级状态，归 `RunFlow` 所有的 `RunLog`；两个系统写同一字段必然有一处漂移；同时满足两条件时无法保证唯一裁决者。
- **Rejection Reason**: 单一所有权原则；本服务只报告事实。

### Alternative 5: 用信号广播回合变化

- **Description**: `spend_turn()` 后发 `turn_changed(remaining)`。
- **Pros**: HUD 无需轮询。
- **Cons**: 多一个可能与快照不同龄的事实源；同一帧内信号顺序与快照生成顺序需要额外约定。
- **Rejection Reason**: 回合变化已由 `action_committed` 与快照覆盖；GDD 明确「不发信号」。

### Alternative 6: 给 `spend_turn(cost)` 加代价参数

- **Description**: 让动作自己传代价，为未来的「多回合动作」留口。
- **Pros**: 扩展性看起来更好。
- **Cons**: 代价不再是常量，折扣、返还与负代价都可以被表达；`turns_used` 恒等式失去结构性保证。
- **Rejection Reason**: 当前内容没有多回合动作；未来的时间类升级必须在开局冻结进预算（见 GDD Open Question 3），而不是局中改代价。

### Alternative 7: 用「试扣 + 回滚」代替 `can_spend_turn()`

- **Description**: 只提供 `spend_turn()`，失败时回滚。
- **Pros**: API 更少。
- **Cons**: 判断变成有副作用；回滚路径本身需要测试；调用方容易在失败后继续执行。
- **Rejection Reason**: GDD 要求「能不能」是可测试的纯读取。

## Consequences

### Positive

- 回合只有一个计数器、一个所有者，`turns_used` 恒等式可在集成测试里直接核对。
- 没有墙钟依赖，暂停、失焦、掉帧都不改变玩法，回放与结算可复现。
- 非法预算立即失败，内容错误不会被伪装成平衡问题。
- `can_spend_turn()` / `spend_turn()` 分离让调用时序可断言，重复提交会暴露成测试失败。
- 结束原因只由 `RunFlow` 裁决，同一帧不会出现两个 `end_reason`。

### Negative

- `RunFlow` 必须承担「哪些动作花回合」的全部映射，改动动作集时要同步改它，而不是改本服务。
- 没有「加回合」入口，未来的时间类升级必须以新字段进入 `upgrade_snapshot` 并在开局冻结，不能局中补时。
- 只有状态与读数，没有事件；任何想「听回合变化」的新系统必须改为读快照。

## Risks

- **`RunFlow` 重复调用 `spend_turn()`**：一次提交扣两格。用集成 AC（第 7、8 条）与 `RunLog` 恒等式拦截；本服务不去重，避免掩盖真正的重复提交。
- **局中重复 `initialize()`**：会静默重置一局。由 `run_id` 变化与 `RunFlow` 生命周期测试发现；本服务不加锁。
- **HUD 自行推算耗尽**：可能与真实读数差一帧。契约规定 HUD 只读 `HudFrame`，`is_exhausted` 不暴露给表现层做推断。
- **`turns_remaining = 1` 的付费跳波被误放行**：这不是本服务的防御点，必须在 ADR-0013 与 `menu-waves` / `ui` 的按钮禁用上同时落实。
- **未来放宽 `turn_budget` 类型**：若改成 int64 或浮点，`turns_used` 的整数恒等式与溢出分析都要重做；改 schema 前必须先改本 ADR。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `time-and-turns.md` | TR-time-and-turns-001：`initialize(turn_budget)` 每局一次；非法输入返回 `INVALID_TURN_BUDGET`，不钳制 | 决策 2；唯一初始化入口，失败保持原状态，无默认值 |
| `time-and-turns.md` | TR-time-and-turns-002：`spend_turn()` 代价恒为 1；`can_spend_turn()` 纯读取；不可透支、不返还 | 决策 3；读与写分离，无参数代价，`REJECTED_EXHAUSTED` 不改变状态 |
| `time-and-turns.md` | TR-time-and-turns-003：`is_exhausted()` 只报告；结束原因归 `run-flow` | 决策 4；不保存 `end_reason`、不排序、不发信号 |
| `time-and-turns.md` | TR-time-and-turns-004：预算在本局内冻结，下一局 `initialize` 才生效 | 决策 2、5；`turns_total` 开局后只读 |
| `time-and-turns.md` | Core Rule 2 / 11：回合是离散单位，没有秒、没有计时器、不发信号 | 决策 1、3、4；服务无 `_process`、无墙钟、无 `turn_*` 信号 |
| `time-and-turns.md` | Core Rule 6：唯一调用者是餐局流程；跳波由 `RunFlow` 代扣 | 决策 6；调用者表明确禁止 `MenuWaves` / HUD 直接扣减 |
| `time-and-turns.md` | Core Rule 12：`run_ended` 之后只读，局外界面不得读当前读数 | 决策 5；冻结值只经 `RunLog` 交付结算 |
| `time-and-turns.md` | Core Rule 13 / AC 23：`turns_remaining = 1` 时拒绝非最后一波付费跳波 | 决策 6；该判断由 `RunFlow` 在 `spend_turn()` 之前执行，本服务只提供读数 |
| `time-and-turns.md` | Formulas：`turns_used = turns_total − turns_remaining = 吃菜数 + 付费跳波数` | 决策 3、7；每次扣减后重算 `turns_used`，不独立自增 |
| `time-and-turns.md` | AC 1–10：初始化、非法预算、未初始化、耗尽与 `turn_budget = 1` | 决策 2、3、4；状态机与返回码逐条对应 |
| `time-and-turns.md` | AC 12–14：局中改数据不影响当前局；重复初始化完整替换；0 代价动作不扣回合 | 决策 2、5、6 |
| `time-and-turns.md` | AC 15–17 / 21：结束冻结、`turns_used` 恒等式、耗尽优先、超预算不拒绝 | 决策 4、5、7；只报告耗尽，不裁决优先级 |
| `run-flow.md` | Core Rule 8 / 10：只有两个结束原因，`TIME_EXHAUSTED` 优先 | 本服务不参与裁决；`RunFlow` 读取 `is_exhausted()` 后自行排序（ADR-0013） |

## Performance Implications

- **CPU**：每次查询与扣减 O(1)，单次 < 1 µs；无循环、无分配。
- **Memory**：四个字段，常驻开销可忽略；无缓存、无数组。
- **Load Time**：不参与启动门；`initialize` 在 `Menu → Run` 时同步执行，耗时纳秒级。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 先定义 `State` / `InitStatus` / `SpendStatus` / `InitError` / `InitResult` 与单元测试骨架。
2. 实现 `initialize` 的成功与失败路径，跑 GDD AC 1–3、13。
3. 实现 `can_spend_turn` / `spend_turn` / `turns_used` 重算，跑 AC 4–7、9。
4. 实现 `is_exhausted` 与未初始化读数，跑 AC 5、6、10。
5. 接入 `RunFlow` 的开局与动作提交（ADR-0013 落地后），跑 AC 8、12、14–23。

## Validation Criteria

- GDD AC 1–23 全部通过，其中 AC 2、8、14、16、17、18、19、20、23 需要 `RunFlow` / `MenuWaves` / 结算的测试替身。
- 静态扫描：全项目无 `Time.get_ticks_*` / `OS.get_ticks_*` 参与回合记账。
- 不变量测试：随机序列 10,000 次 `spend_turn()` 后恒等式始终成立，且越过 0 的调用全部返回 `REJECTED_EXHAUSTED`。
- 集成测试：吃 4 道 + 付费跳 1 波 → `turns_used = 5`；自动推进与 `end_pickup` 不改变任何读数。

## Related

- ADR-0004 服务组合与信号 — `AppRoot` 构造并注入 `TimeTurns`
- ADR-0007 局生命周期与结算事务 — `run_id`、`run_ended` 与 `RunLog`
- ADR-0010 菜品数值契约 — 同一 Core 层纯服务形态
- ADR-0012 胃容量 — 并列的每局预算，互不调用
- ADR-0013 局内动作提交 — `spend_turn()` 的唯一调用点与 `NO_TURNS` 前置拒绝
- ADR-0015 结算纯函数 — 消费 `turns_total` / `turns_used`
- ADR-0019 HUD 数据绑定 — 回合的只读快照
- ADR-0020 首局引导 — 回合节奏演示
- `design/gdd/time-and-turns.md`
- `design/gdd/run-flow.md`
