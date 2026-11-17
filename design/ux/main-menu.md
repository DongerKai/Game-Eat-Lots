# UX Spec: 主菜单（MainMenu）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Journey Phase(s)**: 局外 / 每局之间（`Menu` 根屏）
> **Screen / Flow Name**: `MainMenu`
> **Platform Target**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主、手柄部分支持（`Gamepad Support: Partial`）、无触屏；参考分辨率 1280×720，支持至 3840×2160 与 21:9
> **Related GDDs**: `design/gdd/ui-flow.md`（屏幕契约与规则 3–8、13、19–21、25–28）、`design/gdd/input.md`、`design/gdd/meta-upgrades.md`、`design/gdd/records-and-milestones.md`、`design/gdd/save-system.md`、`design/gdd/settlement-and-entry-fee.md`、`design/gdd/first-run-onboarding.md`、`design/gdd/unlocks-and-stations.md`、`design/gdd/run-flow.md`、`design/gdd/scene-flow.md`
> **Sources**: 上述 GDD 的 `## UI Requirements`；`design/ux/interaction-patterns.md`；`design/ux/hud.md`（结构与质量基线，未复制内容）；`design/accessibility-requirements.md`；`.claude/docs/technical-preferences.md`
> **Sibling Specs**: `design/ux/hud.md`、`interaction-patterns.md`、`receipt-screen.md`、`upgrade-screen.md`、`records-screen.md`、`settings-modal.md`。`ConfirmModal`、`ReadOnlyNotice` / `BlockedModal` 的版式归其各自规格，本文件只定义从本屏进入时的状态与焦点归还
> **Accessibility Tier**: Standard（承诺与逐条要求见 `design/accessibility-requirements.md`）
> **Style Reference**: `design/art/art-bible.md` § 4 Color System；**§ 7 UI/HUD Visual Direction 尚未撰写**（见 Open Questions）
> **Template**: UX Spec

> **范围边界**：本文件只覆盖玩家**不在局内**时的菜单根屏：店招、入场费、钱包、五个入口、保存指示与载入/进入结果提示。升级、纪录、设置、收据各有独立规格；退出确认、只读说明、强制 `Blocked` 说明是模态/提示层，本文件只规定它们与主菜单的交互边界。判断标准：玩家此刻是否在决定“开始下一局 / 查看账本 / 改设置 / 离开”。

## 引用代号

| 代号 | 出处 |
|---|---|
| `UIF-Rn` / `UIF-ACn` | `design/gdd/ui-flow.md` Core Rules 第 n 条 / Acceptance Criteria 第 n 条 |
| `UIF-UI` | 同文件 `## UI Requirements` 的屏幕契约表 |
| `INP-Rn` / `INP-UI` | `design/gdd/input.md` |
| `META-Rn` / `META-UI` | `design/gdd/meta-upgrades.md` |
| `REC-Rn` / `REC-UI` | `design/gdd/records-and-milestones.md` |
| `SAVE-Rn` / `SAVE-UI` | `design/gdd/save-system.md` |
| `SETT-Rn` / `SETT-UI` | `design/gdd/settlement-and-entry-fee.md` |
| `ONB-Rn` / `ONB-S9` | `design/gdd/first-run-onboarding.md`（S9 为主菜单入场费一致性要求） |
| `UNL-UI` | `design/gdd/unlocks-and-stations.md` UI Requirements |
| `PAT-NN` | `design/ux/interaction-patterns.md` 模式编号 |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. Purpose & Player Need

主菜单是玩家的**柜台**：进店前先看清两件事——这一局要付多少钱、我账上还有多少钱。它不是封面，也不是导航页；它是一张写着今日价格与余额的价目板，五个入口只是柜台上的五个抽屉。玩家在这里做的决定只有一个：现在开始下一局，还是先看账本/升级。因此屏幕的第一顺位不是“好看”，而是**入场费与钱包必须同屏、可读、可比较**（`SETT-UI`、`UIF-R8`）。

**The player goal**：在不进入任何子屏的情况下，确认“这局入场费多少、我还剩多少”，然后开始营业。

**The game goal**：把本局实际会扣的入场费与当前钱包如实交出，让玩家在付费前知道价格，并让所有写盘依赖的入口在不可写时说出原因而不是静默失效。

---

## 2. Player Context on Arrival

| Question | Answer |
|---|---|
| What was the player just doing? | 启动完成（`Boot → Menu`）、看完一张收据、从升级/纪录返回、取消退出确认，或刚从一次进入失败/放弃回到菜单 |
| What is their emotional state? | 平静、精算；刚亏完一局时可能带着“还差多少”的挫败感 |
| What cognitive load are they carrying? | 低：只需比较两个数（入场费、钱包）。若刚从收据回来，脑内还留着净利与回本率 |
| What information do they already have? | 上一局的净利与是否回本；钱包在收据上已经出现过一次 |
| What are they most likely trying to do? | 直接开始下一局；或在亏钱后先看升级/纪录 |
| What are they likely afraid of? | “进去才发现钱不够/扣得比显示的多”“点了没反应”“存档是不是坏了” |

