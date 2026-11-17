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

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 1 | Recommended: 13
Summary: 规则 13 / AC 23 的职责切分本身是对的——拒绝由餐局流程执行、阈值读本系统的 `turns_remaining`、拒绝先于 `spend_turn()` 与 `advance()`、最后一波 `end_pickup` 代价 0 不受限，与 `run-flow.md` 规则 7 / AC 11a、`menu-waves.md` 规则 6 的最后一回合例外、`in-game-hud.md` 按钮禁用、ADR-0011 决策 6、ADR-0013 决策 4（`≥ 2` 门控）逐项一致；AC 18（剩余 2 → 1、推进波次、不结束）与 AC 16 的恒等式也成立。唯一 blocking 是新规则关掉了"花掉最后一回合跳波"这个出口，却没有为"最后一回合 + 当前波所有菜都放不进剩余容量"补出口：该状态下三种动作全部不可用、两个结束原因都不成立，本局无法结束。其余 13 条为「待回指」清单过期、接口"逐字一致"不成立、两处 AC 覆盖表未跟上新增判据、ADR-0006 的扣费顺序与门控陈旧，以及若干跨文档口径。设计本体与数值规则没有发现相反规则。
Prior verdict resolved: Yes（2026-11-04 的 APPROVED 针对旧字节；本轮为对当前字节的全量复审）

### 收据与陈旧性

上一轮 APPROVED（2026-11-04）绑定的本文件哈希为 `090c7e7f…`。本轮 `bash .claude/scripts/review-receipts.sh check design/gdd/reviews/time-and-turns-review-log.md design/gdd/time-and-turns.md` 读回 `CHANGED: design/gdd/time-and-turns.md`，同时注册表与 `run-flow` / `menu-waves` / `in-game-hud` / `settlement-and-entry-fee` / `data-config` / `dish-values` / `systems-index` 亦为 `CHANGED`（`SKILL.md`、`game-concept.md`、`project.yaml` 为 `UNCHANGED`）。该批改动新增规则 13、重写 AC 18、新增 AC 23、改写最后回合的边界情形与 Cross-References 列表，属 substantive，旧批准失效；本轮按 Phase 1 要求做全量复审，未做 delta 复审。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria（`gdd-structure-check.sh` 输出 `PRESENT: Overview,Detailed Rules,Formulas,Edge Cases,Dependencies,Acceptance Criteria`）。
Formulas 对本系统是**必需**的，不能清为 advisory：它定义回合预算 `turns_total = turn_budget`、固定消耗 1 的 `spend_turn`、已用恒等式 `turns_used = turns_total − turns_remaining` 与耗尽判据，并给出 `turn_budget` 的 1–INT32_MAX 取值域与 10±1 可调带——正是标准里"速率、曲线、阈值、成本"那一类。
Advisory 缺口（不阻塞，与上一轮相同）：Player Fantasy、Tuning Knobs（`ABSENT:` 两节）。Game Feel 覆盖 Player Fantasy 的实质内容，Formulas 的「可调带（设计期）」覆盖唯一可调值 `turn_budget`；补两行互相指向即可。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 餐局流程 | 上游（调用者，硬） | `design/gdd/run-flow.md` | ✓ 存在；第 296 行 §Dependencies 与第 369 行 §Cross-References 均回指本系统，接口仍为子集（见 R2） |
| 结算与入场费 | 下游（硬） | `design/gdd/settlement-and-entry-fee.md` | ✓ 存在；第 328 行回指；其不变式 4（第 276 行）与本 GDD AC 16 一致 |
| 局内 HUD | 下游（硬） | `design/gdd/in-game-hud.md` | ✓ 存在；第 384/449 行回指 |
| 菜单波次 | 间接（硬，经餐局流程） | `design/gdd/menu-waves.md` | ✓ 存在；第 175 行把 `T` 的来源列为本系统；其 `initialize(...)` 入参名为 `turn_budget`（第 63 行，见 R7） |
| 菜品数值 | 下游（软，仅设计期） | `design/gdd/dish-values.md` | ✓ 存在；第 270 行回指，软（仅设计期）口径一致 |
| 数据配置 | 数据所有者，非运行时依赖 | `design/gdd/data-config.md` | ✓ 存在；第 194 行回指，但该行仍标「硬 / 新增字段」（见 R12） |
| 局外升级 / 存档 / 输入 / 音效反馈 | 无直接交互 | `meta-upgrades.md` / `save-system.md` / `input.md` / `audio-feedback.md` | ✓ 四份均存在 |
| 胃容量 | 同级，不构成依赖 | `design/gdd/stomach-capacity.md` | ✓ 存在 |

无断链。索引第 6 行为 `Approved` + `design/gdd/time-and-turns.md` + `Depends On = 餐局流程`，与本文件第 217 行的自述一致。

### Registry Values

`design/registry/entities.yaml` 共 79 条命名条目。按 Phase 3 用全部条目名（`_` 展开为 `[_ ]`、大小写不敏感）扫描本 GDD 全文，命中 13 行，逐行核对后**无冲突**：

