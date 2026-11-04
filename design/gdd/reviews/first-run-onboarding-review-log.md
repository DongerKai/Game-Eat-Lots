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

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 3 | Recommended: 13
Summary: The teaching economy still reproduces exactly — base satiety 20 = capacity 20, six dishes need 21–22 cells, best five 82 against fee 40 for +42 / 205.0%, and the 12-yuan stomach upgrade is affordable. Four of the six prior blockers are fully closed: HUD now exposes `get_anchor_rect` and `get_protected_rects()` (OQ 8 closed), 界面流程 applies `get_first_run_preset()` with both overrides, the three named contract gaps have landed, and the satiety set is corrected to `{2,3,4,6}`. Two blockers are only partially closed: the `ui_accept` ownership fix specifies `consume_action` for B1 only and still attributes exclusivity to the `Blocked` whitelist in the edge cases, so a B0 confirm can still reach HUD and eat the first dish; and `replacement_hint` now names `run_menu` but 菜单波次 exposes no interface that returns the run's full effective menu, while this GDD's own 菜单波次 row says the guide never reads more than the current wave. A new downstream gap also remains: 界面流程 has no `FirstRunExplanation` (or `onboarding_finished`) receiving contract anywhere in its receipt spec, so B4's core output has no landing.
Prior verdict resolved: No — 4 of 6 resolved, 2 partially (see mapping below)
Suggested index status: `18 | 首局引导 | Meta | Vertical Slice | Needs Revision | design/gdd/first-run-onboarding.md | 餐局流程, 局内 HUD, 输入, 场景与流程管理, 菜品数值, 结算与入场费`
Note: this GDD may not be edited in this session and `entities.yaml` may not be edited; the registry item below is for the orchestration session.

Prior-blocker resolution:
- **#1 `ui_accept` ownership — PARTIAL.** `consume_action` now exists (`input.md:54`, API AC `:222`) and B1 calls it (`:187`). But the mechanism is scoped to "B1 的模态分支"; B0 advances on `ui_accept` (`:48`, `:171`) and is never said to consume it. HUD's first-frame focus is the first eatable card (`in-game-hud.md:57`, AC 35), and HUD submits the focused dish on every `ui_accept` without reading the context (`:55`, AC 22; `:183`), so a B0 confirm can eat a dish. Edge case `:340` still asserts "吃菜在 `Blocked` 下本就到不了餐局流程" — the availability-is-not-exclusivity rationale the first review rejected. AC 6 checks `turns_remaining` / `capacity_used` (eating costs 1 turn, `time-and-turns.md:19`, so the bug is test-detectable) but no AC states B0 consumes the confirm.
- **#2 `replacement_hint` domain — PARTIAL.** The domain is now `run_menu` (`:310`, `:320`), the example is corrected to 麻婆豆腐 with `uv_gap = 6.00 − 3.33 = 2.67` (`:331`), and `skipped = ∅` is labelled an unreachable defensive branch (`:327`, `:347`). But `run_menu` is sourced from 菜单波次 (`:320`) while `menu-waves.md` exposes only `visible_dishes` / `preview_dishes` / wave index (`:62-63`), never the full run menu; and this GDD's own 菜单波次 row (`:127`) says "本系统不读波次内部，只读当前波的可见数与波序" — the two statements contradict each other. In the `pool_override`-ignored degradation path the guide also has no way to learn the real pool (it does not query 数据配置, `:389`).
- **#3 HUD anchor contract — RESOLVED.** `in-game-hud.md:175,394` now provide `get_anchor_rect(anchor_id) -> Rect2`, `get_protected_rects() -> Rect2[3]`, and OQ 8 is closed (`:541`).
- **#4 界面流程 applies the preset — RESOLVED.** `ui-flow.md:57` fetches `首局引导.get_first_run_preset()` and sets both `pool_override` and `entry_fee_override`; AC at `:556` verifies it.
- **#5 Stale contract claims — RESOLVED for the three named gaps.** `input.md:52,54` (`allow`, `consume_action`), `run-flow.md:65` (`entry_fee_override`), `menu-waves.md:62,69,176-184` (`pool_override` in partition and health check) all landed. Residual bidirectional claims are listed as recommended items 3–5.
- **#6 Satiety set — RESOLVED.** `:296` now reads `2 / 3 / 4 / 6`, matching the registry.

