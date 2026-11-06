# Error Screen Design（错误屏）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Game**: 回本 (Break Even)
> **Platform Targets**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主，手柄部分支持；无触屏。参考 1280×720
> **Related GDDs**: `scene-flow.md`（SCN-R3/4/9/10）、`ui-flow.md`（UIF-R18）、`data-config.md`（启动校验）、`input.md`（Error → Menu 上下文）、`audio-feedback.md`
> **Accessibility Tier**: Standard（逐条要求见 `design/accessibility-requirements.md`）
> **Sources**: `design/ux/interaction-patterns.md`（PAT-07/09）、`design/ux/hud.md`（价位板排版基线）、ADR-0005（`Menu` 上下文仅用于单按钮导航）
> **Sibling Specs**: `design/ux/pause-overlay.md`、`design/ux/confirm-modal.md`、`design/ux/readonly-notice.md`、`design/ux/onboarding-layer.md`、`design/ux/hud.md`、`design/ux/interaction-patterns.md`
> **Template**: UX Screen Spec

> **范围边界**：本文件只覆盖**顶层状态 `Error`** 的那一屏：启动期数据校验失败后，游戏无法进入主菜单时的唯一界面。它**不覆盖**局内故障/进入失败的非模态提示（`UIF-R19`，归主菜单与 `NoticeLayer`）、不覆盖存档 `Blocked` 模态（归 `readonly-notice.md`）、不覆盖启动等待 `BootWaiting`（无按钮、无进度条）。

## 引用代号

| 代号 | 出处 |
|---|---|
| `SCN-Rn` / `SCN-ACn` | `design/gdd/scene-flow.md` Core Rules 第 n 条 / Acceptance Criteria 第 n 条 |
| `UIF-Rn` / `UIF-UI` | `design/gdd/ui-flow.md` Core Rules 第 n 条 / UI Requirements 表 |
| `INP-Rn` | `design/gdd/input.md` Core Rules 第 n 条 |
| `DAT-Rn` | `design/gdd/data-config.md` Core Rules 第 n 条 |
| `ADR-0005` | `docs/architecture/adr-0005-input-context-stack.md` |
| `PAT-nn` | `design/ux/interaction-patterns.md` |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. 目的与玩家需求

错误屏不是「主菜单的变体」，而是**一扇只能出去的门**。它出现时，游戏已经无法进入 `Menu` 或 `Run`；玩家能做的只有退出（debug 构建额外可以重走一次加载）。因此它的设计目标不是「让玩家继续玩」，而是：

1. **说清楚发生了什么**——用玩家能懂的话，不甩错误码。
2. **给出唯一确定的选择**——「退出」；不提供「重试」「忽略」「用默认数据进入」等假希望（`SCN-R9`、`UIF-R18`）。
3. **保持平静**——不是惩罚、不是崩溃恐吓；价格板式的排版、稳定的米白纸面、无红色闪烁。
4. **可支持**——debug 构建把失败文件与记录 id 留在屏幕上，方便 QA；release 不泄露路径与内部标识。

> **设计立场**：错误屏是**价目板上贴的一张停业通知**，不是警报器。玩家此刻已经受挫，界面不能再制造第二层焦虑。

---

## 2. 进入与退出契约

### 2.1 进入

| 来源 | 触发 | 载荷 | 说明 |
|---|---|---|---|
| `Boot` | 数据配置加载/校验 `Failed` | 失败明细（仅 debug 展示） | `SCN-R4`：`Boot → Error`；不进入 `Menu`/`Run` |
| `Boot`（debug 重载后再次失败） | 再次 `Failed` | 新明细 | 同一屏重新渲染，不累加历史 |

**不可进入 `Error` 的其他路径**：局内故障、进入失败、结算异常都回 `Menu` 并以非模态提示呈现（`UIF-R19`）；它们**不**把整局/整屏变成 `Error`（`SCN-R9` 的受限态只用于启动校验失败）。

### 2.2 退出

| 出口 | 触发 | 目标 | 语义 |
|---|---|---|---|
| 退出 | 激活「退出」/ `ui_accept` | `Quit` | `Error` 中退出**不需确认**（`SCN-R10`：启动阶段没有一局可损失） |
| 重新加载数据（仅 debug） | 激活第二按钮 | `Boot` | 重走加载与校验（`SCN-R9`） |
| 窗口关闭 | OS 关闭事件 | `Quit` | 与游戏内退出同规则（`SCN-R10`） |

