# ADR-0017: 解锁与档口购买事务

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

本 ADR 把 `unlocks-and-stations.md` 固定成一套**推导优先、单调账本、单事务扣款**的实现契约：`Unlocks` 是 `RefCounted`，档口可用性由「`unlock_cost = 0` 或 id 在持久化集合中」实时推导，`unlocks.dishes` 只是账本而非第二道闸门；载入时把免费档口与它们的菜回填进内存账本并标 dirty，可用性不等待写盘；付费解锁通过 `MetaUpgrades.spend(amount, STATION_UNLOCK, {unlocks: …})` 把扣款与账本放进**同一次 `commit`**，失败一起回滚。能力开关 `supports_station_selection()` 未上线时，付费档口隐藏、不可买、不可进，但已写入的 id 永不删除——门禁只让解锁暂时不生效。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Feature |
| **Layer** | Feature |
| **Knowledge Risk** | LOW — 纯 `StringName` / `Array` / `Dictionary` 集合运算与排序；无浮点、无时钟、无随机、无文件 I/O |
| **References Consulted** | `design/gdd/unlocks-and-stations.md`、`design/gdd/meta-upgrades.md`、`design/gdd/save-system.md`、`design/gdd/data-config.md`、`design/gdd/ui-flow.md`、`docs/architecture/architecture.md`、`docs/architecture/adr-0002-persistence-format.md`、`docs/architecture/adr-0016-meta-upgrades-wallet.md`、`docs/engine-reference/godot/VERSION.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① `Array[StringName]` 的 `sort()` 在 4.7.2 上按字符串字节序稳定排序；② `Dictionary` / `Array` 的 `duplicate(true)` 深拷贝后修改返回值不影响内部账本；③ `StringName` 与 `String` 的相等/排序在 id 集合中一致；④ 同一数据与同一账本连续 100 次规范化逐项相同；⑤ `get("unlock_requires")` 在字段不存在时返回 `null` 而不报错；⑥ `MetaUpgrades.spend` 失败时 `extra` 命名空间零生效 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述六条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0004（服务组合）— `Unlocks` 由 `AppRoot` 构造，`Save` / `MetaUpgrades` 先于它构造；ADR-0001（数据配置）— `StationData` / `DishData` 与 `Ready`；ADR-0002（存档格式）— `unlocks` 命名空间、`mark_dirty`、`commit`；ADR-0016（局外升级）— `spend(amount, reason, extra)`、`get_wallet`、`get_level` |
| **Enables** | ADR-0019（HUD）的档口选择区与升级页；界面流程的档口目录与解锁确认；`RunConfig.station_id` 的授权来源 |
| **Blocks** | `TR-unlocks-001..005`；档口目录、解锁事务、进度显示、能力门禁的全部实现 story |
| **Ordering Note** | 本 ADR 不定义 `StationData` schema（ADR-0001）、不定义钱包（ADR-0016）、不定义存档原子写（ADR-0002）、不定义档口选择器 UI（ADR-0019 / 界面流程）。它只定义可用性、规范化账本、解锁校验与成交编排。 |

## Context

### Problem Statement

GDD 已经给出规则，但实现层必须回答四个问题：

1. **空存档不能等于没有可玩档口。** 免费档口必须由数据推导，不能等一次写盘；否则新档或 `Blocked` 存档会卡在「没有档口」。
2. **`unlocks.dishes` 不能成为第二道闸门。** 如果可用性同时检查档口集合与菜集合，任何一次漏写都会制造「档口已解锁、菜不可吃」的不可玩状态，且只能靠额外写盘修复。
3. **扣款与账本必须同一笔事务。** 分别写 `progression` 与 `unlocks` 会出现「钱扣了、档口没开」或反之；失败必须一起回滚。
4. **门禁是能力开关，不是数据删除。** 档口选择器未上线时，付费档口要隐藏，但旧存档里已买的 id 不能丢；开关恢复后必须立即可用。

### Current State

`src/` 只有占位文件。`architecture.md` 尚未列出 `Unlocks` 服务；ADR-0016 已把 `spend(amount, reason, extra)` 作为唯一扣款入口。缺的是：可用性/规范化公式的唯一实现、载入结果四态、九步校验顺序、`spend` 失败映射，以及能力门禁在运行期的实时求值。

### Constraints

- **推导优先**：可用性 = `unlock_cost = 0 ∨ id ∈ persisted.stations`，再叠加门禁；不依赖 `dishes`。
- **账本单调**：不删除任何 id；未知 id 保留；唯一清空路径是 `delete_career`。
- **一次事务**：`progression.wallet` 与 `unlocks` 同一 `commit`，不拆写。
- **先落盘再确认**：`spend` 成功才替换内存账本、才向界面返回成交。
- **只读查询**：所有目录/可用性/进度查询不改变状态，返回新副本。
- **确定性**：所有集合输出按 id 升序；不读时钟、不用随机、不做 I/O。
- **门禁实时**：`supports_station_selection()` 每次查询都重新求值，不缓存载入时的开关。
- **单人单线程**：同一时刻只允许一笔解锁在途；不排队、不合并。

### Requirements

- 必须提供 `get_load_outcome` / `get_station_catalog` / `get_available_stations` / `is_station_available` / `get_available_dishes` / `can_unlock` / `unlock` / `get_unlock_progress` / `get_effective_unlocks` / `normalize`。
- 必须在载入时回填免费档口与菜，并把差异标记 dirty。
- 必须按固定顺序返回九类成交前校验错误与两类事务结果。
- 必须把 `unlocks` 作为 `spend` 的 `extra` 并入同一 `commit`。
- 必须让能力门禁同时影响目录、可用性、进度分母与 `can_unlock` / `unlock`。
- 必须让未知 id、重复 id、空集合与 `NoPlayableStation` 都有确定行为。

## Decision

### 1. 模块形态与依赖

`Unlocks` 是 `RefCounted`，位于 Feature 层：

- 构造参数：`data: DataConfig`、`save: SaveSystem`、`meta: MetaUpgrades`、`is_menu_active: Callable`、`supports_station_selection: Callable`。
- 不持有 `UIRoot`、`RunFlow`、`Records` 或 `AppRoot`。
- 唯一可变状态：`_persisted: {stations: Array[StringName], dishes: Array[StringName]}`、`_effective: 同形`、`_unlocking: bool`、`_writable: bool`、`_initialized: bool`、`_backfilled_at_load: bool`。
- 不注册 `_process`、不读墙钟、不直接写文件；写盘只经 `MetaUpgrades.spend`（解锁）或 `SaveSystem.mark_dirty`（回填）。
- 信号：`unlocks_changed(progress)`（成功解锁或回填改变账本后发一次；被拒绝的解锁不发）。

```gdscript
class_name Unlocks extends RefCounted