| 条目 | 注册表值 | 本 GDD | 结论 |
|---|---|---|---|
| `station_canteen_turn_budget` | `10` 回合，安全带 10±1，改值须重跑 `opt_value` / `calibration_ratio`；`referenced_by` = 本文件 / dish-values / menu-waves | 第 102/104/106/108/109/111 行同 | ✓ |
| `base_capacity` | `20` 格（`C0`） | 第 23/102 行 `C0 = 20` | ✓ |
| `opt_value` / `calibration_ratio` | `M, C, T` 为变量；阈值 0.9 / 校准带 `[1.15, 1.35]` | 第 23/77/108/111/181/201/212 行同口径 | ✓ |
| `end_reason` | `source: design/gdd/run-flow.md`，`TIME_EXHAUSTED` 优先 | 第 43 行明确本系统不保存、不排序 | ✓ |

`station_canteen_turn_budget.referenced_by` 未列结算，与本 GDD「结算只取 `RunLog.turns_total` / `turns_used`」的边界一致。`transition_turn_cost = 0` 与 `wave_advance_cost = 1` 未被本 GDD 按注册表名引用，但其正文的「自动推进与最后一波结束取餐代价 0、付费跳波 1」（第 144 行）与两值不冲突。

### Findings

- [BLOCKING] Detailed Design 规则 13（第 53 行）+ Edge Cases（第 178 行）+ AC 23（第 292 行）：最后回合拒绝付费跳波后，「最后一回合 + 当前波所有菜都放不进剩余容量」成为一个**无动作可做、无判据可结束**的局。此时 `eat` 被 `CAPACITY` 拒绝、`skip_wave` 被 `NO_TURNS` 拒绝、`end_pickup` 只存在于最后一波；结束判据只有 `TIME_EXHAUSTED` 与 `NO_EATABLE_DISH`，而 `no_eatable` 按 `menu-waves.md` 规则 13 与 `run-flow.md:36/261` 明确**不覆盖**"可见菜都放不进剩余容量"，`turns_remaining = 1` 也不触发耗尽——本局永久停在 `Active`，只能由玩家放弃（不结算、不写纪录）。可达性：`C0 = 20`、已用 16 格、非最后一波且该波剩余菜的 `effective_satiety ≥ 5`、`turns_remaining = 1` 即成立。规则 13 之前的"花掉最后一回合跳波"恰好是这一状态的唯一出口，新规则关掉了它却没有补出口。第 178 行"本局停在 `Active`，玩家仍可吃菜"是一个**没有容量前提的错误保证**；AC 23 把拒绝判据写成可测的 THEN，却没有为这一状态写任何预期。Authority：`run-flow.md` 规则 8（只有两个结束原因、容量不结束本局）+ `menu-waves.md` 规则 13（`no_eatable` 的三种成立条件），并与本文件第 17 行「它不决定一局什么时候结束」的边界自相矛盾——本系统提供的读数经新规则组合出了一个无终止态。同一假设还出现在 `menu-waves.md:31`（"玩家仍可吃当前波的菜"）、`menu-waves.md:221`（容量卡住时"玩家可以付费跳过"）、`run-flow.md:259/261`、`in-game-hud.md:503`（"此时吃菜按钮保持可用"），四处都需同步。修法三选一，选后退回一致：①规则 13 加例外——当前波无可吃菜时允许支付最后一回合跳波，由 `TIME_EXHAUSTED` 正常结束；②把"最后一回合 + 可见菜都放不进"定义为 `NO_EATABLE_DISH`（需 `menu-waves` 扩 `no_eatable` 的成立条件）；③允许任意波 0 回合「结束取餐」。任一方案都必须给出可测 AC 与 HUD 文案（当前 HUD 对"最后一回合不能跳波"与"放不下"给出两个互不相干的理由，玩家会看到死局）。

