# ADR-0001: 内容数据格式、校验与启动门

## Status

Accepted

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。
> 其他 agent 与任何 skill 都不能自行设置。** 引用 `Proposed` ADR 的 story 会被置为
> `Blocked`，epic 要求 `Accepted`，所以这个字段必须有明确的负责人。

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

- 用户（`modes.automation: autonomous`，由用户先行授权）
- `godot-specialist`（引擎 API 与导出行为校验）
- `technical-director` 战略审阅：跳过（`review_mode: lean`）

## Summary

数据配置是《回本》唯一的数值来源，也是启动时唯一必须先通过的门：所有菜品、升级、档口、平衡、里程碑与音频 cue 数值都必须来自可校验的数据记录，任何一条不合法都不能进入游戏。本 ADR 决定用带类型的 Godot `Resource`（`.tres`）加显式清单资源作为唯一内容格式，由 `DataConfig` 在启动时全有或全无地加载与校验，里程碑目录由「纪录与里程碑」自带的只读加载器并入同一道启动门，音频 cue 目录作为第六个清单由 `DataConfig` 自己加载（ADR-0009）。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；本机 `Installed at pin time = NOT DETERMINED`，尚未安装编辑器） |
| **Domain** | Core（`Resource` 加载、`@export` 序列化、启动期数据校验） |
| **Layer** | Foundation |
| **Knowledge Risk** | MEDIUM — `docs/engine-reference/godot/VERSION.md` 将 Core / persistence 与 Resources 标为 MEDIUM；4.5 新增 `@abstract` 与 `duplicate_deep()`，4.7 改变类型化返回与 packed array setter 行为 |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`breaking-changes.md`、`current-best-practices.md`、`deprecated-apis.md`、`modules/ui.md` |
| **Post-Cutoff APIs Used** | `@abstract`（4.5）用于记录基类。`duplicate_deep()`（4.5）不使用；`FileAccess`（4.4 起 `store_*` 返回 `bool`）不在本 ADR 路径上 |
| **Verification Required** | 1) 只读属性语义：getter-only 属性赋值是否被拒绝（静态赋值与 `set()` 两条路径）；2) `Array[DishData]` / `Array[StringName]` 的 Inspector 编辑与 `.tres` 往返；3) 导出预设开启「Convert text resources to binary」后，固定路径清单与记录仍可加载；4) `ThemeDB.fallback_icon` 在 4.7 编辑器、导出包与 headless 下存在且非空；5) 300 条记录的 Validating p95 ≤ 100 ms（GDD AC 22） |

> ⚠️ **引擎知识缺口**：4.5–4.7 均在 LLM 训练截止之后，且本机没有安装编辑器。
> 上述 Verification Required 项在第一次写代码前必须在 pin 定的 4.7.x 上实测；
> 在实测完成前，本 ADR 中涉及运行时语义的结论按「待验证」处理。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | None — 本 ADR 是 `architecture.md` Required ADRs 的 Foundation 顺序第一项 |
| **Enables** | ADR-0002（存档：持久化 id 契约）、ADR-0004（服务组合：注入 `DataConfig`）、ADR-0007（结算事务：数据 `Ready` 前提）；以及全部 gameplay epic |
| **Blocks** | 所有 gameplay epic 与 story（待 `/create-epics` 命名）——数据未 `Ready` 不得构造任何 gameplay 模块 |
| **Ordering Note** | 其他 Foundation ADR 可并行撰写，但任何读取内容数据的系统都必须以本 ADR 的查询 API 与启动门为准。ADR 编号按 `architecture.md` Required ADRs 的排列给出，若编号后续调整，以本文件名 `ADR-0001` 为稳定引用 |

## Context

### Problem Statement

《回本》的支柱 1 是「每口都是算术题」：玩家能算，前提是系统自己先算得清。这要求
全部平衡数值只有一个来源、可被找到、可被修改、可被验证，并且不合法的数据必须在
任何 gameplay 模块被构造之前就被拦下。现在 18 份 GDD 已把数值字段、取值范围、
交叉引用和校验规则写清楚，但没有任何代码，也没有决定这些记录以什么格式存在、
如何枚举、如何校验、失败后游戏怎么走。这个决定必须先于所有 gameplay 代码。

### Current State

- 全部 18 个已设计系统的 GDD 为 Approved；`data-config.md` 明确了四类记录
  （`DishData` / `UpgradeData` / `StationData` / `BalanceData`）及字段与校验规则。
- ADR-0009 追加 `AudioCueCatalogResource` 作为第六个清单，schema 与
  `AUDIO_CUE_*` 规则归其 D7，加载与发布归本 ADR 的同一道启动门。
- `records-and-milestones.md` 的 OQ2 已定：`MilestoneData` 不进入数据配置 schema，
  由纪录系统自带只读加载器提供，并与数据配置的 `Ready` 门并列。
- `architecture.md` v1 的启动顺序是 `DataConfig.load()` → `ready` → `Save.load()`
  → `Boot → Menu`；数据失败则 `Boot → Error`，不构造 gameplay 模块。
- 仓库中没有 `src/` 实现、没有 `data/` 目录、没有 ADR；`docs/registry/architecture.yaml`
  为空。本 ADR 是第一条落地的架构约束。

### Constraints

- 引擎固定 Godot 4.7 / GDScript；已安装编辑器版本未确定，运行时语义需实测。
- 游戏数值必须数据驱动：GDScript 中不得出现平衡数值字面量（0、1 等结构性常量除外）。
- 启动校验必须成为所有下游系统的硬门；不允许部分加载、静默降级或默认值兜底。
- 内容规模：MVP 12 道菜、少量升级/档口、1 条平衡记录；完整愿景约 40 道菜。
- 记录运行时只读；存档只存 id 与等级/数值，载入后回查数据配置，不信任快照。
- 单人开发、trunk-based；内容文件需要可读 diff、可并行编辑、无生成物污染。

### Requirements

- 必须支持四类记录加里程碑目录，字段与取值范围与 GDD 一致。
- 必须把音频 cue 目录作为第六个清单并入同一道启动门（ADR-0009 D8）。
- 必须支持稳定 id、类型前缀、发布后不可改名。
- 必须在启动时完成结构、字段、引用、双向一致性与环检测，并输出每一条违反项的
  文件、记录 id、字段名、期望值与实际值。
