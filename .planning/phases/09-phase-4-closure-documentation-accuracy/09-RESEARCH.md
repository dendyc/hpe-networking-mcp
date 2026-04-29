# Phase 9: Phase 4 Closure & Documentation Accuracy — Research

**Researched:** 2026-04-28
**Domain:** Documentation hygiene + planning-artifact integrity (no new code, no new libraries)
**Confidence:** HIGH (every fact is grounded in files already in the repo)

## Summary

Phase 9 is a pure planning/doc-cleanup phase. Phases 7 and 8 absorbed all DIFF-01..09 implementation work; Phase 4's planning directory was left with only a `04-CONTEXT.md` and `04-DISCUSSION-LOG.md` and no `04-VERIFICATION.md`, leaving an audit gap. In parallel, the Phase 6 doc batch (CHANGELOG 2.4.0.0, README capability table, README architecture diagrams, README project tree, docs/TOOLS.md, server.py `execute_description`) was finalized while AOS8 still had only 38 tools — the 9 differentiator tools were added in Phase 7 but the user-facing strings were never refreshed, so every doc surface still reports 38.

There is also a small code-string fix: `server.py` line 373 lists callable tool prefixes inside the code-mode `execute_description` literal (`mist_`, `central_`, `greenlake_`, `clearpass_`, `apstra_`, `axis_`) and omits `aos8_`. The string is not generated — it is hard-coded — so the fix is a one-line literal edit, validated by an existing or new test that asserts `"aos8_"` appears in the rendered description.

**Primary recommendation:** Treat Phase 9 as a single-plan structural/text-edit phase. No research libraries, no new patterns. Execution is a checklist: write `04-VERIFICATION.md`, flip 9 checkboxes + 9 traceability rows in `REQUIREMENTS.md`, change `38` → `47` in 5 specific spots across `README.md`, `CHANGELOG.md`, `docs/TOOLS.md`, add a "Differentiators (9)" subsection to `docs/TOOLS.md`, and add `aos8_` to one tuple-literal string in `server.py`. Add one regression test guarding the `execute_description` string so this drift cannot recur.

## User Constraints (from CONTEXT.md)

No `09-CONTEXT.md` exists yet — this phase has not been through `/gsd:discuss-phase`. Constraints come directly from the ROADMAP Phase 9 success criteria (REQUIREMENTS.md DIFF-01..09 + DOCS-01..04) and from prior phase context files:

### Locked Decisions (from ROADMAP success criteria + Phase 8 SUMMARY)

