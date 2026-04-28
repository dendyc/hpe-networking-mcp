# Phase 2: API Client - Context

**Gathered:** 2026-04-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver `AOS8Client` — the async httpx API client for AOS8/Mobility Conductor — and wire it fully into `server.py`. Deliverables:

- `src/hpe_networking_mcp/platforms/aos8/client.py` — `AOS8Client` with lazy login, UIDARUBA token reuse, asyncio.Lock, 401-refresh, `_global_result` error detection, token masking, `health_check()`, `aclose()` logout
- `src/hpe_networking_mcp/platforms/aos8/__init__.py` — empty `TOOLS` dict + `register_tools()` stub
- `src/hpe_networking_mcp/platforms/aos8/_registry.py` — module-level FastMCP holder + `tool()` shim (pattern from other platforms)
- `src/hpe_networking_mcp/platforms/health.py` — `_probe_aos8` added to `_ALL_PLATFORMS` and probe dispatch table
- `src/hpe_networking_mcp/server.py` — lifespan wiring (AOS8Client init/close + context dict entries), write-gate visibility transform, `_register_aos8_tools()` call
- `tests/unit/test_aos8_client.py` — TDD unit tests for the client (alongside implementation, not deferred)

**Not in scope for Phase 2:** Any actual `aos8_*` tool implementations — those land in Phase 3. The tools/ directory will be empty (or contain only an `__init__.py`) after this phase.

</domain>

<decisions>
## Implementation Decisions

### Health Check (CLIENT-09)
- **D-01:** `client.health_check()` calls a GET show-command endpoint to retrieve controller version/hostname — not just a login probe. The health tool surfaces controller hostname and AOS software version string alongside the other six platforms.
- **D-02:** Matches the depth of what Mist and Central surface in the health tool — gives operators actionable info at a glance rather than just reachable/unreachable.

### Logout on Shutdown (CLIENT-10)
- **D-03:** `aclose()` calls an explicit AOS8 logout endpoint (POST) before closing the httpx client. This releases the UIDARUBA session on the Conductor and prevents session accumulation across server restarts.
- **D-04:** If the logout call errors (network gone, Conductor unreachable), the error is caught, logged as a WARNING, and swallowed — shutdown must not fail because of a logout failure. Pattern: `try/except Exception` around the logout call (Bandit B110 permitted for shutdown paths per project conventions).

### server.py Scope
- **D-05:** Phase 2 delivers **full server.py wiring** — no server.py changes needed in Phase 3 or later for the AOS8 platform bootstrap:
  - Lifespan: `AOS8Client` instantiation + `context["aos8_client"]` + `context["aos8_config"]` entries (matching Apstra pattern at lines 118–130)
  - Shutdown: `aos8.aclose()` call in the lifespan cleanup block
  - Health probe: `_probe_aos8` function in `health.py`, added to `_ALL_PLATFORMS` tuple and `_PROBE_DISPATCH` dict
  - Write-gate: `mcp.add_transform(Visibility(False, tags={"aos8_write", "aos8_write_delete"}, components={"tool"}))` when `not config.enable_aos8_write_tools`
  - Tool registration: `_register_aos8_tools(mcp, config)` function + call in `create_server()` — stub only, no tools yet

### TDD Sequencing
- **D-06:** Phase 2 follows TDD — `tests/unit/test_aos8_client.py` is written alongside (or before) the client implementation. Tests must pass in CI on phase completion. Phase 7 adds tool-level tests and regression coverage on top; it does NOT revisit client-level tests.
- **D-07:** Client tests cover: login success, login failure, token reuse (no second login on second call), 401 single-refresh (no infinite loop), `_global_result.status != 0` → `AOS8APIError`, verify_ssl WARNING, token masking (UIDARUBA never in log output), `aclose()` logout behavior.

