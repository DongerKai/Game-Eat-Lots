# ADR-0018: 纪录、里程碑判定与生涯完成

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

本 ADR 把 `records-and-milestones.md` 固定成一套**只消费结算事实、严格单点写入、存档安全编码**的实现契约：`Records` 是 `RefCounted`，只接收 `RunResult`，不重算净利、不读钱包、不写文件；每局按「断言 → 档口纪录 → 生涯合计 → 里程碑 → 完成锁存 → 发信号」的固定顺序**先在副本上算完再整体采纳**；档口最佳按 `net_profit` 严格更大才替换，平局保留旧纪录与旧时间戳；三个信号 `record_broken` → `milestone_earned`（id 升序）→ `career_completed` 各发一次，且在内存应用后立即发出，不等写盘。为满足 ADR-0002「文件中没有浮点」的值域，`records` 不持久化 `return_ratio`（由 `return_percent_x10` 派生展示），`upgrade_snapshot` 中的 `cloying_resistance` 以规范十进制字符串落盘、在只读快照中再解码为 float。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Feature |
| **Layer** | Feature |
| **Knowledge Risk** | LOW — 纯 `Dictionary` / `Array` / int64 运算与排序；时钟只经注入的 `Callable` 采样一次；无随机、无文件 I/O、无物理 |
| **References Consulted** | `design/gdd/records-and-milestones.md`、`design/gdd/settlement-and-entry-fee.md`、`design/gdd/save-system.md`、`design/gdd/data-config.md`、`design/gdd/meta-upgrades.md`、`docs/architecture/architecture.md`、`docs/architecture/adr-0001-content-data-format.md`、`docs/architecture/adr-0002-persistence-format.md`、`docs/architecture/adr-0004-service-composition-signals.md`、`docs/architecture/adr-0007-run-state-settlement-transaction.md`、`docs/architecture/adr-0015-settlement-pure-functions.md`、`docs/architecture/adr-0016-meta-upgrades-wallet.md`、`docs/engine-reference/godot/VERSION.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | ① `RefCounted` 在 4.7.2 上可以声明并同步 `emit_signal`；② `Dictionary.duplicate(true)` 对嵌套字典/数组是深拷贝；③ `StringName` 与 `String` 在字典键与 `==` 比较中的行为一致；④ 注入的 `utc_now` 在无头测试中可返回固定字符串；⑤ `String.rstrip("0")` 与 `"%06d"` 在 4.7.2 上产生稳定的规范十进制串；⑥ int64 边界常量与受检加法不回绕 |

> ⚠️ **引擎知识缺口**：本机没有 Godot 编辑器，上述六条均未实测。未实测前，任何 story 不得声称「已在引擎上验证」。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001（里程碑目录 `MilestoneCatalog` 与四种指标校验、启动门）；ADR-0002（`records` 命名空间的十进制 int64、无浮点值域、原子提交）；ADR-0004（`Records` 由 `AppRoot` 构造，`Save` / `DataConfig` / `MilestoneCatalog` 先于它构造）；ADR-0007（`SettlementTransaction` 的「纪录 → 钱包 → 合并 → 一次提交」顺序与失败矩阵）；ADR-0015（`RunResult` 字段与 `run_id` int64）；ADR-0016（`upgrade_snapshot` 的字段语义） |
| **Enables** | 界面流程的纪录页、里程碑列表、完成横幅与收据「新纪录」印章；音效反馈的三个 cue；`SettlementTransaction` 的 `records` 键；`TR-records-003/005/006` 的验收 |
| **Blocks** | `TR-records-003/005/006`；纪录更新、里程碑判定、完成锁存、三信号的全部实现 story |
| **Ordering Note** | 本 ADR 不定义 `RunResult`（ADR-0015）、不定义 `records` 的文件格式与原子写（ADR-0002）、不定义 `MilestoneData` 的加载与校验（ADR-0001）、不定义结算事务（ADR-0007）、不定义 UI 布局（ADR-0019）。它只定义 `records` 命名空间的所有者侧内容、单局更新管线、比较/累加/判定/锁存规则、信号的发出顺序与只读快照。 |

## Context

### Problem Statement

GDD 已经给出规则，但实现层必须回答五个问题：

1. **一条永久纪录不能被半成品写进去。** 断言、档口比较、生涯累加、里程碑判定与完成锁存必须在一份副本上全部成功后才整体采纳；任何一步失败都不能留下部分更新。
2. **时间戳只能做展示，但完成状态不能因此回退。** 「不读时钟」与「记录达成时间」看似矛盾；实现必须把时钟收束到一个注入的采样点，并规定损坏时间戳的恢复方式。
3. **存档值域里没有浮点。** GDD 的 `return_ratio` 与 `upgrade_snapshot.cloying_resistance` 是 float，而 ADR-0002 只允许 `null` / `bool` / `String` / int64 / `Array` / `Dictionary`。必须有确定的编码，而不是让写入层拒绝整个事务。
4. **信号既不能重复，也不能被写盘失败吞掉。** 一局已经结算，收据必须显示新纪录；写盘失败只影响「未保存」标记，不影响「这一局算数」。重复投递、重载与再来一局都不得重放信号。
5. **里程碑是单调的，完成是历史事实。** 阈值调高、定义删除、通关后新增必需项，都不能撤销已达成状态；但「目录里没有任何必需项」也不能按空集合恒真自动通关。

### Current State

`src/` 只有占位文件。`architecture.md` 已把 `Records` 列为 Feature 层服务；ADR-0001 已把 `MilestoneCatalog` 的加载与校验放在 `Data Ready` 与 `Save.load()` 之间；ADR-0007 已把 `Records.prepare_run` 定为结算事务的第一道致命闸门。缺的是：`records` 命名空间的字段级编码、载入语义校验、单局更新的固定顺序、严格比较、里程碑判定、完成锁存、信号时序与只读快照。

### Constraints

- **只消费事实**：净利、总值、回本率、结束原因与升级快照全部取自 `RunResult`；不读菜品价格、不重放腻度、不重新求和。
- **单点写入**：只有 `Records` 修改 `records` 命名空间；写盘只经 `SettlementTransaction` 的一次 `commit`。
- **整体采纳**：先在副本上算完，全部成功后才替换内部值；失败时内部值逐字段不变。
- **严格更优**：`net_profit` 严格大于旧纪录才替换；平局保留旧纪录，不读时间、不比总值。
- **单调**：已达成里程碑永不撤销；`completed_at` 锁存后不因目录变化回退。
- **确定性**：除注入时钟产生的时间戳外，同一输入给同一输出；不依赖帧率、随机数或字典遍历顺序。
- **无浮点落盘**：`change.records` 必须通过 ADR-0002 的值域检查。

### Requirements

- 必须提供 `initialize` / `load` / `prepare_run` / `snapshot` / `milestone_progress` / `get_milestone_catalog` / `reset_after_career_delete` 与三个信号。
- 必须在载入时做语义校验，保留未知字段与未知 id，只丢弃无法可靠修复的档口纪录。
- 必须按固定的六步顺序应用一局，并在失败时保持内部值不变。
- 必须实现四种里程碑指标、档口过滤、id 升序判定、不追溯授予与完成锁存。
- 必须让时间戳只来自一次注入采样，任何逻辑都不读时间。
- 必须让 `change.records` 不含 float / `StringName` / 引擎类型。

## Decision

### 1. 模块形态与依赖

`Records` 是 `RefCounted`，位于 Feature 层：

- 构造参数：`save: SaveSystem`、`data: DataConfig`、`catalog: MilestoneCatalog`、`utc_now: Callable`（由 `AppRoot` 绑定到「返回 UTC ISO-8601 字符串」的函数）。
- 不持有 `UIRoot`、`RunFlow`、`SettlementTransaction`、`MetaUpgrades`、`Unlocks` 或 `AppRoot`。
- 唯一可变状态：`_records: Dictionary`、`_last_applied_run_id: int`、`_loaded: bool`、`_catalogue_checked: bool`、`_catalogue_ok: bool`。
- 不注册 `_process`、不读墙钟（时钟只经 `utc_now`）、不直接写文件、不调用 `save.commit` / `save.mark_dirty`。
- 信号：`record_broken(station_id, new_best, previous_best)`、`milestone_earned(milestone_id)`、`career_completed()`。

```gdscript
class_name Records extends RefCounted