- [RECOMMENDED] Cross-References「待回指」第 2 项（第 260 行）：该条把 S4 的三处落地写成待办，但三项均已落地——`run-flow.md:35`（规则 7 的 `turns_remaining = 1` 前置拒绝）、`:402`（AC 11a）、`:262`（边界情形）；`menu-waves.md:31`（规则 6 的「最后一回合例外（S4）」）；`in-game-hud.md:372` 与 `:503`（按钮禁用 + 原因文案 + 最后一波仍可用）。因此"剩余两项"只剩第 1 项。属 stale cross-reference（不改规则/AC/数值），不据此阻塞，但应在下次实质修改时一并删去，否则会引导重复改动已验证的文档。Owner：本文件。
- [RECOMMENDED] Dependencies §双向一致核对（第 216 行）：「餐局流程（下游表，接口逐字一致）」仍不成立。`run-flow.md:120/296` 的接口只列 `initialize(turn_budget) → turns_total`、`spend_turn()`、`turns_remaining`、`turns_used` 与 `TIME_EXHAUSTED` 判据，缺本 GDD 规则 7 与 AC 23 所依赖的 `can_spend_turn()` 与 `is_exhausted()`（`NO_TURNS` 只出现在拒绝码里）。本 GDD 的接口是超集、方向正确；应改为「超集，`can_spend_turn()` / `is_exhausted()` 待餐局流程回填」。Owner：`run-flow.md` + 本文件。
- [RECOMMENDED] Acceptance Criteria 头部的两张覆盖表（第 264、266 行）：`Formulas 覆盖` 未列 AC 18（付费跳波在 `turns_remaining ≥ 2` 时扣 1 回合），而 AC 18 正是 `spend_turn` 规则的唯一正向用例；`集成判据` 只列"第 2、8、14、16、17、18、19、20 条"，未列 AC 23——AC 23 断言的是餐局流程的拒绝与"`spend_turn()` 未被调用"，只测本系统无法驱动。ADR-0011 的 Validation Criteria（第 308 行）已把 AC 23 列为需要测试替身，此处应同步。
- [RECOMMENDED] Edge Cases `turn_budget < 1`（第 168 行）与规则 4（第 35 行）：规则 4 写「保持原状态」，第 168 行写「保持 `Uninitialized`」；第 65 行的转移规则又允许从任意状态调用 `initialize()`，ADR-0011 决策 2（第 114 行）明确"如果此前是 ACTIVE 或 EXHAUSTED，保持原值"。AC 2 只从 `Uninitialized` 夹具验证，两种读法都能通过。建议第 168 行改为「保持调用前的状态（首次初始化即 `Uninitialized`）」，并补一条从 `Active` 用 `turn_budget = 0` 重入的 AC：三读数与状态全部不变。
- [RECOMMENDED] ADR-0006 的扣费顺序与门控（`docs/architecture/adr-0006-deterministic-rng.md:374`、第 540 行）：ADR-0006 写「扣回合由餐局流程在调用前完成：`if Time.can_spend_turn(): Time.spend_turn(); MenuWaves.advance()`」，并在禁令清单里固定为「`can_spend_turn()` → `spend_turn()` → `MenuWaves.advance()`」。这与规则 13 / AC 23 冲突（`can_spend_turn()` 在剩 1 回合时为 `true`，按此伪代码会放行被禁止的跳波），也与 ADR-0013 决策 4（第 154–161 行：门控是 `turns_remaining ≥ 2`，顺序是 `advance()` → `spend_turn()`）矛盾。本 GDD 自身无误，缺陷在 ADR-0006 的这段实现说明（该 ADR 解决的是划分与洗牌，回合顺序只是顺带声明）。Owner：architecture / ADR-0006；应在实现 `skip_wave` 前修正，否则两份 Accepted ADR 给出互斥的门控与顺序。
- [RECOMMENDED] AC 23（第 292 行）弱于其上游断言：`run-flow.md` AC 11a（第 402 行）与 ADR-0013 决策 4（第 161 行）都要求"当前波剩余菜不变"且"`advance()` / `spend_turn()` 调用计数为 0"，AC 23 只写"波次不推进、`turns_remaining` 与 `turns_used` 不变、不播放跳波成功反馈"。"不推进波次"不能拦下"调用了 `advance()` 但波索引未变"的部分实现（例如先丢弃当前波剩余菜再回滚），缺一条可断言的零副作用项。建议 AC 23 补「当前波剩余集合不变」与「`advance()` / `spend_turn()` 调用计数为 0」。
- [RECOMMENDED] 餐局流程传参命名（第 76、200、248 行）：本 GDD 三处写"菜单波次从餐局流程收到 `turns_total` 作为参数"，而 `run-flow.md:122/298` 与 `menu-waves.md:63` 的实参名为 `turn_budget`。数值等价（`turns_total = turn_budget`，本 GDD 第 94 行），但同一整数在两份文档里两个名字，实现时容易并成两个来源。建议本 GDD 改用 `turn_budget`，或注明二者同值。Owner：本文件。
- [RECOMMENDED] 规则 6（第 39 行）：「…然后才广播 `action_committed` 并写入 `RunLog`」读起来是每次动作写一行日志；但 `run-flow.md:40`（规则 12）与 AC 14 规定 `RunLog` 在 `Ending` 组装一次、字段值取自动作发生时刻。建议改为「…并记录该时刻的回合值，供 `Ending` 组装 `RunLog`」。Owner：本文件。
- [RECOMMENDED] Acceptance Criteria 3（第 272 行）与接口签名（第 73 行）：接口声明为 `initialize(turn_budget: int)`，「缺失或不是整数」不是该签名能接收的输入，AC 3 无法写成针对本系统 API 的测试；`data-config.md:117` 与 `run-flow.md` 规则 22 已在解析期拒绝，ADR-0011 决策 2（第 117 行）也把类型/存在性校验归 `RunFlow`。建议二选一：AC 3 只保留整数夹具（`0` / `-1`）并把类型错误归数据配置，或把参数声明为 `Variant` 并显式断言类型。Owner：本文件。
- [RECOMMENDED] Acceptance Criteria 16（第 285 行）：THEN 混合了两个不同的判据来源。「`n ≤ turns_used ≤ turns_total` 且结算能接收该 `RunLog` 而不返回 `TURN_INCONSISTENT`」可由结算校验（`settlement-and-entry-fee.md` 不变式 4，第 276 行；AC 22，第 463 行）；但「`turns_used = n + s`」需要 `s`，而 `RunLog` 不含跳过次数（`run-flow.md:40`），结算无法复核。建议把 AC 16 拆成两条：结算侧只断言不变式 4；恒等式侧由集成夹具（它驱动了动作序列，知道 `s`）断言。Owner：本文件 + 结算。
- [RECOMMENDED] 数据配置行口径（`design/gdd/data-config.md:194`）：仍标「硬」并写「（新增字段，int ≥ 1）」，而字段已在第 52 行定义、第 100 行纳入 `R_sta`、第 117 行进入字段表。与本 GDD 第 78/207 行的「软（间接）、非运行时依赖」及「本系统不查询数据目录」不一致，且"新增"已过期。Owner：数据配置。
- [RECOMMENDED] 局内 HUD 数值域（`design/gdd/in-game-hud.md:242`）：`hud_turn_slots` 把 `turns_total` 的输入范围写成 `1–999`，本 GDD 与 `data-config.md:117` 都是 `1–INT32_MAX`。HUD 在 `turns_total > 12` 时改画比例条（第 248/356 行），放宽范围不影响绘制。若采纳，可顺带把本文件 Open Question 4 的分界从「超过 20」改为 **12**（HUD 实际取值）。Owner：局内 HUD。

