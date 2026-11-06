# Confirm Modal Design（确认模态）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Game**: 回本 (Break Even)
> **Platform Targets**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主，手柄部分支持；无触屏。参考 1280×720
> **Related GDDs**: `ui-flow.md`（UIF-R5/6/7/16/22/24）、`save-system.md`（UI Requirements 4/6）、`scene-flow.md`（退出/放弃）、`input.md`（Modal 上下文）、`audio-feedback.md`
> **Accessibility Tier**: Standard（逐条要求见 `design/accessibility-requirements.md`）
> **Sources**: `design/ux/interaction-patterns.md`（PAT-02/03/06）、`design/ux/hud.md`（焦点与色板基线）、ADR-0005（`Modal` 栈与消费）
> **Sibling Specs**: `design/ux/pause-overlay.md`、`design/ux/readonly-notice.md`、`design/ux/error-screen.md`、`design/ux/onboarding-layer.md`、`design/ux/hud.md`、`design/ux/interaction-patterns.md`
> **Template**: UX Screen Spec

> **范围边界**：本文件覆盖**通用确认模态**：放弃本局、退出游戏、购买升级、删除生涯（两次确认）、以及窗口关闭在需要确认时的同一形态。它**不覆盖** HUD 的「结束取餐」面板（`HUD-R16` 明确不压 `Modal`，归 `hud.md`）、不覆盖 `BlockedModal`（强制模态，`modal_cancel_enabled = false`，归 `readonly-notice.md`）、不覆盖设置滑杆。确认框只收集「是/否」，不承载表单、不承载校验。

## 引用代号

| 代号 | 出处 |
|---|---|
| `UIF-Rn` / `UIF-UI` | `design/gdd/ui-flow.md` Core Rules 第 n 条 / UI Requirements 表 |
| `SAV-UI n` | `design/gdd/save-system.md` UI Requirements 第 n 条 |
| `SCN-Rn` | `design/gdd/scene-flow.md` Core Rules 第 n 条 |
| `INP-Rn` | `design/gdd/input.md` Core Rules 第 n 条 |
| `HUD-Rn` | `design/gdd/in-game-hud.md` Core Rules 第 n 条 |
| `ADR-0005` | `docs/architecture/adr-0005-input-context-stack.md` |
| `PAT-nn` | `design/ux/interaction-patterns.md` |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. 目的与玩家需求

**确认框不是「再问一次」，而是把代价说清楚。** 玩家按下「放弃本局」时，需要的不是「确定吗？」这种同义反复，而是一句能让他改变主意的信息：本局不会结算、不会写纪录、当前净利不会被记录。确认框的质量取决于**取消是否容易、代价是否具体、默认是否安全**。

三条不可退让的立场：

1. **默认取消**：所有不可逆/破坏性动作，焦点落在「取消」（`UIF-R6`、`PAT-06`）。误按 `ui_accept` 不会造成损失。
2. **不重复校验**：「确认」不是第二次业务校验；`purchase(id)`、`delete_career()` 仍完整校验并在失败时给出原因（`UIF-R22`）。确认框不预判、不吞错、不显示「应该能成功」。
3. **无时限**：不自动关闭、不自动确认、不因等待而降级（`ACC-COG-03`）。

---

## 2. 进入与退出契约

### 2.1 调用方与载荷

| 调用方 | 动作 | 标题（动作名） | 正文（代价/成本） | 确认后 |
|---|---|---|---|---|
| `PauseOverlay` | 放弃本局 | 放弃本局 | 本局不会结算，也不会写入纪录。 | `request_transition(Menu, {abandoned: true})` |
| `PauseOverlay` / `MainMenu` | 退出游戏 | 退出游戏 | 本局将作废、不记录成绩；未保存的进度可能丢失。 | `存档.flush()` → `request_transition(Quit)` |
| `UpgradeScreen` | 购买升级 | 购买「{名称}」 | 花费 {价格} 元；购买后余额 {余额} 元。 | `purchase(id)` |
| `SettingsModal` | 删除生涯（第一次） | 删除生涯？ | 旧档会保留为隔离文件，可在文件目录中找回；当前生涯将被重置。 | 打开第二次确认，不调用业务 |
| `SettingsModal` | 删除生涯（第二次） | 确认删除生涯 | 主菜单将回到新生涯；此操作不可在游戏内撤销。 | `delete_career()` |
| 应用入口（`Run` 窗口关闭） | 退出游戏 | 退出游戏 | 本局将作废、不记录成绩；未保存的进度可能丢失。 | `存档.flush()` → `Quit` |

