# Review Log — 存档（Save）

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 4 | Recommended: 5 | Nice-to-have: 4
Prior verdict resolved: First review

Summary: The envelope design, the five-state load result, the per-owner
namespaces and the "store ids, not snapshots" boundary are sound and unusually
precise. The blocking problems are all at the edges of that model: one load path
returns two different outcomes for the same input, the settlement-wallet
write-failure policy is asserted in Dependencies but absent from the core rules
while `meta-upgrades` waits for it, one acceptance criterion contradicts the
rule it is meant to test, and `Blocked` conflates "readable but locked" with
"unreadable" — a conflation whose UI recovery action would quarantine a file
another live instance owns.

### Completeness: 6/6 required sections present (standard tier)

Required at `standard`: Overview, Detailed Rules (`## Detailed Design`),
Formulas, Edge Cases, Dependencies, Acceptance Criteria — all present.
Formulas is required here because the system defines numeric rules (load
decision, migration chain, size cap, slot count), and all four formula blocks
carry variables, ranges and examples.

Advisory gaps (do not block at `standard`):
- **Player Fantasy** — absent as a heading; `## Game Feel` covers the same
  intent ("目标手感是「不存在」").
- **Tuning Knobs** — absent as a section; the constants are present but
  scattered (`MAX_SLOTS`, `SAVE_BACKUP_COUNT`, `SAVE_BYTES_MAX`,
  `SAVE_SCHEMA_VERSION`, the p95 budgets).

### Dependency Graph

- ✓ `design/gdd/data-config.md` — exists (upstream, hard: `Ready`, id lookups, `max_level`)
- ✓ `design/gdd/meta-upgrades.md` — exists (downstream, hard: `progression`)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (downstream, hard: `unlocks`)
- ✓ `design/gdd/records-and-milestones.md` — exists (downstream, hard: `records`)
- ✓ `design/gdd/ui-flow.md` — exists (downstream, hard UI)
- ✓ `design/gdd/scene-flow.md` — exists (timing only, not a dependency)
- ✗ `design/gdd/online-challenge.md` — NOT FOUND; `systems-index.md` #19 is
  `Not Started`. Declared soft/未写 in this GDD, so this is an expected
  forward reference, not a broken link.
- Cross-reference only (not dependency rows): `settlement-and-entry-fee.md`,
  `cloying-and-taste.md`, `menu-waves.md` — all exist.

### Registry Values

Searched the whole document for the registry entries it owns or references:
`load_outcome`, `migrate`, `save_bytes`, `slots_available`,
`save_schema_version`, `save_bytes_max`, `save_max_slots`, `milestone_met`,
`effective_unlocks`, `run_id_alloc`, `entry_fee`, `total_value`, `net_profit`,
`return_ratio`, `upgrade_price`, `capacity_total`, `wallet_after_run`.

- **Matching:** `load_outcome` five values (registry `entities.yaml:778`);
  `migrate` expression; `save_bytes` expression
  `B_base + n_upg*U + n_sta*S + n_dish*D + n_rec*R + n_ms*M + ext_bytes`;
  `slots_available = MAX_SLOTS − used_slots`; constants
  `save_schema_version = 1`, `save_bytes_max = 262144`,
  `save_max_slots = 1`. No value conflicts.
- **Drift:** `save_backup_count` (`SAVE_BACKUP_COUNT = 1`, GDD rule 9) has no
  registry entry — Recommended 1. `save_bytes` lower bound is `0` in both the
  GDD (line 203) and the registry (`entities.yaml:817`) although `B_base = 1024`
  — Recommended 4.
- `milestone_met` and `effective_unlocks` are referenced by this system but
  their values are owned by #14 / #13; the GDD does not restate them, so no
  cross-check conflict.

### Required Before Implementation

1. **[BLOCKING] Formulas vs Detailed Rules — a failed primary migration returns
   `Recovered` in the formula but `Blocked` in the rules (lines 39, 138–140,
   176).**
   `load_outcome` line 139 falls through from the `Migrated` branch to the
   `Recovered` branch when `migrate(P, …) = error`: if the backup is readable
   and migratable, the result is `Recovered`. Rule 13 (line 39) says
   "任何一步失败，载入结果为 `Blocked`，原文件保持不变"; the `migrate` Output
   Range (line 176) repeats it. Same input, two outcomes — and the difference is
   "silently use the backup" vs "refuse everything". The existing edge case at
   line 232 only covers a *corrupt* primary (unreadable), not a readable
   primary whose migration step fails. Fix: terminate the `Migrated` branch as
   `Blocked` on error (or restrict `Recovered` to `¬readable(P)`) and add an
   AC with primary readable + migration failing + backup valid.

2. **[BLOCKING] Detailed Rules — the settlement wallet (`progression`) write
   failure has no rule, and the Dependencies table contradicts the state machine
   (lines 43–44, 74–75, 102, 267).**
   Rule 17 covers purchases (roll back memory); Rule 18 covers records (keep
   memory, retry). A settlement's wallet earnings are neither: the failure row
   at line 75 limits the retry branch to "事务来自纪录或迁移", and the `commit`
   interface (line 102) says only "回滚或标记 dirty". Yet the Dependencies row
   at line 267 states "入账写盘失败时 `progression` 可进入 dirty 并在下次
   `flush` 重试（与购买事务的回滚策略不同）", and `meta-upgrades.md` rule 6
   (line 37) explicitly says "需在存档 GDD 补一条规则，见待回改". A programmer
   cannot decide whether wallet earnings roll back or retry. Fix: add a rule
   alongside Rule 18 ("结算入账失败可重试"), and change the failure-source
   wording in the state table and `commit` to "结算入账 / 纪录 / 迁移" (retry)
   vs "购买" (rollback).

3. **[BLOCKING] Acceptance Criteria — AC 13 contradicts Rule 18 and AC 16 on
   whether memory reverts after a settlement write failure (lines 374, 377 vs
   44).**
   AC 13 says "注入写失败时三者全部不生效，内存回到事务前". Rule 18 says
   "写盘失败时保留内存值"; AC 16 says "纪录在界面上按新值显示… 下一次成功写入
   后磁盘上的纪录与新值一致". The same injected failure cannot both revert and
   retain memory. AC 13's own closing note says the settlement-produced unlock
   does not exist in MVP/VS, but its wallet + records assertion is live. Fix:
   make AC 13 assert "磁盘不发生写入；内存按规则 18 保留并标记 dirty（未来
   解锁同样重试）", or scope AC 13 to a purchase transaction.

4. **[BLOCKING] States / UI Requirements — `Blocked` conflates "readable but
   locked" with "unreadable", and its recovery action would quarantine a file a
   live instance owns (lines 47, 65, 320).**
   Rule 21 (line 47) says a live second instance gets `Blocked`
   "（可只读浏览，写入禁用），绝不覆盖对方的文件". But the load-result table
   (line 65) and UI Requirements 3 (line 320) give every `Blocked` cause the
   same forced modal with only "退出" and "新建生涯（旧档保留）". Two defects
   follow: (a) the promised read-only browsing has no exit, and `ui-flow.md`
   rule 20 (line 79) additionally disables every other menu control until
   `Blocked` is resolved; (b) for the "另一个游戏实例正在使用这份存档" cause,
   choosing "新建生涯" renames/quarantines the file the live instance is still
   writing — contradicting "绝不覆盖对方的文件". Fix: split "lock live, file
   readable" out of `Blocked` (a sixth `ReadOnly` outcome, or keep `Blocked`
   but branch behaviour by `reason`). The lock case offers "只读浏览 / 退出" and
   no "新建生涯"; only "无可用副本 / 版本过高" offers "新建生涯（旧档保留）".
   Synchronise UI Requirements 3 and `ui-flow.md` rule 20. If a sixth outcome is
   chosen, update the registry `load_outcome` entry (it currently pins 5 values).

### Recommended Revisions

1. **[RECOMMENDED] Registry — `save_backup_count` is missing.**
   Rule 9 defines `SAVE_BACKUP_COUNT = 1`, but `entities.yaml` registers only
   `save_schema_version`, `save_bytes_max` and `save_max_slots` from this
   system. Add the constant (the registry is not editable in this review; flag
   for the orchestration session).
2. **[RECOMMENDED] Open Questions — OQ 8 is answered.**
   `ui-flow.md` rule 24 (line 87) now declares "界面流程拥有设置屏与机器级设置
   模型", which answers OQ 8 (line 413, "设置文件由谁拥有"). Mark it answered
   and update the owner/target; the current "界面流程 GDD 完成时" target has
   passed.
3. **[RECOMMENDED] Cross-References — the "待回改" list is stale (lines
   342–348).**
   Items 1–4 are already applied: `data-config.md:183`,
   `settlement-and-entry-fee.md:340`, `cloying-and-taste.md:229`,
   `menu-waves.md:229`. Item 5 is also done: `systems-index.md:60` already
   reads `Designed`. Remove the list or mark it complete so the orchestration
   session does not re-apply it.
4. **[RECOMMENDED] Formulas — `save_bytes` lower bound is wrong (line 203).**
   Output Range says `0–SAVE_BYTES_MAX`, but `B_base = 1024`, so the minimum is
   1024. State `≥ 1024` or `[1024, SAVE_BYTES_MAX]`. The registry mirrors the
   same `0` (`entities.yaml:817`) and should be corrected with it.
5. **[RECOMMENDED] Detailed Rules — Rule 2's non-persisted list should include
   `base_capacity` (line 28).**
   data-config's approved fourth record `BalanceData` is now a hard upstream
   field (`data-config.md:28, 58`). Rule 2's principle already covers it, but
   the illustrative list is the thing a reader checks against the approved
   schema; add `base_capacity` (or say the list is illustrative, not
   exhaustive).

### Nice-to-Have

1. **`n_sta` / `n_dish` ranges omit retained unknown ids (lines 193, 195).**
   Rule 14 preserves unknown station/dish ids in `unlocks`, but only `n_upg`
   says "含保留的未知 id" (line 191). Either say the same for `n_sta` /
   `n_dish`, or note that the 256 KiB cap absorbs retention.
2. **Tuning Knobs (advisory at `standard`).** A short section listing
   `MAX_SLOTS` (1–8), `SAVE_BACKUP_COUNT`, `SAVE_BYTES_MAX`, the p95 budgets and
   `SAVE_SCHEMA_VERSION` with safe ranges would make the constants
   discoverable; not blocking at this tier.
3. **Player Fantasy (advisory at `standard`).** `## Game Feel` already states
   the intent; either add a one-paragraph Player Fantasy or cross-reference the
   Game Feel section so the standard checklist resolves.
4. **Cross-document wording — "存档规则 80".**
   `ui-flow.md:59/616` and `unlocks-and-stations.md:55` cite "存档规则 80"; this
   GDD has 25 numbered rules, so the citation means line 80 (the
   single-transaction paragraph). Suggest "存档 GDD 第 80 行" (the form
   `records-and-milestones.md:36` already uses) so it is not read as rule #80.

### Scope Signal

