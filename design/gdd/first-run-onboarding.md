# 首局引导（First-Run Onboarding）

> **Status**: In Design
> **Author**: user + Claude（autonomous 模式；systems-designer 与 qa-lead 按 Lean 模式仅复核 Formulas 与 Acceptance Criteria）
> **Last Updated**: 2026-11-04
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 每口都是算术题 · 没有白吃的菜

## Summary

首局引导把玩家生涯的第一局变成一堂课：只开六道菜、用教学价入场，用一次「装不下全部」的取餐教玩家价格与饱腹是两个必须一起看的数，并在收据上给出每道菜的单位价值。它只在首局运行，只压输入、只放气泡，不改任何玩法规则；从第二局起它完全不存在。

> **Quick reference** — Layer: `Polish` · Priority: `Vertical Slice` · Key deps: `餐局流程, 局内 HUD, 输入, 场景与流程管理, 菜品数值, 结算与入场费, 腻度与口味`

## Overview

首局引导是《回本》的新玩家入口。它不新建一局流程，而是**接管生涯的第一局**：向组装 `RunConfig` 的一方提供一个首局预置（六道菜的菜池、教学入场费 40 元），在这一局里按固定顺序放几条讲解气泡，把「价格」「饱腹」「容量」「同口味重复」四个词在玩家第一次做决定之前解释清楚。

它存在的原因是支柱 1 与反支柱之间的张力。支柱 1 要求每个决定都能被心算验证，反支柱要求「不需要查攻略就能看懂」；而一个 12 道菜、还要算腻度的菜单，第一次见面就是攻略。首局把菜单压到六道、把入场费降到教学价，让玩家在一局「认真玩就一定能回本、并且一定买得起第一项升级」的局面里先看到公式长什么样，再回到完整菜单。

**为什么不是零入场费**：零费用会让 HUD 的三个核心读数在本局全部退化——`running_net` 从 0 起步、`shortfall` 恒为 0（一直显示「已回本」）、收据的 `return_ratio` 无定义（显示「免单」）。而这一局最该被看见的恰恰是「−40 → 还差 40 元 → 回本」这条线。教学价必须大于零。

边界很清楚：**它拥有教学序列、文案、焦点约束、完成与跳过判定，以及首局预置的参数**；它不拥有 HUD 的绘制、输入的上下文机、餐局流程的生命周期、结算的求和与回本判定，也不写任何存档。它压入 `Blocked` 上下文时改变的只是「这一帧玩家能按什么」，不是「这一帧游戏算什么」。

## Detailed Design

### Core Rules

1. **职责边界**：本系统拥有首局的教学目标序列、气泡文案、焦点约束、完成/跳过判定，以及首局预置（`pool_override` 的六道菜与 `entry_fee_override = 40`）。它不拥有：局内一屏的布局与重绘（局内 HUD）、输入上下文与动作映射（输入）、一局的生命周期与动作仲裁（餐局流程）、净利与回本（结算与入场费）、收据屏与菜单（界面流程）、任何持久化（存档 / 纪录与里程碑）。**它读的上游一律只读，写出去的一律是请求或 payload。**

2. **只在首局运行**：唯一触发条件是 `RunConfig.is_first_run = true`。没有中途开关、没有设置项、没有「再看一次」入口；`is_first_run = false` 时本系统不初始化、不压上下文、不绘制任何元素。谁决定 `is_first_run` 归组装 `RunConfig` 的一方（界面流程），本系统只消费它。**组装方的判据必须是两项同时成立**（S11）：`records.career.runs_settled == 0` **且** `LoadReport.first_run_eligible = true`。`first_run_eligible` 由存档给出，只在载入结果为 `Fresh` / `Loaded` / `Migrated` 且 `runs_settled == 0` 时为 `true`；**载入结果为 `Recovered` 时恒为 `false`**——恢复出的备份可能早于第一次结算，单看 `runs_settled = 0` 会把老玩家重新塞回教学。本系统不读存档，也不重新推导这两项，只消费组装方给出的 `is_first_run`。

3. **首局预置由本系统提供、由组装方写进 `RunConfig`**：`is_first_run = true` 时，界面流程在组装 `RunConfig` 之前向本系统取一份预置：

   | 字段 | 值 | 说明 |
   |---|---|---|
   | `pool_override` | `[dish_seafood_pot, dish_sweet_sour_pork, dish_braised_pork, dish_steamed_fish, dish_fried_chicken, dish_mapo_tofu]` | 六项，是 `station_canteen.dish_pool` 的非空子集（餐局流程的校验前提） |
   | `entry_fee_override` | `40` | 教学价；结算照常 `net_profit = total_value − 40`，收据显示正常回本率 |

   本系统不理解这两个值如何被消费：`pool_override` 原样交给菜单波次，`entry_fee_override` 原样交给结算。**教学价不是新规则**——`entry_fee` 本来就是 ≥ 0 的整数、由 `RunConfig` 冻结整局；本系统只是为一个档口选了一个低费值，没有改写求和或回本。40 的取值理由见 Formulas 的 `first_run_pool`。

   **主菜单必须显示本局实际要收的费**（S9）：`is_first_run = true` 时，主菜单开始按钮旁显示的入场费必须是 `entry_fee_override`（40），不是 `StationData.entry_fee`（85）。这两个数同时出现在屏幕上（菜单 85、结算扣 40）是契约缺陷：玩家会按 85 规划回本，却在收据上看到 40。界面流程在组装 `RunConfig` 时已知道 `is_first_run`，因此必须用同一个标记切换菜单上的读数；本系统不绘制菜单，只声明这个一致性要求。`entry_fee_override` 不被支持时按规则 19 降级：**此时菜单与结算都必须用 85**，且组装方必须在组装时就发现字段缺失（不能菜单显示 40、结算收 85——那是同一缺陷的两种表现）。

4. **只压输入，不改规则**：本系统对一局唯一的写操作是 `push_context(Blocked, allow=[…])` / `pop_context()`。它不得写 `run_state`、不得开始或结束一局、不得消耗回合或容量、不得改变 `run_id`、不得广播 `run_started` / `run_ended`、不得改 `frozen`。气泡打开期间 `run_state` 仍是 `Active`，局内数字仍是最后一帧的合法值。

5. **教学序列是固定的线性序列**：`B0 开场 → B1 两个数 →（自由取餐，B2/B3 为上下文提示）→ B4 收据讲解`。序列本身不分支；分支只在「某条上下文提示是否被触发」上。

   | Beat | 触发 | 形态 | 锚点 | 推进条件 |
   |---|---|---|---|---|
   | `B0_WELCOME` | `run_started` 且 `is_first_run` | 模态 | `top:net` | `ui_accept` |
   | `B1_TWO_NUMBERS` | B0 完成 | 模态 | `card:<当前焦点>`，无焦点时 `grid:first_card` | 任意菜品有焦点时按 `ui_accept` |
   | `B2_CAPACITY` | B1 完成 | 上下文提示 | `bar:capacity` | 下一次 `action_committed`，或 `run_ended` |
   | `B3_CLOYING` | 焦点落在「所属组计数 ≥ 1」的菜上 | 上下文提示 | `card:<该菜>` + `rail:cloy` | 下一次 `action_committed`，或 `run_ended` |
   | `B4_RECEIPT` | `run_settled`（`RunResult` 就绪） | 收据 payload | 收据屏（由界面流程摆放） | payload 交付给界面流程 |

   **每条 beat 只讲一件事**：`B0` 讲规则（入场费 40、20 格、10 回合、六道菜）；`B1` 讲**价格（元）与饱腹（格）这两个数**同时看；`B2` 讲容量是硬上限、这一桌装不下全部、菜会一波一波上且跳过不会回来（波数取实际值，不写死「两波」）；`B3` 讲同组第二道多占一格、含 `酸`/`冰` 的菜能解掉一格；`B4` 讲单位价值与一处替换建议。一条气泡不得同时讲两件事。

   **教学范围是刻意裁剪（D6）**：本局只教四个概念——价格、基础饱腹、容量、同组重复——**不教 `price / effective_satiety` 的比值，也不教腻度追加的完整规则**。一屏同时有约 8 个可读信息流，首局的注意力预算装不下全部；因此「不教」是取舍，不是遗漏。比值留到收据里由玩家自己发现，局内绝不出现（规则 13）。是否在后续局补一条「每格」提示、或把 B1 拆成更细的步骤，交由 `/prototype` 的观察与 `/ux-design` 决定；本版只做措辞级澄清，不新增 beat、不加数字、不改 B1 的推进条件。

6. **一次动作只推进一个 beat**：同一个输入事件最多推进一个 beat；同一帧内第二个推进事件被忽略。B1 的 `ui_accept` **只推进气泡，不吃菜**——`Blocked` 白名单把 `ui_accept` 交给本系统，餐局流程在气泡存在期间永远收不到它。

