# Control Manifest

> ✅ **ALL SOURCE ADRs ARE `Accepted` — 9/9（2026-11-04 用户决定）。**
> 本清单的每一条规则都出自 **Accepted** ADR：规则内容已定，可作为已决编码规则使用。
> `accepted = 9`、`proposed = 0`；本次接受只改状态、未改规则内容，Manifest Version 保持不变。
> 任何 ADR 被修订后，必须重新运行 `/create-control-manifest update`；
> `/story-readiness` 的版本比对只能证明 story 写在同一天的规则上，不能证明规则此后未被修订。

> **Engine**: Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`）
> **Last Updated**: 2026-11-04
> **Manifest Version**: 2026-11-04
> **ADRs Covered**: ADR-0001, ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006, ADR-0007, ADR-0008, ADR-0009（**全部 `Accepted`**）
> **Status**: Active — regenerate with `/create-control-manifest update` when ADRs change
> **Technical review**: `TD-MANIFEST skipped — Lean mode`（`review_gate: skill-gates=skip`；无 technical-director 门）

`Manifest Version` 是生成本清单的日期，与 `Last Updated` 恒等。Story 写入时会嵌入它，`/story-readiness` 用它找出写在旧规则上的 story。

本清单出自各 ADR 的 `Decision`、`Alternatives Considered`、`Performance Implications`、`Engine Compatibility` 四节，加上 `project.yaml` / `.claude/docs/technical-preferences.md` 与 `docs/engine-reference/godot/`。每条规则都带来源；要理由，读那条 ADR。**规则之间的冲突以 ADR 为准，不以本清单为准。**

---

## Foundation Layer Rules

*Applies to: scene management, event architecture, save/load, engine start-up*

### Required Patterns

- **零 autoload**：`AppRoot` 是 `res://src/main/main.tscn` 的根，唯一组合根；服务由构造/attach 注入，不用 `get_node()` 寻址、不用依赖袋 — source: ADR-0004
- **服务形态**：纯逻辑是 `RefCounted`（`DataConfig`、`SaveSystem`、`RunFlow`、`TimeTurns`、`StomachCapacity`、`DishValues`、`CloyingAndTaste`、`MenuWaves`、`Settlement`、`MetaUpgrades`、`Unlocks`、`Records`、`SessionRng`、`BootSequence`、`ScreenFactory`）；只有需要 `_process` / `_input` / `Tween` / `AudioStreamPlayer` / `process_mode` 的才是 `Node`（`SceneFlow`、`InputRouter`、`UIRoot`+四宿主、`HudHost`、`AudioDirector`+voice 池宿主、`SaveHeartbeat`） — source: ADR-0004
- **`SceneFlow` 是顶层状态的唯一所有者**；`state_changed(from, to, payload)` **只有一个订阅者 `AppRoot`**，按固定六步同步扇出：`set_frozen` → `ui.reset_for_state` → `hud.sync_for_state` → `audio.on_flow_state` → `onboarding.on_flow_state` → `fade.play` — source: ADR-0004
- **发射侧契约**：状态先落定再发信号；被拒绝的操作不发；同一事实每 `run_id` 至多一次；同步派发，不是队列；处理器重入深度 ≤ 1 — source: ADR-0004
- **暂停唯一机制是 `SceneTree.paused`**，由 `SceneFlow` 在切换生效时同步写入；`PROCESS_MODE_ALWAYS` 白名单封闭：`AppRoot`、`SceneFlow`、`InputRouter`+分发节点、`UIRoot`+四个宿主+`FadeLayer`、`SaveHeartbeat`、`AudioDirector`+voice 池宿主。`HudHost` 与全部 gameplay 保持默认 `INHERIT` — source: ADR-0004
- **启动八步固定**：CONSTRUCT → LOAD_DATA → LOAD_MILESTONES → LOAD_SAVE → BASELINE → SCENE_BOOT → UI_BUILD → READY — source: ADR-0004
- **`Quit` 序列固定**：flush → 释放写锁 → 音频关闭 → quit — source: ADR-0004
- **内容格式 = 带类型 `Resource` 记录（一记录一 `.tres`）+ 显式清单资源**；记录必须登记在清单里才能进入游戏 — source: ADR-0001
- **`DataConfig` 状态机** `Unloaded→Loading→Validating→Ready/Failed`，全有或全无；`Ready` 之后才允许 `Save.load()` 与任何 gameplay 查询；失败走 `Boot → Error` — source: ADR-0001
- **id 命名空间** `^[a-z][a-z0-9_]*$` + 类型前缀（`dish_` / `upgrade_` / `station_` / `ms_`），重复即 ERROR；`published_ids.txt` + CI 基线；发布后改名禁止 — source: ADR-0001
- **六阶段校验 + `Diagnostic` 码**；占位图标缺失是 `PLACEHOLDER_MISSING`（WARNING，不阻断），退回 `ThemeDB.fallback_icon` — source: ADR-0001
- **存档格式**：规范 JSON、单行、UTF-8 无 BOM、`JSON.stringify(model, "", true, false)`；**所有整数以十进制字符串传输**（避免 JSON number 的 float 解析）；解码统一规则 `^-?(0|[1-9][0-9]*)$` → int64；`last_saved_at` 是唯一被排除在确定性复现之外的字段 — source: ADR-0002
- **写盘五步**：校验 → 写 `.tmp` → flush → 主档 → `.bak` → `.tmp` → 主档；任一步失败删除 `.tmp`，不动主档与备份 — source: ADR-0002
- **写锁**：原子创建目录 + `owner.json` + 每 30 s `heartbeat`（由 `PROCESS_MODE_ALWAYS` 的 `SaveHeartbeat` 驱动）；90 s 存活超时、24 h 上限、按 rename 回收 — source: ADR-0002
- **单写者**：`Blocked` / `ReadOnly` 实例不触碰任何文件；锁存活时任何原因都只给「退出」「重试」，`retry_load()` 是唯一重新探测入口 — source: ADR-0002
- **迁移**：单向纯函数链（不读时钟、不读状态、不读网络）+ 连续性校验，跑在原始树上；未知 id 按 id 原样保留 — source: ADR-0002
- **屏幕与模态**：`UIRoot` 不是 autoload，一屏一个 PackedScene；屏幕栈 ≤ 2、模态栈 ≤ 2；每次模态 push 调 `input.push_context(&"Modal")`、pop 调 `pop_context()`，并断言深度一致 — source: ADR-0003
- **顶层切换时不调 `pop_context()`**：`SceneFlow` 已用 `request_context(default, clear_stack := true)` 清栈，`UIRoot` 只把模态栈清空并归零计数 — source: ADR-0003
- **焦点**：声明式 `FocusSpec`（`order` + `neighbours`），装载时预解析到首个可用控件；模态设焦点陷阱与 `set_blocked(true)`；双焦点分开记录、分开绘制；鼠标移动绝不改键盘焦点、绝不 `grab_focus()` — source: ADR-0003
- **输入四上下文**（`Blocked`/`Menu`/`Run`/`Paused`/`Modal`）的允许表是唯一过滤源；`MAX_CONTEXT_DEPTH = 2`；同值 `push_context` 必须加深并发出 `context_changed(Modal, Modal)`；`pop_context` 在空栈返回 `false`，绝不弹默认上下文 — source: ADR-0005
- **`ActionSink` 只在 `Menu`/`Modal`/`Paused` 投递**，`Run` 不投递（HUD 自己轮询）；`consume_action` 只在 `Blocked` 生效、首个消费者获胜；accept/cancel 处理时调 `set_input_as_handled()` — source: ADR-0005
- **设备识别只看事件类型**（`InputEventKey` / `InputEventMouse*` / `InputEventJoypad*`）；`DEVICE_ID_*` 只用于日志 — source: ADR-0005
- **导航重复与摇杆回滞是纯函数**（`NavRepeater` / `StickGate`，参数在 `InputTuning`），合成 `InputEventAction` 经 `push_input` 投递并带 `synthetic_repeat` meta — source: ADR-0005
- **设置是机器级文件**：`user://config/settings.json`，7 个封闭键，只在模态关闭 / 退出 / 隐式重试时写；不迁移、不进生涯档、不进云/备份、无锁 — source: ADR-0008
- **所有玩家可见字符串走 `tr()`**，key 语法 `^[a-z][a-z0-9_]*(\.[a-z0-9_]+){2,4}$`，域 `ui/hud/err/content/a11y`；记录的 `display_name` 是 key（`DISPLAY_NAME_NOT_A_KEY`） — source: ADR-0008

