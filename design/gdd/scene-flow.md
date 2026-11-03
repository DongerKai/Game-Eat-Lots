# 场景与流程管理（Scene & Flow）

> **Status**: In Design
> **Author**: user + agents
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 时间在走，手不用快；数字看得见地涨

## Summary

场景与流程管理是《回本》的应用级状态机：它启动游戏、等待数据校验、在菜单与一局之间做原子切换、处理暂停与退出。它保证任何时刻只有一个世界场景处于活动状态，且输入上下文始终与当前世界一致。玩家看不到这个系统，但每一次「开始一局」和「回到菜单」都要经过它。

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `数据配置`, `输入`

## Overview

场景与流程管理把「游戏现在处于哪个世界」变成唯一、可判定的事实。它持有七个状态——Boot、Loading、Menu、Run、Transition、Error、Quitting——其中只有 Menu 与 Run 是玩家能操作的世界场景，其余是过场或终态。

启动时它先让数据配置完成加载与校验，只有拿到 Ready 才实例化主菜单；校验失败则停在 Error，主场景从未被创建。菜单与一局之间不做加载条：先隐藏并冻结旧世界、挂载新世界、再释放旧世界，中间用一个约 250 毫秒的淡入淡出盖住。任一步失败，旧世界原样留在屏幕上，玩家不会掉进空场景。

玩家不直接接触这个系统。它存在的意义是让「一局能完整重复」不依赖任何具体界面或局内逻辑——界面流程可以整块重写，餐局流程可以改规则，而「开始一局 / 回到菜单 / 退出」始终只有一条路径。

## Detailed Design

### Core Rules

1. **唯一活动世界**：Menu 与 Run 合称「世界场景」。任何时刻最多一个世界场景处于活动状态；在 Menu 或 Run 状态下恰好一个。切换时旧世界先被隐藏并冻结，再挂载新世界，最后释放旧世界——因此不会出现两个活动世界，也不会出现零个活动世界。
2. **状态是唯一事实**：当前状态由本系统持有。其他系统只能查询 `current_state()`，不得自行推导、缓存或从场景树反推。状态变更只通过 `flow_changed(from, to)` 信号广播。
3. **启动顺序固定**：Boot → 等待数据配置的启动加载与校验 → Loading → 其状态进入 Ready 则实例化主菜单进入 Menu；进入 Failed 则进入 Error。数据配置的规则 7 已规定「启动即校验」，本系统只等待与读取状态，不另行触发加载。Boot 阶段不实例化任何世界场景。
4. **Error 是终态**：数据校验失败后，本次进程不再尝试加载。Error 界面只显示校验报告（面向开发者：文件路径、记录 id、字段、期望值与实际值）和一个「退出」操作。修复数据后需重启游戏。
5. **切换是原子的**：一次切换 = 淡出 → 隐藏并冻结旧世界 → 在后台线程加载目标世界 → 挂载 → 淡入 → 释放旧世界。目标世界加载失败时，旧世界解除冻结、保持挂载，状态回到切换前，切换返回 `false`。目标世界必须在后台线程加载，否则淡出与加载指示器都无法在主线程被绘制——这是规则 5 与 Formulas 中两个公式共同的前提。
6. **切换期间拒绝新请求**：Transition 状态下的 `start_run` / `end_run` 一律返回 `false` 且不改变状态、不排队。退出请求例外——它被记录，在当前切换完成后最多晚一帧执行。理由：排队会把一次误触变成一次多余的场景重建。
7. **输入上下文随世界切换**：进入 Menu 调用 `request_context(&"Menu")`，进入 Run 调用 `request_context(&"Run")`，进入 Boot / Loading / Transition / Error 调用 `request_context(&"Blocked")`。上下文状态机归输入所有（输入 GDD 规则 12），本系统只是唯一请求这三种上下文的调用方；弹窗的 `Modal` 由界面流程请求，暂停的 `Paused` 由暂停操作请求。
8. **暂停只属于 Run**：暂停由输入上下文 `Paused` 表达。本系统监听 `context_changed`：上下文进入 `Paused` 时冻结局内世界，离开时解冻。触发来源有两个——玩家按 `ui_pause`，以及窗口失焦或手柄拔出（后两者由输入 GDD 规定为自动暂停）。`ui_pause` 开关有 0.20 秒冷却，一次按键只切换一次。Menu、Loading、Transition、Error 中不存在暂停；这些状态下失焦不改变任何东西。
9. **局外不保存局内**：MVP 不存在「继续上一局」。局内状态不落盘，进程退出即放弃本局。本系统不提供 `resume_run`。
10. **退出语义**：
    - Menu 中退出：立即进入 Quitting，无确认。
    - Run 中退出：先弹确认框（「退出会放弃本局收益」）。确认后本局被放弃——不进入结算、未入账收益归零；取消则回到原局。确认框在打开的那一帧忽略 `ui_accept`，避免同一次按键既打开又确认。
    - Loading / Transition / Error 中退出：立即进入 Quitting。