**Emotional design target**：像站在食堂柜台前看价目表——安静、透明、没有催促；任何时候都能读清价格与余额。

---
## 3. Navigation Position

```
Menu（顶层状态根屏）
  └── MainMenu（本屏，ui_stack_depth = 1）
        ├── UpgradeScreen（子屏，深度 2）
        ├── RecordsScreen（子屏，深度 2）
        ├── SettingsModal（模态，深度 1；本规格）
        ├── ConfirmModal（退出确认，模态，深度 1）
        ├── ReadOnlyNotice（非模态说明，含两个出口）
        └── BlockedModal（强制模态，深度 1）
```

**Modal behavior**：主菜单本身不是模态；设置与退出确认是模态，打开时焦点移入并在内部循环，关闭后回到打开它的按钮（`PAT-02`、`UIF-R6`）。`BlockedModal` 未解决前主菜单其它控件不可操作（`UIF-R20`）。`ReadOnlyNotice` 是非模态说明，不阻塞布局（`PAT-08`），其焦点语义见 Open Questions 第 2 条。

**Reachability — all entry points**：

| Entry Point | Triggered By | Notes |
|---|---|---|
| 启动完成 | `Boot → Menu`（`Fresh` / `Loaded` / `Migrated` / `Recovered` / `ReadOnly` / `Blocked`） | 载入异常在本屏进入后显示（`UIF-R20`） |
| 收据返回 | 收据「返回菜单」替换根屏（`UIF-R13`） | 不再结算，焦点回「开始营业」 |
| 子屏返回 | 升级/纪录 `ui_cancel` 或「返回」 | `POP_SCREEN`（`UIF-R7`） |
| 模态关闭 | 设置/退出确认取消 | 焦点归还打开者 |
| 放弃/故障回菜单 | `Paused → Menu, abandoned = true`；`init_failed` / `fault.phase` | 只给非模态提示，不给收据（`UIF-R19`） |
| 删除生涯后重建 | `SettingsModal` 内删除生涯成功 | 以新生涯状态重建主菜单（`UIF-R24`） |

---

## 4. Entry & Exit Points

**Entry table**：

| Trigger | Source Screen / State | Transition Type | Data Passed In | Notes |
|---|---|---|---|---|
| 启动完成 | `Boot` | 根屏替换（场景淡入 ≤ 0.2 s） | `LoadReport`、`records.snapshot()`、`progression.get_wallet()` | 无加载屏（`UIF-R17`、`RUN-UI 5`） |
| 收据返回 | `ReceiptScreen` | 根屏替换 | 已提交的钱包/纪录；无 `RunResult` 再结算 | 焦点「开始营业」 |
| 子屏返回 | 升级/纪录 | 弹出 | — | 保留菜单滚动/焦点上下文（本屏无滚动） |
| 放弃/进入失败 | `Paused` / `Run` | 根屏切换 | `abandoned` / `init_failed` / `fault` | 文案按 `fault.phase` 分支（`UIF-R19`） |
| 删除生涯成功 | `SettingsModal` | 同屏刷新 | 新生涯默认值 | 设置值不变（`SAVE-R23`） |

**Exit table**：

| Exit Action | Destination | Transition Type | Data Returned / Saved | Notes |
|---|---|---|---|---|
| 开始营业 | `Run` | 请求 `Menu → Run`，携带 `RunConfig` | `RunConfig`（入场费 override 取自 `plan.has_entry_fee_override()` 时的 `plan.entry_fee_override`，否则 `null`；升级快照副本） | 请求失败留在本屏并给原因（`UIF-R8`） |
| 升级 | `UpgradeScreen` | 推入子屏 | 只读钱包/目录 | 深度 ≤ 2（`UIF-R4`） |
| 纪录 | `RecordsScreen` | 推入子屏 | 只读纪录快照 | 深度 ≤ 2 |
| 设置 | `SettingsModal` | 打开模态 | 设置模型 | 关闭时写机器级设置（`UIF-R24`） |
| 退出 | 退出确认 → `Quit` | 模态确认后 `flush()` | `flush()` 结果 | 本屏退出无需确认；`ui_cancel` 走确认（OQ8） |
| `ui_cancel` | 退出确认 | 模态 | — | 绝不直接退出（`UIF-R7`） |

> `UIF-R16` 说 `Menu` 退出直接执行，`UIF-R7` 说主菜单 `ui_cancel` 打开退出确认。本规格区分「退出」按钮（直接 `flush()`）与 `ui_cancel`（先确认），见 Open Questions 第 8 条；根屏替换与子屏弹出按 `PAT-13`。

