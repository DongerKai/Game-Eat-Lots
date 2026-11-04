# 菜单波次（menu-waves.md）— 设计评审日志

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 5 | Recommended: 14
Summary: 波次划分、可见度、推进代价与"跳过即失去"的核心设计是自洽的，五个公式与注册表数值一致，`consume` / `initialize(... pool_override)` 两个接口已按依赖背景同步。五个 blocking 全部集中在**接口与跨文档契约**：最后一波「结束取餐」没有调用入口、`consume` 未定义"吃下后本波还有菜"这一常态返回值、生成期健康检查所需的容量/抗性没有来路且与胃容量"运行期不读"冲突、跳波确认与本 GDD 相反地被 HUD 禁止、菜品数值的 ±10% 重掷契约未落地。修法多为接口行与一条规则的补齐，不是机制返工。
Prior verdict resolved: First review

### 复审背景

本分支没有先前的评审日志，`review-receipts.sh check` 返回 `RECEIPT: NONE`。按全文评审执行。
`design/gdd/menu-waves.md` 共 339 行 / 29,637 字节，哈希 `a9ed433b…`。本次是首轮评审，不是增量复审。
评审对象最近一次改动是 `ccdd0d7`（Batch 2 修复），即依赖背景中所说的 `consume` / `pool_override` 同步。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。

Formulas 对本系统是**必需**的：它定义波次划分、可见/预览、推进代价、可达最优与健康闸门五条数值规则，是可平衡的设计，不是纯流程散文。

Advisory 缺口（不阻塞）：
- **Player Fantasy** — Game Feel 一节（`:250-256`）实质覆盖了它。
- **Tuning Knobs** — 缺席，但本系统确实有可调值（`wave_size`、0.9 健康阈值、20 次重掷、`preview` 上限、跳过代价）。建议补一节，见 Recommended #14。

### 依赖图

上游（文件均存在）：
- ✓ `data-config.md` — 存在（Approved），`StationData.wave_size` 已在 `:53` 定义，本 GDD 的"新增字段请求"已过时
- ✓ `dish-values.md` — 存在（Approved），`opt_value = 104`、`entry_fee = 85` 与本 GDD 一致
- ✓ `run-flow.md` — 存在（In Review），`initialize` / `consume` / `advance` / `no_eatable` 契约已对齐
- ✓ `cloying-and-taste.md` — 存在（Designed），`extra_satiety` / `effective_satiety` 与 AC 18 一致
- ✗ **`stomach-capacity.md` 存在，但未被本 GDD 声明为依赖** —— `wave_reachable_opt` 的 `C`（`:163`）来自胃容量，Dependencies（`:212-229`）却没有这一行。胃容量侧已把本系统列为"设计期下游"（`stomach-capacity.md:68,207`），是单向引用。

下游（文件均存在并已回指）：
- ✓ `in-game-hud.md`（`:169,388`）
- ✓ `settlement-and-entry-fee.md`（`:342`，软/未来）
- ✓ `meta-upgrades.md`（`:102,347`）
- ✓ `first-run-onboarding.md`（`:127,370`）
- ✓ `save-system.md`（规则 5，`:31`）

无断链；缺失的是**胃容量**这一条上游声明。

### 注册表值

用 `entities.yaml` 的条目名与数值键回扫全文，命中项全部是公式名本身，无第二个数值来源：

