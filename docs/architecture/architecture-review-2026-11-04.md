## Architecture Review Report

Date: 2026-11-04
Mode: full
Workflow: standard
Engine: Godot 4.7（pin 4.7.x；`Installed at pin time = NOT DETERMINED` —— 本机无编辑器，本次评审未执行任何引擎实测，也不对任何运行时语义作「已验证」结论）
GDDs Reviewed: 21 个文件（`design/gdd/*.md` 全部：18 份系统 GDD + `game-concept.md` + `systems-index.md` + `gdd-cross-review-2026-11-04.md`；需求正文以 `tr-registry.yaml` 的 104 条为源，GDD 按 ADR 引用做定点复核）
ADRs Reviewed: 9（ADR-0001 … ADR-0009，全部 `Accepted`；`adr-status.sh`：`accepted 9 / proposed 0 / unfilled 0 / no_status 0`）

Reviewed-Content-Hash: docs/architecture/adr-0001-content-data-format.md d852e625024758f9850044178f407e4afd4139c8
Reviewed-Content-Hash: docs/architecture/adr-0002-persistence-format.md 9e10cec7e38f5f9b9dac69b7f61eb85a0e4f39a0
Reviewed-Content-Hash: docs/architecture/adr-0003-scene-screen-architecture.md 43ca6a5a27411abed15d2f79ccabebec339a1427
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md 5eb75c56013e0e8b63683617276a17e4171b6dd2
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 1b7b8dbaeceec25d8353c0fbbba1068ce9ad8657
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 59f71fe563956e278c0d84fc69eef233005b94e1
Reviewed-Content-Hash: docs/architecture/adr-0008-settings-localization.md d990234c417059bf8916d530b1ef59b824ecaf99
Reviewed-Content-Hash: docs/architecture/adr-0009-audio-playback-pause.md 70283f99412bae1960fcf5a6ad3c25d1e629f680
Reviewed-Content-Hash: design/gdd/audio-feedback.md b02b3fa05db215dff9f4882849161e7651506ef4
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md e6aa08b28eb7739e9a5c342f358f9f55b5292bde
Reviewed-Content-Hash: design/gdd/data-config.md cd42141a823868fc47650210cb108c0ae69259a0
Reviewed-Content-Hash: design/gdd/dish-values.md 8b48f38c7e01eb84a462bf847a8bbac3a7101997
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md db09042623fa8333cd6410aa8e97249c1f45e9cf
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/gdd-cross-review-2026-11-04.md c6efa8393fa2f8c10b80d5964b75aa9793f7d042
Reviewed-Content-Hash: design/gdd/in-game-hud.md f81ce0c2348338cd8bd3858ed659e2b13c882a06
Reviewed-Content-Hash: design/gdd/input.md 27c9d885eb2ba1f8d4b3df72ff6640fed0a5dd36
Reviewed-Content-Hash: design/gdd/menu-waves.md 4c2da744c1c4b19bc861bd0c09509d6deecbd6b7
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 3fa3e55b62e80ebcb1ed150a57577416f639faf9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 7cb99488fee8beeba202523848adecd3c06c2c7c
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/save-system.md 1bd3148294c81eb9a186f218cbb390b30b0e08ad
Reviewed-Content-Hash: design/gdd/scene-flow.md 07753accfc002be7257cbcdd17323c73c2b6845a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 9bf45c1ead7e161599186ca5724fa460b044e314
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/gdd/systems-index.md 1119e17f51ee38b2b6fdb56db5d6301d6db94f47
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/ui-flow.md cf471d2d45476ff8e1db0a5fa4a7fae9a4e146ea
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 26c30916e1446a1cd30ca9d44f71982adecb104e
Reviewed-Content-Hash: docs/architecture/tr-registry.yaml b5633060b13f7d4e94043b1bbd4d0dfa68b8fc4d
Reviewed-Content-Hash: docs/architecture/architecture.md 9404e976565144b370cbd998962a528958c8679f
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

Freshness: `RECEIPT` 命中同日首次评审，`review-receipts.sh check` 报 **15 CHANGED / 41 UNCHANGED / 1 GONE**。
变更的 15 项全部是预期的机械修复：9 份 ADR 的 `Status` 由 `Proposed` 改为 `Accepted`（含 ADR-0001 补第六清单、ADR-0005 `Depends On: None`、ADR-0009 补 `## GDD Requirements Addressed`），5 份 GDD 的复审修订项落地（`audio-feedback`、`data-config`、`menu-waves`、`meta-upgrades`、`systems-index`），以及 `architecture.md` 的覆盖数字与 ADR Audit 同步。`GONE` 项是 `docs/consistency-failures.md`——该文件本就不存在，是可选输入，不是丢失。因此本次是**基于新输入的完整重跑**，不沿用上一版结论。

