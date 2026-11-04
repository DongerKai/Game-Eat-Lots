# 时间与回合（time-and-turns.md）— 设计评审日志

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none (lean — Phase 3b not run)
Blocking items: 1 | Recommended: 12
Summary: 设计本体是扎实的：回合作为离散动作计数、单一调用者、不钳制/不兜底、只报告不裁决这四条边界互相闭合，四条数值规则（预算、单次消耗、已用、耗尽）与 餐局流程 / 菜单波次 / 结算 / 局内 HUD 的接口逐项对得上，支柱 3「时间在走，手不用快」与反支柱「NOT 实时操作」都在本 GDD 有对应硬规则。唯一的 blocking 项不是设计缺陷，而是 Cross-References 与「待回指」清单的陈旧自相矛盾——同一份文档对 局内 HUD / 局外升级 / 音效反馈 给出「未写」与「已写」两种陈述，且五项已落地的跨文档改动仍被写成待办。其余 12 条为跨文档接口归属、`run_ended` 后读数语义、两处 AC 可测性与三处文档口径同步。
Prior verdict resolved: First review

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。
Formulas 对本系统是**必需**的：它定义回合预算、固定消耗 1、已用恒等式与耗尽判据四条规则，并给出 `turn_budget` 的 1–INT32_MAX 取值域与 10±1 可调带——按 standard 层不能清为 advisory。

Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs。Game Feel 一节实质覆盖了 Player Fantasy 的内容（即时、不焦虑、可归因）；Formulas 的「可调带（设计期）」覆盖了 Tuning Knobs 的唯一可调值 `turn_budget`。补一行互相指向即可，不要求新增整节。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 餐局流程 | 上游（调用者，硬） | `design/gdd/run-flow.md` | ✓ 存在；第 117/290 行回指本系统，状态 `GDD 已写` |
| 结算与入场费 | 下游（硬） | `design/gdd/settlement-and-entry-fee.md` | ✓ 存在；第 327 行回指本系统 |
| 局内 HUD | 下游（硬） | `design/gdd/in-game-hud.md` | ✓ 存在；第 165/384 行回指本系统 |
| 菜单波次 | 间接（硬，经餐局流程） | `design/gdd/menu-waves.md` | ✓ 存在；收 `T` 参数、不直接调用本系统 |
| 菜品数值 | 下游（软，仅设计期） | `design/gdd/dish-values.md` | ✓ 存在；第 270 行回指本系统 |
| 数据配置 | 数据所有者，非运行时依赖 | `design/gdd/data-config.md` | ✓ 存在；第 184 行回指本系统 |
| 局外升级 | 无直接交互 | `design/gdd/meta-upgrades.md` | ✓ 存在（本 GDD 将其记为「未写」，见 blocking 项） |
| 存档 | 无直接交互 | `design/gdd/save-system.md` | ✓ 存在 |
| 输入 | 无直接交互 | `design/gdd/input.md` | ✓ 存在 |
| 音效反馈 | 无直接交互 | `design/gdd/audio-feedback.md` | ✓ 存在（本 GDD 将其记为「未写」，见 blocking 项） |
| 胃容量 | 同级，不构成依赖 | `design/gdd/stomach-capacity.md` | ✓ 存在 |

无断链。索引记录 `时间与回合 → 餐局流程`，与 §Dependencies 的「按生命周期所有权记录方向」说明一致，且与餐局流程 GDD 采用同一理由，未形成环。

### Registry Values

`design/registry/entities.yaml` 共 78 条命名条目。按 Phase 3 用全部条目名（`_` 展开为 `[_ ]`、大小写不敏感）对本 GDD 全文扫描：命中 14 行，逐行核对后**无冲突**。涉及本系统的条目与取值：

| 条目 | 注册表值 | 本 GDD | 结论 |
|---|---|---|---|
| `station_canteen_turn_budget` | `10` 回合，安全带 10±1，改值须重跑 `opt_value` / `calibration_ratio` | 第 102/104/109 行同 | ✓ |
| `base_capacity` | `20` 格（`C0`） | 第 23/102 行 `C0 = 20` | ✓ |
| `transition_turn_cost` | `0`（内部转移不扣回合） | 第 142 行、自动推进与「结束取餐」0 回合 | ✓ |
| `wave_advance_cost` | `1`（付费跳波） | 第 142 行「跳一次非最后一波各调用一次」 | ✓ |
| `end_reason` | `source: run-flow.md`，`TIME_EXHAUSTED` 优先 | 第 43/177/288 行同（本系统不保存、不排序） | ✓ |
| `opt_value` / `calibration_ratio` | `M, C, T` 为变量；校准带 `[1.15, 1.35]` | 第 75/106/210 行同 | ✓ |

