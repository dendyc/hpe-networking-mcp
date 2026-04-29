# Phase 7: Testing & Integration - Research

**Researched:** 2026-04-28
**Domain:** Async pytest testing of an AOS8 MCP platform module — DIFF tool implementation + token-leak audit + 7-platform regression
**Confidence:** HIGH

## Summary

Phase 7 is a combined implementation+testing phase. It (1) ships the 9 DIFF tools that were originally scoped to Phase 4 but never executed, (2) adds the missing TEST-04 cross-cutting token-leak audit at the tool layer, and (3) confirms full regression across all 7 platforms (748 tests already collected pre-Phase-7; +9 DIFF tool tests + ~2 security tests will land at ~759 final).

The codebase already has every piece of infrastructure these new tests need: fixture loader (`_load_fixture` pattern), `MagicMock` + `AsyncMock` httpx-response builder (`_make_ctx`), `loguru_capture` fixture, conftest registry stubbing, `_helpers.py` (`run_show`, `get_object`, `strip_meta`, `format_aos8_error`), `READ_ONLY` annotation, and the `TOOLS` dict's importlib loop that auto-picks up a new `"differentiators"` category. There is **zero new test infrastructure to invent** — Plan 7 is pattern-matching against `test_aos8_read_health.py` and `test_aos8_client.py::TestAOS8ClientLogging`.

**Primary recommendation:** Mirror the Phase 3 / Phase 5 TDD discipline exactly — Plan 07-01 lands all DIFF tests + the new tool-layer token-leak test as RED; Plan 07-02 implements `differentiators.py`; Plan 07-03 wires `TOOLS["differentiators"]` and confirms 7-platform regression. Use `run_show()` for the 7 show-command DIFF tools and `get_object()` for DIFF-02 / DIFF-03 if a config-object endpoint is preferred. DIFF-09 aggregates 4 client.request calls (no helpers, just direct calls).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01:** All 9 DIFF tools are in scope for Phase 7 — DIFF-01 through DIFF-09. No trimming.
- DIFF-01: `aos8_get_md_hierarchy`
- DIFF-02: `aos8_get_effective_config`
- DIFF-03: `aos8_get_pending_changes`
- DIFF-04: `aos8_get_rf_neighbors`
- DIFF-05: `aos8_get_cluster_state`
- DIFF-06: `aos8_get_air_monitors`
- DIFF-07: `aos8_get_ap_wired_ports`
- DIFF-08: `aos8_get_ipsec_tunnels`
- DIFF-09: `aos8_get_md_health_check`

**D-02:** Single file: `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py`. A new `"differentiators"` key is added to `TOOLS` dict in `platforms/aos8/__init__.py`. Existing category keys are not modified.

**D-03:** Phase 4 CONTEXT.md design decisions for DIFF tools carry forward — DIFF-09 is multi-call aggregation (AP counts + client count + alarm summary + firmware version) for a `config_path` scope; DIFF-02 takes `object_name: str` + `config_path: str` (default `/md`) using config-object endpoint with show-command fallback; DIFF-03 uses config-object or dedicated endpoint with show-command fallback. For all DIFF tools: if no direct config-object endpoint, fallback to show command API.

**D-04:** 3 plans (TDD):
- **Plan 07-01 (Wave 0):** Write all DIFF tests + tool-layer token-leak security test — all RED. No implementation.
- **Plan 07-02 (Implementation):** Implement all 9 DIFF tools in `differentiators.py`. Tests go GREEN.
- **Plan 07-03 (Wiring + Regression):** Wire `TOOLS["differentiators"]` in `__init__.py`, run full suite, confirm all tests passing.

**D-05:** Add ONE cross-cutting tool-layer token-leak test (in `test_aos8_security.py` OR appended to `test_aos8_client.py`) that:
1. Invokes a read tool while capturing loguru output
2. Invokes a write tool (or simulates the write path) while capturing loguru output
3. Asserts `"UIDARUBA"` and the actual token value never appear in any captured log line
This complements the existing `test_uidaruba_never_in_logs` (client-level) by covering the tool invocation path.

**D-06:** No pytest-cov threshold added to pyproject.toml or CI. Phase 7 success = all tests pass. Coverage report can be added separately later if needed.

**D-07:** TEST-06 verified by running `pytest tests/unit` in full — if any of the 6 existing platform test suites break, the executor stops and reports the failure. No modifications to existing test files.

### Claude's Discretion

