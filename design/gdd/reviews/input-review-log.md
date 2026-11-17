# Input (输入) — Design Review Log

## Review — 2026-11-03 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 5 | Recommended: 7
Summary: The core design is sound — the action set, context machine, dual-focus split and both formulas are precise and engine-correct for Godot 4.7. Five contract-level contradictions must be resolved before implementation: the `Blocked` context has no whitelist (which hard-blocks 首局引导), Esc's dual binding is ambiguous in `Paused`, AC 11 contradicts the context/focus API, `game_skip_dish` means "skip hovered dish" here but "skip wave" downstream, and `push_context` on an equal top value is undefined (blocking the two-level modal).
Prior verdict resolved: First review

Findings:
- [BLOCKING] Detailed Design / Dependencies: `Blocked` allows only `ui_pause` and `push_context(context)` has no `allow` parameter, but 首局引导 hard-depends on `push_context(Blocked, allow=[…])`.
- [BLOCKING] Detailed Design (rule 1 table vs rule 2 vs States): Esc is claimed bound to both `ui_cancel` and `ui_pause`, the mapping table lists it only under `ui_pause`, and `Paused` allows both actions with no precedence — pressing Esc in `Paused` is undefined.
- [BLOCKING] Acceptance Criteria 11: says only `is_action_just_pressed` is exposed, contradicting rule 12 (`request_context` / `push_context` / `pop_context`), AC 18–20, and the focus/hover queries 界面流程 and HUD require.
- [BLOCKING] Core Rules / Edge Cases: `game_skip_dish` is defined as "skip the hovered dish card" and no-ops without hover, while 餐局流程 / 菜单波次 / HUD use it as "skip the current wave / end pickup" with no hover requirement.
- [BLOCKING] Detailed Design (rule 12): `push_context` on a stack top already holding the same context is undefined ("空操作（同值覆盖）不发信号"), blocking 界面流程's two-level `Modal` stack.
- [RECOMMENDED] Acceptance Criteria 3: `focus_owner` is used but never defined, and no focus-query API is specified.
- [RECOMMENDED] Acceptance Criteria 15: cites a target frame rate in `project.yaml`, which has no performance/target-FPS block.
- [RECOMMENDED] Acceptance Criteria 9: the `device ==` static scan would also flag the engine-recommended `device == InputEvent.DEVICE_ID_MOUSE` comparison; the intended pattern is `device == 0`.
- [RECOMMENDED] Cross-References: claims "None — this system references no other GDD" and that reverse links are still to be added, but all six downstream GDDs are written and `scene-flow.md` is cited in Edge Cases.
- [RECOMMENDED] Dependencies: 音效反馈 is listed as a soft downstream consumer of action events, while `audio-feedback.md` explicitly states it does not subscribe to raw input.
- [RECOMMENDED] Formulas / Registry: `nav_steps` is unbounded in the GDD (1–∞) but `entities.yaml` records `[1, 999]` with a clamp note.
- [RECOMMENDED] Detailed Design: same-frame precedence between different actions (`ui_accept` + `ui_cancel`) is undefined, though `ui-flow.md` cites "输入 规则 2" for it.

---

## Design Review: 输入（Input）
Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules, Edge Cases,
Dependencies, Acceptance Criteria — plus Formulas, because this system defines
numeric rules (navigation repeat rate, analog-stick thresholds). All six are
present; `## Detailed Design` is accepted as the Detailed Rules section.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading. The `## Game Feel` section covers
  most of the intended feeling (immediacy, predictability, no punishment), so
  this is a note, not a gap.
- **Tuning Knobs** — absent as a heading, but the content exists as the
  `### 调参起点` table inside Formulas. Giving it a `## Tuning Knobs` heading
  would let the structure check and downstream skills find it.

### Dependency Graph

Upstream: none declared. ✓

Downstream:
- ✓ `design/gdd/ui-flow.md` — exists
- ✓ `design/gdd/run-flow.md` — exists
- ✓ `design/gdd/in-game-hud.md` — exists
- ✓ `design/gdd/first-run-onboarding.md` — exists
- ✓ `design/gdd/audio-feedback.md` — exists
- ✓ `design/gdd/scene-flow.md` — exists (path missing in the Dependencies status cell)

All six downstream GDDs list 输入 as an upstream dependency, except
`audio-feedback.md`, which explicitly declares it is **not** a dependency.

### Registry Values

Registry entries owned by this system: `nav_steps`, `stick_engaged`.
Searched the whole document for `nav_steps`, `stick_engaged`, `repeat_delay`,
`repeat_interval`, `stick_engage`, `stick_release` (case-insensitive).

- `nav_steps` expression and defaults (D=0.35, I=0.12) match the registry. The
  output range does not: GDD says `1–∞`, registry says `[1, 999]` with a
  practical clamp note. See Recommended item 6.
- `stick_engaged` expression, defaults (E=0.60, R=0.40) and the
  `R ≤ E − 0.10` constraint match the registry.
- No other entity-value conflicts found.

### Required Before Implementation

1. **[BLOCKING] `Blocked` context has no action whitelist — hard conflict with 首局引导.**
   - `input.md:61` makes `Blocked` respond to `ui_pause` only ("若该界面允许").
   - `input.md:52` defines `push_context(context) -> bool` with no `allow` parameter.
   - `first-run-onboarding.md:42,56,123` hard-depends on
     `push_context(Blocked, allow=[…])` to pass `ui_accept`, `ui_cancel` and the
     four navigation actions through to the tutorial; `first-run-onboarding.md:503`
     records this as an open contract gap.
   - The undefined phrase "若该界面允许" also leaves no mechanism by which a
     `Blocked` screen declares that pause is permitted.
   - Fix: add `allow: Array[StringName] = []` to `push_context` (and, if needed,
     `request_context`), define that `Blocked` responds to `ui_pause` only when
     the pushed entry permits it, and state that an empty `allow` preserves the
     current "pause-only" semantics.

2. **[BLOCKING] Esc's dual binding and `Paused` precedence are contradictory.**
   - `input.md:33` (rule 1 table) maps `ui_cancel` to Backspace only and
     `ui_pause` to "Esc（Run 上下文）、P".
   - `input.md:37` (rule 2) says Esc is bound to **both** `ui_cancel` and
     `ui_pause`, and gives precedence for Run / Menu / Modal only.
   - `input.md:64` (Paused row) allows both `ui_pause`（恢复）and
     `ui_cancel`（打开退出确认）.
   - AC 2 (`input.md:212`) tests Run and Modal, never Paused.
   - An InputMap binding cannot be context-dependent, so the table is wrong at
     the mapping layer, and pressing Esc in `Paused` is genuinely ambiguous:
     resume or open the quit confirmation?
   - `ui-flow.md:108,136` assumes Paused `ui_cancel` opens the quit
     confirmation, and `scene-flow.md:33` assumes `ui_pause` resumes.
   - Fix: list Esc under **both** actions in the mapping table; define the
     context-level precedence explicitly, including `Paused` (recommend
     `ui_pause` resumes, `ui_cancel`/Backspace opens the quit confirmation, or
     the reverse — but pick one); add a Paused case to AC 2.

3. **[BLOCKING] AC 11 contradicts the context API and the focus/hover API.**
   - `input.md:221`: "只暴露 `is_action_just_pressed(action) -> bool`".
   - Rule 12 (`input.md:47–55`) adds `request_context` / `push_context` /
     `pop_context`; AC 18–20 exercise them.
   - `ui-flow.md:152` reads focus and hover state; `in-game-hud.md:171` reads
     focus state; `ui-flow.md:152` and `first-run-onboarding.md:123` write the
     context API.
   - As written, AC 11 forbids the API the rest of the GDD and every downstream
     system require.
   - Fix: enumerate the exposed API surface in AC 11 — action query, focus and
     hover queries, and the three context-request calls — and keep the
     "no `InputEvent` outside the input layer" static scan as the actual
     assertion.

4. **[BLOCKING] `game_skip_dish` has two incompatible meanings.**
   - `input.md:35` defines it as "右键（悬停的菜品卡）"; `input.md:151` says
     pressing it with no hovered dish card is a no-op.
   - `run-flow.md:33,35` treats it as "主动跳波 / 最后一波结束取餐", valid with no
     focused dish; AC 10–11 press it without one.
   - `menu-waves.md:29` treats it as "叫下一波"; `in-game-hud.md:59,534` renders
     it as "跳过本波 / 结束取餐" and flags the name mismatch.
   - Both `run-flow.md:445` and `in-game-hud.md:534` record this as an open
     question.
   - The action vocabulary is owned by this GDD, so it must be resolved here.
   - Fix: rename the action to `game_skip_wave` (or split into
     `game_skip_dish` + `game_end_pickup`), drop the hover precondition, and
     update the mapping table, edge cases, ACs and the three downstream GDDs.

5. **[BLOCKING] `push_context` on an equal top value is undefined.**
   - `input.md:55` says "空操作（同值覆盖）不发信号", without saying whether this
     covers `push_context(Modal)` when the stack top is already `Modal`.
   - `ui-flow.md:49` requires pushing `Modal` twice (settings modal → delete-career
     confirmation) and popping twice; `ui-flow.md` Open Question 14 explicitly
     asks this GDD to settle it. If the second push is a no-op, the first
     `pop_context()` unwinds the settings modal instead of the confirmation.
   - Fix: state that `push_context` always increases stack depth and always emits
     `context_changed(from, to)` — including `context_changed(Modal, Modal)` —
     and that `pop_context` likewise emits; only `request_context` with an
     identical current value is the no-signal no-op.

### Recommended Revisions

1. **`focus_owner` is undefined (AC 3).** AC 3 (`input.md:213`) asserts
   `focus_owner` equals a per-screen node path, but the term appears nowhere else
   and no focus-query API is specified. Define the query (e.g.
   `get_focus_owner() -> Control` and its path form) and how F9 reads it; then
   the AC is independently testable.
2. **AC 15 cites a nonexistent config key.** AC 15 (`input.md:225`) says the
   latency budget depends on "`project.yaml` 的目标帧率与输入轮询方式", but
   `project.yaml` has no performance/target-FPS block (top-level keys are
   `project`, `modes`, `engine`, `specialists`, `naming`, `commands`). Either add
   the setting in a later config pass or reword the AC to a fixed 60 FPS
   assumption.
3. **AC 9's static scan is over-broad.** A repo-wide search for `device ==`
   (`input.md:219`) also matches the engine-recommended
   `event.device == InputEvent.DEVICE_ID_MOUSE` comparison
   (`docs/engine-reference/godot/deprecated-apis.md:53`). Narrow the pattern to
   `device\s*==\s*0` (or `device == 0`).
4. **Cross-References is stale.** `input.md:205` says "None — this system
   references no other GDD" and "其余文档写好后需要在本表补充反向链接", but all six
   downstream GDDs are written and `scene-flow.md` is cited in Edge Cases
   (`input.md:142`). List the reverse links (or state that the Dependencies table
   is authoritative), and add the `design/gdd/scene-flow.md` path to the
   Dependencies status cell (`input.md:171`).
