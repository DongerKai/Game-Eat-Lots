# UX Spec: 收据屏（ReceiptScreen）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Journey Phase(s)**: 一局结束 → 局外（`Menu` 根屏，替换主菜单）
> **Screen / Flow Name**: `ReceiptScreen`
> **Platform Target**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主、手柄部分支持、无触屏；参考分辨率 1280×720，支持至 3840×2160 与 21:9
> **Related GDDs**: `design/gdd/ui-flow.md`（规则 3–4、9–14、23、25–28）、`design/gdd/settlement-and-entry-fee.md`、`design/gdd/run-flow.md`、`design/gdd/meta-upgrades.md`、`design/gdd/records-and-milestones.md`、`design/gdd/first-run-onboarding.md`、`design/gdd/input.md`、`design/gdd/save-system.md`
> **Sources**: 上述 GDD 的 `## UI Requirements`；`design/ux/interaction-patterns.md`；`design/ux/hud.md`（结构与质量基线）；`design/accessibility-requirements.md`；`.claude/docs/technical-preferences.md`
> **Sibling Specs**: `design/ux/hud.md`、`interaction-patterns.md`、`main-menu.md`、`upgrade-screen.md`、`records-screen.md`、`settings-modal.md`
> **Accessibility Tier**: Standard（`design/accessibility-requirements.md`）
> **Style Reference**: `design/art/art-bible.md` § 4 Color System；收据纸与等宽数字沿用 `SETT` Visual；**§ 7 UI/HUD Visual Direction 尚未撰写**（见 Open Questions）
> **Template**: UX Spec

> **范围边界**：本文件只覆盖一局结算后的那一张收据：逐行明细、合计、回本判定、新纪录、新达成、首局讲解与两个出口。它不拥有 `RunResult` 的任何字段、不重算净利/回本率、不判断破纪录；局内 HUD 与主菜单另有规格。判断标准：玩家此刻是否在核对“这一局吃了什么、赚了还是亏了、接下来再来一局还是回菜单”。

## 引用代号

| 代号 | 出处 |
|---|---|
| `UIF-Rn` / `UIF-ACn` / `UIF-F:<名>` | `design/gdd/ui-flow.md` Core Rules / Acceptance Criteria / Formulas |
| `UIF-UI` | 同文件 `## UI Requirements` 的屏幕契约表 |
| `SETT-Rn` / `SETT-UI` | `design/gdd/settlement-and-entry-fee.md` |
| `RUN-Rn` / `RUN-UI n` | `design/gdd/run-flow.md` |
| `META-Rn` / `META-ACn` / `META-UI` | `design/gdd/meta-upgrades.md` |
| `REC-Rn` / `REC-UI` | `design/gdd/records-and-milestones.md` |
| `ONB-Rn` / `ONB-S9` | `design/gdd/first-run-onboarding.md` |
| `SAVE-Rn` / `SAVE-UI` | `design/gdd/save-system.md` |
| `INP-Rn` | `design/gdd/input.md` |
| `PAT-NN` | `design/ux/interaction-patterns.md` |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. Purpose & Player Need

收据是这一局的**账本原件**：它必须让玩家在三十秒内回答三个问题——我吃了什么、总值多少、扣掉入场费后赚了还是亏了。它不是庆祝页，也不是教学页；即使亏钱，它也要平静、完整、逐行可核。收据的全部数字只有一个来源：`RunResult`。界面不排序、不合并、不补算、不显示“本可以多赚”，否则玩家会看到第二本账（`UIF-R11`、`SETT-R9`、`PAT-12`）。

**The player goal**：核对这一局的结果，判断“再来一局”还是“回菜单先升级”。

**The game goal**：把 `RunResult` 逐字段如实呈现，并把回本判定、新纪录、新达成与首局讲解放在同一张纸上，不要求玩家自己比较或记忆。

---

## 2. Player Context on Arrival

| Question | Answer |
|---|---|
| What was the player just doing? | 在局内吃完最后一波 / 回合耗尽 / 无可吃菜，画面停在最后一帧（`RUN-UI 6`） |
| What is their emotional state? | 期待或挫败；刚亏钱时需要的是可核对的账，不是安慰或庆祝 |
| What cognitive load are they carrying? | 中：刚做完一串取舍，脑内还留着“差一点回本”的估算 |
| What information do they already have? | 局内实时净利与回本差额；但不一定记得每一道的单价与饱腹 |
| What are they most likely trying to do? | 先看净利与是否回本，再看是哪几道菜拖累/拉高 |
| What are they likely afraid of? | “是不是算错了”“为什么和我记的不一样”“新纪录是不是没算上” |

