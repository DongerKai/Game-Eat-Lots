## Architecture Review Report

Date: 2026-11-04
Mode: full
Workflow: standard
Engine: Godot 4.7（pin 4.7.x；`Installed at pin time = NOT DETERMINED` —— 本机无编辑器，本次评审未执行任何引擎实测，也不对任何运行时语义作「已验证」结论）
GDDs Reviewed: 21 个文件（`design/gdd/*.md` 全部：18 份系统 GDD + `game-concept.md` + `systems-index.md` + `gdd-cross-review-2026-11-04.md`；需求正文以 `tr-registry.yaml` 的 104 条为源，GDD 按 ADR 引用做定点复核）
ADRs Reviewed: 9（ADR-0001 … ADR-0009，全部 `Proposed`，`accepted = 0`）

Reviewed-Content-Hash: docs/architecture/adr-0001-content-data-format.md 690ba54214abca21824e90df1cef21a07e685eba
Reviewed-Content-Hash: docs/architecture/adr-0002-persistence-format.md 193c4df5965a70590031579d39a5f670279b196e
Reviewed-Content-Hash: docs/architecture/adr-0003-scene-screen-architecture.md be91deb2243b3759483467889e20dd6cede0393b
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md e41bd1c9451e477eca9ebd8bd54a411a85cc6ef7
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md a42c1fd5a9bf120536ce1e482fd0fa5767f8adc8
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md f8bbbce1efbe2312fb8283ea8c73d61591e92f7a
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 7cf5114d67eb4056b21faf41b77b6473c602fcb8
Reviewed-Content-Hash: docs/architecture/adr-0008-settings-localization.md 4c5e3de653467dc33623305163fc1844e5c93b3e
Reviewed-Content-Hash: docs/architecture/adr-0009-audio-playback-pause.md 2d9c23341fd18c18b131cf3c9fd3ece9f8b29427
Reviewed-Content-Hash: design/gdd/audio-feedback.md 1a81a3856a1c5b6254f03485f04caf65afdb431f
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md e6aa08b28eb7739e9a5c342f358f9f55b5292bde
Reviewed-Content-Hash: design/gdd/data-config.md 89626e0d680a97b2887e2d2e76f07a4e3126ec2a
Reviewed-Content-Hash: design/gdd/dish-values.md 8b48f38c7e01eb84a462bf847a8bbac3a7101997
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md db09042623fa8333cd6410aa8e97249c1f45e9cf
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/gdd-cross-review-2026-11-04.md c6efa8393fa2f8c10b80d5964b75aa9793f7d042
Reviewed-Content-Hash: design/gdd/in-game-hud.md f81ce0c2348338cd8bd3858ed659e2b13c882a06
Reviewed-Content-Hash: design/gdd/input.md 27c9d885eb2ba1f8d4b3df72ff6640fed0a5dd36
Reviewed-Content-Hash: design/gdd/menu-waves.md effad23c5f5bd13b0c6a657ab32c2680a078fc6b
Reviewed-Content-Hash: design/gdd/meta-upgrades.md ba07eaa56adfaa489b0ba0fdac9065b492f68c64
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 7cb99488fee8beeba202523848adecd3c06c2c7c
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/save-system.md 1bd3148294c81eb9a186f218cbb390b30b0e08ad
Reviewed-Content-Hash: design/gdd/scene-flow.md 07753accfc002be7257cbcdd17323c73c2b6845a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 9bf45c1ead7e161599186ca5724fa460b044e314
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/gdd/systems-index.md e0174253c2b36603edfef7b4d00d005380530feb
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/ui-flow.md cf471d2d45476ff8e1db0a5fa4a7fae9a4e146ea
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 26c30916e1446a1cd30ca9d44f71982adecb104e
Reviewed-Content-Hash: docs/architecture/tr-registry.yaml b5633060b13f7d4e94043b1bbd4d0dfa68b8fc4d
Reviewed-Content-Hash: docs/architecture/architecture.md 1b3c1a1e83f4e21fc6b29fb075292c502be2bf6d
Reviewed-Content-Hash: docs/engine-reference/godot/VERSION.md a4d46ef0a92ad9acc5000c5398b4cb843919993e
Reviewed-Content-Hash: docs/engine-reference/godot/breaking-changes.md 4bc55e16796664ff88c7582c9f08e3530f7c94d4
Reviewed-Content-Hash: docs/engine-reference/godot/current-best-practices.md a45cd2c39a8ec53eaa3e924f39f93594e62085fd
Reviewed-Content-Hash: docs/engine-reference/godot/deprecated-apis.md 6a1353983e1db0d7bc0cc41a568b039495b5413c
Reviewed-Content-Hash: docs/engine-reference/godot/modules/animation.md ee51f50a46f8c63b771f62fe438947d4d84c59e3
Reviewed-Content-Hash: docs/engine-reference/godot/modules/audio.md c72af247505908a04f84a1dd3e765939efe700bd
Reviewed-Content-Hash: docs/engine-reference/godot/modules/input.md 3406b1b813858898696e510ea0accfcc7df98af8
Reviewed-Content-Hash: docs/engine-reference/godot/modules/navigation.md 7ae18ffbd2c3230cb19d610373225bfba187a472
Reviewed-Content-Hash: docs/engine-reference/godot/modules/networking.md 6606880462a2241014f77feb9610c5611e659ec6
Reviewed-Content-Hash: docs/engine-reference/godot/modules/physics.md 3b6ac8947f3f3e8c9ac9e6e0ff6cac06fdb8d805
Reviewed-Content-Hash: docs/engine-reference/godot/modules/rendering.md 2a303185bb57eed31dd1151a036784820747b211
Reviewed-Content-Hash: docs/engine-reference/godot/modules/ui.md 30ad166ab8c4e92c15aa37ed3464483b92b7845b
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: .claude/docs/technical-preferences.md 7b872eb635445e60e192d7dda828cbd00cfdedc4
Reviewed-Content-Hash: docs/consistency-failures.md — not stamped, no such file
Reviewed-Content-Hash: .claude/skills/architecture-review/SKILL.md 1a9d28aadbcbd26b5fc6b8ed1c64d73ae82b7b73
Reviewed-Content-Hash: .claude/skills/architecture-review/references/1-load.md 9a65759ebdbb2cb1f9744775bfed18cb25e76347
Reviewed-Content-Hash: .claude/skills/architecture-review/references/2-extract-requirements.md d08ea0b0e9ab9760b0d8bfe0cd9115f0bc40f8e2
Reviewed-Content-Hash: .claude/skills/architecture-review/references/3-traceability.md bb7b6b65c5e5e5b8869fa692413aa831615dede7
Reviewed-Content-Hash: .claude/skills/architecture-review/references/4-conflicts.md 0d30f8173b23248a32523a365b606dd1ce9642ab
Reviewed-Content-Hash: .claude/skills/architecture-review/references/5-engine.md 318ecbc1361d430badaf5dd8ce4cfc6ab0dff0ca
Reviewed-Content-Hash: .claude/skills/architecture-review/references/6-document-coverage.md dcce78482ec2ce60048989b6f9d1f797dd0a9a0e
Reviewed-Content-Hash: .claude/skills/architecture-review/references/7-report.md a007fd66bdac87812dc272e4b14cd231174e979d
Reviewed-Content-Hash: .claude/skills/architecture-review/references/8-write.md bf6d8946fc7b9e19eda7f65503a5c8f593a0d2b1
Reviewed-Content-Hash: .claude/skills/architecture-review/references/9-handoff.md fd5f681f4c3517c6a2e388c6d78075d82dded22e

