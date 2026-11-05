# ADR-0015: 结算纯函数、入场费与回本判定

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

本 ADR 把 `settlement-and-entry-fee.md` 固定成一套**无 I/O、无时钟、无随机数的纯计算契约**：`Settlement` 是 `RefCounted`，只依赖 `DishValues`，把一份 `RunLog` 变成一份 `RunResult`。总价值用受检 int64 求和，净利允许为负且不钳制，`broke_even` 是 `net_profit ≥ 0` 的派生谓词而不入库；收据行严格按吃下顺序，入场费为零时 `return_ratio` 为 `null` 且不做除法。运行期实时净利仍归 `RunFlow` 的累加器，本服务只定义 `running_net(k)` / `shortfall(k)` 的纯函数口径，并让 `total_value` 复用同一个求和应用，从根上消灭第二套算术。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Feature |
| **Layer** | Feature |
| **Knowledge Risk** | LOW — 只用 `int`（GDScript 为 64 位有符号）/ `float` / `StringName` / `Array` / `Dictionary` 与纯运算；不触碰文件、场景树、时钟或随机源 |
| **References Consulted** | `design/gdd/settlement-and-entry-fee.md`、`design/gdd/dish-values.md`、`design/gdd/run-flow.md`、`design/gdd/meta-upgrades.md`、`design/gdd/records-and-milestones.md`、`design/gdd/in-game-hud.md`、`docs/architecture/architecture.md`、`docs/architecture/adr-0007-run-state-settlement-transaction.md`、`docs/engine-reference/godot/VERSION.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① GDScript `int` 的 64 位溢出在 4.7.2 上确实**静默回绕**，因此前置比较能阻止负总额；② `1000 × total / fee` 的整数除法在所有合法范围（`total ≤ turns_total × INT32_MAX`，MVP ≤ 25 行）不溢出；③ `float(total) / float(fee)` 与 `return_percent_x10` 的展示值在两位/一位小数格式化后一致；④ `entry_fee = 0` 路径不执行任何除法，不产生 `INF` / `NAN`；⑤ 同一 `RunLog` 连续两次 `settle`（不同 `Settlement` 实例）逐字段相同；⑥ 行顺序在 `Array` 追加与遍历中保持稳定 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述六条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0004（服务组合）— `Settlement` 由 `AppRoot` 构造并注入 `RunFlow` / `SettlementTransaction`；ADR-0007（结算事务）— `RunLog` 的组装时机、`settle` 每局一次、`RunResult` 的交付与收据快照；ADR-0010（菜品数值）— `get_dish` / `price` / `satiety` / `unit_value` 的唯一来源；ADR-0012（胃容量）— `capacity_total` 与逐行实际占用；ADR-0014（腻度）— `extra_satiety` / `effective_satiety` 在吃下时冻结 |
| **Enables** | ADR-0016（局外升级）的 `net_profit` / `broke_even` 入账；ADR-0018（纪录与里程碑）的最佳纪录比较；ADR-0019（HUD）的 `running_net(k)` / `shortfall(k)` 口径；界面流程的收据屏 |
| **Blocks** | `TR-settlement-002` / `003` / `005` 的验收；结算与入场费的全部实现 story；收据屏的任何“合计”验收 |
| **Ordering Note** | `RunLog` 何时组装、`settle` 何时调用、`RunResult` 如何交接与何时提交存档，全部归 ADR-0007。本 ADR 只回答“给定这份日志，收据上每个数是多少”。 |

## Context

### Problem Statement

GDD 已经把每个公式写死，但实现层必须回答四个问题：

1. **局内与结算页必须是同一套算术。** 玩家在局内看到 `−85` 起步、每道菜往上加；如果结算页用另一段求和代码，任何一次漂移都会让玩家看到两个世界。
2. **行值不能重算。** `extra_satiety` 与 `effective_satiety` 是吃下那一刻的事实；结算若按菜名重放腻度序列，就会复制一套解腻规则，规则一改，收据上的占用和玩家实际付掉的容量就对不上。
3. **金额是有符号整数，派生量不存第二份。** 净利可以为负且不得钳制；`broke_even` 若作为字段存储，迟早与 `net_profit` 不一致；入场费为零时回本率无定义，不能显示 `∞` 或 `NaN`。
4. **结算不能碰钱包与纪录。** 一旦 `Settlement` 读 `MetaUpgrades`、`Records` 或 `Save`，就会与 `局外升级 → 结算 → 局外升级` 形成环，并且不再可单测、不可回放。

### Current State

`src/` 只有占位文件。`architecture.md` 已把 `Settlement` 列为 Feature 层纯 `RefCounted`，并给出 `settle(log: RunLog) -> Result[RunResult, SettlementError]`；ADR-0007 已固定事务边界与 `RunResult` 的唯一交付路径。缺的是：`RunLog` / `RunResult` 的字段级契约、求和与错误词表、`return_ratio` 的零费用分支、四条运行时不变式的判定顺序，以及 `running_net` / `shortfall` 的单一实现。

### Constraints

- **纯函数**：无文件/网络 I/O、无系统时钟、无随机数；同一输入逐字段相同。
- **只读**：不写 `RunLog`、不写任何子系统状态、不触发钱包装账或纪录比较。
- **单一来源**：`price` / `satiety` / 静态 `unit_value` 只从 `DishValues` 读；实际占用只从 `RunLog.eaten` 读。
- **有符号 int64**：所有金额与占用求和使用受检 int64；任何一步溢出返回 `OVERFLOW`，不回绕。
- **不钳制**：负净利原样返回；容量越界、行不一致、回合不一致都返回错误，不“就地修正”。
- **行序即事实**：`rows` 严格按 `eaten` 顺序，不排序、不合并、不聚合重复菜。
- **无最优解**：不读、不存、不展示 `opt_value` 或任何“本可以多赚”的派生量。
- **每局一次**：`last_settled_run_id` 去重；成功的第二次调用返回 `ALREADY_SETTLED`，失败不写入该字段。
- **单人单线程**：`settle` 是同步调用，不需要锁。

### Requirements

- 必须输出 GDD 定义的 `RunResult` 全部字段，行内静态单位价值直接取自 `DishValues`。
- 必须让 `total_value`、`running_net`、`shortfall` 共用同一个价格前缀求和实现。
- 必须让 `broke_even` 只作为派生读法存在。
- 必须对 `entry_fee = 0`、空局、重复菜、负净利、非法数据与溢出给出确定结果。
- 必须在产出结果前验证 GDD 的四条运行时不变式。
- 必须让结果事件的语义（`broke_even` / `loss` 二选一）在本 ADR 固定，但发放点留在 ADR-0007 的事务提交之后。

## Decision

### 1. 模块形态与依赖

`Settlement` 是 `RefCounted`，位于 Feature 层：

- 构造参数：`dish_values: DishValues`。不持有 `RunFlow`、`StomachCapacity`、`CloyingAndTaste`、`MenuWaves`、`MetaUpgrades`、`Records` 或 `SaveSystem`。
- 唯一可变字段：`last_settled_run_id: int = 0`（与 ADR-0007 的事务幂等闸相互独立，二者不互相替代）。
- 不发射信号、不注册 `_process`、不读墙钟、不写存档、不访问场景树。
- 档口存在性、`entry_fee` 的解析与冻结由 `RunFlow` 在开局完成；本服务不重复查询档口。

```gdscript
class_name Settlement extends RefCounted

