# Control Manifest

> ✅ **全部 20 条源 ADR 均为 `Accepted`（`accepted = 20`、`proposed = 0`，核对于 2026-11-06）。**
> 本清单的每一条规则都出自 **Accepted** ADR：ADR-0001…ADR-0020。规则内容已定，可作为已决编码规则使用。
> 本次为 `update` 调用：在保留 ADR-0001…0009 既有规则的基础上，纳入 ADR-0010…0020 贡献的 Core / Feature / Presentation 三层规则，Manifest Version 升至 2026-11-06。
> 任何 ADR 被修订后，必须重新运行 `/create-control-manifest update`；
> `/story-readiness` 的版本比对只能证明 story 写在同一天的规则上，不能证明规则此后未被修订。

> **Engine**: Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`）
> **Last Updated**: 2026-11-06
> **Manifest Version**: 2026-11-06
> **ADRs Covered**: ADR-0001 … ADR-0020（共 20 条，**全部 `Accepted`**）
> **Status**: Active — regenerate with `/create-control-manifest update` when ADRs change
> **Technical review**: `TD-MANIFEST skipped — Lean mode`（`review_gate: skill-gates=skip`；无 technical-director 门）

`Manifest Version` 是生成本清单的日期，与 `Last Updated` 恒等。Story 写入时会嵌入它，`/story-readiness` 用它找出写在旧规则上的 story。

本清单出自各 ADR 的 `Decision`、`Alternatives Considered`、`Performance Implications`、`Engine Compatibility` 四节，加上 `project.yaml` / `.claude/docs/technical-preferences.md` 与 `docs/engine-reference/godot/`。每条规则都带来源；要理由，读那条 ADR。**规则之间的冲突以 ADR 为准，不以本清单为准。**

---

## Foundation Layer Rules

*Applies to: scene management, event architecture, save/load, engine start-up*

### Required Patterns

- **零 autoload**：`AppRoot` 是 `res://src/main/main.tscn` 的根，唯一组合根；服务由构造/attach 注入，不用 `get_node()` 寻址、不用依赖袋 — source: ADR-0004
- **服务形态**：纯逻辑是 `RefCounted`（`DataConfig`、`SaveSystem`、`RunFlow`、`TimeTurns`、`StomachCapacity`、`DishValues`、`CloyingAndTaste`、`MenuWaves`、`Settlement`、`MetaUpgrades`、`Unlocks`、`Records`、`SessionRng`、`BootSequence`、`ScreenFactory`）；只有需要 `_process` / `_input` / `Tween` / `AudioStreamPlayer` / `process_mode` 的才是 `Node`（`SceneFlow`、`InputRouter`、`UIRoot`+四宿主、`HudHost`、`OnboardingHost`、`AudioDirector`+voice 池宿主、`SaveHeartbeat`） — source: ADR-0004, ADR-0010, ADR-0011, ADR-0012, ADR-0014, ADR-0015, ADR-0016, ADR-0017, ADR-0018, ADR-0019, ADR-0020
- **`SceneFlow` 是顶层状态的唯一所有者**；`state_changed(from, to, payload)` **只有一个订阅者 `AppRoot`**，按固定六步同步扇出：`set_frozen` → `ui.reset_for_state` → `hud.sync_for_state` → `audio.on_flow_state` → `onboarding.on_flow_state` → `fade.play` — source: ADR-0004
- **发射侧契约**：状态先落定再发信号；被拒绝的操作不发；同一事实每 `run_id` 至多一次；同步派发，不是队列；处理器重入深度 ≤ 1 — source: ADR-0004
- **暂停唯一机制是 `SceneTree.paused`**，由 `SceneFlow` 在切换生效时同步写入；`PROCESS_MODE_ALWAYS` 白名单封闭：`AppRoot`、`SceneFlow`、`InputRouter`+分发节点、`UIRoot`+四个宿主+`FadeLayer`、`SaveHeartbeat`、`AudioDirector`+voice 池宿主。`HudHost`、`OnboardingHost` 与全部 gameplay 保持默认 `INHERIT` — source: ADR-0004, ADR-0019, ADR-0020
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
- **启动门**：`DishValues` / `TimeTurns` / `StomachCapacity` / `Settlement` / `MetaUpgrades` / `Unlocks` / `Records` 都不参与启动门，`initialize` 在 `Menu → Run` 或 `Save.load()` 之后同步执行 — source: ADR-0010, ADR-0011, ADR-0012, ADR-0015, ADR-0016, ADR-0017, ADR-0018

---

## Core Layer Rules

*Applies to: core gameplay loop, main player systems, physics, collision*

### Required Patterns

