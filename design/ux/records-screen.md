# UX Spec: 纪录页（RecordsScreen）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Journey Phase(s)**: 局外 / 每局之间（`Menu` 子屏）
> **Screen / Flow Name**: `RecordsScreen`
> **Platform Target**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主、手柄部分支持（`Gamepad Support: Partial`）、无触屏；参考分辨率 1280×720，支持至 3840×2160 与 21:9
> **Related GDDs**: `design/gdd/ui-flow.md`（规则 7、23、25–28、屏幕契约）、`design/gdd/records-and-milestones.md`（schema、`snapshot` / `milestone_progress` / `get_milestone_catalog`、UI Requirements 1–7）、`design/gdd/save-system.md`（只读、警告）、`design/gdd/input.md`、`design/gdd/data-config.md`（仅 `get_station().display_name`）
> **Sources**: 上述 GDD 的 `## UI Requirements`；`design/ux/interaction-patterns.md`；`design/ux/hud.md`（结构与质量基线，未复制内容）；`design/accessibility-requirements.md`；`.claude/docs/technical-preferences.md`
> **Sibling Specs**: `main-menu.md`（入口与焦点归还）、`receipt-screen.md`（「新纪录」印章与完成横幅的首发地）、`upgrade-screen.md`、`settings-modal.md`、`hud.md`；`ConfirmModal` 版式归其自身规格
> **Accessibility Tier**: Standard（承诺与逐条要求见 `design/accessibility-requirements.md`）
> **Style Reference**: `design/art/art-bible.md` § 4 Color System；**§ 7 UI/HUD Visual Direction 尚未撰写**（见 Open Questions）
> **Template**: UX Spec

> **范围边界**：本文件只覆盖「查看账本」这一屏：档口最佳纪录、里程碑进度、通关状态。收据上的「新纪录」印章、「新达成」条、完成庆祝横幅不属本屏；删除生涯在设置模态；升级价格与钱包不在本屏出现。判断标准：玩家此刻是否只想**读**已发生的成绩。

## 引用代号

| 代号 | 出处 |
|---|---|
| `UIF-Rn` / `UIF-ACn` | `design/gdd/ui-flow.md` Core Rules 第 n 条 / Acceptance Criteria 第 n 条 |
| `UIF-UI` | 同文件 `## UI Requirements` 的屏幕契约表（含每屏初始焦点） |
| `REC-Rn` / `REC-UI` / `REC-ACn` | `design/gdd/records-and-milestones.md` 规则 / UI Requirements / Acceptance Criteria |
| `SAVE-Rn` / `SAVE-UI` | `design/gdd/save-system.md` |
| `INP-Rn` / `INP-UI` | `design/gdd/input.md` |
| `PAT-NN` | `design/ux/interaction-patterns.md` 模式编号 |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. Purpose & Player Need

纪录页是玩家的**账本底页**：不是排行榜，也不是成就墙，而是「我每个档口最好打到多少、离通关还差什么」。它必须让玩家在决定「再来一局」之前，用一屏回答两个问题——**最好的一次是多少**、**还差哪几项**。因此数字必须与收据逐字段一致：同一份 `records`，同一个只读快照，界面不做第二次判断。任何「界面说破纪录、账本却是旧值」的分裂都会摧毁玩家对结算的信任（`REC` 接口所有权、`REC-AC3`）。

**The player goal**：读完自己的最佳纪录与里程碑进度，确认下一步该冲哪个档口或哪条里程碑。

**The game goal**：把已冻结的历史事实如实交出，让「最好的一局」与「通关进度」可被独立核对，且不泄露隐藏里程碑、不暗示最优解。

---

## 2. Player Context on Arrival

| Question | Answer |
|---|---|
| What was the player just doing? | 在主菜单比较入场费与余额后，点「纪录」；或刚亏完一局、想确认自己是否仍保有旧最佳 |
| What is their emotional state? | 平静、求证；连续亏损后可能带着「到底有没有进步」的怀疑 |
| What cognitive load are they carrying? | 低到中：记忆里只有上一局净利；需要把「本局」与「最佳」分开 |
| What information do they already have? | 上一局净利、是否回本、本局是否破纪录（收据已说过一次） |
| What are they most likely trying to do? | 查某档口的最佳净利与回本率；确认距离通关还差几项 |
| What are they likely afraid of? | 纪录被覆盖、数字与收据不一致、通关条件看不懂 |

