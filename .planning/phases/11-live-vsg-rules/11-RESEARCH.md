# Phase 11: Live VSG Rules - Research

**Researched:** 2026-04-29
**Domain:** Markdown skill authoring (FastMCP skills) — VSG-anchored rule narration over live AOS8 data
**Confidence:** HIGH

## Summary

Phase 11 is a **markdown-only** edit to `hpe-networking-mcp/src/hpe_networking_mcp/skills/aos-migration-readiness.md`. No Python code, no new tools, no library decisions. Phase 10 already wired Stage -1 detection and Stage 1's four-batch live-mode collection into the skill; Phase 11 layers an "AOS8 live-mode sub-path" at the top of Stage 3 that evaluates RULES-01, RULES-02, RULES-04 from data already in context, defers RULES-03 to Stage 4 A11, and inserts a Stage 2 skip clause so the AI doesn't wait for paste when live mode is active.

All four rules are restatements of existing paste-mode rules (C2 → RULES-01, C4 → RULES-02, U2 → RULES-04) with the same severities and VSG anchors — no new rule logic is being invented. The risk surface is entirely about **field-level prose precision**: the skill must name the exact JSON field per rule so the AI doesn't hallucinate a structure.

**Primary recommendation:** Mirror the Stage 1 sub-path pattern verbatim — same conditional header style, same fallback footer, same "(source: tool_name(args), Batch N)" citation format. Edit in three localized blocks (Stage 2 skip clause, Stage 3 live-mode sub-path, Stage 4 A11 expansion) and run `pytest tests/unit/test_skill_tool_references.py` after each batch.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Stage 3 Structure (RULES-01, RULES-02, RULES-04)**
- **D-01:** Stage 3 mirrors Stage 1's live/paste sub-path pattern. Add an explicit **"AOS8 live-mode sub-path"** section within Stage 3's AOS8-specific rules block — parallel to the paste-driven evaluation path that already exists implicitly (Stage 2 parsed data).
- **D-02:** The live-mode sub-path runs **before** the existing universal + AOS6/8 rule tables. It evaluates RULES-01, RULES-02, and RULES-04 from Stage 1 context data (no re-fetching). After the live-mode block, the universal rules (U1, U3, U5, etc.) continue as normal.

**Stage 2 Live Bypass**
- **D-03:** Add a **skip clause at the top of Stage 2** for AOS8 live mode: *"If AOS8 live mode (Stage -1 announced API mode): Stage 1 data is already available — skip paste parsing for AOS8 data points and proceed directly to Stage 3."*

**RULES-03 ClearPass Timing**
- **D-04:** RULES-03 **defers to Stage 4 A11**. Stage 3's live-mode sub-path notes: *"RULES-03 result pending Stage 4 check A11"*. Stage 4 A11 (`clearpass_get_local_users()`) gets expanded to perform the cross-check using the AOS8 local-user count already in context from Stage 1 Batch 3. No duplicate ClearPass call.

**Rule Parsing Guidance (Field-Level)**
- **D-05:** Each rule in the live-mode sub-path specifies the **exact config object field** to check:
  - **RULES-01 (VRRP VIP)**: In the `ap_sys_prof` effective config response from Batch 1, check the LMS IP field — flag REGRESSION if it matches a controller management IP rather than the VRRP virtual IP (VSG §1654-§1657). Each finding cites which AP system profile name contains the wrong IP.
  - **RULES-02 (ARM/radio profiles)**: In the Batch 1 effective config results for `arm_profile`, `dot11a_radio_prof`, `dot11g_radio_prof`, `reg_domain_profile` — detect presence (any non-empty response) and enumerate which profiles are active per AP-group. Flag as DRIFT (VSG §1163-§1166).
  - **RULES-04 (static AP IPs)**: In the `aos8_get_ap_database()` response from Batch 2, check the `ip_mode` field per AP — flag REGRESSION for every AP where `ip_mode` is not DHCP (VSG §1232-§1234). List affected AP names/MACs.

### Claude's Discretion

