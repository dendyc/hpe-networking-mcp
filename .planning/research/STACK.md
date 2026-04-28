# Technology Stack — AOS8 Platform Module

**Project:** hpe-networking-mcp — AOS8 / Mobility Conductor extension
**Researched:** 2026-04-27
**Scope:** Net-new dependencies and patterns for the `platforms/aos8/` module ONLY. Existing server, FastMCP, middleware, secrets, and Docker stack are inherited unchanged from the parent project.
**Overall confidence:** HIGH (all decisions align with existing, working patterns in the same codebase)

---

## Executive Summary

The AOS8 module needs **zero new dependencies**. Every requirement — async HTTPS, cookie-based session auth, self-signed-cert tolerance, concurrent-safe token caching, JSON parsing, structured logging, Pydantic models, mocked HTTP tests — is already covered by the libraries pinned in `pyproject.toml` for the existing Apstra, GreenLake, and Axis platforms.

The Apstra client (`platforms/apstra/client.py`) is the correct template. Apstra and AOS8 share the critical traits: async `httpx.AsyncClient`, login-returns-token, no refresh endpoint, asyncio-lock-serialized token acquisition, configurable `verify_ssl`, lifespan-managed singleton. The two material differences are (1) AOS8 carries the token both as a cookie (`SESSION`) and as a query parameter (`UIDARUBA=<token>`) instead of a header, and (2) AOS8 only supports GET/POST.

**Recommendation:** Build `platforms/aos8/client.py` as a near-copy of `apstra/client.py` with three deltas — cookie-based session storage via `httpx.AsyncClient`'s built-in cookie jar, automatic `UIDARUBA` query-param injection on every authenticated request, and a `request()` signature restricted to GET/POST.

---

## Recommended Stack (AOS8 module)

### HTTP Client

| Technology | Version | Purpose | Why |
|---|---|---|---|
| `httpx` | `>= 0.28.0` (already pinned) | Async HTTPS client for AOS8 REST API | Already used by Apstra, GreenLake, Axis. Built-in async cookie jar, `verify=False` for self-signed certs, per-call timeout, `params=` query injection, automatic JSON parsing. No alternative fits the existing async lifespan pattern. |

**`httpx` vs `requests`:** **httpx, definitively.** The entire server is async (FastMCP lifespan, `@mcp.tool()` async functions, `asyncio.Lock` patterns). `requests` is sync-only — using it would force `asyncio.to_thread()` wrappers on every call and break the concurrency model. `requests` is also not in the lockfile and adding it would be a regression. Every other custom-built platform client (Apstra, GreenLake, Axis) uses `httpx.AsyncClient`; AOS8 must follow suit for consistency.

### Session & Token Management

| Concern | Pattern | Source |
|---|---|---|
| Singleton client per server lifetime | `lifespan_context["aos8_client"]`, retrieved via `get_aos8_client()` helper | Mirrors `get_apstra_client()` in `apstra/client.py:173-188` |
| Concurrent-safe token acquisition | `asyncio.Lock` serializes login; double-checked-locking pattern in `_ensure_token()` | `apstra/client.py:63-79` |
| Token storage | In-memory `self._token: str \| None` + `httpx.AsyncClient` cookie jar (auto-populated by login response) | New — combines Apstra pattern with httpx cookie support |
| 401 handling | Single retry: clear token, re-login, replay request once | `apstra/client.py:154-159` |
| No proactive refresh | AOS8 has no refresh endpoint; only recourse on 401 is re-login. Same as Apstra. | AOS8 API behavior |
| Token transport | Append `UIDARUBA=<token>` to `params` on every authenticated request **and** rely on `SESSION` cookie set by login response | AOS8 docs (per project context); httpx auto-handles cookie persistence within a single `AsyncClient` |

**Token reuse is mandatory.** AOS8 documentation explicitly warns against per-call login (session-table exhaustion on the Mobility Conductor). The `_ensure_token()` / `_refresh_token()` split from Apstra solves this exactly — token is acquired once on first use, cached for the life of the client (server lifetime), refreshed only on 401.

### SSL Verification

| Setting | Default | Override |
|---|---|---|
| `httpx.AsyncClient(verify=...)` | `verify=config.verify_ssl` (default `True`) | Operator sets `aos8_verify_ssl=false` via Docker secret for self-signed certs |

Same exact pattern as `apstra/client.py:51` (`verify=config.verify_ssl`). No new code needed beyond plumbing the secret through `config.py:AOS8Secrets`. Self-signed certs are common in on-prem AOS8 deployments, so the override must work end-to-end.

### Configuration & Validation