Nice-to-have：

- 规则 11（不发信号）是 HUD 与音效的接口前提，但没有判据断言它。可补一条「本系统不暴露任何 `turn_*` 信号」的 AC（反射遍历信号表）。
- Advisory 章节：补一行 Player Fantasy（指向 Game Feel）与一行 Tuning Knobs（指向 Formulas 的 10±1 可调带），即满足 standard 层 advisory 标准，不必新增内容。
- 头部 `Status: In Design` / `Last Verified: 2026-11-03` 与索引第 6 行的 `Approved` 不一致。按约束本轮**未修改**头部与索引，由编排会话统一写入。

### 已核对且无问题的跨文档接口（记录以免后续重复检查）

- 规则 13 的四项边界与上游逐项对齐：拒绝者 = 餐局流程（`run-flow.md:35`；ADR-0013 决策 4 的校验表第 2 行）；阈值读数 = `turns.get_turns_remaining() ≥ 2`，归 `RunFlow` 而非 `TimeTurns`（ADR-0011 第 169 行、ADR-0013 第 157 行）；拒绝先于 `spend_turn()` 与 `advance()`（`run-flow.md:262`、ADR-0013 第 161 行、"`advance()` 与 `spend_turn()` 调用计数为 0"的验证项第 35 行）；最后一波 `end_pickup` 代价 0、不受限（`run-flow.md:35`、`menu-waves.md:30`、ADR-0013 决策 5 第 172 行）。
- AC 18 的算术成立：`turns_remaining = 2` → 付费跳波 → 1，`advance()` 推进一波，`turns_used` +1，结束判定读到 `turns_remaining ≠ 0` 且 `no_eatable = false` → `NONE`，本局继续；与 `run-flow.md` 规则 9/10 一致，且不与规则 13 的门控冲突（2 ≥ 2）。
- AC 16 的不变式映射：`n ≤ turns_used ≤ turns_total` = `settlement-and-entry-fee.md` 不变式 4（第 276 行，错误码 `TURN_INCONSISTENT`）；`RunLog.turns_total` / `turns_used` 由 `run-flow.md` 规则 12 组装，本系统经快照提供（第 142 行）。
- 一次性初始化与无钳制：`run-flow.md` 规则 22（第 50 行）解析 `StationData.turn_budget` 后传整数；`data-config.md:52/100/117` 定义 `int ≥ 1` 并在加载期校验；ADR-0011 决策 2 与本 GDD 规则 4、Formulas 的 Output Range、AC 2/3、边界情形 168/169 同口径（`INVALID_TURN_BUDGET`、不钳到 1、不默认 10）。
- 0 成本动作：自动推进与最后一波「结束取餐」不扣回合 = `run-flow.md` 规则 7 / 过渡成本公式（第 219 行）+ `menu-waves.md:136` 的 `advance_cost ∈ {0,1}`；恒等式 `turns_used = 吃菜数 + 付费跳波数` 与 `menu-waves.md:136` 的 `eats + skips ≤ T` 一致。
- 局内 HUD：同帧更新、只读快照、不自推耗尽、无倒计时 = `in-game-hud.md` 规则 4/5/12/17；其不变式 `turns_used = turns_total − turns_remaining`（第 328 行）与本 GDD 公式一致；S4 的按钮禁用与原因文案在 `:372` 与 AC 26（`:503`）已落地。
- 支柱引用：`game-concept.md:141/153/159` 核对，支柱 1「每口都是算术题」、支柱 3「时间在走，手不用快」、支柱 4「数字看得见地涨」编号与措辞正确；反支柱「NOT 实时操作」与规则 2 的硬边界一致。

### Verdict

**NEEDS REVISION**（1 条 blocking、13 条 recommended）。规则 13 与 AC 23 的所有权切分、阈值归属、拒绝顺序与最后一波例外均正确，AC 18 / AC 16 也成立；阻塞项是**规则层面的状态缺口**而非文字问题：新规则移除了"最后一回合付费跳波"，却没有为"最后一回合 + 当前波所有菜都放不进剩余容量"定义任何终止路径，本局会停在 `Active` 且无合法动作，AC 23 也未覆盖该组合。修法只需在本文件（及三份姊妹文档的对应断言）上三选一，改动面小但必须先定方案再复审。

