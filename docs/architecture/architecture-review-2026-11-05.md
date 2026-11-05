## Architecture Review Report

Date: 2026-11-05
Mode: full
Workflow: standard
Engine: Godot 4.7（pin 4.7.2；本机无编辑器，未执行任何引擎实测，也不对任何运行时语义作「已验证」结论）
GDDs Reviewed: 21 个文件（`design/gdd/*.md` 全部：18 份系统 GDD + `game-concept.md` + `systems-index.md` + `gdd-cross-review-2026-11-04.md`；需求正文以 `tr-registry.yaml` 的 104 条为源）
ADRs Reviewed: 20（ADR-0001 … ADR-0009 为 `Accepted`；ADR-0010 … ADR-0020 为 `Proposed`；`adr-status.sh` 口径：accepted 9 / proposed 11 / unfilled 0 / no_status 0）

Reviewed-Content-Hash: docs/architecture/adr-0001-content-data-format.md d852e625024758f9850044178f407e4afd4139c8
Reviewed-Content-Hash: docs/architecture/adr-0002-persistence-format.md 9e10cec7e38f5f9b9dac69b7f61eb85a0e4f39a0
Reviewed-Content-Hash: docs/architecture/adr-0003-scene-screen-architecture.md c0a93fbb6f9e86d6c06d2e4f9173a2258fd94055
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md 5eb75c56013e0e8b63683617276a17e4171b6dd2
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 1b7b8dbaeceec25d8353c0fbbba1068ce9ad8657
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 842e888d73932cc3a913a92b720b16587aa801d2
Reviewed-Content-Hash: docs/architecture/adr-0008-settings-localization.md d990234c417059bf8916d530b1ef59b824ecaf99
Reviewed-Content-Hash: docs/architecture/adr-0009-audio-playback-pause.md a4f849c038b2a3244e98a7ed91fb4604200efe44
Reviewed-Content-Hash: docs/architecture/adr-0010-dish-values-numeric-contract.md 3ce4e7de6c5f19497d04a1114d3d84abb2b99c81
Reviewed-Content-Hash: docs/architecture/adr-0011-turn-budget-exhaustion.md 78c89936482a65b6f3b1756ad33a26881ace776f
Reviewed-Content-Hash: docs/architecture/adr-0012-stomach-capacity-ledger.md 6b1cd5a8dfaf190d940f78cc6ebf138093790249
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 4c6aa231340a557ee42ab36e06b3f359c5a0be65
Reviewed-Content-Hash: docs/architecture/adr-0014-cloying-taste-state-machine.md 7e7671ce0e476f943b44961a1801c7f52c019b9b
Reviewed-Content-Hash: docs/architecture/adr-0015-settlement-pure-functions.md 9b7d189987bdcb4e766a036973f8a5ccdeb73b7d
Reviewed-Content-Hash: docs/architecture/adr-0016-meta-upgrades-wallet.md ea64ce5f996c6b76220201ccc253e241da4163c4
Reviewed-Content-Hash: docs/architecture/adr-0017-unlocks-station-purchase.md 044da463a17c3efe1b3fa581f4f49cc552eda4eb
Reviewed-Content-Hash: docs/architecture/adr-0018-records-milestones-career.md 09861c904839d40a4bfd59bd5b58099e61c7a768
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 05f4963102a79a9760d80f9b0793979f4080aba8
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 1a5437a94ce966790b37db40be845e0993d44551
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
Reviewed-Content-Hash: docs/architecture/tr-registry.yaml b26b2e0ff92eec3bda1b34779ed6dd850f0b4941
Reviewed-Content-Hash: docs/architecture/architecture.md 6542bc47c17572813ea6540bd39ad813c81a3cbf
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
review-receipts: not stamped, no such file: docs/consistency-failures.md
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

---

### Traceability Summary

| 口径 | 2026-11-04 | 2026-11-05 | 变化 |
|---|---|---|---|
| 总需求 | 104 | 104 | — |
| ✅ 由 **Accepted** ADR 覆盖 | 56 | **57** | +1 |
| 🟡 仅由 **Proposed** ADR 覆盖 | 0 | **46** | +46 |
| ⚠️ Partial | 4 | **1** | −3 |
| ❌ Gap | 44 | **0** | −44 |

