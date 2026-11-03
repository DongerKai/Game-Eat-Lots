# 解锁与档口（Unlocks and Stations）

> **Status**: In Design
> **Author**: user + agents
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 数字看得见地涨（局外进度可见）；承载概念文档的 Curiosity 留存钩子（未解锁的档口和菜谱）

## Summary

解锁与档口决定玩家现在能进哪些档口、每个档口里哪些菜可吃。它把 `StationData.unlock_cost` 变成一次可购买的解锁，把 `unlock_cost = 0` 的档口视为开局就有，并把已解锁档口的整个菜池记进存档的 `unlocks` 命名空间。它不碰钱包，也不写盘：付费解锁通过局外升级的 `spend` 事务扣款，并与 `unlocks` 在同一次 `commit` 里生效。

> **Quick reference** — Layer: `Feature` · Priority: `Vertical Slice` · Key deps: `局外升级`, `存档`, `数据配置`

## Overview

解锁与档口回答两个问题：**玩家现在能进哪些档口**，以及**每个档口里哪些菜可吃**。MVP 只有一个 `unlock_cost = 0` 的档口，所以这个系统在 MVP 的主要工作不是卖东西，而是保证「存档里没有任何解锁记录」永远不等于「没有可玩的档口」：免费档口与它的菜由数据直接推导，集合为空时在内存中回填，可用性不等待写盘。

Vertical Slice 加入第二个付费档口后，玩家在菜单里用钱包买下它。扣款不走本系统：解锁调用局外升级的 `spend` 事务，由它计算新钱包，并把本系统算好的 `unlocks` 并入同一次写盘；写盘失败时钱包与解锁一起回滚。这保证了「界面说解锁了、重启后没了」不会发生。

解锁是**单调**的。本系统从不删除任何解锁 id：数据里消失的档口与菜保留在集合中，只是不再生效；档口涨价不会让已经写进存档的玩家失去它（从未写盘的空集合是唯一例外）；唯一的清空路径是删除生涯。这条单调性让解锁集合可以承受数据改版与存档迁移，而不需要在两个系统里各写一遍规则。

## Detailed Design

### Core Rules

1. **职责边界** — 本系统拥有：档口与菜品的可用性判定、免费档口与免费菜的回填、付费解锁的校验顺序与成交事务编排、`unlocks` 命名空间的内容（`stations` / `dishes` 两组 id）、解锁进度计数、锁定与失效判定。它不拥有：钱包的加减与负值策略（局外升级）、持久化与原子写入（存档）、`StationData` 的 schema 与校验（数据配置）、档口选择与开局（界面流程 / 餐局流程）、入场费与净利（结算与入场费）、纪录排名（纪录与里程碑）、界面绘制（界面流程）。

2. **可用性是推导的，不是查集合** — 档口可用当且仅当 `unlock_cost = 0` 或它的 id 在 `unlocks.stations` 中。这让**空集合成为合法且可玩的状态**，直接回答存档 GDD 的前置契约（存档 AC 20：存档不自行添加免费档口）。免费档口的存在不依赖任何存档写入；「回填」只是把推导结果写回集合，不改变可用性。

3. **菜的可用性跟随档口** — MVP 与 Vertical Slice 的解锁粒度是档口：一个档口被解锁，它的整个 `dish_pool` 一起可用。`unlocks.dishes` 是**已解锁菜的账本**（供界面计数、纪录展示与存档体积使用），不是第二道闸门。可用性判定不读它——否则会出现「档口已解锁、菜没写进集合」的不可玩状态，而那个状态只能靠一次额外的写盘来修复。

4. **数据里至少有一个免费档口** — 每份合法数据都应至少有一个 `unlock_cost = 0` 的档口。数据配置当前不校验这一点，本系统在运行期兜底：若某次载入后可用档口集合为空，载入结果为 `NoPlayableStation`，菜单禁用「开始」并显示明确原因。**建议数据配置把「至少一个 `unlock_cost = 0` 的档口」加为启动校验**（见待回改）。

5. **解锁是单调的：本系统不删除任何 id** — 数据里消失的档口/菜保留在集合中（存档规则 14）、重复 id 去重、免费档口的 id **一旦回填写盘**，即使 `unlock_cost` 后来被调高也仍然可用。唯一的清空路径是 `delete_career()`（存档）。**这就是「删除条件」：本系统没有删除路径，只有生涯删除会清空。** 空集合若从未写盘就遇到涨价，是单调性的唯一例外，见规则 14 与 Edge Cases。

6. **付费解锁是一次事务：先落盘，再确认** — 顺序固定为：本系统校验 → 调用局外升级 `spend`（扣款、合并 `unlocks`、`commit`）→ `spend` 成功才返回成交。本系统只在 `spend` 成功后把准备好的集合替换为当前集合；失败时保留旧集合。写盘失败则钱包与解锁一起回滚，返回 `COMMIT_FAILED`，界面显示「未保存，解锁未生效」。这是存档规则 17 的落实。

7. **钱包只由局外升级写** — 本系统从不直接改写 `progression.wallet`。它构造好新的 `unlocks` 值，调用局外升级的 `spend(amount, reason, extra)`：由后者验证余额、计算 `wallet' = wallet − amount`、把 `extra`（即 `{unlocks: …}`）并入同一个 `save.commit`。这同时保住「钱包归局外升级」与「解锁归本系统」，也回答局外升级 Open Question 11。

8. **解锁只在菜单发生** — 局内、收据未完成、场景切换中一律返回 `NOT_IN_MENU`。同一时刻只允许一笔解锁在途（`BUSY`）；不排队、不合并。解锁不消耗回合、不改当前局。

9. **校验顺序固定** — `DATA_NOT_READY` → `NOT_WRITABLE` → `NOT_IN_MENU` → `UNKNOWN_STATION` → `INVALID_STATION_DATA` → `BUSY` → `ALREADY_UNLOCKED` → `PREREQUISITE_LOCKED` → `INSUFFICIENT_FUNDS`。这一串是**成交前**的校验；`OK` 与 `COMMIT_FAILED` 是事务结果，排在校验之后，不再回到上面任何一条。`INVALID_STATION_DATA` 只用于 `unlock_cost < 0` 这类已越过启动校验的单条记录损坏——它不能落进 `INSUFFICIENT_FUNDS`，否则负价格会让玩家「花 −1 元」反而加钱。同一状态下永远返回同一个错误，界面可以稳定地显示同一个原因。

10. **价格就是记录上的 `unlock_cost`** — 本系统不发明价格曲线。一次解锁固定花 `unlock_cost`，不随已解锁数量、钱包余额或已玩局数浮动。调价只改 `.tres`。已解锁的档口不退款、不补差价。`unlock_cost` 是**一次性**价格，与每局都收的 `entry_fee` 是两回事。

11. **前置门槛是等级，不是消耗** — 若档口声明了 `unlock_requires`（升级 id 列表），其中每个升级的当前等级必须 ≥ 1，否则返回 `PREREQUISITE_LOCKED`。该字段**当前不存在于 `StationData`**，MVP 与 Vertical Slice 的档口都不使用它；实现时按空列表处理（见 Open Question 4）。局外升级的 `prerequisite_id` 链保证玩家不能绕过更底层的升级前置。

12. **数据未就绪时整体不可用** — 数据配置不在 `Ready` 时，目录查询返回空、解锁返回 `DATA_NOT_READY`；不显示占位价格、不用默认值兜底。本系统不持有自己的价格表副本。