**Emotional design target**：像接过一张打印清楚的收据——安静、可信、可以逐行核对；输赢都体面。

---

## 3. Navigation Position

```
Menu
  ├── MainMenu
  └── ReceiptScreen（本屏；根屏，替换 MainMenu，不叠在其上）
        ├── 再来一局 → Run（重开）
        └── 返回菜单 → MainMenu（替换根）
```

**Modal behavior**：本屏不是模态，也没有自己的模态。完成横幅是 PAT-08 非模态提示；「新达成」条固定在底部，不阻塞、不夺焦。屏幕栈深度恒为 1（`UIF-R4`、`UIF-R13`）。

**Reachability — all entry points**：

| Entry Point | Triggered By | Notes |
|---|---|---|
| 一局结算 | `Run → Menu` 生效的同一帧交付 `RunResult` | `receipt_open_latency_frames = 0`（`UIF-R9`） |
| 结算事务放弃 | `prepare_run` 返回 `INCONSISTENT_RESULT` / `OVERFLOW` / `DATA_NOT_READY` | 仍打开收据，显示原始值与异常标注（`UIF-R10`） |
| 放弃/进入失败 | — | **不打开收据**（`UIF-R16`、`UIF-R19`） |

---

## 4. Entry & Exit Points

**Entry table**：

| Trigger | Source Screen / State | Transition Type | Data Passed In | Notes |
|---|---|---|---|---|
| `RunResult` 交付 | `Run` → `Menu` | 根屏替换（同帧） | `RunResult` 引用、`previous_best`（若有）、新达成里程碑、完成标志、`FirstRunExplanation`（首局） | 不插加载屏；最终画面保留到覆盖帧（`RUN-UI 5/6`） |
| 结算失败 | 同上 | 同帧 | 原始 `RunResult` + 错误码 | 标注「结算数据异常，未入账、未记录」 |

**Exit table**：

| Exit Action | Destination | Transition Type | Data Returned / Saved | Notes |
|---|---|---|---|---|
| 再来一局 | `Run` | 请求 `Menu → Run` | 全新 `RunConfig`（新快照、`is_first_run = false`、新 `run_id`） | 请求失败留在本屏并给原因（`UIF-R12`） |
| 返回菜单 | `MainMenu` | 根屏替换 | 不再结算、不再入账 | 焦点「开始营业」（`UIF-R13`） |
| `ui_cancel` | 先跳过滚动，再回主菜单 | 同上 | — | 永不直接退出（`UIF-R7`） |
| 窗口关闭 | `Quit` | 直接 | `flush()` | 发 `receipt_closed(reason = quit)`（`UIF-R16`、`UIF-R25`） |

---

## 5. Layout Specification

### 5.1 ASCII Wireframe