Freshness: `RECEIPT: NONE` —— `docs/architecture/` 下此前没有 `architecture-review-*.md`，本次是首次全量评审，无 delta 可用。`docs/consistency-failures.md` 不存在（未创建、未追加）。

---

### Traceability Summary

Total requirements: 104（`tr-registry.yaml` 104 条 `status: active`；无 deprecated / superseded）
✅ Covered (Accepted ADR): 0
🟡 Covered (Proposed ADR): 56（53.8%）
⚠️ Partial: 4
❌ Gaps: 44（42.3%）
❓ Not assessed: 0（9 份 ADR 都可读且都有 `## Status`）

| Layer | Total | 🟡 | ⚠️ | ❌ |
|---|---|---|---|---|
| Foundation（data-config, input, scene-flow, save-system） | 24 | 23 | 1 | 0 |
| Core（run-flow, time-and-turns, stomach-capacity, dish-values） | 21 | 4 | 2 | **15** |
| Feature（cloying, settlement, meta-upgrades, menu-waves, unlocks, records） | 34 | 12 | 1 | **21** |
| Presentation（hud, ui-flow, audio-feedback, onboarding） | 25 | 17 | 0 | 8 |
| **Total** | **104** | **56** | **4** | **44** |

Coverage 的来源是每份 ADR 的 `## GDD Requirements Addressed` 表（ADR-0009 用 `## TR Coverage`），不是正文里顺带提到的 TR 编号。逐条矩阵见 `docs/architecture/requirements-traceability.md`。

⚠️ Partial（ADR 自述不完整，不能记作 covered）：