- Exact AOS8 REST endpoint paths or show commands for each DIFF tool (researcher to verify via Phase 4 CONTEXT.md canonical refs and API docs).
- JSON fixture file names and structure for DIFF test mocks (follow `tests/unit/fixtures/aos8/` pattern from Phase 3).
- Whether `config_path` is scope-sensitive for DIFF-04, DIFF-06, DIFF-07, DIFF-08 — researcher to confirm.
- File placement of the cross-cutting token-leak test (new `test_aos8_security.py` vs appended class in `test_aos8_client.py`) — researcher/planner to decide based on test count and coherence.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TEST-01 | Unit tests for `AOS8Client` covering login success/failure, token reuse, 401 refresh, `_global_result` error detection, `verify_ssl`, token masking | **ALREADY SATISFIED** — `tests/unit/test_aos8_client.py` exists with `TestAOS8ClientLogin`, `TestAOS8ClientLazy`, `TestAOS8ClientRequest`, `TestAOS8ClientLogging`, `TestAOS8ClientMisc`, `TestAOS8ClientHealth`, `TestAOS8ClientShutdown`. Researcher MUST verify in Plan 07-01 that all CLIENT-NN sub-cases are present; if any are missing, add them. Otherwise mark TEST-01 satisfied via existing tests. |
| TEST-02 | Unit test per read tool category using fixture JSON | **5 of 5 already exist** — `test_aos8_read_health.py`, `test_aos8_read_clients.py`, `test_aos8_read_alerts.py`, `test_aos8_read_wlan.py`, `test_aos8_read_troubleshooting.py`. Phase 7 adds the 6th: `test_aos8_read_differentiators.py` for the 9 DIFF tools. |
| TEST-03 | Unit tests for write tools — `aos8_write` tag, `aos8_write_delete` tag, `config_path` required (no default) | **ALREADY SATISFIED** — `tests/unit/test_aos8_write.py` covers all 12 write tools, `test_all_write_tools_carry_aos8_write_tag`, `test_manage_tools_carry_aos8_write_delete_tag`, `test_config_path_required_on_*` are present. Plan 07-01 verifies, no new code needed. |
| TEST-04 | Unit test that UIDARUBA token never appears in any log output across client and tool tests | **PARTIALLY SATISFIED** — `TestAOS8ClientLogging.test_uidaruba_never_in_logs` covers client layer. Phase 7 adds the **tool-layer** counterpart per D-05: invoke read tool + invoke write tool through full path, assert no token in logs. |
| TEST-05 | Unit test that platform auto-disables cleanly when secrets are absent | **VERIFY** — Likely covered by `tests/unit/test_aos8_config.py` (FOUND-02 was completed in Phase 1). Plan 07-01 confirms; if absent, add a one-line test that creates a `secrets_dir` missing `aos8_host` and asserts `load_config()` returns a `ServerConfig` with `aos8` disabled. |
| TEST-06 | All 6 existing platform tests (Mist, Central, GreenLake, ClearPass, Apstra, Axis) pass without modification | **VERIFY at gate** — Plan 07-03 runs full `pytest tests/unit` after wiring. 748 tests collect today; expect ~759 after Phase 7 with all passing. No modifications to existing platform test files allowed. |

**Gap Inventory:** Of the 6 TEST-NN requirements, 4 are already substantively satisfied by existing infrastructure. Phase 7's net-new test code is:
1. `tests/unit/test_aos8_read_differentiators.py` — ~9 DIFF tool tests + cross-cutting category check
2. Tool-layer token-leak test (new `test_aos8_security.py` OR appended class in `test_aos8_client.py`) — 1–2 tests
3. Verification scan of TEST-01/03/05 against existing tests during Plan 07-01 (no new tests unless gap found)
</phase_requirements>

## Standard Stack

### Core (already in pyproject.toml — no installs needed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| pytest | >= 8.4.0 | Test runner | Project standard; markers `unit` / `integration` |
| pytest-asyncio | >= 1.2.0 | async test support | `asyncio_mode = "auto"` already set; all DIFF tools are `async def` |
| pytest-cov | >= 7.0.0 | Coverage reports | Available; D-06 says no threshold enforcement, but `--cov` flag works on demand |
| httpx | >= 0.28.0 | HTTP client | `httpx.MockTransport` is the canonical async mock pattern in this repo (see `test_aos8_client.py::_install_mock_transport`) |
| loguru | >= 0.7.3 | Logging | Already wired into `loguru_capture` fixture |