- Exact wording of the "AOS8 live-mode sub-path" header and intro line in Stage 3
- How to format per-rule findings inline vs in the final rule tables (annotate the C2/C4/U2 rows or produce a separate list)
- Whether to add a "fallback note" to each RULES-0X if the relevant Stage 1 batch failed (could reuse D-07/D-08 from Phase 10 — ask operator for the specific CLI command only)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| RULES-01 | VRRP VIP check — AP system profiles use VRRP VIP for LMS, not individual controller IPs (REGRESSION, VSG §1654-§1657) | Batch 1 already collects `aos8_get_effective_config(object_name="ap_sys_prof", config_path="/md")` per Phase 10 D-06; LMS IP field is in the returned object. Equivalent paste-mode rule is C2 (line 247). |
| RULES-02 | ARM / radio profile detection across AP-groups (DRIFT, VSG §1163-§1166) | Batch 1 already collects `arm_profile`, `dot11a_radio_prof`, `dot11g_radio_prof`, `reg_domain_profile` via `aos8_get_effective_config()` (Phase 10 skill prose, lines 99-100). Equivalent paste-mode rule is C4 (line 249). |
| RULES-03 | AOS8 local-user db count vs ClearPass count cross-check (DRIFT) | Batch 3 collects `aos8_show_command(command="show local-user db")` (line 109). Stage 4 A11 currently calls `clearpass_get_local_users()` (line 328). D-04 defers cross-check to A11 expansion. |
| RULES-04 | Static AP IP detection (REGRESSION, VSG §1232-§1234) | Batch 2 already collects `aos8_get_ap_database()` with per-AP `ip_mode` field (line 104, "Stage 3 RULES-04 reuses `ip_mode` directly"). Equivalent paste-mode rule is U2 (line 231). |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Markdown-only**: REQUIREMENTS.md "Out of Scope" — no Python code changes for v1.1
- **No new tools**: All AOS8 tools used in Phase 11 prose (`aos8_get_effective_config`, `aos8_get_ap_database`, `aos8_show_command`, `clearpass_get_local_users`) already exist in the catalog from v1.0 / Phase 10 frontmatter — no `tools:` frontmatter additions required for Phase 11
- **Skill regression test gates**: `tests/unit/test_skill_tool_references.py` validates every tool name in the skill body against the catalog and the `tools:` frontmatter. Tool name typos in the new prose will fail CI. The Phase 10 work extended `_TOOL_REF_PATTERN` regex with `|aos8` so `aos8_*` typos fail loudly.
- **File size limits**: 500 lines/file max for Python — does not strictly apply to markdown skill files, but the skill is ~530 lines today and Phase 11 will add ~50-80 more. No structural concern.
- **Pre-push CI**: Run inside Docker — `uv run ruff check . && uv run ruff format --check . && uv run mypy src/ --ignore-missing-imports && uv run pytest tests/ -q`. Markdown changes only invoke pytest's skill-reference test, but Docker run is still required per CLAUDE.md pre-push checklist.
- **Commit message format**: Never include "claude code" or "written by claude code" in commit messages (project-level CLAUDE.md, not just root)

## Standard Stack

Phase 11 introduces no new technology. The "stack" is:

| Item | Version | Purpose | Source |
|------|---------|---------|--------|
| Markdown skill format | n/a | FastMCP skill discovery + body | `src/hpe_networking_mcp/skills/_engine.py` already loads the skill at startup |
| YAML frontmatter | n/a | Tool list + platforms tag | Already present, no Phase 11 changes needed |
| Existing AOS8 tools | catalog (v1.0+) | Data sources referenced by name in prose | `differentiators.py`, `clients.py`, `wlan.py`, `troubleshooting.py` |
| `clearpass_get_local_users` | catalog | Stage 4 A11 cross-check | Already in skill `tools:` frontmatter |
| `pytest tests/unit/test_skill_tool_references.py` | pytest >= 8.4.0 | Validates skill tool refs | Phase 10 already pinned this test as the regression gate |

**Installation:** None — markdown-only edits to an existing file.

**Version verification:** N/A — no library updates.

## Architecture Patterns

### Recommended Edit Plan

The skill file is one document; Phase 11 makes three localized inserts/edits:

```
aos-migration-readiness.md
├── Stage -1 (DETECT-01)              # Phase 10 — UNCHANGED
├── Stage 0 (interview)               # UNCHANGED
├── Stage 1 (live-mode + paste-fallback sub-paths)  # Phase 10 — UNCHANGED
├── Stage 2 (parse paste bundle)      # Phase 11 EDIT 1: insert skip clause at top
├── Stage 3 (rule application)
│   ├── [NEW] AOS8 live-mode sub-path # Phase 11 EDIT 2: insert before Universal rules
│   ├── Universal rules (U1..U11)     # UNCHANGED
│   ├── AOS6/8 rules (C1..C10)        # UNCHANGED (C2/C4 may get cross-ref annotation per discretion)
│   ├── IAP rules                     # UNCHANGED
│   └── Per-target-mode rules         # UNCHANGED
├── Stage 4 (Central API checks)
│   └── A11 (ClearPass local-user)    # Phase 11 EDIT 3: expand to perform cross-check
├── Stage 5 (cutover)                 # UNCHANGED
├── Stage 6 (report)                  # UNCHANGED
├── Decision matrix                   # UNCHANGED (existing rows already cover the same conditions)
└── Output formatting                 # UNCHANGED
```

