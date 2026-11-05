# ADR-0009: 音频播放、voice 池与暂停语义（Audio Playback, Pooling & Pause Semantics）

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
- `godot-specialist`（`AudioServer` 总线拓扑、`AudioStreamPlayer` 在树暂停下的播放行为、设备探测）
- `audio-director`（cue 目录与声音规范的边界：本 ADR 只裁定「放不放、放多响、放几次」，不裁定音色）
- `technical-director` 战略审阅：跳过（`review_mode: lean`）

## Summary

`audio-feedback.md` 把「事件到 cue、准入、抢占、闪避、cue 日志」写得足够细，却没有回答四个会决定代码形状的问题：**音频节点在 `SceneTree.paused` 下怎么活**、**总线图到底有哪几条**、**cue 目录放在哪里、由谁校验**、**没有音频设备时怎么证明它做对了**。这四个问题合起来就是 `architecture.md` 的 `QQ-09` 与 Required ADR #9。

本 ADR 裁定：

1. **音频是组合根下的两个常驻 `Node`，不是 autoload，也不归 `UIRoot`**：`AudioDirector`（`PROCESS_MODE_ALWAYS`）+ `VoicePoolHost`（`PROCESS_MODE_ALWAYS`）。暂停静默**由显式 `stop()` 实现，不由引擎自动暂停实现**。
2. **总线图固定为 `Master / Music / SFX / UI` 四条**，没有 `Ambience`。SFX/UI/Master 的总线音量恒为 0 dB；全部增益在**每个 voice 上**按 GDD 公式计算一次，避免「总线音量 + 公式」双重衰减。`Music` 是唯一的闪避目标，MVP 无音乐，闪避休眠。
3. **voice 池是 10 个启动期预分配、永不释放的 `AudioStreamPlayer`**，按优先级 1/3/3/3 分槽；抢占、50 ms 守卫、冷却、去重全部实现为**纯逻辑 `AudioPolicy`**（`RefCounted`），由 `AudioDirector` 只做引擎适配。`sfx.flow.*` 四个转场 cue 走**池外的单个 `UiTransitionPlayer`**，因此暂停音可以活过冻结帧，而池内 voice 仍然在进入 `Paused` 的同一帧归零。
4. **cue 目录是一个类型化 Resource**：`res://data/audio/cue_catalog.tres`（`AudioCueCatalogResource` 持有 `Array[AudioCueData]`），作为 `DataConfig` 的**第六个显式清单**与其余五个清单同一次 `Loading → Validating → Ready/Failed` 校验、同一次全有或全无发布。事件到 cue 的映射是**代码里的静态 `CueMap`**，在 `bind_catalog()` 时与目录做双向校验；映射缺陷按 `DATA_FAILED` 进 `Boot → Error`，运行期播放失败永不阻断。
5. **无头可测**：`AudioPolicy` 只依赖四个端口（catalog view / bus port / sink / clock）。测试注入 `RecordingSink` + `FixedClock`，不碰 `AudioServer`、不需要场景树，断言的是**与生产完全相同的决策路径**。变体轮转是确定性的且不消耗 RNG；MVP 的 pitch 抖动被校验器强制关闭，因此本 ADR 不引入任何新的随机源。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Audio（`AudioServer` 总线拓扑、`AudioStreamPlayer` 播放/停止、设备探测）；Core（`process_mode`、`_process` 时钟） |
| **Layer** | Foundation（`AudioDirector` / `AudioPolicy`）+ Presentation（cue 目录是呈现数据） |
| **Knowledge Risk** | MEDIUM — `AudioStreamPlayer`、`AudioServer` 的 4.0 起稳定 API 已用多年；但「树暂停时 `ALWAYS` 节点上的 `AudioStreamPlayer` 是否继续推进」「输出设备列表/热插拔」「headless 下 `AudioStream.get_length()`」三处必须在 pin 定的 4.7.x 上实测。`docs/engine-reference/godot/VERSION.md` 把 Audio 标为 MEDIUM |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`breaking-changes.md`、`current-best-practices.md`、`deprecated-apis.md`、`modules/audio.md`；`design/gdd/audio-feedback.md`、`ui-flow.md`、`input.md`、`scene-flow.md`、`run-flow.md`；`docs/architecture/architecture.md`、ADR-0001 / ADR-0003 / ADR-0004 / ADR-0006 / ADR-0008 |
| **Post-Cutoff APIs Used** | 无新增 API。只用 `AudioServer.add_bus / set_bus_name / set_bus_send / set_bus_volume_db / get_bus_count / get_bus_name / get_output_device_list`、`AudioStreamPlayer.play / stop / playing / volume_db / pitch_scale`、`AudioStream.get_length()`。**刻意不用** `AudioStreamPlayer.finished` 做 voice 回收（暂停与抢占下语义未验证），改为每帧轮询 `playing` |
| **Verification Required** | ① `get_tree().paused = true` 后，`PROCESS_MODE_ALWAYS` 节点上的 `AudioStreamPlayer` 是否仍推进播放位置（决定「池外暂停音」能否响完）；② headless / Dummy 驱动下 `AudioStreamPlayer.play()` 是否报错、`playing` 是否返回 `true`（决定 `device_available` 的默认值）；③ `AudioServer.get_output_device_list()` 在 4.7 的返回值形状与热插拔是否发信号（决定设备探测是否只能轮询）；④ `AudioStreamOggVorbis` / `AudioStreamWAV` 在无设备时 `get_length()` 是否等于资源标称时长（决定 `zero_duration` 校验能否在 headless 跑）；⑤ `AudioServer.set_bus_volume_db` 在每帧写入时的线性/分贝语义与 `volume_db` 属性一致 |

> ⚠️ **引擎知识缺口**：4.5–4.7 均在 LLM 训练截止之后，且本机没有安装编辑器。
> 上述五项必须在 pin 定的 4.7.x 上实测。其中 ① 若不成立，`sfx.flow.pause`
> 退化为「随冻结一起消失」并在 cue 日志记 `dropped:paused`，本 ADR 的其余部分不变。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001（`DataConfig` 的 `Loading → Validating → Ready/Failed`、六阶段校验、`Diagnostic` 码、单次全有或全无发布——本 ADR 只是新增第六个清单）；ADR-0003（`UIRoot` 与四个宿主、`ui_button_activated` 唯一入口、整条 UI 路径 `ALWAYS`）；ADR-0004（零 autoload、`AppRoot` 组合根、`state_changed` 六步扇出、`PROCESS_MODE_ALWAYS` 白名单、`Quit` 关闭序列）；ADR-0006（若未来启用 pitch 抖动，只能用 `SeedHash` + `SeededStream`）；ADR-0008（`SettingsModel.audio()` 提供 `master_volume_db` / `sfx_volume_db` / `muted` / `reduce_repetitive_cues`） |
| **Enables** | 音频系统的全部实现 story；`TR-audio-feedback-001..006` 的验收；无头 CI 对 cue 决策的穷举回归；未来音乐（`Music` 总线与闪避已就位，只差音源） |
| **Blocks** | 任何需要「同一帧事件 → 同一帧声音」的 story；任何以 pause 静默为验收前提的 story；任何在 headless 下断言 cue 行为的 story |
| **Ordering Note** | 本 ADR 是 `architecture.md` Required ADRs 的第 9 项，闭合 `QQ-09`。ADR-0004 D10 已经规定「`AudioDirector` + voice 池宿主 `ALWAYS`」「进入 `Paused` 同一帧 `stop()` 全部 voice 并清空 `pending_outcome`」；本 ADR 只把该条的**对象**精确到「池内 voice」，并新增一个不属于池的转场播放器。两者不冲突，但实现时必须同时满足两份文档的验收项 |

## Context

### Problem Statement

`audio-feedback.md` 的可听契约已经完整（cue 表、优先级、上限、冷却、音量公式、闪避、日志词表），但它把四个实现形状的问题留成了 Open Question：

- **OQ6（暂停）**：「暂停时音频处理必须继续运行，用 `process_mode = ALWAYS` 还是独立单例」。
- **OQ7（目录）**：「cue id → 音频流的注册表尚未决定」；`design/gdd/audio-feedback.md:359` 明确「尚无 `design/audio/` 或 sound bible」。
- **OQ2（内容所有权）**：声音规范是外部交付物，本系统只消费映射；但映射以什么格式、放在哪个路径、由谁校验，没有答案。
- **`architecture.md` `QQ-09`（Medium）**：音频 `process_mode` 与 cue 目录数据位置。

同时 `audio-feedback.md` 内部有两处自相矛盾，如果不裁定，实现者只能二选一：**暂停音在进入冻结的同一帧起播，而准入表第 3 步又把 `Frozen` 期间的一切请求丢弃**；**状态表只建 `Master / SFX / UI`，闪避公式却要求 `Music` 与 `Ambience`**。这两处必须在写第一个 cue 之前定死，否则「暂停有没有声音」「闪避作用在哪条总线」会随实现者漂移。

### Current State

没有代码，没有 `res://data/audio/`，没有 `res://assets/audio/`。`DataConfig`（ADR-0001）已有五个清单（dish / upgrade / station / balance / milestone），`MilestoneCatalog` 是「同一 `Diagnostic` 词表、独立加载器、`DataConfig` `Ready` 之后加载」的先例。ADR-0004 已把 `AudioDirector` 写进 `PROCESS_MODE_ALWAYS` 白名单，并规定 `Quit` 序列第 3 步调用 `AudioDirector.shutdown()`。ADR-0008 已把音量与静音放进 `SettingsModel`（机器级，不进生涯存档）。缺的是**音频自己的数据形状、总线图、池与暂停行为**。