- **建议 systems-index 状态**：`Needs Revision`（当前为 `Approved`）。本会话按编排约束**未修改** `design/gdd/systems-index.md`，由编排会话统一写入。
- **本 GDD 文件未被修改**：本次只评审，未改动任何 GDD、ADR、注册表或索引。

Reviewed-Content-Hash: design/gdd/time-and-turns.md eb20c7ac3b422dbf3d5dcec1759061c55de41cd8
Reviewed-Content-Hash: design/registry/entities.yaml f7b5aded130f371160d84bdaf678f8530ddc0807
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/menu-waves.md 4282c9a66552dbb5023418b9c19542c1b5787af3
Reviewed-Content-Hash: design/gdd/in-game-hud.md fee7199048c3a124fcab2eccae8b88f93d1a69c0
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/data-config.md 9ecc2a7c18aba33c64bc3391594748e1063851d2
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0011-turn-budget-exhaustion.md c8f08fd667b22c8ef334f23bb3f17cc936846990
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 401cac388aed858afdc60a3a456e086bfc7310f0

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION（第 4 轮：S4「无合法动作」出口补入后的全量 lean 复审）

Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 1 | Recommended: 9
Summary: 上一轮唯一的 blocking 已在设计层关闭：规则 13（`:53`）、边界（`:178`）、AC 23（`:292`）现在都指向「无任何合法动作 → `NO_EATABLE_DISH`」，与 `run-flow.md:36`（规则 8）/`:438`（AC 43）、ADR-0013 §6 `:195-211`、`in-game-hud.md:504`（AC 26）/`:505`（AC 26a）、`stomach-capacity.md:192` 逐项一致；AC 23 补上了「当前波剩余集合不变 + `advance()`/`spend_turn()` 调用计数 0」，AC 23a 补上了「以 `NO_EATABLE_DISH` 结束且 `is_exhausted() = false`」的断言，且与 `run-flow.md` 规则 7 的 `turns_remaining ≥ 2` 门控不冲突（AC 18 已覆盖 2→1 的放行）。唯一 blocking 是同一修复在 AC 18 与边界 `:178` 末句留下的无条件保证：跳波在 `turns_remaining ≥ 2` 时被允许，但跳完落到 1 回合且新波无菜可放时，结束判定同帧以 `NO_EATABLE_DISH` 结束，"随后仍可吃一道菜"不成立。
Prior verdict resolved: Yes（上一轮 1 条 blocking 已关闭；2026-11-04 的 APPROVED 针对旧字节，本轮仍按全量复审）

Findings:
- [BLOCKING] Acceptance Criteria 18（`:287`，并 Edge Cases `:178` 末句）：`GIVEN turns_remaining = 2 ... THEN ... 本局不因此立即结束；随后仍可吃一道菜或在最后一波结束取餐`。反例可达：容量已满（`capacity_used = 20`，8 道 2–3 格的菜即可）或新波所有菜 `effective_satiety > capacity_remaining`、非最后一波、`turns_remaining = 2`；付费跳波通过 `≥ 2` 门控 → 扣到 1 回合 → 新波无菜可放 → 餐局流程同帧 `_no_legal_action()` 为真 → `NO_EATABLE_DISH`。该 AC 会按字面写出一个必然失败的用例，也与 ADR-0013 §6 `:195-211`、`run-flow.md` 规则 8 不一致。修法：AC 18 与 `:178` 补第三分支「新波无菜可放时同帧按 `NO_EATABLE_DISH` 结束」，或把 GIVEN 收紧为"新波至少有一道菜放得下"。
- [RECOMMENDED] Dependencies（`:219`）/ Open Questions（`:302`）：两处仍引用 `menu-waves.md` UI Requirements 的「本系统只发起扣除」，而该句已于 2026-11-06 改为「只声明代价…从不写回合计数」（`:259` 自己也这么记，`menu-waves.md:287` 为现行文本）。`:219` 与 `:259` 自相矛盾；OQ6 应标记为已回答。
- [RECOMMENDED] Cross-References（`:260`）：仍把「无合法动作」出口写成"剩余的唯一补充"，但 `run-flow.md` 规则 8 / AC 43 `:438`、ADR-0013 §6、本文件 AC 23a 均已落地；`:257` 又说"不再保留待办"。删除该句。
- [RECOMMENDED] Acceptance Criteria 头部（`:264`）：`Formulas 覆盖` 仍未列 AC 18（`spend_turn` 的唯一正向付费跳波用例）；`:266` 已补 AC 23/23a（上一轮 R3 只关闭一半）。
- [RECOMMENDED] Core Rules 4（`:35`）vs Edge Cases（`:168`）：规则 4 写"保持原状态"，`:168` 写"保持 `Uninitialized`"；ADR-0011 §2 `:114` 规定 ACTIVE/EXHAUSTED 下保持原值。`:168` 应改为"保持调用前状态（首次初始化即 `Uninitialized`）"（carried）。
- [RECOMMENDED] Interactions（`:76`/`:200`/`:248`）：三处写菜单波次从餐局流程收到 `turns_total`，而 `run-flow.md:123` 与 `menu-waves.md:63` 的实参名是 `turn_budget`；数值等价（`:94`）但同值两名（carried）。
- [RECOMMENDED] Core Rules 6（`:39`）："然后才广播 `action_committed` 并写入 `RunLog`" 读作每动作写一次日志；`run-flow.md` 规则 12 在 `Ending` 组装一次（carried）。
- [RECOMMENDED] 跨文档 ADR（`docs/architecture/adr-0006-deterministic-rng.md:374`）：仍固定 `can_spend_turn() → spend_turn() → MenuWaves.advance()`，与本文件规则 13 的 `≥ 2` 门控及 ADR-0013 §4 `:159-161` 的 `advance() → spend_turn()` 互斥；两份 Accepted ADR 不能同时指导实现（carried，owner architecture）。
- [RECOMMENDED] Dependencies（`:216`）：接口「超集、`can_spend_turn()` / `is_exhausted()` 待餐局流程回填」的措辞已改对，但 `run-flow.md:121` 仍未列这两个方法；保留为待回填项（carried）。
- [RECOMMENDED] systems-index（`:63`）：索引仍写 `Approved`，本文件头部 `:3` 为 `In Review`；`:217` 还把索引的 `Approved` 当作已对齐事实（carried；本轮不改索引）。
- 上一轮 R9（AC 3 类型夹具）、R10（AC 16 拆分 `n + s`）、R11（`data-config.md:194` 仍标「硬／新增字段」）、R12（`in-game-hud.md:242` 的 `1–999` 值域）未复述，状态未变。