**Emotional design target**：像翻一本自己记账的旧账本——数字稳定、笔迹清楚，翻到哪页都敢信。

---

## 3. Navigation Position

```
Menu（顶层状态根屏）
  └── MainMenu（ui_stack_depth = 1）
        └── RecordsScreen（本屏，子屏，ui_stack_depth = 2）
```

**Modal behavior**：非模态子屏；无模态子层，不打开确认框，不叠加第三个上下文（`UIF-R5`）。`ui_cancel` 等价「返回」。

**Reachability — all entry points**：

| Entry Point | Triggered By | Notes |
|---|---|---|
| 主菜单「纪录」 | 点击 / `ui_accept` | 一级入口，不埋在设置里（`REC-UI1`） |
| 收据 | 不提供直达 | 收据只有「再来一局」「返回菜单」（`UIF-R11`） |
| 升级 / 设置 | 不提供 | 子屏之间不横向跳转 |

---

## 4. Entry & Exit Points

**Entry table**：

| Trigger | Source Screen / State | Transition Type | Data Passed In | Notes |
|---|---|---|---|---|
| 激活「纪录」 | `MainMenu` | 子屏推入（淡入 ≤ 200 ms） | 无载荷；本屏自行读取只读快照 | 读盘完成前不显示占位数字（不伪造默认值） |
| 从设置/升级返回 | 不经过本屏 | — | — | 返回目标由 `back_target` 决定（`UIF-R7`） |

**Exit table**：

| Exit Action | Destination | Transition Type | Data Returned / Saved | Notes |
|---|---|---|---|---|
| 「返回」/ `ui_cancel` | `MainMenu` | 子屏弹出（淡出 ≤ 200 ms） | 无；本屏不写任何数据 | 焦点归还「纪录」按钮（`PAT-02`） |
| 顶层状态切换（退出游戏等） | 不经过本屏 | — | — | 本屏无未保存状态，无需确认 |

> 本屏是**纯读屏**：进入与退出都不调用 `prepare_run` / `prepare_run_result` / `commit`，也不改变任何纪录（`UIF-R23`）。

---

## 5. Layout Specification

### 5.1 ASCII Wireframe

```
┌──────────────────────────────────────────────────────────────────────┐
│ 纪录                                      已结算 12 局               │
│ 终身净利 +340 元 · 终身吃进 2,180 元       距离通关还差 2 项         │
├──────────────────────────────────────────────────────────────────────┤
│ 档口纪录                                                   （可滚动）│
│ 档口          最佳净利   回本率     总值      达成日期               │
│ ● 食堂        +19        122.3%     104 元    2026-11-04             │
│   面档        −5          92.3%      60 元    2026-11-05             │
│   夜宵摊      尚无纪录                                               │
│ 里程碑                                                               │
│ 未达成（3）                                                          │
│   首笔回本     单局净利 ≥ 0 元        +19 / 0 元                     │
│   吃满十局     已结算 10 局           7 / 10 局                      │
│ 已达成（2）                                                          │
│   食堂回本 100% 食堂回本率 ≥ 100%     122.3% / 100.0%  2026-11-04    │
│ 通关进度                                                             │
│ 还差：食堂回本 100% · 面档回本 100%                                  │
├──────────────────────────────────────────────────────────────────────┤
│ [ 返回 ]                                     ↑↓ 浏览 · Esc 返回      │
└──────────────────────────────────────────────────────────────────────┘
```

> 示意数据；实际文案来自 `get_milestone_catalog()` 与读数，不写死名字。

### 5.2 Layout Zones

| Zone Name | Description | Approximate Size | Scrollable? | Overflow Behavior |
|---|---|---|---|---|
| A 标题与生涯合计 | 屏名、`career` 三项、通关状态摘要、只读条 | 高 18%，全宽减安全边距 | 否 | 150% 时合计换行，状态摘要移到第二行右侧；不截断 |
| B 主滚动体 | 档口表、里程碑两组、通关进度 | 高约 68% | 是（纵向） | 单列纵向滚动；不横向滚动、不压缩字号（`PAT-14`） |
| C 底栏 | 「返回」按钮 + 动作提示 | 高约 14% | 否 | 按钮增高，提示换行；焦点不被底栏裁切 |