**Rough scope signal: L** — 4 formula blocks, 1 upstream + 6 hard downstream
systems, and three ADR-stage decisions (serialization format OQ 2, save path /
loader OQ 3, write-lock implementation OQ 9; Steam Cloud OQ 4 later).
`systems-index.md:200` rates the design effort `S`; this review signals **L**
and the producer should verify before sprint planning.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (currently `Designed`;
`systems-index.md:60`). Per the parallel constraint this review did not modify
the index; the orchestration session updates it. The four blocking items are
rule/AC text and one state split, not a redesign of the envelope — they are
fixable in one revision pass, after which a re-review decides Approved.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/save-system.md 7869c6057c8c752726d42d116b5cab3849843fa2
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md bcde83821bc73ad7c67a24d38a8659ce2d53693b
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md 0f58ea62a2e8fa9ace2effe652cc644acab89d5b
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/menu-waves.md 3c802aab667b644e6c8118e8f63b33a1cbdd87f4
Reviewed-Content-Hash: design/gdd/systems-index.md 8f9bdb61bf682a7e4d7191933202f9efb3d68cde
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
```

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 2 | Recommended: 6 | Nice-to-have: 4
Prior verdict resolved: No — 3 of 4 blocking items resolved; item 4 partially

Summary: The fix commit resolved the migration-failure fall-through, the
settlement-wallet retry policy and the AC 13 contradiction cleanly, and the
`ReadOnly` split is wired through the load table, formula, UI requirements,
`ui-flow` and the registry. Two defects remain: `load_outcome` still cannot
reach `Migrated` for the version-0 fixture its own ACs and migrate section
require, and `ReadOnly` is defined by "版本 = 当前" in the rule/table/formula
but by "主档可读" in rule 21, the edge case, UI Requirements 3, AC 23 and
`ui-flow` — leaving a live-lock/version-skew path that returns `Blocked` and
offers "新建生涯" over a file another live instance owns.

### Prior Blocking Items — Status

1. **RESOLVED** — failed primary migration. `load_outcome` now terminates the
   `Migrated` branch as `Blocked` on error (lines 140–141), rule 13 agrees, and
   AC 29 covers readable primary + migration failure + valid backup.
2. **RESOLVED** — settlement wallet retry. Rule 18 now covers `progression`,
   `records` and unlocks; the state table (line 76), `commit` (line 103), edge
   case (line 238) and Dependencies (line 270) all use the same
   结算入账 / 纪录 / 迁移 → dirty (retry) vs 购买 → rollback split.
   `meta-upgrades` rule 6 and `records` rule 16 match.
3. **RESOLVED** — AC 13 now asserts "磁盘不发生写入，内存按规则 18 保留并标记
   dirty", matching rule 18 and AC 16; the MVP/VS unlock note is consistent
   with `unlocks-and-stations` rule 15.
4. **PARTIALLY RESOLVED** — the sixth state exists and is wired end to end, but
   its definition is inconsistent across the document. See blocking item 2.

### Completeness: 6/6 required sections present (standard tier)

Required at `standard`: Overview, Detailed Rules (`## Detailed Design`),
Formulas, Edge Cases, Dependencies, Acceptance Criteria — all present.
Formulas is required because the system defines four numeric rules.

Advisory gaps (do not block at `standard`):
- **Player Fantasy** — absent as a heading; `## Game Feel` covers the intent.
- **Tuning Knobs** — absent as a section; constants remain scattered.

### Dependency Graph

- ✓ `design/gdd/data-config.md` — exists (upstream, hard)
- ✓ `design/gdd/meta-upgrades.md` — exists (downstream, hard)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (downstream, hard)
- ✓ `design/gdd/records-and-milestones.md` — exists (downstream, hard)
- ✓ `design/gdd/ui-flow.md` — exists (downstream, hard UI)
- ✓ `design/gdd/scene-flow.md` — exists (timing only)
- ✗ `design/gdd/online-challenge.md` — NOT FOUND; `systems-index.md` #19 is
  `Not Started`. Declared soft/未写, so an expected forward reference.

### Registry Values

Searched the whole document for the registry entries it owns or references:
`load_outcome`, `migrate`, `save_bytes`, `slots_available`,
`save_schema_version`, `save_bytes_max`, `save_max_slots`, `milestone_met`,
`effective_unlocks`, `entry_fee`, `total_value`, `net_profit`.

- **Matches:** `load_outcome` now carries six values (registry line 778) and the
  registry expression mirrors the GDD formula; `migrate`, `slots_available`,
  `save_schema_version = 1`, `save_bytes_max = 262144`,
  `save_max_slots = 1` all agree.
- **Conflicts:** `save_backup_count` (rule 9) still has no registry entry —
  Recommended 1. `save_bytes` output range is `0–SAVE_BYTES_MAX` in the GDD
  (line 206) and `[0, 262144]` in the registry (line 817) although
  `B_base = 1024` — Recommended 2. The registry `load_outcome` note says
  "live lock over a readable current file"; the GDD's rule 21 / AC 23 say
  readable alone — blocking item 2.

### Required Before Implementation

1. **[BLOCKING] Formulas vs Acceptance Criteria — `load_outcome` cannot reach
   `Migrated` for the version-0 fixture (lines 140–141, 154–155, 174, 181,
   370–371).**
   Both migration branches require `1 ≤ version(P) < V`, but `version(x)`'s
   declared range is `0–INT32_MAX` with "0 只用于迁移测试夹具" (line 155),
   `migrate`'s `v_src` range starts at 0 (line 174), and both the migrate
   example (line 181) and AC 8 (line 370) expect a `version = 0` fixture to
   produce `Migrated`; AC 9 (line 371) expects that fixture to return
   `MigrationError`. Under the formula, a readable v0 file fails the guard, is
   never passed to `migrate`, and falls to `Recovered`/`Blocked`. Second,
   independent cause: `valid(x)` (line 154) does not say which schema it
   validates against. Read against the current v1 schema, the v0 fixture is
   invalid (it lacks `career_id`, `slot_id`, `records`), so it is diverted to
   the corrupt path before migration regardless of the guard. Fix: define
   `valid(x)` as "structurally legal under the schema of its declared
   `version(x)`", and use `version(P) < V` (no lower bound beyond the type's 0)
   in both migration branches.

2. **[BLOCKING] States / UI Requirements — the `ReadOnly` definition is
   inconsistent, and the residual path still offers to quarantine a live
   instance's file (lines 36, 47, 65–66, 135–136, 249, 323–324, 394;
   `ui-flow.md:79`).**
   Rule 10, the load-result table (line 65) and the formula (line 135) require
   "主档可读且版本 = 当前" for `ReadOnly`, and the formula's `L = live` fallback
   (line 136) sends every other lock case to `Blocked`. But rule 21 (line 47),
   the edge case (line 249), UI Requirements 3 (line 323), AC 23 (line 394) and
   `ui-flow.md:79` all say a live lock over a readable file is `ReadOnly`,
   without a version condition. Consequence: instance A running an older build
   holds a live lock; instance B (new build) sees a readable primary at
   `version < V`; the formula returns `Blocked`, and UI Requirements 4 offers
   only "退出" and "新建生涯（旧档保留）" — choosing the latter renames the file
   A is still writing. That is the exact hazard the sixth state was added to
   remove, narrowed to version skew. Fix: make `ReadOnly` = `L = live ∧
   readable(P)` (browse without migration, never write) and keep `Blocked` for
   `L = live ∧ ¬readable(P)`; or keep the formula and branch `Blocked`'s UI by
   reason so a lock-caused `Blocked` never offers "新建生涯". Then align rule
   10, rule 21, the load table, the formula, the edge case, UI Requirements 3–4,
   AC 23 and the registry note to whichever definition is chosen.

### Recommended Revisions

1. **[RECOMMENDED] Registry — `save_backup_count` still missing.** Rule 9
   defines `SAVE_BACKUP_COUNT = 1`; `entities.yaml` still registers only
   `save_schema_version`, `save_bytes_max` and `save_max_slots` from this
   system. (Carried from the prior review; registry not editable here.)
2. **[RECOMMENDED] Formulas — `save_bytes` lower bound is still wrong.** GDD
   line 206 says `0–SAVE_BYTES_MAX`, registry line 817 says `[0, 262144]`, but
   `B_base = 1024`. Both should start at 1024.
3. **[RECOMMENDED] Detailed Rules — Rule 2's illustrative list still omits
   `base_capacity` (line 28).** It is a current hard upstream field
   (`data-config.md:59, 117`); add it or mark the list illustrative.
4. **[RECOMMENDED] Open Questions — OQ 8 is answered.** `ui-flow.md` rule 24
   owns the settings screen and model; mark OQ 8 (line 418) answered and update
   its owner/target.
5. **[RECOMMENDED] Cross-References — the `待回改` list is stale (lines
   346–352).** Items 1–4 are applied; item 5 describes the index as `Designed`
   while it now reads `In Review`. Also `meta-upgrades.md:37` still says "需在
   存档 GDD 补一条规则，见待回改" — Rule 18 now exists, so that parenthetical
   can be dropped.
6. **[RECOMMENDED] Edge Cases — line 238 still says only "纪录类事务保留内存".**
   Rule 18 now also covers settlement wallet (`progression`) and unlocks;
   restate the retry branch as "结算入账 / 纪录 / 迁移" to match the rule and
   the state table.

### Nice-to-Have

1. **`n_sta` / `n_dish` / `n_rec` ranges omit retained unknown ids (lines 197,
   199, 200).** Rule 14 preserves unknown ids in all three namespaces, but only
   `n_upg` says "含保留的未知 id"; the 256 KiB cap absorbs the overage, so
   either say so or widen the ranges.
2. **Player Fantasy / Tuning Knobs (advisory at `standard`).** Add a short
   cross-reference or section so the standard checklist resolves.
3. **AC 13's "磁盘不发生写入" (line 378) is imprecise.** An injected failure
   during the temp-file write still touches disk before failing; "主档与备份
   哈希不变" matches AC 11 and is directly assertable.
4. **Cross-document wording — "存档规则 80".** `ui-flow.md:59/616` and
   `unlocks-and-stations.md:55` still cite it; this GDD has 25 numbered rules,
   so the citation means line 80. Use "存档 GDD 第 80 行".

### Scope Signal

