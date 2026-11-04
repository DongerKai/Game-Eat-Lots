# Cross-GDD Review Report

- **Date**: 2026-11-04
- **GDDs Reviewed**: 18 of 18 system GDDs present
- **Not read**: 无（但 2f 的部分 AC 区间未覆盖，见文末「未能评估的部分」）
- **Focus**: full（一致性 + 设计整体性 + 跨系统场景走查）
- **Reviewer**: `/review-all-gdds`，三个阶段各由独立 game-designer 子代理执行
- **Systems Covered**: 数据配置、输入、场景与流程管理、存档、餐局流程、时间与回合、胃容量、菜品数值、腻度与口味、菜单波次、结算与入场费、局外升级、解锁与档口、纪录与里程碑、局内 HUD、界面流程、音效反馈、首局引导

> 本报告只提出问题与选项，**不判定哪一份 GDD 是对的**。每一条 blocking 都已在当前正文中逐条复核，不依赖复审日志。

---

## 一、跨文档一致性

### Blocking

#### 🔴 B1. `±10% 重掷` 是双侧重掷契约，但菜单波次只实现了单侧下界

- **涉及**: `dish-values.md` §Formulas·「审查产出的使用方式」/ §Edge Cases / §Open Questions ↔ `menu-waves.md` §Detailed Design·规则 9 / §Formulas·`wave_healthy`
- **原文**: dish-values 规则「菜单波次生成单次菜单时，其 `opt_value` 相对全池校准值的偏差不得超过 **±10%**，否则该次刷新重掷」，并注明「本条是跨系统契约，不归本系统执行」；menu-waves 规则 9「可达最优 ≥ **0.9 ×** `opt_value(pool, C0, T)`（**只卡下界**；上界为防回归断言，由 `dish-values` 处理）」
- **冲突**: dish-values 把上界也交给波次，波次把上界又退回 dish-values，而 dish-values 在 §Open Questions 明说「±10% 菜单重掷属于菜单波次契约，本系统只提供 `opt_value`，不在本 GDD 内断言」。**上界目前没有任何一方执行**；同时「单次刷新菜单」与「确定性波次划分」也不是同一个对象。
- **需要决定**: 波次划分是否继承 ±10%，若是上界落在哪一方；或者 dish-values 正式撤销该契约、只保留 0.9 可达下界。
- **影响面**: `dish-values.md`、`menu-waves.md`、`design/registry/entities.yaml`（`opt_value` / `wave_reachable_opt` 的 notes）

#### 🔴 B2. `preview_dishes` 在三个 GDD 里有两种元数

- **涉及**: `menu-waves.md` 规则 7 ↔ 同文件 §Formulas / `meta-upgrades.md` 规则 14 / `in-game-hud.md` §Dependencies
- **原文**: 规则 7「`preview_dishes = min(menu_vision, wave_size − 1)`」；Formulas「`preview_dishes(run) = min(menu_vision, wave_size − 1, |W_{i+1}|)`」
- **冲突**: 两项式在下一波不足 `wave_size−1` 道时会多报预览卡（首局六道菜池按 4+2 划分时 `W_1` 只有 2 道）。其余三处（Formulas、局外升级规则 14、注册表）都是三项式。HUD 按计数渲染，两种写法会画出不同界面。
- **需要决定**: 统一为三项式，并改 `menu-waves.md` 规则 7 的一句措辞。

#### 🔴 B3. 输入把音效列为下游消费者，音效明确否认

- **涉及**: `input.md` §Dependencies（下游表两行）↔ `audio-feedback.md` §Dependencies·「不属于依赖的关系」/ §Interactions
- **原文**: input「| 音效反馈 | 软 | 动作触发事件（用于 UI 音效）|」；audio「**输入**：本系统不订阅任何原始输入动作。它消费的是界面流程与 HUD 发出的语义事件；把输入列为依赖会把焦点移动、悬停与『按下但未提交』混进 cue 目录」
- **冲突**: 两处不能同时成立。按 input 的表去接线，会得到音效规则 19 明确排除的焦点音与重复 cue。音效的立场与 `ui-flow.md`「没有焦点移动音」一致。
- **建议**: 删除或改写 `input.md` 的两行（音效的表述是唯一自洽的），**但这条决定应在你手上**。

