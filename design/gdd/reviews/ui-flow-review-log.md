# Design Review Log — 界面流程 (UI Flow)

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 8 | Recommended: 12
Summary: The navigation core is unusually precise — the screen/modal stack model, the total `back_target` decision table, the focus and `ui_button_activated` contracts, and the "receipt never recomputes" boundary all hold. The blockers are at the seams this GDD inherits rather than invents: the first-run `RunConfig` assembly never takes 首局引导's preset, the `WALLET_OVERFLOW` outcome contradicts both `meta-upgrades` and `records`, the single-commit acceptance criterion cannot hold while `apply_run_result` commits internally, two rendered values (dish names, milestone copy) have no declared data source, one cited save rule does not exist, and the fixed screen inventory omits two screens the rest of the document treats as normative. Several of these sit on interfaces currently being revised (#4 / #11 / #12), and are marked as such.
Prior verdict resolved: First review

Findings:
- [BLOCKING] Detailed Design rule 8 (`:55`) / AC 6 (`:552`) / Interactions (`:161`) / Dependencies (`:437-439`): first-run assembly omits 首局引导's preset (`pool_override` six dishes, `entry_fee_override = 40`); `is_first_run = true` is sent with `pool_override = null` and no fee override, so the teaching run runs on the full pool at 85 instead of the designed six dishes at 40.
- [BLOCKING] Edge Cases (`:380`) / rule 10 step ② (`:59`): `WALLET_OVERFLOW` is stated to abandon the record update, while `meta-upgrades.md:33,294` says the run still reaches records and `records-and-milestones.md:29` does not list it as fatal; the stated order (records applied first, `record_broken` already emitted) makes the abandonment unhonorable.
- [BLOCKING] rule 10 step ③ (`:59`) / AC 11 (`:560`) require `commit` exactly once across `progression` + `records`, but `meta-upgrades.md:37,122` has `apply_run_result` commit internally and `save-system.md:378` AC 13 forbids two writes. **Depends on the in-flight `meta-upgrades` revision (its review log Blocking 1 records the same conflict).**
- [BLOCKING] rule 11 (`:61`) / UI Requirements (`:493`) / Dependencies (`:159,431`): receipt rows include 菜名 and claim to read it from `RunResult`, but `RunResult.rows` carries only `dish_id` (`settlement-and-entry-fee.md:83`) and the 数据配置 dependency declares no `get_dish`.
- [BLOCKING] rule 10 (`:59`) / Open Question 1 (`:618`): both cite "`存档` 规则 80", which does not exist — `save-system.md` has 25 core rules. The intended references are rule 18 (`:44`) and AC 13 (`:378`).
- [BLOCKING] rule 2 (`:31-43`) vs UI Requirements (`:499-500`): the normatively "fixed" screen inventory omits `BlockedModal` and `ReadOnlyNotice`, both of which the rest of the document treats as distinct screens with distinct behavior.
- [BLOCKING] UI Requirements (`:501`) vs `back_target` (`:210-240`): a dismissible non-modal notice is closed with `ui_cancel`, but the return decision table has no notice input, so at `MenuRoot` the same key opens the quit confirm instead of dismissing.
- [BLOCKING] Dependencies (`:159,431`) / rule 23 (`:85`): the milestone catalogue (names, condition copy, thresholds) has no owner — `data-config.md:28,186` keeps `MilestoneData` out of the schema pending records Open Question 2, and `records-and-milestones.md` exposes no display name or condition string.
- [RECOMMENDED] Open Question 14 (`:631`): already answered by `input.md:55` — `push_context` always pushes and emits `context_changed(Modal, Modal)`; close it.
- [RECOMMENDED] Open Question 1 (`:618`): "纪录致命错误时回滚已入账钱包" contradicts rule 10's own order (records first); the real exposure is a records commit failing after the wallet commit.
- [RECOMMENDED] Acceptance Criteria: no criterion covers `LoadReport = ReadOnly` (rule 20 `:79`, UI row `:499`), the state `save-system.md:65,323` just added.
- [RECOMMENDED] Open Question 8 (`:625`): names three `Blocked` reasons, while `save-system.md:324` defines two player-facing copy groups; align.
- [RECOMMENDED] Dependencies (`:439`) vs `unlocks-and-stations.md:92,366`: #13 declares a hard UI dependency (`unlock(id)`, station list, selection); this GDD lists it soft/"将来" and the fixed inventory has no place for it.
- [RECOMMENDED] Interactions (`:161-162`) still say #18 / #13 "未设计" while the Dependencies table (`:438-439`) says "GDD 已写".
- [RECOMMENDED] rule 2 (`:40`): `SettingsModal` is listed for "任意有界面的状态", but `ErrorScreen` has only 退出 (`:75`) and no transition opens settings there; restrict to `Menu` / `Paused`.
- [RECOMMENDED] UI Requirements (`:494-495`): `UpgradeScreen` / `RecordsScreen` initial focus falls back to a 「返回」 control that their 必须包含 lists do not include.
- [RECOMMENDED] Tuning Knobs: absent (advisory at `standard`); name `RECEIPT_ROLL_SECONDS`, `SAVE_INDICATOR_SECONDS`, `MAX_UI_STACK_DEPTH`, `MAX_MODAL_DEPTH` in one place.
- [RECOMMENDED] Registry (`design/registry/entities.yaml`): `ui-flow.md` is missing from `referenced_by` of `net_profit`, `total_value`, `return_ratio`, `broke_even`, `station_canteen_entry_fee`; it is falsely listed under `station_available`, `unlock_decision`, `unlock_progress`, which the GDD never mentions. Orchestration-session edit.
- [RECOMMENDED] `records-and-milestones.md:120,442`: still says the settlement orchestrator is unspecified; this GDD answers it and the answer must be recorded there.
- [RECOMMENDED] `audio-feedback.md:110,360`: names the receipt-close notification generically; adopt `receipt_closed(reason)` so the event name is fixed on both sides.
- [RECOMMENDED] `input.md:246` Open Question 2 (gamepad glyphs) was due "界面流程 GDD 编写时"; route it to `/ux-design` explicitly.

Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/input.md dd15ea96523a49136b96cfef827c5619dee75e6a
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md bcde83821bc73ad7c67a24d38a8659ce2d53693b
Reviewed-Content-Hash: design/gdd/systems-index.md 182d446c25ab8aa50ccf7d672bc2132eeb2b22d3

