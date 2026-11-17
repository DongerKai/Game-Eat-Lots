## Architecture Review Report

Date: 2026-11-06
Mode: full
Workflow: standard
Engine: Godot 4.7（pin 4.7.2；本机未安装编辑器 —— **本轮执行引擎检查 0 项**：没有 import、没有构建、没有跑任何测试，也不对任何运行时语义作「已验证」结论）
GDDs Reviewed: 21 个文件（`design/gdd/*.md` 全部：18 份系统 GDD + `game-concept.md` + `systems-index.md` + `gdd-cross-review-2026-11-04.md`；需求正文以 `tr-registry.yaml` 的 104 条为源）
ADRs Reviewed: 20（ADR-0001 … ADR-0020；逐份读 `## Status`，20/20 为 `Accepted`）

Reviewed-Content-Hash: docs/architecture/adr-0001-content-data-format.md d852e625024758f9850044178f407e4afd4139c8
Reviewed-Content-Hash: docs/architecture/adr-0002-persistence-format.md ed6c768fc306cdb6fff375868e69241f12969fe2
Reviewed-Content-Hash: docs/architecture/adr-0003-scene-screen-architecture.md 3847588766a746042167ef131a23bb79765ec74f
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0008-settings-localization.md 8b6fa0ad6e6c2482e291b25fae54ceae6d16ef85
Reviewed-Content-Hash: docs/architecture/adr-0009-audio-playback-pause.md a4f849c038b2a3244e98a7ed91fb4604200efe44
Reviewed-Content-Hash: docs/architecture/adr-0010-dish-values-numeric-contract.md 4d6fba36a78f3937ac5ba678e7c9a1809f206172
Reviewed-Content-Hash: docs/architecture/adr-0011-turn-budget-exhaustion.md c8f08fd667b22c8ef334f23bb3f17cc936846990
Reviewed-Content-Hash: docs/architecture/adr-0012-stomach-capacity-ledger.md c2bef2e743e052fe85feb7b145de41591273cedf
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 401cac388aed858afdc60a3a456e086bfc7310f0
Reviewed-Content-Hash: docs/architecture/adr-0014-cloying-taste-state-machine.md 65b2a675e3a12378e41a021949a4aa1342e15bee
Reviewed-Content-Hash: docs/architecture/adr-0015-settlement-pure-functions.md cecf7e42a64c4edf19bdb3a50fc7b8029ca743eb
Reviewed-Content-Hash: docs/architecture/adr-0016-meta-upgrades-wallet.md 706e35071b31be2f0bb46ec426d40ba95ce430b4
Reviewed-Content-Hash: docs/architecture/adr-0017-unlocks-station-purchase.md c362b96a4151622e13f12a67ea47e450c49c6142
Reviewed-Content-Hash: docs/architecture/adr-0018-records-milestones-career.md 51d04732502434867fe649ab73a8c1b335a18847
Reviewed-Content-Hash: docs/architecture/adr-0019-in-game-hud-binding-layout.md 23304c8212ab59cede6f6f3d81cd40cea079cde5
Reviewed-Content-Hash: docs/architecture/adr-0020-first-run-onboarding-beat-machine.md 5252deab0b23680664d07fe7ce1c7255e89240ed
Reviewed-Content-Hash: design/gdd/audio-feedback.md b02b3fa05db215dff9f4882849161e7651506ef4
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md e6aa08b28eb7739e9a5c342f358f9f55b5292bde
Reviewed-Content-Hash: design/gdd/data-config.md cd42141a823868fc47650210cb108c0ae69259a0
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/gdd-cross-review-2026-11-04.md c6efa8393fa2f8c10b80d5964b75aa9793f7d042
Reviewed-Content-Hash: design/gdd/in-game-hud.md fee7199048c3a124fcab2eccae8b88f93d1a69c0
Reviewed-Content-Hash: design/gdd/input.md 27c9d885eb2ba1f8d4b3df72ff6640fed0a5dd36
Reviewed-Content-Hash: design/gdd/menu-waves.md 4c2da744c1c4b19bc861bd0c09509d6deecbd6b7
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 3fa3e55b62e80ebcb1ed150a57577416f639faf9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 213f8dbe40b44748fc32857680f09991fff9d612
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/save-system.md a7d54e3ba4d7f8a23be1d28a6c165bb422648abd
Reviewed-Content-Hash: design/gdd/scene-flow.md 07753accfc002be7257cbcdd17323c73c2b6845a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 06a670b544d8998eef4a8219cc9fa9d5a6b740b9
Reviewed-Content-Hash: docs/architecture/tr-registry.yaml b26b2e0ff92eec3bda1b34779ed6dd850f0b4941
Reviewed-Content-Hash: docs/architecture/architecture.md 414280510c46e6d11a51896c235798c6969793f3
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

