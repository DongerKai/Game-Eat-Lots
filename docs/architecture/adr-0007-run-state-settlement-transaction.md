# ADR-0007: 局内状态所有权与结算事务边界（Run State Ownership & Settlement Transaction Boundary）

## Status

Accepted

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。
> 其他 agent 与任何 skill 都不能自行设置。**

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

- 用户（`modes.automation: autonomous`，由用户先行授权）
- `systems-designer` / `lead-programmer`（餐局生命周期、结算、纪录与存档事务契约）
- `technical-director` 战略审阅：跳过（`review_mode: lean`）

## Summary

`architecture.md` 把「局内状态归谁、结算事务边界在哪」登记为 **QQ-07（High）**，但三份 GDD 各自给出了答案的一半：`run-flow.md` 说 `run_id`、`RunLog` 与局内状态归餐局流程；`meta-upgrades.md` 与 `ui-flow.md` 说界面流程是「结算提交编排者」；`records-and-milestones.md` 又说编排者尚未指定。同时 `ADR-0004` 把 `run_settled` 的订阅者写成「收据编排」，却没有说清那次通知与 `Run → Menu` 交付之间谁是持久化时点。若按字面实现，会出现三种只有重写才能修掉的事故：玩家在 `Handoff` 中退出却已经写入纪录；收据屏重新调用 `settle` 得到第二份结果；钱包与纪录在同一个 `run_id` 上落成两种事实。

本 ADR 裁定四件事：

1. **局内状态只有一个所有者：`RunFlow`。** `run_id`、生命周期状态、动作序号、`running_net`、`RunLog` 与交接用的 `RunResult` 全部由它分配、组装与释放；结算仍是纯函数，纪录与局外只拥有各自的命名空间。不新增「局内状态服务」。
2. **结算事务只有一个构造者：`SettlementTransaction`。** 它是一个纯 `RefCounted`，由 `AppRoot` 构造、注入 `UIRoot`；它把 `records`、`progression` 与（未来才可能存在的）`unlocks` 合并成**一次** `SaveSystem.commit`。界面流程是唯一调用者与收据编排者，`RunFlow`、`Settlement`、`Records`、`MetaUpgrades` 都不再各自提交。
3. **提交发生在 `Run → Menu` 被接受的那一刻，不在 `run_settled`。** `run_settled` 只是「结果已就绪」的预通知；真正的 `Save.commit` 在 `UIRoot.reset_for_state(Run, Menu, RunResult)` 内、`ReceiptScreen` 挂载之前同步执行一次。这样「放弃不是结算」（`run-flow.md` 规则 16）在 `Handoff` 中仍然成立。
4. **收据只读一份不可变快照。** `SettlementTransaction.apply(result)` 返回 `ReceiptSnapshot`；`ReceiptScreen` 不调用 `Settlement.settle`、不读取 `RunLog`、不重算任何合计。故障按 `fault.phase ∈ {INIT, RUN}` 区分文案，`init_failed` 只保留为兼容布尔位，不再承担「本局是否开始过」的语义。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；本机 `Installed at pin time = NOT DETERMINED`，尚未安装编辑器） |
| **Domain** | Core（运行期状态所有权）；Feature（结算 / 局外 / 纪录）；Presentation（收据同帧） |
| **Layer** | Core / Feature |
| **Knowledge Risk** | MEDIUM — 本 ADR **不引入任何 4.5–4.7 新 API**。风险集中在本 ADR 依赖的两处既有结论：`SaveSystem.commit` 的同步性与失败词汇（ADR-0002）、`state_changed` / `run_settled` 的同步派发（ADR-0004）。`Engine.get_process_frames()` 与 `Array` / `Dictionary.duplicate(true)` 是 4.0 起的稳定 API |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`current-best-practices.md`、`breaking-changes.md`；`design/gdd/run-flow.md`、`settlement-and-entry-fee.md`、`meta-upgrades.md`、`records-and-milestones.md`、`unlocks-and-stations.md`、`save-system.md`、`ui-flow.md`、`scene-flow.md`；`docs/architecture/architecture.md`、ADR-0002、ADR-0003、ADR-0004 |
| **Post-Cutoff APIs Used** | 无。刻意不使用 `Resource.duplicate_deep()`（4.5）：收据快照只复制 `Array` / `Dictionary`，用长期存在的 `duplicate(true)` 即可；也不依赖任何 4.7 的信号顺序承诺 |
| **Verification Required** | ① `Engine.get_process_frames()` 在无头运行中与收据挂载同帧（用于断言 `receipt_open_latency_frames = 0`）；② `Dictionary.duplicate(true)` 对 `RunResult.rows` 这类嵌套数组确实深拷贝，修改副本不污染 `RunResult`；③ `SaveSystem.commit` 在主线程同步返回（ADR-0002 的验证项）；④ `run_settled` / `state_changed` 在 `emit()` 内同步调用处理器（ADR-0004 的验证项） |

> ⚠️ **引擎知识缺口**：4.5–4.7 均在 LLM 训练截止之后，且本机没有安装编辑器。
> 本 ADR 的运行时结论都建立在 ADR-0002 / ADR-0004 的同步假设上；那两条的
> `Verification Required` 未完成前，本 ADR 的事务顺序按「待验证」处理。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0002（`Save.commit` 的原子性、`CommitResult` 词汇、失败时「购买回滚 / 结算保留 dirty」的区分、所有权表中「结算事务编排」不属于 Save）；ADR-0003（`ReceiptScreen` 在 `Run → Menu` 同帧成为根屏、先于 `HudHost` 卸载）；ADR-0004（`AppRoot` 构造并注入全部服务；`state_changed` 单订户固定扇出；`run_settled` 的订阅关系；`BootSequence` 是唯一基线写者）；ADR-0001（`Settlement.settle` 与收据行要求 `DataConfig` 为 `Ready`） |
| **Enables** | 结算与入场费的实现 story；局外升级的入账 story；纪录与里程碑的 `prepare_run` story；界面流程的收据屏 story；`TR-run-flow-005` / `TR-ui-flow-004` / `TR-meta-upgrades-005` / `TR-records-002` 的验收 |
| **Blocks** | 任何「结束一局并写盘」或「绘制收据」的 story；任何依赖「同一 `run_id` 只写一次」的测试；任何故障文案 story（没有 `phase` 就没有正确文案） |
| **Ordering Note** | 本 ADR 是 `architecture.md` Required ADRs 的第 7 项，编号与之一致。ADR-0004 D9 把 `run_settled` 的处理器写成「`UIRoot` 发起结算编排」；本 ADR 把该行细化为「`run_settled` 只装载待交付结果，持久化发生在 `Run → Menu` 交付时」。这是对 ADR-0004 措辞的修订，不是对「`state_changed` 单订户 + 固定扇出」的修订；ADR-0004 的 D8 六步顺序保持不变 |

## Context

### Problem Statement

`architecture.md` 的 QQ-07 只写了一句话：「Settlement transaction boundary: UI Flow as orchestrator vs a dedicated transaction service.」但真正的缺口有五个，且互相咬合：

1. **状态归属未闭合。** `run-flow.md` 规则 1 / 2 / 12 已说 `run_id` 与 `RunLog` 归餐局流程，但 `settlement-and-entry-fee.md` 又保留 `last_settled_run_id`，`records-and-milestones.md` 保留会话级 `run_id` 去重，`meta-upgrades.md` 保留 `last_applied_run_id`。三个去重键各自成立，但没有任何文档说明「谁会持有局内状态到什么时候」，收据关闭后 `RunResult` 归谁也没有写。
2. **持久化时点未闭合。** `ui-flow.md` 规则 9 要求收据与 `RunResult` 交付同帧打开，规则 10 要求「先校验纪录，再动钱包，最后一次提交」；`run-flow.md` 规则 16 又要求在 `Handoff` 中退出时「已算出的 `RunResult` 不交付、不写纪录」。如果持久化挂在 `run_settled`（ADR-0004 D9 的字面读法），玩家在 `Handoff` 中退出会留下一局已被记录、却没有收据的结果；如果挂在 `Run → Menu`，又必须证明收据仍在同帧出现。
3. **钱包有两个潜在写者。** `settlement-and-entry-fee.md` 规则 18 说结算不碰钱包，`meta-upgrades.md` 规则 6 说界面流程合并 `progression` 与 `records`，`unlocks-and-stations.md` 规则 7 说解锁经 `MetaUpgrades.spend` 提交 `{progression, unlocks}`。三处都合法，但没有一份文档规定「一局结束时到底谁构造那一个 `Dictionary`、谁调用 `commit`、谁能在一个事务里加入第四个键」。
4. **失败后的可见状态未定义。** `records.prepare_run` 致命失败时 `ui-flow.md` 规则 10 要求整笔放弃、收据仍显示原始值；`meta-upgrades.md` 的 `WALLET_OVERFLOW` 要求钱包不变但「本局仍交由纪录系统处理」；`save-system.md` 规则 18 要求写盘失败保留内存并重试。三者组合出的失败矩阵没有被任何文档枚举，实现时必然各写各的。
5. **故障阶段只有一个布尔位。** `run-flow.md` 规则 18 要求 `fault.phase ∈ {INIT, RUN}`，但 `Run → Menu` 的 payload 仍写作 `init_failed = true`。`scene-flow.md` 规则 6 又把 `init_failed` 定义为「进入失败」，与「开局后结算失败」共用一个名字。`ui-flow.md` 规则 19 已经按 `phase` 分支，但没有任何 ADR 规定 `phase` 从哪一刻推导。