`station_canteen_turn_budget` 的 `referenced_by` 列 dish-values / menu-waves 而未列结算，与本 GDD「结算只取 `RunLog.turns_total` / `turns_used`、不读预算常量」的边界一致，不属于缺漏。

### Findings

- [BLOCKING] Cross-References（第 249–253 行）与「待回指」（第 254–262 行）：`局内 HUD`、`局外升级`、`音效反馈` 三行仍标为「未写」，但三份 GDD 均已存在于磁盘，且同一文档的 Dependencies 表（第 197 行）对局内 HUD 标为「GDD 已写（design/gdd/in-game-hud.md）」——同一文档对同一事实给出两个相反陈述。同时「待回指」六项中第 1、2、3、4、6 项**已经落地**：`run-flow.md:290` 与 §Cross-References 已指向本文件；`settlement-and-entry-fee.md:327` 已是「GDD 已写」；`dish-values.md:270` 已补「来自 `StationData.turn_budget`」；`data-config.md:52/98/115` 已定义 `turn_budget` 并纳入 `R_sta`；`systems-index.md:62` 第 6 行已是 `Designed` 且文档路径正确。§Dependencies 第 216 行仍称「数据配置缺口：需要新增 turn_budget」、第 248 行仍写「字段（需新增）」，均与现状相反。**唯一仍未完成的是第 5 项**（`menu-waves.md:263` 的「本系统只发起扣除」措辞）。修法：①三行改为实际路径；②「待回指」只保留 menu-waves 一条；③删除第 216 行与第 248 行的「需新增」。这与 `scene-flow-review-log.md:43` 阻塞的缺陷同类，属同一份文档的自相矛盾，故阻塞。

