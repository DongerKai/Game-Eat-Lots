# ADR-0020: 首局引导的 beat 机、输入阻断与收据讲解（First-Run Onboarding Beat Machine & Receipt Explanation）

## Status

Proposed

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。
> 其他 agent 与任何 skill 都不能自行设置。** 引用 `Proposed` ADR 的 story 会被置为
> `Blocked`，epic 要求 `Accepted`，所以这个字段必须有明确的负责人。

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

- 用户（`modes.automation: autonomous`，由用户先行授权）
- `godot-specialist`（`CanvasLayer` 层级、逆缩放坐标转换、`process_priority`、`MouseFilter`、`Tween` 暂停行为校验）
- `technical-director` 战略审阅：跳过（`review_mode: lean`）

## Summary

`first-run-onboarding.md` 把首局定义成一段**固定顺序、可跳过、无计时器**的讲解：B0 开场、B1 讲价格与饱腹、B2/B3 在自由取餐时作上下文提示、B4 在收据上给单位价值与一处替换建议。它只压输入、只放气泡，不改任何玩法规则，也不写任何存档。GDD 已经把「讲什么、何时讲、缺东西时怎么退」写到可验收的粒度；实现层仍有四个会直接制造双真相或死锁的缺口：

1. **谁决定这局是首局、菜单显示多少钱？** `is_first_run` 需要 `records.career.runs_settled == 0 ∧ LoadReport.first_run_eligible` 两项同时成立；菜单上的 40 与 `RunConfig.entry_fee_override` 的 40 必须同源，否则玩家按 85 规划、结算却扣 40。
2. **beat 到底由谁推进、输入由谁取走？** `Blocked` 上下文不向 `ActionSink` 投递（ADR-0005 D4），B1 的 `ui_accept` 若不被引导主动取走，就会在同一帧被 HUD 读成一次吃菜；而「同一帧只推进一个 beat」需要单调的 `input_seq`，不能靠节点顺序碰运气。
3. **气泡放在哪、怎样保证不压读数？** 锚点归 HUD、位置归引导；150% 字号、超宽屏、锚点缺失三种情况都必须有确定输出，且任何模态遮罩都不能压暗回合/容量/净利三个受保护读数。
4. **收据讲解何时构建？** GDD 说 `run_settled` 后交付，但钱包入账发生在 `Run → Menu` 的结算事务里（ADR-0007 D3）；在 `run_settled` 算「第一笔升级买不买得起」会读到结算前的旧钱包。

本 ADR 决定：

1. **首局判定与预置由 Application 层的 `FirstRunPlanner` 产出唯一 `FirstRunRunPlan`**：`FirstRunGate` 做三态判定（`Fresh` / `Loaded` / `Migrated` 且 `runs_settled == 0`；`Recovered` 恒否），`FirstRunPreset` 持有六道菜与 40 元常量并跑七条内容不变式，`FirstRunRunPlan` 同时供主菜单显示与 `RunConfig` 组装——菜单费与实收费只有一个来源。
2. **`OnboardingHost` 是唯一的引导适配器**（`CanvasLayer`，layer 15，`process_mode = INHERIT`），内部由**纯 beat 机**（`BeatMachine`）与**纯几何/文本模型**（`BubbleLayout` / `ScrimPieces` / `ExplanationBuilder` / `OnboardingTextTable`）驱动；它只写 `push_context(Blocked, allow)` / `pop_context()`，其余全部只读。
3. **模态 beat 用拉模型取走输入**：INTRO / NUMBERS 阶段每帧经 `InputRouter.consume_action` 同时取走 `ui_cancel` 与 `ui_accept`（cancel 优先），HUD 同帧看不到残留的确认；`OnboardingHost.process_priority = -50`，严格位于 `SceneFlow`（−100）与 `HudHost`（0）之间。
4. **上下文提示不压输入**：B2/B3 不改变上下文、不挡动作、不移动焦点；同一时刻最多一条，B3 替换 B2；触发即完成，无计时器、无自动推进。
5. **锚点与受保护矩形复用 ADR-0019**：`get_anchor_rect` 取六个 id，`get_protected_rects` 固定三元组（回合/容量/净利）；`BubbleLayout` 在窗口像素坐标里按 GDD 公式算上方/下方/底部停靠，`ScrimPieces` 是受保护矩形的**补集**，三个读数保持原亮度。
6. **`FirstRunExplanation` 在 `Run → Menu` 第 5 步构建并交付**：`run_settled` 只深拷贝 `RunResult` 与 `get_run_menu()`，等结算事务（第 2 步）把钱包与纪录落定后，再算逐行单位价值、替换建议与首笔升级可负担性；同一次调用广播一次 `onboarding_finished(ALL_BEATS)`。
7. **降级永不阻塞**：缺锚点 → 停靠；缺费覆盖 → 菜单与本局都用档口价并省略「教学价」句；缺池覆盖 → 用实际菜池并省略定量句；缺单位价值 → B4 只给文案；缺文案 → 跳过该 beat 并记 ERROR；预置不变式失败 → 关闭引导、记 ERROR、CI 失败，但这一局照常开始。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；本机编辑器 `NOT DETERMINED`） |
| **Domain** | UI / Input |
| **Layer** | Presentation（`OnboardingHost`）、Application（`FirstRunPlanner`） |
| **Knowledge Risk** | HIGH — `CanvasLayer` 层序与逆缩放后的坐标转换、`MouseFilter.IGNORE` 遮罩下的悬停/点击穿透、`process_priority` 跨节点顺序、`consume_action` 的共享消费位、`request_context(clear_stack)` 后重压栈，均未在参考库中完全覆盖 |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`modules/ui.md`、`modules/input.md`、`breaking-changes.md`、`deprecated-apis.md`、`current-best-practices.md` |
| **Post-Cutoff APIs Used** | 4.6 双焦点分离（键盘/手柄焦点与鼠标悬停各自独立）；4.7 `canvas_items` / `expand` 默认值；`Control.focus_neighbor_*`；`CanvasLayer.transform`（与 ADR-0019 相同） |
| **Verification Required** | ① `CanvasLayer` layer 15 位于 HUD（10）之上、`ScreenHost`（20）与 `ModalHost`（30）之下，收据与暂停菜单能正常覆盖引导；② HUD 锚点矩形（窗口像素）转入 `TutorialLayer` 局部坐标后与气泡/遮罩逐像素一致，逆缩放与设计单位降级两种模式都成立；③ 全屏 `Control.mouse_filter = IGNORE` 的遮罩不阻断 HUD 卡片的悬停与点击聚焦；④ `process_priority = -100 / -50 / 0` 在真实 `_process` 顺序中稳定（暂停 → 引导 → HUD）；⑤ 同一帧两个节点调用 `consume_action` 时共享消费位使第二个读到 `false`；⑥ `request_context(Paused, clear_stack := true)` 后 `push_context(Blocked)` 能成功重建且不残留旧栈；⑦ `Tween` 在 `process_mode = INHERIT` 下随暂停停在最后一帧；⑧ 4.6 下鼠标悬停是否满足 `has_focus()`（ADR-0003 Verification ⑦），以及 HUD 的 `focused_dish_id()` 在「键盘焦点优先、悬停兜底」下是否稳定；⑨ 150% 字号下 48 字上限与实际换行行数一致，UI 不出现省略号或运行时截断；⑩ `CanvasLayer` 上的 `Control` 在 `MOUSE_FILTER_IGNORE` 遮罩存在时仍能收到 `mouse_entered` / `mouse_exited`，供 HUD 维护悬停记录 |

> **Knowledge Risk 为 HIGH。** 升级引擎版本时本 ADR 必须重新验证；若 `CanvasLayer` 逆缩放或层序不可行，按 ADR-0019 D5.1 的降级路径改为「设计单位布局 + 引擎缩放」，引导随之改用设计单位坐标，但 `get_protected_rects()` 的语义不变。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0002（`LoadReport.outcome` 词汇与 `Recovered` 语义；本 ADR 消费首局资格判定）— 见 GDD Sync Check 第 1 条；ADR-0003（`CanvasLayer` 分层、`FocusSpec`、双焦点、收据先于 HUD 卸载挂载）；ADR-0004（`state_changed` 单订户固定扇出、`run_*` 订阅顺序、`AppRoot` 构造全部服务）；ADR-0005（`Blocked` 的 `allow` 白名单、`consume_action` 仅 `Blocked` 生效、共享消费位、`input_seq`、`ui_pause` 恒允许）；ADR-0006（`MenuWaves.get_run_menu` / `visible_dishes` / `wave_index` / `wave_count`）；ADR-0007（`run_settled` 是预通知、唯一 `Save.commit` 在 `Run → Menu` 第 2 步）；ADR-0010（`DishValues.price` / `satiety` / `unit_value` / `uv_cmp`）；ADR-0014（`CloyingAndTaste.preview` / `record_eat` / `group_counts`）；ADR-0015（`RunResult.rows[]` / `entry_fee` / `capacity_total` / `capacity_used`）；ADR-0016（钱包与升级目录）；ADR-0018（`RecordsSnapshot.career.runs_settled`）；ADR-0019（`get_anchor_rect` / `get_protected_rects` / `focused_dish_id` / `HudLayoutRects` / 逆缩放） |
| **Enables** | 首局引导（#18）的全部实现 story；`TR-first-run-onboarding-001/003/004/005` 的实现验收；收据屏的 `FirstRunExplanation` 讲解块；音效反馈对 `onboarding_beat_shown` / `onboarding_finished` 的可选订阅 |
| **Blocks** | 任何需要 `FirstRunRunPlan`、`FirstRunExplanation`、引导遮罩或 `Blocked` 白名单验收的 story；`design/ux/` 的 tutorial layer 规格（必须先与本 ADR 的气泡/遮罩契约对齐） |
| **Ordering Note** | 本 ADR 是 `architecture.md` Required ADRs 的第 16 项之后的收尾项，编号与决策顺序一致。拓扑序保持 0001 → 0002 → 0005 → 0003 → 0004 → 0006..0009 → 0010..0018 → 0019 → 0020。ADR-0005 已覆盖 `TR-first-run-onboarding-002`（`Blocked` 白名单、一次输入一 beat），本 ADR 只消费其结论并实现 beat 机的消费侧，不重复认领该 TR |

## Context

### Problem Statement

`first-run-onboarding.md` 的 22 条 Core Rules 与 7 条 Formulas 已经把「讲什么、何时讲、缺东西怎么退」写得很细，但实现层仍有四个会直接制造双真相或死锁的缺口：

1. **首局身份与菜单价格有两个潜在来源。** `is_first_run` 由界面流程组装 `RunConfig` 时决定，主菜单又要显示「本局实际要收的费」。若菜单读 `StationData.entry_fee`、结算读 `entry_fee_override`，玩家会按 85 规划、在收据上看到 40。GDD 规则 3 已声明这条一致性要求（S9），但没有规定**谁产出那个唯一值**、`Recovered` 怎样参与、预置不变式失败时又该显示什么。
2. **`Blocked` 不投递，意味着引导必须主动拉。** ADR-0005 D4 规定 `Blocked` 不向 `ActionSink` 投递，`ui_accept` 只能经 `consume_action` 被取走。若引导不取，HUD 的 `is_action_just_pressed(&"ui_accept")` 会读到 `true` 并提交一次吃菜；若引导取晚一帧，B1 的「只推进、不吃菜」就失效。GDD 规则 6 与 AC 21 要求同一帧至多推进一个 beat，这需要单调 `input_seq` 与确定的帧内优先级，而不是节点树顺序。
3. **气泡几何与遮罩没有落到类型上。** GDD 给了 `bubble_rect` 公式与「不得与三个受保护矩形相交」，但 B2 的锚点是 `bar:capacity`、视觉要求又说非模态提示固定在详情行上方；B3 同时给了 `card:<该菜>` 与 `rail:cloy` 两个锚点。没有一条规则说明哪一个决定位置、另一个是否只用于指引线。遮罩更危险：全屏压暗若盖住净利/容量/回合，就直接违反规则 12 与 AC 5。
4. **收据讲解的构建时点与结算事务错位。** GDD 说 `run_settled` → `Closed` → 交付 `FirstRunExplanation` 并广播 `ALL_BEATS`；ADR-0007 又把唯一 `Save.commit` 放在 `Run → Menu` 的 `UIRoot.reset_for_state` 内。若在 `run_settled` 构造讲解，「首笔升级是否买得起」会读到**未入账**的钱包；若等到 `Run → Menu` 之后太久，收据已经画完。必须把「捕获结果」与「构建讲解」拆成两个明确时点，且都在收据首次绘制之前。

### Current State

