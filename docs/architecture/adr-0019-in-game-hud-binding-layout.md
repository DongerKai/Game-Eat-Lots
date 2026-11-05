# ADR-0019: 局内 HUD 的数据绑定、布局与呈现契约（In-Game HUD Binding, Layout & Presentation）

## Status

Proposed

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。
> 其他 agent 与任何 skill 都不能自行设置。** 引用 `Proposed` ADR 的 story 会被置为
> `Blocked`，epic 要求 `Accepted`，所以这个字段必须有明确的负责人。

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

- 用户（`modes.automation: autonomous`，由用户先行授权）
- `godot-specialist`（CanvasLayer 逆缩放、`Control.focus_neighbor_*`、双焦点、`Tween` 的 4.7 行为校验）
- `technical-director` 战略审阅：跳过（`review_mode: lean`）

## Summary

`in-game-hud.md` 把 HUD 定义成一块只读仪表盘：所有数字由上游发布，HUD 只负责读取、排版、合成与呈现；动作提交后同一帧出新值；缺失值显示 `—` 而不是 0；卡片按波次顺序陈列且永不推荐；焦点由 HUD 解析、动作合法性归餐局流程。这些约束在 GDD 里是行为，在代码里必须变成**结构**，否则实现迟早会滑向「HUD 自己求和」「HUD 自己判可吃」「动画先播、数字后改」三种最典型的双真相。

本 ADR 决定：

1. **`HudHost` 是唯一的 HUD 适配器**（`CanvasLayer`，layer 10，由 `AppRoot` 挂载/卸载），内部由**纯投影层**（`HudSource` → `HudFrameBuilder` → `HudFrame`）与**纯视图层**（`HudView`）组成。`HudFrame` 在一次事件内构造、构造后不可变，所有元素引用同一个 `run_id` 与 `action_seq`。
2. **上游只读一次**：`HudSource.capture()` 在一次同步调用里取一份 `RunSnapshot` 并读完所有依赖，构造 `HudFrame` 时不再访问任何服务。两帧之间不轮询上游；`_process` 只用于 ADR-0005 规定的 `ui_accept` / `ui_cancel` / `game_skip_wave` 轮询。
3. **HUD 不计算玩法值**：`running_net` / `shortfall` 只读 `RunFlow.snapshot()`；卡片字段只读 `DishValues` / `CloyingAndTaste` / `StomachCapacity` / `MenuWaves`；唯一的派生值是 GDD 明确允许的显示量 `short_cells`、列数、槽位数与容量比例。HUD 代码中不存在价格求和、第三份净利累加器、`unit_value` 或推荐顺序。
4. **布局是纯函数**：`HudLayout` 在窗口像素坐标里按 `hud_safe_margin` / `hud_ui_scale` / `hud_card_columns` / `hud_turn_slots` / `hud_capacity_cells` / `hud_area_usage` 计算全部矩形；`HudHost` 用一次逆缩放抵消 `canvas_items` 的内容缩放，使 1080p 的 `s = 1.5` 是最终缩放而不是二次放大。
5. **锚点与受保护矩形是 HUD 的公开只读接口**：`get_anchor_rect(anchor_id)` 支持六个 id，`get_protected_rects()` 固定返回「回合 / 容量 / 净利」三个矩形；首局引导只拿矩形，不知道 HUD 布局。
6. **焦点、确认面板与输入**：卡片与操作按钮是真实可聚焦控件，不可吃/不可用的卡仍可获焦并读出原因；结束取餐确认面板是 HUD 内的焦点陷阱，**不**调用 `push_context(Modal)`；`game_skip_wave` 在面板打开期间被吞掉。
7. **数字格式与颜色冗余是代码契约**：`HudFormat` 产出 GDD 规定的固定字符串（`+39` / `−46` / `还差 46 元` / `已回本` / `—` / `剩余 6 / 10`），`HudPalette` 的每个语义色都必须有文字或符号作为第二通道，主题对比度 ≥ 4.5:1。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；本机编辑器 `NOT DETERMINED`） |
| **Domain** | UI |
| **Layer** | Presentation |
| **Knowledge Risk** | HIGH — 4.6 双焦点分离、4.7 `canvas_items` 默认值、`CanvasLayer` 与根内容缩放的乘算关系均未在参考库中完全覆盖 |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`modules/ui.md`、`modules/input.md`、`breaking-changes.md`、`deprecated-apis.md`、`current-best-practices.md` |
| **Post-Cutoff APIs Used** | 4.6 双焦点分离（键盘/手柄焦点与鼠标悬停各自独立）；4.7 `canvas_items` / `expand` 默认值；`Control.focus_neighbor_*`（与 ADR-0003 相同） |
| **Verification Required** | ① `CanvasLayer.transform` 取根内容缩放的倒数后，HUD 子控件在 1080p 按窗口像素 1:1 绘制，且 GUI 命中测试与焦点几何解析仍然正确；② 同一逆缩放下 `get_global_rect()` / `get_global_transform()` 返回的矩形与 HUD 布局坐标一致；③ `Control.focus_neighbor_*` 在逆缩放层内按声明顺序跳转，不按缩放后的几何回退；④ 4.6 下 `has_focus()` 是否会被鼠标悬停满足（ADR-0003 Verification ⑦）；⑤ `CanvasLayer` 上的 `Tween` 在 `HudHost.process_mode = INHERIT` 暂停时确实停在最后一帧；⑥ 4.7 下 `Control.accessibility_*` / 读屏顺序对本 ADR 的卡片与按钮可用 |

> **Knowledge Risk 为 HIGH。** 升级引擎版本时本 ADR 必须重新验证；若 `CanvasLayer` 逆缩放不可行，按 D5 的降级路径改为「设计单位布局 + 引擎缩放」，并把 `hud_ui_scale` 的上限 2.0 降级为警告，不得静默接受二次缩放。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0003（`CanvasLayer` 分层、`FocusSpec`、双焦点、`ScreenView` 的 `present → apply → grab_focus` 顺序）— HUD 复用其焦点与宿主语义；ADR-0005（`InputRouter` 的 `Run` 轮询契约、`ui_accept` / `ui_cancel` 消费、焦点/悬停查询）— HUD 是 `Run` 上下文唯一动作读者；ADR-0007（`RunFlow.snapshot()` 是 `run_state` / `running_net` / `shortfall` / `action_seq` 的唯一发布面）；ADR-0008（`PresentationPrefs` 与 `text_scale_percent` / `hud_opacity_percent` / `reduce_motion` 的消费契约）；ADR-0012（`capacity_remaining` / `can_fit`）；ADR-0014（`preview` / `group_counts`）；ADR-0006（`visible_dishes` / `preview_dishes` / `hidden_counts`）；ADR-0010（`get_dish` / `price` / `satiety`，**不读 `unit_value`**） |
| **Enables** | 局内 HUD（#15）的全部实现 story；首局引导（#18）的锚点与受保护矩形；音效反馈（#9）的 `dish_action_rejected` / `number_roll_*` 订阅；`TR-run-flow-007` 的 HUD 消费面 |
| **Blocks** | 任何需要 `HudFrame`、`get_anchor_rect`、`get_protected_rects` 或 HUD 焦点验收的 story；ADR-0020（首局引导）的锚点 story |
| **Ordering Note** | ADR-0013 的 Related 已把「ADR-0019 HUD 数据绑定」列为 `ActionResult` 与快照的消费方；本 ADR 不改 `RunFlow` 的写入口，只在 D3 追加一个**只读**的 `action_availability()` 投影（见 GDD Sync Check 第 2 条）。拓扑序保持 0001 → 0002 → 0005 → 0003 → 0004 → 0006..0009 → 0010..0018 → 0019 → 0020 |

## Context

### Problem Statement

`in-game-hud.md` 的 22 条 Core Rules 与 9 个 Formulas 已经把「显示什么」写得很细，但实现层仍有四个会直接制造双真相的缺口：

1. **一份帧还是逐元素读取？** 如果净利读一次 `RunFlow.snapshot()`、容量读一次 `StomachCapacity`、卡片读一次 `MenuWaves`，三次读取之间夹着一次信号派发，就会得到「净利是新值、容量是旧值」的帧。GDD 规则 3 要求一份不可变 `HudFrame`，但帧的**构造边界**与**字段所有权**没有落到类型上。
2. **HUD 到底能算什么？** `running_net` 已有 `RunFlow` 累加器，`shortfall` 有结算纯函数，`effective_satiety` 有腻度服务，`can_fit` 有容量服务。若 HUD 为了「显示一致」自行把价格相加、把 `max(0, −net)` 再算一遍，第三份真相就出现了；GDD 规则 6 禁止，但没有规定**哪些派生显示量是允许的**。
3. **布局与缩放谁说了算？** `canvas_items` 已经缩放根视口；若 HUD 再按窗口像素套用 `hud_ui_scale`，1080p 会得到 2.25× 而不是 1.5×（ADR-0003 Risk 4 已登记）。同时，150% 字号、超宽屏、`hud_opacity` 三个设置会改变布局，必须有唯一的重排入口，不能每个卡片自己算。
4. **焦点与确认面板归谁？** HUD 在 `Run` 上下文，`InputRouter` 明确不投递 `ui_accept`（ADR-0005 D4），HUD 必须自己轮询；而结束取餐确认面板又不能是 `Modal`（GDD 规则 16）。焦点陷阱、`ui_cancel` 关闭、`game_skip_wave` 吞掉三者必须由一个可测模型统一实现，否则「面板打开时按跳过键」会变成未定义行为。

### Current State

- 仓库没有 `.gd`；`project.godot` 只有引擎与渲染设置；`run/main_scene` 未设置。
- ADR-0003 已固定 `HudHost` 为 `CanvasLayer`（layer 10），由组合根在进入 `Run` 时挂载、离开时卸载；`UIRoot` 没有 `HudHost` 引用。
- ADR-0004 已固定 `HudHost` 为 `run_started` / `action_committed` / `run_ended` / `run_faulted` / `run_discarded` 的处理器，且 `process_mode = INHERIT`（暂停时停在最后一帧）。
- ADR-0005 已固定 `Run` 上下文里 `ui_accept` / `game_skip_wave` 的唯一读者是 `HudHost`；`ui_accept` / `ui_cancel` 在产生时即消费，不走 `BaseButton` 自动激活；导航重复由输入层生成。
- ADR-0007 已固定 `RunFlow.snapshot()` 发布 `run_id`、`run_state`、`frozen`、`action_seq`、`running_net`、`shortfall`、`run_seed`；但没有 snapshot 的完整字段表与 `action_availability`。
- ADR-0008 已固定 `PresentationPrefs`（`text_scale_percent` / `hud_opacity_percent` / `reduce_motion`）与 HUD 的消费关系；但没有规定组合根如何把它送给 `HudHost`。
- `design/ux/hud.md` 尚不存在；本 ADR 只固定代码契约与布局下限，视觉规格的最终稿由 `/ux-design` 产出。

