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

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 跳过）
Blocking items: 1 | Recommended: 13

**陈旧回执（stale receipt）**：上一份 APPROVED（2026-11-04，本日志 :245）的 `Reviewed-Content-Hash` 绑定 `stomach-capacity.md 280240fd…`，当前字节为 `77622412…`。`review-receipts.sh check` 输出 `CHANGED`（目标文档）、`CHANGED`（`design/registry/entities.yaml`）、`CHANGED`（run-flow / data-config / cloying-and-taste / settlement-and-entry-fee / dish-values / menu-waves / meta-upgrades / in-game-hud / records-and-milestones / audio-feedback / systems-index），仅 `.claude/skills/design-review/SKILL.md` 与 `game-concept.md` 为 `UNCHANGED`。差异集中在规则 2（`end_run(reason)` 覆盖四条终止路径）、规则 3（基础 `satiety` 1–6、`effective_satiety` 1–(6 + `cloy_cap`)）、依赖行与新增的 `capacity_bonus` 决策记录（`:175`），全部属 substantive，故按收据规则作一次**全量 lean 复审**，不接受 delta 复审。工作树另有 cloying / data-config / meta-upgrades / menu-waves / registry 的未提交改动，本次已按当前字节核对。

Summary: 焦点 1 的主干成立——规则 2（`:28`）、状态表的 `未初始化` 行（`:48`）、Edge `:189`、两张接口表（`:62`、`:203`）与 `run-flow.md:52` 规则 24、`run-flow.md:121`、ADR-0007 D7、ADR-0012 §3 一致：四条终止路径（`ended` / `discarded` / `failed` / `faulted`）各调一次 `end_run`，覆盖「从未开始」的 `Failed`，`initialize` 因此不需要 run identity；跨局 `initialize` 是正常重开。规则 3 的值域在上游闭合：`data-config.md:33,98,109,159-160` 硬校验 `satiety ∈ [1,6]`，`cloying-and-taste.md:77-110` 给出 `extra_satiety ∈ [0, cloy_cap]` 且 `effective_satiety ∈ [1, 6 + cloy_cap]`（MVP 1–9，默认 `cloy_cap = 3`），`entities.yaml:494-524` 的 `[0,3]` / `[1,9]` 与之一致——焦点 2 的两个端点都闭。焦点 3、4 亦成立：快照缺失 / 负数 / `null` / 非整数一律 `CAPACITY_INIT_INVALID`、越界 `CAPACITY_TOTAL_OUT_OF_RANGE`，不缺省 0、不钳制（`:30`、`:97`、`:183`、`:185`、AC 3/4）；`can_fit` 等号放行（`:31`、`:141`、`:152`、AC 7/16），`consume` 溢出返回 `CAPACITY_OVERFLOW` 且字段不变（`:115`、`:187`、AC 14）。焦点 5 的决策记录数值经复核**完全正确**：`entities.yaml:462` 的 `capacity_ceiling` 表 20→25 为 104 / 106 / 107 / 112 / 116 / 120，增量正是 +2 / +1 / +5 / +4 / +4，L2 价格 17（`meta-upgrades.md:268` 的 12 / 17 / 22 / 30 / 40）对应 +1 元/局，与 `dish-values.md:63` 的 104 元 / 20 格基准不冲突。唯一的 blocking 是状态表 `初始化失败` 行与 AC 5 / Edge `:186` / ADR-0012 的互斥：该行把「同局重复初始化」也写成「立即回到 `未初始化`」，等于把规则 2 明令禁止的静默重置合法化。其余 13 项为跨文档措辞、验收覆盖与注册表同步。

### Findings

| 严重度 | 位置 | 发现 | 违反的权威 |
|---|---|---|---|
| **BLOCKING** | `design/gdd/stomach-capacity.md:51` | 状态表 `初始化失败` 行的进入条件含「同局重复初始化」，离开列写「**立即回到 `未初始化`**；下一次 `initialize` 是新局」。这与同一文档的 Edge `:186`（「`capacity_used` 保持不变」）、AC 5（`:277`，二次 `initialize` 失败后 `capacity_used` 仍为 **17**）以及 AC 25（`:306`，未初始化读取返回 `0` + ERROR）互斥：若真回到 `未初始化`，AC 5 的 17 必然读成 0；而按「下一次 `initialize` 是新局」，规则 2（`:28`）的「同局第二次是契约违例、不得静默重置」就失效——这正是该轮改动要封堵的失败模式。ADR-0012 §2 第 1 步写死了正确语义（`state == READY` 时返回 `CAPACITY_INIT_INVALID`，**不改变任何字段**）。修法：该行改为「保持 `就绪` 与全部字段，仅返回 `CAPACITY_INIT_INVALID`，由餐局流程进入 `Faulted` 并调 `end_run(faulted)` 才清零」，并把「输入非法 / 上限越界」（首次 `initialize` 失败，账本本就是 `未初始化`）与它拆成两行 | `stomach-capacity.md:186`、`:277`、`:306`；ADR-0012 Decision 2 第 1 步与「一次失败的 `initialize` 不把状态留在任何半初始化值上」；`run-flow.md:52` 规则 24（契约违例 → `Faulted`） |
| RECOMMENDED | `design/gdd/stomach-capacity.md:52` | `已丢弃` 阶段在规则 2 下不可达（任何 `end_run` 都立即回 `未初始化`），且进入条件只列 `run_faulted`，漏了 `ended` / `discarded` / `failed`。删除该行或注明它是 `未初始化` 的别名 | 规则 2 `:28`；ADR-0012 §3「四条路径都只丢弃账本」 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:307`（及 AC 全表 `:269-307`） | 验收缺口：26 条 AC 没有任何一条断言 `end_run(ended / faulted)` 送达后账本处于 `未初始化`、读取返回 `0` + ERROR，也没有「恰好一次」或「重复 `end_run` 幂等」的用例；AC 26 仍把 `Failed` 与 `Faulted` 写在一句里。建议一条参数化用例覆盖四个 reason，并把 AC 26 拆为 Failed / INIT-Faulted 两条 | 规则 2 `:28`、`:189`；ADR-0012 §3（幂等）；design-docs「验收标准必须可测」 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:295` | AC 17 枚举「能改变 `capacity_used` 的方法」时只列 `initialize` 与 `consume`，但 `end_run` 是第三条清零路径。把该 AC 限定为「一局内」，并把边界清零单列 | 规则 2 `:28`；ADR-0012 §3、§6 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:34`、`:182` | 错误词表不完整：规则 8 / Edge `:182` 只给 `INVALID_SATIETY` 与 `CAPACITY_OVERFLOW`，未定义「未初始化时调用 `consume`」。ADR-0012 §5 定义了 `NOT_INITIALIZED`（字段不变），GDD 未同步，也没有对应 AC（AC 25 只覆盖读取）。补该码与一条 AC | ADR-0012 Decision 5；规则 8 / 12 的「唯一写入口」契约 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:28` | `end_run` 的 reason 映射按路径类型划分（初始化失败 → `failed`、运行期故障 → `faulted`），但 `run-flow.md:107` 的 `RUN_ID_OVERFLOW` 是「`Faulted` 状态、`phase = INIT`、从未开始」，ADR-0007 D7 明确要求该路径传 `failed`。行为无差别（ADR-0012 §3 说明 reason 无副作用），但措辞应改为「按 `fault.phase` 而非状态名选 reason」 | ADR-0007 D7 的 `Capacity.end_run(reason)` 行；ADR-0012 §3 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:175`、`:317` | 决策记录与 OQ3 引用 `opt_value(C)`（只给 C），但该签名不存在：容量可变的那条扫描是 `capacity_ceiling(C)`（`meta-upgrades.md:252-262`，注册于 `entities.yaml:462`），`opt_value` 的签名是 `opt_value(M, C, T)`。改为 `capacity_ceiling(C)` 并注明扫描区间 20–27+（27 起恒 121）；数值本身无需改 | `meta-upgrades.md:250-262`；`entities.yaml:443-476`；design-docs「平衡值必须链到来源公式」 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:36`、`:64`、`:205`、`:316` | 规则 10 与依赖行要求局外升级「在购买时保证 `C0 + 最大可能加成 ≤ 999`」，但 `meta-upgrades.md:260` 给出的实际数据约束是更紧的 `base_capacity + max_level ≤ 26`，全文没有 999 购买校验（`resolution` 只把等级钳到 `max_level`）。当前 20+5=25 ≪ 999，999 守卫是空转的；应改引 26 / `capacity_ceiling` 约束，并把 OQ2 标记为已被上游回答 | `meta-upgrades.md:203`、`:260-262`；design-docs「Dependencies 必须双向」 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:216` | 「双向一致性」称 9 份 GDD 均已把本系统列入依赖或下游表，实测只有 7 份：`records-and-milestones.md` 全文无「胃容量」，`menu-waves.md` 也不引用本系统（它直接读 `data-config` 的 `base_capacity`，见 `menu-waves.md:174`）。两处都是软 / 设计期边，改法是把该句降级为「7 份已回指，2 份待回指」或驱动对方补行，不要声称已完成 | design-docs「Dependencies 必须双向」 |
| RECOMMENDED | `design/gdd/stomach-capacity.md:94`、`:169` | 公式表把 `capacity_bonus` 的值域写作 `0–(999 − C0)`，而上游声明 `0–99`（`meta-upgrades.md:203`）、MVP 设计带 `0–5`（`max_level 5 × effect_value 1`）。数值兼容，但应标明本文件写的是**结构上界**，生产者上界在局外升级，否则读者会以为一次加成可达 979 | `meta-upgrades.md:276-279`；`entities.yaml`（`upgrade_price` / `capacity_ceiling`） |
| RECOMMENDED | `design/gdd/stomach-capacity.md:318` | OQ4 已部分落地：`entities.yaml:719-720` 已把本 GDD 列入 `capacity_used.referenced_by`。剩余只有 `source` 归属一项，按 OQ1 的做法标注为已采纳 / 已解决，避免复审时重复开单 | `entities.yaml:714-727` |
| RECOMMENDED | `design/registry/entities.yaml:899-929`、`:1286-1296` | 注册表缺口（编排会话维护，本次不改）：`capacity_remaining`（本文件 `:119`，HUD 容量条分母）与 `capacity_bonus`（`:156`，局外升级拥有）无条目；`capacity_total`（`:911`）与 `base_capacity`（`:1296`）的 notes 仍称数据归属是胃容量 Open Question，而 `data-config.md:59` 已拥有 `BalanceData.base_capacity` | 本文件 `:119`、`:156`；`data-config.md:59`；ADR-0012 Related |
| RECOMMENDED | `design/gdd/cloying-and-taste.md:62` | 对方依赖行仍把本系统记为 `双向` 并称容量决定「吃撑」，与规则 6（无 `STUFFED`、无第三种结束原因）矛盾。修在对方 GDD（本次范围外），但本文件 `:75` 的无调用边表述应保留并在复审时持续跟踪 | 本文件 `:32`、`:75`；`run-flow.md:36` 规则 8 |
| RECOMMENDED | `design/gdd/meta-upgrades.md:268` | 上游的 `value(L)` 估计带写「2–5 元/局」，与决策记录 `:175` 引用的实测边际（L2 仅 +1）矛盾——本文件的数字才是对的（与 `capacity_ceiling` 一致）。应在上游把该带改成逐级值或注明非单调，否则 `/prototype` 的验收口径会被「2–5」误导 | `entities.yaml:462`；本文件 `:175`、`:317` |

Advisory（不计入推荐数）：`Tuning Knobs` 仍缺失（`standard` 下 advisory）——`C0`（1–999，MVP 20）与 999 结构上限是本系统的两个可调值，补一节写安全区间会顺手暴露第 7 条；`Player Fantasy` 无独立标题，`## Game Feel` `:231-237` 覆盖了幻想（满格不是失败）。公式声明 `capacity_total ∈ [1, 999]` 而 `initialize` 前读取返回 `0`（`:194`、AC 25）——注明区间描述已初始化实例。`systems-index.md:64` 记 `Approved` 但本文件头仍 `In Design`（`:3`），且 `systems-index.md:26` 仍说容量 “decides when the player is stuffed”，与 OQ5 / 规则 6 矛盾——索引归编排会话。