- 仓库没有 `.gd`；本 ADR 是绿地决策。
- ADR-0003 已固定 `CanvasLayer` 分层：HUD 10、`ScreenHost` 20、`OverlayHost` 25、`ModalHost` 30、`NoticeHost` 40、`FadeLayer` 50；**没有引导层**。
- ADR-0004 已固定 `state_changed` 单订户六步扇出，第 5 步是 `_onboarding.on_flow_state(from, to)`；`run_started` 的订阅顺序是 `HudHost → Onboarding → Audio`；`run_settled` 的订阅者只列了 `UIRoot` 与 `Audio`。
- ADR-0005 已固定 `Blocked` 不投递、`consume_action` 仅 `Blocked` 生效、`push_context` 无条件加深、`ui_pause` 在 `Blocked` 上恒允许、`input_seq` 单调；`process_priority` 只登记了 `SceneFlow`（−100）与 `HudHost`（0）。
- ADR-0007 已固定 `run_settled` 是预通知、`Run → Menu` 第 2 步才 `Save.commit`、收据同帧挂载；并明确「`run_settled` 处理器可把结果记为待交付，供首局引导 B4 使用」。
- ADR-0019 已固定 `HudHost.get_anchor_rect` 六个 id、`get_protected_rects` 三元组、`focused_dish_id`、逆缩放布局；**没有暴露安全边距/行高/字号/详情行顶边**。
- ADR-0002 的 `LoadReport` 字段沿用 GDD，但字段表**没有 `first_run_eligible`**；`save-system.md` 规则 26 已定义该派生值。
- `ui-flow.md` 规则 8 与 AC 47 使用 `load_report().first_run_eligible ∧ runs_settled == 0`，但 **AC 6 仍写 `is_first_run = (career.runs_settled == 0)`**，与 S11 / AC 31 冲突。
- `design/gdd/first-run-onboarding.md` 头部仍标 `Status: In Design`，而 `systems-index.md` 第 75 行标 `Approved`。
- `design/ux/` 尚无 tutorial layer 规格；本 ADR 只固定代码契约与几何下限。

### Constraints

- **只压输入，不改规则**：唯一写操作是 `push_context(Blocked, allow)` / `pop_context()`；不得写 `run_state`、`run_id`、`frozen`、回合、容量、菜单波次、存档、钱包或纪录。
- **无计时器、无自动推进**：不得注册 `Timer`、`SceneTreeTimer`、`await`、`call_deferred()`；空闲 30 秒不改变任何状态。
- **同帧**：B0 必须在 `run_started` 同帧压入 `Blocked`；B1 的 `ui_accept` 必须在 HUD 轮询之前被取走；B4 必须在收据首次绘制前交付。
- **无单例**：全部上游以构造参数注入 `OnboardingSource`；禁止 `get_node("/root/…")`、`static var` 服务或依赖袋（ADR-0004 D3）。
- **无头可测**：`BeatMachine` / `BubbleLayout` / `ScrimPieces` / `ExplanationBuilder` / `FirstRunGate` / `FirstRunPreset` / `FirstRunRunPlan` / `OnboardingTextTable` 必须不引用 `Node`，能在 GdUnit 无渲染环境下断言。
- **1280×720 基准**：最低支持分辨率；150% 字号是最坏文案情况（48 字），4K 上气泡宽度封顶 440 px。
- **颜色不是唯一通道**：跳过按钮、焦点、高亮都必须有文字标签；遮罩不得降低三个受保护读数的对比度。

### Requirements

- 必须给出 `is_first_run`、`first_run_eligible`、六道菜池与 40 元教学价的唯一判定与唯一来源，并让菜单显示值与 `RunConfig` 实收值同源。
- 必须给出 B0–B4 的状态机、事件表、推进顺序与「同一帧只推进一个 beat」的实现方式。
- 必须保证模态 beat 的 `ui_accept` / `ui_cancel` 不到达 HUD/`RunFlow`，上下文提示不改变输入上下文。
- 必须给出气泡几何、遮罩补集、锚点降级与三个受保护矩形在 720p / 1080p / 4K / 150% 下的确定输出。
- 必须让跳过、暂停、恢复、放弃、故障五条路径都不产生死锁，且完成信号每局至多一次。
- 必须给出 `FirstRunExplanation` 的字段、构建时点、替换建议五分支与首笔升级可负担性。
- 必须把全部玩家可见文本集中到 `onboarding_text`，用生成期 lint 保证 48 字上限与禁用措辞。
- 必须让预置不变式失败、锚点缺失、费覆盖缺失、池覆盖缺失、单位价值缺失都有明确降级，且都不阻止开局。

## Decision

### D1 — 组成：Application 规划、Presentation 执行、纯模型可测

```
AppRoot (组合根，Application)
├── FirstRunPlanner (RefCounted)                 # 首局判定 + 预置校验 + 唯一 plan
│     ├── FirstRunGate                           # 三态真值 + Recovered 例外
│     ├── FirstRunPreset                         # 六道菜 / 40 元 / 七条不变式
│     └── FirstRunRunPlan                        # 菜单与 RunConfig 的共同输入
│
├── OnboardingHost (CanvasLayer, layer 15, INHERIT)   # 唯一引导适配器
│     └── TutorialLayer (Control, 全屏, 逆缩放同 HudHost)
│          ├── Scrim (Control, _draw, MOUSE_FILTER_IGNORE)
│          ├── GuideLine (Control, _draw, MOUSE_FILTER_IGNORE)
│          └── Bubble (PanelContainer, 默认隐藏)
│               ├── Body (Label)
│               └── SkipButton (Button, FOCUS_NONE)
│
└── 纯模型（全部 RefCounted，无 Node）
      BeatMachine / BubbleLayout / ScrimPieces / ExplanationBuilder /
      OnboardingTextTable / OnboardingSource / FirstRunInvariants
```

- **`FirstRunPlanner` 属于 Application 层**：它读 `LoadReport`、`RecordsSnapshot`、`DataConfig`、`MetaUpgrades` 与 `DishValues`，产出 `FirstRunRunPlan`；它不绘制、不压上下文、不写任何状态。`UIRoot` 只在组装 `RunConfig` 时读 `plan`，不自行推导 `is_first_run`。
- **`OnboardingHost` 属于 Presentation 层**：它只消费 `plan`、`RunFlow` 快照、HUD 锚点/焦点、`MenuWaves` 只读投影与 `CloyingAndTaste` 组计数；唯一写是 `push_context(Blocked)` / `pop_context()`。
- **`OnboardingHost` 不是 `UIRoot` 的屏幕**：它不进入 `ScreenHost`、不参与 `replace_root`、不改变屏幕栈深度；`Run` 的 `ui_stack_depth = 0` 保持成立（`ui-flow.md` 状态表）。
- **不是 `Modal`**：引导不调用 `push_context(Modal)`；`ModalHost`（layer 30）永远在引导之上，暂停菜单/确认框可以覆盖它。
- **层序固定**：HUD 10 < **引导 15** < 屏幕 20 < 暂停/覆盖 25 < 模态 30 < 通知 40 < 淡出 50。引导只压在 HUD 之上，收据出现时同帧卸载，不会盖住收据。
- **`TutorialLayer` 与 `HudHost` 使用同一套逆缩放**（ADR-0019 D5.1）：1080p 下两者都按窗口像素 1:1 绘制；若 HUD 走设计单位降级，引导也走同一降级，不出现「气泡按窗口像素、锚点按设计单位」的错位。

### D2 — 首局判定、预置与唯一 `FirstRunRunPlan`（TR-001）

#### D2.1 三态真值与 `Recovered` 例外

```gdscript
class FirstRunGate extends RefCounted

enum Reason {
    ELIGIBLE, CAREER_EXISTS, RECOVERED,
    RECORDS_UNAVAILABLE, PRESET_INVALID,
}

class Decision extends RefCounted
var eligible: bool
var reason: int
var first_run_eligible: bool     # 存档资格（与 runs_settled 相与前的值）
var runs_settled: int

static func evaluate(load_report: LoadReport,
                     records: RecordsSnapshot,
                     preset_valid: bool) -> Decision
```

固定判定顺序：

1. `first_run_eligible = load_report.first_run_eligible`（字段存在时直接用）。
   字段缺失时按 `save-system.md` 规则 26 派生：`outcome ∈ {Fresh, Loaded, Migrated}`；`Recovered` / `ReadOnly` / `Blocked` 恒为 `false`。调试构建断言「字段值 == 派生值」，正式构建以派生值为准并记一次 WARNING（见 GDD Sync Check 第 1 条）。
2. `runs_settled = records.career.runs_settled`；`records` 不可用（未初始化 / 结构损坏）→ `RECORDS_UNAVAILABLE`，`eligible = false`，记 WARNING。宁可少教一次，也不把老玩家重新塞回教学。
3. `preset_valid = false` → `PRESET_INVALID`，`eligible = false`，记 ERROR；**CI 必须失败**（见 D2.2）。
4. `load_report.outcome == Recovered` → `RECOVERED`，`eligible = false`，即使 `runs_settled == 0`。
5. 其余 `runs_settled == 0 ∧ first_run_eligible` → `ELIGIBLE`；否则 `CAREER_EXISTS`。

**禁止**：`OnboardingHost` 读 `SaveSystem`、`LoadReport` 或 `RecordsSnapshot`；它只消费 `plan.is_first_run`。GDD 规则 2 的「本系统不读存档」由此变成结构事实。

#### D2.2 六道菜、40 元与七条内容不变式

```gdscript
class FirstRunPreset extends RefCounted

const DISH_IDS: Array[StringName] = [
    &"dish_seafood_pot", &"dish_sweet_sour_pork", &"dish_braised_pork",
    &"dish_steamed_fish", &"dish_fried_chicken", &"dish_mapo_tofu",
]
const ENTRY_FEE: int = 40

class Context extends RefCounted
var station_id: StringName
var capacity_total: int                    # C0 + capacity_bonus
var official_entry_fee: int                # StationData.entry_fee
var cheapest_capacity_upgrade_price: int   # 最便宜容量升级的 1 级价

class Validation extends RefCounted
var valid: bool
var failed: Array[int]          # 失败的不变式编号 1..7
var detail: Dictionary

static func validate(data: DataConfig, dishes: DishValues,
                     ctx: Context, make_cloying: Callable) -> Validation
```

七条不变式逐条来自 GDD 的 `first_run_pool_invariants`：

| # | 检查 | 实现要点 |
|---|---|---|
| 1 | `P ⊆ station_canteen.dish_pool`，`|P| = 6`，id 唯一，六道 `station_id` 相同 | 从 `DataConfig.get_station` 与 `get_dish` 读；缺 id 即失败 |
| 2 | `Σ satiety(d) = C0` | `DishValues.satiety` 求和；`C0 = ctx.capacity_total` |
| 3 | 恰好两对同组，其余两道分属不同组 | 按 `DishData.cloying_group` 计数；`group == ""` 不参与配对 |
| 4 | 存在可行 5 道子集、不存在可行 6 道子集；放弃价格最低的一道是唯一最优 | 对 6 道全排列用 `make_cloying()` 新建临时 `CloyingAndTaste` 重放 `preview` / `record_eat`，`Σ effective ≤ C0` 即可行；**不得另写一份腻度公式** |
| 5 | `max unit_value / min unit_value ≥ 1.5`；最低单位价值属于最高价格档；最便宜的一道单位价值在前两名 | `DishValues.unit_value` 与 `price`；并列按 `dish_id` 升序 |
| 6 | `Σ price(P \ 最便宜) − 40 ≥ ctx.cheapest_capacity_upgrade_price` 且 `≥ 0` | 价格求和只用于设计期校验，不进入运行期 HUD/结算 |
| 7 | `Σ price(P \ 最便宜) − ctx.official_entry_fee < 0` | 正式价下必亏，否则教学价没有理由 |

- **执行时机**：`FirstRunPlanner.refresh()` 在 `Boot → Menu`、`Run → Menu` 结算之后、以及生涯删除之后各跑一次；不做每帧检查。
- **失败语义**：`valid = false` → `plan.is_first_run = false`、`plan.preset_valid = false`、记 ERROR、关闭引导；**这一局照常开始**。CI 另用固定夹具（`C0 = 20`、正式费 85、最便宜升级 12）跑同一函数，任一失败即红。
- **常量而非配置**：`DISH_IDS` 与 `ENTRY_FEE` 是内容常量，**不放进 `OnboardingConfig`**；调参不得改变首局的取舍（GDD 规则 20）。

#### D2.3 唯一 `FirstRunRunPlan`

```gdscript
class FirstRunRunPlan extends RefCounted
var is_first_run: bool
var pool_override: Array[StringName]     # is_first_run = false 时为空
var entry_fee_override: int              # -1 表示 null
var menu_entry_fee: int                  # S9：主菜单显示值
var official_entry_fee: int
var preset_valid: bool
var reason: int                          # FirstRunGate.Reason

func has_entry_fee_override() -> bool
func run_config_fields() -> Dictionary   # {is_first_run, pool_override, entry_fee_override}
```

**单一来源**：