### Constraints

- **零 autoload**（ADR-0004 D1）。音频不能是单例，不能按名查找。
- **UI 不拥有音频**（ADR-0003）：`UIRoot` 不得引用 `AudioDirector`，只发语义事件。
- **数据是类型化 Resource + 显式清单**（ADR-0001）：不得运行时 `DirAccess` 扫描，不得在 GDScript 里写平衡字面量。
- **发布是全有或全无**（ADR-0001）：不得让音频先拿到一半 cue。
- **播放失败永不阻断**（`audio-feedback.md:21`）：最坏结果是安静，不是卡住或崩溃。
- **暂停是 `SceneTree.paused`**（ADR-0004 D10）：不得自造第二套冻结机制。
- **不新增随机源**（ADR-0004 D4 / ADR-0006）：服务内不得出现 `randi()` / `randf()` / `RandomNumberGenerator.new()`。
- **`MAX_VOICES = 10`、优先级上限 `1/3/3/3`、`MIN_STEAL_AGE_MS = 50`、`[-80, 0]` 钳制是固定不变量**（`audio-feedback.md:150`），本 ADR 不得改数。

### Requirements

1. 事件在到达的同一帧发出声音；跨系统只经信号/直接调用，不反查状态（`TR-audio-feedback-001`）。
2. 并发上限、抢占与 50 ms 守卫必须可穷举断言（`TR-audio-feedback-002`）。
3. 变体轮转确定性、冷却与每 `run_id` 去重（`TR-audio-feedback-003`）。
4. 音量映射恰好一次，钳制到 `[-80, 0]`（`TR-audio-feedback-004`）。
5. 暂停静默、清空 `pending_outcome`、静音/无设备是 drop（`TR-audio-feedback-005`）。
6. 1 万条 cue 日志与封闭原因词表（`TR-audio-feedback-006`）。
7. 无头 CI 能在没有音频设备的机器上断言上述全部行为。

## Decision

### D1 — 组合位置：`AudioDirector` + `VoicePoolHost`，由 `AppRoot` 构造，零 autoload

`AudioDirector`（`Node`）与 `VoicePoolHost`（`Node`，`AudioDirector` 的子节点）由 `AppRoot` 在 `CONSTRUCT` 步骤构造并 `add_child()`，两者的 `process_mode = PROCESS_MODE_ALWAYS`。`AudioDirector` **不是** autoload，**不是** `UIRoot` 的子节点，**不**被任何屏幕或 HUD 持有。

`AudioDirector` 只做三件事：把引擎事件翻译成 `AudioPolicy` 的输入、把 `AudioPolicy` 的输出写给 `AudioSink`、在 `_process` 里推进时钟。**全部判定逻辑在 `AudioPolicy`（`RefCounted`）里**，它不认识 `AudioServer`、不认识场景树、不读系统时钟。

依赖由构造参数注入，方向固定为：

```
AudioDirector ──> AudioPolicy ──> AudioCatalogView（DataConfig 的只读适配器）
      │                │
      │                ├──> AudioBusPort（生产：AudioServer；测试：RecordingBusPort）
      │                └──> AudioSink（生产：GodotAudioSink；测试：RecordingSink）
      └──> Clock（生产：Time.get_ticks_msec；测试：FixedClock）
```

`UIRoot`、HUD、`SceneFlow`、`RunFlow` **都不持有** `AudioDirector`。它们只发信号；`AppRoot` 在固定扇出里调用音频（ADR-0004 D8 第 4 步）。

### D2 — 总线图：`Master / Music / SFX / UI`，四条，没有 `Ambience`

| 总线 | 父总线 | 总线音量 | 用途 | 是否可被闪避 |
|---|---|---|---|---|
| `Master` | —（根） | 0 dB，**永不写入** | 汇总 | 否 |
| `Music` | `Master` | 0 dB，仅由闪避包络写入 | 未来背景音乐 | **是**（唯一目标） |
| `SFX` | `Master` | 0 dB，**永不写入** | 世界内音效 | 否 |
| `UI` | `Master` | 0 dB，**永不写入** | 界面与转场 | 否 |

三条关键裁定：

1. **`bus_offset_db` 在 voice 上应用，不在总线上应用。** GDD 的音量公式已经含 `bus_offset_db`（SFX 0、UI −3）。若再把 `UI` 总线设成 −3 dB，就会双重衰减 6 dB。因此四条总线的音量都钉在 0 dB，`UI` 的 −3 dB 只出现在 `AudioPolicy` 的公式里——它也是无头测试唯一需要断言的常量。
2. **没有 `Ambience` 总线。** GDD 明说 Vertical Slice 与 MVP 不做环境声（`audio-feedback.md:81`、`:420`、`:442`）。为一条永远为空的母线增加拓扑、闪避分支与验收项，只增加噪音。未来若加入环境声，它走 `SFX` 或新增总线，属于修订。
3. **`Music` 在 MVP 里存在但恒为 0 dB、无播放器。** 提前建出来是为了让闪避公式的验收对象存在，且避免未来加音乐时改总线图。MVP 不创建 `MusicPlayer` 节点；`Music` 总线的写入是一条可断言的空操作。

`AudioDirector.ensure_buses()` 在 `CONSTRUCT` 幂等地建立拓扑：按名查找（`get_bus_count` / `get_bus_name`），缺失则按 `Music → SFX → UI` 的固定顺序 `add_bus()` 到末尾并 `set_bus_send(idx, "Master")`，已存在则接受其位置并记一条 INFO。`project.godot` 的 `audio/buses/default_bus_layout` **可以**指向同一布局，但**不**是权威；代码路径是权威，测试因此不需要加载项目设置。

### D3 — 暂停：`PROCESS_MODE_ALWAYS` + 显式 `stop()`；禁止 `WHEN_PAUSED`；暂停语义是「安静」

**唯一冻结机制是 `get_tree().paused`**（ADR-0004 D10）。音频的四个节点全部 `PROCESS_MODE_ALWAYS`：

| 节点 | `process_mode` | 理由 |
|---|---|---|
| `AudioDirector` | `ALWAYS` | 必须在暂停中收到 `state_changed(Paused→Run)` 并发出恢复音；`Quit` 也发生在暂停中 |
| `VoicePoolHost` | `ALWAYS` | 必须在进入 `Paused` 的同一帧执行 `stop()`；拆除淡出与 voice 回收在暂停中也必须推进 |
| `UiTransitionPlayer` | `ALWAYS` | 唯一允许活过冻结帧的声音（见 D4） |
| `MusicPlayer`（未来） | `ALWAYS`，但暂停时由代码写入 `Music` 总线静音 | 与池一致，不依赖引擎自动暂停 |
| 其余节点 | `INHERIT` | 默认冻结 |

**项目内不出现 `PROCESS_MODE_WHEN_PAUSED`。** 它会在恢复的那一帧停止处理，正好错过 `Paused → Run` 的恢复音；而且它让「暂停中谁在跑」变成两张表而不是一张。音频的暂停语义只有一句话：**进入 `Paused` 就是安静，离开 `Paused` 才有下一声。**

进入 `Paused` 的同一帧（ADR-0004 D8 第 4 步，`AudioDirector.on_flow_state(from, Paused)`）：

1. `state = Frozen`，`accepting = false`；
2. 对池内 10 个 voice 逐个 `stop()`（**0 ms，无淡出**——GDD 要求 1 帧内静默）；
3. 清空 `pending_outcome`、清空本帧的结果 cue 队列；
4. `play_transition(&"sfx.flow.pause")`（唯一例外，见 D4）。

暂停期间的池请求：**全部 `dropped:paused`，不排队、不延后、不补播**、不更新 `last_play`。包括 `PauseOverlay` 与设置模态里的按钮点击——GDD 的准入表第 3 步如此规定，本 ADR 不开口子。

离开 `Paused` 的同一帧：`accepting = true`、`state = Ready`；若目标是 `Run` 则发 `sfx.flow.resume`，若目标是 `Menu` 则发 `sfx.flow.menu`，若目标是 `Quit` 则不发（由 `shutdown()` 处理）。**恢复音在解冻之后、不在之前**，否则它会和暂停音在同一个冻结帧里重叠。

### D4 — cue 分类：池内 voice 与池外转场，暂停音的唯一例外

每个 cue 在目录里有两个正交的分类字段：`voice_class ∈ {POOL, TRANSITION}` 与 `pause_policy ∈ {STOP_ON_PAUSE, PLAY_THROUGH_FREEZE}`。

| 类 | 成员 | 承载者 | 计入 `MAX_VOICES` | 暂停行为 |
|---|---|---|---|---|
| `POOL` | 除 `sfx.flow.*` 外的全部 cue | 10 个池 voice | 是 | 进入 `Paused` 同一帧 `stop()`，`dropped:paused` |
| `TRANSITION` | `sfx.flow.pause` / `resume` / `menu` / `quit` | `UiTransitionPlayer`（1 个，池外） | **否** | 只有 `pause` 是 `PLAY_THROUGH_FREEZE`；其余在解冻帧播放 |

规则：

