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

---

## Re-stamp — 2026-11-06 (GDD Revision Flags #10 / #13)

Approval stands. Closed `architecture-review-2026-11-05` flags #10 and #13 for this GDD: the header status now reads `Approved` (matching `systems-index.md`), and rule 5 now confirms ADR-0020's fixed-dock reading for the non-modal B2/B3 prompts (`bar:capacity` records the explanation target only; modal B0/B1 may still choose above/below by anchor). No rule, formula, AC, dependency, contract or balance value changed. Source: ADR-0020 #2/#3/#7/#8.

Reviewed-Content-Hash: design/gdd/first-run-onboarding.md a4ccdc8438300cb4892c8f3b84f5ea39b39eef83

---

## Re-Review — 2026-11-06 (ADR-0020 D2.3 plan source)

**Verdict: NEEDS REVISION**

Review mode: lean · Workflow: standard · Scope: delta of `5442a16` plus a full re-read of the current file
Blocking items: 2 | Recommended: 1
Prior verdict: APPROVED (2026-11-04, `a4ccdc8`) — **not** re-stamped; approval is invalidated.

**What the commit actually did.** `git diff 5442a16^ 5442a16 -- design/gdd/first-run-onboarding.md` is confined to two rows as claimed: the 界面流程 row in Interactions (`:135`) and the same row in 下游依赖方 (`:402`). No rule, formula, AC, balance value or content table was touched. The same commit also rewrote `design/gdd/ui-flow.md` rule 8 (`:57`), its 首局引导 dependency row (`:165`), `MainMenu` state row (`:500`), AC 6 (`:561`) and AC 47 (`:623`) — that is ui-flow's remediation of its own open NEEDS REVISION item, and ui-flow correctly remains `In Review`.

**Why it is substantive, not advisory.** § Review receipts of `.claude/rules/design-docs.md` lists a change to a "dependency" or "cross-system contract" as substantive; the two edited rows are this file's canonical statement of the 首局引导↔界面流程 interface, and they flip the contract from "UI fetches the preset" to "UI reads `FirstRunRunPlan`". Independently of that classification, the edit leaves the GDD asserting the opposite contract in its own normative rule text, which is the "inconsistent" branch: rules 1/2/3/15/22 and the Dependencies notes still put the gate and the preset lookup on the UI. The edited `:135` ("不直接读预置常量（ADR-0020 决策 2.3）") and `:402` now contradict `:33` in the same file.

Findings:

- [BLOCKING] Core Rule 3 (`:33`): "`is_first_run = true` 时，界面流程在组装 `RunConfig` 之前向本系统取一份预置" — still instructs 界面流程 to call this system for the preset directly. ADR-0020 D2.3 replaces that path with `var plan := _first_run_planner.plan()`; Implementation Guideline 1 (`adr-0020` § Implementation Guidelines) forbids `UIRoot` / `MainMenu` / `OnboardingHost` from deriving any of it. The same conflict sits in the rule-3 heading "首局预置由本系统提供、由组装方写进 `RunConfig`" (`:33`), in Core Rule 1's ownership claim "首局预置（`pool_override` 的六道菜与 `entry_fee_override = 40`）" (`:29`) and in 接口所有权 "界面流程拥有……预置的落位" (`:140`): ownership of the preset content now belongs to `FirstRunPreset` under `FirstRunPlanner`, and the UI owns only `plan` consumption. Rules 1/3 must say the preset is published as `FirstRunRunPlan`, not fetched by the UI.
- [BLOCKING] The gate formula is still duplicated as an assembler duty, in rule text rather than in the two edited rows. `:31`: "谁决定 `is_first_run` 归组装 `RunConfig` 的一方（界面流程）……**组装方的判据必须是两项同时成立**（S11）：`records.career.runs_settled == 0` **且** `LoadReport.first_run_eligible = true`"; `:80` (rule 15): "**「下一局还算不算首局」只有一个判据，由组装方按 `records.career.runs_settled == 0 ∧ LoadReport.first_run_eligible` 判定**"; `:96` (rule 22): "组装方不再标记 `is_first_run`" / "组装方也必须因 `first_run_eligible = false` 而不标记首局"; `:423`: "`is_first_run` 由界面流程组装 `RunConfig` 时决定"; `:408`: "**`is_first_run` 的三态真值表（推荐给组装方，本系统不实现）**" with `:410` "计算式：`is_first_run = (career.runs_settled == 0) ∧ LoadReport.first_run_eligible`". Each of these now reads as a UI-side computation or a requirement addressed to the assembler, contradicting `:135` ("`is_first_run` 由 `FirstRunPlanner` 唯一产出") and ui-flow rule 8 (`:57`, "不自行重算"). They must be restated as the `FirstRunGate` input spec that `FirstRunPlanner` consumes, with 界面流程 as a reader of `plan.is_first_run`.
- [RECOMMENDED] Core Rule 3 (`:42`): "界面流程在组装 `RunConfig` 时已知道 `is_first_run`，因此必须用同一个标记切换菜单上的读数……组装方必须在组装时就发现字段缺失". After D2.3 the menu reads `plan.menu_entry_fee` and the `entry_fee_override` degradation is decided planner-side (`menu_entry_fee = official_entry_fee`, `has_entry_fee_override() = false`). Point this sentence at the plan to match ui-flow rule 8 / AC 47, which already carry the three-branch treatment.

