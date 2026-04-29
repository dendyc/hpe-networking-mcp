---
phase: 10-live-detection-collection
plan: "02"
subsystem: skills
tags: [aos8, skill, detection, live-mode, migration-readiness, DETECT-01, COLLECT-01, COLLECT-02, COLLECT-03, COLLECT-04]

# Dependency graph
requires:
  - phase: 10-01
    provides: test_skill_tool_references.py regex extended for aos8_*, test_skill_aos8_live_detection.py parametrized catalog assertions
provides:
  - Stage -1 session-start detection block in aos-migration-readiness.md (DETECT-01)
  - AOS8 Stage 1 live-mode sub-path: 4 API batch narration blocks (COLLECT-01..04)
  - AOS8 Stage 1 paste-fallback sub-path: original 16-command table retained
  - Frontmatter tools list extended with all 9 AOS8 tool names
affects: [10-03-and-beyond, phase-11-rules, phase-12-enrichment, phase-13-output]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Two-sub-path structure for conditional skill behavior: live-mode (API detected) vs paste-fallback (API unreachable)"
    - "Session-start health() probe gates live vs paste path selection — detection never blocks the audit"
    - "Per-batch partial-failure policy: individual batch failure triggers targeted paste ask, not full table reprint"

key-files:
  created: []
  modified:
    - hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md

key-decisions:
  - "Two-sub-path structure under AOS8 Stage 1: live-mode when AOS8 reachable, paste-fallback when not — D-04 amendment honored"
  - "Stage -1 detection is silent on failure: no announcement, no UX change for AOS6/IAP/unreachable-AOS8 sessions (D-02)"
  - "Trim candidates 1-2-3-4 applied to hold 530-line ceiling: VSG anchor citations removed from batch narration, error-fallback prose compressed, Batch 1 step-2 object list moved inline, post-batch summary paragraph removed"
  - "Scenario A (AOS8 reachable live-mode announcement) deferred: operator does not have access to live AOS8 environment; prose is mechanically correct per skill instruction clarity — manual verify pending"

patterns-established:
  - "Pattern: skill two-sub-path — use Stage -N detection to branch skill paths; all branches under same stage heading"
  - "Pattern: per-batch fallback prose — identify only failed commands, never reprint full table in fallback request"

requirements-completed: [DETECT-01, COLLECT-01, COLLECT-02, COLLECT-03, COLLECT-04]

# Metrics
duration: ~60min
completed: "2026-04-29"
---

# Phase 10 Plan 02: AOS8 Live Detection & Collection Summary

**AOS8 skill rewritten with session-start health() detection gate + four-batch live API collection path, preserving AOS6/IAP paste flows byte-for-byte**

## Performance

- **Duration:** ~60 min
- **Started:** 2026-04-29
- **Completed:** 2026-04-29
- **Tasks:** 3 (Tasks 1 and 2 complete; Task 3 partially approved — Scenario A deferred)
- **Files modified:** 1 (skill markdown only)

## Accomplishments

- Extended frontmatter `tools:` list with all 9 AOS8 tool names (was 19 tools, now 28)
- Inserted `### Stage -1 — Session-start AOS8 detection (DETECT-01)` block before Stage 0 — calls `health()`, announces "AOS8 API mode — live data" when reachable, proceeds silently otherwise
- Replaced the original AOS8 Stage 1 CLI-paste-only section with a two-sub-path structure: live-mode (4 API batches mapping to COLLECT-01..04) and paste-fallback (original 16-command table preserved)
- AOS6 and IAP Stage 1 sections verified byte-unchanged — no regression
- All 790 unit tests pass (50 skill-specific, 790 total)
- Final file line count: 530 (at ceiling; trims applied)

## Task Commits

Tasks committed atomically in hpe-networking-mcp sub-repo:

1. **Task 1: Stage -1 detection block + frontmatter tools list extension** — `22c0f34` (feat)
2. **Task 2: AOS8 Stage 1 live-mode + paste-fallback sub-paths** — `af0fc95` (feat)
3. **Task 3: Operator smoke test** — partial approval; no code commit (checkpoint only)

