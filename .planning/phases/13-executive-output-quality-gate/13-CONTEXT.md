# Phase 13: Executive Output & Quality Gate - Context

**Gathered:** 2026-04-29
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 13 modifies `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` (markdown only — no Python code changes) so that the skill renders a customer-grade report:

1. **Executive summary paragraph** (OUTPUT-01) — a freestanding prose paragraph at the very top of the report, before the existing header block, containing GO/BLOCKED/PARTIAL verdict, finding counts, and one SE-ready sentence for customer email copy.
2. **Clean markdown guardrails** (OUTPUT-02) — explicit prose rules added to the "Output formatting" section prohibiting raw JSON blobs, tool-call syntax in finding text, stack traces, and ellipsis/truncated markers.
3. **Decision matrix extension** — a new PARTIAL row covering AOS8 live-mode partial collection failures.
4. **Frontmatter audit + update** (QUALITY-01..03) — explicit diff of AOS8 tool names in skill body vs `tools:` frontmatter; add `aos8` to `platforms:` tag; regression test confirms all references resolve.

**Requirements in scope:** OUTPUT-01, OUTPUT-02, QUALITY-01, QUALITY-02, QUALITY-03

</domain>

<decisions>
## Implementation Decisions

### Executive Summary Placement (OUTPUT-01)

- **D-01:** The executive summary is a **freestanding paragraph at the very top of the report**, before the `## AOS migration readiness —` header block. It is the first thing the reader sees — the SE can copy it into a customer email without scrolling past structured data.

### Executive Summary Content (OUTPUT-01)

- **D-02:** The paragraph contains three elements, always in this order:
  1. **Verdict** — GO / BLOCKED / PARTIAL with capitalised label
  2. **Finding counts** — exact counts of REGRESSION / DRIFT / INFO findings
  3. **SE-ready sentence** — one plain-English sentence naming the source platform, AP count, controller count, and the key action: e.g. *"This AOS8 deployment (47 APs, 2 controllers) has 3 migration blockers; resolve VRRP VIP mismatch, static AP IPs, and firmware floor before scheduling cutover."*

- **D-03:** The paragraph is always exactly 2–4 sentences. Never a bullet list. Never a table. Plain prose only.

- **D-04:** The SE-ready sentence includes: source platform (AOS8/AOS6/IAP), AP count from live or parsed inventory, controller count (from MD hierarchy or paste), and a plain-language action summary. For PARTIAL verdict, the sentence notes which data is still missing.

### Clean Output Guardrails (OUTPUT-02)

- **D-05:** Add a **"Output hygiene rules"** subsection under "Output formatting" with 4 explicit prohibitions:
  1. **No raw JSON blobs** — extract and cite only the specific field value from any API response; never dump the full response dict.
  2. **No tool-call syntax in finding text** — tool names (e.g. `aos8_get_cluster_state()`) belong only in the parenthetical `(source: ...)` attribution, never in the finding sentence itself.
  3. **No stack traces** — if a tool call raises an exception, emit only a brief error note and the fallback CLI command; never include a Python traceback.
  4. **No ellipsis / truncation markers** — do not write `...`, `[truncated]`, or similar. Extract the relevant fields explicitly; if a response is large, summarise in prose.

- **D-06:** These rules apply to **all stages** of the report — Stages 3, 4, 5, and 6. They are stated once in "Output formatting" (not repeated per-stage) because the existing finding format template already illustrates correct style — the rules make the prohibition explicit for edge cases.

### PARTIAL Verdict Extension (Decision Matrix)

- **D-07:** Add a new row to the decision matrix in Stage 6:

  | Condition | Action |
  |---|---|
  | AOS8 live mode AND one or more Stage 1 batches failed | **PARTIAL** — list which AOS8 checks could not run (e.g. "Batch 3 failed: cluster state, local-user db"), plus the exact CLI commands needed to complete those data points manually. All checks that did succeed are still reported. |

  This closes the gap where the current matrix only defines PARTIAL as "operator hasn't pasted the data bundle yet."

- **D-08:** For the executive summary sentence in PARTIAL verdict (live-mode batch failure): *"AOS8 live collection partially succeeded — [N] checks completed; [M] checks require manual CLI paste (see below)."*

### Frontmatter Audit (QUALITY-01..03)

- **D-09:** Plan explicitly diffs AOS8 tool names referenced in the skill body against the `tools:` frontmatter list to confirm no gaps. Expected result: the 9 AOS8 tools added in Phase 10 (`aos8_get_md_hierarchy`, `aos8_get_effective_config`, `aos8_get_ap_database`, `aos8_get_cluster_state`, `aos8_show_command`, `aos8_get_clients`, `aos8_get_bss_table`, `aos8_get_active_aps`, `aos8_get_ap_wired_ports`) already cover all references added in Phases 11 and 12.

