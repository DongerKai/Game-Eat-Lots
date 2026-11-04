# 胃容量（Stomach Capacity）

> **Status**: In Design
> **Author**: user + Claude（autonomous 模式；systems-designer 复核规则与公式，qa-lead 复核验收标准，两份意见已并入）
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 每口都是算术题 · 没有白吃的菜

## Summary

胃容量是一局里“还能吃多少”的唯一账本：开局按基础容量与局外升级快照算出总格数，每吃下一道菜就扣掉它的实际饱腹，并在入口处判定一道菜是否还放得下。它只做闸门，不结束本局，也不计算饱腹——玩家看到的每一次“吃得下 / 吃不下”都来自同一份计数。

> **Quick reference** — Layer: `Core` · Priority: `MVP` · Key deps: `餐局流程, 数据配置`（快照来自 `局外升级`；`effective_satiety` 来自 `腻度与口味`）

## Overview

胃容量是《回本》里“有限空间”的那一半：一局只有这么多格，每道菜按它的实际饱腹占用格数，放不下就是放不下。它把菜单上的价格与饱腹翻译成一个玩家每口都要回答的问题——“这一盘还塞得进去吗？”——并且保证这个问题的答案在一局内始终一致、可复盘。

它不拥有任何一个菜品的数字。基础饱腹属于数据配置与菜品数值，重复追加属于腻度与口味，回合属于时间与回合，一局的生命周期与结束条件属于餐局流程。本系统只拥有三件事：本局的总容量、本局已经用掉的容量、以及“这道菜还放不放得下”的判定。它把容量变成一个可查询、可记录、不可回退的账本。

没有它，“吃不下”会散落在菜品卡、结算与 HUD 三处各自判断，玩家会看到三个不一致的剩余格数；重开与放弃也没有统一的清零点。它主要服务支柱 1「每口都是算术题」——容量是算术题的分母之一；也服务支柱 2「没有白吃的菜」——每一格都被占用，且占掉的格不会退还。

## Detailed Design

### Core Rules

1. **职责边界**：本系统拥有**一局的容量上限、已用容量计数、可吃性判定**，以及容量状态在一局开始时的重置。它不拥有 `effective_satiety`（腻度与口味）、`satiety`（数据配置 / 菜品数值）、回合（时间与回合）、可见菜与焦点（菜单波次 / 局内 HUD）、结束条件（餐局流程）、等级到加成的换算（局外升级）或净利与浪费容量的展示（结算与入场费）。
2. **一局一份容量状态**：餐局流程在每局的 `Initializing` 中**恰好调用一次** `initialize(upgrade_snapshot)`，且在 `run_started` 之前完成。`initialize` 把 `capacity_used` 置 0，并按快照算出 `capacity_total`。**同一局内**第二次调用是契约违例，进入 `Faulted`，不得静默重置——静默重置会抹掉已用容量。跨局的第二次调用是正常重开，不是违例。**局边界触发器**：本系统订阅餐局流程的 `run_ended` 与 `run_discarded`；收到任一时立即回到 `未初始化`（丢弃本局容量）。因此下一次 `initialize` 必然是新局，`initialize` 本身不需要携带 run identity。
3. **容量按整格计**：容量与 `effective_satiety` 都是整数格，没有小数、没有四舍五入。`effective_satiety` 由腻度与口味给出且恒 ≥ 1；本系统只消费这个值，**不重算、不缓存、不改写**。
4. **上限来自快照，不来自等级**：`capacity_total = C0 + capacity_bonus`。`C0` 是基础容量（数据配置提供的平衡常数，MVP = 20）。`capacity_bonus` 是局外升级在开局前算好的非负整数，本系统只相加。快照缺少 `capacity_bonus`、值为负数、`null` 或非整数时，初始化失败——**不缺省为 0、不钳制**。
5. **可吃性是唯一闸门**：`can_fit(e) ⇔ e 是 ≥ 1 的整数 ∧ capacity_used + e ≤ capacity_total`。边界**包含等号**：恰好填满最后一格是允许的（104 元最优餐正好占满 20 / 20，改成严格小于会让校准最优不可达）。
6. **容量不结束本局**：本系统没有“吃撑”结果，也不产生任何结束原因。已用容量等于上限、或当前可见菜都放不下时，餐局流程保持 `Active`；玩家可以付费跳波、等下一波，或用掉剩余回合。**“吃不下”是展示层的判断，不是本系统的状态，也不是第三种结束原因。**
7. **已用容量单调不减**：一局内 `capacity_used` 只增不减。没有退还、没有消化、没有释放接口。被拒绝的尝试不改变任何状态；已付的腻度额外占用不因后续解腻而退回（与「腻度与口味」的规则一致）。
8. **`consume(e)` 是唯一的写入口**：餐局流程在一次动作完整校验通过后，对每次成功吃菜调用一次 `consume(e)`。它先校验 `e` 是 ≥ 1 的整数、且 `capacity_used + e ≤ capacity_total`，全部通过才原子增加；任一失败返回结构化错误且**不改变任何字段**。`e < 1` 或非整数时绝不允许通过——否则 `consume(-1)` 会减少已用容量，直接破坏规则 7。
9. **判定是纯的，不预留**：`can_fit` 只读，不锁、不预留、不改变状态；同一状态重复调用结果相同。餐局流程在同一帧内完成“判定 → 提交”，单线程下两者之间没有其他写入者，因此不需要预留机制。
10. **上限有结构边界**：`capacity_total` 的有效范围是 `[1, 999]`。上界与结算 GDD 注册的 `capacity_used ∈ [0, 999]` 对齐；超过 999 是数据错误，初始化失败，**不钳制**。局外升级必须在**购买时**保证 `C0 + 最大可能加成 ≤ 999`，否则玩家会做出一个永久无法开局的存档——这个检查不能只留在开局。
11. **快照在一局内冻结**：一局开始后容量不随升级、数据配置或其他状态变化。局外升级只能在局间生效；同一局内改写 `UpgradeData` 或快照不影响本局 `capacity_total`，下一局重新 `initialize` 才生效。
12. **只读暴露**：对外只提供 `capacity_total`、`capacity_used`、`capacity_remaining` 与 `can_fit`。餐局流程把前两者写入 `RunLog` 与只读快照；局内 HUD 与结算只读。任何系统都不得直接写已用容量，也不得自行维护第二份容量计数器。
13. **确定性**：给定 `C0`、`capacity_bonus` 与按顺序提交的 `effective_satiety` 序列，`capacity_total`、`capacity_used` 与每次 `can_fit` 的返回值完全确定；不使用时钟、随机数或遍历顺序。
14. **不读取菜品数据**：运行时本系统只接收 `effective_satiety` 参数，不查询 `DishData`、不按菜名重放、不依赖菜单内容。注入空菜单或与菜品目录不一致的夹具时，容量仍能独立工作。