内容列在 1280 宽度下最大 1024 px、左对齐于安全边距 `m`；更宽分辨率不拉伸列宽比例，只增加两侧留白。

### 5.3 Component Inventory

| Component Name | Type | Zone | Purpose | Required? | Reuses Existing Component? |
|---|---|---|---|---|---|
| 屏名「纪录」 | Label | A | 定位 | Yes | Yes — 菜单标题样式 |
| 生涯合计行 | Label group | A | 局数、终身净利、终身吃进 | Yes（非空生涯） | No — 新组件 |
| 通关状态摘要 | Label chip | A | 「距离通关还差 N 项」/「已通关」 | Yes | No — 新组件 |
| 只读条 | Banner | A | 存档不可写时说明本次不保存 | 条件 | Yes — `ReadOnlyNotice` 的紧凑变体 |
| 分组标题 | Label | B | 「档口纪录」「里程碑」「通关进度」 | Yes | Yes — 分区标题样式 |
| 档口表头 | Label row | B | 五列列名 | Yes | No |
| 档口行 | List row（PAT-14） | B | 一个档口的冻结最佳局 | Yes | No — 新组件 |
| 空状态行 | List row（PAT-09） | B | 目录中尚无纪录的档口 | 条件 | Yes — 空状态行变体 |
| 里程碑组标题 | Label | B | 「未达成（n）」「已达成（n）」 | Yes | No |
| 里程碑行 | List row（PAT-14） | B | 名称、条件、进度、日期、必需标记 | Yes | Yes — 与档口行共用行基座 |
| 通关进度块 | Text block | B | 「还差 N 项」+ 必需项名单 / 通关后目标 | Yes | No |
| 「返回」按钮 | Button | C | 退出本屏 | Yes | Yes — 菜单按钮 |

**Primary focus element on open**：第一条**有纪录**的档口行；没有这样的行时聚焦「返回」（`UIF-UI` RecordsScreen 行）。空状态行与分组标题不获焦，不在焦点链中。

### 5.4 Information Hierarchy

1. **档口最佳净利与回本率**——玩家来这里的第一目的，同屏、带符号、右对齐等宽。
2. **距离通关还差几项**——决定下一步的动机；放在标题区右侧常驻，滚动时仍可见。
3. **里程碑逐条进度**——需要时向下读；不抢首屏。
4. **达成日期、条件文案**——核对用，最末。

---

## 6. States & Variants

| State Name | Trigger | What Changes Visually | What Changes Behaviorally | Notes |
|---|---|---|---|---|
| Populated | `snapshot().stations` 非空 | 五列表 + 两组里程碑 + 通关块 | 初始焦点=第一条纪录行 | 默认态 |
| EmptyCareer | `career.runs_settled = 0` 且 `stations` 为空 | 只显示「还没有账本」+ 一句说明（「结算第一局后，这里会记录每个档口的最好一局」）；隐藏合计、表、里程碑、通关块 | 初始焦点=「返回」；无任何 0 值表 | `REC-UI6`、`REC-AC8` |
| NoStationRecords | 有局数但所有档口都无纪录（如纪录被裁掉） | 每档口一行「尚无纪录」；里程碑正常显示，`current = null` 显示「未开始」 | 初始焦点=「返回」；空状态行不可聚焦 | 空状态不伪装成 0（`PAT-09`） |
| InProgress | `snapshot().completed_at = null` | 摘要「距离通关还差 N 项」；通关块列出未达成必需项 | N 为展示计数，不参与判定 | `UIF-R23` |
| Completed | `completed_at ≠ null` | 摘要「已通关 · YYYY-MM-DD」；通关块改为「通关后目标」列出未达成项 | 永不回退；不重播庆祝 | `REC-R14`、`REC-AC21` |
| NoCompletionPath | 目录无 `required_for_completion = true` | 摘要与通关块显示「尚未配置通关条件」 | 不显示「还差 0 项」，不显示「已通关」 | 数据警告 `NO_COMPLETION_PATH`；见 OQ9 |
| RemovedStation | `get_station(id)` 返回空但有纪录 | 行首显示 `station_id` + 「（档口已移除）」；冻结数值照常 | 同普通行，进入焦点链 | `REC-R7`；见 OQ2 |
| ArchivedMilestone | 已达成但目录已移除 | 不出现在任何组 | 组计数只数目录内条目 | 活动列表不含已移除项（`REC` Edge Cases）；见 OQ5 |
| ReadOnly | `LoadReport.writable = false` | 标题区出现只读条 | 数据仍可读；无写操作可禁用 | `SAVE`；见 OQ10 |
| DataNotReady | `snapshot()` 不可用（防御分支） | 「纪录暂不可用」+「返回」；不显示 0 或旧值 | 初始焦点=「返回」 | 同 `DATA_NOT_READY` 口径（`REC-R22`）；见 OQ8 |

