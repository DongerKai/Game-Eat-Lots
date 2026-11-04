# Settlement & Entry Fee (结算与入场费) — Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 3 | Recommended: 9
Summary: The arithmetic is sound — every worked example reproduces exactly (104 − 85 = +19, 54 − 85 = −31, 104/85 = 122.3%, F5 at exactly 0, the cloying row 10/3 = 3.33), the four runtime invariants are stated as errors rather than clamps, and the no-`opt_value` / no-bonus boundary is held throughout. The blocking problems are all contract gaps introduced by the recent migration to `RunLog.entry_fee`: the fee's provenance is still `StationData.entry_fee` in rule 2 and three formula blocks, the first-run edge case says no value changes even though the teaching fee is 40, and the live `running_net` the HUD must read from before the first dish has no run-start initialization at all. None of the three requires redesigning the calculation; each is a stated-input or stated-lifecycle fix.
Prior verdict resolved: First review

Findings:
- [BLOCKING] Detailed Design (rule 2 `:26`) / Formulas (`:137`, `:143`, `:170`, `:249-253`) / Dependencies (`:324`) / Cross-References (`:414`): the entry-fee source is only half-migrated. The `RunLog` table (`:64`) and the Interactions row (`:95`) now say the fee comes from `RunLog.entry_fee` (`RunConfig.entry_fee_override` or `StationData.entry_fee`), but rule 2, the `net_profit` / `return_ratio` expressions and variable sources, the `running_net` / `shortfall` variable tables, the upstream dependency row and the Cross-References table still read `StationData.entry_fee` / `entry_fee(station_id)`. A programmer implementing from the formulas bypasses the frozen log and re-reads the station.
- [BLOCKING] Edge Cases (`:315`) / Acceptance Criteria (`:466`): the first-run edge case says settlement "数值完全不变" and that onboarding only touches the unit-value column, but the first run uses `entry_fee_override = 40` (registry `first_run_entry_fee = 40`; `first-run-onboarding.md:38,87,458` requires net +42 and 205.0%). No acceptance criterion ever feeds a `RunLog.entry_fee` that differs from `StationData.entry_fee` — AC 25 mutates the station value instead — so a regression back to station reads would pass all 30 ACs.
- [BLOCKING] Detailed Design (rule 19 `:43`) / States (`:47`) / Formulas (`:245-265`) / UI Requirements (`:398-399`) / AC 12 (`:450`) / AC 27 (`:468`): the live HUD net is unimplementable as specified. The HUD must show `running_net = −entry_fee` before the first dish and update it every commit (`in-game-hud.md:33,117-118,471`), but this system "不持有餐局进行中的状态" (`:47`), stores only `last_settled_run_id`, receives its only input at run end via `settle(RunLog)` (`:100`), and `running_net(k)`'s variables name only `k` — no run-start call carries the frozen fee, and no accumulator owner, reset point or discard handling is defined.
- [RECOMMENDED] Interactions (`:99`): `end_reason` is listed under 时间与回合, but `time-and-turns.md:43` (rule 8) says that system neither saves nor orders `end_reason`, and the registry makes run-flow the owner (`entities.yaml:866-877`). The `RunLog` table (`:65`) already says 餐局流程; drop it from the time row.
- [RECOMMENDED] Formulas (`:180`, `:239`) / AC 3 (`:441`): `return_ratio`'s declared range `(0, +∞)` excludes the empty run, where `total_value = 0` gives `0` (F3, `:297`), and `effective_unit_value`'s `(0, price_max]` excludes a zero-priced dish even though the same formula block declares `price` starting at 0 (`:126`). The registry's `[0, 999]` (`entities.yaml:646`) already includes 0.
- [RECOMMENDED] Registry (`design/registry/entities.yaml`): several `output_range` bounds are narrower than the GDD's declared int64 domain — `total_value [0, 999999]` (`:603`), `net_profit [-99999, 999999]` (`:619`), `running_net` (`:675`), `shortfall` (`:690`), `return_ratio [0, 999]` (`:646`), `effective_unit_value [0, 9999]` (`:660`) — while the GDD allows `price`/`entry_fee` up to INT32_MAX. Registry edits belong to the orchestration session.
- [RECOMMENDED] Open Questions (`:487-492`): OQ2, OQ3 and OQ4 are already answered elsewhere — `run-flow.md:36` (no early cash-out), `records-and-milestones.md:31,309` (rank by net profit), and onboarding's `replacement_hint` (`entities.yaml:1128-1141`). Mark them answered so the remaining open set is honest.
- [RECOMMENDED] Cross-document (route to `/consistency-check`): `data-config.md:81,182` still assigns `entry_fee` reads to settlement; with the new contract run-flow resolves the override and writes `RunLog.entry_fee`, and settlement reads the log. `run-flow.md:453` (OQ10) tracks the same correction.
- [RECOMMENDED] Interactions (`:91-107`): hardness disagrees with the other side for two rows — 音效反馈 is 软 here but "硬（事件契约）" in `audio-feedback.md:107,351`; 首局引导 is 软 here but 硬（B4）in `first-run-onboarding.md:126`. 菜单波次 appears only in Dependencies (`:342`), and 场景与流程管理 is cited four times in Rules/Edge Cases (`:31`, `:308-310`) but has no row at all.
- [RECOMMENDED] Scope signal: `systems-index.md:206` records `S`, but this GDD has 6 hard upstream + 8 downstream systems and nine formula blocks. That is `L` by the review scale; the index edit belongs to the orchestration session.
- [RECOMMENDED] Tuning Knobs: absent (advisory at `standard`). The only runtime knob is `entry_fee` (data-driven, `:26`); a short section naming it plus the `[1.15, 1.35]` unupgraded band and the `[1.35, 1.45]` full-level band (`meta-upgrades.md:261`) would give a balance pass one place to look.
- [RECOMMENDED] Player Fantasy: absent as a heading (advisory at `standard`). `## Game Feel` (`:375-382`) covers the intended feeling — the receipt as a verifiable account, loss without shame — well enough that a one-line cross-reference would close the gap.