**Unaffected, verified.** `FirstRunExplanation` contract is untouched and still two-sided: rule 14 (`:78`), Visual/Audio (`:441`) and UI Requirements (`:463`) keep "两段文字 + 两个菜 id"; ui-flow `:500` still renders it; the edited rows still name the payload. No formula block (`:151`–`:360`), balance value (40 / 85 / 20 / 10 / 82 / `205.0%` / 12), content table (`:289`–`:316`) or acceptance criterion changed, and no AC depends on the old wording: AC 1 (`:491`) and AC 32 (`:537`) assert `RunConfig` values and menu/fee agreement, which the new rows satisfy. `entities.yaml` carries no key touched by this delta. AC 31 (`:536`) still matches. Cross-file agreement where the ADRs are enforced is now good: ui-flow rule 8 (`:57`), dependency row (`:165`), AC 6 (`:561`) and AC 47 (`:623`) all read `plan.menu_entry_fee` / `has_entry_fee_override()` / `run_config_fields()` and match ADR-0020 D2.3 (`adr-0020:219`–`:252`).

**Status action required (not performed here — this log was the only file modified):** set `design/gdd/first-run-onboarding.md:3` to `In Review` and `systems-index.md:75` accordingly; approval now needs a fresh lean review after rules 1/2/3/15/22 and `:408`–`:423` are rewritten to the plan source. The 2026-11-04 receipt is also stale for `design/gdd/ui-flow.md` (`:374` records `9c70925`; current bytes are `1033170`), which reinforces rather than causes this verdict.

Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 4a521ab5e825cb8b27d08afdf1f3089a5d90d21c
Reviewed-Content-Hash: design/gdd/ui-flow.md 10331708e8493cb2d1022209c086ffc55ec61f49
Reviewed-Content-Hash: design/registry/entities.yaml 5d5722fcce313d3fcbe76129bf56fd232dc80735
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 5252deab0b23680664d07fe7ce1c7255e89240ed
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43

---

## Re-Review — 2026-11-06 (round 2)

**Verdict: NEEDS REVISION**

Review mode: lean · Workflow: standard · Scope signal: L
Specialists: none (lean) · Blocking items: 3 | Recommended: 9
Prior verdict: NEEDS REVISION (2026-11-06 round 1, receipt `4a521ab5`). This run re-read all 553 lines; no Godot on this machine, so this is a documentation review only — no engine or test run is claimed.
Status: header `:3` reads `In Review` and `systems-index.md:75` reads `In Review` — they agree. `gdd-structure-check` reports the 6 sections present; Player Fantasy and Tuning Knobs are absent, both advisory at `standard`.

### Prior cited contradictions — resolved?