- `wave_partition`（`:538-551`）↔ `:73-93`：表达式 `ceil(|M|/wave_size)`、切片定义、MVP 3×4 —— 一致。
- `preview_dishes`（`:553-565`）↔ `:95-115`：`min(menu_vision, wave_size−1, |W_next|)`、range `[0,7]` —— 一致。
- `wave_reachable_opt`（`:567-582`）↔ `:151-170`：表达式一致。**range 不一致**：注册表 `[0, 99999]`，本 GDD `[0, opt_value(M,C,T)]`（即 `[0,104]`）。见 Recommended #13。
- `wave_healthy`（`:584-594`）↔ `:172-188`：三子句与 0.9 阈值 —— 一致。
- `wave_size_default = 4`（`:1166-1174`）↔ `:87` —— 一致；但 `referenced_by` 只有本文件，漏了实际定义字段的 `data-config.md`。
- `wave_advance_cost = 1`（`:1176-1182`）↔ `:117-137` —— 一致。
- `opt_value = 104`（`dish-values.md`）↔ `:34,170,182,188,316,317` —— 一致。
- `station_canteen_entry_fee = 85`（`:1183-1190`）↔ AC 17 `:321` 的 `0 − 85 = −85` —— 一致。
- `menu_vision_useful_max = ceil((wave_size−1)/effect_value)`（`:988-998`）↔ 预览上限 `wave_size−1` —— 一致。

**值冲突：0。** 一处区间不一致（`wave_reachable_opt`），一处 `referenced_by` 缺项。

---

## Required Before Implementation

### 1. [BLOCKING] 最后一波「结束取餐」没有任何调用入口

**位置**：本 GDD `:30`（规则 5 最后一波例外）、`:49-50`（状态表）、`:62`（交互行）、`:69`（接口所有权）；`run-flow.md:33,35`。

本 GDD 把「结束取餐」定义为一个独立动作：**免费、丢弃剩余菜、结束本局、发出"无可吃菜"信号**，并且明确 `advance()` **只承载付费跳波**（`:62,69`）。状态表（`:50`）也把"最后一波结束取餐"列为一个必须执行的状态转移。

但对外暴露的三个调用里，没有一个能执行它：
- `consume(dish_id)` 只处理"吃一道菜"；
- `advance()` 被限定为付费跳波；
- "无可吃菜信号"是**输出**，不是 run-flow 能主动触发的输入。

`run-flow.md:33` 把最后一波的 `game_skip_wave` 路由为「结束取餐（同一动作，0 回合）」，`:35` 又说"本系统只负责路由输入、确认代价、执行结束判定，不自行计算跳过代价"。于是 run-flow 没有任何函数可以调用到本 GDD 的结束取餐语义。AC 7（`:302`）直接依赖这条路径，无法实现。

**修复**：二选一，并与 `run-flow.md` 的菜单波次行同步：
- 新增 `end_pickup() → ENDED`（或 `advance() → {ADVANCED, ENDED}`，由 `advance_cost` 决定 0/1 回合），并在 `:62` 与 `run-flow.md:120` 写明；
- 或把"结束取餐"并入 `consume` 的最后一个返回值（不推荐，语义会混）。

### 2. [BLOCKING] `consume(dish_id)` 没有定义"吃下后本波仍有菜"这一常态

**位置**：本 GDD `:28`（规则 4）、`:48`（状态表）、`:62`（`consume(dish_id) → {ADVANCED, NO_EATABLE}`）；`run-flow.md:34`。

规则 4 只在**当前波变空**时自动推进；状态表同理。也就是说，吃下本波非最后一道菜时，`consume` 既不 `ADVANCED` 也不 `NO_EATABLE` —— 这是玩家绝大多数吃菜动作的情形。但两处接口都把它写成 `{ADVANCED, NO_EATABLE}` 的二值联合，没有第三个值，也没有说明"两个标志都可能为 false"。

run-flow 的吃菜提交路径（`run-flow.md:34`）每次成功吃菜都会调用它并读取返回值；实现者只能自行臆造一个 `NONE`，或把返回值当成 bitmask —— 两者都会与另一侧的理解漂移。

**修复**：`consume(dish_id) → {ADVANCED, NO_EATABLE, NONE}`（或明确写成两个独立布尔标志，可同时为 false），并补一条 AC：吃下本波第 1 道（本波尚余 3 道）时返回 `NONE` 且 `wave_index` 不变。

