# Phase 11: Live VSG Rules - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-29
**Phase:** 11-live-vsg-rules
**Areas discussed:** Stage 3 structure, Stage 2 live bypass, RULES-03 ClearPass timing, Rule parsing guidance

---

## Stage 3 Structure

| Option | Description | Selected |
|--------|-------------|----------|
| Mirror Stage 1 | Explicit AOS8 live-mode sub-path parallel to paste path, same pattern as Stage 1 | ✓ |
| Preamble clause only | Keep existing tables, add preamble referencing Stage 1 data | |
| New dedicated Stage 3-AOS8 section | Separate Stage 3-AOS8 section after Stage 2 | |

**User's choice:** Mirror Stage 1

---

### Stage 3 sub-question: Placement of live-mode sub-path

| Option | Description | Selected |
|--------|-------------|----------|
| Before the rule tables | Live-mode sub-path runs first, then universal rules continue | ✓ |
| Replace AOS8-specific rows only | Annotate C2/C4/U2 rows, everything else unchanged | |
| Parallel to paste in full AOS8 block | Full live-vs-paste split across entire AOS8 Stage 3 rules | |

**User's choice:** Before the rule tables

---

## Stage 2 Live Bypass

| Option | Description | Selected |
|--------|-------------|----------|
| Skip clause for AOS8 live path | Explicit skip clause at top of Stage 2 | ✓ |
| No change to Stage 2 | Stage 3 references Stage 1 data directly; Stage 2 unchanged | |
| Stage 2 AOS8 live annotation | Annotate extraction table rows with availability from Stage 1 | |

**User's choice:** Skip clause for AOS8 live path

---

## RULES-03 ClearPass Timing

| Option | Description | Selected |
|--------|-------------|----------|
| Defer to Stage 4 A11 | RULES-03 noted as pending in Stage 3; Stage 4 A11 expanded to do cross-check | ✓ |
| Fresh call in Stage 3 | Stage 3 calls clearpass_get_local_users() directly | |
| Collapse into Stage 4 A11 | Remove RULES-03 from Stage 3, expand A11 only | |

**User's choice:** Defer to Stage 4 A11
**Notes:** Avoids duplicate ClearPass call; Stage 4 has both data points naturally.

---

## Rule Parsing Guidance

| Option | Description | Selected |
|--------|-------------|----------|
| Field-level guidance | Name specific config object fields per rule | ✓ |
| Guidance-level only | Plain English, rely on AI's AOS8 schema knowledge | |
| Hybrid — guidance + fallback hint | Plain English + parenthetical field name hint | |

**User's choice:** Field-level guidance

---

## Claude's Discretion

- Exact wording of live-mode sub-path headers and intro lines
- Formatting of per-rule findings inline vs in final rule tables
- Fallback note language for rules where Stage 1 batch failed

## Deferred Ideas

None.
