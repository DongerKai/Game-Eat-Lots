# 音效反馈（Audio Feedback）

> **Status**: In Design
> **Author**: user + Claude（autonomous 模式；systems-designer、qa-lead、audio-director 按 Lean 模式复核 Formulas、Acceptance Criteria 与音频方向）
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 数字看得见地涨 · 没有白吃的菜

## Summary

音效反馈把《回本》里已经发生的事实变成声音：吃菜、数字上涨、收据打印、回本或亏损、放弃与故障。它订阅餐局流程、结算与场景流程广播的事件，用一套固定的优先级、并发上限、冷却与变体轮换决定“这一声放不放、放几次、放多响”，但不拥有音色、录音或任何游戏数值。它服务支柱 4「数字看得见地涨」与支柱 2「没有白吃的菜」：声音必须与数字同帧，且失败、放弃与亏损永远听起来平静，不像奖励。

> **Quick reference** — Layer: `Presentation` · Priority: `Vertical Slice` · Key deps: `餐局流程, 结算与入场费, 场景与流程管理`（下游契约：`局内 HUD, 界面流程`；内容权威：声音规范）

## Overview

音效反馈把已经发生的事实变成声音：玩家点下一道菜、数字上涨、一局结束、收据打印、回本或亏损。它订阅「餐局流程」「结算与入场费」「场景与流程管理」已经广播的事件，把每个事件映射成一个语义化的音效 cue，再按一套明确的播放策略决定这一声是否真的发声——优先级、并发上限、冷却与变体轮换。它不判断“该不该吃”“赚没赚”，也不决定音色好不好听；它只保证**事件发生的那一刻，玩家听到的东西是对的、不重叠、不延迟、不撒谎**。

它服务支柱 4「数字看得见地涨」：数字的变化必须同时被看见和听见，声音是读数的第二条通道，而不是延迟读数的过场。它也服务支柱 2「没有白吃的菜」：放弃一局、初始化失败与结算亏损都必须用平静、不庆祝的声音收尾，声音不能把一次失败包装成一次奖励。

它的边界是刻意收窄的。游戏状态、数值、收据内容分别属于上游系统；录音、音色、混音美学属于 audio-director 与 sound-designer 的声音规范。本系统拥有的是**订阅与播放策略**：哪些事件发声、发哪一个 cue、什么时候允许它抢占、什么时候必须把它丢掉。音频出问题时游戏照常运行——最坏的结果是安静，而不是卡住或崩溃。

## Detailed Design

### Core Rules

1. **职责边界**：本系统拥有“事件 → cue → 是否发声”的映射与播放策略；不拥有事件本身（餐局流程 / 结算 / 场景与流程管理）、游戏数值（任何上游系统）、音色与录音（audio-director / sound-designer）、音量设置的界面与持久化（界面流程 / 存档）。
2. **只订阅、不推导**：所有 cue 只能由已广播的事件触发；本系统不轮询状态、不根据画面或数值自行判断“现在该响了”。它只读取事件 payload 中选 cue 必需的字段（`action`、`end_reason`、`run_id`、数字滚动的 `from` / `to` / `duration_ms`），不重算任何游戏值。
3. **语义 cue，不选音色**：cue id 使用 `sfx.<域>.<事件>` 命名。cue → 音频流（一个或多个变体）的映射属于声音规范与数据；本 GDD 固定 cue 目录与语义，不固定录音。
4. **一次事件至多一次请求**：同一信号实例只产生一次 cue 请求；同一 `run_id` 的生命周期 cue 重复到达时忽略并记 WARN。重复事件是上游 bug，不能变成双声。
5. **同帧派发**：事件广播的同一帧内完成 cue 请求；不得 `await` 播放结束、不得延迟到下一帧、不得用定时器补播。唯一的例外是结算结果 cue：它必须等到收据数字滚动结束（或玩家跳过）才起播，以保证“声音与落定的数字同帧”而不是与被滚动的数字重叠。
6. **永不阻塞**：音频路径不得门控输入、画面或状态转移。设备忙、流缺失、总线不可用时的统一行为是**丢弃并记录**，不是等待、不是重试、不是降级到别的 cue。
7. **四级优先级**：`Critical(4) > Lifecycle(3) > Gameplay(2) > UI(1)`。Critical 只给结算的两个结果语义（`broke_even` / `loss`）；Lifecycle 给开局 / 结束 / 丢弃 / 故障 / 收据打印；Gameplay 给吃菜 / 跳波 / 拒绝 / 数字滚动；UI 给点击 / 暂停 / 恢复 / 菜单。
8. **并发上限与抢占**：全局最多 10 个 voice；每级上限 `Critical 1 / Lifecycle 2 / Gameplay 3 / UI 3`。无空位时，只有**严格更高**优先级可以抢占该级中**最旧**的 voice；同级或更低优先级的请求被丢弃。**Critical 只能被另一个 Critical 抢占**，不能被任何低优先级抢占；被替换的 Critical 立即停止并记为 `stolen`。一个 voice 起播后的前 50 ms 不可被抢占，避免截断造成的咔哒声。
9. **冷却**：每个 cue 有独立的最小重触发间隔（见 Formulas）；间隔内的请求记为 `dropped:cooldown`，且**不刷新**冷却起点。结果 cue 与“每局一次”的生命周期 cue 不受冷却约束，由“每 `run_id` 一次”保证不重复。吃菜提交不用冷却，改用“同一 `action_id` 每帧至多一次”的去重，避免把确认反馈丢掉。
10. **变体轮换是确定的**：同一 cue 有多个变体时按固定顺序轮换（round-robin），不使用随机数；同一事件序列必须产生同一 cue 序列，以便回放与自动化测试。轮换游标在会话内单调前进，不因暂停、静音或丢弃而重置。
11. **音量由四段相加**：`cue_volume_db = clamp(base_db + bus_offset_db + master_gain_db + sfx_gain_db, -80, 0)`。`base_db` 属于声音规范（数据驱动），`bus_offset_db` 属于混音路由（SFX / UI 两条总线），后两项来自玩家设置。任何 cue 不得以正增益播放。
12. **结算结果 cue 每局一次、绝不叠加**：`settlement_started`（Lifecycle）先起播；`broke_even` 或 `loss`（Critical）在数字滚动结束时起播。结果 cue 每 `run_id` 只允许一次。同一帧同时收到两个结果语义是上游 bug；**`loss` 优先**——宁可少一次庆祝，也绝不把亏损听成回本。后到者记为 `suppressed:duplicate_outcome`。回本 cue 时长上限 0.4 秒（结算 GDD），本系统不得延长。每张收据的多个 `number_roll_started(field, ...)` 是独立滚动；结果 cue 只在**最后一个** `number_roll_finished`（或玩家跳过全部）后起播一次。缺少配对的 `number_roll_finished` 时，等待 `duration_ms + 100 ms` 后强制清除 `pending_outcome` 并起播结果 cue。
13. **丢弃与故障必须平静**：`run_discarded` 与 `run_faulted` 使用中性、低存在感的 cue，不得使用上扬、收银机或庆祝音色。故障使结果失效：`run_faulted` 到达时取消待播或正在播放的结果 cue。`run_discarded` 与**待播**的结果 cue 同时成立时，丢弃优先，结果 cue 记为 `suppressed:preempted`——放弃不是结算，声音不能暗示玩家拿到了结果；若结果 cue 已经在播，则让它自然结束，丢弃 cue 自己按 Lifecycle 规则排队或被丢弃，不去截断一个已经成立的结果。
14. **暂停即静默**：进入顶层 `Paused` 时，1 帧内停止所有正在播放的 voice，并取消待播的结算结果 cue；暂停期间的所有请求记为 `dropped:paused`；恢复后**不补播**暂停期间错过的事件。恢复本身可以有一个 UI 级 cue。
15. **静音是丢弃，不是静默播放**：主音量静音或有效增益到达 `-80 dB` 时，请求记为 `dropped:muted`，不分配 voice、不推进冷却；解除静音后不补播，下一条新请求正常发声。这样“静音”有唯一可观测的行为，测试不必区分“在放但听不见”和“没放”。
16. **缺失资产是可见错误**：cue id 没有对应音频流时，不播放、不替代，按 cue id 每次会话记一条 WARN。静默失败必须留下痕迹，否则“少了一声”永远查不到原因。
17. **无设备也运行**：无音频设备（headless 测试、设备被拔出）时置 `device_available = false`：照常接收与记录请求，不播放、不报错、不阻塞。cue 请求日志是本系统的可观测接口，也是验收测试的主要证据。
18. **音乐不在本系统内**：MVP 与 Vertical Slice 不含背景音乐；若未来加入音乐，本系统只负责在 Critical cue 播放时对 Music 总线做闪避（ducking），音乐内容与混音仍归 audio-director。
19. **没有焦点音**：焦点移动不发声。它会奖励“快速扫视”，与支柱 3「时间在走，手不用快」相反，也是切换刷屏时最大的噪声源；输入 GDD 已要求可见的描边 / 高亮，声音不是焦点的唯一指示。
20. **无效操作无声**：点击空白、无焦点时按确认、对非交互元素的操作——没有 cue。错误反馈只给“玩家做了合法但失败的动作”（如菜品放不下），用低闷短音，不用警示蜂鸣；具体音色由声音规范决定。
21. **场景态 cue 随状态退出而取消**：由某个界面状态拥有的 cue（暂停 / 恢复 / 菜单 / 点击）在该状态退出时停止；这与“抢占”是两件事——抢占按优先级，取消按生命周期。结算结果 cue 不属于任何界面状态，只由 `run_id`、故障与丢弃约束。