**不存在的出口**：进入 `Menu`、进入 `Run`、重试（release）、使用默认数据、忽略损坏文件、打开设置（设置也在 `Menu` 内）。release 构建中「重新加载数据」不是存在后被拒绝，而是**根本不存在**（`UIF-R18`）。

---

## 3. 导航位置与输入上下文

- `Error` 是顶层状态；本屏**不进入**屏幕栈，`ui_stack_depth = 0`，没有返回栈。
- 输入上下文为 **`Menu`**，唯一目的是让按钮可以被导航与确认（`SCN-R3`）；`Error` 内不存在任何能触发 `Menu` 或 `Run` 的动作。
- `Menu` 上下文屏蔽 `ui_pause` 与 `game_skip_wave`；Esc 在该上下文被翻译为 `ui_cancel`（`ADR-0005` D3），而 `ui_cancel` 在本屏无效。
- 没有模态、没有 `push_context`；不要在 `Error` 上再压 `Modal`。

---

## 4. 布局

### 4.1 线框（1280×720，release）

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│                                                              │
│              游戏数据未能加载                                │  ← 标题 32 px
│                                                              │
│              启动时读取游戏数据失败。                         │  ← 正文 20 px
│              请重新启动游戏；如果问题仍然存在，               │
│              请检查安装文件是否完整。                         │
│                                                              │
│                        [ 退出 ]                              │  ← 唯一按钮
│                                                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 布局区

| 区 | 内容 | 焦点 | 说明 |
|---|---|---|---|
| Z1 标题 | 「游戏数据未能加载」 | 不可达 | 不写「错误」「崩溃」「Fatal」 |
| Z2 正文 | 两到三句可读说明 + 玩家可做的事 | 不可达 | 不显示错误码、路径、记录 id |
| Z3 动作 | 「退出」 | 可获焦 | release 唯一控件；debug 追加「重新加载数据」 |
| Z4 debug 详情（仅 debug） | 失败文件、记录 id、失败类型 | 不可达（文本） | 等宽 16 px；可换行 |

### 4.3 尺寸与排版

- 内容列宽 `min(0.60 × W, 760)`，水平居中；垂直方向在安全区内居中偏上（标题基线约 38% H）。
- 标题 32 px / 行高 40；正文 20 px / 行高 30；按钮高 56 px、最小宽 160 px（`ACC-MOT-06`）。
- 米白纸面、墨色文字；标题左侧 4 px 中性橄榄绿竖线（非文本），**不使用红色、不使用闪烁**。
- 按钮在正文下方居中；按钮与正文间距 32 px。
- 150% 字号：列宽上限提高到 `0.72 × W`，正文换行；按钮随文字增高；不截断、不横向滚动（`ACC-VIS-10`）。debug 路径允许在任意字符处换行，或中段省略，见 OQ-6。

### 4.4 焦点顺序

| 构建 | `focus_order` | 初始焦点 |
|---|---|---|
| release | `[退出]` | 退出 |
| debug | `[退出, 重新加载数据]` | 退出（不因 QA 方便而改变默认） |

- 上下/左右导航只在可用项之间移动；release 中只有一个按钮，方向键不产生位移。
- 鼠标悬停与键盘焦点分开绘制（`INP-R8`、`PAT-01`）：悬停浅底、焦点外框。
- 焦点环 2 px 实线 + 2 px 间距，永不被裁切（`ACC-MOT-06`）。

---

## 5. 状态与变体

| 状态 | 条件 | 可见 | 可交互 |
|---|---|---|---|
| `SHOWN` | 进入 `Error`，release | 标题/正文/退出 | 是 |
| `SHOWN_DEBUG` | 进入 `Error`，debug | 标题/正文/退出/重新加载/debug 详情 | 是 |
| `RELOADING` | debug 点了「重新加载数据」 | 两按钮禁用 + 「正在重新加载…」 | 否 |
| `CLOSING` | 已请求 `Quit` | 淡出 | 否 |
| `REMOVED` | 状态离开 `Error` | — | — |

**转移**

