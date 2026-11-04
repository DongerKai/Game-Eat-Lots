# 音效反馈 (Audio Feedback) — Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none (lean — Phase 3b not run)
Blocking items: 8 | Recommended: 11
Summary: The core design is unusually strong for an audio spec — the cue catalogue is a real contract, the closed reason vocabulary makes every decision observable from a log, the determinism and "mute = drop" rules are testable, and the pillar discipline (loss never sounds like a reward, no focus tick) holds throughout. The blocking problems are almost entirely interface-level, and most were created by GDDs written *after* this one: the receipt roll is now defined per-field by `ui-flow` with a 0.3–1.5 s range (this GDD assumes one roll of ≤ 0.6 s), the HUD never adopted `dish_action_rejected`, run-flow never enumerates the `action_committed.action` token, and settlement (§11, Needs Revision) says its three events are mutually exclusive while this GDD plays print and result as two cues. One genuine internal defect remains: the per-level caps sum to 9, so the stated global limit of 10 and AC 7 / AC 9's "10 voices full" setup are unreachable.
Prior verdict resolved: First review

Findings:
- [BLOCKING] Formulas (`voice_admission` `:174-184`, example `:196`) / Acceptance Criteria 7 (`:493`), 9 (`:495`): the global voice limit of 10 is unreachable. `cap[]` sums to 9 (`Critical 1 + Lifecycle 2 + Gameplay 3 + UI 3`), no rule lets a level exceed its cap, so `active_count` can never reach 10 — yet AC 7 and AC 9 both set up "10 voices full" and the worked example relies on it. The admission table also runs step 7 (steal a strictly lower priority) before step 8 (Critical replaces Critical), so a new Critical with any older lower-priority voice present steals that voice and leaves two Critical voices active, breaking `cap[Critical] = 1`.
- [BLOCKING] Detailed Design / Interactions (`:109-110`) / Edge Cases (`:300-302`) / Acceptance Criteria 10 (`:499`): `number_roll_*` granularity is undefined and contradicts `ui-flow`. `ui-flow.md:67,166,343` emits `number_roll_started(field, from, to, duration_ms)` / `number_roll_finished(field, skipped)` **per field**, with every field sharing one duration. This GDD treats a receipt as one roll: one `pending_outcome`, `sfx.num.land` at "1 / 次滚动" (`:72`), and `≤ 10` ticks per roll (`:71`). With three receipt fields the implementation produces three land cues and up to 30 ticks, and `pending_outcome` clears on the first finish event. The GDD also never covers `number_roll_started` arriving without a matching `number_roll_finished` (a HUD/UI-flow bug), which leaves the outcome deferred indefinitely.
- [BLOCKING] Formulas (`roll_ticks` `:238-243`) / Variables: the declared roll duration `d ∈ 0 … 600 ms` and the derivation `MAX_TICKS = 600 / 60 = 10` conflict with the designed owner of roll timing. `ui-flow.md:67` defines `RECEIPT_ROLL_SECONDS` as 0.3–1.5 s, default 0.8 s (and 0 under `reduce_motion`). At the default 0.8 s the tick interval becomes 80 ms, and at 1.5 s it becomes 150 ms — still within `MAX_TICKS`, but the stated range is wrong and AC 20 only ever exercises 600 ms. Settlement (`settlement-and-entry-fee.md:370`) says ≤ 0.6 s, so #11 and #16 disagree with each other and this GDD silently inherits #11's side; #11 is currently Needs Revision.
- [BLOCKING] Detailed Design, Core Rule 12 (`:38`) / Edge Cases (`:299-300`) / Dependencies (`:351`): settlement's contract says the opposite of this GDD's timing model. `settlement-and-entry-fee.md:371` defines `settlement_started` / `broke_even` / `loss` as "三个事件互斥，不叠加", while this GDD treats `settlement_started` (Lifecycle, print) and the outcome (Critical, deferred) as two cues in one settlement — and the no-roll edge case (`:300`) starts the result in the `settlement_started` frame, guaranteeing the overlap settlement forbids. Neither document defines the emitter or the broadcast moment (this GDD's Open Question 1), and #11 is Needs Revision, so the whole settlement branch of the cue catalogue rests on an unstable contract.
- [BLOCKING] Dependencies (`:359`, `:366`) / Edge Cases (`:339`): the HUD never adopted the prerequisite contract. `in-game-hud.md` contains zero occurrences of `dish_action_rejected` or `number_roll_*`; its audio row (`in-game-hud.md:176,395`) says audio subscribes to "焦点移动、确认、拒绝、跳过意图" — including focus movement, which this GDD explicitly refuses to sound (rule 19). Worse, HUD rule 11 (`in-game-hud.md:49`) declines to submit a known-invalid dish action at all, so no upstream event exists for `sfx.eat.reject`; the cue is a dead id. `ui-flow.md` *did* adopt its side (`:89`, `:166`), so this is a one-sided gap.
- [BLOCKING] Detailed Design, cue catalogue (`:57-58`) / Edge Cases (`:292`): `action_committed.action` has no agreed vocabulary. This GDD accepts exactly `{eat, skip_wave, end_meal}` and drops anything else as `dropped:invalid`; `run-flow.md:33,131` defines `action_committed(action, payload)` but never enumerates `action`, and its input action is named `game_skip_wave`, with "结束取餐" as the *same* action on the last wave (`run-flow.md:35`) — not a separate `end_meal`. If run-flow emits its own names, `sfx.wave.skip` is silently dropped and `sfx.meal.end` never fires.
- [BLOCKING] Acceptance Criteria 6 (`:492`), 11 (`:500`) vs Core Rule 12 (`:38`) and AC 10 (`:499`): AC 6 asserts `sfx.settle.broke_even` is `admitted` the moment `broke_even` arrives with 8 voices active, and AC 11 asserts `sfx.settle.loss` is `admitted` on arrival in the same frame — but rule 5 and rule 12 say every outcome cue waits for the roll to finish or be skipped, and AC 10 tests exactly that deferral. As written, AC 6 and AC 11 fail against a correct implementation. They also never send `settlement_started` or a roll-finish event, so they do not describe a reachable state.
- [BLOCKING] Dependencies (`:352`, `:368`) / States (`:97`): the pause contract is a behaviour, not an event name, and scene-flow does not define it. This GDD requires that the audio path keep receiving `Paused` / resumed notifications while the scene tree is paused, and makes that a hard state contract. `scene-flow.md` defines no state-change signal and explicitly leaves the notification mechanism to an ADR (`:270`), and it is still In Review. `Open Questions` 6 acknowledges the implementation question but not that the upstream contract itself is undefined.

Recommended:
- [RECOMMENDED] Formulas (`duck_db` `:272`) vs Core Rule 18 (`:44`): rule 18 says ducking happens only while a Critical cue plays, but `env(t)` includes `settlement_started`, a Lifecycle cue. Pick one; the formula's broader envelope is probably the intent.
- [RECOMMENDED] Edge Cases (`:312-313`): `d = 0` ("no tick, no land") and `d < 60 ms` ("0 ticks, land at end") overlap at exactly 0. State the second as `0 < d < TICK_MS`.
- [RECOMMENDED] Acceptance Criteria: no criterion covers "减少重复音效" (`:451`), even though it changes `cooldown_ms`, `TICK_MS` and variant selection. Add one: with the option on, `d = 600` yields 5 ticks and two `ui.click` 30 ms apart are both `dropped:cooldown`.
- [RECOMMENDED] Acceptance Criteria 17 (`:509`): "派发时间在预算内" is not measurable from the log. Point it at AC 25's `≤ 0.25 ms` median / `≤ 1.0 ms` peak, or drop the phrase.
- [RECOMMENDED] Acceptance Criteria 22 (`:517`): the ducking AC cannot pass in the Vertical Slice — this GDD states there is no music or ambience (`:282`, `:408`). Mark it as deferred until a Music bus exists, or test it against an injected mock bus and say so.
- [RECOMMENDED] Dependencies (`:350-360`) / Cross-References (`:472-473`): statuses are stale and now understate the coupling. Settlement is `Needs Revision` in `systems-index.md:67`, not "Designed"; run-flow is `Approved`, not "Designed"; #15/#16 are `Designed` GDDs on disk, yet the Interactions table still calls them "未设计" (`:109-110`) and Cross-References says "未写" (`:472-473`).
- [RECOMMENDED] Dependencies (`:360`, `:367`) / Edge Cases (`:302`): name the cancellation event. `ui-flow.md:89,166` emits `receipt_closed(run_id, reason: restart | menu | quit)` specifically so a pending outcome can be cancelled; this GDD only says "收据关闭通知". Naming it makes the edge case testable and matches the upstream GDD.
- [RECOMMENDED] Open Questions 7 / registry: the event `broke_even` and the registered formula `broke_even` (`entities.yaml:625-633`, `expression: net_profit(R) >= 0`) share one identifier, and `run_settled` carries a `RunResult` this system must not read. Note in the registry discussion that cue/event names need their own namespace; no value conflict exists today.
- [RECOMMENDED] UI Requirements (`:451`): "变体轮换避免立即重复" is redundant against round-robin when a cue has ≥ 2 variants, and impossible with one. Say what happens with a single-variant cue.
- [RECOMMENDED] Player Fantasy: absent as a heading (advisory at `standard`); `## Game Feel` (`:431-440`) carries it well — a one-line cross-reference closes the gap.
- [RECOMMENDED] Tuning Knobs: absent as a heading (advisory at `standard`); the formulas already carry ranges, but a short table collecting `cooldown_ms`, `TICK_MS`, `MAX_TICKS`, `K`, `DUCK_DB` and the four caps would give a balance pass one place to look.
- [RECOMMENDED] Scope signal: `systems-index.md:213` records `S`. Three hard upstream contracts, two downstream contracts and six formula blocks put this at `M`. Index edit belongs to the orchestration session.

Advisory (not counted): Core Rule 4 says a duplicate lifecycle cue is "忽略并记 WARN", but admission step 4 records `DROP(dedupe)` with no WARN; state whether the warning lives in the decision path. `run_settled` has no cue but also no OPEN QUESTION tying it to settlement's open ownership question — worth one cross-reference. The GDD's own "双向一致核对" (`:378-384`) is now out of date: run-flow and settlement have both already been updated, and the index row is already `Designed`.

---

## Design Review: 音效反馈（Audio Feedback）

Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Design (accepted as
Detailed Rules), Edge Cases, Dependencies, Acceptance Criteria — plus Formulas,
because this system defines numeric rules (volume mapping, admission, cooldown,
tick count, pitch, ducking). All six are present.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:431-440`) states
  the intended feel ("声音是数字的影子，不是数字的前奏") and the failure tone.
- **Tuning Knobs** — absent as a heading; every constant carries a range and a
  source (data file / constant) inside Formulas.

### Dependency Graph

Every declared dependency has a GDD on disk.

Upstream:
- ✓ run-flow.md — exists (Approved)
- ✓ settlement-and-entry-fee.md — exists (**Needs Revision**, `systems-index.md:67`)
- ✓ scene-flow.md — exists (In Review)
- ✗ sound bible (`design/audio/`) — does not exist; Open Question 2 covers it

Downstream:
- ✓ in-game-hud.md — exists (Designed)
- ✓ ui-flow.md — exists (Designed)

Cited but not dependencies (correctly declared as such): input.md, save-system.md,
data-config.md, game-concept.md — all exist.

### Registry Values

Searched the whole document for the registry terms it touches — `net_profit`,
`broke_even`, `entry_fee`, `total_value`, `return_ratio`, `capacity_used`,
`run_id`, `run_id_alloc` — case-insensitive. Nine matching lines.

No numeric conflict:
- The worked examples `−85 → +19` (`:257`) and `104 − 85 = 19` match the
  registry's own note on `opt_net_profit` (`entities.yaml:1072-1075`) and
  settlement's F1.
- `broke_even`'s `≥ 0` semantics are never restated numerically here — this GDD
  only consumes the event, which is the correct boundary.

Gaps:
- **No registry entries exist for this system** (no cue type, no audio constants).
  `Open Questions` 7 acknowledges this; the registry's own rule that
  internal-only facts need no registration supports leaving it alone for now.
- **Name collision**: the signal `broke_even` shares an identifier with the
  registered formula `broke_even` (`entities.yaml:625-633`). No value conflict,
  but see Recommended.

### Required Before Implementation

1. **Fix the voice pool arithmetic and the admission ordering** — `cap[]` sums to
   `1 + 2 + 3 + 3 = 9` (`:190`), while `MAX_VOICES = 10` and the text claims one
   slot of headroom (`:34`, `:190`). No rule permits a level to exceed its cap,
   so `active_count` tops out at 9: AC 7 (`:493`), AC 9 (`:495`) and the example
   at `:196` all describe an unreachable state, and `dropped:pool_full` can only
   ever mean "this level is full", never "the pool is full". Decide which limit
   is real — either drop `MAX_VOICES` and make the per-level caps the whole
   story, or let one level use the spare slot and say which. Then reorder
   `voice_admission`: step 8 (Critical replaces Critical) must run before step 7
   (steal a lower priority), otherwise a new Critical with any older
   lower-priority voice present steals a UI voice and leaves two Critical voices
   alive, breaking `cap[Critical] = 1` and rule 8's "Critical 只能被另一个
   Critical 抢占". Rewrite AC 7 / AC 9 around a reachable configuration
   (e.g. 9 voices: Lifecycle 2 + Gameplay 3 + UI 3 + Critical 1).

2. **Define roll granularity with `ui-flow`** — `ui-flow.md:67,166,343` emits
   `number_roll_started` / `number_roll_finished` **once per field**, all fields
   sharing one duration; this GDD counts one roll per receipt (`pending_outcome`
   single-slot, `sfx.num.land` "1 / 次滚动", `≤ 10` ticks per roll). Three
   receipt fields therefore produce three finish events, three land cues and up
   to 30 ticks, and `pending_outcome` clears on the first field. Choose and
   write down one of: (a) the receipt aggregates its fields and emits one
   roll-start / roll-finish pair — requires a `ui-flow` change; (b) this GDD
   waits for all fields named in the opening `number_roll_started` set; or
   (c) only the net-profit field drives audio, and the others are silent. Add
   the missing edge case "roll started, never finished" and give it a bounded
   fallback (the same-frame rule at `:300` already handles "no roll started").

3. **Reconcile the roll duration** — `d ∈ 0 … 600 ms` and `MAX_TICKS = 600/60 =
   10` (`:238-243`) assume settlement's ≤ 0.6 s. `ui-flow.md:67` sets
   `RECEIPT_ROLL_SECONDS` to 0.3–1.5 s, default 0.8 s, so the stated input range
   is wrong and `MAX_TICKS`'s derivation is wrong. `MAX_TICKS` still bounds the
   output correctly (`min(floor(d/TICK_MS), 10)`), but the GDD must state the
   real range and stop justifying 10 as "600 / 60". If settlement's 0.6 s is
   meant to win, that is a `ui-flow` change and belongs in `ui-flow`'s review —
   route it there rather than encoding it here.

4. **Resolve the settlement timing conflict with #11** — settlement defines
   `settlement_started` / `broke_even` / `loss` as mutually exclusive and
   non-stacking (`settlement-and-entry-fee.md:371`); this GDD plays print and
   outcome as two cues, and the no-roll edge case (`:300`) starts the result in
   the print frame. One of the two documents has to move: either settlement
   accepts two sequential cues and defines the emitters and broadcast moments
   (Open Question 1), or audio's `settlement_started` cue is dropped when an
   outcome follows. #11 is Needs Revision, so this is a live contract, not a
   settled one — the audio branch that depends on it should be flagged as
   provisional until #11's re-review closes.

5. **Get the HUD contract adopted or drop the cue** — `in-game-hud.md` never
   names `dish_action_rejected` or `number_roll_*`; its audio row offers
   "焦点移动、确认、拒绝、跳过意图" (`in-game-hud.md:176,395`), which both omits
   the roll events and includes focus movement that this GDD refuses to sound.
   HUD rule 11 (`:49`) also declines to submit an invalid dish action, so no
   upstream rejection exists for `sfx.eat.reject` to consume. Either add the
   semantic events to `in-game-hud.md` (a small upstream change) or remove
   `sfx.eat.reject` and the in-run roll rows from the cue catalogue. The two
   GDDs currently describe different systems.

6. **Pin the `action_committed.action` vocabulary with run-flow** — this GDD
   accepts `{eat, skip_wave, end_meal}` and drops everything else as `invalid`
   (`:292`). `run-flow.md:33,35,131` defines three actions but names only
   `game_skip_wave`, and treats last-wave "结束取餐" as the same action rather
   than a separate `end_meal`. As written, a conforming run-flow that emits
   `game_skip_wave` loses `sfx.wave.skip` to `dropped:invalid`, and
   `sfx.meal.end` can never fire. Either run-flow publishes the token set, or
   this GDD matches run-flow's names and merges `meal.end` into `wave.skip` with
   a differentiator.

7. **Rewrite AC 6 and AC 11 to respect the deferral rule** — both assert an
   outcome cue is `admitted` on arrival, which rule 5, rule 12 and AC 10
   contradict. Give each the same three-step shape as AC 10:
   `broke_even` arrives → `deferred:pending_outcome`; `number_roll_finished`
   arrives → `admitted` and the voice counts asserted afterwards. AC 11's
   "活跃 voice 数为 1" is only true after the roll finishes.

8. **Define the pause notification with scene-flow** — this GDD makes "audio
   keeps receiving `Paused` / resumed while the tree is paused" a hard state
   contract (`:352`, `:368`), but `scene-flow.md` defines no state-change signal
   and leaves the mechanism to an ADR (`scene-flow.md:270`). Before
   implementation, scene-flow must name the notification (signal or direct call)
   and guarantee it is delivered while paused; audio's `Frozen` state and both
   `flow.pause` / `flow.resume` cues depend on it. Track in Open Question 6 until
   scene-flow's re-review settles it.

### Recommended Revisions

9. **Ducking scope** — rule 18 (`:44`) says Critical only; `duck_db`'s envelope
   (`:272`) includes `settlement_started`. Align the rule to the formula unless
   the print cue is meant to be silent over music.
10. **`d = 0` vs `d < 60 ms`** — the two adjacent edge cases (`:312-313`)
    overlap at zero. Write the second as `0 < d < TICK_MS`.
11. **Add an AC for "减少重复音效"** — `:451` changes `cooldown_ms` × 2 and
    `TICK_MS` × 2 but no criterion exercises it. One GIVEN/WHEN/THEN covers it.
12. **AC 17's budget phrase** — "派发时间在预算内" is not measurable from the
    log; reference AC 25's numeric budget.
13. **AC 22 is untestable in the Vertical Slice** — there is no Music bus
    (`:282`, `:408`). Mark it deferred or specify an injected mock bus.
14. **Refresh dependency statuses and labels** — settlement is Needs Revision,
    run-flow is Approved, and #15/#16 are written GDDs. The "未设计" / "未写"
    labels (`:109-110`, `:472-473`) and the stale "双向一致核对" items
    (`:380-382`) make the file look older than the tree it sits in.
15. **Name `receipt_closed(run_id, reason)`** — `ui-flow.md:89,166` already
    emits it for exactly the cancellation this GDD describes generically at
    `:302`. Using the real event name makes the edge case implementable and
    testable.
16. **Registry naming** — note that cue ids and event names need a namespace
    distinct from formula names, since `broke_even` currently means both a
    formula and a signal. No value conflict; a tooling note is enough.
17. **Variant rotation under "减少重复音效"** — `:451` says "变体轮换避免立即
    重复", which round-robin already does with ≥ 2 variants and cannot do with
    one. Say what happens with a single-variant cue.
18. **Player Fantasy** — add a one-line reference to `## Game Feel`.
19. **Tuning Knobs** — collect `cooldown_ms`, `TICK_MS`, `MAX_TICKS`, `K`,
    `DUCK_DB` and the four caps in one advisory table.

### Specialist Disagreements

None — Lean mode runs no specialists (Phase 3b skipped).

### Nice-to-Have

- Core Rule 4 says a duplicate lifecycle cue is "忽略并记 WARN"; admission
  step 4 logs `DROP(dedupe)` without one. State where the warning is emitted.
- `run_settled` correctly produces no cue; add one line pointing at Open
  Question 1 so the ownership question is visible from the catalogue.
- The "双向一致核对" block (`:378-384`) has already been acted on by the
  orchestration session — run-flow and settlement both name this GDD now, and
  the index row is already `Designed`. Trim it to avoid re-reporting stale work.
- `sfx.run.end` and `sfx.settle.print` are both Lifecycle and can land in the
  same frame; with `cap[Lifecycle] = 2` this fits, but an explicit same-frame
  note in the catalog would help implementers.

### Scope Signal

Rough scope signal: **M** (3 hard upstream contracts + 2 downstream contracts,
6 formula blocks, no new ADR expected at design level — the pause
`process_mode` ADR is already anticipated in Open Question 6). The index records
`S`; producer should verify before sprint planning.

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/input.md dd15ea96523a49136b96cfef827c5619dee75e6a
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 182d446c25ab8aa50ccf7d672bc2132eeb2b22d3
