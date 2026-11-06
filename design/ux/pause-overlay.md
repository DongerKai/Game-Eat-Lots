# Pause Overlay Design（暂停覆盖层）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Game**: 回本 (Break Even)
> **Platform Targets**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主，手柄部分支持；无触屏。参考 1280×720，支持至 3840×2160 与 21:9
> **Related GDDs**: `ui-flow.md`（UIF-R15/16）、`scene-flow.md`（暂停与退出）、`input.md`（Paused 上下文）、`run-flow.md`（UI Requirements 9）、`first-run-onboarding.md`（暂停清栈）、`audio-feedback.md`
> **Accessibility Tier**: Standard（逐条要求见 `design/accessibility-requirements.md`）
> **Sources**: `design/ux/interaction-patterns.md`（PAT-02/03/06/08/13）、`design/ux/hud.md`（视觉与焦点基线）、ADR-0005（输入上下文栈）
> **Sibling Specs**: `design/ux/confirm-modal.md`、`design/ux/readonly-notice.md`、`design/ux/error-screen.md`、`design/ux/onboarding-layer.md`、`design/ux/hud.md`、`design/ux/interaction-patterns.md`
> **Template**: UX Screen Spec

> **范围边界**：本文件只覆盖 `Paused` 顶层状态下叠在局内 HUD 之上的那一层。它**不拥有** HUD 的数字、不拥有设置面板（`SettingsModal`）、不拥有确认框（`ConfirmModal`）、不拥有顶层状态机。暂停覆盖层不是屏幕栈中的一屏，也不压 `Modal` 上下文；只有从它打开设置或确认时才由界面流程压 `Modal`。

## 引用代号