### Constraints

- **同帧**：动作提交后的第一个渲染帧必须显示新值（GDD 规则 4、AC 41）；帧构造与视图应用必须在同一同步调用内完成，禁止 `await` / `call_deferred()` / 异步加载资源。
- **只读**：HUD 不得调用 `spend_turn()` / `consume()` / `advance()` / `end_pickup()` / `record_eat()` / `initialize()` / `end_run()` / `commit()` / `mark_dirty()`；唯一的写是 `RunFlow.submit_action`。
- **无轮询上游**：`_process` 只允许读输入；GDD AC 40 要求空闲 60 帧不重建帧。
- **无单例**：全部上游以构造参数注入 `HudSource`；禁止 `get_node("/root/…")`、`static var` 服务或依赖袋（ADR-0004 D3）。
- **无头可测**：`HudFrameBuilder` / `HudLayout` / `HudFormat` / `HudFocusResolver` / `HudConfirmModel` 必须不引用 `Node`，能在 GdUnit 无渲染环境下断言文本与矩形。
- **1280×720 基准**：`run` 命令以该窗口运行；最低支持 1280×720；超宽屏额外宽度给主区，不放大字号。
- **颜色不是唯一通道**：每个语义色都必须有文字或符号冗余（GDD 规则 19）。

### Requirements

- 必须给出 `HudFrame` 的字段所有权、构造边界与不可变规则。
- 必须给出「一次采集、一次构造、一次应用」的调用顺序，并证明空闲时零上游读取。
- 必须列出允许的派生显示量，并禁止第二份净利求和、`unit_value`、自动排序与推荐。
- 必须让全部布局公式在窗口像素坐标下可计算、可断言，并处理 `canvas_items` 的二次缩放。
- 必须让不可吃卡与不可用按钮仍可获焦、可读原因，只有流程状态才真正禁用。
- 必须让结束取餐确认面板成为 HUD 内焦点陷阱，且 `game_skip_wave` 在面板打开期间不产生任何意图。
- 必须提供 `get_anchor_rect(anchor_id)` 与三个受保护矩形，且矩形在 150% 字号与超宽屏下仍可判定。
- 必须让全部玩家可见文本来自一处文本表，数字格式逐字符固定。
- 必须把五条运行时不变式变成调试断言 + 正式构建的最后合法渲染降级。

## Decision

### D1 — 组成：`HudHost` + 纯投影层 + 纯视图层

```
AppRoot (组合根)
  ├── HudHost (CanvasLayer, layer 10)          # 适配器：生命周期、信号、输入轮询、锚点
  │     ├── HudView (Control)                  # 视图：卡片/按钮/条/详情行，FocusSpec 应用
  │     │     ├── TopBar
  │     │     ├── CardGrid
  │     │     ├── RightRail
  │     │     ├── DetailRow
  │     │     └── ConfirmPanel (默认隐藏)
  │     └── WorldScrim (ColorRect)             # 只压暗玩法画面，不压暗 HUD 文本
  ├── HudSource (RefCounted)                   # 单次采集：RunFlow + 4 个上游 + DataConfig
  └── HudFrameBuilder / HudLayout / HudFormat / HudFocusResolver / HudConfirmModel
```

- **`HudHost` 是唯一允许调用 `RunFlow.submit_action` 的 HUD 类**（ADR-0005 D9 已把该调用点限定在 `hud_host.gd`）。
- **`HudSource` 是唯一允许读上游的类**；`HudFrameBuilder` 只接收已经采集好的 `HudInputs`，因此无法在构造过程中「顺手再查一次」。
- **`HudView` 只读 `HudFrame` 与 `HudFocusResolver` 的结果**；它不持有 `RunFlow` / `MenuWaves` / `DishValues` / `CloyingAndTaste` / `StomachCapacity` 的引用。视图代码中出现任何上游类名即视为架构违规（静态检查）。
- **`HudHost` 不绘制任何东西**：它没有 `_draw`、没有主题、没有卡片节点；它只负责挂载/卸载 `HudView`、转发事件、轮询输入、返回锚点矩形。
- **`WorldScrim` 在 `HudView` 内容之下、玩法画面之上**：确认面板打开时压暗 60%，但 HUD 数字与面板保持原亮度与原对比度（GDD Visual/Audio Requirements「确认面板之外的 HUD 数字仍然可读」）。

### D2 — `HudFrame`：一次事件、一份快照、构造后不可变

`HudFrame` 是 `RefCounted`，字段全部在 `_init` 中赋值，**没有 setter、没有 `apply()`、没有 `refresh()`**。数组字段在构造时 `duplicate(true)`；`HudCardView` / `HudNetView` 等子对象同样只在构造中赋值。视图与测试只能读。

```gdscript
class_name HudFrame extends RefCounted

enum State { HIDDEN, LOADING, ACTIVE, FROZEN, ENDING, SETTLING, HANDOFF, FAULTED }
enum Source {
    RUN_STARTED, ACTION_COMMITTED, RUN_ENDED, RUN_SETTLED, RUN_FAULTED,
    RUN_DISCARDED, STATE_SYNC, RELAYOUT,
}
enum Status { OK, UNAVAILABLE }

var built_frame: int              # Engine.get_process_frames()
var source: int                   # Source
var state: int                    # State
var interactive: bool             # 见 D8；与 state/frozen/degraded 共同决定
var degraded: bool                # 任一字段 UNAVAILABLE（只影响动作，不影响其余显示）

var run_id: int                   # 本帧唯一身份
var action_seq: int               # 本帧唯一身份
var station_id: StringName
var station_name: String
var end_reason: int               # EndReason.NONE 表示非 Ending
var fault_code: StringName
var run_seed: int                 # 仅 debug 构建非 0；release 恒 0

var turns: HudTurnsView
var capacity: HudCapacityView
var net: HudNetView
var wave: HudWaveView
var cards: Array[HudCardView]     # 顺序 = visible_dishes 顺序
var cloy_groups: Array[HudCloyGroupView]
var preview: Array[HudDishBrief]
var actions: HudActionsView
var layout: HudLayoutRects
var settings: HudSettingsView
var initial_focus_id: StringName

func _init(...) -> void           # 唯一构造入口；全部字段在此赋值
```

**字段所有权**：

| 字段 | 来源 | HUD 是否允许改写 | 说明 |
|---|---|---|---|
| `run_id` / `action_seq` / `run_state` / `frozen` / `run_seed` / `end_reason` | `RunFlow.snapshot()` | 否 | 帧的身份；不允许跨快照混用 |
| `turns_remaining` / `turns_total` / `turns_used` | 快照（`TimeTurns`） | 否 | 不推算耗尽 |
| `capacity_used` / `capacity_total` / `capacity_remaining` | 快照（`StomachCapacity`） | 否 | 不自行相减 |
| `running_net` / `shortfall` | 快照（`RunFlow` 的累加器，口径由 ADR-0015 定义） | 否 | 不求和、不维护第三份 |
| 卡片 `price` / `satiety` / 图标 / 名称 | `DishValues.get_dish` + `DataConfig.get_dish_icon` | 否 | 不请求 `unit_value` |
| 卡片 `extra_satiety` / `effective_satiety` / `is_cleansing` / `cleanse_target` | `CloyingAndTaste.preview` | 否 | 不重算腻度 |
| 卡片 `can_fit` | `StomachCapacity.can_fit` | 否 | 权威可吃判定 |
| 卡片 `block_reason` / `short_cells` | **HUD 派生显示量**（GDD `hud_short_cells`） | 是 | 只生成文案，不参与判定 |
| `wave_index` / `wave_count` / `visible_dishes` / `preview_dishes` / `hidden_counts` | `MenuWaves` | 否 | 不排序、不过滤 |
| `actions.*` | `RunFlow.action_availability()` | 否 | 可用性与代价归 RunFlow / MenuWaves |
| `layout.*` / `settings.*` | `HudLayout` / `PresentationPrefs` | 否 | 纯布局与呈现设置 |
| `initial_focus_id` | HUD 派生（首张可吃菜，否则首张卡） | 是 | 只用于首帧焦点，不写入玩法 |

**不可变性**：`HudFrame` 的字段类型全部是值、`StringName`、`Rect2`、`RefCounted` 只读子对象或深拷贝数组；没有任何方法把内部数组引用交给调用方。测试以「构造后对副本写入、原帧哈希不变」验证。

### D3 — 数据源：一次采集，不轮询，不求和

`HudSource.capture(caller)` 是 HUD 对上游的唯一读入口。它**在一次同步调用内**完成：

```gdscript
class HudSource extends RefCounted

func _init(run_flow: RunFlow, menu_waves: MenuWaves, dishes: DishValues,
           cloying: CloyingAndTaste, capacity: StomachCapacity,
           data: DataConfig, debug_build: bool) -> void

func capture(caller: StringName) -> HudInputs
```

固定顺序：

1. `snap := run_flow.snapshot()` —— **只取一次**；记录 `run_id`、`action_seq`、`run_state`、`frozen`、`end_reason`、`run_seed`、回合/容量读数、`running_net`、`shortfall`。
2. `visible := menu_waves.visible_dishes()` —— 只取一次；按返回顺序，不排序、不去重、不过滤。
3. 对每道可见菜：`dishes.get_dish(id)` → 名称/图标/`price`/`satiety`；`cloying.preview(id)` → `extra` / `effective` / `is_cleansing` / `cleanse_target`；`capacity.can_fit(effective)` → `can_fit`。每项只读一次。
4. `menu_waves.preview_dishes()` / `hidden_counts()` / `wave_index()` / `wave_count()`。
5. `cloying.group_counts()` —— 上游已按规范序返回，HUD 不再排序。
6. `run_flow.action_availability()` —— 跳过/结束取餐的可用性与原因（见 GDD Sync Check 第 2 条）。
7. `data.get_station( station_id )` 的显示名与 `data.get_dish_icon(id)`；HUD 不直接查数据目录。

**必须**：`capture()` 不写任何上游状态；不调用 `preview()` 之外的腻度方法；不调用 `can_fit()` 之外的容量方法；不调用任何 `MenuWaves` 写方法。