| 从 | 事件 | 到 | 副作用 |
|---|---|---|---|
| `SHOWN` | `ui_accept` 于退出 | `CLOSING` | `request_transition(Quit)` 恰好一次 |
| `SHOWN` | 窗口关闭 | `CLOSING` | 同上 |
| `SHOWN_DEBUG` | `ui_accept` 于退出 | `CLOSING` | 同上 |
| `SHOWN_DEBUG` | `ui_accept` 于重载 | `RELOADING` | `request_transition(Boot)` 恰好一次；重复激活忽略 |
| `RELOADING` | `Boot` 校验成功 | 离开本屏 | 进入 `Menu`（由场景流程推进，本屏不绘制主菜单） |
| `RELOADING` | 再次 `Failed` | `SHOWN_DEBUG` | 新明细替换旧明细；不累加日志 |
| 任意 | 状态离开 `Error` | `REMOVED` | 立即拆除，不等待动画 |

**边界**

- **`ui_cancel` 不产生隐藏动作**（`UIF-R18`）：Esc/Backspace/B 不关闭、不退出、不重载、不打开任何东西。
- **无计时**：不会在 N 秒后自动退出；等待 60 s 界面不变（`ACC-COG-03`）。
- **无焦点**：release 只有一个按钮，仍必须有可见焦点；`initial_focus = NONE` 视为缺陷。
- **重载失败**：回到 `SHOWN_DEBUG`，正文可追加「仍然失败」一行；按钮恢复可用（见 OQ-4）。
- **退出失败？** `Quit` 是终态，没有失败路径；若 OS 阻止关闭，由平台处理，不在本屏显示。

---

## 6. 交互映射

| 操作 | 键盘 | 手柄 | 鼠标 |
|---|---|---|---|
| 移动 | ↑ / ↓、W / S | 摇杆/十字键 | 悬停 |
| 激活 | Enter / Space | A | 左键单击 |
| 退出 | 激活唯一/首个按钮 | A | 点击「退出」 |
| 返回 / 取消 | Esc / Backspace | B | 无 |
| 暂停 | 无（`Menu` 屏蔽 `ui_pause`） | 无 | — |

### 6.1 `ui_cancel` 落点矩阵（精确）

| 前置 | 按键 | 上下文翻译 | 结果 |
|---|---|---|---|
| `Error` | Esc | `ui_cancel`（`Menu` 上下文；`ADR-0005` D3） | `NONE`：无动作；焦点环重绘一次 |
| `Error` | Backspace | `ui_cancel` | `NONE`：同上 |
| `Error` | 手柄 B | `ui_cancel` | `NONE`：同上 |
| `Error` | P / 手柄 Start | `ui_pause` 被 `Menu` 屏蔽 | 无动作 |
| `Error`，debug | `ui_cancel` | `ui_cancel` | `NONE`：不关闭重载确认（本屏没有确认） |
| `Error` | 窗口关闭 | 平台事件 | 直接 `Quit` |

**NONE 的可见性**：`back_target` 规则 9 要求 `NONE` 产生**可见的「无动作」**而不是静默吞键。本屏的实现是：按下取消时让当前焦点环重绘一次（不变粗、不位移，100 ms 内恢复；`reduce_motion` 时单帧高亮），不播放声音、不显示新文案。这与 `UIF-R18` 的「`ui_cancel` 不产生隐藏动作」一致；是否足够，见 OQ-2。

---

## 7. 数据契约

| 值 | 来源 | release | debug |
|---|---|---|---|
| 失败事实 | `Boot` 数据校验结果 `Failed` | 显示通用标题/正文 | 同左，另加详情 |
| 失败类型 | 校验器返回的枚举/描述 | 映射为通用文案 | 显示原始类型名 |
| 失败文件 | 校验器 | **不显示** | 显示路径（必要时中段省略） |
| 记录 id | 校验器 | **不显示** | 显示原始 id |
| 重载结果 | `Boot` 再次校验 | 不适用（按钮不存在） | 成功离开 / 失败回显 |
| 按钮可用性 | 构建类型 + 状态 | 仅退出 | 退出 + 重载 |

- 本屏**不读**存档、钱包、升级、纪录、`LoadReport` 或任何局内数据；数据配置尚未 `Ready`，读了也不可信。
- 不显示「第 N 个文件」「第 N 行」这类内部定位给 release；debug 才有。
- 文案由本地化表按失败类型映射；未知类型回退到通用文案，不显示空白（见 OQ-3）。
- **无数字**：不显示版本号、构建号、校验和；debug 的失败文件/记录 id 不是玩法数字。