- `TRANSITION` cue 走 `UI` 总线、优先级恒为 `UI`、遵守自己的冷却（`pause`/`resume` 200 ms，`menu` 300 ms）与每会话上限（`quit` 1 次）。
- 同一时刻只有一个转场 voice：新的转场请求先 `stop()` 旧的，再起新的。它的 `max_duration_ms` 硬上限为 **250 ms**（新结构性常量，见 D6）；超出即 `stop()`，不循环。
- `TRANSITION` **不参与** 抢占、不参与 `MAX_VOICES`、不进 `pool_full` 判定。它的日志条目 `voice_id = 10`（池外哨兵值），`active_count` 仍只统计池内 voice。
- 这解决了 GDD 的自相矛盾：`sfx.flow.pause` 的「进入 `Paused` 同帧起播」由池外播放器满足；池内 voice 仍在同一帧归零，满足 ADR-0004 D10 的验收项「`active_voice_count == 0`」——`active_voice_count()` **只**返回池内数量，转场状态由独立的 `transition_voice_active()` 暴露。

`PLAY_THROUGH_FREEZE` 只授予 `sfx.flow.pause` 一个 cue，且由目录校验器强制：任何其他 cue 声明该策略都是 `AUDIO_CUE_BAD_RANGE`。新增豁免必须改本 ADR。

### D5 — voice 池：10 个预分配播放器，优先级 1/3/3/3，准入表逐条实现

`VoicePoolHost` 在 `CONSTRUCT` 一次性创建 **10 个** `AudioStreamPlayer`（`voice_id = 0..9`），全部 `bus = &"SFX"` 或 `&"UI"`（起播时按 cue 设置）、全部 `process_mode = ALWAYS`、**永不 `queue_free()`**。池的容量与 GDD 的 `MAX_VOICES` 是同一个常量 `AudioPolicy.MAX_VOICES = 10`。

优先级枚举与上限（固定不变量，不从数据读）：

| 优先级 | 值 | 上限 | 典型 cue |
|---|---|---|---|
| `CRITICAL` | 4 | 1 | `sfx.settle.broke_even` / `sfx.settle.loss` |
| `LIFECYCLE` | 3 | 3 | `sfx.run.start` / `sfx.run.end` / `sfx.settle.print` / `sfx.career.complete` |
| `GAMEPLAY` | 2 | 3 | `sfx.eat.commit` / `sfx.wave.skip` / `sfx.num.tick` |
| `UI` | 1 | 3 | `sfx.ui.click` / `sfx.milestone` / `sfx.record.broken` |

**准入按 GDD 的九步顺序执行，先命中者生效**，且每一步都返回一个 `CueDecision`（`admitted` / `dropped` / `suppressed` / `deferred` + 原因）：

1. `state != Ready` → `dropped:not_ready`（不排队）。
2. `device_available == false` → `dropped:no_device`。
3. `muted` 或 `volume_db == -80` → `dropped:muted`。
4. `pause_policy == STOP_ON_PAUSE` 且 `state == Frozen` → `dropped:paused`。
5. `once_per_run` 且该 `run_id` 已发过 → `dropped:dedupe`；`dedupe_key`（同一 `action_id` 每帧一次）重复 → `dropped:dedupe`。
6. `now - last_play[cue] < cooldown_ms` → `dropped:cooldown`。
7. 该优先级 `voices_used(p) < cap[p]` **且** 存在空闲 voice → `admitted`（分配最小空闲 `voice_id`）。
8. 请求为 `CRITICAL` 且存在 `age ≥ 50 ms` 的 `CRITICAL` voice → `admitted` + 抢占最旧者。
9. 存在优先级**严格更低**且 `age ≥ 50 ms` 的 voice，**且抢占后 `voices_used(p) < cap[p]`** → `admitted` + 抢占（先取最低优先级，同级取最旧，再取最小 `voice_id`）。
10. 其他 → `dropped:pool_full`。

第 9 步的 `voices_used(p) < cap[p]` 是对 GDD 准入表的**澄清而非改数**：GDD 把 `cap[p]` 定义为「每级上限」，如果不加这个守卫，一个已达上限的 Gameplay 请求可以抢掉 UI voice 后让 Gameplay 变成 4 个，直接违反固定不变量。本 ADR 选择「上限是硬顶，抢占也必须满足」。这条澄清写进 Consequences，建议同步修订 GDD 的准入表。

`priority` 这个原因码在 v1 **不会被发出**（所有拒绝都由 `pool_full` 或 `cooldown`/`muted` 覆盖）；它保留在封闭词表里，删除或启用都需要先改 GDD。

**每 cue 的 `max_voices`**：目录字段，取值 `1..cap[priority]`，默认 1。它只收紧、不放大：`eat.commit` 为 3（对应 GDD「同一帧两个吃菜都准入」），其余为 1。校验器拒绝任何大于 `cap[priority]` 的值，因此**它永远不会先于 GDD 的优先级上限生效**，MVP 的可听行为与 GDD 完全一致。

### D6 — 抢占、释放与拆除：手动分贝斜坡，不用 `Tween`

- **抢占**：立即 `stop()` 被抢 voice，记 `stolen_voice_id`；不做淡出（GDD 的 50 ms 守卫已经保证不会截断起播瞬间）。
- **自然结束**：`AudioDirector._process` 每帧轮询 `sink.active(voice_id)`，`false` 即归还 voice。**不用 `AudioStreamPlayer.finished`**——它在暂停、抢占和设备切换下的发射时机未验证。
- **拆除淡出**：仅在三种「有下一帧」的场景使用——`RUN_TEARDOWN`（`run_ended` / `run_faulted` / `run_discarded` 之后的结果 cue 收尾）、`STATE_BOUND_EXIT`（GDD 规则 21：离开产生该 cue 的状态）、`MUTE_TOGGLE`。淡出为 **40 ms** 的**手动分贝斜坡**（每帧 `volume_db` 线性下降 40 ms 后 `stop()`），由 `VoicePoolHost._process` 推进。新增常量 `TEARDOWN_FADE_MS = 40`：大于 20 ms 的可听咔哒阈值，小于 `MIN_STEAL_AGE_MS = 50`，因此不会与抢占守卫的计数混淆。**不用 `Tween`**：`Tween` 在 `ALWAYS` 节点上是否随树暂停推进是 ADR-0004 的 Verification ①，音频不能把拆除正确性押在它上面。
- **暂停**：`0 ms` 硬停（D3）。
- **`Quit`**：`0 ms` 硬停；`sfx.flow.quit` 被 `shutdown()` 截断是可接受的（不承诺响完）。
- **`run_faulted`**：按 GDD 属于**取消**，不是抢占：取消待播与正在播放的结果 cue，不受 50 ms 守卫限制，记 `suppressed:preempted`。

### D7 — cue 目录：`res://data/audio/cue_catalog.tres`，`AudioCueData` 一条一个子资源

**新目录 `res://data/audio/`**（不是 `design/audio/`）。目录只有一个文件：

```
res://data/audio/cue_catalog.tres      # AudioCueCatalogResource
```

`AudioCueCatalogResource`（`Resource`）字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `schema_version` | `int` | 固定 `1`；不等即 `AUDIO_CUE_SCHEMA` |
| `cues` | `Array[AudioCueData]` | 全部 cue，id 唯一 |
| `duck_db` | `float` | 6…15，默认 12（GDD 默认值） |
| `duck_attack_ms` | `int` | 30…80，默认 50 |
| `duck_release_ms` | `int` | 80…150，默认 120 |

`AudioCueData`（`Resource`）字段：

| 字段 | 类型 | 取值 / 默认 | 说明 |
|---|---|---|---|
| `id` | `StringName` | `sfx.<domain>.<event>` | 与 GDD cue 表逐字一致 |
| `streams` | `Array[AudioStream]` | 1…4 个 | 变体；外部 `.ogg` / `.wav`，随 `.tres` 依赖图预载 |
| `bus` | `StringName` | `&"SFX"` / `&"UI"` | 仅这两条；`Music` 不可被 cue 直接使用 |
| `base_db` | `float` | −40…0 | GDD 音量公式的 `base_db[c]` |
| `pitch_min` / `pitch_max` | `float` | MVP 必须都为 `1.0` | 见 D11 |
| `max_voices` | `int` | 1…`cap[priority]`，默认 1 | 见 D5 |
| `priority` | `int` | 1…4 | 枚举，不从数据推断 |
| `voice_class` | `int` | `POOL` / `TRANSITION` | 见 D4 |
| `pause_policy` | `int` | `STOP_ON_PAUSE` / `PLAY_THROUGH_FREEZE` | 后者仅 `sfx.flow.pause` |
| `cooldown_ms` | `int` | 0…300 | GDD 默认表逐条落值 |
| `once_per_run` | `bool` | 默认 `false` | 结果 cue、`run_*` |
| `once_per_session` | `bool` | 默认 `false` | `career.complete` / `flow.quit` |
| `dedupe_same_action_frame` | `bool` | 默认 `false` | `eat.commit` 为 `true` |
| `state_bound` | `bool` | 默认 `false` | GDD 规则 21，离开状态即取消 |
| `duck` | `bool` | 默认 `false` | 仅 `settlement_started` / `broke_even` / `loss` |
| `result_cue` | `bool` | 默认 `false` | 进入 `pending_outcome` 状态机 |
| `max_duration_ms` | `int` | 1…5000 | 校验与日志用；`TRANSITION` 上限 250 |
| `expected_length_ms` | `int` | `> 0` | 与 `AudioStream.get_length()` 比对，容差 ±50 ms |