### Current State

- ADR-0001 / 0002 / 0003 / 0004 / 0005 / 0006 / 0008 均为 `Proposed`；`docs/registry/architecture.yaml` 已登记这些 ADR 的 state / interface / api stance，但**没有任何 `run_*`、`wallet` 或 `settlement` 条目**。
- `architecture.md` 第 7 项 Required ADR 明确写「Not written (ADR-0007)」；QQ-07 仍为 Open。
- `ui-flow.md` Open Question 1 已自答「界面流程为编排者」，但 `records-and-milestones.md` Open Question 1 仍写「编排者身份尚未指定」；两份 GDD 没有互相对齐。
- `run-flow.md` Open Question 1 已自答故障返回复用 `init_failed + fault.phase`，但没有说明 `run_faulted` 与「进入失败」是不是同一个事件。
- 仓库里没有任何 `.gd`，`project.godot` 只有引擎与渲染设置。本 ADR 是绿地决策。

### Constraints

- **同帧**：`ui-flow.md` 的 `receipt_open_latency_frames = 0` 与 `scene-flow.md` 的 `state_latency_frames = 0`。事务路径上禁止 `await`、`call_deferred()`、`Tween.tween_await()`、后台线程。
- **单写者**：`SaveSystem.commit` 是唯一持久化入口（ADR-0002 / `architecture.md` 不变量）。一局结束只允许一次 `commit`；不允许「钱包一次、纪录一次」。
- **结算不改内存策略**：写盘失败时结算来源的变化必须保留内存并重试（`save-system.md` 规则 18），不能像购买那样回滚。事务对象必须把「保留」当成正常路径，而不是异常分支。
- **依赖方向**：`Save ← MetaUpgrades / Unlocks / Records / UIRoot`；`RunFlow` 不依赖 Save、Records、MetaUpgrades 或任何界面类型（`architecture.md` Dependency direction）。因此事务对象不能住在 `RunFlow` 里。
- **无服务定位器**：ADR-0004 D3 禁止 `get_node("/root/…")`、`static var` 服务与依赖袋。事务对象只能经构造参数拿到 Save / Meta / Records / Unlocks。
- **无头可测**：`project.yaml` 的 `test` 是 headless GdUnit。事务顺序、失败矩阵与幂等必须能在不启动窗口的情况下用 commit spy 断言。
- **单线程主循环**：没有并发事务；「同一帧两个事务」只能是调用顺序问题，不是锁问题。

### Requirements

- 必须指定 `run_id`、`RunLog`、局内状态、`RunResult` 在每一段生命周期里的唯一持有者，并说明何时释放。
- 必须指定唯一的结算事务构造者与唯一的 `Save.commit` 调用点，且该点能在 `Handoff` 放弃时被跳过。
- 必须给出固定顺序：纪录校验、钱包入账、解锁（若有）、合并、提交、收据快照。
- 必须枚举失败矩阵：哪一笔子写入被丢弃、哪一笔保留在内存、收据显示什么、音效播放什么。
- 必须定义同一 `run_id` 重复投递时的幂等结果，以及「已应用但未落盘」后再次投递的行为。
- 必须定义 `INIT` 与 `RUN` 两个故障阶段的产物差异，并说明 `init_failed` 与 `run_discarded` / `run_faulted` / `settlement_aborted` 的区别。
- 必须让收据屏只读一个快照，且快照包含失败态与未保存态。

## Decision

### D1 — 局内状态唯一所有者是 `RunFlow`；不新增状态服务

| 状态 | 唯一所有者 | 生命周期 | 其他系统的访问方式 |
|---|---|---|---|
| `run_id`（int64，1–INT64_MAX） | `RunFlow` | 从 `Menu → Run` 生效到 `Handoff → Idle` | `run_started` / `run_settled` / `run_faulted` 的 payload；`snapshot().run_id` |
| 生命周期状态（`Idle/Initializing/Active/Ending/Settling/Handoff/Failed/Faulted`） | `RunFlow` | 同上 | `snapshot().run_state`（只读） |
| `run_seed`、`action_seq`、`running_net`、`shortfall` | `RunFlow` | 同上 | `snapshot()`（只读、不可变副本） |
| `RunLog` | `RunFlow` 组装并持有 | `Ending` 组装，`Settling` 交给 `Settlement.settle`，`Handoff` 后释放 | 只作为 `settle(log)` 的参数；`Settlement` 不得保留引用 |
| `RunResult` | `RunFlow` 持有到交接；交付后归 `UIRoot` 的收据快照 | 结算成功 → 交接完成 | `run_settled` 预通知、`Run → Menu` payload、`ReceiptSnapshot.result` |
| `last_settled_run_id` | `Settlement`（仅去重，不含其他局内状态） | 进程级 | `settle()` 内部；对外不可见 |
| `records` / `progression` / `unlocks` 命名空间值 | 各自系统（Records / MetaUpgrades / Unlocks） | 进程级 + 持久化 | `get_namespace()` 只读；新值经 `prepare_*` 返回 |

三条硬约束：

1. **`RunFlow` 不持有 Save / Records / MetaUpgrades 的引用。** 事务边界不在 Core 层；`RunFlow` 只负责「算出 `RunResult` 并交付」，不负责「把它写进生涯」。
2. **`Settlement` 保持纯函数。** 它只拥有 `last_settled_run_id` 这一个去重标量；不持有 `RunLog`、不读存档、不发信号、不写文件。`settle(log)` 返回后，`RunFlow` 可以立即丢弃 `RunLog`。
3. **`run_id = 0` 是保留值，表示「尚未分配」。** GDD 的分配范围是 1–INT64_MAX，因此 0 不会与任何真实局冲突。`run_id` 分配本身溢出时，`run_faulted` 仍需要一个可观测身份，用 0 表示「无局身份」；界面只按 `fault.phase = INIT` 处理，不显示 0。

**为什么不新增 `RunStateService`**：局内状态是单局、单线程、随 `RunFlow` 生命周期生灭的对象。把它抽成服务只会制造第二个 `run_id` 副本与第二个 `run_state` 副本，而本 ADR 要消除的正是「两个真相」。`SettlementTransaction` 只处理「局结束后的事实」，不持有任何局内可推进状态。

### D2 — 结算事务的唯一构造者是 `SettlementTransaction`（纯 `RefCounted`）

`SettlementTransaction` 是 `Feature` 层的纯逻辑对象，`AppRoot` 在 `BootStep.CONSTRUCT` 中用构造参数装配，并注入 `UIRoot`：

```gdscript
var _settlement_tx := SettlementTransaction.new(_save, _meta, _records, _unlocks, _data)
_ui.attach(_scene_flow, _input, _save, _meta, _records, _run_flow, _settlement_tx, _screen_factory)
```

- **它不是 `Node`**：没有 `_process`、没有 `Tween`、没有引擎生命周期；可以在无头测试里 `new()` 后用 spy 替换五个依赖。
- **它不是 autoload**：ADR-0004 D1 的零 autoload 不变。
- **它不是 `RunFlow` 的成员**：`RunFlow` 不依赖 Save / Meta / Records；把事务塞进 `RunFlow` 会把 Core 层直接绑到持久化与界面时序。
- **它不是 `UIRoot` 内的匿名代码**：事务顺序、失败矩阵与幂等需要单测；写进 `Control` 脚本就无法脱离场景树验证。
- **它拥有两个进程级标量**：`_last_run_id` 与 `_last_snapshot`。它们只服务幂等，不是局内状态；收据关闭后快照由 `UIRoot` 释放，事务对象只保留「最后应用的 `run_id` + 结果摘要」用于拒绝重复投递。

**QQ-07 的答案**：界面流程是编排者（触发者与收据拥有者），但编排算法的唯一实现是一个专门的 `SettlementTransaction` 对象。既不是「UI 里随手拼字典」，也不是「新增一个常驻事务服务」。

### D3 — 唯一 `Save.commit` 边界：`Run → Menu` 被接受时，收据挂载之前

固定时序（同一帧、同步）：

```
RunFlow.Settling
  1. assemble RunLog
  2. Settlement.settle(RunLog) -> RunResult          # 纯函数，一次
  3. state = Handoff; emit run_settled(run_id, result)  # 预通知，不写盘
  4. request_transition(Run -> Menu, result)
SceneFlow
  5. 写入 state = Menu；emit state_changed(Run, Menu, result)
AppRoot._on_state_changed 第 2 步
  6. UIRoot.reset_for_state(Run, Menu, result):
       a. snapshot = SettlementTransaction.apply(result)   # 唯一 commit 在这里
       b. replace_root(RECEIPT_SCREEN) 并 present(snapshot)  # 同帧
       c. emit receipt_opened(run_id)、settlement_started(run_id)
  7. AppRoot 第 3 步卸载 HudHost
```

