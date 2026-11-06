# Onboarding Layer Design（首局引导层）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Game**: 回本 (Break Even)
> **Platform Targets**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主，手柄部分支持；无触屏。参考 1280×720，支持至 4K 与 21:9
> **Related GDDs**: `first-run-onboarding.md`（ONB-R1–R22）、`ui-flow.md`（结算与首局判定）、`run-flow.md`、`in-game-hud.md`、`input.md`、`audio-feedback.md`
> **Accessibility Tier**: Standard（逐条要求见 `design/accessibility-requirements.md`）
> **Sources**: `design/ux/interaction-patterns.md`（PAT-02/08/11/12）、`design/ux/hud.md`（锚点/受保护区/焦点基线）、ADR-0019（HUD 锚点）、ADR-0020（beat 机、图层、气泡、讲解）
> **Sibling Specs**: `design/ux/pause-overlay.md`、`design/ux/confirm-modal.md`、`design/ux/readonly-notice.md`、`design/ux/error-screen.md`、`design/ux/hud.md`、`design/ux/interaction-patterns.md`
> **Template**: UX Screen Spec

> **范围边界**：本文件覆盖首局引导的**气泡、聚光遮罩、锚点标注、跳过入口**，以及交付给收据的 `FirstRunExplanation` 契约；**不覆盖** HUD 的锚点布局与读数（归 `hud.md`）、不覆盖收据排版（归 `ReceiptScreen` 规格）、不覆盖 beat 机的代码结构（归 ADR-0020）。本层不拥有焦点、不拥有玩法规则、不写存档。

## 引用代号

| 代号 | 出处 |
|---|---|
| `ONB-Rn` / `ONB-ACn` | `design/gdd/first-run-onboarding.md` Core Rules 第 n 条 / Acceptance Criteria 第 n 条 |
| `ONB-F:<名>` | 同文件 Formulas（`bubble_rect` / `bubble_text_cap` / `beats_required`） |
| `UIF-Rn` | `design/gdd/ui-flow.md` Core Rules 第 n 条 |
| `RUN-UI n` | `design/gdd/run-flow.md` UI Requirements 第 n 条 |
| `HUD-Rn` | `design/gdd/in-game-hud.md` Core Rules 第 n 条 |
| `INP-Rn` | `design/gdd/input.md` Core Rules 第 n 条 |
| `ADR-0005` / `ADR-0019` / `ADR-0020` | `docs/architecture/` 下同名 ADR |
| `PAT-nn` | `design/ux/interaction-patterns.md` |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. 目的与玩家需求

首局玩家同时面对约 8 个可读信息流（回合、容量、净利、差额、四张卡、波次、腻度）。引导的任务不是把规则讲全，而是**在玩家需要的那一刻，只讲一件事**：入场费与回本线、价格与饱腹两个数、容量是硬上限、同组重复多占一格。教多了等于没教（`ONB-R5` D6）。

四条立场：

1. **一次一条、只讲一件事**：同一时刻最多一个气泡；B3 替换 B2 而不是并排（`ONB-R7`）。
2. **不替玩家做决定**：不移动焦点、不选菜、不排序、不显示单位价值（`ONB-R10`、`ONB-R13`）。
3. **不遮三个核心读数**：回合、容量、净利/差额永远保持原亮度（`ONB-R12`）。
4. **没有计时器**：不自动推进、不自动关闭、不淡出（`ONB-F:onboarding_timeout = ⊥`）。

---

## 2. 玩家情境

玩家第一次看到 HUD，还在建立「价格 / 饱腹 / 容量 / 回合」的映射；他可能先乱点、先看菜、先按 Esc。引导必须允许这些，且误按不能破坏一局。中途暂停、放弃、退出都要干净拆除，不写任何东西、不留下「气泡在、按键无效」。他不会读长文案：一条气泡最多三行（150% 字号下 48 个 CJK 字符），超过即文案缺陷（`ONB-F:bubble_text_cap`）。

---

## 3. 进入与退出契约

### 3.1 进入

| 条件 | 行为 |
|---|---|
| `run_started` 且 `RunConfig.is_first_run = true` | `OnboardingHost` 从 `IDLE → INTRO`；同帧 `push_context(Blocked, B0_ALLOW)`；显示 B0 模态气泡 |
| `is_first_run = false` | 本层完全隐藏，不压上下文、不读锚点 |
| 锚点尚未布局 | 先压 `Blocked`（防误操作），气泡绘制推迟到锚点可用帧，最多 2 帧；仍为空则底部停靠（`ONB` 边界用例） |
| HUD 首帧未完成 | 同上；不阻塞开局 |

### 3.2 退出

