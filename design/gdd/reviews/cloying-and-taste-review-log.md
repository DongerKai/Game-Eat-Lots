# Cloying & Taste (腻度与口味) — Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 2 | Recommended: 9
Summary: 规则本体成立——五组公式全是整数运算，先付后记、解腻不自解、并列按规范组序都写死了顺序；20 条 AC 的数值逐条复算全部成立（104 元 / 20 格见证顺序、各组计数、拒绝边界、钳制、空组夹具均对），规范组序与菜品数值表的首次出现顺序逐项一致，`menu_cloy_pressure = 3` 复算正确。两处契约级缺口阻断：(1) 空 `cloying_group` 的语义在「腻度与口味」「数据配置」「菜品数值」三份 GDD 中给出三种不同行为；(2) 公式依赖的 `cloying_resistance` 没有任何进入本系统的接口路径——本 GDD 未定义接收签名，餐局流程的交互表与初始化序列也没有传递它。
Prior verdict resolved: First review

Findings:
- [BLOCKING] Rule 15 (`:39`) / Edge Cases (`:201`) / AC 20 (`:324`) vs `data-config.md:35,154` vs `dish-values.md:34,249`: 空串 `cloying_group` 三份 GDD 互相矛盾。本 GDD 说空串菜共享一个计数且第二道付 +1（AC 20 明写额外饱腹 1 格）；数据配置说「为空表示不参与腻度」「不参与腻度递减」；菜品数值说两个空组「都没有腻度惩罚」。同一输入三种行为，AC 20 无法同时通过两份上游。修法：统一为「空串 = 不参与」（与两份上游一致）并把 rule 15 / edge / AC 20 改为计数不产生、额外饱腹恒 0；或反之，但需改数据配置与菜品数值（不在本文件范围）。
- [BLOCKING] Formulas (`:76`) / Rule 4 (`:28`) / Variables (`:80-86`) / Interactions (`:60`) / `run-flow.md:31,121,131` / `meta-upgrades.md:358`: `cloying_resistance` 没有交付路径。公式含 `− floor(cloying_resistance)`，但 Variables 表没有它的行，rule 4 也仍写 `satiety + min(c, cloy_cap)`；`:90` 与局外升级都称「餐局流程开局时传入」，而 run-flow 的腻度交互行只列 `run_started`/`run_ended`、`extra_satiety`/`effective_satiety` 与组计数，初始化序列只初始化时间与回合、胃容量、菜单波次，`run_started(run_id, station_id, seed)` 不带快照。本 GDD 也没有任何 `initialize(...)` / 应用修正值的签名。结果：MVP 卖出 `upgrade_cloying_resistance` 后，该值要么被静默当作 0，要么两边各按不同路径实现。修法：命名接口（如 `initialize(cloying_resistance: float)`，由餐局流程在 `run_started` 前调用），在 run-flow 交互表与初始化序列补行，Variables 表补 `cloying_resistance`（float64，设计带 0–(cloy_cap−1)，来源本局快照），并在 rule 4 引用 `extra_satiety` 而非直写上限。
- [RECOMMENDED] Interactions (`:59`) / Dependencies (`:217`) / States (`:47-50`) vs `run-flow.md:45,131`: 生命周期写成「开始 / 结束 / 重开三个事件」，但 run-flow 规则 17 明写「重开不是独立生命周期事件」，信号只有 `run_started` / `run_ended` / `run_discarded`。改为两个信号 + 一句「重开 = 新的 `run_started`」，并补 `run_discarded`（放弃/退出）作为丢弃路径。
- [RECOMMENDED] Dependencies (`:214-218`): 上游表只有 菜品数值、餐局流程、数据配置，缺「局外升级」，尽管 Interactions (`:60`) 已把它列为上游（经快照），且 `meta-upgrades.md:370` 的改动清单明确要求补这一行。
- [RECOMMENDED] Formulas (`:175`) vs `menu-waves.md:34,61`: 「可发布菜单要求 `menu_cloy_pressure ≥ 1`」没有执行者。菜单波次的生成校验只查每波非空、划分覆盖、`wave_reachable_opt ≥ 0.9 × opt_value`，从不查 pressure；压力为 0 的菜单可达最优仍达标，会照常通过。要么指定这道闸门的拥有者（CI / 平衡审查），要么把句子降级为「设计期信号」——Edge (`:208`) 已经把压力 0 当作不阻断开局。
- [RECOMMENDED] Open Questions 1 (`:332`) 与 7 (`:338`): 均已被 `menu-waves.md` 回答（规则 3/5：可见集默认一波、跳过有回合代价；规则 2：每道菜恰好属于一波、不重复出现），menu-waves 自己 (`:233`) 也声明回答了 OQ1。标记为已回答。
- [RECOMMENDED] Tuning Knobs: 缺失（`standard` 下 advisory）。`cloy_cap`（设计带 1–5，默认 3）、规范组序、`cloying_resistance` 是本系统全部可调值，值得一节说明安全范围与影响。
- [RECOMMENDED] Player Fantasy: 无此标题（`standard` 下 advisory）；`## Game Feel` (`:251-257`) 已覆盖「顺序有意义、必须能复盘、解腻有节拍」，补一段或加交叉引用即可。
- [RECOMMENDED] Interactions (`:61`): 「容量系统用它判定可吃性与『吃撑』」与 `stomach-capacity.md:32` 冲突——胃容量规则 6 明确没有「吃撑」结果、不存在 `STUFFED`，可吃性判定叫 `can_fit`。改为「`can_fit(effective_satiety)`」。（胃容量评审已把该项路由到本文件。）
- [RECOMMENDED] Registry (`design/registry/entities.yaml`): `effective_satiety` (`:496`) 的 `referenced_by` 缺 `stomach-capacity.md`、`in-game-hud.md`、`run-flow.md`（三者都消费它）；`cleanse_target` (`:511`) 缺 `in-game-hud.md`；`cloy_cap` (`:1155`) 缺 `settlement-and-entry-fee.md`（`:222` 用作上界）；`group_cloy` 与 `is_cleansing` 无条目。注册表修改归编排会话。
- [RECOMMENDED] Formulas (`:106`, `:108`): 范围写作 `satiety_max + cloy_cap`，但 `satiety_max` 全文未定义（菜品数值规则 7 给的是 6）。写出 `satiety_max = 6`，读者的 1–9 才可复算。