- 主菜单开始按钮旁的入场费显示 `plan.menu_entry_fee`；`is_first_run = true` 时恒为 40，否则为 `official_entry_fee`（MVP 85）。
- `RunConfig.entry_fee_override` 写入 `plan.has_entry_fee_override() ? plan.entry_fee_override : null`；`RunConfig.pool_override` 写入 `plan.is_first_run ? plan.pool_override.duplicate() : null`。
- `RunConfig.is_first_run = plan.is_first_run`。
- **降级**：`entry_fee_override` 不被上游支持时，`FirstRunPlanner` 在组装时就把它视为不存在：`menu_entry_fee = official_entry_fee`、`has_entry_fee_override() = false`、`B0` 省略「教学价」句并记 WARNING。菜单与本局同时变成 85，不允许一边 40 一边 85（GDD 规则 3 末段）。
- **「再来一局」**：`ui-flow.md` 规则 12 已固定 `is_first_run = false`；`FirstRunPlanner` 不为该入口产出新 plan，`OnboardingHost` 不激活。

**装配与传递**：

```gdscript
# UIRoot 组装 RunConfig 的同一函数内
var plan := _first_run_planner.plan()
var config := RunConfig.new(...)
config.is_first_run = plan.is_first_run
config.pool_override = plan.is_first_run ? plan.pool_override.duplicate() : null
config.entry_fee_override = plan.has_entry_fee_override() ? plan.entry_fee_override : null
```

`UIRoot` 在请求 `Menu → Run` 之前发一次 `run_config_assembled(config, plan)`（新增的 UIRoot 信号）；`AppRoot` 订阅它并调用 `onboarding.configure(plan)`。这样 plan 在 `run_started` 之前或之后到达，`OnboardingHost` 都能在**同一帧**完成「配置 + 压 `Blocked` + 显示 B0」，不依赖信号先后（见 GDD Sync Check 第 10 条）。

### D3 — Beat 机：固定序列、一次输入一 beat、无计时器

#### D3.1 状态与事件

```gdscript
class BeatMachine extends RefCounted

enum Phase { IDLE, INTRO, NUMBERS, COACHING, CLOSING, CLOSED, CANCELLED }
enum Beat { NONE, B0_WELCOME, B1_TWO_NUMBERS, B2_CAPACITY, B3_CLOYING, B4_RECEIPT }
enum Event {
    RUN_STARTED, ACCEPT, CANCEL, FOCUS_CHANGED, ACTION_COMMITTED,
    RUN_ENDED, RUN_SETTLED, FLOW_LEFT_RUN, DISCARDED, FAULTED,
    PAUSED, RESUMED, CONTEXT_RESTORED, CONTEXT_RESTORE_FAILED,
    TEXT_MISSING,
}
enum Intent {
    PUSH_BLOCKED, POP_BLOCKED, SHOW_BEAT, HIDE_BEAT,
    MARK_SHOWN, DEGRADE_BEAT, STASH_RESULT, DELIVER_EXPLANATION,
    FINISH, WARN,
}

class Facts extends RefCounted
var input_seq: int              # 输入帧序号；非输入事件为 0
var focus_id: StringName        # &"" 表示无焦点
var group_count: int            # 焦点菜所属组的当前计数（B3）
var frame: int                  # Engine.get_process_frames()
var run_id: int
var result: RunResult           # 仅 RUN_SETTLED
var run_menu: Array[StringName] # 仅 RUN_SETTLED

func on(event: int, facts: Facts) -> Array[Intent]
func required() -> Array[int]
func shown() -> Array[int]
func phase() -> int
func current_beat() -> int
```

#### D3.2 转移表

| 当前 Phase | 事件 | 下一 Phase | 输出 Intent | 备注 |
|---|---|---|---|---|
| `IDLE` | `RUN_STARTED` | `INTRO` | `PUSH_BLOCKED(B0_ALLOW)`、`SHOW_BEAT(B0, MODAL)` | `run_started` 同帧；锚点可空则先隐藏，等下一帧 |
| `INTRO` | `ACCEPT` | `NUMBERS` | `POP_BLOCKED`、`PUSH_BLOCKED(B1_ALLOW)`、`MARK_SHOWN(B0)`、`SHOW_BEAT(B1, MODAL)` | 只推进，不提交动作 |
| `INTRO` | `CANCEL` | `CLOSED` | `POP_BLOCKED`、`HIDE_BEAT`、`FINISH(SKIPPED)` | 跳过 |
| `NUMBERS` | `ACCEPT` 且 `focus_id != &""` | `COACHING` | `POP_BLOCKED`、`MARK_SHOWN(B1)`、`SHOW_BEAT(B2, CONTEXTUAL)`、`MARK_SHOWN(B2)` | 先压 B2 再弹栈；同帧 HUD 看不到 accept |
| `NUMBERS` | `ACCEPT` 且 `focus_id == &""` | `NUMBERS` | `WARN(NO_FOCUS)` | 不推进、不吃菜、无计时 |
| `COACHING` | `FOCUS_CHANGED` 且 `group_count ≥ 1` 且 B3 未展示 | `COACHING` | `SHOW_BEAT(B3, CONTEXTUAL)`、`MARK_SHOWN(B3)` | 替换 B2；`B3.triggered = true` |
| `COACHING` | `ACTION_COMMITTED` | `COACHING` | `HIDE_BEAT`；若 B2 未展示 → `SHOW_BEAT(B2)`、`MARK_SHOWN(B2)` | 兜底补展示；不重放已展示的 B3 |
| `COACHING` | `RUN_ENDED` | `CLOSING` | `HIDE_BEAT` | 丢弃未展示提示，不补弹 |
| `CLOSING` | `RUN_SETTLED` | `CLOSING` | `STASH_RESULT` | 只捕获，不构建讲解（见 D7） |
| `CLOSING` | `FLOW_LEFT_RUN` | `CLOSED` | `DELIVER_EXPLANATION`、`FINISH(ALL_BEATS)` | 结算事务已跑完；收据已挂载 |
| 任意活跃态 | `DISCARDED` / `FAULTED` | `CANCELLED` | `POP_BLOCKED`（若持有）、`HIDE_BEAT` | 不产出 B4、不广播完成 |
| 任意模态态 | `CANCEL` | `CLOSED` | `POP_BLOCKED`、`HIDE_BEAT`、`FINISH(SKIPPED)` | 跳过只影响本局 |
| 任意活跃态 | `PAUSED` | 不变 | 无 | 宿主记 `_blocked_owned = false`（栈已被清） |
| `INTRO` / `NUMBERS` | `RESUMED` | 不变 | `PUSH_BLOCKED(同一白名单)` | 失败则 `DEGRADE_BEAT`（见 D4.3） |
| 任意活跃态 | `TEXT_MISSING` | 不变 | `DEGRADE_BEAT(beat)`、`WARN(TEXT_MISSING)` | 该 beat 移出 `required()`；不截断、不缩字号 |

**「同一帧只推进一个 beat」**：`ACCEPT` / `CANCEL` 只在 `facts.input_seq > _last_consumed_seq` 时生效；生效即写 `_last_consumed_seq = facts.input_seq`。`input_seq` 来自 ADR-0005 D5，单调递增、不随上下文重置，因此同一帧两次 `ui_accept` 只推进一次，第二个事件被丢弃（GDD AC 21）。

**`COACHING` 不消费任何动作**：它不压上下文，`consume_action` 在 `Run` 中调用会返回 `false` 并记 WARNING（ADR-0005 D4）；提示只观察 `action_committed` 与焦点变化。

#### D3.3 必需集合与完成

```gdscript
required = {B0, B1, B2, B4} ∪ ({B3} if B3.triggered else ∅)
complete = required ⊆ shown ∧ B4 delivered
```

- `B3.triggered` 只在「焦点进入组计数 ≥ 1 的菜」那一刻置位；一旦置位就进入必需集合，即使随后跳过波次导致该组消失。
- 完成时不发奖励、不弹横幅、不播庆祝音；只广播一次 `onboarding_finished(ALL_BEATS)`。
- 被 `DEGRADE_BEAT` 移除的 beat 不计入 `required`；完成信号仍为 `ALL_BEATS`，但 `OnboardingHost` 同时记一次 `DEGRADED` 诊断（不新增枚举，供日志与验收）。
- `onboarding_finished` 每局至多一次；放弃/故障不发（GDD 规则 17）。

### D4 — 输入阻断、消费、跳过与暂停（TR-002 的消费侧）

#### D4.1 白名单

| Beat | `allow` | 说明 |
|---|---|---|
| `B0_WELCOME` | `[&"ui_accept", &"ui_cancel"]` | 四个导航动作不在表内：焦点不动 |
| `B1_TWO_NUMBERS` | `[&"ui_accept", &"ui_cancel", &"ui_up", &"ui_down", &"ui_left", &"ui_right"]` | 玩家自己选菜；引导不替玩家选 |
| `B2_CAPACITY` / `B3_CLOYING` | 不压栈 | 上下文提示，不改变 `Run` 的允许集 |
| `ui_pause` | 恒允许（`Blocked` 不可移除） | 暂停的最后一道出口（ADR-0005 D3） |

- `push_context(&"Blocked", allow)` 只在 `INTRO` / `NUMBERS` 调用；同一时刻至多一层。`_blocked_owned` 记录本系统是否持有该层，`pop_context()` 只在为真时调用，且成功后置 `false`。
- **禁止**在 `COACHING` / `CLOSING` 调用 `push_context` / `pop_context`；这两个阶段没有引导持有的上下文。

#### D4.2 拉模型与帧内顺序

```gdscript
# OnboardingHost
process_priority = -50          # SceneFlow -100 < Onboarding -50 < HudHost 0
process_mode = PROCESS_MODE_INHERIT

func _process(_delta: float) -> void:
    if not _active or _machine.phase() in [CLOSED, CANCELLED]:
        return
    if _machine.phase() == BeatMachine.Phase.COACHING:
        _poll_focus()
        return
    if _machine.phase() not in [BeatMachine.Phase.INTRO, BeatMachine.Phase.NUMBERS]:
        return

    var snap := _input.get_context_snapshot()
    if snap.input_seq == _last_input_seq:
        return
    _last_input_seq = snap.input_seq

    # 两个都取：cancel 优先；取走 accept 保证 HUD 同帧读不到
    var cancel := _input.consume_action(&"ui_cancel")
    var accept := _input.consume_action(&"ui_accept")
    if cancel:
        _advance(BeatMachine.Event.CANCEL, snap.input_seq)
    elif accept:
        _advance(BeatMachine.Event.ACCEPT, snap.input_seq)
```

- **必须先取 cancel 再取 accept**：Esc 在 `Blocked` 里被翻译成 `ui_cancel`（ADR-0005 D3）；两者同一帧都成立时，跳过优先，不推进。
- **每次轮询都尝试取走 accept**，即使本帧只想处理 cancel；否则 HUD 的 `is_action_just_pressed(&"ui_accept")` 会在同一帧读到 `true`。
- `get_context_snapshot().input_seq` 是单调帧序号；相同序号直接返回，避免同一帧重复推进。
- `process_priority = -50` 是契约常量：`SceneFlow`（−100）先处理暂停，引导（−50）先于 HUD（0）取走确认。启动时断言三者顺序（GDD Sync Check 第 12 条）。

#### D4.3 跳过、暂停与恢复

**跳过**：

- 模态 beat 中 `ui_cancel` → `CANCEL` 路径：`pop_context()`、隐藏遮罩/气泡、丢弃剩余 beat、`FINISH(SKIPPED)` 一次。
- 气泡右下角「跳过引导」`Button` 的 `pressed` → 同一条 `_skip()` 路径；按钮 `focus_mode = FOCUS_NONE`，键盘/手柄的等价入口是 `ui_cancel`（AC 27 的「`ui_cancel` 或焦点到达文字按钮」由此满足）。按钮文本进读屏顺序，鼠标只是可选加速器。
- **不二次确认**：跳过代价为零；确认框只是多一次点击（GDD 规则 16）。

**暂停**：

- `ui_pause` 由 `SceneFlow`（−100）处理；`Paused` 生效时 `request_context(Paused, clear_stack := true)` 清空临时栈，引导持有的 `Blocked` 随之失效。
- `OnboardingHost` 在 `on_flow_state(Run, Paused)` 收到 `PAUSED`：置 `_blocked_owned = false`，气泡与遮罩保持可见（`OverlayHost` layer 25 覆盖其上），不自行弹出任何东西。
- `on_flow_state(Paused, Run)` 收到 `RESUMED`：若仍在 `INTRO` / `NUMBERS`，用**同一白名单**重新 `push_context(Blocked)`。
- **重压失败**（返回 `false`）：不重试、不等待；`BeatMachine` 收到 `CONTEXT_RESTORE_FAILED` → `DEGRADE_BEAT`：当前 beat 改为无遮罩的底部停靠提示，玩家可以正常操作，首个 `action_committed` 或下一次 `ACCEPT`（此时 `Run` 中引导不再消费，动作会到达 HUD）推进/收起。**绝不出现「气泡在、按键无效」**（GDD 规则 17、AC 19）。
- 暂停期间 `_process` 随 `INHERIT` 停止；恢复后不补播、不重新计时。