```
┌──────────────────────────────────────────────────────────────────────┐
│ 本局收据 · 食堂                                 [再来一局] [返回菜单]│
│ 钱包  +19 元                                       [未保存]          │
├──────────────────────────────────────────────────────────────────────┤
│ 菜名          价格   基础   腻度   实际   单位价值   有效单位价值    │
│ 红烧肉          16     3      —      3      5.3        5.3           │
│ 清蒸鱼          18     4     +1      5      4.5        3.6           │
│ …（滚动区：第 1–7 行 / 共 10 行）                                    │
├──────────────────────────────────────────────────────────────────────┤
│ 总值 +104    入场费 −85    净利 +19    [回本]    回本率 122.3%       │
│ 容量 12 / 20 格    浪费 2 格    回合 8 / 10    腻度额外 3 格         │
├──────────────────────────────────────────────────────────────────────┤
│ [新纪录]  旧最佳 +12                                                 │
│ 首局讲解：教学价 40 / 正式 85（仅 override 时；降级无此句）（两段）  │
│ 替换建议：…        首笔升级建议：…                                   │
├──────────────────────────────────────────────────────────────────────┤
│ 新达成：首次回本 · 食堂回本                     [完成横幅（一次性）] │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 Layout Zones

| Zone Name | Description | Approximate Size | Scrollable? | Overflow Behavior |
|---|---|---|---|---|
| A 抬头 | 「本局收据」+ 档口名；右上两个出口；钱包与标记 | 全宽 × 14% | 否 | 150% 时档口名折行，按钮不折 |
| B 行表 | 表头 + 逐行明细，严格按 `rows` 顺序 | 全宽 × 38% | 是（纵向） | 行折 2–3 行、行高增加；不横向滚动、不缩字号 |
| C 合计 | 总值、入场费、净利、回本判定、回本率、容量、浪费、回合、腻度额外 | 全宽 × 22% | 与 B 同一滚动体 | 单列；净利与回本判定不折到屏外 |
| D 印章与讲解 | 新纪录 + 旧最佳；首局讲解与建议（教学价句**仅** `plan.has_entry_fee_override()` 时出现） | 全宽 × 18% | 同上 | 无内容时整块不占位 |
| E 底部条 | 「新达成」固定条 + 滚动读数 + 完成横幅 | 全宽 × 8% | 否 | 不推移 A–D；最多一行 + 一行读数 |

**滚动策略**：A 与 E 固定；B–D 属于同一个纵向滚动体（不是三个独立滚动区）。↑/↓ 与滚轮按 48 px（150% 时 72 px）滚动；底部读数写「第 X–Y 行 / 共 N 行」。这样 150% 时长收据可读，且不会出现嵌套滚动。

### 5.3 Component Inventory

| Component Name | Type | Zone | Purpose | Required? | Reuses Existing Component? |
|---|---|---|---|---|---|
| `ReceiptHeader` | 文本组 | A | 标题、档口名 | 是 | 否（收据纸样式） |
| `ReceiptActions` | 按钮 ×2 | A | 再来一局 / 返回菜单 | 是 | 是 — PAT-01、PAT-13 |
| `ReceiptWallet` | 文本 + 标记 | A | 结算后钱包；「未保存」/「入账失败」 | 是 | 是 — 主菜单钱包样式 |
| `ReceiptTable` | 表头 + 行模型 | B | 逐行明细，行数 = `|rows|` | 是 | 否（等宽数字、右对齐） |
| `ReceiptTotals` | 文本组 | C | 合计与回本判定 | 是 | 否 |
| `RecordStamp` | 印章 + 旧最佳 | D | 仅在 `net_profit > 0` 的新纪录 | 条件 | 否 |
| `FirstRunExplanation` | 文本块 | D | 单位价值说明、替换建议、首笔升级建议；教学价说明仅 `plan.is_first_run ∧ plan.has_entry_fee_override()` 时出现 | 条件（首局） | 否 |
| `MilestoneStrip` | 非模态条 | E | 「新达成」名单 | 条件 | 是 — PAT-08 |
| `CompletionBanner` | 非模态横幅 | E | 通关一次性横幅 | 条件 | 是 — PAT-08 |
| `ScrollReadout` | 文本 | E | 可见行范围 / 总行数 | 是 | 否 |

**Focus order（左→右）**：`再来一局` → `返回菜单`。两个按钮固定在抬头右上；滚动体与底部条**不进入焦点链**（它们是只读内容）。**Primary focus on open**：`再来一局`（`UIF-UI`）。滚动期间焦点不变（`UIF-R14`）。

### 5.4 Information Hierarchy

1. **净利与回本判定**——这一局唯一的“赢没赢”（`SETT-R6`）。
2. **总值、入场费、回本率**——净利的两个来源与相对成绩。
3. **逐行明细**——玩家核对“哪道菜拖累/拉高”的原始凭证。
4. **新纪录 / 新达成 / 完成**——一次性事实，不能盖住净利。
5. **首局讲解**——只对首局出现，放在账本之后，不抢正文。
6. **钱包**——结算后的余额，供下一局决策。

---

## 6. States & Variants

| State Name | Trigger | What Changes Visually | What Changes Behaviorally | Notes |
|---|---|---|---|---|
| `Rolling` | 收据打开且 `receipt_roll_duration > 0` | 总值/净利/百分比从 0 滚到终值 | 第一次 `ui_accept`/`ui_cancel` 只跳过滚动；按钮不激活 | 终点逐字段等于 `RunResult`（`UIF-R14`） |
| `Settled` | 滚动结束或跳过 | 全部终值 | 两个按钮可激活 | 焦点仍在 `再来一局` |
| `EmptyRun` | `rows = []` | 行表位置写「这一局什么都没吃」；总值 0、净利 `−entry_fee` | 不显示全 0 表 | 回本判定按净利（`UIF-AC14`） |
| `FreeMeal` | `entry_fee = 0` | 回本率位置写「免单」；回本判定「回本」 | 不出现 `∞` / `NaN` / `100%` | `SETT-R15`、`UIF-AC15` |
| `NewRecord` | `record_broken` 且新最佳 `net_profit > 0` | 印章「新纪录」+ 旧最佳并排 | 首次纪录无旧最佳；负纪录不盖章 | `REC-UI 5` |
| `Milestones` | 本次新达成 ≥ 1 | 底部「新达成：A · B」 | 不阻塞、不夺焦、不可关闭 | 屏幕变化时消失 |
| `Completion` | `career_completed()` | 底部一次性横幅 | 滚动结束后出现；可关闭；不重复 | `UIF-R23` |
| `Aborted` | 结算事务整笔放弃 | 顶部标注「结算数据异常，未入账、未记录」 | 不显示印章；不播结果 cue | 仍显示原始值（`UIF-R10`） |
| `WalletOverflow` | 入账返回 `WALLET_OVERFLOW` | 钱包旁「入账失败」，钱包保持原值 | 不显示「+N」成功语义 | `META-AC6` |
| `Unsaved` | `APPLIED_UNSAVED` / 写盘失败 | 钱包旁「未保存」 | 保留内存值 | `UIF-R21`、`SAVE-UI 2` |
| `FirstRun` | `plan.is_first_run = true` 且 `plan.has_entry_fee_override() = true` | D 区出现讲解块，含教学价说明（40 / 85） | 无额外操作 | `ONB-R14` |
| `FirstRunDegraded` | `plan.is_first_run = true` 但 `plan.has_entry_fee_override() = false` | D 区只出现单位价值说明、替换建议与首笔升级建议，**没有**教学价句；入场费行显示档口费（MVP 85） | 无额外操作；文案与费用行必须一致 | ADR-0020 降级表第 3 行（`B4` 无 `fee_note`）；`RCP-AC21` |

---

## 7. Interaction Map

### 7.1 Navigation Inputs

| Input | Platform | Action | Visual Response | Audio Cue | Notes |
|---|---|---|---|---|---|
| ← / →、A / D、左摇杆左右 | 键盘 / 手柄 | 在两个出口之间移动焦点 | 2 px 交互蓝外框 | 无（焦点移动不发声） | 只有两个焦点位 |
| ↑ / ↓、W / S、左摇杆上下、滚轮 | 键盘 / 手柄 / 鼠标 | 滚动收据体 | 行表/合计移动；底部读数更新 | 无 | 焦点不移动；一步 48 px（150% 72 px） |
| 鼠标移动 | 鼠标 | 按钮悬停 | 浅色底 | 无 | 不改变键盘焦点（`PAT-01`） |

### 7.2 Action Inputs

| Input | Platform | Context | Action | Response | Animation | Audio Cue | Notes |
|---|---|---|---|---|---|---|---|
| `ui_accept` / 左键 | 键盘 / 手柄 / 鼠标 | 滚动中 | 只跳过滚动 | 全部字段立即到终值 | 0 ms | `number_roll_finished(skipped = true)` | 不激活按钮（`UIF-R14`） |
| `ui_accept` / 左键 | 同上 | `Settled` + 再来一局 | 组装全新 `RunConfig` 并请求 `Menu → Run` | 成功进局；失败原地给原因 | — | `sfx.ui.click` | 绝不复用上一局状态（`UIF-R12`） |
| `ui_accept` / 左键 | 同上 | `Settled` + 返回菜单 | 根屏替换为 `MainMenu` | 主菜单出现 | 场景淡入 ≤ 200 ms | `sfx.ui.click` → `sfx.flow.menu` | 不再结算（`UIF-R13`） |
| `ui_cancel` | 键盘 / 手柄 | 完成横幅可见 | 关闭横幅 | 横幅消失 | 淡出 | — | `back_target` ③ |
| `ui_cancel` | 键盘 / 手柄 | 滚动中（无提示） | 跳过滚动 | 终值立即显示 | 0 ms | `number_roll_finished(skipped = true)` | `back_target` ④ |
| `ui_cancel` | 键盘 / 手柄 | `Settled`（无提示） | 返回主菜单 | 根屏替换 | 淡入 | — | `back_target` ⑤ |
| 窗口关闭 | 系统 | 任意 | 请求 `Quit`，先 `flush()` | — | — | — | 发 `receipt_closed(quit)`（`UIF-R16`） |

### 7.3 State-Specific Behaviors

| State | Input Restriction | Reason |
|---|---|---|
| `Rolling` | 两个按钮不可激活；`ui_accept`/`ui_cancel` 只跳过 | `UIF-R14` |
| `Aborted` | 两个按钮仍可用；无结果音效 | 账虽异常，玩家仍须能离开 |
| 完成横幅可见 | `ui_cancel` 先关横幅，不返回菜单 | `back_target` ③ 优先于 ⑤ |
| 无模态 | 本屏不压 `Modal` 上下文 | 无设置/确认（`UIF-R5`） |

**`ui_cancel` 落点**：可关闭提示 → 关闭；滚动中 → 跳过；否则 → 替换根为 `MainMenu`。**永不直接退出**（`UIF-R7`）。

---

## 8. Data Requirements

| Data Element | Source System | Update Frequency | Who Owns It | Format | Null / Missing Handling |
|---|---|---|---|---|---|
| 逐行明细 | `RunResult.rows`（按吃下顺序） | 收据创建时一次 | 结算与入场费 | `dish_id, price, satiety, extra_satiety, effective_satiety, unit_value, effective_unit_value` | 缺字段 → 该格 `—`，不补算 |
| 菜名 | `数据配置.get_dish(dish_id).display_name` | 创建时一次 | 数据配置 | 字符串 | 查不到 → `—` + 保留 `dish_id`（debug） |
| 总值 / 入场费 / 净利 | `total_value` / `entry_fee` / `net_profit` | 创建时一次 | 结算与入场费 | int64 / int / int64 | 缺失 → 显示 `—`，不显示 0 |
| 回本判定 | `net_profit ≥ 0` 派生谓词 | 创建时一次 | 结算与入场费 | 「回本」/「未回本」 | 不由界面重算（`SETT-R6`） |
| 回本率 | `return_percent_x10`（`entry_fee = 0` 时为 null） | 创建时一次 | 结算与入场费 | `122.3%` / 「免单」 | null → 「免单」，不显示 `∞` |
| 容量 / 回合 / 腻度合计 | `capacity_*` / `wasted_capacity` / `turns_*` / `cloy_extra_total` | 创建时一次 | 胃容量 / 时间与回合 / 腻度 | 整数 | 缺失 → `—` |
| 旧最佳 / 印章 | `record_broken(station_id, new_best, previous_best)` | 结算时一次 | 纪录与里程碑 | 旧 `net_profit` 或 null | `previous_best = null` → 不显示旧值 |
| 新达成 | `milestone_earned(id)` + 目录名称 | 结算时 | 纪录与里程碑 | 名称列表 | 未知 id → 「已归档」 |
| 完成 | `career_completed()` | 结算时 | 纪录与里程碑 | bool | 只显示一次 |
| 钱包 | `局外升级.get_wallet()` | 结算后 | 局外升级 | int64，带符号 | 查询失败 → `—` |
| 入账状态 | `prepare_run_result` 返回值 / `commit` 结果 | 结算时 | 局外升级 / 存档 | `APPLIED` / `APPLIED_UNSAVED` / `WALLET_OVERFLOW` | 未返回 → 不绘制收据（同帧问题） |
| 首局讲解 | `FirstRunExplanation` payload（`fee_note` 仅当 `plan.is_first_run ∧ plan.has_entry_fee_override()` 时非空，由 `FirstRunPlanner` / `B4` 决定） | 首局创建时 | 首局引导 | 两段文案 + 逐行说明 + 两条建议；教学价句可为空 | 教学价句缺失（降级）→ 只省略该句，其余照常显示；整块缺失 → 整块不显示 |
| 档口名 | `get_station(station_id).display_name` | 创建时 | 数据配置 | 字符串 | 缺失 → `—` |

> **禁止**：不排序、不合并、不截断行；不显示 `opt_value`、`effective_unit_value` 之外的“最优值”；不把 `rows[].price` 相加得到总值；不自行判断破纪录或通关（`PAT-12`、`UIF-R11`）。

---

## 9. Events Fired

| Player Action | Event Fired | Payload | Receiver System | Notes |
|---|---|---|---|---|
| 收据根屏建立 | `settlement_started(run_id)` | `run_id` | 音效反馈 | 同帧、任何动画之前，恰好一次（`UIF-R9`） |
| 数字滚动开始/结束 | `number_roll_started(field, from, to, duration_ms)` / `number_roll_finished(field, skipped)` | 字段级 | 音效反馈 | 每个字段各一次；跳过时 `skipped = true` |
| 结算事务放弃 | `settlement_aborted(run_id, code)` | 错误码 | 音效反馈 | 与收据同帧（`UIF-R10`） |
| 激活控件 | `ui_button_activated(control_id)` | 控件 id | 音效反馈 | 跳过滚动时为 0 次（`UIF-F:activation_event_count`） |
| 再来一局 | `receipt_closed(reason = restart)` + `request_transition(Menu → Run)` | 新 `RunConfig` | 音效 / 场景与流程管理 | 先关收据语义，再请求切换 |
| 返回菜单 | `receipt_closed(reason = menu)` + 根屏替换 | — | 音效 / 界面流程 | 不重复入账 |
| 窗口关闭 | `receipt_closed(reason = quit)` | — | 音效 | 随后 `flush()` |

---

## 10. Transitions & Animations

| Transition | Trigger | Direction / Type | Duration (ms) | Easing | Interruptible? | Skipped by Reduced Motion? |
|---|---|---|---|---|---|---|
| 收据覆盖 | `RunResult` 交付 | 同帧替换根屏，无淡入 | 0 | — | — | 是（本就 0） |
| 数字滚动 | 收据打开 | 0 → 终值 | 300–1500，默认 800（数据值） | 线性 | 是（跳过） | 是 → 0 ms |
| 印章出现 | 滚动结束 | 缩放 0.9 → 1.0 | 180 | ease-out | 否 | 是 → 瞬时 |
| 新达成条 | 滚动结束 | 淡入 | 150 | 线性 | 否 | 是 |
| 完成横幅 | `career_completed` 且滚动结束 | 淡入 | 180 | ease-out | 否 | 是 |
| 离开本屏 | 再来一局 / 返回菜单 | 场景淡入（由场景流程） | ≤ 200 | 线性 | 否 | 是 |

> 滚动只影响呈现，不产生新的数值事实；`reduce_motion` 时 `D = 0`，立即发 `started(0)` + `finished(skipped = false)`（`UIF-R14`）。

---

## 11. Input Method Completeness Checklist

**Keyboard**
- [ ] 两个出口可用 ←/→ 到达；↑/↓ 滚动收据体，焦点不丢失
- [ ] 只用键盘可完成：读完整张收据、跳过滚动、再来一局、返回菜单
- [ ] 滚动中按 `ui_accept` / `ui_cancel` 只跳过，不激活按钮
- [ ] `ui_cancel` 在滚动结束后返回主菜单，不直接退出
- [ ] 底部有「第 X–Y 行 / 共 N 行」读数，键盘滚动时同步更新

**Gamepad**
- [ ] 十字键/左摇杆上下滚动、左右切焦点；A 确认、B 取消
- [ ] 无模拟量精度要求；回滞 0.60 / 0.40
- [ ] 手柄断开时焦点停在原按钮
- [ ] 不绘制键位图标，只写动作语义

**Mouse**
- [ ] 两个按钮有悬停态，与焦点态可区分
- [ ] 滚轮滚动收据体；不改变键盘焦点
- [ ] 按钮命中区 ≥ 56 px 高、≥ 44 px 宽
- [ ] 右键无动作（不误触跳波）

**Touch**：不适用——无触屏目标，不提供虚拟摇杆（`INP-OQ5`）。

---

## 12. Accessibility

**Text contrast**：

| Text Element | Background Context | Required Ratio | Current Ratio | Pass? |
|---|---|---|---|---|
| 行表与合计正文 | 米白 `#F5F1E6` | ≥ 4.5:1 | 墨黑 14.6:1 | 是 |
| 净利正数 / 回本 | 米白 | ≥ 4.5:1 | 回本红 4.6:1 | 是 |
| 净利负数 / 未回本 | 米白 | ≥ 4.5:1 | 墨黑 14.6:1 | 是 |
| 表头与注释 | 米白 | ≥ 4.5:1 | 深灰 7.5:1 | 是 |
| 腻度追加 `—` | 米白 | ≥ 4.5:1 | 深灰 7.5:1 | 是 |