| Library | Version | Purpose |
|---|---|---|
| `pydantic` | `>= 2.12.0` (already pinned) | `AOS8Secrets` dataclass/model in `config.py`; Pydantic models for AOS8 response shapes (controller info, AP info, client info, alerts) in `platforms/aos8/models.py` if structured returns are needed |
| `pydantic-settings` | `>= 2.11.0` (already pinned) | Not used directly — secrets come from `_read_secret()` files, not env-var settings — but available if needed |

### Logging & Observability

| Library | Version | Purpose |
|---|---|---|
| `loguru` | `>= 0.7.3` (already pinned) | All client + tool logging; `mask_secret()` from `utils.logging` for the UIDARUBA token in log lines (same as Apstra masks its AuthToken) |

### Error Handling

| Library | Purpose |
|---|---|
| `httpx.HTTPStatusError`, `httpx.HTTPError` | Raised by `response.raise_for_status()` and transport layer; consumed by tool-level `except Exception` blocks |
| `format_http_error()` (existing in `apstra/client.py:191-206`) | Reusable helper — copy-and-adapt into `aos8/client.py` (or factor into a shared utility if it makes sense; not required for this milestone) |
| Custom `AOS8AuthError(RuntimeError)` | New — mirrors `ApstraAuthError`. Raised on login failures distinct from regular HTTP errors. |
| `fastmcp.exceptions.ToolError` | Raised by `get_aos8_client()` when the platform is disabled / client absent (returns 503 to the LLM) |

### Testing

| Library | Version | Purpose |
|---|---|---|
| `pytest` | `>= 8.4.0` (already pinned) | Test runner; mark with `@pytest.mark.unit` |
| `pytest-asyncio` | `>= 1.2.0` (already pinned) | `asyncio_mode = "auto"` already set; async tests work out of the box |
| `responses` | `>= 0.25.7` (already pinned) | HTTP mocking. **Note:** `responses` mocks `requests`, not `httpx`. For httpx mocking the existing test suite uses `respx` or `pytest-httpx`. **Verify which one is actually in `pyproject.toml`'s test deps before writing tests** — if neither, use `pytest-httpx` (most common httpx mocking lib) and check whether existing Apstra/GreenLake/Axis tests already pull it in transitively. |

> **PITFALL FLAG for roadmap:** The existing `STACK.md` lists `responses` for HTTP mocking, but `responses` only patches the `requests` library. The Apstra, GreenLake, and Axis tests must be using something else (likely `respx` or `pytest-httpx`, or they patch `httpx.AsyncClient.request` directly with `unittest.mock`). Phase-1 of AOS8 work should **read one existing httpx-based test** (e.g. `tests/unit/test_apstra_client.py` if it exists) and copy that exact mocking strategy. Do not introduce a new mocking library.

---

## Pattern: AOS8 Client Skeleton

