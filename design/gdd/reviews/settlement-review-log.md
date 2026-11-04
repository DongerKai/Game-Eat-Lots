# Settlement & Entry Fee (结算与入场费) — Design Review Log

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean — Phase 3b not run)
Blocking items: 3 | Recommended: 9
Summary: The arithmetic is sound — every worked example reproduces exactly (104 − 85 = +19, 54 − 85 = −31, 104/85 = 122.3%, F5 at exactly 0, the cloying row 10/3 = 3.33), the four runtime invariants are stated as errors rather than clamps, and the no-`opt_value` / no-bonus boundary is held throughout. The blocking problems are all contract gaps introduced by the recent migration to `RunLog.entry_fee`: the fee's provenance is still `StationData.entry_fee` in rule 2 and three formula blocks, the first-run edge case says no value changes even though the teaching fee is 40, and the live `running_net` the HUD must read from before the first dish has no run-start initialization at all. None of the three requires redesigning the calculation; each is a stated-input or stated-lifecycle fix.
Prior verdict resolved: First review

Findings:
- [BLOCKING] Detailed Design (rule 2 `:26`) / Formulas (`:137`, `:143`, `:170`, `:249-253`) / Dependencies (`:324`) / Cross-References (`:414`): the entry-fee source is only half-migrated. The `RunLog` table (`:64`) and the Interactions row (`:95`) now say the fee comes from `RunLog.entry_fee` (`RunConfig.entry_fee_override` or `StationData.entry_fee`), but rule 2, the `net_profit` / `return_ratio` expressions and variable sources, the `running_net` / `shortfall` variable tables, the upstream dependency row and the Cross-References table still read `StationData.entry_fee` / `entry_fee(station_id)`. A programmer implementing from the formulas bypasses the frozen log and re-reads the station.
- [BLOCKING] Edge Cases (`:315`) / Acceptance Criteria (`:466`): the first-run edge case says settlement "数值完全不变" and that onboarding only touches the unit-value column, but the first run uses `entry_fee_override = 40` (registry `first_run_entry_fee = 40`; `first-run-onboarding.md:38,87,458` requires net +42 and 205.0%). No acceptance criterion ever feeds a `RunLog.entry_fee` that differs from `StationData.entry_fee` — AC 25 mutates the station value instead — so a regression back to station reads would pass all 30 ACs.
- [BLOCKING] Detailed Design (rule 19 `:43`) / States (`:47`) / Formulas (`:245-265`) / UI Requirements (`:398-399`) / AC 12 (`:450`) / AC 27 (`:468`): the live HUD net is unimplementable as specified. The HUD must show `running_net = −entry_fee` before the first dish and update it every commit (`in-game-hud.md:33,117-118,471`), but this system "不持有餐局进行中的状态" (`:47`), stores only `last_settled_run_id`, receives its only input at run end via `settle(RunLog)` (`:100`), and `running_net(k)`'s variables name only `k` — no run-start call carries the frozen fee, and no accumulator owner, reset point or discard handling is defined.
- [RECOMMENDED] Interactions (`:99`): `end_reason` is listed under 时间与回合, but `time-and-turns.md:43` (rule 8) says that system neither saves nor orders `end_reason`, and the registry makes run-flow the owner (`entities.yaml:866-877`). The `RunLog` table (`:65`) already says 餐局流程; drop it from the time row.
- [RECOMMENDED] Formulas (`:180`, `:239`) / AC 3 (`:441`): `return_ratio`'s declared range `(0, +∞)` excludes the empty run, where `total_value = 0` gives `0` (F3, `:297`), and `effective_unit_value`'s `(0, price_max]` excludes a zero-priced dish even though the same formula block declares `price` starting at 0 (`:126`). The registry's `[0, 999]` (`entities.yaml:646`) already includes 0.
- [RECOMMENDED] Registry (`design/registry/entities.yaml`): several `output_range` bounds are narrower than the GDD's declared int64 domain — `total_value [0, 999999]` (`:603`), `net_profit [-99999, 999999]` (`:619`), `running_net` (`:675`), `shortfall` (`:690`), `return_ratio [0, 999]` (`:646`), `effective_unit_value [0, 9999]` (`:660`) — while the GDD allows `price`/`entry_fee` up to INT32_MAX. Registry edits belong to the orchestration session.
- [RECOMMENDED] Open Questions (`:487-492`): OQ2, OQ3 and OQ4 are already answered elsewhere — `run-flow.md:36` (no early cash-out), `records-and-milestones.md:31,309` (rank by net profit), and onboarding's `replacement_hint` (`entities.yaml:1128-1141`). Mark them answered so the remaining open set is honest.
- [RECOMMENDED] Cross-document (route to `/consistency-check`): `data-config.md:81,182` still assigns `entry_fee` reads to settlement; with the new contract run-flow resolves the override and writes `RunLog.entry_fee`, and settlement reads the log. `run-flow.md:453` (OQ10) tracks the same correction.
- [RECOMMENDED] Interactions (`:91-107`): hardness disagrees with the other side for two rows — 音效反馈 is 软 here but "硬（事件契约）" in `audio-feedback.md:107,351`; 首局引导 is 软 here but 硬（B4）in `first-run-onboarding.md:126`. 菜单波次 appears only in Dependencies (`:342`), and 场景与流程管理 is cited four times in Rules/Edge Cases (`:31`, `:308-310`) but has no row at all.
- [RECOMMENDED] Scope signal: `systems-index.md:206` records `S`, but this GDD has 6 hard upstream + 8 downstream systems and nine formula blocks. That is `L` by the review scale; the index edit belongs to the orchestration session.
- [RECOMMENDED] Tuning Knobs: absent (advisory at `standard`). The only runtime knob is `entry_fee` (data-driven, `:26`); a short section naming it plus the `[1.15, 1.35]` unupgraded band and the `[1.35, 1.45]` full-level band (`meta-upgrades.md:261`) would give a balance pass one place to look.
- [RECOMMENDED] Player Fantasy: absent as a heading (advisory at `standard`). `## Game Feel` (`:375-382`) covers the intended feeling — the receipt as a verifiable account, loss without shame — well enough that a one-line cross-reference would close the gap.