| Layer | Total | ✅ Accepted | 🟡 Proposed | ⚠️ Partial | ❌ Gap |
|---|---|---|---|---|---|
| Foundation | 24 | 24 | 0 | 0 | 0 |
| Core | 21 | 4 | 16 | 1 | 0 |
| Feature | 34 | 12 | 22 | 0 | 0 |
| Presentation | 25 | 17 | 8 | 0 | 0 |
| **Total** | **104** | **57** | **46** | **1** | **0** |

逐条矩阵见 `docs/architecture/requirements-traceability.md`。判定口径按
`references/3-traceability.md`：只有 **Accepted** ADR 的覆盖才是 `✅`；`Proposed`
覆盖记 `🟡`，并只把判定上限压到 CONCERNS。ADR-0010…ADR-0020 是 2026-11-04
评审之后写的，本轮首次计入。

### Coverage Gaps (no ADR exists)

**无。** 104 条 active 需求每一条都至少被一份 ADR 的 `## GDD Requirements Addressed`
显式引用。2026-11-04 的 44 条 gap 已全部由 ADR-0010…ADR-0020 关闭。

因此本轮的阻塞点不再是「缺 ADR」，而是**状态**：其中 46 条只由 `Proposed` ADR
覆盖，唯一出口是 `/architecture-decision accept ADR-NNNN`，逐份接受。

### Cross-ADR Conflicts

**先核对、判定不构成冲突的配对**（避免误报）：

- ADR-0011（回合）与 ADR-0012（容量）都只报告事实，不产生 `end_reason`；结束原因与
  `TIME_EXHAUSTED` / `NO_EATABLE_DISH` 优先级唯一归 ADR-0013。三份口径一致。
- ADR-0006（洗牌与健康门）与 ADR-0013（`skip_wave` / `NO_TURNS`）：前者只冻结候选集，
  后者只调用 `advance()`；`turns_remaining = 1` 的付费跳波拒绝在 `spend_turn()` 之前，
  两处一致。
- ADR-0007（事务边界）与 ADR-0015（结算纯函数）：前者拥有提交与钱包写者，后者无 I/O、
  不碰存档；`running_net` 累加器在 `RunFlow`，求和公式在 `Settlement`，与 ADR-0015 D8 一致。
- ADR-0016（钱包）与 ADR-0017（解锁购买）：钱包只有 `MetaUpgrades` 一个写者，付费解锁
  经 `spend(..., {unlocks: …})` 并入同一次 `commit`，失败一起回滚。
- ADR-0004 的 `SessionRng` 与 ADR-0006 的 `SeedHash`：ADR-0009 的变体轮转与未来 pitch
  抖动只允许走 `SeedHash`，不消耗会话 RNG；ADR-0004/0009 已互指。
- ADR-0003 的暂停模型与 ADR-0009 的 `ALWAYS` 白名单：`AudioDirector` / `VoicePoolHost`
  都是 `PROCESS_MODE_ALWAYS` 且都不是 autoload，与 ADR-0004 D10 一致。
- ADR-0010 的 `unit_value` 与 ADR-0014 的 `effective_satiety`：前者定刻度，后者定每口修正；
  ADR-0015 的 `effective_unit_value` 同时读两者，没有第二套除法。

以下 5 条是真实的冲突或未闭合的集成契约。

## Conflict: ADR-0008 vs ADR-0009（音量写入路径，E4）
Type: Integration
ADR-0008 claims: 设置生效路径为 `AudioFeedback.apply_settings(audio) → AudioServer 总线`
（架构图 :312），迁移步骤 8 写「音频总线首帧前应用音量」（:621）。
ADR-0009 claims: `Master / SFX / UI` 三条总线音量**恒为 0 dB、运行期永不写入**；全部
增益在每个 voice 的 `volume_db` 上算一次，只有 `Music` 总线被闪避包络写入（D9）。
Impact: 两份 ADR 都是 `Accepted`。按 ADR-0008 实现会把 `master_volume_db` / `sfx_volume_db`
写进总线，再叠加 voice 公式里的 `master_gain_db` / `sfx_gain_db`，造成双重衰减，并直接
违反 ADR-0009「单一响度来源」不变量与对应的无头断言。
Resolution options:
  1. **（推荐）** 把 ADR-0008:312 的箭头改为 `AudioFeedback.apply_settings(audio) → SettingsModel 快照（不作为总线写入）`，并把 :621 改为「音频在首帧读取设置快照，增益在 voice 上应用」。
  2. 让 ADR-0009 改为写三条总线，同时放弃 voice 单一响度来源——与 ADR-0009 D12 的可观测性设计冲突，不推荐。