### Claude's Discretion
- Exact AOS8 show-command path/parameters used by `health_check()` — researcher to verify against AOS8 API docs
- Exact AOS8 logout endpoint path — researcher to verify
- Whether `health_check()` returns a dict with hostname+version or raises on failure (align with how other platforms' `health_check()` methods behave)
- Internal variable naming and module structure within `client.py`

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Client Pattern (Primary analog)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/apstra/client.py` — canonical pattern for: asyncio.Lock, lazy token acquisition via `_ensure_token()`, `_refresh_token()`, `_login_locked()`, `request()` with 401-retry, `aclose()`, `health_check()`. AOS8Client mirrors this structure with AOS8-specific auth mechanics.

### server.py Wiring Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/server.py` — Apstra wiring at lines 118–130 (lifespan), 180–182 (shutdown aclose), 235–236 (register_tools call), 276–277 (write-gate visibility), 407–409 (`_register_apstra_tools` function). AOS8 additions follow the same structure.

### Health Probe Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/health.py` — `_probe_apstra` (line 143) and `_probe_axis` (line 154) as reference for `_probe_aos8`. `_ALL_PLATFORMS` tuple (line 32) and `_PROBE_DISPATCH` dict (lines 186–191) to extend.

### Platform Module Structure
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/apstra/__init__.py` — `TOOLS` dict + `register_tools()` pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/apstra/_registry.py` — module-level `mcp = None` holder + `tool()` shim pattern

### Secrets & Config (from Phase 1)
- `hpe-networking-mcp/src/hpe_networking_mcp/config.py` — `AOS8Secrets` dataclass and `_load_aos8()` already implemented in Phase 1; `ServerConfig.aos8` field present

### Token Masking Utility
- `hpe-networking-mcp/src/hpe_networking_mcp/utils/logging.py` — `mask_secret()` function; must be applied to UIDARUBA token in all log calls

### Existing Client Tests (reference for test structure)
- `hpe-networking-mcp/tests/unit/test_apstra_client.py` (if exists) — reference for mocked httpx test patterns
- `hpe-networking-mcp/tests/conftest.py` — shared fixtures including `secrets_dir` (AOS8 keys added in Phase 1)

### AOS8 API Auth Semantics
- `hpe-networking-mcp/.planning/PROJECT.md` §AOS8 API characteristics — login endpoint, UIDARUBA cookie mechanics, show command URL pattern, `_global_result` error structure

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ApstraClient` in `apstra/client.py`: direct structural template — `__init__`, `_ensure_token`, `_refresh_token`, `_login_locked`, `request`, `aclose`, `health_check` map 1:1 to AOS8Client methods. Key difference: AOS8 uses cookie-based UIDARUBA token (httpx cookie jar + `self._token`) vs Apstra's bearer token in `AuthToken` header.
- `mask_secret()` in `utils/logging.py`: apply to UIDARUBA token in every log line that touches auth.
- `asyncio.Lock` pattern: copy exactly from Apstra — same concurrent-request safety requirement.
- Phase 1 conftest fixture: already has AOS8 secret keys in `secrets_dir`; test_aos8_client.py can reuse it.

### Established Patterns
- Lazy login: `_ensure_token()` checks `self._token is not None` before acquiring lock; double-check inside lock.
- 401 retry: `request()` calls `_ensure_token()`, sends request; on 401 calls `_refresh_token()` once, retries; second 401 surfaces error.
- Shutdown cleanup: `try/except Exception` around `aclose()` call in lifespan finally block — server shutdown must not raise.
- Write-gate: `mcp.add_transform(Visibility(False, tags={"aos8_write", "aos8_write_delete"}, components={"tool"}))` — added after `_register_aos8_tools()`, guarded by `not config.enable_aos8_write_tools`.
- `on_duplicate="replace"` in FastMCP init: already set, no change needed.

### Integration Points
- `server.py:lifespan()` — add AOS8Client init block after Axis block (last existing platform); add `aos8_client` and `aos8_config` to context dict.
- `server.py:create_server()` — add `if config.aos8: _register_aos8_tools(mcp, config)` after Axis registration; add write-gate transform.
- `health.py:_ALL_PLATFORMS` — append `"aos8"` to tuple.
- `health.py:_PROBE_DISPATCH` — add `"aos8": _probe_aos8` entry.

</code_context>

<specifics>
## Specific Ideas

- Health check should return controller hostname + AOS version string — same level of detail that Mist/Central surface. Researcher should identify the lightest AOS8 endpoint that returns both (likely `show version` via the show command API).
- Logout errors on shutdown: log as WARNING, do not re-raise. This is the same treatment as any other cleanup failure in the lifespan finally block.
- AOS8 sessions on the Conductor have a finite limit — explicit logout on `aclose()` is a correctness concern, not just hygiene.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 02-api-client*
*Context gathered: 2026-04-28*