| 出口 | 触发 | 结果 |
|---|---|---|
| 全部完成 | `beats_required` 展示完且 B4 payload 已交付 | 广播一次 `onboarding_finished(ALL_BEATS)`；卸载 |
| 跳过 | 模态 beat 中 `ui_cancel` 或点击「跳过引导」 | 弹 `Blocked`、丢弃剩余 beat、广播一次 `onboarding_finished(SKIPPED)`；**不二次确认**（`ONB-R16`） |
| 暂停 | `ui_pause`（P / 手柄 Start） | 进入 `Paused`，`Blocked` 被 `clear_stack` 清空；本层保持挂载但不可交互 |
| 放弃 / 退出 | `run_discarded` | 立即拆除；不产出 B4、不广播完成、不写任何东西 |
| 故障 / 初始化失败 | `run_faulted` / `init_failed` | 同上；`INIT` 失败时本层从未激活，只做清理 |
| 结算异常 | 无 `RunResult` | B4 静默结束；不补弹窗 |

> 完成信号**不参与**「下一局是否首局」的判定；该判据只由 `runs_settled == 0 ∧ first_run_eligible` 决定（`ONB-R15`、`UIF-R8`、ADR-0020 D1）。

---

## 4. 导航位置与层级

层序固定为 HUD 10 < **引导 15** < 屏幕 20 < 暂停 25 < 模态 30 < 通知 40 < 淡入淡出 50；引导读取 HUD 锚点与受保护矩形，收据与暂停能覆盖它。`OnboardingHost.process_priority = -50`，严格在 `SceneFlow`（−100，先处理暂停）与 `HudHost`（0，先于 HUD 取走确认）之间（ADR-0020 D4、ADR-0005 D5）。
- 本层**不进入** `ScreenHost` 屏幕栈；`ui_stack_depth` 不变。
- 本层只写 `push_context(Blocked, allow)` / `pop_context()`；不写 `run_state`、不改任何局内数字（`ONB-R4`）。

---

## 5. 布局

### 5.1 线框（B0，1280×720）

```
┌──────────────────────────────────────────────────────────────┐
│  顶栏：波次 │ 回合 3/10 │ 净利 +12  差额 −3   ← 保持原亮度   │
│         ┌──────────────────────────────┐                     │
│         │ 这一局入场费 40 元……         │  ← 气泡（上方）     │
│         │ 你有 20 格胃、10 回合……      │                     │
│         │                  [跳过引导]  │                     │
│         └──────────────┬───────────────┘                     │
│                        │ 1 px 指引线                          │
│  ┌────────┐ ┌────────┐ │ ┌────────┐ ┌────────┐               │
│  │ 菜品卡 │ │ 菜品卡 │ ▼ │ 菜品卡 │ │ 菜品卡 │  ← 压暗      │
│  └────────┘ └────────┘   └────────┘ └────────┘               │
│  容量条 12/20 格（不压暗） │ 腻度栏（不压暗）                  │
│  详情行（压暗）                                               │
└──────────────────────────────────────────────────────────────┘
```

- 遮罩只压暗三个受保护矩形之外的区域；三个读数保持原亮度与原对比度（`ONB-R12`、ADR-0020 D5.5）。
- 气泡为米白纸面 + 墨色文字；正文 20 px（100%），行高按 HUD 主题；圆角与 1 px 边线复用 HUD 的纸面样式。
- 指引线 1 px，仅模态 beat 且气泡位于锚点上方/下方时绘制；`DOCKED` 不画。

### 5.2 气泡几何

严格复用 `ONB-F:bubble_rect` 与 ADR-0020 D5.3：

```
w = clamp(round(0.36 × W), 280, 440)
w = min(w, W − 2m)
h = lines × line_height + 2 × padding
x = clamp(anchor.center.x − w/2, m, W − m − w)
y = anchor.top − gap − h                 # 首选上方
若 y < m 或 与受保护矩形相交 → y = anchor.bottom + gap
若仍越界/相交 → y = H − m − h           # 底部停靠
若仍相交 → HIDDEN
```

| 参数 | 值 | 来源 |
|---|---|---|
| `m` | HUD `safe_margin`（32–96） | HUD，不另算 |
| `gap` | 12 px | `OnboardingConfig` |
| `padding` | 12 px | `OnboardingConfig` |
| `line_height` | HUD 主题 × 字号缩放 | HUD |
| `lines` | 1–3 | 文本表 |
| 受保护矩形 | `[turns, capacity, net]` 固定顺序 | `HudHost.get_protected_rects()` |

- 每帧展示前重新取锚点，**不缓存**（窗口缩放、150% 字号、超宽屏）。
- 所有矩形取整到像素；同一输入必须得到逐像素相同的 `Rect2`（无头断言）。
- `HIDDEN` 是最后手段：仅 < 480×320 退化视口；模态 beat 变 `HIDDEN` 立即 `FINISH(SKIPPED)` 并记 ERROR，避免「有遮罩、无说明、无法推进」；上下文提示变 `HIDDEN` 静默跳过。
- **跳过按钮行**：按钮放在气泡右下角，高 56 px（`ACC-MOT-06`），宽度按文字；气泡的**正文矩形**仍按上式，按钮行附加在正文之下、面板边界之内。总高度 = 正文 `h` + 按钮行 56 + 间距 8。这是本规格对 `bubble_rect` 的版式补充，公式冲突见 OQ-3。

