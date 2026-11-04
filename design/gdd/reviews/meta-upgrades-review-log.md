# 局外升级（Meta Upgrades）— Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 3 | Recommended: 8
Summary: The economy core holds up: the price chain, tree totals, snapshot clamp, resistance offset and the whole `capacity_ceiling` table reproduce exactly from the dish table (104/106/107/112/116/120/121 at C=20–26), and the GDD is unusually explicit about failure policy. Three contract-level conflicts block, all in the settlement hand-off rather than the upgrade math: rule 6 commits internally while `ui-flow` rule 10 and `save-system` AC 13 require one orchestrated commit; rule 4 says a `WALLET_OVERFLOW` run still reaches 纪录 while `ui-flow` abandons the record update; and the audio row asserts three events that `audio-feedback` explicitly says do not exist. The remaining findings are stale cross-document claims (the "待回改" list is 8/9 applied) and one stale Open Question that `首局引导` has since answered with a 40 元 teaching fee.
Prior verdict resolved: First review

Findings:
- [BLOCKING] Detailed Design rule 6 (`:37`) / Interfaces (`:122`) / AC 2 (`:444`), AC 5 (`:447`) vs `ui-flow.md:59` rule 10 and `save-system.md:378` AC 13: `apply_run_result` performs its own `commit`, but the settlement transaction is required to be a single orchestrated write across `progression` + `records` (+ `unlocks`). Two writers, two write counts, no prepare/commit split.
- [BLOCKING] Detailed Design rule 4 (`:33`) / Edge Cases (`:294`) vs `ui-flow.md:380`: on `WALLET_OVERFLOW` this GDD says the run "仍照常结算给纪录系统 / 仍交由纪录系统处理"; `ui-flow` says "纪录更新一并放弃". Same input, opposite record outcome.
- [BLOCKING] Interactions (`:107`) / Cross-References vs `audio-feedback.md:376`: this GDD declares `purchase_succeeded` / `purchase_failed` / `run_result_applied` as an interface; `audio-feedback` states 局外升级 has no direct call and its events "不产生声音". The dependency is not bidirectional.
- [RECOMMENDED] Dependencies（待回改）(`:364-374`) / rule 6 (`:37`) / `:339`, `:348`, `:356`, `:435`: the back-write list and several status annotations are stale — items 1–7 and 9 have landed.
- [RECOMMENDED] Open Question 1 (`:491`) and Edge Cases (`:327`): still recommend `entry_fee = 0` / "免单" for the first run; `first-run-onboarding.md:21,38,297` chose 40 via `entry_fee_override` and rejects zero explicitly.
- [RECOMMENDED] Interfaces (`:121`) / rule 2 (`:29`): `spend(amount, reason, extra)` is declared but its validation order, `extra` merge and rollback are only defined in `unlocks-and-stations.md:120-138`.
- [RECOMMENDED] Interfaces (`:118-120`) / rule 15 (`:55`) / AC 26 (`:477`): `purchase`、`can_purchase`、`next_price`、`get_catalog` take no station, but `NO_EFFECT` and the displayed cap depend on `wave_size(s)`.
- [RECOMMENDED] Design-time constraint (`:259`, `:382`): `base_capacity + max_level ≤ 26` is called a data constraint but no validator owns it (`data-config.md:97` `R_upg` has no cross-record check).
- [RECOMMENDED] Rule 15 (`:55`) vs `menu-waves.md:64,227,334`: the GDD claims to answer menu-waves OQ 3, which is an occlusion/UX question; menu-waves still says it consumes "整数等级" and that the cap is guaranteed by 局外升级.
- [RECOMMENDED] Cross-References (`:428`) / Formulas (`:206`) vs `run-flow.md:65,454`: Open Question 11 is claimed answered here but remains open in run-flow, and the `levels` key is not part of run-flow's stated snapshot schema.
- [RECOMMENDED] Open Questions 3 / 8 / 10 (`:493`, `:498`, `:500`): target dates have passed — `time-and-turns.md:292`, `records-and-milestones.md` and `ui-flow.md:83` all now exist and change the owner/answer.

---

