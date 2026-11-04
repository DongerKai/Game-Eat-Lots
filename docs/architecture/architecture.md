# 回本 (Break Even) — Master Architecture

## Document Status

- Version: 1
- Last Updated: 2026-11-04
- Engine: Godot 4.7 (pinned; installed version NOT DETERMINED — no editor on this machine)
- GDDs Covered: 18 (all designed systems; #19 在线挑战 and #20 同屏合作 are Not Started)
- ADRs Referenced: ADR-0001 内容数据格式；ADR-0002 持久化格式；ADR-0003 场景与屏幕架构；ADR-0004 服务组合与信号；ADR-0005 输入上下文栈；ADR-0006 确定性随机；ADR-0007 局内状态与结算事务；ADR-0008 设置与本地化；ADR-0009 音频播放与暂停（均为 Proposed）
- Architecture Review: 2026-11-04 — FAIL — 56 covered / 4 partial / 44 gaps；ADR-0003↔0005↔0004 依赖环（见 `docs/architecture/architecture-review-2026-11-04.md`）
- TD-ARCHITECTURE: 2026-11-04 — skipped — lean mode
- LP-FEASIBILITY: 2026-11-04 — skipped — lean mode

> This document is the standard-tier architecture: layer map, ownership, key data
> flows, API boundaries and the critical ADR list. It targets Godot 4.7; every
> engine API named here is cross-referenced against
> `docs/engine-reference/godot/`. The full 104-requirement baseline is Appendix A.

## Engine Knowledge Gap Summary

| Field | Value |
|---|---|
| Engine | Godot 4.7 (latest patch 4.7.2, Aug 2026) |
| LLM training covers | approximately up to 4.3 / May 2025 |
| Post-cutoff versions | 4.5, 4.6, 4.7 — all HIGH risk |
| Installed editor | NOT DETERMINED — Godot is not installed on this machine |

**HIGH risk domains for this game**

- **UI** — 4.6 separates mouse/touch focus from keyboard/gamepad focus; 4.5 adds AccessKit screen-reader support; 4.7 renames `RichTextLabel.ImageUpdateMask.UPDATE_WIDTH_IN_PERCENT` → `UPDATE_WIDTH_UNIT` and retypes `add_image()`/`update_image()` parameters. The whole game is Control-based, so this domain is load-bearing.
- **Input** — 4.7 reports mouse/keyboard as `InputEvent.DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD`, not device `0`. Identify events by type, never `device == 0`.
- **Rendering** — 4.7 draws `CanvasItem` lines without an antialiasing feather, so 1px separators look thinner; widen lines that relied on the old look. New 4.7 projects default to `display/window/stretch/mode = canvas_items`, `aspect = expand`.
- **Animation** — `Tween.tween_await()` is new in 4.7; UI transitions use `Tween`, so it is available but not required.

**MEDIUM risk domains**

- **Core / persistence** — `FileAccess.store_*` returns `bool` since 4.4 (was `void`). Save writes must check the return value.
- **Audio** — 4.7 changes `area_mask` on `AudioStreamPlayer2D/3D` from layer 1 to 0 (disabled). This game uses non-positional `AudioStreamPlayer`, so the impact is limited to any future `audio_bus_override` area.
- **Resources** — `duplicate_deep()` (4.5) is the explicit deep-copy API for nested resource trees; `duplicate()` keeps its old shallow-ish behaviour.

**Systems touching HIGH/MEDIUM domains**

| GDD system | Domain | Risk |
|---|---|---|
| 界面流程, 局内 HUD, 首局引导 | UI | HIGH |
| 输入 | Input | HIGH |
| 存档 | Core / persistence | MEDIUM |
| 音效反馈 | Audio | MEDIUM |
| 菜单波次, 菜品数值, 结算 | Rendering / Core | LOW–MEDIUM |

The engine reference is **ahead of the installed editor**: `Installed at pin time`
is `NOT DETERMINED`. Pin the editor to 4.7.x before writing version-qualified
API calls, and re-verify the HIGH-risk rows above on that install.

## System Layer Map

```
┌──────────────────────────────────────────────────────────────┐
│ PRESENTATION  In-Game HUD · UI Flow · Audio Feedback ·        │
│               First-Run Onboarding                           │
├──────────────────────────────────────────────────────────────┤
│ FEATURE       Cloying & Taste · Menu Waves · Settlement ·     │
│               Meta Upgrades · Unlocks & Stations · Records   │
├──────────────────────────────────────────────────────────────┤
│ CORE          Run Flow · Time & Turns · Stomach Capacity ·    │
│               Dish Values                                    │
├──────────────────────────────────────────────────────────────┤
│ FOUNDATION    Data Config · Input · Scene Flow · Save        │
├──────────────────────────────────────────────────────────────┤
│ PLATFORM      Godot 4.7 / PC (Steam, Epic)                    │
└──────────────────────────────────────────────────────────────┘
```

| # | System | Layer | Why this layer |
|---|---|---|---|
| 1 | 数据配置 Data Config | Foundation | Boot-time catalogs and validation gate every other system. |
| 2 | 输入 Input | Foundation | Engine-facing context/focus layer; all presentation and gameplay read it. |
| 3 | 场景与流程管理 Scene Flow | Foundation | Sole owner of top-level state and payload handoff. |
| 4 | 存档 Save | **Foundation** | Persistence, locking, migration and atomic write are infrastructure. The systems index lists it under Core; this document moves it to Foundation because its dependencies are engine/OS APIs, not gameplay state. |
| 5 | 餐局流程 Run Flow | Core | Owns the run lifecycle and action arbitration. |
| 6 | 时间与回合 Time & Turns | Core | Owns one of the two per-run budgets; pure arithmetic. |
| 7 | 胃容量 Stomach Capacity | Core | Owns the other per-run budget; pure arithmetic. |
| 8 | 菜品数值 Dish Values | Core | Static comparison scale (`unit_value`) and catalog queries. |
| 9 | 腻度与口味 Cloying & Taste | Feature | Mutates effective satiety during a run; read by Run Flow. |
| 10 | 菜单波次 Menu Waves | Feature | Seeded partition + health gate; feeds visible set. |
| 11 | 结算与入场费 Settlement | Feature | Pure `RunLog → RunResult` computation. |
| 12 | 局外升级 Meta Upgrades | Feature | Wallet and upgrade levels; writes through Save. |
| 13 | 解锁与档口 Unlocks & Stations | Feature | Derived availability; spends through Meta Upgrades. |
| 14 | 纪录与里程碑 Records | Feature | Best-run records and milestones; writes through Save. |
| 15 | 局内 HUD In-Game HUD | Presentation | Read-only projection of run state. |
| 16 | 界面流程 UI Flow | Presentation | Screen/modal stacks, receipt orchestration, settings model. |
| 17 | 音效反馈 Audio Feedback | Presentation | Event→cue mapping; consumes UI/run signals. |
| 18 | 首局引导 First-Run Onboarding | Presentation | Overlay beats over HUD and receipt; no persistence. |
| 19 | 在线挑战 Online Challenges | Not Started | Deferred; would add Networking + Save schema v2. |
| 20 | 同屏合作 Co-op | Not Started | Deferred; would add Input/Scene Flow split. |

**Engine risk flags**

- `界面流程` / `局内 HUD` / `首局引导` → UI → **HIGH**. Use Control/Theme, verify dual-focus and AccessKit behaviour on 4.7.
- `输入` → Input → **HIGH**. Type-check events; never compare `device == 0`.
- `存档` → persistence → **MEDIUM**. `FileAccess.store_*` returns `bool`.
- `音效反馈` → Audio → **MEDIUM**. `AudioStreamPlayer` only; no positional audio.

## Module Ownership

| Module | Owns | Exposes | Consumes | Engine APIs (all 4.7) |
|---|---|---|---|---|
| Data Config | Dish/Upgrade/Station/Balance catalogs, validation state | `get_dish`, `get_upgrade`, `get_station`, `get_balance`, `ready`/`failed` | Files on disk | `Resource`, `FileAccess`, `ResourceLoader` |
| Input | Context stack, action consumption, focus + hover owners | `push_context`, `pop_context`, `consume_action`, `get_focus_owner`, `get_hover_owner` | `InputEvent`s | `Input`, `Control`, `InputEvent.DEVICE_ID_*` |
| Scene Flow | Top-level state, transition validity, payload handoff | `request_transition`, `state`, `state_changed` | Nothing | `Node`, `SceneTree`, `process_mode` |
| Save | Envelope, lock, migration, atomic write, namespace values | `load`, `retry_load`, `commit`, `flush`, `get_namespace`, `delete_career`, `save_status_changed` | Data Config ids | `FileAccess`, `DirAccess`, `Time` (display only), `OS` |
| Run Flow | `run_id`, lifecycle state, action arbitration, `RunLog` | `submit_action`, `snapshot`, `run_started`, `action_committed`, `run_ended`, `run_settled` | Time, Capacity, Menu Waves, Cloying | none (plain `RefCounted`/`Node`) |
| Time & Turns | `turns_total`, `turns_remaining`, `turns_used` | `initialize`, `can_spend_turn`, `spend_turn`, `is_exhausted` | None | none |
| Stomach Capacity | `capacity_total`, `capacity_used`, run boundary | `initialize`, `can_fit`, `consume`, `end_run` | Upgrade snapshot | none |
| Dish Values | Static dish data view, `unit_value`, ordering | `get_dish`, `unit_value`, `uv_cmp` | Data Config | none |
| Cloying & Taste | Group counters, resistance, cleanse target | `initialize`, `effective_satiety`, `extra_satiety`, `group_counts`, `end_run` | Dish Values, Data Config | none |
| Menu Waves | Partition, `wave_index`, remaining dishes, health report | `initialize`, `visible_dishes`, `preview_dishes`, `consume`, `advance`, `end_pickup`, `health` | Data Config, Dish Values | `RandomNumberGenerator` (seeded) |
| Settlement | `RunLog → RunResult` | `settle` | Dish Values, Data Config | none |
| Meta Upgrades | Wallet, upgrade levels, catalog view | `get_wallet`, `get_catalog`, `can_purchase`, `purchase`, `resolve_upgrade_snapshot`, `prepare_run_result`, `spend` | Save, Data Config | none |
| Unlocks & Stations | Effective unlock set, availability | `normalize`, `is_available`, `purchase`, `effective_unlocks` | Save, Meta Upgrades, Data Config | none |
| Records | Best-run records, career aggregates, milestones | `prepare_run`, `snapshot`, `milestone_progress`, signals | Save, Data Config | none |
| In-Game HUD | `HudFrame`, focus/hover presentation, anchor rects | `snapshot`, `get_anchor_rect`, `submit_action` (UI intent) | Run Flow, Menu Waves, Cloying, Capacity, Time, Settlement | `Control`, `Theme`, `Tween` |
| UI Flow | Screen stack, modal stack, focus order, settings model, receipt orchestration | `push_screen`/`pop_screen`, semantic events, `settings` | Scene Flow, Save, Meta Upgrades, Records, Run Flow | `Control`, `Theme`, `Tween`, `Window` |
| Audio Feedback | Buses, voice pool, cue catalog, cooldowns | `play(cue_id)`, `set_bus_volume`, mute | UI Flow/run signals, settings | `AudioStreamPlayer`, `AudioServer` |
| First-Run Onboarding | Beat index, explanation payload | `get_first_run_preset`, `onboarding_finished` | Run Config, HUD anchors, Data Config | `Control` |

**Dependency direction**

```
Data Config ← every content consumer
Scene Flow  ← UI Flow, Run Flow
Input       ← UI Flow, HUD, Onboarding
Save        ← Meta Upgrades, Unlocks, Records, UI Flow
Run Flow    → Time, Capacity, Menu Waves, Cloying
Settlement  ← Run Flow (RunLog)
UI Flow     → Scene Flow, Save, Meta Upgrades, Records
HUD         → Run Flow (read-only snapshot)
Audio       ← UI Flow + Run Flow signals
```

No gameplay module reads another module's mutable state directly: it reads a
snapshot or calls an exposed method. The one deliberate exception is the
run-action path, where `RunFlow` calls the four run subsystems in a fixed order
so the transaction can roll back as a unit.

## Data Flow

### 1. Boot and initialisation order

```
AppRoot (composition root)
  → Data Config.load()          # Unloaded → Loading → Validating
  → Data Config.ready           # gates everything below
  → Save.load()                 # after Data Config is Ready
  → Scene Flow Boot → Menu      # exactly one top-level transition
  → UI Flow builds MenuRoot     # reads LoadReport, shows Recovered/Blocked
```

If `Data Config` fails, `Scene Flow` goes `Boot → Error`; no gameplay module is
constructed. Save load never blocks the transition to `Menu` (save rule 11).

### 2. Run start

```
UI Flow  assemble RunConfig
  → Scene Flow.request_transition(Menu → Run, RunConfig)
  → Run Flow.initialize(RunConfig)
      → Time.initialize(turn_budget)
      → Capacity.initialize(upgrade_snapshot)
      → Menu Waves.initialize(station_id, seed, turn_budget, menu_vision[, pool_override])
      → Cloying.initialize(cloying_resistance)
  → run_started
  → HUD builds the first HudFrame
```

All four subsystem initialisations happen in one frame. If any returns an error,
`Run Flow` emits `init_failed`; no `RunResult` is produced.

### 3. Action frame (`eat`)

```
Input.context = Run
  → HUD resolves focused dish
  → Run Flow.submit_action(EAT, dish_id)
      1. Menu Waves: dish still in visible set?
      2. Cloying: effective_satiety(dish_id)
      3. Capacity: can_fit(effective_satiety)?
      4. Time: can_spend_turn()?
      → atomic commit:
           append EatenEntry(dish_id, extra, effective)
           Cloying.increment(dish_id)
           Menu Waves.consume(dish_id)
           Time.spend_turn()
      → end check (TIME_EXHAUSTED / NO_EATABLE_DISH)
  → action_committed / run_ended
  → HUD refreshes one HudFrame
```

`skip_wave` and `end_pickup` follow the same validate-then-commit shape with
their own validation order. No subsystem writes state before the whole action
passes validation.

### 4. Run end and settlement

```
Run Flow: Ending → assemble RunLog
  → Settlement.settle(RunLog) → RunResult
  → Run Flow: Settling → Handoff
  → Scene Flow: Run → Menu (payload RunResult)
  → UI Flow: receipt orchestration, same frame
      Records.prepare_run(RunResult)
      Meta Upgrades.prepare_run_result(RunResult)
      Save.commit({progression, records, unlocks})     # exactly one write; unlocks empty in MVP
  → ReceiptScreen renders RunResult
```

A failed record consistency check aborts the whole transaction. A failed disk
write leaves the in-memory namespace values dirty and retries at the next write
point; the receipt shows the unsaved marker.

### 5. Save / load path

- **Producer**: Meta Upgrades, Unlocks, Records produce namespace values; UI Flow
  merges them into one transaction.
- **Owner**: Save owns envelope, validation, migration, atomic write and lock.
- **Consumer**: boot reads `LoadReport`; UI Flow renders it; gameplay modules
  receive namespace values only after a successful load.
- **Threading**: single-threaded main thread. No background save thread in MVP;
  the write budget is p95 ≤ 50 ms with a ~8.5 KiB Full Vision file.

### 6. Signals and snapshots

Past-tense signals for notification: `state_changed`, `run_started`,
`action_committed`, `run_ended`, `run_settled`, `save_status_changed`,
`progression_changed`, `record_broken`, `milestone_earned`, `career_completed`,
`ui_button_activated`, `receipt_closed`.

HUD and UI screens never poll gameplay state between events: each event produces
one immutable `HudFrame` / screen model. Synchronous contracts (run action,
purchase, commit) are direct calls with result values, not signals.

## API Boundaries

Contracts below are design-level GDScript. Names match the GDDs.

```gdscript
# Foundation

class DataConfig:
    signal ready
    signal failed(error: DataError)
    func load() -> Result[void, DataError]
    func get_dish(id: StringName) -> Result[DishData, QueryError]
    func get_upgrade(id: StringName) -> Result[UpgradeData, QueryError]
    func get_station(id: StringName) -> Result[StationData, QueryError]
    func get_balance() -> BalanceData

class SceneFlow:
    var state: SceneState
    signal state_changed(from: SceneState, to: SceneState, payload: Variant)
    func request_transition(to: SceneState, payload: Variant = null) -> bool

class InputRouter:
    func push_context(ctx: InputContext) -> void
    func pop_context() -> void
    func consume_action(action: StringName) -> bool
    func get_focus_owner() -> Control
    func get_hover_owner() -> Control

class SaveSystem:
    signal save_status_changed(status: SaveStatus)
    func load() -> LoadReport
    func retry_load() -> LoadReport
    func get_namespace(owner: StringName) -> Dictionary   # read-only
    func commit(tx: Dictionary) -> CommitResult            # atomic
    func flush() -> FlushResult
    func delete_career(token: ConfirmToken) -> Result[void, SaveError]

# Core

class RunFlow:
    signal run_started(run_id: int)
    signal action_committed(action: RunAction, payload: Dictionary)
    signal run_ended(reason: EndReason)
    signal run_settled(result: RunResult)
    func submit_action(action: RunAction, dish_id: StringName = &"") -> ActionResult
    func snapshot() -> RunSnapshot

class TimeTurns:
    func initialize(turn_budget: int) -> Result[void, TurnError]
    func can_spend_turn() -> bool
    func spend_turn() -> TurnResult
    func is_exhausted() -> bool

class StomachCapacity:
    func initialize(snapshot: UpgradeSnapshot) -> Result[void, CapacityError]
    func can_fit(effective_satiety: int) -> bool
    func consume(effective_satiety: int) -> CapacityResult
    func end_run(reason: EndReason) -> void

# Feature

class CloyingAndTaste:
    func initialize(resistance: int) -> void
    func effective_satiety(dish: DishData) -> int
    func extra_satiety(dish: DishData) -> int
    func end_run(reason: EndReason) -> void

class MenuWaves:
    func initialize(station_id: StringName, seed: int, turn_budget: int,
                    menu_vision: int, pool_override: Array[StringName] = []) -> WaveInitResult
    func visible_dishes() -> Array[DishData]
    func preview_dishes() -> Array[DishData]
    func hidden_counts() -> Array[int]
    func consume(dish_id: StringName) -> ConsumeResult
    func advance() -> AdvanceResult
    func end_pickup() -> EndPickupResult
    func health() -> HealthEval

class Settlement:
    func settle(log: RunLog) -> Result[RunResult, SettlementError]

class MetaUpgrades:
    func resolve_upgrade_snapshot() -> UpgradeSnapshot
    func purchase(id: StringName) -> PurchaseResult
    func prepare_run_result(result: RunResult) -> ProgressionResult
    func spend(amount: int, reason: StringName, extra: Dictionary = {}) -> SpendResult

class Records:
    func prepare_run(result: RunResult) -> RecordResult
    func snapshot() -> RecordsSnapshot
```

**Invariants**

- No caller writes another module's state. Mutation goes through the owner's
  method or a `Save.commit` namespace.
- Every result type has a closed error vocabulary; no `null` / `0` placeholder
  on failure.
- `RunFlow.submit_action` is the only entry to a run action; HUD/UI submit
  intent, never mutate.
- `Save.commit` is the only persistence write; all namespace changes in one
  transaction land atomically.
- `SceneFlow.request_transition` is the only top-level transition entry; UI and
  gameplay request, never assign state.
- Signals are notification only; no signal handler may synchronously change the
  emitting module's state.

## ADR Audit

Nine Foundation/Feature ADRs are written and `Proposed`; none of the Required
ADRs remains unwritten. The TR registry holds all 104 table requirements; ADR
linkage below is authoritative.

| ADR | Engine Compat | Version | GDD Linkage | Conflicts | Valid |
|---|---|---|---|---|---|
| ADR-0001 内容数据格式、校验与启动门 | ✅ | ✅ 4.7 | ✅ TR-data-config-001..006, TR-records-004 | None | ⚠️ Proposed — engine verification pending |
| ADR-0002 持久化格式、原子写入与跨平台写锁 | ✅ | ✅ 4.7 | ✅ TR-save-system-001..007, TR-ui-flow-005, TR-records-001 | None | ⚠️ Proposed — engine verification pending |
| ADR-0003 场景与屏幕架构 | ✅ | ✅ 4.7 | ✅ TR-scene-flow-001..005, TR-ui-flow-001..003, TR-in-game-hud-002 | None | ⚠️ Proposed — engine verification pending |
| ADR-0004 服务组合与信号机制 | ✅ | ✅ 4.7 | ✅ TR-audio-feedback-001/005, TR-run-flow-006, TR-save-system-006, TR-scene-flow-003/005, TR-ui-flow-004 | None | ⚠️ Proposed — engine verification pending |
| ADR-0005 输入上下文栈语义 | ✅ | ✅ 4.7 | ✅ TR-input-001..006, TR-first-run-onboarding-002, TR-in-game-hud-004, TR-scene-flow-002, TR-ui-flow-001 | None | ⚠️ Proposed — engine verification pending |
| ADR-0006 确定性随机、种子派生与健康校验预算 | ✅ | ✅ 4.7 | ✅ TR-menu-waves-001..006, TR-dish-values-003, TR-run-flow-002/003 | None | ⚠️ Proposed — engine verification pending |
| ADR-0008 机器级设置与本地化管线 | ✅ | ✅ 4.7 | ✅ TR-ui-flow-006, TR-ui-flow-003（呈现侧） | None | ⚠️ Proposed — engine verification pending |
| ADR-0007 局内状态所有权与结算事务边界 | ✅ | ✅ 4.7 | ✅ TR-run-flow-001/002/005, TR-settlement-001/004, TR-ui-flow-004, TR-meta-upgrades-005, TR-records-002 | None | ⚠️ Proposed — engine verification pending |
| ADR-0009 音频播放、池化与暂停语义 | ✅ | ✅ 4.7 | ✅ TR-audio-feedback-001..007, TR-ui-flow-007 | None | ⚠️ Proposed — engine verification pending |

**Traceability coverage:** 56 of 104 requirements covered by a Proposed ADR,
4 partial, 44 gaps. No requirement is covered by an Accepted ADR — all nine
ADRs remain `Proposed`. The per-requirement matrix is authoritative:
`docs/architecture/requirements-traceability.md` (review:
`docs/architecture/architecture-review-2026-11-04.md`).

## Required ADRs

Grouped by layer, Foundation first. Each group lists the TR areas it must cover;
`/architecture-review` will register the permanent TR IDs.

**Foundation — must exist before coding starts**

1. **Content data format, validation and boot gate** — `Resource` vs JSON/CSV, id rules, `DataConfig` state machine, milestone catalog loader. Covers `TR-data-config-*`, `TR-records-004`. **Written: ADR-0001 (Proposed).**
2. **Persistence format, atomic write and cross-platform lock** — JSON vs `store_var` vs binary, int64 encoding, path/naming, lock mechanism, migration harness. Covers `TR-save-system-001..007`. **Written: ADR-0002 (Proposed).**
3. **Scene and screen architecture** — one root controller vs one scene per screen; screen stack, modal stack, focus neighbours, dual-focus presentation. Covers `TR-scene-flow-*`, `TR-ui-flow-001..003`, `TR-in-game-hud-002`. **Written: ADR-0003 (Proposed).**
4. **Service composition and signal mechanism** — composition root vs autoloads, dependency injection, signal ordering/reentrancy, paused `process_mode`. Covers the cross-cutting lifecycle contracts and `TR-audio-feedback-005`. **Written: ADR-0004 (Proposed).**
5. **Input context stack semantics** — `push_context` on a same-value top, `Blocked` allow-lists, action consumption, nav repeat. Covers `TR-input-001..006`. **Written: ADR-0005 (Proposed).**

**Core / Feature — before the relevant system is built**

6. **Deterministic RNG and seeded generation** — `RandomNumberGenerator` seeding, partition stability, health-check DP budget. Covers `TR-menu-waves-001..006`. **Written: ADR-0006 (Proposed).**
7. **Run state ownership and settlement transaction boundary** — who owns `run_id`, `RunLog`, the single `Save.commit`, and the receipt orchestration order. Covers `TR-run-flow-*`, `TR-settlement-*`, `TR-ui-flow-004`. **Written: ADR-0007 (Proposed).**
8. **Settings persistence and localization pipeline** — machine-level file format/path/write timing, `tr()` key strategy, CJK font fallback, 150% scaling. Covers `TR-ui-flow-006`, `TR-ui-flow-003` presentation aspects. **Written: ADR-0008 (Proposed).**
9. **Audio playback, pooling and pause semantics** — bus graph, voice pool, cue catalog location, pause behaviour. Covers `TR-audio-feedback-001..007`. **Written: ADR-0009 (Proposed).**

**Can defer to implementation**

10. Shader/theme asset pipeline details, icon placeholder policy, and the exact `Theme` resource structure.

## Architecture Principles

1. **One owner per number.** Every gameplay value has exactly one owning system;
   HUD, UI Flow and Settlement read snapshots and never recompute.
2. **Read-only presentation.** Presentation receives immutable frames/models and
   emits intent; it never mutates gameplay state directly.
3. **Single-writer persistence, one transaction orchestrator.** Save is the only
   writer; UI Flow merges settlement namespaces into one `commit`.
4. **Frozen per-run snapshots.** Upgrade snapshot, entry fee, partition, capacity
   and turn budget are frozen at run start; upgrades affect the next run only.
5. **Deterministic pure core.** Integer arithmetic, seeded RNG, no wall clock in
   logic; UTC timestamps are display-only and save round trips are byte-identical.
6. **Result-style APIs with closed error vocabularies.** No exceptions, no silent
   defaults, no `0`/`null` placeholders on failure.
7. **Data-driven content.** All tuning values live in typed resources validated at
   boot; no balance literals in GDScript.
8. **Composition root over singletons.** Gameplay services are constructed and
   injected at boot; autoloads are limited to the boot/orchestration root.
9. **Signals notify, direct calls contract.** Cross-system notification uses
   past-tense signals; synchronous contracts (run action, purchase, commit) use
   calls with result values.

## Open Questions

| ID | Summary | Priority | Resolution path |
|---|---|---|---|
| QQ-01 | Godot 4.7 is pinned but no editor is installed; HIGH-risk UI/Input APIs are unverified on the actual build. | High | Install 4.7.x, verify dual-focus, `DEVICE_ID_*`, AccessKit, line AA, stretch defaults. |
| QQ-02 | Save serialization format and int64 encoding (JSON double precision vs `store_var` vs binary). | High | ADR 2 |
| QQ-03 | Cross-platform write-lock mechanism (PID file, named mutex, port lock). | High | ADR 2 |
| QQ-04 | Signal ordering and reentrancy; whether a signal handler may request a transition. | High | ADR 4 |
| QQ-05 | Screen controller structure: one root controller vs one scene per screen; manual vs automatic focus neighbours. | High | ADR 3 |
| QQ-06 | `push_context(Modal)` when the top is already `Modal` — same-value push semantics. | High | ADR 5 |
| QQ-07 | Settlement transaction boundary: UI Flow as orchestrator vs a dedicated transaction service. | High | Resolved — ADR-0007 D2–D4 (UIRoot calls, SettlementTransaction implements) |
| QQ-08 | Settings file format/path/write timing and CJK font fallback. | Medium | Resolved — ADR-0008 §1–7 |
| QQ-09 | Audio `process_mode` under pause; cue catalog data location (no `design/audio/` yet). | Medium | Resolved — ADR-0009 D3/D7 (`PROCESS_MODE_ALWAYS`, `res://data/audio/cue_catalog.tres`) |
| QQ-10 | Whether `/review-all-gdds` runs before the Technical Setup gate; it is still ABSENT. | Medium | Resolved — ran 2026-11-04 (`design/gdd/gdd-cross-review-2026-11-04.md`); blocking findings closed and re-reviewed |
| QQ-11 | Full Vision systems (#19/#20) will add schema v2 and an input/scene split; not designed. | Low | When #19/#20 start |
| QQ-12 | Fresh-baseline writer: Save cannot depend on Unlocks to merge `unlocks.normalize()` into the first commit; the boot orchestrator identity is unassigned. | High | Resolved — ADR-0004 D5 (`AppRoot.BootSequence`) |
| QQ-13 | UI Flow action delivery under `Menu`/`Modal`/`Paused`: `InputRouter.consume_action` is Blocked/onboarding-only in the input GDD; the screen-side path is undefined. | High | Resolved — ADR-0005 D4 (`ActionSink`, first consumer wins) |

## Appendix A — Technical Requirements Baseline

Extracted from 18 GDDs | 104 requirements | 104 registered in
`docs/architecture/tr-registry.yaml`

| Req ID | GDD | System | Requirement | Domain |
|---|---|---|---|---|
| TR-data-config-001 | data-config.md | 数据配置 | Four typed Resource catalogs (`DishData`/`UpgradeData`/`StationData`/`BalanceData`) loaded and validated at boot; any failure → `Failed`, no partial load | Data |
| TR-data-config-002 | data-config.md | 数据配置 | Query API `get_dish/get_upgrade/get_station/get_balance` returns immutable records; no runtime mutation | Data |
| TR-data-config-003 | data-config.md | 数据配置 | `StationData` carries `entry_fee`, `unlock_cost`, `dish_pool`, `cloy_cap`, `turn_budget`, `wave_size`; `UpgradeData` carries `cost`, `cost_growth`, `max_level`, `effect_type`, `effect_value` | Data |
| TR-data-config-004 | data-config.md | 数据配置 | Stable id namespace (`^[a-z][a-z0-9_]*$`, typed prefixes); post-release rename forbidden | Data |
| TR-data-config-005 | data-config.md | 数据配置 | Boot state machine `Unloaded→Loading→Validating→Ready/Failed`; `Ready` gates every downstream system | Core |
| TR-data-config-006 | data-config.md | 数据配置 | Records carry `display_name` and optional `Texture2D icon`; display-only, placeholder on missing icon | Rendering |
| TR-input-001 | input.md | 输入 | Fixed action set (`ui_up/down/left/right/accept/cancel/pause`, `game_skip_wave`); no system outside input listens to raw events | Input |
| TR-input-002 | input.md | 输入 | Context state machine `Blocked/Menu/Run/Paused/Modal`; `request_context`/`push_context`/`pop_context` with temporary stack and clear semantics | Input |
| TR-input-003 | input.md | 输入 | Dual focus: mouse hover and keyboard/gamepad focus recorded and drawn separately | Input |
| TR-input-004 | input.md | 输入 | Full keyboard/gamepad play; nav repeat delay and stick hysteresis 0.60/0.40 | Input |
| TR-input-005 | input.md | 输入 | Godot 4.7 device handling: identify by event type, never `device == 0` | Input |
| TR-input-006 | input.md | 输入 | `consume_action(action)` for `Blocked` allow-lists; first consumer wins per frame | Input |
| TR-scene-flow-001 | scene-flow.md | 场景与流程管理 | Exactly one top-level state among `Boot/Error/Menu/Run/Paused/Quit`; sole owner of transitions | Core |
| TR-scene-flow-002 | scene-flow.md | 场景与流程管理 | `request_transition(to, payload) -> bool`; per-frame validation, first legal request wins, payload handoff | Core |
| TR-scene-flow-003 | scene-flow.md | 场景与流程管理 | `Boot` waits for data `Ready/Failed`; `Error` cannot enter `Menu`/`Run`; debug reload only | Core |
| TR-scene-flow-004 | scene-flow.md | 场景与流程管理 | Pause freezes run subsystems; resume restores exact state; no turn/settlement advance while paused | Core |
| TR-scene-flow-005 | scene-flow.md | 场景与流程管理 | `state_changed(from,to)` signal; transitions take effect same frame; fades ≤0.2 s are presentation-only | Core |
| TR-save-system-001 | save-system.md | 存档 | Envelope plus `progression`/`unlocks`/`records` namespaces | Save/Load |
| TR-save-system-002 | save-system.md | 存档 | Atomic write order: validate → temp → flush → primary→backup rename → temp→primary rename; failure leaves original intact | Save/Load |
| TR-save-system-003 | save-system.md | 存档 | Six load outcomes + `LoadReport` (`outcome`, reason, warnings, `writable`, `lock_live`) | Save/Load |
| TR-save-system-004 | save-system.md | 存档 | Single-writer lock; live-lock `Blocked` offers only exit/retry; new-career and delete require `lock_live = false` | Core |
| TR-save-system-005 | save-system.md | 存档 | One-way migration chain of pure per-version functions; failure → `Blocked`; unknown ids preserved | Save/Load |
| TR-save-system-006 | save-system.md | 存档 | `commit({namespace: value})` atomic transaction; purchase rollback vs settlement deferred-retry | Save/Load |
| TR-save-system-007 | save-system.md | 存档 | Deterministic serialization: sorted map keys, lossless int64, byte-identical round trip excluding `last_saved_at` | Save/Load |
| TR-run-flow-001 | run-flow.md | 餐局流程 | Run lifecycle `Idle/Initializing/Active/Ending/Settling/Handoff/Failed/Faulted`; exactly one `run_id` per run | Core |
| TR-run-flow-002 | run-flow.md | 餐局流程 | Init order: turns → capacity → menu waves → cloying in one frame; failure → `init_failed`, no `RunResult` | Core |
| TR-run-flow-003 | run-flow.md | 餐局流程 | `submit_action(eat/skip_wave/end_pickup, dish_id)` with fixed validation order and atomic commit | Core |
| TR-run-flow-004 | run-flow.md | 餐局流程 | Only two end reasons (`TIME_EXHAUSTED`, `NO_EATABLE_DISH`); fixed priority; state frozen after `Ending` | Core |
| TR-run-flow-005 | run-flow.md | 餐局流程 | `RunLog` assembled from action-time values; `settle(RunLog)` once; `RunResult` handed off once; `run_settled` is a pre-notification and persistence happens only after the `Run -> Menu` delivery is accepted | Core |
| TR-run-flow-006 | run-flow.md | 餐局流程 | Signals `run_started/action_committed/run_ended/run_settled/run_faulted/run_discarded`, at most once per `run_id`; `run_faulted.detail` carries `phase ∈ {INIT, RUN}` and `run_id = 0` means allocation failed | Core |
| TR-run-flow-007 | run-flow.md | 餐局流程 | Read-only run snapshot for HUD incl. `run_state`, `frozen`, `running_net`, `shortfall`, `action_seq` | UI |
| TR-time-and-turns-001 | time-and-turns.md | 时间与回合 | `initialize(turn_budget)` once per run; `INVALID_TURN_BUDGET` on bad input, no clamping | Core |
| TR-time-and-turns-002 | time-and-turns.md | 时间与回合 | `spend_turn()` cost exactly 1; `can_spend_turn()` pure; no overdraft, no refund | Core |
| TR-time-and-turns-003 | time-and-turns.md | 时间与回合 | `is_exhausted()` reports only; end-reason ownership stays in run-flow | Core |
| TR-time-and-turns-004 | time-and-turns.md | 时间与回合 | Budget frozen for the run; changes apply next `initialize` | Core |
| TR-stomach-capacity-001 | stomach-capacity.md | 胃容量 | `initialize(upgrade_snapshot)` exactly once per run; `end_run(reason)` is the authoritative run boundary on every termination path | Core |
| TR-stomach-capacity-002 | stomach-capacity.md | 胃容量 | `capacity_total = C0 + capacity_bonus`, range `[1,999]`; invalid/missing snapshot fails, no default 0 | Core |
| TR-stomach-capacity-003 | stomach-capacity.md | 胃容量 | `can_fit(e)` / `consume(e)` use integer `effective_satiety`, inclusive boundary, monotonic used capacity | Core |
| TR-stomach-capacity-004 | stomach-capacity.md | 胃容量 | No “full” end state; capacity only gates individual dishes | Core |
| TR-stomach-capacity-005 | stomach-capacity.md | 胃容量 | Read-only accessors only; no signals, no second counter | Core |
| TR-dish-values-001 | dish-values.md | 菜品数值 | `unit_value(d) = price/satiety` is the single cross-dish comparison scale; no downstream recomputation | Data |
| TR-dish-values-002 | dish-values.md | 菜品数值 | Static 12-dish catalog; balance audits (`menu_healthy`, `opt_value`) run in design/CI only | Data |
| TR-dish-values-003 | dish-values.md | 菜品数值 | No runtime aggregation, optimum solving, or `opt_value` exposure | Core |
| TR-dish-values-004 | dish-values.md | 菜品数值 | Deterministic `uv_cmp` ordering (unit value desc, id asc) | Data |
| TR-dish-values-005 | dish-values.md | 菜品数值 | Result-style `QueryError {caller, dish_id, reason}`; no zero/placeholder returns | Data |
| TR-cloying-and-taste-001 | cloying-and-taste.md | 腻度与口味 | Per-`cloying_group` integer counters reset per run; explicit canonical group order array | Core |
| TR-cloying-and-taste-002 | cloying-and-taste.md | 腻度与口味 | `effective_satiety = satiety + max(0, min(count, cloy_cap) − floor(resistance))`, computed before increment | Core |
| TR-cloying-and-taste-003 | cloying-and-taste.md | 腻度与口味 | `酸`/`冰` cleanse decrements the highest other group once; ties by canonical order | Core |
| TR-cloying-and-taste-004 | cloying-and-taste.md | 腻度与口味 | `initialize(cloying_resistance)` before `run_started`; no persistence in MVP | Core |
| TR-cloying-and-taste-005 | cloying-and-taste.md | 腻度与口味 | Expose `extra_satiety`, `cleanse_target`, group counts to HUD before selection | UI |
| TR-menu-waves-001 | menu-waves.md | 菜单波次 | Deterministic seeded partition; same `(station_id, seed)` identical across platforms | Core |
| TR-menu-waves-002 | menu-waves.md | 菜单波次 | `visible_dishes` / `preview_dishes`; preview capped at `wave_size−1` | Core |
| TR-menu-waves-003 | menu-waves.md | 菜单波次 | `advance_cost` 0 for empty/final wave, 1 otherwise; `end_pickup` free | Core |
| TR-menu-waves-004 | menu-waves.md | 菜单波次 | `wave_reachable_opt` wave-reset DP: 924 carry states, per-dish turn cost, cloying-ignoring upper bound, ≤1e4 nodes, `DEGRADED` fallback | Core |
| TR-menu-waves-005 | menu-waves.md | 菜单波次 | `wave_healthy` lower bound `0.9 × opt_value`; up to 20 rerolls then best + warning | Core |
| TR-menu-waves-006 | menu-waves.md | 菜单波次 | `pool_override` accepted by initialize and health check | Core |
| TR-settlement-001 | settlement-and-entry-fee.md | 结算与入场费 | `settle(RunLog)` once per `run_id`; duplicate returns `ALREADY_SETTLED`; the transaction additionally rejects a repeated application by `run_id + digest` | Core |
| TR-settlement-002 | settlement-and-entry-fee.md | 结算与入场费 | `total_value` int64 sum; `net_profit = total_value − entry_fee`; `broke_even` derived; no clamping | Economy |
| TR-settlement-003 | settlement-and-entry-fee.md | 结算与入场费 | `RunResult` schema: rows in eaten order, `return_ratio` null when fee 0, `upgrade_snapshot` passthrough | Data |
| TR-settlement-004 | settlement-and-entry-fee.md | 结算与入场费 | Pure function: no I/O, clock, or RNG; checked int64 overflow returns error | Core |
| TR-settlement-005 | settlement-and-entry-fee.md | 结算与入场费 | Shared `running_net(k)` / `shortfall(k)` definitions for HUD; single summation source | Core |
| TR-meta-upgrades-001 | meta-upgrades.md | 局外升级 | Wallet is int64 with exactly three flows (run net, purchase, unlock spend); negative allowed; checked overflow | Economy |
| TR-meta-upgrades-002 | meta-upgrades.md | 局外升级 | Exponential `upgrade_price` from `cost`/`cost_growth`; linear effect per level; hard `max_level` | Economy |
| TR-meta-upgrades-003 | meta-upgrades.md | 局外升级 | `resolve_upgrade_snapshot()` returns parsed bonuses plus frozen `levels` copy | Core |
| TR-meta-upgrades-004 | meta-upgrades.md | 局外升级 | Purchase transaction: validate → memory apply → `save.commit` → success; rollback on write failure | Save/Load |
| TR-meta-upgrades-005 | meta-upgrades.md | 局外升级 | `prepare_run_result` computes progression only; orchestrator commits once; `WALLET_OVERFLOW` omits progression while records still commit; write failure returns `APPLIED_UNSAVED` and marks both namespaces dirty | Save/Load |
| TR-meta-upgrades-006 | meta-upgrades.md | 局外升级 | `spend(amount, reason, extra)` generic wallet transaction; `extra` merged into the same commit | Economy |
| TR-meta-upgrades-007 | meta-upgrades.md | 局外升级 | `menu_vision_useful_max = ceil((wave_size−1)/effect_value)`; no no-effect level sold | Economy |
| TR-unlocks-001 | unlocks-and-stations.md | 解锁与档口 | Availability derived: `unlock_cost = 0` or id in `unlocks.stations`; dishes follow the station | Core |
| TR-unlocks-002 | unlocks-and-stations.md | 解锁与档口 | Normalize/backfill free stations and dishes; empty set → `NoPlayableStation`; never delete ids | Core |
| TR-unlocks-003 | unlocks-and-stations.md | 解锁与档口 | Unlock transaction via `meta-upgrades.spend` with `{unlocks}` merged into the same commit | Save/Load |
| TR-unlocks-004 | unlocks-and-stations.md | 解锁与档口 | Fixed validation order incl. `NOT_IN_MENU`, `BUSY`, `INSUFFICIENT_FUNDS` | Core |
| TR-unlocks-005 | unlocks-and-stations.md | 解锁与档口 | Catalog/progress/effective-unlock read APIs; id-sorted output | Data |
| TR-records-001 | records-and-milestones.md | 纪录与里程碑 | `records` namespace schema: per-station best, career aggregates, earned milestones, `completed_at` | Save/Load |
| TR-records-002 | records-and-milestones.md | 纪录与里程碑 | `prepare_run(RunResult)` once per `run_id`; consistency assertions; any error aborts the whole settlement transaction (no other namespace commits); `ALREADY_APPLIED` triggers no commit | Core |
| TR-records-003 | records-and-milestones.md | 纪录与里程碑 | Per-station ranking by `net_profit`; strict improvement only; ties keep the old record | Core |
| TR-records-004 | records-and-milestones.md | 纪录与里程碑 | Separate read-only `MilestoneData` loader with four metrics; monotonic lock-in | Data |
| TR-records-005 | records-and-milestones.md | 纪录与里程碑 | UTC timestamps display-only; no logic reads the clock | Data |
| TR-records-006 | records-and-milestones.md | 纪录与里程碑 | Signals `record_broken`, `milestone_earned`, `career_completed` | Core |
| TR-in-game-hud-001 | in-game-hud.md | 局内 HUD | Read-only projection; one immutable `HudFrame` per event; no polling between frames | UI |
| TR-in-game-hud-002 | in-game-hud.md | 局内 HUD | 1280×720 `canvas_items`/expand layout: top bar, card grid, right rail, detail row | UI |
| TR-in-game-hud-003 | in-game-hud.md | 局内 HUD | Card composed from dish/cloying/capacity/wave providers; no sorting, recommendation, or `unit_value` | UI |
| TR-in-game-hud-004 | in-game-hud.md | 局内 HUD | Focus resolution and `REJECTED(reason)` feedback; non-eatable cards remain focusable | UI |
| TR-in-game-hud-005 | in-game-hud.md | 局内 HUD | Fixed number formats; color always backed by text/symbol | UI |
| TR-in-game-hud-006 | in-game-hud.md | 局内 HUD | `get_anchor_rect(anchor_id)` and three protected rects for onboarding | UI |
| TR-ui-flow-001 | ui-flow.md | 界面流程 | Fixed screen inventory; UI stack max depth 2, modal stack max depth 2 | UI |
| TR-ui-flow-002 | ui-flow.md | 界面流程 | Deterministic `back_target` priority; forced `Blocked` modal ignores `ui_cancel` | Input |
| TR-ui-flow-003 | ui-flow.md | 界面流程 | Focus order per screen, `initial_focus`, `focus_restore` | UI |
| TR-ui-flow-004 | ui-flow.md | 界面流程 | Receipt orchestration: the transaction runs inside `UIRoot.reset_for_state(Menu, RunResult)` before the receipt mounts; `prepare_run` -> `prepare_run_result` -> one `save.commit`; receipt opens same frame; a fatal records error means zero commits plus `settlement_aborted` | UI |
| TR-ui-flow-005 | ui-flow.md | 界面流程 | Lock-first `Blocked` exits and `retry_load` path; load-report branching | UI |
| TR-ui-flow-006 | ui-flow.md | 界面流程 | Machine-level settings model: volumes, mute, text scale 100/125/150, reduce motion, HUD opacity | UI |
| TR-ui-flow-007 | ui-flow.md | 界面流程 | Semantic events for audio: `ui_button_activated`, `number_roll_*`, `receipt_closed` | Audio |
| TR-audio-feedback-001 | audio-feedback.md | 音效反馈 | Event→cue mapping; fixed cue catalog published as a typed Resource validated by DataConfig; same-frame dispatch | Audio |
| TR-audio-feedback-002 | audio-feedback.md | 音效反馈 | Voice pool max 10 with per-priority caps, preemption rules, 50 ms guard; per-priority cap is a hard ceiling after any steal | Audio |
| TR-audio-feedback-003 | audio-feedback.md | 音效反馈 | Deterministic variant round-robin; per-cue cooldowns; per-`run_id` dedupe; MVP pitch jitter disabled, future jitter only from SeedHash | Audio |
| TR-audio-feedback-004 | audio-feedback.md | 音效反馈 | `cue_volume_db = clamp(base + bus + master + sfx, -80, 0)` | Audio |
| TR-audio-feedback-005 | audio-feedback.md | 音效反馈 | Pause silences and clears pending outcome; mute/missing-device are drop semantics; `sfx.flow.*` runs out-of-pool so the pause cue starts on the freeze frame | Audio |
| TR-audio-feedback-006 | audio-feedback.md | 音效反馈 | Cue request log (ring buffer 10k) with closed reason vocabulary; `priority` stays in the vocabulary but is not emitted in v1 | Audio |
| TR-audio-feedback-007 | audio-feedback.md | 音效反馈 | Content defects (invalid cue catalog or event map) fail boot through the data gate; playback failures degrade silently at runtime | Audio |
| TR-first-run-onboarding-001 | first-run-onboarding.md | 首局引导 | Runs only when `RunConfig.is_first_run`; preset `pool_override` six dishes and `entry_fee_override = 40` | Core |
| TR-first-run-onboarding-002 | first-run-onboarding.md | 首局引导 | Fixed B0–B4 beat sequence; one beat per input; modal beats use `Blocked` allow-list | Core |
| TR-first-run-onboarding-003 | first-run-onboarding.md | 首局引导 | Anchors from HUD `get_anchor_rect`; bubbles never cover the three protected rects | UI |
| TR-first-run-onboarding-004 | first-run-onboarding.md | 首局引导 | `FirstRunExplanation` payload for the receipt; no persistence; `onboarding_finished` once per run | UI |
| TR-first-run-onboarding-005 | first-run-onboarding.md | 首局引导 | Degradation order for missing preset/anchors/unit values; never blocks the run | Core |

## Appendix B — Traceability Summary

| Area | Requirements | Covered (Proposed) | Partial | Gaps |
|---|---|---|---|---|
| Foundation (data, input, scene, save) | 24 | 23 | 1 | 0 |
| Core (run, turns, capacity, dish values) | 21 | 4 | 2 | 15 |
| Feature (cloying, waves, settlement, meta, unlocks, records) | 34 | 12 | 1 | 21 |
| Presentation (HUD, UI, audio, onboarding) | 25 | 17 | 0 | 8 |
| **Total** | **104** | **56** | **4** | **44** |

The 2026-11-04 `/architecture-review` produced
`docs/architecture/requirements-traceability.md`, which carries the full
per-requirement matrix; every provisional ID now has its permanent row.
