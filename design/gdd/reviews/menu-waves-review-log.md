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

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 3 | Recommended: 14
Summary: 本轮修复方向正确：`end_pickup() → ENDED`、`consume` 的 `NONE`、跳过不弹确认、双侧带与 `pool_override` 基准都已落进正文并有 AC 覆盖，3 条原 blocking 完全解决。残留 2 条：规则 9 的"设计/CI 期预计算 + 划分落盘"与仍保留的"运行期按种子划分 / 生成 20 次不阻断开局"互斥；"不弹二次确认"从跳过扩到了最后一波「结束取餐」，与本 GDD 的 AC 7 和 HUD 的 AC 25 相反。另有一处注册表公式未随双侧带同步（跨文件）。
Prior verdict resolved: No —— 5 条原 blocking 中 3 条完全解决、2 条残留（详见下表）。

### 复审背景

上一轮评审（同日，本日志首条）对象为 `a9ed433b…`（339 行）；本轮为 `ce532d32…`（341 行），改动集中在 `427409f`（Batch 2/3 修复，仅 `menu-waves.md` 的规则 5/9、交互行、`wave_healthy`、Visual/Game Feel/UI 三节与 AC 20–22）。`review-receipts.sh check` 显示目标文件 `CHANGED`，按全文复审执行，不是增量补丁式复查。

### 上一轮 blocking 逐条复核

| # | 原 finding | 状态 | 依据 |
|---|---|---|---|
| 1 | 最后一波「结束取餐」没有调用入口 | **已解决** | `:30`、`:62`、`:69` 定义 `end_pickup() → ENDED`；`run-flow.md:121` 已同步；AC 7 `:302` 可落地 |
| 2 | `consume` 缺"本波仍有菜"的返回值 | **已解决** | `:62` 改为 `{ADVANCED, NO_EATABLE, NONE}`；AC 21 `:325` 覆盖。仅 `run-flow.md:34,:296` 的散文仍写二值联合（见 Recommended #14） |
| 3 | 健康检查的 `C` / 腻度抗性没有来路 | **部分解决** | 基准已定死为 `C0 = 20` / `T = turn_budget` / `cloying_resistance = 0`（`:34`、`:182`）；但**执行时机**改成了"设计/CI 期预计算 + 划分落盘"，与保留下来的运行期种子划分和"不阻断开局"冲突（见 Blocking #1）。`wave_healthy` 签名仍缺参数（见 Recommended #2） |
| 4 | 跳波确认：本 GDD 要求，HUD 禁止 | **部分解决** | 非最后一波已改为"代价写在按钮上、不弹确认"（`:243,:252,:261`，AC 20 `:324`，OQ8 `:341` 已答）；但同一句把最后一波「结束取餐」也扫进了"不弹二次确认"，与 AC 7 和 HUD AC 25 相反（见 Blocking #2） |
| 5 | ±10% 重掷契约未落地、基线不明 | **已解决（按原要求）** | `:176` 改为 `[0.9, 1.1] × opt_value(pool, C0, T)`；`:182` 明确 `pool_override` 时基准为覆盖池；AC 22 `:326` 覆盖。上界恒真问题见 Recommended #1 |

---

## Required Before Implementation

### 1. [BLOCKING] 健康检查的执行时机仍自相矛盾（原 B3 残留）

**位置**：本 GDD `:34`（规则 9）、`:26`（规则 2）、`:35`（规则 10）、`:47`（状态表）、`:205`（Edge）、`:291-292`（AC 2/3）、`:316-317`（AC 15/16）、`:341`（OQ7）。

规则 9 现在写「**（设计/CI 期预计算）**……健康划分随数据落盘，运行期直接使用，不重算 `opt_value`」——即划分在构建期固定。但同一份文档仍要求划分由**本局种子**在开局生成：
- 规则 2 `:26`「用本局种子对 `dish_pool` 做确定性洗牌」；
- 规则 10 `:35`「同一 `(station_id, seed)` 产生同一划分」；
- 状态表 `:47`「餐局开始 → 用 seed 生成划分」；
- AC 2/3 `:291-292` 用两个不同种子验证划分，并要求基准文件按 `(station_id, seed=12345)` 比对。

同时 Edge `:205` 与 AC 16 `:317` 写「生成 20 次仍无健康划分……**本局照常开始（不阻塞）**」——这是运行期措辞。而运行期种子是随机的、且 MVP 不持久化（OQ7 `:341`、`save-system.md:31`），构建期不可能为任意种子预生成并落盘"健康划分"。

三种读法互斥，实现者必须猜：
- (a) **运行期模型**：按种子划分，健康检查在开局跑，最多重掷 20 次。则删除"健康划分随数据落盘"；`opt_value` 基准可在构建期预计算成表（`dish-values.md:37` 的"不进运行期"由此满足），但 `wave_reachable_opt` 的穷举仍在运行期——上一轮 Recommended #11 的耗时预算（最坏约 5.4×10⁶ 条序列）必须一并给出。
- (b) **构建期模型**：档口随数据交付一份固定划分，运行期直接加载。则规则 2/10、状态表 `:47`、AC 2/3、OQ7 以及 `run-flow.md:390` 的 `seed` 参数全部要改，种子不再参与划分。
- (c) **混合模型**：构建期只预计算基准 `opt_value(pool, C0, T)` 与健康种子的白名单，运行期用预计算表做检查。则要写明白名单如何与任意种子对应（例如 `seed % N` 选表），否则 (a) 与 (b) 的矛盾仍在。

**修复**：三选一写死，并把 `:205`、`:316-317` 的"不阻断开局 / 本局照常开始"改成与所选模型一致的措辞。

### 2. [BLOCKING] 最后一波「结束取餐」的确认去留不一致（原 B4 残留 / 新引入）

**位置**：本 GDD `:261`（UI Requirements）、`:302`（AC 7）；对照 `:243`、`:252`（仅跳过）；`in-game-hud.md:59`（规则 16）、`:371`（Edge）、`:503`（AC 25）。

本轮修复把"不弹二次确认"从「非最后一波跳过」扩写到了整个按钮：

> `:261`「**跳过按钮**：……最后一波变为「结束取餐」（免费）。**点击立即提交，不弹二次确认。**」

但同一份 GDD 的 AC 7（`:302`）仍写「**WHEN** 点击「结束取餐」**并确认**，**THEN** 回合计数仍为 6」；HUD 则明确规定最后一波**要**打开确认面板（`in-game-hud.md:59`「最后一波……激活后打开 HUD 内的确认面板，面板默认焦点在「取消」」、`:371`、AC 25 `:503`）。

上一轮 B4 的形态是"GDD 要确认 / HUD 禁止"；本轮变成"GDD 的 UI 节禁止 / GDD 的 AC 7 与 HUD AC 25 要确认"——同一个 AC 冲突换了个方向，仍然不可能三条同时通过。

**修复**：把 `:261` 末句限定为「非最后一波的跳过按钮点击立即提交」；最后一波「结束取餐」改为「打开 HUD 的确认面板（见 `in-game-hud.md` 规则 16 / AC 25）」，与 AC 7 的"并确认"对齐。`end_pickup()` 本身仍是"确认之后"的调用，接口不变。

### 3. [BLOCKING · 跨文件] 注册表 `wave_healthy` 未随双侧带同步

**位置**：`design/registry/entities.yaml:584-594`（`wave_healthy`）对照本 GDD `:176`；另 `entities.yaml:567-582`。

注册表仍是旧公式：

```
variables: [M, seed]
expression: "(all waves non-empty) and (waves partition M) and (wave_reachable_opt >= ceil(0.9 * opt_value(M, C, T)))"
```

没有 `1.1` 上界，也没有 `pool` / `C0` / `T`。该条目的 `source` 就是 `design/gdd/menu-waves.md`，所以是注册表落后于 GDD，不是 GDD 写错。

**修复（跨文件，不在本 GDD 执行；本分支按并行约束不改 `entities.yaml`）**：把 `wave_healthy` 的 `expression` / `variables` 更新为双侧带 + `pool, C0, T`；顺带把 `wave_reachable_opt` 的 `output_range` 从 `[0, 99999]` 改为 `[0, opt_value]`（上一轮 Recommended #13 未处理）。这条不阻塞 GDD 文本本身，但阻塞 `Approved` 的签核——注册表是用给下游工具读的。

---

## Recommended Revisions

1. **[HIGH] 双侧带的上界按构造不可达，±10% 实际只剩半边。** `:168` 的 Output Range 已写明 `wave_reachable_opt ∈ [0, opt_value(M, C, T)]`，而 `:176` 的基准就是**同一菜池**的 `opt_value(pool, C0, T)`（`C = C0`、`pool = M`）；可达最优永远 ≤ 静态最优，所以 `≤ 1.1 × opt_value` 恒真，`wave_healthy` 只可能在下界失败。代价是：`dish-values.md:241,:255` 的「偏差不得超过 ±10%」只有下界被真正执行。建议二选一——(a) 在 `dish-values.md` 把上界删掉或改成只对"单次刷新菜单 vs 全池"有意义的口径；(b) 在本 GDD 注明上界是防回归断言、永不触发，避免读者以为它在起作用。AC 22 `:326` 本身可测（注入夹具），但只能注入下界失败。

2. **`wave_healthy(M, seed)` 的签名仍缺公式体用到的东西。** `:176` 的签名只有 `M, seed`，体内用到 `pool`（覆盖池或档口全池）、`C0`、`T`。上一轮 Recommended #2 未处理。改为 `wave_healthy(M, seed, pool, C0, T)`，或在签名下注明这三个是"构建期常量，来自 StationData 与 C0"。

3. **`wave_reachable_opt` 的 `menu_vision` 参数不被公式使用。** `:155` 有该参数，`:157` 的合法序列定义里没有它（信息量不影响"神谕最优"）；注册表 `:567-582` 照抄。上一轮 Recommended #3 未处理。删掉，或注明它只影响"玩家能否找到"而不影响可达上限。

4. **`wave_reachable_opt` 的注册表区间仍不一致。** 注册表 `output_range: [0, 99999]`（`:578`）vs 本 GDD `[0, opt_value(M, C, T)]`（`:168`）。上一轮 Recommended #13 未处理，可并入 Blocking #3 的注册表同步。

5. **胃容量仍未列入 Dependencies，但基准 `C0 = 20` 来自它。** `:212-219` 的上游表没有「胃容量」行，`:163` 的 `C` 却写"来源：胃容量系统"。既然健康检查已改为构建期、只读 `C0`，建议补一行「胃容量 | 软（仅设计期）| `C0 = 20` 作为健康基准」——不是为了运行期调用，而是让依赖图完整、`/consistency-check` 能对上。`dish-values.md:269` 对同一参数就是这么处理的。

6. **`menu_vision` 仍被称作"整数等级"。** `:64` 与 `:227` 写「本系统只消费整数等级」；`meta-upgrades.md:49` 规则 12 与 `run-flow.md:71` 都定义为**已解析额度** `Σ effect_value × level`，"不是等级"。`:110` 与公式本身是对的，只有这两处散文是错的——照 `:64` 实现的程序员会把额度再乘一次 `effect_value`。上一轮 Recommended #1 未处理。

7. **AC 1 的"任一 `station_id`"仍过宽。** `:290` 断言"3 个波次、每波恰好 4 道菜、总数 12"，但 Edge `:192-193` 明确允许非整除与小于 `wave_size` 的菜池。改为"`station_canteen`（12 道菜）"或把 `|M| = 12` 写进 GIVEN。上一轮 Recommended #5 未处理。

8. **三处"待回改"已过时。** `:216`「字段为新增，需回改」——`data-config.md:53` 已定义 `wave_size`；`:231`「菜品数值的下游表中本系统仍是「GDD 未写」」——`dish-values.md:277` 已写"GDD 已写"；`:274`、`:283` 的"新增字段请求"同理。上一轮 Recommended #6 未处理，且 `dish-values.md` 本轮被重新盖章，说明它确实没改。

9. **`wave_size` 的区间仍自相矛盾。** Formulas `:87` 写 `1–8`；Dependencies `:216` 写 `int ≥ 1`；`data-config.md:53` 写 `≥ 1`，其 schema `:116` 写 `1–8`。统一成一档，并说明池小于 `wave_size` 被钳制（`:193`）后 `preview` 上限是否随之变化。上一轮 Recommended #7 未处理。

10. **规则 6「代价不是回合」读起来仍与规则 5/8 冲突。** `:31`「代价不是回合，是那几道菜本身」，而 `:29` 与 `:33` 都明说跳波消耗 1 回合。改为「主代价是失去的菜，另付 1 回合跨波」。上一轮 Recommended #8 未处理。

11. **"跳过对上限的影响"表仍缺前提。** `:139-149` 写「每主动跳过一波，就少一个吃菜的回合」，但跳过丢的是该波**剩余**菜。1 次跳过 = 9 道的结果，前提是"先吃掉该波 3 道、再跳过第 4 道"。把前提写进表注，否则读者会以为跳过整波只丢 1 道菜。上一轮 Recommended #9 未处理。

12. **重掷的"下一个种子"仍未定义，存档记哪个种子也仍未写。** `:34`「用下一个种子重掷」——`seed+1`、`seed+k` 还是 RNG 下一输出？`:207` 要求存 `seed`，但存**输入**种子还是**实际采用**的种子没说。两者都应写死，否则读档重放可能得到不同划分。上一轮 Recommended #10 未处理。

13. **存档交互行仍应标"MVP 未启用"。** `:67` 的"条件"与 Edge `:207` 描述中途存档；`save-system.md:31` 规则 5 明确 MVP 不做中途存档、这些契约在 MVP 均不生效。上一轮 Recommended #12 未处理（`:229` 已写"MVP 无中途存档"，但 `:67` 的表行没有）。

14. **跨文档陈旧两处（不在本文件修改范围）。** `run-flow.md:34` 仍写「`consume(dish_id)`（返回 `ADVANCED` 或 `NO_EATABLE`）」、`:296` 的依赖行仍是 `{ADVANCED, NO_EATABLE}` 且缺 `end_pickup()`——`run-flow.md:121` 的正式接口行已正确，这两处是残留散文；`first-run-onboarding.md:370` 仍说「当前只读 `StationData.dish_pool`」，而本 GDD `:62` 已接受 `pool_override`。

15. **Tuning Knobs 仍缺席（standard 层 advisory）。** 本系统确有可调值：`wave_size`（1–8，暂定 4）、健康带 `[0.9, 1.1]`、重掷上限 20、`preview` 上限 `wave_size − 1`、跳过代价 1 回合。它们直接决定单局上限（95–104），应集中列出安全范围与影响面。上一轮 Recommended #14 未处理。

## Nice-to-Have

- `advance_cost` 的"空波自动推进"与"最后一波"两个 0 回合分支在最后一次吃空时重叠（`:123-124`），逻辑无害，可注明优先级。
- `end_pickup()` 在非最后一波被调用时没有定义行为（拒绝？还是等价于结束本局？）。`run-flow.md:33` 只在最后一波路由它，但接口本身没有护栏。
- `consume` 的 `NO_EATABLE` 现在只在最后一波吃空时出现；AC 19 覆盖了信号，但没有一条 AC 直接断言 `consume` 在该情形返回 `NO_EATABLE`。可补半句。
- `:188` 的 Example 仍用 104 / 95 说明阈值偏宽，与双侧带无冲突，但可一并说明上界永不触发（见 Recommended #1）。
- AC 18（`:322`）与 `cloying-and-taste.md:92` 的示例逐字一致；AC 17（`:321`）的 `0 − 85 = −85` 与结算、注册表一致——本轮仍无需改。

## Scope Signal

**Rough scope signal: L**（producer 应在 sprint planning 前复核）

- 5 个公式；4 条上游（数据配置、菜品数值、餐局流程、腻度，另需胃容量的 `C0`）；5 条下游（HUD、结算、局外升级、首局引导、存档）。
- 3 个 ADR 级 Open Question：随机洗牌 vs 策划编排（OQ6）、跳过的代价模型（OQ2）、多档口供给稀缺（OQ5）。
- 5 条跨文档硬契约（`consume` / `advance` / `end_pickup` / `pool_override` / ±10%），本轮闭合 3 条，健康检查的执行模型仍未闭合。

## Verdict: NEEDS REVISION

修复质量比上一轮好：接口层已闭合，AC 也补上了。剩下两条是**同一处文字的内部矛盾**（规则 9 的"落盘"与运行期种子；`:261` 的"不弹确认"与 AC 7 / HUD AC 25），各改一两句即可，不需要机制返工。建议状态 **Needs Revision**（索引当前为 In Review，按并行约束未改）。