---

## 7. Interaction Map

### 7.1 Navigation Inputs

| Input | Platform | Action | Visual Response | Audio Cue | Notes |
|---|---|---|---|---|---|
| ↑ / ↓ | 键盘、手柄 | 在可聚焦行之间上/下移动（档口行 → 里程碑行 → 返回） | 焦点环 + 目标行滚入视口 | 无（焦点移动不发声） | 跨过不可聚焦的空状态行与标题；重复 `D = 0.35 s`、`I = 0.12 s`（`INP`） |
| ← / → | 键盘、手柄 | 无动作 | 无 | 无 | 行内无横向控件；不切换列（见 OQ6） |
| 滚轮 | 鼠标 | 滚动 B 区 | 内容滚动，焦点不动 | 无 | 不改变键盘/手柄焦点 |
| 鼠标悬停 | 鼠标 | 高亮行底 | 浅底、无外框 | 无 | 与焦点环形状不同（`PAT-01`、`ACC-VIS-07`） |
| 鼠标点击行 | 鼠标 | 将焦点移到该行 | 焦点环出现 | 无（焦点移动不发声） | 不打开详情、不激活动作 |

### 7.2 Action Inputs

| Input | Platform | Context (What must be focused) | Action | Response | Animation | Audio Cue | Notes |
|---|---|---|---|---|---|---|---|
| `ui_accept` | 键盘、手柄 | 「返回」 | 退出到主菜单 | 子屏淡出 | ≤ 200 ms | 按钮激活一次 | 发 `ui_button_activated(records_back)` |
| `ui_accept` | 键盘、手柄 | 任一纪录/里程碑行 | 无动作 | 无 | 无 | 无 | 只读清单；不弹出「详情」伪造可操作（见 OQ6） |
| `ui_cancel` | 键盘、手柄 | 任意 | 退出到主菜单 | 子屏淡出 | ≤ 200 ms | 无 | **不**退出游戏、不打开确认（`UIF-R7` ⑤） |
| A / Enter | 手柄 / 键盘 | 同 `ui_accept` | 同 `ui_accept` | — | — | — | 不画键位图标（`ACC-MOT-04`） |

### 7.3 State-Specific Behaviors

| State | Input Restriction | Reason |
|---|---|---|
| EmptyCareer / DataNotReady | 仅「返回」可聚焦；`ui_accept` 在别处无目标 | 没有可浏览数据 |
| NoStationRecords | 里程碑行与「返回」可聚焦 | 列表仍需键盘可浏览（`REC-UI7`） |
| 滚动中 | 无输入锁；焦点移动即滚动 | 无计时、无动画依赖 |

---

## 8. Data Requirements

本屏只读。每个显示值都必须来自下列 owner 接口，禁止在界面里比较、重算或排序（`PAT-12`、`UIF-R23`）。