- 必须保证任何错误都导致整体 `Failed`，且 `Failed` 后不发布任何记录。
- 必须让「纪录与里程碑」的只读目录加载器并入同一道启动门。
- 必须支持缺失图标时的占位显示，且不阻断启动。
- 必须能在导出的 PCK 中可靠枚举全部记录，不依赖运行时目录扫描。
- 300 条记录的 Validating p95 必须 ≤ 100 ms（GDD AC 22）。

## Decision

### 1. 记录格式：纯类型化 `Resource`，不做运行时 JSON/CSV 解析

六类记录都是 `Resource` 子类；除音频 cue 外，一条记录一个 `.tres`：

| 记录 | 基类 | 归属 | 文件位置 |
|---|---|---|---|
| `DishData` | `ContentRecord` | Foundation / 数据配置 | `res://data/dish/*.tres` |
| `UpgradeData` | `ContentRecord` | Foundation / 数据配置 | `res://data/upgrade/*.tres` |
| `StationData` | `ContentRecord` | Foundation / 数据配置 | `res://data/station/*.tres` |
| `BalanceData` | `Resource` | Foundation / 数据配置 | `res://data/balance.tres` |
| `MilestoneData` | `ContentRecord` | Feature / 纪录与里程碑 | `res://data/milestone/*.tres` |
| `AudioCueData` | `Resource` | Presentation / 音效反馈 | `res://data/audio/cue_catalog.tres` 的 `cues` 内嵌子资源 |

`AudioCueData` 是唯一不以独立 `.tres` 落盘的记录：它作为
`AudioCueCatalogResource.cues` 的子资源内嵌在单一目录文件中（ADR-0009 D7），
使音频目录一次加载、一次校验、一次发布，并与 `CueMap` 的双向校验共用同一份数据。

`ContentRecord` 是 4.5 `@abstract` 基类，只携带 `id: StringName` 与只读访问器；
`BalanceData` 是单例，不需要 id。

选择理由（12 道菜 → 约 40 道菜）：

- 记录总量在完整愿景下仍不足百条，类型化资源不需要手写解析器、schema 校验器或
  导入管线；Godot 的 `@export` 与 Inspector 直接提供字段类型和可视化编辑。
- 引用（`station_id`、`dish_pool`、`prerequisite_id`）与 `Texture2D` 图标是资源
  引用，`.tres` 原生表达；JSON/CSV 需要字符串路径解析与额外解析代码。
- 枚举、整数范围、`Array[StringName]` 都能被类型系统与校验器共同约束。
- 40 条记录的 `.tres` diff 仍然可读；规模远未到「必须用表格批量编辑」的阈值。
- 发布时由 Godot 导出器把文本资源转换为二进制；源文件仍是 `.tres`，无需自定义
  打包格式。这同时回答 `data-config.md` OQ4。

**逃生口**：若记录数达到数百条，或出现非技术作者需要表格批量编辑，可在
`tools/content/` 增加一个 **只写** 的 CSV→`.tres` 导入工具。运行时永远只有类型化
`Resource` 一种格式；该工具不得成为第二套 schema。

### 2. 枚举：显式清单资源，禁止运行时目录扫描

每个类型有一个清单资源（manifest），记录必须登记在清单里才能进入游戏：

| 清单资源 | 路径 | 内容 |
|---|---|---|
| `DishCatalogResource` | `res://data/catalog/dish_catalog.tres` | `records: Array[DishData]`、`placeholder_icon: Texture2D` |
| `UpgradeCatalogResource` | `res://data/catalog/upgrade_catalog.tres` | `records: Array[UpgradeData]` |
| `StationCatalogResource` | `res://data/catalog/station_catalog.tres` | `records: Array[StationData]` |
| `BalanceCatalogResource` | `res://data/catalog/balance_catalog.tres` | `balance: BalanceData` |
| `MilestoneCatalogResource` | `res://data/milestone/milestone_catalog.tres` | `records: Array[MilestoneData]` |
| `AudioCueCatalogResource` | `res://data/audio/cue_catalog.tres` | `schema_version: int`、`cues: Array[AudioCueData]`、`duck_db: float`、`duck_attack_ms: int`、`duck_release_ms: int`（由 ADR-0009 D7/D8 引入） |

`DataConfig` 只按固定 `res://` 路径 `ResourceLoader.load()`，**永不** 用 `DirAccess`
扫描 `res://data/`。理由：

- 导出包会把文本资源转换为二进制，并带有 `.uid` 侧车文件；对 `*.tres` 的目录
  扫描在导出包里可能漏项或重复计数（`godot-specialist` 复核确认此风险真实）。
- 清单给出确定的加载顺序，重复 id 的报告顺序稳定，测试可复现。
- 一次 `load()` 读一个清单，而不是逐个文件探测，加载更快。
- 清单本身就是「哪些记录属于这个 catalog」的显式契约，不会把备份文件、
  实验文件或未完成资源意外带进游戏。

代价是新增记录要登记一次。用 **编辑器/CI 专用** 的完整性测试兜底：扫描
`res://data/dish|upgrade|station/*.tres`，断言每个文件都被对应清单引用；运行时
永远不做这次扫描。

### 3. id 命名空间与发布后规则

**语法**：

- 通用：`^[a-z][a-z0-9_]*$`（与 `data-config.md` 规则 3 一致）。
- 类型前缀：`dish_`、`upgrade_`、`station_`、`ms_`。
- 里程碑另有 `^ms_[a-z0-9_]+$`（至少一个字符，与 `records-and-milestones.md` 一致）。
- 同一类型内 id 唯一；跨类型由前缀天然隔离，不做全局唯一性要求。
- id 是记录字段，不是文件名。文件名与 id 不一致只产生 `FILE_NAME_MISMATCH`
  警告；id 永远是权威。

**发布后规则**：

- 第一个公开版本发布时，生成 `data/published_ids.txt`（每行一个 id，按类型分组、
  组内升序），并入版本管理。
- 已发布 id **永不改名、永不删除**。新增 id 不受限制；修改数值、`display_name`、
  图标不受限制。
- 改名等于删除旧 id 并新增新 id：存档只存 id（`save-system.md` 规则 2），旧 id
  会变成未知条目被原样保留但永不生效（规则 14），玩家的升级等级与解锁进度会
  静默失效。
