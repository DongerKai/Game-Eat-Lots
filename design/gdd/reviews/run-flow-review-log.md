# 餐局流程（run-flow.md）— 设计评审日志

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 5 | Recommended: 11
Summary: 状态机、动作仲裁、结束判定与"放弃不是结算"的边界写得非常扎实，四个 Formulas 与注册表逐字一致，13 条上下游接口中 11 条完全对得上。五个 blocking 项全部是**接口契约缺口**，不是设计返工：菜单波次没有"吃下这道菜"的调用入口、`RunConfig` 缺 `entry_fee_override`（首局教学价 40 无法到达结算）、`StationData.turn_budget` 的读取方未声明、动作被拒的返回值未定义（HUD 却在消费它）、`pool_override` 在本文件被声称而菜单波次尚未接受。前四项都是几行 schema / 接口行的补齐。
Prior verdict resolved: First review。

### 复审背景

本分支没有先前的评审日志，`review-receipts.sh check` 返回 `RECEIPT: NONE`。按全文评审执行。
`design/gdd/run-flow.md` 自写作完成后未再改动（哈希 `02d37ce0…`，与输入 GDD 复审时的快照一致），因此本次是首轮评审，不是增量复审。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。

Formulas 对本系统是**必需**的：它定义了 `run_id` 分配、`run_seed` 选择、结束原因判定与状态转移成本四条数值/判定规则，不是纯流程散文。

Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs。Game Feel 一节实质覆盖了 Player Fantasy；Tuning Knobs 的缺席在 `:149` 有明确理由（四条规则是流程不变量，不是平衡数值），接受。

### 依赖图

上游（全部存在）：
- ✓ scene-flow.md — 存在（In Review）
- ✓ data-config.md — 存在（Approved）
- ✓ input.md — 存在（In Review）

协调的下游（全部存在）：
- ✓ time-and-turns.md、stomach-capacity.md、menu-waves.md、cloying-and-taste.md、settlement-and-entry-fee.md
- ✓ in-game-hud.md、ui-flow.md、first-run-onboarding.md、audio-feedback.md、save-system.md

无断链。**双向引用**核对通过：13 个相关 GDD 中 12 个已在自身的 Dependencies 中回指「餐局流程」；`save-system.md` 无直接调用，不要求回指。

### 注册表值

用 `entities.yaml` 全部 78 个条目名与数值键回扫全文，命中项全部是公式名本身，无第二个数值来源：

- `run_id_alloc`（`entities.yaml:839`）↔ `run-flow.md:153-166`：表达式、`1..INT64_MAX`、不回绕、每会话从 1 开始——逐字一致。
- `run_seed`（`:853`）↔ `run-flow.md:172-186`：`RunConfig.seed` 优先、`0` 合法、否则 `int64(next_u64(session_rng))`——一致。
- `end_reason`（`:868`）↔ `run-flow.md:192-208`：`TIME_EXHAUSTED` 优先、判定时机——一致。
- `transition_turn_cost` / `transition_capacity_cost`（`:1230`、`:1241`）↔ `run-flow.md:212-226`：恒 0——一致。
- `capacity_total` / `can_fit`（`stomach-capacity.md` 拥有，本文件为 referenced_by）：本文件未重述任何容量数值，未冲突。
- `onboarding_active` 使用 `run_state ∈ {Active, Ending}`：本文件的 8 状态枚举包含 `Ending`，一致。

**值冲突：0。**

### Required Before Implementation

1. **[BLOCKING] Detailed Design 规则 6（`:34`）/ Interactions 菜单波次行（`:119`）/ Cross-References（`:368`）——"吃下一道菜"没有调用入口。**
   规则 6 要求提交时"把该菜从当前波移除"，边角案例（`:251`）也以"菜已被吃 → 不在可见集"为前提；但菜单波次的接口表（`menu-waves.md:62`）只声明"传入 seed、回合预算、menu_vision；接收无可吃菜信号"，既没有 `consume(dish_id)` / `eat(dish_id)` 之类的写入调用，也没有本文件所广告的 `advance()`（`menu-waves.md:48` 只说"吃下一道菜 → 从当前波移除该菜"，未给 API）。实现者无法在不臆造接口的情况下完成核心吃菜动作。
   **修复**：在菜单波次行明确写调用名与返回值，例如 `consume(dish_id) → {ADVANCED, NO_EATABLE}`；同时在 `menu-waves.md` 的接口表补同一行（跨文档）。`advance()` 应只承载付费跳波，避免与"吃空自动推进"混淆。

2. **[BLOCKING] "RunConfig 与 RunResult 的归属"字段表（`:59-68`）+ 规则 12 的 `RunLog` 字段（`:40`）——缺 `entry_fee_override` / `RunLog.entry_fee`。**
   首局引导把教学入场费 40 定为 Vertical Slice 的核心教学参数（`first-run-onboarding.md:29,37-40,455`），并明确记录"现有 `RunConfig` schema 没有 `entry_fee_override`"（同文件 `:395`、OQ1 `:501`）。结算从 `get_station()` 读 `entry_fee`（`settlement-and-entry-fee.md:99`），`RunLog` 字段表也没有 `entry_fee`（`:58-66`）——于是 40 元没有任何一条路能到达结算。本节正是本文件回答场景与流程管理 Open Question 的 schema 所有权声明，缺口在本文件。
   **修复**：`RunConfig` 增加 `entry_fee_override: int | null`（`null` = 用 `StationData.entry_fee`，非负校验，与 `capacity_bonus` 的非法值处理一致）；本文件解析后写入 `RunLog.entry_fee`；结算改读 `RunLog.entry_fee`。AC 1/14/28 同步补该字段。这是与结算 GDD 的联合改动。

3. **[BLOCKING] Interactions 数据配置行（`:115`）+ 规则 22（`:50`）——`StationData.turn_budget` 由谁读没写。**
   `time-and-turns.md:33,35,71` 明确要求调用方传入 `initialize(turn_budget)`，并写死"本系统不查询数据目录"。本文件规则 3 只说"初始化时间与回合"，规则 22 又声明只读 `station_id` 是否存在（外加 `pool_override` 用的 `dish_pool` id 集合），AC 1 则写成"收到 `turns_total`"——那是返回值，不是入参。实现者拿不到这个整数的来源。
   **修复**：规则 22 与数据配置行写明"解析 `station_id` 后读取 `StationData.turn_budget`，作为 `initialize(turn_budget)` 的入参传给时间与回合"；AC 1 改为 `initialize(turn_budget)`，并注明返回 `turns_total`。

4. **[BLOCKING] 规则 5（`:33`）+ 局内 HUD 行（`:122`）——动作被拒的返回值未定义，而 HUD 在消费它。**
   `in-game-hud.md:373` 写"跳过或吃菜被餐局流程拒绝……显示一行拒绝原因（来自 `run_flow` 的结构化错误）"，`:164` 也写"HUD 回传 `focused_dish_id` 与跳过/结束取餐意图"。本文件只说"被拒绝的尝试不改变任何状态"，没有定义动作提交的返回类型、拒绝码或拒绝事件，HUD 的拒绝文案因此无源可依，实现者只能靠猜。
   **修复**：定义提交入口与结构化返回，例如 `submit_action(action, dish_id) → COMMITTED | REJECTED(code)`，`code` 至少覆盖校验顺序的四个失败点（`STALE_FOCUS` / `CAPACITY` / `NO_TURNS` / `FROZEN`），并写明 HUD 直接渲染该码对应文案。

5. **[BLOCKING，跨文档] `RunConfig.pool_override`（`:67`）+ 菜单波次行（`:119`）+ AC 29（`:414`）——菜单波次的 `initialize` 尚未接受该参数。**
   `menu-waves.md:62` 的入参只有 `seed`、回合预算、`menu_vision`；首局引导交接记录也把"菜单波次必须接受 `pool_override`"列为待改契约（`first-run-onboarding.md:395`、OQ2）。本文件已经 advertise 了 `initialize(..., [, pool_override])`（`:119`），并以此写 AC 29。修复落点在 `menu-waves.md`（含"菜单健康"校验须对覆盖后的菜池运行），但在它落地前 AC 29 与 Vertical Slice 的首局菜池不可实现。
   **修复**：由编排会话把该参数补进 `menu-waves.md`，或本文件在补丁落地前把 AC 29 标为受该契约阻塞。

### Recommended Revisions

1. **[RECOMMENDED] Cross-References 表（`:357-376`）与 Dependencies 自相矛盾。** 该表把 time-and-turns、stomach-capacity、in-game-hud、ui-flow、first-run-onboarding、audio-feedback、save-system 全部标为"未写"，而 Dependencies（`:290-299`）对同一批系统写的是"GDD 已写"。七份文件都在盘上；按 Dependencies 的写法更新该表。
2. **[RECOMMENDED] Dependencies 状态列陈旧（`:282-284`）。** 数据配置标 "In Design"，索引为 Approved（`systems-index.md:53`）；输入标 "In Design"，索引为 In Review（`:54`）。编排会话统一刷新时一并改。
3. **[RECOMMENDED] "双向一致核对"（`:313-314`）的两条主张已完成，应删除或标为已解决。** 索引第 9/10 行现在都已列出「餐局流程」（`systems-index.md:65-66`），第 5 行也已是 `Designed` + 文档路径（`:61`）。
4. **[RECOMMENDED] Open Question 3（`:445`）已被输入 GDD 回答。** `input.md:35,151` 已把 `game_skip_wave` 定义为"跳过当前波 / 最后一波结束取餐"，并明确不要求悬停。可关闭该 OQ。
5. **[RECOMMENDED] AC 13（`:398`）与 AC 37（`:422`）建立在 `turns_total = 0` 上，而该状态在真实契约下不可达。** `time-and-turns.md:35,166` 规定 `turn_budget < 1` 返回 `INVALID_TURN_BUDGET`，餐局流程应映射为 `init_failed`，并自己称本文件的 0 回合分支"只是防御分支，正常数据走不到"。建议把 AC 13 明确标为"仅以桩测试的防御分支"，AC 37 换一条不依赖 0 回合预算的同帧拒绝路径（例如 AC 38 的初始化失败），或直接改为验证 `INVALID_TURN_BUDGET → Failed`。
6. **[RECOMMENDED] AC 1（`:386`）的参数名与可选参数。** 时间与回合收到的是 `turn_budget`（返回 `turns_total`）；菜单波次签名应带上 `[, pool_override]`，与 `:119` 保持一致。
7. **[RECOMMENDED] 状态转移表的"任何局内状态 → Idle"行（`:103`）范围过宽。** 它读起来覆盖 `Failed`，而信号契约（`:130`）规定 `run_discarded` 对"从未开始的局"不发。建议限定为 `run_started` 已广播的状态。
8. **[RECOMMENDED，跨文档] 结算把 `RunLog.end_reason` 的来源记为"时间与回合 / 胃容量"（`settlement-and-entry-fee.md:62`）。** 胃容量明确"不参与结束判定"（`stomach-capacity.md:44-51`），注册表把 `end_reason` 归本文件（`entities.yaml:868`）。结算 GDD 的来源列应改为「餐局流程」。
9. **[RECOMMENDED，跨文档] 输入 GDD 仍把 `ui_pause` 记在餐局流程名下（`input.md:75,167`）。** 与本文件规则（不读取 `ui_pause`）及 `scene-flow.md:33` 冲突；本文件 OQ2 已跟踪，等输入侧修复后再关闭。
10. **[RECOMMENDED，跨文档] 菜单波次对 `menu_vision` 的描述混用了"总预览额度"与"整数等级"。** `menu-waves.md:110` 与 `meta-upgrades.md:21,102,204` 定义为已解析额度，`menu-waves.md:64,227` 又写成"只消费整数等级"。本文件的写法（已解析值，不是等级）是对的，建议菜单波次统一措辞。
11. **[RECOMMENDED] 信号 `action_committed(action, payload)` 的 `payload` schema 未定义。** 音效与 HUD 目前只订阅事件，但实现者会需要知道里面有什么（至少 `dish_id` / `turns_used` / `capacity_used` 之一）。补一个字段表或明确"payload 由订阅方约定"。

### Nice-to-Have