### 5.3 锚点使用

| Beat | 锚点 | 用途 |
|---|---|---|
| `B0_WELCOME` | `top:net` | 决定气泡位置与指引线 |
| `B1_TWO_NUMBERS` | `card:<焦点菜>`；无焦点 `grid:first_card` | 决定位置；焦点变化时重贴 |
| `B2_CAPACITY` | `bar:capacity` | **仅标注讲解对象**；提示固定停靠（ADR-0020 D5.2/D5.4） |
| `B3_CLOYING` | `card:<该菜>` + `rail:cloy` | `card` 为讲解对象；`rail` 只用于定位腻度栏，不参与气泡位置 |
| `B4_RECEIPT` | 无 | 收据由界面流程摆放 |

### 5.4 上下文提示（B2/B3）

- 一律 `DOCKED`：`y = detail_row_top − gap − h`，再钳制到 `[m, H − m − h]`；`x` 同模态公式。
- 无遮罩、无指引线、不压暗、不改输入上下文（`ONB-R7`）。
- 不遮挡菜品卡；与详情行保持 `gap` 间距。
- `detail_row_top` 不可得（HUD 降级）时用 `H − m − 44 × scale` 保守值并记 WARNING；仍相交则静默跳过（不阻塞）。
- 锚点对象通过**文字**指明（「容量条」「右栏的腻度」），不依赖颜色；是否加静态描边见 OQ-1。

### 5.5 焦点与点击

- 本层**不设、不移、不清焦点**；B1 期间导航仍由 HUD 维护（`ONB-R10`）。
- 跳过按钮 `focus_mode = FOCUS_NONE`，不进入 HUD 的焦点顺序；键盘/手柄等价入口是 `ui_cancel`（ADR-0020 D4.3）。按钮文本进入读屏顺序（当前无读屏，保留结构）。
- 遮罩 `mouse_filter = MOUSE_FILTER_IGNORE`：不阻断 HUD 卡片的悬停与点击聚焦（ADR-0020 Verification ③）。
- 气泡与跳过按钮自身 `MOUSE_FILTER_STOP`，点击不穿透到下面的卡。

---

## 6. 状态与变体

### 6.1 视图状态（`TutorialLayer`）

| 状态 | 遮罩 | 指引线 | 气泡 | 输入 |
|---|---|---|---|---|
| `hidden` | 无 | 无 | 隐藏 | 无 |
| `modal` | 受保护区补集 | 有（可定位时） | 显示 | `Blocked` + 白名单 |
| `contextual` | 无 | 无 | 停靠 | `Run` 原样 |

### 6.2 Beat 机阶段

`IDLE → INTRO → NUMBERS → COACHING → CLOSING → CLOSED`，另有 `CANCELLED`。

| 阶段 | 气泡 | 上下文 | 推进 |
|---|---|---|---|
| `INTRO` | B0 模态 | `Blocked(B0_ALLOW)` | `ui_accept` → B1；`ui_cancel` → 跳过 |
| `NUMBERS` | B1 模态 | `Blocked(B1_ALLOW)` | `ui_accept` 且有焦点菜 → B2；无焦点不推进 |
| `COACHING` | B2/B3 上下文 | `Run` | 观察 `action_committed` / `run_ended`；不消费动作 |
| `CLOSING` | 无 | `Run` | 等待 B4 / 完成 |
| `CLOSED` | 无 | `Run` | 已广播完成 |
| `CANCELLED` | 无 | — | 放弃/故障/跳过 |

### 6.3 转移（含暂停与降级）