### Pattern 1: Live/Paste Sub-Path Header (mirror Stage 1)

**What:** Each sub-path uses an `#####` heading with a conditional clause naming the trigger.
**When to use:** Both Stage 2 skip clause and Stage 3 live-mode sub-path should mirror the existing Stage 1 patterns at lines 95 and 122 of the skill.
**Example (from existing skill, lines 95-96):**
```markdown
##### AOS8 live-mode sub-path — used when Stage -1 announced "AOS8 API mode"
No CLI paste required. Calls AOS8 tools directly in **four grouped batches** (COLLECT-01..04). On any tool error, fall back per-batch to an exact-CLI paste request for that batch only.
```

Phase 11 Stage 3 should use the same pattern, e.g.:
```markdown
##### AOS8 live-mode sub-path — rules evaluated from Stage 1 data (used when Stage -1 announced "AOS8 API mode")
RULES-01, RULES-02, and RULES-04 are evaluated directly from the Stage 1 batch data already in context — no re-fetching, no operator paste. RULES-03 result is pending Stage 4 A11 (ClearPass cross-check).
```

### Pattern 2: Per-Rule Finding Format

**What:** Each finding emits **Severity — Description (VSG §anchor) (source: tool_call(args), Batch N)**.
**When to use:** Every per-rule output line in the new live-mode sub-path.
**Example (synthesized from existing Stage 3 finding format + Phase 11 specifics line 103):**
```markdown
- **REGRESSION** — AP system profile `<profile_name>` LMS IP is `<controller_mgmt_ip>` (an individual controller management IP); must be the VRRP virtual IP. APs will strand on first controller upgrade. (VSG §1654-§1657) (source: `aos8_get_effective_config(object_name='ap_sys_prof', config_path='/md')`, Batch 1)
```

### Pattern 3: Per-Batch Fallback Note (reuse Phase 10 D-07/D-08)

**What:** If the upstream Stage 1 batch failed, note that the rule cannot be evaluated from live data and remind the AI to consult the paste fallback.
**When to use:** Optional per CONTEXT.md discretion — recommend including for parity with Stage 1 sub-path.
**Example:**
```markdown
*If Batch 1 failed:* RULES-01 and RULES-02 cannot be evaluated from live data. Mark as "inconclusive — paste required" or evaluate from any pasted `show running-config` / `show configuration effective detail` output already supplied via the per-batch fallback.
```

### Anti-Patterns to Avoid

- **Inventing new severities or anchors.** RULES-01/02/04 mirror existing paste-mode rules C2/C4/U2 — same severities, same VSG sections. Don't redefine.
- **Re-fetching data inside Stage 3.** All four rules consume data already collected in Stage 1 batches. Adding fresh tool calls in Stage 3 violates D-02 ("no re-fetching") and adds latency.
- **Tool-name typos in prose.** The skill regression test (`test_skill_tool_references.py`) extracts every `<word>_<word>(...)` pattern and asserts catalog membership. Misspelled tool names will fail CI immediately.
- **Removing or rewriting C2/C4/U2 rule rows.** They serve the paste-fallback path and remain authoritative there. Cross-reference annotation is the discretion choice; replacement is not.
- **Calling `clearpass_get_local_users()` twice.** D-04 explicitly says no duplicate call — Stage 3 references the value pending; Stage 4 A11 does the work.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| AOS8 LMS IP retrieval | New tool / new endpoint call | Reuse `aos8_get_effective_config(object_name='ap_sys_prof')` already in Batch 1 | Phase 10 already wires this; result is in skill context |
| ARM / radio profile enumeration | Per-AP-group iteration logic in prose | Trust the Batch 1 prose's existing iteration over `arm_profile`, `dot11a_radio_prof`, `dot11g_radio_prof`, `reg_domain_profile` | Already done in Phase 10 |
| Per-AP `ip_mode` lookup | New tool | `aos8_get_ap_database()` already returns `ip_mode` per AP — Phase 10 prose explicitly notes "Stage 3 RULES-04 reuses `ip_mode` directly" | Field is already documented |
| ClearPass local-user count fetch | Second call from Stage 3 | Defer to Stage 4 A11 per D-04; A11 already calls `clearpass_get_local_users()` | Avoids duplicate auth + API hit |
| New skill regression test | New test file | Existing `tests/unit/test_skill_tool_references.py` already validates tool name references | Phase 10 already extended its regex for `aos8` |