判断：措辞级、非阻塞（沿用 2026-11-04 的分类），但必须在第一个音频 story 之前改。

## Conflict: ADR-0020 vs ADR-0003 / ADR-0004 / ADR-0005 / ADR-0009（首局引导的宿主接线）
Type: Integration / Pattern
ADR-0020 claims:
  - `OnboardingHost` 需要 `CanvasLayer.layer = 15`（HUD 10 之上、Screen 20 之下）；
  - `OnboardingHost` 必须订阅 `run_settled` 并只做 `STASH_RESULT`，顺序 `UIRoot → Onboarding → Audio`；
  - `OnboardingHost.process_priority = -50`，启动时断言 `-100 < -50 < 0`；
  - 新增 `UIRoot.run_config_assembled(config, plan)` 信号，由 `AppRoot` 订阅后 `onboarding.configure(plan)`。
ADR-0003 claims: 宿主分层固定为 `HudHost 10 / ScreenHost 20 / OverlayHost 25 / ModalHost 30 / NoticeHost 40 / FadeLayer 50`，没有第 15 层。
ADR-0004 claims: `run_settled` 的订阅者只有 `UIRoot`（收据编排）与 `Audio`；D8 第 5 步只有 `onboarding.on_flow_state(from,to)`，没有 plan 传递。
ADR-0005 claims: `process_priority` 只登记 `SceneFlow (−100)` 与 `HudHost (0)`。
Impact: 按两份 Accepted ADR 实现，引导宿主没有层位、收不到 `run_settled`、拿不到教学 plan，
B4 收据讲解永远不会触发；这四处在 ADR-0020 的 GDD Sync Check 里已被自己列为「需要谁对齐」。
Resolution options:
  1. **（推荐）** 在 ADR-0003 D2 层表补 `OnboardingHost 15`；ADR-0004 D9 的 `run_settled` 订阅者补 `Onboarding`、D8 补 `run_config_assembled` 一行；ADR-0005 D5/D10 补 `-50` 常量与断言。
  2. 把引导并入 `HudHost`（放弃独立宿主）——会把「引导只压输入、不改玩法」与 HUD 的只读投影混在一起，不推荐。
判断：ADR-0020 **接受之前必须解决**；因为它仍是 `Proposed`，本轮不计为阻塞判定。

## Conflict: ADR-0019 vs ADR-0004 / ADR-0005 / ADR-0007 / ADR-0008 / ADR-0013（HUD 的只读接口）
Type: Integration
ADR-0019 claims: HUD 需要 `RunFlow.action_availability()` 统一发布 `skip_visible` / `skip_enabled` / `skip_reason` / `end_*`；HUD 必须轮询 `ui_cancel`；`HudHost.apply_presentation(prefs)` 必须收到 `PresentationPrefs`；`sync_for_state(Run↔Paused)` 就是 `frozen` 变化通知。
ADR-0007 / ADR-0013 claim: `snapshot()` 只发布 `run_state` / `run_seed` / `action_seq` / `running_net` / `shortfall`，没有 `action_availability()`；`TR-run-flow-007` 的字段表不含 `frozen`。
ADR-0005 claims: D9 的静态扫描表只列 `ui_accept` / `game_skip_wave` 的 `HudHost` 读者，没有 `ui_cancel`。
ADR-0008 claims: 组合根只调用 `UIRoot.apply_presentation(prefs)`，没有 `HudHost` 入口。
ADR-0004 claims: D8 的 `hud.sync_for_state(from,to)` 只描述挂载/卸载，没有定义为冻结通知。
Impact: 按现有 Accepted ADR 实现，HUD 无法在不复制 `turns_remaining ≥ 2` 判定的前提下画跳波按钮，
确认面板收不到 `ui_cancel`，`PresentationPrefs` 到不了 HUD，`frozen` 也没有权威来源。
Resolution options:
  1. **（推荐）** ADR-0007 补 `action_availability()` 与 `frozen` 字段（同时关闭 `TR-run-flow-007` 的 partial）；ADR-0005 D9 补 `ui_cancel` 读者；ADR-0008/ADR-0004 的接线清单补 `HudHost.apply_presentation` 与 `sync_for_state(Run↔Paused) = frozen 通知`。
  2. 让 HUD 自己读 `SceneFlow.state` 与 `TimeTurns` 读数——复制判定、绕开只读投影，违反 ADR-0019 D1 与 ADR-0007 D1。