### 本轮口径与新鲜度

- 新鲜度检查（Phase 1a，`bash .claude/scripts/review-receipts.sh check docs/architecture/architecture-review-2026-11-05.md …`）
  的输出：`1 RECEIPT`、**26 `CHANGED`**（17 份 ADR：0002、0003、0004、0005、0007、0008、0010—0020；
  8 份 GDD：`dish-values`、`first-run-onboarding`、`in-game-hud`、`records-and-milestones`、`save-system`、
  `systems-index`、`ui-flow`、`unlocks-and-stations`；外加 `architecture.md`）、**41 `UNCHANGED`**、0 `NEW`、0 `GONE`。
  上一份报告的收据不覆盖当前树，因此本轮是合法重跑，不是增量核对。
- 前两份报告 `architecture-review-2026-11-04.md`（FAIL，56/4/44）、`architecture-review-2026-11-05.md`（CONCERNS，57/46/1/0）
  是日期化的历史产物，**未改动**；本报告是新的第 3 份。
- 本轮对前一轮登记的 5 条跨 ADR 冲突逐条复核，**5 条全部已闭合**（证据见 `Cross-ADR Conflicts` 的「已关闭」表），
  但复核过程中发现 **1 条新的实质性冲突（C1）**，它不在上一轮的清单里。

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

- 只有 **Accepted** ADR 的覆盖才记 `✅`（`references/3-traceability.md`）；20/20 份 ADR 都是 `Accepted`，
  所以本轮 `🟡` 为 0 —— 这是与 2026-11-05（46 条 `🟡`）的主要差别，原因不是新写了 ADR，而是那 11 份被接受了。
- 逐条重判，不继承旧结论：`requirements-traceability.md` 上一版写 104 / 0 / 0 / 0，本轮实测 **103 / 1 / 0 / 0**。
  被改判的是 `TR-run-flow-002`（见下），不是新找出的 gap。
- 逐层的是「TR 所属系统的层」，与 `architecture.md` 的 System Layer Map 一致（Save → Foundation；首局引导 → Presentation）。

### Coverage Gaps (no ADR exists)

**无。** 104 条 active 需求每条都能落到一份 Accepted ADR 的 `## GDD Requirements Addressed` 或决策条款上。

我逐条核对了 2026-11-04 的 44 条 gap 是否真的被 ADR-0010…ADR-0020 关闭，而不是只相信矩阵上的数字：
44 条全部有主（`TR-dish-values-*`→ADR-0010，`TR-time-and-turns-*`→0011，`TR-stomach-capacity-*`→0012，
`TR-run-flow-003/004`→0013，`TR-cloying-and-taste-*`→0014，`TR-settlement-002/003/005`→0015，
`TR-meta-upgrades-*`→0016，`TR-unlocks-*`→0017，`TR-records-*`→0018，`TR-in-game-hud-*`→0019，
`TR-first-run-onboarding-*`→0020）。**结论：44 条 gap 关闭属实。**

### Partial Coverage（1 条）

⚠️ **`TR-run-flow-002`** — `design/gdd/run-flow.md`（run-flow，Core 层）：
「初始化顺序：回合 → 容量 → 菜单波次 → 腻度，同一帧完成；失败 → `init_failed`，不产生 `RunResult`」。

- 失败那一半**已决定**：ADR-0007 D7 用 `run_started` 是否发出区分 `INIT` / `RUN`，`INIT` 不产生
  `RunLog` / `RunResult` / 收据，`init_failed` 降为兼容位。
- 顺序那一半**无人认领**：`adr-0013-action-pipeline-end-judge.md:46` 明确写「本 ADR 不定义初始化顺序
  （`TR-run-flow-002`，归 ADR-0007）」；`adr-0006-deterministic-rng.md:43` 写「本 ADR 不规定构造顺序」；
  而 ADR-0007 在 `:518` 把该 TR 记为 `covered`，其依据只有 D7（失败/阶段），正文没有任何一句固定
  `TimeTurns → StomachCapacity → MenuWaves → CloyingAndTaste` 的调用次序。
- 判 partial 而不是 covered，理由与用户口径一致：ADR 只是「指向」这条 TR 的一半。
- 影响：两个实现可以合法地让 `MenuWaves.initialize` 读到或读不到 `TimeTurns` 的预算而不违反任何 Accepted ADR，
  首个 run-flow story 就会把差异固化进测试夹具。

### Cross-ADR Conflicts

#### 已关闭（前一轮 5 条，逐条复核）