#### 🔴 B4. 非最后一波跳波：一条 AC 要求确认，另外三条要求不确认

- **涉及**: `menu-waves.md` AC 6 ↔ `menu-waves.md` AC 20 / §Open Questions 8 ↔ `in-game-hud.md` 规则 16 / AC 24
- **原文**: AC 6「**WHEN** 点击『下一波』**并确认**，**THEN** 回合计数变为 6」；AC 20「**WHEN** 点击『下一波』，**THEN** 立即提交、不弹二次确认」；OQ8 已答「MVP 不弹二次确认」
- **冲突**: 两条 AC 不可能同时通过。AC 6 是唯一残留的旧措辞。
- **需要决定**: 删除 AC 6 的「并确认」；若确实想为非最后一波加确认，则要同时回退 HUD 规则 16 / AC 24 / OQ8。

### Warnings

| # | 问题 | 涉及 |
|---|------|------|
| W1 | `systems-index.md` 缺 4 条边：胃容量→局外升级（快照）、腻度→数据配置（`cloy_cap`）、音效→纪录与里程碑（事件契约）；且 `audio-feedback.md` 自述「`Depends On` 维持不变（与索引 Dependency Map 一致）」与它自己的上游表矛盾。菜单波次↔腻度的同一条软边在两份文档里方向相反 | index 行 7/9/17、audio、menu-waves、cloying |
| W2 | 首局引导的 `beat_next` 读 `group_count(g)`（腻度状态），却把腻度写在**下游**表；索引行 18 完全没列腻度 | first-run-onboarding、index |
| W3 | `running_net` / `shortfall` 有两个实现者：结算说「本系统定义两个纯函数」，餐局流程说「实时净值由本系统维护」并把它放进快照；HUD 规则 6 署名结算，实际由餐局流程发布。AC 只要求「三处一致」，没说谁是权威 | settlement、run-flow、in-game-hud |
| W4 | 两个互不包含的「菜单可发布」闸门：`menu_healthy`（UV 跨度 / 同组支配 / 档位覆盖）与被引用为发布前提的 `menu_cloy_pressure ≥ 1`；CI 只跑前者 | dish-values、cloying |
| W5 | 首局引导把六道菜的价格 / 饱腹 / 单位价值 / 组重新列表，而 dish-values 明令「所有数值进入 `DishData` 后不得在本表之外重复定义」。任何一次改数值会静默作废首局的四段推导（基础饱腹 20、放弃项 82、跨度、可负担性 42） | dish-values、first-run-onboarding |
| W6 | `data-config` 只校验 `satiety ≥ 1`，`dish-values` 的 1–6 是设计带而非校验；7 点饱腹的菜能通过校验并越出 `effective_satiety`、`can_fit.e`、结算三处声明的值域 | data-config、dish-values、cloying、stomach-capacity、settlement |
| W7 | 波次健康校验固定 `C0 = 20`，但容量满级是 25（上限 120）。玩家实际体验的容量与闸门度量用的容量不是一个，且结算/菜品数值已按 25 重新校准 | menu-waves、meta-upgrades、dish-values、settlement |
| W8 | HUD 声称音效「订阅**焦点移动**、确认、拒绝、跳过意图」，音效与界面流程都明确排除焦点音 | in-game-hud、audio、ui-flow |
| W9 | 陈旧引用 / 状态簇（均已在当前正文核实）：`run-flow.md` §Cross-References 仍把时间与回合、HUD、界面流程、首局引导、音效、存档等 7 篇标为「未写」；run-flow / time-and-turns 的「双向一致核对」仍要求索引把 `Not Started` 改为 `Designed`；HUD 的索引核对描述与实际八项依赖不符；局外升级仍称腻度「公式尚未含抗性项」；多份 §待回改 的要求其实已完成；run-flow 下游契约漏了 `consume` 的 `NONE` 返回值；若干已答 OQ 未标记 | run-flow、time-and-turns、in-game-hud、meta-upgrades、unlocks、first-run-onboarding、settlement、dish-values |
| W10 | 索引 §Circular Dependencies 写「No circular dependencies were found」，但界面流程 ⇄ 音效反馈 是双向的（设置模型 + 事件总线）。作为分层未必是错误，但「无环」这句话按字面不成立 | systems-index、ui-flow、audio |
| W11 | 注册表值域与权威公式不一致：`unit_value [0,9999]` vs dish-values「`[0, price_max]`」；`return_ratio [0,999]` vs 结算「(0, +∞)」；`nav_steps [1,999]` vs 输入「无上界」；`wave_advance_cost` 缺 `added` / `revised` | entities.yaml |
| W12 | `capacity_ceiling(C)` 是与 `opt_value` 同定义的第二个最优解，但未注册、由局外升级拥有。若 dish-values 的 OQ5 改变背包模型，升级价格表会静默沿用旧模型 | meta-upgrades、dish-values、entities.yaml |