### Forbidden Approaches

- **Never autoload 任何服务** — 隐式全局状态无法在无头测试里替换，且构造顺序失控 — source: ADR-0004
- **Never 依赖袋 / 服务定位器**（`Services.get(&"save")`） — 依赖不可见、不可测 — source: ADR-0004
- **Never 给 `state_changed` 挂第二个订阅者并依赖 `connect()` 顺序** — 扇出顺序是硬契约 — source: ADR-0004
- **Never 手写 `frozen` 标志替代 `SceneTree.paused`** — 缺帧推进闸门 — source: ADR-0004
- **Never 让 `Save` 直接调用 `unlocks.normalize()`** — 命名空间内容归各自所有者，只经 `commit` 写入 — source: ADR-0004
- **Never 用 `DirAccess` 扫描 `res://data/*` 发现记录** — 导出后行为不确定，且清单是显式契约 — source: ADR-0001
- **Never 运行期用 JSON/CSV 当内容格式**（含「作者格式 + 运行转换」的混合方案） — 失去类型与 Inspector 校验 — source: ADR-0001
- **Never 内容热重载（MVP）** — 与全有或全无的启动门冲突 — source: ADR-0001
- **Never 用 `FileAccess.store_var` 存档** — Variant 编码不可读、跨版本不稳 — source: ADR-0002
- **Never 自定义二进制存档格式** — 无法人工诊断、无迁移夹具 — source: ADR-0002
- **Never 每字段 schema JSON** — 体积与解析开销不划算 — source: ADR-0002
- **Never 端口锁 / 命名互斥体** — 跨平台语义不一致 — source: ADR-0002
- **Never 用 PID + `OS.is_process_running` 判存活** — 它只看得到子进程；改用心跳（`save-system.md` 规则 21 待修订） — source: ADR-0002
- **Never 每屏一个场景 + `SceneTree.change_scene_to_file()`** — 丢 UI 层与栈模型 — source: ADR-0003
- **Never 单一巨型 `ui.tscn` 靠 `visible` 切换** — 生命周期与焦点不可控 — source: ADR-0003
- **Never 依赖 `find_next_valid_focus()` 的几何解析** 作为焦点顺序来源 — 布局一变就乱 — source: ADR-0003
- **Never 让屏幕自己轮询 `is_action_just_pressed`** — 绕过允许表与消费位 — source: ADR-0005
- **Never 用 `_unhandled_input` 链让屏幕依次处理输入** — 顺序隐式、不可断言 — source: ADR-0005
- **Never 让界面也走 `consume_action`** — 消费位只服务 `Blocked` — source: ADR-0005
- **Never 用 `event.echo` 实现导航重复** — 回显时机不可控 — source: ADR-0005
- **Never 把同值 `push_context` 当幂等空操作** — 模态深度必须与栈深度配对 — source: ADR-0005
- **Never 用 `ConfigFile` / `.cfg` / 二进制 / `store_var` 存设置** — 全项目只保留一种序列化格式 — source: ADR-0008
- **Never 每次修改立即写盘或加防抖轮询** — 写入时机是封闭集合 — source: ADR-0008
- **Never 把设置存进生涯档的命名空间** — 删除生涯必须保持设置文件逐字节不变 — source: ADR-0008
- **Never 让 `display_name` 保留中文字面量**（只对界面 chrome 用 `tr()`） — 记录名也必须可翻译 — source: ADR-0008