| Data Element | Source System | Update Frequency | Who Owns It | Format | Null / Missing Handling |
|---|---|---|---|---|---|
| 档口行集合 | `records.snapshot().stations`（键） | 进入本屏时读一次 | 纪录与里程碑 | map 键并集：先目录顺序，后已移除档口按 id 升序 | 空 → EmptyCareer / NoStationRecords |
| 档口展示名 | `data_config.get_station(id).display_name` | 同上 | 数据配置 | 文本；改名后跟随配置（`REC-AC30`） | 缺失 → id + 「（档口已移除）」 |
| 最佳净利 | `stations[id].best.net_profit` | 同上 | 纪录与里程碑（冻结值） | 整数元；正 `+` / 负 `−`（U+2212） / 0 为 `0` | 无记录 → 「尚无纪录」 |
| 回本率 | `best.return_percent_x10` | 同上 | 纪录与里程碑（冻结值） | 一位小数 + `%`（`1223` → `122.3%`） | `entry_fee = 0` → 「免单」；其余 null → 「—」 |
| 总值 | `best.total_value` | 同上 | 纪录与里程碑（冻结值） | 整数元 | 无记录 → 「尚无纪录」 |
| 达成日期 | `best.achieved_at` | 同上 | 纪录与里程碑 | `YYYY-MM-DD`（取 UTC ISO-8601 的日期段） | null → 「—」（`RECORD_TIMESTAMP_MISSING`） |
| 生涯合计 | `snapshot().career.{runs_settled, lifetime_net_profit, lifetime_value}` | 同上 | 纪录与里程碑 | 局数；有符号元；整数元 | 空生涯不显示 |
| 里程碑目录 | `get_milestone_catalog()`（`id`、名称、条件文案、阈值、`required_for_completion`） | 同上 | 纪录与里程碑 | 目录顺序；不显示目录未返回的条目 | 空目录 → 不显示里程碑组 |
| 里程碑进度 | `milestone_progress(id).{current, threshold, unit, earned}` | 同上 | 纪录与里程碑 | `当前 / 阈值 单位`；比率按 0.1% 折百分比 | `current = null` → 「未开始」，不显示 0 |
| 已达成阈值 | `snapshot().milestones[id].threshold_at_earn` | 同上 | 纪录与里程碑（冻结值） | 同 `unit` | 缺失 → 回退目录阈值并记入 OQ5 |
| 达成日期（里程碑） | `snapshot().milestones[id].earned_at` | 同上 | 纪录与里程碑 | `YYYY-MM-DD` | null → 「—」 |
| 通关状态 | `snapshot().completed_at != null` | 同上 | 纪录与里程碑 | 布尔 → 「已通关 · 日期」 | 只取锁存值，**不**自行判定（`UIF-R23`） |
| 距通关项数 | 目录必需项 − `snapshot().milestones` 键 | 同上 | 纪录取数 + 目录取数；只计数 | 整数 | 无必需项 → 不显示（OQ9） |
| 只读状态 | `LoadReport.writable` | 进入本屏时 | 存档 | 布尔 | 仅 `false` 时显示只读条 |

**禁止**：按 `net_profit` 或回本率重排档口、按时间戳排序里程碑（`REC-R15` 明确时间戳不参与排序）、把 `return_ratio` 与 `return_percent_x10` 互相换算后断言、把 `broke_even` 之类的派生谓词在界面重算。

---

## 9. Events Fired

| Player Action | Event Fired | Payload | Receiver System | Notes |
|---|---|---|---|---|
| 打开本屏 | `screen_changed(RecordsScreen, 2)` | `screen_id`、`ui_stack_depth` | 音效反馈 / QA | 由界面流程发；本屏不额外发 |
| 激活「返回」 | `ui_button_activated(records_back)` | `control_id` | 音效反馈 | 每次已提交激活恰好一次（`UIF-R25`） |
| 关闭本屏 | `screen_changed(MainMenu, 1)` | 同上 | 音效反馈 / QA | 焦点归还「纪录」 |
| 任何浏览 | 无事件 | — | — | 焦点移动不发 `ui_button_activated` |

> 本屏不订阅 `record_broken` / `milestone_earned` / `career_completed`：这些信号只在结算事务内发生，届时收据在场；在纪录页里补播会造成重载后重复庆祝（`REC-AC21`）。

---

## 10. Transitions & Animations

| Transition | Trigger | Direction / Type | Duration (ms) | Easing | Interruptible? | Skipped by Reduced Motion? |
|---|---|---|---|---|---|---|
| 子屏进入 | 主菜单「纪录」 | 淡入 | ≤ 200 | 线性 | 否 | 是 → 0 ms |
| 子屏退出 | 返回 / `ui_cancel` | 淡出 | ≤ 200 | 线性 | 否 | 是 → 0 ms |
| 焦点滚入视口 | ↑/↓ 移动 | 滚动到行完整可见 | 0（同帧） | — | — | 是（本就瞬时） |
| 内容刷新 | 无（本屏打开后不刷新） | — | 0 | — | — | — |

> 过渡不得推迟内容出现，也不得推迟焦点（`UIF-R26`、`PAT-04`）。无逐帧动画、无滚动惯性、无自动滚动。

---

## 11. Input Method Completeness Checklist

