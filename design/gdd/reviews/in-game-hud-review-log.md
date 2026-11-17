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

---

## Revision — 2026-11-06 — GDD Revision Flags write-back

Closed the `architecture-review-2026-11-05` GDD Revision Flags for this system; this entry records a revision, not a fresh review.

- Flag #12 — rule 9 now states MVP does not implement player sorting; any future sorting is a view-only permutation and `display_index` still records the canonical `visible_dishes` position. Source: ADR-0019 #6.

Status: substantive edit — the GDD returns to `In Review` per `.claude/rules/design-docs.md`; approval requires a fresh full `/design-review`.

Reviewed-Content-Hash: design/gdd/in-game-hud.md 822d364b7350ef2449e862ba3052a3d415f4b98a

---

## Review — 2026-11-06 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean)
Blocking items: 0 | Recommended: 2
Summary: 规则 9 现在与 ADR-0019 的 GDD Sync Check 第 6 条及 `TR-in-game-hud-003`「不排序」一致：MVP 不实现玩家主动排序，未来排序只能是视图层置换且 `display_index` 仍记录 `visible_dishes` 的规范位置。信息清单、UI Requirements 组件表、公式与其余 AC 都没有残留「玩家可排序」的承诺。
Prior verdict resolved: Yes — flag 12 关闭。
Findings:
- [RECOMMENDED] Acceptance Criteria 21（`:498`）：「无默认排序控件被激活」仍允许一个「非默认」排序控件被想象出来；把新规则可测化：MVP 的 HUD 不存在任何排序控件或排序动作。
- [RECOMMENDED] 跨文档（非本文件）：`dish-values.md:312` 的 UI Requirements 仍写「可选排序：允许按价格、饱腹排序」，而 ADR-0019 D4「排序扩展」仍以「GDD 规则 9 允许玩家主动排序」为前提描述本文件；两处都需按「MVP 不实现、未来视图置换」更新，属架构/上游文档维护会话。