Reviewed-Content-Hash: design/gdd/menu-waves.md ce532d3265df1ed7555ba5aa46383be35bbee4a1
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 72584c8a3cc37bfd9c68d285c4223ae8425a63f0
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 5734623b4a60d9072657ef631c8ae61f05359f21
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/save-system.md 772ed596d68eb5cbae207b7bac9a170ec55c1524
Reviewed-Content-Hash: design/gdd/systems-index.md c0b526ff05088d426598f5dc34e3bd9e3153cf59
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 1 | Recommended: 20
Summary: 上一轮 3 条 blocking **全部解决**：规则 9 改为运行期健康检查（构建期只预计算 `opt_value` 基准表）、最后一波「结束取餐」改指 HUD 确认面板、注册表 `wave_healthy` 已同步为双侧带 + `pool/C0/T`。本轮新增 1 条 blocking：规则 9 新加的「单次 ≤ 10 ms」预算与正文指定的穷举算法不相容，按 GDScript 估算单次约 40–250 ms，降级分支会常态化，`wave_healthy` 退化为恒真，±10% 可达校验在运行期实际永不执行（AC 15/16/22 也随之无法稳定通过）。其余为上一轮未处理的 14 条 recommended 与 6 条新发现。
Prior verdict resolved: Yes —— 3/3 解决（注册表 `wave_reachable_opt.output_range` 一项残留，归入 Recommended #5）。

### 复审背景

上一轮对象为 `ce532d32…`（341 行）；本轮为 `4e4854f…`（341 行）。`review-receipts.sh check` 显示目标文件 `CHANGED`，按全文复审执行，不是增量复查。改动全部来自 `cf14f37`（`docs: fix remaining blocking in 7 Needs Revision GDDs`）：本 GDD 4 处 hunk（规则 9、交互表「餐局流程」行、Visual/Game Feel/UI 三节的确认口径），`entities.yaml` 中 `wave_healthy` 的 `variables` + `expression`，`systems-index.md` 的 #4/#7/#10/#11/#16/#17/#18 → In Review。

### 上一轮 blocking 逐条复核

| # | 原 finding | 状态 | 依据 |
|---|---|---|---|
| 1 | 健康检查执行模型互斥（构建期落盘 vs 运行期种子划分） | **已解决** | `:34` 改为「运行期，基准预计算」：开局按本局种子划分 → 用构建期预计算的 `opt_value(pool, C0, T)` 表校验 → 未达标换种子重掷 ≤ 20 次 → 采用最好一次 + warning，不阻断开局。与规则 2 `:26`、规则 10 `:35`、状态表 `:47`、Edge `:205`、AC 2/3 `:291-292`、AC 15/16 `:316-317` 全部一致；"健康划分随数据落盘"已删除。 |
| 2 | 最后一波「结束取餐」的确认去留不一致 | **已解决** | `:243`、`:252`、`:261` 三处统一为「非最后一波点击立即提交、不弹确认；最后一波打开 HUD 确认面板（`in-game-hud.md` 规则 16 / AC 25）」，与 HUD `:59`、`:371`、`:503` 逐条对上；AC 7 `:302`「点击…并确认」、AC 20 `:324`「第 2 波立即提交」各自对应一侧，三条 AC 可同时通过。 |
| 3 | 注册表 `wave_healthy` 未随双侧带同步 | **已解决（主项）** | `entities.yaml:584-597` 现为 `variables: [M, seed, pool, C0, T]`、`expression: 0.9 * opt_value(pool, C0, T) <= wave_reachable_opt <= 1.1 * …`，与本 GDD `:176` 一致。残留：同条目的兄弟 `wave_reachable_opt.output_range` 仍是 `[0, 99999]`（`:578`），见 Recommended #5；GDD 自己的签名 `wave_healthy(M, seed)` 未跟上注册表的五个变量，见 #4。 |

---

## Required Before Implementation

### 1. [BLOCKING] 10 ms 健康校验预算与正文指定的枚举算法不相容，使 ±10% 契约在运行期失效

**位置**：本 GDD `:34`（规则 9 末句「单次健康校验预算 ≤ 10 ms……超预算时只检查每波非空与覆盖，跳过可达最优检查并写 warning」）、`:168`（「三波合计约 2.7×10⁵ 条序列，生成期可穷举；**不需要动态规划**」）、`:316-317`（AC 15/16）、`:326`（AC 22）。

**问题**：按本 GDD 自己定义的搜索空间（每波有序子序列，可随时付 1 回合跳过），MVP 4/4/4、`C = 20`、`T = 10` 下用真实菜表计数（含容量与回合剪枝）约 **1.25×10⁵** 个 DFS 节点，不剪枝上界 `65³ ≈ 2.75×10⁵`。GDScript 每节点按 0.3–2 µs 的保守估计（递归调用 + 数组/字典操作），单次检查约 **40–250 ms**，比 10 ms 高 1–2 个数量级；20 次重掷的最坏路径则是 0.8–5 s 的开局停顿。

后果不是"慢一点"，而是**降级分支常态化**：

- 超预算时只剩「每波非空 + 划分恰好覆盖菜池」——对任何正确实现的 `wave_partition` 都恒真，`wave_healthy` 变成恒真函数；
- `dish-values.md:241`、`:255` 的「±10% 偏差则重掷」这条跨系统契约在运行期**永不执行**；
- AC 15（95 通过）、AC 16（93 触发重掷）、AC 22（越界重掷）在超预算的机器上全部无法通过——三条 AC 没有区分"正常路径"与"降级路径"。

**修复（三选一，均为一句到一段的量）**：
- (a) 把 `wave_reachable_opt` 的计算方式从穷举改为**按 (波索引, 剩余容量, 剩余回合) 的有界 DP**（3 × 21 × 11 状态，几十微秒级），10 ms 预算即刻成立；正文 `:168` 的"不需要动态规划"要一并删掉；
- (b) 保留穷举，把预算改成实测口径并写到能容纳它的量级（例如「单次 ≤ 250 ms，20 次重掷 ≤ 5 s，在目标机上测量」），并说明这是开局一次性的可接受停顿；
- (c) 保留两个数，但把"超预算降级"写成**明确的产品决策**：MVP 默认不跑可达检查，`wave_healthy` 只做结构与覆盖校验，±10% 契约延后到 Vertical Slice——同时删除或改写 AC 15/16/22。

无论选哪条，都应补一条 AC 覆盖预算路径（见 Recommended #7）。

---

## Recommended Revisions

1. **[HIGH] 双侧带的上界按构造恒真（上一轮 #1，未处理）。** `:168` 已写 `wave_reachable_opt ∈ [0, opt_value(M, C, T)]`，而 `:176` 的基准就是同一菜池的 `opt_value(pool, C0, T)`；可达最优 ≤ 静态最优，故 `≤ 1.1 ×` 在无升级的本局永远成立。`dish-values.md:241/:255` 的 ±10% 实际只有下界被执行。建议在 `dish-values.md` 删掉或改口径，或在 `:176` 下注明上界是防回归断言。AC 22 `:326` 因此只能注入下界失败的夹具。

2. **[HIGH] 构建期 `opt_value` 表的键集与"未命中"行为未定义（新）。** `:34` 说用"构建期预计算的 `opt_value(pool, C0, T)` 表"校验，基准是"本局实际菜池（`pool_override` 非空时为覆盖池）"。但 `pool_override` 在 `run-flow.md` / `first-run-onboarding.md` 里是一个通用 `RunConfig` 字段（首局恰为固定六道），GDD 没有说明表里预置了哪些池、缓存未命中时怎么办。建议写成「表键 = 各档口 `dish_pool` + 首局六道池；未命中按超预算降级并写 warning」，或明确 `pool_override` 在 MVP 只能取首局那一个值。

3. **[HIGH] 健康基准忽略本局快照，满级局会把最好的划分判为不健康（新，规则 9 改动的副作用）。** `:34` 的基准是 `C0 = 20`、`cloying_resistance = 0`，而上一版本里解释这一选择的那句「升级只会提高容量，基础值健康的菜单对升级后仍健康」在本次改动中被删掉了——而且这句话本身不成立：`meta-upgrades.md:67` 明说满级把上限抬到容量 25 / 120 元。按 MVP 菜表计算，容量 24 时静态最优已是 **116 > 1.1 × 104 = 114.4**，容量 25 时 120。也就是说，一个把高价值菜正确排进可达范围的**好**划分，会在升级过的局里触发 20 次重掷并按"不健康"记 warning——闸门方向反了。建议：把基准定义为"未升级快照的下界"，上界只对未升级局生效；或直接声明基准固定为 C0 且只做下界闸门（与 #1 一并处理）。

4. **`wave_healthy` 的签名与注册表不一致（上一轮 #2）。** `:176` 仍是 `wave_healthy(M, seed)`，体内却用 `pool`、`C0`、`T`；注册表 `:589-594` 已列出五个变量。改为 `wave_healthy(M, seed, pool, C0, T)`，或在签名下注明三者来自 `StationData` / `BalanceData` 与 `turn_budget`。

5. **注册表 `wave_reachable_opt.output_range` 仍不一致（上一轮 #4）。** `entities.yaml:578` = `[0, 99999]`，本 GDD `:168` = `[0, opt_value(M, C, T)]`。跨文件，本分支未改。

6. **新增的 `get_run_menu()` 只在本 GDD 出现，run-flow 不认（新）。** `:62` 把 `get_run_menu() -> StringName[]` 写进「餐局流程」行的接口清单，`first-run-onboarding.md:127`、`:320` 已消费它，但 `run-flow.md` 全文没有这个函数（`:122`、`:297` 的依赖行都没有）。另外返回值的顺序没定义（波序还是菜池序），而 `replacement_hint` 会按它挑替换建议。建议：定义顺序，并把该函数补进 `run-flow.md:122` 的依赖行。

7. **没有 AC 覆盖 10 ms 预算与降级路径（新）。** AC 15/16/22 都假定可达检查会执行。补一条：「GIVEN 注入一个耗时超过预算的候选生成器，WHEN 开局，THEN 只做非空与覆盖校验、写一条 warning、不重掷、不抛异常，本局正常开始」。这条 AC 也是 Blocking #1 三种修法中任何一种的最小验收。

8. **`wave_reachable_opt` 的 `menu_vision` 参数不被公式使用（上一轮 #3）。** `:155` 有该参数，`:157` 的合法序列定义里没有信息量这一项。删掉，或注明它只影响"玩家能否找到"而不影响可达上限；注册表 `:567-582` 照抄了同一签名。

9. **`menu_vision` 仍被称作"整数等级"（上一轮 #6）。** `:64`、`:227` 写「本系统只消费整数等级」；`meta-upgrades.md:49` 规则 12 与 `:21` 都定义为**已解析额度** `Σ effect_value × level`。照 `:64` 实现会把额度再乘一次 `effect_value`。`:110` 与公式本身是对的。

10. **胃容量未列进 Dependencies，且 `C` 的来源写错了归属（上一轮 #5）。** `:212-219` 的上游表没有「胃容量」行，`:163` 却写 `C` 来源是"胃容量系统"。实际上 `C0` 是 `BalanceData.base_capacity`（`stomach-capacity.md:93` 明写由数据配置提供），注册表 `base_capacity.referenced_by`（`:1266-1272`）也已包含 `menu-waves.md`——注册表认为依赖成立，GDD 的依赖表却没有这一行。补一行「基础容量 `C0` | 软（常量）| `BalanceData.base_capacity = 20`」即可。顺带 `stomach-capacity.md:68` 的「设计期下游 / 运行期不读」在健康检查改为运行期后读起来矛盾，建议改成「只读常量 `C0`，不读运行期容量状态」。

11. **AC 1 的"任一 `station_id`"仍过宽（上一轮 #7）。** `:290` 断言"3 个波次、每波恰好 4 道、总数 12"，而 Edge `:192-193` 明确允许非整除与小于 `wave_size` 的菜池（首局六道池就是 4+2）。把 GIVEN 限定为 `station_canteen`，或把 `|M| = 12` 写进 GIVEN。

12. **三处"待回改"已过时（上一轮 #8）。** `:216`「字段为新增，需回改」——`data-config.md:53` 已定义 `wave_size`；`:231`「菜品数值的下游表中本系统仍是「GDD 未写」」——`dish-values.md` 已写；`:274`、`:283-284` 同理。另 `dish-values.md:11` 仍写 `opt_value` "不进入游戏运行时"，与本 GDD 的"构建期预计算表进入运行期查表"需要一句措辞对齐。

13. **`wave_size` 的区间四处不一致（上一轮 #9）。** 本 GDD `:87` `1–8`、`:216` `int ≥ 1`；`data-config.md:53` `≥ 1`、schema `:116` `1–8`。统一成一档，并说明池小于 `wave_size` 被钳制（`:193`）后 `preview` 的实际上限（`|pool| − 1`）如何随之变化。

14. **规则 6「代价不是回合」仍与规则 5/8 冲突（上一轮 #10）。** `:31`「代价不是回合，是那几道菜本身」，而 `:29`、`:33` 都明说跳波消耗 1 回合。改为「主代价是失去的菜，另付 1 回合跨波」。

15. **"跳过对上限的影响"表仍缺前提（上一轮 #11）。** `:139-149` 说「每主动跳过一波，就少一个吃菜的回合」，但跳过丢的是该波**剩余**菜；1 次跳过 = 9 道的前提是"先吃掉该波 3 道、再跳过第 4 道"。把前提写进表注。

16. **重掷的"下一个种子"仍未定义，存档记哪个种子也没写（上一轮 #12）。** `:34`「换下一个种子」是 `seed+1`、`seed+k` 还是 RNG 下一输出？`:207` 要求存 `seed`，但存**输入**种子还是**实际采用**的种子没说清——两者在重掷后不同，读档重放会得到不同划分。

17. **存档交互行仍应标"MVP 未启用"（上一轮 #13）。** `:67` 的"条件"与 Edge `:207-208` 描述中途存档；`save-system.md:31` 规则 5 明确 MVP 不做中途存档、这些契约在 MVP 均不生效（`:229` 已写对，`:67` 的表行没写）。

18. **跨文档陈旧两处（上一轮 #14，不在本文件修改范围）。** `run-flow.md:34` 仍写「`consume(dish_id)`（返回 `ADVANCED` 或 `NO_EATABLE`）」、`:297` 的依赖行仍是二值且无 `end_pickup()`——同文件 `:122` 的正式接口行已正确，这两处是残留散文；`first-run-onboarding.md:370` 仍说本系统「当前只读 `StationData.dish_pool`」，而同文件 `:127`、`:502` 已按 `pool_override` + `get_run_menu()` 更新。

19. **Tuning Knobs 仍缺席（上一轮 #15，standard 层 advisory）。** 本系统确有可调值：`wave_size`（1–8，暂定 4）、健康带 `[0.9, 1.1]`、重掷上限 20、`preview` 上限 `wave_size − 1`、跳过代价 1 回合、健康校验预算。它们直接决定单局上限（95–104），应集中列出安全范围与影响面。

20. **[低] `wave_size_default.referenced_by` 仍缺 `data-config.md`（上一轮 #13 附带）。** `entities.yaml:1169-1177` 只列了 `menu-waves.md`，但该常量的消费方与 schema 在数据配置。

## Nice-to-Have

- `advance_cost` 的两个 0 回合分支（空波 / 最后一波）在"最后一次吃空"时重叠（`:123-124`），逻辑无害，可注明优先级。
- `end_pickup()` 在非最后一波被调用时没有定义行为（拒绝？等价于结束本局？）。`run-flow.md:33` 只在最后一波路由它，但接口本身没有护栏。
- `consume` 的 `NO_EATABLE` 现在只在最后一波吃空时出现；AC 19 覆盖了信号，但没有一条 AC 直接断言 `consume` 在该情形返回 `NO_EATABLE`。
- `:188` 的 Example 仍用 104 / 95 说明阈值偏宽，可一并说明上界永不触发（见 Recommended #1）。
- GDD 头部 `Status: In Design` / `Last Updated: 2026-11-03` 与索引的 `In Review`、文件实际修改日期不一致；全项目 GDD 都写 `In Design`，属既有约定，但本次修订后可顺手更新。

## Scope Signal

**Rough scope signal: L**（producer 应在 sprint planning 前复核；索引 `systems-index.md:210` 仍标 **S**，本轮信号不变）

- 5 个公式；4 条上游（数据配置、菜品数值、餐局流程、腻度，另需 `BalanceData.base_capacity`）；5 条下游（HUD、结算、局外升级、首局引导、存档）。
- 3 个 ADR 级 Open Question：随机洗牌 vs 策划编排（OQ6）、跳过的代价模型（OQ2）、多档口供给稀缺（OQ5）。
- 5 条跨文档硬契约（`consume` / `advance` / `end_pickup` / `pool_override` / ±10%），本轮接口层全部闭合；新发现的缺口集中在健康检查的性能模型与"谁来付这笔账"。

## Verdict: NEEDS REVISION

上一轮 3 条 blocking 全部解决，修复质量好：执行模型写死在运行期且与全文一致，确认口径按"非最后一波 / 最后一波"分开，注册表主项已同步。新增的 1 条 blocking 不是返工，而是新加的性能预算与既有算法选择互相否定——**10 ms 与"穷举、不需要动态规划"不可能同时成立**，而它的降级分支（只查非空与覆盖）会让整个 ±10% 闸门变成恒真函数。这一条改动很小（换 DP，或改预算数字并明确降级是产品决定），但必须在实现前定下来，否则 AC 15/16/22 在真机上会随机失败。

