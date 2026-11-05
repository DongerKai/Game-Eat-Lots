# ADR-0004: 服务组合与信号机制（Service Composition & Signal Mechanism）

## Status

Accepted

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。
> 其他 agent 与任何 skill 都不能自行设置。** 引用 `Proposed` ADR 的 story 会被置为
> `Blocked`，epic 要求 `Accepted`，所以这个字段必须有明确的负责人。

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

- 用户（`modes.automation: autonomous`，由用户先行授权）
- `godot-specialist`（`SceneTree.paused`、`process_mode`、信号同步派发的引擎行为校验）
- `technical-director` 战略审阅：跳过（`review_mode: lean`）

## Summary

《回本》的服务图必须在第一帧之前就全部就位，且每次顶层状态切换都要在同一帧内完成「状态、输入上下文、界面、HUD、音频」五件事——这要求通知是**同步且有序**的，而 Godot 的 `connect()` 顺序不是可以依赖的契约。本 ADR 裁定两件事：

1. **组合根**：`res://src/main/main.tscn` 的根节点 `AppRoot`（`Node`）是唯一的组合根与启动编排者；项目 **autoload 数量为 0**。全部服务在 `AppRoot._ready()` 里按固定顺序构造，并经**构造参数**注入；纯逻辑服务是 `RefCounted`，需要输入、帧、音频或 `Tween` 的服务是 `AppRoot` 的子 `Node`。没有任何服务定位器、没有按名字查 autoload。
2. **信号**：信号只做「状态已经定稿之后的过去式通知」，跨系统同步契约一律直接调用并返回结果；**`state_changed` 只有一个订阅者（`AppRoot`）**，由 `AppRoot` 按固定顺序同步扇出，因此顺序与重入是可枚举、可断言的，而不是「谁先 connect 谁先跑」。

同一份裁定还闭合三处缺口：暂停用 `SceneTree.paused` 加一份显式的 `PROCESS_MODE_ALWAYS` 白名单实现（`TR-audio-feedback-005`、ADR-0002 的锁心跳都依赖它）；`QQ-12` 的新生涯基线写者指派为 `AppRoot.BootSequence`，Save 依旧不依赖 Unlocks；`QQ-04` 的「处理器能否请求状态切换」给出封闭答案。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；本机 `Installed at pin time = NOT DETERMINED`，尚未安装编辑器） |
| **Domain** | Core（服务组合、`Node.process_mode`、信号派发）；Presentation（音频与界面在暂停中的存活） |
| **Layer** | Foundation |
| **Knowledge Risk** | MEDIUM — `SceneTree.paused` / `process_mode` 属 4.0 起的稳定语义；但「`AudioStreamPlayer` 在树暂停后仍继续播放」「`PROCESS_MODE_ALWAYS` 的 `Tween` 是否推进」两处需要在 pin 定的 4.7.x 上实测。`docs/engine-reference/godot/VERSION.md` 把 Audio 标为 MEDIUM |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`breaking-changes.md`、`current-best-practices.md`、`deprecated-apis.md`、`modules/ui.md`、`modules/input.md` |
| **Post-Cutoff APIs Used** | 无新增 API。`InputEvent.DEVICE_ID_*`（4.7）经 ADR-0005 间接使用。命令行的 4.7 行为（`--check-only` 与 autoload、`--import` 与解析错误）来自 `current-best-practices.md` 的实测记录 |
| **Verification Required** | ① `get_tree().paused = true` 后，`PROCESS_MODE_ALWAYS` 的 `Tween` 是否继续推进（决定 `FadeLayer` 与 `NoticeLayer` 的 1.5 s 淡出是否需要手动时钟）；② `AudioStreamPlayer` 在树暂停后是否继续播放（决定 `AudioDirector` 是否必须自己 stop）；③ 信号在 `emit()` 内同步调用处理器时，`get_tree().paused` 的写入是否立即对同帧后续节点生效；④ `Node._ready()` 中子节点的 `_ready()` 触发顺序（本 ADR 不依赖它，仅确认构造注入不被 `_ready` 提前反噬）；⑤ 无 autoload 时 `--check-only -s res://<file>.gd` 逐文件通过（对照 `current-best-practices.md` 记录的「命名 autoload 会退出 1」） |

> ⚠️ **引擎知识缺口**：4.5–4.7 均在 LLM 训练截止之后，且本机没有安装编辑器。
> 上述 Verification Required 项必须在 pin 定的 4.7.x 上实测；在实测完成前，
> 本 ADR 中涉及运行时语义的结论按「待验证」处理。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001（`DataConfig` 的 `Ready` 门；里程碑目录插在 `Data Ready` 与 `Save.load()` 之间）；ADR-0002（`Save.commit` 原子事务；锁心跳驱动者必须 `PROCESS_MODE_ALWAYS`；Save 不得依赖 Unlocks / Records / MetaUpgrades）；ADR-0003（`UIRoot` 由组合根构造；收据必须先于 HUD 卸载挂载）；ADR-0005（`context_changed` 的发射时机、`push/pop` 成对、`ui_pause` 由 SceneFlow 独占） |
| **Enables** | ADR-0007（结算事务边界）——本 ADR 固定「谁构造事务参与者、谁扇出 `state_changed`」，事务顺序在其之上编排；ADR-0009（音频播放与暂停）——本 ADR 固定音频节点必须常驻运行；全部 epic 与 story |
| **Blocks** | 任何 story：没有组合根就没有可运行的服务图；`QQ-12` 未闭合则首次启动的基线写者不存在 |
| **Ordering Note** | 本 ADR 是 `architecture.md` Required ADRs 的第 4 项，编号与之一致。ADR-0003 D1 已把 `UIRoot` 定为「非 autoload、由组合根构造」，本 ADR 只是给出那个组合根的名字与形状；两者没有冲突。若 Required ADRs 重新编号，以文件名 `ADR-0004` 为稳定引用 |

## Context

### Problem Statement

`architecture.md` 把四个问题登记为必须在写第一个 story 之前裁定的架构缺口，它们全部落在本 ADR：

- **QQ-04（High）**：信号顺序与重入——`state_changed` 抵达多个消费者时，谁先跑？处理器能不能在回调里请求状态切换？`scene-flow.md` 规则 15 明确「信号的实现机制（信号还是直接调用、顺序与重入）仍由 ADR 决定」。
- **QQ-12（High）**：新生涯基线写者——`save-system.md` 要求 `Fresh` 时「写基线之前先调用解锁与档口的 `normalize()`，把回填后的 `unlocks` 并入同一基线事务，不写空集合基线」，但 ADR-0002 的所有权表把「正常化（`unlocks.normalize()` 的调用方）」明确列为**既不属于 Save 也不属于命名空间**，同时 ADR-0002 的实现指南禁止 Save 依赖 Unlocks 的类型。写者未指派，首次启动就会写出一份空集合基线。
- **暂停的 `process_mode`**：`audio-feedback.md` Open Question 6 把「暂停时音频处理必须继续运行，用 `process_mode = ALWAYS` 还是独立单例」留给实现；ADR-0003 D2.4 则无条件要求整条 UI 路径与输入分发必须常驻运行，「若暂停用 `SceneTree.paused` 实现（实现方式归 ADR-0004）」。
- **`UIRoot` 由谁构造**：ADR-0003 D1 定为「组合根」，但组合根是场景根节点还是一个 autoload、其余服务是不是 autoload，全部未定。

这四件事一旦在代码里各写各的，就会出现「菜单能跑但收据晚一帧」「暂停后心跳停了、另一个实例抢走了写锁」「首次启动的解锁集合永远是空」这三类只能靠重写修掉的问题。

### Current State

