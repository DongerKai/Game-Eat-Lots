# ADR-0003: 场景与屏幕架构（Scene & Screen Architecture）

## Status

Accepted

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

user + Claude（architecture-decision 流程，autonomous 模式）；`godot-specialist` 引擎校验；`technical-director` 战略评审按 lean 模式跳过（ADR 非 PHASE-GATE）。

## Summary

《回本》的全部界面都是 Control，在 `Menu` / `Run` / `Paused` 之间切换时必须同帧完成，且暂停不能卸载 HUD、收据不能晚一帧出现。本 ADR 决定：**一个常驻的 `UIRoot` 控制器 + 每个屏幕一个独立 `PackedScene`**，由控制器持有屏幕栈（≤2）与模态栈（≤2）、按声明式焦点规格设置 `Control.focus_neighbor_*`，并把淡入淡出实现为不参与状态机的独立覆盖层。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（最新补丁 4.7.2；本机编辑器版本 NOT DETERMINED） |
| **Domain** | UI |
| **Layer** | Foundation |
| **Knowledge Risk** | HIGH — 4.5/4.6/4.7 三代均在 LLM 训练截止之后；本 ADR 依赖 4.6 的双焦点分离与 4.7 的拉伸默认值 |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`docs/engine-reference/godot/modules/ui.md`、`docs/engine-reference/godot/modules/input.md`、`docs/engine-reference/godot/breaking-changes.md`、`docs/engine-reference/godot/deprecated-apis.md` |
| **Post-Cutoff APIs Used** | 4.6 双焦点分离（鼠标悬停焦点与键盘/手柄焦点各自独立）；4.7 新项目默认 `display/window/stretch/mode = canvas_items`、`aspect = expand`；4.7 `InputEvent.DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD`（经输入模块间接使用） |
| **Verification Required** | ① 在 4.7.x 实机上确认 `Viewport.gui_get_focus_owner()` 与 `Control.focus_neighbor_*` 行为未变；② 确认在 `_input` 中 `set_input_as_handled()` 能阻止聚焦按钮对 `ui_accept` 的自动激活；③ 确认 `mouse_entered` / `mouse_exited` 在模态遮罩存在时仍按控件边界触发；④ 确认 `canvas_items` + `expand` 下 `CanvasLayer` 的缩放行为（HUD 独立缩放是否必须走 `SubViewport` 或反向缩放）；⑤ 确认 `remove_child()` + `queue_free()` 在同一帧内不会残留一帧旧画面；⑥ 确认显式 `focus_neighbor_*` 指向不可聚焦控件时引擎是否回落到几何解析（`find_next_valid_focus` 未在参考库中覆盖）；⑦ 确认 4.6 下鼠标悬停是否满足 `has_focus()`（决定 `focus` 样式读哪一路）；⑧ 确认 `SceneTree.paused`（若采用）下 `PROCESS_MODE_ALWAYS` 的 UI 节点仍接收 `_input` 与 `Tween` 推进 |

> **Knowledge Risk 为 HIGH。** 升级引擎版本时本 ADR 必须重新验证；若 4.6 双焦点或 4.7 拉伸语义改变，标记为 Superseded 并另写。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0005（输入上下文栈语义）— 模态深度 2 依赖“栈顶已是 `Modal` 时再次 `push_context(Modal)` 仍增加栈深”这一裁定（`ui-flow.md` Open Question 14）；同时依赖输入模块在 `ui_accept` 上不双发 |
| **Enables** | ADR-0007（结算事务边界）— 本 ADR 固定收据的挂载点与同帧钩子，事务顺序在其之上编排 |
| **Blocks** | Epic: 界面流程（#16）、局内 HUD（#15）；`PauseOverlay`、`ReceiptScreen` 与焦点验收的任何 story |
| **Ordering Note** | ADR-0004 决定 `UIRoot` 由谁构造（autoload 还是组合根）；本 ADR 不依赖该结论，只要求 `UIRoot` 全局恰好一个实例并通过构造注入依赖 |

## Context

### Problem Statement

界面流程 GDD 把行为定义得很硬：任意时刻恰好一个顶层状态、状态切换同帧生效、屏幕栈 ≤2、模态栈 ≤2、每个屏幕有确定初始焦点与焦点恢复、暂停不卸载 HUD、收据在 `RunResult` 交付的同一帧成为根屏。这些约束没有一条能靠“每个屏幕自己 `change_scene_to_file`”或“一个大场景里 12 个屏幕切换可见性”实现。架构文档把这个问题登记为 **QQ-05**（一个根控制器 vs 每屏一个场景、手工 vs 自动焦点邻居），必须在写第一个界面 story 之前裁定。

### Current State

仓库中没有任何 `.gd` 界面代码，`project.godot` 只有引擎与渲染器设置，`run/main_scene` 未设置（注释明确写明“no scene exists yet”）。`docs/architecture/architecture.md` 描述了 `SceneFlow` 与 `InputRouter` 的接口边界，但屏幕如何实例化、焦点如何设置、暂停覆盖层如何在不卸载 HUD 的情况下存在，全部未定。

### Constraints

- **同帧生效**：`scene-flow.md` 规则 12 与 Formulas 要求 `state_latency_frames = 0`；`ui-flow.md` 规则 9 要求 `receipt_open_latency_frames = 0`。任何 `await`、`call_deferred`、`Tween.tween_await()`、`ResourceLoader.load_threaded_request()` 参与状态路径都会直接违反这两条。
- **HUD 不是界面流程的屏幕**：`ui-flow.md` Core Rule 2 与 `in-game-hud.md` 规则 21 明确 HUD 是同级系统，不进入界面流程的屏幕栈。
- **焦点可判定**：焦点顺序按屏幕位置（上→下、左→右），不按节点树顺序（`input.md` UI Requirements）；初始焦点与恢复是公式（`ui-flow.md` Formulas）。
- **无每帧轮询**：`ui-flow.md` 规则 28 要求界面状态是输入与数据的纯函数，除保存指示与收据滚动外没有隐藏计时器。
- **参考分辨率 1280×720**：`project.yaml` 的 `run` 命令以该窗口运行；`in-game-hud.md` 要求 `canvas_items` 拉伸、基准 1280×720、`expand` 宽高比。
- **无头可测**：`project.yaml` 的 `test` 命令是 `godot --headless ... GdUnitCmdTool.gd`；界面栈、返回决策、焦点选择必须能在无渲染的情况下断言。
- **4.6 双焦点**：鼠标悬停焦点与键盘/手柄焦点在 4.6 起分离，可同时存在于不同控件；任何“单一选中态”的自绘焦点都会违反双焦点要求。

### Requirements

