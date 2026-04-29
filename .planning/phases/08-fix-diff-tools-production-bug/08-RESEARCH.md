# Phase 8: Fix DIFF Tools Production Bug - Research

**Researched:** 2026-04-28
**Domain:** Python async test mocking — `httpx.Response` contract / AsyncMock with side-effect-bearing methods
**Confidence:** HIGH

## Summary

This is a 2-line response-contract bug in `src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py`. The local helpers `_show()` (line 66) and `_object()` (line 81) treat the return value of `client.request(...)` as a parsed JSON body and pass it directly to `strip_meta()`. But `AOS8Client.request()` (declared at `client.py:210`) returns `httpx.Response`, not a dict. In production, `strip_meta()` receives an `httpx.Response`, fails the `isinstance(body, dict)` check, and returns the `Response` object unchanged — every one of the 9 DIFF tools therefore returns a non-serialisable `httpx.Response` to the AI client.

The 13 DIFF unit tests in `tests/unit/test_aos8_read_differentiators.py` masked the bug because the mock helper `_make_ctx()` (line 26) does `client.request = AsyncMock(return_value=body)` — i.e. the mock returns the body dict directly. `_show()` then "works" against the test mock but is broken in production. STATE.md `[Phase 07]` decision logs the original (incorrect) rationale: *"differentiators.py uses local _show/_object helpers (direct client.request return) because frozen test mocks return dict directly, not httpx.Response."* That is the bug — the code was written to match the test mock instead of fixing the mock.

The canonical reference implementation already exists in the same module tree: `src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` lines 41–86 (`run_show` / `get_object`) call `response.json()` on the return value. All 26 existing READ tools use these helpers and work correctly. Phase 7 chose to introduce duplicate local helpers in `differentiators.py` instead of reusing the canonical helpers — that duplication is what diverged.

**Primary recommendation:** Delete the local `_show()` / `_object()` helpers in `differentiators.py` and import `run_show` / `get_object` from `_helpers`. Update all 13 DIFF test mocks to use the standard `MagicMock()` + `mock_response.json.return_value = body` pattern. This restores response-contract uniformity across all 35 AOS8 read tools and removes the parallel-helper duplication.

## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DIFF-01 | `aos8_get_md_hierarchy` — Conductor → MD hierarchy tree | Existing tool body unchanged; only `_show` helper is rewired to call `.json()`; test mock updated |
| DIFF-02 | `aos8_get_effective_config` — resolved config at `config_path` | Existing tool body unchanged; `_object` helper rewired; 2 test mocks updated |
| DIFF-03 | `aos8_get_pending_changes` — staged changes not yet `write_memory`'d | Tool body unchanged; `_show` helper rewired; test mock updated |
| DIFF-04 | `aos8_get_rf_neighbors` — ARM neighbor graph for an AP | Tool body unchanged; helper rewired; test mock updated |
| DIFF-05 | `aos8_get_cluster_state` — LC-cluster master/standby | Tool body unchanged; helper rewired; test mock updated |
| DIFF-06 | `aos8_get_air_monitors` — air-monitor mode APs | Tool body unchanged; helper rewired; test mock updated |
| DIFF-07 | `aos8_get_ap_wired_ports` — wired-port state for an AP | Tool body unchanged; helper rewired; test mock updated |
| DIFF-08 | `aos8_get_ipsec_tunnels` — site-to-site / RAP IPsec state | Tool body unchanged; helper rewired; test mock updated |
| DIFF-09 | `aos8_get_md_health_check` — composite multi-call aggregator | Tool body unchanged; partial-failure path preserved; 2 composite tests update their `_route(...)` mocks to return Response-like objects with `.json()` |

## Project Constraints (from CLAUDE.md)

The following CLAUDE.md directives must be honored:

- **API surface:** GET/POST only; this fix changes nothing — same `client.request("GET", ...)` calls.
- **UIDARUBA token reuse:** Already handled by `AOS8Client._ensure_token()`; no change.
- **Testing:** No live AOS8 system — all tests use mocked HTTP responses (preserved).
- **Compatibility:** Must not break any existing platform module or test. The 764-test full suite must remain GREEN with zero regressions.
- **Style:** Ruff lint, ruff format, mypy clean. Max 500 lines/file (`differentiators.py` is currently 354 — well under). Google-style docstrings preserved.
- **Commits:** "Never include claude code, or written by claude code in commit messages." Honor this if any commit step happens during execution.
- **Documentation Checklist:** This fix touches a tool's response shape — the user-visible contract narrows from "raw httpx.Response" to "parsed JSON dict". If anything in INSTRUCTIONS.md / docs/TOOLS.md / CHANGELOG.md described the broken contract it must be updated; if those docs already describe parsed JSON (which is the AI-correct shape), no doc update is required beyond CHANGELOG. The planner should verify by `grep -n "differentiator\|DIFF\|httpx.Response" INSTRUCTIONS.md docs/TOOLS.md CHANGELOG.md README.md` and only update files that mention the response contract.
- **Branch / PR hygiene:** Do NOT push directly to `main`; use a `fix/diff-tools-response-contract` branch with a PR. Pre-push checklist must run in Docker per CLAUDE.md.

## Standard Stack

This phase introduces no new dependencies. All required libraries are already in `pyproject.toml`:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `pytest` | >= 8.4.0 | Test runner | Already in `pyproject.toml`, used by all 764 existing tests |
| `pytest-asyncio` | >= 1.2.0 | `asyncio_mode = "auto"` | Already configured; DIFF tests are `async def` |
| `unittest.mock` | stdlib | `MagicMock` / `AsyncMock` | Standard for mocking sync/async return values; already used in DIFF tests |
| `httpx` | >= 0.28.0 | Source of `httpx.Response` | Already a top-level dep; `Response.json()` is its canonical body-parse method |

### Supporting
None required — the entire fix is internal.

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `MagicMock(); m.json.return_value=body` | A real `httpx.Response(200, json=body)` | More realistic but heavier; `MagicMock` matches the rest of the AOS8 test suite (see `test_aos8_read_*.py` in tests/unit) |
| Reuse `_helpers.run_show` / `get_object` | Keep a corrected local `_show` / `_object` | Reuse is strictly better — removes duplicated helpers, restores symmetry across 35 read tools, deletes ~30 lines |

**Installation:** No installs required. Verify dev environment:
```bash
cd hpe-networking-mcp
uv sync --frozen
uv run pytest tests/unit/test_aos8_read_differentiators.py -q
```

**Version verification:** All deps already locked in `uv.lock`. No registry checks needed for this phase.

## Architecture Patterns

### Recommended Project Structure (no change)
```
src/hpe_networking_mcp/platforms/aos8/
├── _registry.py
├── client.py                       # AOS8Client.request() → httpx.Response (canonical)
└── tools/
    ├── _helpers.py                 # run_show / get_object — call .json() on response
    ├── differentiators.py          # ← fix here (use _helpers; drop local _show/_object)
    ├── reads_health.py             # already uses run_show / get_object correctly
    ├── reads_clients.py
    ├── reads_alerts.py
    ├── reads_wlan.py
    ├── reads_troubleshoot.py
    └── writes.py                   # uses post_object (also calls .json())
```

### Pattern 1: Canonical AOS8 read-tool response-handling pattern
**What:** Every AOS8 read tool calls `await run_show(client, ...)` or `await get_object(client, ..., config_path=...)`. Both helpers internally invoke `client.request(...)` (which returns `httpx.Response`), then call `response.json()` to obtain the parsed body, then call `strip_meta()` to drop `_meta` / `_global_result`.

**When to use:** Always — for every AOS8 tool that consumes a JSON body from the controller.

**Example:**
```python
# Source: src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py:41-63 (existing, correct)
async def run_show(
    client: AOS8Client,
    command: str,
    *,
    config_path: str | None = None,
) -> Any:
    params: dict[str, Any] = {"command": command}
    if config_path is not None:
        params["config_path"] = config_path
    response = await client.request("GET", "/v1/configuration/showcommand", params=params)
    return strip_meta(response.json())   # ← .json() is the load-bearing call
```

