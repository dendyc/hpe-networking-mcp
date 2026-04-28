# Phase 3: Read Tools - Research

**Researched:** 2026-04-28
**Domain:** AOS8 / Mobility Conductor REST API read tools (showcommand passthrough + configuration object reads, surfaced as MCP tools via per-platform `tool()` shim)
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**File Organization:**
- **D-01:** 5 category files, one per functional area: `health.py`, `clients.py`, `alerts.py`, `wlan.py`, `troubleshooting.py`. Mirrors Central's per-category layout. Each file stays under 500 lines.
- **D-02:** File placement: `src/hpe_networking_mcp/platforms/aos8/tools/<category>.py`. Import via the existing `_registry.tool()` shim pattern (same as all other platforms).

**config_path Parameter:**
- **D-03:** `config_path` is **selective** — only tools that are genuinely scope-sensitive accept it. Default `/md` for those that do.
- **D-04:** Tools that accept `config_path` (scope-sensitive):
  - All `wlan.py` tools (SSID profiles, virtual APs, AP groups, user roles — config lives under a hierarchy node)
  - `aos8_get_ap_database` (READ-02), `aos8_get_active_aps` (READ-03), `aos8_get_ap_detail` (READ-04), `aos8_get_bss_table` (READ-05), `aos8_get_radio_summary` (READ-06) — AP state is per-MD/AP-group scope
  - `aos8_get_clients` (READ-09), `aos8_find_client` (READ-10), `aos8_get_client_detail` (READ-11) — client visibility is scope-dependent
  - `aos8_get_alarms` (READ-13), `aos8_get_events` (READ-15) — alerting has config_path scope
- **D-05:** Tools that do NOT accept `config_path` (Conductor-root queries):
  - `aos8_get_controllers` (READ-01) — always queries the full Conductor inventory
  - `aos8_get_version` (READ-07), `aos8_get_licenses` (READ-08) — Conductor-level, no per-MD scope
  - `aos8_get_client_history` (READ-12) — client-specific lookup, not scope-filtered
  - `aos8_get_audit_trail` (READ-14) — Conductor-wide audit log
  - `aos8_ping` (READ-20), `aos8_traceroute` (READ-21) — target-IP operations, no config scope
  - `aos8_get_logs` (READ-23), `aos8_get_controller_stats` (READ-24), `aos8_get_arm_history` (READ-25), `aos8_get_rf_monitor` (READ-26) — researcher to confirm appropriate scope; default to Conductor-wide unless API requires config_path
  - `aos8_show_command` (READ-22) — passthrough; command string encodes any needed scope

**show_command Passthrough Design (READ-22):**
- **D-06:** `aos8_show_command` accepts any `command` string but enforces it starts with `"show "` (case-insensitive). Any command not matching this prefix is rejected with a clear error message — prevents accidental config/write commands.
- **D-07:** `_meta` field is stripped from the response before returning. Where the response is structured JSON, it is returned as-is (minus `_meta`). Where it is a plain string, return it wrapped in `{"output": "..."}`.
- **D-08:** No allowlist — the `show ` prefix check is the only guard. Keeps the tool maximally useful for AI troubleshooting workflows while blocking obvious misuse.

**Plan Sequencing / TDD:**
- **D-09:** Wave 0 scaffold first: Plan 1 writes all failing tests across all 26 tools (5 test files, one per category) before any implementation.
- **D-10:** Plans 2–6 implement one category each: health, clients, alerts, wlan, troubleshooting. Final plan wires `build_meta_tools` and populates `TOOLS` dict.

