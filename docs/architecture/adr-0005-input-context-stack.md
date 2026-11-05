# ADR-0005: 输入上下文栈语义（Input Context Stack Semantics）

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
- `godot-specialist`（`InputEvent` 合成投递、`Viewport.push_input`、`DEVICE_ID_*`、GUI 焦点导航的 4.7 行为校验）
- `technical-director` 战略审阅：跳过（`review_mode: lean`）

## Summary

输入的上下文状态机在 `input.md` 里已经有四个接口、五张允许表和两条时间公式，但三处语义仍然空着：**同值压栈算不算一次切换**（`QQ-06`）、**动作在 `Menu` / `Modal` / `Paused` 里怎么送达界面**（`QQ-13`，输入 GDD 只给 `Blocked` 定义了 `consume_action`）、以及**导航重复由谁生成**。这三处不闭合，ADR-0003 的模态深度 2、「每次按下恰好一次激活」和「界面不碰原始事件」三条约束都无法同时成立。

本 ADR 裁定：

1. **权威上下文 = 一个基底 + 一个临时栈**（`base` + `stack`），有效上下文是栈顶（栈空时是 `base`）。`push_context` **总是**加深并发出 `context_changed(from, to)`，**包括同值压栈 `context_changed(Modal, Modal)`**；`pop_context` 同样发声；只有 `request_context` 在目标等于当前有效上下文且 `clear_stack = false` 时才是无信号空操作。同值事件的语义是「深度变了，语义没变」，消费者只允许对深度作出反应，**禁止**重跑进入逻辑。
2. **动作只有一个投递路径**：`InputRouter` 在上下文为 `Menu` / `Modal` / `Paused` 时，把 `ui_accept` / `ui_cancel` 同步投递给唯一的 `ActionSink`（生产中由 `UIRootActionSink` 适配到 `UIRoot.handle_action`）；返回 `true` 即置位本帧「已消费」。`Run` 不投递（屏幕栈为空，没有屏可接收），`Blocked` 不投递（只有首局引导经 `consume_action` 取走）。所有读者共享同一个「已消费」位，这就是**首个消费者获胜**的唯一实现。
3. **导航重复由输入层生成，不由引擎**：按住方向的重复步数由纯函数 `NavRepeater` 按 `nav_steps` 公式计算，每一步以合成 `InputEventAction` 同步投递到视口，标记为 `synthetic` 后不再被输入层重复计数；摇杆回滞由纯函数 `StickGate` 按 `stick_engaged` 公式计算。两者都不读引擎的 `event.echo`。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；本机 `Installed at pin time = NOT DETERMINED`，尚未安装编辑器） |
| **Domain** | Input |
| **Layer** | Foundation |
| **Knowledge Risk** | HIGH — 4.7 改变了鼠标/键盘的设备 ID 语义（`InputEvent.DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD`，不再报 `device == 0`）；`Viewport.push_input` 对 `InputEventAction` 是否进入 GUI 焦点导航路径未在参考库中覆盖；4.6 起鼠标悬停焦点与键盘焦点分离 |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`modules/input.md`、`modules/ui.md`、`breaking-changes.md`、`current-best-practices.md`、`deprecated-apis.md` |
| **Post-Cutoff APIs Used** | `InputEvent.DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD`（4.7，仅用于诊断与日志，不用于分类）；4.6 双焦点分离（`get_viewport().gui_get_focus_owner()` 与悬停记录各自独立） |
| **Verification Required** | ① `Viewport.push_input(InputEventAction)` 是否进入 `Control` 的 `ui_up/down/left/right` 焦点导航（决定重复步能否用合成事件实现）；② 合成事件是否在同一次调用内同步处理（决定 1 帧延迟是否成立）；③ 在手柄/键盘事件上 `event.is_action_pressed("ui_accept")` 与 `InputEventAction` 两条路径是否给出同一次「按下沿」；④ `Control.focus_neighbor_*` 在显式指向可用控件时指针设备与方向动作行为一致（ADR-0003 已列，本 ADR 只确认重复步走同一条路径）；⑤ 4.7 下 `InputEventJoypadMotion.axis_value` 的合成幅度与 0.60/0.40 阈值在死区设置下是否符合夹具预期；⑥ `ui_pause` 与 `ui_cancel` 同绑 Esc 时，单个事件在 `_input` 中只被翻译一次（不得双发） |

> ⚠️ **引擎知识缺口**：4.7 已把 Input 列为 HIGH 风险域，且本机没有安装编辑器。
> 上述六项必须在 pin 定的 4.7.x 上实测；Verification Required ① 若为否，
> `NavRepeater` 的投递层要改成「直接对焦点控件调用 `grab_focus()` 到预解析邻居」，
> 并放弃滑杆的左右微调（本作设置页用滑杆，代价明确）。在实测完成前，本 ADR 中
> 涉及运行时语义的结论按「待验证」处理。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | None |
| **Enables** | ADR-0003 的全部模态 story（无同值压栈裁定则删除生涯的深度 2 无法实现）；界面流程、局内 HUD、首局引导的输入 story |
| **Blocks** | 任何需要「弹窗里再弹窗」或「引导压住局内输入」的 story；`QQ-06` / `QQ-13` 未闭合则 `UIRoot.handle_action` 的调用方不存在 |
| **Ordering Note** | ADR-0003 D9 写的是「输入模块把**已消费**的动作交给 `UIRoot.handle_action`」。本 ADR 把它精确化为「先投递，由返回值为真来置位消费位」——两者语义等价（每次按下只送一次、界面不碰原始事件），差别只在于消费位对 `is_action_just_pressed` 的读者可见，而这正是 `QQ-13` 需要的。若 Required ADRs 重新编号，以文件名 `ADR-0005` 为稳定引用。本 ADR 无前置依赖，与 ADR-0003 / ADR-0004 是契约级共同设计：本 ADR 固定输入栈语义与 `ActionSink` 投递点；ADR-0003 消费 `UIRoot.handle_action`，ADR-0004 由组合根构造 `InputRouter`。依赖边只记在 ADR-0003 → ADR-0005 与 ADR-0004 → ADR-0005，拓扑序为 0001 → 0002 → 0005 → 0003 → 0004 → 0006..0009 |

## Context

### Problem Statement

`input.md` 的规则 12 与状态表已经把「谁能请求什么」写得很紧，但三个语义问题被明确留在架构阶段，而它们各自都会让别的 ADR 无法落地：

- **QQ-06（High）**：`ui-flow.md` 规则 5 与 Open Question 14 要求「设置模态里再打开删除生涯确认」——两层模态、两次 `push_context(Modal)`。若第二次同值压栈是空操作，则两次 `pop_context` 会弹出两层却只有一层深度，`UIRoot` 的 `_modal_context_depth == modal_stack.depth()` 断言必然失败；若它是真压栈，那么 `context_changed(Modal, Modal)` 这种 `from == to` 的事件必须有一个约定的处理方式，否则监听者会误以为「刚进入模态」而重跑焦点陷阱。
- **QQ-13（High）**：输入 GDD 只为 `Blocked` 定义了 `consume_action`，且唯一调用方是首局引导。但 `Menu` / `Modal` / `Paused` 里 `ui_accept` 与 `ui_cancel` 必须抵达 `UIRoot.handle_action`——**送达机制完全没有定义**：是界面自己轮询？是信号？还是也走 `consume_action`？ADR-0003 D9 只固定了「只有一个入口、每次按下只送一次、界面不碰原始事件」，把机制留给本 ADR。
- **导航重复**：`input.md` 给了 `nav_steps` 公式与 0.35 / 0.12 的起点值，但 Godot 的 GUI 焦点导航不会自动重复，`event.echo` 又被规则 5 明确忽略。重复步由谁生成、怎么送达焦点控件，未定。