signal record_broken(station_id: StringName, new_best: Dictionary, previous_best)
signal milestone_earned(milestone_id: StringName)
signal career_completed()

func _init(save: SaveSystem, data: DataConfig, catalog: MilestoneCatalog,
           utc_now: Callable) -> void

func initialize() -> RecordsLoadReport          # 读 save.get_namespace("records") 后调用 load
func load(saved: Variant) -> RecordsLoadReport  # 纯解析；供 BootSequence / 迁移 / 测试
func prepare_run(result: RunResult) -> RecordsApplyResult
func snapshot() -> Variant                      # 未载入返回 null
func milestone_progress(id: StringName) -> Variant
func get_milestone_catalog() -> MilestoneCatalog
func reset_after_career_delete() -> void
```

**不提供**：`add_record`、`set_records`、`grant_milestone`、`complete_career`、`commit`。纪录只有 `prepare_run` 一条写入路径；里程碑只能由判定自动授予。

> **与 ADR-0004 的构造签名差异**：ADR-0004 的节选写作 `Records.new(_save, _data, _milestones)`。本 ADR 追加必需的 `utc_now` 注入，以便静态检查「`Records` 不引用 `Time`」并让测试可固定时间。组合根需改为 `Records.new(_save, _data, _milestones, _utc_now)`。

### 2. `records` 命名空间 schema 与存档编码

`_records` 的顶层形状（所有键为 `String`，值域满足 ADR-0002）：

```gdscript
{
    "stations": {
        "<station_id>": {
            "best": {
                "net_profit": int,
                "total_value": int,
                "entry_fee": int,
                "return_percent_x10": int | null,
                "end_reason": "TIME_EXHAUSTED" | "NO_EATABLE_DISH",
                "upgrade_snapshot": Dictionary,
                "achieved_at": String | null
            },
            # 未知字段原样保留
        },
        # 未知档口 id 原样保留
    },
    "career": {
        "runs_settled": int,
        "lifetime_net_profit": int,
        "lifetime_value": int
    },
    "milestones": {
        "<milestone_id>": {
            "earned_at": String | null,
            "threshold_at_earn": int
        },
        # 未知里程碑 id 原样保留
    },
    "completed_at": String | null,
    # 未知顶层字段原样保留
}
```

**两条编码决策**：

1. **不持久化 `return_ratio`。** GDD 的 `StationRecord.best` 同时列了 `return_ratio`（float64）与 `return_percent_x10`（int）。ADR-0002 的值域没有 float，且两者是同一个数的两种精度。本 ADR 只持久化 `return_percent_x10`，在只读快照中派生 `return_ratio = return_percent_x10 / 1000.0`（`entry_fee = 0` 时为 `null`）。派生值是 0.1% 精度的展示值，不参与任何逻辑；GDD 的展示格式本来就只有一位小数。
2. **`upgrade_snapshot` 以存档安全形式冻结。** ADR-0016 的快照含 `cloying_resistance: float`。写入 `records` 前，`Records` 递归编码：`int` / `bool` / `String` / `null` / `Array` / `Dictionary` 原样（字典键转 `String`），`StringName` 转 `String`，`float` 转**规范十进制字符串**（最多 6 位小数、去尾零、`-0` 写 `0`）。只读快照只把已知键 `cloying_resistance` 解码回 float；未知键中的浮点保持字符串。这样落盘无浮点、展示仍拿到 float，且不需要扩展 ADR-0002 的值域。

**规范十进制**：`round(value × 1_000_000)` 后格式化为 `整.小数`，去掉尾零与孤立小数点；绝对值超过 `9e12` 视为不可编码。该函数只用于展示型冻结值，不参与任何判定。

**保存前断言**：`change.records` 必须通过递归 `_is_save_safe()`（只允许 `null` / `bool` / `int` / `String` / `Array` / `Dictionary`，键为 `String`）。任何 float / `StringName` / `Object` / `Vector*` 出现即返回 `INCONSISTENT_RESULT`，不把非法值交给存档层。

### 3. 载入与语义校验

`initialize()` 在 `Save.load()` 之后由 `BootSequence` 调用一次：

1. `var saved := save.get_namespace(&"records")`。
2. `return load(saved)`。

`load(saved)` 不读时钟、不写盘、不依赖 `DataConfig`；它只解析并规范化命名空间：

- `saved == null`：空纪录，无警告（新生涯或旧档迁移由存档补默认值）。
- `saved` 不是 `Dictionary`：空纪录 + `RECORDS_MALFORMED`。
- `stations` / `milestones` / `career` 不是 `Dictionary`：对应子树置空/置零 + `RECORDS_MALFORMED`。
- 单条档口纪录：
  - 不是 `Dictionary`、`best` 不是 `Dictionary`、或 `best` 缺 `net_profit` / `total_value` / `entry_fee`（或类型不是 int）→ **整条丢弃** + `STATION_RECORD_DROPPED`。
  - `end_reason` 缺失或不在两个枚举值内 → 保留纪录，写 `TIME_EXHAUSTED` + `RECORD_END_REASON_DEFAULTED`（展示字段，不因它丢一条真实净利）。
  - `return_percent_x10` 缺失/类型错误/与 `total_value`、`entry_fee` 不一致 → 按 `floor(1000 × total / fee)` 重算（`fee = 0` 时强制 `null`）+ `RECORD_RETURN_PERCENT_DERIVED`。
  - `upgrade_snapshot` 缺失或不是 `Dictionary` → `{}` + `RECORD_SNAPSHOT_MISSING`；其中含 float → 编码为规范十进制串 + `RECORD_SNAPSHOT_NORMALIZED`。
  - `achieved_at` 缺失/`null` → `null` + `RECORD_TIMESTAMP_MISSING`；非 `String` → `null` + `RECORD_TIMESTAMP_INVALID`。
  - 未知字段原样保留。
- 单条里程碑条目：
  - 不是 `Dictionary` → 保留 id，写 `{earned_at: null, threshold_at_earn: 0}` + 两条缺失警告。
  - `earned_at` 缺失/`null` → `null` + `MILESTONE_TIMESTAMP_MISSING`；非 `String` → `null` + `MILESTONE_TIMESTAMP_INVALID`。
  - `threshold_at_earn` 缺失/非 int → `0` + `MILESTONE_THRESHOLD_MISSING`。
  - 未知字段原样保留。
  - **已达成条目永不删除**；未知 id 原样保留，只是不进入当前完成条件。
- `career`：
  - 缺失 → 三个 0。
  - `runs_settled` / `lifetime_value` 为负 → 钳为 0 + `CLAMPED_NEGATIVE_COUNTER`；非 int → 0 + `CAREER_FIELD_RESET`。
  - `lifetime_net_profit` 可为负；非 int → 0 + `CAREER_FIELD_RESET`。
  - 未知字段保留。
- `completed_at`：
  - `null` → `null`。
  - 非空 `String` → 原样保留（含 `"unknown"` 哨兵）。
  - 空串或非 `String` → `"unknown"` + `COMPLETED_AT_INVALID`。**保留「已通关」这个事实**，不伪造日期，也不因一个展示字段把生涯改回未通关。
- `_last_applied_run_id = 0`、`_loaded = true`；不发任何信号。

`RecordsLoadReport`：`{warnings: Array[Dictionary]}`，每条与 ADR-0001 的 `Diagnostic` 同形：`{code: StringName, detail: Dictionary}`。`BootSequence` 把它并入 `LoadReport.warnings`。

### 4. 单局更新的固定顺序与一致性断言

`prepare_run(result)` 是**唯一**的纪录写入点。它不写盘，但会更新内存并同步发信号；这是「prepare」在本 ADR 中的唯一含义——**不提交，不是无副作用**。

固定顺序，任一步失败立即返回且 `_records` 逐字段不变：

1. **幂等**：`_last_applied_run_id != 0 && result.run_id == _last_applied_run_id` → `ALREADY_APPLIED`（`applied = false`、`change = null`、无信号）。
2. **数据就绪**：`!_loaded` 或 `data.get_state() != READY` → `DATA_NOT_READY`。
3. **目录可用**：`_catalogue_ok == false` → `MILESTONE_CATALOGUE_INVALID`。
4. **一致性断言**（全部通过才继续）：
   - `run_id` 是 int 且 `> 0`；
   - `station_id` 是非空 `StringName`；
   - `entry_fee` 是 int 且 `0 ≤ entry_fee ≤ INT32_MAX`；
   - `total_value` 是 int 且 `≥ 0`；
   - `net_profit` 是 int，且 `net_profit == checked_sub(total_value, entry_fee)`；
   - `entry_fee > 0` 时：`return_percent_x10` 是 int、`≥ 0`、等于 `_expected_percent_x10(total_value, entry_fee)`，且 `return_ratio != null`；
   - `entry_fee = 0` 时：`return_percent_x10 == null` 且 `return_ratio == null`；
   - `end_reason` ∈ `{TIME_EXHAUSTED, NO_EATABLE_DISH}`；
   - `upgrade_snapshot` 是 `Dictionary` 且可安全编码。
   任一不成立 → `INCONSISTENT_RESULT`，`error_detail` 记录字段、期望值与实际值。
5. **在副本上计算**：`new_records = _records.duplicate(true)`；依次更新档口纪录（决策 5）、累加生涯（决策 6）、判定里程碑（决策 7）、锁存完成（决策 8）。任一步 int64 溢出 → `OVERFLOW`，副本丢弃。
6. **整体采纳**：`_records = new_records`、`_last_applied_run_id = result.run_id`；构建 `change` 与 `pending_events`，按决策 10 的顺序发信号；返回 `APPLIED`。

> 把第 5 步全部放在副本上，是为了让「断言通过但累加溢出」与「里程碑判定异常」都不会留下半条纪录。`OVERFLOW` 不修改 `_last_applied_run_id`，同一个 `run_id` 的后续调用仍会重新校验。

**`_expected_percent_x10(total, fee)`** 只用于断言，不产出任何字段：

```
q = total / fee
r = total % fee
percent = q × 1000 + (r × 1000) / fee
```

`r < fee ≤ INT32_MAX`，`r × 1000` 不会溢出；`q × 1000` 用受检乘法，溢出则断言失败（`INCONSISTENT_RESULT`）。这条实现与 ADR-0015 的 `floor(1000 × total / fee)` 同值，但不复制结算的输出路径。

**`return_ratio` 只做非空断言**：float 的逐位相等不可靠，GDD 也只要求「非空」。回本率的整数展示值由 `return_percent_x10` 负责。

### 5. 档口最佳纪录（`TR-records-003`）

比较式固定为：

```
record_is_better(new_net, old_best) =
    (old_best = ⊥) OR (new_net > old_best.net_profit)
