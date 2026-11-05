# ADR-0012: 胃容量快照、容量消耗与耗尽判定

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

胃容量是本局唯一一份「还能装几格」的账本。本 ADR 把 `stomach-capacity.md` 固定成一套**唯一的运行时实现契约**：`StomachCapacity` 是 `RefCounted` 纯服务，构造时只接收已解析的基础容量 `C0`；`initialize(upgrade_snapshot)` 每局一次，从冻结快照取 `capacity_bonus`，缺失、负数、非整数或越界一律失败且不默认；`can_fit(e)` 是无副作用判定（等号允许），`consume(e)` 是唯一写入口且原子；`end_run(reason)` 是每条终止路径上的权威边界；没有「吃撑」状态、没有结束原因、没有信号、没有第二份计数器。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Core |
| **Layer** | Core |
| **Knowledge Risk** | LOW — 只用 `int` / `bool` / `Dictionary` / `StringName` 与纯 GDScript 整数运算，不依赖 4.5–4.7 新 API |
| **References Consulted** | `design/gdd/stomach-capacity.md`、`design/gdd/run-flow.md`、`design/gdd/meta-upgrades.md`、`design/gdd/cloying-and-taste.md`、`docs/architecture/architecture.md`、`docs/engine-reference/godot/VERSION.md`、`current-best-practices.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① `typeof(value) == TYPE_INT` 能区分 Godot 4.7 的 `int`（64 位）与 `float`，`1.5` 在快照校验中被拒绝而不是被截断；② `C0 = 1`、`capacity_bonus = 998` 时 `capacity_total = 999` 合法，`1000` 被拒绝且不写入；③ `capacity_used + e == capacity_total`（等号）时 `can_fit` / `consume` 都成功；④ `capacity_used` 在 10,000 次随机合法提交后始终落在 `[0, capacity_total]` 且单调不减；⑤ 暂停 / 冻结 / 失焦后读数逐字段不变（静态扫描确认无墙钟与随机数参与容量记账） |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述五条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0004（服务组合）— `StomachCapacity` 由 `AppRoot` 在 CONSTRUCT 阶段构造并注入 `RunFlow`；ADR-0007（局生命周期）— `run_id`、`end_run` 的终止路径与 `RunLog` 由该 ADR 的启动/结算模型定义 |
| **Enables** | ADR-0013（局内动作提交）的 `can_fit` / `consume` 校验；ADR-0015（结算）的 `capacity_total` / `capacity_used` 校验；ADR-0016（局外升级）的 `capacity_bonus` 消费方；ADR-0019（HUD）的容量只读绑定 |
| **Blocks** | 胃容量系统的全部实现 story；`run-flow` 的吃菜提交与 `RunLog` 不变式；结算的 `CAPACITY_MISMATCH` 校验 |
| **Ordering Note** | `UpgradeSnapshot` 的字典 schema 由 ADR-0016 冻结；本 ADR 只消费其中的 `capacity_bonus`，并把「缺失 / 负数 / 非整数」定义为本服务的失败条件。两者同时落地前，story 以本 ADR 的字段契约为准 |

## Context

### Problem Statement

`stomach-capacity.md` 已经写死了公式与边界，但没有回答实现层必须回答的四件事：

1. **谁是唯一账本。** 如果菜品卡、HUD 与结算各自判断「放不放得下」，玩家会同时看到三个不同的剩余格数；如果各系统各自维护计数器，`capacity_used = Σ effective_satiety` 这个结算不变式就永远无法保证。
2. **快照坏了怎么办。** `capacity_bonus` 缺失、为负、为 `1.5` 或为 `null` 时，如果默认成 0，玩家会带着一个错误的容量开局，而错误看起来像数值平衡问题。
3. **「吃不下」是不是结束条件。** 如果容量满触发结束，玩家会在还有回合时被强行结算；GDD 明确没有「吃撑」结束原因，容量只决定一道菜单次能不能吃。
4. **一局什么时候清零。** 结束、放弃、初始化失败、运行期故障四条路径都必须回到同一个清零点；只靠 `initialize` 重置会漏掉「从未成功开始」的失败路径。

### Current State

`src/` 只有占位文件，没有任何实现。`architecture.md` 已把 `StomachCapacity` 列为 Core 层纯 `RefCounted` 服务，并声明 `initialize(snapshot)` / `can_fit` / `consume` / `end_run` 四个入口。缺的是快照校验规则、状态机、错误词表与快照冻结语义。

### Constraints

- **只做加法**：`capacity_used` 一局内单调不减；没有退还、消化、释放或公共 setter。
- **整数格**：不存在小数或四舍五入；`effective_satiety` 由腻度给出且恒 ≥ 1。
- **不结束本局**：本服务不产生 `end_reason`，也没有「已满」状态；满了只让后续 `can_fit` 返回 `false`。
- **一局一个快照**：`capacity_total` 在 `initialize` 后冻结，升级只影响下一局。
- **唯一写入口**：`initialize`（置 0）与 `consume`（累加）是仅有的两个写方法。
- **不发信号**：变化经 `RunFlow` 的 `action_committed` 与只读快照观察。
- **不读菜品数据**：运行期只接收 `effective_satiety` 整数，不查 `DishData`、不按菜名重放。
- **单人单线程**：MVP 无并发；`can_fit` 与 `consume` 之间不需要预留机制。

### Requirements

- 必须每局恰好一次 `initialize(upgrade_snapshot)`，且在 `run_started` 之前完成。
- 必须从快照严格解析 `capacity_bonus`：缺失、负数、非整数、`null` 一律失败，不缺省为 0。
- 必须把 `capacity_total` 限制在 `[1, 999]`，越界失败且不钳制。
- 必须提供无副作用的 `can_fit(e)` 与原子的 `consume(e)`，等号边界允许。
- 必须让 `capacity_used` 单调不减，且拒绝 `e < 1` 或非整数，防止负值减少已用容量。
- 必须只暴露只读读数，不广播信号、不维护第二份计数器。
- 必须让 `end_run(reason)` 成为四条终止路径上的权威边界。
- 必须让「已满」只是派生判断，不是状态、不是结束原因。

## Decision

### 1. 模块形态与依赖注入

`StomachCapacity` 是 `RefCounted`，位于 Core 层：

- 构造参数只有 `base_capacity: int`（`C0`），由 `AppRoot` 在 `DataConfig Ready` 之后从 `BalanceData.base_capacity` 读出后传入；**本服务不持有 `DataConfig` 引用**，也不在运行期查数据目录。
- 不持有 `RunFlow`、`CloyingAndTaste`、`Settlement` 或 `Save` 的引用；`effective_satiety` 由 `RunFlow` 作为整数参数传入。
- 不发射信号、不注册 `_process` / `_input`、不读墙钟、不写存档。
- 只持有 `state`、`capacity_total`、`capacity_used` 三个字段。

`C0` 在构造时校验；若 `C0 < 1` 或不是整数（绕过启动门直接注入），所有 `initialize` 返回 `CAPACITY_INIT_INVALID`，不建立账本。

### 2. 初始化契约：严格快照、一次、不兜底

```gdscript
enum State { UNINITIALIZED, READY }
enum InitStatus { OK, CAPACITY_INIT_INVALID, CAPACITY_TOTAL_OUT_OF_RANGE }