> 正文只写**这次动作的后果**，不写通用说明；不写「你确定吗？」；不把标题重复一遍。金额一律整数元、带符号规则见 §7。

### 2.2 进入前置

| 条件 | 处理 |
|---|---|
| `modal_depth < 2` | 允许打开；界面流程 `push_context(Modal)` |
| `modal_depth = 2` | **拒绝打开**，返回 `false`，不排队、不替换；焦点留在原控件（`UIF-R5`） |
| 已有同 id 确认框 | 不重复打开；复用现有实例并聚焦「取消」 |
| 调用方控件已禁用或消失 | 不打开；记 WARNING，焦点回父屏第一个可用控件 |

### 2.3 退出

| 出口 | 结果 | 副作用 |
|---|---|---|
| 取消（按钮 / `ui_cancel` / Esc / 手柄 B） | 关闭 | `pop_context()`；焦点归还打开者；**零业务调用** |
| 确认（按钮 / `ui_accept`） | 执行动作 | 按钮立即禁用；按动作进入 `SUBMITTING` 或 `AWAIT_SECOND` |
| 第二次确认的取消 | 回第一次确认 | `modal_depth` 从 2 降到 1；焦点回「取消」 |
| 第二次确认的确认 | 执行删除 | `delete_career()`；成功后关闭两层并刷新主菜单 |
| 外部强制拆除（状态切换） | 立即关闭 | 不执行动作；不补动画 |

**禁止**：点击遮罩关闭、点击面板外关闭、`ui_pause` 关闭、自动关闭、把「关闭窗口」当成第三个按钮。窗口关闭由场景流程直接处理，不经过确认框的按钮集合（`UIF-R16` 注释、`SCN-R10`）。

---

## 3. 导航位置与屏幕栈

- 确认框不是屏幕，是 `MODAL_STACK` 的一项；`modal_depth` 最多 2（`UIF-R5`）。
- 合法组合只有：0 无模态；1 设置**或**任一确认；2 设置 + 删除生涯确认。**确认框上再开确认框**（如放弃确认上再开退出确认）不允许。
- 打开时 `push_context(Modal)` 总是加深栈并广播 `context_changed`，即使当前已是 `Modal`（ADR-0005 D2）；关闭时 `pop_context()` 成对（`UIF-R27`）。
- 顶层状态切换会 `clear_stack := true` 清空模态（`INP-R12`）；界面流程不得在状态切换后补 `pop_context()`（`UIF-R5` 配对断言）。

---

## 4. 布局

### 4.1 线框（1280×720，100% 字号）