**必须**：`HudSource` 返回的 `HudInputs` 携带 `snap.run_id` 与 `snap.action_seq`；`HudFrameBuilder` 把这两个值原样写入 `HudFrame`，从而在结构上保证「一帧一快照」。

**禁止**：HUD 代码中出现 `running_net +=`、`shortfall =`、`price +`、`sum(`、`reduce(`、`unit_value`、`opt_value`、`recommend`、`best`。允许的派生只有 `short_cells`、`columns`、`slots`、`cells`、`fill_ratio`、`area_usage` 与格式化字符串。

**失败语义**：

- `snapshot()` 不可用（`run_id = 0` / `run_state = Idle` / `Initializing`）→ `HudFrame.State.LOADING`，所有数字 `UNAVAILABLE`，`interactive = false`。
- 单个字段读取失败（例如 `running_net` 查询失败）→ 该字段 `UNAVAILABLE`，`degraded = true`，其余字段仍来自同一份快照；该字段显示 `—`，**不回退到自己求和、不显示旧值**。
- 整次采集失败（抛错 / 帧构造不变式失败）→ `HudHost` 保留 `_last_legal_frame`，不替换画面，`interactive = false`，调试构建记录一次；正式构建继续显示最后合法帧，不崩游戏。
- `run_state = Faulted` → `State.FAULTED`，受影响字段 `UNAVAILABLE`，顶栏显示 `fault_code` 文本，`interactive = false`。

### D4 — 卡片合成：字段同屏、顺序固定、不推荐

`HudCardView` 的字段来自四个上游，**必须同屏**，不得折叠进悬浮提示或详情页：

```gdscript
class HudCardView extends RefCounted
var dish_id: StringName
var display_index: int            # visible_dishes 中的位置；焦点恢复用
var name: String
var icon: Texture2D
var price: int
var base_satiety: int
var extra_satiety: int
var effective_satiety: int
var can_fit: bool
var block_reason: int             # NONE / CAPACITY / CAPACITY_ZERO
var short_cells: int              # 仅显示；can_fit 时恒 0
var taste_tags: Array[StringName]
var cloying_group: StringName
var is_cleansing: bool
var cleanse_target: StringName
```

- **顺序**：`cards[i].display_index == i`，严格等于 `visible_dishes()` 的返回顺序。HUD 不做默认排序、不做可吃性过滤、不按价格/饱腹/单位价值重排。
- **排序扩展**：GDD 规则 9 允许玩家主动按价格或饱腹排序；MVP **不实现**排序 UI。若未来加入，它只能是视图层的置换，`display_index` 仍记录 `visible_dishes` 的规范位置，焦点恢复与锚点 `grid:first_card` 仍按规范顺序解析；不得引入「推荐排序」。
- **`block_reason`**：
  - `CAPACITY`：`can_fit = false`，文案 `需要 X 格，剩 Y 格`，详情行补充 `还差 short_cells 格`；
  - `CAPACITY_ZERO`：`capacity_total = 0`，文案 `总容量为 0`；这是错误态，不静默当成「吃不下」；
  - `NONE`：可吃。
- **不可吃卡不是禁用控件**：`can_fit = false` 只置灰、写原因，卡片仍可获焦、可悬停、可读全部字段；只有 `State ∈ {ENDING, SETTLING, HANDOFF, FAULTED}` 或 `frozen = true` 时 `focus_mode = FOCUS_NONE` 且 `mouse_filter = IGNORE`（GDD 规则 10）。
- **禁止**：卡片出现 `unit_value`、`元/格`、推荐顺序、最优解、排序建议；静态 grep 命中即失败。
- **图标**：只来自 `DataConfig.get_dish_icon(id)`；缺失时由 DataConfig 返回占位图标，HUD 不实现 fallback、不修改 `DishData`（ADR-0001 D8）。

### D5 — 布局：窗口像素、一次逆缩放、纯函数

#### D5.1 坐标空间

`HudHost` 在 `canvas_items` + `expand` 下按**窗口像素**布局。它读取根视口的内容缩放 `c`（`get_viewport().get_final_transform().get_scale().x`，`c > 0`），并把 `CanvasLayer.transform` 设为 `Transform2D(0, Vector2(1.0 / c, 1.0 / c), 0, Vector2.ZERO)`。

- `HudView.size = viewport_size`（窗口像素）；子控件坐标即窗口像素。
- 布局公式中的 `W` / `H` 是窗口像素；`s = hud_ui_scale` 是最终缩放，不再乘 `c`。
- 若 `c` 不可得或 `<= 0`，按 `c = 1` 处理并记一次 WARNING；不得让 HUD 消失。
- **降级路径**：若 4.7 实机验证发现 `CanvasLayer` 逆缩放会破坏 GUI 命中测试（Verification ①/②），改为「设计单位布局 + 引擎缩放」：`HudLayout` 以 1280×720 设计单位计算，`s` 固定 1.0，`hud_ui_scale` 只作为超宽屏告警；此时 AC 10（2560×1080 的 1.5 不放大字号）由 `expand` 的额外宽度满足，但 2160p 的 2.0 上限失效，必须在验收里标注为降级项。

#### D5.2 主题资源是唯一常量来源

全部布局常量存放在 `HudTheme`（`res://data/config/hud_theme.tres`）：

```gdscript
class_name HudTheme extends Resource
@export var safe_margin_ratio: float = 0.05
@export var safe_margin_min_px: int = 32
@export var safe_margin_max_px: int = 96
@export var top_bar_height: Array[int] = [72, 84, 96]      # 100 / 125 / 150%
@export var rail_width_px: int = 280
@export var card_min_width_px: int = 200
@export var card_height: Array[int] = [176, 198, 220]      # 100 / 125 / 150%
@export var card_gap_px: int = 16
@export var detail_row_height_px: int = 44
@export var main_max_width_px: int = 1600
@export var turn_block_width_px: int = 320
@export var capacity_bar_width_px: int = 600
@export var slot_gap_px: int = 4
@export var cell_gap_px: int = 2
@export var number_roll_ms: int = 250
@export var focus_ring_px: int = 2
@export var contrast_min: float = 4.5
@export var area_max: float = 0.28
```

代码中**禁止**出现上述字面量；测试用 `HudTheme.default()` 与生产资源逐字段比对。

#### D5.3 公式实现

`HudLayout.compute(ctx, theme, input) -> HudLayoutRects` 是纯函数；`ctx` 含 `viewport: Vector2i`、`content_scale: float`、`prefs: PresentationPrefs`、`debug_build: bool`；`input` 含 `card_count`、`turns_total`、`capacity_total`。

```
m = clamp(round(safe_margin_ratio * H), safe_margin_min_px, safe_margin_max_px)
s = clamp(min(W / 1280.0, H / 720.0), 1.0, 2.0)
τ = text_scale_percent / 100.0

top_h     = theme.top_bar_height[τ_index] * s
rail_w    = theme.rail_width_px * s
gap       = theme.card_gap_px * s
detail_h  = theme.detail_row_height_px * s
top_bar   = Rect2(m, m, W - 2*m, top_h)
content_y = m + top_h + gap
detail    = Rect2(m, H - m - detail_h, W - 2*m - rail_w - gap, detail_h)
rail      = Rect2(W - m - rail_w, content_y, rail_w, H - m - content_y)
raw_grid_w = W - 2*m - rail_w - gap
grid_w    = min(raw_grid_w, theme.main_max_width_px * s)
grid_x    = m + round((raw_grid_w - grid_w) / 2.0)
grid_h    = detail.position.y - gap - content_y
grid      = Rect2(grid_x, content_y, grid_w, grid_h)

w_min = theme.card_min_width_px * s * τ
n     = clamp(floor((grid_w + gap) / (w_min + gap)), 1, 4)
w     = (grid_w - (n - 1) * gap) / n
h     = theme.card_height[τ_index] * s
card_rects[i] = Rect2(grid.position.x + (i % n) * (w + gap),
                      grid.position.y + floor(i / n) * (h + gap), w, h)
```

- `grid` 的宽度上限 `main_max_width_px * s` 只在超宽屏触发；额外宽度留在主区两侧，右栏贴右。
- `card_rects` 只按 `visible_dishes` 顺序生成，不做任何排序。
- **回合块**（`B = min(theme.turn_block_width_px * s, top_bar.size.x * 0.4)`）：

```
slots = max(1, min(turns_total, 12))          # turns_total = 0 时 slots 取 1 只为避免除零，不渲染
k     = clamp(floor((B - (slots - 1) * slot_gap_px * s) / slots),
              8 * s, 24 * s)
若 turns_total = 0 或 turns_total > 12：
    slots = 0；不画槽位；只画比例条 remaining_ratio = turns_remaining / turns_total（total > 0 时）
```

- **容量块**（`C = min(theme.capacity_bar_width_px * s, top_bar.size.x - capacity_label_w)`）：

```
cells = max(1, min(capacity_total, 24))
k     = clamp(floor((C - (cells - 1) * cell_gap_px * s) / cells),
              6 * s, 14 * s)
fill_ratio = capacity_total > 0 ? float(capacity_used) / float(capacity_total) : 0.0
若 capacity_total > 24：只画前 24 格；数字标签仍为精确值
若 capacity_total = 0：cells = 0；显示 0 / 0；不做除法
```

- **面积**：`area_usage = (top_bar.size.x * top_bar.size.y + rail.size.x * rail.size.y) / (W * H)`；`Active` 下必须 `≤ theme.area_max`（@1280×720 为 26.5%）。调试构建超限记 `HUD_INVARIANT` 6。
- **中央 40%**：`central = Rect2(W * 0.3, H * 0.3, W * 0.4, H * 0.4)`；`top_bar` / `rail` / `detail` / `cloy_block` / `preview_block` 与 `central` 不相交；只有 `card_rects` 与 `confirm_panel` 允许进入该区域。调试构建断言。
- **主区最大宽度**：`main_max_width_px` 是参考像素；`* s` 后是窗口像素。GDD Edge Case 的「1600 px」按参考像素解释。
- **所有矩形取整**：`round` 到整数像素，避免 0.5 px 造成的 1 px 漂移；同一输入必须得到逐像素相同的矩形（无头测试以 `Rect2 ==` 断言）。

#### D5.4 `HudLayoutRects`