| # | 前一轮冲突 | 现状 | 证据 |
|---|---|---|---|
| 1 | ADR-0008 音量写入路径（E4） | **已关闭** | `adr-0008:128` / `:312` / `:621` 全部改为「保存 gain、不写总线」；`adr-0009:316` 成为唯一口径 |
| 2 | ADR-0020 vs ADR-0003/0004/0005/0009 引导宿主接线 | **已关闭** | `adr-0003:100,112`（layer 15）、`adr-0004:215,232,234`（`run_settled` 订阅者、`run_config_assembled`）、`adr-0005:178`（`-50` 与 `-100 < -50 < 0` 断言） |
| 3 | ADR-0019 vs ADR-0007/0005/0008/0004（HUD 只读接口） | **已关闭** | `adr-0007:108,111`（`frozen` + `action_availability()`）、`adr-0019:859,864`（同步检查逐行「已对齐」） |
| 4 | ADR-0020 vs ADR-0002/0014/0016（接口形状） | **已关闭** | `adr-0014:146-152`（`GroupCount` 字段）、`adr-0016:113-119`（`UpgradeView` 字段）、`adr-0002:271-273`（`LoadReport.first_run_eligible`） |
| 5 | ADR-0007 D4 第 7 步 vs D5（纪录信号采纳点） | **已关闭** | `adr-0007:193` 第 7 步改写为「本步不补发任何信号」，与 D5 `:226` 同源 |

#### C1 — ADR-0006 §6（整数健康门）vs `menu-waves.md` / `entities.yaml`（浮点 `0.9`）

Type: Determinism / Contract
Severity: **BLOCKING for the menu-waves implementation story**；**不阻塞架构判定**（无设计结果变化、无 ADR 对 ADR 的矛盾）

ADR-0006 claims（Accepted）：健康门阈值**必须**用整数交叉相乘。
`adr-0006:314`：「**不得**写成 `reachable < 0.9 * baseline`：float 在 `93.6` 附近的行为依赖舍入，且违反原则 5」；
`adr-0006:528`（Implementation Guideline 1）：「任何 `float` 变量、常量或比较出现在这条路径上都视为违反原则 5；
`0.9` 必须写成 `9/10` 交叉相乘」；`adr-0006` Validation #12 是一条 **CI 静态扫描：`src/feature/menu_waves/`
中不得出现 `float`**。同一口径已落到 `control-manifest.md:108`（`reachable * 10 < baseline * 9`）与
`entities.yaml:458`（`baseline_opt = opt_value(actual pool, C0, T)`，`10 * reachable >= 9 * baseline`）。

仍在用浮点的位置（**必须改，逐条给出行**）：

| 文件:行 | 现状 | 应改为 |
|---|---|---|
| `design/registry/entities.yaml:611` | `expression: "… wave_reachable_opt < 0.9 * opt_value(pool, C0, T) …"` | 与同文件 `:458` 一致的整数形式：`10 * wave_reachable_opt < 9 * baseline_opt`（或 `≥` 反向表述） |
| `design/registry/entities.yaml:612` | `notes: "… MVP lower bound 94 (0.9 x 104) …"` | `lower bound = ceil(9 * 104 / 10) = 94`（去掉浮点乘式） |
| `design/gdd/menu-waves.md:190` | 公式 `∨ (可达检查已执行 ∧ wave_reachable_opt < 0.9 × opt_value(pool, C0, T))` | `10 × wave_reachable_opt < 9 × opt_value(pool, C0, T)` |
| `design/gdd/menu-waves.md:200` | 变量表「下界 … `0.9 × opt_value(pool, C0, T)`」 | `ceil(9 × opt_value / 10)`（整数定义） |
| `design/gdd/menu-waves.md:343` | AC 22「低于 `0.9 × opt_value(pool, C0, T)` 的候选划分」 | 同上整数式（这是验收条件，会直接变成测试代码） |

判定细节（避免误报）：

- **行为等价已证明**：对 `baseline ≤ 2000`、`reachable ≤ baseline` 的整数域穷举比较 `r < 0.9·b`（IEEE-754 double
  逐点计算）与 `10·r < 9·b`，**0 处分歧**。所以这不是「阈值算错」，改写的风险极低，也不会改变任何设计结论。
- **为什么仍算 blocking（对 story，不对本轮判定）**：ADR-0006 的验收项是一条静态扫描，禁止该目录出现 `float`。
  若 story 按 GDD 公式实现，它会**通不过 Accept ADR 自己的验收测试**；若按 ADR 实现，GDD 的公式与 AC 22 就与代码不符。
  两边都不能「都对」。
