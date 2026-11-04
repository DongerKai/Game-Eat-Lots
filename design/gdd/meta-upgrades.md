# 局外升级（Meta Upgrades）

> **Status**: In Design
> **Author**: user + agents
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 数字看得见地涨

## Summary

局外升级是《回本》里唯一把「一局的收据」变成「下一局的起点」的系统。它拥有钱包的加减与正负策略、三项升级（胃容量、解腻、菜单视野）的价格与等级曲线、等级到本局修正值的解析，以及「购买先落盘再确认」的事务顺序。它不计算净利——净利由结算给出；它不写文件——落盘由存档提交；它不解释数值——单项效果由胃容量、腻度与菜单波次消费。它只回答一个问题：这局赚到的钱，能换成什么，以及换的时候会不会丢。

> **Quick reference** — Layer: `Feature` · Priority: `MVP` · Key deps: `结算与入场费`, `存档`, `数据配置`

## Overview

一局结束时，「结算与入场费」产出一份可以为负的 `net_profit`。本系统把它记进生涯钱包，并把钱包里的钱换成三项永久成长：多一格胃、少一点腻、早看一道菜。概念文档把这三项列为 MVP 的全部局外内容，所以本系统同时承担了 MVP 的**全部成长感**——如果升级买不起、买了没感觉、或者买了之后重启消失，游戏就没有局外循环。

它的设计立场由三条边界决定。第一，**钱包只有两个入口**：一局的净利进来，一次购买出去；没有利息、没有每日奖励、没有入场门槛。第二，**损失是真的**：负净利原样扣钱，钱包可以为负，因为「回本」如果不会失败，收据上的亏损就只是装饰。第三，**成长必须可读**：每一级的效果都以玩家能心算的整数表达（多 1 格、少 1 点、多看 1 道），价格与效果都摆在购买前，不在局内偷偷加成。

对下游，本系统是「升级快照」的生产者：开局前把等级解析成 `capacity_bonus`、`cloying_resistance`、`menu_vision` 三个已解析值，随 `RunConfig` 冻结整局。局内系统只消费这些值，不读等级、不算价格、不知道钱包的存在。

## Detailed Design

### Core Rules

1. **职责边界** — 本系统拥有：钱包的加减与正负策略、升级价格曲线与等级上限的**数值语义**、等级到本局修正值的解析、购买事务的顺序与回滚、升级目录的展示数据。它不拥有：净利与回本判定（结算与入场费）、持久化与事务原子性（存档）、`UpgradeData` 的 schema 与校验（数据配置）、容量的基础值与消耗（胃容量）、腻度计数的维护（腻度与口味）、波次划分与预览上限（菜单波次）、档口解锁费用（解锁与档口）、纪录排名（纪录与里程碑）、界面绘制（界面流程）。

2. **钱包只有三条流水** — `progression.wallet` 是 int64，只允许三种变化：一局结算的 `net_profit` 入账、一次升级购买扣款、一次档口解锁扣款（`spend(amount, reason, extra)`，由 #13 调用）。没有任何系统可以直接改写钱包；没有利息、签到、每日奖励、破产救济或入场门槛。任何新的钱包流水都必须先在本 GDD 增加一个事务接口，再实现；不得由下游自行扣款。

3. **负净利原样扣款，钱包可以为负** — 入账规则是 `wallet' = wallet + net_profit`，不取 0、不取绝对值、不做「只奖不罚」。**这是对结算 Open Question 1 的回答。** 理由：入场费 85 是本作唯一的价格压力；如果亏损被吞掉，回本与否就退化成两个数字的装饰，支柱 1（每口都是算术题）与收据上的亏损栏都会失去意义。代价是玩家可能负资产——这在机制上是合法的：没有入场门槛（结算规则 4），负钱包只意味着买不起升级，而不是不能继续玩。

4. **钱包不设上下限，只有 int64 边界** — 不设债务下限、不设财富上限。写入用 checked int64；`wallet + net_profit` 溢出时返回 `WALLET_OVERFLOW`，钱包保持原值，本局仍照常结算给纪录系统，界面显示「入账失败」。正常玩法在 `|wallet| ≤ 10⁶` 量级，溢出只用于把非法数据变成可见错误。

5. **一局只入账一次** — 结算保证同一 `run_id` 只产出一份 `RunResult`；本系统再以会话内的 `last_applied_run_id` 兜底：同一 `run_id` 的第二次 `apply_run_result` 被拒绝为 `ALREADY_APPLIED`，钱包不变。`run_id` 是会话级的（餐局流程），因此该去重不跨会话、不写档；见 Open Question 8。

6. **入账先于收据显示（编排者一次提交）** — 界面流程是结算提交编排者：收到 `RunResult` 后先调用本系统的 `prepare_run_result` 与纪录的 `prepare_run`，把 `progression` 与 `records` 合并为一次 `save.commit`，成功后再绘制收据。本系统只计算并保留 `progression` 新值，不自行 `commit`。写盘失败时**保留内存中的新值**、由存档标记 dirty、返回 `APPLIED_UNSAVED`，收据照常显示并带「未保存」标记，下一次 `flush`/成功提交时重试。理由：玩家已经打完这一局，因为磁盘故障没收收益比显示「未保存」更糟；这与存档对纪录类更新使用 `mark_dirty` 的策略一致。

7. **购买是事务：先落盘，再确认** — 购买顺序固定为：校验 → 内存应用 → `save.commit({progression: …})` → 成功后才返回成交；写盘失败则**回滚内存**到购买前的钱包与等级，返回 `COMMIT_FAILED`，界面显示「未保存，购买未生效」。这条直接落实存档 Core Rule 17，禁止出现「界面说买了、重启后没了」。校验顺序固定为 `DATA_NOT_READY` → `NOT_WRITABLE` → `NOT_IN_MENU` → `UNKNOWN_UPGRADE` → `BUSY` → `MAX_LEVEL` → `PREREQUISITE_LOCKED` → `NO_EFFECT` → `INSUFFICIENT_FUNDS`，保证同一状态下永远返回同一个错误。

8. **升级只买等级，不买数值** — 每项升级是一条 `UpgradeData`，每次购买使等级 +1，直到 `max_level`。所有效果值、价格基数与增长率来自记录，GDScript 中不得出现平衡数字字面量（0、1 等结构性常量除外）。本 GDD 只拥有「等级 × 效果」的换算与「等级 → 价格」的曲线。

9. **价格是等级的指数函数，效果是等级的线性函数** — 价格曲线由记录的 `cost_growth`（每级增长率，默认 1.35）与 `cost`（第 1 级价格）决定；效果是每级固定增量（数据配置的 `effect_value`）。效果线性、价格渐增，意味着每级收益相同而越往后越贵——这是本作唯一被允许的边际递减，且必须通过「买不买下一级」的选择体现，而不是通过隐藏的软上限。

10. **等级上限是硬上限** — `max_level` 由记录定义；达到上限后购买返回 `MAX_LEVEL`，界面显示「已满级」，不提供溢出等级或转换。降低某一记录的 `max_level` 不会退还已花掉的钱：快照按钳制后的等级解析（见规则 12），存档在下次载入时钳制并写 warning。

11. **前置升级是门槛，不是消耗** — `prerequisite_id` 非空时，购买前提是该前置升级的当前等级 ≥ 1；否则返回 `PREREQUISITE_LOCKED`。MVP 的三项升级都没有前置（空串）；判定逻辑仍然实现，供 Vertical Slice 的升级树使用。