```gdscript
class HudLayoutRects extends RefCounted
var top_bar: Rect2
var status_row: Rect2
var capacity_row: Rect2
var grid: Rect2
var card_rects: Array[Rect2]
var rail: Rect2
var detail: Rect2
var confirm_panel: Rect2
var wave_block: Rect2
var turns_block: Rect2
var net_block: Rect2
var capacity_block: Rect2
var cloy_block: Rect2
var preview_block: Rect2
var actions_block: Rect2
var protected_rects: Array[Rect2]   # [turns_block, capacity_block, net_block]，顺序固定
func area_usage(viewport: Vector2) -> float
```

**必须**：`protected_rects.size() == 3` 且顺序恒为「回合 / 容量 / 净利」；任何布局模式下（含 150% 字号、超宽屏、1080p）都必须非空且不相交（三者可相邻，不可重叠）。

#### D5.5 锚点

`get_anchor_rect(anchor_id)` 的语法与来源：

| `anchor_id` | 返回矩形 | 取不到时 |
|---|---|---|
| `&"top:net"` | `layout.net_block` | `Rect2()` |
| `&"top:turns"` | `layout.turns_block` | `Rect2()` |
| `&"bar:capacity"` | `layout.capacity_block` | `Rect2()` |
| `&"rail:cloy"` | `layout.cloy_block` | `Rect2()` |
| `&"grid:first_card"` | `layout.card_rects[0]` | 无卡片时 `Rect2()` |
| `&"card:<dish_id>"` | 该菜当前 `card_rects[display_index]` | 不在当前帧时 `Rect2()` |

- 只允许 `State ∈ {ACTIVE, FROZEN, ENDING}` 返回非空；`LOADING` / `HIDDEN` / `FAULTED` 返回空矩形（引导按降级路径处理）。
- `card:` 前缀只解析一次；`<dish_id>` 必须与 `HudCardView.dish_id` 完全相等（`StringName` 比较），不做模糊匹配。
- 锚点矩形随每次布局重算，**不缓存**；窗口缩放、150% 字号、超宽屏都由 `HudLayout` 重新产出。
- 返回坐标是**窗口像素**，与 `HudView` 的坐标空间一致；首局引导若在另一个 `CanvasLayer` 上绘制，必须使用同样的逆缩放（ADR-0020 复述）。

### D6 — 焦点、导航与确认面板

#### D6.1 焦点解析

- 键盘/手柄焦点与鼠标悬停是两条独立记录，来自 ADR-0005 的 `InputRouter.get_focus_owner()` / `get_hover_owner()`（ADR-0003 D6）。
- `HudFocusResolver.resolve(keyboard_id, hover_id, frame)`：有效键盘焦点优先；否则有效悬停；否则 `&""`。**没有焦点时 `ui_accept` 不提交任何动作。**
- 所有可聚焦控件的 id 使用 `&"dish:<dish_id>"` / `&"action:skip"` / `&"action:end_pickup"` / `&"confirm:cancel"` / `&"confirm:accept"`。
- 卡片与操作按钮的 `Control` 都是 `focus_mode = FOCUS_ALL`；只读块（顶栏、腻度、预览、详情行）不进入焦点链。
- **不可吃卡与不可用按钮仍可获焦**：`can_fit = false` 或 `skip_enabled = false` 只影响 `ui_accept` 的结果与文案，不改变 `focus_mode`。只有 `State ∈ {ENDING, SETTLING, HANDOFF, FAULTED}` 或 `frozen = true` 才 `FOCUS_NONE`。
- **双焦点视觉**：键盘焦点用纯描边 `focus` 样式（`draw_center = false`，≥2 px）；悬停用浅色底 `hover` 样式；两者可同时出现在不同控件上（ADR-0003 D6）。HUD 不自绘第三套「选中态」。

#### D6.2 导航与焦点恢复

`HudFocusSpec.build(frame, columns, confirm_open) -> FocusSpec`（复用 ADR-0003 的 `FocusSpec`）：

- `order`：卡片按 `display_index` 行优先排列，然后 `action:skip` 或 `action:end_pickup`。`initial_focus` 由 `HudFrame.initial_focus_id` 决定。
- **屏幕位置导航**：网格内 `up/down` 按行、`left/right` 按列；行内到行首/行尾**不跨行回绕**（保持直觉），最后一行的 `down` 指向操作按钮；操作按钮的 `up/left` 回到最后一张卡，`down/right` 回到第一张卡；整条链首尾闭环到第一张卡（GDD「循环回第一张卡」）。
- **初始焦点**：`Active` 首帧 = 当前波第一张可吃菜；若全不可吃 = 第一张卡；若没有卡 = 操作按钮；若操作按钮也不可用 = 无焦点（`INVALID`，但这是数据错误，正常不可达）。
- **重绘后焦点**：`HudFocusResolver.reconcile(prev_id, prev_index, new_frame)`：
  1. `prev_id` 仍在 `new_frame` → 保持；
  2. 否则取 `new_frame.cards[prev_index]`（原位置的下一张卡）；索引越界时取最后一张；
  3. 否则第一张可吃菜；否则第一张卡；否则操作按钮。
  焦点恢复按 id 重新 `grab_focus()`；控件消失时按上述顺序降级，**不跳回第一张**（GDD AC 37）。
- **悬停**：鼠标移动只更新 `InputRouter` 的悬停记录，HUD 订阅 `hover_owner_changed` 换 `hover` 样式；**不调用 `grab_focus()`**，键盘导航也不清除悬停高亮（GDD AC 38）。
- **点击**：HUD 不从点击合成 `ui_accept`。卡片点击是否直接提交属于 GDD Open Question 6，MVP 采用保守默认：点击只让引擎把焦点移到该卡，提交仍需 `ui_accept`。若 UX 规格最终选择「点击即提交」，只改 `HudHost` 的点击路径，不改焦点模型。

#### D6.3 确认面板

- **只在最后一波**、焦点在「结束取餐（免费）」且 `end_enabled` 时，`ui_accept` **打开面板**，不直接提交。
- 面板内容：当前波仍会失去的每道菜逐条列出（`frame.cards` 的 `name`，按 `display_index`），加一行汇总 `另有 N 道本局剩余`；`N = sum(frame.wave.hidden_counts)`。最后一波 `hidden_counts` 为空，`N = 0`，该行不显示。面板**不假装知道未揭晓的内容**。
- 面板是 HUD 内的焦点陷阱，**不是**输入上下文意义上的 `Modal`：不调用 `push_context(Modal)`，不改变 `InputRouter` 的上下文；只在 `HudFocusSpec` 内把 `order` 限制为 `[confirm:cancel, confirm:accept]` 并把四个方向闭环。
- 默认焦点「取消」；`ui_cancel` 关闭并把焦点还给 `action:end_pickup`；面板打开期间 `game_skip_wave` 一律被吞掉（不重开、不确认、不提交、视图不变）。
- 面板打开期间 `ui_pause` 仍由 `SceneFlow` 处理；暂停恢复后若 `State = ACTIVE` 且 `action_seq` 未变，面板保持打开；`run_ended` / `run_faulted` / `run_discarded` / 任何 `action_committed` 都会关闭面板。
- 面板的 `WorldScrim` 只压暗玩法画面，不压暗 HUD 数字与面板本身。

### D7 — 输入轮询与动作提交

`HudHost._process(_delta)` **只做三件事**（`process_priority = 0`，见 ADR-0005 D5）：

```gdscript
func _process(_delta: float) -> void:
    if not _is_input_eligible():
        return
    if _input.is_action_just_pressed(&"game_skip_wave"):
        _on_primary_action_pressed(true)
    if _input.is_action_just_pressed(&"ui_accept"):
        _on_accept_pressed()
    if _input.is_action_just_pressed(&"ui_cancel"):
        _on_cancel_pressed()
```

- `_is_input_eligible()`：`_view != null` 且 `State ∈ {ACTIVE}` 且 `frozen = false`；`FROZEN` / `ENDING` / `SETTLING` / `HANDOFF` / `FAULTED` / `LOADING` 一律不处理。
- `ui_accept`：
  - 面板打开 → 激活面板焦点项（确认提交 `END_PICKUP`，取消关闭）；
  - 焦点是卡片 → 若 `can_fit = false`：不提交，卡片抖动 ≤150ms，详情行写 `放不下`，发 `dish_action_rejected(&"capacity", id)`；若 `can_fit = true` 但 `capacity_remaining < effective_satiety`：不提交，调试构建记 `HUD_INVARIANT` 4，发 `dish_action_rejected(&"hud_invariant", id)`；否则 `run_flow.submit_action(EAT, id)`；
  - 焦点是 `action:skip` → 非最后一波：`skip_enabled` 时 `submit_action(SKIP_WAVE)`，否则不提交并显示原因；最后一波：打开确认面板；
  - 焦点是 `action:end_pickup` → 打开确认面板；
  - 无焦点 → 不提交。
- `game_skip_wave`：
  - 面板打开 → 吞掉，不产生任何意图；
  - 否则与非最后一波的「跳过本波」同义；最后一波等价于「结束取餐」，打开确认面板；
  - `skip_enabled = false` → 不提交、显示原因；
  - 动作名与 GDD Open Question 1（`game_skip_wave` vs `skip_dish`）保持现状：HUD 只认 `game_skip_wave`，语义映射归 `RunFlow`。
- `ui_cancel`：只在面板打开时关闭面板；否则不处理（`Run` 中的 Esc 已由 InputMap 映射为 `ui_pause`，见 ADR-0005 D3）。
- **拒绝不乐观更新**：`submit_action` 返回 `committed = false` 时，HUD 不修改任何数字，只显示原因 150ms；成功提交由随后的 `action_committed` 重建帧（GDD Edge Case）。
- **HUD 不自行扣回合、不丢菜、不改容量、不推进波次**；所有写都经 `RunFlow.submit_action`。

### D8 — 状态映射与生命周期

HUD 状态是 `run_state` 与 `frozen` 的投影，不新增玩法状态：

| `run_state` / 条件 | HUD `State` | `interactive` | 进入方式 |
|---|---|---|---|
| `Idle` / 未开局 | `HIDDEN` | false | `sync_for_state(*, Run)` 时若 `run_id = 0` |
| `Initializing` | `LOADING` | false | 挂载时首份快照未到 |
| `Active` 且 `frozen = false` | `ACTIVE` | true | `run_started` / `action_committed` / `sync_for_state(Paused, Run)` |
| `Active` 且 `frozen = true` | `FROZEN` | false | `sync_for_state(Run, Paused)` |
| `Ending` | `ENDING` | false | `run_ended` |
| `Settling` | `SETTLING` | false | `run_settled` |
| `Handoff` | `HANDOFF` | false | `sync_for_state(Run, Menu)` 之前的最后一帧；不延迟卸载 |
| `Faulted` | `FAULTED` | false | `run_faulted` |
| `Failed`（初始化失败） | `HIDDEN` | false | `sync_for_state(Run, Error)` 卸载 |