- `design/gdd/menu-waves.md:333`（AC 15「阈值 94 = ⌈0.9 × 104⌉」）与 `:35` / `:181` / `:206` / `:293` / `:353` / `:355`
  的散文提到「0.9 阈值」，是**人话描述一个比例**、且算术结果与整数式一致，**不必改**；ADR-0006 自己也用
  「`0.9 × opt_value`」作标题（`:230`）。
- `tr-registry.yaml:480`（`TR-menu-waves-005` 的需求文本）同样只是比例描述，未改；若要一并收紧，需要更新
  `revised` 日期，属设计侧变更，本轮不动。

#### C2 — ADR-0013 把初始化顺序「转交」给 ADR-0007，而 ADR-0007 只接了一半

Type: Ownership / Integration
Severity: **MEDIUM**（同一件事在 `TR-run-flow-002` 已记 partial，此处记录冲突成因）

`adr-0013:46` 说初始化顺序「归 ADR-0007」；`adr-0007:518` 接受该 TR 为 `covered`，但 D7 只规定
失败阶段与产物。**这是一条悬空的委派**：转交方不写，接收方没写。
Resolution：在 ADR-0007 补一条决策（四个 `initialize()` 的固定次序 + 同帧约束），或新开一份 ADR 认领；
`TR-run-flow-002` 随之变 `✅`。

#### C3 — ADR-0007 D5 失败矩阵 vs `settlement_aborted` 发射清单（`INCONSISTENT_TRANSACTION_STATE` 丢失）

Type: Signal contract（同一份 ADR 内部）
Severity: **MEDIUM**（非阻塞，结算 story 之前必修）

`adr-0007:210` 的失败矩阵把「`records` 返回 `ALREADY_APPLIED` 但钱包不是」判为 `ABORTED` +
`INCONSISTENT_TRANSACTION_STATE`；`adr-0007:336` 的 `AbortCode` 枚举也含该值；`design/gdd/ui-flow.md:63`（规则 10）
已经写「四者都由 `UIRoot` 发一次 `settlement_aborted(run_id, code)`」。但 `adr-0007:225` 写
「`settlement_aborted` … 只在 `records` 致命、重入、`BUSY`、`RUN_ID_REUSED` 这几类」——**漏了这一类**，
于是同一份 Accepted ADR 的信号清单与自己的失败矩阵、以及 GDD 规则 10 三方不一致。
按字面实现一个白名单，这一格会得到 `ABORTED` 收据却**不发**中止事件，音效不取消结果 cue。
Resolution：把 `:225` 的清单补上 `INCONSISTENT_TRANSACTION_STATE`。`design/gdd/reviews/ui-flow-review-log.md:1329`
已把同一处记为「the ADR's own signal bullet and its matrix disagree」，本轮把它从「Recommended」升为「story 前必修」。

#### C4 — `architecture.md:124` 的 `set_bus_volume` vs ADR-0009 D9

Type: Integration
Severity: **MEDIUM**

`docs/architecture/architecture.md:124` 把 `set_bus_volume` 列为 Audio Feedback 的关键接口。
ADR-0009 D9 / D2 写死「`Master / SFX / UI` 三条总线恒为 0 dB、**永不写入**」，唯一允许的总线写入是
`Music` 的闪避包络（`adr-0009:327`，`AudioServer.set_bus_volume_db(Music, …)`）。
按 `architecture.md` 实现会给出一条 ADR 禁止的公开 API，E4 修的正是这条路径，文档侧漏改。
Resolution：把该行改为 `play(cue_id)`、`shutdown`、`get_cue_log` 之类（或 `duck_music(db)`），删掉 `set_bus_volume`。

#### C5 — `architecture.md:125` 的引导接口 vs ADR-0020 §2.3（单一来源）

Type: Integration / Pattern
Severity: **MEDIUM**

`architecture.md:125` 把 `get_first_run_preset` 与 `onboarding_finished` 列为 First-Run Onboarding 的
「Key Interfaces」，`architecture.md:399` 只出现 `configure(plan: FirstRunRunPlan)`，**全文没有 `FirstRunPlanner`**。
ADR-0020 §2.3（`adr-0020:902`）要求 `FirstRunPlanner` 是 `is_first_run` / `menu_entry_fee` / 预置的**唯一**生产者，
并**禁止** UI 直接调用 `get_first_run_preset()`（`ui-flow.md` 规则 8/AC 6/AC 47 已照此改写）。
`architecture.md` 目前给的是被 ADR-0020 明确废弃的那条读法。
Resolution：在 API Boundaries 补 `class FirstRunPlanner`（`plan()` / `refresh()` / `FirstRunRunPlan` 字段），
Module Ownership 行改为 `plan()`, `FirstRunRunPlan`, `onboarding_finished`。

### ADR Dependency Order