| 从 | 事件 | 到 | 副作用 |
|---|---|---|---|
| `INTRO` | `ui_accept` | `NUMBERS` | `pop_context()` → `push_context(Blocked, B1_ALLOW)`；B0 记已展示；B1 显示 |
| `INTRO` | `ui_cancel` | `CLOSED` | 弹 `Blocked`、隐藏、`FINISH(SKIPPED)` 一次 |
| `NUMBERS` | `ui_accept` 且 `focused_dish_id ≠ ⊥` | `COACHING` | 弹 B1 的 `Blocked`、显示 B2（同帧 HUD 看不到 accept）；B1/B2 记已展示 |
| `NUMBERS` | `ui_accept` 且无焦点 | `NUMBERS` | WARN(NO_FOCUS)；不推进、不吃菜、无计时 |
| `COACHING` | 焦点落在组计数 ≥ 1 的菜 | `COACHING` | 显示 B3（替换 B2）、记已展示 |
| `COACHING` | `action_committed` / `run_ended` | 隐藏提示 | 不重复展示 |
| 任意活动阶段 | `ui_pause` | 保持阶段，视图挂起 | `SceneFlow` 清栈；本层 `_blocked_owned = false`；气泡保持可见于暂停层之下 |
| `Paused → Run` | 恢复 | 保持阶段 | 若在 `INTRO`/`NUMBERS`，用**同一白名单**重压 `Blocked` |
| 重压失败 | `CONTEXT_RESTORE_FAILED` | `COACHING`（降级） | 当前 beat 改为无遮罩底部停靠；玩家可正常操作；不重试、不卡住 |
| 任意活动阶段 | `run_discarded` / `run_faulted` | `CANCELLED` | 若持有则弹 `Blocked`，隐藏全部，不产出 B4、不发完成 |
| `run_settled` | 有 `RunResult` | `CLOSING` | 深拷贝 result 与 menu；不读钱包、不构建讲解 |
| `Run → Menu` 第 5 步 | 结算已提交 | `CLOSED` | 构建 `FirstRunExplanation`、发 `explanation_ready`、发 `ALL_BEATS` 一次 |

**同一帧最多推进一个 beat**；`input_seq` 相同则直接返回（ADR-0020 D4.2）。任何 beat 都没有超时。

---

## 7. 交互映射

### 7.1 每个 beat 的动作表

| Beat | `ui_accept` | `ui_cancel` | 导航 | `ui_pause` | 鼠标 |
|---|---|---|---|---|---|
| B0 | 推进到 B1 | 跳过全部 | 白名单不含导航，焦点不动 | 暂停 | 点「跳过引导」= 跳过 |
| B1 | 有焦点菜才推进；不提交吃菜 | 跳过全部 | 四方向交给 HUD，玩家自己选菜 | 暂停 | 点「跳过引导」= 跳过 |
| B2 | 交给 HUD（吃菜） | `Run` 无消费者 | HUD 正常 | 暂停 | 无跳过按钮 |
| B3 | 交给 HUD（吃菜） | `Run` 无消费者 | HUD 正常 | 暂停 | 无跳过按钮 |
| B4 | 收据滚动跳过（归收据） | 收据返回菜单（归收据） | 收据 | 收据在 `Menu`，不可暂停 | 收据 |

- `ui_accept` 在 B0/B1 只能推进气泡，**不吃菜**：`Blocked` 白名单把动作交给引导，HUD 同帧读不到（`ONB-R6`、ADR-0020 D4.2）。
- B1 的 `ui_accept` 在无焦点时是**无动作**：不推进、不选菜、不报错弹窗；只记 WARN。
- 上下文提示不消费任何动作；玩家可以完全忽略它们继续吃菜（`ONB-R7`）。

### 7.2 `ui_cancel` 落点矩阵（精确）

| 前置 | 按键 | 经过的动作 | 结果 |
|---|---|---|---|
| B0/B1，`Blocked` | Esc | `ui_cancel`（`Blocked` 中 Esc 只翻译为 `ui_cancel`，ADR-0005 D3） | **跳过全部引导** |
| B0/B1，`Blocked` | Backspace | `ui_cancel` | 跳过全部引导 |
| B0/B1，`Blocked` | 手柄 B | `ui_cancel` | 跳过全部引导 |
| B0/B1，`Blocked` | P / 手柄 Start | `ui_pause` | 暂停；`Blocked` 被清空；恢复后重压 |
| B2/B3，`Run` | Esc | `ui_pause` | 暂停（不是跳过；上下文提示没有跳过入口） |
| B2/B3，`Run` | Backspace | `ui_cancel` | 无消费者，帧末作废 |
| 暂停层中 | Esc | `ui_pause` | 恢复 `Run`；引导重压 |
| `CLOSING` / `CLOSED` | `ui_cancel` | — | 本层不再消费 |

> **Esc 的双重语义在这里分叉**：模态 beat 中 Esc = 跳过，上下文提示中 Esc = 暂停，暂停层中 Esc = 恢复。三处都符合各自上下文，但对玩家是同一颗键的三种结果；见 OQ-2。

---

## 8. 数据与输入契约

