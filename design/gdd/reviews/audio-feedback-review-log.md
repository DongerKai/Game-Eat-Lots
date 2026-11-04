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

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none (lean — Phase 3b not run)
Blocking items: 3 | Recommended: 15
Summary: The revision closed most of the interface gaps: the roll is now counted per `number_roll_started`, its duration matches `ui-flow`'s 0.3–1.5 s, settlement now says the print cue and the result cue may coexist, run-flow publishes `eat` / `skip_wave` / `end_pickup`, scene-flow defines `state_changed(from, to)`, and AC 6 / AC 11 respect the deferral rule. Five of the eight prior blockers are closed outright. Three remain, two of them one-line residues: Core Rule 8 still caps Lifecycle at 2 while the Formulas table (and AC 7 / AC 9's "10 voices full") require 3, and the `action_committed` edge case still accepts `end_meal` while the catalogue accepts `end_pickup`. The third is new, created by this round's fix: three record/milestone cues were added to the catalogue without adopting the Approved producer's stated "三者互斥，不叠加" contract, and with no merge rule for a settlement that emits all of them.
Prior verdict resolved: No — 5 of 8 prior blocking items closed; #1 and #6 leave a one-line contradiction, #5 leaves the HUD producer unspecified

Findings:
- [BLOCKING] Core Rule 8 (`:34`) vs Formulas `cap[p]` (`:193`) / AC 7 (`:497`) / AC 9 (`:499`): the voice-cap arithmetic is still stated two ways. Rule 8 reads `Critical 1 / Lifecycle 2 / Gameplay 3 / UI 3` (sum 9); the Formulas table reads `Lifecycle 3` (sum 10, "等于 `MAX_VOICES`"). Only the 3-cap makes the "10 个 voice 全满" premise of AC 7 and AC 9 reachable, so an implementation built from the Core Rules tops out at 9 and both ACs fail. Fix is one word at `:34`.
- [BLOCKING] Edge Cases (`:295`): the accepted `action_committed.action` set is still `{eat, skip_wave, end_meal}`, while the catalogue (`:58`) and `run-flow.md:33` both name `end_pickup`. A conforming run-flow emission is rejected as `dropped:invalid` and `sfx.meal.end` never fires — the exact failure prior blocking #6 described, left on this one line when `:58` was corrected.
- [BLOCKING] 音效事件目录 (`:72-74`) / 同帧裁定顺序 (`:77`): the three newly added record cues do not adopt their producer's contract. `records-and-milestones.md:347` (Approved) states `record_broken` / `milestone_earned` / `career_completed` are "三者互斥，不叠加" and that same-frame multiple `milestone_earned` are merged into one play by this system. This GDD adds the three cues with no exclusion rule and omits them from `:77`'s same-frame ruling order, so one settlement can play print + record stamp + milestone + career-complete together, and a second same-frame milestone is dropped by the 120 ms cooldown rather than by a stated merge rule. (Records' own AC 20 asserts four separate emissions in one settlement, so the two documents disagree in both directions — audio owns playback policy and must state the resolution.)
- [RECOMMENDED] Dependencies (`:363`) / `in-game-hud.md:176` vs its Detailed Design: the HUD now names `dish_action_rejected` in its interface table, which closes the naming half of prior blocking #5, but no rule in HUD's Core Rules emits it. Rule 11 (`in-game-hud.md:54`) gives visual feedback for an inedible focus and declines to submit, and `:373` covers the run-flow rejection path — neither names the event. Route a producer rule to HUD's re-review before `sfx.eat.reject` is implementable.
- [RECOMMENDED] Dependencies (`:363`) / `in-game-hud.md:395,416`: the HUD still claims audio subscribes to "焦点移动、确认、拒绝、跳过意图". This GDD's rule 19 (`:45`) refuses any focus cue and `ui-flow.md:473` states the same. Remove the focus-movement wording in HUD.
- [RECOMMENDED] Acceptance Criteria 10 (`:503`): "两条 cue 的播放区间不重叠" is not verifiable from this GDD. The print cue's duration belongs to the sound spec and is unstated, and `ui-flow` permits a 0.3 s roll, so print and result can overlap at the short end. Assert only "result cue starts at or after the last `number_roll_finished`", or bound the print cue below 0.3 s in the sound spec.
- [RECOMMENDED] Detailed Design, Core Rule 12 (`:38`): "结果 cue 只在**最后一个** `number_roll_finished` 后起播" has no mechanism. `ui-flow.md:347` shares one duration `D` across all fields, so every field's finish lands in the same frame. State the rule as "track the open-roll set; fire when it empties". The "started but never finished" fallback exists only inside rule 12 — it deserves an Edge Cases entry.
- [RECOMMENDED] Edge Cases (`:315-316`): `d = 0` ("no tick, no `land`") and `d < 60 ms` ("0 ticks, `land` at end") still overlap at exactly zero. Write the second as `0 < d < TICK_MS`. Carried over from the prior review.
- [RECOMMENDED] Formulas `duck_db` (`:275`) vs Core Rule 18 (`:44`): rule 18 says ducking happens only while a Critical cue plays; `env(t)` includes `settlement_started`, a Lifecycle cue. Carried over.
- [RECOMMENDED] Acceptance Criteria 22 (`:521`): the ducking AC cannot pass in the Vertical Slice — `:285` and `:412` state there is no music or ambience. Mark it deferred or test against an injected mock bus. Carried over.
- [RECOMMENDED] Acceptance Criteria 17 (`:513`): "派发时间在预算内" is still not measurable from the log; point it at AC 25's `≤ 0.25 ms` median / `≤ 1.0 ms` peak. Carried over.
- [RECOMMENDED] Acceptance Criteria: still no criterion for "减少重复音效" (`:455`), which changes `cooldown_ms`, `TICK_MS` and variant selection. Carried over.
- [RECOMMENDED] Dependencies (`:112-113`, `:366`, `:476-477`) / 双向一致核对 (`:384-388`): the stale labels remain and the reconciliation block is now wholly obsolete — run-flow, settlement, scene-flow and `systems-index.md` have all been updated already, and #15/#16 are written GDDs.
- [RECOMMENDED] Interactions (`:109-117`) / Cross-References (`:472-480`): 纪录与里程碑 is missing from both tables even though Dependencies now lists it as a hard upstream (`:356`). The bullet at `:380` puts the record cues under "不属于依赖的关系", which reads as a contradiction of the table above it.
- [RECOMMENDED] Edge Cases (`:305`) / Interactions (`:113`): `ui-flow.md:91` emits `receipt_closed(run_id, reason: restart | menu | quit)` and cites this GDD's edge case by that name. Name it here instead of "收据关闭".
- [RECOMMENDED] 音效事件目录 (`:73`): `sfx.milestone` is the only cue id that does not follow `sfx.<域>.<事件>`. `sfx.record.milestone` or `sfx.milestone.earned` matches the rest.
- [RECOMMENDED] 音效事件目录 (`:72`): `sfx.record.broken`'s 起播时机 "收据打开时" sits outside Core Rule 5's same-frame dispatch unless the upstream emits at receipt open. Say "事件同帧", or confirm the emitter with the records GDD.
- [RECOMMENDED] Scope signal: `systems-index.md:213` records `S`. Four hard upstream contracts (records added this round), two downstream contracts and six formula blocks put this at `M`. Index edit belongs to the orchestration session.

Advisory (not counted): `MAX_TICKS`' justification at `:243` now reads "1500 / 150 = 10", which mixes the cap with the derived interval — the real derivation is `min(floor(d / TICK_MS), 10)`. The signal `broke_even` still shares an identifier with the registered formula `broke_even`; no value conflict, but cue and event names need their own namespace. Player Fantasy and Tuning Knobs remain absent as headings (advisory at `standard`; `## Game Feel` `:435-444` and the Formulas tables cover them). `run-flow.md:377` still carries a stale "音效反馈 | 未写" row in its older summary table.

---

## Design Review: 音效反馈（Audio Feedback）

Specialists consulted: none (lean mode)
Re-review: Yes — prior verdict was NEEDS REVISION on 2026-11-04

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Design (accepted as
Detailed Rules), Edge Cases, Dependencies, Acceptance Criteria — plus Formulas,
because this system defines numeric rules (volume mapping, admission, cooldown,
tick count, pitch, ducking). All six are present.

Advisory at `standard`:
- **Player Fantasy** — still absent as a heading; `## Game Feel` (`:435-444`)
  carries it.
- **Tuning Knobs** — still absent as a heading; constants carry ranges and
  sources inside Formulas.

### Prior Blocking Disposition

| # | Prior blocking | Status |
|---|---|---|
| 1 | Voice pool arithmetic / admission order | **Partial** — step order fixed (`:185-186`); Formulas sums to 10 (`:193`); Core Rule 8 still says Lifecycle 2 (`:34`) |
| 2 | Roll granularity vs `ui-flow` | **Closed** — rule 12 (`:38`) and the catalogue (`:71`) now count per `number_roll_started`; residual is the "last finish" mechanism |
| 3 | Roll duration vs `ui-flow` | **Closed** — `d ∈ 0 … 1500 ms` (`:241`) matches `RECEIPT_ROLL_SECONDS` 0.3–1.5 s |
| 4 | Settlement timing conflict | **Closed** — `settlement-and-entry-fee.md:371` now states print and result may coexist; `:340` adopts the roll-end timing |
| 5 | HUD never adopted the contract | **Partial** — `in-game-hud.md:176` names all three events; no rule emits `dish_action_rejected`, and `:395,416` still claim focus audio |
| 6 | `action_committed.action` vocabulary | **Partial** — `run-flow.md:33` publishes `eat` / `skip_wave` / `end_pickup`; Edge Cases `:295` still accepts `end_meal` |
| 7 | AC 6 / AC 11 vs the deferral rule | **Closed** — both now go `deferred:pending_outcome` → `number_roll_finished` → `admitted` |
| 8 | Pause notification is a behaviour | **Closed** — `scene-flow.md:41` defines `state_changed(from, to)`; the delivery mechanism stays with the ADR, which Open Question 6 already covers |

### Dependency Graph

Every declared dependency has a GDD on disk. 纪录与里程碑 is newly listed and
exists.

Upstream:
- ✓ run-flow.md — exists (Approved)
- ✓ settlement-and-entry-fee.md — exists (Needs Revision); its timing clause is now aligned
- ✓ scene-flow.md — exists (In Review)
- ✓ records-and-milestones.md — exists (Approved)
- ✗ sound bible (`design/audio/`) — does not exist; Open Question 2 covers it

Downstream:
- ✓ in-game-hud.md — exists (Approved)
- ✓ ui-flow.md — exists (Needs Revision)

Cited but not dependencies (correctly declared as such): input.md, save-system.md,
data-config.md, game-concept.md — all exist.

### Registry Values

`design/registry/entities.yaml` carries no audio, cue, voice or SFX entry, so
this system owns no registered values. The whole-document search for the
registry terms it touches (`net_profit`, `broke_even`, `entry_fee`,
`total_value`, `return_ratio`, `capacity_used`, `run_id`) printed 24 lines.

No numeric conflict:
- `net_profit` is only used as an example range (`start = −85`, `end = 19`,
  `:267`, `:519`); the GDD never restates its formula or value.
- `run_id` is typed `int64` (`:131`), matching the registry.
- `broke_even` is consumed as an event and never derived — the correct boundary.

Gaps:
- **No registry entries exist for this system.** Open Question 7 covers it.
- **Name collision**: the signal `broke_even` shares an identifier with the
  registered formula `broke_even`. No value conflict; see Advisory.

### Required Before Implementation

1. **Fix the Lifecycle cap in Core Rule 8** — `:34` says `Lifecycle 2` (sum 9)
   while `:193` says `Lifecycle 3` (sum 10). AC 7 and AC 9 both need the 10-voice
   pool. One edit at `:34`; the example at `:199` and AC 6's "Lifecycle 2" setup
   remain valid because they describe an 8-voice configuration, not the cap.

2. **Change `end_meal` to `end_pickup` in the edge case** — `:295` is the last
   surviving copy of the old vocabulary. `run-flow.md:33` and the catalogue
   (`:58`) both say `end_pickup`; until this line matches, a conforming upstream
   emission is dropped as invalid and `sfx.meal.end` is unreachable.

3. **State the record-cue playback policy** — the three cues added at `:72-74`
   arrive without a rule. `records-and-milestones.md:347` (Approved) claims
   "三者互斥，不叠加" and same-frame milestone merging; that document's own AC 20
   asserts separate emissions. Decide in this GDD: either (a) `record_broken`
   and `career_completed` are Critical-adjacent and mutually exclusive with each
   other and with the settlement result, or (b) they may layer and the producer's
   wording is wrong. Either way, add them to `:77`'s same-frame ruling order and
   say whether N same-frame `milestone_earned` play once or N times — the 120 ms
   cooldown currently answers that silently.

### Recommended Revisions

4. **Get HUD to emit `dish_action_rejected`** — `in-game-hud.md:176` declares it
   but no rule produces it. A one-line rule 11 amendment ("HUD emits
   `dish_action_rejected(reason)` on a legal-but-failed dish action") closes it.
5. **Drop the focus-audio claim from HUD** — `in-game-hud.md:395,416` contradict
   rule 19 here and `ui-flow.md:473`.
6. **Make AC 10 testable** — assert the ordering (result starts at or after the
   last finish) rather than non-overlap, or bound the print cue's duration.
7. **Define the "last finish" mechanism** — count open rolls; add the
   "never finished" case to Edge Cases.
8. **`0 < d < TICK_MS`** — remove the overlap at zero (`:316`).
9. **Ducking scope** — align rule 18 with `duck_db`'s `env(t)` (`:44` vs `:275`).
10. **AC 22** — deferred, or an injected mock bus.
11. **AC 17** — point at AC 25's numbers.
12. **Add an AC for "减少重复音效"**.
13. **Refresh labels** — `:112-113`, `:366`, `:476-477`, and delete the obsolete
    "双向一致核对" block (`:384-388`).
14. **Add 纪录与里程碑 to Interactions and Cross-References**, and move it out of
    the "不属于依赖的关系" bullet (`:380`).
15. **Name `receipt_closed(run_id, reason)`** (`:113`, `:305`).
16. **Rename `sfx.milestone`** to match `sfx.<域>.<事件>`.
17. **Restate `sfx.record.broken`'s timing** as "事件同帧" or confirm the emitter.

### Specialist Disagreements

None — Lean mode runs no specialists (Phase 3b skipped).

### Nice-to-Have

- `MAX_TICKS`' "1500 / 150 = 10" (`:243`) describes the derived interval, not the
  derivation; write `min(floor(d / TICK_MS), 10)`.
- Cue and event names still need a namespace distinct from formula names
  (`broke_even`).
- The record cues are also absent from `:77`'s same-frame ruling order, even
  after blocking item 3 is resolved in the catalogue.
- `run-flow.md:377` still has a stale "音效反馈 | 未写" row.

### Scope Signal

Rough scope signal: **M** — four hard upstream contracts, two downstream
contracts, six formula blocks, no new ADR expected at design level (the pause
`process_mode` ADR is already anticipated in Open Question 6). The index records
`S`.

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 5734623b4a60d9072657ef631c8ae61f05359f21
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 9fe5183795ca3d47798510a1d00a040dabe3ca61
Reviewed-Content-Hash: design/gdd/systems-index.md 0c3efa52308253f2243c0771bff7c9b6d3ae5933

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 17
Summary: All three prior blocking items are closed. Core Rule 8 (`:34`) now reads `Critical 1 / Lifecycle 3 / Gameplay 3 / UI 3` (sum 10), matching the Formulas table (`:195`) and making AC 7 / AC 9's "10 voices full" setup reachable; the edge case (`:297`) now accepts `end_pickup`, matching the catalogue and `run-flow.md:33`; and the record cues (`:72-75`, plus the new merge paragraph at `:79`) adopt the Approved producer's contract — `record_broken` and `milestone_earned` may coexist, `career_completed` closes the frame, and same-frame milestones collapse to one play, which is now also what `records-and-milestones.md:347` and its AC 20 describe. The single new blocking defect is in that merge rule's own wording: it records `suppressed:outcome_priority`, a reason the closed vocabulary at `:144` does not contain, while that table forbids adding a reason without editing it first and AC 26 asserts every logged reason is in the vocabulary. Everything else outstanding is carried-over recommended work, most of it in upstream files.
Prior verdict resolved: Yes — 3 of 3 prior blocking items closed (#1 and #2 outright; #3 on substance, with one new one-line defect in its wording)

Findings:
- [BLOCKING] 音效事件目录 (`:79`) vs 封闭原因词表 (`:144`) / AC 26 (`:530`): the new merge rule records `suppressed:outcome_priority`, but `:144` defines `suppressed` reasons as only `duplicate_outcome` and `preempted`, and states "任何新原因必须先改本表，不能临时加字符串——否则测试无法穷举". AC 26 asserts every logged `decision` / `reason` falls inside the closed vocabulary, so the first settlement that completes the career fails AC 26 and the reason cannot be enumerated by a test. Add `outcome_priority` to the `suppressed` list at `:144` (preferred over reusing `preempted`, which already means "cancelled by a higher-priority event").
- [RECOMMENDED] Dependencies (`:365`) / `in-game-hud.md:176`: the naming half of the old HUD gap is closed, but no rule in the HUD emits `dish_action_rejected` — rule 11 (`in-game-hud.md:54`) declines an inedible focus and rule `:373` covers the run-flow rejection path, and neither names the event. `in-game-hud.md` is unchanged this round. Until the HUD re-review adds the producer rule, `sfx.eat.reject` is still a dead cue.
- [RECOMMENDED] Dependencies (`:365`) / `in-game-hud.md:395,416`: the HUD still claims audio subscribes to "焦点移动、确认、拒绝、跳过意图". Rule 19 (`:45`) refuses any focus cue and `ui-flow.md:473` states the same. Remove the focus-movement wording in the HUD.
- [RECOMMENDED] Acceptance Criteria 10 (`:505`): "两条 cue 的播放区间不重叠" is not verifiable from this GDD — the print cue's duration lives in the sound spec and is unstated, and `ui-flow.md:69` permits a 0.3 s roll, so print and result can overlap at the short end. Assert only "result cue starts at or after the last `number_roll_finished`", or bound the print cue below 0.3 s in the sound spec.
- [RECOMMENDED] Detailed Design, Core Rule 12 (`:38`): "结果 cue 只在**最后一个** `number_roll_finished` 后起播" still has no mechanism — `ui-flow.md:347` shares one duration `D` across all fields, so every field finishes in the same frame. State it as "track the open-roll set; fire when it empties". The start-without-finish fallback exists only inside rule 12 and still deserves an Edge Cases entry.
- [RECOMMENDED] Edge Cases (`:317-318`): `d = 0` ("no tick, no `land`") and `d < 60 ms` ("0 ticks, `land` at end") still overlap at exactly zero. Write the second as `0 < d < TICK_MS`. Carried over.
- [RECOMMENDED] Formulas `duck_db` (`:277`) vs Core Rule 18 (`:44`): rule 18 says ducking happens only while a Critical cue plays; `env(t)` includes `settlement_started`, a Lifecycle cue. Carried over.
- [RECOMMENDED] Acceptance Criteria 22 (`:523`): the ducking AC cannot pass in the Vertical Slice — `:287` and `:414` state there is no music or ambience. Mark it deferred or test against an injected mock bus. Carried over.
- [RECOMMENDED] Acceptance Criteria 17 (`:515`): "派发时间在预算内" is still not measurable from the log; point it at AC 25's `≤ 0.25 ms` median / `≤ 1.0 ms` peak. Carried over.
- [RECOMMENDED] Acceptance Criteria: still no criterion for "减少重复音效" (`:457`), which changes `cooldown_ms`, `TICK_MS` and variant selection. Carried over.
- [RECOMMENDED] Dependencies (`:384-390`) / Interactions (`:114-115`) / Cross-References (`:478-479`): the "双向一致核对" block is wholly obsolete — all four named documents already carry the audio row, and #15 / #16 are written GDDs — while the Interactions table still labels them "（#15，未设计）" / "（#16，未设计）" and Cross-References still says "（未写）". Delete the block and refresh the four labels.
- [RECOMMENDED] Interactions (`:109-119`) / Cross-References (`:472-482`): 纪录与里程碑 is absent from both tables even though Dependencies (`:358`) lists it as a hard upstream and `:382` cites the three cue mappings. Add the row to both.
- [RECOMMENDED] Edge Cases (`:307`) / Interactions (`:115`): `ui-flow.md:168` emits `receipt_closed(run_id, reason: restart | menu | quit)` and `:91` cites this GDD's edge case by that name. Name the event here instead of "收据关闭".
- [RECOMMENDED] 音效事件目录 (`:73`): `sfx.milestone` is the only cue id that does not follow `sfx.<域>.<事件>`. `sfx.record.milestone` or `sfx.milestone.earned` matches the rest.
- [RECOMMENDED] 音效事件目录 (`:72`): `sfx.record.broken`'s 起播时机 "收据打开时" sits outside Core Rule 5's same-frame dispatch unless the upstream emits at receipt open. Say "事件同帧", or confirm the emitter with the records GDD.
- [RECOMMENDED] Acceptance Criteria: the merge/exclusion policy at `:79` has no criterion. Add one settlement fixture — `record_broken` + two `milestone_earned` + `career_completed` in one frame — asserting `sfx.career.complete` `admitted`, the other three `suppressed:outcome_priority`, and a second fixture with two milestones and no career completion asserting exactly one `sfx.milestone`. Without it, the new rule is only prose.
- [RECOMMENDED] Formulas (`:230`) / `voice_admission` step 4 (`:184`) / Core Rule 4 (`:30`): the two "1 / 会话" cues (`sfx.flow.quit`, `sfx.career.complete`) have no dedupe mechanism. Step 4 and rule 4 define dedupe only for per-`run_id` cues, and the cooldown table routes "每局一次 / 结果 cue" to `run_id`. Say what rejects a second `career_completed` or `flow.quit` in the same session.
- [RECOMMENDED] Scope signal: `systems-index.md:213` records `S`. Four hard upstream contracts (records added), two downstream contracts and six formula blocks put this at `M`. `systems-index.md:73` also still lists only three `Depends On` entries — 纪录与里程碑 is now a hard upstream in this GDD's Dependencies table. Both index edits belong to the orchestration session.

Advisory (not counted): `MAX_TICKS`' justification at `:245` still reads "1500 / 150 = 10", which mixes the cap with the derived interval — the real derivation is `min(floor(d / TICK_MS), 10)`. The signal `broke_even` still shares an identifier with the registered formula `broke_even`; no value conflict, but cue and event names need their own namespace. Player Fantasy and Tuning Knobs remain absent as headings (advisory at `standard`; `## Game Feel` `:437-446` and the Formulas tables cover them). `run-flow.md:377` still carries a stale "音效反馈 | 未写" row in its older summary table.

---

## Design Review: 音效反馈（Audio Feedback）

Specialists consulted: none (lean mode)
Re-review: Yes — prior verdict was NEEDS REVISION on 2026-11-04 (third pass)

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Design (accepted as
Detailed Rules), Edge Cases, Dependencies, Acceptance Criteria — plus Formulas,
because this system defines numeric rules (volume mapping, admission, cooldown,
tick count, pitch, ducking). All six are present.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:437-446`) carries it.
- **Tuning Knobs** — absent as a heading; constants carry ranges and sources
  inside Formulas.

### Prior Blocking Disposition

| # | Prior blocking | Status |
|---|---|---|
| 1 | Voice pool arithmetic / admission order | **Closed** — Core Rule 8 (`:34`) now reads `Lifecycle 3`; sum is 10 and equals `MAX_VOICES` (`:195`); admission order keeps Critical-replaces-Critical before generic steal |
| 2 | `action_committed.action` vocabulary | **Closed** — Edge Cases `:297` accepts `{eat, skip_wave, end_pickup}`, matching the catalogue (`:58`) and `run-flow.md:33`; no `end_meal` remains in the document |
| 3 | Record-cue playback policy | **Closed on substance** — catalogue `:72-75` and the merge paragraph `:79` now state coexistence, `career_completed` closing the frame, and same-frame milestone collapsing, matching `records-and-milestones.md:347` and its AC 20. The wording introduces one new defect: `suppressed:outcome_priority` is not in the closed vocabulary (`:144`) |

### Dependency Graph

Every declared dependency has a GDD on disk.

Upstream:
- ✓ run-flow.md — exists (Approved)
- ✓ settlement-and-entry-fee.md — exists (Approved); timing clause aligned
- ✓ scene-flow.md — exists (In Review)
- ✓ records-and-milestones.md — exists (Approved); record-cue contract now aligned
- ✗ sound bible (`design/audio/`) — does not exist; Open Question 2 covers it

Downstream:
- ✓ in-game-hud.md — exists (Approved); still lacks the `dish_action_rejected` producer
- ✓ ui-flow.md — exists (Needs Revision); all three contracts present

Cited but not dependencies (correctly declared as such): input.md, save-system.md,
data-config.md, game-concept.md — all exist.

### Registry Values

`design/registry/entities.yaml` still carries no audio, cue, voice or SFX entry
(searched for `audio`, `cue`, `sfx`, `voice` — no matches), so this system owns
no registered values.

No numeric conflict:
- `net_profit` is used only as an example range (`start = −85`, `end = 19`,
  `:262`, `:521`); the GDD never restates its formula or value.
- `run_id` is typed `int64` (`:133`), matching the registry.
- `broke_even` is consumed as an event and never derived — the correct boundary.

Gaps: no registry entries exist for this system (Open Question 7); the signal
`broke_even` shares an identifier with the registered formula (advisory, no value
conflict).

### Required Before Implementation

1. **Put `outcome_priority` in the closed reason table** — `:79` records
   `suppressed:outcome_priority`, `:144` lists only `duplicate_outcome` and
   `preempted` under `suppressed`, and the same table says a new reason must be
   added there first. AC 26 then fails on the first career-completing settlement.
   One edit at `:144`.

### Recommended Revisions

2. **Get the HUD to emit `dish_action_rejected`** — declared at
   `in-game-hud.md:176`, produced by no rule. Route to the HUD re-review.
3. **Drop the focus-audio claim from the HUD** — `in-game-hud.md:395,416`
   contradict rule 19 here and `ui-flow.md:473`.
4. **Make AC 10 testable** — assert ordering, not non-overlap, or bound the print
   cue below 0.3 s.
5. **Define the "last finish" mechanism** — track the open-roll set; add the
   "started but never finished" case to Edge Cases.
6. **`0 < d < TICK_MS`** — remove the overlap at zero (`:317-318`).
7. **Ducking scope** — align rule 18 (`:44`) with `duck_db`'s `env(t)` (`:277`).
8. **AC 22** — deferred, or an injected mock bus.
9. **AC 17** — point at AC 25's numbers.
10. **Add an AC for "减少重复音效"**.
11. **Refresh labels and delete the obsolete block** — `:114-115`, `:478-479`,
    and the "双向一致核对" block at `:384-390`.
12. **Add 纪录与里程碑 to Interactions and Cross-References** — it is a hard
    upstream in Dependencies (`:358`).
13. **Name `receipt_closed(run_id, reason)`** (`:307`, `:115`).
14. **Rename `sfx.milestone`** to match `sfx.<域>.<事件>`.
15. **Restate `sfx.record.broken`'s timing** as "事件同帧" or confirm the emitter.
16. **Add an AC for the record-cue merge rule** (`:79`).
17. **Define dedupe for the "1 / 会话" cues** (`sfx.flow.quit`,
    `sfx.career.complete`) — step 4 (`:184`) only covers per-`run_id` cues.

### Specialist Disagreements

None — Lean mode runs no specialists (Phase 3b skipped).

### Nice-to-Have

- `MAX_TICKS`' "1500 / 150 = 10" (`:245`) describes the derived interval, not the
  derivation; write `min(floor(d / TICK_MS), 10)`.
- Cue and event names still need a namespace distinct from formula names
  (`broke_even`).
- `run-flow.md:377` still has a stale "音效反馈 | 未写" row.
- `systems-index.md:73` should add 纪录与里程碑 to `Depends On` and move the row
  to `Needs Revision`.

### Scope Signal

Rough scope signal: **M** — four hard upstream contracts, two downstream
contracts, six formula blocks, no new ADR expected at design level (the pause
`process_mode` ADR is already anticipated in Open Question 6). The index records
`S` and lists three dependencies.

### Verdict: NEEDS REVISION

One blocking item, and it is a one-line vocabulary entry. The design itself is
sound: the three prior blockers are closed, the record-cue contract now matches
its producer in both directions, and the remaining items are carried-over polish
split between this file and the HUD.

---

Reviewed-Content-Hash: design/gdd/audio-feedback.md ffb1c54a84af1da89b0228395ee86e89b3e87e30
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/ui-flow.md 9c709252b4c631e8fb50d8a2f35a8fb394f230d7
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 84e5c38fc40fb2a75d8ee608c911bdebf369c5a1
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/systems-index.md 12d2e7c5b8de635a090b0d1f4589dcab58c729f8
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

---

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none (lean — Phase 3b not run)
Blocking items: 0 | Recommended: 17
Summary: The prior round's only blocking defect is closed. `outcome_priority` is now in the closed `suppressed` vocabulary (`:144`), matching the record-cue merge rule (`:79`), so AC 26 is exhaustively satisfiable and the first career-completing settlement no longer fails. The change is a one-line addition and introduces no new inconsistency: every `decision:reason` token used anywhere in the document now resolves into that vocabulary — 11 `dropped` reasons, 3 `suppressed` reasons, 1 `deferred` reason. The three earlier blocking items (voice-pool arithmetic, `action_committed.action` vocabulary, record-cue playback policy) remain closed. The design is sound and implementable; the 17 recommended items are carried over from the previous pass, most of them living in the HUD or the systems index rather than in this file.
Prior verdict resolved: Yes — the single prior blocking item is closed, and no new blocking defect was introduced.

Findings:
- [RECOMMENDED] Dependencies (`:365`) / `in-game-hud.md:176`: no HUD rule emits `dish_action_rejected` — rule 11 (`in-game-hud.md:54`) declines an inedible focus and `:373` covers the rejection path, and neither names the event. The HUD is unchanged this round, so `sfx.eat.reject` remains a dead cue until the HUD re-review adds the producer.
- [RECOMMENDED] Dependencies (`:365`) / `in-game-hud.md:395,416`: the HUD still claims audio subscribes to "焦点移动、确认、拒绝、跳过意图". Rule 19 (`:45`) refuses any focus cue and `ui-flow.md:473` states the same. Remove the focus-movement wording in the HUD.
- [RECOMMENDED] Acceptance Criteria 10 (`:505`): "两条 cue 的播放区间不重叠" is not verifiable from this GDD — the print cue's duration lives in the sound spec and is unstated, and `ui-flow.md:69` permits a 0.3 s roll. Assert only "result cue starts at or after the last `number_roll_finished`", or bound the print cue below 0.3 s in the sound spec.
- [RECOMMENDED] Detailed Design, Core Rule 12 (`:38`): "结果 cue 只在**最后一个** `number_roll_finished` 后起播" still has no mechanism — `ui-flow.md:347` shares one duration `D` across all fields, so every field finishes in the same frame. State it as "track the open-roll set; fire when it empties". The start-without-finish fallback exists only inside rule 12 and still deserves an Edge Cases entry.
- [RECOMMENDED] Edge Cases (`:317-318`): `d = 0` ("no tick, no `land`") and `d < 60 ms` ("0 ticks, `land` at end") still overlap at exactly zero. Write the second as `0 < d < TICK_MS`.
- [RECOMMENDED] Formulas `duck_db` (`:277`) vs Core Rule 18 (`:44`): rule 18 says ducking happens only while a Critical cue plays; `env(t)` includes `settlement_started`, a Lifecycle cue.
- [RECOMMENDED] Acceptance Criteria 22 (`:523`): the ducking AC cannot pass in the Vertical Slice — `:287` and `:414` state there is no music or ambience. Mark it deferred or test against an injected mock bus.
- [RECOMMENDED] Acceptance Criteria 17 (`:515`): "派发时间在预算内" is not measurable from the log; point it at AC 25's `≤ 0.25 ms` median / `≤ 1.0 ms` peak.
- [RECOMMENDED] Acceptance Criteria: still no criterion for "减少重复音效" (`:457`), which changes `cooldown_ms`, `TICK_MS` and variant selection.
- [RECOMMENDED] Dependencies (`:384-390`) / Interactions (`:114-115`) / Cross-References (`:478-479`): the "双向一致核对" block is wholly obsolete — all four named documents already carry the audio row, and #15 / #16 are written GDDs — while the Interactions table still labels them "（#15，未设计）" / "（#16，未设计）" and Cross-References still says "（未写）". Delete the block and refresh the four labels.
- [RECOMMENDED] Interactions (`:109-119`) / Cross-References (`:472-482`): 纪录与里程碑 is absent from both tables even though Dependencies (`:358`) lists it as a hard upstream and `:382` cites the three cue mappings. Add the row to both.
- [RECOMMENDED] Edge Cases (`:307`) / Interactions (`:115`): `ui-flow.md:168` emits `receipt_closed(run_id, reason: restart | menu | quit)` and `:91` cites this GDD's edge case by that name. Name the event here instead of "收据关闭".
- [RECOMMENDED] 音效事件目录 (`:73`): `sfx.milestone` is the only cue id that does not follow `sfx.<域>.<事件>`. `sfx.record.milestone` or `sfx.milestone.earned` matches the rest.
- [RECOMMENDED] 音效事件目录 (`:72`): `sfx.record.broken`'s 起播时机 "收据打开时" sits outside Core Rule 5's same-frame dispatch unless the upstream emits at receipt open. Say "事件同帧", or confirm the emitter with the records GDD.
- [RECOMMENDED] Acceptance Criteria: the merge/exclusion policy at `:79` has no criterion. Add one settlement fixture — `record_broken` + two `milestone_earned` + `career_completed` in one frame — asserting `sfx.career.complete` `admitted`, the other three `suppressed:outcome_priority`, and a second fixture with two milestones and no career completion asserting exactly one `sfx.milestone`.
- [RECOMMENDED] Formulas (`:230`) / `voice_admission` step 4 (`:184`) / Core Rule 4 (`:30`): the two "1 / 会话" cues (`sfx.flow.quit`, `sfx.career.complete`) have no dedupe mechanism. Step 4 and rule 4 define dedupe only for per-`run_id` cues, and the cooldown table routes "每局一次 / 结果 cue" to `run_id`.
- [RECOMMENDED] Scope signal: `systems-index.md:213` records `S`. Four hard upstream contracts (records added), two downstream contracts and six formula blocks put this at `M`. `systems-index.md:73` also still lists only three `Depends On` entries — 纪录与里程碑 is now a hard upstream in this GDD's Dependencies table. Both index edits belong to the orchestration session.

Advisory (not counted): `MAX_TICKS`' justification at `:245` still reads "1500 / 150 = 10", which mixes the cap with the derived interval — the real derivation is `min(floor(d / TICK_MS), 10)`. The signal `broke_even` still shares an identifier with the registered formula `broke_even`; no value conflict, but cue and event names need their own namespace. Player Fantasy and Tuning Knobs remain absent as headings (advisory at `standard`; `## Game Feel` `:437-446` and the Formulas tables cover them). `run-flow.md:377` still carries a stale "音效反馈 | 未写" row in its older summary table.

---

## Design Review: 音效反馈（Audio Feedback）

Specialists consulted: none (lean mode)
Re-review: Yes — prior verdict was NEEDS REVISION on 2026-11-04 (third pass)

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Design (accepted as
Detailed Rules), Edge Cases, Dependencies, Acceptance Criteria — plus Formulas,
because this system defines numeric rules (volume mapping, admission, cooldown,
tick count, pitch, ducking). All six are present.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:437-446`) carries it.
- **Tuning Knobs** — absent as a heading; constants carry ranges and sources
  inside Formulas.

### Prior Blocking Disposition

| # | Prior blocking | Status |
|---|---|---|
| 1 | Closed reason vocabulary (`suppressed:outcome_priority`) | **Closed** — `:144` now lists `outcome_priority` under `suppressed`; `:79` records it, and every `decision:reason` token in the document (11 `dropped`, 3 `suppressed`, 1 `deferred`) resolves into the table, so AC 26 is exhaustively satisfiable |
| — | Voice pool arithmetic / admission order (r2) | Still closed — `:34` = `Critical 1 / Lifecycle 3 / Gameplay 3 / UI 3` (sum 10 = `MAX_VOICES` `:195`) |
| — | `action_committed.action` vocabulary (r2) | Still closed — `:297` accepts `{eat, skip_wave, end_pickup}`; no `end_meal` remains |
| — | Record-cue playback policy (r2) | Still closed on substance — `:79` merge rule matches `records-and-milestones.md:347` and its AC 20 |

### Dependency Graph

Every declared dependency has a GDD on disk.

Upstream:
- ✓ run-flow.md — exists (Approved)
- ✓ settlement-and-entry-fee.md — exists (Approved)
- ✓ scene-flow.md — exists (In Review)
- ✓ records-and-milestones.md — exists (Approved); record-cue contract aligned
- ✗ sound bible (`design/audio/`) — does not exist; Open Question 2 covers it

Downstream:
- ✓ in-game-hud.md — exists (Approved); still lacks the `dish_action_rejected` producer
- ✓ ui-flow.md — exists (Needs Revision); all three contracts present, including `receipt_closed(run_id, reason)`

Cited but not dependencies (correctly declared as such): input.md, save-system.md,
data-config.md, game-concept.md — all exist.

### Registry Values

`design/registry/entities.yaml` still carries no audio, cue, voice or SFX entry
(searched for `audio`, `cue`, `sfx`, `voice` — no matches), so this system owns
no registered values.

No numeric conflict:
- `net_profit` is used only as an example range (`start = −85`, `end = 19`,
  `:262`, `:521`); the GDD never restates its formula or value.
- `run_id` is typed `int64` (`:133`), matching the registry.
- `broke_even` is consumed as an event and never derived — the correct boundary.

Gaps: no registry entries exist for this system (Open Question 7); the signal
`broke_even` shares an identifier with the registered formula (advisory, no value
conflict).

### Required Before Implementation

- none — the closed reason vocabulary is complete; no blocking defect remains.

### Recommended Revisions

1. **Get the HUD to emit `dish_action_rejected`** — declared at
   `in-game-hud.md:176`, produced by no rule. Route to the HUD re-review.
2. **Drop the focus-audio claim from the HUD** — `in-game-hud.md:395,416`
   contradict rule 19 here and `ui-flow.md:473`.
3. **Make AC 10 testable** — assert ordering, not non-overlap, or bound the print
   cue below 0.3 s.
4. **Define the "last finish" mechanism** — track the open-roll set; add the
   "started but never finished" case to Edge Cases.
5. **`0 < d < TICK_MS`** — remove the overlap at zero (`:317-318`).
6. **Ducking scope** — align rule 18 (`:44`) with `duck_db`'s `env(t)` (`:277`).
7. **AC 22** — deferred, or an injected mock bus.
8. **AC 17** — point at AC 25's numbers.
9. **Add an AC for "减少重复音效"**.
10. **Refresh labels and delete the obsolete block** — `:114-115`, `:478-479`,
    and the "双向一致核对" block at `:384-390`.
11. **Add 纪录与里程碑 to Interactions and Cross-References** — it is a hard
    upstream in Dependencies (`:358`).
12. **Name `receipt_closed(run_id, reason)`** (`:307`, `:115`).
13. **Rename `sfx.milestone`** to match `sfx.<域>.<事件>`.
14. **Restate `sfx.record.broken`'s timing** as "事件同帧" or confirm the emitter.
15. **Add an AC for the record-cue merge rule** (`:79`).
16. **Define dedupe for the "1 / 会话" cues** (`sfx.flow.quit`,
    `sfx.career.complete`) — step 4 (`:184`) only covers per-`run_id` cues.
17. **Two index edits belong to the orchestration session** — `systems-index.md:73`
    add 纪录与里程碑 to `Depends On`; `:213` scope `S` → `M`.

### Specialist Disagreements

None — Lean mode runs no specialists (Phase 3b skipped).

### Nice-to-Have

- `MAX_TICKS`' "1500 / 150 = 10" (`:245`) describes the derived interval, not the
  derivation; write `min(floor(d / TICK_MS), 10)`.
- Cue and event names still need a namespace distinct from formula names
  (`broke_even`).
- `run-flow.md:377` still has a stale "音效反馈 | 未写" row.

### Scope Signal

Rough scope signal: **M** — four hard upstream contracts, two downstream
contracts, six formula blocks, no new ADR expected at design level (the pause
`process_mode` ADR is already anticipated in Open Question 6). The index records
`S` and lists three dependencies.

### Verdict: APPROVED

No blocking items. The one-line vocabulary fix closes the last defect and
introduces none; every logged reason now falls inside the closed vocabulary, so
AC 26 is exhaustively testable. The 17 recommended items are polish carried over
from the previous pass, most of them in the HUD or the index; none prevents
implementation.

---

Reviewed-Content-Hash: design/gdd/audio-feedback.md 33484ec0141607caa44f2fc3c82774245db5fb82
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/ui-flow.md 46d6938f567d04423303bb4bd789234b6b1ddf6c
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 84e5c38fc40fb2a75d8ee608c911bdebf369c5a1
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 75098fd56028b3ec1b1abb5cb53da81e0eccaf6d