### 音效事件目录

**本表是本系统的对外契约**：cue id、来源事件、优先级与每局上限在此固定；`base_db`、变体数量与录音由声音规范填。`run_settled` 不映射独立 cue——结算的可听结果只由 `settlement_started` / `broke_even` / `loss` 三个语义承载，避免同一时刻两层声音。

| cue id | 来源事件 | 起播时机 | 语义 | 优先级 | 冷却 | 每局上限 |
|---|---|---|---|---|---|---|
| `sfx.run.start` | `run_started` | 进入 `Active` 同帧 | 进店、开局 | Lifecycle | — | 1 |
| `sfx.eat.commit` | `action_committed(eat)` | 提交同帧 | 这一口落定 | Gameplay | —（同一 `action_id` 每帧一次） | 不限 |
| `sfx.wave.skip` | `action_committed(skip_wave)` | 提交同帧 | 翻到下一波 | Gameplay | 120 ms | 不限 |
| `sfx.meal.end` | `action_committed(end_pickup)` | 提交同帧 | 主动收手 | Gameplay | 120 ms | 1 |
| `sfx.eat.reject` | HUD 的拒绝事件（#15 契约） | 拒绝同帧 | 放不下，低闷短音 | Gameplay | 240 ms | 不限 |
| `sfx.run.end` | `run_ended` | 结束判定同帧 | 最后一响，中性 | Lifecycle | — | 1 |
| `sfx.run.discard` | `run_discarded` | 丢弃同帧 | 平静收尾 | Lifecycle | — | 1 |
| `sfx.run.fault` | `run_faulted` | 故障同帧 | 平静低音 | Lifecycle | — | 1 |
| `sfx.settle.print` | `settlement_started` | 收据打开同帧 | 纸张 / 打印 | Lifecycle | — | 1 |
| `sfx.settle.broke_even` | `broke_even` | 数字滚动结束（或跳过） | 回本，短促收银机（≤ 0.4 s） | Critical | — | 1 |
| `sfx.settle.loss` | `loss` | 数字滚动结束（或跳过） | 未回本，低闷（不庆祝） | Critical | — | 1 |
| `sfx.flow.pause` | 顶层进入 `Paused` | 状态生效同帧 | 冻结 | UI | 200 ms | 不限 |
| `sfx.flow.resume` | 顶层离开 `Paused` | 状态生效同帧 | 恢复 | UI | 200 ms | 不限 |
| `sfx.flow.menu` | 顶层进入 `Menu`（非结算交接） | 状态生效同帧 | 回到菜单 | UI | 300 ms | 不限 |
| `sfx.flow.quit` | 顶层进入 `Quit` | 状态生效同帧 | 退出 | UI | — | 1 / 会话 |
| `sfx.ui.click` | 界面流程 `ui_button_activated`（#16 契约） | 激活同帧 | 控件按下 | UI | 30 ms | 不限 |
| `sfx.num.tick` | HUD / 收据的数字滚动（#15 / #16 契约） | 滚动时间表 | 数字上涨的颗粒 | Gameplay | —（由滚动时间表限速） | ≤ 10 / 次滚动（每次 `number_roll_started` 独立计） |
| `sfx.record.broken` | `record_broken`（#14） | 收据打开时 | 新纪录图章 | UI | — | 1 |
| `sfx.milestone` | `milestone_earned`（#14） | 达成同帧 | 翻页 / 勾选 | UI | 120 ms | 不限 |
| `sfx.career.complete` | `career_completed`（#14） | 完成同帧 | 一次性收尾 | Lifecycle | — | 1 / 会话 |
| `sfx.num.land` | 数字滚动结束 / 跳过 | 落定同帧 | 数字停住 | Gameplay | — | 1 / 次滚动 |

**同帧多事件的裁定顺序**（仅用于同一帧内多个 cue 竞争同一 voice 的情形）：`run_faulted` > 结算结果（`loss` 优先于 `broke_even`）> `settlement_started` > `run_ended` > `run_discarded` > `run_started`。低优先级 cue 不因裁定顺序而跳过冷却或去重检查。

**不存在的 cue（刻意不做）**：焦点移动 tick、开局倒计时、背景环境声（Vertical Slice 也不例外）、亏损警报音（`loss` 是低闷收尾，不是警报）、连续吃菜的连击音、时钟滴答、任何意义上的“你本可以”提示音。它们要么制造等待，要么奖励手快，要么把失败包装成奖励，违反支柱 2、3 与 4。

### States and Transitions

本系统只有四个状态；暂停、静音与设备可用性是**正交标志**，不是状态。理由与餐局流程相同：一个标志叠加在状态上，恢复时不必重建状态，也不会漏掉某条转移路径。

| 状态 | 进入条件 | 允许的行为 | 离开 |
|---|---|---|---|
| `Uninitialized` | 应用启动 | 建立 Master / SFX / UI 总线，预分配 voice 池，预载 cue 流；不播放 | 启动校验结束（`Boot → Menu` 或 `Boot → Error`）→ `Ready` |
| `Ready` | 启动校验结束 | 接收全部 cue 请求并按策略播放 | 顶层进入 `Paused` → `Frozen`；顶层进入 `Quit` → `Shutdown` |
| `Frozen` | 顶层进入 `Paused` | 停止所有 voice、取消待播结果 cue；所有请求记 `dropped:paused` | 顶层离开 `Paused` → `Ready` |
| `Shutdown` | 顶层进入 `Quit` | 同一帧只允许 `sfx.flow.quit` 一条；随后停止全部 voice、冲刷 cue 日志，不再接收请求 | 进程结束 |

| 标志 | 取值 | 影响 |
|---|---|---|
| `device_available` | `true` / `false` | `false` 时不播放、不报错，请求记 `dropped:no_device`，不推进冷却；状态仍为 `Ready`。设备恢复后只影响后续请求，不补播 |
| `muted` | `true` / `false` | `true` 时请求记 `dropped:muted`，不分配 voice、不推进冷却；解除后不补播 |
| `pending_outcome` | `run_id` / `null` | 持有已收到但尚未起播的结算结果 cue（等待数字滚动结束或跳过）。开局、暂停、故障、丢弃、收据关闭都清空它，且不补播 |

**转移规则**：

- `Frozen` 由场景与流程管理的顶层状态驱动，不由本系统自行判定；本系统的 audio 处理必须能在场景树暂停时继续接收暂停/恢复通知（`process_mode` 行为属于实现，见 Open Questions 的 ADR 候选）。
- 进入 `Frozen` 时，1 帧内停止所有 voice 并清空 `pending_outcome`；恢复后没有任何“迟到的声音”。
- `device_available` 与 `muted` 都是**丢弃**语义，不改变 `Ready` 状态，也不改变冷却起点。静音或拔设备不是错误，不回退到占位音。
- `Shutdown` 是终态；退出确认后不再播放新的 cue。

### Interactions with Other Systems