class InitError:
    var caller: StringName        # 例如 &"run_flow"
    var reason: InitStatus
    var field: StringName         # &"capacity_bonus" / &"base_capacity" / &""
    var received: Variant

class InitResult:
    var ok: bool
    var capacity_total: int       # ok = true 时有效
    var error: InitError          # ok = false 时非空
```

`initialize(snapshot: Variant) -> InitResult`：

1. 若当前 `state == READY`：返回 `CAPACITY_INIT_INVALID`，**不改变任何字段**。同一局第二次初始化是契约违例，`RunFlow` 据此进入 `Faulted`；静默重置会抹掉已用容量。
2. 校验 `snapshot` 是 `Dictionary` 且含 `capacity_bonus`，且 `typeof(capacity_bonus) == TYPE_INT`、`capacity_bonus ≥ 0`；否则返回 `CAPACITY_INIT_INVALID`，状态保持 `UNINITIALIZED`。
3. 校验 `C0` 为 `1–999` 的整数；否则 `CAPACITY_INIT_INVALID`。
4. 计算 `capacity_total = C0 + capacity_bonus`；`> 999` 时返回 `CAPACITY_TOTAL_OUT_OF_RANGE`，状态保持 `UNINITIALIZED`。
5. 成功：`capacity_total` 写入、`capacity_used = 0`、`state = READY`，返回 `ok = true`。

**必须**：`capacity_bonus` **缺失、`null`、负数、`1.5` 或字符串**都是失败——不取绝对值、不四舍五入、不默认 0。未知键忽略（前向兼容）。

**必须**：`snapshot` 的完整 schema 由 ADR-0016（局外升级）冻结；本服务只读 `capacity_bonus`，不读取 `cloying_resistance` / `menu_vision` / `levels`。

**必须**：一次失败的 `initialize` 不把状态留在任何「半初始化」值上；下一次 `initialize` 视为新局，不触发「同局第二次」判定（因为 `state` 仍是 `UNINITIALIZED`）。

### 3. 终止边界：`end_run(reason)` 是唯一清零点

```gdscript
enum EndReason { ENDED, DISCARDED, FAILED, FAULTED }   # 与 RunFlow 共用

