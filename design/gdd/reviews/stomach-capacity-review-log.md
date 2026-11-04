# Stomach Capacity (胃容量) — Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 8
Summary: The ledger itself is sound — the four formulas are integer-only, inclusive at the boundary, monotonic by construction, and the 20-cell / 104-yuan calibration is reproduced exactly (the 10-dish optimum sums to 20 cells and 104 yuan; the top-10 by price sums to 26, as Open Question 3 says). One contract-level gap blocks: no mechanism is stated that returns the ledger from `就绪` to `未初始化` at a run boundary, so a same-run second `initialize` (AC 5, must fail) and the first `initialize` of the next run (AC 24, must succeed) are the same call on the same inputs. Open Question 1 is now answered by `data-config.md` (`BalanceData.base_capacity`), but four sites still describe the field as missing; several dependency and cross-reference notes are stale against GDDs that have since been merged.
Prior verdict resolved: First review

Findings:
- [BLOCKING] Detailed Design (rule 2 `:28`) / States (`:46-52`) / AC 5 (`:275`) vs AC 24 (`:303`): the run-boundary transition `就绪 → 未初始化` has no defined trigger, and `initialize(upgrade_snapshot)` carries no run identity. The system therefore cannot distinguish "second initialize this run" (must return `CAPACITY_INIT_INVALID`) from "first initialize of the next run" (must return a total) — the two ACs describe the same observable input. Edge Cases `:187` makes it worse by saying discard "needs no cleanup call". Name the trigger — subscribing to run-flow's `run_ended` / `run_discarded` (`run-flow.md:130`) is the cheapest fix that preserves rule 12 and rule 7 — or add an explicit `end_run(discarded)` to the interface, and reconcile edge `:187`.
- [RECOMMENDED] Open Questions 1 (`:312`), Interactions (`:63`), Dependencies (`:202`), Cross-References (`:253`): still call `C0` a missing field, but `data-config.md:59,117,185,246` now defines `BalanceData.base_capacity` (1–999, default 20) and validates it. Point the rows at `get_balance() → base_capacity` and close Open Question 1.
- [RECOMMENDED] Dependencies (`:214`): says 局外升级 / 局内 HUD / 纪录与里程碑 / 存档 GDDs are "尚未撰写", contradicting this file's own rows (`:64`, `:66`, `:70`) and the tree — all four exist. Correct the paragraph, and note that `records-and-milestones.md` contains no reference to 胃容量 at all, so the soft downstream row (`:70`, `:209`) is currently one-sided.
- [RECOMMENDED] Cross-References (`:263`): the "待回改" note says `run-flow.md` omits `consume(e)`; `run-flow.md:118,291` already lists it. Stale — delete the note.
- [RECOMMENDED] Edge Cases (`:186`): attributes duplicate-submission detection to the `capacity_used = Σ effective_satiety` invariant. That invariant cannot catch it — a second committed entry also increments the accumulator, so the sum still matches. The real defense is `run-flow.md:34` rule 6's first check (the dish was removed from the visible set). Cite that, or scope the invariant claim to a double `consume` that logged only one row.
- [RECOMMENDED] Detailed Design (rule 5 `:31`, rule 8 `:34`) / Formulas (`:141`, `:147`) / AC 10 (`:283`), AC 15 (`:291`): the guards cover non-integer `e` (`can_fit(2.5)`, `consume(2.5)`), but no signature is given. A typed `e: int` in GDScript rejects the argument before the guard runs, so those ACs cannot execute as written. Declare `e: Variant` (or state that the test enters through an untyped boundary).
- [RECOMMENDED] Dependencies (`:65`, `:204`): declares no direct call edge to 腻度与口味, but `cloying-and-taste.md:61` still records the row as `双向` and says the capacity system decides "可吃性与\"吃撑\"". The direction and the "吃撑" wording contradict this GDD's no-`STUFFED` rule. The fix belongs in `cloying-and-taste.md` (out of scope for this review) — track it rather than treating the row as settled.
- [RECOMMENDED] Tuning Knobs: absent. Advisory at `standard`, but `C0 = 20` (`:93`) and the `999` structural cap (`:36`, `:97`) are the system's two tunable values and their safe ranges are worth one short section.
- [RECOMMENDED] Registry (`design/registry/entities.yaml`): `capacity_remaining` (formula `:119`) and `capacity_bonus` (`:156`, owned by 局外升级) have no entry; `capacity_used`'s `referenced_by` (`:699-701`) omits `run-flow.md`, which reads it at `run-flow.md:140`; the `base_capacity` note (`:1272`) still says ownership is an open question. Registry edits belong to the orchestration session.