Advisory (not counted): Invariant 1 (`:273`) writes `capacity_used = RunLog.capacity_used`, which reads as a self-comparison; write `Σ effective_satiety = RunLog.capacity_used`. The Cross-References note "所有列出本系统的下游 GDD 已完成并已回指" is true for the table, but `scene-flow.md` — cited as the owner of the `Run → Menu` payload in `:31,417` — contains no reference to this system (0 hits); either add the indirect dependency row or note that scene-flow only carries the payload. F6 (`:435`) should be restated as `RunLog.entry_fee = 0` once the fee migration lands, so the fixture cannot be read as a station-data test.

---

## Design Review: 结算与入场费（Settlement & Entry Fee）
Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules, Edge Cases,
Dependencies, Acceptance Criteria — plus Formulas, because this system defines
numeric rules (sums, thresholds, ratios, checked accumulation). All six are
present; `## Detailed Design` is accepted as the Detailed Rules section.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:375-382`) covers
  the intended feeling (a receipt you can check line by line; loss stated, not
  judged).
- **Tuning Knobs** — absent; the one knob is the data-driven `entry_fee`.

### Dependency Graph

Every declared dependency has a GDD on disk.

Upstream:
- ✓ data-config.md — exists
- ✓ dish-values.md — exists
- ✓ cloying-and-taste.md — exists
- ✓ stomach-capacity.md — exists
- ✓ time-and-turns.md — exists
- ✓ run-flow.md — exists

Downstream:
- ✓ meta-upgrades.md — exists
- ✓ records-and-milestones.md — exists
- ✓ ui-flow.md — exists
- ✓ in-game-hud.md — exists
- ✓ first-run-onboarding.md — exists
- ✓ audio-feedback.md — exists
- ✓ save-system.md — exists
- ✓ menu-waves.md — exists

Cited but not in either table:
- ✓ scene-flow.md — exists; its Rule 6 (`scene-flow.md:32`) does carry the
  `RunResult` on `Run → Menu`, but the file never names this system (see
  Advisory).

### Registry Values

Searched the whole document for every registry term this system owns or
references (`total_value`, `net_profit`, `broke_even`, `return_ratio`,
`effective_unit_value`, `running_net`, `shortfall`, `capacity_used`,
`wasted_capacity`, `cloy_extra_total`, `unit_value`, `calibration_ratio`,
`extra_satiety`, `effective_satiety`, `run_id_alloc`, `end_reason`,
`capacity_total`, `record_is_better`, `replacement_hint`,
`first_run_entry_fee`), case-insensitive with `[_ ]` — 134 matching lines.
No numeric conflict:

- `station_canteen_entry_fee = 85` (`entities.yaml:1183-1193`) matches `:26`,
  `:143` and every worked example.
- `extra_satiety [0,3]`, `effective_satiety [1,9]` match `:222`, `:203`, `:275`.
- `capacity_used` and `wasted_capacity` match `:193-208` (the GDD's tighter
  `[0, capacity_total]` is the same quantity under the registry's 999 cap).
- `unit_value` "displayed at two decimals at settlement only" (`:363-377`)
  matches rule 10 and `:389`.
- `calibration_ratio` notes match `:187`, `:280-290`.
- `run_id_alloc` and `end_reason` semantics match `:62`, `:65`.
- `record_is_better` (net profit only) matches `:488` and is already the
  records GDD's answer (`records-and-milestones.md:31`).
- `replacement_hint` exists upstream (`entities.yaml:1128-1141`), answering
  OQ4.

Conflicts or gaps:
- **`first_run_entry_fee = 40`** (`entities.yaml:1276-1291`) has no path in
  this GDD — no mention of `entry_fee_override`, no teaching-price statement,
  and the first-run edge case says values do not change (Blocking 2).
- **`end_reason` owner** is run-flow (`entities.yaml:866-877`), against the
  Interactions row that assigns it to 时间与回合 (`:99`) (Recommended 1).
- **`return_ratio` range** is `[0, 999]` in the registry (`:646`), which
  includes 0; the GDD's `(0, +∞)` excludes the empty run (Recommended 2).
- **Narrower output ranges** in the registry for the int64 quantities
  (Recommended 3).

### Required Before Implementation

1. **Finish the `entry_fee` migration** — rule 2 (`:26`), the `net_profit`
   expression and variable source (`:137`, `:143`), the `return_ratio`
   expression (`:170`), the `running_net` / `shortfall` variable tables
   (`:249-253`), the upstream data-config row (`:324`) and the Cross-References
   row (`:414`) still name `StationData.entry_fee` / `entry_fee(station_id)`.
   The `RunLog` table (`:64`) and the Interactions row (`:95`) already have the
   right contract, and `run-flow.md:40,64,387` defines it: run-flow resolves
   `RunConfig.entry_fee_override`, falling back to `StationData.entry_fee`, and
   freezes it in `RunLog.entry_fee`. Rewrite rule 2 to say the run's fee is the
   frozen `RunLog.entry_fee`, and make every formula read `R.entry_fee` rather
   than the station record. Keep `StationData.entry_fee` only as the fallback
   source and as the value the menu displays before a run (`ui-flow.md:55`).
   Without this, the first run computes against 85 instead of 40, and a
   station-data hot reload can change the arithmetic of a run already in
   progress.

2. **State the first run's actual numbers and test the override path** — the
   `If 首局` edge case (`:315`) says "结算流程与数值完全不变" and that onboarding
   "不改变任何合计". That contradicts the 40-yuan teaching fee
   (`entities.yaml:1276-1291`; `first-run-onboarding.md:38,87,458`: net +42,
   205.0%, first upgrade affordable) and onboarding's B4, which explains the
   receipt's teaching price. Rewrite the edge case: the first run is a normal
   settlement over a `RunLog` whose `entry_fee = 40`, so `net_profit`,
   `broke_even`, `return_percent_x10` and the receipt's entry-fee line all
   differ; only the calculation rules are unchanged. Add a fixture + AC where
   `RunLog.entry_fee ≠ StationData.entry_fee` (e.g. F2 with 37 against a
   station at 85 → `net_profit = 17`), plus one asserting the HUD starts at
   the overridden fee. AC 25 (`:466`) currently mutates the station value, so
   nothing exercises the contract this revision introduced.

3. **Define the in-run interface for `running_net` / `shortfall`** — rule 19
   (`:43`) and the HUD requirements (`:398-399`) require a live reading from
   before the first dish, and AC 12 (`:450`) / AC 27 (`:468`) test it, but the
   States section (`:47`) says the system holds no in-run state, the only
   input is `settle(RunLog)` at run end (`:100`), and the formula's variables
   name only `k` (`:245-265`). Nothing carries the run's frozen fee into the
   system at `run_started`, nothing owns or resets the int64 accumulator, and
   a discarded run has no cleanup path — so the next run's HUD would start
   from the abandoned run's total. Fix by adding one run-start call (e.g.
   `begin_run(entry_fee)` from run-flow's `run_started`, or a subscription to
   the resolved `RunConfig`), defining who adds a committed row and when the
   accumulator resets, and adding the corresponding transition to the States
   table. Then add an AC that abandons a run and asserts the next run's HUD
   starts at `−entry_fee`. A pure `running_net(entry_fee, partial_rows)`
   alternative is possible but conflicts with `in-game-hud.md:170` ("不读
   `entry_fee` 自行计算"), so the run-start call is the cheaper fix.

### Recommended Revisions

4. **`end_reason` provenance** — remove it from the 时间与回合 row (`:99`);
   `time-and-turns.md:43` explicitly does not hold or order it, and run-flow
   owns it (`entities.yaml:866-877`). The `RunLog` table is already correct.
5. **Boundary ranges** — change `return_ratio` to `[0, +∞)` (`:180`) and state
   what the empty run shows (0.0%, not blank); change `effective_unit_value`
   to `[0, price_max]` (`:239`) to match `price ≥ 0` (`:126`).
6. **Registry output ranges** — widen `total_value` / `net_profit` /
   `running_net` / `shortfall` / `return_ratio` / `effective_unit_value` to the
   GDD's declared int64 domain, or state that the registry records the
   practical band and the GDD the defensive maximum. Registry edit belongs to
   the orchestration session.
7. **Close answered Open Questions** — OQ2 (`run-flow.md:36`), OQ3
   (`records-and-milestones.md:31,309`), OQ4 (`entities.yaml:1128-1141`).
8. **`data-config.md:81,182`** — the row still says settlement reads
   `entry_fee`; after this fix run-flow resolves the override and writes the
   log. Route to `/consistency-check` with `run-flow.md:453` (OQ10).
9. **Interactions table** — align 音效反馈 and 首局引导 to hard as their own
   GDDs declare (`audio-feedback.md:107,351`; `first-run-onboarding.md:126`);
   add rows for 菜单波次 and 场景与流程管理, or state explicitly that they
   reach this system only through run-flow.
10. **Scope signal** — `systems-index.md:206` says `S`; the dependency count
    and formula count put this at `L`. Orchestration-session edit.
11. **Tuning Knobs** — add the `entry_fee` knob and the two calibration bands.
12. **Player Fantasy** — add a short section or cross-reference Game Feel.

### Specialist Disagreements

None — Lean mode runs no specialists (`Phase 3b` skipped).

### Nice-to-Have

- Invariant 1 (`:273`) should read `Σ effective_satiety = RunLog.capacity_used`
  rather than `capacity_used = RunLog.capacity_used`.
- `scene-flow.md` is cited as the owner of the `Run → Menu` payload (`:31`,
  `:417`) but never names this system; an indirect dependency note would close
  the loop.
- Restate F6 (`:435`) as `RunLog.entry_fee = 0` after Blocking 1 so the fixture
  cannot be read as a station-data test.

### Scope Signal

Rough scope signal: **L** (6 hard upstream + 8 downstream systems, nine
formula blocks, no new ADR expected; producer should verify before sprint
planning). The index records `S`.

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 94da89a242967c44827406c4da5714d088df6394
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