- [RECOMMENDED] Dependencies §双向一致核对（第 214 行）：「餐局流程（下游表，接口逐字一致）」不成立。`run-flow.md:117/290` 只列 `initialize(...)`、`spend_turn()`、`turns_remaining`、`turns_used` 与「`TIME_EXHAUSTED` 判据」，没有本 GDD 规则 7 要求调用方使用的 `can_spend_turn()`，也没有 `is_exhausted()`。本 GDD 的接口是超集、方向正确，需改的是餐局流程那两行（Owner：#5）。在此之下，本句应改为「超集，`can_spend_turn()` / `is_exhausted()` 待 #5 回填」。
- [RECOMMENDED] Dependencies / 结算行（第 72/196 行 vs 外部）：本 GDD 正确声明不拥有 `end_reason`，但 `settlement-and-entry-fee.md:64/98/327` 把 `end_reason` 列为本系统提供的接口，注册表 `end_reason` 的 `source` 是 `run-flow.md`。结算的上游表应改为「`turns_total` / `turns_used`」，`end_reason` 指向餐局流程（Owner：#11）。本 GDD 本身无误，不阻塞本评审。
- [RECOMMENDED] States and Transitions（第 58 行）/ 规则 12：`Active` 定义为「本局进行中」，但本局以 `NO_EATABLE_DISH` 结束且仍有剩余回合时，系统没有离开 `Active` 的路径（只有耗尽或下一次 `initialize()`），于是「本局已结束」与「状态 = Active」同时为真。同理 `can_spend_turn()` 在此刻仍会返回 `true`，而规则 12 只说「数值冻结、不再属于当前局读数」。建议把 `Active` 的定义改为「已初始化且未耗尽」，并明确「`run_ended` 之后 `can_spend_turn()` 不再有意义 / 返回 `false`」，否则调用方可能从读数推断出一个仍在进行的局。
- [RECOMMENDED] Edge Cases 末第三条（第 178 行）：「观测到 `turns_used > turns_total` 或 `turns_remaining < 0`：内部状态已损坏，返回结构化错误并停止消耗」——本系统对外只有 `turns_remaining` / `turns_used` / `turns_total` 三个 int 读数，没有能承载「结构化错误」的通道；错误词表也只覆盖 `initialize` 与 `spend_turn`。建议明确为：`spend_turn()` 返回新增的 `STATE_CORRUPTED`，读数保持原值，并以一个只读 `faulted` 标志对外暴露；或删去「返回结构化错误」，改为「停止消耗并记录 ERROR」。
- [RECOMMENDED] Acceptance Criteria 3 / Edge Cases（第 167 行）与接口签名（第 71 行）：接口声明为 `initialize(turn_budget: int)`，因此「`turn_budget` 缺失或不是整数」不是该签名能接收的输入，AC 3 按现状无法写成针对本系统 API 的测试；而 `data-config.md:115` 与 AC 26 已在加载期拒绝非整数。建议二选一：AC 3 只保留 `turn_budget < 1` 的整数夹具，把类型错误归数据配置；或把参数声明为可空/`Variant` 并显式断言类型，再保留本 AC。
- [RECOMMENDED] 规则 6（第 39 行）：「…然后才广播 `action_committed` 并写入 `RunLog`」读起来是每次动作写一行日志；但 `run-flow.md:40` 与 AC 14 规定 `RunLog` 在 `Ending` 组装一次、字段值取自动作发生时刻。建议改为「…并记录该时刻的回合值，供 `Ending` 组装 `RunLog`」，避免实现者做出第二套日志模型。
- [RECOMMENDED] Acceptance Criteria 21（第 292 行）：判据的 THEN 是「设计期审计应报告『回合不承重』」，而该审计的归属正是 Open Question 5（Owner：菜品数值 + 编排会话，尚未落定），今天不存在可运行的规则，因此这条 AC 无法独立判定 pass/fail。建议二选一：指向 `dish-values.md` 已存在的平衡审查并写成具体阈值（例如「最优解道数 < `turns_total` 时报告 warning」），或把本条从 AC 移回 OQ 5。
- [RECOMMENDED] Formulas / Acceptance Criteria 覆盖（第 86、266 行）：`can_spend_turn()` 是规则 7 与接口表（第 71 行）公开的成员，但四条数值规则里没有它，AC 覆盖表也没有它；现有判据（AC 8/9）只覆盖「耗尽后为 false」。建议在 Formulas 补一行 `can_spend_turn = initialized ∧ turns_remaining ≥ 1`（纯读取、无副作用），并补一条 AC：`Uninitialized` 下返回 `false` 且调用前后三个读数不变。
- [RECOMMENDED] Dependencies / 数据配置（第 76/205 行 vs 外部）：本 GDD 把这条边定为「软（间接）、非运行时依赖」，而 `data-config.md:184` 把同一行标为「硬」并仍写「新增字段」。两个口径需统一（建议按本 GDD 的「软/设计期」），并删去 data-config 中已过期的「新增」（Owner：#1）。
- [RECOMMENDED] 局内 HUD 数值域（跨文档）：`in-game-hud.md:242` 的 `hud_turn_slots` 把 `turns_total` 的输入范围写成 `1–999`，本 GDD 与 `data-config.md:115` 都是 `1–INT32_MAX`。HUD 自己的边界情形（`> 12` 改画比例条）已经能处理大值，建议把 HUD 声明的范围放宽到 `1–INT32_MAX`（Owner：#15）。同一处可顺带闭合 Open Question 4：HUD 的槽位/比例条分界是 **12**，不是 OQ 4 里写的「超过 20」，按 12 记录即可。
- [RECOMMENDED] Open Question 6（第 302 行）已复核仍在：`menu-waves.md:263` 仍写「本系统只发起扣除」。行为与语义兼容（发起意图 ≠ 执行扣减），但这是实现时出现第二个调用者的入口；一行措辞修复应跟进（Owner：#10）。
- [RECOMMENDED] 零回合预算的归属（跨文档）：`run-flow.md:247`、AC 13、AC 37 断言「初始化后 `turns_total = 0` → 立即 `TIME_EXHAUSTED`，生成零行 `RunLog` 并正常结算」，而本 GDD 的 AC 2 断言同一份非法数据（`turn_budget = 0`）经餐局流程开局应得到 `init_failed`、不产生 `RunLog`。餐局流程把它标为「防御性判定」却仍写成可测的 AC，需要明确：该分支只允许用桩驱动（`turns_total = 0` 由测试替身返回），还是改用 `init_failed`（Owner：#5）。本 GDD 自身一致（第 166 行已说明该分支不可达），不阻塞本评审。