---

## Design Review: 界面流程 (UI Flow)
Specialists consulted: none (lean mode — Phase 3b not run)
Re-review: No — first review

### Completeness: 6/6 sections present, where 6 is the count required at `standard`

`modes.workflow: standard`. Required: Overview, Detailed Rules
(`## Detailed Design` is the accepted heading), Formulas (the system defines
numeric rules — stack depth, modal depth, a return-decision function, focus
selection, latency assertion, roll timing), Edge Cases, Dependencies,
Acceptance Criteria. All six are present.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:473-482`) carries
  the intended feeling ("翻页，不是过场"; confirmation that "稳").
- **Tuning Knobs** — absent; the constants exist but are scattered
  (`RECEIPT_ROLL_SECONDS` `:67,335`, `SAVE_INDICATOR_SECONDS` `:81`,
  `MAX_UI_STACK_DEPTH` `:47`, `MAX_MODAL_DEPTH` `:49`).

### Dependency Graph

Every declared dependency has a GDD on disk.

Upstream (9):
- ✓ `design/gdd/scene-flow.md` — exists (In Review)
- ✓ `design/gdd/input.md` — exists (In Review)
- ✓ `design/gdd/run-flow.md` — exists (Approved)
- ✓ `design/gdd/settlement-and-entry-fee.md` — exists (Needs Revision)
- ✓ `design/gdd/meta-upgrades.md` — exists (Needs Revision)
- ✓ `design/gdd/records-and-milestones.md` — exists (Designed)
- ✓ `design/gdd/save-system.md` — exists (Needs Revision)
- ✓ `design/gdd/audio-feedback.md` — exists (Designed)
- ✓ `design/gdd/data-config.md` — exists (Approved)

Downstream (3):
- ✓ `design/gdd/audio-feedback.md` — exists
- ✓ `design/gdd/first-run-onboarding.md` — exists (Designed)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (Designed)

Not dependencies, correctly flagged as future work: `design/ux/[screen].md`
(7 screens) and `design/accessibility-requirements.md` do not exist; the GDD
names them as required before Phase 4. These are forward references, not broken
links.

### Registry Values

Searched the whole document for the registry entries it owns or references:
`unit_value`, `effective_unit_value`, `total_value`, `net_profit`,
`return_ratio`, `broke_even`, `capacity_used`, `wasted_capacity`,
`cloy_extra_total`, `opt_value`, `entry_fee`, `first_run_entry_fee`,
`load_outcome`, `station_available`, `unlock_decision`, `unlock_progress`.

- **Matching:** `122.3%` (`:61`) equals settlement's `1223 / 1000`; `entry_fee = 0`
  and `net_profit > 0` stamp rules match `records-and-milestones.md:366`;
  `broke_even` is read as the derived predicate the registry defines, never
  stored; no restated numeric value contradicts the registry.
- **Not stated:** `first_run_entry_fee = 40` — its absence from this GDD is
  Blocking 1, not a value conflict.
- **Registry maintenance (belongs to the orchestration session):** `ui-flow.md`
  is absent from `referenced_by` of `net_profit`, `total_value`, `return_ratio`,
  `broke_even`, `station_canteen_entry_fee`, although the GDD reads all five.
  Conversely the registry lists `ui-flow.md` under `station_available`,
  `unlock_decision` and `unlock_progress` (`entities.yaml:1008,1039,1084`),
  which this document never references.

### Required Before Implementation

All findings are from the main review; no specialists ran.

**1. The first-run `RunConfig` never takes 首局引导's preset — the teaching run
is not the designed run.** [main review]
`rule 8` (`:55`) assembles `pool_override = null` unconditionally, omits
`entry_fee_override` entirely, and sets `is_first_run` from
`records.snapshot().career.runs_settled == 0`. `first-run-onboarding.md:33-38`
requires the opposite for exactly that case: before assembling `RunConfig`, the
UI flow must take `{pool_override: [six dishes], entry_fee_override: 40}` from
首局引导 and put it into the config; onboarding's interface table (`:129`)
declares 界面流程 a **hard** UI dependency for it. `run-flow.md:64` defines the
field as optional with `null` default, so nothing rejects the omission — the
first run simply runs the full pool at 85, and onboarding's B0/B2 copy and the
40-元 calibration never trigger. AC 6 (`:552`) bakes the omission in
(`pool_override = null`), so a fix must also change that criterion. Two
secondary issues fall out of this: this GDD lists #18 as a **soft**
anchor-only downstream (`:161,438`), and 首局引导 publishes no named API for
"取一份预置" (`:33` says only "向本系统取一份预置"), so the interface must be
named on both sides.
Registry: `first_run_entry_fee = 40` (`entities.yaml:1276`). Settlement's
review log (Blocking 2) confirms 40 is the current value.

**2. `WALLET_OVERFLOW` has three incompatible outcomes.**
`ui-flow.md:380` says the record update is abandoned with the transaction;
`meta-upgrades.md:33` and `:294` say the run "仍照常结算给纪录系统 / 仍交由
纪录系统处理"; `records-and-milestones.md:29,120` lists only
`INCONSISTENT_RESULT` and `OVERFLOW` as fatal and never mentions a wallet
overflow. The GDD's own order makes the stated policy unhonorable: step ① runs
`records.apply_run` and step ② runs `apply_run_result`, and `apply_run` emits
`record_broken` / `milestone_earned` synchronously (`records:111-116`) — by the
time the overflow is known, the record has already been applied and signalled,
so "纪录更新一并放弃" would require a rollback records does not define. The
receipt would also contradict itself (no stamp per `:380`, but
`record_broken` already delivered).
**This depends on the in-flight `meta-upgrades` revision**, whose review log
(Blocking 2) records the same ui-flow-vs-meta conflict; whichever side changes,
this GDD must end on the same outcome as meta and records.

**3. The single-commit criterion cannot pass against the current
`apply_run_result`.** [main review]
`rule 10` step ③ (`:59`) requires `progression` + `records` (+ future
`unlocks`) merged into **one** `save.commit`, and AC 11 (`:560`) asserts
`apply_run → apply_run_result → commit` each exactly once. But
`meta-upgrades.md:37,122` defines `apply_run_result` as "内存应用 → `commit` →
返回", i.e. it writes by itself, and `save-system.md:378` AC 13 requires the
settlement-caused wallet + records + unlock changes to be a single disk
transaction. Under the current interface the commit count is at least two and
AC 11 is false. Open Question 1 (`:618`) acknowledges this and defers to
architecture; that is acceptable as a stated open question, but the body and
AC must then be marked provisional rather than asserted as rules — as written,
a QA pass on AC 11 fails on a design the document calls normative.
**Depends on the in-flight `meta-upgrades` revision and a prepare/commit split
or an equivalent single-writer arrangement; flagged, not resolved here.**

**4. Receipt rows render a dish name that no declared source provides.**
`rule 11` (`:61`) and the UI Requirements table (`:493`) require each row to
show 菜名, and the rule states every row field is read directly from
`RunResult`. `RunResult.rows` contains `dish_id`, `price`, `satiety`,
`extra_satiety`, `effective_satiety`, `unit_value`, `effective_unit_value`
(`settlement-and-entry-fee.md:83`) — no name. The 数据配置 dependency
(`:159,431`) declares only `StationData.display_name` / `entry_fee` and the
milestone catalogue; `get_dish(id)` is never listed, although
`DishData.display_name` exists (`data-config.md:31`). Pick one and write it:
read `get_dish(row.dish_id).display_name` as a display-only dependency (and add
the counterpart row to `data-config.md:188`), or have `RunResult.rows` carry
the name (a settlement-side change). Today an implementer must guess.

**5. Two citations point at a rule that does not exist.**
`rule 10` (`:59`) and Open Question 1 (`:618`) both cite "`存档` 规则 80" as
the source of the single-transaction requirement. `save-system.md` has 25 core
rules; the intended references are rule 18 (`:44`, retry policy) and AC 13
(`:378`, one transaction for settlement-caused changes). `unlocks-and-stations.md:55`
carries the same stale citation, and `save-system-review-log.md` already records
it as a cross-document wording item — fixing it here is a one-line edit but the
requirement is unreadable until it is done.

**6. The "fixed" screen inventory omits two screens the document depends on.**
`rule 2` (`:31-43`) is normative — "新增屏幕必须改本 GDD，不得由实现临时增加"
— and lists eleven entries: `BootWaiting`, `ErrorScreen`, `MainMenu`,
`UpgradeScreen`, `RecordsScreen`, `ReceiptScreen`, `PauseOverlay`,
`SettingsModal`, `ConfirmModal`, `NoticeLayer`, 局内一屏. The UI Requirements
table then defines `ReadOnlyNotice` (`:499`) and `BlockedModal` (`:500`) as
separate screens, and `rule 20` (`:79`) gives `BlockedModal` behavior that no
listed screen has (forced modal, `ui_cancel` a no-op, no cancel button, default
focus 退出). Either add both rows with their owners and 形态, or declare them
instances of `ConfirmModal` / `NoticeLayer` and state the overridden rules in
`rule 2` itself. As written, a programmer following the inventory will not
build the `Blocked` path.

**7. A dismissible non-modal notice has no precedence in the return model.**
`UI Requirements` (`:501`) says dismissible items in `NoticeLayer` close on
`ui_accept` / `ui_cancel`. The `back_target` function (`:210-240`) takes only
`state`, `screen_id`, `modal_depth`, `modal_cancel_enabled` and `roll_active`,
so with a `Recovered` or `ReadOnly` notice open at `MenuRoot`, `ui_cancel`
resolves to rule 6 → `OPEN_QUIT_CONFIRM` and the notice never closes. The
`ReadOnlyNotice` row (`:499`) is described as non-modal yet with two buttons,
and has no focus order, so the focus contract (`:51`, UI Requirements `:503`)
does not cover it either. Decide the model — either a dismissible notice
consumes `ui_cancel` before rule 6 (and `back_target` gains the input), or
`ReadOnlyNotice` is a modal (it already gates purchase/unlock entry), in which
case the existing modal rules apply unchanged.

**8. The milestone catalogue has no owner, so the Records screen and the
"新达成" strip cannot be rendered.** [main review]
`rule 23` (`:85`) and the 数据配置 dependency (`:159,431`) read milestone
names, condition copy, thresholds and `required_for_completion` from 数据配置.
`data-config.md:28` states `MilestoneData` is **not** in the schema — its
existence is records Open Question 2 — and `data-config.md:186` marks the row
conditional on that question. `records-and-milestones.md` exposes
`snapshot()` and `milestone_progress(id)` (`:111-116`) returning only
`current / threshold / unit / earned`, with no display name or condition text.
So the two values the UI most needs have no source on either side.
**Depends on records Open Question 2 / data-config's schema decision**; the
dependency row must name whichever owner wins.

### Recommended Revisions

1. **Close Open Question 14** (`:631`). `input.md:55` now states the opposite
   of what the question asks: "`push_context` 总是增加栈深并发出
   `context_changed(from, to)`，包括同值压栈 `context_changed(Modal, Modal)`".
   The nested delete-career confirm is safe; the question is answered.
2. **Reword Open Question 1** (`:618`). Its parenthetical says the fallback
   must "在纪录致命错误时回滚已入账钱包", but rule 10 runs records first, so a
   records fatal error means the wallet was never touched. The real exposure is
   the opposite order's failure: wallet committed, records commit fails. State
   that.
3. **Add an acceptance criterion for `ReadOnly`.** AC 25 (`:580`) covers
   `Fresh / Loaded / Migrated / Recovered`; AC 26 covers `Blocked`; no AC
   exercises `ReadOnly` (notice shown once, purchase/unlock entries hidden, no
   "新建生涯", `ui_cancel` behavior). `save-system.md:65,323` added this state
   recently; without an AC a regression to the old five-state model passes.
4. **Align the `Blocked` reason list.** OQ 8 (`:625`) says three reasons
   (更新版本 / 无法读取 / 另一实例); `save-system.md:324` defines two
   player-facing copy groups (存档属于更新版本 / 存档与备份都无法读取). The
   "另一实例" case that is *readable* is `ReadOnly`, not `Blocked`; make the
   question match save's copy split.
5. **Resolve the #13 hardness mismatch.** `unlocks-and-stations.md:92,366`
   declares 界面流程 a hard UI dependency with `unlock(id)`, a station list and
   station selection; this GDD lists it soft/"将来" (`:162,439`) and the fixed
   inventory has no unlock screen. Deferring to Vertical Slice is defensible,
   but say so as a dated deferral and add the future screen as a named item in
   rule 2 rather than leaving it implicit.
6. **Fix stale statuses in the Interactions table** (`:161-162`): #18 and #13
   are marked "未设计" there while the Dependencies table (`:438-439`) says
   "GDD 已写".
7. **Restrict `SettingsModal`'s scope** (`:40`). "任意有界面的状态" includes
   `Error`, but `rule 18` (`:75`) gives `ErrorScreen` one exit and no settings
   entry, and no transition table row opens settings from `Error`. Write
   `Menu` / `Paused`.
8. **Add 「返回」 to the must-include lists** for `UpgradeScreen` and
   `RecordsScreen` (`:494-495`); both initial-focus rules fall back to it.
9. **Add a Tuning Knobs section** (advisory): `RECEIPT_ROLL_SECONDS`
   (0.3–1.5, default 0.8), `SAVE_INDICATOR_SECONDS` (1.5),
   `MAX_UI_STACK_DEPTH` (2), `MAX_MODAL_DEPTH` (2), with what each affects.
10. **Registry maintenance** — `design/registry/entities.yaml` (orchestration
    session): add `ui-flow.md` to `referenced_by` for `net_profit`,
    `total_value`, `return_ratio`, `broke_even`, `station_canteen_entry_fee`;
    remove it from `station_available`, `unlock_decision`, `unlock_progress`,
    which this GDD never references.
11. **Record the orchestrator answer where it was asked.**
    `records-and-milestones.md:120,442` still says the orchestrator is
    unspecified. This GDD answers it (界面流程); the answer must land in
    records, or the next reader of records will re-open a closed decision.
12. **Fix the receipt-close event name on the audio side.**
    `audio-feedback.md:110,360` describes "收据关闭时通知本系统" without naming
    the signal; adopt `receipt_closed(reason)` so the event catalogue and this
    GDD agree. Also route `input.md:246` OQ 2 (gamepad glyphs, due "界面流程
    GDD 编写时") to `/ux-design` now that this GDD exists.

### Specialist Disagreements

None — lean mode, no specialist agents were spawned (Phase 3b skipped by
design). Every finding above is from the main review.

### Nice-to-Have

- AC 27 (`:582`) says "约 1.5 秒"; with F2's injectable clock the value is
  exact — say `SAVE_INDICATOR_SECONDS` rather than "约".
- AC 44 (`:611`) reads "收据关闭 / 返回菜单 / 开始新一局 … 各发一次
  `receipt_closed(reason)`"; restate as three given/when/then arms so each
  reason string is asserted.
- Once finding 7 is resolved, add a criterion for notice dismissal ordering
  (notice open + `ui_cancel` at `MenuRoot`).
- `text_scale_percent` values (100/125/150) match `in-game-hud.md:227`; no
  action, noted so a future edit does not drift.
- The cross-reference row for `systems-index.md` (`:528`) describes the index
  edit as still needed; it has landed (`systems-index.md:72`). Update the row.

### Senior Verdict

Not applicable — `lean` mode runs no `creative-director` synthesis; the verdict
below is the main review's.

### Scope Signal

Rough scope signal: **L** (producer should verify before sprint planning).
Nine hard upstream systems plus three downstream, seven formula blocks, a
settlement transaction role that spans four systems, and at least two likely
ADRs (screen controller / stack implementation; machine-level settings
serialization — itself Open Question 3).

### Verdict: NEEDS REVISION

The navigation, focus and receipt-display design is sound and precise enough to
implement; the blockers are seams with adjacent GDDs and three internal gaps
(screen inventory, notice return path, two unsourced display values). None
requires redesigning the screen model. Findings 2, 3, 4 (partly) and 8 touch
interfaces that are being revised in parallel — `meta-upgrades`, `settlement`,
`save` — and should be re-checked against their fixed versions at re-review.

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 3 | Recommended: 12
Summary: Re-review of the fix wave that added the first-run preset, the `prepare_run` / `prepare_run_result` / single-`commit` orchestration, the dish-name lookup, the two missing screens, and the notice return path. Five of the eight prior blockers are fully cleared. Three remain, all at the same seams: the settlement interface is now named two different things across this document (and `records` still publishes the old one), the new notice priority sits ahead of the modal rules it is supposed to yield to, and milestone-catalogue ownership was corrected in the Interactions table but not in rule 23 or the cross-references.
Prior verdict resolved: No — 5 of 8 prior blockers fully resolved, 3 partially resolved

Findings:
- [BLOCKING] rule 10 (`:61`) / rule 11 (`:63`) / rule 13 (`:67`) / Interactions (`:157`) / Dependencies (`:431-432`) / Cross-References (`:526-527`) / AC 12 (`:565`) / AC 18 (`:571`) / F3 (`:540`): the settlement interface is named inconsistently. This document now calls `纪录与里程碑.prepare_run(result)` and `局外升级.prepare_run_result(result)`, but six sites still say `apply_run` / `apply_run_result`, and `records-and-milestones.md:111` publishes only `apply_run(result)`. Prior Blocker 3 is half fixed: `meta-upgrades` landed `prepare_run_result` (`:122`), `records` did not.
- [BLOCKING] `back_target` (`:220`) vs rule 5 (`:51`) / rule 7 (`:55`): `DISMISS_NOTICE` is priority 0, ahead of both modal rules, so with a dismissible notice up `ui_cancel` dismisses the notice instead of reaching an open modal — contradicting "模态期间的 `ui_cancel` 只作用于模态". Rule 7's prose priority list (①–⑧) omits `DISMISS_NOTICE` entirely, so the normative rule and the formula disagree. Prior Blocker 7 added the notice to the table but not to the rule text, and placed it above modality.
- [BLOCKING] rule 23 (`:87`) / Cross-References (`:530`) vs Interactions (`:158`): rule 23 still reads the milestone catalogue from `数据配置`, while the Interactions table reads `纪录与里程碑.get_milestone_catalog()`. The owner is records: `records-and-milestones.md:292,314` keep `MilestoneData` out of the data-config schema and expose `get_milestone_catalog()` (`:298`); `data-config.md:186` agrees. Prior Blocker 8 fixed one of three sites.
- [RECOMMENDED] Open Question 14 (`:635`): answered by `input.md:56` — `push_context` always pushes and emits `context_changed(Modal, Modal)` even for a same-value push. Close it.
- [RECOMMENDED] Acceptance Criteria: still no criterion for `LoadReport = ReadOnly` (rule 20 `:81`, edge case `:401`, UI row `:503`); AC 45 (`:616`) lists a `Blocked` screenshot but no `ReadOnly` notice.
- [RECOMMENDED] Interactions (`:164`) / Dependencies (`:443`) vs `unlocks-and-stations.md:92,366`: #13 now declares a **hard** UI dependency (station catalogue, `can_unlock` / `unlock(id)`, unlock progress, station selection in the menu). This GDD still says soft / "未设计", and the fixed screen inventory has no station-selection surface. MVP is single-station, so this blocks Vertical Slice planning, not MVP.
- [RECOMMENDED] rule 8 (`:57`): this document names `首局引导.get_first_run_preset()`, but `first-run-onboarding.md` rule 3 publishes the preset's fields only, with no method name. Values and direction now agree; fix the name on one side.
- [RECOMMENDED] rule 11 (`:63`): `record_broken` is said to arrive synchronously "during `apply_run`". With the pure-prepare split, state when the signal fires — during the prepare call or at `commit` — so the receipt's `previous_best` capture has a defined moment.
- [RECOMMENDED] Open Question 8 (`:629`): lists three `Blocked` causes; `save-system.md:324` defines two `Blocked` causes (update version / unreadable-and-no-backup) and treats the third (another instance) as the `ReadOnly` path.
- [RECOMMENDED] rule 2 (`:40`): `SettingsModal` is still "任意有界面的状态"; `Boot` and `Error` have no settings entry, so restrict to `Menu` / `Paused`.
- [RECOMMENDED] UI Requirements (`:498-499`): `UpgradeScreen` / `RecordsScreen` initial focus falls back to a 「返回」 control their 必须包含 lists do not include.
- [RECOMMENDED] Tuning Knobs: still absent (advisory at `standard`) — `RECEIPT_ROLL_SECONDS`, `SAVE_INDICATOR_SECONDS`, `MAX_UI_STACK_DEPTH`, `MAX_MODAL_DEPTH` remain scattered.
- [RECOMMENDED] Registry (`design/registry/entities.yaml`): `ui-flow.md` is missing from `referenced_by` of `net_profit`, `total_value`, `return_ratio`, `broke_even`, `station_canteen_entry_fee`, `first_run_entry_fee`; it is still listed under `station_available`, `unlock_decision`, `unlock_progress`, which the GDD names only as a future soft dependency. Orchestration-session edit.
- [RECOMMENDED] `audio-feedback.md`: still no `receipt_closed(reason)` event; this GDD emits it (`:91,168`). Fix the name on the audio side.
- [RECOMMENDED] `input.md:246` Open Question 2 (gamepad glyphs) was due "界面流程 GDD 编写时"; route it to `/ux-design` explicitly.

### Re-review Verification of Prior Blocking Items

| # | Prior blocking item | Status | Evidence |
|---|---|---|---|
| 1 | First-run preset never fetched | **Resolved** | rule 8 (`:57`) takes `pool_override` six dishes + `entry_fee_override = 40` from `首局引导.get_first_run_preset()`; AC 6 (`:556`) asserts both; `first-run-onboarding.md:33-38` defines the same values |
| 2 | `WALLET_OVERFLOW` abandons the record update | **Resolved** | edge case (`:384`) now updates records and labels only the wallet; `meta-upgrades.md:33,294` say the run still goes to records |
| 3 | Single-commit criterion impossible | **Partially resolved** | `meta-upgrades.md:37,122` now expose `prepare_run_result` and commit nothing; `save-system.md:378` AC 13 unchanged. `records-and-milestones.md:111` still publishes `apply_run`, not `prepare_run` — see Blocker 1 |
| 4 | Receipt dish name unsourced | **Resolved** | rule 11 (`:63`) resolves via `get_dish(dish_id).display_name`; `data-config.md:57` defines `get_dish`; Interactions (`:161`) and Dependencies (`:435`) declare it |
| 5 | `存档` 规则 80 does not exist | **Resolved** | rule 10 (`:61`) now cites 规则 18 / AC 13; no `规则 80` remains |
| 6 | Inventory omits `BlockedModal` / `ReadOnlyNotice` | **Resolved** | rule 2 (`:42-43`) lists both |
| 7 | Notice has no `back_target` precedence | **Partially resolved** | priority 0 `DISMISS_NOTICE` added (`:220`), but rule 7's list omits it and the position precedes modality — see Blocker 2 |
| 8 | Milestone catalogue ownerless | **Partially resolved** | Interactions (`:158`) now reads `get_milestone_catalog()`; rule 23 (`:87`) and Cross-References (`:530`) still say `数据配置` — see Blocker 3 |

### Completeness: 6/6 sections present, where 6 is the count required at `standard`

Required at `standard` and present: Overview, Detailed Rules (`## Detailed
Design`), Formulas (the system defines numeric rules — stack depth, modal depth,
a return-decision function, focus selection, latency, roll timing), Edge Cases,
Dependencies, Acceptance Criteria.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` still carries it.
- **Tuning Knobs** — absent.

