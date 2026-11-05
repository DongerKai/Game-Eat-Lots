# ADR-0010: 菜品数值与整数数值契约

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

《回本》的每一次取舍都落在「单位价值」这一把尺子上：`unit_value(d) = price(d) / satiety(d)`。本 ADR 把 `dish-values.md` 的公式固定成一套**唯一的运行时实现契约**：`DishValues` 是无状态、无缓存的纯 `RefCounted` 服务，只从 `DataConfig` 读记录；所有比较与排序用 **int64 交叉相乘**，浮点只出现在展示与跨度差值；公开查询统一返回 Result 风格，失败给封闭的 `QueryError {caller, dish_id, reason}`；`opt_value` / `menu_healthy` 等平衡审查只存在于 CI/设计期入口，运行期 API 表面不得暴露任何求和或最优解。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Core |
| **Layer** | Core |
| **Knowledge Risk** | LOW — 本决策不依赖 4.5–4.7 的新 API，只用 `int` / `float` / `StringName` / `Array` 与纯 GDScript 运算 |
| **References Consulted** | `design/gdd/dish-values.md`、`design/gdd/data-config.md`、`docs/architecture/architecture.md`（原则 5、`DishValues` 边界、Appendix A）、`docs/engine-reference/godot/VERSION.md`、`current-best-practices.md`、`deprecated-apis.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① int64 交叉相乘在 `price = INT32_MAX`、`satiety = 6` 时不溢出（约 `1.29×10¹⁰`）；② `float64(price) / float64(satiety)` 与 `floor(uv × 100 + 0.5) / 100` 在 Windows / macOS / Linux 上逐位一致（IEEE 754 双精度）；③ `StringName` 转 `String` 后的字典序对全部合法 id 为字节序；④ 空数组、单元素数组的 `uv_span` / `menu_healthy` 返回 GDD 规定的值而不是抛错；⑤ `opt_value` 的 0/1 背包递推在 `C = 20`、`T = 10`、12 道菜时返回 104 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述五条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001（内容数据格式）— `DishData` 的 schema、只读性、id 规则与 `Ready` 门；ADR-0004（服务组合）— `DishValues` 由 `AppRoot` 构造并注入 `DataConfig` |
| **Enables** | ADR-0014（腻度与口味）的折扣输入；ADR-0015（结算纯函数）的单菜价格与单位价值；ADR-0019（HUD）的 `price` / `satiety` 只读绑定；ADR-0020（首局引导）的结算期讲解数据 |
| **Blocks** | 菜品数值系统的全部实现 story；任何需要「跨菜比较」的验收（HUD 排序、腻度错峰、结算复盘） |
| **Ordering Note** | 本 ADR 不决定 12 道菜的具体数值，也不决定入场费——数值内容归 `DishData` 记录，校准入场费归 ADR-0015。本 ADR 只规定「拿到记录之后如何算、如何比、如何报错」 |

## Context

### Problem Statement

`dish-values.md` 已经把公式写死（`unit_value`、`uv_cmp`、`uv_span`、`menu_healthy`、`dominates`、`opt_value`、`calibration_ratio`），但没有回答三件实现层必须回答的事：

1. **浮点在哪里出现、在哪里绝对不出现。** 单位价值是除法，但比较、支配与排序不能依赖浮点；如果下游各自写一份 `price / satiety`，两处公式会漂移，且 `15/3` 与 `20/4` 的并列会在不同平台上变成随机结果。
2. **查询失败长什么样。** GDD 要求「不返回 0、空值或占位菜」，但没有固定错误对象与错误码；若实现成 `null` 或 `0.0`，数值 bug 会伪装成平衡 bug。
3. **运行时与 CI 的边界。** `opt_value` 一旦被运行期调用，玩家就能背下最优菜单；`menu_healthy` 一旦阻断启动，内容作者改坏一道菜就进不了游戏。两者必须物理隔离，而不是靠注释约定。

### Current State

`src/` 目前只有占位文件，没有任何实现。`architecture.md` 已把 `DishValues` 列为 Core 层的纯 `RefCounted` 服务，并声明 `DataConfig` 是唯一数据源。缺的是可被 story 直接引用的 API 表面、数值类型与错误词表。

### Constraints

- **只读上游**：不修改 `DishData`，不持有副本或缓存；连续两次查询必须返回同一个只读实例。
- **纯整数判定**：排序、支配、健康判定不得出现浮点比较；比较一律用 int64 交叉相乘。
- **无状态**：`DishValues` 没有内部可变状态、没有信号、没有 `_process`。
- **确定性**：同一记录集下，所有输出恒定；排序以 `uv_cmp` 降序、`id` 升序为唯一次序。
- **不聚合**：运行时不得对菜集合求和、求净利或求最优解。
- **平衡失败不阻断启动**：审查失败只写报告。
- **单人单线程**：MVP 无并发调用；API 不需要锁。

### Requirements

- 必须提供 `unit_value` 的唯一实现，下游不得重算。
- 必须提供精确比较，`uv_cmp = 0` 只在精确相等时出现。
- 必须返回结构化错误，且错误里带调用者与 dish id。
- 必须把 `opt_value` / `menu_healthy` 等审查函数限制在 CI/设计期入口。
- 必须让 `menu_healthy` 只作用于档口菜池，不作用于单次刷新菜单。
- 必须在空菜单、单菜菜单、全 1 格菜单、全等 UV 菜单下返回规定值而不是抛错。

## Decision

### 1. 模块形态：`DishValues` 是无状态纯服务

`DishValues` 是 `RefCounted`，位于 Core 层，由 `AppRoot` 在 CONSTRUCT 阶段构造，构造参数只有一个 `DataConfig` 引用。它：

- 不缓存任何 `DishData`，每次查询都委托 `DataConfig.get_dish(id)`；
- 不发射信号、不注册 `_process`、不读墙钟、不写存档；
- 不持有 `MenuWaves`、`Cloying`、`Settlement` 或 `Save` 的引用。

`DataConfig` 未 `Ready` 时，所有公开查询返回 `DATA_NOT_READY`，不构造部分结果。

### 2. 数值契约：整数是事实，浮点只做展示

| 量 | 类型 | 规则 |
|---|---|---|
| `price` | `int`（schema 上界 `INT32_MAX`） | 原样来自 `DishData`，不参与除法以外的运算 |
| `satiety` | `int`（1–6） | 上游已校验；本层仍断言 `1 ≤ satiety ≤ 6` |
| `unit_value` | `float64` | 只在「单位价值查询」与「展示」路径计算；不写回记录 |
| `uv_cmp` | `int` ∈ {−1, 0, 1} | `sign(price_a × satiety_b − price_b × satiety_a)`，int64 乘法 |
| `uv_span` | `float64` | 用 `uv_cmp` 选出最大/最小，再用两者的精确分数在 float64 中相减；容差固定 `1e-9` |
| 展示值 | `float64` | `floor(uv × 100 + 0.5) / 100`，固定两位小数；**不参与任何判定** |

**必须**：`uv_cmp` 的乘积在 int64 中完成；当前上界约为 `1.29×10¹⁰`，安全。若未来放宽 schema，必须先改本 ADR 并改用 128 位或加界，不得静默截断。

**必须**：`unit_value` 在 `satiety = 0` 时返回 `INVALID_SATIETY`，绝不执行除法。

### 3. 公开 API 表面

```gdscript
class_name DishValues
extends RefCounted