---

### Traceability Summary

Total requirements: 104（`tr-registry.yaml` 104 条 `status: active`；`registry.sh validate` 无问题；无 deprecated / superseded）
✅ Covered (Accepted ADR): 56（53.8%）
🟡 Covered (Proposed ADR): 0
⚠️ Partial: 4
❌ Gaps: 44（42.3%）
❓ Not assessed: 0（9 份 ADR 都可读且都有 `## Status`）

| Layer | Total | ✅ | ⚠️ | ❌ |
|---|---|---|---|---|
| Foundation（data-config, input, scene-flow, save-system） | 24 | 23 | 1 | 0 |
| Core（run-flow, time-and-turns, stomach-capacity, dish-values） | 21 | 4 | 2 | **15** |
| Feature（cloying, settlement, meta-upgrades, menu-waves, unlocks, records） | 34 | 12 | 1 | **21** |
| Presentation（hud, ui-flow, audio-feedback, onboarding） | 25 | 17 | 0 | 8 |
| **Total** | **104** | **56** | **4** | **44** |

Coverage 的来源是每份 ADR 的 `## GDD Requirements Addressed` 表（ADR-0009 的两张表都算），不是正文里顺带提到的 TR 编号。逐条矩阵见 `docs/architecture/requirements-traceability.md`。

与上一版报告的唯一实质差别：56 条 covered 的**依据**从 `Proposed` 变成了 `Accepted`，映射本身未变（44 gaps 一条未减）。也就是说，本轮把「覆盖存在但未决」升级为「覆盖已决」，并没有新增架构覆盖。

⚠️ Partial（ADR 自述不完整，不能记作 covered）：

| Requirement | ADR | 缺口 |
|---|---|---|
| `TR-run-flow-006` | ADR-0007 | 只定了 `run_settled` / `run_faulted` / `run_discarded` 的边界；信号表全表仍在 run-flow |
| `TR-run-flow-007` | ADR-0007 | 只定了快照的唯一发布者；字段清单仍在 RunFlow / HUD |
| `TR-settlement-005` | ADR-0007 | 运行期累加器归 `RunFlow`；纯函数公式仍未定 |
| `TR-save-system-006` | ADR-0002 / 0004 / 0007 | 三份 ADR 合起来覆盖「结算来源的一次事务 + 保留 dirty」；原子写与错误词表在 ADR-0002 内部章节，未落在该 TR 的验收面上 |

### Coverage Gaps (no ADR exists)