func _init(dish_values: DishValues) -> void
func settle(log: RunLog, caller: StringName = &"RunFlow") -> SettlementResult
func is_broke_even(net_profit: int) -> bool
func running_net(entries: Array[EatenEntry], k: int, entry_fee: int, caller: StringName) -> Int64Result
func shortfall(entries: Array[EatenEntry], k: int, entry_fee: int, caller: StringName) -> Int64Result
```

`Settlement` 的状态只有 `Idle / Settled` 两个值，由 `last_settled_run_id` 隐含表达；没有“结算中”状态，因为调用是同步且无 I/O 的。

### 2. 输入 `RunLog`

| 字段 | 类型 | 校验 |
|---|---|---|
| `run_id` | int64 | `> 0`；去重键 |
| `station_id` | StringName | 非空；原样转交，不重新查询档口 |
| `entry_fee` | int | `0 ≤ fee ≤ INT32_MAX`；否则 `INVALID_ENTRY_FEE` |
| `end_reason` | enum | 只接受 `TIME_EXHAUSTED` / `NO_EATABLE_DISH`；其他值 `ROW_INCONSISTENT`（防御） |
| `eaten` | EatenEntry[] | 按吃下顺序；每项 `dish_id`、`extra_satiety ≥ 0`、`effective_satiety ≥ 1` |
| `capacity_total` | int | `≥ 1`；否则 `INVALID_CAPACITY` |
| `capacity_used` | int | `≥ 0`；必须等于各行 `effective_satiety` 之和 |
| `turns_total` / `turns_used` | int | `0 ≤ used ≤ total`；且 `eaten.size() ≤ turns_used` |
| `upgrade_snapshot` | Dictionary | 原样深拷贝转交；不解释、不校验业务含义 |

`RunLog` 由 `RunFlow` 在 `Ending` 组装并一次性传入（ADR-0007）。`Settlement` 不读取 `RunFlow` 快照，也不读取腻度/容量/回合的当前状态——**日志之外没有第二个输入**。

### 3. 输出 `RunResult`

| 字段 | 类型 | 来源 |
|---|---|---|
| `run_id` / `station_id` / `end_reason` | 同 `RunLog` | 原样转交 |
| `entry_fee` | int | 原样转交；纪录必须冻结它 |
| `total_value` | int64 | `_sum_prices(eaten, n)` |
| `net_profit` | int64 | `total_value − entry_fee`，有符号 |
| `return_ratio` | float64 \| null | `fee > 0` 时 `float(total)/float(fee)`；`fee = 0` 时 `null` |
| `return_percent_x10` | int \| null | `fee > 0` 时 `floor(1000 × total / fee)`；`fee = 0` 时 `null` |
| `rows` | EatenRow[] | 按吃下顺序；见下 |
| `capacity_total` / `capacity_used` / `wasted_capacity` | int | 原值 / 逐行求和 / `total − used` |
| `cloy_extra_total` | int | `Σ extra_satiety` |
| `turns_total` / `turns_used` | int | 原样转交 |
| `upgrade_snapshot` | Dictionary | 深拷贝转交 |

`EatenRow`：`dish_id`、`price`、`satiety`（基础值）、`extra_satiety`、`effective_satiety`、`unit_value`（取自 `DishValues`，不重算）、`effective_unit_value`（本服务计算，仅展示）。

- **`broke_even` 不存储**：`func is_broke_even(net_profit: int) -> bool: return net_profit >= 0`。收据、局外升级、纪录、音效全部从 `net_profit` 派生；任何地方不得缓存布尔副本。
- **不包含菜名**：`RunResult` 只携带数值事实；显示名由 `ReceiptViewModel` 通过 `DishValues.get_dish` 解析（ADR-0007 D6），避免把展示层字符串写进可回放的结算事实。
- **不包含 `opt_value`**：`RunResult` 没有任何“上限 / 最优 / 差额”字段。

### 4. `settle` 的固定校验顺序

```gdscript
enum ErrorCode {
    ALREADY_SETTLED, DATA_NOT_READY, UNKNOWN_DISH,
    INVALID_ENTRY_FEE, INVALID_PRICE, INVALID_CAPACITY,
    OVERFLOW, CAPACITY_MISMATCH, CAPACITY_OVERFLOW,
    ROW_INCONSISTENT, TURN_INCONSISTENT,
}