| 系统 | 方向 | 数据 / 接口 | 硬 / 软 |
|---|---|---|---|
| 餐局流程 | 上游事件源 | 订阅 `run_started`、`action_committed(action, payload)`、`run_ended`、`run_faulted`、`run_discarded`；`run_settled` 只用于确认交接完成，**不映射独立 cue**（结算声音由结算的三个事件承载） | 硬（信号契约）；产品意义上软——音频失败不阻塞一局 |
| 结算与入场费 | 上游事件源 | 订阅 `settlement_started`、`broke_even`、`loss`；只读事件语义，**不读 `RunResult`、不重算 `net_profit` 或 `broke_even`** | 硬（事件契约）；产品意义上软 |
| 场景与流程管理 | 上游状态源 | 顶层状态转移（`Boot / Error / Menu / Run / Paused / Quit`）与冻结/恢复通知；用于 `Ready` / `Frozen` / `Shutdown` 与 `flow.*` cue | 硬（状态契约）；产品意义上软 |
| 局内 HUD（#15，未设计） | 下游请求方 | 必须发出 `dish_action_rejected(reason)`、`number_roll_started(field, from, to, duration_ms)`、`number_roll_finished(field, skipped: bool)`；不得直接调用音频内部接口 | 软 |
| 界面流程（#16，未设计） | 下游请求方 + 设置所有者 | 必须发出 `ui_button_activated(control_id)`、收据的 `number_roll_started/finished`；拥有主音量、音效音量、静音与“减少重复音效”设置，并在收据关闭时通知本系统 | 软 |
| 声音规范（audio-director / sound-designer，非系统） | 上游资产权威 | cue → 音频流与变体、`base_db`、时长、录音素材；本系统只消费映射，不改写 | 硬（内容）；不属于代码依赖 |
| 数据配置 | 无直接调用 | cue → 流的映射在声音规范与音频数据中；MVP 不要求进入 `数据配置` 的菜品/档位 schema | 无 |
| 输入 | 无直接调用 | 本系统**不订阅**原始输入动作；只消费界面流程与 HUD 发出的语义事件，避免把焦点移动、长按与非提交操作误当点击 | 无 |
| 存档 | 无直接调用 | 音量与静音属于设置，不属于生涯存档（存档 GDD 明确设置文件独立） | 无 |

**接口所有权**：本系统拥有 cue id 目录、优先级、并发上限、冷却、抢占与闪避策略；上游拥有事件与语义；HUD / 界面流程拥有“数字何时滚动、滚动多久”；声音规范拥有“每个 cue 听起来是什么”。任何一方都不得替另一方做决定。

### 可观测接口：cue 请求日志

本系统在 headless、无音频设备与 release 构建下都记录 cue 请求日志；debug 构建额外提供屏幕浮层。日志是本系统的验收证据，`frame`、`time_ms` 与决策必须足以让 QA 不看代码就判断对错。

| 字段 | 类型 | 说明 |
|---|---|---|
| `frame` | int | 请求所在帧号 |
| `time_ms` | int | 注入时钟的毫秒时间；测试使用固定步长 |
| `cue_id` | StringName | 事件目录中的 id |
| `source_event` | StringName | 触发它的事件名 |
| `run_id` | int64 / null | 生命周期与结算 cue 带 run_id；UI cue 为空 |
| `priority` | int | 1 … 4 |
| `decision` | enum | `admitted` / `deferred` / `dropped` / `suppressed` |
| `reason` | enum / null | `admitted` 时为空；`deferred` 也带原因（目前只有 `pending_outcome`）；其余见下方封闭词表 |
| `stolen_voice_id` | int / null | `admitted` 且发生抢占时，被停止的 voice |
| `voice_id` | int / null | 本次分配的 voice |
| `active_count` | int | 决策后的活跃 voice 数（0 … 10） |
| `variant` | int | 使用的变体序号（从 0 起） |
| `volume_db` | float | 按 `cue_volume_db` 计算的结果 |
| `pitch` | float | 仅数字 tick 有值 |

**封闭原因词表**：`no_device`、`muted`、`paused`、`dedupe`、`cooldown`、`pool_full`、`no_stream`、`invalid`、`not_ready`、`priority`、`zero_duration`（`dropped`）；`duplicate_outcome`、`preempted`（`suppressed`）；`pending_outcome`（`deferred`）。任何新原因必须先改本表，不能临时加字符串——否则测试无法穷举。

日志使用固定容量环形缓冲（10,000 条），永不无限增长；溢出时覆盖最旧条目并记一次 WARN。

## Formulas

本系统的数值规则只有六条：音量映射、发声准入、重触发冷却、数字滚动的 tick 数、数字滚动的音高、音乐闪避。它们决定“这一声放不放、放多响、放几次、什么音高”，不决定音色。除 `MAX_VOICES`、优先级顺序、`[-80, 0]` 钳制与 Critical 规则是**固定不变量**外，其余数值都由声音规范或设置数据驱动。

### 音量映射 `cue_volume_db`

The `cue_volume_db` formula is defined as:

`cue_volume_db(c) = clamp(base_db[c] + bus_offset_db(bus(c)) + master_gain_db + sfx_gain_db, -80, 0)`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `base_db[c]` | float | −40 … 0 dB | data file（声音规范） | 该 cue 相对结算结果的基准响度；结算结果为 0 dB 参考 |
| `bus_offset_db` | float | −6 … 0 dB | data file（混音路由） | SFX 总线 0 dB、UI 总线 −3 dB |
| `master_gain_db` | float | −80 … 0 dB | settings（界面流程拥有） | 主音量；−80 表示静音 |
| `sfx_gain_db` | float | −40 … 0 dB | settings（界面流程拥有） | 音效音量，同时作用于 SFX 与 UI 总线 |

**Output Range:** −80 到 0 dB。结果等于 −80 时按 `muted` 丢弃，不分配 voice。系统永不输出正增益。
**Example:** 吃菜 cue `base_db = −8`，SFX 总线 0，主音量 0，音效 −6 → `−14 dB`。同一设置下 UI 点击 `base_db = −18`、UI 总线 −3 → `−27 dB`。

> 相对响度关系（进入 `base_db` 的默认值）：结算结果 0 dB 参考；吃菜 / 容量变化 −6 … −10 dB；数字滚动 −14 … −20 dB；错误 / 拒绝 −10 … −14 dB（不得响于吃菜）；UI 点击 −16 … −22 dB。绝对值与 LUFS 留给资产规范。

### 发声准入 `voice_admission`

The `voice_admission` formula is defined as:

`admit(r, t) = PLAY | PLAY_STEAL(v) | DROP(reason)`

判定按顺序执行，先命中者生效：

| 顺序 | 条件 | 结果 |
|---|---|---|
| 1 | `device_available = false` | `DROP(no_device)` |
| 2 | `muted` 或 `cue_volume_db(r) = −80` | `DROP(muted)` |
| 3 | 本系统状态为 `Frozen` | `DROP(paused)` |
| 4 | 该 cue 是每 `run_id` 一次的 cue 且该 `run_id` 已发过 | `DROP(dedupe)` |
| 5 | `retrigger_ok(r.cue, t) = false` | `DROP(cooldown)` |
| 6 | 全局空闲 voice > 0 且该优先级未达上限 | `PLAY` |
| 7 | `r.priority = Critical` 且存在已起播 ≥ 50 ms 的 Critical voice | `PLAY_STEAL(最旧者)` |
| 8 | 存在优先级严格更低、且已起播 ≥ 50 ms 的 voice | `PLAY_STEAL(最旧者)` |
| 9 | 其他 | `DROP(pool_full)` |

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `MAX_VOICES` | int | 10 | constant（固定） | 全局同时发声上限 |
| `cap[p]` | int | Critical 1 / Lifecycle 3 / Gameplay 3 / UI 3 | constant（固定） | 每级上限；合计 10，等于 `MAX_VOICES` |
| `MIN_STEAL_AGE_MS` | int | 50 ms | constant（固定） | 起播不足 50 ms 的 voice 不可被抢占，避免截断咔哒声 |
| `p(r)` | int | 1 … 4 | 事件目录 | 请求的优先级 |
| `voices_used(p)` | int | 0 … cap[p] | runtime | 该级正在发声的 voice 数 |