- `entities.yaml` 的 `first_run_pool` 把 `run-flow.md` 列进 `referenced_by`，但本文件从未引用那六道菜（只做子集校验）。可从该条目的 referenced_by 移除，或在本文件显式引用。
- 菜单波次的信号在本文件叫 `no_eatable`，在 `menu-waves.md` 里只有中文名"无可吃菜"。若实现期需要单一标识符，建议让菜单波次采纳该名称。
- 可考虑为 B1 补一条 AC："吃下一道菜后 `visible_dishes` 少一道；吃空当前波自动推进且 `turns_used` 不变"——现有 AC 6/10 只覆盖计数，不覆盖可见集收缩。

### Specialist Disagreements

无。lean 模式未派发 specialist，Phase 3b 未运行。

### Senior Verdict

lean 模式不派发 `creative-director`。主评审意见即最终意见：设计本身站得住，五条 blocking 全部是接口契约的缺行，不需要重写任何规则。

### Scope Signal

**Rough scope signal: L（producer 应在 sprint planning 前复核）** — Core 系统，4 条数值/判定规则，13 条上下游依赖，且正文末尾明确列出需要 ADR 落地的实现问题（状态机载体、随机源、`RunLog` 不可变构造、故障返回通道）。

### Verdict: NEEDS REVISION

Reviewed-Content-Hash: design/gdd/run-flow.md 02d37ce06fa293e6b76cb22277a68b705e413448
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/input.md dd15ea96523a49136b96cfef827c5619dee75e6a
Reviewed-Content-Hash: design/gdd/time-and-turns.md 7f55a619613d6c2ff9f99b9458e39d0f5f48895c
Reviewed-Content-Hash: design/gdd/stomach-capacity.md bb97f41a121c6c18d15f7d10c23387ff25688b11
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 0f58ea62a2e8fa9ace2effe652cc644acab89d5b
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/systems-index.md 8f9bdb61bf682a7e4d7191933202f9efb3d68cde

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 0 | Recommended: 14（上轮遗留 9 + 本次新发现 5）
Summary: `ccdd0d7` 把上轮 5 项 blocking 全部补齐，且每项都落到了两侧契约——run-flow 的规则/接口表与 menu-waves、settlement 的对应行现在互相对得上（`consume(dish_id)`、`RunLog.entry_fee`、`turn_budget` 入参、`submit_action` 拒绝码、`pool_override` 在覆盖池上跑健康校验）。设计本身未改动，仍是同一套状态机与边界；剩余 14 条全部是文档同步与防御分支标注，不阻塞实现。
Prior verdict resolved: Yes — 上轮 5 项 blocking 全部已解决。

### 复审背景

- 触发：`ccdd0d7`（Batch 2 修复）改动 `run-flow.md`（哈希 `02d37ce` → `e1f10d2`）及其 6 份关联 GDD。
- 复审范围：**全文重读**（56 KB，按标题分段），未按"只读改动段"跳读。
- 改动内容与本文件 27 行 diff 全部为契约补行，未重写任何规则、公式、边角案例或验收判据。

### 完整性与依赖

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统必需（4 条数值/判定规则）。

Advisory 缺口（不阻塞）：Player Fantasy（Game Feel 实质覆盖）、Tuning Knobs（`:150` 已说明四条规则是流程不变量，不是平衡数值）。与上轮一致。

依赖图：14 份被引 GDD 全部在盘上（scene-flow、data-config、input、time-and-turns、stomach-capacity、menu-waves、cloying-and-taste、settlement-and-entry-fee、in-game-hud、ui-flow、first-run-onboarding、audio-feedback、save-system、dish-values）。无断链。

### 注册表值

本系统自有 5 条逐字一致：`run_id_alloc`、`run_seed`、`end_reason`、`transition_turn_cost`、`transition_capacity_cost`。

`ccdd0d7` 对 `entities.yaml` 的改动只涉及 `save_load_decision`（save 系统），未触碰本系统条目。另回扫 `station_canteen_entry_fee = 85`、`station_canteen_turn_budget = 10`、`first_run_entry_fee = 40`、`first_run_pool`（6 道）——本文件**不重述任何一个数值**，只按引用转交，无冲突。

### 上轮 blocking 复核

| # | 上轮 blocking | 结论 | 证据 |
|---|---|---|---|
| B1 | 吃菜没有调用入口 | **已解决** | run-flow `:34` 规则 6、`:120` 接口表、`:293` 依赖表三处均写 `consume(dish_id) → {ADVANCED, NO_EATABLE}`，并写明 `advance()` 只承载付费跳波；menu-waves `:62` 同步该签名，`:69` 明确 `consume` 负责吃空自动推进。 |
| B2 | 缺 `entry_fee_override` / `RunLog.entry_fee` | **已解决** | run-flow `:64` 字段（`int \| null`，`null` = `StationData.entry_fee`，≥ 0）、`:40` `RunLog` 含 `entry_fee`、AC 1/14/28（`:387`、`:400`、`:414`）；settlement `:64` 输入表改由餐局流程提供，`:95` 明确改从 `RunLog.entry_fee` 读取。教学价 40 现在有完整通路。 |
| B3 | `StationData.turn_budget` 读取方未声明 | **已解决** | run-flow `:50` 规则 22 写明解析后读 `turn_budget` 并作为 `initialize(turn_budget)` 入参；`:116` 数据配置行、`:118`/`:291` 时间与回合行同步；AC 1 改为 `turn_budget`（返回 `turns_total`）。 |
| B4 | 动作拒绝返回值未定义 | **已解决** | `:33` 规则 5 定义 `submit_action(action, dish_id) → COMMITTED \| REJECTED(code)`，码覆盖 `STALE_FOCUS` / `CAPACITY` / `NO_TURNS` / `FROZEN`，并写明 HUD 直接渲染；`:123` HUD 行同步消费该码。 |
| B5 | 菜单波次 `initialize` 未接受 `pool_override` | **已解决** | menu-waves `:62` 已接受 `[, pool_override]`，`:69` 明确"波次划分与健康校验都在覆盖后的菜池上运行"；run-flow `:68`/`:120`/AC 29 与之一致。 |

### Required Before Implementation

- none

### Recommended Revisions

**上轮遗留（9）**

1. **[RECOMMENDED] Cross-References 表（`:358-376`）仍写"未写"**：time-and-turns、stomach-capacity、in-game-hud、ui-flow、first-run-onboarding、audio-feedback、save-system 七份文件均在盘上，而 Dependencies（`:283-300`）对同一批写"GDD 已写"。按 Dependencies 口径刷新。
2. **[RECOMMENDED] Dependencies 状态列陈旧（`:284-285`）**：数据配置标 "In Design"（索引 `:57` 为 Approved），输入标 "In Design"（索引 `:58` 为 In Review）。
3. **[RECOMMENDED] "双向一致核对"（`:314-315`）的两条主张已完成，应删除或标为已解决**：索引 `:65-66` 已列出「餐局流程」，`:61` 已是 In Review + 文档路径。
4. **[RECOMMENDED] Open Question 3（`:446`）已被输入 GDD 回答**：input.md `:151` 明确 `game_skip_wave` 为"跳过当前波 / 最后一波结束取餐"且不要求悬停。可关闭。
5. **[RECOMMENDED] AC 13（`:399`）与 AC 37（`:423`）依赖 `turns_total = 0`**：time-and-turns 对 `turn_budget < 1` 返回 `INVALID_TURN_BUDGET`，真实契约下不可达。建议 AC 13 标为"仅防御分支"，AC 37 改用初始化失败路径（AC 38 已覆盖）验证同帧拒绝。
6. **[RECOMMENDED] 状态转移"任何局内状态 → Idle"（`:104`）范围过宽**：覆盖 `Initializing` / `Failed` 时与信号契约（`:131` "`run_discarded` 在从未开始的局上不发"）冲突。限定为 `run_started` 已广播后的状态。
7. **[RECOMMENDED] `action_committed(action, payload)`（`:131`）payload schema 未定义**：音效与 HUD 实现需要知道至少含哪些字段（`dish_id` / `turns_used` / `capacity_used`）。
8. **[RECOMMENDED，跨文档] input.md `:75` 仍把 `ui_pause` 记在本系统读取项下**，与本文件规则 5/15 冲突；本文件 OQ2 跟踪中，待输入侧修正后关闭。
9. **[RECOMMENDED，跨文档] menu-waves.md `:64` 仍写"本系统只消费整数等级"，`:110` 写"总预览额度"**；本文件（`:65`）与 `meta-upgrades` 的口径是已解析值。建议菜单波次统一措辞。

**本次新发现（5）**

10. **[RECOMMENDED] `entry_fee_override` 为负时没有失败路径**：schema（`:64`）要求 ≥ 0，但规则 4（`:32`）的失败枚举与 Edge Cases（`:241-276`）都没有这一项（`upgrade_snapshot` 的负值有，`entry_fee_override` 的没有）。补"`entry_fee_override < 0` → `init_failed`"。
11. **[RECOMMENDED] `StationData.entry_fee` 的读取方未写进规则 22 与接口表**：AC 1（`:387`）规定 `entry_fee_override = null` 时取 `StationData.entry_fee`，因此本系统确实要读它；但规则 22（`:50`）只列 `turn_budget` 与 `dish_pool`，`:116` 数据配置行也只列 `turn_budget`。补一句，与 B3 同样的处理方式。
12. **[RECOMMENDED] `REJECTED(code)` 缺"非 `Active`"码**：`STALE_FOCUS` / `CAPACITY` / `NO_TURNS` / `FROZEN` 覆盖不到进入 `Ending` 后仍在队列里的动作（规则 9 说此后不再接受）。UI 要求 2（`:347`）会禁用控件使其基本不可达，但 HUD 的拒绝文案仍无码可渲染。另：`submit_action` 的 `action` 枚举与跳波/结束取餐时 `dish_id` 的可空性未定义。
13. **[RECOMMENDED] 过时表述**：Edge Case `:272` 仍写"若菜单波次当前只读 `StationData.dish_pool`，该字段暂不可用"，OQ5（`:448`）同样按"无处落地"描述——menu-waves `:62,69` 落地后两句均已过时，OQ5 可关闭。
14. **[RECOMMENDED，跨文档] settlement-and-entry-fee.md 的公式仍以 `entry_fee(station_id)` / `StationData.entry_fee` 表达**（`:137`、`:143`），与同一份 GDD 的输入表（`:64`）和数据配置行（`:95`）已改为读 `RunLog.entry_fee` 相矛盾；`:99` 仍把 `end_reason` 归"时间与回合"。本次修复只改了两侧的接口表，公式正文未跟上。

### Nice-to-Have

- `entities.yaml` 的 `first_run_pool` 把 `run-flow.md` 列进 `referenced_by`，但本文件只做子集校验、不引用那六道菜；可从该条目移除或在本文件显式引用。
- 补一条 AC：吃空当前波自动推进**不额外**扣回合、`visible_dishes` 收缩一道（现有 AC 6/10 只覆盖计数，不覆盖可见集收缩与"自动推进免费"）。
- 菜单波次的信号在本文件叫 `no_eatable`，menu-waves 里只有中文名"无可吃菜"；实现期建议统一标识符。

### Specialist Disagreements

无。lean 模式未派发 specialist，Phase 3b 未运行。

### Senior Verdict

lean 模式不派发 `creative-director`。主评审意见即最终意见：**上轮 5 项 blocking 全部解决且两侧契约自洽**，本次未发现新的设计缺陷；14 条 recommended 中 9 条是同一批交叉引用台账的陈旧状态（三轮未清），5 条是新增字段带来的同步遗漏，均不影响程序员按现行文本实现。

### Scope Signal

**Rough scope signal: L（producer 应在 sprint planning 前复核）** — Core 系统，4 条数值/判定规则，14 条上下游依赖，正文末尾列出的 6 项实现问题需要 ADR。

