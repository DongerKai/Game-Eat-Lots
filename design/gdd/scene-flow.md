# 场景与流程管理（Scene & Flow）

> **Status**: In Review — Formulas 章节补写、输入上下文接口定名（2026-11-03），待重新评审
> **Author**: user + Claude（首次设计未咨询 systems-designer — Lean 模式；Formulas 后续补写）
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 数字看得见地涨

## Summary

场景与流程管理是《回本》的应用外壳：任意时刻恰好一个顶层状态（`Boot` / `Error` / `Menu` / `Run` / `Paused` / `Quit`），所有切换只经一个入口、在该帧渲染前生效。它不绘制界面、不算数值、不读写存档，只负责启动校验后的路由、一局的进入与结算交接、暂停与退出的确认规则，并通知输入切换上下文。结算、升级、设置都是 `Menu` 内的界面，由界面流程推进，顶层流程不感知它们。

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `数据配置, 输入`

## Overview

场景与流程管理是《回本》的应用外壳。它持有唯一一个事实——"此刻整个游戏处在哪个顶层状态"——并负责这些状态之间的每一次合法切换：启动、菜单、一局进行中、暂停、数据错误、退出。它不绘制任何界面，不参与一局内的任何数值计算，也不读写存档；它只把切换请求变成状态变更，把切换要携带的数据原样交给下一个状态的拥有者。

它与输入是分工而非重复：本系统决定应用处在哪个顶层状态，输入系统据此把上下文档位切到 Menu / Run / Paused；界面流程和餐局流程再在各自的顶层状态内自行推进。一局正常结算后，本系统把 `RunResult` 交给界面流程，由界面流程打开结算屏；中途放弃则不给 `RunResult`，本局不产生任何收益。

没有它，每个系统都要自己判断"现在能不能开始一局"，启动失败、重开、暂停退出这些路径就没有唯一的位置可以落，很容易出现两个系统同时认为自己拥有当前界面。它主要服务支柱 4「数字看得见地涨」——状态切换在单帧内完成，玩家不会在两次决策之间被加载或过场挡住。

## Detailed Design

### Core Rules