### Completeness: 6/6 required sections present
`modes.workflow: standard` → Overview、Detailed Rules（`## Detailed Design`）、Formulas、Edge Cases、Dependencies、Acceptance Criteria 全部存在；本系统定义数值规则（容量上限、`consume` 校验、可吃性谓词），故 Formulas 为 REQUIRED 且已满足（`:79-175`，五个公式均含变量表、值域与算例）。ADVISORY 缺口：Player Fantasy、Tuning Knobs（见上）。

### Dependency Graph
声明的 10 个依赖在磁盘上均有 GDD：`run-flow.md`、`data-config.md`、`meta-upgrades.md`、`cloying-and-taste.md`、`in-game-hud.md`、`settlement-and-entry-fee.md`、`menu-waves.md`、`dish-values.md`、`records-and-milestones.md`、`save-system.md`。反向回指：7 份已列本系统，`records-and-milestones.md` 与 `menu-waves.md` 未列（软 / 设计期边，见上表第 9 条）。

### Registry Values
按 Phase 3 的全文档检索（`capacity_total|capacity_used|capacity_remaining|capacity_bonus|base_capacity|can_fit|effective_satiety|extra_satiety|cloy_cap|satiety|opt_value|capacity_ceiling`，大小写不敏感）命中 **124** 行，逐行比对**无数值冲突**：`capacity_total ∈ [1,999]`、`base_capacity = 20`（`entities.yaml:1294`）、`can_fit` 等号放行（`entities.yaml:926-927`）、`effective_satiety ∈ [1,9]`（`:520`，MVP `cloy_cap = 3`）、`extra_satiety ∈ [0,3]`（`:505`）、`capacity_used ∈ [0,999]`（`:722`；本系统运行期更窄的 `[0, capacity_total]` 与之兼容）。`opt_value` 的 104 元 / 20 格基准（`dish-values.md:63`、`:215`）与 OQ3 的「价格最高 10 道 26 格」均复算正确；D5 的 +2/+1/+5/+4/+4 与 `entities.yaml:462` 的 `capacity_ceiling` 表逐值一致。条目与注释缺口见上表第 12 条。

### Verdict: NEEDS REVISION

建议索引状态：**Needs Revision**（`systems-index.md:64` 当前为 `Approved`；2026-11-04 的那份 APPROVED 绑定旧字节，按 `.claude/rules/design-docs.md` 的收据规则已失效。按并行约束本次未改索引与文件头，由编排会话统一更新；blocking 修好并复审后可回到 `In Review`）。

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/registry/entities.yaml f7b5aded130f371160d84bdaf678f8530ddc0807
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/data-config.md 9ecc2a7c18aba33c64bc3391594748e1063851d2
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/menu-waves.md 4282c9a66552dbb5023418b9c19542c1b5787af3
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/in-game-hud.md fee7199048c3a124fcab2eccae8b88f93d1a69c0
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 213f8dbe40b44748fc32857680f09991fff9d612
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: docs/architecture/adr-0012-stomach-capacity-ledger.md c2bef2e743e052fe85feb7b145de41591273cedf
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0014-cloying-taste-state-machine.md 65b2a675e3a12378e41a021949a4aa1342e15bee
Reviewed-Content-Hash: design/gdd/gdd-cross-review-2026-11-04.md c6efa8393fa2f8c10b80d5964b75aa9793f7d042

## Review — 2026-11-17 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 6
Summary: 全文 lean 复审（非增量）。上一轮唯一的 blocking 已修好，且修法正是建议的拆行：状态表现在有 `:51`「初始化失败（输入非法 / 上限越界）」与 `:52`「同局重复初始化（契约违例）」两行，后者明写"保持 `就绪` 与全部字段不变，直到餐局流程调 `end_run(faulted)` 才清零"，与 ADR-0012 §2 第 1 步（`state == READY` → `CAPACITY_INIT_INVALID`、不改字段）、本文件 Edge `:187`、AC 5、AC 26 一致，静默重置的漏洞已封。`已丢弃` 阶段整行删除，grep 在三份权威范围内（两个 GDD + 架构目录）不再命中该词（`run-flow.md:82` 的"已丢弃"是 `Idle` 的进入条件描述，与容量无关）。`end_run(reason)` 在 `:55` 补成唯一清零点，reason 按 `fault.phase` 选（`failed` = INIT 含 `run_id` 溢出、`faulted` = RUN），与 ADR-0007 D7 的 `Capacity.end_run(reason)` 行逐字对应；新增的 `:192` 无合法动作边界把 `turns_remaining = 1` ∧ 非最后一波 ∧ 全部 `can_fit = false` → `NO_EATABLE_DISH` 归于餐局流程，并重申容量只被调一次 `end_run(ended)`，与 `run-flow.md:36` 规则 8、ADR-0013 §6 `_no_legal_action()` 同义；新增 AC 27（四类 `end_run` + 幂等）与 AC 28（未初始化时 `can_fit` / `consume`）补上了上轮的覆盖缺口；AC 17 加了"跨局清零只有 `end_run`"的 carve-out。剩余 6 项均为接口清单措辞与验收写法，不需要结构性返工。
Prior verdict resolved: Yes —— 2026-11-06 的 1 个 blocking（`初始化失败` 行把同局重复初始化写成"立即回到 `未初始化`"）已按建议拆行修复；上轮 13 条 recommended 中 9 条已落地（`已丢弃` 删除、`NOT_INITIALIZED` 有了 AC、AC 17 加 carve-out、reason 改按 `fault.phase` 选、OQ1 标已解决、注册表 `capacity_used` 补 `referenced_by`），余者见下。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/stomach-capacity.md`、`design/registry/entities.yaml`、`run-flow.md`、`data-config.md`（本次仅会话内实测这四项 + 未变项）；`UNCHANGED`：`SKILL.md`、`cloying-and-taste.md`、`settlement-and-entry-fee.md`。上一份 NEEDS REVISION（2026-11-06）绑定 `stomach-capacity.md 77622412…`，当前字节为 `43fc521a…`；目标文档与跨文档事实（餐局流程、注册表）均变动，故按全文复审而非 delta。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全。Formulas（`:80` 起）五个公式各有变量表、值域、输出范围与算例，可求值；`capacity_total ∈ [1, 999]` 与 `consume` 的拒绝面闭合。Advisory（不计入）：无 `Player Fantasy` 标题（`## Game Feel` 覆盖）、无 `Tuning Knobs`（`C0` 与 999 结构上界事实上是两个旋钮）。

### Findings

| # | 级别 | 位置（file:line） | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | RECOMMENDED | `design/gdd/stomach-capacity.md:38` | 规则 12「只读暴露」写"对外**只提供** `capacity_total`、`capacity_used`、`capacity_remaining` 与 `can_fit`"，把 `consume`（规则 8 的"唯一写入口"）与 `initialize` / `end_run` 全部排除在"对外接口"之外，与 AC 17（`:296`）「局内只有 `initialize` 与 `consume`、跨局只有 `end_run`」正面冲突。按本句实现会没有一个写入面 | 本文件规则 8（`:34`）、`:63`/`:74` 接口表、AC 17；design-docs「规则之间不得自相矛盾」 |
| 2 | RECOMMENDED | `design/gdd/stomach-capacity.md:57`、`:50` | `:57`"重开是全新一局：新的 `initialize` 把 `used` 置 0" 与 `:50` 已满行"仍接受下一局的 `initialize`"都没写前提 `end_run(reason)`；而 `:52` 与 Edge `:187` 规定账本为 `就绪` 时第二次 `initialize` 返回 `CAPACITY_INIT_INVALID`。两句合读会得出"重开不需要 `end_run`"的错误路径 | 本文件 `:55`（唯一清零点）、`:52`、`:187`；ADR-0012 §3 |
| 3 | RECOMMENDED | `design/gdd/stomach-capacity.md:309`（AC 27） | "`WHEN` 依次收到 `end_run(ended)` / `(discarded)` / `(failed)` / `(faulted)`（四个夹具各一次），`THEN` 每次都回到 `未初始化`"——从 `就绪` 出发只有**第一次**是状态迁移，后三次落在 `:55` 的幂等分支，"每次都回到 `未初始化`"对后三次不可判定。应拆成四个独立夹具、各自从 `就绪` 开始，再单列一条断言重复 `end_run` 幂等 | design-docs「验收标准必须可独立判定」；本文件 `:55` |
| 4 | RECOMMENDED | `design/gdd/stomach-capacity.md:310`（AC 28） | 只断言 `can_fit` 返回 `false`，没断言 `can_fit` 的 ERROR；ADR-0012 §4 是"`state == UNINITIALIZED` → `can_fit` 返回 `false` **并记 ERROR**"。AC 25（`:307`）只覆盖三个读数的 ERROR，写入口与判定面的 ERROR 无人验 | ADR-0012 §4（`adr-0012-stomach-capacity-ledger.md`）；本文件 Edge `:195` |
| 5 | RECOMMENDED | `design/gdd/stomach-capacity.md:34`、`:183` | 规则 8 与 Edge 的错误码表仍只有 `INVALID_SATIETY` / `CAPACITY_OVERFLOW`，未点名 `NOT_INITIALIZED`（该码只出现在 AC 28）。ADR-0012 §5 定义了四个 `ConsumeStatus`，GDD 的写入口契约应给全 | ADR-0012 §5；本文件 AC 28（`:310`） |
| 6 | RECOMMENDED | `design/gdd/stomach-capacity.md:296`（AC 17） | 括号里"不可被下游直接调用"在**本文件的方向约定**（`:215`：餐局流程记作**上游**）下技术自洽，但读者按常识会把唯一的调用者（餐局流程）当成下游，从而把该句读成"连餐局流程也不许调 `end_run`"。建议直写"只允许餐局流程在终止路径调用，其它系统不得调用" | 本文件 `:215`、`:28`、`:63`；ADR-0012 §3 |

### 已核对且无问题（记录以免重复检查）