**Completeness**: 6/6 required at `standard` — Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria（`gdd-structure-check.sh` 输出 `PRESENT: Overview,Detailed Rules,Formulas,Edge Cases,Dependencies,Acceptance Criteria`）。Formulas 对本系统必需（预算、固定消耗 1、`turns_used` 恒等式、耗尽判据与 10±1 可调带）。Advisory 缺口：`## Player Fantasy`、`## Tuning Knobs` 缺席，不阻断。

**Dependency Graph**: 声明的上游 `run-flow.md` ✓；下游 `settlement-and-entry-fee.md` ✓、`in-game-hud.md` ✓、`menu-waves.md` ✓、`dish-values.md` ✓；无直接交互的 `data-config.md` ✓、`meta-upgrades.md` ✓、`save-system.md` ✓、`input.md` ✓、`audio-feedback.md` ✓；同级 `stomach-capacity.md` ✓。回指复核：`run-flow.md:121`、`settlement-and-entry-fee.md:69`、`in-game-hud.md:199`、`menu-waves.md:175`、`dish-values.md:270`、`data-config.md:194` 均在；索引 `:63` 的 `Depends On = 餐局流程` 与本文件 `:203` 一致。无断链。

**Registry Values**: 按 79 个条目名全文扫描（`_` → `[_ ]`，大小写不敏感），命中 `station_canteen_turn_budget`、`base_capacity`、`opt_value`、`calibration_ratio`、`end_reason`、`capacity_used`、`running_net`（经 `run-flow` AC 11a 引用）。逐项无冲突：`turn_budget = 10`／安全带 10±1（`:106`）、`C0 = 20`（`:23`/`:102`）、`calibration_ratio` 带 `[1.15, 1.35]`（`:23`/`:108`）、`end_reason` 的 `TIME_EXHAUSTED` 优先（`:43`）、`capacity_used` 由容量拥有（`:49`）。`turns_used = 吃菜数 + 付费跳波数` 与 `menu-waves.md:136` 的 `eats + skips ≤ T` 一致；`running_net` 未被本文件重述，`run-flow.md` AC 11a 的「拒绝不改 `running_net`」与 AC 23 的零副作用一致。

### Verdict

**NEEDS REVISION**（1 条 blocking、9 条 recommended）。S4 的设计缺口本身已经关闭，规则 13 / AC 23 / AC 23a 与餐局流程、ADR-0013、HUD、胃容量逐项对齐；剩余阻塞是修复没有覆盖到的两处无条件保证（AC 18 与边界 `:178` 末句），属文字与可测性缺陷，补一个分支即可。

- **建议 systems-index 状态**：`In Review`（当前仍写 `Approved`）。本会话按约束**未修改** `design/gdd/systems-index.md`。
- **本 GDD 文件未被修改**：本次只评审，未改动任何 GDD、ADR、注册表或索引。

Reviewed-Content-Hash: design/gdd/time-and-turns.md 0f2ea128a9a42008413a15dd8608246513ef36f4
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: design/gdd/menu-waves.md a0b26691ca0f6ba5cac4c01ac0704913179b43ed
Reviewed-Content-Hash: design/gdd/in-game-hud.md 27e2978106c35139e3a3d0f371472a7aafd5dc2a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/data-config.md 92b3a28fc63465eacf3a88d3fec13bdff9f9455f
Reviewed-Content-Hash: design/gdd/dish-values.md 5f37fef5ac080b324a9b05a955d363dbae065a83
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 43fc521a857c5ce53ba3e1855c271ae2ffdfb1b6
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

---