### Verdict: APPROVED

Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/input.md dd15ea96523a49136b96cfef827c5619dee75e6a
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 94da89a242967c44827406c4da5714d088df6394
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 2 | Recommended: 13
Summary: 新增的规则 23、规则 24 与 `fault.phase` / 「`Ending`/`Settling`/`Handoff` 退出即弃」条款本身与 ADR-0007 D1/D3/D5/D7、ADR-0015 §8、`stomach-capacity` 规则 2 一致：六项指定契约中四项完全通过、一项（规则 7）门控结论正确但理由句自相矛盾、一项（规则 3 初始化顺序）记为 `TR-run-flow-002` 的架构 partial 而非设计阻塞。两项 blocking 都出自新增内容与本文旧口径的正面冲突：规则 23 让本系统开始消费 `entry_fee` 与 `price(d)`，而规则 1 / 规则 22 / 依赖章 / 交叉引用四处仍写"不读取任何价格"，且没有声明任何数值来源；本系统始终没有发布 ADR-0007 D1 与 ADR-0013 §9 指定的 `action_availability()`，导致 `in-game-hud` 只能自行比较 `turns_remaining ≥ 2`，违反"不得复制门控阈值"。
Prior verdict resolved: N/A — 上轮 APPROVED（2026-11-04，`e1f10d2`）的收据已失效，本轮按全量 lean 复审重判。

### 复审背景（失效收据与新增内容）

`review-receipts.sh check design/gdd/reviews/run-flow-review-log.md design/gdd/run-flow.md` 输出：`CHANGED: design/gdd/run-flow.md`、`CHANGED: design/registry/entities.yaml`、`CHANGED:` 14 份关联 GDD 与 `systems-index.md`，仅 `UNCHANGED: .claude/skills/design-review/SKILL.md`。上轮 APPROVED 绑定的哈希是 `e1f10d2`，当前字节为 `56f6f48`，之后 `cf14f37` / `be69169` / `03f8a92` / `1b41d84` 等提交继续改过本文件——收据按 `.claude/rules/design-docs.md`「Review receipts and post-approval edits」判为失效，本轮的输入是当前字节的全量重读（60 KB，按标题分段）。

按"从未被独立审过"处理的新增内容：规则 23（`:51`）、规则 24（`:52`）、规则 16 的 `Ending`/`Settling`/`Handoff` 覆盖段（`:44`）、规则 18 的 `fault.phase`（`:46`）、转移表 `run_id` 溢出与 `fault.phase = INIT` 行（`:107`）、快照 `running_net` / `shortfall` 行与"唯一副本"段（`:146`、`:150`）、HUD 行与 UI 要求 1/9（`:125`、`:350`、`:358`）、边角案例 `:262` / `:268` / `:271` / `:274`、AC 11a（`:402`）、AC 34（`:425`）、AC 41（`:432`）、OQ1 的回答（`:449`）。

### 指定契约核对（以 ADR 为准）

| # | 契约 | 结论 | 依据 |
|---|---|---|---|
| 1 | 规则 23：`running_net` 唯一运行期累加器归本系统；结算只定义纯函数；HUD 读快照或纯函数 | **通过（所有权）** | ADR-0007 D1 `:108` 把 `running_net` / `shortfall` 列为 `RunFlow` 所有并由 `snapshot()` 发布；ADR-0015 §8 `:214` 明写"运行期累加器仍归 `RunFlow`"、结算只做前缀求和并在结算时复核；`settlement-and-entry-fee` 规则 19（`:43`）、`:263` 与 `in-game-hud` 规则 6（`:39`）、`:117-118`、`:170` 同口径；`shortfall = max(0, −running_net)` 与 `shortfall(k)` 代数等价。**但数值来源未声明，见 B1。** |
| 2 | 规则 24：`end_run(reason)` 覆盖每条终止路径，`reason` 四值含从未开始的 `Failed` | **通过** | `stomach-capacity` 规则 2（`:28`）逐字要求同一组四值与同一枚举，依赖行（`:62`、`:203`）同步；ADR-0007 D7 `:271` 按 `phase` 给出 `INIT → failed` / `RUN → faulted` 的映射。缺口见 R2 / R3。 |
| 3 | 规则 18 / `scene-flow` 规则 6：`fault.phase ∈ {INIT, RUN}` 是文案判据，`init_failed` 只表示"无 `RunResult`" | **通过** | ADR-0007 D7 `:258-275`（`run_started` 是唯一分界、`init_failed` 只是兼容位）；`scene-flow.md:6` 与 `ui-flow.md:81`、`:144` 均按 `phase` 分支、`phase` 缺失按 `INIT` 兜底。缺口见 R4。 |
| 4 | 规则 16：`Ending`/`Settling`/`Handoff` 经暂停菜单退出即弃，`run_settled` 只是预通知、从不落盘 | **通过** | ADR-0007 D3 `:162`（"`run_settled` 不是提交点"，理由正是规则 16 允许在 `Handoff` 退出）、`:163`（被拒时留在 `Handoff`，无任何持久化）、D5 `:230`；`ui-flow.md:73`、`:81` 已实现按 `run_state` 隐藏「放弃本局」并按 `abandoned` 提示。缺判据见 R7。 |
| 5 | 规则 7：`turns_remaining = 1` 拒绝付费跳波 | **结论通过，理由句不成立** | 门控结果与 ADR-0013 §4 `:155`、`:157` 的 `turns_remaining ≥ 2` 一致，也与 `time-and-turns` 规则 13（`:53`）/ AC 23（`:292`）、`menu-waves.md:31`、AC 11a（`:402`）一致；见 R1。 |
| 6 | 规则 3 的初始化顺序 | **不与任何 Accepted ADR 冲突 → 记为跨文档说明，不阻塞** | ADR-0013 Ordering Note 把顺序显式让给 ADR-0007，而 ADR-0007 只决定失败半（D7），`requirements-traceability.md:78` 与 `:168-182` 因此把 `TR-run-flow-002` 记为 PARTIAL。这是架构缺口（需 ADR-0007 补一句固定四次 `initialize()` 顺序），不是本 GDD 的设计缺陷；本文件规则 3 与各子系统的 `initialize` 签名（`time-and-turns:73`、`stomach-capacity:62`、`menu-waves:63`、`cloying` `cloying_resistance`）逐一对得上。 |

### 完整性

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统必需（四条数值/判定规则），已含变量表、范围与算例。

Advisory 缺口（不阻塞）：Player Fantasy 缺失（Game Feel 实质覆盖）；Tuning Knobs 缺失（`:154` 已声明四条规则是流程不变量而非平衡数值）。

### 依赖图

14 份被引 GDD 全部在盘上（scene-flow、data-config、input、time-and-turns、stomach-capacity、menu-waves、cloying-and-taste、settlement-and-entry-fee、in-game-hud、ui-flow、first-run-onboarding、audio-feedback、save-system、dish-values），无断链。依赖**边**的语义问题见 B1 与 R3。

### 注册表值

本系统自有 5 条逐字一致：`run_id_alloc`、`run_seed`、`end_reason`、`transition_turn_cost`、`transition_capacity_cost`（`entities.yaml:857`、`:871`、`:886`、`:1253`、`:1264`）。

全文档回扫 `running_net|shortfall|entry_fee|turn_budget|menu_vision|cloying_resistance|first_run_pool` 及 `85` / `40` / `10`：本文件不重述任何平衡数值，只按引用转交，无数值冲突。唯一登记问题是 `running_net` / `shortfall` 的 `referenced_by` 未随规则 23 更新（见 R12）。

### Required Before Implementation

| # | 位置 | 问题 | Authority |
|---|---|---|---|
| B1 | `design/gdd/run-flow.md:51`（规则 23）对 `:50`（规则 22）、`:29`（规则 1）、`:311`（"不属于依赖 · 菜品数值"）、`:371`（Cross-References 菜品数值行）、`design/gdd/data-config.md:82` | 规则 23 要求本系统以 `−entry_fee` 初始化累加器、每次吃菜 `+price(d)`，但同一份文件四处写着"不读取任何菜品价格或饱腹""所有数字来自各子系统"，依赖章明确把菜品数值列为"不属于依赖"；`entry_fee` 的解析归属同样无着落（`data-config.md:82`、`:192` 已把 `entry_fee` 归给结算），而 AC 1（`:391`）与 `RunLog.entry_fee`（`:40`）又要求本系统在 `entry_fee_override = null` 时取 `StationData.entry_fee`。规则 23 因此无法按现行文本实现：既不声明 `price(d)` / `entry_fee` 的来源（`dish-values.md:278-279` 的消费者只有结算与 HUD，没有餐局流程），又与自己的规则 22 正面矛盾 | ADR-0015 §8（`adr-0015:214`）要求该累加器存在，故输入必须被声明；`.claude/rules/design-docs.md:14`（依赖必须双向）；本文件规则 1 / 22 |
| B2 | `design/gdd/run-flow.md:125`（HUD 行）、`:135-150`（只读快照表）、`:131`（接口所有权） | 本系统始终没有发布 `action_availability()`：快照表只有 13 个字段，HUD 行只承诺"渲染 `REJECTED(code)` 文案"。ADR-0007 D1（`:111`）把它指定为 `RunFlow` 的只读投影、HUD 的**唯一入口**，并明令 HUD 不得自行拼 `turns_remaining ≥ 2` 或 `is_final_wave`；ADR-0013 §9（`:221-238`）同文。缺了这个契约，`in-game-hud.md:372` 与 AC 26（`:503`）只能自己比较 `turns_remaining ≤ 1` 来置灰按钮——正是两条 Accepted ADR 禁止的复制；`requirements-traceability.md:83` 却已把 `TR-run-flow-007` 记为 covered 并援引该接口 | ADR-0007 D1（`adr-0007:111`）、ADR-0013 §9（`adr-0013:221-238`）、ADR-0019 D3（`adr-0019:832`：未实现前 HUD 不得复制阈值） |

### Recommended Revisions