其余：上一轮 14 条 recommended 再次全部未处理，本轮追加 6 条（#2/#3/#6/#7 为新）。其中 #1–#3 建议与 blocking 一起修，因为它们都落在规则 9 和 `wave_healthy` 这一小段上。建议状态 **Needs Revision**（索引 `systems-index.md:66` 当前为 `In Review`；按并行约束未改索引）。

Reviewed-Content-Hash: design/gdd/menu-waves.md 4e4854fb6a38c7b0989503d8c261b69c5746adbe
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 280240fddb3258864af5f96550cc7cbfd49d42eb
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 1cafff5e7ad8aab8912e716d98b69cea02cbbc43
Reviewed-Content-Hash: design/gdd/save-system.md f22c2228d60be43669125831bccea834e94d6026
Reviewed-Content-Hash: design/gdd/systems-index.md 56e9869c70371816e88ed49f24735f67ab3d199d
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 1 | Recommended: 22
Summary: 上一轮唯一 blocking（10 ms 预算与穷举算法不相容）的**数值矛盾已消除**：预算改为 300 ms 并注明 40–250 ms 的实测区间，规则 9 与正文"可穷举、不需要动态规划"不再互相否定。但该修复只改了数字，没有补上 r3 明确要求的"降级路径 AC"，也没有让降级结果与真通过可区分；同时 300 ms 只覆盖**剪枝后**约 1.25×10⁵ 节点，而 `:168` 正文声明的搜索空间是未剪枝的 2.7×10⁵ 条序列（同一单节点口径下 82–550 ms）。因此本轮仍有 1 条 blocking，范围就是规则 9 与 AC 15/16/22 这一小段；其余 22 条为上一轮未处理的 recommended 与本轮新增的度量口径问题。
Prior verdict resolved: Partial —— 数值不相容已解决（Yes）；降级路径的可判别性与 AC 未解决（No）。见下表。

### 复审背景

上一轮对象为 `4e4854f…`（341 行）；本轮为 `7edb32a…`（341 行），来自 `ecee931`（`docs: fix final blocking in #4/#10/#16/#17`）。全文 hunk 只有 **1 处**：规则 9 `:34` 的「≤ 10 ms（12 道菜、3 波）」改为「≤ 300 ms（12 道菜、3 波最坏情形实测 40–250 ms）」。`review-receipts.sh check` 显示目标文件 `CHANGED`，按全文复审执行；`entities.yaml` 与评审 skill 均 `UNCHANGED`，本次没有新增注册表冲突。

### 上一轮 blocking 逐条复核

| # | 原 finding | 状态 | 依据 |
|---|---|---|---|
| 1 | 10 ms 健康校验预算与正文指定的枚举算法不相容，使 ±10% 契约在运行期失效 | **部分解决** | `:34` 的 10 ms → 300 ms，覆盖了 r3 估算的 40–250 ms，数值上的相互否定**已消除**。但 r3 在同一 blocking 里写明：修复必须让 AC 15/16/22 区分正常/降级路径，并补一条预算路径 AC（Recommended #7「无论选哪条，都应补一条 AC」）。本轮一处未动：降级分支仍是 `:34` 原文的「只检查每波非空与覆盖」，`wave_healthy` 仍是返回 `true` 的布尔函数，AC 15/16/22 仍假定可达检查一定执行。见本轮 Blocking #1。 |

---

## Required Before Implementation

### 1. [BLOCKING] 300 ms 预算只对未写明的剪枝实现成立，且降级结果无法与真通过区分、没有 AC

**位置**：本 GDD `:34`（规则 9 的预算与降级句）、`:168`（搜索空间与"不需要动态规划"）、`:176`（`wave_healthy` 返回 bool）、`:316-317`（AC 15/16）、`:326`（AC 22）。

**问题一：预算覆盖的节点数与正文声明的搜索空间不是同一个数。**
- `:168` 写「每波 4 道菜的有序子序列共 65 种，三波合计约 **2.7×10⁵** 条序列，生成期可穷举」；这是**未剪枝**的上界（65³ = 274,625）。
- `:34` 的「实测 40–250 ms」对应的是 r3 用真实菜表**剪枝后**约 **1.25×10⁵** 个 DFS 节点（容量与回合剪枝）。同一单节点口径（GDScript 0.3–2 µs）下，未剪枝的 2.7×10⁵ 条序列是 **82–550 ms**，上端已越过 300 ms。
- 结论：一个照 `:168` 实现的、不做剪枝的穷举，会在慢机上触发降级分支；而 GDD 从未写剪枝规则，300 ms 的可行性依赖一个未声明的实现细节。300 ms 对 250 ms 实测只有 20% 余量，且没有写测量硬件、构建与节点数，"实测"这个断言在文档内不可复现。

**问题二：降级与真通过不可区分，且没有 AC。**
- 超预算时规则 9 只查「每波非空 + 覆盖」，这两项对任何正确的 `wave_partition` 都恒真；`wave_healthy` 仍返回 `true`。调用方无法从返回值知道「可达检查没跑」。`:34` 要求写 warning，这给了外部可观测性，但布尔返回值本身仍是"假通过"。
- AC 15（95 通过）在降级时**空真**通过；AC 16（93 触发重掷）与 AC 22（越界重掷）在降级时**必然失败**——两条 AC 的结果取决于机器速度，违反 `coding-standards.md` 的测试确定性要求（"Tests must produce the same result every run"）。
- 这正是 r3 Recommended #7 要求补的那条 AC，也是 r3 在 blocking 修复清单里点名的最小验收；本轮未补。

**修复（三小步，均落在同一段）**：
- (a) 把性能模型写死：或写明剪枝规则与**节点数**（≈1.25×10⁵），并把 `:168` 的 2.7×10⁵ 改成剪枝后的口径；或采用 r3 的方案 (a) —— 改为 `(波索引, 剩余容量, 剩余回合)` 的有界 DP（3 × 21 × 11 态），预算即刻成立，`:168` 的"不需要动态规划"一并删除。
- (b) 让降级可判别：`wave_healthy` 改为三态（`HEALTHY / DEGRADED / UNHEALTHY`），或返回 `{healthy: bool, reachable_checked: bool}`；预算按**节点数**或可注入参数计（确定性），墙钟只作二级护栏。这样 AC 与遥测都能区分"真的通过"和"没跑可达检查"。
- (c) 补 AC：GIVEN 注入一个超预算（或超节点数）的候选生成器，WHEN 开局，THEN 只做非空与覆盖校验、结果标记为 `DEGRADED`、写一条 warning、不重掷、不抛异常、本局正常开始。同时把 AC 15/16/22 的 GIVEN 限定为「可达检查已执行」。

---

## Recommended Revisions

上一轮 20 条 recommended **再次全部未处理**（本轮唯一改动是 `:34` 的一个数字）。行号按当前 `7edb32a` 重列：

1. **[HIGH] 双侧带的上界按构造恒真（上一轮 #1）。** `:168` 已写 `wave_reachable_opt ∈ [0, opt_value(M, C, T)]`，`:176` 的基准就是同一菜池的 `opt_value(pool, C0, T)`；可达最优 ≤ 静态最优，故 `≤ 1.1 ×` 在未升级局永远成立。`dish-values.md:241/:255` 的 ±10% 实际只有下界被执行。建议在 `dish-values.md` 删掉或改口径，或在 `:176` 下注明上界是防回归断言；AC 22 只能注入下界失败的夹具。
2. **[HIGH] 构建期 `opt_value` 表的键集与"未命中"行为未定义（上一轮 #2）。** `:34` 用"构建期预计算的 `opt_value(pool, C0, T)` 表"，基准是"本局实际菜池（`pool_override` 非空时为覆盖池）"。但 `pool_override` 是通用 `RunConfig` 字段（`run-flow.md:70`、`:74` 明确"也可用于挑战或调试"，首局恰为固定六道，`first-run-onboarding.md:37`），表里预置哪些池、未命中怎么办都没有写。建议写成「表键 = 各档口 `dish_pool` + 首局六道池；未命中按降级并写 warning」，或明确 MVP 的 `pool_override` 只能取首局那一个值。
3. **[HIGH] 健康检查使用的容量参数未写明是 `C0` 还是本局 `C`（上一轮 #3 的收窄版）。** `:34` 的基准是 `C0 = 20`，但 `:155` 的 `wave_reachable_opt(M, seed, C, T, menu_vision)` 需要一个 `C`，而注册表 `entities.yaml:589-594` 的 `wave_healthy` 变量只列了 `C0`。两种读法结果不同：若可达最优用本局 `C`、基准用 `C0`，升级局（容量 24 时静态最优 116 > 1.1×104 = 114.4，见 `meta-upgrades.md:67`）会把**好**划分判为不健康，方向反了；若两侧都用 `C0`，检查是"未升级基线的划分质量"，单调且保守，是目前唯一自洽的读法。建议在 `:34` 显式写 `wave_reachable_opt(M, seed, C0, T, ·)`，并补一句"升级容量不参与生成校验，只影响玩家实际可达值"。
4. **`wave_healthy` 的签名与注册表不一致（上一轮 #4）。** `:176` 仍是 `wave_healthy(M, seed)`，体内却用 `pool`、`C0`、`T`；注册表 `:589-594` 已列五个变量。改为 `wave_healthy(M, seed, pool, C0, T)`，或注明三者来自 `StationData` / `BalanceData` 与 `turn_budget`。
5. **注册表 `wave_reachable_opt.output_range` 仍不一致（上一轮 #5）。** `entities.yaml:578` = `[0, 99999]`，本 GDD `:168` = `[0, opt_value(M, C, T)]`。跨文件，本分支未改。
6. **`get_run_menu()` 仍只在本 GDD 与首局引导出现，run-flow 不认（上一轮 #6）。** `:62` 写进接口清单，`first-run-onboarding.md:127` 已消费，但 `run-flow.md` 全文仍没有该函数（`:122`、`:297` 的依赖行都没有）。同时返回值的顺序没定义（波序还是菜池序），`replacement_hint` 会按它挑替换建议。建议定义顺序并把该函数补进 `run-flow.md:122`。
7. **`wave_reachable_opt` 的 `menu_vision` 参数不被公式使用（上一轮 #8）。** `:155` 有该参数，`:157` 的合法序列定义里没有信息量这一项。删掉，或注明它只影响"玩家能否找到"而不影响可达上限；注册表 `:567-582` 照抄了同一签名。
8. **`menu_vision` 仍被称作"整数等级"（上一轮 #9）。** `:64`、`:227` 写「本系统只消费整数等级」；`meta-upgrades.md:49` 规则 12 与 `:21` 都定义为**已解析额度** `Σ effect_value × level`。照 `:64` 实现会把额度再乘一次 `effect_value`。`:110` 与公式本身是对的。
9. **胃容量未列进 Dependencies，且 `C` 的来源写错了归属（上一轮 #10）。** `:212-219` 的上游表没有「胃容量」行，`:163` 却写 `C` 来源是"胃容量系统"；`C0` 实际是 `BalanceData.base_capacity`（`stomach-capacity.md:93`），注册表 `base_capacity.referenced_by` 已包含 `menu-waves.md`。补一行「基础容量 `C0` | 软（常量）| `BalanceData.base_capacity = 20`」。
10. **AC 1 的"任一 `station_id`"仍过宽（上一轮 #11）。** `:290` 断言"3 个波次、每波恰好 4 道、总数 12"，而 Edge `:192-193` 允许非整除与小于 `wave_size` 的菜池（首局六道池就是 4+2，`first-run-onboarding.md:455`）。把 GIVEN 限定为 `station_canteen`，或把 `|M| = 12` 写进 GIVEN。
11. **四处"待回改"已过时（上一轮 #12）。** `:216`「字段为新增，需回改」——`data-config.md:53` 已定义 `wave_size`；`:231`「菜品数值的下游表中本系统仍是「GDD 未写」」——`dish-values.md:277` 已写；`:274`、`:283-284` 同理。另 `dish-values.md:37` 仍写 `opt_value` "只用于设计期与 CI……不进入游戏运行时"，与本 GDD"构建期预计算、运行期查表"需要一句措辞对齐。
12. **`wave_size` 的区间仍四处不一致（上一轮 #13）。** 本 GDD `:87` `1–8`、`:216` `int ≥ 1`；`data-config.md:53` `≥ 1`、schema `:116` `1–8`。统一成一档，并说明池小于 `wave_size` 被钳制（`:193`）后 `preview` 的实际上限（`|pool| − 1`）如何变化。
13. **规则 6「代价不是回合」仍与规则 5/8 冲突（上一轮 #14）。** `:31`「代价不是回合，是那几道菜本身」，而 `:29`、`:33` 都明说跳波消耗 1 回合。改为「主代价是失去的菜，另付 1 回合跨波」。
14. **"跳过对上限的影响"表仍缺前提（上一轮 #15）。** `:139-149` 说「每主动跳过一波，就少一个吃菜的回合」，但跳过丢的是该波**剩余**菜；1 次跳过 = 9 道的前提是"先吃掉该波 3 道、再跳过第 4 道"。把前提写进表注。
15. **重掷的"下一个种子"仍未定义，存档记哪个种子也没写（上一轮 #16）。** `:34`「换下一个种子」是 `seed+1`、`seed+k` 还是 RNG 下一输出？`:207` 要求存 `seed`，但存**输入**种子还是**实际采用**的种子没说清——重掷后两者不同，读档重放会得到不同划分。
16. **存档交互行仍应标"MVP 未启用"（上一轮 #17）。** `:67` 的"条件"与 Edge `:207-208` 描述中途存档；`save-system.md` 明确 MVP 不做中途存档（`:229` 已写对，`:67` 的表行没写）。
17. **跨文档陈旧两处（上一轮 #18，不在本文件修改范围）。** `run-flow.md:34` 仍写「`consume(dish_id)`（返回 `ADVANCED` 或 `NO_EATABLE`）」、`:297` 的依赖行仍是二值且无 `end_pickup()`——同文件 `:122` 的正式接口行已正确，这两处是残留散文；`first-run-onboarding.md:370` 仍说本系统「当前只读 `StationData.dish_pool`」，而同文件 `:127`、`:502` 已按 `pool_override` + `get_run_menu()` 更新。
18. **Tuning Knobs 仍缺席（上一轮 #19，standard 层 advisory）。** 本系统确有可调值：`wave_size`（1–8，暂定 4）、健康带 `[0.9, 1.1]`、重掷上限 20、`preview` 上限 `wave_size − 1`、跳过代价 1 回合、健康校验预算。它们直接决定单局上限（95–104），应集中列出安全范围与影响面。
19. **`wave_size_default.referenced_by` 仍缺 `data-config.md`（上一轮 #20）。** `entities.yaml:1171-1173` 只列了 `menu-waves.md`，但该常量的消费方与 schema 在数据配置。
20. **[低] `advance_cost` 的两个 0 回合分支在"最后一次吃空"时重叠（`:123-124`）。** 逻辑无害，可注明优先级。
21. **[新] "实测 40–250 ms"缺测量条件。** 按 `skill-authoring` 的「never assert what you cannot source」，这个数字是 300 ms 唯一的根据，却没有写硬件、构建、节点数与是否剪枝。补一句测量条件，或把口径降为"按 1.25×10⁵ 节点、0.3–2 µs/节点估算"。
22. **[新] 20 次重掷的总开销没有上界。** 规则 9 只限"单次 ≤ 300 ms"；最坏路径 20 × 300 ms = **6 s** 开局停顿（r3 在方案 (b) 里要求的口径是"20 次重掷 ≤ 5 s，并说明这是开局一次性的可接受停顿"）。补一条总预算（总时长或总节点数），或把重掷上限与预算挂钩。

## Nice-to-Have

- `end_pickup()` 在非最后一波被调用时没有定义行为（拒绝？等价于结束本局？）。`run-flow.md:33` 只在最后一波路由它，但接口本身没有护栏。
- `consume` 的 `NO_EATABLE` 现在只在最后一波吃空时出现；AC 19 覆盖了信号，但没有一条 AC 直接断言 `consume` 在该情形返回 `NO_EATABLE`。
- `:188` 的 Example 仍用 104 / 95 说明阈值偏宽，可一并说明上界永不触发（见 Recommended #1）。
- GDD 头部 `Status: In Design` / `Last Updated: 2026-11-03` 与索引的 `In Review`、文件实际修改日期不一致；全项目 GDD 都写 `In Design`，属既有约定，但本次修订后可顺手更新。

## Completeness

`modes.workflow: standard` → 必需 6 节（Overview、Detailed Rules、Formulas、Edge Cases、Dependencies、Acceptance Criteria）**全部存在**；Formulas 为必需（本系统定义阈值、代价与波次参数）。Advisory 缺席：`## Player Fantasy`（`## Game Feel` `:250-256` 覆盖实质）、`## Tuning Knobs`（见 Recommended #18）。`gdd-structure-check.sh` 输出 `ABSENT: Player Fantasy, Tuning Knobs`。

## Dependency Graph

Dependencies `:212-229` 与 Interactions `:57-67` 声明的 GDD 全部存在：`data-config.md`、`dish-values.md`、`run-flow.md`、`cloying-and-taste.md`、`in-game-hud.md`、`settlement-and-entry-fee.md`、`meta-upgrades.md`、`first-run-onboarding.md`、`save-system.md`。缺口：`胃容量` 未进上游表（Recommended #9）。UX 文档 `design/ux/hud.md` 尚未产出，文件已在 `:268` 声明由 `/ux-design` 生成。