**生命周期**：

- `sync_for_state(from, to)` 是唯一的挂载/卸载入口（ADR-0004 D8 第 3 步）：
  - `to == RUN` → 挂载 `HudView`，读一次 `HudSource.capture()` 构造首帧；若 `run_state = Initializing` 则 `LOADING`；
  - `to == PAUSED` → **不卸载**，用当前快照重建 `FROZEN` 帧（`frozen = true`），焦点全部关闭；
  - `to == RUN`（从 `PAUSED`）→ 重建 `ACTIVE` 帧，恢复焦点；
  - `to ∈ {MENU, ERROR, BOOT, QUIT}` → `remove_child()` + `queue_free()`，清空 `_frame` / `_last_legal_frame` / 确认面板。
- **`frozen` 的到达方式**：`sync_for_state(Run, Paused)` / `sync_for_state(Paused, Run)` 就是冻结状态的通知；HUD 不监听 `ui_pause`，也不要求 `RunFlow` 新增 `frozen_changed` 信号。若未来出现非暂停来源的 `frozen` 变化，必须由 `AppRoot` 再调一次 `hud.sync_for_state`，不得让 HUD 轮询。
- `run_started` / `action_committed` / `run_ended` / `run_settled` / `run_faulted` / `run_discarded` 处理器只做 `_rebuild(source)`；`run_settled` 的 `RunResult` 参数**不读取**（静态检查 `result.` 在 `hud_host.gd` 中出现即失败）。
- `run_discarded` → 直接 `_unmount()`，不播结束动画、不显示收据、不保留帧。
- **`Settling` 不清屏**：HUD 保持 `ENDING` 最后一帧，直到 `Run → Menu` 的同一帧被 `ReceiptScreen` 覆盖（ADR-0003 D7）。`HANDOFF` 的淡出 ≤200ms 只在 HUD 仍被挂载时播放；`reduce_motion = true` 时为 0；**淡出绝不延迟卸载**。
- **无隐藏计时器**：HUD 不注册除 `_process` 输入轮询之外的周期逻辑；数字滚动由 `Tween` 驱动且不参与状态判定。
- **重排**：`apply_presentation(prefs)` 与视口 `size_changed` 只调用 `HudFrameBuilder.relayout(last_frame, new_ctx)`；它复用上一帧的全部值对象，只替换 `layout` / `settings`，**不读上游**。`source = RELAYOUT`。
- **焦点变化不重建帧**：`HudView.set_focus(id)` 只换高亮与详情行，帧不变；因此焦点移动不产生上游读取，也不触发 `number_roll`。

### D9 — 数字格式、文本表与颜色冗余

#### D9.1 文本表

全部玩家可见字符串来自 `HudTextTable`（`res://data/config/hud_text.tres` 或本地化变体），HUD 代码中**禁止**出现中文字面量：

```gdscript
class_name HudTextTable extends Resource
@export var empty_mark: String = "—"          # U+2014
@export var minus_sign: String = "−"          # U+2212
@export var plus_sign: String = "+"
@export var labels: Dictionary = {}           # key -> 文案模板
func text(key: StringName, args: Array = []) -> String
```

生产实现用 `tr()` 取本地化模板；测试注入固定语言的夹具，保证无头输出逐字符稳定。调试文本用 `debug_*` 前缀的独立键，不进入正式文本表。

#### D9.2 固定格式

`HudFormat` 是纯静态函数，输入值 + `Status` + `HudTextTable`，输出字符串：

| 函数 | 规则 | 样例 |
|---|---|---|
| `net(value, status)` | `UNAVAILABLE → —`；否则带符号、整数、无千分位；负号用 U+2212 | `+39`、`−46`、`+0` |
| `shortfall(value, status)` | `UNAVAILABLE → —`；`0 → 已回本`；否则 `还差 N 元` | `还差 46 元`、`已回本` |
| `turns(remaining, total, status)` | `UNAVAILABLE → —`；否则 `剩余 R / T` | `剩余 6 / 10`、`剩余 0 / 0` |
| `capacity(used, total, status)` | `UNAVAILABLE → —`；否则 `已用 U / T 格` | `已用 12 / 20 格`、`已用 0 / 0 格` |
| `cannot_fit(effective, remaining)` | `需要 E 格，剩 R 格` | `需要 6 格，剩 4 格` |
| `short_cells(short)` | `还差 S 格` | `还差 2 格` |
| `wave(index, count, status)` | `波次 I / N` | `波次 2 / 3` |
| `hidden(count)` | `?×N` | `?×2` |
| `skip_cost(turns)` | `跳过本波 · 花 1 回合` | — |
| `end_pickup()` | `结束取餐（免费）` | — |
| `skip_blocked(reason)` | `没有回合了` / `最后一回合不能跳波` | — |

- 金额最多 5 位；超过时照常完整显示并记 `HUD_FORMAT_OVERFLOW`，**不截断数字**。
- 斜杠两侧空格固定；数字列 `min-width` 按 5 位预留，位数变化不重排。
- `unit_value` / `元/格` / 百分比 / 推荐顺序**永不出现**。
- `run_seed` 只在 `ctx.debug_build = true` 时显示；release 中 `HudFrame.run_seed` 恒 0，视图不显示。

#### D9.3 颜色冗余

`HudPalette`（主题资源）定义语义角色：`ink` / `paper` / `olive` / `brick` / `muted`。每个角色必须有一条**文字或符号**第二通道：

| 语义 | 颜色 | 第二通道 |
|---|---|---|
| 净利为正 | 橄榄绿 | `+` 前缀 + 「净利」标签 |
| 净利为负 | 砖红 | `−` 前缀 + 「净利」标签 |
| 回本差额 | 墨色 | 「还差 N 元」/「已回本」整句 |
| 不可吃 | 砖红左边界 + 40% 置灰 | 「需要 X 格，剩 Y 格」原因句（100% 不透明） |
| 跳过代价 | 墨色 | 「花 1 回合」/「没有回合了」 |

`HudContrast.ratio(fg, bg)` 按 GDD 的 `hud_contrast_ratio` 实现；CI 遍历主题的每一组文本/背景（含 60% 不透明度的有效底色），断言 `≥ 4.5`。`hud_opacity_percent` 只改面板底色 alpha，不改文字 alpha；若某面板在 60% 下达不到 4.5，主题记录 `panel_floor` 并取 `max(requested, panel_floor)`（ADR-0008 已规定）。

### D10 — 音效信号与数字滚动

`HudHost` 发出三个软信号，**不播放音频**：

```gdscript
signal dish_action_rejected(reason: StringName, dish_id: StringName)
signal number_roll_started(field: StringName, from_value: int, to_value: int, duration_ms: int)
signal number_roll_finished(field: StringName, skipped: bool)
```

- `field ∈ {&"net", &"shortfall", &"turns", &"capacity"}`；`duration_ms = reduce_motion ? 0 : theme.number_roll_ms`（≤250）。
- **数字先变，动画后跟**：`_rebuild` 先把新值写入视图，再启动 `Tween`；`Tween` 只作用于显示插值，不推迟数值、不参与状态判定。`number_roll_started` 在数值已更新后发出；`number_roll_finished` 在 `Tween` 完成时发出，`skipped = true` 表示被下一次变化或暂停中断。
- 暂停发生在滚动中途：`Tween` 随 `HudHost.process_mode = INHERIT` 停止；恢复后不重播、不补播；数值已是新值。
- `reduce_motion = true`：不启动 `Tween`，直接 `started(0ms)` + `finished(skipped = false)`，数值同帧可见。
- `dish_action_rejected` 的 `reason`：`RunFlow` 的 `RejectCode` 小写名（`capacity` / `no_turns` / `stale_focus` / `frozen` / `not_active` / `not_final_wave` / `no_eatable`），HUD 本地拒绝用 `hud_capacity` / `hud_invariant` / `hud_no_focus`。
- 焦点移动、悬停、点击空白**不产生任何音频事件**（音效规则 19）。

### D11 — 不变式、降级与可测性

#### D11.1 五条运行时不变式

调试构建在每次 `HudFrame` 构造后校验；任一失败记一次 `HUD_INVARIANT`（编号 + `run_id` + `action_seq`）：

| # | 不变式 | 失败处理 |
|---|---|---|
| 1 | `capacity_used + capacity_remaining = capacity_total` | 不采纳新帧；保留 `_last_legal_frame`；`interactive = false` |
| 2 | `turns_used = turns_total − turns_remaining` | 同上 |
| 3 | `shortfall = max(0, −running_net)` | 同上 |
| 4 | `can_fit(d) = true ⇒ capacity_remaining ≥ effective_satiety(d)` | 不采纳新帧；若玩家正激活该卡，不提交并记 `hud_invariant` |
| 5 | 本帧所有 `OK` 字段引用同一个 `run_id` 与 `action_seq` | 不采纳新帧 |

正式构建：任何不变式失败都保持最后一次合法渲染，不崩游戏；`UNAVAILABLE` 字段不参与第 1–4 条（没有值可比），第 5 条只约束 `OK` 字段。`Faulted` 帧的受影响字段为 `UNAVAILABLE`，不触发上述断言。

#### D11.2 纯逻辑与视图分离

以下类不引用 `Node` / `SceneTree`，可在无头测试中实例化：

- `HudSource`（可用 stub 上游构造）、`HudInputs`
- `HudFrameBuilder`、`HudFrame` 及其子视图对象
- `HudLayout`、`HudLayoutRects`、`HudTheme`
- `HudFormat`、`HudTextTable`、`HudPalette`、`HudContrast`
- `HudFocusResolver`、`HudFocusSpec`、`HudConfirmModel`
- `HudInvariants`

`HudHost` 只做适配：挂载/卸载、事件转发、输入轮询、锚点转发、信号发出。`HudViewFactory` 允许测试注入 stub `HudView`（与 ADR-0003 的 `ScreenFactory` 同形）。

#### D11.3 静态检查

