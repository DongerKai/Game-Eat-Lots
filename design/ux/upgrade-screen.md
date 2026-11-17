# UX Spec: 升级页（UpgradeScreen）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Journey Phase(s)**: 局外 / 每局之间（`Menu` 子屏）
> **Screen / Flow Name**: `UpgradeScreen`
> **Platform Target**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主、手柄部分支持、无触屏；参考分辨率 1280×720，支持至 3840×2160 与 21:9
> **Related GDDs**: `design/gdd/ui-flow.md`（规则 4、6、21–22、25–28）、`design/gdd/meta-upgrades.md`、`design/gdd/save-system.md`、`design/gdd/first-run-onboarding.md`、`design/gdd/input.md`
> **Sources**: 上述 GDD 的 `## UI Requirements`；`design/ux/interaction-patterns.md`；`design/ux/hud.md`（结构与质量基线）；`design/accessibility-requirements.md`；`.claude/docs/technical-preferences.md`
> **Sibling Specs**: `design/ux/hud.md`、`interaction-patterns.md`、`main-menu.md`、`receipt-screen.md`、`records-screen.md`、`settings-modal.md`
> **Accessibility Tier**: Standard（`design/accessibility-requirements.md`）
> **Style Reference**: `design/art/art-bible.md` § 4 Color System；**§ 7 UI/HUD Visual Direction 尚未撰写**（见 Open Questions）
> **Template**: UX Spec

> **范围边界**：本文件只覆盖升级目录、钱包、购买确认与购买失败态。它不拥有价格、等级、效果、可购买性（全部来自 `局外升级`），也不覆盖主菜单、收据或设置。判断标准：玩家此刻是否在决定“把余额花在哪一项升级上”。

## 引用代号

| 代号 | 出处 |
|---|---|
| `UIF-Rn` / `UIF-ACn` | `design/gdd/ui-flow.md` Core Rules / Acceptance Criteria |
| `UIF-UI` | 同文件 `## UI Requirements` 的屏幕契约表 |
| `META-Rn` / `META-ACn` / `META-UI` | `design/gdd/meta-upgrades.md` |
| `SAVE-Rn` / `SAVE-UI` | `design/gdd/save-system.md` |
| `ONB-Rn` | `design/gdd/first-run-onboarding.md` |
| `INP-Rn` | `design/gdd/input.md` |
| `PAT-NN` | `design/ux/interaction-patterns.md` |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. Purpose & Player Need

升级页把“下一局怎么打”变成一个可负担的选择：钱包里的钱只够买一两项，玩家要看清**现在是什么效果、买了变成什么效果、要花多少、买完还剩多少**。它不是数值搭配表，也不是攻略；它是一张会把余额扣掉的价目单。因此每一项都必须同时给出当前值、下一值与价格，禁止只写“升级”或只写价格（`META-UI`、`UIF-R22`）。

**The player goal**：在余额范围内选一项升级，确认价格与买后余额，然后买下。

**The game goal**：把 `get_catalog()` 的只读视图如实排出，购买走一次确认与一次完整校验，写盘失败时如实说“没生效”。

---

## 2. Player Context on Arrival

| Question | Answer |
|---|---|
| What was the player just doing? | 在主菜单决定先升级；或刚看完收据，带着“下一局怎么更容易回本”的问题进来 |
| What is their emotional state? | 计算、犹豫；亏损后可能想把钱花在“最有用”的一项 |
| What cognitive load are they carrying? | 中：钱包余额 + 三项价格 + 三项效果，最多同时比较 4 个数 |
| What information do they already have? | 钱包余额（主菜单/收据已出现）；上一局输在哪（容量/腻度/预览） |
| What are they most likely trying to do? | 买得起的那一项里选一个，确认买完还剩多少 |
| What are they likely afraid of? | “买贵了”“买完没生效”“钱扣了但存档没写进去” |

**Emotional design target**：像在柜台前改菜单——价格、效果、余额三列并排，买与不买都清楚，不催。

---

## 3. Navigation Position