### States and Transitions

容量没有自己的状态机；它的生命周期完全由餐局流程驱动。下表的“状态”只是对同一份账本的阶段描述，**不广播任何信号，也不参与结束判定**。

| 阶段 | 进入条件 | 允许的写 | 离开 |
|---|---|---|---|
| `未初始化` | 应用启动 / 收到 `run_ended` 或 `run_discarded` | 仅 `initialize` | 成功 → `就绪`；失败 → `初始化失败`（餐局流程 `Failed`） |
| `就绪` | `initialize` 成功，`used = 0` | `consume` | 本局结束 / 放弃 → `未初始化` |
| `已满` | `used = total`（**派生判断，不是独立状态**） | 本局 `consume` 全部拒绝；仍接受下一局的 `initialize` | 同 `就绪` |
| `初始化失败` | 输入非法 / 上限越界 / 同局重复初始化 | 无（交餐局流程进入 `Failed` 或 `Faulted`） | 下一局 `initialize` |
| `已丢弃` | 餐局流程放弃或退出 | 无 | 下一局 `initialize` |

- `已满` 与 `已丢弃` 都不是结束判定：本系统不产生 `end_reason`，也不调用结算。
- 暂停/冻结（`frozen = true`）不改变容量，也不阻止读取；恢复后数值与冻结前逐字段一致。
- 重开是全新一局：新的 `initialize` 把 `used` 置 0，总容量按新快照重算，上一局的任何字段都不延续。

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 餐局流程 | 上游 | `initialize(upgrade_snapshot) → capacity_total \| 结构化错误`；`can_fit(e)`；`consume(e)`；读取 `capacity_total` / `capacity_used` / `capacity_remaining`；订阅 `run_ended` / `run_discarded` 作为局边界 | 硬 |
| 数据配置 | 上游 | `BalanceData.base_capacity`（`C0`，MVP 20）；**不读取 `DishData`** | 硬（字段缺口见 Open Questions 1） |
| 局外升级 | 上游（经快照） | `upgrade_snapshot.capacity_bonus: int ≥ 0`（已解析值）；购买时保证 `C0 + 最大加成 ≤ 999` | 硬（GDD 已写：`design/gdd/meta-upgrades.md`） |
| 腻度与口味 | 间接上游 | `effective_satiety` 经餐局流程传入；本系统不直接调用、不重算 | 硬（值依赖，无调用边） |
| 局内 HUD | 下游 | 只读 `capacity_total` / `capacity_used` / `capacity_remaining` / `can_fit`；不得写、不得自建计数器 | 硬（GDD 已写：`design/gdd/in-game-hud.md`） |
| 结算与入场费 | 下游 | 从 `RunLog` 读 `capacity_total`；校验 `capacity_used = Σ effective_satiety` | 硬 |
| 菜单波次 | 设计期下游 | `wave_reachable_opt` 的 `C` 参数；运行期不读 | 软（设计期） |
| 菜品数值 | 设计期下游 | `opt_value` / `calibration_ratio` 的 `C0` 审计参数 | 软（设计期） |
| 纪录与里程碑 | 下游（软） | 可存本局 `capacity_total` 用于展示；不参与判定 | 软（GDD 已写：`design/gdd/records-and-milestones.md`） |
| 存档 | 无直接调用 | MVP 无中途存档；容量是局内状态，不持久化 | 条件 |