```

- **严格更大才替换**；相等时旧纪录整条保留——包括 `achieved_at`、`upgrade_snapshot` 与所有冻结字段。平局不比较时间、不比较总值、不比较回本率。
- **第一次结算无论净利正负都建立纪录**；`previous_best = null`。净利为负时仍是该档口的「最佳」，只是界面不盖「新纪录」印章（`record_broken` 信号照发，见决策 10）。
- 替换时新 `best` 的已知字段全部来自本局 `RunResult` 与本次时间戳；旧 `best` 的未知字段**不复制**（它们描述的是旧纪录）。档口条目中 `best` 之外的未知字段保留。
- 档口 id 未知于数据目录时仍按 id 建立/替换纪录；展示名由快照解析，解析不到就标为「已归档」。
- 比较只读 `best.net_profit` 的 int 值；载入校验已保证进入内部的纪录具备该字段。

### 6. 生涯累加与溢出

```
runs_settled'        = checked_add(runs_settled, 1)
lifetime_net_profit' = checked_add(lifetime_net_profit, net_profit)
lifetime_value'      = checked_add(lifetime_value, total_value)
```

- 三个值都是 int64；任何一步溢出 → `OVERFLOW`，本次更新整体不生效（档口纪录也不替换），不回绕、不饱和。
- `lifetime_net_profit` 有符号；亏损局与空局都计入。`runs_settled` 对空局也 +1；`lifetime_value` 对空局 +0。
- 累加是事实，不是最佳值；与档口纪录是否刷新无关。
- 正常玩法受局数与单价约束不可能溢出；检查是为了把非法数据变成可见错误，测试用注入夹具覆盖。

### 7. 里程碑判定与进度

判定只读**更新后的生涯合计**与**当前这一局的 `RunResult`**：

```
milestone_met(m, u, c) = station_ok(m, u) AND metric_met(m.metric, m.threshold, u, c)

