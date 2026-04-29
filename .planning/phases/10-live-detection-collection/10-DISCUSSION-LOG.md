# Phase 10: Live Detection & Collection - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-29
**Phase:** 10-live-detection-collection
**Areas discussed:** Detection anchor, Paste section fate, Live collection narration, Partial collection policy

---

## Detection Anchor

| Option | Description | Selected |
|--------|-------------|----------|
| Before Stage 0 | Skill calls health() at start, announces "AOS8 API mode" before Stage 0 interview begins | ✓ |
| Lazy (Stage 0-gated) | Detection only runs after operator confirms source=aos8 in Stage 0 | |
| Both — announce upfront, re-confirm at Stage 0 | health() probe + brief announcement, then Stage 0 confirms | |

**User's choice:** Before Stage 0 (recommended)
**Notes:** Operator knows upfront that paste won't be needed for AOS8.

---

## Detection — Paste Fallback Announcement

| Option | Description | Selected |
|--------|-------------|----------|
| Silent fallback | If AOS8 unavailable, proceed into Stage 0 interview with no announcement | ✓ |
| Explicit announcement | State "AOS8 API not connected — using paste mode" before Stage 0 | |
| You decide | Claude picks based on cleaner prose | |

**User's choice:** Silent fallback (recommended)
**Notes:** Existing paste UX unchanged when AOS8 not connected.

---

## Paste Section Fate (Stage 1 AOS8 Table)

| Option | Description | Selected |
|--------|-------------|----------|
| Remove entirely | 16-command paste table deleted; Stage 1 AOS8 path is API-only | ✓ |
| Keep as paste-mode fallback | Stage 1 split into API path + paste fallback sub-sections | |
| Conditionalize with a note | Keep table with "Paste mode only — skip if using API mode" header | |

**User's choice:** Remove entirely (recommended)
**Notes:** Commands are still in v1.0 TOOLS.md / INSTRUCTIONS.md for reference.

---

## Live Collection Narration

| Option | Description | Selected |
|--------|-------------|----------|
| Minimal announcement | Single "Collecting via AOS8 API..." line, then compiled results | |
| Narrated per-tool | Each tool call announced individually | |
| Grouped by COLLECT req | 4 batch announcements matching COLLECT-01..04 | ✓ |

**User's choice:** Narrated per-tool → then narrowed to Grouped by COLLECT req (4 batches)
**Notes:** Balances transparency with conciseness. Each batch gets one narration line then tool calls then brief confirmation.

---

## Partial Collection Policy

| Option | Description | Selected |
|--------|-------------|----------|
| Hybrid — API data + paste for gaps | Use successful API results; request paste only for failed items | ✓ |
| Full paste fallback | Any failure → entire AOS8 path switches to paste mode | |
| Proceed with flagged partial data | Continue audit; missing data flagged as UNKNOWN | |

**User's choice:** Hybrid (recommended)
**Notes:** Per-batch granularity — a failure in one batch doesn't invalidate other batches' data.

---

## Partial Collection — Paste Request Specificity

| Option | Description | Selected |
|--------|-------------|----------|
| Exact commands only | Tell operator which specific CLI command to run for failed item | ✓ |
| Reprint relevant rows | Paste relevant rows from Stage 1 table including VSG anchor | |
| You decide | Claude picks what's clearest | |

**User's choice:** Exact commands only (recommended)
**Notes:** e.g., "run `show ap database long` and paste the output" — no table reprint.

---

## Claude's Discretion

- Exact wording for "AOS8 API mode — live data" announcement
- Exact wording for per-batch narration lines
- Format for compiled Stage 1 inventory after collection
- Whether collection batches are sequential or indicated as parallel

## Deferred Ideas

None.