**Output Range:** `PLAY` 或 `PLAY_STEAL` 时活跃 voice 数 0 … 10；`DROP` 时不变。`run_faulted` 属于**取消**而非抢占：它取消待播或正在播放的结果 cue，不受 `MIN_STEAL_AGE_MS` 限制。
**Example:** 8 个 voice 正在播放（UI 3 + Gameplay 3 + Lifecycle 2），Critical 未使用、全局还剩 2 个空位 → 结算结果直接 `PLAY`，不抢占任何声音。若 10 个 voice 全满且最低级最旧 voice 已起播 120 ms → `PLAY_STEAL(该 voice)`。

### 重触发冷却 `retrigger_ok`

The `retrigger_ok` formula is defined as:

`retrigger_ok(c, t) = (t − last_play[c] ≥ cooldown_ms[c])`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `t` | int | 毫秒，单调递增 | runtime（注入时钟） | 当前时间；测试使用固定步长时钟 |
| `last_play[c]` | int | 毫秒 | runtime | 该 cue 上次**成功起播**的时间 |
| `cooldown_ms[c]` | int | 0 … 300 ms | data file（声音规范） | 该 cue 的最小重触发间隔 |

**Output Range:** `true` / `false`。被丢弃的请求**不更新** `last_play[c]`；静音、暂停、无设备同样不更新。
**Example:** `ui.click` 冷却 30 ms。t=100 ms 起播一次，t=120 ms 的请求 `20 < 30` → `false`；t=131 ms 的请求 → `true`。

默认冷却（全部数据驱动，可整体归零以做压力测试）：

| cue | `cooldown_ms` |
|---|---|
| `sfx.ui.click` | 30 |
| `sfx.wave.skip` / `sfx.meal.end` | 120 |
| `sfx.eat.reject` | 240（大于输入 0.12 s 的重复步进，避免边界抖动丢反馈） |
| `sfx.flow.pause` / `sfx.flow.resume` | 200 |
| `sfx.flow.menu` | 300 |
| `sfx.eat.commit` | 0（改用“同一 `action_id` 每帧一次”去重） |
| `sfx.num.tick` | 0（由滚动时间表自限速） |
| 其余（每局一次 / 结果 cue） | 0（由 `run_id` 去重） |

### 数字滚动 tick 数 `roll_ticks`

The `roll_ticks` formula is defined as:

`roll_ticks(d) = 0 if d < TICK_MS else min(floor(d / TICK_MS), MAX_TICKS)`

`tick_interval_ms(d) = d / roll_ticks(d)`（`roll_ticks(d) = 0` 时无定义，不产生 tick）

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `d` | int | 0 … 1500 ms | HUD / 界面流程事件 | 数字滚动时长；收据滚动 0.3–1.5 s（默认 0.8），局内即时变化不发滚动事件 |
| `TICK_MS` | int | 60 ms | data file | 一个 tick 的目标间隔 |
| `MAX_TICKS` | int | 10 | constant（固定） | 1500 / 150 = 10；上限保证一次滚动不刷屏 |

**Output Range:** tick 数 0 … 10；间隔 60 … 150 ms。`d = 0` 或 `end = start` 时**不产生任何 tick 与 `land` cue**——局内即时变化由吃菜 cue 承载，不叠加第二层声音。
**Example:** `d = 800` → 10 个 tick（封顶），每 80 ms 一个；`d = 1500` → 10 个 tick，每 150 ms 一个；`d = 250` → 4 个 tick；`d = 40` → 0 个 tick。

### 数字滚动音高 `number_tick_pitch`

The `number_tick_pitch` formula is defined as:

`r = (end − start) / max(|start|, |end|, 1)`
`semis = clamp(round(K · sign(r) · log2(1 + |r|)), −5, +7)`
`pitch_target = 2^(semis / 12)`
`pitch_i = 1 + (pitch_target − 1) · i / (ticks − 1)`，`i = 0 … ticks − 1`；`ticks = 1` 时 `pitch_1 = 1.0`

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `start` / `end` | int64 | ±9.99e5 | HUD / 界面流程事件 | 滚动起点与终点（如净利从 −85 到 19） |
| `r` | float | −2 … 2 | calculated | 归一化变化量；分母至少为 1，防止跨零与起点为 0 时除零 |
| `K` | float | 4 … 8 | data file | 每翻一倍变化的半音数；默认 6 |
| `semis` | int | −5 … +7 | calculated | 量化后的半音移调；`2^(−5/12)=0.749`、`2^(7/12)=1.498`，正好落在 `[0.75, 1.5]` |
| `ticks` | int | 0 … 10 | `roll_ticks` | 本次滚动的 tick 数 |

**Output Range:** `pitch_i` ∈ 0.749 … 1.498。正增量上行、负增量下行、零增量不变调；每次滚动重置，不跨局累积。
**Example:** 净利从 −85 滚到 +19，`r = 104 / max(85, 19, 1) = 1.224`，`semis = round(6 · 1 · log2(2.224)) = round(6.917) = 7`，`pitch_target = 2^(7/12) = 1.498`；10 个 tick 的音高从 1.000 线性升到 1.498。亏损从 0 滚到 −31，`r = −1`，`semis = −5`（钳制），音高降到 0.749——下行、不庆祝。

### 音乐闪避 `duck_db`

The `duck_db` formula is defined as:

`duck_db(t) = −DUCK_DB · env(t)`

`env(t)` 在 `settlement_started`、`broke_even`、`loss` 任一 cue 活跃时为 1，其余为 0；0 → 1 用 `ATTACK_MS` 线性上升，1 → 0 用 `RELEASE_MS` 线性下降。

**Variables:**
| Variable | Type | Range | Source | Description |
|----------|------|-------|--------|-------------|
| `DUCK_DB` | float | 6 … 15 dB | data file | 闪避深度；默认 12 dB，给收据阅读留出空间 |
| `ATTACK_MS` | int | 30 … 80 ms | data file | 默认 50 ms |
| `RELEASE_MS` | int | 80 … 150 ms | data file | 默认 120 ms；释放可以在 cue 结束后继续，但只作用于 Music / Ambience，不影响 SFX 与 UI |

**Output Range:** −15 … 0 dB，只作用于 Music 与 Ambience 总线；SFX 与 UI 总线永不闪避。
**Example:** 结算打印 cue 0.4 s：50 ms 内降到 −12 dB，保持，之后 120 ms 回到 0 dB，总计 520 ms。暂停或停止时立即以 50 ms 释放，避免下次恢复时音乐仍被压低。**Vertical Slice 无背景音乐与环境声，本公式处于休眠状态**，仅在加入音乐后生效。

## Edge Cases

**事件与去重**

- **If 同一 `run_id` 的 `run_started` 到达两次**：第二次记 `dropped:dedupe` 并记一条 WARN，不发声。重复信号是上游 bug，不能变成双声。
- **If `run_started` 与 `run_ended` 在同一帧到达**（零回合局，初始化后立即判定结束）：`run_ended` 生效，`run_started` 记 `suppressed:preempted`。一局短到没有内容时，不应先给“进店”再给“结束”两层声音。
- **If 同一 `run_id` 的 `run_ended` / `run_discarded` / `run_faulted` 到达两次**：第二次 `dropped:dedupe`，不发声。
- **If `run_settled` 到达**：不产生任何 cue，只用于确认交接完成；结算声音由 `settlement_started` / `broke_even` / `loss` 承载。重复 `run_settled` 同样无声。
- **If `action_committed` 的 `action` 不在 `{eat, skip_wave, end_meal}` 内**：记 `dropped:invalid`，一条 WARN，不猜测替代 cue。
- **If 同一帧收到两个吃菜提交**：两个 `sfx.eat.commit` 都准入（Gameplay 上限 3，足够）；第三个起按并发上限处理——同级不互相抢占，超出即丢弃。不得合并成一声，否则玩家无法确认第二口。
- **If 同一 `action_id` 在同一帧被重复广播**：只发一次 `sfx.eat.commit`，第二次 `dropped:dedupe`。若 payload 没有 `action_id`，按 `(action, dish_id, 帧号)` 去重；仍无法区分时以“每帧每个动作一次”为下限。
- **If 事件在 `Uninitialized` 阶段到达**（启动早期）：`dropped:not_ready`，不排队、不延后播放。音频不补偿自己没准备好的时间。