### Claude's Discretion
- Exact AOS8 REST endpoint paths per tool (researcher to verify against AOS8 API docs and Phase 2 RESEARCH.md)
- JSON fixture file names and structure for test mocks
- Which `aos8_get_logs`/`aos8_get_controller_stats`/`aos8_get_arm_history`/`aos8_get_rf_monitor` tools need `config_path` — researcher to confirm based on actual API endpoints
- Whether `aos8_get_client_history` uses a show command or a config object endpoint

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Tool | config_path? | API Endpoint Strategy | Research Support |
|----|------|--------------|------------------------|------------------|
| READ-01 | `aos8_get_controllers` | No | `GET /v1/configuration/showcommand?command=show+switches` | showcommand returns structured JSON with controller table |
| READ-02 | `aos8_get_ap_database` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+ap+database&config_path=<p>` | AP database is per-MD scope; showcommand accepts config_path |
| READ-03 | `aos8_get_active_aps` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+ap+active` | "show ap active" returns currently up APs |
| READ-04 | `aos8_get_ap_detail` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+ap+details+ap-name+<n>` (or `ap-mac <m>`) | "show ap details" with name or MAC selector |
| READ-05 | `aos8_get_bss_table` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+ap+bss-table` | "show ap bss-table" lists radio/BSSID associations |
| READ-06 | `aos8_get_radio_summary` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+ap+radio-summary` | Per-AP radio state |
| READ-07 | `aos8_get_version` | No | `GET /v1/configuration/showcommand?command=show+version` | Already used by `health_check()` |
| READ-08 | `aos8_get_licenses` | No | `GET /v1/configuration/showcommand?command=show+license` | License list as structured JSON |
| READ-09 | `aos8_get_clients` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+user-table` | "show user-table" lists wireless clients |
| READ-10 | `aos8_find_client` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+user-table+mac+<m>` (or `ip <ip>` or `name <u>`) | Client lookup by MAC/IP/username |
| READ-11 | `aos8_get_client_detail` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+user-table+verbose+mac+<m>` | Verbose client info |
| READ-12 | `aos8_get_client_history` | No | `GET /v1/configuration/showcommand?command=show+ap+association+history+client-mac+<m>` | Connection history |
| READ-13 | `aos8_get_alarms` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+alarms` | Active alarms |
| READ-14 | `aos8_get_audit_trail` | No | `GET /v1/configuration/showcommand?command=show+audit-trail` | Conductor-wide audit log |
| READ-15 | `aos8_get_events` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+events` | System event log |
| READ-16 | `aos8_get_ssid_profiles` | Yes (`/md`) | `GET /v1/configuration/object/ssid_prof?config_path=<p>` | Vendor docs confirm `/object/ssid_prof` endpoint |
| READ-17 | `aos8_get_virtual_aps` | Yes (`/md`) | `GET /v1/configuration/object/virtual_ap?config_path=<p>` | Same object pattern |
| READ-18 | `aos8_get_ap_groups` | Yes (`/md`) | `GET /v1/configuration/object/ap_group?config_path=<p>` | Same object pattern |
| READ-19 | `aos8_get_user_roles` | Yes (`/md`) | `GET /v1/configuration/object/role?config_path=<p>` | Same object pattern |
| READ-20 | `aos8_ping` | No | `GET /v1/configuration/showcommand?command=ping+<dest>` | "ping" works through showcommand even though not a "show" command on the wire — AOS8 wraps `ping` in showcommand JSON. **Bypass D-06 prefix check internally** since the tool itself constructs the command string. |
| READ-21 | `aos8_traceroute` | No | `GET /v1/configuration/showcommand?command=traceroute+<dest>` | Same wrapper pattern |
| READ-22 | `aos8_show_command` | No | `GET /v1/configuration/showcommand?command=<arbitrary>` | Passthrough; enforces `show ` prefix per D-06; strips `_meta` per D-07 |
| READ-23 | `aos8_get_logs` | No | `GET /v1/configuration/showcommand?command=show+log+system+<n>` | System log; `n` = lines |
| READ-24 | `aos8_get_controller_stats` | No | `GET /v1/configuration/showcommand?command=show+cpuload` + `show+memory` aggregation | Multiple show commands aggregated |
| READ-25 | `aos8_get_arm_history` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+ap+arm+history` | ARM channel/power history |
| READ-26 | `aos8_get_rf_monitor` | Yes (`/md`) | `GET /v1/configuration/showcommand?command=show+ap+monitor+stats` | RF monitor state |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

These directives are extracted from `./CLAUDE.md` and `hpe-networking-mcp/CLAUDE.md`:

- **GSD workflow enforcement:** Direct repo edits forbidden outside a GSD command flow.
- **API:** GET and POST only — no PUT/PATCH/DELETE on AOS8. All read tools use GET.
- **Auth:** UIDARUBA reused across calls (handled inside `AOS8Client.request()`; tools never touch tokens directly).
- **Testing:** No live AOS8 — all tests use mocked HTTP responses (`httpx.MockTransport`, same as Phase 2).
- **Compatibility:** Existing 645-test suite must remain green.
- **Style:** Ruff lint + format, mypy clean, **max 500 lines per file**, max 50 lines per function, max 100 lines per class, Google-style docstrings.
- **Line length:** 120 chars; double quotes; trailing commas in multi-line.
- **Naming:** `snake_case` for tool functions (`aos8_get_controllers`); explicit `name=` kwarg on `@tool()` decorator.
- **Logging:** All output to stderr (loguru); never log UIDARUBA (handled by `AOS8Client._sanitize_url_for_log`).
- **TDD:** Tests first (Wave 0 / Plan 1), then implementation per category.
- **Pre-push gate:** `docker compose ... uv run ruff check && ruff format --check && mypy src/ && pytest tests/ -q`
- **Commit messages:** No "claude code" or "written by claude code".
- **Documentation:** Tool changes update README/INSTRUCTIONS/TOOLS.md/CHANGELOG IN THE SAME PR — but per `<deferred>` in CONTEXT, docs land in Phase 6, not Phase 3.

## Summary

Phase 3 is a **mechanical port** of the established platform tool pattern (canonical reference: `central/tools/{clients,devices,alerts,events,troubleshooting}.py`) onto the AOS8 surface. Every tool reduces to `await client.request("GET", "/v1/configuration/showcommand", params={"command": "...", "config_path": "..."})` followed by JSON cleanup. There is no new SDK to wrap, no auth to design, no novel error handling: `AOS8Client` from Phase 2 already enforces UIDARUBA injection, 401-refresh, `_global_result` checking, and URL sanitization.

The work is volume, not complexity: 26 tools across 5 files, each ~10-30 lines after Google-docstring inflation. The five files comfortably stay under the 500-line limit (estimated 200-400 lines each, with the largest being `troubleshooting.py` because of the show-command passthrough, ping, and traceroute helpers).

The two non-obvious risks: (1) **`_meta` stripping** must be applied uniformly across every showcommand response — not just `aos8_show_command` — because every showcommand result includes `_meta` describing the column schema, and AI clients don't need it cluttering the result. Phase 3 should add a small `_strip_meta(body)` helper in a new `tools/_helpers.py` and route every showcommand response through it. (2) **`_global_result` is already stripped at the client layer** by `_check_global_result` (it's checked, not removed), so tools may need to remove it after-the-fact too if it's noisy in tool output.

The chief discretion items are now resolved:
- `aos8_get_logs/_controller_stats/_arm_history/_rf_monitor` — only ARM history and RF monitor genuinely need `config_path` (they're per-MD AP state); logs and controller stats are typically Conductor-level. Final mapping locked in `<phase_requirements>` table above.
- `aos8_get_client_history` — uses a show command (`show ap association history client-mac <m>`), not a config object endpoint. No `config_path` because the MAC address uniquely identifies the client globally.

**Primary recommendation:** Build a thin `_helpers.py` with `_strip_meta(body)`, `_run_show(client, command, config_path=None)`, and `_format_showcommand_error(exc)` helpers; have every tool call those helpers; mirror Central's category file shape; and ensure the `troubleshooting.py` show-command passthrough strictly enforces the `"show "` prefix per D-06.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `fastmcp` | `>= 3.1.1` (already pinned) | `Context` injection into every tool function; `ToolAnnotations` for READ_ONLY marker | Used by every existing tool. |
| `httpx` | `>= 0.28.0` (already pinned) | Underlying HTTP via `AOS8Client.request()` (which returns `httpx.Response`) | Already wired in Phase 2; tools call `.json()` on the response. |
| `loguru` | `>= 0.7.3` (already pinned) | Optional structured logging within tool helpers | Project standard. |
| `pytest` + `pytest-asyncio` | `>= 8.4.0` / `>= 1.2.0` | `@pytest.mark.unit` async tests with mocked `AOS8Client` | Same fixtures as Phase 2 tests. |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `httpx.MockTransport` | (built into httpx) | Drive `AOS8Client._http` with deterministic responses in tests | Mirror Phase 2 `_install_mock_transport()` pattern verbatim |
| `unittest.mock.AsyncMock` | (stdlib) | Alternative — mock `AOS8Client.request()` directly without httpx layer | Simpler for tool-level tests where HTTP details are irrelevant; preferred |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Per-tool `try/except` blocks | A single decorator wrapping the tool body | Decorator hides control flow and breaks Pydantic schema introspection used by `_get_tool_schema`. **Reject.** |
| Pydantic models for response shapes | Raw dict returns | Central uses Pydantic models (`Client`, `Device`, `Alert`); they help typed clients but require model definitions per tool. Phase 3 returns raw dicts — schemas vary across firmware and aren't worth pinning down for v1. **Accept raw dict[str, Any]**. Models are deferred to a future iteration. |
| Mocking `httpx.MockTransport` | Mocking `AOS8Client.request` directly via `AsyncMock` | MockTransport tests the full HTTP path including UIDARUBA injection, which is already proven in Phase 2. Tool tests should focus on shape transformation. **Use AsyncMock for tool tests; reserve MockTransport for client-level tests.** |
| New `aos8_show_command` allowlist | `show ` prefix check only | Allowlist is brittle; show commands are sprawling. CONTEXT D-08 explicitly rejects allowlist. **Prefix check only.** |

**Installation:** No new packages required. All deps already in `pyproject.toml` and `uv.lock`.

**Version verification:** Skipped — no new deps. Existing pins inherited from prior phases.

## Architecture Patterns

### Recommended Project Structure (added by this phase)

```
src/hpe_networking_mcp/platforms/aos8/
├── __init__.py            # MODIFY: populate TOOLS dict, call build_meta_tools()
├── _registry.py           # UNCHANGED (Phase 2)
├── client.py              # UNCHANGED (Phase 2)
└── tools/
    ├── __init__.py        # NEW: exports READ_ONLY annotations constant
    ├── _helpers.py        # NEW: _strip_meta(), _run_show(), _format_error()
    ├── health.py          # NEW: READ-01..08 (8 tools)
    ├── clients.py         # NEW: READ-09..12 (4 tools)
    ├── alerts.py          # NEW: READ-13..15 (3 tools)
    ├── wlan.py            # NEW: READ-16..19 (4 tools)
    └── troubleshooting.py # NEW: READ-20..26 (7 tools)