13. **存档不可写时只读** — `LoadReport.writable = false`（`Blocked`）或存档尚未载入完成时，解锁返回 `NOT_WRITABLE`；可用性查询与进度照常显示，玩家仍能进已解锁的档口。

14. **回填是内存优先、写盘跟随现有写入时点** — 载入后立即计算 `effective_unlocks`（`effective_unlocks` 公式）。若它与持久化的集合不同，在内存中采用推导值并把 `unlocks` 标记为 dirty：新建生涯（`Fresh`）时随存档的首次基线一起写出；已有生涯则在下一个写入时点（结算 / 购买 / 退出 / flush，见存档规则 6）落盘。可用性不等待写盘，界面不会因为一次晚到的写盘闪回锁定态。这条规则留下一个很窄的窗口：旧档的空集合若在退出前从未写盘，而两次会话之间数据把该免费档口改为付费，下一次载入会把它当作付费档口——见 Edge Cases 的「免费档口涨价」，根治方式是新建生涯立即写基线，本设计已要求。

15. **不参与结算事务** — 一局结束不改变解锁（MVP 与 Vertical Slice 都不存在局内奖励解锁）。若未来某一局要奖励解锁，必须并入结算事务（存档规则 80 与纪录 GDD 的编排契约），不得让解锁单独写盘。**这是对纪录与里程碑 Open Question 4 的回答：完成条件在 MVP 与 Vertical Slice 只展示，不发放解锁。**

16. **确定性、无副作用** — 可用性、规范化、进度只依赖数据记录与 `unlocks` 集合，不读时钟、不用随机数、不做 I/O。所有集合输出按 id 升序，便于比对、写测试与存档 diff。

### States and Transitions

本系统只持有解锁事务的短状态；生涯数据属于存档。载入结果与运行状态分开描述。

**载入结果**（`get_load_outcome()`）：

| 结果 | 判定 | 行为 |
|---|---|---|
| `Playable` | 可用档口集合非空，且 `persisted = effective_unlocks` | 正常；无回填、无写盘 |
| `Backfilled` | 可用档口集合非空，且 `persisted ≠ effective_unlocks`（缺免费档口、缺免费菜，或已解锁档口的菜池新增了菜） | 内存采用规范化值并发一次 `unlocks_changed`；存档可写时标记 dirty，`Blocked` 时不标记 dirty、每次载入重新推导；不报错 |
| `NoPlayableStation` | 可用档口集合为空 | 菜单禁用「开始」并显示原因；不解锁、不写盘 |
| `Unavailable` | 数据配置未 `Ready` | 目录为空；解锁一律 `DATA_NOT_READY` |

**运行状态**：

| 状态 | 触发 | 下一个状态 | 说明 |
|---|---|---|---|
| `Uninitialized` | 应用启动 | `Ready` / `ReadOnly` / `Unavailable` | 等数据 `Ready` 与存档载入完成 |
| `Ready` | 数据 `Ready` 且存档载入完成、可写 | `Unlocking` / `Ready` | 可查询、可解锁 |
| `Ready` | 存档 `Blocked`（不可写） | `ReadOnly` | 可查询、不可解锁 |
| `Unavailable` | 数据未 `Ready` | `Ready` / `ReadOnly` | 空目录，拒绝解锁 |
| `Unlocking` | `unlock(id)` 通过校验，等待 `spend` / `commit` | `Ready` | 同时只允许一笔；第二笔返回 `BUSY` |
| `Unlocking` | `commit` 成功 | `Ready` | 内存与磁盘一致，返回成交 |
| `Unlocking` | `commit` 失败 | `Ready` | 钱包与解锁一起回滚，返回 `COMMIT_FAILED` |
| `ReadOnly` | 新建生涯或存档恢复可写（新的 `SaveStatus`） | `Ready` | 方向由存档状态驱动，本系统不自行切换 |

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 数据配置 | 上游 | `get_station(id)` → `id`、`display_name`、`unlock_cost`、`dish_pool`；`get_dish(id)` 存在性；前提 `Ready` | 硬 |
| 存档 | 上游 | 读 `unlocks.stations` / `unlocks.dishes`；`commit`（经局外升级的 `spend`）、`mark_dirty('unlocks')`、`SaveStatus`、`LoadReport.writable` | 硬 |
| 局外升级 | 上游 | `spend(amount, reason, extra)`（新增接口，见下）、`get_wallet()`、`get_level(id)`、`progression_changed` | 硬 |
| 界面流程 | 下游 | 档口目录、可解锁性与原因、`unlock(id)`、解锁进度；档口选择只在菜单 | 硬（UI） |
| 餐局流程 | 下游（协调） | 界面流程只把**可用**档口放进 `RunConfig.station_id`；餐局流程现有的存在性校验不是授权校验 | 软（协调） |
| 菜单波次 | 下游（间接） | 只读所选档口的 `dish_pool`；本设计下无需过滤。若未来做逐菜解锁，必须改读 `get_available_dishes(station_id)` | 软（间接） |
| 纪录与里程碑 | 下游 | 读取解锁进度做展示；按 `station_id` 归档。**不发放解锁**（回答其 Open Question 4） | 软 |
| 首局引导 | 下游 | 可读取锁定档口与价格做引导高亮；不得自动解锁或自动购买 | 软 |
| 音效反馈 | 下游 | 无直接调用（音效 GDD 已声明）；若 Vertical Slice 要解锁音效，由界面流程在收到 `unlocks_changed` 后触发 | 软 |
| 场景与流程管理 | 时序 | 只影响顶层状态：解锁只在 `Menu` 生效；本系统不驱动顶层切换 | 软（编排） |
| 在线挑战 | 未来 | 挑战可能限定档口；MVP 不存在，不建立依赖 | — |

**接口所有权**：本系统拥有可用性语义、解锁条件、`unlocks` 内容与进度；局外升级拥有钱包与扣款事务；存档拥有字段持久化与原子性；数据配置拥有记录 schema；界面流程拥有档口选择与解锁入口的绘制。

**对外接口（设计级）**：

| 接口 | 输入 | 输出 | 说明 |
|---|---|---|---|
| `get_load_outcome()` | — | `Playable` \| `Backfilled` \| `NoPlayableStation` \| `Unavailable` | 载入后一次确定；供菜单决定是否禁用「开始」 |
| `get_station_catalog()` | — | `StationView[]` | 每个档口的 id、名称、是否可用、解锁价、锁定原因、菜数；升序 |
| `get_available_stations()` | — | `StringName[]` | 可用档口 id，升序；`NoPlayableStation` 时为空 |
| `is_station_available(id)` | 档口 id | `bool` | 未知 id 返回 `false` |
| `get_available_dishes(station_id)` | 档口 id | `StringName[]` | 该档口可用菜的 id；不可用档口返回空 |
| `can_unlock(id)` | 档口 id | `{ok, reason, cost}` | 只校验，不改状态、不扣款；`ALREADY_UNLOCKED` 时 `cost = null` |
| `unlock(id)` | 档口 id | `UnlockResult` | 事务；成功后磁盘已更新 |
| `get_unlock_progress()` | — | `{stations, stations_total, dishes, dishes_total}` | 只统计数据中存在的记录 |
| `get_effective_unlocks()` | — | `{stations, dishes}` | 规范化后的集合（含免费档口）；新副本 |
| `unlocks_changed` | — | 信号 | 携带进度；成功解锁或回填改变集合后发一次；被拒绝的解锁不发 |

