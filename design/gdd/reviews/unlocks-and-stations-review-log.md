# 解锁与档口（Unlocks and Stations）— Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L（Vertical Slice 跨系统事务）；MVP 切片单独为 S–M
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 8
Summary: The rule layer is unusually solid: availability is derived rather than queried, the unlock set is monotonic, progress counts current playability rather than the ledger, `INVALID_STATION_DATA` is ordered ahead of `INSUFFICIENT_FUNDS`, and all 41 ACs are independently testable. One sequencing gap blocks: the GDD requires the free-station backfill to ride the save's first baseline, but its own state machine only initialises after the save load completes, while `save-system` defines the Fresh baseline as an immediate write of an **empty** unlock set. That makes the claimed root fix for the "free station raised to paid" window unachievable as specified, and leaves AC 2's persisted value ambiguous between Fresh and Loaded. The remaining findings are cross-document bookkeeping: the 9-item back-write list is 7/9 already applied on main, four downstream GDDs never mention this system, and `spend`'s shared-`BUSY` half lives only in this document while `meta-upgrades` is under revision.
Prior verdict resolved: First review

Findings:
- [BLOCKING] Detailed Design rule 14 (`:53`) / States (`:76`) / AC 2 (`:451`) vs `save-system.md:31` rule 6, `:232` Fresh edge case, `:356` AC 1: the backfill is said to be written "随存档的首次基线一起写出", but this system reaches `Ready` only after "存档载入完成", and save writes the Fresh baseline immediately with `空解锁集`. No ordering, hook or interface exists for the two to meet.
- [RECOMMENDED] `spend` contract (`:120-131`) vs `meta-upgrades.md:87-89`, `:121`: the shared in-flight `BUSY` and the rollback behaviour are defined only here; `meta-upgrades` still describes a purchase-only `Purchasing` state. Depends on #12's in-progress revision — track, do not treat as settled.
- [RECOMMENDED] Dependencies「待回改」(`:376-386`) / status rows (`:358-360`): items 1–5, 7 and 8 have landed on main (systems-index row, six registry formulas, data-config row + rule 7 note, `spend` + OQ 11, save row + edge case + AC 13, records OQ 4). Only item 6 (run-flow defence-in-depth) and item 9 (UX note) remain open; "`spend` 待回写" is stale.
- [RECOMMENDED] Dependencies (`:352-374`): `run-flow`, `menu-waves`, `scene-flow`, `first-run-onboarding` contain no mention of this system at all, yet four rows are marked `GDD 已写` and the 双向一致要求 paragraph claims they are done. `records-and-milestones.md:303` goes further and says the two systems "互不读写", contradicting the display dependency asserted here.
- [RECOMMENDED] Interfaces (`:108`) / UI Requirements (`:420`): `StationView` carries id, name, availability, price, lock reason and dish count — no `entry_fee` — while the catalog row must display 入场费, and the data-config contract (`:89`, `:358`) omits `entry_fee` too.
- [RECOMMENDED] Rule 8 (`:41`) / rule 9 (`:43`): `NOT_IN_MENU` also covers "收据未完成 / 场景切换中", but no interface supplies the receipt-open or transition flag; "receipt open" is owned by `ui-flow`/`scene-flow` and is not in the interaction table.
- [RECOMMENDED] Formulas `runs_to_unlock` (`:250-273`): the 4–8 run band assumes every yuan of `opt_net_profit` goes to the unlock, ignoring the upgrades bought from the same wallet (the 交互 note at `:398` acknowledges the sharing but the band does not model it). `opt_net_profit` is also an unregistered derived formula, and `opt_value`'s real signature is `(M, C, T)`, not a station.
- [RECOMMENDED] Detailed Design rule 15 (`:55`): cites "存档规则 80"; `save-system.md` has 25 core rules, not 80. `ui-flow.md:59` carries the same dangling reference. Point at save rule 18 + AC 13, or at the records orchestrator contract (`records-and-milestones.md:120`).
- [RECOMMENDED] Header (`:3`) / Dependencies (`:360`): `Status: In Design` while `systems-index.md:69` says `Designed`; `spend` is called "新增接口" although `meta-upgrades.md:121` already lists it.

---