---
## 5. Layout Specification

### 5.1 ASCII Wireframe

```
┌──────────────────────────────────────────────────────────────────────┐
│ 食堂                                            钱包  +19 元         │  ① 店招 / 钱包
│ 本次入场费 85 元                                [本次会话不保存]      │  ② 价格 / 标记
├──────────────────────────────────────────────────────────────────────┤
│ 只读：存档为只读——另一窗口正在使用，本次可浏览，购买与解锁已禁用     │  ③ 只读说明（仅 ReadOnly）
│   [以只读继续]   [退出]                                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                      ● [ 开始营业 ]                                  │  ④ 主按钮
│                        [ 升级 ]                                      │
│                        [ 纪录 ]                                      │
│                        [ 设置 ]                                      │
│                        [ 退出 ]                                      │
│                                                                      │
├──────────────────────────────────────────────────────────────────────┤
│ 已保存                                              [非模态提示区]   │  ⑤ 保存指示 / 提示
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 Layout Zones

| Zone Name | Description | Approximate Size | Scrollable? | Overflow Behavior |
|---|---|---|---|---|
| A 店招 / 钱包 | 档口名、本次入场费、钱包（带符号）、`APPLIED_UNSAVED` 标记 | 全宽 × 20% | 否 | 150% 时价格与钱包分两行，不截断 |
| B 只读说明 | `ReadOnly` 时的说明与两个出口；其它载入结果不占此区 | 全宽 × 12% | 否 | 文案折行，按钮不折；不推移 A |
| C 入口栈 | 五个按钮纵向排列，固定顺序 | 居中 320×360 px（随 `s`） | 否 | 150% 时按钮增高，间距压缩，不换列 |
| D 提示层 | 左下「已保存」槽 + 中下持久提示槽 | 全宽 × 10% | 否 | 两槽互不挤占；持久提示最多两行 |

**安全边距**：`m = clamp(round(0.05 × H), 32, 96)`；缩放 `s = clamp(min(W/1280, H/720), 1.0, 2.0)`。所有外缘距视口 ≥ `m`。

### 5.3 Component Inventory

| Component Name | Type | Zone | Purpose | Required? | Reuses Existing Component? |
|---|---|---|---|---|---|
| `StationPlate` | 文本组 | A | 档口名（`display_name`）+ 「本次入场费 N 元」（`N = plan.menu_entry_fee`；「首局教学价」小字仅 `plan.has_entry_fee_override()` 时有） | 是 | 否（新，价目板样式） |
| `WalletReadout` | 文本 + 标记 | A | 有符号钱包；负值墨黑，正值回本红 + `+` | 是 | 是 — HUD 净利读数样式（只复用排版，不复用数据） |
| `UnsavedBadge` | 标签 | A | 「本次会话不保存」（`APPLIED_UNSAVED`） | 条件 | 是 — PAT-08 标记样式 |
| `ReadOnlyBanner` | 说明 + 两按钮 | B | 只读说明、「以只读继续」「退出」 | 条件 | 否（归 `ReadOnlyNotice` 规格） |
| `MenuButton` | 按钮 ×5 | C | 开始营业 / 升级 / 纪录 / 设置 / 退出 | 是 | 是 — PAT-01、PAT-03、PAT-14（单列清单：名称→状态→操作） |
| `SaveIndicator` | 非模态标签 | D | 「已保存」1.5 s | 条件 | 是 — `hud.md` §8 保存指示 |
| `NoticeSlot` | 非模态提示 | D | 恢复说明 / 进入失败 / 放弃 / 完成横幅 | 条件 | 是 — PAT-08 |

**Focus order（上→下、左→右）**：

1. `ReadOnlyBanner.以只读继续`（仅 `ReadOnly`）
2. `ReadOnlyBanner.退出`（仅 `ReadOnly`）
3. `开始营业`
4. `升级`
5. `纪录`
6. `设置`
7. `退出`

**Primary focus element on open**：`开始营业`。`ReadOnly` 时为 `以只读继续`（`UIF-UI` 的 `ReadOnlyNotice` 行）；两者冲突见 Open Questions 第 2 条。模态关闭后回到打开它的按钮；若该按钮已不可用，退回本表第一个可用控件（`UIF-R6`）。

### 5.4 Information Hierarchy

1. **本次入场费**——付费前必须知道价格（`UIF-R8`、`SETT-UI`）。
2. **钱包（带符号）**——决定“买不买得起/亏不亏得起”（`META-UI`）。
3. **开始营业**——主操作，焦点默认位。
4. **升级 / 纪录**——次级决策；升级在钱不够时仍可进入查看。
5. **设置 / 退出**——低频，固定底部。
6. **提示层**——保存与载入事实，不参与决策但必须可读。

---

## 6. States & Variants

| State Name | Trigger | What Changes Visually | What Changes Behaviorally | Notes |
|---|---|---|---|---|
| `Default` | `Fresh` / `Loaded` / `Migrated` | 店招、85 元、钱包、五按钮 | 全部可用 | 不弹载入提示（`UIF-R20`） |
| `FirstRun` | `plan.is_first_run = true` 且 `plan.has_entry_fee_override() = true` | 入场费显示 40，附「首局教学价」小字 | 开始营业按 `plan.run_config_fields()` 组装教学局 | 与 `RunConfig.entry_fee_override` 同源（`ONB-S9`、`UIF-AC47`） |
| `FirstRunDegraded` | `plan.is_first_run = true` 但 `plan.has_entry_fee_override() = false` | 入场费显示档口费（MVP 85），**不出现**「首局教学价」小字；其余与 `FirstRun` 相同 | 开始营业按档口费开局（`entry_fee_override = null`），菜单与本局同时 85 | ADR-0020 降级表第 3 行；不得边 40 边 85（`UIF-AC47`） |
| `ReadOnly` | `LoadReport = ReadOnly` | 顶部只读说明 + 两出口；钱包/等级只读 | 购买/解锁禁用；开始营业是否可用见 OQ1 | 不提供「新建生涯」（`SAVE-UI 3`） |
| `Blocked` | `LoadReport = Blocked` | 强制模态覆盖；主菜单控件不可操作 | `ui_cancel` 无效 | 归 `BlockedModal` 规格（`PAT-07`） |
| `Recovered` | `LoadReport = Recovered` | 中下持久提示：来源备份与可能丢失范围 | 可关闭，不重复 | 一次性（`UIF-R20`） |
| `Notice` | `init_failed` / `fault.phase = RUN` / `abandoned` | 中下提示一句 | 焦点回开始营业；不打开收据 | 文案按 phase 分支（`UIF-R19`） |
| `DataNotReady` | 数据未 `Ready` / 档口解析失败 | 开始营业禁用 + 原因；费用显示 `—` | 不发起切换 | `UIF-AC8`；不显示占位价格（`META-R20`） |
| `NoPlayableStation` | 无可用档口 | 开始营业禁用 + 「没有可用档口——请检查数据」 | 其余入口可用 | `UNL-UI` |
| `StartRejected` | `request_transition` 返回 `false` | 按钮旁一行原因 | 焦点留在开始营业；不重试 | `UIF-AC7`、`PAT-05` |
| `QuitConfirm` | 点「退出」或 `ui_cancel` | 确认模态，默认「取消」 | 确认后 `flush()` → `Quit` | `PAT-06`、`UIF-R16` |

---

## 7. Interaction Map

### 7.1 Navigation Inputs

| Input | Platform | Action | Visual Response | Audio Cue | Notes |
|---|---|---|---|---|---|
| ↑ / ↓、W / S、左摇杆、十字键 | 键盘 / 手柄 | 在五个按钮与只读说明按钮之间移动焦点 | 2 px 交互蓝外框 + 2 px 间距 | 无（焦点移动不发声） | 按屏幕位置，不按节点树（`INP-R6`、`INP-UI`） |
| 鼠标移动 | 鼠标 | 悬停高亮 | 浅色底、无外框 | 无 | 不改变键盘焦点（`PAT-01`） |

### 7.2 Action Inputs

| Input | Platform | Context | Action | Response | Animation | Audio Cue | Notes |
|---|---|---|---|---|---|---|---|
| `ui_accept` / 左键 | 键盘 / 手柄 / 鼠标 | 开始营业 | 组装 `RunConfig` 并请求 `Menu → Run` | 成功进局；失败原地给原因 | 按钮按下态 ≤ 80 ms | `sfx.ui.click` → `sfx.run.start` | 一次点击一次 `ui_button_activated` |
| `ui_accept` | 键盘 / 手柄 | 升级 / 纪录 | 推入子屏 | 子屏淡入 ≤ 0.2 s | — | `sfx.ui.click` | 深度 2 |
| `ui_accept` | 键盘 / 手柄 | 设置 | 打开 `SettingsModal` | 焦点移入主音量 | 模态淡入 ≤ 180 ms | `sfx.ui.click` | 焦点归还「设置」 |
| `ui_accept` | 键盘 / 手柄 | 退出 | 打开退出确认 | 默认焦点「取消」 | 同上 | `sfx.ui.click` | `flush()` 失败见 `UIF-OQ10` |
| `ui_cancel` | 键盘 / 手柄 | 设置/确认模态 | 关闭模态并归还焦点 | 模态消失 | 0 ms（reduce_motion） | — | `back_target` ① |
| `ui_cancel` | 键盘 / 手柄 | 可关闭提示 | 关闭提示 | 提示消失 | 淡出 | — | `back_target` ③ |
| `ui_cancel` | 键盘 / 手柄 | 主菜单根屏 | 打开退出确认 | 默认「取消」 | 模态淡入 | — | `back_target` ⑦；绝不直接退出 |

### 7.3 State-Specific Behaviors

| State | Input Restriction | Reason |
|---|---|---|
| `Blocked` | 主菜单控件全部不可操作；`ui_cancel` 无效 | 强制模态（`PAT-07`、`UIF-R20`） |
| `ReadOnly` | 购买/解锁不可用；开始营业见 OQ1 | 写入禁用（`SAVE-UI 3`） |
| `DataNotReady` | 开始营业禁用并写原因 | 无数据不伪造（`UIF-AC8`） |

**`ui_cancel` 落点**：模态（设置/退出确认）→ 关闭；可关闭提示 → 关闭；否则 → 退出确认。**永不直接退出游戏**（`UIF-R7`）。

---
## 8. Data Requirements

| Data Element | Source System | Update Frequency | Who Owns It | Format | Null / Missing Handling |
|---|---|---|---|---|---|
| 档口名 | `数据配置.get_station(station_id).display_name`（经菜单视图模型） | 进入 `Menu` 一次 | 数据配置 | 字符串 | 缺失 → `DataNotReady`，不显示编造名 |
| 本次入场费 | `FirstRunPlanner` 产出的 `plan.menu_entry_fee`（唯一来源：`plan.has_entry_fee_override()` 时为 40，否则为档口费 MVP 85），与同一次 `RunConfig` 组装同源 | `FirstRunPlanner.refresh()` 之后：进入 `Menu`、结算后、删除生涯后 | 首局引导（`FirstRunPlanner`）；本屏只读（`UIF-R8`） | 整数元 + 「首局教学价」小字（小字**仅** `plan.has_entry_fee_override()` 时出现） | 缺失 → `—` + 开始禁用 |
| `is_first_run` | `FirstRunPlanner` 产出的 `plan.is_first_run`；谓词由 planner 拥有（`LoadReport.first_run_eligible ∧ records.snapshot().career.runs_settled == 0` 在 planner 内求值），本屏不重算 | 同上 | 首局引导（`FirstRunPlanner`）；本屏只读（`UIF-R8`） | bool | plan 不可用 → `false` + `DataNotReady` |
| 钱包 | `局外升级.get_wallet()` | `progression_changed` | 局外升级 | int64，带 `+` / `−` | 查询失败 → `—`，不显示旧值 |
| `APPLIED_UNSAVED` | 结算/购买返回值 → 保存状态 | 每次写盘结果 | 局外升级/存档 | bool + 标记 | 未标记即不显示 |
| 保存状态 | `存档.save_status_changed` / `commit` 结果 | 每次写入 | 存档 | enum | 未知 → 不显示「已保存」 |
| `LoadReport` | `存档.load()`（进入 `Menu` 前已调用一次） | 进入 `Menu` 一次 | 存档 | 6 值枚举 + 字段 | 缺失 → `Blocked` 路径，不静默当 `Loaded` |
| 开始可用性 | 菜单视图模型：档口存在 ∧ 数据 `Ready` ∧ `RunConfig` 可组装（+ OQ1 的只读条件） | 进入/刷新 | 界面流程 | bool + 原因码 | 不可用必须给原因（`PAT-03`） |
| 完成横幅 | `records.career_completed()` 一次性标志 | 事件 | 纪录 | bool | 只显示一次，见 Open Questions 第 6 条 |

> **禁止**：本屏不得用「85 − 钱包」推算任何值；**不得自行判断 `is_first_run`**——只读 `plan.is_first_run`，谓词归 `FirstRunPlanner`；不得把 `entry_fee_override` 与档口价混用——`plan.has_entry_fee_override() = false` 时显示档口费（MVP 85）且不出现「首局教学价」小字；开始按钮旁的显示值就是 `plan.menu_entry_fee`，必须与同一次 `RunConfig` 组装写入 `entry_fee_override` 的值同源（`PAT-12`、`UIF-R8`、`UIF-AC47`、ADR-0020 D2.3）。

---

## 9. Events Fired

| Player Action | Event Fired | Payload | Receiver System | Notes |
|---|---|---|---|---|
| 激活任一控件 | `ui_button_activated(control_id)` | 控件 id | 音效反馈 | 每次已提交激活恰好一次（`UIF-R25`） |
| 开始营业 | `request_transition(Menu → Run, RunConfig)` | `station_id`、`upgrade_snapshot` 副本、`seed = null`、`is_first_run`、override | 场景与流程管理 | 只调用一次；失败不重试（`UIF-R8`） |
| 打开升级/纪录 | 屏幕栈推入 | `screen_id` | 界面流程自身 | 深度 ≤ 2 |
| 打开设置 | `push_context(Modal)` + 模态栈 | `SettingsModal` | 输入 / 界面流程 | 成对弹出（`UIF-R27`） |
| 退出（按钮或确认） | `flush()` → `Quit` | 无 | 存档 / 场景与流程管理 | flush 失败二次确认（`UIF-OQ10`） |
| 关闭提示 | 无对外事件；提示为界面内部状态 | — | 界面流程自身 | 同一事件不重复弹（`PAT-08`）；不发 `ui_button_activated` |

---

## 10. Transitions & Animations

| Transition | Trigger | Direction / Type | Duration (ms) | Easing | Interruptible? | Skipped by Reduced Motion? |
|---|---|---|---|---|---|---|
| 屏幕进入 | `Boot → Menu` / 收据返回 / 子屏弹出 | 场景淡入 | ≤ 200（由场景流程定上限） | 线性 | 否 | 是 → 0 ms |
| 子屏推入 | 升级/纪录 | 淡入 | ≤ 200 | 线性 | 否 | 是 |
| 模态打开 | 设置/退出确认 | 淡入 + 焦点移入 | ≤ 180 | ease-out | 否 | 是 |
| 模态关闭 | 取消/确认 | 淡出 + 焦点归还 | ≤ 180 | ease-in | 否 | 是 |
| 「已保存」 | 写入成功 | 淡入 → 停留 1.5 s → 淡出 | 1500 + 2×150 | 线性 | 否 | 是 → 瞬时出现/消失 |

> 过渡不得推迟状态生效，也不得推迟入场费/钱包出现（`UIF-R26`、`PAT-04`）。

---

## 11. Input Method Completeness Checklist

**Keyboard**
- [ ] 五个入口与只读说明的两个出口都可用方向键到达；顺序为上→下
- [ ] 只用键盘可完成开始营业、进出三个子屏/模态、取消退出确认
- [ ] 焦点始终可见；模态内焦点循环，不逃到下层
- [ ] `ui_cancel` 从主菜单打开退出确认，不直接退出
- [ ] 按住方向键重复导航（`D = 0.35 s`、`I = 0.12 s`），不加速

**Gamepad**
- [ ] 十字键/左摇杆可达全部控件；A 确认、B 取消、Start 不用于本屏
- [ ] 无模拟量精度要求；摇杆回滞 0.60 / 0.40
- [ ] 手柄断开时焦点停在原控件，无错误提示
- [ ] 不绘制键位图标，只写动作语义（`ACC-MOT-04`）

**Mouse**
- [ ] 五个按钮与只读说明按钮有悬停态，且与焦点态形状可区分
- [ ] 所有可点目标 ≥ 44×44 px，按钮 ≥ 56 px 高
- [ ] 滚轮在本屏无滚动区时为无动作，不误触
- [ ] 右键不绑定任何动作（右键是局内跳波，不在本屏）

**Touch**：不适用——无触屏目标，不提供虚拟摇杆（`INP-OQ5`）。

---

## 12. Accessibility

**Text contrast requirements**（实测值，来自 `ACC` 对比度表）：

| Text Element | Background Context | Required Ratio | Current Ratio | Pass? |
|---|---|---|---|---|
| 档口名 / 入场费 / 按钮文字 | 米白 `#F5F1E6` | ≥ 4.5:1 | 墨黑 14.6:1 | 是 |
| 钱包正数（回本红） | 米白 | ≥ 4.5:1 | 4.6:1 | 是 |
| 钱包负数 | 米白 | ≥ 4.5:1 | 墨黑 14.6:1 | 是 |
| 次级说明（「首局教学价」） | 米白 | ≥ 4.5:1 | 深灰 `#4A4E4A` 7.5:1 | 是 |
| 禁用按钮原因句 | 米白 | ≥ 4.5:1 | 墨黑 100% 不透明 | 是 |