`bash .claude/scripts/adr-dep-graph.sh` 输出（原样引用，尾部）：

```
ADRS: 20
EDGES:
  …（72 条，略；完整列表见脚本输出）
  adr-0020 -> adr-0002
  adr-0020 -> adr-0003
  adr-0020 -> adr-0004
  adr-0020 -> adr-0005
  adr-0020 -> adr-0006
  adr-0020 -> adr-0007
  adr-0020 -> adr-0010
  adr-0020 -> adr-0014
  adr-0020 -> adr-0015
  adr-0020 -> adr-0016
  adr-0020 -> adr-0018
  adr-0020 -> adr-0019
ORDER: adr-0001, adr-0002, adr-0005, adr-0003, adr-0004, adr-0008, adr-0006, adr-0007, adr-0010, adr-0009, adr-0011, adr-0012, adr-0014, adr-0013, adr-0015, adr-0019, adr-0016, adr-0017, adr-0018, adr-0020
```

- `NO_DEPS_SECTION:` 无、`UNFILLED_DEPS:` 无、无 `CYCLE`、无 `BLOCKED_BY_CYCLE`。**20 份 ADR、72 条边，干净 DAG。**
- 因为 20/20 都是 `Accepted`，「Accepted 依赖 Proposed」这一条已经不存在；上一轮那句「拓扑序不等于已决」本轮不成立。
- 实现顺序就是 `ORDER`（Foundation → Core/Feature → Presentation）：0001 → 0002 → 0005 → 0003 → 0004 → 0008 →
  0006 → 0007 → 0010 → 0009 → 0011 → 0012 → 0014 → 0013 → 0015 → 0019 → 0016 → 0017 → 0018 → 0020。

`bash .claude/scripts/registry.sh validate` 输出（原样引用）：

```
registry	entities	design/registry/entities.yaml	79
registry	tr	docs/architecture/tr-registry.yaml	104
registry	architecture	docs/registry/architecture.yaml	83
checked	production/epics	0
duplicate	0
tr_unknown	0
path_missing	0
status	0
layout	0
```

`duplicate` / `tr_unknown` / `path_missing` / `status` / `layout` 全为 0；79 个实体、104 条 TR、83 条架构条目一致。

### GDD Revision Flags

前一轮登记 13 条，本轮**逐条验证它们各自点名的文件均已回写**（`save-system.md:47` 锁心跳、`dish-values.md:37`
`opt_value` 运行期例外、`records-and-milestones.md:73/82/85/132/141`、`unlocks-and-stations.md:195`、`ui-flow.md`
规则 8 与 AC 6、`first-run-onboarding.md` 头部状态、`settlement-and-entry-fee.md:82`、`in-game-hud.md:45`、
`first-run-onboarding.md` 规则 5 锚点）。但同一类问题在**没有被那 13 条点名的 10 份 GDD 上仍然存在**：
它们的头部状态仍写 `In Design` / `In Review`，而 `systems-index.md` 第 58–75 行把 18 个系统全部标为 `Approved`，
它们自己的 `design/gdd/reviews/*-review-log.md` 末次判词也都是 `APPROVED`（日期早于 2026-11-06）。

**任务前提需要修正**：「18 份系统 GDD 全部 `Approved`（2026-11-06 复审）」只对 **8 份**成立——
`dish-values`、`first-run-onboarding`、`in-game-hud`、`records-and-milestones`、`save-system`、
`settlement-and-entry-fee`、`ui-flow`、`unlocks-and-stations`（头部与 review log 都有 2026-11-06 记录）。
另外 **10 份**的头部是旧状态：`audio-feedback`、`cloying-and-taste`、`data-config`、`input`、`menu-waves`、
`meta-upgrades`、`run-flow`、`stomach-capacity`、`time-and-turns` 写 `In Design`，`scene-flow` 写
`In Review — 待重新评审`。这不影响需求正文与 ADR 的对应关系（本轮覆盖矩阵读的是需求文本，不是状态行），
但索引的 `Approved` 在这 10 份上没有本文件级的依据，story 侧的 `/story-readiness` 会读到两套状态。