**接口所有权**：本系统拥有 `capacity_total`、`capacity_used`、`capacity_remaining`、`can_fit` 与 `consume`；餐局流程拥有**何时调用它们、按什么顺序调用**；腻度与口味拥有 `effective_satiety`；局外升级拥有 `capacity_bonus` 的解析。下游不得自己推算剩余容量，也不得把 `capacity_used` 当作可变字段写回。

**与腻度 `meal_step` 的边界**：「腻度与口味」的 `meal_step` 第 2 步写的是 `remaining_capacity < satiety + e → 拒绝`，第 3 步扣减 `remaining_capacity`。这里的 `remaining_capacity` 就是本系统的 `capacity_remaining`，第 2 步就是本系统的 `can_fit`，第 3 步就是本系统的 `consume`。实现上**只有一个权威判定器**（本系统）；`meal_step` 是描述一次吃菜组合效果的伪代码，不是第二份实现。餐局流程按“取 `effective_satiety` → `can_fit` → 提交时 `consume`”的顺序编排两者，因此本系统与腻度之间没有直接调用边，也不构成依赖环。

**信号**：本系统不广播信号。容量变化由餐局流程的 `action_committed` 与只读快照体现；新增独立信号会与快照形成两份真相。

## Formulas

本系统定义四个公式与一条上游契约式。所有数值都是整数格，不含除法，不含钳制；任何一个失败都返回错误而不是改数。

### 本局总容量 `capacity_total`

The `capacity_total` formula is defined as:

`capacity_total = C0 + capacity_bonus`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `C0` | int | 1–999 | data file（`BalanceData.base_capacity`；MVP = 20） | 基础容量格数 |
| `capacity_bonus` | int | 0–(999 − `C0`) | `upgrade_snapshot.capacity_bonus`（局外升级解析） | 本局生效的容量加成，已解析、非等级 |
| `capacity_total` | int | 1–999 | calculated | 本局总容量格数 |

**Output Range:** `[1, 999]`。`C0 ≥ 1` 且 `capacity_bonus ≥ 0`，因此和恒 ≥ 1；`C0` 非法、`capacity_bonus` 不是整数、`< 0` 或键缺失时返回 `CAPACITY_INIT_INVALID`；和超过 999 时返回 `CAPACITY_TOTAL_OUT_OF_RANGE`。两种失败都**不钳制、不写入 `capacity_total`**，由餐局流程进入 `Failed`。

**Example:** `C0 = 20`、`capacity_bonus = 0` → `20`；`capacity_bonus = 3` → `23`；`C0 = 1`、`capacity_bonus = 998` → `999`（合法边界）；`capacity_bonus = 999` → `1000`，初始化失败。

### 已用容量 `capacity_used`

The `capacity_used` formula is defined as:

`capacity_used' = capacity_used + effective_satiety`，初始值 `capacity_used = 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `capacity_used` | int | 0–`capacity_total` | calculated（本系统运行期累加器） | 本局已占用格数 |
| `effective_satiety` | int | 1–9 | 腻度与口味在吃下时刻给出，经餐局流程传入（与 `RunLog.eaten[i].effective_satiety` 同值） | 本次吃菜占用的实际格数（基础饱腹 + 腻度追加） |
| `capacity_used'` | int | 0–`capacity_total` | calculated | 本次提交后的已用格数 |

**Output Range:** `[0, capacity_total]`，且一局内单调不减。若 `capacity_used + effective_satiety > capacity_total`，`consume` 返回 `CAPACITY_OVERFLOW` 且不改变任何字段——**不钳到上限**。结算 GDD 的 `capacity_used(R) = Σ effective_satiety(d_i)` 是本式在 `RunLog` 上的重算，用于校验；运行期累加值以本系统为准，两者不相等时结算返回 `CAPACITY_MISMATCH`。

**Example:** 依次吃 3 格、1 格、4 格 → `capacity_used` 为 `0 → 3 → 4 → 8`。

### 剩余容量 `capacity_remaining`

The `capacity_remaining` formula is defined as:

`capacity_remaining = capacity_total − capacity_used`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `capacity_total` | int | 1–999 | 本系统 | 本局总容量 |
| `capacity_used` | int | 0–`capacity_total` | 本系统 | 本局已占用 |
| `capacity_remaining` | int | 0–`capacity_total` | calculated | 还能放的格数 |

**Output Range:** `[0, capacity_total]`。`capacity_remaining = 0` 时本局“已满”，但**不结束**（规则 6）。该值是腻度 `meal_step` 的 `remaining_capacity`，也是 HUD 容量条的分母来源。

**Example:** 总容量 20、已用 15 → 剩余 5。

### 可吃性判定 `can_fit`

The `can_fit` formula is defined as:

`can_fit(e) = (e ∈ ℤ 且 e ≥ 1) 且 (capacity_used + e ≤ capacity_total)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `e` | int | 1–9（正常值域） | 调用方传入（餐局流程从腻度取得） | 待判定菜品的 `effective_satiety` |
| `capacity_used` | int | 0–`capacity_total` | 本系统 | 本局已占用 |
| `capacity_total` | int | 1–999 | 本系统 | 本局总容量 |
| `can_fit` | bool | `true` / `false` | calculated | 这道菜现在是否放得下 |

**Output Range:** `true` 或 `false`。**等号成立时返回 `true`**：`capacity_remaining = e` 允许吃下，正好填满最后一格。`e < 1` 或 `e` 不是整数时返回 `false` 并记一条 ERROR——正常流程不可能出现（腻度保证 `e ≥ 1`），这是防御性契约，**绝不放行“免费菜”**。`e > capacity_total` 是合法输入（这道菜整局都放不下），返回 `false` 且**不记错误**。

**Example:** 总容量 20、已用 15 → `can_fit(5) = true`、`can_fit(6) = false`；已用 20 → `can_fit(1) = false`，但本局继续。

### 升级加成 `capacity_bonus`（上游拥有，仅引用）

The `capacity_bonus` formula is defined as:

`capacity_bonus = Σ_{u ∈ U_CAP} level(u) × effect_value(u)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `U_CAP` | id 集合 | 数据配置中的 `STOMACH_CAPACITY` 升级 | data file | 所有容量类升级 |
| `level(u)` | int | 0–`max_level` | 存档 / 局外升级 | 该升级当前等级 |
| `effect_value(u)` | int | 1–(999 − `C0`) | `UpgradeData.effect_value` | 每级增量；数据配置校验其必须为整数 |
| `capacity_bonus` | int | 0–(999 − `C0`) | `upgrade_snapshot` | 本局容量加成 |

**Output Range:** `[0, 999 − C0]`。**本式由「局外升级」拥有**；本系统只消费已解析的整数结果，不读等级、不读存档、不做乘法。若局外升级未来引入非线性或递减曲线，本式随上游改变，而 `capacity_total = C0 + capacity_bonus` 不变。局外升级必须在购买时保证任意可达组合都满足 `C0 + capacity_bonus ≤ 999`（规则 10）。

**Example:** 3 级 × 每级 1 格 → `capacity_bonus = 3` → `capacity_total = 23`。

## Edge Cases

- **If 一道菜的 `effective_satiety > capacity_total`**：`can_fit` 整局返回 `false`，该菜本局不可吃；不记错误、不结束本局。这是合法数据（例如低容量 + 高腻度追加的极端组合），不是 bug。
- **If `capacity_used = capacity_total`**：`can_fit(e)` 对任何 `e ≥ 1` 返回 `false`；本局保持 `Active`，不产生结束原因；玩家仍可付费跳波、等下一波或耗尽回合。
- **If `capacity_remaining = e`**：允许吃下，已用容量恰好到上限。等号是允许的，不允许改成严格小于。
- **If `e < 1` 或 `e` 不是整数**：`can_fit` 返回 `false` 并记 ERROR；`consume` 返回 `INVALID_SATIETY` 且不改变任何字段。绝不放行免费菜，也绝不让负值减少已用容量。
- **If `upgrade_snapshot` 缺少 `capacity_bonus`，或值为 `null` / 负数 / 非整数**：初始化失败 `CAPACITY_INIT_INVALID`；餐局流程进入 `Failed`。**不缺省为 0、不取绝对值、不四舍五入。**
- **If `C0` 缺失、`< 1` 或不是整数**：数据配置在启动校验时失败，游戏不进入 `Ready`，本系统不会被初始化。若绕过启动校验直接注入非法 `C0`，`initialize` 返回 `CAPACITY_INIT_INVALID`。
- **If `C0 + capacity_bonus > 999`**：`initialize` 返回 `CAPACITY_TOTAL_OUT_OF_RANGE`，进入 `Failed`，不钳制。购买侧的预防见规则 10 与 Open Question 2。
- **If 同一局内第二次调用 `initialize`**：返回 `CAPACITY_INIT_INVALID`，餐局流程进入 `Faulted`；`capacity_used` 保持不变。静默重置会抹掉本局已用容量，因此禁止。
- **If `consume` 被调用时 `capacity_used + e > capacity_total`**（无论调用方是否先走过 `can_fit`）：返回 `CAPACITY_OVERFLOW`，所有字段不变，餐局流程进入 `Faulted`。不钳到上限——钳制会把越界伪装成一次正常吃菜。本系统无法识别“跳过 `can_fit` 但数值仍放得下”的调用；那种调用在账本上是合法的，重复提交由 `RunLog` 不变式捕获（见下一条）。
- **If 同一次吃菜被提交两次**：第二次仍受本系统校验；若放得下，本系统无法仅凭自身识别重复提交。该错误由餐局流程在组装 `RunLog` 时用 `capacity_used = Σ effective_satiety` 的不变式捕获，或由结算返回 `CAPACITY_MISMATCH`。本系统不引入额外的提交令牌。
- **If 本局被放弃或退出**：收到 `run_discarded`（或正常结束的 `run_ended`）后立即回到 `未初始化`，容量状态直接丢弃，不结算、不写纪录；下一局 `initialize` 从 `used = 0` 开始。丢弃不需要额外清理调用——订阅回调即触发。
- **If 暂停 / 冻结**：容量不变化、不读时钟、不推进；恢复后逐字段一致。
- **If 有剩余回合但当前可见菜都放不下**：本系统只如实返回 `can_fit = false`；不结束、不替代玩家做决定。HUD 可显示“吃不下”提示，但那不是状态。
- **If 本局一道菜也没吃**：`capacity_used = 0`、`capacity_remaining = capacity_total`；结算的 `wasted_capacity = capacity_total`，正常结算。
- **If 在同一状态重复查询 `can_fit`**：返回值相同，`capacity_used` 不变（无预留）。
- **If 在 `initialize` 之前读取容量**：返回 `0` 并记 ERROR；正常流程不会发生（HUD 只在本局 `Active` 后读取）。
- **If `upgrade_snapshot` 含本系统不认识的键**：忽略，保持前向兼容。
- **If 一局内 `UpgradeData` 或快照被改写**：本局 `capacity_total` 不变；下一局 `initialize` 才采用新值。
- **If 同一帧出现两次吃菜提交**（契约违例；餐局流程每次输入至多提交一个动作）：按到达顺序逐次校验，第二次若放不下则被拒，第一次的提交保留。本系统不做批量回滚。