**Colorblind-safe elements**：

| Element | Colorblind Risk | Mitigation |
|---|---|---|
| 钱包正负 | 红/墨黑 | `+` / `−`（U+2212）+ 右对齐数字（`ACC-VIS-06`） |
| 按钮可用/禁用 | 冷灰 | 原因句 + 不可获焦 + 无焦点环；不只靠灰 |
| 「未保存」标记 | 颜色 | 文字「本次会话不保存」 |
| 焦点 vs 悬停 | 蓝 vs 浅底 | 形状不同，可同屏并存（`ACC-VIS-07`） |

**Text scaling**：100% / 125% / 150%，只放大文字与行高。150% 时：A 区价格与钱包分两行；C 区按钮增高、文案不折行；B 区说明可折行、按钮不折；D 区提示最多两行。三档均不得出现横向滚动、截断或重叠（`ACC-VIS-09/10`）。

**Focus visibility**：2 px 实线外框 + 2 px 间距，永不被裁切；蓝环落在深底上时外加 2 px 米白隔边（`ACC-VIS-05`）。焦点环与悬停底同时命中同一按钮时叠加绘制（`PAT-01`）。

**Cognitive**：同时追踪 2 个数（入场费、钱包）；无计时器、无自动关闭；确认框 60 s 不操作仍保持（`ACC-COG-02/03`）。