- ADR-0001 / ADR-0002 / ADR-0003 均为 `Proposed`；`docs/registry/architecture.yaml` 已登记这三条 ADR 的 state / interface / api stance。
- `project.godot` 共 315 字节：只有引擎、渲染器与物理设置，`run/main_scene` **未设置**，**没有 autoload 段**；仓库里没有任何 `.gd`。
- `architecture.md` 的启动顺序写作 `AppRoot (composition root) → DataConfig.load() → Save.load() → Scene Flow Boot → Menu → UI Flow builds MenuRoot`，但 `AppRoot` 只出现在这一行示意里，没有类型、没有归属、没有构造顺序。
- 九个架构原则中的第 8、9 条（组合根优于单例；信号通知、直接调用契约）已经写死方向，缺的是可执行的形状。

### Constraints

- **同帧生效**：`scene-flow.md` 的 `state_latency_frames = 0` 与 `ui-flow.md` 的 `receipt_open_latency_frames = 0`；任何 `await`、`call_deferred()`、`Tween.tween_await()` 进入通知路径即违反。
- **暂停期间必须继续工作的东西是封闭清单**：音频处理（`TR-audio-feedback-005`）、写锁心跳（ADR-0002：暂停超过 90 s 会让另一个实例回收锁）、输入分发与整条 UI 路径（ADR-0003 D2.4）。
- **Save 的依赖方向单向**：`MetaUpgrades / Unlocks / Records → Save`，反向禁止（ADR-0002 实现指南）。因此基线合并不能由 Save 发起。
- **无头可测**：`project.yaml` 的 `test` 是 `godot --headless ... GdUnitCmdTool.gd`，启动链、扇出顺序、暂停白名单必须能不开窗口断言。
- **无服务定位器**：不得按名字查 autoload，不得 `get_node("/root/SaveSystem")`，不得用 `static var` 承载服务。
- **解析检查的 4.7 坑**：`current-best-practices.md` 记录 `--check-only -s <file>` 在脚本命名 autoload 时退出 1（`Identifier not found`），而 `--import` / `--quit-after` 在解析错误时退出 0。autoload 越少，逐文件解析检查越可用。

### Requirements

- 必须唯一确定服务的构造顺序与注入方向，且该顺序可被一条测试断言。
- 必须让 `state_changed` 的消费者顺序可枚举、可断言，并保证「收据挂载」先于「HUD 卸载」（ADR-0003 D7）。
- 必须给出处理器「可做 / 不可做」的封闭清单，禁止在回调里同步改写发射者状态（`architecture.md` 不变量）。
- 必须给出暂停时继续运行的节点封闭清单，每个节点带理由。
- 必须指派新生涯基线写者，并保持 Save 不依赖 Unlocks / Records / MetaUpgrades。
- 必须让「数据失败 → `Boot → Error`、不构造 gameplay」可断言。
- 必须让 `session_rng` 只有一个来源（`run-flow.md` 的 `run_seed` 公式把播种方式留给 ADR）。

## Decision

### D1 — 组合根：`AppRoot` 是 `main.tscn` 的根节点，autoload 数为 0

- `res://src/main/main.tscn` 的根节点脚本是 `AppRoot`（`class_name AppRoot extends Node`），并在 `project.godot` 中显式写 `run/main_scene = "res://src/main/main.tscn"`（ADR-0003 实现指南第 10 条已要求同一次写入）。
- **项目注册的 autoload 数量为 0。** 原则 8 的「autoloads 限于启动/编排根」在本项目取严格解：既然唯一的编排根就是主场景根节点，且本项目从不用 `SceneTree.change_scene_to_file()`（ADR-0003 D1 禁止），主场景根节点在整个进程生命周期内已经常驻，autoload 没有额外的存活价值。
- 因此 `AppRoot` 是启动编排者、服务构造者、`state_changed` 的唯一订户、以及进程退出的唯一执行者。它是**唯一**允许同时持有全部服务引用的对象。
- 未来若必须引入 autoload（目前没有用例），唯一允许的候选是 `AppRoot` 自身改为 autoload 形态，且总数仍为 1——禁止第二个。

**为什么不把每个服务做成 autoload**：autoload 是全局可寻址的，任何脚本都能绕过依赖声明直接取用，等于把「依赖方向」交给代码评审；同时 4.7 的 `--check-only -s` 会在命名 autoload 的脚本上退出 1，逐文件解析检查失效（`current-best-practices.md`）。零 autoload 让「谁能看到谁」由构造签名回答。

### D2 — 服务图：纯逻辑是 `RefCounted`，引擎/帧依赖是 `Node`

| 归属 | 服务 | 理由 |
|---|---|---|
| **`Node` 子节点**（进树） | `SceneFlow`、`InputRouter`（+ 输入分发节点）、`UIRoot`（+ `ScreenHost` / `OverlayHost` / `ModalHost` / `NoticeHost` / `FadeLayer`）、`HudHost`、`AudioDirector`（+ voice 池宿主）、`SaveHeartbeat` | 需要 `_process` / `_input` / `Tween` / `AudioStreamPlayer` / `process_mode` |
| **`RefCounted` 字段**（不进树） | `DataConfig`、`SaveSystem`、`RunFlow`、`TimeTurns`、`StomachCapacity`、`DishValues`、`CloyingAndTaste`、`MenuWaves`、`Settlement`、`MetaUpgrades`、`Unlocks`、`Records`、`SessionRng`、`BootSequence`、`ScreenFactory` | 确定性纯逻辑或纯数据；无 `_process`，可在无头测试里直接 `new()` |

- **`Node` 服务一律是 `AppRoot` 的直接子节点**，节点名即服务名（`SceneFlow`、`InputRouter`…），但不作为寻址入口：任何跨服务引用都来自构造注入，而不是 `get_node()`。
- **`SaveSystem` 是 `RefCounted`**，但它需要周期性心跳。心跳被拆成 `SaveHeartbeat`（`Node`，`PROCESS_MODE_ALWAYS`），构造时注入 `SaveSystem` 并只调用它的 `pulse_lock()`。这是 ADR-0002 实现指南「必须用 `PROCESS_MODE_ALWAYS` 的驱动者发送心跳」的落地形状。
- **`RunFlow` 是 `RefCounted`**：它没有 `_process`，冻结由 `set_frozen(bool)` 这个显式入口表达（见 D8 扇出顺序第 1 步），不依赖节点暂停。
- **`AudioDirector` 是 `Node`**：它需要 `AudioStreamPlayer` 与 `AudioServer`。它不是 autoload，也不由 `UIRoot` 拥有——音频的生命周期比任何屏都长。

### D3 — 注入：构造参数、单向、禁止名字查找

- 每个服务通过**构造参数**（`RefCounted` 的 `_init(...)`）或**一次性 `attach(...)`**（`Node` 服务，因为 `Node` 不能有带参 `_init`）拿到依赖；`attach` 必须在 `add_child()` 之前调用，且只允许调用一次（第二次记 ERROR 并忽略）。
- 依赖只注入**下游方向**：`Data ←*`、`Save ← Meta/Unlocks/Records/UIRoot`、`SceneFlow ← UIRoot/RunFlow`、`Input ← UIRoot/HUD/Onboarding`、`RunFlow → Time/Capacity/Waves/Cloying`。方向的唯一来源是 `architecture.md` 的 Dependency direction 段。
- **禁止**：`get_node("/root/…")` 取服务、`Engine.get_singleton()` 取服务、`load()` / `preload()` 服务脚本、`static var` 服务实例、按字符串名解析依赖、把整个 `AppRoot` 或依赖袋（`Services`）传给任何服务。
- 只允许注入**接口面**：`UIRoot` 拿到的是 `SceneFlow` / `InputRouter` / `SaveSystem` / `MetaUpgrades` / `Records` / `RunFlow` 的只读或公开面，拿不到 `AppRoot`；`InputRouter` 拿不到 `UIRoot` 类型，只拿到 `ActionSink`（ADR-0005 D4）——Foundation 不得在类型层面依赖 Presentation。
- 构造失败（`null` 依赖、类型不符、`attach` 重复）走**封闭错误词汇** `BootError.SERVICE_CONSTRUCTION_FAILED`，`boot()` 立即返回该错误并**不**请求任何顶层切换（此时 `SceneFlow` 可能尚未构造）。release 构建下 `push_error` 后退出进程，debug 构建下停在错误报告上——与服务图残缺相比，静默半启动更危险。