### Mock-Building Patterns (already in conftest / test files)
| Pattern | Location | Purpose |
|---------|----------|---------|
| `_install_registry_stubs()` | `tests/conftest.py` | Stubs `_registry.mcp` so tool modules import cleanly without `register_tools()` being called |
| `loguru_capture` fixture | `tests/conftest.py` | Captures all loguru output as `list[str]` for token-leak assertions |
| `secrets_dir` fixture | `tests/conftest.py` | Tmpdir with all 7 platforms' example secrets — for FOUND-02 / TEST-05 |
| `_make_ctx(body)` helper | `test_aos8_read_health.py`, `test_aos8_write.py` | Builds `ctx.lifespan_context["aos8_client"]` with `AsyncMock` request returning a body — DIFF tests use this verbatim |
| `httpx.MockTransport` | `test_aos8_client.py::_install_mock_transport` | When the test must drive the **real** `AOS8Client` (e.g. for the tool-layer token-leak test that exercises the full request path) |

**No new dependencies. No installs. No config changes.** Phase 7 is pure test + tool code.

## Architecture Patterns

### Recommended Project Structure (delta from current)

```
src/hpe_networking_mcp/platforms/aos8/
├── __init__.py                # ADD: TOOLS["differentiators"] = [...9 names]
├── tools/
│   ├── differentiators.py     # NEW (~250–300 lines, well under 500)
│   └── ... (existing files unchanged)
└── ... (no other changes)

tests/unit/
├── test_aos8_read_differentiators.py    # NEW (~9 tests + 1 read-only smoke)
├── test_aos8_security.py                # NEW (preferred — see Pattern 3)
└── fixtures/aos8/
    ├── show_switch_hierarchy.json       # NEW — DIFF-01
    ├── show_running_config.json         # NEW — DIFF-02 fallback
    ├── show_pending_config.json         # NEW — DIFF-03
    ├── show_ap_arm_neighbors.json       # NEW — DIFF-04
    ├── show_lc_cluster_group.json       # NEW — DIFF-05
    ├── show_ap_monitor_active.json      # NEW — DIFF-06
    ├── show_ap_port_status.json         # NEW — DIFF-07
    ├── show_crypto_ipsec_sa.json        # NEW — DIFF-08
    └── (DIFF-09 reuses show_switches.json + show_ap_database.json + show_alarms.json + show_version.json — already exist)
```

### Pattern 1: DIFF tool implementation — show-command path

```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py:65-82
@tool(name="aos8_get_rf_neighbors", annotations=READ_ONLY)
async def aos8_get_rf_neighbors(
    ctx: Context,
    ap_name: str,
    config_path: str = "/md",
) -> dict[str, Any] | str:
    """Return ARM neighbor graph for an AP."""
    client = ctx.lifespan_context["aos8_client"]
    try:
        return await run_show(
            client,
            f"show ap arm-neighbors ap-name {ap_name}",
            config_path=config_path,
        )
    except Exception as exc:  # noqa: BLE001
        return format_aos8_error(exc, "fetch RF neighbors")
```

All 7 show-command-based DIFF tools (DIFF-01, 03, 04, 05, 06, 07, 08) follow this exact shape. Differences are only the show command string and any required selector parameter.

### Pattern 2: DIFF-09 multi-call aggregation

```python
# Pattern: 4 client.request calls in parallel via asyncio.gather, merge dicts
@tool(name="aos8_get_md_health_check", annotations=READ_ONLY)
async def aos8_get_md_health_check(
    ctx: Context,
    config_path: str,  # required, no default — D-05 from Phase 4
) -> dict[str, Any] | str:
    """Unified health report for an MD or scope."""
    client = ctx.lifespan_context["aos8_client"]
    try:
        # Mirror central get_site_health: top-level keyed sections.
        ap_active, ap_db, alarms, version = await asyncio.gather(
            run_show(client, "show ap active", config_path=config_path),
            run_show(client, "show ap database", config_path=config_path),
            run_show(client, "show alarms all", config_path=config_path),
            run_show(client, "show version", config_path=config_path),
            return_exceptions=True,  # one failed sub-call shouldn't kill the report
        )
        return {
            "config_path": config_path,
            "aps": {...},          # synthesized from ap_active + ap_db
            "clients": {...},      # from a 5th call to show user-table or similar — see test contract
            "alarms": {...},       # severity counts from alarms
            "firmware": {...},     # from version
        }
    except Exception as exc:
        return format_aos8_error(exc, "run MD health check")
```

**Test contract pins the exact show commands.** The Wave 0 test files in Phase 3 (e.g. `test_aos8_read_health.py:60-67`) have shown that we lock the params dict in `client.request.assert_awaited_once_with(...)` — for DIFF-09 the test will assert the four (or five) calls in order.

### Pattern 3: Tool-layer token-leak test (TEST-04)

**Recommendation: new `test_aos8_security.py`** rather than appending to `test_aos8_client.py`. Rationale: (a) the test exercises the *tool* path, not the client class; (b) keeps `test_aos8_client.py` focused on client unit tests; (c) gives a clean home for any future cross-cutting AOS8 security tests.