Findings:
- [BLOCKING] Formulas `beat_next` (`:187`) / Detailed Design rule 6 (`:56`) / Edge Cases (`:340`) / AC 6 (`:463`): `consume_action` is specified only for B1. B0 also advances on `ui_accept`, and no rule consumes that confirm; HUD submits its focused (first, eatable) card on `ui_accept` without consulting the context. `:340` still explains exclusivity by the `Blocked` whitelist rather than by consumption. State that every modal beat consumes its advance action before advancing (B0 and B1), and rewrite `:340`.
- [BLOCKING] Formulas `replacement_hint` (`:310`, `:320`) / Interactions 菜单波次 (`:127`): `run_menu` has no provider. `menu-waves.md` exposes `visible_dishes`, `preview_dishes`, skip cost and wave index — not the run's full effective menu — while this GDD's own row says the guide reads only the current wave. Add the getter on the 菜单波次 side (e.g. `get_run_menu() -> StringName[]`) and update both rows, or redefine `skipped` against a provider that exists.
- [BLOCKING] Detailed Design rule 14 (`:72`) / Interactions 界面流程 (`:129`) / Dependencies (`:369`) / OQ 4 (`:504`): 界面流程 has no `FirstRunExplanation` receiving contract. `ui-flow.md` names `首局引导` (`:163`, `:442`, `:556`) but its receipt spec (`:63`, `:497`) has no explanation block, no replacement hint, no first-upgrade suggestion, and no teaching-price disclaimer ("正式食堂是 85 元", required by `:298`); it also never subscribes `onboarding_finished`. Close OQ 4 against the now-written `ui-flow.md` and define the receiving side, or narrow the claim.

- [RECOMMENDED] Registry (`entities.yaml:1128-1134`): the `replacement_hint` entry still lists `station.dish_pool` and not `run_menu`. The GDD is now right and the owned registry entry is stale; regenerate it (orchestration session — `entities.yaml` is out of scope here).
- [RECOMMENDED] Core Rule 3 (`:33`) / Interactions (`:129`): this GDD never declares `get_first_run_preset()`, although `ui-flow.md:57,163` calls it by that name. Name the getter in this system's own interface so the contract is two-sided.
- [RECOMMENDED] Interactions (`:129`) / Dependencies (`:369`): the GDD says 界面流程 reads `onboarding_finished` to decide the next `is_first_run`, but `ui-flow.md:57` computes `is_first_run = career.runs_settled == 0` and never reads the signal. Describe the actual path (records → `runs_settled`) or make the signal the contract.
- [RECOMMENDED] Interactions (`:132`) / Dependencies (`:373`) versus Cross-References note (`:394`) / Visual-Audio (`:406`): 音效反馈 is still listed as a downstream subscriber of `onboarding_beat_shown` / `onboarding_finished`, while `:394` correctly says `audio-feedback.md` has no 首局引导 row and `:406` treats silence as a legal degradation. Move it to "不属于依赖" or add the row there.
- [RECOMMENDED] Interactions (`:119-132`) / Dependencies (`:356-374`): the six 硬 rows are still 软 on the upstream side — `run-flow.md:126`, `in-game-hud.md:175,394`, `input.md:78`, `scene-flow.md`, `dish-values.md`, `settlement-and-entry-fee.md:339`. Reconcile, at least for the four contract-level rows (输入, 局内 HUD, 餐局流程, 结算).
- [RECOMMENDED] Open Questions (`:504`): OQ 4's premise "界面流程尚未设计" is false — `ui-flow.md` exists and is 90 KB. The real open item is its OQ 11 (`ui-flow.md:632`): how the guide anchors 界面流程 controls. Answer it (even if the answer is "B4 is a payload; the guide anchors no 界面流程 control").
- [RECOMMENDED] Quick reference (`:13`): key deps list five systems; the Dependencies table has six. 结算与入场费 is still missing.
- [RECOMMENDED] Beat table (`:50-51`) / rule 8 (`:60`) / `beat_next` (`:173-174`): B2/B3's 推进条件 says "下一次 `action_committed`，或 `run_ended`", but rule 8 says they are complete the moment they are shown. Reword to "展示后即完成；下一次 `action_committed` 时清掉残留提示".
- [RECOMMENDED] Detailed Design (`:54`): "每条 beat 只讲一件事…一条气泡不得同时讲两件事" is still contradicted by its own content — B0 carries four facts (fee, capacity, turns, dish count) and B2 carries three (hard cap, table does not fit, waves/skip loses dishes). Restate as "each beat resolves one decision" or split B2.
- [RECOMMENDED] Formulas (`:211`): "只有吃菜数 ≤ 2 … B3 才可能不触发" remains false for 3–4 dish runs. 海鲜锅 + 糖醋里脊 + 炸鸡腿 (the 酸 cleanses 鲜) never reaches a `group_count ≥ 1` focus, so B3 does not trigger at three dishes. `beats_required` already handles it; only the justification overstates.
- [RECOMMENDED] Formulas (`:295`): the drop alternatives "(79 / 78 / 76 / 81)" still omit dropping 海鲜锅, `94 − 20 = 74`. Five alternatives exist for six dishes.
- [RECOMMENDED] AC 29 (`:495`): the test exercises `skipped = ∅`, which `:327` now correctly marks unreachable in the teaching run. Reframe it as a unit-level defensive test rather than a playthrough AC.
- [RECOMMENDED] Dependencies status column (`:358-363`): "GDD 已写" mixes in review states for two rows (`In Review` for scene-flow, `Approved` for dish-values) and asserts nothing for the others. Mark the unstable ones explicitly, since B4's fee path and the anchor contract still depend on GDDs under revision.