- D-01: A `04-VERIFICATION.md` MUST be created in `.planning/phases/04-differentiator-tools/` (success criterion #1) — even though Phase 4 was absorbed, planning integrity requires the file to exist and cross-reference Phase 7's VERIFICATION.md Observable Truth #1.
- D-02: REQUIREMENTS.md DIFF-01..09 must be marked `[x]` and the traceability table must report "Complete" for each.
- D-03: Tool count is 47 (26 read + 12 write + 9 differentiators), not 38. This is verified by `tests/unit/test_aos8_init.py`'s `EXPECTED_TOTAL = 26 + 12 + 9 = 47` and Phase 7 VERIFICATION.md.
- D-04: Phase 4 is to be documented as "absorbed into Phase 7 (plans 07-01/07-02/07-03)" — this language already appears in REQUIREMENTS.md line 222 and ROADMAP line 14.
- D-05: Per Phase 8 SUMMARY decision: "doc refresh remains tracked under Phase 9 (version bump + CHANGELOG entry)." Phase 9 owns the version bump narrative (the pyproject.toml bump is a separate concern — STATE.md "Open Todos" lists it as 2.4.0.1 or 2.4.1.0).
- D-06 (carried from Phase 7 CONTEXT): "authorized doc deviation" — Phase 7 was completed knowing docs would lag; that authorization is retired by completing this phase.

### Claude's Discretion

- Whether to bump pyproject.toml version (2.4.0.1 vs 2.4.1.0). REQUIREMENTS.md DOCS-05 was already checked at Phase 6, so the version bump is not strictly part of DOCS-01..04. STATE.md flags it as a Phase 8 / next-patch open todo. Recommendation: bump to **2.4.0.1** (patch) because the user-visible behavior change is "doc accuracy + 9 already-existing tools now correctly counted + DIFF response-contract bug fix from Phase 8" — no new functionality, fits patch semantics.
- Format of `04-VERIFICATION.md` — recommend mirroring `07-VERIFICATION.md` structure (Observable Truths table, Required Artifacts, Score) but with most rows showing "DELEGATED to Phase 7" with cross-reference links.
- Whether to add a regression test for the `execute_description` aos8_ inclusion. Recommendation: YES — without one, the same string-drift bug could recur silently.
- Whether to update CHANGELOG by editing the existing 2.4.0.0 entry vs adding a new 2.4.0.1 entry. Recommendation: **add a new 2.4.0.1 entry** documenting (a) Phase 8 DIFF response-contract fix, (b) corrected tool counts, (c) `execute_description` fix. Editing released entries violates Keep a Changelog convention.

### Deferred Ideas (OUT OF SCOPE)

- Re-running the Phase 4 plans as actual plans — Phase 4 is permanently merged.
- Any change to `differentiators.py`, `_helpers.py`, test infrastructure, or runtime behavior — Phase 8 closed those concerns.
- Adding INSTRUCTIONS.md content for differentiators — DOCS-01 was already satisfied at Phase 6 and is not in Phase 9's requirement list.
- Updating cross-platform tools or other platform tool counts.

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DIFF-01 | `aos8_get_md_hierarchy` | Implementation lives in `differentiators.py` (verified Phase 7 + Phase 8). Phase 9 only ticks `[x]` and updates traceability table. |
| DIFF-02 | `aos8_get_effective_config` | Same — implementation done; doc-only. |
| DIFF-03 | `aos8_get_pending_changes` | Same. |
| DIFF-04 | `aos8_get_rf_neighbors` | Same. |
| DIFF-05 | `aos8_get_cluster_state` | Same. |
| DIFF-06 | `aos8_get_air_monitors` | Same. |
| DIFF-07 | `aos8_get_ap_wired_ports` | Same. |
| DIFF-08 | `aos8_get_ipsec_tunnels` | Same. |
| DIFF-09 | `aos8_get_md_health_check` | Same. All 9 already verified GREEN by Phase 7 and Phase 8 (13 DIFF tests + 5 DIFF-09 sub-calls in `aos8_get_md_health_check`). |
| DOCS-01 | INSTRUCTIONS.md AOS8 section | **Already satisfied at Phase 6.** Phase 9 only re-ticks the row to keep the matrix consistent — no edit needed unless adding differentiator-specific guidance is desired (out of scope per Deferred Ideas). |
| DOCS-02 | README.md AOS8 row + tool count | Five `38` → `47` edits across capability table line 55, narrative line 59, dynamic-mode summary line 199, ASCII diagram line 417, project tree line 564. |
| DOCS-03 | CHANGELOG.md version entry | Add new `[2.4.0.1] - 2026-04-29` entry (or chosen version) documenting the doc fix + Phase 8 response-contract fix + execute_description fix. Optionally annotate existing 2.4.0.0 entry: "Note: tool count corrected to 47 in 2.4.0.1." |
| DOCS-04 | docs/TOOLS.md AOS8 reference | Update header `Aruba OS 8 / Mobility Conductor (38 tools + 9 prompts)` → `(47 tools + 9 prompts)` AND add a new `### Differentiators (9)` subsection between Troubleshooting (line 1979) and Writes (line 1979) listing all 9 DIFF tools with descriptions sourced from REQUIREMENTS.md DIFF-01..09. |

## Standard Stack

No new libraries. Phase 9 uses what is already in the repo:

| Tool | Purpose | Why Standard |
|------|---------|--------------|
| `pytest` 8.4+ | Run regression for the new `execute_description` test | Already the project test framework |
| `ruff` 0.15.6 | Lint check after edits | Project standard |
| `mypy` 1.18+ | Type check after edits | Project standard |
| Git (sub-repo + meta-repo workflow) | Two-repo commit pattern from Phase 8 | Established by Phase 7/8 — `hpe-networking-mcp/` source edits commit in sub-repo; `.planning/` artifacts commit in `adams_mcp/` worktree |

## Architecture Patterns

### Pattern 1: Cross-Reference VERIFICATION.md for Merged Phases

**What:** When a phase is administratively retired and its work absorbed elsewhere, create a VERIFICATION.md that exists, scores 0/0 or "DELEGATED", and contains hyperlink cross-references to the actual verifying phase.

**When to use:** Phase 4 here. The ROADMAP and REQUIREMENTS already use the language "merged into Phase 7"; the VERIFICATION.md formalizes that.

**Example skeleton:**

```markdown
---
phase: 04-differentiator-tools
verified: 2026-04-29T00:00:00Z
status: delegated
score: 0/0 truths owned by this phase (all delegated)
re_verification: false
---

# Phase 4: Differentiator Tools — Verification Report (Delegated)

**Phase Goal:** Implement DIFF-01..09 (9 AOS8 differentiator read tools).
**Status:** DELEGATED to Phase 7 (plans 07-01, 07-02, 07-03) and corrected by Phase 8 (plan 08-01).
**Verified by:** `.planning/phases/07-testing-integration/07-VERIFICATION.md` (Observable Truth #1) and `.planning/phases/08-fix-diff-tools-production-bug/08-01-SUMMARY.md`.

## Why this phase has no plans of its own
After Phase 6 closed, the v1.0 milestone audit determined the differentiators
were tightly coupled to the Testing & Integration phase's TDD workflow. The
9 DIFF tools were planned and implemented as part of Phase 7 plans 07-01
(red scaffold), 07-02 (implementation), and 07-03 (registration & regression).
Phase 8 plan 08-01 corrected a response-contract bug in the same tools.

## Cross-References

| DIFF Requirement | Implemented In | Verified In |
|---|---|---|
| DIFF-01..09 | `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` (Phase 7 P02; refactored Phase 8 P01) | `07-VERIFICATION.md` Truth #1 + `tests/unit/test_aos8_read_differentiators.py` (13 tests GREEN) |

## Observable Truths

All 5 Observable Truths from this phase are owned by Phase 7 — see
`07-VERIFICATION.md`. No truths are owned locally.

---
_Verified: 2026-04-29_
_Verifier: Claude (gsd-verifier) — administrative closure_
```

### Pattern 2: Atomic Doc-Sweep Edits

**What:** When changing a tool count, find every occurrence with `grep -n "38" <files>` (scoped to the doc set), edit each one, then `grep` again to confirm zero remaining hits in the doc files. Mechanical, exhaustive, verified.

**When to use:** Phase 9 task for the 38 → 47 sweep.

**Verified target list (lines confirmed by inspection):**

```
hpe-networking-mcp/README.md:55       — capability table footer row "AOS8 column"
hpe-networking-mcp/README.md:59       — v2.4.0.0 narrative "(38 tools + 9 prompts)"
hpe-networking-mcp/README.md:199      — startup log example "AOS8: 38 underlying tools"
hpe-networking-mcp/README.md:417      — ASCII architecture diagram "│38 tools│"
hpe-networking-mcp/README.md:564      — project tree comment "38 AOS8 tools + 9 prompts"
hpe-networking-mcp/CHANGELOG.md:12    — 2.4.0.0 entry "38 tools across 6 categories"
hpe-networking-mcp/CHANGELOG.md:24    — "AOS8 (38 + 9 prompts)"
hpe-networking-mcp/docs/TOOLS.md:1922 — section header "(38 tools + 9 prompts)"
```

The 6-categories breakdown in `CHANGELOG.md:12` becomes 7 categories: 8 health/inventory + 4 client + 3 alert/audit + 4 WLAN/config + 7 troubleshooting + **9 differentiators** + 12 write = 47.

### Pattern 3: One-Line Code Fix Guarded by a Test

**What:** When fixing a literal string in code (here `execute_description`), pair the edit with a unit test that asserts the contract. Without the test, the same drift recurs after the next platform addition.

**When to use:** Phase 9 task for `server.py` line 373.

**Example test:**

```python
# tests/unit/test_server_code_mode.py
from hpe_networking_mcp.server import create_server  # or import the literal directly
import re

def test_execute_description_lists_aos8_prefix():
    """Code-mode execute() must advertise aos8_ as a callable platform prefix."""
    # The execute_description is a module-level literal; reading the source is fine
    import hpe_networking_mcp.server as srv
    src = open(srv.__file__, encoding="utf-8").read()
    # Find the execute_description block
    match = re.search(r"execute_description\s*=\s*\((.*?)\)", src, re.DOTALL)
    assert match, "execute_description literal not found"
    body = match.group(1)
    for prefix in ("mist_", "central_", "greenlake_", "clearpass_",
                   "apstra_", "axis_", "aos8_"):
        assert f"`{prefix}`" in body, f"execute_description missing {prefix}"
```

Alternative (cleaner, decoupled from file parsing): refactor `execute_description` to be assembled from `_PLATFORM_PREFIXES = ("mist_", "central_", ...)` so the test asserts membership in the constant. Trade-off: bigger diff. Recommend the simple file-read test for minimal blast radius.

### Anti-Patterns to Avoid

- **Editing the existing CHANGELOG 2.4.0.0 entry in place** — violates Keep a Changelog "released entries are immutable." Add a new patch entry instead.
- **Sweeping `38` globally with sed** — `38` appears legitimately in unrelated places (`docs/TOOLS.md:546-547` references `38 clients` in central_get_site_health, `:1692` references `~38 mapped fields` in WLAN docs). Constrain edits to the verified line list above.
- **Assuming `INSTRUCTIONS.md` needs updating** — DOCS-01 was satisfied at Phase 6; differentiator-specific operator guidance is out of scope per Deferred Ideas.
- **Bumping minor version (2.4.1.0) instead of patch (2.4.0.1)** — no new user-visible features; only doc accuracy + bug fix from Phase 8.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Verifying tool count after edit | A Python script that re-counts 47 | The existing `tests/unit/test_aos8_init.py` | Already asserts `EXPECTED_TOTAL = 47` and is run in CI |
| Asserting DIFF tools register | Custom registration probe | Existing test_aos8_init.py + test_aos8_read_differentiators.py | All 13 DIFF + 3 init tests already GREEN |
| Verifying VERIFICATION.md format | Bespoke YAML schema | Copy structure from `07-VERIFICATION.md` | Same agent (`gsd-verifier`) consumed both |

## Common Pitfalls

### Pitfall 1: Missing `38` Occurrences

**What goes wrong:** Edit 4 of 5 `38` references in README.md, miss one, ship inconsistent docs.
**Why it happens:** Doc files are long (README.md > 600 lines, TOOLS.md > 2000 lines); `38` is also a legitimate number in unrelated content.
**How to avoid:** Use the explicit line-list above. After editing, run `grep -nE "(^|[^0-9])38( |$|\b.*tool|\b.*aos8|\b.*AOS8)" hpe-networking-mcp/README.md hpe-networking-mcp/CHANGELOG.md hpe-networking-mcp/docs/TOOLS.md` and confirm zero AOS8-related results.
**Warning signs:** Any remaining `aos8.*38` or `38.*tool.*aos8` match after the sweep.

### Pitfall 2: Editing Wrong CHANGELOG Section

**What goes wrong:** Modifying the released `[2.4.0.0]` section instead of adding a new entry.
**Why it happens:** Familiar reflex when the release notes are slightly wrong.
**How to avoid:** Insert the new `[2.4.0.1] - 2026-04-29` entry at the top of CHANGELOG.md (immediately after the introductory paragraph, before line 8). Optionally include a one-line "Note in [2.4.0.0]: tool count was incorrectly stated as 38; corrected to 47 in 2.4.0.1."
**Warning signs:** Diff shows changes inside the existing 2.4.0.0 entry beyond an annotation note.

### Pitfall 3: docs/TOOLS.md Differentiators Section Inserted in Wrong Place

**What goes wrong:** New `### Differentiators (9)` subsection placed at the bottom of the AOS8 section or before reads.
**Why it happens:** Section ordering in TOOLS.md follows a category ordering that's not obvious.
**How to avoid:** Insert immediately after Troubleshooting (line 1977 area) and before `### Writes (12)` (line 1979). The category order then matches CHANGELOG breakdown: health → clients → alerts → WLAN → troubleshooting → differentiators → writes.
**Warning signs:** Section header for `### Writes (12)` no longer immediately follows differentiators.

### Pitfall 4: Two-Repo Commit Confusion

**What goes wrong:** Committing planning artifact changes (`.planning/phases/04-VERIFICATION.md`, `.planning/REQUIREMENTS.md`) in the `hpe-networking-mcp/` sub-repo instead of the `adams_mcp/` worktree.
**Why it happens:** This project has two sibling git repos: `adams_mcp/` (planning + meta) and `hpe-networking-mcp/` (source code). Phase 8 SUMMARY documents the convention.
**How to avoid:**
  - Source edits (`server.py`, `README.md`, `CHANGELOG.md`, `docs/TOOLS.md`, `tests/unit/test_server_code_mode.py`) → commit in `hpe-networking-mcp/` sub-repo.
  - Planning artifacts (`04-VERIFICATION.md`, `REQUIREMENTS.md`, `STATE.md`, `ROADMAP.md`, `09-*.md`) → commit in `adams_mcp/` worktree.
**Warning signs:** `git status` in `adams_mcp/` shows source-file changes, or vice versa.

### Pitfall 5: Forgetting that DOCS-01..05 Are Already Checked

**What goes wrong:** Re-implementing INSTRUCTIONS.md or pyproject.toml updates that were already satisfied at Phase 6.
**Why it happens:** Phase 9 lists DOCS-01..04 in its requirement set, but inspection of REQUIREMENTS.md shows all 5 DOCS rows already `[x]`.
**How to avoid:** Re-read REQUIREMENTS.md lines 105-109 before planning DOCS work. Phase 9's DOCS work is **correcting** what was written at Phase 6, not re-doing it. The traceability table rows for DOCS-01..04 say "Phase 6"; Phase 9 does NOT change those — only adds the corrective note in CHANGELOG.
**Warning signs:** Plan tasks duplicate Phase 6 deliverables.

## Code Examples

### Verified file locations (all paths absolute or repo-relative)

```
.planning/REQUIREMENTS.md                                        — DIFF rows 61-69 (checkboxes), table rows 178-186
.planning/phases/04-differentiator-tools/                        — needs new 04-VERIFICATION.md (no plans, no summary)
.planning/phases/07-testing-integration/07-VERIFICATION.md       — Observable Truth #1 cross-reference target
hpe-networking-mcp/README.md                                     — 5 edits at lines 55, 59, 199, 417, 564
hpe-networking-mcp/CHANGELOG.md                                  — new 2.4.0.1 entry; references at lines 12, 24
hpe-networking-mcp/docs/TOOLS.md                                 — line 1922 header + new subsection between 1977 and 1979
hpe-networking-mcp/src/hpe_networking_mcp/server.py              — line 373 add `aos8_` to prefix tuple in execute_description
hpe-networking-mcp/tests/unit/test_aos8_init.py                  — already asserts 47 (no change)
hpe-networking-mcp/tests/unit/test_server_code_mode.py           — NEW (or extend existing if present) for execute_description test
```

### server.py edit (literal patch)

```python
# Before (server.py line 372-374)
"`call_tool` ONLY dispatches to platform tools — names start with one "
"of: `mist_`, `central_`, `greenlake_`, `clearpass_`, `apstra_`, "
"`axis_` — plus the cross-platform `health` tool.\n\n"

# After
"`call_tool` ONLY dispatches to platform tools — names start with one "
"of: `mist_`, `central_`, `greenlake_`, `clearpass_`, `apstra_`, "
"`axis_`, `aos8_` — plus the cross-platform `health` tool.\n\n"
```

### docs/TOOLS.md new subsection (insert before line 1979 "### Writes (12)")

```markdown
### Differentiators (9)

AOS8-unique read tools that go beyond Aruba Central parity — Conductor
hierarchy, effective configuration after inheritance, RF neighbors,
cluster state, IPsec tunnels, and a unified per-MD health roll-up.

| Tool | Purpose |
|---|---|
| `aos8_get_md_hierarchy` | Conductor → Managed Device tree with config_path for each node |
| `aos8_get_effective_config` | Resolved config a specific MD or AP group sees after inheritance |
| `aos8_get_pending_changes` | Staged Conductor changes not yet persisted via `aos8_write_memory` |
| `aos8_get_rf_neighbors` | ARM neighbor graph for an AP — co-channel and adjacent-channel |
| `aos8_get_cluster_state` | AP cluster membership, master/standby roles, failover state |
| `aos8_get_air_monitors` | APs in air-monitor mode with scan results |
| `aos8_get_ap_wired_ports` | Wired downlink port configuration and state for APs |
| `aos8_get_ipsec_tunnels` | Site-to-site IPsec and Remote AP tunnel state |
| `aos8_get_md_health_check` | Unified per-MD health: APs + clients + alarms + firmware in one call |
```

## Runtime State Inventory

This phase is documentation + a one-line code-string edit. There is no rename, no datastore migration, no service reconfiguration. The Runtime State Inventory categories are not applicable:

| Category | Status |
|----------|--------|
| Stored data | None — no datastores reference any string being changed. |
| Live service config | None — no external service has a config holding "38" that would auto-update. |
| OS-registered state | None — no OS-level registrations involved. |
| Secrets/env vars | None — no secrets reference tool counts. |
| Build artifacts | None — wheel rebuild on next release picks up `server.py` change automatically; pip egg-info will refresh on install. |

## Environment Availability

This phase has no external dependencies beyond what Phase 7/8 already used:

| Dependency | Required By | Available | Version | Fallback |
|------------|-------------|-----------|---------|----------|
| `pytest` | run regression after edits | ✓ | 8.4+ (pyproject.toml) | — |
| `ruff` | lint check after edits | ✓ | 0.15.6+ | — |
| `mypy` | type check after edits | ✓ | 1.18+ | — |
| `uv` | task-runner for the above | ✓ | latest | — |
| Git (sub-repo + meta-repo) | two-repo commit pattern | ✓ | system git | — |

No missing dependencies.

## Validation Architecture

(Included because `.planning/config.json` has `workflow.nyquist_validation: true`.)

### Test Framework

| Property | Value |
|----------|-------|
| Framework | pytest 8.4+ with `pytest-asyncio` (asyncio_mode=auto) |
| Config file | `hpe-networking-mcp/pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `cd hpe-networking-mcp && uv run pytest tests/unit -k "aos8 or server" -q` |
| Full suite command | `cd hpe-networking-mcp && uv run pytest tests/unit -q` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|--------------|
| DIFF-01..09 | All 9 DIFF tools registered in TOOLS["differentiators"] and reachable | unit | `uv run pytest tests/unit/test_aos8_init.py -v` | ✅ (existing — asserts EXPECTED_TOTAL=47) |
| DIFF-01..09 | All 9 DIFF tools return parsed JSON (not httpx.Response) | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py -v` | ✅ (existing — 13 tests GREEN after Phase 8) |
| DOCS-02 | README.md tool count consistent at 47 | manual + grep | `grep -nE "aos8.{0,40}38|38.{0,40}aos8" hpe-networking-mcp/README.md` returns 0 | ❌ (Wave 0 — add a doc-grep assertion or accept as manual gate) |
| DOCS-03 | CHANGELOG.md has 2.4.0.1 entry | manual | Visual review | manual-only |
| DOCS-04 | docs/TOOLS.md AOS8 section lists 9 differentiators | manual + grep | `grep -nE "aos8_get_md_hierarchy" hpe-networking-mcp/docs/TOOLS.md` returns ≥ 1 | ❌ (Wave 0 — add to a doc-completeness test or accept manual) |
| (server fix) | execute_description includes `aos8_` | unit | `uv run pytest tests/unit/test_server_code_mode.py::test_execute_description_lists_aos8_prefix -v` | ❌ (Wave 0 — new test) |

### Sampling Rate

- **Per task commit (sub-repo):** `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_init.py tests/unit/test_aos8_read_differentiators.py tests/unit/test_server_code_mode.py -q`
- **Per wave merge / phase gate (sub-repo):** `cd hpe-networking-mcp && uv run pytest tests/unit -q && uv run ruff check . && uv run ruff format --check . && uv run mypy src/ --ignore-missing-imports`
- **Per planning-artifact commit (meta-repo):** No automated test (text-only); rely on visual diff review.

### Wave 0 Gaps

- [ ] `tests/unit/test_server_code_mode.py` — new test asserting `execute_description` lists `aos8_` (also covers all 7 platform prefixes for future-proofing). Required because the bug pattern (literal-string drift) has no other automated guard.
- [ ] (Optional, recommended) Doc-completeness assertion: a small test in `tests/unit/test_aos8_docs.py` that greps `docs/TOOLS.md` for the 9 DIFF tool names. Trade-off: doc tests are noisy in CI; recommendation: skip unless the team wants stronger guardrails.
- [ ] No framework install needed; pytest already configured.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Phase 7 SUMMARY noted "authorized doc deviation per CONTEXT.md D-06" | Phase 9 retires that authorization by completing the doc refresh | This phase | Closes the v1.0 audit finding |
| Phase 4 dir held only CONTEXT + DISCUSSION-LOG | Phase 4 dir gains a delegated 04-VERIFICATION.md cross-referencing Phase 7 | This phase | Planning integrity restored |
| `execute_description` listed 6 platform prefixes | Lists 7 (adds `aos8_`) | This phase | Code-mode sandbox can dispatch AOS8 tools without `Unknown tool: aos8_*` failure |

**Deprecated/outdated:** None — Phase 9 introduces no replacements, only fixes.

## Open Questions

1. **Should pyproject.toml version bump be in scope of Phase 9?**
   - What we know: STATE.md "Open Todos" lists "pyproject.toml version bump to 2.4.0.1 or 2.4.1.0" under Phase 8 / next-patch. Phase 9 success criteria does NOT explicitly require it.
   - What's unclear: Whether the user wants the version bump now or queued for a separate release-prep phase.
   - Recommendation: Include the bump in Phase 9 (one-line edit in pyproject.toml from 2.4.0.0 → 2.4.0.1) AND add a CHANGELOG 2.4.0.1 entry. Rationale: a CHANGELOG entry without a version-source bump creates a publish/source mismatch on the next release tag. Surface this as a decision for the planner / `/gsd:discuss-phase` if not already locked.

2. **Should the `04-VERIFICATION.md` use `score: 5/5` (mirroring Phase 7's score for the same truths) or `score: 0/0 delegated`?**
   - Recommendation: `score: 0/0 (all DELEGATED to Phase 7)` with explicit note that the Phase 7 Observable Truths cover all 5 success criteria. Avoids double-counting in any future audit aggregation.

3. **Should we also annotate REQUIREMENTS.md line 222 ("Note: DIFF-01..09 implemented by Phase 7 ...")?**
   - That line already says "Phase 4 formally retired as merged" — appears correct as-is. Recommendation: leave unchanged; the Phase column for DIFF-01..09 in the traceability table currently says "Phase 8" (line 178-186), which is also accurate (Phase 8 closed them). No edit needed.

## Project Constraints (from CLAUDE.md)

- **API:** GET and POST only — N/A (no API work in Phase 9).
- **Testing:** All tests use mocked HTTP responses — applies to the new `test_server_code_mode.py` (no network).
- **Compatibility:** Must not break any existing platform module or test — Phase 9 plan must end with full suite green (764 tests + any new test).
- **Style:** Ruff lint + format, mypy type-checked, max 500 lines/file, Google-style docstrings — applies to the one-line `server.py` edit and new test file.
- **Docstrings:** New test must have a Google-style docstring.
- **Documentation Checklist (CLAUDE.md):** "Every PR that changes tools, prompts, or functionality MUST update documentation IN THE SAME PR" — Phase 9 IS that doc update for the Phase 7/8 changes. The checklist items map directly: README.md tool counts ✓, CHANGELOG.md new version entry ✓, INSTRUCTIONS.md (no change — Phase 6 covered), docs/TOOLS.md ✓, pyproject.toml version bump (recommended, see Open Question 1).
- **Git workflow:** Two-repo pattern from Phase 8 SUMMARY — source edits in `hpe-networking-mcp/`, planning artifacts in `adams_mcp/`. Branch protection on `main` requires PR (per CLAUDE.md), but project status indicates direct commit to `master` is acceptable for this fork (recent commits are direct to master per `git log`).
- **Pre-Push Checklist:** `ruff check . && ruff format --check . && mypy src/ --ignore-missing-imports && pytest tests/ -q` — run in `hpe-networking-mcp/` after every plan task.

## Sources

### Primary (HIGH confidence)
- `.planning/REQUIREMENTS.md` (lines 60-69 DIFF block, 105-109 DOCS block, 178-186 traceability, 222 Phase 4 retirement note)
- `.planning/STATE.md` (Phase status, open todos, performance metrics for Phases 7-8)
- `.planning/ROADMAP.md` (lines 144-156 Phase 9 spec; lines 14, 165, 222 Phase 4 retirement language)
- `.planning/phases/07-testing-integration/07-VERIFICATION.md` (Observable Truth #1 — cross-ref target)
- `.planning/phases/08-fix-diff-tools-production-bug/08-01-SUMMARY.md` (Phase 8 closure; doc-deferral decision)
- `hpe-networking-mcp/README.md` (5 edit-target lines verified)
- `hpe-networking-mcp/CHANGELOG.md` (lines 8-29 — 2.4.0.0 entry verified)
- `hpe-networking-mcp/docs/TOOLS.md` (lines 1922-1998 AOS8 section verified)
- `hpe-networking-mcp/src/hpe_networking_mcp/server.py` (lines 368-385 execute_description literal verified)
- `hpe-networking-mcp/CLAUDE.md` (Documentation Checklist constraint)
- `CLAUDE.md` (project-level) — constraints section

### Secondary (MEDIUM confidence)
- Phase 8 plan recommends 2.4.0.1 version bump; Phase 9 inherits — version choice still discretion-level.

### Tertiary (LOW confidence)
- None. Every claim in this RESEARCH.md is grounded in a file already in the repo.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — same toolchain as Phases 7/8, no novel dependencies.
- Architecture: HIGH — three patterns (delegated VERIFICATION.md, atomic doc sweep, one-line code edit + test) are all simple text/structural operations on verified files.
- Pitfalls: HIGH — pitfalls are observable (e.g., 38 occurrences enumerated; CHANGELOG immutability is project convention; two-repo pattern documented in Phase 8 SUMMARY).

**Research date:** 2026-04-28
**Valid until:** 2026-05-28 (stable scope; only invalidated if README/CHANGELOG/TOOLS.md change before plan execution)