判断：ADR-0019 **接受之前必须解决**；本轮不计为阻塞判定。

## Conflict: ADR-0020 vs ADR-0002 / ADR-0014 / ADR-0016（接口形状未定义）
Type: Integration
ADR-0020 claims: `GroupCount = {group, count}`、`UpgradeView = {id, name, current_level, max_level}` + `next_price(id)`、`LoadReport.first_run_eligible`。
ADR-0014 / ADR-0016 / ADR-0002 claim: `group_counts()`、`get_catalog()` 被导出，但 `GroupCount` / `UpgradeView` 的字段没有定义；`LoadReport` 字段表不含 `first_run_eligible`。
Impact: 引导 B3/B4 的预置与文案依赖这些字段；实现者只能各自猜形状，UI 与引导会对同一个服务产生两套读法。
Resolution options:
  1. **（推荐）** ADR-0014 补 `GroupCount` 字段、ADR-0016 补 `UpgradeView` 字段与 `next_price()`；ADR-0002 明确 `first_run_eligible` 由 `LoadReport` 提供，或由 UI 流程按 `outcome` 派生并在 ADR-0020 写明。
  2. 让 ADR-0020 自己适配现有形状——会把「引导专用视图」塞进通用服务，扩大 API 面。
判断：ADR-0020 接受之前解决；属于 `Proposed` 之间的契约缺口。

## Conflict: ADR-0007 内部 D4 第 7 步 vs D5（纪录信号的采纳点，E17）
Type: State / Integration
ADR-0007 D4 claims: 事务第 7 步「采纳与发信号」发生在 `Save.commit` 之后。
ADR-0007 D5 claims: `record_broken` / `milestone_earned` / `career_completed` 在 `prepare_run` 返回 `APPLIED` 时发出，不把发信号搬到 `commit` 之后。
Impact: 同一份已 `Accepted` 的 ADR 给出两个采纳点。写盘失败时按 D4 实现不会发纪录信号、收据没有印章；按 D5 实现会发。ADR-0018 的 GDD Sync Check 第 8 条已指出这一点。
Resolution options:
  1. **（推荐）** 把 D4 第 7 步改写为「采纳 progression（钱包）；Records 的采纳与信号发生在第 2 步 `prepare_run = APPLIED` 时」，与 D5 和 ADR-0004 D9 的「事务已应用后」统一。
  2. 把 D5 改成「commit 之后发信号」——会让 `UNSAVED` 收据丢失纪录印章，与 ADR-0007 的「永不回滚内存」取舍冲突。
判断：Accepted ADR 的内部措辞矛盾，非跨 ADR 阻塞；在第一个纪录 story 之前改。

### ADR Dependency Order

`adr-dep-graph.sh`：`ADRS: 20`、`EDGES: 72`、`NO_DEPS_SECTION:` 无、`UNFILLED_DEPS:` 无、无 `CYCLE`、无 `BLOCKED_BY_CYCLE`。

```
ORDER: adr-0001, adr-0002, adr-0005, adr-0003, adr-0004, adr-0008, adr-0006, adr-0007, adr-0010, adr-0009, adr-0011, adr-0012, adr-0014, adr-0013, adr-0015, adr-0019, adr-0016, adr-0017, adr-0018, adr-0020
```

**未决依赖检查**：没有一份 `Accepted` ADR 依赖 `Proposed` ADR；新 11 份的依赖都落在
ADR-0001…0009 与彼此之间。图是干净 DAG，因此「无环」不是「缺依赖段」造成的假象。