- 必须支持固定的 12 项屏幕/层清单，且新增屏幕只能通过改 GDD。
- 必须支持屏幕栈深度 1–2、模态栈深度 1–2，深度满时拒绝而不排队。
- 必须支持 `back_target` 的 9 级优先级，且 `Blocked` 强制模态忽略 `ui_cancel`。
- 必须支持每屏 `initial_focus` 与模态关闭后的 `focus_restore`（含降级到顺序表第一可用控件的 WARNING 路径）。
- 必须在 `Paused` 时不卸载、不重建 HUD（HUD 实例 id 不变）。
- 必须在 `Run → Menu` 携带 `RunResult` 的同一帧把 `ReceiptScreen` 设为根屏。
- 必须在关闭视觉过渡时（`reduce_motion`）切换仍立即生效。
- 必须让界面栈与返回决策在 `--headless` 下可测。

## Decision

### D1 — 控制器与场景单位：一个 `UIRoot` + 每屏一个 `PackedScene`（QQ-05）

采用**混合方案**：

- **一个常驻控制器** `UIRoot`（`Node`，全局恰好一个实例，挂在启动场景下，**不是 autoload**）。它拥有屏幕栈、模态栈、焦点规格应用、返回决策与全部界面语义事件，生命周期与应用相同，从不被替换。
- **每个屏幕一个独立 `PackedScene`**，根节点脚本继承 `ScreenView`（`Control`）。场景由 `UIRoot` 在推入时 `instantiate()` + `add_child()`，弹出时 `remove_child()` + `queue_free()`。

这直接回答 QQ-05：不是“每屏一个场景、用 `change_scene_to_file` 切换”，也不是“一个巨型场景”，而是“一个控制器管理多个独立场景的实例”。

**不采用 `SceneTree.change_scene_to_file()` 的理由**：它会卸载整棵场景树，连带摧毁 HUD 与 `SceneFlow` 自身；无法保留屏幕栈；切换至少推迟一帧；且与 `Run → Paused` 的“不卸载 HUD”直接冲突。**不采用单一巨型场景的理由**：12 个屏幕的控件树全部常驻会让焦点解析变成全树扫描，且屏幕状态的“重建”语义（顶层切换时丢弃旧状态界面）无法表达。

**实例不是长驻的**：同一 `ScreenId` 再次打开时是一个**新实例**。这直接落地 `ui-flow.md` 规则 3（顶层切换生效时丢弃屏幕栈与模态栈、按新状态重建）与规则 12（再来一局不复用上一局任何状态）。唯一常驻的界面节点是 `NoticeLayer`（见 D3）。

### D2 — 节点层级与 CanvasLayer 分层

```
Main (res://src/main/main.tscn)                 # 组合根；ADR-0004 决定其构造方式
├── SceneFlow            (Node)                 # 顶层状态机（Foundation）
├── InputRouter          (Node)                 # 上下文档位 + 焦点/悬停查询（Foundation）
├── SaveSystem / MetaUpgrades / Records / ...   # 其余服务（构造方式见 ADR-0004）
├── HudHost              (CanvasLayer, layer 10)# 局内 HUD 所有；UIRoot 不持有、不操作
├── UIRoot               (Node)                 # 唯一界面控制器
│   ├── ScreenHost       (CanvasLayer, layer 20) # 屏幕实例直接作为子节点（最多 2 个）
│   ├── OverlayHost      (CanvasLayer, layer 25)# PauseOverlay
│   ├── ModalHost        (CanvasLayer, layer 30)# 模态遮罩 + 面板
│   └── NoticeHost       (CanvasLayer, layer 40)# NoticeLayer / ReadOnlyNotice
└── FadeLayer            (CanvasLayer, layer 50)# 过渡覆盖，presentation only
```

分层的三条硬规则：

1. **HUD 在 ScreenHost 之下**：这样 `Settling` 时 `ReceiptScreen` 天然覆盖 HUD，不需要改 HUD 的 z 序。
2. **OverlayHost 在 HUD 之上、ModalHost 之下**：暂停菜单盖住 HUD，但暂停菜单里打开的设置模态盖住暂停菜单。
3. **FadeLayer 在最高层且 `mouse_filter = MOUSE_FILTER_IGNORE`**：它永远不接收输入，也不参与焦点。`NoticeLayer` 的全屏根节点同样是 `IGNORE`，只有其上的提示条目接收输入，否则它会吞掉所有点击。
4. **整条 UI 路径必须是 `PROCESS_MODE_ALWAYS`**：`UIRoot`、四个宿主、`FadeLayer`、`InputRouter` 及其输入分发节点。若暂停用 `SceneTree.paused = true` 实现（实现方式归 ADR-0004），默认的 `PROCESS_MODE_INHERIT` 会同时停掉 `PauseOverlay` 的输入与过渡，暂停菜单变成死界面。这条要求与本 ADR 选哪种暂停机制无关，因此无条件成立。

`UIRoot` 只操作 `ScreenHost` / `OverlayHost` / `ModalHost` / `NoticeHost` 四个宿主；**它没有 `HudHost` 的引用**。Run 场景的挂载与卸载由组合根按 `SceneFlow.state_changed` 驱动，界面流程不感知。

### D3 — 屏幕清单的实例化策略

| Screen / 层 | 宿主 | 场景文件 | 生命周期 |
|---|---|---|---|
| `BootWaiting` | ScreenHost slot 0 | `src/ui/screens/boot_waiting.tscn` | 进入 `Boot` 时创建，离开时销毁 |
| `ErrorScreen` | ScreenHost slot 0 | `src/ui/screens/error_screen.tscn` | 进入 `Error` 时创建 |
| `MainMenu` | ScreenHost slot 0 | `src/ui/screens/main_menu.tscn` | `Menu` 且无 `RunResult` 时成为根屏 |
| `ReceiptScreen` | ScreenHost slot 0（**替换根**） | `src/ui/screens/receipt_screen.tscn` | `Menu` 且携带 `RunResult` 时成为根屏 |
| `UpgradeScreen` | ScreenHost slot 1 | `src/ui/screens/upgrade_screen.tscn` | 从根屏 push；禁止再 push |
| `RecordsScreen` | ScreenHost slot 1 | `src/ui/screens/records_screen.tscn` | 从根屏 push；禁止再 push |
| `PauseOverlay` | OverlayHost | `src/ui/overlays/pause_overlay.tscn` | `Paused` 进入时创建，恢复 `Run` 时销毁 |
| `SettingsModal` | ModalHost slot 0 | `src/ui/modals/settings_modal.tscn` | push / pop |
| `ConfirmModal` | ModalHost slot 0 或 1 | `src/ui/modals/confirm_modal.tscn` | push / pop |
| `BlockedModal` | ModalHost slot 0（强制） | `src/ui/modals/blocked_modal.tscn` | `Menu` 入口按 `LoadReport = Blocked` 创建 |
| `ReadOnlyNotice` | NoticeHost | `src/ui/notices/read_only_notice.tscn` | `Menu` 入口按 `LoadReport = ReadOnly` 创建一次 |
| `NoticeLayer` | NoticeHost | `src/ui/notices/notice_layer.tscn` | **启动时创建一次，永不移除**；顶层切换只清空条目 |

**实例化规则**：

