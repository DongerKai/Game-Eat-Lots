# 纪录与里程碑（Records and Milestones）— Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 3 | Recommended: 10
Summary: 纪录模型本身站得住：严格大于才刷新、平局保留旧纪录、纪录字段冻结、生涯合计 checked int64、完成条件的存在量词与锁存语义都精确，AC 也逐条可断言。三条阻塞都在跨文档契约：`audio-feedback` 明确说纪录事件不产生声音，与本 GDD 的三个信号相反；VS 基线目录没有任何绑定第二档口 `station_banquet` 的必需里程碑，通关因此不等于概念文档的「吃遍所有档口」；`MilestoneData` 归属仍在 OQ2，而已 Approved 的 `data-config` 明确不含它，且本 GDD 内部「第四类 / 第五类」自相矛盾。其余为陈旧回指（OQ1 已由 `ui-flow` 回答）、`upgrade_snapshot` / `return_ratio` 的上游措辞冲突，以及注册表回指缺失。
Prior verdict resolved: First review

Findings:
- [BLOCKING] Interactions（`:103`）/ Dependencies（`:298`）/ Visual-Audio（`:346`）vs `audio-feedback.md:376`：本 GDD 把 `record_broken` / `milestone_earned` / `career_completed` 三个信号交给音效反馈实现；`audio-feedback` 写「局外升级 / 纪录与里程碑 / 解锁与档口：无直接调用。它们的事件不产生声音」，其依赖表也没有本系统的行。依赖不双向，且里程碑与通关没有任何听觉反馈的归属。
- [BLOCKING] Formulas 基线目录（`:237-249`）vs `game-concept.md:128`、`:251`、`unlocks-and-stations.md:301`：VS 有两个档口（`station_canteen` + `station_banquet`），基线 9 条里程碑没有一条绑定 `station_banquet`，唯一必需项是 `ms_canteen_return_1000`。只在食堂打一局达标即可「通关」，与「完成全部档口挑战即视为通关」相反，也违反本 GDD 自己「每个档口至少一条 `*_return_1000` 并标记为完成必需」的规则。AC 32（`:428`）只校验「必需里程碑的目标档口存在」，没有反向校验。
- [BLOCKING] Dependencies 扩展契约（`:291`、`:313`）/ OQ2（`:443`）vs `data-config.md:28`、`:186`：本系统把里程碑目录列为硬上游并要求扩展，但已 APPROVED 的 `data-config` 明写 `MilestoneData` 当前不在 schema 内、归属待本 GDD 的 OQ2；同时 `:291`/`:313` 称「第四类记录」，`:443` 称「第五类记录」，同一份文档给出两个序数。
- [RECOMMENDED] 编排者契约（`:120`）/ OQ1（`:442`）/ AC 23（`:416`）、AC 28（`:421`）vs `ui-flow.md:59`：OQ1 已被 `ui-flow` 规则 10 回答——界面流程就是 MVP 的结算事务编排者，且调用顺序（先 `apply_run`，致命错误即整笔放弃）与本 GDD 规则 3 一致。本 GDD 仍写「编排者身份尚未指定」，两条 AC 仍标注「编排者指定后可执行」，依赖表（`:297`）只写界面流程「只读」。
- [RECOMMENDED] `StationRecord.best.upgrade_snapshot`（`:84`）vs `settlement-and-entry-fee.md:70`、`run-flow.md:70`：本 GDD 断言它是「已解析的本局修正值」，`run-flow` 同意，但直接上游 `settlement` 称其为「等级快照」。三方未统一，本 GDD 单方面选了 `run-flow` 一侧。
- [RECOMMENDED] 规则 3 一致性断言（`:29`）vs `settlement-and-entry-fee.md:180-183`、`:297`、注册表 `return_ratio`：空局给出 `total_value = 0`，本系统按公式得到 `return_ratio = 0` / `return_percent_x10 = 0`，而 `settlement` 声明 `return_ratio` 范围 `(0, +∞)` 且「> 0」。上游声明与本系统的合法输入冲突，实现在空局上会撞到 `INCONSISTENT_RESULT` 的解释分歧。
- [RECOMMENDED] 编排者契约（`:120`）vs `save-system.md:254`、`unlocks-and-stations.md:55`：本 GDD 要求把 `unlocks` 与 `progression`、`records` 合并为一次提交；存档与解锁的 GDD 都明确 MVP/VS 结算不产生解锁，解锁是菜单购买的独立事务，`ui-flow.md:59` 的写法是「未来存在的 `unlocks`」。
- [RECOMMENDED] 完成条件评估时点（`:116` vs `:270`）：接口表写「载入时补齐状态不发」，暗示载入会补锁 `completed_at`；边界用例写移除必需项后「下一次结算可能立即通关」，即载入不评估。两处对「载入是否写入 `completed_at`」给出不同答案，AC 未覆盖。
- [RECOMMENDED] 去重键（`:28`、`:261`、`:65-72`）：规则只与「上一次已应用的 `run_id`」比较，但命名空间 schema 与状态表都没有 `last_applied_run_id` 字段。实现者需要猜它是内存字段还是持久化字段、重载后是否失效、更早 `run_id` 的重放会不会被重复累计。
- [RECOMMENDED] 注册表（`design/registry/entities.yaml`）：只登记了 `record_is_better` 与 `milestone_met`，本 GDD 定义的 `career_accumulate`、`career_complete` 缺失；`record_is_better.referenced_by` 列了 `settlement-and-entry-fee.md`，而该文件从未出现这个名字；`save_bytes`、`total_value`、`net_profit`、`return_ratio`、`end_reason`、`run_id_alloc` 的 `referenced_by` 均未加 `records-and-milestones.md`。
- [RECOMMENDED] `save_bytes` 单条常量（`save-system.md:201`、`:203`）与 AC 33（`:429`）：`R = 128 B/条纪录`、`M = 48 B/个里程碑` 相对本 GDD 的 `StationRecord`（8 字段 + `upgrade_snapshot` 字典）与 `MilestoneEntry`（id + ISO 时间戳 + int64 阈值）偏小；总量仍远低于 256 KiB，于是 AC 33 的「不超过 `SAVE_BYTES_MAX`」几乎是恒真，失去预算意义。
- [RECOMMENDED] 规则 4（`:30`）vs 基线 `ms_first_break_even`（`:242`）/ 进度取值（`:207`）：规则 4 说不同档口入场费不同、绝对净利不可直接比较，但无档口过滤的 `NET_PROFIT` 里程碑与「所有档口 best 的最大值」进度都跨档口比较绝对净利。τ = 0 时无碍，τ > 0 时会偏向低费档口。

Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md bcde83821bc73ad7c67a24d38a8659ce2d53693b
Reviewed-Content-Hash: design/gdd/systems-index.md 182d446c25ab8aa50ccf7d672bc2132eeb2b22d3
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