station_ok(m, u) = (m.station_id == "") OR (m.station_id == u.station_id)

metric_met(NET_PROFIT, τ, u, c)           = u.net_profit >= τ
metric_met(RETURN_RATIO_X10, τ, u, c)     = (u.return_percent_x10 != null) AND (u.return_percent_x10 >= τ)
metric_met(LIFETIME_NET_PROFIT, τ, u, c)  = c.lifetime_net_profit >= τ
metric_met(RUNS_SETTLED, τ, u, c)         = c.runs_settled >= τ
```

- 只遍历**当前目录中尚未达成**的里程碑，按 id 升序；每条只判定一次，达成后写入 `{earned_at: 本次时间戳, threshold_at_earn: m.threshold}`。
- 里程碑用第 N 局更新后的 `runs_settled`，所以「累计 1 局」「累计 10 局」在对应局当次达成。
- **不追溯**：目录新增的里程碑只从下一次结算起可达成，不回扫旧纪录；单局阈值无法从聚合数据忠实重建。
- **单调**：已达成 id 永不删除、永不撤销；阈值后来调高，`threshold_at_earn` 仍是达成时的值；定义被删除，条目保留但不再出现在活动列表。
- 生涯类指标（`LIFETIME_NET_PROFIT` / `RUNS_SETTLED`）带非空 `station_id` 属于非法目录，由 `MilestoneCatalog` 在启动期拒绝（ADR-0001）；本服务不做第二次判定，只把目录不可用映射为 `MILESTONE_CATALOGUE_INVALID`。
- `RETURN_RATIO_X10` 绑定零入场费档口同样由目录加载器拒绝；运行期遇到 `return_percent_x10 == null` 时该条自然不成立。

**`milestone_progress(id)`** 只读、不参与判定：

| 指标 | `current` 来源 | 无纪录时 |
|---|---|---|
| `RUNS_SETTLED` | `career.runs_settled` | `0` |
| `LIFETIME_NET_PROFIT` | `career.lifetime_net_profit` | `0` |
| `NET_PROFIT` 带档口 | 该档口 `best.net_profit` | `null` |
| `NET_PROFIT` 无档口 | 所有档口 `best.net_profit` 的最大值 | `null` |
| `RETURN_RATIO_X10` 带档口 | 该档口 `best.return_percent_x10` | `null` |
| `RETURN_RATIO_X10` 无档口 | 所有非空 `best.return_percent_x10` 的最大值 | `null` |

返回 `{current, threshold, unit, earned}`；`unit` 取 `&"CNY"` / `&"PERCENT_X10"` / `&"RUNS"`；未知 id 返回 `null`。`current = null` 表示「还没打过」，不是 0。

### 8. 完成条件与锁存

```
required(M) = {m ∈ M : m.required_for_completion}
all_required_earned(M, E) = (|required(M)| > 0) AND (∀m ∈ required(M) : m.id ∈ E)

