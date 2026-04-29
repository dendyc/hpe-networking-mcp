# Phase 13: Executive Output & Quality Gate - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-29
**Phase:** 13-executive-output-quality-gate
**Areas discussed:** Executive summary structure, Clean output guardrails, PARTIAL verdict extension, Frontmatter audit

---

## Executive Summary Structure

| Option | Description | Selected |
|--------|-------------|----------|
| Top of report, before header block | Freestanding paragraph before the Captured/Verdict header block — the SE reads it first | ✓ |
| Right after the Verdict: line | Kept inside the header block — verdict field stays, prose paragraph expands inline | |
| New section after source inventory | Summary section after inventory and target-side state | |

**User's choice:** Top of report, before header block
**Notes:** SE can copy the paragraph into a customer email without scrolling past structured data.

---

## Executive Summary Content

| Option | Description | Selected |
|--------|-------------|----------|
| Verdict + counts + SE sentence | GO/BLOCKED/PARTIAL, X REGRESSIONs/Y DRIFTs/Z INFOs, plus one SE-ready prose sentence | ✓ |
| Verdict + counts only | Just verdict and finding counts — no SE-narrative sentence | |
| Verdict + top 3 REGRESSIONs named | Names the most critical blockers by title in the summary | |

**User's choice:** Verdict + counts + SE sentence
**Notes:** Always 2–4 sentences, plain prose, never a bullet list or table.

---

## SE-Ready Sentence Context

| Option | Description | Selected |
|--------|-------------|----------|
| AP count + controller count + source platform | E.g. "This AOS8 deployment (47 APs, 2 controllers) has 3 blockers..." | ✓ |
| Source + target platform only | E.g. "This AOS8 → AOS10 Tunnel Mode migration has 3 blockers..." | |
| Scope label from Stage 0 only | E.g. "This multi-site AOS8 migration has 3 blockers..." | |

**User's choice:** AP count + controller count + source platform
**Notes:** Enough for a customer to recognise their own environment without reading further.

---

## Clean Output Enforcement Method

| Option | Description | Selected |
|--------|-------------|----------|
| Explicit prose rules in Output formatting section | Add 3–4 bullet rules: no raw JSON, no tool-call syntax in finding text, no stack traces, no ellipsis | ✓ |
| Template-only enforcement | Existing finding template already shows correct format | |
| Stage-level notes alongside each sub-path | One-liner per live-mode sub-path in Stages 3, 4, and 5 | |

**User's choice:** Explicit prose rules in Output formatting section
**Notes:** AI can't reliably infer these constraints from the template alone.

---

## Clean Output Artifact Types

| Option | Description | Selected |
|--------|-------------|----------|
| Raw JSON blobs from API responses | No dumping full tool response | ✓ (recommended) |
| Tool call syntax in finding text | Tool names only in parenthetical source attribution | ✓ (recommended) |
| Stack traces / Python exceptions | Error message only, not full traceback | ✓ (recommended) |
| Ellipsis / truncated output markers | No "...", "[truncated]" — extract fields explicitly | ✓ (recommended) |

**User's choice:** All four (accepted recommendation)
**Notes:** All four take one line each; skipping any leaves a gap. Each addresses a distinct failure mode.

---

## PARTIAL Verdict Extension

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — add live-mode PARTIAL row to decision matrix | New row: AOS8 live mode AND one or more Stage 1 batches failed → PARTIAL | ✓ |
| No — Phase 10 hybrid fallback covers it | D-07..D-09 already handle this in Stage 1 narration | |
| Yes — in exec summary only, not decision matrix | Exec summary notes partial success; matrix unchanged | |

**User's choice:** Yes — add live-mode PARTIAL row to decision matrix
**Notes:** Current decision matrix only defines PARTIAL for "operator hasn't pasted the bundle yet" — the live-mode gap is real and should be explicit.

---

## Frontmatter Audit Approach

| Option | Description | Selected |
|--------|-------------|----------|
| Verify by diff + add aos8 to platforms: | Explicit diff of AOS8 tool names in skill body vs tools: list, then update platforms: | ✓ |
| Just add aos8 to platforms: — trust Phase 10 | Phase 10 added all AOS8 tools; just update platforms: and run the test | |

**User's choice:** Verify by diff + add aos8 to platforms:
**Notes:** Mechanical but makes the check auditable; Phase 13 plans document the diff result.

---

## Claude's Discretion

- Exact wording of the executive summary paragraph template in the skill instructions
- Whether "Output hygiene rules" subsection appears before or after the existing format template block
- Exact phrasing of the PARTIAL live-mode row in the decision matrix

## Deferred Ideas

None — discussion stayed within phase scope.