---

## 二、游戏设计整体性

### Blocking

#### 🔴 D1. 解腻在静态最优里是免费的，`CLOYING_RESISTANCE`（20 元）因此近乎空买

- **涉及**: `cloying-and-taste.md` §Formulas·「与校准值的关系」↔ `meta-upgrades.md` §MVP 升级目录 ↔ `dish-values.md` §MVP 菜品表
- **原文**: 「红烧肉 → 拍黄瓜（解腻 `肉`）→ 糖醋里脊 → 其余七道任意 … 这条顺序共占 20 格、得分 104，与静态最优完全一致」；局外升级把解腻定价 20 元、估值「3–6 元/局（省下 1 格容量与一次解腻菜）」
- **问题**: 104 元最优餐同时含红烧肉与糖醋里脊（同属 `肉`），而拍黄瓜（5 元/格，本来就在最优餐里）正好夹在中间。于是全池唯一的重复惩罚可以被**零机会成本**清掉。定价假设「解腻要牺牲一次菜」，但熟练玩家不需要牺牲 —— 这是支柱 2「没有白吃的菜」在它自己的执行系统上失效。
- **设计文档自己的判断**: `cloying-and-taste.md` 已承认「在此之前，本系统在 MVP 中的强度按『顺序谜题』评估，不按『资源约束』评估」，并把资源压力押在波次上（每波 4 道）。
- **选项**: (a) 在 `肉` 组放一道非解腻高价值菜，让重复无法干净避开；(b) 把拍黄瓜移出最优餐（降价 / 降单位价值）；(c) 让解腻消耗 1 回合或 1 格容量；(d) 承认 MVP 只卖「顺序题」，把 `CLOYING_RESISTANCE` 降到更便宜的 1 级防御位并改写其估值，或推迟到第二个档口再加。

#### 🔴 D2. 主循环可被算穿，两道防护在 MVP 参数下都不承重

- **涉及**: `menu-waves.md` §Formulas·「菜单健康」/ §Open Questions 4 ↔ `cloying-and-taste.md` OQ1 ↔ `dish-values.md` §Formulas
- **原文**: 「当前参数下阈值偏宽，重掷很少触发 —— 0.9 × 104 = 93.6 的带宽能吸收两次跳过」
- **问题**: 104 元的解已在文档里被算出来并验证。两道防「背答案」的机制都空转：解腻免费（D1），波次健康闸门允许两次跳过（95 ≥ 94），所以几乎每个种子都能拿满 104。 pillars 1 的前提（算术）退化成查表，核心幻想「我一眼看穿菜单」从技能变成记忆。
- **设计文档自己的判断**: 这已登记为 `menu-waves.md` OQ4（是否收紧到 0.95），标注「原型后」。
- **选项**: (a) 阈值收紧到 0.95 且 / 或 `wave_size` 降到 3；(b) 让解腻承担真实代价（同 D1）；(c) 把「防算穿」整体推到 `/prototype` 验证后再定 —— 但这等于承认 MVP 阶段支柱 2 未被证明，属范围决策。