**Colorblind-safe**：

| Element | Colorblind Risk | Mitigation |
|---|---|---|
| 回本/未回本 | 红/墨黑 | 文字「回本」/「未回本」+ 净利符号 + 百分比 |
| 新纪录 | 红印章 | 「新纪录」文字 + 旧最佳并排 |
| 正负金额 | 红/墨黑 | `+` / `−`（U+2212）+ 右对齐等宽数字 |
| 「未保存」 | 颜色 | 文字「未保存」/「入账失败」 |

**Text scaling**：100/125/150%。150% 时行折 2–3 行、行高增加并纵向滚动；金额列与单位价值列不重叠；合计区单列；净利与回本判定不被折出可滚动范围（`ACC-VIS-10`）。不横向滚动、不缩字号。

**Focus**：2 px 外框 + 2 px 间距；滚动时焦点环不动、不裁切。只读内容不进入焦点链（`PAT-01`）。

**Grayscale**：去色后仍能读出净利正负、是否回本、新纪录、哪一行腻度追加为 0、是否「未保存」（`ACC-VIS-08`）。

---

## 13. Localization

- 所有字符串走 `tr()`；数字/百分比来自读数，不拼进句子（`ACC-COG-08`）。
- 中文基准，允许 40% 扩写；表头上限 6 字/列，列名可折两行；按钮上限 6 字。
- 金额整数元、无小数、无千分位；正数 `+`、负数 `−`（U+2212）；回本率一位小数 + `%`（`122.3%`）。
- 「免单」「这一局什么都没吃」「新达成」为固定文案；不得在语言切换后改写数值格式。
- 日期不在本屏出现（`RunResult` 不带日期）。