11. **场景引用是代码常量**：主菜单场景与局内场景的路径是本系统的常量，不写进数据配置——它们是程序结构，不是平衡数值。
12. **不做规则校验**：本系统不判断入场费是否够、胃容量是否已满、一局是否该结束。它把调用方视为已确认可执行，只负责世界切换。规则判断全部留在发起请求的下游系统。

### States and Transitions

| 状态 | 进入条件 | 允许的操作 | 离开条件 |
|---|---|---|---|
| Boot | 进程启动 | 退出 | 启动序列发起数据加载 → Loading |
| Loading | 数据配置校验进行中 | 退出 | 其状态为 Ready → Menu；为 Failed → Error |
| Error | 数据校验失败 | 退出 | —（终态） |
| Menu | 主菜单世界已挂载且活动 | `start_run`、退出 | `start_run` → Transition（目标 Menu→Run） |
| Run | 局内世界已挂载且活动 | 暂停 / 恢复、`end_run`、退出 | `end_run` → Transition（目标 Run→Menu） |
| Transition | 收到切换请求且目标合法 | 退出（延迟执行） | 切换成功 → 目标状态；切换失败 → 原状态 |
| Quitting | 退出请求被确认 | — | 进程退出 |

**转换表**

| 从 | 事件 | 到 | 副作用 |
|---|---|---|---|
| Boot | 启动序列 | Loading | 请求数据配置加载与校验；输入上下文 `Blocked` |
| Loading | 数据 Ready | Menu | 实例化主菜单并挂载；输入上下文 `Menu` |
| Loading | 数据 Failed | Error | 不实例化任何世界场景；输入上下文 `Blocked` |
| Menu | `start_run()` | Transition(Run) | 输入上下文 `Blocked`；淡出开始 |
| Run | `end_run(entry_screen)` | Transition(Menu) | 输入上下文 `Blocked`；淡出开始；记录目标入口屏 |
| Transition | 目标实例化成功 | Menu / Run | 隐藏并冻结旧世界 → 挂载新世界 → 淡入 → 释放旧世界 → 设置输入上下文 → 广播 `flow_changed` |
| Transition | 目标实例化失败 | 原状态 | 解除旧世界冻结；记录错误；切换返回 `false` |
| Menu / Run / Error | 退出请求（Run 需确认） | Quitting | 释放世界场景；退出进程 |
| 任意 | 进程被操作系统终止 | — | 不做清理，不做保存 |

**暂停不是状态机的一个状态**：暂停发生在 Run 内部，由输入上下文 `Paused` 与局内场景的冻结共同表达。本系统的 `current_state()` 在暂停期间仍返回 `Run`——对流程而言，一局仍在进行。

**暂停/恢复转换**

| 从 | 事件 | 到 | 副作用 |
|---|---|---|---|
| Run（未暂停） | `ui_pause`，或窗口失焦 / 手柄拔出 | Run（暂停中） | 输入上下文 `Paused`；本系统收到 `context_changed` 后冻结局内场景（`process_mode` 置为禁用） |
| Run（暂停中） | `ui_pause` 或 `ui_accept` | Run（未暂停） | 输入上下文 `Run`；本系统收到 `context_changed` 后解冻局内场景 |
| Run（暂停中） | 暂停菜单请求退出 | Quitting | 视为 Run 中退出，已确认过 |

### Interactions with Other Systems

