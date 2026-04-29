# Phase 7: Testing & Integration - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-28
**Phase:** 07-testing-integration
**Areas discussed:** DIFF tools gap, Token-leak audit scope, Plan structure, Coverage enforcement

---

## DIFF Tools Gap

| Option | Description | Selected |
|--------|-------------|----------|
| Implement them in Phase 7 | Extend Phase 7 scope to include building all 9 DIFF tools + tests. Phase 4 context has design decisions captured. | ✓ |
| Skip them entirely | Phase 7 tests only what was built (Phases 1–6). DIFF tools noted as out-of-scope for this milestone. | |
| Defer to a Phase 8 | Create a new Phase 8 for DIFF tools after Phase 7 completes. | |

**User's choice:** Implement them in Phase 7

**Follow-up — All 9 or trimmed:**

| Option | Description | Selected |
|--------|-------------|----------|
| All 9 DIFF tools | Full original Phase 4 scope: DIFF-01..09 | ✓ |
| Core 8 only, skip DIFF-09 | Skip aos8_get_md_health_check (the aggregator) | |
| Pick specific ones | Choose which DIFF tools matter most | |

**User's choice:** All 9 DIFF tools

---

## Token-Leak Audit Scope

| Option | Description | Selected |
|--------|-------------|----------|
| One cross-cutting test | Single test invoking a read + write tool while capturing logs | ✓ |
| Per-category coverage | Token-leak assertion in each category test file | |
| Client-only is sufficient | Existing test_uidaruba_never_in_logs already sufficient | |

**User's choice:** One cross-cutting test (in `test_aos8_security.py` or appended to `test_aos8_client.py`)

---

## Plan Structure

| Option | Description | Selected |
|--------|-------------|----------|
| 3 plans mirroring Phase 4 design | Wave 0 → Implementation → Wiring + Regression | ✓ |
| 2 plans: implement then verify | DIFF tools + tests in one pass, then CI/regression | |
| 1 plan: everything together | Single plan, simpler but harder to checkpoint | |

**User's choice:** 3 plans — Plan 07-01 (Wave 0 tests, RED), Plan 07-02 (implement differentiators.py, GREEN), Plan 07-03 (wire TOOLS dict + full regression)

---

## Coverage Enforcement

| Option | Description | Selected |
|--------|-------------|----------|
| No coverage floor | Just confirm all tests pass | ✓ |
| Add coverage threshold | Set floor (e.g., 80%) in pyproject.toml; CI fails below it | |
| Coverage report only | Generate report in CI but don't fail on it | |

**User's choice:** No coverage floor — success = all tests pass

---

## Claude's Discretion

- File placement for cross-cutting token-leak test (new file vs appended class)
- Exact AOS8 REST endpoints for DIFF tools — researcher verifies
- JSON fixture file names and structure for DIFF tests
- Whether config_path is scope-sensitive for DIFF-04, 06, 07, 08

## Deferred Ideas

None.