### Warnings

| # | 问题 | 涉及 |
|---|------|------|
| D3 | **入场费是扣分项，不是钱包流出。** 结算规则 4 明说「开局不做钱包扣款」。全树一次性支出只有 209（升级）+ 120（解锁）= 329 元，之后 `net_profit` 无限累积：无经常性流出、无追赶机制。作为「分数」可以接受，但「回本」因此没有经济牙齿 —— 每局 +3 与每局 −31 面对相同的钱包压力 | settlement 规则 4/5、meta-upgrades 规则 2/3 |
| D4 | **档口解锁按最优局配速。** `runs_to_unlock` 用 `opt_net_profit = 19` 得 7 局（落在 4–8 带内），但结算的中位前三局净利是 +3…+11，中位玩家要 11–40 局；第二个档口的菜池与入场费尚未定义，跨档口入场费曲线无法评估 | unlocks §Formulas、settlement 校准带 |
| D5 | **容量成长是跳变的。** 边际收益 +2 / +1 / +5 / +4 / +4：第 2 级（17 元换 +1 元/局）是死等级，与「价格渐增、收益线性」的自述不符。同时 10 回合 = 10 道菜 = 20 格，游戏卡在刀刃上，任何单点内容改动都会破 | meta-upgrades 设计期表、stomach-capacity OQ3 |
| D6 | **注意力预算超载。** 一次「吃 / 跳过」要同时权衡价格、基础饱腹、实际饱腹(+N)、解腻预览、剩余容量、剩余回合、running net / 差额、波次与预览 —— 8 个数字流。首局引导只教四个概念，且从未教真正的决策量 `price / effective_satiety` | in-game-hud 规则 8、first-run-onboarding |

### Info

- ℹ️ **缺可选章节**：18 篇均无 `## Player Fantasy`，仅 `meta-upgrades` / `unlocks-and-stations` 有 `## Tuning Knobs`。在 `workflow: standard` 下两者为可选，不阻塞；支柱对齐改由 Summary / Overview / Game Feel 判定。
- ℹ️ **两个累计器**：纪录的 `lifetime_net_profit` 与钱包是同一 `net_profit` 的两条分叉累计，里程碑 `ms_lifetime_2000` 跟前者而玩家读后者；功能性无碍，玩家可能疑惑「钱包 50，终身 2000」。

---

## 三、跨系统场景问题

**Scenarios walked**: 5 —— 吃一道菜；跳过当前波 / 最后一波「结束取餐」；回合耗尽→结算→纪录→存档→收据→音效；局外购买（升级 / 档口解锁）→存档 commit；首局引导

### Blockers

#### 🔴 S1. 「结束取餐」确认面板打开时，`game_skip_wave` 仍然有效且行为未定义

- **涉及**: `in-game-hud.md` 规则 16 ↔ `input.md` §States and Transitions
- **步骤**: 最后一波打开确认面板后，玩家再按 S / X / 右键
- **原文**: HUD「面板**不是**输入上下文意义上的 `Modal`……打开时导航限制在面板内，`ui_cancel` 关闭并把焦点还给原按钮」；input「Run | 一局进行中且无弹窗 | 导航、`ui_accept`、`game_skip_wave`、`ui_pause`」
- **失败类型**: **未定义行为**。面板不压 `Modal`，`game_skip_wave` 仍活跃；规则没说第二次按下是被忽略、重开面板还是当作确认。连点可能绕过或重复触发 AC 25 要保证的那次确认。
- **需要决定**: HUD 在面板打开时吞掉 `game_skip_wave`（忽略）/ 视为确认 / 或允许面板 `push_context(Modal)`（规则 16 目前禁止）。补一条 AC。

#### 🔴 S2. 结算失败走 `init_failed` 通道，玩家被告知「本局未能开始」

