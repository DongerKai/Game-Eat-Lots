# First-Run Onboarding (首局引导) — Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 6 | Recommended: 13
Summary: The teaching economy is sound and every worked number reproduces exactly against the registry — base satiety 20 = capacity 20, six dishes needing 21–22 cells, best five 82, fee 40, net +42, 205.0%, and the 12-yuan stomach upgrade is affordable. The structure is also unusually disciplined: one owner per concern, a real three-state `is_first_run` truth table, no timers anywhere, and a degradation ladder that never blocks the run. The blockers are contract gaps, not design faults: the `Blocked` whitelist grants `ui_accept` to the *context* with no mechanism that keeps run-flow from also receiving it, `replacement_hint`'s pool domain is undefined and its example is impossible under the stated six-dish pool, `get_anchor_rect` and the protected rects do not exist on HUD's side, `ui-flow` never applies the preset, three "contract gaps" the GDD asks the orchestration session to close have already landed, and one registry-derived number in the rationale block is wrong.
Prior verdict resolved: First review
Suggested index status: `18 | 首局引导 | Meta | Vertical Slice | Needs Revision | design/gdd/first-run-onboarding.md | 餐局流程, 局内 HUD, 输入, 场景与流程管理, 菜品数值, 结算与入场费`

Findings:
- [BLOCKING] Detailed Design (`:56`, `:187`) / Acceptance Criteria (`:465`, `:473`): the `Blocked` whitelist grants `ui_accept` to the input *context*, not to this system, and nothing keeps the same press from reaching 餐局流程. `input.md:221` exposes only `is_action_just_pressed` / `get_focus_owner` / `get_hover_owner` / `request_context` / `push_context` / `pop_context` — no context getter and no consume/mark API, so `:187`'s "把该事件标记为已消费" is unimplementable. HUD polls `ui_accept` and submits the focused dish (`in-game-hud.md:31,55,500`), never reads the context (`:183`), and its first-frame focus is already the first eatable card (`:57`, AC `:519`) — so one confirm during B0 both advances the bubble and eats a dish.
- [BLOCKING] Formulas (`:310`, `:320`, `:331`) / Edge Cases (`:347`) / AC (`:495`): `skipped = station.dish_pool \ eaten` has no defined domain. `:320` calls it "本档口完整菜池（首局为六道）", `:389` says this system never queries 数据配置, and the worked example (`:331`) picks 冰淇淋 (`unit_value = 7.00`, `entities.yaml:146`) as `best_menu` — a dish that is not one of the six and that `game-concept.md:97` says the first run does not offer. Either the pool is the run's six (example wrong: `best_menu = 麻婆豆腐`, `uv_gap = 6.00 − 3.33 = 2.67`) or it is the station's twelve (recommendation names a dish the player could not have taken). Under both readings `skipped = ∅` is unreachable in the teaching run — eating all six needs ≥ 21 cells against capacity 20 (`:294`, `:211`) — so edge case `:347` and AC 29 test an impossible state.
- [BLOCKING] Detailed Design (`:66`) / Interactions (`:124`) / Dependencies (`:359`) / UI Requirements (`:421-423`): `get_anchor_rect(anchor_id)` — the hard dependency the whole bubble layer is built on — does not exist on HUD's side. `in-game-hud.md` has no such interface; its rows offer only "焦点与卡片锚点" (`:175`, `:394`), and its Open Question 8 (`:534`) is still open and explicitly asks whether HUD should hand over a rect or an anchor id. The three 受保护矩形 the clamp needs (`:254`, `:423`) are likewise absent from HUD's interface. `:66`'s "这回答了局内 HUD Open Question 8" is a unilateral close: HUD's GDD was not updated, so the contract is one-sided.
- [BLOCKING] Detailed Design (`:33`) / Interactions (`:129`) / Dependencies (`:369`): 界面流程 never applies the preset. Core Rule 3 requires it to fetch `pool_override` + `entry_fee_override` before assembling `RunConfig`, but `ui-flow.md:55` builds the config with `pool_override = null` and never sets `entry_fee_override` (optional, so the run silently takes 85). On current main the first run would therefore use the full 12-dish pool and the full fee, and AC 1 / AC 4 fail. The preset getter is implied at `:33` but never named in the Interactions table.
- [BLOCKING] Dependencies (`:393-395`) / Open Questions (`:501-503`): the three "契约缺口" the GDD tells the orchestration session to close are already closed, and the bidirectional verification claim is false. `input.md:52,61` now has `push_context(context, allow := [])`; `run-flow.md:64` now has `entry_fee_override: int | null`; `menu-waves.md:62,69` already runs partition *and* health check on `pool_override`. OQ1/OQ2/OQ3 therefore ask for fields that exist, each gated "写 epics 之前". `:393`'s "上游 GDD 的交互表都已有…现已统一" does not hold: `run-flow.md:373` still says 未写, `ui-flow.md:161` still says "（#18，未设计）", and `audio-feedback.md` has no 首局引导 row at all.
- [BLOCKING] Formulas `first_run_pool` (`:296`): "六道覆盖 1 / 2 / 3 / 4 / 6 格" contradicts the registry. The six satieties are 6, 4, 3, 3, 2, 2 — `{2, 3, 4, 6}`; there is no 1-cell dish (the only 1-cell dish, `dish_white_rice`, is `entities.yaml:98,105-107` and is not in the pool). A block labelled 可复核 cannot carry a wrong input.