---

## Design Review: 纪录与里程碑（Records and Milestones）

Specialists consulted: none (lean mode — Phase 3b not run)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview、Detailed Rules（`## Detailed Design`
是接受的标题）、Formulas（本系统定义数值规则：比较阈值、checked 累加、四种里程碑
指标、完成条件）、Edge Cases、Dependencies、Acceptance Criteria。六节全部存在。

Advisory at `standard`:
- **Player Fantasy** — 无独立标题；`## Game Feel`（`:350-356`）承担了预期感受
  （「像翻账本，不像看成就」「比较是即时的」「失败不羞辱」）。
- **Tuning Knobs** — 无独立标题。阈值确实全部落在 `MilestoneData` 数据里（这是对的），
  但缺一张「安全范围」表；见 Recommended 10。

> 说明：玩家幻想与调参旋钮在 `standard` 下都是 advisory，不阻断。

### Dependency Graph

- ✓ `settlement-and-entry-fee.md` — 存在（`:289`）
- ✓ `save-system.md` — 存在（`:290`）
- ✓ `data-config.md` — 存在（`:291`），但**当前不提供 `MilestoneData`**（Blocking 3）
- ✓ `ui-flow.md` — 存在（`:297`）；它同时是本系统的结算编排者（Recommended 4）
- ✓ `audio-feedback.md` — 存在（`:298`），但**明确否认本系统的依赖**（Blocking 1）
- ✗ `在线挑战` — GDD 未写（#19），本文件已正确标注「GDD 未写」（`:299`）