**Key insight:** All field-level data needed for RULES-01/02/03/04 is already in Stage 1 / Stage 4 context. Phase 11 is **prose layering**, not data engineering.

## Common Pitfalls

### Pitfall 1: Hallucinating LMS field names

**What goes wrong:** AOS8 effective-config JSON for `ap_sys_prof` may name the LMS field `lms_ip`, `lms-ip`, `controller_lms_ip`, or similar — the exact field name in the API response is not documented in CLAUDE.md or REQUIREMENTS.md.
**Why it happens:** No live AOS8 environment available (per STATE.md: Phase 10 Scenario A deferred — partial approval, no live AOS8 box).
**How to avoid:** Reference the field by intent (e.g., "LMS IP field in the `ap_sys_prof` response") rather than a specific JSON key. The AI consuming the skill at runtime can introspect the actual response structure. If a specific field name is needed, the planner should grep the AOS8 platform tests for fixtures.
**Warning signs:** Plan prose pinning a hardcoded JSON path like `response['ap_sys_prof'][0]['lms_ip']`.

### Pitfall 2: Confusing "no AOS8 live env" with "no validation possible"

**What goes wrong:** Like Phase 10 Scenario A, Phase 11 cannot be exercised end-to-end against a live AOS8 box.
**Why it happens:** No live AOS8 system per CLAUDE.md ("No live AOS8 system — all tests use mocked HTTP responses") and STATE.md.
**How to avoid:** Validate prose mechanically (skill test passes, tool refs resolve, language matches Stage 1 patterns). Document Scenario A-style deferral in the plan if appropriate.
**Warning signs:** Plan asserting "manually verified live findings produced correctly" without an AOS8 environment.

### Pitfall 3: Inserting at the wrong skill location

**What goes wrong:** The Stage 3 live-mode sub-path is inserted under the wrong header (e.g., before Universal rules at the document level rather than inside the AOS6/8-specific block).
**Why it happens:** Stage 3 has multiple rule sections (Universal → AOS6/8 → IAP → per-target-mode). D-02 says "before the existing universal + AOS6/8 rule tables" — placement matters.
**How to avoid:** Re-read CONTEXT.md D-02 carefully — the live-mode sub-path runs **first**, before any rule tables. Insert immediately after the `### Stage 3 — Apply VSG-anchored readiness rules` header, before `#### Universal rules`.
**Warning signs:** Live-mode block buried below paste-mode rules; AI processes paste-mode first and ignores live data.

### Pitfall 4: Stage 2 skip clause too aggressive

**What goes wrong:** A Stage 2 skip clause that says "skip Stage 2 entirely for AOS8 live mode" causes the AI to skip parsing AOS6/IAP pasted data in mixed-source scenarios — but mixed source is out of scope per the skill's per-source-platform conditional structure.
**Why it happens:** Over-broad wording.
**How to avoid:** Use D-03's exact phrasing — "skip paste parsing **for AOS8 data points**". This narrows the skip to only the AOS8 extraction table (lines 183-200 of the skill), leaving AOS6/IAP extraction intact for those source paths.
**Warning signs:** Skip clause uses "skip Stage 2" without qualifying which data points.

### Pitfall 5: RULES-02 false positive on absent profiles

**What goes wrong:** D-05 says "detect presence (any non-empty response)" — but `aos8_get_effective_config` returning an empty list / no objects is not the same as the profile being unused. AOS8 effective-config inheritance can produce results at lower nodes that are empty at higher ones.
**Why it happens:** The "non-empty" check has to mean "any AP-group has the profile applied", not "the API returned bytes".
**How to avoid:** RULES-02 prose should say "if any of `arm_profile`, `dot11a_radio_prof`, `dot11g_radio_prof`, `reg_domain_profile` has at least one configured object across the queried config_paths, flag DRIFT and list the profile names." Be specific that empty/absent at all paths means no DRIFT for that profile type.
**Warning signs:** Prose says "if response is non-empty" without distinguishing "envelope present" from "objects present".

## Code Examples