四条硬约束：

1. **`run_settled` 不是提交点。** 它的处理器只做两件事：把 `result` 记为「待交付结果」（供首局引导 B4 与断言使用），并检查 `run_id` 尚未交付。它**不得**调用 `prepare_run`、`prepare_run_result` 或 `commit`。理由：`run-flow.md` 规则 16 允许玩家在 `Handoff` 中退出，此时结果「不交付、不写纪录」；若在 `run_settled` 写盘，这个窗口就被破坏。
2. **`RunFlow` 请求的 `Run → Menu` 被接受，才算交付。** 被拒绝（同帧暂停先到、或状态已变）时 `RunFlow` 留在 `Handoff` 重发；待交付结果仍在 `UIRoot` 中，但**没有任何持久化发生**。若玩家此时退出，`run_discarded` 发出，待交付结果被丢弃，事务从未运行。
3. **一局一次 `commit`。** `SettlementTransaction.apply(result)` 对同一 `run_id` 至多执行一次持久化；重复交付返回缓存快照，`commit` spy 不增加。
4. **事务形状固定**：

```
{
  &"records":     <Records.prepare_run 产出的完整新值>,
  &"progression": <MetaUpgrades.prepare_run_result 产出的完整 progression>,
  # &"unlocks":   <仅当本局产生奖励解锁时存在；MVP / Vertical Slice 不存在>
}
```

- 每个键都是**完整新值**，不是增量（`save-system.md` 对局外升级的要求；`records` 同理）。
- `unlocks` 在 MVP / Vertical Slice **不属于结算**：`unlocks-and-stations.md` 规则 15 明确「一局结束不改变解锁」。该键是为未来「一局奖励解锁」预留的**同一事务槽位**；届时只能加入奖励集合，绝不能在结算里调用 `spend` / `unlock`。菜单里的付费解锁是另一笔事务（`MetaUpgrades.spend`），与本事务串行、不合并。
- 事务对象**不得**调用 `purchase` / `spend` / `unlock`，也不得嵌套 `commit`。

**为什么不是 `AppRoot` 的 2.5 步**：ADR-0004 D8 的第 2 步已经是 `UIRoot.reset_for_state`，收据必须在该步内挂载；把事务拆成第 2.5 步会让「收据先挂、HUD 后卸」与「收据只读已提交快照」二者不能同时成立，除非再引入一个 `ReceiptViewModel` 的延迟构建阶段。把事务放在第 2 步内部、挂载之前，既满足同帧，也不改 ADR-0004 的六步扇出表。

### D4 — 事务内部固定顺序：纪录 → 钱包 → 解锁槽位 → 合并 → 一次提交

`SettlementTransaction.apply(result)` 的顺序不可交换：

| # | 步骤 | 输入 / 输出 | 失败时 |
|---|---|---|---|
| 1 | 幂等检查 | `_last_run_id` 与结果摘要（`run_id`、`net_profit`、`total_value`、`rows.size()` 的整数摘要） | 同 id 同摘要 → 返回缓存快照，后续步骤全跳过；同 id 不同摘要 → `RUN_ID_REUSED`，整笔中止 |
| 2 | `Records.prepare_run(result)` | `RecordsApplyResult{status, change, pending_events}` | `INCONSISTENT_RESULT` / `OVERFLOW` / `DATA_NOT_READY` / `MILESTONE_CATALOGUE_INVALID` → 整笔中止：**钱包步骤不执行、不构造事务、不 commit** |
| 3 | `MetaUpgrades.prepare_run_result(result)` | `{progression_update, status}` | `ALREADY_APPLIED` → 钱包键不加入事务（见 D5 的一致性规则）；`WALLET_OVERFLOW` → 钱包键不加入事务，纪录照常提交 |
| 4 | 解锁槽位 | MVP / VS：不执行，`unlocks` 键不存在；未来：`Unlocks.prepare_run_rewards(result)` 只返回奖励集合 | 预留接口，当前无失败路径；未来失败 → 整笔中止 |
| 5 | 合并 | `tx := {records: R'}`；钱包可用时加 `progression: P'`；有奖励解锁时加 `unlocks: U'` | — |
| 6 | `SaveSystem.commit(tx)` | `CommitResult` | 任何非 `OK`：保留内存新值、`mark_dirty` 每个已构造的命名空间、快速失败，**不回滚** |
| 7 | 采纳与发信号 | 内存值成为当前值；`Records` 的三个信号、`progression_changed` | — |
| 8 | 构建快照 | `ReceiptSnapshot`（见 D6） | — |

**为什么纪录必须排在钱包之前**：`records-and-milestones.md` 规则 3 与规则 11 把纪录一致性断言定义为整个结算事务的**唯一致命闸门**；`ui-flow.md` 规则 10 也写作「先校验纪录，再动钱包」。若先入账钱包，`records` 失败后就必须回滚钱包，而 `save-system.md` 规则 18 明确结算来源不允许回滚。把致命校验放在最前，钱包只在「确定要提交」时才在内存中改写，失败矩阵因此只有「全弃」与「部分键」两种，没有「回滚」。

**为什么 `WALLET_OVERFLOW` 仍提交纪录**：`meta-upgrades.md` 规则 4 与 Edge Case 明确要求「钱包保持原值，本局仍照常结算给纪录系统，界面显示入账失败」。这不是「部分事务」，而是**事务内容按策略缩键**：`records` 是同一份结算结果的事实，钱包只是入账失败；收据同时显示新纪录与「入账失败」并不矛盾。

### D5 — 失败矩阵与幂等：一笔子写入的命运必须可枚举

**同 `run_id` 重复投递**

| 场景 | `prepare` 调用 | `commit` 调用 | 返回 | 说明 |
|---|---|---|---|---|
| 同 `run_id`、同摘要，第一次 `COMMITTED` | 1 | 1 | 缓存快照 | 第二次投递不得重复入账、不得重开收据 |
| 同 `run_id`、同摘要，第一次 `UNSAVED` | 1 | 1 | 缓存快照（`UNSAVED`） | 不重试 `commit`；落盘由下一个写入时点（购买 / 下一局 / 退出 flush）完成 |
| 同 `run_id`、同摘要，第一次 `ABORTED` | 1 | 0 | 缓存快照（`ABORTED`，同一 `abort_code`） | 数据不会再变，重试只会得到同一失败；不再调用 `prepare` |
| 同 `run_id`、不同摘要 | 0 | 0 | `RUN_ID_REUSED` | 记 ERROR；这是上游 bug，不产生第二张收据 |
| `records` 返回 `ALREADY_APPLIED`，但钱包不是 `ALREADY_APPLIED` | 1 | 0 | `INCONSISTENT_TRANSACTION_STATE` | 同一 `run_id` 只可能整笔落账；一半已应用说明上一笔事务被拆散，必须中止并记 ERROR |
| 钱包返回 `ALREADY_APPLIED`，但 `records` 返回 `APPLIED` | 1 | 1（仅 `records`） | 正常快照 | 只补纪录，不重复加钱；这是「上一笔只落了钱包」的修复路径，仍是一次提交 |

**`Save.commit` 失败矩阵**

| 失败 | 丢弃的子写入 | 保留在内存 | `ReceiptSnapshot.save_state` | 收据显示 | 后续 |
|---|---|---|---|---|---|
| `records.prepare_run` 致命 | **全部**：钱包未计算、事务未构造 | 无 | `ABORTED` + `abort_code` | 原始 `RunResult` + 「结算数据异常，未入账、未记录」；无新纪录印章 | 发 `settlement_aborted(run_id, code)`；音效取消结果 cue；不重试 |
| `WALLET_OVERFLOW` | 只丢弃 `progression` 键 | `records` 新值 | `COMMITTED` 或 `UNSAVED`（按 `commit` 结果） | 收据正常 + 「入账失败」；纪录印章保留 | 钱包保持旧值；不重试入账 |
| `commit` 返回 `COMMIT_FAILED` / `SAVE_DEFERRED` / `NOT_WRITABLE` / `SAVE_TOO_LARGE` | **不丢弃任何内存子写入** | `progression` + `records` 全部新值 | `UNSAVED` + `reason` | 收据正常 + 「未保存」；`NOT_WRITABLE` 显示「本次会话不保存」 | `mark_dirty(progression)`、`mark_dirty(records)`；下一个写入时点重试 |
| 事务对象重入（`_applying == true`） | 本次调用 | 上一次的内存值 | `ABORTED` + `APPLY_REENTRANT` | 与上一张收据相同（不新建） | 记 ERROR；不 commit |
| `MetaUpgrades` 处于 `BUSY`（异常路径） | 全部 | 无 | `ABORTED` + `ORCHESTRATOR_BUSY` | 「结算数据异常，未入账、未记录」 | 记 ERROR；菜单流程保证购买是同步的，正常不可达 |

**失败后的信号**