| Requirement | ADR | 缺口 |
|---|---|---|
| `TR-run-flow-006` | ADR-0007 | 只定了 `run_settled` / `run_faulted` / `run_discarded` 的边界；信号表全表仍在 run-flow |
| `TR-run-flow-007` | ADR-0007 | 只定了快照的唯一发布者；字段清单仍在 RunFlow / HUD |
| `TR-settlement-005` | ADR-0007 | 运行期累加器归 `RunFlow`；纯函数公式仍未定 |
| `TR-save-system-006` | ADR-0002 / 0004 / 0007 | 三份 ADR 合起来覆盖「结算来源的一次事务 + 保留 dirty」；原子写与错误词表在 ADR-0002 内部章节，未落在该 TR 的验收面上 |

**与 `architecture.md` 的差异（可完全解释，需编排者修文档）**：architecture.md:366 与 ADR Audit 表写「59 covered / 45 gaps」，原因是它把 `TR-run-flow-006`、`TR-run-flow-007`、`TR-save-system-006` 记作 covered（ADR-0007 自己标 partial）、把 `TR-settlement-005` 记作 gap，并把 `TR-dish-values-003`（ADR-0006 §4 正文覆盖，可算）与 `TR-run-flow-003`（ADR-0006 只在正文的冲突说明里提到，**不**构成覆盖）一并归给 ADR-0006。本次矩阵按 ADR 自述计：**56 / 4 / 44**。

### Coverage Gaps (no ADR exists)