**Rough scope signal: L** — four formula blocks, one upstream + six hard
downstream systems, and three ADR-stage decisions (serialization format OQ 2,
save path / loader OQ 3, write-lock implementation OQ 9; Steam Cloud OQ 4
later). `systems-index.md:200` rates the design effort `S`; this review
signals **L** and the producer should verify before sprint planning.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (`systems-index.md:60`
currently reads `In Review`). Per the parallel constraint this review did not
modify the index; the orchestration session updates it. Both blocking items are
small text-level fixes to the load formula and the `ReadOnly` definition — no
envelope or state-model redesign — so one revision pass should clear the way
for a re-review.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/save-system.md 5d80a9bdcd8ffc603a0fa7d635fc736b40a4fe04
Reviewed-Content-Hash: design/registry/entities.yaml 802229dacc5554b1509f98fd6d13ec637c576f7c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/meta-upgrades.md 7dc02ff087d4ac059d4a81caa1b0c375483113ed
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md bcde83821bc73ad7c67a24d38a8659ce2d53693b
Reviewed-Content-Hash: design/gdd/records-and-milestones.md ca1ecf5a176da065f59c2b3b8261174fb1479571
Reviewed-Content-Hash: design/gdd/ui-flow.md 8f8255543b934bb5bfc11e7cbe914b668743e474
Reviewed-Content-Hash: design/gdd/scene-flow.md e1d10bd89706bef8f4cffa93065a9206fb1f54b6
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md c4175b09389504117ef102bad74116cead0f044e
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 38ebc87b9b529f9d32ce4fba2202411ce7891876
Reviewed-Content-Hash: design/gdd/menu-waves.md a9ed433be48d155399168a5519ae0f3c99c7d9f8
Reviewed-Content-Hash: design/gdd/systems-index.md 01259cc27663f7a47234e09c674e143fa3e2ba5c
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
```

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 2 | Recommended: 8 | Nice-to-have: 5
Prior verdict resolved: Partially — prior blocking 1 resolved; prior blocking 2 unified in
wording but two consequences of the chosen fix remain (new blocking item 1, residual
blocking item 2).

Summary: The version-0 migration path is now correct end to end — `valid(x)` validates
against the schema of the file's declared `version(x)` (line 154) and both migration
branches use `version(P) < V` with no lower bound (lines 140–141), so AC 8/9 and the
migrate example are reachable. The settlement-retry policy and AC 13 are also clean.
The `ReadOnly` definition is now stated identically in rule 10, rule 21, the load
table, the formula, the edge case, UI Requirements 3, AC 23, `ui-flow` and the registry
note. Two defects remain, both consequences of that unification: `ReadOnly` is defined
over `readable(P)` at *any* version while also forbidding migration, which leaves no
defined data source for browsing and contradicts rule 12 for a future-version file; and
a lock-caused `Blocked` still offers "新建生涯（旧档保留）", the action rule 21 forbids
against a live instance's files.

### Prior Blocking Items — Status

1. **RESOLVED** — `load_outcome` could not reach `Migrated` for the version-0 fixture.
   `valid(x)` (line 154) is now "structurally legal under the schema of its declared
   `version(x)`", and lines 140–141 test `version(P) < V` with no `1 ≤` floor. The
   `migrate` `v_src` range (line 174), the example (line 181), AC 8 (line 370) and
   AC 9 (line 371) now agree; AC 29 (line 403) covers the readable-primary /
   failed-migration / valid-backup case.

2. **PARTIALLY RESOLVED** — `ReadOnly` is now defined as `L = live ∧ readable(P)` in
   every site that round 2 listed: rule 10 (line 36), the load table (line 65), the
   formula (lines 135–136), rule 21 (line 47), the edge case (line 249), UI
   Requirements 3 (line 323), AC 23 (line 394), `ui-flow.md:81/401/503` and the
   registry note (`entities.yaml:780`). The version-skew hazard round 2 named is gone.
   Two consequences remain: the "any version" widening leaves `ReadOnly` without a
   defined rendering for `version(P) ≠ V` (blocking 1), and the lock-caused `Blocked`
   still carries the quarantine action (blocking 2).

**This round's four named fixes — verified:** per-version `valid(x)` ✓ (line 154),
settlement/records/unlock dirty-retry ✓ (rule 18 line 44; state table line 76;
`commit` line 103; Dependencies line 270), AC 13's retained-memory clause ✓ (line 378),
`ReadOnly` wording ✓ (all sites above). The Fresh-baseline backfill is present
(edge case line 232) and matches `unlocks-and-stations.md:53`, but see Recommended 6–7.

### Completeness: 6/6 required sections present (standard tier)

Required at `standard`: Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge
Cases, Dependencies, Acceptance Criteria — all present. Formulas is required because the
system defines four numeric rules.

Advisory gaps (do not block at `standard`):
- **Player Fantasy** — absent as a heading; `## Game Feel` covers the intent.
- **Tuning Knobs** — absent as a section; constants remain scattered.

### Dependency Graph

- ✓ `design/gdd/data-config.md` — exists (upstream, hard)
- ✓ `design/gdd/meta-upgrades.md` — exists (downstream, hard)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (downstream, hard)
- ✓ `design/gdd/records-and-milestones.md` — exists (downstream, hard)
- ✓ `design/gdd/ui-flow.md` — exists (downstream, hard UI)
- ✓ `design/gdd/scene-flow.md` — exists (declared non-dependency, timing only)
- ✗ `design/gdd/online-challenge.md` — NOT FOUND; `systems-index.md:75` #19 is
  `Not Started`. Declared soft / 未写, so an expected forward reference.

### Registry Values

Searched the whole document for every registry entry name this system owns or references
(`load_outcome`, `migrate`, `save_bytes`, `slots_available`, `save_schema_version`,
`save_bytes_max`, `save_max_slots`, `base_capacity`, `total_value`, `entry_fee`,
`milestone_met`, `effective_unlocks`), plus the `[_ ]` variants.

- **Matches:** `load_outcome` has six values in both GDD (line 157) and registry (line
  778), and the registry expression (line 779) mirrors the GDD formula including the
  `Blocked`-on-migration-failure term; `migrate` (784), `save_schema_version = 1`
  (1202), `save_bytes_max = 262144` (1210), `save_max_slots = 1` (1221) all agree; the
  `save_bytes` formula terms match.
- **Conflicts:** `save_backup_count` — rule 9 (line 35) defines `SAVE_BACKUP_COUNT = 1`
  but the registry still has no such entry (Recommended 1). `save_bytes` output range —
  GDD line 206 says `0–SAVE_BYTES_MAX`, registry line 817 says `[0, 262144]`, while
  `B_base = 1024` (Recommended 2). `base_capacity` exists in the registry (line 1263)
  but is missing from rule 2's illustrative list (line 28) (Recommended 3).
- The registry's `ReadOnly` note (line 780) mirrors the GDD's "live lock over any
  readable file", so blocking item 1 is an internal GDD inconsistency carried into the
  registry rather than a GDD-vs-registry divergence.

### Required Before Implementation

1. **[BLOCKING] Formulas / States / Acceptance Criteria — `ReadOnly` over "any version"
   with "不迁移" leaves no defined rendering and contradicts rule 12 for a future
   version (lines 36, 38, 65, 135–138, 323, 394; `entities.yaml:780`;
   `ui-flow.md:81,401`).**
   Rule 10 and the load table now define `ReadOnly` as "另一实例持锁、主档可读（任意
   版本）… 可浏览；不迁移", and the formula's lock branches (135–136) run *before* the
   version test (138). Two cases have no defined behavior:
   (a) `version(P) < V` — `ReadOnly` wins over the `Migrated` branch and migration is
   forbidden, yet the menu must show wallet, levels, unlocks and records, which an older
   schema does not have in current form; the GDD never says where those displayed values
   come from.
   (b) `version(P) > V` — rule 12 says "不加载、不迁移、不回退到备份、不覆盖", but the
   lock branch returns `ReadOnly`, which is a loaded-for-browsing state: an older build
   (B, V=1) starting while a newer build (A, V=2) holds the lock on a v2 file would
   "browse" a file rule 12 forbids loading, with no schema to read it.
   Fix: either (i) let `ReadOnly` render from an **in-memory** migration when
   `version(P) < V` (never written), and move the `version(P) > V` check ahead of the
   lock branches so a lock never downgrades a too-new file to `ReadOnly`; or (ii)
   restrict `ReadOnly` to `version(P) = V` and route other lock cases to a lock-caused
   `Blocked` that offers no destructive action (see blocking 2). Whichever is chosen,
   align rule 10, rule 21, the load table, the formula, UI Requirements 3, AC 23, the
   registry note and `ui-flow.md:81/401`, and add an AC asserting the `ReadOnly` outcome
   at `version(P) ≠ V`.

2. **[BLOCKING] Detailed Rules / UI Requirements — a lock-caused `Blocked` still offers
   "新建生涯（旧档保留）", quarantining a live instance's files (lines 47, 66, 249, 324;
   `ui-flow.md:81,401,504,629`; AC 23 line 394 covers only the readable case).**
   After the split, the only lock case that reaches `Blocked` is `L = live ∧
   ¬readable(P)` (formula line 136; rule 21 line 47; edge case line 249). That state is
   reachable during instance A's own atomic sequence — between "主档改名备份" and
   "临时文件改名主档" the primary is absent for a moment (rule 8, line 34) — so instance
   B can legitimately see a live lock over a missing/unreadable primary while A is
   healthy. The GDD then sends B to `Blocked`, and UI Requirements 4 (line 324) offers
   only 「退出」/「新建生涯（旧档保留）」; `ui-flow.md:81/401` makes the same modal
   mandatory. Choosing 「新建生涯」 renames A's backup and temp file to
   `corrupt-<UTC>.sav` and creates a second career, directly contradicting rule 21's
   "绝不覆盖对方的文件 … 宁可只读，不可覆盖".
   Fix: branch the `Blocked` modal by `reason`. Only 无可用副本 and 版本高于游戏 may
   offer 「新建生涯」; 持锁且文件不可读 must offer 「只读」/「退出」 only (or wait for the
   lock to clear), mirroring UI Requirements 3. Also: UI Requirements 4 and
   `ui-flow.md:504` enumerate **two** `Blocked` reasons, while the load table (line 66)
   lists **four** (无可用副本 / 版本过高 / 迁移失败 / 持锁且文件不可读) and
   `ui-flow.md:629` lists **three** (更新版本 / 无法读取 / 另一实例 — the last no longer a
   `Blocked` reason at all). Enumerate one canonical reason set and use it in all three
   places. Add an AC for `L = live ∧ ¬readable(P)` asserting no quarantine action is
   offered and no file is renamed.

### Recommended Revisions

1. **[RECOMMENDED] Registry — `save_backup_count` still missing.** Rule 9 (line 35)
   defines it; `entities.yaml` registers only `save_schema_version`,
   `save_bytes_max` and `save_max_slots` for this system. (Carried; registry not
   editable here.)
2. **[RECOMMENDED] Formulas — `save_bytes` lower bound is still 0.** GDD line 206 and
   registry line 817 both start at 0, but `B_base = 1024`. (Carried.)
3. **[RECOMMENDED] Detailed Rules — rule 2's illustrative list still omits
   `base_capacity` (line 28).** It is a current hard upstream field
   (`data-config.md`, registry line 1263). (Carried.)
4. **[RECOMMENDED] Cross-References — the 待回改 list is stale (lines 346–352).** Items
   1–4 are applied and item 5 still says the index should read `Designed` while it reads
   `In Review`. Delete the list or mark items done. (Carried.)
5. **[RECOMMENDED] Edge Cases — line 238 still says only "纪录类事务保留内存".** Rule 18
   now covers settlement wallet (`progression`), records and unlocks; restate as
   "结算入账 / 纪录 / 迁移". (Carried from round 2.)