**Keyboard**
- [ ] 所有可聚焦行与「返回」都可用 ↑/↓ 到达，顺序为档口行 → 里程碑行 → 返回
- [ ] 只用键盘可读完整个账本（焦点驱动滚动）并返回主菜单
- [ ] 焦点始终可见；焦点移出视口时自动滚入，环不被裁切
- [ ] `ui_cancel` 返回主菜单，不退出游戏、不弹确认
- [ ] 空生涯与数据未就绪时，仅「返回」可聚焦

**Gamepad**
- [ ] 十字键/左摇杆上下可达全部可聚焦行；A 确认、B 取消
- [ ] 无模拟量精度要求；回滞 0.60 / 0.40（`INP`）
- [ ] 手柄断开时焦点停在原行，无错误提示
- [ ] 不绘制键位图标，只写动作语义

**Mouse**
- [ ] 行与按钮有悬停态，且与焦点态形状可区分
- [ ] 可点目标 ≥ 44×44 px，「返回」≥ 56 px 高
- [ ] 滚轮只滚动 B 区；焦点不变
- [ ] 右键无动作（右键是局内跳波，不在本屏）

**Touch**：不适用——无触屏目标，不提供虚拟摇杆（`INP-OQ5`）。

---

## 12. Accessibility

**Text contrast requirements**（实测值，来自 `ACC` 对比度表）：

| Text Element | Background Context | Required Ratio | Current Ratio | Pass? |
|---|---|---|---|---|
| 档口名 / 列名 / 组标题 / 按钮文字 | 米白 `#F5F1E6` | ≥ 4.5:1 | 墨黑 14.6:1 | 是 |
| 正净利（回本红 + `+`） | 米白底 | ≥ 4.5:1 | 4.6:1 | 是（仅米白底） |
| 负净利（墨黑 + `−`） | 米白底 | ≥ 4.5:1 | 14.6:1 | 是 |
| 条件文案 / 日期 / 「尚无纪录」 | 米白底 | ≥ 4.5:1 | 深灰 `#4A4E4A` 7.5:1 | 是 |
| 焦点环 | 米白 / 深底 | ≥ 3:1 | 交互蓝 4.9:1（米白）；深底外加 2 px 米白隔边 | 是 |

**Colorblind-safe elements**：

| Element | Colorblind Risk | Mitigation |
|---|---|---|
| 净利正负 | 红/墨黑 | `+` / `−` + 右对齐等宽数字（`ACC-VIS-06`） |
| 是否回本 | 仅百分比 | 百分比数字本身；不额外加红/绿标签 |
| 已达成 / 未达成 | 仅颜色或对勾 | 分组标题写「已达成（n）」；行内写「已达成」+ 日期（`PAT-14`） |
| 通关必需项 | 仅图标 | 行内文字「通关必需」 |
| 焦点 vs 悬停 | 蓝 vs 浅底 | 外框 vs 浅底，形状不同，可同屏（`ACC-VIS-07`） |
| 只读条 | 仅颜色 | 文字「本次会话不保存」 |

**Text scaling**：100% / 125% / 150%，只放大文字与行高。100/125% 档口行为单行五列；150% 时档口行改为三行网格——① 档口名；②「最佳 +19 · 回本率 122.3%」；③「总值 104 元 · 2026-11-04」，标签与值同现，列不重叠。里程碑行条件文案最多两行，进度与日期垂直居中。三档均不得出现横向滚动、截断或重叠（`ACC-VIS-09/10`）。

**Focus visibility**：2 px 实线外框 + 2 px 间距，永不被裁切；焦点驱动滚动时环完整可见（`ACC-VIS-05`）。行悬停浅底与焦点环可同时出现（`PAT-01`）。

**Cognitive**：同时追踪项 ≤ 3（最佳净利、回本率、距通关项数）；无计时器、无自动关闭、无滚动动画强制等待（`ACC-COG-02/03`）。

**Grayscale**：去色后仍能读出净利正负、是否免单、哪些里程碑已达成、距通关项数、只读条（`ACC-VIS-08`）。

**Screen reader**：本作**不声称**菜单可被 OS 读屏使用（`ACC` Known Limitations）；补偿是全部信息文字化 + 固定初始焦点 + 焦点驱动滚动。不新增朗读承诺。

---

## 13. Localization