## Design Review: 解锁与档口（Unlocks and Stations）

Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules (`## Detailed Design`
is the accepted heading), Formulas (the system defines numeric rules — unlock
cost, availability predicate, normalisation, progress counters), Edge Cases,
Dependencies, Acceptance Criteria. All six are present.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:410-416`) carries
  the intended feeling（把门推开、目标常驻、不制造 FOMO）.
- **Tuning Knobs** — present (`:388-398`), with safe ranges and the
  4–8 × 19 derivation.

### Dependency Graph

Every declared dependency has a GDD on disk.

- ✓ data-config.md — exists (`:358`); reciprocal row at `:187`, rule 7 note at
  `:55`
- ✓ save-system.md — exists (`:359`); reciprocal row at `:89`, `:271`
- ✓ meta-upgrades.md — exists (`:360`); reciprocal row at `:104`, `:350`
- ✓ ui-flow.md — exists (`:366`); row at `:439`, but a second stale row at
  `:162` still says "#13，未设计"
- ✓ run-flow.md — exists (`:367`) — **no mention of this system**
- ✓ menu-waves.md — exists (`:368`) — **no mention of this system**
- ✓ records-and-milestones.md — exists (`:369`) — mentions it, but as "互不读写"
  (`:303`), contradicting the display dependency here
- ✓ first-run-onboarding.md — exists (`:370`) — **no mention of this system**
- ✓ audio-feedback.md — exists (`:371`); `:376` agrees there is no direct call
- ✓ scene-flow.md — exists (`:372`) — **no mention of this system**

Bidirectional check: three upstream edges are reciprocated; four downstream
edges are one-sided and one is actively contradicted. The 双向一致要求
paragraph (`:374`) asserts the opposite.

### Registry Values

Searched the whole document for the registry's station and pacing terms —
`station_canteen`, `station_banquet`, `unlock_cost`, `dish_pool`, `entry_fee`,
`opt_value`, `net_profit`, `upgrade_tree_cost`, `save_bytes_max`,
`save_schema_version`, plus the literals `209 / 104 / 85 / 120 / 262144`.
No value conflicts. Independent checks:

- **Six owned formulas** — `station_available` (`entities.yaml:1003`),
  `effective_unlocks` (`:1019`), `unlock_decision` (`:1034`),
  `station_unlock_cost` (`:1051`), `runs_to_unlock` (`:1065`),
  `unlock_progress` (`:1077`) all reproduce the GDD's expressions verbatim and
  carry `source: design/gdd/unlocks-and-stations.md`.
- **Consumed values** — `station_canteen_entry_fee = 85` (`:1183`),
  `opt_value` notes 104 at C=20/T=10 (`:443-458`), `upgrade_tree_cost` notes
  209 (`:975-984`), `save_bytes_max = 262144` (`:1206-1213`). The GDD's
  `104 − 85 = 19`, `ceil(120 / 19) = 7`, `76–152 = 4–8 × 19` and "远小于
  256 KiB" all match.
- **`station_banquet` is not registered**, which is correct — the document marks
  it 暂定 id and not released data (`:151`).
- **`opt_net_profit` has no registry entry** although `runs_to_unlock` lists it
  as a variable. Registry edits belong to the orchestration session.

### Required Before Implementation

1. **The Fresh-career baseline has two conflicting definitions.**
   Rule 14 (`:53`) says the normalised unlock set is written "随存档的首次
   基线一起写出"; AC 2 (`:451`) then asserts that after load and before any
   write point "持久化集合仍是 `[]`". But `save-system.md:232` defines the
   `Fresh` path as "用默认值建立生涯（…空解锁集…），立即写入一次基线", and
   AC 1 (`:356`) fixes that write as happening "进入 `Menu` 后立即". This
   system's own state table (`:76`) reaches `Ready` only after "数据 `Ready` 且
   存档载入完成" — i.e. after the baseline already exists. There is no hook,
   ordering statement or interface by which this system contributes `unlocks`
   to that first write, so:
   - the "涨价窗口" that rule 14 (`:53`) and the edge case (`:333`) call
     "已根治" is still open — the free station's id is not on disk until the
     next write point;
   - AC 2 is not executable as written: for a `Fresh` career `persisted` is
     `[]` only until the baseline write, which save performs immediately; for a
     `Loaded` career it is genuinely `[]`. The GIVEN never says which.

   **Fix:** pick and state one sequencing. Cheapest options: (a) `save-system`
   defers the `Fresh` baseline until this system reports its normalised set
   (save already owns the write point); (b) this system exposes
   `prepare_baseline()` that save calls before its first commit; or (c) this
   system issues an immediate `commit` after backfill on `Fresh` (a fifth
   write point in save rule 6). Then split AC 2 into a `Fresh` case and a
   `Loaded` case with the expected `persisted` value for each. Whichever is
   chosen is a `save-system` change, so record it in 待回改 as well.

### Recommended Revisions

1. **`spend`'s shared lock is defined only here.** The contract table (`:131`)
   says `spend` shares one in-flight constraint (`BUSY`) with upgrade
   purchases, and (`:129`) that `extra` is merged into one `commit`. But
   `meta-upgrades.md:87-89` still describes `Purchasing` as the only in-flight
   state and `:121` lists `spend` without a validation order, `extra` merge or
   rollback. `meta-upgrades.md` is currently `Needs Revision` and its own
   review already logged this as recommended (待回改 item 4). Track it against
   #12's revision rather than treating the edge as settled. Also name the
   mapping from `SpendResult` to `UnlockResult` for the one case this document
   cannot pre-check — a purchase already in flight returning `BUSY`.
2. **The 待回改 list is stale.** Items 1–5, 7 and 8 of `:376-386` have landed:
   `systems-index.md:69` is `Designed` with 数据配置 in `Depends On`; the six
   formulas exist in the registry; `data-config.md:187` and rule 7 (`:55`) are
   present; `meta-upgrades.md:29`, `:121` and OQ 11 (`:501`) are present;
   `save-system.md:271`, the same-frame edge case (`:256`) and AC 13 (`:378`)
   are corrected; `records-and-milestones.md:445` answers OQ 4. Only item 6
   (`run-flow` defence-in-depth) and item 9 (UX note) are genuinely open, and
   the "`spend` 待回写" status at `:360` is stale.
3. **Four downstream edges are one-sided and one is contradicted.**
   `run-flow.md`, `menu-waves.md`, `scene-flow.md` and
   `first-run-onboarding.md` contain no occurrence of 解锁 / 档口 / unlock, so
   the `GDD 已写` rows at `:367-372` and the 双向一致要求 paragraph (`:374`)
   overstate the state. Separately, `records-and-milestones.md:303` says
   局外升级 / 解锁与档口 "与本系统同属结算的下游，但互不读写", while the row
   at `:369` claims records reads unlock progress for display. Either add the
   reciprocal rows (or downgrade the claim), or drop the records display edge.
   For `run-flow`, item 6's optional `is_station_available(station_id)`
   guard is the concrete content that row is missing.
4. **`StationView` cannot supply 入场费.** UI Requirements (`:420`) require the
   catalog row to show 入场费, but the interface (`:108`) lists id, name,
   availability, price, lock reason and dish count only, and the data-config
   contract (`:89`, `:358`) omits `entry_fee`. `ui-flow.md:622` (OQ 5) already
   flags that 界面流程 currently reads `entry_fee` straight from 数据配置.
   Decide whether `StationView` gains `entry_fee` or the row is rendered by
   `ui-flow` from its own data-config read.
5. **"Menu 就绪" has no interface.** Rules 8–9 (`:41`, `:43`) return
   `NOT_IN_MENU` while "收据未完成、场景切换中", but nothing in the interaction
   table tells this system the receipt state — that belongs to `ui-flow` (rule
   10, `:59`) and the top-level state to `scene-flow`. Name the predicate
   (`ui-flow.is_menu_ready()` or equivalent), otherwise the guard is
   unimplementable as written.
6. **The pacing band ignores upgrade spending.** `runs_to_unlock` (`:250-273`)
   divides the unlock price by the previous station's optimal net profit as if
   every yuan went to the unlock. Upgrades are bought from the same wallet
   (`:398` acknowledges the sharing), so the real first-session unlock lands
   later than 7 runs whenever the player buys anything. State the assumption
   explicitly (e.g. "band assumes zero upgrade spend") or model the split in
   the design-time check. Also note that `opt_value` is defined over
   `(M, C, T)` (`entities.yaml:443-458`), not over a station, and that
   `opt_net_profit` is an unregistered derived formula.
7. **Dangling rule citation.** Rule 15 (`:55`) cites "存档规则 80";
   `save-system.md` has 25 core rules. The intended content is save rule 18
   (settlement changes retried, not rolled back) plus AC 13 (`:378`) and the
   records orchestrator contract (`records-and-milestones.md:120`). Fix the
   citation here; `ui-flow.md:59` carries the same stale number.
8. **Status header and interface wording.** `:3` says `In Design` while
   `systems-index.md:69` says `Designed`; `:91` calls `spend` a "新增接口"
   although `meta-upgrades.md:121` already lists it. Suggested status for this
   review: **`Needs Revision`** until blocking item 1 is resolved, then
   `In Review` for a re-review.

### Nice-to-Have

- **Locked-station dish count.** Visual/Audio (`:402`) says locked stations
  must not reveal 菜品卡、价格或菜名 because "好奇心来自不知道里面有什么",
  while the catalog row (`:420`) and the confirmation dialog (`:421`) show
  菜数. Showing a count is consistent with the rules but sits oddly next to
  the curiosity rationale — worth one sentence either way.
- **`get_load_outcome()` "载入后一次确定"** (`:107`) conflicts with the
  `Unavailable → Ready` transition (`:79`): if data becomes `Ready` later, the
  outcome must be recomputed. Say whether the outcome is latched per load or
  derived on demand.
- **`can_unlock` and `BUSY`.** The interface (`:112`) does not say whether
  `can_unlock` reports `BUSY` during an in-flight transaction. State it, since
  the UI enables the button from this call.
- **AC 38** (`:505`) tests "先结算、后解锁" in the same frame, a sequence rule 8
  (`:41`) forbids (receipt open). Keep it as a save-serialisation test, but
  label it as a defensive case rather than a reachable UI path.
- **Sort collation.** "升序" is asserted for every set output (`:57`, `:200`)
  without naming the comparison (byte/Unicode). Determinism ACs depend on it.
- **`runs_to_unlock`'s "前一个档口"** (`:267`) is unambiguous for the second
  station and ambiguous for the third; define it as "the highest-fee station
  currently available" or similar.

### Verdict: NEEDS REVISION

The design itself is implementable and internally coherent on almost every
axis; the single blocking item is a cross-document sequencing contract that
the GDD relies on for a stated invariant. Fix that (and clarify AC 2), and the
rest is bookkeeping that belongs mostly to the orchestration session.

Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md bcde83821bc73ad7c67a24d38a8659ce2d53693b
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

---

## Review — 2026-11-04 — Verdict: APPROVED

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b not run)
Blocking items: 0 | Recommended: 9
Prior verdict resolved: Yes — the single blocking item is resolved

Summary: The Fresh-baseline sequencing now has a real hook on both sides: rule 14 requires save to call this system's `normalize()` before the first baseline write, `save-system.md:232` states the same order, AC 1 asserts the Fresh baseline carries the backfilled set, and AC 2 is now explicitly scoped to a `Loaded` career so its `persisted = []` expectation is unambiguous. With no blocking item left, the design's rule layer (derived availability, monotonic ledger, ordered validation, `INVALID_STATION_DATA` ahead of `INSUFFICIENT_FUNDS`, 41 testable ACs) stands as approved. The remaining findings are cross-document bookkeeping: `normalize()` is still missing from this GDD's own interface table, the four downstream edges remain one-sided, `records` still says "互不读写", and three stale citations/status markers persist.

### Prior Blocking Items — Status

1. **RESOLVED** — Fresh-career baseline sequencing. Rule 14 (`:53`) now requires save to call `normalize(∅)` before writing the first baseline and to merge the backfill into that same transaction; `save-system.md:232` independently states "写基线之前先调用解锁与档口的 `normalize()`…不写空集合基线"; AC 1 (`:450`) adds the `Fresh` case asserting the first baseline carries the backfilled `stations` / `dishes`; AC 2 (`:451`) is rewritten as a `Loaded` career so "持久化集合仍是 `[]`" is executable. The narrow residual window for old careers whose empty set was never written is acknowledged in rule 14 and the edge case (`:333`).

### Findings

- [RECOMMENDED] Interfaces (`:105-116`) / rule 14 (`:53`) vs `save-system.md:232`: `normalize()` is now load-bearing but absent from the "对外接口（设计级）" table, and its call form differs between the two documents (`normalize(∅)` here, `normalize()` there). Save's own AC 1 (`:360`) still asserts only "内存中解锁集为空" and never that the baseline carries the backfill. Add the interface row (input, return, whether it mutates the in-memory set, dirty semantics), align the call form, and ask the save session to extend its AC 1.
- [RECOMMENDED] `spend` contract (`:120-131`) vs `meta-upgrades.md:84-89`, `:121`: `spend` now appears in #12's interface table, rule 2 and OQ 11, but #12 still does not state its validation order, `extra` merge, rollback or the shared in-flight `BUSY`, and its state table still names the in-flight state `Purchasing` with "只允许一笔" scoped to purchases. This GDD's dependency row (`:360`) still says "`spend` 待回写" — now stale.
- [RECOMMENDED] 待回改 (`:376-386`) / status rows (`:358-372`): items 1, 2, 3, 5, 7 and 9 have landed — `systems-index.md:69` is `In Review` with 数据配置 in `Depends On`; the six formulas exist at `entities.yaml:1003-1087`; `data-config.md:187` and rule 7 (`:55`) exist; `save-system.md:232`, `:254`, `:271`, AC 13 (`:378`) and AC 20 (`:388`) are updated; `records-and-milestones.md:446` answers OQ 4; the UX note is at `systems-index.md:113`. Item 4 is partial (above); only item 6 (the optional `run-flow` guard) is genuinely open.
- [RECOMMENDED] Dependencies (`:367-372`) / 双向一致要求 (`:374`): `run-flow`, `menu-waves`, `scene-flow` and `first-run-onboarding` contain zero occurrences of 解锁与档口 yet are marked `GDD 已写`. `records-and-milestones.md:304` still says 局外升级 / 解锁与档口 "互不读写", contradicting the display dependency asserted at `:95` and `:369`. `ui-flow.md:164` still says "#13，未设计" while `:443` says `GDD 已写`.
- [RECOMMENDED] Interfaces (`:108`) / UI Requirements (`:420`): `StationView` still has no `entry_fee`, the data-config contract (`:89`, `:358`) omits it, and the catalog row must show 入场费. `ui-flow.md:161` currently reads `entry_fee` directly from 数据配置 — decide whether the row is rendered from `StationView` or from `ui-flow`'s own read.
- [RECOMMENDED] Rule 8 (`:41`) / rule 9 (`:43`): `NOT_IN_MENU` still covers "收据未完成 / 场景切换中", but no interface supplies either flag. Name the predicate (e.g. `ui-flow.is_menu_ready()`), otherwise the guard cannot be implemented as written.
- [RECOMMENDED] Formulas `runs_to_unlock` (`:250-273`): unchanged; the 4–8 band still divides by the previous station's optimal net profit as if every yuan went to the unlock, while upgrades draw on the same wallet (`:398`). State the zero-upgrade-spend assumption in the band. `opt_net_profit` is still not a registry entry (named as a variable at `entities.yaml:1072`, no owning entry), and `opt_value` remains defined over `(M, C, T)` (`:443-458`), not over a station.
- [RECOMMENDED] Rule 15 (`:55`): still cites "存档规则 80"; `save-system.md` has 25 core rules. Point at save rule 18 + AC 13 (`:378`) and the records orchestrator contract (`records-and-milestones.md:120`).
- [RECOMMENDED] Header (`:3`) / interfaces (`:91`) / dependencies (`:360`): `Status: In Design` while `systems-index.md:69` says `In Review`; `spend` is still called a "新增接口" although `meta-upgrades.md:121` lists it; "`spend` 待回写" is stale.

### Nice-to-Have

The prior review's six remaining nice-to-haves are unchanged: locked-station dish count (`:402` vs `:420`), `get_load_outcome()` "载入后一次确定" vs the `Unavailable → Ready` transition (`:107` / `:79`), whether `can_unlock` reports `BUSY` (`:112`), AC 38's same-frame ordering being a defensive save case rather than a reachable UI path (`:505`), the unspecified sort collation (`:57`, `:200`), and "前一个档口" being ambiguous for the third station (`:267`).

### Completeness

6/6 required sections present at `workflow: standard` (Overview, Detailed Design as Detailed Rules, Formulas, Edge Cases, Dependencies, Acceptance Criteria). Player Fantasy is absent as a heading (`## Game Feel` carries it) — advisory only; Tuning Knobs present.