tests/unit/
├── fixtures/aos8/         # NEW: per-category JSON response fixtures
│   ├── show_switches.json
│   ├── show_ap_database.json
│   ├── show_user_table.json
│   ├── show_alarms.json
│   ├── ssid_prof.json
│   ├── show_log_system.json
│   └── ... (one per tool, named after the show command or object path)
├── test_aos8_read_health.py        # NEW: covers READ-01..08
├── test_aos8_read_clients.py       # NEW: covers READ-09..12
├── test_aos8_read_alerts.py        # NEW: covers READ-13..15
├── test_aos8_read_wlan.py          # NEW: covers READ-16..19
└── test_aos8_read_troubleshooting.py  # NEW: covers READ-20..26
```

**Line budget per file** (estimates; recheck if any approaches 500):
- `health.py`: ~340 lines (8 tools × ~40 lines including docstrings)
- `clients.py`: ~190 lines (4 tools)
- `alerts.py`: ~150 lines (3 tools)
- `wlan.py`: ~190 lines (4 tools)
- `troubleshooting.py`: ~360 lines (7 tools, includes show-command passthrough validation)
- `_helpers.py`: ~80 lines

### Pattern 1: Category file structure (mirror Central)

**What:** Each category file imports the `_registry.tool` shim, the `READ_ONLY` annotations, and the `_helpers` module. Each tool is an `async def aos8_<name>(ctx: Context, ...)` decorated with `@tool(name="aos8_<name>", annotations=READ_ONLY)`.

**Source:** `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/clients.py:28-93`

**Example:**
```python
# src/hpe_networking_mcp/platforms/aos8/tools/health.py
"""AOS8 health and inventory read tools."""

from __future__ import annotations

from typing import Any

from fastmcp import Context

from hpe_networking_mcp.platforms.aos8._registry import tool
from hpe_networking_mcp.platforms.aos8.tools import READ_ONLY
from hpe_networking_mcp.platforms.aos8.tools._helpers import (
    format_aos8_error,
    run_show,
)


@tool(name="aos8_get_controllers", annotations=READ_ONLY)
async def aos8_get_controllers(ctx: Context) -> dict[str, Any] | str:
    """List all controllers (Mobility Conductor + Managed Devices).

    Returns the structured `show switches` table — every controller
    discovered by the Conductor, with role, status, IP, and version.

    Returns:
        Dict with `switches` array, or an error message string on failure.
    """
    client = ctx.lifespan_context["aos8_client"]
    try:
        return await run_show(client, "show switches")
    except Exception as exc:  # noqa: BLE001 — surface to AI as string
        return format_aos8_error(exc, "list controllers")


@tool(name="aos8_get_ap_database", annotations=READ_ONLY)
async def aos8_get_ap_database(
    ctx: Context,
    config_path: str = "/md",
) -> dict[str, Any] | str:
    """Return the AP database for the given config_path scope."""
    client = ctx.lifespan_context["aos8_client"]
    try:
        return await run_show(client, "show ap database", config_path=config_path)
    except Exception as exc:  # noqa: BLE001
        return format_aos8_error(exc, "fetch AP database")
```

### Pattern 2: Shared `_helpers.py`

```python
# src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py
"""Shared helpers for AOS8 tool modules."""

from __future__ import annotations

from typing import Any

import httpx

from hpe_networking_mcp.platforms.aos8.client import (
    AOS8APIError,
    AOS8AuthError,
    AOS8Client,
)


def strip_meta(body: Any) -> Any:
    """Remove the AOS8 _meta and _data scaffolding from a showcommand response.

    AOS8 showcommand responses include:
        - `_meta`: list of column names (schema description)
        - `_global_result`: status envelope (already validated by AOS8Client)

    Both are infrastructure noise; AI clients don't need them. Returns the
    body with these keys removed (when body is a dict) or unchanged otherwise.
    """
    if not isinstance(body, dict):
        return body
    cleaned = {k: v for k, v in body.items() if k not in ("_meta", "_global_result")}
    return cleaned


async def run_show(
    client: AOS8Client,
    command: str,
    *,
    config_path: str | None = None,
) -> dict[str, Any]:
    """Execute an AOS8 show command and return the cleaned JSON body.

    Args:
        client: AOS8Client from lifespan_context.
        command: Full show command string (e.g. "show ap database").
        config_path: Optional hierarchy node (e.g. "/md").

    Returns:
        Parsed JSON body with `_meta` and `_global_result` removed.

    Raises:
        AOS8AuthError, AOS8APIError, httpx.HTTPError: as raised by the client.
    """
    params: dict[str, Any] = {"command": command}
    if config_path is not None:
        params["config_path"] = config_path
    response = await client.request("GET", "/v1/configuration/showcommand", params=params)
    return strip_meta(response.json())