| 规则 | 扫描目标 | 失败含义 |
|---|---|---|
| 无上游写 | `src/presentation/hud/` 不出现 `spend_turn(`、`consume(`、`advance(`、`end_pickup(`、`record_eat(`、`initialize(`、`end_run(`、`commit(`、`mark_dirty(` | HUD 越权写 |
| 无第二份净利 | 不出现 `running_net +=`、`shortfall =`、`price +`、`sum(`、`reduce(` | 第三份累加器 |
| 无答案泄露 | 不出现 `unit_value`、`opt_value`、`recommend`、`best_dish` | 推荐/最优解 |
| 无自动排序 | `cards` 构建路径不出现 `.sort(` / `.sort_custom(` | 默认排序 |
| 无 HUD 轮询上游 | `_process` 只调用 `_input.is_action_just_pressed` 与 `_view` 方法 | 每帧读服务 |
| 无 HUD 切上下文 | 不出现 `push_context(`、`pop_context(`、`request_context(` | 确认面板被做成 Modal |
| 无 HUD 监听暂停 | 不出现 `is_action_just_pressed(&"ui_pause")` | 与 SceneFlow 抢暂停 |
| 无 HUD 读 `RunResult` | `hud_host.gd` 不出现 `result.` | 收据数据泄漏进 HUD |
| 无 HUD 直接查数据 | 不出现 `DataConfig.` / `get_dish_icon(`（除 `HudSource`） | 绕开投影层 |
| 文本集中 | `src/presentation/hud/` 不出现中文/英文 UI 字面量 | 文案散落 |

## Architecture Diagram

```
                     RunFlow signals / SceneFlow.state_changed
                                      │
                                      ▼
┌────────────────────────── HudHost (CanvasLayer, layer 10) ──────────────────────────┐
│  sync_for_state(from,to)   on_run_started / action_committed / run_ended             │
│  on_run_settled / run_faulted / run_discarded   _process(输入轮询)   apply_presentation│
│                                                                                      │
│  ┌────────── HudSource.capture() ──────────┐   ┌──── HudFrameBuilder ────┐          │
│  │ RunFlow.snapshot()  (一次)              │   │ build(HudInputs)        │          │
│  │ MenuWaves.visible/preview/hidden        │──▶│ relayout(prev, ctx)     │──┐       │
│  │ DishValues.get_dish / price / satiety   │   │ HudLayout / HudFormat   │  │       │
│  │ Cloying.preview / group_counts          │   │ HudInvariants           │  │       │
│  │ Capacity.can_fit / remaining            │   └─────────────────────────┘  │       │
│  │ RunFlow.action_availability()           │                                 ▼       │
│  └─────────────────────────────────────────┘                         HudFrame (immutable)
│                                                                              │       │
│  ┌───────────────────────────── HudView (Control) ◀──────────────────────────┘       │
│  │ TopBar · CardGrid · RightRail · DetailRow · ConfirmPanel · WorldScrim           │
│  │ FocusSpec.apply(order, neighbours) → grab_focus(initial)                        │
│  └──────────────────────────────────────────────────────────────────────────────────┘
│  get_anchor_rect(anchor_id) · get_protected_rects() · focused_dish_id()             │
└──────────────────────────────────────────────────────────────────────────────────────┘
        │ submit_action(EAT / SKIP_WAVE / END_PICKUP)          │ dish_action_rejected
        ▼                                                       │ number_roll_started/finished
      RunFlow                                              Audio Feedback (订阅)
```

## Key Interfaces

```gdscript
# res://src/presentation/hud/

class_name HudHost extends CanvasLayer

signal dish_action_rejected(reason: StringName, dish_id: StringName)
signal number_roll_started(field: StringName, from_value: int, to_value: int, duration_ms: int)
signal number_roll_finished(field: StringName, skipped: bool)

func attach(source: HudSource, input: InputRouter, run_flow: RunFlow,
            text: HudTextTable, theme: HudTheme, factory: HudViewFactory) -> void

func sync_for_state(from: int, to: int) -> void
func apply_presentation(prefs: PresentationPrefs) -> void
func on_run_started(run_id: int, station_id: StringName, seed: int) -> void
func on_action_committed(action: int, payload: Dictionary) -> void
func on_run_ended(run_id: int, end_reason: int) -> void
func on_run_settled(run_id: int, result: RunResult) -> void      # result 不读取
func on_run_faulted(run_id: int, code: StringName, detail: Dictionary) -> void
func on_run_discarded(run_id: int) -> void

func get_anchor_rect(anchor_id: StringName) -> Rect2
func get_protected_rects() -> Array[Rect2]                        # [turns, capacity, net]
func focused_dish_id() -> StringName
func current_frame() -> HudFrame
func is_degraded() -> bool


class HudSource extends RefCounted
func _init(run_flow: RunFlow, menu_waves: MenuWaves, dishes: DishValues,
           cloying: CloyingAndTaste, capacity: StomachCapacity,
           data: DataConfig, debug_build: bool) -> void
func capture(caller: StringName) -> HudInputs


class HudFrameBuilder extends RefCounted
func build(inputs: HudInputs, ctx: HudBuildContext, source: int) -> HudBuildResult
func relayout(prev: HudFrame, ctx: HudBuildContext) -> HudFrame

class HudBuildResult extends RefCounted
var frame: HudFrame
var legal: bool
var violations: Array[HudViolation]


class HudLayout extends RefCounted
static func compute(ctx: HudBuildContext, theme: HudTheme,
                    input: HudLayoutInput) -> HudLayoutRects

class HudFormat extends RefCounted
static func net(value: int, status: int, text: HudTextTable) -> String
static func shortfall(value: int, status: int, text: HudTextTable) -> String
static func turns(remaining: int, total: int, status: int, text: HudTextTable) -> String
static func capacity(used: int, total: int, status: int, text: HudTextTable) -> String
static func cannot_fit(effective: int, remaining: int, text: HudTextTable) -> String
static func short_cells(short: int, text: HudTextTable) -> String


class HudFocusResolver extends RefCounted
static func resolve(keyboard_id: StringName, hover_id: StringName,
                    frame: HudFrame) -> StringName
static func reconcile(prev_id: StringName, prev_index: int,
                      frame: HudFrame) -> StringName

class HudFocusSpec extends RefCounted
static func build(frame: HudFrame, columns: int, confirm_open: bool) -> FocusSpec

class HudConfirmModel extends RefCounted
func open_from(frame: HudFrame) -> void
func close() -> void
func move(direction: int) -> void
func activate() -> int             # CONFIRM / CANCEL


class HudView extends Control
func present(frame: HudFrame, focus_id: StringName) -> void
func set_focus(focus_id: StringName) -> void
func set_confirm(model: HudConfirmModel) -> void
func set_interactive(enabled: bool) -> void
func reflow(rects: HudLayoutRects, settings: HudSettingsView) -> void
```

`RunFlow.action_availability()` 的**只读**形状（生产者归 ADR-0013；本 ADR 只定义消费面）：

```gdscript
class RunActionAvailability extends RefCounted
var eat_enabled: bool
var skip_visible: bool
var skip_enabled: bool
var skip_reason: int          # NONE / NO_TURNS_ZERO / NO_TURNS_ONE / NO_EATABLE
var end_visible: bool
var end_enabled: bool
var end_reason: int           # NONE / NO_EATABLE / NOT_FINAL_WAVE
```

## Implementation Guidelines

1. **必须**让 `HudHost` 是唯一持有 `RunFlow` 引用的 HUD 类；**禁止** `HudView` / `HudFrame` / `HudLayout` / `HudFormat` 持有任何上游服务或读取其字段。
2. **必须**在每次事件内只调用一次 `HudSource.capture()`；**禁止**在 `HudFrameBuilder` 或 `HudView` 中再次读取上游。
3. **必须**在构造后不修改 `HudFrame`；**禁止**为 `HudFrame` 添加 setter、`refresh()` 或可变数组暴露。
4. **必须**把 `cards` 保持为 `visible_dishes()` 的原始顺序；**禁止**默认排序、可吃性过滤、推荐或 `unit_value`。
5. **必须**在 `HudLayout` 中按窗口像素计算，并用 `CanvasLayer` 逆缩放抵消根内容缩放；**禁止**把 `s` 与引擎内容缩放相乘。
6. **必须**把全部布局常量放在 `HudTheme`；**禁止**在代码中写字面量 72 / 280 / 200 / 16 / 176 / 44 等。
7. **必须**让不可吃卡与不可用按钮保持 `focus_mode = FOCUS_ALL`；**禁止**用 `disabled` 表达数据状态。
8. **必须**让确认面板只经 `HudFocusSpec` 做焦点陷阱；**禁止**调用 `push_context(Modal)` / `pop_context()` / `request_context()`。
9. **必须**在面板打开期间吞掉 `game_skip_wave`；**禁止**把它解释为确认或重新打开。
10. **必须**在动作提交后同一帧更新数字，再启动 `Tween`；**禁止**用动画时长推迟数值或先播动画后改数。
11. **必须**在 `submit_action` 返回 `committed = false` 时保持旧值；**禁止**乐观更新。
12. **必须**在 `State ∈ {ENDING, SETTLING, HANDOFF, FAULTED}` 或 `frozen = true` 时关闭焦点与鼠标；**禁止**在这些状态下提交任何动作。
13. **必须**让 `get_protected_rects()` 恒为三个矩形且顺序固定；**禁止**返回少于三个或在 `ACTIVE` 下返回空。
14. **必须**让全部玩家可见文本经 `HudTextTable`；**禁止**在 HUD 代码中写中文字面量。
15. **必须**在整次采集或不变式失败时保留 `_last_legal_frame`；**禁止**用部分数据构造新帧。
16. **必须**在 `run_settled` 中忽略 `RunResult`；**禁止**在 HUD 中读取 `rows` / `net_profit` / `unit_value`。

## Alternatives Considered

### Alternative 1: 每个 HUD 元素各自读取上游

- **Description**：净利组件直接调 `RunFlow.snapshot()`，容量条直接调 `StomachCapacity`，卡片直接调 `MenuWaves`。
- **Pros**：组件独立、无需 `HudFrame`。
- **Cons**：一次事件内多次读取会产生跨快照混帧；重绘顺序影响结果；无法满足 GDD 规则 3 / 不变式 5。
- **Rejection Reason**：直接制造双真相，且不可无头断言。

### Alternative 2: HUD 自己维护一份净利累加器，以保证「显示一致」

- **Description**：HUD 监听 `action_committed`，自己把价格加到本地 `running_net`。
- **Pros**：不依赖 `RunFlow` 的字段。
- **Cons**：免单、错误局、结算复核三条边界必然漂移；GDD 规则 6 明令禁止；`RunLog` 与 HUD 会出现两个数字。
- **Rejection Reason**：第三份真相，且与 ADR-0007 D1 的单一所有者冲突。

### Alternative 3: 确认面板用 `push_context(Modal)` 做焦点陷阱