5. **音效反馈 dependency direction conflicts.** `input.md:78,170` list 音效反馈 as
   a soft downstream consumer of "动作触发事件"; `audio-feedback.md:373` says
   explicitly that it does **not** subscribe to raw input and consumes semantic
   events from 界面流程 / HUD. Remove the row, or restate it as "not a consumer —
   input requires a visible focus indicator, sound is not the only channel."
6. **`nav_steps` upper bound differs from the registry.** The GDD says `1–∞`
   (`input.md:97`); `entities.yaml` records `[1, 999]` and a practical clamp. If
   the 999 clamp is intended, state it in Formulas and add it to AC 6; if not,
   the registry needs correcting (orchestrator-owned, not this session).
7. **Same-frame different-action precedence is undefined.** `ui-flow.md:145`
   says "同一帧收到 `ui_accept` 与 `ui_cancel` 时，按 `输入` 规则 2 与当前上下文只
   消费一个动作", but rule 2 only addresses Esc's dual binding. Define a general
   same-frame rule (recommend: process events in arrival order, first valid
   action wins; or a fixed per-context priority) so the behaviour is
   deterministic and testable.

### Specialist Disagreements

None — lean mode runs no specialist agents, so there are no specialist views to
reconcile.

### Nice-to-Have

- Give the existing `### 调参起点` table its own `## Tuning Knobs` heading so the
  structure check and downstream skills find it (advisory at `standard`).
- Add a short `## Player Fantasy` section (advisory at `standard`); the Game Feel
  section already carries most of the content.
- `ui_cancel` on Backspace alone is unusual on PC; confirm discoverability once
  the Esc precedence in item 2 is settled.
- Consider spelling out in rule 5 that Godot does not auto-repeat key events, so
  the "长按不重复" requirement is a property of the implementation rather than an
  engine default.

### Senior Verdict [creative-director]

Not run — `lean` mode skips Phase 3b (adversarial specialists and the
creative-director synthesis). The verdict below is the main review's own.

### Scope Signal

**S** — single Foundation system, two formulas, no upstream dependencies, six
downstream consumers, no new ADR expected. Producer should still verify before
sprint planning because five contract fixes touch downstream GDDs.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (currently `Designed`).
The orchestrating session owns `design/gdd/systems-index.md`; this review did not
edit it.

---

## Receipt

Reviewed-Content-Hash: design/gdd/input.md c96715b12512d1a093a10f6cf2e32030740be509
Reviewed-Content-Hash: design/registry/entities.yaml cdef3f7a3f4a270848207a53e3eef821342c6294
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/run-flow.md 8d36e76a8f8024efa498c83cf83c804b856e77ba
Reviewed-Content-Hash: design/gdd/in-game-hud.md 9b437b7e4ca24a41f4ceb8dd0672f38578f58b43
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 778f0a836330a6410aee8812fa0801f9dd3cc4f3
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/scene-flow.md ac1d04a52a78a3027e95b4c81cde898e1b5ead06
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/systems-index.md 721a697860f5cbf632fa991257dc4145828b1955
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/engine-reference/godot/breaking-changes.md 4bc55e16796664ff88c7582c9f08e3530f7c94d4
Reviewed-Content-Hash: docs/engine-reference/godot/current-best-practices.md a45cd2c39a8ec53eaa3e924f39f93594e62085fd

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 9
Summary: Four of the five prior blockers are fully fixed — Esc/Paused precedence, AC 11's API surface, the `game_skip_wave` rename and equal-top `push_context` are now unambiguous and match downstream (including the focus/hover getters). The `Blocked` whitelist fix is incomplete: a non-empty `allow` replaces the default instead of adding to it, so `ui_pause` is discarded and 首局引导 cannot pause during a modal beat, contradicting its rule 17 / AC 19. The remaining recommended items are the stale 餐局流程 dependency row, AC 15's nonexistent `project.yaml` FPS key, the `device ==` scan, stale Cross-References, the 音效反馈 dependency, the `nav_steps` range, same-frame accept/cancel precedence, missing AC coverage for equal-top push, and the `focus_owner` naming.
Prior verdict resolved: No — 4 of 5 fully resolved; the `Blocked` whitelist blocker is only partially fixed.

Findings:
- [BLOCKING] Detailed Design (rule 12) / States: a non-empty `allow` replaces the default rather than adding to it, so `ui_pause` is dropped in `Blocked`; first-run-onboarding rule 17 / AC 19 require pausing during a modal beat, and its B0/B1 lists do not include `ui_pause`. Make `ui_pause` always allowed (allow additive), or state that callers must include it and align 首局引导.
- [RECOMMENDED] Interactions / Dependencies: 餐局流程 is listed as reading `ui_pause`, but `run-flow.md:116,284` says it does not read it (场景与流程管理 owns it); run-flow Open Question 2 already asks for this row to be fixed.
- [RECOMMENDED] Acceptance Criteria 15: the 1-frame budget still cites `project.yaml`'s target frame rate, which does not exist (`scene-flow.md` now uses a 60 Hz constant).
- [RECOMMENDED] Acceptance Criteria 9: the `device ==` scan still flags the engine-recommended `device == InputEvent.DEVICE_ID_MOUSE`; narrow it to `device == 0`.
- [RECOMMENDED] Cross-References: still says "None — this system references no other GDD" and that reverse links are pending, though `scene-flow.md` is cited in Edge Cases and all six downstream GDDs exist.
- [RECOMMENDED] Dependencies: 音效反馈 is listed as a soft consumer of action events while `audio-feedback.md:113,373,471` says it does not subscribe to input at all.
- [RECOMMENDED] Formulas / Registry: `nav_steps` output range is `1–∞` here but `[1, 999]` in the registry (practical clamp).
- [RECOMMENDED] Detailed Design: same-frame `ui_accept` + `ui_cancel` precedence is still undefined, though `ui-flow.md:145` cites "输入 规则 2" for it.
- [RECOMMENDED] Acceptance Criteria 19: covers push from `Run` only; the equal-top push added to rule 12 (`Modal` on `Modal`, depth 2) has no AC, and that is exactly ui-flow's depth-2 modal case.
- [RECOMMENDED] Acceptance Criteria 3: uses `focus_owner` while rule 8 defines `get_focus_owner()`; align the name.
Reviewed-Content-Hash: design/gdd/input.md 776c41de0eb3edd497ae2ace8c78ba80192fc78f
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/run-flow.md 02d37ce06fa293e6b76cb22277a68b705e413448
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/scene-flow.md c19ee4c2a48183476032f3ebbe2ceb9e43588324
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/systems-index.md 63d73495be76b5b37ed9bac50452c07ef281d41a
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/engine-reference/godot/breaking-changes.md 4bc55e16796664ff88c7582c9f08e3530f7c94d4
Reviewed-Content-Hash: docs/engine-reference/godot/current-best-practices.md a45cd2c39a8ec53eaa3e924f39f93594e62085fd

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 10
Summary: The `Blocked` whitelist is now additive in the API rule (rule 12), the States table, the `Blocked` edge case and AC 12 — `ui_pause` survives a non-empty `allow`, so 首局引导 can pause during a modal beat, and the other four prior blockers remain fixed. One Core Rule was not updated: rule 2 still says `Blocked` takes only the actions inside `allow`, which is the exclusive semantics the last round rejected and which contradicts those four sites. It is a one-sentence fix. The ten recommended items are stale wording and missing AC coverage; none blocks implementation on its own.
Prior verdict resolved: No — the contract is correct in four places but rule 2 still carries the exclusive wording.

Findings:
- [BLOCKING] Detailed Design (rule 2, `input.md:37`): "Blocked 只取 `allow` 白名单内的动作（默认仅 `ui_pause`）" still defines an exclusive whitelist. Rules 12 (`:55`), States (`:61`), Edge Cases (`:154`) and AC 12 (`:222`) now say `ui_pause` is always allowed and `allow` appends. With the non-empty `allow` 首局引导 passes (`first-run-onboarding.md:58`, B0/B1) the two readings disagree on whether Esc pauses — the precise defect the prior round blocked. Rewrite rule 2 to "Blocked 取 `ui_pause` 与 `allow` 追加的动作"; the two dependency descriptions (`:77`, `:169`) should say the same.
- [RECOMMENDED] Acceptance Criteria 12: states the additive rule in prose but never exercises it; add a case that pushes `Blocked` with `allow=[ui_accept]` and confirms `ui_pause` still fires — the regression this round fixed has no test.
- [RECOMMENDED] Interactions / Dependencies: 餐局流程 is listed as reading `ui_pause` (`:75`, `:167`), but `run-flow.md:116,284` says it does not read it (场景与流程管理 owns it); run-flow Open Question 2 already asks for this row to be fixed.
- [RECOMMENDED] Acceptance Criteria 15 (`:225`): the 1-frame budget still cites `project.yaml`'s target frame rate, which does not exist (`scene-flow.md` uses a 60 Hz constant).
- [RECOMMENDED] Acceptance Criteria 9 (`:219`): the `device ==` scan still flags the engine-recommended `device == InputEvent.DEVICE_ID_MOUSE`; narrow it to `device == 0`.
- [RECOMMENDED] Cross-References (`:205`): still says "None — this system references no other GDD" and that reverse links are pending, though `scene-flow.md` is cited in Edge Cases and all six downstream GDDs exist.
- [RECOMMENDED] Dependencies (`:78`, `:170`): 音效反馈 is listed as a soft consumer of action events while `audio-feedback.md:113,373,471` says it does not subscribe to input at all.
- [RECOMMENDED] Formulas / Registry (`:97`): `nav_steps` output range is `1–∞` here but `[1, 999]` in the registry (practical clamp).
- [RECOMMENDED] Detailed Design (`:37`): same-frame `ui_accept` + `ui_cancel` precedence is still undefined, though `ui-flow.md:145` cites "输入 规则 2" for it.
- [RECOMMENDED] Acceptance Criteria 19 (`:229`): covers push from `Run` only; the equal-top push defined in rule 12 (`Modal` on `Modal`, depth 2) has no AC, and that is exactly ui-flow's depth-2 modal case.
- [RECOMMENDED] Acceptance Criteria 3 (`:213`): uses `focus_owner` while rule 8 defines `get_focus_owner()`; align the name.
Advisory (not counted): `ui_pause` is now explicitly always allowed in every `Blocked` context, including the `Boot` loading screen (States `:61`); scene-flow has no `Boot → Paused` transition, so a press there is rejected and logged WARNING (`scene-flow.md:2,28`). Pre-existing, not introduced by this fix, but worth a no-op-without-log rule if Esc spamming during boot is expected.
Reviewed-Content-Hash: design/gdd/input.md ee9ea6022f8e289d9e00ef8f9eabb181b0c5349f
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/run-flow.md 02d37ce06fa293e6b76cb22277a68b705e413448
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/scene-flow.md a2c3cc81155d51d9fae36ecc8ef27bfc1d689a59
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/systems-index.md 63d73495be76b5b37ed9bac50452c07ef281d41a
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/engine-reference/godot/breaking-changes.md 4bc55e16796664ff88c7582c9f08e3530f7c94d4
Reviewed-Content-Hash: docs/engine-reference/godot/current-best-practices.md a45cd2c39a8ec53eaa3e924f39f93594e62085fd