## Registry Values

用注册表词条名（`wave_partition` / `visible_dishes` / `preview_dishes` / `wave_reachable_opt` / `wave_healthy` / `wave_size` / `menu_vision` / `base_capacity` / `opt_value` / `cloying_resistance` 等）对全文做大小写不敏感搜索，命中 60 余行。**本次改动（300 ms）不涉及任何注册表词条，未新增数值冲突**。既有缺口：`wave_reachable_opt.output_range`（`entities.yaml:578` = `[0, 99999]` vs `:168` = `[0, opt_value]`，Recommended #5）；`wave_size_default.referenced_by` 缺 `data-config.md`（Recommended #19）；`wave_healthy` 变量已含 `pool/C0/T`，与 `:176` 表达式一致，但 GDD 签名未跟上（Recommended #4）。

## Scope Signal

**Rough scope signal: L**（producer 应在 sprint planning 前复核；索引 `systems-index.md:210` 仍标 **S**）

- 5 个公式；4 条上游（数据配置、菜品数值、餐局流程、腻度，另需 `BalanceData.base_capacity`）；5 条下游（HUD、结算、局外升级、首局引导、存档）。
- 3 个 ADR 级 Open Question：随机洗牌 vs 策划编排（OQ6）、跳过的代价模型（OQ2）、多档口供给稀缺（OQ5）。
- 5 条跨文档硬契约（`consume` / `advance` / `end_pickup` / `pool_override` / ±10%）。

## Verdict: NEEDS REVISION

预算从 10 ms 提到 300 ms **确实解决了上一轮 blocking 的数值矛盾**——300 ms 能容纳 40–250 ms，规则 9 与"穷举、不需要动态规划"不再互相否定。但同一 blocking 里要求的另一半没有做：降级路径没有 AC，降级时的 `wave_healthy` 与真通过不可区分，AC 16/22 的结果仍取决于墙钟；而 300 ms 只覆盖剪枝后的 1.25×10⁵ 节点，`:168` 却仍写着未剪枝的 2.7×10⁵ 条序列（82–550 ms）。这三处都在规则 9 与 AC 15/16/22 这一小段，改动量很小（写明节点数或换有界 DP + 三态返回值 + 一条 AC），但必须在实现前定下来，否则换一台慢机或一个不剪枝的实现，±10% 契约会再次静默失效。

其余 22 条均为文档同步与措辞问题，其中 #1–#3 是 HIGH，建议与 blocking 一起修，因为它们都落在规则 9 和 `wave_healthy` 这一小段上。建议状态 **Needs Revision**（索引 `systems-index.md:66` 当前为 `In Review`；按并行约束未改索引、未改 GDD、未改 `entities.yaml`、未改其它 GDD）。

Reviewed-Content-Hash: design/gdd/menu-waves.md 7edb32ad4c200e2b3a8e858adef8828afd74ce3a
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 280240fddb3258864af5f96550cc7cbfd49d42eb
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 1cafff5e7ad8aab8912e716d98b69cea02cbbc43
Reviewed-Content-Hash: design/gdd/ui-flow.md 46d6938f567d04423303bb4bd789234b6b1ddf6c
Reviewed-Content-Hash: design/gdd/save-system.md a362801aa491c54afaf3aa2448c396fa420c976b
Reviewed-Content-Hash: design/gdd/systems-index.md 75098fd56028b3ec1b1abb5cb53da81e0eccaf6d
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 2 | Recommended: 22
Summary: 上一轮要求的三步修复**形式上全部落地**：`:168` 改为有界 DP 并删除"不需要动态规划"、`wave_healthy` 改三态、AC 23 补齐且 AC 15/16/22 加上「可达检查已执行」。但落地的 DP 只有 `(波索引, 剩余容量, 剩余回合)` 三轴，**无法计算公式本身定义的可达最优**——`satiety_eff` 依赖跨波保留的组腻度计数（`cloying-and-taste.md:27`、`:39`；本 GDD `:61`、`:165`），同波内顺序还影响解腻；因此 693 态与 ≤5×10⁴ 节点的预算不成立。同时三态化只改了一半：`:193` 仍写 `{true, false}`、`:195` 仍写 `= true`，规则 9 仍写"双侧带"而公式只卡下界。修复范围仍集中在同一段，但不能按"再改一个数字"处理。
Prior verdict resolved: No —— 三步中 (b)(c) 的**形式**已满足，(a) 的算法与状态空间不成立，且 (b) 遗留布尔口径。见下表。

### 复审背景

上一轮对象为 `7edb32a…`（341 行）；本轮为 `9075480…`（349 行），来自 `61f2b7f`（`docs: fix last blocking in #4/#10/#16`）。改动 24 行，全部落在规则 9、`wave_reachable_opt` 的 Output Range、`wave_healthy`、AC 15/16/22 与新增 AC 23。`review-receipts.sh check` 对目标文件、注册表、`save-system.md`、`systems-index.md`、`ui-flow.md` 报 `CHANGED`，按全文复审执行。注册表 `wave_healthy` 确已同步为三态（`entities.yaml:595-597`）。

### 上一轮 blocking 逐条复核

| # | 原 finding | 状态 | 依据 |
|---|---|---|---|
| 1 | 300 ms 预算与算法不相容；降级与真通过不可区分；无降级 AC | **No** | (a) 已写有界 DP 并删"不需要动态规划"，但状态轴不足以计算所定义的量（本轮 Blocking #1）；(b) 三态已加、注册表已同步，但 GDD 内留下 `{true,false}` 与 `= true` 两处旧口径（Blocking #2）；(c) AC 23 与 AC 15/16/22 的 GIVEN 限定**已完成**。 |

---

## Required Before Implementation

### 1. [BLOCKING] `(波索引, 剩余容量, 剩余回合)` 三轴 DP 算不出公式定义的可达最优

**位置**：`:168`（状态数与节点预算）、`:155-157`（公式与合法序列）、`:165`（`satiety_eff` 含重复追加）、`:34`（规则 9 用该值做闸门）。

**问题一：状态不是马尔可夫的。**
- `:157` 的合法序列允许"每一波内选择一个**有序子序列**"，`:165` 明确 `satiety_eff` 是"实际饱腹，**含重复追加**"，`:61` 与规则 9 都要求生成校验计入重复惩罚。
- 反例（数据取自 `dish-values.md:46-57`）：先吃红烧肉（饱腹 3，组 `肉`）与先吃清蒸鱼（饱腹 3，组 `鲜`），之后 `(波索引, 剩余容量, 剩余回合)` 完全相同；但接着吃糖醋里脊，前者 `group_cloy(肉)=1` → 占 4+1 = **5 格**，后者 `肉` 计数 0 → 占 **4 格**。两个同状态的可选集与后续价值不同，三轴 DP 会把它们合并，得到一个高于定义值的上界。
- 解腻同样依赖顺序（`cloying-and-taste.md:122`、`:146-149`：酸/冰菜降低其它组计数），而组计数**跨波保留**（`:39`、本 GDD 规则 14），不能按波重置。

**问题二：5×10⁴ 节点的预算无从推导。**
- 正确状态至少要多带两样：当前波内已吃到第几道（有序子序列还剩哪些）与各 `cloying_group` 计数（钳到 `cloy_cap`）。
- 粗算下界：12 道菜"已吃集合"上界 2¹² = 4096（解腻会让同一集合对应不同计数，所以这是下界不是上界）× 21 容量 × 11 回合 ≈ **9.5×10⁵**；换成 MVP 的 9 个组、每组至多 2 道，计数向量上界 3⁹ ≈ 1.97×10⁴，再乘容量/回合/波内位置后更远超 5×10⁴。可达性剪枝会砍掉很多，但 GDD 必须给出**剪枝后的实数**，不能直接断言 693。
- 后果二选一：照 `:168` 字面实现 → 闸门算的是另一个量（系统性偏乐观，坏划分可能被判 `HEALTHY`）；按定义正确实现 → 节点数越过 5×10⁴，正常开局就可能落进 `DEGRADED`，与"DP ≤ 5×10⁴"的断言矛盾。

**修复（择一，都在同一段）**：
- (i) 把状态写全：`(扁平菜序位置 0..11, 剩余容量, 剩余回合, 各组 cloy 计数 ∈ [0, cloy_cap])`，并用一个设计期脚本跑出 MVP 参数下的**实际可达态数**，把 693 / 5×10⁴ 换成实测值；
- (ii) 明确健康校验**不计腻度**（改用静态 `satiety`），同步改 `:61`、`:165` 与注册表 `wave_reachable_opt.expression` 里的 `effective_satiety`，并注明闸门从此是可达值的**上界**、只用于挡明显坏的划分；
- (iii) 回到穷举，但写明剪枝规则与节点数（上一轮 r4 给的 ≈1.25×10⁵）。

> 这不是新开的口子：上一轮 blocking 要的就是"性能模型必须与算法一致"。本轮写下的模型与它要计算的公式不一致。

### 2. [BLOCKING] 三态化只改了一半，健康契约在同一段里自相矛盾

**位置**：`:34`、`:176`、`:189-190`、`:193`、`:195`、`:323-324`、`:333`。

- **返回类型**：`:190` 已改枚举 `HEALTHY / DEGRADED / UNHEALTHY`，但 `:193` 的 Output Range 仍是 `{true, false}`，`:195` 的 Example 仍写 `wave_healthy = true`。
- **闸门上下界**：规则 9 `:34` 仍写"可达最优落在基准的**双侧带 `[0.9, 1.1]`** 内"，变量表 `:189` 仍列"双侧带 `[0.9, 1.1]`"，但 `:179` 的公式与 AC 22 都只检查**下界**。上界按构造恒真（上一轮 Recommended #1），删掉它对；但规则 9 与变量表没跟着改，实施者无法判断上界到底卡不卡。
- **签名与注册表**：`:176` 仍是 `wave_healthy(M, seed)`，注册表 `entities.yaml:590-594` 列的是 `M, seed, pool, C0, T`（上一轮 Recommended #4，本轮仍未改）。
- **注册表措辞**：`entities.yaml:597` 的 `notes` 仍写 "Boolean gate"；`wave_reachable_opt` 的 `notes`（`:580`）仍描述 "65 ordered subsets per wave"，与新的 DP 口径不一致。

**后果**：同一段内并存"三态枚举"与"布尔输出"、"双侧带"与"只卡下界"两套契约。AC 15/16/22/23 在实现任选一种读法时都能通过，闸门行为却不同。

**修复**：一次改齐——规则 9 与变量表删掉上界（只留下界，并注明上界为防回归断言、由 `dish-values` 侧处理）、Output Range 改三态、Example 改 `HEALTHY`、签名补 `pool, C0, T`、注册表两条 `notes` 同步。

---

## Recommended Revisions

上一轮 22 条中 **#21 已解决**（"实测 40–250 ms"已删除，改为节点数为主口径）。其余 20 条**本轮均未处理**，行号按 `9075480` 重列；另加 2 条新的。

| # | 级别 | Finding |
|---|---|---|
| 1 | HIGH | 双侧带上界按构造恒真（上轮 #1）。公式已删上界，但 `dish-values.md:241`、`:255` 仍要求"偏差不得超过 ±10%，否则重掷"——跨文档口径需对齐：要么写成只卡下界，要么注明上界是防回归断言。 |
| 2 | HIGH | 构建期 `opt_value` 表的键集与"未命中"行为仍未定义（上轮 #2）。`pool_override` 是通用 `RunConfig` 字段（`run-flow.md:70`、`:74`）。建议写明表键 = 各档口 `dish_pool` + 首局六道池，未命中按降级并写 warning。 |
| 3 | HIGH | 健康校验的容量参数仍未写明用 `C0` 还是本局 `C`（上轮 #3）。`:34` 基准是 `C0 = 20`，`:155` 的公式吃 `C`；`:168` 的 21 个容量态只在 `C0 = 20` 下成立，而 `meta-upgrades.md:254` 的设计期重算要求容量到 25。建议显式写 `wave_reachable_opt(M, seed, C0, T, ·)` 并说明升级容量不参与生成校验。 |
| 4 | MED | `menu_vision` 参数仍不被公式使用（上轮 #7）：`:155` 有该参数，`:157` 的合法序列里没有信息量这一项；注册表 `:573-577` 照抄了同一签名。 |
| 5 | MED | `menu_vision` 仍被称作"整数等级"（上轮 #8）：`:64`、`:235` 写"本系统只消费整数等级"，而 `meta-upgrades.md:49` 定义为**已解析额度** `Σ effect_value × level`。照 `:64` 实现会把额度再乘一次。 |
| 6 | MED | 注册表 `wave_reachable_opt.output_range` 仍为 `[0, 99999]`（上轮 #5），本 GDD `:168` 为 `[0, opt_value(M, C, T)]`。 |
| 7 | MED | `get_run_menu()` 仍只在本 GDD 与首局引导出现（上轮 #6）：`run-flow.md` 全文（含 `:122`、`:297`）没有该函数；返回值顺序（波序还是菜池序）未定义。 |
| 8 | MED | 胃容量仍未列进 Dependencies（上轮 #9），`:221-225` 的上游表缺「基础容量 `C0`」行；`:163` 的 `C` 归属没错，但 `C0` 来自 `BalanceData.base_capacity`。 |
| 9 | MED | AC 1 的"任一 `station_id`"仍过宽（上轮 #10）：`:297` 断言 3 波 × 4 道 × 12 道，而 Edge `:199-200` 允许非整除与小于 `wave_size` 的菜池（首局六道池是 4+2）。限定为 `station_canteen` 或把 `\|M\| = 12` 写进 GIVEN。 |
| 10 | MED | 四处"待回改"已过时（上轮 #11）：`:223`「字段为新增，需回改」——`data-config.md:53` 已定义；`:231`「菜品数值的下游表中本系统仍是「GDD 未写」」——`dish-values.md:277` 已写；`:281`、`:290-291` 同理。另 `dish-values.md:37` 仍写 `opt_value` 不进入运行时，需一句措辞对齐。 |
| 11 | MED | `wave_size` 区间仍四处不一致（上轮 #12）：本 GDD `:87` `1–8`、`:223` `int ≥ 1`；`data-config.md:53` `≥ 1`、`:116` `1–8`。统一并说明池小于 `wave_size` 被钳制后预览上限的变化。 |
| 12 | MED | 规则 6「代价不是回合」仍与规则 5/8 冲突（上轮 #14）：`:31` 说"代价不是回合，是那几道菜本身"，而 `:29`、`:33` 都明说跳波消耗 1 回合。改为「主代价是失去的菜，另付 1 回合跨波」。 |
| 13 | MED | "跳过对上限的影响"表仍缺前提（上轮 #15）：`:141-149` 的"1 次跳过 = 9 道"以"先吃掉该波 3 道、再跳过第 4 道"为前提，需写进表注。 |
| 14 | MED | 重掷的"下一个种子"仍未定义（上轮 #16）：`seed+1`、`seed+k` 还是 RNG 下一输出？`:214` 要求存 `seed`，但存输入种子还是实际采用种子未写清——重掷后两者不同。 |
| 15 | MED | 存档交互行仍应标"MVP 未启用"（上轮 #17）：`:67` 的"条件"行没写；`save-system.md:31`、`:348` 已明确 MVP 不做中途存档。 |
| 16 | MED | Tuning Knobs 仍缺席（上轮 #19）：`wave_size`、健康带、重掷上限 20、预览上限、跳过代价、校验预算都直接决定单局上限（95–104），应集中列出安全范围与影响面。 |
| 17 | LOW | 跨文档陈旧两处（上轮 #18）：`run-flow.md:34` 仍写 `consume` 返回二值、`:297` 的依赖行二值且无 `end_pickup()`；`first-run-onboarding.md:370` 仍说本系统"当前只读 `dish_pool`"。 |
| 18 | LOW | `wave_size_default.referenced_by` 仍缺 `data-config.md`（上轮 #20）：`entities.yaml:1172-1173` 只列了本 GDD。 |
| 19 | LOW | `advance_cost` 的两个 0 回合分支在"最后一次吃空"时重叠（`:123-124`），逻辑无害，可注明优先级。 |
| 20 | LOW | 20 次重掷的总开销仍无上界（上轮 #22）：最坏 20 × 5×10⁴ 节点（或 20 × 50 ms 墙钟），未写总预算，也未说明这是开局一次性可接受停顿。 |
| 21 | **新** | 墙钟 ≤ 50 ms 只被称作"二级护栏"，但**超时后的动作没定义**。若墙钟超限也返回 `DEGRADED`，AC 15/16/22 的结果又回到机器相关，违反测试确定性。建议写明"只写 warning，不改变返回值；只有节点预算能决定 `DEGRADED`"。 |
| 22 | **新** | 按 GDD 自己声明的 693 态，5×10⁴ 节点预算有约 72 倍余量，**`DEGRADED` 永远不可达**——运行期分支只有注入夹具（AC 23）能走到。要么按真实状态空间重定预算，要么注明该分支仅供防御/测试路径、不参与运行期判定。 |

## Nice-to-Have