### Current State

- `input.md` 已定义固定八个动作、五个上下文、四张允许表、四个请求接口、两条公式与 AC 1–20（其中 AC 18–20 钉住 `request_context` 的返回值与信号、`Modal` 的压/弹、以及「上下文只经三个接口改变」）。
- `ui-flow.md` 规则 27 把界面流程限制在 `push_context(Modal)` / `pop_context()`；它**不得**调用 `request_context`、不得给上下文赋值。
- `scene-flow.md` 规则 3 是唯一的 `request_context(default, clear_stack := true)` 调用方。
- `first-run-onboarding.md` 规则 6/17 与 AC 13/19/21 依赖 `consume_action` 抢走 `ui_accept`、依赖暂停清栈后重新压入同一白名单。
- 仓库没有 `.gd`；ADR-0003 的 `UIRoot.handle_action(action) -> bool` 已在接口里，但没有调用方。

### Constraints

- **单一动作集**：全局只有八个动作；任何系统不得监听原始事件（`input.md` 规则 1/11）。因此所有投递必须是「动作」而不是 `InputEvent`。
- **同帧**：`input.md` AC 15 要求从事件到消费者 ≤ 1 帧；`ui-flow.md` 要求「每次已提交的激活恰好一次」。
- **双焦点**：`get_focus_owner()` 与 `get_hover_owner()` 分开记录、分开绘制；鼠标移动不得改变键盘焦点。
- **界面不碰原始事件**：屏幕禁止监听 `InputEvent`（ADR-0003 实现指南第 6 条）。
- **模态深度 2 必须可用**：`MAX_MODAL_DEPTH = 2`，第二层仍以 `push_context(Modal)` 进入。
- **暂停会清栈**：`request_context(..., clear_stack := true)` 在每个顶层切换时清空临时栈，因此首局引导的 `Blocked` 在暂停后消失，恢复时必须由它自己重压（`first-run-onboarding.md` 规则 17）。
- **无头可测**：上下文栈、投递、重复步都必须能在 `--headless` 下断言，不依赖真实设备。
- **4.7 设备识别**：一律按事件类型判断，绝不比较 `device == 0`。

### Requirements

- 必须唯一确定同值压栈的语义，并给出 `from == to` 事件的消费约定。
- 必须给出 `Menu` / `Modal` / `Paused` / `Run` / `Blocked` 五种上下文的动作投递表，且每行只有一个消费者。
- 必须让 `consume_action` 与界面投递共享同一个「首个消费者获胜」的状态。
- 必须让上下文切换清空按键残留（按住方向必须松开重按）。
- 必须把导航重复与摇杆回滞实现为可单测的纯函数，并固定递送方式。
- 必须保持「任何代码不得直接给上下文赋值」，且可用静态扫描验证。
- 必须让 `Requested` 的调用方边界可静态检查：`request_context` 只属场景与流程管理，`push/pop` 只属界面流程与首局引导，`consume_action` 只属首局引导。

## Decision

### D1 — 栈模型：一个基底 + 一个临时栈，四个接口的返回值与信号

```gdscript
base:  InputContext          # 权威上下文，只由 request_context 写
stack: Array[Entry]          # 临时栈；Entry = { ctx, allow }
effective = stack.is_empty() ? base : stack.back().ctx
depth = stack.size()
```

| 调用 | 前置 | 结果 | 信号 | 返回 |
|---|---|---|---|---|
| `request_context(ctx, clear_stack = true, allow = [])` | `ctx` 是五个已知值之一；`allow` 里每个名字是八个动作之一 | `base = ctx`；`clear_stack` 为真时清空 `stack` | 有效上下文或深度改变时发**一次** `context_changed(from, to)`；否则不发 | `true` |
| `request_context(ctx, clear_stack = false)` 且 `ctx == effective` | — | 无变化 | 不发 | `true`（幂等空操作） |
| `push_context(ctx, allow = [])` | 同上；`depth < MAX_CONTEXT_DEPTH` | `stack.push_back(Entry)` **无条件** | 恰发一次 `context_changed(from, to)`，同值时为 `(X, X)` | `true` |
| `pop_context()` | `stack` 非空 | `stack.pop_back()` | 恰发一次 `context_changed(from, to)` | `true` |
| `pop_context()` 且 `stack` 为空 | — | 无变化（**绝不弹基底**） | 不发 | `false` |
| 任一接口，`ctx` 不是五个已知值 | — | 无变化 | 不发 | `false` |
| 任一接口，`allow` 含未知动作名 | — | 无变化 | 不发 | `false`（`UNKNOWN_ACTION`） |
| `push_context` 且 `depth == MAX_CONTEXT_DEPTH` | — | 无变化 | 不发 | `false`（`STACK_FULL`） |

- **`MAX_CONTEXT_DEPTH = 2`**：`ui-flow.md` 的模态栈上限是 2，首局引导只压 1 层 `Blocked`，且两者不可能同时存在（引导只在 `Run`，模态只在 `Menu` / `Paused` / `Error`）。因此 2 已覆盖全部合法组合，第三层一定是调用方缺陷——记 ERROR 并拒绝，**不排队、不替换**（与 `ui-flow.md` 规则 5 的模态拒绝同形）。这是不变量，不是可调旋钮。
- **`allow` 的校验边界**：本 ADR 只校验**词汇**（是不是八个动作之一）与**深度**，不校验**策略**（例如「`Menu` 能不能进 `Run`」）。这与 `input.md` 规则 12「输入不判断请求是否合理」一致：`false` 表示「我不知道这个值」，不表示「你不该这么做」。未知动作返回 `false` 而不是静默丢弃，是为了让首局引导的降级路径（AC 19：重压失败则降级为无遮罩提示）能被触发；静默丢弃会造成「气泡还在但按键无效」的死锁。
- **`context_changed` 只由这四个接口发**，且每次被接受的调用恰发一次、payload 是两个有效上下文。`request_context` 在「`ctx == effective` 且 `clear_stack = false`」时是唯一的无信号路径（`input.md` AC 18）。

### D2 — QQ-06：同值压栈总是加深，`from == to` 只表示深度变化

- **`push_context(Modal)` 在栈顶已经是 `Modal` 时仍压入第二层**，发 `context_changed(Modal, Modal)`。这是 `ui-flow.md` Open Question 14 的答案，也是 ADR-0003 模态深度 2 的前提。
- **`from == to` 的消费约定**（封闭）：
  - 只关心**语义**的消费者（焦点陷阱、音效、界面重绘）看到 `from == to` 时必须**什么都不做**：不得重跑进入逻辑、不得重放进入音效、不得重置焦点。
  - 只关心**深度**的消费者（`UIRoot` 的 `_modal_context_depth`、栈断言）看到任何 `context_changed` 都把计数与栈深对齐；同值时计数 +1。
  - 关心**有效上下文**的消费者（首局引导判断 `Blocked` 是否仍生效）只在 `to` 变化时反应。