## Design Review: 局外升级（Meta Upgrades）

Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules
(`## Detailed Design` is the accepted heading), Formulas (the system defines
numeric rules — price curves, snapshot resolution, resistance offset, preview
cap), Edge Cases, Dependencies, Acceptance Criteria. All six are present.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:401-407`) carries
  the intended feeling ("付款即见数", "升级必须能被下一局感知").
- **Tuning Knobs** — present (`:376-388`), with safe ranges and the
  `base + max_level ≤ 26` / `R_max ≤ cloy_cap − 1` guards.

### Dependency Graph

Every declared dependency has a GDD on disk.

- ✓ settlement-and-entry-fee.md — exists (`:337`)
- ✓ save-system.md — exists (`:338`)
- ✓ data-config.md — exists (`:339`)
- ✓ run-flow.md — exists (`:345`)
- ✓ stomach-capacity.md — exists (`:346`)
- ✓ menu-waves.md — exists (`:347`)
- ✓ cloying-and-taste.md — exists (`:348`)
- ✓ ui-flow.md — exists (`:349`)
- ✓ unlocks-and-stations.md — exists (`:350`)
- ✓ records-and-milestones.md — exists (`:351`)
- ✓ first-run-onboarding.md — exists (`:352`)
- ✓ audio-feedback.md — exists (`:353`) — **but it does not acknowledge this
  system** (see Blocking 3)
- ✓ scene-flow.md — exists (`:354`)

Bidirectional check: settlement (`:335`), save (`:270`), data-config (`:181`),
cloying (`:60`), menu-waves (`:227`), ui-flow (`:427`), unlocks (`:91`),
records (`:303`), onboarding (`:372`) and scene-flow all acknowledge the
relationship. Audio is the sole one-way edge. Run-flow acknowledges it in prose
(`:307`) rather than a dependency table row, which is consistent with its
"indirect" classification.

### Registry Values

Registry entries this GDD owns — `wallet_after_run` (`entities.yaml:948`),
`upgrade_price` (`:956`), `upgrade_tree_cost` (`:970`), `menu_vision_useful_max`
(`:983`) — match the formulas byte-for-byte. Entries it consumes match too:
`extra_satiety` expression and floor-once note (`:487-492`), `net_profit`
(`:614-620`), `calibration_ratio` including the maxed band 120/85 = 1.412
(`:469-477`), `opt_value = 104` (`:443-455`).

Searched the whole document for `cloy_cap`, `wave_size`, `base_capacity`,
`entry_fee`, `turn_budget`, `opt_value`, `calibration_ratio`, `first_run`,
`upgrade_price`, `tree_cost`, `useful_max`. No value conflicts. Independent
checks:

- **Price chain** — `12, 17, 22, 30, 40` (tree 121), `16, 22, 30` (68),
  `20`; full tree 209, all reproduce with `ceil(cost × 1.35^(L−1))`.
- **Capacity ceilings** — exhaustive search over the 12-dish table (T = 10)
  reproduces the table at `:254-257` exactly: C = 20..26 → 104, 106, 107, 112,
  116, 120, 121. The claim that 26 cells is the effective ceiling (27+ flat at
  121) is also true.
- **Calibration** — 104/85 = 1.224 in `[1.15, 1.35]`; 120/85 = 1.412 in the
  maxed band `[1.35, 1.45]`.
- **Resistance** — `max(0, min(group_cloy, cloy_cap) − floor(R))` matches
  `cloying-and-taste.md:76` and the registry.
- **First-run fee** — `entities.yaml:1276-1284` lists `first_run_entry_fee = 40`
  with `meta-upgrades.md` in `referenced_by`, but this document never mentions
  40 and its OQ 1 recommends 0. Not a numeric conflict with this GDD's values,
  but the reference is currently unjustified (Recommended 2).

Registry gaps (belong to the orchestration session): `cloy_cap` (`:1155`),
`station_canteen_entry_fee` (`:1183`) and `base_capacity` (`:1270`) are all used
by this GDD but omit `design/gdd/meta-upgrades.md` from `referenced_by`.

### Required Before Implementation

1. **Settlement transaction shape conflicts with `ui-flow` and `save-system`.**
   Rule 6 (`:37`) fixes `apply_run_result` as "内存应用 → `commit` → 收据",
   and the interface table (`:122`) plus AC 2 (`:444`, "`commit` 被调用一次")
   and AC 5 (`:447`, "恢复写盘并 `flush`") assume this system owns the write.
   `ui-flow.md:59` assigns the settlement transaction to 界面流程: records →
   wallet → **one** `存档.commit` of `progression` + `records` (+ `unlocks`),
   and states that `apply_run_result` must be splittable into
   prepare/commit; `save-system.md:378` AC 13 requires exactly one disk write
   for a settlement. As written, three documents disagree on how many writes
   happen and who performs them — and `save-system`'s atomicity guarantee
   cannot be met. Fix: either define `apply_run_result` as a prepare-only call
   whose commit is performed by the orchestrator (and reword AC 2/5), or
   explicitly accept two sequential commits and state the rollback rule when
   the records namespace fails. `ui-flow` OQ 1 (`:618`) already tracks the
   decision; this GDD must acknowledge it instead of asserting a fixed order.

2. **`WALLET_OVERFLOW` has two record outcomes.** Rule 4 (`:33`) says the
   wallet is unchanged and "本局仍照常结算给纪录系统"; the edge case
   (`:294`) repeats "本局仍交由纪录系统处理". `ui-flow.md:380` says on
   `WALLET_OVERFLOW` the settlement transaction is not committed and
   "纪录更新一并放弃". Since overflow is this system's return value, this GDD
   is the document that must define whether the record survives. Pick one and
   make the other document match; if records should survive, say how (records
   applied independently, wallet write skipped) and add the AC.

3. **Audio dependency is one-way.** The downstream row (`:107`) promises
   `purchase_succeeded` / `purchase_failed` / `run_result_applied`, and the
   GDD's own bidirectional rule (`:356`) requires the latter to know. But
   `audio-feedback.md:376` states 局外升级 has no direct call and its events
   produce no sound. Either audio adds the three events (and a row), or this
   GDD drops the row and stops claiming audio coverage. Either way the
   dependency table must not assert a contract the other GDD denies.

### Recommended Revisions

1. **The "待回改" list is stale.** Items 1–7 and 9 of `:364-374` have landed:
   registry (`entities.yaml:948-1000`), data-config `cost_growth` (`:41`) and
   `wave_size` (`:53`), settlement OQ 1/6 (`:486`, `:491`), cloying row and
   formula (`:60`, `:76`, `:340`), menu-waves row (`:227`), save rule 18
   (`:44`), dish-values maxed band (`:236`). The parenthetical at `:37`
   ("需在存档 GDD 补一条规则") and the status annotations at `:339`
   ("`cost_growth` 待补"), `:348` ("公式尚未含抗性项"), `:356` and `:435`
   are likewise out of date. Prune to item 8 (run-flow, optional) and mark the
   rest applied.

2. **Open Question 1 is answered — with 40, not 0.** `first-run-onboarding.md`
   defines `entry_fee_override = 40` (`:38`), gives the reasons zero is wrong
   (`:21`) and shows the first run buys the 12 元 capacity upgrade (`:297`).
   This GDD's OQ 1 (`:491`) still recommends `entry_fee = 0` and the edge case
   (`:327`) still says "免单/低费档口". Update both to the `RunConfig.
   entry_fee_override` solution, and reconcile the registry's
   `first_run_entry_fee` reference (or remove this file from that entry's
   `referenced_by`).

3. **`spend` is only a signature.** Rule 2 (`:29`) makes it the only legal
   third wallet flow, and unlocks depends on it as a hard upstream
   (`unlocks-and-stations.md:13,91,120-138`), but this GDD never states its
   validation order, `BUSY`/single-transaction rule, `extra` merge, or rollback
   on commit failure. Add a short rule mirroring rule 7 (or explicitly defer
   the detail to unlocks' contract), and list `SpendReason`'s values.

4. **Station context is missing from the purchase API.** `NO_EFFECT` uses
   `wave_size(s)` (`:55`, `:234`), and `get_catalog()` must show "本档口上限
   Z 道", yet `next_price` / `can_purchase` / `purchase` / `get_catalog`
   (`:118-124`) take no station and no current-station context is defined. AC 26
   (`:477`) supplies a "`wave_size = 4` 的档口" fixture with no way to select
   it. MVP's single station hides this; OQ 4 (`:494`) covers the multi-station
   *policy* but not the API. Name the parameter (or state the single-station
   assumption and where the current station comes from).

5. **`base_capacity + max_level ≤ 26` has no owner.** `:259` and Tuning Knobs
   (`:382`) call it a data constraint, but `data-config.md:97` `R_upg` cannot
   see `BalanceData` and checks no cross-record rule. Either add the check to
   data-config's validation (it loads both records) or label it a design-time
   review item, as `R_max ≤ cloy_cap − 1` already is.

6. **menu-waves' dependency row states the opposite ownership.** This GDD
   splits ownership cleanly — levels here, cap there (`:53`, `:360`) — while
   `menu-waves.md:64,227` says it "只消费整数等级" and "上限由局外升级保证不
   超卖". The formula (`:103-110`) already consumes a resolved quota, so the
   prose is stale. Separately, `:55` claims to answer menu-waves OQ 3, but
   `menu-waves.md:334` OQ 3 asks about exclusion inference near the last wave,
   which this GDD does not address; re-point the citation.

7. **run-flow OQ 11 is not actually closed.** Cross-References (`:428`) says
   this GDD answers it, and `:206` adds a frozen `levels` copy, but
   `run-flow.md:65` still specifies "至少含" the three resolved keys and
   `:454` still lists OQ 11 as open with this review as its trigger. Have
   run-flow close it and name `levels` as an optional key (or drop the
   `upgrade_snapshot.levels` dependency at `:351`).

8. **Three Open Questions have outlived their trigger.** OQ 3 (`:493`):
   `time-and-turns.md:292` now says MVP has no time upgrade and re-homes the
   decision to Vertical Slice playtest (owner 局外升级 + 用户). OQ 8 (`:498`):
   the records GDD is written; the question is real but the owner/date is
   stale. OQ 10 (`:500`): `ui-flow.md:83` rule 22 places `UpgradeScreen` under
   `Menu` and defines the receipt → upgrade flow, so it can close.

### Specialist Disagreements

None — lean mode runs no specialists (Phase 3b skipped).

### Nice-to-Have

- **ReadOnly missing from rule 19.** `:69` equates `writable = false` with
  `Blocked`, but `save-system.md:36,65-66` has six outcomes, and `ReadOnly` also
  sets `writable = false`. The state table (`:83-90`) already knows
  `ReadOnly`; make rule 19 say "`Blocked` / `ReadOnly`".
- **Invisible resistance levels are still sold.** Edge `:311` allows a
  fractional `effect_value` whose total floors to 0 for the player's current
  level, displaying "当前 0 点 / 下一级后 0 点"; rule 15 refuses `NO_EFFECT`
  only for `MENU_VISION`. Decide whether `CLOYING_RESISTANCE` should also reject
  a zero-gain level (MVP's integer 1.0 never reaches this).
- **`spend` has no state.** The state table (`:81-90`) models only
  `Purchasing`; `BUSY` is described under purchases. A station unlock in flight
  should either reuse `BUSY` explicitly or get a line in the table.
- **Registry `referenced_by` gaps** — `cloy_cap`, `station_canteen_entry_fee`
  and `base_capacity` should list this file; registry edits belong to the
  orchestration session.
- **Player Fantasy** is advisory-absent at `standard`; Game Feel covers the
  intent, so this is optional.
- **Scope** — `systems-index.md:207` records `M`. This review suggests **L**:
  6 formula groups, 3 hard upstreams, 10 downstreams, and a settlement
  transaction that needs an ADR-level decision (ui-flow OQ 1).
- **AC 24** uses `cloying_resistance = 5` outside the design band
  `[0, cloy_cap − 1]`. This is a valid defensive-floor test, but it should say
  so, otherwise a tester may treat the band as violated.

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md bcde83821bc73ad7c67a24d38a8659ce2d53693b
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

---

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 0 | Recommended: 10
Summary: All three prior blocking items are resolved. The settlement hand-off now matches `ui-flow` rule 10 and `save-system` AC 13 (prepare/commit split, one orchestrated write), `WALLET_OVERFLOW` keeps the record update in both documents, and the audio row is now marked optional with `audio-feedback` acknowledging the same events. The economy core is unchanged and still reproduces exactly (price chains 121/68/20, capacity ceilings 104→121, calibration 1.224 / 1.412). What remains is cross-document drift left by the rename (`apply_run_result` still appears twice here and five times in `ui-flow`), an undefined producer for `APPLIED_UNSAVED`, and the stale "待回改"/Open-Question annotations carried over from the first review.
Prior verdict resolved: Yes

Findings:
- [RECOMMENDED] Detailed Design rule 5 (`:35`) / Interactions (`:99`): both still name `apply_run_result` after the interface was renamed to `prepare_run_result`; `ui-flow.md:67,157,383,431,526` and AC 12 (`:565`) also still call `apply_run_result`. One settlement entry-point name across both documents.
- [RECOMMENDED] Rule 6 (`:37`) / AC 5 (`:447`) vs rule 6 + Interface (`:122`): `prepare_run_result` never writes, so it cannot return `APPLIED_UNSAVED`; the only meta call precedes the orchestrator's commit. State who produces that status and when `progression_changed` fires.
- [RECOMMENDED] Interfaces (`:121`) vs `unlocks-and-stations.md:124-131`: `spend` gained only "`amount < 0` 立即拒绝", which contradicts unlocks' fixed order (INVALID_AMOUNT after DATA_NOT_READY/NOT_WRITABLE/NOT_IN_MENU/BUSY); validation order, `extra` merge, rollback and `SpendReason` values are still only in unlocks.
- [RECOMMENDED] Dependencies（待回改）(`:364-374`) / `:339`, `:348`, `:356`, `:435`: still stale — items 1–7 and 9 have landed; `cost_growth` is in `data-config.md:41`, cloying has the resistance term (`:77`,`:92`), `ui-flow`/`first-run-onboarding` both回指 this GDD. Prune to item 8.
- [RECOMMENDED] Open Question 1 (`:491`) / Edge Cases (`:327`): still recommend `entry_fee = 0` / "免单"; `first-run-onboarding.md:21,38` chose 40 via `entry_fee_override` and rejects zero explicitly. `entities.yaml:1282` still lists this file in `first_run_entry_fee.referenced_by` although the doc never cites 40.
- [RECOMMENDED] Interfaces (`:118-120`) / rule 15 (`:55`) / AC 26 (`:477`): `next_price`、`can_purchase`、`purchase`、`get_catalog` still take no station while `NO_EFFECT` needs `wave_size(s)`; AC 26's station fixture cannot be selected.
- [RECOMMENDED] Design-time constraint (`:259`, `:382`): `base_capacity + max_level ≤ 26` is still called a data constraint but `data-config.md:97` `R_upg` cannot see `BalanceData` and has no cross-record check.
- [RECOMMENDED] Rule 15 (`:55`) vs `menu-waves.md:64,227,336`: menu-waves still says it consumes "整数等级" and that the cap is guaranteed by 局外升级; and OQ 3 there is the exclusion-inference/occlusion question, which this GDD does not answer.
- [RECOMMENDED] Cross-References (`:428`) / Dependencies (`:351`) vs `run-flow.md:66,457` and `records-and-milestones.md:84`: run-flow OQ 11 is still open, the snapshot is "至少含" three keys (so `levels` is not contractual), and records says it never interprets the snapshot — yet `:105` claims it freezes from `upgrade_snapshot.levels`.
- [RECOMMENDED] Open Questions 3 / 8 / 10 (`:493`, `:498`, `:500`): triggers have passed — `time-and-turns.md` OQ 3 now re-homes MVP time upgrades to the VS playtest, the records GDD exists, and `ui-flow.md:84` rule 22 defines `UpgradeScreen` under `Menu`.

---

## Design Review: 局外升级（Meta Upgrades）

Specialists consulted: none (lean mode)
Re-review: Yes — prior verdict was NEEDS REVISION on 2026-11-04

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules
(`## Detailed Design` is the accepted heading), Formulas (numeric rules:
price curve, tree cost, snapshot resolution, resistance offset, preview cap),
Edge Cases, Dependencies, Acceptance Criteria. All six present.