- `end_pickup()` 在非最后一波被调用时没有定义行为（拒绝？等价于结束本局？）。`run-flow.md:33` 只在最后一波路由它。
- `consume` 的 `NO_EATABLE` 现在只在最后一波吃空时出现；AC 19 覆盖了信号，但没有一条 AC 直接断言 `consume` 在该情形返回 `NO_EATABLE`。
- GDD 头部 `Status: In Design` / `Last Updated: 2026-11-03` 与索引 `In Review`、文件实际修改日期不一致；全项目 GDD 都写 `In Design`，属既有约定。

## Completeness

`modes.workflow: standard` → 必需 6 节（Overview、Detailed Rules、Formulas、Edge Cases、Dependencies、Acceptance Criteria）**全部存在**；Formulas 为必需（本系统定义阈值、代价与波次参数）。Advisory 缺席：`## Player Fantasy`（`## Game Feel` `:257-263` 覆盖实质）、`## Tuning Knobs`（Recommended #16）。

## Dependency Graph

Dependencies `:217-244` 与 Interactions `:57-69` 声明的 GDD 全部存在。缺口：`胃容量`/`C0` 未进上游表（Recommended #8）。UX 文档 `design/ux/hud.md` 尚未产出，`:275` 已声明由 `/ux-design` 生成。

## Registry Values

三处改动已同步为三态：`entities.yaml:595`（`output_range`）、`:596`（`expression`）、`:592-594`（变量）。未同步：`:597` 的 `notes` 仍写 "Boolean gate"；`wave_reachable_opt` 的 `notes`（`:580`）仍按穷举口径描述。既有缺口：`wave_reachable_opt.output_range`（`[0, 99999]` vs `[0, opt_value]`，Recommended #6）；`wave_size_default.referenced_by` 缺 `data-config.md`（Recommended #18）；`wave_healthy` 的 GDD 签名未跟注册表变量（Blocking #2）。本次改动**未新增数值冲突**。

## Scope Signal

**Rough scope signal: L**（producer 应在 sprint planning 前复核；索引 `systems-index.md:210` 仍标 **S**）

- 5 个公式；4 条上游（数据配置、菜品数值、餐局流程、腻度，另需 `BalanceData.base_capacity`）；5 条下游（HUD、结算、局外升级、首局引导、存档）。
- 3 个 ADR 级 Open Question：随机洗牌 vs 策划编排（OQ6）、跳过的代价模型（OQ2）、多档口供给稀缺（OQ5）。
- 5 条跨文档硬契约（`consume` / `advance` / `end_pickup` / `pool_override` / ±10%）。

## Verdict: NEEDS REVISION

三步修复确实都落纸了，其中 **(c) 降级路径的 AC 与 GIVEN 限定完成得干净**，三态枚举也同步进了注册表——这是上一轮 blocking 里最容易被漏掉的一半，值得肯定。但 (a) 换上的有界 DP 只有三轴状态，而 `satiety_eff` 依赖跨波保留的组腻度、同波顺序还影响解腻：`红烧肉` 与 `清蒸鱼` 各吃一份后，`(波, 容量, 回合)` 相同却让糖醋里脊分别占 5 格与 4 格，三轴 DP 会把这两个状态合并。因此 693 态与 ≤5×10⁴ 节点既算不出定义值，也推不出预算，`DEGRADED` 的触发条件随实现方式漂移。与此同时 (b) 只改了一半：`:193`/`:195` 仍是布尔口径、规则 9 仍要求双侧带，公式却只卡下界。两处都在同一段，改动量仍然很小，但必须先定下状态模型与唯一一套返回契约，否则 ±10% 闸门会以新的形式再次静默失效。

其余 22 条为文档同步与严格性缺口，其中 #1–#3 是 HIGH，且都落在规则 9 与 `wave_healthy` 附近，建议与 blocking 一起修。建议状态 **Needs Revision**（索引 `systems-index.md:66` 当前为 `In Review`；按并行约束未改索引、未改 GDD、未改 `entities.yaml`、未改其它 GDD）。

Reviewed-Content-Hash: design/gdd/menu-waves.md 90754805044963eef29e76a1a031a706e0c30a2f
Reviewed-Content-Hash: design/registry/entities.yaml 5e3b667c300bdf6b9daad9b294aecf224b502b8e
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 280240fddb3258864af5f96550cc7cbfd49d42eb
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 1cafff5e7ad8aab8912e716d98b69cea02cbbc43
Reviewed-Content-Hash: design/gdd/ui-flow.md 35a3291d40d49cf1b6f35190ed231112462f0bc6
Reviewed-Content-Hash: design/gdd/save-system.md 0d0edcdeed2612058390f2a0da75dab9cc15e1da
Reviewed-Content-Hash: design/gdd/systems-index.md b59ef2478102399659f8ae8b533c98a63a1c90b0
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

## Review — 2026-11-04 — Verdict: NEEDS REVISION（第 6 轮）

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 2 | Recommended: 22
Summary: `f7ebe00` / `74bad1a` 把"健康校验不计腻度"写进了规则 9、`wave_reachable_opt` 公式、变量表与注册表，并把 DP 换成 `(菜序位置, 剩余容量, 剩余回合)`；三态枚举、Output Range、Example、签名、注册表**均已同步**（上一轮 blocking #2 的表面部分全部落地）。但两处仍未闭合：**(1)** `:61`、`:226`、`:240`、`:283` 仍断言"生成校验计入 `effective_satiety` / 重复惩罚"，与 `:34`、`:157` 直接对立；`:240` 更把腻度顺序后果的责任判给本系统，而新模型只给上界、无法承担该责任。**(2)** 三轴 DP 只给了状态轴、没给转移；照字面实现的两种自然读法都与穷举不符（随机实例：自由跨波读法 1147/5000 不符，full+skip 读法 937/4000 不符），正确写法需要"逐波重置"这一步，GDD 未写。12 × 21 × 11 ≈ 2.8×10³ 的算术本身正确，但 10⁴ 节点预算只在正确转移下成立。
Prior verdict resolved: No —— 上一轮 blocking #1 的算法问题换了形式再次出现（三轴仍不足以保证定义值），blocking #2 的表面部分已改、实质口径仍分裂。

### 复审背景

对象 `2070cb32…`（349 行），来自 `f7ebe00` + `74bad1a`；与上一轮 `9075480…` 相比 11 行改动，全部落在规则 9、`wave_reachable_opt`、`wave_healthy`。`review-receipts.sh check` 对目标文件、注册表、`save-system.md`、`ui-flow.md` 报 `CHANGED`，其余（含 skill）`UNCHANGED`；按全文复审执行。注册表 `wave_healthy`（三态 `output_range`、`DEGRADED` 表达式、`notes`）与 `wave_reachable_opt`（`[0, opt_value(pool, C0, T)]`、上界说明）确已同步。

### 上一轮 blocking 逐条复核

| # | 原 finding | 状态 | 依据 |
|---|---|---|---|
| 1 | 三轴 DP 算不出定义值；预算不成立 | **部分** | 选 (ii) 不计腻度后，"算不出"的根因（跨波 cloy 计数）已消除，静态模型下三轴**存在**正确解；但转移从未写明，字面实现仍出错（本轮 Blocking #2）。态数算术正确。 |
| 2 | 三态化只改一半 | **表面已闭合，实质未闭合** | `:193` 三态、`:195` `HEALTHY`、`:177` 五参、注册表 notes 均已改；但 `:61`/`:226`/`:240`/`:283` 仍保留"计入腻度"的旧契约（本轮 Blocking #1）。 |

---

## Required Before Implementation

### 1. [BLOCKING] "不计腻度"只改了核心段，四处正文仍断言计入，`wave_healthy` 的责任边界自相矛盾

**位置**：`:61`（Interactions 表）、`:226`（Dependencies 上游表）、`:240`（与腻度的交叉契约）、`:283`（Cross-References）。

- `:34` 与 `:157` 现在明确："健康校验不计腻度……结果是真实可达值的**上界**"；`:165` 的 `satiety` 也改为静态饱腹。这是正确的方向。
- 但 `:61` 仍写"生成校验**需要**把重复惩罚计入可达最优"，`:226` 仍写"生成校验时把 `effective_satiety` 计入可达最优"，`:283` 仍写"`effective_satiety` 计入可达最优"。
- `:240` 更进一步："这个后果由波次的生成校验负责（`wave_reachable_opt` 已计入）"。在不计腻度的模型下，`wave_reachable_opt` 只是**上界**，无法对"解腻菜被切到无法插入的位置"这一腻度顺序后果负责——把它写成"已计入"是把一个上界冒充成精确判定。

**后果**：实施者按 `:34`/`:157` 写静态 DP，按 `:61`/`:226`/`:240`/`:283` 写 `effective_satiety` 版本，会得到两套闸门；后者回到上一轮被否决的状态空间（跨波组计数），且与注册表 `wave_reachable_opt.notes` 的 "ignores cloying" 冲突。AC 15/16/22/23 在两套读法下都能通过，闸门行为却不同。

**修复**：把四处统一为"不计腻度；`wave_reachable_opt` 是真实可达值的上界，只挡明显坏的划分"；`:240` 改写为"本系统只能保证不因波次划分把上界压到阈值以下；腻度顺序造成的额外损失由腻度系统与试玩校准承担，本系统不声称已计入"。

### 2. [BLOCKING] 三轴 DP 只给了状态轴、没给转移；字面实现的两种自然读法都与定义值不符

**位置**：`:168`（态数与节点预算）、`:157`（合法序列）、`:34`（规则 9）、`:211`（回合耗尽的 Edge）。

**已验证的部分**：
- `12 × 21 × 11 = 2772 ≈ 2.8×10³` 的算术正确（`T = 10` → 11 个回合态）。
- 用穷举对照随机实例确认：**存在**一个三轴实现能精确匹配——按波重置的 DP，携带 `(已处理波数, 已用容量, 已用回合)`（`4 × 21 × 11 = 924` 个携带态），每波内部先做子集背包，再在波边界合并；5000/5000 随机实例与穷举一致，MVP 形状实例 98/98 一致。该实现的节点数在 `≤10⁴` 内。

**未写明的部分**（因此字面实现会错）：
- 非末波的推进代价是"吃完整波 = 0 回合；否则 +1 回合"；回合耗尽时，**未吃完的当前波可以免费结束**（Edge `:211`）。这两个条件要求"本波是否吃全"在转移里显式处理。
- 若在菜序 DP 里直接"吃/跳过自由转移、到波尾再补 1 回合"，需要携带"本波是否已弃菜"位；三轴不够，态数翻倍到 ~5.5×10³。
- 若用"整波吃 = 免费跳，否则先付 1 回合再走"的结构，则会漏掉"回合在非末波中途耗尽、免费结束"的合法路径。**反例**（参数在 GDD 允许范围内）：波 0 = `[(饱腹1, 价100), (饱腹9, 价1)]`（非末波），波 1 = `[(1,1)]`（末波），`C0 = 10`、`T = 1`。真实可达 = **100**（吃掉 100 那道，回合耗尽，本局结束）；该结构返回 **0**，阈值 `0.9 × 100 = 90` 会把一个好划分判成 `UNHEALTHY`。
- 两个字面读法在随机实例上的偏差：自由跨越波界 1147/5000 不符（偏乐观）；full+skip 937/4000 不符（偏悲观）。

**修复（择一，都是一句话）**：
- (i) 写明"逐波重置"：每波独立做子集背包，全局只携带 `(波索引, 已用容量, 已用回合)`；非末波两种转移——整波吃（0 跳过）或付 1 回合后吃任意子集；回合耗尽可中途结束；末波永不计跳过。态数改为 `(wave_count + 1) × 21 × 11`（MVP 924），节点预算随之可推。
- (ii) 若坚持菜序轴，补上"本波是否已弃菜"位与中途结束转移，并把态数改为 `2 × 12 × 21 × 11 ≈ 5.5×10³`、节点预算重算。

> 这不是新口子：上一轮 blocking #1 要的是"性能模型与算法一致"；本轮换了状态轴，但转移仍未被钉死，定义值与预算依旧不能同时从文字推出。

---

## Recommended Revisions

上一轮 22 条中 **#4（`menu_vision` 未使用）、#6（注册表 `wave_reachable_opt.output_range`）、#21（墙钟护栏）已解决**；#3、#15、#17 部分解决；其余未处理。按当前 `2070cb32` 重列，另加 3 条新的。

| # | 级别 | Finding |
|---|---|---|
| 1 | HIGH | 上界口径跨文档仍不一致：本 GDD `:34`/`:189` 说上界是"防回归断言、由 `dish-values` 处理"，但 `dish-values.md:241`、`:255` 仍要求"单次刷新 `opt_value` 偏差 > ±10% 就重掷"。对齐成"只卡下界；上界只做回归告警"或反之。 |
| 2 | HIGH | 构建期 `opt_value` 表的键集与"未命中"行为仍未定义（`:34`）：表按档口全池 + 首局六道池建？`pool_override` 用于挑战/调试时未命中怎么办？建议写明"未命中按 `DEGRADED` + warning"，与 AC 23 的降级路径一致。 |
| 3 | HIGH | 设计期容量 25 与生成期 `C0 = 20` 的关系仍未写（上轮 #3 部分解决）：`meta-upgrades.md:67`、`:254` 要求改升级时重跑 `wave_reachable_opt` 到容量 25，而 `:168` 的 21 个容量态按 `C0 = 20` 推出。写明"生成校验只用 `C0`；设计期重算用可变 `C`，预算按 `C + 1` 个容量态"。 |
| 4 | MED | `menu_vision` 仍被称作"整数等级"（`:64`、`:235`）：`meta-upgrades.md:49` 明确快照给的是**已解析额度** `Σ effect_value × level`。照 `:64` 实现会再乘一次。 |
| 5 | MED | `get_run_menu()` 仍只在本 GDD 与首局引导出现（`:62`、`first-run-onboarding.md:127`、`:320`）；`run-flow.md` 全文没有该函数，返回值顺序（波序还是菜池序）未定义。 |
| 6 | MED | 胃容量/`C0` 仍未进上游 Dependencies（`:221-225`）：`:163` 把 `C0` 的 Source 写成"数据配置 `BalanceData.base_capacity`"，但注册表 `base_capacity` 的 `source` 是 `stomach-capacity.md`（`entities.yaml:1265-1272`）。补一行上游或改 Source 归属。 |
| 7 | MED | AC 1 的"任一 `station_id`"仍过宽（`:297`）：断言 3 波 × 4 道 × 12 道，而 Edge `:199-200` 允许非整除与小于 `wave_size` 的菜池（首局六道池 = 4+2）。限定为 `station_canteen` 或把 `\|M\| = 12` 写进 GIVEN。 |
| 8 | MED | 四处"待回改"仍过时：`:223`「字段为新增，需回改」——`data-config.md:53` 已定义；`:238`「菜品数值的下游表中本系统仍是「GDD 未写」」——`dish-values.md:277` 已写；`:281`、`:290-291` 同理。 |
| 9 | MED | `wave_size` 区间仍不一致：本 GDD `:87` `1–8`、`:223` `int ≥ 1`；`data-config.md:53` `≥ 1`、`:116` `1–8`。统一并说明池小于 `wave_size` 被钳制后预览上限的变化。 |
| 10 | MED | 规则 6「代价不是回合」仍与规则 5/8 冲突（`:31` vs `:29`、`:33`）。改为「主代价是失去的菜，另付 1 回合跨波」。 |
| 11 | MED | "跳过对上限的影响"表仍缺前提（`:141-149`）："1 次跳过 = 9 道"以"先吃掉该波 3 道、再跳过第 4 道"为前提；`:170` 的 Example 还隐含"去掉的两道正好是 `白米饭`/`拍黄瓜` 且位于末波"。写进表注/例注。 |
| 12 | MED | 重掷的"下一个种子"仍未定义（`:34`、`:212`）：`seed+1`、`seed+k` 还是 RNG 下一输出？`:214` 要求存 `seed`，但存输入种子还是实际采用种子未写清。 |
| 13 | LOW | 存档交互行仍未标"MVP 未启用"（`:67` 的"条件"行），而下游表 `:236` 已改。两处口径统一。 |
| 14 | MED | Tuning Knobs 仍缺席（advisory at `standard`）：`wave_size`、健康带、重掷上限 20、预览上限、跳过代价、校验预算都直接决定单局上限（95–104），应集中列出安全范围与影响面。 |
| 15 | LOW | 跨文档陈旧（上轮 #17 部分解决）：`run-flow.md:297` 仍写 `consume → {ADVANCED, NO_EATABLE}` 且无 `end_pickup()`（`:121-122` 已同步）；`first-run-onboarding.md:370` 仍说本系统"当前只读 `dish_pool`"，而同文件 `:502` 已把该 OQ 标为已回答。 |
| 16 | LOW | `wave_size_default.referenced_by` 仍缺 `data-config.md`（`entities.yaml:1172-1173` 只列本 GDD）。 |
| 17 | LOW | `advance_cost` 的两个 0 回合分支在"最后一次吃空"时重叠（`:123-124`），逻辑无害，可注明优先级。 |
| 18 | LOW | 20 次重掷的总开销仍无上界（`:34`）：最坏 20 × 10⁴ 节点，未写总预算或"开局一次性可接受停顿"的口径。 |
| 19 | LOW | `DEGRADED` 在 MVP 仍不可达（正确实现的节点数 ≤ 10⁴），只能由 AC 23 的注入夹具走到。建议注明"防御/测试路径，不参与运行期判定"。 |
| 20 | **新** | `wave_healthy(M, seed, pool, C0, T)` 有两个池参数却未定义区别（`:177`）：`:69` 说 `pool_override` 非空时划分与校验都在覆盖池上跑，则 `M` 与 `pool` 恒相等，`pool` 冗余；若二者不同，`⊎ W_i ≠ M` 与 `opt_value(pool,…)` 会校验两个不同的池。合并或写明。 |
| 21 | **新** | 注册表细节：`wave_reachable_opt.output_range` 引用了 `pool`，但 `variables` 只列 `M, seed, C0, T`（`entities.yaml:572-577`）；`wave_healthy`、`wave_advance_cost` 条目缺 `added`/`revised`。 |
| 22 | **新** | 规则 9 的降级措辞与公式优先级不一致：`:34` 说"超节点预算时只检查每波非空与覆盖，返回 `DEGRADED`"，而 `:178-180` 的公式是"空波/覆盖不过 → `UNHEALTHY`，覆盖通过且跳过可达检查 → `DEGRADED`"。改为"只检查非空与覆盖：通过则 `DEGRADED`，不通过则 `UNHEALTHY`"。 |