Advisory (not counted):
- Formulas (`:247`): the `W / H` range "1280×720 – 4320" still mixes an aspect pair with a single number (4K is 3840×2160), and `w`'s lower clamp of 280 is unreachable at any supported width (`0.36 × 1280 = 460.8`).
- Formulas (`:309`): `argmin effective_unit_value` uses a value the registry annotates "Never used for sorting or gate decisions". The registry's own `replacement_hint` entry encodes the same argmin, so this is a registry wording problem; one clarifying clause would close it.
- States (`:114`): `init_failed` from `Idle` → `Cancelled` — the run never started, so there is nothing to tear down. Scope the row to states after `Intro` or call it a no-op.
- Formulas (`:342`, `:173`): B2 can display on the same frame as the player's first commit (2-frame anchor wait, then the first `action_committed` fallback), so the capacity lesson can arrive after the decision it explains. No timer is allowed, so this may be accepted — but say so.
- Formulas (`:325`): `uv_gap ∈ [−7, 7]` is a loose domain bound; the MVP pool cannot reach either extreme (max static UV 7.00, min effective UV ≈ 2.86). State it as a bound rather than an observed range.

### Design Review: 首局引导（First-Run Onboarding）
Specialists consulted: none (lean mode)
Re-review: Yes — prior verdict was NEEDS REVISION on 2026-11-04

#### Completeness: 6/6 required sections present
`modes.workflow: standard`. Required: Overview, Detailed Design, Formulas, Edge Cases, Dependencies, Acceptance Criteria — all present. Formulas is required because the system defines numeric rules (activation predicate, beat transitions, bubble geometry, text cap, replacement threshold). Player Fantasy and Tuning Knobs remain advisory at `standard`; both are carried inline (`## Game Feel`; geometry knobs at `:86`).

#### Dependency Graph
Every declared dependency has a GDD on disk, and all six dish ids resolve to `station_canteen` entries.
- ✓ run-flow.md — Approved — `entry_fee_override` (`:65`), `pool_override` (`:69`)
- ✓ in-game-hud.md — `get_anchor_rect` + `get_protected_rects()` present (`:175`, `:394`); OQ 8 closed
- ✓ input.md — `allow` (`:52`) and `consume_action` (`:54`); hard/soft still 软 (`:78`)
- ✓ scene-flow.md — `clear_stack := true` on top-level switch
- ✓ dish-values.md — Approved — rules 5/6/11 match the citations
- ✓ settlement-and-entry-fee.md — `RunResult` carries every field B4 needs (`:83`)
- ✓ ui-flow.md — applies the preset (`:57`); **no `FirstRunExplanation` receiver** (blocking item 3)
- ✓ menu-waves.md — `pool_override` in `initialize` and health check (`:62`, `:69`); **no full-menu getter** (blocking item 2)
- ✓ cloying-and-taste.md / meta-upgrades.md / save-system.md — as claimed
- ✓ audio-feedback.md — exists, **no 首局引导 row** (recommended item 4)