### D4 — 启动顺序与 `BootSequence`

`AppRoot.boot()` 是唯一启动入口，**同步**执行下列步骤，每步失败即按表处理：

| # | `BootStep` | 动作 | 失败行为 |
|---|---|---|---|
| 1 | `CONSTRUCT` | 按 D2/D3 构造全部服务（含 `InputRouter` → `SceneFlow` → 各 gameplay → `UIRoot` → `HudHost` → `AudioDirector` → `SaveHeartbeat`） | `SERVICE_CONSTRUCTION_FAILED`，进程终止（debug 停在报告） |
| 2 | `LOAD_DATA` | `DataConfig.load()`（ADR-0001 状态机） | `DATA_FAILED`：`SceneFlow` 构造完成后请求 `Boot → Error`，**不构造任何 gameplay 查询路径** |
| 3 | `LOAD_MILESTONES` | `MilestoneCatalog.load()`（ADR-0001 决策 6 要求插在 `Data Ready` 与 `Save.load()` 之间） | `DATA_FAILED`，同上 |
| 4 | `LOAD_SAVE` | `SaveSystem.load()` 恰好一次 | 载入结果**不阻断**进入 `Menu`（`save-system.md` 规则 11）：`Blocked` 也继续到 `Menu`，由 `LoadReport` 决定界面分支 |
| 5 | `BASELINE` | `BootSequence.ensure_baseline(load_report)`（见 D5） | `BASELINE_COMMIT_FAILED`：**不阻断** `Menu`，保持三命名空间 dirty，由 `save_status_changed` 驱动保存指示 |
| 6 | `SCENE_BOOT` | `SceneFlow.request_transition(Boot → Menu)`；被拒绝则记 ERROR 并重试一次 | 与 ADR-0003 的 `BootWaiting` 屏幕共存 |
| 7 | `UI_BUILD` | `UIRoot.reset_for_state(Boot, Menu, null)`；`HudHost` 不挂载 | — |
| 8 | `READY` | 发 `boot_finished(report)`；`SaveHeartbeat` 开始心跳 | — |

**`BootSequence` 不持有 `SceneFlow` 与 `UIRoot`**：它只做「把数据与档读进来、把首个基线写出去」，返回 `LoadReport` / `CommitResult`；场景与界面由 `AppRoot` 在它返回之后驱动。这让基线写者可以在纯单元测试里构造，不需要起树。

**`session_rng`（`run_seed` 的来源）**：`SessionRng`（`RefCounted`）在步骤 1 构造，内部持有一个 `RandomNumberGenerator`，构造时 `randomize()` **恰好一次**；只暴露 `next_u64() -> int`（按二补码解释为 int64）。它被注入 `RunFlow`，`run-flow.md` 的 `run_seed` 公式因此有唯一实现点：`RunConfig.seed != null` 用它，否则用 `SessionRng.next_u64()`。除 `SessionRng` 外，任何代码禁止调用 `randi()` / `randf()` / `RandomNumberGenerator`（`MenuWaves` 只接收已确定的 seed）。

### D5 — QQ-12：新生涯基线写者是 `AppRoot.BootSequence`

`BootSequence.ensure_baseline(report)` 是**全项目唯一**的新生涯基线写者：

```gdscript
# BootSequence 只运输，不解释内容：三个命名空间的值都由各自所有者产出
func ensure_baseline(report: LoadReport) -> int:      # 返回 CommitResult
    if report.outcome != LoadReport.Outcome.FRESH:
        return CommitResult.OK                        # 非 Fresh 不写基线
    var persisted: Dictionary = _save.get_namespace(&"unlocks")   # Fresh 时为空集合
    var unlocks_value: Dictionary = _unlocks.normalize(persisted) # 纯函数，回填免费档口与菜
    var progression: Dictionary = _meta.default_progression()     # wallet 0 + 空等级表
    var records: Dictionary = _records.default_snapshot()
    var result: int = _save.commit({
        &"progression": progression,
        &"unlocks": unlocks_value,
        &"records": records,
    })
    if result == CommitResult.OK:
        _unlocks.adopt_normalized(unlocks_value)      # 内存与磁盘对齐；失败时内存已按规则 14 推导
    else:
        _save.mark_dirty(&"progression")
        _save.mark_dirty(&"unlocks")
        _save.mark_dirty(&"records")
    return result
```

四条硬约束：

1. **Save 不依赖 Unlocks**：`SaveSystem` 里不出现 `Unlocks` 类型、不调用 `normalize()`、不解释 `unlocks` 的含义；它只收到一个 `Dictionary` 并做结构校验（ADR-0002 §6）。
2. **Unlocks 不为基线发起写盘**：`Unlocks` 只在自己的一笔解锁事务里经 `MetaUpgrades.spend(..., extra)` 参与 `commit`（`unlocks-and-stations.md` 规则 6/7）。基线写入不是它的调用点。
3. **一次事务**：三个命名空间必须在**同一个** `save.commit` 里落地，不得先写空集合再补（`save-system.md` 边界用例「写基线之前先调用 `normalize()`，不写空集合基线」）。
4. **`ensure_baseline` 同时覆盖两条进入 Fresh 的路径**：① 首次启动（`load()` 返回 `Fresh`）；② `delete_career()` 成功之后、或 `Blocked` 强制模态里选择「新建生涯（旧档保留）」之后（`ui-flow.md` 规则 20）。第二条路径由 `UIRoot` 发起、`AppRoot` 执行 `ensure_baseline`——`UIRoot` 不自己拼基线字典。

**失败不阻塞**：基线写失败时游戏照常进入 `Menu`（`save-system.md`：写失败则保持 dirty 并提示，不阻塞进入菜单），三个命名空间全部标 dirty，下一个写入时点（结算 / 购买 / 退出 / `flush`）落盘。**不重试、不循环**：一次失败即返回，由 `save_status_changed` 的可见性提醒玩家。

### D6 — 信号契约（发射侧）

1. **过去式命名 + 通知语义**：信号名描述「已经发生的事」，不描述请求（`run_started`、`save_status_changed`、`receipt_closed`）。任何「请做某事」的跨系统契约都是直接调用并返回结果（`architecture.md` 原则 9）。
2. **状态先定稿，再发射**：`emit()` 之前，发射者必须已经写完自己的全部状态，且 payload 是可安全持有的副本或不可变快照（HUD 的 `HudFrame`、`RunResult`、`LoadReport`）。处理器从 payload 与查询 API 看到的世界必须一致。
3. **被拒绝的操作不发信号**：`REJECTED(code)` / `false` / 非 `OK` 的结果通过返回值交付，不伴随任何信号（`run-flow.md` 规则 5、`unlocks-and-stations.md` 规则 9 的同一形状）。
4. **每件事恰发一次**：同一 `run_id` 的 `run_*` 各至多一次；`state_changed` 每次被接受的切换恰一次；`context_changed` 每次被接受的请求恰一次。重复是 bug，不是常态。
5. **发射点是同步的**：在完成该变更的同一次调用栈里 `emit()`，不使用 `call_deferred`、`await`、定时器。帧末渲染时世界与信号已一致。
6. **禁止把信号当队列**：不得为了「晚一点再通知」而延后发射，也不得在信号里携带 Callable 让处理器回调发射者。

### D7 — 信号契约（处理侧）与 QQ-04 重入

处理器（`signal.connect` 的目标）的封闭清单：

**允许**：查询任何服务的只读 API 或快照；调用自己拥有的子系统的变更方法；调用 `SceneFlow.request_transition(...)` / `InputRouter.request_context(...)` 等「请求型」入口，但**必须**处理 `false` 返回值，且在同一次回调里最多请求一次；`push_error` / 记日志；播放自己拥有的音效（经 `AudioDirector.play`）。