- [NICE-TO-HAVE] Open Question 5 / 首局：`first-run-onboarding.md:455` 的首局是六道菜、`turns_total = 10` 的两波局，回合在首局**永远不承重**（最多吃 6 道）。若「回合不承重」审计最终落在菜品数值，需显式豁免首局档口，否则每次首局都会触发告警。
- [NICE-TO-HAVE] Rules 11（不发信号）：规则 11 是局内 HUD 与音效反馈的接口前提，但没有任何判据断言它。可补一条「本系统对外不暴露 `turn_*` 信号」的 AC（反射遍历信号表），成本很低。
- [NICE-TO-HAVE] Advisory 章节：补一行 Player Fantasy（指向 Game Feel）与一行 Tuning Knobs（指向 Formulas 的 10±1 可调带），即满足 standard 层的 advisory 标准，不必新增内容。

### 已核对且无问题的跨文档接口（记录以免后续重复检查）

- 初始化与结束判定时机：本 GDD 规则 8「初始化后一次 + 每次成功提交后一次」= `run-flow.md:37`（规则 9）；动作校验顺序「回合仍可用」在 `run-flow.md:34/35` 有对应。
- 0 成本动作：自动推进与最后一波「结束取餐」不扣回合 = `run-flow.md:35`（规则 7）与 `transition_turn_cost = 0`；恒等式 `turns_used = 吃菜数 + 付费跳波数` 与 `menu-waves.md:33` 的 `eats + skips ≤ T` 一致。
- AC 20 的算术成立：3 波 × 4 道（`wave_size_default = 4`，`menu-waves.md:91`）下，先吃再付费离开当前波（`menu-waves.md:11`「永久失去剩余菜」）允许「8 道 + 2 次跳过 = 10 回合」，`eats ≤ T − skips` 的上界也成立——该条不是错误夹具。
- 结算不变式 4：`n ≤ turns_used ≤ turns_total` 的下界来源（跳过让 `turns_used > n`）与 `settlement-and-entry-fee.md:275` 一致。
- 局内 HUD：同帧更新、只读快照、不自推耗尽、无倒计时/秒数 = `in-game-hud.md:35/37/51/61`（规则 4/5/12/17），HUD 的不变式 `turns_used = turns_total − turns_remaining`（第 328 行）与本 GDD 的公式一致。
- 首局：`first-run-onboarding.md:455` 的 `turns_total = 10` 与 `station_canteen.turn_budget = 10` 一致；B0 文案「10 回合」同值。
- 支柱引用：`game-concept.md:141–163` 核对，支柱 1「每口都是算术题」、支柱 3「时间在走，手不用快」、支柱 4「数字看得见地涨」编号与措辞全部正确；反支柱「NOT 实时操作」与规则 2 的硬边界一致。

### Verdict

**NEEDS REVISION**（1 条 blocking、12 条 recommended）。blocking 项是 Cross-References 与「待回指」清单的陈旧自相矛盾，属局部文字修复；设计本体与既有 GDD 的接口没有发现相反规则，修完 blocking 即可进入复审。

- **建议 systems-index 状态**：`Needs Revision`（当前为 `Designed`）。本会话按编排约束**未修改** `design/gdd/systems-index.md`，由编排会话统一写入。
- **本 GDD 文件未被修改**：本次只评审，所有修复留待复审前由作者或编排会话执行。