| Round-1 line | Claim then | Current text | Resolved |
|---|---|---|---|
| `:29` | rule 1 owned the preset, not the plan | `:29` owns `FirstRunPlanner` + `FirstRunRunPlan` (`is_first_run` / `pool_override` / `entry_fee_override` / `menu_entry_fee`) | **yes** |
| `:31` | rule 2: "组装方的判据必须是两项同时成立" | `:31` planner is the sole producer, the conjunction is its *input*, 界面流程 / 主菜单 / `OnboardingHost` "一律只消费 `plan.is_first_run`，不得各自重算" | **yes** |
| `:33` | rule 3: "界面流程…向本系统取一份预置" | `:33` "界面流程按 `plan.run_config_fields()` 组装，不直接读预置常量、不调用 `get_first_run_preset()`" | **yes** |
| `:42` | menu fee decided by the assembler | `:42` menu shows `plan.menu_entry_fee`; degradation decided by `FirstRunPlanner` at plan-assembly time | **yes** |
| `:80` | rule 15: next-run decision by 组装方 | `:80` by `FirstRunPlanner` on `runs_settled == 0 ∧ first_run_eligible` | **yes** |
| `:96` | rule 22: "组装方不再标记 `is_first_run`" | `:96` planner produces `false` on `Recovered`; "界面流程不参与这条判定" | **yes** |
| `:135` | interactions row said UI reads the preset | `:135` reads the plan only | **yes** |
| `:140` | 接口所有权: UI owned 预置的落位 | `:141` UI "只负责把 plan 字段原样写入 `RunConfig` 与菜单显示，不得另算" | **yes** |
| `:408-410` | truth table "推荐给组装方，本系统不实现" | `:410` "由 `FirstRunPlanner` 实现，界面流程只消费"; `:412` the gate formula | **yes** |
| `:423` | "`is_first_run` 由界面流程组装 `RunConfig` 时决定" | `:425` "由 `FirstRunPlanner` 唯一产出…界面流程只消费 `plan.is_first_run`" | **yes** |
| 存档 / 纪录与里程碑 rows | preset lookup attributed to UI | `:137` / `:138` are upstream read-only (`FirstRunPlanner` reads the two inputs); `:407` / `:408` restate the same reads | **yes** (but see R1 for table placement) |

**Full-file sweep.** Searched every occurrence of 界面流程 / 组装 / 自行 / 计算 / `get_first_run_preset` / `FirstRunPreset` / `is_first_run` / `menu_entry_fee` / `has_entry_fee_override()` / `pool_override` / `entry_fee_override`. **No surviving rule, table, formula, edge case, dependency row or AC assigns the gate, the fee decision or the preset lookup to 界面流程, 主菜单 or `OnboardingHost`.** The ownership rewrite is complete: `FirstRunPlanner` is the single producer in Overview `:17`, boundary `:23`, rules 1–3 (`:29`–`:42`), rules 15/22 (`:80`, `:96`), the interactions rows (`:135`, `:137`, `:138`), 接口所有权 `:141`, the truth table `:410`–`:421`, the 不属于依赖 bullet `:425`, AC 1 `:493`, AC 28 `:532`, AC 31 `:538`, AC 32 `:539`.

**Claimed inputs are legal.** `LoadReport.first_run_eligible` exists and means exactly what `:31` says — `save-system.md:52`–`:56` (rule 26: `(outcome ∈ {Fresh, Loaded, Migrated}) ∧ runs_settled == 0`, `Recovered` / `ReadOnly` / `Blocked` false), AC 31 `:431`, AC 32 `:432`, edge case `:242`. `records.snapshot().career.runs_settled` exists — `records-and-milestones.md:113` (`snapshot()` → `RecordsSnapshot`), `:69` / `:181` (`career.runs_settled`, int64). `ui-flow.md` now agrees: rule 8 `:57`, dependency row `:165`, `MainMenu` state row `:500`, AC 47 `:623`, AC 48 `:624` all read `plan.menu_entry_fee` / `has_entry_fee_override()` / `run_config_fields()` and forbid self-derivation. ADR-0020 D2.1 `:145`–`:175` (gate order, `Recovered`, `RECORDS_UNAVAILABLE`, `PRESET_INVALID`), D2.2 `:215` (`refresh()` on Boot→Menu, after Run→Menu settlement, after career deletion — matches `:33`), D2.3 `:219`–`:254` (`run_config_fields()`, menu fee source, degradation, `run_config_assembled(config, plan)`) match the plan contract; ADR-0004 `:234` registers the same handoff.

### New findings — Blocking