- `settlement_aborted(run_id, code)` 只在「整笔中止」时由 `UIRoot` 发出一次，且**只在 `records` 致命、重入、`BUSY`、`RUN_ID_REUSED` 这几类**；`WALLET_OVERFLOW` 与写盘失败**不是中止**，不发该事件（收据已成功进入内存态）。
- `record_broken` / `milestone_earned` / `career_completed` 由 `Records` 在 `prepare_run` 返回 `APPLIED` 时发出（`records-and-milestones.md` 规则 11 / AC 20 的固定顺序）。本 ADR 不把发信号搬到 `commit` 之后：结算事务**永不回滚内存**，所以「内存已应用」就是玩家看到的成功态；把信号延后到写盘成功反而会让「未保存」的收据没有印章。ADR-0004 D9 的「事务成功后」应读作「事务已应用后」，这是本 ADR 对那条措辞的修订（见 Consequences）。
- `progression_changed` 在钱包值实际变化后发一次，`APPLIED_UNSAVED` 也发（`meta-upgrades.md` 规则 6）。`WALLET_OVERFLOW` 不发。
- 同一 `run_id` 的重复投递不重发任何信号。

**跨进程**：`run_id` 是会话级（`run-flow.md` 的 `run_id_alloc`），MVP 不持久化；进程崩溃发生在内存应用与 `commit` 之间时，本局无任何可恢复状态（`run-flow.md` 规则 16 的「崩溃作废」）。若未来引入局史或断线续跑，必须把 `_last_run_id` 与摘要写进新命名空间并走 schema 迁移；本 ADR 不预做。

### D6 — 收据只读 `ReceiptSnapshot`，绝不重跑结算

`SettlementTransaction.apply()` 返回一个不可变快照；`UIRoot` 把它交给 `ReceiptScreen.present()`。收据屏对上游只有这一个数据入口。

| 字段 | 类型 | 来源 | 用途 |
|---|---|---|---|
| `run_id` / `station_id` | int64 / StringName | `RunResult` | 去重与标题 |
| `result` | `RunResult`（深拷贝 `rows` 与 `upgrade_snapshot`） | `Settlement.settle` | 全部金额、容量、回合、回本率 |
| `save_state` | `COMMITTED / UNSAVED / ABORTED` | 事务 | 「已保存」/「未保存」/「未入账、未记录」 |
| `abort_code` | `StringName`（`&""` 表示无） | 事务 | 中止原因（debug 显示；玩家文案由 `ui-flow` 映射） |
| `wallet_before` / `wallet_after` / `wallet_state` | int64 / int64 / enum | `MetaUpgrades` | 钱包差额与「入账失败」标记 |
| `progression_delta` | int64 | 事务计算 | 收据上的「+N / −N」 |
| `record_broken` / `previous_best` | bool / `int \| null` | `RecordsApplyResult` | 「新纪录」印章与旧值并排 |
| `milestones_earned` | `Array[StringName]`（id 升序） | `RecordsApplyResult` | 里程碑条目 |
| `career_completed` | bool | `RecordsApplyResult` | 通关横幅 |
| `built_frame` | int | `Engine.get_process_frames()` | 断言 `receipt_open_latency_frames = 0` |

硬约束：

- `ReceiptScreen` **不得**调用 `Settlement.settle`、不得读 `RunFlow.snapshot()`、不得读 `RunLog`、不得自行对 `rows` 求和或计算 `broke_even`。`RunResult` 已经是最终事实；`broke_even` 是 `net_profit ≥ 0` 的派生读法。
- `ReceiptSnapshot.result` 必须是深拷贝或不可变约定下的只读引用；`ReceiptScreen` 只读不写。首局引导的 `FirstRunExplanation` 由 `UIRoot` 在快照之外组合成 `ReceiptViewModel`，不污染结算事实。
- 快照的生命周期 = 收据根屏的生命周期。`receipt_closed` 或根屏被替换时释放；不跨局缓存（`ui-flow.md` 规则 9 的「不缓存到收据关闭之后」）。
- `settled` 的 `RunResult` 只交付一次：`SceneFlow` 拒绝第二次 `Run → Menu`（非 `Run` 状态记 ERROR）；`SettlementTransaction` 与 `UIRoot` 各有一道幂等闸。

### D7 — `INIT` 与 `RUN`：`run_started` 是唯一分界，`init_failed` 只是兼容位

`fault.phase` 的推导规则只有一条：**`run_started` 是否已经发出**。

| | `INIT` | `RUN` |
|---|---|---|
| 判据 | `run_started` 从未发出 | `run_started` 已发出 |
| 典型原因 | `RunConfig` 解析失败、`station_id` 不存在、升级快照非法、任一子系统 `initialize` 失败、`run_id` 分配溢出 | `RunLog` 不变式失败（`CAPACITY_MISMATCH` 等）、`Settlement.settle` 返回结构化错误、交付前发现重复 `run_id` |
| 状态名 | 可能是 `Failed`，也可能是 `Faulted`（`run_id` 溢出） | `Faulted` |
| `RunLog` / `RunResult` | 都没有 | `RunLog` 可能已组装；`RunResult` 没有 |
| `run_faulted` | 发一次，`detail.phase = INIT` | 发一次，`detail.phase = RUN` |
| `run_discarded` | **不发**（本局从未开始） | 不发（故障不是放弃） |
| 收据 | 不打开 | 不打开 |
| 结算事务 | 从不运行 | 从不运行 |
| 界面文案 | 「本局未能开始」+ 原因 | 「本局结算失败，收益不计」+ 原因 |
| `Capacity.end_run(reason)` | `failed` | `faulted` |

三条实现约束：

1. **`init_failed` 是对「payload 中没有 `RunResult`」的兼容布尔位。** 为兼容 `scene-flow.md` 规则 6 与 `run-flow.md` 规则 18 的既有形状，`INIT` 与 `RUN` 故障都把 `init_failed = true`；但**任何界面不得只读它**。`fault.phase` 必须始终存在；`ui-flow.md` 规则 19 的「`phase` 缺失按 `INIT`」只作为损坏数据的兜底，并在 debug 记 WARNING。
2. **`settlement_aborted` 不是故障。** 它表示「局已经正常结算出 `RunResult`，但持久化事务整笔中止」；此时 `RunFlow` 已经 `Handoff` 并切到 `Menu`，收据仍然打开，显示原始值加「未入账、未记录」。它不发 `run_faulted`，也不改 `run_state`。
3. **`run_faulted` 与 `run_discarded` 互斥且各自至多一次。** `run_discarded` 只在「已开始的局被非交接方式离开」时发；`run_faulted` 只在「生命周期无法继续、必须无结果返回」时发。两者同时出现视为上游 bug，`AudioDirector` 按 `run_faulted` 优先（`audio-feedback.md` 的同帧裁定顺序）。

**`run_id = 0` 的边界**：分配溢出时本局没有有效身份。`run_faulted` 仍发一次，`run_id = 0`，`detail.phase = INIT`，`detail.code = RUN_ID_OVERFLOW`；不允许回绕到 1，也不允许用 `-1`。

### D8 — 与其他提交来源的串行关系：不合并、不嵌套、不重入

- **主线程串行**：全部 `commit` 都在主线程同步执行；`SaveSystem` 内部按到达顺序串行（`save-system.md`）。本 ADR 不引入事务队列。
- **购买 / 解锁不进入结算事务**：两者只在 `Menu` 发起，且 `purchase` / `unlock` 是同步的「先落盘再确认」。收据事务运行在 `Run → Menu` 的入口帧，此时 `UpgradeScreen` / 档口选择器尚未挂载；不存在「购买在途 + 结算提交」的合法交错。若 `MetaUpgrades` 报告 `BUSY`（异常路径），事务按 D5 中止并记 ERROR，不排队等下一帧（排队会破坏同帧收据）。
- **不嵌套**：`SettlementTransaction` 只调用 `prepare_*` 与 `commit`，绝不调用 `purchase` / `spend` / `unlock`，避免在 `commit` 内部再发一次 `commit`。
- **不重入**：`apply()` 置 `_applying = true`，任何递归调用（例如信号处理器在收据挂载期间请求开始下一局）立即返回 `APPLY_REENTRANT` 并记 ERROR；正常路径下 `run_settled` / `state_changed` 的派发深度已经由 ADR-0004 D7 限制为 ≤ 1。
- **与 `BootSequence` 的关系**：基线写者仍是 `AppRoot.BootSequence`（ADR-0004 D5），只在 `Fresh` / 删除生涯后运行；两者不会在同一帧发生（结算需要一局，基线需要启动或删除生涯）。

### Architecture Diagram

```
RunFlow (Core, 唯一局内状态所有者)
  Settling: RunLog → Settlement.settle() → RunResult
      │
      ├─ emit run_settled(run_id, result)        # 预通知：不写盘
      │      └─ UIRoot: pending_result = result
      │      └─ Onboarding: 构建 B4 explanation（若有）
      │
      └─ request_transition(Run -> Menu, result)
             │
             ▼  SceneFlow 接受，emit state_changed(Run, Menu, result)
       AppRoot._on_state_changed
         1 run_flow.set_frozen(false)
         2 UIRoot.reset_for_state(Run, Menu, result)
             │
             ├─ SettlementTransaction.apply(result)        ◄── 唯一 Save.commit
             │     1 Records.prepare_run(result)  ── 致命 → ABORTED（0 commit）
             │     2 MetaUpgrades.prepare_run_result(result)
             │     3 (unlocks 槽位：MVP 无)
             │     4 Save.commit({records, progression[, unlocks]}) 一次
             │     5 ReceiptSnapshot
             │
             ├─ ReceiptScreen.present(snapshot)             # 同帧
             └─ emit receipt_opened / settlement_started
         3 HudHost.sync_for_state(Run -> Menu)             # 收据先挂、HUD 后卸

局外（不经过 RunFlow）：
  MetaUpgrades.purchase()  ── commit({progression})         # 购买：失败回滚
  Unlocks.unlock() → MetaUpgrades.spend(..., {unlocks})     # 解锁：失败回滚
  BootSequence.ensure_baseline() ── commit({progression, unlocks, records})
```