`get_station_catalog()` 总是列出数据中存在的全部档口（升序），包括锁定的；`NoPlayableStation` 时它是界面唯一还能显示的东西。只有 `Unavailable`（数据未 `Ready`）才返回空。

**对局外升级的契约（新增 `spend`，需回写其 GDD）**：

| 项 | 定义 |
|---|---|
| 签名 | `spend(amount: int64, reason: SpendReason, extra: Dictionary = {}) -> SpendResult` |
| `reason` | `STATION_UNLOCK`（唯一取值；枚举为未来的其他流水预留） |
| `extra` | 本系统算好的 `{unlocks: {stations: [...], dishes: [...]}}`；局外升级不解释它，只并入事务 |
| 校验顺序 | `DATA_NOT_READY` → `NOT_WRITABLE` → `NOT_IN_MENU` → `BUSY` → `INVALID_AMOUNT` → `INSUFFICIENT_FUNDS`；`amount < 0` 一律 `INVALID_AMOUNT`，钱包不变 |
| 事务 | `wallet' = checked_sub_int64(wallet, amount)`；`save.commit({progression: {wallet: wallet', upgrade_levels: …}, …extra})` |
| 失败 | 余额不足：`INSUFFICIENT_FUNDS`，不改内存、不写盘；写盘失败：恢复钱包、`extra` 不生效、返回 `COMMIT_FAILED` |
| 成功 | 内存与磁盘一致后才返回；`progression_changed` 发一次 |
| 并发 | 与升级购买共用同一条在途约束（`BUSY`），不并存两笔 |

**对存档的契约**：

- 本系统只经 `unlocks` 命名空间读写；`unlocks.stations` / `unlocks.dishes` 只含 id，不含价格、名称或任何派生值。
- 空集合合法。存档不添加免费档口（存档 AC 20），回填由本系统负责；本系统在载入后把规范化值标记 dirty——新建生涯随首次基线写出，已有生涯在下一个写入时点落盘。
- 未知 id 原样保留、原样写回，不计入进度，不报错。
- 一次解锁的所有变化（`progression.wallet` 与 `unlocks`）必须落在**同一个事务**里，不拆成两次写入。
- 删除生涯后本系统读到默认值：钱包 0、解锁集合为空；免费档口立即可用，下一次载入即回填。

**对数据配置的契约**：

- `unlock_cost = 0` 的语义是「开局可用」，不是「可以 0 元购买」。界面不得为它显示购买按钮。
- `unlock_cost` 是一次性解锁价，`entry_fee` 是每局入场费；两者不得互相推导。
- 数据改版时，新增到已解锁档口菜池里的菜自动可用；从菜池移除的菜不再可用，但 id 留在账本里。

**待回改（不在本文件执行，交由编排会话统一处理）**：见 Dependencies 章节末尾的「待回改」清单。

## Formulas

本系统定义五组运行时规则（可用性、规范化、解锁判定、成本、进度）与一组设计期配速量。运行时规则必须可判定；设计期量只用于定价与验收带。下文示例中的 `station_banquet` 是 Vertical Slice 第二个档口的**暂定 id**，不是已发布数据。

### 档口可用性 `station_available`

The `station_available` formula is defined as:

`station_available(s, U) = (s.unlock_cost = 0) ∨ (s.id ∈ U.stations)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `s` | `StationData` | 数据中存在的记录 | data file | 一个档口记录 |
| `s.unlock_cost` | int | 0–INT32_MAX | data file | 一次性解锁价（元）；0 表示开局可用 |
| `s.id` | `StringName` | `^[a-z][a-z0-9_]*$` | data file | 稳定的档口 id |
| `U.stations` | `StringName[]` | 任意长度，元素唯一 | `unlocks.stations` | 已持久化的解锁档口 id |
| `station_available(s, U)` | bool | {true, false} | calculated | 该档口当前是否可进入 |

**Output Range:** `{true, false}`，无中间态。未知档口 id 不进入本公式（调用方先返回 `UNKNOWN_STATION` 或 `false`）。

**Example:** `station_canteen.unlock_cost = 0`、`U.stations = []` → `station_available = true`。`station_banquet.unlock_cost = 120`、`U.stations = []` → `false`；把它写进 `U.stations` 后 → `true`。

### 规范化与回填 `effective_unlocks`

The `effective_unlocks` formula is defined as:

```
effective_unlocks(U, S) =
  let S_free = {s ∈ S : s.unlock_cost = 0}
      S_eff  = S_free ∪ {s ∈ S : s.id ∈ U.stations}
  in (
      stations = sort( {s.id : s ∈ S_eff} ∪ (U.stations \ S_ids) ),
      dishes   = sort( U.dishes ∪ ⋃_{s ∈ S_eff} s.dish_pool )
  )
```

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `U` | `{stations, dishes}` | 两个 StringName 数组 | `unlocks` 命名空间 | 持久化的解锁集合 |
| `S` | `StationData[]` | ≥ 0 项 | data file | 全部档口记录 |
| `S_ids` | `StringName[]` | = `{s.id : s ∈ S}` | calculated | 已知档口 id 集 |
| `D_ids` | `StringName[]` | = `{d.id : d ∈ D}` | calculated | 已知菜品 id 集 |
| `S_free` | `StationData[]` | 0–`|S|` | calculated | 免费档口 |
| `S_eff` | `StationData[]` | 0–`|S|` | calculated | 当前可用的档口 |
| `stations` | `StringName[]` | 0–`|S| + |U.stations|` | calculated | 写回 `unlocks.stations` 的集合 |
| `dishes` | `StringName[]` | 0–`|D| + |U.dishes \ D_ids|` | calculated | 写回 `unlocks.dishes` 的集合 |

**Output Range:** 两个升序、元素唯一的数组。只增不减：`stations ⊇ U.stations`、`dishes ⊇ U.dishes`。`U.stations \ S_ids` 与 `U.dishes \ D_ids` 即未知 id，原样保留。

**Example:** 数据含 `station_canteen`（免费，12 道菜）与 `station_banquet`（120 元，12 道菜），`U = {[], []}` → `stations = [station_canteen]`、`dishes = [12 个 station_canteen 的菜 id]`。这是空集合的全部回填结果，不需要任何玩家操作。玩家买下 `station_banquet` 后 `U.stations = [station_banquet]`，`dishes` 增加 12 个 id。

### 解锁判定 `unlock_decision`

The `unlock_decision` formula is defined as:

```
unlock_decision(s, U, L, wallet) =
  INVALID_STATION_DATA if s.unlock_cost < 0
  ALREADY_UNLOCKED     if station_available(s, U)
  PREREQUISITE_LOCKED  if ∃r ∈ requires(s) : L(r) < 1
  INSUFFICIENT_FUNDS   if wallet < s.unlock_cost
  OK                   otherwise