- **`RunFlow` 是 `run_id` / 生命周期 / `RunLog` / `RunResult` 的唯一所有者**；`run_id = 0` 保留为「未分配」，永不作为有效局 — source: ADR-0007
- **结算只在一处提交**：`Run → Menu` 被接受时，在 `UIRoot.reset_for_state` 内、收据挂载之前；`run_settled` 只是**预通知**，持久化与交付发生在 `Run → Menu` — source: ADR-0007
- **事务顺序固定**：幂等闸（`run_id` + 摘要）→ `Records.prepare_run` → `MetaUpgrades.prepare_run_result` →（MVP 无 unlocks 槽位）→ merge → `Save.commit` → adopt 与发信号 → 快照 — source: ADR-0007, ADR-0017, ADR-0018
- **失败矩阵**：Records 致命 → 整笔中止、`commit` 次数为 0；`WALLET_OVERFLOW` → 省略 `progression`、仍提交 `records`；写盘失败 → `UNSAVED` + 两个命名空间都 dirty，**不回滚**；重入 → `APPLY_REENTRANT`；并发 → `ORCHESTRATOR_BUSY` — source: ADR-0007
- **`SettlementTransaction` 是纯 `RefCounted`**，在 CONSTRUCT 阶段构造并注入 `UIRoot`；串行、不嵌套、不重入 — source: ADR-0007
- **钱包只由 `MetaUpgrades` 计算**；`Settlement` 不持有 Save / Records / Meta 引用（纯函数、无 I/O） — source: ADR-0007, ADR-0016
- **唯一随机源是 `SessionRng`**（构造时 `randomize()` 一次）；所有可复现随机走种子派生：`FNV-1a` 双道 over `UTF8(station_id) ‖ 0x00 ‖ LE64(seed) ‖ LE16(attempt)` — source: ADR-0004, ADR-0006
- **洗牌 = 规范序 + 反向 Fisher–Yates + 拒绝采样**；划分与可见度在 `initialize` 一次性冻结 — source: ADR-0006
- **健康门 = 逐波重置 DP**：状态 `(i, c, t)`、五条转移、`NODE_BUDGET = 10000`、`STATE_CELL_BUDGET = 262144`，超预算 `TRUNCATED → DEGRADED`；整数阈值 `reachable * 10 < baseline * 9`；`MAX_ATTEMPTS = 21`，并列取更早，`DEGRADED` 不再重掷，恰好一条 warning — source: ADR-0006
- **逻辑不读墙钟、不用浮点**：全部整数运算（int64 语义），确定性纯核心 — source: ADR-0006
- **`opt_value` 是内部基准**：只用于健康门与 CI/设计，不暴露公开 API、不进存档 / HUD / 结算；审计方法与运行期 API 表面隔离 — source: ADR-0006, ADR-0010
- **`DishValues` 是无状态纯服务**：`RefCounted`，构造只注入 `DataConfig`；不缓存 `DishData`、不发信号、不注册 `_process`、不读墙钟、不写存档；`DataConfig` 未 `Ready` 时所有公开查询返回 `DATA_NOT_READY`，不构造部分结果 — source: ADR-0010
- **单位价值只有一个实现**：`uv_cmp = sign(price_a × satiety_b − price_b × satiety_a)`，乘积在 int64 中完成；`unit_value` 只在「单位价值查询」与「展示」路径计算，不写回记录；`satiety = 0` 返回 `INVALID_SATIETY`，绝不执行除法 — source: ADR-0010
- **失败一律是结构化 Result**（`ok` / `value` / `error`，错误码封闭为 `DATA_NOT_READY` / `UNKNOWN_DISH` / `INVALID_SATIETY`）：`value` 保持未设置，禁止用 `0` / `0.0` / `null` / 空记录 / 占位菜表示失败 — source: ADR-0010
- **排序与支配**：唯一键是 `uv_cmp` 降序 + `id` 字典序升序；`same_group` 把两个空 `cloying_group` 视为同组；`menu_healthy` 只接档口菜池，不接单次刷新菜单 — source: ADR-0010
- **运行时 / 审计隔离**：`opt_value`、`calibration_ratio`、`menu_healthy`、`uv_span`、`*_dominated_pairs` 只允许 `tests/`、`tools/` 与 CI 调用；CI 用静态断言扫描运行期 API 表面（无求和、无净利、无最优解入口）；审计失败只写报告，不改变 `DataConfig`、不阻断 `Boot` — source: ADR-0010
- **`TimeTurns` 只持有 `state` / `turns_total` / `turns_remaining` / `turns_used`**：无构造参数、不持有上游引用、不发信号、不注册 `_process`、不读墙钟、不写存档，也不提供「加回合 / 退回合 / 设置剩余」 — source: ADR-0011
- **回合初始化一次、冻结、不兜底**：`initialize(turn_budget)` 完整替换上一局；`turn_budget < 1` → `INVALID_TURN_BUDGET` 且三读数不变；不存在「缺省 10」或「钳到 1」；开局后 `turns_total` 永久冻结，改 `StationData.turn_budget` 只影响下一局 — source: ADR-0011
- **扣回合恒为 1、无可选代价**：`can_spend_turn()` 是纯读取（可每帧调用）；`spend_turn()` 不做去重，同一动作调两次就扣两格并必须暴露成测试失败；`turns_used = turns_total − turns_remaining` 同帧重算 — source: ADR-0011
- **耗尽只报告、不裁决**：`EXHAUSTED` 是可观测终态；`TimeTurns` 不保存 `end_reason`、不排序 `TIME_EXHAUSTED` 与 `NO_EATABLE_DISH`、不调用任何结束流程、不发 `turn_*` 信号 — source: ADR-0011
- **只有 `RunFlow` 扣回合**：开局 `initialize(turn_budget)` 一次；每个成功提交的 `eat` 与非最后一波 `skip_wave` 各 `spend_turn()` 一次；`MenuWaves`、局内 HUD、音效反馈都不调用 — source: ADR-0011, ADR-0013
- **`StomachCapacity` 是唯一容量账本**：`RefCounted`，构造只注入 `base_capacity`（`C0`），不持 `DataConfig` / `RunFlow` / `CloyingAndTaste` / `Save`；`effective_satiety` 由 `RunFlow` 以整数参数传入 — source: ADR-0012
- **容量初始化是严格快照**：`initialize(snapshot)` 要求 `capacity_bonus` 为 `int ≥ 0`（缺失 / `null` / 负数 / `1.5` / 字符串都是失败，不取绝对值、不四舍五入、不默认 0）；`capacity_total = C0 + bonus ≤ 999`；同局第二次 `initialize` → `CAPACITY_INIT_INVALID`，由 `RunFlow` 进入 `Faulted` — source: ADR-0012
- **`end_run(reason)` 是唯一清零点**：`RunFlow` 在 `ENDED` / `DISCARDED` / `FAILED` / `FAULTED` 每条终止路径各调一次；幂等；不订阅 `run_ended` / `run_discarded` / `run_faulted` — source: ADR-0012
- **可吃性等号允许，写入唯一且原子**：`can_fit(e)` 允许 `used + e == total`，形参用 `Variant` 以便把 `2.5` 判成 `false` 并记 ERROR；`consume(e)` 越界返回 `CAPACITY_OVERFLOW`（不钳到上限）；容量满只是派生读数，不产生状态、不结束本局、不发信号 — source: ADR-0012
- **`submit_action(action, dish_id)` 是唯一动作入口**：`RunFlow` 不读输入；一次输入事件最多调用一次，同帧第二个动作由 `InputRouter` 的「首个消费者获胜」拦下；`RunFlow` 不读帧计数器、不维护「本帧已提交」标志 — source: ADR-0013, ADR-0005
- **公共前置顺序**：`frozen == true` → `Rejected(FROZEN)`；`run_state != ACTIVE` → `Rejected(NOT_ACTIVE)`；两项都在动作语义校验之前且零副作用 — source: ADR-0013
- **`eat` 先五步校验、再九步原子提交**：校验顺序固定为「可见集 → 腻度 `extra/effective` → `price` → `can_fit` → `can_spend_turn`」；提交顺序固定为「追加 `EatenEntry` → `cloying.record_eat` → `menu_waves.consume` → `turns.spend_turn` → `capacity.consume` → `running_net += price` → `action_seq += 1` → `action_committed` → 结束判定」，任一「不应失败」的错误直接 `Faulted`，不部分回滚 — source: ADR-0013
- **`skip_wave` 先问波次、再问回合**：最后一波等价 `end_pickup`（0 回合、不调 `spend_turn()`）；非最后一波用 `turns.get_turns_remaining() ≥ 2` 而不是 `can_spend_turn()`；拒绝发生在 `advance()` 与 `spend_turn()` 之前（波次不推进、菜不丢弃、回合不变、不写日志） — source: ADR-0013
- **结束判定只有一个函数、两个时机**：`_judge_end()` 在 `run_started` 后立即一次、每次成功提交后一次；`TIME_EXHAUSTED` 优先于 `NO_EATABLE_DISH`，顺序写死在函数里；被拒绝的动作与暂停帧不判定；`Ending` 后 `submit_action` 一律 `NOT_ACTIVE` — source: ADR-0013
- **`no_eatable` 单调**：只由 `menu_waves.consume == NO_EATABLE`、`end_pickup() == ENDED`、最后一波被吃空三处置真，一旦为真不回落；可见菜因容量放不下**不**设置它 — source: ADR-0013
- **`action_availability()` 是只读投影**：HUD 只格式化 `skip_reason` / `end_reason` 与代价文案，不自行比较 `turns_remaining ≥ 2` 或 `is_final_wave`；投影不提交、不推进、不发信号 — source: ADR-0013, ADR-0019

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
- **Never 把 `unit_value` 存进 `DishData`** — 记录变成派生数据的副本，改价格/饱腹就漂移，浮点序列化不稳 — source: ADR-0010
- **Never 让下游各自算 `price / satiety`** — 两份公式必然漂移，浮点并列判定不一致（GDD 明确禁止） — source: ADR-0010
- **Never 用浮点直接比较单位价值**（`a.price / a.satiety > b.price / b.satiety`） — 末位误差会把 `15/3` 与 `20/4` 判成不等，无法处理精确并列 — source: ADR-0010
- **Never 把 `menu_healthy` 放进启动门并阻断开局** — 平衡失败不阻断启动，审查归 CI — source: ADR-0010
- **Never 在运行期引用审计方法或暴露求和 / 净利 / 最优解入口** — 泄露答案，且运行期不依赖审计代码 — source: ADR-0010
- **Never 按真实时间扣格**（`_process` / 计时器） — 暂停、失焦与低帧率会改变玩法，结算不可复现 — source: ADR-0011
- **Never 让 `TimeTurns` 自己读 `StationData.turn_budget`** — 新增一条数据层依赖边，热重载与调试语义要重定义 — source: ADR-0011
- **Never 静默兜底非法输入或钳制损坏状态** — 缺配字段会伪装成平衡问题；钳制让越界变成一张看起来正常的收据 — source: ADR-0011
- **Never 给 `spend_turn(cost)` 加代价参数**，也 **Never 让 `TimeTurns` 保存 `end_reason`** — 恒等式失去结构性保证，结束原因出现第二个写者 — source: ADR-0011
- **Never 用「试扣 + 回滚」代替 `can_spend_turn()`** — 判断变成有副作用，调用方容易在失败后继续执行 — source: ADR-0011
- **Never 让 HUD / 结算各自维护剩余容量或第二份计数器** — 三个不一致的剩余格数，重复提交无法被发现 — source: ADR-0012
- **Never 把非法快照默认成 0 加成** — 漏配升级会伪装成「玩家没买升级」 — source: ADR-0012
- **Never 把容量满当结束原因（`STUFFED` / `full` 标志）** — 违反「容量只决定一道菜能不能吃」，玩家失去剩余回合与跳波权利 — source: ADR-0012
- **Never 提供 `refund` / `release` / `reset` 容量公共方法** — 破坏单调性与结算不变式；未来加入消化必须重写 ADR — source: ADR-0012
- **Never 边校验边提交** — 失败会留下半提交（先扣回合再查容量） — source: ADR-0013
- **Never 用 `can_spend_turn()` 直接门控付费跳波** — 剩 1 回合跳波等于零收益丢掉当前波 — source: ADR-0013
- **Never 让每个动作各自判定结束，或在 `_process` 里每帧轮询结束** — 三份优先级必然漂移，且帧率影响行为 — source: ADR-0013
- **Never 在 `Ending` 撤销最后一次动作** — 结束即冻结，撤销会让 `turns_used` / `capacity_used` 与动作历史不符 — source: ADR-0013
- **Never 在 `Ending` 之后再调 `spend_turn()` / `consume()` / `advance()` / `end_pickup()`** — 剩余工作只有组装 `RunLog` 与交接 — source: ADR-0013