**禁止**：同步调用**发射者**的任何变更方法（`architecture.md` 不变量：「任何信号处理器不得同步改变发射模块的状态」）；再次发射同名信号；`await` / `call_deferred` / 启动 `Tween` 后依赖它完成状态转移；`queue_free()` 或 `remove_child()` 发射者；把 payload 的引用存进自己状态而不做拷贝；在回调里发起第二次 `request_transition`。

**QQ-04 的答案：处理器可以请求状态切换，但请求必然按帧初状态校验，因此派发深度天然 ≤ 1。** `scene-flow.md` 规则 2 规定「一帧内的多个请求全部针对帧初状态校验…其余一律拒绝」：处理器在 `state_changed(Run, Menu)` 派发中请求 `Menu → Run`，这一对会按帧初状态（`Run`）判成 `Run → Run`，落进「同状态无 payload → 幂等空操作、不发信号」或直接拒绝。所以不存在递归。`AppRoot` 仍必须置 `_in_dispatch = true` 并在为真时忽略任何再次进入 `_on_state_changed` 的调用（防御，记 ERROR）。

**处理器不得假设自己先于别人**：除 D8 表中声明的顺序对之外，任何处理器不得依赖另一个处理器已经跑过。

### D8 — `state_changed` 的单一订阅者与固定扇出顺序

`SceneFlow` 的 `state_changed(from, to, payload)` **只有一个订阅者：`AppRoot`**。`AppRoot._on_state_changed()` 按下列**固定顺序同步**扇出，全部在同一帧内完成：

| 步 | 调用 | 为什么必须在这个位置 |
|---|---|---|
| 1 | `_run_flow.set_frozen(to == Paused)`（本局存在时；幂等） | 冻结早于任何界面或音频逻辑，`submit_action` 立刻开始返回 `REJECTED(FROZEN)` |
| 2 | `_ui.reset_for_state(from, to, payload)` | ADR-0003 D7：收据必须在 HUD 卸载**之前**挂上（`to == Menu` 且 payload 为 `RunResult`） |
| 3 | `_hud_host.sync_for_state(from, to)`（挂载 / 卸载） | 与第 2 步的先后是硬约束；顺序反过来会闪一帧无收据的空菜单。`Run ↔ Paused` 的同一调用同时是 HUD 的 `frozen` 变化通知：HUD 不监听 `ui_pause`，只读 `snapshot().frozen`（ADR-0019 D9） |
| 4 | `_audio.on_flow_state(from, to)` | `AudioDirector` 由 `(from, to)` 推导 `Frozen` / `Shutdown` 与 `flow.*` cue，1 帧内静默（`TR-audio-feedback-005`） |
| 5 | `_onboarding.on_flow_state(from, to)` | 首局引导需要知道已离开 `Run` / `Paused` 才能重压或拆除 `Blocked` |
| 6 | `_fade.play(from, to)` | presentation only，最后启动；`reduce_motion` 时长为 0，切换仍已生效 |

`SceneFlow` 在 `emit()` 之前已完成三件事，顺序也是硬约束：写入新状态 → `InputRouter.request_context(default_ctx, clear_stack := true)` → `get_tree().paused = (to == Paused)`。因此任何处理器看到的世界都是「状态已变、输入上下文已切、暂停已生效」。

**其余信号允许多个订阅者**，但连接点集中在 `AppRoot._wire_signals()` 一处，按 D9 表列出的顺序连接；只有表中「顺序」列写出的对（signal, 前→后）是契约，其余订阅者之间不得互相依赖副作用。这样 `AppRoot` 是唯一的 connect 场地，`grep connect(` 的结果可以逐个审。

### D9 — 跨系统生命周期契约表

| 事件 | 发射者 | 发射时机（不变量） | 订阅者（顺序） | 处理器上限 |
|---|---|---|---|---|
| `ready()` / `failed(err)` | `DataConfig` | 状态机进入 `Ready` / `Failed` 之后 | `AppRoot`（推进 `BootStep`） | 只读查询；不得构造 gameplay |
| `state_changed(from, to, payload)` | `SceneFlow` | 状态写入 + `request_context` + `paused` 之后 | `AppRoot`（唯一）→ D8 六步 | 见 D7 |
| `context_changed(from, to)` | `InputRouter` | 上下文或栈深改变之后 | `UIRoot`（深度断言）、`AppRoot`（日志） | 见 ADR-0005 D2 |
| `run_started(run_id, station_id, seed)` | `RunFlow` | 四子系统初始化成功、`Active` 之后 | `HudHost` → `Onboarding` → `Audio` | 只读快照；`Onboarding` 可 `push_context(Blocked)` |
| `action_committed(action, payload)` | `RunFlow` | 动作原子提交之后 | `HudHost` → `Audio` → `Onboarding` | 只读快照 |
| `run_ended(run_id, end_reason)` | `RunFlow` | 结束判定命中、动作冻结之后 | `HudHost` → `Audio` → `Onboarding` | 只读；不得结算 |
| `run_settled(run_id, result)` | `RunFlow` | `Handoff` 之后，`RunResult` 就绪 | `UIRoot`（收据编排）→ `Onboarding`（只 `STASH_RESULT`）→ `Audio` | `UIRoot` 发起结算编排（ADR-0007）；`Onboarding` 只暂存结果供首局引导 B4，不写盘、不发信号 |
| `run_faulted(run_id, code, detail)` / `run_discarded(run_id)` | `RunFlow` | 进入 `Faulted` / 丢弃之后 | `HudHost`、`Onboarding`、`Audio`、`UIRoot`（提示条目） | 只记日志与展示 |
| `run_config_assembled(config, plan)` | `UIRoot` | `RunConfig` 组装完成、`request_transition(Menu → Run)` 之前 | `AppRoot` → `Onboarding.configure(plan)` | 只读；`plan = get_first_run_preset()` 或 `null`；`RunConfig` 不新增字段（ADR-0020 决策 10） |
| `save_status_changed(status)` | `SaveSystem` | 每次可写性/保存态变化后 | `UIRoot`（保存指示、禁用写入口）、`MetaUpgrades`、`Unlocks` | 只读；不得触发写盘 |
| `progression_changed` / `unlocks_changed` | `MetaUpgrades` / `Unlocks` | 事务成功或回填改变集合后 | `UIRoot` → `Audio` | 刷新展示 |
| `record_broken` / `milestone_earned` / `career_completed` | `Records` | 结算事务已应用后（不是写盘成功后；见 ADR-0007 D5） | `UIRoot`（收据印章）→ `Audio` | 展示 |
| `receipt_opened` / `receipt_closed` | `UIRoot` | 收据成为根屏 / 关闭之后 | `Audio` | 展示 |
| `number_roll_started` / `number_roll_finished` | `UIRoot` | 滚动开始 / 落定之后 | `Audio`（`pending_outcome`） | 展示 |
| `ui_button_activated(control_id)` | `UIRoot` | 一次已提交的激活之后，至多一次 | `Audio` | 展示 |
| `onboarding_beat_shown` / `onboarding_finished` | `Onboarding` | beat 展示后 / 完成或跳过之后 | `Audio` | 展示 |
| `boot_finished(report)` | `AppRoot` | 步骤 8 | 无（日志与测试） | — |

「处理器上限」列是封闭的：表中未列出的行为视为缺陷，测试以「处理器内禁止出现的方法名」静态扫描兜底。

### D10 — 暂停：`SceneTree.paused` + `PROCESS_MODE_ALWAYS` 白名单（TR-audio-feedback-005）

**暂停的唯一机制是 `get_tree().paused`**，由 `SceneFlow` 在切换生效时同步写入（见 D8）。默认 `PROCESS_MODE_INHERIT` 的节点因此冻结，只有下列白名单节点显式写 `PROCESS_MODE_ALWAYS`：