**放弃 / 故障 / 初始化失败**：

- `run_discarded` / `run_faulted` → `CANCELLED`：若 `_blocked_owned` 则 `pop_context()`（失败也继续），隐藏所有元素，丢弃 stashed result，不产出 B4、不广播完成。
- `Menu → Run` 初始化失败不会发出 `run_started`；`OnboardingHost` 从未激活，`run_faulted(phase = INIT)` 只做清理，不触碰输入栈。

### D5 — 锚点、气泡几何与遮罩补集（TR-003）

#### D5.1 坐标空间

- `TutorialLayer` 与 `HudView` 同为窗口像素坐标：`OnboardingHost` 读取根视口内容缩放 `c`，把 `CanvasLayer.transform` 设为 `Transform2D(0, Vector2(1.0 / c, 1.0 / c), 0, Vector2.ZERO)`（ADR-0019 D5.1）。
- `HudHost.get_anchor_rect(id)` 与 `get_protected_rects()` 返回**窗口像素**；引导在边界处统一转换到 `TutorialLayer` 局部坐标：
  ```gdscript
  func _to_local(rect: Rect2) -> Rect2:
      return get_global_transform_with_canvas().affine_inverse() * rect
  ```
  逆缩放模式下该变换近似恒等；设计单位降级模式下它负责换算。**禁止**在 `BubbleLayout` 内部做坐标转换（纯函数只接收已转换的矩形）。
- 锚点每次展示前重新取，**不缓存**；窗口缩放、150% 字号、超宽屏都由 HUD 重新布局（GDD UI Requirements）。

#### D5.2 锚点使用表

| Beat | 锚点 | 用途 |
|---|---|---|
| `B0_WELCOME` | `top:net` | 决定气泡上方/下方与指引线 |
| `B1_TWO_NUMBERS` | `card:<焦点菜>`；无焦点时 `grid:first_card` | 决定气泡位置；焦点变化时重贴 |
| `B2_CAPACITY` | `bar:capacity` | **仅记录讲解对象**；非模态提示按 D5.4 固定停靠 |
| `B3_CLOYING` | `card:<该菜>` + `rail:cloy` | `card` 为讲解对象；`rail` 只用于定位腻度栏，不参与气泡位置 |
| `B4_RECEIPT` | 无 | 收据屏由界面流程摆放 |

> B2/B3 的锚点与「非模态提示固定在详情行上方」在 GDD 里同时存在（规则 5 vs Visual/Audio），本 ADR 取**视觉规格优先**：上下文提示一律停靠，锚点只用于标注讲解对象与降级判定（见 GDD Sync Check 第 7、8 条）。

#### D5.3 `BubbleLayout`

```gdscript
class BubbleLayout extends RefCounted

enum Mode { ABOVE, BELOW, DOCKED, HIDDEN }

class Placement extends RefCounted
var rect: Rect2
var mode: int
var anchor: Rect2

static func compute(anchor: Rect2, viewport: Vector2i,
                    metrics: HudLayoutMetrics, config: OnboardingConfig,
                    line_count: int, contextual: bool) -> Placement
```

固定公式（GDD `bubble_rect`）：

```
m  = metrics.safe_margin
W  = viewport.x; H = viewport.y
w  = clamp(round(config.width_ratio * W), config.width_min_px, config.width_max_px)
w  = min(w, W - 2*m)
h  = line_count * metrics.line_height + 2*config.padding_px

# 上下文提示与「锚点不可用」：直接停靠
contextual_dock_y = metrics.detail_row_top - config.gap_px - h

# 模态 beat 的首选位
x = clamp(round(anchor.center.x - w/2), m, W - m - w)
y = anchor.top - config.gap_px - h
if y < m or intersects(protected): y = anchor.bottom + config.gap_px
if y + h > H - m or intersects(protected): y = H - m - h
if still_intersects(protected) or y < m: mode = HIDDEN
```

- 所有矩形 `round` 到整数像素；同一输入必须得到逐像素相同的 `Rect2`（无头断言）。
- `protected` 来自 `HudHost.get_protected_rects()`，顺序固定 `[turns, capacity, net]`；任一为空按「不相交」处理并记一次 `ONBOARDING_INVARIANT`（GDD 公式的调试要求）。
- **`HIDDEN` 是最后手段**：在 < 480×320 的退化视口才可能出现。模态 beat 变 `HIDDEN` 时立即 `FINISH(SKIPPED)` 并记 ERROR，避免「有遮罩、无说明、无法推进」；上下文提示变 `HIDDEN` 时静默跳过，不阻塞。
- **150% 字号**：`metrics.line_height` / `font_px` 来自 HUD 的缩放后度量；气泡最多三行，文案上限 48 字（D6.2）。4K 上 `w` 封顶 440 px，不随分辨率变宽。

#### D5.4 上下文提示的停靠

- B2/B3 一律 `DOCKED`：`y = detail_row_top − gap − h`，再 `clamp` 到 `[m, H − m − h]`；`x` 与模态同公式。
- 不画指引线、不压暗、不使用 `Scrim`。
- 若 `detail_row_top` 不可得（HUD 降级），用 `H − m − 44*s` 作为保守值并记 WARNING；仍可能相交时按 D5.3 的 `HIDDEN` 处理（跳过提示，不阻塞）。

#### D5.5 遮罩补集

```gdscript
class ScrimPieces extends RefCounted
static func compute(viewport: Rect2, protected: Array[Rect2]) -> Array[Rect2]
```

- 从 `[viewport]` 开始，按 `[turns, capacity, net]` 顺序依次做矩形差（最多切成 4 块）；输出顺序固定：左、上、右、下。
- 受保护矩形先 `intersection(viewport)` 并取整；空矩形、`viewport` 外矩形直接忽略（并记一次断言失败）。
- `Scrim` 是一个 `Control`，`_draw` 逐块填充半透明深色；`mouse_filter = MOUSE_FILTER_IGNORE`。**三个受保护矩形内不绘制任何颜色**，因此读数保持原亮度与原对比度（AC 5、AC 15）。
- `Scrim` 只在 `INTRO` / `NUMBERS` 可见；`COACHING` / `CLOSING` 隐藏。
- `GuideLine` 是 1 px 线，仅在模态 beat 且 `mode ∈ {ABOVE, BELOW}` 时绘制；`DOCKED` / `HIDDEN` 不画。

### D6 — 呈现、文本表与无障碍

#### D6.1 视图状态

`TutorialLayer` 只有三个状态：

| 状态 | 遮罩 | 指引线 | 气泡 | 输入 |
|---|---|---|---|---|
| `hidden` | 无 | 无 | 隐藏 | 无（`_active = false` 或已完成） |
| `modal` | 补集遮罩 | 有（可定位时） | 显示 | `Blocked` + 白名单 |
| `contextual` | 无 | 无 | 停靠 | `Run` 原样 |

- 淡入 ≤ 100 ms、淡出 ≤ 80 ms；`reduce_motion = true` 时两者为 0。`Tween` 只改 `modulate.a`，不参与状态判定，不延迟隐藏或推进。
- 打字机、逐字动画、自动滚动、自动关闭**一律禁止**。
- 气泡是单实例；同一时刻最多一条。B3 替换 B2 时不重建节点，只换文本、行数与锚点。

#### D6.2 文本表

```gdscript
class_name OnboardingTextTable extends Resource
@export var labels: Dictionary = {}     # key -> {full: String, short: String}
func text(key: StringName, args: Dictionary, prefer_short: bool) -> String
```

- 键按 `beat_id + 分支` 组织：`b0.full` / `b0.short` / `b0.no_fee_override` / `b1.full` / `b1.short` / `b1.no_focus` / `b2.full` / `b2.short` / `b2.no_pool_override` / `b3.full` / `b3.short` / `skip.label` / `a11y.hint` / `b4.*`（headline / summary / fee_note / replacement 五分支 / upgrade / empty / degraded）。
- 允许的占位符是**封闭集合**：`{entry_fee}`、`{capacity}`、`{turns}`、`{dish_count}`、`{wave_count}`、`{visible_count}`、`{group_name}`、`{extra}`、`{remaining}`、`{satiety}`、`{upgrade_name}`、`{upgrade_price}`、`{wallet}`、`{lowest_value}`、`{best_value}`、`{uv_gap}`。未知占位符或缺少参数即 lint 失败。
- **模板里不得出现数字字面量**（`0-9`）；数字一律来自上游读数或预置表（GDD 规则 20）。
- **局内文本禁止** `元/格`、`每格`、`unit_value`、任何比值；单位价值只出现在 `b4.*`。
- **禁用措辞**：`最亏`、`白吃`、`浪费`、`下次先拿`；任何分支命中即 lint 失败（GDD `replacement_hint`）。
- **长度**：按 150% 字号的最坏情况验收，单条 ≤ 48 字（CJK 按 1 em 计，`3 × floor(440/27) = 48`）。超限使用 `short` 版本；仍超限则**生成期 lint 失败**，运行时不截断、不缩到 16 px 以下。
- 缺条目：先回退 `short`，再缺则 `TEXT_MISSING` → 该 beat 降级并从 `required` 移除；缺 B0/B1 文案时整段引导以 `SKIPPED` 收尾并记 ERROR，气泡缺文案不得卡住玩家（GDD 边界用例）。

#### D6.3 无障碍与焦点

- 引导**不设焦点、不移焦点、不清焦点**，只读 HUD 的 `focused_dish_id()`；`B0` 的导航动作不在白名单，焦点不动；`B1` 的导航归 HUD。
- 跳过按钮 `focus_mode = FOCUS_NONE`，`ui_cancel` 是键盘/手柄的等价入口；按钮文本参与读屏顺序。
- 颜色不是唯一通道：跳过按钮有文字；模态遮罩之外的读数不被压暗；气泡正文对比度复用 HUD 的 `hud_contrast_ratio ≥ 4.5:1`。
- 150% 字号下气泡最多三行、不溢出、不遮挡受保护矩形；1920×1080 与超宽屏各留截图（AC 25）。

### D7 — `FirstRunExplanation` 与收据交付（TR-004）

#### D7.1 两个时点

| 时点 | 做什么 | 不做什么 |
|---|---|---|
| `run_settled(run_id, result)` | 深拷贝 `result` 与 `MenuWaves.get_run_menu()`，记 `run_id`；`BeatMachine` → `CLOSING` | 不读钱包、不读升级目录、不构建讲解、不发完成信号 |
| `on_flow_state(Run, Menu)`（ADR-0004 D8 第 5 步） | 结算事务（第 2 步）已提交；构建 `FirstRunExplanation`；发 `explanation_ready`；发 `onboarding_finished(ALL_BEATS)` 一次；卸载引导 | 不写存档、不购买升级、不高亮按钮 |

- 为什么不在 `run_settled` 构建：`MetaUpgrades.prepare_run_result` 在 `Run → Menu` 第 2 步才把本局净利加进钱包（ADR-0007 D3）；提前读钱包会把「首笔升级买得起」算错。
- 为什么仍在同一帧：第 5 步发生在收据 `present()` 之后、该帧绘制之前；`UIRoot` 把 `explanation_ready` 转给已挂载的 `ReceiptScreen.attach_explanation()`，玩家看到的仍是同帧收据。
- 若 `Run → Menu` 被拒绝（同帧暂停先到），`RunFlow` 在 `Handoff` 重发；引导保持 `CLOSING`，stashed result 不丢。若玩家在 `Handoff` 退出 → `run_discarded` → `CANCELLED`，不交付、不发完成信号。

#### D7.2 payload 字段

```gdscript
class FirstRunExplanation extends RefCounted

enum ReplacementKind { EMPTY_RUN, ALL_EATEN, DID_NOT_FIT, HIGHER_UV_DIFFERENT_CELLS, ALREADY_BEST }

class Row extends RefCounted
var dish_id: StringName
var name: String
var price: int
var satiety: int
var unit_value: float
var effective_unit_value: float
var is_lowest_effective: bool

class Replacement extends RefCounted
var kind: int
var lowest_row_index: int          # -1 表示空局/无行
var best_menu_dish_id: StringName  # &"" 表示无
var best_menu_name: String
var uv_gap: float
var remaining_at_end: int
var best_menu_satiety: int
var best_menu_fit: bool

class UpgradeHint extends RefCounted
var upgrade_id: StringName
var name: String
var price: int
var affordable: bool

var run_id: int
var entry_fee: int
var official_entry_fee: int
var fee_is_override: bool
var headline: String               # 两段短文案，已由文本表解析
var summary: String
var fee_note: String               # fee_is_override 时说明 40 与 85；否则空串
var rows: Array[Row]               # 按吃下顺序
var replacement: Replacement
var replacement_text: String
var first_upgrade: UpgradeHint     # 可为 null
var upgrade_text: String
var degraded: bool                 # 单位价值或升级目录缺失
```