### Recommended ADR Implementation Order

Foundation（无依赖）：
  1. ADR-0001 内容数据格式、校验与启动门
  2. ADR-0002 持久化格式、原子写入与跨平台写锁
  3. ADR-0005 输入上下文栈语义

Depends on Foundation：
  4. ADR-0003 场景与屏幕架构（requires 0005）
  5. ADR-0004 服务组合与信号机制（requires 0001, 0002, 0003, 0005）
  6. ADR-0008 机器级设置与本地化管线（requires 0001, 0002, 0003）
  7. ADR-0006 确定性随机、种子派生与健康校验预算（requires 0001, 0004）
  8. ADR-0007 局内状态所有权与结算事务边界（requires 0001–0004）
  9. ADR-0009 音频播放、池化与暂停语义（requires 0001, 0003, 0004, 0006, 0008）

Core / Feature（新 11 份，按 `ORDER` 的实际位置）：
  10. ADR-0010 菜品数值与整数契约（requires 0001, 0004）
  11. ADR-0011 回合预算（requires 0004, 0007）
  12. ADR-0012 胃容量账本（requires 0004, 0007）
  13. ADR-0014 腻度与口味状态机（requires 0004, 0010）
  14. ADR-0013 局内动作管线与结束判定（requires 0004, 0007, 0010–0012）
  15. ADR-0015 结算纯函数（requires 0004, 0007, 0010, 0012, 0014）
  16. ADR-0019 HUD 数据绑定（requires 0003, 0005–0008, 0010, 0012, 0014）
  17. ADR-0016 局外升级与钱包（requires 0001, 0002, 0004, 0007, 0015）
  18. ADR-0017 解锁与档口购买（requires 0001, 0002, 0004, 0016）
  19. ADR-0018 纪录、里程碑与生涯（requires 0001, 0002, 0004, 0007, 0015, 0016）
  20. ADR-0020 首局引导（requires 0002–0007, 0010, 0014–0016, 0018, 0019）

⚠️ 拓扑序只解决「文档间谁先实现」，不解决「是否已决」：11 份新 ADR 仍是 `Proposed`，
116 条 `Verification Required` 全部未实测。

### GDD Revision Flags

上一版登记 17 条，15 条已落地；下面的 GDD 仍与 Accepted ADR 或新 ADR 的裁决不一致。
它们都还是 `Approved`，实现会以 ADR 为准，但读者会被旧措辞误导。