12. **快照是已解析值，不是等级** — 本系统对外只提供 `resolve_upgrade_snapshot()`：把 `progression.upgrade_levels` 解析成 `capacity_bonus`（int ≥ 0）、`cloying_resistance`（float ≥ 0）、`menu_vision`（int ≥ 0），并附带一份冻结的 `levels` 副本供纪录展示。解析时等级先钳制到 `[0, max_level]`；记录不存在或 id 未知的条目贡献 0，但**不从存档中删除**（存档规则 5）。同类型多条记录的效果相加，结果与顺序无关。

13. **快照在开局冻结，升级只影响下一局** — 界面流程在组装 `RunConfig` 时调用一次 `resolve_upgrade_snapshot()` 并复制结果；局内系统只消费这份副本。局内不允许购买（`NOT_IN_MENU`），中途改等级不会改变当前局。这条与餐局流程 `RunConfig.upgrade_snapshot` 的开局冻结（RunConfig 表与 Edge Cases“中途获得升级”）及菜单波次规则 11 一致。

14. **`MENU_VISION` 买的是「下一波多看几道」** — 每级为本局的下一波预览额度 +1（`effect_value` 必须是正整数，由数据配置保证）；解析值 `menu_vision = Σ effect_value × level`。菜单波次按 `preview_dishes = min(menu_vision, wave_size − 1, |W_next|)` 消费它：**只揭示下一波，不揭示更远的波，也永远不揭示整波**。本系统拥有等级语义，菜单波次拥有上限语义；两者不得各自再定义一遍。升级页必须在购买前显示「下一波预览 X → Y 道（本档口上限 Z 道）」。

15. **`MENU_VISION` 不卖无效等级** — 对当前档口，`menu_vision_useful_max = ceil((wave_size − 1) / effect_value)`；当前等级已达该值后再买一级不会增加任何预览，购买被拒绝为 `NO_EFFECT`，界面显示「本档口已看满」。MVP 的 `wave_size = 4`、`effect_value = 1` → 上限 3 级；这正好是记录的 `max_level`，所以正常路径不会触发 `NO_EFFECT`；它是防止未来档口 `wave_size` 变小后继续卖废等级的护栏。**这是对菜单波次 Open Question 3 / 依赖行的数值回答。**

16. **`CLOYING_RESISTANCE` 是「每级减少 1 点追加饱腹」** — **这是对腻度与口味 Open Question 9 的回答。** 解析值 `R = floor(Σ effect_value × level)`；腻度系统按
    `extra_satiety(d) = max(0, min(group_cloy(group(d)), cloy_cap) − R)`
    计算。抗性是**读取时偏移**，不修改 `group_cloy` 计数：解腻（计数 −1）与抗性（读数 −R）互不干扰，吃菜顺序的一切既有结论保持不变。
    - 选「定点减 1」而不是「百分比缩放」的理由：本作的全部算术必须能心算（支柱 1）。`1 × 80% = 0.8 格`无法在桌上结算；定点减法只说「少 1 点」，并且与容量升级的「多 1 格」是同一把尺子。
    - 百分比方案在 MVP 里没有区别：组最大 2 道，原始追加最多 1 点，任何 `r > 0` 都会把它抹成 0，却引入小数与取整争议。定点方案在后续大组（原始 2–3 点）里线性、可预期。
    - 护栏：任一档口的设计必须满足 `R_max ≤ cloy_cap − 1`，保证满级玩家在同组吃满时仍至少付 1 点重复代价（支柱 2「没有白吃的菜」）。MVP `cloy_cap = 3`、`R_max = 1`，满足。
    - 抗性不能让 `extra_satiety` 变负；下限 0。抗性也不改变 `cloy_cap` 的含义，后者仍是原始追加的上限。

**20 元定价是保险位，不是收益位** —— 在「腻度与口味」的 canonical order（红烧肉 → 拍黄瓜 → 糖醋里脊 → 其余）下重复惩罚本来就是 0，抗性收益恒为 0；它只对尚未掌握顺序的玩家兑现，价值上限约 4 元/局（省下的 1 格换成更高值的菜）。因此这项升级不参与「每局稳定回本」的校准。`/prototype` 若测出新手在 3 局内普遍学会 canonical order，应把 L1 价格下调到 12–16，而不是加效果——加效果会让掌握顺序的玩家白拿收益，回到 D1 已否决的「解腻免费」。

17. **三项效果各自独立，不交叉相乘** — 容量升级只加 `capacity_bonus`，不解腻；解腻只减追加饱腹，不加容量；视野只买信息，不改价格、饱腹或腻度。任何「升级影响另一种升级收益」的乘区都推迟到 Vertical Slice 并需重写本 GDD。

18. **价格与结算的入场费无关** — 入场费是档口数据，运行期只读；本系统不在玩家升级后重算或上调入场费。**这是对结算 Open Question 6 的回答**：每座档口的入场费按**未升级**的可达上限校准（MVP `opt_value = 104`、`entry_fee = 85`，比值 1.224 落在 `[1.15, 1.35]`）；升级把玩家的上限抬到 120（容量 25）是设计内的成长，但必须落在**满级带 `[1.35, 1.45]`**（120 / 85 = 1.412），超出就改档口入场费或少卖等级，而不是运行时动态改价。任何 `effect_value`、`max_level` 或 `cost_growth` 的改动都必须在设计期重跑 `capacity_ceiling` 与 `wave_reachable_opt`，但不改运行时逻辑。

19. **存档不可写时，升级页只读** — `LoadReport.writable = false`（`Blocked`）或存档尚未完成载入时，购买一律返回 `NOT_WRITABLE`；界面禁用按钮并显示原因。局内入账仍可更新内存钱包（规则 6），但带「本次会话不保存」标记。理由：购买要求「先落盘再确认」，无法落盘就不能确认。

20. **数据未就绪时整体不可用** — 数据配置不在 `Ready`、或升级记录缺失时，`purchase` 返回 `DATA_NOT_READY`，升级页显示错误，不显示占位价格、不用默认值兜底。本系统不持有自己的价格表副本。

21. **删除生涯立即回到默认值** — 存档执行 `delete_career()` 后，本系统在下一次读取命名空间时得到 `wallet = 0`、空等级表；不保留内存缓存、不退还、不迁移。界面流程负责在删除后刷新钱包与升级页。

22. **纯函数、确定性** — `upgrade_price`、`upgrade_tree_cost`、`resolve_upgrade_snapshot` 只依赖记录与等级，不读时钟、不用随机数、不做 I/O。相同输入在任何平台、任何帧序下给出相同结果；价格计算使用 float64 后向上取整，只在整数边界上取整一次。

### States and Transitions

本系统只持有购买事务的短状态；生涯数据本身属于存档。