Reviewed-Content-Hash: design/gdd/in-game-hud.md fee7199048c3a124fcab2eccae8b88f93d1a69c0
Reviewed-Content-Hash: design/registry/entities.yaml 02f5c072fc535530c710fd71d77d31e2dce021ed
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/dish-values.md 72cf43fcd49019dd6e5281ed8f6d7df8705a71e0
Reviewed-Content-Hash: design/gdd/menu-waves.md 4c2da744c1c4b19bc861bd0c09509d6deecbd6b7

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean)
Blocking items: 5 | Recommended: 9
Summary: The round's central move — availability, visibility and reason all read from `action_availability()`, HUD only formats `*_reason` — is the right contract and the no-legal-action same-frame framing matches ADR-0013 §6. But the reason map contradicts the code vocabulary (FROZEN is "paused", not "ended"; `end_reason` has no copy at all), one ownership line still assigns availability to menu-waves, the disabled-button focus rule contradicts both rule 10 and ADR-0019 #6, and AC 3 still treats `turns_remaining = 0` as a stayable HUD state. Whole-document read (no delta), 6/6 standard sections present; Player Fantasy and Tuning Knobs remain advisory at `standard`.
Prior verdict resolved: Yes — latest entry (2026-11-06, lean, APPROVED) carried 0 blocking items. Its recommended item on AC 21 wording (still "无默认排序控件被激活", `:499`) is still open; the dish-values cross-doc item is outside this file.
Findings:
- [BLOCKING] Core Rules 16 / Edge Cases `:372` — `FROZEN` is mapped to "本局已结束", but ADR-0013 §1 `RejectCode.FROZEN` is "顶层 Paused" and HUD rule 21 (`:69`) itself renders pause as frozen; "本局已结束" belongs to `NOT_ACTIVE`. The two copies are swapped, so the only readable explanation of a frozen skip button states the wrong fact.
- [BLOCKING] Core Rules 16 / Interactions `:164` — rule 16 promises a mapping for both `skip_reason` and `end_reason`, but no edge case, rule or AC gives any `end_reason` copy, and ADR-0013 §9 emits `NO_EATABLE` / `NOT_FINAL_WAVE` / `NONE` as well as the three HUD maps; `NO_TURNS` copy "最后一回合不能跳波" is also false on the `turns_remaining = 0` transient AC 26 itself asserts. Define the closed set, a copy per value and a fallback.
- [BLOCKING] Naming `:164`, `:504` vs `:128` — the snapshot field `end_reason` (run end: `NONE`/`TIME_EXHAUSTED`/`NO_EATABLE_DISH`, registry `:890`) and the availability field `end_reason` (RejectCode for the end-pickup button) are two different enums under one identifier, both listed in the same dependency cell; HUD displays the first and would map the second. Rename one (e.g. `end_disabled_reason`) before any code names it.
- [BLOCKING] Core Rules 10 (`:47`) vs Edge Cases `:372` — rule 10 says controls become truly disabled and unfocusable only in `{Ending, Settling, Handoff, Faulted}` or `frozen`, and deliberately keeps data-state controls focusable so the reason can be read; the new skip rule says an unavailable button "不获得焦点" while `Active`. ADR-0019 #6 (`:36`) likewise requires unavailable controls to stay focusable and read out their reason. As written, AC 26's "按钮不可用并写明…" has no surface to show the text, and C3 becomes an empty focus stop (nav order `:107` has no skip).
- [BLOCKING] Acceptance Criteria 3 (`:472`) — still models `turns_remaining = 0` + `run_state = Active` as a state HUD renders and "等待餐局流程的结束信号", contradicting rule 5 (`:37`), edge `:372` (declared a same-frame transient, "不需要单独…文案") and AC 26's third GIVEN. Either delete AC 3 or restate it as the intra-frame `action_committed`→`run_ended` sequence with no second render.
- [RECOMMENDED] Acceptance Criteria 26 (`:504`) — the static scan only bans the literal comparisons `turns_remaining >= 2` / `<= 1` / `is_final_wave()`; a bare `if menu_waves.is_final_wave():`, `turns_remaining == 0`, `> 1` or a helper alias passes. "全仓静态扫描 HUD 代码" is also scope-ambiguous (run-flow and menu-waves legitimately contain `≥ 2`). Assert "HUD module calls no is_final_wave()/no availability threshold; availability values only from `action_availability()`".
- [RECOMMENDED] Acceptance Criteria 26a (`:505`) — "调试截图中不存在…停留态" is a negative screenshot claim; replace with a frame-probe assertion (exactly one frame between `action_committed` and `run_ended`, no interactive `Active` frame after the judgment).
- [RECOMMENDED] Core Rules 6 (`:39`) — "`entry_fee` 与取整由结算拥有" is stale: `entry_fee` is resolved and frozen by run-flow (run-flow rule 22, settlement rule 2, registry `:641`); settlement owns only the pure functions and rounding. HUD's own dependency row (`:390`) already says so.
- [RECOMMENDED] Edge Cases `:360` — empty-run net still "负数或 0（由结算给值）"; the value is `−entry_fee` from the run-flow snapshot (settlement rule 5 / AC 5), never "由结算给值" at runtime. Prior round's item 21 is unfixed.
- [RECOMMENDED] Edge Cases `:372` — "（规则 9）" points at this file's sorting rule; the intended reference is run-flow rule 9 (end-judgment timing).
- [RECOMMENDED] Interactions `:170` — "本系统另可调用结算的纯函数复核" is not implementable: both pure functions take a `RunLog`/prefix, which HUD never holds (ADR-0007 `:109`, ADR-0019 capture list `:193-213`); only AC 1 compares externally. Say HUD never calls them.
- [RECOMMENDED] Interactions `:164` — the snapshot field list omits `turns_used` and `action_seq`, both used by invariant 2/5 (`:328`, `:331`) and listed in Dependencies (`:385`); add them so the contract cell is the published set.
- [RECOMMENDED] Registry `:686-718`, `:890`, `:1119` — `running_net`/`shortfall` are bidirectional now, but `end_reason`'s `referenced_by` omits this file although HUD displays it, and `onboarding_active` lists HUD although this file never names it; HUD money "最多 5 位" (`:63`, `:316`) still conflicts with `running_net` upper bound `999999` (6 digits); `advance_cost` (`:389`) vs registry `wave_advance_cost`.
- [RECOMMENDED] Process (`systems-index.md:72`) — HUD row is still `Approved` while the header is `In Review`; this substantive edit invalidates the approval (`.claude/rules/design-docs.md:28-31`). `:400` also asserts the index says `Approved`, which is exactly the stale state.