## Nice-to-Have

- `end_pickup()` 在非最后一波被调用时没有定义行为（拒绝？等价于结束本局？）。`run-flow.md:33` 只在最后一波路由它。
- `consume` 的 `NO_EATABLE` 现在只在最后一波吃空时出现；AC 19 覆盖了信号，但没有一条 AC 直接断言 `consume` 在该情形返回 `NO_EATABLE`。
- GDD 头部 `Status: In Design` / `Last Updated: 2026-11-03` 与索引 `In Review`、文件实际修改日期不一致；全项目 GDD 都写 `In Design`，属既有约定。

## Completeness

`modes.workflow: standard` → 必需 6 节（Overview、Detailed Rules、Formulas、Edge Cases、Dependencies、Acceptance Criteria）**全部存在**；Formulas 为必需（本系统定义阈值、代价与波次参数）。Advisory 缺席：`## Player Fantasy`（`## Game Feel` `:257-263` 覆盖实质）、`## Tuning Knobs`（Recommended #14）。

## Dependency Graph

Dependencies `:217-244` 与 Interactions `:57-69` 声明的 GDD 全部存在。缺口：`胃容量`/`C0` 未进上游表（Recommended #6）。UX 文档 `design/ux/hud.md` 尚未产出，`:275` 已声明由 `/ux-design` 生成。

## Registry Values

`wave_healthy` 与 `wave_reachable_opt` 的两处同步**已完成**：三态 `output_range`、`DEGRADED` 表达式、`[0, opt_value(pool, C0, T)]`、上界说明、五参变量表均与 GDD 一致（`entities.yaml:567-596`）。剩余缺口：`wave_reachable_opt.variables` 缺 `pool`（Recommended #21）；`wave_size_default.referenced_by` 缺 `data-config.md`（#16）；`wave_advance_cost` 缺 `added`/`revised`（#21）。本次改动**未新增数值冲突**。

## Scope Signal

**Rough scope signal: L**（producer 应在 sprint planning 前复核；索引 `systems-index.md:210` 仍标 **S**）

- 5 个公式；4 条上游（数据配置、菜品数值、餐局流程、腻度，另需 `BalanceData.base_capacity`）；5 条下游（HUD、结算、局外升级、首局引导、存档）。
- 3 个 ADR 级 Open Question：随机洗牌 vs 策划编排（OQ6）、跳过的代价模型（OQ2）、多档口供给稀缺（OQ5）。
- 5 条跨文档硬契约（`consume` / `advance` / `end_pickup` / `pool_override` / ±10%）。

## Verdict: NEEDS REVISION

`f7ebe00`/`74bad1a` 的方向是对的：把健康校验改成"静态饱腹 + 上界"，并同步了公式、变量表、Output Range、Example、签名与注册表——上一轮 blocking #2 的**表面部分**这次改齐了，三态枚举与注册表口径现在一致，值得肯定。但"不计腻度"只改了核心段：`:61`、`:226`、`:240`、`:283` 仍四处断言计入 `effective_satiety`/重复惩罚，`:240` 还把腻度顺序后果的责任判给本系统。同一条契约在文档内分裂成两套，实施者可以任选一套而 AC 全过。另一方面，三轴 DP 的**状态轴**这次可以承载静态模型（穷举验证：按波重置的写法 5000/5000 一致、924 个携带态、预算内），但 GDD 只写了轴、没写转移；字面实现的两种自然读法分别偏乐观（1147/5000 不符）与偏悲观（937/4000 不符），且"回合在非末波中途耗尽可免费结束"这条 Edge 会让 full+skip 写法把好划分误判 `UNHEALTHY`。两处都不需要重设计，各补一段契约即可，但必须先钉死唯一一套，否则闸门会以第三种形式再次静默失效。

其余 22 条为文档同步与严格性缺口，其中 #1–#3 是 HIGH，且都落在规则 9 与 `wave_healthy` 附近，建议与 blocking 一起修。建议状态 **Needs Revision**（索引 `systems-index.md:66` 当前为 `In Review`；按并行约束未改索引、未改 GDD、未改 `entities.yaml`、未改其它 GDD）。

Reviewed-Content-Hash: design/gdd/menu-waves.md 2070cb32e98733d3f873b792546fb6c7fd0083d0
Reviewed-Content-Hash: design/registry/entities.yaml 66d713b708546a4f5beb044b8abe9f93d7fd04e1
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 280240fddb3258864af5f96550cc7cbfd49d42eb
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 1cafff5e7ad8aab8912e716d98b69cea02cbbc43
Reviewed-Content-Hash: design/gdd/ui-flow.md 22c40c147277af09d1d3c9e3c8be149bfe0126f8
Reviewed-Content-Hash: design/gdd/save-system.md 16b4a4a8fc45095392100202545509c7305a244f
Reviewed-Content-Hash: design/gdd/systems-index.md b59ef2478102399659f8ae8b533c98a63a1c90b0
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean)
Blocking items: 1 | Recommended: 4
Summary: 上一轮两项 blocking 在 GDD 正文中已关闭——「不计腻度」的四处口径统一，三轴 DP 改为逐波重置并补齐转移（MVP 924 携带态）。唯一剩余 blocking 不在 GDD，而在注册表：`wave_reachable_opt` 仍描述旧的菜序 DP。
Prior verdict resolved: Yes
Findings:
- [BLOCKING] Registry: `wave_reachable_opt.notes` 仍写 "DP over (dish position, remaining capacity, remaining turns)"，与 GDD 的逐波重置 `(waves processed, capacity used, turns used)` 不符；expression 未写明每吃一道计 1 回合。
- [RECOMMENDED] dish-values: ±10% 上界重掷契约仍与本 GDD 的「只卡下界」冲突（carried）。
- [RECOMMENDED] Formulas: 设计期容量 25 与生成期 `C0 = 20` 的关系仍未写（carried）。
- [RECOMMENDED] Formulas: 924 携带态 × 每波至多 16 个子集与「≤ 10⁴ 节点」的计数口径需说明。
- [RECOMMENDED] Registry: `wave_advance_cost = 1` 与公式 `advance_cost ∈ {0,1}` 需区分「付费跳过代价」与「推进代价」。

Reviewed-Content-Hash: design/gdd/menu-waves.md a7ed4b78ce361c02dac574838660f2c6b50918cf
Reviewed-Content-Hash: design/registry/entities.yaml 66d713b708546a4f5beb044b8abe9f93d7fd04e1
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f

---

## Review — 2026-11-04 — Verdict: APPROVED（确认 pass）
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean)
Blocking items: 0 | Recommended: 4（沿用上一条）
Summary: 注册表 `wave_reachable_opt` 已同步为逐波重置 DP：924 携带态、每吃一道 1 回合、非末波跳过再 +1、回合耗尽免费结束。GDD 正文未改动；上一条的 recommended 项保留，不阻断。
Prior verdict resolved: Yes
Findings:
- none（上一条 blocking #1 已由注册表同步关闭）

Reviewed-Content-Hash: design/gdd/menu-waves.md a7ed4b78ce361c02dac574838660f2c6b50918cf
Reviewed-Content-Hash: design/registry/entities.yaml 08de917004eb720428b21dee6a0359c90b31c5ec
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 2 | Recommended: 10

Summary: D2 改写后的全文复审（不是 delta）。规则 9、逐波重置 DP（924 携带态、吃菜 1 回合、非末波跳过 +1、回合耗尽免费结束）、0.9 单侧门与 OQ2/OQ4 统计全部用穷举独立复核并逐项对上：`opt_value = 104`、34,650 个 3×4 划分的最小可达 97（510 个恰为 97）、100% 通过 93.6、0.95 通过率 91.11%、接受均值 100.79 vs 不收紧 100.53、`wave_size = 3` 时 104 可达率 4.55%。两条 blocking：注册表 `wave_reachable_opt.notes` 的示例序列 100/95 与本 GDD 的 101/97 冲突（跨文件，未改）；`menu_vision` 在两处散文里被称作"整数等级"（已就地改）。其余 10 条是口径与跨文档同步，AC 1 越界、规则 6 自相矛盾、`M`/`pool` 冗余等已在复审中修好。

Prior verdict resolved: No — 上一条 APPROVED 的收据 `a7ed4b78` 与当前字节不符（D2 改写了规则 9、可达最优 DP 与 OQ2/OQ4）；且那次是确认 pass（只盖了 5 个文件、沿用前一条的 4 条 recommended），`menu_vision` 口径与注册表示例值因此从未被全文检查。

Findings:
- [BLOCKING] Registry: `wave_reachable_opt.notes` 写 "104 with no skips, 100 with one, 95 with two"，与本 GDD 的 104/101/97（`跳过对上限的影响`表、`wave_healthy` Example、校准契约段）冲突。穷举 34,650 个划分复核：恰好 1 次跳过的最大可达 **101**、2 次 **97**，注册表的 100/95 在任何口径（最大值）下都不成立。需注册表同步，本文件不改。
- [BLOCKING] Interactions / Dependencies（局外升级行）: 把 `menu_vision` 称作"本系统只消费整数等级"，与 `meta-upgrades.md` 规则 12/14（快照给的是**已解析额度** `Σ effect_value × level`，"不是等级"）和 `run-flow.md` 的"总预览额度"冲突；变量表与公式本身是对的，按该散文实现会把额度再乘一次 `effect_value`。**已在复审中修正**为"已解析的整数预览额度"。
- [RECOMMENDED] Acceptance Criteria: AC 1 的 "任一 `station_id`" 与 Edge Case（首局六道池、`wave_size` 被钳制为单波）冲突，断言在合法档口上必然失败。**已限定为 `station_canteen`（`|M| = 12`）。**
- [RECOMMENDED] Detailed Rules: 规则 6 "代价不是回合" 与规则 5/8（跳过花 1 回合、共用预算）自相矛盾。**已改为"主代价是失去的菜，另为跨波付 1 回合"。**
- [RECOMMENDED] Formulas: `wave_healthy(M, seed, pool, C0, T)` 的 `M` 与 `pool` 未定义区别（划分集合 vs 基准集合）。**已补"恒为同一有效菜池（`pool_override` 生效时为覆盖池）"。**
- [RECOMMENDED] Formulas: "跳过对上限的影响" 表未写 9/8 道菜的前提（先在被跳过的波里吃掉保留菜、再付 1 回合）。**已补前提，并注明末波永不付跳过回合。**
- [RECOMMENDED] Dependencies / Cross-References: 四处"需回改/待回改"已过时——`data-config.md` 已定义 `wave_size`，`dish-values.md` 下游表已回指本系统。**已清理为现状。**
- [RECOMMENDED] Detailed Rules: 规则 9 的"超节点预算……返回 `DEGRADED`"未写 `UNHEALTHY` 优先。**已补：非空/覆盖不通过仍为 `UNHEALTHY`。**
- [RECOMMENDED] Open Questions: OQ2/OQ4 的 "E[重掷] = 1.10" 实为 E[尝试次数]（91.11% → 期望重掷 0.098、期望尝试 1.098）；标签把重掷代价放大约 11 倍。为不重开 D2，**未改任何数值与结论**，留待决策留痕决定是否更正标签。
- [RECOMMENDED] Cross-doc: 局内 HUD 规则 16 / AC 24 的按钮文案写「跳过本波 · 花 1 回合」，缺本 GDD 要求的「放弃剩余 N 道」（Visual/Audio、UI Requirements、AC 20）。HUD 已 Approved，建议其下次修订对齐。
- [RECOMMENDED] Cross-doc: `dish-values.md` 规则 11 与公式段仍写 `opt_value` "不进入游戏运行时"；ADR-0006 §4 已声明本系统健康门是唯一例外、并要求 dish-values 下次修订补一句。本 GDD 与 ADR 一致，不改。
- [RECOMMENDED] Registry metadata: `wave_size_default.referenced_by` 缺 `data-config.md`；`wave_reachable_opt.variables` 缺 `pool`（其 expression 用到）。非数值冲突。
- 备注（advisory at `standard`）：`## Tuning Knobs` 仍缺席；`get_run_menu()` 在 `run-flow.md` 全文仍未出现（首局引导依赖它）。

Reviewed-Content-Hash: design/gdd/menu-waves.md effad23c5f5bd13b0c6a657ab32c2680a078fc6b
Reviewed-Content-Hash: design/registry/entities.yaml ac9dd64cdbbf756d272d52dd8d27bed6bf7a5b27
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/dish-values.md 8b48f38c7e01eb84a462bf847a8bbac3a7101997
Reviewed-Content-Hash: design/gdd/data-config.md 89626e0d680a97b2887e2d2e76f07a4e3126ec2a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md e6aa08b28eb7739e9a5c342f358f9f55b5292bde
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/in-game-hud.md f81ce0c2348338cd8bd3858ed659e2b13c882a06
Reviewed-Content-Hash: design/gdd/meta-upgrades.md ba07eaa56adfaa489b0ba0fdac9065b492f68c64
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 9bf45c1ead7e161599186ca5724fa460b044e314
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md db09042623fa8333cd6410aa8e97249c1f45e9cf
Reviewed-Content-Hash: design/gdd/ui-flow.md cf471d2d45476ff8e1db0a5fa4a7fae9a4e146ea
Reviewed-Content-Hash: design/gdd/save-system.md 1bd3148294c81eb9a186f218cbb390b30b0e08ad
Reviewed-Content-Hash: design/gdd/systems-index.md e0174253c2b36603edfef7b4d00d005380530feb
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md f8bbbce1efbe2312fb8283ea8c73d61591e92f7a

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION（复审：正文已清，剩余 blocking 在注册表）

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean）
Blocking items: 1 | Recommended: 5

Summary: 上一条的两条 blocking 里，GDD 内那条（`menu_vision` 等级/额度）已关闭；本轮按全文重读复核，正文未再发现缺陷，D2 统计仍逐项可复现。唯一剩余的 blocking 在注册表：`wave_reachable_opt.notes` 的示例序列 100/95 需改为 101/97。该文件不在本分支可改范围，等编排会话同步后即可确认通过。

Prior verdict resolved: 部分 — GDD 内 blocking 已修（`menu_vision` 口径、AC 1、规则 6、`M`/`pool`、表前提、`DEGRADED` 优先级、过时的"待回改"）；注册表 blocking 仍未解决（跨文件）。

Findings:
- [BLOCKING] Registry: `wave_reachable_opt.notes` 的 "104 with no skips, 100 with one, 95 with two" → 应为 "104 with no skips, 101 with one, 97 with two"（与正文表、Example、校准契约、穷举复核一致）。未改注册表。
- [RECOMMENDED] Open Questions: "E[重掷] = 1.10" 标签（实为 E[尝试次数]），未改，待决策留痕决定是否更正。
- [RECOMMENDED] Cross-doc: 局内 HUD 按钮文案缺「放弃剩余 N 道」。
- [RECOMMENDED] Cross-doc: `dish-values.md` 的 `opt_value` "不进入游戏运行时" 措辞（ADR-0006 §4 已声明例外）。
- [RECOMMENDED] Registry metadata: `wave_size_default.referenced_by` 缺 `data-config.md`；`wave_reachable_opt.variables` 缺 `pool`。
- [RECOMMENDED] 索引：`design/gdd/systems-index.md:67` 仍为 `In Review`；本分支不改索引，待本文件通过后由编排会话改为 `Approved`。

