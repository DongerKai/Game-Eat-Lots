# Architecture Review Report

## Architecture Review Report
Date: 2026-11-17
Mode: full
Workflow: standard
Engine: Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`——A 机无编辑器）
GDDs Reviewed: 18
ADRs Reviewed: 20

### 本轮口径与新鲜度

上一份报告是 `architecture-review-2026-11-06.md`（Mode: full / Workflow: standard / Verdict: CONCERNS）。按 Phase 1a 复核其回执，以下输入在上次报告之后发生了变化：

- `docs/architecture/adr-0013-action-pipeline-end-judge.md`：§6 补入 `_no_legal_action()` 状态判据；§9 给 `skip_reason` / `end_reason` 标注取值域 `RejectCode ∪ {NONE}` 并消歧同名域。决策 1–8 未变。
- `design/gdd/*.md`：9 份 GDD 在 2026-11-17 完成事后复审并转为 `Approved`（data-config、dish-values、in-game-hud、input、menu-waves、run-flow、scene-flow、stomach-capacity、time-and-turns），其余 9 份状态未动。
- `docs/architecture/architecture.md`：更新为 103 / 1 矩阵，补齐 Audio Feedback 与 First-Run Planner 的接口边界。
- `docs/architecture/adr-0007-…`、`adr-0019-…`：本轮修复 C3（信号清单）与 C6（可用性取值域）各一行。
- `docs/architecture/control-manifest.md`：Manifest Version 升至 2026-11-17，纳入 ADR-0013 §6/§9 的新规则。

`tr-registry.yaml` 未变（104 条 active）。由于 `architecture.md` 与 ADR 均在变化集合内，按 Phase 1a 不缩小为 `single-gdd`，本轮按 `full` / `standard` 重跑。

---

### Traceability Summary

Total requirements: 104
✅ Covered（Accepted ADR）: 103
🟡 Covered（Proposed ADR）: 0
⚠️ Partial: 1
❌ Gaps: 0
❓ Not assessed: 0

| Layer | Total | ✅ Accepted | 🟡 Proposed | ⚠️ Partial | ❌ Gap |
|---|---|---|---|---|---|
| Foundation | 24 | 24 | 0 | 0 | 0 |
| Core | 21 | 20 | 0 | 1 | 0 |
| Feature | 34 | 34 | 0 | 0 | 0 |
| Presentation | 25 | 25 | 0 | 0 | 0 |
| **Total** | **104** | **103** | **0** | **1** | **0** |

口径说明：

- 20/20 份 ADR 仍为 `Accepted`，`tr-registry.yaml` 的 104 条需求文本未变，所以逐层计数与 2026-11-06 相同。
- 本轮逐条复核了变化 ADR 涉及的 TR：ADR-0013 §6/§9 只细化 `TR-run-flow-003/004` 与 `TR-in-game-hud-001/005` 的契约措辞，覆盖状态不变（仍 `✅`）。
- `bash .claude/scripts/registry.sh validate`：`duplicate / tr_unknown / path_missing / status / layout` 全为 0；79 个实体、104 条 TR、83 条架构条目一致。

### Coverage Gaps (no ADR exists)

**无。** 104 条 active 需求每条都能落到一份 Accepted ADR 的 `## GDD Requirements Addressed` 或决策条款上。本轮的 ADR-0013 增补没有引入新的未覆盖需求。

### Partial Coverage（1 条）

⚠️ **`TR-run-flow-002`** — `design/gdd/run-flow.md`（run-flow，Core 层）：
「初始化顺序：回合 → 容量 → 菜单波次 → 腻度，同一帧完成；失败 → `init_failed`，不产生 `RunResult`」。

- 失败那一半**已决定**：ADR-0007 D7 用 `run_started` 是否发出区分 `INIT` / `RUN`，`INIT` 不产生 `RunLog` / `RunResult` / 收据。
- 顺序那一半**仍无人认领**：`adr-0013:46` 写「本 ADR 不定义初始化顺序（`TR-run-flow-002`，归 ADR-0007）」；`adr-0006:43` 写「本 ADR 不规定构造顺序」；`adr-0007:518` 把该 TR 记为 `covered`，其依据只有 D7（失败/阶段），正文**没有**固定 `TimeTurns → StomachCapacity → MenuWaves → CloyingAndTaste` 的调用次序。
- 影响不变：两个实现可以合法地让 `MenuWaves.initialize` 读到或读不到 `TimeTurns` 的预算而不违反任何 Accepted ADR，首个 run-flow story 会把差异固化进测试夹具。
- Resolution（需人决定，见 C2）：在 ADR-0007 补一条固定次序决策，或新开一份 ADR 认领；`TR-run-flow-002` 随之变 `✅`。

### Cross-ADR Conflicts

#### 前一轮 5 条，逐条复核

| # | 前一轮冲突 | 现状 | 证据 |
|---|---|---|---|
| C1 | ADR-0006 §6 整数健康门 vs `menu-waves.md` / `entities.yaml` 的浮点 `0.9` | **已关闭** | `menu-waves.md` 公式 / 变量表 / AC 22 全部改为整数交叉相乘；`entities.yaml` `wave_healthy` 三个 `DEGRADED` 来源补齐；`menu-waves.md` 2026-11-17 复审 `APPROVED` |
| C2 | ADR-0013 把初始化顺序转交 ADR-0007，接收方只接一半 | **仍未关闭** | 见下方 C2；`TR-run-flow-002` 仍 partial |
| C3 | ADR-0007 D5 失败矩阵 vs `settlement_aborted` 发射清单 | **已关闭（2026-11-17）** | `adr-0007:225` 的清单补入 `INCONSISTENT_TRANSACTION_STATE`，与失败矩阵 `:210`、`AbortCode` 枚举、`ui-flow.md` 规则 10 三方一致 |
| C4 | `architecture.md:124` 的 `set_bus_volume` vs ADR-0009 D9 | **已关闭** | `architecture.md` 的 Audio Feedback 行改为 `request(event)` / `play(cue_id)` / `shutdown` / `cue_log` / `duck_gain_db`，删除被禁止的总线写接口 |
| C5 | `architecture.md:125` 的引导接口 vs ADR-0020 §2.3 单一来源 | **已关闭** | `architecture.md` 补入 `First-Run Planner` 行（`refresh` / `plan` / `FirstRunRunPlan`），Onboarding 行改为 `configure(plan)` / `onboarding_finished` / `current_beat` |

#### C2 — ADR-0013 把初始化顺序「转交」给 ADR-0007，而 ADR-0007 只接了一半

Type: Ownership / Integration
Severity: **MEDIUM**（与 `TR-run-flow-002` 的 partial 同源）

`adr-0013:46` 说初始化顺序归 ADR-0007；`adr-0007:518` 接受该 TR 为 `covered`，但 D7 只规定失败阶段与产物。转交方不写、接收方没写。
Resolution：在 ADR-0007 补一条决策（四个 `initialize()` 的固定次序 + 同帧约束），或新开一份 ADR 认领。这是进入 `/create-epics` 前唯一需要人拍板的结构问题。

#### C6 — ADR-0013 §9 的取值域 vs ADR-0019 的消费面枚举

Type: Contract
Severity: **MEDIUM**（本轮发现并已关闭）

ADR-0013 §9（2026-11-17）把 `skip_reason` / `end_reason` 定为一个取值域 `RejectCode ∪ {NONE}`，并声明可用性 `end_reason` 与快照的本局结束原因不同域。
ADR-0019 的消费面形状却自列了第三个枚举：`skip_reason: NONE / NO_TURNS_ZERO / NO_TURNS_ONE / NO_EATABLE`、`end_reason: NONE / NO_EATABLE / NOT_FINAL_WAVE`——生产者与消费者对同一个 `int` 字段给出两套不兼容的值集，实现无法同时满足。
Resolution（2026-11-17）：ADR-0019 该形状改为「取值域见 ADR-0013 §9」，不再自列枚举；生产者仍是 ADR-0013。改动只对齐消费面注释，不改变任何行为。

#### 本轮新 ADR-0013 增补的兼容性核对

- **vs ADR-0011（回合预算）**：`_no_legal_action()` 只读 `get_turns_remaining()`，不扣回合、不裁决结束，与 ADR-0011「耗尽只报告、不裁决」一致。
- **vs ADR-0012（容量账本）**：容量满仍只是派生读数，不产生结束原因；唯一的结束路径是 `_no_legal_action()` 三条件齐备，原因是 `NO_EATABLE_DISH`，与 ADR-0012 的首条禁止项一致。
- **vs ADR-0019（HUD 只读面）**：HUD 只格式化 `*_reason`，不复制 `turns_remaining ≥ 2` / `is_final_wave` 判定；C6 修正后两边取值域一致。
- **vs ADR-0005（输入）**：`_no_legal_action()` 是状态判据，不新增动作、不改变 `submit_action` 的输入契约。

### ADR Dependency Order

`bash .claude/scripts/adr-dep-graph.sh` 输出（原样引用，尾部）：

```
ADRS: 20
  adr-0020 -> adr-0014
  adr-0020 -> adr-0015
  adr-0020 -> adr-0016
  adr-0020 -> adr-0018
  adr-0020 -> adr-0019
ORDER: adr-0001, adr-0002, adr-0005, adr-0003, adr-0004, adr-0008, adr-0006, adr-0007, adr-0010, adr-0009, adr-0011, adr-0012, adr-0014, adr-0013, adr-0015, adr-0019, adr-0016, adr-0017, adr-0018, adr-0020
```

- `NO_DEPS_SECTION:` 无、`UNFILLED_DEPS:` 无、无 `CYCLE`、无 `BLOCKED_BY_CYCLE`。20 份 ADR、72 条边，干净 DAG。
- 20/20 都是 `Accepted`，实现顺序就是 `ORDER`：0001 → 0002 → 0005 → 0003 → 0004 → 0008 → 0006 → 0007 → 0010 → 0009 → 0011 → 0012 → 0014 → 0013 → 0015 → 0019 → 0016 → 0017 → 0018 → 0020。

### GDD Revision Flags

**无新增。** 本轮 9 份 GDD 的修改只关闭 lean 复审 finding，未引入与引擎行为冲突的新假设；ADR-0013 增补为状态判据，不依赖任何引擎 API。

### Engine Compatibility Issues

**Engine validation: NOT ASSESSED** —— A 机没有安装 Godot，Phase 5 的引擎兼容核对无法执行；本项不得按「未发现」结案。

- 20 份 ADR 的 `Engine Compatibility` 段仍为文档自评；`Installed at pin time = NOT DETERMINED`。
- 自评 `Knowledge Risk: HIGH` 的 6 份仍需在装有 Godot 4.7.2 的机器上逐条验证：ADR-0003（4.6 双焦点 + 4.7 拉伸默认值）、ADR-0004、ADR-0007、ADR-0013（`RefCounted` 投影与信号顺序）、ADR-0019（`CanvasLayer` 逆缩放、焦点陷阱）、ADR-0020（输入阻断顺序）。
- `/gate-check` 读取的 deprecated-API 审计也建立在这份未验证的自评上。

### Architecture Document Coverage

- `architecture.md` 的矩阵数字已与上一轮报告一致（103 covered / 1 partial / 0 gaps），上一轮记的 `:10` / `:452` / Appendix B 三处陈旧写法已消除。
- C4（`set_bus_volume`）与 C5（缺 `FirstRunPlanner`）已修复；Audio Feedback、First-Run Onboarding、First-Run Planner 三行的接口边界与 ADR-0009 D9 / ADR-0020 §2.3 一致。
- 未发现新的缺失系统或 orphaned architecture。

---

### Verdict: CONCERNS

1 条 partial（`TR-run-flow-002` 的初始化顺序半边）+ 引擎验证 0 项执行。没有 gap，没有未解决的 ADR 对 ADR 冲突（C6 已在本轮关闭），因此不是 FAIL；但引擎能力没有被检查、初始化顺序仍无主，因此不是 PASS。

与 2026-11-06 相比：C1 / C3 / C4 / C5 关闭，新增并关闭 C6；`⚠️` 仍是 1，仍是同一条 `TR-run-flow-002`。判定维持 CONCERNS，原因从「1 条 partial + 引擎未验证」变为「同一条 partial + 引擎未验证」，其余冲突已清零。

### Blocking Issues（到 PASS 之前必须解决）

1. **引擎验证未执行**——在装有 Godot 4.7.2 的机器上跑完 116 条 `Verification Required` 项与 `/gate-check`；本项不是文档可关闭的。
2. **C2 未决**——初始化顺序归谁；关闭后 `TR-run-flow-002` 变 `✅`。

### Required ADRs

**无需新建 ADR**，除非 C2 选择「新开一份」而不是修订 ADR-0007。

### Handoff — 需要人决定的 2 件事（autonomous 模式：本轮不代为决定）

1. **初始化顺序归谁？** 在 ADR-0007 补一条决策（同时把 `TR-run-flow-002` 变 `✅`、关闭 C2），还是新开一份 ADR？推荐前者：ADR-0007 已经拥有 `INIT` 阶段与失败语义，补次序即可自洽。
2. **B 机装 Godot 4.7.2** 并执行引擎验证与 `/gate-check`；在此之前架构门不可能 PASS。

---

Reviewed-Content-Hash: docs/architecture/adr-0001-content-data-format.md d852e625024758f9850044178f407e4afd4139c8
Reviewed-Content-Hash: docs/architecture/adr-0002-persistence-format.md ed6c768fc306cdb6fff375868e69241f12969fe2
Reviewed-Content-Hash: docs/architecture/adr-0003-scene-screen-architecture.md 3847588766a746042167ef131a23bb79765ec74f
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 30b02733b0ec60c3e18d0acf82d8fd5b740909b9
Reviewed-Content-Hash: docs/architecture/adr-0008-settings-localization.md 8b6fa0ad6e6c2482e291b25fae54ceae6d16ef85
Reviewed-Content-Hash: docs/architecture/adr-0009-audio-playback-pause.md a4f849c038b2a3244e98a7ed91fb4604200efe44
Reviewed-Content-Hash: docs/architecture/adr-0010-dish-values-numeric-contract.md 4d6fba36a78f3937ac5ba678e7c9a1809f206172
Reviewed-Content-Hash: docs/architecture/adr-0011-turn-budget-exhaustion.md c8f08fd667b22c8ef334f23bb3f17cc936846990
Reviewed-Content-Hash: docs/architecture/adr-0012-stomach-capacity-ledger.md c2bef2e743e052fe85feb7b145de41591273cedf
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: docs/architecture/adr-0014-cloying-taste-state-machine.md 65b2a675e3a12378e41a021949a4aa1342e15bee
Reviewed-Content-Hash: docs/architecture/adr-0015-settlement-pure-functions.md cecf7e42a64c4edf19bdb3a50fc7b8029ca743eb
Reviewed-Content-Hash: docs/architecture/adr-0016-meta-upgrades-wallet.md 706e35071b31be2f0bb46ec426d40ba95ce430b4
Reviewed-Content-Hash: docs/architecture/adr-0017-unlocks-station-purchase.md c362b96a4151622e13f12a67ea47e450c49c6142
Reviewed-Content-Hash: docs/architecture/adr-0018-records-milestones-career.md 51d04732502434867fe649ab73a8c1b335a18847
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 492f84e663be2d81c22127d40ef68fd096dcb3a4
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 5252deab0b23680664d07fe7ce1c7255e89240ed
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/data-config.md e4d2ee1de1f4c9affd59a3993adb89056d5e4fa9
Reviewed-Content-Hash: design/gdd/dish-values.md 3eac1141e6c1209b59cc5e1ceeaa5c9ce85bc186
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/gdd-cross-review-2026-11-04.md c6efa8393fa2f8c10b80d5964b75aa9793f7d042
Reviewed-Content-Hash: design/gdd/in-game-hud.md 6d7a4700f0f8b718eeb3bf0cbe9db7993df1f43d
Reviewed-Content-Hash: design/gdd/input.md 423ac2b02d68a816905cc5060da7541aa726980c
Reviewed-Content-Hash: design/gdd/menu-waves.md af47aef125e14677602eeab6eed5ae98132c6113
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 213f8dbe40b44748fc32857680f09991fff9d612
Reviewed-Content-Hash: design/gdd/run-flow.md da67f644d65824cb6ae98081cea03f69f0e77a0d
Reviewed-Content-Hash: design/gdd/save-system.md a7d54e3ba4d7f8a23be1d28a6c165bb422648abd
Reviewed-Content-Hash: design/gdd/scene-flow.md 2439af499a5bdd48e2623ed7abe998d20c8b397c
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 82c3ea703a3c2d1e2b6bd0a8f00b4f87569cc4a8
Reviewed-Content-Hash: design/gdd/systems-index.md ccecd7af3ef6993252cb6901db7435d0985c0953
Reviewed-Content-Hash: design/gdd/time-and-turns.md ce54e5bd984a1c7a2bbb973bf124bd998563f5b4
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 06a670b544d8998eef4a8219cc9fa9d5a6b740b9
Reviewed-Content-Hash: docs/architecture/tr-registry.yaml b26b2e0ff92eec3bda1b34779ed6dd850f0b4941
Reviewed-Content-Hash: docs/architecture/architecture.md 2e142bafcd81f67d8f2be533db8186bef451a037
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