---

## 14. Acceptance Criteria

**打开与数据**

- [ ] `RCP-AC01` `RunResult` 在帧 N 交付，收据在帧 N 成为根屏；`receipt_open_latency_frames = 0`（`UIF-AC11`）。
- [ ] `RCP-AC02` 行数等于 `|rows|`，顺序与 `rows` 完全一致；不存在排序、合并、截断或分页。
- [ ] `RCP-AC03` 每行显示菜名、价格、基础饱腹、腻度追加、实际饱腹、静态单位价值、有效单位价值；`extra_satiety = 0` 显示 `—`。
- [ ] `RCP-AC04` 合计区显示总值、入场费（负项）、净利（带符号）、回本判定、回本率、已用/总容量、浪费容量、已用/总回合、腻度额外占用合计；逐字段等于 `RunResult`。
- [ ] `RCP-AC05` 回本判定只按 `net_profit ≥ 0` 显示文字，不由界面比较两个数；净利 0 显示「回本」。
- [ ] `RCP-AC06` `return_percent_x10 = 1223` 显示 `122.3%`；`entry_fee = 0` 显示「免单」，不出现 `∞` / `NaN` / `100%`。
- [ ] `RCP-AC07` `rows = []` 显示「这一局什么都没吃」，不显示全 0 表；总值为 0、净利为 `−entry_fee`。