Advisory (not counted): AC 3 (`:289`) 只断言「已用 16 格、肉计数 1」，但按规则第 7 步的番茄炒蛋（酸）会把规范组序最前的 `主食` 减到 0，第 8 步可乐（冰）再把 `蔬菜` 减到 0——复算者会看到两个未列出的计数变化，建议把中间各组计数写出。AC 4 (`:290`) 用 红烧肉→可乐→糖醋里脊，与 Formulas 的见证顺序（红烧肉→拍黄瓜→糖醋里脊）不同但同样零惩罚，成立，建议注明是第二条见证。`cleanse_target` 在注册表 (`:511`) 以 0–9 组索引建模，本 GDD 以组名 ∪ {⊥} 建模；等价，但实现者不要返回 int。`first-run-onboarding.md:185` 把 `group_count` 范围写成 0–`cloy_cap`，实际计数可达组内菜数（AC 17/18 到 4），偏紧。Cross-References (`:272-277`) 只列 4 份文档，读起来像全集，建议要么补全要么注明是节选。规范组序 (`:36`) 说「取菜品数值表中的首次出现顺序」，运行时却硬编码数组——说明它是冻结值、不随表序重算。

---

## Design Review: 腻度与口味（Cloying & Taste）
Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`。必需：Overview、Detailed Rules、Edge Cases、Dependencies、Acceptance Criteria，加 Formulas（系统定义数值规则：追加饱腹、可吃性判据、钳制、菜单压力）。六项齐全；`## Detailed Design` 承认为 Detailed Rules。

Advisory at `standard`:
- **Player Fantasy** — 无此标题；`## Game Feel` (`:251-257`) 覆盖意图感受。
- **Tuning Knobs** — 缺失；见 Recommended 第 7 项。

### Dependency Graph

声明依赖的 GDD 全部存在。另有 `局外升级` 经快照参与（Interactions `:60`）但未进 Dependencies 表。
- ✓ dish-values.md — exists
- ✓ run-flow.md — exists（但生命周期与抗性交接见 Blocking 2 / Recommended 3）
- ✓ data-config.md — exists
- ✓ meta-upgrades.md — exists（未列入 Dependencies 表）
- ✓ stomach-capacity.md — exists
- ✓ settlement-and-entry-fee.md — exists
- ✓ in-game-hud.md — exists
- ✓ menu-waves.md — exists
- ✓ first-run-onboarding.md — exists
- ✓ save-system.md — exists

### Registry Values

按注册表条目名与数值键对全文做了 71 行扫描（`extra_satiety`、`effective_satiety`、`cleanse_target`、`menu_cloy_pressure`、`cloy_cap`、`cloying_resistance`、`group_cloy`、规范组序、`opt_value`、`calibration_ratio`）。无数值冲突：

