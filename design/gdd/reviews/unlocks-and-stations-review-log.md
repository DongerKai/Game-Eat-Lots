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