| # | 文件 | 位置 | 现状 | 建议改为 | 来源 |
|---|---|---|---|---|---|
| 1 | `design/gdd/menu-waves.md` | 规则 9（:35）、公式 `:190`、变量表 `:200`、AC 22 `:343` | 健康门下界写成 `0.9 × opt_value(...)`（浮点比较） | 机器可读处一律改为整数交叉相乘 `10 × reachable ≥ 9 × baseline`；散文里的「0.9 阈值」可保留 | ADR-0006 §6 + Validation #12；C1 |
| 2 | `design/registry/entities.yaml` | `:611`（`wave_healthy.expression`）、`:612`（notes） | 同上，且与本文件 `:458` 的整数口径自相矛盾 | 与 `:458` 对齐 | ADR-0006 §6；C1 |
| 3 | 上述 **10 份** GDD | 各文件头部 `> **Status**` | 9 份 `In Design`、`scene-flow` `In Review` | 若确实已按 2026-11-06 复审通过，改为 `Approved (2026-11-06)`；否则把 `systems-index.md` 改回真实状态 | 索引 :58–75 vs 各文件头部；不属于本轮可改范围（用户约束） |

`design/gdd/systems-index.md` 按用户约束**未改**；18 份系统 GDD 的 `Status` 也**未改**（本轮只读）。
第 1、2 条是机器可读口径的实质性修改，回写后按回执政策 `menu-waves.md` 应退回 `In Review`；
第 3 条是状态同步，需要人判定「是索引写早了，还是头部写旧了」——本轮不替任何一边下结论。

### Engine Compatibility Issues

- **版本一致性 ✅**：20/20 份 ADR 的 `Engine Compatibility` 段都写 Godot 4.7（`4.7.x` / patch `4.7.2`），
  与 `docs/engine-reference/godot/VERSION.md`、`project.yaml:42`（`version: "4.7"`）一致。
  `Installed at pin time = NOT DETERMINED`，所以「版本一致」只表示文档自洽。
- **Post-cutoff API 一致性 ⚠️**：6 份 ADR 自评 `Knowledge Risk: HIGH` —— ADR-0003（4.6 双焦点 + 4.7 拉伸默认值）、
  ADR-0005（4.7 `DEVICE_ID_*`）、ADR-0006（`RandomNumberGenerator` PCG 细节）、ADR-0008（4.5 AccessKit / 4.7
  CSV plural 与 `hinting` 默认值）、ADR-0019（`CanvasLayer` 逆缩放 × `canvas_items`）、ADR-0020（层序/逆缩放坐标、
  `process_priority` 跨节点顺序、共享消费位）。`CanvasLayer` 逆缩放关系不在参考库中，必须实测。
- **废弃 API 扫描 ✅（0 处实际使用）**：`TileMap`、`yield(`、`.instance()`、`VisibilityNotifier*`、`get_world()`、
  `OS.get_ticks_msec`、`width_in_percent`、`AccessibilityLiveMode`、`AudioStreamPolyphonic`、
  `PROCESS_MODE_WHEN_PAUSED`、`device == 0` 在 `docs/architecture/` 与 `design/gdd/` 中**只作为禁令或替代说明**
  出现（如 `adr-0009:150`、`adr-0005:219`、`architecture.md:31`），没有一处是实现调用。
- **Engine Compatibility 段缺失：无**（20/20），模块文档（ui / input / audio / animation / rendering / physics /
  navigation / networking + `VERSION` / `breaking-changes` / `deprecated-apis` / `current-best-practices`）全部在库。
- **引擎验证：0 项执行（关键）** —— 本机没有 Godot：没有 `godot` 可执行文件、没有 `/Applications/Godot*.app`，
  `VERSION.md` 记 `Installed at pin time = NOT DETERMINED`，`project.yaml` 的待办也写着「godot is not on PATH here」。
  因此本轮**没有** import 项目、**没有**构建、**没有**跑 `tests.yml`，20 份 ADR 的 `Verification Required` 条目
  一条都未实测。本报告里任何 ✅ 只表示「与参考文档不矛盾」，不表示「已在 4.7 上验证」。
- **引擎专项复核未运行**：本环境的工具集里没有可委派的 `godot-specialist` 子代理，Phase 5 的专家二次意见无法生成。
  记为 **Engine validation: PARTIALLY ASSESSED — 文档级审计完成，专家复核与实机验证均未执行**，不把缺口读成通过。

| # | 严重度 | 位置 | 发现 | 建议动作 |
|---|---|---|---|---|
| E1 | — | `project.godot:13-14` | **已关闭**：`window/stretch/mode="canvas_items"`、`aspect="expand"` 已显式写入 | 无 |
| E2 | LOW | `.gitignore:71` / 仓库根 | 半关闭：`.gitignore` 已不再忽略 `export_presets.cfg`（注释声明「versioned」），但**文件本身仍不存在** | 补 `export_presets.cfg`（或模板），否则导出仍不可复现；需人决定是否版本化 |
| E3 | MEDIUM | `src/` | `src/` 只有 `.gitkeep` 与 `CLAUDE.md`，没有任何 GDScript；ADR-0005 D9、ADR-0006 Validation #12、ADR-0010 的三条静态扫描（`device == 0`、`float`、`opt_value` API 表面）在代码落地前无处执行 | 第一个 story 落地时把三条扫描接进 `tests.yml`，否则这些验收项永远不会真正跑起来 |
| E12 | LOW | `tests/unit`、`tests/integration` | 只有 `framework_test.gd`，`integration/` 为空；`tests.yml`（gdUnit4 4.7.2）无法本机验证 | B 机装 4.7.2 后跑 `/test-setup` 与无头冒烟 |
| E18 | LOW | `production/stage.txt` | 仍写 `Concept`，而 2026-11-04 的门禁已判过一轮 | 门禁重跑通过后再推进 stage；本轮不自动改 |