---

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b not run）
Blocking items: 0 | Recommended: 11
Summary: The sole prior blocker is fixed. Rule 2 (`input.md:37`) now reads "Blocked 取 `ui_pause` 与 `allow` 追加的动作" and the two dependency descriptions (`:78`, `:170`) say the same, so rule 12, the States table, the `Blocked` edge case and AC 12 no longer disagree with it: `ui_pause` survives a non-empty `allow`, and 首局引导 can pause during a modal beat. The four blockers retired in earlier rounds remain fixed, and the `consume_action` addition from `be69169` (rule 12, AC 11) matches 首局引导, which already calls it. The design body is unchanged and consistent with the registry and with 界面流程 / 餐局流程 / 局内 HUD / 首局引导 / 场景与流程管理. The 11 remaining items are stale cross-document wording and range sync (5), three imprecise ACs, two missing ACs, and one new contract question about `consume_action`; none blocks implementation.
Prior verdict resolved: Yes

### Prior blocking — verified

| Prior blocking fix | Current state | Result |
|---|---|---|
| Rule 2 must say `ui_pause` plus `allow` (additive), not "only the allow whitelist" | `input.md:37` now "Blocked 取 `ui_pause` 与 `allow` 追加的动作"; agrees with rule 12 (`:56`), States (`:62`), Edge Cases (`:155`), AC 12 (`:223`) | ✓ resolved in `a7ac320` |
| The two dependency descriptions (`:77`, `:169`) must say the same | `:78` and `:170` both now "`ui_pause` 始终允许，`allow` 追加…" | ✓ resolved in `a7ac320` |

Re-verified unchanged and still correct: Esc dual binding + Paused precedence (`:37`, `:65`, AC 2 `:213`); AC 11's enumerated API surface (`:222`, now including `consume_action`); `game_skip_wave` rename with the hover precondition dropped (`:35`, `:152`, `:168`); equal-top `push_context` always pushing and signalling (`:56`); dual focus drawn simultaneously (`:43`, `:146`, AC 8 `:219`).

### Completeness (standard tier: 5 required + Formulas when the system defines numeric rules)

6/6 present: Overview, Detailed Design (= Detailed Rules), Formulas, Edge Cases, Dependencies, Acceptance Criteria. Formulas remains required — the system defines `nav_steps` and `stick_engaged`. Advisory (unchanged, not counted): no `## Player Fantasy` heading (Game Feel covers the substance) and Tuning Knobs is a `### 调参起点` subsection of Formulas.

### Dependency Graph

| Declared dependency | Direction | File | State |
|---|---|---|---|
| 场景与流程管理 | downstream reads + requests | `design/gdd/scene-flow.md` | ✓ exists; back-links input at `:76` |
| 界面流程 | downstream reads + requests | `design/gdd/ui-flow.md` | ✓ exists; back-links input |
| 餐局流程 | downstream reads | `design/gdd/run-flow.md` | ✓ exists; `:118`,`:288` back-link but explicitly exclude `ui_pause` (Recommended 2) |
| 局内 HUD | downstream reads | `design/gdd/in-game-hud.md` | ✓ exists; back-links input |
| 首局引导 | downstream reads + requests | `design/gdd/first-run-onboarding.md` | ✓ exists; `:123`,`:360` back-link and use `allow` |
| 音效反馈 | downstream reads | `design/gdd/audio-feedback.md` | ✓ exists but states it does not subscribe (Recommended 6) |
| Upstream | — | none | ✓ input is the foundation layer |

No broken links; all six declared downstream GDDs exist.

### Registry Values

`design/registry/entities.yaml` has two entries sourced from this GDD:

| Entry | Registry | This GDD | Result |
|---|---|---|---|
| `stick_engaged` | `(m >= E) or (stick_engaged_prev and m >= R)`, `[0,1]`, E=0.60 / R=0.40, `R <= E - 0.10` | `:108`, `:120`, `:136-137` identical | ✓ |
| `nav_steps` | `output_range: [1, 999]`, note "unbounded in formula; 999 is the practical clamp" | `:98` `1–∞` | ⚠ range only (Recommended 7) |

No other registry entry names this system.

### Findings

All 11 are RECOMMENDED; none is blocking.

1. [RECOMMENDED] Acceptance Criteria 12 (`:223`): states the additive rule in prose but never exercises it. Push `Blocked` with `allow=[ui_accept]` and assert `ui_pause` still fires — the regression fixed in `a7ac320` has no test.
2. [RECOMMENDED] Interactions (`:76`) / Dependencies (`:168`): 餐局流程 is listed as reading `ui_pause`, but `run-flow.md:118,288` says it does not (场景与流程管理 owns it) and `run-flow.md:448` OQ 2 already asks for this row to be fixed. Drop `ui_pause` from both rows here.
3. [RECOMMENDED] Acceptance Criteria 15 (`:226`): cites a `project.yaml` target-frame-rate key that does not exist (`project.yaml` has only `framework:` / `engine:`); `scene-flow.md:102` uses a 60 Hz constant instead. Point the AC at that constant, or restate the frame budget as a test-fixture value.
4. [RECOMMENDED] Acceptance Criteria 9 (`:220`): "源码搜索 `device ==` 命中 0 次" also forbids the engine-recommended `device == InputEvent.DEVICE_ID_MOUSE` (`breaking-changes.md:24`, `current-best-practices.md:148-150`). Narrow the scan to `device == 0`, the pattern rule 9 actually bans.
5. [RECOMMENDED] Cross-References (`:206`): still "None — this system references no other GDD" with "reverse links still to be added", though `scene-flow.md` is cited in Edge Cases (`:143-144`) and all six downstream GDDs exist. List the real paths and drop the stale "pending" claim.
6. [RECOMMENDED] Interactions (`:79`), Dependencies (`:171`) and Visual/Audio (`:184`): 音效反馈 is a soft consumer of action events, but `audio-feedback.md:116,377,475` says it does not subscribe to input at all. Align — remove the edge or mark it design-time only.
7. [RECOMMENDED] Formulas (`:98`): `nav_steps` output range `1–∞` vs registry `[1, 999]`. Match one; the registry note already documents the clamp.
8. [RECOMMENDED] Detailed Design (`:37`): same-frame `ui_accept` + `ui_cancel` precedence is still undefined; rule 2 resolves only the Esc dual binding. `ui-flow.md:147` ("同帧冲突") cites "输入 规则 2" for this case, so either define it here or have ui-flow stop citing rule 2 for it.
9. [RECOMMENDED] Acceptance Criteria 19 (`:230`): covers push from `Run` only; the equal-top push defined in rule 12 (`Modal` on `Modal`, depth 2) has no AC, and that is exactly ui-flow's depth-2 modal case.
10. [RECOMMENDED] Acceptance Criteria 3 (`:214`): uses `focus_owner` while rule 8 defines `get_focus_owner()` (`:43`); align the name.
11. [RECOMMENDED] Detailed Design (rule 12 table, `:54`) / AC 11 (`:222`): `consume_action` is new in `be69169` and its effect on `is_action_just_pressed` is unstated. 首局引导 relies on that effect — `first-run-onboarding.md:187` says the run flow "随后轮询同一个动作得到 `false`" — so consuming must suppress the polled action for the rest of the frame, not only return `false` to a second `consume_action`. State that, and add an AC: consume `ui_accept`, then assert `is_action_just_pressed(ui_accept) == false` and a second `consume_action` returns `false`. This is the only new item this round; the other ten were carried over.

Advisory (unchanged, not counted): `ui_pause` is allowed in every `Blocked`, including `Boot` (`:62`), but scene-flow has no `Boot → Paused` transition (`scene-flow.md:47,56,67`), so a press during boot is rejected and WARNING-logged. Pre-existing; a no-op-without-log rule would fix it if boot-time Esc spamming is expected.

### Verdict

**APPROVED** (0 blocking · 11 recommended). The one prior blocker and the two dependency descriptions are fixed exactly as asked in `a7ac320`; the remaining findings are documentation and test hygiene — none changes the input contract or blocks an implementer.

- Suggested `systems-index` status: **Approved** (row 2 is currently `In Review`). Not edited here — the orchestrator owns the index.
- This GDD was not modified; only this log entry was appended.
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/menu-waves.md ce532d3265df1ed7555ba5aa46383be35bbee4a1
Reviewed-Content-Hash: design/gdd/systems-index.md c0b526ff05088d426598f5dc34e3bd9e3153cf59
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/engine-reference/godot/breaking-changes.md 4bc55e16796664ff88c7582c9f08e3530f7c94d4
Reviewed-Content-Hash: docs/engine-reference/godot/current-best-practices.md a45cd2c39a8ec53eaa3e924f39f93594e62085fd
Reviewed-Content-Hash: docs/engine-reference/godot/modules/input.md 3406b1b813858898696e510ea0accfcc7df98af8

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 4 | Recommended: 9
Summary: Fresh full lean review of the current bytes. The 2026-11-04 stamp is stale: `1b41d84` edited the file after it (Blocked row lost "结算动画", a new "Blocked 的两类请求方" paragraph landed, the 餐局流程 rows dropped `ui_pause`, 音效反馈 became "无直接依赖"), and three of those four edits are correct and now match `scene-flow.md` (`Boot → Blocked`), `run-flow.md:119,290` (no `ui_pause`), `first-run-onboarding.md` (`push_context(Blocked, allow)`), `in-game-hud.md:59` (confirm panel stays in `Run`, no `push_context`) and `audio-feedback.md:136,404`. Four contract gaps remain, all against ADRs Accepted after that stamp (ADR-0005 `ca65edc`/`ff61386`, ADR-0020 `ff61386`): rule 2 still resolves Esc to `ui_pause` inside `Blocked` where the ADRs mandate `ui_cancel`; the States table omits navigation from `Paused` although `ui-flow`'s pause overlay needs it; rule 12 says only the context name is validated although the Accepted stack defines `STACK_FULL` / `UNKNOWN_ACTION`; and AC 11 pins a seven-member surface that the Accepted interface list extends with `attach()` / `get_context_snapshot()`. Five earlier recommendations (nav_steps range, 餐局流程 `ui_pause`, 音效反馈 edge, `device ==` scan, `consume_action` semantics) are closed by the same edits and ADRs.
Prior verdict resolved: No — no earlier blocker regressed, but the Accepted-ADR contract has moved and four new blockers follow.

### Why the receipt is stale