**校验规则**（全部 `ERROR`，任一命中即整份目录不发布；沿用 ADR-0001 的 `Diagnostic` 形状）：

- `AUDIO_CUE_DUPLICATE_ID`：id 重复。
- `AUDIO_CUE_UNKNOWN_BUS`：`bus` 不是 `SFX` / `UI`。
- `AUDIO_CUE_BAD_RANGE`：任何数值越界；`pitch_min > pitch_max`；MVP 下两者不都为 1.0；`max_voices > cap[priority]`；`PLAY_THROUGH_FREEZE` 授予非 `sfx.flow.pause` 的 cue；`loop` 为 `true`（MVP 无循环 cue）。
- `AUDIO_CUE_NO_STREAM`：`streams` 为空。
- `AUDIO_CUE_STREAM_MISSING`：引用的 `AudioStream` 为 `null`（资源缺失）。
- `AUDIO_CUE_ZERO_DURATION`：`get_length() == 0` 或与 `expected_length_ms` 偏差超过 ±50 ms。
- `AUDIO_CUE_BAD_PRIORITY`：`priority` 不在 1…4。

**事件到 cue 的映射不放在数据里。** 映射是 `res://src/audio/audio_cue_map.gd` 的静态 `CueMap`：`Dictionary[StringName, StringName]`，把上游事件（`run_started`、`action_committed(eat)`、`ui_button_activated(control_id)`、`state_changed` 等）映射到 cue id。理由：事件名是代码契约（信号名、枚举），把字符串映射放进 `.tres` 只会在重命名信号时静默失效；而**发什么声**是内容，已经在目录里。`CueMap` 在 `bind_catalog()` 时双向校验：每个映射目标必须存在（`AUDIO_CUE_UNMAPPED`），每个 `POOL` cue 必须至少被一个事件引用（`AUDIO_CUE_ORPHAN`），`TRANSITION` cue 只能映射到 `state_changed` 的四个顶层转移。

### D8 — 与 `DataConfig` 集成：第六个清单，同一次校验、同一次发布

`DataConfig` 的清单表新增一项：

```gdscript
const CUE_CATALOG := "res://data/audio/cue_catalog.tres"
```

它进入与其余五个清单**完全相同**的 `Loading → Validating → Ready/Failed` 流程：`Loading` 阶段一次性 `ResourceLoader.load()`（同步，ADR-0004 D4 禁止启动期异步），`Validating` 阶段跑 D7 的八条规则（作为发布阶段之前的独立子阶段 5b），任 `ERROR` → 整份 `DataConfig` 停在 `Failed`，**不发布任何部分**。发布后只读：

```gdscript
DataConfig.get_cue(id: StringName) -> DataResult      # value = AudioCueData
DataConfig.has_cue(id: StringName) -> bool
DataConfig.cue_catalog() -> AudioCueCatalogResource   # READY only
```

`AudioDirector` **不直接持有 `DataConfig`**，而是持有一个 `AudioCatalogView` 适配器（`RefCounted`，只暴露 `get_cue` / `has_cue` / `duck_params` / `schema_ok`）。这样 `AudioPolicy` 在测试里可以用一个字典桩实现同一接口，完全脱离 `DataConfig`。

**两次失败，两种后果，必须分清：**

| 失败 | 时机 | 后果 |
|---|---|---|
| cue 目录内容非法 | `DataConfig` 校验（启动） | `DATA_FAILED` → `Boot → Error`；内容缺陷必须在写 story 前被夹具抓住 |
| `CueMap` 与目录不一致 | `AudioDirector.bind_catalog()`（启动） | `bind_report.error != NONE` → `AppRoot` 按 `BootError.DATA_FAILED` 处理（复用 ADR-0004 的封闭枚举，不新增值）→ `Boot → Error` |
| 起播失败、设备消失、流解码失败 | 运行期 | 记日志（`no_device` / `no_stream` / `error`），**继续运行**，不回退占位音、不阻断任何流程 |

这保持了 ADR-0001 的「单次校验、全有或全无」，也保持了 GDD 的「音频运行期失败永不阻断」。两者不是矛盾：**内容缺陷是构建期错误，播放失败是运行期噪声。**

### D9 — 音量、静音与设置：公式只算一次，总线恒为 0 dB

`AudioPolicy` 对每个被准入的 voice 计算恰好一次：

```
volume_db = clamp(base_db + bus_offset_db + master_gain_db + sfx_gain_db, -80, 0)

bus_offset_db  = 0.0   (SFX)  |  -3.0 (UI)
master_gain_db = SettingsModel.audio().master_volume_db     # -80…0
sfx_gain_db    = SettingsModel.audio().sfx_volume_db        # -40…0，作用于 SFX 与 UI
```

- 结果 `== -80` → 按 GDD 走 `dropped:muted`，不分配 voice（在准入第 3 步短路，不进入第 7 步）。
- **`Master / SFX / UI` 三条总线的音量恒为 0 dB，运行期永不写入。** 只有 `Music` 总线会被闪避包络写入。这样「响度」只有一个来源（voice 的 `volume_db`），无头测试不需要读 `AudioServer` 就能断言 GDD 公式。
- 设置变化**不回溯**已在播放的 voice；下一次起播生效。例外是 `muted` 从 `false → true`：立即 `0 ms` 停掉全部池 voice 与转场 voice（静音必须立刻静音），记 `muted` 不记 `paused`。
- `reduce_repetitive_cues` 只影响 `sfx.num.tick` 的密度（GDD 归 HUD 的滚动时间表），音频侧不额外降音量、不改冷却。
- 音量与静音属于 `SettingsModel`（ADR-0008），**不进生涯存档**；音频不写任何持久化。

### D10 — 闪避：只写 `Music` 总线，MVP 休眠，包络由注入时钟推进

`duck_db(t) = -DUCK_DB * env(t)`；`env` 在任一 `duck = true` 的 cue **处于活跃播放**时为 1，否则为 0；0→1 用 `ATTACK_MS` 线性、1→0 用 `RELEASE_MS` 线性。参数从目录根字段读（GDD 默认 12 dB / 50 ms / 120 ms），校验范围与 GDD 表一致。

- 触发集合是 **`sfx.settle.print`（`settlement_started`）、`sfx.settle.broke_even`、`sfx.settle.loss`** 三个 cue，即 GDD 的 `duck_db` 公式所写的集合。GDD 规则 18 说「Critical cue 播放时闪避」，但公式与变量表写的是这三个事件；本 ADR 以**公式**为准（见 Consequences 的矛盾 #2）。
- **只写 `Music` 总线**，SFX/UI 永不闪避。**没有 `Ambience`**。
- `env` 由 `AudioDirector._process` 用注入时钟推进，每帧最多一次 `set_bus_volume_db(Music, -DUCK_DB * env)`；`env` 未变化时不写入。每帧成本是一次比较。
- 暂停/停止/`shutdown`：直接把 `Music` 写回 0 dB（一次写入，不做斜坡）——GDD 要求「立即以 50 ms 释放」，但在本作里音乐不存在，写入 0 与斜坡在可听上等价，且少一条暂停期的时间依赖。
- **MVP 的 `Music` 总线没有播放器，因此闪避是「可断言但不可听」的**：测试断言总线写入序列，QA 的听感验收（GDD AC 22）在加入音乐前**无法执行**。这不是实现偷懒，是 GDD 的 AC 22 与「VS 无音乐」之间的直接冲突（见 Consequences 的矛盾 #8）。

### D11 — 确定性：变体轮转确定、pitch 抖动在 MVP 被禁用、绝不碰 `SessionRng`

- **变体选择**：每个 cue 维护 `play_index`，只在**成功起播**时 `+1`，取 `streams[play_index % streams.size()]`。`once_per_run` / 结果 cue 的 `play_index` 在 `run_started` 时重置；UI cue 按会话累加。给定同一事件序列，变体序列唯一。**不抽随机数。**
- **pitch**：`number_tick_pitch` 是 GDD 的闭式公式（每 tick 由 HUD 传入的 `start/end/ticks/i` 决定），**不抽随机数**。其余 cue 的 `pitch_min == pitch_max == 1.0`，由目录校验器强制；因此 MVP 的 pitch 没有任何随机性，`pitch` 字段在日志里恒为 `1.0`（数字 tick 除外）。
- **如果未来启用 pitch 抖动**，唯一允许的写法是：`seed = SeedHash.candidate_seed(cue_id, run_seed, play_index)`，用 `SeededStream` 取值映射到 `[pitch_min, pitch_max]`；**禁止** `randf()` / `randi()` / `randi_range()` / `Array.shuffle()`，**禁止**消耗 `SessionRng.next_u64()`（那会让未来的 `run_seed` 依赖音频事件数，直接破坏 ADR-0006 的复现承诺）。启用抖动需要同时改：目录校验器（允许 `pitch_min < pitch_max`）、cue 日志语义（`pitch` 对所有 cue 有意义）、以及本条。这是刻意的摩擦。
- 音频**不创建**任何 `RandomNumberGenerator`，也不持有 `SessionRng`。

### D12 — 可观测性：1 万条环形日志，字段与词表照抄 GDD，另加两个测试专用视图