func end_run(reason: EndReason) -> void
```

- `RunFlow` 在**每一条**终止路径调用一次：正常结算交接（`ENDED`）、放弃/退出（`DISCARDED`）、初始化失败（`FAILED`）、运行期故障（`FAULTED`）。
- 收到后无论当前状态如何，都回到 `UNINITIALIZED`：`capacity_total = 0`、`capacity_used = 0`。
- 幂等：重复调用只是再次归零，不报错、不产生结算、不写纪录。
- `end_run` 不区分原因的副作用——四条路径都只丢弃账本；原因只用于日志与 `RunFlow` 的分支。
- 本服务不订阅 `run_ended` / `run_discarded` / `run_faulted` 信号；`end_run` 是权威边界，信号订阅无法覆盖「从未成功开始」的 `FAILED` 路径。

### 4. 可吃性判定：纯读取、等号允许

```gdscript
func can_fit(effective_satiety: Variant) -> bool
```

- `can_fit(e) = (typeof(e) == TYPE_INT ∧ e ≥ 1) ∧ (capacity_used + e ≤ capacity_total)`。
- **等号允许**：`capacity_used + e == capacity_total` 返回 `true`；改严格小于会让 104 元最优餐正好占满 20 格变得不可达。
- `e < 1` 或非整数：返回 `false` 并记一条 ERROR——正常流程不可达（腻度保证 `e ≥ 1`），这是防御性契约，绝不放行「免费菜」。
- `e > capacity_total`：合法输入，返回 `false` 且**不记错误**（这道菜整局都放不下）。
- `state == UNINITIALIZED`：返回 `false` 并记 ERROR。
- **无预留**：同一状态重复调用结果相同，不锁、不扣、不改变字段。

参数用 `Variant` 而非 `int` 是刻意的：GDD AC 10 要求 `can_fit(2.5)` 返回 `false` 并记 ERROR；GDScript 的 `int` 形参会在进入函数体前抛类型错误，无法满足「返回 false」的契约。本方法把类型校验放在函数体内。

### 5. 唯一写入口：原子的 `consume`

```gdscript
enum ConsumeStatus { CONSUMED, CAPACITY_OVERFLOW, INVALID_SATIETY, NOT_INITIALIZED }