- **字符串只用于展示**：`FirstRunExplanation` 不进入存档、不进入 `RunResult`、不参与任何玩法判定；收据屏只按字段排版，不重算。
- **不包含菜名到 `RunResult`**：菜名只在本 payload 里由 `DishValues.get_dish` 解析，`RunResult` 仍只有数值事实（ADR-0015 D3）。
- **不含最优解/总分**：没有 `opt_value`、没有「替换后净利」、没有推荐顺序；只有 GDD 允许的每格比较与一处对照建议。

#### D7.3 `ExplanationBuilder`

```gdscript
class ExplanationBuilder extends RefCounted
static func build(input: ExplanationInput, text: OnboardingTextTable) -> FirstRunExplanation
```

`ExplanationInput` 是纯数据：`run_id`、`entry_fee`、`official_entry_fee`、`rows`（深拷贝）、`menu_facts`（`{dish_id, name, unit_value, satiety}`）、`eaten_ids`、`capacity_total`、`capacity_used`、`wallet`、`catalog`（`{id, name, price, current_level, max_level}`）、`values_ok`。

固定算法：

```
lowest = argmin_{row ∈ rows} row.effective_unit_value
         # 并列：dish_id 字典序升序；同一道菜重复出现时取先吃下的行
skipped = menu_facts \ eaten_ids
best    = argmax_{d ∈ skipped} d.unit_value
          # 用 DishValues.uv_cmp 比较；并列取 dish_id 升序
uv_gap  = best.unit_value − lowest.effective_unit_value
remaining_at_end = capacity_total − capacity_used
best_menu_fit = best.satiety ≤ remaining_at_end
```

分支顺序**严格按 GDD**：

| 顺序 | 条件 | `ReplacementKind` |
|---|---|---|
| 1 | `rows = ∅` | `EMPTY_RUN`：只说「这一局你没有吃」，不给建议、不给高亮 |
| 2 | `skipped = ∅` | `ALL_EATEN`：「这一桌你都吃了，没有可换的」 |
| 3 | `¬best_menu_fit` | `DID_NOT_FIT`：写明「当时只剩 R 格，它需要 S 格——装不下；放弃是容量决定的，不是选错」 |
| 4 | `best_menu_fit ∧ uv_gap > 0.5` | `HIGHER_UV_DIFFERENT_CELLS`：每格更高，但格数不同，不能等价替换，也不给替换后总分 |
| 5 | `best_menu_fit ∧ uv_gap ≤ 0.5` | `ALREADY_BEST`：「你吃到的每格已经接近或更高，选对了」 |

- `uv_gap` 用 `float64` 计算，比较阈值 `0.5` 时加 `1e-9` 容差；展示固定 2 位小数。
- 任一 `DishValues` 查询失败 → `values_ok = false`：rows 仍给（`RunResult` 已含值），但 `replacement_text = ""`、`best_menu` 为空、`degraded = true`；收据只显示逐行值（GDD 边界用例）。
- **首笔升级**：从 `catalog` 中取 `current_level < max_level` 且 `price` 最小的一项；并列取 `upgrade_id` 升序；`affordable = wallet ≥ price`。只出文字，不购买、不高亮、不预选（GDD Open Question 7）。
- **教学价说明**：`entry_fee != official_entry_fee` 时 `fee_is_override = true`，`fee_note` 写「首局入场费是教学价 {entry_fee} 元，正式食堂是 {official_entry_fee} 元」，避免玩家把首局回本率当基准。
- 禁用措辞（最亏/白吃/浪费/下次先拿）在文本表 lint 与 builder 输出断言两处检查。

#### D7.4 交付接口

`OnboardingHost` 发：

```gdscript
signal explanation_ready(run_id: int, explanation: FirstRunExplanation)
signal onboarding_finished(reason: int)   # ALL_BEATS / SKIPPED
func get_explanation() -> FirstRunExplanation   # 只读，供测试与断线重取
```

- `UIRoot` 订阅 `explanation_ready`，转给 `ReceiptScreen.attach_explanation(explanation)`；收据屏负责排版、滚动、折叠，**不重算**任何值。
- `explanation_ready` 对同一 `run_id` 至多一次；`onboarding_finished` 每局至多一次。重复调用返回已缓存的 explanation，不再发信号。
- 跳过、放弃、故障、`init_failed`：不产出 explanation、不发 `explanation_ready`、不发 `onboarding_finished`（GDD 规则 17、AC 20、AC 22）。

### D8 — 生命周期与状态映射

`OnboardingHost` 的 Phase 与 `run_state` 正交；它只读 `run_state` 判断是否还该画。

| `run_state` | 引导 Phase | 行为 |
|---|---|---|
| `Initializing` / `Failed` | `IDLE` | 不激活；`run_faulted(INIT)` 只清理 |
| `Active` 且 `plan.is_first_run` | `INTRO` → `NUMBERS` → `COACHING` | 按 D3 推进 |
| `Active` 且 `plan.is_first_run = false` | `IDLE` | 零上下文、零绘制、零信号 |
| `Active` + `Paused` | 保持不变 | 气泡保留但 `_process` 停止；栈由 `SceneFlow` 清空 |
| `Ending` | `CLOSING` | 隐藏气泡/遮罩；等待 `run_settled` |
| `Settling` / `Handoff` | `CLOSING` | 不绘制；stashed result 保留到 `Run → Menu` 或丢弃 |
| `Faulted` | `CANCELLED` | 立即清理；不产出 B4 |
| `Menu` | `CLOSED` / `CANCELLED` / `IDLE` | 卸载子视图；`explanation_ready` 已交付 |

**挂载/卸载**：

- `AppRoot` 在 `Boot` 构造 `OnboardingHost`（layer 15）并 `attach(...)`，但 `_active = false`；`TutorialLayer.visible = false`。
- `Menu → Run` 接受后，`UIRoot` 发 `run_config_assembled(config, plan)`；`AppRoot` 调 `onboarding.configure(plan)`。若 `plan.is_first_run`，等待 `run_started` 与/或 `on_flow_state(Menu, Run)` 同帧激活（两者任一先到都能启动，见 D2.3）。
- `Run → Menu` 第 5 步交付后 `_active = false`、`TutorialLayer.visible = false`；节点常驻，不 `queue_free()`，避免重开一局时重建。
- `run_discarded` / `run_faulted` 立即隐藏并清空 stashed result；`Run` 结束后不保留任何本局对象（除了只读的 explanation 用于测试）。
- **`sync_for_state` vs `on_flow_state`**：ADR-0004 D8 第 5 步已调用 `onboarding.on_flow_state`；本 ADR 不新增第二个生命周期入口，挂载由 `configure(plan)` 完成（GDD Sync Check 第 10 条）。

### D9 — 只读数据流与 `OnboardingSource`

```gdscript
class OnboardingSource extends RefCounted

func _init(run_flow: RunFlow, hud: HudHost, menu_waves: MenuWaves,
           cloying: CloyingAndTaste, dishes: DishValues,
           meta: MetaUpgrades, text: OnboardingTextTable,
           plan: FirstRunRunPlan) -> void

func facts(caller: StringName) -> OnboardingFacts
func focused_dish_id() -> StringName
func group_count(dish_id: StringName) -> int
func stash_result(result: RunResult) -> void
func build_explanation_input() -> ExplanationInput
```

**每次事件只读一次**：

- `run_started` / `action_committed` / `run_ended`：`run_flow.snapshot()` 一次，取 `run_state`、`turns_remaining`、`turns_total`、`capacity_used`、`capacity_total`、`eaten_count`、`action_seq`。
- B0 文案：`turns_total`、`capacity_total`、`plan.pool_override.size()` 或 `menu_waves.get_run_menu().size()`（池覆盖降级时）、`menu_waves.wave_count()`。
- B2 文案：`capacity_total`、`visible_dishes().size()`、`wave_count()`。
- B3：只在 `COACHING` 且 `focused_dish_id()` 变化时调一次 `group_counts()`；找到该菜的 `GroupCount.count`，不重算腻度、不移动焦点。
- `run_settled`：`stash_result` 深拷贝 `RunResult` 与 `get_run_menu()`。
- `Run → Menu`：`build_explanation_input()` 读一次 `meta.get_wallet()`、`meta.get_catalog(station_id)`、`meta.next_price(id)`；这是本系统唯一在交付时读钱包的地方，且只读。
- **不读**：`SaveSystem`、`LoadReport`、`RecordsSnapshot`、`InputEvent`、系统时钟、RNG、`get_node("/root/…")`。

**焦点轮询的边界**：`focused_dish_id()` 只在 `COACHING` 每帧读一次（廉价投影），且只在 id 变化时读 `group_counts()`；INTRO/NUMBERS 不读焦点（B0 不需要，B1 只在 accept 时读一次）。这满足 GDD「不缓存第二份计数表」且不会把 HUD 的焦点模型复制过来。

### D10 — 降级阶梯（TR-005）

严格按 GDD 规则 19 的顺序，且每一级都**不阻止开局**：

| # | 缺失 | 行为 | 记录 |
|---|---|---|---|
| 1 | 预置内容不变式失败 | `plan.is_first_run = false`、不激活引导、菜单显示档口价、本局正常开始 | ERROR + CI 失败 |
| 2 | `LoadReport.first_run_eligible` 缺失 | 按 `outcome ∈ {Fresh, Loaded, Migrated}` 派生；`Recovered` 仍为否 | WARNING（调试断言字段值） |
| 3 | `entry_fee_override` 不被支持 | `menu_entry_fee = official_entry_fee`；B0 省略「教学价」句；B4 无 `fee_note` | WARNING |
| 4 | `pool_override` 被忽略 | 用实际菜池；B2 省略「六道菜 / 20 格」定量句 | WARNING |
| 5 | HUD 锚点为空（B0/B1） | 模态气泡先隐藏，最多等 2 帧；仍空则底部停靠；B2/B3 直接停靠 | WARNING |
| 6 | `HudLayoutMetrics` 不可得 | 用 `OnboardingConfig` 的保守值（`m = clamp(0.05H, 32, 96)`、`line_height = round(22*s)`、`font_px = round(18*s)`、`detail_row_top = H − m − 44*s`） | WARNING |
| 7 | 单位价值/菜名查询失败 | B4 只给文案与逐行静态值；`replacement_text = ""`；`degraded = true` | WARNING |
| 8 | 升级目录不可用 | `first_upgrade = null`、`upgrade_text = ""` | WARNING |
| 9 | 气泡文案缺失 | `TEXT_MISSING`：该 beat 移出 `required()`；B0/B1 缺失则整段引导 `SKIPPED` | ERROR |
| 10 | 无安全气泡位置（退化视口） | 模态 → `SKIPPED` + ERROR；上下文 → 静默跳过 | ERROR |
| 11 | 暂停后重压 `Blocked` 失败 | 当前 beat 降级为无遮罩停靠提示；玩家可正常操作 | WARNING |
| 12 | `group_counts()` 不可用 | B3 不触发、不进入 `required()`；其余 beat 不变 | WARNING |

**底线**：任何一级失败都不能让 `run_state` 停在非 `Active`、不能扣回合/容量、不能写存档、不能让玩家「按键无效」。

### D11 — 可测性与静态检查

#### D11.1 纯模型

以下类不引用 `Node` / `SceneTree`，可在 GdUnit 无渲染环境实例化：

- `FirstRunGate`、`FirstRunPreset`、`FirstRunRunPlan`、`FirstRunPlanner`
- `BeatMachine`、`BeatIntent`、`BeatFacts`
- `BubbleLayout`、`ScrimPieces`、`OnboardingTextTable`、`ExplanationBuilder`、`FirstRunExplanation`
- `OnboardingInvariants`（见 D11.2）

`OnboardingHost` 只做适配：`configure`、事件转发、`_process` 输入拉取、视图应用、信号发出。`TutorialViewFactory` 允许测试注入 stub 视图。

#### D11.2 运行时不变式（调试构建断言 + 正式构建降级）

| # | 不变式 | 失败处理 |
|---|---|---|
| 1 | 模态 beat 的 `allow` 只含 `ui_accept` / `ui_cancel`（B0）或再加四个导航（B1）；`ui_pause` 不在 `allow` 里也被隐式允许 | 拒绝压栈，`DEGRADE_BEAT` |
| 2 | `INTRO` / `NUMBERS` 期间 `_blocked_owned == true`；`COACHING` / `CLOSING` 期间为 `false` | 记 `ONBOARDING_INVARIANT`；按实际栈修正 |
| 3 | 气泡矩形与三个受保护矩形不相交，或 `mode == HIDDEN` | 翻面 → 停靠 → 隐藏；正式构建不绘制相交气泡 |
| 4 | `ScrimPieces` 的并集 + 受保护矩形 = 视口；受保护区域内无绘制 | 丢弃本帧遮罩，改为整屏半透明并记 ERROR（读数仍可读） |
| 5 | `onboarding_finished` 至多一次；`explanation_ready` 每 `run_id` 至多一次 | 后续调用不生效并记 ERROR |
| 6 | `required ⊆ shown` 才发 `ALL_BEATS`；否则不发完成 | 保持 `CLOSING`，等缺失 beat 或降级 |
| 7 | 本帧所有读数引用同一个 `run_id` / `action_seq` | 不采纳本帧文案，用上一帧合法值 |