**新纪录、新达成与完成**

- [ ] `RCP-AC08` 新最佳 `net_profit > 0` 时显示「新纪录」印章与旧最佳；`previous_best = null` 时不显示旧值；`net_profit ≤ 0` 时不盖章、不显示安慰文案。
- [ ] `RCP-AC09` 「新达成」条列出本次所有新达成名称，不阻塞、不夺焦、不重复。
- [ ] `RCP-AC10` `career_completed()` 在收据打开时：横幅等滚动结束后显示一次；离开收据后不在任何新收据上显示（`UIF-AC32`）。

**滚动与焦点**

- [ ] `RCP-AC11` 滚动时长取 `RECEIPT_ROLL_SECONDS`（0.3–1.5，默认 0.8）；终点逐字段精确等于 `RunResult`；`reduce_motion` 时为 0 且立即到终值。
- [ ] `RCP-AC12` 滚动中第一次 `ui_accept` / `ui_cancel` 只跳过滚动，不发 `ui_button_activated`；第二次才激活控件。
- [ ] `RCP-AC13` 跳过或滚动结束后焦点仍在「再来一局」；两个按钮位置不随净利正负变化。
- [ ] `RCP-AC14` 只用键盘与只用手柄都能滚动整张收据并读到合计；底部「第 X–Y 行 / 共 N 行」随滚动更新。
- [ ] `RCP-AC15` `ui_cancel` 在无提示、无滚动时返回主菜单；有完成横幅时先关闭横幅；从不直接退出。