- **初始化失败 vs 重复初始化**：`:48`/`:51`/`:52` 三行与 ADR-0012 §2.1 的"失败不留在半初始化值上"一致；`:187` 的括号（"若第一次 `initialize` 就失败，账本保持 `未初始化`，下一次是新局"）把两类情形彻底分开，AC 26（`:308`）与之一致。
- **`end_run` 契约**：`:28` 规则 2 要求"每一条终止路径调用一次"，`:55` 的四 reason 与 `fault.phase` 映射、幂等、零化两字段，与 ADR-0012 §3、ADR-0007 D7 的 `Capacity.end_run(reason)` 行、`run-flow.md` 规则 24 一致；`UNINITIALIZED` 再调不改变字段与 ADR-0012 §3 的幂等声明一致。
- **容量不结束本局**：规则 6（`:32`）、`已满` 行的派生声明（`:50`）、Edge `:181`、`:192` 的四次否定（不产生 reason、不调结算、不是第三种结束原因、`NO_EATABLE_DISH` 由餐局流程判）与 `run-flow.md:36`、ADR-0013 §6 完全对齐，无一处把结束权交给本系统。
- **可吃性与等号**：规则 5（`:31`）、Edge `:182`、AC 16（`:295`）都是"`used + e ≤ total` 含等号"，与 ADR-0012 §4 的 `can_fit` 公式、`entities.yaml:932` 的 `(e >= 1) and ...` 一致；`consume` 越界返回 `CAPACITY_OVERFLOW` 且不钳制（`:188`、AC 14）。
- **无 `STUFFED` / 无信号**：全文无 STUFFED 状态、无 `capacity_changed`（`:78` 明确不广播），与 ADR-0012 §6、`in-game-hud.md` 的只读投影一致。
- **数值面**：`capacity_total ∈ [1,999]`、`base_capacity = 20`、`effective_satiety ∈ [1,9]`、`extra_satiety ∈ [0,3]` 与 `entities.yaml`（`:720`、`:905`、`:1294` 等）逐值一致，未见冲突；104 元 / 20 格基准与 `capacity_ceiling` 表的 +2/+1/+5/+4/+4 复算仍正确。

### 跨文档待办（不属于本 GDD，仅记录）

- `entities.yaml:917`（`capacity_total.notes`）与 `:1302`（`base_capacity.notes`）仍写"data ownership is stomach Open Question 1"，而本文件 `:318` 已把 OQ1 标为**已解决**（`BalanceData.base_capacity` 已定义并在启动校验）；注册表注释落后于设计，由编排会话更新。
- `cloying-and-taste.md` 的依赖行仍把本系统记为"双向"并称容量决定"吃撑"（上轮第 13 条，仍未修；本文件规则 6 已三次否认）。
- `systems-index.md:64` 记 `Approved`、`:26` 概述仍说容量"decides when the player is stuffed"；本文件头 `:3` 为 `In Review 2026-11-06`。索引与头部由编排会话统一同步，本次未改。

### Suggested systems-index status

**Approved**（0 blocking）。索引与文件头的状态同步（`systems-index.md:26` 的"stuffed"措辞应按 OQ5 / 规则 6 改写）留给编排会话；本文件可随下次头部同步转入 `Approved`。

### Verdict: APPROVED

---

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 43fc521a857c5ce53ba3e1855c271ae2ffdfb1b6
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: docs/architecture/adr-0012-stomach-capacity-ledger.md c2bef2e743e052fe85feb7b145de41591273cedf
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION（第 4 轮：规则 6「无合法动作」例外后的全量 lean 复审）
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 1 | Recommended: 13

Summary: 全文 lean 复审（41.3 KB / 326 行逐节重读，非 delta）。**规则 6 的例外本身正确、精确且可执行**：非最后一波 ∧ `turns_remaining = 1` ∧ 可见菜都放不下 → 餐局流程按 `NO_EATABLE_DISH` 收尾，并与四处权威逐字同义——`run-flow.md:36` 规则 8、`run-flow.md:201` 的 `end_reason` 公式与 `:209` 的 `_no_legal_action()` 变量行、ADR-0013 §6 `:195-211`（含 `is_final_wave()` 与 `turns_remaining != 1` 两道早退）、`entities.yaml:900-902` 的 `expression` / `notes`（"Capacity alone never ends a run"）。「那也不是容量产生的结束原因」与 Edge `:192`「本系统只被调一次 `end_run(ended)`」也正确，容量侧语义没有被这次修改破坏。**但修复只落在规则 6 一处**：同一句旧口径仍有两个未加例外的副本——Edge `:181`「本局保持 `Active`…玩家仍可付费跳波」与公式算例 `:155`「但本局继续」。二者在可达夹具（吃下一道菜把回合从 2 用到 1 且容量恰好填满、非最后一波）下为假：`_no_legal_action()` 恒真、`skip_wave` 先被 `NO_TURNS` 拒，本局同帧结束。这与 `run-flow-review-log.md:357` B3 判 blocking 的缺陷同形，故本轮不能放行。其余 13 项为 AC 夹具约束、接口清单措辞、双向回指与注册表同步。
Prior verdict resolved: No —— 2026-11-17 的第 3 轮 APPROVED（绑定 `stomach-capacity.md 43fc521a…`）已被本轮规则 6 的修改作废（`.claude/rules/design-docs.md`「Review receipts and post-approval edits」：改规则 = substantive）。该轮的核心结论（状态表拆行、`end_run` 唯一清零点、`NOT_INITIALIZED`、AC 27/28）经复核仍成立；但上轮 6 条 recommended 只有第 3 条（AC 27 拆四夹具）落地，且当时没有发现 Edge `:181` 与 `:155` 的无条件「保持 `Active`」口径与 `_no_legal_action()` 冲突——本轮据此重开。

### Findings

| # | 级别 | 位置（file:line） | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | **BLOCKING** | `design/gdd/stomach-capacity.md:181`（同源副本 `:155`） | 「**If `capacity_used = capacity_total`**：`can_fit(e)` 对任何 `e ≥ 1` 返回 `false`；本局保持 `Active`，不产生结束原因；玩家仍可付费跳波、等下一波或耗尽回合。」这一句在本轮修复后成了全文唯一一处**无条件**把「容量满」等同于「本局留在 `Active`」的规则陈述，也是 `run-flow-review-log.md:357` B3 判 blocking 的同形缺陷。可达反例：非最后一波、吃下一道菜把 `turns_remaining` 从 2 用到 1 且 `capacity_used` 恰好到 20 → 提交后同帧判定，`no_eatable = false` 但 `_no_legal_action()` 为真 → `NO_EATABLE_DISH`、进入 `Ending`；此时「本局保持 `Active`」与「玩家仍可付费跳波」**同时为假**（`skip_wave` 在 `turns_remaining = 1` 被 `NO_TURNS` 拒绝，`menu-waves.md:31` / `time-and-turns.md:53`）。公式算例 `:155`「已用 20 → `can_fit(1) = false`，**但本局继续**」是同一口径的第二个副本。修法：两处各补一句限定（「容量本身不结束本局；若此时已无任何合法动作，例外见 Edge `:192` / 规则 6」），`skip_wave` 可用性改写成「`turns_remaining ≥ 2` 时仍可付费跳波，最后一波仍可免费结束取餐」 | 本文件规则 6 `:32`；`run-flow.md:36` 规则 8、`:263-265` 边角案例、AC 43 `:440`；ADR-0013 §6 `:210-211`；`entities.yaml:902`；`time-and-turns.md:53`、AC 23a `:293` |
| 2 | RECOMMENDED | `:32`（规则 6） | 例外虽在同一规则内，但「餐局流程保持 `Active` 并让玩家付费跳波、等下一波或用掉剩余回合」这一分句本身不带限定，要读到下一句「唯一例外是…」才收紧；快读者在规则 6 内部就先得到一个无条件结论，这正是 B1 两个副本的成因。建议把限定内联进该分句（「除本规则末句的例外」），并删去「等下一波」——推进波次只有 `skip_wave` / `end_pickup` 两条动作，`## Game Feel` `:237` 也写「没有自动推进」 | 本文件 `:181`、`:192`、`:237`；`run-flow.md:36` |
| 3 | RECOMMENDED | `:284`（AC 8） | `GIVEN capacity_used = 20、capacity_total = 20` 未限定 `turns_remaining` 与波次，`THEN` 却断言「餐局流程仍为 `Active`…没有产生任何结束原因」。测试替身（`:270` 只提供 `Active` / `Faulted` 探针）下该断言恒真、等于空验；真实集成下若夹具取 `turns_remaining = 1`、非最后一波，该状态在产生它那次提交后同帧已结束，夹具本身不可构造。补「且 `turns_remaining ≥ 2`（或最后一波）」 | 规则 6 `:32`；ADR-0013 §6；design-docs「验收标准必须可独立判定」 |
| 4 | RECOMMENDED | `:192`（Edge） | 新例外是本轮唯一新增的跨系统行为，但「那仍不是本系统的结束原因，本系统只被调一次 `end_run(ended)`」这一容量侧后果**没有任何 AC 覆盖**（AC 1–28 无一条经过 `NO_EATABLE_DISH` 路径）。建议补一条：本局按 `NO_EATABLE_DISH` 结束时容量收到恰好一次 `end_run(ended)`，此后三个读数返回 `0` + ERROR | ADR-0012 §3；本文件 `:55`、`:192` |
| 5 | RECOMMENDED | `:38`（规则 12） | （carried，上轮 #1）「只读暴露：对外**只提供** `capacity_total`、`capacity_used`、`capacity_remaining` 与 `can_fit`」把 `consume`（规则 8 的唯一写入口）、`initialize`、`end_run` 全排除在「对外接口」外，与 AC 17（`:296`）正面冲突。改为「只读**查询**面为…；写入面仅 `initialize` / `consume` / `end_run`」 | 本文件规则 8 `:34`、AC 17；`entities.yaml:922-936` |
| 6 | RECOMMENDED | `:50`、`:57` | （carried，上轮 #2）`已满` 行「仍接受下一局的 `initialize`」与「重开是全新一局：新的 `initialize` 把 `used` 置 0」都缺前提 `end_run(reason)`；与 `:52`、Edge `:187`（`就绪` 时第二次 `initialize` 返回 `CAPACITY_INIT_INVALID`）合读会推出「重开无需 `end_run`」 | 本文件 `:55`（唯一清零点）、`:52`；ADR-0012 §3 |
| 7 | RECOMMENDED | `:34`（规则 8）、`:183`（Edge） | （carried，上轮 #5）错误码表仍只有 `INVALID_SATIETY` / `CAPACITY_OVERFLOW`，未点名 `NOT_INITIALIZED`（该码只出现在 AC 28）。ADR-0012 §5 `:162` 的 `ConsumeStatus` 是四值封闭枚举，写入口契约应给全 | ADR-0012 §5；本文件 AC 28 `:310` |
| 8 | RECOMMENDED | `:310`（AC 28） | （carried，上轮 #4）只断言 `can_fit` 返回 `false`，未断言其 ERROR；ADR-0012 §4 `:154` 是「`state == UNINITIALIZED` → 返回 `false` **并记 ERROR**」。AC 25（`:307`）只覆盖三个读数的 ERROR | ADR-0012 §4；本文件 Edge `:195` |
| 9 | RECOMMENDED | `:296`（AC 17） | （carried，上轮 #6）「不可被下游直接调用」在本文件的方位约定（`:215` 把餐局流程记作**上游**）下自洽，但按常识读者会把唯一调用者当成下游，从而读成「餐局流程也不许调」。直写「只允许餐局流程在终止路径调用」 | 本文件 `:215`、`:28`、`:63` |
| 10 | RECOMMENDED | `:217`、`:212` | （carried，2026-11-06 第 9 条）「双向一致性」称 9 份 GDD 均已把本系统列入依赖或下游表，实测 `records-and-milestones.md` 与 `menu-waves.md` 全文 0 次「胃容量」（`grep -c` = 0）；`:212` 却给纪录与里程碑行标注「GDD 已写」。该句应降级为「7 份已回指，2 份待回指」，或删掉 `:212` 的括注 | design-docs「Dependencies 必须双向」 |
| 11 | RECOMMENDED | `:308`（AC 26） | （carried）「一次 `initialize` 因非法 `capacity_bonus` 失败、餐局流程进入 `Failed` / `Faulted`」把两种终态混写：非法快照只走 `Failed`（`phase = INIT`），`Faulted` 是 RUN 期不变式失败。测试者按后半句建夹具会得到 `CAPACITY_INIT_INVALID` 却期望 `Faulted` | 本文件 `:28`、`:51`、`:187`；`run-flow.md:46` 规则 18 |
| 12 | RECOMMENDED | `:251-262`（Cross-References） | 表内只有 GDD 与索引，**没有任何 ADR 行**，而正文已引 `ADR-0013 §6`（规则 6 `:32`）与 `ADR-0012 §5`（AC 28 `:310`）；本轮修复把 ADR-0013 从「隐性对齐」变成「显式依据」，缺行会让读者以为容量结尾只由 run-flow 定义。补两行（ADR-0012 台账、ADR-0013 §6 判据） | 本文件 `:32`、`:310`；`docs/architecture/adr-0013-action-pipeline-end-judge.md` §6 |
| 13 | RECOMMENDED | 跨文档：`docs/architecture/adr-0012-stomach-capacity-ledger.md:182` | 容量的自家 ADR §6 仍写「`capacity_used == capacity_total` 只是派生读数…**本局保持 `Active`**」，与 ADR-0013 §6 `:210`、`run-flow.md:36` 规则 8 冲突（同形旧口径）。本 GDD 修好 Edge `:181` 后反而会与自家 ADR 不同形，建议同批给 ADR-0012 §6 补「除 `_no_legal_action()` 外」的限定（ADR 不在本次可改范围） | ADR-0013 §6；`run-flow.md:36`；`entities.yaml:902` |
| 14 | RECOMMENDED | 跨文档：`entities.yaml:918`、`:1303`、`:722-726`；`systems-index.md:26`、`:64`；`cloying-and-taste.md:62` | 注册表与索引未跟上：`capacity_total` / `base_capacity` 的 notes 仍称「data ownership is stomach Open Question 1」，而本文件 `:318` 已标 `已解决`；`capacity_used.referenced_by` 仍缺 `run-flow.md`（其规则 12 读该值）；`capacity_remaining` / `capacity_bonus` 仍无条目；`systems-index.md:64` 记 `Approved` 而本文件头为 `In Review`，`:26` 仍写容量 "decides when the player is stuffed"；`cloying-and-taste.md:62` 仍记「双向 / 吃撑」。均由编排会话统一处理 | 本文件 `:3`、`:318`；ADR-0012 Related |