Reviewed-Content-Hash: design/gdd/menu-waves.md 4c2da744c1c4b19bc861bd0c09509d6deecbd6b7
Reviewed-Content-Hash: design/registry/entities.yaml ac9dd64cdbbf756d272d52dd8d27bed6bf7a5b27
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/dish-values.md 8b48f38c7e01eb84a462bf847a8bbac3a7101997
Reviewed-Content-Hash: design/gdd/data-config.md 89626e0d680a97b2887e2d2e76f07a4e3126ec2a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md e6aa08b28eb7739e9a5c342f358f9f55b5292bde
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/in-game-hud.md f81ce0c2348338cd8bd3858ed659e2b13c882a06
Reviewed-Content-Hash: design/gdd/meta-upgrades.md ba07eaa56adfaa489b0ba0fdac9065b492f68c64
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 9bf45c1ead7e161599186ca5724fa460b044e314
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md db09042623fa8333cd6410aa8e97249c1f45e9cf
Reviewed-Content-Hash: design/gdd/ui-flow.md cf471d2d45476ff8e1db0a5fa4a7fae9a4e146ea
Reviewed-Content-Hash: design/gdd/save-system.md 1bd3148294c81eb9a186f218cbb390b30b0e08ad
Reviewed-Content-Hash: design/gdd/systems-index.md e0174253c2b36603edfef7b4d00d005380530feb
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md f8bbbce1efbe2312fb8283ea8c73d61591e92f7a

---

## Review — 2026-11-04 — Verdict: APPROVED（复审：注册表 blocking 已关闭）

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 5

Summary: 按 Phase 1 的「目标文档 `UNCHANGED`、只有注册表 `CHANGED`」分支执行：`menu-waves.md` 与上轮收据 `4c2da74` 逐字节相同，正文未再改动；`entities.yaml` 由 `ac9dd64` 变为 `02f5c07`（`e1b4f22`）。逐项复核注册表 `wave_reachable_opt`：`notes` 现写 "104 with no skips, 101 with one, 97 with two"，与正文「跳过对上限的影响」表（`:146-148`）、`wave_reachable_opt` Example（`:181`，97）、`wave_healthy` Example（`:206`，97）、与结算的校准契约（`:253`，1 次约 101 / 2 次约 97）四处一致；`variables` 已补 `pool`，`output_range`（`[0, opt_value(pool, C0, T)]`）与 `expression`（每吃一道 1 回合、非末波跳过 +1、`eats + skips ≤ T`、回合耗尽免费结束）仍与逐波重置 DP 一致；`wave_healthy` 五参含 `pool`、三态 `output_range` 与 94 下界（`0.9 × 104`）一致；`wave_size_default.referenced_by` 已补 `data-config.md`。上轮唯一 blocking 关闭，注册表未引入其它数值改动，未发现新的跨文件冲突，签发 APPROVED。

Prior verdict resolved: Yes — 上轮唯一 blocking（注册表 `notes` 的 100/95）已由 `e1b4f22` 改为 101/97；该提交共 4 处 hunk，其余三处仅为 `referenced_by` 补录（`cloy_cap`、`wave_size_default`、`station_canteen_entry_fee`、`base_capacity`），不涉及取值。

Findings:
- none（上轮 blocking 已由注册表 `e1b4f22` 关闭；GDD 正文自 `4c2da74` 起未改动，上轮已全文复核无缺陷）
- [RECOMMENDED] Open Questions: OQ2/OQ4 的 "E[重掷] = 1.10" 标签（实为 E[尝试次数]），沿用上轮，未改数值与结论。
- [RECOMMENDED] Cross-doc: `in-game-hud.md` 规则 16 / AC 24 的跳过按钮文案缺本 GDD 要求的「放弃剩余 N 道」。
- [RECOMMENDED] Cross-doc: `dish-values.md` 仍写 `opt_value` "不进入游戏运行时"，与 ADR-0006 §4 声明的健康门例外不符。
- [RECOMMENDED] Registry: `wave_healthy.notes` 写 "Reroll up to 20 seeds"，本 GDD 的口径是「最多 20 次候选种子（按 `(station_id, seed, attempt)` 派生）」——同为 20，仅措辞待下次同步。
- [RECOMMENDED] 索引：`systems-index.md` 本系统仍为 `In Review`，通过后应由编排会话改为 `Approved`（本分支不改索引）。

### 复审方式与收据口径

本次工作树基线 `113a3e3` 早于 `d29023a` / `e1b4f22`，工作树内的 `menu-waves.md` 与注册表落后于编排分支。复审前已把 `menu-waves.md` 对齐到 `4c2da74`（与 `main` 逐字节相同，未做任何设计改动），并以 `main` 的 blob 生成以下收据哈希。`design/registry/entities.yaml` 未在本分支修改（约束禁止跨文件改动），收据中的 `02f5c07` 即 `e1b4f22` 之后的注册表内容；`meta-upgrades.md` 的 `3fa3e55` 为本轮复核 `MENU_VISION`（已解析额度 `Σ effect_value × level`、预览上限 `min(menu_vision, wave_size−1, |W_next|)`）所读的当前版本。

Reviewed-Content-Hash: design/gdd/menu-waves.md 4c2da744c1c4b19bc861bd0c09509d6deecbd6b7
Reviewed-Content-Hash: design/registry/entities.yaml 02f5c072fc535530c710fd71d77d31e2dce021ed
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/dish-values.md 8b48f38c7e01eb84a462bf847a8bbac3a7101997
Reviewed-Content-Hash: design/gdd/data-config.md 89626e0d680a97b2887e2d2e76f07a4e3126ec2a
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md e6aa08b28eb7739e9a5c342f358f9f55b5292bde
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 77622412130d27b404e93a2fe2cdf3a99a8718ab
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/in-game-hud.md f81ce0c2348338cd8bd3858ed659e2b13c882a06
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 3fa3e55b62e80ebcb1ed150a57577416f639faf9
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 9bf45c1ead7e161599186ca5724fa460b044e314
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md db09042623fa8333cd6410aa8e97249c1f45e9cf
Reviewed-Content-Hash: design/gdd/ui-flow.md cf471d2d45476ff8e1db0a5fa4a7fae9a4e146ea
Reviewed-Content-Hash: design/gdd/save-system.md 1bd3148294c81eb9a186f218cbb390b30b0e08ad
Reviewed-Content-Hash: design/gdd/systems-index.md e0174253c2b36603edfef7b4d00d005380530feb
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md f8bbbce1efbe2312fb8283ea8c73d61591e92f7a

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: S（沿用 `systems-index.md:220` 的 scope 令牌；按技能的实施规模口径实为 L）
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 3 | Recommended: 8

**C1 回写与收据口径**：本文件最后一次 APPROVED（2026-11-04，收据 `4c2da74`）早于今日改动，而今日改动是**实质性**的（改 `wave_healthy` 判定式、一条变量下界与 AC 15/22），旧批准据 `.claude/rules/design-docs.md`「Review receipts and post-approval edits」失效，因此本轮是当前字节的**全文 lean 复审**，不是 delta。今日的 C1（健康门改整数交叉相乘）回写在公式 `:190`、变量表 `:200`（`ceil(9 × opt_value / 10)`，MVP 104 → 94）、AC 15 `:333`、AC 22 `:343`、注册表 `wave_healthy`（`:611-612`）与 `opt_value.notes` `:458`、ADR-0006 §4/§6（`:251`、`:306-314`、Validation #1 `:528` / #12 `:704`）、`control-manifest.md:108`、`dish-values.md:37/:241/:372` 之间已经一致，公式与注册表层已无第二种判定式残留；**漏掉的只有 Core Rule 9 `:35`** 这一处规范语句仍写浮点，以及 `DEGRADED` 分支与注册表元数据。本轮用整数 DP 穷举复核了全文统计：34,650 个有序 3×4 划分，可达最优最小 97（恰 510 个）、最大 104、全部 ≥ 94，`opt_value(20,10) = 104`，0.95 门槛拒 8.89%、接受均值 100.79（不收紧 100.53），`wave_size = 3` 时 104 占 4.55%——全部与正文一致；`wave_partition`/`advance_cost`/`preview_dishes`/`end_pickup`/`pool_override` 健康口径与 `run-flow.md:35-37`、`time-and-turns.md:53`、`in-game-hud.md:59/:372` 不冲突。

Findings:

| # | 级别 | file:line | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | BLOCKING | `design/gdd/menu-waves.md:35` | Core Rule 9 仍把门槛写成「**可达最优 ≥ `0.9 ×` `opt_value(pool, C0, T)`**」。这是**规则语句**（不是理由叙述），也是本文件今日唯一未被 C1 覆盖的判定式写法；按它实现就会在 `src/feature/menu_waves/` 里写出一个 `float` 常量与浮点比较。改为 `10 × wave_reachable_opt ≥ 9 × opt_value(pool, C0, T)`（与 `:190`/`:200` 同形）。注：在 double 下该谓词对整数 `r` 与 `r ≥ ceil(9 × b / 10)` 在 `b ≤ 2000` 内尚未测到分歧，所以本条不是「已证实的错判」，而是**明文禁止的写法**——ADR 之所以禁止，是因为 `9 × b / 10` 不可精确表示（`104 → 93.60000000000001`）且是否越界依赖舍入与 `0.9` 的书写方式，一旦有人把 `0.9` 写成 `float(9) / 10` 或做累乘，行为就会漂移 | ADR-0006 §6 `docs/architecture/adr-0006-deterministic-rng.md:314`（「**不得**写成 `reachable < 0.9 * baseline`」）、Validation #1 `:528`、#12 `:704`（`src/feature/menu_waves/` 静态扫描禁 `float`）；`control-manifest.md:108`；与本文件 `:190`/`:200` 自相矛盾 |
| 2 | BLOCKING | `design/gdd/menu-waves.md:191`（并 `:35`） | `DEGRADED` 只定义为「可达检查因**超节点预算**被跳过」，缺 ADR-0006 的另外两个来源：§4 `:251` 的 `BASELINE_CELL_BUDGET = 65536`（基准算不出 → `base = -1` → `DEGRADED` + `BASELINE_BUDGET_EXCEEDED`）与 §5 `:303` 的 `STATE_CELL_BUDGET = 262144`（→ `TRUNCATED`）。按本文件公式字面实现，`base < 0` 会走「其余」分支得 `HEALTHY`（`10 × r < 9 × (−1)` 恒假），与 ADR §6 `:324-328` 的 `if reach.truncated or base < 0: return DEGRADED` 相反；也与本文件 `:201`「`DEGRADED` 表示可达检查未执行」的较宽定义不自洽。AC 23 `:344` 只覆盖节点预算一条路径。后续档口（OQ5、更大 `T`）会真实触发 | ADR-0006 §4 `:251`、§5 `:300-304`、§6 `:316-332`；`control-manifest.md:108`；本文件 `:201` |
| 3 | BLOCKING | `design/registry/entities.yaml:599-612` | `wave_healthy` 条目缺 `added:` 与 `revised:` 两个字段（其余条目均有），而它今日刚被实质改写；同一改动的 `opt_value` 已正确盖 `revised: "2026-11-06"`（`:460`）。缺必填字段 + 未留版本戳，下一个读注册表的会话无法判断该表达式是何时改的 | 注册表自身规则 `design/registry/entities.yaml:12`（值变更须 `revised:` 今日并留旧值注释）、`:47` 与 `:64`（必填 `added`）；同批 `opt_value` 的做法 |
| 4 | RECOMMENDED | `design/gdd/menu-waves.md:206`（并 `:181`、`:293`、`:353`） | 散文仍以 `93.6` 表述下界（`:206`「100% 通过 93.6 的下界」，位于 Formulas 段内），而 `:200`/`:333` 的下界是 `94 = ceil(9 × 104 / 10)`；`:181` 写「0.9 阈值（94）」，同一门槛两种写法。均为理由叙述、整数判据下等价，但 Formulas 段内的 `93.6` 有被测试夹具照抄的风险，建议统一为「94（`ceil(9 × 104 / 10)`）」或 `9/10` | 本文件 `:200`/`:333`；`.claude/rules/design-docs.md`「Balance values must link to their source formula」 |
| 5 | RECOMMENDED | `design/gdd/systems-index.md:67` | 索引已写 **Approved**，而本文件头部 `:3` 为 `In Review`（今日实质改动待复审）。索引超前于复审结论；本轮不改索引，建议改为 `Needs Revision` | `.claude/rules/design-docs.md`「Review receipts and post-approval edits」；本技能 Phase 5 的索引口径 |
| 6 | RECOMMENDED | `design/gdd/menu-waves.md:281` | UI Requirements 的「本系统只发起扣除」正是 `time-and-turns.md:219`/`:259`/`:301` 点名要本文件统一措辞的一句（Owner：菜单波次）。语义可兼容，但唯一扣回合调用者是餐局流程，`advance()` 从不碰计数器；现有写法会诱导出第二个写入点 | `time-and-turns.md:53`（规则 13）、`:259`/`:301`；ADR-0006 §7 `docs/architecture/adr-0006-deterministic-rng.md:371-375`；`control-manifest.md:120` |
| 7 | RECOMMENDED | `design/gdd/menu-waves.md:63` | 交互表「餐局流程」行列出 `get_run_menu() -> StringName[]`，但 `design/gdd/run-flow.md` 全文没有该接口（只有 `first-run-onboarding.md:154` 把它记在菜单波次名下）。归属与出借方不明 | `.claude/rules/design-docs.md`「Dependencies must be bidirectional」；`run-flow.md:122`/`:298` |
| 8 | RECOMMENDED | `design/gdd/menu-waves.md:156`、`:179` | `wave_reachable_opt` 签名不含 `pool`，Output Range 写 `[0, opt_value(M, C0, T)]`，而 `:158`/`:190`/`:200` 与注册表 `:593` 用 `pool`，注册表 `variables` 还列了 `pool`（`:592`）。`:204` 已声明二者恒相同，属命名不统一 | 注册表 `wave_reachable_opt` `:582-597`；本文件 `:204` |
| 9 | RECOMMENDED | `design/gdd/menu-waves.md:50`、`:282` | States 表与 UI Requirements 只给「跳过代价」而不给「可用性」：S4 的 `turns_remaining ≥ 2` 前置由 `run-flow.md:35`/`time-and-turns.md:53` 执行、由 `in-game-hud.md:372` 直接按 `turns_remaining` 置灰按钮，而 `in-game-hud.md:182` 又称「跳过/结束取餐的代价与**可用性**归菜单波次」。三分归属需本文件给出一个显式判据 | `in-game-hud.md:182`/`:372`；`run-flow.md:35`；本文件 `:31` |
| 10 | RECOMMENDED | `design/gdd/in-game-hud.md:59` | 跳过按钮文案仍为「跳过本波 · 花 1 回合」，缺本文件要求的「放弃剩余 N 道」（`menu-waves.md:261`/`:279`/AC 20 `:341`）。HUD 已 Approved，属其下次修订 | 本文件 `:261`/`:279`/`:341` |
| 11 | RECOMMENDED | `design/gdd/menu-waves.md:353`、`:355` | OQ2/OQ4 的「E[重掷] = 1.10」实为 E[尝试次数]（91.11% → 期望重掷 0.099）；沿用上轮，未改数值与结论 | 口径标签，无跨文档权威 |

**Completeness**: 6/6 required at `standard` — Overview、Detailed Rules（`## Detailed Design`）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 因本系统定义阈值与代价而**必需**，已计入。`## Player Fantasy` 与 `## Tuning Knobs` 缺席，均为 `standard` 下的 advisory（Tuning Knobs 值得补：`wave_size`、0.9 下界、20 次重掷上限、`NODE_BUDGET`）；不阻断。

**Dependency Graph**: 九个声明依赖全部存在 —— `data-config.md` ✓、`dish-values.md` ✓、`run-flow.md` ✓、`cloying-and-taste.md` ✓、`in-game-hud.md` ✓、`settlement-and-entry-fee.md` ✓、`meta-upgrades.md` ✓、`first-run-onboarding.md` ✓、`save-system.md` ✓。下游回指已核（`data-config.md:190`、`dish-values.md:87`、`run-flow.md:122`、`in-game-hud.md:169`、`settlement-and-entry-fee.md:342`、`meta-upgrades.md:348`、`first-run-onboarding.md:154`、`save-system.md:356`）。

**Registry Values**: 按注册表 79 个条目名与数值键做全文检索（`wave_healthy`、`wave_reachable_opt`、`opt_value`、`advance_cost`/`wave_advance_cost`、`wave_size`/`wave_size_default`、`wave_partition`、`preview_dishes`、`visible_dishes`、`menu_vision`、`end_pickup`、`pool_override`）。数值层面无冲突：`:190`/`:200`/`:343` 与 `wave_healthy.expression`/`notes` 与 `opt_value.notes` 均为整数交叉相乘、94 下界；`wave_size = 4`（`:88`）与 `wave_size_default` 一致；`advance_cost = 1`（`:136`）显式对上 `wave_advance_cost = 1`；`preview_dishes` 公式（`:104`）与注册表 `:577` 逐字一致；`C0 = 20`、`T = 10` 与 `base_capacity`/`station_canteen_turn_budget` 一致；101/97 与注册表 `:595` 一致。唯一注册表问题为元数据类 blocking #3 与命名类 recommended #8。