### Dependency Graph

All twelve declared system dependencies have a GDD on disk.

### Registry Values

Registry names searched across the whole document (`net_profit`, `total_value`,
`broke_even`, `return_ratio`, `station_canteen_entry_fee`, `first_run_entry_fee`,
`first_run_pool_size`, `capacity_total`, `running_net`, `load_outcome`,
`wasted_capacity`, `cloy_extra_total`). Nine lines matched. **No value
conflicts**: this GDD restates none of these numbers — it reads them from
`RunResult` or delegates to the preset (`first_run_entry_fee = 40` is cited only
as a preset field, never as this system's own constant). The only registry
defects are back-reference gaps, listed under Recommended.

### Senior Verdict

Not applicable — `lean` mode runs no `creative-director` synthesis.

### Scope Signal

Rough scope signal: **L** (producer should verify before sprint planning).

### Verdict: NEEDS REVISION

The navigation core remains implementable and the fix wave cleared the three
findings that were purely this document's own (screen inventory, save-rule
citation, dish-name source). The three remaining blockers are each a fix applied
to one site out of several: the prepare/commit rename landed in `meta-upgrades`
and in rule 10 but not in `records` or in this document's Dependencies and ACs;
the notice priority landed in the formula table but not in rule 7 and out of
order against rule 5; the milestone owner landed in the Interactions table but
not in rule 23. None requires redesigning the screen model — each is a
one-pass consistency sweep.

Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 9fe5183795ca3d47798510a1d00a040dabe3ca61
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/save-system.md 772ed596d68eb5cbae207b7bac9a170ec55c1524
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
Reviewed-Content-Hash: design/gdd/systems-index.md 15e0bd3d307ed09738b74dd3c099816b497b69e8

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 13
Summary: All three prior blockers are resolved — `apply_run` is gone and the settlement API is uniform, the notice now sits after the modal rules in both rule 7 and `back_target`, and the milestone catalogue is owned by 纪录与里程碑 at every site this document controls. One new blocker surfaced, and it is the same shape as the last wave: a fix applied to this document while its hard dependency moved underneath it. `存档` UI Requirements 4 was revised to split `Blocked` into four causes with two distinct exit sets, one of which offers 「重试」 and forbids 「新建生涯」; this GDD still fixes `BlockedModal` at exactly two buttons and AC 26 asserts that. The navigation core itself is unchanged and still implementable.
Prior verdict resolved: Partial — 3/3 prior blockers resolved; a new blocker appeared in `存档`'s revised UI requirements.