| Requirement ID | GDD | System | Requirement | Suggested ADR | Domain | Engine Risk |
|---|---|---|---|---|---|---|
| `TR-run-flow-003` | run-flow.md | run-flow | `submit_action(eat/skip_wave/end_pickup, dish_id)` with fixed validation order and atomic… | 阶段化的局内动作提交与结束判定 | run-flow | LOW |
| `TR-run-flow-004` | run-flow.md | run-flow | Only two end reasons (`TIME_EXHAUSTED`, `NO_EATABLE_DISH`); fixed priority; state frozen… | 阶段化的局内动作提交与结束判定 | run-flow | LOW |
| `TR-time-and-turns-001` | time-and-turns.md | time-and-turns | `initialize(turn_budget)` once per run; `INVALID_TURN_BUDGET` on bad input, no clamping | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-time-and-turns-002` | time-and-turns.md | time-and-turns | `spend_turn()` cost exactly 1; `can_spend_turn()` pure; no overdraft, no refund | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-time-and-turns-003` | time-and-turns.md | time-and-turns | `is_exhausted()` reports only; end-reason ownership stays in run-flow | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-time-and-turns-004` | time-and-turns.md | time-and-turns | Budget frozen for the run; changes apply next `initialize` | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-stomach-capacity-001` | stomach-capacity.md | stomach-capacity | `initialize(upgrade_snapshot)` exactly once per run; `end_run(reason)` is the authoritati… | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-002` | stomach-capacity.md | stomach-capacity | `capacity_total = C0 + capacity_bonus`, range `[1,999]`; invalid/missing snapshot fails,… | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-003` | stomach-capacity.md | stomach-capacity | `can_fit(e)` / `consume(e)` use integer `effective_satiety`, inclusive boundary, monotoni… | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-004` | stomach-capacity.md | stomach-capacity | No “full” end state; capacity only gates individual dishes | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-005` | stomach-capacity.md | stomach-capacity | Read-only accessors only; no signals, no second counter | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-dish-values-001` | dish-values.md | dish-values | `unit_value(d) = price/satiety` is the single cross-dish comparison scale; no downstream… | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-dish-values-002` | dish-values.md | dish-values | Static 12-dish catalog; balance audits (`menu_healthy`, `opt_value`) run in design/CI only | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-dish-values-004` | dish-values.md | dish-values | Deterministic `uv_cmp` ordering (unit value desc, id asc) | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-dish-values-005` | dish-values.md | dish-values | Result-style `QueryError {caller, dish_id, reason}`; no zero/placeholder returns | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-cloying-and-taste-001` | cloying-and-taste.md | cloying-and-taste | Per-`cloying_group` integer counters reset per run; explicit canonical group order array | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-002` | cloying-and-taste.md | cloying-and-taste | `effective_satiety = satiety + max(0, min(count, cloy_cap) − floor(resistance))`, compute… | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-003` | cloying-and-taste.md | cloying-and-taste | `酸`/`冰` cleanse decrements the highest other group once; ties by canonical order | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-004` | cloying-and-taste.md | cloying-and-taste | `initialize(cloying_resistance)` before `run_started`; no persistence in MVP | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-005` | cloying-and-taste.md | cloying-and-taste | Expose `extra_satiety`, `cleanse_target`, group counts to HUD before selection | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-settlement-002` | settlement-and-entry-fee.md | settlement | `total_value` int64 sum; `net_profit = total_value − entry_fee`; `broke_even` derived; no… | 结算纯函数：单位价值、入场费与回本判定 | settlement | LOW |
| `TR-settlement-003` | settlement-and-entry-fee.md | settlement | `RunResult` schema: rows in eaten order, `return_ratio` null when fee 0, `upgrade_snapsho… | 结算纯函数：单位价值、入场费与回本判定 | settlement | LOW |
| `TR-meta-upgrades-001` | meta-upgrades.md | meta-upgrades | Wallet is int64 with exactly three flows (run net, purchase, unlock spend); negative allo… | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-002` | meta-upgrades.md | meta-upgrades | Exponential `upgrade_price` from `cost`/`cost_growth`; linear effect per level; hard `max… | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-003` | meta-upgrades.md | meta-upgrades | `resolve_upgrade_snapshot()` returns parsed bonuses plus frozen `levels` copy | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-004` | meta-upgrades.md | meta-upgrades | Purchase transaction: validate → memory apply → `save.commit` → success; rollback on writ… | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-006` | meta-upgrades.md | meta-upgrades | `spend(amount, reason, extra)` generic wallet transaction; `extra` merged into the same c… | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-007` | meta-upgrades.md | meta-upgrades | `menu_vision_useful_max = ceil((wave_size−1)/effect_value)`; no no-effect level sold | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-unlocks-001` | unlocks-and-stations.md | unlocks | Availability derived: `unlock_cost = 0` or id in `unlocks.stations`; dishes follow the st… | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-unlocks-002` | unlocks-and-stations.md | unlocks | Normalize/backfill free stations and dishes; empty set → `NoPlayableStation`; never delet… | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-unlocks-003` | unlocks-and-stations.md | unlocks | Unlock transaction via `meta-upgrades.spend` with `{unlocks}` merged into the same commit | 解锁与档口购买事务（wallet + unlocks 同一 commit）（ADR-0007 已显式移交给未来的 Unlocks ADR） | unlocks | MEDIUM |
| `TR-unlocks-004` | unlocks-and-stations.md | unlocks | Fixed validation order incl. `NOT_IN_MENU`, `BUSY`, `INSUFFICIENT_FUNDS` | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-unlocks-005` | unlocks-and-stations.md | unlocks | Catalog/progress/effective-unlock read APIs; id-sorted output | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-records-003` | records-and-milestones.md | records | Per-station ranking by `net_profit`; strict improvement only; ties keep the old record | 纪录、里程碑判定与生涯完成 | records | LOW |
| `TR-records-005` | records-and-milestones.md | records | UTC timestamps display-only; no logic reads the clock | 纪录、里程碑判定与生涯完成 | records | LOW |
| `TR-records-006` | records-and-milestones.md | records | Signals `record_broken`, `milestone_earned`, `career_completed` | 纪录、里程碑判定与生涯完成 | records | LOW |
| `TR-in-game-hud-001` | in-game-hud.md | in-game-hud | Read-only projection; one immutable `HudFrame` per event; no polling between frames | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-in-game-hud-003` | in-game-hud.md | in-game-hud | Card composed from dish/cloying/capacity/wave providers; no sorting, recommendation, or `… | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-in-game-hud-005` | in-game-hud.md | in-game-hud | Fixed number formats; color always backed by text/symbol | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-in-game-hud-006` | in-game-hud.md | in-game-hud | `get_anchor_rect(anchor_id)` and three protected rects for onboarding | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-first-run-onboarding-001` | first-run-onboarding.md | first-run-onboarding | Runs only when `RunConfig.is_first_run`; preset `pool_override` six dishes and `entry_fee… | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |
| `TR-first-run-onboarding-003` | first-run-onboarding.md | first-run-onboarding | Anchors from HUD `get_anchor_rect`; bubbles never cover the three protected rects | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |
| `TR-first-run-onboarding-004` | first-run-onboarding.md | first-run-onboarding | `FirstRunExplanation` payload for the receipt; no persistence; `onboarding_finished` once… | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |
| `TR-first-run-onboarding-005` | first-run-onboarding.md | first-run-onboarding | Degradation order for missing preset/anchors/unit values; never blocks the run | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |

### Cross-ADR Conflicts