```python
# tests/unit/test_aos8_security.py  (NEW)
"""Cross-cutting AOS8 security tests at the tool layer (TEST-04 follow-up)."""
from __future__ import annotations
import httpx
import pytest
from hpe_networking_mcp.config import AOS8Secrets
from hpe_networking_mcp.platforms.aos8.client import AOS8Client

pytestmark = pytest.mark.unit

_LOGIN_OK = {"_global_result": {"status": "0", "UIDARUBA": "tok-supersecret-leak-bait-99"}}


def _make_secrets(**overrides):
    base = {"host": "conductor.test", "port": 4343, "username": "admin",
            "password": "secret", "verify_ssl": True}
    base.update(overrides)
    return AOS8Secrets(**base)


def _install(client, handler):
    transport = httpx.MockTransport(handler)
    client._http = httpx.AsyncClient(
        base_url=f"https://{client._config.host}:{client._config.port}",
        transport=transport, verify=client._config.verify_ssl,
    )


async def test_uidaruba_not_logged_during_read_tool(loguru_capture):
    """Drive aos8_get_version through real AOS8Client + MockTransport, assert token not leaked."""
    from fastmcp import Context  # noqa: F401  (sentinel, real ctx is mocked below)
    from unittest.mock import MagicMock
    from hpe_networking_mcp.platforms.aos8.tools.health import aos8_get_version

    client = AOS8Client(_make_secrets())
    def handler(req):
        if req.url.path == "/v1/api/login":
            return httpx.Response(200, json=_LOGIN_OK)
        return httpx.Response(200, json={"_global_result": {"status": "0"}, "Version": "8.10"})
    _install(client, handler)

    ctx = MagicMock()
    ctx.lifespan_context = {"aos8_client": client}
    await aos8_get_version(ctx)
    await client.aclose()

    joined = "\n".join(loguru_capture)
    assert "tok-supersecret-leak-bait-99" not in joined
    # NOTE: the literal string "UIDARUBA" DOES appear in legitimate logs (login info line)
    # — assert only the *value* leaks, not the parameter NAME. See Pitfall 2.


async def test_uidaruba_not_logged_during_write_tool(loguru_capture):
    """Drive aos8_manage_ssid_profile through real AOS8Client + MockTransport, assert no leak."""
    # ... mirror above with a POST handler returning write_ssid_prof_success.json shape
```

**Critical decision in test design:** D-05 says "asserts `"UIDARUBA"` and the actual token value never appear." Read this carefully — the parameter NAME `UIDARUBA=` *does* appear in `_sanitize_url_for_log` legitimately as `UIDARUBA=<redacted>`. The assertion must scan for the actual token *value* not the literal word "UIDARUBA". Otherwise the test will fail against legitimate redacted logging. **Recommend updating the test contract:** assert the token value never appears AND assert any line containing "UIDARUBA" also contains "<redacted>". Flag this for the planner.

### Pattern 4: Wave 0 RED test scaffold

Mirror `test_aos8_write.py` exactly:
```python
"""Red tests for AOS8 differentiator read tools (DIFF-01..09).

These tests fail with ModuleNotFoundError until plan 07-02 implements
``platforms.aos8.tools.differentiators``. Each test pins the exact
endpoint path, parameter shape, and response contract every implementation
must satisfy.
"""
from __future__ import annotations
import json
from pathlib import Path
from unittest.mock import AsyncMock, MagicMock
import httpx
import pytest

pytestmark = pytest.mark.unit
_FIXTURES = Path(__file__).parent / "fixtures" / "aos8"

# ... copy _load_fixture, _make_ctx from test_aos8_read_health.py
```

### Anti-Patterns to Avoid