- 12 个场景全部以 `const SCENES := { ScreenId.BOOT_WAITING: preload("..."), ... }` 形式预加载。预加载在脚本加载时完成，避免首次打开时的磁盘 I/O，也避免异步加载与“同帧”约束冲突。
- 实例按需创建，**创建与挂载必须在同一次同步调用内完成**。
- 销毁一律 `remove_child()` 后 `queue_free()`。**禁止直接 `queue_free()` 仍在树上的屏幕**：`queue_free()` 在帧末才释放，节点在该帧剩余时间里仍然参与绘制，会产生“旧画面多渲染一帧”的中间帧，违反 `scene-flow.md` AC 31。
- 模态与 `ReadOnlyNotice` 每次打开都新建实例，关闭即销毁；不做实例复用，避免跨次打开残留状态。

### D4 — 屏幕栈与模态栈

逻辑放在两个无节点依赖的 `RefCounted` 模型中，`UIRoot` 只是它们的适配器：

```gdscript
class ScreenStackModel extends RefCounted:
    const MAX_UI_STACK_DEPTH := 2
    var _stack: Array[ScreenId] = []
    func depth() -> int
    func top() -> ScreenId
    func push(id: ScreenId) -> ScreenPushResult      # 首次 push 必须是该状态的根屏；深度 2 只能从根屏推一个子屏
    func replace_root(id: ScreenId) -> ScreenPushResult
    func pop() -> ScreenPushResult

class ModalStackModel extends RefCounted:
    const MAX_MODAL_DEPTH := 2
    var _stack: Array[ModalId] = []
    func depth() -> int
    func push(id: ModalId) -> ModalPushResult        # 深度已满返回 REFUSED，不排队、不替换
    func pop() -> ModalPopResult
```

**屏幕栈规则**：

- 深度 2 只允许 `[根, UpgradeScreen | RecordsScreen]`；`[ReceiptScreen, 子屏]` 是缺陷，`push` 返回 `INVALID_ROOT` 并记 ERROR。
- `ReceiptScreen` 只能经 `replace_root()` 出现，不能 `push` 到 `MainMenu` 之上。`返回菜单` 同样走 `replace_root(MAIN_MENU)`。
- `BootWaiting` / `ErrorScreen` / `MainMenu` / `ReceiptScreen` 各自是所在状态的唯一根；在它们之上只允许推一个 `UpgradeScreen` / `RecordsScreen`，再推第三层返回 `INVALID_ROOT`。`PauseOverlay` 是 `Paused` 状态在 `OverlayHost` 中的唯一界面根，但它**不进入 `ScreenStackModel`**，因此在 `Paused` 状态下屏幕栈始终为空。
- `Run` 与 `Quit` 的 `ui_stack_depth = 0`。

**模态栈规则**：

- 每次 `push` 调用 `input.push_context(&"Modal")`，每次 `pop` 调用 `input.pop_context()`；`UIRoot` 维护 `_modal_context_depth` 计数并在每次操作后断言 `_modal_context_depth == _modal_stack.depth()`。
- **顶层切换时不得调用 `pop_context()`**：`SceneFlow` 已用 `request_context(default, clear_stack := true)` 清空临时栈（`scene-flow.md` 规则 3）；`UIRoot` 在重建时直接把 `_modal_stack` 清空并把 `_modal_context_depth` 归零，避免二次弹栈（`ui-flow.md` Edge Cases）。
- 深度已满的新模态请求返回 `false` 并记 WARNING；调用方等待，不排队（`ui-flow.md` 规则 5）。
- `ConfirmModal` 在深度 1 时用于退出/放弃/购买/删除生涯；`SettingsModal` 在深度 1，其内部“删除生涯”打开第二个 `ConfirmModal` 到深度 2。第三层一律拒绝。

**`back_target`** 由 `BackResolver.resolve()` 静态函数按 `ui-flow.md` Formulas 的 9 级优先级逐条实现，返回 `BackTarget` 枚举；`UIRoot.handle_action(&"ui_cancel")` 只负责把结果映射成动作（关闭模态 / 关闭提示 / 跳过滚动 / 替换根 / 弹屏 / 打开退出确认 / 无动作）。`BlockedModal` 的 `cancel_enabled = false`，因此命中第 2 级返回 `NONE`。

### D5 — 焦点：声明式顺序 + 手工邻居

**结论：全手工、由每个屏幕声明，不使用 Godot 的几何自动解析。**

每个 `ScreenView` 暴露一份 `FocusSpec`：

```gdscript
class FocusSpec extends RefCounted:
    var order: PackedStringArray          # 屏幕位置顺序（上→下、左→右），用于 initial_focus 与降级
    var neighbours: Dictionary            # id -> { up, down, left, right }（StringName 或 &""）
    var wrap: bool                        # 是否首尾循环；模态恒为 true
```

`order` 与 `neighbours` 在同一个文件中声明、同一个 `apply()` 调用中生效，避免两个来源漂移。

- `apply(spec, controls)` 在 **`present(view_model)` 之后**调用，并在每次控件增删或可见性/可用性变化后重新调用。顺序固定为 `instantiate → present → apply → grab_focus(initial)`；在 `_ready()` 里应用会在动态控件尚未创建时写下一份过期邻居图。
- **方向目标必须预先解析到“下一个可用控件”，不能直接指向声明中的相邻项**：`find_next_valid_focus()` 在显式邻居不可聚焦（隐藏或 `disabled`）时会回落到几何解析，而几何在文字缩放 150% 重排后可能给出与声明顺序相反的结果。因此 `apply()` 对每个方向沿声明顺序前进，**跳过禁用或隐藏的控件**，把 `focus_neighbor_*` 直接指向第一个可用者；整条方向都没有可用控件时才置空，由 `wrap` 或屏幕根节点兜底。升级页的“已满级”按钮、主菜单的“开始营业”被禁用时，导航不得跳到意外位置。
- **禁止依赖几何自动解析**：不得省略 `focus_neighbor_*` 让引擎自行按矩形挑选，也不得让任何可聚焦控件的四个方向中有一个指向不可用目标。
- `initial_focus = first(c ∈ order | c.visible ∧ c.enabled)`；`order` 中没有任何可用控件时返回 `INVALID` 并记 ERROR（`ui-flow.md` Formulas 与断言）。破坏性确认的 `order` 以「取消」开头。
- 焦点移动本身**不由界面流程执行**：方向动作的重复节奏归输入模块（`input.md` Formulas 的 `0.35 s / 0.12 s`），界面流程只负责把邻居图设好，使 `find_next_valid_focus()` 得到声明式结果。
- **模态焦点陷阱**：`ModalHost` 推入模态时，对模态 `FocusSpec` 的四个方向做首尾闭环（`wrap = true`），同时闭环 `focus_next` / `focus_previous`，使 Tab 与 `ui_focus_next` 也留在模态内。遮罩 `Scrim` 设为 `mouse_filter = MOUSE_FILTER_STOP` 只能挡住鼠标，**挡不住 Tab 与 `ui_focus_next`**，因此推入模态的同时必须调用底层屏幕的 `ScreenView.set_blocked(true)`：递归把其可聚焦控件的 `focus_mode` 置 `FOCUS_NONE`、`mouse_filter` 置 `IGNORE`，关闭模态时按打开前的状态恢复。仅靠遮罩会让“主菜单其他控件不可操作”在键盘路径下失效。
- **`focus_restore`**：打开模态时记录 `ModalEntry { modal_id, previous_focus_id: StringName, origin_screen: ScreenId }`，**记录 id 而不是节点引用**（节点可能已被替换）。关闭时按 id 在来源屏幕中查找：存在且 `visible ∧ enabled` → `grab_focus()`；否则取来源屏幕 `order` 中第一个可用控件并记 WARNING。`focus_restore` 的结果写入 `modal_closed(modal_id, restored_focus_id)` 事件。
- 屏幕重绘（例如菜品卡被吃掉、升级项变灰）不得丢失焦点：`ScreenView` 重建子控件后必须按 id 重新 `grab_focus()` 到同一控件；该控件消失时按 `order` 落到相邻可用控件。