```
Menu
  └── MainMenu（根屏）
        └── UpgradeScreen（本屏，子屏，ui_stack_depth = 2）
              └── ConfirmModal（购买确认，模态，depth 1）
```

**Modal behavior**：本屏是子屏，不是模态；购买确认是模态，打开时 `push_context(Modal)`，焦点移入并循环，关闭后回到发起购买的按钮（`PAT-02`、`UIF-R6`）。不允许从本屏再推子屏（`UIF-R4`）。

**Reachability — all entry points**：

| Entry Point | Triggered By | Notes |
|---|---|---|
| 主菜单「升级」 | `ui_accept` / 点击 | 推入子屏；焦点为第一条可购买项 |
| 首次引导推荐 | 首局引导高亮一项（若提供） | 只加「推荐」标记，不自动购买（`META-UI`） |
| 购买成功后 | 同一屏刷新 | 不离开本屏 |

---

## 4. Entry & Exit Points

**Entry table**：

| Trigger | Source Screen / State | Transition Type | Data Passed In | Notes |
|---|---|---|---|---|
| 打开升级 | `MainMenu` | 推入子屏（深度 2） | 只读目录、钱包、可购买性 | 每次都重新读 `get_catalog()` |
| 返回后再进 | `MainMenu` | 同上 | 最新目录/钱包 | 不缓存上次视图 |

**Exit table**：

| Exit Action | Destination | Transition Type | Data Returned / Saved | Notes |
|---|---|---|---|---|
| 返回 | `MainMenu` | 弹出子屏 | — | 焦点回主菜单「升级」 |
| `ui_cancel` | `MainMenu` | 弹出子屏 | — | `back_target` ⑤（`UIF-R7`） |
| 购买成功 | 本屏 | 关闭确认模态 + 刷新 | 钱包/等级已提交 | 焦点回该行按钮或首个可用项 |
| 购买失败 | 本屏 | 关闭/保留确认模态 | 无数据变化 | 就地显示原因（`PAT-05`） |

---

## 5. Layout Specification

### 5.1 ASCII Wireframe