Advisory at `standard`: Player Fantasy absent as a heading (`## Game Feel`
carries the intent); Tuning Knobs present with safe ranges and the
`base + max_level ≤ 26` / `R_max ≤ cloy_cap − 1` guards.

### Prior blocking findings — resolution

1. **Transaction shape — RESOLVED.** Rule 6 (`:37`) now makes 界面流程 the
   settlement orchestrator, calls `prepare_run_result` + records `prepare_run`,
   and performs one `save.commit` of `progression` + `records`; the interface
   table (`:122`) documents prepare-only; AC 2 (`:444`) and AC 5 (`:447`) were
   reworded around the orchestrator. `ui-flow.md:61` rule 10 matches (records →
   wallet → one commit; prepare stages write nothing) and its OQ 1 is marked
   answered; `save-system.md:378` AC 13 still requires exactly one disk write and
   the keep-in-memory/dirty-retry policy matches. Remaining work is naming and
   status-origin (Recommended 1–2), not the write count.
2. **`WALLET_OVERFLOW` record outcome — RESOLVED.** `ui-flow.md:384` now says
   the wallet is unchanged but the record still updates for this run, matching
   rule 4 (`:33`) and edge (`:294`).
3. **Audio — RESOLVED.** `audio-feedback.md:380` now names the three events and
   states MVP does not map them to cues; this GDD marks the row 下游（可选）and
   says MVP 不消费 (`:107`, `:353`). The edge is bidirectional.