### Performance Guardrails

- **DataConfig**：300 条记录元数据 < 2 MB；`Loading + Validating` p95 ≤ 250 ms；`Validating` p95 ≤ 100 ms — source: ADR-0001
- **Save**：8.5 KiB 存档 CPU < 1 ms；`load` p95 ≤ 100 ms；`commit`/写盘 p95 ≤ 50 ms — source: ADR-0002
- **Screen mount**：一次挂载 = 一次 `instantiate()` + `add_child()`，p99 在一帧内；单帧最多 2 次挂载 + 2 次卸载 — source: ADR-0003
- **Boot**：数据 p95 ≤ 250 ms + 存档 p95 ≤ 100 ms（`Fresh` 时再加一次基线写 ≤ 50 ms） — source: ADR-0004
- **Input**：事件到消费者 ≤ 1 帧（同步投递）；上下文栈 ≤ 2 项；`_pressed` / `_consumed` ≤ 8 项；每帧不重新分配 — source: ADR-0005
- **Settings**：文件 < 1 KiB；write < 2 ms；load p95 ≤ 5 ms — source: ADR-0008

---

## Core Layer Rules

*Applies to: core gameplay loop, main player systems, physics, collision*

### Required Patterns

- **`RunFlow` 是 `run_id` / 生命周期 / `RunLog` / `RunResult` 的唯一所有者**；`run_id = 0` 保留为「未分配」，永不作为有效局 — source: ADR-0007
- **结算只在一处提交**：`Run → Menu` 被接受时，在 `UIRoot.reset_for_state` 内、收据挂载之前；`run_settled` 只是**预通知**，持久化与交付发生在 `Run → Menu` — source: ADR-0007
- **事务顺序固定**：幂等闸（`run_id` + 摘要）→ `Records.prepare_run` → `MetaUpgrades.prepare_run_result` →（MVP 无 unlocks 槽位）→ merge → `Save.commit` → adopt 与发信号 → 快照 — source: ADR-0007
- **失败矩阵**：Records 致命 → 整笔中止、`commit` 次数为 0；`WALLET_OVERFLOW` → 省略 `progression`、仍提交 `records`；写盘失败 → `UNSAVED` + 两个命名空间都 dirty，**不回滚**；重入 → `APPLY_REENTRANT`；并发 → `ORCHESTRATOR_BUSY` — source: ADR-0007
- **`SettlementTransaction` 是纯 `RefCounted`**，在 CONSTRUCT 阶段构造并注入 `UIRoot`；串行、不嵌套、不重入 — source: ADR-0007
- **钱包只由 `MetaUpgrades` 计算**；`Settlement` 不持有 Save / Records / Meta 引用（纯函数、无 I/O） — source: ADR-0007
- **唯一随机源是 `SessionRng`**（构造时 `randomize()` 一次）；所有可复现随机走种子派生：`FNV-1a` 双道 over `UTF8(station_id) ‖ 0x00 ‖ LE64(seed) ‖ LE16(attempt)` — source: ADR-0004, ADR-0006
- **洗牌 = 规范序 + 反向 Fisher–Yates + 拒绝采样**；划分与可见度在 `initialize` 一次性冻结 — source: ADR-0006
- **健康门 = 逐波重置 DP**：状态 `(i, c, t)`、五条转移、`NODE_BUDGET = 10000`、`STATE_CELL_BUDGET = 262144`，超预算 `TRUNCATED → DEGRADED`；整数阈值 `reachable * 10 < baseline * 9`；`MAX_ATTEMPTS = 21`，并列取更早，`DEGRADED` 不再重掷，恰好一条 warning — source: ADR-0006
- **逻辑不读墙钟、不用浮点**：全部整数运算（int64 语义），确定性纯核心 — source: ADR-0006
- **`opt_value` 是内部基准**：只用于健康门与 CI/设计，不暴露公开 API、不进存档 / HUD / 结算（`dish-values.md` 需补这一句例外） — source: ADR-0006