| 系统 | 方向 | 数据 / 接口 | 硬/软 |
|---|---|---|---|
| 数据配置 | 上游 | 读取其启动校验状态 Ready / Failed | 硬 |
| 输入 | 上游 | `request_context(Blocked / Menu / Run)` 请求切换；监听 `context_changed(from, to)` 以冻结/解冻局内世界 | 硬 |
| 餐局流程 | 下游 | 接收 `run_started()`；调用 `end_run(entry_screen)`；查询 `current_state()` | 硬 |
| 界面流程 | 下游 | 调用 `start_run()`；接收 `menu_entered(entry_screen)`；在 Menu 内自行管理各屏 | 硬 |
| 局内 HUD | 无直接接口 | 活在 Run 世界内部，不经过本系统 | — |
| 存档 | 无接口 | MVP 不读取、不写入 | — |

**对外接口（本系统拥有）**

| 接口 | 调用方 | 说明 |
|---|---|---|
| `start_run() -> bool` | 界面流程 | 请求从 Menu 进入一局。成功返回 `true`；在非 Menu 状态或 Transition 中返回 `false`。 |
| `end_run(entry_screen := &"settlement") -> bool` | 餐局流程 | 请求从 Run 回到 Menu，并指定进入哪个屏。`entry_screen` 由界面流程解释，本系统只透传。 |
| `request_quit(confirmed := false)` | 界面流程 / 暂停菜单 | Menu 中直接退出；Run 中未确认时只打开确认框。 |
| `current_state() -> FlowState` | 全部下游 | 返回七个状态之一。 |
| 信号 `flow_changed(from, to)` | 全部下游 | 每次状态变更广播一次，用于 HUD、音效等需要同步的模块。 |

**接口所有权**：本系统拥有流程状态、世界场景的挂载与释放，以及「何时请求哪个输入上下文」的决定权——但上下文本身归输入所有，本系统只调用 `request_context()`。下游不直接 `change_scene`、不直接 `add_child` 世界场景、不直接请求 `Blocked` / `Menu` / `Run`。界面流程拥有 Menu 内部各屏之间的导航；餐局流程拥有 Run 内部的一切。

**边界说明**：`end_run` 不携带本局收益。收益在请求切换之前已由餐局流程交给结算与入场费，本系统不接触任何经济数据——这保证流程系统永远不会因为数值改动而需要修改。

## Formulas

**前提**：目标世界在后台线程加载（规则 5）。若同步加载，主线程会被阻塞，下表的时间量无法被测量，加载指示器也无法被绘制。

**这些常量与场景路径放在一起，是代码常量，不进数据配置（规则 11）。** 它们是表现时序，不是平衡数值——改它们不需要重跑一次平衡验证。

### 切换总时长

The transition_total formula is defined as:

`transition_total = FADE_OUT + load_time + FADE_IN`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| FADE_OUT | float (s) | 0.08–0.20（当前 0.12） | constant | 淡出时长 |
| load_time | float (s) | 0.005–5.0（实测，无上界） | measured | 目标世界后台加载到挂载完成；**不含**旧世界的 `queue_free` |
| FADE_IN | float (s) | 0.08–0.20（当前 0.12） | constant | 淡入时长 |

**Output Range:** 最小值 0.121 s（0.08 + 0.005 + 0.08）；正常桌面机上约 0.25 s。**这不是钳制**——设计目标是 ≤ 0.40 s，超过就说明读盘慢，切换照常完成，只是更久，并由下面的看门狗暴露出来。

`load_time` 的起止必须按上表定义：从发起后台加载起算，到目标世界挂载完成止。旧世界的 `queue_free` 发生在淡入开始之后，不计入——否则两个人会量出两个数。

**Example:** FADE_OUT = 0.12、load_time = 0.01、FADE_IN = 0.12 → `transition_total = 0.25 s`。

### 加载指示器

指示器不能在加载结束后才判断该不该显示——那时已经不需要它了。它是一个看门狗：

The shows_loading_indicator formula is defined as:

`shows_loading_indicator = (elapsed_since_fade_out_end >= LOADING_INDICATOR_THRESHOLD) and (not world_mounted)`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| elapsed_since_fade_out_end | float (s) | 0.0–5.0 | measured | 自淡出结束起的墙上时间 |
| world_mounted | bool | 0 / 1 | state | 目标世界是否已挂载 |
| LOADING_INDICATOR_THRESHOLD | float (s) | 0.20–1.00（当前 0.40） | constant | 淡出结束后多久仍未挂载就显示指示器 |

**Output Range:** 0 或 1。正常开发机上恒为 0；冷启动、慢盘或首次加载新场景时为 1。

**一旦为 1 就保持到遮罩完全清除**，不设最短显示时长——否则会出现「出现—消失—再出现」的二次闪烁。

**Example:** 淡出结束时 t = 0；t = 0.45 s 时目标世界仍未挂载，阈值 0.40 s → `shows_loading_indicator = 1`；t = 0.52 s 挂载完成，指示器继续显示到淡入结束。

### 加载超时

The switch_aborted formula is defined as:

`switch_aborted = elapsed_since_load_start > LOAD_TIMEOUT`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| elapsed_since_load_start | float (s) | 0.0–30.0 | measured | 自发起后台加载起的墙上时间 |
| LOAD_TIMEOUT | float (s) | 3.0–15.0（当前 5.0） | constant | 超过此时长仍未挂载即放弃本次切换 |

**Output Range:** 0 或 1。

**Example:** 磁盘卡住，加载 5.3 s 仍未完成 → `switch_aborted = 1`：放弃本次切换，旧世界解冻并保持挂载，切换返回 `false`，错误进日志。**不进入 Error**——Error 是数据校验失败的终态，加载超时是一次可恢复的切换失败。

### 启动阶段例外

Boot 与首次 Loading 没有可遮蔽的旧世界，且冷启动包含着色器编译（实测 0.3–2 s）。因此启动阶段的加载指示器**不套用上面的阈值，从 t = 0 起无条件显示**。

## Edge Cases

- **If 切换请求发生在 Transition 中**：返回 `false`，状态不变，不排队。当前切换照常完成。理由：排队会把一次误触变成一次多余的场景重建，而玩家看到的是两次淡出。
- **If 退出请求发生在 Transition 中**：标记为「待执行」；当前切换完成后最多晚一帧进入 Quitting，不再淡入、不再释放旧世界（进程即将结束，释放没有意义）。
- **If 目标世界加载或实例化失败**（场景文件缺失、脚本 `_ready` 抛错、依赖缺失、超过 `LOAD_TIMEOUT`）：旧世界解除冻结、保持挂载与可见，状态回到切换前的 Menu 或 Run，切换返回 `false`，错误写入日志；下一次成功进入 Menu 时在屏幕角落提示一次「上次切换失败」，不弹窗、不打断。
- **If 数据校验返回 Failed**：进入 Error，主菜单从未被实例化。这与数据配置 GDD 的验收条件「Failed 时主场景未被实例化」是同一条事实，两处必须一致。
- **If 玩家在 Loading 中请求退出**：退出优先，不再等待校验结果。即使 Ready 随后到达，也不实例化任何世界场景，进程直接结束。
- **If Run 中重复收到 `end_run`**：第一次之后状态已不是 Run，后续调用返回 `false`，只发生一次切换。
- **If Menu 中收到 `start_run` 但一局已在运行**：状态为 Run，返回 `false` 并记一条 WARNING——这是调用方 bug，界面流程不应在 Run 中渲染开始按钮。
- **If Error 状态下收到 `start_run` 或 `end_run`**：返回 `false`，状态保持 Error。Error 不接受任何进入游戏的请求。
- **If `ui_pause` 在一帧内被重复上报**：暂停是幂等的。重复进入 Paused 不产生额外的冻结或恢复，也不叠加暂停菜单。
- **If 暂停确认框已打开时再次请求退出**：确认框保持打开，不叠加第二个，也不重置已选项。
- **If Run 中窗口失焦或被最小化**：输入请求进入 `Paused`，本系统随 `context_changed` 冻结局内场景；局面完整保留，切回后按 `ui_pause` 或 `ui_accept` 恢复。这与输入 GDD 的 Edge Case「窗口失去焦点 → 局内自动进入 Paused」是同一条行为。
- **If 操作系统直接终止进程**（强制退出、断电）：不清理、不保存，本局丢失。这是已声明的代价（规则 9），不是缺陷。
- **If 主菜单在 Menu 状态下被意外释放**（代码 bug）：本系统检测到活动世界为空时，立即重建主菜单并记 ERROR 日志；不进入 Run，也不停在空场景。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 数据配置 | 硬 | 读取启动校验状态 Ready / Failed（其规则 7「启动即校验」） | GDD 已写 |
| 输入 | 硬 | `request_context(Blocked / Menu / Run)`；`context_changed(from, to)` 信号 | GDD 已写 |

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 餐局流程 | 硬 | `run_started()`；`end_run(entry_screen)`；`current_state()` | GDD 未写 |
| 界面流程 | 硬 | `start_run()`；`menu_entered(entry_screen)`；`request_quit()` | GDD 未写 |