- **Don't write integration tests against a live Conductor.** Project constraint: no live AOS8 system available. Use `httpx.MockTransport` (full-stack) or `MagicMock(spec=httpx.Response)` (tool layer).
- **Don't modify existing platform test files** (D-07). TEST-06 fails if we touch `test_mist_*`, `test_central_*`, `test_greenlake_*`, `test_clearpass_*`, `test_apstra_*`, `test_axis_*`.
- **Don't add a `requires_write_memory_for` field** to DIFF tools — they are READ-only. Only the `writes.py` tools carry that response contract.
- **Don't import `differentiators.py` from `__init__.py` directly.** The importlib loop in `register_tools()` (line 87–89 of `__init__.py`) handles new modules automatically once the `TOOLS["differentiators"]` key exists.
- **Don't assert on the literal string `"UIDARUBA"` in logs** — see Pattern 3 critical note. The parameter name appears in legitimate redacted log lines.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Mock async httpx call at tool layer | Custom async mock class | `MagicMock` + `AsyncMock` per `_make_ctx` pattern | Already proven across 5 read-tool test files |
| Mock async httpx call at client layer | `aiohttp` mocks, `respx`, custom transport | `httpx.MockTransport(handler)` | Already in `test_aos8_client.py::_install_mock_transport` — exact pattern reused |
| Capture loguru output | `caplog`, custom handler | `loguru_capture` fixture | conftest.py provides it |
| Stub `_registry.mcp` for tool imports | Per-test patching | `_install_registry_stubs()` runs at conftest load | Idempotent, free for new test files |
| Build error responses for DIFF tools | Custom error handling | `format_aos8_error(exc, action)` from `_helpers.py` | Same shape as health/clients/alerts/wlan/troubleshooting |
| Strip `_meta` from show responses | Manual dict comprehension | `strip_meta()` from `_helpers.py` | Already covers DIFF needs |
| Issue show command from a tool | Direct `client.request()` | `run_show(client, command, *, config_path=...)` | Established in `_helpers.py`; tests assert this exact path |

**Key insight:** Phase 3 (and Phase 5) built every helper Phase 7 needs. The DIFF tool implementation is mostly mechanical pattern matching — the work is verifying the show commands and writing fixtures.

## Common Pitfalls

### Pitfall 1: Token-leak test asserts on literal "UIDARUBA" string
**What goes wrong:** Assertion `assert "UIDARUBA" not in joined` fails because legitimate redacted log lines contain `UIDARUBA=<redacted>`.
**Why it happens:** D-05 wording is "asserts `"UIDARUBA"` and the actual token value never appear" — but the parameter name appears legitimately after sanitization.
**How to avoid:** Test asserts (a) actual token value not in logs, (b) any line containing "UIDARUBA" must also contain "<redacted>". Flag this in the plan and confirm with user during Plan 07-01.
**Warning signs:** Existing client-level test at `test_aos8_client.py:206-207` only asserts on the token value, not the parameter name — follow that precedent.

### Pitfall 2: `_make_ctx` body must include `_global_result.status="0"`
**What goes wrong:** Test fixtures missing `_global_result.status` key cause `AOS8Client._check_global_result` to silently pass — but tool tests use `MagicMock(spec=httpx.Response)`, where `_check_global_result` isn't invoked. Mismatch can hide future regressions.
**Why it happens:** Tool-layer tests bypass the client; fixtures should still mimic real responses for documentation value.
**How to avoid:** Always include `_global_result: {"status": "0"}` in fixture JSON files. The existing `show_*.json` files do this; new DIFF fixtures must too.

### Pitfall 3: DIFF-09 `asyncio.gather(..., return_exceptions=True)` swallows errors silently
**What goes wrong:** A single failing sub-call (e.g., wrong show command) returns an `Exception` object instead of data; tool returns garbage dict, no error visible.
**Why it happens:** `return_exceptions=True` is needed so one degraded MD doesn't fail the whole health check, but each sub-result must be type-checked.
**How to avoid:** After `gather`, inspect each result; if `isinstance(r, Exception)`, log a warning and put `{"error": format_aos8_error(r, "...")}` into that section of the response. Test must cover the "one sub-call failed" case.

### Pitfall 4: New DIFF fixtures break existing tests via path collision
**What goes wrong:** A fixture filename collides with an existing one (e.g. naming `show_version.json` for DIFF-09 when it already exists for the client health check).
**Why it happens:** Phase 3 / Phase 5 already populated `tests/unit/fixtures/aos8/` with 10 files; DIFF-09 reuses `show_switches.json`, `show_ap_database.json`, `show_alarms.json`, plus needs `show_user_table.json` (already exists) and `show_version.json` (used by `test_aos8_client.py::TestAOS8ClientHealth` as `aos8_show_version.json` at the parent fixtures dir, NOT inside `aos8/`).
**How to avoid:** Inventory existing fixtures (this RESEARCH lists them above); reuse where possible; never overwrite. Wave 0 test contract specifies fixture filenames so Plan 07-02 has zero ambiguity.

### Pitfall 5: `register_tools()` count check fails after wiring
**What goes wrong:** `test_aos8_init.py` hard-codes `EXPECTED_TOTAL = 38` (26 read + 12 write). After Plan 07-03 it must be 47 (26 + 9 + 12).
**Why it happens:** Wiring change without test update.
**How to avoid:** Plan 07-01 updates `EXPECTED_TOOLS` in `test_aos8_init.py` to add the 9 DIFF tool names AND updates `EXPECTED_CATEGORY_COUNTS` to add `"differentiators": 9` AND updates `EXPECTED_TOTAL`. This is a deliberate edit — D-07 forbids modifying *other platform* test files, but `test_aos8_init.py` is the AOS8 wiring test and MUST evolve with each phase.