**结算**

- **If `settlement_started` 与结果事件在同一帧到达**：打印 cue 以 Lifecycle 起播；结果 cue 进入 `pending_outcome`，等数字滚动结束。
- **If 收据没有数字滚动**（HUD / 界面流程未发 `number_roll_started`）：在 `settlement_started` 所在帧结束时立即播放结果 cue，不无限等待。
- **If 玩家跳过数字滚动**：`number_roll_finished(skipped = true)` 立即起播结果 cue，停止尚未派发的 tick；已起的 `sfx.num.land` 不再重复。
- **If 玩家在结果 cue 起播前关闭收据 / 返回菜单 / 开始新一局**：取消 `pending_outcome`，不补播。结算结果只在收据还开着的时候有意义。
- **If `broke_even` 与 `loss` 在同一帧都到达**：`loss` 生效，`broke_even` 记 `suppressed:duplicate_outcome`。宁可少一次庆祝，也不把亏损听成回本。
- **If `run_faulted` 到达时结果 cue 待播或正在播放**：取消结果 cue（不走抢占，不受 50 ms 下限约束），播放 `sfx.run.fault`。故障使结果失效，声音必须先说“出错了”。
- **If `run_discarded` 到达时结果 cue 仍在待播**：取消结果 cue，记 `suppressed:preempted`，播放丢弃 cue。
- **If `run_discarded` 到达时结果 cue 已经在播**：让结果自然结束；丢弃 cue 记 `dropped:priority`，不截断一个已经成立的结果。
- **If 结果 cue 正在播放时有任何 Lifecycle cue 到达**（结束、丢弃、打印）：Lifecycle 不得抢占 Critical，一律记 `dropped:priority`，不叠加。故障是唯一例外（见上一条）。
- **If 音频流实际时长超过 cue 目录上限**（如回本 cue > 0.4 s）：debug 构建记 ERROR，release 继续播放，不截断。截断声音比超时更糟；资产规范必须修正素材。

**数字滚动**

- **If 滚动时长 `d = 0` 或起终点相同**：不产生 tick、不产生 `sfx.num.land`，只记一条 `dropped:zero_duration`（每次滚动）。局内即时变化由吃菜 cue 承载，不叠加第二层声音。
- **If `d < 60 ms`**：0 个 tick，只在结束时播放 `sfx.num.land`。
- **If 滚动跨过零点或起点为 0**：使用 `max(|start|, |end|, 1)` 作分母；音高公式不除零、不反向。
- **If 滚动进行中暂停**：停止全部 tick 与 land，`pending_outcome` 一并清空；恢复后不重放这次滚动的声音（数字本身由界面恢复，音频不补）。
- **If 滚动进行中静音**：立即停止正在播放的 cue，不再派发后续 tick；解除静音后不补播。

**优先级、并发与设备**

- **If 10 个 voice 全满且新请求与最旧 voice 同级**：`dropped:pool_full`；同级不互相抢占，避免快速输入下互相截断。
- **If 一个 voice 起播不足 50 ms**：不可被抢占；本次请求若无其他可抢占对象则 `dropped:pool_full`。
- **If 第二个 Critical 到达且第一个已起播 ≥ 50 ms**：新 Critical 替换旧 Critical，旧 voice 停止并在新记录中记 `stolen_voice_id`。结算结果每局一次，实际只会发生在“结果 cue 被更新结果替换”的 bug 场景。
- **If 设备在启动时不存在**（headless 测试、容器）：`device_available = false`，状态仍为 `Ready`；所有请求照常记录、不播放、不弹窗、不报 ERROR。
- **If 设备在运行中被拔出**：立即停止全部 voice，置 `device_available = false`；本次会话不重试、不补播；设备恢复后只影响后续请求。
- **If cue 没有对应音频流**：不播放、不替代；每个 cue id 每次会话记一条 WARN，后续请求记 `dropped:no_stream`。静默失败必须留下痕迹。
- **If 主音量被静音或有效增益为 −80 dB**：立即停止全部 voice；请求记 `dropped:muted`；解除静音后不补播。
- **If 窗口失去焦点**：不自动暂停，音频继续；操作系统的音频挂起不视为本系统的错误。

**状态与生命周期**

- **If 顶层进入 `Paused`**：1 帧内停止所有 voice、清空 `pending_outcome`；暂停期间的请求全部 `dropped:paused`。恢复后不补播。
- **If 顶层进入 `Quit`**：停止全部 voice，冲刷 cue 日志，不再接收请求。
- **If 场景态 cue 所属的状态退出**（点击声还在播时进入 `Run`，暂停声还在播时恢复）：停止该 cue；这是生命周期取消，不是抢占，不受优先级约束。
- **If 一局结束并立刻重开**：新 `run_id` 重置“每局一次”的去重；变体轮换游标**不重置**，避免每局第一声都相同。
- **If 连续两局使用相同 `run_id`**（上游 bug）：第二局的生命周期 cue 全部 `dropped:dedupe`；记一条 ERROR，不崩溃。

**输入与设置**

- **If 玩家点击空白处 / 无焦点按确认**：上游不发送 `ui_button_activated`，本系统无声；即使误发，`control_id` 为空或未知时记 `dropped:invalid`。
- **If 玩家快速连续点击同一按钮**：30 ms 内只响一次；被丢弃的请求不刷新冷却，因此持续按住不会无限延长静默。
- **If 玩家开启“减少重复音效”**：只影响重复型 cue 的冷却与变体策略（由设置读取），不改变优先级、并发上限、结算结果与吃菜确认音；该选项不得让结算变轻，也不得让确认反馈消失。
- **If 玩家把音效音量调到 0 后又调回**：调回后的下一条新请求正常发声；不补播调低期间的任何 cue。

## Dependencies

**上游依赖（本系统消费它们的事件、状态或内容）：**

| 系统 | 硬 / 软 | 接口 | 状态 |
|---|---|---|---|
| 餐局流程 | 硬（事件契约）；产品意义上软 | 订阅 `run_started`、`action_committed(action, payload)`、`run_ended`、`run_faulted`、`run_discarded`；`run_settled` 只用于确认交接，不映射 cue | GDD 已写（`design/gdd/run-flow.md`，Designed） |
| 结算与入场费 | 硬（事件契约）；产品意义上软 | 订阅 `settlement_started`、`broke_even`、`loss`；只读事件语义，不读 `RunResult`、不重算 `net_profit` / `broke_even` | GDD 已写（`design/gdd/settlement-and-entry-fee.md`，Designed） |
| 场景与流程管理 | 硬（状态契约）；产品意义上软 | 顶级状态转移（`Boot / Error / Menu / Run / Paused / Quit`）与 `state_changed(from, to)` 信号；驱动 `Ready` / `Frozen` / `Shutdown` 与 `flow.*` cue | GDD 已写（`design/gdd/scene-flow.md`，In Review） |
| 纪录与里程碑 | 硬（事件契约） | 订阅 `record_broken` / `milestone_earned` / `career_completed` | GDD 已写（`design/gdd/records-and-milestones.md`） |
| 声音规范（audio-director / sound-designer，非代码系统） | 硬（内容） | cue → 音频流与变体、`base_db`、时长、录音素材；本系统只消费映射 | **未建立**（尚无 `design/audio/` 或 sound bible；见 Open Question 2） |

**下游依赖方（它们把事件交给本系统，或消费本系统的行为）：**

| 系统 | 硬 / 软 | 接口 | 状态 |
|---|---|---|---|
| 局内 HUD（#15） | 软 | 必须发出 `dish_action_rejected(reason)`、`number_roll_started(field, from, to, duration_ms)`、`number_roll_finished(field, skipped: bool)`；不得直接调用音频内部接口 | GDD 已写（`design/gdd/in-game-hud.md`） |
| 界面流程（#16） | 软 | 必须发出 `ui_button_activated(control_id)`、收据的 `number_roll_started/finished`、收据关闭通知；拥有主音量 / 音效音量 / 静音 / “减少重复音效”设置 | GDD 已写（`design/gdd/ui-flow.md`） |

**对未设计系统的前置契约（provisional）：**