### Dependency Graph

Every declared dependency has a GDD on disk; all upstream/downstream rows now
回指 this system (audio included). No broken references.

### Registry Values

Owned entries — `wallet_after_run` (`entities.yaml:946`), `upgrade_price`
(`:960`), `upgrade_tree_cost` (`:975`), `menu_vision_useful_max` (`:988`) —
match the formulas byte-for-byte. Consumed entries match too: `extra_satiety`
includes `cloying_resistance` with the floor-once note (`:479-492`),
`calibration_ratio` carries the maxed band 120/85 = 1.412 (`:462-477`),
`net_profit` (`:609-620`), `opt_value = 104` (`:443-455`). No numeric conflicts.

Independent checks (unchanged inputs): price chains `12/17/22/30/40` (121),
`16/22/30` (68), `20`, full tree 209; capacity ceilings 104, 106, 107, 112,
116, 120, 121 at C = 20–26 with 27+ flat; calibration 1.224 and 1.412.

Registry gaps that remain (orchestration session): `cloy_cap` (`:1155`),
`station_canteen_entry_fee` (`:1183`), `base_capacity` (`:1263`) still omit
`design/gdd/meta-upgrades.md` from `referenced_by`; `first_run_entry_fee`
(`:1276-1283`) lists it without the document citing 40.