### D6 — 双焦点呈现（Godot 4.6 行为）

- **键盘/手柄焦点**与**鼠标悬停**是两条独立记录：
  - `InputRouter.get_focus_owner()` 是 `get_viewport().gui_get_focus_owner()` 的投影；
  - `InputRouter.get_hover_owner()` 由 `InputRouter` 订阅各已注册 `Control` 的 `mouse_entered` / `mouse_exited` 维护（引擎不提供“当前悬停控件”的查询接口，这一条是 4.6 双焦点下的实现责任）。节点被移除或隐藏时 `mouse_exited` 不保证触发，因此 `InputRouter` 还必须在 `tree_exiting` 与每次屏幕拆卸时清空悬停记录；否则悬停状态会指向已释放的控件。
- **视觉分离**：每个可交互控件的 `Theme` 必须同时定义 `focus` 与 `hover` 两个 `StyleBox`，且二者互不推导。`focus` 只在 `has_focus()` 时绘制（≥2 px 描边或外扩轮廓，不只换颜色）；`hover` 只在鼠标位于控件内时绘制（更淡的底色）。两者可同时出现在不同控件上。**`focus` 样式必须是纯描边**（`draw_center = false`），否则它会盖住 `hover` 的底色，视觉上就只剩一种状态。
- **鼠标移动绝不调用 `grab_focus()`**：悬停只更新 hover 记录。`has_focus()` 在 4.6 双焦点下是否会被“鼠标焦点”满足尚未在实机验证，因此该行为列入 `Verification Required`；若验证结果是会被满足，`ActionButton` 改为以 `InputRouter.get_focus_owner()` 是否等于自身来决定是否绘制 `focus` 样式，而不是读 `has_focus()`。无论走哪条路径，悬停移动都不得改变键盘焦点。
- **屏幕不得自绘“选中态”**：任何“当前项高亮”都必须来自 `has_focus()` 或悬停，不得引入第三套状态。
- **点击不改变键盘焦点的语义**保持不变：鼠标移动只更新 hover，不调用 `grab_focus()`；点击触发一次激活（见 D9），是否顺便把键盘焦点移到被点击控件由引擎的 `focus_mode` 决定，界面流程不额外干预。

### D7 — 暂停覆盖层与 `Settling` 收据覆盖

**`PauseOverlay` 不卸载、不重建 HUD。** HUD 归 `HudHost`，由组合根在进入 `Run` 时挂载、离开 `Run` 时卸载；`Paused` **不是**离开 `Run`。因此：

| 切换 | `HudHost` | `UIRoot` |
|---|---|---|
| `Menu → Run` | 挂载 HUD | 清空屏幕栈（深度 0） |
| `Run → Paused` | **不动** | OverlayHost 挂载 `PauseOverlay` |
| `Paused → Run` | **不动** | OverlayHost 移除并销毁 `PauseOverlay` |
| `Run/Paused → Menu` | 卸载 HUD | 按 payload 重建 `[MainMenu]` 或 `[ReceiptScreen]` |

`PauseOverlay` 的条目按 `ui-flow.md` 规则 15 固定为「继续」「设置」「放弃本局」「退出游戏」，初始焦点「继续」；当 `run_state ∈ {Ending, Settling, Handoff}` 时隐藏「放弃本局」。它只读 `RunFlow` 的 `run_state` 快照，不缓存第二份阶段状态。

**`Settling` 收据覆盖的时序**（同一帧内，按此顺序）：

1. `SceneFlow` 置状态为 `Menu` 并同步发出 `state_changed(Run, Menu, RunResult)`；
2. `UIRoot` 处理 payload：`replace_root(RECEIPT_SCREEN)` — 收据实例在同一帧挂到 ScreenHost（layer 20，位于 HudHost layer 10 之上）；
3. 组合根随后卸载 `HudHost`；
4. 渲染该帧时画面上是收据。

第 2 步先于第 3 步是硬约束：它同时满足“收据覆盖 HUD”与“HUD 保留到收据出现的那一帧”。顺序由组合根的单一 `state_changed` 处理器保证，界面流程与 HUD 之间**没有直接调用**。

### D8 — 过渡与淡入淡出（≤0.2 s，presentation only）

`FadeLayer` 是一个铺满视口的 `ColorRect`（纸色 `#F2EDE3`，`mouse_filter = IGNORE`），只有它能播放顶层过渡。

- **状态先变，过渡后播**：`SceneFlow` 同步写入新状态并发出信号 → `UIRoot` 同步重建界面 → 之后才启动 `FadeLayer` 的 `Tween`。过渡从不 `await`，也不进入任何状态机；渲染该帧时目标状态已在位。
- **实现为“遮盖—揭示”**：切换帧把 `FadeLayer.color.a` 置 1，随后 `Tween` 到 0，时长 `FADE_OUT + FADE_IN`（两者各 0–0.10 s，合计 ≤0.20 s）。**两个时长相加成一个揭示时长，而不是先淡出旧屏再淡入新屏**——真正的两段交叉淡变要求旧屏幕在状态已改变后继续渲染，直接违反 `scene-flow.md` AC 31（不得出现“状态已变但画面仍是旧状态”的中间帧）。新状态始终在遮盖之下、同一帧生效。
- **`reduce_motion = true`** → `FADE_OUT = FADE_IN = 0`，`FadeLayer` 从不置 1，切换仍然同帧生效。
- **新请求中止旧过渡**：任何新的合法切换立即 `kill()` 现有 `Tween` 并重新开始，不叠加、不排队。
- **不播过渡的切换**：`Run ↔ Paused`（暂停要“原样”，`ui-flow.md` Game Feel）与任何进入 `Quit` 的切换。`Boot → Menu`、`Boot → Error`、`Menu → Run`、`Run → Menu` 播放。

### D9 — 动作进入与激活的唯一权威

界面语义事件 `ui_button_activated(control_id)` 必须“每次已提交的激活恰好一次”，而 Godot 的聚焦 `BaseButton` 会在 `ui_accept` 上自动激活。为避免双发，规定：