### Key Interfaces

```gdscript
# res://src/feature/settlement/settlement_transaction.gd
class_name SettlementTransaction
extends RefCounted

enum SaveState { COMMITTED, UNSAVED, ABORTED }
enum WalletState { APPLIED, UNCHANGED, OVERFLOW, ALREADY_APPLIED }
enum AbortCode {
    NONE,
    RECORDS_INCONSISTENT, RECORDS_OVERFLOW, RECORDS_DATA_NOT_READY,
    MILESTONE_CATALOGUE_INVALID,
    RUN_ID_REUSED, INCONSISTENT_TRANSACTION_STATE,
    ORCHESTRATOR_BUSY, APPLY_REENTRANT,
}

func _init(save: SaveSystem, meta: MetaUpgrades, records: Records,
           unlocks: Unlocks, data: DataConfig) -> void
func apply(result: RunResult) -> ReceiptSnapshot   # 同步；同一 run_id 幂等
func last_applied_run_id() -> int                  # 0 = 从未应用


# res://src/feature/settlement/receipt_snapshot.gd
class_name ReceiptSnapshot
extends RefCounted

var run_id: int
var station_id: StringName
var result: RunResult                 # rows / upgrade_snapshot 已深拷贝
var save_state: int                   # SaveState
var abort_code: int                   # AbortCode
var wallet_before: int                # int64
var wallet_after: int                 # int64
var wallet_state: int                 # WalletState
var progression_delta: int            # int64，收据显示用
var record_broken: bool
var previous_best: Variant            # int | null
var milestones_earned: Array[StringName]
var career_completed: bool
var built_frame: int
```

```gdscript
# UIRoot 侧的调用形状（节选；不存在第三个 commit 调用点）
func reset_for_state(from: int, to: int, payload: Variant) -> void:
    if to == SceneState.MENU and payload is RunResult:
        var snapshot: ReceiptSnapshot = _settlement_tx.apply(payload)
        _screen_factory.replace_root(ScreenId.RECEIPT, snapshot)
        if snapshot.save_state == SettlementTransaction.SaveState.ABORTED:
            settlement_aborted.emit(snapshot.run_id, snapshot.abort_code)
        else:
            settlement_started.emit(snapshot.run_id)
    # ... 其余状态的屏幕重建
```

**事务对象的依赖只注入下游方向**：`Save ← SettlementTransaction`、`MetaUpgrades / Records / Unlocks` 由事务对象读取并调用其 `prepare_*`；事务对象不持有 `RunFlow`、`SceneFlow`、`UIRoot` 或 `AppRoot`。它不知道「当前顶层状态」，调用者保证只在 `Run → Menu` 的交付路径上调用。

### Implementation Guidelines

1. **必须**让 `RunFlow` 成为 `run_id`、生命周期、`RunLog`、`RunResult` 的唯一所有者；**禁止** `Settlement`、`Records`、`MetaUpgrades`、`UIRoot` 持有可推进的局内状态。
2. **必须**把结算事务的构造放在 `AppRoot.BootStep.CONSTRUCT`，经构造参数注入 `UIRoot`；**禁止** `UIRoot` 自己 `new()` 事务对象、禁止 autoload、禁止按名字查找。
3. **必须**只在 `UIRoot.reset_for_state(Run → Menu, RunResult)` 内、`ReceiptScreen` 挂载之前调用 `SettlementTransaction.apply`；**禁止**在 `run_settled` 处理器里 `prepare` 或 `commit`。
4. **必须**对同一 `run_id` 至多调用一次 `Save.commit`；**禁止**把 `{records}` 与 `{progression}` 拆成两次写入。
5. **必须**按「`Records.prepare_run` → `MetaUpgrades.prepare_run_result` → 合并 → `commit`」的顺序执行；**禁止**先动钱包再校验纪录。
6. **必须**在 `records.prepare_run` 致命时整笔放弃，不构造事务、不 commit、不发结果信号；**禁止**只让纪录失败而钱包照常入账。
7. **必须**在 `WALLET_OVERFLOW` 时省略 `progression` 键并照常提交 `records`；**禁止**因此中止整笔事务或回滚纪录。
8. **必须**在 `commit` 非 `OK` 时保留全部内存新值并对每个已构造命名空间 `mark_dirty`；**禁止**回滚结算来源的变化。
9. **必须**把 `unlocks` 键限制为「本局奖励解锁」；MVP / VS 不产生该键；**禁止**在结算事务中调用 `purchase` / `spend` / `unlock`。
10. **必须**让 `ReceiptScreen` 只读 `ReceiptSnapshot`；**禁止**在收据路径调用 `Settlement.settle`、读 `RunLog`、重新求和或计算 `broke_even`。
11. **必须**用 `run_started` 是否发出推导 `fault.phase`；**禁止**用 `Failed` / `Faulted` 状态名、`init_failed` 或错误码字符串反推阶段。
12. **必须**在 `RUN` 故障时使用「本局结算失败，收益不计」文案；**禁止**复用「本局未能开始」。
13. **必须**在 `settlement_aborted` 时取消同一 `run_id` 的结果 cue；**禁止**把事务中止当作一局故障（不播 `run_faulted` 的 cue、不关收据）。
14. **必须**在收据关闭或根屏替换时释放快照；**禁止**把上一局快照缓存到下一局。
15. **必须**让 `_applying` 防重入，并在 `apply` 内完成全部步骤；**禁止** `await`、`call_deferred()`、后台线程或把事务拆到两帧。

## Alternatives Considered

### Alternative 1: `RunFlow` 直接编排结算事务

- **Description**：`RunFlow` 在 `Settling` 里调用 `records.prepare_run`、`meta.prepare_run_result` 与 `save.commit`，然后携带结果请求 `Run → Menu`。
- **Pros**：时序最集中；`RunLog` 与 `RunResult` 不用离开所有者；`Handoff` 放弃窗口天然不存在（提交早于交付）。
- **Cons**：Core 层直接依赖 Save / Records / MetaUpgrades / DataConfig，违反 `architecture.md` 的依赖方向；`RunFlow` 会变成「既算局又写生涯」的上帝对象；写盘失败需要 Core 理解 `dirty` 与重试；无头测试必须装配存档目录。更严重的是，提交早于交付意味着「玩家在 `Handoff` 中退出」仍已落盘，与 `run-flow.md` 规则 16 冲突。
- **Estimated Effort**：中（1.2×），但把分层破坏写进代码。
- **Rejection Reason**：直接违反「Core 不依赖持久化 / 界面」的方向，并且无法满足「放弃不写纪录」。

### Alternative 2: `AppRoot` 作为事务编排者（D8 的 2.5 步）

- **Description**：`state_changed(Run, Menu, RunResult)` 的扇出里，第 2 步只挂一个空收据壳，第 2.5 步由 `AppRoot` 调事务，第 3 步再填充内容。
- **Pros**：Presentation 不碰持久化；`AppRoot` 已经持有全部服务；顺序显式。
- **Cons**：收据必须在第 2 步「先挂、后填」，否则 `receipt_open_latency_frames` 不成立；这会把 `ReceiptScreen` 变成两阶段视图（先空壳后数据），并新增一个「事务失败时如何撤回已挂载壳」的分支。`ui-flow.md` 规则 10 与 `architecture.md` 原则 3 都明确写「界面流程是编排者」，改成 `AppRoot` 需要同时改文档与验收。
- **Estimated Effort**：中（1.3×）。
- **Rejection Reason**：为了一行扇出顺序，把收据变成两阶段渲染，并推翻已有的 GDD / 原则表述；收益不抵复杂度。

### Alternative 3: 常驻 `SettlementService`（`Node` 或 autoload）

- **Description**：把事务做成一个常驻服务，任何系统在局结束时调用 `services.settle(result)`。
- **Pros**：调用点单一；未来可在服务内加队列。
- **Cons**：ADR-0004 D1 的零 autoload 被破坏；`Node` 服务会引入 `_process` / `_ready` 生命周期，而事务只在交付帧运行一次；服务化会让「最后一张收据」变成全局状态，收据关闭后还要清理；测试必须起树。
- **Estimated Effort**：低（0.6×）。
- **Rejection Reason**：把一个每局一次、同帧完成的纯算法变成常驻服务，增加生命周期而没有增加能力。

### Alternative 4: UI Flow 内联拼事务（不抽对象）