**间接相关（无接口）：** 局内 HUD 活在 Run 世界内部；存档在 MVP 不与局内状态往来。两者都不经过本系统。

**双向一致状态**：已核对完成。`design/gdd/data-config.md` 与 `design/gdd/input.md` 的 Interactions / Dependencies 都已补上「场景与流程管理」的反向条目，`design/gdd/systems-index.md` 第 3 行的「Depends On」已补 `数据配置, 输入`。

**本次补写的内容**（`/design-system` 之后）：
- 输入 GDD 新增规则 12「上下文切换由请求驱动」，定义 `request_context(context) -> bool` 与 `context_changed(from, to)`，并列出允许的请求方与目标。
- 输入 GDD 新增验收条件 18–20。
- 输入 GDD 的 Cross-References 由「None」改为引用本 GDD 的共用契约。

## Visual/Audio Requirements

- **淡入淡出**：一层纯色遮罩，颜色为米白纸底（与价目板同色），不做缩放、不做模糊、不做位移，曲线线性。淡出 120 ms、淡入 120 ms。支柱 4「数字看得见地涨」要求过渡不能模糊读数，因此遮罩是全不透明，不是半透明压暗。
- **加载指示器**：超过阈值才出现，内容为三个静态圆点加「载入中」，不动画、不闪烁、不转圈。它出现本身就意味着这台机器读盘慢，不做美化，也不假装轻松。
- **Error 屏**：等宽字体、左对齐的校验报告文本，可直接截图发给开发者。顶部一条「回本红」分隔线，没有插画、没有图标。
- **无音效**：本系统不播放任何声音。切换若需要音效，由音效反馈系统在新世界 `_ready` 之后自行播放，本系统不介入。

> 📌 **Asset Spec** — Visual/Audio requirements are defined. After the art bible is approved, run `/asset-spec system:scene-flow` to produce per-asset visual descriptions, dimensions, and generation prompts from this section.

## Game Feel

- **切换要快**：菜单与一局之间的完整过渡目标 250 ms，上限 400 ms。超过 400 ms 玩家会感到「卡了一下」，而不是「过渡了一下」。
- **切换要稳**：淡出开始后不再接受新的切换请求。玩家永远不会看到两次淡出叠在一起，也不会在淡出中看到两个世界。
- **不跳过**：即使目标世界 0 ms 就绪，也走完 120 ms 淡出与 120 ms 淡入。「加载快就跳过过渡」听上去是优化，实际效果是闪烁。
- **退出有重量**：Run 中退出必须先确认，Menu 中退出直接走。两种节奏的差别让玩家意识到这一局是有代价的——这与支柱 2「没有白吃的菜」同源。
- **失败不惊动玩家**：切换失败时旧世界原样保留，玩家看到的是「按了没反应」，而不是掉进黑屏。错误只进日志，下次回到菜单时角落提示一次。

## UI Requirements

- **启动**：Boot 阶段只显示空白纸底，不显示 logo 动画、不显示进度条、不显示版本号。
- **Error 屏**：由本系统自己生成，不经过界面流程。包含标题「数据校验失败」、可滚动的等宽报告文本、一个「退出」按钮。该屏是 MVP 唯一的开发者面向界面。
- **加载指示器**：屏幕居中，绘制在淡出遮罩之上，不阻塞淡出与淡入的时序。
- **退出确认框**：Run 中触发。文案「退出会放弃本局收益」，两个按钮「继续这一局」/「退出」。默认焦点在「继续这一局」，`ui_cancel` 等同取消。它属于 Modal 输入上下文，由暂停菜单承载。
- **不渲染其他内容**：本系统不渲染菜单、不渲染 HUD、不渲染结算屏。这些全部归界面流程与局内 HUD。

