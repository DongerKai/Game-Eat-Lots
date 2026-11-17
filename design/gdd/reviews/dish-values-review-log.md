# Review Log — 菜品数值（Dish Values）

## Revision — 2026-11-06 — GDD Revision Flags write-back

Closed the `architecture-review-2026-11-05` GDD Revision Flags for this system; this entry records a revision, not a fresh review.

- Flag #2 — rule 11 now states the single runtime exception to "`opt_value` does not enter runtime": the menu-waves health gate computes `wave_reachable_opt` in-process as the `wave_healthy` lower-bound baseline, and the result stays inside `HealthEval` (not exported to HUD / settlement / save, not a public API). Source: ADR-0006 §4.

Status: substantive edit — the GDD returns to `In Review` per `.claude/rules/design-docs.md`; approval requires a fresh full `/design-review`.

Reviewed-Content-Hash: design/gdd/dish-values.md 72cf43fcd49019dd6e5281ed8f6d7df8705a71e0

---

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none (lean — Phase 3b not run)
Blocking items: 4 | Recommended: 2
Summary: Flag #2's intent — the health gate is the one runtime exception to "`opt_value` never enters runtime" — is right, but the exception names the wrong quantity. ADR-0006 §4 makes the gate compute `opt_value(本局实际菜池, C0, T)` in-process (same `V[i][c][t]` recurrence, memoized) and compare `wave_reachable_opt` against `0.9 ×` it; the write-back instead says the gate computes `wave_reachable_opt` and uses it as the baseline, which conflates the measured per-wave optimum with the benchmark. Because the absolute bans at `:194`, `:211`, `:241` and AC 28 were left in place, the document now both forbids and requires a runtime `opt_value`. The registry `opt_value` note carries the same absolute ban.
Prior verdict resolved: First review — no prior entry for this system (the log was created by the write-back)
Findings:
- [BLOCKING] Core Rule 11 (`:37`): the exception reads "菜单波次的健康门在进程内用同一递推计算 `wave_reachable_opt`，作为 `wave_healthy` 的下界基准". Required: the gate computes the **`opt_value(本局实际菜池, C0, T)` baseline** with the same recurrence and compares `wave_reachable_opt ≥ 0.9 × baseline`; `wave_reachable_opt` is the measured partition optimum, not the baseline. Source: ADR-0006 decision §4 (`docs/architecture/adr-0006-deterministic-rng.md:230,253`), §6 (`reachable × 10 < baseline × 9`), `design/gdd/menu-waves.md` `wave_healthy` rule.
- [BLOCKING] Formulas heading (`:194`) and `opt_value` Output Range (`:211`): "仅设计期 / CI" and "**不进入运行时**——只在设计审查与 CI 中调用" remain absolute and contradict rule 11's new exception. Required: qualify both — `opt_value` is not exposed as a runtime query API; the only in-process caller is the menu-waves health gate, whose result stays inside `HealthEval`. Source: ADR-0006 §4 ("不暴露 … 健康门是唯一例外，且结果不外流").
- [BLOCKING] Acceptance Criteria 28 (`:364`): "`opt_value` 只能从 CI/设计审查入口到达" fails against a correct implementation of rule 11. Required: assert the absence of a **public/exported** caller — no HUD, settlement or save path may call it, and the only internal producer is `MenuWaves`' health baseline, whose value must not leave `HealthEval`. Source: ADR-0006 §4; Implementation Guidelines `:548`.
- [BLOCKING] Registry `design/registry/entities.yaml:458` (`opt_value` notes): "Design-time and CI only; never runs in-game" contradicts rule 11 and ADR-0006 §4. Required: "… in-game only via the menu-waves health gate (in-process, memoized, result never leaves HealthEval)". Not editable in this review (registry belongs to the orchestration session); Phase 3 treats a registry line that differs from the owning GDD as required before implementation.
- [RECOMMENDED] Formulas closing note (`:241`): "运行期的菜单划分闸门只有一条，且 … `wave_reachable_opt ≥ 0.9 × opt_value(pool, C0, T)`" is the correct contract, but the same paragraph ends "不进入运行时", so two adjacent sentences disagree on the same function. Reword the trailing clause to "不提供玩法查询接口" (ADR-0006 §4's own wording).
- [RECOMMENDED] Rule 11 (`:37`) / ADR-0006 §4: the ADR asks for the exception in the "接口所有权" section rather than in a core rule; either add a one-line cross-reference there or state in rule 11 that AC 28 is the API-surface counterpart.

Reviewed-Content-Hash: design/gdd/dish-values.md 72cf43fcd49019dd6e5281ed8f6d7df8705a71e0
Reviewed-Content-Hash: design/registry/entities.yaml 02f5c072fc535530c710fd71d77d31e2dce021ed
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/menu-waves.md 4c2da744c1c4b19bc861bd0c09509d6deecbd6b7
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 9bf45c1ead7e161599186ca5724fa460b044e314
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/systems-index.md 1119e17f51ee38b2b6fdb56db5d6301d6db94f47
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d

---

## Re-Review — 2026-11-06
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none (lean — Phase 3b not run)
Prior verdict: NEEDS REVISION (2026-11-06, 4 blocking / 2 recommended) — 4 of 4 blockers resolved; 1 prior recommended item still open.

### Blocker verification (current line numbers)

- **Resolved (1/4) — wrong baseline named.** `design/gdd/dish-values.md:37` (Core Rule 11) now states the gate computes `baseline = opt_value(实际菜池, C0, T)` with the same `V[i][c][t]` recurrence and compares the per-wave DP value `wave_reachable_opt` against it by integer cross-multiplication (`10 × reachable ≥ 9 × baseline`, equivalently `≥ ceil(9 × baseline / 10)`). `wave_reachable_opt` is the candidate, the baseline is `opt_value` over the actual pool — the two are no longer conflated.
- **Resolved (2/4) — absolute "never in-game" claims.** The `opt_value` heading no longer reads "仅设计期 / CI" (`:194`); the Output Range paragraph (`:211`) now scopes it: design/CI call directly, and the only runtime caller is the menu-waves health gate, result confined to `HealthEval` (`reachable_opt`, `baseline_opt`, `attempts`, `nodes`, `truncated`). `:241` scopes the CI regression check to CI only and names the single runtime gate. No remaining sentence in the GDD asserts that `opt_value` never runs at runtime.
- **Resolved (3/4) — AC 28.** `:364` now permits exactly one internal static path (`MenuWaves` → `HealthEval` → `baseline_opt`) and bans only public/exported callers plus UI / settlement / save consumers. Consistent with ADR-0006 §4 (`docs/architecture/adr-0006-deterministic-rng.md:253`) and Implementation Guidelines (`:538`).
- **Resolved (4/4) — registry `opt_value`.** `design/registry/entities.yaml:458` now says "Design-time and CI, plus the MenuWaves health gate as the sole internal baseline: baseline_opt = opt_value(actual pool, C0, T) … confined to HealthEval … never exposed to HUD, settlement, or save"; `revised: "2026-11-06"` (`:460`).

### New findings

**Blocking:** none.

**Recommended** (cross-document drift; not editable from this review, which may touch only the review log):

1. `design/registry/entities.yaml:611` (`wave_healthy` expression) still reads `... (reachable check ran and wave_reachable_opt < 0.9 * opt_value(pool, C0, T)) ...`, and `:612` reads `MVP lower bound 94 (0.9 x 104)`. Both use the float form ADR-0006 §6 forbids (`adr-0006:311-314`: `reachable × 10 < baseline × 9`, "**不得**写成 `reachable < 0.9 * baseline`") and never name `baseline_opt`. Ingestion of this entry reproduces the exact defect blocker 1 removed from the GDD.
2. `design/gdd/menu-waves.md` still describes the gate as a bare `0.9 ×` float rather than a comparison to `baseline_opt`: rule 9 `:35` ("**可达最优 ≥ 0.9 × `opt_value(pool, C0, T)`**"), `wave_healthy` formula `:190` (`wave_reachable_opt < 0.9 × opt_value(pool, C0, T)`), variables table `:200` ("下界 … `0.9 × opt_value(pool, C0, T)`；MVP 全池 104 → 94"), examples `:181` / `:206` ("0.9 阈值（94）" / "93.6 的下界"), AC 15 `:333` ("阈值 94 = ⌈0.9 × 104⌉"), AC 22 `:343` ("低于 `0.9 × opt_value(pool, C0, T)`"). Semantics agree with the GDD; the notation and the missing `baseline_opt` name do not.
3. `design/registry/entities.yaml:595` (`wave_reachable_opt` notes) opens with "Generation-time only." The DP runs inside `MenuWaves.initialize` at run start (ADR-0006 §5/§6), so "run-start only (inside `initialize`)" is the accurate phrasing; as written it re-suggests the design-time-only framing blocker 2 removed.
4. `design/gdd/dish-values.md:92` (`接口所有权`) still lacks the one-line exception ADR-0006 §4 explicitly requested there (`adr-0006:253`). The contract is stated at `:37`, `:211` and `:364`, but not in the section the ADR names. This was Recommended item 2 of the prior review and remains open.
5. `design/gdd/dish-values.md:36` (Core Rule 10) says the per-refresh threshold is "`opt_value` 偏差（见 Formulas 末节）". Read literally that points back at the baseline instead of the candidate; "`wave_reachable_opt` 相对 `baseline_opt` 的单侧下界" removes the ambiguity.
6. `design/gdd/menu-waves.md:301` (`待回改`) asks the dish-values calibration section to note that skips lower the per-run reachable ceiling; `design/gdd/dish-values.md:232-236` still discusses only the zero-penalty witness order. Cross-document follow-up, not a contradiction (0-skip partitions do reach 104).
7. Minor, pre-existing: registry output caps are narrower than the GDD's schema-derived bounds — `entities.yaml:406` `uv_span [0, 9999]` vs `dish-values.md:159` (`uv_span ≥ 0`), and `entities.yaml:456` `opt_value [0, 99999]` vs `dish-values.md:211` (`[0, Σ price]`, price ≤ INT32_MAX).

### Verified coherent (no finding)

- `return_ratio` range: `entities.yaml:664-666` `[0, +∞) | null`, empty run `0.0`, `null` only when `entry_fee = 0` — matches `design/gdd/settlement-and-entry-fee.md:180,183`.
- Calibration band `[1.15, 1.35]` with `104 / 85 = 1.224` and full-level `[1.35, 1.45]` with `120 / 85 = 1.412` agree across `dish-values.md:232-236`, `entities.yaml:490`, `settlement-and-entry-fee.md:280,493`.

### Verdict: APPROVED

> Conditional on the review's scope: `dish-values.md` now matches ADR-0006 §4/§6 and its own registry entry. Recommended items 1–3 and 7 are registry / menu-waves drift outside this document and must be fixed there; items 4–6 are wording and cross-document follow-ups. No engine or test run was performed (no Godot on this machine); findings are documentation-only.

Reviewed-Content-Hash: design/gdd/dish-values.md 38ec19c03e52375d9b2cb6581e1f76491da4d25a
Reviewed-Content-Hash: design/registry/entities.yaml 5d5722fcce313d3fcbe76129bf56fd232dc80735
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/menu-waves.md 4c2da744c1c4b19bc861bd0c09509d6deecbd6b7
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4

---

## Review — 2026-11-17 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 7
Summary: 全文 lean 复审（非增量）。上一条条目是 2026-11-06 的 APPROVED，之后本文件有实质修订（`In Review`），故重新全量读。本轮两项修订都成立：① 交互表 `:89` 与依赖表 `:280` 新增"餐局流程 | 下游 | 单菜 `price(d)`（只读，用于本局 `running_net` 累加）；不读 `satiety` / `unit_value` / `opt_value`"——与 `run-flow.md:50`、`:51`、`:119`、`:294`、`:319`、`:377` 逐字相符（run-flow 只累加 `price`，不读其他数值），已反向回指，且不违反 Core Rule 11 的"不聚合 / 运行期不提供最优解"（求和动作发生在餐局流程内部，本系统仍只给单菜数值）；② OQ2 的"`data-config.md` 的公式示例已对齐为 `entry_fee = 85`"经核对为真（`data-config.md:131`），与注册表 `entry_fee 85`（`entities.yaml:1219`）、`calibration_ratio 1.224`（`:490`）一致。遗留问题都是措辞与 OQ 陈旧，不阻断实现。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/dish-values.md`、`design/registry/entities.yaml`、`menu-waves.md`、`systems-index.md`；`UNCHANGED`：`SKILL.md`、`design-docs.md`、`settlement-and-entry-fee.md`、`game-concept.md`、`project.yaml`、`adr-0006`。上一份 APPROVED 绑定 `38ec19c0…`（2026-11-06），当前字节为 `1a899cb0…`；目标文档与注册表均已变动，故按全文复审。

### 本轮修订核对（新增边与 OQ2）

- **新增边属实且只读**：`run-flow.md:50` "本系统还读取每道菜的 `price(d)`——**只用于累加，不参与任何校验、判定或排序**……不读 `unit_value` / `opt_value`"；`:51` 的 `running_net` 累加器；`:294`/`:377` 的上游行；`:319` "只读、只用于累加"。本文件 `:89`/`:280` 的措辞与之同义，无夸大。
- **不违反"不聚合"**：Core Rule 11（`:37`）"本系统在运行时不计算一局的总价值、净利或最优解，也不提供查询最优解的接口"仍然成立——新增的是**读取方**，不是本系统新增的聚合接口；AC 28（`:366`）的静态检查面未被这条边破坏。
- **双向**：`run-flow.md:119` 与 `:294` 已把「菜品数值」列为上游，`:324` 还要求索引同步（索引缺口见下标 #2）。
- **OQ2 的 85**：`data-config.md:131` 示例确为 `entry_fee=85`；本文件 `:235`、`:365`（AC 27）与 `:381`（OQ2）三处一致；`settlement-and-entry-fee.md:26` 也确认 85。
- **注册表数值**：12 道菜的 `price` / `satiety` / `unit_value`（`entities.yaml:98-286`）与本文件菜品表 `:46-57` **逐格相等**（含 5.333333333333333 与 3.3333333333333335）；`unit_value`（`:366-381`）、`uv_cmp`、`uv_span`、`menu_healthy`（`:412-424`，1.5 − 1e-9 + 三档覆盖）、`dominates`、`opt_value`（`:443-461`，含 `baseline_opt` 例外与 `10 × r ≥ 9 × b`）表达式与本文件 Formulas 一致；`unit_value` 的 `referenced_by` 未列 `run-flow`，与该边只读 `price` 的事实相符。

### Findings

| # | 严重度 | 位置 | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | RECOMMENDED | `:37`（Core Rule 11） | 加入 run-flow 边后，"它只提供单菜的 `price` 与 `unit_value`；**求和由「结算与入场费」在 int64 中完成**"已不完整：运行期求和的实际持有者是餐局流程（`run-flow.md:51` "该累加器是本局运行期净值的**唯一权威副本**"；`in-game-hud.md:19`；注册表 `:699`），结算只提供 `running_net(k)` / `shortfall(k)` 纯函数并在结算时复核。建议改成"运行期累加归餐局流程，结算期求和归结算与入场费" | `run-flow.md:51`、`:319`；`settlement-and-entry-fee.md:25`；`entities.yaml:699` |
| 2 | RECOMMENDED | `:284` | "数据配置的 GDD 已在上游侧列出本系统；**其余系统仍待设计**，此要求记录在此供后续核对"与本表六行全为"GDD 已写"（`:277-282`）及 `:328` "所有列出本系统的下游 GDD 已完成并已回指"直接矛盾。另：`systems-index.md:62` 第 5 行 `Depends On` 仍为"场景与流程管理, 数据配置, 输入"，缺「菜品数值」（`run-flow.md:324` 明确要求补），本次新增边在索引侧尚未闭合（索引不可由本复审编辑） | 本文件 `:275-282`、`:328`；`run-flow.md:324`；`systems-index.md:62` |
| 3 | RECOMMENDED | `:95` | `查询错误契约` 把 `get_dish(id, caller)` 列为本系统的公开查询，但规则 2（`:28`）与上游行（`:85`）都把 `get_dish` 记为「数据配置」的接口，`data-config.md:57` 与其 AC 19 的签名是 `get_dish(id)`（无 caller）。同一个名字两个owner、两种签名，AC 6/7 的"查询"因此指向不明；要么把 `get_dish` 从本系统公开查询里去掉，要么改名（`dish(id, caller)`）并说明是委托 | `data-config.md:57`、`:248`；本文件 `:28`、`:85` |
| 4 | RECOMMENDED | OQ 表 `:380`、`:382`、`:384` | OQ 已陈旧：OQ1 以"胃容量 GDD 未写"开篇后同格写"已回答（2026-11-03）：C0 = 20"；OQ3 说"9 个 `cloying_group` 的归属尚未与「腻度与口味」核对"，而 `cloying-and-taste.md:36` 已把规范组序固定为同样这九个（主食 / 蔬菜 / 饮料 / 甜 / 蛋 / 辣 / 油 / 鲜 / 肉），本文件 `:233` 也写"「腻度与口味」已落地（2026-11-03）"；OQ5（是否允许重复抽取）已被 `menu-waves.md:26` "每道菜在一局中恰好属于一波，不重复出现"回答 | 本文件 `:233`；`cloying-and-taste.md:36`；`menu-waves.md:26` |
| 5 | RECOMMENDED | `:36`（Core Rule 10） | "单次刷新的门槛是 `opt_value` 偏差（见 Formulas 末节）"仍把候选指成基准；Formulas 末节（`:242`）判据是 `wave_reachable_opt ≥ ceil(9 × baseline / 10)`，没有"偏差"这个量（上轮 recommended 5，未修） | 本文件 `:37`、`:242`；ADR-0006 §6（`adr-0006:306-318`） |
| 6 | RECOMMENDED | `:93`（接口所有权） | ADR-0006 §4 明确要求"该例外需在 `dish-values.md` **下次修订时在其「接口所有权」段落**补一句"（`adr-0006:253`）。本轮就是那次修订，但接口所有权段仍只有两句话，`opt_value` 的运行时例外只出现在规则 11 / Formulas / AC 28（上轮 recommended 4，未修） | ADR-0006 §4（`adr-0006:253`） |
| 7 | RECOMMENDED | `:212` | "按 12 道菜、`C = 20`、`T = 10` 计算，状态数 2,400"与 ADR-0006 §4 的"`|pool| × (C0+1) × (T+1)` 个格子……MVP：`12 × 21 × 11 = 2772`"（`adr-0006:249`）给了同一计算的两个数。注明口径（2,400 = 12 × 20 × 10 个非常量格；2,772 = 含 c=0 / t=0 边界的格）即可，避免读者当成冲突 | ADR-0006 §4（`adr-0006:249`） |

### 已核对且无问题（记录以免重复检查）

- **`opt_value` / 健康门措辞**：规则 11（`:37`）、Output Range（`:212`）、`审查产出的使用方式`（`:242`）、Edge（`:256`）、AC 28（`:366`）四处口径统一（唯一运行期调用者是「菜单波次」健康门，结果留在 `HealthEval`），与 `menu-waves.md:35` 的整数判据、`adr-0006:228-253`（同一 `V[i][c][t]` 递推、`C0 = 20`、`baseline_opt = −1` 退化为 DEGRADED）一致；本文件未复述浮点 `0.9 ×` 形式。
- **`menu_healthy` 的作用域**：规则 10（`:36`）"只作用于档口菜池、不作用于单次刷新菜单、不阻断启动"与 Formulas `:155`（`M` = 档口 `dish_pool`）、注册表 `menu_healthy` notes（`:419`）一致。
- **口径与档位**：`satiety` 1–6、三档覆盖、UV 跨度 ≥ 1.5 与 `:147-149` 公式、AC 14/16/17/18 自洽；`:63` 的 104 / 20 格 / 10 回合与 AC 25 一致；`:64` 的贪心 94、9.6% 与 AC 25 一致。
- **校准带**：`[1.15, 1.35]`、`104 / 85 = 1.224`、满级 `120 / 85 = 1.412` ∈ `[1.35, 1.45]` 与注册表 `:490`、`settlement-and-entry-fee.md:280`、`:493` 一致。
- **完整性（standard 层）**：Overview、Detailed Design、Formulas、Edge Cases、Dependencies、Acceptance Criteria 6/6 齐全；Formulas 六条公式全部有变量表、值域、算例，数值规则必需项已满足。Player Fantasy、Tuning Knobs 为 advisory 缺口。
- **Dependency Graph**：上游 `data-config.md`、`stomach-capacity.md`、`time-and-turns.md` 均存在；下游 `cloying-and-taste.md`（`:13`、`:25`）、`menu-waves.md`（`:61`）、`settlement-and-entry-fee.md`（`:13`、`:25`）、`run-flow.md`（`:119`、`:294`）、`in-game-hud.md`（`:167`、`:387`）、`first-run-onboarding.md`（`:152`）全部存在并回指。无断链。

### Verdict: APPROVED

**APPROVED**（0 blocking · 7 recommended）。新增的餐局流程只读边属实、只读、已双向，不违反"不聚合 / 运行期不提供最优解"；OQ2 的 85 与 `data-config.md:131`、注册表一致；12 道菜与全部派生公式的数值与注册表逐格相符。7 条 recommended 都是措辞、索引同步与 OQ 陈旧（#1、#2 最值得在下一轮顺手改），不阻断实现。

- 条件：本结论只覆盖文档。未运行 Godot 或任何测试（本机无引擎运行）；`opt_value`、`menu_healthy` 等由 AC 12–27 在 CI 侧证明。
- 建议 `systems-index` 状态：**Approved**（第 8 行当前 `Approved`，与本次结论一致；索引里第 5 行的 `菜品数值` 边仍需同步，见 #2）。
- 本 GDD 未被修改，只追加本日志条目。

Reviewed-Content-Hash: design/gdd/dish-values.md 1a899cb0831e3b8e3afebaed272862124d97f1c8
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/menu-waves.md b4eb4df304c1a82859ea02ee18d0162c9c5c4485
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: design/gdd/in-game-hud.md b8ead1263039519de2ff2ad9378fe43a8ea58373
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/data-config.md 36fd1ebaca36d1edee0c93b674f21d0418798713
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/dish-values.md 3eac1141e6c1209b59cc5e1ceeaa5c9ce85bc186

## Re-stamp — 2026-11-17（架构复审后）
仅同步闸门输入文件的当前字节（GDD 本体自上次复审后只改了 Status 头行）；未改规则、公式、AC、依赖、契约或平衡数值。

Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md d8c7b56e7ffa8611ce9c271b25b064d38a8fb204
Reviewed-Content-Hash: design/gdd/data-config.md e4d2ee1de1f4c9affd59a3993adb89056d5e4fa9
Reviewed-Content-Hash: design/gdd/dish-values.md 3eac1141e6c1209b59cc5e1ceeaa5c9ce85bc186
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123
Reviewed-Content-Hash: design/gdd/in-game-hud.md 6d7a4700f0f8b718eeb3bf0cbe9db7993df1f43d
Reviewed-Content-Hash: design/gdd/menu-waves.md af47aef125e14677602eeab6eed5ae98132c6113
Reviewed-Content-Hash: design/gdd/run-flow.md da67f644d65824cb6ae98081cea03f69f0e77a0d
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md e738123010c4be9b0687a356209235d25c9904d4
Reviewed-Content-Hash: design/gdd/systems-index.md ccecd7af3ef6993252cb6901db7435d0985c0953
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: docs/architecture/adr-0006-deterministic-rng.md 70c7c9246c99c906f4b524504a491d337a475e3d
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