## Review — 2026-11-17 — Verdict: APPROVED（第 5 轮：AC 18 / 规则 13 边界补「跳波后无合法动作同帧结束」后的全量 lean 复审）

Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 11
Summary: 上一轮唯一的 blocking 已关闭 —— AC 18 `:287` 与边界 `:178` 都改成条件式：2 → 1 的付费跳波之后，若新波至少一道菜放得下则停在 `Active`，若一道都放不下且非最后一波则同帧按 `NO_EATABLE_DISH` 结束且 `is_exhausted() = false`。该分支与 AC 23 `:292`（剩 1 回合拒绝、零副作用）、AC 23a `:293`（状态式出口、`is_exhausted() = false`）、规则 13 `:53`、`run-flow.md` 规则 8 `:37` 的 `_no_legal_action()`（非末波 ∧ `turns_remaining = 1` ∧ 无菜 `can_fit`）、规则 9 的「每次成功提交后判定一次」、ADR-0013 §6 `:195-211`、`in-game-hud.md:372`/`:373`/`:504`/`:505` 逐项一致；`end_reason` 注册表条目也已带 `no_legal_action`。全文已无"跳波后本局一定留在 `Active`"的无条件句（`:178`、`:287`、`menu-waves.md:31`、`:226`、`run-flow.md:263`/`:265` 均为条件式）。门控仍是 `turns_remaining ≥ 2`，与 `can_spend_turn()` 明确区分（`run-flow.md:35`、ADR-0013 §4 `:155-157`）。AC 编号 1–23 + 23a 无重复；`turn_budget = 10`、`C0 = 20`、`calibration_ratio` 带 `[1.15, 1.35]`、`wave_advance_cost = 1` 与注册表一致。
Prior verdict resolved: Yes（2026-11-06 的唯一 blocking 已关闭；AC 18 的 2→1 放行与 AC 23 的 1 回合拒绝互补，不重叠）

Findings:

- [RECOMMENDED] Dependencies（`:216`）：「接口是**超集**，`can_spend_turn()` / `is_exhausted()` 仍待餐局流程回填」已过期 —— `run-flow.md:301` 在本批同时补上了这两个方法（其 `:120` 的 Interactions 行仍缺）。两份列表现实际等长，建议改为「已回填；`run-flow.md:120` 的 Interactions 行仍缺 `can_spend_turn()` / `is_exhausted()`」。
- [RECOMMENDED] Dependencies（`:219`）/ Open Questions（`:302`）：两处仍引 `menu-waves.md` 的「本系统只发起扣除」，该句已于 2026-11-06 改为「只声明代价……从不写回合计数」（`:259` 自己记了这件事，现行文本是 `menu-waves.md:287`）；`:219` 与 `:259` 自相矛盾，OQ6 应标已回答（carried）。
- [RECOMMENDED] Cross-References（`:260`）：「剩余的唯一补充是「无合法动作」出口（`run-flow.md` 规则 8 / 本文件 AC 23a）」与 `:257`「本文件不再保留待办」相反，且该出口已落地（`run-flow.md` 规则 8 / AC 43 `:438`）；删除该句（carried）。
- [RECOMMENDED] Acceptance Criteria 头部（`:264`）：`Formulas 覆盖` 仍未列 AC 18（`spend_turn` 的唯一正向付费跳波用例）与 AC 23a（`is_exhausted() = false`）；`:266` 的集成判据已补 23/23a（carried）。
- [RECOMMENDED] Core Rules 4（`:35`）vs Edge Cases（`:168`）：规则 4 写"保持原状态"，`:168` 写"保持 `Uninitialized`"；ADR-0011 §2 规定 `Active`/`Exhausted` 下保持原值。`:168` 应改为"保持调用前状态（首次初始化即 `Uninitialized`）"（carried）。
- [RECOMMENDED] Interactions（`:76`）/ Dependencies（`:200`）：两处写"从餐局流程收到 `turns_total`"，而 `run-flow.md:123` 与 `menu-waves.md:63` 的实参名是 `turn_budget`（同一文件 `:216` 已用 `turn_budget`）；数值等价（`:94`）但同值两名（carried）。
- [RECOMMENDED] Core Rules 6（`:39`）：「然后才广播 `action_committed` 并写入 `RunLog`」读作每动作写一行；`run-flow.md` 规则 12 规定 `RunLog` 在 `Ending` 组装一次、字段取自动作发生时刻（carried）。
- [RECOMMENDED] 跨文档（`docs/architecture/adr-0006-deterministic-rng.md:374`）：仍固定 `if Time.can_spend_turn(): Time.spend_turn(); MenuWaves.advance()`，与本文件规则 13 的 `≥ 2` 门控及 ADR-0013 §4 `:159-161` 的 `advance()` → `spend_turn()` 顺序互斥；本 GDD 无误，缺陷在 ADR（carried，owner architecture）。
- [RECOMMENDED] Acceptance Criteria 23（`:292`）：零副作用清单可再补「`action_seq` 不变」与「不写日志」，与 `run-flow.md` 规则 7 `:35`、ADR-0013 §4 `:161` 的验证项同口径。
- [RECOMMENDED] Core Rules 13（`:53`）：建议补一句"不得以 `can_spend_turn()` 代替门控（它在剩 1 回合时为 `true`）"，与 `run-flow.md:35` 同句 —— 这是实现侧最易犯的入口。
- [RECOMMENDED] 索引（`design/gdd/systems-index.md:63`）：索引仍写 `Approved`，本文件头部 `:3` 为 `In Review`；本轮按约束不改索引。