- **`ui_accept` 与 `ui_cancel` 由输入模块在 `_input` 阶段消费并标记为已处理**（`get_viewport().set_input_as_handled()`），使其不再进入 GUI 的自动激活路径；方向动作**不消费**，照常到达聚焦控件（滑杆的左右调整、列表滚动依赖引擎内置处理）。
- 输入模块把已消费的动作交给 `UIRoot.handle_action(action) -> bool` 这一个入口；**这是界面流程接收动作的唯一入口**。
- `UIRoot` 决定语义：收据滚动中 → 跳过滚动（`skip_consumed = true`，不发激活）；否则对当前焦点控件调用其 `activate()`。
- 鼠标点击不合成 `ui_accept`：点击经引擎 GUI 到达控件，`ActionButton.pressed` 触发同一个 `UIRoot._activate(control)`。两条路径汇聚到一处，保证同一帧至多一次 `ui_button_activated`。
- 屏幕**禁止直接监听 `InputEvent`**，也**禁止靠 `BaseButton` 的自动激活产生语义事件**；所有按钮使用项目内的 `ActionButton`（`BaseButton` 子类）并只通过 `activate()` 提交。
- **激活可能销毁发出信号的控件**：`再来一局` / `返回菜单` 会在 `pressed` 的调用栈内触发 `replace_root()`。`remove_child()` + `queue_free()` 的顺序保证控件不会在信号发射过程中被释放，但 `UIRoot` 仍必须把“销毁旧屏”放在本次激活处理的最后一步，且不在信号回调里同步 `free()`。
- **输入侧消费 `ui_cancel` 的边界**：MVP 的界面里没有 `LineEdit` / `TextEdit` / `PopupMenu`（设置用滑杆、开关与按钮），因此全局消费 `ui_cancel` 不会夺走文本编辑的取消语义。若将来引入文本输入，必须改为按焦点控件类型决定是否消费。

> 该约束把“动作如何送达界面流程”留给 ADR-0005 决定（信号、轮询或 `consume_action`），本 ADR 只固定“只有一个入口、每次按下只送一次、界面不碰原始事件”。

### D10 — 状态映射、重建与同帧时序

`UIRoot.reset_for_state(from, to, payload)` 是唯一的重建入口，同步完成以下动作，不 `await`、不 `call_deferred`：

| `to` | 屏幕栈 | 模态栈 | 其他 |
|---|---|---|---|
| `Boot` | `[BootWaiting]` | 清空 | — |
| `Error` | `[ErrorScreen]` | 清空 | 只提供「退出」；debug 构建额外「重新加载数据」 |
| `Menu` | `payload is RunResult` → `[ReceiptScreen]`，否则 `[MainMenu]` | 清空 | 首次进入 `Menu` 时**查询**一次 `SaveSystem` 已产出的 `LoadReport`（界面流程不发起 `load()`，调用时机归 ADR-0004）；`Blocked` → 同帧 push `BlockedModal`；`ReadOnly` → 同帧挂 `ReadOnlyNotice`；`init_failed`/`fault`/`abandoned` → `NoticeLayer` 条目 |
| `Run` | `[]` | 清空 | `UIRoot` 不挂任何屏幕；HUD 由组合根挂载 |
| `Paused` | `[]` | 清空 | OverlayHost 挂载 `PauseOverlay` |
| `Quit` | 清空 | 清空 | 不播过渡 |

规则：

- **重建即丢弃**：任何顶层切换都先丢弃屏幕栈、模态栈与 `NoticeLayer` 条目，再按新状态创建。旧实例 `remove_child()` + `queue_free()`。
- **不 `pop_context()`**：见 D4。
- **`RunResult` 恰好一次**：`UIRoot` 按 `run_id` 去重。同一 `run_id` 第二次到达时忽略并记 WARNING；非 `Menu` 状态下到达时拒绝并记 ERROR。
- **不同步暂停**：`PauseOverlay` 的挂载不做淡入。
- **无隐藏计时器**：`UIRoot` 不注册 `_process`；唯一的时间推进是收据滚动（可注入时钟）与保存指示的 1.5 s 淡出（由 `NoticeLayer` 内部的 `Tween` 完成，不携带数据语义）。

### D11 — 无头可测性

- **逻辑与节点分离**：`ScreenStackModel`、`ModalStackModel`、`BackResolver`、`FocusSpec`、`RollEngine` 全部是 `RefCounted`，不引用 `Node` / `SceneTree`，可在纯 GDScript 单元测试中实例化。
- **屏幕工厂可替换**：`UIRoot` 通过 `ScreenFactory` 获取实例：

  ```gdscript
  class ScreenFactory extends RefCounted:
      func create(id: int) -> ScreenView   # 生产中读 preload 表；测试中返回 stub Control
  ```

- **服务注入**：`UIRoot` 的所有上游（`SceneFlow`、`InputRouter`、`SaveSystem`、`MetaUpgrades`、`Records`、`RunFlow` 快照读取器）都在构造时注入，不按名字查 autoload，符合架构原则 8 与 `ui-flow.md` AC 的 F3 调用记录器夹具。
- **帧号可观测**：`UIRoot` 的所有语义事件带当前帧号（`Engine.get_process_frames()`），使 `receipt_open_latency_frames = 0` 与 `activation_event_count` 能在无头日志中逐帧断言。

### Architecture Diagram

```
                    SceneFlow.state_changed(from, to, payload)   [synchronous, same frame]
                                      │
                                      ▼
┌───────────────────────────── UIRoot (Node) ─────────────────────────────┐
│  ScreenStackModel (≤2)   ModalStackModel (≤2)   BackResolver (pure)     │
│  FocusSpec.apply()       RollEngine (pure)      ScreenFactory           │
│                                                                          │
│  reset_for_state(from, to, payload) ──► mount / remove (synchronous)    │
│  handle_action(action) ──► back_target / activation / skip-roll         │
└───┬───────────────┬────────────────┬───────────────────┬─────────────────┘
    │               │                │                   │
 ScreenHost      OverlayHost     ModalHost          NoticeHost
 (layer 20)      (layer 25)      (layer 30)         (layer 40)
    │               │                │                   │
 [BootWaiting]   [PauseOverlay]  [SettingsModal]     [NoticeLayer]  ← 常驻
 [ErrorScreen]                   [ConfirmModal]      [ReadOnlyNotice]
 [MainMenu]                      [BlockedModal]
 [ReceiptScreen]
 [ + Upgrade/Records (slot 1) ]

 HudHost (layer 10)  ← 组合根按 Run 状态挂载/卸载；UIRoot 无引用
 FadeLayer (layer 50)← 纸色 ColorRect，presentation only，不接收输入
```

### Key Interfaces