- 所有玩家可见字符串走 `tr()`；里程碑名称与条件文案来自目录，不写死在本屏（`ACC-COG-08`）。
- 中文为基准；允许 40% 扩写。列名上限 6 字；里程碑条件文案在 150% 下最多两行，超出按目录文案原文换行，不截断成歧义句。
- 金额：整数元、无小数、无千分位（`1,234` 仅出现在示意，实际按项目格式）；正数 `+`、负数 `−`（U+2212）。日期 `YYYY-MM-DD`，不随语言改序（见 OQ7）。
- 「尚无纪录」「未开始」「免单」「已达成」「通关必需」为独立术语，不拼进数字句子，避免语序变化导致数字与标签错位。

---

## 14. Acceptance Criteria

**焦点与入口**

- [ ] `RCD-AC01` 打开时焦点落在第一条有纪录的档口行；没有任何纪录行时落在「返回」（`UIF-AC37`）。
- [ ] `RCD-AC02` 空生涯（`runs_settled = 0`、`stations` 空）只显示「还没有账本」与一句说明，不显示全 0 表、里程碑组或 0 局合计（`REC-AC8`）。
- [ ] `RCD-AC03` 目录中尚无纪录的档口显示「尚无纪录」行，不显示 `0`；该行不获焦。
- [ ] `RCD-AC04` 进入与退出各只发一次 `screen_changed`；「返回」发一次 `ui_button_activated(records_back)`。

**数值与口径**

- [ ] `RCD-AC05` 档口行逐字段等于 `snapshot().stations[id].best`：净利符号、`return_percent_x10` 一位小数、总值、日期；与同一局的收据字节级一致（`PAT-12`）。
- [ ] `RCD-AC06` `entry_fee = 0` 的档口显示「免单」；`return_percent_x10 = null` 且入场费非 0 时显示「—」，不显示 `∞` / `NaN` / `0%`。
- [ ] `RCD-AC07` 净利 `+19` 显示 `+19`、`−5` 显示 `−5`、`0` 显示 `0`；数值右对齐等宽。
- [ ] `RCD-AC08` 界面不调用 `return_ratio` / `broke_even` 计算，不按净利或回本率重排行，不按时间戳排序（`REC-R15`、`UIF-R23`）。
- [ ] `RCD-AC09` 里程碑按「未达成 / 已达成」分组，顺序取目录顺序；每条显示名称、条件文案、`当前 / 阈值 单位`、达成日期（有则显示）。
- [ ] `RCD-AC10` `milestone_progress().current = null` 显示「未开始」而非 `0`；阈值 `500`、当前 `340` 显示 `340 / 500`（`REC-AC7`）。
- [ ] `RCD-AC11` 已达成行显示冻结 `threshold_at_earn` 与「已达成」文字，即使目录阈值后来调高也不改写（见 OQ5）。

**完成、异常与只读**

- [ ] `RCD-AC12` 通关状态只取 `snapshot().completed_at != null`；锁存后即使新增必需里程碑也保持「已通关」，不撤销（`REC-AC19`）。
- [ ] `RCD-AC13` 未通关时显示「距离通关还差 N 项」并列出未达成必需项；N 为目录与已达成集合的展示计数，不参与判定（`UIF-R23`）。
- [ ] `RCD-AC14` 目录无必需项时不显示「还差 0 项」也不显示「已通关」，显示「尚未配置通关条件」（文案待 OQ9 确认）。
- [ ] `RCD-AC15` 已从目录移除的已达成里程碑不显示；组计数只数目录内条目。
- [ ] `RCD-AC16` 存档不可写时显示只读条「本次会话不保存」；数据仍完整可读，无写入调用。
- [ ] `RCD-AC17` 快照不可用时显示「纪录暂不可用」与「返回」，不显示 0、旧值或占位表。
- [ ] `RCD-AC18` 进入本屏不调用 `prepare_run` / `prepare_run_result` / `commit`；离开不改变任何纪录（`UIF-AC18` 同口径）。

**输入与滚动**

- [ ] `RCD-AC19` 只用键盘或只用手柄都能浏览全部可聚焦行并返回；焦点移出视口时自动滚入且环不被裁切。
- [ ] `RCD-AC20` 滚轮只滚动内容区，不移动焦点；`ui_cancel` 从任意滚动位置返回主菜单并归还焦点到「纪录」，从不退出游戏。
- [ ] `RCD-AC21` 行聚焦后按 `ui_accept` 不产生任何导航、购买或写入（只读行无隐藏动作）。
- [ ] `RCD-AC22` 文件打开到可交互 ≤ 500 ms（本机档，无网络）；滚动无掉帧。

