# Phase 13: Executive Output & Quality Gate - Research

**Researched:** 2026-04-29
**Domain:** Markdown skill authoring + frontmatter audit + pytest regression gate
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Executive Summary Placement (OUTPUT-01)**
- **D-01:** Executive summary is a freestanding paragraph at the very top of the report, BEFORE the `## AOS migration readiness —` header block. First thing the reader sees — the SE can copy it into a customer email without scrolling past structured data.

**Executive Summary Content (OUTPUT-01)**
- **D-02:** Paragraph contains three elements, always in this order: (1) Verdict — GO/BLOCKED/PARTIAL with capitalised label, (2) Finding counts — exact counts of REGRESSION/DRIFT/INFO, (3) SE-ready sentence naming source platform, AP count, controller count, key action.
- **D-03:** Paragraph is always exactly 2–4 sentences. Never a bullet list. Never a table. Plain prose only.
- **D-04:** SE-ready sentence includes: source platform (AOS8/AOS6/IAP), AP count from live or parsed inventory, controller count (from MD hierarchy or paste), and a plain-language action summary. For PARTIAL verdict, sentence notes which data is still missing.

**Clean Output Guardrails (OUTPUT-02)**
- **D-05:** Add an "Output hygiene rules" subsection under "Output formatting" with 4 explicit prohibitions:
  1. No raw JSON blobs — extract and cite only the specific field value; never dump the full response dict.
  2. No tool-call syntax in finding text — tool names belong only in the `(source: ...)` attribution, never in the finding sentence itself.
  3. No stack traces — emit only a brief error note and the fallback CLI command.
  4. No ellipsis / truncation markers — extract relevant fields explicitly; if response is large, summarise in prose.
- **D-06:** These rules apply to all stages of the report — Stages 3, 4, 5, 6. Stated once in "Output formatting" (not repeated per-stage).

**PARTIAL Verdict Extension (Decision Matrix)**
- **D-07:** Add a new row to Stage 6 decision matrix: `AOS8 live mode AND one or more Stage 1 batches failed` → **PARTIAL** — list which AOS8 checks could not run, plus exact CLI commands to complete those data points manually. All checks that did succeed are still reported.
- **D-08:** PARTIAL exec summary sentence template: *"AOS8 live collection partially succeeded — [N] checks completed; [M] checks require manual CLI paste (see below)."*

**Frontmatter Audit (QUALITY-01..03)**
- **D-09:** Plan explicitly diffs AOS8 tool names referenced in the skill body against the `tools:` frontmatter. Expected: 9 AOS8 tools added in Phase 10 (`aos8_get_md_hierarchy`, `aos8_get_effective_config`, `aos8_get_ap_database`, `aos8_get_cluster_state`, `aos8_show_command`, `aos8_get_clients`, `aos8_get_bss_table`, `aos8_get_active_aps`, `aos8_get_ap_wired_ports`) already cover all references added in Phases 11 and 12.
- **D-10:** Update `platforms: [central]` → `platforms: [central, aos8]`. Only frontmatter change expected. No new tool names should be added to `tools:` by Phase 13's edits.
- **D-11:** Run `pytest tests/unit/test_skill_tool_references.py -k aos-migration-readiness` to confirm QUALITY-03. Test uses `_TOOL_REF_PATTERN` regex (already extended for `aos8` in Phase 10 Plan 01) and catalog builder that includes the AOS8 platform registry.

### Claude's Discretion