7. **模态 beat 压 `Blocked`，上下文提示不压**：`B0` / `B1` 压入 `Blocked` 并带动作白名单（`B0` 只放行 `ui_accept` / `ui_cancel`；`B1` 额外放行四个导航动作）。`B2` / `B3` 是**非模态**提示，不改变输入上下文、不挡任何动作；玩家可以直接忽略它们继续吃菜。**同一时刻最多显示一条非模态提示**，新提示替换旧提示，被替换者记为已展示。

8. **上下文提示的完成判据是「展示过」，不是「读完」**：`B2` / `B3` 各有自己的触发条件，触发即视为完成（写入 `shown` 集合），之后不再重放。它们不等待玩家确认，也不用计时器。

9. **B3 的触发条件**：玩家把焦点（键盘/手柄焦点或鼠标悬停，按 HUD 的双焦点规则）放到一道「所属 `cloying_group` 的**吃这道之前的**计数 ≥ 1」的菜上——计数口径与腻度与口味的 `extra_satiety` 完全一致，不重算。焦点解析归 HUD，本系统只读它发布的 `focused_dish_id`；本系统**不移动焦点**。若整局都没有这种时刻，B3 不进入必需集合（见 Formulas 的 `beats_required`）。

10. **焦点约束只来自白名单**：本系统不接管焦点链、不隐藏不可获焦的控件、不改 HUD 的导航顺序。`B0` 期间四个导航动作不在白名单里，因此焦点不动；`B1` 期间导航可用，玩家自己选一道菜。**本系统从不替玩家选菜**——这是支柱 1 的底线：可以少按几个键，不能替玩家做算术。

11. **锚点由 HUD 解析，位置由本系统计算**：本系统向 HUD 请求 `get_anchor_rect(anchor_id) -> Rect2`，`anchor_id` 取 `top:net` / `top:turns` / `bar:capacity` / `rail:cloy` / `card:<dish_id>` / `grid:first_card` 六个值。HUD 拥有「这个 id 在屏幕哪里」，本系统拥有「气泡放在它上方还是下方、留多少间距」。两边都不需要知道对方的布局实现——这回答了局内 HUD Open Question 8。

12. **不遮核心数字**：任何气泡的矩形都不得与「回合读数」「容量读数」「净利/回本差额」三个受保护区域相交。模态 beat 的遮罩只压暗受保护区域之外的画面，**受保护区域保持原亮度**。冲突时先翻到锚点另一侧，仍冲突则停靠到底部详情行之上（见 Formulas 的 `bubble_rect`）。

13. **局内不显示单位价值**：气泡只使用 HUD 已经展示的数字与结构化事实——价格、基础饱腹、实际饱腹、剩余/总容量、剩余/总回合、当前波、菜品数。**任何 beat 都不得写出「这道菜每格 N 元」或等价的比值**。单位价值只在 `B4` 的收据讲解里出现（菜品数值规则 5、规则 6；局内 HUD 规则 18）。

14. **收据讲解是 payload，不是本系统绘制的界面**：`B4` 产出 `FirstRunExplanation`（两段短文案、逐行单位价值的取舍说明、一处替换建议、一条首笔升级建议），交给界面流程在收据屏上渲染。本系统不拥有收据的布局、顺序、字体或折叠；收据没有出现（放弃、故障、`init_failed`）时本系统静默结束，不补弹窗。

15. **完成判定**：`beats_required` 全部展示过、且 `B4` payload 已交付，即视为完成，广播一次 `onboarding_finished(ALL_BEATS)`。完成不写任何持久化。**「下一局还算不算首局」只有一个判据，由组装方按 `records.career.runs_settled == 0 ∧ LoadReport.first_run_eligible` 判定**；`onboarding_finished` 只用于关闭本局的遮罩与讲解，**不参与该判定**——否则「先跳过、再放弃」会让下一局在第一局从未结算的情况下丢掉教学。

16. **跳过**：模态 beat 上按 `ui_cancel`，或点击气泡上的「跳过引导」文字按钮，立即跳过本局的全部引导：弹出 `Blocked`、丢弃剩余 beat、不再显示任何提示与收据讲解，广播一次 `onboarding_finished(SKIPPED)`。**不二次确认**——跳过的代价为零，确认框只是多一次点击。跳过只影响本局，不写存档。

17. **暂停、放弃、故障**：
    - 气泡打开期间玩家按 `ui_pause`：场景与流程管理切到 `Paused` 时用 `request_context(Paused, clear_stack := true)` 清空临时栈，本系统的 `Blocked` 随之失效。恢复回 `Run` 时若仍有待读的模态 beat，本系统**重新压入**同一个白名单；重新压入失败则不重试、把该 beat 降级为非模态提示，绝不卡住玩家。
    - 放弃、退出、`run_discarded`、`run_faulted`、`init_failed`：立即拆除遮罩与气泡、弹出上下文、不产出 `B4`、不广播 `onboarding_finished`、不写任何东西。

18. **不写持久化**：本系统没有存档命名空间、不调 `commit`、不写 `extensions`、不读钱包或纪录。它对「首局是否已经发生过」只消费 `RunConfig.is_first_run`，对「首局是否完成过」只广播信号供上层使用。

19. **降级优先于阻塞**：任何一项上游能力缺失（`pool_override` 被菜单波次忽略、`entry_fee_override` 未被支持、HUD 不提供锚点、菜品数值查询失败）都不得阻止这一局开始。降级顺序：缺锚点 → 气泡停靠底部；缺 `entry_fee_override` → 本局按 85 元入场，B0 改说「这一局入场费 N 元」并省略「教学价」句，其余 beat 不变；缺 `pool_override` → 用档口原菜池，B2 的「六道菜」改为实际可见数；缺单位价值 → `B4` 只给文案，不给逐行比值与替换建议。

20. **文案与数值分离**：所有玩家可见字符串来自本系统的文本表（`onboarding_text`），气泡里的数字来自上游读数或预置表，不在字符串里写死。文案表必须能承受 150% 字号与本地化扩写（见 Formulas 的 `bubble_text_cap`）。**气泡几何是数据驱动的**（`OnboardingConfig`：安全边距来源、`gap`、`padding`、宽度系数与上下限、行高、最大行数）；**六道菜池与 40 元教学价是常量**，不随 `OnboardingConfig` 调参——它们被 `first_run_pool` 的四条推导锁死，改了会静默破坏首局的取舍与第一笔升级的可负担性。

21. **不改写任何上游记录**：`pool_override` 只选 id，不复制、不修改 `DishData`；`entry_fee_override` 只对一个 `run_id` 生效，不回写 `StationData.entry_fee`。

22. **首局可重复进入，但不可重复获利**：首局被放弃时不结算、不入账，「下一局仍是首局」是合法结果；一旦结算，`career.runs_settled` 增加，组装方不再标记 `is_first_run`。**`Recovered` 载入是例外**：即使恢复出的 `runs_settled` 为 0，组装方也必须因 `first_run_eligible = false` 而不标记首局——否则一个已结算过的老玩家会在备份恢复后重看整段教学（S11）。本系统不参与这条判定，也不提供任何奖励。

### States and Transitions

本系统的状态只在一局内存在，与 `run_state` 正交（`run_state` 始终由餐局流程拥有）。

| 状态 | 进入条件 | 本系统可见行为 | 离开 |
|---|---|---|---|
| `Idle` | 未开局，或 `is_first_run = false` | 无 | `run_started` 且 `is_first_run` → `Intro` |
| `Intro` | 首局开局，B0 待展示 | 模态气泡 + 遮罩；`Blocked[ui_accept, ui_cancel]` | B0 完成 → `Numbers`；跳过 → `Closed`；局结束/丢弃 → `Cancelled` |
| `Numbers` | B0 完成 | 模态气泡；`Blocked[nav, ui_accept, ui_cancel]` | B1 完成 → `Coaching`；跳过 → `Closed`；局结束/丢弃 → `Cancelled` |
| `Coaching` | B1 完成 | 无上下文限制；按事件显示 B2 / B3 | `run_ended` → `Closing`；跳过（仅模态 beat 可用，此状态无）→ 不适用；丢弃/故障 → `Cancelled` |
| `Closing` | `run_ended` | 等待 `run_settled`；结果到达后产出 `FirstRunExplanation` 并交付，不绘制局内元素 | 交付完成 → `Closed`；`run_faulted` / 丢弃 → `Cancelled` |
| `Closed` | 完成或跳过 | 无 | 一局结束即终态；重开是新的一局 |
| `Cancelled` | 局被丢弃、故障或 `init_failed` | 无；不产出 `B4`、不广播完成 | 终态；内部状态，不对上层可见 |