### Forbidden Approaches

- **Never `randi()` / `randf()` / `Array.shuffle()` 或任何全局 RNG** — 不可复现、不可跨平台 — source: ADR-0006
- **Never 用 `hash(dish_id, seed)` 排序代替洗牌** — 分布与均匀性不成立 — source: ADR-0006
- **Never 用 `String.hash()` 派生种子** — 跨版本不保证稳定 — source: ADR-0006
- **Never 用完整背包（不按波重置）求「可达最优」** — 与逐波规则不符且更贵 — source: ADR-0006
- **Never 重掷用 `seed + 1` 递增** — 候选序列相关，破坏金标准 — source: ADR-0006
- **Never 让 `RunFlow` 直接编排结算事务** — 交易对象是唯一编排者 — source: ADR-0007
- **Never 让 `AppRoot` 充当事务编排者** — 组合根只做扇出与装配 — source: ADR-0007
- **Never 常驻 `SettlementService`（`Node` 或 autoload）** — 生命周期比每个屏都长，且不可注入 — source: ADR-0007
- **Never 在 UI Flow 内联拼事务** — 逻辑不可在无头测试里直接跑 — source: ADR-0007
- **Never 两次提交（先钱包、再纪录）** — 破坏原子性与失败矩阵 — source: ADR-0007