Reviewed-Content-Hash: design/gdd/in-game-hud.md 27e2978106c35139e3a3d0f371472a7aafd5dc2a
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: design/gdd/time-and-turns.md 0f2ea128a9a42008413a15dd8608246513ef36f4
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 43fc521a857c5ce53ba3e1855c271ae2ffdfb1b6
Reviewed-Content-Hash: design/gdd/menu-waves.md a0b26691ca0f6ba5cac4c01ac0704913179b43ed
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/data-config.md 92b3a28fc63465eacf3a88d3fec13bdff9f9455f
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 23304c8212ab59cede6f6f3d81cd40cea079cde5
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean)
Blocking items: 1 | Recommended: 13
Summary: All five prior blocking items are closed and closed correctly: the RejectCode copy map is now FROZEN→"暂停中" / NOT_ACTIVE→"本局已结束" / NO_EATABLE→"现在没有可处理的菜" / NONE silent, the end-pickup button uses the same map, `action_availability().end_reason` is declared a RejectCode domain distinct from the snapshot's run-end `end_reason`, unavailable buttons stay focusable and only fail to submit (`:372`), and AC 3 restates turns=0 as the intra-frame `action_committed`→`run_ended` sequence with no stayable state (`:472`, matching ADR-0013's commit-then-judge order). The one new blocker is the interface cell itself: the "本局只读快照" list at `:164` still omits `running_net` / `shortfall` / `action_seq` / `turns_used`, the exact four fields every rule, invariant and AC in this document reads. Whole-document read (no delta), 6/6 standard sections present; Player Fantasy and Tuning Knobs remain advisory at `standard`.
Prior verdict resolved: Partially — 5/5 blocking closed. Of the prior 9 recommended: 1 closed (the `:372` reference now names `run-flow.md` 规则 9), 8 open and re-raised below; one of them (the snapshot field list) is escalated to blocking because it survived a full revision cycle.
Findings:
- [BLOCKING] Interactions `:164` — the snapshot cell lists `run_id`/`station_id`/`run_state`/`frozen`/`turns_remaining`/`turns_total`/`capacity_used`/`capacity_total`/`eaten_count`/`end_reason`/`run_seed` and omits `running_net`, `shortfall`, `action_seq` and `turns_used`, although `run-flow.md:143-147` publishes all four and rule 6 (`:39`), AC 1/2/4 (`:470-473`), invariants 2/5 (`:328`, `:331`) and Dependencies (`:385`) all read them; an implementer wiring HUD from the interface cell would never receive the two numbers this document exists to show.
- [RECOMMENDED] Edge Cases `:372` — `NONE` is not a member of ADR-0013 §1 `RejectCode` (`adr-0013:91-99`); it exists only as the "可用时 NONE" sentinel in §9 (`:250`) and in ADR-0019 `:747`. Call it a sentinel, or add it to the enum. The same sentence explains `NOT_FINAL_WAVE`/`STALE_FOCUS`/`CAPACITY` as unreachable "because `skip_visible = false`"; only `NOT_FINAL_WAVE` follows from that — the other two are never emitted by the projection at all.
- [RECOMMENDED] Edge Cases `:372` — the end-pickup half gives two entries (`FROZEN`, `NOT_ACTIVE`) and never states its closed set, its `NONE` case or which codes are unreachable (`NOT_FINAL_WAVE` when `end_visible = false`). ADR-0019 `:747` sketches a third set (`NONE / NO_EATABLE / NOT_FINAL_WAVE`) that contradicts ADR-0013 §9; enumerate the end-reachable codes explicitly so one document wins.
- [RECOMMENDED] Naming `:59`/`:164`/`:384` — the disambiguation is now stated, but only in rule 16's parenthetical; the interface cell and Dependencies row still name only the snapshot's `end_reason`, and ADR-0019 `:153` (EndReason) / `:747` (RejectCode) keep the same identifier for the projection inside one snapshot table (`run-flow.md:148` vs `:150`). Rename the projection field (e.g. `end_block_reason`) or restate the qualifier where the contract is read.
- [RECOMMENDED] Acceptance Criteria 26 (`:504`) — the static scan still bans only the literals `turns_remaining >= 2`, `turns_remaining <= 1`, `is_final_wave()`; `== 0`, `> 1`, a helper alias or `menu_waves.is_final_wave()` passes, and "全仓" vs "HUD 代码" is scope-ambiguous. Assert instead: the HUD module contains no availability threshold and no `is_final_wave()` call; availability values come only from `action_availability()`. (Second recurrence.)
- [RECOMMENDED] Acceptance Criteria 26 (`:504`) — the third GIVEN says HUD "按 `skip_reason` 渲染禁用态" at `turns_remaining = 0`; per `:372` the run has already ended in that same frame, so the frame is constructed and superseded, never presented, and the `NO_TURNS` copy would be false if it were drawn. Drop the GIVEN (AC 3 covers it) or say "构造但不呈现".
- [RECOMMENDED] Acceptance Criteria — the new focus rule from fix (4) has no criterion: AC 26 asserts activation submits nothing and the reason is written, but nothing asserts the unavailable skip/end button keeps `focus_mode` and that its reason is reachable from D 焦点详情行, which rule 10 and ADR-0019 `:758` now require.
- [RECOMMENDED] Interactions `:170` — "本系统另可调用结算的纯函数复核" remains unimplementable: `running_net(k)` / `shortfall(k)` take a `RunLog`/prefix that HUD never holds; only AC 1's external comparison does that. Say HUD never calls them. (Second recurrence; run-flow `:126` carries the same sentence.)
- [RECOMMENDED] Core Rules 6 (`:39`) — "`entry_fee` 与取整由结算拥有" is still stale: run-flow resolves and freezes `entry_fee` (run-flow rule 22/23), settlement owns only the pure functions and rounding. (Second recurrence.)
- [RECOMMENDED] Edge Cases `:360` — empty run "净利显示负数或 0（由结算给值）": at runtime the value is `−entry_fee` from the run-flow snapshot; settlement is not called until the receipt. (Second recurrence.)
- [RECOMMENDED] Core Rules 18 (`:63`) / number format (`:316-319`) — "HUD 金额最多 5 位" and the 5-digit min-width still conflict with registry `running_net` range `[-99999, 999999]` (`entities.yaml:695`); net needs 6 digits. (Second recurrence.)
- [RECOMMENDED] Dependencies `:389` — `advance_cost = 1` vs registry `wave_advance_cost` (`entities.yaml:1203`, whose `referenced_by` still omits this file). (Second recurrence.)
- [RECOMMENDED] Registry metadata — `end_reason` (`entities.yaml:890`) `referenced_by` omits this file although `:128` displays it, while `onboarding_active` (`:1119`) lists this file although it never names that symbol; `running_net`/`shortfall` (`:692`, `:709`) are correctly bidirectional. Registry edits are out of scope here.
- [RECOMMENDED] Process — `systems-index.md:72` still reads `Approved` while the header is `In Review`; `.claude/rules/design-docs.md:28-31` voids that approval. `:400` asserts the index says `Approved`, i.e. it records the stale state. (Second recurrence; index not edited here.)