- **Description**：`UIRoot` 直接调用 `records.prepare_run`、`meta.prepare_run_result`，在方法体里拼 `Dictionary` 并 `commit`。
- **Pros**：少一个文件；调用链最短；GDD 字面上就是「界面流程合并」。
- **Cons**：失败矩阵、幂等键、快照构建全部塞进 `Control` 脚本；无头测试必须实例化屏幕；静态扫描无法区分「收据渲染」与「事务编排」；第二次有人需要读结算事务时只能复制。
- **Estimated Effort**：低（0.4×），但不可测。
- **Rejection Reason**：与本 ADR 要求的「失败可枚举、幂等可断言」正面冲突。

### Alternative 5: 两次提交：先钱包、再纪录

- **Description**：钱包入账成功后立刻 `commit({progression})`，纪录随后 `commit({records})`。
- **Pros**：每一步都短；某一步失败时另一部分已经落盘。
- **Cons**：正是 `save-system.md` 与 `records-and-milestones.md` 明令禁止的「同一份结算结果在两个命名空间里变成两种事实」；写盘失败时要么钱包已落、纪录没落，要么反之；收据无法给出单一「已保存」状态；`run_id` 去重也失去意义。
- **Estimated Effort**：低（0.3×）。
- **Rejection Reason**：直接违反单事务要求与失败策略。

## Consequences

### Positive

- **QQ-07 关闭**：状态所有者、事务构造者、提交时点、收据数据源四处都有唯一答案；`docs/registry/architecture.yaml` 可以登记 `run_state`、`run_log`、`wallet`、`receipt_snapshot` 与事务契约。
- **「放弃不是结算」在 `Handoff` 中仍然成立**：提交只在 `Run → Menu` 被接受后发生；`run_settled` 不再是一个隐式写盘点。
- **收据是纯投影**：同一份 `ReceiptSnapshot` 在收据、首局引导与 QA 日志中逐字段一致；不存在「收据重新结算」的第二套算术。
- **失败矩阵可测试**：commit spy 能对每个分支断言「调用几次、载荷几个键、哪些命名空间 dirty」。
- **故障文案不再混淆**：`phase` 由 `run_started` 推导，`run_id` 溢出这种「`Faulted` 但从未开局」的边界也能得到 `INIT` 文案。

### Negative

- `UIRoot` 的责任变重：它既是屏幕栈所有者，又是事务触发者。必须靠 `SettlementTransaction` 把算法挤出去，否则会重演「界面里藏业务」。
- 多了一个 `RefCounted` 对象与一份快照；收据打开期间 `RunResult` 实际存在两份（`RunFlow` 在 `Handoff` 已释放，`Records` / `Meta` 不保留，所以稳定态只有快照一份，但组装瞬间会有短暂重叠）。
- `run_settled` 的语义变成「预通知」，与 ADR-0004 D9 的「发起结算编排」措辞不一致；需要修订 ADR-0004 的文字，否则后来者会照字面在 `run_settled` 里提交。
- 纪录的三个信号在 `commit` 之前发出（`prepare_run` 内），与 ADR-0004 D9 的「事务成功后」不一致。本 ADR 选择保留 GDD 行为，因为结算永不回滚内存；代价是「未保存」收据也会显示新纪录印章。
- `SaveSystem.commit` 的 `COMMIT_FAILED` 与 `SAVE_DEFERRED` 由谁决定，在 ADR-0002 中没有完全闭合；本 ADR 在调用方一侧做映射（任何非 `OK` 都按「保留 + dirty」处理），这意味着 Save 不需要知道事务来源，但也意味着购买路径必须自己回滚。

### Neutral

- `SettlementTransaction` 常驻但不持有局内状态；`_last_run_id` 是会话级去重键，跨会话不保证唯一（与 `run_id` 的会话作用域一致）。
- `ReceiptSnapshot` 深拷贝 `rows` 与 `upgrade_snapshot`；这是有意的内存换确定性，行数上限由回合数约束（MVP 10 行、满级 25 行）。
- `unlocks` 键在事务形状中存在但在 MVP / VS 不出现；这是预留而不是未决。
- `run_id = 0` 作为「未分配」哨兵；真实 `run_id` 从 1 开始，不会冲突。

### GDD Contradictions Recorded (not edited)

以下矛盾按任务约束**只记录、不修改 GDD**；修复应由各 GDD 的所有者在下一轮复审中完成。

1. **编排者身份**：`design/gdd/records-and-milestones.md:141` 与 `:470`（Open Question 1）写「编排者身份尚未指定」，而 `design/gdd/ui-flow.md:63`（规则 10）与 `:629`（Open Question 1 已答）写「界面流程是编排者」。本 ADR 裁定：`SettlementTransaction` 是唯一算法实现，`UIRoot` 是唯一调用者；Records 的 OQ1 应标记为已解决。
2. **纪录信号时机**：`design/gdd/records-and-milestones.md:37`（规则 11）与 `:435`（AC 20）把 `record_broken` / `milestone_earned` / `career_completed` 的发信号放在 `prepare_run` 管线内；`docs/architecture/adr-0004-service-composition-signals.md:236` 写「结算事务成功后」。本 ADR 保留 GDD 行为，并建议把 ADR-0004 D9 的措辞改为「事务已应用后」。
3. **`Handoff` 中退出与 `run_settled`**：`design/gdd/run-flow.md:44`（规则 16）说在 `Ending` / `Settling` / `Handoff` 退出时「已算出的 `RunResult` 不交付、不写纪录」；同文件 `:102`（状态表）在 `Settling → Handoff` 就广播 `run_settled(RunResult)`，`:133` 也把该信号写作携带 `RunResult`。结果在「交付」之前已经被订阅者看到。本 ADR 裁定：`run_settled` 是预通知，`Run → Menu` 才是交付；写盘只挂在交付上。建议 `run-flow.md` 在规则 13 / 16 中明确这一区分。
4. **`init_failed` 的语义**：`design/gdd/scene-flow.md:32`（规则 6）与 `:61`（转移表）把 `init_failed` 定义为「进入失败（`RunConfig` 无法解析）」；`design/gdd/run-flow.md:46`（规则 18）把同一布尔位也用于开局后的 `RUN` 故障。本 ADR 裁定：`init_failed` 只表示「payload 中没有 `RunResult`」，`fault.phase` 才是阶段判据；建议 `scene-flow.md` 的 payload 说明补充 `fault.phase`。
5. **`Faulted` 与 `RUN` 阶段**：`design/gdd/run-flow.md:107`（状态表）把 `run_id` 分配溢出放进 `Faulted`，但该失败发生在 `run_started` 之前。若按「`Faulted` ⇒ RUN」实现，会对从未开始的局说「本局结算失败」。本 ADR 用 `run_started` 是否发出推导阶段，明确该行是 `Faulted + INIT`；建议 `run-flow.md` 在状态表补一句。
6. **解锁是否属于结算**：`design/gdd/unlocks-and-stations.md:55`（规则 15）写「不参与结算事务」，`design/gdd/save-system.md:88` 与 `:264` 一边说「结算引起的钱包、纪录与解锁变化必须合并为同一事务」，一边说「MVP/VS 的菜单解锁是独立事务」。本 ADR 裁定：MVP / VS 的结算事务只含 `{records, progression}`；`unlocks` 只在未来「局内奖励解锁」时进入同一事务。建议 `save-system.md` 把「解锁」限定为「由结算本身产生的解锁」。
7. **文档层面的同一句**：`docs/architecture/architecture.md:212` 的 `Save.commit({progression, records[, unlocks]})` 与 `:216` 的失败描述仍是未加限定的三命名空间形状；它不是 GDD，但应随本 ADR 一起把 `[, unlocks]` 标注为「仅未来奖励解锁」。

## Risks

