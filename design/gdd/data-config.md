# 数据配置（Data Config）

> **Status**: In Design
> **Author**: user + agents
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 每口都是算术题；数字看得见地涨

## Summary

数据配置是《回本》唯一的数值来源：菜品、升级、档口都是带类型的 Godot Resource，按稳定 id 加载并在启动时校验。它保证游戏里的每个平衡数字都能被找到、被修改、被验证，代码里不写死任何数值。玩家不直接接触它，但 HUD 上的每个数字都来自这里。

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `None`

## Overview

数据配置是《回本》唯一的数值来源。每道菜、每项升级、每个档口都是一份带类型的 Godot Resource（`DishData`、`UpgradeData`、`StationData`），按稳定 id 存取；游戏脚本里不写死任何平衡数字，改数值只改 `.tres`。

它在启动时加载并校验全部记录：id 唯一、数值在合法范围、引用完整。任何一条不合法就拒绝进入游戏，并指出文件与字段——这是"看得懂"这一反支柱在数据层的落地，也是支柱 1「每口都是算术题」的前提：玩家能算，是因为系统自己先算得清。

玩家不直接接触这个系统，但玩家在 HUD 上看到的每个数字都来自这里。

## Detailed Design

### Core Rules

1. **唯一数值来源**：菜品、升级、档口的每一处可调数值都来自数据记录；GDScript 中不得出现平衡数字字面量（0、1 等结构性常量除外）。
2. **三类记录，三类 Resource**：`DishData`、`UpgradeData`、`StationData`，字段固定，在 Godot 编辑器里可视化编辑。
3. **`DishData` 字段**：
   - `id: StringName` — 稳定唯一，格式 `^[a-z][a-z0-9_]*$`
   - `display_name: String`
   - `price: int` — 单点价（元），≥ 0
   - `satiety: int` — 占用的饱腹格，≥ 1
   - `taste_tags: Array[StringName]` — 只能取自固定标签表：酸、甜、苦、辣、鲜、冰、油；可为空（无味菜）
   - `cloying_group: StringName` — 腻度分组；为空表示不参与腻度
   - `station_id: StringName` — 必须指向存在的档口，且该档口的 `dish_pool` 必须包含这道菜
   - `icon: Texture2D` — 缺失时用占位图标（警告，不致命）
4. **`UpgradeData` 字段**：
   - `id`、`display_name`、`description`
   - `cost: int` — ≥ 0
   - `max_level: int` — ≥ 1
   - `effect_type: enum {STOMACH_CAPACITY, CLOYING_RESISTANCE, MENU_VISION}`
   - `effect_value: float` — 每级增量，> 0；`STOMACH_CAPACITY` 与 `MENU_VISION` 必须是正整数，`CLOYING_RESISTANCE` 可带小数
   - `prerequisite_id: StringName` — 可空；非空时必须存在
5. **`StationData` 字段**：
   - `id`、`display_name`
   - `entry_fee: int` — ≥ 0
   - `unlock_cost: int` — ≥ 0（0 表示开局可用）
   - `dish_pool: Array[StringName]` — 至少 1 项，每项必须存在，且每道菜的 `station_id` 必须指回本档口（双向一致）
6. **id 是契约**：按类型命名空间隔离（`dish_*` / `upgrade_*` / `station_*`）。发布后不得改名——改名等于删除旧记录并新增，会让存档失效。
7. **启动即校验**：加载全部记录 → 校验 → 建立目录 → Ready。任一规则失败就停在这里，不进入游戏。
8. **运行时只读**：局内状态（当前饱腹、净利、剩余时间格）属于餐局流程，任何系统都不得写回数据记录。
9. **查询走目录**：下游系统通过 `get_dish(id)` / `get_upgrade(id)` / `get_station(id)` 取记录；不得自行扫描资源目录或按文件名猜测。

### States and Transitions

