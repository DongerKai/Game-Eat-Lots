# Gate Check: Technical Setup → Pre-Production

**Date**: 2026-11-04
**Checked by**: gate-check skill
**Resolved settings**: workflow `standard` · qa.level `standard` · review_mode `lean` · team.size `individual` · performance.enforce `warn`
**HEAD at check**: `a9043f4` (docs(architecture): break the ADR dependency cycle and complete the boot gate)

> Detected stage vs configured stage: configured `project.stage` is **Concept**, but artifacts already
> pass Systems Design (18 systems Approved) and have entered Technical Setup. The configured stage is
> stale; on a FAIL it is left untouched.

## Required Artifacts (standard tier, after reductions): 6/9 present

| Item | Status | Notes |
|---|---|---|
| Engine chosen | PRESENT | Godot 4.7 (GDScript, Forward+, `project.yaml` + `CLAUDE.md`) |
| Project config populated | PRESENT | engine/naming set; performance budgets in `.claude/docs/technical-preferences.md`; `commands.build` still `[TO BE CONFIGURED]` |
| ≥3 Foundation ADRs | PRESENT | 9 ADRs, 41K–64K each |
| Engine reference docs | PRESENT | `docs/engine-reference/godot/{VERSION,breaking-changes,current-best-practices,deprecated-apis}.md` + `modules/` |
| Test framework root (`tests/unit/`, `tests/integration/`) | **MISSING** | no `tests/` directory anywhere in the repo |
| CI test workflow `.github/workflows/tests.yml` | **MISSING** | `.github/` has only CODEOWNERS, FUNDING.yml, ISSUE_TEMPLATE, PR template |
| ≥1 example test file | **MISSING** | no gdUnit4 addon (`addons/` absent) |
| Master architecture doc | PRESENT | `docs/architecture/architecture.md` (45K) |
| `/architecture-review` run (Mode: full) | PRESENT (stale) | `docs/architecture/architecture-review-2026-11-04.md` (34K), `Mode: full`, Verdict FAIL |
| Art bible §1–4 | N/A at standard | Required only if visual-asset stories exist — none exist; sections 1–9 present anyway |
| Traceability index (recommended at standard) | PRESENT | `requirements-traceability.md` (24K) + `tr-registry.yaml` (30K) |
| Accessibility doc (recommended) | PRESENT | `design/accessibility-requirements.md` (51K), Tier Target Standard |
| `interaction-patterns.md` (recommended) | **MISSING** | `design/ux/interaction-patterns.md` |

## Quality Checks: 9/11 passing, 1 concern, 1 NOT ASSESSED