### 3. [BLOCKING] 生成期健康检查的 `C` / 腻度抗性没有来路，且与"仅设计期"声明冲突

**位置**：本 GDD `:155-166`（`wave_reachable_opt` 的 `C`、`satiety_eff`）、`:176`（`wave_healthy(M, seed)`）、`:182`（`opt_value(M, C0, T)`）、`:62`（`initialize` 签名）、`:212-229`（Dependencies）；`stomach-capacity.md:68,207`；`dish-values.md:37`；`cloying-and-taste.md:60,76`。

`wave_reachable_opt` 的定义需要三个运行期数值：
- `C`（一局容量，来源写"胃容量系统"，`:163`）；
- `satiety_eff = satiety + max(0, min(group_cloy, cloy_cap) − floor(cloying_resistance))`（`cloying-and-taste.md:76`），需要 `cloying_resistance`；
- `T`（回合预算）。

但 `initialize(station_id, seed, turn_budget, menu_vision[, pool_override])`（`:62`）**只给了 `T`**：既没有容量 `C`，也没有 `cloying_resistance`。同时：
- 胃容量没有出现在本 GDD 的 Dependencies 里（`:212-229`）；
- 胃容量明确把 `C` 定为"**设计期下游**……运行期不读"（`stomach-capacity.md:68,207`）；
- 菜品数值明确 `opt_value` "**只用于设计期与 CI**……不进入游戏运行时"（`dish-values.md:37`）。

而本 GDD 的健康检查是"生成时校验 + 最多 20 次重掷、不阻断开局"（规则 9），读起来是**运行期**行为。于是两条路都走不通：运行期拿不到 `C`/抗性；设计期又与"重掷才不阻断开局"的运行时承诺矛盾。AC 15/16（`:316-317`）按当前写法无法真正跑起来。

**修复（建议一次性说明清楚）**：
- 明确健康检查使用 **`C0`（基础容量 20）+ `StationData.turn_budget` + `cloying_resistance = 0`** 的静态基准 —— 升级只会提高容量、不会降低，因此按基础值健康的菜单对升级后的玩家仍然健康；
- 选择执行时机：要么在 CI / 设计期预计算健康划分并随档口落盘（与 `dish-values.md:37` 一致），要么给 `initialize` 增加 `capacity` 与快照参数并在 Dependencies 补「胃容量」硬依赖；
- 把 `wave_healthy` 的签名改成含 `C, T`（见 Recommended #2）。

### 4. [BLOCKING] 跳波确认：本 GDD 要求，局内 HUD 明确禁止

**位置**：本 GDD `:243`（Visual）、`:252`（Game Feel）、`:261`（UI Requirements）、AC 20（`:324`）；`in-game-hud.md:59`（规则 16）、`:372`（Edge Case）、`:502`（AC 24）；`systems-index.md:85-86`。

本 GDD 对**非最后一波**的 `下一波` 要求确认框逐条列出将失去的菜：
- `:243`「点"下一波"时，确认框逐条列出**本局将失去的菜**（名称 + 价格 + 饱腹）」；
- `:252`「没有确认的跳过会变成误操作」；
- AC 20 用确认弹窗作为验收条件。

HUD 的规则 16 与 AC 24 则明确相反：
- `in-game-hud.md:59`「非最后一波……**激活即提交，不再二次确认**（代价已写在按钮上）」；
- `:372`「非最后一波点「跳过本波」：**立即提交，不弹确认**」；
- AC 24「**不弹二次确认**」。

两条 AC 不可能同时通过。（`systems-index.md:85-86` 还写着"跳过/结束取餐的确认框"，站在本 GDD 一侧。）

**修复**：二选一并同步另一份——若保留本 GDD 的确认，HUD 的规则 16 / Edge / AC 24 必须改；若采纳 HUD 的"立即提交"，本 GDD 的 `:243`、`:252`、`:261` 与 AC 20 必须改，且 Open Question 8（`:339`）应改为"确认是否**可以开启**"而不是"是否可以关闭"。