### Required Before Implementation

None.

### Recommended Revisions

1. **Finish the API rename inside this GDD.** Rule 5 (`:35`) and the
   Interactions row (`:99`) still say `apply_run_result`. `ui-flow.md` still
   names `apply_run_result` in its interface table (`:157`), dependency table
   (`:431`), cross-reference (`:526`), AC 12 (`:565`), rule 13 (`:67`) and edge
   (`:383`), while its rule 10 uses `prepare_run_result`. Pick one name and
   apply it everywhere.
2. **Define who returns `APPLIED_UNSAVED`.** `prepare_run_result` returns
   `{progression_update, status}` before the orchestrator commits, so it cannot
   observe a write failure; rule 6 (`:37`) and AC 5 (`:447`) nevertheless say
   "返回 `APPLIED_UNSAVED`", and `ui-flow` rule 10 step ② lists it among the
   values prepare handles. Say whether the orchestrator's settlement result
   carries it (and meta only marks dirty), and when `progression_changed`
   fires relative to the commit.
3. **Make `spend` match the contract it defers to.** `:121` adds
   "`amount < 0` 立即拒绝", but `unlocks-and-stations.md:127` places
   `INVALID_AMOUNT` after `DATA_NOT_READY` / `NOT_WRITABLE` / `NOT_IN_MENU` /
   `BUSY`; "立即" contradicts that order. Add the validation order, `BUSY`
   sharing, `extra` merge and rollback here (mirroring rule 7), or state that
   `unlocks-and-stations.md` owns them. Name the `SpendReason` values.