`run-flow.md` 与 `unlocks-and-stations.md` 在 Cross-References / 正文中被引用，两者均存在。

### Registry Values

对注册表条目名与本文档数值做全文检索（`total_value`、`net_profit`、`return_ratio`、
`end_reason`、`run_id_alloc`、`save_bytes`、`save_bytes_max`、
`station_canteen_entry_fee`、`opt_value`、`calibration_ratio`、`save_schema_version`，
以及数值 85 / 104 / 1223 / 500 / 2000 / 1100 / 1200 / 24 / 262144）：

- 入场费 85、`opt_value` 104、`calibration_ratio` 1.224 → 上限 1223、基准点 1100/1200：
  与注册表 `station_canteen_entry_fee`、`opt_value`、`calibration_ratio` 一致。
- `SAVE_BYTES_MAX` 262144（`:429`）：与注册表一致。
- `record_is_better`、`milestone_met`：名称与表达式与注册表一致。
- `return_ratio`：数值行为一致，但上游声明的输出范围与空局取值冲突（Recommended 6）。
- **无值冲突**。唯一的 id 问题：AC 5（`:395`）把 `station_noodle` 用作第二档口，
  但 VS 的第二档口是 `station_banquet`（暂定 id，`unlocks-and-stations.md:151`、
  `:202`、`:301`）。作为单元测试夹具可以虚构 id；作为「玩家可见行为 + 截图留存」
  的验收用例，应改用真实档口 id，否则截图里的档口在产品中不存在。

### Required Before Implementation

1. **[音效反馈依赖是单向的，且双方给出相反事实]**
   `records-and-milestones.md:103`（Interactions：音效反馈 ← `record_broken` /
   `milestone_earned` / `career_completed`，软）、`:298`（Dependencies 同）、
   `:346`（Visual/Audio：「音效事件（由音效反馈实现…）」并要求同帧合并） vs
   `audio-feedback.md:376`（「局外升级 / 纪录与里程碑 / 解锁与档口：无直接调用。
   它们的事件不产生声音」）。`audio-feedback` 的 Dependencies 表没有本系统的行，
   `systems-index.md` 第 17 行也不含本系统。
   
   后果：里程碑达成与通关这两个奖励时刻没有任何声音归属；两个 GDD 对同一接口
   给出相反事实，实现者不知道该订阅谁。
   
   修法（二选一，建议 A）：
   - A. `audio-feedback` 增加「纪录与里程碑」行与三个 cue（`record_broken` 图章声、
     `milestone_earned` 勾选声、`career_completed` 收尾音），并把 `:376` 的排除范围
     改为只剩局外升级 / 解锁；本 GDD 保留三个信号。
   - B. 本 GDD 删除「由音效反馈实现」的表述与 `:346` 的音效事件段，明确这三个
     时刻只由界面流程的 UI 音效承担。
   
   > 稳定性提示：`audio-feedback` 此刻正在评审中（review-audio），其结论可能改变
   > 这条依赖的可用性。本结论基于 main 上的 `:376`。

2. **[基线里程碑目录不覆盖第二档口，因此通关 ≠「吃遍所有档口」]**
   `:237-249` 的 9 条基线里程碑中，只有 `ms_canteen_return_1000` 标为完成必需
   （`:243`），其余都是食堂的加码项或与档口无关的生涯项，**没有任何一条绑定
   `station_banquet`**。`:251` 自己规定「新增档口时，每个档口至少一条
   `*_return_1000` 并标记为完成必需，通关条件才覆盖全部档口」，同句又说
   「当前基线在 Vertical Slice 只有 2 个档口时才成立」——句子与表格自相矛盾。
   `game-concept.md:128` 的长期目标是「吃遍所有档口……完成全部档口挑战即视为通关」；
   VS 的两档口是 `station_canteen` 与 `station_banquet`（后者为暂定 id，
   `unlocks-and-stations.md:151`、`:202`、`:301`）。
   
   后果：玩家只在食堂打到达标就能通关，宴会档口（120 元解锁）可以完全不碰；
   概念文档承诺的通关条件不成立。
   
   修法：
   - 为 `station_banquet` 增加一条必需里程碑（如 `ms_banquet_return_1000`，
     阈值待 OQ3 校准），并把 `:251` 的规则从散文提升为可校验的约束。
   - 在数据校验里增加反向约束：**每个已定义档口至少有一条
     `required_for_completion = true` 且绑定该档口的里程碑**，违反给
     `STATION_WITHOUT_REQUIRED_MILESTONE`（与 `REQUIRED_MILESTONE_UNREACHABLE`
     对称：一条防「指向不存在的档口」，一条防「档口没有任何必需项」）。
   - AC 32（`:428`）补上这条校验。
   - AC 5（`:395`）的截图用例改用 `station_banquet`（或明确标注该 AC 只在测试数据
     集下执行，不要求截图）。