`CueLogEntry`（`RefCounted`）字段与 GDD 的日志表逐字一致：`frame`、`time_ms`、`cue_id`、`source_event`、`run_id`、`priority`、`decision`、`reason`、`stolen_voice_id`、`voice_id`、`active_count`、`variant`、`volume_db`、`pitch`。环形缓冲 **10,000** 条，覆盖最旧并记一次 WARN。原因词表是**封闭枚举**，字符串常量集中在 `AudioLogReason`，新增必须先改 GDD。

对测试额外暴露（只读，返回副本）：

```gdscript
AudioDirector.cue_log() -> Array[CueLogEntry]        # 时间顺序，最多 10_000
AudioDirector.cue_log_since(seq: int) -> Array[CueLogEntry]
AudioDirector.active_voice_count() -> int             # 池内，0…10
AudioDirector.transition_voice_active() -> bool
AudioDirector.pending_outcome() -> StringName         # 空串表示无
AudioDirector.device_available() -> bool
AudioDirector.duck_gain_db() -> float                 # 当前写入 Music 的值
AudioDirector.state() -> int                          # UNINITIALIZED/READY/FROZEN/SHUTDOWN
```

`cue_log()` 是音频验收的唯一证据来源：headless、无设备、release 构建都记录，测试不读屏、不读 `AudioServer`。

### D13 — 无头测试：四个端口，`RecordingSink` 走与生产完全相同的决策路径

`AudioPolicy` 的构造签名只有端口：

```gdscript
AudioPolicy.new(catalog: AudioCatalogView, sink: AudioSink, clock: Clock,
                bus: AudioBusPort, settings: AudioSettingsView)
```

| 端口 | 生产实现 | 测试实现 |
|---|---|---|
| `AudioCatalogView` | `DataConfigAudioCatalog`（包 `DataConfig`） | `StubCatalog`（字典） |
| `AudioSink` | `GodotAudioSink`（持有 11 个 `AudioStreamPlayer`） | `RecordingSink`（记录 `start/stop/set_bus` 命令，不碰引擎） |
| `AudioBusPort` | `AudioServerBusPort` | `RecordingBusPort`（记录总线写入） |
| `Clock` | `EngineClock`（`Time.get_ticks_msec()`） | `FixedClock`（`advance(ms)`） |
| `AudioSettingsView` | `SettingsAudioView`（包 `SettingsModel.audio()`） | `StubSettings` |

`RecordingSink.device_available` 默认 `true`，可切成 `false` 以覆盖 `no_device`。`RecordingSink.start()` 永远返回 `true`，因此 **headless CI 不安装音频驱动也能跑完整个 cue 决策矩阵**。`AudioDirector` 本身只是一个 `Node` 适配器：`_process` 调 `policy.tick(now_ms)` 与 `sink.poll()`，信号回调调 `policy.request(event, payload)`。**生产与测试共用 `AudioPolicy`**，测试不是「另一套实现」。

启动门：`AudioDirector.bind_catalog()` 之前的一切请求记 `not_ready`；`bind_catalog()` 之后 `state = Ready`。

### D14 — 启动、降级与关闭

- **构造**（`CONSTRUCT`）：`ensure_buses()` + 建 10 个池 voice + 1 个转场 voice；不加载流、不播放。
- **绑定**（`LOAD_DATA` 之后）：`bind_catalog()` 做 `CueMap` 双向校验；失败 → `BootError.DATA_FAILED` → `Boot → Error`。
- **就绪**：`DataConfig` `Ready` 且绑定成功 → `Ready`；`Boot → Menu` 之前音频已经可以接收请求。
- **降级**：设备消失 → `device_available = false`，所有请求 `dropped:no_device`，`state` 仍是 `Ready`；设备回来 → 下一次请求正常准入（轮询探测，见 Verification ③）。**任何运行期音频故障都不改变 `SceneFlow` 状态。**
- **关闭**（`Quit`，ADR-0004 D11 第 3 步）：`shutdown()` 幂等——`state = Shutdown`、`0 ms` 停全部 voice、清空 `pending_outcome`、把 `Music` 写回 0 dB、拒绝后续请求（`not_ready`）。不释放 `AudioStreamPlayer`（随场景树销毁）。

## Architecture Diagram

```
AppRoot (Node, 唯一组合根)
  ├── AudioDirector (Node, ALWAYS) ─────────────── 唯一音频入口
  │     ├── AudioPolicy (RefCounted) ───────────── 全部判定，无引擎依赖
  │     │     ├── AudioCatalogView ──> DataConfig.get_cue()（只读）
  │     │     ├── AudioSink ─────────> GodotAudioSink / RecordingSink
  │     │     ├── AudioBusPort ──────> AudioServer / RecordingBusPort
  │     │     ├── Clock ─────────────> Time.get_ticks_msec / FixedClock
  │     │     └── AudioSettingsView ─> SettingsModel.audio()
  │     └── VoicePoolHost (Node, ALWAYS)
  │           ├── voice 0..9  AudioStreamPlayer   （MAX_VOICES = 10）
  │           └── voice 10    UiTransitionPlayer  （池外，sfx.flow.*）
  ├── UIRoot (ALWAYS)      ── ui_button_activated / receipt_closed ──┐
  ├── RunFlow (RefCounted) ── run_* / action_committed ──────────────┤
  ├── Settlement           ── settlement_started / broke_even / loss ─┤
  └── SceneFlow (ALWAYS)   ── state_changed ──> AppRoot 六步扇出 ─────┘
                                                    │ 第 4 步
                                                    ▼
                                        AudioDirector.on_flow_state()

总线：Master ── Music (仅闪避写入)
             ├─ SFX  (0 dB，永不写)
             └─ UI   (0 dB，永不写；−3 dB 在 voice 公式里)
```

## Key Interfaces

```gdscript
# res://src/audio/audio_director.gd
class_name AudioDirector
extends Node

enum AudioState { UNINITIALIZED, READY, FROZEN, SHUTDOWN }
enum Decision { ADMITTED, DEFERRED, DROPPED, SUPPRESSED }

func configure(policy: AudioPolicy) -> void          # 构造期一次性注入
func ensure_buses() -> int                            # 0 = OK；幂等
func bind_catalog(map: CueMap) -> BindReport          # LOAD_DATA 之后；失败映射 DATA_FAILED
func on_flow_state(from: int, to: int) -> void         # AppRoot 扇出第 4 步
func request(event: StringName, payload: Dictionary) -> int
func play(cue_id: StringName, context: CueContext) -> int   # 测试与内部共用
func on_settings_changed(audio: AudioSettings) -> void
func shutdown() -> void                               # 幂等；Quit 第 3 步

func cue_log() -> Array[CueLogEntry]
func cue_log_since(seq: int) -> Array[CueLogEntry]
func active_voice_count() -> int                      # 池内 0…10
func transition_voice_active() -> bool
func pending_outcome() -> StringName
func device_available() -> bool
func duck_gain_db() -> float
func state() -> int


# res://src/audio/audio_policy.gd —— 纯逻辑，无 Node、无 AudioServer、无系统时钟
class_name AudioPolicy
extends RefCounted

const MAX_VOICES := 10
const CAP := {4: 1, 3: 3, 2: 3, 1: 3}
const MIN_STEAL_AGE_MS := 50
const TEARDOWN_FADE_MS := 40
const TRANSITION_MAX_MS := 250
const UI_BUS_OFFSET_DB := -3.0

func request(cue_id: StringName, context: CueContext) -> CueDecision
func tick(now_ms: int) -> void                        # 回收 voice、推进淡出与闪避包络
func active_count() -> int
func pending_outcome() -> StringName
func duck_gain_db() -> float


# 端口（生产与测试的唯一接缝）
class AudioSink extends RefCounted:
    func start(voice_id: int, stream: AudioStream, bus: StringName,
               volume_db: float, pitch: float) -> bool
    func stop(voice_id: int) -> void
    func set_bus_db(bus: StringName, db: float) -> void
    func active(voice_id: int) -> bool
    func device_available() -> bool

class AudioBusPort extends RefCounted:
    func ensure(bus: StringName, parent: StringName) -> int
    func set_db(bus: StringName, db: float) -> void

class Clock extends RefCounted:
    func now_ms() -> int
```

`CueDecision` 是不可变值对象：`decision`、`reason`、`voice_id`、`stolen_voice_id`、`variant`、`volume_db`、`active_count`。

## Implementation Guidelines