```gdscript
# 呈现层类型（src/ui/ui_types.gd，class_name UIFlowTypes）

enum ScreenId {
    BOOT_WAITING, ERROR_SCREEN, MAIN_MENU, UPGRADE_SCREEN,
    RECORDS_SCREEN, RECEIPT_SCREEN, PAUSE_OVERLAY,
}
enum ModalId { SETTINGS, CONFIRM, BLOCKED }
enum BackTarget {
    DISMISS_NOTICE, CLOSE_MODAL, SKIP_ROLL, REPLACE_ROOT,
    POP_SCREEN, OPEN_QUIT_CONFIRM, NONE,
}

# 唯一界面控制器
class_name UIRoot
extends Node

signal screen_changed(screen_id: int, ui_stack_depth: int)
signal modal_opened(modal_id: int, modal_depth: int)
signal modal_closed(modal_id: int, restored_focus_id: StringName)
signal receipt_opened(run_id: int)
signal receipt_closed(run_id: int, reason: StringName)
signal ui_button_activated(control_id: StringName)

func attach(scene_flow, input, save_system, meta_upgrades, records,
            run_flow, factory: ScreenFactory) -> void

func reset_for_state(from: int, to: int, payload: Variant) -> void
func push_screen(id: int, model: Variant = null) -> bool
func replace_root(id: int, model: Variant = null) -> bool
func pop_screen() -> bool
func push_modal(id: int, model: Variant = null) -> bool
func pop_modal(accepted: bool = false) -> bool
func handle_action(action: StringName) -> bool

func screen_stack() -> Array[int]     # 只读副本
func modal_stack() -> Array[int]      # 只读副本
func screen_id() -> int
func is_roll_active() -> bool

# 屏幕契约：每个 .tscn 根节点脚本必须继承
class_name ScreenView
extends Control

var screen_id: int

func present(view_model: Variant) -> void
func focus_spec() -> FocusSpec
func control_by_id(id: StringName) -> Control
func set_blocked(blocked: bool) -> void      # Blocked 强制模态期间冻结底层屏幕
```

### Implementation Guidelines

1. **必须**把 `UIRoot` 实现为常驻单实例；**禁止**把它做成 autoload，也禁止在游戏过程中替换它。
2. **必须**在推入屏幕/模态的同一同步调用内完成 `instantiate()` + `add_child()`；**禁止**在状态或界面切换路径上使用 `await`、`call_deferred()`、`Tween.tween_await()`、`ResourceLoader.load_threaded_request()`。
3. **必须**用 `remove_child()` 后 `queue_free()` 销毁屏幕；**禁止**直接 `queue_free()` 仍在树上的屏幕节点。
4. **必须**让每个屏幕声明 `FocusSpec`（`order` + `neighbours`），并在 `present()` 之后、每次控件增删或启用状态变化后重新应用；**必须**把每个方向预先解析到第一个可用控件；**禁止**依赖 `find_next_valid_focus()` 的几何自动解析，也禁止把方向指向隐藏或禁用的控件。
5. **必须**为每个可交互控件在 `Theme` 中同时提供 `focus` 与 `hover` 两个 `StyleBox`；**禁止**用同一个样式或同一个状态标志表达两者。
6. **必须**让所有界面按钮继承 `ActionButton` 并经 `activate()` 提交；**禁止**屏幕直接监听 `InputEvent`，也禁止依赖 `BaseButton` 对 `ui_accept` 的自动激活。
7. **必须**在顶层切换时丢弃界面栈与模态栈且**不调用** `pop_context()`；**必须**保持 `_modal_context_depth == modal_stack.depth()`。
8. **必须**在 `Run → Menu` 携带 `RunResult` 时先挂 `ReceiptScreen`、后卸载 `HudHost`；**禁止**界面流程直接操作 HUD 节点。
9. **必须**让 `FadeLayer` 的 `mouse_filter` 为 `IGNORE`，且过渡 `Tween` 不参与任何状态判定；**禁止**在过渡未播完时拒绝或延迟状态切换。
10. **必须**在 `project.godot` 中显式写入 `run/main_scene = "res://src/main/main.tscn"`、`display/window/size/viewport_width = 1280`、`display/window/size/viewport_height = 720`、`display/window/stretch/mode = "canvas_items"`、`display/window/stretch/aspect = "expand"`。4.7 只为**新项目**改默认值，已有项目保持自己的设置，因此显式写入是唯一能保证 `TR-in-game-hud-002` 的方式。
11. **禁止**用 `SceneTree.change_scene_to_file()` 切换界面；**禁止**把 12 个屏幕做成一个场景的可见性开关。
12. **必须**把 `NoticeLayer` 做成启动时创建一次的常驻节点，顶层切换时只清空条目；其全屏根节点 `mouse_filter` 为 `IGNORE`。
13. **必须**把 `UIRoot`、四个宿主、`FadeLayer` 与输入分发节点的 `process_mode` 设为 `PROCESS_MODE_ALWAYS`；**禁止**让它们继承暂停。
14. **必须**在推入任何模态时调用底层屏幕的 `set_blocked(true)`，关闭时恢复；**禁止**只靠遮罩 `MOUSE_FILTER_STOP` 实现“底层不可操作”。

## Alternatives Considered

### Alternative 1: 每屏一个场景 + `SceneTree.change_scene_to_file()`

- **Description**：每次切换调用 `change_scene_to_file()` 加载下一个屏幕场景，引擎负责卸载旧场景。
- **Pros**：最少的自写代码；天然“一屏一场景”；内存占用低。
- **Cons**：卸载整棵场景树，`SceneFlow`、`SaveSystem` 与 HUD 一起被销毁；无法保留屏幕栈；切换至少跨一帧；`Paused` 必然重建 HUD，直接违反 `scene-flow.md` UI Requirements 3 与 `ui-flow.md` AC 4。
- **Estimated Effort**：低。
- **Rejection Reason**：与“暂停不卸载 HUD”“收据同帧出现”两条硬约束不相容。

### Alternative 2: 单一巨型 `ui.tscn`，12 个屏幕常驻并切换 `visible`

- **Description**：所有屏幕作为同一场景的子节点常驻内存，通过 `visible` / `process_mode` 切换。
- **Pros**：无实例化开销；焦点节点路径稳定。
- **Cons**：12 棵控件树同时存在，焦点解析与无障碍树规模失控；无法表达“顶层切换时丢弃旧状态界面”的重建语义；屏幕间状态泄漏（上一局的卡片、上一张收据的内容）需要额外重置逻辑，正是 GDD 规则 12 要避免的。
- **Estimated Effort**：中。
- **Rejection Reason**：与“重建而非复用”的 GDD 语义冲突，且焦点与无障碍成本随屏幕数线性增长。

### Alternative 3: 自动几何焦点邻居（`find_next_valid_focus()` 默认解析）

- **Description**：不设置 `focus_neighbor_*`，让引擎按控件矩形挑选最近邻居。
- **Pros**：零声明成本；布局改动自动跟随。
- **Cons**：顺序依赖几何而非声明的屏幕位置；150% 字号重排后会改变导航顺序；无头测试无法稳定断言邻居；模态焦点陷阱无法表达。
- **Estimated Effort**：低。
- **Rejection Reason**：违反 `input.md` UI Requirements（按屏幕位置、不按几何/节点树），且不可测。

## Consequences

### Positive