1. **[RECOMMENDED] 规则 7 的理由句自相矛盾（`:35`）**："付费跳波需要至少 1 个可用回合来支付它自己"与紧接着的"`turns_remaining = 1` 时第二次校验判定为不可用"直接打架。ADR-0013 §4（`:155`、`:157`）的真实条件是 `turns_remaining ≥ 2`——支付自己**并**在新波留 1 回合吃菜；门控结论本身正确，改这句理由即可。
2. **[RECOMMENDED] 规则 24 的 `reason` 应按 `fault.phase` 而不是症状/状态名（`:52` 对 `:107`）**：`run_id` 溢出的状态名是 `Faulted`、`phase` 是 `INIT`，ADR-0007 D7（`:271`）要求 `Capacity.end_run(failed)`；现行措辞按"初始化失败 / 运行期故障"分组，实现者极易按状态名给成 `faulted`。写一句"`reason` 由 `fault.phase` 决定：`INIT → failed`，`RUN → faulted`"。
3. **[RECOMMENDED] 规则 24 缺 AC、也缺依赖表行**：`:297`（胃容量下游行）没有 `end_run(reason)`，而 `:121`（接口表）与 `stomach-capacity:62` / `:203` 都有；四条终止路径（`ended` / `discarded` / `failed` / `faulted`）各补一条可断言判据（调用次数 = 1、参数正确、下一次 `initialize` 不被判为同局第二次）。
4. **[RECOMMENDED] 规则 18 的 `fault.phase` 没有任何判据**：AC 4 / 5 / 30 / 35（`:394`、`:395`、`:421`、`:426`）只断言 `init_failed`，AC 32（`:423`）断言 `fault` 明细却不含 `phase = RUN`。`ui-flow.md:81` 完全靠该字段选文案，应至少各补一条。
5. **[RECOMMENDED] `run_faulted` 的发放时机未写**：ADR-0007 D7（`:266`）要求 `INIT` 路径也发一次（`run_id = 0`、`code = RUN_ID_OVERFLOW`，`:279`），ADR-0007 的 Requirements（`:97`）点名本 GDD 应说明 `run_faulted` 与"进入失败"的关系；规则 4（`:32`）、状态表（`:88-89`）与信号段（`:133`）只承诺"至多一次"，AC 33（`:424`）不覆盖该信号，而 `audio-feedback:77` 依赖它起 `sfx.run.fault`。
6. **[RECOMMENDED] `StationData.entry_fee` 的读取方仍缺位（上轮 #11 未修）**：AC 1（`:391`）要求 `entry_fee_override = null` 时读它，但规则 22（`:50`）只列 `turn_budget` 与 `dish_pool`，`:118` 数据配置行同样不提。与 B1 的累加器初值同一缺口，建议一并解决。
7. **[RECOMMENDED] 规则 16 的新增覆盖段（`:44`）无判据**：AC 22 / 23（`:413`、`:414`）只覆盖 `Active` 的放弃与退出，AC 41（`:432`）只测暂停菜单项。补一条：`Handoff` 中经暂停菜单退出 → 不交付 `RunResult`、不出收据、不写纪录、广播 `run_discarded` 一次、`settle` 未被二次调用。
8. **[RECOMMENDED] 拒绝码与动作载荷仍未闭合（上轮 #7 / #12）**：`REJECTED(code)` 仍缺 `NOT_ACTIVE`（ADR-0013 §2 `:116` 与 §7 `:187` 都把 `state != ACTIVE` 定为该码）与 `NOT_FINAL_WAVE`；`action_committed(action, payload)`（`:133`）的 payload 仍未定义字段，音效与 HUD 无法按契约实现。
9. **[RECOMMENDED] Formulas 前言与规则 23 冲突（`:154`）**："本系统只定义四个数值/判定规则"现在不成立——`shortfall = max(0, −running_net)` 是第五条数值关系，却没有变量表 / 范围 / 算例，也没进 `:235` 的不变式表。二选一：明确写"该式由结算纯函数定义，本系统只维护状态"，或补一行不变式（`running_net = Σprice(eaten) − entry_fee`，违反 → `Faulted`）。
10. **[RECOMMENDED] 状态转移 `:106` 的范围仍过宽（上轮 #6）**："任何局内状态 → `Idle`"覆盖 `Initializing` / `Failed`，与 `:133`"`run_discarded` 在从未开始的局上不发"以及 ADR-0007 D7（`:267`）冲突；限定为 `run_started` 已发出之后的局内状态。
11. **[RECOMMENDED] 陈旧台账**：表头 `Last Updated: 2026-11-03`（`:5`）早于规则 23/24；`:318` 断言索引 `Status = Approved`（本判决已使其失效）；边角案例 `:277` 与 OQ5（`:453`）仍称菜单波次只读 `StationData.dish_pool`（`menu-waves:62`、`:70` 已接受 `pool_override`）；OQ3（`:451`）的动作命名已在 `input.md:35` 统一；OQ10（`:458`）已被 `data-config.md:82` / `:192` 回答（但其结论与 AC 1 相抵，见 B1 / R6）；OQ4（`:452`）可改写：`scene-flow.md:8` 现以 `abandoned = true` 显式交付，本系统不必只靠观察顶层状态。
12. **[RECOMMENDED] 注册表 `running_net` / `shortfall` 未随规则 23 更新**（`entities.yaml:684`、`:699`）：`referenced_by` 仍只有结算，notes 仍写"HUD must call this instead of summing prices"，而规则 23（`:51`）已把运行期来源定为本系统快照、纯函数仅作复核；补 `design/gdd/run-flow.md` 引用并改 notes（无数值冲突）。
13. **[RECOMMENDED，跨文档] `settlement-and-entry-fee.md:99` 仍把 `end_reason` 归在"时间与回合"名下**：`time-and-turns` 规则 8（`:43`）明写不保存、不排序 `end_reason`，本文件规则 8 / 10 与 `RunLog`（`:40`）都把它归本系统；该行应改为"餐局流程经 `RunLog` 提供"。

### Nice-to-Have

- AC 13（`:404`）与 AC 37（`:428`）依赖 `turns_total = 0`，而 `time-and-turns` 对 `turn_budget < 1` 返回 `INVALID_TURN_BUDGET`：AC 13 可沿用 ADR-0013 `:185` 的"防御分支、正常数据不可达"措辞，AC 37 可改用初始化失败路径（AC 38 已覆盖同帧拒绝）。
- 补一条 AC：吃空当前波自动推进**不额外**扣回合、`visible_dishes` 收缩一道（现有 AC 6 / 10 只覆盖计数）。
- `first_run_pool` 的 `referenced_by` 含本文件，但本文件只做子集校验、不引用那六道菜。

### 判定说明

lean 模式不派发 `creative-director`，主评审意见即最终意见。核心机制（状态机、动作仲裁、结束判定、交接与「放弃不是结算」）与四份 Accepted ADR 自洽，新增规则 24 与「终止阶段退出即弃」条款是正确的跨系统收口；两项 blocking 都是新增内容把旧口径留在原处造成的契约断裂——规则 23 需要"我确实读价格与入场费"的显式声明与依赖边，`action_availability()` 需要进入快照/HUD 契约，二者都属于实现前必须闭合的范围。

### Scope Signal

**Rough scope signal: S（给定）** — 单系统、四条流程不变量、无新增 ADR；但本判决的两项 blocking 会牵动 `dish-values` / `data-config` / `in-game-hud` 三处依赖行。

### Verdict: NEEDS REVISION

Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/registry/entities.yaml f7b5aded130f371160d84bdaf678f8530ddc0807
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/scene-flow.md 07753accfc002be7257cbcdd17323c73c2b6845a
Reviewed-Content-Hash: design/gdd/data-config.md 9ecc2a7c18aba33c64bc3391594748e1063851d2
Reviewed-Content-Hash: design/gdd/input.md 27c9d885eb2ba1f8d4b3df72ff6640fed0a5dd36
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/gdd/menu-waves.md 4282c9a66552dbb5023418b9c19542c1b5787af3
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/in-game-hud.md fee7199048c3a124fcab2eccae8b88f93d1a69c0
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: docs/architecture/requirements-traceability.md efc28c613de2b73657961d3829f460307d5dbe92
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 401cac388aed858afdc60a3a456e086bfc7310f0
Reviewed-Content-Hash: docs/architecture/adr-0015-settlement-pure-functions.md cecf7e42a64c4edf19bdb3a50fc7b8029ca743eb
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 23304c8212ab59cede6f6f3d81cd40cea079cde5

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 4 | Recommended: 10
Summary: 上轮两项 blocking 的上半场已实质修复——`entry_fee` / `price(d)` 的来源（规则 22/23 + 数据配置行）、`action_availability()` 的发布（规则 5 + 快照表 + HUD 行）、跳波门控 `turns_remaining ≥ 2`、`fault.phase` 驱动 `end_run(reason)`、AC 42–45 都写对了。但这一轮又出现同一类缺陷：改动只落在被点名的那几处，同一个事实的第二、第三处副本仍是旧口径——`end_reason` 的公式与注册表表达式没有 `_no_legal_action` 项，Dependencies 上游表没有「菜品数值」这一行（而 `:317` 声称「已在依赖表列出」），边角案例 `:262` 仍写「回合未耗尽即不结束」，OQ10 仍写「本系统只解析 `station_id`」。四项都可直接指到具体句子，因此判 NEEDS REVISION。
Prior verdict resolved: 部分 — 上轮 B1（`entry_fee` / `price(d)` 来源）与 B2（缺 `action_availability()`）已修；上轮 R1、R2、R4、R6、R7、R9、R10、R12 已修；R3（胃容量行缺 `end_run`）、R8（payload 未定义）、R11（陈旧台账）未修，转为本轮 R1 / R4 / R8–R10。

### 复审背景（全量重读，非增量）

`review-receipts.sh check` 输出 `CHANGED: design/gdd/run-flow.md`、`CHANGED: design/registry/entities.yaml`，以及 15 份关联文件 `CHANGED`，仅本 skill 未变。按 `.claude/rules/design-docs.md`「Review receipts and post-approval edits」，上轮 2026-11-06 的 NEEDS REVISION 判决不构成跳过依据；本轮按 66 KB 全文分段重读（Rules 1–24、状态表与转移表、交互表、快照表、四条公式 + 不变式表、37 条边角案例、依赖章、AC 1–45、OQ 1–11），并逐条核对 ADR-0013 §1–§9（含 2026-11-06 `_no_legal_action()` 补入段）、ADR-0007 D1/D7、ADR-0015 §8、ADR-0019 D3、ADR-0006 §4–§6。

### 完整性

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统必需（四条判定/分配规则 + 一条净值恒等式），变量表、范围与算例齐全。

Advisory 缺口（不阻塞）：Player Fantasy 缺失（Game Feel `:339-348` 实质覆盖）；Tuning Knobs 缺失（`:156` 已声明这些是流程不变量而非平衡数值）。AC 编号 1–45 加 11a 连续无重号，AC 42–45 均可由探针或替身独立断言。

### 依赖图

14 份被引 GDD 全部在盘上（scene-flow、data-config、input、time-and-turns、stomach-capacity、menu-waves、cloying-and-taste、settlement-and-entry-fee、in-game-hud、ui-flow、first-run-onboarding、audio-feedback、save-system、dish-values），无断链。**回指不完整**：`dish-values.md` 全文 0 次出现「餐局流程」，其下游表（`dish-values.md:272-280`）只列腻度、菜单波次、结算、HUD、首局引导；而本文件 `:119`、`:317`、`:375` 已把菜品数值列为硬上游。依赖**边**的登记问题见 B2。

### 注册表值

本系统自有的 5 条与注册表逐字一致：`run_id_alloc`（`entities.yaml:861`）、`run_seed`（`:875`）、`end_reason`（`:890`）、`transition_turn_cost`（`:1257`）、`transition_capacity_cost`（`:1268`）。`running_net`（`:686`）/ `shortfall`（`:703`）的 `referenced_by` 已补 `run-flow`、notes 已改为「run-flow owns this single live accumulator」，与规则 23/不变式 `:244` 一致；上轮 R12 已关闭。全文档按注册表实体与数值键回扫（`running_net|shortfall|entry_fee|run_id|run_seed|end_reason|transition_*|capacity_total|can_fit|turn_budget|menu_vision|price|satiety|opt_value|unit_value`）命中 100 行，无第二套数值（本文件不重述 85 / 104 / 40 / 10）。唯一冲突是 `end_reason`：注册表表达式与 `variables` 同样没有 `_no_legal_action` 项（见 B1，属台账侧待改，本文件不得自行改注册表）。

### Required Before Implementation

| # | 位置 | 问题 | Authority |
|---|---|---|---|
| B1 | `design/gdd/run-flow.md:201`（`end_reason` 公式）与 `:208`（`no_eatable` 变量说明）；`design/registry/entities.yaml:896-899` | 公式仍是 `TIME_EXHAUSTED if turns_remaining = 0 else (NO_EATABLE_DISH if no_eatable else NONE)`，变量表把 `no_eatable` 限定为菜单波次信号。规则 8（`:36`）与 AC 43（`:438`）却要求「非最后一波 ∧ `turns_remaining = 1` ∧ 无菜 `can_fit` → `NO_EATABLE_DISH`」且`no_eatable` 保持 0 次。按写下的公式该状态只能产出 `NONE`，AC 43 永远红。ADR-0013 §6（`adr-0013:177-185`）的函数里有第三个分支 `if _no_legal_action(): return NO_EATABLE_DISH`——公式必须补该项（变量表加 `legal_action_available` 或 `_no_legal_action()` 状态判据），注册表 `end_reason` 同步（含 `revised`） | ADR-0013 §6（Accepted，2026-11-06 补入）；本文件规则 8 / AC 43；`.claude/rules/design-docs.md:14` |
| B2 | `design/gdd/run-flow.md:289-293`（上游依赖表）对 `:317`、`:119`、`:375`；`design/gdd/systems-index.md:62`；`design/gdd/dish-values.md:272-280` | 规则 22/23 让本系统在运行期读 `entry_fee` 与 `price(d)`，`:119` 与 `:375` 已把「菜品数值」列为硬上游，`:317` 更明写「这条边已在依赖表列出」——但 Dependencies 的上游表只有场景与流程管理、数据配置、输入三行，下游表也没有它。于是「已列出」是假陈述，实现者按依赖表不会给 `price` 注入只读入口。同时索引第 5 行 `Depends On = 场景与流程管理, 数据配置, 输入` 缺该边，`dish-values.md` 下游表也没有回指餐局流程——三条边只有一条被登记，双向要求不成立 | `.claude/rules/design-docs.md:14`；本文件规则 22/23；ADR-0015 §8（`adr-0015:225`） |
| B3 | `design/gdd/run-flow.md:262`（边角案例「容量已满但仍有可见菜且回合未耗尽」） | 写着「本局**不**结束；`turns_remaining ≥ 2` 时玩家可以付费跳波，最终由回合耗尽或无可吃菜结束」。`turns_remaining = 1` 包含在「回合未耗尽」里，而规则 8（`:36`）、边角案例 `:264`、AC 43（`:438`）与 `time-and-turns:178`、`menu-waves:226`、`stomach-capacity:192` 都规定该状态同帧以 `NO_EATABLE_DISH` 结束；「最终只由回合耗尽或无可吃菜结束」也漏掉这条无信号结束路径。AC 27 / 39（`:422`、`:434`）已收窄到 `≥ 2`，这句没跟着改 | 本文件规则 8 / AC 43；ADR-0013 §6 `:210-211` |
| B4 | `design/gdd/run-flow.md:466`（OQ 10） | 仍写「数据配置把 `entry_fee` 的读取记在餐局流程名下，但本设计由结算读取；**本系统只解析 `station_id`**。是否修正 `data-config.md` 的下游行？」——与规则 22（`:50`「解析成功后从 `StationData` 读取 … `entry_fee`」）、`:323`（「`entry_fee` 的解析归本系统 … `data-config.md` 两处下游行已同步」）以及 `data-config.md:82` / `:192`（已是「餐局流程解析 `station_id` 与 `entry_fee`」）正面相反。这是一条已解决却留着反结论的台账，会把实现者引向修错方向 | 本文件规则 22 / `:323`；`data-config.md:82`、`:192` |