Direct copy-and-modify of `apstra/client.py`. Key shape (not full implementation — that's the build phase):

```python
class AOS8Client:
    def __init__(self, config: AOS8Secrets) -> None:
        self._config = config
        self._token: str | None = None
        self._lock = asyncio.Lock()
        base_url = f"https://{config.host}:{config.port}"  # default port 4343
        self._http = httpx.AsyncClient(
            base_url=base_url,
            verify=config.verify_ssl,
            timeout=_REQUEST_TIMEOUT,
            # cookies kept in the client's jar automatically across requests
        )

    async def _login_locked(self) -> None:
        # POST /v1/api/login with username/password as form data
        # Response sets SESSION cookie automatically + body has {"_global_result": {"UIDARUBA": "<token>"}}
        # Store token in self._token; cookie jar already updated by httpx
        ...

    async def request(
        self,
        method: Literal["GET", "POST"],   # AOS8 supports only GET/POST
        path: str,
        *,
        params: dict[str, Any] | None = None,
        json_body: Any = None,
        timeout: float | None = None,
    ) -> httpx.Response:
        token = await self._ensure_token()
        merged_params = {"UIDARUBA": token, **(params or {})}
        # ... send request, on 401 refresh token + retry once, raise_for_status
```

The shape of `request()` is the only material divergence from Apstra: query-param token instead of header token, and a method whitelist.

---

## Alternatives Considered

| Concern | Recommended | Alternative | Why Not |
|---|---|---|---|
| HTTP client | `httpx` (async) | `requests` (sync) | Server is async end-to-end; sync would require thread-pool wrappers and break the established lifespan/lock pattern. |
| HTTP client | `httpx` | `aiohttp` | `httpx` already in lockfile; `aiohttp` would be a new dep with no benefit. Consistency with 3 other platform clients. |
| Vendor SDK | None — raw `httpx` | `pyaoscx` (Aruba's Python SDK) | `pyaoscx` targets **AOS-CX switches**, not AOS8 wireless controllers. Different product, different API. **No official Python SDK exists for AOS8 / Mobility Conductor.** Raw httpx is the standard approach (consistent with how the community wraps the AOS8 API). |
| Token transport | Cookie jar + query-param `UIDARUBA` | Header-only | AOS8 API requires `UIDARUBA` as query param on most endpoints per the API spec; cookie alone is insufficient on some endpoints. Belt-and-suspenders is the documented pattern. |
| Concurrency primitive | `asyncio.Lock` | `asyncio.Semaphore` / no lock | Lock is the minimum sufficient primitive — only one login can be in-flight at a time; reads after login don't need serialization. Matches Apstra. |
| Error type | Custom `AOS8AuthError` + `httpx.HTTPStatusError` | Generic `Exception` | Distinguishes auth failure (re-check creds) from 5xx (retry middleware) from 4xx (caller error). Matches Apstra. |
| Token storage | In-memory only | Disk / Redis / file | Server is single-process; in-memory is simpler, more secure (no token-at-rest), and matches every other platform. Re-login on container restart is acceptable. |

---

## Installation

**No new packages.** All required libraries are already pinned in `pyproject.toml` and present in `uv.lock`. No `uv add` is needed for this milestone.

**Verification step (run before starting build):**

```bash
# Confirm the existing pyproject.toml is sufficient — no dep changes expected
docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm hpe-networking-mcp \
  sh -c "uv pip list | grep -E '^(httpx|pydantic|loguru|pytest|pytest-asyncio)'"
```

The only file-system additions for this milestone are:
- `src/hpe_networking_mcp/platforms/aos8/` (new package)
- `tests/unit/test_aos8_*.py` (new tests)
- New entries in `secrets/` (operator-supplied at deploy time)
- New `AOS8Secrets` block in `config.py`
- New `aos8_*` env vars (e.g. `ENABLE_AOS8_WRITE_TOOLS`)

---

## Quality Gate Verification

| Gate | Status | Notes |
|---|---|---|
| Confirm httpx vs requests recommendation with reasoning | ✓ | httpx — async-required; consistent with Apstra/GreenLake/Axis; no sync wrappers; already pinned. |
| Token/session management pattern clearly defined | ✓ | `asyncio.Lock` + cached `self._token` + httpx cookie jar; lazy acquire on first use; refresh once on 401; no proactive refresh. Lifespan singleton. |
| SSL verification approach documented | ✓ | `httpx.AsyncClient(verify=config.verify_ssl)`; default `True`; operator overrides via `aos8_verify_ssl` Docker secret. Same pattern as Apstra. |
| No new dependencies needed beyond pyproject.toml | ✓ | Verified against existing `STACK.md` — httpx, pydantic, loguru, pytest, pytest-asyncio all already pinned. **One open item:** confirm the httpx mocking library (likely `respx` or `pytest-httpx`) used by existing Apstra/GreenLake/Axis tests; do not introduce a new one. |

---

## Sources & Confidence

| Claim | Source | Confidence |
|---|---|---|
| Apstra client uses async httpx + asyncio.Lock + lifespan singleton | `platforms/apstra/client.py` (read directly) | HIGH |
| httpx, pydantic, loguru, pytest, pytest-asyncio, responses already pinned | `.planning/codebase/STACK.md` (read directly) | HIGH |
| AOS8 requires UIDARUBA query param + cookie session, GET/POST only, verify_ssl flag | Project context block in research prompt (already-validated upstream research) | HIGH |
| Token reuse mandatory (session exhaustion risk) | Project context block; AOS8 vendor documentation pattern | HIGH |
| No official Python SDK for AOS8 / Mobility Conductor | Aruba publishes `pyaoscx` for AOS-CX switches only; no equivalent for AOS8 wireless controllers | MEDIUM (training-data + ecosystem knowledge; verify by `pip search` / PyPI before build phase if any concern) |
| `responses` library mocks `requests`, not `httpx` | Library documentation (well-established) | HIGH |
| Existing httpx-platform tests must use `respx` or `pytest-httpx` or direct `unittest.mock` patching | Inferred — not verified by reading a test file | MEDIUM — flagged as a Phase-1 verification step |

---

## Open Items for Build Phase

1. **Verify httpx mocking library** — read one existing httpx-platform test file (e.g. `tests/unit/test_apstra_*.py` or `tests/unit/test_greenlake_*.py`) and adopt the exact same mocking approach for AOS8 tests. Don't introduce a new lib.
2. **Confirm AOS8 login response shape** — the build phase needs to know whether the token is at `body["_global_result"]["UIDARUBA"]` or elsewhere. Existing project context implies this format; first-implementation should log the raw response (with token masked) on first use to confirm.
3. **Confirm `format_http_error()` reuse** — either copy verbatim into `aos8/client.py` or extract into `utils/http_errors.py` for shared use across Apstra + AOS8. Either is fine; pick during build.