- **配对不变量**：每次 `push_context` 必须在同一条离开路径上有且只有一个 `pop_context`，且 `UIRoot` 在每次操作后断言 `_modal_context_depth == modal_stack.depth()`（ADR-0003 D4）。`pop_context` 在栈空时返回 `false` 且不弹基底，因此「多弹一次」不会把上下文降级成 `Blocked`。
- **顶层切换不弹栈**：`request_context(default, clear_stack := true)` 已清空栈；`UIRoot` 在重建时直接把 `_modal_context_depth` 归零，**不得**补调 `pop_context`（`ui-flow.md` 边界用例「避免二次弹栈」）。
- **同值压栈不是「刷新」**：它不改变 `base`，`allow` 只作用于栈顶条目。若调用方想要「换一个白名单」而栈顶已是 `Blocked`，必须先 `pop` 再 `push`——两次调用、两次信号。这是刻意的：白名单变化必须有可观测的边界。

### D3 — 动作允许表：唯一过滤源

`InputRouter` 内只有一张表决定「当前能产生哪些动作」。所有查询（`is_action_just_pressed`、`consume_action`、投递）都先过这张表，因此**表外动作在任何路径上都不可见**：

| 上下文 | 允许的动作 |
|---|---|
| `Blocked` | `ui_pause`（恒允许，不可移除） ∪ 栈顶条目的 `allow` |
| `Menu` | `ui_up` / `ui_down` / `ui_left` / `ui_right` / `ui_accept` / `ui_cancel` |
| `Run` | 八个动作全部 |
| `Paused` | `ui_up` / `ui_down` / `ui_left` / `ui_right` / `ui_accept` / `ui_cancel` / `ui_pause` |
| `Modal` | `ui_up` / `ui_down` / `ui_left` / `ui_right` / `ui_accept` / `ui_cancel` |

- `allow` 只在 `Blocked` 上生效；在其它上下文里传 `allow` 会被校验（词汇合法）但不影响允许集，并记一次 WARNING（几乎总是调用方写错了上下文）。
- `ui_pause` 在 `Blocked` 上不可移除：即使 `allow` 为空、即使 `allow` 里写了别的东西，`ui_pause` 仍然可用。这是「引导绝不能把玩家锁死」的最后一道出口（`first-run-onboarding.md` 规则 17）。
- **一次物理事件只产生一个动作**：Esc 在 InputMap 里同时绑 `ui_cancel` 与 `ui_pause`，`InputRouter` 在翻译时按上下文取一个——`Run` / `Paused` 取 `ui_pause`，`Menu` / `Modal` / `Blocked` 取 `ui_cancel`。被淘汰的那个动作**从不进入本帧的动作集**，因此 `is_action_just_pressed(&"ui_cancel")` 在 `Run` 中读到 `false`，即使物理键是同一个。

### D4 — QQ-13：动作投递（唯一路径、按上下文分发、首个消费者获胜）

`InputRouter` 持有一个 `ActionSink`（可空）。**投递表是封闭的**：

| 有效上下文 | `ui_accept` / `ui_cancel` | 导航（四个方向） | `game_skip_wave` | `ui_pause` |
|---|---|---|---|---|
| `Blocked` | **不投递**；只经 `consume_action` 由首局引导取走 | 引擎 GUI（若 `allow` 放行） | 表外，永不产生 | 由 `SceneFlow` 轮询 |
| `Menu` | 投递给 `ActionSink` | 引擎 GUI | 表外，永不产生 | 表外，永不产生 |
| `Modal` | 投递给 `ActionSink` | 引擎 GUI | 表外，永不产生 | 表外，永不产生 |
| `Paused` | 投递给 `ActionSink` | 引擎 GUI | 表外，永不产生 | 由 `SceneFlow` 轮询 |
| `Run` | **不投递**（屏幕栈为空，无屏可接收）→ HUD 轮询 | 引擎 GUI | HUD 轮询后提交 `skip_wave` | 由 `SceneFlow` 轮询 |

- **投递是同步的**：在翻译该动作的同一次 `_input` 调用里完成，不 `call_deferred`、不发信号、不排队。`ActionSink.handle_action(action) -> bool` 返回 `true` 表示界面认领了这次按下（**包括刻意的空操作**，例如强制模态忽略 `ui_cancel`），返回 `false` 表示界面没有消费者（例如 `Run` 里不该出现的情况）。
- **首个消费者获胜**：本帧维护一个 `_consumed: Dictionary[StringName, true]`。`handle_action` 返回 `true` → 置位；`consume_action` 成功 → 置位；`is_action_just_pressed` 在置位后返回 `false`。三者共享同一位，因此「界面先拿到、局内后拿到」不会造成双触发。
- **`consume_action` 只在 `Blocked` 生效**：其它上下文中调用返回 `false`、不置位、并在一帧内只记一次 WARNING。这既守住 `input.md` 对它的定义（「在 `Blocked` 上下文中取走本帧的一个动作」），也让「界面也去 `consume_action`」这条错路在测试里立刻可见。
- **`Run` 不投递的理由**：`ui-flow.md` 的状态表里 `Run` 的 `ui_stack_depth = 0`，没有任何屏。把动作投给一个恒返回 `false` 的 sink 只是形式上的统一，却会让「将来某个屏幕在 `Run` 里被误挂」变成静默的输入被吞。宁可在 `Run` 里明确不投递，让 HUD 是唯一读者。
- **动作投递与 `set_input_as_handled()`**：`ui_accept` / `ui_cancel` 在产生时即调用 `get_viewport().set_input_as_handled()`，阻止聚焦的 `BaseButton` 自动激活（ADR-0003 D9 已要求）。方向动作不消费，照常进入 GUI（滑杆微调、列表滚动依赖引擎内置处理）。
- **`ActionSink` 是一个 `RefCounted` 接口**，不是 `UIRoot`：`InputRouter` 属 Foundation，`UIRoot` 属 Presentation，Foundation 不得在类型层面依赖 Presentation（ADR-0004 D3）。生产中由 `UIRootActionSink`（Presentation 侧）把调用转发到 `UIRoot.handle_action`；无头测试注入一个记录器 sink。

### D5 — 一帧内的动作模型：翻译一次、消费位、`input_seq`

- 每帧开始时（`_input` 之前）清空 `_pressed`、`_consumed`，`input_seq += 1`。
- `_input(event)` 只做三件事：忽略 `event.echo`；按事件类型识别设备；按 D3 的表把事件翻译成「至多一个动作」，并检查该动作是否在当前允许集内。翻译一次即写 `_pressed[action] = true`，同一动作在同一帧的第二次按下被丢弃（这实现「两个设备同时触发同一动作只产生一次动作」）。
- **同一帧的多个不同动作都保留**：`ui_accept` 与 `ui_cancel` 同帧到达时两者都在 `_pressed` 里，但按 D3/D4 只有一个会被投递或轮询成功——`ui-flow.md` 的「只消费一个动作」由此成立，`InputRouter` 不自行合并。
- **`ui_pause` 的帧内优先级**：`SceneFlow` 轮询 `ui_pause` 的 `process_priority = -100`，`OnboardingHost` 的 `process_priority = -50`（`Blocked` 下消费引导动作），`HudHost` 轮询 `ui_accept` / `game_skip_wave` / `ui_cancel` 的 `process_priority = 0`。同帧同时出现 `ui_accept` 与 `ui_pause` 时，暂停先生效、`RunFlow` 已被冻结，局内动作随之落空。这是确定的、可断言的顺序，而不是「看谁先下一帧」。三个常量在启动时断言 `-100 < -50 < 0`。
- **`input_seq`** 每帧递增一次，暴露给需要「同一帧至多推进一次」的消费者（首局引导的 beat 推进）。它是单调整数，不随上下文重置；消费者记下已消费的序号并只接受更大的值。