#### Registry Values
Searched the whole document for all 78 `entities.yaml` entry names plus the numeric keys they carry. Clean:
- Six dish prices/satieties/groups/UVs match exactly (20/18/16/15/13/12; 6/4/3/3/2/2; 鲜/肉/肉/鲜/油/辣; 3.33/4.5/5.33/5.0/6.5/6.0), and all six are `station_canteen`.
- `first_run_entry_fee = 40`, `first_run_pool_size = 6`, `first_run_pool` order, `onboarding_active` expression are byte-consistent.
- `cloy_cap = 3`, `wave_size_default = 4`, `capacity_total = 20`, `station_canteen_turn_budget = 10` all match.
- Arithmetic re-reproduces: 20 = capacity; +1 鲜 +1 肉 → 22; one cleanse → 21; best five `94 − 12 = 82`; `82 − 40 = +42`; `82/40 = 205.0%`; three dishes `54 → +14`; two dishes lose; `ceil(6/4) = 2` → 4+2.
Conflicts:
- `entities.yaml:1128-1134` — `replacement_hint` still lists `station.dish_pool` rather than `run_menu`. Registry is stale against its owning GDD (recommended item 1).

#### Required Before Implementation
1. **[BLOCKING]** Extend the `consume_action` rule to every modal beat: B0's `ui_accept` must be consumed before advancing, and `:340`'s "Blocked 下本就到不了" rationale must be replaced with the consumption mechanism. Add `eaten_count` to AC 6.
2. **[BLOCKING]** Give `run_menu` a provider: add a full-effective-menu getter to 菜单波次 and update this GDD's 菜单波次 row, or redefine `skipped` against a provider that exists. The current Formulas row and Interactions row contradict each other.
3. **[BLOCKING]** Define 界面流程's receiving side for `FirstRunExplanation` (including the teaching-price disclaimer) and `onboarding_finished`, and close OQ 4 against the now-written `ui-flow.md`.

#### Recommended Revisions
Items 1–13 above. The highest-leverage are the registry sync (1), the missing `get_first_run_preset()` declaration (2), the `onboarding_finished` claim (3), and the audio-feedback contradiction (4).

#### Specialist Disagreements
None — lean mode, Phase 3b not run. Every finding is from the main review.

#### Unstable Interfaces
- **界面流程** — no B4 contract; blocking item 3 is a change to a now-written but unreviewed document.
- **菜单波次** — `pool_override` landed, but the menu getter blocking item 2 needs does not exist.
- **结算与入场费 / 局内 HUD / 输入** — all under revision; re-check the fee path, the anchor rects, and the consume semantics before writing epics.

#### Scope Signal
Rough scope signal: **L** — 6 upstream + 6 downstream systems, 7 formula blocks plus a frozen content table, a new input-consumption mechanism, and two cross-document payload contracts. `systems-index.md:214` still records `S`; that is low by the review scale.

### Verdict: NEEDS REVISION

Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 5734623b4a60d9072657ef631c8ae61f05359f21
Reviewed-Content-Hash: design/gdd/menu-waves.md ce532d3265df1ed7555ba5aa46383be35bbee4a1
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/gdd/save-system.md 772ed596d68eb5cbae207b7bac9a170ec55c1524
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 0c3efa52308253f2243c0771bff7c9b6d3ae5933