### Performance Guardrails

- **`SettlementTransaction`**：整笔 p95 ≤ 60 ms（一次状态切换内） — source: ADR-0007
- **`Settlement`**：p95 ≤ 1 ms/局，硬上限 5 ms；**`Records.prepare_run`** p99 ≤ 5 ms — source: ADR-0007
- **`MenuWaves`**：单候选 < 0.3 ms；20 次重掷 < 4 ms；`initialize` p95 ≤ 4 ms — source: ADR-0006

---

## Feature Layer Rules

*Applies to: secondary mechanics, AI systems, secondary features*

> 本层是本次评审的最大缺口：21/34 条需求没有 ADR（腻度与口味 5、局外升级 6、解锁 5、纪录 3、结算 2），规则只能来自 ADR-0001 / ADR-0002 / ADR-0007 的横切约束。

### Required Patterns

- **每条记录必须登记在对应清单**；CI 反向断言「`res://data/<type>/*.tres` 的每个文件都被清单引用」 — source: ADR-0001
- **`MilestoneCatalog` 由 `Records` 拥有**，在 `Data Ready` 与 `Save.load()` 之间加载，并入同一道启动门 — source: ADR-0001
- **命名空间写入只经 `Save.commit`**：`progression`（钱包/升级）、`records`、`unlocks` 各自的所有者写自己的键 — source: ADR-0002
- **错误词表封闭**：`COMMIT_FAILED` / `SAVE_DEFERRED` / `APPLIED_UNSAVED` / `ALREADY_SETTLED` / `ALREADY_APPLIED` / `ORCHESTRATOR_BUSY` / `APPLY_REENTRANT`；不用 `null` / `0` 占位失败 — source: ADR-0002, ADR-0007
- **未知 id 在加载与迁移中按 id 保留**，不删除、不解释 — source: ADR-0002

### Forbidden Approaches

- **Never 直接改别人命名空间的键** — 只经 `Save.commit`，否则原子性与回滚失效 — source: ADR-0002
- **Never 把 `unlocks` 键放进 MVP/VS 的结算事务** — 付费解锁是独立购买事务（未来奖励解锁才并入同一 `commit`） — source: ADR-0007
- **Never 用「写盘成功」作为纪录/里程碑信号的前提** — 信号在事务已应用后发，写盘失败只影响快照标记 — source: ADR-0004, ADR-0007

### Performance Guardrails

- 本层无独立预算；Feature 的耗时计入上面 `SettlementTransaction` p95 ≤ 60 ms 与 `Records.prepare_run` p99 ≤ 5 ms — source: ADR-0007

---

## Presentation Layer Rules

*Applies to: rendering, audio, UI, VFX, shaders, animations*

### Required Patterns

