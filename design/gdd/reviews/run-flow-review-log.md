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