- CI/单元测试对比 `published_ids.txt` 与当前清单；任何已发布 id 消失即失败。
- 内容下线（retire）在 MVP 不支持：`data-config.md` 要求每道菜必须属于其档口且
  档口池必须包含它，直接删除会破坏双向一致性。下线方案见 Open Questions。

### 4. `DataConfig` 状态机与全有或全无

```
Unloaded ──load()──► Loading ──全部文件读入──► Validating ──全部通过──► Ready
                        │                          │
                        └──文件缺失/解析失败────────┴──任一 ERROR──► Failed
```

| 状态 | 进入条件 | 查询行为 | 出口 |
|---|---|---|---|
| `Unloaded` | 构造后 | `NOT_READY` | `Loading` |
| `Loading` | `load()` 开始 | `NOT_READY` | `Validating` / `Failed` |
| `Validating` | 全部清单与记录读入 | `NOT_READY` | `Ready` / `Failed` |
| `Ready` | 全部校验通过并发布 | 正常 | `Unloaded`（退出游戏） |
| `Failed` | 任一 ERROR | `NOT_READY` | 终态 |

**不允许部分加载**：

- `Loading` 与 `Validating` 期间只写临时数组与临时字典；任何对象都不对外可见。
- 任一 ERROR → 丢弃临时数据，状态 `Failed`；有效子集永远不会被发布。
- `get_dish` / `get_upgrade` / `get_station` / `get_dish_icon` 在非 `Ready` 时返回
  `Err(NOT_READY)`，绝不返回 `null` 或默认记录。
- `get_balance()` 沿用 `architecture.md` 的直接返回签名，前置条件是 `Ready`；
  在非 `Ready` 调用是编程错误（`push_error` 并返回 `null`），不是可恢复查询失败。
- `load()` 是同步的：40 条记录在启动期远低于预算，不做线程化加载。
- `load()` 在 `Ready` 时再次调用返回当前（成功的）报告；在 `Failed` 时返回缓存的
  失败报告，**不重新读盘、不重试**（GDD AC 18）。恢复路径只有重启游戏。
- `ready` / `failed` 信号各发一次；调用方必须在 `load()` 之前连接，或直接读取
  `load()` 的返回值与 `get_state()`。

### 5. 校验流程与错误词汇

校验分六个阶段（另有 5b 音频 cue 子阶段），按顺序执行；每阶段收集全部独立诊断，不因第一条错误提前退出：

| 阶段 | 检查 | 前置 |
|---|---|---|
| 1. 结构 | 文件存在、可解析、类型正确、清单内无 `null`、`BalanceData` 存在 | — |
| 2. 身份 | id 语法、类型前缀、同类型重复 | 阶段 1 的记录集合完整 |
| 3. 字段 | 整数范围、枚举、标签表、有限浮点、设计带警告 | 阶段 2 无错误 |
| 4. 引用 | `station_id` 存在、`dish_pool` 存在且双向一致、前置存在且无环、至少一个免费档口 | 阶段 2 无重复 id、阶段 1 无 `null` |
| 5. 警告 | 缺图标、文件名不一致、`cost_growth` 偏离设计带 | 阶段 1 完成 |
| 5b. 音频 cue | ADR-0009 D7 的八条 `AUDIO_CUE_*` 规则（schema、id、bus、范围、stream、priority） | 阶段 1 的 cue 目录类型正确 |
| 6. 发布 | 建立只读目录、封存记录、状态 `Ready` | 前五阶段与 5b 无 ERROR |

**阶段跳过规则**：若某阶段的前置不成立，该阶段不执行，并在报告中写入
`CHECKS_SKIPPED`（警告级），`detail` 指明被跳过的阶段与原因。跳过的检查不得
被读成通过。

**诊断条目**（`Diagnostic`）：`code`、`severity`、`file`、`record_id`、`field`、
`expected`、`actual`、`detail`。报告（`DataError`）持有 `diagnostics` 与
`skipped_phases`，`is_valid()` 等价于 `errors().is_empty()`。

