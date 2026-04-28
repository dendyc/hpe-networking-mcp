# Architecture Patterns — AOS8 Platform Module Integration

**Domain:** Adding a 7th platform module (AOS8 / Aruba Mobility Conductor) to an existing FastMCP plugin-based server
**Researched:** 2026-04-27
**Confidence:** HIGH (all decisions grounded in the existing Apstra module — verified via direct file reads)

## Overview

AOS8 fits the existing `platforms/<name>/` plugin pattern with **zero changes** to the core architecture (middleware, dynamic mode, write-gate, lifespan). The Apstra module is the closest behavioral twin (async httpx + session-token auth + asyncio.Lock for refresh + `aclose()` cleanup) and is the canonical model to copy. The work is mostly mechanical: copy Apstra's structure, swap the auth flow (UIDARUBA cookie vs. AuthToken header), and wire AOS8 into the same six integration touchpoints every other platform uses.

## Recommended Architecture

### Module layout (mirror of `platforms/apstra/`)

```
src/hpe_networking_mcp/platforms/aos8/
├── __init__.py            # TOOLS dict + register_tools(mcp, config) -> int
├── _registry.py           # tool() shim + module-level mcp holder
├── client.py              # AOS8Client (async httpx, UIDARUBA cookie, asyncio.Lock)
├── guidelines.py          # Static formatting snippets (status icons, table formats)
├── models.py              # Pydantic v2 response models (Controller, AP, Client, Alert, etc.)
├── config_path.py         # Helper for /md, /mm, /md/<device>, /md/<ap-group> path resolution
└── tools/
    ├── __init__.py        # Re-exports READ_ONLY/WRITE/WRITE_DELETE ToolAnnotations
    ├── controllers.py     # aos8_get_controllers, aos8_get_controller_stats
    ├── access_points.py   # aos8_get_aps, aos8_get_ap_stats, aos8_get_ap_radios
    ├── clients.py         # aos8_search_clients, aos8_get_client_details, aos8_get_client_assoc
    ├── alerts.py          # aos8_get_active_alerts, aos8_get_event_history, aos8_get_audit_logs
    ├── wlan.py            # aos8_get_ssid_profiles, aos8_get_wlan_configs
    ├── show_command.py    # aos8_run_show_command (passthrough)
    ├── troubleshoot.py    # aos8_ping, aos8_traceroute
    ├── manage_clients.py  # aos8_disconnect_client (write_delete)
    ├── manage_wlan.py     # aos8_create_ssid_profile, aos8_update_wlan_config (write)
    ├── manage_config.py   # aos8_write_memory (write — persists per config_path)
    └── prompts.py         # Guided prompt registrations
```

### File-by-file purpose and source of truth

| File | Purpose | Model after |
|------|---------|-------------|
| `aos8/__init__.py` | Declares `TOOLS: dict[str, list[str]]` (category -> tool names) and `register_tools(mcp, config) -> int` that imports every category, sets `_registry.mcp = mcp`, and calls `build_meta_tools("aos8", mcp)` when `tool_mode == "dynamic"` | `apstra/__init__.py` (lines 49-86) |
| `aos8/_registry.py` | `tool()` decorator shim that records into `REGISTRIES["aos8"]` and merges `dynamic_managed` + `aos8` tags onto every FastMCP registration | `apstra/_registry.py` (verbatim, change `_PLATFORM = "aos8"`) |
| `aos8/client.py` | `AOS8Client` async httpx wrapper. Login POST `/v1/api/login` with `username`+`password` form data → `UIDARUBA` cookie. Cookie reused across all requests via `httpx.AsyncClient` cookie jar. `asyncio.Lock` serializes (re-)login. 401 triggers single re-login retry. Exposes `request()`, `get_json()`, `post_config_object()`, `run_show_command()`, `write_memory()`, `health_check()`, `aclose()`. Plus `get_aos8_client()` lifespan helper and `format_http_error()` | `apstra/client.py` (verbatim auth pattern; swap header/JSON-login for cookie/form-login) |
| `aos8/guidelines.py` | Static markdown snippets returned alongside tool responses for consistent rendering (status labels, device table format, change-mgmt notes) | `apstra/guidelines.py` |
| `aos8/models.py` | Pydantic v2 models for typed responses (Controller, AP, AssociatedClient, Alert, AuditLog, SSIDProfile) | `central/models.py` (Aruba domain proximity) |
| `aos8/config_path.py` | Pure helper: validates and normalizes `config_path` (default `/md`, must start with `/md`, `/mm`, or `/md/<...>`). Avoids string scattering across tools | New — no analog needed in other platforms |
| `aos8/tools/__init__.py` | Re-exports `READ_ONLY`, `WRITE`, `WRITE_DELETE` ToolAnnotations | `apstra/tools/__init__.py` (verbatim) |