```
┌──────────────────────────────────────────────────────────────────────┐
│ 升级                                              钱包  +19 元       │
│                                                   [本次会话不保存]   │
├──────────────────────────────────────────────────────────────────────┤
│ 加宽胃袋        等级 0 / 5      容量 20 → 21 格      12 元  [购买] ● │
│ 久吃不腻        等级 0 / 1      抗性 0 → 1 点/次     20 元  [购买]   │
│ 先看一步      等级 0 / 3    预览 0→1 道（上限 3）  16 元  [购买]     │
│                                                                      │
│ （目录超过一屏时在此滚动；焦点移动带动滚动）                         │
├──────────────────────────────────────────────────────────────────────┤
│ [返回]                                                               │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 Layout Zones

| Zone Name | Description | Approximate Size | Scrollable? | Overflow Behavior |
|---|---|---|---|---|
| A 抬头 | 「升级」+ 钱包 + 未保存标记 | 全宽 × 18% | 否 | 150% 时钱包换行到下一行，不截断 |
| B 目录 | 三条目录行；每行名称/等级/当前→下一效果/价格/按钮/原因 | 全宽 × 68% | 是（纵向） | 行高增加；原因句折行；不横向滚动、不缩字号 |
| C 底栏 | 「返回」固定在左下 | 全宽 × 10% | 否 | 按钮文字不折 |

**滚动策略**：焦点驱动。焦点移到视口外的购买按钮时，目录按最小距离滚动，使**整行（含原因句）**进入视口；鼠标滚轮可滚动但不改变焦点；不显示横向滚动条。滚动条仅在内容溢出时出现（见 Open Questions 第 2 条）。

### 5.3 Component Inventory

| Component Name | Type | Zone | Purpose | Required? | Reuses Existing Component? |
|---|---|---|---|---|---|
| `UpgradeHeader` | 文本组 | A | 标题 + 钱包 | 是 | 是 — 主菜单钱包样式 |
| `UnsavedBadge` | 标签 | A | 「本次会话不保存」 | 条件 | 是 — PAT-08 |
| `UpgradeRow` | 目录行 | B | 名称、等级 `n / max`、当前效果 → 下一效果、价格、原因 | 是 | 是 — PAT-14 |
| `BuyButton` | 按钮 | B | 「购买」；可购买时唯一焦点位 | 是 | 是 — PAT-01、PAT-03 |
| `RecommendBadge` | 标签 | B | 「推荐」（若上游提供） | 条件 | 否 |
| `BackButton` | 按钮 | C | 返回主菜单 | 是 | 是 — PAT-01 |
| `PurchaseConfirm` | 模态 | 覆盖 | 价格 + 买后余额；取消/确认 | 条件 | 是 — `ConfirmModal` 规格（PAT-06） |

**Focus order（上→下）**：第一条可购买行的 `购买` → 第二条可购买行的 `购买` → … → `返回`。不可购买行的按钮**不进入焦点链**（原因句仍 100% 不透明常显）。**Primary focus on open**：第一条可购买项的购买按钮；若无可购买项，`返回`（`UIF-UI`、`UIF-AC29`）。

### 5.4 Information Hierarchy

1. **钱包余额**——决定“买不买得起”。
2. **下一效果与价格**——决定“值不值”。
3. **当前效果 / 等级**——决定“离上限还有多远”。
4. **不可购买原因**——解释为什么按钮不亮。
5. **返回**——固定出口。

---

## 6. States & Variants

| State Name | Trigger | What Changes Visually | What Changes Behaviorally | Notes |
|---|---|---|---|---|
| `Default` | 目录已就绪 | 三行 + 钱包 | 可购买行可激活 | 进入即刷新 |
| `NoPurchasable` | 三项都不可购买 | 所有按钮禁用/灰 + 原因 | 初始焦点为 `返回` | `UIF-AC29` |
| `EmptyCatalog` | 目录为空 | 「暂无可升级项」 | 焦点 `返回` | PAT-09，不显示空表 |
| `DataNotReady` | 数据未 `Ready` | 整屏错误说明 + `返回` | 不显示占位价格 | `META-R20`、`UIF-R22` |
| `ReadOnly` | `LoadReport.writable = false` | 钱包旁只读/未保存标记；全部购买禁用 | `purchase` 不会被调用 | `META-R19`、`SAVE-UI 3` |
| `Confirm` | 点可购买行的「购买」 | 确认模态：价格 + 买后余额，默认「取消」 | 只打开一次 | `PAT-06` |
| `Busy` | 确认后等待 `purchase` 返回 | 确认/取消都禁用；写「正在处理上一笔」 | 忽略重复确认；`ui_cancel` 无效 | `META-R7`、`UIF-R22` |
| `CommitFailed` | `COMMIT_FAILED` | 就地写「未保存，购买未生效」；钱包/等级回滚 | 按钮恢复 | `META-UI` |
| `MaxLevel` | `MAX_LEVEL` | 该行写「已满级」，按钮消失/禁用 | 不可购买 | `META-R10` |
| `NoEffect` | `NO_EFFECT` | 写「本档口已看满（最多 Z 道）」 | 不可购买 | `META-R15`；Z 见 OQ3 |
| `InsufficientFunds` | `INSUFFICIENT_FUNDS` | 写「还差 N 元」；价格与余额同屏 | 不可购买 | `META-AC13` |
| `NotWritable` | `NOT_WRITABLE` | 写「本次会话无法保存，购买已禁用」 | 不可购买 | `META-R19` |
| `PrerequisiteLocked` | `PREREQUISITE_LOCKED` | 写缺哪项前置 | 不可购买 | 顺序校验见 `META-R7` |

---

## 7. Interaction Map

### 7.1 Navigation Inputs

| Input | Platform | Action | Visual Response | Audio Cue | Notes |
|---|---|---|---|---|---|
| ↑ / ↓、W / S、左摇杆、十字键 | 键盘 / 手柄 | 在可购买按钮与「返回」之间移动焦点 | 2 px 交互蓝外框 | 无（焦点移动不发声） | 按屏幕位置；跳过不可购买行 |
| 滚轮 | 鼠标 | 滚动目录 | 列表移动 | 无 | 不改变焦点 |
| 鼠标移动 | 鼠标 | 按钮悬停 | 浅色底 | 无 | 不改变键盘焦点（`PAT-01`） |

### 7.2 Action Inputs

| Input | Platform | Context | Action | Response | Animation | Audio Cue | Notes |
|---|---|---|---|---|---|---|---|
| `ui_accept` / 左键 | 键盘 / 手柄 / 鼠标 | 可购买按钮 | 打开购买确认 | 模态出现，默认「取消」 | 淡入 ≤ 180 ms | `sfx.ui.click` | 显示价格与买后余额 |
| ← / → | 键盘 / 手柄 | 确认模态 | 在「取消」「确认」间移动 | 焦点环 | 无（焦点移动不发声） | — | 默认「取消」 |
| `ui_accept` | 键盘 / 手柄 | 确认模态「确认」 | 调用一次 `purchase(id)` | 进入 `Busy` | 按钮禁用 | `sfx.ui.click` | `purchase` 仍完整校验 |
| `ui_accept` | 键盘 / 手柄 | 确认模态「取消」 | 关闭模态 | 焦点回原购买按钮 | 淡出 | — | `PAT-02` |
| `ui_cancel` | 键盘 / 手柄 | 确认模态（非 Busy） | 取消并归还焦点 | 同上 | — | — | `back_target` ① |
| `ui_cancel` | 键盘 / 手柄 | 本屏（无模态） | 弹回主菜单 | 子屏消失 | 淡入 | — | `back_target` ⑤ |

### 7.3 State-Specific Behaviors

| State | Input Restriction | Reason |
|---|---|---|
| `Busy` | 确认/取消禁用；重复确认忽略；`ui_cancel` 无效 | 只有一笔在飞（`META-R7`） |
| `ReadOnly` / `NotWritable` | 所有购买不可激活 | 不能落盘就不能买（`META-R19`） |
| `NoPurchasable` | 只有 `返回` 可聚焦 | `UIF-AC29` |
| `DataNotReady` | 只有 `返回` 可聚焦 | 不显示占位数据 |

**`ui_cancel` 落点**：确认模态 → 取消；否则 → 弹回 `MainMenu`。`Busy` 中显式消费为空操作并保留「正在处理上一笔」提示（见 OQ4）。

---

## 8. Data Requirements

| Data Element | Source System | Update Frequency | Who Owns It | Format | Null / Missing Handling |
|---|---|---|---|---|---|
| 目录行 | `局外升级.get_catalog()` | 进入/`progression_changed` | 局外升级 | 名称、等级、当前效果、下一效果、下一价格 | 缺字段 → 该行不渲染并记 ERROR；不补算 |
| 等级 | 目录行的 `level / max_level` | 同上 | 局外升级 | `n / max` | `max_level = 0` → 不显示该行 |
| 当前/下一效果 | 目录行已解析文案 | 同上 | 局外升级 | 文本 + 数值 | 不自行用 `cost_growth` 或公式重算 |
| 本档口预览上限 Z | `menu_vision_useful_max`（口径归 `menu-waves` / `META-R15`） | 同上 | 菜单波次 / 局外升级 | 整数级 | 上游未提供 → 省略「（上限 Z 道）」并在 OQ3 跟踪，不写死 3 |
| 价格 | 目录行 `next_price` | 同上 | 局外升级 | 整数元；`null` 表示不可购买 | `null` → 「已满级」，不显示 0 |
| 可购买性/原因 | `can_purchase(id)`（或目录行等价字段） | 同上 | 局外升级 | `{ok, reason}` | 未知原因 → 「暂时无法购买」，不猜 |
| 钱包 | `get_wallet()` | `progression_changed` | 局外升级 | int64，带符号 | 失败 → `—`，不显示旧值 |
| 未保存标记 | `APPLIED_UNSAVED` / 保存状态 | 写盘结果 | 局外升级 / 存档 | bool | 未标记不显示 |
| 推荐项 | 首局引导/上游推荐 id（待定） | 进入时一次 | 首局引导（待定） | 一行 id | 无来源 → 不显示「推荐」（OQ5） |

> **禁止**：用 `cost_growth` 自行算价格、用等级自行算效果、把 `next_price` 与钱包相减得到“还差多少”以外的任何值；显示必须直接取目录/返回值（`PAT-12`、`UIF-R22`）。

---

## 9. Events Fired

| Player Action | Event Fired | Payload | Receiver System | Notes |
|---|---|---|---|---|
| 激活控件 | `ui_button_activated(control_id)` | 控件 id | 音效反馈 | 每次已提交激活一次（`UIF-R25`） |
| 确认购买 | `purchase(id)` | 升级 id | 局外升级 | 确认后才调用；恰好一次；返回前不发第二次 |
| 打开确认 | `push_context(Modal)` | `ConfirmModal` | 输入 / 界面流程 | 成对弹出 |
| 关闭确认 | `pop_context()` | — | 输入 | 任何路径都不得泄漏 |
| 购买成功 | `progression_changed`（接收）后刷新 | 钱包/等级 | 局外升级 | 同帧刷新（`PAT-04`） |
| 购买写盘失败 | 无事件；显示「未保存，购买未生效」 | — | 存档 | 钱包/等级回滚（`META-UI`） |

---

## 10. Transitions & Animations

| Transition | Trigger | Direction / Type | Duration (ms) | Easing | Interruptible? | Skipped by Reduced Motion? |
|---|---|---|---|---|---|---|
| 子屏进入 | 主菜单「升级」 | 淡入 | ≤ 200 | 线性 | 否 | 是 → 0 ms |
| 子屏退出 | 返回 / `ui_cancel` | 淡出 | ≤ 200 | 线性 | 否 | 是 |
| 确认模态 | 购买 | 淡入 | ≤ 180 | ease-out | 否 | 是 |
| 购买成功 | `purchase` 返回 | 钱包/等级同帧更新；模态关闭 | 0（数字） | — | 否 | 是 |
| 拒绝/失败 | `COMMIT_FAILED` 等 | 行内原因出现，无抖动（非 HUD） | 0 | — | — | — |

> 数字必须同帧落定，动画只能叠加在新值上（`PAT-04`）。

---

## 11. Input Method Completeness Checklist

**Keyboard**
- [ ] 只用方向键可在所有可购买按钮与「返回」之间移动；不可购买行被跳过
- [ ] 只用键盘可完成购买确认、取消、失败后重试
- [ ] 焦点移到视口外时列表滚动，焦点环不被裁切
- [ ] `ui_cancel` 关闭确认或返回主菜单，不直接退出

**Gamepad**
- [ ] 十字键/左摇杆可达全部可购买项；A 确认、B 取消
- [ ] 无模拟量精度要求；摇杆回滞 0.60 / 0.40
- [ ] 手柄断开时焦点停在原按钮
- [ ] 不绘制键位图标，只写动作语义

**Mouse**
- [ ] 购买按钮有悬停态，与焦点态可区分
- [ ] 点击命中区 ≥ 56 px 高、≥ 44 px 宽
- [ ] 滚轮滚动目录，不改变焦点
- [ ] 右键无动作

**Touch**：不适用——无触屏目标，不提供虚拟摇杆（`INP-OQ5`）。

---

## 12. Accessibility

**Text contrast**：

| Text Element | Background Context | Required Ratio | Current Ratio | Pass? |
|---|---|---|---|---|
| 名称 / 等级 / 价格 / 效果 | 米白 `#F5F1E6` | ≥ 4.5:1 | 墨黑 14.6:1 | 是 |
| 钱包正数 | 米白 | ≥ 4.5:1 | 回本红 4.6:1 | 是 |
| 原因句 / 注释 | 米白 | ≥ 4.5:1 | 深灰 7.5:1 | 是 |
| 禁用按钮文字 | 米白 | ≥ 4.5:1 | 墨黑 100% 不透明 | 是 |