## Conflict: ADR-0003 vs ADR-0005（ADR-0004 同环）
Type: Dependency
ADR-0003 claims: `Depends On: ADR-0005`，因为 `ModalStackModel` 每次 push/pop 都调用 `input.push_context(&"Modal")` / `pop_context()`（D4）
ADR-0005 claims: `Depends On: ADR-0003, ADR-0004`，因为 `InputRouter` 是 `AppRoot` 的直接子节点（ADR-0004 D2）且 `ActionSink` 只投递给 `UIRoot.handle_action`（D4）
ADR-0004 claims: `Depends On: ADR-0001, ADR-0002, ADR-0003, ADR-0005`，组合根同时构造三者
Impact: `adr-dep-graph.sh` 输出三行 CYCLE（`0003 → 0005`、`0004 → 0003 → 0005`、`0005 → 0003 → 0004`），`ORDER:` 只剩 `adr-0001, adr-0002`，`BLOCKED_BY_CYCLE: adr-0006, adr-0007, adr-0008, adr-0009`。没有拓扑序就没有实现顺序，也无法回答「先做输入还是先做界面」。
Resolution options:
  1. （推荐）ADR-0005 的 `Depends On` 改为 `None`，正文声明「与 ADR-0003 / ADR-0004 契约级共同设计：消费 `UIRoot.handle_action`，由 `AppRoot` 构造」。输入层只依赖接口名，不依赖实现，得 DAG 顺序 `0001 → 0002 → 0005 → 0003 → 0004 → 0006 → 0007 → 0008 → 0009`
  2. 删 `ADR-0003 → ADR-0005` 一条边：`0004 → 0005` 与 `0005 → 0004` 仍成环，无效
  3. 把 ADR-0003 D4 的模态上下文规则并入 ADR-0005，ADR-0003 不再引用 push/pop

## Conflict: ADR-0001 vs ADR-0009
Type: Integration
ADR-0001 claims: `DataConfig` 有五类记录、五个显式清单（dish / upgrade / station / balance / milestone），`Validating` 第 1 阶段校验「`BalanceData` 存在」
ADR-0009 claims: `res://data/audio/cue_catalog.tres` 是 `DataConfig` 的**第六个**清单，与其余五个同一次 `Loading → Validating → Ready/Failed`、同一次全有或全无发布（D8）
Impact: 契约本身不矛盾（ADR-0009 明说「本 ADR 只是新增第六个清单」），但只读 ADR-0001 实现会漏建第六个清单，音频目录与 `CueMap` 的双向校验不进启动门；ADR-0001 的清单表与校验阶段描述已不完整。
Resolution options:
  1. （推荐）ADR-0001 §2 清单表补第六行并注明「由 ADR-0009 引入」，`Validating` 阶段说明同步
  2. ADR-0009 自带验证器、不进 `DataConfig`：与启动门和「全有或全无」冲突，不推荐

**已核对、判定不构成冲突**（避免误报）：
- 暂停模型：ADR-0003 D2.4 的 `ALWAYS` 清单是「`UIRoot` + 四个宿主 + `FadeLayer` + `InputRouter`」，`HudHost` 不在其中；ADR-0004 D10 把 `HudHost` 定为 `INHERIT`。两者一致。
- 随机源：ADR-0004 把 `SessionRng` 定为唯一 RNG，ADR-0006 的确定性 RNG 明确「绝不触碰 `SessionRng`」。不冲突。
- 收据与提交流程：ADR-0003 D7 的「挂收据 → 卸 HUD」与 ADR-0007 D3 的「`reset_for_state` 内提交、挂载之前」顺序一致。
- 纪录信号时机：ADR-0007「GDD 矛盾」第 2 条指出的 ADR-0004 D9 措辞已在 ADR-0004:236 改为「结算事务已应用后（不是写盘成功后；见 ADR-0007 D5）」，已消解。
- `state_changed` 单订户 / 六步扇出、`InputRouter` 调用方所有权、设置文件所有权（`ui-flow` 写、音频只读）三处交叉引用均一致。

### ADR Dependency Order

`adr-dep-graph.sh`：`ADRS: 9`、21 条边、`NO_DEPS_SECTION:` 无、`UNFILLED_DEPS:` 无（9 份 ADR 的 `Depends On` 行都填了，ADR-0001 / ADR-0002 为 `None`——这是「图是干净的」而不是「没声明」）。

### Recommended ADR Implementation Order (topologically sorted)

前提：按上节「冲突 1」方案 1 解除环；否则 `ORDER:` 只有 `adr-0001, adr-0002`，其余全部阻塞。

Foundation (no dependencies):
  1. ADR-0001：内容数据格式、校验与启动门
  2. ADR-0002：持久化格式、原子写入与跨平台写锁
Depends on Foundation:
  3. ADR-0005：输入上下文栈语义（建议改为无依赖）
  4. ADR-0003：场景与屏幕架构（requires ADR-0005）
  5. ADR-0004：服务组合与信号机制（requires ADR-0001, 0002, 0003, 0005）
Feature / Presentation:
  6. ADR-0006：确定性随机、种子派生与健康校验预算（requires ADR-0001, 0004）
  7. ADR-0007：局内状态所有权与结算事务边界（requires ADR-0001–0004）
  8. ADR-0008：机器级设置与本地化管线（requires ADR-0001, 0002, 0003）
  9. ADR-0009：音频播放、池化与暂停语义（requires ADR-0001, 0003, 0004, 0006, 0008）