## Dependencies

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 餐局流程 | 上游 | `initialize(upgrade_snapshot) → capacity_total \| 结构化错误`；`can_fit(e)`；`consume(e)`；读取 `capacity_total` / `capacity_used` / `capacity_remaining`；订阅 `run_ended` / `run_discarded` 作为局边界 | 硬 |
| 数据配置 | 上游 | `BalanceData.base_capacity`（`C0`，MVP 20）；不读 `DishData` | 硬（GDD 已写：`design/gdd/data-config.md`） |
| 局外升级 | 上游（经快照） | `upgrade_snapshot.capacity_bonus: int ≥ 0`；购买时保证 `C0 + 最大加成 ≤ 999` | 硬（GDD 已写：`design/gdd/meta-upgrades.md`） |
| 腻度与口味 | 间接上游 | `effective_satiety` 由餐局流程在吃下前取得并传入；本系统不直接调用、不重算 | 硬（值依赖，无调用边） |
| 局内 HUD | 下游 | 只读 `capacity_total` / `capacity_used` / `capacity_remaining` / `can_fit`；不得写、不得自建计数器 | 硬（GDD 已写：`design/gdd/in-game-hud.md`） |
| 结算与入场费 | 下游 | 从 `RunLog` 读 `capacity_total`；校验 `capacity_used = Σ effective_satiety` | 硬 |
| 菜单波次 | 设计期下游 | `wave_reachable_opt` 的 `C` 参数；运行期不读 | 软（设计期） |
| 菜品数值 | 设计期下游 | `opt_value` / `calibration_ratio` 的 `C0` 审计参数 | 软（设计期） |
| 纪录与里程碑 | 下游（软） | 可存 `capacity_total` 用于展示；不参与判定 | 软（GDD 已写：`design/gdd/records-and-milestones.md`） |
| 存档 | 无直接调用 | MVP 无中途存档；容量不持久化 | 条件 |

**方向说明**：索引里「胃容量 → 餐局流程」按**设计依赖**记录，而运行期是餐局流程调用本系统；这与 run-flow GDD 对其它子系统的处理一致——它们按生命周期所有权方向记录，避免形成环。本系统与腻度之间没有直接调用边：腻度给出 `effective_satiety`，餐局流程把它交给本系统判定，因此索引中不新增「胃容量 → 腻度」的边。

**双向一致性**：`餐局流程`、`腻度与口味`、`结算与入场费`、`菜品数值`、`菜单波次` 的 GDD 均已把本系统列入依赖或下游表；`数据配置` 目前没有「胃容量」下游行（见最终报告的共享文件清单）；`局外升级`、`局内 HUD`、`纪录与里程碑`、`存档` 的 GDD 尚未撰写。

## Visual/Audio Requirements

本系统的可见输出只有“还剩多少格”与“这一盘放不放得下”，视觉目标是**让剩余空间在点下去之前就可读**，不是让容量条好看。

- **容量用橄榄绿**：已用容量与安全区使用橄榄绿（概念文档的既有语义），回本红只属于正收益与关键数字；不使用大面积暖黄。
- **数字优先于条**：容量条旁必须同时显示整数读数（如 `12 / 20`），字号不小于菜品卡上的价格字号；条是辅助，数字是判据。
- **等宽数字**：容量读数使用等宽字体，数值变化时列宽不跳动。
- **零特效延迟**：容量变化在同一帧完成，不做缩放、弹跳、延迟入场；唯一允许的动效是数字原地替换。
- **不用颜色单独表达可吃性**：放不下时除了置灰，必须写出“需要 X 格，剩 Y 格”——颜色不是唯一通道。
- **音频不归本系统所有**：吃菜音、容量满提示音由「音效反馈」决定；本系统只保证数值在同一帧可读，不要求额外延迟。