**Colorblind-safe**：

| Element | Colorblind Risk | Mitigation |
|---|---|---|
| 可购买/不可购买 | 颜色 + 灰 | 价格与余额同屏 + 原因句；不可购买不可获焦 |
| 钱包正负 | 红/墨黑 | `+` / `−`（U+2212） |
| 「推荐」 | 颜色 | 「推荐」文字标签 |
| 「未保存」 | 颜色 | 文字标记 |

**Text scaling**：100/125/150%。150% 时行高增加、效果文案折行、原因句折行；价格与按钮不重叠；允许目录滚动，不允许横向滚动或字号缩小（`ACC-VIS-10`）。

**Focus**：2 px 外框 + 2 px 间距；焦点驱动滚动时环完整可见；不可购买按钮不画焦点环（`ACC-VIS-15`）。

**Grayscale**：去色后仍能读出哪项可买、价格、余额正负、满级/看满/不可写原因（`ACC-VIS-08`）。

---

## 13. Localization

- 所有字符串走 `tr()`；数值来自目录，不拼进句子（`ACC-COG-08`）。
- 中文基准，允许 40% 扩写；名称上限 10 字，效果句上限 20 字，原因句上限 24 字（超长折行，不截断数值）。
- 金额整数元；正数 `+`、负数 `−`（U+2212）；等级写 `n / max`，不写百分比。
- 按钮「购买」「取消」「确认」「返回」固定短词；不得在语言切换后改变数字与单位顺序。
- `NO_EFFECT` 的「Z 道」必须来自上游数值，不写死 3。