1. **单一顶层状态**：任意时刻恰好一个顶层状态。集合固定为六个：`Boot`、`Error`、`Menu`、`Run`、`Paused`、`Quit`。不为细分界面新增顶层状态——结算、升级、设置都是 `Menu` 内的界面，由界面流程推进。
2. **唯一切换入口与合法性**：所有顶层切换只经 `request_transition(to, payload)`，并返回该请求是否被接受（bool）。其他系统不得自行挂载/卸载场景或写状态。请求先校验 `(current, to)` 是否在下面的转移表内：合法则在该帧渲染前生效；非法则拒绝、保持原状态并记日志。**一帧内的多个请求全部针对帧初状态校验**，按到达顺序取第一个合法者生效，其余一律拒绝并各记日志——后到的请求不会因为前一个已经改变状态而重新变得合法。**日志级别**：普通非法转移与同帧落败记 WARNING；在非 `Run` 状态收到 `RunResult` 记 ERROR。调用方拿到 `false` 即知 payload 未被交付，不得假设成功。**幂等空操作**：若 `to == current` 且 payload 为空，返回 `true`，不改变状态、不发信号、不记日志（同一帧重复的 `Paused → Run` 落到此路径）；若 `to == current` 但带 payload，返回 `false`（payload 未被交付）。**作用域**：`Quit` 终态与 `Boot` 尚未完成校验期间不适用幂等空操作——这两段时间内任何请求一律返回 `false`。
3. **状态 → 输入上下文**：切换生效时，本系统调用 `request_context(default_ctx, clear_stack := true)`（输入 GDD 规则 12），把输入置于该状态的默认上下文并清空临时栈：`Boot` → Blocked、`Error` → Menu、`Menu` → Menu、`Run` → Run、`Paused` → Paused。`Error` 沿用 Menu 上下文只是为了让唯一按钮可以被导航与确认；`Error` 内不存在任何能触发 `Menu` 或 `Run` 的动作。`Modal` 与 `Blocked` 可由界面流程或首局引导临时压栈；本系统不感知其内容，只在顶层切换时保证输入栈回到该状态的默认上下文。
4. **启动序列**：进入 `Boot` 后读取数据配置的加载与校验结果；`Ready` → `Menu`，`Failed` → `Error`。启动期间输入为 Blocked。
5. **一局的进入**：`Menu → Run` 携带 `RunConfig`，由发起方（界面流程）组装：`station_id` 与当前局外升级等级快照。本系统不读存档、不组装 `RunConfig`，只原样转交。
6. **一局的离开**：`Run → Menu` 携带 `RunResult`，由餐局流程在结算完成时发起。`RunResult` 在切换生效的同一帧内、恰好一次同步交给界面流程，不落盘、不缓存；界面流程据此打开结算屏。**进入失败不是结算**：若 `RunConfig` 在进入时无法解析，餐局流程立即请求 `Run → Menu`，携带 `init_failed = true`；不产生 `RunResult`、不结算。
7. **暂停**：`Run` 中收到 `ui_pause` → `Paused`，本系统通知餐局流程冻结；`Paused` 中收到 `ui_pause` → `Run`，通知恢复。`ui_accept` 不直达本系统：它激活暂停菜单的焦点控件（初始焦点为「继续」），由界面流程消费后请求 `Paused → Run`。暂停期间不得推进回合、不得结算。
8. **放弃一局**：`Paused` 中请求回到菜单需二次确认；确认后 `Paused → Menu`，携带 `abandoned = true`——不产生 `RunResult`、不结算、不写纪录。这是支柱 2「没有白吃的菜」在流程层的落点。
9. **错误是受限态**：`Error` 不得进入 `Menu` 或 `Run`，只能退出；debug 构建额外可以"重新加载数据"回到 `Boot` 重走校验。release 构建中该入口**不存在**，不是存在后被拒绝。
10. **退出**：`Run` / `Paused` 中退出需确认，确认后本局作废；`Boot` / `Menu` / `Error` 中退出直接执行——启动阶段没有任何一局可损失。**窗口关闭请求与游戏内退出走同一规则**，由应用入口转成一次 `request_transition(Quit)`。所有退出都落到 `Quit`。
11. **重开**：结算屏可以同一档口再来一局，这是一次新的 `Menu → Run`，携带新的 `RunConfig`。上一局的任何状态不得延续到新局。
12. **切换不阻塞**：状态切换在单帧内生效，不插入等待态。淡入淡出不超过 0.2 秒，属于演示层（界面流程 / 音效反馈），不得推迟状态生效；关闭视觉过渡时切换照常立即生效。
13. **边界**：本系统不写存档、不改升级、不算数值、不绘制界面、不持有任何一局的内部状态。它只持有顶层状态，以及切换瞬间转交的 payload。
14. **确认对话框归界面流程**：退出确认与放弃确认由界面流程以 `Modal` 实现。本系统只接收最终结果（确认或取消），不绘制、不持有对话框状态。

### States and Transitions

| 状态 | 进入条件 | 默认输入上下文 | 该状态内由谁推进 |
|---|---|---|---|
| `Boot` | 应用启动 | Blocked | 本系统（等待数据校验结果） |
| `Error` | 启动校验失败 | Menu | 本系统（只提供退出；debug 可重载） |
| `Menu` | 启动成功 / 一局结算 / 进入失败返回 / 放弃一局 | Menu | 界面流程（主菜单、结算、升级、设置） |
| `Run` | 开始一局 | Run | 餐局流程（回合、饱腹、净利） |
| `Paused` | `Run` 中 `ui_pause` | Paused | 本系统（餐局流程已冻结） |
| `Quit` | 退出请求（`Run` / `Paused` 确认后；其余状态直接） | — | 无 |