6. **[RECOMMENDED] Acceptance Criteria — AC 1 (line 360) does not account for the new
   Fresh-time backfill.** It asserts "内存中 … 解锁集为空" then a baseline write, while
   edge case line 232 now requires `normalize()` before the baseline and the baseline to
   carry the backfilled `unlocks`. State "回填前解锁集为空；基线包含回填后的集合", or AC 1
   and AC 20 can be read as contradicting the edge case.
7. **[RECOMMENDED] Interfaces — the outbound `normalize()` call is not recorded.**
   Edge case line 232 and `unlocks-and-stations.md:53` agree that the save system calls
   `normalize(∅)` before the first baseline, but the interface table (lines 99–107) and
   the 解锁与档口 row (line 89) describe only inbound `commit`/`mark_dirty`. Since
   unlocks is declared downstream and itself waits for save load to complete, state the
   bootstrapping order (unlocks `Ready` after load, before the baseline transaction) and
   what happens if `normalize()` is unavailable — otherwise both sides can deadlock on
   "wait for the other to be ready".
8. **[RECOMMENDED] Cross-document — `ui-flow.md:629` still lists 「另一实例」 as a
   `Blocked` reason ("三种原因").** After the split it is a `ReadOnly` reason. Fix in the
   ui-flow pass; the save GDD's own UI Requirements 3–4 are correct on this point.

### Nice-to-Have

1. `n_sta` / `n_dish` / `n_rec` ranges omit retained unknown ids (lines 197, 199, 200);
   only `n_upg` says "含保留的未知 id". (Carried.)
2. Player Fantasy / Tuning Knobs remain advisory gaps at `standard`. (Carried.)
3. AC 13's "磁盘不发生写入" (line 378) is imprecise — an injected temp-file failure still
   touches disk; "主档与备份哈希不变" matches AC 11. (Carried.)
4. Cross-document wording — "存档规则 80" (`ui-flow.md:59/616`,
   `unlocks-and-stations.md:55`) means line 80 of a GDD with 25 numbered rules. (Carried.)
5. The version-0 fixture schema is undefined. AC 8 needs `valid(v0)` to accept a string
   `wallet` and AC 9 needs it to accept `"abc"` so migration, not the corrupt path,
   runs. One sentence pinning the v0 fixture schema would remove the last guess in the
   test build.

### Scope Signal

**Rough scope signal: L** — four formula blocks, one upstream + six hard downstream
systems, and three ADR-stage decisions (serialization format OQ 2, save path / loader
OQ 3, write-lock implementation; Steam Cloud OQ 4 later). `systems-index.md:200` rates
the design effort `S`; this review retains **L**.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (`systems-index.md:60` currently
reads `In Review`). Per the parallel constraint this review did not modify the index; the
orchestration session updates it. Both blocking items are local to the `ReadOnly` /
`Blocked` branch: the load formula, the two UI requirement rows and the reason set — no
envelope, state-model or migration redesign. One revision pass should clear the way for
a re-review.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/save-system.md 772ed596d68eb5cbae207b7bac9a170ec55c1524
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/gdd/systems-index.md c0b526ff05088d426598f5dc34e3bd9e3153cf59
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 9fe5183795ca3d47798510a1d00a040dabe3ca61
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
```

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 3 | Recommended: 10 | Nice-to-have: 6
Prior verdict resolved: Partially — prior blocking 1 is resolved in the formula and
covered by a new AC but not in the three sites that still say "不迁移"; prior blocking 2
is resolved in rule 21, UI 4 and AC 23 but survives in the state table; the newly
offered 「重试」 action has no defined mechanism and contradicts the `load()` contract.

Summary: Both fixes named in the round-3 brief are present, and the version-precedence
half of blocking 1 is now exactly right: the load formula tests `version(P) > V` ahead
of the lock branches (line 135), so a live lock can no longer downgrade a too-new file
to `ReadOnly`, and AC 24b (line 400) pins both halves of that branch. What did not travel
with the fix is the wording: rule 10 (line 36), the load table (line 65) and the formula
example (line 161) still define `ReadOnly` as "不迁移", which is the same defect blocking 1
named — for `version(P) < V` the definition sites forbid the only behaviour the AC allows.
Likewise blocking 2 was applied to rule 21, UI 4 and AC 23 but not to the state table
(lines 66, 78), which still sends every `Blocked` reason to 「新建生涯（旧档保留）」. One new
defect is a direct consequence of the new exit: 「重试」 is now the only non-exit action in
two `Blocked` reasons, yet `load()` is specified to cache its report and never re-read
(line 101), and the state table has no retry transition — the button cannot do anything.

### Prior Blocking Items — Status

1. **PARTIALLY RESOLVED** — round 3 asked for the version check ahead of the lock
   branches and a defined rendering source for a lower-version `ReadOnly`, plus an AC.
   Done: line 135 returns `Blocked` for `readable(P) ∧ version(P) > V` before the lock
   branches; AC 24b (line 400) asserts `ReadOnly` + in-memory read-only migration for
   `version(P) < V` and `Blocked`, no browsing, for `version(P) > V`. Not done: rule 10
   (line 36, "可浏览不可写、**不迁移**"), the load table (line 65, "可浏览；写入禁用；
   **不迁移**") and the formula example (line 161, "允许浏览但不写入、**不迁移**") were not
   updated. Rule 21 (line 47) and UI 3 (line 323) do not name the data source either.
   The AC is now the only site that permits the behaviour the other three forbid.

2. **PARTIALLY RESOLVED** — round 3 asked to branch the `Blocked` exits by reason and to
   use one canonical reason set. Done in this document: rule 21 (line 47) now says the
   lock-plus-unreadable case offers **only** 「退出」与「重试」 and never 「新建生涯」;
   UI 4 (lines 324–329) branches all four reasons and withholds 新建生涯 from 迁移失败 and
   持锁不可读; AC 23 (line 399) covers the atomic-window case and asserts no file is
   renamed. Not done: the load table's `Blocked` row (line 66) still reads "只有玩家确认
   新建生涯或退出" for all four reasons, and the transition row (line 78)
   `Blocked → 玩家确认新建生涯 → Fresh → Clean / 旧文件改名隔离` is still unconditional —
   the exact quarantine action rule 21 forbids against a live instance is still reachable
   through the table. Cross-document alignment is also incomplete: `ui-flow.md`
   (42–43, 81, 144, 401–402, 504, 586) still gives the modal exactly two exits, line 504
   says "两种原因之一" and line 630 says "三种原因（更新版本 / 无法读取 / 另一实例）" with
   另一实例 listed as a `Blocked` reason (it is `ReadOnly` when readable).

**This round's three named fixes — verified:** version-higher-than-current checked before
the lock branch ✓ (line 135; registry expression line 782 already matches); older-version
`ReadOnly` rendered from an in-memory, never-persisted migration ✓ (AC 24b, line 400 —
but see blocking 1); lock-plus-unreadable `Blocked` offers only 退出/重试 ✓ (rule 21
line 47, UI 4 line 328, AC 23 line 399 — but see blocking 2); AC 23/24b added ✓.

### Completeness: 6/6 required sections present (standard tier)

Required at `standard`: Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge
Cases, Dependencies, Acceptance Criteria — all present. Formulas is required because the
system defines four numeric rules.

Advisory gaps (do not block at `standard`):
- **Player Fantasy** — absent as a heading; `## Game Feel` covers the intent.
- **Tuning Knobs** — absent as a section; constants remain scattered.

### Dependency Graph

- ✓ `design/gdd/data-config.md` — exists (upstream, hard)
- ✓ `design/gdd/meta-upgrades.md` — exists (downstream, hard)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (downstream, hard)
- ✓ `design/gdd/records-and-milestones.md` — exists (downstream, hard)
- ✓ `design/gdd/ui-flow.md` — exists (downstream, hard UI)
- ✓ `design/gdd/scene-flow.md` — exists (declared non-dependency, timing only)
- ✗ `design/gdd/online-challenge.md` — NOT FOUND; `systems-index.md` #19 is
  `Not Started`. Declared soft / 未写, so an expected forward reference.

### Registry Values

Searched the whole document for every registry entry name this system owns or references
(`load_outcome`, `migrate`, `save_bytes`, `slots_available`, `save_schema_version`,
`save_bytes_max`, `save_max_slots`, `save_backup_count`, `base_capacity`, `total_value`,
`entry_fee`, `milestone_met`, `effective_unlocks`).

- **Matches:** `load_outcome` has six values in both GDD (line 157) and registry (line
  781), and the registry expression (line 782) mirrors the GDD formula *including* the
  version-before-lock precedence — the two agree on the decision order for the first
  time. `migrate`, `save_schema_version = 1`, `save_bytes_max = 262144`,
  `save_max_slots = 1` and the `save_bytes` terms all agree.
- **Conflicts:** `save_backup_count` — rule 9 (line 35) defines `SAVE_BACKUP_COUNT = 1`
  but the registry still has no such entry (Recommended 2). `save_bytes` output range —
  GDD line 206 says `0–SAVE_BYTES_MAX`, registry line 817 says `[0, 262144]`, while
  `B_base = 1024` (Recommended 3). `base_capacity` exists in the registry but is missing
  from rule 2's illustrative list (line 28) (Recommended 4).
- **Registry note lags the fix:** `load_outcome`'s note (line 783) still says
  "ReadOnly = a live lock over any readable file (browse, no write, **no migration**)",
  while AC 24b requires an in-memory migration for rendering. The expression is right; the
  prose is stale (Recommended 10).

### Required Before Implementation

1. **[BLOCKING] Detailed Rules / States / Formulas — `ReadOnly` is still defined as
   "不迁移" in the three definition sites, contradicting AC 24b (lines 36, 65, 161 vs
   line 400; `entities.yaml:783`; `ui-flow.md:81,401`).**
   Rule 10 reads "`ReadOnly`（… 可浏览不可写、不迁移）", the load table reads
   "可浏览；写入禁用；不迁移", and the formula example reads "允许浏览但不写入、不迁移".
   AC 24b is the opposite: "用内存中的只读迁移渲染，不写盘". For `version(P) < V` with a
   live lock, an implementer working from rule 10 or the table has no defined source for
   the wallet, levels, unlocks and records the menu must display — this is exactly the
   defect round 3's blocking 1(a) named, now confined to the definition sites.
   Fix: in rule 10, the load table and the formula example, replace "不迁移" with
   "迁移结果只在内存中用于渲染，绝不写盘"; add the same clause to rule 21 (line 47) and
   UI 3 (line 323). Keep AC 24b as the test.