### Prior blockers

| # | Item | Status |
|---|---|---|
| 1 | Settlement interface named two ways | **Resolved** — `apply_run` / `apply_run_result` no longer appear anywhere in the file; rule 10 (`:61`), rule 13 (`:67`), Interactions (`:157-158`), Dependencies (`:431-432`), Cross-References (`:526-527`), AC 11/18 (`:565,572`), fixtures (`:541`) and OQ1 (`:623`) all read `prepare_run` / `prepare_run_result`. `records-and-milestones.md:111` publishes `prepare_run(result)` and `meta-upgrades.md:122` publishes `prepare_run_result(result)`. See Recommended 5 for the residual OQ1 recording gap. |
| 2 | Notice priority precedes modality | **Resolved** — `back_target` (`:218-228`) now orders 1 modal-cancel → 2 forced-modal `NONE` → 3 `DISMISS_NOTICE` → 4 `SKIP_ROLL` → 5–8 screens; rule 7 (`:55`) carries the same order in prose (①–⑨), so the modal always captures `ui_cancel` first and rule 5 is no longer contradicted. See Recommended 1 for missing test coverage of the new priority. |
| 3 | Milestone catalogue owner split | **Resolved** — rule 23 (`:87`), Interactions (`:158`), Cross-References (`:531`) all read `纪录与里程碑.get_milestone_catalog()`; `records-and-milestones.md:298` publishes it. See Recommended 4 for two sites that still omit it from their interface lists. |