After the fix, `differentiators.py` should look like:
```python
# Target shape — replaces lines 42-86 of differentiators.py
from hpe_networking_mcp.platforms.aos8.tools._helpers import (
    format_aos8_error,
    get_object,
    run_show,
    strip_meta,  # only kept if used elsewhere; otherwise drop
)

# DIFF-01 example — body of tool unchanged except helper name:
@tool(name="aos8_get_md_hierarchy", annotations=READ_ONLY)
async def aos8_get_md_hierarchy(ctx: Context) -> dict[str, Any] | str:
    client = ctx.lifespan_context["aos8_client"]
    try:
        return await run_show(client, "show switch hierarchy")
    except Exception as exc:  # noqa: BLE001
        return format_aos8_error(exc, "fetch MD hierarchy")
```

### Pattern 2: Canonical AsyncMock pattern for `httpx.Response`-returning callables
**What:** When mocking a method whose contract returns `httpx.Response`, the mock's *return value* must itself expose a `.json()` method. With `unittest.mock`:
```python
mock_response = MagicMock()
mock_response.json.return_value = body            # parsed JSON dict
client = MagicMock()
client.request = AsyncMock(return_value=mock_response)
```
`AsyncMock(return_value=...)` only awaits the OUTER call — `await client.request(...)` returns whatever `return_value` is set to. The `.json()` call on that return value is synchronous (matching the real `httpx.Response.json()` signature in httpx 0.28+).

**When to use:** Every test in `test_aos8_read_differentiators.py`. This is identical to the pattern already used in the other AOS8 read-tool tests (e.g. `tests/unit/test_aos8_read_health.py`).

**Example for the composite DIFF-09 test** (which uses `side_effect`, not `return_value`):
```python
# Existing buggy mock (test_get_md_health_check, line 228-243):
def _route(method, path, params=None, **kwargs):
    cmd = (params or {}).get("command", "")
    if cmd == "show ap active":
        return aps_active            # ← returns dict directly
    ...
client.request = AsyncMock(side_effect=_route)

# Fixed mock — wrap each branch in a Response-like MagicMock:
def _route(method, path, params=None, **kwargs):
    cmd = (params or {}).get("command", "")
    response = MagicMock()
    if cmd == "show ap active":
        response.json.return_value = aps_active
    elif cmd == "show ap database":
        response.json.return_value = aps_db
    elif cmd == "show alarms all":
        response.json.return_value = alarms
    elif cmd == "show version":
        response.json.return_value = version
    elif cmd == "show user summary":
        response.json.return_value = users
    else:
        response.json.return_value = {"_global_result": {"status": "0"}}
    return response                  # AsyncMock(side_effect=...) awaits this

client.request = AsyncMock(side_effect=_route)
```

A small helper local to the test file would reduce repetition:
```python
def _resp(body: dict) -> MagicMock:
    r = MagicMock()
    r.json.return_value = body
    return r
```
…which the planner may choose to introduce in `_make_ctx()` so all single-call tests stay one-line.

### Anti-Patterns to Avoid
- **Anti-pattern: Adding a `.json()` shim on the dict return value.** Tempting but wrong. Mock the outer object, not the body shape. Each helper expects `Response.json()`; faking that anywhere except on the response object obscures the contract.
- **Anti-pattern: Keeping local `_show` / `_object` "for symmetry with the docstring".** The docstrings on these local helpers explicitly say *"matching the test contract for differentiator tools where mocks return a dict (not an httpx.Response)"* — that comment encodes the bug. Delete the helpers; the canonical helpers in `_helpers.py` are correct.
- **Anti-pattern: Patching tests to assert on `result.json()` calls.** The current tests assert on the *parsed body* (e.g. `result["Switch Hierarchy"][0]["Name"] == "MC-01"`). Those assertions are correct; only the *mock setup* needs to change.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Showcommand response unwrapping | A new `_show` helper local to `differentiators.py` | `_helpers.run_show` | Already handles `.json()` + `strip_meta`; symmetric across all 26 read tools; tested; documented |
| Config-object response unwrapping | A new `_object` helper local to `differentiators.py` | `_helpers.get_object` | Same reason; `default config_path="/md"` already lives there |
| Response-mock builder | Inline 3-line `MagicMock` setup repeated 13 times | A single `_resp(body)` helper at the top of the test file (≤4 lines) | DRY; one place to update if the response contract evolves |