| Code | 级别 | 含义 |
|---|---|---|
| `CATALOG_FILE_MISSING` | ERROR | 清单或平衡文件不存在 |
| `CATALOG_PARSE_FAILED` | ERROR | `ResourceLoader.load()` 返回 `null` 或解析失败 |
| `CATALOG_WRONG_TYPE` | ERROR | 加载到的资源不是期望的清单/记录类型 |
| `RECORD_NULL` | ERROR | 清单数组含 `null` |
| `RECORD_WRONG_TYPE` | ERROR | 清单项不是期望的记录类型 |
| `ID_SYNTAX` | ERROR | id 不匹配 `^[a-z][a-z0-9_]*$` |
| `ID_PREFIX` | ERROR | id 缺少本类型前缀 |
| `DUPLICATE_ID` | ERROR | 同一类型内 id 重复 |
| `EMPTY_FIELD` | ERROR | `display_name` / `description` 等必填文本为空 |
| `VALUE_OUT_OF_RANGE` | ERROR | 整数超出 GDD 范围（含 `INT32_MAX` 上限） |
| `VALUE_NOT_INTEGER` | ERROR | 离散字段（`effect_value` 用于 `STOMACH_CAPACITY` / `MENU_VISION`）不是正整数 |
| `VALUE_NOT_FINITE` | ERROR | 浮点字段为 NaN 或 INF |
| `UNKNOWN_ENUM` | ERROR | `effect_type` 等枚举值不在定义内 |
| `UNKNOWN_TAG` | ERROR | `taste_tags` 含固定七标签之外的标签 |
| `VALUE_OUTSIDE_DESIGN_BAND` | WARNING | `cost_growth` 合法但落在 1.2–1.6 设计带之外 |
| `UNKNOWN_REFERENCE` | ERROR | `station_id` / `dish_pool` / `prerequisite_id` 指向不存在的记录 |
| `REFERENCE_CYCLE` | ERROR | 升级前置链成环（含自引用） |
| `BIDIRECTIONAL_MISMATCH` | ERROR | 菜的 `station_id` 与档口 `dish_pool` 不一致 |
| `DUPLICATE_POOL_ENTRY` | ERROR | `dish_pool` 重复引用同一道菜 |
| `NO_UNLOCKED_STATION` | ERROR | 没有任何 `unlock_cost = 0` 的档口（采纳两份 GDD 的建议） |
| `BALANCE_MISSING` | ERROR | `BalanceCatalogResource.balance` 为空 |
| `MISSING_ICON` | WARNING | 菜品缺失 `icon`，使用占位图标 |
| `PLACEHOLDER_MISSING` | WARNING | 清单未配置占位图标，退回引擎默认图标 |
| `FILE_NAME_MISMATCH` | WARNING | `.tres` 文件名与记录 id 不一致 |
| `CHECKS_SKIPPED` | WARNING | 因前置失败跳过某校验阶段 |
| `MILESTONE_ID_SYNTAX` | ERROR | 里程碑 id 不匹配 `^ms_[a-z0-9_]+$` |
| `MILESTONE_DUPLICATE_ID` | ERROR | 里程碑 id 重复 |
| `MILESTONE_EMPTY_FIELD` | ERROR | 里程碑名称或条件文案为空 |
| `MILESTONE_UNKNOWN_METRIC` | ERROR | 指标不在四种定义内 |
| `MILESTONE_THRESHOLD_RANGE` | ERROR | 阈值违反 `RUNS_SETTLED ≥ 1` / `RETURN_RATIO_X10 ≥ 1` |
| `MILESTONE_UNKNOWN_STATION` | ERROR | 档口过滤器指向不存在的档口 |
| `MILESTONE_STATION_FILTER_ON_CAREER` | ERROR | 生涯类指标（终身净利、结算局数）带档口过滤器 |
| `MILESTONE_NO_COMPLETION_PATH` | WARNING | 没有任何 `required_for_completion = true` 的里程碑 |
| `AUDIO_CUE_SCHEMA` | ERROR | `AudioCueCatalogResource.schema_version` 不为 1 |
| `AUDIO_CUE_DUPLICATE_ID` | ERROR | 音频 cue id 重复 |
| `AUDIO_CUE_UNKNOWN_BUS` | ERROR | cue `bus` 不是 `SFX` / `UI` |
| `AUDIO_CUE_BAD_RANGE` | ERROR | cue 数值越界、pitch 非 1.0、`max_voices` 超硬顶或 `PLAY_THROUGH_FREEZE` 授予非 `sfx.flow.pause` |
| `AUDIO_CUE_NO_STREAM` | ERROR | cue 的 `streams` 为空 |
| `AUDIO_CUE_STREAM_MISSING` | ERROR | cue 引用的 `AudioStream` 为 `null` |
| `AUDIO_CUE_ZERO_DURATION` | ERROR | stream 长度为 0 或与 `expected_length_ms` 偏差超过 ±50 ms |
| `AUDIO_CUE_BAD_PRIORITY` | ERROR | cue `priority` 不在 1…4 |

`AUDIO_CUE_UNMAPPED` / `AUDIO_CUE_ORPHAN` 是 `AudioDirector.bind_catalog()` 的错误，不属于 `DataConfig` 报告；两者同样使 `Boot → Error`（ADR-0009 D7/D8）。

`NO_UNLOCKED_STATION` 采用 ERROR 而非运行期兜底：`unlocks-and-stations.md`
规则 4 与 `data-config.md` 规则 7 都建议把它加为启动校验。解锁系统的
`NoPlayableStation` 仍保留为防御路径，但在合法数据下不应再触发。

### 6. 启动门与失败行为

`AppRoot`（组合根，见 ADR-0004）是唯一启动编排者：

```gdscript
# AppRoot — 组合根
func _boot() -> void:
    _data_config = DataConfig.new()
    var report := _data_config.load()
    if not report.is_valid():
        _scene_flow.request_transition(SceneState.ERROR, BootFailure.data_config(report))
        return

    _milestone_catalog = MilestoneCatalog.new(_data_config)
    var ms_report := _milestone_catalog.load()
    if not ms_report.is_valid():
        _scene_flow.request_transition(SceneState.ERROR, BootFailure.milestone_catalog(ms_report))
        return

    _save.load()                       # 存档载入永不阻断 Menu（save-system 规则 11）
    _scene_flow.request_transition(SceneState.MENU)
```

- 数据配置失败 → **不构造** 任何 gameplay 模块、不载入存档、不进入 `Menu`；
  `SceneFlow` 走 `Boot → Error`（`architecture.md` 启动流程）。
- 里程碑目录失败 → 同一条 `Boot → Error` 路径，payload 标明来源。
- 错误屏由场景/界面 ADR 拥有；它必须显示非技术性说明，并把完整诊断写入日志。
  本 ADR 只固定 payload：来源（`DATA_CONFIG` / `MILESTONE_CATALOG`）+ 完整
  `DataError`。
- `DataConfig` 是 `RefCounted`，不是 autoload、不是静态单例。`AppRoot` 必须持有
  并注入它；没有持有者时对象会被释放（`godot-specialist` 复核提示）。
- 下游模块的构造函数必须断言 `data_config.get_state() == READY`；启动门是
  构造顺序约束，不只是信号通知。

### 7. 里程碑目录加载器

- 归属：**纪录与里程碑**。代码位于 `src/feature/records/data/milestone_catalog.gd`。
- 它不进入 `DataConfig` schema（`data-config.md` 规则 2、`records-and-milestones.md`
  OQ2 已定）。
- 构造时注入 `DataConfig`，在 `DataConfig` `Ready` 之后加载；校验 `station_id`
  过滤器时通过 `get_station()` 查询档口是否存在，因此顺序不可颠倒。
- 与数据配置共用 Foundation 的诊断词汇（`Diagnostic` / `DataError`），但由
  Records 拥有校验规则与 `MilestoneData` 类型。
- 成功后 `Records.get_milestone_catalog()` 返回只读目录；`get_all()` 按 id 升序，
  保证里程碑判定顺序与界面列表稳定。
- 失败与 `DataConfig` 失败同权：`Boot → Error`，不构造 gameplay 模块。

### 8. 占位图标策略

- `DishData.icon` 是可选 `Texture2D`，纯展示字段，**永不参与** 任何玩法计算。
- 缺失 `icon` → `MISSING_ICON` 警告，状态仍为 `Ready`（GDD AC 9）。
- 展示层必须通过 `DataConfig.get_dish_icon(id)` 取图标，不得自行实现 fallback：
  1. 记录的 `icon` 非空 → 返回它；
  2. 否则返回清单的 `placeholder_icon`；
  3. 清单占位也为空 → `PLACEHOLDER_MISSING` 警告，返回 `ThemeDB.fallback_icon`。