#### D11.3 静态检查

| 规则 | 扫描目标 | 失败含义 |
|---|---|---|
| 无存档/纪录读 | `src/presentation/onboarding/` 不出现 `save.`、`LoadReport`、`RecordsSnapshot`、`first_run_eligible` | 引导越权决定首局 |
| 无持久化写 | 不出现 `commit(`、`mark_dirty(`、`prepare_run`、`purchase(`、`spend(` | 引导写状态 |
| 无规则写 | 不出现 `spend_turn(`、`consume(`、`advance(`、`end_pickup(`、`submit_action(`、`set_frozen(` | 引导改玩法 |
| 无计时/异步 | 不出现 `Timer`、`create_timer`、`await`、`call_deferred(`、`tween_callback` | 自动推进 |
| 无原始事件 | 不出现 `InputEvent`、`_input(`、`_unhandled_input(` | 绕过 `InputRouter` |
| 无 `is_action_just_pressed` | `onboarding` 目录只调用 `consume_action` / `get_context_snapshot` | 与 HUD 抢动作 |
| 上下文唯一 | `push_context(` / `pop_context(` 只出现在 `onboarding_host.gd`，且只以 `&"Blocked"` 调用 | 误开 Modal |
| 无 UI 单位价值 | `onboarding` 目录（除 `explanation_builder.gd` 与文本表资源）不出现 `unit_value`、`每格`、`元/格` | 局内泄题 |
| 无中文 UI 字面量 | `.gd` 不出现中文字符串；全部走 `OnboardingTextTable` | 文案散落 |
| 无锚点缓存 | `get_anchor_rect(` / `get_protected_rects(` 只在展示路径调用，不写入成员变量 | 窗口缩放后错位 |

## Architecture Diagram

```
                         AppRoot (Application)
        ┌───────────────────────────┴────────────────────────────┐
        │ FirstRunPlanner                                         │
        │   FirstRunGate ── LoadReport / RecordsSnapshot           │
        │   FirstRunPreset ── DataConfig / DishValues / Cloying    │
        │   FirstRunRunPlan ──▶ UIRoot (menu fee + RunConfig)      │
        │                    ──▶ OnboardingHost.configure(plan)    │
        └───────────────────────────┬────────────────────────────┘
                                    │ plan
┌───────────────────── OnboardingHost (CanvasLayer, layer 15, INHERIT) ─────────────────────┐
│  configure(plan)  on_run_started / action_committed / run_ended / run_settled             │
│  on_run_faulted / run_discarded  on_flow_state(from,to)  _process(priority -50)           │
│                                                                                           │
│  ┌──── BeatMachine (pure) ────┐        ┌──── OnboardingSource (read-only) ────┐          │
│  │ phase / event / intent     │        │ RunFlow.snapshot() (once/event)      │          │
│  │ input_seq guard            │◀───────│ HudHost.anchor/protected/focus       │          │
│  │ required / shown           │        │ MenuWaves.visible / wave_count       │          │
│  └────────────┬───────────────┘        │ Cloying.group_counts() (on change)   │          │
│               │ intents                │ DishValues (B4)                      │          │
│               ▼                        │ MetaUpgrades (wallet/catalog, B4)    │          │
│  ┌──── TutorialLayer (Control, inverse scale) ────────────────────────────────┐          │
│  │  Scrim (complement of 3 protected rects, MOUSE_FILTER_IGNORE)              │          │
│  │  GuideLine (modal only)                                                    │          │
│  │  Bubble (PanelContainer) ── Body + SkipButton (FOCUS_NONE)                 │          │
│  └────────────────────────────────────────────────────────────────────────────┘          │
│  signals: onboarding_beat_shown / onboarding_finished / explanation_ready                 │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                 │ push_context(Blocked, allow) / pop_context()
                 ▼
            InputRouter  ──▶  HUD (layer 10) / RunFlow (submit_action)
                 │
                 ▼
     Run → Menu 第 2 步: SettlementTransaction.apply (唯一 commit)
     Run → Menu 第 5 步: ExplanationBuilder.build → explanation_ready → ReceiptScreen
```

## Key Interfaces

```gdscript
# res://src/application/onboarding/

class_name FirstRunPlanner extends RefCounted
func _init(load_report: LoadReport, records: RecordsSnapshot, data: DataConfig,
           dishes: DishValues, meta: MetaUpgrades, station_id: StringName,
           make_cloying: Callable) -> void
func refresh(reason: int) -> FirstRunRunPlan
func plan() -> FirstRunRunPlan
func preset_view() -> FirstRunPresetView

class FirstRunPresetView extends RefCounted
var pool_override: Array[StringName]
var entry_fee_override: int

class FirstRunGate extends RefCounted
static func evaluate(load_report: LoadReport, records: RecordsSnapshot,
                     preset_valid: bool) -> Decision

class FirstRunPreset extends RefCounted
static func validate(data: DataConfig, dishes: DishValues,
                     ctx: Context, make_cloying: Callable) -> Validation

class FirstRunRunPlan extends RefCounted
func has_entry_fee_override() -> bool
func run_config_fields() -> Dictionary


# res://src/presentation/onboarding/

class_name OnboardingHost extends CanvasLayer

signal onboarding_beat_shown(beat_id: StringName)
signal onboarding_finished(reason: int)
signal explanation_ready(run_id: int, explanation: FirstRunExplanation)

func attach(source: OnboardingSource, input: InputRouter,
            text: OnboardingTextTable, config: OnboardingConfig,
            factory: TutorialViewFactory) -> void
func configure(plan: FirstRunRunPlan) -> void

func on_run_started(run_id: int, station_id: StringName, seed: int) -> void
func on_action_committed(action: int, payload: Dictionary) -> void
func on_run_ended(run_id: int, end_reason: int) -> void
func on_run_settled(run_id: int, result: RunResult) -> void
func on_run_faulted(run_id: int, code: StringName, detail: Dictionary) -> void
func on_run_discarded(run_id: int) -> void
func on_flow_state(from: int, to: int) -> void

func is_active() -> bool
func current_beat() -> int
func current_bubble_rect() -> Rect2
func get_explanation() -> FirstRunExplanation

class BeatMachine extends RefCounted
func on(event: int, facts: Facts) -> Array[Intent]
func required() -> Array[int]
func shown() -> Array[int]
func phase() -> int

class BubbleLayout extends RefCounted
static func compute(anchor: Rect2, viewport: Vector2i,
                    metrics: HudLayoutMetrics, config: OnboardingConfig,
                    line_count: int, contextual: bool) -> Placement

class ScrimPieces extends RefCounted
static func compute(viewport: Rect2, protected: Array[Rect2]) -> Array[Rect2]

class ExplanationBuilder extends RefCounted
static func build(input: ExplanationInput, text: OnboardingTextTable) -> FirstRunExplanation

class OnboardingSource extends RefCounted
func facts(caller: StringName) -> OnboardingFacts
func focused_dish_id() -> StringName
func group_count(dish_id: StringName) -> int
func stash_result(result: RunResult) -> void
func build_explanation_input() -> ExplanationInput
```

**HUD 侧新增的只读接口**（生产者归 ADR-0019，本 ADR 只定义消费面）：

```gdscript
class HudLayoutMetrics extends RefCounted
var safe_margin: int
var font_px: int
var line_height: int
var detail_row_top: int

# HudHost
func get_layout_metrics() -> HudLayoutMetrics
```

**`FirstRunPlanner` 依赖的只读上游形状**（其中两处尚未在既有 ADR 定义，见 GDD Sync Check）：

```gdscript
class GroupCount extends RefCounted        # ADR-0014 待补
var group: StringName
var count: int

class UpgradeView extends RefCounted       # ADR-0016 待补
var id: StringName
var name: String
var current_level: int
var max_level: int
```

## Implementation Guidelines

1. **必须**让 `FirstRunPlanner` 是 `is_first_run`、`first_run_eligible`、菜单入场费与 `RunConfig` 预置的唯一生产者；**禁止** `UIRoot`、`MainMenu`、`OnboardingHost` 各自推导。
2. **必须**在 `Run → Menu` 第 2 步（结算事务）完成后再读钱包与升级目录；**禁止**在 `run_settled` 构造 `FirstRunExplanation`。
3. **必须**让 `OnboardingHost` 在 `run_started` / `Menu → Run` 的同一帧完成「配置 + 压 `Blocked` + 显示 B0」；**禁止**等到下一帧。
4. **必须**在 `INTRO` / `NUMBERS` 的每次轮询同时取走 `ui_cancel` 与 `ui_accept`，cancel 优先；**禁止**只取其中一个。
5. **必须**用 `input_seq` 做「同一帧只推进一个 beat」的闸；**禁止**用 `Engine.get_process_frames()` 或节点顺序代替。
6. **必须**让 `COACHING` / `CLOSING` 不调用 `consume_action`、不压/弹上下文；**禁止**把上下文提示做成第二层 `Blocked`。
7. **必须**在 `COACHING` 只在焦点 id 变化时读 `group_counts()`；**禁止**每帧读腻度全表、缓存第二份计数、移动焦点。
8. **必须**让 B2/B3 走 D5.4 的底部停靠；**禁止**用锚点位置覆盖视觉规格。
9. **必须**让 `ScrimPieces` 是受保护矩形的补集；**禁止**整屏压暗后只在读数上重绘。
10. **必须**让 `BubbleLayout` 在相交时依次「翻面 → 停靠 → 隐藏」；**禁止**让气泡压住回合/容量/净利。
11. **必须**让模态气泡消失后立即 `pop_context()`；**禁止**在 `COACHING` 继续持有 `Blocked`。
12. **必须**在暂停恢复后用**同一白名单**重压；重压失败立即降级为无遮罩提示；**禁止**重试或把玩家留在「泡泡在、按键无效」。
13. **必须**让跳过按钮与 `ui_cancel` 走同一条 `_skip()` 路径；**禁止**二次确认或写存档。
14. **必须**让 `onboarding_finished` / `explanation_ready` 各自每局至多一次；**禁止**在放弃/故障路径发完成。
15. **必须**让全部玩家可见文本经 `OnboardingTextTable`；**禁止**在 `.gd` 写中文字面量、在模板写数字、在局内写单位价值。
16. **必须**让 `FirstRunExplanation` 只用于展示；**禁止**把字符串、菜名或替换建议写入 `RunResult` / 存档 / `RunFlow` 快照。
17. **必须**让 `B4` 的替换建议严格按五分支顺序；**禁止**给「替换后总分」「最优解」「推荐排序」。
18. **必须**让任一级降级都不改变 `run_state`、不扣回合/容量、不写存档；**禁止**用阻塞式错误框替代降级。

## Alternatives Considered

### Alternative 1: 引导做成 `UIRoot` 的模态屏幕

- **Description**：`push_context(Modal)` + `ScreenHost` 挂一个 `TutorialScreen`。
- **Pros**：复用屏幕栈、焦点陷阱与遮罩。
- **Cons**：`Run` 的 `ui_stack_depth` 会变成 1，违反 `ui-flow.md` 状态表；`Modal` 会向 `ActionSink` 投递 `ui_accept`，B1 的确认会被界面消费；HUD 会被屏幕遮住，玩家看不到被讲解的数字。
- **Rejection Reason**：引导是 `Run` 内的覆盖层，不是界面流程的模态。

### Alternative 2: 不压 `Blocked`，让 B1 的确认照常到达 HUD

- **Description**：引导只画气泡，玩家按 `ui_accept` 时正常吃菜，吃完再推进。
- **Pros**：实现最简单，不需要 `consume_action`。
- **Cons**：B1 的「只推进、不吃菜」失败（AC 8）；玩家可能在没读完前就吃掉了被讲解的菜；GDD 规则 6 明确禁止。
- **Rejection Reason**：教学与动作会互相污染，且无法在无头测试中断言「餐局流程没收到确认」。

### Alternative 3: 引导自己解析锚点、自己算布局

- **Description**：引导读 HUD 的主题与卡片节点，自己找目标位置。
- **Pros**：不依赖 HUD 的公开接口。
- **Cons**：两个布局实现会漂移；150% 字号、超宽屏、逆缩放三处都要各改一遍；违反 ADR-0019 的「HUD 拥有锚点」边界。
- **Rejection Reason**：锚点矩形是既有的只读契约，引导只应拥有「放在上方还是下方」。

### Alternative 4: 在 `run_settled` 直接构建 `FirstRunExplanation`

- **Description**：收到结果就构建并交付，不等 `Run → Menu`。
- **Pros**：与 GDD 状态表的字面顺序一致。
- **Cons**：钱包尚未入账，首笔升级可负担性会读旧值；`Run → Menu` 若被拒绝或玩家在 `Handoff` 退出，讲解会被错误交付；违反 ADR-0007「`run_settled` 只预通知」。
- **Rejection Reason**：交付必须与唯一结算提交点对齐；GDD 的 `run_settled` 应读作「结果就绪」，不是「持久化完成」。