3. **[`MilestoneData` 的归属未定，而 Approved 的上游明确不含它；序数自相矛盾]**
   `:291`（依赖表：「GDD 已写，**需扩展第四类记录**」）、`:313`（「要求新增第四类
   记录 `MilestoneData`」） vs `:443`（OQ2：「数据配置现有四类记录……将成为
   **第五类**记录」）；`data-config.md:28` 明写「四类记录，四类 Resource……
   `MilestoneData` 是否成为第五类由纪录与里程碑 Open Question 2 决定，当前不在
   schema 内」，`:186` 同；`data-config` 已于 2026-11-04 APPROVED，该状态仍然成立。
   
   后果：本系统唯一的硬上游不提供里程碑目录，实现无法开始；同时「第四类」的写法
   会与既有的 `BalanceData` 序数冲突，若被照抄进 data-config 会产生歧义。
   
   修法：
   - 在架构阶段前关闭 OQ2（负责人 technical-director，`:443` 已写）。
   - 若选「放进数据配置」：把 `:291`、`:313` 的「第四类」改为「第五类」，
     并让 data-config 增补该类型（届时需重开其评审）。
   - 若选「纪录自带只读 loader」：删除「唯一数值来源」与之冲突的表述（`:325`
     已把这条路写出来），并把 `:291` 的依赖状态改为「记录自带目录，不依赖
     data-config 扩展」。

### Recommended Revisions

4. **OQ1 已被 `ui-flow` 回答，本 GDD 的回指是陈旧的。**
   `ui-flow.md:59`（规则 10）写「界面流程是 MVP 的**结算事务编排者**（回答
   `纪录与里程碑` Open Question 1）」，并规定：①先 `apply_run`（即
   `apply_run`），致命错误整笔放弃；②再 `apply_run_result`；③一次
   `commit({progression, records[, unlocks]})`。这与本 GDD 规则 3 与 `:120` 的
   契约实质一致。但本 GDD `:120` 仍写「编排者身份尚未指定」，`:442` 的 OQ1 仍为
   未决，AC 23（`:416`）与 AC 28（`:421`）仍标注「编排者指定后可执行」，
   Dependencies（`:297`）只把界面流程记为「只读」。
   
   修法：把 OQ1 标为已回答（界面流程，2026-11-04 由 `ui-flow.md:59`），依赖表
   的界面流程行补「结算事务编排者」角色，删去两条 AC 的括号注（测试现在可执行），
   并保留一句「若架构阶段改由场景流程编排，则…」的条件句。
   
   > 稳定性提示：`ui-flow` 当前是 Designed、评审进行中（review-ui-flow）。若其
   > 规则 10 被改动，本结论需要重新核对。

5. **`upgrade_snapshot` 的语义三方未统一。**
   本 GDD `:84` 说它是「已解析的本局修正值，语义归局外升级；本系统不解释」；
   `run-flow.md:70` 明确「携带的是**已解析的本局修正值**，不是等级」，并指出
   「结算 GDD 用『等级快照』概括该字段」；`settlement-and-entry-fee.md:70` 仍写
   「本局的局外升级**等级快照**」。本系统直接断言了一种语义，与直接上游相反。
   
   修法：`:84` 改为「原样冻结 `RunResult.upgrade_snapshot`；其语义以 `run-flow`
   为准（已解析修正值），本系统不解释、不重算；`settlement` 的『等级快照』措辞
   待其修订时统一」。若未来纪录页要展示「当时带了哪些升级」，需要等级而非解析值，
   应在该字段里同时保留两者（`run-flow:70` 已提出这个条件）。
   
   > 稳定性提示：`settlement`（#11）当前是 Needs Revision，此冲突很可能在其修订中
   > 被消除；本结论基于 main 上的 `:70`。