### Performance Guardrails

- **`SettlementTransaction`**：整笔 p95 ≤ 60 ms（一次状态切换内） — source: ADR-0007
- **`Settlement`**：p95 ≤ 1 ms/局，硬上限 5 ms；**`Records.prepare_run`** p99 ≤ 5 ms — source: ADR-0007
- **`MenuWaves`**：单候选 < 0.3 ms；20 次重掷 < 4 ms；`initialize` p95 ≤ 4 ms — source: ADR-0006
- **`DishValues`**：`unit_value` / `uv_cmp` 单次 < 1 µs；完整审计（132 个有向对 + 0/1 背包）≤ 50 ms，其中支配扫描 ≤ 1 ms、`opt_value` ≤ 5 ms；无缓存、常驻开销 0 — source: ADR-0010
- **`TimeTurns`**：查询与扣减 O(1)、单次 < 1 µs、无循环无分配；`initialize` 纳秒级 — source: ADR-0011
- **`StomachCapacity`**：`can_fit` / `consume` O(1)、单次 < 1 µs；三个字段 + 一个只读 `C0` — source: ADR-0012
- **动作管线**：一次动作 O(1)、无循环无分配，`eat` 的九步提交远低于 1 ms；`eaten` 最多 `turns_total` 条（MVP 10） — source: ADR-0013

---

## Feature Layer Rules

*Applies to: secondary mechanics, AI systems, secondary features*

> 本层已由 ADR-0014…ADR-0018 补齐：腻度与口味、结算纯函数、局外升级/钱包、解锁/档口购买、纪录/里程碑/生涯。规则不再只靠 ADR-0001 / ADR-0002 / ADR-0007 的横切约束。

### Required Patterns