**Grayscale**：去色后仍能读出钱包正负、入场费、哪个按钮禁用及原因、是否有只读/恢复提示（`ACC-VIS-08`）。

---

## 13. Localization

- 所有玩家可见字符串走 `tr()`；数字来自读数，不写死在句子里（`ACC-COG-08`、`UIF-R12`）。
- 中文为基准；允许 40% 扩写。按钮文案上限 8 字，档口名上限 12 字（超长两行截断，完整名不进入本屏详情）。
- 金额格式：整数元、无小数、无千分位；正数 `+`、负数 `−`（U+2212）。不在本屏显示百分比。
- 「本次入场费」与钱包标签不得合并成一个句子，避免语序变化导致数字与标签错位。
- 日期不在本屏出现。

---

## 14. Acceptance Criteria

> 参考分辨率 1280×720；另注明处除外。每条可独立判定。

**布局与信息**

- [ ] `MENU-AC01` 首局（`runs_settled = 0` 且 `first_run_eligible`）进入本屏，开始按钮旁显示「本次入场费 40 元」；同一会话首局结算后回到本屏，显示 85。
- [ ] `MENU-AC02` 显示值与同一次 `RunConfig` 组装使用的入场费一致；界面上不存在任何用钱包与价格相减得到的第三个数字。
- [ ] `MENU-AC03` 钱包为 `19` 显示 `+19 元`，`−12` 显示 `−12 元` 且减号为 U+2212，`0` 显示 `0 元`；不只靠颜色表达正负。
- [ ] `MENU-AC04` 五个按钮顺序固定为 开始营业 / 升级 / 纪录 / 设置 / 退出；开始营业在纵向最上。
- [ ] `MENU-AC05` 主菜单不显示档口选择器；数据中存在 `station_banquet` 时本屏不出现其可购买入口（`UIF-AC46`）。
- [ ] `MENU-AC06` 档口解析失败或数据未 `Ready`：开始营业禁用、旁有原因句、入场费显示 `—`，且不出现占位价格（`UIF-AC8`）。