**出口与异常**

- [ ] `RCP-AC16` 「再来一局」组装全新 `RunConfig`：`is_first_run = false`、`entry_fee_override = null`、新快照、新 `run_id`；不复用上一局任何状态。
- [ ] `RCP-AC17` 「再来一局」的切换请求被拒绝：留在收据、显示原因、焦点回「再来一局」、不组装第二份配置。
- [ ] `RCP-AC18` 「返回菜单」只替换根屏；`prepare_run` / `prepare_run_result` 调用计数不增加，焦点回「开始营业」。
- [ ] `RCP-AC19` 结算事务放弃时显示「结算数据异常，未入账、未记录」，仍显示原始值，不显示印章，不播结果 cue。
- [ ] `RCP-AC20` `WALLET_OVERFLOW` 显示「入账失败」；`APPLIED_UNSAVED` 显示「未保存」；两者钱包显示规则符合 `META-UI` / `SAVE-UI 2`。
- [ ] `RCP-AC21` 首局显示讲解块：`plan.is_first_run ∧ plan.has_entry_fee_override()` 时含教学价说明（40 / 85）+ 逐行单位价值说明 + 一条替换建议 + 一条首笔升级建议；**降级**（首局但 `plan.has_entry_fee_override() = false`）时入场费行显示档口费（MVP 85）、**不出现**任何教学价句、其余三部分照常显示；非首局整块不存在。三种情形下教学价句出现 ⇔ 入场费行 = 40，截图同时证明「40 元的说明不配 85 元的行」（ADR-0020 降级表第 3 行、`UIF-AC48`）。