### Pitfall 6: ruff line-length / file-size violations land in CI
**What goes wrong:** `differentiators.py` exceeds 500 lines, or a docstring/signature exceeds 120 chars.
**Why it happens:** Aggregating 9 tools with full Google-style docstrings and Pydantic Field descriptions.
**How to avoid:** Phase 4 estimate was 250–300 lines; budget for that. Use `Annotated[str, Field(description=...)]` aliases as `writes.py` did to keep signatures tight (writes.py:42-45 pattern).

## Code Examples

### Existing canonical AOS8 read-tool pattern (DIFF tools mirror this)
```python
# Source: src/hpe_networking_mcp/platforms/aos8/tools/health.py:25-42
@tool(name="aos8_get_controllers", annotations=READ_ONLY)
async def aos8_get_controllers(ctx: Context) -> dict[str, Any] | str:
    """Return the list of controllers known to the Mobility Conductor."""
    client = ctx.lifespan_context["aos8_client"]
    try:
        return await run_show(client, "show switches")
    except Exception as exc:  # noqa: BLE001
        return format_aos8_error(exc, "list controllers")
```

### Existing canonical Wave 0 test pattern (DIFF tests mirror this)
```python
# Source: tests/unit/test_aos8_read_health.py:38-53
async def test_get_controllers():
    from hpe_networking_mcp.platforms.aos8.tools.health import aos8_get_controllers

    body = _load("show_switches.json")
    ctx, client = _make_ctx(body)
    result = await aos8_get_controllers(ctx)

    client.request.assert_awaited_once_with(
        "GET",
        "/v1/configuration/showcommand",
        params={"command": "show switches"},
    )
    assert "_meta" not in result
    assert "_global_result" not in result
    assert result["Switch List"][0]["Name"] == "MD-01"
```

### Existing canonical token-leak test (extend to tool layer for TEST-04)
```python
# Source: tests/unit/test_aos8_client.py:187-207
class TestAOS8ClientLogging:
    async def test_uidaruba_never_in_logs(self, loguru_capture):
        client = AOS8Client(_make_secrets())
        # ... mock transport with login + 401-then-200 ...
        await client.request("GET", "/v1/configuration/object/foo")
        await client.aclose()
        joined = "\n".join(loguru_capture)
        assert "tok-supersecret-12345" not in joined, (
            f"UIDARUBA token leaked into log output: {joined!r}"
        )
```

The Phase 7 tool-layer counterpart drives a *tool function* (e.g. `aos8_get_version(ctx)`) instead of `client.request()`, while keeping the same `httpx.MockTransport` setup so the full client→tool path is exercised.

## Runtime State Inventory

This phase is greenfield code addition (new tools + new tests). No rename / refactor / migration. **Section omitted by trigger rule.**

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|-------------|-----------|---------|----------|
| Python | All tests | ✓ | 3.12+ (uv) | — |
| pytest | Test runner | ✓ | >= 8.4.0 (in pyproject.toml) | — |
| pytest-asyncio | Async DIFF tool tests | ✓ | >= 1.2.0 | — |
| pytest-cov | Optional coverage | ✓ | >= 7.0.0 | — |
| httpx | Mock transports | ✓ | >= 0.28.0 | — |
| loguru | `loguru_capture` fixture | ✓ | >= 0.7.3 | — |
| Live AOS8 Conductor | NONE — explicit project constraint | ✗ | — | All tests use mocked HTTP responses (constraint, not gap) |

**No missing dependencies.** Phase 7 has zero external runtime requirements beyond the existing test stack.

## Validation Architecture