### Recommended Revisions

1. **[RECOMMENDED] 胃容量下游行仍缺 `end_run(reason)`（`:300`，上轮 R3 未修）**：接口表 `:122` 有、`stomach-capacity.md:63`/`:204` 有、AC 45（`:440`）直接测它，唯独两张依赖表都没有，AC 45 的夹具契约在依赖章里查不到。
2. **[RECOMMENDED] 时间与回合行缺 `can_spend_turn()` / `is_exhausted()`（`:299`、`:121`）**：ADR-0013 §3 第 5 道闸门是 `turns.can_spend_turn()`，规则 6（`:34`）只写「回合仍可用」；`time-and-turns.md:216` 明写这两个接口「仍待餐局流程回填」。补进接口行，并说明吃菜用 `can_spend_turn()`、跳波用 `≥ 2`（规则 7 已写后者的「不能用」）。
3. **[RECOMMENDED] `action_availability()` 的 `*_reason` 值域未在本文件声明（`:33`、`:150`）**：AC 42（`:437`）断言的 `skip_reason = NONE` 不在 `:33` 的封闭 `RejectCode` 词表内；控件不可见时（最后一波的 `skip_*`、非最后一波的 `end_*`、`can_advance()` 为假）该带什么 reason、`run_state != Active` 时是否改报 `NOT_ACTIVE`（ADR-0013 §9 `:258`）都没写。加一句「`*_reason ∈ RejectCode ∪ {NONE}`，控件不可见时仍按同源校验表给出首个失败码」。
4. **[RECOMMENDED] `action_committed(action, payload)` 的 payload 仍未定义（`:134`，上轮 R8 未修）**：ADR-0013 §3 提交第 8 步给 `{dish_id, effective_satiety, extra_satiety}`，音效反馈（`audio-feedback.md`）与 HUD 无法按契约实现。至少把 `eat` 的 payload 字段写出来，跳波/结束取餐写 `{}`。
5. **[RECOMMENDED] `running_net` 不变式违反时没有错误码（`:244` 对 `:46`）**：不变式表写「不新增对外错误码」，但 `run_faulted(run_id, code, detail)` 的 `code` 必填，AC 32 只覆盖 `CAPACITY_MISMATCH`。给出 `RUNNING_NET_MISMATCH` 或明确该路径复用哪个码。
6. **[RECOMMENDED] 规则 8 的「无菜可吃」措辞与 Game Feel 的归因句不再相容（`:346`）**：新分支里后续波仍有菜、`no_eatable = false`，玩家却在看得到菜的画面上收到 `NO_EATABLE_DISH`；Game Feel 承诺「无菜可吃看剩余菜」无法解释。要求收据/结束文案区分「无菜可吃」与「已无合法动作」（或显示 `turns_remaining = 1` 与剩余容量），否则玩家会觉得本局莫名其妙结束。
7. **[RECOMMENDED] 判据在初始化时机也能命中，但只写了 `turns_total = 0`（`:36`、`:255`）**：ADR-0013 §6 `:211` 明写两个判定时机都能命中（`turns_total = 1` 且首波全放不下）。规则 8 或边角案例补一句，否则实现者会在初始化判定里只处理 `turns_total = 0`。
8. **[RECOMMENDED] `:322` 的「索引已对齐」已失效**：索引第 5 行（`systems-index.md:62`）仍是 `Status = Approved`，而本文件是 In Review 且带 4 项 blocking；该行的 `Depends On` 也缺`菜品数值`（见 B2）。把 `Status = Approved` 的自述改为当前事实，或删掉该断言，别让下一次复审继续读到一个假前提。
9. **[RECOMMENDED] 菜单波次「只读 `StationData.dish_pool`」的旧口径（`:280`、OQ5 `:461`，上轮 R11 未修）**：`menu-waves.md:63` 的 `initialize(...[, pool_override])` 与 `:349`（`pool_override` 非空时按覆盖池算基准）、AC 29（`:424`）都已接受覆盖，边角案例与 OQ5 仍写「暂不可用」。
10. **[RECOMMENDED] 两处台账漂移**：`:389` 的 Formulas 覆盖说明仍只把 `end_reason` 记到第 12、13、36 条，漏掉 AC 43；OQ4（`:460`）仍称「本系统暂定由观察顶层状态来丢弃」，而场景与流程管理已显式交付 `abandoned`（`ui-flow.md` 订阅该 payload），可改写成「已由 `abandoned` 显式交付」。

### Nice-to-Have

- AC 42 的「三次调用返回相同结构」很好；可再加一句 `action_seq` 不增（现已包含），并把 `end_reason` 在非最后一波的取值一并断言，顺手覆盖 R3 的值域。
- AC 45 可补 `phase = INIT` 的 `RUN_ID_OVERFLOW` 分支（`run_id = 0` + `reason = failed`），目前只由 AC 3 与规则 18 文字覆盖，没有合起来的判据。
- 规则 5 的「动作集只有三种」与 ADR-0013 §5「最后一波的 `skip_wave` 等价于 `end_pickup`」可以合并成一句，避免实现者给最后一波再造一个 `END_PICKUP` 入口。

### 判定说明

lean 模式不派发 `creative-director`，主评审意见即最终意见。核心骨架（1–24 条规则、状态/转移表、动作提交顺序、`fault.phase`、`end_run(reason)`、`running_net` 唯一累加器）与 ADR-0007 D1/D7、ADR-0013 §1–§9、ADR-0015 §8、ADR-0019 D3 自洽，六项指定契约中四项完全通过、一项（规则 7 门控）结论正确且理由句已修、一项（规则 3 初始化顺序）仍是 `TR-run-flow-002` 的架构 partial，不属本 GDD 的设计缺陷。四项 blocking 全部是「新决定只改了一处副本」：`end_reason` 的两处表达式、依赖表的三处登记、一条未收窄的边角案例、一条留着反结论的 OQ。改动量很小，但都属于实现前必须闭合的契约。

### Scope Signal

**Rough scope signal: S** — 单系统、四条流程不变量、无新增 ADR；本轮 blocking 只牵动本文件的公式/依赖表/边角案例/OQ 与注册表 `end_reason` 一行（以及 B2 的 `dish-values` 回指与索引第 5 行）。

### Verdict: NEEDS REVISION

- [BLOCKING] Formulas（`end_reason`）: 公式与变量表缺 `_no_legal_action` 分支，`no_eatable = false` 时无法产出 AC 43 要求的 `NO_EATABLE_DISH`（注册表同缺）
- [BLOCKING] Dependencies（上游表）: `菜品数值` 未列入却声称「已在依赖表列出」，索引第 5 行与 `dish-values.md` 下游表也无该边
- [BLOCKING] Edge Cases（容量已满行）: `:262` 仍写「回合未耗尽即不结束」，与 `turns_remaining = 1` 的同帧结束相反
- [BLOCKING] Open Questions #10: 仍写「本系统只解析 `station_id`」，与规则 22 / `:323` / `data-config.md` 相反
- [RECOMMENDED] Dependencies: 胃容量行缺 `end_run(reason)`；时间与回合行缺 `can_spend_turn()` / `is_exhausted()`
- [RECOMMENDED] Detailed Design: `*_reason` 值域未声明（`NONE` 不在封闭词表）；`action_committed` 的 payload 未定义
- [RECOMMENDED] Formulas: `running_net` 违反时无错误码；初始化时机命中新分支未写
- [RECOMMENDED] Game Feel: 「无菜可吃看剩余菜」无法解释「已无合法动作」的收据
- [RECOMMENDED] Dependencies/AC: `:322` 索引 `Status = Approved` 自述失效；菜单波次 `dish_pool` 旧口径；Formulas 覆盖漏 AC 43；OQ4 待改写

Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/scene-flow.md 4f0e50743f8b72012ff85540a58dfa1481cd676f
Reviewed-Content-Hash: design/gdd/data-config.md 92b3a28fc63465eacf3a88d3fec13bdff9f9455f
Reviewed-Content-Hash: design/gdd/time-and-turns.md 0f2ea128a9a42008413a15dd8608246513ef36f4
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 43fc521a857c5ce53ba3e1855c271ae2ffdfb1b6
Reviewed-Content-Hash: design/gdd/menu-waves.md a0b26691ca0f6ba5cac4c01ac0704913179b43ed
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/in-game-hud.md 27e2978106c35139e3a3d0f371472a7aafd5dc2a
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: design/gdd/input.md 18815e7d6735d9a0995b8cd79a762b516a07be1a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8
Reviewed-Content-Hash: docs/architecture/adr-0015-settlement-pure-functions.md cecf7e42a64c4edf19bdb3a50fc7b8029ca743eb
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 23304c8212ab59cede6f6f3d81cd40cea079cde5

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 2 | Recommended: 11
Summary: 上轮 4 条 blocking 全部在字节里闭合——`end_reason` 公式（`:201`）与变量表（`:209`）补了 `_no_legal_action()`、注册表同名字段（`entities.yaml:890-904`）带上了 `no_legal_action`；菜品数值已是上游行（`:294`）且 `dish-values.md:89`/`:280` 回指；容量满的边角案例（`:263`、`:265`）写明了 `turns_remaining = 1` 的同帧 `NO_EATABLE_DISH`；OQ10 已解决（`data-config.md:82`/`:193` 逐字核对）。"容量单独结束一局"的句子在全文已不存在。但本轮两条 blocking 是修的时候新引入的：`action_committed` 的 payload 被收窄成 `eat → {dish_id}`，与 ADR-0013 §3 第 8 步的 `{dish_id, effective_satiety, extra_satiety}` 相反；AC 44 把 `Ending`/`Settling`/`Handoff` 合并成一条断言「`settle` 调用次数仍为 1」，而 `Ending` 里结算从未发生。四项修复本身全部通过，判决由这两条新缺陷决定。
Prior verdict resolved: 部分 — 上轮 B1–B4 全部闭合；R1（胃容量行缺 `end_run`）、R2（`can_spend_turn()` / `is_exhausted()`）、R8（索引 `Approved` 自述）已修；R3（`*_reason` 值域）与 R4（payload）只修了半边；R5、R6、R7、R9、R10 未修，转为本轮 R4–R11。

### 复审背景（全量重读，非增量）

`review-receipts.sh check` 输出 `CHANGED: design/gdd/run-flow.md`、`CHANGED: design/registry/entities.yaml`、本 skill `UNCHANGED`，另有 15 份关联文档 `CHANGED`。按 `.claude/rules/design-docs.md`「Review receipts and post-approval edits」，目标文档已变，不做增量复审。本轮按 471 行全文分段重读（规则 1–24、RunConfig 表、状态表与转移表、交互表、快照/投影表、四条公式 + 不变式表、37 条边角案例、依赖章、AC 1–45 与 11a、OQ 1–11），并逐条重核 ADR-0013 §1–§9、ADR-0019 D3、ADR-0007 D3/D7、ADR-0015 §8、10 份上游/下游 GDD 与注册表自有 5 条实体。

