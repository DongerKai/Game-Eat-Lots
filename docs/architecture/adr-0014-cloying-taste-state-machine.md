# ADR-0014: 腻度折扣与口味解腻的数值状态机

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

本 ADR 把 `cloying-and-taste.md` 固定成一套**唯一的有状态实现契约**：`CloyingAndTaste` 是 `RefCounted`，按 `cloying_group` 持有本局整数计数，持有一份显式的规范组序；`initialize(resistance)` 每局清空并冻结抗性；`preview(dish)` 用**吃之前**的计数算出 `extra_satiety` / `effective_satiety` / `cleanse_target`，供 HUD 预选与 `RunFlow` 判定；`record_eat(dish)` 在提交时按固定三步（结算额外、组计数 +1、解腻目标 −1）推进状态。没有信号、没有缓存副本、没有百分比抗性，也没有“有效单位价值”这类把答案算给玩家的派生量。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Feature |
| **Layer** | Feature |
| **Knowledge Risk** | LOW — 只用 `int` / `float` / `String` / `StringName` / `Array` / `Dictionary` 与纯 GDScript 运算；`floor()` 为 4.x 稳定 API |
| **References Consulted** | `design/gdd/cloying-and-taste.md`、`design/gdd/dish-values.md`、`design/gdd/stomach-capacity.md`、`design/gdd/run-flow.md`、`design/gdd/in-game-hud.md`、`docs/architecture/architecture.md`、`docs/engine-reference/godot/VERSION.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① 规范组序并列时 `cleanse_target` 在两次运行、两个平台上返回同一组；② `group_cloy` 从未被解腻减到负数；③ `floor(2.9) = 2`、`floor(0.0) = 0` 在 Godot 4.7 的 `float`→`int` 路径上符合定点减免语义；④ 同一输入序列重放两次，各组计数与 `effective_satiety` 逐项相同；⑤ `preview` 连续调用 1,000 次不改变任何计数 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述五条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0004（服务组合）— `CloyingAndTaste` 由 `AppRoot` 构造并注入 `RunFlow`；ADR-0010（菜品数值）— `get_dish` / `satiety` / `taste_tags` / `cloying_group` 的唯一来源 |
| **Enables** | ADR-0013（动作管线）的提交第 2–3 步；ADR-0012（胃容量）的 `effective_satiety` 消费；ADR-0019（HUD）的预选预览；ADR-0020（首局引导）的“重复加格 / 酸冰解腻”教学 |
| **Blocks** | 腻度与口味系统的全部实现 story；`TR-cloying-and-taste-001..005`；任何“重复加格 / 解腻”的验收 |
| **Ordering Note** | 本 ADR 不定义 `satiety` / `taste_tags` / `cloying_group` 的内容（归 `DishData` 与 ADR-0010），也不定义容量判定（归 ADR-0012）。它只定义“给定一局历史，这道菜此刻多占几格、会解掉哪一组” |

## Context

### Problem Statement

`cloying-and-taste.md` 已经把公式写死，但实现层必须回答四个问题：

1. **预览与提交如何共用一份状态。** HUD 在选中态要显示“基础 4 + 额外 1 = 5”，`RunFlow` 在提交时要按同一个 `5` 扣容量；如果两处各自算一遍，或 HUD 缓存一份计数表，玩家看到的数与实际扣的数会漂移。
2. **“先付后记”的顺序。** 额外饱腹必须用**吃之前**的计数；如果先 `+1` 再算，一道菜会用自己的计数抬高自己的代价，糖醋里脊第二次会变成 6 格而不是 5 格。
3. **解腻的确定性。** 并列最高时必须按显式规范组序选一个；如果依赖 `Dictionary` 遍历顺序，同一份存档在不同平台会解掉不同的组，回放与复盘全部失效。
4. **抗性的语义。** `cloying_resistance` 是定点减免（`floor`），不是百分比；负数、`NaN`、缺失都不能静默变成 0。

### Current State

`src/` 只有占位文件。`architecture.md` 把 `CloyingAndTaste` 列为 Feature 层纯 `RefCounted`，但只列了 `initialize` / `effective_satiety` / `extra_satiety` / `end_run` 四个方法，**缺少提交时的状态推进入口**。缺的是 `preview` / `record_eat` 的分离、规范组序常量、错误词表与重置边界。

### Constraints

- **按组计数**：计数只看 `cloying_group`，不看 dish id、价格或口味标签。
- **先付后记**：`extra_satiety` 用吃之前的计数；本道菜的计数在容量扣减之后才 +1。
- **解腻只一次**：含 `酸` 与 `冰` 也只触发一次；只看其他组；不储存、不顺延、不退还已付容量。
- **空组不建组**：`cloying_group = ""` 的菜不计数、额外恒 0。
- **抗性定点**：`floor(cloying_resistance)` 从追加值中减去，最小为 0；不按百分比缩放。
- **确定性**：规范组序是显式数组；并列、目标选择、面板顺序全部按它。
- **无聚合**：只回答单菜的额外饱腹与解腻目标，不算总分、不算有效单位价值。
- **无持久化**：MVP 无中途存档，计数表不落盘。
- **无信号**：变化经 `RunFlow` 的 `action_committed` 与快照观察。
- **单人单线程**：预览与提交之间没有其他写入者；不需要锁。

### Requirements

- 必须每局 `initialize(resistance)` 一次，清零所有组计数并冻结抗性。
- 必须提供 `preview(dish)` 与 `record_eat(dish)`，前者纯读取、后者唯一写入口。
- 必须保证 `extra_satiety = max(0, min(count, cloy_cap) − floor(resistance))`，且用吃之前的计数。
- 必须保证解腻只减一个**其他**组、并列按规范组序、下限 0。
- 必须把 `extra_satiety` / `effective_satiety` / `cleanse_target` / 组计数暴露给 HUD，且在选择之前可读。
- 必须让空组、未知组与非法抗性有确定行为。
- 必须让 `menu_cloy_pressure` 只存在于 CI/设计期。

## Decision

### 1. 模块形态与依赖注入

`CloyingAndTaste` 是 `RefCounted`，位于 Feature 层：

- 构造参数：`dish_values: DishValues`、`cloy_cap: int`。`cloy_cap` 由 `AppRoot` 在 `DataConfig Ready` 后从当前档口的 `StationData.cloy_cap` 读出后传入（MVP 只有一个档口，值为 3）。
- 不持有 `RunFlow`、`StomachCapacity`、`HudHost` 或 `Save` 的引用。
- 不发射信号、不注册 `_process`、不读墙钟、不写存档。
- 只持有：`initialized`、`resistance_floor: int`、`counts: Dictionary[StringName, int]`、`cloy_cap: int`。

`cloy_cap < 1` 是数据错误：构造时断言；若绕过启动门直接注入，所有 `preview` 返回 `INVALID_CONFIG`。

**多档口说明**：`cloy_cap` 目前按 MVP 单档口在构造时注入。若 ADR-0017 引入多档口且各档口 `cloy_cap` 不同，必须改为每局参数并在新 ADR 中重写构造契约——不允许在局中改 `cloy_cap`。

### 2. 规范组序与确定性

```gdscript
const CANONICAL_GROUPS: Array[StringName] = [
    &"主食", &"蔬菜", &"饮料", &"甜", &"蛋", &"辣", &"油", &"鲜", &"肉",
]
```

- 并列时取 `CANONICAL_GROUPS` 中索引更小者。
- 不在数组内的组（未来内容）追加在末尾，并按 `StringName` 的字节序升序排列；该顺序在 `initialize` 时计算一次并冻结，局中不变。
- 禁止依赖 `Dictionary` 遍历顺序、哈希顺序或菜单加载顺序。
- 面板只显示计数 > 0 的组，顺序仍按上述规范序。

### 3. 初始化与重置

```gdscript
enum ErrorCode { NOT_INITIALIZED, INVALID_RESISTANCE, INVALID_CONFIG, DATA_NOT_READY, UNKNOWN_DISH }