- **宿主分层固定**：`HudHost` 10 / `ScreenHost` 20 / `OverlayHost` 25 / `ModalHost` 30 / `NoticeHost` 40 / `FadeLayer` 50；`FadeLayer` 与 `NoticeLayer` 的全屏根 `MOUSE_FILTER_IGNORE` — source: ADR-0003
- **`Settling` 同帧时序**：状态置 `Menu` 并发 `state_changed` → 挂载 `ReceiptScreen` → 卸载 `HudHost` → 渲染该帧 — source: ADR-0003, ADR-0004
- **`PauseOverlay` 不卸载、不重建 HUD**；它只读 `RunFlow` 的 `run_state` 快照，不缓存第二份阶段 — source: ADR-0003
- **双焦点两套 `StyleBox`**；focus 环是纯描边（`draw_center = false`），2 px 描边 + 4 px 预留，且不随文字缩放变化 — source: ADR-0003, ADR-0008
- **过渡只播、不延迟状态**：总时长 ≤ 0.20 s；`reduce_motion = true` 时全部时长置 0 — source: ADR-0003, ADR-0008
- **文字缩放 τ 表**：font 20/25/30、number 28/35/42、label 16/20/24、title 36/45/54、button 高 44/55/66、`w_min` 200/250/300、卡高 176/198/220、顶栏 72/84/96；`gap` 16 与安全边距 36 **不缩放**；列数 4/3/2；不截断，超出走滚动；焦点按 id 保持 — source: ADR-0008
- **CJK 字体三级回退**：Noto Sans SC → `SystemFont` → `ThemeDB`；`hinting = 1` 显式写入导入参数（4.7 默认是 3） — source: ADR-0008
- **`hud_opacity` 只作用于面板背景**，下限 60%，且与文字保持 ≥ 4.5:1 对比度；`PresentationPrefs` 对 `SceneFlow` 不可变 — source: ADR-0008
- **音频生命周期**：`AudioDirector` + `VoicePoolHost` 都是 `PROCESS_MODE_ALWAYS`，都不是 autoload，都不归 `UIRoot` 所有 — source: ADR-0009
- **四条总线** `Master / Music / SFX / UI`，全部 0 dB；无 `Ambience` 总线；音量衰减只在每个 voice 上算一次 — source: ADR-0009
- **暂停 = 显式 `stop()`**：进入 `Paused` 同帧停全部池内 voice 并清空 `pending_outcome`；`sfx.flow.*` 走池外 `UiTransitionPlayer`，是唯一转场例外 — source: ADR-0009
- **池 10 个 voice 预分配 1/3/3/3**，九步准入 + 50 ms 守卫 + 硬顶 `cap[p]`；抢占是手动 ramp，不是 Tween；回收靠轮询 `playing` — source: ADR-0009
- **cue 目录是类型化 Resource**（`res://data/audio/cue_catalog.tres`，`AudioCueData` 子资源）+ 代码里的静态 `CueMap`，`bind_catalog()` 时双向校验；作为 `DataConfig` 的第六个清单同一次校验、同一次发布 — source: ADR-0009
- **变体轮转确定性**；MVP 禁用 pitch 抖动；启用时只能由 ADR-0006 的 `SeedHash` 派生，绝不触碰 `SessionRng` — source: ADR-0009
- **播放日志是 1 万条环形缓冲**，字段与枚举封闭；`priority` 原因码 v1 保留但不发出 — source: ADR-0009
- **可测试性**：`UIRoot` 不注册 `_process`，纯模型 + 可注入 `ScreenFactory` / 时钟；音频留四个端口（时钟、设备、RNG 视图、日志 sink） — source: ADR-0003, ADR-0009

### Forbidden Approaches

- **Never 把音频做成 autoload 单例** — 生命周期与注入都不受控 — source: ADR-0009
- **Never 单播放器 + 起播 `new()` / 播完 `queue_free()`** — 分配抖动、抢占不可控 — source: ADR-0009
- **Never 用 `AudioStreamPolyphonic` / `AudioStreamPlaybackPolyphonic`** — 无逐 voice 的音量/抢占控制 — source: ADR-0009
- **Never 用 `AudioServer` 总线音量做音量映射** — 与每 voice 公式冲突，会双重衰减 — source: ADR-0009
- **Never 把 cue 目录写成 `design/audio/cues.json` 运行时解析** — 绕过启动门与类型校验 — source: ADR-0009
- **Never 把暂停音放进池 / 「除 pause 外都停」** — 冻结帧内无法起播 — source: ADR-0009
- **Never 使用 `PROCESS_MODE_WHEN_PAUSED`** — 恢复帧停止处理，恰好错过恢复音 — source: ADR-0009
- **Never 用 `Tween` 做抢占淡出**，也 **Never 用 `finished` 信号回收 voice** — 暂停/抢占下语义未验证 — source: ADR-0009
- **Never 让鼠标事件调用 `grab_focus()`**，也 **Never 让 `UIRoot` 持有 `HudHost` 引用** — 破坏双焦点与分层 — source: ADR-0003
- **Never 在 150% 溢出时自动缩字号（auto-fit）** — 只允许重排与滚动 — source: ADR-0008
- **Never 用 MSDF/SDF 字体做任意字号** — 钉住三档 τ 表 — source: ADR-0008