### 四项指定修复的核对

1. **`end_reason` 公式**（通过）：`:201` 为 `TIME_EXHAUSTED if turns_remaining = 0 else (NO_EATABLE_DISH if (no_eatable or _no_legal_action()) else NONE)`；变量表 `:209` 定义 `_no_legal_action()` 并注明「容量满本身不算」；注册表 `entities.yaml:890-904` 的 `expression` 已带 `no_legal_action`、`notes` 写明「Capacity alone never ends a run」、`revised: 2026-11-06`。全部整数/布尔/枚举，无浮点，无第三原因；`TIME_EXHAUSTED` 优先级在 `:38`、`:212`、`:267` 与 AC 12 四处罚。
2. **菜品数值上游边**（通过）：`:294` 新增该行（硬，只读数值）；`:319` 的「已在依赖表列出」现在为真；`:323` 的双向核对列出「菜品数值」；`dish-values.md:89`（交互表下游）与 `:280`（依赖表下游）均已回指，其 header 亦标 2026-11-17。
3. **容量满边角案例**（通过）：`:263` 保留「本局不因容量结束」并补出「唯一的例外是非最后一波且 `turns_remaining = 1`」；`:265` 同一判据的第二处副本同步。
4. **OQ10**（通过）：`:468` 标为「已解决（2026-11-17）」；`data-config.md:82`/`:193` 确为「餐局流程解析 `station_id` 与 `entry_fee`（写进 `RunLog`）；结算只读 `RunLog.entry_fee`」，与规则 22/23、`:325` 一致（引用行号 `:192` 应为 `:193`，见 R9）。

**「容量单独结束 / 缺 `_no_legal_action()`」全文回扫**：`:36`、`:38`、`:201`、`:209`、`:214`、`:263`、`:265`、`:267`、AC 27（`:424`）、AC 39（`:436`）、AC 43（`:440`）共 11 处副本口径一致，无一处允许容量单独结束本局，也无一处缺新分支使 AC 43 不可达。**AC 27 仍然正确**：夹具限定 `turns_remaining ≥ 2`，该状态下 `_no_legal_action()` 恒为假（付费跳波可用），「最终只由回合耗尽或无可吃菜结束」在 ≥2 时成立；与 AC 43 的 `turns_remaining = 1` 分支互补而非冲突。

### 完整性

必需 6/6：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria（`gdd-structure-check.sh` 确认）。Advisory 缺口不阻塞：Player Fantasy、Tuning Knobs 缺失（`:156` 已声明这些是流程不变量而非平衡数值）。AC 编号 1–45 + 11a 连续无重号（46 条），措辞均可测；但 AC 44 的断言与规则自相矛盾，见 B1。

### 依赖图

上游 4 个声明全部在盘上（场景与流程管理、数据配置、菜品数值、输入）。13 份下游 GDD 全部回指本系统（逐文件「餐局流程」计数：scene-flow 19、data-config 4、input 3、dish-values 4、time-and-turns 42、stomach-capacity 38、menu-waves 14、cloying-and-taste 5、settlement 24、HUD 36、ui-flow 14、first-run 19、audio-feedback 8；save-system 0，与「无直接调用」声明一致）。**索引仍需更新（本文件不改）**：`systems-index.md:62` 的餐局流程行 `Status = Approved`（应为 In Review）且 `Depends On` 缺`菜品数值`；`:175` 的 Dependency Map Core Layer 第 2 项同样缺该边。附带一条编排决策：`菜品数值` 在索引里是 Core 第 5 项、`餐局流程` 是第 2 项，新增该边后 Core 层出现「后者依赖前者」的顺序倒置（`:168-178` 与 Recommended Design Order `:211`），要么把菜品数值前移，要么在索引里注明这是只读数值边、不改变设计顺序。

### 注册表值

自有 5 条（`run_id_alloc` `:861`、`run_seed` `:875`、`end_reason` `:890`、`transition_turn_cost` `:1257`、`transition_capacity_cost` `:1268`）与全文逐字一致。按实体名与数值键整篇回扫（`run_id|run_seed|end_reason|transition_*|running_net|shortfall|capacity_total|capacity_used|can_fit|entry_fee|turn_budget|base_capacity|cloy_cap|menu_vision|wasted_capacity|net_profit|extra_satiety|effective_satiety|opt_value|unit_value|*_cost` 等）命中 108 行，无第二套数值（本文件不重述 85 / 104 / 40 / 20 / 25 / 999）。仅两处台账级差异：注册表变量名 `no_legal_action` 与本文件 `_no_legal_action()` 不同形；`revised` 未随本轮表达式改动更新（R9）。

### Required Before Implementation

| # | 位置 | 问题 | Authority |
|---|---|---|---|
| B1 | `design/gdd/run-flow.md:441`（AC 44）对 `:41`（规则 13）、`:99`–`:102`（转移表）、`:274`（边角案例） | AC 44 把 `Ending` / `Settling` / `Handoff` 三阶段合为一条断言：「已算出的 `RunResult` 不交付…`settle` 调用次数**仍为 1** 且没有第二次」。但 `settle` 只在 `Ending → Settling` 中调用（规则 13），`Ending` 里既不调用、也还没有 `RunResult`；`:274` 明写「暂停发生在 `Settling` 之前：不调用 `settle`」。以 `Ending` 为夹具跑这条 AC，正确实现给出 `settle = 0`，该 AC 必红——三个互斥阶段共用一个断言，其中一个与规则相反。改法：按阶段拆开（`Ending` → `settle = 0`、无 `RunResult`；`Settling` / `Handoff` → `settle = 1`、结果不交付），并同步 `:44` 与 UI 要求 9（`:364`）的同一措辞 | 本文件规则 13、转移表 `:99-102`、边角案例 `:274`；`.claude/rules/design-docs.md:16`（AC 必须可验证） |
| B2 | `design/gdd/run-flow.md:33`、`:134` 对 `docs/architecture/adr-0013-action-pipeline-end-judge.md:141` | 规则 5 把 `action_committed` 的 payload 收窄为「`eat → {dish_id}`、`skip_wave → {}`、`end_pickup → {}`（不携带任何结算或波次数据）」，而 ADR-0013 §3 提交第 8 步广播的是 `action_committed(EAT, {dish_id, effective_satiety, extra_satiety})`（§4 / §5 的 `{}` 则与本文件一致）。上轮 R4 正是引 ADR 这三个字段要求把 payload 写出来，本轮写成了它的**子集**，于是 Accepted ADR 与本 GDD 对同一条信号给出两个结构。要么本文件按 ADR 补齐 `effective_satiety` / `extra_satiety`（都是 `EatenEntry` 的既有快照，无新增计算），要么在 ADR §3 记一笔「payload 收窄为 `{dish_id}`，消费方改读快照」并重开该 ADR——不能两份契约并存 | ADR-0013 §3（Accepted，`:141`）；本文件规则 5 `:33` 与信号声明 `:134`；`.claude/rules/design-docs.md:14`（依赖/契约双向一致） |

### Recommended Revisions

1. **[RECOMMENDED] `*_reason` 值域仍与 ADR-0013 §1 的封闭 `RejectCode` 冲突（`:33`，上轮 R3 只修半边）**：`:33` 写「两个 `*_reason` 均为 RejectCode 域，可用时为 `NONE`」，但 §1（`adr-0013:88-95`）的 `enum RejectCode` 里**没有** `NONE`，§9（`:250`、`:253`）又把字段声明为 `int  # …可用时 NONE`。同一句同时断言「在封闭词表内」与「取一个不在词表里的值」。写成 `RejectCode ∪ {NONE}`（并给 `NONE` 固定值）或在 §1 加 `NONE`，二者取一。
2. **[RECOMMENDED] 投影在不可见/非 `Active` 时的原因未声明（`:33`、AC 42 `:439`）**：ADR §9（`:259`）规定 `frozen` → `FROZEN`、`run_state != Active` → `NOT_ACTIVE`，且不可见控件（非最后一波的 `end_*`、最后一波的 `skip_*`）仍要给同源失败码；本文件只写「可用时为 `NONE`」，AC 42 只测 `FROZEN`。HUD 已按 `NOT_ACTIVE` 写文案（`in-game-hud.md:372`），本系统这一侧无据可依。
3. **[RECOMMENDED] `action_availability()` 依赖的 `can_advance()` / `can_end_pickup()` 在任何 GDD 里都不存在**：ADR-0013 §4（`:152`）、§5（`:169`）、§9（`:256`）把这两个谓词当作菜单波次的接口，而本文件 `:303` 的菜单波次接口行只有 `advance()` / `end_pickup()`，`menu-waves.md` 全文不出现 `can_advance`。投影要「与校验表同源」实现，必须先有这两个谓词的归属与失败码（ADR §4 给的是 `NO_EATABLE`）。附带风险：若 `can_advance()` 在非最后一波可为假，规则 8「`turns_remaining ≥ 2` 时玩家总能付费跳波」的前提就不成立，`_no_legal_action()` 会漏掉一个僵局。
4. **[RECOMMENDED] `:387` 的「双向回指（已完成）…本节无待办」与 `:323` 的清单分叉**：`:387` 只点名 9 个系统，漏掉 菜品数值（本轮新增的那条边）、场景与流程管理、数据配置、输入，而 `:323` 列了 13 个。同一事实的两份副本又出现分叉——前两轮反复出现的缺陷类型（本轮不阻塞：边本身已在三处登记且实际回指成立）。
5. **[RECOMMENDED] `:281` 仍写 `pool_override`「暂不可用」（上轮 R9 未修）**：该条件句的前提已不成立——`:303` 的 `initialize(..., [, pool_override])`、AC 29（`:426`）与 `menu-waves.md`（`pool_override` 非空时按覆盖池算基准）都接受覆盖。OQ5（`:463`）同样停在旧口径。
6. **[RECOMMENDED] `:391` 的 Formulas 覆盖说明仍漏 AC 43**：写着 `end_reason` 由「第 12、13、36 条」覆盖，而新分支唯一的判据是 AC 43（上轮 R10 未修）。
7. **[RECOMMENDED] 初始化时机命中新分支仍无判据（上轮 R7 未修）**：ADR-0013 §6（`:211`）明写两个判定时机都能命中（`turns_total = 1` 且首波全放不下）；本文件 `:216` 只写「初始化完成后一次」，边角案例 `:256` 只覆盖 `turns_total = 0`，AC 43 只覆盖「提交造成的状态」。补一条边角案例，或把 AC 43 的夹具扩成两个时机。
8. **[RECOMMENDED] `stomach-capacity.md:32`（跨文档，非本文件字节）规则 6 仍是绝对口径**：「当前可见菜都放不下时，餐局流程保持 `Active`；玩家可以付费跳波、等下一波，或用掉剩余回合」——缺 `turns_remaining = 1` 的例外，与同文件 `:192` 和本文件规则 8 相反。本文件这一侧正确且已被回指，缺陷在对方规则 6 尚未收窄。
9. **[RECOMMENDED] 两处台账漂移**：注册表 `entities.yaml:904` 的 `revised` 仍是 `2026-11-06`，而 `expression` 是本轮才补上 `no_legal_action` 的；变量名 `no_legal_action` 与本文件 `_no_legal_action()` 不同形。OQ10（`:468`）引用 `data-config.md:82`/`:192`，第二处实际是 `:193`。
10. **[RECOMMENDED] Game Feel 的归因句仍解释不了新分支（`:348`，上轮 R6 未修）**：「结束原因只有两种，且都能在收据上被解释——时间用完看浪费容量，无菜可吃看剩余菜」，但 `_no_legal_action()` 命中时画面上仍有菜、仍有剩余容量且仍显示 `turns_remaining = 1`，收据无法归因，与同句「玩家不应觉得这局莫名其妙就结束了」相反。要求收据/结束文案区分「无菜可吃」与「已无合法动作」。
11. **[RECOMMENDED] `running_net` 不变式违反时仍无错误码（`:245` 对 `:46`，上轮 R5 未修）**：不变式表写「不新增对外错误码」，但 `run_faulted(run_id, code, detail)` 的 `code` 必填，AC 32 只覆盖 `CAPACITY_MISMATCH`。给出 `RUNNING_NET_MISMATCH` 或明写复用哪个码。