- 记录本身不被修改（不把占位图标写回 `DishData`），保持只读与同一实例语义。
- 占位图标是美术可以晚于数据到位的保障，不阻断启动。

### 9. 热重载：MVP 不支持

- 内容目录每个进程只加载一次；编辑 `.tres` 后必须重启游戏才生效。
- `DataConfig` 不提供 `reload()` 或文件监听 API。
- 理由：热重载要求原子重校验并替换目录，而下游系统可能持有记录引用与每局冻结
  快照（`architecture.md` 原则 4）；中途替换会破坏确定性与「一局内数值不变」。
  该收益在 40 条记录的规模下不值得一致性风险。GDD OQ3 已倾向放到 Vertical
  Slice，本 ADR 确认 MVP 不做。
- 若未来加入，只允许在 **两次运行之间** 整体替换（新实例加载成功后再切换），
  不得在 run 进行中替换；且必须复用同一套全有或全无校验。

### Architecture Diagram

```
res://data/
  catalog/
    dish_catalog.tres        ─┐
    upgrade_catalog.tres      │  ResourceLoader.load()（固定路径，不扫描）
    station_catalog.tres      │
    balance_catalog.tres     ─┘
  dish/*.tres  upgrade/*.tres  station/*.tres  balance.tres
  milestone/milestone_catalog.tres  milestone/*.tres
  audio/cue_catalog.tres            # AudioCueCatalogResource（ADR-0009）

AppRoot（组合根）
  ├─ DataConfig.load()        Unloaded→Loading→Validating→Ready|Failed
  │     └─ CatalogValidator（纯函数，无 I/O）
  ├─ MilestoneCatalog.load(data_config)
  └─ Save.load() → SceneFlow Boot→Menu
                          └─ 任一失败 → Boot→Error（不构造 gameplay）

下游（菜品数值 / 腻度 / 菜单波次 / 局外升级 / 餐局流程 / 结算 / 解锁 / HUD / UI / 音效反馈）
  └─ DataConfig.get_dish / get_upgrade / get_station / get_balance（只读）
```

### Key Interfaces

> `architecture.md` 的 `Result[T, E]` 是设计级记法；GDScript 4.7 没有用户自定义泛型，
> 因此具体实现用 `DataResult` 承载同一语义，调用方在 `is_ok()` 后做一次静态类型赋值。
> 这是记法的落地，不是接口语义的改变。
>
> 下面每个 `class_name` 各自位于一个 `.gd` 文件；此处合并展示只为阅读，不是单文件结构。

```gdscript
# Foundation — src/foundation/data_config/

class_name DataConfig extends RefCounted

signal ready()
signal failed(error: DataError)

enum State { UNLOADED, LOADING, VALIDATING, READY, FAILED }

func _init(paths: CatalogPaths = null) -> void
func load() -> DataError                              # errors 为空 == 成功
func get_state() -> State
func get_report() -> DataError

func get_dish(id: StringName) -> DataResult            # value: DishData
func get_upgrade(id: StringName) -> DataResult         # value: UpgradeData
func get_station(id: StringName) -> DataResult         # value: StationData
func get_dish_icon(id: StringName) -> DataResult       # value: Texture2D；已知 id 永不为 null
func get_cue(id: StringName) -> DataResult             # value: AudioCueData
func has_cue(id: StringName) -> bool
func cue_catalog() -> AudioCueCatalogResource          # 前置条件：READY
func get_balance() -> BalanceData                      # 前置条件：READY


class_name DataResult extends RefCounted

enum QueryError { OK, NOT_READY, UNKNOWN_ID }

var value: Variant = null
var error: QueryError = QueryError.OK

func is_ok() -> bool


class_name DataError extends RefCounted               # 命名沿用 architecture.md

var diagnostics: Array[Diagnostic] = []
var skipped_phases: Array[StringName] = []

func is_valid() -> bool                               # errors().is_empty()
func errors() -> Array[Diagnostic]
func warnings() -> Array[Diagnostic]


class_name Diagnostic extends RefCounted

enum Severity { ERROR, WARNING }
enum Code {
    CATALOG_FILE_MISSING, CATALOG_PARSE_FAILED, CATALOG_WRONG_TYPE,
    RECORD_NULL, RECORD_WRONG_TYPE,
    ID_SYNTAX, ID_PREFIX, DUPLICATE_ID,
    EMPTY_FIELD, VALUE_OUT_OF_RANGE, VALUE_NOT_INTEGER, VALUE_NOT_FINITE,
    UNKNOWN_ENUM, UNKNOWN_TAG, VALUE_OUTSIDE_DESIGN_BAND,
    UNKNOWN_REFERENCE, REFERENCE_CYCLE, BIDIRECTIONAL_MISMATCH,
    DUPLICATE_POOL_ENTRY, NO_UNLOCKED_STATION, BALANCE_MISSING,
    MISSING_ICON, PLACEHOLDER_MISSING, FILE_NAME_MISMATCH, CHECKS_SKIPPED,
    MILESTONE_ID_SYNTAX, MILESTONE_DUPLICATE_ID, MILESTONE_EMPTY_FIELD,
    MILESTONE_UNKNOWN_METRIC, MILESTONE_THRESHOLD_RANGE,
    MILESTONE_UNKNOWN_STATION, MILESTONE_STATION_FILTER_ON_CAREER,
    MILESTONE_NO_COMPLETION_PATH,
    AUDIO_CUE_SCHEMA, AUDIO_CUE_DUPLICATE_ID, AUDIO_CUE_UNKNOWN_BUS,
    AUDIO_CUE_BAD_RANGE, AUDIO_CUE_NO_STREAM, AUDIO_CUE_STREAM_MISSING,
    AUDIO_CUE_ZERO_DURATION, AUDIO_CUE_BAD_PRIORITY
}

var code: Code
var severity: Severity
var file: String
var record_id: StringName
var field: StringName
var expected: String
var actual: String
var detail: String


# 记录：公开只读属性 + `_` 前缀的 @export 存储字段
@abstract
class_name ContentRecord extends Resource

@export var _id: StringName = &""
var id: StringName: get: return _id


class_name DishData extends ContentRecord

@export var _display_name: String = ""
@export var _price: int = 0
@export var _satiety: int = 1
@export var _taste_tags: Array[StringName] = []
@export var _cloying_group: StringName = &""
@export var _station_id: StringName = &""
@export var _icon: Texture2D = null

var display_name: String: get: return _display_name
var price: int: get: return _price
var satiety: int: get: return _satiety
var cloying_group: StringName: get: return _cloying_group
var station_id: StringName: get: return _station_id
var icon: Texture2D: get: return _icon
var taste_tags: Array[StringName]:
    get:
        var copy: Array[StringName] = []
        copy.assign(_taste_tags)
        return copy                    # 防御性拷贝：外部无法通过数组修改目录
```