---

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 0 | Recommended: 15
Summary: All three prior blockers are resolved. Both modal beats now consume their advance action before advancing (`:187`), 菜单波次 exposes `get_run_menu() -> StringName[]` (`menu-waves.md:62`) and this GDD consumes it (`:127`, `:320`), and 界面流程 receives `FirstRunExplanation` — teaching-price disclaimer included — and subscribes `onboarding_finished` (`ui-flow.md:163`, `:497`). The teaching economy still reproduces exactly against the registry, the design still has no timers anywhere, and the degradation ladder still never blocks a run. What remains is documentation drift — a whitelist-versus-consumption phrase in rule 6, provider-side rows that do not name the new getter, and stale cross-references — none of which changes what an implementer must build.
Prior verdict resolved: Yes — 3/3 blocking resolved

Prior-blocker resolution:
- **#1 `ui_accept` ownership — RESOLVED.** `:187` now states that both B0 and B1 call `输入.consume_action(ui_accept)` before advancing, so 餐局流程 and HUD poll `false`; edge case `:340` is rewritten around consumption rather than whitelist availability; AC 6 (`:463`) now asserts `eaten_count` is unchanged. Residual wording only: rule 6 (`:56`) still credits the whitelist as the mechanism (recommended 1).
- **#2 `run_menu` provider — RESOLVED.** `menu-waves.md:62` exposes `get_run_menu() -> StringName[]`（本局有效菜单）; this GDD lists it in the 菜单波次 row (`:127`) and uses it as `replacement_hint`'s domain (`:310`, `:320`). Residuals: the provider's own 首局引导 row (`menu-waves.md:66`, `:228`) still says only "教学数据", and the getter's call window at B4 is not stated (recommended 2–3).
- **#3 界面流程 receiving side — RESOLVED.** `ui-flow.md:163` records `FirstRunExplanation` + `onboarding_finished`, and `:497` requires the receipt to render the teaching-price disclaimer（「首局 40 元，正式食堂 85 元」）, the replacement hint and the first-upgrade suggestion; OQ 4 is closed (`:504`).

Findings:
- [RECOMMENDED] Core rule 6 (`:56`) / 输入 row (`:123`): rule 6 still explains exclusivity as "`Blocked` 白名单把 `ui_accept` 交给本系统", and the 输入 interaction row lists only the query actions, not `consume_action`. Use the consumption mechanism in rule 6 and add `consume_action(action) -> bool` to the row.
- [RECOMMENDED] Provider side, 菜单波次 (`menu-waves.md:66`, `:228`): the 首局引导 row still says only "教学数据"; `get_run_menu()` appears on the 餐局流程 row instead. Name the getter on the 首局引导 row so both sides state the same contract. `run-flow.md` does not declare `get_run_menu()` either (menu-waves review r3, finding 6) — harmless for a direct menu-waves call, but the ownership should be stated once.
- [RECOMMENDED] B4 timing (`:320` → `menu-waves.md:52`): menu-waves discards its partition on 餐局结束, while B4 builds `FirstRunExplanation` on `run_settled`. State when the guide reads `run_menu` (capture during `Active`, or confirm the getter survives until `Handoff`). The design is sound only under the reading that menu-waves clears state at `Handoff → Idle`.
- [RECOMMENDED] Registry (`entities.yaml:1131-1139`): `replacement_hint.variables` still lists `station.dish_pool` rather than `run_menu`. The GDD is right and the registry entry is stale; regenerate it (orchestration session — `entities.yaml` is out of scope here).
- [RECOMMENDED] Core Rule 3 (`:33`) / Interactions (`:129`): this GDD never declares `get_first_run_preset()`, although `ui-flow.md:57,163` calls it by that name. Name it in this system's own interface so the contract is two-sided.
- [RECOMMENDED] Interactions (`:129`) / Dependencies (`:369`): the GDD says 界面流程 reads `onboarding_finished` to decide the next `is_first_run`, but `ui-flow.md:57` computes `is_first_run = career.runs_settled == 0`. Describe the actual path (records → `runs_settled`) or make the signal the contract.
- [RECOMMENDED] Interactions (`:132`) / Dependencies (`:373`) versus (`:394`): 音效反馈 is still listed as a downstream subscriber of `onboarding_beat_shown` / `onboarding_finished`, while `:394` correctly says `audio-feedback.md` has no 首局引导 row and `:406` treats silence as a legal degradation. Move it to "不属于依赖" or add the row there.
- [RECOMMENDED] Interactions (`:119-132`) / Dependencies (`:356-374`): the six 硬 rows are still 软 on the upstream side — `run-flow.md:127,302`; `in-game-hud.md:175,394`; `input.md:78`; `scene-flow.md:81,169`; `dish-values.md:90,280`; `settlement-and-entry-fee.md:106,339`. Reconcile, at least for the four contract-level rows (输入, 局内 HUD, 餐局流程, 结算).
- [RECOMMENDED] Quick reference (`:13`): key deps list five systems; the Dependencies table has six. 结算与入场费 is still missing.
- [RECOMMENDED] Beat table (`:50-51`) / rule 8 (`:60`) / `beat_next` (`:173-174`): B2/B3's 推进条件 says "下一次 `action_committed`，或 `run_ended`", but rule 8 says they complete the moment they are shown. Reword to "展示后即完成；下一次 `action_committed` 时清掉残留提示".
- [RECOMMENDED] Detailed Design (`:54`): "每条 beat 只讲一件事…一条气泡不得同时讲两件事" is still contradicted by its own content — B0 carries four facts (fee, capacity, turns, dish count) and B2 carries three (hard cap, table does not fit, waves/skip loses dishes). Restate as "each beat resolves one decision" or split B2.
- [RECOMMENDED] Formulas (`:211`): "只有吃菜数 ≤ 2 … B3 才可能不触发" remains false for 3–4 dish runs. 海鲜锅 + 糖醋里脊 + 炸鸡腿 (the 酸 cleanses 鲜) never reaches a `group_count ≥ 1` focus. `beats_required` already handles it; only the justification overstates.
- [RECOMMENDED] Formulas (`:295`): the drop alternatives "(79 / 78 / 76 / 81)" still omit dropping 海鲜锅, `94 − 20 = 74`. Five alternatives exist for six dishes.
- [RECOMMENDED] AC 29 (`:495`): the test exercises `skipped = ∅`, which `:327` now correctly marks unreachable in the teaching run. Reframe it as a unit-level defensive test rather than a playthrough AC.
- [RECOMMENDED] Dependencies (`:358-363`, `:370`, `:393`) / `ui-flow.md:633`: the status column still mixes "GDD 已写" with review states; `:370` still says menu-waves "当前只读 `StationData.dish_pool`" though `:127`/`:502` and `menu-waves.md:62,69` now handle `pool_override`; `:393` says `run-flow.md` already points back, but `run-flow.md:377` still reads 未写. Refresh all three, and answer `ui-flow.md:633` (OQ 11) on the ui-flow side — the implicit answer is already in `:504`.