- **Description**：打开面板时压入 `Modal`，关闭时弹出。
- **Pros**：复用 ADR-0003 的模态焦点陷阱与遮罩。
- **Cons**：会改变 `InputRouter` 的上下文，导致 `Run` 的 `ui_accept` / `game_skip_wave` 投递路径变化；`ui_pause` 仍归 `SceneFlow`，两个栈的深度断言会打架；GDD 规则 16 明确禁止。
- **Rejection Reason**：确认面板是局内视图，不是界面流程的模态。

### Alternative 4: 用设计单位布局，让 `canvas_items` 完成全部缩放

- **Description**：HUD 以 1280×720 设计单位布局，`s` 固定 1.0，依赖引擎内容缩放。
- **Pros**：不碰 `CanvasLayer.transform`，实现最简。
- **Cons**：2160p 会被引擎放大 3×，违反 GDD `hud_ui_scale` 的 2.0 上限；超宽屏的额外宽度也无法只给主区。
- **Rejection Reason**：只在逆缩放不可行时作为降级路径（D5.1）。

### Alternative 5: 焦点由 HUD 纯模型驱动，不依赖引擎 `Control` 焦点

- **Description**：所有卡片自绘焦点，HUD 自己处理方向键并调用 `submit_action`。
- **Pros**：完全可控、无头可测。
- **Cons**：放弃引擎的 GUI 导航、读屏焦点与 `NavRepeater` 合成投递；双焦点（4.6）与手柄重复输入要重写一遍；与 ADR-0003 D5 / ADR-0005 D8 冲突。
- **Rejection Reason**：引擎焦点是既有契约，HUD 只负责声明 `FocusSpec` 与解析 id。

## Consequences

### Positive

- 「一帧一快照」成为类型事实：`HudFrame` 的 `run_id` / `action_seq` 由唯一一次 `capture()` 写入，跨帧混用无法通过构造。
- 上游读取点收敛到 `HudSource` 一处，静态检查与无头测试都能证明 HUD 不求和、不推荐、不轮询。
- 布局公式与锚点矩形是纯函数，720p / 1080p / 超宽 / 150% 字号可以在 CI 里逐像素断言，不需要截图才能发现 1 px 漂移。
- 不可吃卡、不可用按钮、确认面板三者共用「数据状态可获焦、流程状态才禁用」的规则，输入与无障碍行为一致。
- 首局引导只依赖矩形接口，HUD 改布局不会连带改引导。

### Negative

- `HudFrame` 的子对象较多，样板量高于「组件各自读服务」；这是为可测性与单快照付出的显式成本。
- `CanvasLayer` 逆缩放依赖 4.7 实机验证；若失败需要走设计单位降级，2160p 的 2.0 上限会失效。
- 焦点恢复、确认面板、数字滚动都需要各自的纯模型与测试，实现量集中在 Presentation 层。
- 卡片字段同屏要求布局在 150% 字号下降到 2 列，信息密度低于手游式悬浮卡。

### Neutral

- HUD 不实现玩家主动排序；MVP 的卡片顺序唯一。
- `game_skip_wave` 的动作名与 GDD Open Question 1 保持未决；HUD 只认这一个动作名。
- 数字滚动是纯呈现，`number_roll_*` 只为音效服务，不参与任何状态。

## Risks

- **`CanvasLayer` 逆缩放破坏输入命中或焦点几何。** 缓解：Verification ①/②；失败时走 D5.1 降级，并在验收中标注 2160p 上限失效。
- **`RunFlow.action_availability()` 未实现，HUD 只能自己拼 `turns_remaining ≥ 2`。** 缓解：D3 把它列为必需接口；在 ADR-0013 补齐前，HUD 不得复制该阈值（宁愿按钮不可用也不产生第二份动作语义）。
- **`frozen` 变化没有独立信号。** 缓解：D8 把 `sync_for_state(Run↔Paused)` 定义为冻结通知；若未来出现非暂停来源，由 `AppRoot` 显式再调一次，不引入 HUD 轮询。
- **数值动画让玩家误以为数字还没变。** 缓解：D10 先写值再启动 `Tween`；AC 41 以「提交帧 == 数字变化帧」为门禁。
- **150% 字号下卡片 2 列仍放不下四个字段。** 缓解：`HudLayout` 用主题三档卡高；详情行承载完整字段；若仍溢出，UX 规格必须改布局而不是缩字号（GDD AC 30）。
- **确认面板的 `WorldScrim` 压暗到 HUD 数字。** 缓解：`WorldScrim` 位于 HUD 内容之下；数字与面板 alpha 恒 1.0；对比度测试覆盖 60% 面板底色。
- **`hidden_counts()` 在最后一波为空，确认面板的「另有 N 道」恒为 0。** 缓解：N = 0 时不显示该行；面板只列当前波可见菜，不假装知道未揭晓内容（GDD Sync Check 第 8 条）。
- **静态检查漏掉通过局部变量间接求和。** 缓解：除 grep 外，增加「帧字段来源」测试：逐字段断言等于 `HudInputs` 的对应值；`running_net` 与 `shortfall` 必须与 `snapshot()` 完全相等。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `in-game-hud.md` | `TR-in-game-hud-001`：只读投影；每次事件构造一个不可变 `HudFrame`；帧间不轮询 | D1/D2/D3：`HudSource.capture()` 单次采集、`HudFrame` 构造后不可变、`run_id`/`action_seq` 同源；`_process` 只轮询输入；D8 的 `relayout` 不读上游 |
| `in-game-hud.md` | `TR-in-game-hud-003`：卡片由菜品/腻度/容量/波次提供方合成；不排序、不推荐、不出现 `unit_value` | D4：四源字段同屏、`display_index` 固定为 `visible_dishes` 顺序、`block_reason` / `short_cells` 仅显示；静态检查禁止 `unit_value` / `sort` / 推荐 |
| `in-game-hud.md` | `TR-in-game-hud-005`：固定数字格式；颜色永远有文字/符号作为第二通道 | D9：`HudFormat` 逐字符固定 `+39` / `−46` / `还差 46 元` / `已回本` / `—` / `剩余 6 / 10` / `已用 12 / 20 格`；`HudPalette` 每个语义色都有文字或符号；对比度 ≥ 4.5:1 |
| `in-game-hud.md` | `TR-in-game-hud-006`：`get_anchor_rect(anchor_id)` 与三个受保护矩形供首局引导 | D5.5 / D10：六个锚点 id、空矩形降级、`get_protected_rects()` 固定三元组（回合/容量/净利）、窗口像素坐标与逆缩放一致 |

> `TR-in-game-hud-002` 由 ADR-0003 覆盖（1280×720 `canvas_items`/expand 与分层）；`TR-in-game-hud-004` 由 ADR-0005 覆盖（焦点解析与 `REJECTED(reason)` 投递）。本 ADR 只消费它们的结论，不重复认领。

## GDD Sync Check

| # | GDD / ADR 原文 | 本 ADR 的落地 | 需要谁对齐 |
|---|---|---|---|
| 1 | `in-game-hud.md` 规则 7：受影响的数字显示 `—`，其余数字保持最后一次合法值并标为只读 | 单字段失败 → 该字段 `UNAVAILABLE` 显示 `—`，其余字段仍取同一份快照的当前值；整次采集/不变式失败 → 保留 `_last_legal_frame` 且 `interactive = false`。不变量 5 只约束 `OK` 字段 | 本 ADR 解释；若产品要求「单字段失败也冻结全部数字」，改 `degraded → interactive=false` |
| 2 | `TR-run-flow-007` 只列 `run_state` / `frozen` / `running_net` / `shortfall` / `action_seq`；`menu-waves.md` 只给 `visible_dishes` / `preview_dishes` / `hidden_counts`；`≥2` 回合的跳波门控在 ADR-0013 | HUD 需要 `RunFlow.action_availability()` 统一发布 `skip_visible` / `skip_enabled` / `skip_reason` / `end_*`，避免 HUD 复制 `turns_remaining ≥ 2` 或 `is_final_wave` 判定 | ADR-0007 / ADR-0013 必须补只读接口；`requirements-traceability.md` 的 `TR-run-flow-007` 说明可更新 |
| 3 | ADR-0005 D9 只把 `ui_accept` / `game_skip_wave` 的读者限定为 `HudHost` | 确认面板关闭需要 `ui_cancel`；`Run` 中 Esc 被映射为 `ui_pause`，手柄 B 仍产生 `ui_cancel`。HUD 必须轮询 `ui_cancel` | ADR-0005 D9 表增加 `ui_cancel` 的 `HudHost` 读者 |
| 4 | ADR-0008 只写「组合根调用 `UIRoot.apply_presentation(prefs)`」 | HUD 也必须收到 `PresentationPrefs`；本 ADR 定义 `HudHost.apply_presentation(prefs)`，由 `AppRoot` 在 `presentation_changed` 与挂载时调用 | ADR-0004 / ADR-0008 的接线清单补一行；不新增设置状态 |
| 5 | ADR-0004 D8 的 `hud.sync_for_state(from,to)` 只描述挂载/卸载 | 本 ADR 把 `Run ↔ Paused` 的 `sync_for_state` 定义为 `frozen` 变化通知；HUD 不监听 `ui_pause`，不要求新增 `frozen_changed` 信号 | ADR-0004 的 D9/D10 措辞对齐 |
| 6 | `in-game-hud.md` 规则 9 允许玩家按价格/饱腹排序；`TR-in-game-hud-003` 写「不排序」 | MVP 不实现玩家排序；默认顺序恒为 `visible_dishes`。若未来加入，只能是视图置换，`display_index` 仍记录规范位置 | 本 ADR 解释；UX 规格若要求排序，需先修订 TR 措辞 |
| 7 | `in-game-hud.md` Edge Case：「最后一波确认面板以『另有 N 道本局剩余』汇总不可见部分」 | `N = sum(hidden_counts)`；最后一波 `hidden_counts` 为空，N = 0，该行不显示；面板不假装知道未揭晓内容 | 游戏设计确认「最后一波 N 恒为 0」是否可接受 |
| 8 | `in-game-hud.md` Open Question 1：`game_skip_wave` 与 `skip_dish` 命名未决 | HUD 只认 `game_skip_wave`，映射到 `RunFlow` 的 `SKIP_WAVE` / `END_PICKUP`；文案区分「跳过本波」/「结束取餐（免费）」 | game-designer（输入 + 菜单波次）在写 `design/ux/hud.md` 前定名 |
| 9 | `in-game-hud.md` Formulas 的 `hud_ui_scale` 假定 HUD 独立于 `canvas_items` | 本 ADR 用 `CanvasLayer.transform` 的逆内容缩放实现；4.7 实机未验证，失败时走设计单位降级 | godot-specialist 验证；ADR-0003 Risk 4 同步 |
| 10 | `in-game-hud.md` 规则 3 的事件表包含 `frozen` 变化，但 ADR-0004 的信号表没有 `frozen_changed` | 冻结经 `sync_for_state(Run↔Paused)` 到达；`run_started` / `action_committed` 等信号处理器不负责冻结 | ADR-0004 D9/D10 措辞 |
| 11 | `in-game-hud.md` AC 19：`can_fit = true` 但剩余容量不足时「信任 `can_fit`，不提交动作」 | 调试构建：不提交、记 `HUD_INVARIANT` 4、发 `hud_invariant`；正式构建：不采纳矛盾帧，保留最后合法渲染。显示与提交都不基于矛盾数据 | 本 ADR 解释；与规则 11 的第一句「信任 `can_fit`」在字面上有张力，但 AC 19 与本 ADR 一致 |
| 12 | `in-game-hud.md` 规则 26：`turns_remaining = 1` 时「跳过本波」不可用并写「最后一回合不能跳波」 | 可用性来自 `RunFlow.action_availability().skip_reason`，HUD 只格式化；按钮仍可获焦并读出原因 | 依赖 GDD Sync Check 第 2 条 |