⚠️ 全部 21 条边都指向仍是 `Proposed` 的 ADR：按第三节的规则，每一条「A 依赖 B 而 B 仍 Proposed」都应报一次；合并成一条——**在依赖被 Accept 之前，链上的任何 ADR 都不能安全实现**，而 `accepted = 0` 使这条对整个图成立。

### GDD Revision Flags

本次没有可用的引擎实测（`Installed at pin time = NOT DETERMINED`），所以下表不是「GDD 与引擎实测冲突」，而是「ADR 显式偏离 GDD」与「ADR 记录的 GDD 内部矛盾」。`design/gdd/systems-index.md` 的 Status 已是 `Needs Revision`，方向一致；该文件不在本次写入范围内，未修改。

| # | GDD | 位置 | GDD 现状 | 建议改为 | 来源 |
|---|---|---|---|---|---|
| 1 | save-system.md | 规则 21 | 写锁存活用 `OS.is_process_running`（只能看到子进程） | 锁目录 + `owner.json` + 心跳，`PROCESS_MODE_ALWAYS` 驱动者，90 s 超时 / 24 h 上限 | ADR-0002 §3 |
| 2 | dish-values.md | 接口所有权段落 | 未说明 `opt_value` 的唯一例外 | 补一句：`MenuWaves` 健康门是唯一例外，结果不外流、不进存档/HUD/结算 | ADR-0006 §4 |
| 3 | records-and-milestones.md | :141、:470（OQ1） | 「编排者身份尚未指定」 | 标记已解决：算法在 `SettlementTransaction`，唯一调用者是 `UIRoot` | ADR-0007 矛盾 1 |
| 4 | run-flow.md | 规则 13/16（:44、:102、:133） | `Settling → Handoff` 即广播 `run_settled(RunResult)`，与「Handoff 中放弃不交付」冲突 | 明确 `run_settled` 是**预通知**，`Run → Menu` 才是交付；写盘只挂在交付上 | ADR-0007 矛盾 3 |
| 5 | scene-flow.md | 规则 6 / 转移表（:32、:61） | `init_failed` = 「进入失败」 | payload 说明补 `fault.phase`；`init_failed` 只表示「payload 里没有 `RunResult`」 | ADR-0007 矛盾 4 |
| 6 | run-flow.md | 状态表（:107） | `run_id` 溢出的 `Faulted` 行未标阶段 | 补「`Faulted + INIT`，因 `run_started` 未发出」 | ADR-0007 矛盾 5 |
| 7 | unlocks-and-stations.md | 规则 15（:55） | 「不参与结算事务」 | 限定：MVP/VS 结算事务只含 `{records, progression}`；未来「奖励解锁」并入同一 `commit` | ADR-0007 矛盾 6 |
| 8 | save-system.md | :88、:264 | 「结算引起的钱包、纪录、解锁变化必须合并」与「MVP 菜单解锁是独立事务」并存 | 把「解锁」限定为「由结算本身产生的解锁」 | ADR-0007 矛盾 6 |
| 9 | audio-feedback.md | 状态表（:89） | `Uninitialized` 只建 `Master / SFX / UI` | 补 `Music`（空总线，MVP 无播放器） | ADR-0009 表 1 |
| 10 | audio-feedback.md | 规则 18（:44） | 只在 Critical cue 播放时闪避 | 「结算三 cue 活跃时闪避」，与 :275-278 公式一致 | ADR-0009 表 2 |
| 11 | audio-feedback.md | :284、:286、:414 | 闪避作用于「Music 与 Ambience」 | 删去 `Ambience`，或并入 `SFX` 并注明未来修订 | ADR-0009 表 3 |
| 12 | audio-feedback.md | :66 vs :40、:183 | pause cue 要求同帧起播，准入表第 3 步丢弃 `Frozen` 期间一切请求 | 准入表注明池外转场 cue 不受第 3 步约束（或 pause 音在冻结前起播） | ADR-0009 表 4 |
| 13 | audio-feedback.md | :117、:380 | 「数据配置无直接调用；cue 映射不进 Data Config schema」 | 更新依赖表：经 `AudioCatalogView` 只读消费 `DataConfig` | ADR-0009 表 5 |
| 14 | audio-feedback.md | :21、:43 | 「音频出问题游戏照常运行」 | 区分「内容缺陷（`Boot → Error`）」与「播放失败（静默降级）」 | ADR-0009 表 6 |
| 15 | audio-feedback.md | :142 | `pitch` 日志字段「仅数字 tick 有值」 | 所有 cue 都有 `pitch`，MVP 除数字 tick 外恒 `1.0` | ADR-0009 表 7 |
| 16 | audio-feedback.md | AC 22（:523） | 50 ms 内降 12 dB 的听感验收 | 标 `Deferred`（有音乐后启用），或改为总线写入级验收 | ADR-0009 表 8 |
| 17 | audio-feedback.md | 准入表第 8/9 步 | 未要求抢占后仍满足 `cap[p]` | 补「抢占后 `voices_used(p) < cap[p]`」，使硬顶自洽 | ADR-0009 表 9 |