**无障碍与本地化**

- [ ] `RCD-AC23` 所有文字/背景组合实测 ≥ 4.5:1；正负、已达成、免单、只读条均有颜色之外的通道（`ACC-VIS-06/08`）。
- [ ] `RCD-AC24` 100/125/150% 三档无重叠、无截断、无横向滚动；150% 档口行按三行网格重排且金额列不重叠（`ACC-VIS-09/10`）。
- [ ] `RCD-AC25` 焦点环完整可见，命中区 ≥ 44×44 px，「返回」≥ 56 px 高；所有字符串来自 `tr()`；无计时器、无自动关闭。

---

## 15. Open Questions

| # | 问题 | 归属 | 何时解决 | 状态 |
|---|---|---|---|---|
| 1 | **档口行顺序**未在 GDD 定义：本规格用「目录顺序，已移除档口按 id 升序追加」，保证同一状态同一结果（`UIF-R28`）。是否改为「最近达成优先」？后者需读时间戳，与 `REC-R15` 冲突。 | ux-designer + game-designer | 写 epic 前 | 未决 |
| 2 | **已移除档口的展示名**：`StationRecord` 没有 `display_name`，`get_station` 已无此 id。本规格显示原始 id + 「（档口已移除）」，冻结数值照常。是否需要在存档 schema 补冻结名？ | systems-designer | 架构阶段 | 未决 |
| 3 | **完成横幅归属冲突**：`REC-UI4` 要求纪录页显示「一次性横幅」，`UIF-R23` 却把一次性横幅的首发固定为收据/下一次主菜单，且不得重复。本规格在纪录页只显示静态「已通关 · 日期」状态，不播庆祝。需二者取一。 | ui-flow + records 作者 | 写 epic 前 | 未决 |
| 4 | **新增档口空状态行**：`REC-UI2` 要求「新档口尚无纪录时显示明确的空状态行」。空生涯被 `REC-UI6` 要求整页空状态，两者在「一局未打」时冲突；本规格优先 `REC-UI6`。有局数时才显示逐档口空状态行。 | ux-designer | 写 epic 前 | 未决 |
| 5 | **已达成里程碑阈值的展示**：达成后目录阈值可被调高；本规格用冻结 `threshold_at_earn`，但条件文案仍取当前目录，可能出现「条件写 500、进度 340 / 300」。是否需要目录提供冻结条件文案，或已达成行隐藏进度？ | records 作者 + writer | Phase 4 | 未决 |
| 6 | **只读行的可聚焦含义**：`UIF-UI` 要求初始焦点为「第一条纪录行」，`PAT-14` 要求整行可聚焦，但行内没有动作。本规格接受「聚焦但 `ui_accept` 无动作」。是否改为行聚焦显示底部详情行（会新增信息通道）？ | ux-designer + input | 垂直切片前 | 未决 |
| 7 | **日期格式与时区**：`achieved_at` 是 UTC ISO-8601；本规格显示 `YYYY-MM-DD` 的 UTC 日期段。本地时区跨日时可能与玩家记忆差一天；是否改本地日期并按语言格式化？ | ux-designer + writer | 写 epic 前 | 未决 |
| 8 | **「纪录暂不可用」的恢复路径**：本屏没有重试入口，`retry_load()` 只在载入错误屏/主菜单。若快照在菜单可用、进纪录页失败，玩家只能返回。是否需要屏内「重试」？ | ui-flow | Phase 4 | 未决 |
| 9 | **无必需里程碑时的玩家文案**：`NO_COMPLETION_PATH` 是数据校验警告。本规格暂显示「尚未配置通关条件」，但这是把开发警告暴露给玩家。是否改为隐藏整个通关块？ | game-designer + writer | 写 epic 前 | 未决 |
| 10 | **只读条与主菜单只读说明的重复**：主菜单已有 `ReadOnlyNotice`；本屏再显示紧凑条是否造成噪音？本规格选择显示，因为纪录页是核对历史的地方。 | ux-designer | 垂直切片前 | 未决 |
| 11 | **生涯合计是否属于本屏**：`REC-UI2–4` 未要求显示 `career` 三项；本规格在标题区显示，用于「终身」语境。是否删掉以减少首屏数字？ | game-designer + ux-designer | 写 epic 前 | 未决 |