| 系统 | 本系统要求的接口 | 若最终实现不同 |
|---|---|---|
| 局内 HUD | 只在**合法但失败**的菜品操作上发 `dish_action_rejected`（如放不下）；点击空白、悬停、焦点移动不发。只有**动画滚动**的数字才发 `number_roll_*`；局内即时变化不发，避免与吃菜 cue 叠层 | 若 HUD 不发拒绝事件，拒绝反馈静音（`sfx.eat.reject` 成为死 cue）；若对即时变化也发滚动事件，同一动作会出现两层声音，必须由 HUD 关闭 |
| 界面流程 | `ui_button_activated` 只在**已提交的控件激活**上发（`ui_accept` 按下或鼠标点击有效控件），不在焦点移动、悬停、点击空白时发；收据跳过滚动时发 `number_roll_finished(skipped = true)`；收据关闭 / 返回菜单 / 开始新一局时通知本系统取消待播结果 | 若界面流程用原始 `ui_accept` 代替语义事件，焦点移动和长按会被误当点击；本 GDD 明确不订阅原始输入 |
| 场景与流程管理 | 顶层状态变化可被订阅或读取，且音频处理在场景树暂停时仍能收到进入 / 离开 `Paused` 的通知 | 若暂停时音频处理也被冻结，暂停音与恢复音会迟到；实现上需要 `process_mode` 常驻（见 Open Question 6） |
| 声音规范 | 每个 cue id 至少一个音频流与变体；`base_db` 落在音量映射的范围内；结算结果 cue ≤ 0.4 s | 缺流时静音 + WARN，不替代；超时只在 debug 报错，不截断 |

**不属于依赖的关系：**

- **输入**：本系统不订阅任何原始输入动作。它消费的是界面流程与 HUD 发出的语义事件；把输入列为依赖会把焦点移动、悬停与“按下但未提交”混进 cue 目录。
- **数据配置**：cue → 流的映射属于声音规范与音频数据，不进入 MVP 的菜品 / 档位 schema。若未来把 cue 表并入 `数据配置`，那是一次 schema 扩展，不是本 GDD 的既有依赖。
- **存档**：音量、静音与“减少重复音效”属于设置，存档 GDD 已明确设置文件独立于生涯档；本系统不读写存档。
- **局外升级 / 解锁与档口**：无直接调用。局外升级虽发出 `purchase_succeeded` / `purchase_failed` / `run_result_applied`，MVP 不把它们映射为 cue；结算声音只由结算的三个语义承载。纪录与里程碑的 `record_broken` / `milestone_earned` / `career_completed` 已在上表映射为 `sfx.record.broken` / `sfx.milestone` / `sfx.career.complete`。

**双向一致核对（不在本文件修改）：**

- `run-flow.md` 的下游表把「音效反馈」记为 `GDD 未写`，应改为 `GDD 已写（design/gdd/audio-feedback.md）`。其接口行“订阅 `run_started` / `run_ended` / `run_settled` / `run_discarded` / `run_faulted`”需要补一句：`run_settled` 不产生独立 cue，声音由结算的三个事件承载。
- `settlement-and-entry-fee.md` 的下游表把「音效反馈」记为 `GDD 未写`，应改为 `GDD 已写（design/gdd/audio-feedback.md）`；其接口行可以补上“结果 cue 在数字滚动结束时起播”的时序约定。
- `scene-flow.md` 的下游表把「音效反馈」记为 `GDD 未写`，应改为 `GDD 已写（design/gdd/audio-feedback.md）`；其接口行“订阅状态切换事件”可以补上 `Paused` / `Resumed` / `Quit` 三个具体点。
- `design/gdd/systems-index.md` 第 17 行：`Status` 由 `Not Started` 改为 `Designed`，`Design Doc` 指向 `design/gdd/audio-feedback.md`；`Depends On` 维持「餐局流程, 结算与入场费, 场景与流程管理」不变（与索引 Dependency Map 一致）。`Design docs started` +1。
- `局内 HUD` 与 `界面流程` 的 GDD 尚未编写；它们编写时必须在下游 / 上游表中列出「音效反馈」，并实现上表的三个事件。这是本 GDD 对它们的硬性接口要求（产品意义上仍是软依赖：没有音频，游戏照常运行）。

## Visual/Audio Requirements

本系统就是音频系统，`Visual/Audio Requirements` 这一节固定**声音方向与边界**，不固定录音。方向由 audio-director 提供（Lean 模式下按“音频反馈即本系统的核心”特例咨询），具体素材由未来的声音规范与 `/asset-spec` 产出。

### 声音身份：食堂价目板

1. **材质**：干、实的食堂表面——纸、陶瓷、木、金属托盘、收据打印机、收银机。不做管弦 stinger、不做赌场金币式 shimmer、不做合成 whoosh。
2. **频段**：中频靠前、瞬态紧、衰减短、低中频温暖。避免 sub-bass 轰鸣与高频亮片——两者都会被读成“奖励”，而亏损与丢弃绝不允许听起来像奖励。
3. **密度**：稀疏。静止状态就是安静；一次动作只给一声。重复型 cue（吃菜、点击、拒绝、数字 tick）必须短、干、无长混响尾巴，否则快速操作时会堆叠成噪声。
4. **音高纪律**：数字上涨用**有上限、每次交易重置**的上行音型（支柱 4）；绝不累积爬升到庆祝音区。亏损用下行音型。

### 相对响度

以结算结果为 0 dB 参考，绝对增益与 LUFS 留给资产规范：

| 家族 | 相对响度 | 说明 |
|---|---|---|
| 结算结果（`broke_even` / `loss`） | 0 dB | 全局最响、最清晰；回本 ≤ 0.4 s |
| 吃菜 / 容量变化 | −6 … −10 dB | 确认反馈，不能比结算响 |
| 数字滚动 tick | −14 … −20 dB | 纹理，不是事件；密度受 `roll_ticks` 限制 |
| 错误 / 拒绝 | −10 … −14 dB | **不得响于吃菜**；是提示，不是警报 |
| UI 点击 | −16 … −22 dB | 存在感最低；不做焦点音 |
| 环境声 / 音乐（VS 不做） | −26 … −32 dB | 若未来加入，结算期间按 `duck_db` 闪避 12 dB |

### 包含与刻意排除

**包含**：控件点击、吃菜提交、跳波、主动结束取餐、拒绝（放不下）、数字滚动 tick 与落定、收据打印、回本、亏损、开局、结束、丢弃、故障、暂停 / 恢复 / 菜单 / 退出。

**排除**：焦点移动 tick、背景环境声（Vertical Slice 也不做）、背景音乐、时钟滴答或任何暗示手速的持续节奏、亏损警报音、连击音、结算“你本可以”提示音、开局倒计时。理由见 Core Rules 与事件目录。

### 边界：本 GDD 固定什么、留什么

| 本 GDD 固定 | 留给声音规范 / 资产规范 |
|---|---|
| 事件名、触发时机、cue id 目录 | 每个 cue 的录音、合成方式、乐器 |
| cue 的存在与不存在 | 绝对增益、LUFS、采样率与格式 |
| 优先级、并发上限、冷却、抢占 | 变体数量与具体素材 |
| 相对响度范围与闪避深度 | 空间化（本作不做 3D 音频） |
| 结算结果 ≤ 0.4 s、回本 cue 不庆祝 | 具体尾音、混响、母带处理 |
| 无障碍：声音永不是唯一信息通道 | 中间件与实现 API |

> 📌 **Asset Spec** — Visual/Audio requirements are defined. 声音规范（sound bible）批准后，运行 `/asset-spec system:audio-feedback` 生成逐 cue 的素材描述、时长与生成/录制提示，再进入音频制作。

> `art-director` 未参与——本系统没有视觉资产，视觉反馈由 HUD 与结算屏的既有规范负责。`audio-director` 已提供方向；`sound-designer` 的逐 cue 规格属于声音规范，不在本 GDD 内。

## Game Feel

**目标手感：声音是数字的影子，不是数字的前奏。** 玩家按下的同一帧，数字变了，声音也响了；不存在“先响一下再变数”或“等动画走完再响”。