func initialize() -> InitResult
func notify_save_status(status: SaveStatus) -> void
func reset_after_career_delete() -> void

func get_load_outcome() -> LoadOutcome
func get_station_catalog() -> Array[StationView]
func get_available_stations() -> Array[StringName]
func is_station_available(id: StringName) -> bool
func get_available_dishes(station_id: StringName) -> Array[StringName]
func can_unlock(id: StringName) -> UnlockCheck
func unlock(id: StringName) -> UnlockResult
func get_unlock_progress() -> Dictionary
func get_effective_unlocks() -> Dictionary
func normalize(source: Dictionary) -> Dictionary
```

**不提供**：`add_station`、`remove_station`、`set_unlocks`、`grant`。解锁只有 `unlock` 与载入回填两条写入路径。

### 2. 载入、初始化与回填

`initialize()` 在 `Save.load()` 与 `MetaUpgrades.initialize()` 之后由 `AppRoot` 调用一次：

1. 数据未 `Ready` → `_initialized = true`，`get_load_outcome() = Unavailable`；目录为空、`unlock` 返回 `DATA_NOT_READY`。
2. 读 `save.get_namespace(&"unlocks")`：`stations` / `dishes` 必须是数组；元素必须是 `StringName` 或 `String`，否则 `DATA_NOT_READY`（不默认空）。
3. 去重、转 `StringName`、按 id 升序，得到 `_persisted`。
4. `_effective = normalize(_persisted)`。
5. `_backfilled_at_load = (_effective != _persisted)`。
6. 若 `_backfilled_at_load`：内存立即采用 `_effective`；`_writable` 时 `save.mark_dirty(&"unlocks")`（不立即 commit），`Blocked` 时不标 dirty、每次载入重新推导；发一次 `unlocks_changed`。
7. `_writable` 由 `SaveStatus` 决定，后续由 `notify_save_status` 更新。

**新建生涯**：`BootSequence` 在写首个基线之前调用 `normalize({})`，把结果随基线一起提交；`initialize()` 随后读到的持久化集合已含免费档口与菜。若基线提交失败，按 ADR-0002 的启动失败处理，不进入游戏。

**未知 id**：保留在 `_persisted` 与 `_effective` 中，原样写回；不计入进度分母/分子，不出现在目录。

**重复 id**：存档结构校验应拒绝；运行期遇到时去重，不报错。

### 3. 可用性与门禁

```gdscript
gated(s) = (supports_station_selection.call() != true) and (s.unlock_cost > 0)
station_available(s) = not gated(s) and (s.unlock_cost == 0 or s.id in _persisted.stations)
```

- `supports_station_selection` **每次查询都调用**；开关从 `false → true` 时，被门禁的档口立即恢复规则判定，无需重新载入。
- 未知 id → `false`，不抛错。
- `unlock_cost < 0` 属于损坏数据：可用性按 `> 0` 判定会把负价当免费；本 ADR 在 `can_unlock` / `unlock` 中以 `INVALID_STATION_DATA` 拦截，并在 `is_station_available` 中返回 `false`（防御）。
- 免费档口不受门禁影响：`unlock_cost = 0` → `gated` 恒为 `false`。
- 已解锁的付费档口在门禁生效时 `station_available = false`，但 id 仍在账本中。

### 4. 账本规范化 `normalize`

```gdscript
func normalize(source: Dictionary) -> Dictionary
# 返回 {stations: Array[StringName], dishes: Array[StringName]}，均升序、去重
```

- `S_free = {s : s.unlock_cost == 0}`
- `S_eff = S_free ∪ {s : s.id ∈ source.stations ∧ not gated(s)}`
- `stations = sort(source.stations ∪ {s.id : s ∈ S_free})`
  - **保留全部来源 id**（包括未知 id 与被门禁的已知 id），满足「只增不减」`stations ⊇ source.stations`。
- `dishes = sort(source.dishes ∪ ⋃_{s ∈ S_eff} s.dish_pool)`
  - 只有 `S_eff` 的菜进入账本；被门禁的付费档口即使已在 `source.stations` 中，也不把菜写进 `dishes`。
- 未知档口 id / 未知菜 id 原样保留。
- 结果只依赖数据与 `source`，不读时钟、不用随机、不做 I/O。

> **与 GDD 公式的差异**：GDD 的 `effective_unlocks` 写成 `U.stations \ S_ids`（只保留未知 id），会在门禁生效时丢掉已知的已购档口 id，与规则 17、AC 44 的「id 仍留在账本」冲突。本 ADR 采用 `U.stations` 全量保留，详见 GDD Sync Check。

### 5. 载入结果与实时查询

`get_load_outcome()` 每次实时计算：

| 条件 | 结果 |
|---|---|
| 数据未 `Ready` | `Unavailable` |
| 可用档口集合为空（含所有档口被门禁） | `NoPlayableStation` |
| `_backfilled_at_load == true` | `Backfilled` |
| 否则 | `Playable` |

- `get_available_stations()`：可见且可用的档口 id，升序。
- `get_available_dishes(station_id)`：不可用或被门禁 → `[]`；否则返回 `dish_pool` 的排序副本。
- `get_station_catalog()`：**未门禁**的全部档口，升序；每项含 `id`、`display_name_key`、`available`、`unlock_cost`、`lock_reason`、`dish_count`。被门禁的付费档口不出现。
- `get_unlock_progress()`：分母只统计未门禁档口；分子只统计其中免费或已解锁的档口；`dishes` 同理按 `dish_pool` 计数。未知 id 不计入任何一边。
- `get_effective_unlocks()`：返回 `_effective` 的深拷贝（新数组）。
- 所有查询返回新副本；调用方修改不影响内部状态。

### 6. 解锁校验顺序

`can_unlock(id)` 与 `unlock(id)` 共用同一套判定，顺序**固定**，第一个失败即返回：

1. `DATA_NOT_READY`：数据未 `Ready`。
2. `NOT_WRITABLE`：`_writable == false`。
3. `NOT_IN_MENU`：`is_menu_active.call() != true`。
4. `UNKNOWN_STATION`：id 不在数据目录。
5. `INVALID_STATION_DATA`：`unlock_cost < 0` 或非整数。
6. `FEATURE_GATED`：`gated(s)`。
7. `BUSY`：`_unlocking == true`。
8. `ALREADY_UNLOCKED`：`station_available(s)`。
9. `PREREQUISITE_LOCKED`：`requires(s)` 中任一升级等级 `< 1`。
10. `INSUFFICIENT_FUNDS`：`meta.get_wallet() < s.unlock_cost`。

- `can_unlock` 返回 `{ok, reason, cost}`；`ALREADY_UNLOCKED` / `FEATURE_GATED` / 未知 / 数据错误时 `cost = null`。
- `can_unlock` 不改变状态、不调用 `spend`、不 `commit`；`BUSY` 也按同一顺序返回。
- `requires(s)`：`StationData.unlock_requires` 当前不存在；用 `s.get("unlock_requires")`，缺失或非数组时按 `[]` 处理。字段落地后无需改本 ADR。
- 负价格必须排在 `FEATURE_GATED` 与 `INSUFFICIENT_FUNDS` 之前：否则负价会在门禁下被掩盖，或在 `wallet ≥ 负价` 时被判成可成交并反向加钱。

### 7. 解锁事务

`unlock(id)` 通过全部校验后：

1. `new_ledger = normalize({stations: _persisted.stations + [id], dishes: _persisted.dishes})`。
2. `_unlocking = true`。
3. `result = meta.spend(s.unlock_cost, SpendReason.STATION_UNLOCK, {&"unlocks": new_ledger})`。
4. `result == SPENT`：`_persisted = new_ledger`、`_effective = new_ledger`、`_unlocking = false`；发 `unlocks_changed`；返回 `OK{cost, wallet_after}`。
5. `result != SPENT`：`_unlocking = false`；内存账本保持旧值；把 `spend` 的错误映射为：
   - `INSUFFICIENT_FUNDS` / `NOT_WRITABLE` / `NOT_IN_MENU` / `BUSY` / `INVALID_AMOUNT` 原样返回；
   - `COMMIT_FAILED` 原样返回；
   - `RESERVED_NAMESPACE` 视为实现 bug，记 ERROR，返回 `COMMIT_FAILED`（不重试）。

- 成交只在 `spend` 返回 `SPENT` 后确认；界面不得在等待期间显示已解锁。
- `spend` 内部会把 `{progression: 完整新值}` 与 `{unlocks: new_ledger}` 放进**同一次** `commit`；本服务不调用 `commit`。
- 写盘失败时 `MetaUpgrades` 回滚钱包，本服务不替换账本；两者一起保持解锁前状态。
- 解锁不消耗回合、不改当前局、不参与结算事务。

### 8. 能力门禁（S8）

- 门禁是**能力开关**，不是价格或数据改动：`station_banquet.unlock_cost` 仍为 120，`StationData` 不新增字段。
- `supports_station_selection() = false` 时：
  - 付费档口不出现在 `get_station_catalog()` 与 `get_available_stations()`；
  - 不计入 `get_unlock_progress()` 的分子与分母；
  - `is_station_available = false`、`get_available_dishes = []`；
  - `can_unlock = {ok: false, reason: FEATURE_GATED, cost: null}`；`unlock` 返回 `FEATURE_GATED`，`spend` 与 `commit` 均不调用。
- 开关变 `true` 后，已写入账本的付费档口立即可用；不需要重新购买、不退款。
- 门禁期间的 `unlocks` 回填仍保留 id；不因开关状态删除任何条目。
- 界面流程必须用同一开关决定是否绘制档口列表与是否允许 `RunConfig.station_id` 取非免费档口；`RunFlow` 的存在性校验不是授权校验。

### 9. 生涯删除与状态

- `reset_after_career_delete()`：`_persisted = {[], []}`、`_effective = normalize(∅)`、`_backfilled_at_load = true`、`_unlocking = false`；发一次 `unlocks_changed`。不重读旧档、不保留缓存。
- `notify_save_status(status)`：更新 `_writable`；从 `ReadOnly` 恢复 `Ready` 时不做任何写入，只允许后续购买。
- 状态：`Uninitialized → Unavailable / Ready / ReadOnly`；`Ready → Unlocking → Ready`；`ReadOnly` 只影响 `unlock`，不影响查询。

### 10. 事务与查询图

```
DataConfig (Ready) ── StationData / DishData
        │
        ▼