### 5. [BLOCKING] 菜品数值的 ±10% 单次刷新重掷契约没有落地，且健康阈值基线不明

**位置**：本 GDD 规则 9（`:34`）、`wave_healthy`（`:176`）、Cross-References（`:275`）；`dish-values.md:241,255,372`。

`dish-values.md` 把一条跨系统契约明确交给本系统：
- `:241`「菜单波次生成单次菜单时，其 `opt_value` 相对全池校准值的偏差不得超过 ±10%，否则该次刷新重掷」；
- `:255`「若单次刷新菜单的 `opt_value` 偏离全池校准值超过 ±10%：**「菜单波次」应重掷**」；
- `:372` 再次声明该重掷"属于菜单波次契约"。

本 GDD 的 Cross-References（`:275`）也引用了"±10% 校准规则"，但规则 9 与 `wave_healthy` 只实现了一个**单侧下界**：`wave_reachable_opt ≥ 0.9 × opt_value(M, C, T)`。差异有二：
1. 没有上界（≤110%），也没有"相对**全池**校准值"的基准；
2. `opt_value(M, C, T)` 用的是**当前池** `M`。当 `pool_override` 生效（首局 6 道菜，`opt_value = 94`）时，阈值变成 `ceil(0.9×94)=85`，而不是正文写的 94（"全池 104 → 阈值 94"，`:34`）。同一份 GDD 里"全池"与 `M` 指向不同集合。

**修复**：二选一并写死：
- 采纳 ±10%：健康检查改为相对**该档口全池**的 `opt_value` 做双侧带 `[0.9, 1.1] × opt_value(station_pool, C0, T)`，并补一条 AC；`pool_override` 时说明按"覆盖池相对全池"还是"覆盖池自身"计算。
- 或声明规则 9 **取代**菜品数值的 ±10% 条款，并请编排会话在 `dish-values.md` 删除/改写该句。

---

## Recommended Revisions

1. **[HIGH] `menu_vision` 被称作"整数等级"，与上游定义冲突。** `:64` 与 `:227` 写"本系统只消费整数等级"；但 `meta-upgrades.md` 规则 12（`:49-51`）把快照的 `menu_vision` 定义为**已解析额度** `Σ effect_value × level`，"不是等级"，`run-flow.md:70` 也复述"作为总预览额度，不是等级"。本 GDD `:110` 与公式本身是对的，只有这两处散文是错的；一个按 `:64` 实现的程序员会把额度再乘一次 `effect_value`。改为"已解析的总预览额度（int），不是等级"。

2. **`wave_healthy` 的签名漏了公式体用到的 `C`、`T`。** `:176` 写 `wave_healthy(M, seed)`，但右侧用 `opt_value(M, C, T)`；`:182` 又写成 `opt_value(M, C0, T)`。把签名改为 `wave_healthy(M, seed, C, T)`，并统一 `C0` / `C` 的用法（健康检查用 `C0` 时明说）。

3. **`wave_reachable_opt` 的签名含 `menu_vision`，公式体却不使用它。** `:155` 有 `menu_vision` 参数，`:157` 的合法序列定义里没有它；注册表 `:567-582` 也照抄了。信息量不影响"神谕最优"，删掉该参数（或说明它只影响"玩家能否找到"而不影响可达上限）。同理，`:166` 的 Output Range 用 `opt_value(M, C, T)` 需要 `C`。

4. **`wave_healthy` 的前两个子句是恒真的。** `wave_partition` 用"洗牌 + 顺序切片"构造，只要 `|M| ≥ 1`、`wave_size ≥ 1`，就必然"每波非空"且"划分恰好覆盖菜池一次"（`:26,79`）。`:186` 已经点出"全池最优是死代码"，但没说这两项也是构造性恒真。建议注明：实际只有可达最优子句能失败，前两项是防回归断言。

