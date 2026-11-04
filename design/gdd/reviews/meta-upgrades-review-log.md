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