### Dependency Graph

All eleven declared systems have GDDs on disk. Three upstream edges are reciprocated; four downstream edges are one-sided (finding above).

### Registry Values

No value conflicts. `station_available`, `effective_unlocks`, `unlock_decision`, `station_unlock_cost`, `runs_to_unlock` and `unlock_progress` reproduce the GDD's expressions; `station_canteen_entry_fee = 85`, `opt_value` at 104 (C=20/T=10) and `save_bytes_max = 262144` match. `opt_net_profit` remains unregistered.

### Suggested Status

`Approved` in `design/gdd/systems-index.md` (currently `In Review`). Not written by this review.

Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/save-system.md 772ed596d68eb5cbae207b7bac9a170ec55c1524
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 9fe5183795ca3d47798510a1d00a040dabe3ca61
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/menu-waves.md ce532d3265df1ed7555ba5aa46383be35bbee4a1
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/systems-index.md 15e0bd3d307ed09738b74dd3c099816b497b69e8

---

## Revision — 2026-11-06 — GDD Revision Flags write-back

Closed the `architecture-review-2026-11-05` GDD Revision Flags for this system; this entry records a revision, not a fresh review.

- Flag #8 — `effective_unlocks` now uses `sort({s.id : s ∈ S_eff} ∪ U.stations)` instead of `U.stations \ S_ids`, so a gated but known purchased station keeps its id in the ledger (matching rule 17 and AC 44). Source: ADR-0017 #1.

