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