- **每条记录必须登记在对应清单**；CI 反向断言「`res://data/<type>/*.tres` 的每个文件都被清单引用」 — source: ADR-0001
- **`MilestoneCatalog` 由 `Records` 拥有**，在 `Data Ready` 与 `Save.load()` 之间加载，并入同一道启动门 — source: ADR-0001
- **命名空间写入只经 `Save.commit`**：`progression`（钱包/升级）、`records`、`unlocks` 各自的所有者写自己的键 — source: ADR-0002
- **错误词表封闭**：`COMMIT_FAILED` / `SAVE_DEFERRED` / `APPLIED_UNSAVED` / `ALREADY_SETTLED` / `ALREADY_APPLIED` / `ORCHESTRATOR_BUSY` / `APPLY_REENTRANT`；不用 `null` / `0` 占位失败 — source: ADR-0002, ADR-0007
- **未知 id 在加载与迁移中按 id 保留**，不删除、不解释 — source: ADR-0002
- **`CloyingAndTaste` 是 `RefCounted`**，构造注入 `(DishValues, cloy_cap)`；只持有 `initialized` / `resistance_floor` / `counts` / `cloy_cap`；不持 `RunFlow` / `StomachCapacity` / `HudHost` / `Save`，不发信号、不读墙钟、不写存档 — source: ADR-0014
- **`preview` 是纯读取**：`extra = max(0, min(counts[g], cloy_cap) − resistance_floor)`、`effective = satiety + extra`，用**吃之前**的计数；连续调用 1,000 次结果相同；状态推进只在 `record_eat` — source: ADR-0014
- **`record_eat(dish, expected_extra)` 三步固定顺序**：校验（`preview().extra_satiety == expected_extra`，否则 `STALE_PREVIEW` 且不改状态）→ 记录（`counts[g] += 1`）→ 解腻（只减一个组）；解腻不退还已占容量、不改本菜 `extra`、不影响已写入的 `EatenEntry` — source: ADR-0014
- **规范组序是显式常量** `CANONICAL_GROUPS`，并列取索引更小者；数组外的组追加在末尾并按 `StringName` 字节序升序，`initialize` 时冻结；禁止依赖 `Dictionary` 遍历顺序 — source: ADR-0014
- **抗性是定点减免**：`resistance_floor = int(floor(resistance))`，`initialize` 时冻结；`resistance < 0` 或非有限值 → `INVALID_RESISTANCE`，不默认 0、不取绝对值 — source: ADR-0014
- **`Settlement` 是纯函数**：`RefCounted`，构造只注入 `DishValues`；不持 `RunFlow` / `StomachCapacity` / `CloyingAndTaste` / `MenuWaves` / `MetaUpgrades` / `Records` / `SaveSystem`；日志之外没有第二个输入 — source: ADR-0015
- **`settle(log)` 十二步固定顺序**：去重（`ALREADY_SETTLED` 必须排在数据检查之前）→ 数据就绪 → 入场费 → 回合 → 解析菜品 → 逐行校验 → 容量求和 → 总价值 → 净利 → 派生量 → 组装行 → 最后才写 `last_settled_run_id`；任一步失败不产生部分结果 — source: ADR-0015
- **金额一律受检 int64**：`_checked_add` 前置比较，溢出返回 `OVERFLOW` 且不输出部分金额、不回绕成负数；`net_profit` 不钳制（`−85` 是合法结果）；展示层单独格式化 — source: ADR-0015
- **零入场费不除法**：`fee = 0` 时 `return_ratio = null`、`return_percent_x10 = null`，收据显示「免单」，绝不出现 `∞` / `NaN` / `0` 或任何回本率数值 — source: ADR-0015
- **行序 = 吃下顺序**，不提供排序入口；重复菜出现两次就两行、两次 `price`；`unit_value` 取自 `DishValues.unit_value`，不重算 `price / satiety`；`effective_unit_value` 只用于展示 — source: ADR-0015
- **`broke_even` 是派生谓词**：`net_profit >= 0`，任何地方都不得缓存布尔副本；`RunResult` 不含菜名、不含 `opt_value`、不含任何「上限 / 最优 / 差额」字段 — source: ADR-0015
- **单一求和来源**：`total_value` / `running_net(k)` / `shortfall(k)` 共用同一个前缀求和 `_sum_prices`；运行期累加器仍归 `RunFlow`（开局 `−entry_fee`，每次成功提交 `+price`），HUD 不自建累加器 — source: ADR-0015, ADR-0019
- **`MetaUpgrades` 是钱包唯一写者**：三条流水（`prepare_run_result` / `purchase` / `spend`）；写入一律是完整新值 `{wallet, upgrade_levels}`，从不提交增量；负钱包合法，无利息 / 救济 / 上下限钳制 — source: ADR-0016
- **结算入账用受检加法**：`WALLET_OVERFLOW` 时钱包保持原值、不构造 `progression_update`、不发 `progression_changed`；`run_id` 去重返回 `ALREADY_APPLIED`；**不调用 `save.commit`**（合并提交归 ADR-0007 的事务对象） — source: ADR-0016
- **价格只在最后取整一次**：`price(u, L) = ceil(cost × cost_growth^(L−1))`，`L ∈ [1, max_level]`；整值吸附阈值 `1e-9`（避免 `16 × 1.5 = 24.000000000000004 → 25`）；`max_level` 是硬上限，不提供溢出等级 — source: ADR-0016
- **升级快照每次新建**：`resolve_upgrade_snapshot()` 返回新字典 + 深拷贝的 `levels`；同类型多条记录效果相加、顺序无关；未知 id 贡献 0 但保留在副本中；下游不得拿 `levels` 自行重算效果 — source: ADR-0016
- **`MENU_VISION` 不卖废等级**：`menu_vision_useful_max = ceil((wave_size − 1) / effect_value)`（整数实现）；`current_level + 1 > useful_max` → `NO_EFFECT`；解析值不按档口截断 — source: ADR-0016
- **购买事务固定九步校验后「快照 → 内存 → commit → 成功或回滚」**：`commit` 失败回滚 `{wallet, levels}` 并发 `purchase_failed(COMMIT_FAILED)`，不发 `progression_changed`；成交只在 `commit` 成功后才确认 — source: ADR-0016
- **`spend(amount, reason, extra)` 的 `extra` 是其他命名空间的完整新值**（不是增量），原样并入同一次 `commit`；含 `progression` 一律 `RESERVED_NAMESPACE` — source: ADR-0016
- **`Unlocks` 的账本单调**：`normalize()` 只增不减（`stations ⊇ source.stations`），未知 id 原样保留；结果只依赖数据与输入，不读时钟、不用随机、不做 I/O — source: ADR-0017
- **可用性由数据推导，不靠集合成员**：免费档口（`unlock_cost == 0`）始终可用；`supports_station_selection` 每次查询都调用，开关从 `false → true` 时被门禁的档口立即恢复判定，无需重新载入 — source: ADR-0017
- **`can_unlock` / `unlock` 共用十步固定校验**：`DATA_NOT_READY` → `NOT_WRITABLE` → `NOT_IN_MENU` → `UNKNOWN_STATION` → `INVALID_STATION_DATA` → `FEATURE_GATED` → `BUSY` → `ALREADY_UNLOCKED` → `PREREQUISITE_LOCKED` → `INSUFFICIENT_FUNDS`；`INVALID_STATION_DATA` 必须排在门禁与余额之前 — source: ADR-0017
- **解锁只经 `MetaUpgrades.spend`**：先算 `new_ledger`，再把 `{unlocks: new_ledger}` 交给 `spend`，与 `{progression}` 放进**同一次** `commit`；本服务不调用 `commit`；失败时账本与钱包一起保持解锁前状态 — source: ADR-0017
- **`Records.prepare_run(result)` 是唯一纪录写入点**：在副本上算完再整体采纳；断言失败返回 `INCONSISTENT_RESULT` 且 `_records` 逐字段不变；溢出返回 `OVERFLOW` 且不改 `_last_applied_run_id` — source: ADR-0018
- **`records` 命名空间无浮点**：不持久化 `return_ratio`（只存 `return_percent_x10`，快照中派生 `percent_x10 / 1000.0`）；`upgrade_snapshot` 的 `float` 递归编码为规范十进制字符串（最多 6 位小数、去尾零、`-0` 写 `0`），只读快照只把已知键 `cloying_resistance` 解码回 float — source: ADR-0018
- **保存前递归断言 `_is_save_safe()`**：只允许 `null` / `bool` / `int` / `String` / `Array` / `Dictionary`（键为 `String`）；出现 float / `StringName` / `Object` / `Vector*` 即 `INCONSISTENT_RESULT` — source: ADR-0018, ADR-0002
- **档口纪录严格更大才替换**：`(old_best == ⊥) ∨ (new_net > old_best.net_profit)`；平局整条保留（含 `achieved_at` 与快照）；第一次结算无论正负都建立纪录 — source: ADR-0018
- **生涯累加是事实**：`runs_settled` / `lifetime_net_profit` / `lifetime_value` 全走受检加法，任一步溢出整次更新不生效（档口纪录也不替换）；空局也 `runs_settled +1`、`lifetime_value +0` — source: ADR-0018
- **里程碑不追溯**：只从下一次结算起可达成，不回扫旧纪录；已达成条目永不删除、永不撤销（`threshold_at_earn` 冻结）；`career` 类指标的 `station_id` 必须为空 — source: ADR-0018
- **通关条件显式判真**：`all_required_earned = (|required| > 0) ∧ (∀m : m.id ∈ E)`，空必需集不得自动通关；`completed_at` 一旦非空永不改写、永不回退 — source: ADR-0018
- **时间戳只经注入的 `utc_now`**：一次 `prepare_run` 采一次、三个字段共用；`Records` 源码不引用 `Time` / `OS`；任何逻辑都不读时间戳（比较、排序、判定、迁移都不读）；缺失写 `null`，`completed_at` 损坏写 `"unknown"` 哨兵 — source: ADR-0018
- **只在该局 `APPLIED` 时发信号，顺序固定**：`record_broken` → `milestone_earned`（按 id 升序，逐条发）→ `career_completed`；信号在 `Save.commit` **之前**发出，写盘成功不是前提 — source: ADR-0018, ADR-0004, ADR-0007
- **`snapshot()` 是视图不是第二份状态**：深拷贝、按 id 升序、已删除定义的里程碑以 `archived = true` 保留；未达成的条目必须出现 — source: ADR-0018