### Performance Guardrails

- **Audio request**：p95 ≤ 0.5 ms，最差 2 ms；每帧 tick ≤ 0.05 ms；日志约 1.25 MB 常驻 — source: ADR-0009
- **Themes / 字体**：约 10 MB 常驻（三档字号同一主题） — source: ADR-0008
- **UI 过渡**：≤ 0.20 s，`reduce_motion` 时为 0；`UIRoot` 空闲时零每帧开销 — source: ADR-0003

---

## Global Rules (All Layers)

### Naming Conventions

| Element | Convention | Example | Source |
|---------|-----------|---------|--------|
| Classes | PascalCase | `PlayerController` | `naming.classes` in project.yaml |
| Variables | snake_case | `move_speed` | `naming.variables` in project.yaml |
| Signals/Events | snake_case, past tense | `health_changed` | `naming.signals` in project.yaml |
| Files | snake_case matching the class | `player_controller.gd` | `naming.files` in project.yaml |
| Scenes | snake_case matching the file | `player_controller.tscn` | `naming.scenes` in project.yaml |
| Constants | UPPER_SNAKE_CASE | `MAX_HEALTH` | `naming.constants` in project.yaml |

### Performance Budgets

`project.yaml` 没有 `performance.*` 键 —— 以下取自 `.claude/docs/technical-preferences.md`（legacy fallback）。

| Target | Value | Source |
|--------|-------|--------|
| Framerate | 60 fps | technical-preferences.md → Performance Budgets |
| Frame budget | 16.6 ms | technical-preferences.md → Performance Budgets |
| Draw calls | ≤ 500 | technical-preferences.md → Performance Budgets |
| Memory ceiling | 512 MB | technical-preferences.md → Performance Budgets |

### Approved Libraries / Addons

- **gdUnit4** — 唯一测试框架（`testing.framework`；测试根 `tests/unit/**/*_test.gd`、`tests/integration/**/*_test.gd`）— source: project.yaml `testing`, technical-preferences.md

### Forbidden APIs (Godot 4.7)

已按「本项目实际构建的内容（2D、纯 UI/数值、GDScript、无 3D 物理/无骨骼/无编辑器插件）」裁剪；**未复制整表**。

- `instance()` / `PackedScene.instance()` — 4.0 起改名；用 `instantiate()`
- `yield()` — 用 `await signal`
- `connect("signal", obj, "method")` — 用 `signal.connect(callable)`
- `OS.get_ticks_msec()` — 用 `Time.get_ticks_msec()`
- `duplicate()` 用于嵌套资源 — 用 `duplicate_deep()`（4.5）
- `get_world()` — 2D 用 `get_world_2d()`（`get_world_3d()` 只属 3D 路径）
- `TileMap` — 用 `TileMapLayer`（仅在本项目引入网格时才相关）
- `VisibilityNotifier2D` — 用 `VisibleOnScreenNotifier2D`
- `YSort` — 用 `Node2D.y_sort_enabled`
- `Navigation2D` — 用 `NavigationServer2D`（本项目当前无寻路）
- `RichTextLabel.ImageUpdateMask.UPDATE_WIDTH_IN_PERCENT` — 4.7 改名为 `UPDATE_WIDTH_UNIT`（**GDScript 直接报错**）
- `add_image()` / `update_image()` 的 `width_in_percent` / `height_in_percent` — 用 `width_unit` / `height_unit`（`RichTextLabel.ImageUnit`）
- `DisplayServer.AccessibilityLiveMode` — 4.7 移到 `AccessibilityServer.AccessibilityLiveMode`
- `AudioEffectSpectrumAnalyzer.tap_back_pos` — 4.7 已移除
- `event.device == 0` 判鼠标/键盘 — 4.7 起设备 ID 不再是 0；按事件类型或 `InputEvent.DEVICE_ID_*` 判断
- `AnimationPlayer.method_call_mode` / `playback_active` — 用 `AnimationMixer.callback_mode_method` / `active`
- Source: `docs/engine-reference/godot/deprecated-apis.md`