> `art-director` not consulted — Lean mode（本系统是计数与判定层，视觉呈现归局内 HUD；概念文档的“数字优先 / 橄榄绿容量”锚点已覆盖配色原则）。进入生产前需人工复核。

## Game Feel

- **同一帧说清代价**：玩家确认一道菜后，容量读数与剩余格数必须在同一帧变化。心算被验证的快感依赖“我按下按钮，账本立刻同意我”。
- **点之前就能看见放不放得下**：菜品卡在选中态就要显示这道菜需要几格、还剩几格；等到点击后才拒绝，等于把算术题变成抽奖。
- **满格不是失败**：`20 / 20` 应当读作“这局吃满了”，而不是“你被惩罚了”。不闪红、不震动、不弹责备提示。玩家吃不下是因为算过账，不是因为机器不高兴。
- **压力来自剩余格数，不来自时间**（支柱 3）：没有倒计时、没有自动推进；容量条不得用动画催促决策。
- **失败可归因**：一局结束时，玩家应能指着某道菜说“这一口把最后 3 格吃掉了”。如果复盘时说不出容量花在哪，是展示的问题。

## UI Requirements

- **容量条**：显示 `已用 / 总数` 与剩余格数；已用与剩余都必须是整数读数，条与数字同屏。
- **菜品卡的可吃态**：由 `can_fit` 驱动；不可吃时置灰并写出“需要 X 格，剩 Y 格”。不得只靠颜色区分。
- **不做按容量排序**：不提供“按剩余格数可吃性”排序或自动隐藏放不下的菜——那会把最优解直接交给玩家（与菜品数值的排序规则一致）。
- **结算页**：显示本局总容量与实际已用；`wasted_capacity` 由结算系统计算与展示，本系统只提供 `capacity_total` 与逐行占用的记录。
- **重开**：容量条在下一局开始时回到 `0 / 新总数`，不保留上一局的动画或残留数字。
- **无障碍**：数字与背景对比度 ≥ 4.5:1；字号放大到 150% 时容量条与菜品卡不换行错位。完整要求见未来的 `design/accessibility-requirements.md`。

> **📌 UX Flag — 胃容量**：本系统有 UI 需求。Phase 4（Pre-Production）写 epic 之前，先用 `/ux-design` 为局内 HUD 的容量条与菜品卡不可吃态出 UX 规格；相关 story 引用 `design/ux/hud.md`，不直接引用本 GDD。

## Cross-References

| 文档 | 关系 | 本系统引用/依据 |
|---|---|---|
| `design/gdd/run-flow.md` | 上游 | `initialize(upgrade_snapshot) → capacity_total`、`can_fit(e)`、`capacity_used`；动作校验顺序；无“吃撑”结束原因；`RunLog.capacity_total` / `capacity_used` |
| `design/gdd/data-config.md` | 上游 | `UpgradeData.effect_type = STOMACH_CAPACITY`、`effect_value` 必须为整数；基础容量字段缺口见 Open Questions 1 |
| `design/gdd/cloying-and-taste.md` | 值来源 | `effective_satiety = satiety + extra_satiety`；`meal_step` 的 `remaining_capacity` 即本系统剩余容量；“已付的额外占用不退还” |
| `design/gdd/settlement-and-entry-fee.md` | 下游 | `capacity_total`、`wasted_capacity`、`CAPACITY_MISMATCH` / `CAPACITY_OVERFLOW` 不变式与错误码 |
| `design/gdd/dish-values.md` | 设计期下游 | 初始容量 `C0 = 20` 作为 `opt_value` 与 `calibration_ratio` 的审计参数；`opt_value = 104` 正好占满 20 格 |
| `design/gdd/menu-waves.md` | 设计期下游 | `wave_reachable_opt` / `wave_healthy` 的 `C` 参数 |
| `design/gdd/game-concept.md` | 来源 | 支柱 1/2/3；MVP「局外升级：容量 +1」；视觉锚点「橄榄绿表示胃容量与安全区」 |
| `design/gdd/systems-index.md` | 索引 | 优先级 MVP、层 Core、依赖清单与设计顺序 |

**待回指（不由本文件修改）**：`数据配置` 的下游表需要新增「胃容量」行（基础容量常数）；`局外升级`、`局内 HUD`、`纪录与里程碑` 的 GDD 写好后，需在各自的 Dependencies 与 Cross-References 中列出「胃容量」。

**待回改**：`run-flow.md` 的 Interactions 表把本系统接口缩写为 `initialize` / `capacity_used` / `can_fit`，漏了提交用的 `consume(e)`；建议在 merge 后补上，避免实现者把“判定”当成“扣减”。