Status: substantive edit — the GDD returns to `In Review` per `.claude/rules/design-docs.md`; approval requires a fresh full `/design-review`.

Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 729311c1e48fad9d1f935240bba7850da718a1af

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 3
Summary: Flag #8 is written correctly in the GDD: `normalize` now emits `stations = sort({s.id : s ∈ S_eff} ∪ U.stations)`, the prose note explains that a gated-but-owned id stays in the ledger, and the formula agrees with `S_eff`'s `¬gated(s)` filter. The document is otherwise clean and its AC 44 already asserts the gated-purchase case. The single blocker is cross-document: the registry's canonical `normalize` expression still carries the pre-fix `(U.stations − S_ids)` term and omits `¬gated(s)`, so the fact source and the owning GDD now disagree about the exact behaviour the flag was raised to fix. No GDD text change is required — only the registry row.
Prior verdict resolved: No — flag #8 closed in the GDD, but the registry still encodes the old expression
Findings:
- [BLOCKING] `design/registry/entities.yaml:1047` (`unlocks.normalize.expression`): still reads `S_eff = S_free + {s in S : s.id in U.stations}; stations = sort(ids(S_eff) + (U.stations - S_ids))` — the pre-fix form that drops a gated purchased id, and it omits the `¬gated(s)` qualifier on `S_eff`. Required: "`S_eff = S_free + {s in S : s.id in U.stations and not gated(s)}; stations = sort(ids(S_eff) + U.stations); dishes = sort(U.dishes + union(dish_pool of S_eff))`". Source: ADR-0017 §4 and its GDD Sync Check; `design/gdd/unlocks-and-stations.md:195`. Not editable in this review (registries belong to the orchestration session); per Phase 3 a registry line that differs from the owning GDD is required before implementation. Once that row is synced this GDD needs no further edit.
- [RECOMMENDED] Variables (`:208`): with `U.stations − S_ids` gone, `S_ids` has no consumer left in the formula. Either drop the row or keep it and say it exists only to identify unknown ids in the `dishes` line.
- [RECOMMENDED] Alignment note (`:200`): the citation "ADR-0017 第 1 条" refers to the ADR's GDD Sync Check row, not decision §4 (which defines `S_eff` and `stations`). Cite §4, as `records-and-milestones` also needs to do for ADR-0018.
- [RECOMMENDED] Output Range (`:215`): "只增不减：`stations ⊇ U.stations`" is true, but with the new union the guarantee now also holds for ids whose station record was removed from data; state that unknown ids survive transitions unchanged, not merely that they are "原样保留" in the unknown-id sentence.

Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 729311c1e48fad9d1f935240bba7850da718a1af
Reviewed-Content-Hash: design/registry/entities.yaml 02f5c072fc535530c710fd71d77d31e2dce021ed
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/save-system.md 0656b8687f961899dc87f58af176e917bd6c8729
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 3fa3e55b62e80ebcb1ed150a57577416f639faf9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ce3ec63c69b52c965ac670e2b68c708a43fc8100
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 1119e17f51ee38b2b6fdb56db5d6301d6db94f47
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d