5. **AC 1 的"任一 `station_id`"过宽。** `:290` 断言"3 个波次、每波恰好 4 道菜、总数 12"，但同一 GDD 的 Edge Cases（`:192-193`）明确允许非整除与小于 `wave_size` 的菜池；`station_banquet` 之后若改成非 12 道菜，这条 AC 必然失败。改为"`station_canteen`（12 道菜）"或在 AC 里带上 `|M| = 12`。

6. **"待回改"两处已经过时。** `:274`、`:283` 仍把 `StationData.wave_size` 写成"新增字段请求，需回改"，但 `data-config.md:53` 已定义该字段（Approved）。`:231` 说"菜品数值的下游表中本系统仍是「GDD 未写」"，但 `dish-values.md:277` 已写"GDD 已写（`design/gdd/menu-waves.md`）"。`dish-values` 的 OQ5（`:382`）已由本 GDD 规则 2 回答（不重复抽取），可一并标记。

7. **`wave_size` 的区间自相矛盾。** Formulas `:87` 写 `1–8`；Dependencies `:216` 写 `int ≥ 1`；`data-config.md:53` 写 `≥ 1`，其 schema 表 `:116` 写 `1–8`。三处统一成 `1–8`（或 `≥1`），并说明 `wave_size` 在池小于它时被钳制（`:193`）后，`preview` 上限是否也随钳制值变化。

8. **规则 6 "代价不是回合"读起来与规则 5/8 冲突。** `:31` 写"代价不是回合，是那几道菜本身"，但规则 5（`:29`）与规则 8（`:33`）都明说跳波消耗 1 回合。改为"主代价是失去的菜，另付 1 回合跨波"。

9. **"跳过对上限的影响"表的措辞会误导。** `:139-149` 写"每主动跳过一波，就少一个吃菜的回合"，但"跳过一波"会丢弃该波**剩余**菜（`:29`、AC 6）。1 次跳过 = 9 道菜的结果，前提是"先吃掉该波 3 道、再跳过第 4 道"。把这一前提写进表注，否则读者会以为跳过整波只丢 1 道菜。

10. **重掷时"下一个种子"未定义，且存档应记哪个种子。** 规则 9（`:34`）说"用下一个种子重掷"，是 `seed+1`、`seed+k` 还是 RNG 下一输出？另外若健康检查触发了重掷，Edge Case（`:207`）要求存 `seed` —— 存**输入**种子还是**实际采用**的种子？两者都应在规则里写死，否则读档重放可能得到不同划分。

11. **生成期穷举的耗时没有预算。** `:168` 说每波 65 条有序子序列、三波约 2.7×10⁵ 条，生成期可穷举；但规则 9 允许最多 20 次重掷，最坏约 5.4×10⁶ 条序列，发生在开局帧。建议给一个生成耗时上限（例如 ≤50 ms），或说明健康划分在 CI 预计算、运行期只洗牌不穷举。

12. **存档交互行应标"未来/未启用"。** `:67` 的"条件"与 Edge Case `:207` 描述中途存档要保存 `seed`/`wave_index`/本波剩余；`save-system.md:31` 规则 5 明确 MVP 不做中途存档、这些契约"在 MVP 均不生效"。建议在 `:67` 直接写"未来（MVP 未启用）"，避免被当成 MVP 范围。

13. **注册表卫生（跨文件，非本文件可改）。** `wave_reachable_opt` 的 `output_range` 是 `[0, 99999]`（`entities.yaml:578`），本 GDD 是 `[0, opt_value]`；`wave_size_default`（`:1166`）的 `referenced_by` 缺 `data-config.md`，尽管该字段由 data-config 定义。建议编排会话统一。

14. **补一节 Tuning Knobs（standard 层 advisory）。** 本系统确有可调值：`wave_size`（1–8，暂定 4）、健康阈值 0.9、重掷上限 20、`preview` 上限 `wave_size−1`、跳过代价 1 回合。它们直接决定单局上限（95–104），应集中列出安全范围与影响面。Game Feel（`:250-256`）已覆盖 Player Fantasy，无需另写。