class SettlementError:
    var caller: StringName
    var code: ErrorCode
    var detail: Dictionary   # run_id / dish_ids / field / expected / actual

class SettlementResult:
    var ok: bool
    var value: RunResult
    var error: SettlementError
```

`settle(log)` 按以下顺序执行，任一步失败立即返回，**不产生部分结果、不更新 `last_settled_run_id`**：

1. **去重**：`log.run_id == last_settled_run_id` → `ALREADY_SETTLED`。
2. **数据就绪**：`DishValues` 的首次查询返回 `DATA_NOT_READY` 时原样透传。
3. **入场费**：`typeof(fee) == TYPE_INT` 且 `0 ≤ fee ≤ INT32_MAX`；否则 `INVALID_ENTRY_FEE`。
4. **回合**：`0 ≤ turns_used ≤ turns_total` 且 `eaten.size() ≤ turns_used`；否则 `TURN_INCONSISTENT`。
5. **解析菜品**：按 `eaten` 顺序解析全部 `dish_id`；收集**所有**缺失 id 后一次性返回 `UNKNOWN_DISH`，`detail.dish_ids` 为去重后的列表。
6. **逐行校验**：`price ≥ 0`（否则 `INVALID_PRICE`）；`satiety ≥ 1`；`extra_satiety ≥ 0`；`effective_satiety == satiety + extra_satiety` 且 `≥ 1`（否则 `ROW_INCONSISTENT`）。
7. **容量求和**：`used = Σ effective_satiety`（受检 int64）；`used != log.capacity_used` → `CAPACITY_MISMATCH`；`used > capacity_total` → `CAPACITY_OVERFLOW`；`capacity_total < 1` 或 `log.capacity_used < 0` → `INVALID_CAPACITY`。
8. **总价值**：`total = Σ price`（受检 int64，见决策 5）。
9. **净利**：`net = total − fee`（`total ≥ 0`、`fee ≥ 0`，int64 内不会下溢）。
10. **派生量**：`wasted = capacity_total − used`（由步骤 7 保证 `≥ 0`）；`cloy_extra = Σ extra`；回本率按决策 6。
11. **组装行**：保持 `eaten` 顺序，逐行填入价格、基础饱腹、额外/实际饱腹、静态单位价值与有效单位价值。
12. **提交去重键**：仅在此刻写入 `last_settled_run_id = log.run_id`，返回 `ok = true`。

> 去重必须排在数据检查之前：同一 `run_id` 的第二次调用即使数据已变化，也必须稳定返回 `ALREADY_SETTLED`，而不是先报另一个错误。

### 5. 金额与溢出

- 所有累加使用 `_checked_add(a: int, b: int) -> Int64Result`：`b > 0 and a > INT64_MAX − b` → `OVERFLOW`；GDScript 的 `int` 为 64 位有符号，发生溢出会静默回绕，因此**不能依赖运行时异常**。
- `INT64_MAX = 9_223_372_036_854_775_807`。
- `price` 上限 `INT32_MAX`、行数由 `turns_total` 约束（MVP 10、满级 25），正常玩法不可能溢出；检查是为了让非法数据变成可见错误。
- 任何一步溢出返回 `OVERFLOW`，不输出部分金额、不回绕成负数。
- `net_profit` 不做上下钳制；`0 − 85 = −85` 是合法结果，原样返回。

### 6. 回本率与零入场费

```gdscript
if fee == 0:
    ratio = null
    percent_x10 = null          # 收据显示「免单」，不执行除法
