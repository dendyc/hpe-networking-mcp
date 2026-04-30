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

## Milestone: v1.1 — AOS8-Powered Migration Readiness

**Shipped:** 2026-04-30  
**Phases:** 4 (10-13) | **Plans:** 6 | **Duration:** 2 days (2026-04-29 → 2026-04-30)

### What Was Built
- `aos-migration-readiness` skill rewritten: session-start AOS8 detection + 4-batch live API collection (9 AOS8 tools replacing 16 CLI paste commands)
- Stage 3 live VSG rules — VRRP VIP, ARM profiles, static AP IPs flagged as REGRESSION/DRIFT from live data
- Stage 4 Central enrichment — AP count gap, per-model AOS10 firmware rec table, SSID/role/VLAN conflict detection (REGRESSION)
- Stage 5 cutover prerequisites — cluster L2-connected health, controller firmware floor (fresh `show version`), pre-cutover AP baseline
- Executive summary paragraph + output hygiene rules + PARTIAL matrix row + `platforms:[central, aos8]` frontmatter

### What Worked
- **Skill-only scope discipline** — limiting v1.1 to markdown changes (no Python) eliminated the risk of breaking existing tools or tests. All 47 AOS8 tools from v1.0 were available and needed no modification.
- **Two-sub-path pattern** — the live-mode / paste-fallback structure per stage made partial failures graceful and kept AOS6/IAP paths untouched. This pattern is reusable for any future conditional skill behavior.
- **Single-call firmware recommendation** — recognizing that `central_recommend_firmware()` returns fleet-wide data (no per-model iteration needed) simplified Stage 4 significantly and honored the D-05 constraint without extra logic.
- **Audit-before-tag cadence** — running `/gsd:audit-milestone` before closing caught the ENRICH-02 A13 supersession gap and QUALITY-03 meta-tool loop gap. The quick task (260429-ta0) fixed both in ~30 min.
- **Test regression gate from Phase 10-01** — adding the `|aos8` regex alternation and 9-tool parametrized catalog assertion in the first plan meant every subsequent skill edit was instantly validated. Zero tool-name typos reached review.

### What Was Inefficient
- **SUMMARY.md `requirements-completed` frontmatter gap** — Phases 11 and 12 SUMMARY files are missing the `requirements-completed` frontmatter field (only Phase 10 and 13 have it). This created audit tracking debt. Executors should populate this field even for markdown-only phases.
- **Nyquist VALIDATION.md for markdown phases** — Phases 11, 12, 13 all have `nyquist_compliant=false` because no new unit tests were added (markdown-only). The wave_0_complete=false status creates false-negative noise in the audit. Need a clearer convention for markdown-only phases: either mark `not_applicable` or skip the VALIDATION.md.
- **gsd-tools CLI one-liner extraction** — the `milestone complete` CLI couldn't extract one-liners from 3 of 6 SUMMARY files (fallback header parsing mismatch). Required manual MILESTONES.md cleanup. SUMMARY files for markdown-only phases use `**One-liner:** ` inline format instead of a block — CLI should handle both.

### Patterns Established
- **Skill two-sub-path pattern** — use a Stage -N detection block to branch skill paths; all branches under the same stage heading with clear `used when Stage -1 announced X` labels.
- **Per-batch fallback prose** — on batch failure, identify only failed commands; never reprint the full paste table. Keeps prose tight and operator-friendly.
- **Naming conflicts = REGRESSION** — conflicts in SSID names, roles, or VLAN IDs are migration blockers; emit one REGRESSION bullet per conflicting item with explicit conflict description.

### Key Lessons
1. **Scope skill milestones to existing tools when possible** — if the platform tools already exist, a skill-only milestone is far faster and safer than a mixed code+skill milestone. v1.1 shipped 21 requirements in 2 days.
2. **`central_recommend_firmware()` returns fleet data** — a single no-filter call returns recommendations for all AP models; per-model iteration is unnecessary and creates duplicate-call risk.
3. **Guard explicit "fresh call" constraints in skill prose** — CUTOVER-02 needed a fresh `show version` call (not Batch 3 reuse). Adding "do NOT pull firmware from Batch 3" inline eliminated the Pitfall 2 ambiguity.
4. **Markdown-only SUMMARY files need `requirements-completed` too** — even if no code changes, populating the frontmatter field avoids audit noise and makes requirement tracking automatic.

### Cost Observations
- Model mix: primarily claude-sonnet-4-6
- Sessions: multi-session across 2 days
- Notable: All 6 plans were markdown edits — fastest milestone to date at 6 plans covering 21 requirements

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Duration | Phases | Key Change |
|-----------|----------|--------|------------|
| v1.0 | 2 days | 9 | First milestone — baseline established |
| v1.1 | 2 days | 4 | Skill-only scope — 21 requirements, 0 Python changes |

### Cumulative Quality

| Milestone | Tests | Platforms | AOS8 Tools |
|-----------|-------|-----------|-----------|
| v1.0 | 766 | 7 | 47 + 9 prompts |
| v1.1 | 790 | 7 | 47 + 9 prompts (unchanged — skill milestone) |

### Top Lessons (Verified Across Milestones)

1. Mirror real return types in mocks — frozen mocks that return wrong types mask production bugs
2. Post-milestone audit before tagging is worth the time — catches real bugs, not just planning drift
3. Skill-only milestones can cover significant functional scope with minimal risk — scope to existing tools when possible