- **收据同帧被 `commit` 的 50 ms 拖慢**：`Save.commit` p95 ≤ 50 ms 会直接拉长 `Run → Menu` 那一帧。→ 缓解：这是 `save-system.md` 已接受的写盘预算；事务不增加第二次 I/O，收据动画在提交之后才启动，且不 `await`。若实测超预算，先查存档编码而不是把事务拆帧（拆帧会违反 `receipt_open_latency_frames = 0`）。
- **纪录信号在 `prepare` 内发出**：若未来把 `prepare_run` 改成可失败后重试，信号可能重复。→ 缓解：`prepare_run` 对同一 `run_id` 返回 `ALREADY_APPLIED` 且不重发；事务的快照缓存是第二道闸。
- **`UIRoot` 重新长出业务逻辑**：新增「结算奖励」时有人会直接在收据屏里加 `if`。→ 缓解：静态扫描 `ReceiptScreen` 不得出现 `Settlement.` / `RunLog` / `commit(`；所有事务分支只存在于 `SettlementTransaction`。
- **`run_settled` 的预通知被误用**：有人在监听器里直接写盘。→ 缓解：D5 的 commit spy 在 `run_settled` 与 `Run → Menu` 之间必须为 0；代码评审把「监听器内出现 `prepare_` / `commit(`」列为阻断项。
- **内存快照的生命周期泄漏**：收据根屏被替换时忘记释放。→ 缓解：快照只由 `ReceiptScreen` 持有；`UIRoot` 在 `replace_root` / `receipt_closed` 时置空并断言引用计数归零（测试）。
- **`WALLET_OVERFLOW` 的「部分事务」被误解为原子性破坏**：它只提交 `records`，不提交 `progression`。→ 缓解：这是 GDD 明确要求的策略；收据同时显示「新纪录」与「入账失败」，并由 `progression_changed` 不发出证明钱包未变。
- **4.7 引擎未知**：本 ADR 没有新增 API；若 ADR-0002 / ADR-0004 的同步假设被实测推翻（例如 `commit` 变成异步），事务时序需要重写。→ 缓解：把 ①②③④ 列入 Verification Required，未验证前不写依赖同帧的 story。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `run-flow.md` | TR-run-flow-001：生命周期与唯一 `run_id` | D1：`RunFlow` 是唯一所有者；`run_id = 0` 仅表示未分配 |
| `run-flow.md` | TR-run-flow-002：初始化顺序与 `init_failed`，失败不产生 `RunResult` | D7：`phase` 由 `run_started` 推导；`INIT` 不产生 `RunLog` / `RunResult` / 收据 |
| `run-flow.md` | TR-run-flow-005：`RunLog` 来自动作时刻、`settle` 一次、`RunResult` 交接 | D1 + D3：`RunLog` 只由 `RunFlow` 组装并交给 `Settlement` 一次；`RunResult` 在 `Run → Menu` 交付一次 |
| `run-flow.md` | TR-run-flow-006：`run_*` 信号每 `run_id` 至多一次 | D3 / D7：`run_settled` 预通知一次；`run_faulted` / `run_discarded` 互斥且各一次 |
| `run-flow.md` | TR-run-flow-007：只读快照供 HUD | D1：`snapshot()` 是 `run_state` / `running_net` / `shortfall` 的唯一发布面 |
| `run-flow.md` | 规则 16：`Handoff` 中放弃不写纪录；规则 18：`fault.phase` | D3（提交只在交付后）+ D7（阶段推导） |
| `settlement-and-entry-fee.md` | TR-settlement-001：`settle` 每 `run_id` 一次，重复 `ALREADY_SETTLED` | D1 / D4：`Settlement` 保留 `last_settled_run_id`；`SettlementTransaction` 另有一道 `run_id` 幂等闸 |
| `settlement-and-entry-fee.md` | TR-settlement-004：纯函数、无 I/O；规则 18：不碰钱包 | D1 / D4：`Settlement` 无 Save / Records / Meta 引用；钱包只由 `MetaUpgrades` 计算 |
| `settlement-and-entry-fee.md` | 规则 19 / TR-settlement-005：`running_net` / `shortfall` 单一来源 | D1：运行期累加器在 `RunFlow`；收据只读 `RunResult` |
| `meta-upgrades.md` | TR-meta-upgrades-005：`prepare_run_result` 只算内存；编排者一次提交；写失败 `APPLIED_UNSAVED` | D4 / D5：事务对象合并 `progression` 并一次 `commit`；非 `OK` 保留内存、`mark_dirty`、快照 `UNSAVED` |
| `meta-upgrades.md` | 规则 4 / Edge Case：`WALLET_OVERFLOW` 钱包不变、纪录照常 | D4 / D5：省略 `progression` 键，只提交 `records`，收据显示「入账失败」 |
| `records-and-milestones.md` | TR-records-002：`prepare_run` 每 `run_id` 一次；任何错误中止整笔 | D4：纪录是唯一致命闸门，失败时不计算钱包、不构造事务、不 `commit` |
| `records-and-milestones.md` | 规则 11 / AC 20：纪录 → 合计 → 里程碑 → 信号的固定顺序 | D4：`Records.prepare_run` 内部顺序不变；事务只负责合并与提交 |
| `records-and-milestones.md` | Open Question 1：谁拥有结算提交编排者 | D2：`SettlementTransaction`；`UIRoot` 唯一调用 |
| `unlocks-and-stations.md` | 规则 15：解锁不参与结算；未来奖励解锁必须并入结算事务 | D3 / D4：MVP 事务无 `unlocks` 键；未来奖励槽位固定在同一 `commit` |
| `save-system.md` | TR-save-system-006：一次事务原子提交；购买回滚 vs 结算保留 dirty | D3 / D5：一局一次 `commit`；结算失败不回滚；购买仍走自己的回滚路径 |
| `ui-flow.md` | TR-ui-flow-004：`prepare_run` → `prepare_run_result` → 一次 `commit`；收据同帧 | D3 / D4 / D6：顺序固定；事务在 `reset_for_state` 内、挂载前执行；快照同帧 |
| `ui-flow.md` | 规则 19：按 `fault.phase` 选择「未能开始 / 结算失败」文案 | D7：`phase` 由 `run_started` 推导；`init_failed` 不再单独使用 |
| `scene-flow.md` | 规则 6：`Run → Menu` 携带 `RunResult` 或 `init_failed`；交付一次 | D3 / D7：交付点是唯一提交点；`init_failed` 作为兼容位 |
| `audio-feedback.md` | `settlement_aborted` 取消结果 cue；`run_faulted` 使结果失效 | D5 / D7：整笔中止发 `settlement_aborted`；故障发 `run_faulted`；两者不混用 |

## TR Coverage

| TR ID | Coverage | Note |
|---|---|---|
| `TR-run-flow-001` | covered | D1：生命周期与 `run_id` 所有权 |
| `TR-run-flow-002` | covered | D7：初始化失败 → `INIT`，无结果 |
| `TR-run-flow-005` | covered | D1 / D3：`RunLog` 组装、`settle` 一次、交付一次 |
| `TR-run-flow-006` | partial | D3 / D7 规定 `run_settled` / `run_faulted` / `run_discarded` 的边界；信号表全文仍归 RunFlow |
| `TR-run-flow-007` | partial | D1 固定快照的唯一发布者；字段完整清单归 RunFlow / HUD |
| `TR-settlement-001` | covered | D1 / D4：结算去重与事务去重 |
| `TR-settlement-004` | covered | D1 / D4：`Settlement` 无 I/O，事务才有 I/O |
| `TR-settlement-005` | partial | D1：运行期唯一累加器在 `RunFlow`；纯函数公式归结算 |
| `TR-ui-flow-004` | covered | D3 / D4 / D6：顺序、单次提交、同帧快照 |
| `TR-meta-upgrades-005` | covered | D4 / D5：`prepare_run_result` + 一次提交 + `APPLIED_UNSAVED` |
| `TR-records-002` | covered | D4：`prepare_run` 一次、错误整笔中止 |
| `TR-save-system-006` | partial | 本 ADR 只覆盖「结算来源的一次事务与保留 dirty」；原子写入与词汇归 ADR-0002 |
| `TR-unlocks-003` | not covered | 付费解锁是独立购买事务，由 Unlocks / ADR-0002 约束；本 ADR 只规定它不并入结算 |

**建议修订的 TR 文本**

- `TR-run-flow-006`：在信号清单后补「`run_faulted` 的 `detail` 必须含 `phase ∈ {INIT, RUN}`；`run_id = 0` 表示分配失败、从未开局」。当前文本只列信号名，无法验收阶段区分。
- `TR-run-flow-005`：补「`run_settled` 是预通知，持久化发生在 `Run → Menu` 被接受之后；`Handoff` 中放弃不写盘」。当前文本只说「`RunResult` handed off」，无法区分通知与交付。
- `TR-ui-flow-004`：补「事务在 `UIRoot.reset_for_state(Menu, RunResult)` 内、收据挂载之前执行；`records` 致命时 `commit` 次数为 0 且发 `settlement_aborted`」。当前文本只要求顺序与同帧，没写失败路径。
- `TR-records-002`：补「致命错误时其他命名空间不得提交；`ALREADY_APPLIED` 不得触发任何 `commit`」。
- `TR-meta-upgrades-005`：补「`WALLET_OVERFLOW` 时 `progression` 键省略、`records` 仍提交；写盘失败返回 `APPLIED_UNSAVED` 且两个命名空间都 dirty」。
- `TR-settlement-001`：补「`ALREADY_SETTLED` 由 `Settlement` 判定；`SettlementTransaction` 另以 `run_id + 摘要` 拒绝重复应用，二者不互相替代」。

## Performance Implications

- **CPU**：每局一次同步事务，包含 `Records.prepare_run`（p99 ≤ 5 ms，AC 34）、`MetaUpgrades.prepare_run_result`（一次 int64 加法 + 字典构建，微秒级）、`Save.commit`（p95 ≤ 50 ms，ADR-0002）。事务总预算 **p95 ≤ 60 ms**、单帧内完成；结算本身 p95 ≤ 1 ms、硬上限 5 ms（AC 30）。没有任何每帧轮询。
- **Memory**：`ReceiptSnapshot` 持有 `RunResult` 的深拷贝（最多 25 行、每行约 7 个整数字段 + 两个 float），量级 < 16 KiB；收据关闭即释放。`SettlementTransaction` 只多持有 `_last_run_id` 与最后一份快照引用；后者在收据关闭时置空。
- **Load Time**：不影响启动。事务对象在 `CONSTRUCT` 步骤构造，零 I/O。
- **Frame cost**：收据帧包含一次 `Save.commit`，是 `Run → Menu` 切换帧的主要成本；不新增第二次写入、不启动后台线程。首帧渲染仍在同帧完成（`receipt_open_latency_frames = 0`），但该帧的时长会包含写盘。
- **Network**：无。