| Requirement ID | GDD | System | Requirement | Suggested ADR | Domain | Engine Risk |
|---|---|---|---|---|---|---|
| `TR-run-flow-003` | run-flow.md | run-flow | `submit_action(eat/skip_wave/end_pickup, dish_id)` with fixed validation order and atomic commit; non-final paid skip rejected as `NO_TURNS` at `turns_remaining = 1` before any mutation | 阶段化的局内动作提交与结束判定 | run-flow | LOW |
| `TR-run-flow-004` | run-flow.md | run-flow | Only two end reasons (`TIME_EXHAUSTED`, `NO_EATABLE_DISH`); fixed priority; state frozen after `Ending` | 阶段化的局内动作提交与结束判定 | run-flow | LOW |
| `TR-time-and-turns-001` | time-and-turns.md | time-and-turns | `initialize(turn_budget)` once per run; `INVALID_TURN_BUDGET` on bad input, no clamping | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-time-and-turns-002` | time-and-turns.md | time-and-turns | `spend_turn()` cost exactly 1; `can_spend_turn()` pure; no overdraft, no refund | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-time-and-turns-003` | time-and-turns.md | time-and-turns | `is_exhausted()` reports only; end-reason ownership stays in run-flow | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-time-and-turns-004` | time-and-turns.md | time-and-turns | Budget frozen for the run; changes apply next `initialize` | 回合预算、扣减所有权与耗尽判定 | time-and-turns | LOW |
| `TR-stomach-capacity-001` | stomach-capacity.md | stomach-capacity | `initialize(upgrade_snapshot)` exactly once per run; `end_run(reason)` authoritative on every termination path | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-002` | stomach-capacity.md | stomach-capacity | `capacity_total = C0 + capacity_bonus`, range `[1,999]`; invalid/missing snapshot fails, no default 0 | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-003` | stomach-capacity.md | stomach-capacity | `can_fit(e)` / `consume(e)` use integer `effective_satiety`, inclusive boundary, monotonic used capacity | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-004` | stomach-capacity.md | stomach-capacity | No "full" end state; capacity only gates individual dishes | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-stomach-capacity-005` | stomach-capacity.md | stomach-capacity | Read-only accessors only; no signals, no second counter | 胃容量快照、容量消耗与耗尽判定 | stomach-capacity | LOW |
| `TR-dish-values-001` | dish-values.md | dish-values | `unit_value(d) = price/satiety` is the single cross-dish comparison scale; no downstream recomputation | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-dish-values-002` | dish-values.md | dish-values | Static 12-dish catalog; balance audits (`menu_healthy`, `opt_value`) run in design/CI only | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-dish-values-004` | dish-values.md | dish-values | Deterministic `uv_cmp` ordering (unit value desc, id asc) | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-dish-values-005` | dish-values.md | dish-values | Result-style `QueryError {caller, dish_id, reason}`; no zero/placeholder returns | 菜品价值公式、整数整形与展示值 | dish-values | LOW |
| `TR-cloying-and-taste-001` | cloying-and-taste.md | cloying-and-taste | Per-`cloying_group` integer counters reset per run; explicit canonical group order array | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-002` | cloying-and-taste.md | cloying-and-taste | `effective_satiety = satiety + max(0, min(count, cloy_cap) − floor(resistance))`, computed before increment | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-003` | cloying-and-taste.md | cloying-and-taste | `酸`/`冰` cleanse decrements the highest other group once; ties by canonical order | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-004` | cloying-and-taste.md | cloying-and-taste | `initialize(cloying_resistance)` before `run_started`; no persistence in MVP | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-cloying-and-taste-005` | cloying-and-taste.md | cloying-and-taste | Expose `extra_satiety`, `cleanse_target`, group counts to HUD before selection | 腻度折扣与口味偏好的数值与状态机 | cloying-and-taste | LOW |
| `TR-settlement-002` | settlement-and-entry-fee.md | settlement | `total_value` int64 sum; `net_profit = total_value − entry_fee`; `broke_even` derived; no clamping | 结算纯函数：单位价值、入场费与回本判定 | settlement | LOW |
| `TR-settlement-003` | settlement-and-entry-fee.md | settlement | `RunResult` schema: rows in eaten order, `return_ratio` null when fee 0, `upgrade_snapshot` passthrough | 结算纯函数：单位价值、入场费与回本判定 | settlement | LOW |
| `TR-meta-upgrades-001` | meta-upgrades.md | meta-upgrades | Wallet is int64 with exactly three flows (run net, purchase, unlock spend); negative allowed; checked overflow | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-002` | meta-upgrades.md | meta-upgrades | Exponential `upgrade_price` from `cost`/`cost_growth`; linear effect per level; hard `max_level` | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-003` | meta-upgrades.md | meta-upgrades | `resolve_upgrade_snapshot()` returns parsed bonuses plus frozen `levels` copy | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-004` | meta-upgrades.md | meta-upgrades | Purchase transaction: validate → memory apply → `save.commit` → success; rollback on write failure | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-006` | meta-upgrades.md | meta-upgrades | `spend(amount, reason, extra)` generic wallet transaction; `extra` merged into the same commit | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-meta-upgrades-007` | meta-upgrades.md | meta-upgrades | `menu_vision_useful_max = ceil((wave_size−1)/effect_value)`; no no-effect level sold | 局外升级树、钱包事务与升级快照 | meta-upgrades | MEDIUM |
| `TR-unlocks-001` | unlocks-and-stations.md | unlocks | Availability derived: `unlock_cost = 0` or id in `unlocks.stations`; dishes follow the station | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-unlocks-002` | unlocks-and-stations.md | unlocks | Normalize/backfill free stations and dishes; empty set → `NoPlayableStation`; never delete ids | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-unlocks-003` | unlocks-and-stations.md | unlocks | Unlock transaction via `meta-upgrades.spend` with `{unlocks}` merged into the same commit | 解锁与档口购买事务（wallet + unlocks 同一 commit；ADR-0007 已显式移交） | unlocks | MEDIUM |
| `TR-unlocks-004` | unlocks-and-stations.md | unlocks | Fixed validation order incl. `NOT_IN_MENU`, `BUSY`, `INSUFFICIENT_FUNDS` | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-unlocks-005` | unlocks-and-stations.md | unlocks | Catalog/progress/effective-unlock read APIs; id-sorted output | 解锁与档口购买事务（wallet + unlocks 同一 commit） | unlocks | MEDIUM |
| `TR-records-003` | records-and-milestones.md | records | Per-station ranking by `net_profit`; strict improvement only; ties keep the old record | 纪录、里程碑判定与生涯完成 | records | LOW |
| `TR-records-005` | records-and-milestones.md | records | UTC timestamps display-only; no logic reads the clock | 纪录、里程碑判定与生涯完成 | records | LOW |
| `TR-records-006` | records-and-milestones.md | records | Signals `record_broken`, `milestone_earned`, `career_completed` | 纪录、里程碑判定与生涯完成 | records | LOW |
| `TR-in-game-hud-001` | in-game-hud.md | in-game-hud | Read-only projection; one immutable `HudFrame` per event; no polling between frames | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-in-game-hud-003` | in-game-hud.md | in-game-hud | Card composed from dish/cloying/capacity/wave providers; no sorting, recommendation, or `unit_value` | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-in-game-hud-005` | in-game-hud.md | in-game-hud | Fixed number formats; color always backed by text/symbol | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-in-game-hud-006` | in-game-hud.md | in-game-hud | `get_anchor_rect(anchor_id)` and three protected rects for onboarding | HUD 数据绑定、布局与呈现契约 | in-game-hud | HIGH |
| `TR-first-run-onboarding-001` | first-run-onboarding.md | first-run-onboarding | Runs only when `RunConfig.is_first_run`; preset `pool_override` six dishes and `entry_fee_override = 40` | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |
| `TR-first-run-onboarding-003` | first-run-onboarding.md | first-run-onboarding | Anchors from HUD `get_anchor_rect`; bubbles never cover the three protected rects | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |
| `TR-first-run-onboarding-004` | first-run-onboarding.md | first-run-onboarding | `FirstRunExplanation` payload for the receipt; no persistence; `onboarding_finished` once per run | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |
| `TR-first-run-onboarding-005` | first-run-onboarding.md | first-run-onboarding | Degradation order for missing preset/anchors/unit values; never blocks the run | 首局引导 beat 机、阻断输入与跳过 | first-run-onboarding | HIGH |

### Cross-ADR Conflicts

**上一版报告的两条冲突均已消解：**

- **ADR-0003 ↔ ADR-0005（含 ADR-0004）依赖环** — ADR-0005 的 `Depends On` 已改为 `None`，`adr-dep-graph.sh` 不再输出 `CYCLE` / `BLOCKED_BY_CYCLE`，`ORDER` 覆盖全部 9 份 ADR。
- **ADR-0001 ↔ ADR-0009 第六清单** — ADR-0001 §2 清单表已补 `AudioCueCatalogResource`，`Validating` 增加 5b 子阶段与 8 条 `AUDIO_CUE_*` 规则，契约闭合。

**本轮新增 1 条集成级冲突（不阻塞，需措辞对齐）：**

## Conflict: ADR-0008 vs ADR-0009（音量写入路径）
Type: Integration
ADR-0008 claims: 设置生效路径为 `AudioFeedback.apply_settings(audio) → AudioServer 总线`（`architecture.md` 内嵌图，ADR-0008:312），即由音频侧把设置写到 `AudioServer` 总线。
ADR-0009 claims: `Master / SFX / UI` 三条总线音量**恒为 0 dB、运行期永不写入**；响度只在每个 voice 的 `volume_db` 上计算一次，只有 `Music` 总线会被闪避包络写入（D9）。
Impact: 两份 ADR 都 `Accepted`，但实现者按 ADR-0008 的图会把音量写进总线，直接违反 ADR-0009 的「单一响度来源」不变量，使无头测试无法只读 voice 断言 GDD 公式。不是契约级矛盾（设置项的所有权仍在 ADR-0008），是负责写入的组件与写入位置的表述冲突。
Resolution options:
  1. （推荐）把 ADR-0008:312 的箭头改为 `AudioFeedback.apply_settings(audio) → SettingsModel 快照（不作为总线写入）`，并注明总线写入由 ADR-0009 D9 禁止。
  2. ADR-0009 改为写三条总线，同时放弃「voice 单一响度来源」与对应的无头断言——与 D12 的可观测性设计冲突，不推荐。

**已核对、判定不构成冲突**（避免误报）：
- 暂停模型：ADR-0003 D2.4 的 `ALWAYS` 清单与 ADR-0004 D10 的 `HudHost = INHERIT` 一致。
- 随机源：ADR-0004 的 `SessionRng` 与 ADR-0006 的确定性 RNG 明确互不触碰。
- 收据与提交流程：ADR-0003 D7 与 ADR-0007 D3 顺序一致。
- 纪录信号时机：ADR-0004:236 已改为「结算事务已应用后」，消解了 ADR-0007 登记的 GDD 矛盾 2。

**ADR 自洽性缺陷（不是跨 ADR 冲突，但会误导实现）：**
- `adr-0003-scene-screen-architecture.md:443` 仍写「未验证前不得把本 ADR 标记为 Accepted」，而该 ADR 的 `Status` 已是 `Accepted`。同一份文档里 `Verification Required` ① 与状态互相否定。

### ADR Dependency Order

`adr-dep-graph.sh`：`ADRS: 9`、20 条边、`NO_DEPS_SECTION:` 无、`UNFILLED_DEPS:` 无、无 `CYCLE`、无 `BLOCKED_BY_CYCLE`。

```
ORDER: adr-0001, adr-0002, adr-0005, adr-0003, adr-0004, adr-0008, adr-0006, adr-0007, adr-0009
```

前提：ADR-0001 / ADR-0002 为 `None`，ADR-0005 为 `None`（本轮修复）；其余 6 份的依赖都落在这三份与彼此之间，因此图是干净 DAG。

### Recommended ADR Implementation Order

Foundation (no dependencies):
  1. ADR-0001：内容数据格式、校验与启动门
  2. ADR-0002：持久化格式、原子写入与跨平台写锁
  3. ADR-0005：输入上下文栈语义
Depends on Foundation:
  4. ADR-0003：场景与屏幕架构（requires ADR-0005）
  5. ADR-0004：服务组合与信号机制（requires ADR-0001, 0002, 0003, 0005）
Feature / Presentation:
  6. ADR-0008：机器级设置与本地化管线（requires ADR-0001, 0002, 0003）
  7. ADR-0006：确定性随机、种子派生与健康校验预算（requires ADR-0001, 0004）
  8. ADR-0007：局内状态所有权与结算事务边界（requires ADR-0001–0004）
  9. ADR-0009：音频播放、池化与暂停语义（requires ADR-0001, 0003, 0004, 0006, 0008）

⚠️ 拓扑序解决的是「文档间谁先实现」，不解决「链上是否有未决规则」：9 份 ADR 虽已 `Accepted`，49 条 `Verification Required` 仍全部未实测，因此任何「引擎语义已验证」的结论都不成立。

### GDD Revision Flags

上一版报告登记 17 条，本轮复核后 **15 条已落地**（`audio-feedback` 的 9 条全部应用、`data-config` 补第六清单、`menu-waves` 的 ±10% 重掷、`meta-upgrades` 的 `APPLIED_UNSAVED`、`systems-index` 的复审状态同步、`records` / `run-flow` / `scene-flow` / `unlocks` 的编排者与 `fault.phase` 修订）。**仍剩 2 条未改：**

| # | GDD | 位置 | GDD 现状 | 建议改为 | 来源 |
|---|---|---|---|---|---|
| 1 | save-system.md | 规则 21（:47） | 进程崩溃遗留的锁「以进程存活为主、时间戳为辅」回收 | 锁目录 + `owner.json` + 心跳，`PROCESS_MODE_ALWAYS` 驱动者，90 s 超时 / 24 h 上限；无法判定时按「存活」处理 | ADR-0002 §3 |
| 2 | dish-values.md | 规则 11（:37） | 「`opt_value` … 不进入游戏运行时」 | 补一句：唯一例外是 `MenuWaves` 的运行期健康门（`wave_reachable_opt`），结果不外流、不进存档/HUD/结算 | ADR-0006 §4 |

两条都属于「ADR 已决、GDD 未回写」：实现会以 ADR 为准，但 GDD 仍是 Approved，读者会被旧措辞误导。`design/gdd/systems-index.md` 已把全部 18 个系统标为 Approved，因此这两条不再由索引兜底提示。

### Engine Compatibility Issues

- **版本一致性 ✅**：9/9 份 ADR 的 `Engine Compatibility` 都写 Godot 4.7（`4.7.x` / latest patch `4.7.2`），与 `docs/engine-reference/godot/VERSION.md`、`project.yaml` 一致，无第二版本声明。
- **Post-cutoff API 一致性 ✅**：逐条比对 `breaking-changes.md` / `current-best-practices.md`——4.5 `@abstract`（ADR-0001）、4.6 双焦点（ADR-0003/0005/0008）、4.7 新项目 stretch 默认 `canvas_items`+`expand`（ADR-0003）、4.7 `DEVICE_ID_MOUSE`/`DEVICE_ID_KEYBOARD`（ADR-0003/0005）、4.7 `ResourceImporterDynamicFont.hinting` 默认 3（ADR-0008）、4.6 CSV plural/context 列（ADR-0008）——与参考库一致，无「发明 API」。
- **废弃 API 扫描 ✅（ADR 中 0 命中）**：9 份 ADR 未出现 `TileMap`、`yield(`、`.instance()`、`VisibilityNotifier*`、`get_world()`、`OS.get_ticks_msec`、`width_in_percent`、嵌套资源 `duplicate()`；`device == 0` 仅作为禁令出现（ADR-0005 D8 / 指南 11）。
- **模块参考 ✅**：ADR 引用的 `modules/ui.md`、`modules/input.md`、`modules/audio.md` 均存在（8 份模块文档），无缺失引用。
- **Engine Compatibility 段缺失：无**（9/9 都有）。ADR-0009 的 `## GDD Requirements Addressed` 已补齐（8 行，含 `TR-audio-feedback-001..007`），`adr-status.sh` 不再把它列入 missing。
- **引擎验证未执行（关键）**：`Installed at pin time = NOT DETERMINED`，因此 **49 条 `Verification Required` 全部未实测**（0001:5、0002:5、0003:8、0004:5、0005:6、0006:5、0007:4、0008:6、0009:5）。本报告任何 `✅` 只表示「与参考文档不矛盾」，不表示「已在本机 4.7 上验证」。
- **参考库未覆盖、需优先实测的 4 条行为**：`Viewport.push_input(InputEventAction)` 是否进入 GUI 焦点导航（ADR-0005 ①）、`focus_neighbor_*` 指向不可聚焦控件时的几何回落（ADR-0003 ⑥）、`PROCESS_MODE_ALWAYS` 下 `Tween` 是否推进（ADR-0004 ①）、`SceneTree.paused` 下 `AudioStreamPlayer` 是否继续播放（ADR-0009 ①）。这 4 条决定 UI 输入与暂停音的实现形状。

### 引擎专项复核发现（E1–E13）

以下 13 项是对仓库事实的定点核对，不是运行时实测；`E` 编号为本次评审内部编号。

| # | 严重度 | 位置 | 发现 | 建议动作 |
|---|---|---|---|---|
| E1 | HIGH | `project.godot` | 没有任何 `display/window/stretch/mode` / `aspect` 键。ADR-0003 依赖 4.7「新项目默认 `canvas_items` + `expand`」，但参考库的措辞是「NEW projects 默认，existing projects keep their settings」——缺键时取引擎内置默认，不能等同于新项目模板写入的值 | 在 `project.godot` 显式写入 `display/window/stretch/mode="canvas_items"` 与 `display/window/stretch/aspect="expand"`，让 ADR-0003 的假设成为仓库事实 |
| E2 | HIGH | `.gitignore:70-71` | `*.translation` 与 `export_presets.cfg` 被忽略。ADR-0008 依赖 CSV→`.translation` 的导入产物与导出预设（CJK 字体、`locale/translations` 注册）；新克隆在跑 `--import` 前没有 `.translation`，导出配置也不可复现 | 保留忽略生成物，但在 CI/克隆说明中加「导入后才能过启动门」的门禁；`export_presets.cfg` 应改为版本化或提供模板 |
| E3 | HIGH | ADR-0003:443 | 「未验证前不得把本 ADR 标记为 Accepted」与 `Status: Accepted` 直接矛盾 | 改写为「已接受；`Verification Required` ① 未完成前，涉及 `focus_neighbor_*` 回落的实现按待验证处理」 |
| E4 | MEDIUM | ADR-0008:312 | `AudioFeedback.apply_settings(audio) → AudioServer 总线` 与 ADR-0009 D9「三条总线恒 0 dB、运行期永不写入」冲突 | 见 Cross-ADR Conflicts 的推荐方案 1 |
| E5 | MEDIUM | `tr-registry.yaml:48` / `:56` | `TR-data-config-001` 仍写「Four typed Resource catalogs」，`TR-data-config-002` 的 API 列表仍只有 4 个 getter；ADR-0001 已是 6 类清单 + `get_cue`/`has_cue`/`cue_catalog` | 修订两条 TR 文本并置 `revised: 2026-11-04`（ADR-0009 的建议修订一并处理） |
| E6 | MEDIUM | `architecture.md:249-256` | DataConfig 契约缺 `get_cue` / `has_cue` / `cue_catalog`，与 ADR-0001 不一致 | 同步 API Boundaries |
| E7 | MEDIUM | `architecture.md:351` | 「Nine Foundation/Feature ADRs … none of the Required ADRs remains unwritten」——实际仍有 11 份 Required ADR 未写（44 条 gap） | 改为「9 份已写并接受；11 份 Required ADR 仍缺」，并补列 44 gaps |
| E8 | MEDIUM | `architecture.md:362` | ADR-0006 行把 `TR-run-flow-003` 记为 `✅`，但 ADR-0006 只在冲突说明正文提到它，traceability 判为 gap | 从 ADR-0006 的 GDD Linkage 移除 `TR-run-flow-003` |
| E9 | LOW | `architecture.md:363-364` | ADR-0008 行排在 ADR-0007 之前；`Conflicts` 列全为 `None`，与 E4 的发现不一致 | 按编号排序；在 ADR-0008 行注明与 ADR-0009 的措辞冲突 |
| E10 | LOW | ADR-0009:600 | `## TR Coverage` 表只有 001–006 与 `TR-ui-flow-007`，漏了 `## GDD Requirements Addressed` 里的 `TR-audio-feedback-007` | 补一行，或删掉重复的 `## TR Coverage` 段（两表已近乎重复） |
| E11 | LOW | ADR-0007 验证 ② | 只断言 `Dictionary.duplicate(true)` 深拷贝嵌套数组，未覆盖 `Resource` 值——`duplicate(true)` 不递归复制 `Resource`，若 `RunResult.upgrade_snapshot` 持有 `Resource`，「冻结快照」不变量可能被破坏 | 在验证 ② 明确：快照只存纯值（Dictionary/Array/int），不存 `Resource` 引用 |
| E12 | LOW | `tests/` | 仅 `tests/unit/framework_test.gd` + 两个 `.gdignore_placeholder` + `tests/smoke/critical-paths.md`；Godot 未安装，`/test-setup` 无法跑通，`tests.yml` 无法本机验证 | 安装 4.7.x 后跑 `/test-setup`，先补一条无头启动冒烟 |
| E13 | LOW | `design/gdd/systems-index.md` | 18 个系统全标 `Approved`，但 `save-system` 规则 21 与 `dish-values` 规则 11 仍与 ADR-0002 §3 / ADR-0006 §4 不一致（GDD Revision Flags 第 1、2 条） | 回写这两条 GDD；在回写前索引不应单独作为「无待修订」的依据 |

### Architecture Document Coverage

- `docs/architecture/architecture.md` 存在（v1，562 行）→ Phase 6 已运行，本项**不是** NOT ASSESSED。
- **系统覆盖 ✅**：System Layer Map 列出全部 18 个已设计系统，另列 2 个 deferred（在线挑战、同屏合作）；Data Flow 1–6 完整；不变量 6 条与 ADR 一致。
- **API Boundaries ⚠️ 12/18**：只有 DataConfig、SceneFlow、InputRouter、SaveSystem、RunFlow、TimeTurns、StomachCapacity、CloyingAndTaste、MenuWaves、Settlement、MetaUpgrades、Records 有契约。缺 6 个：**Dish Values、Unlocks & Stations**、In-Game HUD、UI Flow、Audio Feedback、First-Run Onboarding。后四个由 ADR-0003/0005/0008/0009 补齐；**Dish Values 与 Unlocks 在全部文档中都没有 API 边界**，这与其 9 条 gap 同源。
- **孤儿架构 ✅ 无**：`SessionRng` / `BootSequence` / `ScreenFactory` 都能追溯到 ADR-0004 / ADR-0003 的基础设施决定。
- **文档漂移（需编排者修，本次未改）**：
  - :10 仍写「评审报告的 `Reviewed-Content-Hash` 早于该状态变更；依赖报告结论前需重跑 `/architecture-review full`」——本次重跑后已过期，应改为「已重跑，回执晚于接受」。
  - :351 称「none of the Required ADRs remains unwritten」——11 份 Required ADR 仍缺（E7）。
  - :362 把 `TR-run-flow-003` 归给 ADR-0006（E8）。
  - :249-256 DataConfig 契约缺 cue API（E6）。
  - `ADR Audit` 的 `Conflicts` 列全为 `None`，未反映 ADR-0008/0009 的措辞冲突（E9）。
  - ADR-0008 行与 ADR-0007 行顺序颠倒（E9）。
  - `tr-registry.yaml` 的 `TR-data-config-001/002` 文本落后于 ADR-0001（E5）。

---

### Verdict: FAIL

判定依据（`references/7-report.md`）：FAIL 的条件是「Critical gaps（Foundation / Core 层需求未覆盖）**或** 存在阻塞性跨 ADR 冲突」。本轮只剩第一条成立：

1. **Core 层 15/21 条需求没有 ADR**：时间与回合 4/4、胃容量 5/5、菜品数值 4/5、餐局流程 2/7。Core 层就是「回本」的全部算式（回合预算、容量预算、单位价值），没有任何一条被 Accepted 的 ADR 约束。
2. **依赖环已消解**，`ORDER` 覆盖全部 9 份 ADR；ADR-0008/0009 的冲突是措辞级、非阻塞。因此 FAIL 的**唯一**主因是覆盖率。

补充说明（不改变判定，但决定后续路线）：
- 与上一版相比，`accepted = 9` 使 56 条 covered 从 `🟡` 升为 `✅`，PASS 的「状态」门槛已跨过；剩下的纯粹是 44 条 gap。这意味着**只要补上 11 份 ADR 并保持无阻塞冲突，下一次 full 评审即可从 FAIL 直接跳到 PASS**（前提是修正 `TR-run-flow-003` 的归属口径，避免新增 partial）。
- 评审是 advisory：FAIL 不阻止用户继续，但阻止 `/create-stories` 为时间与回合 / 胃容量 / 菜品数值 / 腻度与口味开故事。

### Blocking Issues (must resolve before PASS)

1. **Core 层 15 条需求无 ADR**：`TR-time-and-turns-001..004`、`TR-stomach-capacity-001..005`、`TR-dish-values-001/002/004/005`、`TR-run-flow-003/004`。补齐前不得为这四个系统写 story；`architecture.md` 的 API Boundaries 应同时补 Dish Values。
2. **11 份 Required ADR 未写**：按「Required ADRs」的依赖序写。`architecture.md:351` 的「none remains unwritten」需先改掉，否则编排者会误以为已完结。
3. **`project.godot` 缺 stretch 键（E1）**：ADR-0003 的布局契约建立在一个未写入仓库的默认值上。这是第一个 story 之前就能修、且不修会静默跑偏的项。
4. **ADR-0003:443 自相矛盾（E3）**：已 Accepted 的 ADR 仍写「未验证前不得 Accepted」，会让实现者把状态当未决。
5. **ADR-0008/0009 音量写入路径（E4）**：两份 Accepted ADR 给出相反的写入位置；按 ADR-0008 实现会违反 ADR-0009 的核心不变量。

（上一版的「依赖环」与「9 份 ADR 全 Proposed」两条阻塞项已关闭，不再列出。）

### Required ADRs

按依赖序排列，共 11 份，覆盖 44 条 gap：

1. **菜品价值公式、整数整形与展示值**（dish-values 4 条）— 其余全部算式的基础
2. **回合预算、扣减所有权与耗尽判定**（time-and-turns 4 条）
3. **胃容量快照、容量消耗与耗尽判定**（stomach-capacity 5 条）
4. **阶段化的局内动作提交与结束判定**（run-flow-003/004）
5. **腻度折扣与口味偏好的数值与状态机**（cloying-and-taste 5 条）
6. **结算纯函数：单位价值、入场费与回本判定**（settlement-002/003）
7. **局外升级树、钱包事务与升级快照**（meta-upgrades 6 条）
8. **解锁与档口购买事务**（unlocks 5 条，含 ADR-0007 移交的 `TR-unlocks-003`）
9. **纪录、里程碑判定与生涯完成**（records-003/005/006）
10. **HUD 数据绑定、布局与呈现契约**（in-game-hud 4 条）
11. **首局引导 beat 机、阻断输入与跳过**（first-run-onboarding 4 条）