2. **[BLOCKING] States and Transitions — the `Blocked` rows still offer 新建生涯 for all
   four reasons (lines 66, 78; contradicts rule 21 line 47 and UI 4 lines 324–329).**
   The load table's `Blocked` row says "进入 `Menu` 后由界面流程说明；只有玩家确认新建生涯
   或退出", and the transition table says `Blocked | 玩家确认新建生涯 | Fresh → Clean | 旧文件
   改名隔离`. Both are unqualified, so 迁移失败 and 持锁且文件不可读 still lead to an action
   that renames a live instance's files — the precise violation of rule 21's
   "绝不覆盖对方的文件 … 宁可只读，不可覆盖" that round 3 raised.
   Fix: branch the table's 后续行为 by reason in the same four terms UI 4 uses —
   「无可用副本」「版本高于当前」→ 退出 / 新建生涯（旧档保留）；「迁移失败」「持锁且文件不可读」
   → 退出 / 重试；and qualify the `Blocked → Fresh` transition as 仅前两种原因.
   Cross-document (not editable in this review): align `ui-flow.md:42–43, 81, 144,
   401–402, 504, 586, 630` to the same four-reason set.

3. **[BLOCKING] Interfaces / UI Requirements — 「重试」 has no mechanism, and `load()` is
   specified never to re-read (lines 101, 327–328, 47, 399; no transition at lines 70–79,
   no AC).**
   Rule 21, UI 4 (迁移失败 and 持锁不可读 rows) and AC 23 make 「重试」 the only non-exit
   action in two `Blocked` reasons. The interface table says `load()` "重复调用返回同一结果，
   不重复读写", so a retry that calls `load()` returns the same `Blocked` report; the state
   table defines no retry transition; no AC exercises it. In the 迁移失败 case the retry is
   a re-run of a pure function over an unchanged file and is guaranteed to fail again
   unless it re-reads from disk.
   Fix: define the retry contract (a `retry_load()` — or `load({force: true})` — that
   discards the cached report, re-probes the lock and re-reads P/B, callable only from
   `Blocked`), add the `Blocked → Dirty / Clean` transition, state per reason what retry
   does (lock: re-probe and wait; migration failure: re-read then re-run), and add an AC
   asserting one retry after the lock clears yields the non-`Blocked` outcome.

### Recommended Revisions

1. **[RECOMMENDED] Formulas — the `ReadOnly` branch does not define what happens when the
   in-memory migration fails (line 136).** It tests only `readable(P)`; `valid(P)` and
   `migrate(...)` are not in the branch. A parseable file that is illegal under its
   declared schema — or whose migration returns `MigrationError` — while another instance
   holds the lock lands in `ReadOnly` with no rendering source. Define the fallback, most
   consistently `Blocked` with 退出 / 重试 only (no quarantine), and cover it with the
   retry AC from blocking 3.
2. **[RECOMMENDED] Registry — `save_backup_count` is still missing.** Rule 9 (line 35)
   defines `SAVE_BACKUP_COUNT = 1`; `entities.yaml` registers only `save_schema_version`,
   `save_bytes_max` and `save_max_slots` for this system. (Carried; registry not editable
   here.)
3. **[RECOMMENDED] Formulas — `save_bytes` lower bound is still 0.** GDD line 206 and
   registry line 817 both start at 0, but `B_base = 1024` (line 193). (Carried.)
4. **[RECOMMENDED] Detailed Rules — rule 2's illustrative list still omits `base_capacity`
   (line 28).** It is a current hard upstream field (`data-config.md`, registry line
   1263). (Carried.)
5. **[RECOMMENDED] Cross-References — the 待回改 list is stale (lines 351–357).** Items
   1–4 are applied and item 5 still says the index should read `Designed` while
   `systems-index.md:60` reads `In Review`. Delete the list or mark the items done.
   (Carried.)
6. **[RECOMMENDED] Edge Cases — line 238 still says only "纪录类事务保留内存".** Rule 18
   covers settlement wallet (`progression`), records, unlocks and migration; restate as
   "结算入账 / 纪录 / 迁移". (Carried from round 2.)
7. **[RECOMMENDED] Acceptance Criteria — AC 1 (line 365) still does not account for the
   Fresh-time backfill.** It asserts "解锁集为空" then a baseline write, while edge case
   line 232 requires `normalize()` before the baseline and the baseline to carry the
   backfilled `unlocks`. State "回填前解锁集为空；基线包含回填后的集合". (Carried.)
8. **[RECOMMENDED] Interfaces — the outbound `normalize()` call is still not recorded.**
   Edge case line 232 and `unlocks-and-stations.md:53` agree that the save system calls
   `normalize(∅)` before the first baseline, but the interface table (lines 99–107) and
   the 解锁与档口 row (line 89) describe only inbound `commit`/`mark_dirty`. State the
   bootstrapping order (unlocks `Ready` after load, before the baseline transaction) and
   what happens if `normalize()` is unavailable — otherwise both sides can deadlock on
   "wait for the other to be ready". (Carried.)
9. **[RECOMMENDED] Cross-document — `ui-flow.md` still carries the old `Blocked` reason
   set.** Lines 42–43/504 give the modal two exits and "两种原因", line 630 lists three
   reasons with 另一实例 among them. The canonical set is the load table's four
   (line 66). Fix in the ui-flow pass. (Carried, now broader than round 3's item 8.)
10. **[RECOMMENDED] Cross-document — the registry `load_outcome` note lags the memory-
    migration fix (`entities.yaml:783`).** The expression is correct; the note still says
    "no migration" for `ReadOnly`. Update once blocking 1 is written into the GDD.

### Nice-to-Have

1. `n_sta` / `n_dish` / `n_rec` ranges omit retained unknown ids (lines 197, 199, 200);
   only `n_upg` says "含保留的未知 id". (Carried.)
2. Player Fantasy / Tuning Knobs remain advisory gaps at `standard`. (Carried.)
3. AC 13's "磁盘不发生写入" (line 383) is imprecise — an injected temp-file failure still
   touches disk; "主档与备份哈希不变" matches AC 11. (Carried.)
4. Cross-document wording — "存档规则 80" (`ui-flow.md:59/616`,
   `unlocks-and-stations.md:55`) means line 80 of a GDD with 25 numbered rules. (Carried.)
5. The version-0 fixture schema is undefined. AC 8 needs `valid(v0)` to accept a string
   `wallet` and AC 9 needs it to accept `"abc"`. One sentence pinning the v0 fixture
   schema would remove the last guess in the test build. (Carried.)
6. AC 24b (line 400) is numbered between 23 and 24. Renumber to 25 and shift 25 → 26,
   26 → 27 if the section is ever touched again; the test-fixture note (line 411) does not
   reference it, so nothing else moves.

### Scope Signal

**Rough scope signal: L** — four formula blocks, one upstream + six hard downstream
systems, and three ADR-stage decisions (serialization format OQ 2, save path / loader
OQ 3, write-lock implementation; Steam Cloud OQ 4 later). `systems-index.md:200` rates
the design effort `S`; this review retains **L**.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (`systems-index.md:60` currently
reads `In Review`). Per the parallel constraint this review did not modify the index; the
orchestration session updates it. All three blocking items are wording and contract
defects in the load-result branch — the state table's two `Blocked` rows, the three
"不迁移" sites, and the retry contract — no envelope, state-model or migration redesign.
One revision pass should clear the way for a re-review.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/save-system.md f22c2228d60be43669125831bccea834e94d6026
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
Reviewed-Content-Hash: design/gdd/ui-flow.md 9c709252b4c631e8fb50d8a2f35a8fb394f230d7
Reviewed-Content-Hash: design/gdd/systems-index.md 56e9869c70371816e88ed49f24735f67ab3d199d
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 84e5c38fc40fb2a75d8ee608c911bdebf369c5a1
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
```

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 2 | Recommended: 10 | Nice-to-have: 4
Prior verdict resolved: Partially — prior blocking 2 and 3 are fully resolved; prior
blocking 1 is resolved for the "不迁移" wording but its version-precedence half is still
contradicted by the definition sites, and the version-too-high exit can now isolate a live
instance's file.

Summary: The two fixes named in the round-4 brief are present and correctly applied:
`ReadOnly` now renders an older save through an in-memory, never-persisted migration at
rule 10 (line 36), the load table (line 65) and the formula example (line 163), and the
`Blocked` exits are branched by reason in the load table (line 66), the transition table
(lines 78–79), rule 21 (line 47), UI 4 (lines 326–330) and AC 23 (line 401). The retry
contract is now real: `retry_load()` is in the interface table (line 103), has a
transition (line 79) and AC 30 (line 412). What the fix did not carry is the
version axis of the same branch: the load table still defines `ReadOnly` as "任意版本"
(line 65), and rule 10 / Edge Case 251 / UI 3 (line 325) still say any readable primary
under a live lock is `ReadOnly`, while the formula (line 137) and AC 24b (line 402) give
`version(P) > V` precedence and return `Blocked`. Worse, the `Blocked` row for "版本高于
当前" still offers 「新建生涯」(line 66, line 327) with no lock condition — under a live
lock that quarantine renames the other instance's file, which rule 21 and rule 12 forbid.
All other findings are carried recommendations or cross-document stale copies.

### Prior Blocking Items — Status

1. **PARTIALLY RESOLVED** — the "不迁移" defect is gone: rule 10 (line 36), the load
   table (line 65) and the formula example (line 163) now all say an older-version
   `ReadOnly` is rendered from an in-memory migration that is never written, matching AC
   24b (line 402). But the version-precedence half is still not uniform across the
   definition sites:
   - Load table (line 65) still says "另一实例持锁、主档可读（**任意版本**)" → `ReadOnly`.
   - Rule 10 (line 36), Edge Case 251 and UI 3 (line 325) say "主档可读 → `ReadOnly`"
     with no version qualifier.
   - Formula (line 137) and AC 24b (line 402) say `version(P) > V` → `Blocked` even
     under a live lock, and "不提供只读浏览".
   So for `L = live ∧ readable(P) ∧ version(P) > V` the document still gives two
   answers, which is the same definition-site-vs-AC split that blocking 1 named. This
   round also exposes its unsafe exit: the `Blocked` row for "版本高于当前" (line 66)
   and UI 4 (line 327) offer 「新建生涯（旧档保留）」 unconditionally; choosing it runs
   rule 12's "旧档改名隔离" against a file another live instance is writing — the exact
   action rule 21 (line 47, "绝不覆盖对方的文件") and AC 23 (line 401) forbid for the
   other `Blocked` reasons.
   **Fix:** qualify `ReadOnly` with `version(P) ≤ V` at every site (line 36, 65, 251,
   325), and make every `Blocked` exit set lock-aware — when `L = live`, offer only
   「退出」/「重试」 regardless of reason, never 「新建生涯」.

2. **RESOLVED** — the load table's `Blocked` row (line 66) now branches by reason
   ("更新版本 / 无法读取 → 退出 / 新建生涯；迁移失败 / 持锁不可读 → 退出 / 重试"), the
   transition row `Blocked → 新建生涯` is conditioned on reason (line 78), rule 21
   (line 47) states the lock-plus-unreadable exit set, UI 4 (lines 326–330) branches all
   four reasons, and AC 23 (line 401) now covers both the readable (`ReadOnly`) and
   unreadable (`Blocked`, 退出/重试 only, no rename) halves of the atomic window. The
   version-too-high overlap noted in blocking 1 is a separate, still-open cell.

3. **RESOLVED** — `retry_load()` is now in the interface table (line 103) with defined
   semantics (re-run the full load, replace the cached `LoadReport`, isolate nothing), has
   a state transition (line 79) and AC 30 (line 412). Rule 21 (line 47) names 「重试」 as
   the second of exactly two exits for the lock-plus-unreadable case. `load()` no longer
   claims to be the retry path (line 102).

