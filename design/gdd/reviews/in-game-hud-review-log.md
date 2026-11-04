# 局内 HUD（In-Game HUD）— Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 7 | Recommended: 16
Summary: 设计骨架是干净的：HUD 严格是只读投影，所有玩法数值都声明为上游读数，"同一快照、同一重绘"与"动画不得推迟数字"两条契约贯穿规则、公式与验收；十个呈现公式的例子都能手算复现（36/54/96 px、4 列 216 px、10 槽 24 px 共 276 px、20 格 14 px、`fill_ratio = 0.6`、对比度 14.9:1、不可吃差额 2 格）。阻断项集中在四类：两条显示公式在文档自己声明的边界（`capacity_total = 0`、`turns_total = 0`）上除零；运行时不变式引用了 `action_seq` / `turns_used` 这两个上游并不发布的字段；`hud_ui_scale` 没有说明 `W/H` 的坐标空间，与 `canvas_items` 拉伸叠加会二次缩放；以及三处与仍在修订中的上游（#11 结算的实时净利、#7 胃容量的重置时机、#2 输入的双焦点绘制）直接冲突的接口。已按并行约束标注：其中 3 条要等对应系统修好才能关闭，不应记在 HUD 头上。
Prior verdict resolved: First review

Findings:
- [BLOCKING] Formulas（`hud_capacity_cells` `:252-253`、`hud_turn_slots` `:235-236`）/ Edge Cases（`:358`）/ AC 17（`:492`）：两个分段公式在自己声明的零值上除零。`cells = min(capacity_total, 24)` 后 `k` 除以 `cells`，`capacity_total = 0` 时 `cells = 0`；`slots = min(turns_total, 12)` 后 `k` 除以 `slots`，`turns_total = 0` 时 `slots = 0`。而 Edge `:358` 明确 `capacity_total = 0` 是可达错误态并写"不做除法"，AC 17 又要求 `turns_total = 0 → 剩余 0 / 0`。照公式实现会崩在文档自己承认的输入上。
- [BLOCKING] Formulas（运行时不变式 `:328`、`:331`）/ Dependencies（`:384`）/ Cross-References（`:450`）/ AC 5（`:474`）、AC 8（`:477`）：不变式断言了上游不发布的字段。`action_seq` 全仓库只在本文件 `:331` 出现一次，`run-flow.md:137-144` 的只读快照表没有它，`action_committed(action, payload)` 的 payload 也未定义；`turns_used` 同样不在该快照里，`time-and-turns.md:73` 只承诺给 HUD `turns_remaining` / `turns_total`。AC 5 与 AC 8 因此无法按字面验证。
- [BLOCKING] 布局与信息层级（`:75`）/ Formulas（`hud_ui_scale` `:205`、`:215`）/ AC 10（`:482`）：`hud_ui_scale` 没有说明 `W/H` 是窗口像素还是拉伸后的逻辑尺寸。本项目采用 `canvas_items` + `expand`（`docs/engine-reference/godot/breaking-changes.md:39`，Godot 4.7 新工程默认值），画布内容已被拉伸缩放：若 `W/H` 取窗口像素，1080p 下 `s = 1.5` 会和拉伸的 1.5 叠加成 2.25×；若取逻辑尺寸，16:9 下 `s ≡ 1`，`:215` 与 AC 10 断言的 1.5 就不成立。两种读法给出肉眼可辨的不同结果。
- [BLOCKING · 依赖 #11 结算（Needs Revision）] Interactions（`:170`）/ Cross-References（`:455`）/ AC 2（`:471`）：局内净利的实时读数没有可实现的契约。HUD 要在 `run_started` 当帧显示 `−85`，但结算 GDD 当前不持有任何局内状态——其评审已记录"无 run-start 初始化、无累加器归属、无放弃局清理"（`settlement-review-log.md` 2026-11-04 blocking 3），`running_net(k)` 的 `k` 也只有结算侧知道。HUD 不能绕过（规则 6 `:39` 禁止自行求和）。本条在 #11 落地 `begin_run(entry_fee)` + 单一 int64 累加器后才能关闭，不应由 HUD 单方面修复。
- [BLOCKING · 依赖 #7 胃容量（Needs Revision）] States（`:148`）/ Interactions（`:166`、`:385`）：Ending 帧的"数值冻结"与容量的重置时机冲突。HUD 要求 `Ending` 期间数值冻结，却又让 HUD 直接读 `capacity_remaining` 与 `can_fit`（不经 `HudFrame` 快照）。胃容量目前在 `run_ended` 上立即重置，而 `run_ended` 正是 HUD 构造 Ending 帧的同一信号——先重置还是先构帧，决定玩家看到的是终局容量还是新局的 0。修法在 #7（改为 `run_settled` / `run_discarded` 重置），但 HUD 也应写明 Ending 期间读最后一帧快照而非实时查询。
- [BLOCKING · 依赖 #2 输入（In Review）] Game Feel（`:425`）/ AC 37（`:521`）对照 `input.md:145`：双焦点的绘制规则两边相反。输入写"以最近一次输入的方式决定当前绘制哪种焦点样式"，HUD 要求两种焦点同时绘制、同时可辨。按输入 GDD 实现只能画一种，AC 37 无法通过。
- [BLOCKING] Interactions（`:176`）/ Dependencies（`:395`）：音效反馈要求 HUD 发出的三个事件在本 GDD 中不存在。`audio-feedback.md:109,359,384` 写明 HUD "必须发出" `dish_action_rejected(reason)`、`number_roll_started(field, from, to, duration_ms)`、`number_roll_finished(field, skipped)`，并称这是硬性接口要求；HUD 只写了"订阅 HUD 的焦点移动、确认、拒绝、跳过意图"，全文没有这三个事件名，而规则 4 的数字动画与 AC 23 的拒绝反馈正是它们的来源。
- [RECOMMENDED] Interactions（`:166`）/ Formulas（`:300`、`:330`）/ Dependencies（`:385`）：上游调用签名少了已发布的参数。`can_fit(d)` 应为 `can_fit(e)`（`stomach-capacity.md:31,:141`；`entities.yaml:897`），`get_dish(id)` 应为 `get_dish(id, caller)`（`dish-values.md:94`）。卡片合成（`:43`）需要写明"HUD 先取 `effective_satiety(d)`，再把它交给 `can_fit`"这一步。
- [RECOMMENDED] Formulas（`hud_turn_slots` `:235`、`:248`）/ Edge Cases（`:356`）/ AC 14（`:486`）：公式与自己的注释矛盾。`slots = min(turns_total, 12)` 在 `turns_total = 20` 时得 12，但注释与 AC 14 都说 `> 12` 时完全不画槽位。二选一：画 12 格 + 比例条（此时 `slots` 公式才成立），或彻底不画（则删掉 `slots`，只留比例条）。
- [RECOMMENDED] Formulas（`:242`）/ AC 17（`:492`）对照 `time-and-turns.md:98`：`turns_total` 的范围写成 1–999，上游是 1–INT32_MAX，AC 17 又测 0。取一个一致口径（建议 0–INT32_MAX，0 只作防御分支）。
- [RECOMMENDED] 信息清单（`:125`）：下一波预览的更新时机写成"开局锁定"，但 `preview_dishes` 依赖 `wave_index`（`menu-waves.md:103`，最后一波为 0），波次推进后必须刷新。规则 3 的快照重建已覆盖，改这一行即可。
- [RECOMMENDED] Core Rules（`:49`，规则 11）：句子自相矛盾——`can_fit = true` 但剩余容量显示不足时"信任 `can_fit`，不提交动作"。信任可吃就该提交；AC 19 实际定的是"不提交"。改为"显示以 `can_fit` 为准；提交前再查 `capacity_remaining`，两者矛盾则不提交并记 `HUD_INVARIANT` 4"。
- [RECOMMENDED] Core Rules（`:33`，规则 3）/ 信息清单（`:127`）/ AC 35–38：重绘触发清单只有上游信号，但焦点与悬停变化（信息清单写明更新时机是"焦点变化"）不产生任何上游信号。补一句：焦点变化时用缓存的 `HudFrame` 重建，不重新读取上游。
- [RECOMMENDED] Formulas（`hud_area_usage` `:284-296`）/ AC 12（`:484`）：右栏高度没有来源。公式要 `H_rail`，分区表（`:98-103`）只给了右栏的 x 与宽度；例子用 560，而按图（顶栏下缘 108、内容下缘 684）应是 576，比值 26.9% 而非 26.5%。两者都 ≤ 28%，但 AC 12 断言的"实测 26.5%"无法从文档复现。给分区表补 y 范围。
- [RECOMMENDED] Interactions（`:170`）/ 信息清单（`:117-118`）：`running_net(k)` 的 `k` 没写。写明 `k = eaten_count`（快照 `:164` 已有该字段），或说明 provider 暴露无参读数——这是 #11 修复时唯一需要 HUD 侧确定的调用形状。
- [RECOMMENDED] Open Questions（`:532-541`）：三条已作答。OQ1 的前提不成立——`input.md:35` 定义的是 `game_skip_wave`，`:151` 明确是"跳过当前波 / 最后一波结束取餐"，`:211` 断言 InputMap 恰有 8 个动作、没有 `skip_dish`；OQ6 已被规则 14 与 AC 37 答了一半，只剩"鼠标点击是否直接提交"；OQ8 由 `first-run-onboarding.md:66` 回答（`get_anchor_rect(anchor_id) -> Rect2`，六个 id），该文并明说"这回答了局内 HUD Open Question 8"。
- [RECOMMENDED] Interactions（`:175`）/ Dependencies（`:394`）：首局引导需要的 `get_anchor_rect(anchor_id)` 没有出现在本 GDD 的接口表里，而对方把它列为硬上游契约（`first-run-onboarding.md:124,359`）。补这一行，同时关闭 OQ8。
- [RECOMMENDED] Dependencies（`:397-402`）："双向一致性（待编排会话处理）"整段已过期。`systems-index.md:71` 的 #15 行已经列出 `腻度与口味` 与 `菜单波次`；八份上游 GDD 现在都回指本文件（`run-flow` 14 次、`time-and-turns` 15 次、`stomach-capacity` 10 次、`dish-values` 5 次、`cloying-and-taste` 5 次、`menu-waves` 5 次、`settlement` 6 次、`input` 3 次）。删掉或降级为"已完成"。
- [RECOMMENDED] States（`:147`）：`frozen` 只在 `Active` 下建模，但 `run-flow.md:89` 说 `frozen` 与 `Active` / `Ending` / `Settling` / `Handoff` 正交。补一行：暂停在 `Ending` 及之后到达时 HUD 保持原状态（本来就不可交互），恢复后不重播。
- [RECOMMENDED] Edge Cases（`:360`）：空局文案"净利显示负数或 0（由结算给值）"含糊。结算保证空局 `net_profit = −entry_fee`（`settlement-and-entry-fee.md:297`），直接写这个值。
- [RECOMMENDED] Dependencies（`:392`、`:458`）/ Cross-References：`scene-flow.md` 被声明为间接上游，但该文件 0 次提到局内 HUD（`grep -c "局内 HUD"` = 0）。补一条间接回指，或说明它只经 run-flow 的 `frozen` 到达。路由到 `/consistency-check`。
- [RECOMMENDED] 注册表（`design/registry/entities.yaml`）：`referenced_by` 未同步。全表只有 `onboarding_active`（`:1095-1101`）列了 `design/gdd/in-game-hud.md`；`running_net`（`:666`）、`shortfall`（`:681`）、`capacity_used`（`:696`）、`can_fit`（`:897`）、`effective_satiety`（`:496`）、`preview_dishes`（`:553`）都缺。属编排会话的写入范围，本次不改。
- [RECOMMENDED] Cross-document：`capacity_used` 的归属三方不一致——HUD（`:166`、`:261`）、`run-flow.md:141`、`stomach-capacity.md:38` 都说它来自胃容量，注册表 `entities.yaml:696` 把 `source` 写成 settlement。路由到 `/consistency-check`。

Advisory（不计入）："中央 40% 不得放 HUD"这条判据（`:101`、AC 16 `:488`）在 `design/` 里没有出处，规则文本出自 `.claude/docs/templates/hud-design.md:95`——补一句引用即可。`hud_number_format` 写"HUD 金额最多 5 位"（`:63`），而注册表的 `running_net` 上界是 `999999`（6 位，`:675`），确认一下取整策略。

---

## Design Review: 局内 HUD（In-Game HUD）
Specialists consulted: none（lean 模式，Phase 3b 未运行）
Re-review: No — 首次评审

### Completeness: 6/6 required sections present

`modes.workflow: standard`。必需：Overview、Detailed Rules、Formulas、
Edge Cases、Dependencies、Acceptance Criteria。六项全部存在；`## Detailed
Design` 按规则视为 Detailed Rules。Formulas 在本系统是必需的：它定义了
安全边距、缩放、列数、槽位、容量格、对比度、面积预算与不可吃差额八组
数值规则。

Advisory at `standard`：
- **Player Fantasy** —— 无独立章节；`## Game Feel`（`:418-425`）覆盖了
  意图中的感受（即时、可预期、不催促、拒绝看得见、收尾要稳），补一行
  交叉引用即可。
- **Tuning Knobs** —— 无独立章节，但 Formulas 末尾的「调参起点」
  （`:333-351`）以表格形式给出了 15 个参数、起点值、安全范围与增减效果，
  实质上满足了要求，只是没按章节标题组织。

### Dependency Graph

Dependencies 表 13 行、Interactions 表 15 行所声明的 GDD 全部存在。

上游（硬）：
- ✓ run-flow.md —— 存在（Approved）
- ✓ time-and-turns.md —— 存在（Approved）
- ✓ stomach-capacity.md —— 存在（Needs Revision）
- ✓ dish-values.md —— 存在（Approved）
- ✓ cloying-and-taste.md —— 存在（Needs Revision）
- ✓ menu-waves.md —— 存在（Needs Revision）
- ✓ settlement-and-entry-fee.md —— 存在（Needs Revision）
- ✓ input.md —— 存在（In Review）

间接 / 同级 / 下游：
- ✓ data-config.md —— 存在
- ✓ scene-flow.md —— 存在，但 0 次回指本文件（见 Recommended）
- ✓ ui-flow.md —— 存在，13 次提到局内 HUD
- ✓ first-run-onboarding.md —— 存在，9 次提到局内 HUD
- ✓ audio-feedback.md —— 存在，6 次提到局内 HUD

UX 依赖（非 GDD，本 GDD 已明说未创建）：
- ✗ `design/ux/hud.md` —— 不存在（UX Flag `:429` 已声明，属 `/ux-design`）
- ✗ `design/accessibility-requirements.md` —— 不存在（Cross-References `:462` 已声明）

### Registry Values

按 Phase 3 的要求，用注册表里的词条名对全文做了一次整篇搜索
（`running_net` / `shortfall` / `capacity_used` / `capacity_total` /
`can_fit` / `capacity_remaining` / `effective_satiety` / `extra_satiety` /
`cleanse_target` / `taste_tags` / `cloying_group` / `preview_dishes` /
`visible_dishes` / `unit_value` / `turn_budget` / `turns_total` /
`entry_fee` / `opt_value`，大小写不敏感、`_` 写作 `[_ ]`）——68 行命中。
无数值冲突：

- `station_canteen_entry_fee = 85`（`entities.yaml:1183-1193`）与 AC 2
  （`:471`）的 `−85` / `还差 85 元`一致，也与 AC 1（`:470`）的
  16 + 5 + 18 − 85 = `−46` 一致。
- `capacity_total` 20、`capacity_used` 12 与公式例子（`:267`）、AC 15
  （`:487`）一致；`can_fit` 的等号边界（`entities.yaml:897`）与 HUD 的
  「需要 6 格，剩 4 格」写法（AC 18 `:493`）一致。
- `unit_value` 在 HUD 里只以否定形式出现（`:45`、`:63`、`:131`、
  `:167`、`:386`、`:452`、AC 20 `:495`），与「不在 HUD」一致。
- `turns_total = 10`、`turn_budget = 10` 与 AC 14 的 10 槽 276 px 一致。
- 无 HUD 拥有的注册表词条，与「HUD 只持有呈现层内部规则」的声明一致。

冲突或缺口：
- **`referenced_by` 未同步**：八个 HUD 读取的词条没有列本文件
  （见 Recommended）。
- **`capacity_used` 归属**：注册表写 settlement，三份 GDD 写胃容量
  （见 Recommended）。

### Required Before Implementation

1. **给两条分段公式补零值分支** —— `hud_capacity_cells`（`:252-253`）在
   `capacity_total = 0` 时 `cells = 0`，`k` 除零；`hud_turn_slots`
   （`:235-236`）在 `turns_total = 0` 时 `slots = 0`，同样除零。这两个值
   不是假想输入：Edge Cases `:358` 把 `capacity_total = 0` 写成可达错误态
   并明说"不做除法"，AC 17（`:492`）又要求 `turns_total = 0 → 剩余 0 / 0`。
   在两个公式里各写一条显式分支（`cells = 0` / `slots = 0` 时不画分段，
   只画文字与比例条，比例条本身也要避开 0 分母），并让 Output Range 与
   分支一致。

2. **让不变式只断言拿得到的事实** —— 不变式 5（`:331`）用 `action_seq`，
   全仓库仅此一处出现；`run-flow.md:131` 的 `action_committed(action,
   payload)` 没有定义 payload 结构，只读快照表（`run-flow.md:137-144`）
   也没有该字段。不变式 2（`:328`）与 Dependencies（`:384`）、
   Cross-References（`:450`）都把 `turns_used` 当作快照字段，但快照没有它，
   `time-and-turns.md:73` 也只承诺 `turns_remaining` / `turns_total`。
   两条路选一条：(a) 请 #5 在快照里补 `action_seq`（单调提交计数）与
   `turns_used`，HUD 侧不动；(b) 不变式 5 改为"HUD 本地帧序号 + `run_id`"，
   并从声明的接口里删掉 `turns_used` —— HUD 不自行相减（规则 1 `:29`）。
   无论选哪条，AC 5（`:474`）与 AC 8（`:477`）都要跟着改。

3. **写明 `hud_ui_scale` 的坐标空间** —— 布局段（`:75`）说采用
   `canvas_items` + 1280×720 基准 + `expand`，"UI 整体随窗口等比放大"；
   公式（`:205`）又用 `W/H`（"运行时视口"）算 `s`，例子（`:215`）把
   1920×1080 算成 1.5。Godot 4.7 新工程默认就是 `canvas_items` +
   `expand`（`docs/engine-reference/godot/breaking-changes.md:39`），
   画布内容已被拉伸：`W/H` 若取窗口像素，1080p 下就是 1.5 × 1.5 = 2.25×
   的二次放大；若取拉伸后的逻辑尺寸，16:9 下 `s` 恒为 1，例子与 AC 10
   （`:482`）的 1.5 又不成立。在公式的 Variables 表里给 `W` / `H` 指定
   坐标空间，并据此重算 `:215` 的例子与 AC 10。这条会决定所有布局判据的
   实测值，所以必须先定。

4. **[依赖 #11] 固定局内净利的调用形状** —— AC 2（`:471`）要求
   `run_started` 当帧显示 `−85`，`:170` 又规定 HUD 只读 `running_net` /
   `shortfall`、不读 `entry_fee`。结算目前没有局内状态、没有 run-start
   初始化、没有累加器归属（`settlement-review-log.md` 2026-11-04
   blocking 3），所以本条今天无法实现，也不该由 HUD 绕过（规则 6 `:39`
   禁止自行求和）。HUD 侧现在能做的只有一件事：把调用写成
   `running_net(k)` 且 `k = eaten_count`（快照 `:164` 已有该字段），
   或明确 provider 暴露无参读数。其余等 #11 落地后再复审。

5. **[依赖 #7] 说清 Ending 期间读的是哪份数** —— `:148` 要求 `Ending`
   数值冻结，`:166` / `:385` 却让 HUD 直接读 `capacity_remaining` 与
   `can_fit`。胃容量在 `run_ended` 上立即重置，而 `run_ended` 正是 HUD
   构造 Ending 帧的信号。改法在 #7（重置移到 `run_settled` /
   `run_discarded`），HUD 侧应补一句：Ending 帧读最后一帧快照，不再实时
   查询容量。

6. **[依赖 #2] 裁定双焦点的绘制规则** —— `input.md:145` 说"以最近一次
   输入的方式决定当前绘制哪种焦点样式"，HUD `:425` 与 AC 37（`:521`）
   要求两种焦点同时绘制、同时可辨。Godot 4.6 的双焦点只保证记录分离
   （`docs/engine-reference/godot/modules/ui.md:18`），绘制规则由项目定。
   输入当前 In Review，需要 #2 的修订版给出唯一答案，HUD 再据此改 Game
   Feel 与 AC 37。

7. **补三个音效事件** —— `audio-feedback.md:109,359,384` 要求 HUD 发出
   `dish_action_rejected(reason)`、`number_roll_started(field, from, to,
   duration_ms)`、`number_roll_finished(field, skipped)`。HUD 的
   Interactions（`:176`）与 Dependencies（`:395`）只有笼统的"订阅意图"。
   在接口表里列出三个事件名、触发条件与字段，并在规则 4（数字动画）与
   规则 11 / AC 23（拒绝反馈）里点名它们。这是一处表格补充，不动设计。

### Recommended Revisions

8. **上游调用签名** —— `can_fit(d)`（`:166`、`:300`、`:330`）改为
   `can_fit(e)`；`get_dish(id)`（`:167`、`:386`）改为 `get_dish(id,
   caller)`。卡片合成（`:43`）要补上"HUD 先取 `effective_satiety(d)`，
   再把结果交给 `can_fit`"这一步，否则读者会以为胃容量收菜 id。
9. **回合槽位公式与 AC 14 对齐** —— `:235` 的 `slots = min(turns_total,
   12)` 与 `:248` / `:356` / AC 14 的"`> 12` 不画槽位"二选一。
10. **`turns_total` 范围** —— `:242` 的 1–999 与 `time-and-turns.md:98`
    的 1–INT32_MAX、AC 17 的 0 统一。
11. **下一波预览的更新时机** —— `:125` 的"开局锁定"改为"开局、跳波、自动
    推进"，与 `preview_dishes` 依赖 `wave_index`（`menu-waves.md:103`）一致。
12. **规则 11 的措辞** —— `:49` 的"信任 `can_fit`，不提交动作"自相矛盾，
    按 AC 19 拆成"显示信任 `can_fit`；提交前复查 `capacity_remaining`，
    矛盾则不提交并记 invariant 4"。
13. **规则 3 的触发清单** —— `:33` 补上焦点 / 悬停变化：用缓存的快照重建
    帧，不重新读上游。
14. **`hud_area_usage` 的右栏高度** —— 在分区表（`:98-103`）补右栏 y 范围；
    例子的 560 与 AC 12 的 26.5% 需要能从分区表推出来。
15. **Open Questions 收口** —— OQ1 前提已不成立（`input.md:35,151,211`），
    OQ6 只剩"点击是否直接提交"，OQ8 已由 `first-run-onboarding.md:66` 回答。
16. **补 `get_anchor_rect(anchor_id)`** —— 首局引导把它列为硬上游契约
    （`first-run-onboarding.md:124,359`），HUD 的接口表里没有。补进
    Interactions（`:175`）与 Dependencies（`:394`），同时关闭 OQ8。
17. **删除过期的"双向一致性"段** —— `:397-402` 描述的三件事都已完成。
18. **`frozen` 在 Ending 之后的行为** —— 补一行，说明暂停在 `Ending` /
    `Settling` / `Handoff` 到达时 HUD 保持原状态。
19. **空局文案** —— `:360` 直接写 `net_profit = −entry_fee`。
20. **`scene-flow.md` 回指** —— 该文件 0 次提到本系统；补间接回指或说明
    只经 run-flow 到达。
21. **注册表 `referenced_by`** —— 八个词条缺本文件（编排会话写入范围）。
22. **`capacity_used` 归属** —— 注册表与三份 GDD 说法不同，路由到
    `/consistency-check`。

### Specialist Disagreements

无 —— Lean 模式不派生专家（Phase 3b 跳过）。

### Nice-to-Have

- 「中央 40% 不得放 HUD」（`:101`、AC 16 `:488`）在 `design/` 里没有
  出处，规则文本在 `.claude/docs/templates/hud-design.md:95`；补一句引用。
- `hud_number_format`（`:63`）的"HUD 金额最多 5 位"与注册表 `running_net`
  上界 `999999`（`entities.yaml:675`）差一位，确认取整策略。
- `hud_safe_margin` 的下限 32 px 在最低支持分辨率 1280×720 下不可达
  （`round(0.05 × 720) = 36`），可保留为防御值，但可以注明。
- `hud_contrast_ratio` 的算例精确（白字 on `#1F2A24` → `L2 ≈ 0.0207`，
  `CR ≈ 14.9:1`），可作为无障碍标准的基准行。
- Edge `:372` 描述"`turns_remaining = 0` 且餐局流程尚未切到 `Ending` 的
  那一帧"；`run-flow.md:37` 说判定与转移在同一帧完成，这个中间帧可能
  不存在。写成"若快照出现该组合"更稳。

### Scope Signal

Rough scope signal: **L**（8 个硬上游 + 5 个下游 / 同级系统，10 组呈现
公式，无新增 ADR 预期；producer 应在排期前复核）。依赖数量已接近 XL
的判据，但数值全部是呈现层约束，不引入玩法风险。

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 94da89a242967c44827406c4da5714d088df6394
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/input.md dd15ea96523a49136b96cfef827c5619dee75e6a
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/systems-index.md 182d446c25ab8aa50ccf7d672bc2132eeb2b22d3
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

---

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 21
Summary: 上一轮 7 条 blocking 全部关闭：两条分段公式用 `max(1, …)` 与 `fill_ratio` 三元分支消除了零分母；`action_seq` / `turns_used` / `running_net` / `shortfall` 已由 `run-flow.md:141-145` 的只读快照发布，不变式 2/5 与 AC 5/8 可验证；`hud_ui_scale` 明确 `W/H` 取窗口像素且 HUD 走不叠加拉伸的独立层，1080p 的 1.5 是终值；净利累加器归餐局流程（`run-flow.md:51`），HUD 只读快照；`Ending` 规定读最后一帧快照；双焦点改为同时绘制，与 `input.md:43,146` 一致；三个音效事件已写入接口表；`get_anchor_rect` 与三个受保护矩形已补。剩余 21 条均为文档同步与措辞问题，不阻断实现。
Prior verdict resolved: Yes

### Prior blocking — verified

| Prior blocking | Current state | Result |
|---|---|---|
| 两条显示公式在 0 值上除零（`hud_capacity_cells` / `hud_turn_slots`） | `:235` `slots = max(1, min(turns_total, 12))`；`:252` `cells = max(1, min(capacity_total, 24))`；`:254` `fill_ratio = capacity_total > 0 ? … : 0`；`:248` 写明 `turns_total = 0` 不渲染槽位 | ✓ |
| 不变式引用上游不发布的 `action_seq` / `turns_used` | `run-flow.md:141` 快照含 `turns_used`，`:144` 含 `action_seq`，`:145` 含 `running_net` / `shortfall`；不变式 2/5（`:328`、`:331`）与 AC 5/8 可验证 | ✓（HUD 侧字段清单待补，R2） |
| `hud_ui_scale` 坐标空间未定义 | `:211` / `:215` 明确 `W/H` 为窗口像素、HUD 独立层不叠加 `canvas_items`，1080p `s = 1.5` 为最终缩放；AC 10 一致 | ✓（布局段与 OQ7 待同步，R1） |
| [依赖 #11] 局内净利无可实现契约 | `run-flow.md:51` 规则 23 由餐局流程持有累加器（开局 `−entry_fee`、每次提交 `+price`）；快照 `:145` 发布读数；HUD `:170` 只读快照 | ✓ |
| [依赖 #7] Ending 数值冻结与容量重置冲突 | `:166` 明写「`Ending` 期间读最后一帧快照，不实时查询（容量可能在 `run_ended` 重置）」；`:148` 数值冻结 | ✓（构帧时序仍建议写明，R15） |
| [依赖 #2] 双焦点绘制规则相反 | `input.md:43` 规则 8「分开记录、分开绘制」，`:146` 「各自记录、同时绘制，最近一次输入决定主高亮但不隐藏另一种样式」；HUD `:425` 与 AC 37 已一致 | ✓ |
| 音效三个事件缺失 | `:176` 列出 `dish_action_rejected(reason)` / `number_roll_started(field, from, to, duration_ms)` / `number_roll_finished(field, skipped: bool)`，与 `audio-feedback.md:112,363` 一致 | ✓（触发条件待补，R4） |

### Completeness: 6/6 required sections present

`modes.workflow: standard`。Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria 全部存在；Formulas 为必需（八组呈现数值规则）。Advisory：无 `## Player Fantasy` 标题（`## Game Feel` `:418-425` 覆盖实质）；Tuning Knobs 以 `### 调参起点`（`:333-351`）表格形式存在。

### Dependency Graph

Dependencies 表 13 行、Interactions 表 15 行声明的 GDD 全部存在：`run-flow.md`、`time-and-turns.md`、`stomach-capacity.md`、`dish-values.md`、`cloying-and-taste.md`、`menu-waves.md`、`settlement-and-entry-fee.md`、`input.md`、`data-config.md`、`scene-flow.md`、`ui-flow.md`、`first-run-onboarding.md`、`audio-feedback.md`。UX 依赖 `design/ux/hud.md` 与 `design/accessibility-requirements.md` 不存在，文件已在 `:429`、`:462` 声明由 `/ux-design` 产出。

### Registry Values

用注册表词条名（`running_net` / `shortfall` / `capacity_used` / `capacity_total` / `can_fit` / `capacity_remaining` / `effective_satiety` / `extra_satiety` / `cleanse_target` / `preview_dishes` / `visible_dishes` / `unit_value` / `turns_total` / `entry_fee` 等）对全文做大小写不敏感搜索，68 行命中。无数值冲突：

- `station_canteen_entry_fee = 85`（`entities.yaml:1183-1193`）与 AC 2 的 `−85` / `还差 85 元`一致；AC 1 的 16 + 5 + 18 − 85 = `−46` 一致。
- `capacity_total` 20 / `capacity_used` 12 与公式例子（`:267`）、AC 15 一致；`can_fit` 的等号边界与 AC 18 的「需要 6 格，剩 4 格」一致。
- `running_net` / `shortfall` 的表达式与 HUD 的显示口径一致；免单档口 `entry_fee = 0` 推出 `shortfall = 0`，与 Edge `:359` 一致。
- `unit_value` 只以否定形式出现（`:45`、`:63`、`:131`、`:167`、`:386`、`:452`、AC 20），与「不在 HUD」一致。
- 无 HUD 拥有的注册表词条，与「HUD 只持有呈现层规则」一致。

缺口：`referenced_by` 未同步（R16）；`capacity_used` 归属仍写 settlement（R17）；金额位数 5 vs `999999`（R18）。

### Required Before Implementation

无。

### Recommended Revisions

1. **布局段与 `hud_ui_scale` 互相矛盾** —— `:75` 仍写「采用 `canvas_items` 拉伸…UI 整体随窗口等比放大」，OQ7（`:540`）也仍假设 `canvas_items`；而 `:211` / `:215` 明写 HUD 用不叠加拉伸的独立层。统一为一套说法并写明独立层的配置方式，否则实现者可能再次叠加出 2.25×。
2. **快照字段清单不完整** —— Interactions `:164` 列出的快照字段没有 `turns_used`、`action_seq`、`running_net`、`shortfall`，但 Dependencies `:384`、不变式 2/5、`:170` 都在使用它们。按 `run-flow.md:136-148` 补齐。
3. **`hud_capacity_cells` 的 Output Range 与公式不符** —— `:267` 写 `cells ∈ [0, 24]`，而 `cells = max(1, …)` 的值域是 [1, 24]；同时没有像 `hud_turn_slots`（`:248`）那样写明 `capacity_total = 0` 时「不渲染分段」。
4. **音效事件的触发条件缺失** —— `:176` 只给了签名；规则 4（`:35`）与规则 11（`:49`）没有点名事件。写明 `dish_action_rejected` 只在规则 11 / AC 23 的拒绝上发、`number_roll_*` 只在真正的滚动动画上发（并说明滚动动画与规则 4「动画不得推迟数值」如何兼容），否则 `audio-feedback.md:370` 的「只有动画滚动才发」无法落实。
5. **`can_fit(d)` 签名不符** —— `:47`、`:166`、`:300`、`:309`、`:330`、`:385` 应改为 `can_fit(e)`（`stomach-capacity.md:62`；注册表 expression 用 `e`），并补「HUD 先取 `effective_satiety(d)`，再把该整数交给 `can_fit`」。
6. **`get_dish(id)` 签名不符** —— `:167`、`:386` 应为 `get_dish(id, caller)`（`dish-values.md:94`）。
7. **`turns_total` 范围** —— `:242` 写 1–999，上游是 1–INT32_MAX，AC 17 又测 0；统一为 0–INT32_MAX（0 仅防御分支）。
8. **下一波预览更新时机** —— 信息清单 `:125` 的「开局锁定」应改为「开局、跳波、自动推进」，与 `preview_dishes` 依赖 `wave_index`（`menu-waves.md:103`）一致。
9. **规则 11 措辞自相矛盾** —— `:49` 「信任 `can_fit`，不提交动作」；按 AC 19 拆成「显示信任 `can_fit`；提交前复查剩余容量，矛盾则不提交并记 invariant 4」。
10. **规则 3 的重绘触发清单** —— `:33` 仍只有上游信号与 `frozen`，而信息清单 `:127` 写明焦点变化也要更新；补「焦点/悬停变化用缓存的 `HudFrame` 重建，不重新读上游」。
11. **`hud_area_usage` 的右栏高度无来源** —— 分区表 `:98-103` 只给右栏 x 与宽度；例子用 560，按图（顶栏下缘 108、内容下缘 684）应为 576，比值 26.9% 而非 AC 12 的 26.5%。补 y 范围，或改例子与 AC。
12. **OQ1 前提已不成立** —— `input.md:35,151` 定义的是 `game_skip_wave`（跳过当前波 / 最后一波结束取餐），`:211` 断言无 `skip_dish`；HUD 规则 16 用的也是 `game_skip_wave`。关闭 OQ1。
13. **删除过期的「双向一致性」段** —— `:397-402` 的三件事均已完成：`systems-index.md:71` 已列全 8 个依赖，八份上游 GDD 都已回指。删除或标为已完成。
14. **锚点 id 词表** —— HUD 只写 `get_anchor_rect(anchor_id)`（`:175`、`:394`），未列 id；`first-run-onboarding.md:66` 定义了六个（`top:net` / `top:turns` / `bar:capacity` / `rail:cloy` / `card:<dish_id>` / `grid:first_card`）。在 HUD 接口表列出，或明确「id 词表由首局引导定义」。
15. **Ending 帧的构帧时序** —— `:166` 说 Ending 读最后一帧快照，但规则 3（`:33`）在 `run_ended` 时仍「读取一次完整数据」，而胃容量在 `run_ended` 重置（`stomach-capacity.md:28`）。写明 Ending 帧复用 `run_ended` 之前缓存的快照，或请 #5 声明快照组装先于胃容量重置。
16. **注册表 `referenced_by`** —— `running_net`、`shortfall`、`capacity_used`、`can_fit`、`effective_satiety`、`preview_dishes` 等仍未列本文件（仅 `onboarding_active` 有）。属编排会话写入范围。
17. **`capacity_used` 归属** —— 注册表 `entities.yaml:696` 的 `source` 仍是 settlement，HUD/stomach-capacity/run-flow 都当作胃容量读数。路由 `/consistency-check`。
18. **金额位数** —— `hud_number_format`（`:63`）「HUD 金额最多 5 位」与注册表 `running_net` 上界 `999999`（6 位，`entities.yaml:666-675`）差一位；确认取整/截断策略。
19. **`scene-flow.md` 回指** —— `grep -c '局内 HUD' design/gdd/scene-flow.md` 仍为 0；补间接回指或说明只经 run-flow 的 `frozen` 到达。
20. **`frozen` 在 Ending 之后** —— States 表（`:142-151`）仍只在 `Active` 下建模 `Frozen`；`run-flow.md:90` 说 `frozen` 与四个局内状态正交。补一行：暂停在 `Ending` / `Settling` / `Handoff` 到达时保持原状态，恢复后不重播。
21. **空局文案** —— Edge `:360` 仍写「净利显示负数或 0（由结算给值）」；结算保证空局 `net_profit = −entry_fee`，直接写该值。

### Specialist Disagreements

无 —— Lean 模式不派生专家（Phase 3b 跳过）。

### Nice-to-Have

- `hud_safe_margin` 的下限 32 px 在最低支持分辨率下不可达（`round(0.05 × 720) = 36`），可标注为纯防御值。
- `effective_satiety` 的变量范围写 1–999，注册表是 [1, 9]；放宽不冲突，但可对齐。
- 「中央 40% 不得放 HUD」（`:101`、AC 16）的出处是 `.claude/docs/templates/hud-design.md:95`，可补一句引用。

### Scope Signal

Rough scope signal: **L**（8 个硬上游 + 5 个下游 / 同级系统，10 组呈现公式，无新增 ADR；producer 应在排期前复核）。

### Verdict: APPROVED

7 条 blocking 全部关闭，剩余 21 条为文档同步、签名与措辞问题，不阻断实现。建议 `systems-index.md` 第 15 行状态由 `In Review` 改为 `Approved`（本次不修改索引）。本 GDD 未被修改。

Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 72584c8a3cc37bfd9c68d285c4223ae8425a63f0
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/menu-waves.md ce532d3265df1ed7555ba5aa46383be35bbee4a1
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/systems-index.md 15e0bd3d307ed09738b74dd3c099816b497b69e8
