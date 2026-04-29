# Phase 7: Testing & Integration - Context

**Gathered:** 2026-04-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Phase 7 is a combined **implementation + testing** phase with two deliverables:

1. **DIFF tools implementation** — All 9 AOS8 differentiator read tools (DIFF-01..09) that were scoped in Phase 4 but never executed. These expose AOS8-unique capabilities (Conductor hierarchy, effective config, RF neighbors, cluster state, etc.) that have no Aruba Central equivalent.

2. **Testing & Integration** — Token-leak audit at the tool layer (TEST-04), gap fill for any thin test coverage, and full regression confirmation across all 7 platforms.

Deliverables:
- `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` — all 9 DIFF tools
- `tests/unit/test_aos8_read_differentiators.py` — unit tests with mocked API responses
- `tests/unit/test_aos8_security.py` (or extension of `test_aos8_client.py`) — cross-cutting token-leak audit covering tool invocations
- `platforms/aos8/__init__.py` updated — `TOOLS["differentiators"]` key added
- Full suite (767+ tests) passing; no existing platform tests broken

**Not in scope:** Coverage thresholds, CI pipeline changes, documentation updates beyond what Phase 6 already completed.

</domain>

<decisions>
## Implementation Decisions

### DIFF Tools — Include All 9
- **D-01:** All 9 DIFF tools are in scope for Phase 7 — DIFF-01 through DIFF-09. No trimming.
  - DIFF-01: `aos8_get_md_hierarchy`
  - DIFF-02: `aos8_get_effective_config`
  - DIFF-03: `aos8_get_pending_changes`
  - DIFF-04: `aos8_get_rf_neighbors`
  - DIFF-05: `aos8_get_cluster_state`
  - DIFF-06: `aos8_get_air_monitors`
  - DIFF-07: `aos8_get_ap_wired_ports`
  - DIFF-08: `aos8_get_ipsec_tunnels`
  - DIFF-09: `aos8_get_md_health_check`

- **D-02:** Single file: `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py`. A new `"differentiators"` key is added to `TOOLS` dict in `platforms/aos8/__init__.py`. Existing category keys are not modified.

- **D-03:** Phase 4 CONTEXT.md already has design decisions for DIFF tools — researcher MUST read it before planning. Key decisions from Phase 4:
  - DIFF-09 (`aos8_get_md_health_check`) is a multi-call aggregator: AP counts, client count, alarm summary, firmware version for a `config_path` scope. `config_path` required (no default).
  - DIFF-02 (`aos8_get_effective_config`) accepts `object_name: str` + `config_path: str` (default `/md`). Uses config-object endpoint with inheritance resolution; fallback to show command.
  - DIFF-03 (`aos8_get_pending_changes`) uses config-object or dedicated endpoint; show command fallback if no direct path exists.
  - For all DIFF tools: if no direct config-object endpoint, fallback to show command API.

### Plan Structure — 3 Plans (TDD)
- **D-04:** 3 plans mirroring Phase 3/4 discipline:
  - **Plan 07-01 (Wave 0):** Write all DIFF tests (`test_aos8_read_differentiators.py`) + token-leak security test — all RED baseline. No implementation.
  - **Plan 07-02 (Implementation):** Implement all 9 DIFF tools in `differentiators.py`. Tests go GREEN.
  - **Plan 07-03 (Wiring + Regression):** Wire `TOOLS["differentiators"]` in `__init__.py`, run full suite, confirm all 767+ tests passing and no existing platform tests broken.

### Token-Leak Audit — One Cross-Cutting Test (TEST-04)
- **D-05:** Add one cross-cutting test (in `test_aos8_security.py` or appended to `test_aos8_client.py`) that:
  1. Invokes a read tool (e.g., `aos8_get_version` or similar) while capturing loguru output
  2. Invokes a write tool (or simulates the write path) while capturing loguru output
  3. Asserts `"UIDARUBA"` and the actual token value never appear in any captured log line
  This complements the existing `test_uidaruba_never_in_logs` (client-level) by covering the tool invocation path.

