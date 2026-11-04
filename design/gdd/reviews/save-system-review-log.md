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