`UpgradeData` / `StationData` / `BalanceData` / `MilestoneData` 按同一模式实现，字段
与 `data-config.md` / `records-and-milestones.md` 一一对应。

```gdscript
# Feature — src/feature/records/data/milestone_catalog.gd

class_name MilestoneCatalog extends RefCounted

func _init(data_config: DataConfig) -> void
func load() -> DataError
func get_all() -> Array[MilestoneData]                 # id 升序
func get_milestone(id: StringName) -> DataResult       # value: MilestoneData

# Records 对外只读访问器（records-and-milestones.md）
func get_milestone_catalog() -> MilestoneCatalog
```

### Implementation Guidelines

**必须**

1. `DataConfig` 必须由组合根构造并注入；不得做成 autoload、静态单例或全局可写对象。
2. 所有 gameplay 数值必须经 `get_dish` / `get_upgrade` / `get_station` /
   `get_balance` 读取；GDScript 中只允许结构性常量（`0`、`1`、空数组/空字符串）。
3. 记录的公开字段必须是 getter-only 属性；`_` 前缀的 `@export` 字段是记录类与
   校验器的内部存储，其他任何脚本读或写它们都视为 bug。
4. 数组字段的 getter 必须返回防御性拷贝；不得把内部数组直接交出去。
5. `Ready` 之前不得发布任何记录；查询必须返回 `NOT_READY`，不得返回 `null`、
   空记录或部分目录。
6. `Failed` 必须是进程级终态；`load()` 不得重试，恢复只能靠重启。
7. 每条记录必须登记在对应清单中（`AudioCueData` 登记在 `AudioCueCatalogResource.cues`）；运行时不得扫描 `res://` 目录来发现记录。
8. 校验器必须收集全部独立诊断，并显式报告被跳过的阶段（`CHECKS_SKIPPED`）。
9. 里程碑目录必须在 `DataConfig` `Ready` 之后、`Save.load()` 之前加载；失败走
   同一条 `Boot → Error`。
10. 展示层必须通过 `get_dish_icon()` 取图标；不得对 `DishData.icon` 自行兜底。
11. 导出预设必须包含全部 catalog / record / milestone / audio cue 资源：开启
    「Export all resources in the project」，或把每个清单显式列为依赖。
    仅靠代码里的固定路径 **不会** 让资源进入导出包。
12. 发布后 id 基线测试（`published_ids.txt`）必须进 CI；已发布 id 消失即失败。
13. 音频 cue 目录必须在 `Validating` 的 5b 子阶段跑 ADR-0009 D7 的规则；
    `AudioDirector.bind_catalog()` 的 `CueMap` 双向校验在 `DataConfig Ready`
    之后执行，失败同样走 `Boot → Error`。

**不得**

1. 不得在 GDScript 中写平衡数值、价格、容量、阈值或调参常量。
2. 不得在运行时解析 JSON/CSV，也不得把 `.tres` 作为“生成物的副本”反向导出。
3. 不得修改已发布的 id，或用改名表达“替换一条记录”。
4. 不得在 `Validating` 期间向任何调用方暴露临时目录。
5. 不得在 MVP 加入 `reload()`、文件监听或局内热替换。
6. 不得让里程碑校验绕过 `DataConfig` 的档口存在性检查。
7. 不得在导出包里依赖目录扫描来枚举内容。

## Alternatives Considered

### Alternative 1: JSON 或 CSV 作为运行时格式

- **Description**：记录写成 JSON（或 CSV），启动时解析成字典/对象，再建立 id 索引。
- **Pros**：纯文本、diff 友好；不需要编辑器即可编辑；便于外部工具生成。
- **Cons**：需要手写解析器、schema 校验器与类型转换；枚举、`Array[StringName]`、
  `Texture2D` 引用都要用字符串路径二次解析；GDScript 失去 `@export` 的类型检查；
  CSV 无法自然表达数组与嵌套引用；两套类型系统容易漂移。
- **Estimated Effort**：比选定方案多 1–2 个 story 的解析与校验代码。
- **Rejection Reason**：40 条记录的规模不值得引入第二套 schema；`data-config.md`
  规则 2 明确要求记录在 Godot 编辑器里可视化编辑，JSON/CSV 与之相悖。

### Alternative 2: 混合——CSV/JSON 作者格式 + 导入为 `.tres`

- **Description**：策划在表格里编辑，构建时由工具生成 `.tres`，运行时仍只有
  `Resource`。
- **Pros**：保留运行时类型安全；批量编辑方便；未来可接非技术作者。
- **Cons**：需要导入工具、生成物入库/不入库的取舍、漂移检测与再生成纪律；
  同一份数据在编辑期有两个来源。
- **Estimated Effort**：中；工具本身与校验器相当。
- **Rejection Reason**：当前只有 40 条记录且作者就是开发者，收益为零；保留为
  未来 `tools/content/` 的只写工具（见决策 1 的逃生口），不进入 MVP。

### Alternative 3: 类型化 `Resource` + 运行时目录扫描

- **Description**：不写清单，启动时 `DirAccess` 枚举 `res://data/dish/` 等目录，
  把全部 `.tres` 读入。
- **Pros**：新增记录只需放文件，无登记步骤；目录即约定。
- **Cons**：导出包中文本资源被转换为二进制并带 `.uid` 侧车，扫描可能漏项或重复；
  加载顺序依赖文件系统；备份/实验文件会被意外带入；测试夹具难以隔离。
- **Estimated Effort**：略低于选定方案（少写清单类）。
- **Rejection Reason**：导出行为不确定性正是本项目最不能承担的风险；清单把
  “哪些记录属于游戏”变成显式契约，并让重复 id 报告顺序可复现。

### Alternative 4: MVP 支持内容热重载