| 状态 | 触发 | 下一个状态 |
|---|---|---|
| Unloaded | 游戏启动 | Loading |
| Loading | 全部 `.tres` 读取成功 | Validating |
| Loading | 文件缺失或解析失败 | Failed |
| Validating | 全部校验规则通过 | Ready |
| Validating | 任一规则失败 | Failed |
| Ready | 退出游戏 | Unloaded |
| Failed | —（终态，不进入游戏） | — |

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 菜品数值 | 下游读取 | `DishData` 的 `price`、`satiety` | 硬 |
| 腻度与口味 | 下游读取 | `DishData` 的 `taste_tags`、`cloying_group` | 硬 |
| 菜单波次 | 下游读取 | `StationData` 的 `dish_pool` | 硬（Vertical Slice 期） |
| 局外升级 | 下游读取 | `UpgradeData` 的全部字段 | 硬 |
| 餐局流程 | 下游读取 | `StationData` 的 `entry_fee` | 硬 |
| 存档 | 下游引用 | 只存 id 与等级/数值，不存记录本身 | 硬 |

**接口所有权**：数据配置拥有三个 catalog 的查询与校验；下游只读，不拥有写入路径。

## Formulas

### 记录集合合法性

The `valid_record_set` formula is defined as:

`valid(D) = uniqueIds(D_dish) ∧ uniqueIds(D_upg) ∧ uniqueIds(D_sta) ∧ (∀d ∈ D_dish: R_dish(d)) ∧ (∀u ∈ D_upg: R_upg(u)) ∧ (∀s ∈ D_sta: R_sta(s))`

其中：

