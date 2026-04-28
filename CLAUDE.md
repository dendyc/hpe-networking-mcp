<!-- GSD:project-start source:PROJECT.md -->
## Project

**HPE Networking MCP — AOS8 Extension**

A fork of the `hpe-networking-mcp` community project that adds **Aruba OS 8 (AOS8) / Mobility Conductor** as a seventh platform module alongside the existing six (Juniper Mist, Aruba Central, HPE GreenLake, Aruba ClearPass, Juniper Apstra, Axis Atmos Cloud). The goal is full feature parity with the existing Aruba Central module — health overview, device management, client troubleshooting, alerts/events, and WLAN/config management — surfaced through the same MCP tool patterns (dynamic mode, write-gate, Docker secrets) as every other platform.

**Core Value:** An AI assistant that can monitor, troubleshoot, and configure an AOS8/Mobility Conductor network with the same depth and safety as it does Aruba Central — without the operator needing to touch the CLI.

### Constraints

- **API:** GET and POST only — no PUT/PATCH/DELETE; all mutations via POST
- **Auth:** UIDARUBA token must be reused across calls (not re-login per call) to avoid session exhaustion
- **Testing:** No live AOS8 system — all tests use mocked HTTP responses
- **Compatibility:** Must not break any existing platform module or test
- **Style:** Ruff lint + format, mypy type-checked, max 500 lines/file, Google-style docstrings
- **Write Memory:** Config changes require a follow-up Write Memory call per config_path — must be handled or documented clearly
- **SSL:** Default `verify_ssl=true`; support `false` for self-signed certs (common in AOS8 deployments)
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Overview
## Languages
- Python 3.12+ — all server code under `src/hpe_networking_mcp/`
- None (YAML for Docker/CI config; Markdown for skills runbooks in `src/hpe_networking_mcp/skills/`)
## Runtime
- CPython 3.12 (base image: `python:3.12-slim-bookworm`)
- Minimum required: Python >= 3.12 (set in `pyproject.toml`)
- `uv` (Astral) — copied from `ghcr.io/astral-sh/uv:latest` in Dockerfile
- Lockfile: present (`uv.lock` implied by `--frozen` flags in Dockerfile and CI)
- No `requirements.txt` — all deps declared in `pyproject.toml`
## Frameworks
- `fastmcp[code-mode] >= 3.1.1` — MCP server framework; provides `@mcp.tool()` decorator, lifespan context, middleware support, `CodeMode` transform, and `MCP_TOOL_MODE=code` experimental mode
- `mcp[cli] >= 1.20.0` — underlying MCP protocol library (FastMCP depends on this; version floor set by GreenLake requirements)
- `hatchling` — PEP 517 build backend; wheel packages `src/hpe_networking_mcp`; configured in `pyproject.toml` under `[build-system]`
- `pytest >= 8.4.0` — test runner; config in `[tool.pytest.ini_options]` in `pyproject.toml`
- `pytest-cov >= 7.0.0` — coverage
- `pytest-asyncio >= 1.2.0` — async test support; `asyncio_mode = "auto"`
- `responses >= 0.25.7` — HTTP request mocking
- Tests live in `tests/`; unit tests at `tests/unit/`; 639+ unit tests as of README
## Key Dependencies
- `mistapi >= 0.60.4` — Juniper Mist REST API client; used in `src/hpe_networking_mcp/platforms/mist/client.py`
- `pycentral == 2.0a17` — HPE Aruba Central API client; used in `src/hpe_networking_mcp/platforms/central/client.py`
- `pyclearpass >= 1.0.0` — Aruba ClearPass API client; used in `src/hpe_networking_mcp/platforms/clearpass/client.py`
- `httpx >= 0.28.0` — async HTTP client used directly for GreenLake, Apstra, and Axis platforms (no vendor SDK for these)
- `pydantic >= 2.12.0` — data validation; v2 API used throughout; models in `src/hpe_networking_mcp/platforms/central/models.py`
- `pydantic-settings >= 2.11.0` — settings management
- `python-dotenv >= 1.2.0` — `.env` file loading for local dev (`SECRETS_DIR=./secrets uv run python -m hpe_networking_mcp`)
- `loguru >= 0.7.3` — structured logging; all output to stderr (stdout reserved for MCP JSON-RPC); configured in `src/hpe_networking_mcp/utils/logging.py`
- `treelib >= 1.7.0` — tree data structure for Aruba Central scope hierarchy
## Code Quality Tools
- `ruff >= 0.15.6` — linter and formatter (replaces Black + flake8); line length 120; target Python 3.12
- `mypy >= 1.18.0` — static type checking; Python 3.12 target; untyped defs allowed; vendor SDK stubs ignored
- `bandit >= 1.8.0` — static security analysis; excludes `tests/`; skips B101 (assert), B104 (0.0.0.0 bind), B110 (try/except/pass)
- `pip-audit >= 2.7.0` — dependency vulnerability scanning (runs in security CI workflow)
- `pre-commit >= 4.0.0` — git hook management
## Build System
## Configuration
| Variable | Default | Purpose |
|---|---|---|
| `MCP_PORT` | `8000` | HTTP listen port |
| `MCP_HOST` | `0.0.0.0` | HTTP bind address |
| `LOG_LEVEL` | `info` | Loguru log level |
| `SECRETS_DIR` | `/run/secrets` | Docker secrets directory |
| `MCP_TOOL_MODE` | `dynamic` | `dynamic`, `static`, or `code` |
| `ENABLE_MIST_WRITE_TOOLS` | `false` | Enable Mist write/mutation tools |
| `ENABLE_CENTRAL_WRITE_TOOLS` | `false` | Enable Central write tools |
| `ENABLE_CLEARPASS_WRITE_TOOLS` | `false` | Enable ClearPass write tools |
| `ENABLE_APSTRA_WRITE_TOOLS` | `false` | Enable Apstra write tools |
| `ENABLE_AXIS_WRITE_TOOLS` | `false` | Enable Axis write tools |
| `DISABLE_ELICITATION` | `false` | Skip user confirmation for writes |
| `RETRY_MAX_ATTEMPTS` | `3` | Max retry attempts on 5xx/429 |
| `RETRY_INITIAL_DELAY` | `1.0` | Initial backoff seconds |
| `RETRY_MAX_DELAY` | `60.0` | Max retry sleep cap |
## Platform Requirements
- Docker + Docker Compose v2.24+ (for `!reset` override syntax)
- uv (for local dev without Docker: `SECRETS_DIR=./secrets uv run python -m hpe_networking_mcp`)
- Node.js / npx (only needed if connecting Claude Desktop via `supergateway` bridge)
- Docker image: `ghcr.io/nowireless4u/hpe-networking-mcp:latest` (pre-built; supports amd64 and arm64)
- Docker Compose for secrets management
- Secrets directory on host with credential files
## CI/CD
- `ci.yml` — lint (`ruff check`, `ruff format --check`), type check (`mypy`), unit tests (`pytest tests/unit`), Docker build; triggers on push/PR to `main`
- `docker.yml` — Docker image publish to GHCR
- `security.yml` — security scanning (likely `pip-audit` + `bandit`)
## Notes
- `pycentral 2.0a17` is an alpha release with a non-standard API — `NewCentralBase` is at top-level `pycentral`, not `pycentral.base`, and uses `api_params=` not `params=` (noted as a known issue in CLAUDE.md)
- `mistapi` stubs are excluded from mypy (`ignore_missing_imports = true`); same for `pycentral` and `pyclearpass`
- Mist tool files use class-name convention that violates PEP 8 N801 (inherited from vendor API conventions); ruff suppresses this per `[tool.ruff.lint.per-file-ignores]`
- The `fastmcp[code-mode]` extra enables `CodeMode` transform — an experimental sandboxed Python `execute` for multi-step workflows (`MCP_TOOL_MODE=code`)
- Skills feature (v2.3.0.0+): markdown-defined multi-step procedures in `src/hpe_networking_mcp/skills/`, discoverable via `skills_list` / `skills_load` tools
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## Overview
## Formatting
- Line length: 120 characters
- Target version: Python 3.12
- Double quotes for strings
- Trailing commas in multi-line structures
## Linting
- `E`, `W` — pycodestyle errors/warnings
- `F` — pyflakes (unused imports, undefined names)
- `I` — isort (import ordering)
- `N` — pep8-naming
- `UP` — pyupgrade (modern Python idioms)
- `B` — flake8-bugbear
- `SIM` — flake8-simplify
- `src/hpe_networking_mcp/platforms/mist/tools/schemas_data.py`: `E501`, `E711` — auto-generated data, exempt from line length and `is` comparisons
- `src/hpe_networking_mcp/platforms/mist/tools/*.py`: `N801` — ported Mist tool files use original API-convention class names
- `src/hpe_networking_mcp/platforms/central/models.py`: `N815` — ported upstream model names preserved
## Security Checks
- `B101` — `assert` in code (acceptable in non-critical paths)
- `B104` — `0.0.0.0` bind (required for Docker)
- `B110` — `try/except/pass` in shutdown handlers
## Type Checking
## Naming Patterns
| Construct | Pattern | Example |
|-----------|---------|---------|
| Functions / methods | `snake_case` | `get_apisession`, `mask_secret` |
| Variables | `snake_case` | `api_params`, `token_info` |
| Classes | `PascalCase` | `TokenManager`, `ServerConfig` |
| Constants | `UPPER_SNAKE_CASE` | `SECRETS_DIR`, `READ_ONLY` |
| Private attrs / methods | `_leading_underscore` | `_read_secret`, `_token_info` |
| Type aliases | `PascalCase` | `FilterField` |
| MCP tool functions | `snake_case` function name, string `name=` arg for registration | `async def get_site_health(...)` with `@tool(name="mist_get_site_health")` |
## Docstring Standards
## Import Organization
## Type Hint Patterns
## Pydantic Usage
## Logging
## Error Handling
- Tool functions use `try/except Exception as _exc` with a `handle_network_error(_exc)` helper (Mist platform pattern).
- `try/except/pass` is permitted in shutdown/cleanup paths (Bandit B110 is skipped).
- Middleware classes (`ValidationCatchMiddleware`, `SandboxErrorCatchMiddleware`, `RetryMiddleware`) catch specific exception types and return structured `ToolResult` objects rather than letting errors propagate to the LLM as unreadable errors.
## File and Function Size Limits
- Files: max 500 lines; if approaching the limit, split into modules
- Functions: max 50 lines, single responsibility
- Classes: max 100 lines, single concept
## Module Design
## Pre-commit Hooks
## Notes
- `axis` platform tools reference `AdminApiSwagger.json` from `api-endpoints/axis/`. The Mist platform's `schemas_data.py` is auto-generated from API specs and is entirely exempt from style rules.
- The `N801` exemption for Mist tool files preserves original Mist API class-naming conventions from ported upstream code; new code in those files should still follow `PascalCase`.
- `warn_return_any = false` in mypy means untyped return values from third-party SDKs do not generate warnings.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## Overview
## Pattern Overview
- Each platform (mist, central, greenlake, clearpass, apstra, axis) is a self-contained module in `src/hpe_networking_mcp/platforms/`
- A shared `_common/` layer provides the dynamic-mode meta-tool factory and per-platform tool registry
- Tool exposure has two modes: `dynamic` (3 meta-tools per platform, underlying tools hidden) and `static` (every tool individually visible); `code` mode is an experimental third option using FastMCP's sandboxed Python executor
- Write tools are gated behind per-platform environment flags and protected by elicitation middleware requiring user confirmation
## Layers
- Purpose: Parse CLI args, set env vars, call `load_config()` and `create_server()`, run server
- Location: `src/hpe_networking_mcp/__main__.py`
- Depends on: `config.py`, `server.py`, `utils/logging.py`
- Purpose: Load secrets from Docker Compose files, validate per-platform credentials, produce `ServerConfig`
- Location: `src/hpe_networking_mcp/config.py`
- Contains: `ServerConfig`, `MistSecrets`, `CentralSecrets`, `GreenLakeSecrets`, `ClearPassSecrets`, `ApstraSecrets`, `AxisSecrets` dataclasses; `load_config()` function
- Pattern: `_read_secret(name)` reads from `SECRETS_DIR/<name>` (default `/run/secrets`); returns `None` for missing/empty files, which disables that platform
- Purpose: Create FastMCP instance, wire middleware, register all enabled platform tools, manage lifespan
- Location: `src/hpe_networking_mcp/server.py`
- Key functions: `create_server(config)`, `lifespan(server)` (async context manager)
- Lifespan context dict populated at startup, yielded to all tools via `ctx.lifespan_context`
- `NullStripMiddleware` — drops null args before Pydantic validation (`middleware/null_strip.py`)
- `ValidationCatchMiddleware` — converts Pydantic `ValidationError` to human-readable string (`middleware/validation_catch.py`)
- `SandboxErrorCatchMiddleware` — catches code-mode `MontyRuntimeError` → string (`middleware/sandbox_error_catch.py`)
- `ElicitationMiddleware` — write-tool confirmation gate at session init (`middleware/elicitation.py`)
- `RetryMiddleware` — retries transient 5xx (reads only) and 429 (reads+writes) with exponential backoff (`middleware/retry.py`)
- Purpose: Encapsulate all tools, auth client, and registration logic for one platform
- Location: `src/hpe_networking_mcp/platforms/{platform}/`
- Each contains: `__init__.py` (TOOLS dict + `register_tools()`), `_registry.py` (module-level FastMCP holder + `tool()` shim), `client.py` (API client), `tools/` (individual tool files)
- Used by: `server.py` via `register_tools(mcp, config)`
- Purpose: Shared tool registry and dynamic-mode meta-tool factory
- Location: `src/hpe_networking_mcp/platforms/_common/`
- `tool_registry.py`: `REGISTRIES` dict (one per platform), `ToolSpec` dataclass, `record_tool()`, `is_tool_enabled()`
- `meta_tools.py`: `build_meta_tools(platform, mcp)` — registers `<platform>_list_tools`, `<platform>_get_tool_schema`, `<platform>_invoke_tool` on the FastMCP instance
- Location: `src/hpe_networking_mcp/platforms/`
- `health.py` — `health` tool: probes all platforms, same logic used at startup and runtime
- `site_health_check.py` — aggregates Mist + Central + ClearPass site data in one call
- `site_rf_check.py` — per-AP radio state across Mist and Central
- `manage_wlan.py` — WLAN management entry point spanning Mist and Central
- `sync_prompts.py` — WLAN sync workflow prompts (Mist ↔ Central)
- Purpose: Markdown-defined multi-step runbooks discoverable by the AI
- Location: `src/hpe_networking_mcp/skills/`
- `_engine.py`: Loads `*.md` files with YAML frontmatter at startup, exposes `skills_list()` and `skills_load()` tools
- Skill files: `infrastructure-health-check.md`, `change-pre-check.md`, `change-post-check.md`, `wlan-sync-validation.md`
- Location: `src/hpe_networking_mcp/utils/`
- `logging.py`: loguru setup, `mask_secret()`, stderr-only output (stdout reserved for MCP JSON-RPC)
## Tool Registration Flow
```
```
## Lifespan Context
- `ctx.lifespan_context["mist_session"]` — `mistapi.APISession`
- `ctx.lifespan_context["mist_org_id"]` — resolved org ID string
- `ctx.lifespan_context["central_conn"]` — `pycentral.NewCentralBase`
- `ctx.lifespan_context["greenlake_token_manager"]` — `TokenManager` (async httpx, OAuth2)
- `ctx.lifespan_context["clearpass_token_manager"]` — `ClearPassTokenManager`
- `ctx.lifespan_context["clearpass_config"]` — `ClearPassSecrets`
- `ctx.lifespan_context["apstra_client"]` — `ApstraClient` (async httpx, session token)
- `ctx.lifespan_context["apstra_config"]` — `ApstraSecrets`
- `ctx.lifespan_context["axis_client"]` — `AxisClient` (async httpx, JWT bearer)
- `ctx.lifespan_context["axis_config"]` — `AxisSecrets`
- `ctx.lifespan_context["config"]` — `ServerConfig`
## Tool Mode Data Flow
## Write Tool Safety
## Error Handling
- Startup: platform init failures logged as warnings; `context[key] = None`; platform stays in degraded state
- Tool execution: platform clients raise exceptions; tools catch and return structured error dicts
- Middleware: `ValidationCatchMiddleware` intercepts Pydantic errors before they surface as raw exceptions
- Retry: `RetryMiddleware` handles 5xx (reads) and 429 (all) transparently; exponential backoff with `Retry-After` header support
## Platform Client Authentication
- **Mist**: `mistapi.APISession` with static API token + host
- **Central**: `pycentral.NewCentralBase` with OAuth2 client credentials; token managed by pycentral
- **GreenLake**: Custom `TokenManager` in `platforms/greenlake/auth.py`; async httpx; 5-minute expiry buffer
- **ClearPass**: Custom `ClearPassTokenManager` in `platforms/clearpass/client.py`; OAuth2 client_credentials grant; `pyclearpass` SDK wraps the REST layer
- **Apstra**: Custom `ApstraClient` in `platforms/apstra/client.py`; async httpx; POST `/api/user/login` for session token; asyncio.Lock serializes token refresh; 401 triggers re-login
- **Axis**: Custom `AxisClient` in `platforms/axis/client.py`; async httpx; static JWT bearer token; JWT `exp` decoded at startup for expiry warnings
## Cross-Cutting Concerns
## Notes
- The `_registry.py` module-level `mcp = None` holder is set by `register_tools()` before tool modules are imported; tools imported before `register_tools()` is called (e.g., during test collection) get the raw undecorated function back
- `_LifespanProbeCtx` shim in `server.py` allows the same `_probe_*()` functions in `health.py` to run at startup (against the in-construction context dict) and at runtime (against a real `fastmcp.Context`)
- GreenLake is read-only (no write tools); write tag set is empty in `tool_registry.py`
- Axis writes use a staged workflow: changes are staged, then `axis_commit_changes` applies them — unlike other platforms where writes are immediate
- `on_duplicate="replace"` in FastMCP initialization means later registrations silently replace earlier ones; used intentionally for platform reloading scenarios
- `mask_error_details=True` in FastMCP setup hides internal error details from AI clients in production
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