**Key insight:** The bug exists *because* parallel helpers were introduced. The fix is consolidation, not patching.

## Runtime State Inventory

This is a code-fix phase — no rename, no datastore change, no service-config change.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | None — verified by inspecting `differentiators.py`, `_helpers.py`, and `client.py`. No DB, file, or cache stores DIFF tool output. | None |
| Live service config | None — no n8n/Datadog/Tailscale/Cloudflare integration touches AOS8. | None |
| OS-registered state | None — no Task Scheduler / pm2 / systemd entries reference AOS8 tool names. | None |
| Secrets/env vars | None — `aos8_*` secrets and `ENABLE_AOS8_WRITE_TOOLS` are unaffected; only read tools change behavior. | None |
| Build artifacts | None — `pyproject.toml` version may bump (patch level) for the fix release; no egg-info / Docker tag rename required mid-phase. | Optional version bump (planner decides) |

## Common Pitfalls

### Pitfall 1: Forgetting that `AsyncMock(return_value=X)` does not auto-await `X`
**What goes wrong:** Authors set `mock_response = AsyncMock(); mock_response.json = AsyncMock(return_value=body)` because everything else in the file is async — turning `.json()` into an awaitable.
**Why it happens:** `httpx.Response.json()` is synchronous in httpx 0.28+, but the surrounding context is async, so the instinct is to make it async too.
**How to avoid:** Use `MagicMock` (sync) for the response itself; only `client.request` is `AsyncMock`. The helper's call site is `response.json()`, never `await response.json()`.
**Warning signs:** `TypeError: object MagicMock can't be used in 'await' expression`, or tests passing while production fails because the helper does not `await` `.json()`.

### Pitfall 2: Composite test (`test_get_md_health_check`) `side_effect` still returning dicts
**What goes wrong:** Plan only updates the simple `_make_ctx` helper, leaves the `_route` closures in the two composite tests untouched. The aggregator `aos8_get_md_health_check` then crashes on `strip_meta(response.json())` for every sub-call once `_show` is fixed.
**Why it happens:** The composite tests have their own inline mock setup; they don't go through `_make_ctx`.
**How to avoid:** Two composite tests at lines 219–258 and 269–307 BOTH need their `_route(...)` closures updated. Plan must list both explicitly.
**Warning signs:** `test_get_md_hierarchy` etc. green, but `test_get_md_health_check_*` fail with `AttributeError: 'dict' object has no attribute 'json'`.

### Pitfall 3: `strip_meta` silently passing through non-dicts
**What goes wrong:** `_helpers.strip_meta()` (line 36) does `if not isinstance(body, dict): return body` — exactly the behavior that hides the production bug. After the fix, that defensive branch still exists and could re-hide a future regression.
**Why it happens:** Defensive coding for non-dict responses (some AOS8 endpoints return lists).
**How to avoid:** Don't change `strip_meta` — its non-dict pass-through is correct for list responses (e.g. `_data: [...]`). The fix is upstream of `strip_meta`: ensure `.json()` is always called first.
**Warning signs:** Future regression in another tool returning `httpx.Response` will silently pass through; rely on assertion-on-payload tests (already present — e.g. `result["Switch Hierarchy"][0]["Name"] == "MC-01"`) to catch this. No additional guard recommended.

### Pitfall 4: Removing `strip_meta` import from `differentiators.py` when it's no longer directly used
**What goes wrong:** After `_show` / `_object` are deleted, `strip_meta` is no longer referenced in `differentiators.py`. Author leaves the import → ruff F401 (unused import). Author removes the import without checking → fine for now but breaks any future inline use.
**How to avoid:** After refactor, run `uv run ruff check src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` — drop unused imports.
**Warning signs:** Ruff F401, mypy clean. Trivial.

### Pitfall 5: `format_aos8_error` exception path — `httpx.Response.json()` can raise
**What goes wrong:** `Response.json()` raises `json.JSONDecodeError` if the body is non-JSON (e.g. controller returned HTML during a maintenance redirect). Currently masked by the bug; once `.json()` is wired in, this becomes a live failure mode.
**Why it happens:** Real AOS8 controllers occasionally return non-JSON during session expiry or maintenance windows.
**How to avoid:** `format_aos8_error` already handles `httpx.HTTPError` and a generic `Exception` fallback — `JSONDecodeError` (a `ValueError` subclass) lands in the generic branch, which is acceptable for v1. No code change required, but the planner should note this for v2 hardening.
**Warning signs:** Operator reports DIFF tool returns "Unexpected error while attempting to fetch X: Expecting value: line 1 column 1 (char 0)". Indicates non-JSON response, not a bug in the fix.