- **涉及**: `run-flow.md` 规则 18 ↔ `ui-flow.md` 规则 19
- **步骤**: `settle()` 返回结构化错误 → `Faulted` → `Run → Menu (init_failed + fault)`
- **原文**: run-flow「通过当前唯一可用的无结果返回（`Run → Menu`，`init_failed = true` 附带 `fault` 明细）」，并注明「复用 `init_failed` 是暂定方案」；ui-flow「因 `init_failed = true` 或 `fault` 明细进入时，显示非模态提示『**本局未能开始**』」
- **失败类型**: **矛盾反馈**。打完一整局、HUD 已经展示过全程的玩家，会在主菜单被告知这局没有开始。
- **需要决定**: 为「局后故障」定义独立载荷与文案，或至少让 ui-flow 按是否存在 `RunResult` 分支。

#### 🔴 S3. `is_first_run` 有两个来源，跳过后再放弃时结论相反

- **涉及**: `ui-flow.md` 规则 8 / §Interactions ↔ `first-run-onboarding.md` 规则 15 / 22
- **原文**: ui-flow「`is_first_run = 纪录与里程碑.snapshot().career.runs_settled == 0`」，但同文件 §Interactions 又写「订阅 `onboarding_finished` 决定下一次 `is_first_run`」；首局引导规则 15「完成不写任何持久化；下一局还算不算首局由组装方决定（推荐：`runs_settled == 0`）」，规则 22「首局被放弃时不结算、不入账，『下一局仍是首局』是合法结果」
- **失败类型**: **所有权冲突 / 未定义行为**。跳过会广播 `onboarding_finished(SKIPPED)`，随后放弃不结算；按 `runs_settled` 下一局仍是首局，按信号处理则教学被静默丢弃。
- **需要决定**: 以 `runs_settled` 为唯一判据并删掉信号的判定权，或规定信号处理必须回落到 `runs_settled`。

### Warnings

| # | 场景 | 问题 |
|---|------|------|
| S4 | 跳过当前波 | `turns_remaining = 1` 时付费跳波合法，会白看一帧无法取菜的新波，并先放 `sfx.wave.skip` 再放 `sfx.run.end`。时间与回合 OQ1 提议在 1 回合时禁用，未决 |
| S5 | 回合耗尽→收据 | 纪录 `prepare_run` 失败时界面流程整笔放弃、收据标注「未入账、未记录」，但音效仍按 `RunResult` 播 Critical 的 `broke_even` —— 成功音 + 失败界面。只有 `run_faulted` 会取消结果 cue |
| S6 | 回合耗尽→收据 | `settlement_started` 无人负责发出：结算只说「只定义语义」，音效要求「收据打开同帧」，界面流程发的是 `receipt_opened`。若由结算在 `settle()` 发，暂停会先播打印音再冻结 |
| S7 | 局外购买 | `upgrade_price()` 需要 `UpgradeData.cost_growth`，但数据配置尚未声明该字段（局外升级 §待回改 2 已提出）。在此之前价格不可计算 |
| S8 | 档口解锁 | `station_banquet`（120 元）若随数据发布，可购买但主菜单无选择器（ui-flow 规则 8 固定 `station_canteen`）—— 购买即空操作 |
| S9 | 首局引导 | 主菜单按规则 8 显示本档口入场费 85，实际扣 40；差异只在进店后的 B0 与收据上解释 |
| S10 | 首局引导 | 玩家照设计最优放弃麻婆豆腐后，B4 的 `replacement_hint` 会把这道菜标为「最亏」并建议「下一局先算一算再拿」—— 教学把正确答案说成错误 |
| S11 | 首局引导 | `Recovered` 载入若备份早于首次结算，会把老生涯的 `runs_settled` 恢复成 0，重新触发六道菜 / 40 元教学 |

### Info