### Nice-to-Have

- OQ10（`:468`）仍以「但本设计由结算读取；本系统只解析 `station_id`」作为问题前提，虽标了「已解决」，那句话读起来仍是当前结论；OQ1 的做法（用「已回答」替换前提）更干净。
- OQ9（`:467`）的「吃不下却还有回合是空转」在 `_no_legal_action()` 之后已部分失效：最后一回合的那种空转现在同帧结束，不再是可停留状态。
- `:214` 的算例可补一个新分支样例（回合从 2 用到 1、可见菜全放不下 → `NO_EATABLE_DISH` 且 `no_eatable = false`），让公式第三个输入有非零例子。
- 修完 B2 后可给 `action_committed` 加一条 payload 断言——46 条 AC 里没有一条检查 payload 形状。

### 判定说明

lean 模式不派发 `creative-director`，主评审意见即最终意见。上轮 4 条 blocking 逐条验证均已闭合，新分支的 11 处副本口径统一，AC 27 在 `≥ 2` 限定下依然正确——这是本轮最实质的进展。两条 blocking 都不是「漏改一处」，而是修改本身引入的新矛盾：payload 被收窄到与 Accepted ADR 相反，AC 44 用一条断言覆盖三个互斥阶段并使 `Ending` 夹具不可通过。两者都直接影响实现（信号结构）与验收（AC 会红），改动量小但必须在实现前闭合。

### Scope Signal

**Rough scope signal: S** — 单系统、四条流程不变量、无新增 ADR；本轮 blocking 只牵动本文件 `:33` / `:134` / `:44` / `:364` 与 AC 44，以及（或）ADR-0013 §3 的一行。

### Verdict: NEEDS REVISION

- [BLOCKING] Acceptance Criteria（AC 44）: 三阶段合并断言 `settle` 仍为 1 次，`Ending` 里从未结算，该夹具必红
- [BLOCKING] Detailed Design（`action_committed` payload）: `eat → {dish_id}` 与 ADR-0013 §3 的 `{dish_id, effective_satiety, extra_satiety}` 相反
- [RECOMMENDED] Detailed Design: `NONE` 不在 `RejectCode` 封闭词表；不可见/非 `Active` 的 `*_reason` 未声明；`can_advance()` / `can_end_pickup()` 无归属
- [RECOMMENDED] Dependencies: `:387` 回指清单漏 菜品数值 等 4 个系统；`:281` / OQ5 的 `pool_override` 旧口径
- [RECOMMENDED] Formulas / Edge Cases: 覆盖说明漏 AC 43；初始化时机命中新分支无判据；`running_net` 违反无错误码
- [RECOMMENDED] 跨文档: `stomach-capacity.md:32` 规则 6 缺 `turns_remaining = 1` 例外；注册表 `revised` 与变量名漂移；OQ10 行号 `:192` → `:193`
- [RECOMMENDED] Game Feel: 「无菜可吃看剩余菜」解释不了「已无合法动作」的收据

Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8
Reviewed-Content-Hash: design/gdd/data-config.md 36fd1ebaca36d1edee0c93b674f21d0418798713
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 65248e9286c1f43dbf1c07e037544a10e0f0bb01
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/in-game-hud.md b8ead1263039519de2ff2ad9378fe43a8ea58373
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

## Review — 2026-11-17 — Verdict: APPROVED（第 4 轮：逐条验证第 3 轮 AC 44 与 `action_committed` payload 两条 blocking 的闭合，并对当前 471 行全文做一次全新全量复审）

Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 0 | Recommended: 14
Summary: 第 3 轮两条 blocking 都已在当前字节里闭合，且不是「只改一半」：AC 44（`:441`）按阶段拆成两条互斥断言——`Ending` 下 `settle` 调用次数为 0 且无 `RunResult`，`Settling` / `Handoff` 下仍为 1 且结果不交付；两个夹具在规则 13、转移表 `:99-102`、边角案例 `:274` 下都能通过。`action_committed` 的 payload（`:33`）现在写成 `eat → {dish_id, effective_satiety, extra_satiety}` / `skip_wave → {}` / `end_pickup → {}`，与 ADR-0013 §3 第 8 步、§4、§5 逐字一致。本轮按全文（非增量）重读了 24 条规则、RunConfig 表、状态表与转移表、交互表、快照与投影表、四条公式 + 不变式表、37 条边角案例、两张依赖表、46 条 AC（1–45 加 11a）与 11 条 OQ，并重核 ADR-0013 §1–§9、ADR-0007 D1/D3/D7、ADR-0015 §8、7 份上下游 GDD 与注册表自有 5 条实体，未再发现会改变实现、改变信号结构或使某条规则/判据不可通过的问题；14 条 recommended 全部是「同一个事实的第二、三处副本」与台账同步，属清零工作而非契约缺口。
Prior verdict resolved: Yes — 第 3 轮 B1（AC 44 三阶段合并断言）与 B2（payload 收窄）全部闭合；第 3 轮 R1（`*_reason` 值域写成 `RejectCode ∪ {NONE}`）与 R8（`stomach-capacity` 规则 6 补 `turns_remaining = 1` 例外）亦已闭合。R2–R7、R9–R11 未修，转为本轮 R1–R11。

### 复审背景（全量重读，非增量）

`review-receipts.sh check` 输出 `CHANGED: design/gdd/run-flow.md`、`UNCHANGED: design/registry/entities.yaml`、本 skill `UNCHANGED`，另有 6 份关联文件 `CHANGED`（scene-flow、data-config、input、stomach-capacity、in-game-hud、save-system）与 ADR-0013 `CHANGED`。按 `.claude/rules/design-docs.md`「Review receipts and post-approval edits」，目标文档已变，不做增量复审；本轮以当前字节的全文重读为唯一输入，第 3 轮判决不作为任何段落的跳过依据。

ADR-0013 的 `CHANGED` 是本轮必须重核的一项：其当前字节在 §6 判据函数里保留 `_no_legal_action()` 分支、§9 把 `*_reason` 标注为 `RejectCode ∪ {NONE}` 并消歧同名 `end_reason` 域、§3 第 8 步仍是 `action_committed(EAT, {dish_id, effective_satiety, extra_satiety})`。本文件这三处（`:201`、`:33`、`:33`/`:134`）与之一致。

### 第 3 轮 blocking 闭合核对

| # | 第 3 轮 blocking | 当前字节 | 结论 |
|---|---|---|---|
| B1 | AC 44 把 `Ending` / `Settling` / `Handoff` 合成一条「`settle` 仍为 1」的断言，`Ending` 里从未结算 | `:441` 已拆为两段：`GIVEN` 当前在 `Ending`（`settle` 尚未调用）→ `settle` 调用次数为 0；`GIVEN` 当前在 `Settling` / `Handoff`（`settle` 已调用一次）→ 调用次数仍为 1、不出现第二次。与规则 13（`Ending → Settling` 中调用一次）、`:100`（`Ending` 暂停不调用）、`:274`（暂停在 `Settling` 之前不调用 `settle`）、`:86`/`:104`（`Handoff` 持有结果）逐条吻合 | **闭合**，两个夹具均可通过 |
| B2 | 规则 5 把 payload 收窄成 `eat → {dish_id}`，与 ADR-0013 §3 第 8 步相反 | `:33` 现为 `eat → {dish_id, effective_satiety, extra_satiety}`（注明「数值取吃下时刻的读数，ADR-0013 §3」）、`skip_wave → {}`、`end_pickup → {}`；与 ADR-0013 §3 第 8 步、§4 提交序列、§5 提交序列逐字一致；`:134` 的信号声明只写 `action_committed(action, payload)`，不再有第二套结构 | **闭合** |

**`settle` 计数的全文回扫**（防止第三处副本仍写「三阶段共用 1 次」）：`:41`（规则 13「恰好一次」）、`:85`（`Ending` 只组装 `RunLog`）、`:86`（`Settling` 只调用一次）、`:99-103`（转移表）、`:242`（不变式 `settle ≤ 1 / run_id`）、`:274`（暂停在 `Settling` 之前不调用）、`:441`（AC 44 两段）、AC 16（`:413`「`settle` 恰好调用一次」出现在 `Settling` 语境）共 9 处口径一致，没有一处允许 `Ending` 里结算，也没有一处把 `Ending` 的计数写成 1。

### 完整性

`gdd-structure-check.sh` 输出 `PRESENT: Overview, Detailed Rules, Formulas, Edge Cases, Dependencies, Acceptance Criteria`，`ABSENT: Player Fantasy, Tuning Knobs`。standard 层必需 6/6 齐全；两项缺失为 advisory（Player Fantasy 由 Game Feel `:341-350` 实质覆盖；Tuning Knobs 的缺席在 `:156` 有理由——这些是流程不变量而非平衡数值）。AC 编号 1–45 加 11a 连续无重号，46 条措辞均可测，本轮未发现新的不可断言判据。

### 依赖图

上游 4 行（场景与流程管理、数据配置、菜品数值、输入）与下游 10 行（时间与回合、胃容量、菜单波次、腻度与口味、结算与入场费、局内 HUD、界面流程、首局引导、音效反馈、存档）全部在盘上，无断链。**菜品数值这条边三处登记都在**：`:119`（交互表）、`:294`（上游表）、`:377`（Cross-References），且 `dish-values.md:89`（交互表下游）与 `:280`（依赖表下游）均已回指、`:3` 的表头标注 2026-11-17。`:319` 的「这条边已作为上游行列在依赖表」现为真陈述。回指计数逐文件复核（餐局流程出现次数）：stomach-capacity 38、time-and-turns 42、in-game-hud 36、settlement 24、menu-waves 14、data-config 3、dish-values 4，全部非零。

**索引仍需编排同步（本文件不改）**：`systems-index.md:62` 仍是 `Status = Approved`、`Depends On = 场景与流程管理, 数据配置, 输入`（缺菜品数值），`:175` 的 Dependency Map 第 2 项同缺。本文件 `:324` 已正确写成「索引由编排会话同步（不在本文件修改）」并给出应有的 `Depends On`，因此不再是假前提，只是待办动作（见 R12）。

### 注册表值

本系统自有 5 条与注册表逐字一致：`run_id_alloc`（`entities.yaml:861`）、`run_seed`（`:875`）、`end_reason`（`:890`）、`transition_turn_cost`（`:1257`）、`transition_capacity_cost`（`:1268`）。`end_reason` 的 `variables` 含 `no_legal_action`、`expression` 为 `TIME_EXHAUSTED if turns_remaining = 0 else (NO_EATABLE_DISH if (no_eatable or no_legal_action) else NONE)`、`notes` 写明「Capacity alone never ends a run」，与本文件 `:201`/`:209` 语义一致（仅变量名不同形，见 R12）。`running_net`（`:686`）/ `shortfall`（`:703`）的 `referenced_by` 含本文件、notes 写「run-flow owns this run's single live accumulator」，与规则 23 / 不变式 `:245` 一致。

按实体名与数值键整篇回扫（`run_id|run_seed|end_reason|transition_*|running_net|shortfall|capacity_total|capacity_used|can_fit|entry_fee|turn_budget|menu_vision|wasted_capacity|extra_satiety|effective_satiety|opt_value|unit_value|price|satiety`）：本文件不重述任何平衡数值（85 / 104 / 40 / 20 / 999 均只按引用转交），无数值冲突。

### Required Before Implementation

- none

### Recommended Revisions