| # | GDD | 位置 | GDD 现状 | 建议改为 | 来源 |
|---|---|---|---|---|---|
| 1 | `save-system.md` | 规则 21（:47） | 崩溃遗留锁「以进程存活为主、时间戳为辅」回收 | 锁目录 + `owner.json` + 心跳，90 s 超时 / 24 h 上限；无法判定按存活 | ADR-0002 §3（2026-11-04 已登记，仍未改） |
| 2 | `dish-values.md` | 规则 11（:37） | 「`opt_value` … 不进入游戏运行时」 | 补唯一例外：`MenuWaves` 健康门的 `wave_reachable_opt`，结果不外流 | ADR-0006 §4（2026-11-04 已登记，仍未改） |
| 3 | `records-and-milestones.md` | 字段表 :82 | `best.return_ratio: float64` 持久化 | 标记为派生展示字段（由 `return_percent_x10 / 1000.0` 得出），或从持久化 schema 移除 | ADR-0018 第 1 条；ADR-0002 禁 float |
| 4 | `records-and-milestones.md` | 字段表 :85 | `best.upgrade_snapshot` 原样冻结，含 `cloying_resistance: float` | 注明存档安全编码（规范十进制字符串），只读快照再解码回 float | ADR-0018 第 2 条 |
| 5 | `records-and-milestones.md` | `RecordsApplyResult` 形状 | 只有 `status/applied/change/error_detail` | 补 `pending_events`（同步事件日志，不是延迟队列） | ADR-0018 第 3 条 |
| 6 | `records-and-milestones.md` | `completed_at` schema | 只允许 `String \| null` | 登记损坏时间戳的 `"unknown"` 哨兵 | ADR-0018 第 4 条 |
| 7 | `records-and-milestones.md` | AC 27 | `MILESTONE_CATALOGUE_INVALID` 由 `load` 返回 | 改为由 `prepare_run` 返回，或注明目录加载器先失败 | ADR-0018 第 11 条 |
| 8 | `unlocks-and-stations.md` | `effective_unlocks` 公式 :195 | `{s.id : s ∈ S_eff} ∪ (U.stations \ S_ids)` 会丢掉被门禁的已知已购档口 | 改为 `sort({s.id : s ∈ S_eff} ∪ U.stations)` | ADR-0017 第 1 条；与 AC 44 冲突 |
| 9 | `ui-flow.md` | AC 6（:561） | `is_first_run = (career.runs_settled == 0)` | 与规则 8 统一为 `first_run_eligible ∧ runs_settled == 0` | ADR-0020 第 2 条 |
| 10 | `first-run-onboarding.md` | 头部 | `Status: In Design` | 与 `systems-index.md` 的 `Approved` 同步 | ADR-0020 第 3 条 |
| 11 | `settlement-and-entry-fee.md` | 字段表 :180 | `return_ratio` 范围 `(0, +∞)` | 改为 `[0, +∞)`；空局合法为 `0.0`，零费用仍为 `null` | ADR-0015 的 GDD Sync Check |
| 12 | `in-game-hud.md` | 规则 9（:45） | 允许玩家主动按价格/饱腹排序 | 明确 MVP 不实现排序；未来只做视图置换，`display_index` 仍记录规范位置 | ADR-0019 第 6 条；与 `TR-in-game-hud-003`「不排序」冲突 |
| 13 | `first-run-onboarding.md` | 规则 5 | B2 锚点 `bar:capacity` 与非模态「固定底部」两种读法 | 确认 ADR-0020 的固定停靠读法（`bar:capacity` 只记讲解对象） | ADR-0020 第 7/8 条（设计确认） |

第 1、2 条是 2026-11-04 留下未改的；第 3–13 条是 ADR-0015…ADR-0020 写完后新暴露的。
按 `.claude/rules/design-docs.md` 的回执政策，第 3–9、11–12 条是实质性修改，回写后
对应 GDD 应退回 `In Review`；第 10、13 条是状态/口径同步。

### Engine Compatibility Issues

- **版本一致性 ✅**：20/20 份 ADR 的 `Engine Compatibility` 都写 Godot 4.7（`4.7.x` /
  patch `4.7.2`），与 `docs/engine-reference/godot/VERSION.md`、`project.yaml` 一致。
- **Post-cutoff API 一致性 ⚠️**：新 ADR 里 0010–0018 都声明 `Post-Cutoff APIs Used: None`；
  ADR-0019 / ADR-0020 依赖 4.6 双焦点、4.7 `canvas_items` / `expand` 默认值与
  `CanvasLayer.transform` 逆缩放，这两份自评 `Knowledge Risk: HIGH`，且逆缩放关系不在
  参考库中，必须实测。ADR-0016 的 `pow` / `ceil` / `round` float64 路径也有跨平台取整风险。
- **废弃 API 扫描 ✅（ADR 中 0 处实际使用）**：`TileMap`、`yield(`、`.instance()`、
  `VisibilityNotifier*`、`get_world()`、`OS.get_ticks_msec`、`width_in_percent`、
  `AccessibilityLiveMode`、`AudioStreamPolyphonic`、`PROCESS_MODE_WHEN_PAUSED`、
  `device == 0` 都只作为禁令或替代说明出现，没有作为实现调用。
- **Engine Compatibility 段缺失：无**（20/20）。模块引用（ui / input / audio）都存在。
- **引擎验证未执行（关键）**：`Installed at pin time = NOT DETERMINED`。9 份 Accepted ADR
  的 49 条 `Verification Required` + 11 份新 ADR 的 67 条，共 **116 条全部未实测**。
  本报告任何 `✅` 只表示「与参考文档不矛盾」，不表示「已在本机 4.7 上验证」。
- **引擎专项复核未运行**：本会话的可用子代理类型里没有 `godot-specialist`，Phase 5 的
  专家二次意见无法生成。记录为 **Engine validation: PARTIALLY ASSESSED — 文档级审计完成，
  专家复核与实机验证均未执行**，不把缺口读成通过。