Reviewed-Content-Hash: design/gdd/time-and-turns.md 7f55a619613d6c2ff9f99b9458e39d0f5f48895c
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 8f9bdb61bf682a7e4d7191933202f9efb3d68cde
Reviewed-Content-Hash: design/gdd/run-flow.md 02d37ce06fa293e6b76cb22277a68b705e413448
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 0f58ea62a2e8fa9ace2effe652cc644acab89d5b
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md bcde83821bc73ad7c67a24d38a8659ce2d53693b
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 0 | Recommended: 15
Summary: 上一轮唯一的 blocking 是 Cross-References 与「待回指」清单的陈旧自相矛盾；`ccdd0d7` 按上一轮给出的三条修法全部落地——三行改为实际 GDD 路径、全文已无「未写」字样、「待回指」只保留 menu-waves 一条、第 216/248 行的「需新增」已删除。设计本体未变，四条数值规则与注册表、餐局流程 / 菜单波次 / 结算 / 局内 HUD 的接口仍然咬合；本轮未发现新的设计冲突或数值冲突。剩余 15 条均为跨文档口径同步、`run_ended` 后读数语义、两处 AC 可测性与若干已过期的状态标签，均不阻塞。
Prior verdict resolved: Yes

### 上一轮 blocking 复核（逐条）

| 上一轮 blocking 的修法 | 现状 | 结论 |
|---|---|---|
| ① Cross-References 三行（局内 HUD / 局外升级 / 音效反馈）改为实际路径 | 第 249 行 `design/gdd/in-game-hud.md`、第 250 行 `design/gdd/meta-upgrades.md`、第 253 行 `design/gdd/audio-feedback.md`；全文 `grep 未写` 无命中 | ✓ 已解决 |
| ②「待回指」只保留 menu-waves 一条 | 第 255 行只剩 menu-waves 的「本系统只发起扣除」措辞项 | ✓ 已解决 |
| ③ 删除第 216/248 行的「需新增」 | 第 216 行改为「数据配置已定义…已在 data-config.md 声明并纳入加载期校验」；第 248 行改为「数据所有者：`StationData.turn_budget` 字段」 | ✓ 已解决 |

补充核对：上一轮把「`run-flow.md` §Cross-References 已指向本文件」列为已落地项，该断言当时不成立。`git blame` 显示 `run-flow.md:365` 的 `| 时间与回合 | 未写 |` 自 `3a83c1f`（run-flow 首次提交）以来从未修改，`git log -S` 也无后续提交触碰该行。这不影响本 GDD 的 blocking 结论（本文件自身已无「未写」），但第 255 行新写的「其余回指均已完成」因此并不完全成立，见 Recommended 1。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统仍是必需章节：它定义回合预算、固定消耗 1、已用恒等式与耗尽判据四条规则，并给出 `turn_budget` 的 1–INT32_MAX 取值域与 10±1 可调带。
Advisory 缺口（不阻塞，与上一轮相同）：Player Fantasy、Tuning Knobs。Game Feel 一节覆盖 Player Fantasy 的实质内容；Formulas 的「可调带（设计期）」覆盖 Tuning Knobs 的唯一可调值 `turn_budget`。补两行互相指向即可，不要求新增整节。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 餐局流程 | 上游（调用者，硬） | `design/gdd/run-flow.md` | ✓ 存在；第 291 行 §Dependencies 回指本系统。§Cross-References 第 365 行仍标「未写」（run-flow 自身矛盾，见 Recommended 1） |
| 结算与入场费 | 下游（硬） | `design/gdd/settlement-and-entry-fee.md` | ✓ 存在；第 328 行回指本系统；其第 65 行已把 `end_reason` 来源改为餐局流程 |
| 局内 HUD | 下游（硬） | `design/gdd/in-game-hud.md` | ✓ 存在；第 165/384/450 行回指本系统 |
| 菜单波次 | 间接（硬，经餐局流程） | `design/gdd/menu-waves.md` | ✓ 存在；第 164 行把 `T` 的来源列为本系统，不直接调用 |
| 菜品数值 | 下游（软，仅设计期） | `design/gdd/dish-values.md` | ✓ 存在；第 270 行回指本系统，状态 `Approved` 与索引一致 |
| 数据配置 | 数据所有者，非运行时依赖 | `design/gdd/data-config.md` | ✓ 存在；第 184 行回指本系统（该行自身的「硬 / 新增字段」仍过期，见 Recommended 6） |
| 局外升级 | 无直接交互 | `design/gdd/meta-upgrades.md` | ✓ 存在 |
| 存档 | 无直接交互 | `design/gdd/save-system.md` | ✓ 存在 |
| 输入 | 无直接交互 | `design/gdd/input.md` | ✓ 存在 |
| 音效反馈 | 无直接交互 | `design/gdd/audio-feedback.md` | ✓ 存在 |
| 胃容量 | 同级，不构成依赖 | `design/gdd/stomach-capacity.md` | ✓ 存在 |