| 值 | 来源 | 口径 |
|---|---|---|
| 是否首局 | `RunConfig.is_first_run` | 由 `FirstRunPlanner` 唯一产出（`runs_settled == 0 ∧ first_run_eligible`）；本层不重算 |
| 当前阶段 | `run_state` | 只读；气泡期间仍是 `Active` |
| 焦点菜 | `HudHost.focused_dish_id()` | 键盘焦点优先、悬停兜底；本层不移动焦点 |
| B1 锚点 | `card:<焦点菜>`；无焦点 `grid:first_card` | 每次展示重新取 |
| 受保护矩形 | `get_protected_rects()` → `[turns, capacity, net]` | 只做相交判定；取不到按不相交并记断言 |
| 布局度量 | `HudLayoutMetrics`（`safe_margin` / `line_height` / `font_px` / `detail_row_top`） | 150% 由 HUD 缩放后提供 |
| B2 波次/菜品数 | `MenuWaves.get_run_menu()`、`wave_count` / `wave_index` | 实际值，不写死「两波」 |
| B3 组计数 | `CloyingAndTaste.group_counts()` | 只读；焦点变化时才读 |
| B0 入场费 | `RunConfig.entry_fee`（教学价 40） | 与主菜单同源；缺覆盖时用档口价并省略「教学价」句 |
| B4 数据 | `RunResult`、`DishValues`、钱包、升级目录 | 结算提交后再读；缺值降级 |

### 8.1 输入上下文

- 模态 beat：`push_context(Blocked, allow)`；B0 `allow = [ui_accept, ui_cancel]`；B1 追加四方向。`ui_pause` 恒允许，不写在 `allow` 里（`INP-R12`）。
- 每帧 `consume_action(ui_cancel)` 再 `consume_action(ui_accept)`；即使只处理 cancel 也要取走 accept，保证 HUD 同帧读不到（ADR-0020 D4.2）。
- `COACHING` 不调用 `consume_action`（在 `Run` 中调用会返回 `false` 并记 WARNING）。
- 重压：`pop` 后 `push` 产生两次 `context_changed`；同值压栈也会加深，不能把「已是 Blocked」当成刷新（ADR-0005 D2）。暂停时 `request_context(Paused, clear_stack := true)` 清空引导层，本层置 `_blocked_owned = false`，不自行弹栈（`ONB-R17`、ADR-0020 D4.3）。

---

## 9. 文案与 B4 讲解

### 9.1 文案约束

| 约束 | 值 | 依据 |
|---|---|---|
| 单条上限 | 48 个 CJK 字符 @150% 字号（3 行 × 440 px / 27 px） | `ONB-F:bubble_text_cap`、ADR-0020 D6.2 |
| 行数 | 1–3 行 | 同上 |
| 正文字号 | 20 px（100%）起，随 `text_scale_percent` 到 25 / 30 px | `ACC-VIS-02`；GDD 的 16–27 px 范围内 |
| 占位符 | 封闭集合 `{entry_fee}` / `{capacity}` / `{turns}` / `{dish_count}` / `{wave_count}` / `{visible_count}` / `{group_name}` / `{extra}` 等 | ADR-0020 D6.2 |
| 禁止 | 模板内数字字面量；`元/格`、`每格`、`unit_value`、任何比值（局内） | `ONB-R13`、`ONB-R20` |
| 禁用措辞 | 最亏、白吃、浪费、下次先拿 | `ONB` 替换建议、ADR-0020 D6.2 |

### 9.2 B4 `FirstRunExplanation`

- 构建时点：`run_settled` 只深拷贝；`Run → Menu` 第 5 步、结算事务提交之后、收据首次绘制之前构建并交付（ADR-0020 D7.1）。
- 内容：两段短文案、按吃下顺序的逐行单位价值、一次替换建议（五种分支）、一条首笔升级建议、教学价说明（`fee_is_override` 时）。
- 收据屏只按字段排版，不重算；本层不拥有收据布局。
- 没有收据（放弃/故障/`init_failed`）时不构建、不补弹、不发完成信号。
- 不含最优解、替换后总分、推荐顺序；只做每格比较与一处对照。

---

## 10. 事件

| 事件 | 何时 | 载荷 |
|---|---|---|
| `onboarding_beat_shown(beat_id)` | 气泡首次显示 | beat id（音效可选订阅） |
| `onboarding_finished(reason)` | 完成或跳过，恰好一次 | `ALL_BEATS` / `SKIPPED` |
| `explanation_ready(run_id, explanation)` | B4 构建完成 | `FirstRunExplanation` |
| `ui_button_activated("onboarding.skip")` | 点击跳过 | 走与 `ui_cancel` 相同的 `_skip()` |

---

## 11. 转场与动画

淡入 ≤ 100 ms、淡出 ≤ 80 ms；`reduce_motion` 时均为 0（ADR-0020 D6.1、`ACC-VIS-12`）。只改变 `modulate.a`，不位移、不缩放、不模糊；`Tween` 不参与状态判定、不延迟推进，在 `INHERIT` 下随暂停停在最后一帧。禁止打字机、逐字动画、自动滚动、自动关闭。B2→B3 替换不重建节点，只换文本、行数与锚点；启动/恢复时若气泡已存在，不重播淡入。

---

## 12. 模式应用