else:
    ratio = float(total) / float(fee)
    percent_x10 = int((1000 * total) / fee)   # total ≥ 0、fee > 0，整数除法即 floor
```

- `percent_x10` 用整数运算生成，避免浮点格式化漂移；展示层固定一位小数（`1223 → 122.3%`）。
- `ratio` 仅供 API / 测试使用；收据只读 `percent_x10` 或 `ratio`，不得二次舍入。
- `fee = 0` 时绝不允许出现 `∞`、`NaN`、`0` 或任何“回本率”字样的数值；`broke_even` 恒为 `true`。

### 7. 行序、重复菜与单位价值

- `rows` 顺序 = `eaten` 顺序；不提供排序入口，不按价格或单位价值重排。
- 同一道菜出现两次 → 两行、两次 `price`、两次各自的实际饱腹；`cloy_extra_total` 累加两次各自的 `extra_satiety`。
- `unit_value` 直接取自 `DishValues.unit_value(d)`（ADR-0010）；本服务不重算 `price / satiety`。
- `effective_unit_value = float(price) / float(effective_satiety)`；仅收据展示，不参与任何判定、排序或筛选。
- 空局：`rows = []`，`total = 0`，`net = −fee`，`percent_x10 = 0`（`fee > 0`），`broke_even = false`；仍是一份完整收据。

### 8. 单一求和来源

```gdscript
func _sum_prices(entries: Array[EatenEntry], k: int) -> Int64Result   # 前 k 道菜的 price 和
```

- `total_value = _sum_prices(eaten, n)`。
- `running_net(k) = _sum_prices(eaten, k) − entry_fee`。
- `shortfall(k) = max(0, entry_fee − _sum_prices(eaten, k))`。
- 三个公开量共用同一个前缀求和；不存在“结算一套、HUD 一套”。
- `k` 必须满足 `0 ≤ k ≤ entries.size()`；越界返回 `ROW_INCONSISTENT`。
- **运行期累加器仍归 `RunFlow`**：开局初始化为 `−entry_fee`，每次成功提交 `+price`，随快照发布；本服务在结算时用同一份 `RunLog` 复核。HUD 读快照或调用这两个纯函数，不得自建累加器（ADR-0007 / 0019）。

### 9. 结果事件的语义与发放点

- 结果事件只有两个，互斥：`broke_even`（`net ≥ 0`）、`loss`（`net < 0`）。
- 语义与命名归本 ADR；**发放点在 ADR-0007 的事务提交之后**。`Settlement` 本身是纯函数，不发射信号；若 `records.prepare_run` 致命失败、事务未提交，界面流程发 `settlement_aborted`，`broke_even` / `loss` 均不得播放。
- `settlement_started` / `settlement_aborted` 归界面流程，本服务不定义、不发出。

### 10. 状态与边界图

```
RunFlow（唯一 RunLog 生产者，唯一运行期累加器）
   │ settle(RunLog)  每 run_id 一次
   ▼