This phase produces no code. The artifacts produced are markdown blocks for the skill file. See **Architecture Patterns** above for skill-prose templates derived from the existing Stage 1 sub-path conventions.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Operator pastes 16 CLI commands; Stage 3 evaluates rules from Stage 2 parsed text | Stage 1 collects via API in 4 batches; Stage 3 evaluates rules from Stage 1 context | Phase 10 (Stage 1) + Phase 11 (Stage 3) | Reduces operator burden; rules cite tool calls rather than CLI line numbers |
| C2/C4/U2 paste-mode rules evaluated from running-config text | RULES-01/02/04 live-mode rules evaluated from JSON fields | Phase 11 | Field-level precision; less AI hallucination on rule application |
| ClearPass cross-check done casually inside Stage 4 A11 | A11 explicitly cross-checks against AOS8 Stage 1 Batch 3 count | Phase 11 (D-04) | Auto-flagged DRIFT instead of "operator should compare" prose |

**Deprecated/outdated:**
- None for Phase 11. Paste-mode (C2/C4/U2) rules **remain valid** for AOS6, IAP, and unreachable-AOS8 sessions.

## Runtime State Inventory

> Phase 11 is markdown-only with no rename/refactor/migration semantics. No stored data, live service config, OS-registered state, secrets, or build artifacts are touched.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | None — markdown-only edit; no databases, no caches, no IDs renamed | None |
| Live service config | None — no n8n, Datadog, or external service references touched | None |
| OS-registered state | None — no Task Scheduler, pm2, launchd registrations involved | None |
| Secrets/env vars | None — no secret keys or env var names changed | None |
| Build artifacts | None — markdown is not built; no egg-info, no compiled binaries | None |

**Verification:** Phase 10 (which had the same markdown-only profile) confirmed no runtime state via the same audit. Phase 11 is structurally identical.

## Environment Availability

> Skipped — Phase 11 has no external runtime dependencies beyond the existing Python toolchain already in use (uv, pytest, ruff, mypy). No new tools, services, runtimes, or CLI utilities are required.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `uv` | Pre-push CI | ✓ (existing) | per Dockerfile | — |
| `pytest >= 8.4.0` | Skill regression test | ✓ (existing) | pyproject.toml | — |
| Live AOS8 controller | End-to-end skill validation | ✗ | — | Mechanical-correctness review only (matches Phase 10 Scenario A precedent) |

**Missing dependencies with no fallback:** None blocking — live AOS8 absence is known-and-accepted (STATE.md / CLAUDE.md).

**Missing dependencies with fallback:** Live AOS8 environment → mechanical correctness check + partial approval pattern from Phase 10.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest 8.4.0+ with pytest-asyncio |
| Config file | `hpe-networking-mcp/pyproject.toml` (`[tool.pytest.ini_options]`) |
| Quick run command | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` |
| Full suite command | `cd hpe-networking-mcp && uv run pytest tests/unit -q` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RULES-01 | Skill prose references valid tool name `aos8_get_effective_config` for VRRP VIP rule | unit (regression) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` | ✅ |
| RULES-02 | Skill prose references valid tool name `aos8_get_effective_config` for ARM/radio profiles | unit (regression) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` | ✅ |
| RULES-03 | Skill prose references valid tool names `aos8_show_command` and `clearpass_get_local_users` | unit (regression) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` | ✅ |
| RULES-04 | Skill prose references valid tool name `aos8_get_ap_database` | unit (regression) | `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x` | ✅ |
| RULES-01..04 | Live-mode sub-path executes correctly against a real AOS8 controller | manual-only | n/a (no live AOS8 env per STATE.md) | n/a (Scenario-A-style deferral) |

**Justification for manual-only:** No live AOS8 controller is available in this project (CLAUDE.md "No live AOS8 system"; STATE.md "Scenario A deferred — no live AOS8 environment"). End-to-end execution validation is impossible. Mechanical correctness — prose structure mirrors Stage 1 sub-path patterns, all tool names resolve via the regression test, all VSG anchors match existing rule rows — is the achievable bar.

### Sampling Rate
- **Per task commit:** `cd hpe-networking-mcp && uv run pytest tests/unit/test_skill_tool_references.py -x`
- **Per wave merge:** `cd hpe-networking-mcp && uv run pytest tests/unit -q` (full unit suite — no skill-related changes should affect other modules, but verifies)
- **Phase gate:** Full suite green + Pre-push checklist (ruff check + ruff format --check + mypy + pytest) before `/gsd:verify-work`

### Wave 0 Gaps

