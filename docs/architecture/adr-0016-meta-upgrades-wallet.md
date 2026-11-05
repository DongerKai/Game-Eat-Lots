# ADR-0016: 局外升级树、钱包事务与升级快照

## Status

Accepted

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。其他 agent 与任何 skill 都不能自行设置。**

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

user + agents（lean 模式；本 ADR 由编排会话起草，待用户接受）

## Summary

本 ADR 把 `meta-upgrades.md` 固定成一套**单写者、三流水、两阶段**的实现契约：`MetaUpgrades` 是 `RefCounted`，内存中持有 `wallet` 与 `upgrade_levels` 的唯一副本，钱包只允许「结算入账 / 升级购买 / 档口解锁」三种变化；价格按 `ceil(cost × cost_growth^(L−1))` 在最后一次取整，效果按等级线性累加；`resolve_upgrade_snapshot()` 每局返回一份含三个已解析值与冻结 `levels` 副本的新字典。购买与解锁都走「校验 → 内存应用 → 一次 `Save.commit`（完整新值）→ 成功才确认；失败回滚」；结算入账则走 ADR-0007 的「只算内存 → 编排者合并提交 → 写失败保留内存并标 dirty」。没有任何路径能直接改写钱包。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Feature |
| **Layer** | Feature |
| **Knowledge Risk** | MEDIUM — 价格曲线使用 `pow` / `ceil` / `round` 的 float64 路径，跨平台边界取整必须实测；其余为纯 `int` / `float` / `Dictionary` 运算 |
| **References Consulted** | `design/gdd/meta-upgrades.md`、`design/gdd/data-config.md`、`design/gdd/save-system.md`、`design/gdd/settlement-and-entry-fee.md`、`design/gdd/unlocks-and-stations.md`、`docs/architecture/architecture.md`、`docs/architecture/adr-0007-run-state-settlement-transaction.md`、`docs/engine-reference/godot/VERSION.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① `ceil(12 × 1.35^4) = 40`、`ceil(12 × 1.35^3) = 30`、`ceil(12 × 1.35^2) = 22` 在 4.7.2 上逐级成立；② `cost_growth = 1.5`、`cost = 16` 这类数学整值不会被浮点误差抬成 +1，整值吸附阈值 `1e-9` 足够；③ `int` 的 int64 加/减在边界上配合前置比较可阻止回绕；④ `Dictionary` 深拷贝后修改返回值不影响内部状态；⑤ 同一等级表 100 次解析逐字段相同；⑥ `Save.commit` 同步返回，`BUSY` 只在重入回调中出现 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述六条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0004（服务组合）— `MetaUpgrades` 由 `AppRoot` 构造，`Save` 先于它构造；ADR-0002（存档格式）— `progression.wallet` / `progression.upgrade_levels` 与原子 `commit`；ADR-0001（数据配置）— `UpgradeData` / `StationData.wave_size` 与 `Ready` 门；ADR-0007（结算事务）— `prepare_run_result` 的调用时机、合并提交与 `WALLET_OVERFLOW` 的缩键策略；ADR-0015（结算纯函数）— `RunResult.net_profit` / `run_id` |
| **Enables** | ADR-0017（解锁与档口）的 `spend(amount, reason, extra)`；ADR-0019（HUD）的局外页与钱包显示；首局引导的升级推荐；`RunConfig.upgrade_snapshot` 的生产 |
| **Blocks** | `TR-meta-upgrades-001` / `002` / `003` / `004` / `006` / `007`；钱包、购买、升级页、快照的全部实现 story |
| **Ordering Note** | 本 ADR 不定义 `UpgradeData` schema（ADR-0001）、不定义存档原子写（ADR-0002）、不定义结算事务编排（ADR-0007）、不定义档口目录与解锁集合（ADR-0017）。它只定义钱包语义、价格曲线、等级→修正值解析与两阶段事务的内存/磁盘边界。 |

## Context

### Problem Statement

GDD 已经给出规则，但实现层必须回答四个问题：

1. **钱包只有一个写者，且只有三种流水。** 任何下游直接改 `progression.wallet` 都会让「先落盘再确认」失效；也必须禁止第四种流水（利息、补偿、退款）悄悄出现。
2. **价格与等级必须可复现。** 价格是浮点指数、效果是整数线性；逐级取整会漂移，等级被下游自行换算又会复制一套规则。
3. **购买与入账的失败策略不同。** 购买失败必须回滚内存（玩家没买到）；结算入账失败必须保留内存并标 dirty（玩家已经打完这一局），两者不能共用一套“失败就回滚”。
4. **快照是一次冻结的副本。** 局内不能购买、不能中途升级；快照被下游修改也不得污染生涯状态，未知 id 必须保留但不贡献效果。

### Current State

`src/` 只有占位文件。`architecture.md` 已列出 `MetaUpgrades.resolve_upgrade_snapshot` / `purchase` / `prepare_run_result` / `spend`；ADR-0007 已固定 `prepare_run_result` 的输出形状与事务合并规则。缺的是：钱包三流水的唯一入口、价格/效果/上限的封闭公式、购买与 `spend` 的校验顺序与回滚、`progression_changed` 的发射边界，以及 `levels` 副本的深拷贝语义。

### Constraints

- **单写者**：只有 `MetaUpgrades` 能改 `wallet` 与 `upgrade_levels`；下游只读或调用本服务方法。
- **三流水**：结算入账、购买扣款、解锁扣款；没有第四种。
- **负钱包合法**：不设玩法上下限；只有 int64 边界，溢出返回 `WALLET_OVERFLOW`。
- **购买先落盘再确认**：`commit` 成功前不得向界面返回成交；失败回滚内存。
- **结算入账不回滚**：写失败保留内存、标 dirty、返回未保存状态；由 ADR-0007 报告 `APPLIED_UNSAVED`。
- **效果线性、价格指数**：每级效果常量；价格只在最后一次 `ceil`。
- **硬上限**：`max_level`；`MENU_VISION` 另有 `menu_vision_useful_max`。
- **快照冻结**：每局一次解析，返回新副本；局内等级变化不影响当前局。
- **无信号依赖顺序**：`progression_changed` 是过去式通知，不是跨系统同步契约。
- **单人单线程**：`commit` 同步完成，`BUSY` 只做重入防御。

### Requirements

- 必须提供 `get_wallet` / `get_level` / `next_price` / `can_purchase` / `purchase` / `spend` / `prepare_run_result` / `resolve_upgrade_snapshot` / `get_catalog`。
- 必须把钱包的全部变化收敛到三个方法，且每个方法都写完整新值、不写增量。
- 必须让 `purchase` 的校验顺序固定，并让失败不改变内存与磁盘。
- 必须让 `WALLET_OVERFLOW` 只丢弃 `progression` 键，纪录仍提交。
- 必须让快照含三个已解析值与 `levels` 副本，且未知 id 保留、等级钳制。
- 必须让 `menu_vision_useful_max` 阻止卖无效等级。

## Decision

### 1. 模块形态与依赖

`MetaUpgrades` 是 `RefCounted`，位于 Feature 层：

- 构造参数：`data: DataConfig`、`save: SaveSystem`、`is_menu_active: Callable`（由 `AppRoot` 绑定到 `SceneFlow.state == Menu` 的只读探针）。
- 不持有 `UIRoot`、`RunFlow`、`Unlocks`、`Records` 或 `AppRoot`。
- 唯一可变状态：`_wallet: int`、`_levels: Dictionary`、`_last_applied_run_id: int`、`_purchasing: bool`、`_writable: bool`、`_initialized: bool`。
- 不注册 `_process`、不读墙钟、不写文件（写盘只经 `SaveSystem.commit`）。
- 信号（过去式通知）：`progression_changed(wallet, levels)`、`purchase_succeeded(id, price, wallet_after)`、`purchase_failed(id, code)`、`run_result_applied(run_id, status, wallet_after)`。

```gdscript
class_name MetaUpgrades extends RefCounted