| 模式 | 本层应用 | 例外 |
|---|---|---|
| **PAT-11 锚点高亮与受保护区域** | 模态 beat 用锚点定位 + 1 px 指引线；遮罩取三个受保护矩形的补集，读数保持原亮度 | B2/B3 固定停靠、不画线（ADR-0020 D5.4）；只有文字指代锚点，见 OQ-1 |
| **PAT-02 焦点陷阱与焦点归还** | 模态 beat 用 `Blocked` 困住**输入**，不是困住焦点；B1 导航仍归 HUD；跳过按钮不获焦 | 与 PAT-02 的「焦点陷阱」只共享输入阻断，不共享焦点移动；见 OQ-3 |
| **PAT-08 非模态提示** | B2/B3 是标准非模态提示：不挡动作、不改上下文、可忽略 | 无自动消失、无计时；由 `action_committed` / `run_ended` 收起 |
| **PAT-12 只读快照与口径所有权** | 所有数字来自 HUD/上游读数；不重算、不缓存第二份；局内不出现单位价值 | B4 的单位价值来自 `RunResult` + `DishValues`，不在局内展示 |
| **PAT-13 屏幕根替换与返回决策** | 不适用：本层不是屏幕根；B4 交给收据 | 跳过不经过 `back_target`，由 `ONB-R16` 直接定义 |

---

## 13. 无障碍

| 要求 | 落点 |
|---|---|
| `ACC-VIS-02` | 气泡正文 ≥ 20 px @720p；跳过按钮 ≥ 20 px |
| `ACC-VIS-03` | 米白气泡 + 墨色文字 ≥ 4.5:1；遮罩不降低读数对比度 |
| `ACC-VIS-05` / `07` | 跳过按钮有可见边界；不新增焦点环；HUD 的焦点环不被气泡遮挡 |
| `ACC-VIS-08` | 灰度下能读出「跳过引导」、气泡层级与三个受保护读数；锚点不靠颜色 |
| `ACC-VIS-09` / `10` | 150% 下 48 字、3 行、不截断；4K 气泡宽度封顶 440；超宽屏重新停靠 |
| `ACC-VIS-12` | 淡入/淡出为 0；无位移/缩放 |
| `ACC-MOT-01` / `06` | 纯键盘/手柄可完成 B0→B1→选菜→结算；跳过用 `ui_cancel`；跳过按钮 ≥ 56 px |
| `ACC-COG-02` | 无任何计时器、无自动推进 |
| `ACC-COG-05` | 教学不可回看是已知 MVP 缺口（`ONB-R2`）；本规格不补 Help 屏 |
| `ACC-AUD-02` / `06` | 无音频专属信息；静音可完成引导 |
| `ACC-VIS-15` | 引导期间 HUD 的「不可吃」仍可获焦、有原因；本层不把 HUD 控件变成流程禁用 |

**额外**：遮罩是 `MOUSE_FILTER_IGNORE`，鼠标仍能悬停菜品卡并触发 B3；这既是功能要求，也是「鼠标用户不被锁死」的无障碍要求。

---

## 14. 本地化

| Key | 中文（基准） | 约束 |
|---|---|---|
| `onb.b0.full` | 这一局入场费 40 元，回本线就是它。你有 20 格胃、10 回合，先看每道菜的价格和格子数。 | 41 字；≤ 48 |
| `onb.b0.no_fee_override` | 这一局入场费 {entry_fee} 元，回本线就是它。…… | 省略「教学价」句 |
| `onb.b1.full` | 每道菜有两个数：价格（元）和基础饱腹（格）。先看这两个，再决定吃哪道。 | ≤ 48 |
| `onb.b1.no_focus` | 先用方向键或鼠标选一道菜，再按确认。 | 无焦点时使用 |
| `onb.b2.full` | 容量是硬上限，这一桌装不下全部；菜会一波一波上，跳过就不会回来。 | 波数用 `{wave_count}`，不写死 |
| `onb.b3.full` | 同组第二道会多占一格；带「酸」或「冰」的菜能解掉一格。 | 组名用 `{group_name}` |
| `onb.skip.label` | 跳过引导 | 按钮，≤ 6 字 |
| `onb.a11y.hint` | 按 Esc 或 B 跳过引导。 | 可选辅助提示；不占焦点 |
| `onb.b4.*` | 见 `FirstRunExplanation` 五分支 | 只出现在收据 |

- 本地化只替换表内容，不改结构；扩写超过 48 字用 `short` 版本，仍超则 lint 失败，运行时不截断、不缩到 16 px 以下。
- 数字一律占位符；模板内不得出现 `0-9` 字面量。

---

## 15. 验收标准