completed_at' =
    completed_at                              if completed_at != null
    now                                       if completed_at == null AND all_required_earned
    null                                      otherwise

career_complete = (completed_at != null)
```

- **存在量词**必须显式判真：目录没有任何必需项时，`all_required_earned` 恒为 false，不能靠「空集合恒真」在第一局自动通关。`NO_COMPLETION_PATH` 警告由 `MilestoneCatalog` 在启动期给出（ADR-0001），本服务不重复。
- **锁存**：`completed_at` 一旦非空，永不改写、永不回退；通关后新增必需里程碑不撤销完成状态，只作为「通关后目标」。
- **锁存前**跟随当前目录：删除一条必需里程碑会放宽条件，下一次结算可能立即通关；这是会改变完成难度的数据编辑，必须与通关条件一起评审（GDD 规则 14）。
- **锁存只在本服务成功应用一局时发生**；载入不重新判定、不发 `career_completed`。载入时 `completed_at != null` 的生涯直接派生 `career_complete = true`。
- 完成条件不发放解锁（GDD 规则 15；解锁 GDD 已确认 MVP/VS 不发放）。

### 9. 时间戳（`TR-records-005`）

- `achieved_at` / `earned_at` / `completed_at` 由注入的 `utc_now` 在**一次 `prepare_run` 中采样一次**，三者共用同一个字符串；格式为 `YYYY-MM-DDTHH:MM:SSZ`（UTC）。
- `utc_now` 是唯一的时间来源；`Records` 源码**不引用 `Time` / `OS` / `Engine.get_frames_drawn`**。`AppRoot` 绑定默认实现，测试注入固定值。
- **任何逻辑都不读时间戳**：比较、排序、判定、冲突、迁移、过期都不读；快照按 id 排序，不按时间。时钟回拨不会重排、撤销或重发任何纪录。
- `load`、`snapshot`、`milestone_progress`、`reset_after_career_delete` **不调用** `utc_now`。
- 时间戳缺失时写 `null` 并警告，不丢纪录；`completed_at` 损坏时写 `"unknown"` 哨兵，保留完成事实。
- 时间戳是唯一允许在两次相同输入的运行之间变化的字段；确定性测试必须排除它。

### 10. 信号与事件顺序（`TR-records-006`）

```gdscript
signal record_broken(station_id: StringName, new_best: Dictionary, previous_best)
signal milestone_earned(milestone_id: StringName)
signal career_completed()
```

**只在 `prepare_run` 返回 `APPLIED` 时发**，同步、按固定顺序：

1. `record_broken`：本局替换/建立了档口纪录时发一次；首次建立时 `previous_best = null`。净利为负也发，界面决定是否盖印章。
2. `milestone_earned`：本次新达成的每条里程碑各发一次，**按 id 升序**。
3. `career_completed`：`completed_at` 从 `null` 变为非空的这一次发一次；此后永不重发。

- `ALREADY_APPLIED`、四类失败、载入、删除生涯、快照读取都不发信号。
- 同一帧多次到达的 `milestone_earned` 由音效反馈合并播放，但 `Records` 仍逐条发（GDD 规则 11 / AC 20）。
- `new_best` / `previous_best` 是**只读视图**（与 `snapshot().station_records[].best` 同形），深拷贝；`previous_best = null` 表示首次。
- `pending_events` 与信号同序、同内容，供 `SettlementTransaction` 记录与测试断言；它**不是延迟队列**，信号已经在 `prepare_run` 返回前发出。
- 信号在 `Save.commit` 之前发出（ADR-0007 D5）：结算事务永不回滚内存，写盘失败只标 dirty；把信号延后到写盘成功会让「未保存」的收据没有印章。

### 11. 只读快照

`snapshot()` 在 `!_loaded` 时返回 `null`；否则返回深拷贝：

```gdscript
{
    "station_records": Array[Dictionary],   # 按 station_id 升序
    "career": Dictionary,                   # 三个合计 + 未知字段
    "milestones": Array[Dictionary],        # 按 id 升序，活动 + 已归档
    "completed_at": String | null,
    "career_complete": bool
}
```

**档口行**：

```gdscript
{
    "station_id": StringName,
    "display_name": String | null,          # 数据目录查不到 → null
    "archived": bool,                       # display_name == null
    "best": {
        "net_profit": int,
        "total_value": int,
        "entry_fee": int,
        "return_percent_x10": int | null,
        "return_ratio": float | null,       # 派生：percent_x10 / 1000.0；fee = 0 时 null
        "end_reason": StringName,
        "upgrade_snapshot": Dictionary,     # 已知键 cloying_resistance 解码为 float
        "achieved_at": String | null
    }
}
```

**里程碑行**：

```gdscript
{
    "id": StringName,
    "display_name": String | null,
    "description": String | null,
    "metric": StringName | null,
    "threshold": int | null,
    "required_for_completion": bool,
    "earned": bool,
    "earned_at": String | null,
    "threshold_at_earn": int | null,
    "progress": { "current": int | null, "unit": StringName } | null,
    "archived": bool                        # 不在当前目录的已达成条目
}
```

- 活动里程碑来自当前目录，按 id 升序；已达成但已删除定义的条目追加为 `archived = true`，仍显示达成日期与冻结阈值。
- 未达成条目必须出现，不隐藏；`progress.current = null` 显示「未开始」。
- 档口被数据目录移除时，纪录仍按 id 显示，`archived = true`，冻结值不变。
- 快照是视图，不是第二份状态：调用方修改返回值不影响 `_records`。

### 12. 生命周期与只读语义

- **初始化**：`initialize()` 读命名空间 → `load()` → `_validate_catalogue()`；目录不可用只影响 `prepare_run`（返回 `MILESTONE_CATALOGUE_INVALID`），不影响已载入纪录的读取。
- **只读存档**：本服务**不感知** `SaveStatus.writable`，也不返回 `NOT_WRITABLE`。结算来源的写盘失败由 ADR-0007 保留内存并标 dirty，纪录不回滚；购买来源的回滚与本服务无关。
- **删除生涯**：`reset_after_career_delete()` 置 `_records = _empty_records()`、`_last_applied_run_id = 0`、`_loaded = true`；不发信号、不保留旧纪录、不读旧档。由 `AppRoot` 在 `save.delete_career()` 成功后调用。
- **数据未就绪**：`prepare_run` → `DATA_NOT_READY`；`snapshot` 仍可返回已载入纪录（展示名解析不到时为 null）。
- **目录校验**：`_validate_catalogue()` 只做 `MilestoneCatalog` 构造后仍应成立的形状与语义护栏（id 唯一且匹配 `^ms_[a-z0-9_]+$`、指标合法、`RUNS_SETTLED` / `RETURN_RATIO_X10` 阈值 ≥ 1、生涯指标无档口过滤、回本率指标不绑零入场费档口）。**权威校验在 ADR-0001 的加载器**；这里是防御，不另立一套规则。

### 13. 事务与查询图

```
Settlement (ADR-0015) ── RunResult
        │
        ▼