### Completeness: 6/6 sections present, where 6 is the count required at `standard`

Required at `standard` and present: Overview, Detailed Rules (`## Detailed
Design`), Formulas (the system defines numeric rules — stack depth, modal depth,
a return-decision function, focus selection, latency, roll timing), Edge Cases,
Dependencies, Acceptance Criteria.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:477`) carries it.
- **Tuning Knobs** — absent. The four tunables (`RECEIPT_ROLL_SECONDS`,
  `SAVE_INDICATOR_SECONDS`, `MAX_UI_STACK_DEPTH`, `MAX_MODAL_DEPTH`) are each
  defined inside the formula that uses them, so nothing is unspecified — the
  section is a findability gap, not a hole.

### Dependency Graph

All twelve declared system dependencies have a GDD on disk (`场景与流程管理`,
`输入`, `餐局流程`, `结算与入场费`, `局外升级`, `纪录与里程碑`, `存档`,
`音效反馈`, `数据配置`, `首局引导`, `解锁与档口`, `局内 HUD`). No broken
references.

### Registry Values

Registry names searched across the whole document: `net_profit`, `total_value`,
`broke_even`, `return_ratio`, `station_canteen_entry_fee`, `first_run_entry_fee`,
`first_run_pool_size`, `capacity_total`, `running_net`, `load_outcome`,
`wasted_capacity`, `cloy_extra_total`, `unit_value`, `effective_unit_value`, plus
the three back-reference names `station_available`, `unlock_decision`,
`unlock_progress`. Ten lines matched. **No value conflicts**: the GDD restates no
registry number. `entry_fee = 0` (`:63,378,569`) and `net_profit > 0`
(`:63,378-380,570`) are predicate conditions, not restatements of
`station_canteen_entry_fee = 85` or `first_run_entry_fee = 40`. The only registry
defects are back-reference errors, listed under Recommended 11.

### Required Before Implementation

1. **`Blocked` modal's exits contradict `存档` UI Requirements 4.** `[main review]`
   This document fixes `BlockedModal` as exactly two buttons —
   「退出」+「新建生涯（旧档保留）」 — at rule 2's inventory (`:43`), rule 20
   (`:81`), Edge Cases (`:402`), UI Requirements (`:504`, "两种原因之一") and
   AC 26 (`:586`), all citing `存档` UI Requirements 4. That requirement has since
   been rewritten (`save-system.md:320-325`) to split `Blocked` into **four**
   causes with **two different exit sets**:
   - 「存档属于更新版本」→ 退出 / 新建生涯
   - 「存档与备份都无法读取」→ 退出 / 新建生涯
   - 「迁移失败」→ 退出 / **重试**; 不提供新建生涯
   - 「另一实例正在使用这份存档且文件暂不可读」→ 退出 / **重试**; 不提供新建生涯

   `save-system.md` rule 21 (`:47`) states the same for the held-lock case:
   "该情形的出口只有「退出」与「重试」…不提供「新建生涯」". `Rule 2` makes the
   inventory fixed and forbids implementation-added controls, so an implementer
   has nowhere to put 「重试」: two of the four causes cannot be served, and
   AC 26 would fail against the requirement it cites. `Blocked` 的「重试」出口在
   全文档中不存在，其默认焦点也未定义（规则 20 只给「退出」定焦点）。
   → Split `BlockedModal` by cause: exit set as a function of the
   `LoadReport` reason code, add 「重试」 (re-run `load()`), default focus 「退出」
   for every cause, and state that 「新建生涯」 appears only for the two causes
   `存档` permits. Update `:43`, `:81`, `:402`, `:504`, AC 26 and OQ8 (`:630`,
   which still says "三种原因" with a third partition — 更新版本 / 无法读取 /
   另一实例 — and omits 迁移失败 entirely).
   **Note for the orchestrator:** `save-system.md:66` still says every `Blocked`
   outcome offers 新建生涯 or 退出, contradicting its own rule 21 and UI
   Requirements 4. That is a defect in `存档`, not in this document, and this
   GDD must be aligned to whichever of the two `存档` states is authoritative.

### Recommended Revisions

1. **AC 38 overclaims coverage of `back_target`.** `[main review]` It says the
   result must match "`back_target` 表逐项一致" but enumerates 7 scenarios for a
   9-row table. Missing: priority 3 `DISMISS_NOTICE` — the behaviour this fix wave
   added — and priority 4 `SKIP_ROLL` in the `back_target` context (AC 19 covers
   skip on the receipt, but not a notice-versus-roll ordering). Add
   `GIVEN` a dismissible notice at `MenuRoot`, `WHEN ui_cancel`, `THEN`
   `DISMISS_NOTICE` and no quit confirm opens; and the same with
   `roll_active = true` and `modal_depth > 0` to pin the ordering.
2. **No acceptance criterion for `LoadReport = ReadOnly`.** `[main review]` Rule
   20 (`:81`), Edge Cases (`:401`) and UI Requirements (`:503`) define a
   non-modal notice with exactly 「以只读继续」/「退出」, purchase entries hidden,
   and no 「新建生涯」 — but AC 25 covers only Fresh / Loaded / Migrated /
   Recovered and AC 26 only `Blocked`. AC 45's retained-evidence list
   (`:617`) has no `ReadOnly` screenshot either.
3. **`get_milestone_catalog()` is missing from two interface lists.** `[main review]`
   Dependencies (`:432`) and Cross-References (`:527`) enumerate 纪录与里程碑's
   surface without it, while Interactions (`:158`), the new Cross-References row
   (`:531`) and rule 23 (`:87`) all have it.
4. **`record_broken` timing is now stated — good.** Rule 11 (`:63`) pins it to
   `prepare_run` and stashes `previous_best` for the receipt; this closes the
   round-2 question. No action.
5. **`records` still reports the orchestrator as unnamed.** `[main review]` Rule
   10 (`:61`) and OQ1 (`:623`) say this document answers `纪录与里程碑` Open
   Question 1, and `meta-upgrades.md:37` now records 界面流程 as the orchestrator —
   but `records-and-milestones.md:120` still says "编排者身份尚未指定" and its OQ1
   (`:450`) is still open. Records-side edit; the dependency requires the answer be
   recorded on both sides.
6. **`get_first_run_preset()` is named only here.** `[main review]` Rule 8 (`:57`)
   and Interactions (`:163`) call `首局引导.get_first_run_preset()`;
   `first-run-onboarding.md:129` publishes the preset values but no method name.
   Either onboarding adopts the name or this document names the two fields it
   reads.
7. **`receipt_closed(reason)` has no counterpart in `音效反馈`.** `[main review]`
   `audio-feedback.md:115,366,373` describe "收据关闭通知 / 取消待播结果" without
   the event name this document owns (`:91,168`, AC 44).
8. **#13's dependency is declared hard on one side only.** `[main review]`
   `unlocks-and-stations.md:92,366` declare 界面流程 硬（UI）and `:386` requires
   `/ux-design` to cover a station-selection page with lock states, price, unlock
   confirmation, `COMMIT_FAILED` and progress; Dependencies (`:443`) and
   Interactions (`:164`) here still say 软 / "将来". Blocks Vertical Slice
   planning, not MVP.
9. **`SettingsModal`'s scope reads wider than its entry points.** `[main review]`
   "任意有界面的状态" (`:40`, `:111`, `:501`) includes `Error`, whose rule 18
   (`:77`) allows only 退出, and `Boot`, whose context is `Blocked` (`:75`).
   Restrict to `Menu` / `Paused`.
10. **The 「返回」 fallback control is not in two screens' 必须包含.** `[main review]`
    Rule 6 and AC 29/31 make `UpgradeScreen` / `RecordsScreen` fall back to
    「返回」 when nothing is purchasable / no record exists, but `:498` and `:499`
    do not list it.
11. **Registry back-references (orchestration edit).** `[main review]`
    `ui-flow.md` is missing from `referenced_by` of `net_profit`, `total_value`,
    `return_ratio`, `broke_even`, `station_canteen_entry_fee` and
    `first_run_entry_fee`, all of which this document reads from `RunResult` or
    the preset; it is falsely listed under `station_available`,
    `unlock_decision` and `unlock_progress`, terms it never mentions.
12. **Stale Cross-References note.** `[main review]` `:537`
    ("双向回指（待编排会话处理）… 需改为已写") contradicts `:452`
    ("双向一致（已完成，2026-11-03）") and the current sibling GDDs, which all
    already carry the 界面流程 row. Delete `:537`.
13. **Open Question 14 is answered; close it.** `[main review]` `:636` asks
    whether a same-value `push_context(Modal)` pushes or no-ops;
    `input.md:56` states `push_context` always increases stack depth and emits
    `context_changed(Modal, Modal)`, and `input.md:230` gives the two-modal AC.
    The question is resolved and only its row needs updating.

### Nice-to-Have

- `meta-upgrades.md:37` lists `prepare_run_result` before `prepare_run` while rule
  10 (`:61`) fixes records-first order — the short-circuit rule (records fatal →
  no wallet call) is only correct under rule 10's order. Align the wording so both
  read the same way.
- Rule 7's ⑤/⑥ (sub-screen then receipt) and `back_target`'s rows 5/6 (receipt
  then sub-screen) list the same two mutually exclusive cases in opposite order.
  Harmless, but a reader comparing the two tables will pause.

### Specialist Disagreements

None — `lean` mode runs no specialist agents.

### Senior Verdict

Not applicable — `lean` mode runs no `creative-director` synthesis.

### Verdict: NEEDS REVISION

The three prior blockers are genuinely closed: the rename sweep is complete
(no `apply_run` remains), the notice now sits behind the modal rules in both the
prose rule and the formula table, and the milestone catalogue has one owner
everywhere this document can reach. The new blocker is the same class of defect
the last wave fixed — a site that was correct against an older `存档` and is now
stale — and it is one screen contract, not a redesign: `BlockedModal` must take
its exit set from the `LoadReport` reason code instead of fixing two buttons. The
remaining thirteen items are coverage and naming gaps; none changes the screen
model.

Reviewed-Content-Hash: design/gdd/ui-flow.md 9c709252b4c631e8fb50d8a2f35a8fb394f230d7
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 84e5c38fc40fb2a75d8ee608c911bdebf369c5a1
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/save-system.md f22c2228d60be43669125831bccea834e94d6026
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 1cafff5e7ad8aab8912e716d98b69cea02cbbc43
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/audio-feedback.md ffb1c54a84af1da89b0228395ee86e89b3e87e30
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/systems-index.md 56e9869c70371816e88ed49f24735f67ab3d199d
