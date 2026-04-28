# Phase 1: Platform Foundation - Context

**Gathered:** 2026-04-27
**Status:** Ready for planning

<domain>
## Phase Boundary

Wire the AOS8 platform into the server's bootstrap layer. Deliverables:
- `AOS8Secrets` dataclass and `_load_aos8()` in `config.py`
- `enable_aos8_write_tools` flag in `ServerConfig`
- `aos8` entry in `REGISTRIES`, `_WRITE_TAG_BY_PLATFORM`, and `_GATE_CONFIG_ATTR` in `tool_registry.py`
- Five AOS8 secrets + `ENABLE_AOS8_WRITE_TOOLS` env var in `docker-compose.yml`
- Five `.example` secret template files in `secrets/`

**Not in scope for Phase 1:** `server.py` changes, `AOS8Client`, any tool implementations, lifespan context wiring.

</domain>

<decisions>
## Implementation Decisions

### Secrets & Auto-Disable (FOUND-01, FOUND-02)
- **D-01:** Required secrets for platform enable: `aos8_host`, `aos8_username`, `aos8_password`. All three must be present and non-empty; missing any triggers auto-disable with a `logger.info` message listing the missing secret names.
- **D-02:** Optional secrets with defaults: `aos8_port` (default `4343`) and `aos8_verify_ssl` (default `True`). Missing or empty secret files for these are silently accepted — values fall back to defaults.
- **D-03:** Pattern mirrors `_load_apstra()` exactly: collect missing required secrets in a list, log and return `None` if any missing, then parse optional fields with fallbacks.

### Write-Gate Flag (FOUND-03)
- **D-04:** `ENABLE_AOS8_WRITE_TOOLS` environment variable, parsed as truthy string (`"true"`, `"1"`, `"yes"`), defaults to `false`. Field name on `ServerConfig`: `enable_aos8_write_tools: bool = False`.

### ServerConfig Field Placement (FOUND-01, FOUND-03)
- **D-05:** Append `enable_aos8_write_tools` and `aos8: AOS8Secrets | None = None` **after** the existing `axis` entries — do not reorder existing fields. Minimal diff, no churn.

### Tool Registry Wiring (FOUND-04)
- **D-06:** Add `"aos8": {}` to `REGISTRIES`.
- **D-07:** Add `"aos8": {"aos8_write", "aos8_write_delete"}` to `_WRITE_TAG_BY_PLATFORM`.
- **D-08:** Add `"aos8": "enable_aos8_write_tools"` to `_GATE_CONFIG_ATTR`.

### Docker Compose & Secret Files (FOUND-05)
- **D-09:** Add all 5 AOS8 secrets to the `secrets:` block in `docker-compose.yml` and mount them in the service's `secrets:` list, following the same comment/grouping style as existing platforms.
- **D-10:** Create `.example` files for all 5 secrets in `secrets/`. Content: descriptive placeholder values matching the style of existing examples (e.g., `your-conductor-hostname-or-ip`, `admin`, `your-password`, `4343`, `true`).

### server.py Scope
- **D-11:** Phase 1 does NOT touch `server.py`. No lifespan wiring, no write-gate visibility block, no `register_tools` call. All `server.py` changes are deferred to Phase 2 (API Client) when real tools exist to register.

### Claude's Discretion
- Exact placeholder text in `.example` files (realistic but not real credentials)
- `_load_aos8()` log message wording (follow existing platform style)
- `AOS8Secrets` dataclass attribute order (host, username, password, port, verify_ssl)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Config & Secrets Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/config.py` — `AOS8Secrets` dataclass and `_load_aos8()` must mirror `ApstraSecrets` / `_load_apstra()` exactly; `ServerConfig` fields appended after `axis`

### Tool Registry Pattern
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/_common/tool_registry.py` — `REGISTRIES`, `_WRITE_TAG_BY_PLATFORM`, `_GATE_CONFIG_ATTR` dicts to extend

### Docker Compose Pattern
- `hpe-networking-mcp/docker-compose.yml` — secrets section and service secrets list to extend

### Secret File Examples
- `hpe-networking-mcp/secrets/apstra_server.example` — reference for example file style/content
- `hpe-networking-mcp/secrets/apstra_verify_ssl.example` — reference for boolean secret example

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `ApstraSecrets` dataclass in `config.py`: direct model for `AOS8Secrets` — same pattern (hostname, username, password, port with default, verify_ssl with default)
- `_load_apstra()` in `config.py`: copy-adapt for `_load_aos8()` — identical required/optional secret handling pattern
- `REGISTRIES` dict in `tool_registry.py`: add `"aos8": {}` entry alongside existing platforms
- `_WRITE_TAG_BY_PLATFORM` dict: add `"aos8"` entry with `{"aos8_write", "aos8_write_delete"}` tags

### Established Patterns
- Required secrets → collected in `missing: list[str]`, logged as comma-separated, return `None`
- Optional secrets → parsed with fallback: `int(port_str) if port_str else 4343`
- `verify_ssl` string → `verify_ssl_str.lower() not in ("false", "0", "no") if verify_ssl_str else True`
- Secret file naming: `{platform}_{credential_name}` (e.g., `aos8_host`, `aos8_port`)
- Write-gate flag: `enable_{platform}_write_tools: bool = False` in `ServerConfig`, parsed from `ENABLE_{PLATFORM}_WRITE_TOOLS` env var in `load_config()`

### Integration Points
- `config.py:load_config()` — add `ENABLE_AOS8_WRITE_TOOLS` env var parse + `_load_aos8()` call + populate `ServerConfig.aos8`
- `config.py:ServerConfig.enabled_platforms` property — add `if self.aos8: platforms.append("aos8")`
- `docker-compose.yml` — two places: service `secrets:` list and top-level `secrets:` block definitions

</code_context>

<specifics>
## Specific Ideas

No specific requirements beyond the pattern-matching described above. Implementation is a direct adaptation of the Apstra platform wiring.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 01-platform-foundation*
*Context gathered: 2026-04-27*