> **📌 UX Flag — 场景与流程管理**：This system has UI requirements. In Phase 4 (Pre-Production), run `/ux-design` to create a UX spec for each screen or HUD element this system contributes to **before** writing epics. Stories that reference UI should cite `design/ux/[screen].md`, not the GDD directly.
>
> Screens contributed: 启动纸底、Error 屏、加载指示器、退出确认框。

## Cross-References

| 引用 | 在本 GDD 的位置 | 引用的内容 |
|---|---|---|
| 数据配置 | Detailed Design 规则 3、States and Transitions、Edge Cases、Dependencies | 其规则 7「启动即校验」与 Ready / Failed 状态的契约；「Failed 时主场景未被实例化」必须与数据配置 GDD 验收条件第 2 条一致 |
| 输入 | Detailed Design 规则 7、8、States and Transitions（暂停）、Dependencies | 上下文枚举 `Blocked / Menu / Run / Paused / Modal`；`request_context()` 与 `context_changed()` 的契约；失焦与手柄拔出会自动进入 `Paused`，本系统只响应不发起 |
| 界面流程 | Detailed Design（Interactions）、States and Transitions、Dependencies | 承载 Menu 内的全部屏幕；`start_run()` / `menu_entered(entry_screen)` 的调用方向 |
| 餐局流程 | Detailed Design（Interactions）、Dependencies | `run_started()` / `end_run(entry_screen)` 的调用方向；收益数据不经过本系统 |

> 三份 GDD 的反向引用已互相补全（见 Dependencies 的双向一致状态）。

## Acceptance Criteria

**测试夹具**（一次定义，下文引用）：

- **F1 主菜单占位场景** — 加入 `world_scene` 组，向日志通道 `L` 写 `_ready` / `_exit_tree` 与时间戳。
- **F2 局内占位场景** — 同上。
- **F3 场景探针** — 挂在 `/root`，每帧记录 `world_scene` 组中的节点数、各自的 `process_mode`，以及每次淡入/淡出的起止时刻。
- **F4 坏数据夹具** — 数据配置 GDD 中使校验失败的夹具，复用同一份。
- **F5 实例化失败场景** — `_ready` 主动抛错的场景，同样加入 `world_scene` 组。
- **F6 输入上下文探针** — 记录每次上下文变更及其请求来源。
- **F7 可控慢加载场景** — 后台加载延时可在 0–5 s 之间设定。
- **F8 退出钩子注入器** — 替换进程退出调用，记录调用次数与时刻，不真正退出。

> **术语**：下文「已挂载」一律指「在场景树中且 `process_mode != DISABLED`」。只有这两个条件同时成立才算活动世界，否则「挂载」在不同实现下会量出不同结果。

### 世界与状态

1. **GIVEN** F1、F2、F3，**WHEN** 跑完一次 Menu → Run → Menu，**THEN** F3 的每一个采样点中，`world_scene` 组里「已挂载」的节点数均为 0 或 1；且在状态为 Menu 或 Run 且不在 Transition 时恰为 1。
2. **GIVEN** F1、F2、F3，**WHEN** 一次 Menu → Run 切换执行到「淡出结束、新世界尚未挂载」这一刻，**THEN** 旧世界已不在 `world_scene` 的活动集合中（`process_mode == DISABLED`）。
3. **GIVEN** F1、F2，**WHEN** 调用 `start_run`，**THEN** `flow_changed` 恰好发出两次——`(Menu, Transition)` 与 `(Transition, Run)`——且每次状态变更恰好一次，不多不少。
4. **GIVEN** F1、F2、正常数据，**WHEN** 启动，**THEN** 状态序列为 Boot → Loading → Menu，F1 在状态进入 Menu 之后才出现在日志通道 `L`，Boot 期间 `world_scene` 组为空。
5. **GIVEN** F1、F2、F4，**WHEN** 启动，**THEN** 状态序列为 Boot → Loading → Error，且日志通道 `L` 中不存在 F1 的 `_ready`。
6. **GIVEN** 状态 Error，**WHEN** 观察 5 秒并调用 `start_run`、`end_run`，**THEN** 两者均返回 `false`，状态保持 Error，`world_scene` 组为空，且 Error 屏上可操作的控件只有一个。 
7. **GIVEN** 状态 Run，**WHEN** 连续调用 `end_run` 两次，**THEN** 第二次返回 `false`，F3 记录的淡出次数为 1。