func initialize() -> InitResult
func notify_save_status(status: SaveStatus) -> void
func reset_after_career_delete() -> void

func get_wallet() -> int
func get_level(id: StringName) -> int
func next_price(id: StringName) -> PriceResult
func can_purchase(id: StringName, station_id: StringName) -> CheckResult
func purchase(id: StringName, station_id: StringName) -> PurchaseResult
func spend(amount: int, reason: SpendReason, extra: Dictionary = {}) -> SpendResult

func prepare_run_result(result: RunResult) -> ProgressionResult
func resolve_upgrade_snapshot() -> Dictionary
func get_catalog(station_id: StringName) -> Array[UpgradeView]
```

`UpgradeView` 是只读值对象（`RefCounted`，字段不可改）：

```gdscript
class UpgradeView extends RefCounted
var id: StringName
var display_name_key: StringName
var description_key: StringName
var level: int
var max_level: int
var current_effect: float        # 已解析效果
var next_effect: Variant         # 已解析；满级时为 null
var next_price: Variant          # int；满级或未知时为 null
var purchasable: bool
var reason: int                  # 与 can_purchase 同一套 RejectCode
```

**不提供**：`set_wallet`、`add_wallet`、`set_level`、`set_levels`、`refund`、`grant`、`reset_wallet`。任何新流水都必须先改 GDD 并新增方法。

### 2. 初始化、可写性与生涯删除

`initialize()` 在 `Save.load()` 完成后由 `AppRoot` 调用一次：

- 读取 `save.get_namespace(&"progression")`。
- `wallet`：必须存在且为 `int`，否则 `DATA_NOT_READY`（不默认 0）；负值合法。
- `upgrade_levels`：必须存在且为 `Dictionary`；值必须是 `int ≥ 0`，否则 `DATA_NOT_READY`。
- 未知 id **保留**在 `_levels` 中，不删除；载入时高于 `max_level` 的等级在读取时钳制，并写 warning（不回写存档、不退钱）。
- `_writable = (save.status.writable == true)`；之后由 `notify_save_status` 更新 `_writable`，本服务不自行轮询。
- 重复 `initialize` 返回 `ALREADY_INITIALIZED`，不重读、不清空。

`reset_after_career_delete()`：在 `Save.delete_career` 成功后由 `AppRoot` 调用；`_wallet = 0`、`_levels = {}`、`_last_applied_run_id = 0`、发一次 `progression_changed`。不重读旧命名空间、不保留缓存。

### 3. 钱包：三条流水与受检算术

| 流水 | 入口 | 方向 |
|---|---|---|
| 一局净利 | `prepare_run_result(result)` | `wallet += result.net_profit` |
| 升级购买 | `purchase(id, station_id)` | `wallet -= price` |
| 档口解锁 | `spend(amount, reason, extra)` | `wallet -= amount` |

- 写入一律是完整新值 `{wallet: W', upgrade_levels: {...}}`，从不提交增量。
- `prepare_run_result` 使用受检加法：`net > 0 && wallet > INT64_MAX − net` 或 `net < 0 && wallet < INT64_MIN − net` → `WALLET_OVERFLOW`；钱包保持原值、不构造 `progression_update`。
- `purchase` / `spend` 先做 `wallet ≥ price` 判定；通过后减法结果必然落在 int64 内，不需额外溢出分支。
- 负钱包合法；`wallet = −12` 时任何正价格都返回 `INSUFFICIENT_FUNDS`。
- 没有利息、签到、救济、退款、入场门槛；没有上下限钳制。

### 4. 价格、效果与硬上限

```gdscript
func upgrade_price(u: UpgradeData, level: int) -> PriceResult   # level 是“将要达到”的等级 L
func upgrade_tree_cost(u: UpgradeData) -> int
```

- `price(u, L) = ceil(cost × cost_growth^(L−1))`，`L ∈ [1, max_level]`。
- **只在最后取整一次**；内部 `raw = float(cost) * pow(cost_growth, L−1)`。
- **整值吸附**：`nearest = round(raw)`；若 `abs(raw − nearest) ≤ 1e-9`，用 `int(nearest)`；否则 `ceil(raw)`。避免 `16 × 1.5 = 24.000000000000004 → 25`。
- `cost = 0` 合法，价格恒为 0；设计期记 warning，运行期不拒绝。
- `cost_growth < 1.0`、`NaN`、`INF`、`cost < 0`、`max_level < 1` 由 ADR-0001 在加载期拒绝；若运行期仍遇到，返回 `DATA_NOT_READY`，不静默兜底。
- `upgrade_tree_cost(u) = Σ_{L=1..max_level} price(u, L)`；只用于设计期与升级页“全树花费”，不参与购买判定。
- `next_price(id)`：`level+1 > max_level` → `{price: null, reason: MAX_LEVEL}`；未知 id → `{price: null, reason: UNKNOWN_UPGRADE}`；否则返回该级价格。
- `max_level` 是硬上限；达到后 `purchase` 返回 `MAX_LEVEL`，不提供溢出等级或转换。

### 5. 效果解析与快照

```gdscript
func resolve_upgrade_snapshot() -> Dictionary
# {
#   capacity_bonus: int,
#   cloying_resistance: float,
#   menu_vision: int,
#   levels: Dictionary,   # 冻结副本
# }
```

- 对每条 `UpgradeData`：`lvl = clamp(int(_levels.get(id, 0)), 0, max_level)`。
- `STOMACH_CAPACITY`：`capacity_bonus += int(effect_value) × lvl`（ADR-0001 保证正整数）。
- `CLOYING_RESISTANCE`：`cloying_resistance += effect_value × lvl`（保留小数）。
- `MENU_VISION`：`menu_vision += int(effect_value) × lvl`（ADR-0001 保证正整数）。
- 同类型多条记录效果**相加、顺序无关**；未知 id 贡献 0 但保留在 `levels` 副本中。
- 每次调用返回**新字典 + 深拷贝的 `levels`**；下游修改返回值不影响 `_levels`，也不影响下一次快照。
- 解析不读时钟、不用随机数、不做 I/O；同一等级表与同一记录集逐字段相同。
- `effect_value ≤ 0` 或该类型要求整数却给小数：返回 `DATA_NOT_READY`（不按 0 处理、不四舍五入）。
- 下游只能消费已解析值；不得拿 `levels` 自行重算效果。

### 6. `MENU_VISION` 的无效等级护栏

```gdscript
func menu_vision_useful_max(u: UpgradeData, wave_size: int) -> int
# ceil((wave_size − 1) / effect_value)，wave_size ≤ 1 时为 0
```

- `effect_value` 必须是正整数；否则 `DATA_NOT_READY`。
- 整数运算实现 `ceil`：`(n + d − 1) / d`，`n = wave_size − 1`、`d = effect_value`。
- `can_purchase` / `purchase` 在 `effect_type == MENU_VISION` 时：
  - `station_id` 为空或不存在 → `DATA_NOT_READY`；
  - `current_level + 1 > useful_max` → `NO_EFFECT`。
- 本服务只负责“不卖废等级”；真正的预览值仍由菜单波次按 `min(menu_vision, wave_size−1, |W_next|)` 截断。
- 解析值本身不按档口截断；快照反映真实等级，跨档口由菜单波次执行上限。

### 7. 购买事务：校验 → 内存应用 → 提交 → 成功或回滚

`purchase(id, station_id)` 的校验顺序**固定**，第一个失败即返回：

1. `DATA_NOT_READY`：数据未 `Ready`、记录缺失/非法、`MENU_VISION` 缺档口上下文。
2. `NOT_WRITABLE`：`_writable == false`（`Blocked` / 只读 / 载入未完成）。
3. `NOT_IN_MENU`：`is_menu_active.call() != true`。
4. `UNKNOWN_UPGRADE`：id 在数据目录中不存在。
5. `BUSY`：`_purchasing == true`。
6. `MAX_LEVEL`：`current_level >= max_level`。
7. `PREREQUISITE_LOCKED`：`prerequisite_id` 非空且其等级 `< 1`。
8. `NO_EFFECT`：`MENU_VISION` 已达 `useful_max`。
9. `INSUFFICIENT_FUNDS`：`wallet < price`。

通过后：

1. 快照旧值 `{wallet, levels}`。
2. 内存应用：`wallet -= price`、`levels[id] += 1`；`_purchasing = true`。
3. `save.commit({&"progression": _progression_full_value()})`。
4. `commit.ok` → `_purchasing = false`；发 `progression_changed` 与 `purchase_succeeded`；返回 `PURCHASED{price, wallet_after, level_after}`。
5. `commit` 失败 → 回滚 `{wallet, levels}` 到步骤 1；`_purchasing = false`；发 `purchase_failed(COMMIT_FAILED)`；返回 `COMMIT_FAILED`，**不**发 `progression_changed`。

- 成交只在 `commit` 返回成功后确认；界面不得在等待期间显示已购买。
- 购买不可退款；不提供折扣、限时优惠、等级重置。
- `purchase` 不改 `last_applied_run_id`，也不影响结算入账。

### 8. 通用扣款 `spend`

```gdscript
enum SpendReason { STATION_UNLOCK }

func spend(amount: int, reason: SpendReason, extra: Dictionary = {}) -> SpendResult
```

- 校验顺序：`INVALID_AMOUNT`（`typeof != TYPE_INT` 或 `amount < 0`）→ `NOT_WRITABLE` → `NOT_IN_MENU` → `BUSY` → `RESERVED_NAMESPACE`（`extra` 含 `progression`）→ `INSUFFICIENT_FUNDS`。
- `extra` 是**其他命名空间的完整新值**（例如 `{unlocks: {stations: [...]}}`），不是增量；本服务把它原样并入同一次 `commit`，不解释、不修改。
- 通过后：内存 `wallet -= amount` → `save.commit({progression: …, ...extra})` → 成功则发 `progression_changed` 并返回 `SPENT`；失败则回滚钱包并返回 `COMMIT_FAILED`。
- `amount = 0` 合法但不产生任何效果；调用方不应为免费解锁调用 `spend`。
- `reason` 目前只有 `STATION_UNLOCK`；新增原因必须先改本 ADR 与 GDD。

### 9. 结算入账 `prepare_run_result`

```gdscript
enum RunApplyStatus { PREPARED, ALREADY_APPLIED, WALLET_OVERFLOW, DATA_NOT_READY }

func prepare_run_result(result: RunResult) -> ProgressionResult
```

- **去重**：`result.run_id == _last_applied_run_id` → `ALREADY_APPLIED`，钱包不变、不发信号、不构造 `progression_update`。
- **受检加法**：`wallet' = checked_add(wallet, net_profit)`；溢出 → `WALLET_OVERFLOW`，钱包保持原值、`progression_update = null`、不发 `progression_changed`。
- 成功：**立即把 `wallet'` 写入内存**（这是 ADR-0007 的“内存即成功态”），`_last_applied_run_id = run_id`，返回 `PREPARED{progression_update: {wallet: wallet', upgrade_levels: …}, wallet_after}`，并发 `progression_changed` 与 `run_result_applied(run_id, PREPARED, wallet_after)`。
- **不调用 `save.commit`**：合并提交归 ADR-0007 的事务对象；本服务只提供完整新值。
- 写盘失败时：内存值保留，`SaveSystem.mark_dirty(&"progression")` 由事务对象执行；本服务不发“已保存”类信号。收据的 `APPLIED_UNSAVED` 是事务级状态（ADR-0007），不是本方法的返回值。
- `upgrade_levels` 原样进入 `progression_update`（含未知 id、未钳制值）；钳制只发生在快照解析时。

### 10. 目录与可购买性

`get_catalog(station_id)` 返回每条升级的视图：

| 字段 | 含义 |
|---|---|
| `id` / `display_name_key` / `description_key` | 来自 `UpgradeData`；不解析翻译 |
| `level` / `max_level` | 当前等级（读取时钳制）/ 硬上限 |
| `current_effect` / `next_effect` | 已解析效果；满级时 `next_effect = null` |
| `next_price` | 下一级价格；满级或未知 `null` |
| `purchasable` / `reason` | 与 `can_purchase` 同一套判定；只读，不改状态 |

- `can_purchase(id, station_id)` 只校验、不改状态、不 `commit`；顺序与 `purchase` 完全一致。
- 未知 id 不出现在目录中；存档里的未知 id 不出现在目录中，但保留在 `levels`。
- 价格与效果都来自记录，GDScript 中不得出现平衡数字字面量。
- 目录按 `UpgradeData` 的稳定加载顺序或 id 排序；本 ADR 不规定 UI 排序，只要求同一版本稳定。

### 11. 状态与事务图

```
MetaUpgrades（RefCounted，钱包唯一写者）
   ├─ initialize / notify_save_status / reset_after_career_delete
   ├─ get_wallet / get_level / next_price / can_purchase / get_catalog
   ├─ purchase(id, station)  ── 校验 → 内存 → commit → 成功/回滚
   ├─ spend(amount, reason, extra) ── 内存 → commit(progression + extra) → 成功/回滚
   ├─ prepare_run_result(result) ── 去重 → checked add → 内存 → progression_update
   └─ resolve_upgrade_snapshot() ── 新字典 + levels 深拷贝
              │
              ├─▶ SaveSystem（progression 命名空间，原子 commit）
              ├─▶ SettlementTransaction（ADR-0007：合并 progression + records）
              ├─▶ Unlocks（ADR-0017：spend + {unlocks} 同一 commit）
              └─▶ RunConfig.upgrade_snapshot（ADR-0013 / 0019）
```

## Alternatives Considered

### Alternative 1: 提交钱包增量

- **Description**: `commit({progression_delta: {wallet: +19}})`。
- **Pros**: 载荷小。
- **Cons**: 存档要求完整新值；并发/重放/回滚无法判定基线；重复投递会把同一笔钱加两次。
- **Rejection Reason**: 一律提交完整 `wallet` 与完整 `upgrade_levels`。

### Alternative 2: 结算写盘失败也回滚内存

- **Description**: 与购买一致，失败即回滚。
- **Pros**: 内存与磁盘永远一致。
- **Cons**: 玩家已打完一局却看不到收益；收据无法显示“未保存”；与存档的 dirty/重试策略冲突。
- **Rejection Reason**: 结算入账保留内存并标 dirty；购买才回滚。

### Alternative 3: 逐级 `ceil` 价格

- **Description**: 每级价格基于上一级取整后的值再乘增长率。
- **Pros**: 价格表看起来更“整齐”。
- **Cons**: 取整误差逐级累积；同一记录在不同实现下得到不同价格；GDD 明确只取整一次。
- **Rejection Reason**: 用原始 `cost × growth^(L−1)`，最后取整一次。

### Alternative 4: 把已解析效果写进存档

- **Description**: 存 `capacity_bonus` / `cloying_resistance` / `menu_vision`。
- **Pros**: 载入更快。
- **Cons**: 等级与效果两份事实；改 `effect_value` 后旧档不更新；无法支持“历史纪录冻结旧快照”。
- **Rejection Reason**: 只存等级，效果每次解析。

### Alternative 5: 允许结算事务调用 `purchase`

- **Description**: 收据页直接买升级，合并成一笔。
- **Pros**: 少一次写盘。
- **Cons**: 结算事务不得嵌套 `commit`；购买与结算的失败策略相反；`run_settled` 的未交付窗口会被打破。
- **Rejection Reason**: 购买与结算串行、互不合并（ADR-0007 / ADR-0002）。

### Alternative 6: 用百分比抗性

- **Description**: `extra × (1 − resistance)`。
- **Pros**: 曲线平滑。
- **Cons**: 与腻度 D1 决策冲突；小整数上不可心算；引入浮点判定。
- **Rejection Reason**: 定点减免，`floor` 后生效。

### Alternative 7: 提供 `set_wallet` 供调试/补偿

- **Description**: 一个直接写钱包的后门。
- **Pros**: 调试方便。
- **Cons**: 打开第四种流水；任何系统都能绕过事务；与“钱包唯一写者”冲突。
- **Rejection Reason**: 不提供；调试走测试夹具或独立工具。

## Consequences

### Positive

- 钱包只有一个写者、三种流水，任何入账/扣款都可审计。
- 价格与快照是纯函数，可脱离存档与场景做全量单测。
- 购买回滚与入账保留的策略差异被写死，不会在实现时被“统一”掉。
- `levels` 深拷贝 + 未知 id 保留，让删除记录与版本变化都不会破坏旧档。
- `NO_EFFECT` 护栏防止未来档口继续卖无效的 `MENU_VISION` 等级。

### Negative

- 购买是同步阻塞的 `commit`；磁盘慢时 UI 会等待（GDD 接受，慢的是磁盘）。
- `prepare_run_result` 在提交前就改内存，必须依赖 ADR-0007 的事务顺序保证不出现“已加钱但纪录失败”。
- 价格曲线依赖 float64；边界值需要整值吸附与实测。
- `purchase` / `can_purchase` 多一个 `station_id` 上下文参数，调用方必须传当前档口。

## Risks

- **浮点边界抬价**：整值吸附 `1e-9`；用 AC 7–10 的固定价格表做回归。
- **结算入账后纪录中止**：ADR-0007 把 `Records.prepare_run` 排在钱包之前；本服务在 `prepare_run_result` 前不会收到调用。
- **`extra` 覆盖 `progression`**：`RESERVED_NAMESPACE` 拒绝；`extra` 只允许其他命名空间。
- **内存与磁盘分叉**：`APPLIED_UNSAVED` 后 `mark_dirty`；下一次购买/入账/退出 flush 重试；收据显示未保存。
- **未知 id 被清理**：`initialize` 与快照都保留；只有 `delete_career` 清空。
- **UI 在 commit 前显示成交**：`purchase` 同步返回；UI 只按返回值刷新。
- **多档口 `wave_size` 不同**：`NO_EFFECT` 按当前 `station_id` 判定；跨档口显示上限归 ADR-0017 / UX。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `meta-upgrades.md` | `TR-meta-upgrades-001`：钱包 int64，只有结算/购买/解锁三种流水；可负；受检溢出 | 决策 3、7、8、9；三入口 + `_checked_add`，无 setter |
| `meta-upgrades.md` | `TR-meta-upgrades-002`：`upgrade_price` 指数曲线；效果线性；硬 `max_level` | 决策 4、5；单次 `ceil` + 整值吸附，等级钳制 |
| `meta-upgrades.md` | `TR-meta-upgrades-003`：`resolve_upgrade_snapshot()` 返回已解析加成与冻结 `levels` 副本 | 决策 5；新字典 + 深拷贝，未知 id 保留 |
| `meta-upgrades.md` | `TR-meta-upgrades-004`：购买事务 校验 → 内存应用 → `save.commit` → 成功；写失败回滚 | 决策 7；固定九步校验与回滚 |
| `meta-upgrades.md` | `TR-meta-upgrades-006`：`spend(amount, reason, extra)` 通用扣款；`extra` 并入同一 commit | 决策 8；保留 `progression` 命名空间，`extra` 完整新值 |
| `meta-upgrades.md` | `TR-meta-upgrades-007`：`menu_vision_useful_max = ceil((wave_size−1)/effect_value)`；不卖无效等级 | 决策 6；整数 ceil 与 `NO_EFFECT` |
| `meta-upgrades.md` | Core Rule 2 / 3 / 4：三条流水；负净利原样扣款；无上下限只有 int64 边界 | 决策 3；`prepare_run_result` 受检加法 |
| `meta-upgrades.md` | Core Rule 5 / 6：一局只入账一次；编排者一次提交；写失败保留内存 | 决策 9 与 ADR-0007；`last_applied_run_id` + `PREPARED` |
| `meta-upgrades.md` | Core Rule 7：购买先落盘再确认；固定校验顺序 | 决策 7；九步顺序与回滚 |
| `meta-upgrades.md` | Core Rule 8 / 9 / 10：只买等级；价格指数、效果线性；硬上限 | 决策 4、5；无平衡字面量 |
| `meta-upgrades.md` | Core Rule 11 / 12：前置是门槛；快照是已解析值 | 决策 5、7；未知 id 贡献 0 |
| `meta-upgrades.md` | Core Rule 13：快照开局冻结、局内不购买 | 决策 7 的 `NOT_IN_MENU`；快照每次新副本 |
| `meta-upgrades.md` | Core Rule 14 / 15：`MENU_VISION` 只揭示下一波；不卖无效等级 | 决策 6；上限由菜单波次执行 |
| `meta-upgrades.md` | Core Rule 16 / 17：抗性定点减免；三项效果不交叉 | 决策 5；只解析值，不改其他效果 |
| `meta-upgrades.md` | Core Rule 19 / 20 / 21：不可写只读；数据未就绪不可用；删除生涯回默认 | 决策 2、7；`NOT_WRITABLE` / `DATA_NOT_READY` / `reset_after_career_delete` |
| `meta-upgrades.md` | Formulas：`wallet_after_run` / `upgrade_price` / `upgrade_tree_cost` / `resolve_upgrade_snapshot` / `menu_vision_useful_max` | 决策 3–6；逐式落地 |
| `meta-upgrades.md` | AC 1–31（钱包、价格、购买、快照、生命周期） | 决策 3–10；测试矩阵见 Validation |

## Performance Implications

- **CPU**：`resolve_upgrade_snapshot` / `get_catalog` 为 O(U)（U ≤ 12）；单次 < 1 ms。`purchase` 的主要成本是同步磁盘提交，逻辑 < 0.1 ms。
- **Memory**：`wallet` 一个 int64；`levels` 最多 12 项；快照每次 < 1 KiB。
- **Load Time**：`initialize` 只读一个命名空间；不参与启动门。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 定义 `UpgradeData` 读取路径、错误词表、`ProgressionResult` / `PurchaseResult` / `SpendResult` 形状。
2. 实现 `upgrade_price` / `upgrade_tree_cost` / `menu_vision_useful_max` 与纯函数测试（AC 7–10、26）。
3. 实现 `initialize` / `notify_save_status` / `reset_after_career_delete` 与钱包读取（AC 1、28）。
4. 实现 `purchase` 的九步校验、内存应用、`commit`、回滚（AC 11–19）。
5. 实现 `spend` 与 `extra` 合并（AC 供 ADR-0017 使用）。
6. 实现 `prepare_run_result` 的去重、受检加法与 `progression_update`（AC 2–6）。
7. 实现 `resolve_upgrade_snapshot` 与深拷贝（AC 20–25、27、29）。
8. 接入 `SettlementTransaction`（ADR-0007）与升级页（ADR-0019）。

## Validation Criteria

- GDD AC 1–31 全部通过；重点：AC 7–10 价格表、AC 11/12 购买与回滚、AC 13/14 余额边界、AC 20–25 快照与抗性、AC 26 `NO_EFFECT`、AC 29 确定性。
- 静态检查：`src/` 中不出现升级价格、效果、增长率、钱包上下限的字面量；`MetaUpgrades` 之外没有写 `progression` 命名空间的调用。
- 回滚：注入 `commit` 失败，钱包与等级逐字段回到购买前，`purchase_failed` 1 次、`progression_changed` 0 次。
- 入账：注入 `commit` 失败，内存钱包为新值，`APPLIED_UNSAVED` 由事务层报告，`progression_changed` 1 次。
- 溢出：注入 `INT64_MAX` 钱包与正净利，返回 `WALLET_OVERFLOW`，`progression_update == null`，`commit` 载荷不含 `progression`。
- 深拷贝：修改快照的 `levels` 后再次解析，内部 `_levels` 与下一次快照不变。
- 浮点：对固定 `(cost, growth, L)` 表重复 1,000 次，价格逐项相同且等于 GDD 示例。

## GDD Sync Check

- **`purchase(id)` → `purchase(id, station_id)`**：GDD 接口表只写 `id`；本 ADR 增加当前档口上下文以满足 `MENU_VISION` 的 `NO_EFFECT` 判定。MVP 单档口，属签名细化，不需回改 GDD；多档口时由 ADR-0017 / UX 处理。
- **`APPLIED_UNSAVED` 的归属**：GDD 写 `prepare_run_result` 在写盘失败时返回 `APPLIED_UNSAVED`；ADR-0007 已把写盘失败定为事务级状态。本 ADR 将 `prepare_run_result` 的状态固定为 `PREPARED`，由事务层报告 `APPLIED_UNSAVED`。行为一致，建议下次修订 GDD 时把该状态明确为事务级。
- **`effect_value` 取整**：GDD 写“小数部分在解析时取整”，本 ADR 对 `STOMACH_CAPACITY` / `MENU_VISION` 要求整数并由数据配置保证；`CLOYING_RESISTANCE` 保留 float、由腻度 `floor`。与 GDD 一致。

## Related

- ADR-0001 数据配置 — `UpgradeData` / `StationData.wave_size` / `Ready`
- ADR-0002 存档格式 — `progression` 命名空间与原子 `commit`
- ADR-0004 服务组合 — `AppRoot` 构造与 `Callable` 可写性探针
- ADR-0007 结算事务 — `prepare_run_result` 的调用与合并提交
- ADR-0010 菜品数值 — 校准与 `capacity_ceiling` 的设计输入
- ADR-0015 结算纯函数 — `net_profit` / `run_id`
- ADR-0017 解锁与档口 — `spend` + `{unlocks}` 同一 commit
- ADR-0019 HUD 数据绑定 — 升级页与钱包显示
- `design/gdd/meta-upgrades.md`
- `design/gdd/save-system.md`