### Client design specifics (AOS8-flavored)

The Apstra client is the right shape; only the auth particulars change:

| Apstra | AOS8 |
|--------|------|
| POST `/api/user/login` JSON `{username,password}` | POST `/v1/api/login` form-encoded `username=...&password=...` |
| Returns `{"_links":..., "token":"..."}` in body | Returns `{"_global_result":{"UIDARUBA":"..."}}` AND sets `SESSION` cookie |
| Sent as `AuthToken: <token>` header on every request | Cookie automatically attached by `httpx.AsyncClient`'s cookie jar; show-command API also requires `?UIDARUBA=<token>` query param |
| 401 → re-login | 401 OR session-expired body marker → re-login |
| Methods: GET/POST/PUT/DELETE | Methods: **GET and POST only** — surface this as a constraint check in `request()` |

The cookie-jar approach (rather than manually injecting a header) means the `httpx.AsyncClient` keeps the cookie automatically — but the show-command API also wants the token as a query param, so `client.run_show_command(cmd, config_path)` must do both: rely on the cookie AND append `UIDARUBA` as a query param.

`write_memory()` is its own method because operators must call it deliberately after config writes — one POST to `/v1/configuration/object/write_memory?config_path=<path>` per affected config_path.

## Integration Touch-Points (existing files to modify)

### 1. `src/hpe_networking_mcp/config.py`

Add **above** the `ServerConfig` dataclass:

```python
@dataclass
class AOS8Secrets:
    host: str          # hostname or IP, e.g. mc.example.com
    username: str
    password: str
    port: int = 4343
    verify_ssl: bool = True
```

Add **inside** `ServerConfig`:
- New field: `enable_aos8_write_tools: bool = False`
- New field: `aos8: AOS8Secrets | None = None`
- Add `if self.aos8: platforms.append("aos8")` to `enabled_platforms` property

Add **`_load_aos8()`** function (mirror `_load_apstra` exactly, lines 278-323): reads `aos8_host`, `aos8_username`, `aos8_password`, `aos8_port` (optional, default 4343), `aos8_verify_ssl` (optional, default true).

Add **inside `load_config()`**:
- Env parse: `enable_aos8_write = os.getenv("ENABLE_AOS8_WRITE_TOOLS", "false").lower() in _truthy`
- Call: `aos8 = _load_aos8()`
- Pass to `ServerConfig(...)`: `enable_aos8_write_tools=enable_aos8_write, aos8=aos8`

Update the module docstring's secret file mapping to include `aos8_host`, `aos8_port`, `aos8_username`, `aos8_password`, `aos8_verify_ssl`.

### 2. `src/hpe_networking_mcp/server.py`

**In `lifespan()`** (after Axis block, around line 145):

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

**In the `finally` cleanup block** (around line 186):

```python
aos8 = context.get("aos8_client")
if aos8 is not None:
    await aos8.aclose()
```

**In `create_server()`** platform registration (line 238, after `_register_axis_tools`):

```python
if config.aos8:
    _register_aos8_tools(mcp, config)
```

**In write-tool visibility transforms** (around line 279):

```python
if not config.enable_aos8_write_tools:
    mcp.add_transform(Visibility(False, tags={"aos8_write", "aos8_write_delete"}, components={"tool"}))
```