### Engine Compatibility Issues

- **版本一致性 ✅**：9/9 份 ADR 的 `Engine Compatibility` 都写 Godot 4.7（`4.7.x` / latest patch `4.7.2`），与 `docs/engine-reference/godot/VERSION.md`、`project.yaml` 一致，无第二版本声明。
- **Post-cutoff API 一致性 ✅**：逐条比对 `breaking-changes.md` / `current-best-practices.md`——4.5 `@abstract`（ADR-0001）、4.6 双焦点（ADR-0003/0005/0008）、4.7 新项目 stretch 默认 `canvas_items`+`expand`（ADR-0003）、4.7 `DEVICE_ID_MOUSE`/`DEVICE_ID_KEYBOARD`（ADR-0003/0005）、4.7 `ResourceImporterDynamicFont.hinting` 默认 3（ADR-0008）、4.6 CSV plural/context 列（ADR-0008）——与参考库一致，无「发明 API」。
- **废弃 API 扫描 ✅（0 命中）**：9 份 ADR 中未出现 `TileMap`、`yield(`、`.instance()`、`VisibilityNotifier*`、`get_world()`、`OS.get_ticks_msec`、`width_in_percent`、嵌套资源 `duplicate()`；`device == 0` 仅作为禁令出现（ADR-0005 D8 / 指南 11）。
- **模块参考 ✅**：ADR 引用的 `modules/ui.md`、`modules/input.md`、`modules/audio.md` 均存在（8 份模块文档）。无缺失引用，无需补模块文档。
- **Engine Compatibility 段缺失：无**（9/9 都有）。但 **ADR-0009 缺 `## GDD Requirements Addressed`**（用 `## GDD References` + `## TR Coverage`），`adr-status.sh` 已把它列入 missing 列；标准段扫描会漏计其 8 条 TR，属结构缺口而非内容缺口。
- **引擎验证未执行（关键）**：`Installed at pin time = NOT DETERMINED`，因此 **49 条 `Verification Required` 全部未实测**（0001:5、0002:5、0003:8、0004:5、0005:6、0006:5、0007:4、0008:6、0009:5）。本报告任何 `✅` 只表示「与参考文档不矛盾」，不表示「已在本机 4.7 上验证」。
- **参考库未覆盖的 4 条行为**（ADR 已自行登记，建议优先关闭）：`Viewport.push_input(InputEventAction)` 是否进入 GUI 焦点导航（ADR-0005 验证①）、`focus_neighbor_*` 指向不可聚焦控件时的几何回落（ADR-0003 验证⑥）、`PROCESS_MODE_ALWAYS` 下 `Tween` 是否推进（ADR-0004 验证①）、`SceneTree.paused` 下 `AudioStreamPlayer` 是否继续播放（ADR-0009 验证①）。这 4 条决定 UI 输入与暂停音的实现形状，`/setup-engine` 首次实测应最先跑。

### Architecture Document Coverage

- `docs/architecture/architecture.md` 存在（v1，558 行）→ Phase 6 已运行，本项**不是** NOT ASSESSED。
- **系统覆盖 ✅**：System Layer Map 列出全部 18 个已设计系统，另列 2 个 deferred（在线挑战、同屏合作）；Data Flow 1–6 完整；不变量 6 条与 ADR 一致。
- **API Boundaries ⚠️ 12/18**：只有 DataConfig、SceneFlow、InputRouter、SaveSystem、RunFlow、TimeTurns、StomachCapacity、CloyingAndTaste、MenuWaves、Settlement、MetaUpgrades、Records 有契约。缺 6 个：**Dish Values、Unlocks & Stations**、In-Game HUD、UI Flow、Audio Feedback、First-Run Onboarding。后四个由 ADR-0003/0005/0008/0009 补齐；**Dish Values 与 Unlocks 在全部文档中都没有 API 边界**，这与其 9 条未覆盖需求同源。
- **孤儿架构 ✅ 无**：`SessionRng` / `BootSequence` / `ScreenFactory` 都能追溯到 ADR-0004 / ADR-0003 的基础设施决定，不是无主设计。
- **文档漂移（需编排者修，本次未改）**：
  - :16 与 :437 写「103 requirements / 0 registered, 103 provisional」，:351 与 :555 写 104 —— 实际 104。
  - :366 与 ADR Audit 表写「59 covered / 45 gaps」；本次矩阵 56 / 4 / 44（差异解释见 Traceability Summary）。
  - ADR Audit 表称 ADR-0009 覆盖 `TR-audio-feedback-001..007`，但其 `## TR Coverage` 表只有 001–006 与 `TR-ui-flow-007`；007 由 D8 正文覆盖、未列表。
  - ADR Audit 表把 `TR-dish-values-003`、`TR-run-flow-003` 归给 ADR-0006，二者均只出现在正文（§4 边界说明 / 冲突说明），不在 `## GDD Requirements Addressed` 表内。
  - QQ-10 写 `/review-all-gdds`「still ABSENT」，但 `design/gdd/gdd-cross-review-2026-11-04.md` 已存在。
  - Data Flow 4 的 `Save.commit({progression, records[, unlocks]})` 需按 ADR-0007「GDD 矛盾」第 7 条加「仅未来奖励解锁」限定。