```
┌──────────────────────────────────────────────────────────────┐
│  背景屏幕（暂停层 / 升级页 / 设置）——压暗 55%，不可交互      │
│                                                              │
│              ┌────────────────────────────────┐              │
│              │  购买「保温柜」                │  ← 标题=动作 │
│              │                                │              │
│              │  花费 120 元；购买后余额       │  ← 正文=代价 │
│              │  −35 元。                      │              │
│              │                                │              │
│              │              [ 取消 ] [ 确认 ] │              │
│              └────────────────────────────────┘              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

- 面板宽 `clamp(0.42 × W, 360, 560)`；正文最大 3 行（100%），超出见 OQ-1。
- 面板为**米白纸面 + 墨色文字**；危险动作不使用红字（`ACC-VIS-04`：红只许落在米白底且大字号也不放宽），改用「动词 + 明确代价 + 左侧 4 px 警示条（非文本 3:1）」。
- 按钮行右对齐：**取消在左、确认在右**；两按钮高 ≥ 56 px、间距 ≥ 16 px；确认按钮宽度不小于取消按钮。
- 面板内边距 24 px；标题与正文间距 16 px；正文与按钮行间距 24 px。
- 150% 字号：面板宽度上限提高到 `0.52 × W`，正文换行、不截断；按钮文字不换行，按钮随文字增高。

### 4.2 布局区与焦点顺序

| 区 | 内容 | 焦点 |
|---|---|---|
| Z1 遮罩 | 55% 墨色，`MOUSE_FILTER_STOP` | 不可达 |
| Z2 标题 | 动作名（删除生涯含「？」） | 不可达 |
| Z3 正文 | 代价/成本；金额来自调用方 | 不可达 |
| Z4 按钮行 | 取消 / 确认 | `focus_order = [取消, 确认]` |

- 初始焦点 =「取消」（`UIF-UI`、`PAT-06`）；**删除生涯的第二次确认同样默认「取消」**（`SAV-UI 6`）。
- 左右导航在两按钮间移动；上/下不离开按钮行；焦点陷阱：Tab 不得逃出模态（`PAT-02`）。
- 关闭后 `focus_restore = focus_before_open`；若打开者已禁用/移除，回父屏 `focus_order` 第一个可用控件并记 WARNING（`UIF` Formula 焦点恢复）。

### 4.3 组件清单

| 组件 | 类型 | 必需 | 备注 |
|---|---|---|---|
| `ConfirmScrim` | ColorRect | 是 | 点击不关闭 |
| `ConfirmPanel` | PanelContainer | 是 | 单实例可复用 |
| `ConfirmTitle` | Label | 是 | 动作名，不写「确认」 |
| `ConfirmBody` | RichTextLabel / Label | 是 | 自动换行；不滚动 |
| `ConfirmCancel` | Button | 是 | 默认焦点 |
| `ConfirmAccept` | Button | 是 | 动词标签；危险动作有警示条 |
| `ConfirmBusyText` | Label | 否 | 提交中显示「正在处理上一笔」 |

---

## 5. 状态与变体

| 状态 | 条件 | 可见 | 可交互 | 说明 |
|---|---|---|---|---|
| `HIDDEN` | 无确认 | — | — | 单实例隐藏 |
| `OPEN` | 已打开，未提交 | 标题/正文/取消/确认 | 是 | 常规 |
| `SUBMITTING` | 已确认，业务调用未返回 | 两按钮禁用 + 状态行 | 否 | 防第二次调用 |
| `AWAIT_SECOND` | 删除生涯第一次确认完成 | 第二层确认 | 是 | `modal_depth = 2` |
| `CLOSING` | 已取消或成功 | 淡出 | 否 | ≤ 0.12 s |
| `REMOVED` | 关闭完成 | — | — | `pop_context()` 已完成 |

**状态转移**

| 从 | 事件 | 到 | 副作用 |
|---|---|---|---|
| `OPEN` | `ui_cancel` / Esc / B / 取消 | `CLOSING` | 无业务调用；焦点归还 |
| `OPEN` | `ui_accept` 于确认 | `SUBMITTING` | 禁用两按钮；记录 `activation_event_count = 1` |
| `OPEN`（删除第一次） | 确认 | `AWAIT_SECOND` | 不调 `delete_career()`；打开第二层 |
| `AWAIT_SECOND` | 取消 / `ui_cancel` | `OPEN` | 关第二层，`modal_depth` 2→1 |
| `AWAIT_SECOND` | 确认 | `SUBMITTING` | 调 `delete_career()` 恰好一次 |
| `SUBMITTING` | 业务返回成功 | `CLOSING` | 关闭并把结果交给调用方 |
| `SUBMITTING` | 业务返回失败 | `CLOSING` | 关闭；由调用方就地显示失败原因 |
| `SUBMITTING` | 再次 `ui_accept` / 点击 | 不变 | 忽略，不发起第二笔（`UIF-R22`） |
| 任意 | 顶层状态切换 | `REMOVED` | 不执行动作 |

**边界**

- **购买进行中**：`SUBMITTING` 期间所有输入只保留 `ui_cancel`？否——**提交中取消也无效**，避免「以为取消但已扣款」。`ui_cancel` 被显式吞掉并在状态行保持「正在处理上一笔」，直到业务返回。
- **业务失败**：确认框**不**变成错误框；关闭后由来源屏就地显示「未保存，购买未生效」「已满级」「本档口已看满」「存档不可写」等（`UIF-R22`）。错误信息不进入本模态。
- **删除失败**：见 OQ-5；MVP 至少保留两层模态并显示失败原因，不假装成功。
- **同一帧多次激活**：只处理第一次；`activation_event_count ≤ 1`（`UIF` Formula）。
- **无焦点**：`initial_focus = NONE` 视为缺陷，阻止打开（`UIF` Formula）。

---

## 6. 交互映射

| 操作 | 键盘 | 手柄 | 鼠标 |
|---|---|---|---|
| 左/右移动 | ← / →、A / D | 左摇杆、十字键 | 悬停 |
| 激活 | Enter / Space | A | 左键单击 |
| 取消 | Esc / Backspace | B | 点击「取消」 |
| 关闭窗口 | OS 关闭 | — | OS 关闭 |

### 6.1 `ui_cancel` 落点矩阵（精确）

| 前置 | 按键 | 结果 | 依据 |
|---|---|---|---|
| `modal_depth = 1`，确认框 | `ui_cancel`（Esc/Backspace/B） | 取消并关闭，`pop_context()`，焦点归还 | `back_target` 优先级 1（`modal_cancel_enabled = true`） |
| `modal_depth = 2`，删除第二层 | `ui_cancel` | 关第二层，回第一次确认 | 只需弹一层 |
| `modal_depth = 2`，删除第二层 | 取消按钮 | 同上 | 同上 |
| `SUBMITTING` | `ui_cancel` | **无动作**（吞掉），状态行保持 | 防「取消已扣款」歧义；见 OQ-6 |
| 任意 | Esc（同时绑定 `ui_pause`） | 只产生 `ui_cancel`，即取消 | `Modal` 中 Esc 翻译为 `ui_cancel`（ADR-0005 D3）；`ui_pause` 从不进入本帧动作集 |
| `OPEN` | 点击面板外 | 无动作 | 防止误触取消/确认；见 OQ-2 |

> `Modal` 中 Esc 取 `ui_cancel`，不是 `ui_pause`；这与 `Paused` 中 Esc 恢复相反（`INP-R2`）。确认框打开时不存在「Esc 暂停」的路径。

---

## 7. 数据与输入契约

| 值 | 来源 | 规则 |
|---|---|---|
| 标题 | 调用方传入的动作名 | 必填；不得为空；不得写「确认」 |
| 正文 | 调用方按 §2.1 生成 | 必填；只写后果；不含业务校验结论 |
| 确认按钮标签 | 调用方传入的动词 | 必填；危险动作写动词（删除/放弃/退出），不写「确定」 |
| 购买价格 / 购买后余额 | `局外升级` 目录与钱包 | 只在购买确认中显示；余额可为负，格式见下 |
| 删除范围 | `存档` 的生涯数据 | 只写定性后果，不显示纪录数值 |
| flush 结果 | `存档.flush()` | 只影响退出后的第二确认，不进入第一次确认 |
| `modal_depth` | 界面流程 | 决定能否打开与 `ui_cancel` 落点 |
| 焦点前值 | 打开者控件 id | 关闭时归还 |

**数字格式**（`UIF-UI`）：正数带 `+`、负数带 `−`（U+2212）；金额整数元；余额为 0 显示 `0`；不显示小数、千分位。购买确认中的余额是**本次购买后**的预测值，来源为钱包减价格，不由界面重新计算其他字段。

### 7.1 输入上下文

- 打开：`push_context(Modal)`，由界面流程发起；`Modal` 屏蔽 `game_skip_wave`、`ui_pause` 与一切未列动作。
- 关闭：`pop_context()`；`push` / `pop` 必须成对（`UIF-R27`）。若 `push_context` 返回 `false`（栈满），不打开并记 WARNING。
- 确认动作不改变上下文；业务调用不应在输入回调里递归打开新模态。

---

## 8. 事件

| 事件 | 何时 | 载荷 |
|---|---|---|
| `ui_button_activated(control_id)` | 取消/确认/第二次确认被激活 | `confirm.cancel` / `confirm.accept` / `confirm.delete_final` |
| `purchase(id)` | 购买确认后 | 升级 id（由 `局外升级` 校验） |
| `delete_career()` | 第二次确认后 | 无 |
| `request_transition(...)` | 放弃/退出确认后 | `abandoned` 或无 |
| `存档.flush()` | 退出确认后 | 无 |

> 不新增 GDD 未定义的事件名；失败结果通过调用方返回值和来源屏的就地提示表达，不广播「确认失败」新事件。

---

## 9. 转场与动画

- 打开：120 ms 淡入 + 面板从 98% 缩放到 100%（≤ 120 ms）；`reduce_motion` 时全部为 0（`ACC-VIS-12`）。
- 关闭：80 ms 淡出；不位移。
- 提交中不做 loading 旋转动画；只用状态行文字，避免「没有进度却一直转」。
- 动画不得延迟 `ui_cancel` 的关闭生效，也不得延迟业务调用；状态在输入帧内改变。

---

## 10. 模式应用

| 模式 | 本屏应用 | 说明 |
|---|---|---|
| **PAT-06 破坏性确认（默认取消）** | 所有当前调用点默认「取消」；标题=动作、正文=代价；确认不是第二次校验 | 删除生涯两次确认，第二次仍默认取消 |
| **PAT-02 焦点陷阱与焦点归还** | 焦点困在按钮行；关闭归还打开者；打开者失效则回父屏首个可用项 | `SUBMITTING` 期间两按钮禁用，焦点环保留在「取消」位置（不可激活）并标注状态 |
| **PAT-03 带原因的禁用态** | `SUBMITTING` 的按钮禁用需状态行说明原因；业务失败原因回来源屏显示 | 本模态不自造「不可购买」原因，原因由 `can_purchase` 拥有 |
| **PAT-07 强制模态** | 不适用：确认框始终可取消（`modal_cancel_enabled = true`） | 强制模态见 `readonly-notice.md` |

---

## 11. 无障碍

| 要求 | 落点 |
|---|---|
| `ACC-VIS-02` | 标题/正文/按钮 ≥ 20 px @720p；注释 ≥ 16 px |
| `ACC-VIS-03` | 米白底 + 墨色文字 ≥ 4.5:1；危险不靠红字 |
| `ACC-VIS-05` / `07` | 焦点环 2 px + 2 px 间距；悬停浅底、焦点外框 |
| `ACC-VIS-08` | 灰度下能读出「取消/确认」、危险动作与余额正负 |
| `ACC-VIS-09` / `10` | 150% 下正文换行、不截断、不横向滚动；按钮不换行 |
| `ACC-VIS-12` | 减少动态时无淡入/缩放 |
| `ACC-MOT-01` / `06` | 纯键盘/手柄可完成取消、确认、两次删除确认；按钮 ≥ 56 px |
| `ACC-COG-03` | 60 s 不操作不自动关闭、不自动确认 |
| `ACC-COG-08` | 金额与文案同源；正文数字来自调用方，不在字符串写死 |
| `ACC-AUD-02` / `06` | 无音频专属信息；静音下语义完整 |

**危险表达**：确认按钮的标签本身是动词（「删除生涯」「放弃本局」「退出游戏」），并配左侧 4 px 警示条与图标；颜色不是唯一通道（`ACC-VIS-06`、`ACC-VIS-08`）。在米白纸面上，警示红只用于非文本条/图标，且对比 ≥ 3:1。

---

## 12. 本地化

| Key | 中文（基准） | 约束 |
|---|---|---|
| `ui.confirm.cancel` | 取消 | ≤ 6 字 |
| `ui.confirm.purchase.title` | 购买「{name}」 | 名称 ≤ 12 字 |
| `ui.confirm.purchase.body` | 花费 {price} 元；购买后余额 {balance} 元。 | 数字占位符，不写死 |
| `ui.confirm.abandon.title` | 放弃本局 | — |
| `ui.confirm.abandon.body` | 本局不会结算，也不会写入纪录。 | — |
| `ui.confirm.quit.title` | 退出游戏 | — |
| `ui.confirm.quit.body` | 本局将作废、不记录成绩；未保存的进度可能丢失。 | — |
| `ui.confirm.delete.title` | 删除生涯？ | 问号是标题的一部分 |
| `ui.confirm.delete.body` | 旧档会保留为隔离文件，可在文件目录中找回；当前生涯将被重置。 | 避免「永久删除」的恐吓 |
| `ui.confirm.delete.final_title` | 确认删除生涯 | — |
| `ui.confirm.delete.final_body` | 主菜单将回到新生涯；此操作不可在游戏内撤销。 | — |
| `ui.confirm.busy` | 正在处理上一笔 | 只在购买提交中出现 |

- 正文按 1.4× 扩写预留；不把金额写进翻译字符串的结构里（数字是参数）。
- 「取消」与「确认」不得翻译成同形词；危险动词必须保持动作语义。

---

## 13. 验收标准

- [ ] **CONFIRM-AC01**（核心）GIVEN 任一确认调用，WHEN 打开，THEN 标题=动作名、正文=该动作的代价、按钮为「取消」「确认」、初始焦点在「取消」。
- [ ] **CONFIRM-AC02**（默认安全）GIVEN 打开确认框，WHEN 立即按 `ui_accept`，THEN 只触发取消，业务调用次数为 0。
- [ ] **CONFIRM-AC03**（取消路径）GIVEN 确认框打开，WHEN 按 Esc / Backspace / 手柄 B / 点击取消，THEN 关闭、`pop_context()` 一次、焦点回打开者、无业务调用。
- [ ] **CONFIRM-AC04**（确认路径）GIVEN 确认框打开，WHEN 激活确认，THEN 动作恰好执行一次；同一帧再次激活不产生第二次调用。
- [ ] **CONFIRM-AC05**（深度）GIVEN `modal_depth = 2`，WHEN 请求再开确认，THEN 返回 `false`、不排队、不替换，现有两层不变。
- [ ] **CONFIRM-AC06**（购买）GIVEN 升级项可购买，WHEN 确认购买，THEN 显示价格与购买后余额，调用 `purchase(id)` 一次；提交期间再次确认被忽略，状态行显示「正在处理上一笔」。
- [ ] **CONFIRM-AC07**（购买失败）GIVEN `purchase` 返回 `COMMIT_FAILED`，WHEN 确认框关闭，THEN 来源屏显示「未保存，购买未生效」并恢复按钮；确认框不显示成功。
- [ ] **CONFIRM-AC08**（放弃）GIVEN 暂停层放弃确认，WHEN 确认，THEN 请求 `Paused → Menu` 携带 `abandoned = true`，不产生 `RunResult`、不写纪录。
- [ ] **CONFIRM-AC09**（退出）GIVEN 退出确认，WHEN 确认且 flush 成功，THEN 恰好一次 `flush()` 后请求 `Quit`；flush 失败则出现「未保存，仍要退出？」第二次确认。
- [ ] **CONFIRM-AC10**（删除两次）GIVEN 设置中删除生涯，WHEN 第一次确认，THEN 不调用 `delete_career()`，打开第二层；WHEN 第二次取消，THEN 回第一次确认且无删除。
- [ ] **CONFIRM-AC11**（删除成功）GIVEN 第二次确认，WHEN 确认，THEN `delete_career()` 恰好一次，两层模态关闭，主菜单刷新为新生涯状态，设置不变。
- [ ] **CONFIRM-AC12**（无超时）GIVEN 确认框打开，WHEN 等待 60 s，THEN 仍打开、焦点不变、无自动确认/关闭。
- [ ] **CONFIRM-AC13**（鼠标）GIVEN 确认框打开，WHEN 点击面板外遮罩或背景控件，THEN 不关闭、不激活背景、不产生业务调用。
- [ ] **CONFIRM-AC14**（手柄）GIVEN 仅手柄，WHEN 完成打开→取消、打开→确认、两次删除确认，THEN 全程可操作；B 取消、A 激活。
- [ ] **CONFIRM-AC15**（焦点归还）GIVEN 打开者是「升级项按钮」，WHEN 购买后该按钮变为不可购买，THEN 焦点回升级屏首个可用控件并记 WARNING，不落在已禁用控件上。
- [ ] **CONFIRM-AC16**（文本缩放）GIVEN 150% 字号，WHEN 打开最长正文的确认框，THEN 正文完整可读、按钮不换行、无重叠/截断/横向滚动。
- [ ] **CONFIRM-AC17**（减少动态）GIVEN `reduce_motion = true`，WHEN 打开/关闭，THEN 无淡入、无缩放，状态 1 帧内生效。
- [ ] **CONFIRM-AC18**（灰度）GIVEN 确认框截图去色相，WHEN 检查，THEN 能区分取消/确认、危险动作与余额正负，不靠颜色。
- [ ] **CONFIRM-AC19**（无 Esc 歧义）GIVEN 确认框打开，WHEN 按 Esc，THEN 只取消确认，不暂停/恢复游戏、不改变顶层状态。
- [ ] **CONFIRM-AC20**（上下文配对）GIVEN 连续打开/取消确认 10 次，WHEN 检查输入栈，THEN 每次 `push_context(Modal)` 均有 `pop_context()`，最终栈深回到打开前基线。

---

## 14. 开放问题

| # | 问题 | 双方依据 | 当前处理 |
|---|---|---|---|
| 1 | 长后果文案的最大宽度、换行与是否允许滚动？ | `interaction-patterns.md` Gaps 表 `ConfirmModal` 待定 vs `UIF-UI` 字号要求 | 本规格定为宽 ≤ 560 px、正文 ≤ 3 行（100%）；超出由文案 lint 拦截，不滚动。需 `ux-designer` 复核德语/俄语 |
| 2 | 点击遮罩是否应视为取消？ | `PAT-06` 默认安全 vs 误触风险 | 本规格定为**不关闭**；只有按钮与 Esc/B 可取消。若用户测试显示困惑再改 |
| 3 | 非破坏性确认是否允许默认「确认」？MVP 没有此类调用点。 | `UIF` `initial_focus`（破坏性以取消开头）vs `PAT-06` | 本规格要求所有调用点默认取消；未来新增非破坏性确认需先补模式 |
| 4 | 删除生涯入口放在设置还是主菜单二级？ | `SAV-UI 6`（「设置或主菜单」）vs `UIF-R24`（固定在设置次级分区） | 本规格按 `UIF-R24` 放设置；若主菜单也放入口，需统一为同一模态与同一文案 |
| 5 | `delete_career()` 失败时的界面行为未定义：关闭两层？保留第二层？刷新到什么状态？ | `SAV-UI 6` 只定义成功路径 vs `UIF-R24` | 本规格暂定保留第二层并就地显示失败原因，不假装成功；需存档/界面流程确认 |
| 6 | 购买提交中按取消应无效还是允许「撤销」？提交是异步还是同步？ | `UIF-R22`（进行中再次确认被忽略）vs `PAT-06`（取消应始终可用） | 本规格暂定提交中两按钮禁用、`ui_cancel` 吞掉；若 `purchase` 实际同步返回，`SUBMITTING` 只存在 1 帧，可简化 |
| 7 | `BUSY` 的上限时长与超时文案：若提交长时间不返回，状态行会一直「正在处理上一笔」，无 spinner 也无出口。 | `UIF-R22` vs `ACC-COG-03`（无时限） | 未定；需要 `purchase` 的时限契约。是否允许超时后回到可取消态，见 `ui-programmer` |
| 8 | 窗口关闭发生在 `Run` 时，确认框叠在未冻结的局上；回合制无计时，但「局在背后仍可操作」是否可接受？ | `SCN-R10` vs `UIF-R16` | 本规格只定义模态行为；是否先暂停归场景流程与 `pause-overlay.md` OQ-6 |
| 9 | 警示红在深色面板不达标（2.9:1），本规格改用米白面板；若全局模态统一为深色，危险表达需换方案。 | `ACC-VIS-04` vs 现有暂停层深色面板 | 本规格选米白纸面以复用价格板色板；需 `art-bible` 确认 |
| 10 | 第二次删除确认的「隔离文件」对玩家是否过于技术化？是否需要「可在文件目录找回」的示例路径？ | `SAV-UI 6` 明文要求 vs `ACC-COG-08` 术语一致 | 保留 GDD 原意；示例路径会泄露用户名，不显示。文案待用户测试 |