## Nice-to-Have

- `advance_cost` 的"空波自动推进"子句与"最后一波"子句在最后一波吃空时重叠（`:123-124`），逻辑上无害，可注明优先级。
- 跨文档陈旧：`first-run-onboarding.md:395` 与 OQ2（`:502`）仍说"`menu-waves.md` 的 `initialize` 未接受 `pool_override`"，但本 GDD `:62` 现在已接受，可请编排会话标记完成。
- 跨文档陈旧：`cloying-and-taste.md:332` 的 OQ1 仍列为未决，本 GDD 规则 3/5（`:27,29`）已回答。
- `systems-index.md:210` 把本系统规模标为 **S**；本评审信号为 **L**（5 个公式、4 条上游 + 5 条下游、3 个 ADR 级 Open Question：随机 vs 编排、跳过代价、多档口供给）。
- AC 18（`:322`）与 `cloying-and-taste.md:92` 的示例逐字一致（红烧肉 → 糖醋里脊 = 4 + 1 = 5 格），无需改。
- AC 17（`:321`）的 `0 − 85 = −85` 与 `settlement-and-entry-fee.md:148`、注册表 `:1183-1190` 一致。

## 跨文档信号（不在本文件的修改范围）

- **胃容量未列入 Dependencies**（见 Blocking #3），胃容量侧已单向回指。
- **跳波确认冲突**（Blocking #4）：`in-game-hud.md:59,372,502` vs 本 GDD。
- **±10% 契约**（Blocking #5）：`dish-values.md:241,255,372` vs 本 GDD 规则 9。
- **`menu_vision` 等级/额度**（Recommended #1）：本 GDD `:64,227` vs `meta-upgrades.md` 规则 12、`run-flow.md:70`。
- 索引规模 S vs 评审 L（`:210`）。

## 注册表复核清单

| 条目 | 注册表 | 本 GDD | 结果 |
|---|---|---|---|
| `wave_partition` | `:538-551` | `:73-93` | ✓ |
| `preview_dishes` | `:553-565` `[0,7]` | `:103,110` `[0,wave_size−1]` | ✓ |
| `wave_reachable_opt` | `:567-582` `[0,99999]` | `:155-166` `[0,opt_value]` | 区间不一致 |
| `wave_healthy` | `:584-594` | `:176` | ✓ |
| `wave_size_default` | `:1166` = 4 | `:87` = 4 | ✓（`referenced_by` 缺 data-config） |
| `wave_advance_cost` | `:1176` = 1 | `:125` = 1 | ✓ |
| `opt_value` | `dish-values.md` = 104 | `:34,170,188` | ✓ |
| `station_canteen_entry_fee` | `:1183` = 85 | AC 17 `−85` | ✓ |
| `menu_vision_useful_max` | `:988` | 预览上限 `wave_size−1` | ✓ |

## Scope Signal

**Rough scope signal: L**（producer 应在 sprint planning 前复核）

- 5 个公式；4 条上游（数据配置、菜品数值、餐局流程、腻度，另需胃容量的 `C`）；5 条下游（HUD、结算、局外升级、首局引导、存档）。
- 3 个 ADR 级 Open Question：随机洗牌 vs 策划编排（OQ6）、跳过的代价模型（OQ2）、多档口供给稀缺（OQ5）。
- 4 条跨文档硬契约（`consume` / `advance` / `pool_override` / ±10%），其中两条未闭合。

## Verdict: NEEDS REVISION

五个 blocking 全部是接口与跨文档契约缺口，不是机制返工。补齐接口行、写死健康检查输入与基准、与 HUD / 菜品数值对齐后即可进入复审。

---

Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 94da89a242967c44827406c4da5714d088df6394
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