# 构造：AppRoot 注入 DataConfig
func _init(data_config: DataConfig) -> void

# --- 运行时（story 可用） ---
func get_dish(dish_id: StringName, caller: StringName) -> DishResult
func price(dish_id: StringName, caller: StringName) -> IntResult
func satiety(dish_id: StringName, caller: StringName) -> IntResult
func unit_value(dish_id: StringName, caller: StringName) -> ValueResult
func uv_cmp(a: DishData, b: DishData, caller: StringName) -> CompareResult

# --- 审计（仅 CI / 设计期；不出现在运行期调用链） ---
func uv_span(pool: Array[DishData]) -> FloatResult
func same_group_dominated_pairs(pool: Array[DishData]) -> IntResult
func cross_dominated_pairs(pool: Array[DishData]) -> IntResult
func menu_healthy(pool: Array[DishData]) -> BoolResult
func opt_value(pool: Array[DishData], capacity: int, turns: int) -> Int64Result
func calibration_ratio(pool: Array[DishData], capacity: int, turns: int, entry_fee: int) -> FloatResult
```

`DishResult` / `ValueResult` / `IntResult` / `CompareResult` / `FloatResult` / `BoolResult` / `Int64Result` 都是同一个 Result 形状：

```gdscript
class Result:
    var ok: bool
    var value: Variant          # 类型由具体 Result 固定
    var error: QueryError       # ok = false 时非空
```

`QueryError`：

```gdscript
class QueryError:
    var caller: StringName      # 谁发起的查询，例如 &"hud" / &"settlement"
    var dish_id: StringName     # 出错的那道菜；成对比较时是被判定非法的那一侧
    var reason: Reason          # 封闭枚举