1. **[BLOCKING] Fee degradation contradicts the receipt rule it must degrade (`:90` rule 19, `:315`, `:366`, `:515` AC 17).** `:315` mandates that the receipt explanation state "首局入场费是教学价 40 元，正式食堂是 85 元". `:90` says on `entry_fee_override` failure only "B0 改说「这一局入场费 N 元」并省略「教学价」句，**其余 beat 不变**"; `:366` says "其余 beat 与 `B4` 不变"; AC 17 `:515` says "其余 beat 与收据讲解不变". Read together, the degraded run charges 85 while the receipt still says 40 — the exact mismatch `:42` and `ui-flow.md` AC 48 (`:624`, "不存在「85 元的行配 40 元的说明」") forbid. `ADR-0020:700` (D10 #3) and `:575` / `:623` (D7.2 `fee_note`) require `fee_note = ""` in that branch, and `ADR-0020:1077` makes it a validation case. Fix: state the condition on `:315` (`fee_is_override` / `entry_fee ≠ official_entry_fee`) and change `:90` / `:366` / AC 17 to "B4 不给教学价说明，其余 payload 不变". No AC currently asserts the *absence* of the fee note in the degraded receipt, so the degraded case is not independently covered.
2. **[BLOCKING] Rule 19's ladder omits the planner-side failures it is supposed to head (`:90`, `:299`, `:317`).** The GDD never mentions `preset_valid` or a preset-invariant failure path; `:299` only says the seven invariants are recomputed by CI and startup self-check and that a failure means "数据改动破坏了首局", and `:317` only asks for a re-run. There is no rule for preset failure, `RecordsSnapshot` unavailable, or a missing `LoadReport.first_run_eligible` field. `ADR-0020:171` (`PRESET_INVALID` → `eligible = false`, ERROR, CI red), `:216` (`plan.is_first_run = false`, `plan.preset_valid = false`, ERROR, 这一局照常开始), `:698`–`:699` (D10 rows 1–2 treat rule 19 as the authority) and the ADR's own sync table `:1022` (#15: "规则 19 / 边界用例：预置不变式失败…没有统一措辞…把 D10 写进 AC 的降级用例") all assume these branches exist here. Add them to rule 19 and give ACs for preset failure (guide off, menu 85, run starts, ERROR) and records-unavailable.
3. **[BLOCKING] `FirstRunExplanation` is still built at `run_settled` (`:108`, `:119`).** The `Closing` state row and the transition row say "`run_settled` … 构建并交付 `FirstRunExplanation`；广播 `onboarding_finished(ALL_BEATS)`". `ADR-0020:530`–`:531` (D7.1) splits the two moments — `run_settled` only stashes the result, construction happens on `Run → Menu` step 5 because the wallet and upgrade catalog are only updated by the settlement transaction (`:533`), and `ADR-0004:232` limits `run_settled` to `STASH_RESULT`. Building at `run_settled` computes the "首笔升级建议" from a pre-settlement wallet. The ADR's sync table `:1021` (#14) explicitly asks this GDD's state table to be reworded to ADR-0007; it has not been. Fix `:108` and `:119` (capture at `run_settled`, build + emit at `Run → Menu` step 5).

### New findings — Recommended

1. **`design/gdd/first-run-onboarding.md:399`** — 存档 (`:407`) and 纪录与里程碑 (`:408`) sit under the "**下游依赖方**" heading, but both rows read "`FirstRunPlanner` 读 …" and `:137` / `:138` / `:425` classify them as **上游（只读）**. Move both rows into the upstream table (or relabel the block); as written the bidirectional check asks save/records to treat onboarding as a dependent.
2. **Host input artifact is stated two ways.** `:88` (rule 18) "只消费 `RunConfig.is_first_run`", `:128` and `:389` (餐局流程 row) `RunConfig.is_first_run`, AC 1 `:493` / AC 2 `:494` — versus `:31`, `:104`, `:114`, `:135`, `:410` and the formula variable table `:162` which say `plan.is_first_run`. `ADR-0020:175` says `OnboardingHost` consumes **`plan.is_first_run` only**, and `ADR-0020:254` / `ADR-0004:234` deliver it through `run_config_assembled(config, plan)` → `onboarding.configure(plan)`, a signal this GDD never names. Pick one source and name the handoff, otherwise "same-frame B0" (AC 1) depends on undocumented signal ordering.
3. **Truth table `:416`.** The settled row lists `LoadReport.first_run_eligible = true (Loaded)` next to `runs_settled = +1`. That is only true of the *session's boot* `LoadReport`; a fresh load with `runs_settled ≥ 1` yields `false` (`save-system.md:54`, AC 31 `:431`), and `:31` itself defines the field as "`Fresh`/`Loaded`/`Migrated` 且 `runs_settled == 0`". Label the column "本次会话载入时的 `LoadReport`" or the row contradicts `:31`.
4. **`save-system.md` residue** — `:52` "供组装 `RunConfig` 的一方使用", `:99` and `:284` "界面流程 … 读取 `LoadReport.first_run_eligible` 决定 `is_first_run`", `:371` "跨文件回写（S11，由界面流程执行）". All three still assign the gate to the UI, contradicting `ADR-0020:902` (Guideline 1) and this GDD's `:31`. The field itself is unchanged and legal; the reader needs an S11 wording refresh (not edited here).
5. **`records-and-milestones.md` has no 首局引导 row at all** — no interactions row, no dependency row, no back-reference (§ Interactions and both dependency tables, `:336`–`:360`); `:113` scopes `snapshot()` to "供界面与音效读取", while `FirstRunPlanner` (Application, ADR-0020 D1) is a third reader. This GDD's `:138` / `:408` / `:425` claim a bidirectional read-only dependency that the upstream does not acknowledge.
6. **`design/gdd/first-run-onboarding.md:485`** — Cross-References says "索引已列本文件并标 `Approved`" and "仍需在 `Depends On` 补 `腻度与口味`". Both are stale: `systems-index.md:75` reads `In Review` and its Depends On cell already lists 腻度与口味 (`:75`, `:197`).
7. **`design/gdd/first-run-onboarding.md:404`** — "菜单波次（当前只读 `StationData.dish_pool`，见 Open Questions）" contradicts `:433` ("`initialize` 与健康检查已接受 `pool_override`") and OQ 2 `:546`, which is closed.
8. **`design/gdd/first-run-onboarding.md:23`** — the sentence "它压入 `Blocked` 上下文时改变的只是「这一帧玩家能按什么」，不是「这一帧游戏算什么」。" appears twice in the same paragraph (rewrite residue).
9. **Rule 19 `:90`** — the fee branch says "本局按 85 元入场" but neither the menu's 85 nor the required WARNING; both appear only at `:42` / `:366` / AC 32 `:539`. Since rule 19 is the ladder the ADR cites (`ADR-0020:694`), restate all three in the rule.

### Registry Values
No registry key touched by `cafff91`. Spot-checked `onboarding_active` (`entities.yaml:1113`), `first_run_entry_fee` (`:1300`), `first_run_pool_size` (`:1313`) against `:156`, `:38`, `:490` — unchanged and consistent. Balance values 40 / 85 / 20 / 10 / 82 / +42 / 205.0% / 12 are untouched by the rewrite and still arithmetic-consistent.

### Dependency Graph
All 13 named GDDs exist on disk (run-flow, in-game-hud, input, scene-flow, dish-values, settlement-and-entry-fee, menu-waves, cloying-and-taste, meta-upgrades, audio-feedback, save-system, records-and-milestones, ui-flow) plus game-concept and systems-index. No broken reference.

### Verdict: NEEDS REVISION

Approval is not re-stamped; the 2026-11-04 receipt remains invalid. No files other than this log were modified — items B1–B3 and R1–R9 are for the GDD author; R4 and R5 belong to `save-system.md` / `records-and-milestones.md`. `systems-index.md` already reads `In Review` and needs no change for this round.

Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 164209b9a8b290ce0add49761fc59138696822a7
Reviewed-Content-Hash: design/gdd/ui-flow.md 05d5a22e1e6376e45b9863166f8dd32310e7f006
Reviewed-Content-Hash: design/gdd/save-system.md a7d54e3ba4d7f8a23be1d28a6c165bb422648abd
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 6964692c5d4850582e014c362342bcdb2ea0b28b
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 5252deab0b23680664d07fe7ce1c7255e89240ed
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/registry/entities.yaml 0646d0161ee8488d78ec5454abe3a61d0d0a14f9
Reviewed-Content-Hash: design/gdd/systems-index.md 4714602ef782db915793ed55ee8f3c1f089ee886