---

## 14. Acceptance Criteria

**目录与数据**

- [ ] `UPG-AC01` 每条显示名称、等级 `n / max`、当前效果、下一效果、下一价格；值与 `get_catalog()` 逐字段一致，界面不重算。「先看一步」行额外显示「（本档口上限 Z 道）」（`META-R14`）。
- [ ] `UPG-AC02` 钱包常驻同屏，正数 `+19 元`、负数 `−12 元`、零 `0 元`；不只靠颜色表达。
- [ ] `UPG-AC03` 不可购买行显示原因：钱包不足「还差 N 元」、满级「已满级」、看满「本档口已看满（最多 Z 道）」、不可写、前置未解锁；原因句 100% 不透明。
- [ ] `UPG-AC04` 数据未 `Ready`：整屏错误 + `返回`，不显示占位价格或默认等级。
- [ ] `UPG-AC05` 目录为空显示「暂无可升级项」，不显示空表。
- [ ] `UPG-AC06` `progression_changed` 到达后，钱包与等级在同帧刷新，不等待下一帧或动画结束。

**购买流程**

- [ ] `UPG-AC07` 点击可购买行：打开确认，正文含价格与购买后余额，默认焦点「取消」；取消后焦点回原按钮。
- [ ] `UPG-AC08` 确认后只调用一次 `purchase(id)`；`purchase` 仍完整校验，确认框不替代校验。
- [ ] `UPG-AC09` `Busy` 期间确认/取消禁用并显示「正在处理上一笔」；重复确认不发起第二笔。
- [ ] `UPG-AC10` `COMMIT_FAILED`：显示「未保存，购买未生效」；钱包与等级回到购买前；按钮恢复可重试。
- [ ] `UPG-AC11` `MAX_LEVEL` 后该行写「已满级」，价格位置不显示 0；`NO_EFFECT` 写「本档口已看满（最多 Z 道）」。
- [ ] `UPG-AC12` `NOT_WRITABLE` 时所有购买禁用并显示原因，`purchase` 调用次数为 0。
- [ ] `UPG-AC13` 购买成功后钱包/等级同帧更新，确认关闭，焦点回该行按钮；若该行已不可购买，回第一个可用控件。