func consume(effective_satiety: Variant) -> ConsumeStatus
```

- `state == UNINITIALIZED` → `NOT_INITIALIZED`，字段不变。
- `e < 1` 或非整数 → `INVALID_SATIETY`，字段不变。**绝不**允许 `consume(-1)` 减少已用容量。
- `capacity_used + e > capacity_total` → `CAPACITY_OVERFLOW`，字段不变，`RunFlow` 进入 `Faulted`；不钳到上限。
- 成功：`capacity_used += e`，返回 `CONSUMED`。
- 本方法不调用 `can_fit`，但执行同一套边界校验；调用方先 `can_fit` 再 `consume` 是契约，重复校验是防御而非替代。
- 本服务无法识别「跳过 `can_fit` 但数值仍放得下」的调用——那在账本上是合法的；重复提交由 `RunLog` 的 `capacity_used = Σ effective_satiety` 不变式与结算的 `CAPACITY_MISMATCH` 捕获。

### 6. 不结束本局、不发信号、无第二计数器

```gdscript
func get_capacity_total() -> int       # UNINITIALIZED 下为 0（记 ERROR）
func get_capacity_used() -> int        # UNINITIALIZED 下为 0（记 ERROR）
func get_capacity_remaining() -> int   # total − used；UNINITIALIZED 下为 0（记 ERROR）
```

- `capacity_used == capacity_total` 只是派生读数，不产生状态、不产生 `end_reason`、不调用结算；本局保持 `Active`。
- 没有 `STUFFED` 枚举，也没有 `full` 标志；「吃不下」是 `can_fit` 的返回值，由 HUD 展示，不是状态。
- 不广播 `capacity_changed`；容量变化经 `RunFlow` 的 `action_committed` 与只读快照观察。
- 下游不得自建计数器；`RunLog.capacity_used` 是 `RunFlow` 在动作提交时从本服务取得的冻结副本。
- 局外界面不得读取本服务当前读数；结算只读 `RunLog`。

### 7. 冻结与确定性

- 一局开始后 `capacity_total` 不随升级、`UpgradeData` 或快照改写而变化；下一局 `initialize` 才采用新值。
- 给定 `C0`、`capacity_bonus` 与按顺序提交的 `effective_satiety` 序列，所有读数与返回码完全确定；不读时钟、不用随机数、不依赖容器遍历顺序。
- 全部运算在 int64 域；`C0 + capacity_bonus ≤ 999`，不存在溢出。

### 8. 架构图

```
BalanceData.base_capacity (C0)
        │  AppRoot 在 Data Ready 后读出，构造时注入
        ▼
StomachCapacity (RefCounted)
   ▲   initialize(upgrade_snapshot) ── RunFlow（开局一次）
   │   end_run(reason) ─────────────── RunFlow（每条终止路径一次）
   │
   ├─ can_fit(e) ──▶ bool（纯读取）
   ├─ consume(e) ──▶ CONSUMED / CAPACITY_OVERFLOW / INVALID_SATIETY / NOT_INITIALIZED
   └─ capacity_total / capacity_used / capacity_remaining（只读）
              │
              ├──▶ RunLog（冻结副本）──▶ Settlement
              └──▶ HudFrame（同一帧快照）──▶ 局内 HUD