- **E1 / E2 仍未修**：见下节。

### 引擎专项复核发现（E14–E18）

2026-11-04 报告的 E1–E13 已关闭 8 条（E3、E5–E11）；E1、E2、E4、E12、E13 仍开。
本轮新增 5 条：

| # | 严重度 | 位置 | 发现 | 建议动作 |
|---|---|---|---|---|
| E1 | HIGH | `project.godot` | 仍无 `display/window/stretch/mode` / `aspect` 键；ADR-0003 的布局契约建立在未写入仓库的默认值上 | 显式写入 `canvas_items` + `expand` |
| E2 | HIGH | `.gitignore:70-71` | `*.translation` 与 `export_presets.cfg` 仍被忽略；新克隆在 `--import` 前没有翻译产物，导出预设不可复现 | 保留忽略生成物但补 CI/克隆门禁；`export_presets.cfg` 版本化或提供模板（需用户决策） |
| E4 | MEDIUM | ADR-0008:312 / :621 | 音量写入路径与 ADR-0009 D9 冲突 | 见 Cross-ADR Conflicts 方案 1 |
| E12 | LOW | `tests/` | 只有 `framework_test.gd` 与占位文件；Godot 未安装，`tests.yml` 无法本机验证 | 在 B 机安装 4.7.2 后跑 `/test-setup` 与无头冒烟 |
| E13 | LOW | `design/gdd/systems-index.md` | 18 个系统全标 `Approved`，但 GDD Revision Flags 第 1、2 条仍未回写 | 回写这两条；回写前索引不能作为「无待修订」的依据 |
| **E14** | MEDIUM | `architecture.md` ADR Audit | ADR-0013 行写 `TR-run-flow-002..007`，但 ADR-0013 只认领 003/004，并在 Ordering Note 里明确 002/005/006/007 归 ADR-0007；ADR-0019 行写 `TR-in-game-hud-001..006` + `TR-run-flow-007`，但表内只认领 001/003/005/006，002/004 归 ADR-0003/0005，007 只在正文提及 | 按各 ADR 的 `## GDD Requirements Addressed` 逐行修正；覆盖率数字 56/4/44 也已被本轮 57/46/1/0 取代 |
| **E15** | MEDIUM | `architecture.md` API Boundaries | 仍只有 12/18 个系统有契约；**Dish Values、Unlocks & Stations、In-Game HUD、UI Flow、Audio Feedback、First-Run Onboarding** 六项缺失 | 用 ADR-0010 / 0017 / 0019 / 0008 / 0009 / 0020 的 Key Interfaces 补齐 |
| **E16** | MEDIUM | ADR-0019 / ADR-0020 的 GDD Sync Check | 两份新 ADR 自列了 20+ 条「需要 ADR-0003/0004/0005/0007/0008/0009/0014/0016 对齐」的接线缺口，其中层 15、`run_settled` 订阅者、`action_availability()`、`GroupCount`/`UpgradeView` 是硬契约缺口 | 接受这两份 ADR 之前逐条落地；见 Cross-ADR Conflicts 2–4 |
| **E17** | MEDIUM | ADR-0007 D4 第 7 步 vs D5 | 同一份 Accepted ADR 给出两个纪录信号采纳点 | 见 Cross-ADR Conflicts 第 5 条 |
| **E18** | LOW | `production/stage.txt` / `project.yaml` | 仍写 `Concept`；`production/gate-checks/gate-pre-production-2026-11-04.md` 已判 FAIL 并记录在案 | 门禁重跑并真正通过后再推进 stage；本轮评审不自动改 stage |

### Architecture Document Coverage

- `docs/architecture/architecture.md` 存在 → Phase 6 已运行，本项**不是** NOT ASSESSED。
- **系统覆盖 ✅**：System Layer Map 列出全部 18 个已设计系统，另列 2 个 deferred；
  六条 Data Flow 与 ADR 一致；没有孤儿架构。
- **API Boundaries ⚠️ 12/18**：缺 E15 列的 6 个系统。这不是覆盖率缺口（ADR 已写），
  而是架构文档没有把新 ADR 的接口汇总进来。
