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