### D6 — 导航重复与摇杆回滞：纯函数 + 同步合成投递

两个类都不引用 `Node` / `SceneTree`，可脱离引擎单测（对应 `input.md` AC 6/7）：

```gdscript
class NavRepeater extends RefCounted:
    # 首次按下恒 1 步；此后在 t = D + k·I <= T 时各产生一步（边界取 <=）
    func press(dir: StringName, now: float) -> int   # 返回本次应产生的步数（通常为 1）
    func tick(now: float) -> int                     # 按住期间返回新增步数（0 或 >= 1）
    func release(dir: StringName) -> void
    func reset() -> void

class StickGate extends RefCounted:
    # stick_engaged = (m >= E) || (engaged_prev && m >= R)
    func update(magnitude: float) -> bool
    func reset() -> void
```

- **重复步是整数计数，不是浮点累加**：`NavRepeater` 以 `D + k·I` 的整数步数记账（`k` 单调递增），每次 `tick` 只发出 `floor((T − D) / I) + 1 − 已发步数`。浮点只用于与 `T` 比较，避免两套实现差一步（`input.md` Formulas 的边界取 `≤`）。
- **递送方式**：每一个重复步由 `InputRouter` 合成一个 `InputEventAction`（`action = "ui_up"` 等，`pressed = true`），调用 `get_viewport().push_input(ev)` **同步**投递，随后投递对应的 `pressed = false`。合成事件带 `set_meta(&"synthetic_repeat", true)`，`_input` 见到该标记只放行、不再计数、不再触发回滞或重复。
- **为什么不靠 `event.echo`**：规则 5 明确忽略回显，且 GUI 焦点导航本身不重复。为什么不直接 `grab_focus()` 到邻居：设置页的滑杆需要方向键调整数值，绕过 GUI 会丢掉这条内置行为。
- **为什么不自己移动焦点**：焦点顺序由 ADR-0003 D5 的显式邻居图决定，`InputRouter` 只提供「按了几次」，不解析邻居、不调 `grab_focus`。
- **摇杆**：每帧由 `StickGate.update(magnitude)` 得出 `engaged`。`false → true` 产生一个首次步；`true` 期间交给 `NavRepeater`；`true → false` 只 `release`，不产生动作。`magnitude` 的合成（左摇杆与十字键并存）取绝对值较大者，并受手柄死区影响——死区设置列入 Verification Required ⑤。
- **参数来源**：`D` / `I` / `E` / `R` 来自 `InputTuning`（类型化 `Resource`，`res://data/config/input_tuning.tres`，由 Input 拥有、由组合根注入），区间校验 `D ∈ [0.25, 0.60]`、`I ∈ [0.08, 0.20]`、`E ∈ [0.50, 0.75]`、`R ∈ [0.25, 0.50]`、`R ≤ E − 0.10`；越界或缺失时使用代码内默认值（`0.35 / 0.12 / 0.60 / 0.40`）并记 WARNING，**不阻断启动**。`InputTuning` **不进入** `DataConfig` 的启动门：内容校验失败时玩家仍必须能在 `ErrorScreen` 上导航与确认退出，输入时序不能成为启动门的下游。

### D7 — 上下文切换清空按键残留

任何 `context_changed`（含 `from == to`）之后，`InputRouter` 立即清空：`_pressed`、`_consumed`、`NavRepeater` 的全部按住状态与重复计时、`StickGate` 的 `engaged`、以及任何待发的合成重复。

- 这落地 `input.md` 边界用例「上下文切换时方向键正被按住：旧按键不会延续到新上下文，必须重新按下」与 AC 13。
- **不清空**焦点与悬停记录：指针没动，悬停不该消失；键盘焦点由 `UIRoot` 的模态陷阱与 `focus_restore` 负责（ADR-0003 D5）。
- **不重置 `input_seq`**：它是单调序号，消费者依赖「更大的值」而不是「从 0 开始」。

### D8 — 焦点与设备识别（`TR-input-003` / `TR-input-005`）

- `get_focus_owner()` 是 `get_viewport().gui_get_focus_owner()` 的投影，输入层不维护第二份键盘焦点。
- `get_hover_owner()` 由 `InputRouter` 订阅各注册控件的 `mouse_entered` / `mouse_exited` 维护（引擎不提供「当前悬停控件」查询）；控件被移除或隐藏时 `mouse_exited` 不保证触发，因此 `tree_exiting` 与每次屏幕拆卸都必须清空记录（ADR-0003 D6 已要求，本 ADR 只确认归属在输入层）。
- 鼠标移动只更新悬停记录，**绝不** `grab_focus()`；键盘导航不清除悬停。两者可同时存在于不同控件。
- 设备识别**只按事件类型**：`InputEventKey` → 键盘、`InputEventMouseButton` / `InputEventMouseMotion` → 鼠标、`InputEventJoypadButton` / `InputEventJoypadMotion` → 手柄。`InputEvent.DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD` 只用于日志与诊断，**禁止**参与判断，也**禁止**任何地方出现 `device == 0`（`input.md` AC 9 以此为验收）。
- 手柄热插拔：拔出时该设备所有按住的动作按「释放」处理并停止接受该设备事件；上下文不变、不自动暂停（`scene-flow.md` 决定）。接入时立即开始工作。

### D9 — 调用方所有权（可静态检查）

| 接口 | 唯一调用方 | 静态扫描规则 |
|---|---|---|
| `request_context` | `SceneFlow` | 除 `scene_flow.gd` 外命中即失败 |
| `push_context(Modal)` / `pop_context()` | `UIRoot`（模态） | 以字符串字面量 `&"Modal"` 调用 `push_context` 的调用点只允许 `ui_root.gd` |
| `push_context(Blocked, allow)` / `pop_context()` | 首局引导 | 以 `&"Blocked"` 调用 `push_context` 的调用点只允许 `first_run_onboarding.gd` |
| `consume_action` | 首局引导 | 除 `first_run_onboarding.gd` 外命中即失败 |
| `is_action_just_pressed(&"ui_pause")` | `SceneFlow` | 除 `scene_flow.gd` 外命中即失败 |
| `is_action_just_pressed(&"ui_accept")` / `&"game_skip_wave"` / `&"ui_cancel"` | `HudHost`（提交 `RunFlow.submit_action`；`ui_cancel` 仅用于关闭结束取餐确认面板） | 只允许 `hud_host.gd`；`Menu` / `Modal` 的 `ui_cancel` 仍由 `UIRoot.handle_action` 消费 |
| 任何上下文赋值 | 无人 | 除 `input_router.gd` 内部外，`context =` / `_base =` / `_stack =` 命中 0 次 |

- **`SceneFlow` 不订阅 `context_changed`**：它是唯一 `request_context` 调用方，回执就是返回值。让它再订阅会形成「自己触发、自己响应」的回环，是重入的常见来源（ADR-0004 D7）。
- **`ui_pause` 只有 `SceneFlow` 一个读者**：因此不需要消费位，也不会与界面抢。`ui_pause` 同时是 `Blocked` 的恒允许动作，这是引导期间的唯一出口。

### Architecture Diagram