1. **必须**把 `AudioDirector` 与 `VoicePoolHost` 挂在 `AppRoot` 下并设 `PROCESS_MODE_ALWAYS`；**禁止** autoload、`get_node("/root/...")`、`static var` 服务。
2. **必须**让 `AudioPolicy` 保持 `RefCounted` 且零引擎依赖（只有端口）；**禁止**在 policy 里 `get_tree()`、读 `Time`、读 `AudioServer`、加载资源。
3. **必须**在 `CONSTRUCT` 一次性建 10 个池 voice 与 1 个转场 voice；**禁止**在播放路径上 `AudioStreamPlayer.new()` 或 `queue_free()`。
4. **必须**让四条总线存在且 `Master/SFX/UI` 音量恒为 0 dB；**禁止**把 `bus_offset_db` 写进总线（会与 voice 公式双重衰减）。
5. **必须**在进入 `Paused` 的同一帧 `0 ms` 停掉全部池 voice 并清空 `pending_outcome`；**禁止**依赖 `process_mode` 或 `AudioServer` 自动静音。
6. **必须**让 `sfx.flow.pause` 走池外的 `UiTransitionPlayer`；**禁止**把任何池外 voice 计入 `MAX_VOICES` 或 `active_voice_count()`。
7. **必须**把抢占限制在「严格更低优先级」或「Critical 抢 Critical」，且 `age ≥ 50 ms`；**禁止**同级互抢（Critical 除外）、**禁止**抢 `age < 50 ms` 的 voice。
8. **必须**在抢占后保证 `voices_used(p) < cap[p]`；**禁止**让任何优先级超过 GDD 的 1/3/3/3。
9. **必须**在 `TRANSITION` cue 超过 250 ms 时 `stop()`；**禁止**循环播放任何 cue（MVP 的 `loop` 只能是 `false`）。
10. **必须**用 40 ms 手动分贝斜坡做拆除，并在 `VoicePoolHost._process` 里推进；**禁止**用 `Tween` 承载拆除正确性（暂停语义未验证）。
11. **必须**让音量公式在每个 voice 上只算一次并钳制到 `[-80, 0]`；**禁止**在 `Master` 总线上再叠一次 master gain。
12. **必须**把 cue 目录建成 `res://data/audio/cue_catalog.tres` 并作为 `DataConfig` 第六个清单校验；**禁止**运行时扫 `res://data/audio/`、**禁止**在 GDScript 里硬编码 cue 的 `base_db` / 冷却 / 变体表。
13. **必须**在 `bind_catalog()` 做 `CueMap` 双向校验；**禁止**允许孤儿 cue 或悬空映射进入 `Ready`。
14. **必须**让 `AudioPolicy` 的变体轮转只用 `play_index`；**禁止** `randf()` / `randi()` / `randi_range()` / `Array.shuffle()` / 新建 `RandomNumberGenerator` / 消耗 `SessionRng`。
15. **必须**把运行期播放失败降级为日志并继续；**禁止**因缺设备、缺流或解码失败改变 `SceneFlow` 状态或抛异常到主循环。
16. **必须**在 `shutdown()` 后拒绝一切请求并记 `not_ready`；**禁止**在 `Quit` 路径上 `await` 音频淡出。

## Alternatives Considered

### Alternative 1: 音频做成 autoload 单例

Godot 项目最常见的做法，暂停时行为最省事（`process_mode` 在 autoload 上设一次即可）。但 ADR-0004 D1 已经把 autoload 数钉为 0，且单例会让「谁在什么时候播放」无法在测试里隔离。**否决**：与组合根契约直接冲突。

### Alternative 2: 只用一个 `AudioStreamPlayer` + 起播时 `new()`，播完 `queue_free()`

实现最短，内存占用随并发浮动。但每次起播都要分配节点、走 `add_child`、跨帧释放，`MAX_VOICES = 10` 的上限反而变成「节点数不受控」；暂停时也无法遍历固定的 voice 集合做同帧 `stop()`。**否决**：可控性差，且无头断言需要额外簿记。

### Alternative 3: 用 `AudioStreamPolyphonic` + `AudioStreamPlaybackPolyphonic`

Godot 4 提供的多声部播放器，理论上一个节点播全部。但它的 voice 上限、抢占与「按 id 停止单个声部」的 API 在 4.7 的语义没有把握，且 `play_stream` 返回的 playback id 与 GDD 的 `voice_id` 日志字段对不上。**否决**：把正确性押在未验证的 API 上，与 ADR-0004 的风险策略不符。

### Alternative 4: 用 `AudioServer` 总线音量做音量映射（master / sfx / UI 各写一次总线）

这是最「混音台」的做法，设置变化会自然回溯到正在播放的声音。但 GDD 的公式已经把 `bus_offset_db + master_gain_db + sfx_gain_db` 写进 `cue_volume_db`，再写总线就是双重衰减；而且无头测试必须读 `AudioServer` 才能验证公式。**否决**：单一来源原则；总线只留 `Music` 给闪避。

### Alternative 5: cue 目录放 `design/audio/cues.json`，运行时解析

内容同学编辑最方便，也不需要新 Resource 类型。但 ADR-0001 要求类型化 Resource + 显式清单 + 可校验；JSON 会在导出后与 `.tres` 形成两套真相，且 `AudioStream` 引用无法被资源依赖图预载。**否决**：违反 ADR-0001 的数据格式契约。

### Alternative 6: 暂停音也进池，进入 `Paused` 时只停「除 `sfx.flow.pause` 之外」的 voice

改动最小，日志也统一。但这会让 `active_voice_count() == 0` 在进入暂停的同一帧不成立，直接违反 ADR-0004 D10 的验收项；而让 `active_voice_count()` 排除某个池 voice 又会让 GDD 的 `MAX_VOICES` 计数出现特例。**否决**：池外单播放器把特例收敛到一个明确的对象。

## Consequences

### Positive

- **暂停行为是一张封闭表**：`ALWAYS` 的四个节点、`WHEN_PAUSED` 为零、暂停音只有一个例外，逐条可断言。
- **音量公式只有一个来源**：总线恒为 0 dB，`cue_volume_db` 在 voice 上算一次；无头测试不需要音频设备就能验证 GDD 的每一条音量规则。
- **池的大小与 GDD 常量同源**：`MAX_VOICES`、`1/3/3/3`、`50 ms` 都是 policy 的 `const`，测试可以按值穷举，不依赖目录内容。
- **cue 目录进入既有的数据门**：复用 ADR-0001 的状态机、`Diagnostic` 与「全有或全无」，音频不需要第二套加载器，也不会出现「一半 cue 可用」。
- **测试与生产共用同一条决策路径**：`RecordingSink` 只替换输出端，不替换判定逻辑，CI 能在无音频驱动的机器上跑完整矩阵。
- **确定性与 ADR-0006 兼容**：MVP 完全不抽随机数；未来抖动有唯一指定写法，且明确禁止消耗 `SessionRng`。
- **未来音乐只差音源**：`Music` 总线与闪避包络已经就位，加音乐不需要改池、暂停或公式。

### Negative

- **多一个 `UiTransitionPlayer` 与一条池外规则**：读代码的人必须知道 `sfx.flow.*` 不走池。缓解：目录字段 `voice_class` 显式标注，校验器强制，日志用 `voice_id = 10` 区分。
- **`AudioPolicy` 与 `AudioDirector` 的分离多一层间接**：回调要穿过 policy 才能到 sink，调试时多一个栈帧。缓解：policy 的 `request()` 返回完整 `CueDecision`，单步即可看到原因。
- **音量不回溯**：改设置后正在响的声音保持旧音量，直到下一次起播。GDD 未要求实时回溯；若产品要求，需要在设置变化时遍历活跃 voice 重算（小改动）。
- **拆除用 40 ms 手动斜坡而不是 `Tween`**：多一个每帧遍历 10 个 voice 的循环。量级是 10 次比较，可接受。
- **`sfx.flow.quit` 会被 `shutdown()` 截断**：退出时最后一声可能不完整。这是「退出必须立即」与「声音必须完整」的取舍，选前者。
- **类别字段让 cue 目录更宽**：`voice_class`、`pause_policy`、`state_bound` 等是为行为服务的，不是音色；内容同学填错会被校验器挡住，但目录的认知负担上升。

### Neutral

- **`Music` 总线在 MVP 为空**：建它是为了闪避公式有对象、未来加音乐不改拓扑；成本是 1 条空总线。
- **`Ambience` 不存在**：与 GDD 的「环境声不做」一致，但与 GDD 闪避公式的文字不一致（见下）。
- **`priority` 原因码在 v1 不发出**：保留在封闭词表里，删除也要改 GDD。
- **`pitch` 字段对所有 cue 都有值**：MVP 恒为 `1.0`，与 GDD「仅数字 tick 有值」的文字不同（见下）；这样做是为了让未来抖动不需要改日志结构。
- **`AudioDirector` 不释放 `AudioStreamPlayer`**：随场景树销毁；`shutdown()` 只停声，不拆节点。

### GDD 矛盾与待修订项（本 ADR 不修改任何 GDD，仅登记）