---

## Re-Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Specialists: none (lean — Phase 3b not run)
Blocking items: 5 | Recommended: 8
Freshness: `review-receipts.sh check` reports the target doc and this skill `UNCHANGED`, `design/registry/entities.yaml` `CHANGED`. `design/gdd/unlocks-and-stations.md` is byte-identical to the 2026-11-06 review (`729311c1`), so this run is the registry-delta + cross-document re-verification case, not a re-read of new GDD prose.
Prior verdict resolved: Partially — the registry `effective_unlocks` blocker is fixed; the same class of defect survives in three sibling rows of the same owning entry plus one ui-flow contradiction.
Engine/tests: none run — no Godot on this machine; every claim below is a document comparison.

### Prior Blocking Item — Verified Resolved

- **REGISTRY `normalize` / `effective_unlocks` — RESOLVED.** `design/registry/entities.yaml:1047` now reads `S_eff = {s in S : s.unlock_cost = 0} + {s in S : s.id in U.stations and not gated(s)}; stations = sort({s.id : s in S_eff} + U.stations); dishes = sort(U.dishes + union(dish_pool of S_eff))`, with `revised: "2026-11-06"` (`:1050`) and a note (`:1048`) stating that a gated known station stays in `U.stations` and therefore in `stations` but contributes no `dish_pool`, that the result is monotonic, and that unknown ids are preserved. This matches the owning GDD term for term — free set `S_free` (`:192`), gated-paid exclusion from `S_eff` (`:193`), `stations` as the union with all of `U.stations` (`:195`), `dishes` from `S_eff` only (`:196`), the monotonicity/unknown-id claims (`:215`) — and matches ADR-0017 §4 (`docs/architecture/adr-0017-unlocks-station-purchase.md:154-159`). Same free/paid/gated treatment, same `sort()` ordering, same `⊇` claim.
- **GDD text carries no live old formula.** The only `U.stations \ S_ids` occurrences are the explicit rejection of the old form (`design/gdd/unlocks-and-stations.md:200`) and the unknown-id set difference inside the `dishes` line's Output Range (`:215`). No residual `stations = … \ S_ids` construction.
- **Header status agrees.** GDD `:3` `In Review` == `design/gdd/systems-index.md:70` `In Review`. (The older “In Design vs Designed” finding is gone.)
- **Rule 17 / ADR-0017 §6/§5 agree with the GDD.** GDD validation order (`:43`) == ADR-0017 `:187-196`; GDD `unlock_progress` (`:299-307`) == ADR-0017 `:179`; GDD `gated(s)` (`:166`) == ADR-0017 `:138`.