| 从 | 触发 | 到 | 携带 | 备注 |
|---|---|---|---|---|
| `Boot` | 数据校验 `Ready` | `Menu` | — | |
| `Boot` | 数据校验 `Failed` | `Error` | 失败明细（仅 debug 构建展示） | |
| `Error` | 重新加载数据（debug） | `Boot` | — | 重走加载与校验 |
| `Menu` | 请求开始一局 | `Run` | `RunConfig` | 同一档口可重复开始 |
| `Run` | 餐局流程结算完成 | `Menu` | `RunResult` | 由餐局流程发起 |
| `Run` | 进入失败（`RunConfig` 无法解析） | `Menu` | `init_failed = true` | 不是结算、不产生 `RunResult`、不写纪录 |
| `Run` | `ui_pause` | `Paused` | — | |
| `Paused` | `ui_pause` / 界面流程请求恢复 | `Run` | — | 恢复现场；`ui_accept` 由界面流程消费 |
| `Paused` | 放弃确认 | `Menu` | `abandoned = true` | 无结算、无纪录 |
| `Boot` / `Error` / `Menu` / `Run` / `Paused` | 退出（`Run` / `Paused` 需确认） | `Quit` | — | 窗口关闭同规则 |

**非法转移一律拒绝**，包括：`Error → Menu`、`Error → Run`、`Menu → Paused`、任何从 `Quit` 出发的转移，以及从 `Boot` 发起但校验结果尚未返回时的所有请求——**唯独 `Boot → Quit` 例外**，退出在启动阶段始终合法（规则 10）。

**同状态请求的判定**：请求的目标状态与当前状态相同时，**不带 payload 者**直接返回、不触发切换、不记警告（幂等空操作）；**带 payload 者按该请求自己的语义拒绝**，并记其规定的日志级别——例如 `Run` 中再次请求开始一局（带 `RunConfig`）、或在非 `Run` 状态收到 `RunResult`。payload 是否存在是区分这两种情形的唯一判据。

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 数据配置 | 上游读取 | `Boot` 期读取加载与校验的最终结果（`Ready` / `Failed`）；不重复实现校验 | 硬 |
| 输入 | 下游通知 / 上游动作 | 输出：切换时调用 `request_context(default_ctx, clear_stack := true)`。输入：订阅 `context_changed` 与 `ui_pause` 触发 `Run ⇄ Paused`；`ui_accept` 由界面流程消费，本系统不监听 | 硬 |
| 界面流程 | 下游读取 | 发起 `request_transition(Menu → Run, RunConfig)`；订阅 `RunResult`、`abandoned`、`init_failed` 与当前顶层状态；接收 `ui_cancel` 打开退出确认，并把确认/取消的结果以切换请求交回本系统；暂停菜单消费 `ui_accept` 后请求 `Paused → Run` | 硬 |
| 餐局流程 | 下游读取 | 进入 `Run` 时取得 `RunConfig`；收到暂停/恢复通知；结算完成时发起 `Run → Menu, RunResult`；启动失败时发起 `Run → Menu, init_failed` | 硬 |
| 存档 | 无直接调用 | `RunConfig` 与纪录写入由发起方负责；本系统不读写存档 | 软（间接） |
| 音效反馈 | 下游订阅 | 状态切换事件（进入 `Run`、结算、退出） | 软 |
| 首局引导 | 下游请求 | 在 `Run` 内请求 Blocked 上下文并只放行指定动作；不改变顶层状态 | 软 |

**接口所有权**：本系统拥有顶层状态、转移合法性与 payload 转交；下游只请求切换、订阅状态与 payload，不得自行改写状态或挂载场景。

## Formulas

本系统只有两个数值规则——**状态生效延迟**与**过渡时长上限**。两者都来自规则 12「切换不阻塞」，且都必须是可判定的，否则"单帧生效"只是说法而不是规格。

这些值不进数据配置：它们是表现时序，不是平衡数值。`FADE_OUT` / `FADE_IN` 的具体时长由 `/ux-design` 的交互规格拥有，本 GDD 只规定上限与不可阻塞性。

### 状态生效延迟

The `state_latency_frames` formula is defined as:

`state_latency_frames = ceil(t_apply / FRAME_TIME)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `t_apply` | float (s) | 0 | calculated | 从 `request_transition` 被接受到目标状态写入的耗时 |
| `FRAME_TIME` | float (s) | 0.0167（60 Hz） | constant | 一帧的时长；MVP 固定 60 Hz 设计常量，不读 `project.yaml` |
| `state_latency_frames` | int | 0 | calculated | 请求被接受到目标状态渲染之间隔了几帧 |

**Output Range:** 恒为 0。**这不是"通常为 0"**：`t_apply` 必须为 0——状态在同一次调用内写入，当前帧渲染时目标状态已经在位。任何大于 0 的值都是规则 12 的违反，不是性能问题。

**Example:** 界面流程在帧 N 调用 `request_transition(Menu → Run, RunConfig)` 并收到 `true`；帧 N 渲染时已显示 `Run`，`state_latency_frames = 0`。

### 过渡总时长

The `transition_total` formula is defined as:

`transition_total = FADE_OUT + FADE_IN`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `FADE_OUT` | float (s) | 0–0.10 | `/ux-design` 交互规格 | 旧状态淡出时长 |
| `FADE_IN` | float (s) | 0–0.10 | `/ux-design` 交互规格 | 新状态淡入时长 |
| `TRANSITION_CAP` | float (s) | 0.20（常量） | constant | 过渡总时长上限 |

**Output Range:** 0–0.20 s。无障碍设置关闭视觉过渡时 `FADE_OUT = FADE_IN = 0`，即 `transition_total = 0`。

**约束：** `transition_total ≤ TRANSITION_CAP`；且 `transition_total` **不参与** `state_latency_frames` 的计算——过渡在状态已生效之后才播放。把两者相加是错误实现，会直接违反规则 12。

**Example:** `FADE_OUT = 0.08`、`FADE_IN = 0.08` → `transition_total = 0.16 s`，未超上限；状态仍在帧 N 生效。

> 过渡播放期间到达的新合法请求会立即中止剩余过渡（Edge Cases 最后一条）。过渡不是状态机的一部分，因此不存在"过渡被拒绝"——这一点没有数值，不单独成式。

## Edge Cases

- **If 应用启动时数据校验失败**：进入 `Error`，不得进入 `Menu` 或 `Run`；玩家唯一的出路是退出，debug 构建额外可重载数据。
- **If 同一帧内收到多个切换请求**：全部针对帧初状态校验；第一个合法请求在该帧渲染前生效，其余拒绝并各记 WARNING；不得出现两个状态同时生效。
- **If 同一帧内 `ui_pause` 与结算回传同时到达**：两者都按帧初状态校验。帧初为 `Run` 时先到者决定状态——`ui_pause` 先到则变 `Paused`，结算先到则变 `Menu`；后到者拒绝并记 WARNING，被拒绝的 `RunResult` 不交付。餐局流程依据 `request_transition` 返回 `false` 重新发起，不假设成功。
- **If 暂停菜单的「继续」与退出/放弃确认同时出现**：`ui_accept` 只由界面流程按当前焦点消费，本系统不监听该动作；只有界面流程发出的 `Paused → Run` 请求才会恢复，因此不存在恢复与对话确认争同一动作的情形。
- **If 请求的转移不在转移表内**：拒绝、保持原状态、记 WARNING，不向请求方抛异常。
- **If 请求的目标状态与当前状态相同且不携带 payload**：直接返回，不触发切换、不记警告——幂等空操作。携带 payload 的同状态请求按非法转移处理：拒绝、保持原状态；`RunResult` 记 ERROR，其余记 WARNING。
- **If `Boot` 尚未拿到校验结果就收到切换请求**：除 `Quit` 外全部拒绝；请求方不得假设状态已就绪。
- **If 结算回传 `RunResult` 时当前不在 `Run`**（例如已经放弃本局）：拒绝并记 ERROR——防止重复结算或跨局结算。
- **If `Run` 中收到第二次"开始一局"请求**：拒绝（不在合法转移表内），本局继续。
- **If `RunConfig` 的 `station_id` 无法解析**：由餐局流程发起 `Run → Menu`，携带 `init_failed = true`，不得停在半初始化的 `Run`；本系统只按合法转移处理这次返回，不校验 payload 内容。不产生 `RunResult`、不写纪录。
- **If `Boot` 中请求退出或窗口关闭**：直接进入 `Quit`，不确认——启动阶段没有任何一局可以损失。
- **If release 构建的 `Error` 屏被要求重新加载数据**：该入口不存在，界面上没有这个按钮；不存在"请求被拒绝"的分支。
- **If 暂停中窗口关闭或请求退出**：先走确认；取消则回到 `Paused`，本局继续。
- **If 暂停中确认放弃**：`Paused → Menu`，不产生 `RunResult`、不结算、不写纪录；下一次 `Menu → Run` 从全新 `RunConfig` 开始。
- **If 视觉过渡被关闭（无障碍设置）**：状态切换立即生效，不等待动画。
- **If 过渡动画尚未播完又收到切换请求**：新请求照常按转移表校验，合法则立即生效并中止未完成的过渡——过渡永远不阻塞状态。
- **If 窗口失去焦点**：不自动暂停。回合制没有实时压力（支柱 3），自动暂停只会打断心算。
- **If 一局在 `Paused` 或 `Run` 中崩溃、被系统杀掉或断电**：本系统不负责恢复；未结算的一局不产生收益，也不写纪录。
- **If 当前已在 `Quit`**：拒绝任何后续请求，不产生任何状态变更。
- **If 下游系统试图直接改状态或自行挂载场景**：这是契约违反，不是可处理情形——本系统不提供该路径，实现时以不暴露写接口为准。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 数据配置 | 硬 | `Boot` 期读取加载与校验的最终结果（`Ready` / `Failed`）；校验逻辑由数据配置拥有，本系统不重复实现 | GDD 已写 |
| 输入 | 硬 | 切换生效时调用 `request_context(default_ctx, clear_stack := true)`；订阅 `context_changed`；订阅 `ui_pause` 驱动 `Run ⇄ Paused` | GDD 已写 |

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 界面流程 | 硬 | 发起 `Menu → Run`（携带 `RunConfig`）；订阅 `RunResult`、`abandoned`、`init_failed` 与当前顶层状态；暂停菜单消费 `ui_accept` 后请求 `Paused → Run` | GDD 已写（`design/gdd/ui-flow.md`） |
| 餐局流程 | 硬 | 进入 `Run` 时取得 `RunConfig`；接收暂停/恢复通知；结算完成时发起 `Run → Menu, RunResult`；启动失败时发起 `Run → Menu, init_failed` | GDD 已写（`design/gdd/run-flow.md`） |
| 首局引导 | 软 | 在 `Run` 内请求 Blocked 上下文，只放行引导需要的动作；不改变顶层状态 | GDD 已写（`design/gdd/first-run-onboarding.md`） |
| 音效反馈 | 软 | 订阅状态切换事件（进入 `Run`、结算、退出；具体点 `Paused` / `Resumed` / `Quit`） | GDD 已写（`design/gdd/audio-feedback.md`） |

**不属于依赖的关系：**
- **存档**：本系统不读写存档，`RunConfig` 与纪录写入都由发起方负责。间接关系，不记为依赖。
- **界面流程 / 餐局流程**：它们是下游依赖方，不是上游。

**双向一致（已完成）**：按 `.claude/rules/design-docs.md`「依赖必须双向」，三处链接现已闭合——`design/gdd/systems-index.md` 第 3 行的 `Depends On` 记为「数据配置, 输入」；`data-config.md` 与 `input.md` 的下游依赖表都已各补一行「场景与流程管理」。

> **索引更正记录**：索引原先把本系统记为「无依赖」。设计后确认它必须拿到数据校验结果、并必须通知输入切换上下文，因此存在两个硬接口依赖，`Depends On` 已更正为「数据配置, 输入」。两者与数据配置、输入同属基础层，分层和设计顺序都不变。

## Visual/Audio Requirements

**本系统不绘制任何元素、不播放任何声音。** 它只定义三个交接点，视觉与音效由界面流程、音效反馈实现：

1. **状态切换**：允许一段不超过 0.2 秒的淡入淡出；不得推迟状态生效，可由无障碍设置整体关闭。具体时长由 `/ux-design` 的规格拥有，本 GDD 只规定上限与不可阻塞性。
2. **Boot 启动反馈**：由界面流程绘制等待画面；本系统只保证在拿到校验结果前停留在 `Boot`，不额外规定最短展示时长。
3. **Error 屏**：一段可读的错误说明 + 唯一的退出入口；debug 构建额外显示失败文件、记录 id 与字段。排版沿用「食堂价目板」——平静、无装饰、文字优先。

**音效**：本系统无专属音效；状态切换事件可作为音效反馈的触发源，具体音色由音效反馈决定。

## Game Feel

**目标手感是"没有手感"。** 切换必须感觉是瞬间发生的：按下"开始"到看见第一道菜之间，不存在任何可被察觉的等待；暂停与恢复不丢失任何画面状态。

- **预算**：状态切换在 1 帧内生效。淡入淡出是唯一的视觉延迟，且不推迟状态生效。
- **不做**：加载进度条、转场动画、装饰性动效。任何让玩家等一等才看到数字的东西都违反支柱 4「数字看得见地涨」。
- **失败感**：`Error` 屏要平静而不是恐慌——用价目板式的排版说明"数据有问题，游戏无法启动"，并给出唯一出路。

## UI Requirements

本系统不拥有任何界面，但对界面流程拥有的屏幕提出约束：

1. 除 `Quit` 外，每个顶层状态都必须有明确归属的界面；不存在"没有界面"的顶层状态。
2. `Error` 屏只提供退出（debug 额外提供重载数据），不得出现任何进入游戏的入口。
3. `Paused` 是覆盖在 `Run` 之上的遮罩，不得卸载或重建局内界面——恢复后画面与暂停前一致。
4. 退出确认是一个 `Modal`，取消后回到来源状态。
5. 界面可以自行切换内部屏幕，但显示内容不得与当前顶层状态矛盾；界面流程每次渲染前以本系统的状态为准。
6. 无障碍：过渡效果可整体关闭；`Error` 屏的信息不能只靠颜色传达。

## Cross-References

| 系统 | GDD | 关系 |
|---|---|---|
| 数据配置 | design/gdd/data-config.md | 上游：读取加载与校验结果，决定 `Boot → Menu` 还是 `Boot → Error` |
| 输入 | design/gdd/input.md | 上游：切换时调用 `request_context(..., clear_stack := true)`；订阅 `context_changed` 与 `ui_pause` 驱动暂停 |
| 界面流程 | design/gdd/ui-flow.md | 下游：发起 `Menu → Run`，接收 `RunResult`、`init_failed` 与 `abandoned`，按顶层状态切界面 |
| 餐局流程 | design/gdd/run-flow.md | 下游：取得 `RunConfig`，接收暂停/恢复通知，结算时回传 `RunResult`，启动失败时回传 `init_failed` |
| 首局引导 | design/gdd/first-run-onboarding.md | 下游（软）：在 `Run` 内请求 Blocked 上下文 |
| 音效反馈 | design/gdd/audio-feedback.md | 下游（软）：订阅状态切换事件 |

**双向核对**：数据配置与输入的 GDD 已各自在自己的下游依赖表中回指本系统，索引也已记录本系统的上游依赖。

## Acceptance Criteria

> **Formulas 覆盖**：Detailed Design 与 Visual/Audio 各声明了一条数值规则（单帧生效、过渡 ≤ 0.2 秒），Formulas 已给出对应的两个量与约束，第 31–33 条覆盖它们。其余判据全部来自 Detailed Design。

> **集成判据**：第 9、14 条依赖界面流程；第 10、25 条依赖存档；第 22、23 条依赖餐局流程，其中第 23 条还需要界面流程的结算屏发起重开。其余判据只需本系统加上输入与数据配置的测试替身。

1. **GIVEN** 应用启动，**WHEN** 数据校验结果尚未返回，**THEN** 顶层状态为 `Boot` 且输入上下文为 Blocked。
2. **GIVEN** `Boot`，**WHEN** 数据配置返回 `Ready`，**THEN** 状态切为 `Menu`，输入上下文为 Menu。
3. **GIVEN** `Boot`，**WHEN** 数据配置返回 `Failed`，**THEN** 状态切为 `Error`，且停留在 `Error` 期间不存在任何到 `Menu` 或 `Run` 的合法转移。
4. **GIVEN** debug 构建且处于 `Error`，**WHEN** 请求重新加载数据，**THEN** 回到 `Boot` 并重读校验结果。
5. **GIVEN** release 构建且处于 `Error`，**WHEN** 检查界面与公开接口，**THEN** 不存在"重新加载数据"入口，只有退出。
6. **GIVEN** `Boot` 且校验结果未返回，**WHEN** 收到除 `Quit` 外的任意切换请求，**THEN** 全部拒绝，状态保持 `Boot`；退出请求仍直接进入 `Quit`（见第 13 条）。
7. **GIVEN** `Menu`，**WHEN** 界面流程携带 `RunConfig` 请求开始一局，**THEN** 该帧渲染前切为 `Run`，餐局流程收到同一 `RunConfig`，输入上下文为 Run。
8. **GIVEN** `Run`，**WHEN** 收到 `ui_pause`，**THEN** 切为 `Paused`，餐局流程收到冻结通知，输入上下文为 Paused。
9. **GIVEN** `Paused`，**WHEN** 收到 `ui_pause` 或界面流程请求恢复，**THEN** 切回 `Run` 并发送恢复通知，画面与暂停前一致。
10. **GIVEN** `Paused`，**WHEN** 玩家确认放弃，**THEN** 切为 `Menu` 且携带 `abandoned = true`；不产生 `RunResult`，存档中的纪录与放弃前完全一致。
11. **GIVEN** `Run`，**WHEN** 结算完成并请求回 `Menu` 携带 `RunResult`，**THEN** 切为 `Menu`，界面流程在切换生效的同一帧内恰好收到一次该 `RunResult`。
12. **GIVEN** 当前不在 `Run`，**WHEN** 收到 `RunResult` 回传，**THEN** 拒绝并记 ERROR，状态不变。
13. **GIVEN** `Boot` / `Menu` / `Error`，**WHEN** 请求退出或窗口关闭，**THEN** 无确认直接进入 `Quit`。
14. **GIVEN** `Run` 或 `Paused`，**WHEN** 请求退出或窗口关闭，**THEN** 先要求确认；确认后进入 `Quit` 且本局作废，取消则回到原状态。
15. **GIVEN** `Error`，**WHEN** 请求 `Menu` 或 `Run`，**THEN** 拒绝、保持 `Error`、记 WARNING，调用方不收到异常。
16. **GIVEN** 同一帧收到多个合法请求，**WHEN** 该帧渲染前结算，**THEN** 仅第一个生效，其余拒绝并各记 WARNING；后到请求不因状态已改变而变得合法。
17. **GIVEN** 当前状态不是 `Quit` 且 `Boot` 已完成校验，请求目标等于当前状态且不携带 payload，**WHEN** 处理该请求，**THEN** 返回 `true`、状态不变且不记 WARNING；**GIVEN** 当前为 `Quit` 或 `Boot` 待校验期，**THEN** 返回 `false`（见 AC 19）。
18. **GIVEN** `Run`，**WHEN** 再次请求开始一局，**THEN** 拒绝并记 WARNING，本局继续。
19. **GIVEN** 处于 `Quit`，**WHEN** 收到任何切换请求，**THEN** 全部拒绝，状态不变。
20. **GIVEN** 除进入 `Quit` 外的任意顶层切换生效，**WHEN** 切换完成，**THEN** 输入临时栈被清空并置于目标状态的默认上下文。
21. **GIVEN** 视觉过渡被关闭，**WHEN** 合法请求被接受，**THEN** 该帧渲染前即显示目标状态，不等待动画。
22. **GIVEN** `Paused`，**WHEN** 餐局流程尝试推进回合或结算，**THEN** 在回到 `Run` 之前一律被拒绝。
23. **GIVEN** 结算屏重开，**WHEN** 进入新一局，**THEN** 使用全新 `RunConfig`，上一局的任何字段都不延续。
24. **GIVEN** `Run` 中窗口失去焦点，**WHEN** 焦点离开，**THEN** 不自动暂停，状态保持 `Run`。
25. **GIVEN** `Run` 或 `Paused` 时进程被强制终止，**WHEN** 重启，**THEN** 该局无收益、无纪录，存档与前一次正常退出时一致。
26. **GIVEN** 本系统的公开接口，**WHEN** 查找写状态、写存档、改数值或挂载场景的入口，**THEN** 均不存在。
27. **GIVEN** `Run` 中 `RunConfig.station_id` 无法解析，**WHEN** 餐局流程请求返回菜单，**THEN** 切为 `Menu` 且携带 `init_failed = true`，不产生 `RunResult`、不写纪录。
28. **GIVEN** `Paused`，**WHEN** 玩家按 `ui_accept`，**THEN** 本系统不自行恢复 `Run`（不监听该动作）；只有界面流程请求恢复才切回 `Run`。退出确认 `Modal` 打开时同理，`ui_accept` 只属于对话框。
29. **GIVEN** 任一请求被拒绝，**WHEN** 调用 `request_transition`，**THEN** 返回 `false`，调用方据此可以重新发起。**GIVEN** 当前不是 `Quit` 且 `Boot` 已就绪、目标状态与当前相同且不带 payload，**WHEN** 请求，**THEN** 返回 `true`，状态不变、不发切换信号、不记日志；**GIVEN** 目标状态与当前相同但带 payload，或当前为 `Quit` / `Boot` 待校验期，**THEN** 返回 `false`。
30. **GIVEN** 视觉过渡尚未播完，**WHEN** 收到新的合法请求，**THEN** 立即生效并中止未完成的过渡。
31. **GIVEN** 任意合法请求，**WHEN** `request_transition` 返回 `true`，**THEN** 该请求所在帧的渲染即显示目标状态，`state_latency_frames = 0`；不得出现"状态已变但画面仍是旧状态"的中间帧。
32. **GIVEN** `/ux-design` 给出 `FADE_OUT` 与 `FADE_IN`，**WHEN** 检查其和，**THEN** `transition_total ≤ 0.20 s`；**GIVEN** 无障碍设置关闭过渡，**THEN** `FADE_OUT = FADE_IN = 0` 且切换照常在同一帧生效。
33. **GIVEN** 过渡正在播放且已计入 `FADE_OUT`，**WHEN** 在此期间发生合法切换，**THEN** 剩余过渡被取消、新状态仍在同一帧生效，且 `state_latency_frames = 0`——过渡时长不计入状态延迟。

## Open Questions

| 问题 | 负责人 | 何时解决 |
|---|---|---|
| 结算屏是否需要提升为独立顶层状态？当前归入 `Menu`；若结算屏需要阻止一切返回主菜单的操作，可能需要独立状态。 | game-designer | 垂直切片前 |
| `RunConfig` 与 `RunResult` 的字段 schema 由谁拥有？本系统只负责转交；`RunConfig` 应定义在界面流程或局外升级，`RunResult` 应定义在餐局流程或结算。 | systems-designer | 设计餐局流程时 |
| 发布版 `Error` 屏是否显示失败文件路径？显示便于排查，但会暴露内部结构。 | qa-lead | 发布前 |
| 是否需要断点续局（崩溃后恢复未结算的一局）？当前明确不做；若后续要求，需要新增恢复态与一局的持久化。 | producer | 不在 MVP |

**GDD 定义行为，ADR 记录实现路径。** 以下是需要在 ADR 中落地的实现问题：状态变更与 `RunResult` 的通知机制（信号还是直接调用、顺序与重入）、`Quit` 是独立状态还是直接退出调用、操作系统 `close_requested` 的拦截方式、关闭视觉过渡的无障碍设置从哪里读取。封闭转移表、first-wins 语义与输入栈重置的**行为**已在正文定义，ADR 只决定实现手段。

**→ 需要一份 critical ADR**：`docs/architecture/` 目前只有 `tr-registry.yaml`。`workflow-modes.md` 把场景管理列为基础层需要 critical ADR 的典型例子，若缺这份 ADR，`/gate-check technical-setup` 会在这里卡住。