- **Description**：监听 `.tres` 变化，重新校验并替换目录，不重启游戏。
- **Pros**：调参迭代快；改一个价格立刻能试。
- **Cons**：下游可能持有记录引用与每局冻结快照；中途替换破坏确定性与一局内
  数值一致性；需要原子交换、版本与失效通知；测试复杂度显著上升。
- **Estimated Effort**：高，且会牵连 RunFlow / MenuWaves / HUD 的生命周期。
- **Rejection Reason**：40 条记录下重启成本可接受；GDD OQ3 已倾向放到 Vertical
  Slice。MVP 用「重启生效」换取确定性。

## Consequences

### Positive

- 类型安全与可视化编辑免费获得：字段类型、枚举、资源引用由 Godot 与校验器共同约束。
- 启动门全有或全无，任何数据损坏都在构造 gameplay 前被拦下，并给出文件、id、
  字段、期望值与实际值。
- 清单让导出包的枚举行为确定，不依赖目录扫描与文件系统顺序。
- 下游只有一个查询面与一道门，测试可注入夹具路径，不碰真实数据目录。
- 发布后 id 基线让「改名导致存档进度失效」在 CI 就暴露，而不是上线后才发现。

### Negative

- 新增记录需要登记到清单；忘记登记只能在编辑器/CI 完整性测试里发现。
- 记录类要为每个字段写公开 getter，字段多时有样板代码；数组 getter 还要拷贝。
- MVP 没有热重载，调参需要重启，迭代变慢。
- 记录 schema 变更（字段增删改）需要新 ADR 并评估存档迁移，不能随手改。
- `NO_UNLOCKED_STATION` 从运行期兜底提升为启动错误，比解锁 GDD 的现状更严格。

### Neutral

- `BalanceData` 的「零个或两个」在清单模型下变成「单槽为空」或「结构上不可能」：
  GDD AC 30 的“两个”不再是可构造的失败态，而是被类型结构排除。
- 源文件仍是 `.tres`，导出时转二进制；没有自定义打包格式。
- `MilestoneData` 不进入 `DataConfig` schema，但与数据配置共用诊断词汇与启动门。
- 记录的 `_` 前缀存储字段在 Inspector 中可见；这是换取只读公开 API 的代价。

## Risks

- **只读属性语义未在 4.7.x 实测**：getter-only 赋值可能按预期被拒绝，也可能被
  静默忽略。缓解：列入 Verification Required 第一条；写第一个 story 前用最小
  脚本验证静态赋值与 `set()` 两条路径。若实测不符，回退方案是从查询 API 返回
  只读视图对象（保持 `DataConfig` 接口不变），或接受“约定 + 代码评审”并在 ADR
  修订中记录。
- **导出预设漏掉资源**：固定路径不构成导出依赖。缓解：实现指南第 11 条 + 导出包
  冒烟测试（加载全部清单并断言记录数）。
- **清单漏登记记录**：运行时不会发现。缓解：编辑器/CI 完整性测试扫描目录并对比
  清单；该测试只在开发环境运行。
- **记录仍可被内部字段写穿**：GDScript 无真正私有。缓解：`_` 前缀约定 + 代码
  评审 + 记录不离开 `DataConfig`；公开 API 的赋值有语言级只读约束。
- **单清单合并冲突**：多人同时新增记录会改同一个清单。缓解：按类型拆分清单；
  当前单人开发，冲突面很小。
- **发布后 id 改名漏网**：缓解：`published_ids.txt` 与 CI 对比；基线在首个公开
  版本生成并入库。
- **关闭/下线内容无法表达**：双向一致性禁止直接删除菜品。缓解：MVP 只做增量内容；
  首个发布后内容更新前必须解决（Open Questions 1）。
- **`NO_UNLOCKED_STATION` 阻断既有数据集**：缓解：它是两份 GDD 的建议；若确有
  合法用例，在 ADR 修订中降级为警告，而不是在代码里绕过。
- **本机无编辑器**：所有运行时结论未经实测。缓解：把 Verification Required 作为
  第一个垂直切片的前置 spike，未通过前不写依赖该语义的 story。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `data-config.md` | TR-data-config-001：四类类型化 Resource catalog，启动加载校验，失败无部分加载 | 决策 1、2、4；六类记录 + 显式清单；全有或全无状态机 |
| `data-config.md` | TR-data-config-002：查询 API 返回不可变记录，无运行时修改 | 决策 4；公开 getter-only 属性、防御性数组拷贝、`_` 前缀存储字段 |
| `data-config.md` | TR-data-config-003：`StationData` / `UpgradeData` 字段集合 | 决策 1；记录类与 GDD 字段一一对应，校验器覆盖取值范围 |
| `data-config.md` | TR-data-config-004：稳定 id 命名空间 `^[a-z][a-z0-9_]*$`，发布后改名禁止 | 决策 3；类型前缀、重复检测、`published_ids.txt` + CI 基线 |
| `data-config.md` | TR-data-config-005：`Unloaded→Loading→Validating→Ready/Failed`，`Ready` 门禁全部下游 | 决策 4、6；状态机、无部分发布、`Boot → Error` |
| `data-config.md` | TR-data-config-006：`display_name` 与可选 `Texture2D icon`，展示只读，缺失用占位 | 决策 8；警告不阻断、`get_dish_icon()` 解析器、记录不变 |
| `data-config.md` | Core Rule 7 / OQ3 / OQ4 / OQ5：启动校验、热重载、打包格式、目录组织 | 决策 1、2、5、9；目录布局已定、`tres` 源 + 导出二进制、MVP 不热重载 |
| `records-and-milestones.md` | TR-records-004：独立只读 `MilestoneData` 加载器，四种指标，单调锁存 | 决策 7；Records 拥有的 `MilestoneCatalog`，四种指标校验、id 升序、并入启动门 |
| `audio-feedback.md` | TR-audio-feedback-001 / 007（目录契约与内容缺陷启动阻断） | 决策 2 / 5；第六清单同门加载、`AUDIO_CUE_*` 诊断、全有或全无 |
| `unlocks-and-stations.md` | Rule 4 建议：至少一个 `unlock_cost = 0` 的档口加为启动校验 | 决策 5；`NO_UNLOCKED_STATION` ERROR，保留 `NoPlayableStation` 防御路径 |
| `architecture.md` | Boot flow：`DataConfig.load()` → `ready` → `Save.load()`；失败 `Boot → Error` | 决策 6；组合根编排，里程碑目录插入 `DataConfig Ready` 与 `Save.load()` 之间 |