- 状态与界面切换严格同帧：`state_latency_frames = 0`、`receipt_open_latency_frames = 0` 由结构保证，而不是靠开发者自觉。
- 屏幕栈/模态栈/返回决策是纯函数，可被无头测试逐条覆盖（9 级 `back_target`、深度上限、`focus_restore` 降级）。
- 暂停不碰 HUD，`ui-flow.md` AC 4 的“HUD 实例 id 不变”成为结构性事实。
- 焦点邻居声明式，文字缩放 150% 与超宽屏重排不改变导航顺序。

### Negative

- 每个屏幕需要一个 `PackedScene` 与一份 `FocusSpec`，样板量高于自动方案；漏写方向会在启动校验中报错，属于有意为之的显式成本。
- 12 个场景预加载增加启动内存（每个都是 Control 场景，量级很小）。
- `ActionButton` + 输入侧消费 `ui_accept` 的方案要求输入模块与界面流程严格配合；两者任一方改动都会引入双发或吞键风险。

### Neutral

- Godot 的 GUI 自动激活被有意绕开，按钮行为由项目自己的 `ActionButton` 定义。
- 过渡采用“遮盖—揭示”而非交叉淡变；`FADE_OUT` 与 `FADE_IN` 相加为一个揭示时长。

## Risks

- **输入模块未消费 `ui_accept`，导致键盘激活与 `UIRoot.handle_action` 双发。** 缓解：ADR-0005 必须确认消费点；在集成测试中以 `activation_event_count = 1` 为门禁。
- **`queue_free()` 误用产生一帧旧画面。** 缓解：代码审查门禁 + 一条无头测试：替换根后立即读取 `ScreenHost.get_child_count()`，必须为 1。
- **`Blocked` 强制模态期间底层屏幕仍可通过鼠标点击操作。** 缓解：遮罩 `MOUSE_FILTER_STOP` + `ScreenView.set_blocked(true)` 禁用底层动作按钮 + AC 26 的“其他控件不可操作”。
- **4.7 拉伸设置与 HUD 独立缩放冲突。** `canvas_items` 作用于根视口，包括 `CanvasLayer`；HUD 的 `hud_ui_scale` 若要与界面不同步缩放，可能需要 `SubViewport` 或反向缩放。缓解：列入 `Verification Required` 第 ④ 条，由 HUD 实现决定手段，本 ADR 只固定工程级拉伸设置。
- **`gui_get_focus_owner()` / `focus_neighbor_*` 在 4.7 未经验证。** 缓解：`Verification Required` 第 ① 条。本 ADR 已接受；在该验证完成前，涉及 `focus_neighbor_*` 回落的实现按待验证处理。
- **`reduce_motion` 的读取来源未定。** 设置模型归界面流程，过渡归 `SceneFlow`；本 ADR 规定 `UIRoot` 在每次切换前把 `reduce_motion` 折算为 `FADE_OUT = FADE_IN = 0` 传给 `FadeLayer`，不要求 `SceneFlow` 读设置（`ui-flow.md` Open Question 2 的一种实现）。
- **禁用控件让焦点邻居失效并回落到几何解析。** 升级页的按钮会动态禁用、主菜单的“开始营业”会因档口无法解析而禁用。缓解：`FocusSpec.apply()` 每次重建都跳过不可用目标（D5），并加一条测试：把所有中间项禁用后，导航仍按声明顺序前进。
- **暂停可能同时冻住界面。** 若暂停用 `SceneTree.paused` 实现，`PauseOverlay` 与过渡会一起停止。缓解：D2 第 4 条要求整条 UI 路径 `PROCESS_MODE_ALWAYS`，并在 `Verification Required` ⑧ 实机验证。
- **模态遮罩只挡鼠标。** Tab 与 `ui_focus_next` 仍可到达底层控件。缓解：`ScreenView.set_blocked(true)` 递归关闭焦点与鼠标（D5），遮罩只作视觉降饱和。
- **悬停记录指向已移除节点。** `mouse_exited` 在节点被移除时不保证触发。缓解：`tree_exiting` 与屏幕拆卸时清空（D6）。
- **`focus` 样式与 `has_focus()` 在双焦点下的对应关系未验证。** 若鼠标悬停满足 `has_focus()`，两套样式会塌成一套。缓解：`focus` 用纯描边、悬停不调用 `grab_focus()`，并按 `Verification Required` ⑦ 决定绘制来源。
- **HUD 独立缩放可能与工程级拉伸叠加。** `canvas_items` 已把根视口缩放到窗口；HUD 再按原始窗口像素计算 `hud_ui_scale` 会二次缩放。缓解：HUD 实现必须先除以当前内容缩放（或改用 `SubViewport`），见 `Verification Required` ④。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `scene-flow.md` | `TR-scene-flow-001`：任意时刻恰好一个顶层状态，本系统是唯一切换所有者 | `UIRoot` 不持有也不写状态；它只听 `SceneFlow.state_changed` 并把状态翻译成界面（D10）。顶层状态所有权保持单一 |
| `scene-flow.md` | `TR-scene-flow-002`：`request_transition(to, payload)` 返回 bool，同帧校验、首个合法者生效、payload 原样转交 | payload 在 `state_changed` 中以同一引用同步到达 `UIRoot.reset_for_state()`，无中间缓存；`RunResult` 按 `run_id` 恰好交付一次（D10） |
| `scene-flow.md` | `TR-scene-flow-003`：`Boot` 等待数据 `Ready/Failed`；`Error` 不可进 `Menu`/`Run`；debug 可重载 | 屏幕映射表把 `Error` 固定为 `[ErrorScreen]` 且 `ErrorScreen` 只有退出控件；debug 重载走 `SceneFlow` 的 `Error → Boot`，不经界面流程（D10） |
| `scene-flow.md` | `TR-scene-flow-004`：暂停冻结餐局子系统；恢复还原现场；暂停期间不推进回合/结算 | `Run ↔ Paused` 不触碰 `HudHost`，`PauseOverlay` 只读 `run_state`；冻结由 `RunFlow` 与暂停机制负责，界面流程不复制状态（D7） |
| `scene-flow.md` | `TR-scene-flow-005`：`state_changed` 信号；切换同帧生效；淡入淡出 ≤0.2 s 且只属表现层 | 状态同步写入后由 `UIRoot` 同步重建；`FadeLayer` 在状态生效之后启动、从不 `await`、总时长 ≤0.2 s（D8） |
| `ui-flow.md` | `TR-ui-flow-001`：固定屏幕清单；屏幕栈 ≤2、模态栈 ≤2 | 12 项清单逐项规定宿主、场景与生命周期（D3）；两个 `RefCounted` 模型强制上限并拒绝第三层（D4） |
| `ui-flow.md` | `TR-ui-flow-002`：确定性 `back_target` 优先级；强制 `Blocked` 模态忽略 `ui_cancel` | `BackResolver.resolve()` 按 9 级优先级实现；`BlockedModal.cancel_enabled = false` 命中第 2 级返回 `NONE`（D4） |
| `ui-flow.md` | `TR-ui-flow-003`：每屏焦点顺序、`initial_focus`、`focus_restore` | 声明式 `FocusSpec`（`order` + `neighbours`）生成手工邻居；`initial_focus` 取首个可用控件；模态关闭按 id 恢复并带降级 WARNING（D5） |
| `in-game-hud.md` | `TR-in-game-hud-002`：1280×720 `canvas_items`/`expand` 布局（顶栏、卡片网格、右栏、详情行） | `project.godot` 显式写入基准分辨率与拉伸设置（Implementation Guideline 10）；HUD 在独立 `HudHost` 层绘制，与界面层共存于同一根视口（D2、Risks 第 4 条） |
| `ui-flow.md` | 规则 4–5：`ReceiptScreen` 替换根、模态与屏幕栈分离 | `replace_root(RECEIPT_SCREEN)`；模态只进 `ModalHost`，不入屏幕栈（D3、D4） |
| `ui-flow.md` | 规则 6–7：焦点唯一可预测可恢复；返回是确定决策 | D5 的 `FocusSpec` 与 D4 的 `BackResolver` |
| `ui-flow.md` | 规则 15：暂停覆盖层不卸载、不重建 HUD；`Settling` 收据覆盖 HUD | D7 的宿主分层与同帧挂载顺序 |
| `ui-flow.md` | 规则 26：过渡只播、不延迟状态、没有加载屏 | D8 |
| `ui-flow.md` | 规则 28：界面状态是输入的纯函数，无隐藏计时器 | `UIRoot` 不注册 `_process`；唯一时间推进是可注入时钟的收据滚动与保存指示淡出（D10、D11） |
| `input.md` | 规则 8 / UI Requirements：双焦点分开记录、分开绘制 | D6：两套 `StyleBox`、两个查询、同一控件可同时处于两种状态 |