enum Reason { DATA_NOT_READY, UNKNOWN_DISH, INVALID_SATIETY }
```

**必须**：失败时返回 `ok = false` 的 Result，`value` 保持未设置；不得返回 `0`、`0.0`、`null`、空记录或占位菜。

**必须**：`uv_cmp` 接受已经取得的 `DishData` 记录；若任一记录不是 `DataConfig` 当前发布的实例，返回 `UNKNOWN_DISH`。这样调用者可以复用 `get_dish` 的结果，避免每帧重复解析 id。

### 4. 排序与支配

- 排序唯一键：`uv_cmp` 降序，`id` 字典序升序。
- 不得依赖 `Dictionary` 遍历、资源加载顺序或 `Array.sort_custom` 的稳定性假设。
- `dominates(B, A)` 的 `same_group` 判定必须把两个空 `cloying_group` 视为同组。
- `menu_healthy` 只接收档口菜池；调用者不得把单次刷新菜单传进来（GDD Rule 10）。
- `tier_coverage` 按 `satiety = 1`、`2 ≤ satiety ≤ 4`、`satiety ≥ 5` 三档判定。

### 5. 运行时 / 审计隔离

- `opt_value`、`calibration_ratio`、`menu_healthy`、`uv_span`、`*_dominated_pairs` 只允许被 `tests/`、`tools/` 与 CI 脚本调用。
- 运行期 story 不得导入或引用审计方法；CI 用一条静态断言扫描运行期 API 表面，确保没有求和、净利或最优解入口。
- 审计失败只写报告（warning / CI failure），不改变 `DataConfig` 状态、不阻断 `Boot`。

### 6. 架构图

```
DataConfig (Ready)
   │  get_dish(id) → 只读 DishData
   ▼
DishValues (RefCounted, 无状态)
   ├─ 运行时：get_dish / price / satiety / unit_value / uv_cmp
   └─ 审计：uv_span / menu_healthy / dominates / opt_value / calibration_ratio
        ▲                                   ▲
        │                                   │
   HUD / Cloying / Settlement          tests/ + CI（禁止运行期引用）