- Exact wording of the executive summary paragraph template (skill instructs the AI on what to produce; planner writes the instruction, not the actual summary).
- Whether the "Output hygiene rules" subsection appears before or after the existing format template block.
- Exact phrasing of the PARTIAL live-mode row in the decision matrix.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| OUTPUT-01 | Report includes an executive summary paragraph at top: GO/BLOCKED/PARTIAL verdict, count of REGRESSIONs/DRIFTs/INFOs, one-sentence SE-shareable context | Skill template (lines 537–544) currently opens with `## AOS migration readiness —` header. Insert freestanding paragraph BEFORE that header per D-01..D-04. |
| OUTPUT-02 | Structured findings render as clean markdown — no raw JSON, no tool call artifacts — pasteable into customer-facing doc | Add "Output hygiene rules" subsection in "Output formatting" (line ~533) with the 4 prohibitions in D-05; existing finding format (lines 583–629) already demonstrates correct style. |
| QUALITY-01 | `tools:` frontmatter includes every AOS8 tool name referenced in skill body | Frontmatter line 23 already lists 9 AOS8 tools (added Phase 10). D-09 — verify no Phase 11/12 references slipped past. Audit step only; expect zero diff. |
| QUALITY-02 | `platforms:` tag includes `aos8` alongside `central` | Frontmatter line 21: `platforms: [central]` → `platforms: [central, aos8]`. Single field change per D-10. |
| QUALITY-03 | `tests/unit/test_skill_tool_references.py` passes with all new AOS8 tool name references resolving to real catalog entries | Test already exists at `hpe-networking-mcp/tests/unit/test_skill_tool_references.py`; `_TOOL_REF_PATTERN` (line 39) includes `aos8`; `_build_full_catalog` (line 82) imports `aos8` platform registry. Test currently passes — verify it still passes after edits. |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

Phase 13 makes no Python code changes — most CLAUDE.md directives don't apply. The constraints that DO apply:

- **GSD workflow:** All edits must go through GSD commands (this research → planning → execution). Markdown edits to bundled skills are repo edits.
- **File size limit (500 lines):** The skill file is ~669 lines. Adding the exec summary paragraph instruction + output hygiene subsection + one matrix row will push it further over. The 500-line rule explicitly says *"if approaching this limit, provide why the limit should be exceeded or refactor by splitting into modules."* Skill files are markdown runbooks, not code modules — splitting the skill is a v1.2-scope decision, not Phase 13. **Recommendation:** acknowledge the file is over the soft limit, document why (single canonical runbook by design), and do not refactor in this phase.
- **Branch strategy:** `main` branch protected; PR required. Phase 13 deliverable is a PR (or set of commits on a feature branch) that the user merges.
- **Pre-push checklist:** Run `ruff check`, `ruff format --check`, `mypy`, `pytest tests/` in Docker before pushing. Even though this is a markdown-only phase, the regression test (`test_skill_tool_references.py`) is the primary CI gate.
- **Documentation checklist:** Skill content changes — verify whether `docs/TOOLS.md`, `INSTRUCTIONS.md`, or `CHANGELOG.md` need entries (not in current scope per CONTEXT.md, but flag for verifier).
- **Commit message format:** Never include "claude code" or "written by claude code" in messages.

## Summary

Phase 13 is a **markdown-only skill edit phase** with one Python regression test as the quality gate. The skill file (`hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md`) gains:

1. An exec summary paragraph instruction at the very top of the Stage 6 output template (before line 538's `## AOS migration readiness —` header).
2. An "Output hygiene rules" subsection inserted into the "Output formatting" section (around line 533) prohibiting raw JSON, tool-call syntax in finding text, stack traces, and ellipsis markers.
3. One new PARTIAL row in the Stage 6 decision matrix (around lines 500–531) covering AOS8 live-mode batch failures.
4. One frontmatter field update: `platforms: [central]` → `platforms: [central, aos8]` (line 21).

There is **no new Python**. The frontmatter `tools:` list already contains all 9 AOS8 tools (added in Phase 10). The regression test (`tests/unit/test_skill_tool_references.py`) was already extended for `aos8` in Phase 10 — it should pass unchanged after Phase 13 edits, confirming QUALITY-03.

**Primary recommendation:** Treat this as a precision documentation-edit phase. Read the skill, draft the exec summary paragraph instruction (skill instructs the AI; planner writes the instruction, not the summary itself), draft the output hygiene rules block, draft the new decision-matrix row, change the `platforms:` line, then run the regression test and the full unit suite. Use a single feature branch per CLAUDE.md branch strategy.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pytest | >= 8.4.0 | Existing regression test runner | Already configured in `pyproject.toml`; QUALITY-03 verification |
| (none — markdown only) | — | Skill edits | Skills are plain markdown with YAML frontmatter; no library |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| ruff | >= 0.15.6 | Lint + format check on commit | Pre-push hygiene per CLAUDE.md (markdown unaffected, but CI runs anyway) |
| mypy | >= 1.18.0 | Type check | Pre-push (no code changes — runs clean by default) |

### Alternatives Considered

None. The phase has no library decisions to make — the test infrastructure already exists, the skill format is fixed, and there is no Python code.

**Installation:** No new packages.

## Architecture Patterns

### Skill File Anatomy
```
hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md
├── Lines 1–24    YAML frontmatter (name, title, description, platforms, tags, tools)
├── Line 26       Title H1
├── Lines 28–48   Objective + scope sections
├── ...           Stages 0–5 instructions
├── Lines 500–531 Stage 6 decision matrix table
├── Line 533      "## Output formatting" heading
├── Lines 535–536 Format directive prose
├── Lines 537–661 Code-fenced report template (the canonical structure)
└── Lines 663+    Trigger query examples
```

### Pattern 1: Skill Instructs AI, Doesn't Produce Output

**What:** The skill is a runbook. It tells the AI *what to produce* and *what not to produce*. The skill file does not contain the actual exec summary; it contains the **template + rules** the AI follows when generating one.

**When to use:** Always — this is how every section of this skill works.

**Example (existing pattern, lines 537–544):**
```markdown
\`\`\`
## AOS migration readiness — <source: aos6/aos8/iap> → AOS 10 <target: tunnel/bridge/mixed>
**Captured:** <ISO timestamp>
**Verdict:** GO / BLOCKED / PARTIAL
\`\`\`
```
The angle-bracket placeholders are AI fill-ins. The exec summary instruction follows the same convention.

### Pattern 2: Decision Matrix Row Format

**What:** Two-column markdown table — `| Condition | Action |`. Action begins with the verdict label (REGRESSION / DRIFT / INFO / PARTIAL / BLOCKED / GO) in bold, then prose, then VSG anchor parenthetical when applicable.

**When to use:** Adding the new D-07 PARTIAL row.

**Example (existing PARTIAL row, line 505):**
```markdown
| Operator hasn't pasted the data bundle | Output **PARTIAL** verdict — Central-side checks complete, source-side blocked. List exactly what's needed. |
```

### Pattern 3: Output Hygiene Rules — Sub-section under Output formatting

**What:** Markdown subsection with a `###` heading, prose intro, ordered/numbered list of prohibitions, each prohibition stating the rule and the rationale in one or two sentences.

**When to use:** D-05 — single insertion in the "Output formatting" section. Per D-06, applies globally (do not duplicate per-stage).

### Anti-Patterns to Avoid

- **Editing finding text in Stages 3/4/5:** D-06 explicitly says output hygiene rules go in "Output formatting" *once*, not per-stage. The existing finding format template (lines 583–629) already demonstrates correct style — adding rules per-stage creates redundant maintenance burden.
- **Adding new tool references in exec summary or hygiene rules:** D-10 forbids it. Phase 13 introduces zero new tool names. If the planner finds itself adding an AOS8 tool name not already in line 23's frontmatter, stop and reassess — Phase 11/12 should have already added it.
- **Refactoring the skill to fit 500-line cap:** Out of scope for v1.1. The skill is a single canonical runbook; fragmenting it is a v1.2 architectural change.
- **Running full Docker pre-push for a markdown-only PR:** CLAUDE.md says run lint+format+mypy+pytest, but markdown changes only meaningfully exercise pytest. Run `pytest tests/unit/` directly; ruff and mypy will pass trivially.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| "Verify every AOS8 tool referenced in skill body exists" | Custom grep + tool-name parser | `tests/unit/test_skill_tool_references.py` | Already exists; already extended for `aos8` (Phase 10); already imports the catalog from `REGISTRIES`. Just run it. |
| "Resolve cross-platform tool catalog" | Walk `platforms/*/tools/*.py` | `_build_full_catalog()` in the regression test | Handles import-cache reload pitfalls, reads `REGISTRIES`, includes meta-tools and cross-platform tools |
| "Diff `tools:` frontmatter against body references" | Manual cross-check spreadsheet | The same regression test — it parses the body and checks against the catalog (which is built from real platform registries, not the frontmatter). The frontmatter `tools:` list is informational metadata, not the source of truth — the catalog is. | The test catches missing tools regardless of frontmatter. The frontmatter audit (D-09) is therefore purely about *advertised* tools matching *referenced* tools, for human/IDE clarity, not test correctness. |

**Key insight:** The regression test verifies referenced-tools-exist (catalog-side correctness). The frontmatter `tools:` list is human/agent metadata — there is no automated check that frontmatter matches body. D-09's audit is a manual diff. Plan should make this explicit.

## Common Pitfalls

### Pitfall 1: Confusing "skill instructs AI" with "skill produces output"

**What goes wrong:** Author writes a literal exec summary into the skill file (e.g. `**GO** — 0 REGRESSION / 2 DRIFT / 5 INFO findings...`) instead of the *instruction* for the AI to produce one.

**Why it happens:** The instruction template uses concrete-looking placeholders, which look like real output.

**How to avoid:** Use angle-bracket placeholders (`<verdict>`, `<N>`, `<M>`) consistent with the rest of the template. Add a one-line note above the template block: *"AI fills in the placeholders at runtime; the structure below is the contract."*

**Warning signs:** Hard-coded numbers, hard-coded platform names, hard-coded verdict labels in the template body.

### Pitfall 2: Putting hygiene rules per-stage instead of in Output formatting

**What goes wrong:** Author copies the 4 prohibitions into Stage 3, 4, 5 sections "to be safe."

**Why it happens:** Wanting to make the rules unmissable.

**How to avoid:** Per D-06, single insertion under "Output formatting." The existing finding-format template already exemplifies the style; the rules just call out edge cases explicitly.

### Pitfall 3: Adding `aos8_*` tool names that don't exist to satisfy a finding example

**What goes wrong:** Author writes an exec summary example like *"...source: aos8_get_migration_status()..."* — but no such tool exists.

**Why it happens:** Drafting from imagination rather than the live catalog.

**How to avoid:** The 9 AOS8 tools available are exhaustively listed in D-09 above. Any tool name written into the skill body MUST be one of those 9 (plus the existing Central, ClearPass, GreenLake tools already in frontmatter). The regression test will catch violations.

**Warning signs:** Test failure with output like `Skill 'aos-migration-readiness.md' references non-existent tools: ['aos8_foo_bar']`.

### Pitfall 4: Forgetting `platforms: [central, aos8]` is a list, not a comma-string

**What goes wrong:** YAML frontmatter parser misreads `platforms: central, aos8` (no brackets) as a single string.

**Why it happens:** YAML accepts both flow-style (`[a, b]`) and comma-strings, but the rest of the skill ecosystem expects flow-style lists.

**How to avoid:** Match the existing `tags:` line format exactly: `platforms: [central, aos8]` with brackets. Verify by spot-checking another skill if available.

### Pitfall 5: Decision matrix table column alignment breaks rendering

**What goes wrong:** Inserting a new row with mismatched pipe count breaks the markdown table.

**Why it happens:** Long action prose wraps the row visually in the editor; a missing trailing `|` is easy to miss.

**How to avoid:** Always 3 pipes per row (start, middle, end). After insertion, render the markdown locally to confirm the table is intact.

## Runtime State Inventory

This is a markdown-only documentation phase with no rename, refactor, or migration component. No runtime state is in scope.

- **Stored data:** None — no databases or datastores hold the executive summary or output hygiene rules.
- **Live service config:** None — no n8n / Datadog / Cloudflare config refers to this skill.
- **OS-registered state:** None — skills are loaded at MCP server startup from the `skills/` directory; no OS-level registration.
- **Secrets/env vars:** None — no secrets reference the skill content. The skill itself references secret names (e.g. `aos8_*` tool names that read from `secrets/aos8_*` files), but those references already exist and are not modified by Phase 13.
- **Build artifacts:** None — markdown skills are not compiled. The Docker image bundles the skills directory at build time; rebuilding the image after the merge is the standard release path (deferred to packaging, not this phase).

**Verified by:** reading the skill engine implementation pattern (`src/hpe_networking_mcp/skills/_engine.py` per architecture notes — loads `*.md` at startup, exposes via `skills_list` / `skills_load` tools) and confirming no other component caches skill content.

## Code Examples

### Example 1: Frontmatter `platforms:` update (single-line diff)

**Source:** Skill file line 21 (read at this session)

Before:
```yaml
platforms: [central]
```

After:
```yaml
platforms: [central, aos8]
```

### Example 2: New decision-matrix row (D-07)

Insertion point: after existing PARTIAL row at skill line 505.

```markdown
| AOS8 live mode AND one or more Stage 1 batches failed | **PARTIAL** — list which AOS8 checks could not run (e.g. "Batch 3 failed: cluster state, local-user db"). For each failed batch, include the exact CLI command(s) operator must paste manually to complete those data points. All checks that did succeed are still reported. |
```

### Example 3: Exec summary paragraph instruction template (D-02..D-04, D-08)

Insertion point: at the very top of the report template — at skill line 537, BEFORE the existing `## AOS migration readiness —` header.

The instruction tells the AI to produce a 2–4 sentence prose paragraph, never a list. Suggested skeleton (planner refines exact wording):

```markdown
Open the report with this paragraph (plain prose, 2–4 sentences, never a bullet list, never a table). Three elements in order:

1. Verdict in bold caps — **GO** / **BLOCKED** / **PARTIAL**.
2. Finding counts — exact integers for REGRESSION, DRIFT, INFO.
3. One SE-ready sentence — name the source platform (AOS8 / AOS6 / IAP), AP count, controller count, and the key action a human SE can paste verbatim into a customer email.

For PARTIAL verdict caused by AOS8 live-mode batch failure, append: "AOS8 live collection partially succeeded — <N> checks completed; <M> checks require manual CLI paste (see below)."

Template:
**<VERDICT>** — <X> REGRESSION / <Y> DRIFT / <Z> INFO findings. This <source> deployment (<N> APs, <M> controllers) <one plain-English action sentence>. <Optional context sentence — e.g. "Live AOS8 data collection was used for all checks." or PARTIAL note if applicable.>
```

### Example 4: Output hygiene rules subsection (D-05)

Insertion point: under `## Output formatting` heading at skill line 533, before the `\`\`\`` code-fence on line 537.

```markdown
### Output hygiene (mandatory)

These rules apply to every finding across Stages 3, 4, 5, and 6:

1. **No raw JSON blobs.** When citing an API response, extract and quote only the specific field value relevant to the finding. Never dump the full response dict.
2. **No tool-call syntax in finding text.** Tool names belong in the `(source: tool_name(), Batch N)` attribution at the end of the finding — never inside the finding sentence itself.
3. **No stack traces.** If a tool call raises an exception, emit a brief one-line error note and the fallback CLI command. Never include a Python traceback.
4. **No ellipsis or truncation markers.** Do not write `...`, `[truncated]`, or similar. Extract the relevant fields explicitly; if a response is large, summarise the salient values in prose.
```

### Example 5: Regression test invocation (QUALITY-03)

```bash
# From hpe-networking-mcp/ working dir:
uv run pytest tests/unit/test_skill_tool_references.py -k aos-migration-readiness -v
# Or full suite to confirm no collateral damage:
uv run pytest tests/unit/ -q
```

Expected output: 1 passed (the `aos-migration-readiness.md` parametrized case under `TestSkillToolReferences::test_skill_references_resolve`).

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manually grep skill body for tool names; cross-check by eye | Automated regex + catalog regression test (`test_skill_tool_references.py`) | v2.3.0.0 / Phase 10 (v1.1) | Phase 13 just runs the existing test |
| Per-stage finding-format reminders | Single canonical "Output formatting" section + format template | Existing pattern (pre-v1.1) | D-06 reuses this — single hygiene-rules block |
| `platforms: [central]` (single-platform skill) | `platforms: [central, aos8]` (multi-platform) | This phase | Surfaces the skill to AOS8-aware skill discovery |

**Deprecated/outdated:** None applicable.

## Open Questions

1. **Should the planner introduce a separate "exec summary template" code-fence block, or fold it into the existing report template at lines 537–661?**
   - What we know: D-01 says exec summary paragraph precedes the entire `## AOS migration readiness` header block. Two implementation options: (a) add a second code-fence block above the existing one, or (b) prepend the exec summary instruction lines inside the existing fenced block, before the `## AOS migration readiness — ...` line.
   - What's unclear: which is more readable for the AI runtime.
   - Recommendation: Single fenced block, prepend exec summary at the top — matches how the AI parses the template top-down, no risk of the AI emitting only the summary and dropping the structured report.

2. **Does the existing PARTIAL row at line 505 conflict with the new D-07 PARTIAL row?**
   - What we know: Line 505 covers "operator hasn't pasted the data bundle" — paste-mode partial. D-07 covers "AOS8 live mode AND batches failed" — live-mode partial. Different conditions, both legitimately PARTIAL.
   - What's unclear: nothing — they're orthogonal.
   - Recommendation: Insert D-07 row immediately after the line 505 row to keep PARTIAL conditions adjacent.

3. **Does CHANGELOG.md need an entry per CLAUDE.md docs checklist?**
   - What we know: CLAUDE.md says every code PR updates CHANGELOG.md, README, INSTRUCTIONS.md, docs/TOOLS.md, pyproject.toml. Phase 13 is markdown-only — does that count as "code"?
   - What's unclear: Skill content is shipped with the package; arguably user-facing.
   - Recommendation: Plan should include an explicit task to update CHANGELOG.md (one-line entry: "Skill `aos-migration-readiness`: added executive summary paragraph, output hygiene rules, AOS8 live-mode PARTIAL verdict, `platforms: aos8`"). No version bump needed if Phase 13 batches with the rest of v1.1; one bump at v1.1.0 release covers all 4 phases.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python 3.12 | Running pytest regression test | ✓ (assumed — CLAUDE.md mandates) | 3.12+ | — |
| pytest | QUALITY-03 verification | ✓ — installed via `uv sync` | >= 8.4.0 | — |
| `uv` | Running tests locally without Docker | ✓ (assumed) | latest | `pip install -e .` + raw pytest |
| Docker + Compose | Pre-push checklist | ✓ (assumed for CI parity) | Compose v2.24+ | Run pytest directly |
| Live AOS8 environment | Validating the exec summary actually renders correctly end-to-end | ✗ | — | Same as Phase 10/12 — Scenario A "live verification" deferred; the markdown changes are mechanically correct and the regression test gates the structural correctness |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:** Live AOS8 environment — fallback is the same partial-approval pattern used in Phase 10/12 (skill is mechanically correct + test passes; live runtime validation deferred until an AOS8 lab is available).

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest >= 8.4.0 |
| Config file | `hpe-networking-mcp/pyproject.toml` (`[tool.pytest.ini_options]`, `asyncio_mode = "auto"`) |
| Quick run command | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -k aos-migration-readiness -v` |
| Full suite command | `cd hpe-networking-mcp && uv run pytest tests/unit/ -q` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| OUTPUT-01 | Exec summary paragraph appears at top of Stage 6 output template, in correct position before the `## AOS migration readiness` header | manual-only | (visual review of skill diff; runtime AI behavior cannot be unit-tested without live model invocation) | n/a — skill content review |
| OUTPUT-02 | Output hygiene rules subsection present under "Output formatting" with all 4 prohibitions | manual-only | (visual review of skill diff) | n/a — skill content review |
| OUTPUT-02 (lighter) | Skill body still parses as valid YAML frontmatter + markdown | smoke | `python -c "import yaml; from pathlib import Path; t = Path('src/hpe_networking_mcp/skills/aos-migration-readiness.md').read_text(encoding='utf-8'); fm = t.split('---', 2)[1]; yaml.safe_load(fm)"` | ✓ (one-liner; no dedicated test file) |
| QUALITY-01 | Every `aos8_*` token in skill body resolves to a real tool in the catalog | unit | `pytest tests/unit/test_skill_tool_references.py::TestSkillToolReferences::test_skill_references_resolve -k aos-migration-readiness -v` | ✓ |
| QUALITY-02 | `platforms:` frontmatter contains `aos8` | unit (light) | `python -c "import yaml; t = open('src/hpe_networking_mcp/skills/aos-migration-readiness.md', encoding='utf-8').read(); fm = yaml.safe_load(t.split('---', 2)[1]); assert 'aos8' in fm['platforms'], fm['platforms']"` | ✓ (no dedicated test; one-liner is sufficient) |
| QUALITY-03 | Regression test passes against the live tool catalog | unit | `pytest tests/unit/test_skill_tool_references.py -v` | ✓ |

**Manual review gate:** OUTPUT-01 and OUTPUT-02 are content-quality requirements that pytest cannot verify (the test only checks that referenced tool names exist, not that the skill produces good prose at runtime). The verifier agent should confirm by reading the skill diff that:

- Exec summary paragraph instruction is at line ~537 (before the `## AOS migration readiness —` header inside the report-template code fence).
- Output hygiene rules subsection has all 4 prohibitions per D-05.
- New PARTIAL row added per D-07.
- `platforms: [central, aos8]` is present.

### Sampling Rate
- **Per task commit:** `pytest tests/unit/test_skill_tool_references.py -v` (~ 5 s)
- **Per wave merge:** `pytest tests/unit/ -q` (~ 60–90 s for 790 tests)
- **Phase gate:** Full suite green before `/gsd:verify-work`. Pre-push checklist additionally runs ruff + mypy (will pass trivially — no Python changes).

### Wave 0 Gaps

None — existing test infrastructure covers all phase requirements. The regression test, the catalog builder, and the `_TOOL_REF_PATTERN` regex were all completed in Phase 10. No new test files are needed.

*(If the planner wants to add a YAML-frontmatter validity assertion as a defensive smoke test, that is optional polish, not a Wave 0 gap.)*

## Sources

### Primary (HIGH confidence)
- `.planning/phases/13-executive-output-quality-gate/13-CONTEXT.md` — locked decisions D-01..D-11
- `.planning/REQUIREMENTS.md` §OUTPUT-01..02, §QUALITY-01..03 — authoritative acceptance criteria
- `.planning/STATE.md` — phase status, prior-phase context
- `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md` (lines 1–24, 490–661) — skill structure, current decision matrix, current report template, current frontmatter
- `hpe-networking-mcp/tests/unit/test_skill_tool_references.py` — regression test that gates QUALITY-03; confirms `_TOOL_REF_PATTERN` already includes `aos8` (line 39) and `_build_full_catalog()` already imports `aos8` registry (line 104)
- `CLAUDE.md` (project root) — file size, branch, commit, docs-checklist constraints
- `hpe-networking-mcp/CLAUDE.md` — Python style, pre-push checklist, branch protection rules

### Secondary (MEDIUM confidence)
- `.planning/phases/12-central-enrichment-cutover-validation/12-CONTEXT.md` (referenced by D-09 and CONTEXT canonical_refs) — finding format and source attribution patterns Phase 13 reinforces
- `.planning/phases/10-live-detection-collection/10-CONTEXT.md` (referenced by D-07 canonical_refs) — partial-failure pattern for Stage 1 that the new PARTIAL matrix row must align with

### Tertiary (LOW confidence)
None.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — no new libraries; existing pytest + ruff + mypy + uv
- Architecture: HIGH — skill structure inspected directly; insertion points verified at specific line numbers
- Pitfalls: HIGH — risks are inherent to YAML/markdown editing and have known mitigations; the test will catch tool-name pitfalls automatically
- Validation: HIGH — regression test exists and was extended in Phase 10; QUALITY-03 is a one-command gate

**Research date:** 2026-04-29
**Valid until:** 2026-05-29 (30 days; stable — no fast-moving dependencies)