Reviewed-Content-Hash: design/gdd/in-game-hud.md b8ead1263039519de2ff2ad9378fe43a8ea58373
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 23304c8212ab59cede6f6f3d81cd40cea079cde5

---

## Review — 2026-11-17 — Verdict: APPROVED（第 4 轮：快照字段行 + 整篇复检）
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean)
Blocking items: 0 | Recommended: 18
Summary: 第三轮的唯一 blocker 已关闭且改到位：`Interactions :164` 的「本局只读快照」现在逐字列出 15 个字段（`run_id` / `station_id` / `run_state` / `frozen` / `turns_remaining` / `turns_total` / `turns_used` / `capacity_used` / `capacity_total` / `eaten_count` / `running_net` / `shortfall` / `action_seq` / `end_reason` / `run_seed`），与 `run-flow.md:138-150` 的快照表一一对应，规则 6、AC 1/2/4、不变式 2/5 与 Dependencies 读的四个字段全部在内。整篇复检确认：HUD 呈现的每一个数字都能在快照或 `action_availability()` 里找到来源（档口显示名与菜品图标由投影层向数据层取、错误码经 `run_faulted` 参数到达，见 N1）；规则 16 与边角案例的可用性契约只读 `action_availability()`，并把 `*_reason` 与快照的 run-end `end_reason` 明确声明为两个域；`FROZEN`/`NOT_ACTIVE`/`NO_EATABLE`/`NONE` 文案与「不可用按钮仍可获焦、激活不提交」与 ADR-0013 §9、ADR-0019 D6.1/实现准则 7 一致；AC 3（单帧瞬态）、AC 24（跳波文案与 `visible_dishes` 数量钉死）、AC 26（可用性不自行判定）、AC 26a（同帧 `Ending`）四者互相自洽，AC 1–42 + 26a 编号连续无重复。剩余 18 条为跨文档口径、字段命名与公式自洽问题，不阻断实现。全篇阅读（非 delta），6/6 standard 节齐备；Player Fantasy 与 Tuning Knobs 在 `standard` 下仍为 advisory。
Prior verdict resolved: Yes — 第三轮 1/1 blocking 关闭。其 13 条 recommended 中 0 条关闭（`NONE` 域表述、结束取餐闭合集、投影字段命名、AC 26 静态扫描、AC 26 的 turns=0 GIVEN、焦点可获焦判据、结算纯函数、`entry_fee` 归属、空局净利、5 位金额、`advance_cost`、注册表元数据、索引状态），全部按原样再提；另补入第一轮的 4 条遗留（布局拉伸口径、`hud_capacity_cells` 值域、`hud_area_usage` 右栏高度、`can_fit`/`get_dish`/`turns_total` 口径）与 1 条新问题。
Findings:
- [RECOMMENDED] Edge Cases `:372` / Core Rules 16 `:59` — 取值域仍写作「RejectCode」，上游是 `RejectCode ∪ {NONE}`（`run-flow.md:33`、ADR-0013 §9 `:250`）；同一句把 `STALE_FOCUS` / `CAPACITY` 的不可达解释为「`skip_visible = false`」，但这两个码根本不由跳波投影产生（ADR-0013 §3 只由 `eat` 产生），只有 `NOT_FINAL_WAVE` 才由 `skip_visible` 决定。（第三轮再提）
- [RECOMMENDED] Edge Cases `:372` — 结束取餐半个映射只列 `FROZEN` → 「暂停中」、`NOT_ACTIVE` → 「本局已结束」，未写闭合集、`NONE` 与不可达码；ADR-0019 `:744-747` 又给出第三套词表（`NO_TURNS_ZERO` / `NO_TURNS_ONE`、`NONE` / `NO_EATABLE` / `NOT_FINAL_WAVE`），与 ADR-0013 §9 不一致。写明以生产者 ADR-0013 §9 为准，并列出结束取餐可达码与兜底文案。（第三轮再提）
- [RECOMMENDED] Naming `:59` / `:164` / `:384` — 两个 `end_reason` 的消歧仍只写在规则 16 的括注里；接口单元与 Dependencies 行只出现快照那一个，实现者按契约单元命名时仍会撞名。建议改投影字段名（如 `end_block_reason`）或在两处契约单元重复限定。（第三轮再提）
- [RECOMMENDED] Acceptance Criteria 26 `:504` — 静态扫描仍只禁三处字面量（`turns_remaining >= 2`、`<= 1`、`is_final_wave()`），`== 0`、`> 1`、helper 别名或 `menu_waves.is_final_wave()` 都能通过；「全仓静态扫描 HUD 代码」范围仍含混。改为断言「HUD 模块内不存在任何可用性阈值与 `is_final_wave()` 调用；可用性只来自 `action_availability()`」。（第三轮再提）
- [RECOMMENDED] Acceptance Criteria 26 `:504` — 第三个 GIVEN 仍写 `turns_remaining = 0` 时「HUD 按 `skip_reason` 渲染禁用态」，与同一 AC 与 `:372`「该帧已按 `TIME_EXHAUSTED` 结束、不存在停留画面」自相矛盾；改为「构造但不呈现」或删除（由 AC 3 覆盖）。（第二轮再提）
- [RECOMMENDED] Acceptance Criteria — 规则 10、`:372` 与 ADR-0019 D6.1 `:410` / 实现准则 7 要求的「不可用按钮保持 `focus_mode = FOCUS_ALL`、原因可从 D 焦点详情行读出」仍无判据：AC 26/26a 只断言激活不提交与同帧结束。（第二轮再提）
- [RECOMMENDED] Interactions `:170` — 「本系统另可调用结算的纯函数复核」仍不可实现：`running_net(k)` / `shortfall(k)` 吃 `RunLog` / 前缀，HUD 从不持有；只有 AC 1 的外部对照做这件事。改为「HUD 从不调用它们」。同一句仍留在 `settlement-and-entry-fee.md:105`、`:338` 与 `run-flow.md:126`（跨文档）。（第三轮再提）
- [RECOMMENDED] Core Rules 6 `:39` — 「`entry_fee` 与取整由结算拥有」仍陈旧：`entry_fee` 由餐局流程解析并冻结（`run-flow.md` 规则 22/23），结算只拥有两个纯函数与取整；本文件自己的 `:390` 已写对。（第三轮再提）
- [RECOMMENDED] Edge Cases `:360` — 空局仍写「净利显示负数或 0（由结算给值）」；运行期值是快照的 `−entry_fee`，结算在收据前不参与。（第三轮再提）
- [RECOMMENDED] Core Rules 18 `:63` 与 `hud_number_format` `:316-319` — 「HUD 金额最多 5 位」与 5 位 `min-width` 仍与注册表 `running_net` 上界 `999999`（6 位，`entities.yaml:695`）差一位。（第三轮再提）
- [RECOMMENDED] Dependencies `:389` — 仍写 `advance_cost = 1`，注册表条目名是 `wave_advance_cost`（`entities.yaml:1203`，其 `referenced_by` 仍不含本文件）。（第三轮再提）
- [RECOMMENDED] 布局与信息层级 `:75` 对 `hud_ui_scale` `:215` / ADR-0019 D5.1 `:265` — 前者写「采用 `canvas_items` 拉伸…UI 整体随窗口等比放大」，后者写 HUD 画在不叠加拉伸的独立层、按窗口像素并用 `1/c` 逆缩放，OQ7（`:542`）仍沿用前一套前提。统一为「工程用 `canvas_items` + `expand`；HUD 用 `CanvasLayer` 逆内容缩放、以窗口像素布局」。（第一轮遗留）
- [RECOMMENDED] `hud_capacity_cells` `:252` / `:267` — Output Range 仍写 `cells ∈ [0, 24]`，而式子是 `max(1, …)`，值域是 [1, 24]；也仍未像 `hud_turn_slots` 那样写明 `capacity_total = 0` 时「不渲染分段」。（第一轮遗留）
- [RECOMMENDED] `hud_area_usage` `:296` / 分区表 `:98-103` / AC 12 `:484` — 右栏仍无 y 范围，例子取高 560 得 26.5%，而按图（顶栏下缘 108、内容下缘 684）应为 576 → 26.9%；补 y 范围或改算例与 AC。
- [RECOMMENDED] 三处上游口径仍不符：`can_fit(d)`（`:47`、`:166`、`:300`、`:309`、`:330`、`:385`）对 `stomach-capacity.md:31` / 注册表 `can_fit` 的 `can_fit(e)`；`get_dish(id)`（`:167`、`:386`）对 `dish-values.md:95` 的 `get_dish(id, caller)`；`turns_total` 范围 1–999（`:242`）对 `time-and-turns.md:101` 的 1–INT32_MAX，而 AC 17 又测 `turns_total = 0`。（第一轮遗留）
- [RECOMMENDED] Interactions `:173` / Dependencies `:392` — 「菜品与档口记录由菜品数值/菜单波次解析后传入；HUD 不直接查询数据目录」漏掉档口显示名与菜品图标：ADR-0019 D3 第 7 步（`:213`）由 `HudSource` 调 `data.get_station(station_id)` 与 `data.get_dish_icon(id)`，首帧 `station_name` 无其它来源。写明「投影层（`HudSource`）可读数据层，视图层不可」。
- [RECOMMENDED] Registry 元数据 — `end_reason`（`entities.yaml:890`）与 `can_fit`（`:922`）的 `referenced_by` 仍不含本文件（`:128` 显示前者、`:47` 等消费后者），而 `onboarding_active`（`:1119`）仍列本文件却从未点名该符号；`running_net` / `shortfall` 已双向正确。注册表不在本次可改范围。
- [RECOMMENDED] Process — `systems-index.md:72` 仍读 `Approved`，而本文件表头是 `In Review`，`.claude/rules/design-docs.md:28-31` 使该批准失效；`:400` 的「双向一致性」段仍断言索引写 `Approved`，等于把陈旧状态记录在案。（第三轮再提；索引未在此修改）