### Alternative 5: 持久化「引导已看」

- **Description**：在 `onboarding` 命名空间写 `finished = true`，下次不再播放。
- **Pros**：不受 `runs_settled` 与备份恢复影响。
- **Cons**：GDD 明确不写持久化；新增命名空间要 schema 迁移；「跳过、再放弃」会让玩家永远看不到教学，与规则 22 的「下一局仍是首局」冲突。
- **Rejection Reason**：首局身份已由 `runs_settled == 0 ∧ first_run_eligible` 唯一决定。

## Consequences

### Positive

- 首局身份、菜单费、`RunConfig` 预置三者同源，40/85 不可能分叉；`Recovered` 例外有唯一实现。
- beat 推进与输入取走是 `input_seq` + `process_priority` 的确定组合，不依赖节点树顺序；「同帧两次确认只推进一次」可无头断言。
- 气泡几何、遮罩补集、锚点降级全是纯函数，720p / 1080p / 4K / 150% 可以逐像素测试，不必靠截图发现 1 px 漂移。
- 收据讲解在结算提交后、首次绘制前构建，既读到正确钱包，又不晚一帧。
- 全部降级路径都不阻塞开局，预置损坏只会让首局退化为普通局并让 CI 变红。

### Negative

- `FirstRunPlanner` 需要同时读存档、纪录、数据、升级目录，初始化接线比「引导自己判断」多一层；这是为唯一真相付出的显式成本。
- `OnboardingHost` 需要与 HUD 共用逆缩放和坐标转换，两个 `CanvasLayer` 的层序/变换必须一起验证。
- `FirstRunExplanation` 是字符串与数值混合的展示 payload，跨层传递比纯数值契约更宽；用「不进入存档、不参与判定」的静态检查约束。
- 文本表 + lint 的维护成本高于把文案写在脚本里；这是 48 字上限与本地化扩写的必要成本。

### Neutral

- B2/B3 的锚点在 GDD 里有两种读法；本 ADR 取「视觉规格优先、上下文提示固定停靠」，锚点只用于标注讲解对象。
- 跳过按钮不做键盘焦点项，`ui_cancel` 是键盘/手柄的等价入口；AC 27 的「或」由此满足。
- 音效反馈 ADR 尚未定义引导 cue；当前静音是合法降级，不影响本 ADR 的任何状态。

## Risks

- **`process_priority` 顺序被改。** 若 HUD 或引导的优先级被改成相同值，同帧 `ui_accept` 可能先被 HUD 读到。缓解：启动断言 `SceneFlow(−100) < Onboarding(−50) < HudHost(0)`；GDD Sync Check 第 12 条要求 ADR-0005 登记该常量。
- **`consume_action` 只取 accept、不取 cancel。** 若实现漏取 cancel，Esc 会在同一帧被 HUD 当成别的动作（或残留到下一帧）。缓解：D4.2 固定「两个都取、cancel 优先」，并用「HUD 读不到 accept」与「cancel 关闭」两条测试夹住。
- **暂停恢复后重压失败导致死锁。** 缓解：`CONTEXT_RESTORE_FAILED` 必须降级为无遮罩停靠，且 `_blocked_owned` 以 `push_context` 返回值为唯一真相；AC 19 用假 `InputRouter` 强制返回 `false`。
- **HUD 锚点与引导坐标空间不一致。** 逆缩放 vs 设计单位两种模式下 `get_global_transform_with_canvas()` 可能不同。缓解：Verification ②；`BubbleLayout` 只接收已转换矩形，转换只在边界做一次。
- **`Recovered` 判定被绕过。** 若 `FirstRunPlanner` 只看 `runs_settled`，老玩家会在备份恢复后重看教学。缓解：`FirstRunGate.evaluate` 的固定顺序 + AC 31 夹具 + 调试断言。
- **预置不变式 #4 与腻度实现漂移。** 若校验另写一份 `extra_satiety` 公式，改动腻度后会静默放过错误菜池。缓解：校验通过 `make_cloying()` 新建临时 `CloyingAndTaste` 重放，不复制公式。
- **B4 的替换建议被读成「推荐」。** 缓解：五分支文案明确「格数不同、不能等价替换、不给替换后总分」；禁用措辞 lint；`FirstRunExplanation` 无 `opt_value` / 推荐字段。
- **150% 字号下 B0 超三行。** 缓解：48 字上限在生成期 lint；`short` 版本兜底；仍超限则 lint 失败，不在运行时缩字号。
- **`FirstRunExplanation` 交付晚于收据首绘。** 缓解：`Run → Menu` 第 5 步与第 2 步同帧；`ReceiptScreen.attach_explanation()` 在首次绘制前调用；无头断言 `receipt_open_frame == explanation_frame`。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `first-run-onboarding.md` | `TR-first-run-onboarding-001`：只在 `RunConfig.is_first_run` 时运行；预置 `pool_override` 六道菜与 `entry_fee_override = 40` | D2：`FirstRunGate` 三态真值 + `Recovered` 例外；`FirstRunPreset` 六道菜/40 元与七条不变式；`FirstRunRunPlan` 同时供菜单费与 `RunConfig`；D10 的预置失败降级 |
| `first-run-onboarding.md` | `TR-first-run-onboarding-003`：锚点来自 HUD `get_anchor_rect`；气泡永不覆盖三个受保护矩形 | D5：六个锚点的使用表、`BubbleLayout` 的翻面/停靠/隐藏、`ScrimPieces` 补集、`get_protected_rects` 三元组；D11 不变式 3/4 与无头几何验收 |
| `first-run-onboarding.md` | `TR-first-run-onboarding-004`：收据 `FirstRunExplanation`；不写持久化；`onboarding_finished` 每局一次 | D7：两个时点、五分支替换建议、首笔升级、payload 字段；D8 生命周期；D11 静态检查禁止 `commit` / `save.`；`onboarding_finished` 幂等 |
| `first-run-onboarding.md` | `TR-first-run-onboarding-005`：缺预置/锚点/单位价值时的降级顺序；永不阻塞开局 | D10：12 级降级阶梯（预置 → 资格字段 → 费覆盖 → 池覆盖 → 锚点 → 度量 → 单位价值 → 升级目录 → 文案 → 安全位置 → 重压 → 组计数），每级都不改变 `run_state`、不扣回合/容量、不写存档 |

> `TR-first-run-onboarding-002`（固定 B0–B4 序列、一次输入一 beat、模态 beat 使用 `Blocked` 白名单）由 ADR-0005 覆盖；本 ADR 的 D3/D4 实现其消费侧（beat 机、拉模型、跳过与暂停重压），不重复认领。

## GDD Sync Check

| # | GDD / ADR 原文 | 本 ADR 的落地 | 需要谁对齐 |
|---|---|---|---|
| 1 | `save-system.md` 规则 26 定义 `LoadReport.first_run_eligible`；ADR-0002 的 `LoadReport` 字段表只列 `outcome` / `reason` / `warnings` / `writable` / `lock_live` / `source_version` / `source` | `FirstRunGate` 优先读字段；字段缺失时按 `outcome ∈ {Fresh, Loaded, Migrated}` 派生，`Recovered` 恒假。调试构建断言两者相等 | **已对齐**：ADR-0002 的 `LoadReport` 字段表已补 `first_run_eligible` 并写明计算式与 `Recovered` 钳制 |
| 2 | `ui-flow.md` 规则 8 / AC 47 用 `first_run_eligible ∧ runs_settled == 0`；**AC 6 仍写 `is_first_run = (career.runs_settled == 0)`** | 本 ADR 只实现规则 8 的判据；AC 6 的字面会导致 `Recovered` 重播教学，与 S11 / AC 31 冲突 | game-designer / ui-flow 修订 AC 6 |
| 3 | `first-run-onboarding.md` 头部 `Status: In Design`；`systems-index.md` 第 75 行标 `Approved` | 本 ADR 按 Approved 的 GDD 设计；状态头不一致会让 traceability 与 gate-check 各读一份 | 文档维护者同步头部状态 |
| 4 | ADR-0019 定义 `HudLayoutRects` 与 `HudTheme`，但没有公开安全边距 / 行高 / 字号 / 详情行顶边 | 本 ADR 消费 `HudLayoutMetrics { safe_margin, font_px, line_height, detail_row_top }`；缺失时用 `OnboardingConfig` 保守值并记 WARNING | **已对齐**：ADR-0019 Key Interfaces 已增加 `HudHost.get_layout_metrics()` 与 `HudLayoutMetrics` |
| 5 | ADR-0014 暴露 `group_counts() -> Array[GroupCount]`，但 `GroupCount` 的字段没有定义 | B3 需要 `{group: StringName, count: int}`；本 ADR 在 Key Interfaces 暂定该形状 | **已对齐**：ADR-0014 已定义 `GroupCount { group, count }` 只读值对象 |
| 6 | ADR-0016 暴露 `get_catalog(station_id) -> Array[UpgradeView]`，但 `UpgradeView` 的字段没有定义 | B4 的首笔升级需要 `{id, name, current_level, max_level}` 与 `next_price(id)` | **已对齐**：ADR-0016 已定义 `UpgradeView { id, display_name_key, level, max_level, current_effect, next_effect, next_price, purchasable, reason }` |
| 7 | `first-run-onboarding.md` 规则 5 给 B2 锚点 `bar:capacity`；Visual/Audio 又说非模态提示「固定在安全区底部的详情行上方」 | 本 ADR 取视觉规格优先：B2 固定停靠；`bar:capacity` 只记录讲解对象，不参与气泡位置 | game-designer 确认两种读法 |
| 8 | 规则 5 给 B3 两个锚点 `card:<该菜>` + `rail:cloy`，未说哪个决定位置 | 本 ADR 用 `card:<该菜>` 作为讲解对象；B3 同样按非模态固定停靠；`rail:cloy` 只用于定位腻度栏 | game-designer / ux-designer 在 tutorial spec 中定稿 |
| 9 | ADR-0004 D9 的 `run_settled` 订阅者只列 `UIRoot` 与 `Audio`；ADR-0007 D3 又说处理器可把结果留给首局引导 B4 | 本 ADR 要求 `OnboardingHost` 订阅 `run_settled` 并只做 `STASH_RESULT`；顺序建议 `UIRoot → Onboarding → Audio` | **已对齐**：ADR-0004 D9 已把订阅顺序改为 `UIRoot → Onboarding → Audio`，并限定 Onboarding 只暂存 |
| 10 | ADR-0004 D8 第 5 步只有 `onboarding.on_flow_state(from,to)`，没有 plan 的传递；`RunConfig` 不含 `menu_entry_fee` / `official_entry_fee` | 本 ADR 新增 `UIRoot.run_config_assembled(config, plan)` 信号，`AppRoot` 订阅后 `onboarding.configure(plan)`；`RunConfig` 不变 | **已对齐**：ADR-0004 D9 已登记 `run_config_assembled(config, plan)` 与 `Onboarding.configure(plan)`，`RunConfig` 不变 |
| 11 | ADR-0003 D2 的层表只有 HUD 10 / Screen 20 / Overlay 25 / Modal 30 / Notice 40 / Fade 50 | 本 ADR 增加 `OnboardingHost` layer 15（HUD 之上、屏幕之下） | **已对齐**：ADR-0003 D2 节点图与分层规则已加入 `OnboardingHost`（layer 15） |
| 12 | ADR-0005 D5 只登记 `SceneFlow`（−100）与 `HudHost`（0）的 `process_priority`；未登记引导 | 本 ADR 要求 `OnboardingHost.process_priority = -50`，并在启动时断言 `-100 < -50 < 0` | **已对齐**：ADR-0005 D5 已登记三档优先级与启动断言 |
| 13 | ADR-0004 D9 把 `onboarding_beat_shown` / `onboarding_finished` 路由给 `Audio`；ADR-0009 未定义任何引导 cue | 静音是合法降级；本 ADR 不要求音效，只保证信号每局至多一次 | audio-director / ADR-0009 确认是否新增 cue |
| 14 | 规则 15 / 状态表：`run_settled` → `Closed` → 构建并交付 `FirstRunExplanation`、广播 `ALL_BEATS` | 本 ADR 把构建与交付挪到 `Run → Menu` 第 5 步（结算提交后、首绘前），`run_settled` 只捕获结果；完成信号仍在交付时发一次 | game-designer / `first-run-onboarding.md` 状态表措辞对齐 ADR-0007 |
| 15 | 规则 19 / 边界用例：预置不变式失败、文案缺失、无安全位置的行为没有统一措辞 | D10 固定 12 级降级；预置失败关闭引导并让 CI 红；文案缺失把 beat 移出 `required()`；无安全位置时模态 `SKIPPED`、上下文静默跳过 | systems-designer / qa-lead 把 D10 写进 AC 的降级用例 |