```

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `s` | `StationData` | 数据中存在 | data file | 目标档口 |
| `s.unlock_cost` | int | 0–INT32_MAX（合法时） | data file | 负值只可能来自绕过启动校验的损坏数据，由第一行拦截 |
| `U` | `{stations, dishes}` | 同上 | `unlocks` | 当前解锁集合 |
| `L` | `map<String, int>` | 等级 ≥ 0 | `progression.upgrade_levels` | 升级等级 |
| `requires(s)` | `StringName[]` | 0–4 项 | data file | `StationData.unlock_requires`，**当前字段不存在，按 `[]` 处理** |
| `wallet` | int64 | −9.22×10¹⁸ – 9.22×10¹⁸ | `progression.wallet` | 当前钱包，可为负 |
| `unlock_decision` | 枚举 | 5 值 | calculated | 校验结果 |

**Output Range:** `{OK, ALREADY_UNLOCKED, PREREQUISITE_LOCKED, INSUFFICIENT_FUNDS, INVALID_STATION_DATA}`。未知档口与数据/存档/菜单前提不进入本公式——它们在校验顺序中更早返回。`INVALID_STATION_DATA` 必须排在 `INSUFFICIENT_FUNDS` 之前：`wallet = 0 < unlock_cost = -1` 为假，否则负价格会被判成 `OK` 并被 `spend` 反向加钱。

**Example:** `station_banquet` 未解锁、无前置、`wallet = 119`、`unlock_cost = 120` → `INSUFFICIENT_FUNDS`；`wallet = 120` → `OK`，成交后钱包为 0。

### 解锁成本 `station_unlock_cost`

The `station_unlock_cost` formula is defined as:

`station_unlock_cost(s) = s.unlock_cost`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `s.unlock_cost` | int | 0–INT32_MAX | data file | 该档口的一次性解锁价（元） |
| `station_unlock_cost(s)` | int | 0–INT32_MAX | calculated | 本次解锁实际要扣的钱 |

**Output Range:** 0–INT32_MAX。`0` 不产生购买路径（免费档口返回 `ALREADY_UNLOCKED`）。负值只可能来自损坏数据，`unlock_decision` 以 `INVALID_STATION_DATA` 拦截，因此 `spend` 永远不会收到负的 `amount`。本公式没有等级、次数或已解锁数量的输入——价格不随任何状态变化。

**Example:** `station_banquet.unlock_cost = 120` → 每次（且只可能有一次）解锁扣 120。已解锁后再调用不扣款，因此不存在「第二次价格」。

### 解锁配速 `runs_to_unlock`

The `runs_to_unlock` formula is defined as:

```
runs_to_unlock(s, p) =
  null                              if opt_net_profit(p) ≤ 0
  ceil( s.unlock_cost / opt_net_profit(p) )   otherwise
```

其中 `opt_net_profit(p) = opt_value(p) − p.entry_fee`（菜品数值 / 结算与入场费提供）。

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `s.unlock_cost` | int | 0–INT32_MAX | data file | 目标档口的一次性解锁价 |
| `p` | `StationData` | 数据中存在 | data file | **前一个**档口（玩家解锁 `s` 时主要刷的那一个） |
| `opt_net_profit(p)` | int | −entry_fee – INT32_MAX | calculated | 前一个档口在最优菜单下的净利；MVP `104 − 85 = 19` |
| `runs_to_unlock` | int \| null | 0–INT32_MAX，或 `null` | calculated | 用最优局偿还解锁价所需局数；设计期量，不进存档 |

**Output Range:** 0（免费档口）到很大的整数，或 `null`（前一个档口的最优局都无法回本）。正常玩法目标带 **4–8 局**。低于 4 会让档口显得廉价、压缩升级树的购买窗口；高于 8 会让第二个档口在首次会话内不可见，失去好奇心钩子（概念文档 Full Vision 的 3–6 局目标可放宽到 8）。`null` 是校准错误信号，由结算负责修复，不用 `max(1, …)` 把它伪装成一个很大的局数。

**Example:** `station_banquet.unlock_cost = 120`、`opt_net_profit(station_canteen) = 19` → `ceil(120 / 19) = 7` 局。

### 解锁进度 `unlock_progress`

The `unlock_progress` formula is defined as:

```
unlock_progress(U, S) =
  let S_eff = {s ∈ S : s.unlock_cost = 0 ∨ s.id ∈ U.stations}
  in (
      stations       = |S_eff|,
      stations_total = |S|,
      dishes         = Σ_{s ∈ S_eff} |s.dish_pool|,
      dishes_total   = |D|
  )