### Forbidden Approaches

- **Never 直接改别人命名空间的键** — 只经 `Save.commit`，否则原子性与回滚失效 — source: ADR-0002
- **Never 把 `unlocks` 键放进 MVP/VS 的结算事务** — 付费解锁是独立购买事务（未来奖励解锁才并入同一 `commit`） — source: ADR-0007, ADR-0017
- **Never 用「写盘成功」作为纪录/里程碑信号的前提** — 信号在事务已应用后发，写盘失败只影响快照标记 — source: ADR-0004, ADR-0007
- **Never 让 `preview` / `effective_satiety` 推进计数** — HUD 预选会污染计数，同一道菜会被算多次 — source: ADR-0014
- **Never 先 `+1` 再算额外饱腹** — 一道菜用自己的计数抬高自己，违反「先付后记」 — source: ADR-0014
- **Never 让解腻在多个并列组上同时生效** — 只减一个，并列按规范序，否则强度随菜单规模漂移 — source: ADR-0014
- **Never 用 `Dictionary` 遍历找解腻目标** — 遍历顺序不稳定，并列结果跨平台不一致 — source: ADR-0014
- **Never 用百分比抗性**（`extra *= (1 − resistance)`） — 小整数上不可预测，会引入浮点判定 — source: ADR-0014
- **Never 把腻度计数表持久化进存档（MVP）** — 无中途存档；启用时须由存档 ADR 重开并定义迁移 — source: ADR-0014
- **Never 让结算按菜名重放腻度序列** — 复制一套规则，抗性/`cloy_cap`/规范序一改历史局就重算 — source: ADR-0015
- **Never 用 `float` 求和金额** — 合计与净利在边界值上不可回放 — source: ADR-0015
- **Never 把负净利钳制到 0** — 隐藏失败、破坏支柱 1 的算术题 — source: ADR-0015
- **Never 把 `broke_even` 存进 `RunResult`** — 与 `net_profit` 形成两份事实 — source: ADR-0015
- **Never 让 `settle` 顺手写钱包与纪录** — 形成环、结算不再纯、失败无法分类 — source: ADR-0015
- **Never 零入场费显示 `∞` 回本率** — 无定义值进入 UI — source: ADR-0015
- **Never 让收据按价格或单位价值排序** — 顺序本身是复盘信息 — source: ADR-0015
- **Never 提交钱包增量** — 存档要完整新值，增量无法判定基线、重放会加两次 — source: ADR-0016
- **Never 在结算写盘失败时回滚内存** — 玩家已打完一局却看不到收益（购买才回滚） — source: ADR-0016
- **Never 逐级 `ceil` 价格** — 取整误差逐级累积，同一记录在不同实现下价格不同 — source: ADR-0016
- **Never 把已解析效果写进存档** — 等级与效果两份事实，改 `effect_value` 后旧档不更新 — source: ADR-0016
- **Never 在结算事务里调用 `purchase`** — 事务不得嵌套 `commit`，且两者失败策略相反 — source: ADR-0016
- **Never 提供 `set_wallet` / `add_wallet` / `set_level` / `refund` / `grant` / `reset_wallet`** — 打开第四种流水，任何系统都能绕过事务 — source: ADR-0016
- **Never 用「id 在持久化集合中」判定可用性** — 空存档与 `Blocked` 存档将没有可玩档口，且需要写盘才能开始 — source: ADR-0017
- **Never 把菜作为第二道闸门** — 漏写会制造「档口开了、菜不能吃」的不可玩状态 — source: ADR-0017
- **Never 让解锁自己 `commit`，或先扣款再规范化账本** — 两次写盘会出现半成交 — source: ADR-0017
- **Never 在载入时清理未知 id** — 数据回滚或改名会丢失玩家已买的解锁 — source: ADR-0017
- **Never 缓存载入时的门禁可用性** — 开关变化后已购档口不会立即恢复 — source: ADR-0017
- **Never 让解锁排队等待** — 单笔在途，第二笔返回 `BUSY`（不排队、不合并） — source: ADR-0017
- **Never 持久化 `return_ratio`（float）** — 违反 ADR-0002 的「文件里没有 float」，两个精度还可能不一致 — source: ADR-0018
- **Never 把 `upgrade_snapshot` 原样落盘** — 写入层必须拒绝整个事务或临时转字符串 — source: ADR-0018
- **Never 用全局最佳代替每档口最佳，或按回本率排名** — 入场费不同，绝对净利不可比；钱包来源是净利 — source: ADR-0018
- **Never 平局用时间或总值二次比较** — 读时钟会受玩家机器时间回拨影响，历史不再稳定 — source: ADR-0018
- **Never 在 `Records` 里重算结算** — 第二套算术，只做一致性断言 — source: ADR-0018
- **Never 新增里程碑时回溯授予** — 单局阈值无法从聚合数据忠实重建，补时间戳等于伪造 — source: ADR-0018
- **Never 写盘成功后才发纪录信号，也 Never 写盘失败回滚纪录** — 收据会缺印章；或抹掉已发生的结算事实 — source: ADR-0018