Settlement（RefCounted，纯函数）
   ├─ 去重 last_settled_run_id → ALREADY_SETTLED
   ├─ 校验 4 条不变式 + 入场费/回合/菜品
   ├─ total = Σ price（受检 int64）
   ├─ net = total − fee（有符号，不钳制）
   ├─ rows 按吃下顺序；unit_value 取自 DishValues
   └─ return {RunResult | SettlementError}
              │
              ├─▶ SettlementTransaction（ADR-0007：钱包/纪录/存档/收据快照）
              ├─▶ MetaUpgrades（读 net_profit / broke_even）
              ├─▶ Records（读净利、总值、回本率、档口、结束原因）
              └─▶ HUD（只读 running_net / shortfall 口径，实时值仍来自 RunFlow 快照）
```

## Alternatives Considered

### Alternative 1: 结算按菜名重放腻度序列

- **Description**: 只用 `dish_id` 列表，结算时重新跑一遍腻度状态机。
- **Pros**: `RunLog` 更小。
- **Cons**: 复制一套腻度/解腻规则；抗性、`cloy_cap` 或规范序一改，历史局重算结果改变；收据与玩家实际付出的容量不一致。
- **Rejection Reason**: GDD 明确要求行值在吃下时冻结，结算不重算。

### Alternative 2: 用 float 求和金额

- **Description**: 所有金额用 `float` 累加，方便显示。
- **Pros**: 少写溢出检查。
- **Cons**: 大整数与小数格式化会漂移；合计与净利在边界值上不可回放；`int64` 语义已经写进 GDD 与 TR。
- **Rejection Reason**: 金额一律 int64，展示层单独格式化。

### Alternative 3: 把负净利钳制到 0

- **Description**: 亏损局按 0 计，避免负钱包。
- **Pros**: 钱包永远非负。
- **Cons**: 隐藏失败、破坏支柱 1 的算术题；钱包策略属于局外升级，不属于结算。
- **Rejection Reason**: 结算不钳制；是否允许负钱包由 ADR-0016 决定。

### Alternative 4: 把 `broke_even` 存进 `RunResult`

- **Description**: 结算时算好布尔并入字段。
- **Pros**: 下游少一次比较。
- **Cons**: 与 `net_profit` 形成两份事实；任何一处修改或序列化都可能漂移。
- **Rejection Reason**: 派生谓词，读取时计算。

### Alternative 5: 让结算顺手写钱包与纪录

- **Description**: `settle` 内直接入账、比较纪录、提交存档。
- **Pros**: 调用方少一步。
- **Cons**: 形成 `局外升级 → 结算 → 局外升级` 环；结算不再纯；失败时无法区分“算错了”与“存不下”。
- **Rejection Reason**: 事务边界归 ADR-0007；结算只产出 `RunResult`。

### Alternative 6: 零入场费显示 `∞` 回本率

- **Description**: `total / 0` 直接展示为无限大。
- **Pros**: 形式上连续。
- **Cons**: 无定义值进入 UI；格式化、排序、测试都不稳定。
- **Rejection Reason**: `return_ratio = null`，收据显示「免单」。

### Alternative 7: 让 `Settlement` 也维护运行期累加器

- **Description**: HUD 直接读 `Settlement.running_net`，`RunFlow` 不再累加。
- **Pros**: 只有一处求和。
- **Cons**: 结算服务持有局内可变状态；动作提交与快照发布跨服务同步；与 ADR-0007 的唯一累加器冲突。
- **Rejection Reason**: 运行期累加器在 `RunFlow`；本服务只提供同一实现的纯函数。

### Alternative 8: 收据按价格或单位价值排序

- **Description**: 默认按价格降序，方便玩家看“最贵的一口”。
- **Pros**: 视觉上更“有用”。
- **Cons**: 顺序本身是复盘信息；排序会把“从第三盘开始走错”抹掉；也违反 GDD 的“不提供排序控件”。
- **Rejection Reason**: 严格按吃下顺序。

## Consequences

### Positive

- 局内与结算页共用同一个价格前缀求和，玩家看到的世界只有一套。
- 行值来自日志快照，腻度规则演进不会改写历史收据。
- 有符号净利、派生回本判定与零费用分支都有封闭语义，UI 不需要猜。
- 纯函数 + 单一依赖让结算可以脱离游戏循环做全量单测与回放。
- 结算不碰钱包/纪录，避免与未定型的局外系统形成设计环。

### Negative

- `RunLog` 必须携带 `extra_satiety` / `effective_satiety`，上游（`RunFlow` / 腻度 / 容量）不能只传 `dish_id`。
- `RunResult` 不含菜名，收据需要额外的显示名解析步骤（归 ADR-0007 D6）。
- 溢出检查是手写前置比较，GDScript 不提供 checked 算术。
- `last_settled_run_id` 只在会话内有效；跨会话的幂等仍依赖 ADR-0007 的 `run_id + digest` 闸。

## Risks

- **GDScript 溢出静默回绕**：所有累加走 `_checked_add`；测试注入 `INT64_MAX` 价格夹具。
- **上游 `capacity_used` 与逐行之和不一致**：返回 `CAPACITY_MISMATCH`，由 `RunFlow` 进入 `Faulted`；不钳制、不补行。
- **`unit_value` 被下游重算**：`EatenRow.unit_value` 直接取自 `DishValues`；静态检查禁止收据路径出现 `price / satiety`。
- **结果事件被提前播放**：发放点固定在 ADR-0007 事务提交之后；事务中止时测试断言 cue 未被 admitted。
- **`upgrade_snapshot` 共享引用被下游改写**：结算时深拷贝；ADR-0007 再做一次快照深拷贝，两层都不依赖调用方自律。
- **零费用路径误入除法**：`fee == 0` 分支在任何除法之前返回 `null`；测试断言结果中不含 `INF` / `NAN`。
- **跨会话 `run_id` 重用**：本服务的去重只覆盖当前会话；持久化幂等由 ADR-0007 与存档命名空间负责。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `settlement-and-entry-fee.md` | `TR-settlement-002`：`total_value` int64 求和；`net_profit = total − entry_fee`；`broke_even` 派生；不钳制 | 决策 5、6；`_checked_add`、有符号净利、`is_broke_even` 派生读法 |
| `settlement-and-entry-fee.md` | `TR-settlement-003`：`RunResult` schema；行按吃下顺序；`fee = 0` 时 `return_ratio` 为 `null`；`upgrade_snapshot` 透传 | 决策 2、3、6、7；字段表、零费用分支、深拷贝转交 |
| `settlement-and-entry-fee.md` | `TR-settlement-005`：`running_net(k)` / `shortfall(k)` 供 HUD；单一求和来源 | 决策 8；`_sum_prices` 同时实现 `total_value` 与两个纯函数 |
| `settlement-and-entry-fee.md` | Core Rule 1 / 18：结算不碰钱包、纪录、存档 | 决策 1；构造依赖只有 `DishValues` |
| `settlement-and-entry-fee.md` | Core Rule 2 / 15：`entry_fee` 来自 `RunLog` 且冻结；零入场费合法 | 决策 2、6；不查档口、零费用分支 |
| `settlement-and-entry-fee.md` | Core Rule 3 / 12：价格即得分、无额外加成；不展示最优解 | 决策 3、7；`RunResult` 无 `opt_value` 字段 |
| `settlement-and-entry-fee.md` | Core Rule 4 / 5：一次交易一张收据；净利可为负且不钳制 | 决策 5；有符号净利原样返回 |
| `settlement-and-entry-fee.md` | Core Rule 6：`broke_even = net_profit ≥ 0` | 决策 3；派生谓词，不存储 |
| `settlement-and-entry-fee.md` | Core Rule 7 / 13：每局恰好结算一次；纯函数、可回放 | 决策 1、4、10；`last_settled_run_id` 与无 I/O 约束 |
| `settlement-and-entry-fee.md` | Core Rule 9 / 10 / 11 / 16：行序、行字段、合计覆盖容量与回合、重复菜按次计价 | 决策 3、7、4（步骤 7–11） |
| `settlement-and-entry-fee.md` | Core Rule 14 / 17：受检 int64；数据未就绪或未知菜是错误不是零分 | 决策 4、5；`OVERFLOW` / `DATA_NOT_READY` / `UNKNOWN_DISH` |
| `settlement-and-entry-fee.md` | Core Rule 19：局内净利与结算同源；运行期累加器归 `RunFlow` | 决策 8；唯一前缀求和，累加器不在本服务 |
| `settlement-and-entry-fee.md` | Formulas：`total_value` / `net_profit` / `broke_even` / `return_ratio` / `return_percent_x10` / `capacity_used` / `wasted_capacity` / `cloy_extra_total` / `effective_unit_value` | 决策 3–8；逐式落地 |
| `settlement-and-entry-fee.md` | 运行时不变式 1–4：容量匹配、容量不越界、行一致、回合一致 | 决策 4 步骤 4–7；封闭错误码 |
| `settlement-and-entry-fee.md` | Edge Cases：空局、`net = 0`、`fee = 0`、重复菜、负价、非法容量、未知菜、溢出、重复结算 | 决策 4–7；逐项判定 |
| `settlement-and-entry-fee.md` | AC 1–32（结算计算、行序、零费用、负数、空局、重复菜、错误路径） | 决策 3–9；测试矩阵见 Validation |

## Performance Implications

- **CPU**：`settle` 对每道菜做常数次查询与加法，O(n)；MVP `n ≤ 10`、满级 `n ≤ 25`。典型 < 0.2 ms，硬上限 5 ms（含 `DishValues` 查询）。
- **Memory**：`RunResult` 每行 7 个数值字段，25 行 < 16 KiB；无缓存、无全局表。
- **Load Time**：不参与启动门；`Settlement` 构造只保存一个引用。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 定义 `RunLog` / `EatenEntry` / `RunResult` / `EatenRow` / 错误词表与 Result 形状。
2. 实现 `_checked_add`、`_sum_prices`、`is_broke_even` 与 `running_net` / `shortfall`，跑纯函数单测。
3. 实现 `settle` 的 12 步顺序与四条不变式，跑 AC 的错误路径。
4. 接入零费用、空局、重复菜与深拷贝夹具。
5. 接入 `SettlementTransaction`（ADR-0007）与收据快照字段映射。
6. 接入 HUD 的 `running_net` / `shortfall` 复核判据（ADR-0019）。

## Validation Criteria

- GDD AC 1–32 全部通过；至少覆盖：F1（104 / +19 / 122.3%）、F2（54 / −31 / 63.5%）、空局（0 / −85）、零费用（免单、`null`）、`net = 0`（回本）、重复菜两行。
- 不变量：注入 `capacity_used` 不一致、行和不一致、回合倒挂，分别得到 `CAPACITY_MISMATCH`、`ROW_INCONSISTENT`、`TURN_INCONSISTENT`。
- 溢出：注入 `INT64_MAX` 级价格，断言 `OVERFLOW` 且结果中无任何金额。
- 纯函数：同一 `RunLog` 两个实例各结算一次，`RunResult` 深比较相等；静态检查确认 `Settlement` 不引用 `FileAccess` / `Time` / `RandomNumberGenerator` / `SaveSystem` / `MetaUpgrades` / `Records`。
- 零费用：断言 `return_ratio == null`、`return_percent_x10 == null`，且收据字符串不含 `∞` / `NaN`。
- 行序：乱序价格夹具下，`rows` 的 `dish_id` 序列与 `eaten` 完全一致。
- 同源：对 1,000 条随机合法日志，`total_value == running_net(n, 0)` 且 `running_net(k)` 与 `RunFlow` 累加器逐 k 相等。

## GDD Sync Check

- **结果事件发放点**：GDD 写“本系统只发 `broke_even` / `loss`”，但纯函数不发射信号；本 ADR 明确“语义归结算、发放点在 ADR-0007 事务提交之后”。无需回改 GDD，属实现层细化。
- **菜名不在 `RunResult`**：GDD 要求“每行至少包含菜名”，但 `RunResult` 字段表未列名称；本 ADR 判定名称由收据 ViewModel 解析，显示仍满足 GDD。无需回改。
- **空局回本率**：GDD 的 `return_ratio` 范围写 `(0, +∞)`，空局公式实际得 `0`；本 ADR 确认 `0` 是合法展示值（`0.0%`），零费用仍为 `null`。建议下次修订 GDD 时把范围改为 `[0, +∞)`。

## Related

- ADR-0004 服务组合与信号 — `AppRoot` 构造并注入 `Settlement`
- ADR-0007 结算事务 — `RunLog` 组装、`settle` 一次、`RunResult` 交付与提交
- ADR-0010 菜品数值契约 — `price` / `satiety` / `unit_value` 来源
- ADR-0011 回合预算 — `turns_total` / `turns_used`
- ADR-0012 胃容量账本 — `capacity_total` 与逐行实际占用
- ADR-0014 腻度状态机 — `extra_satiety` / `effective_satiety` 的冻结来源
- ADR-0016 局外升级 — 消费 `net_profit` / `broke_even`
- ADR-0018 纪录与里程碑 — 消费总值、净利、回本率
- ADR-0019 HUD 数据绑定 — `running_net` / `shortfall` 口径
- `design/gdd/settlement-and-entry-fee.md`
- `design/gdd/dish-values.md`