Note on the brief: the unlocks ADR is **ADR-0017** (`adr-0017-unlocks-station-purchase.md`); **ADR-0016** is `meta-upgrades-wallet.md`, relevant here only through `spend`. Both are stamped below.

### Blocking Items

1. **Registry `station_available` still has no gate.** `design/registry/entities.yaml:1032` = `(s.unlock_cost = 0) or (s.id in U.stations)`; the owning GDD `:167` requires `¬gated(s) ∧ ((s.unlock_cost = 0) ∨ (s.id ∈ U.stations))`, and ADR-0017 `:138` states the same. As registered, a gated but already-owned station evaluates available → `true`, directly contradicting rule 17 (`:62`), the explicit example (`:184`) and AC 44 (`:535`).
2. **Registry `unlock_decision` drops `FEATURE_GATED`.** `entities.yaml:1063` lists five outcomes and `:1064` evaluates `ALREADY_UNLOCKED` immediately after the negative-price guard, with no gated branch. The GDD requires six values with `FEATURE_GATED` after `INVALID_STATION_DATA` and before `ALREADY_UNLOCKED` (`:224-231`, `:244`, `:246`), and rule 9 makes that ordering normative (`:43`: `FEATURE_GATED` before `BUSY`/`ALREADY_UNLOCKED`). This is the “FEATURE_GATED semantics” item from the prior review, still open at registry level.
3. **Registry `unlock_progress` counts gated stations.** `entities.yaml:1108` = `stations_total=|S|`, `dishes_total=|D|`, no `¬gated` filter. GDD `:299-307` and `:319` require `S_vis = {s ∈ S : ¬gated(s)}` denominators (`stations_total=|S_vis|`, `dishes_total=Σ_{s∈S_vis}|s.dish_pool|`), which rule 17 (`:61`) and AC 42 (`:533`) pin to MVP `1/1`, `12/12`. Registered as-is, MVP reads `1/2`, `12/24`.
4. **ui-flow contradicts this GDD on gated-station visibility and state model.** `design/gdd/ui-flow.md:59` requires the unlock entry to mark the second station 「不可购买并显示『需要档口选择器』的原因」 and asserts 「不存在『已购买但无法选择』的中间态」; AC 46 (`ui-flow.md:622`) asserts the same for opening the main menu, plus 「也不存在『已解锁档口』的中间状态」. This GDD requires the gated station to be **hidden**, not shown as locked (`:60`, `:116`, `:126`, AC 42 `:533`), and requires the opposite state to exist: a purchased-but-gated station keeps its id and is temporarily unusable (`:65`, `:343`, AC 44 `:535`). The gated **formula** agrees in both documents; the visibility and the intermediate state do not.
5. **`spend` validation order differs from the Accepted ADR.** GDD contract table `design/gdd/unlocks-and-stations.md:135` = `DATA_NOT_READY → NOT_WRITABLE → NOT_IN_MENU → BUSY → INVALID_AMOUNT → INSUFFICIENT_FUNDS`; `docs/architecture/adr-0016-meta-upgrades-wallet.md:248` (Accepted) = `INVALID_AMOUNT → NOT_WRITABLE → NOT_IN_MENU → BUSY → RESERVED_NAMESPACE → INSUFFICIENT_FUNDS` — no `DATA_NOT_READY`, `INVALID_AMOUNT` first. `design/gdd/meta-upgrades.md:122` sides with the ADR (“`amount < 0` 立即拒绝”). Either order can stand, but the GDD's table and the ADR that implements it must state the same one.

### Recommended

1. `unlocks-and-stations.md:200` still cites “ADR-0017 第 1 条”. The rule being cited is ADR-0017 §4 `账本规范化` (`:147-163`) with the GDD Sync Check (`:386-390`); §1 is module shape. Carried, unresolved.
2. `:128` heading still reads “新增 `spend`，需回写其 GDD”, while `:383` says “`spend` 已回写” and `meta-upgrades.md:122` already lists it. Stale heading.
3. `:55` still cites “存档规则 80”; `save-system.md` has 25 core rules. Point at save rule 18 + AC 13, or the records orchestrator contract. Carried, unresolved.
4. `gated(s)` is defined only inside the GDD (`:166`, `:179`) and ADR-0017 §3; it has no registry entry, and `opt_net_profit` (named as a variable at `entities.yaml:1090`) still has no owning entry. Registry additions belong to the orchestration session.
5. `:208` `S_ids` now has no consumer inside the formula (only the prose at `:215`). Keep it with that one-line justification, or drop the row.
6. `StationView` (`:116-123`) still carries no `entry_fee` although UI Requirements (`:441`) requires the catalog row to show 入场费; `ui-flow.md:163` reads `entry_fee` straight from 数据配置. Decide one source. Carried.
7. `NOT_IN_MENU` (`:41`, `:43`) still covers “收据未完成 / 场景切换中” without naming a predicate that supplies those flags. Carried.
8. `runs_to_unlock` (`:288-292`) still divides the whole 120 by optimal net profit as if no upgrade were bought from the same wallet; state the zero-upgrade-spend assumption inside the band. Minor: `:6` `Last Verified: 2026-11-03` trails `Last Updated: 2026-11-06`.

### Verdict: NEEDS REVISION