`review-receipts.sh check` prints `CHANGED` for `design/gdd/input.md` and `CHANGED` for `design/registry/entities.yaml`; the log's last `APPROVED` (2026-11-04) is stamped `9432d857…`, the current file is `27c9d888…`. `git diff 42d4d31..HEAD -- design/gdd/input.md` shows the post-approval edits: the `Blocked` row no longer claims "结算动画", the new "Blocked 的两类请求方（无第三方）" paragraph was added, both 餐局流程 rows stopped listing `ui_pause`, and 音效反馈 became "无直接依赖 / 不订阅原始输入" with the Visual/Audio paragraph rewritten. Those four are substantive in aggregate (they change what `Blocked` means and who reads `ui_pause`), so the 2026-11-04 approval cannot cover the current bytes; this is a fresh full lean review, not a delta review.

**Post-approval edits verified.** The "Blocked 的两类请求方" paragraph (`:70`) is correct against `scene-flow.md:29` (`Boot → Blocked`), `first-run-onboarding.md:44,64` (`push_context(Blocked, allow)`), `run-flow.md:84-87` (`Ending`/`Settling`/`Handoff` are `run_state` values inside the `Run` top-level state), `in-game-hud.md:59` (the confirm panel is a focus trap in `Run`, "不是输入上下文意义上的 `Modal`", no `push_context`) and ADR-0005 D9 (`push_context(&"Blocked")` only in `first_run_onboarding.gd`). It also closes ADR-0005 Open Question 6 (Blocked for the settlement animation). The 餐局流程 rows (`:78`, `:170`) now match `run-flow.md:119,290`. Rule 12 (`:56`) and the `Blocked` row (`:62`) both say `ui_pause` always allowed plus the `allow` list, matching ADR-0005 D3 (`:144,151`). Dual focus (`:21`, `:43`, `:183`) is engine-correct: 4.6 dual focus and 4.7 `DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD` per `breaking-changes.md:24,55`, `modules/input.md:8-9,51-61`.

### Findings

| # | Severity | Location | Finding | Authority it violates |
|---|---|---|---|---|
| 1 | BLOCKING | `design/gdd/input.md:37` | Rule 2's Esc-resolution sentence names only Run/Paused → `ui_pause` and Menu/Modal → `ui_cancel`, then gives `Blocked` as "`ui_pause` 与 `allow` 追加的动作" — two answers to a "取一个" question. Read literally, Esc in `Blocked` yields `ui_pause`, so 首局引导's documented "Esc 跳过" never fires and the press pauses instead. | ADR-0005 D3 `:152` ("`Menu` / `Modal` / `Blocked` 取 `ui_cancel`"); ADR-0020 `:373` ("Esc 在 `Blocked` 里被翻译成 `ui_cancel`"); `first-run-onboarding.md:82,540` (skip on `ui_cancel`) |
| 2 | BLOCKING | `design/gdd/input.md:65` | The `Paused` row's 响应动作 lists `ui_pause` / `ui_accept` / `ui_cancel` but not 导航; only `game_skip_wave` is listed as blocked, so navigation is neither allowed nor blocked. The pause overlay has four items ("继续/设置/放弃本局/退出游戏"), so an implementer following this table can freeze focus on 「继续」. | ADR-0005 D3 `:147` (Paused allows the four nav actions) and D4 `:163` (Paused nav → engine GUI); `ui-flow.md:112` (`PauseOverlay`: "导航、`ui_accept`、`ui_cancel`") |
| 3 | BLOCKING | `design/gdd/input.md:56` | Rule 12 says input "只校验 `context` 是否为五个已知上下文之一" and that `push_context` "总是增加栈深"; no stack cap and no `allow`-vocabulary check exist. Both are caller-observable failures the Accepted stack defines, and the depth rejection is what `ui-flow` and 首局引导's degradation path rely on. | ADR-0005 D1 `:116-123` (`STACK_FULL` at `MAX_CONTEXT_DEPTH = 2`, `UNKNOWN_ACTION` for a bad `allow` name); `ui-flow.md:51` ("深度已满时新的模态请求被拒绝并记 WARNING") |
| 4 | BLOCKING | `design/gdd/input.md:224` | AC 11 asserts the exposed surface is "恰好" the seven runtime members. The Accepted interface list adds `attach(tuning, sink)` (construction-time) and `get_context_snapshot()` (read-only, carries `input_seq` for the beat machine), so the AC fails as written and cannot be executed without interpreting the ADR. | ADR-0005 Key Interfaces `:276,288` and Open Question 1 `:502`, which explicitly asks this GDD to amend AC 11 |
| 5 | RECOMMENDED | `design/gdd/input.md:228` | AC 15 still says the 1-frame budget "依赖 `project.yaml` 的目标帧率"; `project.yaml` has only `framework:` / `engine:` — no frame-rate key. | `scene-flow.md:102` (`FRAME_TIME = 0.0167`, 60 Hz constant, "不读 `project.yaml`") |
| 6 | RECOMMENDED | `design/gdd/input.md:148` | "最近一次输入决定主高亮" introduces a primary-highlight concept no other doc owns and that reads against HUD's fixed priority; submission stays keyboard-first regardless of which device moved last. | `in-game-hud.md:55` (rule 14: `focused_dish_id` = "键盘焦点优先于悬停"), `in-game-hud.md:521` (AC 38) |
| 7 | RECOMMENDED | `design/gdd/input.md:70` | "`Ending` / `Settling` / `Handoff` 阶段保持 `Run` 上下文" is stated unconditionally, but the player may pause during those phases (the pause overlay stays up and shrinks to 「继续」「退出游戏」), which makes the context `Paused`, not `Run`. | `run-flow.md:41,100` (pause during `Settling` waits), `run-flow.md:16`, `ui-flow.md:73` |
| 8 | RECOMMENDED | `design/gdd/input.md:208` | Cross-References still opens "**None — this system references no other GDD.**" while the body cites `scene-flow.md` (`:145`), `in-game-hud.md` (`:70`) and `audio-feedback.md` (`:186`); the "反向链接" claim is true but the section contradicts itself. | Design rules § Dependencies (bidirectional, no stale framing) |
| 9 | RECOMMENDED | `design/gdd/scene-flow.md:29` | "`Modal` 与 `Blocked` 可由界面流程或首局引导临时压栈" lets 界面流程 push `Blocked`. `input.md:70` and the ADR's static-scan table allow exactly two requesters, and `ui-flow.md` pushes only `Modal`. The wording, not the rule under review, is wrong. | ADR-0005 D9 `:227-228`; `input.md:70`; `ui-flow.md:51` (`push_context(Modal)` only) |
| 10 | RECOMMENDED | `design/gdd/input.md:225` | AC 12 states the additive `allow` rule in prose but never exercises it; the regression fixed in `a7ac320` (non-empty `allow` must not drop `ui_pause`) still has no GDD test. | Design rules § AC testability; ADR-0005 Validation Criteria `:486` covers only the ADR side |
| 11 | RECOMMENDED | `design/gdd/input.md:232` | AC 19 pushes from `Run` only; the equal-top push rule 12 added (`context_changed(Modal, Modal)`, depth 2) has no AC, and that is exactly ui-flow's delete-career modal. | `ui-flow.md:51,643` (OQ 14, `MAX_MODAL_DEPTH = 2`); ADR-0005 `:473` |
| 12 | RECOMMENDED | `design/gdd/input.md:216` | AC 3 asserts `focus_owner` while rule 8 (`:43`) defines the query as `get_focus_owner() -> NodePath`; the bare name is also the name of a signal. | internal consistency; ADR-0005 Key Interfaces `:286` |
| 13 | RECOMMENDED | `design/gdd/input.md:37` | Same-frame `ui_accept` + `ui_cancel` precedence is still undefined here, though 界面流程 delegates to it ("按 `输入` 规则只消费一个动作"). The rule exists only in the Accepted ADR. | `ui-flow.md:372`; ADR-0005 D5 `:177` |

Closed since the 2026-11-04 approval (re-verified, do not re-raise): `nav_steps` range now `[1, +∞)` in both GDD (`:100`) and registry (`entities.yaml:344`); 餐局流程 `ui_pause` rows dropped; 音效反馈 edge restated as "无直接依赖" matching `audio-feedback.md:136,404`; AC 9's `device ==` scan now matches ADR-0005 D8 `:219` and its Validation Criteria `:483` (the engine allows `DEVICE_ID_*` comparisons, the ADR forbids them — the ADR is authoritative here); `consume_action`'s consumed-bit effect is defined by ADR-0005 D4 `:167`.

### Completeness: 6/6 required sections present (standard tier)

Overview, Detailed Design (= Detailed Rules), Formulas, Edge Cases, Dependencies, Acceptance Criteria — all present and non-empty. Formulas is REQUIRED here and present: the system defines the numeric rules `nav_steps` (repeat delay/interval) and `stick_engaged` (engage/release thresholds), both with variable tables, ranges, a worked example and an output-range statement. Advisory at `standard`, not counted: no `## Player Fantasy` heading (Game Feel covers the substance) and Tuning Knobs exists as the `### 调参起点` table inside Formulas — promoting it to its own heading would let the structure check and downstream skills find it.

### Dependency graph

Upstream: none declared — correct, input is the Foundation layer. Downstream: `ui-flow.md`, `run-flow.md`, `in-game-hud.md`, `first-run-onboarding.md`, `scene-flow.md` all exist and all back-link 输入; `audio-feedback.md` exists and correctly declares no dependency (`:136,404,500`). No broken reference. `systems-index.md:59` row 2 (输入) reads `Approved` and is not edited here.

### Registry values

Two entries are sourced from this GDD: `nav_steps` (`entities.yaml:335-347`) and `stick_engaged` (`:350-362`). Expression, variables, defaults (`D=0.35`, `I=0.12`, `E=0.60`, `R=0.40`), the `R <= E - 0.10` constraint and both output ranges now match `input.md:91,100,110,118,122` exactly. No other registry entry names this system, and no interface entity is registered, so AC 11's surface is checked against ADR-0005 only.

### Verdict

**NEEDS REVISION** (4 blocking · 9 recommended). Three of the four blockers are one-cell/one-sentence alignments to ADRs Accepted on 2026-11-04/05 (Esc inside `Blocked`, navigation inside `Paused`, the `push_context` rejection surface); the fourth needs AC 11 re-enumerated to include `attach()` and `get_context_snapshot()`. Nothing in the post-approval edits is wrong — the blockers come from the Accepted ADR set moving past the GDD, which is why the stale stamp could not have caught them.