6. **空局的 `return_ratio` 取值与上游声明的范围冲突。**
   本 GDD `:29` 要求 `entry_fee > 0` 时 `return_percent_x10 = floor(1000 × total /
   fee)` 且 `return_ratio` 非空。空局（`settlement:297`：`total_value = 0`）因此
   必然得到 `return_ratio = 0`、`return_percent_x10 = 0`。但 `settlement:180-183`
   声明 `return_ratio` 的范围是 `(0, +∞)` 且「`return_ratio > 0`」；注册表的
   `return_ratio` 写 `[0, 999]`。
   
   修法：本 GDD 明确「空局 `return_ratio = 0` 是合法结果」（规则 3 与边界用例各加
   半句），并请 `settlement` 在其修订中把范围改为 `[0, +∞)`。否则实现者面对
   「上游说 > 0、本系统算出 0」时，只能选择忽略一边。
   
   > 稳定性提示：`settlement`（#11）当前是 Needs Revision。

7. **`unlocks` 被写成结算事务的必合并命名空间。**
   `:120` 要求编排者把 `progression`、`records`、`unlocks` 合并为**一次**提交。
   但 `unlocks-and-stations.md:55`（规则 15）明确「一局结束不改变解锁（MVP 与
   Vertical Slice 都不存在局内奖励解锁）」，`save-system.md:254` 说结算产生的解锁
   才合并、MVP/VS 的菜单解锁是独立事务，`ui-flow.md:59` 的措辞是「（以及未来存在
   的 `unlocks`）」。
   
   修法：改为「若该次结算产生 `unlocks`（当前 MVP/VS 不产生），一并提交」。
   否则实现者可能把 `unlocks` 命名空间无条件塞进结算事务，与存档的串行规则冲突。
   
   > 稳定性提示：`save-system`（#4）当前是 Needs Revision，`:254` 可能随修订移动。

8. **完成条件的评估时点不唯一。**
   `:116`（接口表）写 `career_completed()`「完成条件首次锁存时发一次；载入时补齐
   状态不发」，暗示载入会补写 `completed_at`；`:270`（边界用例）写移除必需里程碑后
   「下一次结算可能立即通关」，即载入时不评估。两处对「载入是否写入 `completed_at`」
   给出不同答案，AC 也没有覆盖这个时点。
   
   修法：定一条并写进规则 14 与 AC 21：建议「`completed_at` 只在 `apply_run` 的
   锁存步骤写入；载入只读」。若改为「载入补锁」，则必须同时规定横幅与
   `career_completed()` 在载入路径上的行为（不重播），以及目录在两次启动之间变化时
   的语义。

9. **去重键没有落在 schema 或状态表里。**
   `:28` 只与「上一次已应用的 `run_id`」比较，`:261` 说与结算的
   `last_settled_run_id` 同口径；但命名空间 schema（`:65-72`）、`StationRecord`
   （`:74-85`）、`MilestoneEntry`（`:87-92`）与状态表（`:53-61`）都没有
   `last_applied_run_id`。实现者需要猜：它是内存字段还是持久化字段？重载后是否清零？
   更早 `run_id` 的重放（A → B → 再投递 A）会不会被重复累计？
   
   修法：加一行说明——`last_applied_run_id` 为会话内内存字段、不持久化、只挡住
   连续重放；更早结果的重放由结算的 exactly-once 保证兜底。若要连「A → B → A」
   也挡住，需要把去重键改成会话内的 `run_id` 集合（仍不持久化），并补一条 AC。