```

## Alternatives Considered

### Alternative 1: 让 HUD / 结算各自维护剩余容量

- **Description**: 每个消费方自己算 `total − Σ effective_satiety`。
- **Pros**: 无需共享服务。
- **Cons**: 三个不一致的剩余格数；结算无法判断谁是权威；重复提交无法被发现。
- **Rejection Reason**: 容量必须是单一账本；`capacity_used = Σ effective_satiety` 是结算不变式。

### Alternative 2: 非法快照默认成 0 加成

- **Description**: `capacity_bonus` 缺失或非法时按 0 处理。
- **Pros**: 游戏总能开始。
- **Cons**: 漏配升级或解析 bug 会伪装成「玩家没买升级」；永久无法开局的存档问题被推迟到别处爆发。
- **Rejection Reason**: GDD 明确 `CAPACITY_INIT_INVALID` 且不默认。

### Alternative 3: 容量满即结束本局

- **Description**: `consume` 后 `used == total` 时产生 `STUFFED` 结束原因。
- **Pros**: 玩家不会「有回合却吃不下」空转。
- **Cons**: 违反「容量只决定一道菜能不能吃」；玩家失去用剩余回合或跳波的权利；结算多出第三种原因。
- **Rejection Reason**: GDD 明确没有「吃撑」结束状态；空转问题回到 `run-flow` Open Question 9 讨论。

### Alternative 4: 提供 `refund` / `release` / `reset` 公共方法

- **Description**: 允许后续系统退还容量或手动清零。
- **Pros**: 为未来的消化机制留口。
- **Cons**: 破坏单调性与结算不变式；「已付的额外占用不退还」是腻度规则的一部分。
- **Rejection Reason**: 唯一清零点应是 `initialize` 与 `end_run`；未来加入消化必须重写本 ADR。

### Alternative 5: 用信号广播容量变化

- **Description**: `consume` 后发 `capacity_changed(used, total)`。
- **Pros**: HUD 无需轮询。
- **Cons**: 与快照形成两份事实源；同一帧的顺序需要额外约定。
- **Rejection Reason**: 变化已由 `action_committed` 与 `HudFrame` 覆盖；GDD 明确不发信号。

### Alternative 6: 让本服务直接查 `BalanceData` 或存档

- **Description**: 构造时传 `DataConfig`，开局读 `base_capacity`，或直接读升级等级。
- **Pros**: 减少调用方传参。
- **Cons**: 新增数据层依赖；等级→加成的换算会把局外升级的公式复制一份；快照冻结语义更难保证。
- **Rejection Reason**: `C0` 与 `capacity_bonus` 都是已解析整数；本服务只做加法。

### Alternative 7: 同局第二次 `initialize` 静默重置

- **Description**: 第二次调用按新局处理。
- **Pros**: 实现简单。
- **Cons**: 抹掉已用容量，且与真正的新局无法区分。
- **Rejection Reason**: GDD 要求返回 `CAPACITY_INIT_INVALID` 并进入 `Faulted`；跨局重置由 `end_run` + `initialize` 承担。

## Consequences

### Positive

- 一局只有一份容量账本，`RunLog` 与结算的不变式可以直接核对。
- 非法快照立即失败，内容/存档错误不会被伪装成平衡问题。
- 等号边界与单调性由同一实现保证，最优餐 20 / 20 可达。
- 「吃不下」不再有结束语义，玩家保有剩余回合与跳波的选择。
- 四条终止路径共用一个清零点，重开不会继承上一局的残留。

### Negative

- `RunFlow` 必须同时维护 `end_run` 的调用纪律；漏调会让下一局误判为同局第二次初始化。
- `can_fit` / `consume` 的参数是 `Variant`，调用方失去编译期类型提示，需靠测试覆盖。
- 没有退还接口，未来的消化/解腻机制必须以新 ADR 重开，而不是加一个方法。

## Risks

- **`end_run` 漏调**：下一局 `initialize` 返回 `CAPACITY_INIT_INVALID`，`RunFlow` 进入 `Faulted`。用集成测试覆盖四条终止路径。
- **同帧两次吃菜提交**：第二次按到达顺序独立校验；若放得下会被接受，重复提交由 `RunLog` 不变式捕获。
- **`C0` 与升级购买上限不一致**：购买侧必须保证 `C0 + 最大加成 ≤ 999`（ADR-0016）；本服务在开局再检查一次，双重防线。
- **未来按档口区分容量**：需要给 `initialize` 增加 `station_id`，会改签名；当前 MVP 不做，改动前必须先改本 ADR。
- **`capacity_used` 被下游写回**：契约禁止公共 setter；注册表的写访问必须只指向本服务。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `stomach-capacity.md` | TR-stomach-capacity-001：`initialize(upgrade_snapshot)` 每局恰好一次；`end_run(reason)` 是每条终止路径的权威边界 | 决策 2、3；同局第二次失败、四条路径统一归零 |
| `stomach-capacity.md` | TR-stomach-capacity-002：`capacity_total = C0 + capacity_bonus`，范围 `[1,999]`；非法/缺失快照失败，不默认 0 | 决策 2；严格类型与缺失校验，越界不钳制 |
| `stomach-capacity.md` | TR-stomach-capacity-003：`can_fit` / `consume` 用整数 `effective_satiety`，等号允许，已用容量单调 | 决策 4、5、7；等号边界、`INVALID_SATIETY` 拒绝负值、唯一写入口 |
| `stomach-capacity.md` | TR-stomach-capacity-004：没有「吃撑」结束状态；容量只决定单道菜能否吃 | 决策 6；无 `STUFFED`、无 `end_reason`、满了不结束 |
| `stomach-capacity.md` | TR-stomach-capacity-005：只读访问器；无信号、无第二计数器 | 决策 6；公开方法只有三个 getter，无信号 |
| `stomach-capacity.md` | Core Rule 3：容量与 `effective_satiety` 均为整数，本服务不重算、不缓存 | 决策 1、4；只接收整数参数，不读 `DishData` |
| `stomach-capacity.md` | Core Rule 10：`capacity_total ∈ [1,999]`，购买侧与开局双重保证 | 决策 2、7；`CAPACITY_TOTAL_OUT_OF_RANGE` |
| `stomach-capacity.md` | Core Rule 11：快照在一局内冻结 | 决策 7；`capacity_total` 开局后只读 |
| `stomach-capacity.md` | Core Rule 14 / AC 21：不读取菜品数据 | 决策 1；构造只收 `C0`，`consume` 只收整数 |
| `stomach-capacity.md` | AC 1–6：初始化、非法快照、999 边界、同局重复初始化、跨局重置 | 决策 2、3；状态与返回码逐条对应 |
| `stomach-capacity.md` | AC 7–11：等号允许、整局放不下不记错、非法 `e` 记录 ERROR、判定不预留 | 决策 4；边界与纯读取契约 |
| `stomach-capacity.md` | AC 12–17：原子提交、单调、溢出拒绝、无 refund/setter | 决策 5、7；唯一写入口 |
| `stomach-capacity.md` | AC 18–26：`Σ effective_satiety` 一致、冻结、确定性、放弃后清零 | 决策 3、6、7 |
| `run-flow.md` | Core Rule 8：没有「吃撑」结束原因 | 本服务只在 `can_fit` 返回 `false` 时参与展示，不参与结束判定 |

## Performance Implications

- **CPU**：`can_fit` / `consume` 为 O(1)，单次 < 1 µs；无循环、无分配。
- **Memory**：三个字段 + 一个只读的 `C0`，常驻开销可忽略。
- **Load Time**：不参与启动门；`initialize` 在 `Menu → Run` 时同步执行。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 定义 `State` / `InitStatus` / `ConsumeStatus` / `InitError` / `InitResult` 与测试骨架。
2. 实现快照校验与 `initialize` 成功/失败路径，跑 AC 1–6。
3. 实现 `can_fit` 与边界校验，跑 AC 7–11。
4. 实现 `consume` 与单调性，跑 AC 12–17。
5. 实现 `end_run` 与只读 getter，跑 AC 18–26。
6. 接入 `RunFlow` 的开局、吃菜提交与四条终止路径（ADR-0013 落地后）。

## Validation Criteria

- GDD AC 1–26 全部通过；AC 18–20、22–26 需要 `RunFlow` / 结算替身。
- 不变量测试：10,000 次随机合法提交后 `0 ≤ used ≤ total` 且单调不减；所有越界调用返回 `CAPACITY_OVERFLOW` 且字段不变。
- 快照矩阵：缺失 / `-1` / `1.5` / `null` 四个夹具都返回 `CAPACITY_INIT_INVALID`，且 `capacity_total` 与 `capacity_used` 未被写入。
- 静态断言：公开方法中只有 `initialize` / `consume` 能改变 `capacity_used`，不存在 `refund` / `release` / `reset` / 公共 setter。

## Related

- ADR-0004 服务组合与信号 — `AppRoot` 构造并注入 `StomachCapacity`
- ADR-0007 局生命周期与结算事务 — `run_id`、`end_run` 与 `RunLog`
- ADR-0011 回合预算与耗尽判定 — 并列的每局预算，互不调用
- ADR-0013 局内动作提交 — `can_fit` / `consume` 的唯一调用点
- ADR-0015 结算纯函数 — `capacity_total` / `capacity_used` 校验
- ADR-0016 局外升级 — `UpgradeSnapshot.capacity_bonus` 的生产者
- ADR-0019 HUD 数据绑定 — 容量只读快照
- `design/gdd/stomach-capacity.md`
- `design/gdd/run-flow.md`