### 7.1 输入上下文

- 进入 `Error` 时场景流程调用 `request_context(Menu, clear_stack := true)`（`SCN-R3`）；本屏不压/弹任何上下文。
- `Menu` 允许导航、`ui_accept`、`ui_cancel`；`ui_cancel` 到达本屏后被显式视为无操作，不投递给任何其他系统。
- `Error` 中没有任何模态，`modal_depth = 0`；不要用 `Modal` 来实现「退出确认」，因为退出不需要确认。

---

## 8. 事件与音频

| 事件 | 何时 | 载荷 |
|---|---|---|
| `ui_button_activated("error.quit")` | 激活退出 | 音效可选订阅 |
| `ui_button_activated("error.reload")` | debug 激活重载 | 同上 |
| `state_changed(Boot, Error)` | 进入本屏 | 场景流程广播 |
| `request_transition(Quit)` / `request_transition(Boot)` | 出口 | 场景流程 |

- 不新增「错误音效」：`sfx.run.fault` 只映射 `run_faulted`，启动校验失败不属于该事件；本屏默认安静，静音下信息完整（`ACC-AUD-02`/`06`）。
- 若未来加入错误提示音，必须平静、低存在感，且不得成为唯一通知（`AUD` 规则 13/20 的精神）。

---

## 9. 转场与动画

- 进入：120 ms 淡入；离开：80 ms 淡出；`reduce_motion` 时均为 0（`ACC-VIS-12`）。
- 不位移、不缩放、不闪烁、不抖动；不做「错误震动」。
- 重载期间不显示 spinner；只显示静态状态行「正在重新加载…」，避免无限旋转。
- 转场不得延迟 `Quit` / `Boot` 的状态生效（`SCN-R12`）。

---

## 10. 模式应用

| 模式 | 本屏应用 | 说明 |
|---|---|---|
| **PAT-07 强制模态与出口矩阵** | 本屏是**受限态**而非模态：只有一个安全出口，`ui_cancel` 无效；不存在可取消的路径 | 与 `BlockedModal` 的区别：本屏是顶层状态、没有背景界面、没有重试（release） |
| **PAT-09 显式空状态** | 用完整句子说明「无法加载」，不显示空白表格、不显示占位数字 | 空状态不是「没有数据」，而是「这扇门关了」 |
| **PAT-03 带原因的禁用态** | 本屏没有禁用控件；debug 重载在 `RELOADING` 时禁用并给状态行原因 | release 不显示「重载」再置灰——它根本不存在 |
| **PAT-13 返回决策** | `Error` 在 `back_target` 中属于「其余 → NONE」；没有返回栈 | 不适用屏幕根替换 |
| **PAT-01 双焦点** | 悬停与焦点分开绘制；只有一个按钮时仍保持两种状态可辨 | 纯鼠标用户不需要先移动鼠标 |

---

## 11. 无障碍

| 要求 | 落点 |
|---|---|
| `ACC-VIS-02` | 标题 32 / 正文 20 / 按钮 20 px @720p；debug 详情 ≥ 16 px |
| `ACC-VIS-03` | 米白底 + 墨色文字 ≥ 4.5:1；竖线非文本 ≥ 3:1 |
| `ACC-VIS-05` / `07` | 焦点环 2 px + 2 px 间距；悬停浅底、焦点外框；单按钮也不得无焦点 |
| `ACC-VIS-08` | 灰度下能读出标题、说明与「退出」；失败不靠红色 |
| `ACC-VIS-09` / `10` | 100/125/150% 三档；150% 下正文换行、按钮不截断、无横向滚动 |
| `ACC-VIS-12` | 减少动态无淡入淡出、无焦点环过渡 |
| `ACC-VIS-13` | 无闪光、无频闪、无屏幕震动 |
| `ACC-MOT-01` / `06` | 纯键盘/手柄可退出（debug 可重载）；按钮 ≥ 56 px；焦点环不被裁切 |
| `ACC-COG-02` / `03` | 无计时、不自动退出；等待 60 s 界面不变 |
| `ACC-COG-08` | 文案与术语一致；不显示内部错误码（debug 除外） |
| `ACC-AUD-02` / `06` | 无音频专属信息；静音下可完成退出 |