---

### Verdict: FAIL

判定依据（按 `references/7-report.md` 的定义）：FAIL 的条件是「Critical gaps（Foundation / Core 层需求未覆盖）**或** 存在阻塞性跨 ADR 冲突」，本次两条同时成立——

1. **Core 层 15/21 条需求没有 ADR**：时间与回合 4/4、胃容量 5/5、菜品数值 4/5、餐局流程 2/7。Core 层就是「回本」的全部算式（回合预算、容量预算、单位价值），没有任何一条被接受或被提出的 ADR 约束。
2. **阻塞性依赖环**：`ADR-0003 ↔ ADR-0005`（含 ADR-0004）使 `adr-0006..0009` 全部 `BLOCKED_BY_CYCLE`，无法给出实现顺序。

补充说明（不改变判定，但决定后续路线）：
- 即便 44 条 gap 全部补上，`accepted = 0` 仍使 PASS 不可达——`🟡` 的上限是 CONCERNS，PASS 只对 **Accepted** ADR 成立。因此本次 FAIL 的主因是覆盖率，不是 ADR 状态；两条路线必须并行：补 11 份 ADR + `/architecture-decision accept` 现有 9 份。
- 评审是 advisory：FAIL 不阻止用户继续，但它阻止 `/create-epics` 与 `/create-stories` 为时间与回合 / 胃容量 / 菜品数值 / 腻度与口味开故事。

### Blocking Issues (must resolve before PASS)

1. **Core 层 15 条需求无 ADR**：`TR-time-and-turns-001..004`、`TR-stomach-capacity-001..005`、`TR-dish-values-001/002/004/005`、`TR-run-flow-003/004`。在补齐前不得为这四个系统写 story；`tr-registry.yaml` 与 `architecture.md` 的 API Boundaries 也应同时补 Dish Values。
2. **依赖环 ADR-0003 ↔ ADR-0005（+ADR-0004）**：按冲突 1 方案 1 解除，否则 0006–0009 无实现顺序。
3. **9 份 ADR 全部 `Proposed`**：需 `/architecture-decision accept` 逐份接受；在 `accepted = 0` 下 `create-control-manifest`、`create-epics`、`create-stories`、`gate-check` 都会拒绝把规则当已决。
4. **ADR-0009 缺 `## GDD Requirements Addressed`**：补齐（或把 `## TR Coverage` 表并进去），否则 `adr-status.sh` 与后续标准段扫描继续漏计 8 条 TR。
5. **`Verification Required` 49 条未跑**：在 pin 定的 4.7.x 上实测前，任何「引擎已验证」的结论都不成立；优先级见「参考库未覆盖的 4 条行为」。

### Required ADRs

按依赖序排列，共 11 份，覆盖 44 条 gap：

1. **菜品数值与整数数值契约**（dish-values 4 条）— 其余全部算式的基础
2. **回合预算、扣减所有权与耗尽判定**（time-and-turns 4 条）
3. **胃容量快照、容量消耗与耗尽判定**（stomach-capacity 5 条）
4. **局内动作提交与结束判定**（run-flow-003/004）
5. **腻度折扣与口味偏好的数值与状态机**（cloying-and-taste 5 条）
6. **结算纯函数：单位价值、入场费与回本判定**（settlement-002/003）
7. **局外升级树、钱包事务与升级快照**（meta-upgrades 6 条）
8. **解锁与档口购买事务**（unlocks 5 条，含 ADR-0007 移交的 `TR-unlocks-003`）
9. **纪录、里程碑判定与生涯完成**（records-003/005/006）
10. **HUD 数据绑定、布局与呈现契约**（in-game-hud 4 条）
11. **首局引导 beat 机、阻断输入与跳过**（first-run-onboarding 4 条）