Advisory (not counted): Invariant 1 (`:273`) writes `capacity_used = RunLog.capacity_used`, which reads as a self-comparison; write `Σ effective_satiety = RunLog.capacity_used`. The Cross-References note "所有列出本系统的下游 GDD 已完成并已回指" is true for the table, but `scene-flow.md` — cited as the owner of the `Run → Menu` payload in `:31,417` — contains no reference to this system (0 hits); either add the indirect dependency row or note that scene-flow only carries the payload. F6 (`:435`) should be restated as `RunLog.entry_fee = 0` once the fee migration lands, so the fixture cannot be read as a station-data test.

---

## Design Review: 结算与入场费（Settlement & Entry Fee）
Specialists consulted: none (lean mode)
Re-review: No — first review

### Completeness: 6/6 required sections present

`modes.workflow: standard`. Required: Overview, Detailed Rules, Edge Cases,
Dependencies, Acceptance Criteria — plus Formulas, because this system defines
numeric rules (sums, thresholds, ratios, checked accumulation). All six are
present; `## Detailed Design` is accepted as the Detailed Rules section.

Advisory at `standard`:
- **Player Fantasy** — absent as a heading; `## Game Feel` (`:375-382`) covers
  the intended feeling (a receipt you can check line by line; loss stated, not
  judged).
- **Tuning Knobs** — absent; the one knob is the data-driven `entry_fee`.

### Dependency Graph

Every declared dependency has a GDD on disk.

Upstream:
- ✓ data-config.md — exists
- ✓ dish-values.md — exists
- ✓ cloying-and-taste.md — exists
- ✓ stomach-capacity.md — exists
- ✓ time-and-turns.md — exists
- ✓ run-flow.md — exists

Downstream:
- ✓ meta-upgrades.md — exists
- ✓ records-and-milestones.md — exists
- ✓ ui-flow.md — exists
- ✓ in-game-hud.md — exists
- ✓ first-run-onboarding.md — exists
- ✓ audio-feedback.md — exists
- ✓ save-system.md — exists
- ✓ menu-waves.md — exists

Cited but not in either table:
- ✓ scene-flow.md — exists; its Rule 6 (`scene-flow.md:32`) does carry the
  `RunResult` on `Run → Menu`, but the file never names this system (see
  Advisory).

### Registry Values

Searched the whole document for every registry term this system owns or
references (`total_value`, `net_profit`, `broke_even`, `return_ratio`,
`effective_unit_value`, `running_net`, `shortfall`, `capacity_used`,
`wasted_capacity`, `cloy_extra_total`, `unit_value`, `calibration_ratio`,
`extra_satiety`, `effective_satiety`, `run_id_alloc`, `end_reason`,
`capacity_total`, `record_is_better`, `replacement_hint`,
`first_run_entry_fee`), case-insensitive with `[_ ]` — 134 matching lines.
No numeric conflict:

- `station_canteen_entry_fee = 85` (`entities.yaml:1183-1193`) matches `:26`,
  `:143` and every worked example.