- **即时**：从事件到 cue 请求不超过 1 帧；派发开销中位数 ≤ 0.25 ms、峰值 ≤ 1.0 ms，在一帧 16.7 ms 的预算内增加不超过 2 ms。音频不得成为帧率的下限。
- **安静是默认**：没有环境声、没有背景音乐、没有持续节奏。安静让每一声都有意义；持续的声音会让玩家学会忽略它。
- **一次动作一声**：吃菜不叠连击，结算不叠庆祝，错误不叠警报。层级由优先级决定，不由音量堆叠决定。
- **结算像打印**：收据打开的纸声 → 数字滚动的 tick → 结果落定的一声。回本短促、干净；亏损下行、安静、不羞辱。玩家可以跳过滚动，声音跟着跳过，不拖住按钮。
- **失败平静**：亏损、放弃、故障、退出都不使用上扬或奖励音色。玩家应当从声音里听出“这一局结束了”，而不是“你赢了”或“你输了”。
- **不做**：音频驱动的动画、需要听声音才能理解的规则、延迟确认音、焦点音、时钟滴答、开场倒计时。任何让玩家等声音的东西都违反支柱 4。

## UI Requirements

本系统不拥有任何屏幕，但对「界面流程」的设置区提出以下要求。音效设置属于**设置文件**，不属于生涯存档（存档 GDD 已明确设置独立）。

| 设置项 | 取值 | 行为 |
|---|---|---|
| 主音量 | −80 … 0 dB，滑杆 | 作用于 Master 总线；拉到 −80 视为静音，按 `dropped:muted` 处理 |
| 音效音量 | −40 … 0 dB，滑杆 | 作用于 SFX 与 UI 总线；不影响音乐（若未来加入） |
| 静音 | 开 / 关 | 立即停止全部 voice；解除后不补播 |
| 减少重复音效 | 开 / 关，默认关 | 开启时：可重复 cue 的 `cooldown_ms` × 2（`ui.click`、`eat.reject`、`wave.skip`、`meal.end`）；`TICK_MS` × 2（每次滚动最多 5 个 tick）；变体轮换避免立即重复。**不影响**结算结果、生命周期 cue 与 `eat.commit`——确认与结果是信息，不是噪声 |

**无障碍**：

- 每一条 cue 都必须有视觉对应物：数字变化可见、容量条可见、结算文字给出回本 / 未回本、非法操作有可见状态、回合数可见。**不存在只在音频里出现的信息**，游戏可以完全静音通关。
- 不依赖立体声声像传达信息；单声道设备上语义不变。
- 音高只是装饰，不是判断依据——不能要求玩家靠“听音高”判断赚亏。
- 音量与静音控件必须可由键盘与手柄操作，并与其它设置项使用同一焦点规范。

**调试**：debug 构建提供 cue 日志浮层（最近 20 条：cue、决策、原因、活跃 voice 数）；release 不显示，但日志仍写入环形缓冲，供 QA 抓取。

> **📌 UX Flag — 音效反馈**：本系统对「界面流程」的设置区提出音量、静音、减少重复音效三项要求，并要求每一条 cue 有视觉对应物。Phase 4 写 epic 之前，`/ux-design` 需要覆盖设置屏的音频分区；相关 story 引用 `design/ux/settings.md`，不要直接引用本 GDD。

## Cross-References

| 系统 / 文档 | GDD | 关系 |
|---|---|---|
| 餐局流程 | `design/gdd/run-flow.md` | 事件源：`run_started`、`action_committed`、`run_ended`、`run_discarded`、`run_faulted`；`run_settled` 不映射 cue |
| 结算与入场费 | `design/gdd/settlement-and-entry-fee.md` | 事件源：`settlement_started`、`broke_even`、`loss`；回本 cue ≤ 0.4 s、亏损不庆祝、结果 cue 与数字滚动对齐 |
| 场景与流程管理 | `design/gdd/scene-flow.md` | 顶层状态与暂停 / 恢复 / 退出；驱动本系统的 `Ready` / `Frozen` / `Shutdown` |
| 输入 | `design/gdd/input.md` | 本系统不订阅原始输入；焦点必须有可见指示，声音不是唯一通道 |
| 局内 HUD（未写） | — | 前置契约：`dish_action_rejected`、`number_roll_started/finished` |
| 界面流程（未写） | — | 前置契约：`ui_button_activated`、收据滚动事件、音频设置 |
| 游戏概念 | `design/gdd/game-concept.md` | 支柱 4「数字看得见地涨」、支柱 2「没有白吃的菜」、视觉锚点「食堂价目板」、MDA Sensation |
| 存档 | `design/gdd/save-system.md` | 音频设置属于设置文件，不属于生涯存档 |
| 声音规范（未来） | `design/audio/`（待建） | cue → 音频流、`base_db`、时长、变体与录音素材的所有者 |

## Acceptance Criteria

自动化测试使用**注入时钟与固定帧步长**，不使用墙钟毫秒；所有判定从 cue 日志读取。每条标准都可以由一个没读过本 GDD 的 QA 用日志独立判断。

**事件与去重**

1. **GIVEN** 新会话，**WHEN** `run_started(run_id=1)` 触发，**THEN** 日志恰好一条 `sfx.run.start`、`decision=admitted`、`active_count` +1；**WHEN** 同一 `run_id` 再触发一次，**THEN** `dropped:dedupe`，`active_count` 不变。
2. **GIVEN** `run_started` 与 `run_ended` 在同一帧，**WHEN** 处理，**THEN** `sfx.run.end` 为 `admitted`，`sfx.run.start` 为 `suppressed:preempted`，活跃 voice 数不超过 1。
3. **GIVEN** 两个不同的 `action_committed(eat)` 在两个连续帧，**WHEN** 处理，**THEN** 两条 `sfx.eat.commit` 都 `admitted`；**GIVEN** 同一 `action_id` 在同一帧重复广播，**THEN** 第二条 `dropped:dedupe`。
4. **GIVEN** 同一帧收到 5 次 `ui_button_activated`，**WHEN** 处理，**THEN** 恰好 1 条 `sfx.ui.click` 为 `admitted`，其余 4 条为 `dropped:cooldown`，`active_count` 只 +1。
5. **GIVEN** 一条 `run_settled`，**WHEN** 处理，**THEN** 日志中没有任何 `decision=admitted` 的条目。

**优先级与并发**

6. **GIVEN** 已有 8 个 voice（UI 3 + Gameplay 3 + Lifecycle 2），**WHEN** `broke_even` 到达，**THEN** `sfx.settle.broke_even` 为 `deferred:pending_outcome`；**WHEN** 最后一个 `number_roll_finished` 到达，**THEN** 它才 `admitted`、无 `stolen_voice_id`、`active_count = 9`。
7. **GIVEN** 10 个 voice 全满且最旧的低优先级 voice 已起播 ≥ 50 ms，**WHEN** 一个更高优先级请求到达，**THEN** 该请求 `admitted` 且 `stolen_voice_id` 指向最旧 voice，`active_count = 10`。
8. **GIVEN** 一个 Critical 正在播放且已起播 30 ms，**WHEN** 第二个 Critical 到达，**THEN** `dropped:pool_full`；**WHEN** 第一个已起播 60 ms，**THEN** 第二个 `admitted`、旧 voice 被停止、`active_count = 1`。
9. **GIVEN** 10 个 voice 全满且没有任何严格更低优先级的 voice，**WHEN** 新请求到达，**THEN** `dropped:pool_full`，无 voice 被停止。

**结算时序**

10. **GIVEN** `settlement_started` 与 `broke_even` 在同一帧，**WHEN** 处理，**THEN** `sfx.settle.print` 立即 `admitted`；`sfx.settle.broke_even` 为 `deferred:pending_outcome`，**WHEN** `number_roll_finished` 到达，**THEN** 它才 `admitted`，且两条 cue 的播放区间不重叠。
11. **GIVEN** `settlement_started`、同一帧的 `broke_even` 与 `loss`，**WHEN** 处理，**THEN** `sfx.settle.print` 立即 `admitted`；结果 cue 待 `number_roll_finished`；到齐后 `sfx.settle.loss` 为 `admitted`，`sfx.settle.broke_even` 为 `suppressed:duplicate_outcome`，结果活跃 voice 数为 1。
12. **GIVEN** 结果 cue 待播，**WHEN** `run_discarded` 到达，**THEN** `sfx.run.discard` 为 `admitted`、结果 cue 为 `suppressed:preempted`，结果 cue 的 `voice_id` 为空。
13. **GIVEN** 结果 cue 正在播放，**WHEN** `run_faulted` 到达，**THEN** 结果 voice 被停止，`sfx.run.fault` 为 `admitted`，不受 50 ms 下限限制。
14. **GIVEN** 结果 cue 待播，**WHEN** 玩家跳过滚动（`number_roll_finished(skipped=true)`），**THEN** 结果 cue 立即 `admitted`，之后不再出现该次滚动的 `sfx.num.tick`。

