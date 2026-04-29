# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.0 — AOS8 MVP

**Shipped:** 2026-04-29  
**Phases:** 9 | **Plans:** 26 | **Duration:** 2 days (2026-04-27 → 2026-04-29)

### What Was Built
- 47 AOS8 tools (26 read + 9 differentiator + 12 write) + 9 guided prompts wired as a 7th platform module
- `AOS8Client` with UIDARUBA token reuse, asyncio.Lock refresh serialization, lazy login, and end-to-end token masking
- 766-test suite covering all AOS8 surfaces plus non-regression across all 6 existing platforms
- Full operator documentation: INSTRUCTIONS.md (config_path semantics, write_memory contract), README, TOOLS.md, CHANGELOG

### What Worked
- **Wave 0 TDD discipline** — writing red test scaffolds before implementation gave each plan a clear contract and eliminated ambiguity during green phase. Plans converged faster when the contract was pinned in test fixtures.
- **Platform pattern reuse** — following the existing `central`/`apstra`/`axis` platform structure exactly (same file layout, same importlib registration, same lifespan pattern) meant zero friction integrating with middleware, dynamic mode, and the health probe.
- **Post-audit gap closure** — running `/gsd:audit-milestone` before tagging caught a real production bug (DIFF tools returning raw `httpx.Response`) and planning drift (Phase 4 not formally closed). Adding Phases 8 and 9 for proper closure was the right call.
- **Merge-into-Phase-7 decision** — absorbing Phase 4 differentiator work into Phase 7 (integration/testing) kept the plan count lean and delivered a coherent testing phase.

### What Was Inefficient
- **Local response helpers as footgun** — Phase 7 introduced `_show`/`_object` helpers in `differentiators.py` instead of using the canonical `_helpers.run_show`/`get_object`. Frozen test mocks masked the bug. Phase 8 had to fix this retroactively — cost an extra phase.
- **SUMMARY.md frontmatter coverage** — many plan SUMMARY files have empty `requirements-completed` fields. This is purely administrative hygiene but creates audit noise. Future phases should populate this field during execution.
- **Merge conflict in STATE.md** — a stale worktree conflict marker in the performance metrics table wasn't caught until milestone completion. Worktree merges should include a conflict-check step.
- **REQUIREMENTS.md footer count drift** — the footer read "71/71" when actual count was 82 (due to PROMPT/TEST groups being added later). Small but created audit noise.

### Patterns Established
- **`_helpers.run_show`/`get_object` as mandatory** — every AOS8 tool that calls `client.request()` MUST call `.json()` before passing to `strip_meta()`. Use the canonical helpers; never re-implement locally.
- **Delegated VERIFICATION.md (score 0/0)** — when a phase is absorbed into another, create a `{phase}-VERIFICATION.md` with score `0/0 (DELEGATED)` and explicit cross-references to the absorbing phase. This prevents orphaned requirements in future audits.
- **Post-milestone audit before tagging** — `/gsd:audit-milestone` should be standard before `complete-milestone`. It caught a production bug on this milestone.

### Key Lessons
1. **Frozen test mocks mask production bugs** — if tests mock `client.request()` to return a plain dict instead of an `httpx.Response`, you can't detect the missing `.json()` call. Always mirror the real return type in mocks.
2. **Gap closure phases are worth it** — two extra phases (08, 09) after the audit added ~1 day but shipped a clean v1.0. The alternative (accepting the DIFF bug as tech debt) would have required a v1.0.1 patch immediately.
3. **Platform module structure scales well** — no modifications to existing code were needed. The AOS8 module dropped in cleanly. The existing pattern is solid and should be the template for any future 8th platform.
4. **Phase numbering should be locked early** — adding phases 8 and 9 after the fact was fine, but required updating ROADMAP.md coverage counts and created some doc drift. A cleaner approach: plan a "gap closure" phase slot in every roadmap.

### Cost Observations
- Model mix: primarily claude-sonnet-4-6 (fast execution)
- Sessions: multi-session across 2 days
- Notable: Phase 4 was delivered at zero additional cost by merging into Phase 7; audit-driven quality gate added ~1 day but eliminated a production bug

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Duration | Phases | Key Change |
|-----------|----------|--------|------------|
| v1.0 | 2 days | 9 | First milestone — baseline established |

### Cumulative Quality

| Milestone | Tests | Platforms | AOS8 Tools |
|-----------|-------|-----------|-----------|
| v1.0 | 766 | 7 | 47 + 9 prompts |

### Top Lessons (Verified Across Milestones)

1. Mirror real return types in mocks — frozen mocks that return wrong types mask production bugs
2. Post-milestone audit before tagging is worth the time — catches real bugs, not just planning drift
