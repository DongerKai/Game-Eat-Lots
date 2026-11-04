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