### Coverage Enforcement — None
- **D-06:** No pytest-cov threshold added to pyproject.toml or CI. Phase 7 success is: all tests pass. Coverage report generation can be added separately later if needed.

### Regression Strategy
- **D-07:** TEST-06 is verified by running `pytest tests/unit` in full — if any of the 6 existing platform test suites break, the executor stops and reports the failure. No modifications to existing test files.

### Claude's Discretion
- Exact AOS8 REST endpoint paths or show commands for each DIFF tool (researcher to verify via Phase 4 CONTEXT.md canonical refs and API docs)
- JSON fixture file names and structure for DIFF test mocks (follow `tests/unit/fixtures/aos8/` pattern from Phase 3)
- Whether `config_path` is scope-sensitive for DIFF-04, DIFF-06, DIFF-07, DIFF-08 — researcher to confirm
- File placement of the cross-cutting token-leak test (new `test_aos8_security.py` vs appended class in `test_aos8_client.py`) — researcher/planner to decide based on test count and coherence

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase 4 Design Decisions (MUST READ)
- `.planning/phases/04-differentiator-tools/04-CONTEXT.md` — All design decisions for DIFF-01..09 (file structure, API fallback strategy, DIFF-09 aggregation approach, config_path sensitivity). These decisions are locked and carry forward.

### AOS8 Tool Implementation Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py` — canonical AOS8 tool file: `@tool(annotations=READ_ONLY)`, Context injection, `run_show()` / `get_object()` usage, error handling
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` — `run_show()`, `get_object()`, `strip_meta()`, `format_aos8_error()` helpers

### AOS8 Platform Module
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — `TOOLS` dict to extend with `"differentiators"` key; `register_tools()` import loop (already handles new modules automatically)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` — `AOS8Client.request()`, `AOS8APIError`, `AOS8AuthError`

### Existing Test Patterns
- `hpe-networking-mcp/tests/unit/test_aos8_read_health.py` — canonical test file for read tools: fixture JSON loading, mock transport setup, `loguru_capture` fixture usage
- `hpe-networking-mcp/tests/unit/test_aos8_client.py` — `TestAOS8ClientLogging.test_uidaruba_never_in_logs` — the client-level token audit test to extend/mirror at tool layer
- `hpe-networking-mcp/tests/unit/fixtures/` — fixture JSON directory; DIFF tool fixtures go here

### No External API Specs
- No external docs referenced during discussion — all API endpoint details are to be determined by researcher using Phase 4 CONTEXT.md + AOS8 REST API knowledge.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `_helpers.py`: `run_show()`, `get_object()`, `strip_meta()`, `format_aos8_error()` — all DIFF tools should use these, no new helpers needed unless researcher finds a gap
- `tests/unit/fixtures/aos8/` — fixture JSON directory already established; add DIFF fixtures here
- `loguru_capture` fixture — already in conftest; use for token-leak test

### Established Patterns
- All read tools: single-file per category, `@tool(annotations=READ_ONLY)`, `lifespan_context["aos8_client"]` access
- TDD discipline: Wave 0 tests written first (all RED), then implementation (GREEN), then wiring
- TOOLS dict loop: `importlib.import_module(f"hpe_networking_mcp.platforms.aos8.tools.{category}")` — adding `"differentiators"` key automatically picks up the new module

### Integration Points
- `platforms/aos8/__init__.py` `TOOLS` dict: add `"differentiators": ["aos8_get_md_hierarchy", ...]` key
- `register_tools()` importlib loop already handles new category modules — no code change needed beyond the TOOLS dict entry

</code_context>

<specifics>
## Specific Ideas

- The user confirmed Phase 7 should implement all 9 DIFF tools (not just test what was already built) — Phase 7 is explicitly a combined implementation+testing phase.
- Phase 4 context is the authoritative design document for DIFF tools; researcher must read it before any planning.
- Token-leak audit should cover the tool invocation layer (not just the client) — one cross-cutting test is sufficient.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 07-testing-integration*
*Context gathered: 2026-04-28*