### 切换的原子性

8. **GIVEN** Menu 且 F1 已挂载，**WHEN** `start_run` 的目标为 F5，**THEN** 返回 `false`，`current_state()` 仍为 Menu，F1 仍在树中且 `process_mode != DISABLED`，日志通道 `L` 中无 F1 的 `_exit_tree`。
9. **GIVEN** 同上但目标是 F2（成功路径），**WHEN** 切换完成，**THEN** 日志通道 `L` 中 F1 的 `_exit_tree` 恰好出现一次。**第 8、9 条必须同时通过**——只测失败路径的「未释放」时，一个从不释放任何世界的实现也会通过。
10. **GIVEN** 一次 Transition 正在进行，**WHEN** 分别调用 `start_run` 与 `end_run`，**THEN** 两者都返回 `false`，F3 记录的淡出次数仍为 1。
11. **GIVEN** F1、F2、F3，**WHEN** 完成一次 Menu → Run → Menu，**THEN** F3 的顺序为「淡出 → 挂载新世界 → 淡入 → 释放旧世界」，且旧世界的 `_exit_tree` 时间戳晚于新世界的 `_ready` 时间戳。
12. **GIVEN** F2 曾被挂载并释放一次，**WHEN** 再次 `start_run`，**THEN** 日志通道 `L` 中 F2 的 `_ready` 第二次出现，且没有对已释放实例的访问错误（`is_instance_valid` 全程为真）。

### 输入上下文

13. **GIVEN** F6，**WHEN** 依次进入 Boot、Loading、Menu、Run、Transition、Error，**THEN** 上下文依次为 `Blocked`、`Blocked`、`Menu`、`Run`、`Blocked`、`Blocked`。
14. **GIVEN** 全仓库静态扫描，**THEN** 除本系统外没有其他脚本请求 `Menu` / `Run` / `Blocked` 上下文（`Modal` 与 `Paused` 由界面流程与暂停操作请求）。

### 暂停与退出

15. **GIVEN** 状态 Run，**WHEN** 按下 `ui_pause`，**THEN** 上下文为 `Paused` 且 F2 的 `process_mode == DISABLED`；**WHEN** 再按 `ui_accept`，**THEN** 上下文回到 `Run` 且 F2 的 `process_mode != DISABLED`。
16. **GIVEN** 状态 Menu，**WHEN** 按下 `ui_pause`，**THEN** 上下文与状态均不变。
17. **GIVEN** 状态 Run，**WHEN** 在 0.20 秒内连按三次 `ui_pause`，**THEN** 只发生一次暂停切换——第 1 次生效，第 2、3 次被冷却吞掉，最终上下文为 `Paused`。
18. **GIVEN** 状态 Run，**WHEN** 窗口失焦，**THEN** 上下文为 `Paused` 且 F2 的 `process_mode == DISABLED`，状态仍为 Run；**WHEN** 窗口恢复并触发恢复，**THEN** 上下文回到 `Run` 且 F2 解冻。**GIVEN** 状态 Menu，**WHEN** 窗口失焦，**THEN** 上下文与状态均不变。
18b. **GIVEN** F6 直接调用 `request_context(&"Paused")`（模拟失焦，绕过按键），**WHEN** `context_changed(Run, Paused)` 到达，**THEN** F2 的 `process_mode == DISABLED`。冻结必须由信号驱动，不能只在 `ui_pause` 处理函数里做——否则失焦路径会漏掉。
19. **GIVEN** 状态 Menu，**WHEN** 调用 `request_quit()`，**THEN** 不出现确认框，一帧内状态为 Quitting，F8 记录的退出钩子被调用一次。
20. **GIVEN** 状态 Run，**WHEN** 调用 `request_quit(false)`，**THEN** 确认框出现、状态仍为 Run；**WHEN** 在确认框出现的同一帧内触发 `ui_accept`，**THEN** 确认框保持打开、状态仍为 Run。
21. **GIVEN** 确认框已打开，**WHEN** 选择「退出」，**THEN** 状态直接从 Run 进入 Quitting——不发生 Transition、不调用 `end_run`、不发出目标为 Menu 的 `flow_changed`。**WHEN** 选择「继续这一局」，**THEN** 状态仍为 Run 且不发出任何 `flow_changed`。
22. **GIVEN** F8 与全新进程夹具，**WHEN** 在 Loading 中请求退出且数据随后返回 Ready，**THEN** 一帧内状态为 Quitting、F8 的钩子被调用一次；在子进程中运行时退出码为 0，且日志通道 `L` 中无 F1 的 `_ready`。
23. **GIVEN** Transition 中收到退出请求，**WHEN** 当前切换完成，**THEN** 下一帧内状态为 Quitting，F3 中无淡入记录，旧世界的 `_exit_tree` 未触发。