`workflow.nyquist_validation` is `true` in `.planning/config.json` — section included.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest >= 8.4.0 (with pytest-asyncio >= 1.2.0, asyncio_mode="auto") |
| Config file | `pyproject.toml` `[tool.pytest.ini_options]` (markers: `unit`, `integration`; `testpaths=["tests"]`) |
| Quick run command | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py -x` |
| Full suite command | `cd hpe-networking-mcp && uv run pytest tests/unit -q` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|--------------|
| TEST-01 | AOS8Client login/refresh/global_result/ssl/token mask | unit | `uv run pytest tests/unit/test_aos8_client.py -x` | ✅ |
| TEST-02 (existing) | Read tool category coverage (5 categories) | unit | `uv run pytest tests/unit/test_aos8_read_health.py tests/unit/test_aos8_read_clients.py tests/unit/test_aos8_read_alerts.py tests/unit/test_aos8_read_wlan.py tests/unit/test_aos8_read_troubleshooting.py -x` | ✅ |
| TEST-02 (DIFF) | DIFF-01..09 tool category coverage | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py -x` | ❌ Wave 0 |
| TEST-03 | Write tools — tags, config_path requirement, response shape | unit | `uv run pytest tests/unit/test_aos8_write.py -x` | ✅ |
| TEST-04 (client) | UIDARUBA never in client logs | unit | `uv run pytest tests/unit/test_aos8_client.py::TestAOS8ClientLogging -x` | ✅ |
| TEST-04 (tool) | UIDARUBA never in logs across tool invocation | unit | `uv run pytest tests/unit/test_aos8_security.py -x` | ❌ Wave 0 |
| TEST-05 | Platform auto-disables when secrets absent | unit | `uv run pytest tests/unit/test_aos8_config.py -x` | ✅ (verify in Plan 07-01) |
| TEST-06 | Full 7-platform suite green | regression | `uv run pytest tests/unit -q` | gate at end of Plan 07-03 |
| DIFF-01..09 | New AOS8 differentiator tools | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py -x` | ❌ Wave 0 |
| Wiring | TOOLS["differentiators"] registered, count = 47 | unit | `uv run pytest tests/unit/test_aos8_init.py -x` | ✅ (must update EXPECTED_* constants in Plan 07-01) |

### Sampling Rate
- **Per task commit:** `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py tests/unit/test_aos8_security.py tests/unit/test_aos8_init.py -x` (~5 sec)
- **Per wave merge:** `cd hpe-networking-mcp && uv run pytest tests/unit/ -k aos8 -q` (~10 sec)
- **Phase gate:** `cd hpe-networking-mcp && uv run pytest tests/unit -q` — must pass 100%, expect ~759 tests collected (748 today + 9 DIFF + 2 security)

### Wave 0 Gaps
- [ ] `tests/unit/test_aos8_read_differentiators.py` — covers DIFF-01..09 (TEST-02 extension)
- [ ] `tests/unit/test_aos8_security.py` — covers tool-layer TEST-04
- [ ] Update `tests/unit/test_aos8_init.py` constants: add 9 DIFF tool names to `EXPECTED_TOOLS` (or new `EXPECTED_DIFF_TOOLS`), add `"differentiators": 9` to `EXPECTED_CATEGORY_COUNTS`, set `EXPECTED_TOTAL = 47`
- [ ] Fixture JSON files for DIFF-01..08 in `tests/unit/fixtures/aos8/` (DIFF-09 reuses existing fixtures)
- [ ] Verify TEST-05 covered by `test_aos8_config.py` — if missing, add a one-line auto-disable test

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Mock httpx via `aiohttp`-style mocks | `httpx.MockTransport` for full client tests | httpx 0.18+ | Simpler, no extra deps; already in use |
| Stub registry per-test | `conftest.py` `_install_registry_stubs()` runs at module load | This project's #155 fix | Idempotent, transparent to test authors |
| Coverage threshold enforcement | Project decision: NO threshold (D-06) | Phase 7 | Coverage is observable on demand via `--cov`, not a CI gate |

**Deprecated/outdated:**
- pytest's `caplog` for loguru — does NOT capture loguru output (loguru bypasses stdlib logging). Use the `loguru_capture` fixture instead. (Already established in this repo.)

## Open Questions

1. **Which AOS8 show commands resolve to which DIFF tool?**
   - What we know: Phase 4 CONTEXT.md decided show-command fallback for DIFF-02 / DIFF-03; the other 7 DIFFs are show-command first.
   - What's unclear: exact command strings (e.g., `show ap arm-neighbors` vs `show ap arm neighbors` vs `show arm`). Wave 0 tests must pin one — but the contract is per-test, so a planning iteration with the user can adjust.
   - Recommendation: Plan 07-01 documents proposed commands inline in test scaffolds. If Plan 07-02 finds AOS8 doesn't accept the literal string, the test contract changes (one-line edit) and implementation matches.

2. **Should TEST-05 (auto-disable on missing secrets) live in `test_aos8_config.py` already?**
   - What we know: FOUND-02 was completed in Phase 1 and Phase 1 had its own test additions.
   - What's unclear: whether a precise auto-disable test exists or coverage is implicit.
   - Recommendation: Plan 07-01 inspects `test_aos8_config.py` and explicitly marks TEST-05 satisfied or adds a 5-line test.

3. **Token-leak test — placement decision.**
   - What we know: D-05 leaves placement to discretion: new `test_aos8_security.py` vs appended class in `test_aos8_client.py`.
   - Recommendation (in this RESEARCH): new `test_aos8_security.py` because the test exercises tool-layer behavior, not client class behavior. Keeps `test_aos8_client.py` semantically coherent. Planner confirms.

## Project Constraints (from CLAUDE.md)

- **API methods:** GET and POST only — no PUT/PATCH/DELETE. (Tests must not assert other methods.)
- **Auth:** UIDARUBA reused across calls — verify in tool-layer test that ONE login + many tool calls = one `/v1/api/login` request.
- **Testing:** No live AOS8 system. All Phase 7 tests use `MagicMock`+`AsyncMock` (tool layer) or `httpx.MockTransport` (client layer).
- **Compatibility:** Must not break existing platform tests. (TEST-06 gate.)
- **Style:** ruff (line-length 120, double quotes, trailing commas), mypy (Python 3.12, untyped defs allowed), max 500 lines/file, 50 lines/function, Google-style docstrings.
- **Write Memory:** DIFF tools are READ tools — no `requires_write_memory_for` field. (Avoid copy-paste from `writes.py`.)
- **SSL:** Not relevant for Phase 7 — already validated in Phase 2 client tests.
- **Pre-push checklist:** `uv run ruff check .`, `uv run ruff format --check .`, `uv run mypy src/ --ignore-missing-imports`, `uv run pytest tests/ -q`. Plan 07-03 runs all four.
- **Documentation Checklist:** README/CHANGELOG/INSTRUCTIONS/TOOLS.md/pyproject.toml — Phase 6 already updated for AOS8 at version 2.4.0.0; Phase 7 should bump version (e.g. 2.5.0.0) and add a CHANGELOG entry for the 9 new DIFF tools and the new test files. Update tool count in README from "38" → "47" and TOOLS.md tool count similarly. **This is mandated by CLAUDE.md and is NOT optional even though D-06 says no coverage thresholds.**

## Sources

### Primary (HIGH confidence)
- `.planning/phases/07-testing-integration/07-CONTEXT.md` — Phase 7 user decisions (D-01 through D-07)
- `.planning/phases/04-differentiator-tools/04-CONTEXT.md` — Phase 4 design decisions for DIFF-01..09 (still authoritative)
- `.planning/REQUIREMENTS.md` — TEST-01..06 specifications
- `.planning/STATE.md` — current state (Phase 6 complete, 767 tests at last run; current collect shows 748 — minor drift, planner verifies)
- `hpe-networking-mcp/CLAUDE.md` — project standards (line length 120, file ≤500 lines, no live system, etc.)
- `hpe-networking-mcp/pyproject.toml` — `[tool.pytest.ini_options]`, `asyncio_mode = "auto"`, marker definitions
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/health.py` — canonical read-tool implementation pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` — `run_show`, `get_object`, `strip_meta`, `format_aos8_error`
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — TOOLS dict + register_tools importlib loop
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` — AOS8Client request semantics, sanitization
- `hpe-networking-mcp/tests/unit/test_aos8_read_health.py` — canonical Wave 0 RED test scaffold
- `hpe-networking-mcp/tests/unit/test_aos8_client.py` — `TestAOS8ClientLogging.test_uidaruba_never_in_logs` precedent
- `hpe-networking-mcp/tests/unit/test_aos8_write.py` — write-tool test patterns + tag enforcement tests (TEST-03 reference)
- `hpe-networking-mcp/tests/unit/test_aos8_init.py` — wiring/total-count test (must be updated in Plan 07-01)
- `hpe-networking-mcp/tests/conftest.py` — `loguru_capture`, `secrets_dir`, `_install_registry_stubs`

### Secondary (MEDIUM confidence)
- Current `pytest tests/unit --collect-only` count: 748 tests collected (verified by execution; STATE.md says 767 from a prior run — drift is most likely from earlier test churn during Phase 5/6, not a regression. Planner verifies the baseline before Phase 7).

### Tertiary (LOW confidence)
- Exact AOS8 show command strings for DIFFs (`show ap arm-neighbors`, `show lc-cluster group-membership`, etc.) — these come from operator/AOS8 API knowledge, not from a fetched authoritative source in this research session. The planner / executor verifies against AOS8 documentation when implementing each DIFF; the test contract pins whichever string is chosen.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — every tool, fixture, and pattern already exists in the repo.
- Architecture: HIGH — Phase 3/5 patterns are directly applicable; only thing changing is the category name.
- Pitfalls: HIGH — Pitfalls 1, 5, 6 are observable in existing test files; Pitfalls 2-4 are derived from the existing helper/code semantics.
- DIFF show-command strings: LOW — these were left to "Claude's discretion" in Phase 4; planner/executor verifies during implementation.

**Research date:** 2026-04-28
**Valid until:** 2026-05-28 (30 days; project is stable, no fast-moving deps)