| # | GDD 位置 | GDD 写法 | 本 ADR 的裁定 | 建议 |
|---|---|---|---|---|
| 1 | `design/gdd/audio-feedback.md:89` | `Uninitialized` 只建 `Master / SFX / UI` 三条总线 | 建 `Master / Music / SFX / UI` 四条 | GDD 状态表补 `Music`（空总线，MVP 无播放器） |
| 2 | `audio-feedback.md:44`（规则 18） | 「只在 Critical cue 播放时对 Music 闪避」 | 按 `:275-278` 的公式，由 `settlement_started`（Lifecycle）+ `broke_even` + `loss` 触发 | 规则 18 改为「结算三 cue 活跃时闪避」，与公式一致 |
| 3 | `audio-feedback.md:284`、`:286`、`:414` | 闪避作用于「Music 与 Ambience」；响度表列「环境声 / 音乐」 | 只有 `Music`，没有 `Ambience` | 删去 `Ambience` 提法，或把环境声并入 `SFX` 并注明未来修订 |
| 4 | `audio-feedback.md:66` vs `:40`、`:183` | cue 表要求 `sfx.flow.pause` 在进入 `Paused` 同帧起播；规则 14 与准入表第 3 步把 `Frozen` 期间一切请求丢弃 | `sfx.flow.pause` 走池外 `UiTransitionPlayer`，是唯一 `PLAY_THROUGH_FREEZE` 的 cue | 准入表加一句「池外转场 cue 不受第 3 步约束」，或明确 pause 音在冻结前起播 |
| 5 | `audio-feedback.md:117`、`:380` | 「数据配置无直接调用；cue→流映射不要求进入 Data Config schema」 | cue 目录作为 `DataConfig` 第六个清单校验 | 更新依赖表：音频经 `AudioCatalogView` 只读消费 `DataConfig` |
| 6 | `audio-feedback.md:21`、`:43` | 「音频出问题游戏照常运行」 | 运行期失败不阻断；**内容缺陷**（目录/映射非法）在启动期 `Boot → Error` | 区分「内容缺陷」与「播放失败」；前者属于数据门，后者才是静默降级 |
| 7 | `audio-feedback.md:142` | `pitch` 日志字段「仅数字 tick 有值」 | 所有 cue 都有 `pitch`，MVP 恒为 `1.0` | 改为「所有 cue 的播放音高；MVP 除数字 tick 外恒为 1.0」 |
| 8 | `audio-feedback.md:523`（AC 22） | GIVEN Music 在基准音量，WHEN 结果 cue 起播，THEN 50 ms 内降 12 dB | MVP 无音乐，AC 22 **无法听感验收**；只能断言 `Music` 总线的写入序列 | 把 AC 22 标为 `Deferred`（加入音乐后启用），或改为「总线写入」级验收 |
| 9 | `audio-feedback.md:150`（`cap[p]` 为固定不变量） | 准入表第 8/9 步未要求抢占后仍满足 `cap[p]` | 增加「抢占后 `voices_used(p) < cap[p]`」守卫 | 准入表第 9 步补该条件，使「上限是硬顶」自洽 |

## Risks

- **`AudioStreamPlayer` 在暂停中是否继续推进**（Verification ①）：若「不推进」，`sfx.flow.pause` 会在冻结帧停在半途。→ 缓解：它只有 250 ms 上限，实测不成立时退化为「暂停音被冻结吞掉」并记日志，不影响池内语义。
- **headless 下 `playing` 的返回值**（Verification ②）：若 Dummy 驱动让 `playing` 恒为 `false`，voice 会在下一帧被误回收。→ 缓解：`GodotAudioSink.active()` 在 `device_available == false` 时回退到「按 `max_duration_ms` 计时的逻辑 voice」；`RecordingSink` 不受影响，CI 仍可跑。
- **设备热插拔只能轮询**（Verification ③）：若 4.7 没有可靠的设备信号，拔耳机后最多延迟一个探测周期才静音。→ 缓解：探测周期 ≤ 1 s，且 `no_device` 是 drop 不是错误。
- **`AudioStream.get_length()` 在无设备时不可用**（Verification ④）：`zero_duration` 校验会误报。→ 缓解：该规则只在 `expected_length_ms > 0` 且 `get_length() > 0` 时比较；`get_length() == 0` 在 headless 下降级为 WARN，不阻断发布。
- **`CueMap` 与 GDD cue 表漂移**：cue 表有 20 个 id，代码里手写映射容易漏。→ 缓解：`bind_catalog()` 双向校验（孤儿 + 悬空），夹具逐条对照 cue 表；任何新增 cue 必须同时改表、目录、映射三处。
- **池外转场被滥用**：有人把新 cue 声明成 `TRANSITION` 来绕过 10 voice 上限。→ 缓解：校验器只允许 `sfx.flow.pause/resume/menu/quit` 四个 id 使用 `TRANSITION`，其余一律 `AUDIO_CUE_BAD_RANGE`。
- **`Music` 总线的空写入**：未来加音乐时若忘记把 `MusicPlayer` 挂到 `ALWAYS`，暂停会带着音乐。→ 缓解：D3 的表已经预留该行；加音乐时必须改本 ADR 的 D2/D3 并补验收。
- **日志 1 万条常驻内存**：约 1.25 MB（每条 ~128 B），在低端机上不可忽略。→ 缓解：这是 GDD 的固定要求；只在 debug 浮层额外渲染，release 不渲染。

## GDD References

- `design/gdd/audio-feedback.md` — 规则 1–21（尤其 5、8、9、10、11、12、14、18、21）；cue 表 `:53-75`；状态表 `:89`；音量公式 `:154-176`；准入表 `:180-200`；冷却 `:226`；`roll_ticks` `:238-262`；`number_tick_pitch` `:264-269`；`duck_db` `:271-287`；cue 日志 `:127-146`；Open Questions 2 / 6 / 7 / 8。
- `design/gdd/ui-flow.md` — 规则 24（音量/静音设置的所有权）、规则 25（`ui_button_activated` / `number_roll_*` / `receipt_closed` 语义事件）；PauseOverlay 不卸载 HUD。
- `design/gdd/input.md` — `ui_pause` 由 SceneFlow 独占；输入不直接产生音频事件。
- `design/gdd/scene-flow.md` — 六个顶层状态、`state_changed(from, to)`、暂停冻结、无焦点自动暂停、过渡不承载状态。
- `design/gdd/run-flow.md` — `run_started(run_id, station_id, seed)`、`action_committed`、`run_ended` / `run_faulted` / `run_discarded`；`run_settled` 不映射 cue；`run_seed` 来源。
- `design/gdd/time-and-turns.md`、`design/gdd/menu-waves.md` — 无独立音频信号；音频只经上述事件派生。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|---|---|---|
| `audio-feedback.md` | TR-audio-feedback-001：事件→cue 映射；固定 cue 目录；同帧派发 | D1（`AppRoot` 扇出第 4 步直接调用）、D7（`CueMap` 静态映射 + 目录）、D8（目录同帧可用）、D12（每请求一条日志） |
| `audio-feedback.md` | TR-audio-feedback-002：10 voice；1/3/3/3；抢占；50 ms 守卫 | D5（九步准入、硬顶守卫）、D6（抢占即时停止）、D13（`RecordingSink` 可穷举） |
| `audio-feedback.md` | TR-audio-feedback-003：确定性轮转；冷却；每 `run_id` 去重 | D11（`play_index`）、D5 第 5/6 步（`dedupe` / `cooldown`）、D7 字段 |
| `audio-feedback.md` | TR-audio-feedback-004：音量公式与钳制 | D9（voice 上一次计算；总线 0 dB）、D2（无双重衰减）、D12（日志 `volume_db`） |
| `audio-feedback.md` | TR-audio-feedback-005：暂停静默；清 `pending_outcome`；mute/无设备 drop | D3（同帧 0 ms 停池）、D4（唯一转场例外）、D9（mute 立即静音）、D5 第 2/3/4 步 |
| `audio-feedback.md` | TR-audio-feedback-006：1 万条日志；封闭词表 | D12（字段、容量、枚举、额外只读视图） |
| `audio-feedback.md` | TR-audio-feedback-007：内容缺陷启动期阻断；运行期播放失败静默降级 | D7（目录与 `CueMap` 校验）、D8（`DATA_FAILED` 启动门）、D14（运行期降级） |
| `ui-flow.md` | TR-ui-flow-007：音频语义事件 | D7 `CueMap` 的事件侧；`ui_button_activated` / `number_roll_*` / `receipt_closed` 由 UI 发出，音频只消费 |

## TR Coverage

| TR | 覆盖方式 |
|---|---|
| `TR-audio-feedback-001`（事件→cue；固定目录；同帧派发） | D1（`AppRoot` 扇出第 4 步直接调用）、D7（`CueMap` 静态映射 + 目录）、D8（目录同帧可用）、D12（每请求一条日志） |
| `TR-audio-feedback-002`（10 voice；1/3/3/3；抢占；50 ms） | D5（九步准入、硬顶守卫、`max_voices` 只收紧）、D6（抢占即时停止）、D13（`RecordingSink` 可穷举） |
| `TR-audio-feedback-003`（确定性轮转；冷却；每 `run_id` 去重） | D11（`play_index`）、D5 第 5/6 步（`dedupe` / `cooldown`）、D7 字段 |
| `TR-audio-feedback-004`（音量公式与钳制） | D9（voice 上一次计算；总线 0 dB）、D2（无双重衰减）、D12（日志 `volume_db`） |
| `TR-audio-feedback-005`（暂停静默；清 `pending_outcome`；mute/无设备 drop） | D3（同帧 0 ms 停池）、D4（唯一转场例外）、D9（mute 立即静音）、D5 第 2/3/4 步 |
| `TR-audio-feedback-006`（1 万条日志；封闭词表） | D12（字段、容量、枚举、额外只读视图） |
| `TR-audio-feedback-007`（内容缺陷启动阻断；运行期播放失败静默降级） | D7（目录与 `CueMap` 校验）、D8（`DATA_FAILED` 启动门）、D14（运行期降级） |
| `TR-ui-flow-007`（音频语义事件） | D7 `CueMap` 的事件侧；`ui_button_activated` / `number_roll_*` / `receipt_closed` 由 UI 发出，音频只消费 |

**建议的 TR 文本修订**（`tr-registry.yaml` 由编排者单写，本 ADR 不改）：