The document that was blocked (`entities.yaml` `effective_unlocks`) is fixed and now matches the GDD and ADR-0017 §4 exactly, and the GDD prose itself needs no change for that fix. But the same gating contract is still missing from the three sibling registry rows that this GDD owns (`station_available`, `unlock_decision`, `unlock_progress`), `ui-flow.md` still requires the gated station to be visible with a reason and forbids the purchased-but-gated state that rule 17 and AC 44 mandate, and the `spend` order in `:135` disagrees with Accepted ADR-0016 §8. None of these is cosmetic wording; each changes a player-visible or test-asserted behaviour. Fixing them is a registry / ui-flow / contract-table edit — this GDD's rules themselves are unchanged and still coherent.

Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 729311c1e48fad9d1f935240bba7850da718a1af
Reviewed-Content-Hash: design/registry/entities.yaml 5d5722fcce313d3fcbe76129bf56fd232dc80735
Reviewed-Content-Hash: docs/architecture/adr-0016-meta-upgrades-wallet.md 706e35071b31be2f0bb46ec426d40ba95ce430b4
Reviewed-Content-Hash: docs/architecture/adr-0017-unlocks-station-purchase.md c362b96a4151622e13f12a67ea47e450c49c6142
Reviewed-Content-Hash: design/gdd/ui-flow.md 10331708e8493cb2d1022209c086ffc55ec61f49
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/save-system.md fa9f1bff8eede09aaf9a68dc7dcb3f7fb851ba1e
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 3fa3e55b62e80ebcb1ed150a57577416f639faf9
Reviewed-Content-Hash: design/gdd/systems-index.md 2a3dabc219eb8765940d7f6aa93ccd7d4f0e41b9

---

## Re-Review — 2026-11-06 (round 2) — Verdict: APPROVED

Review mode: lean · Workflow: standard
Specialists: none (lean — Phase 3b not run)
Blocking items: 0 | Recommended: 11
Freshness: target doc, registry and ui-flow all `CHANGED` since the previous entry (`design/gdd/unlocks-and-stations.md` `729311c` → `ecf95a3`), so this is a full re-read, not a delta review. Read: the whole GDD by heading, the six registry rows this system owns (`entities.yaml:1021-1111`), `ui-flow.md` rule 8 / AC 46, ADR-0016 §8, ADR-0017 §2–§6 plus its GDD Sync Check, `systems-index.md:70`.
Completeness: 6/6 required sections at `standard` (`Player Fantasy` absent — advisory at this tier; the doc carries `Game Feel` instead).
Prior verdict resolved: Yes — all five cross-document blockers closed by `cafff91`.
Engine/tests: none run — no Godot on this machine; every claim below is a document comparison.

### Prior Blocking Items — Verified

1. **Registry `station_available` gate — RESOLVED.** `design/registry/entities.yaml:1032` now reads `gated(s) = (supports_station_selection() != true) and (s.unlock_cost > 0); station_available = not gated(s) and (s.unlock_cost = 0 or s.id in U.stations)`, term for term with GDD `:166-167` and ADR-0017 `:137-138`. Notes `:1033` cover all four claimed cases — free never gated, unknown id → `false`, `unlock_cost < 0` → `false` as defence, purchased-but-gated keeps its id in `U.stations` and recovers when the switch flips; `revised: "2026-11-06"` at `:1035`. AC 44 (`:535`) now follows from the registered formula instead of contradicting it.
2. **Registry `unlock_decision` — RESOLVED.** `entities.yaml:1063` lists six outcomes including `PREREQUISITE_LOCKED` and `FEATURE_GATED`; `:1064` orders `INVALID_STATION_DATA → FEATURE_GATED → ALREADY_UNLOCKED → PREREQUISITE_LOCKED → INSUFFICIENT_FUNDS → OK`, identical to GDD `:224-231`, to rule 9 `:43` and to ADR-0017 §6 `:185-196`. `PREREQUISITE_LOCKED` is genuinely the sixth outcome (ADR §6 step 9) and not a `DATA_NOT_READY` case: the other five names in the GDD's validation string (`DATA_NOT_READY`, `NOT_WRITABLE`, `NOT_IN_MENU`, `UNKNOWN_STATION`, `BUSY`) are pre-formula guards that ADR §6 also keeps outside the outcome set. `cost = null` note `:1065` covers both cases the GDD names (`:63`, `:120`).
3. **Registry `unlock_progress` — RESOLVED.** `entities.yaml:1108` now defines `S_vis = {s in S : not gated(s)}`, `S_eff ⊆ S_vis`, and takes both totals from `S_vis`; `:1109` states the MVP reading `1/1`, `12/12` and the `NoPlayableStation` no-`0/0` rule. Matches GDD `:299-307`, `:319`, `:321` and ADR-0017 `:179`.
4. **ui-flow visibility — RESOLVED.** `design/gdd/ui-flow.md:59` (rule 8) and `:622` (AC 46) now **hide** the gated paid station: no rendered row, no disabled button, no 「需要档口选择器」 label, excluded from progress (`1/1`, `12/12`), `spend` 0 calls, `unlock` → `FEATURE_GATED`, `supports_station_selection()` the single switch. The label survives only as an explicit negation (`:59`, `:622`); `:166` and `:500` agree. No `1/2` / `12/24` and no locked-row text remain anywhere in ui-flow.
5. **`spend` guard order — RESOLVED.** GDD `:135` now equals ADR-0016 §8 `:248` exactly: `INVALID_AMOUNT` (`typeof != TYPE_INT` or `amount < 0`) → `NOT_WRITABLE` → `NOT_IN_MENU` → `BUSY` → `RESERVED_NAMESPACE` (`extra` 含 `progression`) → `INSUFFICIENT_FUNDS`. Neither side carries a `DATA_NOT_READY` guard, and the GDD now says so explicitly (`:135`), so there is no guard present on one side and absent on the other.