- [ ] **ONB-L-AC01**（触发）GIVEN `RunConfig.is_first_run = true`，WHEN `run_started`，THEN `TutorialLayer` 可见、`push_context(Blocked)` 一层、B0 显示；非首局时本层完全隐藏且不压栈。
- [ ] **ONB-L-AC02**（B0 布局）GIVEN B0 显示，WHEN 检查矩形，THEN 气泡位于 `top:net` 上方/下方/停靠之一，且不与回合、容量、净利三个矩形相交。
- [ ] **ONB-L-AC03**（遮罩）GIVEN B0/B1，WHEN 截图，THEN 三个受保护区域像素亮度与无遮罩时一致；遮罩 `mouse_filter = IGNORE`，悬停菜品卡仍能改变焦点。
- [ ] **ONB-L-AC04**（B0 推进）GIVEN B0，WHEN 按 `ui_accept`，THEN 只进入 B1；`pop` 后 `push` 的 `Blocked` 白名单为 B1；同帧 `RunFlow.submit_action` 调用 0 次。
- [ ] **ONB-L-AC05**（B1 无焦点）GIVEN B1 且 `focused_dish_id = ⊥`，WHEN 按 `ui_accept`，THEN 不推进、不吃菜、只记 WARN(NO_FOCUS)。
- [ ] **ONB-L-AC06**（B1 有焦点）GIVEN B1 且有焦点菜，WHEN 按 `ui_accept`，THEN 进入 COACHING、显示 B2、B1 记已展示；HUD 同帧读不到 `ui_accept`。
- [ ] **ONB-L-AC07**（同帧一 beat）GIVEN 同一帧两个推进事件，WHEN 处理，THEN 最多推进一个 beat；`input_seq` 相同不重复推进。
- [ ] **ONB-L-AC08**（B2 非模态）GIVEN B2 显示，WHEN 检查上下文与位置，THEN 上下文仍为 `Run`、气泡停靠在详情行上方、无遮罩无指引线、不挡任何动作。
- [ ] **ONB-L-AC09**（B3 触发）GIVEN 焦点移到组计数 ≥ 1 的菜，WHEN 处理焦点变化，THEN 显示 B3 并替换 B2（不并排、不重建节点），B3 记已展示。
- [ ] **ONB-L-AC10**（跳过）GIVEN B0/B1，WHEN 按 Esc/Backspace/B 或点「跳过引导」，THEN 弹 `Blocked`、隐藏全部、`onboarding_finished(SKIPPED)` 恰好一次、不产出 B4、不二次确认。
- [ ] **ONB-L-AC11**（暂停）GIVEN B0/B1，WHEN 按 P/Start，THEN 进入 `Paused`、`Blocked` 被清空、气泡仍可见但不可交互；恢复后重压同一白名单。
- [ ] **ONB-L-AC12**（重压失败）GIVEN 恢复时 `push_context` 返回 `false`，WHEN 处理，THEN 当前 beat 降级为无遮罩底部停靠提示，玩家可正常吃菜，不出现「气泡在、按键无效」。
- [ ] **ONB-L-AC13**（放弃/故障）GIVEN 引导中放弃或 `run_faulted`，WHEN 处理，THEN 隐藏全部、弹栈（若持有）、不产出 B4、不广播完成、不写存档。
- [ ] **ONB-L-AC14**（无计时）GIVEN 任意 beat，WHEN 等待 60 s，THEN 不推进、不关闭、不淡出。
- [ ] **ONB-L-AC15**（几何稳定）GIVEN 100/125/150% 字号与 1280×720 / 1920×1080 / 4K / 21:9，WHEN 显示各 beat，THEN 气泡不越界、不压受保护矩形；4K 宽度 ≤ 440。
- [ ] **ONB-L-AC16**（锚点缺失）GIVEN HUD 锚点连续 2 帧为空，WHEN 显示模态 beat，THEN 气泡底部停靠；上下文提示静默跳过，不阻塞。
- [ ] **ONB-L-AC17**（文案）GIVEN 150% 字号，WHEN 渲染全部文本，THEN 每条 ≤ 48 字、≤ 3 行、无截断；禁用措辞与 `元/格` 在 lint 中失败。
- [ ] **ONB-L-AC18**（B4 时点）GIVEN 一局结算，WHEN `Run → Menu` 第 5 步完成，THEN 讲解在结算提交后构建、收据首次绘制前交付；`ALL_BEATS` 恰好一次。
- [ ] **ONB-L-AC19**（B4 缺失）GIVEN 放弃/故障导致没有收据，WHEN 离开 `Run`，THEN 不构建讲解、不发 `explanation_ready`、不发完成。
- [ ] **ONB-L-AC20**（无障碍）GIVEN 灰度截图 + 仅键盘/手柄 + 静音，WHEN 完成 B0→B1→B2/B3→收据，THEN 全程可操作、气泡可读、跳过可达、无信息只靠颜色。
- [ ] **ONB-L-AC21**（性能）GIVEN 60 fps，WHEN 引导运行，THEN 空闲 ≤ 0.05 ms/帧、模态 beat ≤ 0.1 ms/帧；不触发 HUD 重新布局。
- [ ] **ONB-L-AC22**（图层）GIVEN 引导显示中打开暂停/收据/模态，WHEN 检查层序，THEN 它们覆盖引导（10 < 15 < 20/25/30），关闭后引导恢复原状态。