### Completeness: 6/6 required sections present (standard tier)

Required at `standard`: Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge
Cases, Dependencies, Acceptance Criteria — all present. Formulas is required because the
system defines four numeric rules.

Advisory gaps (do not block at `standard`):
- **Player Fantasy** — absent as a heading; `## Game Feel` covers the intent.
- **Tuning Knobs** — absent as a section; constants (lines 33, 35, 207) remain scattered.

### Dependency Graph

- ✓ `design/gdd/data-config.md` — exists (upstream, hard)
- ✓ `design/gdd/meta-upgrades.md` — exists (downstream, hard)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (downstream, hard)
- ✓ `design/gdd/records-and-milestones.md` — exists (downstream, hard)
- ✓ `design/gdd/ui-flow.md` — exists (downstream, hard/UI)
- ✗ `design/gdd/online-challenges.md` — NOT FOUND (downstream, soft; declared "GDD 未写",
  line 275 — acceptable for a future system, not a broken reference)

`scene-flow` and `settlement-and-entry-fee` are explicitly declared non-dependencies
(lines 280–281); both sibling GDDs agree (`scene-flow.md:79,173`,
`settlement-and-entry-fee.md:104,341`).

### Registry Values

Search terms: `load_outcome`, `migrate`, `save_bytes`, `slots_available`,
`save_schema_version`, `save_bytes_max`, `save_max_slots`, `milestone_met`,
`effective_unlocks`, `base_capacity`, `entry_fee`, `max_level`, `wallet` — 40 lines
matched. No numeric conflict: `SAVE_BYTES_MAX = 262144` matches registry line 1214,
`MAX_SLOTS = 1` matches line 1226, `SAVE_SCHEMA_VERSION = 1` matches line 1203.