**认知负荷**：本屏最多两个按钮、三句正文。不再提供「查看日志」「复制错误码」「打开帮助」等入口；支持信息只在 debug 构建出现。

---

## 12. 本地化

| Key | 中文（基准） | 约束 |
|---|---|---|
| `ui.error.title` | 游戏数据未能加载 | ≤ 12 字；不写「错误」「崩溃」 |
| `ui.error.body.generic` | 启动时读取游戏数据失败。请重新启动游戏；如果问题仍然存在，请检查安装文件是否完整。 | 两句话，≤ 60 字 |
| `ui.error.body.still_failing` | 仍然无法加载游戏数据。 | debug 重载失败后追加 |
| `ui.error.quit` | 退出 | ≤ 6 字 |
| `ui.error.reload` | 重新加载数据 | debug only，≤ 8 字 |
| `ui.error.reloading` | 正在重新加载… | 状态行 |
| `ui.error.debug.file` | 失败文件：{path} | debug only |
| `ui.error.debug.record` | 记录 id：{id} | debug only |
| `ui.error.debug.kind` | 失败类型：{kind} | debug only |

- release 不渲染 debug key；本地化表可以包含它们，但构建期应剔除或只在 debug 加载。
- 不把路径拼进翻译句子中间作为不可换行的整体；路径是参数，允许换行。
- 文案避免「致命」「损坏」「崩溃」等恐吓词；除非失败类型确实是文件损坏，且仍用「无法读取」表述。

---

## 13. 验收标准

- [ ] **ERR-AC01**（核心）GIVEN 启动数据校验失败，WHEN 进入 `Error`，THEN 显示标题、可读说明与唯一「退出」按钮；不存在进入 `Menu`/`Run` 的控件。
- [ ] **ERR-AC02**（状态）GIVEN 进入 `Error`，WHEN 检查场景，THEN 顶层状态为 `Error`、输入上下文为 `Menu`、`ui_stack_depth = 0`、`modal_depth = 0`。
- [ ] **ERR-AC03**（退出）GIVEN `Error` 显示，WHEN 激活「退出」或按 `ui_accept`，THEN 请求 `Quit` 恰好一次，不弹确认。
- [ ] **ERR-AC04**（窗口关闭）GIVEN `Error` 显示，WHEN 触发窗口关闭，THEN 直接进入 `Quit`，与其他状态的退出规则一致。
- [ ] **ERR-AC05**（取消无效）GIVEN `Error` 显示，WHEN 按 Esc/Backspace/手柄 B，THEN 不退出、不重载、不关闭、不打开任何界面；焦点环重绘一次作为可见无动作。
- [ ] **ERR-AC06**（无暂停）GIVEN `Error` 显示，WHEN 按 P/Start，THEN 无动作，不进入 `Paused`。
- [ ] **ERR-AC07**（release 无重载）GIVEN release 构建，WHEN 遍历控件，THEN 只有「退出」；不存在「重新加载数据」节点，而不是禁用节点。
- [ ] **ERR-AC08**（debug 重载）GIVEN debug 构建，WHEN 激活「重新加载数据」，THEN `request_transition(Boot)` 恰好一次、按钮禁用、显示「正在重新加载…」。
- [ ] **ERR-AC09**（重载成功）GIVEN debug 重载后校验通过，WHEN 状态切换，THEN 离开 `Error` 进入 `Menu`；本屏不绘制主菜单。
- [ ] **ERR-AC10**（重载失败）GIVEN debug 重载后再次失败，WHEN 回到 `Error`，THEN 显示新失败明细、不累加旧明细、按钮恢复可用。
- [ ] **ERR-AC11**（初始焦点）GIVEN 任意构建，WHEN 进入 `Error`，THEN 焦点在「退出」；debug 的第二按钮不抢初始焦点。
- [ ] **ERR-AC12**（鼠标）GIVEN `Error` 显示，WHEN 点击空白或标题，THEN 无动作、无焦点变化、无退出。
- [ ] **ERR-AC13**（手柄）GIVEN 仅手柄，WHEN 进入并退出（debug 再重载一次），THEN 全程可操作，A 激活、B 无动作。
- [ ] **ERR-AC14**（无计时）GIVEN `Error` 显示，WHEN 等待 60 s，THEN 不自动退出、不自动重载、焦点不变。
- [ ] **ERR-AC15**（文本缩放）GIVEN 150% 字号，WHEN 显示最长文案与 debug 路径，THEN 正文/路径完整可读、按钮不截断、无横向滚动。
- [ ] **ERR-AC16**（灰度/对比度）GIVEN 去色相截图，WHEN 检查，THEN 能读出标题、说明、按钮与焦点；无红色、无闪烁；文字对比 ≥ 4.5:1。
- [ ] **ERR-AC17**（隐私）GIVEN release 构建，WHEN 检查全部可见文本，THEN 不出现文件路径、记录 id、错误码、版本号或用户名。
- [ ] **ERR-AC18**（性能）GIVEN 60 fps，WHEN 进入/退出，THEN 单帧开销不导致掉帧；转场不延迟状态生效。