- `extra_satiety [0,3]`, `effective_satiety [1,9]` match `:222`, `:203`, `:275`.
- `capacity_used` and `wasted_capacity` match `:193-208` (the GDD's tighter
  `[0, capacity_total]` is the same quantity under the registry's 999 cap).
- `unit_value` "displayed at two decimals at settlement only" (`:363-377`)
  matches rule 10 and `:389`.
- `calibration_ratio` notes match `:187`, `:280-290`.
- `run_id_alloc` and `end_reason` semantics match `:62`, `:65`.
- `record_is_better` (net profit only) matches `:488` and is already the
  records GDD's answer (`records-and-milestones.md:31`).
- `replacement_hint` exists upstream (`entities.yaml:1128-1141`), answering
  OQ4.

Conflicts or gaps:
- **`first_run_entry_fee = 40`** (`entities.yaml:1276-1291`) has no path in
  this GDD — no mention of `entry_fee_override`, no teaching-price statement,
  and the first-run edge case says values do not change (Blocking 2).
- **`end_reason` owner** is run-flow (`entities.yaml:866-877`), against the
  Interactions row that assigns it to 时间与回合 (`:99`) (Recommended 1).
- **`return_ratio` range** is `[0, 999]` in the registry (`:646`), which
  includes 0; the GDD's `(0, +∞)` excludes the empty run (Recommended 2).
- **Narrower output ranges** in the registry for the int64 quantities
  (Recommended 3).

### Required Before Implementation

1. **Finish the `entry_fee` migration** — rule 2 (`:26`), the `net_profit`
   expression and variable source (`:137`, `:143`), the `return_ratio`
   expression (`:170`), the `running_net` / `shortfall` variable tables
   (`:249-253`), the upstream data-config row (`:324`) and the Cross-References
   row (`:414`) still name `StationData.entry_fee` / `entry_fee(station_id)`.
   The `RunLog` table (`:64`) and the Interactions row (`:95`) already have the
   right contract, and `run-flow.md:40,64,387` defines it: run-flow resolves
   `RunConfig.entry_fee_override`, falling back to `StationData.entry_fee`, and
   freezes it in `RunLog.entry_fee`. Rewrite rule 2 to say the run's fee is the
   frozen `RunLog.entry_fee`, and make every formula read `R.entry_fee` rather
   than the station record. Keep `StationData.entry_fee` only as the fallback
   source and as the value the menu displays before a run (`ui-flow.md:55`).
   Without this, the first run computes against 85 instead of 40, and a
   station-data hot reload can change the arithmetic of a run already in
   progress.

2. **State the first run's actual numbers and test the override path** — the
   `If 首局` edge case (`:315`) says "结算流程与数值完全不变" and that onboarding
   "不改变任何合计". That contradicts the 40-yuan teaching fee
   (`entities.yaml:1276-1291`; `first-run-onboarding.md:38,87,458`: net +42,
   205.0%, first upgrade affordable) and onboarding's B4, which explains the
   receipt's teaching price. Rewrite the edge case: the first run is a normal
   settlement over a `RunLog` whose `entry_fee = 40`, so `net_profit`,
   `broke_even`, `return_percent_x10` and the receipt's entry-fee line all
   differ; only the calculation rules are unchanged. Add a fixture + AC where
   `RunLog.entry_fee ≠ StationData.entry_fee` (e.g. F2 with 37 against a
   station at 85 → `net_profit = 17`), plus one asserting the HUD starts at
   the overridden fee. AC 25 (`:466`) currently mutates the station value, so
   nothing exercises the contract this revision introduced.

3. **Define the in-run interface for `running_net` / `shortfall`** — rule 19
   (`:43`) and the HUD requirements (`:398-399`) require a live reading from
   before the first dish, and AC 12 (`:450`) / AC 27 (`:468`) test it, but the
   States section (`:47`) says the system holds no in-run state, the only
   input is `settle(RunLog)` at run end (`:100`), and the formula's variables
   name only `k` (`:245-265`). Nothing carries the run's frozen fee into the
   system at `run_started`, nothing owns or resets the int64 accumulator, and
   a discarded run has no cleanup path — so the next run's HUD would start
   from the abandoned run's total. Fix by adding one run-start call (e.g.
   `begin_run(entry_fee)` from run-flow's `run_started`, or a subscription to
   the resolved `RunConfig`), defining who adds a committed row and when the
   accumulator resets, and adding the corresponding transition to the States
   table. Then add an AC that abandons a run and asserts the next run's HUD
   starts at `−entry_fee`. A pure `running_net(entry_fee, partial_rows)`
   alternative is possible but conflicts with `in-game-hud.md:170` ("不读
   `entry_fee` 自行计算"), so the run-start call is the cheaper fix.

### Recommended Revisions

4. **`end_reason` provenance** — remove it from the 时间与回合 row (`:99`);
   `time-and-turns.md:43` explicitly does not hold or order it, and run-flow
   owns it (`entities.yaml:866-877`). The `RunLog` table is already correct.
5. **Boundary ranges** — change `return_ratio` to `[0, +∞)` (`:180`) and state
   what the empty run shows (0.0%, not blank); change `effective_unit_value`
   to `[0, price_max]` (`:239`) to match `price ≥ 0` (`:126`).
6. **Registry output ranges** — widen `total_value` / `net_profit` /
   `running_net` / `shortfall` / `return_ratio` / `effective_unit_value` to the
   GDD's declared int64 domain, or state that the registry records the
   practical band and the GDD the defensive maximum. Registry edit belongs to
   the orchestration session.
7. **Close answered Open Questions** — OQ2 (`run-flow.md:36`), OQ3
   (`records-and-milestones.md:31,309`), OQ4 (`entities.yaml:1128-1141`).
8. **`data-config.md:81,182`** — the row still says settlement reads
   `entry_fee`; after this fix run-flow resolves the override and writes the
   log. Route to `/consistency-check` with `run-flow.md:453` (OQ10).
9. **Interactions table** — align 音效反馈 and 首局引导 to hard as their own
   GDDs declare (`audio-feedback.md:107,351`; `first-run-onboarding.md:126`);
   add rows for 菜单波次 and 场景与流程管理, or state explicitly that they
   reach this system only through run-flow.
10. **Scope signal** — `systems-index.md:206` says `S`; the dependency count
    and formula count put this at `L`. Orchestration-session edit.
11. **Tuning Knobs** — add the `entry_fee` knob and the two calibration bands.
12. **Player Fantasy** — add a short section or cross-reference Game Feel.

### Specialist Disagreements

None — Lean mode runs no specialists (`Phase 3b` skipped).

### Nice-to-Have

- Invariant 1 (`:273`) should read `Σ effective_satiety = RunLog.capacity_used`
  rather than `capacity_used = RunLog.capacity_used`.
- `scene-flow.md` is cited as the owner of the `Run → Menu` payload (`:31`,
  `:417`) but never names this system; an indirect dependency note would close
  the loop.
- Restate F6 (`:435`) as `RunLog.entry_fee = 0` after Blocking 1 so the fixture
  cannot be read as a station-data test.

### Scope Signal

Rough scope signal: **L** (6 hard upstream + 8 downstream systems, nine
formula blocks, no new ADR expected; producer should verify before sprint
planning). The index records `S`.

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md e1f10d21046a92cf452a81f4d6fd544367d7f3dc
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/dish-values.md 60a580b19c4b2f980cd0a33df73e8c3aa32f6d3e
Reviewed-Content-Hash: design/gdd/time-and-turns.md 090c7e7fcd313a7c38092e9bf07cdd12b77dd34f
Reviewed-Content-Hash: design/gdd/in-game-hud.md 1c51b6fff07c5d8eae58559e50e31079d165bbd9
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/stomach-capacity.md 94da89a242967c44827406c4da5714d088df6394
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 2 | Recommended: 10
Summary: 三项 blocking 中两项已实质解决。首局现在明确按 `RunLog.entry_fee` 计算，并新增 AC 31 覆盖 40 元教学价（82 → +42、2050）；实时累加器的归属已改为餐局流程，`run-flow.md` 规则 23 与 AC 22/24 补齐了初始化、逐次累加与放弃丢弃，HUD 只读快照的边界两侧一致。但 `entry_fee` 迁移仍留有两处同类残留：本文件上游依赖表仍写 `get_station(id) → entry_fee`，注册表的 `net_profit` / `return_ratio` 表达式仍写 `entry_fee(station_id)`——以机器可读契约实现时仍会绕过冻结日志。设计本体、算术与边界未发现新问题。
Prior verdict resolved: Partial — B2（首局）、B3（实时净利）已解决；B1（`entry_fee` 迁移）主体已解决，上游依赖行与注册表表达式未迁移。

### 复审背景

- 触发：`427409f`（Batch 2/3 修复）与 `be69169`（Batch 4-6 修复）改动本文件及其关联 GDD；上轮评审哈希为 `c4175b0`。
- 复审范围：**全文重读**（493 行，按标题分段），未按 diff 跳读。
- 本轮实际改动：规则 2（冻结费）、规则 19（累加器归属）、`net_profit` / `return_ratio` 公式与变量、首局边角案例、Cross-References 数据配置行、HUD 下游依赖行、AC 31、音效事件互斥句。依赖图、公式表其余部分、Edge Cases 其余条目、交互表均未改。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统必需：它定义求和、净利、回本判定、回本率、容量、腻度、单位价值与实时净值九组数值规则。

Advisory 缺口（不阻塞，与上轮相同）：Player Fantasy、Tuning Knobs 无独立标题。Game Feel（`:375-382`）实质覆盖前者；唯一运行期旋钮是数据驱动的 `entry_fee`，后者可由一行交叉引用补齐。

### Dependency Graph

上轮已核对 14 份被引 GDD 全部在盘；本轮复核关联文件仍存在，无断链：`data-config.md`、`dish-values.md`、`cloying-and-taste.md`、`stomach-capacity.md`、`time-and-turns.md`、`run-flow.md`、`meta-upgrades.md`、`records-and-milestones.md`、`ui-flow.md`、`in-game-hud.md`、`first-run-onboarding.md`、`audio-feedback.md`、`save-system.md`、`menu-waves.md`、`scene-flow.md`。

跨文档事实复核：
- ✓ `run-flow.md:51` 规则 23 已接收累加器所有权：开局 `−entry_fee`、每次提交 `+price(d)`、放弃/退出/故障丢弃；`run-flow.md:390` AC 1 保证 `RunLog.entry_fee` 等于 `RunConfig.entry_fee_override`（`null` 时取 `StationData.entry_fee`）。B3 两侧咬合。
- ✓ `in-game-hud.md:170` 明确 HUD 经快照读 `running_net` / `shortfall`，不读 `entry_fee` 自行计算；与本文件规则 19、UI Requirements 一致。
- ✓ `audio-feedback.md:38` 规则 12 与 `:302-303` 支持 `settlement_started` 先播、结果 cue 在数字滚动结束时起播；本文件新写的「`settlement_started` 与结果三选一可并存、结果三者互斥」两侧一致。
- ✓ `first-run-onboarding.md:38`、`:297-302`、`:458` 的 40 元教学价、总值 82、净利 +42、回本率 205.0% 与 AC 31 完全一致。
- ⚠ `data-config.md:81,182` 仍写「结算读取 `entry_fee`」；`run-flow.md:316` 与 OQ10（`:456`）仍写「本设计中入场费由结算读取」。按 `run-flow.md:40,64,387` 与本文件规则 2，实际是餐局流程解析并冻结、结算只读 `RunLog.entry_fee`。属跨文档措辞同步，路由 `/consistency-check`。

### Registry Values

对全文做了 145 行 registry 术语扫描（`total_value`、`net_profit`、`broke_even`、`return_ratio`、`effective_unit_value`、`running_net`、`shortfall`、`capacity_used`、`wasted_capacity`、`cloy_extra_total`、`unit_value`、`calibration_ratio`、`extra_satiety`、`effective_satiety`、`run_id_alloc`、`end_reason`、`capacity_total`、`record_is_better`、`replacement_hint`、`first_run_entry_fee`、`station_canteen_entry_fee`、`entry_fee`，大小写不敏感，`_` 按 `[_ ]`）。

数值无冲突：`station_canteen_entry_fee = 85`（`entities.yaml:1183-1193`）与全文及所有算例一致；`first_run_entry_fee = 40`（`:1276-1287`）与首局边角、AC 31 一致；`extra_satiety [0,3]`、`effective_satiety [1,9]`、`capacity_used` / `wasted_capacity` 与 `:193-208` 一致。

表达式冲突 2 处（见 Required 2）：`net_profit`（`:620`）与 `return_ratio`（`:647`）仍写 `entry_fee(station_id)`，与规则 2、`:137`、`:170` 的 `R.entry_fee` 相反。上轮的窄输出域问题（`:603`、`:619`、`:646`、`:660`、`:675`、`:690`）仍在。

### 上一轮 blocking 复核

| # | 上轮 blocking | 结论 | 证据 |
|---|---|---|---|
| B1 | `entry_fee` 出处只迁移了一半 | **部分解决** | 已改：规则 2（`:26`）、`net_profit` 表达式与变量源（`:137`、`:143`）、`return_ratio`（`:170`）、交互表数据配置行（`:95`）、Cross-References（`:414`）、HUD 下游依赖行（`:338`）。未改：上游依赖表仍写 `get_station(id) → entry_fee`（`:324`），注册表两条表达式仍写 `entry_fee(station_id)`（`:620`、`:647`），AC 25 / F6 仍以档口措辞描述费用（见 Recommended 1）。 |
| B2 | 首局声称数值不变、override 无测试 | **已解决** | 边角案例 `:315` 改为「按本局 `RunLog.entry_fee` 计算，首局可传 `entry_fee_override = 40`，净利/回本率随 override 变化」；新增 AC 31（`:475`）：`entry_fee = 40`、总值 82 → `+42`、`2050`、`broke_even = true`，并断言同一 `eaten` 改为 85 不产出该组值。算术与 `first-run-onboarding.md:302,458` 一致。 |
| B3 | 实时 `running_net` 无开局初始化 | **已解决** | 规则 19（`:43`）把累加器交给餐局流程：开局 `−entry_fee`、每次提交 `+price(d)`、结算用同一公式复核；`run-flow.md:51` 规则 23 落实实现，`run-flow.md:411-413` AC 22/24 覆盖放弃丢弃与重开清零；`in-game-hud.md:170` 明确 HUD 只读快照、不读 `entry_fee`。本文件 States（`:47`）仍声明不持有局内状态，与委派一致。 |

### Required Before Implementation

1. **[BLOCKING] Dependencies（`:324`）— 上游数据配置行仍声明直接读入场费。** 该行仍写 ``get_station(id)` → `entry_fee``，与本文件规则 2（`:26`）、交互表（`:95`）、Cross-References（`:414`）矛盾。结算不调用 `get_station` 取费；它只读 `RunLog.entry_fee`，档口记录仅是餐局流程的默认来源。改为 ``get_station(id)` → 存在性与 `display_name`；`get_dish(id)` → `price`、`satiety`；`entry_fee` 由餐局流程冻结进 `RunLog``。这是上一轮 B1 点名但本轮未触碰的一行，按接口表实现依赖接线会重新引入直接读站点的路径。
2. **[BLOCKING] 注册表（`entities.yaml:620`、`:647`）— 公式表达式仍按站点取费。** `net_profit` 写 `total_value(R) - entry_fee(station_id)`，`return_ratio` 写 `total_value(R) / entry_fee(station_id)`；本文件已改为 `R.entry_fee`。注册表是机器可读契约，下游按它实现仍会绕过冻结日志，首局会算成 85。改为 `total_value(R) - entry_fee` / `total_value(R) / entry_fee`，并在 `notes` 注明 `entry_fee` 来自 `RunLog.entry_fee`。注册表编辑属编排会话；本条只报告，不代改。

### Recommended Revisions

1. **AC 25（`:466`）与 F6（`:435`）改用 `RunLog.entry_fee` 措辞。** 现在写的是「将档口 `entry_fee` 改为 37 / 0」。按规则 2，结算只读日志；应改为「把 F2 的 `RunLog.entry_fee` 设为 37（档口仍为 85）」与「F2 的日志，`RunLog.entry_fee = 0`」。AC 31 已覆盖 override 路径，故此条是精度问题而非覆盖率缺口。
2. **`running_net` / `shortfall` 变量表（`:257-261`）补 `entry_fee` 行。** 两条公式使用裸 `entry_fee`，但变量表只列 `k`、`running_net(k)`、`shortfall(k)`；补一行 `entry_fee | int | 0–INT32_MAX | RunLog.entry_fee（开局冻结）`，与 `net_profit` 的 `R.entry_fee` 行对齐。
3. **`end_reason` 出处（`:99`）。** 交互表仍把 `end_reason` 列在时间与回合名下；`time-and-turns.md:43` 明确它不保存、不排序 `end_reason`，`run-flow.md:40` 与注册表把它归餐局流程。删掉时间行的 `end_reason`，RunLog 表（`:65`）已正确。
4. **边界值域（`:180`、`:239`）。** `return_ratio` 的 `(0, +∞)` 排除空局（`total_value = 0` 时为 0，F3 `:297`）；`effective_unit_value` 的 `(0, price_max]` 排除零价菜，而同一块声明 `price ≥ 0`（`:126`）。改为 `[0, +∞)` 与 `[0, price_max]`，并说明空局显示 `0.0%`。
5. **注册表输出域宽于/窄于实际（`:603`、`:619`、`:646`、`:660`、`:675`、`:690`）。** `total_value`、`net_profit`、`running_net`、`shortfall`、`return_ratio`、`effective_unit_value` 的 `output_range` 窄于 GDD 声明的 int64 域；或加注「registry 记实用带，GDD 记防御上限」。编排会话处理。
6. **关闭已回答的 Open Questions。** OQ2（`:488`）→ `run-flow.md:36` 不提供提前结账；OQ3（`:489`）→ `records-and-milestones.md:31,309` 按净利排名；OQ4（`:490`）→ `entities.yaml` 的 `replacement_hint`（`:1128-1141`）。标注「已回答」即可。
7. **跨文档取费措辞（路由 `/consistency-check`）。** `data-config.md:81,182` 与 `run-flow.md:316,456`（OQ10）仍写「结算读取 `entry_fee`」。实际链路是餐局流程解析 `RunConfig.entry_fee_override` / `StationData.entry_fee` 并冻结进 `RunLog.entry_fee`（`run-flow.md:390` AC 1），结算只读日志。
8. **交互表硬度与缺行（`:91-107`）。** 音效反馈本文件记软（`:107`），但 `audio-feedback.md:110,354` 记「硬（事件契约）；产品意义上软」；首局引导记软（`:106`），但 `first-run-onboarding.md:126,363` 记「硬（B4）」。菜单波次只出现在 Dependencies（`:342`），场景与流程管理在规则/边角中多次引用却无交互行；补行或注明经餐局流程间接。
9. **Scope signal。** `systems-index.md:206` 记 `S`；本 GDD 有 6 个硬上游、8 个下游、九组公式，按评审尺度为 `L`。索引编辑属编排会话。
10. **音效事件的发射者与时机（新增，回应 `audio-feedback.md:546` OQ1）。** 本文件只定义事件语义，`audio-feedback.md` 假设「结算在 `settle` 成功、收据数据可用时发 `settlement_started`，结果语义同时到达」。补一句发射时机（`settle` 成功同帧发 `settlement_started`；`broke_even` / `loss` 同帧到达，由音频延后到滚动结束），即可关闭音频侧 OQ1。

### Nice-to-Have

- 不变式 1（`:273`）写 `capacity_used = RunLog.capacity_used`，读起来像自比较；写成 `Σ effective_satiety = RunLog.capacity_used`。
- `scene-flow.md` 被引为 `Run → Menu` payload 的承担者（`:31`、`:417`），但该文件 0 次提及本系统；补一条间接依赖说明或注明它只搬运 payload。
- Tuning Knobs / Player Fantasy：各补一行交叉引用即可，不要求新增整节。

### Verdict: NEEDS REVISION

设计本体与算术成立，B2、B3 已解决；B1 的公式、规则与 Cross-References 已迁移，但依赖接口表与注册表表达式仍保留站点取费，属同一契约的未完成迁移。两条 blocking 都是单点文本/注册表修改，不需要重新设计。

Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 5734623b4a60d9072657ef631c8ae61f05359f21
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none（lean 模式，Phase 3b 未运行）
Blocking items: 0 | Recommended: 10
Summary: 上一轮的两条 blocking 均已解决。上游数据配置行改为 `` `get_station(id)` → 存在性与 `display_name`；`entry_fee` 由餐局流程冻结进 `RunLog` ``（`:324`，交互表 `:95` 同步），注册表 `net_profit` / `return_ratio` 的表达式改为使用冻结的 `entry_fee`，`notes` 注明来源为 `RunLog.entry_fee`（`entities.yaml:612-626`、`:641-653`）。全文已无站点取费路径；九组公式、全部算例与边界规则本轮未改动且仍自洽。剩余十项均为措辞精度与跨文档同步，不影响实现。
Prior verdict resolved: Yes — 上轮 B1（`entry_fee` 迁移）与 B2（首局 override）已全部解决；B3（实时净利）维持已解决。

### 复审背景

- 触发：`cf14f37`（7 份 Needs Revision GDD 的 blocking 修复）改动了本文件、注册表与多份关联 GDD。
- 复审范围：**全文重读**（493 行，按标题分段），未按 diff 跳读。
- 本轮实际改动：上游数据配置依赖行（`:324`）；注册表 `net_profit` 与 `return_ratio` 的表达式与 `notes`。正文、公式、边角案例、AC 未改。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统必需：它定义求和、净利、回本判定、回本率、容量、腻度、单位价值与实时净值九组数值规则。

Advisory 缺口（不阻塞，与上两轮相同）：Player Fantasy、Tuning Knobs 无独立标题。`## Game Feel`（`:375-382`）实质覆盖前者；唯一运行期旋钮是数据驱动的 `entry_fee`，后者可由一行交叉引用补齐。

### Dependency Graph

15 份被引 GDD 全部在盘，无断链：`data-config.md`、`dish-values.md`、`cloying-and-taste.md`、`stomach-capacity.md`、`time-and-turns.md`、`run-flow.md`、`meta-upgrades.md`、`records-and-milestones.md`、`ui-flow.md`、`in-game-hud.md`、`first-run-onboarding.md`、`audio-feedback.md`、`save-system.md`、`menu-waves.md`、`scene-flow.md`。Dependencies 下游表列出的 8 份 GDD 全部回指本系统。

跨文档事实复核：
- ✓ `run-flow.md:40,51,391`：`RunLog.entry_fee` 由餐局流程从 `RunConfig.entry_fee_override`（`null` 时取 `StationData.entry_fee`）解析并冻结，AC 1 断言该等式；规则 23 以 `−entry_fee` 初始化累加器。与本文件规则 2、规则 19 咬合。
- ✓ `first-run-onboarding.md:38,297-302,458`：教学价 40、总值 82、净利 +42、`return_percent_x10 = 2050`，与 AC 31（`:475`）完全一致；`ui-flow.md:497` 的 `ReceiptScreen` 渲染教学价说明，标签归属不在结算。
- ✓ `audio-feedback.md:38,63-65`：`settlement_started` 先播、`broke_even` / `loss` 在数字滚动结束时起播、结果三者互斥；与本文件 `:340`、`:371` 一致。
- ✓ `in-game-hud.md:170,471`：HUD 经快照读 `running_net` / `shortfall`，不读 `entry_fee` 自行求和，开局显示 `−85`。
- ✓ `data-config.md:81,182` 的取费归属仍写「结算读取 `entry_fee`」；`run-flow.md:317,457`（OQ10）仍写「本设计中入场费由结算读取」。两者都已被 `run-flow.md:391` AC 1 与本文件规则 2 取代——跨文档措辞同步（见 Recommended 7），不阻塞本文件。

### Registry Values

对全文做了 131 行 registry 术语扫描（`total_value`、`net_profit`、`broke_even`、`return_ratio`、`effective_unit_value`、`running_net`、`shortfall`、`capacity_used`、`wasted_capacity`、`cloy_extra_total`、`unit_value`、`calibration_ratio`、`extra_satiety`、`effective_satiety`、`end_reason`、`capacity_total`、`replacement_hint`、`first_run_entry_fee`、`station_canteen_entry_fee`、`entry_fee`，大小写不敏感，`_` 按 `[_ ]`）。

数值无冲突：`station_canteen_entry_fee = 85`、`first_run_entry_fee = 40`、`opt_value = 104`、`calibration_ratio ≈ 1.224`、`base_capacity = 20`、`cloy_cap = 3` 均与全文及算例一致。

表达式冲突已清零：`net_profit` 为 `total_value(R) - entry_fee`、`return_ratio` 为 `total_value(R) / entry_fee`，`notes` 均注明「`entry_fee` is `RunLog.entry_fee`（frozen at run start）… settlement never re-reads the station」——与规则 2、`:137`、`:170` 一致。

上轮的窄输出域问题仍在（`total_value`、`net_profit`、`running_net`、`shortfall`、`return_ratio`、`effective_unit_value`），见 Recommended 5。

### 上一轮 blocking 复核

| # | 上轮 blocking | 结论 | 证据 |
|---|---|---|---|
| B1 | `entry_fee` 出处只迁移了一半 | **已解决** | 上游依赖行 `:324` 改为「存在性与 `display_name`；`entry_fee` 由餐局流程冻结进 `RunLog`」；交互表 `:95` 与 Cross-References `:414` 同义；注册表 `entities.yaml:623,650` 的两条表达式改为使用冻结的 `entry_fee`，`:624,651` 的 `notes` 注明来源。全文 `entry_fee` 提及逐行核对（27 处），已无「结算读档口」路径。 |
| B2 | 首局声称数值不变、override 无测试 | **已解决（维持）** | 边角 `:315` 按 `RunLog.entry_fee` 计算；AC 31（`:475`）覆盖 40 → `+42` / `2050` 并断言改回 85 不产出该组值。 |
| B3 | 实时 `running_net` 无开局初始化 | **已解决（维持）** | 规则 19（`:43`）把累加器交给餐局流程；`run-flow.md:51` 规则 23 与 `in-game-hud.md:170` 两侧一致。 |

### Required Before Implementation

- none

### Recommended Revisions

1. **AC 25（`:466`）与 F6（`:435`）改用 `RunLog.entry_fee` 措辞。** 二者仍写「将档口 `entry_fee` 改为 37 / 0」。结算只读日志，单元测必须改日志字段；现在是上轮迁移唯一残留的旧措辞。AC 31 已覆盖 override 路径，故属精度问题而非覆盖率缺口。
2. **`running_net` / `shortfall` 变量表（`:257-261`）补 `entry_fee` 行。** 两条公式使用裸 `entry_fee`，表里只有 `k`、`running_net(k)`、`shortfall(k)`。
3. **`end_reason` 出处（`:99`、`:328`）。** 仍列在时间与回合名下；`run-flow.md:40` 与注册表把它归餐局流程，`RunLog` 表（`:65`）已正确。
4. **边界值域（`:180`、`:183`、`:239`）。** `return_ratio` 的 `(0, +∞)` 排除空局（F3 时总值 0 → 0，本文件 `:297`）；`effective_unit_value` 的 `(0, price_max]` 排除零价菜，而同块声明 `price ≥ 0`（`:126`）。改为 `[0, +∞)` / `[0, price_max]`，并说明空局回本率显示 `0.0%`。注册表 `return_ratio [0, 999]`、`effective_unit_value [0, 9999]` 已包含 0，反而比本文件正确。
5. **注册表输出域窄于本文件声明的 int64 域。** `total_value`、`net_profit`、`running_net`、`shortfall`、`return_ratio`、`effective_unit_value` 的 `output_range` 应加注「registry 记实用带，GDD 记防御上限」。注册表编辑属编排会话。
6. **关闭已回答的 Open Questions。** OQ2（`:488`）→ `run-flow.md:36` 不提供提前结账；OQ3（`:489`）→ `records-and-milestones.md:31` 按净利排名；OQ4（`:490`）→ `replacement_hint`（`entities.yaml`）。标注「已回答」即可。
7. **跨文档取费措辞（路由 `/consistency-check`）。** `data-config.md:81,182` 与 `run-flow.md:317,457`（OQ10）仍写「结算读取 `entry_fee`」，与 `run-flow.md:391` AC 1、本文件规则 2 相反。这是两份上游 GDD 的陈旧行，不影响本文件实现。
8. **交互表硬度与缺行（`:91-107`）。** 音效反馈本文件记软（`:107`），`audio-feedback.md:111` 记「硬（事件契约）；产品意义上软」；首局引导记软（`:106`），`first-run-onboarding.md:126` 记「硬（B4）」。菜单波次只在 Dependencies（`:342`），场景与流程管理在规则与边角中多次引用却无交互行。
9. **Scope signal。** `systems-index.md:206` 记 `S`；本 GDD 有 6 个硬上游、8 个下游、九组公式，按评审尺度为 `L`。索引编辑属编排会话。
10. **音效事件的发射时机（回应 `audio-feedback.md:547` OQ1）。** 本文件 `:340` 规定了结果 cue 何时起播，但未写事件何时发射。补一句「`settle` 成功、收据数据可用的同一帧发 `settlement_started`；`broke_even` / `loss` 同帧到达，由音频延后到滚动结束」，即可关闭音频侧 OQ1。

### Nice-to-Have

- 不变式 1（`:273`）写 `capacity_used = RunLog.capacity_used`，读起来像自比较；写成 `Σ effective_satiety = RunLog.capacity_used`。
- `:346` 的索引说明已过期：它说 `systems-index.md` 的 HUD 上游「漏了结算」，但 `:71` 与 `:181` 现已列出「结算与入场费」；该段可删或改为已补正。
- `scene-flow.md` 描述了 `Run → Menu` 的 `RunResult` 交接（规则 6）却从未点名本系统；补一条间接依赖说明即可。
- 本文件头 `:3` 的 `Status: In Design` 与 `systems-index.md:67` 的 `In Review` 不一致；索引侧已由编排会话推进，文件头可同步。
- Tuning Knobs / Player Fantasy：各补一行交叉引用即可，不要求新增整节。

### Verdict: APPROVED

设计本体、九组公式、全部算例（104−85=+19、54−85=−31、82−40=+42 / 2050、F4 的 10/3=3.33、F5 恰好回本）与边界规则自洽；`entry_fee` 契约在正文、依赖表与注册表三处已统一为「餐局流程冻结 → 结算只读 `RunLog`」。剩余十项为措辞精度与跨文档同步，不构成实现阻塞。

Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/run-flow.md ea6e49d52b2a08d72dc75c2b295cf54ba5b88279
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 1cafff5e7ad8aab8912e716d98b69cea02cbbc43
Reviewed-Content-Hash: design/gdd/audio-feedback.md ffb1c54a84af1da89b0228395ee86e89b3e87e30
Reviewed-Content-Hash: design/gdd/in-game-hud.md 654e48d14dbf85fdbab3b923e590a4f356dd0218
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/systems-index.md 56e9869c70371816e88ed49f24735f67ab3d199d
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 84e5c38fc40fb2a75d8ee608c911bdebf369c5a1
Reviewed-Content-Hash: design/gdd/ui-flow.md 9c709252b4c631e8fb50d8a2f35a8fb394f230d7
Reviewed-Content-Hash: design/gdd/menu-waves.md 4e4854fb6a38c7b0989503d8c261b69c5746adbe
Reviewed-Content-Hash: design/gdd/save-system.md f22c2228d60be43669125831bccea834e94d6026
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