One semantic conflict, in the registry rather than this document:
- `entities.yaml:783` — the `load_outcome` note still reads "ReadOnly = a live lock over
  any readable file (browse, no write, **no migration**)". The GDD now says the opposite
  (in-memory read-only migration, rule 10 / line 65 / line 163 / AC 24b) and gives
  `version(P) > V` precedence over the lock (formula line 137). The registry's own
  `expression` (line 782) is already correct; only the note is stale. Fix in the registry
  (out of this session's write scope).
- Non-blocking nit: both GDD (line 208) and registry (line 820) give `save_bytes` a lower
  bound of 0, but `B_base = 1024` (line 195) makes the true minimum 1024 B.

### Required Before Implementation

1. **[BLOCKING — GDD] `版本过高` + 存活锁的出口与定义站点** (lines 36, 65, 66, 251, 325;
   AC 24b line 402; formula line 137). The load table says `ReadOnly` for "任意版本";
   rule 10 / Edge Case / UI 3 say any readable primary under a live lock is `ReadOnly`;
   the formula and AC 24b say `version(P) > V` is `Blocked` first. Then the `Blocked`
   exit mapping for "版本高于当前" offers 「新建生涯」, which isolates the live instance's
   file (rule 12 line 38, rule 21 line 47). Unify the sites on `version(P) ≤ V` and make
   the exit set lock-aware (`L = live` → 退出/重试 only).
2. **[BLOCKING — registry] `load_outcome` note contradicts the corrected rule**
   (`entities.yaml:783`, "no migration"; "any readable file"). An implementer reading the
   registry builds the exact `ReadOnly` behaviour blocking 1 of round 4 removed. The fix
   belongs in `entities.yaml`, not this GDD.

### Recommended Revisions

1. `ReadOnly` 分支仍未定义「可读但 `valid(P) = false`」与「内存迁移失败」的结果
   (formula line 138 tests `readable(P)` only, no `valid(P)`; lines 36/65/163 promise
   browsing). There is no render source and no fallback to the `Blocked` 迁移失败 exit.
   Carried from round 4. Give the branch a defined failure outcome.
2. `ui-flow.md:402` (Edge Case: `Blocked` "只提供「退出」「新建生涯（旧档保留）」") and
   `ui-flow.md:419` ("只接受其两个出口") are stale two-exit copies. They contradict that
   GDD's own rule 20 (`ui-flow.md:81`), table (`:504`) and AC 26 (`:586`), which already
   branch four reasons. Fix in `ui-flow.md`.
3. `registry` has no `save_backup_count` entry although rule 9 (line 35) defines
   `SAVE_BACKUP_COUNT = 1`. Carried from rounds 3–4.
4. `save_bytes` lower bound is 0 (line 208; registry line 820) but `B_base = 1024`.
   Carried from rounds 3–4.
5. Rule 2's example list (line 28) omits `base_capacity`, which is the same class of
   derived runtime value as `dish_pool`. Carried from rounds 3–4.
6. Edge Case 240 says only "纪录类事务保留内存", while rule 18 (line 44) and the state
   table (line 76) also cover 结算入账 and 解锁. Carried from rounds 3–4.
7. AC 1 (line 367) still asserts an empty unlock set at `Fresh` and says nothing about
   the `normalize()` backfill that Edge Case 234 requires before the baseline write. A
   literal AC 1 test would expect a baseline different from what the edge case mandates.
   Carried from rounds 3–4.
8. The interface table (lines 100–109) still omits the outbound `normalize()` call to
   unlocks and the startup order (data `Ready` → `load()` → `normalize()` → baseline).
   It is stated only in Edge Case 234 and Dependencies line 273. Carried.
9. Cross-References "待回改" items 1–4 (lines 355–358) contradict Dependencies lines
   285–288, which record those same documents as already 回改; item 5 (line 359) still
   asks for index `#4 = Designed` while the index reads `In Review`. Carried.
10. `ReadOnly` session behaviour for a run is undefined. UI 3 (line 325) hides purchase
    and unlock entry points but does not forbid 「开始营业」; in `ReadOnly` every write
    is disabled (line 161, interface line 102), so a played run's settlement can never
    commit and its whole result is discarded at exit. That contradicts Game Feel's
    "最多丢掉尚未成功写入的那一次事务" (line 317). Either forbid starting a run or show
    an explicit "本局结果不会保存" notice before it starts.

### Nice-to-Have

1. AC 24b (line 402) is numbered before AC 24 (line 403). Renumber if the section is
   touched again. (Carried.)
2. The retry transition (line 79) uses `Unloaded` → `Loading`, two states that the
   load-outcome model never defines.
3. `retry_load()` on a 迁移失败 reason re-runs a pure function and must fail again
   (AC 30 line 412 defines the behaviour honestly). Consider suppressing 「重试」 for that
   reason in the UI so the affordance is not misleading.
4. `menu-waves.md:208` says a missing id makes the load fail and return to the menu; save
   rule 14 (line 40) preserves unknown ids instead. Not reachable in MVP (no mid-run
   save), but the two must agree if Open Question 1 is ever enabled.

### Scope Signal

**Rough scope signal: L** — four formula blocks, one upstream + four hard downstream
systems, and four ADR/architecture-stage decisions (serialization format OQ 2, save path
and loader OQ 3, write-lock implementation OQ 9; Steam Cloud OQ 4 later).
`systems-index.md:200` rates the design effort `S`; this review retains **L**.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (`systems-index.md:60` currently
reads `In Review`). Per the parallel constraint this review did not modify the index, the
GDD, or the registry; the orchestration session applies the status. The blocking set is
narrow: one version-axis wording/exit defect inside the load-result branch, plus one stale
registry note. The envelope, state model, atomic-write and migration design are unchanged
and consistently specified. One revision pass should clear the way for another re-review.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/save-system.md a362801aa491c54afaf3aa2448c396fa420c976b
Reviewed-Content-Hash: design/registry/entities.yaml 9915429b2bba990a0ae53a735bc2b21b24789d9a
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/ui-flow.md 46d6938f567d04423303bb4bd789234b6b1ddf6c
Reviewed-Content-Hash: design/gdd/systems-index.md 75098fd56028b3ec1b1abb5cb53da81e0eccaf6d
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/menu-waves.md 7edb32ad4c200e2b3a8e858adef8828afd74ce3a
```

---

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 1 | Recommended: 10 | Nice-to-have: 5
Prior verdict resolved: Partially — prior blocking 2 (registry note) is fully resolved;
prior blocking 3 was already resolved; prior blocking 1's version-axis half is now
uniform, but its lock-aware-exit half is still absent from the load-result table, so one
cell still authorises quarantining a live instance's file.

Summary: The version axis is now consistent across every definition site. Rule 10 (line
36), the load table (line 65), the formula (lines 137–139), the formula example (line
163), Edge Case 251 and AC 24b (line 403) all agree that `ReadOnly` requires
`version(P) ≤ V`, that `version(P) > V` is `Blocked` before the lock branches, and that
the older-version browse path is an in-memory migration never written to disk. Rule 21
(line 47) and UI 4 (lines 326–331) now state that any `L = live` `Blocked` offers only
exit/retry. The registry `load_outcome` note has been rewritten to match. What remains
is the one site the round-5 brief named explicitly: the load-result table's `Blocked`
row (line 66) still maps "更新版本 / 无法读取" to "退出 / 新建生涯" with no lock
condition, and the `ui-flow.md` copies of that mapping (lines 81, 402, 504, 586) carry
the same unconditional exit set. For `L = live` that table answer authorises
「新建生涯」, whose rule-12 rename would isolate a file the other instance is writing —
the exact action rule 21, UI 4 (line 331) and AC 23 (line 401) forbid.

### Prior Blocking Items — Status

1. **PARTIALLY RESOLVED** — the version-axis defect is gone everywhere it was named in
   round 5: load table line 65 now reads "版本 ≤ 当前", rule 10 (line 36) and Edge Case
   251 carry the same qualifier, UI 3 (line 325) is reached only for a `ReadOnly`
   outcome whose definition already carries `≤ V`, and formula lines 137–139 plus AC 24b
   (line 403) give `version(P) > V` precedence over the lock. The lock-aware exit half is
   applied in rule 21 (line 47: "任何 `L = live` 的 `Blocked` 出口只有「退出」与「重试」")
   and UI 4 (line 331: same sentence, "即使原因是「更新版本」或「无法读取」"), but **not**
   in the load-result table:
   - Line 66: "按原因给出口：更新版本 / 无法读取 → 退出 / 新建生涯；迁移失败 / 持锁不可读
     → 退出 / 重试" — reason-keyed only. For `L = live ∧ readable(P) ∧ version(P) > V`
     and `L = live ∧ ¬readable(P)` it still offers 「新建生涯」.
   - `ui-flow.md:81` (rule 20), `:402` (Edge Case), `:504` (table) and `:586` (AC 26) all
     repeat the reason-keyed mapping with no live-lock override; `:419` still says the
     `Blocked` modal accepts "其两个出口".
   - Edge Case 239 authorises the rename "只有当玩家在界面明确选择「新建生涯」时" without a
     lock condition, so it implies the offer exists in every unreadable case.
   **Fix:** make the load table's `Blocked` row lock-aware — e.g. "无存活锁：更新版本 /
   无法读取 → 退出 / 新建生涯；迁移失败 → 退出 / 重试；任何 `L = live` → 退出 / 重试，
   不提供新建生涯" — and add the same qualifier to Edge Case 239. The `ui-flow.md` copies
   are a follow-on fix in that file.

2. **RESOLVED** — `entities.yaml:783` now reads "ReadOnly = a live lock over a readable
   primary with version(P) <= V (browse, never write; an older version renders via
   in-memory read-only migration and is not migrated on disk). A higher-than-current
   version is Blocked before the lock branches; a live-lock Blocked offers only
   exit/retry, never a new career." This matches rule 10 (line 36), the formula (lines
   137–139), UI 4 (line 331) and AC 24b (line 403); the `expression` (line 782) was
   already correct. The note still needs its `revised` field set, which is housekeeping,
   not a finding.

3. **RESOLVED** — unchanged from round 5: `retry_load()` at interface line 103,
   transition line 79, AC 30 (line 413), rule 21 (line 47).

### Completeness: 6/6 required sections present (standard tier)

Required at `standard`: Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge
Cases, Dependencies, Acceptance Criteria — all present. Formulas is required because the
system defines four numeric rules.

Advisory gaps (do not block at `standard`):
- **Player Fantasy** — absent as a heading; `## Game Feel` covers the intent.
- **Tuning Knobs** — absent as a section; constants (lines 33, 35, 207) remain scattered.

### Dependency Graph

- ✓ `design/gdd/data-config.md` — exists (upstream, hard)
- ✓ `design/gdd/meta-upgrades.md` — exists (downstream, hard)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (downstream, hard)
- ✓ `design/gdd/records-and-milestones.md` — exists (downstream, hard)
- ✓ `design/gdd/ui-flow.md` — exists (downstream, hard/UI)
- ✗ `design/gdd/online-challenges.md` — NOT FOUND (downstream, soft; declared "GDD 未写",
  line 275 — acceptable for a future system)

`scene-flow` and `settlement-and-entry-fee` are explicitly declared non-dependencies
(lines 280–281); both sibling GDDs agree.

### Registry Values

Search terms: `load_outcome`, `migrate`, `save_bytes`, `slots_available`,
`save_schema_version`, `save_bytes_max`, `save_max_slots`, `save_backup_count`,
`milestone_met`, `effective_unlocks`, `base_capacity`, `entry_fee`, `max_level`,
`wallet`. No numeric conflict: `SAVE_BYTES_MAX = 262144` matches registry line 1214,
`MAX_SLOTS = 1` matches line 1226, `SAVE_SCHEMA_VERSION = 1` matches line 1203. The
`load_outcome` note conflict is resolved (prior blocking 2). Two carried registry gaps
remain: no `save_backup_count` entry for `SAVE_BACKUP_COUNT = 1`, and `save_bytes`
`output_range: [0, 262144]` still claims a 0 lower bound although `B_base = 1024`.

### Required Before Implementation

1. **[BLOCKING — GDD] `Blocked` exit row is not lock-aware** (line 66; Edge Case 239
   line 239). Rule 21 (line 47) and UI 4 (line 331) say a live-lock `Blocked` offers only
   exit/retry for every reason; the load-result table and Edge Case 239 still offer
   「新建生涯」 for "更新版本 / 无法读取" with no lock condition. Under `L = live` that
   choice runs rule 12's rename against a file another live instance may be mid-write on,
   which rule 21 ("绝不覆盖对方的文件") and AC 23 (line 401) forbid. One table row plus one
   edge-case bullet; no schema, formula or state-model change.

### Recommended Revisions

1. `ReadOnly` branch still has no outcome for `readable(P) ∧ ¬valid(P)`, and no outcome
   for an in-memory migration that fails (formula line 138 tests `readable(P)` only,
   while lines 36/65/163 promise a rendered browse). Give the branch a defined failure
   result (fall through to `Blocked` 迁移失败) or state that `ReadOnly` implies
   `valid(P)` and a successful in-memory migration. Carried.
2. `ui-flow.md:81` (rule 20), `:402` (Edge), `:504` (table) and `:586` (AC 26) still
   carry the reason-keyed `Blocked` mapping without the live-lock override; `:419` says
   the modal accepts "其两个出口". Update after blocking 1 so the two GDDs agree for
   `L = live`. Fix in `ui-flow.md`. Carried, narrowed.
3. `registry` has no `save_backup_count` entry although rule 9 (line 35) defines
   `SAVE_BACKUP_COUNT = 1`. Carried.
4. `save_bytes` lower bound is 0 (GDD line 208; registry `output_range`) but
   `B_base = 1024`. Carried.
5. Rule 2's example list (line 28) omits `base_capacity`, the same class of
   config-derived runtime value as `dish_pool`. Carried.
6. Edge Case 240 says only "纪录类事务保留内存", while rule 18 (line 44) and the state
   table (line 76) also cover 结算入账 and 解锁. Carried.
7. AC 1 (line 368) still asserts an empty unlock set at `Fresh` with no mention of the
   `normalize()` backfill Edge Case 234 requires before the baseline write. Carried.
8. The interface table (lines 100–109) still omits the outbound `normalize()` call and
   the startup order (data `Ready` → `load()` → `normalize()` → baseline), stated only in
   Edge Case 234 and Dependencies line 273. Carried.
9. Cross-References "待回改" items 1–4 (lines 356–359) contradict Dependencies lines
   285–288, which record those same documents as already 回改; item 5 (line 360) still
   asks for index `#4 = Designed` while `systems-index.md:60` reads `In Review`.
   Carried.
10. `ReadOnly` session behaviour for a run is still undefined: UI 3 (line 325) hides
    purchase/unlock entry points but does not forbid 「开始营业」, and every write is
    disabled in `ReadOnly`, so a played run's settlement can never commit. Either forbid
    starting a run or show an explicit "本局结果不会保存" notice. Carried.

### Nice-to-Have

1. AC 24b (line 403) is numbered before AC 24 (line 404). Renumber if the section is
   touched again. (Carried.)
2. The retry transition (line 79) uses `Unloaded` → `Loading`, two states the
   load-outcome model never defines. (Carried.)
3. `retry_load()` on a 迁移失败 reason re-runs a pure function and must fail again (AC 30
   defines this honestly); consider suppressing 「重试」 for that reason in the UI.
   (Carried.)
4. `menu-waves.md:215` says a missing id makes the load fail and return to the menu;
   save rule 14 (line 40) preserves unknown ids instead. Unreachable in MVP (no mid-run
   save), but the two must agree if Open Question 1 is enabled. (Carried.)
5. UI 3 (line 325) describes `ReadOnly` as "另一实例持锁但文件可读时" without repeating
   `version ≤ 当前`; the outcome definition already implies it, so this is wording only.

### Scope Signal

**Rough scope signal: L** — four formula blocks, one upstream + four hard downstream
systems, and four architecture-stage decisions (serialization format OQ 2, save path and
loader OQ 3, write-lock implementation OQ 9; Steam Cloud OQ 4 later).
`systems-index.md:200` rates the design effort `S`; this review retains **L**.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (`systems-index.md:60` currently
reads `In Review`). Per the parallel constraint this review did not modify the index, the
GDD, the registry or `ui-flow.md`; the orchestration session applies the status. The
blocking set is now a single cell: the load-result table's `Blocked` row (and its Edge
Case 239 counterpart) needs the live-lock qualifier already present in rule 21 and UI 4.
The envelope, state model, atomic-write and migration design remain consistently
specified; one small edit should clear the way for re-review.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/save-system.md 0d0edcdeed2612058390f2a0da75dab9cc15e1da
Reviewed-Content-Hash: design/registry/entities.yaml 5e3b667c300bdf6b9daad9b294aecf224b502b8e
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/ui-flow.md 35a3291d40d49cf1b6f35190ed231112462f0bc6
Reviewed-Content-Hash: design/gdd/systems-index.md b59ef2478102399659f8ae8b533c98a63a1c90b0
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/settlement-and-entry-fee.md b033f0acd34e58fa436fd583230335ee143196ae
Reviewed-Content-Hash: design/gdd/cloying-and-taste.md 1c8fd48bbdff0e8d24e5e436a8d801545df4da0b
Reviewed-Content-Hash: design/gdd/menu-waves.md 90754805044963eef29e76a1a031a706e0c30a2f
Reviewed-Content-Hash: design/gdd/meta-upgrades.md b9b9235e9a4ad7d8739ad778e98dd147b1c72f94
Reviewed-Content-Hash: design/gdd/unlocks-and-stations.md 5aafb0674e6a0bc5bd6a370c275e95a2f01ebfa9
Reviewed-Content-Hash: design/gdd/records-and-milestones.md 84e5c38fc40fb2a75d8ee608c911bdebf369c5a1
```

## Review — 2026-11-04 — Verdict: NEEDS REVISION

Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean mode — Phase 3b skipped)
Blocking items: 1 | Recommended: 11 | Nice-to-have: 6
Prior verdict resolved: Partially — the load-result table's `Blocked` row (line 66) now
carries the live-lock override, and `LoadReport` exposes `lock_live` (rule 10, line 36).
The same lock-blind offer survives at four other normative sites inside this GDD: rule 12
(line 38), the transition rows for 「新建生涯」 (line 78) and 「删除生涯」 (line 80), and
Edge Case 239 (line 239).

Summary: The four fixes claimed for main are present and internally correct. Rule 10
(line 36) adds `lock_live` to `LoadReport`; the `Blocked` row (line 66) is lock-first
(`lock_live` → exit/retry only, reason-keyed otherwise); `retry_load()` appears in the
interface table (line 103), the transition table (line 79) and AC 30 (line 413); and
`ReadOnly` is bounded by `version(P) ≤ V` in the rule (line 36), the load table (line 65),
the formula (line 138), the formula example (line 163), Edge Case 251 and AC 24b
(line 403). What the fix did **not** reach is every other site that still offers a file
rename — rule 12, the two transition rows, and Edge 239 — so under `L = live` the document
still authorises quarantining a file another instance may be mid-write on, which rule 21
(line 47), UI 4 (line 331), AC 23 (line 402) and AC 24b (line 403) forbid. One further
contract gap is new: the load table now offers 「重试」 for *every* live-lock `Blocked`,
while `retry_load()`'s stated calling condition (line 103) and the transition row (line 79)
still restrict it to 迁移失败 / 持锁不可读.

Process note: the round-6 fix commit `e9f6200` ("docs: make save-system Blocked exits
lock-aware"), which edited rule 12, both transition rows, Edge Cases 238–239, UI 3 and the
header date, is **not an ancestor of main** — main merged only the round-6 review-log
commit `87d4b6c` (merge `5d29a14`, second parent `87d4b6c`). `e9f6200` exists only on
`claude/competent-kirch-d8ddb2`. That is why the sibling sites are still unfixed while the
table row is not.

### Prior Blocking Items — Status

1. **PARTIALLY RESOLVED** — the load table's `Blocked` row (line 66) is now lock-first and
   matches rule 21 (line 47) and UI 4 (line 331). The remainder of the same defect class
   is untouched:
   - Rule 12 (line 38): "只有玩家明确选择「新建生涯」时，旧档才被改名隔离" — no lock
     condition, so for `L = live ∧ version(P) > V` it authorises the rename.
   - Transition table line 78: "`Blocked`（更新版本 / 无法读取）| 玩家确认新建生涯" — no
     lock condition. The load table says this offer does not exist under `lock_live`.
   - Transition table line 80: "任意 | 玩家确认删除生涯" — `任意` includes `ReadOnly` and
     lock-live `Blocked`; the delete path renames the live instance's primary (rule 23).
   - Edge Case 239 (line 239): rename "只有当玩家在界面明确选择「新建生涯」时" — again
     unconditional. Edge Case 251 (line 251) does carry the lock caveat, which makes the
     two adjacent bullets contradict each other.
   **Fix:** add 「（且无存活写锁 / `L ≠ live`）」 to rule 12 and the three sites above,
   mirroring rule 21 / UI 4 / AC 23. No schema or formula change.
2. **RESOLVED** — unchanged from round 6: registry `load_outcome` notes (entities.yaml:782)
   describe `ReadOnly` with `version(P) <= V`, version precedence over the lock, and
   live-lock `Blocked` as exit/retry only. `revised` is still empty (housekeeping).
3. **RESOLVED** — `retry_load()` present at interface line 103, transition line 79, AC 30
   (line 413), rule 21 (line 47).

### Completeness: 6/6 required sections present (standard tier)

Required at `standard`: Overview, Detailed Rules (`## Detailed Design`), Formulas, Edge
Cases, Dependencies, Acceptance Criteria — all present (`gdd-structure-check.sh` reports
the same). Formulas is required because the system defines four numeric rules.