```
                    ┌──────────────────────────── InputRouter (Node) ───────────────────────────┐
  InputEvent ──────►│ _input: 忽略 echo → 识别类型 → 按 D3 允许表翻译为「至多一个动作」           │
  (kB/pad/mouse)    │         写 _pressed[action]；同帧同动作去重                               │
                    │                                                                          │
                    │ base + stack ──► effective ──► ALLOWED(ctx)                              │
                    │        ▲                              │                                  │
                    │        │ request_context(SceneFlow)   │                                  │
                    │        │ push/pop(UIRoot, Onboarding) │                                  │
                    │        │                              ▼                                  │
                    │   context_changed            ┌────────────────────────┐                  │
                    │   (from, to)                 │ 投递表（D4）            │                  │
                    │        │                     │ Menu/Modal/Paused      │──► ActionSink ───┼──► UIRoot.handle_action
                    │        ▼                     │   → sink.handle_action │    (返回 true 置消费位)
                    │   清空按住/重复/回滞/消费位   │ Blocked → 仅 consume   │                    │
                    │   (D7)                       │ Run     → 不投递       │◄── HUD 轮询 ───────┤
                    └──────────────────────────────┴────────────────────────┘                    │
                                                                                                │
  NavRepeater / StickGate (RefCounted, 纯函数) ──► 合成 InputEventAction ──► Viewport.push_input ─┘
```

### Key Interfaces

```gdscript
# res://src/foundation/input/input_router.gd
class_name InputRouter
extends Node

signal context_changed(from: int, to: int)
signal focus_owner_changed(path: NodePath)
signal hover_owner_changed(path: NodePath)

enum Ctx { BLOCKED, MENU, RUN, PAUSED, MODAL }
enum Reject { NONE, UNKNOWN_CONTEXT, UNKNOWN_ACTION, STACK_FULL }

const MAX_CONTEXT_DEPTH := 2

func attach(tuning: InputTuning, sink: ActionSink = null) -> void   # 构造期一次

func request_context(ctx: int, clear_stack: bool = true,
                     allow: Array[StringName] = []) -> bool
func push_context(ctx: int, allow: Array[StringName] = []) -> bool
func pop_context() -> bool

func consume_action(action: StringName) -> bool     # 仅 Blocked 生效；首个消费者 true
func is_action_just_pressed(action: StringName) -> bool

func get_focus_owner() -> NodePath
func get_hover_owner() -> NodePath
func get_context_snapshot() -> ContextSnapshot      # 只读；供无头断言与引导取 input_seq

func register_hover_target(control: Control) -> void
func unregister_hover_target(control: Control) -> void


class ContextSnapshot extends RefCounted:
    var base: int
    var stack: PackedInt32Array          # 只含 ctx，不含 allow
    var effective: int
    var allow: Array[StringName]         # 栈顶条目的白名单（非 Blocked 时为空）
    var input_seq: int
    var unconsumed: Array[StringName]    # 本帧仍可被 poll 的动作（只读副本）


# 界面动作的接收口；Presentation 侧提供适配器，Foundation 不依赖 UIRoot
class ActionSink extends RefCounted:
    func handle_action(action: StringName) -> bool   # true = 认领（含刻意空操作）

class UIRootActionSink extends ActionSink:           # Presentation 侧
    func _init(ui_root) -> void
    func handle_action(action: StringName) -> bool   # return ui_root.handle_action(action)


# 纯函数：导航重复与摇杆回滞（不引用 Node）
class NavRepeater extends RefCounted:
    func press(dir: StringName, now: float) -> int
    func tick(now: float) -> int
    func release(dir: StringName) -> void
    func reset() -> void

class StickGate extends RefCounted:
    func update(magnitude: float) -> bool
    func reset() -> void


# 输入时序参数（不属于四类平衡记录，不进 DataConfig 启动门）
class InputTuning extends Resource:
    @export var repeat_delay: float = 0.35
    @export var repeat_interval: float = 0.12
    @export var stick_engage: float = 0.60
    @export var stick_release: float = 0.40
    func is_valid() -> bool
```

### Implementation Guidelines

1. **必须**让 `push_context` 无条件加深并发信号（含同值），**禁止**把同值压栈实现成幂等空操作。
2. **必须**让 `pop_context` 在栈空时返回 `false` 且不弹基底；**禁止**用「弹到栈空即 `Blocked`」之类的兜底。
3. **必须**让 `consume_action` 只在 `Blocked` 生效，且与 `handle_action` 的返回值共享同一个「已消费」位；**禁止**为界面另建一套消费状态。
4. **必须**只在 `Menu` / `Modal` / `Paused` 投递 `ui_accept` / `ui_cancel` 给 `ActionSink`；**禁止**在 `Run` 与 `Blocked` 投递。
5. **必须**在产生 `ui_accept` / `ui_cancel` 时立即 `set_input_as_handled()`；**禁止**消费方向动作。
6. **必须**让 `ActionSink` 可为 `null`（无头测试）；为空时投递静默跳过，不报错。
7. **必须**在每次 `context_changed` 后清空 `_pressed` / `_consumed` / 重复计时 / 回滞 / 待发合成事件；**禁止**清空焦点与悬停。
8. **必须**在翻译阶段按 D3 表把 Esc 解析成**一个**动作；**禁止**让同一个物理事件同时出现在 `ui_pause` 与 `ui_cancel` 的动作集里。
9. **必须**用 `NavRepeater` / `StickGate` 计算重复与回滞，并从 `InputTuning` 读参数、做区间校验；**禁止**在其它文件里出现 `0.35` / `0.12` / `0.60` / `0.40` 这类时序字面量。
10. **必须**用 `input.set_meta(&"synthetic_repeat", true)` 标记合成事件并在 `_input` 中只放行不计数；**禁止**让合成事件再次触发回滞或重复。
11. **必须**按事件类型识别设备；**禁止**出现 `device == 0`，也禁止用 `DEVICE_ID_*` 做分类判断。
12. **必须**让 `MAX_CONTEXT_DEPTH = 2` 并拒绝第三层；**禁止**排队、替换或静默丢弃越界压栈。
13. **必须**让 `InputTuning` 缺失或越界时用默认值 + WARNING；**禁止**把它接入 `DataConfig` 的启动门，也禁止让输入在内容失败时不可用。
14. **必须**让 `first_run_onboarding` 在重压 `Blocked` 返回 `false` 时降级为无遮罩提示；**禁止**重试或把玩家留在「气泡在、按键无效」的状态。
15. **禁止**任何屏幕监听 `InputEvent`、调用 `consume_action`、或直接给上下文赋值（D9 表可静态扫描）。

## Alternatives Considered

### Alternative 1: 每个界面自己轮询 `is_action_just_pressed`

- **Description**：不设投递，`UIRoot` 与每个 `ScreenView` 在 `_process` 里查 `ui_accept` / `ui_cancel`。
- **Pros**：零新增接口；界面完全自治；无 sink 抽象。
- **Cons**：每个屏都要自己判断「本帧该不该响应」，`Menu` 与 `Modal` 同时轮询会双触发；`ui-flow.md` 的「每次已提交的激活恰好一次」变成每个屏各自的责任；`Run` 里 HUD 与残留屏幕会同时读到 `ui_accept`；无头测试要为每个屏模拟一帧。
- **Estimated Effort**：低（0.3×）。
- **Rejection Reason**：把「谁先消费」交给节点顺序，与 `QQ-13` 要解决的问题同构。

### Alternative 2: 用 `_unhandled_input` 链让屏幕依次处理

