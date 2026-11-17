# Review Log — 数据配置（Data Config）

## Review — 2026-11-03 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 5 | Recommended: 11
Prior verdict resolved: First review

Summary: The schema, state machine and integrity philosophy are sound, and the
GDD is unusually precise about what it refuses to do (it does not chase
optimality). The blocking problems are all of one kind: several normative rules
stated in Detailed Design are absent from the `valid(D)` formula and from the
acceptance criteria, so a programmer implementing the formula literally would
ship a validator that lets `wave_size = 0` (division by zero in menu-waves) and
`turn_budget = 0` through. Two record types that downstream GDDs treat as hard
dependencies — `BalanceData` and `MilestoneData` — are still outside the
three-type schema.

### Required Before Implementation

1. **[BLOCKING] Formulas / Detailed Rules — `valid(D)` omits three normative field constraints.**
   Core Rule 4 requires `cost_growth ≥ 1.0` (line 41), Core Rule 5 requires
   `turn_budget ≥ 1` (line 52) and `wave_size ≥ 1` (line 53), but `R_upg`
   (line 95) and `R_sta` (line 96) constrain none of them, and no acceptance
   criterion tests them. Downstream consequence: `wave_count = ceil(|M| /
   wave_size)` (menu-waves) divides by zero if `wave_size = 0` passes
   validation; `time-and-turns` states explicitly that "`turn_budget` … 由数据
   配置在加载期校验" — a claim the formula does not back. Fix: add
   `u.cost_growth ≥ 1.0` to `R_upg`; add `s.turn_budget ≥ 1 ∧ s.wave_size ≥ 1`
   to `R_sta`; add one AC per constraint (the AC 23 block already has the
   shape for `cloy_cap`).

2. **[BLOCKING] Formulas — the `dishes_per_station` invariant is asserted but not enforced.**
   Line 123 claims `dishes_per_station(s)` "必须等于 `|s.dish_pool|`，否则
   `R_sta` 失败", but `R_sta` (line 96) never checks pool uniqueness. A station
   with `dish_pool = [dish_a, dish_a]` passes every clause while
   `dishes_per_station(s) = 1 ≠ 2`, and `dish_a` then appears twice in the wave
   shuffle. Fix: add `¬hasDuplicates(s.dish_pool)` (or the equality itself) to
   `R_sta`, plus an AC.

3. **[BLOCKING] Detailed Rules / Formulas — the id type-namespace contract (Core Rule 6) is unenforced.**
   Line 54 makes per-type prefixes (`dish_*` / `upgrade_*` / `station_*`) a
   contract, but the only id rule in `valid(D)` is the regex
   `^[a-z][a-z0-9_]*$`, which no prefix satisfies and no prefix violates.
   Because uniqueness is checked per type, a `DishData` with id
   `station_canteen` validates and then collides with the station id in save
   data and in every `get_*(id)` lookup. AC 3 tests the regex only. Fix: add a
   prefix predicate to each `R_*` and an AC line.

4. **[BLOCKING] Dependencies / schema — `BalanceData.base_capacity` is a hard dependency with no record type.**
   Line 176 lists 胃容量 → `BalanceData.base_capacity` (C0 = 20) as 硬, but
   Core Rule 2 says "三类记录，三类 Resource" and no `BalanceData` schema,
   `valid(D)` term, edge case or AC exists anywhere in this document.
   `stomach-capacity.md` Open Question 1 confirms the gap and assigns the
   decision to "数据配置复审时" — this review. Fix: either add the fourth
   record type in full (fields, range, `R_bal`, AC), or state in Detailed Rules
   that C0 is out of scope here and name its owner.

5. **[BLOCKING] Dependencies / schema — `MilestoneData` is listed as a hard dependency while the schema declares three types.**
   Line 177 calls it "第四类记录，归属见纪录 Open Question 2", but it is absent
   from the schema, the formula and the ACs. `records-and-milestones.md` Open
   Question 2 says both documents must be revised together and only one of the
   two paths can be taken. Until that resolves, the dependency row must read as
   conditional ("待 Open Question 2 决定"), not 硬 — otherwise the document
   simultaneously claims a closed three-type contract and a fourth one.

### Recommended Revisions

1. **[RECOMMENDED] Acceptance Criteria — duplicate number 23.**
   Lines 230 and 231 are both numbered 23, so "Acceptance Criteria 第 23 条"
   (line 194, Game Feel) and "转为第 23 条的行为验证" (line 235) each resolve to
   two different criteria. Renumber 24 → 25 as well.
2. **[RECOMMENDED] Formulas — undefined symbols.**
   `D_ids`, `S_ids`, `U_ids`, `uniqueIds(·)`, `cyclic(·)`, `dish(·)` and
   `upgrade_count` (used in the `prereq_depth` range) appear in the formula but
   not in the variables table. `dish(·)` and `cyclic(·)` carry real meaning —
   define them.
3. **[RECOMMENDED] Formulas — `prereq_depth` range contradicts its own semantics.**
   The range is `1–upgrade_count`, but AC 20 gives depth 2 for a three-node
   chain (edge counting), which makes a prerequisite-free upgrade depth 0.
   Range should be `0–(upgrade_count − 1)`.
4. **[RECOMMENDED] Formulas — `R_upg` does not validate `effect_type` membership.**
   The implication `(u.effect_type ∈ {STOMACH_CAPACITY, MENU_VISION} → …)` is
   vacuously true for an out-of-enum value, and Godot stores enums as plain
   ints in a `.tres`, so `effect_type = 7` validates. Add an explicit
   membership clause and one AC.
5. **[RECOMMENDED] Detailed Rules — Core Rule 7's extra check is phrased as a suggestion.**
   "建议增加一条：至少存在一个 `unlock_cost = 0` 的档口" sits inside a
   normative list and is absent from `valid(D)`. Decide: a set-level predicate
   plus AC, or drop it and keep `NoPlayableStation` as the single guarantee.
6. **[RECOMMENDED] Formulas — the worked example contradicts the registry on a registered name.**
   Line 116 gives `station_canteen` `entry_fee=30`; `entities.yaml` says 85
   (`station_canteen_entry_fee`, owned by settlement-and-entry-fee), and
   `dish-values.md` Open Question 2 already records the 30 as "只是示例". The
   example also invents `upgrade_capacity` (`cost=40`, `max_level=3`) where the
   real catalogue has `upgrade_stomach_capacity` (`cost=12`, `max_level=5`).
   Either use the real numbers or label the block as synthetic fixture data, so
   it cannot be copied into a `.tres`.
7. **[RECOMMENDED] Acceptance Criteria — AC 22 is not independently executable.**
   "p95 ≤ 100 ms（最小规格 PC）" — no minimum-spec machine is defined anywhere
   in the project. Name the reference machine (or cite where it is defined), or
   restate as a relative budget on the same harness.
8. **[RECOMMENDED] Acceptance Criteria — AC 24 cannot run before the first release.**
   Its baseline id list does not exist yet. Mark it as release-gated so QA does
   not report it as a standing failure.
9. **[RECOMMENDED] Acceptance Criteria — AC 19 accepts two different behaviours.**
   "操作被拒绝或抛错" passes either way. Pick one, and state the mechanism —
   `DishData` is a Godot Resource with exported fields, which is mutable by
   default; a setter that pushes an error and ignores the write needs to be
   named.
10. **[RECOMMENDED] Cross-system — state the structural-vs-balance split explicitly.**
    This GDD validates structure only, while consumers carry narrower bands
    (`wave_size` 1–8 in menu-waves, `taste_tags` 0–3 in cloying-and-taste,
    `menu_vision ≤ wave_size − 1`). One sentence — "bands live in the consuming
    GDD; this schema enforces structure" — closes the apparent conflict for
    every future reader.