- [RECOMMENDED] Formulas (`:149`, `:159`) / States (`:102`): `onboarding_active` is defined over `run_state ∈ {Active, Ending}`, but the system enters `Closing` on `run_ended` and builds `FirstRunExplanation` only after `run_settled`, when `run_state = Settling`. Say explicitly that the formula gates in-run overlays only and that `Closing → B4` is not gated by it, or a consumer that treats it as a liveness gate will never deliver the receipt explanation.
- [RECOMMENDED] Detailed Design (`:50-51`) / rule 8 (`:60`) / `beat_next` (`:173-174`): B2 and B3 are declared complete when shown ("触发即视为完成"), but the beat table's 推进条件 column says "下一次 `action_committed`". The table reads as if the beat is waiting for the commit; the rule says it is already done. Reword the column to "展示后即完成；下一次 `action_committed` 时清掉残留提示".
- [RECOMMENDED] Detailed Design (`:54`): "每条 beat 只讲一件事…一条气泡不得同时讲两件事" is contradicted by the same paragraph's own content — B0 carries four facts (fee, capacity, turns, dish count) and B2 carries three (hard cap, table doesn't fit, waves/skip loses dishes). Either restate the rule as "每条 beat 只解决一个决定" or split B2.
- [RECOMMENDED] Formulas (`:211`): "只有吃菜数 ≤ 2、或主动跳过波次导致某一对整组消失时，B3 才可能不触发" is false for 3–4 dish runs. Eating 海鲜锅 + 糖醋里脊 + 炸鸡腿 + 麻婆豆腐 (14 cells, both pairs' second dishes dropped) is legal and triggers no `focus_enter` on a `group_count ≥ 1` dish, because no group ever reaches 2. `beats_required` already handles this correctly; it is the justification sentence that overstates.
- [RECOMMENDED] Interactions (`:119-132`) / Dependencies (`:356-374`): all six dependencies this GDD marks 硬 are marked 软 by the other side — `run-flow.md:125,298`; `in-game-hud.md:175,394`; `input.md:77,169`; `scene-flow.md:80,168,216`; `dish-values.md:90,280`; `settlement-and-entry-fee.md:106,339`. Six for six is systematic, not a typo: reconcile, and make the four contract-level rows (输入, 局内 HUD, 餐局流程, 结算) 硬 on the upstream side since the guide cannot function or degrade without them.
- [RECOMMENDED] Dependencies (`:358-363`): the status column is stale — 餐局流程 is now Approved, 局内 HUD and 结算与入场费 are Needs Revision, 输入 and 场景与流程管理 are In Review, 菜品数值 is Approved. Mark the two unstable ones explicitly, since B4's fee path and the whole anchor contract depend on GDDs under revision.
- [RECOMMENDED] Summary (`:13`): Quick reference lists five key deps; the Dependencies table has six (结算与入场费 is missing from the quick line).
- [RECOMMENDED] Dependencies (`:373`): 音效反馈 is listed as a downstream subscriber of `onboarding_beat_shown` / `onboarding_finished`, but `audio-feedback.md` has no 首局引导 row. Either add the row there or move it to "不属于依赖" alongside 存档, as `:406` already treats silence as a legal degradation.
- [RECOMMENDED] Open Questions (`:504`) / Cross-References (`:431`): OQ4 says "界面流程尚未设计", but `ui-flow.md` exists (90 KB) and this system's row is already recorded there. Replace OQ4 with the actual open item: `ui-flow.md:628` (OQ 11) asks how the guide anchors 界面流程 controls ("开始营业", receipt column, upgrade page) and remains unanswered — even if the answer is "B4 is a payload, the guide never anchors 界面流程 controls", write it down.
- [RECOMMENDED] Dependency contract, 餐局流程 (`:122`, `:358`): the interface list omits `entry_fee_override`, which is the path by which the teaching fee reaches settlement. `run-flow.md:64` assigns it to "界面流程 / 首局引导", and `run-flow.md:21,125` still name only `is_first_run` and `pool_override` — add the field on both sides.
- [RECOMMENDED] Open Questions (`:501`) / meta-upgrades (`:491`): meta-upgrades OQ 1 still recommends "首局 `entry_fee = 0` 或专用低费档口", which this GDD argues against at `:21` and `:297`. Record the answer (40, same station, no wallet special case) so the recommendation is not re-litigated; likewise settlement `:489` (OQ 4) asks how the single replacement suggestion is computed and is answered by `replacement_hint`.
- [RECOMMENDED] AC 13 (`:473`) / Open Questions: the AC names `game_skip_wave`, but `in-game-hud.md:534` (OQ 1) records that the action is still `skip_dish` in the input GDD. Say which name the test uses.
- [RECOMMENDED] Formulas (`:295`): "放弃其余任何一道都更低（79 / 78 / 76 / 81）" omits the drop-海鲜锅 case, `94 − 20 = 74`. Five alternatives exist for six dishes.

Advisory (not counted):
- Formulas (`:247`): the `W / H` range "1280×720 – 4320" mixes an aspect pair with a single number (4K is 3840×2160); and the `w` lower clamp of 280 is unreachable at any supported width (`0.36 × W ≥ 460` at 1280).
- Formulas (`:309`): `argmin effective_unit_value` uses a value the registry annotates "Never used for sorting or gate decisions" (`entities.yaml:652`, `effective_unit_value`). `replacement_hint`'s own registry entry encodes the same argmin, so this is a wording problem in the registry note rather than a design conflict — worth one clarifying clause.
- States (`:114`): `init_failed` from `Idle` → `Cancelled` — the run never started, so there is nothing to tear down. Either scope the row to states after `Intro` or note that from `Idle` it is a no-op.
- Formulas (`:342`, `:173`): B2 can be displayed on the same frame as the player's first commit (the anchor wait is 2 frames, and the fallback shows B2 at the first `action_committed`), so the capacity lesson can arrive after the decision it explains. No timer is allowed, so this may be accepted — but say so.
- Formulas (`:325`): `uv_gap ∈ [−7, 7]` is a loose superset; the MVP 12-dish pool cannot reach either bound (max static UV 7.00, min effective UV ≈ 2.86). Harmless, but state it as a domain bound rather than an observed range.

---

## Design Review: 首局引导（First-Run Onboarding）
Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge Cases, Dependencies, Acceptance Criteria. Formulas is required here because the system defines numeric rules (activation predicate, beat transition table, bubble geometry, text cap, replacement threshold). All six present.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:410-416`) carries it (no timers, one press per beat, the receipt does the talking).
- **Tuning Knobs** — absent as a heading, but the actual knobs are documented inline and marked as constants versus `OnboardingConfig` (`:86`): `gap` 8–24, `padding` 8–16, width factor 0.36 with 280–440 clamp, 1–3 lines, plus the deliberately frozen six-dish pool and 40 fee. A one-paragraph pointer section would close the advisory gap; no new values are needed.

### Dependency Graph

Every declared dependency has a GDD on disk, and the six dish ids all resolve to `station_canteen` entries.

Upstream (declared 硬 here / 软 on the other side):
- ✓ run-flow.md — exists; **Approved** — `entry_fee_override` (`:64`) and `pool_override` (`:68`) present
- ✓ in-game-hud.md — exists; **Needs Revision** — no `get_anchor_rect`, OQ 8 open
- ✓ input.md — exists; **In Review** — `allow` whitelist landed (`:52`, `:61`)
- ✓ scene-flow.md — exists; **In Review** — `clear_stack := true` on top-level switch confirmed (`:29`)
- ✓ dish-values.md — exists; **Approved** — rules 5/6/11 match the citations here
- ✓ settlement-and-entry-fee.md — exists; **Needs Revision** — `RunLog.entry_fee` in the tables, station fee still in three formula blocks (`:137`, `:143`, `:170`)

Downstream:
- ✓ ui-flow.md — exists, not yet reviewed — assembles `RunConfig` with `pool_override = null`, no `entry_fee_override` (`:55`)
- ✓ menu-waves.md — exists; `initialize(..., [, pool_override])` and overridden-pool health check present (`:62`, `:69`)
- ✓ cloying-and-taste.md — exists; rule 6 (self-group excluded from cleanse) makes the 糖醋里脊-cleanses-鲜 derivation correct (`:30`, `:131`)
- ✓ meta-upgrades.md — exists; `upgrade_stomach_capacity` L1 = 12, capacity +5 (`:277`) — the "12 元加宽胃袋" claim holds
- ✓ audio-feedback.md — exists, but **no 首局引导 row**
- ✓ save-system.md — exists; no namespace use, as claimed

### Registry Values

Searched the whole document for every registry entry name it could restate, plus the dish ids and their numeric keys.

Clean:
- Six dish prices, satieties, `cloying_group` and `unit_value` all match `entities.yaml` exactly (20/18/16/15/13/12; 6/4/3/3/2/2; 鲜/肉/肉/鲜/油/辣; 3.33/4.5/5.33/5.0/6.5/6.0).
- `first_run_entry_fee = 40` (`entities.yaml:1276`), `first_run_pool_size = 6` (`:1289`), `first_run_pool` order (`:1112`), `onboarding_active` expression (`:1095`) and `replacement_hint` expression (`:1128`) are byte-consistent with this GDD.
- `cloy_cap = 3`, `wave_size_default = 4`, `capacity_total = 20`, `station_canteen_turn_budget = 10` all match the values used.
- Arithmetic: 20 base = capacity 20; +1 鲜 +1 肉 → 22; one cleanse → 21; best five = 94 − 12 = 82; 82 − 40 = +42; 82/40 = 205.0%; three dishes = 54 → +14; four dishes = 42 → +2. All reproduce.
- `wave_count = ceil(6/4) = 2` → 4+2, matching AC 1.

Conflicts:
- `:296` — "六道覆盖 1 / 2 / 3 / 4 / 6 格"; no pool dish has satiety 1. BLOCKING item 6 above.
- `:295` — the listed alternatives (79 / 78 / 76 / 81) omit 74. Recommended.

### Required Before Implementation

1. **[BLOCKING]** Define how `ui_accept` reaches this system without also reaching 餐局流程 (`:56`, `:187`). The whitelist makes the action available in the `Blocked` context; it does not make it exclusive. Pick one mechanism and write it on both sides: input exposes `get_context()` and HUD gates `submit_action` to `Run`; or input exposes a frame-scoped `consume_action()`; or run-flow rejects submits while the context is `Blocked` (which needs a `REJECTED` code). Without one, AC 6 / 8 / 13 cannot pass.
2. **[BLOCKING]** Fix `replacement_hint`'s domain (`:310`, `:320`, `:331`): use the run's effective menu (菜单波次 partition after `pool_override`) rather than `station.dish_pool`, correct the worked example to a dish that was actually on this run's menu, and restate or remove `skipped = ∅` (`:347`, AC 29) — it is unreachable in the teaching pool.
3. **[BLOCKING]** Get `get_anchor_rect(anchor_id)` and the three protected rects onto HUD's interface (`in-game-hud.md`), and close its Open Question 8. Six anchor ids plus three rects, with the empty-rect degradation this GDD already specifies (`:421`).
4. **[BLOCKING]** Get 界面流程 to apply the preset: name the getter in the Interactions table, and change `ui-flow.md:55` from `pool_override = null` to the preset when `is_first_run`, including `entry_fee_override`.
5. **[BLOCKING]** Rewrite `:393-395` and OQ 1–3 against current main: the three gaps are closed, and the "已统一" claim must either be dropped or made true (run-flow `:373`, ui-flow `:161`, audio-feedback).
6. **[BLOCKING]** Correct `:296` to the actual satiety set.

### Recommended Revisions

1. `onboarding_active` versus the `Closing` state (`:149` vs `:102`) — scope the predicate to in-run overlays.
2. B2/B3 推进条件 versus the "展示即完成" rule (`:50-51`, `:60`, `:173-174`).
3. "一条气泡不得同时讲两件事" versus B0/B2's own content (`:54`).
4. The B3 non-trigger claim for 3–4 dish runs (`:211`).
5. Reconcile the six hard/soft mismatches (`:119-132` versus each upstream GDD).
6. Refresh the dependency status column and mark the two GDDs under revision.
7. Add 结算与入场费 to the Quick reference key-deps line (`:13`).
8. Add a 首局引导 row to `audio-feedback.md`, or move it to "不属于依赖" (`:373`, `:406`).
9. Replace OQ 4 (`:504`) with the real open item — `ui-flow.md:628` OQ 11.
10. Add `entry_fee_override` to the 餐局流程 interface row and to `run-flow.md:21,125`.
11. Record the answers to `meta-upgrades.md:491` (fee 40, not 0) and `settlement-and-entry-fee.md:489` (the hint's algorithm).
12. Resolve the `game_skip_wave` / `skip_dish` naming used in AC 13 (`in-game-hud.md:534`).
13. Complete the alternatives list at `:295` with 74.

### Nice-to-Have

See the Advisory block above: the `W/H` range (`:247`), the `effective_unit_value` sorting note, `init_failed` from `Idle`, B2's no-lead-time display, and `uv_gap`'s loose bound.

### Specialist Disagreements

None — lean mode, Phase 3b not run. Every finding above is from the main review.

### Unstable Interfaces (flagged per the batch instruction)

Three conclusions depend on GDDs that are not on stable ground today:

- **结算与入场费** is `Needs Revision`; its `RunLog.entry_fee` migration (`:64`, `:95`) is in progress while three formula blocks still read `entry_fee(station_id)` (`:137`, `:143`, `:170`). B4's `entry_fee` / `net_profit` readouts and AC 4 depend on that migration landing; the `first_run_entry_fee = 40` value itself is consistent (registry `:1276`).
- **局内 HUD** is `Needs Revision`; `get_anchor_rect` (blocking item 3) may arrive in its revision. Re-check before writing epics.
- **界面流程** exists but has not been reviewed; blocking item 4 is a change to an unreviewed document.
- **输入** is `In Review`; the `allow` whitelist has landed, so the availability half of blocking item 1 is stable — only the ownership half remains.

### Scope Signal

Rough scope signal: **L** — 6 upstream + 6 downstream systems, 7 formula blocks plus a frozen content table, a new input-context mechanism, a cross-document payload contract, and at least one ADR likely (how a `Blocked` whitelist scopes actions to one consumer). The index records `S` (`systems-index.md:214`); that is low by the review scale.

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/input.md dd15ea96523a49136b96cfef827c5619dee75e6a
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 182d446c25ab8aa50ccf7d672bc2132eeb2b22d3