async def get_object(
    client: AOS8Client,
    object_path: str,
    *,
    config_path: str = "/md",
) -> dict[str, Any]:
    """Fetch a configuration object via /v1/configuration/object/<path>.

    Used for WLAN tools (ssid_prof, virtual_ap, ap_group, role).
    """
    response = await client.request(
        "GET",
        f"/v1/configuration/object/{object_path}",
        params={"config_path": config_path},
    )
    return strip_meta(response.json())


def format_aos8_error(exc: BaseException, action: str) -> str:
    """Render an AOS8 exception as a tool-friendly error message."""
    if isinstance(exc, AOS8AuthError):
        return f"AOS8 authentication failed while attempting to {action}: {exc}"
    if isinstance(exc, AOS8APIError):
        return f"AOS8 API error while attempting to {action}: {exc}"
    if isinstance(exc, httpx.HTTPStatusError):
        return f"AOS8 HTTP {exc.response.status_code} while attempting to {action}: {exc}"
    if isinstance(exc, httpx.HTTPError):
        return f"AOS8 transport error while attempting to {action}: {exc}"
    return f"Unexpected error while attempting to {action}: {exc}"
```

### Pattern 3: `aos8_show_command` passthrough with strict prefix check

```python
# src/hpe_networking_mcp/platforms/aos8/tools/troubleshooting.py
@tool(name="aos8_show_command", annotations=READ_ONLY)
async def aos8_show_command(
    ctx: Context,
    command: str,
) -> dict[str, Any] | str:
    """Execute any AOS8 CLI 'show' command via the showcommand endpoint.

    The command MUST start with 'show ' (case-insensitive). Other commands are
    rejected to prevent accidental config or write operations. The `_meta`
    field is stripped from structured responses; plain-string responses are
    wrapped in `{"output": "..."}`.

    Parameters:
        command: A full AOS8 CLI 'show' command, e.g. "show ap active".

    Returns:
        Cleaned JSON body, or `{"output": "..."}` for plain-string responses,
        or an error string on failure.
    """
    normalized = command.strip()
    if not normalized.lower().startswith("show "):
        return f"Only 'show' commands are permitted. Received: {command!r}"

    client = ctx.lifespan_context["aos8_client"]
    try:
        response = await client.request(
            "GET",
            "/v1/configuration/showcommand",
            params={"command": normalized},
        )
    except Exception as exc:  # noqa: BLE001
        return format_aos8_error(exc, f"run show command {normalized!r}")

    # Some show commands return text-wrapped JSON; others return JSON arrays.
    try:
        body = response.json()
    except ValueError:
        return {"output": response.text}
    return strip_meta(body)
```

### Pattern 4: WLAN config-object tool

```python
# src/hpe_networking_mcp/platforms/aos8/tools/wlan.py
@tool(name="aos8_get_ssid_profiles", annotations=READ_ONLY)
async def aos8_get_ssid_profiles(
    ctx: Context,
    config_path: str = "/md",
) -> dict[str, Any] | str:
    """List SSID profiles configured at the given hierarchy node.

    Returns the structured ssid_prof object response.
    """
    client = ctx.lifespan_context["aos8_client"]
    try:
        return await get_object(client, "ssid_prof", config_path=config_path)
    except Exception as exc:  # noqa: BLE001
        return format_aos8_error(exc, "fetch SSID profiles")
```

### Pattern 5: Tool-level test (mock `AOS8Client.request`)

```python
# tests/unit/test_aos8_read_health.py
from __future__ import annotations

import json
from pathlib import Path
from unittest.mock import AsyncMock, MagicMock

import httpx
import pytest

from hpe_networking_mcp.platforms.aos8.tools.health import aos8_get_controllers

pytestmark = pytest.mark.unit

_FIXTURES = Path(__file__).parent / "fixtures" / "aos8"


def _load_fixture(name: str) -> dict:
    return json.loads((_FIXTURES / name).read_text())


def _make_ctx(response_body: dict) -> MagicMock:
    """Build a mock Context whose AOS8Client returns the given JSON body."""
    response = MagicMock(spec=httpx.Response)
    response.json.return_value = response_body
    response.text = json.dumps(response_body)

    client = MagicMock()
    client.request = AsyncMock(return_value=response)

    ctx = MagicMock()
    ctx.lifespan_context = {"aos8_client": client}
    return ctx, client


async def test_get_controllers_returns_cleaned_body():
    body = {
        "_meta": ["Name", "IP Address", "Role"],
        "_global_result": {"status": "0"},
        "switches": [{"Name": "MD-01", "IP Address": "10.0.0.1", "Role": "MD"}],
    }
    ctx, client = _make_ctx(body)

    result = await aos8_get_controllers(ctx)

    assert "_meta" not in result
    assert "_global_result" not in result
    assert result["switches"][0]["Name"] == "MD-01"
    client.request.assert_awaited_once_with(
        "GET",
        "/v1/configuration/showcommand",
        params={"command": "show switches"},
    )


async def test_get_ap_database_passes_config_path():
    body = {"_meta": [], "AP Database": []}
    ctx, client = _make_ctx(body)

    from hpe_networking_mcp.platforms.aos8.tools.health import aos8_get_ap_database

    await aos8_get_ap_database(ctx, config_path="/md/site1")

    client.request.assert_awaited_once_with(
        "GET",
        "/v1/configuration/showcommand",
        params={"command": "show ap database", "config_path": "/md/site1"},
    )
```

### Pattern 6: `__init__.py` final wiring

```python
# src/hpe_networking_mcp/platforms/aos8/__init__.py — Phase 3 final state
"""Aruba OS 8 / Mobility Conductor platform module."""

from __future__ import annotations

import importlib

from fastmcp import FastMCP
from loguru import logger

from hpe_networking_mcp.config import ServerConfig
from hpe_networking_mcp.platforms._common.meta_tools import build_meta_tools