- Suggested `systems-index` status: **In Review** (row 2 currently `Approved`; substantive edits after the stamp invalidate the approval). Not edited here — the orchestrator owns the index.
- `design/gdd/input.md` was not modified; only this log entry was appended.
Reviewed-Content-Hash: design/gdd/input.md 27c9d885eb2ba1f8d4b3df72ff6640fed0a5dd36
Reviewed-Content-Hash: design/registry/entities.yaml f7b5aded130f371160d84bdaf678f8530ddc0807
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/scene-flow.md 07753accfc002be7257cbcdd17323c73c2b6845a
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/in-game-hud.md fee7199048c3a124fcab2eccae8b88f93d1a69c0
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 5252deab0b23680664d07fe7ce1c7255e89240ed
Reviewed-Content-Hash: docs/architecture/tr-registry.yaml b26b2e0ff92eec3bda1b34779ed6dd850f0b4941
Reviewed-Content-Hash: docs/engine-reference/godot/breaking-changes.md 4bc55e16796664ff88c7582c9f08e3530f7c94d4
Reviewed-Content-Hash: docs/engine-reference/godot/modules/input.md 3406b1b813858898696e510ea0accfcc7df98af8
Reviewed-Content-Hash: docs/engine-reference/godot/current-best-practices.md a45cd2c39a8ec53eaa3e924f39f93594e62085fd

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 3 | Recommended: 6
Summary: 全量 lean 复审（非增量）。上一轮 4 个 blocking 已逐条修好：规则 2 的 Esc 五上下文分流（`Run` / `Paused` → `ui_pause`，`Menu` / `Modal` / `Blocked` → `ui_cancel`，Blocked 另留 P / Start 的 `ui_pause`）、Paused 行补齐四个导航动作、规则 12 补 `MAX_CONTEXT_DEPTH = 2` / `UNKNOWN_ACTION` / `STACK_FULL` / 栈空 `pop` 不弹基底、AC 11 补 `attach` / `get_context_snapshot` / 悬停注册并在 AC 2 加首局引导 Blocked/Esc 用例，新增 AC 21 且编号 1–21 无冲突。但规则 12 新增的"深度 2 覆盖全部合法组合"把模态写成"只压 1 层 `Modal`"，与 ADR-0005 D2 和 `ui-flow.md` 规则 5 的 `MAX_MODAL_DEPTH = 2`（设置模态内再开删除生涯确认＝两次 `push_context(Modal)`）正面冲突，AC 21 的"压入一层 `Modal`（深度 2）"复述同一错误并把它写成验收；另外规则 2 的"Run 中读 `ui_cancel` 恒为 `false`"与 Run 行不列 `ui_cancel` 一起，挡掉了 `in-game-hud.md` 规则 16 在 Run 内用 `ui_cancel` 关闭结束取餐确认面板所依赖的动作。
Prior verdict resolved: Yes —— 2026-11-06 的 4 个 blocking 均按建议修好（规则 2 / Paused 行 / 规则 12 校验面 / AC 11）；本轮 3 个 blocking 中两个出自本轮新增的深度 2 表述与 AC 21，一个是对 Run 内 `ui_cancel` 的长期错误断言，不是旧项回归。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/input.md`、`design/registry/entities.yaml`、`run-flow.md`、`in-game-hud.md`、`scene-flow.md`、`menu-waves.md`；`UNCHANGED`：`SKILL.md`、`adr-0005`、`first-run-onboarding.md`、`audio-feedback.md`、`game-concept.md`、`ui-flow.md`、`systems-index.md`、`project.yaml` 与三份引擎参考。上一份 NEEDS REVISION（2026-11-06）绑定 `27c9d885…`，当前为 `18815e7d…`，目标文档与跨文档事实均已变动，故按全文复审。

### Findings

| # | 严重度 | 位置 | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | **BLOCKING** | `design/gdd/input.md:56` | 规则 12 末句"**深度 2 覆盖全部合法组合**：……模态只压 1 层 `Modal`（只在 `Menu` / `Paused` / `Error`），两者不可能同时存在"把模态层的合法深度写成 1。ADR-0005 D2 明确"`push_context(Modal)` 在栈顶已经是 `Modal` 时仍压入第二层……是 `ui-flow.md` Open Question 14 的答案，也是 ADR-0003 模态深度 2 的前提"，`ui-flow.md:51` 与 `:205` 的 `MAX_MODAL_DEPTH = 2` 只允许两种组合：深度 1（单模态）与深度 2（`SettingsModal` 内再开 `ConfirmModal`）。按本句实现会把合法的第二层模态判成缺陷，删生涯确认框无法打开 | ADR-0005 D2（`adr-0005-input-context-stack.md:129`）、D1（`:123` "`ui-flow.md` 的模态栈上限是 2"）、Validation Criteria（`:473` 两次 `push_context(Modal)` → `depth == 2`）；`ui-flow.md:51`、`:210` |
| 2 | **BLOCKING** | `design/gdd/input.md:234`（AC 21） | "GIVEN F8 已在 `Run` 上压入一层 `Modal`（深度 2），WHEN 再次 `push_context(&"Modal")`，THEN 返回 `false`、记 `STACK_FULL`"——"一层"与"深度 2"自相矛盾（栈深只数临时栈：一次压栈 = 1），且它把第 1 条的误解写成验收。真正需要钉的两条反而都没有：① 深度 1 时第二次 `push_context(Modal)` 必须**成功**并发出 `context_changed(Modal, Modal)`（ADR-0005 D2 的核心裁定、`ui-flow.md:612` AC 39 的前提）；② 深度 2 时再压才 `STACK_FULL` | ADR-0005 D2（`:129`）、D1 表（`:116`）、Validation Criteria（`:473`）；`ui-flow.md:51`、AC 39（`:612`） |
| 3 | **BLOCKING** | `design/gdd/input.md:37`、`:64` | 规则 2 写"被淘汰的那个动作当帧不进动作集，因此 Run 中读 `ui_cancel` **恒为 `false`**"，States 表 `Run` 行的响应动作也只列导航 / `ui_accept` / `game_skip_wave` / `ui_pause`。这两句把 Esc 的单动作解析推广成"Run 内不存在 `ui_cancel`"，而 `ui_cancel` 另绑 Backspace：ADR-0005 D4 的 Run 行写"`ui_accept` / `ui_cancel` → **不投递**……HUD 轮询"，D9 把 `is_action_just_pressed(&"ui_cancel")` 授予 `HudHost`（"仅用于关闭结束取餐确认面板"），`in-game-hud.md:59` 规则 16 与 AC 25 / AC 27（`:503`、`:506`）要求 Run 内 `ui_cancel` 关闭确认面板并把焦点还原。按本句实现，该面板只能用焦点/确认关闭，"取消"路径与 HUD AC 27 同时失效 | ADR-0005 D4（`:164`）、D9（`:231`）；`in-game-hud.md:59`、`:503`、`:506` |
| 4 | RECOMMENDED | `design/gdd/input.md:47` | 规则 12 的引导句写"接口三个："，紧随其后的表却有四行（`request_context` / `push_context` / `pop_context` / `consume_action`）；`consume_action` 加入后计数没有更新，与本轮改写的规则 12 正文直接冲突 | 本文件规则 12 表（`:49-54`）；AC 11（`:224`）已按七个计数 |
| 5 | RECOMMENDED | `design/gdd/input.md:224`（AC 11）/ AC 21（`:234`） | AC 21 要求"记 `STACK_FULL`""记 `UNKNOWN_ACTION`"，但公开接口只返回 `bool`，`get_context_snapshot()` 的字段是 `base` / `stack` / `effective` / `allow` / `input_seq` / `unconsumed`，没有 reject 位；ADR-0005 `Reject` 枚举（`:272`）无读取口。该断言只能靠日志捕获，无法用公开接口判定 | ADR-0005 Key Interfaces（`:288-300`）、`enum Reject`（`:272`）；design-docs「验收标准必须可测」 |
| 6 | RECOMMENDED | `design/gdd/ui-flow.md:149`、`:372` | 两处仍让界面流程"按 `输入` 规则 2 与当前上下文只消费一个动作"，但规则 2 只定义 Esc 双绑定的单动作解析；同帧 `ui_accept` + `ui_cancel` 的 precedence 在 ADR-0005 D5（`:177`）有、在本 GDD 没有（沿用上轮第 13 条，未修） | `ui-flow.md:149`、`:372`；ADR-0005 D5（`:177`） |
| 7 | RECOMMENDED | `design/gdd/input.md:79`、`:171` | 局内 HUD 的接口列只写"菜品卡的悬停、点击、右键"，没有 `ui_cancel`；第 3 条的修法需要在这一行同步出现（HUD 是 Run 内 `ui_cancel` 的唯一读者） | 本文件规则 2 / States 表；`in-game-hud.md:391`（HUD 自述读 `ui_cancel`） |
| 8 | RECOMMENDED | `design/gdd/input.md:208` | Cross-References 仍以"**None — this system references no other GDD.**"开篇，而正文引用 `scene-flow.md`（`:145`）、`in-game-hud.md`（`:70`）、`audio-feedback.md`（`:186`）（沿用上轮第 8 条，未修） | design-docs「Dependencies 必须双向」 |
| 9 | RECOMMENDED | `design/gdd/input.md:224` | AC 11 把 `register_hover_target` / `unregister_hover_target` 列入"另有"，引用 `ADR-0005 Key Interfaces / OQ1` 是准确的（Key Interfaces `:290-291` 有这两个函数），但 ADR 自己的 Validation Criteria（`:484`）只列七个 + `attach` + `get_context_snapshot`，漏了悬停注册。应在 ADR 侧补齐，否则"接口面"两处清单互相打脸 | ADR-0005 Key Interfaces（`:290-291`）vs Validation Criteria（`:484`）；本文件 AC 11 |

### 已核对且无问题（记录以免重复检查）

- **Esc 五上下文**：规则 2（`:37`）的 `Run` / `Paused` → `ui_pause`、`Menu` / `Modal` / `Blocked` → `ui_cancel` 与 ADR-0005 D3（`:152`）逐字同义；Blocked 的"`ui_pause` 仍允许（P / Start），但不是 Esc 的翻译结果"与 D3（`:144`、`:151`）一致；Paused 中 `ui_cancel` 只由 Backspace 触发与规则 2 自洽。
- **Paused 行**：导航 + `ui_pause`（恢复）+ `ui_accept` + `ui_cancel`、屏蔽 `game_skip_wave`，与 ADR-0005 D3（`:147`）的七动作集合等价（沿用上轮 blocking 2 的修法）。
- **规则 12 的拒绝面**：`{UNKNOWN_CONTEXT, UNKNOWN_ACTION, STACK_FULL}` + "失败时栈与当前上下文都不变、不发信号" + 栈空 `pop` 返回 `false` 且绝不弹基底，与 ADR-0005 D1 表（`:119-121`、`:118`）一致；ADR 的 `enum Reject { NONE, UNKNOWN_CONTEXT, UNKNOWN_ACTION, STACK_FULL }`（`:272`）多一个 `NONE` 默认值，本 GDD 未点名它，不构成冲突（可观测性问题见推荐 5）。
- **七接口 + 附加面**：`is_action_just_pressed` / `get_focus_owner` / `get_hover_owner` / `request_context` / `push_context` / `pop_context` / `consume_action` 恰为 ADR Key Interfaces（`:278-291`）的运行时集合；`attach(tuning, sink)` 确为构造期一次性入口，`get_context_snapshot()` 只读且带 `input_seq`、不泄漏 `InputEvent`，与 OQ1（`:502`）的建议一致。
- **新增 AC 的编号与内容**：AC 1–21 连续无冲突（AC 2 在 `:215`，AC 11 在 `:224`，AC 21 在 `:234`）；AC 2 的"首局引导已压入 Blocked 并按 Esc → 只触发 `ui_cancel`（引导段据此跳过）"与 ADR-0020（`:373`）"Esc 在 `Blocked` 里被翻译成 `ui_cancel`"一致。
- **悬停注册**：规则 8（`:43`）的 `register_hover_target` / `unregister_hover_target` 与 ADR-0005 D8（`:217`，`mouse_entered` / `mouse_exited` 订阅、`tree_exiting` 清理）一致，"不每帧遍历场景树"与 D8 的动机相同。

### Completeness: 6/6 required sections present（standard 层）

Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria 齐全且非空。Formulas 对本系统是必需且已满足（`nav_steps`、`stick_engaged` 两条，均含变量表、值域、输出范围与算例；`accept_cooldown` 明确为"不设"）。Advisory（不计入）：无 `## Player Fantasy` 标题（Game Feel 覆盖实质）、Tuning Knobs 以 `### 调参起点` 子节存在于 Formulas 内。