Advisory (not counted):
- Formulas (`:247`): the `W / H` range "1280×720 – 4320" mixes an aspect pair with a single number (4K is 3840×2160); `w`'s lower clamp of 280 is unreachable at any supported width (`0.36 × 1280 = 460.8`).
- Formulas (`:309`): `argmin effective_unit_value` uses a value the registry annotates "Never used for sorting or gate decisions"; one clarifying clause would close it.
- States (`:114`): `init_failed` from `Idle` → `Cancelled` — the run never started, so there is nothing to tear down. Scope the row to states after `Intro` or call it a no-op.
- Formulas (`:342`, `:173`): B2 can display on the same frame as the player's first commit (2-frame anchor wait, then the first `action_committed` fallback), so the capacity lesson can arrive after the decision it explains. No timer is allowed; say the late display is accepted.
- Formulas (`:325`): `uv_gap ∈ [−7, 7]` is a loose domain bound; the MVP pool cannot reach either extreme. State it as a bound rather than an observed range.

---

### Design Review: 首局引导（First-Run Onboarding）
Specialists consulted: none (lean mode)
Re-review: Yes — prior verdict was NEEDS REVISION on 2026-11-04

#### Completeness: 6/6 required sections present
`modes.workflow: standard`. Required: Overview, Detailed Design, Formulas, Edge Cases, Dependencies, Acceptance Criteria — all present. Formulas is required because the system defines numeric rules (activation predicate, beat transitions, bubble geometry, text cap, replacement threshold). Player Fantasy and Tuning Knobs remain advisory at `standard`; both are carried inline (`## Game Feel`; geometry knobs at `:86`).

