# Review Log — 菜品数值（Dish Values）

## Revision — 2026-11-06 — GDD Revision Flags write-back

Closed the `architecture-review-2026-11-05` GDD Revision Flags for this system; this entry records a revision, not a fresh review.

- Flag #2 — rule 11 now states the single runtime exception to "`opt_value` does not enter runtime": the menu-waves health gate computes `wave_reachable_opt` in-process as the `wave_healthy` lower-bound baseline, and the result stays inside `HealthEval` (not exported to HUD / settlement / save, not a public API). Source: ADR-0006 §4.

Status: substantive edit — the GDD returns to `In Review` per `.claude/rules/design-docs.md`; approval requires a fresh full `/design-review`.

Reviewed-Content-Hash: design/gdd/dish-values.md 72cf43fcd49019dd6e5281ed8f6d7df8705a71e0