4. **Prune the stale "待回改" list and annotations.** Items 1–7 and 9 of
   `:364-374` have landed. `:339` ("`cost_growth` 待补") — `data-config.md:41`
   defines it; `:348` ("公式尚未含抗性项") — `cloying-and-taste.md:77,92` has
   it; `:356` — cloying lists the row and answers OQ 9, menu-waves lists the
   row; `:435` — both `ui-flow` and `first-run-onboarding` now 回指 this GDD.
5. **Replace the 0-元 first-run recommendation with 40.** OQ 1 (`:491`) and the
   edge case (`:327`) still suggest zero/免单; `first-run-onboarding.md:21,38`
   chose `entry_fee_override = 40` and rejects zero for a reason. Update both
   and either justify or drop this file from `first_run_entry_fee.referenced_by`.
6. **Give the purchase API a station context.** `NO_EFFECT` and the displayed
   cap depend on `wave_size(s)` (`:55`, `:234`) but `next_price` /
   `can_purchase` / `purchase` / `get_catalog` (`:118-124`) take only an id, and
   AC 26's fixture cannot select its station. Name the parameter, or record the
   single-station assumption and its source.
7. **Assign the `base_capacity + max_level ≤ 26` check.** `:259` and `:382`
   call it a data constraint, but `R_upg` (`data-config.md:97`) validates only
   per-record fields. Add a cross-record rule in data-config or label it a
   design-time review item (as `R_max ≤ cloy_cap − 1` already is).