Advisory（不计入推荐数）：`Player Fantasy` 与 `Tuning Knobs` 仍无独立标题（`standard` 下 advisory；`## Game Feel` `:232-238` 覆盖幻想，`C0` 与 999 结构上界事实上是两个旋钮，补一节会顺手暴露规则 10 与上游 `capacity_ceiling` 的 26 上限错位——该错位在 2026-11-06 那轮已开单、2026-11-17 第 3 轮未再提及，本轮亦不阻塞）。`:180`「该菜本局不可吃；不记错误、不结束本局」按「本系统不结束」读成立，与 B1 不同源，不单列。

### 已核对且无问题（记录以免重复检查）

- **规则 6 的例外本体**：三道条件（非最后一波 / `turns_remaining = 1` / 无可见菜 `can_fit`）与 ADR-0013 §6 的 `_no_legal_action()` 逐条对应，无多无少；结束原因取 `NO_EATABLE_DISH`，不新增第三枚举、不改 `no_eatable`（`run-flow.md:36` 明写不改变信号），与 `entities.yaml:901` 的 `no_eatable or no_legal_action` 一致。
- **例外的容量侧后果**：`end_run(ended)` 一次、随后回 `未初始化`、读数 0 + ERROR（`run-flow.md:52` 规则 24 的 `ended` 分支、ADR-0012 §3 `:138-141`）——`run-flow.md:52` 把 reason 定为「按 `fault.phase` 而非状态名」，`NO_EATABLE_DISH` 走正常交接，因此必须是 `ended` 而不是 `faulted`，本文件 `:192` 写对了。
- **没有其他地方把结束权交给本系统**：规则 6、`已满` 行派生声明（`:50`、`:54`）、Edge `:180`、`:192`、AC 8 的「不存在 `STUFFED` 枚举」五处都是否定式；全文无 `STUFFED`、无 `capacity_changed`、无 `end_reason` 写入口（规则 6 只说「不产生」）。
- **`is_exhausted()` 面**：本文件只在规则 6 的例外里引用 `turns_remaining`，不复制 `is_exhausted()` 判据、不写回合耗尽语义，`time-and-turns.md:43` 规则 8「只报告不裁决」的分工未被越界。
- **数值面**：`capacity_total ∈ [1,999]`（`entities.yaml:916`）、`base_capacity = 20`（`:1296`）、`can_fit` 含等号且 `e > capacity_total` 不记错（`:933-934`）、`effective_satiety ∈ [1,9]`、`extra_satiety ∈ [0,3]`、104 元 / 20 格基准与 `capacity_ceiling` 的 +2/+1/+5/+4/+4 复算均与注册表逐值一致。
- **`待回改：无`（`:266`）仍准确**：`run-flow.md:34` 规则 6 已含 `can_fit` / `consume(e)`，`data-config.md:59`、`:197` 已定义 `base_capacity` 并有胃容量下游行。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

`gdd-structure-check.sh` 输出 `PRESENT: Overview, Detailed Rules, Formulas, Edge Cases, Dependencies, Acceptance Criteria`（6/6 必需齐全，Formulas 因本系统定义容量上限与判定谓词而 REQUIRED），`ABSENT: Player Fantasy, Tuning Knobs`（advisory）。Formulas 五个公式各有变量表、值域、输出范围与算例，可求值。

### Dependency Graph

声明的 10 个依赖在磁盘上均有 GDD：run-flow、data-config、meta-upgrades、cloying-and-taste、in-game-hud、settlement-and-entry-fee、menu-waves、dish-values、records-and-milestones、save-system。反向回指：8 份提到「胃容量」，`records-and-milestones.md` 与 `menu-waves.md` 为 0（软 / 设计期边，见 R10），`save-system.md` 为 0（`:217` 已自认无行）。Interactions 表（`:61-72`）与 Dependencies 表（`:202-213`）逐行一致，仅措辞详略不同，未发现方向或硬软度分歧。

### Registry Values

按 Phase 3 的全文档检索（`capacity_total|capacity_used|capacity_remaining|capacity_bonus|base_capacity|can_fit|effective_satiety|cloy_cap|extra_satiety|capacity_ceiling|end_reason|no_eatable|end_run`，大小写不敏感）命中 **130** 行，逐行比对**无数值冲突**；`end_reason` 的两处取值域（`NONE | TIME_EXHAUSTED | NO_EATABLE_DISH`）与 `settlement-and-entry-fee.md:65`、`records-and-milestones.md:84`、ADR-0018 `:220` 一致。条目与注释缺口见 R14。

### 判定说明

判 NEEDS REVISION 而非 APPROVED 的唯一理由是 R1：本轮要修的那句「当前可见菜都放不下时餐局流程保持 `Active`」在规则 6 里被修好了，但**同一句话的两个副本没跟着改**（Edge `:181`、算例 `:155`）。它不是措辞瑕疵：按字面施工的人会在「容量满 + 最后一回合 + 非最后一波」这一可达状态下实现出「留在 `Active`」的分支，而该状态下 `skip_wave` 已被 `NO_TURNS` 拒绝、`eat` 全被 `CAPACITY` 拒绝，界面上不存在任何可提交动作——正是 `time-and-turns-review-log.md:297` 称为「停在 `Active` 且无合法动作」的僵局，也是 `run-flow-review-log.md:357` 给 B3 判 blocking 的理由。本仓库的先例是一致的：同一缺陷在规则里出现一次算 blocking，在 copy 里再出现一次也算。修复面很小（三处句子、零公式、零数值），不涉及结构返工；其余 13 项均可随同批处理。

### Scope Signal

Rough scope signal: **S**（与前三轮一致——单系统账本，5 个公式皆为整数加减无除法，无新 ADR）。注：按本 skill 的字面刻度（3+ 公式 / 5+ 依赖）也可读作 L，但公式复杂度与集成面都落在 `can_fit` / `consume` 两个方法上，实际实现量仍按 S 计；producer 排期前可自行复核。

### Verdict: NEEDS REVISION

建议索引状态：**Needs Revision**（`systems-index.md:64` 当前为 `Approved`、本文件头 `:3` 为 `In Review`；按并行约束本次未改索引与文件头，由编排会话统一更新）。R1 修好并复审后可回到 `In Review → Approved`。

---

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 93968bd02251e306f5b99c7b1cc5800272e54a06
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION（第 5 轮：无合法动作例外全量落实后的 lean 复审）
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 1 | Recommended: 13

**回执状态**：`review-receipts.sh check` 报 `CHANGED: design/gdd/stomach-capacity.md`；本会话另见 `data-config.md` / `dish-values.md` / `audio-feedback.md` 相对第 4 轮的回执也是 `CHANGED`（本轮只按 `base_capacity`、`opt_value = 104` / 20 格基准、`satiety ∈ [1,6]` 抽查，未作全量比对）；其余全部 `UNCHANGED`——`entities.yaml`、`SKILL.md`、`run-flow.md`、`time-and-turns.md`、`menu-waves.md`、`in-game-hud.md`、`cloying-and-taste.md`、`settlement-and-entry-fee.md`、`meta-upgrades.md`、`records-and-milestones.md`、`systems-index.md`、`game-concept.md`、`design-docs.md`、ADR-0007 / ADR-0012 / ADR-0013 / ADR-0014、`gdd-cross-review-2026-11-04.md`。第 4 轮（2026-11-17，NEEDS REVISION）绑定 `stomach-capacity.md 93968bd0…`，当前字节 `fe066709…`；目标文档变动，按 skill Phase 1 走**全量**复审，不接受 delta。