11. **[RECOMMENDED] Detailed Rules — `CLOYING_RESISTANCE`'s `effect_value` is ambiguous.**
    The field is tunable but its semantics read as a constant ("每级减免 1
    点"), and "小数部分在解析时取整" does not say whether 2.5 becomes 2 or 3.
    Either make it an integer like the other two types, or state the rounding
    rule and that the value, not the constant 1, is what applies.
12. **[ADVISORY at `standard`] Tuning Knobs section is absent.**
    Not blocking at this tier, but the document already carries three knobs
    (`cost_growth` 1.35 / 1.2–1.6, `cloy_cap` 3, `wave_size` 4) scattered in
    Detailed Rules. A Tuning Knobs table would list each with its safe range
    and the gameplay aspect it moves, per `.claude/rules/design-docs.md`.
13. **[ADVISORY at `standard`] Player Fantasy section is absent.**
    Defensible for an invisible system — the doc says the player never touches
    it — but if the tier rises to `full` it becomes required; the stated pillar
    ("每口都是算术题") already supplies the one-line fantasy.

### Nice-to-Have

- Line 181's 场景与流程管理 row says only "GDD 已写" with no path, unlike every
  other row (`design/gdd/scene-flow.md`).
- Registry bookkeeping: `wave_size_default` (`entities.yaml`) does not list
  `design/gdd/data-config.md` in `referenced_by`, although this GDD defines the
  field that carries it. `cloy_cap` is correct.
- `display_name` / `description` have no non-empty constraint; an empty string
  passes validation and renders as a blank HUD row.
- The example writes `effect_value=1` (an int literal) for a `float` field.

### Completeness

6/6 sections present that are REQUIRED at `workflow: standard` (Overview,
Detailed Rules, Formulas, Edge Cases, Dependencies, Acceptance Criteria).
Formulas is required here because the system defines numeric rules. Advisory
gaps at this tier: Player Fantasy, Tuning Knobs.

### Dependency Graph

All 13 declared downstream dependencies resolve to an existing GDD.

- ✓ `design/gdd/dish-values.md`
- ✓ `design/gdd/cloying-and-taste.md`
- ✓ `design/gdd/menu-waves.md`
- ✓ `design/gdd/meta-upgrades.md`
- ✓ `design/gdd/run-flow.md` and `design/gdd/settlement-and-entry-fee.md` (one row)
- ✓ `design/gdd/save-system.md`
- ✓ `design/gdd/time-and-turns.md`
- ✓ `design/gdd/stomach-capacity.md`
- ✓ `design/gdd/records-and-milestones.md`
- ✓ `design/gdd/unlocks-and-stations.md`
- ✓ `design/gdd/ui-flow.md`
- ✓ `design/gdd/in-game-hud.md`
- ✓ `design/gdd/scene-flow.md` — row cites no path (nice-to-have above)
- Upstream: none declared, correct for a Foundation system.

Bidirectional check: every GDD in the table lists 数据配置 as an upstream
dependency. Two GDDs that mention 数据配置 but are correctly omitted from this
table — `input.md` ("输入不依赖数据配置") and `audio-feedback.md` ("无直接调用")
— agree that there is no dependency. `first-run-onboarding.md` resolves through
菜品数值 / 菜单波次 and states it does not query the catalog directly. Index
drift noticed but out of scope: `systems-index.md` row 11 (结算与入场费) omits
数据配置 from `Depends On` although `settlement-and-entry-fee.md` lists it as 硬.

### Registry Values

Searched the whole document for every registered name it could restate —
`station_canteen`, `dish_braised_pork`, `cloy_cap`, `wave_size`, `cost_growth`,
`base_capacity`, `entry_fee`, `valid_record_set` — 17 lines matched.

- ✓ `cloy_cap` = 3 (lines 51, 110) — matches `entities.yaml`.
- ✓ `wave_size` default = 4 (line 53) — matches `wave_size_default`.
- ✓ `cost_growth` = 1.35, band 1.2–1.6, floor ≥ 1.0 (line 41) — matches
  `meta-upgrades.md` exactly.
- ✓ `base_capacity` C0 = 20 (line 176) — matches `base_capacity`.
- ✗ Line 116: `station_canteen.entry_fee = 30` vs registry 85
  (`station_canteen_entry_fee`, owner settlement-and-entry-fee).
- `dish_braised_pork` is named in the example pool without values — no conflict
  with its registry entry (price 16, satiety 3, group 肉, tags [甜, 油]).
- `dish_fried_rice` and `upgrade_capacity` are not registered names; they are
  invented fixtures (see Recommended 6).

### Specialist Disagreements

None — lean mode, no specialist agents spawned.

### Verdict Rationale

The design is structurally sound: a single typed-resource catalogue, a fail-closed
startup gate, and an explicit refusal to police balance are the right shape for a
Foundation system, and the Edge Cases section is the strongest part of the
document. The verdict is NEEDS REVISION rather than APPROVED because the formula
and the acceptance criteria are not yet a complete encoding of the rules the
document itself states — five normative constraints (three of them
division-by-zero / zero-turn hazards) are missing from `valid(D)`, one invariant
is asserted without being enforced, and two record types are declared hard
dependencies while the schema says there are three. None of these require a
redesign; all are additive edits to the formula and the AC list.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/data-config.md 1e8e07619c6eb3c027dcb348bcf848c73f2b16b5
Reviewed-Content-Hash: design/registry/entities.yaml cdef3f7a3f4a270848207a53e3eef821342c6294
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/time-and-turns.md 7f55a619613d6c2ff9f99b9458e39d0f5f48895c
Reviewed-Content-Hash: design/gdd/stomach-capacity.md bb97f41a121c6c18d15f7d10c23387ff25688b11
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 9e1293c7b33381aadbc4a578797b14a608a0a9c0
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 0f58ea62a2e8fa9ace2effe652cc644acab89d5b
Reviewed-Content-Hash: design/gdd/scene-flow.md ac1d04a52a78a3027e95b4c81cde898e1b5ead06
Reviewed-Content-Hash: design/gdd/input.md c96715b12512d1a093a10f6cf2e32030740be509
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 778f0a836330a6410aee8812fa0801f9dd3cc4f3
Reviewed-Content-Hash: design/gdd/systems-index.md 721a697860f5cbf632fa991257dc4145828b1955
```

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 2 | Recommended: 15
Prior verdict resolved: No — 5 prior blockers resolved, 2 new blockers introduced by the fixes

Summary: All five prior blocking items are addressed: `R_upg`/`R_sta` now carry the
`cost_growth`/`turn_budget`/`wave_size` floors, `R_sta` enforces pool uniqueness,
all three `R_*` carry `hasPrefix`, `BalanceData` is a documented fourth record type,
and `MilestoneData` is marked conditional. The two remaining blockers are gaps the
fix itself opened: the singleton `BalanceData` has no cardinality constraint, so an
empty `D_bal` validates vacuously and boots a game with no `C0`; and the new id-prefix
rule was never propagated into the existing AC fixtures, leaving AC 3's positive case
unreachable. Both are small additive edits.

### Required Before Implementation

1. **[BLOCKING] Formulas / Detailed Rules — `D_bal` cardinality is unconstrained.**
   Rule 10 (line 58) calls `BalanceData` 单例, and the registry calls it "the
   singleton BalanceData" (`entities.yaml:331`), but `valid(D)` (line 92) only
   quantifies `∀b ∈ D_bal: R_bal(b)`. With no `BalanceData` file, that clause is
   vacuously true: the game reaches `Ready`, `get_balance()` returns null, and
   `stomach-capacity` has no `C0` at first run. With two files, which one
   `get_balance()` returns is unspecified. Fix: add `|D_bal| = 1` to `valid(D)`,
   an Edge Case for missing/duplicate `BalanceData`, and ACs for both; add a
   `BalanceData` record to AC 1's happy-path fixture, which currently holds only
   three types (line 216).

2. **[BLOCKING] Acceptance Criteria — fixture ids contradict the new prefix rule.**
   `R_dish` now requires `hasPrefix(d.id, "dish_")` (line 96), but AC 2 uses dish
   id `rice` (line 217), AC 3 asserts `a_b1` → Ready (line 218), and AC 19 calls
   `get_dish("rice")` on an already-Ready state (line 234). `a_b1` and `rice`
   violate the prefix clause, so AC 3's positive case can never pass, AC 2's
   duplicate-id log assertion is masked by the prefix error, and AC 19's
   precondition cannot be constructed. Fix: use full ids in the fixtures
   (`dish_rice`, `dish_a_b1`; for the regex case, `dish_Rice_01`), or state once
   that AC fixtures use fully-qualified ids.

### Recommended Revisions

1. **[RECOMMENDED] Acceptance Criteria — duplicate number 23 (carried over unfixed).**
   Lines 238 and 239 are both numbered 23, so "Acceptance Criteria 第 23 条"
   (line 202, Game Feel) and "转为第 23 条的行为验证" (line 248) each resolve to
   two criteria. Renumber from 24 onward (29 items in total now).
2. **[RECOMMENDED] Formulas — undefined symbols (partially fixed).**
   `hasPrefix` and `hasDuplicates` were added to the variables table, but
   `D_ids`, `S_ids`, `U_ids`, `uniqueIds(·)`, `cyclic(·)`, `dish(·)` and
   `upgrade_count` still appear in the formulas without a definition.
3. **[RECOMMENDED] Formulas — `prereq_depth` range still contradicts its semantics.**
   Line 132 keeps `1–upgrade_count`, but AC 20 gives depth 2 for a three-node
   chain and the edge-counting definition makes a prerequisite-free upgrade 0.
   Range should be `0–(upgrade_count − 1)`.
4. **[RECOMMENDED] Formulas — `R_upg` still does not validate `effect_type` membership.**
   The implication over `{STOMACH_CAPACITY, MENU_VISION}` is vacuously true for an
   out-of-enum value, and Godot stores enums as plain ints in a `.tres`, so
   `effect_type = 7` validates. Add an explicit membership clause and one AC.
5. **[RECOMMENDED] Detailed Rules — Core Rule 7's extra check is still a suggestion.**
   "建议增加一条：至少存在一个 `unlock_cost = 0` 的档口" (line 55) sits inside a
   normative list and is absent from `valid(D)`. Decide: a set-level predicate
   plus AC, or drop it and keep `NoPlayableStation` as the single guarantee.
6. **[RECOMMENDED] Formulas — the worked example still contradicts the registry.**
   Line 124 gives `station_canteen.entry_fee=30`; the registry says 85
   (`station_canteen_entry_fee`). It also invents `upgrade_capacity` (cost 40 /
   max 3) where the catalogue has `upgrade_stomach_capacity` (cost 12 / max 5,
   `meta-upgrades.md:167`). Use the real numbers or label the block as synthetic
   fixture data so it cannot be copied into a `.tres`.
7. **[RECOMMENDED] Acceptance Criteria — AC 22 is not independently executable.**
   "p95 ≤ 100 ms（最小规格 PC）" (line 237) — no minimum-spec machine is defined
   anywhere in the project. Name the reference machine or restate as a relative
   budget on the same harness.
8. **[RECOMMENDED] Acceptance Criteria — AC 24 cannot run before the first release.**
   Its baseline id list does not exist yet (line 240). Mark it release-gated so
   QA does not report it as a standing failure.
9. **[RECOMMENDED] Acceptance Criteria — AC 19 accepts two different behaviours.**
   "操作被拒绝或抛错" (line 234) passes either way. Pick one and name the
   mechanism — `DishData` is a mutable exported Resource by default.
10. **[RECOMMENDED] Cross-system — state the structural-vs-balance split explicitly.**
    This GDD validates structure only, while consumers carry narrower bands
    (`wave_size` 1–8 in menu-waves, `turn_budget` 10 ± 1, `cloy_cap` consumer
    rules). The variables table also mixes the two: `cost_growth` annotates
    "（设计带 1.2–1.6）" but `wave_size` says only "1–8" (line 116) while
    `R_sta` enforces ≥ 1. One sentence plus matching annotations closes the
    apparent conflict.
11. **[RECOMMENDED] Detailed Rules — `CLOYING_RESISTANCE`'s `effect_value` is still ambiguous.**
    Line 44 reads as both a constant ("每级减免 1 点") and a tunable, and
    "小数部分在解析时取整" does not say whether 2.5 becomes 2 or 3. Either make
    it an integer like the other two types, or state the rounding rule and that
    the value, not the constant 1, applies.
12. **[RECOMMENDED] Formulas — the id regex applies only to `DishData`.**
    The format `^[a-z][a-z0-9_]*$` is stated under Rule 3 only; `R_upg` and
    `R_sta` check `hasPrefix` alone, so `upgrade_BAD` validates. Make the regex a
    shared predicate in all three `R_*`.
13. **[ADVISORY at `standard`] Tuning Knobs section is absent.**
    `cost_growth`, `cloy_cap`, `wave_size`, `turn_budget`, `base_capacity` and
    `effect_value` are all tunables scattered in Detailed Rules; a table with safe
    ranges and the aspect each moves would satisfy `.claude/rules/design-docs.md`
    if the tier rises.
14. **[ADVISORY at `standard`] Player Fantasy section is absent.**
    Defensible for an invisible system; required if `modes.workflow` rises to
    `full`.
15. **[ADVISORY at `standard`] `MilestoneData`'s conditional status is not mirrored in `records-and-milestones.md`.**
    This GDD now correctly marks the dependency 条件 (line 185), but
    `records-and-milestones.md:291/313` still lists 数据配置 as 硬 and still calls
    `MilestoneData` the "第四类记录" — now stale, since this schema has four types
    and it would be the fifth. Cross-document drift owned by that GDD.

### Nice-to-Have

- Line 189's 场景与流程管理 row still says only "GDD 已写" with no path, unlike
  every other row (`design/gdd/scene-flow.md`).
- Registry bookkeeping: `base_capacity` (`entities.yaml:1263`) still notes "Data
  ownership is still a stomach Open Question" and omits `design/gdd/data-config.md`
  from `referenced_by`, although this GDD now owns the field; `wave_size_default`
  likewise omits it. `cloy_cap` is correct.
- `stomach-capacity.md` Open Question 1 is still listed as open although this GDD
  answers it (line 59); mark it answered.
- `display_name` / `description` still have no non-empty constraint; an empty
  string validates and renders as a blank HUD row.
- `hasPrefix(id, "dish_")` accepts a bare `dish_` — the regex allows a trailing
  underscore, so an id with no suffix validates.
- The example writes `effect_value=1` (an int literal) for a `float` field.
- `meta-upgrades.md:96` consumes `get_station(id) → wave_size`, but this GDD's
  局外升级 row (line 180) lists only `get_upgrade`; the interface list is slightly
  incomplete.

### Completeness

6/6 sections present that are REQUIRED at `workflow: standard` (Overview,
Detailed Rules, Formulas, Edge Cases, Dependencies, Acceptance Criteria).
Formulas is required here because the system defines numeric rules. Advisory
gaps at this tier: Player Fantasy, Tuning Knobs.

### Dependency Graph

All 13 declared downstream dependencies resolve to an existing GDD.

- ✓ `design/gdd/dish-values.md`
- ✓ `design/gdd/cloying-and-taste.md`
- ✓ `design/gdd/menu-waves.md`
- ✓ `design/gdd/meta-upgrades.md`
- ✓ `design/gdd/run-flow.md` and `design/gdd/settlement-and-entry-fee.md` (one row)
- ✓ `design/gdd/save-system.md`
- ✓ `design/gdd/time-and-turns.md`
- ✓ `design/gdd/stomach-capacity.md`
- ✓ `design/gdd/records-and-milestones.md`
- ✓ `design/gdd/unlocks-and-stations.md`
- ✓ `design/gdd/ui-flow.md`
- ✓ `design/gdd/in-game-hud.md`
- ✓ `design/gdd/scene-flow.md` — row cites no path (nice-to-have above)
- Upstream: none declared, correct for a Foundation system.

Bidirectional check: every GDD in the table lists 数据配置 as an upstream
dependency. The `MilestoneData` edge is the one asymmetry: this GDD says 条件,
`records-and-milestones.md` still says 硬 (Recommended 15).

### Registry Values

Searched the whole document for every registered name it could restate —
`station_canteen`, `dish_braised_pork`, `dish_fried_rice`, `cloy_cap`,
`wave_size`, `cost_growth`, `base_capacity`, `entry_fee`, `valid_record_set`,
`upgrade_capacity` — 23 lines matched.

- ✓ `cloy_cap` = 3 (lines 51, 113) — matches `cloy_cap`.
- ✓ `wave_size` default = 4 (line 53) — matches `wave_size_default`.
- ✓ `cost_growth` = 1.35, band 1.2–1.6, floor ≥ 1.0 (lines 41, 114) — matches
  `meta-upgrades.md` exactly.
- ✓ `base_capacity` = 20, range 1–999 (lines 59, 117, 245) — matches the registry
  value; the registry's own notes are stale (nice-to-have).
- ✓ `valid_record_set` formula (line 92) matches the registry expression and now
  includes `D_bal`.
- ✗ Line 124: `station_canteen.entry_fee = 30` vs registry 85.
- `upgrade_capacity` is not a registered name (Recommended 6).

### Specialist Disagreements

None — lean mode, no specialist agents spawned.

### Verdict Rationale

The prior blockers were fixed correctly and the schema is now a complete,
self-consistent four-type contract. The verdict stays NEEDS REVISION for two
additive reasons: the new fourth type has no cardinality clause, so the honest
reading of `valid(D)` still permits a boot with no `C0` — the same
"rule stated, formula silent" shape as the blockers just fixed; and the new
prefix predicate contradicts three existing acceptance criteria, one of which
asserts an outcome the document now forbids. Both fixes are a handful of lines,
and the remaining recommendations are unchanged from the first review.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/data-config.md 001e4ec43326ff7ab46c54c4f626664bd9fc7e23
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/time-and-turns.md 7f55a619613d6c2ff9f99b9458e39d0f5f48895c
Reviewed-Content-Hash: design/gdd/stomach-capacity.md bb97f41a121c6c18d15f7d10c23387ff25688b11
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/scene-flow.md c19ee4c2a48183476032f3ebbe2ceb9e43588324
Reviewed-Content-Hash: design/gdd/input.md 776c41de0eb3edd497ae2ace8c78ba80192fc78f
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/systems-index.md 63d73495be76b5b37ed9bac50452c07ef281d41a
```

## Review — 2026-11-04 — Verdict: APPROVED

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 0 | Recommended: 12 (+2 advisory at this tier)
Prior verdict resolved: NEEDS REVISION (2026-11-04) — both blockers fixed

Summary: Round 3 confirms both round-2 blockers were fixed exactly as
prescribed. `valid(D)` now carries `|D_bal| = 1`, the missing/duplicate
`BalanceData` case has an Edge Case and AC 30, AC 1's happy path includes the
singleton, and the three fixture ids that contradicted the prefix rule are now
fully qualified. No new blocking issue was introduced. The remaining findings
are the documentation-level recommendations carried since round 1; none of them
prevents a programmer from implementing the validator or a QA tester from
executing the criteria, so the design is approved for implementation with those
items tracked as follow-up polish.

### Prior Blocking Items — Resolved

1. **`D_bal` cardinality** — `valid(D)` (line 92) now ends `∧ |D_bal| = 1`;
   the Edge Case (line 149) covers zero and multiple; AC 30 (line 247) tests
   zero / two / exactly one; AC 1 (line 217) includes the `BalanceData`
   singleton. ✓
2. **Prefix-inconsistent fixtures** — AC 2 uses `dish_rice` (line 218), AC 3
   uses `dish_Rice_01` / `dish-1abc` / positive `dish_a_b1` (line 219), AC 19
   uses `get_dish("dish_rice")` (line 235). Every one satisfies
   `hasPrefix(…, "dish_")`, and each violates or satisfies the regex exactly as
   the criterion intends. ✓

### Required Before Implementation

None. Every normative constraint in Detailed Rules is represented in `valid(D)`
and has at least one acceptance criterion, and no edge case contradicts a core
rule.

### Recommended Revisions

1. **[RECOMMENDED] Acceptance Criteria — duplicate number 23 (carried over, third round).**
   Lines 239 and 240 are both numbered 23. "Acceptance Criteria 第 23 条"
   (line 203) and "转为第 23 条的行为验证" (line 250) therefore each resolve to
   two criteria. There are 31 entries under 30 labels; renumber from 24 onward.
2. **[RECOMMENDED] Formulas — undefined symbols.**
   `hasPrefix` and `hasDuplicates` are defined (line 118), but `D_ids`, `S_ids`,
   `U_ids`, `uniqueIds(·)`, `cyclic(·)`, `dish(·)` and `upgrade_count`
   (lines 96–98, 130–132) still appear without a definition.
3. **[RECOMMENDED] Formulas — `prereq_depth` range contradicts its semantics.**
   Line 132 keeps `1–upgrade_count`, but AC 20 gives depth 2 for a three-node
   chain and a prerequisite-free upgrade has depth 0. Range should be
   `0–(upgrade_count − 1)`.
4. **[RECOMMENDED] Formulas — `R_upg` still does not validate `effect_type` membership.**
   The implication over `{STOMACH_CAPACITY, MENU_VISION}` is vacuously true for
   an out-of-enum value, and Godot stores enums as plain ints in a `.tres`, so
   `effect_type = 7` validates. Add an explicit membership clause and one AC.
5. **[RECOMMENDED] Detailed Rules — Core Rule 7's extra check is still a suggestion.**
   "建议增加一条：至少存在一个 `unlock_cost = 0` 的档口" (line 55) sits inside a
   normative list and is absent from `valid(D)`. Relatedly, `dish_count ≥ 1`
   (line 130) is claimed to be implied by `R_sta`, but a set with one
   `BalanceData` and zero dishes/stations passes `valid(D)` and yields
   `dish_count = 0`. Decide: a set-level predicate plus AC, or keep
   `NoPlayableStation` as the single guarantee and soften the range claim.
6. **[RECOMMENDED] Formulas — the worked example still contradicts the registry.**
   Line 124 gives `station_canteen.entry_fee=30`; the registry says 85
   (`station_canteen_entry_fee`), and `dish-values.md` Open Question 2 already
   records that 30 is illustrative only. It also invents `upgrade_capacity`
   (cost 40 / max 3) where the catalogue has `upgrade_stomach_capacity`
   (cost 12 / max 5, `meta-upgrades.md:277`). Use the real numbers or label the
   block as synthetic fixture data so it cannot be copied into a `.tres`.
7. **[RECOMMENDED] Acceptance Criteria — AC 22 is not independently executable.**
   "p95 ≤ 100 ms（最小规格 PC）" (line 238) — no minimum-spec machine is defined
   anywhere in the project. Name the reference machine or restate as a relative
   budget on the same harness.
8. **[RECOMMENDED] Acceptance Criteria — AC 24 cannot run before the first release.**
   Its baseline id list does not exist yet (line 241). The document already
   concedes this; mark it release-gated so QA does not report it as a standing
   failure.
9. **[RECOMMENDED] Acceptance Criteria — AC 19 accepts two different behaviours.**
   "操作被拒绝或抛错" (line 235) passes either way. Pick one and name the
   mechanism — a `DishData` is a mutable exported Resource by default.
10. **[RECOMMENDED] Cross-system — state the structural-vs-balance split explicitly.**
    This GDD validates structure only, while consumers carry narrower bands
    (`wave_size` 1–8 in menu-waves, `turn_budget` 10 ± 1, `cloy_cap` consumer
    rules). The variables table mixes the two: `cost_growth` annotates
    "（设计带 1.2–1.6）" (line 114) while `wave_size` says "1–8" (line 116) but
    `R_sta` enforces only ≥ 1. One sentence plus matching annotations closes
    the apparent conflict.
11. **[RECOMMENDED] Detailed Rules — `CLOYING_RESISTANCE`'s `effect_value` is still ambiguous.**
    Line 44 reads as both a constant ("每级减免 1 点") and a tunable, and
    "小数部分在解析时取整" does not say whether 2.5 becomes 2 or 3. Either make
    it an integer like the other two types, or state the rounding rule and that
    the value, not the constant 1, applies.
12. **[RECOMMENDED] Formulas — the id regex applies only to `DishData`.**
    `^[a-z][a-z0-9_]*$` is stated under Rule 3 only; `R_upg` and `R_sta` check
    `hasPrefix` alone, so `upgrade_BAD` validates. Make the regex a shared
    predicate in all three `R_*`.
13. **[ADVISORY at `standard`] Tuning Knobs section is absent.**
    `cost_growth`, `cloy_cap`, `wave_size`, `turn_budget`, `base_capacity` and
    `effect_value` are tunables scattered in Detailed Rules; a table with safe
    ranges and the aspect each moves would satisfy `.claude/rules/design-docs.md`
    if the tier rises.
14. **[ADVISORY at `standard`] Player Fantasy section is absent.**
    Defensible for an invisible system; required if `modes.workflow` rises to
    `full`.

### Specialist Disagreements

None — lean mode, no specialist agents spawned.

### Nice-to-Have

- `hasPrefix(id, "dish_")` accepts a bare `dish_`; the regex allows a trailing
  underscore, so an id with no suffix validates.
- `display_name` / `description` still have no non-empty constraint; an empty
  string validates and renders as a blank HUD row.
- The example writes `effect_value=1` (an int literal) for a `float` field.
- Line 190's 场景与流程管理 row still says only "GDD 已写" with no path, unlike
  every other row.
- `meta-upgrades.md:96` consumes `get_station(id) → wave_size`, but this GDD's
  局外升级 row (line 181) lists only `get_upgrade`; the interface list is
  slightly incomplete.
- `∧ |D_bal| = 1` is unparenthesized in `valid(D)` (line 92); wrap it
  `(|D_bal| = 1)` so it cannot be read as `(… ∧ |D_bal|) = 1`.

### Completeness

6/6 sections present that are REQUIRED at `workflow: standard` (Overview,
Detailed Rules, Formulas, Edge Cases, Dependencies, Acceptance Criteria).
Formulas is required here because the system defines numeric rules. Advisory
gaps at this tier: Player Fantasy, Tuning Knobs.

### Dependency Graph

All 13 declared downstream dependencies resolve to an existing GDD.

- ✓ `design/gdd/dish-values.md`
- ✓ `design/gdd/cloying-and-taste.md`
- ✓ `design/gdd/menu-waves.md`
- ✓ `design/gdd/meta-upgrades.md`
- ✓ `design/gdd/run-flow.md` and `design/gdd/settlement-and-entry-fee.md` (one row)
- ✓ `design/gdd/save-system.md`
- ✓ `design/gdd/time-and-turns.md`
- ✓ `design/gdd/stomach-capacity.md`
- ✓ `design/gdd/records-and-milestones.md`
- ✓ `design/gdd/unlocks-and-stations.md`
- ✓ `design/gdd/ui-flow.md`
- ✓ `design/gdd/in-game-hud.md`
- ✓ `design/gdd/scene-flow.md` — row cites no path (nice-to-have above)
- Upstream: none declared, correct for a Foundation system.

Bidirectional check: all 16 sibling GDDs that name 数据配置 still reference it.
The one asymmetry is wording: this GDD marks the 纪录与里程碑 edge 条件
(line 186), `records-and-milestones.md:100/291` still marks it 硬 and calls
`MilestoneData` the "第四类记录".

### Registry Values

Searched the whole document for every registered name it could restate —
`station_canteen`, `entry_fee`, `cloy_cap`, `wave_size`, `cost_growth`,
`base_capacity`, `valid_record_set`, `upgrade_capacity` — 25 lines matched.

- ✓ `cloy_cap` = 3 (lines 51, 113, 239) — matches `cloy_cap`.
- ✓ `wave_size` default = 4 (lines 53, 116) — matches `wave_size_default`.
- ✓ `cost_growth` = 1.35, band 1.2–1.6, floor ≥ 1.0 (lines 41, 114) — matches
  `meta-upgrades.md` exactly.
- ✓ `base_capacity` = 20, range 1–999 (lines 59, 117, 185, 246) — matches the
  registry value.
- ✓ `valid_record_set` formula (line 92) matches every registry variable and the
  registry's `output_range`; it is now **stronger** than the registry expression
  (see the mirror drift below).
- ✓ `station_canteen_turn_budget` = 10 is not restated (only `≥ 1`), so no
  conflict.
- ✗ Line 124: `station_canteen.entry_fee = 30` vs registry 85 — Recommended 6;
  `dish-values.md` Open Question 2 already records 30 as illustrative.
- `upgrade_capacity` is not a registered name — Recommended 6.

Registry mirror drift (the registry is not editable in this review):
`entities.yaml:330`'s `valid_record_set` expression still reads
"`all R_bal`" without the new `|D_bal| = 1` clause; `base_capacity`
(`entities.yaml:1263`) still calls data ownership an open stomach question and
omits `design/gdd/data-config.md` from `referenced_by`; `wave_size_default`
(`entities.yaml:1166`) also omits it.

### Cross-Document Signals (out of scope — not changed here)

- `records-and-milestones.md:100/291` still says 硬 and "需扩展第四类记录",
  while its own Open Question 2 (line 443) now says the type would be the
  fifth. That document is internally inconsistent; this GDD's 条件 wording is
  the newer one.
- `stomach-capacity.md` Open Question 1 (line 316) is still listed as open
  although this GDD answers it (line 59).
- `systems-index.md:57` still shows 数据配置 as `In Review` — this review's
  recommended status is **Approved**.
- `systems-index.md:65` (结算与入场费) omits 数据配置 from Depends On although
  `settlement-and-entry-fee.md` lists it as 硬.
- `systems-index.md:197` rates the design scope `S`; this review still signals
  **L** (13 downstream systems, 4 formula blocks, 2 ADR-stage open questions).

### Verdict Rationale

Both blockers from round 2 are fixed precisely, and the fix did not introduce a
new contradiction: AC 1, AC 30 and the new Edge Case agree with the cardinality
clause, and AC 2/3/19 are now constructible under the prefix rule. Every
required field constraint is backed by both a formula term and an acceptance
criterion. The twelve remaining recommendations are documentation precision
(undefined symbols, one wrong range, an illustrative example), not design or
implementability defects; they are carried forward unchanged from round 1 and
do not block a programmer or a QA tester. Approved.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/time-and-turns.md 7f55a619613d6c2ff9f99b9458e39d0f5f48895c
Reviewed-Content-Hash: design/gdd/stomach-capacity.md bb97f41a121c6c18d15f7d10c23387ff25688b11
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 0f58ea62a2e8fa9ace2effe652cc644acab89d5b
Reviewed-Content-Hash: design/gdd/scene-flow.md a2c3cc81155d51d9fae36ecc8ef27bfc1d689a59
Reviewed-Content-Hash: design/gdd/input.md ee9ea6022f8e289d9e00ef8f9eabb181b0c5349f
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/systems-index.md 63d73495be76b5b37ed9bac50452c07ef281d41a
```

---

## Review — 2026-11-04 — Verdict: APPROVED

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 0 | Recommended: 4
Prior verdict resolved: Round 3 approval; this round re-reviews the cue-catalog
sync only (mechanical remediation of ADR-0001's sixth manifest / ADR-0009 D7-D8).

### Change Under Review

The document gained the audio cue catalog as a load-time manifest: Summary,
Rule 2, Rule 9 APIs, new Rule 11, a `cueValid(C_cue)` term in `valid(D)`, the
`C_cue` / `cueValid` variables, three edge cases, the 音效反馈 dependency row in
both the narrative and the table, the Visual/Audio note, the Cross-References
note, the AC preamble and AC 33–35. Nothing else was touched; no balance value
changed.

### Required Before Implementation

None.

### Findings

1. **Ordinal numbering — fixed during this review.** The sync first called the
   audio cue catalog "第五条由本系统加载的清单" (Rule 2) and "第五条本系统清单"
   (Rule 11 heading), while `adr-0001` §2, `adr-0009` D4/D8 and
   `audio-feedback.md` all call it the **第六个清单** (milestone is fifth in the
   ADR manifest table). A reader reconciling the GDD with the ADR would have
   found two different "fifth manifests". Corrected to
   "启动门的第六条清单、也是本系统自己加载的五份清单之一", with the milestone
   catalog explicitly placed at the fifth slot and outside this schema.

2. **Rule delegation is faithful.** `cueValid(C)` is defined as "目录无
   `AUDIO_CUE_*` ERROR" and the document states it does not restate field
   semantics. The count matches ADR-0009 D7 exactly: schema, duplicate id,
   unknown bus, bad range, empty streams, missing stream, zero/!= expected
   duration, priority — eight rules, all `ERROR`, all fatal to the whole
   publication.

3. **Formula placement is correct.** `valid(D)` now ends in
   `∧ cueValid(C_cue)`; `C_cue` and `cueValid` are both declared in the
   Variables table with source `data file` / `calculated`. A literal
   implementation of the formula has every symbol it needs.

4. **Failure semantics are consistent with the boot gate.** Cascading failure
   ("任一 `ERROR` 即整体 `Failed`，不发布任何部分") matches Rule 1 and AC 17;
   missing / wrong-type catalog reuses `CATALOG_FILE_MISSING` /
   `CATALOG_WRONG_TYPE`, both of which exist in ADR-0001 §5's diagnostic table.
   Runtime playback failure is correctly declared out of scope and routed to
   the audio system's silent degradation (ADR-0009 D14).

5. **Bidirectional dependency is recorded.** 音效反馈 appears as a 硬 downstream
   consumer in both the Dependencies narrative (line ~199) and the table
   (line ~200), with `AudioCatalogView` named as the read-only adapter;
   `audio-feedback.md` already lists 数据配置 as 硬 and points back to this
   document. Upstream is still "无", with an explicit note that the cue schema
   comes from ADR-0009 rather than from a GDD dependency — so no new edge and
   no new cycle.

6. **Acceptance criteria are executable and scoped.** AC 33 covers the Ready
   path plus `get_cue` / `has_cue` / `cue_catalog()`; AC 34 covers four bad-rule
   fixtures and repeats the all-or-nothing assertion; AC 35 covers missing and
   wrong-type files. The AC preamble excludes 33–35 from the default-fixture
   rule, so earlier criteria remain independently runnable.

### Recommended Revisions

1. `valid(D)` now consumes a second input. Renaming the predicate to
   `valid(D, C_cue)` (or `valid(content)`) would keep the signature and the
   prose "记录集合合法性" aligned; today the extra argument is explained only
   in the Variables table.
2. AC 34 samples four of the eight rules. ADR-0009's test plan mandates one
   fixture per rule, so coverage is not lost project-wide; saying "至少" in
   AC 34 (or pointing to ADR-0009's fixture list) would prevent a QA reader
   from assuming four is the complete set.
3. `cue_catalog()` does not restate its `READY`-only precondition in Rule 9;
   the ADR does. One clause would close the gap.
4. The registry's one-line DataConfig interface summary
   (`architecture.yaml:105`) lists `get_cue` but omits `has_cue` /
   `cue_catalog()`, which the detailed entries (`:328`, `:504`) do include.
   Registry wording only; no design impact.

### Specialist Disagreements

None — lean mode.

### Completeness

`gdd-structure-check.sh` reports all six required sections present; Player
Fantasy and Tuning Knobs remain absent (optional at this tier, unchanged from
round 3). Acceptance criteria now run 1–35 with the pre-existing duplicate
"23" label carried forward unchanged; the review does not renumber it because
renumbering would break external citations, and both entries are independently
identifiable by content.

### Dependency Graph

No edge added. 数据配置 → 音效反馈 is a read-only consumer edge already
recorded on both sides; 数据配置 still has zero upstream GDD dependencies.
`adr-dep-graph.sh` reports no `CYCLE` and the order
`0001, 0002, 0005, 0003, 0004, 0008, 0006, 0007, 0009` is unchanged by this
edit.

### Registry Values

`registry.sh validate`: entities 79, TR 104, architecture 83, 0 problems.
`get_cue` / `has_cue` / `cue_catalog()` already exist in `architecture.yaml`
(lines 105, 328, 504), and `TR-audio-feedback-001/007` already attribute the
sixth-manifest gate to ADR-0009; no registry edit was needed for this sync.

### Verdict Rationale

The sync is additive and defers field semantics to the ADR that owns them,
which is the correct division: this GDD owns execution order, aggregation and
all-or-nothing publication. The one real defect found — the fifth/sixth
ordinal contradiction — was a wording error introduced by the sync itself and
is now fixed in all three places it appeared. Formula, edge cases, dependency
table, cross-reference and ACs agree with `adr-0001` §2/§5b, `adr-0009` D7/D8
and `audio-feedback.md`. The four recommendations are precision items, not
implementability or testability defects. Approved; the index row returns to
`Approved` and the approved count to 18.

---

Receipt at stamp time:
```
Reviewed-Content-Hash: design/gdd/data-config.md cd42141a823868fc47650210cb108c0ae69259a0
Reviewed-Content-Hash: design/registry/entities.yaml 02f5c072fc535530c710fd71d77d31e2dce021ed
Reviewed-Content-Hash: docs/registry/architecture.yaml 57e780c48f452224903dcff5c053973437146a74
Reviewed-Content-Hash: design/gdd/audio-feedback.md b02b3fa05db215dff9f4882849161e7651506ef4
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 7cb99488fee8beeba202523848adecd3c06c2c7c
Reviewed-Content-Hash: design/gdd/scene-flow.md 07753accfc002be7257cbcdd17323c73c2b6845a
Reviewed-Content-Hash: design/gdd/systems-index.md 1119e17f51ee38b2b6fdb56db5d6301d6db94f47
```

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean)
Blocking items: 4 | Recommended: 5
Summary: Both edited `entry_fee` rows now correctly put resolution and freezing in run-flow and make settlement read `RunLog.entry_fee`, matching run-flow rules 22/23, settlement rule 2 and the registry's `running_net`/`shortfall` notes. The over-correction is in the same cells: "结算不直接查数据目录" is false (settlement is a hard downstream that still reads station/dish records and the `Ready` gate), "写进本局快照" is not published by run-flow, and the dependency table still has no 结算与入场费 row. Whole-document read (no delta), 6/6 standard sections present; Player Fantasy and Tuning Knobs remain advisory at `standard`.
Prior verdict resolved: Yes — latest entry (2026-11-04, lean, APPROVED) carried 0 blocking items; its note that the duplicated AC 23 label was carried unchanged still holds and is now re-raised below because Game Feel cites it.
Findings:
- [BLOCKING] Interactions `:82` / Dependencies `:192` — "结算只读 `RunLog.entry_fee`，不直接查数据目录" contradicts `settlement-and-entry-fee.md:95` and `:324`, where 结算与入场费 is a hard downstream reading `get_station(id)` (存在性、`display_name`), `get_dish(id)` (`price`、`satiety`), the int64 rule and the `Ready` gate. Only the `entry_fee` read moved to `RunLog`. The dependency table (`:186-201`) has no 结算与入场费 row at all although settlement declares data-config its upstream, so `:203`'s "均已列出本系统" is false and the bidirectional rule is broken.
- [BLOCKING] Interactions `:82` / Dependencies `:192` — "写进本局快照与 `RunLog`" overstates the contract: run-flow's snapshot field table (`run-flow.md:136-150`) publishes no `entry_fee` (only `running_net` starting at `−entry_fee`); `entry_fee` lives only in `RunLog` (`run-flow.md:40`). `:192` also attaches `price(d)` — the 菜品数值 interface (`run-flow.md:119`) — to the run-flow/data-config edge, while data-config's own price interface is `get_dish(id).price`.
- [BLOCKING] Registry Values `:130` — the formula example gives the real station id `station_canteen` `entry_fee=30`, but registry `station_canteen_entry_fee` is `85` (`entities.yaml:1211-1220`, notes "Stored as StationData.entry_fee"). A copied fixture ships a 55-yuan-wrong entry fee; use 85 or a synthetic `station_example` id.
- [BLOCKING] Acceptance Criteria `:250-251` — two criteria are numbered 23 (`cloy_cap` and the hot-reload price test); Game Feel `:214` cites "Acceptance Criteria 第 23 条" for the hot-reload behaviour, which now resolves to the `cloy_cap` criterion. Label one of them `23a`/`23b` instead of renumbering (preserves external citations).
- [RECOMMENDED] Cross-document (`run-flow.md:466`) — Open Question 10 still says "本设计由结算读取；本系统只解析 `station_id`", contradicting run-flow's own rules 22/23 and this document's corrected rows. Close it, otherwise the two documents disagree about who reads `entry_fee` while both are In Review.
- [RECOMMENDED] Dependencies `:192` — the 餐局流程 row now carries a settlement clause; split it into a 结算与入场费 row (`RunLog.entry_fee` only, plus its record reads from the blocker above) so each edge is readable on its own.
- [RECOMMENDED] Coverage — the changed contract has no criterion in this document; it is only proven by `run-flow.md` AC 1/14 and `settlement-and-entry-fee.md` rule 2 / AC 25. Add a one-line AC ("station `entry_fee` change is visible in `RunLog.entry_fee`, and never re-read at settlement") or cite the owning ACs.
- [RECOMMENDED] Acceptance Criteria 22 (`:249`) — a p95 ≤100 ms budget on 300 records is hardware-dependent; keep it but state it as a non-blocking evidence item, as run-flow does for performance criteria.
- [RECOMMENDED] Process (`systems-index.md:58`) — the row is still `Approved` while the header is `In Review`; this substantive edit invalidates the approval (`.claude/rules/design-docs.md:28-31`).

Reviewed-Content-Hash: design/gdd/data-config.md 92b3a28fc63465eacf3a88d3fec13bdff9f9455f
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/menu-waves.md a0b26691ca0f6ba5cac4c01ac0704913179b43ed
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean)
Blocking items: 3 | Recommended: 8
Summary: All four prior blocking items are closed: the combined 餐局流程/结算 row is split (`:82`, `:83`) with settlement's real reads (`get_station` → `display_name`, `get_dish` → `price`/`satiety`) and `entry_fee` sourced only from `RunLog.entry_fee`; a 结算与入场费 row now exists in Dependencies (`:194`) so reciprocity with `settlement-and-entry-fee.md:95`/`:324` holds; "写进本局快照" is gone; the example's `entry_fee` is 85; and the duplicate AC 23 is renumbered 23a (`:253`). Three residuals: the same edited example cell still fails this document's own validator (`taste_tags=[oil]` is outside `TAGSET`), two "第 23 条" citations still point at the `cloy_cap` test instead of 23a, and the replacement parenthetical now under-claims the snapshot ("只发布 `shortfall`"). Whole-document read (no delta), 6/6 standard sections present; Player Fantasy and Tuning Knobs remain advisory at `standard`.
Prior verdict resolved: Partially — 4/4 blocking closed. Of the prior 5 recommended: 2 closed (run-flow Open Question 10 marked resolved 2026-11-17; the 餐局流程 row split), 3 open and re-raised below, plus one carried-forward item from 2026-11-04 (Recommended 6, the invented ids).
Findings:
- [BLOCKING] Formulas example `:131` — `taste_tags=[oil]` is not in the fixed table (酸/甜/苦/辣/鲜/冰/油, `:34`/`:124`; registry spells it `油`), so `R_dish(d)` (`:99`) is false and the example's own "`valid(D)=true`" claim fails; anyone copying the block as a fixture gets `Failed`. The same cell still attaches `dish_fried_rice` (not one of the 12 registered dishes) and `upgrade_capacity` cost 40 / max 3 (the real record is `upgrade_stomach_capacity`, cost 12 / max 5, `meta-upgrades.md:278`) to the real `station_canteen`. Use `油` plus a registered dish, or relabel the whole block as synthetic (`station_example`, `dish_example_a`) so it cannot be pasted into a `.tres` (carried Recommended 6 from 2026-11-04).
- [BLOCKING] Game Feel `:216` — still cites "Acceptance Criteria 第 23 条" for the `.tres` price-change behaviour, but 23 is now the `cloy_cap` test and the price test is 23a (`:253`); `:268` repeats the same off-by-one ("转为第 23 条的行为验证"). The same sentence's "不重启" is not what AC 23a verifies (it starts the game) and contradicts Open Question 3, where hot reload is deferred to Vertical Slice.
- [BLOCKING] Interactions `:82` — "（运行期快照不另存 `entry_fee`，只发布 `shortfall`）" is false: run-flow's snapshot publishes both `running_net` and `shortfall` (`run-flow.md:147`). The cell edited to remove one over-claim now under-claims instead; write "只发布 `running_net` / `shortfall`".
- [RECOMMENDED] Dependencies `:194` — the settlement row omits the `Ready` gate and the int64 sum rule that `settlement-and-entry-fee.md:324` says it relies on; add them so the row matches its consumer.
- [RECOMMENDED] Interactions `:82` / Dependencies `:193` — neither cell states the resolution precedence (`RunConfig.entry_fee_override` else `StationData.entry_fee`, run-flow rule 22) nor that `station_id` resolution and `dish_pool` id set are read in the same step; "解析 `station_id` 与 `entry_fee`" hides the override.
- [RECOMMENDED] Coverage — the contract the last round asked for still has no criterion here: AC 23a proves price-from-data, nothing proves `RunLog.entry_fee` is frozen from `StationData.entry_fee` and never re-read at settlement (only `run-flow.md` AC 1/14 and settlement rule 2 / AC 25). Add one line or cite those ACs. (Second recurrence.)
- [RECOMMENDED] Acceptance Criteria 22 (`:251`) — p95 ≤ 100 ms on 300 records still names no reference machine (carried Recommended 7, still open).
- [RECOMMENDED] Registry — `station_canteen_entry_fee` (`entities.yaml:1212`) `referenced_by` omits this file although `:131` restates 85; `cloy_cap` and `wave_size_default` already include it. Registry edits are out of scope here.
- [RECOMMENDED] Cross-document (`settlement-and-entry-fee.md:64`) — its `RunLog` field table says "结算不再直接读档口记录", contradicting that file's own `:95` and `:324` (and this file's `:83`) that settlement reads `get_station(display_name)`; the note should read "不再从档口记录读 `entry_fee`". Owner: settlement doc (unchanged this round).
- [RECOMMENDED] Cross-document (`run-flow.md` Open Question 10) — marked resolved 2026-11-17 but cites "`data-config.md:82`/`:192`"; `:192` is now the 局外升级 row (the 餐局流程 row is `:193`). Owner: run-flow doc.
- [RECOMMENDED] Process — `systems-index.md:58` still reads `Approved` while the header is `In Review`; `.claude/rules/design-docs.md:28-31` voids that approval. (Index not edited here.)

Reviewed-Content-Hash: design/gdd/data-config.md 36fd1ebaca36d1edee0c93b674f21d0418798713
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION（第 4 轮：三条 blocker 关闭核验 + 整篇复检）
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean)
Blocking items: 1 | Recommended: 9
Summary: 三条 blocker 里两条完全关闭、一条只做了一半。已关闭：`taste_tags` 改为 `油`（`:131`，在 TAGSET 内，`R_dish` 不再自相矛盾）；两处引用改为「第 23a 条」（`:216`、`:268`），23 与 23a 不再撞号；快照描述改为「运行期快照不另存 `entry_fee`，只发布 `running_net` / `shortfall` 的派生值」（`:82`），与 `run-flow.md:136-152` 逐字相符，既不越权也不再少报。餐局流程行（`:82`/`:193`）与结算行（`:83`/`:194`）的读取范围拆分正确：前者解析 `station_id` 与 `entry_fee` 并冻结进 `RunLog.entry_fee`，后者读 `get_station` → 存在性与 `display_name`、`get_dish` → `price`/`satiety`、`entry_fee` 只从 `RunLog.entry_fee` 取，与 `settlement-and-entry-fee.md:95`/`:324` 逐项相符；示例的 `entry_fee = 85` 与注册表 `station_canteen_entry_fee`（`entities.yaml:1212`）及结算规则 2 一致。未关闭的一半：示例仍把未登记的 `dish_fried_rice` 与 `upgrade_capacity`（`cost=40` / `max_level=3`）挂在真实档口 `station_canteen` 上，与注册表 12 道菜、`meta-upgrades.md:278-280` 的三条升级目录冲突（见 blocking）。全篇阅读（非 delta），6/6 standard 节齐备；Player Fantasy 与 Tuning Knobs 在 `standard` 下仍为 advisory。
Prior verdict resolved: Partially — 3/3 blocking 名义上关闭，但 #1 只落了 `油` 这半句，同一单元的真实 id 混用未处理，按「半应用仍算 blocking」保留；#2、#3 完全关闭。上轮 8 条 recommended 中 0 条关闭（结算行的 `Ready`/int64、`entry_fee_override` 优先级、冻结契约缺 AC、AC 22 参考机、注册表 `referenced_by`、结算 `:64`、run-flow OQ10 行号、索引状态），全部按原样再提，另补 1 条新的跨文档条目。
Findings:
- [BLOCKING] Formulas Example `:131` — 该单元只把 `oil` 改成 `油`，其余照旧：`dish_fried_rice`（`price=8`、`satiety=2`）不在注册表的 12 道菜里，`upgrade_capacity`（`cost=40`、`max_level=3`）也不存在——`meta-upgrades.md:278-280` 的真实目录是 `upgrade_stomach_capacity`（12 / 1.35 / 5）、`upgrade_cloying_resistance`（20 / 1.35 / 1）、`upgrade_menu_vision`（16 / 1.35 / 3）；两者却都挂在真实档口 `station_canteen` 与真实入场费 85 旁边。上一轮的 blocker 原文是「用 `油` 加上一道登记菜，或把整段标为合成（`station_example` / `dish_example_a`）」，两条路一条也没走完。照抄成 `.tres` 的人会得到一个只剩 2 道菜（其中一道不存在）的 `station_canteen`，并多出一条与升级目录冲突的 `upgrade_capacity`。同句「2 个档口、4 道菜、2 项升级」只描述了 2 道菜与 1 项升级，`valid(D)=true` 对未描述的记录无法核验。
- [RECOMMENDED] Dependencies `:194` — 结算行仍缺 `Ready` 前提与 int64 求和规则，而 `settlement-and-entry-fee.md:324` 明写它依赖这两条。（第二轮再提）
- [RECOMMENDED] Interactions `:82` / Dependencies `:193` — 两行都没写入场费的解析优先级（`RunConfig.entry_fee_override` 优先，否则 `StationData.entry_fee`，`run-flow.md` 规则 22），也没写 `station_id` 解析与 `dish_pool` id 集合是同一步读完；「解析 `station_id` 与 `entry_fee`」把 override 藏掉了。（第二轮再提）
- [RECOMMENDED] Coverage — 上一轮要求的判据仍缺：AC 23a 只证价格来自数据层，没有任何一条断言 `RunLog.entry_fee` 由 `StationData.entry_fee` 冻结、结算期不再重查（目前只有 `run-flow.md` AC 1/14 与 settlement 规则 2 / AC 25 间接覆盖）。补一行或明确引用那几条。（第三轮再提）
- [RECOMMENDED] Acceptance Criteria 22 `:251` — 300 条记录 p95 ≤ 100 ms 仍不写参考机器与判定口径（是否阻断）。（第三轮再提）
- [RECOMMENDED] Registry — `station_canteen_entry_fee`（`entities.yaml:1212`）的 `referenced_by` 仍不含本文件，虽然 `:131` 重述 85；`cloy_cap` 与 `wave_size_default` 已含本文件。注册表不在本次可改范围。
- [RECOMMENDED] 跨文档（`settlement-and-entry-fee.md:64`）— `RunLog` 字段表仍写「结算不再直接读档口记录」，与同一文件的 `:95`/`:324` 以及本文件 `:83`（结算读 `get_station` 的 `display_name`、`get_dish` 的 `price`/`satiety`）相反；应改为「不再从档口记录读 `entry_fee`」。owner 是结算文档（本轮未改）。（第二轮再提）
- [RECOMMENDED] 跨文档（`run-flow.md` Open Question 10，`:465`）— 已标「已解决（2026-11-17）」，但引用的 `data-config.md:82`/`:192` 中 `:192` 现在是局外升级行，餐局流程行是 `:193`；引用应随本次拆分更新。owner 是 run-flow 文档。
- [RECOMMENDED] 跨文档（本文件 `:201` 局内 HUD 行）— 「通过菜品数值/时间与回合取得数字，不直接读 catalog」与 ADR-0019 D3 第 7 步（`:213`）不符：档口显示名与菜品图标由 HUD 的投影层 `HudSource` 直读 `data.get_station(station_id)` / `data.get_dish_icon(id)`，`in-game-hud.md:173`/`:392` 也是同一说法。三处应统一为「数字经下游转发；显示名与图标由 HUD 投影层只读直取」。
- [RECOMMENDED] Process — `systems-index.md:58` 仍读 `Approved`，而本文件表头是 `In Review`，`.claude/rules/design-docs.md:28-31` 使该批准失效。（第三轮再提；索引未在此修改）

### Nice-to-Have

- 规则 7 把「建议增加一条：至少存在一个 `unlock_cost = 0` 的档口」写在校验规则正文里，读起来像硬规则；运行期兜底已由「解锁与档口」的 `NoPlayableStation` 提供，可把这条明确为 warning 或移出规则 7。
- TAGSET（`:34`/`:124`）有 7 个标签，12 道菜的注册表实际只用到 6 个（`苦` 未被使用）；合法，可在 `TAGSET` 行注明为预留。

### 判定说明

lean 模式不派发 `creative-director`，主评审意见即最终意见。三条 blocker 的核验结论是「两整半」：`taste_tags`、引用编号、快照描述三处文字都已改到位，而且快照描述既不越权也不像上一轮那样反向少报；餐局流程与结算两条读取边的归属也与下游 GDD 的接口行逐项对齐。剩下的这条 blocking 不是新问题，而是上一轮同一单元里「真实 id 混用未登记/冲突记录」那半句没有执行——技能 Phase 3 的注册表检查对示例、目录与变体注记一视同仁，因此按 Required Before Implementation 处理。它只需改一个单元（换登记菜或整段改合成 id），不需要动任何规则或 AC。

### Scope Signal

Rough scope signal: **S**（基础层、无上游 GDD、公式只有集合合法性与三项派生计数；producer 应在排期前复核）。

### Verdict: NEEDS REVISION

1 条 blocking、9 条 recommended、2 条 nice-to-have。本 GDD 未被修改；`systems-index.md:58` 的状态待 blocking 关闭并由编排会话统一收口。

Reviewed-Content-Hash: design/gdd/data-config.md 8d799e593ad60b254814ede22b4f9b3fc8de99e4
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 93968bd02251e306f5b99c7b1cc5800272e54a06
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 23304c8212ab59cede6f6f3d81cd40cea079cde5
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

---

## Review — 2026-11-17 — Verdict: APPROVED（第 5 轮：示例夹具改用已发布记录后的全量 lean 复审）
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean)
Blocking items: 0 | Recommended: 16
Summary: 第 4 轮唯一的 blocking 已完全关闭。示例（`:131`）现在逐项对得上已发布目录：`station_canteen` 的 `entry_fee=85` 与注册表 `station_canteen_entry_fee`（`entities.yaml:1212`）、结算规则 2、`dish-values.md:42` 一致，`cloy_cap=3` 等于本文件规则 5 的默认值与注册表 `cloy_cap`（`entities.yaml:1180`）；`dish_mapo_tofu` 12 / 2 / 辣,鲜,油 与 `dish_braised_pork` 16 / 3 / 甜,油 逐字段等于 `dish-values.md:52`、`:55` 的 MVP 菜品表；`upgrade_stomach_capacity` 的 `cost=12`、`max_level=5`、`effect_value=1.0` 等于 `meta-upgrades.md:278` 的 MVP 升级目录（`cost_growth` 未写，取 Resource 默认 1.35，与记录一致）。`dish_fried_rice` / `upgrade_capacity` 全文已无命中，`dish_pool` 大小不变式（2 = 2）成立，标签全部落在 TAGSET 内，「1 个档口、2 道菜、1 项升级」与正文描述一致，并明确声明这是截取夹具、不代表完整 12 道菜目录。全篇阅读（非 delta），6/6 standard 节齐备；Player Fantasy 与 Tuning Knobs 在 `standard` 下仍为 advisory。
Prior verdict resolved: Yes — 第 4 轮的 1 条 blocking 关闭（示例的真实 id、真实数值、池大小与叙述四处同时对齐，自造菜与自造升级已清除）。上轮 9 条 recommended 因本轮只改示例而 0 条关闭，全部按原样再提；另补 2 条本轮新发现（示例的 `valid(D)=true` 仍隐含未列出的 `BalanceData` 与 cue 目录；规则 7 的 `DISPLAY_NAME_NOT_A_KEY` 没有对应 AC），以及 5 条更早轮次仍未关闭的公式/AC 条目。
Findings:
- [RECOMMENDED] Formulas Example `:131` — 「`valid(D)=true`」仍依赖夹具未列出的两项：`|D_bal| = 1`（`:95`）与 `cueValid(C_cue)`（`:95`/`:121`）。示例只描述 1 档口 / 2 菜 / 1 升级，照抄成夹具会得到 `valid(D)=false`。补一句「另加 1 个 `BalanceData` 与一份合法 cue 目录」即自洽。
- [RECOMMENDED] Dependencies `:194` — 结算行仍缺 `Ready` 前提与 int64 求和规则，而 `settlement-and-entry-fee.md:324` 明写它依赖这两条；同一文件的 `:83` 已列出完整读取面，两行粒度不一致。（第四轮再提）
- [RECOMMENDED] Interactions `:82` / Dependencies `:193` — 两行仍未写入场费解析优先级（`RunConfig.entry_fee_override` 优先，否则 `StationData.entry_fee`，`run-flow.md:50` 规则 22），也未写 `turn_budget` 与 `dish_pool` id 集合是在同一步读完；「解析 `station_id` 与 `entry_fee`」把 override 与另两项读取藏掉了。（第三轮再提）
- [RECOMMENDED] Coverage — 仍无一条 AC 证明 `RunLog.entry_fee` 由 `StationData.entry_fee` 冻结、结算期不再重查；目前只有 `run-flow.md` AC 1/14 与 settlement 规则 2 / AC 25 间接覆盖。补一行或明确引用那几条。（第四轮再提）
- [RECOMMENDED] Detailed Rules 规则 7 `:55` — `DISPLAY_NAME_NOT_A_KEY` 是规则正文里唯一的 warning 级校验，但 AC 1–35 无一条覆盖它（AC 9 只测 `icon` 警告，AC 31 只测 `COST_GROWTH_OUTSIDE_DESIGN_BAND`）。补一条「`display_name` 非 key → Ready 且 warnings=1」即可闭合。
- [RECOMMENDED] Detailed Rules 规则 7 `:55` / Formulas `:137` — 「建议增加一条：至少存在一个 `unlock_cost = 0` 的档口」仍写在硬规则列表里；同时 `dish_count ≥ 1` 被声明为「由 `R_sta` 蕴含」，但 `D_sta = ∅` 时 `R_sta` 空真、`valid(D)=true`、`dish_count = 0`。要么把集合级谓词写进 `valid(D)` 并配 AC，要么把该建议降为 warning 并软化范围声明。（第三轮再提）
- [RECOMMENDED] Formulas `:99`–`:125` — `D_ids`、`S_ids`、`U_ids`、`uniqueIds(·)`、`cyclic(·)`、`dish(·)`、`upgrade_count` 仍出现在公式而未在变量表定义；另外 `prereq_depth` 的范围 `1–upgrade_count`（`:139`）与 AC 20 的三节点链 `depth=2`、无前置升级 `depth=0` 矛盾，应为 `0–(upgrade_count − 1)`。（第四轮再提）
- [RECOMMENDED] Formulas `:100` — `R_upg` 仍未校验 `effect_type` 枚举成员：`(u.effect_type ∈ {STOMACH_CAPACITY, MENU_VISION} → …)` 对枚举外的整数空真，`.tres` 里 `effect_type=7` 可通过；id 正则 `^[a-z][a-z0-9_]*$` 也只写在 `DishData` 规则下，`upgrade_BAD` 仅靠前缀即可通过。（第三轮再提）
- [RECOMMENDED] Detailed Rules 规则 4 `:44` — 「小数部分在解析时取整」未说明方向；`meta-upgrades.md:57` 规则 16 用 `floor`，2.5 存在两种读法。写明「向下取整」或指向该规则即可。
- [RECOMMENDED] Acceptance Criteria 22 `:251` — 仍只写「最小规格 PC、非调试构建」，没有具体参考机，也未说明超预算是否阻断；`run-flow.md` 对性能判据的做法可参照。（第四轮再提）
- [RECOMMENDED] Acceptance Criteria 19 `:248` — 「操作被拒绝或抛错」两种行为都能通过，未指定机制；`DishData` 是可变导出 Resource，需点名 setter 或只读投影之一。（第二轮再提）
- [RECOMMENDED] Registry — 示例重述了 85 / 12,2 / 16,3，但 `station_canteen_entry_fee`（`entities.yaml:1215`）、`dish_mapo_tofu`（`:197`）、`dish_braised_pork`（`:245`）的 `referenced_by` 都不含本文件；`cloy_cap` 与 `wave_size_default` 已含。注册表不在本次可改范围。
- [RECOMMENDED] 跨文档（`settlement-and-entry-fee.md:64`）— `RunLog` 字段表仍写「结算不再直接读档口记录」，与同一文件 `:324`（读 `get_station` 的 `display_name`）及本文件 `:83` 相反；应改为「不再从档口记录读 `entry_fee`」。（第三轮再提）
- [RECOMMENDED] 跨文档（`run-flow.md:468` Open Question 10）— 已标已解决，但引用的 `data-config.md:82`/`:192` 中 `:192` 现在是局外升级行，餐局流程行是 `:193`；引用应随拆分更新。
- [RECOMMENDED] 跨文档（本文件 `:201` 局内 HUD 行）— 「通过菜品数值/时间与回合取得数字，不直接读 catalog」与 `in-game-hud.md:180`（卡片图标/名称来自 `DataConfig.get_dish_icon`）、`:259` 以及 `adr-0019:213`（`HudSource` 直取 `get_station` 显示名与 `get_dish_icon`）不符；三处应统一为「数字经下游转发；显示名与图标由 HUD 投影层只读直取」。（第二轮再提）
- [RECOMMENDED] Process — `systems-index.md:58` 仍读 `Approved`，而本文件表头是 `In Review`，`.claude/rules/design-docs.md:28-31` 使该批准失效。（第四轮再提；索引未在此修改）

### Nice-to-Have

- 示例的 `station_canteen` 只带 2 道菜，而 `dish-values.md:42` 与 `menu-waves.md:313` 都以 12 道菜描述该档口；这 2 道菜的池也不满足 `dish-values.md:36` 规则 10 的 `menu_healthy`（UV 跨度 1.125，无 ≥5 格档）。正文已声明「截取夹具」，故不阻断；若希望夹具可直接照抄，档口 id 改用 `station_example` 更稳。
- Dependencies `:203` 的「场景与流程管理」行只有「GDD 已写」，没有像其他行那样给出 `design/gdd/scene-flow.md` 路径。
- Open Question 1 与 2 的目标解决时间写「原型阶段（2026-11-17 前）」，今日已到期且仍开放；建议更新日期或给出裁决。
- TAGSET（`:34`/`:124`）有 7 个标签，12 道菜的注册表只用到 6 个（`苦` 未被使用）；合法，可在 `TAGSET` 行注明为预留。
- Player Fantasy 与 Tuning Knobs 在 `standard` 下仍为 advisory（不变）。

### 判定说明

本轮是全新全篇 lean 复审，不是 delta：技能 Phase 3 的注册表检查覆盖表格、目录与示例/变体，因此示例与 `dish-values.md`、`meta-upgrades.md`、`entities.yaml` 和结算规则逐值比对。结论是第 4 轮 blocking 的四项（真实档口 id、真实菜品数值、真实升级目录、叙述计数）全部落地，池大小不变式与 TAGSET 成员都成立，没有残留自造 id；其余 16 条 recommended 都是精度、覆盖或跨文档同步问题——它们不改任何规则、公式、AC 或平衡值，也不会让程序员或 QA 无法实现/无法判定（示例少列的 `BalanceData`/cue 属于说明补全，示例本身不承担 AC 职责）。据 blocking 定义（是否直接改变实现、或让某条 AC/规则无法通过），本轮 0 条 blocking，故 APPROVED。6/6 standard 所需节齐备（Overview、Detailed Rules、Formulas、Edge Cases、Dependencies、Acceptance Criteria），Formulas 因系统定义数值规则而必需。

### Scope Signal

Rough scope signal: **S**（基础层、无上游 GDD、公式只有集合合法性与三项派生计数；producer 应在排期前复核）。

### Verdict: APPROVED

0 条 blocking、16 条 recommended、5 条 nice-to-have。示例夹具已可作合法 `valid(D)` 的说明书使用（补齐 `BalanceData` 与 cue 目录后可直接当夹具）；`systems-index.md:58` 的 `Approved` 与本文件表头 `In Review` 的状态待编排会话统一收口，索引未在此修改。

Reviewed-Content-Hash: design/gdd/data-config.md 4481fe6d8a0047d784a160a32084ac642ba9fa24
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 06a670b544d8998eef4a8219cc9fa9d5a6b740b9
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/data-config.md e4d2ee1de1f4c9affd59a3993adb89056d5e4fa9

## Re-stamp — 2026-11-17（架构复审后）
仅同步闸门输入文件的当前字节（GDD 本体自上次复审后只改了 Status 头行）；未改规则、公式、AC、依赖、契约或平衡数值。

Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/data-config.md e4d2ee1de1f4c9affd59a3993adb89056d5e4fa9
Reviewed-Content-Hash: design/gdd/dish-values.md 3eac1141e6c1209b59cc5e1ceeaa5c9ce85bc186
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/in-game-hud.md 6d7a4700f0f8b718eeb3bf0cbe9db7993df1f43d
Reviewed-Content-Hash: design/gdd/input.md 423ac2b02d68a816905cc5060da7541aa726980c
Reviewed-Content-Hash: design/gdd/menu-waves.md af47aef125e14677602eeab6eed5ae98132c6113
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 213f8dbe40b44748fc32857680f09991fff9d612
Reviewed-Content-Hash: design/gdd/run-flow.md da67f644d65824cb6ae98081cea03f69f0e77a0d
Reviewed-Content-Hash: design/gdd/scene-flow.md 2439af499a5bdd48e2623ed7abe998d20c8b397c
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 82c3ea703a3c2d1e2b6bd0a8f00b4f87569cc4a8
Reviewed-Content-Hash: design/gdd/systems-index.md ccecd7af3ef6993252cb6901db7435d0985c0953
Reviewed-Content-Hash: design/gdd/time-and-turns.md ce54e5bd984a1c7a2bbb973bf124bd998563f5b4
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 06a670b544d8998eef4a8219cc9fa9d5a6b740b9
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 30b02733b0ec60c3e18d0acf82d8fd5b740909b9
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 492f84e663be2d81c22127d40ef68fd096dcb3a4
Reviewed-Content-Hash: docs/registry/architecture.yaml 57e780c48f452224903dcff5c053973437146a74