SettlementTransaction (ADR-0007)
   ├─ 1. idempotency
   ├─ 2. Records.prepare_run(result)
   │        ├─ 断言 → 副本上算完 → 采纳 → 发三信号
   │        └─ change.records（存档安全）
   ├─ 3. MetaUpgrades.prepare_run_result(result)
   ├─ 4. 解锁槽位（MVP/VS 不存在）
   ├─ 5. merge {records, progression}
   └─ 6. SaveSystem.commit 一次
              │
              ▼
        Records（内存值不回滚；失败标 dirty）
              │
              ├─▶ UIRoot（纪录页、里程碑、完成横幅、收据印章）
              └─▶ AudioDirector（record_broken / milestone_earned / career_completed）
```

`Records` 不调用 `Save.commit` / `mark_dirty`；它只返回 `change.records`，由事务合并。

## Alternatives Considered

### Alternative 1: 持久化 `return_ratio`（float）

- **Description**: 按 GDD 字段表把 `return_ratio` 原样写进 `records`。
- **Pros**: 与 GDD 字面一致；展示不需要派生。
- **Cons**: 违反 ADR-0002「文件中没有 JSON number / 没有 float」；需要扩展值域或引入第二套编码；两个精度可能不一致。
- **Rejection Reason**: 只存 `return_percent_x10`，在快照中派生 0.1% 精度的展示值。

### Alternative 2: 把 `upgrade_snapshot` 原样落盘

- **Description**: 冻结字典里保留 `cloying_resistance` 的 float。
- **Pros**: 字段类型不变。
- **Cons**: 写入层必须拒绝整个事务或临时转字符串；重载后类型不稳定。
- **Rejection Reason**: 在 `records` 边界做规范编码，只读快照再解码已知键。

### Alternative 3: 用全局最佳代替每档口最佳

- **Description**: 只存一条生涯最高净利。
- **Pros**: 纪录页更短。
- **Cons**: 不同档口入场费不同，绝对净利不可比；失去「吃遍所有档口」的长期目标。
- **Rejection Reason**: 每个档口一条纪录，按 `station_id` 分键。

### Alternative 4: 按回本率排名

- **Description**: 用 `return_percent_x10` 决定最佳局。
- **Pros**: 跨档口看起来可比。
- **Cons**: 玩家在收据上看到的是净利，钱包来源也是净利；单一档口内两者单调一致，净利更直接。
- **Rejection Reason**: 按 `net_profit` 排名，回本率只冻结展示。

### Alternative 5: 平局用时间或总值二次比较

- **Description**: 净利相等时取更晚/总值更大的那局。
- **Pros**: 总能刷新出一条「更新」的纪录。
- **Cons**: 比较读时钟会依赖玩家机器时间回拨；总值与「这局赚了多少」无关，历史不再稳定。
- **Rejection Reason**: 平局保留旧纪录，只比较净利。

### Alternative 6: 在 `Records` 里重算结算

- **Description**: 不信任 `RunResult`，自己从行明细求和、重算回本率。
- **Pros**: 少一个上游契约。
- **Cons**: 第二套算术；上游修正后两边可能分叉；违反 GDD 规则 3「输入只读，不重算」。
- **Rejection Reason**: 只做一致性断言，净利/总值/回本率全部取自 `RunResult`。

### Alternative 7: 新增里程碑时回溯授予

- **Description**: 目录新增一条里程碑后，用已有聚合数据补发。
- **Pros**: 玩家不会错过目标。
- **Cons**: 单局阈值无法从聚合数据忠实重建；加载时补时间戳等于伪造达成日期。
- **Rejection Reason**: 不追溯，只从下一次结算起可达成。

### Alternative 8: 写盘成功后才发信号

- **Description**: 把三个信号搬到 `Save.commit` 成功之后。
- **Pros**: 信号与磁盘严格一致。
- **Cons**: 写盘失败时一局已经结束，收据却没有「新纪录」印章；玩家会以为这一局没算数。
- **Rejection Reason**: 内存应用后立即发；写盘失败只影响「未保存」标记（ADR-0007 D5）。

### Alternative 9: 写盘失败时回滚纪录

- **Description**: 与购买一样，`commit` 失败就把纪录恢复到上一局。
- **Pros**: 内存与磁盘一致。
- **Cons**: 抹掉一局已经发生的结算事实；同一份 `RunResult` 在钱包与纪录之间出现两种命运；违反 GDD 规则 16。
- **Rejection Reason**: 保留内存、标 dirty、下一个写入时点重试。

## Consequences

### Positive

- 一局的所有纪录变化在副本上算完再采纳，失败不会留下半条纪录。
- 严格净利比较 + 不读时间，让纪录在不同机器、不同时钟下完全稳定。
- 三信号顺序固定、各发一次，收据、音效与测试有唯一契约。
- `records` 不含浮点，ADR-0002 的解码器不需要 schema 特判。
- 里程碑单调、完成锁存，阈值调整与目录增删不会撤销玩家已经拿到的东西。
- 未知档口/里程碑/字段原样保留，旧档降级与数据回滚都不会丢历史。

### Negative

- `prepare_run` 名字像纯准备，实际会改内存并同步发信号；调用方必须理解这一点，且不能在致命校验之前调用它。
- `upgrade_snapshot` 在存档里是编码形式、在快照里是解码形式，存在两种表示；未知键中的浮点只保留字符串。
- `return_ratio` 不再有独立的全精度冻结值，只有 0.1% 精度的派生值。
- `completed_at = "unknown"` 是唯一的哨兵值，展示层需要识别。
- 载入时对损坏 `end_reason` 做默认值，严格来说是对一个展示字段的修正。
- `MILESTONE_CATALOGUE_INVALID` 的护栏与 ADR-0001 的加载器有重叠，需要在实现时保持同一组谓词。

### Risks

- **信号顺序回归**：用 AC 20 的固定顺序断言；任何新增信号必须排在 `career_completed` 之前并更新 ADR。
- **结算与纪录的回本率分叉**：断言只比整数 `return_percent_x10`；float `return_ratio` 只查非空。
- **`upgrade_snapshot` 编码碰撞**：只解码已知键 `cloying_resistance`；未知键中的十进制字符串不反解。
- **`run_id` 跨会话重用**：本服务的幂等只在会话内；跨会话由 ADR-0007 的 `run_id + digest` 负责。
- **异常中止后纪录已采纳**：`SettlementTransaction` 必须在调用 `prepare_run` 前确认 `MetaUpgrades` 不处于 `BUSY`；否则这是上游 bug，纪录已生效而事务快照为 `ABORTED`，需要进入故障态而不是继续。
- **损坏时间戳**：`completed_at = "unknown"` 保留完成事实；UI 必须显示「未知」而不是崩溃或回退为未通关。
- **未知字段膨胀**：`records` 会原样保留旧字段；存档体积上限由 AC 33 的估算覆盖，新增字段时同步更新。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `records-and-milestones.md` | `TR-records-003`：按档口、按 `net_profit` 排名；严格更优才替换；平局保留旧纪录 | 决策 5；`record_is_better` 单一实现，无时间/总值二次比较 |
| `records-and-milestones.md` | `TR-records-005`：UTC 时间戳只做展示；任何逻辑不读时钟 | 决策 9；注入 `utc_now` 单次采样；载入/快照/判定不调用；排序按 id |
| `records-and-milestones.md` | `TR-records-006`：`record_broken` / `milestone_earned` / `career_completed` 三信号 | 决策 10；固定顺序、各发一次、`APPLIED` 才发、payload 深拷贝 |
| `records-and-milestones.md` | Core Rule 2 / 3 / 11 / 22：一局一次、输入只读、固定管线、显式状态 | 决策 4、12；副本计算 + 整体采纳 + 六状态 |
| `records-and-milestones.md` | Core Rule 5 / 6 / 7 / 8：净利排名、严格更优、冻结历史、负纪录不设下限 | 决策 5；旧纪录整条保留，首次负纪录照建 |
| `records-and-milestones.md` | Core Rule 9 / 19 / 20：生涯如实累加、不存逐局历史、单机信任 | 决策 6；受检 int64；schema 无 history/recent 字段 |
| `records-and-milestones.md` | Core Rule 10 / 12 / 13 / 14：稳定 id、四种指标、不追溯、完成锁存 | 决策 7、8；id 升序、四指标、只判未达成、存在量词 |
| `records-and-milestones.md` | Core Rule 15：时间戳只展示、UTC、共用一次采样 | 决策 9；单次 `utc_now`，三字段共用 |
| `records-and-milestones.md` | Core Rule 16 / 17 / 18 / 21：只经命名空间、载入语义校验、只读对外、确定性 | 决策 2、3、11；`change.records` 单一写入口、schema 校验、深拷贝快照 |
| `records-and-milestones.md` | AC 1–37（纪录、生涯、里程碑、完成、幂等、失败、载入、冻结、编排） | 见 Validation Criteria 的逐条映射 |

## GDD Sync Check

| # | 冲突 / 差异 | 本 ADR 的裁决 | 需要回改的对象 |
|---|---|---|---|
| 1 | GDD `StationRecord.best` 持久化 `return_ratio: float64`，ADR-0002 禁止文件内出现 float | 只存 `return_percent_x10`；快照派生 `return_ratio = percent_x10 / 1000.0`，精度 0.1% | `records-and-milestones.md`：把 `return_ratio` 标为派生展示字段，或从持久化 schema 移除 |
| 2 | GDD 冻结 `upgrade_snapshot`（含 `cloying_resistance: float`），ADR-0002 禁止 float | 落盘时把 float 编码为规范十进制字符串；只读快照把已知键解码回 float | `records-and-milestones.md`：注明冻结快照的存档安全编码；`adr-0016` 的快照语义不变 |
| 3 | GDD `RecordsApplyResult` 列 `status/applied/change/error_detail`；ADR-0007 D4 期望 `pending_events` | 增加 `pending_events`（同步事件日志，不是延迟队列），其余字段不变 | `records-and-milestones.md`：在返回形状表中补 `pending_events`（行为不变，仅契约显式化） |
| 4 | GDD `completed_at` 只允许 `String \| null`，未定义损坏时间戳 | 损坏时写 `"unknown"` 哨兵，保留「已通关」事实；不伪造日期、不回退 | `records-and-milestones.md`：在 schema 或 Edge Cases 中登记 `"unknown"` |
| 5 | GDD 只列了四条载入警告码 | 新增 `CAREER_FIELD_RESET`、`RECORD_END_REASON_DEFAULTED`、`RECORD_RETURN_PERCENT_DERIVED`、`RECORD_SNAPSHOT_MISSING`、`RECORD_SNAPSHOT_NORMALIZED`、`RECORD_TIMESTAMP_INVALID`、`MILESTONE_TIMESTAMP_INVALID`、`COMPLETED_AT_INVALID` 等封闭码 | `records-and-milestones.md`：补警告词表或接受 ADR 的词表 |
| 6 | GDD `end_reason` 是语义字段，但只影响展示 | 缺失/非法时保留纪录并默认 `TIME_EXHAUSTED`，写 `RECORD_END_REASON_DEFAULTED` | 无（与「不因展示字段丢真实纪录」一致；建议在 GDD Edge Cases 补一句） |
| 7 | GDD 说 `record_broken` 首次建立纪录时发一次，又说负纪录不盖印章 | 信号照发；印章由界面按 `new_best.net_profit > 0` 决定 | 无（GDD 已分叉描述，此处统一口径） |
| 8 | ADR-0007 D4 第 7 步写「采纳与发信号」在 `commit` 之后，D5 写信号在 `prepare_run` 返回 `APPLIED` 时发出 | 本 ADR 按 D5：`prepare_run` 采纳并同步发信号；D4 第 7 步对 Records 退化为构建快照 | ADR-0007：统一 D4/D5 的措辞；`records` 的采纳点是 `prepare_run` |
| 9 | ADR-0004 的构造节选是 `Records.new(_save, _data, _milestones)` | 追加必需的 `utc_now` 注入，便于固定时间与静态检查 | ADR-0004：更新构造节选为四参 |
| 10 | GDD `load(saved)` 由谁调用未写明 | `initialize()` 内部读 `save.get_namespace("records")` 并调用 `load(saved)`；`load` 仍公开供迁移/测试 | 无（实现细节；GDD 的 `load` 契约保留） |
| 11 | GDD `MILESTONE_CATALOGUE_INVALID` 要求 `prepare_run` / `load` 返回 | `load` 只解析纪录、不因目录失败；目录不可用由 `prepare_run` 返回 `MILESTONE_CATALOGUE_INVALID`，正常启动门会先失败 | `records-and-milestones.md` AC 27：把「载入」改为「应用」，或注明载入阶段由目录加载器先失败 |

## Performance Implications

- **CPU**：`prepare_run` 为 O(S + M log M + M)，S = 档口纪录数（≤ 4）、M = 里程碑数（≤ 24）；单次 < 1 ms。`snapshot` 为 O(S log S + M log M)，只在界面打开时调用。
- **Memory**：`records` 约 4 条档口纪录 + 24 条里程碑 + 三个 int64，通常 < 8 KiB；每次 `prepare_run` 深拷贝一份，峰值 < 32 KiB。
- **Load Time**：`load` 只解析一个命名空间，O(S + M)；不参与启动门，位于 `Save.load()` 之后。
- **Network**: N/A。

## Migration Plan

无既有实现。建议实现顺序：

1. 定义 `RecordsApplyResult` / `RecordsChange` / `RecordsLoadReport` / 状态与警告词表，实现 `_empty_records` 与 `load` 的结构/语义校验（AC 24、29）。
2. 实现 `_record_is_better`、生涯受检累加、一致性断言与 `OVERFLOW`（AC 2、3、13、14、23、25）。
3. 实现里程碑目录护栏、四指标判定、`milestone_progress` 与不追溯（AC 15–19、27）。
4. 实现完成条件、锁存与 `career_completed`（AC 6、19、21）。
5. 实现信号顺序、`pending_events` 与幂等（AC 20、22、35–37）。
6. 实现快照、`return_ratio` 派生、`upgrade_snapshot` 编解码（AC 5、11、30）。
7. 接入 `SettlementTransaction` / `BootSequence`，用 commit spy 验证一次事务（AC 28、36）。
8. 用 headless 基准跑 1,000 次应用（AC 34）。

## Validation Criteria

- GDD AC 1–31 全部通过；时间戳在断言中只检查非空与格式，其余字段逐项比较。
- AC 32–34（目录校验、`save_bytes` 估算、性能）作为 CI / 设计期检查。
- AC 35–37（编排契约）用 commit spy：失败时 0 次 commit、`APPLIED` 时 1 次且载荷含 `records` / `progression`、`ALREADY_APPLIED` 时 0 次。
- **静态断言**：`Records` 源码不出现 `FileAccess`、`Time`、`OS`、`RandomNumberGenerator`、`commit(`、`mark_dirty(`；`change.records` 不含 float / `StringName`。
- **确定性**：固定 `utc_now` 时，同一 `_records` + 同一 `RunResult` 两次应用逐字段深相等；改变 `utc_now` 只改变时间戳字段。
- **不变量**：任意输入下 `runs_settled` 单调 +1；`lifetime_value ≥ 0`；已达成里程碑集合只增不减；`completed_at` 只从 `null` 变为非空。
- **属性测试**：10,000 组随机 `RunResult` + 随机起始 `records`，断言不崩溃、不产生负计数、不撤销已达成里程碑、不重复发信号。
- **编码往返**：`upgrade_snapshot` 编码 → 写入 → 解码后，已知键 `cloying_resistance` 在 6 位小数内相等；其余键逐字段相等。
- **无法直接断言**：单机信任模型（本地纪录可被手工编辑）、完成横幅的叙事效果、阈值校准（由 `/prototype` 与 `/playtest-report` 负责）。

## Related

- ADR-0001 内容数据格式 — `MilestoneCatalog`、`MilestoneData`、四种指标校验与启动门
- ADR-0002 持久化格式 — `records` 命名空间、十进制 int64、无浮点值域、原子提交
- ADR-0004 服务组合与信号 — `Records` 的构造与信号连接
- ADR-0007 结算事务 — `prepare_run` 的调用时机、`records` 键、失败矩阵与 dirty 重试
- ADR-0015 结算纯函数 — `RunResult` 字段与一致性断言的上游事实
- ADR-0016 局外升级 — `upgrade_snapshot` 的字段语义与冻结副本
- `design/gdd/records-and-milestones.md`
- `design/gdd/settlement-and-entry-fee.md`
- `design/gdd/save-system.md`
- `design/gdd/data-config.md`