Summary: 第 4 轮的 blocking 已按建议改写，且改得对：规则 6（`:32`）把「餐局流程是否保持 `Active`」明确绑定到「是否仍有合法动作」并一次列全三道判据（非最后一波 / `turns_remaining = 1` / 可见菜都放不下）；公式算例 `:155` 与 `capacity_used = capacity_total` 边角 `:181` 的旧口径（「本局继续」「玩家仍可付费跳波、等下一波或耗尽回合」）已换成「非最后一波且 `turns_remaining ≥ 2` 可跳波，最后一波可结束点菜」；`capacity_remaining = 0`（`:134`）与 `effective_satiety > capacity_total`（`:180`）两处也补了「容量本身不构成结束原因 / 由餐局流程按规则 8 收尾」。按要求的四项 grep 复核：`保持 \`Active\``、`本局继续`、`等下一波`、`耗尽回合` 在目标文档中**已零命中**，`turns_remaining` 的四处引用（`:32`、`:155`、`:181`、`:192`）中没有任何一处把付费跳波许给 `turns_remaining = 1`。**但 `:181` 的括号把判据截短了**：它把「已无任何合法动作」注释成「（非最后一波、`turns_remaining = 1`）」，漏掉「可见菜都放不下」；同时把「若仍有合法动作」的枚举也截短成「可跳波 / 最后一波可结束点菜」，漏掉「非最后一波 + `turns_remaining = 1` + 仍有菜放得下 → 仍可吃菜」。按该句「只有…才…」的结构，在 `turns_remaining = 1`、非最后一波、**仍有一道菜放得下**的可达状态下会结束本局，与 ADR-0013 §6 `_no_legal_action()`（三道条件缺一不可）及 `run-flow.md:36` 规则 8 相反。这正是第 4 轮同一处的截短副本，按本仓库先例（规则里错一次算 blocking，copy 里再错一次也算）仍不放行；其余 13 项为 carried 的接口清单措辞、AC 夹具约束、双向回指与注册表同步。

Prior verdict resolved: Partly —— 第 4 轮的 1 条 blocking 的两处主副本（`:181` 主句、`:155` 算例）已不再无条件声明「保持 `Active`」，全文「本局继续」清零，规则 6 本体与 ADR-0013 §6 逐条对应；未闭环的是 `:181` 同一句里的**判据括号**仍为两条而非三条，属 partly-applied，因此本轮判 NEEDS REVISION 而非 APPROVED。上一轮 14 条 recommended 中 0 条在本轮落地（第 4 轮 R5–R14 全部 carried），新增 2 条（`:155` 同源截短、999 vs 26 错位重新开单）。

### Findings

| # | 级别 | 位置（file:line） | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | **BLOCKING** | `design/gdd/stomach-capacity.md:181` | 「只有“已无任何合法动作”（非最后一波、`turns_remaining = 1`）才由餐局流程同帧按 `NO_EATABLE_DISH` 收尾」——括号是 `_no_legal_action()` 的**截短副本**（漏「可见菜都放不下」）；前一分句「若仍有合法动作（…可跳波，或最后一波可结束点菜）」同样漏掉「`turns_remaining = 1` 且仍有菜放得下 → `eat` 是合法动作」。两句合读，在「非最后一波 + `turns_remaining = 1` + 至少一道菜 `can_fit`」这一完全可达的状态（跳到/吃到只剩 1 回合但还有能吃的菜）下，该边角说本局同帧结束，实际应留在 `Active` 等玩家吃菜。修法：括号补「且可见菜都放不下」，并把 Active 枚举补第三条（或整句改为与 `:32` / `:192` 逐字同形的三条件表述） | `run-flow.md:36` 规则 8（三条件、「不改变 `no_eatable`」）；`docs/architecture/adr-0013-action-pipeline-end-judge.md:198-207` 的 `_no_legal_action()`（`for d in visible_dishes(): if can_fit(...): return false`）；`entities.yaml:902`（“no_legal_action is the state-based exit: non-final wave, turns_remaining = 1, **and no visible dish that can_fit**”）；本文件 `:32`、`:192` 自身 |
| 2 | RECOMMENDED | `:284`（AC 8） | （carried，第 4 轮 #3）`GIVEN capacity_used = 20、capacity_total = 20` 未限定 `turns_remaining` 与波次，`THEN` 却断言「餐局流程仍为 `Active`…没有产生任何结束原因」。夹具若取 `turns_remaining = 1` + 非最后一波（容量满 ⇒ 无菜 `can_fit`），该状态在产生它那次提交后同帧已结束，AC 与 B1 同源为空验。补「且 `turns_remaining ≥ 2`（或最后一波）」 | 规则 6 `:32`；ADR-0013 §6；design-docs「验收标准必须可独立判定」 |
| 3 | RECOMMENDED | `:38`（规则 12） | （carried，第 4 轮 #5）「只读暴露：对外**只提供** `capacity_total`、`capacity_used`、`capacity_remaining` 与 `can_fit`」把 `consume`（规则 8 声明的唯一写入口）、`initialize`、`end_run` 全排除在「对外接口」之外，与 AC 17（`:296`）正面冲突。改为「只读**查询**面为…；写入面仅 `initialize` / `consume` / `end_run`」 | 本文件规则 8 `:34`、`:63` 接口表、AC 17；`entities.yaml:922-936` |
| 4 | RECOMMENDED | `:50`、`:57` | （carried，第 4 轮 #6）`已满` 行「仍接受下一局的 `initialize`」与「重开是全新一局：新的 `initialize` 把 `used` 置 0」都缺前提 `end_run(reason)`；与 `:52`、Edge `:187`（`就绪` 时第二次 `initialize` 返回 `CAPACITY_INIT_INVALID`）合读会推出「重开无需 `end_run`」 | 本文件 `:55`（唯一起零点）、`:52`；ADR-0012 §3 |
| 5 | RECOMMENDED | `:34`（规则 8）、`:183`（Edge） | （carried，第 4 轮 #7）错误码表仍只有 `INVALID_SATIETY` / `CAPACITY_OVERFLOW`，未点名 `NOT_INITIALIZED`（该码只出现在 AC 28）。ADR-0012 §5 的 `ConsumeStatus` 是四值封闭枚举，写入口契约应给全 | ADR-0012 §5；本文件 AC 28 `:310` |
| 6 | RECOMMENDED | `:310`（AC 28） | （carried，第 4 轮 #8）只断言 `can_fit` 返回 `false`，未断言其 ERROR；ADR-0012 §4 是「`state == UNINITIALIZED` → 返回 `false` **并记 ERROR**」。AC 25（`:307`）只覆盖三个读数的 ERROR，判定面的 ERROR 无人验 | ADR-0012 §4；本文件 Edge `:195` |
| 7 | RECOMMENDED | `:296`（AC 17） | （carried，第 4 轮 #9）「不可被下游直接调用」在本文件的方位约定（`:215` 把餐局流程记作**上游**）下自洽，但按常识读者会把唯一调用者当成下游，从而读成「餐局流程也不许调 `end_run`」。直写「只允许餐局流程在终止路径调用」 | 本文件 `:215`、`:28`、`:63` |
| 8 | RECOMMENDED | `:217`、`:212` | （carried，第 4 轮 #10）「9 份 GDD 均已把本系统列入依赖或下游表」实测不成立：`grep -c 胃容量` 在 `menu-waves.md` = 0、`records-and-milestones.md` = 0、`save-system.md` = 0（后者 `:217` 已自认），即 10 个依赖里 7 份回指。`:212` 却给纪录与里程碑行标注「GDD 已写」。降级为「7 份已回指，2 份待回指」或驱动对方补行 | design-docs「Dependencies 必须双向」 |
| 9 | RECOMMENDED | `:308`（AC 26） | （carried，第 4 轮 #11）「一次 `initialize` 因非法 `capacity_bonus` 失败、餐局流程进入 `Failed` / `Faulted`」把两种终态混写：非法快照只走 `Failed`（`phase = INIT`），`Faulted` 是 RUN 期不变式失败。按后半句建夹具会得到 `CAPACITY_INIT_INVALID` 却期望 `Faulted` | 本文件 `:28`、`:51`、`:187`；`run-flow.md:46` 规则 18 |
| 10 | RECOMMENDED | `:251-262`（Cross-References） | （carried，第 4 轮 #12）表内只有 GDD 与索引，**没有任何 ADR 行**，而正文已显式引 `ADR-0013 §6`（规则 6 `:32`）与 `ADR-0012 §5`（AC 28 `:310`）。本轮修复使 ADR-0013 从隐性对齐变成显式依据，缺行会让读者以为结尾判据只由 run-flow 定义。补两行（ADR-0012 台账、ADR-0013 §6 判据） | 本文件 `:32`、`:310`；`adr-0013-action-pipeline-end-judge.md` §6 |
| 11 | RECOMMENDED | `:155`（公式算例） | 与 B1 同源的**轻微**截短：只列两条「本局继续」分支（`turns_remaining ≥ 2` 可跳波 / 最后一波可结束点菜），未提「非最后一波 + `turns_remaining = 1` + 仍有菜放得下 → 仍可吃菜」；虽已引「见规则 6」、且不含任何「会结束」断言，但算例是读者最先看的位置，建议与 B1 一并按三条件写齐 | 规则 6 `:32`；ADR-0013 §6；`run-flow.md:36` |
| 12 | RECOMMENDED | `:36`（规则 10）、`:65`/`:206`（两张依赖表）、OQ2 `:319` | （2026-11-06 #8 回开）购买侧约束写的是 `C0 + 最大可能加成 ≤ 999`，而上游实际的数据约束更紧：`entities.yaml:473`（`capacity_ceiling` notes）明确「26 is the effective ceiling, so `base_capacity + max_level <= 26`」。999 守卫在当前数据（20 + 5 = 25）下空转；应引用 26 / `capacity_ceiling`，并把 OQ2 标为已被上游部分回答 | `meta-upgrades.md` 的 `capacity_ceiling` 与等级上限；`entities.yaml:462-473`；design-docs「平衡值必须链到来源公式」 |
| 13 | RECOMMENDED | 跨文档：`docs/architecture/adr-0012-stomach-capacity-ledger.md:182` | 容量的自家 ADR §6 仍写「`capacity_used == capacity_total` 只是派生读数…**本局保持 `Active`**」，无任何「除 `_no_legal_action()` 外」限定（第 4 轮 #13 未修）。本 GDD 修好 `:181` 后反而与自家 ADR 不同形，建议同批给该行补限定（ADR 不在本文件可改范围） | ADR-0013 §6；`run-flow.md:36`；`entities.yaml:902` |
| 14 | RECOMMENDED | 跨文档：`entities.yaml:918`、`:1303`、`:722-726`；`systems-index.md:26`、`:64`；`cloying-and-taste.md:62` | 注册表与索引仍未跟上：`capacity_total.notes` / `base_capacity.notes` 仍称「data ownership is stomach Open Question 1」，而本文件 `:318` 已标**已解决**（`data-config.md:59` 已定义 `BalanceData.base_capacity`）；`capacity_used.referenced_by` 仍缺 `run-flow.md`（其规则 12 读该值）；`capacity_remaining` 与 `capacity_bonus` 仍无条目（`grep -E '^  - name:'` 只命中 `capacity_ceiling` / `capacity_used` / `wasted_capacity` / `capacity_total` / `transition_capacity_cost` / `base_capacity`）；`systems-index.md:64` 记 `Approved` 而本文件头 `:3` 为 `In Review`，`:26` 仍写容量 “decides when the player is stuffed”；`cloying-and-taste.md:62` 仍记「双向 / 吃撑」。均由编排会话统一处理 | 本文件 `:3`、`:318`；ADR-0012 Related；`data-config.md:59` |