- `TR-audio-feedback-001`：追加「目录是 `res://data/audio/cue_catalog.tres` 的类型化 Resource，由 `DataConfig` 校验」。
- `TR-audio-feedback-002`：追加「`cap[p]` 是硬顶，抢占后仍需满足」。
- `TR-audio-feedback-003`：追加「MVP 禁用 pitch 抖动；启用时只能由 ADR-0006 的 `SeedHash` 派生」。
- `TR-audio-feedback-005`：追加「`sfx.flow.*` 走池外转场播放器；池内 voice 同帧归零」。
- `TR-audio-feedback-006`：追加「`priority` 原因码在 v1 保留但不发出」。
- 新增 `TR-audio-feedback-007`（建议）：「内容缺陷在启动期阻断（`Boot → Error`），播放失败在运行期静默降级」。

## Performance Implications

- **CPU**：一次 `request()` 是 9 条整数比较 + 一次字典查表，p95 ≤ 0.5 ms，最坏 ≤ 2 ms（含一次 `stop()` + `play()`）；每帧 `tick()` 遍历 11 个 voice 的 `playing` 与 1 个闪避包络，≤ 0.05 ms。稳态每帧音频逻辑成本约 0.01 ms。
- **Memory**：10 + 1 个常驻 `AudioStreamPlayer`（节点本身，KB 级）；`AudioStream` 随 `.tres` 依赖图预载，大小由资产决定，MVP 的 cue 表约 20 条、每条 ≤ 4 变体、单条 ≤ 0.5 s，量级 < 2 MB（资产交付时复核）。cue 日志环形缓冲 **10,000 × ~128 B ≈ 1.25 MB**，常驻。
- **Load Time**：`ensure_buses()` + 11 个节点构造 < 1 ms；cue 目录随 `DataConfig` 一次加载，计入 ADR-0001 的数据预算（p95 ≤ 250 ms），不另设预算。
- **Frame cost**：暂停与恢复各一次同步扇出调用；无 `Tween`、无每帧分配（日志条目复用槽位）。
- **Network**：无。

## Migration Plan

绿地实现，当前无代码。落地顺序（每步可独立验收）：

1. `AudioPolicy` + 端口 + `RecordingSink` / `FixedClock` / `StubCatalog`：先让九步准入、抢占、冷却、去重、音量公式在纯 GDScript 测试里全绿（不需要引擎音频）。
2. `AudioCueCatalogResource` / `AudioCueData` + `DataConfig` 第六清单 + 八条校验；用坏夹具验证「任一 ERROR 不发布」。
3. `AudioDirector.ensure_buses()` + `VoicePoolHost`（10 + 1 个播放器）+ `GodotAudioSink`；先只验证起播/停止/`active_voice_count`。
4. `CueMap` + `bind_catalog()` 双向校验；用缺项夹具验证 `Boot → Error`。
5. 暂停：`on_flow_state` 的同帧停池、`UiTransitionPlayer` 的 pause/resume、`pending_outcome` 清空；逐条对 ADR-0004 D10 与 GDD 规则 14 验收。
6. 结果 cue 状态机（`settlement_started` → `broke_even` / `loss`，`loss` 优先，`duration_ms + 100 ms` 强制清空）。
7. `Music` 总线闪避包络（无播放器，只断言总线写入序列）。
8. `CueLogEntry` 环形缓冲 + `cue_log()`；对照 GDD 的 14 个字段与封闭词表。

**Rollback plan**：本 ADR 不涉及持久化或存档格式。回滚方式是改文档 + 改一处装配：若池外转场被证伪（Verification ① 不成立），把 `sfx.flow.*` 退回池内并把暂停音标为 `dropped:paused`；若 `AudioPolicy` 的端口抽象被证明过重，可把 `AudioDirector` 直接内联，但 `RecordingSink` 必须保留（无头验收是硬需求）。两种回滚都不触碰 `DataConfig` 的五个既有清单。

## Validation Criteria

- **总线图**：启动后 `AudioServer` 的 bus 名集合恰好为 `{Master, Music, SFX, UI}`；`Master/SFX/UI` 的 `volume_db` 全程为 0；`Music` 只在闪避时有非 0 写入。
- **暂停静默**：`Run → Paused` 后同一帧 `active_voice_count() == 0`、`pending_outcome() == ""`；暂停期间 100 次 `request()` 全部 `dropped:paused`；`Paused → Run` 后无补播，且 `sfx.flow.resume` 恰好一条。
- **暂停音**：`sfx.flow.pause` 的日志 `voice_id == 10`、`decision == admitted`；`transition_voice_active()` 在 250 ms 内为 `true`、之后为 `false`。
- **准入矩阵**：用 `StubCatalog` 构造 10 个 voice 全满、各优先级上限、`age` 49/50/51 ms 的边界，断言九步结果与 `CueDecision.reason` 逐条等于 GDD 表；`cap[p]` 在任何抢占后都不被突破。
- **抢占**：Critical 抢 Critical 选最旧；Gameplay 抢 UI 不抢 Gameplay；`age < 50 ms` 的 voice 永不被抢。
- **音量**：`base_db=-8, bus=SFX, master=0, sfx=-6` → `-14.0`；`base_db=-18, bus=UI` → `-27.0`；`master=-80` → `dropped:muted` 且 `active_count` 不变。
- **去重与冷却**：同一 `run_id` 的 `run_ended` 第二次 → `dropped:dedupe`；`ui.click` 在 30 ms 内 → `dropped:cooldown`，被丢弃的请求不更新 `last_play`。
- **目录校验**：对八条规则各一个坏夹具，`DataConfig` 停在 `Failed`，`get_cue()` 不可用，且没有任何 cue 被发布。
- **映射**：`CueMap` 缺目标 → `AUDIO_CUE_UNMAPPED` + `Boot → Error`；目录多一条无事件引用的 `POOL` cue → `AUDIO_CUE_ORPHAN`。
- **确定性**：同一事件序列跑两遍，`cue_log()` 的 `cue_id` / `variant` / `decision` 序列逐条相等；源码扫描 `randf(` / `randi(` / `RandomNumberGenerator` / `SessionRng` 在 `src/audio/` 命中 0 次。
- **设备降级**：`RecordingSink.device_available = false` 下全部 `dropped:no_device`，`state` 仍为 `Ready`，`SceneFlow` 不变。
- **关闭**：`shutdown()` 后再 `request()` → `dropped:not_ready`；`shutdown()` 调两次不报错。
- **无头**：`--headless` 下完整启动到 `Menu`，`cue_log()` 可读，`active_voice_count() == 0`，无 WARN 之外的引擎错误。

## Related

- `docs/architecture/architecture.md` — 四层、所有权表（Audio Feedback 行）、依赖方向 `Audio ← UI Flow + Run Flow signals`、Required ADR #9、`QQ-09`、附录 A 的 `TR-audio-feedback-001..006` 与 `TR-ui-flow-007`
- `docs/architecture/adr-0001-content-data-format.md` — `DataConfig` 状态机、六阶段校验、`Diagnostic`、全有或全无发布、`MilestoneCatalog` 先例
- `docs/architecture/adr-0003-scene-screen-architecture.md` — `UIRoot` 不引用 HUD/音频、`ui_button_activated` 唯一入口、整条 UI 路径 `ALWAYS`
- `docs/architecture/adr-0004-service-composition-signals.md` — 零 autoload、`AppRoot` 组合根、六步扇出第 4 步、D10 暂停白名单、D11 `Quit` 序列
- `docs/architecture/adr-0006-deterministic-rng.md` — `SeedHash` / `SeededStream`、禁止全局 RNG、`SessionRng` 的唯一性
- `docs/architecture/adr-0008-settings-localization.md` — `SettingsModel.audio()`、音量/静音范围、设置不进存档
- `design/gdd/audio-feedback.md`、`ui-flow.md`、`input.md`、`scene-flow.md`、`run-flow.md`、`time-and-turns.md`、`menu-waves.md`
- `docs/engine-reference/godot/VERSION.md`、`modules/audio.md`、`current-best-practices.md`、`breaking-changes.md`、`deprecated-apis.md`

## Open Questions

1. **`sfx.flow.pause` 在冻结帧是否真的能响完**：取决于 Verification ①。若不能，应改为「在 `SceneTree.paused = true` 之前起播」——那需要 `SceneFlow` 在写 `paused` 前给音频一个 hook，属于 ADR-0004 扇出顺序的修订，不是本 ADR 能单方面决定的。
2. **`Music` 总线空转的验收价值**：若团队认为「MVP 没有音乐就不该有 `Music` 总线」，可以删掉它并把 AC 22 标为 `Deferred`；本 ADR 选择保留，是为了让闪避公式在加音乐时不改拓扑。这需要 `audio-director` 确认。
3. **`sfx.flow.quit` 是否值得存在**：`shutdown()` 会截断它，玩家大概率听不到。若确认听不到，应从 cue 表删除，减少一条不可能被验收的 cue。
4. **`reduce_repetitive_cues` 的确切语义**：GDD 把它归给 HUD 的滚动时间表，但名字暗示音频侧也应降密度。本 ADR 选择「音频只消费 HUD 已经降过的 tick」，若产品要求音频自己降，需要新增一条准入规则。
5. **多语言/多平台的音频资产命名**：`res://assets/audio/` 的组织方式（按 cue id 还是按场景）不在本 ADR 范围，但目录里的 `streams` 引用会锁定它；建议由 `audio-director` 在声音规范里先定。
6. **`AudioStream.get_length()` 的容差**：本 ADR 取 ±50 ms，这是结构性常量而非 GDD 数字；若实测 OGG 解码长度偏差更大，需要放宽并记录理由。