1. **[RECOMMENDED] 信号段 `:134` 的「每个信号对同一 `run_id` 至多发一次」把 `action_committed` 也包进去了**：ADR-0004 D6 第 4 条只约束 `run_*`（`adr-0004:189`），ADR-0013 §3 第 8 步 / §4 / §5 要求每次成功提交各广播一次，`in-game-hud.md:33` 与 AC 24 依赖逐动作重绘、`audio-feedback.md:35` 也按「同一 `action_id` 每帧至多一次」去重。本句带「同一 `run_id`」限定、而 `action_committed` 没有 `run_id` 参数，故可勉强读作作用域外——但这正是最容易被实现成 once 守卫的一句。改成「`run_*` 生命周期信号对同一 `run_id` 至多一次；`action_committed` 每次成功提交各一次」。
2. **[RECOMMENDED] 规则 16（`:44`）与 UI 要求 9（`:364`）仍写「已算出的 `RunResult` 不交付」覆盖三阶段**：`Ending` 里既没有 `RunResult` 也没有 `settle`（规则 13、`:100`、`:274`）。AC 44 已按阶段拆开，正文这两处没跟；改成「`Ending` 无结果可弃；`Settling` / `Handoff` 已算出的结果不交付」，否则读者会以为 `Ending` 也是一个「算完被弃」的窗口。
3. **[RECOMMENDED] `:387` 的「双向回指（已完成）…本节无待办」与 `:323` 的清单继续分叉**：`:387` 只点名 9 个系统，漏 菜品数值、场景与流程管理、数据配置、输入（`:323` 列了 13 个，且四者实际都已回指）。同一事实两份副本，第三轮已点名，仍未合并。
4. **[RECOMMENDED] `*_reason` 的非可用取值规则仍未写进本文件（`:33`）**：值域已正确写成 `RejectCode ∪ {NONE}`（上轮 R1 已闭），但 ADR-0013 §9（`:258`）还规定 `frozen → FROZEN`、`run_state != Active → NOT_ACTIVE`，且不可见控件（非最后一波的 `end_*`、最后一波的 `skip_*`、`can_advance()` 为假）仍要给同源失败码；`in-game-hud.md:372` 已按 `NOT_ACTIVE` 写文案，本系统这一侧仍无据。另 `NONE` 的整数值在 ADR §9 与本文件都未定，若与 `RejectCode` 同域应给固定值。
5. **[RECOMMENDED] `action_availability()` 依赖的 `can_advance()` / `can_end_pickup()` 在任何 GDD 里都没有归属**：`menu-waves.md` 全文 0 次出现 `can_advance`；本文件 `:123`、`:303` 的菜单波次行只列 `advance()` / `end_pickup()` / `visible_dishes`，而 ADR-0013 §4（`:154`）、§5（`:170`）、§9（`:256`）把这两个谓词当作菜单波次的接口。投影要求「与校验表同源」，规则 8 的「`turns_remaining ≥ 2` 时玩家总能付费跳波」也建立在其上——若 `can_advance()` 在非最后一波可为假，`_no_legal_action()` 会漏掉一个僵局。
6. **[RECOMMENDED] `:391` 的 Formulas 覆盖说明仍漏 AC 43 与 AC 34**：写着 `end_reason` 由「第 12、13、36 条」覆盖，而新分支唯一的判据是 AC 43（`:440`）；「运行时不变式由第 3、14、16–19 条覆盖」也漏掉 `running_net` 恒等式的唯一判据 AC 34（`:431`）。第三轮已点名，未修。
7. **[RECOMMENDED] 初始化时机命中 `_no_legal_action()` 仍无判据（`:216` 与 `:256`）**：ADR-0013 §6（`:211`）明写两个判定时机都能命中（`turns_total = 1` 且首波全放不下），本文件 `:216` 只写「初始化完成后一次」，边角案例 `:256` 只覆盖 `turns_total = 0`，AC 43 只覆盖「提交造成的状态」。要么补一条边角案例，要么把 AC 43 的夹具扩成两个时机。
8. **[RECOMMENDED] `:245` 的 `running_net` 不变式违反时仍无 `run_faulted` 错误码**：不变式表写「不新增对外错误码」，而 `run_faulted(run_id, code, detail)` 的 `code` 必填（`:134`/`:46`）、AC 32 只覆盖 `CAPACITY_MISMATCH`，全仓也不存在第二个可用码。给出 `RUNNING_NET_MISMATCH` 或明写复用哪个码（三轮未修）。
9. **[RECOMMENDED] Game Feel `:348` 的归因句仍解释不了新分支**：「结束原因只有两种，且都能在收据上被解释——时间用完看浪费容量，无菜可吃看剩余菜」，但 `_no_legal_action()` 命中时画面上仍有菜、仍有剩余容量、仍显示 `turns_remaining = 1`，收据无从归因，与同句「玩家不应觉得这局莫名其妙就结束了」相反。要求收据/结束文案区分「无菜可吃」与「已无合法动作」（三轮未修）。
10. **[RECOMMENDED] `:281` 与 OQ5（`:463`）仍写 `pool_override`「暂不可用」/「菜单波次当前只读 `StationData.dish_pool`」**：`menu-waves.md:63` 已接受 `[, pool_override]`、`:226` 与 AC 29（`:426`）都按覆盖池工作，条件句的前提已不成立（三轮未修）。
11. **[RECOMMENDED] `:468` 的 OQ10 仍留着反结论与错行号**：问题前提仍写「但本设计由结算读取；本系统只解析 `station_id`」，随后才写「已解决」，与规则 22 / `:325` 相反；引用 `data-config.md:82`/`:192`，第二处实际是 `:193`（OQ1 那种「用结论替换前提」的写法更干净）。
12. **[RECOMMENDED] 台账漂移（本文件不改，编排侧处理）**：注册表 `entities.yaml:904` 的 `revised` 仍是 `2026-11-06`，而 `expression`/`notes` 的 `no_legal_action` 是 2026-11-17 才补的；变量名 `no_legal_action` 与本文件 `_no_legal_action()` 不同形；`systems-index.md:62` / `:175` 的 Status 与 `Depends On` 仍待同步（见依赖图）。
13. **[RECOMMENDED] 跨文档两处陈旧自述**：`time-and-turns.md:216` 仍写「`can_spend_turn()` / `is_exhausted()` 仍待餐局流程回填」，而本文件 `:121`/`:301` 已列出；`settlement-and-entry-fee.md:99` 仍把 `end_reason` 记在「时间与回合」上游行（同文件 `:65` 归餐局流程，`time-and-turns.md:43` 明写本系统不保存、不排序 `end_reason`）。
14. **[RECOMMENDED] 接口行与依赖行仍有三处副本不对称**：`:13` 的 Quick reference `Key deps` 仍只有「场景与流程管理, 数据配置, 输入」，缺菜品数值；`:121`（交互表 时间与回合行）仍缺 `can_spend_turn()` / `is_exhausted()`（`:301` 已有），规则 6 的吃菜第 4 道闸门也只写「回合仍可用」而 ADR-0013 §3 第 5 行是 `turns.can_spend_turn()`；`:293`（数据配置上游行）只写 `get_station(station_id)`，未列规则 22 实际读到并写入 `RunLog` 的 `turn_budget` / `entry_fee` / `dish_pool`（`:118` 有）。附带一处缺判据：`RunConfig.entry_fee_override < 0` 只有 `:66` 的「必须 ≥ 0」，规则 4 的失败枚举与边角案例都没有对应项（`upgrade_snapshot` 有）。

### Nice-to-Have

- AC 13（`:410`）与 AC 37（`:434`）仍以 `turns_total = 0` 为夹具，而 `time-and-turns.md:40` 对 `turn_budget < 1` 返回 `INVALID_TURN_BUDGET`：AC 13 可沿用 ADR-0013 的「防御分支、正常数据不可达」措辞，AC 37 可改用初始化失败路径（AC 38 已覆盖同帧拒绝）。
- `:96` 的 `Initializing → Active` 触发只列「时间与回合、胃容量、菜单波次」，漏 腻度与口味（规则 3 与 AC 1 都要求四次 `initialize`）；「腻度计数重置」同时出现在副作用列，与 `cloying-and-taste.md:40`「`run_started` 之前调用 `initialize` 并清零」有半拍错位。
- `:18`（规则 18）「`phase = INIT` 时 `run_id = 0`」与规则 3「先分配 `run_id` 再初始化」在「子系统初始化失败」这条 INIT 路径上会打架：此时 `run_id` 已分配（例如 8），按本句却要报 0，而 ADR-0007 D1 把 0 定义为「尚未分配」。`station_id` 不存在（AC 4）在分配前失败所以无歧义，但没有一条 AC 固定「子系统初始化失败」那条路径的 `run_id`。
- 表头 `Last Verified: 2026-11-03` 早于两轮修订；`Author` 行仍称「systems-designer 与 qa-lead 按 Lean 模式仅复核 Formulas 与 Acceptance Criteria」，而 SKILL.md 的 lean 模式不派发任何专员（本日志四轮均记 `Specialists: none`）。
- 46 条 AC 里仍无一条断言 `action_committed` 的 payload 形状；`end_reason` 公式 `:214` 的算例可补一个 `_no_legal_action()` 命中样例（回合从 2 用到 1、可见菜全放不下 → `NO_EATABLE_DISH` 且 `no_eatable = false`）。

### 判定说明

lean 模式不派发 `creative-director`，主评审意见即最终意见。第 3 轮两条 blocking 都逐字核对通过：AC 44 现在是两条互斥断言且各自与规则 13 / 转移表 / 边角案例对得上，`Ending` 夹具在正确实现下给出 0、`Settling` / `Handoff` 给出 1，不再有一条必红的合并断言；payload 与 ADR-0013 §3 的字段集和字段名完全一致，不再是子集。核心骨架（1–24 条规则、状态/转移表、动作提交顺序、`_no_legal_action` 出口、`fault.phase`、`run_id = 0` 边界、`running_net` 唯一累加器、`end_run(reason)` 的四条终止路径）与 ADR-0007 D1/D3/D7、ADR-0013 §1–§9、ADR-0015 §8、ADR-0019 D3 自洽；六项指定契约中五项完全通过，第六项（规则 3 的初始化顺序）仍是 `TR-run-flow-002` 的架构 partial，属 ADR-0007 待补一句，不构成本 GDD 的设计缺陷。14 条 recommended 里没有一条会改变规则语义或使判据不可通过：它们集中在「同一个事实的第三处副本」（`:13`、`:121`、`:293`、`:387`、`:391`、`:134`）与台账日期/行号，可以在实现前一次性清零，不阻塞程序员按现行文本开工。

### Scope Signal

**Rough scope signal: S** — 单系统、四条流程不变量 + 一条净值恒等式、无新增 ADR；本轮无 blocking，14 条 recommended 只牵动本文件若干派生副本与三份台账（注册表 `revised`、`systems-index` 第 5 行、`time-and-turns:216` / `settlement:99` 两句）。

### Verdict: APPROVED

- [BLOCKING] none
- [RECOMMENDED] Detailed Design: `:134`「每个信号至多发一次」把 `action_committed` 包进去了（ADR-0004 D6 只约束 `run_*`）；`*_reason` 的 `FROZEN` / `NOT_ACTIVE` 与不可见控件取值未写；`can_advance()` / `can_end_pickup()` 无归属；`:121` 缺 `can_spend_turn()` / `is_exhausted()`；`:293` 缺 `turn_budget` / `entry_fee` / `dish_pool`；`entry_fee_override < 0` 无判据
- [RECOMMENDED] 规则 16 / UI 要求 9: 三阶段仍写「已算出的 `RunResult` 不交付」，`Ending` 里没有结果（AC 44 已拆，正文未跟）
- [RECOMMENDED] Formulas / Edge Cases: 覆盖说明漏 AC 43 与 AC 34；初始化时机命中新分支无判据；`running_net` 违反无错误码
- [RECOMMENDED] Dependencies: `:13` Key deps 与 `:387` 回指清单缺菜品数值等；`:281` / OQ5 的 `pool_override` 旧口径
- [RECOMMENDED] Open Questions: OQ10 留着反结论与 `:192` → `:193` 错行号
- [RECOMMENDED] 跨文档: 注册表 `revised` 与变量名漂移；`systems-index:62`/`:175` 待同步；`time-and-turns:216`、`settlement:99` 两句陈旧自述
- [RECOMMENDED] Game Feel: 「无菜可吃看剩余菜」仍解释不了「已无合法动作」的收据

Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: docs/architecture/adr-0015-settlement-pure-functions.md cecf7e42a64c4edf19bdb3a50fc7b8029ca743eb
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 93968bd02251e306f5b99c7b1cc5800272e54a06
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: design/gdd/data-config.md 8d799e593ad60b254814ede22b4f9b3fc8de99e4
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/run-flow.md da67f644d65824cb6ae98081cea03f69f0e77a0d