8. **Fix the menu-waves ownership prose and OQ citation.** `menu-waves.md:64`
   and `:227` still say it consumes "整数等级" and that 局外升级 guarantees no
   overselling, while this GDD owns levels and menu-waves owns the cap. Also
   `:55` cites menu-waves OQ 3 (`:336`), which asks about exclusion inference
   near the last wave — not the numeric cap answered here.
9. **Close run-flow OQ 11 or drop the `levels` claim.** `run-flow.md:457` still
   lists it open and `:66` says the snapshot "至少含" the three resolved keys;
   `records-and-milestones.md:84` states it never interprets the snapshot. So
   `:428` (claims the answer) and `:105`/`:351` (records freezes from
   `upgrade_snapshot.levels`) assert more than the other documents grant.
10. **Retire the outlived Open Questions.** OQ 3 (`:493`): `time-and-turns.md`
    OQ 3 already re-homes MVP time upgrades to the Vertical Slice playtest.
    OQ 8 (`:498`): the records GDD exists; keep the question, update owner/date.
    OQ 10 (`:500`): `ui-flow.md:84` rule 22 defines `UpgradeScreen` under `Menu`
    with the receipt → upgrade flow.

### Specialist Disagreements

None — lean mode runs no specialists (Phase 3b skipped).

### Nice-to-Have

- Rule 19 (`:69`) still equates `writable = false` with `Blocked`; save also
  returns `ReadOnly` (both non-writable). The state table already knows both.
- Fractional resistance can still sell an invisible 0→0 level (`:311`);
  rule 15 rejects `NO_EFFECT` only for `MENU_VISION`.
- `spend` has no state-table row; `Purchasing` is the only in-flight state
  modelled here (unlocks owns `Unlocking`).
- AC 24 (`:475`) uses `cloying_resistance = 5` outside the design band
  `[0, cloy_cap − 1]`; label it a defensive-floor test.
- Scope: `systems-index.md:207` records M; this review still suggests **L**
  (6 formula groups, 3 hard upstreams, 10 downstreams, one ADR-level settlement
  decision).
- Player Fantasy is advisory-absent at `standard`; Game Feel covers the intent.

### Verdict: APPROVED

---

Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/save-system.md 772ed596d68eb5cbae207b7bac9a170ec55c1524
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 5734623b4a60d9072657ef631c8ae61f05359f21
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/menu-waves.md ce532d3265df1ed7555ba5aa46383be35bbee4a1
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 9fe5183795ca3d47798510a1d00a040dabe3ca61
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/systems-index.md 15e0bd3d307ed09738b74dd3c099816b497b69e8
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