**Add the helper function** at module bottom (after `_register_axis_tools`):

```python
def _register_aos8_tools(mcp: FastMCP, config: ServerConfig) -> None:
    """Register all AOS8 platform tools."""
    from hpe_networking_mcp.platforms.aos8 import register_tools

    count = register_tools(mcp, config)
    logger.info("AOS8: registered {} tools", count)
```

**Update `_register_code_mode()`'s `execute_description`**: add `aos8_` to the list of callable platform-tool prefixes in the `call_tool` description string (line 348-349).

### 3. `src/hpe_networking_mcp/platforms/_common/tool_registry.py`

- Add `"aos8": {}` to `REGISTRIES` dict (line 39-50)
- Add `"aos8": {"aos8_write", "aos8_write_delete"}` to `_WRITE_TAG_BY_PLATFORM` (line 63-71)
- Add `"aos8": "enable_aos8_write_tools"` to `_GATE_CONFIG_ATTR` (line 73-81)

### 4. `src/hpe_networking_mcp/platforms/health.py`

- Add `"aos8"` to `_ALL_PLATFORMS` tuple (line 32)
- Add `_probe_aos8(ctx)` function modeled on `_probe_apstra` (line 143-152): pulls `aos8_client` from context, calls `client.health_check()`, returns `{"status": "ok"|"error", "message": ...}`
- Add `"aos8": _probe_aos8` to the probe dispatch dict (line 188-192)
- Update tool docstring example list (lines 41-43, 240-243) to mention `aos8`

### 5. `src/hpe_networking_mcp/platforms/site_health_check.py`

If AOS8 has a per-site/per-MD concept that maps to Mist/Central sites, add `"aos8"` to `_SITE_PLATFORMS` (around line 90, 106) and add an `AOS8Summary` Pydantic model + an `_query_aos8()` helper alongside the existing Mist/Central/ClearPass collectors. **Decision deferred to milestone-end:** AOS8 doesn't have a flat "site" abstraction — it has Conductor → Managed Devices → AP-Groups. A clean mapping requires the operator to provide a `config_path` per site rather than a site name. Recommend implementing as a **separate** future cross-platform tool (`aos8_md_health_check`) rather than retrofitting `site_health_check` in this milestone. Flag for next milestone.

### 6. `src/hpe_networking_mcp/INSTRUCTIONS.md`

Add an "Aruba OS 8 / Mobility Conductor" section describing config_path semantics, write-memory requirements, show-command passthrough, and tool naming. Loaded at startup; reaches the model at session init.

## docker-compose.yml Changes

Add to `services.hpe-networking-mcp.environment`:
```yaml
- ENABLE_AOS8_WRITE_TOOLS=${ENABLE_AOS8_WRITE_TOOLS:-true}
```

Add to `services.hpe-networking-mcp.secrets`:
```yaml
# AOS8 / Mobility Conductor (remove lines for platforms you don't use)
- aos8_host
- aos8_port
- aos8_username
- aos8_password
- aos8_verify_ssl
```

Add to top-level `secrets:` block:
```yaml
# AOS8 / Mobility Conductor
aos8_host:
  file: ./secrets/aos8_host
aos8_port:
  file: ./secrets/aos8_port
aos8_username:
  file: ./secrets/aos8_username
aos8_password:
  file: ./secrets/aos8_password
aos8_verify_ssl:
  file: ./secrets/aos8_verify_ssl
```

Create the corresponding `secrets/aos8_*.example` template files (committed; real files git-ignored). Five files total: `aos8_host.example`, `aos8_port.example` (contains `4343`), `aos8_username.example`, `aos8_password.example`, `aos8_verify_ssl.example` (contains `false` for self-signed-cert defaults common in AOS8).

## Build Order (dependency chain)

The order is forced by import dependencies and the test-driven workflow.