| Item | Status | Evidence |
|---|---|---|
| Architecture decisions cover core systems | PASS (concern) | Input ADR-0005, state/transaction ADR-0007, composition ADR-0004; no dedicated rendering ADR. Core-layer requirement coverage is 15 gaps (below) |
| Naming conventions + performance budgets | PASS | 60 fps / 16.6 ms / ≤500 draw calls / 512 MB |
| Accessibility tier defined | PASS | Standard, committed; 3 Standard items (input rebinding, colorblind toggle, tutorial replay) lack GDD carriers |
| At least one screen UX spec | PASS | `design/ux/hud.md` (73K) |
| All ADRs have Engine Compatibility + version | PASS | 9/9; engine pinned 4.7.x, not installed on host |
| All ADRs have GDD Requirements Addressed | PASS | 9/9; ADR-0009 has the section at L587 (the review's contrary claim is false) |
| No deprecated engine API usage | PASS | Review full-mode audit: 0 hits across 9 ADRs |
| HIGH RISK engine domains addressed | CONCERNS | Addressed/flagged, but 49 `Verification Required` items unrun on pinned 4.7.x |
| RTM zero Foundation-layer gaps | PASS | Foundation 24 → 23 covered / 1 partial / **0 gaps** |
| ADR circular dependency | PASS | `adr-dep-graph.sh`: 9 ADRs, 19 edges, no `CYCLE:` / `BLOCKED_BY_CYCLE:`; ADR-0005 has no outgoing edge |
| Test check executable (qa.level: standard) | **NOT ASSESSED** | no test runner, no gdUnit4, Godot binary not on PATH; `commands.build` placeholder |

Coverage per review (unchanged by this gate): 104 requirements — 0 Accepted-ADR-covered,
56 Proposed-covered, 4 partial, 44 gaps. Layers: Foundation 24 → 23/1/0; Core 21 → 4/2/**15**;
Feature 34 → 12/1/21; Presentation 25 → 17/0/8.

## Director Panel Assessment

Panel: 2 of 4 (`workflow: standard`). Creative and Art perspectives not consulted; raise
`modes.workflow` to `full` for the complete panel.

```
Technical Director: NOT READY
  - Test infrastructure entirely absent: tests/unit, tests/integration, gdUnit4 addon,
    .github/workflows/tests.yml, example test.
  - Godot 4.7.x not installed; commands.build is a placeholder; 49 engine verifications unrun
    (4 of them shape Foundation implementation: push_input→GUI nav, focus_neighbor fallback,
    Tween under ALWAYS, AudioStreamPlayer under paused).
  - Re-verified three review claims against HEAD and found them false: no dependency cycle,
    ADR-0009 does have GDD Requirements Addressed, ADR-0001/0002 are Accepted.

Producer: NOT READY
  - Same three required artifacts missing; without a runnable engine the test/CI floor cannot
    exist, and the vertical slice has no regression net.
  - Critical-path root is installing/pinning Godot 4.7.x; tests, CI, the 49 verifications and
    the slice all queue behind it.
  - Core ADR gaps are a slice-phase risk, not an entry criterion; the remaining plan
    (11 ADRs, 9 accepts, slice + playtest, menu/pause UX, 3 accessibility calls) is ownable
    by one developer once the engine exists.
```

## Blockers

1. **Test framework root missing** — create `tests/unit/` and `tests/integration/` and
   initialize gdUnit4 (`/test-setup`).
2. **CI test workflow missing** — add `.github/workflows/tests.yml` (`/test-setup`).
3. **No example test file** — the framework is unproven (`/test-setup`).
4. **Test check NOT ASSESSED** — Godot 4.7.x is not installed, `commands.build` is
   `[TO BE CONFIGURED]`, and 49 `Verification Required` items are unrun. The tier's
   Logic/Integration test gate could not execute.
5. **Both directors returned NOT READY.**

## Recommendations

- Run `/test-setup` first (closes blockers 1–3), then install/pin Godot 4.7.x and set
  `commands.build` so the suite can actually run (closes blocker 4).
- Re-run `/architecture-review full`: the 2026-11-04 report is stale — its dependency cycle
  (fixed in `a9043f4`), its ADR-0009 claim, and its "all 9 Proposed" claim no longer hold.
- Before the first Foundation story: accept ADR-0003/0004/0005/0008/0009 (7/9 remain Proposed).
- Before Core stories: write the missing Core/Feature ADRs (Core 15 gaps: time-and-turns 4,
  stomach-capacity 5, dish-values 4, run-flow 2; 44 gaps total → 11 ADRs required per review).
- Initialize the interaction-pattern library (`/ux-design patterns`) — recommended at standard.
- Producer decision on 3 Standard accessibility items with no GDD carrier: add or formally downgrade.
- Doc drift: `control-manifest.md` cites a `project.yaml` `testing` key that does not exist;
  `project.stage` / `production/stage.txt` still read `Concept`.

## Verdict: **FAIL**

- Required artifacts missing (test roots, CI, example test) — blocking at `standard`.
- Quality check could not execute (test suite) — NOT ASSESSED, and FAIL takes precedence.
- Both directors NOT READY.
- Stage is **not** advanced: `project.yaml` and `production/stage.txt` stay `Concept`.

## Chain-of-Verification

5 questions checked — verdict unchanged.

1. Hard blockers separated from recommendations? Re-read the gate file's `standard` reductions —
   test roots/CI/example test are unrelaxed; RTM/accessibility/interaction-patterns are recommended. [TOOL ACTION]
2. Any PASS item too lenient? The architecture-review artifact exists but is stale; recorded as
   PRESENT (stale) and a CONCERNS, not a clean pass. Independent blockers stand regardless.
3. Additional blockers missed? Re-ran `artifact-check.sh --phase technical-setup` and re-checked
   `tests/`, `addons/`, `.github/workflows/` at HEAD `a9043f4` — all still absent. [TOOL ACTION]
4. Minimal path to PASS? (a) `/test-setup` creates roots + CI + example test; (b) install/pin
   Godot and set the build command; (c) re-run `/architecture-review full` so its stamps match HEAD.
5. Resolvable? Yes — all three are mechanical and local; no design rework is needed.
