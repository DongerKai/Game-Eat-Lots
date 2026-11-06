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