**焦点与返回**

- [ ] `MENU-AC07` 打开本屏（无只读说明）时焦点在开始营业；有只读说明时按 Open Questions 第 2 条的裁定执行，两种实现都必须在测试夹具中写死。
- [ ] `MENU-AC08` 在本屏按 `ui_cancel`：打开退出确认，默认焦点「取消」；取消后焦点回「退出」按钮，游戏不退出。
- [ ] `MENU-AC09` 退出确认打开 60 秒不操作，仍保持打开，不自动取消/确认。
- [ ] `MENU-AC10` 从设置关闭后焦点回「设置」；从升级/纪录返回后焦点回进入前的按钮。
- [ ] `MENU-AC11` 鼠标悬停在「纪录」、键盘焦点在「升级」时，两套样式同屏可见；移动鼠标不改变键盘焦点。

**存档与提示**

- [ ] `MENU-AC12` `ReadOnly`：顶部出现只读说明与「以只读继续」「退出」；购买/解锁不可用；不出现「新建生涯」（`SAVE-UI 3`）。
- [ ] `MENU-AC13` `Recovered`：出现一次非模态说明，写明来源备份与可能丢失范围；关闭后再次进入本屏不再出现。
- [ ] `MENU-AC14` `init_failed` 显示「本局未能开始」+ 原因；`fault.phase = RUN` 显示「本局结算失败，收益不计」；两者都不打开收据、不入账。
- [ ] `MENU-AC15` `abandoned = true` 显示「本局已放弃，收益不计」；不显示「本局未能开始」。
- [ ] `MENU-AC16` `Blocked`：强制模态覆盖，默认「退出」，`ui_cancel` 无效，主菜单控件不可操作，无「覆盖保存」（`UIF-AC26`）。
- [ ] `MENU-AC17` 一次成功写入后显示「已保存」约 1.5 秒后淡出；不发声、不移动布局、不抢焦点；与持久提示同时存在时各占固定槽，互不覆盖、不排队。
- [ ] `MENU-AC18` `APPLIED_UNSAVED` 时钱包旁出现「本次会话不保存」；正常保存时不出现。
- [ ] `MENU-AC20` 开始营业的切换请求被拒绝：留在本屏、显示一行原因、焦点留在开始营业、不发起第二次请求。