---

## 14. 开放问题

| # | 问题 | 双方依据 | 当前处理 |
|---|---|---|---|
| 1 | debug 信息区是否折叠？Gaps 表把它列为待定；折叠会多一个交互与焦点项，不折叠会让长路径占屏。 | `interaction-patterns.md` Gaps 表 `ErrorScreen` 待定 vs `UIF-R18`（debug 展示失败文件与记录 id） | 本规格**不折叠**，debug 区常驻可读；若路径过长用中段省略。折叠需新增控件与 AC，暂不做 |
| 2 | `ui_cancel` 的「可见无动作」是否必要？GDD 只说不产生隐藏动作；PAT-07 要求 `NONE` 可见。 | `UIF-R18` / `UIF` `back_target` 注 vs `PAT-07` | 本规格用焦点环重绘；若用户测试认为像无响应，可改为静态文案「此界面只能退出」（不新增按钮） |
| 3 | 失败文案的映射归谁？数据配置只返回失败明细，玩家可见文案表由 `ui-flow`/`/ux-design` 拥有，但当前没有 `error_text` 表。 | `UIF-R18` / `DAT-Rn` vs `ACC-COG-08` | 本规格要求新增 `ui.error.*` 文案表；未知失败类型回退通用文案。需数据配置提供稳定枚举 |
| 4 | debug「重新加载数据」失败后是否允许无限重载？是否要显示尝试次数或退避？ | `SCN-R9`（debug 可重走校验）vs `ACC-COG-02` | 本规格允许无限重载、无超时；不新增计数。若 QA 需要，可加 debug-only 计数 |
| 5 | release 是否需要给玩家一个**非敏感**的支持编号（如构建号/短哈希），方便反馈？GDD 只允许 debug 显示文件与记录 id。 | `UIF-R18`（release 只有可读说明）vs 支持需求 | 不加；加编号会与「不显示内部标识」的隐私取舍冲突，需 producer 裁定 |
| 6 | 150% 下 debug 路径的换行规则：任意字符换行、按 `/` 分段换行，还是中段省略？ | `ACC-VIS-10`（不横向滚动、不消失）vs 路径可读性 | 本规格允许中段省略并保留文件名；完整路径不换行到按钮区。需用户测试 |
| 7 | `Error` 只能从 `Boot` 进入是否足够？若运行期数据配置整份失效，是否也应进入 `Error` 而不是主菜单提示？ | `SCN` 状态表（Error 仅启动校验失败）vs `UIF-R19`（局内故障回 Menu 提示） | 本规格按 GDD：运行期故障不进入 `Error`；这是「不让一局因单条数据坏掉而整屏报废」的取舍 |
| 8 | 退出按钮是否应该二次确认，防止误按 Enter 直接退出？ | `SCN-R10`（Error 退出不需确认）vs `PAT-06`（破坏性动作默认取消） | 本规格按 GDD 不确认；此处没有可损失的一局，确认只会增加摩擦 |
| 9 | 错误屏是否需要背景视觉（价目板纹理/插画）？ | `ART-§1` 价位板视觉 vs 加载失败时资产可能不可用 | 本规格只用主题色与排版，不依赖任何可能加载失败的贴图；保持纯色纸面 |
| 10 | 手柄 Start 在 `Menu` 上下文被屏蔽，玩家可能尝试用 Start 退出；是否需要把 Start 也映射为「退出」？ | `INP` 动作表（Start = `ui_pause`）vs 本屏无暂停 | 不映射；屏幕只有一个按钮，A 即可退出，改映射会破坏全局动作语义 |