---

## 16. 开放问题

| # | 问题 | 双方依据 | 当前处理 |
|---|---|---|---|
| 1 | B2/B3 的锚点高亮：PAT-11 要求锚点高亮 + 受保护区域，ADR-0020 D5.4 却规定上下文提示固定停靠、不画指引线、不使用 Scrim。结果是没有视觉线索指向容量条/腻度栏，只能靠文字。 | `PAT-11` vs `ADR-0020` D5.2/D5.4 | 本规格用文字指代锚点，暂不加描边；是否允许 2 px 静态描边（不连线、不压暗）需 `art-bible` 与用户测试 |
| 2 | Esc 在模态 beat 是**跳过**（`Blocked` 中 Esc 翻译为 `ui_cancel`），在上下文提示与暂停层是**暂停/恢复**。同一颗键三种结果，误按会永久跳过教学（本局）。 | `ADR-0005` D3/AC8 vs `ONB-R16` / `INP-R2` | 本规格按 ADR 实现；是否把跳过改为「长按/二次确认」与「跳过不二次确认」冲突，需用户测试 |
| 3 | 跳过按钮的几何与可达性：GDD 要求「键盘可达」，ADR-0020 又定 `focus_mode = NONE`、以 `ui_cancel` 等价；同时 `ACC-MOT-06` 要求按钮 ≥ 56 px，而 `bubble_rect` 的 `h` 公式没有按钮行。 | `ONB` UI Requirements vs `ADR-0020` D4.3 / `ACC-MOT-06` / `ONB-F:bubble_rect` | 本规格加 56 px 按钮行、总高 = 正文 + 按钮行；公式是否要正式扩写，见 OQ-3 |
| 4 | 正文字号：GDD 示例基准 18 px，`ACC-VIS-02` 要求正文 ≥ 20 px。本规格取 20 px（仍在 16–27 范围内），但 100% 下的字符上限从 72 变为 66。 | `ONB-F:bubble_text_cap` vs `ACC-VIS-02` | 本规格取 20 px 满足无障碍；GDD 的字号示例需回改 |
| 5 | B3 提示在焦点离开该菜后是否应立即收起？GDD 只规定「下一次 `action_committed` 或 `run_ended`」推进，提示可能指着已经不再聚焦的菜。 | `ONB-R8` / `ONB-R9` | 本规格按 GDD 保留到动作/结束；是否在焦点离开时收起，需用户测试 |
| 6 | B3 同时给 `card:<该菜>` 与 `rail:cloy` 两个锚点，ADR 规定 `rail` 只用于定位、不参与气泡位置；「定位」具体是否要画一条线/高亮？ | `ONB-R5` 表 / `ONB-R11` vs `ADR-0020` D5.2 | 本规格不画；见 OQ-1 |
| 7 | 首局判定存在两套写法：`ui-flow` 规则 8/S11 与 ADR-0020 用 `runs_settled == 0 ∧ first_run_eligible`，而 `UIF-AC6` 仍写 `runs_settled == 0`。 | `UIF-R8` / `SAV-R26` / `ADR-0020` D1 vs `UIF-AC6` | 本规格按前者；`UIF-AC6` 需回改（ADR-0020 Sync Check #2 已记录） |
| 8 | 降级后的提示在 `Run` 中「下一次 ACCEPT 推进/收起」，但该 ACCEPT 同时会到达 HUD 并提交吃菜；一次按键既吃菜又收提示，语义是否需要拆开？ | `ADR-0020` D4.3 降级路径 vs `ONB-R17` | 本规格接受「同一次确认既吃菜又收起」；若用户觉得突兀，可改为仅由 `action_committed` 收起 |
| 9 | 150% 字号 + 德语/俄语 1.4× 扩写时，48 字上限可能放不下三条语义；GDD 禁止 4 行、禁止运行时截断。 | `ONB-F:bubble_text_cap` / `ADR-0020` D6.2 vs 本地化扩写 | 本规格要求 `short` 版本与生成期 lint；是否允许 4 行需用户与本地化裁定 |
| 10 | 跳过后的本局是否还显示 B4 讲解？GDD 说跳过「不再显示任何提示与收据讲解」，但收据本身仍会显示单位价值列。 | `ONB-R16` vs `UIF-R11`（收据列） | 本规格按跳过不发 B4 讲解，收据列不受影响 |
| 11 | 无读屏实现（Comprehensive 不做），跳过按钮「进入读屏顺序」目前只是结构预留；是否需要额外的视觉「跳过」提示？ | `ACC` 无读屏承诺 vs `ACC-MOT-01` | 不额外加提示；键盘用户用 `ui_cancel`，鼠标用户看得到按钮 |