**无障碍**

- [ ] `RCP-AC22` 所有文字/背景组合实测 ≥ 4.5:1；回本与否、正负、新纪录都有颜色之外的通道。
- [ ] `RCP-AC23` 100/125/150% 三档下无重叠、无横向滚动、无关键信息消失；150% 时金额列不重叠且可滚动读全。
- [ ] `RCP-AC24` 灰度截图仍能读出净利正负、是否回本、新纪录、腻度追加 0 与「未保存」。
- [ ] `RCP-AC25` 焦点环完整可见；按钮命中区 ≥ 56 px 高；所有字符串来自 `tr()`。

---

## 15. Open Questions

| # | 问题 | 归属 | 何时解决 | 状态 |
|---|---|---|---|---|
| 1 | **滚动与跳过的视觉表现**未在模式库定义（`interaction-patterns` Gaps「收据滚动与跳过」）；本规格用「↑/↓ 滚动 + 第 X–Y 行读数」，需确认是否够显眼。 | ux-designer | 写 epic 前 | 未决 |
| 2 | **跳过提示**：`UIF-OQ7` 与模式库 OQ3 都问是否加「按任意键跳过」。本规格不加常驻提示，只在底部读数旁写「↑↓ 滚动 · 任意键跳过」；`reduce_motion` 时不显示后半句。 | writer + ux-designer | Phase 4 | 未决 |
| 3 | **长收据的分段/折叠**（`interaction-patterns` Gaps）：MVP 10 行以内不折叠；150% 时整张收据纵向滚动。是否需要按「行表 / 合计 / 讲解」分段标题？ | ux-designer | 垂直切片前 | 未决 |
| 4 | **收据抬头是否显示结束原因**：`RunResult.end_reason` 存在，但 `UIF-R11` 的字段清单没有它；局内 `Ending` 横幅已显示一次。本规格不重复，只显示档口名。 | game-designer + ux-designer | 写 epic 前 | 未决 |
| 5 | **印章与旧最佳的并排布局**（`interaction-patterns` Gaps）：本规格放在合计区上方一行；是否需要在净利旁常驻？ | art-director + ux-designer | 垂直切片前 | 未决 |
| 6 | **完成横幅与「新达成」条的避让**（`UIF-OQ9`）：本规格让横幅在底部条右侧、滚动结束后出现；若同帧两者都有，横幅优先且不遮挡名单。 | ux-designer | Phase 4 | 未决 |
| 7 | **结算致命错误仍显示原始值**（`UIF-OQ13`）：本规格按暂定执行并加标注；是否会让玩家把已知不一致的收据当成事实？ | game-designer + qa-lead | 架构阶段 | 未决 |
| 8 | **首局讲解块的长度上限**：`ONB` 给两段 + 逐行说明 + 两条建议，未定最大行数；150% 下可能占据整屏。是否允许在讲解块内折叠「逐行说明」？ | writer + ux-designer | 写 epic 前 | 未决 |