- ℹ️ `input.md` §Interactions 写餐局流程读取 `ui_pause`，`run-flow.md` 明确「不读取 `ui_pause`」。改一行。
- ℹ️ `input.md` 把 `Blocked` 分给「结算动画」，但没有任何系统为 `Ending` / `Settling` / `Handoff` 请求它，HUD 规则 21 又说不调 `request_context` —— 该上下文无主。
- ℹ️ `run-flow.md` 规则 16 对任何非交接离开（含「退出游戏」）都丢弃本局；UI 需求 9 已删掉「放弃本局」以防「已算完但被放弃」，退出按钮重新打开了同一扇窗。
- ℹ️ 局外升级规则 7 用 `NOT_IN_MENU`（顶层 `Menu`）设闸，但收据屏也属于 `Menu`；两者没有可区分的接口，只靠屏幕结构保证不可达。
- ℹ️ 首局引导 `Closing` 等 `run_settled` 后交付 `FirstRunExplanation`，界面流程规则 9 在同一帧打开收据 —— 两者先后未定义。

---

## 四、被标记需修订的 GDD

| GDD | 原因 | 类型 | 优先级 |
|-----|------|------|--------|
| `menu-waves.md` | B1 上界契约无人执行；B2 `preview_dishes` 元数；B4 AC 6 与 AC 20 互斥；D2 阈值过宽；S1 面板输入 | 一致性 / 设计 / 场景 | Blocking |
| `input.md` | B3 音效依赖声明与音效 GDD 相反；S1 面板期动作；Info 两条 | 一致性 / 场景 | Blocking |
| `dish-values.md` | B1 ±10% 契约归属；W5 数值重复定义；W6 饱腹值域 | 一致性 | Blocking |
| `first-run-onboarding.md` | S3 `is_first_run` 判定权；S10 教学结论与设计最优相反；W2 腻度方向；W5 重列数值 | 场景 / 一致性 | Blocking |
| `ui-flow.md` | S2 局后故障文案；S3 订阅与判据冲突 | 场景 | Blocking |
| `run-flow.md` | S2 复用 `init_failed`；W3 `running_net` 归属；W9 陈旧引用簇 | 场景 / 一致性 | Blocking |
| `in-game-hud.md` | B2 预览计数；B4 与 AC 24 的确认口径；S1 面板期输入；W8 焦点音 | 一致性 / 场景 | Blocking |
| `cloying-and-taste.md` | D1 解腻免费；D2 防算穿失效 | 设计 | Blocking |
| `meta-upgrades.md` | D1 解腻定价假设；D5 容量边际跳变；W12 `capacity_ceiling` | 设计 / 一致性 | Blocking |
| `audio-feedback.md` | B3 需要与输入的表述对齐（音效一方自洽，改动可能落在 input） | 一致性 | Blocking |
| `settlement-and-entry-fee.md` | W3 `running_net` 归属；D3 入场费非钱包流出 | 一致性 / 设计 | Warning |
| `systems-index.md` | W1 四条缺边；W10「无环」表述 | 一致性 | Warning |
| `unlocks-and-stations.md` | D4 按最优配速；S8 可买不可选 | 设计 / 场景 | Warning |
| `stomach-capacity.md` | W6 值域；D5 与容量曲线相关 | 一致性 | Warning |
| `data-config.md` | W6 饱腹校验；S7 `cost_growth` / `wave_size` 字段 | 一致性 | Warning |
| `time-and-turns.md` | W9 状态陈旧；S4 最后一回合跳波 | 一致性 / 场景 | Warning |
| `save-system.md` | S11 `Recovered` 重新触发教学 | 场景 | Warning |
| `records-and-milestones.md` | S5 中止结算的音效抑制路径 | 场景 | Warning |

---

## 五、判定

### Verdict: **FAIL**

9 项 blocking，24 项 warning，7 项 info。

FAIL 的依据是 blocking 存在，但不全是同一类活：

- **B1–B4 + S1–S3：契约/表述类（7 条）** —— 逐条改一段措辞或补一条 AC 即可闭合，不需要重新设计。建议在进入架构前一次改完。
- **D1–D2：设计类（2 条）** —— 需要一次取舍决策，可能要先跑 `/prototype`。它们不阻塞 Godot 场景结构或 ADR，但决定 MVP 是否成立，建议与架构并行推进而不是串行等待。

### 进入下一阶段前必须完成