### Performance Guardrails

- **`CloyingAndTaste`**：`preview` / `record_eat` 为 O(G)（G ≤ 9），单次 < 5 µs；计数表最多 9 个键 — source: ADR-0014
- **`Settlement`**：`settle` 为 O(n)（MVP ≤ 10 行、满级 ≤ 25），典型 < 0.2 ms，硬上限 5 ms；`RunResult` 25 行 < 16 KiB — source: ADR-0015
- **`MetaUpgrades`**：`resolve_upgrade_snapshot` / `get_catalog` 为 O(U ≤ 12)，单次 < 1 ms；`purchase` 的逻辑 < 0.1 ms（主要成本是同步磁盘提交）；快照 < 1 KiB — source: ADR-0016
- **`Unlocks`**：`normalize` / 目录 / 进度为 O(S + D)，单次 < 0.1 ms；账本 MVP < 2 KiB，查询副本最多 24 项 — source: ADR-0017
- **`Records`**：`prepare_run` 为 O(S + M log M + M)（S ≤ 4、M ≤ 24），单次 < 1 ms；每次深拷贝一份，峰值 < 32 KiB — source: ADR-0018

---

## Presentation Layer Rules

*Applies to: rendering, audio, UI, VFX, shaders, animations*

### Required Patterns

- **宿主分层固定**：`HudHost` 10 / **引导 `OnboardingHost` 15** / `ScreenHost` 20 / `OverlayHost` 25 / `ModalHost` 30 / `NoticeHost` 40 / `FadeLayer` 50；`FadeLayer` 与 `NoticeLayer` 的全屏根 `MOUSE_FILTER_IGNORE` — source: ADR-0003, ADR-0020
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
- **HUD 组成三层**：`HudHost`（适配器：生命周期、信号、输入轮询、锚点，不绘制任何东西）+ `HudSource`（唯一读上游者）+ `HudView`（只读 `HudFrame` 与焦点解析结果）；`HudHost` 是唯一允许调用 `RunFlow.submit_action` 的 HUD 类 — source: ADR-0019, ADR-0005
- **`HudFrame` 一次事件一份快照、构造后不可变**：字段全在 `_init` 赋值，数组 `duplicate(true)`，没有 setter / `apply()` / `refresh()`；`cards[i].display_index == i` 严格等于 `visible_dishes()` 顺序 — source: ADR-0019
- **`HudSource.capture()` 一次采集**：一次同步调用内取一份 `RunFlow.snapshot()`、一份可见菜列表，每项只读一次；不写任何上游状态；`HudInputs` 携带 `run_id` 与 `action_seq` 保证「一帧一快照」 — source: ADR-0019
- **卡片字段同屏**（名称/价格/基础饱腹/额外饱腹/实际饱腹/`can_fit`/原因），不折叠进悬浮提示；不可吃卡只置灰写原因，仍可获焦可读全字段 — source: ADR-0019
- **HUD 按窗口像素布局、一次逆缩放**：读根视口内容缩放 `c`，`CanvasLayer.transform` 设为 `Transform2D(0, Vector2(1.0/c, 1.0/c), 0, Vector2.ZERO)`，`HudView.size = viewport_size`；公式里的 `s = clamp(min(W/1280, H/720), 1.0, 2.0)` 不再乘 `c`；全部布局常量在 `HudTheme`（`res://data/config/hud_theme.tres`），`HudLayout.compute` 是纯函数，所有矩形 `round` 到整数像素 — source: ADR-0019
- **`protected_rects` 恒为三个、顺序固定**（回合 / 容量 / 净利），任何布局模式下非空且互不重叠；`Active` 下 `area_usage ≤ 0.28`，中央 40% 只允许 `card_rects` 与 `confirm_panel` 进入 — source: ADR-0019
- **确认面板是 HUD 内的焦点陷阱**：只经 `HudFocusSpec` 限制 `order` 并闭环方向，**不是**输入上下文意义上的 `Modal`；打开期间吞掉 `game_skip_wave`；默认焦点「取消」；`run_ended` / `run_faulted` / `run_discarded` / 任何 `action_committed` 都关闭面板 — source: ADR-0019
- **HUD 状态是 `run_state` + `frozen` 的投影**，不新增玩法状态；`sync_for_state` 是唯一挂载/卸载入口；`Paused` 不卸载、只重建 `FROZEN` 帧；`Settling` 不清屏；淡出绝不延迟卸载 — source: ADR-0019, ADR-0004
- **数字先变、动画后跟**：`_rebuild` 先写新值再起 `Tween`，`Tween` 不推迟数值、不参与状态判定；`reduce_motion` 时 `duration_ms = 0` — source: ADR-0019
- **文本与格式集中**：全部玩家可见字符串来自 `HudTextTable`，HUD 代码不写中文字面量；`HudFormat` 是纯静态函数；金额负号用 U+2212、空值用 `—`、超 5 位照常完整显示不截断 — source: ADR-0019, ADR-0008
- **颜色必须有第二通道**：净利正/负带 `+` / `−` 前缀与标签，回本差额给整句，不可吃给原因句；`HudContrast.ratio` 对每组文本/背景（含 60% 不透明度的有效底色）断言 ≥ 4.5 — source: ADR-0019
- **五条运行时不变式**（容量相加、回合恒等式、`shortfall = max(0, −running_net)`、`can_fit ⇒ 容量够`、同帧同 `run_id`/`action_seq`）；失败不采纳新帧、保留 `_last_legal_frame`、`interactive = false`，正式构建不崩游戏 — source: ADR-0019
- **首局规划归 Application、引导执行归 Presentation**：`FirstRunPlanner`（`RefCounted`）是 `is_first_run` / `first_run_eligible` / 菜单入场费 / `RunConfig` 预置的**唯一**生产者；`OnboardingHost` 只消费 `plan`、`RunFlow` 快照、HUD 锚点/焦点、`MenuWaves` 与腻度组计数 — source: ADR-0020
- **`OnboardingHost` 不是 `UIRoot` 的屏幕、也不是 `Modal`**：不进入 `ScreenHost`、不参与 `replace_root`，`Run` 的 `ui_stack_depth = 0` 保持成立；唯一下下文写是 `push_context(Blocked)` / `pop_context()`；`TutorialLayer` 与 `HudHost` 用同一套逆缩放 — source: ADR-0020, ADR-0019
- **首局判定三态 + `Recovered` 例外**：`runs_settled == 0 ∧ first_run_eligible` 才 `ELIGIBLE`；`Recovered` / 记录不可用 / 预置非法一律不教；`OnboardingHost` 不读 `SaveSystem` / `LoadReport` / `RecordsSnapshot` — source: ADR-0020
- **预置内容六道菜 / 40 元 + 七条不变式**：由 `FirstRunPreset.validate` 校验，第 4 条用 `make_cloying()` 新建临时 `CloyingAndTaste` 重放 `preview` / `record_eat`，**不得另写一份腻度公式**；失败关闭引导但这一局照常开始，CI 必须红 — source: ADR-0020
- **`FirstRunRunPlan` 是唯一来源**：主菜单入场费读 `plan.menu_entry_fee`，`RunConfig` 读 `run_config_fields()`；`entry_fee_override` 不被支持时菜单与本局同时降级，不允许一边 40 一边 85 — source: ADR-0020
- **Beat 机固定序列、一次输入一 beat、无计时器**：`IDLE → INTRO → NUMBERS → COACHING → CLOSING → CLOSED/CANCELLED`；`ACCEPT` / `CANCEL` 只在 `input_seq` 大于已消费值时生效；`required = {B0, B1, B2, B4} ∪ ({B3} if triggered)` — source: ADR-0020
- **输入白名单封闭**：`B0` 只允许 `ui_accept` / `ui_cancel`，`B1` 再加四个导航，`ui_pause` 恒允许；`COACHING` / `CLOSING` 不压栈、不 `consume_action`；每次轮询同时取走 `ui_cancel` 与 `ui_accept`，cancel 优先 — source: ADR-0020, ADR-0005
- **`process_priority` 是契约常量**：`SceneFlow` −100 < `OnboardingHost` −50 < `HudHost` 0，启动时断言其顺序 — source: ADR-0020
- **暂停恢复用同一白名单重压 `Blocked`**；重压失败立即降级为无遮罩停靠提示，不重试、不等待 — source: ADR-0020
- **气泡几何是纯函数**：`BubbleLayout` 相交时依次「翻面 → 停靠 → 隐藏」；B2/B3 一律 `DOCKED`；`ScrimPieces` 是三个受保护矩形的补集（从 `[viewport]` 依次做矩形差，输出顺序左/上/右/下），受保护区域内不绘制任何颜色 — source: ADR-0020
- **引导文本走 `OnboardingTextTable`**：占位符是封闭集合，模板不得出现数字字面量，单条 ≤ 48 字（150% 字号最坏情况），超限用 `short` 版本，仍超限则生成期 lint 失败 — source: ADR-0020
- **`FirstRunExplanation` 在 `Run → Menu` 第 2 步（结算事务）之后构建**，不在 `run_settled`；`Run → Menu` 被拒绝时保持 `CLOSING` 等 `Handoff` 重发；`explanation_ready` 每 `run_id` 至多一次、`onboarding_finished` 每局至多一次 — source: ADR-0020
- **十二级降级阶梯**：任何一级失败都不阻止开局、不改变 `run_state`、不扣回合/容量、不写存档、不让玩家「按键无效」 — source: ADR-0020
- **可测性**：HUD 的 `HudSource` / `HudFrame*` / `HudLayout*` / `HudFormat` / `HudTextTable` / `HudFocus*` / `HudInvariants` 与引导的 `FirstRun*` / `BeatMachine` / `BubbleLayout` / `ScrimPieces` / `ExplanationBuilder` 都不引用 `Node` / `SceneTree`，可在无头测试实例化；`HudViewFactory` / `TutorialViewFactory` 允许注入 stub — source: ADR-0019, ADR-0020

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
- **Never 让每个 HUD 元素各自读上游** — 一次事件内多次读取会跨快照混帧 — source: ADR-0019
- **Never 让 HUD 自己维护净利累加器** — 免单 / 错误局 / 结算复核三条边界必然漂移，出现第三份真相 — source: ADR-0019
- **Never 用 `push_context(Modal)` 实现确认面板** — 会改变 `InputRouter` 的上下文并让两个栈的深度断言打架 — source: ADR-0019
- **Never 把「设计单位布局 + 引擎缩放」当默认路径** — 只作逆缩放不可行时的降级，否则 2160p 违反 `hud_ui_scale` 2.0 上限 — source: ADR-0019
- **Never 让 HUD 自绘焦点、自己处理方向键** — 放弃引擎聚焦、双焦点与 `NavRepeater` 合成投递 — source: ADR-0019
- **Never 在 HUD 出现 `unit_value` / 元每格 / 推荐顺序 / 最优解 / 默认排序 / 可吃性过滤** — 泄露答案，破坏 GDD 的「玩家自己算」 — source: ADR-0019, ADR-0010
- **Never 乐观更新**：`submit_action` 被拒绝时 HUD 不修改任何数字，只显示原因 — source: ADR-0019
- **Never 让 HUD 自行扣回合、丢菜、改容量、推进波次** — 所有写都经 `RunFlow.submit_action` — source: ADR-0019
- **Never 让 HUD 监听 `ui_pause`**（会与 `SceneFlow` 抢暂停），也 **Never 让 HUD 读 `RunResult`** — source: ADR-0019
- **Never 在 HUD `_process` 里轮询上游服务** — 每帧读服务会制造跨快照混帧 — source: ADR-0019
- **Never 把 HUD 或引导的布局常量写进代码字面量** — 常量只在 `HudTheme` / `OnboardingConfig` 资源里，代码里出现 72 / 280 / 200 / 16 / 176 / 44 即违规 — source: ADR-0019, ADR-0020
- **Never 把引导做成 `UIRoot` 的模态屏幕** — `Run` 的 `ui_stack_depth` 会变成 1，HUD 被屏幕遮住，`Modal` 还会拦截 B1 的确认 — source: ADR-0020
- **Never 不压 `Blocked` 而让 B1 的确认照常到达 HUD** — 教学与动作互相污染，「只推进、不吃菜」无法断言 — source: ADR-0020
- **Never 让引导自己解析锚点或自己算布局** — 两份布局必然漂移 — source: ADR-0020, ADR-0019
- **Never 在 `run_settled` 构建 `FirstRunExplanation`** — 钱包尚未入账，首笔升级可负担性会读旧值 — source: ADR-0020, ADR-0007
- **Never 持久化「引导已看」** — GDD 明确不写持久化，且「跳过再放弃」会让玩家永远看不到教学 — source: ADR-0020
- **Never 用计时器 / `await` / `call_deferred` / 打字机 / 自动关闭推进引导** — 推进只由输入事件驱动 — source: ADR-0020
- **Never 让气泡压住回合/容量/净利，或整屏压暗后再在读数上重绘** — 遮挡与重绘都会破坏受保护矩形的原亮度与原对比度 — source: ADR-0020
- **Never 用阻塞式错误框替代降级** — 十二级降级都不得改变 `run_state`、不得扣回合/容量、不得写存档 — source: ADR-0020
- **Never 把菜名、字符串或替换建议写进 `RunResult` / 存档 / `RunFlow` 快照** — `FirstRunExplanation` 只用于展示 — source: ADR-0020, ADR-0015