## Performance Implications

- **CPU**：空闲时 `UIRoot` **零每帧开销**（不注册 `_process`，全部事件驱动）。一次屏幕挂载 = 一次 `instantiate()` + `add_child()`，`p99` 目标为一帧内完成；单帧内最多发生 2 次挂载与 2 次卸载（替换根 + 模态）。
- **Memory**：任意时刻存活界面节点 = 根屏 + 至多 1 子屏 + 至多 2 模态 + 暂停覆盖 + 常驻 `NoticeLayer`。12 个 `PackedScene` 预加载的常驻成本为场景资源本体，不含实例。
- **Load Time**：预加载在启动时一次完成，界面打开无磁盘 I/O；启动阶段仍受数据校验主导（`Boot` 期）。
- **Network**：不适用（MVP 单机）。

## Migration Plan

当前没有任何界面代码，`run/main_scene` 未设置。实施顺序：

1. 建立 `src/ui/` 目录与 `ScreenView` / `FocusSpec` / `ActionButton` / `UIRoot` 骨架，把 `ScreenStackModel`、`ModalStackModel`、`BackResolver` 作为纯类先写单元测试。
2. 建立 `src/main/main.tscn`，写入 `project.godot` 的 `run/main_scene` 与拉伸设置（Implementation Guideline 10），用 `BootWaiting` + `ErrorScreen` 打通 `Boot → Menu/Error`。
3. 按 `MainMenu → UpgradeScreen / RecordsScreen → ReceiptScreen → PauseOverlay → SettingsModal / ConfirmModal / BlockedModal` 的顺序落地，每个屏幕附 `FocusSpec` 与 stub 工厂测试。
4. 接入 `HudHost` 与 `Run ↔ Paused`，以 `ui-flow.md` AC 4（HUD 实例 id 不变）为门禁。

**Rollback plan**：本 ADR 只引入新增文件与 `project.godot` 的两组键，回滚即删除 `src/ui/`、`src/main/` 并恢复 `project.godot`。没有既有代码依赖此结构。

## Validation Criteria

| # | 验证 | 证据 |
|---|---|---|
| 1 | `ScreenStackModel`：深度 >2 拒绝；`[ReceiptScreen, 子屏]` 返回 `INVALID_ROOT`；`replace_root` 后深度恒为 1 | 无头单元测试 |
| 2 | `ModalStackModel`：深度 2 时第三次 push 返回 `false` 且不排队；同值 `push_context(Modal)` 使深度从 1 增至 2 | 无头单元测试（依赖 ADR-0005 裁定） |
| 3 | `BackResolver`：9 级优先级逐条命中，含 `Blocked` → `NONE`、滚动中 → `SKIP_ROLL` | 表驱动单元测试 |
| 4 | `FocusSpec`：每屏 `initial_focus` 等于声明；`order` 全不可用时记 ERROR；模态邻居闭环；`focus_restore` 目标失效时落到顺序表首项并记 WARNING | 单元测试 + 逐屏焦点清单夹具 |
| 5 | `Run → Menu` 携带 `RunResult`：`receipt_opened` 与 `state_changed` 帧号相同；`screen_stack = [RECEIPT_SCREEN]` | 无头集成（stub `ScreenFactory`）+ 帧号日志 |
| 6 | `Run → Paused → Run`：`HudHost` 子节点实例 id 不变；`screen_stack` 恒为空 | 无头集成 |
| 7 | `activation_event_count`：一次提交恰好 1；滚动跳过、禁用控件、点击空白为 0；同一帧至多 1 | 无头集成 |
| 8 | 替换根后同一帧 `ScreenHost.get_child_count() == 1`（证明 `remove_child` + `queue_free` 顺序） | 无头集成 |
| 9 | 代码中不存在 `change_scene_to_file`、屏幕直接监听 `InputEvent`、`UIRoot` 注册 `_process` | grep 门禁 |
| 10 | 4.7 实机：双焦点同时可见、`focus_neighbor_*` 按声明跳转、`reduce_motion` 关闭过渡后切换仍同帧 | 保留截图 + 手工验收 |
| 11 | 把声明顺序中的中间项禁用或隐藏后，四个方向的导航仍按声明顺序前进，不落到几何最近控件 | 无头单元测试（`FocusSpec` + stub Controls） |
| 12 | 模态打开期间，Tab 与 `ui_focus_next` 无法到达底层屏幕的任何控件；关闭后恢复 | 无头集成 + 手工验收 |

## Related

- `docs/architecture/architecture.md` — Required ADRs 第 3 项、API Boundaries（`SceneFlow` / `InputRouter`）、Open Question QQ-05
- `design/gdd/scene-flow.md` — 顶层状态、同帧生效、过渡上限、暂停与退出规则
- `design/gdd/ui-flow.md` — 屏幕清单、栈深度、`back_target`、`initial_focus`、`focus_restore`、收据同帧打开
- `design/gdd/in-game-hud.md` — 局内一屏归属、暂停覆盖、`Settling` 保留最终画面
- `design/gdd/input.md` — 双焦点、上下文接口、导航重复、禁用控件
- `docs/engine-reference/godot/modules/ui.md`、`docs/engine-reference/godot/modules/input.md` — 4.6 双焦点、4.7 拉伸默认值与 `DEVICE_ID_*`
- ADR-0005（输入上下文栈语义）— 本 ADR 的硬依赖
- ADR-0007（结算事务边界）— 本 ADR 固定其收据挂载点与同帧钩子