class Preview:
    var extra_satiety: int
    var effective_satiety: int
    var cleanse_target: StringName   # &"" 表示无目标
    var is_cleansing: bool

class PreviewResult:
    var ok: bool
    var value: Preview
    var error: CloyingError
```

`initialize(resistance: float) -> InitResult`：

- 校验 `resistance` 是有限 `float` 且 `≥ 0`；否则返回 `INVALID_RESISTANCE`，状态保持未初始化，**不默认 0、不取绝对值**。
- `resistance_floor = int(floor(resistance))`。
- 清空 `counts`（全部组归零），`initialized = true`。
- 同一局内第二次 `initialize` 视为新局并完整替换（计数清零、抗性更新）；`RunFlow` 只在开局调用，重入由 `run_id` 的集成判据发现。
- `end_run(reason)`：丢弃计数与抗性，回到未初始化；幂等。`RunFlow` 可在终止路径调用，但下一次 `initialize` 已足以复位。

### 4. 预览：纯读取，吃之前

```gdscript
func preview(dish_id: StringName, caller: StringName) -> PreviewResult
func extra_satiety(dish_id: StringName, caller: StringName) -> IntResult
func effective_satiety(dish_id: StringName, caller: StringName) -> IntResult
func cleanse_target(dish_id: StringName, caller: StringName) -> StringNameResult
func group_counts() -> Array[GroupCount]      # 只读副本，按规范序
```

`GroupCount` 是只读值对象；HUD 与首局引导只消费它，不持有内部计数表：

```gdscript
class GroupCount extends RefCounted
var group: StringName       # 规范组名
var count: int              # 本局计数；只含 > 0 的组
```

`preview(dish)` 的规则：

1. `dish = dish_values.get_dish(dish_id)`；失败原样返回 `DATA_NOT_READY` / `UNKNOWN_DISH`。
2. `g = dish.cloying_group`；`g == ""` → `extra = 0`。
3. 否则 `extra = max(0, min(counts[g], cloy_cap) − resistance_floor)`。
4. `effective = dish.satiety + extra`；若 `satiety < 1` 返回 `INVALID_CONFIG`（防御）。
5. `is_cleansing = (酸 ∈ taste_tags) ∨ (冰 ∈ taste_tags)`。
6. `target = argmax_{h ≠ g, counts[h] ≥ 1} counts[h]`，并列取规范序靠前者；无候选 → `&""`。空组菜按公式仍可解腻（`g = ""` 时所有组都是候选）。

**必须**：`preview` 不改变任何字段；连续调用 1,000 次结果相同。HUD 在选中态与提交前都调用它，不缓存计数表。

### 5. 提交：三步固定顺序

```gdscript
func record_eat(dish_id: StringName, expected_extra: int, caller: StringName) -> RecordResult
```

`record_eat` 是唯一写入口，顺序**不可交换**：

1. **校验**：`initialized`；`dish` 可解析；`preview(dish).extra_satiety == expected_extra`。不相等返回 `STALE_PREVIEW`，不改变状态。这一步防止 `RunFlow` 把旧预览的额外值写进日志。
2. **先付**：调用方已用 `preview` 的 `effective` 完成容量扣减（ADR-0012）；本方法不再扣容量，只确认 `expected_extra` 与当前状态一致。
3. **记录**：`counts[g] += 1`（`g == ""` 时跳过）。
4. **解腻**：若 `is_cleansing` 且 `target ≠ ""`，`counts[target] -= 1`；`target` 的计数必然 ≥ 1，减后不会为负。只执行一次。

**必须**：解腻不退还已占用的容量、不改变本道菜的 `extra`、不影响已写入的 `EatenEntry`。

**必须**：`record_eat` 不重新计算容量、不调用 `StomachCapacity`；容量扣减归 ADR-0012 与 ADR-0013。

### 6. 抗性与边界

- `resistance_floor` 在 `initialize` 时冻结；局中不可变。`floor(0.9) = 0`、`floor(1.0) = 1`。
- `extra_satiety ∈ [0, cloy_cap]`；`cloy_cap = 3` 时 MVP 的组最多 2 道，`count ≤ 1`，上限永不生效。
- `counts[g]` 永远 `≥ 0`；解腻前已确保目标计数 ≥ 1。
- 空组：不建键、不计数、`extra = 0`；若含 `酸` / `冰`，仍可解掉其他组（公式语义）。
- 未在任何菜单出现的组不占状态，也不进入面板。
- 中途改 `StationData.cloy_cap` 不影响当前局；下一局构造/初始化才生效（MVP 构造期固定）。

### 7. 只读暴露与审计隔离

- HUD 读取 `preview(dish)`（额外、实际、解腻目标）与 `group_counts()`；不得缓存第二份计数表，不得自行推算 `effective`。
- `menu_cloy_pressure(pool)` 只允许 `tests/`、`tools/` 与 CI 调用；运行期不得暴露。
- 不提供“有效单位价值”或任何 `price / effective` 的排序/筛选；玩家自己算。
- 结果类型统一 `ok / value / error`，失败不返回 0、空组或占位菜。

### 8. 架构图

```
DishValues (Ready)
   │ get_dish → satiety / taste_tags / cloying_group
   ▼