## Code Examples

### Production fix — minimal version (preferred: import canonical helpers)
```python
# src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py — top of file

from hpe_networking_mcp.platforms.aos8.tools._helpers import (
    format_aos8_error,
    get_object,
    run_show,
)

# Delete lines 42-86 (local _show, _object, _SHOWCOMMAND_PATH).
# Update each call site:
#   _show(client, cmd, config_path=cp)    →  run_show(client, cmd, config_path=cp)
#   _object(client, name, config_path=cp) →  get_object(client, name, config_path=cp)
# In aos8_get_md_health_check, replace 5x _show(...) calls with run_show(...).
```

### Production fix — alternate version (keep local helpers, just call .json())
If the planner prefers minimum diff:
```python
# Two-line patch — differentiators.py lines 66 and 81-85
# Line 66 was:
#     body = await client.request("GET", _SHOWCOMMAND_PATH, params=params)
#     return strip_meta(body)
# Becomes:
    response = await client.request("GET", _SHOWCOMMAND_PATH, params=params)
    return strip_meta(response.json())

# Lines 81-85 was:
#     body = await client.request("GET", f"/v1/configuration/object/{object_name}", params={...})
#     return strip_meta(body)
# Becomes:
    response = await client.request("GET", f"/v1/configuration/object/{object_name}", params={"config_path": config_path})
    return strip_meta(response.json())
```
The success criterion #1 explicitly accepts this minimal patch. **Recommendation:** still prefer reuse of `_helpers` to avoid future drift, but the 2-line patch is the literal scope described by the phase goal.

### Test mock fix — `_make_ctx` helper update
```python
# tests/unit/test_aos8_read_differentiators.py — lines 25-31
# Old:
def _make_ctx(body: dict):
    client = MagicMock()
    client.request = AsyncMock(return_value=body)
    ctx = MagicMock()
    ctx.lifespan_context = {"aos8_client": client}
    return ctx, client

# New:
def _make_ctx(body: dict):
    response = MagicMock()
    response.json.return_value = body
    client = MagicMock()
    client.request = AsyncMock(return_value=response)
    ctx = MagicMock()
    ctx.lifespan_context = {"aos8_client": client}
    return ctx, client
```

### Test mock fix — composite `_route` closure (DIFF-09)
See Pattern 2 above. Both `test_get_md_health_check` (line 219) and `test_get_md_health_check_handles_partial_failure` (line 269) need their `_route` updated. The partial-failure test's `RuntimeError` branch can stay as a `raise` — it propagates through `AsyncMock(side_effect=...)` correctly.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Local `_show` / `_object` returning `client.request` directly | `run_show` / `get_object` from `_helpers` calling `response.json()` | Phase 7 introduced the divergence; Phase 8 reverts | Restores 1 helper-pattern per HTTP shape across all 35 AOS8 read tools |
| `AsyncMock(return_value=dict)` | `MagicMock` response with `response.json.return_value = dict` | Already the convention in other AOS8 read-tool tests | Test mocks now exercise the real `.json()` call site |

**Deprecated/outdated:**
- The Phase 7 STATE.md note *"differentiators.py uses local _show/_object helpers (direct client.request return) because frozen test mocks return dict directly, not httpx.Response"* — Phase 8 explicitly invalidates this rationale. Planner should add a Phase 8 STATE.md decision overriding it.

## Open Questions