```

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `S_eff` | `StationData[]` | 0–`|S|` | calculated | 当前可用档口 |
| `D` | `DishData[]` | ≥ 0 项 | data file | 全部菜品记录 |
| `stations` / `dishes` | int | 0–`|S|` / 0–`|D|` | calculated | 已解锁计数 |
| `stations_total` / `dishes_total` | int | ≥ 0 | calculated | 数据中的总数 |

**Output Range:** 计数与总量都用 int64 承载，`dishes ≥ dishes_total` 不可能发生（每个菜属于恰好一个档口，由数据配置双向校验保证）。未知 id 不计入分子，也不计入分母。**若数据里没有任何档口（`S = ∅`）**，四个计数全为 0，载入结果为 `NoPlayableStation`；界面显示该错误，不显示 `0/0` 比值，也不做除法。

**Example:** MVP 只有 `station_canteen`（12 道菜）→ `1/1`、`12/12`。Vertical Slice 两档口各 12 道、只解锁免费档口 → `1/2`、`12/24`。

## Edge Cases

**载入与回填**

- **If 解锁集合为空且存在 `unlock_cost = 0` 的档口**：可用性立即为真；`get_load_outcome() = Backfilled`；`unlocks_changed` 发一次；集合随新建生涯的首次基线写出，或在已有生涯的下一个写入时点落盘。不报错——存档的 `EMPTY_UNLOCKS` 只是警告，不是失败（存档 AC 20）。
- **If 解锁集合为空且没有任何 `unlock_cost = 0` 的档口**：`get_load_outcome() = NoPlayableStation`；`get_available_stations()` 为空；菜单禁用「开始」并显示「没有可用档口——请检查数据」。不解锁、不写盘、不自动创建一个档口。
- **If 集合含免费档口但缺它的菜**：规范化把菜补进账本；可用性本来就按档口推导，所以没有不可玩窗口。
- **If 集合含未知档口 id**：保留、原样写回、不计入进度、不出现在目录里；不报错（存档规则 14）。若它是改名前的旧 id，玩家需要在新 id 上重新解锁——id 改名等于删除加新增，这是数据配置规则 6 的既定后果。
- **If 集合含未知菜 id**：保留、原样写回、不计入进度；不报错。
- **If 存档为 `Blocked`**：状态为 `ReadOnly`；可用性、目录与进度正常显示；解锁返回 `NOT_WRITABLE`。回填只改内存，不尝试写盘。

**解锁**

- **If 钱包恰好等于 `unlock_cost`**：允许，扣完为 0；成交。
- **If 钱包小于 `unlock_cost`**：`INSUFFICIENT_FUNDS`，钱包与集合不变，不写盘、不发信号。
- **If 钱包为负**：一律买不起；负资产不阻止玩家继续进免费档口，也不阻止玩已有的档口。
- **If 目标是免费档口（`unlock_cost = 0`）**：`ALREADY_UNLOCKED`；界面不应显示购买按钮，也不存在「0 元成交」路径。
- **If 档口已在集合中**：`ALREADY_UNLOCKED`，不重复扣款。
- **If 档口 id 不存在于数据**：`UNKNOWN_STATION`；不读钱包、不写盘。
- **If 前置升级未满足**：`PREREQUISITE_LOCKED`；不扣款，界面显示缺哪一项（字段启用后）。
- **If 上一笔解锁仍在等待 `commit`**：`BUSY`；不排队、不合并，界面禁用按钮直到回调。
- **If 当前不在 `Menu`**（局内、收据未完成、场景切换中）：`NOT_IN_MENU`。
- **If 数据未 `Ready`**：`DATA_NOT_READY`；目录为空，不显示占位价格。
- **If `commit` 失败**：钱包与解锁一起回滚到解锁前，返回 `COMMIT_FAILED`；界面显示「未保存，解锁未生效」，允许重试。磁盘上没有任何变化。
- **If 解锁成功后立刻崩溃**：磁盘已更新；重启后档口仍在、钱包已扣——这是「先落盘再确认」的全部意义。
- **If 同一帧内先结算、后解锁**：两笔事务按到达顺序串行；解锁不并入结算事务，结算也不等待解锁。写入进行中到达的第二个事务排队，不交错（存档规则的状态表）。正常 UI 下两者不会重叠：收据只在结算 `commit` 成功之后才显示，而解锁只在 `Menu` 且收据关闭后可用。存档 GDD 中「结算与解锁合并为同一事务」只适用于**结算本身产生解锁变化**的情形；MVP/VS 不存在这种变化（见规则 15），因此两笔串行即可。

**数据变化与删除**

- **If 已解锁档口被从数据删除**：id 保留在集合中；它不再出现在可用列表与目录里；不退钱；可在载入报告中列为 warning，但不报错。
- **If 档口 `unlock_cost` 从 0 调高**：如果回填已经写盘（新建生涯的基线、退出前的 flush、任何一次结算或购买都会写），id 已在集合中，档口保留，不因涨价失去。**唯一例外**：旧档的空集合在本次会话从未写盘，且两次会话之间数据把该档口改为付费——下一次载入它被视为未解锁。载入报告能看出集合曾经为空（存档的 `EMPTY_UNLOCKS` 警告即此用途）；根治方式是新建生涯立即写基线，本设计已要求。
- **If 档口 `unlock_cost` 从 > 0 调低**：未解锁玩家按新价购买；已解锁玩家不退款、不补差价。
- **If 档口 `display_name` 改变**：id 不变，解锁状态不变，目录显示新名称。
- **If 档口 id 改变**：旧 id 成为未知条目被保留，新 id 需要重新解锁。发布后不得改 id——这是数据配置规则 6 的直接后果。
- **If 菜从已解锁档口移到未解锁档口**：菜不再可玩（可用性跟随新档口）；旧 id 留在 `unlocks.dishes` 账本里，不删除。进度分子会因此下降——账本单调，可玩计数不单调；界面按当前数据计算，不缓存旧分母。
- **If 菜从已解锁档口 A 移到同样已解锁的 B**：仍然可玩，无需任何操作。
- **If 已解锁档口新增菜**：立即可用（按档口推导），规范化把新菜并入账本；不要求一次写盘才生效。
- **If 删除生涯**：存档清空 `unlocks`；本系统读到空集合，免费档口立即可用；下一次载入回填。没有残留缓存、不退还任何已花的钱。

**边界与并发**

- **If 所有档口都已解锁**：每次 `unlock` 返回 `ALREADY_UNLOCKED`；进度为 `N/N`；菜单可显示「全部档口已解锁」，但这不是通关条件（通关归纪录与里程碑）。
- **If 数据只有一个免费档口**：MVP 正常态；系统没有任何付费路径，`unlock` 永远返回 `ALREADY_UNLOCKED`。
- **If `unlock_cost` 为负**：数据配置校验应已拒绝；运行期若仍遇到（绕过启动校验的夹具），返回 `INVALID_STATION_DATA`，不按 0 或负价成交，也不调用 `spend`。
- **If 账本里的 id 多于当前可玩的菜**（未知菜、已移走的菜、已删除档口的菜）：合法；`unlocks.dishes` 只增不减，存档体积按账本计数，仍远小于 256 KiB 上限。进度只数当前可玩的菜。
- **If 某档口 `dish_pool` 为空**：数据配置校验应已拒绝（数据配置 AC 16）；运行期不特判。
- **If 两个系统同时查询同一档口**：各自拿到独立副本；调用方修改不影响本系统内部状态。
- **If `unlocks.stations` 含重复 id**：存档结构校验拒绝；若运行期遇到，规范化为去重后的升序数组，不报错。

## Dependencies

**上游依赖：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 数据配置 | 硬 | `get_station(id)` → `id`、`display_name`、`unlock_cost`、`dish_pool`；`get_dish(id)` 存在性；前提 `Ready` | GDD 已写（`design/gdd/data-config.md`）；`unlock_requires` 为可选扩展字段，当前不存在 |
| 存档 | 硬 | `unlocks.stations` / `unlocks.dishes`；`commit` / `mark_dirty` / `flush`；`SaveStatus`；`LoadReport.writable` | GDD 已写（`design/gdd/save-system.md`） |
| 局外升级 | 硬 | `spend(amount, reason, extra)`（新增）；`get_wallet()`；`get_level(id)`；`progression_changed` | GDD 已写（`design/gdd/meta-upgrades.md`）；`spend` 待回写 |

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 界面流程 | 硬（UI） | 档口目录与锁定原因、`can_unlock` / `unlock`、解锁进度、档口选择 | GDD 已写（`design/gdd/ui-flow.md`） |
| 餐局流程 | 软（协调） | 界面流程只传可用档口；现有 `station_id` 存在性校验不构成授权校验 | GDD 已写（`design/gdd/run-flow.md`） |
| 菜单波次 | 软（间接） | 读所选档口的 `dish_pool`；本设计下无需过滤 | GDD 已写（`design/gdd/menu-waves.md`） |
| 纪录与里程碑 | 软 | 解锁进度用于展示；按 `station_id` 归档；不发放解锁 | GDD 已写（`design/gdd/records-and-milestones.md`） |
| 首局引导 | 软 | 锁定档口与价格；可高亮、不可自动解锁 | GDD 未写 |
| 音效反馈 | 软 | 无直接调用；解锁音效（若有）由界面流程触发 | GDD 已写（`design/gdd/audio-feedback.md`） |
| 场景与流程管理 | 软（编排） | 解锁只在 `Menu`；本系统不驱动顶层切换 | GDD 已写（`design/gdd/scene-flow.md`） |

**双向一致要求**：以上每个上游必须在其 Dependencies 中列出本系统，并在 Cross-References 中回指；下游必须写明「只读取可用性/进度，不自行判定解锁」。数据配置、存档与局外升级已在下游表列出本系统并标记已写。

**待回改（不在本文件执行，交由编排会话统一处理）**：

1. `systems-index.md` — #13 行改为 `Designed` / `design/gdd/unlocks-and-stations.md`，`Depends On` 补上 **数据配置**（本系统直接读 `get_station` / `get_dish`，现行依赖表漏记）；「Dependency Map → Feature Layer」第 5 条的 depends on 同样补 数据配置；进度计数 `Design docs started` +1、`Vertical Slice systems designed` +1。
2. `design/registry/entities.yaml` — 新增 `station_available`、`effective_unlocks`、`unlock_decision`、`station_unlock_cost`、`runs_to_unlock`、`unlock_progress` 六条公式；不要新增档口解锁价常量（它是 `StationData.unlock_cost` 记录字段）；`opt_value`、`net_profit` 的 `referenced_by` 追加本文件（`runs_to_unlock` 使用它们）；`save_schema_version` 无需变化。
3. `data-config.md` — 下游表补「解锁与档口」行（`get_station(id)` → `unlock_cost`、`dish_pool`；`get_dish(id)` 存在性），状态 `GDD 已写`；建议新增启动校验「至少一个 `unlock_cost = 0` 的档口」；可选新增 `StationData.unlock_requires: Array[StringName]`（默认空，元素必须是存在的升级 id），若不加则本 GDD 规则 11 恒为空集。
4. `meta-upgrades.md` — Core Rule 2 的钱包流水从两条改为三条（净利入账、升级购买、档口解锁）；对外接口表新增 `spend(amount, reason, extra)`；状态表的 `Purchasing` 语义扩为「一笔在途事务」；Open Question 11 标记已回答（接口在本 GDD 定义，MVP 不触发，Vertical Slice 启用）。
5. `save-system.md` — 下游表「解锁与档口」状态改为 `GDD 已写`；补一句「空集合由解锁与档口在载入后回填，标记 dirty，下一次写入时点落盘」（与存档 AC 20 一致）；**修正** Edge Case「同一帧内先结算、后解锁」与 AC 13：合并为同一事务只适用于**结算本身产生解锁变化**的情形；MVP/VS 中解锁只在菜单购买、且收据在结算 commit 之后才显示，因此两笔事务串行，不合并。
6. `run-flow.md` — 建议（非强制）在 `RunConfig` 解析时增加一道 `is_station_available(station_id)` 兜底：不可用则 `init_failed`。当前契约把授权责任放在界面流程；这道校验是纵深防御，避免绕过进度直接开局。
7. `records-and-milestones.md` — Open Question 4 标记已回答（MVP/VS 完成条件不发放解锁）；若未来发放，必须并入结算事务。
8. `menu-waves.md` — 无需改动；若未来启用逐菜解锁，其输入必须从 `dish_pool` 改为 `get_available_dishes(station_id)`。
9. UX 备注（写入 `systems-index.md` 的 UX note 区）— 解锁与档口对界面流程有 UI 需求（档口选择页的锁定态、价格与差额、解锁确认、`COMMIT_FAILED` 提示、进度计数）；`/ux-design` 需覆盖，story 引用对应 UX 规格。

## Tuning Knobs

| 旋钮 | 位置 | 当前值 | 安全范围 | 影响 |
|---|---|---|---|---|
| `station_canteen.unlock_cost` | `StationData.unlock_cost` | 0 | 必须为 0 | 免费起始档口；改成 > 0 会让空集合变成 `NoPlayableStation` |
| `station_banquet.unlock_cost`（Vertical Slice，暂名） | `StationData.unlock_cost` | 120（暂定） | 76–152（4–8 × 19） | 第二个档口的到达时间；低于 4 局显廉价，高于 8 局首次会话看不到 |
| `runs_to_unlock` 目标带 | 设计期规则 | 4–8 | 3–10 | 解锁价与前一档口最优净利的比值；超出时必须同时检查升级树总价（209）与档口总价的比例 |
| 免费档口数量 | 数据设计 | 1 | 1–2 | 多于 2 会让「按档口解锁」的进度感变弱；0 会让空档不可玩 |
| `unlock_requires` | `StationData`（可选字段） | 空 | 0–2 项 | 启用后为档口加等级门槛；不是消耗，只做前置校验 |

**交互**：`unlock_cost` 与升级价格曲线（`upgrade_price`，总价 209）共享同一个钱包。调高解锁价会推迟升级购买，反之亦然；两者之和必须落在「首次会话能摸到一项成长」的预算内。`runs_to_unlock` 的比值以**前一个**档口的最优净利为分母，不以前一个档口的入场费为分母。

## Visual/Audio Requirements

- **锁定档口可见，内容不可见**：列表显示名称、解锁价与锁形标记；不显示菜品卡、价格或菜名——好奇心来自「不知道里面有什么」，不是「知道但进不去」。
- **解锁是一次状态切换，不是一次表演**：成交后锁形标记消失、档口变为可进入；不做飞入、翻牌、粒子或庆祝弹窗。
- **可负担性不靠颜色单独表达**：买不起时同时用文字（「还差 30 元」）与图标表达，符合无障碍要求。
- **音效不归本系统所有**：本系统不播放任何声音；若 Vertical Slice 需要解锁音效，由音效反馈定义事件、界面流程触发。
- **无资产需求**：锁定态与解锁态都是界面流程的既有控件状态，不需要独立美术资源。

> `art-director` not consulted — Lean mode（本系统是规则与进度层，视觉呈现归界面流程；概念文档的「数字优先 / 价目板」锚点已覆盖）。进入生产前需人工复核。

## Game Feel

- **解锁是「把门推开」，不是「抽到奖」**：价格固定、条件透明、没有随机、没有折扣倒计时。玩家在买之前就能算出还要几局。
- **买不起时目标仍然可见**：锁定档口和它的价格一直在列表里，制造「下一局就够了」的短目标；不隐藏、不灰到看不见。
- **解锁后立刻能进**：按钮从锁定变为可进入，不发生页面刷新、不要求重启、不延迟到下一局。
- **不制造 FOMO**：没有限时、没有稀有度、没有「仅此一次」。这是本作对 Achievers 的承诺：进度只由净利与选择决定。
- **失败的解锁必须说清楚**：`COMMIT_FAILED` 显示「未保存，解锁未生效」，`INSUFFICIENT_FUNDS` 显示差额，`NOT_WRITABLE` 说明原因——玩家永远知道为什么没成。

## UI Requirements

- **档口选择区（菜单）**：每个档口一行，显示名称、入场费、菜数与状态；可用档口有「进入」，锁定档口显示解锁价与「解锁」按钮。
- **解锁确认框**：显示价格、解锁后包含的菜数、扣款后钱包余额；确认与取消。价格来自 `unlock_cost`，余额来自局外升级。
- **锁定原因文案**：`INSUFFICIENT_FUNDS` → 「还差 N 元」；`NOT_WRITABLE` → 「本次会话无法保存，解锁已禁用」；`PREREQUISITE_LOCKED` → 缺哪项升级；`ALREADY_UNLOCKED` → 不显示按钮；`COMMIT_FAILED` → 「未保存，解锁未生效，请重试」。
- **进度计数**：`已解锁 1/2 档口 · 12/24 道菜`；数据变化后立即刷新，不缓存旧分母。
- **不可玩态**：`NoPlayableStation` 时「开始」禁用，并显示「没有可用档口——请检查数据」，而不是空白列表。
- **无障碍**：对比度 ≥ 4.5:1；锁定状态必须有文字说明，不能只靠锁形图标；字号 150% 时价格与按钮不换行错位。

> **📌 UX Flag — 解锁与档口**：本系统有 UI 需求。Phase 4 写 epic 之前，先用 `/ux-design` 为档口选择区与解锁确认框出 UX 规格；相关 story 引用 `design/ux/` 下的对应规格，不直接引用本 GDD。

## Cross-References

| 文档 | 关系 | 引用内容 |
|---|---|---|
| `design/gdd/data-config.md` | 上游 | `StationData.unlock_cost` / `dish_pool` / `display_name`；`get_station(id)`；`get_dish(id)`；规则 6（id 不可改名） |
| `design/gdd/save-system.md` | 上游 | `unlocks` 命名空间；规则 14（未知 id 保留）、17（先落盘再确认）；AC 20（存档不添加免费档口） |
| `design/gdd/meta-upgrades.md` | 上游 | 钱包与等级；规则 2（钱包流水必须由本系统开接口）、7（购买事务）；Open Question 11 |
| `design/gdd/run-flow.md` | 下游（协调） | `RunConfig.station_id` 由界面流程提供；存在性校验不等于授权校验 |
| `design/gdd/menu-waves.md` | 下游（间接） | 消费所选档口的 `dish_pool`；逐菜解锁启用时需改读可用菜列表 |
| `design/gdd/records-and-milestones.md` | 下游 | 解锁进度展示；Open Question 4（完成条件不发放解锁） |
| `design/gdd/audio-feedback.md` | 下游（间接） | 本系统无直接音效调用 |
| `design/gdd/scene-flow.md` | 时序 | 解锁只在 `Menu` 生效 |

## Acceptance Criteria

**通用前提**：除用例中另有说明，初始状态均为：数据配置 `Ready`、存档载入完成且可写、顶层状态为 `Menu`、没有在途事务、钱包为用例给定值。

**夹具**：测试夹具放在 `tests/fixtures/` 下，每条用例使用独立夹具；档口与菜的 id 使用稳定假名。`spend` 与 `commit` 可用 spy/double 记录调用次数与载荷，「写入 N 次」指 commit spy 的调用次数；需要暂停一笔在途事务时，使用可手动放行的 commit double。「持久化集合」指存档 `unlocks` 命名空间的当前值（可直接读取）。

**载入与回填**

1. **GIVEN** 解锁集合为空、数据含 `station_canteen`（`unlock_cost = 0`，12 道菜），**WHEN** 载入，**THEN** `get_load_outcome() = Backfilled`，`get_available_stations() = [station_canteen]`，`get_available_dishes("station_canteen")` 返回全部 12 个 id，且不产生错误。
2. **GIVEN** 用例 1 的状态，**WHEN** 在任何写入时点之前查询，**THEN** 可用性与菜列表已完整，而持久化集合仍是 `[]`（证明可用性不等写盘）；**WHEN** `flush()`，**THEN** 持久化 `stations = [station_canteen]`、`dishes` 为 12 个 id，均升序无重复。
3. **GIVEN** 数据中所有档口 `unlock_cost > 0` 且解锁集合为空，**WHEN** 载入，**THEN** `get_load_outcome() = NoPlayableStation`，`get_available_stations()` 为空，`get_station_catalog()` 仍列出全部档口且都标记锁定，commit spy 调用 0 次。
4. **GIVEN** `station_canteen` 免费，**WHEN** `is_station_available("station_canteen")`，**THEN** `true`；**WHEN** `is_station_available("station_nope")`，**THEN** `false`（未知 id 不抛错）。
5. **GIVEN** `station_banquet`（`unlock_cost = 120`，12 道菜）未解锁，**WHEN** `get_available_dishes("station_banquet")`，**THEN** 返回空数组；`get_station_catalog()` 中该档口标记为锁定且 `cost = 120`。
6. **GIVEN** 一份 `Loaded` 的旧档，`unlocks` 为空且数据含免费档口，**WHEN** 载入，**THEN** `get_load_outcome() = Backfilled`，免费档口可用；**WHEN** `flush()`，**THEN** 集合被补齐并写盘。

**可用性与账本**

7. **GIVEN** `station_canteen` 已可用，**WHEN** 数据为它新增一道菜并重新载入，**THEN** 新菜立即可用，且出现在 `get_effective_unlocks().dishes` 中。
8. **GIVEN** 一道菜从已解锁档口移到未解锁档口，**WHEN** 载入，**THEN** 该菜不可玩；它的 id 仍保留在 `unlocks.dishes` 中。
9. **GIVEN** `unlocks` 含一个未知档口 id 与一个未知菜 id，**WHEN** 载入并写盘，**THEN** 两者原样保留，不计入 `get_unlock_progress()` 的分子与分母，不产生错误。
10. **GIVEN** 全部档口已解锁，**WHEN** `get_unlock_progress()`，**THEN** `stations = stations_total`、`dishes = dishes_total`；**WHEN** 再次 `unlock` 任一档口，**THEN** `ALREADY_UNLOCKED`，commit spy 调用 0 次。

**解锁事务**

11. **GIVEN** `wallet = 120`、`station_banquet` 未解锁，**WHEN** `unlock("station_banquet")`，**THEN** 返回 `OK`，`get_wallet() = 0`，档口与其 12 道菜可玩；**AND** `spend` spy 被调用 1 次、`reason = STATION_UNLOCK`、`extra.unlocks` 等于规范化后的集合；**AND** commit spy 被调用 1 次，载荷同时含 `progression` 与 `unlocks`，且 `progression.wallet = 120 − 120`。
12. **GIVEN** `wallet = 119`，**WHEN** `unlock("station_banquet")`，**THEN** 返回 `INSUFFICIENT_FUNDS`，钱包与集合逐字段不变，`spend` 与 commit spy 各调用 0 次。
13. **GIVEN** `wallet = -12`、目标 `unlock_cost = 120`，**WHEN** `can_unlock`，**THEN** `INSUFFICIENT_FUNDS`；**WHEN** `is_station_available("station_canteen")`，**THEN** `true`（负资产不影响已有档口）。
14. **GIVEN** 目标是免费档口，**WHEN** `unlock("station_canteen")`，**THEN** `ALREADY_UNLOCKED`，`spend` 与 commit spy 各调用 0 次。
15. **GIVEN** `station_banquet` 已在集合中，**WHEN** 再次 `unlock`，**THEN** `ALREADY_UNLOCKED`，钱包不变，commit spy 调用 0 次。
16. **GIVEN** 档口 id 不存在于数据，**WHEN** `unlock("station_nope")`，**THEN** `UNKNOWN_STATION`，`spend` 与 commit spy 各调用 0 次。
17. **GIVEN** 存档 `Blocked`（`writable = false`）、Vertical Slice 夹具（2 档口、24 道菜、只解锁免费档口），**WHEN** 查询状态，**THEN** `get_station_catalog()` 返回 2 条、`get_unlock_progress() = 1/2 与 12/24`；**WHEN** `unlock("station_banquet")`，**THEN** `NOT_WRITABLE`，集合不变，commit spy 调用 0 次。
18. **GIVEN** 数据配置未 `Ready`，**WHEN** `get_station_catalog()`，**THEN** 返回空数组；**WHEN** `unlock`，**THEN** `DATA_NOT_READY`。
19. **GIVEN** 当前在 `Run` 状态，**WHEN** `unlock("station_banquet")`，**THEN** `NOT_IN_MENU`，`spend` 与 commit spy 各调用 0 次。
20. **GIVEN** 第一笔解锁仍在等待一个被手动放行的 commit double，**WHEN** 发起第二笔，**THEN** `BUSY`；**WHEN** 放行第一笔，**THEN** 第一笔返回 `OK`，第二笔未扣款。
21. **GIVEN** 注入一次 `commit` 写盘失败，**WHEN** `unlock("station_banquet")`，**THEN** `COMMIT_FAILED`，`get_wallet()` 恢复为 120、集合逐字段不变；**WHEN** 重试且写盘成功，**THEN** 成交且钱包只被扣一次（`get_wallet() = 0`）。
22. **GIVEN** 一次成功解锁，**WHEN** 用磁盘上的存档重新载入（等价于崩溃后重启），**THEN** 档口可玩、钱包为扣款后的值——证明写盘发生在确认之前。

**校验顺序**

23. **GIVEN** 数据未 `Ready`、存档 `Blocked`、档口未知、钱包不足同时成立，**WHEN** `unlock`，**THEN** `DATA_NOT_READY`。
24. **GIVEN** 存档 `Blocked`、档口未知、钱包不足同时成立，**WHEN** `unlock`，**THEN** `NOT_WRITABLE`。
25. **GIVEN** 档口未知且钱包不足，**WHEN** `unlock`，**THEN** `UNKNOWN_STATION`。
26. **GIVEN** 上一笔在途、目标档口免费，**WHEN** `unlock`，**THEN** `BUSY`（`BUSY` 先于 `ALREADY_UNLOCKED`）。

**数据变化与单调性**

27. **GIVEN** 已解锁档口被从数据删除，**WHEN** 载入，**THEN** 它不出现在目录与可用列表，不报错，进度分母也不含它；**WHEN** 下一次写盘，**THEN** 它的 id 仍原样保留在 `unlocks.stations` 中。
28. **GIVEN** 免费档口的回填已经写盘，**WHEN** 其 `unlock_cost` 被改为 200 后重新载入，**THEN** 该档口仍然可用，id 仍在集合中。
29. **GIVEN** 未解锁档口的 `unlock_cost` 从 200 改为 100，**WHEN** `can_unlock`，**THEN** `cost = 100`；**GIVEN** 已解锁玩家遇到同一次改价，**WHEN** 再次 `unlock`，**THEN** `ALREADY_UNLOCKED` 且 `get_wallet()` 不变（无退款）。
30. **GIVEN** 档口 id 不变、`display_name` 改变，**WHEN** 载入，**THEN** 可用性与进度不变，目录显示新名称。
31. **GIVEN** 玩家已解锁两个档口，**WHEN** 删除生涯，**THEN** 可用档口只剩免费档口，进度回到免费档口的计数，下一次载入产生 `Backfilled`。

**进度、成本与确定性**

32. **GIVEN** MVP 夹具（1 个免费档口、12 道菜），**WHEN** `get_unlock_progress()`，**THEN** `1/1` 与 `12/12`；**GIVEN** Vertical Slice 夹具（2 档口、24 道菜、只解锁免费档口），**THEN** `1/2` 与 `12/24`。
33. **GIVEN** `station_banquet.unlock_cost = 120`，**WHEN** 在钱包为 0、500 或 120、已解锁档口数为 0 或 2 时分别查询 `station_unlock_cost`，**THEN** 每次都返回 120——价格不随任何状态变化。
34. **GIVEN** 前一个档口的最优净利为 19、目标解锁价 120，**WHEN** 计算 `runs_to_unlock`，**THEN** 返回 7；**GIVEN** 前一个档口的最优净利为 −5（校准错误夹具），**THEN** 返回 `null`，不返回一个由 `max(1, …)` 伪装出的局数。
35. **GIVEN** Vertical Slice 夹具与暂定解锁价 120，**WHEN** 计算 `runs_to_unlock`，**THEN** 结果落在设计带 `[4, 8]` 内。
36. **GIVEN** 同一份数据与解锁集合，**WHEN** 连续两次调用 `get_effective_unlocks()`，**THEN** 返回内容相同、均为升序、且两次返回是不同副本（修改其一不影响内部状态）。

**边界与跨系统**

37. **GIVEN** 一个绕过启动校验的夹具（`unlock_cost = -1`），**WHEN** `unlock` 或 `can_unlock`，**THEN** `INVALID_STATION_DATA`，`spend` 与 commit spy 各调用 0 次，不按 0 元或负价成交（启动期拒绝由数据配置 AC 15 覆盖）。
38. **GIVEN** 一次结算成功应用（`RunResult` 已入账），**WHEN** 检查 `unlocks`，**THEN** 逐字段不变，`unlock` 的 commit spy 调用 0 次——解锁不参与结算事务；**GIVEN** 同一帧随后发起解锁，**THEN** 两笔事务由存档串行处理（集成用例，依赖结算与存档 double；正常 UI 下收据在结算 commit 之后才显示，两者不会重叠）。
39. **GIVEN** 夹具中的档口声明 `unlock_requires = [upgrade_capacity]` 且该升级等级为 0，**WHEN** `can_unlock`，**THEN** `PREREQUISITE_LOCKED`；**GIVEN** 等级为 1，**THEN** `can_unlock` 返回 `ok`，`unlock` 完成扣款。该用例在 `unlock_requires` 字段落地后启用。
40. **GIVEN** 一份合法但不含任何档口的数据（`S = ∅`、`D = ∅`），**WHEN** 载入，**THEN** `get_load_outcome() = NoPlayableStation`，`get_station_catalog()` 为空，`get_unlock_progress()` 四个计数全为 0，且不发生除零。
41. **GIVEN** 一个已解锁的付费档口，**WHEN** 数据给它的菜池新增一道菜并重新载入，**THEN** `get_load_outcome() = Backfilled`（persisted ≠ effective），`unlocks_changed` 发一次；**WHEN** `flush()`，**THEN** 账本包含新菜 id。免费档口不是触发 `Backfilled` 的必要条件。

**无法直接断言、已改写或转为开放问题的规则**：

- UI 侧断言（菜单禁用「开始」、隐藏购买按钮、差额文案、换行与对比度）留给 `/ux-design` 与界面流程的集成测试；本文件的 AC 3、17 只断言数据侧可观测值。
- 「完成条件不发放解锁」（规则 15）：MVP/VS 没有发放路径，只能以「不存在调用」为负向断言；结论记录在 Open Question 3。
- 「进程被杀后重启」（规则 6）：以「从磁盘重新载入」等价替代，见 AC 22。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | Vertical Slice 第二个档口的 `unlock_cost` 定为 120（约 7 局）是否合适？需要与它的 `entry_fee`、菜池大小和升级树总价 209 一起校准。 | economy-designer + 用户 | 第二个档口设计时 |
| 2 | 是否需要**逐菜解锁**（同一档口里分批开放菜）？当前设计按档口整体解锁，`unlocks.dishes` 只是账本。若要逐菜，需要 `DishData` 增加解锁条件字段，并改菜单波次的输入。 | game-designer | Vertical Slice 内容定稿前 |
| 3 | 完成条件（吃遍所有档口）是否应发放解锁或奖励？本设计回答「MVP/VS 不发放」（纪录 OQ4）；若产品需要实质后果，需定义并入结算事务的授予接口。 | game-designer | 通关条件设计时 |
| 4 | `StationData.unlock_requires` 是否进入数据配置？不进入则等级门槛永远不生效；进入则需要 schema 校验与回写。 | 数据配置 + technical-director | 架构阶段 |
| 5 | 锁定档口是否从第一局就可见？可见服务好奇心，也可能在只有一档口时制造噪音。 | ux-designer + 首局引导 | `/ux-design` 时 |
| 6 | 解锁入口放在档口选择页还是升级页？两者都动钱包，需要统一的「花钱」交互语言。 | 界面流程 | 界面流程 GDD 编写时 |
| 7 | 是否需要退款、重置或「降级回免费档口」？当前一律没有，解锁单调。 | economy-designer | 试玩后 |
| 8 | 免费档口需要几个？多于一个会稀释「按档口解锁」的进度感，少于一个会让空档不可玩。 | game-designer | 第二个档口设计时 |
| 9 | 档口 id 改名的迁移策略：当前按「改名 = 删除 + 新增」处理，玩家会失去解锁。是否需要一次性别名表？ | 数据配置 + 存档 | 发布前 |
| 10 | 在线挑战是否需要限定可用档口？若要，挑战配置必须读取本系统的可用集合。 | live-ops-designer | Full Vision 前 |