- `extra_satiety` (`entities.yaml:479`) `[0,3]` 与 `:86`、`:88` 一致；表达式与 `:76` 逐字相同（含 floor 抗性项）。
- `effective_satiety` (`:496`) `[1,9]` 与 `:106` 的 `satiety_max + cloy_cap`（6+3）一致；`:108` 的 MVP 1–7 是 cloy_cap 不生效时的更紧操作界。
- `cloy_cap` (`:1155`) = 3，与 `:85`、`:88`、AC 18 一致；`data-config.md:51,113` 的「≥ 1，默认 3」一致。
- `cleanse_target` (`:511`) 语义与 `:120` 一致（组名 vs 索引的表示差异见 Advisory）。
- `menu_cloy_pressure` (`:525`) 的 12 道菜池 = 3 复算正确（蔬菜 2、鲜 2、肉 2 → 各 1）。
- `opt_value = 104`、`calibration_ratio = 1.224`：104/85 = 1.2235 成立（`dish-values.md:63,215`）。
- 规范组序 `主食, 蔬菜, 饮料, 甜, 蛋, 辣, 油, 鲜, 肉` 与 `dish-values.md:46-57` 首次出现顺序逐项一致。
- 20 条 AC 的算术全部复算通过：AC 1–4 的容量序列 3→4→8→9→10→11→13→15→17→20；AC 5 的 3+7+3=13；AC 6 的 2+2=4；AC 7 的 2+2+3+3=10；AC 13 的 2+2+3=7；AC 17/18 的 `min(2,3)=2`、`min(4,3)=3`。
- 注册表缺口（`referenced_by` 不全、`group_cloy` / `is_cleansing` 无条目）见 Recommended 第 10 项；空组语义冲突是行为冲突而非数值冲突，见 Blocking 1。

### Required Before Implementation

1. **空 `cloying_group` 语义三向矛盾** — 本 GDD 规则 15 / 边界 / AC 20 说「共享一个计数、第二道 +1」；`data-config.md:35,154` 说「为空表示不参与腻度」；`dish-values.md:34,249` 说「两个空组都没有腻度惩罚」。规则 15 甚至声称「与菜品数值的支配规则一致」，而后者恰好说没有惩罚。AC 20 因此不可能同时通过自己的规则与上游两份 GDD。先定一种语义，再让三份文档一致；若取「不参与」，改 rule 15、edge `:201`、AC 20。
2. **`cloying_resistance` 无交付路径** — 公式已含它，但 Variables 表无行、rule 4 未引用、本系统无接收签名；`run-flow.md:31` 的初始化序列不含腻度，`:121` 的交互行与 `:131` 的信号都不携带该值。MVP 卖出该升级后行为不可实现。补接口、补 run-flow 的交接、补变量行。

### Recommended Revisions

3. 生命周期改为 run-flow 的两个信号 + 重开说明 + `run_discarded`。
4. Dependencies 上游表补「局外升级（经快照）」。
5. `menu_cloy_pressure ≥ 1` 指定闸门拥有者或降级为设计期信号。
6. OQ1、OQ7 标记为已被菜单波次回答。
7. 补 Tuning Knobs（`cloy_cap`、规范组序、`cloying_resistance`）。
8. 补 Player Fantasy 指针（Game Feel 已覆盖）。
9. 「吃撑」改为 `can_fit`，与胃容量的无 `STUFFED` 规则一致。
10. 注册表 `referenced_by` 补齐；考虑新增 `group_cloy` / `is_cleansing`。
11. 定义 `satiety_max = 6`。

### Specialist Disagreements

None — Lean mode runs no specialists (`Phase 3b` skipped).

### Nice-to-Have

- AC 3 的中间组计数未写出（番茄炒蛋会归零 `主食`、可乐会归零 `蔬菜`），断言本身仍成立。
- AC 4 是第二条零惩罚见证，与 Formulas 的见证不同；注明即可。
- `cleanse_target` 注册表用组索引、GDD 用组名，实现边界要统一。
- `first-run-onboarding.md:185` 的 `group_count` 上界偏紧（真实上界是组内菜数）。
- Cross-References 表注明是节选，或补全下游文档。
- 规范组序是冻结值，不随菜品数值表序重算。

### Scope Signal

Rough scope signal: **L**（producer should verify before sprint planning）。5 组公式、3 个硬上游 + 4 个硬下游 + 3 个软/条件下游、跨 10 份 GDD 的契约面。索引 (`systems-index.md:65`) 当前记 **M**，建议上调为 **L**。

### Verdict: NEEDS REVISION

---

## 建议索引状态

**Needs Revision**（`systems-index.md:65` 当前为 `Designed`；按并行约束未改索引，由编排会话统一更新）。
索引行 9 的 `Depends On` 只列「菜品数值, 餐局流程」，缺「数据配置」与「局外升级（经快照）」；Scope 建议 M → L。

Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 94da89a242967c44827406c4da5714d088df6394
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c