- **Description**：模态、屏幕、HUD 各在 `_unhandled_input` 里处理，命中即 `set_input_as_handled()`。
- **Pros**：利用引擎的传播机制；不需要显式栈；顺序由节点树决定。
- **Cons**：处理顺序 = 节点树顺序，而 ADR-0003 的宿主是 `CanvasLayer`，层与树的顺序并不一一对应；`Blocked` 需要「除白名单外全部丢弃」，而 `_unhandled_input` 是「没人处理才下传」，语义相反；无头测试要模拟整棵树的传播。
- **Estimated Effort**：中（1.0×）。
- **Rejection Reason**：传播语义与「上下文决定允许集」相反——本作是白名单，不是冒泡。

### Alternative 3: 所有上下文都用 `consume_action`（把界面也当消费者）

- **Description**：`consume_action` 在五种上下文都可用，`UIRoot` 在每个屏的 `_process` 里主动取。
- **Pros**：只有一套消费机制；与首局引导完全一致。
- **Cons**：`consume_action` 是「拉」，需要有人在正确的帧去拉；`UIRoot` 拉得太早会抢走 `Run` 里 HUD 的动作，拉得太晚会被 HUD 抢走——顺序问题又回来了；并且它把「界面存在」这件事变成隐式的（有屏才拉）；`input.md` 对它的定义明确限定在 `Blocked`。
- **Estimated Effort**：中（0.8×）。
- **Rejection Reason**：把推送改成拉取，只是把帧内顺序问题从 `InputRouter` 挪到 `UIRoot`，且违反 `input.md` 对 `consume_action` 的定义。

### Alternative 4: 依赖引擎按键回显实现导航重复

- **Description**：不自己计时，靠 `event.echo` 或 `Input.is_action_pressed` 的持续状态驱动。
- **Pros**：零计时代码。
- **Cons**：规则 5 明确忽略回显（回显间隔由 OS 决定，不可配、不可测）；`Input.is_action_pressed` 是电平不是边沿，每帧都会为真，一次按住会连跳；公式 `nav_steps` 的 0.35 / 0.12 与边界 `≤` 无法表达。
- **Estimated Effort**：低（0.2×）。
- **Rejection Reason**：`nav_steps` 是 GDD 的可判定公式（AC 6 逐点断言），引擎回显给不出这个精度。

### Alternative 5: `push_context` 同值时视为幂等空操作（QQ-06 的另一解）

- **Description**：栈顶已是 `Modal` 时 `push_context(Modal)` 直接返回 `true`、不加深、不发信号；相应地把模态深度 2 改成「不压第二层」。
- **Pros**：`context_changed` 永远是有效的语义变化，消费者不必处理 `from == to`；栈深与画面上的模态数不必一一对应。
- **Cons**：`ui-flow.md` 规则 5 与 `MAX_MODAL_DEPTH = 2` 明确允许两层模态（删除生涯确认在设置模态之上），若同值压栈不加深，两次 `pop_context` 会多弹一次基底并把上下文降级成 `Blocked`；`UIRoot` 的 `_modal_context_depth == modal_stack.depth()` 断言无法成立；`ADR-0003` 的模态深度 story 全部要改。
- **Estimated Effort**：低（改一行），但连带 ADR-0003 与 `ui-flow.md` 重写。
- **Rejection Reason**：与已 Accepted 的模态深度 2 直接冲突；`input.md` 规则 12 也已写明「`push_context` 总是增加栈深…包括同值压栈」。

## Consequences

### Positive

- **模态深度 2 与「每次按下恰好一次激活」可以同时成立**：同值压栈加深 + 单一投递路径 + 共享消费位，三者互不打架。
- **`from == to` 有明确约定**：监听者知道该忽略还是该记深度，不再靠猜。
- **`Run` 与 `Blocked` 的投递被显式排除**：HUD 与首局引导各自是唯一读者，不存在「界面把局内确认吃掉」的隐患。
- **`consume_action` 的范围被钉死**：它只服务 `Blocked`，因此可以静态扫描调用方，不会扩散成全项目通用的消费 API。
- **重复与回滞是纯函数**：AC 6 / AC 7 可以逐点断言，不需要真手柄、不需要真帧率。
- **`InputTuning` 不进启动门**：内容校验失败时 `ErrorScreen` 仍可导航，这是「输入必须永远可用」的直接推论。
- **清栈即清残留**：暂停、恢复、模态开关都不会带着按住的方向或半程重复跨过边界。

### Negative

- **`InputRouter` 变重**：它同时承担翻译、过滤、投递、重复生成、悬停登记与设备识别。逻辑被拆到纯函数与表里，但文件仍是 Foundation 里最大的一个。
- **`ActionSink` 多一层适配**：`InputRouter` 不能直接持有 `UIRoot`，需要 `UIRootActionSink` 这层薄壳。
- **`MAX_CONTEXT_DEPTH = 2` 是硬上限**：未来若要「暂停中打开设置、再打开删除生涯确认、同时还在引导」，就需要改本 ADR 而不是改常量。
- **合成事件依赖 `push_input` 的同步性**：若实测不同步，重复步会晚一帧且行为退化（见 Risks）。
- **`allow` 的词汇校验会拒绝含错别字的请求**：调用方必须处理 `false` 并降级，不能假设压栈一定成功。

### Neutral

- `get_context_snapshot()` 与 `attach()` 是本 ADR 在 `input.md` AC 11 的七个接口之外新增的两个；AC 11 的意图是「不泄漏 `InputEvent`」，新增的仍是纯查询与构造入口（记入 Open Questions 1）。
- `context_changed` 对同值也发一次，意味着音频若订阅它必须自己过滤 `from == to`；本作音频不订阅它（ADR-0004 D9），因此无实际成本。
- `ui_cancel` 在 `Run` 中虽然允许，但没有消费者，帧末自然作废；这不是漏洞，而是「Run 不投递」的必然结果。

## Risks