1. **Should the planner also bump `pyproject.toml` to 2.4.0.2 (or 2.4.1.0) and add a CHANGELOG entry as part of this phase?**
   - What we know: STATE.md `Open Todos` carries a deferred doc-refresh item from Phase 7 (D-06) covering README count 38→47, CHANGELOG, TOOLS.md, INSTRUCTIONS.md, pyproject.toml. The phase description here is narrowly "patch the bug" — but `CLAUDE.md` Documentation Checklist is mandatory for every code PR.
   - What's unclear: Whether the planner folds D-06 into Phase 8 plans or keeps the bug-fix tightly scoped and tracks docs as a separate plan within Phase 8.
   - Recommendation: Keep the bug fix in plan 08-01 (code + tests), and add a plan 08-02 for the deferred docs/CHANGELOG/version bump. This satisfies CLAUDE.md while keeping the bug-fix commit reviewable.

2. **Should `strip_meta` get a hardening guard that raises on non-dict-non-list inputs (e.g. `httpx.Response`) to prevent regression?**
   - What we know: Today's `strip_meta` returns non-dicts unchanged, masking exactly the class of bug we are fixing.
   - What's unclear: Whether other AOS8 endpoints legitimately return non-dict bodies (e.g. plaintext); a hard guard could break them.
   - Recommendation: Defer. The phase's job is to fix DIFF tools, not redesign envelope handling. Track as a separate hardening idea.

3. **Are there OTHER places in the codebase with the same `client.request` direct-pass bug?**
   - What we know: Grep of `src/hpe_networking_mcp/platforms/aos8/tools/` — only `differentiators.py` defines local helpers; all other read tools route through `_helpers.run_show` / `get_object` and the write path uses `_helpers.post_object`. Phase 7 added the divergence in only one file.
   - What's unclear: Nothing — the bug is contained.
   - Recommendation: Plan should still include a smoke `grep -rn "await client\.request" src/hpe_networking_mcp/platforms/aos8/` step in verification to confirm no other tool re-enters the bug pattern.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python | All code | ✓ | 3.12+ (project requirement; verified by `python:3.12-slim-bookworm` Docker base) | — |
| uv | Dependency / test runner | ✓ | latest (project standard) | — |
| pytest + pytest-asyncio | Test execution | ✓ | 8.4.0+ / 1.2.0+ (locked in `uv.lock`) | — |
| ruff | Lint + format gate | ✓ | 0.15.6+ | — |
| mypy | Type check gate | ✓ | 1.18.0+ | — |
| Docker + Compose v2.24+ | CLAUDE.md pre-push checklist | Assumed available — project is Docker-native | latest | Run pytest/ruff/mypy directly via `uv run …` if Docker is offline locally; CI re-runs in Docker |
| Live AOS8 controller | Manual integration smoke | ✗ — explicitly not required by project rule "all tests use mocked HTTP responses" | — | Not applicable — phase is verified entirely via mocked unit tests |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:** Live AOS8 not required (project policy).

## Validation Architecture

> Phase 8 is a TDD bug-fix phase. `.planning/config.json` includes `workflow.nyquist_validation` (verify with planner — included by default if absent).

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest 8.4.0+ with pytest-asyncio (`asyncio_mode = "auto"`) |
| Config file | `pyproject.toml` `[tool.pytest.ini_options]` (path: `hpe-networking-mcp/pyproject.toml`) |
| Quick run command | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_read_differentiators.py -x -q` |
| Full suite command | `cd hpe-networking-mcp && uv run pytest tests/unit -q` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| DIFF-01 | `aos8_get_md_hierarchy` calls `.json()`; returns parsed body without `_meta` | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_md_hierarchy -x` | ✅ |
| DIFF-02 | `aos8_get_effective_config` parses body; `config_path` default and override | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_effective_config_required_object_name tests/unit/test_aos8_read_differentiators.py::test_get_effective_config_defaults_config_path -x` | ✅ |
| DIFF-03 | `aos8_get_pending_changes` parses body | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_pending_changes -x` | ✅ |
| DIFF-04 | `aos8_get_rf_neighbors` correct command + config_path | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_rf_neighbors -x` | ✅ |
| DIFF-05 | `aos8_get_cluster_state` correct command | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_cluster_state -x` | ✅ |
| DIFF-06 | `aos8_get_air_monitors` correct command | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_air_monitors -x` | ✅ |
| DIFF-07 | `aos8_get_ap_wired_ports` correct command interpolation | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_ap_wired_ports -x` | ✅ |
| DIFF-08 | `aos8_get_ipsec_tunnels` correct command | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_ipsec_tunnels -x` | ✅ |
| DIFF-09 | `aos8_get_md_health_check` aggregates 5 sub-calls; partial-failure path; required `config_path` | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_get_md_health_check tests/unit/test_aos8_read_differentiators.py::test_get_md_health_check_requires_config_path tests/unit/test_aos8_read_differentiators.py::test_get_md_health_check_handles_partial_failure -x` | ✅ |
| Cross-cut | All 9 DIFF tools exposed at module level | unit | `uv run pytest tests/unit/test_aos8_read_differentiators.py::test_diff_tools_count_matches_module -x` | ✅ |
| Regression | Full suite zero regressions | suite | `uv run pytest tests/unit -q` | ✅ (764 tests) |
| Lint | Ruff clean | static | `uv run ruff check . && uv run ruff format --check .` | n/a |
| Types | Mypy clean | static | `uv run mypy src/ --ignore-missing-imports` | n/a |