### Performance Guardrails

- **Audio request**：p95 ≤ 0.5 ms，最差 2 ms；每帧 tick ≤ 0.05 ms；日志约 1.25 MB 常驻 — source: ADR-0009
- **Themes / 字体**：约 10 MB 常驻（三档字号同一主题） — source: ADR-0008
- **UI 过渡**：≤ 0.20 s，`reduce_motion` 时为 0；`UIRoot` 空闲时零每帧开销 — source: ADR-0003
- **HUD**：空闲 `_process` 只有三次 `is_action_just_pressed` 与一次状态判断，≤ 0.5 ms/帧 @1080p；一次事件构造 = O(可见菜 + 腻度组 + 预览)，MVP 上限 4+4+2，远低于 1 ms — source: ADR-0019
- **HUD 内存**：`HudFrame` < 16 KiB，`_last_legal_frame` 只保留一份，卸载时全部释放；挂载无磁盘 I/O — source: ADR-0019
- **引导**：空闲 ≤ 0.05 ms/帧；`INTRO` / `NUMBERS` 每帧一次快照 + 两次 `consume_action` ≤ 0.1 ms；`COACHING` 只在焦点变化时读 `group_counts()` — source: ADR-0020
- **引导内存 / 启动**：`FirstRunExplanation` < 8 KiB；`FirstRunPreset.validate` 在 `Boot → Menu` 跑一次，6 道全排列最多 720 次、< 1 ms — source: ADR-0020
- **引导渲染**：一个全屏 `Control` 的补集遮罩（最多 4×3 个矩形）+ 一条 1 px 指引线 + 一个 `PanelContainer`；无粒子、无 shader、无屏幕震动 — source: ADR-0020

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