- **`Viewport.push_input(InputEventAction)` 不进入 GUI 焦点导航**（Verification Required ①）：重复步将无法移动焦点。→ 缓解：回退方案是 `InputRouter` 直接对当前焦点控件调用预解析邻居的 `grab_focus()`（ADR-0003 D5 已保证邻居指向可用控件），代价是设置页滑杆失去左右微调——届时滑杆改为「点击/确认进入调整态，左右改变数值」，属于 UI 侧改动，不影响本 ADR 的上下文与投递语义。
- **合成事件的同步性不成立**（Verification Required ②）：重复步晚一帧。→ 缓解：一帧的重复延迟在回合制里不可感知，但仍需在实测后把「≤ 1 帧」的措辞改为「≤ 2 帧」，并同步 `input.md`（记入 Open Questions 2）。
- **`MAX_CONTEXT_DEPTH = 2` 被将来的 UI 需求突破**：三层压栈会返回 `false` 并被 `UIRoot` 记 WARNING，表现为「模态打不开」。→ 缓解：`ui-flow.md` 已明确「不允许第三层模态」，本 ADR 与之一致；若产品确需第三层，改本 ADR 的常量并复核 `UIRoot` 的配对断言，而不是在调用处兜底。
- **`allow` 白名单与引导 beat 漂移**：`B0` 只放行 `ui_accept` / `ui_cancel`，若文案改动让玩家必须先导航，就会卡住。→ 缓解：`first_run_onboarding.md` AC 13/19 已把「导航是否到达局内」写成负向断言；本 ADR 的 `false` 返回让重压失败可见。
- **设备热插拔与按住状态**：手柄拔出时若未按「释放」处理，`NavRepeater` 会一直重复。→ 缓解：D8 要求拔出即按释放并 `reset()` 该设备的方向状态；`input.md` 边界用例与 AC 12 覆盖。
- **`InputTuning` 的两处来源**：资源与代码内默认值可能漂移。→ 缓解：默认值只允许出现在 `input_tuning.gd` 一处，且 `is_valid()` 是唯一校验入口；测试断言「缺资源时的行为等于默认值资源」。
- **`ui_pause` 的帧内优先级**（`process_priority = -100` / `-50` / `0`）：若 HUD 被改成更小的优先级，同帧暂停会晚于吃菜。→ 缓解：把三个优先级写成常量并在启动时断言 `-100 < -50 < 0`，而不是散落在各节点上。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `input.md` | TR-input-001：固定八个动作；任何系统不得监听原始事件 | D3 的允许表是唯一过滤源；D4 只投递动作；实现指南第 15 条禁止屏幕监听 `InputEvent` |
| `input.md` | TR-input-002：五上下文状态机；`request_context` / `push_context` / `pop_context` 的临时栈与清栈语义 | D1 的栈模型与返回值/信号表；D2 的同值压栈与配对不变量；D9 的调用方所有权 |
| `input.md` | TR-input-003：双焦点分开记录、分开绘制 | D8：`get_focus_owner` 是 `gui_get_focus_owner()` 的投影，`get_hover_owner` 由输入层维护；鼠标移动不改键盘焦点 |
| `input.md` | TR-input-004：全键盘/手柄可玩；导航重复与摇杆回滞 0.60 / 0.40 | D6：`NavRepeater` / `StickGate` 纯函数 + 同步合成投递；参数来自 `InputTuning` |
| `input.md` | TR-input-005：按事件类型识别设备，绝不比较 `device == 0` | D8：分类只看事件类型；`DEVICE_ID_*` 仅用于诊断；AC 9 的静态扫描写入 Validation Criteria |
| `input.md` | TR-input-006：`consume_action` 服务 `Blocked` 白名单；首个消费者获胜 | D4：共享消费位；D1/D3 的 `allow` 只作用于 `Blocked`；`consume_action` 在其它上下文返回 `false` |
| `input.md` | 规则 12 / AC 18–20：上下文只经三个接口改变；`request_context` 的返回值与信号；`Modal` 压弹 | D1 的表逐行给出返回值与信号；D9 的静态扫描规则；D2 的「顶层切换不弹栈」 |
| `ui-flow.md` | 规则 5 / Open Question 14：模态深度 2，栈顶已是 `Modal` 时再次压栈必须加深 | D2：`push_context` 无条件加深，`context_changed(Modal, Modal)`；深度上限 2 与 `_modal_context_depth` 配对 |
| `ui-flow.md` | TR-ui-flow-001 / 规则 27：屏幕栈与模态栈上限；界面只经 `push/pop` 改上下文 | D1 的 `MAX_CONTEXT_DEPTH` 与 D9 的调用方表；`request_context` 不对界面开放 |
| `ui-flow.md` | 规则 7 / 14：`ui_cancel` 由 `back_target` 唯一决定；收据滚动中第一次按下只跳过 | D4：投递到 `UIRoot.handle_action` 即消费，`UIRoot` 自己决定语义；输入层不解释 `ui_cancel` |
| `first-run-onboarding.md` | TR-first-run-onboarding-002：模态 beat 用 `Blocked` 白名单，一次输入只推进一个 beat | D3 的 `ui_pause` 恒允许 + `allow` 追加；D5 的 `input_seq`；D4 的 `consume_action` 单一调用方 |
| `first-run-onboarding.md` | AC 13 / 19 / 21：`ui_accept` 不到达局内；暂停后重压；同帧两次 `ui_accept` 只推进一次 | D3/D4（白名单与消费位）、D7（清栈清残留）、D5（`input_seq` 单调） |
| `scene-flow.md` | TR-scene-flow-002 / 规则 3：唯一 `request_context(default, clear_stack := true)` 调用方 | D9 的调用方表；ADR-0004 D8 固定其发生时机在 `state_changed` 发射之前 |
| `architecture.md` | QQ-06、QQ-13；原则 9（信号通知 / 直接调用契约） | D2 / D4 / D5 |
| `in-game-hud.md` | TR-in-game-hud-004：焦点解析与 `REJECTED(reason)` 反馈 | D4 的「`Run` 不投递、HUD 是唯一读者」；D8 的双焦点归属 |

## Performance Implications

- **CPU**：`_input` 每事件一次常数工作（类型判断 + 一次表查 + 一次字典写）。导航重复每帧最多一次整数运算（`NavRepeater.tick`），合成事件只在重复步发生时产生（最密 0.12 s 一次）。上下文切换只做数组清空。整条路径远低于单帧预算。
- **Memory**：上下文栈 ≤ 2 个条目；`_pressed` / `_consumed` 各 ≤ 8 项；悬停登记是每个可注册控件一条记录（一屏至多几十条）。
- **Latency**：事件到消费者 ≤ 1 帧（AC 15）；在 `_input` 内同步投递，实际是同一帧。
- **Allocations**：每帧两个小 `Dictionary` 的 `clear()`（不重新分配）；`ContextSnapshot` 只在被请求时构造（测试与引导），不在每帧热路径上。
- **Network**：无。

## Migration Plan

绿地实现，当前没有代码。落地顺序：

1. `InputRouter` 骨架：`base` / `stack` / 四个接口 / `context_changed`，先不接投递与输入。用纯 harness 覆盖 AC 18–20 与 D1 的表。
2. `ContextSnapshot` + `Reject` 枚举 + `MAX_CONTEXT_DEPTH`；补齐「栈空 `pop` 返回 `false`」「第三层拒绝」的用例。
3. 翻译层：`_input` 按类型识别设备、忽略 `echo`、按 D3 表翻译（含 Esc 的单动作解析）、同帧去重。
4. 投递层：`ActionSink` + 投递表 + 共享消费位；用记录器 sink 覆盖「`Run` 不投递」「`Blocked` 只 consume」「`handle_action` 返回 `true` 即消费」。
5. `NavRepeater` / `StickGate` / `InputTuning`：先纯函数单测（AC 6 / 7 逐点），再接合成投递并完成 Verification Required ①②。
6. 清残留（D7）、悬停登记（D8）、`input_seq`（D5）。
7. 接入 `UIRootActionSink` 与 `SceneFlow` 的 `ui_pause` 轮询；跑 `input.md` AC 1–20 与 `first-run-onboarding.md` AC 13/19/21。

**Rollback plan**：上下文栈、允许表、投递表都是内部结构，回滚方式是把 `InputRouter` 换成一份同名接口的实现，调用方不变。唯一有外部耦合的两点：①`ActionSink` 若被证伪（例如界面确实需要拉取），改成 `UIRoot` 直接轮询只需改本 ADR 与一行装配，`input.md` 的接口面不变；②合成重复若不通（Verification Required ①），改成直接 `grab_focus()` 到预解析邻居，代价是滑杆微调要改 UI 交互——这条已写入 Risks 并保留回退路径。

## Validation Criteria