TOOLS: dict[str, list[str]] = {
    "health": [
        "aos8_get_controllers", "aos8_get_ap_database", "aos8_get_active_aps",
        "aos8_get_ap_detail", "aos8_get_bss_table", "aos8_get_radio_summary",
        "aos8_get_version", "aos8_get_licenses",
    ],
    "clients": [
        "aos8_get_clients", "aos8_find_client", "aos8_get_client_detail",
        "aos8_get_client_history",
    ],
    "alerts": [
        "aos8_get_alarms", "aos8_get_audit_trail", "aos8_get_events",
    ],
    "wlan": [
        "aos8_get_ssid_profiles", "aos8_get_virtual_aps", "aos8_get_ap_groups",
        "aos8_get_user_roles",
    ],
    "troubleshooting": [
        "aos8_ping", "aos8_traceroute", "aos8_show_command", "aos8_get_logs",
        "aos8_get_controller_stats", "aos8_get_arm_history", "aos8_get_rf_monitor",
    ],
}

__all__ = ["TOOLS", "register_tools"]


def register_tools(mcp: FastMCP, config: ServerConfig) -> int:
    """Register all AOS8 tools and (in dynamic mode) the meta-tools."""
    from hpe_networking_mcp.platforms.aos8 import _registry
    _registry.mcp = mcp

    # Importing each tool module triggers @tool() decoration -> registry insert
    total = 0
    for category in TOOLS:
        importlib.import_module(f"hpe_networking_mcp.platforms.aos8.tools.{category}")
        total += len(TOOLS[category])

    mode = config.tool_mode
    if mode == "dynamic":
        build_meta_tools("aos8", mcp)
        logger.info("AOS8: registered {} underlying tools + 3 meta-tools (dynamic mode)", total)
    elif mode == "code":
        logger.info("AOS8: registered {} tools (code mode)", total)
    else:
        logger.info("AOS8: registered {} tools (static mode)", total)
    return total