- **gdUnit4** — 唯一测试框架（`.claude/docs/technical-preferences.md` → Testing → Framework；测试根 `tests/unit/**/*_test.gd`、`tests/integration/**/*_test.gd`；运行命令见 `project.yaml` → `commands.test`）— source: `.claude/docs/technical-preferences.md`, `project.yaml` (`commands.test`)

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
- **4.6 双焦点**：鼠标悬停焦点与键盘/手柄焦点是两个状态，绘制分开 — source: `current-best-practices.md` (UI 4.6), ADR-0003, ADR-0019
- **类型化容器**：`Array[T]` / 具名类型的变量，不用裸 `Array` / `Dictionary` — source: `deprecated-apis.md` (Patterns)
- **`@onready` 缓存节点**，不用 `$NodePath` 在 `_process()` 里每次查路径 — source: `deprecated-apis.md` (Patterns)
- **测试命令**：`godot --headless -s -d --remote-debug tcp://127.0.0.1:0 res://addons/gdUnit4/bin/GdUnitCmdTool.gd -a res://tests --ignoreHeadlessMode`；退出码 0 通过 / 100 失败 / 101 孤儿或警告 / 105 解析错误 — source: `current-best-practices.md`, `project.yaml` (`commands.test`)
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
- **游戏逻辑不读墙钟**（时间戳只用于展示与日志） — source: ADR-0006, ADR-0007, ADR-0018
- **所有玩家可见字符串走 `tr()`**，无字符串拼接；HUD 与引导各有自己的文本表资源，`.gd` 里不写界面上可见的字面量 — source: ADR-0008, ADR-0019, ADR-0020
- **不扫 `res://` 目录发现资产**：内容靠清单，音频靠 cue 目录 — source: ADR-0001, ADR-0009
- **`PROCESS_MODE_WHEN_PAUSED` 全项目为零** — source: ADR-0009
- **每个 `Resource` 的公开属性只读**（getter-only + 防御性数组拷贝），运行期不改记录 — source: ADR-0001
- **金额一律 int64 + 受检加法**（`_checked_add` 前置比较）；禁止 float 求和、禁止回绕成负数、禁止静默饱和 — source: ADR-0015, ADR-0016, ADR-0018
- **负钱包合法、负净利合法**：结算不钳制 `net_profit`，局外升级允许 `wallet < 0`；任何上下限都属于局外策略，不属于结算 — source: ADR-0015, ADR-0016
- **派生量不落盘、不缓存**：`broke_even`、`return_ratio`、`effective_unit_value` 都在读取时计算 — source: ADR-0015, ADR-0018
- **存档边界不含 float / `StringName` / `Object` / `Vector*`**：冻结的浮点写成规范十进制字符串，只读快照再解码已知键 — source: ADR-0018, ADR-0002
- **同一事实只有一个所有者**：回合归 `TimeTurns`、容量归 `StomachCapacity`、净利累加器归 `RunFlow`、钱包归 `MetaUpgrades`、纪录与里程碑归 `Records`、解锁账本归 `Unlocks`；下游只读快照或调用纯函数 — source: ADR-0011, ADR-0012, ADR-0015, ADR-0016, ADR-0017, ADR-0018, ADR-0019
- **失败是结构化错误，不是占位值**：统一 `ok / value / error` 形状，`null` / `0` / `空数组` 不得表示失败 — source: ADR-0010, ADR-0011, ADR-0012, ADR-0014, ADR-0015, ADR-0016, ADR-0017, ADR-0018
- **一个事件一次采集**：HUD 与首局引导都只在一个事件内读一次上游，跨快照混帧即架构违规 — source: ADR-0019, ADR-0020
- **局内不泄露答案**：`unit_value`、`opt_value`、每格比值、推荐顺序、最优解不得出现在 HUD 与引导的局内视图中（`FirstRunExplanation` 的 `b4.*` 是唯一例外） — source: ADR-0010, ADR-0019, ADR-0020
- **引擎未实测**：`Installed at pin time = NOT DETERMINED`；ADR-0001…ADR-0020 的 `Verification Required` 条目（含 4.6 双焦点、4.7 `canvas_items`、`CanvasLayer` 逆缩放、`process_priority` 顺序）全部未在实机上跑过。凡依赖 4.5–4.7 运行时语义的 story，先在 pin 定的 4.7.x 上实测，再写实现 — source: `VERSION.md`, ADR-0001…ADR-0020