#### Dependency Graph
Every declared dependency has a GDD on disk, and all six dish ids resolve to `station_canteen` entries.
- ✓ run-flow.md — Approved — `entry_fee_override` (`:66`), `pool_override` (`:70`); its 首局引导 row (`:377`) still says 未写
- ✓ in-game-hud.md — `get_anchor_rect` + `get_protected_rects()` present (`:175`, `:394`); OQ 8 closed
- ✓ input.md — `allow` (`:52`) and `consume_action` (`:54`); hard/soft still 软 (`:78`)
- ✓ scene-flow.md — `clear_stack := true` on top-level switch
- ✓ dish-values.md — Approved — rules 5/6/11 match the citations
- ✓ settlement-and-entry-fee.md — `RunResult` carries every field B4 needs (`:83`)
- ✓ ui-flow.md — applies the preset (`:57`); receives `FirstRunExplanation` + `onboarding_finished` (`:163`, `:497`)
- ✓ menu-waves.md — `pool_override` in `initialize` and health check (`:62`, `:69`); `get_run_menu()` present but attributed to the 餐局流程 row
- ✓ cloying-and-taste.md / meta-upgrades.md / save-system.md — as claimed
- ✓ audio-feedback.md — exists, **no 首局引导 row** (recommended 7)

#### Registry Values
Searched the whole document for all `entities.yaml` entry names plus the numeric keys they carry. Clean:
- Six dish prices/satieties/groups/UVs match exactly (20/18/16/15/13/12; 6/4/3/3/2/2; 鲜/肉/肉/鲜/油/辣; 3.33/4.5/5.33/5.0/6.5/6.0), and all six are `station_canteen`.
- `first_run_entry_fee = 40` (`:1279`), `first_run_pool_size = 6` (`:1292`), `first_run_pool` order (`:1115`), `onboarding_active` expression (`:1098`) are byte-consistent.
- `cloy_cap = 3`, `wave_size_default = 4`, `capacity_total = 20`, `station_canteen_turn_budget = 10` all match.
- Arithmetic re-reproduces: 20 = capacity; +1 鲜 +1 肉 → 22; one cleanse → 21; best five `94 − 12 = 82`; `82 − 40 = +42`; `82/40 = 205.0%`; three dishes `54 → +14`; two dishes lose; `ceil(6/4) = 2` → 4+2.
- The corrected example holds: `best_menu = 麻婆豆腐` (6.00) − `worst_eaten = 海鲜锅` (3.33) = `2.67`.
Conflicts:
- `entities.yaml:1131-1139` — `replacement_hint` still lists `station.dish_pool` rather than `run_menu`. Registry is stale against its owning GDD (recommended 4).

#### Required Before Implementation
- none — the three prior blocking items are resolved.

#### Recommended Revisions
Items 1–15 above. The highest-leverage are the rule-6/input-row consumption wording (1), the provider-side `get_run_menu` row (2), the B4 read window (3), and the registry sync (4).

#### Specialist Disagreements
None — lean mode, Phase 3b not run. Every finding is from the main review.

#### Unstable Interfaces
- **菜单波次** — `get_run_menu()` landed, but its row attribution and call window are unresolved (recommended 2–3).
- **结算与入场费 / 局内 HUD / 输入** — all under revision; re-check the fee path, the anchor rects, and the consume semantics before writing epics.

#### Scope Signal
Rough scope signal: **L** — 6 upstream + 6 downstream systems, 7 formula blocks plus a frozen content table, a new input-consumption mechanism, and two cross-document payload contracts. `systems-index.md:214` still records `S`; that is low by the review scale.

### Verdict: APPROVED

Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 1cafff5e7ad8aab8912e716d98b69cea02cbbc43
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/menu-waves.md 4e4854fb6a38c7b0989503d8c261b69c5746adbe
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/audio-feedback.md ffb1c54a84af1da89b0228395ee86e89b3e87e30
Reviewed-Content-Hash: design/gdd/ui-flow.md 9c709252b4c631e8fb50d8a2f35a8fb394f230d7
Reviewed-Content-Hash: design/gdd/save-system.md f22c2228d60be43669125831bccea834e94d6026
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 12d2e7c5b8de635a090b0d1f4589dcab58c729f8