Advisory（不计入推荐数）：`Player Fantasy` 与 `Tuning Knobs` 仍无独立标题（`standard` 下 advisory；`## Game Feel` `:232-238` 覆盖幻想，「满格不是失败」正是规则 6 的玩家面表述；`C0` 与 999 结构上界事实上是两个旋钮，补一节会顺手暴露第 12 条）。`:6` 的 `Last Verified: 2026-11-03` 已落后于 `Last Updated: 2026-11-17`（头部归编排会话）。`:192` 「HUD 可显示“吃不下”提示」与 `in-game-hud.md:370`「HUD 不自行结束本局，也不提示“无菜可吃”」措辞接近但所指不同（逐卡原因 vs 全局提示），建议把本文件改成「HUD 可在菜品卡上写明原因」，避免读者以为可以在全屏层面提示。

### 已核对且无问题（记录以免重复检查）

- **第 4 轮 blocking 的主句已闭环**：`:181` 首句与 `:155` 的「本局保持 `Active` / 本局继续」已消失，`grep -n '保持 \`Active\`|本局继续|等下一波|耗尽回合'` 在目标文档 0 命中（命中项只剩 `:32`、`:181` 中已带限定语的表述）；`:134`（`capacity_remaining = 0`）与 `:180`（`effective_satiety > capacity_total`）两处新限定与 `:192` 语义一致。
- **规则 6 本体与权威逐条同形**：`:32` 的三道判据 = `adr-0013...:198-207` 的 `_no_legal_action()`（`is_final_wave()` → false；`turns_remaining != 1` → false；任一可见菜 `can_fit` → false）= `run-flow.md:36` 规则 8 的例外句 = `run-flow.md:201`（`end_reason` 公式）与 `:209`（变量行）= `entities.yaml:901-902`（`no_eatable or no_legal_action`、`Capacity alone never ends a run`）。不新增第三枚举，也不改 `no_eatable`（`run-flow.md:36` 明写不改变信号）。
- **`skip_wave` 不被许给最后一回合**：`:32`、`:155`、`:181` 三处都写 `turns_remaining ≥ 2`；与 `time-and-turns.md:53` 规则 13（`NO_TURNS` 先于 `spend_turn()`）、`time-and-turns.md:287/292/293` 的 AC 18 / 23 / 23a、`menu-waves.md:31`（最后一回合例外）一致。
- **最后一波语义**：`:32`/`:155`/`:181` 的「最后一波可结束点菜」= `menu-waves.md:30`（`end_pickup() → ENDED`、0 回合）= `adr-0013...:163-172`（§5 两道校验、不 `spend_turn()`、不改变容量）= `time-and-turns.md:178`。
- **NO_EATABLE 文案与可用性**：本文件不复述 HUD 文案、不自行比较 `turns_remaining`；HUD 侧 `in-game-hud.md:59`/`:372`/`:504`（`*_reason` 映射、`NO_TURNS` → “最后一回合不能跳波”、静态扫描 0 命中 `turns_remaining >= 2`）与 `:373`（`turns_remaining = 1` 全不可吃不是可停留态）与本文件规则 6 无冲突。
- **`end_run` 面**：`:28`、`:55`、AC 27 与 `run-flow.md:52` 规则 24（`reason` 由 `fault.phase` 决定）一致；`NO_EATABLE_DISH` 走正常交接因此是 `ended` 而非 `faulted`，`:192` 写对。
- **数值面**：`capacity_total ∈ [1,999]`（`entities.yaml:916`）、`can_fit` 含等号（`:933-934`）、`base_capacity = 20`（`:1299`）、`effective_satiety ∈ [1,9]`（`:511` 区段）、`extra_satiety ∈ [0,3]`（`:494` 区段）、`end_reason` 取值域（`:900`）与本文件逐值一致；`opt_value = 104` / 20 格（`dish-values.md:63`、`:216`）、`capacity_ceiling` 的 +2/+1/+5/+4/+4（`entities.yaml:473`）复算仍正确。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

`gdd-structure-check.sh` 输出 `PRESENT: Overview, Detailed Rules, Formulas, Edge Cases, Dependencies, Acceptance Criteria`（6/6 必需齐全；Formulas 因本系统定义容量上限与判定谓词而 REQUIRED），`ABSENT: Player Fantasy, Tuning Knobs`（advisory）。Formulas 五个公式各有变量表、值域、输出范围与算例，可求值。

### Dependency Graph

声明的 10 个依赖在磁盘上均有 GDD。反向回指实测（`grep -c 胃容量`）：`run-flow` 20、`in-game-hud` 17、`settlement-and-entry-fee` 10、`meta-upgrades` 7、`cloying-and-taste` 6、`data-config` 5、`dish-values` 5；`menu-waves` / `records-and-milestones` / `save-system` 为 0——与 `:217`「9 份均已列入」不符（第 8 条）。Interactions 表（`:61-72`）与 Dependencies 表（`:202-213`）逐行一致，未发现方向或硬软度分歧。

### Registry Values

按 Phase 3 的全文档检索（`capacity_total|capacity_used|capacity_remaining|capacity_bonus|base_capacity|can_fit|effective_satiety|extra_satiety|cloy_cap|capacity_ceiling|end_reason|no_eatable|no_legal_action|opt_value|satiety`，大小写不敏感）命中 **129** 行，逐行比对**无数值冲突**；`end_reason` 的两处取值域（`NONE | TIME_EXHAUSTED | NO_EATABLE_DISH`）与 `settlement-and-entry-fee.md`、`records-and-milestones.md`、`entities.yaml:900` 一致。条目与注释缺口见第 14 条。

### 判定说明

判 NEEDS REVISION 的唯一理由是 B1。第 4 轮判 blocking 的那句话（Edge `:181`「容量满 → 保持 `Active`、仍可付费跳波」）确实被改掉了，但改法把**同一个判据**写成了两条而非三条：`_no_legal_action()` 是 `is_final_wave()` 为假 **且** `turns_remaining == 1` **且** 没有任何可见菜 `can_fit`，`run-flow.md:36` 与 `entities.yaml:902` 都把第三条写死。`:181` 的括号删掉了第三条，于是「非最后一波 + 剩 1 回合 + 还有菜放得下」这个每局都会出现的普通状态（`skip_wave` 被拒但 `eat` 仍可用）在该句下被读成「同帧结束本局」——与改前那句方向相反、但同样是可达状态下的假结论。它之所以仍算 blocking 而不是措辞瑕疵：按 `:181` 施工的实现会提前用 `NO_EATABLE_DISH` 结束一局玩家还能继续吃的餐局，直接违反 `run-flow.md` 规则 8 与 ADR-0013 §6，并且会让 `time-and-turns.md:178`「若当前波至少有一道菜放得下，本局停在 `Active`」这一条无对应实现；这与 `run-flow-review-log.md:357` 给 B3 判 blocking 的判据同形（同一缺陷在 rule 与 copy 里各算一次）。修复面是一个括号（补「且可见菜都放不下」）+ 一个枚举（补「仍有菜放得下 → 仍可吃菜」），零公式、零数值，不需要结构返工；其余 13 项 recommended 中 12 项为 carried，均可随同批处理。

### Scope Signal

Rough scope signal: **S**（与前三轮一致——单系统账本，5 个公式皆为整数加减无除法，无新 ADR）。按本 skill 的字面刻度（3+ 公式 / 5+ 依赖）也可读作 L，但实现量集中在 `can_fit` / `consume` 两个方法上，producer 排期前可自行复核。

### Verdict: NEEDS REVISION

建议索引状态：**Needs Revision**（`systems-index.md:64` 当前为 `Approved`、本文件头 `:3` 为 `In Review`；按并行约束本次未改索引与文件头，由编排会话统一更新）。B1 修好并复审后可回到 `In Review → Approved`。

---

Reviewed-Content-Hash: design/gdd/stomach-capacity.md fe0667091ab76068526f8c6d48b3d04742ac3f4d
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

---

## Review — 2026-11-17 — Verdict: APPROVED（第 6 轮：无合法动作三条件补齐后的 lean 复审）
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 15

**回执状态**：`review-receipts.sh check` 报 `CHANGED: design/gdd/stomach-capacity.md`（第 5 轮绑定 `fe066709…`，当前 `8d8ed2b5…`），另见 `data-config.md` / `dish-values.md` / `audio-feedback.md` 相对第 5 轮回执为 `CHANGED`（本轮按 `base_capacity`、`satiety 1–6`、`opt_value = 104` / 20 格抽查，未作全量比对）；其余全部 `UNCHANGED`——`run-flow.md`、`time-and-turns.md`、`menu-waves.md`、`in-game-hud.md`、`entities.yaml`、`SKILL.md`、`design-docs.md`、ADR-0012 / ADR-0013 / ADR-0007 / ADR-0014、`systems-index.md`、`cloying-and-taste.md`、`settlement-and-entry-fee.md`、`meta-upgrades.md`、`records-and-milestones.md`、`game-concept.md`、`gdd-cross-review-2026-11-04.md`。目标文档变动，按 skill Phase 1 走**全量**复审（325 行逐节重读），不接受 delta。

Summary: 第 5 轮的 1 条 blocking 已闭环，且是按正确方式闭环的：`capacity_used = capacity_total` 边角（`:181`）不再给出「容量满 → 保持 `Active`」的无条件结论，而是先声明「容量满本身不产生结束原因，也不直接决定本局是否继续」，再在**该边角的自身作用域内**把 Active 的两条分支列全（非最后一波 ∧ `turns_remaining ≥ 2` → 跳波；最后一波 → 结束点菜），最后用三道判据定义「已无任何合法动作」（非最后一波、`turns_remaining = 1`、可见菜都放不下），并用「（容量满时最后一条恒真）」显式说明第三条在此边上必然成立——于是第 5 轮判 blocking 的两种错误读法（把判据截成两条、以及漏掉 Active 的合法动作）都不再成立：容量满时 `can_fit(e)` 对任何 `e ≥ 1` 为 `false`，吃菜本就不是合法动作，因此该边角的 Active 枚举在此作用域内**是穷尽的**。公式算例 `:155` 的同源副本同步改写，第三条以「（此时可见菜都放不下）」出现（见 R1，措辞建议，非缺陷）。按要求的四项 grep 复核：`保持 \`Active\`` 仅剩 `:32` 与 `:181` 两处且均带限定，`本局继续` / `等下一波` / `耗尽回合` 全文 0 命中，`turns_remaining` 的四处引用（`:32`、`:155`、`:181`、`:192`）无一把付费跳波许给 `turns_remaining = 1`。其余 15 项均为 carried 的接口清单措辞、AC 夹具约束、双向回指与注册表同步，无一项改变实现或使 AC/规则不可判定，故本轮放行。
Prior verdict resolved: Yes —— 第 5 轮唯一的 blocking（`design/gdd/stomach-capacity.md:181` 的判据截短副本 + Active 枚举漏项，同源副本 `:155`）已闭环：`:181` 现并列三道判据并给出「容量满时最后一条恒真」的作用域说明，Active 两条分支各自写明「本局保持 `Active`」；`:155` 同步改写且不再含「本局继续」。第 4 轮的 `:134`（`capacity_remaining = 0`）与 `:180`（`effective_satiety > capacity_total`）限定仍与规则 6 同义。上一轮 13 条 recommended 中 1 条（第 11 条 `:155` 的截短）已实质落地、0 条消除，另新增 2 条（R1、R9 回开），14 条 carried。