无断链。`ccdd0d7` 让餐局流程在开局读取 `StationData.turn_budget` 后以整数传入（`run-flow.md` 规则 22、第 118/291 行），与本 GDD 规则 3「本系统不查询数据目录」一致，「时间与回合 → 数据配置」这条边确实没有新增。

### Registry Values

`design/registry/entities.yaml` 共 78 条命名条目。用全部条目名（`_` 展开为 `[_ ]`、大小写不敏感）扫描本 GDD 全文，命中 14 行，逐行核对后**无冲突**：

| 条目 | 注册表值 | 本 GDD | 结论 |
|---|---|---|---|
| `station_canteen_turn_budget` | `10` 回合，10±1 带，改值须重跑 `opt_value` / `calibration_ratio` | 第 102/104/109 行同 | ✓ |
| `base_capacity` | `20` 格（`C0`） | 第 23/102 行 `C0 = 20` | ✓ |
| `transition_turn_cost` | `0` | 第 142 行、自动推进与「结束取餐」0 回合 | ✓ |
| `wave_advance_cost` | `1` | 第 142 行 | ✓ |
| `end_reason` | `source: run-flow.md`，`TIME_EXHAUSTED` 优先 | 第 43/177/281 行同（本系统不保存、不排序） | ✓ |
| `opt_value` | `T` 为变量；12 道 / `C=20` / `T=10` → 104 | 第 75/102/199 行；最优餐 10 道 20 格 | ✓ |
| `calibration_ratio` | 校准带 `[1.15, 1.35]` | 第 106 行「跌破 1.15」 | ✓ |

`ccdd0d7` 对注册表的改动只涉及 `save_load_decision`（新增 `ReadOnly` 输出），与本系统无关。`station_canteen_turn_budget.referenced_by` 未列结算，与本 GDD「结算只取 `RunLog` 值」的边界一致。

### Findings