## Performance Implications

- **CPU**：启动期一次性 O(n) 校验；引用检查使用字典，前置环检测 O(n+e)。运行时
  每次查询是字典查找，无每帧成本。记录 getter 是内联访问；数组 getter 做小拷贝，
  只在菜单/波次构建等低频路径调用。
- **Memory**：300 条记录的元数据 < 2 MB；纹理是导入资源，被引用而非复制。
- **Load Time**：300 条记录下 `Loading + Validating` p95 ≤ 250 ms，其中 Validating
  p95 ≤ 100 ms（GDD AC 22，最小规格 PC、非调试构建）；MVP 约 40 条记录远低于预算。
- **Network**：无。

## Migration Plan

当前没有代码，属于绿地实现。落地顺序：

1. 建 `src/foundation/data_config/` 与记录/清单类，写入空清单。
2. 先实现纯函数 `CatalogValidator` 与单元测试（不碰文件 I/O）。
3. 实现 `DataConfig` 状态机与固定路径加载。
4. 实现 `MilestoneCatalog` 与 Records 访问器。
4b. 实现 `AudioCueCatalogResource` 读取与 ADR-0009 D7 校验（5b 子阶段）。
5. 接入 `AppRoot` 启动编排与 `Boot → Error` payload。
6. 生成 `data/published_ids.txt` 基线（首个公开版本前）。

**Rollback plan**：`DataConfig` 的查询 API 是接缝。若类型化 Resource 方案被证伪，
可在同一 API 后面替换为 JSON/CSV 解析器，下游系统不需要改动；记录类与清单类
保留为作者格式，或整体迁移。回滚的判定点是 Verification Required 第 1 条与导出包
冒烟测试。

## Validation Criteria

- GDD AC 1：合法夹具 → `Unloaded→Loading→Validating→Ready`，四类查询可取到记录。
- GDD AC 2 / 17 / 18：重复 id、1 条坏记录 + 99 条好记录、`Failed` 后再加载 →
  整体失败、无部分记录、状态仍为 `Failed`。
- GDD AC 19：对 `get_dish(id).price` 赋值 → 被拒绝或抛错；再次读取原值；两次查询
  `is_same()` 为 true。
- GDD AC 22：300 条记录冷启动 Validating p95 ≤ 100 ms。
- GDD AC 24：`published_ids.txt` 中的 id 全部仍存在；改名/删除使测试失败。
- GDD AC 28 / 29 / 30：前缀错误失败；`base_capacity` 边界；`BalanceData` 单槽为空
  失败（“两个”由单槽结构排除）。
- GDD AC 3–16 / 25–27：各字段范围、标签、枚举、前置环、双向一致性、池重复。
- 新增：清单遗漏记录的目录扫描测试（编辑器/CI 专用）失败。
- 新增：里程碑目录在 `DataConfig` 之前加载 → 断言启动顺序错误；里程碑失败 →
  `Boot → Error` 且未构造 gameplay。
- 新增：缺失图标 → `Ready`、`errors() = 0`、`warnings() = 1`，`get_dish_icon()`
  返回非空占位纹理。
- 新增：导出包开启二进制转换后，六个清单全部加载且记录数与源数据一致。
- 新增：cue 目录含重复 id / 未知 bus / 空 streams / 越界 priority → `Failed`，
  报告含对应 `AUDIO_CUE_*`，99 条好记录也不可见。
- 新增：300 条记录下 `Failed` 报告包含被跳过阶段的 `CHECKS_SKIPPED`，不得静默。

## Related

- `docs/architecture/architecture.md` — 分层、启动顺序、API Boundaries、原则 7（数据驱动）。
- `design/gdd/data-config.md` — 四类记录与音频 cue 目录、字段、校验规则、状态机、AC。
- `design/gdd/records-and-milestones.md` — 里程碑目录归属与 `MilestoneData` 字段。
- `design/gdd/unlocks-and-stations.md` — `NoPlayableStation` 与免费档口建议。
- `design/gdd/save-system.md` — 规则 2 / 14：只存 id、未知 id 原样保留。
- ADR-0002（存档，待写）— 依赖本 ADR 的 id 稳定性与 `Ready` 前提。
- ADR-0004（服务组合，待写）— 组合根持有并注入 `DataConfig`。
- `docs/engine-reference/godot/VERSION.md`、`breaking-changes.md`、
  `current-best-practices.md`、`deprecated-apis.md`、`modules/ui.md`。

## Open Questions

1. **发布后如何下线一条内容？** 双向一致性要求每道菜必须在其档口池中、且池中每项
   必须指回该档口，因此不能靠“从池里移除”退休一道菜。可选方案：给 `DishData`
   增加 `retired: bool` 并放宽双向规则；或给档口增加“当前轮换池”。这需要 schema
   变更与新 ADR，必须在首次发布后的第一次内容更新前解决。
2. **`NO_UNLOCKED_STATION` 是否保持启动错误？** 本 ADR 选择 ERROR（两份 GDD 的
   建议）。若设计上允许“首次启动没有任何免费档口”的合法状态，应降级为警告。
3. **`published_ids.txt` 的生成时机**：本 ADR 定为首个公开版本发布前生成并入库；
   若项目在 Vertical Slice 就有外部测试者，基线应提前到第一次分发可读存档之前。
4. **记录数量上限**（`data-config.md` OQ1）：影响内存与加载预算；当前 40 条不构成
   问题，超过 300 条时需要重新测 AC 22。
5. **`DataResult` 与未来共享 Result 约定**：若 ADR-0004 定义项目级 `Result`，
   `DataConfig` 必须采用同一形状，但保持本 ADR 的错误词汇与语义。
6. **GDD 措辞澄清**：`data-config.md` 的 Game Feel 写“改一个 `.tres` 数值，下一局
   立刻按新数值结算，不重启”，与本 ADR 的“每进程加载一次”冲突。建议改为“重启后
   下一局立即使用新数值，无缓存不一致”，并保留 OQ3 把热重载放到 Vertical Slice。
7. **里程碑是否永远没有图标？** 当前 `MilestoneData` 无 `icon` 字段；若界面后续需要
   图标，属于 schema 变更，应走新 ADR。