None — `tests/unit/test_skill_tool_references.py` already exists and was extended in Phase 10 with `|aos8` regex coverage. No new test files, fixtures, or framework setup needed.

## Open Questions

1. **Exact field name of LMS IP in `ap_sys_prof` effective-config response**
   - What we know: AOS8 `ap_sys_prof` has an LMS / Backup-LMS configuration; CLI form is `lms-ip` and `bkup-lms-ip`. AOS8 API objects often map CLI dashes to underscores in JSON.
   - What's unclear: Without a live AOS8 box or saved fixture, the actual JSON key (`lms_ip` vs `lms-ip` vs nested under another envelope) is unverified.
   - Recommendation: Skill prose should reference "the LMS IP field" by intent, not by JSON key. The AI client introspects at runtime. Planner should NOT pin a specific path. If a fixture exists in `hpe-networking-mcp/tests/unit/aos8/`, the plan can grep it for confirmation; otherwise, leave intent-level.

2. **Whether to annotate C2/C4/U2 rule-table rows with live-mode cross-references**
   - What we know: Per CONTEXT.md Claude's Discretion, this is a styling choice.
   - What's unclear: Whether annotations help or clutter the rule tables.
   - Recommendation: Annotate inline (e.g., add `(See AOS8 live-mode sub-path for live evaluation)` to C2/C4 description cells). Low cost, improves traceability for any human reading the skill source.

3. **RULES-02 finding granularity — per AP-group or aggregate?**
   - What we know: D-05 says "enumerate which profiles are active per AP-group". Stage 1 Batch 1 narrates iteration over config object types but the prose isn't explicit about per-AP-group iteration.
   - What's unclear: Does Batch 1 actually iterate config_path per AP-group, or only at `/md` root?
   - Recommendation: Skill prose for Batch 1 line 100 currently shows `config_path="/md"` as a single call per object type — root-level effective config inherits down. RULES-02 finding should report at the granularity Stage 1 collected (root level by default), with prose like "ARM/radio/regulatory profiles detected at `/md` root scope: <list>". Per-AP-group enumeration would require Batch 1 changes — out of scope for Phase 11.

## Sources

### Primary (HIGH confidence)
- `C:\Dev\adams_mcp\hpe-networking-mcp\src\hpe_networking_mcp\skills\aos-migration-readiness.md` — current skill body (lines 95-120 = Stage 1 live-mode sub-path; lines 122-146 = paste-fallback; lines 183-200 = AOS8 extraction table; line 247 = C2; line 249 = C4; line 231 = U2; line 328 = A11) — authoritative source for prose patterns to mirror
- `C:\Dev\adams_mcp\.planning\phases\11-live-vsg-rules\11-CONTEXT.md` — locked decisions D-01..D-05 + canonical references
- `C:\Dev\adams_mcp\.planning\phases\10-live-detection-collection\10-CONTEXT.md` — D-04 amendment, D-06 batch structure, D-07/D-08 partial-collection policy used as fallback pattern
- `C:\Dev\adams_mcp\.planning\REQUIREMENTS.md` — RULES-01..04 acceptance criteria
- `C:\Dev\adams_mcp\hpe-networking-mcp\src\hpe_networking_mcp\platforms\aos8\tools\differentiators.py` — `aos8_get_effective_config(object_name, config_path)` signature confirmed
- `C:\Dev\adams_mcp\hpe-networking-mcp\CLAUDE.md` — file size limits, pre-push checklist, commit message constraints
- `C:\Dev\adams_mcp\CLAUDE.md` (project root) — "No live AOS8 system; all tests use mocked HTTP responses"
- `C:\Dev\adams_mcp\.planning\STATE.md` — Phase 10 Scenario A deferral precedent

### Secondary (MEDIUM confidence)
- N/A — All claims rooted in repo-local files; no web research required for a markdown-only skill edit.

### Tertiary (LOW confidence)
- LMS field JSON key name in AOS8 `ap_sys_prof` response — see Open Question 1.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — no new tech; all tools already in catalog
- Architecture: HIGH — patterns lifted verbatim from Phase 10's existing Stage 1 sub-path structure already in the skill
- Pitfalls: HIGH — drawn from Phase 10 Scenario A history (live AOS8 absence), CLAUDE.md constraints, and prose-precision risks documented in the same skill file
- Open Questions: LOW — JSON field names cannot be verified without a live AOS8 box or fixture access

**Research date:** 2026-04-29
**Valid until:** 2026-05-29 (30 days — stable scope; only invalidated if AOS8 catalog changes or skill structure is rewritten outside Phase 11)