| 节点 | `process_mode` | 理由 |
|---|---|---|
| `AppRoot` | `ALWAYS` | `Quit`、`flush`、释放写锁必须在暂停中可执行 |
| `SceneFlow` | `ALWAYS` | 必须读到 `ui_pause` 才能恢复；`Quit` 也要在暂停中可达（`scene-flow.md` 规则 7/10） |
| `InputRouter` + 输入分发节点 | `ALWAYS` | 暂停菜单的按键必须仍被翻译成动作（ADR-0003 D2.4） |
| `UIRoot` + 四个宿主 + `FadeLayer` | `ALWAYS` | `PauseOverlay`、设置/确认模态在暂停中仍需输入与 `Tween` |
| `SaveHeartbeat` | `ALWAYS` | 暂停超过 90 s 会让另一个实例回收写锁（ADR-0002） |
| `AudioDirector` + voice 池宿主 | `ALWAYS` | 否则收不到 `state_changed`，暂停音与恢复音迟到（`audio-feedback.md` OQ6） |
| `HudHost` 及其子节点 | `INHERIT`（默认） | 暂停时 `Tween` 与 `_process` 停止，HUD 停在最后一帧 |
| `Run` 场景与全部 gameplay `RefCounted` | `INHERIT` / 不适用 | 暂停即冻结；`RunFlow.frozen` 提供第二道闸 |

**`TR-audio-feedback-005` 的落地**：`AudioDirector` 常驻运行，在第 4 步收到 `on_flow_state(*, Paused)` 后**同一帧**停止全部 voice（≤ 50 ms 释放）并清空 `pending_outcome`；暂停期间的一切请求记 `dropped:paused`；恢复后不补播。**不得依赖 `process_mode` 让引擎自动停声**——`AudioStreamPlayer` 在树暂停后是否继续播放列入 Verification Required ②，因此 `AudioDirector` 必须自己 `stop()`，无论实测结果如何。静音与无设备是 drop 语义（`muted` / `no_device`），不阻断启动、不进入启动门。

**双重闸门**：`SceneTree.paused` 挡住帧推进，`RunFlow.set_frozen(true)` 让任何漏进来的 `submit_action` 返回 `REJECTED(FROZEN)`。两者都不是可选优化，任一缺失都会让「暂停中吃了一道菜」成为可能。

### D11 — `Quit` 的关闭序列

`Quit` 是顶层状态（`scene-flow.md`），**执行退出的是 `AppRoot`**，`SceneFlow` 只负责把状态切到 `Quit` 并通知。`AppRoot` 在扇出第 6 步之后同步执行：

1. `SaveSystem.flush()`；失败时 `save_status_changed` 已可见，**不询问、不重试**（二次确认属于 `UIRoot` 的退出确认路径，已在上游完成）。
2. `SaveHeartbeat.stop()` 并 `SaveSystem.release_lock()`（释放锁目录）。
3. `AudioDirector.shutdown()`（停 voice、`Shutdown` 状态、不再接受请求）。
4. `get_tree().quit()`。

