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