- **ADR Audit ⚠️**：20 行齐全，但 E14 的链接过度认领、覆盖率数字过期；ADR-0008 行的
  `Conflicts` 已注 E4，ADR-0007 行在 0008 之前（已按编号排序）。
- **Required ADRs ✅**：1–20 全部写成；「Can defer to implementation」段的编号从 10
  重新开始（排版问题，不影响语义）。

---

### Verdict: CONCERNS

判定依据（`references/7-report.md`）：

1. **无 critical gap**：44 条 gap 全部关闭，Foundation 24/24 由 Accepted ADR 覆盖，
   Core 无 `❌`。FAIL 的第一条触发条件不成立。
2. **无阻塞性跨 ADR 冲突**：ADR-0008/0009 的写入路径是措辞级集成冲突（沿用上一轮分类）；
   ADR-0019/0020 的接线缺口是**接受这两份 Proposed ADR 的前置条件**，不是已接受架构之间的
   矛盾，因此不触发 FAIL 的第二条。
3. **覆盖率仍不能 PASS**：46 条需求只由 `Proposed` ADR 覆盖，1 条 partial。
   PASS 要求「全部由 Accepted ADR 覆盖」，所以本轮是 CONCERNS，不是 PASS。

与上一轮相比：判定从 FAIL 升到 CONCERNS，原因不是「新 ADR 写了」（写而不接受不算覆盖），
而是 **44 条 gap 归零、partial 从 4 降到 1**；剩下的 46 条只差接受动作。

### Blocking Issues (must resolve before PASS)

1. **接受 ADR-0010…ADR-0020**（逐份 `/architecture-decision accept ADR-NNNN`，仅用户或
   technical-director 可批）。这是 46 条 🟡 变成 ✅ 的唯一路径。
2. **接受 ADR-0019 / ADR-0020 之前**，先落地 Cross-ADR Conflicts 2–4 的接线：层 15、
   `run_settled` 订阅者、`run_config_assembled`、`process_priority = -50`、
   `action_availability()`、`ui_cancel` 读者、`HudHost.apply_presentation`、
   `GroupCount` / `UpgradeView` / `LoadReport.first_run_eligible`。
3. **修 ADR-0008:312 / :621 的音频写入措辞（E4）**，否则第一个音频 story 会双重衰减。
4. **关闭 `TR-run-flow-007` 的 partial**：在 ADR-0007 补 `frozen` 字段表与
   `action_availability()`，或明确把它拆到 ADR-0019 并接受。
5. **E1**：`project.godot` 写入 stretch 键。
6. **E2**：决定 `export_presets.cfg` 是否版本化，并在 CI/克隆说明里加「先 `--import`」门禁。
7. **GDD Revision Flags 13 条**：按回执政策回写，实质性条目退回 `In Review`。
8. **引擎实测**：在 B 机安装 Godot 4.7.2，跑 116 条 `Verification Required`、`tests.yml`、
   `/gate-check pre-production`。

### Required ADRs

**无缺失。** 20 份 Required ADR 全部写成；下一步不是再写 ADR，而是接受与对齐。
按依赖序，接受顺序建议与 `ORDER` 一致：**0010 → 0011 → 0012 → 0014 → 0013 → 0015 →
0019 → 0016 → 0017 → 0018 → 0020**。

---

### Pre-gate Checklist（Phase 9）

| 项 | 状态 | 说明 |
|---|---|---|
| `tests/unit/` + `tests/integration/` | ✅ | 存在；只有 `framework_test.gd` 与占位文件 |
| `.github/workflows/tests.yml` | ✅ | gdUnit4 action，Godot 4.7.2；本机无法执行 |
| `design/accessibility-requirements.md` | ✅ | 存在 |
| `design/ux/interaction-patterns.md` | ❌ | 缺失 → 先跑 `/ux-design` |
| Godot 4.7.2 可执行文件 | ❌ | 未安装 → B 机安装后跑测试与门禁 |

因此本会话**不提供** `/gate-check pre-production` 作为下一步；按 skill 的交接规则，
先补 `/ux-design` 与引擎安装，再重跑门禁。