### Required Patterns (engine, 4.5–4.7)

- **4.7 类型化返回**：覆写带类型返回值的方法必须显式 `return`（需要空返回时写 `return null`） — source: `current-best-practices.md` (GDScript 4.7)
- **4.7 packed array 逐元素写不再触发属性 setter**：需要 setter 时整体赋值 — source: `current-best-practices.md`
- **`@abstract`** 用于记录/服务基类（4.5） — source: `current-best-practices.md`, ADR-0001
- **4.6 双焦点**：鼠标悬停焦点与键盘/手柄焦点是两个状态，绘制分开 — source: `current-best-practices.md` (UI 4.6), ADR-0003
- **类型化容器**：`Array[T]` / 具名类型的变量，不用裸 `Array` / `Dictionary` — source: `deprecated-apis.md` (Patterns)
- **`@onready` 缓存节点**，不用 `$NodePath` 在 `_process()` 里每次查路径 — source: `deprecated-apis.md` (Patterns)
- **测试命令**：`godot --headless -s -d --remote-debug tcp://127.0.0.1:0 res://addons/gdUnit4/bin/GdUnitCmdTool.gd -a res://tests --ignoreHeadlessMode`；退出码 0 通过 / 100 失败 / 101 孤儿或警告 / 105 解析错误 — source: `current-best-practices.md`
- **解析检查**用 `godot --headless --path . -s res://.claude/scripts/godot-parse-check.gd -- res://<file>.gd`；`--check-only` 与 `--import` 都不是解析检查 — source: `current-best-practices.md`
- **检索 GDScript 文件**用 `--glob "*.gd"`：ripgrep 没有 `gdscript` 类型（`--type gdscript` 直接报错） — source: `current-best-practices.md`
- **新增 addon 走编辑器 Asset Store**（4.7 起 Asset Library 已改名为 Asset Store） — source: `deprecated-apis.md`, `current-best-practices.md`

### Forbidden Patterns (project)

- `[None configured yet — add as architectural decisions are made]` — source: technical-preferences.md（该节仍是占位；未擅自新增）

### Cross-Cutting Constraints

- **零 autoload**：任何服务都不得用 autoload 暴露全局状态 — source: ADR-0004
- **`state_changed` 单订阅者**；其余信号允许多订阅者，但连接点只在 `AppRoot._wire_signals()` — source: ADR-0004
- **`PROCESS_MODE_ALWAYS` 白名单封闭**：新增一个 `ALWAYS` 节点等于改暂停模型，必须新写 ADR — source: ADR-0004
- **存档里所有整数是十进制字符串**；解码只认 `^-?(0|[1-9][0-9]*)$`，其余一律 `CORRUPT` — source: ADR-0002
- **永不对 `FileAccess.store_*` 的返回值视而不见**（4.4 起返回 `bool`）；`rename` 之前必须 `close()` — source: ADR-0002, `breaking-changes.md`
- **唯一随机源 `SessionRng`**；游戏逻辑里不出现 `randi()` / `randf()` / `shuffle()` — source: ADR-0004, ADR-0006
- **游戏逻辑不读墙钟**（时间戳只用于展示与日志） — source: ADR-0006, ADR-0007
- **所有玩家可见字符串走 `tr()`**，无字符串拼接 — source: ADR-0008
- **不扫 `res://` 目录发现资产**：内容靠清单，音频靠 cue 目录 — source: ADR-0001, ADR-0009
- **`PROCESS_MODE_WHEN_PAUSED` 全项目为零** — source: ADR-0009
- **每个 `Resource` 的公开属性只读**（getter-only + 防御性数组拷贝），运行期不改记录 — source: ADR-0001
- **引擎未实测**：`Installed at pin time = NOT DETERMINED`，49 条 `Verification Required` 未跑。凡依赖 4.5–4.7 运行时语义的 story，先在 pin 定的 4.7.x 上实测，再写实现 — source: `VERSION.md`, ADR-0001…0009