- **D-10:** Update `platforms: [central]` → `platforms: [central, aos8]`. This is the only frontmatter change expected. No new tool names should be added to `tools:` by Phase 13's edits (exec summary and output rules do not introduce new tool calls).

- **D-11:** Run `pytest tests/unit/test_skill_tool_references.py -k aos-migration-readiness` to confirm QUALITY-03. The test uses the `_TOOL_REF_PATTERN` regex (already extended for `aos8` in Phase 10 Plan 01) and the catalog builder that includes the AOS8 platform registry.

### Claude's Discretion

- Exact wording of the executive summary paragraph template (the skill should instruct the AI on what to produce; the planner writes the instruction, not the actual summary)
- Whether the "Output hygiene rules" subsection appears before or after the existing format template block
- Exact phrasing of the PARTIAL live-mode row in the decision matrix

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Skill file being modified
- `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` — Read the full file. Key sections for Phase 13:
  - Lines ~533–660 ("Output formatting" section and existing report template) — exec summary paragraph goes before line ~537; output hygiene rules go into the "Output formatting" section
  - Lines ~500–531 (Stage 6 decision matrix) — new PARTIAL live-mode row is added here
  - Lines 1–27 (frontmatter) — `platforms:` update from `[central]` to `[central, aos8]`

### Quality gate test
- `hpe-networking-mcp/tests/unit/test_skill_tool_references.py` — The existing regression test. Confirm it passes after skill edits. The `_TOOL_REF_PATTERN` already includes `aos8`; the `_build_full_catalog()` function already loads the `aos8` platform registry.

### Prior phase contexts (patterns to preserve)
- `.planning/phases/10-live-detection-collection/10-CONTEXT.md` — D-07..D-09 define the partial failure / hybrid fallback pattern for Stage 1. Phase 13's PARTIAL decision matrix row must be consistent with this.
- `.planning/phases/12-central-enrichment-cutover-validation/12-CONTEXT.md` — D-01..D-14 define finding format and source attribution style. Output hygiene rules must be consistent with (not override) these established patterns.

### Requirements
- `.planning/REQUIREMENTS.md` §OUTPUT-01, §OUTPUT-02, §QUALITY-01..03 — authoritative acceptance criteria for Phase 13.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Existing `**Verdict:** GO / BLOCKED / PARTIAL` header field (line ~544 in skill) — stays as-is in the header block; executive summary paragraph precedes the entire header block, it does not replace this field.
- Established finding format: `**SEVERITY — Description. (source: tool_name(), Batch N) (VSG §anchor)**` — output hygiene rules reinforce this, not replace it.
- Stage 6 decision matrix (lines ~500–531) — add one new PARTIAL row at line ~505 area (after the existing PARTIAL row for missing paste bundle).

### Established Patterns
- Source attribution always in parentheses at end of finding: `(source: aos8_get_cluster_state(), Batch 3)`
- Finding severity labels: **REGRESSION**, **DRIFT**, **INFO** — always bold, all-caps
- Partial failure fallback: "If [Batch N] was unavailable: [exact CLI command] required"
- One finding per conflicting item (established in Phase 11 RULES-04, Phase 12 ENRICH-03/04)

### Integration Points
- Exec summary paragraph precedes the `## AOS migration readiness —` header line in the Stage 6 output template
- Output hygiene rules section added to `## Output formatting` — does not change Stage 1–5 sub-path prose
- Decision matrix PARTIAL row added to the table in Stage 6 (consistent with the existing paste-PARTIAL row)
- `platforms:` frontmatter update is a single-field change at line 23 of the skill

</code_context>

<specifics>
## Specific Ideas

- Executive summary template the skill should instruct the AI to produce:
  ```
  **[VERDICT]** — [X] REGRESSION / [Y] DRIFT / [Z] INFO findings. This [source platform] deployment ([N] APs, [M] controllers) [one plain-English action sentence]. [Optional: "Live AOS8 data collection was used for all checks." or PARTIAL note if applicable.]
  ```
- Output hygiene rules subsection title: "Output hygiene (mandatory)"
- PARTIAL live-mode sentence: "AOS8 live collection partially succeeded — [N] checks completed; [M] checks require manual CLI paste (listed below)."
- `platforms:` change: `platforms: [central]` → `platforms: [central, aos8]`

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 13-executive-output-quality-gate*
*Context gathered: 2026-04-29*