1. **Skeleton + config wiring** — Create empty `platforms/aos8/{__init__.py, _registry.py, tools/__init__.py}` stubs (just enough for imports to resolve). Add `AOS8Secrets`, `_load_aos8`, `aos8` field, env var to `config.py`. Add `"aos8": {}` to `REGISTRIES`, `_WRITE_TAG_BY_PLATFORM`, `_GATE_CONFIG_ATTR`. Add `_load_aos8` unit tests. **Validation:** `load_config()` produces `aos8=AOS8Secrets(...)` when secret files present, `None` otherwise.

2. **Client** — Implement `AOS8Client` (login, cookie reuse, lock, 401 retry, `request`, `get_json`, `run_show_command`, `write_memory`, `health_check`, `aclose`). Add `models.py` (Pydantic response shapes). Add `config_path.py` helper. Add `guidelines.py` snippets. Wire client into `server.py:lifespan()` and shutdown cleanup. Add `_probe_aos8` to `health.py`. **Validation:** unit tests with mocked httpx — login, request with cookie, 401-refresh, write_memory, health_check.

3. **Read tools** — Implement `controllers.py`, `access_points.py`, `clients.py`, `alerts.py`, `wlan.py`, `show_command.py` (and any other read-only modules). Each tool decorated with `@tool(annotations=READ_ONLY)`. Populate `TOOLS` dict in `__init__.py` as modules are added. Wire `_register_aos8_tools` into `server.py:create_server()`. **Validation:** unit tests with mocked client per tool; verify `REGISTRIES["aos8"]` populates; verify dynamic-mode meta-tools (`aos8_list_tools`, `aos8_get_tool_schema`, `aos8_invoke_tool`) work end-to-end.

4. **Write tools** — Implement `troubleshoot.py` (ping/traceroute), `manage_clients.py` (disconnect), `manage_wlan.py` (create/update SSID), `manage_config.py` (write_memory). Each decorated with `@tool(annotations=WRITE, tags={"aos8_write"})` or `WRITE_DELETE`/`{"aos8_write_delete"}`. Each calls `confirm_write(ctx, ...)` from `middleware.elicitation`. Add Visibility transform to `server.py`. **Validation:** unit tests for elicitation gating, write-flag visibility, and the elicitation-decline path.

5. **Guided prompts** — Implement `tools/prompts.py` registering FastMCP prompts for common AOS8 workflows (e.g., "client troubleshoot," "AP-group config audit," "post-change write-memory checklist"). Parity goal with Central's 12 prompts; ship a smaller starter set (3-5) and grow. **Validation:** prompt registration tests.

6. **INSTRUCTIONS.md update** — Add AOS8 section with domain knowledge.