### Dependency Graph

上游：无（Foundation 层，正确）。下游六份均存在且回指：`ui-flow.md`、`run-flow.md`、`in-game-hud.md`、`first-run-onboarding.md`、`scene-flow.md`、`audio-feedback.md`（后者正确声明无订阅）。无断链。`systems-index.md:59` 第 2 行当前记 `Approved`，本次未改索引。

### Registry Values

`entities.yaml` 中由本 GDD 拥有的两个条目仍在：`nav_steps`（`:335`，表达式与 `output_range: [1, 999]` 未变）与 `stick_engaged`（`:350-362`，`(m >= E) or (stick_engaged_prev and m >= R)`、E = 0.60 / R = 0.40、`R <= E - 0.10`）。对本文件 `:91`、`:100`、`:110`、`:118`、`:122`、`:136-137` 逐行比对，**无冲突**；本轮改动只触及上下文语义与接口面，未复述任何数值。无接口实体登记，故 AC 11 只能对 ADR-0005 核对（已核）。

### Verdict: NEEDS REVISION

**NEEDS REVISION**（3 blocking · 6 recommended）。三处都是"契约与已 Accepted 的 ADR / 已写 GDD 反向"：深度 2 的合法模态组合被写窄（规则 12），AC 21 把同一误解钉成验收，"Run 内无 `ui_cancel`"否掉了 HUD 取消路径。三处修法都很小（一句话、一条 AC 的 GIVEN、Run 行补一个动作并把"恒为 `false`"限定到 Esc），但都会改变实现行为，不能只当措辞。

- 建议 `systems-index` 状态：**Needs Revision**（第 2 行当前 `Approved`；2026-11-06 之后仍有实质改动）。本次未改索引与文件头，由编排会话统一处理。
- 本 GDD 未被修改，只追加本日志条目。
Reviewed-Content-Hash: design/gdd/input.md 18815e7d6735d9a0995b8cd79a762b516a07be1a
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/scene-flow.md 4f0e50743f8b72012ff85540a58dfa1481cd676f
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/in-game-hud.md 27e2978106c35139e3a3d0f371472a7aafd5dc2a
Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: design/gdd/menu-waves.md a0b26691ca0f6ba5cac4c01ac0704913179b43ed
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 5252deab0b23680664d07fe7ce1c7255e89240ed
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 1 | Recommended: 7
Summary: 全文 lean 复审（非增量）。上轮 3 个 blocking 全部按建议修好：规则 2 把"`ui_cancel` 恒为 `false`"限定到 Esc 产生的动作并单独说明 Backspace，Run 行恢复 `ui_cancel`，规则 12 改为 `depth = stack.size()`（不含基底）且"深度 2"覆盖单模态与 `SettingsModal → ConfirmModal` 同值压栈，AC 21 按"第 1 次深度 1 / 第 2 次深度 2 发 `(Modal, Modal)` / 第 3 次 `STACK_FULL`"重写——与 ADR-0005 D1/D2/D3/D4 及 `ui-flow.md:51` 已完全一致。新发现 1 个 blocking：本文件把 `context_changed` 的订阅者记为场景与流程管理，与两张已 Accepted 的 ADR 归属表冲突。其余为旧 recommended 未修与两处 AC 可测性。
Prior verdict resolved: Yes —— 上轮 3 个 blocking 全部修好（逐条见下）；本轮 blocking 是另一个缺陷（信号订阅者归属），不是回归。上轮 6 条 recommended 中第 4、7、8 条仍未修，第 5 条的"日志不可从公开接口读"在 AC 21 里依旧存在。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/input.md`、`design/registry/entities.yaml`、`run-flow.md`、`in-game-hud.md`、`scene-flow.md`、`menu-waves.md`；`UNCHANGED`：`SKILL.md`、`design-docs.md`、`adr-0005`、`adr-0013`、`adr-0020`、`ui-flow.md`、`first-run-onboarding.md`、`audio-feedback.md`、`game-concept.md`、`systems-index.md`、`project.yaml` 与三份引擎参考。上一份 NEEDS REVISION（2026-11-17）绑定 `18815e7d…`，当前字节为 `1ef73080…`；目标文档与跨文档事实均已变动，按全文复审。头部 `Status` 仍停在 `In Review — 2026-11-17 修订（第二轮）`，只记录不改。

### 上轮 3 个 blocking 的修复核对

- **规则 2（`:37`）修好。** 现在写"由 Esc 产生的 `ui_cancel` 在 Run 中读到 `false`……这不等于 Run 不接受 `ui_cancel`：Run 的允许集含 `ui_cancel`，Backspace 可产生它，HUD 用它关闭「结束取餐」确认面板（`in-game-hud.md` 规则 16 / AC 27）"，并把 Blocked 的 `ui_pause`（P / Start）明确排除在 Esc 翻译之外。逐句对上 ADR-0005 D3（`adr-0005:152`）、D9（`:231`）与 `in-game-hud.md:59`、`:503`、`:506`。
- **Run 行（`:64`）修好。** 响应动作现为导航 / `ui_accept` / `ui_cancel`（Backspace，关闭局内确认面板）/ `game_skip_wave` / `ui_pause`，与 D3 的"八个动作全部"等价；屏蔽列 `—` 正确。
- **规则 12（`:56`）修好。** `depth = stack.size()` 明确不含基底，`push_context` 在 `depth == 2` 返回 `false` 记 `STACK_FULL`，"深度 2"对应 `ui-flow.md` 规则 5 的两种组合（单模态深度 1；`SettingsModal` 上再压 `ConfirmModal` 深度 2，同值压栈同样加深并发 `context_changed(Modal, Modal)`），首局引导只压 1 层 `Blocked`。与 D1 的 `depth = stack.size()` / `MAX_CONTEXT_DEPTH = 2` 表（`adr-0005:107`、`:116`、`:123`）、D2（`:129`）、Validation Criteria（`:473`）以及 `ui-flow.md:51`（`MAX_MODAL_DEPTH = 2`，深度 1 = 单模态、深度 2 = `SettingsModal → ConfirmModal`）**逐值一致，基底不计数**。
- **AC 21（`:234`）修好且可测。** 深度经 `get_context_snapshot().stack`（`adr-0005:294-297`）可观测：第 1 次 `push_context(&"Modal")` 返回 `true`、深度 1；第 2 次返回 `true`、深度 2、发 `context_changed(Modal, Modal)`；第 3 次 `false` / `STACK_FULL` / 深度仍 2 / 不发信号 / 栈顶不变；再补 `UNKNOWN_ACTION` 与"多弹一次不弹基底"。与 D1 表、D2、AC 20 同形。

### Findings

| # | 严重度 | 位置 | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | **BLOCKING** | `design/gdd/input.md:76`、`:174` | 交互表与依赖表都写"场景与流程管理……`context_changed(from, to)` / 订阅 `context_changed`"。ADR-0005 D9 明文"**`SceneFlow` 不订阅 `context_changed`**：它是唯一 `request_context` 调用方，回执就是返回值。让它再订阅会形成「自己触发、自己响应」的回环"；ADR-0004 D9 的订阅者只有 `UIRoot`（深度断言）与 `AppRoot`（日志）。按本行实现即制造该 ADR 禁止的重入，且真正的有效上下文消费者（首局引导的 `Blocked` 判定）在本表 `:80` 反而没写 | ADR-0005 D9（`adr-0005:234`）、D2（`:133`）；ADR-0004 D8/D9（`adr-0004:228`） |
| 2 | RECOMMENDED | `:47` | 规则 12 的引导句仍写"接口三个："，紧随的表有四行（`request_context` / `push_context` / `pop_context` / `consume_action`）；AC 11 按七个计数（上轮第 4 条，未修） | 本文件 `:47-54`；AC 11（`:224`） |
| 3 | RECOMMENDED | `:79`、`:171` | 局内 HUD 的接口列仍只写"菜品卡的悬停、点击、右键"，没有 `ui_cancel`——而规则 2 的修法正是建立在"HUD 用它关闭结束取餐面板"上，HUD 是 Run 内 `ui_cancel` 的唯一读者（`in-game-hud.md:171`、`:391`）（上轮第 7 条，未修） | 本文件规则 2（`:37`）、AC 2（`:215`）；`in-game-hud.md:59` |
| 4 | RECOMMENDED | `:208` | Cross-References 仍以"**None — this system references no other GDD.**"开篇，正文却引用 `scene-flow.md`（`:145`）、`in-game-hud.md`（`:70`）、`audio-feedback.md`（`:186`）（上轮第 8 条，未修） | design-docs「Dependencies 必须双向」 |
| 5 | RECOMMENDED | `:222`（AC 9） | THEN 断言"源码搜索 `device ==` 命中 0 次"，比规则 9 与 ADR D8 都严：D8 只禁止 `device == 0` 参与判断，并允许 `DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD` 用于日志与诊断——一句合法的诊断比较会让本 AC 失败。应改为搜索 `device == 0`（或 `device ==` 且同一行出现判断分支） | ADR-0005 D8（`adr-0005:219`）；本文件规则 9（`:44`） |
| 6 | RECOMMENDED | `:231`（AC 18） | `GIVEN F8，WHEN 调用 request_context(&"Menu", true)，THEN …… context_changed 发出一次` 缺前置状态：F8 若已在 `Menu` 且栈空，ADR D1 规定"有效上下文或深度改变时发一次；否则不发"，本调用不发声，AC 不可复现。同一条规则 12（`:56`）把"唯一无信号路径"限定为 `clear_stack = false` 的同值调用，也没覆盖"`clear_stack = true` + 空栈 + 同值" | ADR-0005 D1（`adr-0005:114`、`:125`） |
| 7 | RECOMMENDED | `:234`（AC 21） | "记 `STACK_FULL`""记 `UNKNOWN_ACTION`"仍只能靠日志捕获：公开接口只返回 `bool`，`ContextSnapshot` 无 reject 位，`enum Reject` 无读取口（上轮第 5 条，未修）。要么补一个可读的 reject 字段，要么把断言降为"返回 `false` 且状态不变" | ADR-0005 Key Interfaces（`adr-0005:288-300`）、`enum Reject`（`:272`） |
| 8 | RECOMMENDED | `:54`、`:56` | 规则 12 只写 `consume_action`"在 `Blocked` 上下文中取走"，没写 ADR D4 的"其它上下文调用返回 `false`、不置位、一帧内只记一次 WARNING"，也没写 D3 的"非 `Blocked` 传 `allow` 校验通过但不生效并记 WARNING"；AC 集合里没有一条覆盖这两处越界调用 | ADR-0005 D4（`adr-0005:166`）、D3（`:145`） |

### 已核对且无问题（记录以免重复检查）

- **五上下文响应集 vs D3 允许表**：Blocked = `ui_pause` ∪ `allow`；Menu = 四向 + `accept` + `cancel`；Run = 八个；Paused = 四向 + `pause` + `accept` + `cancel`；Modal = 四向 + `accept` + `cancel`。逐行与 `adr-0005:141-146` 相等（Paused 缺 `game_skip_wave`、Menu 缺 `game_skip_wave` / `ui_pause`）。
- **Esc 逐上下文翻译**：Run / Paused → `ui_pause`；Menu / Modal / Blocked → `ui_cancel`；被淘汰动作不进动作集——与 D3 末两条（`:151`、`:152`）同义。Paused 的 `ui_cancel` 只由 Backspace 触发，与 `ui-flow.md` 规则 7 ⑧ 自洽。
- **`pop_context` 空栈不弹基底、失败不发信号、不加栈**：与 D1 表（`:118`、`:120`）一致；顶层切换 `clear_stack = true` 与 D2 的"顶层切换不弹栈"一致。
- **AC 11 的接口面**：七个运行时接口 + `attach(tuning, sink)` + `get_context_snapshot()` + `register_hover_target` / `unregister_hover_target` 与 ADR Key Interfaces（`adr-0005:263-291`）逐个对上，不多不少；`ContextSnapshot` 含 `input_seq`、不含 `InputEvent`。ADR 侧 Validation Criteria（`:484`）仍漏悬停注册，属 ADR 侧遗留（上轮第 9 条）。
- **AC 编号**：1–21 连续、无重复、无跳号；AC 2 的首局引导 `Blocked`/Esc 用例与 ADR-0020（`:373`）一致。
- **其它静态扫描断言**：AC 10（InputMap 单点写入、8 动作、0 重绑入口）、AC 20（上下文只经三接口改变、`clear_stack := true` 唯一调用方为场景与流程管理）与 D9 表（`:226-236`）一致。
- **Completeness（standard 层）**：Overview、Detailed Design、Formulas、Edge Cases、Dependencies、Acceptance Criteria 6/6 齐全且非空；Player Fantasy、Tuning Knobs 为 advisory（`### 调参起点` 覆盖后者）。
- **Dependency Graph**：上游无（正确）；下游六份均存在并回指：`ui-flow.md`、`run-flow.md`（`:324` 记 `Depends On` 含「输入」）、`in-game-hud.md`、`first-run-onboarding.md`、`audio-feedback.md`、`scene-flow.md`（`:163`、`:216`）。`systems-index.md:59-75` 第 5、15、16 行确已记「输入」，第 3 行记「数据配置, 输入」，与 `:176` 的自述相符。