## Acceptance Criteria

夹具放在 `tests/fixtures/capacity/`（快照与 `C0`）与 `tests/fixtures/menus/`（空菜单）。`run-flow` 的使用测试替身提供 `Active` / `Faulted` 状态探针与 `turns_used` 读数。每条用例使用独立夹具。

**初始化与上限**

1. **GIVEN** `C0 = 20`、`upgrade_snapshot.capacity_bonus = 0`，**WHEN** `initialize`，**THEN** 返回 `20`，`capacity_used = 0`，`capacity_remaining = 20`。
2. **GIVEN** `C0 = 20`、`capacity_bonus = 3`，**WHEN** `initialize`，**THEN** 返回 `23`，`capacity_used = 0`。
3. **GIVEN** 四个快照夹具（`capacity_bonus` 缺失 / `-1` / `1.5` / `null`），**WHEN** 分别 `initialize`，**THEN** 返回 `CAPACITY_INIT_INVALID`，`capacity_total` 与 `capacity_used` 未被写入，餐局流程进入 `Failed`；没有缺省为 0，也没有钳制。
4. **GIVEN** `C0 = 1`、`capacity_bonus = 998`，**WHEN** `initialize`，**THEN** 返回 `999`（合法边界）；**GIVEN** `capacity_bonus = 999`，**WHEN** `initialize`，**THEN** 返回 `CAPACITY_TOTAL_OUT_OF_RANGE`，餐局流程进入 `Failed`，不钳到 999。
5. **GIVEN** 本局已成功 `initialize` 且 `capacity_used = 17`，**WHEN** 在同一局内再次 `initialize`，**THEN** 返回 `CAPACITY_INIT_INVALID`，`capacity_used` 仍为 `17`，餐局流程进入 `Faulted`。
6. **GIVEN** 上一局以 `capacity_used = 17` 结束或被放弃，**WHEN** 下一局 `initialize`，**THEN** `capacity_used = 0`，`capacity_total` 按新快照重算，上一局任何字段都不延续。

**可吃性**

7. **GIVEN** `capacity_total = 20`、`capacity_used = 15`，**WHEN** `can_fit(5)` 与 `can_fit(6)`，**THEN** 分别为 `true` 与 `false`（等号允许，严格大于拒绝）。
8. **GIVEN** `capacity_used = 20`、`capacity_total = 20`，**WHEN** `can_fit(1)`，**THEN** 返回 `false`，且餐局流程仍为 `Active`、`turns_used` 与 `capacity_used` 不变、没有产生任何结束原因（不存在 `STUFFED` 枚举）。
9. **GIVEN** `capacity_total = 20`、`capacity_used = 0`，**WHEN** `can_fit(21)`，**THEN** 返回 `false`，且不记录 ERROR（整局放不下的菜是合法输入）。
10. **GIVEN** `capacity_total = 20`、`capacity_used = 0`，**WHEN** 分别调用 `can_fit(0)`、`can_fit(-1)`、`can_fit(2.5)`，**THEN** 三者都返回 `false` 且各记录一条 ERROR；没有免费菜通过。
11. **GIVEN** `capacity_used = 15`、`capacity_total = 20`，**WHEN** 连续两次 `can_fit(5)`，**THEN** 两次都返回 `true`，且 `capacity_used` 仍为 `15`（判定不预留）。

**提交与单调**

12. **GIVEN** `can_fit(3) = true` 且 `capacity_used = 0`，**WHEN** `consume(3)`，**THEN** `capacity_used = 3`、`capacity_remaining = 17`。
13. **GIVEN** 三次连续吃菜，`effective_satiety` 依次为 `3, 1, 4`，**WHEN** 逐次提交，**THEN** 每次后 `capacity_used` 为 `3, 4, 8`，全程不减少。
14. **GIVEN** `capacity_used = 18`、`capacity_total = 20`，**WHEN** `consume(3)`（越过上限），**THEN** 返回 `CAPACITY_OVERFLOW`，`capacity_used` 仍为 `18`，餐局流程进入 `Faulted`；没有钳到 `20`。
15. **GIVEN** `capacity_used = 10`，**WHEN** 分别 `consume(0)`、`consume(-1)`、`consume(2.5)`，**THEN** 各自返回 `INVALID_SATIETY`，`capacity_used` 仍为 `10`（负值绝不允许减少已用容量）。
16. **GIVEN** `capacity_used = 15`、`capacity_total = 20`，**WHEN** `consume(5)`，**THEN** `capacity_used = 20`（等号提交成功）。
17. **GIVEN** 本系统的公开接口清单，**WHEN** 枚举所有能改变 `capacity_used` 的方法，**THEN** 只有 `initialize` 与 `consume`，不存在 `refund` / `release` / `reset` / 公共 setter。

**集成、只读与确定性**