**焦点与返回**

- [ ] `UPG-AC14` 打开本屏时焦点在第一条可购买项的购买按钮；无可购买项时为「返回」。
- [ ] `UPG-AC15` 不可购买行的按钮不能获焦、不能激活；原因句仍可读。
- [ ] `UPG-AC16` 在本屏按 `ui_cancel`：无模态时弹回主菜单；确认模态打开时只关闭模态；从不直接退出。
- [ ] `UPG-AC17` 焦点移到目录视口外时，列表按最小距离滚动并保证整行（含原因）可见。
- [ ] `UPG-AC18` 鼠标悬停一项、键盘焦点在另一项时两套样式同屏；移动鼠标不改焦点。

**无障碍与输入**

- [ ] `UPG-AC19` 只用键盘与只用手柄都能完成：进入 → 选一项 → 确认 → 返回主菜单。
- [ ] `UPG-AC20` 所有文字/背景组合 ≥ 4.5:1；可购买性与钱包正负都有颜色之外的通道。
- [ ] `UPG-AC21` 100/125/150% 三档下无重叠、无横向滚动、无关键信息消失；价格与按钮不重叠。
- [ ] `UPG-AC22` 灰度截图仍能读出可购买性、价格、余额正负与原因。
- [ ] `UPG-AC23` 焦点环完整可见；按钮命中区 ≥ 56 px 高；所有字符串来自 `tr()`。