## Performance Implications

- **CPU**：空闲（`IDLE` / `COACHING` 无焦点变化）时 `_process` 只做一次 Phase 判断，≤ 0.05 ms/帧；`INTRO` / `NUMBERS` 每帧一次 `get_context_snapshot` + 两次 `consume_action`，≤ 0.1 ms；`COACHING` 每帧一次 `focused_dish_id()`，焦点变化时才读 `group_counts()`（O(组数)，MVP ≤ 9）。一次 beat 推进是 O(1) 文案与几何。
- **Memory**：`FirstRunExplanation` 持有本局行（≤ `capacity_total` 条，MVP ≤ 24）与一份菜单事实（≤ 12 项），量级 < 8 KiB；`OnboardingHost` 常驻一个 `TutorialLayer` 与三个子控件，卸载只隐藏不释放。
- **Load Time**：`OnboardingTextTable` / `OnboardingConfig` 各预加载一次；引导挂载无磁盘 I/O；`FirstRunPreset.validate` 在 `Boot → Menu` 跑一次，6 道全排列最多 720 次，< 1 ms。
- **Rendering**：一个全屏 `Control` 的补集遮罩（最多 4×3 个矩形）+ 一条 1 px 指引线 + 一个 `PanelContainer`；无粒子、无 shader、无屏幕震动。`reduce_motion` 下无 `Tween`。
- **Network**：不适用（MVP 单机）。

## Migration Plan

仓库没有 `.gd`，本 ADR 是绿地决策。实施顺序：

1. 落 `FirstRunGate` / `FirstRunPreset` / `FirstRunRunPlan` / `FirstRunPlanner`，用固定夹具跑 AC 1、2、4、28、31、32；`FirstRunPreset.validate` 先接 `make_cloying`。
2. 落 `OnboardingTextTable` / `OnboardingConfig` / lint（48 字、数字字面量、禁用措辞、占位符），跑 AC 14、23、24、25、30。
3. 落 `BeatMachine` / `BeatFacts` / `Intent`，用表驱动测试跑 AC 5–13、18、19、21、26、27；此时不需要 Godot 渲染。
4. 落 `BubbleLayout` / `ScrimPieces` / `HudLayoutMetrics` 夹具，跑 AC 15、25 的几何与遮罩断言（720p / 1080p / 4K / 150%）。
5. 落 `OnboardingHost` / `TutorialLayer` / 输入拉取，接 `AppRoot` 的 `run_config_assembled` 与 `run_*` 信号；跑 AC 5–11、18、19、21 的集成测试。
6. 落 `OnboardingSource` / `ExplanationBuilder` / `FirstRunExplanation`，接 `UIRoot` 的 `explanation_ready` 与收据屏；跑 AC 22–24、29、30 的 B4 分支与钱包时序。
7. 接 `HudHost.get_layout_metrics()`、`MenuWaves.get_run_menu()`、`MetaUpgrades.get_catalog()`（后两者已存在）与 `CloyingAndTaste.group_counts()`；补齐 GDD Sync Check 第 4–6 条。
8. 在真实 4.7.x 上跑 Engine Compatibility 的 ①–⑩，截图留档到 `production/qa/evidence/`。

**Rollback plan**：本 ADR 只新增 `src/application/onboarding/` 与 `src/presentation/onboarding/`、两个资源文件，并在 `AppRoot` 增加一个 `run_config_assembled` 订阅与 `OnboardingHost` 构造。回滚即删除两个目录与资源，移除 `AppRoot` 的构造/订阅、`UIRoot` 的信号发射，以及 `HudHost.get_layout_metrics()` 的新增方法；`RunConfig` / `RunFlow` / `RunResult` 均无字段变化。

## Validation Criteria

| # | 验证 | 证据 |
|---|---|---|
| 1 | `FirstRunGate` 真值表：`Fresh/Loaded/Migrated` + `runs_settled=0` → `ELIGIBLE`；`Recovered` 即使 `runs_settled=0` → `RECOVERED`；`runs_settled>0` → `CAREER_EXISTS`；纪录不可用 → `RECORDS_UNAVAILABLE`；预置失败 → `PRESET_INVALID` | 无头表驱动测试 |
| 2 | `FirstRunPreset.validate` 七条不变式：合法六道全过；逐条注入违规（改价格、改组、改饱腹、删菜、改正式费）时恰好失败对应编号 | 无头故障注入 + CI 门禁 |
| 3 | 首局菜单显示 40、`RunConfig.entry_fee_override = 40`、`RunConfig.pool_override` 六道、`is_first_run = true` 同源；非首局菜单 85、两个 override 为 `null` | 无头集成 + 截图 |
| 4 | `is_first_run = false`：零 `push_context`、零气泡、零 `onboarding_beat_shown`、零 `explanation_ready` | spy + 无头集成 |
| 5 | `run_started` 同帧 B0 可见、`Blocked` 深度 1、三个受保护读数未被遮罩压暗、截图留档 | 帧号日志 + 截图 |
| 6 | B0 `ui_accept` → B1，`turns_remaining` / `capacity_used` / `eaten_count` 不变，`RunFlow` 未收到该确认 | 无头集成 + spy |
| 7 | B1 无焦点 `ui_accept` → 不推进、不吃菜、提示选菜；有焦点 → `Coaching`，`eaten_count` 不变 | 无头集成 |
| 8 | B1 推进后 `Blocked` 深度回到 0；`consume_action(ui_accept)` 在 HUD 侧读到 `false` | 无头集成 |
| 9 | B2 显示时输入上下文仍为 `Run`、不挡任何卡片、不改变焦点；直接提交吃菜后提示消失、动作正常提交 | 无头集成 + 截图 |
| 10 | B3：焦点进入组计数 ≥ 1 的菜时替换 B2、只显示一次；未触发时不在 `required`；触发后即使跳过波次仍在 `required` | 无头表驱动 |
| 11 | 同一帧两次 `ui_accept` 只推进一个 beat；`input_seq` 相同的事件被丢弃 | 无头集成 |
| 12 | 任一模态 beat：`ui_accept` / `game_skip_wave` 不到达 `RunFlow`，回合/容量不变 | `submit_action` spy |
| 13 | `ui_cancel` 跳过：遮罩/气泡立即消失、`Blocked` 弹出、`onboarding_finished(SKIPPED)` 一次、无 B4；鼠标点「跳过引导」走同一路径 | 无头集成 + 手工验收 |
| 14 | 暂停恢复：`Blocked` 用同一白名单重压；假 `InputRouter` 返回 `false` 时降级为无遮罩提示，玩家可正常吃菜，无死锁 | 无头集成 + 手工验收 |
| 15 | 放弃/故障：无遮罩残留、无 `onboarding_finished`、无 `explanation_ready`、存档逐字段不变、下一局仍为首局 | 存档 diff + spy |
| 16 | 几何：1280×720 `m=36`、1080p `m=54`、4K `w=440`、150% 字号三行内；气泡与三个受保护矩形不相交；锚点为空 → 停靠；退化视口 → `HIDDEN` | 无头 `Rect2` 断言 + 截图 |
| 17 | 遮罩：`ScrimPieces` 并集 + 三个受保护矩形 = 视口；受保护区域无绘制；灰度截图中回合/容量/净利仍可读 | 几何断言 + 截图 |
| 18 | 文本 lint：全部 ≤ 48 字（150%）、模板无数字、无 `元/格`/`每格`/`unit_value`、无「最亏/白吃/浪费/下次先拿」；缺条目回退 `short`，再缺 → `TEXT_MISSING` | lint 门禁 |
| 19 | B4 五分支：空局 → `EMPTY_RUN`；全吃 → `ALL_EATEN`；装不下 → `DID_NOT_FIT`；装得下且 `uv_gap>0.5` → `HIGHER_UV_DIFFERENT_CELLS`；否则 `ALREADY_BEST`；并列 tie-break 确定 | 无头表驱动 + 手工复算 |
| 20 | B4 数值：`entry_fee` / `official_entry_fee` / `fee_is_override` 正确；`lowest_uv_row` 与 `best_menu` 可手工复算；首笔升级取最便宜且 `affordable` 与结算后钱包一致 | 无头夹具 + 收据截图 |
| 21 | 交付时序：`explanation_ready` 在 `Run → Menu` 第 5 步、结算 `commit` 之后、收据首绘之前；`onboarding_finished(ALL_BEATS)` 同帧一次；重复交付不重发 | 帧号日志 + commit spy |
| 22 | 无计时/异步：代码中无 `Timer` / `create_timer` / `await` / `call_deferred`；静止 30 秒状态不变 | grep + 无头集成 |
| 23 | 无持久化/无规则写：`onboarding` 目录无 `save.` / `commit` / `prepare_run` / `submit_action` / `spend_turn` / `consume(` / `set_frozen` | grep 门禁 |
| 24 | 150% 字号 + 1280×720 / 1920×1080 / 超宽屏三组截图：气泡不溢出、不压受保护矩形、跳过按钮可达 | `production/qa/evidence/` 截图 |
| 25 | 只用键盘/手柄可完成全流程：B0 → B1 → 选菜 → `Coaching` → 结算 → 收据；`ui_cancel` 可跳过；无「只能鼠标」状态 | 手工验收 |
| 26 | 预置损坏夹具：引导关闭、菜单 85、本局正常开始、ERROR 一条、CI 红 | 无头集成 + CI |
| 27 | 费覆盖缺失夹具：菜单 85、本局扣 85、B0 无「教学价」句、B4 无 `fee_note`、WARNING 一条 | 无头集成 + 截图 |
| 28 | 池覆盖缺失夹具：用实际菜池、B2 无定量句、B3 仅在真实重复组触发、无错误弹窗 | 无头集成 |

## Related

- `docs/architecture/architecture.md` — Required ADRs 第 16 项后的收尾项、API Boundaries（Onboarding / HUD / Input）、Data Flow 第 2–3 节
- `design/gdd/first-run-onboarding.md` — 22 条 Core Rules、7 个 Formulas、Edge Cases、AC 1–32、Open Questions 1–9
- `design/gdd/run-flow.md` — `RunConfig` 字段、`run_*` 信号、只读快照、`is_first_run` / `pool_override` / `entry_fee_override`
- `design/gdd/ui-flow.md` — 规则 8 / 12、AC 47、主菜单入场费、收据讲解的渲染归属
- `design/gdd/input.md` — `push_context(Blocked, allow)` / `pop_context` / `consume_action`、双焦点、导航重复
- `design/gdd/scene-flow.md` — 顶层状态、暂停清栈、`Run → Menu` 交接
- `design/gdd/in-game-hud.md` — 六个锚点 id、三个受保护矩形、焦点与确认面板
- `design/gdd/menu-waves.md` — `get_run_menu` / `visible_dishes` / `wave_index` / `wave_count`、`pool_override`
- `design/gdd/dish-values.md` — `price` / `satiety` / `unit_value` / `uv_cmp`；单位价值只在收据
- `design/gdd/settlement-and-entry-fee.md` — `RunResult.rows[]`、`entry_fee` / `net_profit`、回本率
- `design/gdd/cloying-and-taste.md` — `group_count(g)` / `extra_satiety` / 解腻
- `design/gdd/meta-upgrades.md` — 钱包、升级目录、首笔升级只读
- `design/gdd/records-and-milestones.md` — `career.runs_settled`
- `design/gdd/save-system.md` — 规则 26 的 `first_run_eligible`、`Recovered` 语义
- `design/gdd/audio-feedback.md` — `onboarding_beat_shown` / `onboarding_finished` 的可选订阅
- ADR-0002（持久化格式）— `LoadReport` 与 `Recovered`
- ADR-0003（场景与屏幕架构）— `CanvasLayer` 分层、`FocusSpec`、收据先于 HUD 卸载
- ADR-0004（服务组合与信号）— 六步扇出、`run_*` 订阅顺序
- ADR-0005（输入上下文栈语义）— `Blocked`、`consume_action`、`input_seq`、`process_priority`
- ADR-0006（确定性随机与菜单波次）— `get_run_menu`
- ADR-0007（局生命周期与结算事务）— `run_settled` 预通知、`Run → Menu` 唯一 `commit`
- ADR-0010（菜品数值契约）— `price` / `satiety` / `unit_value`
- ADR-0014（腻度与口味状态机）— `preview` / `record_eat` / `group_counts`
- ADR-0015（结算纯函数）— `RunResult.rows[]` / `entry_fee` / `capacity_*`
- ADR-0016（局外升级与钱包）— `get_wallet` / `get_catalog` / `next_price`
- ADR-0018（纪录、里程碑与生涯完成）— `career.runs_settled`
- ADR-0019（局内 HUD 数据绑定与布局）— `get_anchor_rect` / `get_protected_rects` / `focused_dish_id` / 逆缩放
- `docs/engine-reference/godot/modules/ui.md`、`modules/input.md` — 4.6 双焦点、`CanvasLayer` 层序与 `MouseFilter`