Advisory gaps (do not block at `standard`):
- **Player Fantasy** — absent as a heading; `## Game Feel` covers the intent.
- **Tuning Knobs** — absent as a section; constants (lines 33, 35, 207) remain scattered.

### Dependency Graph

- ✓ `design/gdd/data-config.md` — exists (upstream, hard)
- ✓ `design/gdd/meta-upgrades.md` — exists (downstream, hard)
- ✓ `design/gdd/unlocks-and-stations.md` — exists (downstream, hard)
- ✓ `design/gdd/records-and-milestones.md` — exists (downstream, hard)
- ✓ `design/gdd/ui-flow.md` — exists (downstream, hard/UI)
- ✗ `design/gdd/online-challenges.md` — NOT FOUND (downstream, soft; declared "GDD 未写",
  line 275 — acceptable for a future system)

`scene-flow` and `settlement-and-entry-fee` are explicitly declared non-dependencies
(lines 280–281); both sibling GDDs agree.

### Registry Values

Search terms: `load_outcome`, `migrate`, `save_bytes`, `slots_available`,
`save_schema_version`, `save_bytes_max`, `save_max_slots`, `save_backup_count`,
`lock_live`. No numeric conflict: `SAVE_SCHEMA_VERSION = 1` matches entities.yaml:1202,
`SAVE_BYTES_MAX = 262144` matches line 1213, `MAX_SLOTS = 1` matches line 1224. The
`load_outcome` note and expression agree with the GDD (prior blocking 2 resolved). Two
carried registry gaps remain: no `save_backup_count` entry for `SAVE_BACKUP_COUNT = 1`,
and `save_bytes` `output_range: [0, 262144]` (line 819) still claims a 0 lower bound
although `B_base = 1024`. `lock_live` has no registry entry; it is a `LoadReport` field,
not a formula variable, so this is not a finding.

### Required Before Implementation

1. **[BLOCKING — GDD] Lock-blind file-rename offers outside the load table** (rule 12,
   line 38; transition rows, lines 78 and 80; Edge Case 239, line 239). Rule 21 (line 47)
   and UI 4 (line 331) say no `L = live` path may isolate the other instance's file, and
   the load table (line 66), Edge Case 251, AC 23 (line 402) and AC 24b (line 403) now
   implement that. Four normative sites still authorise 「新建生涯」 (rule 12, transition
   line 78, Edge 239) or 「删除生涯」 (transition line 80, `任意`) without a lock
   condition. Implemented literally, a second instance holding `ReadOnly`/`Blocked` could
   rename the primary that the first instance is between "主档改名备份" and "临时文件改名
   主档" on — the exact loss rule 21 exists to prevent. Three row/bullet edits; no schema,
   formula or state-model change.

### Recommended Revisions

1. `ReadOnly` branch still has no outcome for `readable(P) ∧ ¬valid(P)`, and no outcome
   for an in-memory migration that fails (formula line 138 tests `readable(P)` only,
   while lines 36/65/163 promise a rendered browse). Give the branch a defined failure
   result (fall through to `Blocked` 迁移失败) or state that `ReadOnly` implies
   `valid(P)` and a successful in-memory migration. Carried.
2. `ui-flow.md:402` (Edge Case) still gives the reason-keyed exit set with no live-lock
   override, and `:419` still says the `Blocked` modal accepts "其两个出口"; rule 20
   (`:81`), the table (`:504`) and AC 26 (`:586`) are now lock-aware, so the file
   contradicts itself. `:43` and `:144` are shorthand summaries. Fix in `ui-flow.md`
   (under edit by another session — coordinate before touching). Carried, narrowed.
3. `registry` has no `save_backup_count` entry although rule 9 (line 35) defines
   `SAVE_BACKUP_COUNT = 1`. Carried.
4. `save_bytes` lower bound is 0 (GDD line 208; registry line 819) but `B_base = 1024`.
   Carried.
5. Rule 2's example list (line 28) omits `base_capacity`, the same class of
   config-derived runtime value as `dish_pool`. Carried.
6. Edge Case 240 says only "纪录类事务保留内存", while rule 18 (line 44) and the state
   table (line 76) also cover 结算入账 and 解锁. Carried.
7. AC 1 (line 368) still asserts an empty unlock set at `Fresh` with no mention of the
   `normalize()` backfill Edge Case 234 requires before the baseline write. Carried.
8. The interface table (lines 100–109) still omits the outbound `normalize()` call and
   the startup order (data `Ready` → `load()` → `normalize()` → baseline), stated only in
   Edge Case 234 and Dependencies line 273. Carried.
9. Cross-References "待回改" items 1–4 (lines 356–359) contradict Dependencies lines
   285–288, which record those same documents as already 回改; item 5 (line 360) still
   asks for index `#4 = Designed` while `systems-index.md:60` reads `In Review`. Carried.
10. `ReadOnly` session behaviour for a run is still undefined: UI 3 (line 325) hides
    purchase/unlock entry points but does not forbid 「开始营业」, and every write is
    disabled in `ReadOnly`, so a played run's settlement can never commit. Either forbid
    starting a run or show an explicit "本局结果不会保存" notice. Carried.
11. **New** — `retry_load()`'s calling condition (line 103: "仅在 `Blocked`（迁移失败 /
    持锁不可读）时") and the transition row (line 79) are narrower than the load table's
    lock-first offer (line 66), which gives 「重试」 for every live-lock `Blocked`,
    including `lock_live ∧ version(P) > V`. Widen both to "任意 `lock_live` 的 `Blocked`"
    or narrow the table; today AC 24b (line 403) assumes the wider reading.

### Nice-to-Have

1. AC 24b (line 403) is numbered before AC 24 (line 404). Renumber if the section is
   touched again. (Carried.)
2. The retry transition (line 79) uses `Unloaded` → `Loading`, two states the
   load-outcome model never defines. (Carried.)
3. `retry_load()` on a 迁移失败 reason re-runs a pure function and must fail again (AC 30
   defines this honestly); consider suppressing 「重试」 for that reason in the UI.
   (Carried.)
4. `menu-waves.md:215` says a missing id makes the load fail and return to the menu;
   save rule 14 (line 40) preserves unknown ids instead. Unreachable in MVP (no mid-run
   save), but the two must agree if Open Question 1 is enabled. (Carried.)
5. UI 3 (line 325) describes `ReadOnly` as "另一实例持锁但文件可读时" without repeating
   `version ≤ 当前`; the outcome definition already implies it, so this is wording only.
   (Carried.)
6. Header still reads `Last Updated / Last Verified: 2026-11-03` although the file was
   edited on 2026-11-04 (rule 10 and the load table). Housekeeping.

### Scope Signal

**Rough scope signal: L** — four formula blocks, one upstream + four hard downstream
systems, and four architecture-stage decisions (serialization format OQ 2, save path and
loader OQ 3, write-lock implementation OQ 9; Steam Cloud OQ 4 later).
`systems-index.md:200` rates the design effort `S`; this review retains **L**.

### Verdict: NEEDS REVISION

Suggested systems-index status: **Needs Revision** (`systems-index.md:60` currently
reads `In Review`). Per the parallel constraint this review did not modify the index, the
GDD, the registry or `ui-flow.md`; the orchestration session applies the status. The four
claimed main fixes are verified as specified, but the blocking set is not empty: the
live-lock qualifier reaches the load-result table only, while rule 12, the two transition
rows and Edge Case 239 still authorise renaming a live instance's file. Four small edits
(no schema, formula or state-model change) clear the blocking.

---

Receipt (files this review read for cross-document facts):
```
Reviewed-Content-Hash: design/gdd/save-system.md 16b4a4a8fc45095392100202545509c7305a244f
Reviewed-Content-Hash: design/registry/entities.yaml 66d713b708546a4f5beb044b8abe9f93d7fd04e1
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/ui-flow.md 22c40c147277af09d1d3c9e3c8be149bfe0126f8
Reviewed-Content-Hash: design/gdd/systems-index.md b59ef2478102399659f8ae8b533c98a63a1c90b0
Reviewed-Content-Hash: project.yaml 19e09c36a420a92da7473431c9c3b4ff2ccb755d
```

---

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: L
Specialists: none (lean)
Blocking items: 0 | Recommended: 4
Summary: 上一轮四个锁盲出口全部关闭：规则 12、转移表的新建生涯/删除行、Edge 239/253 与 UI 4 均以 `lock_live` 门控；`retry_load()` 现在覆盖存活锁下的任意原因。剩余事项是接口与测试描述的完整性，不阻断实现。注册表在本次复审末尾为菜单波次条目做过同步，存档自己的条目未变。
Prior verdict resolved: Yes
Findings:
- [RECOMMENDED] 对外接口: `delete_career()` 行未重申 `lock_live = false` 前置条件（规则 23 已定义）。
- [RECOMMENDED] Acceptance Criteria: AC 30 仍只列「迁移失败 / 持锁不可读」，未覆盖新支持的「存活锁 + 版本过高」重试路径。
- [RECOMMENDED] 载入公式: `ReadOnly` 分支只测试 `readable(P)`，未定义可读但结构非法、或内存迁移失败的归属（carried）。
- [RECOMMENDED] Registry: `save_bytes.output_range` 下界为 0，但 `B_base = 1024`（carried）。

Reviewed-Content-Hash: design/gdd/save-system.md e2c1577cef86a4a400c5a617fde995fb85d6d0c2
Reviewed-Content-Hash: design/registry/entities.yaml 08de917004eb720428b21dee6a0359c90b31c5ec
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/ui-flow.md 4106b3e9f404f2c7610053bb8b5b19f72c1b186d