Advisory (not counted): Formulas declare `capacity_total ∈ [1, 999]` while the pre-`initialize` accessor returns `0` (`:192`) — say the range describes an initialized instance. `capacity_bonus` is bounded `0–(999 − C0)` here but `0–99` in `meta-upgrades.md:202`; compatible, but naming the upstream bound would make rule 10's purchase-time check concrete. Open Question 7 can close: `audio-feedback.md:164,404` has a "容量变化" cue but no capacity event, which confirms derivation from `action_committed`.

---

## Design Review: 胃容量（Stomach Capacity）
Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules, Edge Cases,
Dependencies, Acceptance Criteria — plus Formulas, because this system defines
numeric rules (capacity totals, consume validation, fit predicate). All six are
present; `## Detailed Design` is accepted as the Detailed Rules section.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:229-235`) covers
  the intended feeling (the receipt confirms the mental arithmetic; full is not
  failure).
- **Tuning Knobs** — absent; see Recommended item 8.

### Dependency Graph

Every declared dependency has a GDD on disk.
- ✓ run-flow.md — exists
- ✓ data-config.md — exists
- ✓ meta-upgrades.md — exists
- ✓ cloying-and-taste.md — exists
- ✓ in-game-hud.md — exists
- ✓ settlement-and-entry-fee.md — exists
- ✓ menu-waves.md — exists
- ✓ dish-values.md — exists
- ✓ records-and-milestones.md — exists
- ✓ save-system.md — exists

### Registry Values

Searched the whole document for the registry's capacity terms (`C0`,
`base_capacity`, `capacity_total`, `capacity_used`, `capacity_remaining`,
`effective_satiety`, `999`, `104`). No value conflicts:

- `base_capacity = 20` (`entities.yaml:1263`) matches `:30`, `:93`, `:99`;
  `data-config.md:59` now owns the field.
- `capacity_total ∈ [1, 999]` (`:881-891`) matches `:36`, `:91-97`.
- `capacity_used ∈ [0, 999]` (`:696-708`) matches the structural-cap
  justification at `:36`; the GDD's tighter `[0, capacity_total]` is the same
  quantity under the cap.
- `can_fit` (`:897-907`) matches rule 5 and `:141`, inclusive boundary included.
- The 104-yuan optimum reproduces exactly: the ten dishes at
  `dish-values.md:63` sum to 20 cells and 104 yuan; the ten highest-priced
  dishes sum to 26 cells — Open Question 3's figures are correct.
- Registry gaps (no entry for `capacity_remaining`, `capacity_bonus`; stale
  `base_capacity` note) are listed as Recommended item 9.

### Required Before Implementation

1. **Detailed Design / States / Acceptance Criteria 5 vs 24** — the run boundary
   is not observable. `initialize(upgrade_snapshot)` has no run id, the state
   table's `就绪 → 未初始化` edge names no trigger, the interaction table lists
   no lifecycle input, and edge `:187` says discard requires no cleanup call.
   AC 5 (second init in the same run → `CAPACITY_INIT_INVALID`, `used` stays 17)
   and AC 24 (abandon at `used = 12`, next init → `used = 0`) are therefore the
   same call sequence with the same arguments and must return different results.
   Fix: subscribe to `run_ended` / `run_discarded` (`run-flow.md:130`) and state
   that both return the ledger to `未初始化`; or add `end_run(discarded)` to the
   owned interface and have run-flow call it. Then rewrite edge `:187` so it no
   longer says no cleanup call exists, and add an AC that drives the boundary
   through that signal.

### Recommended Revisions

2. **Open Question 1 is answered** — `BalanceData.base_capacity` exists
   (`data-config.md:59,117,185,246`). Update `:63`, `:202`, `:253`, `:312`.
3. **Dependencies `:214`** contradicts `:64`, `:66`, `:70` and the tree: the
   局外升级 / 局内 HUD / 纪录与里程碑 / 存档 GDDs are written. Meta-upgrades
   (`meta-upgrades.md:346`) and HUD (`in-game-hud.md:166`) do back-reference
   this system; `records-and-milestones.md` does not mention it, so the soft
   row at `:70`/`:209` is one-sided.
4. **Cross-References `:263`** — the `consume(e)` gap in run-flow was closed
   (`run-flow.md:118,291`); remove the "待回改" note.
5. **Edge Cases `:186`** — duplicate submission is caught by run-flow's
   visible-set check (`run-flow.md:34`), not by the sum invariant.
6. **`e`'s type** — rules 5/8 and AC 10/15 require non-integer input to reach
   the guard; declare `Variant` or document the untyped test boundary.
7. **腻度 row direction** — `cloying-and-taste.md:61` still says `双向` and
   "吃撑"; fix belongs to that GDD.
8. **Tuning Knobs** — add `C0` (1–999, MVP 20) and the 999 structural cap.
9. **Registry** — missing `capacity_remaining` / `capacity_bonus`; `capacity_used`
   lacks a `run-flow.md` reference; `base_capacity` note is stale.

### Specialist Disagreements

None — Lean mode runs no specialists (`Phase 3b` skipped).

### Nice-to-Have

- Formula output range vs the `0` returned before `initialize` (`:192`).
- `capacity_bonus` bound `0–(999 − C0)` vs `0–99` upstream (`meta-upgrades.md:202`).
- Close Open Question 7 — audio has a "容量变化" cue but no capacity event.
- `systems-index.md:26-27` still says capacity "decides when the player is
  stuffed", which Open Question 5 argues against; the index edit belongs to the
  orchestration session.

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/stomach-capacity.md bb97f41a121c6c18d15f7d10c23387ff25688b11
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 02d37ce06fa293e6b76cb22277a68b705e413448
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 0f58ea62a2e8fa9ace2effe652cc644acab89d5b
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/systems-index.md 8f9bdb61bf682a7e4d7191933202f9efb3d68cde

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 12
Summary: 上一轮的 blocking 只解决了一半：正常结束（`run_ended`）与放弃（`run_discarded`）两条路径现在会回到 `未初始化`，AC 5 与 AC 24 在这两条路径上已可区分；但初始化失败与 `Faulted` 两条终止路径不发这两个信号，账本停留在 `就绪`，下一局的 `initialize` 会被判成“同局第二次初始化”，此后每一局都失败，只能重启应用恢复。另有 `run_ended` 时“立即”重置与 HUD 在同一信号上读取 `can_fit` 的顺序冲突。上一轮的 8 项 recommended 未处理（OQ1 字段缺口、Dependencies 的“尚未撰写”、`consume(e)` 待回改、重复提交归因、`e` 的类型、腻度行方向、Tuning Knobs、注册表缺口），本次全部复现。
Prior verdict resolved: No — partial（正常结束 / 放弃已修；Failed / Faulted 仍未覆盖）
Findings:
- [BLOCKING] Detailed Design rule 2 (`:28`) / States (`:48-52`) / Edge `:187`: 局边界只订阅 `run_ended` 与 `run_discarded`，但两条终止路径两者都不发——(a) 胃容量初始化成功、后续子系统初始化失败时，run-flow `Initializing → Failed`（`run-flow.md:95`）并请求 `Run → Menu`，从未开始的局不发 `run_discarded`（`run-flow.md:131`）；(b) 提交期 `CAPACITY_OVERFLOW`（本文件 `:185`、AC 14）使 run-flow 进入 `Faulted`，而 `Faulted → Idle` 是本系统自己的交接，不是“顶层离开 `Run` / `Paused`”（`run-flow.md:104`），`run_ended` 也从未广播。两种情况下账本停在 `就绪`，下一局 `initialize` 被规则 2 判为同局第二次 → `CAPACITY_INIT_INVALID` → run-flow `Failed`，此后每局必败，只有重启可恢复；规则 2 的“因此下一次 `initialize` 必然是新局，`initialize` 本身不需要携带 run identity”因此不成立。修法：给 `initialize` 加 `run_id`，或让 run-flow 在**所有**终止路径（含 Failed / Faulted 清理）显式调用 `end_run(reason)`；只订阅 `run_faulted` 不够——run-flow 声明了该信号（`:131`）但当前没有任何状态转移广播它。
- [RECOMMENDED] 重置时机 vs HUD: 规则 2 要求收到 `run_ended` 时“立即”回到 `未初始化`，但 HUD 正是在 `run_ended` 上构造 Ending 帧，并直接从本系统读 `can_fit(d)` / `capacity_remaining`（`in-game-hud.md:33,166`）。若本系统的回调先执行，Ending 帧会读到 `0` / `false` 并记 ERROR（`:192`），与 HUD 的“数值冻结”（`in-game-hud.md:148`）矛盾。应写明账本在 `Ending` 期间仍可读，或改为在 `run_settled` / `run_discarded`（Ending 消费者之后）重置，并固定回调顺序。
- [RECOMMENDED] States `:52`: `已丢弃` 行在修复后已不可达——`:48` 说收到 `run_discarded` 进入 `未初始化`，`:52` 却仍保留独立的 `已丢弃` 阶段。删除或改为 `未初始化` 的别名。
- [RECOMMENDED] AC 17 (`:293`): 枚举“能改变 `capacity_used` 的方法”时只列 `initialize` 与 `consume`，但边界重置现在是第三条清零路径（`:28`、AC 24）。把 AC 17 限定为“一局内”，并把边界重置补进枚举或单列一条 AC。
- [RECOMMENDED] 验收缺口: 上一轮要求“增加一条通过信号驱动边界的 AC”，未加。AC 6 / 24 只断言新一次 `initialize` 的结果，从未断言 `run_ended` / `run_discarded` 送达后账本处于 `未初始化`。补 GIVEN 送达信号 WHEN 读取 THEN `未初始化` / `0` + ERROR，以及“同局第二次 vs 下一局第一次”的成对用例。
- [RECOMMENDED] Open Question 1 (`:312`) 与 `:63`、`:253`: 仍称 `C0` 无数据归属，但 `data-config.md:59,117,185,246` 已定义并校验 `BalanceData.base_capacity`（1–999，默认 20）。关闭 OQ1 并更新这两处（Dependencies 行 `:202` 已更新）。OQ4 亦可关闭：`entities.yaml:698-701` 已把本 GDD 列入 `capacity_used.referenced_by`。
- [RECOMMENDED] Dependencies `:214`: 仍说 局外升级 / 局内 HUD / 纪录与里程碑 / 存档 的 GDD “尚未撰写”；四份都存在，且本文件 `:64`、`:66`、`:70` 自己就引用了它们。
- [RECOMMENDED] Cross-References `:263`: “待回改：run-flow.md 漏了 `consume(e)`”已过期——`run-flow.md:119,292` 已列出 `consume(e)`。
- [RECOMMENDED] Edge Cases `:186`: 把重复提交的防御归于 `capacity_used = Σ effective_satiety` 不变式；第二次提交同样会累加，和仍然相等。真正的防御是 run-flow 的第一道校验“菜仍在当前可见集”（`run-flow.md:34`）。改引该条，或把不变式的作用域限定为“只记了一行却调用两次 `consume`”。
- [RECOMMENDED] `e` 的类型: 规则 5 / 8 与 AC 10 / 15 需要非整数 `e`（`can_fit(2.5)`）到达守卫，但 GDScript 的 `e: int` 会在进入函数前拒绝该参数。声明 `e: Variant`，或写明测试从无类型边界进入。
- [RECOMMENDED] 腻度行方向: `cloying-and-taste.md:61` 仍把本系统记为 `双向` 并称容量决定“吃撑”，与规则 6（无 `STUFFED`）及本文件 `:75`、`:212` 的无调用边方向矛盾。修在对方 GDD（本次范围外）。
- [RECOMMENDED] 注册表 (`entities.yaml`，由编排会话维护): `capacity_remaining`（`:119`）与 `capacity_bonus`（`:156`）无条目；`capacity_used.referenced_by`（`:698-701`）漏了 `run-flow.md`（其 `:119,292` 读取该值）；`capacity_total` 注 (`:893`) 与 `base_capacity` 注 (`:1272`) 仍称数据归属是胃容量的 Open Question，而 `data-config.md:59` 已拥有该字段。
- [RECOMMENDED] Tuning Knobs: 缺失（`standard` 下 advisory）。`C0`（1–999，MVP 20）与 999 结构上限是本系统的两个可调值，补一节写安全区间与影响面。
Advisory (not counted): 公式声明 `capacity_total ∈ [1, 999]` 但 `initialize` 前读取返回 `0`（`:192`）——注明区间描述的是已初始化实例。本文件的 `capacity_bonus` 上界 `0–(999 − C0)` 与 `meta-upgrades.md:202` 的 `0–99` 兼容，但写出上游上限能让规则 10 的购买时校验具体化（当前 `20 + 99 ≤ 999` 恒成立）。OQ7 可关闭：`audio-feedback.md:164,404` 有“容量变化” cue 而无容量事件，确认由 `action_committed` 派生。`systems-index.md:26` 仍说容量 “decides when the player is stuffed”，与 OQ5 / 规则 6 矛盾（索引编辑归编排会话）。

### Completeness: 6/6 required sections present
`modes.workflow: standard` → Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge Cases, Dependencies, Acceptance Criteria all present. Advisory at `standard`: Player Fantasy and Tuning Knobs absent (Game Feel `:229-235` covers the fantasy informally).

### Dependency Graph
All declared dependencies have a GDD on disk: run-flow, data-config, meta-upgrades, cloying-and-taste, in-game-hud, settlement-and-entry-fee, menu-waves, dish-values, records-and-milestones, save-system.

### Registry Values
Whole-document search for the registry's capacity terms (`capacity_total`, `capacity_used`, `capacity_remaining`, `capacity_bonus`, `base_capacity`, `can_fit`, `effective_satiety`) returned 116 lines; no value conflicts. `base_capacity = 20`, `capacity_total ∈ [1, 999]`, `capacity_used ∈ [0, 999]` and the inclusive `can_fit` predicate all match. Calibration re-verified from `dish-values.md:46-58`: the ratio-optimal ten dishes sum to 20 cells / 104 yuan; the ten highest-priced sum to 26 cells / 121 yuan (Open Question 3's figures are correct). Entry gaps are listed under Recommended item 12.

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 94da89a242967c44827406c4da5714d088df6394
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 12
Summary: 本轮的修复（`427409f`）只覆盖了三条终止路径中的一条。`initialize` 自身失败后账本保持 `未初始化`，这条已修好；但新加的 `run_faulted` 订阅在本项目里永远不会触发——run-flow 声明了该信号（`run-flow.md:132,330`）、HUD 与音效也订阅它，可是状态转移表的“副作用”列是穷举的，进入 `Faulted` 的两条转移（`run-flow.md:88,106`）都不广播它。于是另外两条路径仍把账本留在 `就绪`：初始化成功但后续子系统初始化失败（`Initializing → Failed`，run-flow 不发任何信号），以及提交期 `CAPACITY_OVERFLOW` 进入 `Faulted`（同样不广播）。两种情况下下一局 `initialize` 都被判成“同局第二次” → `CAPACITY_INIT_INVALID` → run-flow 进入 `Faulted`，此后每局必败，只能重启应用——这正是上一轮 blocking 的原样复现。上一轮的 12 项 recommended 也全部复现（OQ1、Dependencies“尚未撰写”、`consume(e)` 待回改、重复提交归因、`e` 的类型、腻度行方向、注册表缺口、Tuning Knobs 等），另有两处由本次修复引入的内部矛盾。
Prior verdict resolved: No — partial（本系统自身 `initialize` 失败已修；`Failed` / `Faulted` 两条终止路径仍未覆盖，且 `run_faulted` 上游从未广播）
Findings:
- [BLOCKING] Detailed Design rule 2 (`:28`) / States (`:48-52`) / Edge `:187`: 局边界订阅 `run_ended` / `run_discarded` / `run_faulted`，但后两者不足以覆盖所有终止路径。(a) `run_faulted` 从不广播：run-flow 的状态转移表 `:96-108` 每一行的“副作用”列是穷举的，进入 `Faulted` 的两行（`:88` 不变式/结算失败、`:106` `run_id` 溢出）都没有“广播 `run_faulted`”；信号只在 `:132`、`:330` 被声明与列为音效事件。(b) 初始化成功、但后续子系统（如菜单波次）初始化失败时，run-flow 走 `Initializing → Failed`（`run-flow.md:96`），既不发 `run_faulted`（那是 `Faulted` 的信号），也不发 `run_discarded`（从未开始的局不发，`run-flow.md:132`），更不发 `run_ended`——账本仍停在 `就绪`。(c) 提交期 `CAPACITY_OVERFLOW`（本文件 `:185`、AC 14）使 run-flow 进入 `Faulted`（`run-flow.md:46`），同样无广播。三条路径的后果一致：下一局 `initialize` 被规则 2 判为同局第二次 → `CAPACITY_INIT_INVALID` → run-flow `Faulted`，此后每局必败，只有重启应用可恢复。规则 2 的“因此下一次 `initialize` 必然是新局”由此仍不成立。修法（三选一）：1) run-flow 在**每一条**进入 `Faulted` 的转移上广播 `run_faulted`，并为 `Failed` 增加等价通知（或显式调用 `end_run("failed")`），本文件改为引用那两个转移；2) 给本系统加显式 `end_run(reason)`，由 run-flow 在**所有**终止路径（含 `Failed` / `Faulted` 清理）调用——这是唯一能覆盖“从未开始”路径的办法；3) 给 `initialize` 加 run identity（`run_id`），无需改上游，但要改已锁定的签名。选定后补一条 AC 同时覆盖 (b) 与 (c)——现有 AC 26 只覆盖本系统自身 `initialize` 失败。
- [RECOMMENDED] 边界描述不一致: 规则 2 与两张 Interactions 表 (`:62`、`:201`) 都写了 `run_faulted`，但 Edge `:187` 仍只说“收到 `run_discarded`（或正常结束的 `run_ended`）”，漏掉第三条。按最终选定的机制改写该条。
- [RECOMMENDED] States `:51` 与 AC 5 冲突: `初始化失败` 行的进入条件包含“同局重复初始化”，离开写“立即回到 `未初始化`”；但 AC 5 (`:275`) 要求该次调用后 `capacity_used` 仍为 `17`，而 `未初始化` 下读取返回 `0` + ERROR（AC 25）。`已丢弃` 行 (`:52`) 同理——它与 `:48` 的“收到信号即回到 `未初始化`”是同一件事，两行都不可达。删除这两行，或把“保持 `used` 直到信号到达”的顺序写清楚。
- [RECOMMENDED] AC 17 (`:293`): 枚举“能改变 `capacity_used` 的方法”时只列 `initialize` 与 `consume`，但边界重置是第三条清零路径（`:28`、AC 24）。把 AC 17 限定为“一局内”，并把边界重置补进枚举或单列一条 AC。
- [RECOMMENDED] 验收缺口: 上一轮要求“增加一条通过信号驱动边界的 AC”，仍只加了 AC 26（失败的 `initialize`）。缺 GIVEN 送达 `run_ended` / `run_discarded` / `run_faulted` WHEN 读取 THEN `未初始化` / `0` + ERROR，以及与“同局第二次”成对的用例，用来证明两条路径确实可区分。
- [RECOMMENDED] Open Question 1 (`:313`)、`:63`、`:253`: 仍称 `C0` 没有数据归属；`data-config.md:59,117,185,246` 已定义并校验 `BalanceData.base_capacity`（1–999，默认 20）。关闭 OQ1，把两处改为 `get_balance() → base_capacity`。OQ4 亦可关闭：`entities.yaml:700-702` 已把本 GDD 列入 `capacity_used.referenced_by`。
- [RECOMMENDED] Dependencies `:214`: 仍说 局外升级 / 局内 HUD / 纪录与里程碑 / 存档 的 GDD “尚未撰写”；四份都存在，且本文件 `:64`、`:66`、`:70` 自己就引用了它们。`records-and-milestones.md` 至今没有提到胃容量，因此 `:70` / `:209` 的软下游行仍是单边。
- [RECOMMENDED] Cross-References `:263`: “待回改：run-flow.md 漏了 `consume(e)`”已过期——`run-flow.md:120,295` 都已列出 `consume(e)`。删除该注。
- [RECOMMENDED] Edge Cases `:186`: 把重复提交的防御归于 `capacity_used = Σ effective_satiety` 不变式；第二次提交若被接受同样会累加，和仍然相等。真正的防御是 run-flow 的第一道校验“菜仍在当前可见集”（`run-flow.md:34`）。改引该条，或把不变式的作用域限定为“只记了一行却调用两次 `consume`”。
- [RECOMMENDED] `e` 的类型: Formulas `:147` 把 `e` 标为 `int`，但 AC 10 (`:283`) / AC 15 (`:291`) 传入 `2.5`。GDScript 的 `e: int` 会在进入守卫前拒绝该参数。声明 `e: Variant`，或写明测试从无类型边界进入。
- [RECOMMENDED] 腻度行方向: `cloying-and-taste.md:62` 仍把本系统记为 `双向` 并称容量决定“吃撑”，与规则 6（无 `STUFFED`）及本文件 `:75`、`:212` 的无调用边方向矛盾。修在对方 GDD（本次范围外）。
- [RECOMMENDED] 注册表 (`entities.yaml`，由编排会话维护): `capacity_remaining` 与 `capacity_bonus` 仍无条目；`capacity_used.referenced_by` (`:700-702`) 仍漏了 `run-flow.md`（其 `:120,295` 读取该值）；`capacity_total` 注 (`:893`) 与 `base_capacity` 注 (`:1272`) 仍称数据归属是胃容量的 Open Question，而 `data-config.md:59` 已拥有该字段。
- [RECOMMENDED] AC 26 (`:305`) 措辞混淆终止状态: “因非法 `capacity_bonus` 失败”只可能让 run-flow 进入 `Failed`（`run-flow.md:96`）；`Faulted` 只由同局第二次 `initialize` 产生。改为只断言 `Failed`，或把两种情形拆成两条。
Advisory (not counted): Tuning Knobs 仍缺失（`standard` 下 advisory）——`C0`（1–999，MVP 20）与 999 结构上限是本系统的两个可调值。上一轮的“`run_ended` 立即重置 vs HUD 读值”冲突已由上游解决：`in-game-hud.md:166` 现在规定 `Ending` 期间读最后一帧快照、不实时查询，本文件无需再改。公式声明 `capacity_total ∈ [1, 999]` 但 `initialize` 前读取返回 `0`（`:192`）——注明区间描述的是已初始化实例。本文件的 `capacity_bonus` 上界 `0–(999 − C0)` 与 `meta-upgrades.md:202` 的 `0–99` 兼容，写出上游上限能让规则 10 的购买时校验具体化。OQ7 可关闭：`audio-feedback.md:167,408` 有“容量变化” cue 而无容量事件，确认由 `action_committed` 派生。`systems-index.md:26-27` 仍说容量 “decides when the player is stuffed”，与 OQ5 / 规则 6 矛盾（索引编辑归编排会话）。

### Completeness: 6/6 required sections present
`modes.workflow: standard` → Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge Cases, Dependencies, Acceptance Criteria all present. Advisory at `standard`: Player Fantasy and Tuning Knobs absent (Game Feel `:229-235` covers the fantasy informally).

### Dependency Graph
All declared dependencies have a GDD on disk: run-flow, data-config, meta-upgrades, cloying-and-taste, in-game-hud, settlement-and-entry-fee, menu-waves, dish-values, records-and-milestones, save-system.

### Registry Values
Whole-document search for the registry's capacity terms (`capacity_total`, `capacity_used`, `capacity_remaining`, `capacity_bonus`, `base_capacity`, `can_fit`, `effective_satiety`) returned 116 lines; no value conflicts. `base_capacity = 20`, `capacity_total ∈ [1, 999]`, `capacity_used ∈ [0, 999]` and the inclusive `can_fit` predicate all match; the 20-cell / 104-yuan calibration still reproduces (ten ratio-optimal dishes sum to 20 cells / 104 yuan, the ten highest-priced to 26). Entry gaps are listed under Recommended item 12.

### Verdict: NEEDS REVISION

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 72584c8a3cc37bfd9c68d285c4223ae8425a63f0
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 5734623b4a60d9072657ef631c8ae61f05359f21
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/menu-waves.md ce532d3265df1ed7555ba5aa46383be35bbee4a1
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 9fe5183795ca3d47798510a1d00a040dabe3ca61
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/systems-index.md c0b526ff05088d426598f5dc34e3bd9e3153cf59

---

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 13
Summary: 第四轮的 `end_run(reason)` 修复解决了此前连续三轮的 blocking。`run-flow.md:52` 规则 24 现在要求在**每一条**终止路径调用一次 `胃容量.end_run(reason)`，`reason ∈ {ended, discarded, failed, faulted}`，并与交互行 `run-flow.md:121` 同步；本文件规则 2（`:28`）、状态表 `未初始化` 行（`:48`）、Edge `:187`、两张接口表（`:62`、`:201`）都已改为以 `end_run` 为权威局边界、信号降为冗余。此前「`Failed` / `Faulted` 后账本停在 `就绪`，下一局被误判为同局第二次、此后每局必败」的失败链不再成立：初始化失败、运行期故障、正常交接完成与放弃/退出四条路径都会先回到 `未初始化`，下一次 `initialize` 必然是新局，且不需要 run identity。剩余 13 项均为同步、措辞与验收覆盖问题，不阻塞实现。
Prior verdict resolved: Yes — 上轮 blocking 已解决（四条终止路径全部由 `end_run` 覆盖）。
Findings:
- [RECOMMENDED] run-flow 转移表 `run-flow.md:93-109`: 规则 24 已是权威，但状态转移表的“副作用”列仍未列出任何 `end_run` 调用——上轮的 blocking 判定正是建立在该列穷举这一前提上。按表施工的实现者仍可能漏调。建议在四条终止转移（`:97` Failed、`:101`/`:103`/`:107` Faulted、`:104` Handoff→Idle、`:106` 丢弃）的副作用列各补一条 `调用 胃容量.end_run(...)`（修改在 run-flow，本次范围外）。
- [RECOMMENDED] States `:52`: `已丢弃` 行已不可达——规则 2（`:28`）说收到任意 `end_run` 立即回到 `未初始化`，该阶段永远不会被占用；进入条件也只列 `run_faulted`，漏了 `ended` / `failed`。删除该行，或改为 `未初始化` 的别名。
- [RECOMMENDED] States `:51`: `初始化失败` 行写“立即回到 `未初始化`”，与 AC 5（`:275`，同局第二次 initialize 后 `capacity_used` 仍为 17）和 AC 25（`:304`，未初始化读取返回 `0` + ERROR）冲突。应写明“保持 `used` 直到 run-flow 进入 `Failed` 并调用 `end_run(failed)`”。
- [RECOMMENDED] 验收缺口: 没有任何 AC 断言 `end_run(ended / discarded / failed / faulted)` 送达后账本处于 `未初始化`、读取返回 `0` + ERROR；AC 26（`:305`）仍把 `Failed` 与 `Faulted` 混写在一句里。补一条参数化用例覆盖四种 reason，并把 AC 26 拆为两条。
- [RECOMMENDED] AC 17（`:293`）: 应限定为“一局内”；`end_run` 是第三条清零路径，未纳入“能改变 `capacity_used` 的方法”枚举。
- [RECOMMENDED] Open Question 1（`:313`）与 `:63`、`:253`: 仍称 `C0` 没有数据归属，但 `data-config.md:59,117,185,246-247` 已定义并校验 `BalanceData.base_capacity`（1–999，默认 20）。关闭 OQ1，两处改为 `get_balance() → base_capacity`。OQ4（`:316`）亦可关闭：`entities.yaml:699-709` 已把本 GDD 列入 `capacity_used.referenced_by`（run-flow 缺失见第 12 条）。
- [RECOMMENDED] Dependencies `:214` 与“待回指” `:261`: 仍称 局外升级 / 局内 HUD / 纪录与里程碑 / 存档 的 GDD “尚未撰写”，四份都存在且本文件 `:64`、`:66`、`:70` 自己就引用了它们；`data-config.md:185` 也已新增胃容量下游行。两处均已过期。
- [RECOMMENDED] Cross-References `:263`: “待回改：run-flow.md 漏了 `consume(e)`”已过期——`run-flow.md:121` 已列出 `consume(e)`。删除该注。
- [RECOMMENDED] Edge Cases `:186`: 把重复提交的防御归于 `capacity_used = Σ effective_satiety` 不变式；第二次提交若被接受同样会累加，和仍然相等。真正的第一道防御是 run-flow 的“菜仍在当前可见集”（`run-flow.md:34`）。改引该条，或把不变式的作用域限定为“只记了一行却调用两次 `consume`”。
- [RECOMMENDED] `e` 的类型: Formulas `:147` 把 `e` 标为 `int`，但 AC 10（`:283`）/ AC 15（`:291`）传入 `2.5`；GDScript 的 `e: int` 会在进入守卫前拒绝该参数。声明 `e: Variant`，或写明测试从无类型边界进入。
- [RECOMMENDED] 腻度行方向: `cloying-and-taste.md:62` 仍把本系统记为 `双向` 并称容量决定“吃撑”，与规则 6（无 `STUFFED`）及本文件 `:75`、`:212` 的无调用边方向矛盾。修在对方 GDD（本次范围外）。
- [RECOMMENDED] 注册表（`entities.yaml`，由编排会话维护）: `capacity_remaining` 与 `capacity_bonus` 仍无条目；`capacity_used.referenced_by`（`:699-702`）仍漏了 `run-flow.md`（其 `:121` 读取该值）；`capacity_total` 注（`:896`）与 `base_capacity` 注（`:1275`）仍称数据归属是胃容量 Open Question，而 `data-config.md:59` 已拥有该字段。`base_capacity.referenced_by` 也漏了 `data-config.md`。
- [RECOMMENDED] `in-game-hud.md:166`: 括注“容量可能在 `run_ended` 重置”与现在的时机不符——重置发生在交接完成后的 `end_run(ended)`，不是 `run_ended`。HUD 因 `Ending` 期间读最后一帧快照而不受影响，但措辞应改（对方 GDD，本次范围外）。
Advisory (not counted): Tuning Knobs 仍缺失（`standard` 下 advisory）——`C0`（1–999，MVP 20）与 999 结构上限是本系统的两个可调值。公式声明 `capacity_total ∈ [1, 999]` 但 `initialize` 前读取返回 `0`（`:192`）——注明区间描述的是已初始化实例。`systems-index.md:26-27` 仍说容量 “decides when the player is stuffed”，与 OQ5 / 规则 6 矛盾（索引编辑归编排会话）。OQ7 可关闭：`audio-feedback.md` 的“容量变化” cue 无容量事件，确认由 `action_committed` 派生。

### Completeness: 6/6 required sections present
`modes.workflow: standard` → Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge Cases, Dependencies, Acceptance Criteria all present. Advisory at `standard`: Player Fantasy and Tuning Knobs absent (Game Feel `:229-235` covers the fantasy informally).

### Dependency Graph
All declared dependencies have a GDD on disk: run-flow, data-config, meta-upgrades, cloying-and-taste, in-game-hud, settlement-and-entry-fee, menu-waves, dish-values, records-and-milestones, save-system. `records-and-milestones.md` still does not mention 胃容量, so the soft downstream row (`:70`, `:209`) remains one-sided.

### Registry Values
Whole-document search for the registry's capacity terms (`capacity_total`, `capacity_used`, `capacity_remaining`, `capacity_bonus`, `base_capacity`, `can_fit`, `effective_satiety`) returned 117 lines; no value conflicts. `base_capacity = 20`, `capacity_total ∈ [1, 999]`, `capacity_used ∈ [0, 999]` and the inclusive `can_fit` predicate all match. Entry and note gaps are listed under Recommended item 12.

### Verdict: APPROVED

---

## 建议索引状态（复审）

**Approved**（`systems-index.md:63` 当前为 `In Review`；按并行约束未改索引，由编排会话统一更新）。`Depends On` 仍为「餐局流程, 数据配置」，与本 GDD 的上游表一致，无需变动。

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 280240fddb3258864af5f96550cc7cbfd49d42eb
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/menu-waves.md 4e4854fb6a38c7b0989503d8c261b69c5746adbe
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 84e5c38fc40fb2a75d8ee608c911bdebf369c5a1
Reviewed-Content-Hash: design/gdd/audio-feedback.md ffb1c54a84af1da89b0228395ee86e89b3e87e30
Reviewed-Content-Hash: design/gdd/systems-index.md 56e9869c70371816e88ed49f24735f67ab3d199d
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