Reviewed-Content-Hash: design/gdd/menu-waves.md 4282c9a66552dbb5023418b9c19542c1b5787af3
Reviewed-Content-Hash: design/registry/entities.yaml f7b5aded130f371160d84bdaf678f8530ddc0807
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/control-manifest.md 15a70b4ad03b641e3c3873772ec05137d76ecc50
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/gdd/in-game-hud.md fee7199048c3a124fcab2eccae8b88f93d1a69c0
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: design/gdd/data-config.md 9ecc2a7c18aba33c64bc3391594748e1063851d2
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/save-system.md a7d54e3ba4d7f8a23be1d28a6c165bb422648abd

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION（第 8 轮：健康门整数化 + `DEGRADED` 双来源的全量 lean 复审）

Review mode: lean · Workflow: standard
Scope signal: S（沿用 `systems-index.md:220` 的 scope 令牌；按技能的实施规模口径实为 L）
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 1 | Recommended: 8
Summary: 上一轮 3 条 blocking 中，规则 9 的浮点写法（`menu-waves.md:35`）已改为整数交叉相乘并显式禁止 `0.9 ×`；`DEGRADED` 在 GDD 侧已补齐两个来源（`:188-198`，AC 23a `:351` 钉 `BASELINE_CELL_BUDGET = 65536` → `baseline_opt = −1`，AC 23b `:352` 钉 `STATE_CELL_BUDGET = 262144` → `TRUNCATED`）；注册表 `wave_healthy` 补回了 `added`/`revised`。全文复核 34,650 个有序 3×4 划分、`opt_value(20,10) = 104`、下界 `94 = ceil(9×104/10)`、101/97/104 与 0.95 敏感性统计，数值一致。唯一 blocking 是注册表 `wave_healthy.expression`/`notes` 仍只承认「节点预算超限」一个 `DEGRADED` 来源——GDD 已修、注册表没跟上，按注册表实现会复活 `base = −1 → HEALTHY` 的误判。
Prior verdict resolved: Yes（上一轮 3 条 blocking 全部关闭；但注册表条目只做了浮点→整数那一半，`DEGRADED` 来源未同步）

Findings:
- [BLOCKING] Registry Values（`design/registry/entities.yaml:611-612`）：`wave_healthy.expression` 与 `notes` 仍把 `DEGRADED` 定义为「可达检查因节点预算被跳过」这单一来源，缺 `baseline_opt = −1`（`BASELINE_CELL_BUDGET = 65536` → `BASELINE_BUDGET_EXCEEDED`）与 `STATE_CELL_BUDGET = 262144` / 波数×子集上限 → `TRUNCATED` 两个来源；与 `menu-waves.md:188-198`、`:35`、AC 23a、AC 23b 及 ADR-0006 §6 `:324-328` 相反，照注册表字面实现会得到 `10 × r < 9 × (−1)` 恒假的 `HEALTHY`。该条目同时缺注册表规则 `:11-12` 要求的旧值注释（`added`/`revised` 已补，旧值注释未补）。
- [RECOMMENDED] Formulas（`:195`）：「判据顺序固定为 `UNHEALTHY` → `DEGRADED` → `HEALTHY`，与 ADR-0006 §6 对齐」不是 ADR 的实际顺序（ADR §6 `:322-328`：非空/覆盖 → `DEGRADED` → 交叉相乘 → `HEALTHY`）。结论等价（截断时交叉相乘不可判），但"对齐"字面不成立，建议照抄 ADR 的三步。
- [RECOMMENDED] Formulas（`:207`）：变量 `可达检查已执行` 的定义只写「节点预算内完成」，而截断还有 `STATE_CELL_BUDGET` 与波数×子集两个来源（`:198`、AC 23b）；定义应覆盖三者。
- [RECOMMENDED] Formulas（`:156`/`:179` vs `:190`/`:200`）：`wave_reachable_opt` 签名与 Output Range 用 `M`，判定式与注册表用 `pool`；`:204` 已声明恒同，属命名不统一（carried）。
- [RECOMMENDED] Core Rules（`:35`）：「未达标就换下一个候选种子重掷」未区分 `UNHEALTHY` 与 `DEGRADED`；ADR §6 `:347-348` 与 AC 23 规定 `DEGRADED` 直接采用、不重掷。规则 9 应写明重掷只对 `UNHEALTHY`。
- [RECOMMENDED] Acceptance Criteria（`:309-352`）：没有一条 AC 钉住「非空/覆盖失败优先于 `DEGRADED`」——ADR §6 的第一分支（截断 + 空波应返回 `UNHEALTHY`）无覆盖；建议补一条夹具。
- [RECOMMENDED] Open Questions（`:361`/`:363`）：OQ2 写 `E[尝试次数] = 1.10`、OQ4 写 `E[重掷] 1.10`，同一数字两种口径（91.11% 通过 → 期望重掷 0.099）；数值与结论未变（carried）。
- [RECOMMENDED] UI Requirements（`:63`）：`get_run_menu()` 已注明「消费者是首局引导、`run-flow.md` 不调用它」（与 `first-run-onboarding.md:154` 一致），但仍在「餐局流程」行的接口列里；建议移到本系统自有接口或单列。
- [RECOMMENDED] systems-index（`:67`）：索引仍写 `Approved`，本文件头部 `:3` 为 `In Review`（2026-11-06 实质改动待复审）；本轮不改索引，建议编排会话改为 `In Review`。

**Completeness**: 6/6 required at `standard` — Overview、Detailed Rules（`## Detailed Design`）、Formulas、Edge Cases、Dependencies、Acceptance Criteria；Formulas 因本系统定义阈值、预算与代价而必需。`## Player Fantasy` 与 `## Tuning Knobs` 缺席，均为 `standard` 下 advisory（Tuning Knobs 仍值得补：`wave_size`、九成下界、20 次重掷、`NODE_BUDGET`/`STATE_CELL_BUDGET`/`BASELINE_CELL_BUDGET`）。

**Dependency Graph**: 九个声明依赖全部存在 —— `data-config.md` ✓、`dish-values.md` ✓、`run-flow.md` ✓、`cloying-and-taste.md` ✓、`in-game-hud.md` ✓、`settlement-and-entry-fee.md` ✓、`meta-upgrades.md` ✓、`first-run-onboarding.md` ✓、`save-system.md` ✓；下游回指逐份复核仍在（`data-config.md:190`、`dish-values.md:37`、`run-flow.md:123`、`in-game-hud.md:169`/`:389`、`settlement-and-entry-fee.md:342`、`meta-upgrades.md:11`、`first-run-onboarding.md:154`、`save-system.md:31`）。无断链，无单向依赖。

**Registry Values**: 按注册表 79 个条目名全文扫描（`wave_healthy`、`wave_reachable_opt`、`opt_value`、`preview_dishes`、`wave_partition`、`wave_advance_cost`、`base_capacity`、`cloy_cap` 等，`_` 展开为 `[_ ]`），命中 8 个条目。数值层无冲突：`:104` 与 `preview_dishes`、`:190`/`:200` 与 `wave_healthy`、`:136` 与 `wave_advance_cost = 1`、`:88` 与 `wave_size_default = 4`、`:35`/`:174` 与 `base_capacity = 20`、`:142-148`/`:181` 与 `wave_reachable_opt.notes` 的 104/101/97 逐项一致；`94`、`104`、`97`、`34,650`、`0.95` 的散文统计与公式互洽。唯一冲突是 blocking 项的 `DEGRADED` 来源集，另有 `revised` 缺旧值注释。

Reviewed-Content-Hash: design/gdd/menu-waves.md a0b26691ca0f6ba5cac4c01ac0704913179b43ed
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8
Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: design/gdd/time-and-turns.md 0f2ea128a9a42008413a15dd8608246513ef36f4
Reviewed-Content-Hash: design/gdd/in-game-hud.md 27e2978106c35139e3a3d0f371472a7aafd5dc2a
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 43fc521a857c5ce53ba3e1855c271ae2ffdfb1b6
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: design/gdd/data-config.md 92b3a28fc63465eacf3a88d3fec13bdff9f9455f
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204

---

## Review — 2026-11-17 — Verdict: APPROVED（第 9 轮：注册表 `wave_healthy` 三来源补齐后的全量 lean 复审）

Review mode: lean · Workflow: standard
Scope signal: S（沿用 `systems-index.md:220` 的 scope 令牌；按技能的实施规模口径实为 L）
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 11
Summary: 上一轮唯一的 blocking 已在注册表关闭 —— `design/registry/entities.yaml:611-612` 的 `wave_healthy.expression`/`notes` 现在列出三类 `DEGRADED` 来源（基准 `BASELINE_CELL_BUDGET` → `base < 0` / `BASELINE_BUDGET_EXCEEDED`；可达 DP 在 `STATE_CELL_BUDGET` 截断；可达检查因节点预算被跳过），写明 `Never HEALTHY when base < 0 or reach.truncated`，判据也换成整数交叉相乘 `10 * wave_reachable_opt < 9 * opt_value(...)`，并补回 `added` / `revised: "2026-11-06"` 与旧值说明。该条目与 `menu-waves.md:35`（规则 9）、`:188-198`（`DEGRADED` 区块）、AC 23 `:350` / 23a `:351` / 23b `:352`、ADR-0006 §4 `docs/architecture/adr-0006-deterministic-rng.md:251` / §5 `:300-304` / §6 `:322-328` 在判定集合、分数判据与优先级上逐项一致：`base < 0` 与两种截断都不再落回 `HEALTHY`。round-1 的两条亦复核仍在（规则 9 的整数交叉相乘与"不得写成 `0.9 ×`"；GDD 侧 `DEGRADED` 三触发齐全）。全文无残留浮点健康门（`0.9`/`0.95` 只出现在禁令与决策记录中），MVP 数值 94 / 104 / 97 / 101 / 34,650 / 0.95 的散文、公式与注册表互洽，AC 编号 1–23 + 23a/23b 无重复。
Prior verdict resolved: Yes（2026-11-06 的唯一 blocking 已关闭；2026-11-04 的 APPROVED 针对旧字节，本轮按当前字节做全量复审）

Findings:

- [RECOMMENDED] Formulas（`:195`）：「判据顺序固定为 `UNHEALTHY` → `DEGRADED` → `HEALTHY`，与 ADR-0006 §6 对齐」仍不是 ADR 的实际三步（空/覆盖 → `DEGRADED` → 交叉相乘 → `HEALTHY`）。结论等价（截断时交叉相乘不可判），但"对齐"字面不成立；建议照抄 ADR 的三段式（carried）。
- [RECOMMENDED] Formulas（`:207`）：变量 `可达检查已执行` 的定义仍只写"节点预算内完成了可达最优 DP"，而 `:198` 与 AC 23b 另有两个截断来源（`STATE_CELL_BUDGET`、波数 × 子集空间上限）；定义应覆盖三者（carried）。
- [RECOMMENDED] Core Rules（`:35`）：「未达标就换下一个候选种子重掷」仍未区分 `UNHEALTHY` 与 `DEGRADED` —— AC 23 `:350` 与 ADR §6 `:347-348` 规定 `DEGRADED` 直接采用、不重掷。建议改为"仅 `UNHEALTHY` 重掷"。
- [RECOMMENDED] Registry Values（`design/registry/entities.yaml:611-612`）：三来源未点名 ADR §5 `:303` 的第三个 `TRUNCATED` 触发（"波数 × 子集提前可判超限"），现由 "skipped over the node budget" 概括 —— 语义上覆盖（该预判即"一定超节点预算"），故不阻断，但与 `menu-waves.md:35`/`:198` 的三触发字面不一致，建议逐项列全。
- [RECOMMENDED] Registry Values（`:612`）：旧值说明只述性质（"named only the node-budget skip"），既未给出旧表达式原文，也未点名改动它的 GDD，未满足注册表 `:11-12` 的"旧值 + 由哪份 GDD 改动"。
- [RECOMMENDED] Formulas（`:156`/`:179` vs `:190`/`:200`、注册表 `:582-597`）：`wave_reachable_opt` 的签名与 Output Range 用 `M`，判定式与注册表用 `pool`（`:204` 已声明二者恒同，属命名不统一，carried）。
- [RECOMMENDED] UI Requirements（`:63`）：`get_run_menu()` 已注明消费者是首局引导、`run-flow.md` 不调用，但仍留在「餐局流程」行的接口列；建议移到本系统自有接口（carried）。
- [RECOMMENDED] UI Requirements（`:288`）：「放不进剩余容量的菜置灰……这不阻止玩家付费跳过」在 `:285` 已把可用性归 `run-flow` 之后仍读作无条件保证；`turns_remaining = 1` 时 `NO_TURNS` 会拦住付费跳过，建议补一句"是否可用读 `action_availability()`"。
- [RECOMMENDED] Acceptance Criteria（`:350`）：`NODE_BUDGET` 由 DP 求值器消耗，不由"候选生成器"消耗，AC 23 的夹具措辞指向了错对象；建议改为"注入必然超 `NODE_BUDGET` 的求值夹具"。
- [RECOMMENDED] Cross-References / 索引（`design/gdd/systems-index.md:67`）：索引仍写 `Approved`，本文件头部 `:3` 为 `In Review`；本轮按约束不改索引。
- [RECOMMENDED] 跨文档（`docs/architecture/adr-0006-deterministic-rng.md:375`）：ADR §7 仍引「GDD 规则 5 说『本系统只发起扣除』」，该句已于 2026-11-06 从 `:287` 删除改为「只声明代价……从不写回合计数」；属 ADR 侧陈旧引文（owner architecture）。
- [RECOMMENDED] Open Questions（`:361`/`:363`）：OQ2 写 `E[尝试次数] = 1.10`、OQ4 写 `E[重掷] 1.10`，同一数字两种口径（91.11% 通过 → 期望重掷 0.099，carried）。

**Completeness**: 6/6 required at `standard` —— Overview、Detailed Rules（`## Detailed Design`）、Formulas、Edge Cases、Dependencies、Acceptance Criteria（`gdd-structure-check.sh` 输出 `PRESENT: Overview,Detailed Rules,Formulas,Edge Cases,Dependencies,Acceptance Criteria`）。Formulas 因本系统定义阈值、预算与代价而必需，已计入。`## Player Fantasy` 与 `## Tuning Knobs` 缺席，均为 `standard` 下 advisory（Tuning Knobs 仍值得补：`wave_size`、九成下界、20 次重掷、`NODE_BUDGET`/`STATE_CELL_BUDGET`/`BASELINE_CELL_BUDGET`）。

**Dependency Graph**: 九个声明依赖全部存在 —— `data-config.md` ✓、`dish-values.md` ✓、`run-flow.md` ✓、`cloying-and-taste.md` ✓、`in-game-hud.md` ✓、`settlement-and-entry-fee.md` ✓、`meta-upgrades.md` ✓、`first-run-onboarding.md` ✓、`save-system.md` ✓；下游回指逐份复核仍在（`data-config.md:191`、`dish-values.md:278`、`run-flow.md:121`、`in-game-hud.md:19`、`settlement-and-entry-fee.md:342`、`meta-upgrades.md:348`、`first-run-onboarding.md:425`、`save-system.md:356`、`cloying-and-taste.md:245`）。无断链，无单向依赖。

**Registry Values**: 按注册表 79 个条目名与数值键全文扫描（`_` 展开为 `[_ ]`，大小写不敏感）——`wave_healthy`、`wave_reachable_opt`、`opt_value`、`preview_dishes`、`wave_partition`、`wave_size(_default)`、`wave_advance_cost`、`base_capacity`、`station_canteen_turn_budget`、`cloy_cap`、`capacity_used`、`running_net`、`end_reason`。数值层无冲突：`:104` 与 `preview_dishes.expression`（`:577`）、`:190`/`:200`/`:339` 与 `wave_healthy`、`:136` 与 `wave_advance_cost = 1`、`:88` 与 `wave_size_default = 4`、`:35`/`:174` 与 `base_capacity = 20`、`:35`/`:175` 与 `station_canteen_turn_budget = 10`、`:142-148`/`:181`/`:211` 与 `wave_reachable_opt.notes` 的 104/101/97 逐项一致；`advance_cost ∈ {0,1}` 与 `transition_turn_cost = 0` 不冲突。唯一注册表问题为上述元数据类 recommended，无数值冲突。

**Verdict 备注**：本轮 blocking 为零。round-2 的注册表修复在语义上完整（既不漏 `base < 0`，也不把截断误判为 `HEALTHY`），剩余 11 条均为文字精度、命名统一与跨文档陈旧引文，不改变任何规则、AC 或数值。

- **建议 systems-index 状态**：`In Review`（当前 `:67` 为 `Approved`）；本轮按约束**未修改** `design/gdd/systems-index.md`。
- **本 GDD 与注册表未被修改**：本次只评审。

Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8
Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/gdd/in-game-hud.md b8ead1263039519de2ff2ad9378fe43a8ea58373
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/data-config.md 36fd1ebaca36d1edee0c93b674f21d0418798713
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/save-system.md a7d54e3ba4d7f8a23be1d28a6c165bb422648abd
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 65248e9286c1f43dbf1c07e037544a10e0f0bb01
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/meta-upgrades.md d9314a9ea5813e2b921939d5275e97261557e4a9
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/save-system.md a7d54e3ba4d7f8a23be1d28a6c165bb422648abd

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/menu-waves.md af47aef125e14677602eeab6eed5ae98132c6113
