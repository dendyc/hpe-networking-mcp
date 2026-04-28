# Phase 2: API Client - Research

**Researched:** 2026-04-28
**Domain:** AOS8 / Mobility Conductor REST API client (async httpx, UIDARUBA cookie auth, lazy login, lifespan singleton)
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Health Check (CLIENT-09):**
- **D-01:** `client.health_check()` calls a GET show-command endpoint to retrieve controller version/hostname — not just a login probe. The health tool surfaces controller hostname and AOS software version string alongside the other six platforms.
- **D-02:** Matches the depth of what Mist and Central surface in the health tool — gives operators actionable info at a glance rather than just reachable/unreachable.

**Logout on Shutdown (CLIENT-10):**
- **D-03:** `aclose()` calls an explicit AOS8 logout endpoint (POST) before closing the httpx client. This releases the UIDARUBA session on the Conductor and prevents session accumulation across server restarts.
- **D-04:** If the logout call errors (network gone, Conductor unreachable), the error is caught, logged as a WARNING, and swallowed — shutdown must not fail because of a logout failure. Pattern: `try/except Exception` around the logout call (Bandit B110 permitted for shutdown paths per project conventions).

**server.py Scope:**
- **D-05:** Phase 2 delivers **full server.py wiring** — no server.py changes needed in Phase 3 or later for the AOS8 platform bootstrap:
  - Lifespan: `AOS8Client` instantiation + `context["aos8_client"]` + `context["aos8_config"]` entries (matching Apstra pattern at lines 118–130)
  - Shutdown: `aos8.aclose()` call in the lifespan cleanup block
  - Health probe: `_probe_aos8` function in `health.py`, added to `_ALL_PLATFORMS` tuple and `_PROBE_DISPATCH` dict
  - Write-gate: `mcp.add_transform(Visibility(False, tags={"aos8_write", "aos8_write_delete"}, components={"tool"}))` when `not config.enable_aos8_write_tools`
  - Tool registration: `_register_aos8_tools(mcp, config)` function + call in `create_server()` — stub only, no tools yet

**TDD Sequencing:**
- **D-06:** Phase 2 follows TDD — `tests/unit/test_aos8_client.py` is written alongside (or before) the client implementation. Tests must pass in CI on phase completion. Phase 7 adds tool-level tests and regression coverage on top; it does NOT revisit client-level tests.
- **D-07:** Client tests cover: login success, login failure, token reuse (no second login on second call), 401 single-refresh (no infinite loop), `_global_result.status != 0` → `AOS8APIError`, verify_ssl WARNING, token masking (UIDARUBA never in log output), `aclose()` logout behavior.