**GDD text leftovers:** no residual copy of the old formula — `U.stations \ S_ids` occurs only as the explicit rejection (`:200`) and inside the unknown-id wording of Output Range (`:215`). The GDD never contained 「需要档口选择器」 in any commit (`git log -S` matches only `ui-flow.md`).

### Blocking Items

- none.

### Recommended

1. GDD `:434` ("锁定档口和它的价格一直在列表里…**不隐藏**、不灰到看不见") and `:423` ("锁定档口可见") state visibility unconditionally, while rule 17 `:60` hides the only paid station in MVP. Scope both to `supports_station_selection() = true` (post-S8) — as written they are the pre-S8 visibility language.
2. GDD `:444` gives the progress example `已解锁 1/2 档口 · 12/24 道菜` unqualified; with the switch false, rule 17 `:61` and AC 42 `:533` require `1/1 · 12/12`. Label it the selector-on example, as `:321` already does.
3. GDD `:126` contradicts itself inside one sentence: "只有 `Unavailable`（数据未 `Ready`）才返回空数组；`NoPlayableStation` 且所有档口都被门禁时也返回空数组". Rewrite as "除 `Unavailable` 外，只有所有档口都被门禁的 `NoPlayableStation` 返回空数组".
4. GDD `:120`'s `cost = null` list omits `INVALID_STATION_DATA` (registry `:1065` includes it) and `UNKNOWN_STATION` (ADR-0017 `:198` includes it). Registry and ADR agree with each other; widen the GDD row to match.
5. ADR-0017 `:163` and its GDD Sync Check `:388` still assert the GDD writes `U.stations \ S_ids`; GDD `:195`/`:200` no longer does. Stale — the ADR is unchanged (`c362b96`) and belongs to the architecture session.
6. GDD `:200` cites "ADR-0017 第 1 条"; the normalization rule it describes is §4 (`:147-163`). Carried.
7. GDD `:55` cites "存档规则 80"; `save-system.md` Core Rules end at 26 (`:58`). Point at save rule 18 + AC 13. Carried.
8. GDD `:128` heading still reads `spend` "需回写其 GDD" while `:383` says 已回写. Stale heading. Carried.
9. Registry-only gaps in rows this system owns: `station_available` `variables:` (`:1027-1030`) omit `supports_station_selection()`, which its expression calls; `unlock_progress` (`:1106`) lists `D`, unused by its expression; `unlock_decision` (`:1058-1062`) omit `requires(s)`. Registry edits belong to the orchestration session.
10. GDD `:116` `StationView` still has no `entry_fee` although `:441` requires the catalog row to show 入场费 (ADR-0017 `:178` likewise omits it). Carried. Also `:5` `Last Verified: 2026-11-03` trails `Last Updated`, and OQ 5 (`:552`, "锁定档口是否从第一局就可见") is already answered for MVP by rule 17.
11. ui-flow `:622` (and `:59`) assert 「不存在『已购买但无法选择』…『已解锁档口』的中间状态」, while GDD AC 44 (`:535`) constructs exactly a purchased-but-gated ledger. The invariant is correct for states reachable without a selector; scope the sentence to that rather than to any ledger.

**Header status:** GDD `:3` `In Review` == `design/gdd/systems-index.md:70` `In Review`. Index not edited.

### Verdict: APPROVED

All five cross-document blockers of the previous round are closed in `cafff91`, and the closing edits are exact rather than approximate: the three registry rows now encode the same `gated(s)` / `S_vis` / six-outcome contract as the GDD and ADR-0017, ui-flow hides the gated station everywhere instead of demanding a reason label, and the `spend` order is byte-comparable with ADR-0016 §8. The GDD's own normative text — rule 17, the API table, `station_available` / `effective_unlocks` / `unlock_decision` / `unlock_progress`, and ACs 42–45 — is internally consistent, `get_station_catalog()` lists only non-gated stations in every statement of it, and every progress denominator excludes gated stations (MVP `1/1`, `12/12`). The eleven recommended items are documentation hygiene: three unscoped legacy sentences in the GDD's UI/Game-Feel sections whose rule 17 is still normative and AC-pinned, one self-contradictory sentence about an empty catalog, two narrower-than-ADR null-cost lists, three stale cross-references, two registry-side omissions, and one ui-flow invariant that reads absolutist where AC 44 makes it conditional. None changes a rule, formula, AC or contract, so none blocks approval.

Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md ecf95a3033756e6111b19a650435d745ea7fd2b0
Reviewed-Content-Hash: design/registry/entities.yaml 0646d0161ee8488d78ec5454abe3a61d0d0a14f9
Reviewed-Content-Hash: design/gdd/ui-flow.md 05d5a22e1e6376e45b9863166f8dd32310e7f006
Reviewed-Content-Hash: docs/architecture/adr-0017-unlocks-station-purchase.md c362b96a4151622e13f12a67ea47e450c49c6142
Reviewed-Content-Hash: docs/architecture/adr-0016-meta-upgrades-wallet.md 706e35071b31be2f0bb46ec426d40ba95ce430b4
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/systems-index.md 4714602ef782db915793ed55ee8f3c1f089ee886
Reviewed-Content-Hash: design/gdd/save-system.md a7d54e3ba4d7f8a23be1d28a6c165bb422648abd
Reviewed-Content-Hash: design/gdd/data-config.md cd42141a823868fc47650210cb108c0ae69259a0
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 3fa3e55b62e80ebcb1ed150a57577416f639faf9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 6964692c5d4850582e014c362342bcdb2ea0b28b