7. **Cross-platform integration** — Decide whether AOS8 joins `site_health_check` this milestone or defers to next. Recommend defer (see touchpoint #5 above).

8. **Integration tests + docs** — End-to-end test against a mocked-server fixture. Update `docs/TOOLS.md`, `README.md` tool counts and architecture diagram, `CHANGELOG.md`, version bump in `pyproject.toml`.

## Cross-Platform Integration: site_health_check

**Recommendation: defer to next milestone.** Justification:

- `site_health_check` operates on a **site name** abstraction shared by Mist/Central/ClearPass. AOS8 has no flat "site" — it has a Conductor hierarchy (`/md`, `/md/<MD-name>`, `/md/<ap-group>`).
- A clean mapping requires either (a) operator-provided `config_path` per site (breaks the existing single-string-name signature), or (b) a heuristic that infers config_path from site name (fragile).
- A separate cross-platform tool — `aos8_md_health_check(config_path: str)` — better matches AOS8's model and avoids polluting `site_health_check` with a special-case parameter.
- The existing `health` tool DOES probe AOS8 alongside other platforms (touchpoint #4), so basic platform-reachability cross-platform parity is achieved without `site_health_check` changes.

If the milestone explicitly requires `site_health_check` AOS8 support, the minimal addition is: add `aos8` to `_SITE_PLATFORMS`, add `AOS8Summary` model with `config_path` field, add `_query_aos8(ctx, config_path)` helper that calls `aos8_get_aps` + `aos8_get_active_alerts` and aggregates. Document the asymmetry (AOS8 needs `config_path`, others use `site_name`) in the tool docstring.

## Patterns to Follow

### Pattern: Tool decorator (every tool file)

```python
from hpe_networking_mcp.platforms.aos8._registry import tool
from hpe_networking_mcp.platforms.aos8.client import format_http_error, get_aos8_client
from hpe_networking_mcp.platforms.aos8.tools import READ_ONLY

@tool(annotations=READ_ONLY)
async def aos8_get_aps(ctx: Context, config_path: str = "/md") -> dict[str, Any] | str:
    """Get all access points reporting to the Conductor.

    Args:
        config_path: Conductor path scope, default '/md' for all managed devices.
    """
    try:
        client = await get_aos8_client()
        payload = await client.get_json("/v1/configuration/showcommand",
                                        params={"command": "show ap database", "config_path": config_path})
        return {"data": payload}
    except Exception as e:
        return f"Error fetching APs: {format_http_error(e) if hasattr(e, 'response') else e}"
```

### Pattern: Write tool with elicitation

Follow `apstra/tools/manage_blueprints.py:apstra_deploy` exactly — `confirmed: bool = False` parameter, `confirm_write()` call, early-return on decline, then perform the action.

### Pattern: Lifespan client retrieval

Tools never construct clients themselves. They call `get_aos8_client()` (defined in `client.py`) which pulls from `ctx.lifespan_context["aos8_client"]` and raises `ToolError` with status 503 if not configured.

## Anti-Patterns to Avoid

| Anti-Pattern | Why bad | Instead |
|--------------|---------|---------|
| Per-call login (re-authenticating each request) | AOS8 docs warn against — exhausts session limit on Conductor; PROJECT.md explicit decision | Cache cookie in `httpx.AsyncClient` cookie jar; serialize refresh with `asyncio.Lock` |
| Auto-calling `write_memory` after every write | Hides operator intent; multiple write_memory calls are wasteful | Surface as standalone `aos8_write_memory(config_path)` tool; document in INSTRUCTIONS.md |
| Hardcoding `config_path = "/md"` everywhere | Breaks targeted MD/AP-group operations | All write tools accept `config_path: str = "/md"` parameter |
| Importing tool modules at the top of `__init__.py` | Tools register on import — but `_registry.mcp` must be set first | Use `importlib.import_module(...)` inside `register_tools()` after `_registry.mcp = mcp`, exactly like Apstra |
| Using PUT/PATCH/DELETE in the client | AOS8 API doesn't support them | Client `request()` should reject non-GET/POST methods at call time |
| Silent SSL bypass | Hides security issue from operators | Default `verify_ssl=True`, log a WARNING when `False` is loaded from secret |

## Scalability Considerations

| Concern | At small site | At large Conductor (1000+ APs / 50K clients) |
|---------|---------------|---------------------------------------------|
| Show-command passthrough | Direct call | Add per-call timeout override; AOS8 show commands can be slow |
| AP/client listing | Single GET | Verify response is paginated by AOS8; if so, add page handling in client |
| Concurrent calls | Cookie jar handles | `asyncio.Lock` already serializes login; concurrent reads share cookie naturally |
| write_memory frequency | One per change | If multiple writes target the same `config_path`, surface batching guidance in INSTRUCTIONS.md |

## Sources

- `src/hpe_networking_mcp/platforms/apstra/` — primary structural reference (verified by direct read)
- `src/hpe_networking_mcp/platforms/_common/tool_registry.py` — REGISTRIES, write-tag, gate-attr maps (verified)
- `src/hpe_networking_mcp/server.py` — lifespan, create_server, write-visibility, code-mode (verified)
- `src/hpe_networking_mcp/config.py` — `_read_secret`, `_load_apstra` pattern (verified)
- `src/hpe_networking_mcp/platforms/health.py` — `_probe_apstra` pattern, `_ALL_PLATFORMS` (verified)
- `src/hpe_networking_mcp/platforms/site_health_check.py` — `_SITE_PLATFORMS` allow-list (verified)
- `docker-compose.yml` — secret declaration pattern (verified)
- `.planning/PROJECT.md` — AOS8 API characteristics, key decisions (verified)