## Migration Plan

绿地实现，当前没有代码。落地顺序（每步可单独验收）：

1. `ReceiptSnapshot` / `AbortCode` / `SaveState` 值对象 + 纯单测（深拷贝、字段映射）。
2. `SettlementTransaction` 骨架：只接受 `RunResult`，用 commit spy 验证「同 `run_id` 第二次 apply 不调用任何依赖」。
3. 接入 `Records.prepare_run` 与 `MetaUpgrades.prepare_run_result`，按 D4 顺序；验证致命失败 → 0 commit、`WALLET_OVERFLOW` → 只提交 `records`。
4. `UIRoot.reset_for_state` 接入事务与快照；断言收据挂载帧 = 交付帧，且 `HudHost` 卸载在其后。
5. `run_settled` 改为只装载待交付结果；用 spy 验证 `run_settled` 与 `Run → Menu` 之间 commit 为 0。
6. 故障路径：`RunFlow` 在 `Failed` / `Faulted` 填 `fault.phase`；`UIRoot` 按 D7 显示文案；验证 `run_discarded` / `run_faulted` / `settlement_aborted` 三者互斥。
7. 首局引导 B4 与收据 `ReceiptViewModel` 组合；验证 `settlement_aborted` 时结果 cue 被取消。
8. 全链路 headless：AC 覆盖 D5 的六行矩阵 + 性能预算。

**Rollback plan**：若 `SettlementTransaction` 被证明过度抽象，回滚方式是把 `apply()` 的方法体搬进 `UIRoot` 的一个私有方法，`ReceiptSnapshot` 与 commit spy 保持不动；若「提交只在交付后」被证明不可行（例如 `Run → Menu` 被拒绝且无法重发），回滚方式是改为 `run_settled` 提交并在收据关闭时补发「已记录」标记——但那会重新打开 `Handoff` 放弃的窗口，属于产品取舍，必须先修订 `run-flow.md` 规则 16。两种回滚都不触碰存档格式或结算数学。

## Validation Criteria

- **状态所有权**：静态扫描 `settlement*.gd` 中不出现 `commit(` / `SaveSystem` / `get_namespace(`；`run_flow.gd` 中不出现 `commit(` / `Records` / `MetaUpgrades` / `UIRoot`。
- **单次提交**：正常局 `commit` spy 调用恰好 1 次，载荷同时含 `records` 与 `progression`，且 `records` 是完整新值（不是增量）。
- **顺序**：spy 记录顺序严格为 `records.prepare_run` → `meta.prepare_run_result` → `commit`；`records` 致命时 `prepare_run_result` 调用 0 次、`commit` 0 次。
- **收据同帧**：`receipt_opened` 与 `state_changed(Run, Menu, RunResult)` 的 `Engine.get_process_frames()` 相等；`ReceiptScreen` 读到的快照 `run_id` 与交付的 `RunResult.run_id` 相等。
- **快照只读**：`ReceiptScreen` 源码中不出现 `Settlement.`、`RunLog`、`settle(`、`running_net(`、`shortfall(`；对快照 `result.rows` 的写入被类型/约定阻止（测试尝试写入副本，原快照不变）。
- **`Handoff` 放弃**：在 `run_settled` 之后、`Run → Menu` 被接受之前请求 `Paused → Menu (abandoned)`，断言 `commit` 0 次、`records` 不变、`progression` 不变、收据不打开、`run_discarded` 1 次、`run_faulted` 0 次。
- **重复投递**：同一 `RunResult` 连续 `apply` 两次，第二次 `prepare` / `commit` 计数不增加，返回同一快照（`is_same` 或逐字段相等）。
- **摘要冲突**：同 `run_id` 不同 `net_profit`，返回 `RUN_ID_REUSED`、`commit` 0 次、无收据、ERROR 日志含 `run_id`。
- **纪录致命**：注入 `INCONSISTENT_RESULT`，断言 `commit` 0 次、钱包逐字段不变、快照 `ABORTED`、`settlement_aborted` 1 次、`broke_even` / `loss` cue 未被 admitted。
- **钱包溢出**：注入 `WALLET_OVERFLOW`，断言 `commit` 1 次且载荷只含 `records`、钱包不变、快照 `wallet_state = OVERFLOW`、收据显示「入账失败」、纪录印章仍在。
- **写盘失败**：注入 `COMMIT_FAILED`，断言内存 `progression` / `records` 为新值、两者均 dirty、快照 `UNSAVED`、收据显示「未保存」；随后 `flush()` 成功，磁盘值等于内存值。
- **`NOT_WRITABLE`**：断言不尝试 I/O、快照 `UNSAVED + NOT_WRITABLE`、收据显示「本次会话不保存」。
- **`INIT` 故障**：`station_id` 不存在 → 无 `RunResult`、无收据、`run_faulted` 1 次且 `phase = INIT`、`run_discarded` 0 次、菜单提示「本局未能开始」、`commit` 0 次。
- **`RUN` 故障**：注入 `CAPACITY_MISMATCH` → 无 `RunResult`、无收据、`phase = RUN`、菜单提示「本局结算失败，收益不计」、`commit` 0 次。
- **`run_id` 溢出**：`max_seen_run_id = INT64_MAX` → `run_faulted` 1 次、`run_id = 0`、`phase = INIT`、不回绕。
- **信号互斥**：正常局 `run_settled` 1、`run_faulted` 0、`run_discarded` 0；故障局 `run_faulted` 1、`run_discarded` 0；放弃局 `run_discarded` 1、`run_faulted` 0。
- **性能**：20 行以内的 `RunResult`，`apply` 的 p95 ≤ 60 ms（含一次 `commit`），结算本身 p95 ≤ 1 ms；1000 次重复应用（同 `run_id`）不产生额外 `commit`。

## Related

- `docs/architecture/architecture.md` — QQ-07、Required ADR #7、原则 3（单写者 + 一个事务编排者）、原则 9、Data Flow 4 / 5、不变量
- `docs/architecture/adr-0002-persistence-format.md` — `commit` 原子性、`CommitResult`、失败策略（购买回滚 / 结算保留 dirty）、所有权表（「结算事务编排」不属于 Save）
- `docs/architecture/adr-0003-scene-screen-architecture.md` — D7（收据先于 HUD 卸载、同帧挂载）
- `docs/architecture/adr-0004-service-composition-signals.md` — D1（零 autoload）、D3（构造注入）、D8（六步扇出）、D9（`run_settled` 订阅）、Open Question 1
- `docs/architecture/adr-0001-content-data-format.md` — `DataConfig` 的 `Ready` 门与只读记录
- `design/gdd/run-flow.md` — 规则 1 / 2 / 12 / 13 / 16 / 18、状态表、`run_id_alloc`
- `design/gdd/settlement-and-entry-fee.md` — 规则 7 / 8 / 18 / 19、`RunResult` schema
- `design/gdd/meta-upgrades.md` — 规则 3 / 4 / 5 / 6、`prepare_run_result`、`WALLET_OVERFLOW`
- `design/gdd/records-and-milestones.md` — 规则 2 / 3 / 11 / 16 / 22、AC 20 / 35 / 36 / 37
- `design/gdd/unlocks-and-stations.md` — 规则 6 / 7 / 15
- `design/gdd/save-system.md` — 规则 6 / 8 / 17 / 18、AC 13
- `design/gdd/ui-flow.md` — 规则 9 / 10 / 13 / 19 / 21 / 25、Open Question 1
- `design/gdd/scene-flow.md` — 规则 6 / 8、转移表
- `design/gdd/audio-feedback.md` — `settlement_aborted` / `run_faulted` / 结果 cue
- `docs/engine-reference/godot/VERSION.md`、`current-best-practices.md`

## Open Questions

1. **`run_settled` 是否应更名为 `run_result_ready`？** 当前名字容易被读成「已经落盘」。本 ADR 不依赖改名（行为已固定为预通知），但若 `run-flow.md` 下一次修订，建议用更明确的名字，避免与 `settlement_started` 的「已提交」混淆。
2. **未来「局内奖励解锁」的 `RunResult` 字段叫什么？** 本 ADR 只预留了事务槽位与「不得在结算内 `spend`」的约束；字段 schema 属于结算与解锁的联合修订，不影响当前实现。
3. **是否需要在收据快照里保留 `run_seed` 供复盘？** `run-flow.md` 只要求 `run_seed` 在 `run_started` 中记录；收据当前不展示种子。若未来做「同种子重放」，需要新增字段并考虑存档 schema，不属于 MVP。
4. **`SaveSystem.commit` 的失败结果由谁决定 `COMMIT_FAILED` vs `SAVE_DEFERRED`？** ADR-0002 的 enum 同时提供两者，但未规定判定者。本 ADR 在调用方一侧映射（结算：任何非 `OK` 都保留 + dirty；购买：非 `OK` 都回滚），因此不阻塞；若 ADR-0002 修订为「Save 按事务来源返回」，本 ADR 的调用方逻辑可保持兼容。