### Registry Values

`entities.yaml` 中本系统只拥有两个条目（`:336-350` `nav_steps`、`:352-364` `stick_engaged`）。对本文件 `:91`、`:100`、`:102`、`:104`、`:110`、`:118`、`:122`、`:136-139` 逐行比对：`nav_steps` 表达式 `1 + max(0, floor((T - D) / I) + 1)`、输出 `[1, +∞)`；`stick_engaged` 表达式 `(m >= E) or (stick_engaged_prev and m >= R)`、E = 0.60 / R = 0.40、约束 `R <= E - 0.10`——**无冲突**。本轮改动不涉及数值。

### Verdict: NEEDS REVISION

**NEEDS REVISION**（1 blocking · 7 recommended）。上轮 3 个 blocking 确认全部修好，"深度 2"与 `MAX_MODAL_DEPTH = 2`、ADR-0005 D1/D2 现在完全一致（含基底不计数）。唯一 blocking 是信号归属：`context_changed` 的订阅者被写成场景与流程管理，与 ADR-0005 D9 / ADR-0004 D9 冲突，删掉该订阅（或改记为 `UIRoot` + `AppRoot`，并把首局引导补进 `:80`）即可。

- 建议 `systems-index` 状态：**Needs Revision**（第 2 行当前 `Approved`；本次未改索引与文件头，由编排会话统一处理）。
- 本 GDD 未被修改，只追加本日志条目。

Reviewed-Content-Hash: design/gdd/input.md 1ef730801dde50820c7d8e3f99ecd31ac476d2c6
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/scene-flow.md 1a984f827d11dd07627e2c82f3b97d4ed58cd93b
Reviewed-Content-Hash: design/gdd/in-game-hud.md b8ead1263039519de2ff2ad9378fe43a8ea58373
Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a

---