### Architecture Document Coverage

- `docs/architecture/architecture.md` 存在 → Phase 6 已运行，本项**不是** NOT ASSESSED。
- **系统覆盖 ✅**：System Layer Map 列全 18 个已设计系统（另列 2 个 deferred），六条 Data Flow 与 ADR 一致，无孤儿架构。
- **API Boundaries ✅ 18/18**：19 个 `class` 覆盖 18 个系统（UI Flow 有 `UIRoot` + `SettingsModel`），
  上一轮 E15 的 6 个缺口已补齐。**但两处接口与 Accepted ADR 不一致**：`architecture.md:124`（`set_bus_volume`，C4）、
  `architecture.md:125` + `:399`（`get_first_run_preset` 且缺 `FirstRunPlanner`，C5）。
- **ADR Audit ✅**：20 行齐全，`✅ Accepted — engine verification pending`，逐行 linkage 已按各 ADR 的
  `## GDD Requirements Addressed` 修正（上一轮 E14 的过度认领已消失：ADR-0013 只认 003/004，ADR-0019 只认 001/003/005/006）。
- **覆盖率数字 ⚠️**：`architecture.md:452` 与 `:10`、`:423-425` 仍写 **104 / 0 / 0 / 0**，与本轮 103 / 1 / 0 / 0 不符；
  `Appendix B` 的分层表也需随之改（Core 21 → 20 ✅ + 1 ⚠️）。**本轮不改该文件**（见下），改动清单已给。
- **分层依据 ⚠️（LOW）**：Save 是 Foundation 有明确理由（`:79`），首局引导改到 Presentation（`:93`）**没有给理由**，
  而 `systems-index.md` 把它归在 `Meta`；两套分层词汇不同，可接受，但缺一句说明会让读者以为二者冲突。

### 属于本 skill 的文件所有权（本轮实际写了什么）

| 文件 | skill 是否拥有 | 本轮动作 |
|---|---|---|
| `docs/architecture/architecture-review-2026-11-06.md` | ✅ 拥有（Phase 8 [A]） | **新建** |
| `docs/architecture/requirements-traceability.md` | ✅ 拥有（Phase 8 [A] 的 traceability index） | **改写**为 103 / 1 / 0 / 0 |
| `docs/architecture/tr-registry.yaml` | ✅ 拥有（仅在有新增/改写/废弃 TR 时） | **未改**：0 条新 ID、0 条需求文本变化 |
| `docs/consistency-failures.md` | 仅「存在时追加」 | **不存在 → 未创建、未追加**（C1 未写入 reflexion log，属预期） |
| `docs/architecture/control-manifest.md` | ❌ 不拥有（归 `/create-control-manifest`） | **未改**。它已经是 2026-11-06 版、覆盖 20 条 Accepted ADR，且 `:108` 已写整数门、`:306` 已写 `FirstRunPlanner` 单一来源 —— 与 C1/C5 的正确口径一致。**唯一需要它重生成的触发条件**：C2/C3 若要改 ADR-0007 文本，改完必须跑 `/create-control-manifest update` |
| `docs/architecture/architecture.md` | ❌ 不拥有（Phase 6 只读审查） | **未改**。必须改的 4 处：`:124` `set_bus_volume`、`:125`/`:399` 引导接口 + `FirstRunPlanner`、`:452` 覆盖率为 103/1/0/0、Appendix B 分层计数 |
| `design/gdd/*.md`、`design/registry/entities.yaml`、`design/gdd/systems-index.md` | ❌ 不拥有 | **未改**；C1 的 5 个行号与 10 份 GDD 的状态口径已列在 GDD Revision Flags |
| 前两份 dated 报告 | ❌ 永不改 | **未改** |

---

### Verdict: CONCERNS

判定依据（`references/7-report.md`）：