Unlocks (RefCounted, 推导 + 单调账本)
   ├─ normalize(source) ──▶ {stations, dishes}（升序、保留未知 id）
   ├─ get_load_outcome / catalog / available / progress / effective
   ├─ can_unlock(id) ── 九步校验，只读
   ├─ unlock(id) ── normalize + MetaUpgrades.spend(amount, STATION_UNLOCK, {unlocks})
   ├─ initialize / notify_save_status / reset_after_career_delete
   └─ unlocks_changed(progress)
              │
              ├─▶ SaveSystem（经 spend 的同一次 commit；回填走 mark_dirty）
              ├─▶ UIRoot / 界面流程（目录、锁定原因、进度）
              └─▶ RunConfig.station_id（只允许可用档口）
```

## Alternatives Considered

### Alternative 1: 可用性 = id 在持久化集合中

- **Description**: 免费档口也必须在集合里。
- **Pros**: 规则单一。
- **Cons**: 空存档与 `Blocked` 存档没有可玩档口；需要一次写盘才能开始，违背「可用性不等待写盘」。
- **Rejection Reason**: 免费档口由数据推导，集合只是账本。

### Alternative 2: 菜也作为第二道闸门

- **Description**: 档口可用且菜 id 在 `unlocks.dishes` 中才可吃。
- **Pros**: 支持逐菜解锁。
- **Cons**: 任何漏写都会制造「档口开了、菜不能吃」的不可玩状态；需要额外写盘修复；MVP 不解锁单菜。
- **Rejection Reason**: 可用性跟随档口；`dishes` 只做账本与计数。

### Alternative 3: 解锁自己 `commit`

- **Description**: 本服务先写 `unlocks`，再让局外升级扣钱。
- **Pros**: 职责看起来更独立。
- **Cons**: 两次写盘；中途失败会出现半成交；违反「同一事务」。
- **Rejection Reason**: 只经 `MetaUpgrades.spend` 合并提交。

### Alternative 4: 先扣款再规范化账本

- **Description**: 先 `spend`，成功后再计算新集合。
- **Pros**: 扣款逻辑更早失败。
- **Cons**: `spend` 需要 `extra` 才能把账本并入同一 commit；顺序反了就必须两次写。
- **Rejection Reason**: 先算 `new_ledger`，再交给 `spend`。

### Alternative 5: 载入时清理未知 id

- **Description**: 只保留数据中存在的档口与菜。
- **Pros**: 账本干净。
- **Cons**: 数据回滚或 id 改名会丢失玩家已买的解锁；违反单调性。
- **Rejection Reason**: 未知 id 原样保留，不计入进度。

### Alternative 6: 缓存载入时的可用性

- **Description**: 门禁结果在启动时算一次。
- **Pros**: 查询更快。
- **Cons**: 能力开关在会话中变化时，已购档口不会立即恢复；与规则 17 冲突。
- **Rejection Reason**: 每次查询实时求值。

### Alternative 7: 解锁排队而不是 `BUSY`

- **Description**: 第二笔解锁等待第一笔完成。
- **Pros**: 玩家连点也能成功。
- **Cons**: 需要异步队列与 UI 状态；GDD 明确「不排队、不合并」。
- **Rejection Reason**: 单笔在途，第二笔返回 `BUSY`。

## Consequences

### Positive

- 空集合、`Blocked` 存档与新档都能立即进入免费档口，不依赖写盘。
- `dishes` 永远不会成为不可玩状态的来源。
- 扣款与账本同事务，崩溃后不会出现半成交。
- 门禁实时求值，能力开关上线/下线都不丢已购解锁。
- 未知 id 与数据改版不会静默删除玩家资产。

### Negative

- 每次查询都要重新推导可用性与门禁，查询成本随档口数增长（MVP 很小）。
- 回填在 `Blocked` 下只在内存生效，每次载入都要重算；免费档口改价存在窄窗口。
- `unlock_requires` 字段尚未落地，前置校验当前恒为空集。
- `unlock` 依赖 `MetaUpgrades.spend` 的错误词表；两者必须同步维护。

## Risks

- **门禁与账本冲突**：本 ADR 全量保留 `source.stations`；用 AC 44 回归。
- **回填重复发信号**：只在 `_backfilled_at_load` 为真且集合实际变化时发一次；重复载入不叠加。
- **`Blocked` 下回填丢失**：不标 dirty、每次重算；文档化为已知窗口。
- **负价格绕过**：`INVALID_STATION_DATA` 排在门禁与资金之前；负价永不进入 `spend`。
- **`spend` 与购买并发**：两者共用 `MetaUpgrades` 的在途约束；本服务另有 `_unlocking` 防重入。
- **进度分母随数据/开关变化**：每次实时计算，不缓存旧分母。
- **未知 id 计入进度**：明确排除，只统计数据中存在的记录。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `unlocks-and-stations.md` | `TR-unlocks-001`：可用性由 `unlock_cost = 0` 或 id 在 `unlocks.stations` 推导；菜跟随档口 | 决策 3、4；`station_available` 与 `normalize` |
| `unlocks-and-stations.md` | `TR-unlocks-002`：规范化/回填免费档口与菜；空集合 → `NoPlayableStation`；不删除 id | 决策 2、4、5；回填、`mark_dirty`、四态载入结果 |
| `unlocks-and-stations.md` | `TR-unlocks-003`：解锁经 `meta-upgrades.spend` 并把 `{unlocks}` 并入同一 commit | 决策 7；`spend(..., {unlocks: new_ledger})` |
| `unlocks-and-stations.md` | `TR-unlocks-004`：固定校验顺序含 `NOT_IN_MENU`、`BUSY`、`INSUFFICIENT_FUNDS` | 决策 6；九步顺序 |
| `unlocks-and-stations.md` | `TR-unlocks-005`：目录/进度/有效解锁读 API；id 升序 | 决策 5；所有查询返回排序副本 |
| `unlocks-and-stations.md` | Core Rule 2 / 3：可用性推导；菜跟随档口、`dishes` 不是闸门 | 决策 3、4；`dishes` 只入账本 |
| `unlocks-and-stations.md` | Core Rule 4 / 5：至少一个免费档口兜底；解锁单调、不删 id | 决策 2、4、5、9；`NoPlayableStation` 与保留未知 id |
| `unlocks-and-stations.md` | Core Rule 6 / 7：先落盘再确认；钱包只由局外升级写 | 决策 7；`spend` 成功后才替换账本 |
| `unlocks-and-stations.md` | Core Rule 8 / 9：解锁只在菜单；固定校验顺序 | 决策 6；`NOT_IN_MENU` 与九步顺序 |
| `unlocks-and-stations.md` | Core Rule 10 / 11：价格固定为 `unlock_cost`；前置是等级门槛 | 决策 6；`requires(s)` 按空列表处理 |
| `unlocks-and-stations.md` | Core Rule 12 / 13：数据未就绪不可用；存档不可写只读 | 决策 2、5、6；`DATA_NOT_READY` / `NOT_WRITABLE` |
| `unlocks-and-stations.md` | Core Rule 14：回填内存优先、写盘跟随现有写入时点 | 决策 2；新建生涯基线 + 既有生涯 `mark_dirty` |
| `unlocks-and-stations.md` | Core Rule 15 / 16：不参与结算事务；确定性、无副作用 | 决策 1、5、7；无结算调用、排序输出 |
| `unlocks-and-stations.md` | Core Rule 17：付费档口受选择器能力门禁 | 决策 8；实时 `supports_station_selection` |
| `unlocks-and-stations.md` | Formulas：`station_available` / `effective_unlocks` / `unlock_decision` / `station_unlock_cost` / `unlock_progress` | 决策 3–6；逐式落地（`effective_unlocks` 保留全量 id） |
| `unlocks-and-stations.md` | AC 1–45（载入、回填、事务、门禁、进度、确定性） | 决策 2–9；测试矩阵见 Validation |

## Performance Implications

- **CPU**：`normalize` / 目录 / 进度为 O(S + D)（MVP 1 档口 12 菜）；单次 < 0.1 ms。`unlock` 的主要成本是同步磁盘提交。
- **Memory**：两份 id 数组，MVP < 2 KiB；查询返回副本，最多 24 项。
- **Load Time**：`initialize` 读取一个命名空间并做一次规范化；不参与启动门。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 定义 `LoadOutcome` / `StationView` / `UnlockCheck` / `UnlockResult` / 错误词表。
2. 实现 `normalize` / `station_available` / `get_effective_unlocks` 与纯函数测试（AC 1–10、36）。
3. 实现 `initialize` / 回填 / `mark_dirty` / `notify_save_status`（AC 1–3、6、17）。
4. 实现目录、可用列表、进度与门禁（AC 4、5、17、32、42–45）。
5. 实现 `can_unlock` 九步校验与 `unlock` 的 `spend` 编排（AC 11–26、37、39）。
6. 实现 `reset_after_career_delete`（AC 31）。
7. 接入界面流程的档口选择区与 `RunConfig.station_id` 授权（ADR-0019）。

## Validation Criteria

- GDD AC 1–45 全部通过；重点：AC 1/2 回填与写盘时机、AC 7/8 菜跟随档口、AC 11/12 同事务与资金不足、AC 20/21 `BUSY` 与回滚、AC 23–26 顺序、AC 42–45 门禁。
- 静态检查：`Unlocks` 不引用 `progression` 命名空间、不直接调用 `Save.commit`、不写 `wallet`。
- 事务：`unlock` 成功后 `commit` 恰好 1 次，载荷同时含 `progression` 与 `unlocks`；失败时钱包与账本逐字段回到解锁前。
- 单调性：任意数据改版与载入序列后，`_effective.stations ⊇ _persisted.stations`、`_effective.dishes ⊇ _persisted.dishes`。
- 门禁：开关 `false → true → false` 三轮后，已购 id 始终保留，可用性随开关切换；`spend` 在门禁期间调用 0 次。
- 确定性：同一数据与账本 100 次规范化，数组逐项相同且升序。
- 副本：修改 `get_effective_unlocks()` / `get_available_dishes()` 的返回值不影响内部状态。

## GDD Sync Check

- **`effective_unlocks` 公式与 AC 44 冲突**：GDD 公式用 `U.stations \ S_ids`，会丢弃被门禁的已知已购档口 id；规则 17、Output Range 与 AC 44 要求保留。本 ADR 采用 `U.stations` 全量保留，`S_eff` 仍排除被门禁档口（不影响可用性、菜账本与进度）。建议把 GDD 公式的 `stations` 行改为 `sort({s.id : s ∈ S_eff} ∪ U.stations)`。
- **`can_unlock` 是否包含 `BUSY` / `NOT_WRITABLE`**：GDD 未逐项写；本 ADR 让 `can_unlock` 与 `unlock` 共用完整顺序，便于界面稳定显示原因。属实现层细化，不需回改。
- **`unlock_requires`**：GDD 明确当前字段不存在、按空列表处理；本 ADR 用 `get()` 兼容未来字段，无需 GDD 改动。

## Related

- ADR-0001 数据配置 — `StationData.unlock_cost` / `dish_pool` / `display_name`
- ADR-0002 存档格式 — `unlocks` 命名空间、`mark_dirty`、原子 `commit`
- ADR-0004 服务组合 — `AppRoot` 构造与能力开关 `Callable`
- ADR-0016 局外升级 — `spend(amount, reason, extra)`、`get_wallet`、`get_level`
- ADR-0019 HUD 数据绑定 — 档口选择区、锁定态与进度
- `design/gdd/unlocks-and-stations.md`
- `design/gdd/meta-upgrades.md`