## Review — 2026-11-17 — Verdict: APPROVED（第 4 轮：`context_changed` 订阅修复后的全文复审）

Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 7
Summary: 全文 lean 复审（非增量），按当前字节重跑 Phase 0–4。上轮唯一 blocking 已按建议修好且是**完整**修复：规则 2 的 Run 允许集含 `ui_cancel`（Backspace）而只有 Esc 产生的那个读到 `false`、状态表 Run 行同步、局内 HUD 交互行补上 `ui_cancel`、交互表与依赖表两处改为「回执是返回值，不订阅 `context_changed`（ADR-0005 D9）」；规则 12 的 `depth = stack.size()`（不含基底）与 AC 21 的「第 1 次深度 1 / 第 2 次深度 2 并发 `context_changed(Modal, Modal)` / 第 3 次 `STACK_FULL`」与 ADR-0005 D1/D2、`ui-flow.md:51` 的 `MAX_MODAL_DEPTH = 2` 逐值一致，上两轮的修复没有回退。本轮新增 0 个 blocking、1 个新 recommended（`context_changed` 删掉唯一被点名的订阅者后，两张表都不再记录该信号的任何读者），其余 6 条为沿用未修的旧 recommended。判据无一条不可通过，接口面与动作集无形状冲突，故判 APPROVED。
Prior verdict resolved: Yes —— 2026-11-17 第三轮的唯一 blocking（把场景与流程管理写成 `context_changed` 订阅者）已在交互表 `:76` 与依赖表 `:174` 两处同时改为否定式并注明 ADR-0005 D9 的理由，无残余。上轮 7 条 recommended 中仅第 3 条（HUD 接口列）在交互行 `:79` 修好、依赖行 `:171` 未同步，其余原文复发（见下表）。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/input.md`、`run-flow.md`、`in-game-hud.md`、`scene-flow.md`、`menu-waves.md`、`adr-0013`；`UNCHANGED`：`entities.yaml`、`SKILL.md`、`design-docs.md`、`adr-0004`、`adr-0005`、`adr-0020`、`ui-flow.md`、`first-run-onboarding.md`、`audio-feedback.md`、`game-concept.md`、`systems-index.md`、`project.yaml` 与三份引擎参考。上一份 NEEDS REVISION（2026-11-17 第三轮）绑定 `1ef73080…`，当前字节为 `4ac257e9…`，目标文档与多份跨文档事实均变动，故按全文复审。头部 `Status` 现为「In Review — 2026-11-17 修订：处理第三轮 lean 复审的 1 条 blocking……待第四轮 lean 复审」，与第三轮日志一致，只记录不改。

### 上轮 blocking 的修复核对（三项指定对照）

- **① 订阅归属（`:76`、`:174`）已修好。** 交互表改为「下游请求上下文 | `request_context(ctx, clear_stack := true)`；回执是返回值，**不订阅 `context_changed`**（ADR-0005 D9：自己触发、自己响应会形成重入回环）」；依赖表改为「顶层状态切换时调用 `request_context(default_ctx, clear_stack := true)` 并读返回值（**不订阅 `context_changed`**，ADR-0005 D9）」。全文 `grep context_changed` 命中 7 行：两处否定式，其余 5 行（`:56`、AC 18/19/21）都是**输入自己发射**该信号的断言，不是订阅。与 ADR-0005 D9（`adr-0005:234`：`SceneFlow` 不订阅、回执即返回值）及 ADR-0004 D9（`adr-0004:228`：订阅者仅 `UIRoot` / `AppRoot`）现在一致。
- **② 深度语义（`:56`）与 AC 21（`:234`）已修好且可测。** 规则 12 写「`MAX_CONTEXT_DEPTH = 2`，且 **`depth = stack.size()` 不含基底**」，`push_context` 在 `depth == 2` 返回 `false` 记 `STACK_FULL`、不排队不替换；「深度 2」覆盖单模态（深度 1）与 `SettingsModal` 上再压 `ConfirmModal`（深度 2，同值压栈同样加深并发 `context_changed(Modal, Modal)`），首局引导只压 1 层 `Blocked`。逐值对上 ADR-0005 D1（`:109` `depth = stack.size()`、`:116` 无条件压栈、`:121` 满栈拒绝）、D2（`:129` 同值压栈加深）与 `ui-flow.md:51`。AC 21 按「1 → 1、2 → 2 发 `(Modal, Modal)`、3 → `false`/`STACK_FULL`/深度仍 2/不发信号」重写，另补 `UNKNOWN_ACTION` 与「多弹不弹基底」，不再是「压一层即深度 2」的自相矛盾写法。
- **③ Run 的 `ui_cancel`（`:37`、`:64`、`:79`）已修好且两行一致。** 规则 2 把 `false` 严格限定到**由 Esc 产生的** `ui_cancel`，并写明「这不等于 Run 不接受 `ui_cancel`：Run 的允许集含 `ui_cancel`，Backspace 可产生它，HUD 用它关闭「结束取餐」确认面板」；状态表 Run 行列出导航 / `ui_accept` / `ui_cancel`（Backspace）/ `game_skip_wave` / `ui_pause`，与 ADR-0005 D3（`:146` 八个动作全部）等价；交互表局内 HUD 行现含 `ui_cancel`（仅关闭结束取餐确认面板），与规则 2 的语义**同一口径**，且与 `in-game-hud.md:59` 规则 16、`:503`/`:506` AC 25/27 及 ADR-0005 D9（`:231`）逐句对上。Paused 行的 `ui_cancel`（打开退出确认，Backspace）与 `ui-flow.md:55` 规则 7 ⑧ 也一致。

### Findings

| # | 严重度 | 位置 | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | RECOMMENDED | `design/gdd/input.md:56`、`:231`（AC 18） | 规则 12 写「**只有** `request_context` 在目标上下文与当前栈顶相同且 `clear_stack = false` 时才是无信号空操作」，比 ADR-0005 D1 窄：D1 的发射条件是「有效上下文或深度改变时发一次，否则不发」，因此 `request_context(同一上下文, clear_stack = true)` 且栈已空时**也不发**（`adr-0005:114`）。同源的 AC 18「GIVEN F8，WHEN 调用 `request_context(&"Menu", true)`，THEN …… `context_changed` 发出一次」缺起始状态：F8 若起点已是 `Menu` 且栈空，正确实现不发声，该 THEN 不可复现（沿用上轮第 6 条，未修；本轮可换成「GIVEN F8 处于 `Run`」即闭合） | ADR-0005 D1（`adr-0005:114`、`:125`）；本文件规则 12（`:56`） |
| 2 | RECOMMENDED | `:171` | 依赖表的局内 HUD 行仍只写「菜品卡的悬停、点击、右键」，未含 `ui_accept` / `game_skip_wave` / `ui_cancel`；同一份文档的交互表 `:79` 已列全。两张表对同一接口给出不同摘要，Dependencies 是下游实现者的速查面（上轮第 3 条只修了 `:79`，本行未同步） | 本文件 `:79`、规则 2（`:37`）；`in-game-hud.md:59`、`:391` |
| 3 | RECOMMENDED | `:208` | Cross-References 仍以「**None — this system references no other GDD.**」开篇，正文却引用 `scene-flow.md`（`:145`、`:174`）、`in-game-hud.md`（`:37`、`:70`）、`audio-feedback.md`（`:186`）（上轮第 4 条，连续三轮未修） | `.claude/rules/design-docs.md`「Dependencies 必须双向」 |
| 4 | RECOMMENDED | `:222`（AC 9） | THEN 断言「源码搜索 `device ==` 命中 0 次」，比规则 9 与 ADR-0005 D8（`:219`）都严：D8 只禁 `device == 0` 参与判断，允许 `DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD` 用于日志与诊断；一句合法诊断比较即令本 AC 失败（上轮第 5 条，未修） | ADR-0005 D8（`adr-0005:219`）；本文件规则 9（`:44`） |
| 5 | RECOMMENDED | `:234`（AC 21） | 「记 `STACK_FULL`」「记 `UNKNOWN_ACTION`」仍只能靠日志捕获：公开接口只返回 `bool`，`ContextSnapshot` 无 reject 位，`enum Reject` 无读取口（上轮第 7 条，未修）。要么补可读 reject 字段，要么把断言降为「返回 `false` 且栈/上下文不变」 | ADR-0005 Key Interfaces（`adr-0005:288-300`）、`enum Reject`（`:272`）；design-docs「验收标准必须可测」 |
| 6 | RECOMMENDED | `:54`、`:56` | 规则 12 只写 `consume_action`「在 `Blocked` 上下文中取走」，未写 ADR-0005 D4 的「其它上下文调用返回 `false`、不置位、一帧内只记一次 WARNING」，也未写 D3 的「非 `Blocked` 传 `allow` 校验通过但不生效并记 WARNING」；AC 集合无一条覆盖这两处越界调用（上轮第 8 条，未修） | ADR-0005 D4（`adr-0005:166`、`:168`）、D3（`:150`） |
| 7 | RECOMMENDED | `:76`、`:79`、`:80` | 修好 blocking 的代价：删掉场景与流程管理这一唯一被点名的订阅者后，本文件不再记录 `context_changed` 的任何读者。ADR-0004 D9（`:228`）的订阅者是 `UIRoot`（模态深度断言）与 `AppRoot`（日志），两张表都没有它们；实现者只读本 GDD 会以为该信号无人消费，`UIRoot` 的 `_modal_context_depth` 断言（ADR-0005 D2、ADR-0003 D4）因此失去出处。建议在交互表补一行「界面流程：订阅 `context_changed` 做模态深度断言」或明确指向 ADR-0004 D9 | ADR-0004 D9（`adr-0004:228`）、ADR-0005 D2（`adr-0005:132`） |

### 已核对且无问题（记录以免重复检查）

- **五上下文响应集 vs ADR-0005 D3 允许表**：Blocked = `ui_pause` ∪ `allow`；Menu = 四向 + `accept` + `cancel`；Run = 八个；Paused = 四向 + `pause` + `accept` + `cancel`；Modal = 四向 + `accept` + `cancel`。逐行与 `adr-0005:142-148` 相等。
- **Esc 逐上下文翻译**：Run / Paused → `ui_pause`；Menu / Modal / Blocked → `ui_cancel`；被淘汰动作不进本帧动作集——与 D3 末条（`adr-0005:152`）同义；Blocked 的 `ui_pause`（P / Start）被显式排除在 Esc 翻译之外，与 D3（`:151`）一致。
- **接口面**：AC 11 的七个运行时接口 + `attach(tuning, sink)` + `get_context_snapshot()` + `register_hover_target` / `unregister_hover_target` 与 ADR-0005 Key Interfaces（`adr-0005:263-291`）逐个对上，不多不少；`ContextSnapshot` 含 `input_seq`、不含 `InputEvent`。
- **所有权静态扫描面**：AC 20（上下文只经三接口改变、`request_context(clear_stack := true)` 调用方仅限场景与流程管理）、AC 10（InputMap 单点写入、8 动作、0 重绑入口）与 ADR-0005 D9（`adr-0005:226-236`）一致；规则 12 表的「唯一调用方」列与 D9 的表逐行相等（`push_context` 双调用方、`pop_context` 双调用方、`consume_action` 仅首局引导）。
- **AC 编号与可测性**：1–21 连续无重号；AC 2 的五段 Esc/Backspace 用例与规则 2、D3、`ui-flow.md:55` 规则 7 全对得上；AC 19/20/21 的栈断言都可用 `get_context_snapshot()` 观测（深度、栈顶、`effective`）。
- **跨文档口径**：`in-game-hud.md:59`、`:171`、`:391`、AC 25/27 读 `ui_cancel` 与 ADR-0005 D9 的 HudHost 授权一致；`run-flow.md:120`、`:295`、`:374` 明确不读 `ui_pause`，与本文件 `:78`、`:170` 一致；`first-run-onboarding.md:44`、`:150`、`:412` 的 `push_context(Blocked, allow=[…])` / `pop_context()` 与本文件 `:52-54`、`:80` 一致；`audio-feedback.md` 规则 19（`:45`）「没有焦点音」与本文件 `:81`、`:186` 的「不发任何音效事件」一致。
- **`ui-flow.md:149`、`:372` 的同帧 precedence 引述**仍只指向「输入规则 2」，而规则 2 只定义 Esc 的单动作解析（同帧 `ui_accept` + `ui_cancel` 的裁定在 ADR-0005 D5，`adr-0005:177`）——这是 `ui-flow.md` 侧的遗留，本文件无法修改，不计入本文件 findings。
- **完整性（standard 层）**：`gdd-structure-check` 报 Overview、Detailed Rules、Formulas、Edge Cases、Dependencies、Acceptance Criteria 6/6 齐全且非空；Player Fantasy、Tuning Knobs 为 advisory 缺口（`### 调参起点` 覆盖后者实质）。
- **Dependency Graph**：上游无（Foundation 层，正确）；下游六份均存在且回指：`ui-flow.md`、`run-flow.md`、`in-game-hud.md`、`first-run-onboarding.md`、`audio-feedback.md`、`scene-flow.md`（`:163`、`:216`）。`:176` 对索引第 3、5、15、16 行的自述与 `systems-index.md:59-75` 相符。
- **Registry Values**：`entities.yaml` 中本系统拥有 `nav_steps`（`:335`）与 `stick_engaged`（`:350-364`）。对本文件 `:91`、`:100`、`:102`、`:104`、`:110`、`:118`、`:122`、`:136-139` 逐行比对：`nav_steps` 表达式与输出范围、`stick_engaged` 的 `(m >= E) or (stick_engaged_prev and m >= R)`、E = 0.60 / R = 0.40、约束 `R <= E - 0.10` 均**无冲突**；本轮改动不涉及数值。

### 判定说明

上轮 blocking 的三条修法都落在同一处根因上（信号归属），且是完整修复而非局部补丁：两处否定式、且都附了 ADR 依据，没有留下「半句订阅」的读法。深度语义与 AC 21 经两轮打磨已与 ADR-0005 D1/D2 和 `ui-flow.md:51` 完全同形，Run 的 `ui_cancel` 口径在规则、状态表、交互表三处一致。剩余 7 条均为文档面一致性：其中第 1 条最接近规则级不一致，但它不改变接口形状（ADR-0005 D1 是 Accepted 实现权威，且 AC 18 换一个起始状态即可复现），按「是否直接改变实现/信号形状或使 AC/规则不可通过」的判据定为 recommended。第 7 条是本轮修复带出的可见性缺口，建议随下一次实质修改一并补上。

### Scope Signal

Rough scope signal: S —— 单系统、两条纯函数公式、上游零依赖、下游六个查询方，无新增 ADR 需求（ADR-0005 已决定全部实现问题）。

### Verdict: APPROVED

- 建议 `systems-index` 状态：**Approved**（第 2 行当前仍记 `Approved`；本次未改索引、未改 GDD 与文件头，按并行约束只追加本日志）。
- 本 GDD 未被本会话修改，只追加本日志条目。
Reviewed-Content-Hash: design/gdd/input.md 4ac257e9cab7df2ce607f4063416a308b56dccb4
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/input.md 423ac2b02d68a816905cc5060da7541aa726980c

## Re-stamp — 2026-11-17（架构复审后）
仅同步闸门输入文件的当前字节（GDD 本体自上次复审后只改了 Status 头行）；未改规则、公式、AC、依赖、契约或平衡数值。

Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/in-game-hud.md 6d7a4700f0f8b718eeb3bf0cbe9db7993df1f43d
Reviewed-Content-Hash: design/gdd/input.md 423ac2b02d68a816905cc5060da7541aa726980c
Reviewed-Content-Hash: design/gdd/menu-waves.md af47aef125e14677602eeab6eed5ae98132c6113
Reviewed-Content-Hash: design/gdd/run-flow.md da67f644d65824cb6ae98081cea03f69f0e77a0d
Reviewed-Content-Hash: design/gdd/scene-flow.md 2439af499a5bdd48e2623ed7abe998d20c8b397c
Reviewed-Content-Hash: design/gdd/systems-index.md ccecd7af3ef6993252cb6901db7435d0985c0953
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 30b02733b0ec60c3e18d0acf82d8fd5b740909b9
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 5252deab0b23680664d07fe7ce1c7255e89240ed
Reviewed-Content-Hash: docs/architecture/tr-registry.yaml b26b2e0ff92eec3bda1b34779ed6dd850f0b4941
Reviewed-Content-Hash: docs/engine-reference/godot/breaking-changes.md 4bc55e16796664ff88c7582c9f08e3530f7c94d4
Reviewed-Content-Hash: docs/engine-reference/godot/current-best-practices.md a45cd2c39a8ec53eaa3e924f39f93594e62085fd
Reviewed-Content-Hash: docs/engine-reference/godot/modules/input.md 3406b1b813858898696e510ea0accfcc7df98af8
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