---

## 15. Open Questions

| # | 问题 | 归属 | 何时解决 | 状态 |
|---|---|---|---|---|
| 1 | **PAT-03 数据态禁用可获焦 vs `UIF-R22/AC29`**：PAT-03 说数据态禁用「可获焦，它是信息」，但 `UIF-AC29` 要求初始焦点为第一条可购买项、禁用控件不可获焦。本规格按 GDD 让不可购买按钮不可获焦，原因句常显；需回改 PAT-03 的适用表。 | ux-designer + game-designer | 写 epic 前 | 未决 |
| 2 | **目录滚动与焦点可见性**（`interaction-patterns` Gaps）：焦点驱动滚动、滚动条是否常显未定。本规格仅在溢出时显示滚动条，焦点移动按最小距离滚动。 | ux-designer + ui-programmer | 写 epic 前 | 未决 |
| 3 | **`NO_EFFECT` 与预览行的 Z 从哪里取？** `META-R14/R15` 要求「先看一步」行写「（本档口上限 Z 道）」、`NO_EFFECT` 写「本档口已看满（最多 Z 道）」，但 `get_catalog()` 的字段不含 `menu_vision_useful_max`，其口径归 `menu-waves` / `META` 公式。若返回值不带 Z，本规格省略括号、`NO_EFFECT` 只写「本档口已看满」。 | systems-designer + ux-designer | 写 epic 前 | 未决 |
| 4 | **`Busy` 中 `ui_cancel` 的语义**：`UIF-R22` 只说按钮禁用直到返回。本规格按「确认/取消都禁用、`ui_cancel` 无动作并保留提示」处理；是否允许取消关闭模态（购买仍在后台完成）？ | ux-designer + lead-programmer | 写 epic 前 | 未决 |
| 5 | **「推荐」高亮的数据来源**：`META-UI` 与 `ONB` 都允许高亮一项，但没有 `recommended_upgrade_id` 接口。本规格默认不显示，等待接口确定。 | game-designer + ux-designer | 设计首局引导时 | 未决 |
| 6 | **确认框是否显示“买后余额为负”**：钱包可为负（`META-R` 钱包规则），确认文案若写 `−5 元` 是否会让玩家误以为不能买？本规格照实显示带符号余额。 | writer + ux-designer | 写 epic 前 | 未决 |
| 7 | **`APPLIED_UNSAVED` 与 `COMMIT_FAILED` 的并存顺序**：购买失败会回滚，不应同时显示未保存；结算入账的未保存标记可能已存在。何时清除标记未定。 | ux-designer + lead-programmer | Phase 4 | 未决 |
| 8 | **`ART-§7` 尚未撰写**：字体、圆角与目录行分隔线样式未定；本规格沿用 `ART-§3` 与 `hud.md` 的线宽/圆角。 | art-director | 垂直切片前 | 未决 |