### Nice-to-Have

- `fault_code` 的来源没写进快照表或接口表：它经 `run_faulted(run_id, code, detail)` 的 `code` 参数到达（ADR-0019 `HudFrame.fault_code` `:154`），而 `:128` 的信息清单只有「结束原因」。补一句即可。
- `hud_safe_margin` 的下限 32 px 在最低支持分辨率下不可达（`round(0.05 × 720) = 36`），可标注为纯防御值。
- `hud_short_cells` 的 `effective_satiety` 变量范围写 1–999，注册表 `effective_satiety` 是 [1, 9]；放宽不冲突，但可对齐。
- 规则 18「净利总带符号」对 `running_net = 0` 未定义（`0` 还是 `+0`）；AC 17 只测 +39 / −46。
- `:166` 的括注「容量可能在 `run_ended` 重置」不准：容量在 `end_run(reason)` 时清零（`stomach-capacity.md` 规则 2、`run-flow.md` 规则 24），`run_ended` 本身不重置；结论（`Ending` 期间只读最后一帧快照）仍然正确。

### 判定说明

lean 模式不派发 `creative-director`，主评审意见即最终意见。第三轮的唯一 blocker 是接口契约单元漏字段，本轮已按 `run-flow.md:138-150` 补齐 15 个字段并与全文读法一致；这是「HUD 为什么存在」的那两个数字（`running_net` / `shortfall`）能否到达实现者的关键，现已闭合。可用性契约（可见性、可用性、原因三件套只读 `action_availability()`，两个 `end_reason` 分域，不可用按钮仍可获焦）与 ADR-0013 §9、ADR-0019 D6.1 一致，也没有任何 AC 依赖 HUD 自己比较阈值。剩余条目全部是文档口径、命名与公式自洽层面的问题：没有一条会改变信号形状或让某条 AC 无法判定，因此不构成 blocking。真正需要在写 `design/ux/hud.md` 之前处理的是第 2、3、4 条（可用性原因词表以谁为准、投影字段命名、静态扫描判据），它们决定文案表与验收脚本的形态。

### Scope Signal

Rough scope signal: **L**（8 个硬上游 + 5 个下游 / 同级系统，10 组呈现公式，无新增 ADR；producer 应在排期前复核）。

### Verdict: APPROVED

0 条 blocking、18 条 recommended、5 条 nice-to-have。`systems-index.md:72` 的状态收口由编排会话完成（本次不修改索引）。本 GDD 未被修改。

Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 93968bd02251e306f5b99c7b1cc5800272e54a06
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/input.md 4ac257e9cab7df2ce607f4063416a308b56dccb4
Reviewed-Content-Hash: design/gdd/data-config.md 8d799e593ad60b254814ede22b4f9b3fc8de99e4
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 23304c8212ab59cede6f6f3d81cd40cea079cde5
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/in-game-hud.md 6d7a4700f0f8b718eeb3bf0cbe9db7993df1f43d