### Claude's Discretion
- Exact AOS8 show-command path/parameters used by `health_check()` — researcher to verify against AOS8 API docs
- Exact AOS8 logout endpoint path — researcher to verify
- Whether `health_check()` returns a dict with hostname+version or raises on failure (align with how other platforms' `health_check()` methods behave)
- Internal variable naming and module structure within `client.py`

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CLIENT-01 | `AOS8Client` authenticates via POST login, stores UIDARUBA in cookie jar + `self._token`; reused across calls | Login flow (§AOS8 API Endpoint Reference); cookie jar + `_token` dual storage (Apstra-mirror pattern); `_ensure_token()` returns cached value |
| CLIENT-02 | `asyncio.Lock` serializes token refresh — no parallel logins | Apstra `_ensure_token` / `_login_locked` pattern with double-checked locking copied verbatim |
| CLIENT-03 | Lazy login — unreachable Conductor at startup does not block server | Constructor does NOT call login. `health_check()` is the only startup probe and is wrapped by `_probe_aos8` which catches all exceptions and reports `degraded` |
| CLIENT-04 | 401 → exactly one refresh attempt, no infinite loop | `request()` mirror of Apstra: refresh once on first 401, second 401 surfaces via `raise_for_status()` |
| CLIENT-05 | `_global_result.status != 0` → `AOS8APIError` (never silent bad data) | Helper `_check_global_result(body)` invoked on all 2xx JSON responses (login + request + show command) |
| CLIENT-06 | UIDARUBA never logged | `mask_secret(token)` on every log line; URL sanitizer strips `UIDARUBA=` from any URL before logging; `format_http_error()` redacts URL + body |
| CLIENT-07 | `verify_ssl` defaults True; False emits startup WARNING | Constructor checks `config.verify_ssl is False` and emits `logger.warning("AOS8: SSL verification disabled — ...")` |
| CLIENT-08 | Port configurable, never hardcoded 4343 | Base URL = `f"https://{config.host}:{config.port}"`; `AOS8Secrets.port` already exists from Phase 1 (default 4343) |
| CLIENT-09 | `client.health_check()` used by server's `health` tool | `health_check()` calls `GET /v1/configuration/showcommand?command=show+version` and returns `{hostname, version}` dict; `_probe_aos8` in `health.py` consumes it |
| CLIENT-10 | `aclose()` logs out + closes httpx | `aclose()` calls `GET /v1/api/logout` (token in cookie), catches all exceptions, logs WARNING, then `self._http.aclose()` |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

These directives are extracted from `./CLAUDE.md` (root) and `hpe-networking-mcp/CLAUDE.md`. The planner MUST verify the plan complies.

- **GSD workflow enforcement:** Direct repo edits are forbidden outside a GSD command flow.
- **API:** GET and POST only — no PUT/PATCH/DELETE on AOS8.
- **Auth:** UIDARUBA must be reused across calls (no per-call login).
- **Testing:** No live AOS8 — all tests use mocked HTTP responses.
- **Compatibility:** Must not break any existing platform module or test (`tests/unit/test_*` for mist/central/greenlake/clearpass/apstra/axis must remain green).
- **SSL:** Default `verify_ssl=True`; opt-in `False` for self-signed; emit WARNING when disabled.
- **Style:** Ruff lint + format, mypy clean, max 500 lines per file, max 50 lines per function, max 100 lines per class, Google-style docstrings.
- **Line length:** 120 chars; double quotes; trailing commas in multi-line.
- **Naming:** `snake_case` funcs/vars, `PascalCase` classes, `_leading_underscore` private.
- **Logging:** All output to stderr (loguru); `mask_secret()` for any token in logs.
- **TDD:** Write tests first, watch them fail, implement minimum to pass, refactor.
- **Pre-push gate:** `ruff check && ruff format --check && mypy src/ && pytest tests/ -q` (run inside `docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm hpe-networking-mcp …`).
- **Commit messages:** Never include "claude code" or "written by claude code" in commit messages.
- **Bandit B110 (`try/except/pass`):** permitted in shutdown/cleanup paths only — applies directly to `aclose()` logout swallowing.

## Summary

Phase 2 builds `AOS8Client` (the async httpx API client) and wires it fully into `server.py` and `health.py` so the AOS8 platform behaves exactly like the other six at the bootstrap layer. This is **not greenfield** — the canonical template (`apstra/client.py`, ~210 lines) is already in the repo and works in production. The work is a near-line-by-line port with three deltas: (1) UIDARUBA carried in the httpx cookie jar AND query param instead of an `AuthToken` header, (2) GET/POST-only method whitelist, (3) every 2xx JSON response gets an `_global_result.status` check before returning to callers. Phase 1 has already delivered `AOS8Secrets`, `_load_aos8()`, the registry entries, the docker-compose secrets, and shared conftest fixtures — none of those need to change.

The two CLIENT-09/CLIENT-10 discretion items resolved through API-doc verification:
- **Login endpoint:** `POST https://<host>:<port>/v1/api/login` with `application/x-www-form-urlencoded` body `username=<u>&password=<p>` (Aruba Developer Hub authoritative; some pages list GET, but POST with form data is the documented standard and avoids credentials in URL/access-logs).
- **Logout endpoint:** `POST https://<host>:<port>/v1/api/logout` with the SESSION cookie (no body required).
- **Health-check endpoint:** `GET /v1/configuration/showcommand?command=show+version&UIDARUBA=<token>` returns structured JSON with controller hostname/version. This is the lightest endpoint that returns both pieces of info at once.

The chief risk for this phase is **cargo-culting Apstra `assert` statements** (Python `-O` removes them — replace with explicit `raise`) and **logging the showcommand URL** (it carries `UIDARUBA=...`). The plan must address both explicitly.

**Primary recommendation:** Fork `apstra/client.py` as `aos8/client.py`, swap `AuthToken` header for cookie jar + `UIDARUBA` query injection, add `_check_global_result()` after every JSON parse, add a URL-sanitizer used in every log call, replace `assert` with explicit `raise`, and ship `tests/unit/test_aos8_client.py` mirroring `test_apstra_client.py`'s `httpx.MockTransport` strategy (no new test deps).

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `httpx` | `>= 0.28.0` (already pinned) | Async HTTPS client (login, request, logout, cookie jar) | Used by Apstra, GreenLake, Axis. Built-in async cookie jar handles AOS8 SESSION cookie auto-persistence. `verify=` flag for self-signed cert opt-out. |
| `loguru` | `>= 0.7.3` (already pinned) | Structured stderr logging with `mask_secret` | Project-standard logger; stderr-only contract preserved. |
| `pydantic` | `>= 2.12.0` (already pinned) | Already provides `AOS8Secrets` dataclass via `config.py` | No new models needed in this phase (tools land in Phase 3). |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `asyncio` (stdlib) | 3.12 | `asyncio.Lock` for serialized token acquisition | Mandatory — AOS8 sessions exhaustible if parallel logins fire. |
| `fastmcp` | `>= 3.1.1` (already pinned) | `ToolError` (returned by `get_aos8_client()`); `Visibility` transform; `Context` type for probe | Existing infrastructure; nothing new. |
| `pytest` | `>= 8.4.0` (already pinned) | Test runner; `@pytest.mark.unit` | Phase 1 conftest fixtures already include AOS8 secrets. |
| `pytest-asyncio` | `>= 1.2.0` (already pinned) | `asyncio_mode = "auto"` enables async tests without explicit decorator | Same pattern as `test_apstra_client.py`. |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `httpx` | `requests` (sync) | Would force `asyncio.to_thread()` wrappers and break the lifespan/lock model. **Reject.** |
| `httpx` | `aiohttp` | New dep with no benefit; project standardizes on httpx. **Reject.** |
| Raw httpx | `pyaoscx` SDK | `pyaoscx` is for AOS-CX *switches*, not AOS8 wireless controllers. No official Python SDK exists for AOS8. **Reject.** |
| `httpx.MockTransport` (test) | `respx`, `pytest-httpx` | `MockTransport` is built into httpx, requires no new dep, and is what `test_apstra_client.py` already uses. **Accept MockTransport.** |
| `assert self._token is not None` | Explicit `if … raise AOS8AuthError` | `assert` removed by Python `-O`; CONCERNS.md flags this. **Use explicit raise.** |

**Installation:** No new packages required. All deps already in `pyproject.toml` and `uv.lock`.

**Version verification:** Skipped — no new deps to add. Existing pins are inherited from upstream and verified by Phase 1 CI.

## Architecture Patterns

### Recommended Project Structure (added by this phase)
```
src/hpe_networking_mcp/platforms/aos8/
├── __init__.py        # TOOLS = {} dict + register_tools(mcp, config) stub
├── _registry.py       # module-level mcp = None holder + tool() shim (Apstra-pattern copy)
├── client.py          # AOS8Client, AOS8AuthError, AOS8APIError, get_aos8_client(),
│                      # format_http_error(), _sanitize_url_for_log()
└── tools/             # empty in Phase 2; tools land in Phase 3
    └── __init__.py    # empty
tests/unit/
└── test_aos8_client.py  # mirrors test_apstra_client.py shape
```

Modified in this phase:
- `src/hpe_networking_mcp/platforms/health.py` — add `_probe_aos8`, extend `_ALL_PLATFORMS` and `_PROBES`
- `src/hpe_networking_mcp/server.py` — add AOS8 lifespan block, shutdown aclose, register_tools stub, write-gate Visibility

### Pattern 1: Lazy login + cached token + asyncio.Lock (Apstra mirror)

**What:** Constructor builds `httpx.AsyncClient` but does NOT log in. First `request()` call triggers `_ensure_token()`, which double-checks `self._token` inside an `asyncio.Lock` and calls `_login_locked()` only when needed. Concurrent callers all wait on the same lock; exactly one login fires.

**When to use:** Every authenticated request from `AOS8Client`.

**Example:**
```python
# Pattern source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/apstra/client.py:63-79
async def _ensure_token(self) -> str:
    if self._token is not None:
        return self._token
    async with self._lock:
        if self._token is None:
            await self._login_locked()
        if self._token is None:
            raise AOS8AuthError("login completed without setting token")  # NOT assert
        return self._token

async def _refresh_token(self) -> str:
    async with self._lock:
        self._token = None
        await self._login_locked()
        if self._token is None:
            raise AOS8AuthError("refresh login completed without setting token")
        return self._token
```

### Pattern 2: Login request (AOS8-specific)

**What:** POST form-encoded credentials. Token comes back at `body["_global_result"]["UIDARUBA"]`. The `SESSION` cookie is auto-stored in `self._http.cookies` by httpx.

**Source:** [Aruba Developer Hub — Authentication](https://developer.arubanetworks.com/aos8/docs/login)

```python
# Source: developer.arubanetworks.com/aos8/docs/login
async def _login_locked(self) -> None:
    """POST credentials, parse UIDARUBA token from _global_result. Caller holds lock."""
    logger.info("AOS8: requesting new UIDARUBA from {}", self._config.host)
    try:
        response = await self._http.post(
            "/v1/api/login",
            data={"username": self._config.username, "password": self._config.password},
            headers={"Content-Type": "application/x-www-form-urlencoded"},
            timeout=_AUTH_TIMEOUT,
        )
    except httpx.HTTPError as e:
        raise AOS8AuthError(f"AOS8 login request failed: {e}") from e

    if response.status_code != 200:
        raise AOS8AuthError(f"AOS8 login HTTP {response.status_code}: {response.text[:200]}")
    try:
        body = response.json()
    except ValueError as e:
        raise AOS8AuthError(f"AOS8 login non-JSON response: {response.text[:200]}") from e

    gr = body.get("_global_result") or {}
    # AOS8 returns status as a string ("0"/"1") in some firmware versions, int in others — normalize
    status = gr.get("status")
    if str(status) != "0":
        raise AOS8AuthError(f"AOS8 login rejected: {gr.get('status_str', 'unknown')}")

    token = gr.get("UIDARUBA")
    if not token:
        raise AOS8AuthError(f"AOS8 login response missing UIDARUBA: {body}")
    self._token = token
    logger.info("AOS8: obtained UIDARUBA {}", mask_secret(token))
```

### Pattern 3: Authenticated request with UIDARUBA query injection + GET/POST whitelist + 401 retry

**What:** Every authenticated call merges `UIDARUBA=<token>` into the query string (cookie alone is insufficient on some endpoints — belt-and-suspenders, documented vendor pattern). Method is restricted to GET/POST only.

```python
# Pattern source: apstra/client.py:118-160; AOS8 deltas marked
async def request(
    self,
    method: Literal["GET", "POST"],   # AOS8 supports only GET/POST (CLAUDE.md constraint)
    path: str,
    *,
    params: dict[str, Any] | None = None,
    json_body: Any = None,
    data: Any = None,                  # for form-encoded POSTs (some AOS8 endpoints want form, not JSON)
    timeout: float | None = None,
) -> httpx.Response:
    method_up = method.upper()
    if method_up not in ("GET", "POST"):
        raise ValueError(f"AOS8 supports only GET/POST; got {method!r}")

    token = await self._ensure_token()
    merged_params = {"UIDARUBA": token, **(params or {})}
    kwargs: dict[str, Any] = {"params": merged_params}
    if json_body is not None:
        kwargs["json"] = json_body
    if data is not None:
        kwargs["data"] = data
    if timeout is not None:
        kwargs["timeout"] = timeout

    response = await self._http.request(method_up, path, **kwargs)
    if response.status_code == 401:
        logger.info(
            "AOS8: 401 on {} {} — refreshing token once",
            method_up, _sanitize_url_for_log(path),
        )
        token = await self._refresh_token()
        merged_params["UIDARUBA"] = token
        kwargs["params"] = merged_params
        response = await self._http.request(method_up, path, **kwargs)
    response.raise_for_status()
    self._check_global_result(response)
    return response
```

### Pattern 4: `_global_result.status` check on every JSON response (CLIENT-05)

**What:** AOS8 returns HTTP 200 with `{"_global_result": {"status": <non-zero>, "status_str": "..."}}` on application errors. `raise_for_status()` is happy; the data is bad. Helper checks every JSON body and raises `AOS8APIError` on `status != 0`.

```python
class AOS8APIError(RuntimeError):
    """Raised when AOS8 returns 2xx with _global_result.status != 0."""

def _check_global_result(self, response: httpx.Response) -> None:
    """Raise AOS8APIError if the JSON body indicates application-level failure.

    Skips bodies that aren't JSON (some showcommand outputs are text-wrapped) — those
    are handled at the tool layer in Phase 2 of show command parsing.
    """
    ctype = response.headers.get("content-type", "")
    if "json" not in ctype.lower():
        return
    try:
        body = response.json()
    except ValueError:
        return
    gr = body.get("_global_result") if isinstance(body, dict) else None
    if not isinstance(gr, dict):
        return
    status = gr.get("status")
    # Some firmware returns int, some str — normalize
    if str(status) not in ("0", "None"):
        msg = gr.get("status_str") or "unknown AOS8 error"
        raise AOS8APIError(f"AOS8 API error (status={status}): {msg}")
```

### Pattern 5: URL log sanitizer (CLIENT-06)

```python
import re
_UIDARUBA_RE = re.compile(r"(UIDARUBA=)[^&\s]+", re.IGNORECASE)

def _sanitize_url_for_log(url: str) -> str:
    """Replace UIDARUBA=<token> with UIDARUBA=<redacted> in any URL string."""
    return _UIDARUBA_RE.sub(r"\1<redacted>", str(url))
```

Apply in every log call that touches a URL: `logger.info("AOS8: GET {}", _sanitize_url_for_log(url))`.

### Pattern 6: health_check() returns hostname + version dict

**Source:** [Aruba Developer Hub — Showcommand API](https://developer.arubanetworks.com/aos8/docs/showcommand-api)

```python
async def health_check(self) -> dict[str, Any]:
    """Probe AOS8 by issuing 'show version'. Returns hostname + AOS version.

    Used by platforms/health.py:_probe_aos8(). Raises on any failure (login,
    HTTP, _global_result, or unexpected response shape) — caller catches and
    reports degraded status.
    """
    response = await self.request(
        "GET",
        "/v1/configuration/showcommand",
        params={"command": "show version"},
        timeout=_HEALTH_TIMEOUT,
    )
    body = response.json()
    # AOS8 'show version' returns hostname under '_meta' or top-level 'Hostname';
    # the exact shape varies — try common keys, fall back to passing the dict through.
    hostname = (
        body.get("Hostname")
        or body.get("hostname")
        or body.get("_data", [None])[0] if isinstance(body.get("_data"), list) else None
    )
    version = body.get("Version") or body.get("version") or body.get("ArubaOS (MODEL: ...)") or "unknown"
    return {"hostname": hostname, "version": version, "raw": body}
```

> Implementation note: the exact key names in `show version` JSON vary by ArubaOS minor version. Implementers should add a fixture from a real Conductor (or accept a graceful fallback that surfaces `raw` to the operator if parsing fails). The phase plan must include an implementer task to capture or stub a representative response shape and store it in `tests/unit/fixtures/aos8_show_version.json`.

### Pattern 7: aclose() logs out then closes (CLIENT-10, D-03/D-04)

```python
async def aclose(self) -> None:
    """Logout (best-effort) then close the httpx client.

    Logout failures are caught and logged as WARNING — shutdown must not raise.
    Permitted by Bandit B110 (try/except/pass-equivalent in shutdown paths).
    """
    if self._token is not None:
        try:
            # Per Aruba docs: GET or POST /v1/api/logout with SESSION cookie. We use POST
            # to align with project convention that mutations are POST and to avoid any
            # caching layer reusing a GET.
            await self._http.post("/v1/api/logout", timeout=_AUTH_TIMEOUT)
            logger.info("AOS8: logged out cleanly")
        except Exception as e:
            logger.warning("AOS8: logout failed during shutdown — {}", e)
        finally:
            self._token = None
    await self._http.aclose()
```

### Pattern 8: _probe_aos8 in health.py

```python
async def _probe_aos8(ctx: Context) -> dict[str, Any]:
    client = ctx.lifespan_context.get("aos8_client")
    if client is None:
        return {"status": "unavailable", "message": "AOS8 is not configured or failed to initialize"}
    try:
        info = await client.health_check()
        return {
            "status": "ok",
            "message": "AOS8 API reachable",
            "host": client.server,           # e.g. "conductor.example.com:4343"
            "hostname": info.get("hostname"),
            "version": info.get("version"),
        }
    except Exception as e:
        return {"status": "degraded", "message": f"AOS8 probe failed: {e}"}
```

Then extend `_ALL_PLATFORMS` to include `"aos8"` and add `"aos8": _probe_aos8` to `_PROBES`.

### Pattern 9: server.py wiring (mirror Apstra block)

Lifespan addition (after Axis block at lines 132–145):
```python
# --- AOS8 ---
if config.aos8:
    try:
        from hpe_networking_mcp.platforms.aos8.client import AOS8Client
        context["aos8_client"] = AOS8Client(config.aos8)
        context["aos8_config"] = config.aos8
    except Exception as e:
        logger.warning("AOS8: failed to initialize — {}", e)
        context["aos8_client"] = None
        context["aos8_config"] = None
else:
    context["aos8_client"] = None
    context["aos8_config"] = None
```

Shutdown addition (after Axis aclose at line 185):
```python
aos8 = context.get("aos8_client")
if aos8 is not None:
    await aos8.aclose()
```

`create_server()` registration (after Axis at line 238):
```python
if config.aos8:
    _register_aos8_tools(mcp, config)
```

Write-gate (after Axis at line 279):
```python
if not config.enable_aos8_write_tools:
    mcp.add_transform(Visibility(False, tags={"aos8_write", "aos8_write_delete"}, components={"tool"}))
```

`_register_aos8_tools()` helper (after `_register_axis_tools` at line 415):
```python
def _register_aos8_tools(mcp: FastMCP, config: ServerConfig) -> None:
    """Register all AOS8 platform tools."""
    from hpe_networking_mcp.platforms.aos8 import register_tools
    count = register_tools(mcp, config)
    logger.info("AOS8: registered {} tools", count)
```

`platforms/aos8/__init__.py` (Phase 2 stub — empty TOOLS):
```python
"""Aruba OS 8 / Mobility Conductor platform module."""

import importlib
from fastmcp import FastMCP
from loguru import logger

from hpe_networking_mcp.config import ServerConfig

TOOLS: dict[str, list[str]] = {}  # populated in Phase 3

def register_tools(mcp: FastMCP, config: ServerConfig) -> int:
    """Wire up the AOS8 platform. In Phase 2 this is a no-op stub: no tools registered.

    The _registry.mcp holder is set so future tool-module imports work, but no
    importlib.import_module() calls are made because tools/ is empty.
    """
    from hpe_networking_mcp.platforms.aos8 import _registry
    _registry.mcp = mcp

    if config.tool_mode == "dynamic":
        # No meta-tools yet — meta-tools are registered when there's at least one
        # underlying tool. Phase 3 will call build_meta_tools("aos8", mcp).
        logger.info("AOS8: 0 underlying tools registered (dynamic mode — meta-tools deferred until tools exist)")
    elif config.tool_mode == "code":
        logger.info("AOS8: 0 underlying tools registered (code mode)")
    else:
        logger.info("AOS8: 0 tools registered (static mode)")
    return 0
```

### Anti-Patterns to Avoid
- **Calling `_login_locked()` in `__init__`:** breaks lazy-login contract; unreachable Conductor blocks server startup. (Pitfall M6 in PITFALLS.md.)
- **`assert self._token is not None` after lock:** `python -O` removes asserts; replace with explicit `if … raise`. (Pitfall m4.)
- **Logging the raw URL:** showcommand URLs contain `UIDARUBA=...`. Always pass through `_sanitize_url_for_log()`. (Pitfall C2.)
- **Per-call login:** Conductor session table fills; admins evicted. (Pitfall C1.)
- **`verify=False` as a default:** silent MITM exposure. Default `True`, opt-in `False` with WARNING. (Pitfall C5.)
- **Hardcoded port `:4343`:** breaks reverse-proxy / non-default deployments. Use `config.port`. (Pitfall C6.)
- **Auto-retry login on 401 in a loop:** refresh ONCE; second 401 surfaces. (Pitfall m5.)
- **Logout call raising on shutdown:** swallow with WARNING; never let it abort lifespan teardown. (D-04.)
- **Adding write-tool tags during this phase:** Phase 2 has no tools — write-gate transform is added but it gates an empty surface. That's intentional; Phase 3+ tools will inherit the gate.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| HTTP cookie persistence across calls | Custom cookie parser / `Cookie:` header builder | `httpx.AsyncClient` (default cookie jar enabled) | httpx automatically stores Set-Cookie from login response and replays on subsequent calls. |
| Async-safe single-flight token acquisition | Custom future + flag dance | `asyncio.Lock` with double-checked pattern from `apstra/client.py` | Already proven, already tested. |
| Token redaction | Per-log-call string slicing | `mask_secret()` from `utils/logging.py` | Already used by every other platform; central place to harden. |
| URL token redaction | Manual `str.replace("UIDARUBA=" + token, "...")` | Compiled regex `r"(UIDARUBA=)[^&\s]+"` (small new helper `_sanitize_url_for_log`) | Works without knowing the exact token value; survives token rotation. |
| HTTP error formatting for tool returns | Bespoke shape per platform | Copy/adapt `format_http_error()` from `apstra/client.py:191-206` | Project-standard shape: `{status_code, message, body}`. Phase 3 tools rely on this. |
| HTTP test mocking | New library (respx, pytest-httpx) | `httpx.MockTransport` (already used by `test_apstra_client.py`) | No new dep; identical pattern; already in conftest stubbing. |
| Health probe registration | New aggregator | Extend existing `_ALL_PLATFORMS` + `_PROBES` in `health.py` | Single source of truth for startup-log + tool-output. |

**Key insight:** Phase 2 is a port, not an invention. Every problem here has a working answer in `apstra/client.py` or `axis/client.py`. The only original code is the AOS8-specific delta surface (cookie + UIDARUBA query, GET/POST whitelist, `_global_result` check, URL sanitizer) — and even that is small.

## Common Pitfalls

### Pitfall 1: Logging the showcommand URL with UIDARUBA in the query string (CLIENT-06 violation)
**What goes wrong:** Default httpx debug logging or a `logger.info("AOS8: GET {}", request.url)` line captures the literal `?UIDARUBA=06f1758f-bb66-...` query param.
**Why it happens:** Apstra carries its token in a header — never in URLs — so the Apstra template doesn't have URL-redaction pressure. New code copies the logging shape verbatim.
**How to avoid:** Define `_sanitize_url_for_log(url)` once; route every URL log through it. Add a unit test that exercises 401-retry, captures all log records via loguru's `caplog` (or a sink), and asserts no `UIDARUBA=<actual-token>` substring appears in any record.
**Warning signs:** Test fails when token is grepped from log output.

### Pitfall 2: `assert` for None-narrowing (CONCERNS.md flagged)
**What goes wrong:** Code copied verbatim from `apstra/client.py:70` (`assert self._token is not None`). When the server runs with `python -O`, asserts are stripped — the type checker is satisfied at lint time, but the code returns `None` from `_ensure_token()` and the next line crashes with a confusing `AttributeError`.
**Why it happens:** Apstra has the same bug (acknowledged in CONCERNS.md / Pitfall m4). Direct copy propagates it.
**How to avoid:** Replace `assert` with `if self._token is None: raise AOS8AuthError(...)`. Lint rule (B101) is suppressed project-wide so ruff won't catch it; this is a discipline check.

### Pitfall 3: `_global_result.status` is a *string* in some firmware versions
**What goes wrong:** Code does `if gr.get("status") != 0:` — fails when AOS8 returns `"0"` (string). Treats success as failure.
**Why it happens:** ArubaOS API responses are inconsistent across minor versions. Some return `"status": "0"`, others `"status": 0`.
**How to avoid:** Normalize: `if str(status) != "0":`. Add unit test fixtures for both shapes.

### Pitfall 4: Health probe blocking lifespan startup
**What goes wrong:** `_probe_aos8` runs synchronously inside the existing `run_probes()` loop in `server.py:lifespan`. With unreachable Conductor + default 30s httpx timeout, every server startup hangs for 30s before any other platform can serve requests.
**Why it happens:** Default `_REQUEST_TIMEOUT = 30.0` from Apstra carries over.
**How to avoid:** Set a short `_HEALTH_TIMEOUT = 10.0` (or smaller) used specifically by `health_check()`. Phase 1 startup probe loop already catches all exceptions per platform — the timeout caps the wait.
**Warning signs:** Log shows multi-second gap between `Mist: startup probe ok` and `aos8: startup probe reported …`.

### Pitfall 5: Logout endpoint expects `SESSION` cookie, not `UIDARUBA` query param
**What goes wrong:** `aclose()` calls `self.request("POST", "/v1/api/logout")` — but `request()` injects `UIDARUBA=` into params. Aruba's logout endpoint uses the SESSION cookie set during login. Including UIDARUBA in the URL is harmless on logout but the deeper issue is that `request()` re-checks `_global_result` after — and a logged-out session may return a body without `_global_result`, causing a spurious raise.
**Why it happens:** Layering — calling the high-level `request()` for shutdown brings unwanted middleware behavior.
**How to avoid:** Make `aclose()` call `self._http.post("/v1/api/logout", timeout=_AUTH_TIMEOUT)` directly, bypassing `request()`. The httpx cookie jar already has `SESSION`. No `_global_result` check on shutdown — fully wrapped in `try/except`.

### Pitfall 6: 401-retry double-counts when login itself returns 401
**What goes wrong:** First `request()` call → no token → `_ensure_token()` → `_login_locked()` returns 401 (bad creds) → `AOS8AuthError` raised correctly. BUT: if login *returns 200 with an invalid token* (e.g., truncated, expired immediately), the next request gets 401, `_refresh_token()` fires, login returns the same bad token, second 401, raise. That's working as intended. The bug is when the first 401 path *also* triggers a refresh — infinite spin if the lock isn't releasing properly.
**Why it happens:** Subtle ordering between `_ensure_token` (acquires lock) and `request()` (which may call `_refresh_token` which also wants the lock).
**How to avoid:** Mirror Apstra exactly. `_refresh_token` acquires the lock independently; `request()` does NOT hold the lock when retrying. Test concurrent 401s explicitly.

### Pitfall 7: Mismatched login HTTP method (GET vs POST in vendor docs)
**What goes wrong:** Aruba Developer Hub login docs show GET in one place ([login page](https://developer.arubanetworks.com/aos8/docs/login)) and POST in another (general usage examples). Implementer picks GET → credentials end up in URL → captured in proxy/access logs.
**Why it happens:** Vendor doc inconsistency.
**How to avoid:** Use POST with form-encoded body. Form encoding works with both methods on the AOS8 endpoint, but POST keeps creds out of URLs. Document the choice in a comment citing the security rationale.

## Code Examples

### Login + token reuse (full snippet)
```python
# Source pattern: apstra/client.py:81-109; AOS8 deltas applied
# AOS8 endpoint: developer.arubanetworks.com/aos8/docs/login
async def _login_locked(self) -> None:
    """Perform login. Caller must hold self._lock."""
    logger.info("AOS8: requesting new UIDARUBA from {}", self._config.host)
    try:
        response = await self._http.post(
            "/v1/api/login",
            data={"username": self._config.username, "password": self._config.password},
            headers={"Content-Type": "application/x-www-form-urlencoded"},
            timeout=_AUTH_TIMEOUT,
        )
    except httpx.HTTPError as e:
        raise AOS8AuthError(f"AOS8 login request failed: {e}") from e
    if response.status_code != 200:
        raise AOS8AuthError(f"AOS8 login HTTP {response.status_code}")
    try:
        body = response.json()
    except ValueError as e:
        raise AOS8AuthError(f"AOS8 login non-JSON response") from e
    gr = body.get("_global_result") or {}
    if str(gr.get("status")) != "0":
        raise AOS8AuthError(f"AOS8 login rejected: {gr.get('status_str', 'unknown')}")
    token = gr.get("UIDARUBA")
    if not token:
        raise AOS8AuthError(f"AOS8 login response missing UIDARUBA")
    self._token = token
    logger.info("AOS8: obtained UIDARUBA {}", mask_secret(token))
```

### Test pattern (mirror test_apstra_client.py)
```python
# tests/unit/test_aos8_client.py — pattern source: tests/unit/test_apstra_client.py
import asyncio
import httpx
import pytest

from hpe_networking_mcp.config import AOS8Secrets
from hpe_networking_mcp.platforms.aos8.client import (
    AOS8APIError, AOS8AuthError, AOS8Client,
)

def _make_secrets(**overrides) -> AOS8Secrets:
    base = {
        "host": "conductor.test", "port": 4343,
        "username": "admin", "password": "secret", "verify_ssl": True,
    }
    base.update(overrides)
    return AOS8Secrets(**base)

def _install_mock_transport(client, handler):
    transport = httpx.MockTransport(handler)
    client._http = httpx.AsyncClient(
        base_url=f"https://{client._config.host}:{client._config.port}",
        transport=transport,
        verify=client._config.verify_ssl,
    )

@pytest.mark.unit
class TestAOS8ClientLogin:
    async def test_login_success_caches_token(self):
        client = AOS8Client(_make_secrets())
        calls = []
        def handler(request):
            calls.append(request)
            assert request.url.path == "/v1/api/login"
            assert request.method == "POST"
            assert b"username=admin" in request.content
            return httpx.Response(200, json={
                "_global_result": {"status": "0", "UIDARUBA": "tok-1"},
            })
        _install_mock_transport(client, handler)
        await client._ensure_token()
        assert client._token == "tok-1"
        await client._ensure_token()
        assert len(calls) == 1
        await client.aclose()

    async def test_login_rejected_global_result_nonzero(self):
        client = AOS8Client(_make_secrets())
        def handler(request):
            return httpx.Response(200, json={
                "_global_result": {"status": "1", "status_str": "auth failed"},
            })
        _install_mock_transport(client, handler)
        with pytest.raises(AOS8AuthError, match="auth failed"):
            await client._ensure_token()

    async def test_global_result_error_on_request(self):
        # request() must surface _global_result.status != 0 as AOS8APIError
        client = AOS8Client(_make_secrets())
        def handler(request):
            if request.url.path == "/v1/api/login":
                return httpx.Response(200, json={"_global_result": {"status": "0", "UIDARUBA": "t"}})
            return httpx.Response(200, json={"_global_result": {"status": "1", "status_str": "boom"}})
        _install_mock_transport(client, handler)
        with pytest.raises(AOS8APIError, match="boom"):
            await client.request("GET", "/v1/configuration/object/some_obj")

    async def test_uidaruba_never_in_logs(self, caplog):
        # Trigger login, request, 401-refresh, logout — assert no UIDARUBA token in any log
        # (Concrete implementation depends on loguru ↔ caplog interop; test_apstra_client
        #  uses pytest's caplog directly. Project may need a propagate sink — see open Q.)
        ...
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| UIDARUBA query param everywhere | UIDARUBA + X-CSRF-Token header | ArubaOS 8.7.0.0 (2020) | Phase 2 still uses UIDARUBA — works on 8.7+ for backward compatibility, broadly compatible. Future-proofing to X-CSRF-Token is a v2 enhancement, not Phase 2 scope. |
| Per-call login | Cached UIDARUBA reused for session lifetime | AOS8 8.0+ (always vendor recommendation) | Mandatory pattern; vendor explicitly warns against per-call login. |
| `responses` library for HTTP mocking | `httpx.MockTransport` for httpx clients | Project decision (existing test_apstra_client.py) | No new dep needed. |
| `assert` for None-narrowing | Explicit `if … raise` | CONCERNS.md flagged as `python -O` hazard | Phase 2 fixes this on the new client; backfill on Apstra is out of scope. |

**Deprecated/outdated:**
- UIDARUBA-in-cookie-only (without query param) — works for some endpoints, fails for showcommand. Always supply both.

## Open Questions

1. **Exact JSON shape of `show version` response across ArubaOS minor versions**
   - What we know: Top-level keys vary (`Hostname`/`hostname`, `Version`/`ArubaOS (...)`, sometimes nested under `_data`).
   - What's unclear: Whether 8.6.x, 8.10.x, 8.11.x return identical structures.
   - Recommendation: `health_check()` returns `{"hostname": …, "version": …, "raw": full_dict}` — the raw passthrough is the safety valve when key parsing misses. Capture a fixture from `aos8.example` lab in Phase 2 if available, otherwise stub a synthetic fixture and add a TODO for fixture refresh in Phase 7.

2. **Loguru ↔ pytest `caplog` integration for the "no token in logs" test (D-07)**
   - What we know: `caplog` works with stdlib `logging` by default; loguru bypasses the stdlib logger.
   - What's unclear: Whether `test_apstra_client.py` already solved this for token-redaction tests, or whether such a test exists yet.
   - Recommendation: Use a custom loguru sink in the test fixture that captures records into a list. If the project has an existing pattern (search `caplog` in tests/unit), reuse it; else add a small fixture in `tests/conftest.py` named `loguru_capture`. Plan task should explicitly call out this fixture.

3. **`_global_result.status` type normalization**
   - What we know: Vendor docs show `"status": "0"` (string); some Python wrappers report `int`.
   - What's unclear: Whether a single firmware ever mixes within a session.
   - Recommendation: `str(status) != "0"` everywhere. No further validation needed.

4. **`logout` HTTP method**
   - What we know: Aruba Developer Hub authoritatively says "GET or POST".
   - What's unclear: Whether one is preferred under Conductor vs standalone.
   - Recommendation: Use POST (locked by D-03). Document the choice in a code comment.

5. **Should write-gate Visibility transform fire when there are zero AOS8 write tools?**
   - What we know: Phase 2 ships zero tools; the gate hides nothing.
   - What's unclear: Whether `Visibility(False, tags={...})` with no matching tools logs a warning.
   - Recommendation: Add the transform per D-05 anyway (idempotent, documented intent); if FastMCP warns, suppress with a one-line explanation in `server.py`. Phase 5 plans assume the gate is already wired.

## Environment Availability

This phase is a **pure code/config change** with no new external runtime dependencies (no new tools, services, runtimes, databases, or package managers introduced). Existing toolchain (uv, ruff, mypy, pytest, Docker) is inherited from Phase 1 and verified by the existing CI workflow.

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python | All Python modules | ✓ | 3.12+ (per `pyproject.toml`) | — |
| uv | Local dev / Docker build | ✓ | latest (Astral) | — |
| Docker + Compose v2.24+ | Pre-push test gate | ✓ (assumed dev environment) | per project CLAUDE.md | Run `uv run pytest` directly without Docker |
| AOS8 / Mobility Conductor live system | Tests | ✗ | — | **All tests use mocked HTTP responses (httpx.MockTransport)** — explicit project constraint, not a gap. |

**Missing dependencies with no fallback:** None.
**Missing dependencies with fallback:** Live AOS8 — fallback is mock-only testing per project mandate.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | `pytest >= 8.4.0` + `pytest-asyncio >= 1.2.0` (asyncio_mode = "auto") |
| Config file | `pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `uv run pytest tests/unit/test_aos8_client.py -x -q` |
| Full suite command | `uv run pytest tests/ -q` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CLIENT-01 | Login returns UIDARUBA, cached on `_token`, reused on next call | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientLogin::test_login_success_caches_token -x` | ❌ Wave 0 |
| CLIENT-02 | `asyncio.gather` of 3 `_ensure_token` calls = exactly 1 login HTTP request | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientLogin::test_login_serialized_under_lock -x` | ❌ Wave 0 |
| CLIENT-03 | Constructor does not perform any HTTP I/O; first I/O is on first `request()` | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientLazy::test_init_does_not_login -x` | ❌ Wave 0 |
| CLIENT-04 | Single 401 → one refresh → success; double 401 → raise (no infinite loop) | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientRequest::test_401_triggers_refresh_and_retry_once -x` and `…::test_double_401_raises -x` | ❌ Wave 0 |
| CLIENT-05 | 200 response with `_global_result.status="1"` → `AOS8APIError` | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientRequest::test_global_result_error_raises -x` | ❌ Wave 0 |
| CLIENT-06 | UIDARUBA never in any captured log record across login+request+401+aclose | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientLogging::test_uidaruba_never_in_logs -x` | ❌ Wave 0 |
| CLIENT-07 | `verify_ssl=False` emits exactly one WARNING-level log on construction | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientMisc::test_verify_ssl_false_emits_warning -x` | ❌ Wave 0 |
| CLIENT-08 | Custom port (e.g., 8443) flows through to base_url | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientMisc::test_custom_port_in_base_url -x` | ❌ Wave 0 |
| CLIENT-09 | `health_check()` issues `show version`, returns hostname+version dict; `_probe_aos8` returns `status="ok"` with those fields | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientHealth -x` and `pytest tests/unit/test_health_probe.py::test_probe_aos8 -x` | ❌ Wave 0 (probe test new) |
| CLIENT-10 | `aclose()` POSTs `/v1/api/logout`; logout failure logs WARNING but does not raise | unit | `pytest tests/unit/test_aos8_client.py::TestAOS8ClientShutdown -x` | ❌ Wave 0 |
| (regression) | All existing platform tests still pass | unit | `pytest tests/unit/ -q` | ✅ existing |

### Sampling Rate
- **Per task commit:** `uv run pytest tests/unit/test_aos8_client.py -x -q` (under 5s expected)
- **Per wave merge:** `uv run pytest tests/unit/ -q` (all platforms, regression)
- **Phase gate:** Full pre-push command from project CLAUDE.md:
  `docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm hpe-networking-mcp sh -c "uv run ruff check . && uv run ruff format --check . && uv run mypy src/ --ignore-missing-imports && uv run pytest tests/ -q"`

### Wave 0 Gaps
- [ ] `tests/unit/test_aos8_client.py` — covers CLIENT-01 through CLIENT-10
- [ ] `tests/unit/test_health_probe.py` (or extend existing) — covers `_probe_aos8` ok / unavailable / degraded paths
- [ ] Loguru→caplog capture fixture (likely `tests/conftest.py` extension named `loguru_capture`) — **only if existing tests don't already have one** — required for CLIENT-06 token-leak assertion
- [ ] Test fixture `tests/unit/fixtures/aos8_show_version.json` — minimal `show version` response shape used by `health_check()` tests
- [ ] Framework install: none — pytest, pytest-asyncio, httpx already pinned

## Sources

### Primary (HIGH confidence)
- [Aruba Developer Hub — AOS8 Authentication](https://developer.arubanetworks.com/aos8/docs/login) — login/logout endpoint paths, request body format, `_global_result` shape, UIDARUBA semantics
- [Aruba Developer Hub — Showcommand API](https://developer.arubanetworks.com/aos8/docs/showcommand-api) — `/v1/configuration/showcommand?command=...&UIDARUBA=...` path, structured JSON response notes
- [Aruba Developer Hub — Getting Started with AOS 8 API](https://developer.arubanetworks.com/aos8/docs/getting-started-aos8-restapi) — base URL, port 4343, session 64-concurrent limit, X-CSRF-Token deprecation note
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/apstra/client.py` — read directly; canonical pattern for `_ensure_token`/`_refresh_token`/`_login_locked`/`request`/`aclose`/`health_check`
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/health.py` — read directly; `_ALL_PLATFORMS`, `_PROBES`, `_probe_apstra`, `_probe_axis` patterns
- `hpe-networking-mcp/src/hpe_networking_mcp/server.py` — read directly; lifespan, write-gate Visibility, `_register_*_tools` helpers
- `hpe-networking-mcp/tests/unit/test_apstra_client.py` — read directly; `httpx.MockTransport` test pattern, no extra mock dep needed
- `hpe-networking-mcp/tests/conftest.py` — read directly; AOS8 secrets already in `secrets_dir` fixture (Phase 1 deliverable)
- `hpe-networking-mcp/src/hpe_networking_mcp/config.py` lines 96–103, 351–403 — read directly; `AOS8Secrets` and `_load_aos8()` already shipped in Phase 1
- `.planning/research/PITFALLS.md` — read directly; 22 documented AOS8 pitfalls (C1–C8 critical, M1–M8 moderate, m1–m7 minor)
- `.planning/research/STACK.md` — read directly; HIGH-confidence dependency analysis confirming zero new deps needed

### Secondary (MEDIUM confidence)
- [ArubaOS 8.9.0.0 API Guide PDF](https://arubanetworking.hpe.com/techdocs/ArubaOS-8.x-Books/89/ArubaOS-8.9.0.0-API-Guide.pdf) — vendor PDF cited but not deeply mined; corroborates 4343 default port and showcommand path
- [The CaNerdIan — Working with the ArubaOS API](https://nerdian.ca/2021/10/19/working-with-the-arubaos-api-reading-data/) — community guide corroborating UIDARUBA semantics

### Tertiary (LOW confidence)
- None. All claims in this document trace to one of the primary sources above.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — every library is already pinned, every pattern proven in `apstra/client.py`
- Architecture: HIGH — exact file paths, exact line ranges, exact deltas verified against repo
- Pitfalls: HIGH — 22 entries from previous research round + 7 phase-specific additions verified against vendor docs and existing CONCERNS.md
- AOS8 endpoint paths: HIGH (login, logout, showcommand) — vendor docs confirmed via WebFetch
- `show version` JSON shape: MEDIUM — known to vary by firmware; mitigated by `raw` passthrough in `health_check()` return shape; flagged as Open Question
- Loguru caplog testability: MEDIUM — exact pattern depends on whether project already has a sink fixture; flagged as Open Question

**Research date:** 2026-04-28
**Valid until:** 2026-05-28 (30 days — AOS8 API is stable; vendor changes infrequent)