| 状态 | 触发 | 下一个状态 | 说明 |
|---|---|---|---|
| `Uninitialized` | 应用启动 | `Ready` / `ReadOnly` / `Unavailable` | 等数据配置 `Ready` 且存档载入完成；存档不可写则进 `ReadOnly` |
| `Ready` | 数据 `Ready` 且存档载入完成、可写 | `Purchasing` / `Ready` | 可入账、可购买 |
| `Ready` | 存档 `Blocked`（不可写） | `ReadOnly` | 可展示、可入账到内存，不可购买 |
| `Unavailable` | 数据未 `Ready` | `Ready` / `ReadOnly` | 不展示价格、不接受购买 |
| `Purchasing` | `purchase(id)` 通过校验，等待 `commit` | `Ready` | 同一时刻只允许一笔；第二笔返回 `BUSY` |
| `Purchasing` | `commit` 成功 | `Ready` | 内存与磁盘一致，返回成交 |
| `Purchasing` | `commit` 失败 | `Ready` | 回滚内存，返回 `COMMIT_FAILED` |
| `ReadOnly` | 新建生涯或存档恢复可写（新的 `SaveStatus`） | `Ready` | 方向由存档状态驱动，本系统不自行切换 |

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 数据配置 | 上游 | `get_upgrade(id)` → `cost`、`cost_growth`、`max_level`、`effect_type`、`effect_value`、`prerequisite_id`；`get_station(id)` → `wave_size`；前提 `Ready` | 硬 |
| 存档 | 上游 | 读取 `progression.wallet`、`progression.upgrade_levels`；`commit({progression: …})`；`mark_dirty` / `flush`；`SaveStatus` | 硬 |
| 结算与入场费 | 上游 | `RunResult.run_id`、`net_profit`、`upgrade_snapshot`（原样转交） | 硬 |
| 界面流程 | 下游 | 升级页读取目录/价格/可购买性并调用 `purchase`；收据调用 `apply_run_result`；钱包与「未保存」标记 | 硬（UI） |
| 餐局流程 | 下游（间接） | 经界面流程把 `resolve_upgrade_snapshot()` 放入 `RunConfig.upgrade_snapshot`；本系统不直接调用餐局流程 | 软（间接） |
| 胃容量 | 下游 | 消费 `capacity_bonus`，加到基础容量上 | 硬 |
| 菜单波次 | 下游 | 消费 `menu_vision` 作为总预览额度；上限由它执行 | 软（强） |
| 腻度与口味 | 下游 | 消费 `cloying_resistance`；按规则 16 的表达式生效 | 硬 |
| 解锁与档口 | 下游 | 读取钱包与等级作为解锁前置；`unlock_cost` 的扣款经 `spend(amount, reason, extra)`（GDD 已写：`design/gdd/unlocks-and-stations.md`） | 软 |
| 纪录与里程碑 | 下游 | 从 `RunResult.upgrade_snapshot.levels` 冻结历史等级；排名规则不归本系统 | 软 |
| 首局引导 | 下游 | 读取价格与可负担性，可推荐第一笔升级；不得自动购买 | 软 |
| 音效反馈 | 下游（可选） | 本系统发出 `purchase_succeeded` / `purchase_failed` / `run_result_applied`；音效反馈 MVP 不消费（其 GDD 明确这些事件不产生声音） | 软 |
| 场景与流程管理 | 时序 | 只影响顶层状态：购买只在 `Menu` 生效；本系统不驱动顶层切换 | 软（编排） |

**接口所有权**：本系统拥有钱包语义、价格曲线、等级语义与快照解析；存档拥有字段的持久化与事务；数据配置拥有记录 schema；下游拥有各自对修正值的消费规则。

**对外接口（设计级）**：

| 接口 | 输入 | 输出 | 说明 |
|---|---|---|---|
| `get_wallet()` | — | int64 | 只读；负值合法 |
| `get_level(id)` | 升级 id | int ≥ 0 | 未知 id 返回 0 |
| `next_price(id)` | 升级 id | int \| null | 已达上限或未知 id 返回 `null` |
| `can_purchase(id)` | 升级 id | `{ok, reason}` | 只校验，不改状态；顺序见 Edge Cases |
| `purchase(id)` | 升级 id | `PurchaseResult` | 事务；成功后磁盘已更新 |
| `spend(amount, reason, extra)` | 金额 int64、原因枚举、额外上下文 | `SpendResult` | 通用扣款事务，用于档口解锁。与购买一样先落盘再确认；`amount < 0` 立即拒绝 |
| `prepare_run_result(result)` | `RunResult` | `{progression_update, status}` | 每局一次；只计算内存新值，不写盘；由结算提交编排者合并后一次 `commit` |
| `resolve_upgrade_snapshot()` | — | `Dictionary` | 含三个已解析值 + `levels` 副本；每次返回新副本 |
| `get_catalog()` | — | 升级视图数组 | 供升级页渲染：当前等级、当前效果、下一效果、下一价格 |
| `progression_changed` | — | 信号 | 携带 `wallet` 与 `upgrade_levels`；钱包或等级实际变化后发一次（含 `APPLIED_UNSAVED`）；被拒绝的购买与重复入账不发 |

## Formulas

本系统定义五组运行时规则（入账、价格、树总价、快照解析、抗性与预览上限）与一组设计期校准量。运行时规则必须可判定；设计期量只用于定价与验收带。

### 钱包入账 `wallet_after_run`

The `wallet_after_run` formula is defined as:

`wallet' = checked_add_int64( wallet, net_profit(R) )`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `wallet` | int64 | −9.22×10¹⁸ – 9.22×10¹⁸ | `progression.wallet` | 本局之前的生涯余额；可以为负 |
| `net_profit(R)` | int64 | −entry_fee – (Σprice − entry_fee) | 结算 `RunResult.net_profit` | 有符号净利，结算不钳制 |
| `wallet'` | int64 | 同上 | calculated | 本局之后的余额；不设玩法上下限 |

**Output Range:** int64 全区间；**不做玩法钳制**。唯一失败模式是 checked 加法溢出 → `WALLET_OVERFLOW`，原值不变、不落盘。

**Example:** 新档 `0 + 19 = 19`；下一局亏损 `19 + (−31) = −12`；再一局打平 `−12 + 0 = −12`；此时买 12 元的升级：`−12 < 12` → `INSUFFICIENT_FUNDS`。

### 升级价格 `upgrade_price`

The `upgrade_price` formula is defined as:

`price(u, L) = ceil( u.cost × u.cost_growth^(L−1) )`，`L ∈ [1, u.max_level]`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `u` | UpgradeData | — | `get_upgrade(id)` | 一条升级记录 |
| `u.cost` | int | 0 – INT32_MAX | `UpgradeData.cost` | **第 1 级的价格**（本 GDD 定义该字段语义） |
| `L` | int | 1 – `max_level` | 本次购买将要达到的等级 | 购买前等级为 `L−1` |
| `u.cost_growth` | float64 | 1.2 – 1.6（设计带） | `UpgradeData.cost_growth`（新增字段，默认 1.35） | 每级价格增长率 |
| `price(u, L)` | int | 0 – INT32_MAX | calculated | 从 `L−1` 升到 `L` 的价格；`price(u, 1) = u.cost` |

**Output Range:** `[0, INT32_MAX]`；`cost = 0` 合法（免费升级），但会在设计期审查中记为 warning（见 Edge Cases）。**取整只在最后做一次 `ceil`**，避免逐级取整造成的漂移；同一 `(u, L)` 恒得同一值。

**Example:** `u.cost = 12`、`u.cost_growth = 1.35`：L1 `12`、L2 `ceil(16.2)=17`、L3 `ceil(21.87)=22`、L4 `ceil(29.5245)=30`、L5 `ceil(39.858)=40`。

### 升级树总价 `upgrade_tree_cost`

The `upgrade_tree_cost` formula is defined as:

`tree_cost(u) = Σ_{L=1}^{max_level(u)} price(u, L)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `tree_cost(u)` | int | 0 – INT32_MAX | calculated | 把一条升级买到满级的总花费 |

**Output Range:** 单调不减于 `price(u, max_level)`。用于设计期核对「一个会话能买几级」，不进入运行时。

**Example:** MVP 三项：容量 `12+17+22+30+40 = 121`；解腻 `20`；视野 `16+22+30 = 68`。全树 `209`。一个典型会话（3–6 局、中位净利 +3–+11）大约能买 1–4 级，符合概念文档「每局结束看账、消费利润升级，然后在升级完成或资金不足时自然停下」。

### 快照解析 `resolve_upgrade_snapshot`

The `resolve_upgrade_snapshot` formula is defined as:

```
capacity_bonus      = Σ_{u : u.effect_type = STOMACH_CAPACITY}   u.effect_value × lvl(u)
cloying_resistance  = Σ_{u : u.effect_type = CLOYING_RESISTANCE} u.effect_value × lvl(u)
menu_vision         = Σ_{u : u.effect_type = MENU_VISION}        u.effect_value × lvl(u)
lvl(u)              = clamp(levels[u] ?? 0, 0, u.max_level)
```

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `levels` | map<String,int> | 等级 ≥ 0 | `progression.upgrade_levels` | 存档中的等级表；未知 id 保留但不参与 |
| `lvl(u)` | int | 0 – `max_level` | calculated | 防御性钳制后的等级 |
| `capacity_bonus` | int | 0 – 99 | calculated | 本局容量加成，交给胃容量 |
| `cloying_resistance` | float64 | 0 – 99 | calculated | 本局抗性点数的精确值，交给腻度 |
| `menu_vision` | int | 0 – 99 | calculated | 本局总预览额度，交给菜单波次 |

**Output Range:** 三个值均 ≥ 0；`STOMACH_CAPACITY` / `MENU_VISION` 的 `effect_value` 由数据配置保证为正整数，因此求和结果为整数；`CLOYING_RESISTANCE` 可为小数。**快照另含 `levels` 的冻结副本**（仅供纪录展示）；下游不得用 `levels` 自行重算效果，重算会复制一套等级→数值规则。

**Example:** 等级表 `{upgrade_stomach_capacity: 2, upgrade_menu_vision: 1, upgrade_cloying_resistance: 1}` → `capacity_bonus = 2`、`menu_vision = 1`、`cloying_resistance = 1.0`，`levels` 为同一份表的副本。存档中若含 `upgrade_unknown: 3`，该条贡献 0 但保留在 `levels` 与存档中。

### 抗性生效后的追加饱腹（并入既有 `extra_satiety`，不单独登记）

The effective `extra_satiety` formula is defined as（腻度与口味按此式替换其现有公式）：

`extra_satiety(d) = max(0, min(group_cloy(group(d)), cloy_cap) − floor(cloying_resistance))`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `group_cloy(g)` | int | 0 – N | 腻度局内状态 | 本局已吃的同组菜数（不含当前） |
| `cloy_cap` | int | ≥ 1 | `StationData.cloy_cap` | 原始追加的封顶，MVP 为 3 |
| `cloying_resistance` | float64 | 0 – `cloy_cap − 1`（设计带） | 本系统快照 | 本局抗性；取 `floor` 后作为整数点数 |
| `R` | int | 0 – `cloy_cap − 1` | `floor(cloying_resistance)` | 生效的定点减免 |
| `extra_satiety` | int | 0 – `cloy_cap` | calculated | 抗性之后仍要付的追加格数 |

**Output Range:** `[0, cloy_cap]`，永不为负。**不修改计数**：`group_cloy` 的维护与解腻的 `−1` 完全不变；抗性只在读取时扣减。`floor` 只作用于总抗性一次，不对每级分别取整。

**Example:** 原始追加 1、`R = 1` → `max(0, 1−1) = 0`（MVP 中重复的肉不再加格）；原始追加 3、`R = 1` → `2`；原始追加 3、`R = 5` → `0`；`R = 1.5` → `floor = 1`，与 `R = 1` 相同。

### 预览上限 `menu_vision_useful_max`

The `menu_vision_useful_max` formula is defined as:

`useful_max(u, s) = ceil( (wave_size(s) − 1) / u.effect_value )`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `wave_size(s)` | int | 1–8 | `StationData.wave_size` | 当前档口每波揭示数，MVP 为 4 |
| `u.effect_value` | float | 1–99 | `UpgradeData.effect_value` | 每级预览增量；该类型必须取正整数值 |
| `useful_max` | int | 0 – 7 | calculated | 该档口下仍有意义的最高等级；`wave_size = 1` 时为 0 |

**Output Range:** `≥ 0`。`level + 1 > useful_max` 时购买被拒绝为 `NO_EFFECT`。**本系统只负责不卖废等级**；真正的预览值仍由菜单波次按 `min(menu_vision, wave_size−1, |W_next|)` 截断。

**Example:** MVP `wave_size = 4`、`effect_value = 1` → `useful_max = 3`，与记录 `max_level = 3` 一致。若 `effect_value = 2`，`useful_max = ceil(3/2) = 2`：第 1 级预览 2 道、第 2 级预览 3 道（被截断），第 3 级不再增加，故拒绝。若某档口 `wave_size = 2`，`useful_max = 1`，第 2 级在该档口不可购买。

### 设计期：容量上限与升级价值

以下量只用于定价与验收，不进入运行时。基准来自菜品数值的 12 道菜、初始容量 20、10 回合。

`capacity_ceiling(C) = max { Σ price(d) : |S| ≤ T, Σ satiety(d) ≤ C }`（0/1 背包；与 `opt_value` 同一定义，只是容量可变）

| 容量 C | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27+ |
|---|---|---|---|---|---|---|---|---|
| 可达上限（元） | 104 | 106 | 107 | 112 | 116 | 120 | 121 | 121 |
| 增量（元/局） | — | +2 | +1 | +5 | +4 | +4 | +1 | 0 |

**读法**：在 10 回合的硬约束下，第 11 道菜永远吃不到，所以容量升级的收益不是「多吃一道」，而是「把低值小份换成高值大份」。26 格是 MVP 菜池的有效上限（27 格起上限不再上涨）；因此 `base_capacity + max_level ≤ 26` 是数据约束，MVP 卖到 25 格留一格余量。**在售区间 20→25 的 +16 元 / 5 级，平均边际约 3.2 元/局**；25→26 的 +1 元不在售，只用于说明有效上限。

**升级后的校准带（回答结算 Open Question 6 的数值部分）**：`calibration_ratio ∈ [1.15, 1.35]` 只约束**未升级**上限（104 / 85 = 1.224）。容量满级 25 格 → 120，`120 / 85 = 1.412`，必须落在**满级带 `[1.35, 1.45]`**。每座档口同时满足两带才算校准通过：满级比值 > 1.45 → 提高该档口 `entry_fee`（改档口数据）或少卖容量等级；< 1.35 → 升级树太弱，先加效果再考虑降价。这样成长幅度有上限，且不需要运行时改价。

`value(L)` 是设计期估计的每级每局收益，`payback_runs(u, L) = price(u, L) / value(L)`：

| 升级 | `value(L)` 估计 | L1/L2/L3/L4/L5 价格 | 回收局数（L1） |
|---|---|---|---|
| 胃容量 | 2–5 元/局，均值 ~3.2（20→25） | 12 / 17 / 22 / 30 / 40 | 2–6 |
| 菜单视野 | 3–4 元/局（一次跳过的代价约 3–4 元） | 16 / 22 / 30 | 4–5 |
| 解腻 | 0–4 元/局（仅对未掌握顺序的玩家兑现；掌握 canonical order 后恒为 0） | 20 | 5–∞ |

**目标带**：首级回收 3–8 局；后期等级允许回收 10–20 局（生涯目标是跨会话的）。若 `/prototype` 测得的实际收益低于估计的一半，先降 `cost`，不要先改 `cost_growth`——后者影响的是整棵树的价格形状。

### MVP 升级目录（将写入 `UpgradeData`）

| id | 名称 | `effect_type` | `cost`（L1） | `cost_growth` | `max_level` | `effect_value` | 满级效果 | 设计作用 |
|---|---|---|---|---|---|---|---|---|
| `upgrade_stomach_capacity` | 加宽胃袋 | `STOMACH_CAPACITY` | 12 | 1.35 | 5 | 1.0 | 容量 +5（20 → 25） | 把低值小份换成高值大份；上限 104 → 120 |
| `upgrade_cloying_resistance` | 久吃不腻 | `CLOYING_RESISTANCE` | 20 | 1.35 | 1 | 1.0 | 抗性 1 点/次 | 保险位：顺序错误时补 1 格（canonical order 下收益为 0） |
| `upgrade_menu_vision` | 先看一步 | `MENU_VISION` | 16 | 1.35 | 3 | 1.0 | 下一波预览 3 道 | 少跳过、早规划；上限 `wave_size − 1 = 3` |

- 三项记录都不设前置（`prerequisite_id = ""`）；购买顺序完全由价格与钱包决定。
- 目录本身由数据配置提供；本表是数值决策，不是第二份 schema。`cost` 的「第 1 级价格」语义需要在 `UpgradeData` 字段说明中补一句（见 Cross-References 的待回改）。
- 解腻只卖 1 级，因为 MVP 的组最大 2 道、原始追加最多 1 点；第 2 级在 MVP 中恒为 0 收益。等大组出现（`cloy_cap` 真正生效）时再扩级，并遵守规则 16 的 `R_max ≤ cloy_cap − 1`。

## Edge Cases

**钱包与入账**

- **If 钱包为负**：合法。购买按 `wallet ≥ price` 判定，负钱包一律买不起；不提供破产救济、不自动归零、不阻止继续游戏（入场无门槛）。
- **If 净利恰好把钱包归零**：合法，余额显示 `0`，不触发任何特殊状态。
- **If `entry_fee = 0`**：`net_profit = total_value`，钱包只增不减；入账逻辑不变。
- **If 同一 `run_id` 第二次入账**：返回 `ALREADY_APPLIED`，钱包不变；不写档、不发信号。
- **If 入账时 `commit` 失败**：内存保留新值，`progression` 标记 dirty，返回 `APPLIED_UNSAVED`；收据显示「+N（未保存）」，钱包余额按内存显示；下次成功提交时一并写入。
- **If 入账加法溢出 int64**：返回 `WALLET_OVERFLOW`，钱包保持原值，不写档；收据显示「入账失败」，本局仍交由纪录系统处理。
- **If 存档为 `Blocked` / 不可写**：入账仍更新内存钱包与「本次会话不保存」标记；购买全部返回 `NOT_WRITABLE`。

**购买**

- **If 钱包恰好等于价格**：允许，购买后钱包为 0。
- **If 钱包不足**：返回 `INSUFFICIENT_FUNDS`，钱包与等级都不变，不发信号。
- **If 已达 `max_level`**：返回 `MAX_LEVEL`，`next_price` 为 `null`，界面显示「已满级」。
- **If 升级 id 不存在于数据**：返回 `UNKNOWN_UPGRADE`；若该 id 只存在于存档，载入时保留、解析贡献 0、升级页隐藏、不报错。
- **If 前置未满足**（`prerequisite_id` 非空且前置等级为 0）：返回 `PREREQUISITE_LOCKED`。
- **If 数据配置未 `Ready`**：返回 `DATA_NOT_READY`；升级页不显示价格，不用默认值兜底。
- **If 当前顶层状态不是 `Menu`**（局内、收据未完成、场景切换中）：返回 `NOT_IN_MENU`；局内不存在升级入口。
- **If 上一笔购买仍在等待 `commit`**：返回 `BUSY`；不排队、不合并，界面禁用按钮直到回调。
- **If `commit` 失败**：回滚钱包与等级到购买前，返回 `COMMIT_FAILED`；界面显示「未保存，购买未生效」，允许玩家重试。
- **If 当前视野等级已达 `menu_vision_useful_max`**：再买一级返回 `NO_EFFECT`，界面显示「本档口已看满」；等级与钱包不变。
- **If `cost = 0`**：合法，价格恒为 0；设计期审查记 warning（免费升级会让「回本」失去购买压力），运行时照常成交。
- **If `effect_value` 为 0 或负**：数据配置校验应已拒绝；若运行期仍遇到，本系统把该记录视为无效并返回 `DATA_NOT_READY`，不静默按 0 处理。
- **If 解腻 `effect_value` 为小数且总抗性不足 1**：解析值保留小数，腻度侧 `floor` 后为 0；界面必须显示「当前 0 点」并标注「下一级后 0 点」，避免卖一个看不见的效果。
- **If 解析后的抗性超过 `cloy_cap − 1`**：效果按公式自然落到 0，不报错、不钳制解析值（本系统不知道当前档口的 `cloy_cap`）；设计期审查必须把它标为数据错误，因为满级玩家在该档口将完全免疫重复惩罚。

**等级与快照**

- **If 存档等级高于 `max_level`**：存档在载入时钳制并写 warning；本系统解析时再次钳制，不退钱、不报错。
- **If 设计师在会话中下调 `max_level`**：下一次解析即按新上限钳制；已花掉的钱不退；当前局已冻结的快照不变。
- **If 同一类型有多条升级记录**：效果相加，顺序无关；界面分别显示每条，总效果在快照里合并。
- **If 某记录的 `effect_value` 在版本间变化**：等级保留，下一次开局按新值解析；不迁移、不补偿。历史纪录里的旧快照保持不变。
- **If 快照被下游修改**：下游收到的是副本；本系统每次调用返回新字典，局内修改不影响生涯状态。
- **If 记录被删除但存档仍有等级**：该条贡献 0，仍保留在 `levels` 与存档中；升级页不显示；不产生错误。
- **If `menu_vision` 解析值超过档口上限**：菜单波次自行截断；本系统不截断解析值（快照反映真实等级），但购买入口按规则 15 阻止继续买。

**生涯与生命周期**

- **If 删除生涯**：钱包 0、等级表空，升级页刷新为初始状态；不保留任何内存缓存。
- **If 首局使用 6 道菜的引导菜池**：本系统**不特判**，照常按该局档口的 `entry_fee` 入账。若引导局必然亏损，应由首局引导选择免单/低费档口，而不是由钱包开特例（见 Open Question 1）。
- **If 玩家在收据页停留很久后退出**：收益已在收据显示前落盘；退出不重复入账、不丢收益。
- **If 购买成功后立刻崩溃**：磁盘已更新；重启后等级与钱包一致（这是「先落盘再确认」的全部意义）。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 结算与入场费 | 硬 | `RunResult.net_profit`（有符号 int64）、`run_id`、`upgrade_snapshot` | GDD 已写（`design/gdd/settlement-and-entry-fee.md`） |
| 存档 | 硬 | `progression.wallet`、`progression.upgrade_levels`；`commit` / `mark_dirty` / `flush`；`SaveStatus` | GDD 已写（`design/gdd/save-system.md`） |
| 数据配置 | 硬 | `get_upgrade(id)` → `cost`、`cost_growth`、`max_level`、`effect_type`、`effect_value`、`prerequisite_id`；`get_station(id)` → `wave_size`；前提 `Ready` | GDD 已写（`design/gdd/data-config.md`） |

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 餐局流程 | 软（间接） | 经界面流程消费快照；run-flow 明确把它列为间接关系，不算直接依赖 | GDD 已写（`design/gdd/run-flow.md`） |
| 胃容量 | 硬 | `capacity_bonus` 加到基础容量 | GDD 已写（`design/gdd/stomach-capacity.md`） |
| 菜单波次 | 软（强） | `menu_vision` 作为总预览额度；上限由它执行 | GDD 已写（`design/gdd/menu-waves.md`） |
| 腻度与口味 | 硬 | `cloying_resistance`；需按本 GDD 规则 16 的表达式接入 | GDD 已写（`design/gdd/cloying-and-taste.md`，公式尚未含抗性项） |
| 界面流程 | 硬（UI） | 升级页、收据入账、钱包与「未保存」状态 | GDD 已写（`design/gdd/ui-flow.md`） |
| 解锁与档口 | 软 | 读取钱包与等级作为解锁前置；`unlock_cost` 扣款经 `spend(amount, reason, extra)` | GDD 已写（`design/gdd/unlocks-and-stations.md`） |
| 纪录与里程碑 | 软 | 从 `upgrade_snapshot.levels` 冻结历史等级 | GDD 已写（`design/gdd/records-and-milestones.md`） |
| 首局引导 | 软 | 价格与可负担性；可推荐、不可自动购买 | GDD 已写（`design/gdd/first-run-onboarding.md`） |
| 音效反馈 | 软（可选） | 本系统发出 `purchase_succeeded` / `purchase_failed` / `run_result_applied`；MVP 不消费 | GDD 已写（`design/gdd/audio-feedback.md`） |
| 场景与流程管理 | 软（编排） | 购买只在 `Menu`；本系统不驱动顶层状态 | GDD 已写（`design/gdd/scene-flow.md`） |

**双向一致要求**：以上每个上游必须在其 Dependencies 中列出本系统，并在 Cross-References 中回指；下游必须写明「只消费快照，不重算等级」。存档已在下游表列出本系统；结算已列出；数据配置已列出。腻度与口味、菜单波次需要补上「局外升级」行并把 Open Question 标记为已回答（见下）。

**对「腻度与口味」的契约**：抗性按 `extra_satiety = max(0, min(group_cloy, cloy_cap) − floor(cloying_resistance))` 生效，作为读取时偏移；不得修改 `group_cloy`、不得改变 `cloy_cap` 语义、不得新增百分比路径。抗性由「餐局流程」在开局时随 `upgrade_snapshot` 传入，腻度不直接依赖本系统——这条依赖方向沿用腻度 GDD 已写明的理由，避免与 `局外升级 → 结算 → 腻度` 成环。

**对「菜单波次」的契约**：`menu_vision` 是总预览额度（整数）；每级 +1；只作用于下一波；上限 `min(menu_vision, wave_size−1, |W_next|)` 由菜单波次执行。本系统只保证不卖超过 `wave_size−1` 的等级。若未来 `wave_size` 逐档口不同，升级页需要按档口显示上限（见 Open Question 4）。

**对「结算与入场费」的契约**：净利全额入账，允许为负；不按 0 计入；入场费运行期不变，校准只按未升级上限设计。结算 Open Question 1 与 6 均在此回答。

**待回改（不在本文件执行，交由编排会话统一处理）**：

1. `systems-index.md` — #12 行改为 `Designed` / `design/gdd/meta-upgrades.md`；进度计数 `Design docs started` 与 `MVP systems designed` 各 +1；补充升级页的 UX 备注。
2. `entities.yaml` — 新增 `upgrade_price`、`upgrade_tree_cost`、`wallet_after_run`、`menu_vision_useful_max` 四条公式；**不新增** `upgrade_cost_growth` 常量（增长率是 `UpgradeData.cost_growth` 记录字段）；`net_profit`、`extra_satiety`、`opt_value`、`calibration_ratio` 的 `referenced_by` 追加本文件；`extra_satiety` 的表达式/notes 就地改为含抗性的版本（不新增 `resisted_extra` 条目，避免同一事实两份定义）；`net_profit` 的 notes 改为「入账规则由局外升级定义」；`calibration_ratio` 的 notes 补「满级容量 25 时 120/85 = 1.412，必须落在满级带 [1.35, 1.45]」。
3. `data-config.md` — 下游表「局外升级」状态改为 `GDD 已写（design/gdd/meta-upgrades.md）`；`UpgradeData` 新增 `cost_growth: float`（默认 1.35，安全范围 1.2–1.6，校验必须 ≥ 1.0）；`UpgradeData.cost` 字段说明补「本字段是第 1 级价格，价格曲线由局外升级拥有」；`effect_value` 说明补「`CLOYING_RESISTANCE` 的语义为每级减免 1 点追加饱腹，小数部分在解析时取整」；补 `StationData.wave_size`（菜单波次声明的新增字段，默认 4），因为本系统的 `menu_vision_useful_max` 读它。
4. `settlement-and-entry-fee.md` — 下游表「局外升级」状态改为 `GDD 已写`；Open Question 1 标记已回答（全额入账、允许负钱包）；Open Question 6 标记已回答（入场费运行期不变、按未升级上限校准）。
5. `cloying-and-taste.md` — 上游/下游表补「局外升级」行（经餐局流程传入抗性，读时偏移）；`extra_satiety` 公式与「未接入：抗性」注释改为已接入；Open Question 9 标记已回答（定点减 1，非百分比）。
6. `menu-waves.md` — 依赖表「局外升级」状态改为 `GDD 已写`；补一句 `menu_vision` 的等级语义由本 GDD 拥有、上限由本系统执行；Open Question 3 标记为「上限由局外升级保证不超卖」。
7. `save-system.md` — 下游表「局外升级」状态改为 `GDD 已写`；无需改 schema；补一句「`progression` 在入账写盘失败时可进入 dirty 并在下次 `flush` 重试」（购买仍必须提交成功才确认，两者策略不同）。
8. `run-flow.md` — **不改依赖方向**：run-flow 已明确把「局外升级」列为间接关系（本系统不调用它，快照经界面流程注入）；可选的补充是在该段加一句「快照由局外升级生产，含解析值与 `levels` 副本」，并据此关闭 Open Question 11。
9. `dish-values.md` — 在 `calibration_ratio` 的说明中补一句：`[1.15, 1.35]` 是**未升级**校准带；满级容量 25 的上限 120 / 85 = 1.412 属于本 GDD 定义的**满级带 `[1.35, 1.45]`**，两者同时成立才算该档口校准通过。

## Tuning Knobs

| 旋钮 | 位置 | 当前值 | 安全范围 | 影响 |
|---|---|---|---|---|
| `upgrade_*.cost_growth` | `UpgradeData.cost_growth` | 1.35 | 1.2 – 1.6 | 价格曲线的陡度；只影响同一记录内后续等级，不影响第 1 级 |
| `upgrade_*.cost` | `UpgradeData.cost` | 12 / 20 / 16 | 0 – 200 | 第 1 级价格，直接决定首笔升级的到达时间 |
| `upgrade_*.max_level` | `UpgradeData.max_level` | 5 / 1 / 3 | 1 – 6 | 成长深度；容量受 `base + max_level ≤ 26` 约束，视野受 `≤ wave_size − 1` 约束 |
| `upgrade_*.effect_value` | `UpgradeData.effect_value` | 1.0 全部 | 容量/视野：正整数 1–3；解腻：0.5 – 2 | 每级效果；解腻满级必须满足 `R_max ≤ cloy_cap − 1` |
| 钱包下限 | 本系统规则 3 | 无 | 无（若未来引入，必须先改结算与 HUD） | 设为 0 会把亏损变成装饰，不建议 |
| 首级回收局数目标 | 设计期 | 3 – 8 局 | 2 – 12 局 | 定价验收带；超出时先调 `cost`，再调 `cost_growth` |
| 容量有效上限 | 设计期 | 26 格 | 由 `capacity_ceiling` 重算 | 超过后容量升级不再提升上限，属于废等级 |

**不允许的旋钮**：不提供「升级折扣」「限时优惠」「退款」「等级重置」。这些会引入本 GDD 未定义的第二个价格来源，并让「先落盘再确认」的事务边界变复杂。

## Visual/Audio Requirements

- **数字优先**：升级卡的「当前效果 → 下一效果」与价格必须用与收据同一套等宽数字；图标不承担信息。
- **不隐藏价格**：买不起时价格照常显示，用「差 X 元」而不是把按钮变灰了事；玩家要能算还差几局。
- **不用进度条替代数字**：等级用 `2 / 5` 这类可读文本，不用无法读出确切值的圆环。
- **钱包符号**：正余额用回本红加粗；负余额用正常墨色加负号，不用红色——红只属于正收益。
- **动效克制**：购买成功后只允许原地替换数字（价格、等级、钱包），不做金币飞行、缩放或彩带。
- **音频不归本系统**：成交与失败是否各有音效由音效反馈决定；本系统只保证事件在提交结果之后发出。

> `art-director` not consulted — Lean mode（本系统是数值与事务层；概念文档的「价目板 / 数字优先」锚点已覆盖表现原则）。进入生产前需人工复核。

## Game Feel

- **付款即见数**：按下购买后，钱包与等级在同一帧内变化（在 `commit` 成功的回调帧），不播「处理中」动画；慢的是磁盘，不是玩家。
- **失败要说清**：写盘失败必须显示「未保存，购买未生效」，并让钱包与等级回到购买前——不能只弹一个「错误」。
- **收据是停点**：一局结束后先看收据，再进升级页；`+19` 与 `余额 19` 同时出现，让「这局的钱能买什么」成为一个自然问题。
- **升级必须能被下一局感知**：容量 +1 应在下一局的开局容量条上直接显示 `21`；视野 +1 应在下一波多出一张可读的菜卡；解腻 1 点应在重复菜的卡片上少一个 `+1`。任何一项在局内不可见，就说明快照没有接上。
- **不制造错失感**：没有限时折扣、没有「再买一级更划算」的倒计时；买不起就是买不起，下次再来。

## UI Requirements

- **升级页字段（每条）**：名称、等级 `n / max`、当前效果、下一级效果、下一级价格、是否可购买及原因（钱包不足 / 已满级 / 本档口已看满 / 存档不可写 / 前置未解锁）。
- **钱包常驻**：升级页与收据页同屏显示当前余额；负数带负号。
- **确认步骤**：购买不可退款，点击后需要一次确认（确认框显示价格与购买后的余额），确认后才调用 `purchase`。确认框不是二次校验，`purchase` 仍须完整校验。
- **失败态**：`COMMIT_FAILED` 显示「未保存，购买未生效」并恢复按钮；`APPLIED_UNSAVED` 在钱包旁显示「本次会话不保存」标记。
- **满级与看满**：`MAX_LEVEL` 显示「已满级」；`NO_EFFECT` 显示「本档口已看满（最多 Z 道）」。
- **读屏与缩放**：价格、等级、效果都可被读屏单独读取；字号放大到 150% 时卡片不换行错位；对比度 ≥ 4.5:1。完整要求见未来的 `design/accessibility-requirements.md`。
- **首局引导**：可以高亮一项推荐升级并显示「推荐」，但不得自动购买、不得替玩家确认。

> **📌 UX Flag — 局外升级**：本系统有 UI 需求。Phase 4 写 epic 之前，先用 `/ux-design` 为升级页与收据页的钱包区出规格；相关 story 引用 `design/ux/` 下的对应文件，不直接引用本 GDD。

## Cross-References

| 文档 | 关系 | 本系统引用/依据 |
|---|---|---|
| `design/gdd/settlement-and-entry-fee.md` | 上游 | `net_profit`（有符号、不钳制）、`RunResult`；Open Question 1/6 的答案 |
| `design/gdd/save-system.md` | 上游 | `progression` 命名空间、`commit` 事务、`mark_dirty`/`flush`、`SaveStatus`、等级钳制与未知 id 保留 |
| `design/gdd/data-config.md` | 上游 | `UpgradeData` 字段、`StationData.wave_size`、`Ready` 前提、id 发布后不可改名 |
| `design/gdd/run-flow.md` | 下游（间接） | `upgrade_snapshot` 的三个必填键与冻结语义；Open Question 11 的答案 |
| `design/gdd/cloying-and-taste.md` | 下游 | 抗性表达式与 Open Question 9 的答案 |
| `design/gdd/menu-waves.md` | 下游 | `menu_vision` 的消费方式与上限；Open Question 3 |
| `design/gdd/dish-values.md` | 校准来源 | `opt_value = 104`、`unit_value`；`capacity_ceiling` 的输入 |
| `design/gdd/game-concept.md` | 来源 | MVP 三项升级、会话级停点、长期成长与「数字看得见地涨」 |
| `design/gdd/systems-index.md` | 索引 | 优先级 MVP、层 Feature、设计顺序 #11、依赖 `结算与入场费 / 存档 / 数据配置` |

**待回指**：界面流程、首局引导的 GDD 写好后，必须在各自的 Dependencies 与 Cross-References 中列出「局外升级」，并写明「只消费快照，不重算等级」。胃容量、纪录与里程碑、音效反馈、解锁与档口已回指。

## Acceptance Criteria

每条都能由 QA 在不读本 GDD 的情况下独立执行。测试夹具放在 `tests/fixtures/meta-upgrades/`；所有涉及写盘失败的用例使用可注入的失败点，不依赖真实杀进程。

**钱包**

1. **GIVEN** 新生涯，**WHEN** 读取状态，**THEN** `wallet = 0`、`upgrade_levels` 为空表、`resolve_upgrade_snapshot()` 返回 `capacity_bonus = 0`、`cloying_resistance = 0.0`、`menu_vision = 0`。
2. **GIVEN** `wallet = 0` 与一局 `net_profit = +19` 的 `RunResult`，**WHEN** 编排者入账，**THEN** `wallet = 19`，编排者对 `progression` 与 `records` 只调用一次 `commit`（携带完整的新钱包值，不是增量），信号 `progression_changed` 恰好发一次。
3. **GIVEN** `wallet = 19` 与 `net_profit = −31`，**WHEN** 入账，**THEN** `wallet = −12`；**不**钳制为 0。
4. **GIVEN** 同一 `RunResult` 连续入账两次，**WHEN** 第二次调用，**THEN** 返回 `ALREADY_APPLIED`，钱包仍为第一次的结果，`commit` 只发生一次。
5. **GIVEN** 注入编排者 `commit` 失败，**WHEN** 入账 `+19`，**THEN** 返回 `APPLIED_UNSAVED`，内存钱包为 19，`progression` 为 dirty；随后恢复写盘并 `flush`，磁盘上的钱包为 19。
6. **GIVEN** 注入 `wallet + net_profit` 溢出 int64，**WHEN** 入账，**THEN** 返回 `WALLET_OVERFLOW`，钱包保持原值，未写档。

**价格**

7. **GIVEN** `upgrade_stomach_capacity`（`cost = 12`、`cost_growth = 1.35`、`max_level = 5`），**WHEN** 依次计算 L1–L5 价格，**THEN** 结果为 `12, 17, 22, 30, 40`，`tree_cost = 121`。
8. **GIVEN** 同一记录，**WHEN** 在等级 5 调用 `next_price`，**THEN** 返回 `null`；`purchase` 返回 `MAX_LEVEL`。
9. **GIVEN** `upgrade_menu_vision`（`cost = 16`，`max_level = 3`），**WHEN** 计算价格，**THEN** 为 `16, 22, 30`，`tree_cost = 68`。
10. **GIVEN** `upgrade_cloying_resistance`（`cost = 20`，`max_level = 1`），**WHEN** 计算价格，**THEN** 只为 `20`。

**购买事务**

11. **GIVEN** `wallet = 30`、容量 L1 价格 12、存档可写、顶层为 `Menu`，**WHEN** 购买，**THEN** 成功后 `wallet = 18`、等级 = 1；`commit` 携带 `wallet = 18` 与完整等级表；**确认只在 `commit` 成功后返回**。
12. **GIVEN** 同上的状态但注入 `commit` 失败，**WHEN** 购买，**THEN** 返回 `COMMIT_FAILED`，钱包仍为 30、等级仍为 0；界面可重试。
13. **GIVEN** `wallet = 19`、价格 20，**WHEN** 购买，**THEN** 返回 `INSUFFICIENT_FUNDS`，状态不变，未调用 `commit`。
14. **GIVEN** `wallet = 20`、价格 20，**WHEN** 购买，**THEN** 成功，`wallet = 0`。
15. **GIVEN** 未知升级 id，**WHEN** 购买，**THEN** 返回 `UNKNOWN_UPGRADE`；`get_level` 返回 0。
16. **GIVEN** 数据配置未 `Ready`，**WHEN** 购买，**THEN** 返回 `DATA_NOT_READY`；**GIVEN** 存档为 `Blocked`，**WHEN** 购买，**THEN** 返回 `NOT_WRITABLE`。
17. **GIVEN** 顶层状态为 `Run`，**WHEN** 购买，**THEN** 返回 `NOT_IN_MENU`，等级不变。
18. **GIVEN** 一笔购买仍在等待 `commit`，**WHEN** 再次购买，**THEN** 返回 `BUSY`，第二次不扣款。
19. **GIVEN** 夹具中一条升级的 `prerequisite_id` 指向等级 0 的记录，**WHEN** 购买，**THEN** 返回 `PREREQUISITE_LOCKED`；前置升到 1 后可购买。

**快照与效果**

20. **GIVEN** 等级表 `{容量 2, 视野 1, 解腻 1}`，**WHEN** 解析快照，**THEN** `capacity_bonus = 2`（int）、`menu_vision = 1`（int）、`cloying_resistance = 1.0`（float），且 `levels` 是同一份表的副本；修改返回值不影响内部状态。
21. **GIVEN** 两条同类型升级各 1 级，**WHEN** 以两种插入顺序解析，**THEN** 两次结果相同。
22. **GIVEN** 存档含 `upgrade_unknown: 3`，**WHEN** 解析，**THEN** 它贡献 0，但仍出现在 `levels` 副本中；载入后存档仍保留该条。
23. **GIVEN** 存档等级 7、记录 `max_level = 5`，**WHEN** 载入并解析，**THEN** 存档写钳制 warning，快照按 5 计算，不退钱。
24. **GIVEN** `group_cloy = 1`、`cloy_cap = 3`、`cloying_resistance = 1.0`，**WHEN** 计算追加饱腹，**THEN** 为 0；**GIVEN** `group_cloy = 3`、抗性 1，**THEN** 为 2；**GIVEN** 抗性 5，**THEN** 为 0，不为负。
25. **GIVEN** 抗性总分 1.5，**WHEN** 生效，**THEN** `floor` 后为 1，与抗性 1.0 相同。
26. **GIVEN** 一条 `max_level = 4` 的视野夹具记录（`effect_value = 1`）与 `wave_size = 4` 的档口，**WHEN** 在当前等级 3 尝试购买第 4 级，**THEN** `purchase` 返回 `NO_EFFECT`；**GIVEN** 当前等级 3，**THEN** 解析值仍为 3，菜单波次预览为 3 道。
27. **GIVEN** 一局开始时快照为 `{容量 1}`，**WHEN** 在局内达到可购买条件后尝试购买，**THEN** 被拒绝；下一局开局快照才变为 `{容量 2}`。

**生命周期与确定性**

28. **GIVEN** 生涯被删除，**WHEN** 重新读取状态，**THEN** 钱包 0、等级表空、快照全 0；没有残留缓存。
29. **GIVEN** 同一记录集与同一等级表，**WHEN** 在任意帧序、任意平台重复解析 100 次，**THEN** 价格与快照逐字段相同。
30. **GIVEN** 静态检查源码，**THEN** GDScript 中没有硬编码的升级价格、效果值、增长率或钱包上下限；这些值只能来自 `UpgradeData`（含 `cost_growth`）。
31. **GIVEN** 正常等级表（≤ 12 条），**WHEN** 解析快照或计算价格，**THEN** 单次 ≤ 1 ms（逻辑预算，不含磁盘）。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | **首局引导的 6 道菜局会不会把新玩家打进债务？** 本系统不特判首局：若引导局的菜池上限低于入场费 85，玩家第一局就负资产。建议首局使用 `entry_fee = 0` 或专用低费档口（数据配置），而不是给钱包开特例。 | ux-designer（首局引导）+ economy-designer | 首局引导 GDD 完成时 |
| 2 | **负钱包是否需要一个债务上限或「重新开始」出口？** 当前无下限、无救济，玩家可以一直负着玩。若试玩显示挫败感过强，可加入「最低 0」或一次性清零，但那会改变「回本」的含义，需重写规则 3。 | economy-designer | `/prototype` 与首轮试玩后 |
| 3 | **容量升级的收益被 10 回合封顶。** 26 格后每点容量都是废等级；是否需要「回合 +1」作为第四项升级（属于时间与回合），或把容量效果改为每级 +2？ | game-designer + economy-designer | 时间与回合 GDD 完成时 |
| 4 | **多档口时，`menu_vision` 的等级上限按哪个 `wave_size` 算？** 当前只有一个档口；若各档口 `wave_size` 不同，升级页需要显示每个档口的上限，或改为允许购买但标注「本档口已看满」。 | ux-designer + game-designer | Vertical Slice 规划时 |
| 5 | **解腻抗性是否永远定点减 1？** 大组出现后，定点减法与「每级减 1、上限 `cloy_cap−1`」是否仍能保持第二轴的张力，需要按 `menu_cloy_pressure` 重新评估。 | systems-designer（腻度 + 局外升级） | 第二个档口设计时 |
| 6 | **升级页是否需要「推荐购买」顺序？** 首局引导可能想高亮一项；本系统只提供价格与可负担性，不定义推荐算法。 | ux-designer（首局引导） | `/ux-design` 升级页时 |
| 7 | **购买是否要二次确认？** UI 要求暂定「要」，因为不可退款；若试玩显示每次购买都确认太啰嗦，可只对价格 ≥ 某阈值的购买确认。 | ux-designer | `/ux-design` 升级页时 |
| 8 | **`last_applied_run_id` 不持久化。** 会话级去重依赖结算的「每局一次」保证；若未来加入局史或断线续跑，需要把它写进存档，并走 schema 迁移。 | lead-programmer（存档） | 纪录与里程碑 GDD 完成时 |
| 9 | **价格增长率是否需要按升级分别配置？已解决（2026-11-03）**：给 `UpgradeData` 增加 `cost_growth: float`（默认 1.35），每条记录自带增长率；MVP 三项均为 1.35。需数据配置 GDD 回改该字段。 | systems-designer + lead-programmer | 数据配置回改时 |
| 10 | **升级页在会话中的位置。** 当前只从 `Menu` 进入，且收据后自然引导；是否允许玩家在任意菜单停留点打开升级页，由界面流程决定，不影响本系统事务。 | ux-designer（界面流程） | 界面流程 GDD 完成时 |
| 11 | **档口解锁的扣款接口。** **已回答（2026-11-03）**：采用通用 `spend(amount, reason, extra)`，与购买一样先落盘再确认；见 `design/gdd/unlocks-and-stations.md`。 | systems-designer（解锁与档口） | 解锁与档口 GDD 完成时 |