### 不越界

24. **GIVEN** 全仓库扫描，**THEN** 本系统不暴露 `resume_run`，且不存在任何在 Run 中写入存档的代码路径。
25. **GIVEN** 入场费资金为 0，**WHEN** 界面流程调用 `start_run`，**THEN** 切换照常完成并返回 `true`——本系统不产生拒绝、不读取任何经济字段。
26. **GIVEN** 全仓库静态扫描，**WHEN** 检查数据配置的三类 Resource 定义与全部 `.tres`，**THEN** 不含任何世界场景路径字段；场景路径只出现在本系统的代码常量中。

### 公式

27. **GIVEN** FADE_OUT = 0.12、FADE_IN = 0.12，**WHEN** F7 以 0.01 s 完成一次切换，**THEN** 测得的 `transition_total` 落在 0.24–0.26 s。
28. **GIVEN** 同上常量，**WHEN** F7 分别以 0.01 s 与 0.21 s 完成两次切换，**THEN** 两次测得值之差为 0.20 s ± 一帧。这条排除「无条件返回 0.25 s」的实现。
29. **GIVEN** FADE_OUT = FADE_IN = 0.08（范围下限），**WHEN** F7 以 0.005 s 完成，**THEN** 测得值 ≥ 0.121 s。
30. **GIVEN** F7 延时 0.45 s、阈值 0.40 s，**WHEN** 淡出结束，**THEN** 指示器在淡出结束后 0.40–0.45 s 之间出现，**且出现在挂载完成（0.52 s）之前**，并一直可见到遮罩清除。出现时刻早于挂载完成，是看门狗与「事后判断」的分界线。
31. **GIVEN** F7 延时 0.30 s，**WHEN** 一次切换完成，**THEN** 指示器全程不可见。
32. **GIVEN** 冷启动且 F7 以 0.01 s 完成，**WHEN** 进入 Loading，**THEN** 指示器在 t = 0 的第一次绘制就出现，不等待任何阈值。
33. **GIVEN** `LOAD_TIMEOUT` = 5.0 s，**WHEN** F7 延时 5.3 s，**THEN** 切换返回 `false`，旧世界 `process_mode != DISABLED` 且仍在树中，状态回到切换前，**且状态不是 Error**。

## Open Questions

| # | 问题 | 归属 | 目标解决时机 |
|---|---|---|---|
| 1 | 主菜单与局内是否合并进同一个场景，以减少一次实例化开销？ | engine-programmer | 原型阶段 |
| 2 | 暂停菜单是 Run 世界内的节点，还是挂在根节点上的独立 UI 层？ | ux-designer | 界面流程 GDD |
| 3 | Error 屏是否需要「复制报告到剪贴板」按钮？ | ui-programmer | 垂直切片 |
| 4 | 首局引导是否需要 `Blocked` 之外的自定义上下文（例如引导专用的 Menu 子态）？ | ux-designer | 首局引导 GDD |
| 5 | 是否需要「重开本局」直达入口，绕过菜单与结算屏？ | game-designer | 界面流程 GDD |