**输入与无障碍**

- [ ] `MENU-AC21` 只用键盘、只用手柄各走一遍：进入本屏 → 打开设置并关闭 → 打开升级并返回 → 开始营业，全程无需鼠标。
- [ ] `MENU-AC22` 所有文字/背景组合实测对比度 ≥ 4.5:1；正负、可用/禁用、只读状态都有颜色之外的通道。
- [ ] `MENU-AC23` 100/125/150% 三档下无文字溢出、无重叠、无关键信息消失；150% 时按钮文字不折行。
- [ ] `MENU-AC24` 转灰度后仍能读出：钱包正负、入场费、哪个按钮不可用及原因、是否存在只读/恢复提示。
- [ ] `MENU-AC25` 焦点环在任一按钮上完整可见；按钮命中区 ≥ 56 px 高、≥ 44 px 宽；所有字符串来自 `tr()`。

---

## 15. Open Questions

| # | 问题 | 归属 | 何时解决 | 状态 |
|---|---|---|---|---|
| 1 | **`ReadOnly` 下能否开始新一局？** `UIF-R20` 只规定隐藏购买/解锁，未说「开始营业」；`SAVE-R10` 定义 `writable = false`，但一局运行本身不写盘。若不能，本屏需要一个禁用原因文案。 | game-designer + ux-designer | 写 epic 前 | 未决 |
| 2 | **`ReadOnlyNotice` 是否夺焦？** `UIF-UI` 给它初始焦点「以只读继续」，`PAT-08` 规定非模态提示不抢焦点、出现不改变焦点，`UIF-AC1` 又要求 `Boot → Menu` 后焦点为「开始营业」。三处冲突，本规格暂按「只读说明先显示并夺焦，关闭后回开始营业」撰写。 | ux-designer + game-designer | 写 epic 前 | 未决 |
| 3 | **首局是否要在菜单上解释 40 与 85 的差异？** `UIF-R8` 要求显示本次实际费用且不得只显示 85；`ONB-S9` 只要求显示 40。本规格显示「40 元」+「首局教学价」小字，不在菜单写 85；是否足够？ | writer + game-designer | 写 epic 前 | 未决 |
| 4 | **`UIF-R20` 的「隐藏购买/解锁入口」是隐藏「升级」按钮，还是保留入口、在页内禁用购买？** 本规格保留入口（升级页仍可查看等级），由 `UIF-R22` 的 `NOT_WRITABLE` 禁用购买按钮。 | ux-designer + systems-designer | 写 epic 前 | 未决 |
| 5 | **`APPLIED_UNSAVED` 是否在主菜单钱包旁显示？** `META-UI` 只说「钱包旁」，未限定升级页；`UIF-R21` 把它与保存指示并列。本规格按显示处理。 | ux-designer | 写 epic 前 | 未决 |
| 6 | **完成横幅是否在本屏显示？** `UIF-R23` 说离开收据后等下一次 `MainMenu` 显示；`REC-UI 4` 又要求纪录页显示一次性横幅。若两处都显示会重复。本规格暂定「本屏显示且只显示一次」，纪录页不重复。 | ux-designer + game-designer | 写 epic 前 | 未决 |
| 7 | **空钱包文案**：`interaction-patterns` 的 MainMenu Gaps 把「空钱包文案」列为待定。钱包 0 是真实值，本规格显示 `0 元`；是否需要一句「还没有收入」的说明？ | writer | 写 epic 前 | 未决 |
| 8 | **「退出」按钮与 `ui_cancel` 的差异**：`UIF-R16` 说 `Menu` 退出直接执行，`UIF-R7` 说主菜单 `ui_cancel` 打开退出确认。本规格让「退出」按钮直接 `flush()` 后退出、`ui_cancel` 走确认；需确认这不是两条不一致的退出路径。 | game-designer + ux-designer | 写 epic 前 | 未决 |
| 9 | **`PAT-05` 未列入 `interaction-patterns` 的 MainMenu 复用清单**（该表只列 PAT-03/08/13/14），但 `UIF-AC7` 要求开始失败时显示可见原因。本规格按 PAT-05 就地反馈；是否回改 Gaps 表？ | ux-designer | 写 epic 前 | 未决 |
| 10 | **退出时 `flush()` 失败的文案与策略**：`UIF-OQ10` 未决（「未保存，仍要退出？」后是否允许强制退出）。本屏的退出路径依赖该答案。 | ux-designer + qa-lead | Phase 4 | 未决 |
| 11 | **`ART-§7` UI/HUD Visual Direction 尚未撰写**：字体族、字重层级与按钮圆角未定；本规格的字号/线宽来自 `ART-§3` 与 `hud.md`。 | art-director | 垂直切片前 | 未决 |