**Completeness**: 6/6 required at `standard` —— Overview、Detailed Rules（`## Detailed Design`）、Formulas、Edge Cases、Dependencies、Acceptance Criteria（`gdd-structure-check.sh` 输出 `PRESENT: Overview,Detailed Rules,Formulas,Edge Cases,Dependencies,Acceptance Criteria`）。Formulas 对本系统必需（预算、固定消耗 1、`turns_used` 恒等式、耗尽判据与 10±1 可调带）。Advisory 缺口：`## Player Fantasy`、`## Tuning Knobs` 缺席，不阻断。

**Dependency Graph**: 上游 `run-flow.md` ✓；下游 `settlement-and-entry-fee.md` ✓、`in-game-hud.md` ✓、`menu-waves.md` ✓、`dish-values.md` ✓；无直接交互的 `data-config.md` ✓、`meta-upgrades.md` ✓、`save-system.md` ✓、`input.md` ✓、`audio-feedback.md` ✓；同级 `stomach-capacity.md` ✓。回指复核：`run-flow.md:301`、`settlement-and-entry-fee.md:328`、`in-game-hud.md:19`、`menu-waves.md:175`、`dish-values.md:271`、`data-config.md:52`/`:196` 均在。无断链；`run-flow.md:301` 现已列全本系统的接口（见上方 `:216` 一条）。

**Registry Values**: 按 79 个条目名与数值键全文扫描（`_` → `[_ ]`，大小写不敏感），命中 `station_canteen_turn_budget`、`base_capacity`、`opt_value`、`calibration_ratio`、`end_reason`、`capacity_used`、`running_net`、`is_exhausted`、`can_spend_turn`。逐项无冲突：`turn_budget = 10` 与安全带 10±1（`:104`/`:106`）、`C0 = 20`（`:23`/`:102`）、`calibration_ratio` 带 `[1.15, 1.35]`（`:23`/`:108`）、`end_reason` 的 `TIME_EXHAUSTED` 优先且新增 `no_legal_action` 分支（`:43`、AC 23a）、`capacity_used` 由容量拥有（`:49`）。`turns_used = 吃菜数 + 付费跳波数` 与 `menu-waves.md:136` 的 `eats + skips ≤ T` 一致；`advance_cost ∈ {0,1}` 与 `wave_advance_cost = 1`、`transition_turn_cost = 0` 不冲突。无注册表冲突。

**Verdict 备注**：本轮 blocking 为零。AC 18 的修法（保留 2→1 的放行、只给"新波无菜可放"加分支）与 AC 23 / 23a 的三态划分互不重叠，也闭合了 `turns_remaining ≥ 2` 门控留下的最后一个状态缺口；剩余 11 条均为跨文档陈旧引文、措辞精度与索引状态。

- **建议 systems-index 状态**：`In Review`（当前 `:63` 为 `Approved`）；本轮按约束**未修改** `design/gdd/systems-index.md`。
- **本 GDD 未被修改**：本次只评审。

Reviewed-Content-Hash: design/gdd/time-and-turns.md 78dcb4e9ce3a408138c7a08ff2464bc571447101
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/in-game-hud.md b8ead1263039519de2ff2ad9378fe43a8ea58373
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/data-config.md 36fd1ebaca36d1edee0c93b674f21d0418798713
Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 65248e9286c1f43dbf1c07e037544a10e0f0bb01
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0011-turn-budget-exhaustion.md c8f08fd667b22c8ef334f23bb3f17cc936846990
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/time-and-turns.md ce54e5bd984a1c7a2bbb973bf124bd998563f5b4

## Re-stamp — 2026-11-17（架构复审后）
仅同步闸门输入文件的当前字节（GDD 本体自上次复审后只改了 Status 头行）；未改规则、公式、AC、依赖、契约或平衡数值。

Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md e4d2ee1de1f4c9affd59a3993adb89056d5e4fa9
Reviewed-Content-Hash: design/gdd/dish-values.md 3eac1141e6c1209b59cc5e1ceeaa5c9ce85bc186
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/in-game-hud.md 6d7a4700f0f8b718eeb3bf0cbe9db7993df1f43d
Reviewed-Content-Hash: design/gdd/menu-waves.md af47aef125e14677602eeab6eed5ae98132c6113
Reviewed-Content-Hash: design/gdd/run-flow.md da67f644d65824cb6ae98081cea03f69f0e77a0d
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 82c3ea703a3c2d1e2b6bd0a8f00b4f87569cc4a8
Reviewed-Content-Hash: design/gdd/systems-index.md ccecd7af3ef6993252cb6901db7435d0985c0953
Reviewed-Content-Hash: design/gdd/time-and-turns.md ce54e5bd984a1c7a2bbb973bf124bd998563f5b4
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 06a670b544d8998eef4a8219cc9fa9d5a6b740b9
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: docs/architecture/adr-0011-turn-budget-exhaustion.md c8f08fd667b22c8ef334f23bb3f17cc936846990
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 8bfc9f56972968772b727c67dcc714e2e71baf1f
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