10. **无档口过滤的 `NET_PROFIT` 里程碑与规则 4 自相矛盾，且 Tuning Knobs 缺失。**
    `:30`（规则 4）说不同档口入场费不同、绝对净利不可直接比较；但基线
    `ms_first_break_even`（`:242`，`NET_PROFIT`、无过滤器）与进度取值
    （`:207`：「所有档口 `best.net_profit` 的最大值」）都在跨档口比较绝对净利。
    τ = 0 时无碍（回本线与费用比例无关），τ > 0 时低入场费档口更容易达标。
    
    修法：规定 `NET_PROFIT` 指标在 τ > 0 时必须带 `station_id`（否则改写为
    `RETURN_RATIO_X10`），并把这条加入 `MilestoneData` 的数据校验（`:315-323`）。
    同时补一节 Tuning Knobs（advisory at `standard`，但本系统所有玩法数值都在
    这里）：`RETURN_RATIO_X10` 阈值应低于该档口可达上限（食堂 1223）、
    `RUNS_SETTLED` 阈值与解锁所需局数的关系（`unlocks-and-stations.md:273`：
    7 局解锁宴会档口）、以及 `LIFETIME_NET_PROFIT` 的时间尺度。

### Specialist Disagreements

无（lean 模式未运行 specialist；主评审内部未出现需要裁决的分歧）。

### Nice-to-Have

- `MILESTONE_CATALOGUE_INVALID` 未出现在 `:120` 的编排者错误清单（只列
  `INCONSISTENT_RESULT` / `OVERFLOW` / `DATA_NOT_READY`），而 AC 27（`:420`）
  会返回它——编排者不知道遇到它时该不该中止整笔事务。补进清单即可。
- `threshold_at_earn` 在载入时缺失的处理未定义（`:43` 只覆盖 `achieved_at` 与
  `earned_at`）。时间戳规则可以复用，但阈值字段是 `MilestoneEntry` 的语义字段，
  建议明确「缺失时保留 id、`threshold_at_earn` 置 null 并写警告」，与时间戳的
  宽松策略区分开。
- 免单档口（`entry_fee = 0`）与「每个档口一条 `*_return_1000` 必需」在
  `:194` 的 `INVALID_MILESTONE_FOR_ZERO_FEE` 下互斥。若未来出现免单档口，
  需要一条替代的必需里程碑（`NET_PROFIT` 或 `RUNS_SETTLED`）。
- AC 19（`:412`）第二个 GIVEN 没有 WHEN，无法直接执行。
- AC 6（`:396`）的「重新进入纪录页不重复播放庆祝」被自己标为观察项；AC 21
  已经覆盖「重启后不重播」，建议把 AC 6 收缩为可截图断言的部分。
- 载入时 `record_broken` / `milestone_earned` 是否发信号未明说（`:116` 只说了
  `career_completed` 不发）。建议在接口表加一句「载入路径不发任何信号」。
- 注册表的 `record_is_better.referenced_by` 里 `settlement-and-entry-fee.md`
  是一处错误回指；补 `career_accumulate` / `career_complete` 时一并修正。

### Senior Verdict

> N/A — lean 模式不运行 `creative-director` 综合评审；本结论由主评审给出。

**结论：纪录模型本身是该批 GDD 里最严谨的一份之一。** 比较规则、冻结语义、
checked 累加、存在量词防「空目录自动通关」、锁存后不回退，每一条都能直接翻成
代码；AC 全部可断言，测试夹具的数值（F1/F2/F3/S1/F5/F6）互相一致，示例可手算复核。
它失败的地方不在机制，而在**契约的另一半**：与它对接的三个系统里，
`audio-feedback` 明确否认连接，`data-config` 尚未承载 `MilestoneData`，
而 VS 的基线目录漏掉了第二个档口——恰好是通关条件的核心承诺。
按当前 main 的版本，三条阻塞都能在不动机制的前提下修复：改依赖归属、补一条必需
里程碑与一条校验、关闭 OQ2 并修正序数。

### Scope Signal

依赖 3 硬上游 + 3 下游、4 条公式、一个新存档命名空间、一个新数据类型
（`MilestoneData`，归属未定，按约定需要一条架构决策记录）、一个新 UI 屏幕
（纪录页）。按本技能的分级属于 **L**（多系统集成、3+ 公式、可能新增 ADR）。
`systems-index.md:212` 把本系统记为 **S**——该估计与本 GDD 暴露的集成面不符，
生产排期前应重新评估。

### Verdict: NEEDS REVISION

### 建议状态

`systems-index.md` 第 14 行：**`Designed` → `Needs Revision`**。
三项阻塞修复并复审通过后再改为 `Approved`。（本轮不修改索引文件，按并行评审约束。）