## Files Created/Modified

- `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` — Stage -1 block added (lines 54-73), frontmatter tools list extended (line 23), AOS8 Stage 1 section replaced with live-mode + paste-fallback sub-paths (lines 95-165 approx). Line count: 530.

## Decisions Made

- **Two-sub-path structure under AOS8 Stage 1:** live-mode sub-path when Stage -1 announced AOS8 API mode; paste-fallback sub-path when Stage -1 was silent. D-04 amendment (2026-04-29) honored exactly — live-mode contains no paste table; paste table is retained as fallback only.
- **Trim candidates applied to hold 530-line ceiling:** Applied all four trim candidates from the plan:
  1. VSG anchor citations removed from most batch narration steps (kept in Batch 2 and Batch 3 step 1 only)
  2. Per-batch error-fallback prose compressed to short directives
  3. Batch 1 step-2 config object list moved inline (was multi-line enumeration)
  4. Post-batch summary paragraph removed
- **Stage -1 detection never blocks:** Failures and unreachables fall silently to paste flow per D-02. No error message shown to operator.

## Manual Verification Status (Task 3)

**Checkpoint type:** human-verify, gate: blocking

| Scenario | Description | Result |
|----------|-------------|--------|
| A | AOS8 reachable — AI announces "AOS8 API mode" before Stage 0 | DEFERRED — operator does not currently have access to a live AOS8 environment; prose is mechanically correct based on skill instruction clarity |
| B | AOS8 unreachable — AI proceeds silently, no announcement | PASSED |
| C | AOS6 paste table regression check | PASSED |

**Operator resolution:** PARTIAL APPROVAL — proceed with documented caveat. Scenario A deferred to future manual verification when AOS8 environment is available.

**Implication:** The skill's detection prose is in place and syntactically correct. The behavioral guarantee for Scenario A (actual LLM announcement when `health()` returns `aos8.status == "ok"`) remains unverified until a live AOS8 environment is available.

## Deviations from Plan

### Trim Candidates Applied

**1. [Plan-specified trim] Trim candidates #1-#4 all applied**
- **Reason:** File reached 530-line ceiling after Task 2 rewrite; all four trim candidates from the plan's action block were applied in order
- **Trim #1:** VSG anchor citations removed from Batch 1 step 1/3, Batch 3 steps 2-5, Batch 4 steps 1-4; retained in Batch 2 and Batch 3 step 1
- **Trim #2:** All four per-batch error-fallback paragraphs compressed to short directives
- **Trim #3:** Batch 1 step-2 object-type list moved inline as comma-separated list
- **Trim #4:** Post-batch summary paragraph removed entirely
- **Final line count:** 530 (within ceiling)
- **Impact:** No loss of operator-facing procedural content; traceability annotations and framing prose trimmed only

### Scenario A Deferral (Task 3 partial approval)

**2. [Checkpoint resolution] Scenario A deferred — no live AOS8 environment**
- **What was expected:** Operator would verify AI announces "AOS8 API mode" when `health()` returns `aos8.status == "ok"`
- **What happened:** Operator does not currently have access to a live AOS8 environment; Scenarios B and C passed
- **Impact:** Behavioral verification deferred; skill prose is mechanically correct; future verification needed when AOS8 environment available
- **Risk:** Low — skill text is unambiguous: `**If \`aos8.status == "ok"\`** → announce verbatim`. An AI following skill instructions will execute this.

---

**Total deviations:** 2 (1 plan-specified trim, 1 checkpoint partial approval)

## Self-Check: PASSED

Verification of SUMMARY claims:

- FOUND: `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md`
- Line count: 530 (at ceiling, within target)
- FOUND: commit 22c0f34 in hpe-networking-mcp sub-repo
- FOUND: commit af0fc95 in hpe-networking-mcp sub-repo
- FOUND: `Stage -1 — Session-start AOS8 detection (DETECT-01)` heading
- FOUND: `AOS8 API mode` announcement text
- FOUND: `AOS8 live-mode sub-path` heading
- FOUND: `AOS8 paste-fallback sub-path` heading