- **同值压栈**：`Run` 中两次 `push_context(&"Modal")` → `context_changed` 依次为 `(Run, Modal)`、`(Modal, Modal)`，`depth == 2`；两次 `pop_context()` → `(Modal, Modal)`、`(Modal, Run)`；第三次 `pop_context()` 返回 `false`、不发信号、`effective` 仍为 `Run`。
- **幂等与非法输入**：`request_context(&"Menu", false)` 在 `effective == Menu` 时返回 `true` 且不发信号；`request_context(&"Bogus")` 返回 `false`、栈与基底不变、不发信号（AC 18）；`push_context(&"Modal", [&"bogus_action"])` 返回 `false` 且不变（D1）。
- **深度上限**：第三次压栈返回 `false`、记 ERROR、不发信号、`depth` 仍为 2。
- **顶层清栈**：`Run` + 一层 `Modal` 时 `request_context(&"Menu", true)` → 栈清空、`effective == Menu`、发一次 `context_changed(Modal, Menu)`；之后 `pop_context()` 返回 `false`（不清栈后二次弹栈）。
- **投递表**：五种上下文各一条用例——`Menu` 中 `ui_accept` 恰好一次到达 sink 且 `is_action_just_pressed` 随即为 `false`；`Run` 中 sink 调用 0 次而 HUD 轮询读到 `true`；`Blocked` 中 sink 调用 0 次、`consume_action` 第一次 `true` 第二次 `false`；`Paused` 中 `ui_accept` 到达 sink；`Modal` 中 `ui_pause` 的 `is_action_just_pressed` 恒为 `false`。
- **单动作解析**：`Run` 中按 Esc → `ui_pause` 计数 1、`ui_cancel` 计数 0；`Modal` 中按 Esc → 反（AC 2）；同帧按手柄 A 与 Enter → 动作计数 1（AC 14）。
- **`consume_action` 范围**：`Run` 中调用 → 返回 `false`、不置位、WARNING 一帧一次；`Blocked` 中先 `consume_action` 再 `is_action_just_pressed` → 后者 `false`。
- **导航重复**：`D = 0.35`、`I = 0.12` 时 `T = 1.00` → `NavRepeater` 累计 7 步、触发点 0.35 / 0.47 / 0.59 / 0.71 / 0.83 / 0.95；`T = 0.35` → 2 步；`T = 0.34` → 1 步（AC 6）。
- **回滞**：`(false, 0.55) → false`、`(true, 0.55) → true`、`(true, 0.35) → false`、`(true, 0.40) → true`、`(false, 0.60) → true`（AC 7）。
- **切换清残留**：按住方向 → 切换上下文再切回 → 不产生导航，必须松开重按（AC 13）；切换后 `unconsumed` 为空。
- **设备识别**：合成 `device = 0` 的 `InputEventKey` 与 `InputEventMouseMotion` → 按类型判为键盘/鼠标；`device = 0` 的手柄事件不误判；源码 `device ==` 命中 0 次（AC 9）。
- **接口面**：运行时公开接口为 `is_action_just_pressed` / `get_focus_owner` / `get_hover_owner` / `request_context` / `push_context` / `pop_context` / `consume_action`，加上构造期的 `attach()` 与只读的 `get_context_snapshot()`；`InputEvent` 引用只出现在输入层（AC 11，按 Open Questions 1 的说明读）。
- **禁止赋值**：全仓库静态扫描 `context =` / `_base =` / `_stack =` 只命中 `input_router.gd`；`request_context` 调用点只命中 `scene_flow.gd`（AC 20）。
- **引导契约**：`Blocked` 下 `ui_accept` 被引导 `consume_action` 后，`RunFlow.submit_action` 调用 0 次；`ui_pause` 在 `allow` 为空时仍可触发暂停；重压 `push_context` 返回 `false` 时引导降级为无遮罩提示且玩家可正常操作（AC 13 / 19 / 21）。
- **时序参数**：缺失 `input_tuning.tres` → 行为等于默认值资源、启动不阻断、WARNING 一次；`D = 0.7` → 采用默认值并 WARNING。

## Related

- `docs/architecture/architecture.md` — 原则 9、Module Ownership（Input 行）、Required ADR #5、Open Questions QQ-06 / QQ-13
- `docs/architecture/adr-0003-scene-screen-architecture.md` — D4（模态栈与 `_modal_context_depth`）、D5（显式邻居、输入只提供节奏）、D9（动作唯一入口、`set_input_as_handled`）、D10（重建不弹栈）
- `docs/architecture/adr-0004-service-composition-signals.md` — `InputRouter` 由组合根构造并注入；`state_changed` 扇出前已完成 `request_context`；`ui_pause` 由 `SceneFlow` 独占；`ActionSink` 的方向约束
- `design/gdd/input.md` — 规则 1–12、状态表、`nav_steps` / `stick_engaged` 公式、AC 1–20、Edge Cases（上下文切换、设备、回显）
- `design/gdd/ui-flow.md` — 规则 5 / 7 / 14 / 15 / 27、状态与转移表、Open Question 14、同帧冲突
- `design/gdd/scene-flow.md` — 规则 2 / 3 / 7 / 15、`state_latency_frames = 0`
- `design/gdd/first-run-onboarding.md` — 规则 6 / 7 / 10 / 16 / 17、`beat_next` 公式、AC 13 / 19 / 21
- `docs/engine-reference/godot/VERSION.md`、`modules/input.md`、`modules/ui.md`、`current-best-practices.md`（4.6 双焦点、4.7 `DEVICE_ID_*`）、`deprecated-apis.md`（`device == 0` 已列废弃模式）

## Open Questions

1. **`input.md` AC 11 的接口清单需要回改**：AC 11 断言「暴露的接口恰好是七个」。本 ADR 增加 `attach()`（构造期一次）与 `get_context_snapshot()`（只读，供无头断言与引导取 `input_seq`）。两者都不泄漏 `InputEvent`，但按字面会让 AC 11 失败。建议把 AC 11 改为「运行时动作与上下文接口恰为七个，另有构造入口 `attach` 与只读快照 `get_context_snapshot`」——**本 ADR 不修改任何 GDD**，由编排会话统一处理。
2. **`Viewport.push_input` 的同步性**（Verification Required ①②）：若合成重复事件实际在下一帧才被处理，`input.md` AC 15 的「≤ 1 帧」需要放宽到 2 帧，并把该放宽写回 GDD。
3. **`MAX_CONTEXT_DEPTH = 2` 是否足够**：若 Full Vision 需要「暂停 → 设置 → 删除生涯确认」与「引导 `Blocked`」同时存在（当前不可能，因为引导只在 `Run`、模态只在有屏状态），上限要提到 3。提到 3 之前必须先证明第三层的离开路径唯一。
4. **重绑（`input.md` Open Question 1）会改变什么**：重绑替换的是映射表，动作集不变；但 `InputTuning` 的时序参数与重绑界面本身需要 `Blocked` 或 `Modal` 上下文——届时本 ADR 的 D9 表要新增一个 `push_context` 调用方。
5. **鼠标右键与 `game_skip_wave` 的悬停关系**：`input.md` 规定「鼠标未悬停菜品卡时右键仍触发跳过」。本 ADR 不消费该动作，但它在 `Run` 中是唯一「不依赖焦点」的动作；若将来 HUD 要求「右键只在游戏区域内生效」，需要在 Input 层加一个矩形门，属于新裁定。
6. **`Blocked` 用于「结算动画」的路径**：`input.md` 状态表提到 `Blocked` 也用于结算动画，但当前没有任何系统为结算压 `Blocked`（收据滚动用 `ui_accept` 跳过）。若将来要压，必须先确认它与 `SceneFlow` 的 `clear_stack` 以及 `UIRoot` 的收据滚动互不冲突。