CloyingAndTaste (RefCounted, 有状态)
   ├─ preview(dish) ──▶ {extra, effective, cleanse_target}   ← HUD / RunFlow
   ├─ record_eat(dish, expected_extra) ── 组计数 +1 → 解腻 −1
   ├─ group_counts() ──▶ 只读副本（规范序）
   ├─ initialize(resistance) / end_run(reason)
   └─ menu_cloy_pressure(pool) ── 仅 CI/设计期
              │
              ▼
        StomachCapacity（消费 effective_satiety）
```

## Alternatives Considered

### Alternative 1: 让 `effective_satiety(dish)` 同时推进计数

- **Description**: 预览即提交，一次调用完成计算与 +1。
- **Pros**: API 更少。
- **Cons**: HUD 预选会污染计数；同一帧的多次预览会把同一道菜算多次；容量判定与提交无法分离。
- **Rejection Reason**: 预览必须是纯读取；状态推进只在 `record_eat`。

### Alternative 2: 先 `+1` 再算额外饱腹

- **Description**: 直接在当前计数上计算。
- **Pros**: 少一次「吃之前」的快照。
- **Cons**: 一道菜用自己的计数抬高自己，第二次糖醋里脊变 6 格；违反“先付后记”。
- **Rejection Reason**: GDD 明确用吃之前的计数。

### Alternative 3: 解腻在多个并列组上同时生效

- **Description**: 并列最高时全部 −1。
- **Pros**: 解腻更强。
- **Cons**: 与「只减一个组」冲突；强度随菜单规模漂移；不可预期。
- **Rejection Reason**: 只减一个，并列按规范序。

### Alternative 4: 用 `Dictionary` 遍历找解腻目标

- **Description**: 遍历计数表取最大值。
- **Pros**: 无需规范序数组。
- **Cons**: 遍历顺序不稳定，并列结果跨平台不一致；回放失败。
- **Rejection Reason**: 规范组序是显式常量。

### Alternative 5: 抗性按百分比缩放

- **Description**: `extra *= (1 − resistance)`。
- **Pros**: 曲线平滑。
- **Cons**: 小整数上不可预测；`floor` 定点语义已经确定；百分比会引入浮点判定。
- **Rejection Reason**: 抗性是定点减免，不是百分比。

### Alternative 6: 把计数表持久化进存档

- **Description**: 中途存档时写入计数与规范序。
- **Pros**: 支持断点续玩。
- **Cons**: MVP 无中途存档；引入后必须定义版本与迁移。
- **Rejection Reason**: MVP 不持久化；未来启用时由存档 ADR 重开。

## Consequences

### Positive

- 预览与提交共用一份状态，HUD 显示与实际扣减不会漂移。
- 先付后记与三步顺序让每道菜的代价可手算、可复盘。
- 显式规范序让并列解腻在跨平台与回放中完全确定。
- 空组、非法抗性、未知菜都有封闭错误，不会静默变成 0。
- 运行期没有审计函数，`menu_cloy_pressure` 不会把“第二轴是否生效”暴露给玩家。

### Negative

- 多一个 `record_eat` 入口，`RunFlow` 必须按“预览 → 容量 → 提交”的顺序调用。
- `counts` 的键集合随吃过的组增长；面板需要按规范序过滤，不能直接遍历字典。
- 构造期固定 `cloy_cap`，多档口需要新 ADR。

## Risks

- **HUD 缓存计数**：契约禁止；预览与快照都从本服务读取，集成测试对比两处。
- **`record_eat` 与预览不一致**：用 `expected_extra` 断言，失败返回 `STALE_PREVIEW` 并让 `RunFlow` 进入 `Faulted`。
- **解腻目标被自己组选中**：候选集显式排除 `group(d)`；测试覆盖糖醋里脊（甜酸油、属肉）不能解肉。
- **未来组不在规范序**：按字节序追加；若内容作者依赖特定顺序，必须先把组加入常量数组。
- **`cloy_cap` 被局中修改**：构造期固定，局中不可变。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `cloying-and-taste.md` | TR-cloying-and-taste-001：按 `cloying_group` 的整数计数每局重置；显式规范组序数组 | 决策 2、3；`CANONICAL_GROUPS` 常量与 `initialize` 清零 |
| `cloying-and-taste.md` | TR-cloying-and-taste-002：`effective_satiety = satiety + max(0, min(count, cloy_cap) − floor(resistance))`，用吃之前计数 | 决策 4、5；`preview` 纯读取、`record_eat` 后置 +1 |
| `cloying-and-taste.md` | TR-cloying-and-taste-003：`酸`/`冰` 只减计数最高的其他组一次；并列按规范序 | 决策 2、4、5；`cleanse_target` 与单次 −1 |
| `cloying-and-taste.md` | TR-cloying-and-taste-004：`initialize(cloying_resistance)` 在 `run_started` 前；MVP 不持久化 | 决策 3；`end_run` 丢弃，无存档写入 |
| `cloying-and-taste.md` | TR-cloying-and-taste-005：向 HUD 暴露 `extra_satiety` / `cleanse_target` / 组计数，且在选择之前 | 决策 4、7；`preview` + `group_counts` 只读 |
| `cloying-and-taste.md` | Core Rule 5 / 8：先付后记；解腻不退还已付容量 | 决策 5；顺序固定，`record_eat` 不碰容量 |
| `cloying-and-taste.md` | Core Rule 14 / 15：MVP 口味只做解腻；空组不参与 | 决策 4、6；空组 `extra = 0`，解腻按公式 |
| `cloying-and-taste.md` | Core Rule 16：每局一次 `initialize`，不读升级表与存档 | 决策 1、3 |
| `cloying-and-taste.md` | Formulas：`menu_cloy_pressure` 只用于审查 | 决策 7；审计方法隔离到 CI |
| `cloying-and-taste.md` | AC 1–20：加格、解腻、并列、空组、重置、确定性 | 决策 2–5；逐条对应 |

## Performance Implications

- **CPU**：`preview` / `record_eat` 为 O(G)（G = 组数，MVP 9）；`cleanse_target` 一次线性扫描；单次 < 5 µs。
- **Memory**：计数表最多 9 个键；无缓存、无数组增长（除 `eaten` 归 `RunFlow`）。
- **Load Time**：不参与启动门；`initialize` 在 `Menu → Run` 时同步执行。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 定义 `CANONICAL_GROUPS`、`Preview` / Result 形状与错误词表。
2. 实现 `initialize` / `end_run` 与空组、非法抗性路径，跑 AC 9、20。
3. 实现 `preview` / `extra_satiety` / `effective_satiety` / `cleanse_target`，跑 AC 1–3、14、15。
4. 实现 `record_eat` 与三步顺序，跑 AC 4–8、13、16–19。
5. 接入 HUD 预览与 `RunFlow` 提交（ADR-0013 / 0019 落地后）。
6. 加 `menu_cloy_pressure` 的 CI 入口（不进入运行期）。

## Validation Criteria

- GDD AC 1–20 全部通过；AC 17–20 使用可注入测试菜单，不改 12 道菜真实数值。
- 不变量：10,000 次随机 `record_eat` 后所有计数 ≥ 0 且 ≤ 吃菜数；每次额外饱腹 ≤ `cloy_cap`。
- 并列测试：构造多组同计数，断言每次解腻目标与规范序一致。
- 纯读取测试：`preview` 连续 1,000 次前后计数表逐键相同。
- 静态断言：运行期目录不引用 `menu_cloy_pressure`；HUD 不持有 `counts` 的引用副本。

## Related

- ADR-0004 服务组合与信号 — `AppRoot` 构造并注入 `CloyingAndTaste`
- ADR-0010 菜品数值契约 — `satiety` / `taste_tags` / `cloying_group` 来源
- ADR-0012 胃容量账本 — `effective_satiety` 的消费方
- ADR-0013 动作管线 — `preview` → `can_fit` → `record_eat` 的调用顺序
- ADR-0015 结算纯函数 — 实际占用格数进入 `RunLog`
- ADR-0019 HUD 数据绑定 — 额外饱腹与解腻预览
- ADR-0020 首局引导 — 腻度教学
- `design/gdd/cloying-and-taste.md`
- `design/gdd/stomach-capacity.md`
