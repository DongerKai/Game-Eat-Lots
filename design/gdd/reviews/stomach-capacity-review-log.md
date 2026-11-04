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