| 代号 | 出处 |
|---|---|
| `UIF-Rn` | `design/gdd/ui-flow.md` Core Rules 第 n 条 |
| `UIF-UI` | 同文件 UI Requirements 表 |
| `SCN-Rn` | `design/gdd/scene-flow.md` Core Rules 第 n 条 |
| `INP-Rn` | `design/gdd/input.md` Core Rules 第 n 条 |
| `RUN-UI n` | `design/gdd/run-flow.md` UI Requirements 第 n 条 |
| `ONB-Rn` | `design/gdd/first-run-onboarding.md` Core Rules 第 n 条 |
| `HUD-Rn` | `design/gdd/in-game-hud.md` Core Rules 第 n 条 |
| `ADR-0005` | `docs/architecture/adr-0005-input-context-stack.md` |
| `PAT-nn` | `design/ux/interaction-patterns.md` 第 nn 号模式 |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` 要求编号 |

---

## 1. 目的与玩家需求

**暂停要回答两个问题，且只回答两个：现在停在哪里、下一步去哪。** 玩家按下 Esc 的时刻，脑中通常还挂着「还差 3 元回本、还剩 4 格」。暂停层如果搬出第二套数字，就会把一本账变成两本，破坏 HUD 的单账前提（`HUD-R1`）。因此本层的职责边界是：**冻结并保留现场，提供四个出口，绝不复制任何局内读数**。

玩家在暂停层的真实需求按优先级：

1. **确认游戏真的停了**——回合、容量、净利一个都没动，画面还是刚才那张（`UIF-R15`、`SCN-R7`）。
2. **继续**——最高频动作，默认焦点，`ui_pause` 一键返回。
3. **调设置**——音量、字号、减少动态；改完回到暂停层，不是回到主菜单。
4. **离开这一局**——放弃（有确认、无结算）或退出游戏（有确认、需 flush）。

> **设计立场**：暂停层是**门厅**，不是仪表盘。它不显示「本局已进行 3/10 回合」「当前净利 +12」这类摘要——那些数字在背后 HUD 上仍然可读，复制一份只会制造不一致的第二个来源。

---

## 2. 进入与退出契约

### 2.1 进入

| 来源 | 触发 | 前置条件 | 进入后 |
|---|---|---|---|
| `Run` | `ui_pause`（Esc / P / 手柄 Start） | 顶层状态为 `Run` | `SCN-R7` 请求 `Run → Paused`；界面流程在 HUD 之上绘制本层 |
| `Paused` | 从设置/确认返回 | 模态关闭 | 本层恢复可交互，焦点回到打开模态的控件 |
| 首局引导期间 | `ui_pause` | `Blocked` 上下文 | `request_context(Paused, clear_stack := true)` 清空引导临时栈（`ONB-R17`） |

**进入时界面流程必须做的事**：读取 `run_state` 只读快照一次用于选择变体（§5），把焦点放到「继续」，不卸载、不重建 HUD（`UIF-R15`）。**不得**读取 `RunResult`、钱包、纪录或任何结算数据。

### 2.2 退出

| 出口 | 触发 | 目标 | 语义 |
|---|---|---|---|
| 继续 | `ui_accept` 在「继续」/ `ui_pause` | `Paused → Run` | 恢复现场；不推进回合、不改数字 |
| 设置 | 激活「设置」 | 压 `Modal`，打开 `SettingsModal` | 关闭后回本层 |
| 放弃本局 | 激活并按 `ConfirmModal` 确认 | `Paused → Menu`，`abandoned = true` | 不产生 `RunResult`、不写纪录（`UIF-R16`） |
| 退出游戏 | 激活并按 `ConfirmModal` 确认 | `Quit` | 先 `存档.flush()`；失败则再确认 |
| 窗口关闭 | OS 关闭事件 | 同「退出游戏」 | 由应用入口转成退出请求（`SCN-R10`） |
| 外部状态切换 | 结算/故障/退出 | 离开 `Paused` | 本层立即拆除，不播放退出动画等待 |

**禁止的出口**：不存在「保存并退出」「返回主菜单但不放弃」「重新开始本局」。MVP 没有手动保存（`UIF-R21`），也没有中途存档。

---

## 3. 导航位置与屏幕栈

- 本层**不进入** `ScreenHost` 的屏幕栈（`ui_stack_depth` 不变），它是 `Paused` 状态的覆盖层，绘制在 HUD 之上、`ScreenHost` 之下。
- 打开设置/确认会压 `Modal`：**暂停层本身不压栈**，所以「暂停菜单 + 确认框」的 `modal_depth = 1`，不是 2。
- 从暂停菜单打开设置、再在设置里打开删除生涯确认时，`modal_depth = 2` 是唯一合法的二级组合（`UIF-R5`）；暂停层在两层模态之下保持挂载但不可交互。
- 顶层切换到任何其他状态时，`request_context(..., clear_stack := true)` 会清空临时栈（`INP-R12`），界面流程**不得再补弹栈**（`UIF-R5` 配对断言）。

---

## 4. 布局

### 4.1 线框（1280×720，100% 字号）

```
┌──────────────────────────────────────────────────────────────┐
│  HUD 顶栏（波次 / 回合 / 净利差额）——压暗 55%，仍可辨认      │
│                                                              │
│         ┌──────────────────────────────┐                     │
│         │  已暂停                      │  ← 标题，非按钮     │
│         │                              │                     │
│         │  ▸ 继续                      │  ← 默认焦点         │
│         │    设置                      │                     │
│         │    放弃本局                  │                     │
│         │    退出游戏                  │                     │
│         └──────────────────────────────┘                     │
│                                                              │
│  HUD 卡片 / 右栏 / 详情行——压暗 55%，不可交互                │
└──────────────────────────────────────────────────────────────┘
```

- 面板宽 `clamp(0.34 × W, 360, 460)`，居中偏右 `+6% W`（避免完全盖住左上的净利/差额）；垂直居中。
- 面板底色为深墨色、不透明度 ≥ 0.92；文字米白，对比度 ≥ 4.5:1（`ACC-VIS-03`）。
- 背景压暗层覆盖全屏，`MOUSE_FILTER_STOP`：**鼠标不能点到背后的 HUD**；滚轮无效。
- 面板内边距 24 px；菜单项高 ≥ 56 px、行距 ≥ 12 px（`ACC-MOT-06`）；标题与首项间距 20 px。
- 150% 字号：面板宽上限放宽到 `0.42 × W`，菜单项随行高增长；不得出现横向滚动或文字截断（`ACC-VIS-10`）。

### 4.2 布局区

| 区 | 内容 | 焦点 | 说明 |
|---|---|---|---|
| Z1 背景 | 冻结的 HUD 画面 + 压暗层 | 不可达 | 只读；不复制数字 |
| Z2 标题 | 「已暂停」 | 不可达 | 不是状态摘要，不加局内数据 |
| Z3 菜单 | 继续 / 设置 / 放弃本局 / 退出游戏 | 可获焦 | 上→下顺序即焦点顺序 |
| Z4 脚注 | 无 | — | 不写键位图标（`ACC-MOT-04`） |

### 4.3 焦点顺序

`focus_order = [继续, 设置, 放弃本局, 退出游戏]`；隐藏项跳过；初始焦点为第一个可用项 =「继续」（`UIF-UI`）。

- 上下导航在可见项之间循环；**不进入背景 HUD**。
- 鼠标悬停与键盘焦点分开绘制（`INP-R8`、`PAT-01`）：悬停 = 浅底；焦点 = 2 px 实线外框 + 2 px 间距。
- 禁用项不可获焦；本层的「放弃本局」在受限阶段是**隐藏**而非禁用（见 §5 与 OQ-1）。

### 4.4 组件清单

| 组件 | 类型 | 必需 | 备注 |
|---|---|---|---|
| `PausePanel` | PanelContainer | 是 | 单实例；打开时淡入 |
| `PauseTitle` | Label | 是 | `tr("ui.pause.title")` |
| `PauseMenu` | VBoxContainer | 是 | 4 项，可隐藏 1 项 |
| `PauseItem` | Button | 是 | 每项一个；`focus_mode = ALL` |
| `PauseScrim` | ColorRect | 是 | 55% 墨色；`MOUSE_FILTER_STOP` |
| `PauseResumeHint` | Label | 否 | 若后续需要「Esc 继续」提示，见 OQ-3 |

---

## 5. 状态与变体

| 状态 | 条件 | 可见项 | 焦点 | 说明 |
|---|---|---|---|---|
| `OPEN` | `Paused` 且 `run_state ∈ {Active}` | 继续 / 设置 / 放弃 / 退出 | 继续 | 常规变体 |
| `RESTRICTED` | `run_state ∈ {Ending, Settling, Handoff}` | 继续 / 退出 | 继续 | 隐藏「放弃本局」（`UIF-R15`、`RUN-UI 9`） |
| `SETTINGS_OPEN` | `modal_depth = 1` 且栈顶为设置 | 同上但整层不可交互 | 无（焦点在模态） | 本层保持挂载；不重复绘制 |
| `CONFIRM_OPEN` | `modal_depth = 1` 且栈顶为确认 | 同上但整层不可交互 | 无 | 放弃/退出/删除生涯共用 |
| `FLUSH_FAILED` | 退出确认后 flush 失败 | 新确认框「未保存，仍要退出？」 | 取消 | 仍留在 `Paused`，不静默丢弃（`UIF-R16`） |
| `CLOSING` | 已请求 `Paused → Run` | 全部不可交互 | 无 | ≤ 0.2 s 淡出；不得延迟状态生效（`SCN-R12`） |
| `TORN_DOWN` | 离开 `Paused` | — | — | 立即拆除，无动画等待 |

**状态转移**

| 从 | 事件 | 到 | 副作用 |
|---|---|---|---|
| `OPEN` | `ui_accept` 于「继续」或 `ui_pause` | `CLOSING` | 请求 `Paused → Run`；不弹栈（本层未压栈） |
| `OPEN` | 激活「设置」 | `SETTINGS_OPEN` | `push_context(Modal)`；记住打开者控件 |
| `SETTINGS_OPEN` | 设置关闭 | `OPEN` | `pop_context()`；焦点回「设置」 |
| `OPEN` | 激活「放弃本局」 | `CONFIRM_OPEN` | `push_context(Modal)`；标题=放弃本局 |
| `OPEN` | 激活「退出游戏」/窗口关闭 | `CONFIRM_OPEN` | `push_context(Modal)`；标题=退出游戏 |
| `CONFIRM_OPEN` | 取消 | `OPEN` | `pop_context()`；无业务调用 |
| `CONFIRM_OPEN` | 确认放弃 | `TORN_DOWN` | 请求 `Paused → Menu`（`abandoned = true`） |
| `CONFIRM_OPEN` | 确认退出 → flush 成功 | `TORN_DOWN` | 请求 `Quit` |
| `CONFIRM_OPEN` | 确认退出 → flush 失败 | `FLUSH_FAILED` | 打开第二个确认；仍停留 `Paused` |
| `OPEN` / `RESTRICTED` | `run_state` 进入 `Ending` | `RESTRICTED` | 隐藏「放弃本局」；焦点若在其上则移到「继续」 |
| 任意 | 顶层状态离开 `Paused` | `TORN_DOWN` | 不广播、不补动画 |

**边界情况**

- **暂停期间局内数字不得变化**：本层只读 `run_state`；不订阅、不缓存任何数值（`UIF-R15`）。
- **暂停期间不推进回合、不结算**（`SCN-R7`）；音频进入 `Paused` 时同帧静默（`AUD` 规则 14）。
- **首局引导中暂停**：`Blocked` 被清空；恢复时引导重压同一白名单，失败则降级为无遮罩提示（`ONB-R17`、ADR-0020 D7）。暂停层不感知、不干预。
- **`Ending` 期间按下暂停**：允许进入暂停层，但只能继续或退出，不能放弃（结算已开始）。
- **窗口失焦不自动暂停**（`SCN` Edge Cases）：本层只由 `ui_pause` 或退出请求打开。
- **HUD 结束取餐确认面板打开时按 `ui_pause`**：该面板不压 `Modal`（`HUD-R16`），暂停仍会进入；两者叠加的行为未定义，见 OQ-4。

---

## 6. 交互映射

### 6.1 键盘 / 鼠标 / 手柄

| 操作 | 键盘 | 手柄 | 鼠标 |
|---|---|---|---|
| 上/下移动 | ↑ / ↓、W / S | 左摇杆、十字键 | 悬停菜单项 |
| 激活 | Enter / Space | A | 左键单击 |
| 继续 | Esc / P | Start | 点击「继续」 |
| 打开退出确认 | Backspace | B | 点击「退出游戏」 |
| 关闭本层 | 仅通过上述出口 | — | — |

- **Esc 的双重绑定按上下文取一个**：`Paused` 取 `ui_pause` = 恢复；`ui_cancel` 只能由 Backspace 触发（`INP-R2`）。这是本层最容易误解的映射，见 OQ-3。
- 手柄 B 是 `ui_cancel`：在暂停层中打开退出确认，不是恢复。
- 鼠标点击只激活悬停控件，一次点击只发一次 `ui_button_activated`；点击面板外空白不关闭本层（防止误触恢复/误触退出）。
- 导航重复：`D = 0.35 s`、`I = 0.12 s`；摇杆 0.60 / 0.40 回滞（`ACC-MOT-07`）。四项菜单最多 3 步，不需要加速。

### 6.2 `ui_cancel` 落点矩阵（精确）

`back_target` 在 `Paused` 且无模态时命中优先级 8 → `OPEN_QUIT_CONFIRM`（`UIF` Formula）。

| 前置 | 按键 | 实际语义 | 落点 |
|---|---|---|---|
| `Paused`，`modal_depth = 0` | Esc | `ui_pause` | **恢复** `Run` |
| `Paused`，`modal_depth = 0` | Backspace | `ui_cancel` | 打开退出确认（焦点「取消」） |
| `Paused`，`modal_depth = 0`，焦点在任意项 | 手柄 B | `ui_cancel` | 打开退出确认 |
| `Paused`，设置打开（`modal_depth = 1`） | Esc / Backspace / B | Modal 取 `ui_cancel` | 关闭设置，回暂停层；**不**打开退出确认 |
| `Paused`，确认框打开（`modal_depth = 1`） | Esc / B | Modal 取 `ui_cancel` | 取消确认，回暂停层；无业务调用 |
| `Paused`，设置 + 删除生涯确认（`modal_depth = 2`） | Esc / B | 顶层模态 | 关闭第二层确认，回设置 |
| `Paused`，`FLUSH_FAILED` 确认框 | Esc / B | Modal 取 `ui_cancel` | 取消退出，回暂停层；不丢弃进度 |
| `Paused`，模态打开 | Esc | **不会**恢复 | 被 Modal 捕获，绝不同时恢复与关框 |

**NONE 的情况**：本层没有 `back_target = NONE` 的路径；所有 `ui_cancel` 要么恢复（Esc 经 `ui_pause`），要么开确认，要么关模态。因此不存在「按键无声消失」。

---

## 7. 数据与输入契约

| 值 / 行为 | 来源 | 口径 | 不得做的事 |
|---|---|---|---|
| 是否显示「放弃本局」 | `run_state` 只读快照 ∈ `{Ending, Settling, Handoff}` | 餐局流程发布；界面流程不推算 | 不根据剩余回合、容量、净利自行推断阶段 |
| 初始焦点 / 焦点顺序 | 本规格 §4.3 | 构建期固定 | 不按节点树顺序 |
| 当前是否可交互 | `modal_depth = 0` | 界面流程 | 不在模态下重复处理 `ui_accept` |
| 退出前 flush | `存档.flush()` | 存档系统 | 不自行写盘、不显示手动保存 |
| 设置项 | 机器级设置模型 | `UIF-R24` | 不写生涯档 |
| 暂停/恢复音效 | `sfx.flow.pause` / `resume` | 音效反馈订阅 `state_changed` | 本层不发原始输入事件 |

### 7.1 输入上下文

- 进入 `Paused`：场景与流程管理调用 `request_context(Paused, clear_stack := true)`；本层**不**调用 `push_context`。
- 打开设置/确认：界面流程调用 `push_context(Modal)`；关闭时 `pop_context()`，成对出现（`UIF-R27`）。
- `Modal` 允许：导航、`ui_accept`、`ui_cancel`；**不允许 `ui_pause`**——所以模态期间 Esc 不会恢复。
- `Paused` 允许：`ui_pause`（恢复）、`ui_accept`（激活焦点）、`ui_cancel`（退出确认）；`game_skip_wave` 被屏蔽（`INP` States 表）。
- `push_context` 总是加深栈并广播 `context_changed`，即使同值（`ADR-0005` D2）；本层不得依赖「同值压栈无事发生」。

---

## 8. 事件

| 事件 | 何时 | 载荷 | 订阅方 |
|---|---|---|---|
| `ui_button_activated(control_id)` | 激活菜单项 | `pause.resume` / `pause.settings` / `pause.abandon` / `pause.quit` | 音效反馈 |
| `state_changed(Paused, Run)` | 恢复生效 | — | 音效（`sfx.flow.resume`） |
| `state_changed(Paused, Menu)` | 放弃生效 | — | 音效、存档（无写入） |
| 无自定义暂停事件 | — | — | 不新增 GDD 未定义事件 |

> 本层**不**广播「打开暂停菜单」之类的语义事件；音频只认 `state_changed` 与 `ui_button_activated`（`AUD` 契约）。音效进入 `Paused` 时同帧停止池内 voice（`AUD` 规则 14），暂停音例外。

---

## 9. 转场与动画

- 打开：面板与压暗层 120 ms 淡入（`modulate.a`），焦点环在淡入结束前即为最终位置；**不位移**。
- 关闭：80 ms 淡出；若 `reduce_motion = true`，两者为 0（`ACC-VIS-12`）。
- 转场总时长 ≤ 0.2 s，且**不得延迟** `Paused → Run` 的状态生效（`SCN-R12`）；视觉未播完时新切换立即中止动画。
- 恢复后画面与暂停前逐像素一致：HUD 未卸载、未重建、未重新布局（`UIF-R15`）。
- 禁止：模糊、缩放、滑动、粒子、屏幕震动；本层不动 HUD 的不透明度设置值（只叠加自己的压暗层）。

---

## 10. 模式应用

| 模式 | 本屏应用 | 不适用 / 例外 |
|---|---|---|
| **PAT-13 屏幕根替换与返回决策** | `ui_cancel` 严格走 `back_target`：Paused 无模态 → 退出确认；有模态 → 关模态 | 本层不是屏幕根，不触发 `REPLACE_ROOT` |
| **PAT-06 破坏性确认（默认取消）** | 「放弃本局」「退出游戏」都经 `ConfirmModal`，默认焦点「取消」，标题写动作、正文写代价 | 确认框细节归 `confirm-modal.md` |
| **PAT-02 焦点陷阱与焦点归还** | 本层自身困住焦点（背景 HUD 不可达）；模态打开后陷阱移到模态，关闭归还「设置」/「放弃本局」/「退出游戏」 | 退出后控件消失时回落到「继续」并记 WARNING |
| **PAT-03 带原因的禁用态** | 本层不使用禁用项；受限阶段采用隐藏（GDD 明文），与 PAT-03 的取舍见 OQ-1 | 若未来改为禁用，必须给原因句且不可获焦 |
| **PAT-08 非模态提示** | 本层本身不是提示；flush 失败用确认框而非 toast（必须阻断） | 保存指示不会在暂停层出现（暂停时不写盘） |

---

## 11. 无障碍

| 要求 | 本屏落点 |
|---|---|
| `ACC-VIS-02` | 菜单项与标题 ≥ 20 px @720p；不因 150% 降字号 |
| `ACC-VIS-03` | 面板文字/底 ≥ 4.5:1；压暗层不降低面板内对比度 |
| `ACC-VIS-05` / `ACC-VIS-07` | 焦点环 2 px 实线 + 2 px 间距；悬停浅底、焦点外框，形状可区分；深底控件外侧加 2 px 米白描边 |
| `ACC-VIS-08` | 灰度下可读出焦点位置与四个出口；「放弃本局」不靠颜色提示危险，靠文字 |
| `ACC-VIS-09` / `10` | 100/125/150% 三档；150% 下四项不换行错位、不截断；面板可增高，不横向滚动 |
| `ACC-VIS-12` | `reduce_motion` 时淡入/淡出为 0，焦点环无过渡 |
| `ACC-MOT-01` / `06` | 纯键盘/手柄可完成继续、设置、放弃、退出；按钮高 ≥ 56 px；焦点环不被裁切 |
| `ACC-COG-01` | 暂停随时可用，冻结不改数字 |
| `ACC-COG-03` | 本层与确认框均无超时；等待 60 s 不自动继续/退出 |
| `ACC-AUD-02` / `06` | 无音频专属信息；静音下四个出口与焦点完全可辨 |
| `ACC-VIS-15` | 本层没有「不可吃」概念；受限项是隐藏，不是流程禁用 |

**额外**：暂停层是「随时暂停」承诺的落点（`ACC-COG-01`）；压暗层不得压到无法辨认 HUD 布局，玩家恢复后不应失去位置感。

---

## 12. 本地化

| Key | 中文（基准） | 约束 |
|---|---|---|
| `ui.pause.title` | 已暂停 | ≤ 8 字 |
| `ui.pause.resume` | 继续 | 动词，≤ 6 字 |
| `ui.pause.settings` | 设置 | ≤ 6 字 |
| `ui.pause.abandon` | 放弃本局 | ≤ 8 字；不得写成「退出」以免与退出游戏混淆 |
| `ui.pause.quit` | 退出游戏 | ≤ 8 字 |
| `ui.pause.flush_failed.title` | 未保存，仍要退出？ | 确认框标题，≤ 16 字 |

- 所有字符串走 `tr()`；不得在场景里硬编码（`UIF-UI` 文案格式）。
- 德语/俄语扩写按 1.4× 预留：按钮不设固定宽度，面板随最长项增长到上限后换行。
- 「放弃本局」与「退出游戏」的差异必须在文案本身可辨，不能只靠确认框正文。

---

## 13. 验收标准

- [ ] **PAUSE-AC01**（核心用途）GIVEN `Run` 中按 Esc，WHEN 本层打开，THEN 顶层状态为 `Paused`、HUD 仍在场景树中且未重建、四个菜单项按 §5 `OPEN` 显示、焦点在「继续」。
- [ ] **PAUSE-AC02**（恢复）GIVEN 本层打开，WHEN 按 Esc 或激活「继续」，THEN 请求 `Paused → Run` 恰好一次，恢复后 HUD 数字、焦点、波次与暂停前逐项相同。
- [ ] **PAUSE-AC03**（返回决策）GIVEN 本层打开且无模态，WHEN 按 Backspace 或手柄 B，THEN 打开退出确认，焦点在「取消」，且**不**恢复游戏。
- [ ] **PAUSE-AC04**（模态优先）GIVEN 设置或确认框打开，WHEN 按 Esc/Backspace/B，THEN 只关闭最上层模态，本层仍在 `Paused`，不触发恢复或第二个确认。
- [ ] **PAUSE-AC05**（放弃）GIVEN 常规局，WHEN 激活「放弃本局」并确认，THEN 请求 `Paused → Menu` 携带 `abandoned = true`，不产生 `RunResult`、不打开收据、不写纪录。
- [ ] **PAUSE-AC06**（退出）GIVEN 本层打开，WHEN 激活「退出游戏」并确认且 flush 成功，THEN 恰好一次 `存档.flush()` 后请求 `Quit`。
- [ ] **PAUSE-AC07**（flush 失败）GIVEN 确认退出但 flush 失败，WHEN 处理完成，THEN 显示「未保存，仍要退出？」确认，仍停留 `Paused`；取消后本局可继续。
- [ ] **PAUSE-AC08**（Ending 限制）GIVEN `run_state = Ending` 或 `Settling` 或 `Handoff`，WHEN 本层打开，THEN 只显示「继续」「退出游戏」；不存在「放弃本局」的可聚焦节点。
- [ ] **PAUSE-AC09**（阶段不推算）GIVEN 剩余回合为 0 但 `run_state = Active`，WHEN 打开本层，THEN 「放弃本局」仍显示；界面流程不得用回合数推断 `Ending`。
- [ ] **PAUSE-AC10**（单账）GIVEN 本层打开，WHEN 遍历本层所有文本节点，THEN 不出现净利、差额、容量、回合、单位价值等局内读数的副本。
- [ ] **PAUSE-AC11**（鼠标）GIVEN 本层打开，WHEN 点击背景 HUD 的菜品卡或按钮，THEN 不产生任何激活、焦点变化或吃菜；点击面板外空白不关闭本层。
- [ ] **PAUSE-AC12**（手柄）GIVEN 仅手柄，WHEN 从打开本层到恢复、到打开确认并取消，THEN 全程可完成；B 打开确认、A 激活、Start 恢复。
- [ ] **PAUSE-AC13**（焦点归还）GIVEN 从「设置」打开模态，WHEN 关闭模态，THEN 焦点回到「设置」；若该控件已被移除，则回到「继续」并记 WARNING。
- [ ] **PAUSE-AC14**（文本缩放）GIVEN `text_scale_percent = 150`，WHEN 本层打开，THEN 四项完整可读、无截断/重叠/横向滚动，焦点环不被裁切。
- [ ] **PAUSE-AC15**（减少动态）GIVEN `reduce_motion = true`，WHEN 打开与关闭本层，THEN 无淡入淡出、无位移，状态切换在 1 帧内生效。
- [ ] **PAUSE-AC16**（无计时）GIVEN 本层打开，WHEN 等待 60 秒不操作，THEN 仍停留本层、不自动恢复、不自动退出、焦点不变。
- [ ] **PAUSE-AC17**（引导暂停）GIVEN B0 气泡打开（`Blocked`），WHEN 按 `ui_pause`，THEN 进入 `Paused` 并清空临时栈；恢复后引导重压白名单或降级为上下文提示，不卡住玩家。
- [ ] **PAUSE-AC18**（灰度）GIVEN 本层截图去色相，WHEN 逐项检查，THEN 能读出四个出口、当前焦点与「已暂停」标题，不依赖颜色。
- [ ] **PAUSE-AC19**（性能）GIVEN 1280×720、60 fps，WHEN 打开本层，THEN 单帧开销不导致掉帧；打开/关闭不触发 HUD 重新布局。
- [ ] **PAUSE-AC20**（上下文配对）GIVEN 从本层打开并关闭设置 10 次，WHEN 检查输入栈，THEN 每次 `push_context(Modal)` 都有对应 `pop_context()`，结束时栈深回到 `Paused` 基线。

---

## 14. 开放问题

| # | 问题 | 双方依据 | 当前处理 |
|---|---|---|---|
| 1 | 受限阶段「放弃本局」是**隐藏**还是**禁用**？隐藏会让面板高度变化（焦点与鼠标位置跳变），禁用则与 PAT-03 一致但 GDD 明写「不提供」。 | `UIF-R15`（只保留两项）vs `PAT-03`（禁用需原因）/ `ACC-VIS-15`（流程禁用 = 不可获焦、无原因） | 本规格按 GDD 隐藏，并让焦点在隐藏时移到「继续」；是否预留占位保持高度，见 #2 |
| 2 | 进入 `Ending` 时面板从 4 项变 2 项，布局稳定性（面板重排、点击目标位移）如何处理？ | `interaction-patterns.md` Gaps 表 `PauseOverlay` 待定 vs `UIF-R15` | 未定；候选：保留不可见占位（但仍不可聚焦）或面板高度固定。需 `ui-programmer` 评估 |
| 3 | Esc 在 `Paused` 是**恢复**、Backspace 才是退出确认，与玩家「Esc 打开菜单」的普遍预期相反；是否加一行「Esc 继续 · Backspace 退出」提示？ | `INP-R2`（映射层不能按上下文区分）vs `UIF-R15` / `PAT-13` | 本规格不写键位图标（`ACC-MOT-04`），仅记录；若加提示必须本地化且不占焦点 |
| 4 | HUD 的「结束取餐」确认面板不压 `Modal`（`HUD-R16`），此时按 `ui_pause` 会发生什么？暂停层叠在面板上、恢复后焦点归谁？ | `HUD-R16` vs `INP` States 表（`Paused` 只屏蔽 `game_skip_wave`） | 未定；本规格假设界面流程先取消或关闭面板再进 `Paused`，需 HUD 规格确认 |
| 5 | 从暂停层打开设置后关闭，是否应恢复暂停层焦点到「设置」，还是回到「继续」（更容易恢复游戏）？ | `UIF-UI` 焦点恢复（回打开者）vs 暂停场景的高频动作 | 本规格按 `UIF-UI` 回「设置」 |
| 6 | 窗口关闭发生在 `Run`（无暂停层）时，确认框叠在活动局上、游戏不冻结；回合制无计时，是否仍需先暂停？ | `SCN-R10`（窗口关闭同退出规则）vs `UIF-R16`（`Run`/`Paused` 需确认） | 未定；本规格只覆盖 `Paused` 路径，`Run` 路径归 `confirm-modal.md` 与场景流程 |
| 7 | 暂停层是否需要显示「本局进度未保存」之类的只读提示？ | `UIF-R21`（无手动保存）vs `ACC-COG` 认知负担 | 不加；暂停不写盘，提示会制造焦虑且无动作可做 |