- `R_dish(d) = d.price ≥ 0 ∧ d.satiety ≥ 1 ∧ d.taste_tags ⊆ TAGSET ∧ d.station_id ∈ S_ids ∧ d.id ∈ dish_pool(d.station_id)`
- `R_upg(u) = u.cost ≥ 0 ∧ u.max_level ≥ 1 ∧ u.effect_value > 0 ∧ (u.effect_type ∈ {STOMACH_CAPACITY, MENU_VISION} → u.effect_value ∈ ℤ⁺) ∧ (u.prerequisite_id = "" ∨ (u.prerequisite_id ∈ U_ids ∧ ¬cyclic(u)))`
- `R_sta(s) = s.entry_fee ≥ 0 ∧ s.unlock_cost ≥ 0 ∧ |s.dish_pool| ≥ 1 ∧ (∀p ∈ s.dish_pool: p ∈ D_ids ∧ dish(p).station_id = s.id)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `D_dish` / `D_upg` / `D_sta` | Resource[] | ≥ 0 项 | data file | 三类记录加载后的集合 |
| `price` | int | 0–INT32_MAX | data file | 单点价（元） |
| `satiety` | int | 1–INT32_MAX | data file | 占用的饱腹格 |
| `cost` / `entry_fee` / `unlock_cost` | int | 0–INT32_MAX | data file | 元 |
| `max_level` | int | 1–INT32_MAX | data file | 升级最大等级 |
| `effect_value` | float | (0, +∞) | data file | 每级增量；离散类型必须为正整数 |
| `taste_tags` | StringName[] | TAGSET 子集，可为空 | data file | 口味标签 |
| `dish_pool` | StringName[] | ≥ 1 项 | data file | 档口菜池 |
| `TAGSET` | StringName[7] | 固定 | constant | 酸 / 甜 / 苦 / 辣 / 鲜 / 冰 / 油 |
| `valid(D)` | bool | {true, false} | calculated | 是否允许启动 |

**Output Range:** `valid(D) ∈ {true, false}`，无中间态。false 时阻断启动，并输出每一条违反项的记录 id 与字段名。

**Example:** 2 个档口、4 道菜、2 项升级。档口 `station_canteen` 的 `entry_fee=30`、`dish_pool=[dish_fried_rice, dish_braised_pork]`；`dish_fried_rice` 的 `price=8`、`satiety=2`、`taste_tags=[oil]`、`station_id=station_canteen`；`upgrade_capacity` 的 `cost=40`、`max_level=3`、`effect_type=STOMACH_CAPACITY`、`effect_value=1`。所有引用互相指回，`valid(D)=true`。

### 加载期派生量

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `dish_count` | int | ≥ 1 | calculated | `|D_dish|`；≥1 由 `R_sta` 蕴含（池非空且引用必须存在） |
| `dishes_per_station(s)` | int | 0–`dish_count` | calculated | `|{d ∈ D_dish : d.station_id = s.id}|`；必须等于 `|s.dish_pool|`，否则 `R_sta` 失败 |
| `prereq_depth(u)` | int | 1–`upgrade_count` | calculated | 前置链最长长度；超过记录数即存在环 |

**Output Range:** 计数一律以 int64 承载；schema 本身不计算价格总和，任何派生求和由下游系统负责并须使用 int64。

### 极值行为

- `price = 0`：合法（免费白水）；是否值得吃由下游单位价值自然得出。
- 单菜池：合法。池太薄是平衡问题，不是数据错误。
- `INT32_MAX`：单值合法；下游任何求和都必须用 int64，否则溢出。
- `taste_tags` 为空：合法（白饭、白水）。
- 多条记录的 `cloying_group` 同为空串：合法，不构成唯一性冲突。
- 前置环或自引用：任一 `prerequisite_id` 链回到自身即失败，属结构错误。
- 档口双向不一致：`dish.station_id` 与 `station.dish_pool` 任一侧缺失对方即失败。

## Edge Cases

- **If 同类型两条记录共用同一个 id**：加载失败，报出两条记录的文件路径与 id；不加载任何记录。
- **If `satiety` 为 0**：校验失败。下游的单位价值计算以饱腹为分母，0 会让局面无法定义；不钳制。
- **If `price` 为 0**：合法（例如免费白水）。下游单位价值为 0，照常参与结算。
- **If `price` 或 `satiety` 为负**：校验失败。
- **If `taste_tags` 含固定表之外的标签**：校验失败，并列出未知标签与文件——拼写错误不能静默失效。
- **If `cloying_group` 为空**：合法，该菜不参与腻度递减。
- **If `station_id` 指向不存在的档口**：校验失败。
- **If 某档口 `dish_pool` 为空**：校验失败。
- **If `dish_pool` 含不存在的 id**：校验失败，列出全部缺失 id。
- **If 某道菜不属于任何档口**：校验失败——它永远无法出现在游戏中，是死数据。MVP 采用严格策略。
- **If `prerequisite_id` 指向不存在的升级**：校验失败。
- **If `prerequisite_id` 形成环**（A 需要 B、B 需要 A）：校验失败，并打印环路径。
- **If `icon` 缺失**：警告并使用占位图标，游戏继续——美术可以晚于数据到位。
- **If `.tres` 解析失败或资源类型不符**：校验失败，报告文件路径与期望的 Resource 类型。
- **If 数值超出 int 范围，或 float 字段为 NaN / INF**：校验失败。
- **If 两个系统同时查询同一条记录**：返回同一个只读实例；调用方不得修改，修改请求一律视为 bug。
- **If 数据层允许"某道菜明显最优"**：这不是数据配置的职责——它只保证记录合法；数值是否失衡由菜品数值与腻度与口味的校验负责。

**退化策略边界**：数据配置不阻止玩家找到最优解，它只阻止不合法的数据进入游戏。把"算穿"当成数据错误会掩盖真正需要的平衡设计。

## Dependencies

**上游依赖：无。** 数据配置是基础层系统，不依赖任何其他系统；它只依赖 Godot 的 Resource 加载机制。

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 菜品数值 | 硬 | `get_dish(id)` → `price`、`satiety` | GDD 未写 |
| 腻度与口味 | 硬 | `get_dish(id)` → `taste_tags`、`cloying_group` | GDD 未写 |
| 菜单波次 | 硬（VS） | `get_station(id)` → `dish_pool` | GDD 未写 |
| 局外升级 | 硬 | `get_upgrade(id)` → 全部字段 | GDD 未写 |
| 餐局流程 | 硬 | `get_station(id)` → `entry_fee` | GDD 未写 |
| 存档 | 硬 | 持久化 id 与等级/数值；读取时不信任存档里的数值快照 | GDD 未写 |
| 局内 HUD | 软（间接） | 通过菜品数值/时间与回合取得数字，不直接读 catalog | GDD 未写 |
| 场景与流程管理 | 硬 | `Boot` 期读取加载与校验的最终结果（`Ready` / `Failed`），决定进入 Menu 还是 Error | GDD 已写 |

**双向一致要求**：以上每个系统在编写自己的 GDD 时，必须在 Dependencies 中列出「数据配置」为上游依赖。目前只有场景与流程管理的 GDD 已完成并已列出本系统；其余系统仍待设计，此要求记录在此供后续核对。

## Visual/Audio Requirements

**不适用。** 数据配置不渲染、不播放任何内容。它唯一可见的输出是启动校验报告，面向开发者而非玩家：

- 校验失败：日志输出文件路径、记录 id、字段名、期望值与实际值。
- 校验通过但有警告（例如缺 `icon`）：WARNING 级日志，并使用占位图标。

## Game Feel

**不适用。** 数据配置没有玩家可感知的即时反馈。它的"手感"体现在下游：改一个 `.tres` 数值，下一局立刻按新数值结算，不重启、不出现缓存不一致——这一点由 Acceptance Criteria 第 23 条验证。

## UI Requirements

**无。** 本系统不提供任何界面、HUD 元素或交互。局内 HUD 上的数字由下游系统读取记录后显示，玩家永远不会直接接触数据层。

## Cross-References

**None — this system references no other GDD.** 数据配置是最先设计的系统。场景与流程管理的 GDD 已完成并已在 Cross-References 中回指本系统；其余下游系统（菜品数值、腻度与口味、菜单波次、局外升级、餐局流程、存档、局内 HUD）在各自 GDD 写好后，需要把「数据配置」列回 Cross-References，届时在此表补充。

## Acceptance Criteria

每条都能由 QA 在不读本 GDD 的情况下独立执行。测试夹具放在 `tests/fixtures/data/`，每条用例使用独立夹具。

1. **GIVEN** 夹具含 1 条 `DishData`、1 条 `UpgradeData`、1 条 `StationData`，互相引用完整，**WHEN** 启动，**THEN** 状态依次为 Unloaded → Loading → Validating → Ready，`get_dish` / `get_upgrade` / `get_station` 都能取到对应记录。
2. **GIVEN** 两个 dish 夹具 id 同为 `rice`，**WHEN** 校验，**THEN** 状态为 Failed，日志含重复 id `rice`，主场景未被实例化。
3. **GIVEN** id 为 `Rice_01` 或 `1abc`，**WHEN** 校验，**THEN** Failed；id 为 `a_b1` 时 **THEN** Ready。
4. **GIVEN** `price=-1`，**WHEN** 校验，**THEN** Failed；`price=0` 时 **THEN** Ready。
5. **GIVEN** `satiety=0`，**WHEN** 校验，**THEN** Failed；`satiety=1` 时 **THEN** Ready。
6. **GIVEN** `taste_tags=["咸"]`，**WHEN** 校验，**THEN** Failed；`taste_tags=[]` 时 **THEN** Ready。
7. **GIVEN** `cloying_group=""`，**WHEN** 校验，**THEN** Ready，且校验报告的 warnings 计数为 0。
8. **GIVEN** `station_id` 指向不存在的档口，**WHEN** 校验，**THEN** Failed；**GIVEN** 档口存在但 `dish_pool` 不含该菜，**WHEN** 校验，**THEN** Failed（两个独立夹具）。
9. **GIVEN** `icon` 为空，**WHEN** 校验，**THEN** Ready，校验报告的 warnings 计数为 1、errors 计数为 0，并记录一条 WARNING 级日志。
10. **GIVEN** 升级 `cost=-1`，**WHEN** 校验，**THEN** Failed；`max_level=0` 时 **THEN** Failed；`max_level=1` 时 **THEN** Ready。
11. **GIVEN** `effect_value` 为 0、-1 或 NaN（三个夹具），**WHEN** 校验，**THEN** Failed。
12. **GIVEN** `effect_type=STOMACH_CAPACITY` 且 `effect_value=1.5`，**WHEN** 校验，**THEN** Failed；`effect_value=2.0` 时 **THEN** Ready。
13. **GIVEN** `effect_type=MENU_VISION` 且 `effect_value=2.5`，**WHEN** 校验，**THEN** Failed；`=3.0` 时 **THEN** Ready。
14. **GIVEN** `prerequisite_id` 指向不存在的升级，**WHEN** 校验，**THEN** Failed；**GIVEN** 前置链 A→B→A，**WHEN** 校验，**THEN** Failed，报错列出环上的全部 id。
15. **GIVEN** `entry_fee=-1` 或 `unlock_cost=-1`（各一夹具），**WHEN** 校验，**THEN** Failed。
16. **GIVEN** 档口 `dish_pool=[]`，**WHEN** 校验，**THEN** Failed；**GIVEN** 池中含不存在的 id，**THEN** Failed；**GIVEN** 池中菜的 `station_id` 指向别的档口，**THEN** Failed。
17. **GIVEN** 1 条坏记录 + 99 条好记录，**WHEN** 校验，**THEN** 整体 Failed，且 99 条好记录也取不到——证明不跳过、不裁剪。
18. **GIVEN** 坏数据使状态进入 Failed，**WHEN** 再次调用加载，**THEN** 仍为 Failed，主场景从未实例化。
19. **GIVEN** 已 Ready，**WHEN** 对 `get_dish("rice").price` 赋新值，**THEN** 操作被拒绝或抛错，再次读取仍为原值；两次 `get_dish("rice")` 返回同一实例（`is_same()` 为 true）。
20. **GIVEN** 夹具含 12 条 dish、2 个档口（池分别 5 / 7 条），**WHEN** 查询派生量，**THEN** `dish_count=12`，每个 `dishes_per_station(s)` 等于该档口 `dish_pool.size()`，前置链 A→B→C 的 `prereq_depth=2`。
21. **GIVEN** `price` 写成 `"abc"` 或 `1e30`（超出 int 范围），**WHEN** 解析，**THEN** Failed，日志含文件路径与字段名。
22. **GIVEN** 300 条记录的夹具，**WHEN** 冷启动校验，**THEN** Validating 阶段 p95 ≤ 100 ms（最小规格 PC、非调试构建）。该预算依赖记录条数、磁盘 I/O 与构建类型，超预算时用同一夹具重测确认。
23. **GIVEN** 把某道菜的 `price` 从 8 改成 12（只改 `.tres`，不改代码），**WHEN** 启动并进入该档口，**THEN** 读取到的价格与结算都使用 12——证明数值确实来自数据层。
24. **GIVEN** 仓库中已发布的 id 清单（首个发布版时生成并纳入版本管理），**WHEN** 对比当前数据，**THEN** 没有任何已发布 id 被删除或改名；新增 id 不受限制。

**无法直接断言、已改写或转为开放问题的规则**：
- 「不得硬编码平衡数字」：静态扫描需要先定义"平衡字面量"的白名单，转为第 23 条的行为验证；静态规则列为 Open Question。
- 「id 发布后不可变」：需要已发布基线，第 24 条依赖首个发布版生成的 id 清单。
- 「并发读取」：GDScript 默认单线程，改为第 19 条的重复读取同一实例。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | 菜品、升级、档口的数量上限各是多少？影响美术工时与启动加载时间。 | game-designer + 用户 | 原型阶段（2026-11-17 前） |
| 2 | `prereq_depth` 允许的最大层级？ | game-designer | 原型阶段（2026-11-17 前） |
| 3 | 编辑器内热重载 `.tres`（不重启游戏）要不要进 MVP？倾向放 Vertical Slice。 | user | 原型之后 |
| 4 | 发布版中数据是随包的 `.tres`，还是导出为二进制资源？ | technical-director | 架构阶段（`/architecture-decision`） |
| 5 | 数据文件的目录组织（按类型分文件夹还是按档口分）？ | user | 实现阶段（ADR） |
| 6 | `display_name` 是否要改为多语言 key？首版只做中文，倾向 Full Vision 再处理。 | ux-designer | Full Vision 之前 |