| 从 | 触发 | 到 | 副作用 |
|---|---|---|---|
| `Idle` | `run_started`，`is_first_run = true` | `Intro` | 同步压 `Blocked`；请求首个锚点 |
| `Intro` | `ui_accept` | `Numbers` | 弹出旧白名单、压入 `B1` 白名单；不出局内动作 |
| `Numbers` | 有焦点时 `ui_accept` | `Coaching` | 弹出 `Blocked`；显示 B2 |
| `Coaching` | 焦点进入已计数组 | `Coaching` | 显示 B3（替换 B2）；两者都只展示一次 |
| `Coaching` | `run_ended` | `Closing` | 丢弃未展示的非模态提示（不补弹） |
| `Closing` | `run_settled`（`RunResult` 就绪） | `Closed` | 构建并交付 `FirstRunExplanation`；广播 `onboarding_finished(ALL_BEATS)` |
| `任意` | `run_discarded` / `run_faulted` / `init_failed` | `Cancelled` | 弹上下文、拆除遮罩、不广播完成、不写任何东西 |
| `任意模态态` | `ui_cancel` | `Closed` | 弹上下文；广播 `onboarding_finished(SKIPPED)` |

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 场景与流程管理 | 上游（间接） | 只读顶层状态以判断本局是否已离开 `Run` / `Paused`；不请求顶层切换 | 软 |
| 餐局流程 | 上游 | `RunConfig.is_first_run`；`run_started` / `action_committed` / `run_ended` / `run_settled` / `run_discarded` / `run_faulted`；只读快照里的 `run_state`、`turns_remaining`、`capacity_used`、`capacity_total`、`eaten_count` | 硬 |
| 输入 | 上游 | `push_context(Blocked, allow=[…])` / `pop_context()`；查询 `ui_accept`、`ui_cancel`、四个导航动作；**不监听原始事件、不改映射** | 硬 |
| 局内 HUD | 上游（协作） | `get_anchor_rect(anchor_id) -> Rect2`、`focused_dish_id`、当前波可见数与波序；气泡绘制在 HUD 之上 | 硬 |
| 菜品数值 | 上游 | 六道菜 id 的 `price` / `satiety` / `cloying_group` / `unit_value`（只在 `B4` 使用）；不自行重算比值 | 硬 |
| 结算与入场费 | 上游 | `RunResult.rows[]` 的 `price`、`satiety`、`extra_satiety`、`effective_satiety`、`unit_value`、`effective_unit_value`；`entry_fee`、`total_value`、`net_profit` | 硬（`B4`） |
| 菜单波次 | 上游（间接） | 接收 `pool_override`；`get_run_menu() -> StringName[]`（本局有效菜单，供 `replacement_hint`）；只读当前波的可见数与波序用于 B2 文案 | 软 |
| 腻度与口味 | 上游（间接） | 读 `focused_dish_id` 的组计数与 `extra_satiety` 用于 B3 触发与文案；不写计数 | 软 |
| 界面流程 | 下游 | 取预置（`pool_override` / `entry_fee_override`）；接收 `FirstRunExplanation` 并渲染收据讲解；**不据 `onboarding_finished` 判定首局**——`is_first_run` 只按 `records.career.runs_settled == 0 ∧ LoadReport.first_run_eligible`；主菜单显示的入场费必须与 `entry_fee_override` 一致（S9） | 硬（UI） |
| 局外升级 | 下游（仅文案） | `B4` 的「首笔升级建议」读升级目录与当前钱包，只出文字，不购买、不高亮按钮 | 软 |
| 存档 | 无直接调用 | 不读写任何命名空间 | — |
| 音效反馈 | 下游（订阅） | `onboarding_beat_shown(beat_id)`、`onboarding_finished(reason)` | 软 |

**接口所有权**：本系统拥有 beat 序列、文案、锚点的使用方式、完成与跳过判定、首局预置的内容；HUD 拥有锚点矩形与焦点；输入拥有上下文合法性；餐局流程拥有一局的生命周期；界面流程拥有收据屏与预置的落位。

**信号**：

- `onboarding_beat_shown(beat_id: StringName)` — 每个 beat 至多一次（含被替换的非模态提示）。
- `onboarding_finished(reason: enum {ALL_BEATS, SKIPPED})` — 每局至多一次；放弃/故障不发。

## Formulas

本节定义七条可判定规则：首局是否生效、序列推进、必需 beat 集合与完成、超时、气泡几何、文案长度、以及收据讲解的替换建议。它们不是平衡数值，不进入数据配置的平衡表；但它们是可验收的硬约束，因此按公式写。`first_run_pool` 另作一个内容常量表列出，并附上它的取舍证明。

### 首局生效 `onboarding_active`

The `onboarding_active` formula is defined as:

`onboarding_active = is_first_run ∧ (run_state ∈ {Active, Ending}) ∧ ¬finished`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `is_first_run` | bool | {true, false} | `RunConfig.is_first_run` | 组装方给出的首局标记 |
| `run_state` | enum | 8 个局内状态 | 餐局流程只读快照 | 本局的当前状态 |
| `finished` | bool | {true, false} | 本系统 | 是否已广播过 `onboarding_finished` |

**Output Range:** `{true, false}`。`Initializing`、`Settling`、`Handoff`、`Failed`、`Faulted` 一律为 `false`——本系统不在没有 `Active` 画面的帧上绘制。

**Example:** 首局进入 `Active`、未完成 → `true`；玩家跳过 → `finished = true` → 之后恒为 `false`，即使本局仍在 `Active`。

### 序列推进 `beat_next`

The `beat_next` formula is defined as:

`beat_next(beat, event) =` 查下表所得的唯一后继（无匹配则保持当前 beat）

| 当前 beat | `event` | 后继 | 备注 |
|---|---|---|---|
| `B0_WELCOME` | `ui_accept` | `B1_TWO_NUMBERS` | 同一帧最多推进一次 |
| `B1_TWO_NUMBERS` | `ui_accept ∧ focused_dish_id ≠ ⊥` | `COACHING` | 有焦点才推进；无焦点时 `ui_accept` 不生效 |
| `COACHING` | `action_committed` 且 B2 未展示 | `COACHING`（补展示 B2） | 兜底：进入 `COACHING` 时若 B2 因锚点或文案缺失未展示，在第一次提交时补一次 |
| `COACHING` | `focus_enter(d)` 且 `group_count(group(d)) ≥ 1` 且 B3 未展示 | `COACHING`（展示 B3） | 与 B2 互斥：替换当前提示 |
| `COACHING` | `run_ended` | `CLOSING` | 丢弃未展示提示 |
| `任意` | `ui_cancel ∧ 当前 beat 为模态` | `CLOSED` | 跳过 |

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `beat` | enum | `B0`…`B4` / `COACHING` | 本系统 | 当前教学节点 |
| `event` | enum | 输入动作 / 局内事件 | 输入、餐局流程、HUD | 本帧到达的事件 |
| `focused_dish_id` | StringName \| ⊥ | 可见菜 id / `⊥` | HUD | 当前焦点菜品 |
| `group_count(g)` | int | 0–`cloy_cap` | 腻度与口味 | 组的当前计数 |

**Output Range:** 上表覆盖的全部节点。**推进事件按输入序列号单调消费**：只接受 `input_seq > last_consumed_seq` 的事件，同一帧到达的两个事件里第二个被丢弃；`ui_accept` 的自动重复已由输入规则 5 排除。B0 与 B1 的模态分支都在推进前调用 `输入.consume_action(ui_accept)` 取走本帧的确认；餐局流程与 HUD 随后轮询同一个动作得到 `false`，因此看不到它。

**Example:** B1 中玩家把鼠标移到「清蒸鱼」再按确认 → `focused_dish_id = dish_steamed_fish ≠ ⊥` → 进入 `COACHING`；餐局流程没有收到这次 `ui_accept`，清蒸鱼没有被吃。

### 必需 beat 与完成 `beats_required`

The `beats_required` formula is defined as:

`beats_required = {B0, B1, B2, B4} ∪ ({B3} if B3.triggered else ∅)`

`onboarding_complete = beats_required ⊆ beats_shown`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `B0`/`B1`/`B2`/`B4` | beat | 固定 | 本 GDD | 无条件必需的四个节点 |
| `B3.triggered` | bool | {true, false} | 本系统 | B3 的触发条件是否在本局出现过 |
| `beats_shown` | beat 集合 | ⊆ 全部 beat | 本系统 | 已展示过的节点 |

**Output Range:** `beats_required` 为 4 或 5 个节点。`onboarding_complete` 为 `{true, false}`。

**Example:** 玩家正常吃完 5 道菜 → B3 被触发 → 必需 5 个；玩家第一波直接跳过、最后一波结束取餐、一道没吃 → B3 未触发、B2 在进入 `Coaching` 时已展示 → 必需 4 个，收据为空但 `B4` 仍交付。