### Findings

| # | 级别 | 位置（file:line） | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | RECOMMENDED | `:155`（公式算例） | 第三条判据以插入语「（此时可见菜都放不下）」出现，与 `:181` 的「且可见菜都放不下（容量满时最后一条恒真）」不同形：整句主语是「容量满」，故此断言在其作用域内为真、不构成假结论；但读者若把「此时」读成「凡非最后一波 ∧ `turns_remaining = 1`」，就会把第三条当成恒真而省掉 `can_fit` 循环。建议改写为与 `:181` 逐字同形的显式合取 | `time-and-turns.md:178`（「若当前波至少有一道菜放得下，本局停在 `Active`」）；`run-flow.md:36` 规则 8；ADR-0013 §6 `_no_legal_action()` |
| 2 | RECOMMENDED | `:32`（规则 6） | 规则 6 是**未按容量满作用域限定**的一般陈述，其合法动作枚举只列「付费跳波」与「结束点菜」，未列「非最后一波 ∧ `turns_remaining = 1` ∧ 仍有菜放得下 → 仍可吃菜」。因「只有…才…」结构使结论仍正确（无假陈述、实现不会错），不阻塞；但该句是全文最常被引用的一句，补全第三类合法动作可让「与容量是否满无关」自洽 | 本文件 `:32` 自身、`:181`；`time-and-turns.md:178`；ADR-0013 §6 `:203-206`（任一可见菜 `can_fit` 即返回 `false`） |
| 3 | RECOMMENDED | `:284`（AC 8） | （carried，第 4 轮 #3 / 第 5 轮 #2）`GIVEN capacity_used = 20、capacity_total = 20` 仍未限定 `turns_remaining` 与波次，`THEN` 却断言「餐局流程仍为 `Active`…没有产生任何结束原因」。测试替身（`:270` 只提供 `Active` / `Faulted` 探针）下该断言恒真；真实集成下若夹具取 `turns_remaining = 1` + 非最后一波（容量满 ⇒ 无菜 `can_fit`），该状态在产生它那次提交后同帧已结束。补「且 `turns_remaining ≥ 2`（或最后一波）」 | 规则 6 `:32`；ADR-0013 §6；design-docs「验收标准必须可独立判定」 |
| 4 | RECOMMENDED | `:38`（规则 12） | （carried，第 4 轮 #5 / 第 5 轮 #3）「只读暴露：对外**只提供** `capacity_total`、`capacity_used`、`capacity_remaining` 与 `can_fit`」把 `consume`（规则 8 声明的唯一写入口）、`initialize`、`end_run` 全排除在「对外接口」之外，与 AC 17（`:296`）和接口表（`:63`）正面冲突。改为「只读**查询**面为…；写入面仅 `initialize` / `consume` / `end_run`」 | 本文件规则 8 `:34`、AC 17、`:63`；`entities.yaml:922-936` |
| 5 | RECOMMENDED | `:50`、`:57` | （carried，第 4 轮 #6 / 第 5 轮 #4）`已满` 行「仍接受下一局的 `initialize`」与「重开是全新一局：新的 `initialize` 把 `used` 置 0」都缺前提 `end_run(reason)`；与 `:55`（唯一回到 `未初始化` 的路径）、`:52`、Edge `:187`（`就绪` 时第二次 `initialize` 返回 `CAPACITY_INIT_INVALID`）合读会推出「重开无需 `end_run`」 | 本文件 `:55`、`:52`；ADR-0012 §3；`run-flow.md:52` 规则 24 |
| 6 | RECOMMENDED | `:34`（规则 8）、`:183`（Edge） | （carried，第 4 轮 #7 / 第 5 轮 #5）错误码表仍只有 `INVALID_SATIETY` / `CAPACITY_OVERFLOW`，未点名 `NOT_INITIALIZED`（该码只出现在 AC 28）。ADR-0012 §5 的 `ConsumeStatus` 是四值封闭枚举，写入口契约应给全 | ADR-0012 §5；本文件 AC 28 `:310`、Edge `:195` |
| 7 | RECOMMENDED | `:310`（AC 28） | （carried，第 4 轮 #8 / 第 5 轮 #6）只断言 `can_fit` 返回 `false`，未断言其 ERROR；ADR-0012 §4 是「`state == UNINITIALIZED` → 返回 `false` **并记 ERROR**」。AC 25（`:307`）只覆盖三个读数的 ERROR | ADR-0012 §4；本文件 Edge `:195` |
| 8 | RECOMMENDED | `:296`（AC 17） | （carried，第 4 轮 #9 / 第 5 轮 #7）「不可被下游直接调用」在本文件的方位约定（`:215` 把餐局流程记作**上游**）下自洽，但按常识读者会把唯一调用者当成下游，从而读成「餐局流程也不许调 `end_run`」。直写「只允许餐局流程在终止路径调用」 | 本文件 `:215`、`:28`、`:63` |
| 9 | RECOMMENDED | `:192`（Edge）／AC 1–28 | （第 4 轮 #4 回开）`NO_EATABLE_DISH` 收尾路径是本系统唯一的跨系统行为，`Edge :192` 写了容量侧后果（「本系统只被调一次 `end_run(ended)`」），但 AC 1–28 **无一条经过该路径**。建议补一条：本局按 `NO_EATABLE_DISH` 结束时容量恰好收到一次 `end_run(ended)`，此后三个读数返回 `0` + ERROR、`can_fit` 返回 `false` + ERROR | ADR-0012 §3 / §4；`run-flow.md:52` 规则 24；本文件 `:55`、`:192` |
| 10 | RECOMMENDED | `:217`、`:212` | （carried，第 4 轮 #10 / 第 5 轮 #8）「9 份 GDD 均已把本系统列入依赖或下游表」本轮实测仍不成立：`grep -c 胃容量` → `run-flow` 20、`in-game-hud` 17、`settlement-and-entry-fee` 10、`meta-upgrades` 7、`cloying-and-taste` 6、`data-config` 5、`dish-values` 5；`menu-waves` 0、`records-and-milestones` 0（`save-system` 0，`:217` 已自认）。即 9 份里 7 份回指。降级为「7 份已回指，2 份待回指」，或驱动对方补行；`:212` 的「GDD 已写」括注同源 | design-docs「Dependencies 必须双向」 |
| 11 | RECOMMENDED | `:308`（AC 26） | （carried，第 4 轮 #11 / 第 5 轮 #9）「一次 `initialize` 因非法 `capacity_bonus` 失败、餐局流程进入 `Failed` / `Faulted`」把两种终态混写：非法快照只走 `Failed`（`phase = INIT`），`Faulted` 是 RUN 期不变式失败。按后半句建夹具会得到 `CAPACITY_INIT_INVALID` 却期望 `Faulted` | 本文件 `:28`、`:51`、`:187`；`run-flow.md:46` 规则 18 |
| 12 | RECOMMENDED | `:251-262`（Cross-References） | （carried，第 4 轮 #12 / 第 5 轮 #10）表内只有 GDD 与索引，**没有任何 ADR 行**，而正文已显式引 `ADR-0013 §6`（规则 6 `:32`）与 `ADR-0012 §5`（AC 28 `:310`）。补两行（ADR-0012 台账、ADR-0013 §6 判据） | 本文件 `:32`、`:310`；`docs/architecture/adr-0013-action-pipeline-end-judge.md` §6 |
| 13 | RECOMMENDED | `:36`（规则 10）、`:65`/`:206`（两张依赖表）、OQ2 `:319` | （2026-11-06 #8 / 第 5 轮 #12 回开）购买侧约束写的是 `C0 + 最大可能加成 ≤ 999`，而上游实际的数据约束更紧：`entities.yaml:473`（`capacity_ceiling` notes）写「26 is the effective ceiling, so `base_capacity + max_level <= 26`」，`meta-upgrades.md:260` 亦同。999 守卫在当前数据（20 + 5 = 25）下空转；应引用 26 / `capacity_ceiling`，并把 OQ2 标为已被上游部分回答 | `meta-upgrades.md:260`、`:385`；`entities.yaml:462-473`；design-docs「平衡值必须链到来源公式」 |
| 14 | RECOMMENDED | 跨文档：`docs/architecture/adr-0012-stomach-capacity-ledger.md:182` | （carried，第 4 轮 #13 / 第 5 轮 #13）容量的自家 ADR §6 仍写「`capacity_used == capacity_total` 只是派生读数…**本局保持 `Active`**」，无任何「除 `_no_legal_action()` 外」限定（本轮复核该行字节未变）。本 GDD 已修好 `:181`，现在与自家 ADR 不同形，建议同批给该行补限定（ADR 不在本次可改范围） | ADR-0013 §6 `:210`；`run-flow.md:36`；`entities.yaml:902` |
| 15 | RECOMMENDED | 跨文档：`entities.yaml:718-732`、`:918`、`:1303`；`systems-index.md:26-27`、`:64`；`cloying-and-taste.md:62` | （carried，第 4 轮 #14 / 第 5 轮 #14）注册表与索引仍未跟上：`capacity_used.referenced_by` 仍缺 `run-flow.md`（其规则 12 读该值）；`capacity_total.notes` / `base_capacity.notes` 仍称「data ownership is stomach Open Question 1」，而本文件 `:318` 已标**已解决**（`data-config.md:59` 已定义 `BalanceData.base_capacity`）；`capacity_remaining` 与 `capacity_bonus` 仍无条目（`grep -E '^  - name:'` 只命中 `capacity_ceiling` / `capacity_used` / `wasted_capacity` / `capacity_total` / `can_fit` / `base_capacity`）；`systems-index.md:64` 记 `Approved` 而本文件头 `:3` 为 `In Review`，`:26-27` 仍写容量 “decides when the player is stuffed”；`cloying-and-taste.md:62` 仍记「双向 / 吃撑」。均由编排会话统一处理 | 本文件 `:3`、`:318`；ADR-0012 Related；`data-config.md:59` |