1. 统一 `preview_dishes` 为三项式（`menu-waves.md` 规则 7）。
2. 删除 `menu-waves.md` AC 6 的「并确认」，或回退 HUD 规则 16 / AC 24 / OQ8。
3. 决定 ±10% 重掷契约的归属，并让 dish-values 与 menu-waves 只保留一份表述。
4. 修正 `input.md` 的音效下游两行，并补「确认面板打开时 `game_skip_wave` 如何处理」。
5. 为「局后故障」定义独立载荷与文案，不再复用 `init_failed`。
6. 把 `is_first_run` 的判据收敛到 `runs_settled`，删除信号的判定权。
7. 对 D1 / D2 各做一次明确决策（改数值 / 改波次参数 / 推迟到原型验证），并写回两份 GDD。

---

## 六、未能评估的部分

- **2f 未覆盖的 AC 区间**：`ui-flow.md` §Acceptance Criteria、`audio-feedback.md` §Acceptance Criteria、`in-game-hud.md` AC 1–15、`records-and-milestones.md` §Detailed Design、`scene-flow.md` §Detailed Design —— 子代理在轮次预算内未读到这些段落。若冲突只存在于这些区间，本次不会发现。
- **跨档口入场费曲线**：`station_banquet` 只有 `unlock_cost = 120`，没有菜池、`entry_fee`、`cloy_cap`，因此「入场费是否跑赢每档可赚价值」无法判定。
- **`design/ux/*` 与 `design/accessibility-requirements.md`**：18 篇 GDD 都把它们标为待创建，本次不适用。
- **可玩性结论**：D1/D2 是文档内可证的数值结论；它们是否真的让玩家觉得「被算穿」，只能由 `/prototype` 回答。

---

## 七、修订记录 — 2026-11-04（契约类闭合）

七条契约 / 场景类 blocking 已在同一日修完，**只改正文措辞与接口声明，未改任何平衡数值**。

| ID | 修法 | 文件 |
|----|------|------|
| B1 | dish-values 撤销逐划分的 ±10% 双侧重掷契约，明确运行期只有「菜单波次」的 0.9 单侧下界；整池 `opt_value` 回归留给 CI 断言。三处措辞（设计期 / Edge Case / 非目标）与 menu-waves 交叉引用表同步 | `dish-values.md`、`menu-waves.md` |
| B2 | 规则 7 改为三项式 `min(menu_vision, wave_size − 1, \\|W_{i+1}\\|)`，并写明第三项是硬上限 | `menu-waves.md` |
| B4 | AC 6 删除「并确认」；最后一波 AC 7 的确认保留（那里确实要确认） | `menu-waves.md` |
| B3 | input 的下游表两行改为「无直接依赖 / 不向音效发送任何事件」，视觉音频节同步；音效 GDD 未改 | `input.md` |
| S1 | HUD 规则 16 增加「面板打开期间 `game_skip_wave` 被吞掉，不重开、不当作确认」；新增 AC 27，原 27–41 顺延为 28–42 | `in-game-hud.md` |
| S2 | `fault` 载荷新增 `phase ∈ {INIT, RUN}`；ui-flow 规则 19 / 转移表 / Edge Case 按 phase 分支文案，`RUN` 显示「本局结算失败，收益不计」。run-flow OQ1 标记已解决 | `run-flow.md`、`ui-flow.md` |
| S3 | `is_first_run` 收敛到 `records.career.runs_settled == 0` 一个判据；ui-flow 与首局引导的四张表 / 规则 15 全部移除「由 `onboarding_finished` 决定」 | `ui-flow.md`、`first-run-onboarding.md` |

**附带修掉的 Info**：input 的餐局流程行不再列 `ui_pause`（run-flow OQ2 标记已解决）。

**修订后的判定**：仍为 **FAIL**，但 blocking 从 9 条降到 2 条 —— 只剩 D1（解腻免费 / `CLOYING_RESISTANCE` 空买）与 D2（0.9 阈值过宽、主循环可算穿）。索引中 `腻度与口味`、`菜单波次`、`局外升级` 保持 `Needs Revision`，其余六篇恢复 `Approved`。

**未做的事**：24 条 warning 全部未动；没有提交。