## Performance Implications

- **CPU**：空闲时 `_process` 只有三次 `is_action_just_pressed` 查询与一次状态判断，≤ 0.5 ms/帧 @1080p（GDD AC 40）。一次事件构造 = O(可见菜数 + 腻度组数 + 预览数)，MVP 上限 4 + 4 + 2，远低于 1 ms。
- **Memory**：`HudFrame` 持有一次局的卡片数组与详情对象，量级 < 16 KiB；`_last_legal_frame` 只保留一份；卸载时全部释放。
- **Load Time**：`HudTheme` / `HudTextTable` / 卡片场景预加载一次；HUD 挂载无磁盘 I/O。
- **Rendering**：卡片与条是 `Control` + `StyleBox`，无 VFX / 粒子 / 屏幕震动；`WorldScrim` 是一个全屏 `ColorRect`，只在确认面板打开时启用。
- **Network**：不适用（MVP 单机）。

## Migration Plan

仓库没有 HUD 代码，本 ADR 是绿地决策。实施顺序：

1. 落 `HudTheme` / `HudTextTable` / `HudPalette` 资源与 `HudFormat` / `HudContrast` 纯函数，跑 AC 17、28、29、32。
2. 落 `HudLayout` / `HudLayoutRects` 与全部公式，跑 AC 9–16、30；用 1280×720 / 1920×1080 / 2560×1080 / 3840×2160 / 150% 五组夹具。
3. 落 `HudSource` / `HudInputs` / `HudFrame` / `HudFrameBuilder`，用 stub 上游跑 AC 1–8、20、21、40、41。
4. 落 `HudFocusResolver` / `HudFocusSpec` / `HudConfirmModel`，跑 AC 22–27、34–39。
5. 落 `HudView` 场景与 `HudHost` 适配器，接 `AppRoot` 的 `sync_for_state` 与五个 `run_*` 信号；跑 AC 34–36、42 的截图证据。
6. 接 `RunFlow.action_availability()`（ADR-0013 补齐后）与 `PresentationPrefs`（ADR-0004/0008 接线后）。

**Rollback plan**：本 ADR 只新增 `src/presentation/hud/` 与三个资源文件；回滚即删除该目录与资源，并移除 `AppRoot` 对 `HudHost` 的构造与信号连接。没有既有代码依赖此结构。

## Validation Criteria

| # | 验证 | 证据 |
|---|---|---|
| 1 | `HudSource.capture()` 在一次构建中恰好调用 1 次；空闲 60 帧调用 0 次 | 无头 spy 计数器 |
| 2 | `HudFrame` 构造后字段哈希不变；对返回的 `cards` 副本写入不影响原帧 | 无头单元测试 |
| 3 | 逐字段来源：`net.running_net` / `net.shortfall` 与 `snapshot()` 完全相等；`capacity.*` / `turns.*` 与快照完全相等；卡片 `price` / `effective_satiety` / `can_fit` 与上游一致 | 无头夹具 + 表驱动 |
| 4 | HUD 代码中不存在 `running_net +=`、`shortfall =`、`price +`、`sum(`、`unit_value`、`sort(`、`push_context(`、`ui_pause` | grep 门禁 |
| 5 | `cards[i].display_index == i`，顺序等于 `visible_dishes()`；无排序/过滤 | 无头单元测试 |
| 6 | 五条不变式：合法夹具全过；人为注入每条错误时恰好记录一次对应 `HUD_INVARIANT`，且不采纳新帧 | 故障注入测试 |
| 7 | 格式：`39 → +39`、`−46 → −46`（U+2212）、`46 → 还差 46 元`、`0 → 已回本`、`6/10 → 剩余 6 / 10`、`12/20 → 已用 12 / 20 格`、`0/0 → 已用 0 / 0 格`、缺失 → `—`（U+2014） | 表驱动单元测试 |
| 8 | 布局：720p 安全边距 36；1080p 54；2160p 96；2560×1080 `s = 1.5`；1280×720 τ=1 → 4 列 / 216 px；τ=1.5 → 2 列；`turns_total=10` → 10 槽 / 24 px / 276 px；`capacity 20/12` → 20 格 / 14 px / 0.6；`Active` 面积 26.5% | 无头单元测试 + 逐像素 `Rect2` 断言 |
| 9 | 中央 40%：`Active` 下只有 `card_rects` / `confirm_panel` 与之相交；顶栏、右栏、详情行不相交 | 无头几何测试 |
| 10 | 1208×72 顶栏 + 280×560 右栏 = 26.5% ≤ 28%；超宽屏主区宽度 ≤ `1600 × s` | 无头单元测试 |
| 11 | 焦点：首帧落在第一张可吃菜；全不可吃落第一张卡；吃掉当前卡后落原位置下一张；波空落新波第一张可吃菜；不可吃卡仍可获焦；`Ending`/`Frozen` 下无控件可获焦 | 无头焦点夹具 + 手工验收 |
| 12 | 确认面板：默认焦点「取消」；`ui_cancel` 关闭并还原焦点；面板打开时 `game_skip_wave` 不产生 `submit_action`；等待 60 s 不自动关闭 | 无头集成 + 手工验收 |
| 13 | 同帧：`action_committed` 的帧号与数字变化帧号相同；`Tween` 不参与；`reduce_motion` 下无补间 | 帧号日志 + 截图 |
| 14 | 锚点：六个 id 返回非空（`card:<id>` 对当前卡）；未知 id / 已吃卡 / `LOADING` → 空矩形；三个受保护矩形在 150% 与超宽屏下非空且互不重叠 | 无头单元测试 |
| 15 | 对比度：主题每组文本/背景 ≥ 4.5:1（含 60% 面板底色）；灰度截图中净利、差额、可吃性、跳过代价仍可读 | 计算测试 + 截图 |
| 16 | 150% 字号：无溢出、无重叠、卡片 2 列、信息块不消失；顶栏 ≤ 96 参考像素 | 截图 + 几何断言 |
| 17 | 逆缩放：1080p 下 HUD 元素按窗口像素 1:1 绘制，焦点邻居与命中测试正确；2160p 下 `s = 2.0` 而非 3.0 | 4.7 实机截图 + 手工验收 |
| 18 | 720p `Active` / 1080p `Active` / 150% / `Frozen` / `Ending` / 确认面板 / 不可吃卡 / 错误态 / 深色浅色对比度样例 | `production/qa/evidence/` 截图 |
| 19 | `run_settled` 处理器不读取 `RunResult`；`run_discarded` 直接卸载、无动画 | grep + 无头集成 |
| 20 | `HudHost` 不注册除输入轮询外的周期逻辑；`relayout` 不调用 `HudSource.capture()` | spy + 代码审查 |

## Related

- `docs/architecture/architecture.md` — Required ADRs 第 15 项、API Boundaries（In-Game HUD）、Data Flow 第 3 节
- `design/gdd/in-game-hud.md` — 22 条 Core Rules、9 个 Formulas、Edge Cases、AC 1–42、Open Questions 1–9
- `design/gdd/run-flow.md` — `snapshot()` 字段、`run_*` 信号、`focused_dish_id` 提交契约
- `design/gdd/menu-waves.md` — `visible_dishes` / `preview_dishes` / `hidden_counts`、跳过与结束取餐语义
- `design/gdd/dish-values.md` — `price` / `satiety`；`unit_value` 只在收据
- `design/gdd/cloying-and-taste.md` — `preview` / `group_counts` / 解腻标记
- `design/gdd/stomach-capacity.md` — `can_fit` / `capacity_remaining` / 容量条
- `design/gdd/time-and-turns.md` — 回合读数、无倒计时、同帧更新
- `design/gdd/settlement-and-entry-fee.md` — `running_net(k)` / `shortfall(k)` 的口径
- `design/gdd/first-run-onboarding.md` — 六个锚点 id、三个受保护矩形
- `design/gdd/input.md` — 双焦点、上下文、导航重复、禁用控件
- `design/gdd/audio-feedback.md` — `dish_action_rejected` / `number_roll_*` 的订阅
- ADR-0003（场景与屏幕架构）— `CanvasLayer` 分层、`FocusSpec`、双焦点
- ADR-0005（输入上下文栈语义）— `Run` 轮询与消费位
- ADR-0007（局生命周期与结算事务）— `RunFlow.snapshot()` 的唯一发布面
- ADR-0008（设置与本地化）— `PresentationPrefs` / `text_scale_percent` / `hud_opacity_percent` / `reduce_motion`
- ADR-0010（菜品数值契约）— `get_dish` / `price` / `satiety`，不读 `unit_value`
- ADR-0012（胃容量账本）— `can_fit` / `capacity_remaining`
- ADR-0013（动作管线与结束判定）— `submit_action` / `RejectCode` / `action_availability`（待补）
- ADR-0014（腻度与口味状态机）— `preview` / `group_counts`
- ADR-0018（纪录、里程碑与生涯完成）— 同批 Required ADR，无直接接口
- ADR-0020（首局引导 beat 机、阻断输入与跳过）— 消费本 ADR 的锚点与受保护矩形
- `docs/engine-reference/godot/modules/ui.md`、`modules/input.md` — 4.6 双焦点、4.7 拉伸与 `CanvasLayer` 行为