```

### Anti-Patterns to Avoid

- **Building Pydantic models per tool:** Central does this; the cost-benefit is poor for AOS8 because `show` command output shapes vary across firmware. Return raw dicts; let downstream callers decide.
- **Calling `client._http` directly:** every HTTP call must go through `client.request(...)` so UIDARUBA injection, 401-refresh, and `_global_result` checking all apply. Never bypass.
- **Forgetting to strip `_meta`:** every showcommand response will include it. Use the helper unconditionally.
- **Catching only `httpx.HTTPError`:** AOS8 raises `AOS8APIError` and `AOS8AuthError` as well. Catch `Exception` at the tool boundary and route through `format_aos8_error`.
- **Hardcoding `command=show foo` strings in the helper:** the helper takes a string parameter; tools pass full strings so the call site is greppable.
- **Allowing the `aos8_show_command` prefix check to use `startswith("show")`:** Must be `startswith("show ")` (with trailing space) to reject `"showtech"` injection-style tricks. Use `.lower().startswith("show ")`.
- **Putting `aos8_ping` and `aos8_traceroute` through the same prefix-checked path:** Those tools build the command internally — they bypass the prefix check by calling `client.request()` (or `run_show` with the constructed command) directly.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTTP error shaping | Per-tool error dict construction | `format_aos8_error(exc, action)` helper | Consistency; AI clients see the same shape across all 26 tools. |
| `_meta` and `_global_result` removal | Per-tool dict comprehension | `strip_meta(body)` helper | Single source of truth; easy to extend if more keys need stripping. |
| Token / UIDARUBA URL handling | Tool-level URL construction | `client.request(...)` from Phase 2 | Already does sanitized logging, refresh, and `_global_result` checking. |
| Tool registration / dynamic mode | Per-tool wiring | `_registry.tool()` shim + `build_meta_tools()` | Same path as every other platform. |
| Test fixtures | Inline dicts in every test | JSON files in `tests/unit/fixtures/aos8/` | Reusable; fixtures double as documentation of expected response shape. |
| HTTP mocking | New library | `unittest.mock.AsyncMock` for tool tests, `httpx.MockTransport` for client-level tests | Already in deps. |
| Pagination | Custom paginators | Trust AOS8 — most show commands return full results, no pagination needed | Showcommand JSON is a single response. |

**Key insight:** Phase 3 is the simplest phase in terms of new infrastructure. Almost all code is "build URL → call client.request → strip meta → return". The complexity budget is in test coverage (one fixture per tool) and in the `_meta` discipline.

## Common Pitfalls

### Pitfall 1: `aos8_show_command` prefix bypass via leading whitespace
**What goes wrong:** User passes `"   show ap database"` (leading whitespace) — the check `command.startswith("show ")` fails, tool rejects a valid command. Worse, attacker passes `" reload"` and a poorly written check accepts it.
**Why it happens:** Easy to forget `.strip()` before `.lower().startswith()`.
**How to avoid:** Always `command.strip().lower().startswith("show ")`. Add a unit test for `"  show version"` and `"reload"` and `"showtech something"` (all should fail differently).

### Pitfall 2: `_meta` absent from non-showcommand responses
**What goes wrong:** `wlan.py` tools call `/v1/configuration/object/ssid_prof` which has a different shape — no `_meta` key. The `strip_meta` helper handles this fine (only removes when present), but a test asserting `"_meta" not in result` for an object response is meaningless.
**Why it happens:** Confusion between showcommand response shape and config-object response shape.
**How to avoid:** Document the shape difference in `_helpers.py` docstring. WLAN tests should assert response equality against the fixture; showcommand tests should additionally assert `_meta` removed.

### Pitfall 3: `show ap details ap-name <X>` quoting and escaping
**What goes wrong:** AP names can contain hyphens, spaces, or special chars. If tool builds `f"show ap details ap-name {ap_name}"` and AP name is `"AP 01"`, the resulting URL-encoded query may break.
**Why it happens:** httpx URL-encodes query params, but the AOS8 server may parse the command string differently than expected.
**How to avoid:** Validate AP names contain only safe characters at the tool layer; reject names with spaces or quotes; document the constraint. Add a test for an AP name with a hyphen (which is safe) and one with a space (which should be rejected up-front).

### Pitfall 4: Forgetting `READ_ONLY` annotation
**What goes wrong:** Tool registered without `annotations=READ_ONLY` — AI clients see it as a destructive tool by default. Elicitation middleware may prompt for confirmation on every call.
**Why it happens:** Easy to copy a snippet that omits annotations.
**How to avoid:** Tests should assert the registered tool's annotations are READ_ONLY. A category-level helper test like `test_all_health_tools_are_read_only` iterates `REGISTRIES["aos8"]` and checks every tool has `readOnlyHint=True`.

### Pitfall 5: `tool()` decorator runs at import time but `mcp` may be `None`
**What goes wrong:** During test collection, `_registry.mcp` is None unless `register_tools()` has been called. The shim in `_registry.py` handles this by returning the raw function — but if a test imports the tool module and then a later test calls `register_tools()`, the same module's tools may not get registered with the real FastMCP instance.
**Why it happens:** Module import is once-per-process; registration is idempotent.
**How to avoid:** The `conftest.py` `_install_registry_stubs()` already pre-stubs `_registry.mcp` for existing platforms. **Phase 3 must extend that stub list to include `hpe_networking_mcp.platforms.aos8._registry`** — otherwise tool modules imported during test collection don't get a working `tool()` decorator. Verify this in Wave 0.

### Pitfall 6: Tests calling tool functions that aren't actually decorated
**What goes wrong:** Test imports `aos8_get_controllers` directly and calls it. If `_registry.mcp` was None during decoration, the function is the raw undecorated callable — works fine for testing. But if a test then asserts `aos8_get_controllers` is a FastMCP `Tool` object, it fails.
**Why it happens:** Mixing unit and integration test concerns.
**How to avoid:** Tool tests treat `aos8_get_controllers` as a plain async function and call it with a mocked Context. Don't assert on FastMCP wrapping inside unit tests; that's covered by `test_aos8_meta_tools.py` (or similar) at integration level — **deferred to Phase 7**.

### Pitfall 7: `show` commands that don't return JSON
**What goes wrong:** `show running-config`, `show tech-support`, and a handful of others return plain text inside a JSON wrapper (or raw text). Calling `response.json()` returns a string, and `strip_meta` is a no-op on strings. Tools may then index into a string and crash.
**Why it happens:** Vendor inconsistency — the showcommand wrapper isn't uniform.
**How to avoid:** Every tool that handles potentially-text responses (mostly `aos8_show_command` and `aos8_get_logs`) wraps the response in `{"output": ...}` if `response.json()` returns a non-dict. The pattern is in Pattern 3 above.

### Pitfall 8: 500-line file limit creep
**What goes wrong:** `troubleshooting.py` grows past 500 lines as docstrings expand. Pre-push lint fails.
**Why it happens:** Google-style docstrings with examples balloon line counts.
**How to avoid:** Estimate per-tool ~50 lines (signature + ~30-line docstring + ~10-line body). 7 tools × 50 = 350; safe. Watch `troubleshooting.py` specifically — if it nears 450, split into `troubleshooting.py` (ping/traceroute/show_command) and `diagnostics.py` (logs/stats/arm/rf_monitor). Locked file split decision deferred to plan author if budget exceeded.

## Code Examples

Verified patterns from the existing codebase and AOS8 vendor documentation.

### Showcommand response shape (vendor-confirmed)
```json
// Source: developer.arubanetworks.com/aos8/docs/showcommand-api
{
    "Vlan Status": [
        {"VlanId": "1", "IPAddress": "10.0.0.1/24", "Adminstate": "up"},
        {"VlanId": "10", "IPAddress": "10.0.10.1/24", "Adminstate": "up"}
    ],
    "_meta": ["VlanId", "IPAddress", "Adminstate"],
    "_global_result": {"status": "0", "status_str": "Success"}
}
```

After `strip_meta`:
```json
{
    "Vlan Status": [
        {"VlanId": "1", "IPAddress": "10.0.0.1/24", "Adminstate": "up"},
        {"VlanId": "10", "IPAddress": "10.0.10.1/24", "Adminstate": "up"}
    ]
}
```

### Config-object endpoint URL (vendor-confirmed)
```
GET https://<conductor>:4343/v1/configuration/object/ssid_prof?config_path=/md&UIDARUBA=<token>
```

Response (representative):
```json
// Source: developer.arubanetworks.com/aos8/reference/post_object-ssid-prof
{
    "_data": [
        {
            "ssid_prof": [
                {"profile-name": "guest", "essid": {"essid": "Guest WiFi"}, "opmode": {"opensystem": true}}
            ]
        }
    ],
    "_global_result": {"status": "0"}
}
```

### Existing AOS8Client.request signature (Phase 2 deliverable)
```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py:201-271
async def request(
    self,
    method: Literal["GET", "POST"],
    path: str,
    *,
    params: dict[str, Any] | None = None,
    json_body: Any = None,
    data: Any = None,
    timeout: float | None = None,
) -> httpx.Response: ...
```

This is the only client interface tools should use.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Per-tool `try/except` for HTTP | Helper-based error formatting | Project standard since Mist platform | Less duplication; consistent AI-facing errors. |
| Pydantic response models | Raw `dict[str, Any]` returns | Project pragmatic choice for AOS8 | Easier for v1; revisit if AI clients need typed access. |
| Per-call show command construction | Centralized `run_show()` helper | New in Phase 3 | Consistency + easier to add config_path opt-in. |

**Deprecated/outdated:** None — all patterns referenced are current.

## Open Questions

1. **Exact JSON field names in `show ap database`, `show user-table`, `show alarms` across firmware versions**
   - What we know: Field names use mixed casing (`AP Database`, `Status`, `Severity`); some are nested.
   - What's unclear: Whether 8.6.x, 8.10.x, 8.11.x return identical structures.
   - Recommendation: Tests use representative fixtures captured from vendor documentation examples (when available) or synthesized from the showcommand API doc. Tools return raw dicts so firmware drift doesn't break the contract — only tests need updating if shape varies.

2. **Whether `aos8_get_controller_stats` should aggregate multiple show commands**
   - What we know: `show cpuload`, `show memory`, `show switchinfo` each return one piece of info.
   - What's unclear: Whether to return three separate result blocks or aggregate into one tool call.
   - Recommendation: Aggregate — call all three sequentially and return `{"cpu": ..., "memory": ..., "uptime": ...}`. This is what an operator wants (one snapshot). Document the multi-call nature in the tool docstring.

3. **`aos8_get_logs` line-count parameter**
   - What we know: AOS8 `show log system` accepts a count argument: `show log system 100` returns last 100 lines.
   - What's unclear: Whether to default to 100 lines or expose `count` as a parameter.
   - Recommendation: Expose `count: int = 100` parameter; reject `count > 1000` to avoid massive responses.

4. **Whether tests should use `httpx.MockTransport` or `AsyncMock`**
   - What we know: Both work; Phase 2 uses MockTransport.
   - What's unclear: Style preference for tool-level tests.
   - Recommendation: **Use `AsyncMock(AOS8Client.request)` for tool tests** because it focuses on the input-output transformation that's the tool's responsibility. `MockTransport` is overkill — Phase 2 already proves the HTTP layer works. Add one MockTransport-based smoke test in `test_aos8_read_health.py::test_get_version_via_full_http_path` to confirm the integration if extra confidence is desired.

## Environment Availability

This phase is a pure code/config change with no new external runtime dependencies (no new tools, services, runtimes, databases, or package managers introduced). Existing toolchain inherited from prior phases.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python | All Python modules | ✓ | 3.12+ (per `pyproject.toml`) | — |
| uv | Local dev / Docker build | ✓ | latest | — |
| pytest, pytest-asyncio, httpx, fastmcp | Tests + tools | ✓ | already pinned (Phase 2) | — |
| Docker + Compose | Pre-push test gate | ✓ | per project CLAUDE.md | `uv run pytest` directly |
| AOS8 / Mobility Conductor live system | Tests | ✗ | — | All tests use mocked responses (`httpx.MockTransport` or `unittest.mock.AsyncMock`) — explicit project constraint |

**Missing dependencies with no fallback:** None.
**Missing dependencies with fallback:** Live AOS8 — fallback is mock-only testing per project mandate.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | `pytest >= 8.4.0` + `pytest-asyncio >= 1.2.0` (asyncio_mode = "auto") |
| Config file | `pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `uv run pytest tests/unit/test_aos8_read_*.py -x -q` |
| Full suite command | `uv run pytest tests/ -q` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| READ-01 | `aos8_get_controllers` calls `show switches`, returns cleaned body | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_controllers -x` | ❌ Wave 0 |
| READ-02 | `aos8_get_ap_database` passes `config_path` query param | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_ap_database_passes_config_path -x` | ❌ Wave 0 |
| READ-03 | `aos8_get_active_aps` calls `show ap active` | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_active_aps -x` | ❌ Wave 0 |
| READ-04 | `aos8_get_ap_detail` accepts ap_name OR ap_mac, builds correct command | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_ap_detail -x` | ❌ Wave 0 |
| READ-05 | `aos8_get_bss_table` calls `show ap bss-table` | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_bss_table -x` | ❌ Wave 0 |
| READ-06 | `aos8_get_radio_summary` calls `show ap radio-summary` | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_radio_summary -x` | ❌ Wave 0 |
| READ-07 | `aos8_get_version` calls `show version` (no config_path) | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_version -x` | ❌ Wave 0 |
| READ-08 | `aos8_get_licenses` calls `show license` | unit | `pytest tests/unit/test_aos8_read_health.py::test_get_licenses -x` | ❌ Wave 0 |
| READ-09 | `aos8_get_clients` calls `show user-table` with config_path | unit | `pytest tests/unit/test_aos8_read_clients.py::test_get_clients -x` | ❌ Wave 0 |
| READ-10 | `aos8_find_client` builds correct command for MAC/IP/username | unit | `pytest tests/unit/test_aos8_read_clients.py::test_find_client_by_mac -x` etc. | ❌ Wave 0 |
| READ-11 | `aos8_get_client_detail` calls `show user-table verbose mac <m>` | unit | `pytest tests/unit/test_aos8_read_clients.py::test_get_client_detail -x` | ❌ Wave 0 |
| READ-12 | `aos8_get_client_history` (no config_path) calls association history command | unit | `pytest tests/unit/test_aos8_read_clients.py::test_get_client_history -x` | ❌ Wave 0 |
| READ-13 | `aos8_get_alarms` calls `show alarms` with config_path | unit | `pytest tests/unit/test_aos8_read_alerts.py::test_get_alarms -x` | ❌ Wave 0 |
| READ-14 | `aos8_get_audit_trail` calls `show audit-trail` (no config_path) | unit | `pytest tests/unit/test_aos8_read_alerts.py::test_get_audit_trail -x` | ❌ Wave 0 |
| READ-15 | `aos8_get_events` calls `show events` with config_path | unit | `pytest tests/unit/test_aos8_read_alerts.py::test_get_events -x` | ❌ Wave 0 |
| READ-16 | `aos8_get_ssid_profiles` GETs `/object/ssid_prof?config_path=...` | unit | `pytest tests/unit/test_aos8_read_wlan.py::test_get_ssid_profiles -x` | ❌ Wave 0 |
| READ-17 | `aos8_get_virtual_aps` GETs `/object/virtual_ap?config_path=...` | unit | `pytest tests/unit/test_aos8_read_wlan.py::test_get_virtual_aps -x` | ❌ Wave 0 |
| READ-18 | `aos8_get_ap_groups` GETs `/object/ap_group?config_path=...` | unit | `pytest tests/unit/test_aos8_read_wlan.py::test_get_ap_groups -x` | ❌ Wave 0 |
| READ-19 | `aos8_get_user_roles` GETs `/object/role?config_path=...` | unit | `pytest tests/unit/test_aos8_read_wlan.py::test_get_user_roles -x` | ❌ Wave 0 |
| READ-20 | `aos8_ping` builds `ping <dest>` command | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_ping -x` | ❌ Wave 0 |
| READ-21 | `aos8_traceroute` builds `traceroute <dest>` command | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_traceroute -x` | ❌ Wave 0 |
| READ-22 | `aos8_show_command` enforces `show ` prefix (case-insensitive, with whitespace strip) | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_show_command_rejects_non_show -x` etc. | ❌ Wave 0 |
| READ-22 | `aos8_show_command` strips `_meta` from response | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_show_command_strips_meta -x` | ❌ Wave 0 |
| READ-22 | `aos8_show_command` wraps non-JSON response in `{"output": ...}` | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_show_command_handles_text_response -x` | ❌ Wave 0 |
| READ-23 | `aos8_get_logs` calls `show log system <n>` with count param | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_get_logs -x` | ❌ Wave 0 |
| READ-24 | `aos8_get_controller_stats` aggregates cpu/memory/uptime calls | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_get_controller_stats -x` | ❌ Wave 0 |
| READ-25 | `aos8_get_arm_history` calls `show ap arm history` with config_path | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_get_arm_history -x` | ❌ Wave 0 |
| READ-26 | `aos8_get_rf_monitor` calls `show ap monitor stats` with config_path | unit | `pytest tests/unit/test_aos8_read_troubleshooting.py::test_get_rf_monitor -x` | ❌ Wave 0 |
| (cross-cut) | All 26 tools registered with READ_ONLY annotations | unit | `pytest tests/unit/test_aos8_read_*.py::test_all_tools_read_only -x` | ❌ Wave 0 |
| (cross-cut) | All 26 tool names appear in `TOOLS` dict in `__init__.py` | unit | `pytest tests/unit/test_aos8_init.py::test_tools_dict_complete -x` | ❌ Wave 0 |
| (cross-cut) | `register_tools()` populates `REGISTRIES["aos8"]` and calls `build_meta_tools` in dynamic mode | unit | `pytest tests/unit/test_aos8_init.py::test_register_tools_dynamic_mode -x` | ❌ Wave 0 |
| (regression) | All existing platform tests still pass | unit | `pytest tests/unit/ -q` | ✅ existing |

### Sampling Rate
- **Per task commit:** `uv run pytest tests/unit/test_aos8_read_<category>.py -x -q` (under 5s expected for one category)
- **Per wave merge:** `uv run pytest tests/unit/test_aos8_*.py -x -q` (full AOS8 surface)
- **Phase gate:** Full pre-push command from project CLAUDE.md:
  `docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm hpe-networking-mcp sh -c "uv run ruff check . && uv run ruff format --check . && uv run mypy src/ --ignore-missing-imports && uv run pytest tests/ -q"`

### Wave 0 Gaps
- [ ] `tests/unit/test_aos8_read_health.py` — covers READ-01..08
- [ ] `tests/unit/test_aos8_read_clients.py` — covers READ-09..12
- [ ] `tests/unit/test_aos8_read_alerts.py` — covers READ-13..15
- [ ] `tests/unit/test_aos8_read_wlan.py` — covers READ-16..19
- [ ] `tests/unit/test_aos8_read_troubleshooting.py` — covers READ-20..26
- [ ] `tests/unit/test_aos8_init.py` — covers `TOOLS` dict, `register_tools()` wiring, meta-tools registration
- [ ] `tests/unit/fixtures/aos8/*.json` — one fixture per show command + one per config-object endpoint
- [ ] **`tests/conftest.py` extension:** add `hpe_networking_mcp.platforms.aos8._registry` to the `_install_registry_stubs()` tuple — currently missing per Pitfall 5; required so tool modules import cleanly at test-collection time.
- [ ] Framework install: none — pytest, pytest-asyncio, httpx, fastmcp already pinned.

## Sources

### Primary (HIGH confidence)
- [Aruba Developer Hub — Showcommand API](https://developer.arubanetworks.com/aos8/docs/showcommand-api) — `/v1/configuration/showcommand` URL, `command` + `UIDARUBA` query params, `_meta` field semantics, structured JSON response shape
- [Aruba Developer Hub — Hierarchical Configuration](https://developer.arubanetworks.com/aos8/docs/configuration-hierarchy) — `config_path` concept, `/md` and `/mm/mynode` defaults
- [Aruba Developer Hub — GET](https://developer.arubanetworks.com/aos8/docs/get) — `/v1/configuration/object/<obj>?config_path=<p>&UIDARUBA=<t>` URL pattern
- [Aruba Developer Hub — POST](https://developer.arubanetworks.com/aos8/docs/post) — same URL pattern for write tools (Phase 5 reference)
- [Aruba Developer Hub — SSID Profile](https://developer.arubanetworks.com/aos8/reference/post_object-ssid-prof) — `/object/ssid_prof` endpoint shape
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/clients.py` — read directly; canonical `@tool(annotations=READ_ONLY)` + Context injection + lifespan_context client access pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/devices.py` — read directly; reference for inventory/health tool pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/troubleshooting.py` — read directly; ping/traceroute/show_commands shape
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/alerts.py` — read directly; alerts pagination pattern (reference only — AOS8 doesn't paginate showcommand output)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/events.py` — read directly; events query shape
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/__init__.py` — read directly; `READ_ONLY = ToolAnnotations(...)` constant pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` — read directly; `AOS8Client.request()` interface confirmed
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/_registry.py` — read directly; `tool()` shim already in place
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/_common/meta_tools.py` — read directly; `build_meta_tools(platform, mcp)` signature confirmed
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/_common/tool_registry.py` — read directly; `REGISTRIES["aos8"]` already pre-allocated; write-tag set already includes `{"aos8_write", "aos8_write_delete"}`
- `hpe-networking-mcp/tests/conftest.py` — read directly; `_install_registry_stubs()` exists for other platforms; **AOS8 NOT YET in stub list — Wave 0 must add it**; `loguru_capture` and `secrets_dir` fixtures available
- `hpe-networking-mcp/tests/unit/test_aos8_client.py` — read directly; `httpx.MockTransport` test pattern reference
- `.planning/phases/02-api-client/02-RESEARCH.md` — read directly; Phase 2 endpoint reference table; established patterns

### Secondary (MEDIUM confidence)
- [Aruba Developer Hub — Getting Started](https://developer.arubanetworks.com/aos8/docs/getting-started-aos8-restapi) — base URL, port 4343, session caps
- [The CaNerdIan — Working with the ArubaOS API](https://nerdian.ca/2021/10/19/working-with-the-arubaos-api-reading-data/) — community guide corroborating UIDARUBA + config_path semantics
- [ArubaOS 8.9.0.x API Guide PDF](https://arubanetworking.hpe.com/techdocs/ArubaOS-8.x-Books/89/ArubaOS-8.9.0.0-API-Guide.pdf) — vendor PDF (cited but not deeply mined; corroborates endpoint structure)

### Tertiary (LOW confidence)
- Specific JSON field names in show command responses (`AP Database`, `User Table`, etc.) — based on cross-referencing CLI doc snippets, not verified against a live system. Mitigated by the explicit "raw dict passthrough" return contract — tools surface whatever the controller returns; tests can be updated when fixtures from a real Conductor are available.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — every library already pinned, pattern proven across 6 platforms
- Architecture: HIGH — exact file paths, exact line ranges, all helpers and call shapes verified against repo
- AOS8 endpoint paths: HIGH (showcommand, config object) — vendor docs confirmed; `config_path` query param confirmed via WebSearch
- Specific show command field names: LOW — show command JSON shapes vary by firmware; mitigated by raw-dict return contract
- Pitfalls: HIGH — derived from Phase 2 lessons + project conventions
- Test architecture: HIGH — mirror Phase 2 pattern with `AsyncMock(client.request)` substitution

**Research date:** 2026-04-28
**Valid until:** 2026-05-28 (30 days — AOS8 API surface is stable; vendor changes infrequent)