顺序不可交换：先释放锁再 flush 会让另一个实例有机会在写入中途接手；先 `quit()` 则 2、3 不会执行。

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│ AppRoot (Node, res://src/main/main.tscn 根节点, 唯一组合根)               │
│   boot(): Data → Milestones → Save → Baseline → SceneBoot → UI → READY    │
│   _on_state_changed(): 固定六步扇出                                       │
│   _wire_signals(): 唯一 connect 场地                                      │
└───┬───────────────────────────┬───────────────────────────────┬──────────┘
    │ 子节点（Node，进树）        │ 字段（RefCounted，不进树）     │ 注入方向
    ▼                           ▼                               ▼
 SceneFlow      (ALWAYS)      DataConfig          Gameplay 只依赖下游
 InputRouter    (ALWAYS)      SaveSystem          Save ← Meta/Unlocks/Records
 UIRoot +4 host (ALWAYS)      RunFlow             SceneFlow ← UIRoot/RunFlow
 HudHost        (INHERIT)     TimeTurns …         Input ← UIRoot/HUD/Onboarding
 AudioDirector  (ALWAYS)      Settlement
 SaveHeartbeat  (ALWAYS)      MetaUpgrades / Unlocks / Records
 SessionRng / BootSequence / ScreenFactory

 状态切换（同帧、同步）：
   SceneFlow: 写状态 → request_context(default, clear_stack=true) → paused=… → emit
        │
        ▼ AppRoot（唯一订户）
   1 run_flow.set_frozen   2 ui.reset_for_state   3 hud.sync_for_state
   4 audio.on_flow_state   5 onboarding.on_flow_state   6 fade.play
```

### Key Interfaces

```gdscript
# res://src/main/app_root.gd
class_name AppRoot
extends Node

signal boot_finished(report: BootReport)

enum BootStep { CONSTRUCT, LOAD_DATA, LOAD_MILESTONES, LOAD_SAVE, BASELINE,
                SCENE_BOOT, UI_BUILD, READY }
enum BootError { NONE, SERVICE_CONSTRUCTION_FAILED, DATA_FAILED, BASELINE_COMMIT_FAILED }

class BootReport extends RefCounted:
    var step: int
    var error: int
    var load_report: LoadReport          # 步骤 4 之后非空
    var commit_result: int               # 仅 BASELINE 步骤有意义

func boot() -> BootReport                # 唯一启动入口；同步执行；可重复调用返回同一报告
func services_ready() -> bool
func ensure_baseline_after_career_reset() -> int    # 删除生涯/新建生涯后由 UIRoot 请求


# 启动编排（纯逻辑，可脱离场景树单元测试）
class BootSequence extends RefCounted:
    func _init(data: DataConfig, milestones: MilestoneCatalog, save: SaveSystem,
               meta: MetaUpgrades, unlocks: Unlocks, records: Records) -> void
    func load_all() -> LoadReport
    func ensure_baseline(report: LoadReport) -> int   # CommitResult；唯一基线写者


# 每进程唯一随机源（run_seed 的来源）
class SessionRng extends RefCounted:
    func _init() -> void                 # 内部 randomize() 恰好一次
    func next_u64() -> int               # 二补码解释为 int64


# 心跳驱动者（SaveSystem 是 RefCounted，心跳必须有常驻 Node）
class SaveHeartbeat extends Node:
    func attach(save: SaveSystem, interval_seconds: float = 30.0) -> void
    func start() -> void
    func stop() -> void


# 界面动作的接收口（Foundation 不依赖 Presentation 类型）
class ActionSink extends RefCounted:
    func handle_action(action: StringName) -> bool      # 基类返回 false，由 UIRoot 适配器重写
```

**依赖注入的形状（节选）**：

```gdscript
# 指针方向即依赖方向；没有任何参数是 AppRoot 或依赖袋
_run_flow = RunFlow.new(_turns, _capacity, _waves, _cloying, _settlement, _rng)
_meta     = MetaUpgrades.new(_save, _data)
_unlocks  = Unlocks.new(_save, _meta, _data)
_records  = Records.new(_save, _data, _milestones)
_ui       = UIRoot.new()
_ui.attach(_scene_flow, _input, _save, _meta, _records, _run_flow, _screen_factory)
_input.bind_action_sink(UIRootActionSink.new(_ui))     # ADR-0005 D4
```

### Implementation Guidelines

1. **必须**把 `run/main_scene` 显式写为 `res://src/main/main.tscn`，并把 `AppRoot` 挂在该场景根节点；**禁止**注册任何 autoload（`autoload/` 段必须为空或不存在）。
2. **必须**让 `AppRoot.boot()` 同步执行 D4 的八步；**禁止**在启动路径上 `await`、`call_deferred()`、`ResourceLoader.load_threaded_request()`。
3. **必须**把 `state_changed` 只连接到 `AppRoot._on_state_changed`；**禁止**其他任何对象 `connect("state_changed", ...)`。
4. **必须**在 `AppRoot._wire_signals()` 里集中连接其余信号，并按 D9 表的顺序；**禁止**在服务内部 `connect` 别的服务。
5. **必须**让 `_on_state_changed` 的前三步保持 `set_frozen → ui.reset_for_state → hud.sync_for_state` 的顺序；**禁止**调换（收据会晚一帧或被 HUD 覆盖）。
6. **必须**在 `emit()` 前完成状态写入与 payload 拷贝；**禁止**把活的内部 `Dictionary` / `Array` 直接作为 payload。
7. **必须**让每个处理器在调用 `request_transition` 后检查 `false`；**禁止**在回调里重试第二次或在回调里 `await`。
8. **必须**把 `PROCESS_MODE_ALWAYS` 只授予 D10 表中列出的节点；**禁止**给 `HudHost`、`Run` 场景或任何 gameplay 节点设 `ALWAYS`。
9. **必须**让 `AudioDirector` 在 `Paused` 的同一帧内自行 `stop()` 全部 voice 并清空 `pending_outcome`；**禁止**依赖 `process_mode` 或 `AudioServer` 自动静音来实现暂停静默。
10. **必须**把新生涯基线写在 `BootSequence.ensure_baseline` 一处，并把三个命名空间放进同一个 `commit`；**禁止** Save 引用 `Unlocks` / `Records` / `MetaUpgrades` 的类型，**禁止** `Unlocks` 为基线发起 `commit`。
11. **必须**让 `SessionRng` 成为唯一随机源；**禁止**在任何服务里调用 `randi()` / `randf()` / 新建 `RandomNumberGenerator`（`MenuWaves` 只接收 seed）。
12. **必须**用构造参数或一次性 `attach()` 注入依赖；**禁止** `get_node("/root/…")`、`Engine.get_singleton()`、`static var` 服务、依赖袋、按名查 autoload。
13. **必须**让 `BootSequence` / `SessionRng` 保持 `RefCounted` 与零节点依赖；**禁止**在其中 `get_tree()` 或读时钟。
14. **必须**在 `Quit` 上按 flush → 释放锁 → 音频 shutdown → `quit()` 的顺序执行；**禁止**在 `SceneFlow` 里直接 `get_tree().quit()`。

## Alternatives Considered

### Alternative 1: 每个服务一个 autoload（Godot 常规做法）

- **Description**：`DataConfig`、`SaveSystem`、`RunFlow`… 各注册为 autoload，任何脚本按名字取用。
- **Pros**：不需要写组合代码；测试里也能按名字替换；新手最熟悉。
- **Cons**：全局可寻址等于没有依赖声明，`architecture.md` 的 Dependency direction 只能靠评审；`--check-only -s` 在命名 autoload 的脚本上退出 1，逐文件解析检查失效；autoload 的 `_ready` 顺序由引擎决定，`DataConfig Ready → Save.load()` 的顺序要靠 `_ready` 里的等待来伪造。
- **Estimated Effort**：低（0.3×）。
- **Rejection Reason**：直接违反原则 8，并把「启动顺序」从可断言的表变成引擎内部行为。

### Alternative 2: 依赖袋 / 服务定位器（`Services.get(&"save")`）

- **Description**：构造一个持有全部服务的 `Services` 对象，任何服务通过它按名取依赖。
- **Pros**：构造签名短；新增依赖不改签名；循环依赖也能「跑起来」。
- **Cons**：循环依赖之所以能跑，是因为依赖方向不再被编译器与签名约束——正是本 ADR 要防的东西；无头测试要替换一个服务必须装配整个袋子；静态扫描无法判断「谁依赖谁」。
- **Estimated Effort**：低（0.4×）。
- **Rejection Reason**：把编译期可查的依赖变成运行期字符串，违反原则 6（闭合错误词汇）与原则 8 的意图。

### Alternative 3: `state_changed` 多订阅者，依赖 `connect()` 顺序

- **Description**：`UIRoot`、`HudHost`、`AudioDirector`、`Onboarding` 各自 `connect("state_changed", ...)`，靠「先连先跑」得到顺序。
- **Pros**：零扇出代码；每个系统自治；新增消费者不改 `AppRoot`。
- **Cons**：Godot 的调用顺序等于连接顺序，而连接顺序取决于各服务 `_ready()` 的先后——4.7 未把它写成跨版本契约；ADR-0003 D7 的「收据先挂、HUD 后卸」会变成两个模块的隐式约定；顺序 bug 只在特定帧、特定路径下出现，无头测试难以复现。
- **Estimated Effort**：低（0.3×），但排障成本高。
- **Rejection Reason**：把一条硬时序约束交给未承诺的引擎行为。

### Alternative 4: 手写 `frozen` 标志代替 `SceneTree.paused`

- **Description**：不碰 `get_tree().paused`，由 `RunFlow.frozen` 与各节点自查决定是否推进。
- **Pros**：完全可控；不会误冻结「本该继续」的节点；单测不需要起树。
- **Cons**：每一个有 `_process` / `Tween` / `Timer` 的节点都要自己查标志，漏一个就出「暂停中数字还在滚」；`Tween` 与 `AnimationPlayer` 有各自的暂停面，逐个转发等于把引擎已有机制重写一遍；新增节点默认忘记处理。
- **Estimated Effort**：中（1.5×），且是持续成本。
- **Rejection Reason**：默认冻结（`INHERIT`）+ 显式白名单，比默认放行 + 逐点检查更安全；`frozen` 仍保留为第二道闸。

### Alternative 5: 让 Save 直接调用 `unlocks.normalize()`（QQ-12 最小改动）

- **Description**：`SaveSystem.load()` 在 `Fresh` 分支里回调 Unlocks 做回填，然后自己写基线。
- **Pros**：改动最小；调用点就在写盘旁边；不需要新增编排对象。
- **Cons**：Save 直接依赖 Unlocks 的类型，违反 ADR-0002 实现指南「禁止 Save 依赖 Unlocks / Records / MetaUpgrades」；`Save.load()` 会因此需要 `DataConfig` 之外的构造依赖，`Blocked` / `ReadOnly` 分支也要考虑「能不能回填」，把纯存储层变成业务编排者；`Records` 的默认值与未来命名空间会继续挂进来。
- **Estimated Effort**：低（0.2×）。
- **Rejection Reason**：把 QQ-12 从「谁写」变成「Save 什么都管」，与分层方向相反。

## Consequences

### Positive

- **顺序是可枚举的常量表**：`state_changed` 的六步与 `BootStep` 的八步都可以被一条测试逐步断言，不依赖引擎的调用顺序。
- **依赖方向由签名保证**：任何越界依赖在编写时就报错，而不是在 review 时被发现；静态扫描（`get_node("/root`、`static var`、`autoload/`）几乎零误报。
- **零 autoload 让无头与解析检查都成立**：测试可以直接 `AppRoot.new()` 装配，`--check-only -s` 逐文件可用。
- **暂停行为有一份封闭白名单**：新增节点默认冻结，「暂停中仍在跑」是一个需要写理由的例外。
- **QQ-12 有了单一写者与单一事务**：首次启动不会再写出空解锁集合，`ensure_baseline` 同时覆盖删除生涯后的新生涯。
- **音频与心跳的存活不再靠运气**：两者都在 `ALWAYS` 表里，且各有一条硬理由。

### Negative

- `AppRoot` 是唯一的「知道所有服务」的文件，它会随系统增加而变长；必须靠 `BootSequence` / `SessionRng` 这类纯对象把逻辑挤出去。
- 零 autoload 意味着任何脚本都不能「随手拿到存档」，初期写小工具时会觉得绕。
- `state_changed` 的唯一订户是一个单点：`AppRoot._on_state_changed` 出 bug 会影响全部界面与音频，而多订阅者方案下其他系统仍能工作。
- 六步扇出是一个隐式契约，新增消费者必须改 `AppRoot`，不能只改自己的文件。
- `ActionSink` 适配器多一层间接：`InputRouter` 想调 `UIRoot.handle_action` 需要经 `UIRootActionSink`。

### Neutral

- `RunFlow` 是 `RefCounted` 而不是 `Node`，因此它不参与 `SceneTree.paused`；冻结靠显式 `set_frozen`。这是刻意的双机制，不是遗漏。
- `SaveHeartbeat` 是唯一为「驱动一个 `RefCounted`」而存在的 `Node`；若未来 `SaveSystem` 变成 `Node`，它可以被吸收。
- `BootStep` / `BootError` 是封闭枚举，新增步骤要改本 ADR——这是期望的摩擦。

## Risks

- **`Tween` 在 `ALWAYS` 节点上是否随树暂停**（Verification Required ①）：若实测「暂停后 `ALWAYS` 节点的 `Tween` 也停」，`FadeLayer` 与保存指示的 1.5 s 淡出会卡在半途。→ 缓解：这两处都不承载状态语义（ADR-0003 D8/D10 已声明），卡住不影响正确性；必要时改为由 `AppRoot` 的 `_process`（`ALWAYS`）推进的显式时钟。
- **`AudioStreamPlayer` 在树暂停后继续播放**（Verification Required ②）：若不实测就依赖 `process_mode`，暂停会带着声音。→ 缓解：实现指南第 9 条要求 `AudioDirector` 自己 `stop()`，与实测结果无关。
- **`AppRoot` 变成上帝对象**：系统增多时构造与扇出都在膨胀。→ 缓解：`BootSequence` / `SessionRng` / `ScreenFactory` 已经把逻辑分出去；扇出表是数据而非代码分支；当扇出超过约 10 步时，应把「界面与 HUD 同步」抽成一个 `PresentationSync` 对象，而不是继续加行。
- **`_in_dispatch` 防御被绕过**：若有人绕过 `AppRoot` 直接 `emit` 或直接调 `reset_for_state`。→ 缓解：静态扫描禁止外部调用 `emit` 与 `reset_for_state`；`UIRoot.reset_for_state` 在 `_in_dispatch == false` 时记 WARNING。
- **基线事务把三个命名空间绑在一起**：某一命名空间的值非法会让整笔 `commit` 失败，首次启动即无档。→ 缓解：三个值都是本进程刚构造的默认值（`wallet 0`、空等级表、回填后的解锁集合、空纪录），失败只可能来自 I/O 或结构校验；失败路径已定义为「保持 dirty + 不阻塞菜单」。
- **`paused` 在 `emit` 之前写入**：若某个 `ALWAYS` 节点依赖「先收到信号再被暂停」，它在 `Run → Paused` 的这一帧会先看到冻结的树。→ 缓解：`ALWAYS` 表里的节点都不读 `paused` 自身，只读 `state_changed`；`RunFlow` 的冻结在第 1 步显式写入。
- **4.7 的 `--check-only` 与 autoload 行为**：零 autoload 让它不再是问题，但这也意味着若未来引入 autoload，解析检查会静默退化。→ 缓解：`autoload/` 为空写进 Validation Criteria。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `audio-feedback.md` | TR-audio-feedback-005：暂停静默并清空待播结果；静音/无设备是 drop 语义 | D10：`AudioDirector` 在 `ALWAYS` 白名单内；第 4 步同一帧 `stop()` 全部 voice 并清空 `pending_outcome`；不依赖引擎自动停声；静音与无设备不进启动门 |
| `audio-feedback.md` | Open Question 6：暂停时音频处理必须继续运行，`process_mode = ALWAYS` 还是独立单例 | D10：选 `PROCESS_MODE_ALWAYS`，且不是 autoload——由 `AppRoot` 构造并注入；独立单例被 D1 排除 |
| `audio-feedback.md` | TR-audio-feedback-001：事件到 cue 的映射与同帧派发 | D8 第 4 步把 `state_changed` 扇出给 `AudioDirector`，与界面重建同帧；D9 表列出音频订阅的全部事件 |
| `scene-flow.md` | TR-scene-flow-005：`state_changed(from, to)`；切换同帧生效；淡入淡出不承载状态 | D8：`emit` 前写入状态 + 切上下文 + `paused`；六步扇出同步完成；第 6 步才启动过渡 |
| `scene-flow.md` | 规则 15：信号或直接调用、顺序与重入由 ADR 决定 | D6 / D7 / D8：通知用信号，契约用直接调用，`state_changed` 单订户固定扇出，重入深度 ≤ 1 |
| `scene-flow.md` | TR-scene-flow-003：`Boot` 等数据 `Ready/Failed`，失败进 `Error` 且不构造 gameplay | D4 步骤 1–3：构造顺序把 gameplay 查询路径排在 `Data Ready` 之后；失败请求 `Boot → Error` |
| `save-system.md` | 「`Fresh` 时先 `normalize()` 再写基线，并入同一事务」+ TR-save-system-006 | D5：`BootSequence.ensure_baseline` 唯一写者，三个命名空间一次 `commit`；失败保持 dirty 不阻塞菜单 |
| `save-system.md` | Open Question 3：启动序列中由谁调用 `load()` | D4 步骤 4：`AppRoot` 调用恰好一次，`UIRoot` 只查询 `LoadReport`，不发起 `load()` |
| `unlocks-and-stations.md` | 规则 14 / AC 1：新建生涯首个基线携带回填后的集合，不是空集合 | D5：`normalize(persisted)` 的返回值进入基线事务；成功后 `adopt_normalized` 对齐内存 |
| `ui-flow.md` | TR-ui-flow-004 / ADR-0003 D7：收据在 `RunResult` 交付同帧成为根屏，HUD 保留到那一帧 | D8 第 2、3 步的顺序：先 `ui.reset_for_state` 再 `hud.sync_for_state` |
| `ui-flow.md` | 规则 20：`Blocked` 强制模态里「新建生涯（旧档保留）」后刷新界面 | D5 第 4 条：该路径复用 `ensure_baseline`，`UIRoot` 不自己拼基线字典 |
| `run-flow.md` | TR-run-flow-006：`run_*` 信号每 `run_id` 至多一次 | D6 第 4 条 + D9 表的订阅顺序 |
| `run-flow.md` | `run_seed` 公式：`session_rng` 播种方式属于 ADR | D4：`SessionRng` 构造时 `randomize()` 一次，唯一随机源；其余 RNG 调用被禁止 |
| `architecture.md` | 原则 8（组合根优于单例）、原则 9（信号通知 / 直接调用契约）、不变量「处理器不得同步改发射者状态」 | D1 / D2 / D3 / D6 / D7 |
| `architecture.md` | QQ-04、QQ-12 | D7、D8（顺序与重入）/ D5（基线写者） |

## Performance Implications

- **CPU**：启动期一次构造 + 一次数据加载 + 一次 `Save.load()`；`state_changed` 的六步扇出是六次同步调用，发生在用户触发的状态切换帧（一帧至多一次），量级为微秒。信号派发本身是直接调用链，无反射、无字符串查表。
- **Memory**：服务图是固定的一小组对象；`RefCounted` 服务常驻。最大的单项是 `DataConfig` 的目录（ADR-0001：300 条记录 < 2 MB）。
- **Load Time**：`BootStep` 1–5 与 ADR-0001 / ADR-0002 的预算相加（数据 p95 ≤ 250 ms、存档 p95 ≤ 100 ms）；基线写入只在 `Fresh` 发生一次，走同一笔 `commit`（p95 ≤ 50 ms）。稳态启动（已有档）没有基线写入。
- **Frame cost**：暂停白名单不引入每帧工作——`SceneFlow` 的 `ui_pause` 轮询与 `SaveHeartbeat` 的 30 s 心跳是仅有的两个周期性动作。
- **Network**：无。

## Migration Plan

绿地实现，当前没有代码。落地顺序（每步都可单独验收）：

1. `AppRoot` 骨架 + `BootReport` / `BootStep` / `BootError`；先只做步骤 1、8，其余留空返回 `NONE`。
2. `DataConfig`（ADR-0001）与 `MilestoneCatalog`；接步骤 2、3，验证 `Boot → Error`。
3. `SaveSystem` + `SaveHeartbeat`（ADR-0002）；接步骤 4，验证 `Blocked` 不阻断 `Menu`。
4. 三命名空间所有者（`MetaUpgrades` / `Unlocks` / `Records`）的最小默认值 + `BootSequence.ensure_baseline`；用 commit spy 验证「一次事务、三个命名空间、非空解锁集合」。
5. `SceneFlow` + `InputRouter`（ADR-0005）；接步骤 6，验证上下文随状态切换。
6. `UIRoot` + 四宿主 + `HudHost`（ADR-0003）；实现 D8 六步扇出并逐条断言顺序。
7. `AudioDirector` + 暂停静默；验证 `TR-audio-feedback-005`。
8. `SessionRng` 接入 `RunFlow`；`Quit` 关闭序列；全链路无头验收。

**Rollback plan**：本 ADR 决定的都是「谁持有谁」与「谁先跑」，没有持久化格式。回滚方式是改文档 + 改一处装配：若零 autoload 被证伪（例如某个第三方插件强依赖 autoload），把 `AppRoot` 改注册为唯一 autoload，其余签名不变；若六步扇出被证明不够，把第 2、3 步合并为 `PresentationSync` 对象，`state_changed` 仍单订户。两种回滚都不触碰 `Save`、`InputRouter` 或任何 GDD 契约。

## Validation Criteria

- **启动顺序**：`boot()` 返回的 `BootReport.step` 序列等于 `CONSTRUCT → LOAD_DATA → LOAD_MILESTONES → LOAD_SAVE → BASELINE → SCENE_BOOT → UI_BUILD → READY`；数据失败时序列止于 `LOAD_MILESTONES` 且 `error = DATA_FAILED`，`SceneFlow.state == Error`，没有任何 gameplay 查询被调用（F-calls 记录器为空）。
- **零 autoload**：`project.godot` 不含 `autoload/` 段；`run/main_scene` 指向 `res://src/main/main.tscn`；静态扫描 `get_node("/root`、`Engine.get_singleton`、`static var` 命中 0 次。
- **基线唯一写者**：`Fresh` 夹具下 `commit` spy 调用 1 次，载荷同时含 `progression` / `unlocks` / `records`，且 `unlocks.stations` 非空（含免费档口）、`dishes` 非空；`Loaded` 夹具下 `commit` 调用 0 次；注入写失败 → 返回 `BASELINE_COMMIT_FAILED`，三个命名空间均 dirty，`SceneFlow` 仍进入 `Menu`。
- **删除生涯路径**：`delete_career()` 成功后再次调用 `ensure_baseline` → `commit` 1 次、解锁集合非空；`UIRoot` 侧没有拼装任何基线字典（静态扫描）。
- **依赖方向**：`SaveSystem` 源码中不出现 `Unlocks` / `Records` / `MetaUpgrades` 标识符；`unlocks.gd` 中不出现 `commit(`（除了经 `spend` 的 `extra`）。
- **扇出顺序**：为六步各注入一个记录器，断言 `state_changed(Run, Menu, RunResult)` 的调用顺序为 `set_frozen → reset_for_state → sync_for_state → on_flow_state(audio) → on_flow_state(onboarding) → fade.play`，且 `ReceiptScreen` 挂载发生在 `HudHost` 卸载之前（同帧帧号相等）。
- **重入**：在 `state_changed` 处理器里请求 `Menu → Run`，断言 `request_transition` 返回 `false`（或幂等无信号），且该帧 `state_changed` 仍只发一次、`_in_dispatch` 未触发 ERROR。
- **处理器禁令**：静态扫描每个 `connect` 目标，方法体内不出现对发射者的变更方法、`await`、`call_deferred`、`queue_free`。
- **暂停白名单**：遍历场景树断言 `ALWAYS` 节点集合恰好等于 D10 表；`HudHost`、`Run` 场景与 gameplay 节点均为 `INHERIT`。
- **暂停静默**：进入 `Paused` 后同一帧 `active_voice_count == 0` 且 `pending_outcome == null`；暂停期间的请求全部为 `dropped:paused`；恢复后无补播。
- **冻结双闸**：`Paused` 中调用 `RunFlow.submit_action` → `REJECTED(FROZEN)`，回合与容量不变。
- **随机源**：静态扫描 `randi(` / `randf(` / `RandomNumberGenerator.new` 只出现在 `session_rng.gd`；同一 `RunConfig.seed` 两局 `run_seed` 相等，`seed = null` 两局不等。
- **关闭序列**：请求 `Quit` 后记录器顺序为 `flush → release_lock → shutdown → quit`；flush 失败时不重试、不静默丢弃。

## Related

- `docs/architecture/architecture.md` — 原则 8 / 9、不变量、Data Flow 1 / 6、Required ADR #4、Open Questions QQ-04 / QQ-09 / QQ-12
- `docs/architecture/adr-0001-content-data-format.md` — `DataConfig` 状态机、启动门、里程碑目录顺序、`DataConfig` 不是 autoload
- `docs/architecture/adr-0002-persistence-format.md` — `commit` 原子性、所有权表（「正常化调用方」缺口）、心跳必须 `ALWAYS`、Save 不得依赖命名空间
- `docs/architecture/adr-0003-scene-screen-architecture.md` — D1（`UIRoot` 非 autoload）、D2.4（整条 UI 路径 `ALWAYS`）、D7（收据先于 HUD 卸载）、D9（动作唯一入口）、D10（重建与 `LoadReport` 查询）
- `docs/architecture/adr-0005-input-context-stack.md` — `context_changed` 发射时机、`push/pop` 成对、`ui_pause` 由 `SceneFlow` 独占、`ActionSink`
- `design/gdd/save-system.md` — 规则 11 / 17 / 18、边界用例「`Fresh` 基线」、Open Question 3
- `design/gdd/unlocks-and-stations.md` — 规则 6 / 7 / 14、AC 1
- `design/gdd/audio-feedback.md` — 规则 14、状态表、Open Question 6
- `design/gdd/scene-flow.md` — 规则 2 / 3 / 15、Formulas 的 `state_latency_frames`
- `design/gdd/run-flow.md` — 规则 3 / 19、`run_seed` 公式
- `docs/engine-reference/godot/VERSION.md`、`current-best-practices.md`（命令行与 4.7 实测记录）、`breaking-changes.md`、`deprecated-apis.md`

## Open Questions

1. **`state_changed` 的单订户是否会成为瓶颈？** 当前六步全部是界面/音频相关。若 ADR-0007 的结算编排需要「在收据挂载之后、HUD 卸载之前」插入事务，则应把事务作为第 2.5 步加入扇出表，而不是让 `UIRoot` 直接调 `Save`——这条要在 ADR-0007 定稿时回看本 ADR。
2. **`BootError.SERVICE_CONSTRUCTION_FAILED` 在 release 构建的处理**：本 ADR 选择「`push_error` 后退出进程」。若产品希望构造失败时进入可读的错误屏，需要 `SceneFlow` 在 `CONSTRUCT` 之前先构造——那是另一种顺序，属于修订。
3. **`SavedHeartbeat` 的间隔是否与 ADR-0002 的 30 s 心跳一致**：本 ADR 取默认 30 s 并作为构造参数，实际值必须在 ADR-0002 的锁存活判定窗口（90 s）内留出至少三倍余量；若锁窗口调整，这里要跟着改。
4. **未来是否需要 `PROCESS_MODE_ALWAYS` 的第三个用途**（例如在线挑战的会话心跳）：新增必须改 D10 表，不允许就地加 `ALWAYS`。
5. **`SessionRng` 的播种是否应记录进存档**：`run-flow.md` 只要求 `run_seed` 记录在 `run_started` 中，不要求跨会话复现。若要支持「同一种子复现整段生涯」，需要新命名空间，属于 schema 变更。