### Sampling Rate
- **Per task commit:** `uv run pytest tests/unit/test_aos8_read_differentiators.py -x -q` (~13 tests, < 5s)
- **Per wave merge:** `uv run pytest tests/unit -q && uv run ruff check . && uv run mypy src/`
- **Phase gate:** Full 764-test suite GREEN + ruff + mypy clean before `/gsd:verify-work`. CLAUDE.md Pre-Push Checklist (Docker-based) before any push.

### Wave 0 Gaps
- None — all 13 DIFF test cases already exist (Phase 7 Plan 01 RED scaffold), the differentiator module exists, fixtures exist (`tests/unit/fixtures/aos8/`), and the module is wired into `TOOLS['differentiators']` (Phase 7 Plan 03). The phase is purely a corrective edit to existing code + existing tests.

## Sources

### Primary (HIGH confidence)
- `C:/Dev/adams_mcp/hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/client.py` lines 201–271 — `AOS8Client.request()` signature returning `httpx.Response` (verified by direct read)
- `C:/Dev/adams_mcp/hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/_helpers.py` lines 41–86 — canonical `run_show` / `get_object` calling `response.json()` (verified by direct read)
- `C:/Dev/adams_mcp/hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/differentiators.py` lines 42–86 — buggy local `_show` / `_object` helpers (verified by direct read)
- `C:/Dev/adams_mcp/hpe-networking-mcp/tests/unit/test_aos8_read_differentiators.py` lines 25–31, 219–258, 269–307 — buggy mock setup (verified by direct read)
- `C:/Dev/adams_mcp/.planning/REQUIREMENTS.md` lines 60–69, 178–186 — DIFF-01..09 phase assignment to Phase 8
- `C:/Dev/adams_mcp/.planning/STATE.md` lines 81–86 — Phase 7 decisions logging the original (incorrect) rationale
- `C:/Dev/adams_mcp/CLAUDE.md` (project root) — project constraints, ruff/mypy/file-size limits, test policy
- `C:/Dev/adams_mcp/hpe-networking-mcp/CLAUDE.md` — Documentation Checklist, Pre-Push Checklist, branch protection

### Secondary (MEDIUM confidence)
- `unittest.mock` stdlib documentation — `AsyncMock(return_value=...)` semantics: outer call awaitable, inner attributes plain `MagicMock` unless explicitly set. (Standard knowledge, cross-checked against existing AOS8 test patterns in `tests/unit/test_aos8_read_*.py`.)
- `httpx.Response.json()` is synchronous in httpx 0.28+ — verified against `httpx >= 0.28.0` pin in `pyproject.toml` and consistent usage throughout `_helpers.py`, `client.py`.

### Tertiary (LOW confidence)
- None — every claim in this research is anchored in a file in this repo.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — no new deps; all libraries in `uv.lock`
- Architecture: HIGH — fix is a literal pattern-restoration to match 26 existing read tools
- Pitfalls: HIGH — bug is fully understood; all 5 pitfalls anchored in concrete file locations and line numbers
- Test mocking strategy: HIGH — pattern is already in use elsewhere in the same test directory

**Research date:** 2026-04-28
**Valid until:** 2026-05-28 (stable code area; only invalidated by upstream `httpx` API change or `AOS8Client.request` contract change — neither anticipated)