**暂停、静音与设备**

15. **GIVEN** 顶层进入 `Paused`，**WHEN** 任何 cue 请求到达，**THEN** 无 voice 分配、记 `dropped:paused`；**WHEN** 恢复后的 60 帧内，**THEN** 没有任何来源时间早于恢复时刻的 cue 被 `admitted`。
16. **GIVEN** 主音量静音，**WHEN** cue 请求到达，**THEN** 记 `dropped:muted`、零 voice；**WHEN** 解除静音，**THEN** 60 帧内无补播，且一条新 cue 正常 `admitted`。
17. **GIVEN** 无音频设备（headless），**WHEN** 任意 cue 请求，**THEN** 日志照常记录、零 voice、无 ERROR 条目、派发时间在预算内。
18. **GIVEN** 某 cue id 没有音频流，**WHEN** 请求两次，**THEN** 零 voice、该 id 恰好一条 WARN、第二条记 `dropped:no_stream`、无替代 cue。

**公式**

19. **GIVEN** `base_db=−8`、SFX 总线 0、主音量 0、音效音量 −6，**WHEN** 请求该 cue，**THEN** 日志 `volume_db = −14.0`；**GIVEN** 主音量为 −80，**THEN** 该请求 `dropped:muted`。
20. **GIVEN** 滚动 `start=−85`、`end=19`、`duration=600 ms`，**WHEN** 滚动执行，**THEN** 恰好 10 条 `sfx.num.tick`，`time_ms` 为 0、60 … 540，音高严格递增且最后一个等于 `2^(7/12)=1.498`（容差 ±0.001），t=600 ms 后没有 tick；**WHEN** `duration=0`，**THEN** 0 条 tick、0 条 `sfx.num.land`、一条 `dropped:zero_duration`。
21. **GIVEN** 滚动 `start=0`、`end=−31`，**WHEN** 执行，**THEN** 音高严格递减，最后一个为 `2^(−5/12)=0.749`；跨零的 `start=−31`、`end=19` 不产生除零，最后一个为 `2^(7/12)=1.498`。
22. **GIVEN** Music 总线在基准音量、**WHEN** 结算结果 cue 起播，**THEN** 50 ms 内 Music 下降 12 dB，cue 结束后 120 ms 内回到基准；连续两条结果 cue 不叠加闪避深度。

**变体、日志与性能**

23. **GIVEN** 某 cue 有 3 个变体、连续请求 6 次，**WHEN** 执行，**THEN** 3 个变体都被使用、没有相邻重复、两次运行序列完全相同。
24. **GIVEN** 未知 cue id 或格式错误的事件，**WHEN** 请求，**THEN** `dropped:invalid`、一条 WARN、不崩溃。
25. **GIVEN** 60 FPS 最低规格设备，**WHEN** 连续请求 100 条 cue，**THEN** 每次决策在同一帧内写入日志，派发耗时中位数 ≤ 0.25 ms、峰值 ≤ 1.0 ms；运行期间 AudioStreamPlayer 节点数不增长，派发路径无 `load()` / `instantiate()`。
26. **GIVEN** 任意一轮测试，**WHEN** 读取日志，**THEN** 每条记录的 `decision` 与 `reason` 都落在封闭词表内，`active_count` 在 0 … 10 之间且从不因抢占而增加。

### 证据要求

| 证据 | 位置 | 门禁 |
|---|---|---|
| 自动化单元测试（headless、注入时钟，覆盖 1–26） | `tests/unit/audio_feedback/` | **BLOCKING** |
| 同一轮测试保留的 cue 日志 | `production/qa/evidence/audio-feedback-[date]-cuelog.txt` | **BLOCKING** |
| 真机脚本场景录音 + 对应时间戳日志 | `production/qa/evidence/audio-feedback-[date].wav` 与 `…-cuelog.txt` | **BLOCKING** |
| audio-director 试听签核（混音、优先级、闪避、手感） | `production/qa/evidence/audio-feedback-[date]-listening.md` | **BLOCKING** |
| 设置屏音频分区截图（若本 story 触及） | `production/qa/evidence/` | **BLOCKING**（UI 行） |

单元测试只证明决策正确，不证明“听起来对”；录音证明输出，日志证明决策，两者缺一不可。真机观察按 `.claude/docs/run-and-observe.md` 执行，仅跑测试不算完成。

## Open Questions

| # | 问题 | 当前假设 | 决策者 | 目标时间 |
|---|---|---|---|---|
| 1 | `settlement_started` / `broke_even` / `loss` 由谁在什么时刻广播？结算 GDD 定义了语义，但没有定义发射者与发射时机 | 假设结算系统在 `settle` 成功、收据数据可用时发 `settlement_started`，结果语义同时到达；音频负责把结果 cue 延后到滚动结束 | 结算 GDD 复审 / 编排会话 | `/design-review` 之前 |
| 2 | 声音规范（sound bible）尚不存在：cue → 流、`base_db`、时长、变体都没有归属 | 由 audio-director / sound-designer 在 Vertical Slice 音频制作前建立；本 GDD 只固定 cue 目录与相对响度 | audio-director | Vertical Slice 音频里程碑前 |
| 3 | `number_roll_started/finished`、`dish_action_rejected`、`ui_button_activated` 是强加给未设计的 #15 / #16 的新契约 | 两份 GDD 编写时必须采纳；若它们选择不同的滚动实现，需回来改本 GDD 的事件目录 | #15 / #16 作者 | 两份 GDD 编写时 |
| 4 | 是否需要“时间将尽”的离散提示音？ | **MVP 与 VS 不做**。它需要 #6 / #15 先提供 `turn_warning(remaining)` 事件；在没有事件前不发明声音 | game-designer / 时间与回合 | 首次可玩性测试后 |
| 5 | 焦点音是否完全不做？ | **不做**。它会奖励手快并制造噪声；若键盘玩家反馈“找不到焦点”，先改视觉焦点，不加声音 | ux-designer | 可访问性测试后 |
| 6 | 暂停时音频处理必须继续运行（否则收不到 `Paused` / 恢复通知）；具体用 `process_mode = ALWAYS` 还是独立单例属于实现选择 | 行为要求已写进 States；实现方式留给 ADR | lead-programmer | 技术设计阶段 |
| 7 | cue 目录是否需要进 `design/registry/entities.yaml`？注册表目前只有 entities / items / formulas / constants，没有 cue 类型 | 暂不注册；cue id 是本 GDD 与声音规范之间的契约，等声音规范建立后再决定是否扩展注册表 | 编排会话 / 工具链 | 声音规范建立时 |
| 8 | Godot 4.7 下音频设备热插拔的行为（拔出后 `AudioServer` 是否报错、恢复后是否自动重连）未在引擎参考中覆盖 | 假设可以轮询 / 订阅设备变化并置 `device_available`；若引擎行为不同，静默降级分支需要改写 | godot-specialist | 原型阶段验证 |
| 9 | “减少重复音效”默认关闭是否合适？ | 默认关；开启后只降低重复 cue 密度，不影响结算与确认反馈 | ux-designer / 可访问性测试 | VS 试玩后 |
| 10 | 背景音乐与环境声是否永远不做？ | VS 不做；若未来加入，`duck_db` 生效，音乐内容归 audio-director | creative-director / audio-director | Full Vision 规划时 |
| 11 | cue 日志在 release 构建是否保留？环形缓冲有内存成本，但对玩家 bug 报告有价值 | 保留 10,000 条环形缓冲；debug 浮层只在 debug 构建显示 | lead-programmer | 技术设计阶段 |
| 12 | 音频延迟预算（设备输出延迟）是否需要在设置中提供校准？ | VS 不需要；本作是回合制，没有对音画同步的硬要求 | audio-director | VS 试玩后 |