- [RECOMMENDED] Dependencies §双向一致核对（第 214、255 行）：第 255 行「其余回指均已完成」不成立——`run-flow.md:365` §Cross-References 仍为 `| 时间与回合 | 未写 |`（`git blame` 显示自 `3a83c1f` 未改）；第 214 行的「餐局流程（下游表，接口逐字一致）」同样不成立，`run-flow.md` 第 118/291 行的接口只列 `initialize(turn_budget) → turns_total`、`spend_turn()`、`turns_remaining`、`turns_used` 与 `TIME_EXHAUSTED` 判据，缺本 GDD 规则 7 所依赖的 `can_spend_turn()` 与 `is_exhausted()`。修法：本文件把这两句收窄为「各自 §Dependencies 已回指；run-flow §Cross-References 仍标未写、接口为超集，待 #5 回填」；`run-flow.md:365` 的改动归 #5。Owner：#5 + 本文件。
- [RECOMMENDED] Dependencies 状态标签过期（第 190、215 行）：索引第 6 行现为 `In Review` 且 `Design Doc` 路径正确，但第 215 行仍写「`Status` 应由 `Not Started` 改为 `Designed`」；第 190 行把餐局流程标为「GDD 已写（Designed）」，而索引第 5 行现为 `In Review`（`ccdd0d7` 同批修改）。建议第 215 行删除（路径已正确），第 190 行改为 `GDD 已写（In Review）`。
- [RECOMMENDED] 餐局流程接口表（第 71 行 vs `run-flow.md:118/291`）：本 GDD 向调用方公开 `can_spend_turn()` 与 `is_exhausted()`，规则 7 还要求调用方先用前者再提交；餐局流程的依赖行没有这两个成员（`ccdd0d7` 新增的 `submit_action` 拒绝码里有 `NO_TURNS`，语义等价但没有声明该读取接口）。本 GDD 自身正确，需 #5 回填两行。Owner：#5。
- [RECOMMENDED] 结算上游表（`settlement-and-entry-fee.md:99/328`）：字段表第 65 行已把 `end_reason` 来源改为餐局流程（✓），但第 99/328 行的上游表仍把 `end_reason` 列在「时间与回合」名下。注册表 `end_reason.source` 与 `run-flow.md` 都指向餐局流程，本 GDD 也明确不拥有它。Owner：#11。
- [RECOMMENDED] 局内 HUD 数值域（`in-game-hud.md:242`）：`hud_turn_slots` 把 `turns_total` 输入范围写成 `1–999`，本 GDD 与 `data-config.md:115` 都是 `1–INT32_MAX`。HUD 自身在 `turns_total > 12` 时改画比例条（第 235/356 行），放宽范围不影响绘制。顺带闭合 Open Question 4：槽位/比例条分界是 **12**，不是 OQ4 写的「超过 20」。Owner：#15。
- [RECOMMENDED] 数据配置行（`data-config.md:184`）：仍标「硬」并写「（新增字段，int ≥ 1）」，而字段已在第 52 行定义、第 98 行纳入 `R_sta`、第 115 行进入字段表。硬/软口径与本 GDD 第 76/205 行的「软（间接）、非运行时依赖」不一致。Owner：#1。
- [RECOMMENDED] Open Question 6（第 295 行）已复核仍在：`menu-waves.md:263` 仍写「本系统只发起扣除」。行为兼容（发起意图 ≠ 执行扣减），但这是实现时出现第二个调用者的入口。Owner：#10。
- [RECOMMENDED] Formulas / Acceptance Criteria 覆盖（第 86、259 行）：`can_spend_turn()` 是规则 7 与接口表公开的成员，却不在四条数值规则与 AC 覆盖表中；现有判据（AC 8/9）只覆盖耗尽后为 `false`。建议在 Formulas 补一行 `can_spend_turn = initialized ∧ turns_remaining ≥ 1`，并补一条 AC：`Uninitialized` 下返回 `false` 且调用前后三个读数不变。
- [RECOMMENDED] States and Transitions（第 58 行）/ 规则 12：`Active` 定义为「本局进行中」，但本局以 `NO_EATABLE_DISH` 结束且仍有剩余回合时，系统没有离开 `Active` 的路径（只有耗尽或下一次 `initialize()`），于是「本局已结束」与「状态 = Active」同时为真；`can_spend_turn()` 在 `run_ended` 之后也仍会返回 `true`。建议把 `Active` 改为「已初始化且未耗尽」，并明确 `run_ended` 后 `can_spend_turn()` 无意义 / 返回 `false`。
- [RECOMMENDED] Edge Cases（第 178 行）：「观测到 `turns_used > turns_total` 或 `turns_remaining < 0`：内部状态已损坏，返回结构化错误并停止消耗」——本系统对外只有三个 int 读数，没有能承载「结构化错误」的通道，错误词表也只覆盖 `initialize` 与 `spend_turn`。建议改为 `spend_turn()` 返回 `STATE_CORRUPTED` + 只读 `faulted` 标志，或删去「返回结构化错误」改为「停止消耗并记 ERROR」。
- [RECOMMENDED] 规则 6（第 39 行）：「…然后才广播 `action_committed` 并写入 `RunLog`」读起来像每次动作写一行日志；但 `run-flow.md` 规则 12 与 AC 14 规定 `RunLog` 在 `Ending` 组装一次、值取自动作发生时刻。建议改为「…并记录该时刻的回合值，供 `Ending` 组装 `RunLog`」。
- [RECOMMENDED] Acceptance Criteria 3（第 267 行）与接口签名（第 71 行）：接口声明为 `initialize(turn_budget: int)`，「缺失或不是整数」不是该签名能接收的输入，AC 3 无法写成针对本系统 API 的测试；`data-config.md:115` 与 AC 26 已在加载期拒绝非整数。建议二选一：AC 3 只保留整数夹具，类型错误归数据配置；或把参数声明为可空/`Variant` 并显式断言类型。
- [RECOMMENDED] Acceptance Criteria 21（第 285 行）：THEN 是「设计期审计应报告『回合不承重』」，而该审计的归属正是 Open Question 5（尚未落定），今天不存在可运行规则，判据无法独立判定 pass/fail。建议指向 `dish-values.md` 已存在的平衡审查并写成具体阈值，或把本条移回 OQ 5。
- [RECOMMENDED] 零回合预算的归属（跨文档）：`run-flow.md:248`、AC 13（第 399 行）、AC 37（第 423 行）仍断言「初始化后 `turns_total = 0` → 立即 `TIME_EXHAUSTED`，生成零行 `RunLog` 并正常结算」，而本 GDD AC 2 断言同一份非法数据（`turn_budget = 0`）经餐局流程开局应得 `init_failed`、不产生 `RunLog`；`data-config.md` 的 `R_sta` 又要求 `turn_budget ≥ 1`。需要明确：该分支只允许用测试桩驱动，还是改用 `init_failed`。本 GDD 自身一致（第 166 行已说明该分支不可达）。Owner：#5。
- [RECOMMENDED] 菜单波次参数命名（本 GDD 第 74/198/246 行 vs `menu-waves.md:164`、`run-flow.md:118/291`）：`ccdd0d7` 把菜单波次的入参统一命名为 `turn_budget`，本 GDD 仍写「从餐局流程收到 `turns_total` 作为参数」。数值等价（`turns_total = turn_budget`），但同一整数在两份文档里两个名字，建议本 GDD 改用 `turn_budget`（或注明二者同值）。Owner：#5 / #10。