> **B3 在本池下正常路径必然触发，但规则不依赖这个事实**：按 `first_run_pool_invariants`，六道菜恰好构成两对同组 + 两道各属不同组；基础饱腹之和等于 `C0`，两对重复再各要 1 格，六道全吃需 `C0 + 2`，用 `酸` 解掉一对后仍需 `C0 + 1`，玩家至多吃 5 道。六道里去掉任意一道只能拆掉一对，另一对仍被完整吃掉；吃其中第二道之前必然先聚焦它 → B3 触发。**只有吃菜数 ≤ 2、或主动跳过波次导致某一对整组消失时，B3 才可能不触发**，此时它不进入必需集合。换成不满足「两对同组」不变式的菜池时，B3 会退化为死文案——这是 `first_run_pool_invariants` 的硬约束，不是 B3 的判据缺陷。

### 超时 `onboarding_timeout`

The `onboarding_timeout` formula is defined as:

`onboarding_timeout(beat) = ⊥`（对所有 beat，恒为无超时）

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `beat` | enum | `B0`–`B4` | 本系统 | 任意教学节点 |
| `onboarding_timeout` | duration \| ⊥ | `⊥` | calculated | 等待上限 |

**Output Range:** 恒为 `⊥`。**本系统没有任何计时器**：气泡不自动关闭、不自动推进、不做倒计时、不因长时间无输入而超时（时间与回合规则 2、支柱 3、局内 HUD 规则 17）。唯一的结束方式是玩家动作、局结束或局消失。

**Example:** 玩家在 B0 停留 10 分钟去接电话，回来按 Enter → 正常进入 B1；期间不扣回合、不扣容量、不产生任何日志。

### 气泡几何 `bubble_rect`

The `bubble_rect` formula is defined as:

```
w  = clamp(0.36 × W, 280, 440)
h  = bubble_text_lines × line_height + 2 × padding
x  = clamp(anchor.center.x − w/2, m, W − m − w)
y  = anchor.top − gap − h                 # 首选上方
若 y < m 或 与任一受保护矩形相交：y = anchor.bottom + gap    # 翻到下方
若仍越界或仍相交：                         y = H − m − h      # 停靠底部
```

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `W` / `H` | int | 1280×720 – 4320 | 运行时视口 | 视口宽高（px） |
| `m` | int | 32–96 | HUD `hud_safe_margin` | 安全边距，直接复用 HUD 的值，不另算 |
| `anchor` | Rect2 | 视口内 | HUD `get_anchor_rect(id)` | 被讲解控件的屏幕矩形 |
| `gap` | int | 8–24，起点 12 | constant | 气泡与锚点的间距 |
| `padding` | int | 8–16，起点 12 | constant | 气泡内边距 |
| `line_height` | int | 20–36 | HUD 主题 | 行高，随 `hud_ui_scale` 缩放 |
| `bubble_text_lines` | int | 1–3 | 文本表 | 当前文案的行数（上限见 `bubble_text_cap`） |
| `受保护矩形` | Rect2 集合 | 3 个 | HUD | 回合读数、容量读数、净利/回本差额 |

**Output Range:** `x ∈ [m, W − m − w]`，`y ∈ [m, H − m − h]`。若 `w > W − 2m`（极端窄窗），`w` 收缩到 `W − 2m`，文本换行。**任何输出都不得与三个受保护矩形相交**；三者取不到矩形时按「不相交」处理，并在 debug 构建记一次断言失败。

**Example:** 1280×720、`m = 36`、`gap = 12`、两行文本 `h = 76`；锚点 `bar:capacity` 位于 y≈150 → 上方 `y = 150 − 12 − 76 = 62 ≥ 36` 且不压顶栏核心读数 → 放在上方；4K（W=3840）时 `w = clamp(1382.4, 280, 440) = 440`，气泡不随分辨率变宽。

### 文案长度 `bubble_text_cap`

The `bubble_text_cap` formula is defined as:

`max_chars = bubble_text_lines_max × floor(w / font_px)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `bubble_text_lines_max` | int | 3（常量） | constant | 气泡最多三行 |
| `w` | int | 280–440 | `bubble_rect` | 气泡宽度（px） |
| `font_px` | int | 16–27 | HUD 主题 × `hud_ui_scale` | 基础 18 px，150% 字号时 27 px |
| `max_chars` | int | — | calculated | 单条文案的字符上限（CJK 按 1 em 计） |

**Output Range:** 1280×720、18 px → `3 × floor(440/18) = 72` 字；150% 字号 → `3 × floor(440/27) = 48` 字。**文本表按 48 字验收**（最坏情况）：超过即视为文案缺陷，不靠缩小字号解决，也不在运行时截断——由生成期 lint 拦下。

**Example:** B0 文案「这一局入场费 40 元，回本线就是它。你有 20 格胃、10 回合，先看每道菜的价格和格子数。」共 41 字，在 48 字上限内。

### 首局菜池 `first_run_pool`

The `first_run_pool` 常量表（内容决策，不是运行时计算）。**本表只声明 id 与教学作用；六道菜的价格、饱腹、`cloying_group` 与单位价值一律由 `DishData` 解析，本文件不复制这些数值**（W5：复制一份会随平衡改动静默过期，而教学推导还按旧值写）。

| # | 菜 id | 教学作用（数值与组由 `DishData` 提供） |
|---|---|---|
| 1 | `dish_seafood_pot` | 最贵档、单位价值最低——「贵 ≠ 值」 |
| 2 | `dish_sweet_sour_pork` | 含 `酸`，唯一解腻菜，用来演示解腻 |
| 3 | `dish_braised_pork` | 与 #2 同组，制造一对重复 |
| 4 | `dish_steamed_fish` | 与 #1 同组，制造一对重复 |
| 5 | `dish_fried_chicken` | 中价高值 |
| 6 | `dish_mapo_tofu` | 最便宜、每格高值——「便宜 ≠ 亏」 |

**内容不变式 `first_run_pool_invariants(P, C0, fee)`**（教学局成立的全部条件；CI 与启动自检从 `DishData` 重算，任一条失败即说明**数据改动破坏了首局**，而不是表过期）：

1. `P ⊆ station_canteen.dish_pool`，`|P| = 6`，id 唯一，且六道的 `DishData.station_id = station_canteen`。
2. `Σ_{d∈P} satiety(d) = C0`（基础饱腹之和恰好等于本局容量）。
3. **组结构**：恰好两对同组（每对 2 道），其余 2 道分属两个不同组。因此六道全吃的基础占用之外还要追加 2 格，即 `C0 + 2`。
4. **可行性**：存在可行的 5 道子集（计入重复追加与 `酸` 解腻），且不存在可行的 6 道子集；「放弃价格最低的一道」是**唯一最优解**（保留五道最贵的）。
5. **价值跨度**：`max unit_value / min unit_value ≥ 1.5`；单位价值最低的一道属于价格最高档，价格最低的一道属于单位价值**前两名**（两个反直觉方向都要在场，但不要求它正好是第一名）。
6. **教学价的可负担性**：`Σ_{P\{价格最低}} price − fee ≥ 最便宜容量升级的 1 级价格`，且 `≥ 0`。
7. **正式入场费下必然亏**：`Σ_{P\{价格最低}} price − station_canteen.entry_fee < 0`——否则教学价没有存在理由。

**取值理由（可复核）**：

- **容量在刀刃上**（不变式 2 + 3）：六道基础饱腹恰好占满 `C0`，两对重复再各要 1 格，六道全吃需 `C0 + 2`；用含 `酸` 的那道解掉一对后仍需 `C0 + 1`。结合不变式 4（5 道可行、6 道不可行），玩家**必须且只能放弃恰好一道**——第一局就出现一次真实取舍，而不是「全拿」。
- **放弃最便宜的一道是最优解**（不变式 4）：容量只允许少拿一道时，总值就是价格之和，因此保留五道最贵的、放弃价格最低的。这个结论与「每格最值」相反（单位价值最低的恰是最贵档），正好演示菜品数值的核心口径「价格是得分、容量是成本」。
- **两个反直觉方向都要在场**（不变式 5）：只有「贵 ≠ 值」是半个教训；同时让最便宜的一道单位价值偏高（前两名），玩家才不会把「便宜」直接等同于「亏」。
- **教学价 40，不是零**（不变式 6 + 7）：40 必须让最优五道盈利、且足以买下最便宜的容量升级（第一局就打开局外循环）；同时正式入场费 85 下同一组合必须亏（最优五道的总值低于 85），否则玩家没有理由理解「教学价」是一份临时优惠。取 40 而不是 85 的另一条理由见 Overview：零/正式费会让 HUD 的 `running_net` / `shortfall` / 回本率在本局退化。
- **回本率偏高是刻意的**：首局菜池只有 6 道、入场费按「保证第一次升级可负担」定，不按 `calibration_ratio` 校准。收据讲解必须写明「首局入场费是教学价 40 元，正式食堂是 85 元」，避免玩家把首局回本率当成基准。

**Output Range:** 六项、唯一、顺序无关（菜单波次会按种子重新洗牌）。**替换或增删任何一道都必须重跑 `first_run_pool_invariants` 的七条**（容量与组结构、可行性与唯一最优放弃项、价值跨度、教学价的可负担性、正式费下必亏），否则首局的取舍会消失、反直觉，或第一次结算买不起任何升级。表里的 id 不变、数值在 `DishData` 改，是允许的——只要七条不变式仍成立。

**Example:** 玩家按最优解吃五道、放弃价格最低的一道：六道基础饱腹之和为 `C0`，其中一对重复组追加 1 格、另一对被含 `酸` 的那道解掉，最终占 `C0 − 1` 格；放弃的那道需要 2 格而只剩 1 格——装不下，所以它是**被迫放弃**而不是选错。入场费 40 → 净利与回本率按当前 `DishData` 计算（验收值见 AC 4），钱包足以买下最便宜的容量升级。

### 收据替换建议 `replacement_hint`

The `replacement_hint` formula is defined as:

```
lowest_uv_row    = argmin_{r ∈ RunResult.rows} r.effective_unit_value   # 并列取 dish_id 字典序最小
skipped          = run_menu \ eaten
best_menu        = argmax_{d ∈ skipped} unit_value(d)                   # 并列取 dish_id 字典序最小
uv_gap           = unit_value(best_menu) − lowest_uv_row.effective_unit_value
remaining_at_end = capacity_total − capacity_used
best_menu_fit    = satiety(best_menu) ≤ remaining_at_end
```

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `RunResult.rows[]` | EatenRow[] | 0–`capacity_total` 条 | 结算与入场费 | 本局吃下的每一行，含静态与实际单位价值 |
| `run_menu` | StringName[] | 1–12 项 | 菜单波次 | 本局有效菜单（分区后，含 `pool_override`；首局为六道） |
| `eaten` | StringName 集合 | ⊆ `run_menu` | `RunResult.rows` | 本局吃过的菜 |
| `skipped` | StringName 集合 | 0–11 项 | calculated | 本局没吃到的菜；教学局不可为空 |
| `lowest_uv_row` | EatenRow \| ⊥ | 行 / `⊥` | calculated | 本局**每格最低**的一行；中性表述，不评价为「最亏」；空局为 `⊥` |
| `best_menu` | DishData \| ⊥ | 菜 / `⊥` | 菜品数值 | 本局没吃到的菜里每格最高的一道 |
| `uv_gap` | float64 | −7 – +7 | calculated | 每格差（元/格） |
| `remaining_at_end` | int | 0–`capacity_total` | 结算与入场费（`capacity_total − capacity_used`） | 收局时还剩多少格 |
| `best_menu_fit` | bool | {true, false} | calculated | 收局时那道「每格最高」的菜**是否还装得下** |
| `satiety(best_menu)` | int | 1–6 | 菜品数值 | 该道的基础饱腹（判定能否装下用基础值，不重算腻度） |

**Output Range:** `uv_gap ∈ [−7, 7]`（MVP 数值带）。文案分支按**先可行性、后比较**的顺序：

1. `rows = ∅` → 只给「这一局你没有吃」；
2. `skipped = ∅` → 「这一桌你都吃了，没有可换的」（防御分支：教学局六道不可行，见不变式 4）；
3. `¬best_menu_fit` → 「当时只剩 `remaining_at_end` 格，它需要 `satiety(best_menu)` 格——**装不下**；放弃它是容量决定的，不是选错」。**这是教学局的主分支**：设计上必须放弃的那道往往每格更高，若把它说成「更值、下次先拿」就是自相矛盾。
4. `best_menu_fit ∧ uv_gap > 0.5` → 「没吃到的那道每格更高，但两道菜的格数不同，不能等价替换，也不代表换掉后总分更高」；
5. `best_menu_fit ∧ uv_gap ≤ 0.5` → 「你吃到的每格已经接近或更高，选对了」。

**禁用措辞**：任何分支都不得把 `lowest_uv_row` 称作「最亏」「白吃」「浪费」，也不得给出「下一局先拿它」式的责备——教学局的最优解本来就是放弃一道，收据不能把玩家的正确决策说成错误。该约束由文本表 lint 检查（关键词：最亏 / 白吃 / 浪费 / 下次先拿）。

**这是一条「每格比较」，不是增量收益式**：替换一道菜会重算整局的腻度与容量，单对差值不等于换掉后的净利差。因此文案只说「每格差」，并补一句「两道菜的格子数不同」；**本系统不声称可以等价替换，也不给出替换后的总分**。这是刻意的：给出总分等于泄露最优解（菜品数值规则 11、局内 HUD 规则 9 的同一理由）。

**Example:** 本局吃了五道、放弃价格最低的那道，收局时只剩 1 格，而放弃的那道需要 2 格 → `best_menu_fit = false` → 收据写「当时只剩 1 格，麻婆豆腐需要 2 格——装不下；放弃它是这一桌的容量决定的，不是选错」，`uv_gap` 只作为逐行数据展示，不写成替换建议。若另一局收局时还剩 2 格（`fit = true`）且 `uv_gap > 0.5`，文案只说明「每格更高、格数不同、不能等价替换」，不给替换后的总分。

## Edge Cases

- **If `pool_override` 被菜单波次忽略或校验失败**：本局按档口原菜池进行；B0/B1 不变，B2 的「六道菜 / 20 格」改为读取实际可见数并省略定量句；B3 只在真实出现重复组时触发。不报错、不中止开局。
- **If `entry_fee_override` 不被支持**：本局仍按档口入场费扣款；B0 改说「这一局的入场费是 N 元，回本线就是它」，其余 beat 与 B4 不变。记录一条 warning 供验收发现契约缺失。
- **If `is_first_run = true` 但存档已有 `runs_settled > 0`**：仍按首局运行（本系统只消费标记），不读存档判断；不产生错误。上线前应由组装方修正标记来源。
- **If 载入结果为 `Recovered`**（主档损坏、从备份恢复，而备份早于第一次结算）：`LoadReport.first_run_eligible = false`，组装方必须给出 `is_first_run = false`，即使备份里的 `runs_settled = 0` 也不得重播教学（S11）。本系统不读存档；若组装方仍传 `true`，按上一条处理（照常运行、不报错），但这是组装方缺陷，不是本系统的降级路径。
- **If 首局被放弃、退出或故障**：立即弹出 `Blocked`、拆除遮罩与气泡、不产出 `B4`、不广播 `onboarding_finished`、不写任何东西。下一局仍是首局是合法结果。
- **If 气泡打开期间玩家暂停**：`Paused` 生效时输入栈被清空，`Blocked` 失效；恢复回 `Run` 时本系统重新压入同一白名单。**若重新压入返回 `false`**：不重试，把当前 beat 降级为无遮罩的停靠提示，玩家可以直接操作——绝不出现「气泡还在但按键无效」的卡死。
- **If 玩家在同一帧连按两次 `ui_accept`**：只推进一个 beat；第二个事件被丢弃。B0 与 B1 都在推进前调用 `输入.consume_action(ui_accept)`，HUD 的 `submit_action` 在同一帧拿不到该动作，因此不会误触一次吃菜。
- **If 玩家用鼠标：焦点为空**：B1 的推进条件要求 `focused_dish_id ≠ ⊥`；气泡文案提示「把鼠标移到一道菜上，或用方向键选一道」。不自动替玩家选菜、不因无焦点超时。
- **If HUD 尚未完成首帧布局，锚点矩形为空**：本系统在 `run_started` 同帧压 `Blocked`（防止误操作），但把气泡绘制推迟到锚点可用的第一帧，最多等 2 帧；仍为空则按 `bubble_rect` 的底部停靠。
- **If 玩家第一波直接跳过、最后一波结束取餐、一道没吃**：B2 在进入 `Coaching` 时已展示；B3 未触发、不进入必需集合；B4 在 `run_settled` 后交付空局文案（「这一局你没有吃」），不给替换建议；结算的零行收据照常显示。
- **If 局在 B1 期间结束**：`Blocked` 下无法提交动作，局不可能因玩家动作结束；若因外部故障进入 `Ending`/`Faulted`，立即拆除引导、不产出 `B4`。
- **If 收据没有出现**（`Run → Menu` 交接失败、`init_failed`）：`B4` 不交付，本系统在 `run_discarded` / `run_faulted` 时静默收尾；不补弹窗、不广播完成。
- **If 文案在本地化后超过 48 字**（150% 字号的最坏情况）：使用文本表里的短版本；仍超限则在生成期 lint 失败，**不在运行时截断、不允许缩小字号到 16 px 以下**。缺条目时回退到短版本，短版本也缺则跳过该 beat 并记 ERROR——气泡缺文案不得卡住玩家。
- **If `skipped = ∅`（本局吃满了菜池）**：`replacement_hint` 不产出，收据只显示逐行单位价值与「这一桌你都吃了」变体。
- **If 窗口小于 1280×720**：按最低支持分辨率处理，`w` 收缩到 `W − 2m`，受保护矩形优先于气泡完整显示——宁可气泡换行，不可压住回合/容量/净利。
- **If 玩家在局内跳过了引导**：`B4` 不产出，收据只显示结算自己的行与合计，不显示讲解；界面上没有单独的「跳过讲解」开关——跳过只有局内那一次。
- **If 菜品数值查询在 `B4` 失败**：`FirstRunExplanation` 降级为纯文案，不显示逐行比值与替换建议；收据本身照常显示结算给出的行值。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 餐局流程 | 硬 | `RunConfig.is_first_run`；`run_started` / `action_committed` / `run_ended` / `run_discarded` / `run_faulted`；只读快照 | GDD 已写（`design/gdd/run-flow.md`） |
| 局内 HUD | 硬 | `get_anchor_rect(anchor_id)`、`focused_dish_id`、当前波可见数与波序；气泡叠在 HUD 之上 | GDD 已写（`design/gdd/in-game-hud.md`） |
| 输入 | 硬 | `push_context(Blocked, allow=[…])` / `pop_context()`；查询动作 | GDD 已写（`design/gdd/input.md`） |
| 场景与流程管理 | 硬 | 只读顶层状态以判断局是否已离开；不请求切换 | GDD 已写（In Review，`design/gdd/scene-flow.md`） |
| 菜品数值 | 硬 | `price` / `satiety` / `cloying_group` / `unit_value`；只读、不重算 | GDD 已写（Approved，`design/gdd/dish-values.md`） |
| 结算与入场费 | 硬（`B4`） | `RunResult.rows[]` 的静态与实际单位价值；`entry_fee` / `net_profit` | GDD 已写（`design/gdd/settlement-and-entry-fee.md`） |
| 腻度与口味 | 软 | `focused_dish_id` 所属 `cloying_group` 的当前计数（`group_count(g)`，与 `extra_satiety` 同口径）与 `extra_satiety`；供 B3 触发与文案；**只读，不写计数** | GDD 已写（`design/gdd/cloying-and-taste.md`） |

> **W2 修正**：`腻度与口味` 原先只列在下游表，但 `B3` 的触发条件直接读它的 `group_count(g)`、文案也读 `extra_satiety`——这是**上游读取**，不是下游。现已移入上游表；`systems-index.md` 第 18 行的 `Depends On` 仍需补上 `腻度与口味`（见 Cross-References）。

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 界面流程 | 硬（UI） | 取首局预置；渲染 `FirstRunExplanation`；`is_first_run` 只按 `records.career.runs_settled == 0 ∧ LoadReport.first_run_eligible` 判定，`onboarding_finished` 不参与；首局菜单显示 40 而不是 85（S9） | GDD 已写（`design/gdd/ui-flow.md`） |
| 菜单波次 | 软 | 接收并应用 `pool_override`（当前只读 `StationData.dish_pool`，见 Open Questions） | GDD 已写（`design/gdd/menu-waves.md`） |
| 局外升级 | 软（仅文案） | `B4` 的首笔升级建议读目录与钱包，只出文字 | GDD 已写（`design/gdd/meta-upgrades.md`） |
| 音效反馈 | 软 | 订阅 `onboarding_beat_shown` / `onboarding_finished` | GDD 已写（`design/gdd/audio-feedback.md`） |
| 存档 | 无直接调用 | 不读写任何命名空间 | GDD 已写（`design/gdd/save-system.md`） |

**`is_first_run` 的三态真值表（推荐给组装方，本系统不实现）**：

计算式：`is_first_run = (career.runs_settled == 0) ∧ LoadReport.first_run_eligible`。

| 首局结局 | `career.runs_settled` | `LoadReport.first_run_eligible` | 下一局 `is_first_run` | 理由 |
|---|---|---|---|---|
| 结算（含跳过引导后结算） | +1 | `true`（`Loaded`） | `false` | 一局已经完整发生，教学目的已达成 |
| 跳过引导后**放弃** | 不变 | `true` | `true` | 放弃不结算；玩家还没见过完整的首局 |
| 放弃 / 退出 / 故障 | 不变 | `true` | `true` | 同上 |
| 完成引导但结算前退出 | 不变 | `true` | `true` | 同上 |
| 主档损坏、从备份恢复（`Recovered`） | 备份里可能是 0 | **`false`** | `false` | 备份可能早于第一次结算；单看 `runs_settled = 0` 会把老玩家塞回教学（S11） |
| 主档缺失（`Fresh`） | 0 | `true` | `true` | 真正的第一次启动 |

**不属于依赖的关系**：

- **存档 / 纪录与里程碑**：本系统不读 `career.runs_settled` 或 `LoadReport`、不写任何命名空间。`is_first_run` 由界面流程组装 `RunConfig` 时决定；本 GDD 只给出推荐规则（`runs_settled == 0 ∧ LoadReport.first_run_eligible`，见上表），不把它变成本系统的依赖。`first_run_eligible` 的定义归存档（S11）。
- **时间与回合 / 胃容量**：本系统只读快照里的 `turns_remaining` / `capacity_used` / `capacity_total` 用于文案，不调用 `spend_turn` / `consume`。经餐局流程的只读快照，不另记依赖。
- **数据配置**：六道菜 id 是 `station_canteen.dish_pool` 的子集，由菜单波次与菜品数值解析；本系统不直接查数据目录。

**双向一致核对**：

- `run-flow.md`、`ui-flow.md` 的交互表已回指本系统；`audio-feedback.md` 目前未定义 `onboarding_beat_shown` / `onboarding_finished`，因此没有本系统行——静音是合法降级（见 Visual/Audio），若其后续定义这两个事件，需回指本系统。
- `menu-waves.md`、`meta-upgrades.md`、`settlement-and-entry-fee.md` 已把本系统列为软下游；`cloying-and-taste.md` 把本系统列为下游，本文件则**按上游消费它的 `group_count(g)` 与 `extra_satiety`**（W2 已移入上游表），两个方向同时成立且不构成环。
- **三项契约缺口均已关闭**：`input.md` 已有 `push_context(context, allow := [])` 与 `consume_action`；`run-flow.md` 已有 `RunConfig.entry_fee_override`；`menu-waves.md` 的 `initialize` 与健康检查已接受 `pool_override`。

## Visual/Audio Requirements

**本系统绘制的只有讲解层**（气泡、遮罩、锚点指引线、收据讲解的内容排版约束）；局内屏的其余部分归局内 HUD，收据屏归界面流程。

- **气泡**：圆角矩形卡片，底色为不透明深色（与 HUD 顶栏同一套主题色），正文 ≥ 16 px、对比度 ≥ 4.5:1（复用 HUD 的 `hud_contrast_ratio`），最多三行。出现 ≤ 100 ms 淡入，消失 ≤ 80 ms；不做打字机、不做逐字动画。
- **模态遮罩**：只压暗受保护区域之外的画面，压暗后仍能看清 HUD 布局；**回合、容量、净利/回本差额三个读数保持原亮度与原对比度**（餐局流程 UI Requirements 7）。
- **锚点指引**：气泡与被讲解控件之间画一条 1 px 指引线，颜色与气泡描边一致；锚点是菜品卡时，卡片保持原样，不加描边、不变色，避免与 HUD 的焦点样式混淆。
- **非模态提示**：无遮罩、无指引线，固定在安全区底部的详情行上方，宽度与 `bubble_rect` 同；不遮挡菜品卡。
- **收据讲解**：不绘图，只给界面流程一份带优先级的文本与两个高亮菜 id（本局**每格最低**的一行 + 全菜单每格最高的一道；后者只在 `best_menu_fit = true` 时作为替换建议出现，装不下时高亮仅用于对照、文案写明「装不下、放弃是对的」；空局或 `skipped = ∅` 时不给高亮）；高亮样式由界面流程决定。**不得使用「最亏」一类评价性高亮标签。**
- **音效**：本系统不播放、不合成任何声音；只发布 `onboarding_beat_shown` 与 `onboarding_finished` 供音效反馈映射。音效反馈 GDD 未定义这两个事件时，静音是合法降级。

> 📌 **Asset Spec** — Visual/Audio requirements are defined. After the art bible is approved, run `/asset-spec system:first-run-onboarding` to produce per-asset visual descriptions, dimensions, and generation prompts from this section.

## Game Feel

- **不抢节奏**：首局没有任何计时器、倒计时或自动推进；玩家停多久都不会错过任何东西，也不会有「再不看就没了」的压力（支柱 3）。
- **一按即走**：每一条模态气泡只消耗一次确认，按下到消失 ≤ 100 ms；玩家关闭气泡后，第一次取餐在下一帧就能提交，中间不插动画、不锁帧。
- **不打断心算**：非模态提示不挡点击、不抢焦点、不移动菜品卡；它出现在底部详情行上方，玩家的视线可以留在价格与饱腹上。
- **讲解跟着焦点走**：B1 的气泡锚定在玩家当前聚焦的卡片上，玩家换卡时气泡重新贴到新卡（不重播文案、不重复计数）——玩家看哪里，讲解就在哪里。
- **收尾不邀功**：完成时不发奖励、不弹「教学完成」横幅、不播放庆祝音效；玩家直接进入收据，第一局的成果由数字自己说话。

## UI Requirements

- **气泡组件**：单实例，同一时刻最多一条；状态为 `hidden / modal / contextual`。组件只接受「锚点 id + 文本 + 行数」，自己完成 `bubble_rect` 与 `bubble_text_cap` 的计算，不接受绝对坐标。
- **锚点契约**：`get_anchor_rect(anchor_id) -> Rect2` 由 HUD 实现，六类 id 见 Core Rule 11；返回空矩形表示该锚点当前不可用，调用方按降级路径处理。**本系统不缓存锚点矩形**，每次展示时重新取——窗口缩放、150% 字号、超宽屏都由 HUD 重新布局。
- **焦点**：本系统不设焦点、不移焦点、不清焦点；只读 `focused_dish_id`。`B1` 期间焦点由 HUD 正常维护，焦点顺序仍是 HUD 的屏幕位置顺序。
- **受保护区域**：三个读数区域由 HUD 提供矩形，本系统只做相交判定。任何布局模式下（含 150% 字号、超宽屏）都必须可判定；取不到时按不相交处理并记断言失败。
- **文本表**：`onboarding_text` 按 `beat_id` + 场景分支索引，含最短版本（≤ 48 字）与完整版本；本地化只替换表内容，不改结构。调试文本与正式文本分开。
- **跳过入口**：模态气泡的右下角固定一个「跳过引导」文字按钮，键盘可达（`ui_cancel` 同义）；非模态提示不提供跳过。
- **无障碍**：全流程只用键盘/手柄可完成；鼠标是可选加速器（输入规则 4）。颜色不是唯一通道：高亮同时用文字标签。150% 字号下气泡最多三行，超出即文案缺陷。
- **收据讲解**：以「两段文字 + 两个菜 id」的 payload 交付，界面流程决定排版；本系统不假设收据的滚动、折叠或分页。

> **📌 UX Flag — 首局引导**：This system has UI requirements. In Phase 4 (Pre-Production), run `/ux-design` to create a UX spec for the tutorial layer (bubble, spotlight mask, anchor callouts, receipt explanation) **before** writing epics. Stories that reference this UI should cite the matching `design/ux/` spec, not the GDD directly.

## Cross-References

| 文档 | 关系 | 本文件如何使用它 |
|---|---|---|
| `design/gdd/run-flow.md` | 上游 | `is_first_run`、`pool_override`、`run_*` 信号与只读快照；本系统不改变 `run_state`、结束条件与生命周期 |
| `design/gdd/in-game-hud.md` | 上游 | 锚点矩形、`focused_dish_id`、卡片字段、受保护区域；本系统不绘制 HUD、不显示 `unit_value` |
| `design/gdd/input.md` | 上游 | `push_context(Blocked)` / `pop_context()`；本系统不改映射、不监听原始事件 |
| `design/gdd/scene-flow.md` | 上游 | 顶层状态与 `clear_stack` 行为决定暂停后是否需要重压上下文 |
| `design/gdd/dish-values.md` | 上游 | 六道菜的价格、饱腹、组；`B4` 的单位价值数据；本系统不重算比值 |
| `design/gdd/settlement-and-entry-fee.md` | 上游 | `RunResult.rows[]` 的静态/实际单位价值；本局 `entry_fee` 冻结进 `RunResult`；首局 40 元是合法的 `entry_fee`，不改任何求和规则 |
| `design/gdd/menu-waves.md` | 下游（软） | 接收 `pool_override`；六道菜按 `wave_size` 分波（默认 4 → 2 波）；「跳过即失去」的讲解口径 |
| `design/gdd/cloying-and-taste.md` | 上游（值来源） | `group_count(g)` 与 `extra_satiety` 是 B3 的触发与文案来源；只读，不写计数（W2 已从下游改为上游） |
| `design/gdd/meta-upgrades.md` | 下游（软） | `B4` 的首笔升级建议只读目录与钱包，不购买、不高亮 |
| `design/gdd/audio-feedback.md` | 下游（软） | 订阅 `onboarding_beat_shown` / `onboarding_finished` |
| `design/gdd/save-system.md` | 无直接调用 | 本系统不读写任何命名空间 |
| `design/gdd/game-concept.md` | 概念 | 「第一局只给 6 道菜，价格与饱腹直接可见；结算时展示单位价值」 |
| `design/gdd/systems-index.md` | 索引 | 第 18 行；索引已列本文件并标 `Approved`。**仍需在 `Depends On` 补 `腻度与口味`**（W2：`B3` 读其 `group_count(g)`）：第 74 行与第 187 行的依赖清单都要补，进度计数无需变化 |

## Acceptance Criteria

> 每条都可由 QA 在不读本 GDD 的情况下执行。UI 与视觉类同时要求 `production/qa/evidence/` 留档截图。

**首局识别与预置**

1. **GIVEN** 生涯 `runs_settled = 0`、界面流程组装 `RunConfig`，**WHEN** 开局，**THEN** `RunConfig.is_first_run = true`、`pool_override` 恰为六道菜、`entry_fee_override = 40`；`capacity_total = 20`、`turns_total = 10`、六道菜按 `wave_size = 4` 分成 4+2 两波；本系统在 `run_started` 同帧压入 `Blocked`。
2. **GIVEN** `RunConfig.is_first_run = false`，**WHEN** 开局并完成若干动作，**THEN** 本系统从未压入上下文、从未绘制气泡、从未广播 `onboarding_beat_shown`。
3. **GIVEN** 首局开局，**WHEN** 查看 `run_state` 与信号记录，**THEN** `run_state = Active`、`run_id` 未变、`run_started` 只广播一次、`run_ended` 未因引导广播。**截图留存**。
4. **GIVEN** 首局按 `first_run_pool_invariants` 的最优解取五道，**WHEN** 查看收据，**THEN** `total_value` 等于五道价格之和（当前数据为 82）、`net_profit = total_value − 40`（当前 `+42`）、回本率 = `total_value / 40`（当前 `205.0%`、`return_percent_x10 = 2050`），钱包足以买下最便宜的容量升级（当前 12 元的「加宽胃袋」）。数值随 `DishData` 重算，不变式 6 必须仍成立。

**Beat 序列**

5. **GIVEN** 首局开局首帧，**WHEN** 画面呈现，**THEN** B0 气泡打开、锚定顶栏净利位，**回合、容量、净利三个读数仍可见且未被遮罩压暗**。**截图留存**。
6. **GIVEN** B0 打开，**WHEN** 玩家按 `ui_accept`，**THEN** 进入 B1，B0 只出现一次；`turns_remaining`、`capacity_used` 与 `eaten_count` 均不变，餐局流程未收到该次确认。
7. **GIVEN** B1 打开且无焦点，**WHEN** 玩家按 `ui_accept`，**THEN** 不推进、不吃菜，气泡提示需要先选一道菜。
8. **GIVEN** B1 打开且有焦点，**WHEN** 玩家按 `ui_accept`，**THEN** 进入 `Coaching`、`eaten_count` 不变、餐局流程未收到该次确认。
9. **GIVEN** 进入 `Coaching`，**WHEN** 画面稳定，**THEN** B2 作为非模态提示显示在底部，不挡任何菜品卡、不改变输入上下文。**截图留存**。
10. **GIVEN** `Coaching` 且 B3 未展示，**WHEN** 焦点移到一道同组计数 ≥ 1 的菜上，**THEN** B3 替换 B2 显示且只显示一次。**截图留存**。
11. **GIVEN** B2 或 B3 显示中，**WHEN** 玩家直接提交一次吃菜，**THEN** 提示消失、动作正常提交、回合与容量按规则变化。
12. **GIVEN** 整局没有任何同组计数 ≥ 1 的焦点时刻，**WHEN** 结算，**THEN** B3 不进入必需集合，完成判定不因缺 B3 而失败。

**约束与降级**

13. **GIVEN** 任一模态 beat 打开，**WHEN** 玩家按导航键（B0）或 `ui_accept`（B1），**THEN** 除了白名单内的推进外，`ui_accept` / `game_skip_wave` 不到达餐局流程，无回合被消耗。
14. **GIVEN** 局内任意时刻，**WHEN** 检查全部气泡文本，**THEN** 不存在「元/格」「每格 N 元」或任何 `unit_value` 数值；单位价值只在收据中出现。
15. **GIVEN** HUD 不返回 `bar:capacity` 的锚点矩形，**WHEN** B2 展示，**THEN** 气泡停靠底部、仍不与三个受保护区域相交、不报错中断。**截图留存**。
16. **GIVEN** `pool_override` 被上游忽略，**WHEN** 首局开局，**THEN** 本局用档口原菜池照常进行，B2 不含「六道菜 / 20 格」的定量句，无错误弹窗。
17. **GIVEN** `entry_fee_override` 未被支持，**WHEN** 首局开局，**THEN** B0 显示实际入场费文案，其余 beat 与收据讲解不变，并记录一条 warning。

**跳过、暂停、异常**

18. **GIVEN** 模态 beat 打开，**WHEN** 玩家按 `ui_cancel`，**THEN** 遮罩与气泡立即消失、`Blocked` 弹出、`onboarding_finished(SKIPPED)` 广播一次、本局剩余引导与收据讲解均不出现。
19. **GIVEN** 模态 beat 打开，**WHEN** 玩家按 `ui_pause` 再恢复，**THEN** 恢复后同一 beat 重新压入 `Blocked`；若重压失败，beat 降级为无遮罩提示且玩家可正常操作，无按键无效的死锁。
20. **GIVEN** 首局在 B0/B1 期间被放弃或发生 `run_faulted`，**WHEN** 回到菜单，**THEN** 无遮罩残留、无 `onboarding_finished`、存档逐字段不变，下一局仍标记为首局。
21. **GIVEN** 玩家同一帧连按两次 `ui_accept`，**WHEN** 输入被处理，**THEN** 只推进一个 beat，不出现跳级或重复计数。
22. **GIVEN** 首局正常结算，**WHEN** 收据展示，**THEN** 每条吃下的菜显示单位价值，且只给出一处替换建议；改为放弃后重开，**THEN** 无收据、无 `B4`、钱包不变。

**收据讲解**

23. **GIVEN** 本局吃了五道、放弃一道，**WHEN** 收据讲解生成，**THEN** `lowest_uv_row` 与 `best_menu` 被正确选出；`uv_gap` 可手工复算，且文案按 `best_menu_fit` 与 `uv_gap` 走对应分支（装不下 / 每格更高但格数不同 / 每格已接近或更高）；任何分支都不出现「最亏」「白吃」「浪费」「下次先拿」。
24. **GIVEN** 空局（一道没吃），**WHEN** 收据展示，**THEN** 显示空局文案、不给替换建议、不崩溃。
25. **GIVEN** 150% 字号与 1280×720，**WHEN** 每条气泡展示，**THEN** 正文不超过三行、不缩小到 16 px 以下、不与受保护区域相交。**截图留存** 三个分辨率（1280×720、1920×1080、超宽屏）。
26. **GIVEN** 任一模态 beat 打开并静止 30 秒且无输入，**WHEN** 观察，**THEN** 气泡不消失、不推进、不扣回合、不播放动画；之后按 `ui_accept` 仍正常推进。
27. **GIVEN** 任一模态 beat 打开，**WHEN** 只用键盘/手柄操作，**THEN** 「跳过引导」可达（`ui_cancel` 或焦点到达文字按钮），不出现只能靠鼠标跳过的状态。
28. **GIVEN** 三条独立的生涯路径——正常结算、跳过引导后结算、直接放弃，**WHEN** 各自重开，**THEN** `is_first_run` 依次为 `false / false / true`（跟随 `runs_settled` 与 `LoadReport.first_run_eligible`）；本系统不写任何持久化，结果与它无关。
29. **GIVEN** 本局吃满了菜池里的每一道（`skipped = ∅`），**WHEN** 收据讲解生成，**THEN** 不产出替换建议、显示「这一桌你都吃了」变体、不崩溃。

**跨系统一致性（S9 / S10 / S11）**

30. **GIVEN** 首局收局时只剩 1 格、被放弃的那道需要 2 格，**WHEN** 收据讲解生成，**THEN** `best_menu_fit = false`，文案说明「装不下、放弃是容量决定的」；两个高亮 id 仍给出，但注明为对照而非替换建议；**AND** 全文不含「最亏 / 白吃 / 浪费 / 下次先拿」。
31. **GIVEN** 生涯已有 `runs_settled ≥ 1`，主档损坏、从备份恢复（`Recovered`）且备份里 `runs_settled = 0`，**WHEN** 载入并组装 `RunConfig`，**THEN** `LoadReport.first_run_eligible = false`、`is_first_run = false`，本系统不初始化、不显示任何气泡，玩家直接进入正常菜单。
32. **GIVEN** `is_first_run = true`（`runs_settled = 0`、`first_run_eligible = true`），**WHEN** 查看主菜单开始按钮旁的入场费，**THEN** 显示 40，与本局 `RunConfig.entry_fee_override` 一致，不显示 85；**GIVEN** `entry_fee_override` 缺失而降级，**THEN** 菜单与本局都显示 85，两者仍一致并记录 warning。

## Open Questions

| # | 问题 | 建议 | 负责人 | 何时解决 |
|---|---|---|---|---|
| 1 | 首局教学价 40 怎么落到 `RunConfig`？**已回答（2026-11-04）**：`RunConfig.entry_fee_override` 已定义。 | — | 编排会话 + 餐局流程 / 结算 | 已关闭 |
| 2 | 菜单波次只读 `StationData.dish_pool`，`RunConfig.pool_override` 目前无处落地。**已回答（2026-11-04）**：`initialize(...)` 与健康检查已接受 `pool_override`。 | — | 编排会话 + 菜单波次 | 已关闭 |
| 3 | `Blocked` 上下文只放行 `ui_pause`。**已回答（2026-11-04）**：`push_context` 已有 `allow` 白名单，`consume_action` 解决确认键争用。 | — | 编排会话 + 输入 | 已关闭 |
| 4 | 收据讲解由界面流程渲染还是由本系统覆盖层渲染？**已回答（2026-11-04）**：界面流程渲染 `FirstRunExplanation`（含教学价说明、替换建议、首笔升级建议），订阅 `onboarding_finished`；本系统不假设收据锚点。 | 本系统只交付 `FirstRunExplanation` payload。 | 界面流程 | 已关闭 |
| 5 | 「跳过/完成」是否需要持久化，而不只是由 `runs_settled == 0` 推断？ | MVP 先用 `runs_settled == 0 ∧ LoadReport.first_run_eligible`：完成一局（含跳过后的那一局）自动结束首局；放弃则下次仍是首局；`Recovered` 一律不重播（S11）。若试玩反馈「跳过还要再看」不可接受，再加一个 `onboarding` 命名空间。 | 界面流程 + 存档 | Vertical Slice 试玩后 |
| 6 | 六道菜池是否应该让容量刚好卡住（本设计）还是留有余量（全都能吃）？ | 采用卡住：只有卡住才会出现一次真实取舍，教学才有对象。代价是玩家必须放弃一道，收据讲解用「装不下」而非「选错了」的口径。 | systems-designer | 试玩验证 |
| 7 | `B4` 的「首笔升级建议」是否要在收据上高亮按钮？ | 只出文字、不高亮、不预选。高亮会把「推荐」变成「默认选择」，与局外升级的「不自动购买」边界冲突。 | 界面流程 + 局外升级 | `/ux-design` 收据屏时 |
| 8 | 超宽屏下 `w = clamp(0.36 × W, 280, 440)` 是否够读？ | 先按 440 px 封顶，避免气泡在 4K 上变成一条横贯屏幕的横幅；若试玩觉得字太少再调上限。 | ux-designer | `/ux-design` 时 |
| 9 | **首局注意力预算与教学覆盖的取舍（D6）**：一屏约 8 个可读信息流，首局只教价格 / 基础饱腹 / 容量 / 同组重复四个概念，`price/effective_satiety` 比值与腻度完整规则不教。是否需要在后续局补一条「每格」提示，或把 B1 拆成更细的步骤？ | 本版只做**措辞级**澄清：B1 明确要求「价格与饱腹并排读」，不引入比值、不加数字、不加 beat。扩展或收缩教学范围交给 `/prototype` 观察与 `/ux-design`，不在本版决定。 | systems-designer + ux-designer | `/prototype` 后 |