Advisory（不计入推荐数）：`Player Fantasy` 与 `Tuning Knobs` 仍无独立标题（`gdd-structure-check.sh` 输出 `ABSENT: Player Fantasy,Tuning Knobs`；`standard` 下 advisory，`## Game Feel` `:232-238` 覆盖幻想，「满格不是失败」正是规则 6 的玩家面表述；补 Tuning Knobs 会顺手暴露 R13）。`:6` 的 `Last Verified: 2026-11-03` 落后于 `Last Updated: 2026-11-17`（文件头归编排会话）。`:192`「HUD 可显示“吃不下”提示」与 `in-game-hud.md:370`「HUD 不自行结束本局，也不提示“无菜可吃”」措辞接近但所指不同（逐卡原因 vs 全局提示），建议改成「HUD 可在菜品卡上写明原因」。

### 已核对且无问题（记录以免重复检查）

- **第 5 轮 blocking 已闭环（本轮核心判据）**：`:181` 先断言「容量满本身不产生结束原因，也不直接决定本局是否继续」，再列 Active 的两条分支（非最后一波 ∧ `turns_remaining ≥ 2` → 跳波，「本局保持 `Active`」；最后一波 → 结束点菜，「本局保持 `Active`」），最后才给「只有非最后一波、`turns_remaining = 1`、且可见菜都放不下（容量满时最后一条恒真）才已无任何合法动作」。**在该边角的作用域内这个枚举是穷尽的**：容量满 ⇒ `can_fit(e)` 对任何 `e ≥ 1` 为 `false` ⇒ 吃菜不是合法动作，第 5 轮所指的「仍可吃菜」分支在此处是**空集而非漏项**；括号把第三条写明并说明其恒真，也不再是「截短副本」。按字面施工得到的 `_no_legal_action()` 与 ADR-0013 §6 `:198-207` 逐条同形。
- **反例逐状态复核**：①非最后一波、吃一道菜把回合从 2 用到 1、容量恰好填满 → 本文件现表述为「已无任何合法动作 → 同帧 `NO_EATABLE_DISH`」，与 `run-flow.md:36`、`:201` 的 `end_reason` 公式、`time-and-turns.md:293` 的 AC 23a 一致；②非最后一波、`turns_remaining ≥ 2`、容量满 → 「跳波 → `Active`」，与 ADR-0013 §4 的两道校验一致（`can_advance()` 在非最后一波恒真：当前波吃空会自动推进，`menu-waves.md:28`）；③最后一波、容量满 → 「结束点菜 → `Active`」，与 `menu-waves.md:30`、`:51` 一致（若最后一波已被吃空，`no_eatable` 已为真、本局已不在 `Active`，故 `can_end_pickup()` 不构成反例）；④非最后一波、`turns_remaining = 1`、仍有菜放得下 → 本文件三处（`:32`、`:155`、`:181`）现均**不**宣称结束，与 `time-and-turns.md:178` 一致。第 5 轮的那个假结论已不出现在任何一处。
- **无条件延续口径已清零**：`grep -n '保持 \`Active\`'` 仅命中 `:32`（带「只取决于是否仍有合法动作」限定与三条件例外）与 `:181`（两条分支各自带条件）；`本局继续`、`等下一波`、`耗尽回合` 全文 0 命中；`:237`「没有自动推进」与「推进波次只有 `skip_wave` / `end_pickup`」自洽。
- **规则 6 本体与权威同形**：`:32` 的三道判据 = ADR-0013 §6 `_no_legal_action()`（`is_final_wave()` → false；`turns_remaining != 1` → false；任一可见菜 `can_fit` → false）= `run-flow.md:36` 规则 8 的例外句 = `:201` 的 `end_reason` 公式 = `entities.yaml:901-902`（`no_eatable or no_legal_action`、「Capacity alone never ends a run」）。不新增第三枚举，也不改 `no_eatable`。
- **`skip_wave` 不被许给最后一回合**：`:32`、`:155`、`:181` 三处都写 `turns_remaining ≥ 2`；与 `time-and-turns.md:53` 规则 13（`NO_TURNS` 先于 `spend_turn()`）、`:292`/`:293` 的 AC 23 / 23a、`menu-waves.md:31`（最后一回合例外）一致。
- **NO_EATABLE 文案与可用性**：本文件不复述 HUD 文案、不自行比较 `turns_remaining`；`in-game-hud.md:372`（`NO_TURNS` → “最后一回合不能跳波”、`NO_EATABLE` → “现在没有可处理的菜”）、`:373`、AC 26a `:505` 与规则 6 无冲突。
- **`end_run` 面**：`:28`、`:55`、AC 27 与 `run-flow.md:52` 规则 24（`reason` 由 `fault.phase` 决定）一致；`NO_EATABLE_DISH` 走正常交接，因此是 `ended` 而非 `faulted`，`:192` 写对。
- **数值面**：`capacity_total ∈ [1,999]`（`entities.yaml:916`）、`can_fit` 含等号（`entities.yaml:933-934`）、`base_capacity` 1–999 默认 20（`data-config.md:59`、`:102`）、`satiety ∈ [1,6]`（`data-config.md:33`、`:110`、`:161` 的 ≥ 7 拒绝）、`opt_value = 104` / 20 格（`dish-values.md:63`、`:216`、AC 25 `:363`）、`capacity_ceiling` 的 +2/+1/+5/+4/+4（`entities.yaml:473`、`meta-upgrades.md:255`）与本文件逐值一致，复算无误。
- **`待回改：无`（`:266`）仍准确**：`run-flow.md:34` 规则 6 已含 `can_fit` / `consume(e)`；`data-config.md:59`、`:197` 已定义 `base_capacity` 并有胃容量下游行。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

`gdd-structure-check.sh` 输出 `PRESENT: Overview,Detailed Rules,Formulas,Edge Cases,Dependencies,Acceptance Criteria`（6/6 必需齐全，Formulas 因本系统定义容量上限与判定谓词而 REQUIRED），`ABSENT: Player Fantasy,Tuning Knobs`（advisory）。Formulas 五个公式各有变量表、值域、输出范围与算例，可求值。

### Dependency Graph

声明的 10 个依赖在磁盘上均有 GDD：run-flow、data-config、meta-upgrades、cloying-and-taste、in-game-hud、settlement-and-entry-fee、menu-waves、dish-values、records-and-milestones、save-system。反向回指实测（`grep -c 胃容量`）：`run-flow` 20、`in-game-hud` 17、`settlement-and-entry-fee` 10、`meta-upgrades` 7、`cloying-and-taste` 6、`data-config` 5、`dish-values` 5；`menu-waves` / `records-and-milestones` / `save-system` 为 0——与 `:217`「9 份均已列入」不符（R10）。Interactions 表（`:61-72`）与 Dependencies 表（`:202-213`）十行逐行一致，方向与硬软度无分歧。

### Registry Values

按 Phase 3 的全文档检索（`capacity_total|capacity_used|capacity_remaining|capacity_bonus|base_capacity|can_fit|effective_satiety|extra_satiety|cloy_cap|capacity_ceiling|end_reason|no_eatable|no_legal_action|opt_value|satiety|end_run`，大小写不敏感）命中 **135** 行，逐行比对**无数值冲突**；`end_reason` 的两处取值域（`NONE | TIME_EXHAUSTED | NO_EATABLE_DISH`）与 `settlement-and-entry-fee.md`、`records-and-milestones.md`、`entities.yaml:900-901` 一致。条目与注释缺口见 R15。

### 判定说明

本轮判 APPROVED 的依据是 blocking 的定义（会直接改变实现，或使某条 AC / 规则不可判定），而非「文档已无瑕疵」。第 5 轮的 blocking 属于前者（按字面施工会提前用 `NO_EATABLE_DISH` 结束一局玩家还能继续吃的餐局），现已被正确闭环——`:181` 三条件齐备且带作用域说明，Active 枚举在容量满作用域内穷尽；`:155` 同步改写且不含「本局继续」。剩余 15 条无一满足该定义：R1（`:155` 的插入语式表述）在其容量满作用域内为真，最坏结果是读者把第三条当成恒真，而规则 6 与 `:181` 都写死了该条件；R2（规则 6 的一般枚举未列「仍可吃菜」）因「只有…才…」的必要条件结构不会推出假结论；R3–R9 是 AC 夹具精度、接口清单措辞与一条回归覆盖缺失（现行为已由规则 6 / Edge / ADR 定义，缺的是护栏不是判据）；R10–R15 是依赖对称性、ADR 表行、26 vs 999 的上游错位与注册表/索引同步，均不改变本系统的实现或可判定性。第 3–5 轮的历史结论（状态表拆行、`end_run` 唯一清零点、`NOT_INITIALIZED`、AC 27/28、`effective_satiety > capacity_total` 限定）经复核仍成立。

### Scope Signal

Rough scope signal: **S**（与前四轮一致——单系统账本，5 个公式皆为整数加减无除法，无新 ADR）。按本 skill 的字面刻度（3+ 公式 / 5+ 依赖）也可读作 L，但实现量集中在 `can_fit` / `consume` 两个方法上，producer 排期前可自行复核。

### Verdict: APPROVED

建议索引状态：**Approved**（`systems-index.md:64` 现为 `Approved`，本文件头 `:3` 为 `In Review`；按并行约束本次未改索引与文件头，由编排会话把 `:3` 收敛为 `Approved`，并考虑一并处理 R13–R15）。R1/R2/R9 建议在最终清理批里顺手处理——它们不阻塞，但 `:155` 的三条件写法与 `:181` 对齐后，下一轮复审不必再逐字比这两处。

---

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 8d8ed2b50317ecdea9471eaba8135364cdc61e4b
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/stomach-capacity.md 82c3ea703a3c2d1e2b6bd0a8f00b4f87569cc4a8

## Re-stamp — 2026-11-17（架构复审后）
仅同步闸门输入文件的当前字节（GDD 本体自上次复审后只改了 Status 头行）；未改规则、公式、AC、依赖、契约或平衡数值。

Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/data-config.md e4d2ee1de1f4c9affd59a3993adb89056d5e4fa9
Reviewed-Content-Hash: design/gdd/dish-values.md 3eac1141e6c1209b59cc5e1ceeaa5c9ce85bc186
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/gdd-cross-review-2026-11-04.md c6efa8393fa2f8c10b80d5964b75aa9793f7d042
Reviewed-Content-Hash: design/gdd/in-game-hud.md 6d7a4700f0f8b718eeb3bf0cbe9db7993df1f43d
Reviewed-Content-Hash: design/gdd/menu-waves.md af47aef125e14677602eeab6eed5ae98132c6113
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 213f8dbe40b44748fc32857680f09991fff9d612
Reviewed-Content-Hash: design/gdd/run-flow.md da67f644d65824cb6ae98081cea03f69f0e77a0d
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 82c3ea703a3c2d1e2b6bd0a8f00b4f87569cc4a8
Reviewed-Content-Hash: design/gdd/systems-index.md ccecd7af3ef6993252cb6901db7435d0985c0953
Reviewed-Content-Hash: design/gdd/time-and-turns.md ce54e5bd984a1c7a2bbb973bf124bd998563f5b4
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 30b02733b0ec60c3e18d0acf82d8fd5b740909b9
Reviewed-Content-Hash: docs/architecture/adr-0012-stomach-capacity-ledger.md c2bef2e743e052fe85feb7b145de41591273cedf
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: docs/architecture/adr-0014-cloying-taste-state-machine.md 65b2a675e3a12378e41a021949a4aa1342e15bee