Nice-to-have：

- Open Question 5 / 首局：`first-run-onboarding.md:277` 的首局 B0 是六道菜、`turns_total = 10`，回合在首局永远不承重（最多吃 6 道）。若审计最终落在菜品数值，需显式豁免首局档口。
- 规则 11（不发信号）没有判据断言。可补一条「本系统对外不暴露 `turn_*` 信号」的 AC（反射遍历信号表）。
- Advisory 章节：补一行 Player Fantasy（指向 Game Feel）与一行 Tuning Knobs（指向 Formulas 的 10±1 可调带）即满足 standard 层 advisory 标准。

### 已核对且无问题的跨文档接口（记录以免后续重复检查）

- `ccdd0d7` 未引入新冲突：餐局流程改为读取 `StationData.turn_budget` 后传整数（规则 22、第 118/291 行）与本 GDD 规则 3 一致；`submit_action` 的 `NO_TURNS` 拒绝码与 AC 8「第 11 次动作被 `can_spend_turn() = false` 拒绝」语义一致；`RunLog.entry_fee` 的加入不影响回合字段。
- 初始化与结束判定时机：本 GDD 规则 8「初始化后一次 + 每次成功提交后一次」= `run-flow.md` 规则 9；动作校验顺序「回合仍可用」在 `run-flow.md` 规则 6 有对应。
- 0 成本动作：自动推进与最后一波「结束取餐」不扣回合 = `run-flow.md` 规则 7 与 `transition_turn_cost = 0`；恒等式 `turns_used = 吃菜数 + 付费跳波数` 与 `menu-waves.md:33` 的 `eats + skips ≤ T` 一致。
- AC 20 的算术成立：3 波 × 4 道（`wave_size_default = 4`）下，「8 道 + 2 次跳过 = 10 回合」可达，`eats ≤ T − skips` 的上界也成立。
- 结算不变式 4：`n ≤ turns_used ≤ turns_total` 的下界来源（跳过让 `turns_used > n`）与 `settlement-and-entry-fee.md:275` 一致。
- 局内 HUD：同帧更新、只读快照、不自推耗尽、无倒计时 = `in-game-hud.md:35/37/51/61`；其不变式 `turns_used = turns_total − turns_remaining`（第 328 行）与本 GDD 公式一致。
- 支柱引用：`game-concept.md:141/153/159/167` 核对，支柱 1、3、4 编号与措辞正确，反支柱「NOT 实时操作」与规则 2 的硬边界一致。

### Verdict

**APPROVED**（0 条 blocking、15 条 recommended）。上一轮 blocking 的三条修法全部落地且正确，本文件已无「未写」字样、无自相矛盾；设计本体、四条数值规则、注册表取值与既有 GDD 的接口没有发现相反规则。剩余 15 条是跨文档口径同步（其中 6 条在别的 GDD 里，需回填）、`run_ended` 后读数语义、两处 AC 可测性与若干已过期的状态标签，均不影响实现，按 Recommended 跟进即可。

- **建议 systems-index 状态**：`Approved`（当前为 `In Review`）。本会话按编排约束**未修改** `design/gdd/systems-index.md`，由编排会话统一写入。
- **本 GDD 文件未被修改**：本次只评审，未改动任何 GDD、注册表或索引。

Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