18. **GIVEN** 一局的所有成功吃菜已按顺序写入 `RunLog.eaten`，**WHEN** 对比 `capacity_used` 与 `Σ effective_satiety`，**THEN** 两者相等；结算不返回 `CAPACITY_MISMATCH`。
19. **GIVEN** 本局一道菜也没吃，**WHEN** 结算，**THEN** `capacity_used = 0`、`capacity_total ≥ 1`、`wasted_capacity = capacity_total`。
20. **GIVEN** `frozen = true`，**WHEN** 尝试吃菜或读取容量，**THEN** 容量逐字段不变，读取仍返回冻结前的值。
21. **GIVEN** 空的菜品目录（无 `DishData`）或与菜品目录不一致的夹具，**WHEN** `initialize` 并用注入的 `effective_satiety` 提交，**THEN** 初始化与提交都正常完成（证明运行期不读菜品数据）。
22. **GIVEN** 本局进行中改写 `UpgradeData` 或快照，**WHEN** 改动落地，**THEN** 本局 `capacity_total` 不变；下一局 `initialize` 才采用新的 `capacity_bonus`。
23. **GIVEN** 同一组输入（`C0`、`capacity_bonus`、吃菜序列）执行两次，**WHEN** 比较，**THEN** `capacity_total`、`capacity_used` 与每次 `can_fit` / `consume` 的返回码逐项相同。
24. **GIVEN** 本局在 `capacity_used = 12` 时被放弃，**WHEN** 下一局 `initialize`，**THEN** `capacity_used = 0`，且放弃的那一局不产生 `RunResult`、不影响下一局。
25. **GIVEN** 在 `initialize` 之前读取 `capacity_total` / `capacity_used` / `capacity_remaining`，**WHEN** 读取，**THEN** 返回 `0` 并各记一条 ERROR（正常流程不会发生）。

> 夹具 3、4、10、15 需要直接注入非法快照与非法 `e`；夹具 21 需要空菜单。不得为通过测试修改 12 道菜或 `UpgradeData` 的真实数值。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | **基础容量 `C0` 没有数据归属。** 数据配置只有 `UpgradeData` 与 `StationData`，没有基础容量字段；若在代码里写 `20`，违反「平衡数值必须数据驱动」。建议新增 `BalanceData`（或 `RunTuningData`）的 `base_capacity: int`（1–999），由数据配置加载并在启动时校验；本系统只消费。这样不改动 run-flow 已锁定的 `initialize(upgrade_snapshot)` 签名。 | 数据配置（编排会话） | 数据配置复审时 |
| 2 | **容量加成的购买上限。** 局外升级必须在购买时保证 `C0 + 最大可能加成 ≤ 999`，否则玩家会做出永久无法开局的存档；只靠开局检查不够。等级→数值的换算（含未来是否非线性）也由局外升级定义。 | 局外升级 | 局外升级 GDD 完成时 |
| 3 | **容量升级的收益断点。** 104 元的最优 10 道正好占满 20 格，而价格最高的 10 道需要 26 格——两者不是同一个集合，说明容量确实在承重。但固定入场费 85 下，`+1` 格是否带来更高分需要扫描确认：可能只在少数 `C` 处出现收益台阶，中间的 `+1` 只增加浪费容量。需要在 Vertical Slice 前做 `opt_value(C)` 扫描（C = 20–30），决定升级定价与入场费是否随容量重算。 | economy-designer（局外升级 + 结算 + 菜品数值） | Vertical Slice 前 |
| 4 | **`capacity_used` 的注册表归属。** `entities.yaml` 当前把 `capacity_used` 的 `source` 记为 `settlement-and-entry-fee.md`；运行期累加器在本系统。建议保留结算为公式来源，并在 `referenced_by` 加本 GDD，或把 `source` 改到本 GDD 并注明结算是同量校验。 | 编排会话（注册表） | 合并第 2 波时 |
| 5 | **“吃撑”是否只是展示概念。** `systems-index.md` 概述说容量“decides when the player is stuffed”，但 run-flow 明确没有 `STUFFED` 结束原因。当前解释为 HUD 的“没有可见菜放得下”提示，不是状态或结束原因。若试玩认为“有回合却吃不下”是空转，应回到 run-flow Open Question 9。 | game-designer（餐局流程 + 局内 HUD） | 垂直切片试玩后 |
| 6 | **是否需要容量释放 / 消化机制。** MVP 没有；加入会破坏规则 7 与结算的 `capacity_used = Σ effective_satiety` 不变式，并需要显式的容量事件，而不是一个调参开关。 | creative-director | Vertical Slice 规划时 |
| 7 | **是否需要 `capacity_changed` 信号。** 当前 HUD 与音效只从餐局流程的只读快照拉取；若音效需要“容量满”的即时事件，应由餐局流程的 `action_committed` 派生，而不是在本系统再开一份信号。 | 局内 HUD / 音效反馈 | 音效反馈 GDD 完成时 |
| 8 | **基础容量是否按档口区分。** 若未来档口有难度差异，`StationData.base_capacity` 会要求 `initialize` 接收 `station_id`，改动已锁定的签名。MVP 不做；由解锁与档口 GDD 决定。 | 解锁与档口 | 第二个档口设计时 |