```

## Alternatives Considered

### Alternative 1: 把 `unit_value` 存进 `DishData`

- **Description**: 在数据记录里预计算并保存 `unit_value`。
- **Pros**: 查询 O(1)，无除法。
- **Cons**: 记录变成派生数据的副本；改价格/饱腹必须同步改比值，容易漂移；浮点存档/序列化不稳定。
- **Rejection Reason**: 违反「原始值只读、派生值实时计算」，且把平衡问题带进数据层。

### Alternative 2: 允许下游各自计算 `price / satiety`

- **Description**: 每个系统需要时自己算。
- **Pros**: 无需共享服务。
- **Cons**: 两份公式必然漂移；浮点并列判定不一致；GDD 明确禁止。
- **Rejection Reason**: 单位价值是跨系统契约，只能有一个实现。

### Alternative 3: 用浮点直接比较单位价值

- **Description**: `if a.price / a.satiety > b.price / b.satiety`。
- **Pros**: 代码直观。
- **Cons**: `15/3` 与 `20/4` 可能因末位误差判成不等；跨平台不确定；无法处理精确并列。
- **Rejection Reason**: 排序与支配必须用整数交叉相乘。

### Alternative 4: 把平衡审查放进运行期并阻断启动

- **Description**: 启动时跑 `menu_healthy`，失败就 `Boot → Error`。
- **Pros**: 坏数据无法进入游戏。
- **Cons**: 内容作者改坏一道菜就进不了游戏，无法调试；平衡问题被当成加载错误。
- **Rejection Reason**: GDD 明确「平衡失败不阻断启动」；审查归 CI。

## Consequences

### Positive

- 跨系统只有一个单位价值实现，消除了公式漂移。
- 精确并列、支配与排序在所有平台一致。
- 错误可归因到调用者与具体菜品。
- 最优解与健康审查被物理隔离在 CI，玩家无法从运行期 API 拿到答案。

### Negative

- 每次查询都走一次 `DataConfig.get_dish`，比缓存多一次间接调用（可忽略，p95 预算见下）。
- 审计函数与运行时代码在同一类里，需要靠目录/静态断言维持边界。
- `uv_cmp` 接受 `DishData` 而不是 id，调用者必须先 `get_dish`，多一步样板代码。

## Risks

- **schema 放宽导致 int64 溢出**：当前上界安全；任何放宽 `price` / `satiety` 的改动必须先改本 ADR，CI 增加边界测试。
- **审计方法被运行期误用**：用静态断言 + 目录分层（`tests/` / `tools/`）双保险；代码评审拒绝运行期 import。
- **浮点展示与精确值不一致**：展示值只用于 UI，所有判定走 `uv_cmp`；AC 覆盖两位小数显示。
- **`cloying_group` 空串语义漂移**：`same_group` 的实现集中在本服务，禁止下游各自判断空串。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `dish-values.md` | TR-dish-values-001：`unit_value = price/satiety` 是唯一跨菜比较刻度，下游不得重算 | 决策 1、2、3；`DishValues.unit_value` 是唯一实现，公开 API 不含其他除法入口 |
| `dish-values.md` | TR-dish-values-002：静态 12 道菜目录；`menu_healthy` / `opt_value` 只在设计期与 CI 运行 | 决策 5；审计方法隔离到 `tests/` / CI，运行期 story 禁止引用 |
| `dish-values.md` | TR-dish-values-003：运行期不得聚合、求最优解或暴露 `opt_value` | 决策 5；运行期 API 表面无求和/净利/最优解方法，CI 静态断言 |
| `dish-values.md` | TR-dish-values-004：确定性的 `uv_cmp` 排序（单位价值降序、id 升序） | 决策 2、4；int64 交叉相乘 + id 字典序，禁止浮点比较与容器顺序 |
| `dish-values.md` | TR-dish-values-005：Result 风格 `QueryError {caller, dish_id, reason}`，不得返回 0/占位 | 决策 3；封闭枚举与统一 Result 形状 |
| `dish-values.md` | Core Rule 2 / AC 11：连续查询返回同一只读实例，赋值被拒绝 | 决策 1；不缓存、不复制，始终返回 `DataConfig` 发布的实例 |
| `dish-values.md` | Core Rule 7 / 8 / 10：数值带、严格支配、`menu_healthy` 只作用于档口菜池 | 决策 4；`menu_healthy` 接收菜池数组，三档覆盖与同组支配集中实现 |
| `dish-values.md` | Formulas：`uv_span` 的 1.5 门限与 `1e-9` 容差 | 决策 2；精确分数相减 + 固定容差常量 |
| `dish-values.md` | Formulas / AC 25：`opt_value` 0/1 背包，12 道菜、C=20、T=10 返回 104 | 决策 5；审计入口实现递推，运行期不可达 |

## Performance Implications

- **CPU**：`unit_value` / `uv_cmp` 为 O(1)，单次 < 1 µs；完整审计（132 个有向对 + 0/1 背包）≤ 50 ms，其中支配扫描 ≤ 1 ms、`opt_value` ≤ 5 ms（GDD AC 29 的预算）。
- **Memory**：无缓存、无状态，常驻开销为 0；审计临时数组在调用结束后释放。
- **Load Time**：不参与启动门；`DataConfig Ready` 后即时可用。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 先写 `QueryError` / Result 形状与单元测试（不碰文件 I/O）。
2. 实现 `unit_value` / `uv_cmp` / 排序，跑 AC 1–5。
3. 实现错误路径，跑 AC 6–7。
4. 实现审计函数与 CI 入口，跑 AC 12–29。
5. 接入 HUD / Cloying / Settlement 的只读查询（各自 ADR 落地后）。

## Validation Criteria

- GDD AC 1–11（运行时）与 AC 12–29（设计期/CI）全部通过。
- 静态断言：运行期目录不引用 `opt_value` / `menu_healthy` / 任何求和函数。
- 跨平台夹具：同一 `mvp` 记录集在三个平台输出逐位一致。
- 错误路径：`DATA_NOT_READY` / `UNKNOWN_DISH` / `INVALID_SATIETY` 各有一条独立测试，且返回的 `caller` 与 `dish_id` 正确。

## Related

- ADR-0001 内容数据格式 — `DishData` schema、只读性与 `Ready` 门
- ADR-0004 服务组合与信号 — `AppRoot` 构造与注入 `DishValues`
- ADR-0006 确定性随机 — `menu-waves` 的健康门使用 `opt_value` 作为基准
- ADR-0014 腻度与口味 — 消费 `cloying_group` / `satiety`
- ADR-0015 结算纯函数 — 消费 `price` / `unit_value`
- ADR-0019 HUD 数据绑定 — 只消费 `price` / `satiety`
- ADR-0020 首局引导 — 结算期单位价值讲解
- `design/gdd/dish-values.md`
