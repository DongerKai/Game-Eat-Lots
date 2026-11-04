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