1. **不是 FAIL**：Foundation 24/24 由 Accepted ADR 覆盖、Core 无 `❌`、44 条 gap 归零，第一条触发条件不成立；
   也没有「阻塞性跨 ADR 冲突」——C1 是 Accepted ADR 与 GDD/注册表的冲突（对 story 阻塞，对架构判定不阻塞，
   且已用穷举证明零行为差异），C2–C5 是 Medium 的接口/内部不一致，不是两份 Accepted ADR 互相矛盾。
2. **不是 PASS**：`TR-run-flow-002` 为 partial（1 条），且 **0 项引擎检查被执行**、专家复核未运行。
   PASS 要求「全部覆盖 + 无冲突 + 引擎一致」，引擎这一项本轮**无法证明**一致，只能证明「与参考文档不矛盾」。
3. **不是 NOT ASSESSED**：ADR 可读、`## Status` 齐全、需求源存在、Phase 6 可运行、引擎已配置（只是本机没装）。

与 2026-11-05 相比：`🟡` 从 46 降到 0（11 份 ADR 被接受），`⚠️` 仍是 1，但**换了一条**（`TR-run-flow-007` 已修，
`TR-run-flow-002` 被重新判为 partial）。判定维持 CONCERNS，原因从「只差接受动作」变成「1 条 partial + 引擎未验证」。

### Blocking Issues（到 PASS 之前必须解决）

1. **C1 的 5 行浮点口径**（`entities.yaml:611,612`、`menu-waves.md:190,200,343`）——不修，menu-waves 的
   story 会同时违反 GDD 与 ADR-0006 Validation #12 中的一个。
2. **`TR-run-flow-002` 的顺序半边**——在 ADR-0007 补固定初始化次序（同时关闭 C2）。
3. **C3**：`adr-0007:225` 补 `INCONSISTENT_TRANSACTION_STATE`，结算 story 前必修。
4. **C4 / C5**：`architecture.md` 的两处接口（`set_bus_volume`、`get_first_run_preset` / 缺 `FirstRunPlanner`）。
5. **`architecture.md` 覆盖率行与 Appendix B**：104/0/0/0 → 103/1/0/0。
6. **引擎实测**：B 机装 Godot 4.7.2 后跑 20 份 ADR 的 `Verification Required`、`tests.yml`、`/gate-check pre-production`；
   `export_presets.cfg` 一并决定（E2）。

### Required ADRs

**无缺失。** 20 份 Required ADR 全部写成且全部 `Accepted`；下一步不是再写 ADR，而是上面 6 条收尾。

---

### Pre-gate Checklist（Phase 9）

| 项 | 状态 | 说明 |
|---|---|---|
| `tests/unit/` + `tests/integration/` | ✅ | 存在；`unit/` 只有 `framework_test.gd`，`integration/` 为空 |
| `.github/workflows/tests.yml` | ✅ | gdUnit4 action，`godot-version: '4.7.2'`；本机无法执行 |
| `design/accessibility-requirements.md` | ✅ | 存在 |
| `design/ux/interaction-patterns.md` | ✅ | 存在（10 份屏幕规格 + `interaction-patterns.md` 齐） |
| Godot 4.7.2 可执行文件 | ❌ | 未安装 → 0 项引擎检查被执行 |

结论：4 项 ✅、1 项 ❌。按交接规则**不提供** `/gate-check pre-production` 作为下一步，先装引擎再重跑门禁。

### Handoff — 需要人决定的 3 件事（autonomous 模式：本轮不代为决定）

1. **C1 现在回写还是等 story？** 回写要动 `menu-waves.md` 与 `entities.yaml`（5 行），按回执政策该 GDD 退回
   `In Review` 并重评；不回写则接受「GDD 公式与 ADR 验收项对不上」，直到 menu-waves story 开工。
2. **初始化顺序归谁？** 在 ADR-0007 补一条决策（同时把 `TR-run-flow-002` 变 ✅、关闭 C2），还是新开一份 ADR？
   顺带决定 C3（`adr-0007:225` 补 `INCONSISTENT_TRANSACTION_STATE`）是否同批改——改 ADR 就要跑
   `/create-control-manifest update`。
3. **引擎验证怎么排？** 在 B 机装 Godot 4.7.2 跑 20 份 ADR 的 `Verification Required` + `tests.yml` +
   `/gate-check pre-production`，并同时决定 `export_presets.cfg` 是否版本化（E2）。
   在那之前，任何「已实现 / 已验证」的说法都不成立。

附带一件低成本的事：`systems-index.md` 的 18 个 `Approved` 与 10 份 GDD 头部状态不一致（见 GDD Revision Flags 第 3 条），
建议在同一次回写里一并定口径。

**Rerun trigger**：每修完 C1 / C2 / C3 或改完 `architecture.md` 的接口与数字后，重跑 `/architecture-review full`；
引擎装上后单独跑 `/architecture-review engine`。
