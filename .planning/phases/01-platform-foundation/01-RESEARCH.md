# Phase 1: Platform Foundation - Research

**Researched:** 2026-04-27
**Domain:** Python config/secrets loading, platform module wiring (no external libraries)
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Secrets & Auto-Disable (FOUND-01, FOUND-02)
- **D-01:** Required secrets for platform enable: `aos8_host`, `aos8_username`, `aos8_password`. All three must be present and non-empty; missing any triggers auto-disable with a `logger.info` message listing the missing secret names.
- **D-02:** Optional secrets with defaults: `aos8_port` (default `4343`) and `aos8_verify_ssl` (default `True`). Missing or empty secret files for these are silently accepted — values fall back to defaults.
- **D-03:** Pattern mirrors `_load_apstra()` exactly: collect missing required secrets in a list, log and return `None` if any missing, then parse optional fields with fallbacks.

#### Write-Gate Flag (FOUND-03)
- **D-04:** `ENABLE_AOS8_WRITE_TOOLS` environment variable, parsed as truthy string (`"true"`, `"1"`, `"yes"`), defaults to `false`. Field name on `ServerConfig`: `enable_aos8_write_tools: bool = False`.

#### ServerConfig Field Placement (FOUND-01, FOUND-03)
- **D-05:** Append `enable_aos8_write_tools` and `aos8: AOS8Secrets | None = None` **after** the existing `axis` entries — do not reorder existing fields. Minimal diff, no churn.

#### Tool Registry Wiring (FOUND-04)
- **D-06:** Add `"aos8": {}` to `REGISTRIES`.
- **D-07:** Add `"aos8": {"aos8_write", "aos8_write_delete"}` to `_WRITE_TAG_BY_PLATFORM`.
- **D-08:** Add `"aos8": "enable_aos8_write_tools"` to `_GATE_CONFIG_ATTR`.

#### Docker Compose & Secret Files (FOUND-05)
- **D-09:** Add all 5 AOS8 secrets to the `secrets:` block in `docker-compose.yml` and mount them in the service's `secrets:` list, following the same comment/grouping style as existing platforms.
- **D-10:** Create `.example` files for all 5 secrets in `secrets/`. Content: descriptive placeholder values matching the style of existing examples.

#### server.py Scope
- **D-11:** Phase 1 does NOT touch `server.py`. No lifespan wiring, no write-gate visibility block, no `register_tools` call. All `server.py` changes are deferred to Phase 2.

### Claude's Discretion
- Exact placeholder text in `.example` files (realistic but not real credentials)
- `_load_aos8()` log message wording (follow existing platform style)
- `AOS8Secrets` dataclass attribute order (host, username, password, port, verify_ssl)

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FOUND-01 | `AOS8Secrets` loads `aos8_host`, `aos8_username`, `aos8_password`, `aos8_port` (default 4343), `aos8_verify_ssl` via `_read_secret()` | `_load_apstra()` pattern in `config.py:278-323` is direct template; `ApstraSecrets` dataclass at `config.py:73-79` is exact structural match |
| FOUND-02 | Platform auto-disables gracefully when any required secret absent or empty | `_load_apstra()` collects `missing: list[str]`, logs comma-joined, returns `None`; `_read_secret()` already returns `None` for missing/empty/whitespace files (`config.py:145-158`) |
| FOUND-03 | `ENABLE_AOS8_WRITE_TOOLS` env var gates write tool visibility (default false) | `enable_apstra_write_tools` precedent in `ServerConfig` (`config.py:105`) and parse pattern in `load_config()` (`config.py:360`); same `_truthy = ("true", "1", "yes")` tuple |
| FOUND-04 | `aos8` registered in `REGISTRIES`, write-tag sets, and gate-config map | All three dicts live in `tool_registry.py:39-81`; pattern: `"apstra": {}`, `"apstra": {"apstra_write", "apstra_write_delete"}`, `"apstra": "enable_apstra_write_tools"` |
| FOUND-05 | `docker-compose.yml` updated with 5 AOS8 secrets and env var; `.example` template files created | Exact pattern visible in `docker-compose.yml:38-43` (apstra service mount) and `:87-97` (apstra top-level secrets); `.example` files in `secrets/apstra_*.example` are 1-line placeholders |
</phase_requirements>

## Project Constraints (from CLAUDE.md)

- Line length: 120 chars; double quotes; trailing commas in multi-line structures
- Python 3.12+; type hints on all signatures
- Google-style docstrings for public functions/classes
- Files ≤ 500 lines, functions ≤ 50 lines, classes ≤ 100 lines
- Ruff lint: `E`, `F`, `W`, `I`, `N`, `UP`, `B`, `SIM` selected
- Loguru for logging; never log secret values without `mask_secret()`
- Tests use `@pytest.mark.unit`; `asyncio_mode = "auto"`
- Pre-push: `ruff check`, `ruff format --check`, `mypy src/`, `pytest tests/`
- Naming: `snake_case` functions, `PascalCase` classes, `UPPER_SNAKE_CASE` constants
- Commit messages must NOT mention Claude or Claude Code
- Docker-first; secrets always at `/run/secrets/`, never env vars or JSON
- Branching: feature/* or fix/* branches, but `.planning/config.json` has `branching_strategy: "none"` — defer to user

## Summary

Phase 1 is pure plumbing — no external libraries, no API client, no tool implementations. Every required artifact has a direct, recently-written precedent in the existing `apstra` platform wiring. The total surface is approximately:

- **5 edits** to `src/hpe_networking_mcp/config.py` (1 dataclass, 1 loader, 3 spots in `ServerConfig`/`load_config()`, 1 spot in `enabled_platforms` property)
- **3 edits** to `src/hpe_networking_mcp/platforms/_common/tool_registry.py` (one entry each in `REGISTRIES`, `_WRITE_TAG_BY_PLATFORM`, `_GATE_CONFIG_ATTR`)
- **2 edits** to `docker-compose.yml` (service `secrets:` mount list + top-level `secrets:` definitions + 1 new env var line)
- **5 new files** in `secrets/`: `aos8_host.example`, `aos8_username.example`, `aos8_password.example`, `aos8_port.example`, `aos8_verify_ssl.example`
- **2 test additions:** unit tests for `_load_aos8()` mirroring `tests/unit/test_apstra_config.py`, plus extending `tests/conftest.py:secrets_dir` fixture to include AOS8 keys

**Primary recommendation:** Copy-adapt `_load_apstra()` and `ApstraSecrets` verbatim, swapping `apstra` → `aos8`, `server` → `host`, default port `443` → `4343`. Mechanical change, low risk.

## Standard Stack

This phase introduces no new dependencies. All required machinery is in-tree.

### Core (already installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `loguru` | >= 0.7.3 | Log "AOS8: disabled (missing secrets: ...)" message | Project's logging primitive (`config.py:36`) |
| `dataclasses` | stdlib | `AOS8Secrets` dataclass | All 6 existing platforms use plain `@dataclass` |
| `pathlib` | stdlib | Used inside existing `_read_secret()` | No change needed |

### Supporting (test-only, already installed)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `pytest` | >= 8.4.0 | Test framework | All Phase 1 tests |
| `pytest`'s `monkeypatch` | n/a | Patch `SECRETS_DIR` for test isolation | Mirrors `patch_secrets_dir` fixture pattern |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `@dataclass` | `pydantic.BaseModel` / `pydantic-settings` | Pydantic is in deps for tools/models, but every existing secrets dataclass uses plain `@dataclass` — staying consistent reduces churn and matches D-03's "mirrors `_load_apstra()` exactly" |
| Custom truthy parser | `distutils.util.strtobool` | `distutils` removed in 3.12; project uses inline `("true", "1", "yes")` tuple; copy that |

**No new package installs required.**

## Architecture Patterns

### File Structure (post-phase)

```
src/hpe_networking_mcp/
├── config.py                                  # EDIT — add AOS8Secrets + _load_aos8()
└── platforms/
    └── _common/
        └── tool_registry.py                   # EDIT — add "aos8" to 3 dicts

docker-compose.yml                             # EDIT — add 5 secrets + env var

secrets/
├── aos8_host.example                          # NEW
├── aos8_username.example                      # NEW
├── aos8_password.example                      # NEW
├── aos8_port.example                          # NEW
└── aos8_verify_ssl.example                    # NEW

tests/
├── conftest.py                                # EDIT — extend secrets_dir fixture
└── unit/
    └── test_aos8_config.py                    # NEW — mirrors test_apstra_config.py
```

### Pattern 1: Secrets dataclass (mirror of `ApstraSecrets`)

**What:** Plain `@dataclass` with required fields first, then defaulted fields.
**When to use:** Every new platform's secret bundle.
**Example (target):**

```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/config.py:73-79 (ApstraSecrets)
@dataclass
class AOS8Secrets:
    """Aruba OS 8 / Mobility Conductor credentials."""

    host: str  # hostname or IP, e.g. conductor.example.com
    username: str
    password: str
    port: int = 4343
    verify_ssl: bool = True
```

### Pattern 2: Loader function (mirror of `_load_apstra`)

**What:** Reads each secret via `_read_secret()`, collects required-but-missing into a list, logs and returns `None` on any miss; otherwise parses optionals with fallbacks and returns the dataclass.
**Example (target):**

```python
# Source: adapted from config.py:278-323 (_load_apstra)
def _load_aos8() -> AOS8Secrets | None:
    """Load AOS8 / Mobility Conductor credentials from Docker secrets."""
    host = _read_secret("aos8_host")
    username = _read_secret("aos8_username")
    password = _read_secret("aos8_password")
    port_str = _read_secret("aos8_port")
    verify_ssl_str = _read_secret("aos8_verify_ssl")

    missing = []
    if not host:
        missing.append("aos8_host")
    if not username:
        missing.append("aos8_username")
    if not password:
        missing.append("aos8_password")

    if missing:
        logger.info("AOS8: disabled (missing secrets: {})", ", ".join(missing))
        return None

    assert host is not None
    assert username is not None
    assert password is not None

    try:
        port = int(port_str) if port_str else 4343
    except ValueError:
        logger.warning("AOS8: invalid aos8_port value '{}', defaulting to 4343", port_str)
        port = 4343

    verify_ssl = verify_ssl_str.lower() not in ("false", "0", "no") if verify_ssl_str else True

    logger.info(
        "AOS8: credentials loaded (host: {}, port: {}, user: {}, verify_ssl: {})",
        host, port, username, verify_ssl,
    )
    return AOS8Secrets(
        host=host, username=username, password=password,
        port=port, verify_ssl=verify_ssl,
    )
```

### Pattern 3: ServerConfig field placement

**What:** Append fields after the last existing platform (`axis`).
**Example (target additions):**

```python
# In ServerConfig (after line 106 / line 125):
enable_axis_write_tools: bool = False
enable_aos8_write_tools: bool = False     # NEW

# ... later, after axis: AxisSecrets | None = None ...
aos8: AOS8Secrets | None = None           # NEW
```

Update `enabled_platforms` property:
```python
if self.axis:
    platforms.append("axis")
if self.aos8:                              # NEW
    platforms.append("aos8")               # NEW
```

Update `load_config()`:
```python
enable_axis_write = os.getenv("ENABLE_AXIS_WRITE_TOOLS", "false").lower() in _truthy
enable_aos8_write = os.getenv("ENABLE_AOS8_WRITE_TOOLS", "false").lower() in _truthy   # NEW

# ...
axis = _load_axis()
aos8 = _load_aos8()                        # NEW

# In ServerConfig(...) call:
enable_aos8_write_tools=enable_aos8_write, # NEW
aos8=aos8,                                 # NEW
```

### Pattern 4: Tool registry three-dict update

**Source:** `tool_registry.py:39-81`

```python
REGISTRIES: dict[str, dict[str, ToolSpec]] = {
    "aos8": {},        # NEW
    "apstra": {},
    "axis": {},
    # ... rest unchanged ...
}

_WRITE_TAG_BY_PLATFORM: dict[str, set[str]] = {
    "aos8": {"aos8_write", "aos8_write_delete"},   # NEW
    "apstra": {"apstra_write", "apstra_write_delete"},
    # ... rest unchanged ...
}

_GATE_CONFIG_ATTR: dict[str, str | None] = {
    "aos8": "enable_aos8_write_tools",             # NEW
    "apstra": "enable_apstra_write_tools",
    # ... rest unchanged ...
}
```

### Pattern 5: docker-compose.yml additions

Append AOS8 to two locations:

```yaml
# Service environment block (after line 17):
- ENABLE_AXIS_WRITE_TOOLS=${ENABLE_AXIS_WRITE_TOOLS:-true}
- ENABLE_AOS8_WRITE_TOOLS=${ENABLE_AOS8_WRITE_TOOLS:-true}    # NEW

# Service secrets block (after axis_api_token, line 45):
  # Aruba OS 8 / Mobility Conductor (remove lines for platforms you don't use)
  - aos8_host
  - aos8_username
  - aos8_password
  - aos8_port
  - aos8_verify_ssl

# Top-level secrets block (after axis_api_token block, line 100):
  # Aruba OS 8 / Mobility Conductor
  aos8_host:
    file: ./secrets/aos8_host
  aos8_username:
    file: ./secrets/aos8_username
  aos8_password:
    file: ./secrets/aos8_password
  aos8_port:
    file: ./secrets/aos8_port
  aos8_verify_ssl:
    file: ./secrets/aos8_verify_ssl
```

### Anti-Patterns to Avoid

- **Don't reorder existing platforms.** D-05 explicitly forbids this. Append-only.
- **Don't use Pydantic for `AOS8Secrets`.** All 6 existing secret bundles are plain `@dataclass`. Diverging breaks pattern symmetry.
- **Don't move `_load_aos8()` to a separate file.** All loaders live in `config.py`. The file is currently ~414 lines; one more loader (~45 lines) keeps it under the 500-line cap (final ~460).
- **Don't add a probe to `health.py`.** D-11 defers this to Phase 2; `_ALL_PLATFORMS` in `health.py:32` will be updated in Phase 2 alongside the AOS8 client.
- **Don't update `tests/conftest.py` `_install_registry_stubs()` for AOS8.** No `_registry.py` exists yet for AOS8 — that's Phase 2. Stubbing now would `ImportError`.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Reading secret files | Custom `open()` + strip + None handling | Existing `_read_secret(name)` in `config.py:145` | Already strips whitespace, handles empty files, handles `OSError`, returns `None` consistently — exactly what every other platform uses |
| Truthy env var parsing | Custom truthy parser, `bool(os.getenv(...))`, `distutils.strtobool` | Inline tuple: `os.getenv("ENABLE_AOS8_WRITE_TOOLS", "false").lower() in _truthy` where `_truthy = ("true", "1", "yes")` | Project standard at `config.py:356-362`; `distutils` was removed in Python 3.12 |
| `verify_ssl` parsing | Custom boolean coercion | `verify_ssl_str.lower() not in ("false", "0", "no") if verify_ssl_str else True` | Exact pattern in `_load_apstra()` and `_load_clearpass()`; covered by parameterized tests in `test_apstra_config.py` |
| Write-tool gating | Per-tool conditional registration | Tag-based gating via `_WRITE_TAG_BY_PLATFORM` + `_GATE_CONFIG_ATTR` | Already implemented; tag the tool, register the platform, done. Tested in `test_tool_registry.py` |
| Test fixtures | New per-platform fixture | Extend the shared `secrets_dir` fixture in `tests/conftest.py:54` | Existing pattern adds keys to the shared dict. AOS8 needs the same; do it once, every test sees it |

**Key insight:** Phase 1 is a copy-adapt exercise, not a design exercise. The Apstra platform is the single most-faithful reference (same auth shape: hostname + username + password + port + verify_ssl). Mechanical reuse is correct.

## Common Pitfalls

### Pitfall 1: Adding "aos8" to `_install_registry_stubs()` in `tests/conftest.py` prematurely
**What goes wrong:** Test collection fails with `ImportError: No module named 'hpe_networking_mcp.platforms.aos8._registry'`.
**Why it happens:** Phase 1 doesn't create the platform package. The stub helper iterates module paths and tries to import them.
**How to avoid:** Leave `platform_registries` tuple in `conftest.py:23-29` alone in Phase 1. Add `"hpe_networking_mcp.platforms.aos8._registry"` in Phase 2 when the package exists.
**Warning signs:** `pytest --collect-only` shows ImportError on conftest.

### Pitfall 2: Test for `load_config()` listing all platforms breaks
**What goes wrong:** `test_returns_config_with_all_platforms_enabled` in `test_config.py:133` asserts `set(config.enabled_platforms) == {"mist", "central", "greenlake", "apstra"}` — the existing test omits clearpass and axis (because the shared `secrets_dir` fixture only has 4 platforms in it).
**Why it happens:** Adding AOS8 secret entries to `secrets_dir` fixture would now enable AOS8, breaking that assertion.
**How to avoid:** Either (a) update the assertion to include `"aos8"` (and `clearpass`/`axis` if those secrets get added in this phase to the fixture too), or (b) only add AOS8 keys conservatively and verify the assertion still matches. Recommended: update the assertion to add `"aos8"` since the fixture must now contain AOS8 keys for `test_aos8_config.py` to work.
**Warning signs:** `pytest tests/unit/test_config.py::TestLoadConfig::test_returns_config_with_all_platforms_enabled` fails after fixture update.

### Pitfall 3: Test `test_returns_config_with_only_mist_enabled` lists files to unlink
**What goes wrong:** That test (`test_config.py:148`) hard-codes the list of files to delete to leave only Mist. Adding 5 AOS8 secret files to the `secrets_dir` fixture without adding them to this delete list will leave AOS8 enabled, failing `assert config.enabled_platforms == ["mist"]`.
**How to avoid:** When extending the fixture, also extend the unlink list in this test by 5 entries (`aos8_host`, `aos8_username`, `aos8_password`, `aos8_port`, `aos8_verify_ssl`).

### Pitfall 4: `_truthy` is locally scoped in `load_config()`
**What goes wrong:** Trying to reference `_truthy` from outside `load_config()` fails — it's a local variable, not module-level.
**How to avoid:** Inline the env var parse line inside `load_config()`, same indentation as the existing six.

### Pitfall 5: `clearpass` and `axis` secret keys missing from shared `secrets_dir` fixture
**What goes wrong:** Existing fixture (`conftest.py:54-72`) only seeds 4 platforms. Tests for clearpass/axis use their own helpers. AOS8 should follow the **shared fixture** convention (per Apstra precedent at line 64-68) so `test_config.py::TestLoadConfig` can run unchanged in spirit.
**How to avoid:** Add 5 AOS8 keys to the shared fixture dict.

### Pitfall 6: Default port confusion
**What goes wrong:** Using `443` (Apstra's default) instead of `4343` (AOS8 default).
**Why it happens:** Mechanical copy from `_load_apstra()` carries the wrong default.
**How to avoid:** Three places to update: dataclass default, `int(port_str) if port_str else 4343`, and the `ValueError` fallback log/value.

### Pitfall 7: Writing log line that includes the password
**What goes wrong:** A logger.info that interpolates `password` would leak it.
**How to avoid:** Match the Apstra log line exactly — only `host`, `port`, `username`, `verify_ssl` are logged. Password is never logged. (Apstra also doesn't `mask_secret(password)` because it's never emitted.)

### Pitfall 8: ServerConfig dataclass field ordering with defaults
**What goes wrong:** Python dataclasses error if a non-default field appears after a default-valued field. `ServerConfig` already only has defaulted fields, so appending `enable_aos8_write_tools: bool = False` and `aos8: AOS8Secrets | None = None` is safe — but verify nothing without a default sneaks in.

## Code Examples

### Example 1: Test mirroring `test_apstra_config.py`
```python
# Source: hpe-networking-mcp/tests/unit/test_apstra_config.py (mechanical adapt)
"""Unit tests for AOS8 config loading."""
from __future__ import annotations
import pytest
from hpe_networking_mcp.config import AOS8Secrets, _load_aos8


@pytest.mark.unit
class TestLoadAOS8:
    def test_returns_secrets_when_all_required_present(self, patch_secrets_dir):
        result = _load_aos8()
        assert isinstance(result, AOS8Secrets)
        assert result.host == "conductor.test.example.com"
        assert result.port == 4343
        assert result.username == "admin"
        assert result.password == "aos8-test-password"
        assert result.verify_ssl is True

    def test_returns_none_when_host_missing(self, patch_secrets_dir):
        (patch_secrets_dir / "aos8_host").unlink()
        assert _load_aos8() is None

    def test_returns_none_when_username_missing(self, patch_secrets_dir):
        (patch_secrets_dir / "aos8_username").unlink()
        assert _load_aos8() is None

    def test_returns_none_when_password_missing(self, patch_secrets_dir):
        (patch_secrets_dir / "aos8_password").unlink()
        assert _load_aos8() is None

    def test_port_defaults_to_4343_when_missing(self, patch_secrets_dir):
        (patch_secrets_dir / "aos8_port").unlink()
        assert _load_aos8().port == 4343

    def test_port_parses_integer(self, patch_secrets_dir):
        (patch_secrets_dir / "aos8_port").write_text("8443")
        assert _load_aos8().port == 8443

    def test_invalid_port_falls_back_to_4343(self, patch_secrets_dir):
        (patch_secrets_dir / "aos8_port").write_text("not-a-port")
        assert _load_aos8().port == 4343

    def test_verify_ssl_defaults_true_when_missing(self, patch_secrets_dir):
        (patch_secrets_dir / "aos8_verify_ssl").unlink()
        assert _load_aos8().verify_ssl is True

    @pytest.mark.parametrize("value", ["false", "FALSE", "0", "no", "No"])
    def test_verify_ssl_false_variants(self, patch_secrets_dir, value):
        (patch_secrets_dir / "aos8_verify_ssl").write_text(value)
        assert _load_aos8().verify_ssl is False

    @pytest.mark.parametrize("value", ["true", "1", "yes", ""])
    def test_verify_ssl_truthy_defaults(self, patch_secrets_dir, value):
        (patch_secrets_dir / "aos8_verify_ssl").write_text(value)
        assert _load_aos8().verify_ssl is True
```

### Example 2: Tool registry write-gate test (extension to `test_tool_registry.py`)

```python
# Adapted from test_tool_registry.py:104-113 (test_central_write_gate)
def test_aos8_write_gate(self):
    spec = ToolSpec(
        name="aos8_manage_ssid_profile",
        func=_fn, platform="aos8", category="w",
        tags={"aos8_write"},
    )
    assert is_tool_enabled(spec, self._make_config()) is False
    assert is_tool_enabled(spec, self._make_config(enable_aos8_write_tools=True)) is True

def test_aos8_write_delete_gate(self):
    spec = ToolSpec(
        name="aos8_delete_ssid",
        func=_fn, platform="aos8", category="w",
        tags={"aos8_write_delete"},
    )
    assert is_tool_enabled(spec, self._make_config()) is False
    assert is_tool_enabled(spec, self._make_config(enable_aos8_write_tools=True)) is True
```

### Example 3: `.example` file contents

Each is a single line, no trailing comments. Following Apstra style:

- `secrets/aos8_host.example` → `your-conductor-hostname-or-ip`
- `secrets/aos8_username.example` → `admin`
- `secrets/aos8_password.example` → `replace-with-real-password`
- `secrets/aos8_port.example` → `4343`
- `secrets/aos8_verify_ssl.example` → `true`

## State of the Art

This phase touches no external technology; the "state of the art" is internal precedent.

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Per-tool registration in `server.py` | Tag-based gating + `_common/tool_registry.py` | v2.0 (Phase 0, #158) | Adding a new platform now means three dict entries, not server.py edits |
| Per-platform `apstra_health` / `clearpass_test_connection` tools | Single cross-platform `health` tool with per-platform probes | v2.0 (#146) | Phase 2 will add `_probe_aos8()` to `health.py` — out of scope here |
| Static-only tool exposure | Default `MCP_TOOL_MODE=dynamic` (3 meta-tools per platform) | v2.0.0.0 | Doesn't affect Phase 1; matters in Phase 3+ |

**Deprecated/outdated:** None — the patterns being copied are current as of v2.3.0.1.

## Open Questions

1. **Should AOS8 secret files (and clearpass/axis) be added to the `secrets_dir` fixture in `tests/conftest.py`?**
   - What we know: Existing fixture only contains mist/central/greenlake/apstra (line 54-72). Clearpass and axis tests presumably manage their own fixtures.
   - What's unclear: Do we follow Apstra precedent (extend shared fixture) or Clearpass precedent (separate fixture)?
   - Recommendation: Follow Apstra — add 5 AOS8 keys to the shared `secrets_dir` fixture so `test_config.py::TestLoadConfig::test_returns_config_with_all_platforms_enabled` has a consistent baseline. Update that test's assertion to include `"aos8"`. Update the unlink list in `test_returns_config_with_only_mist_enabled` to include the 5 AOS8 files.

2. **`docker-compose.yml` env var default — `true` or `false`?**
   - What we know: All five existing platforms default to `${ENABLE_X_WRITE_TOOLS:-true}` in the compose file (`docker-compose.yml:13-17`), even though the underlying Python default is `False`.
   - What's unclear: Why compose defaults to `true` despite docs saying writes are off-by-default.
   - Recommendation: Match the existing pattern (`:-true`) for consistency. The runtime safety still requires user confirmation via elicitation middleware (Phase 5 concern).

3. **Branching strategy for the implementation PR.**
   - What we know: `.planning/config.json` says `branching_strategy: "none"`, but `hpe-networking-mcp/CLAUDE.md` mandates feature branches with branch protection on `main`.
   - What's unclear: Whether the fork has the same branch protection.
   - Recommendation: Defer to user at plan-time. Working branch `feature/aos8-platform-foundation` is a safe default if branch protection exists.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python | All edits + tests | ✓ | 3.12+ (per `pyproject.toml:11`) | — |
| `uv` | Test execution | ✓ (assumed; project standard) | latest | `python -m pytest` directly |
| `pytest` | Tests | ✓ | >= 8.4.0 (in dev deps) | — |
| `ruff` | Lint check | ✓ | >= 0.15.6 (in dev deps) | — |
| `mypy` | Type check | ✓ | >= 1.18.0 (in dev deps) | — |
| Docker | Runtime verification (compose validate) | unknown | — | `docker compose config` (validate-only) — but Phase 1 has no runtime artifacts to verify; YAML parse via `python -c "import yaml; yaml.safe_load(open('docker-compose.yml'))"` is sufficient |
| AOS8 / Mobility Conductor | NONE in Phase 1 | n/a | n/a | n/a — Phase 1 produces no live API calls |

**Missing dependencies with no fallback:** None. Phase 1 is pure source/config edits.

**Missing dependencies with fallback:** None required.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest 8.4.0+ with `pytest-asyncio` 1.2.0+ (`asyncio_mode = "auto"`) |
| Config file | `pyproject.toml:103-109` (`[tool.pytest.ini_options]`) |
| Quick run command | `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_config.py tests/unit/test_tool_registry.py tests/unit/test_config.py -x -q` |
| Full suite command | `cd hpe-networking-mcp && uv run pytest tests/ -q` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FOUND-01 | `AOS8Secrets` loads all 5 secret files via `_read_secret()` | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_secrets_when_all_required_present -x` | ❌ Wave 0 |
| FOUND-01 | Optional `aos8_port` defaults to 4343 | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_port_defaults_to_4343_when_missing -x` | ❌ Wave 0 |
| FOUND-01 | Optional `aos8_verify_ssl` defaults to True | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_verify_ssl_defaults_true_when_missing -x` | ❌ Wave 0 |
| FOUND-02 | Auto-disables when `aos8_host` missing | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_none_when_host_missing -x` | ❌ Wave 0 |
| FOUND-02 | Auto-disables when `aos8_username` missing | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_none_when_username_missing -x` | ❌ Wave 0 |
| FOUND-02 | Auto-disables when `aos8_password` missing | unit | `uv run pytest tests/unit/test_aos8_config.py::TestLoadAOS8::test_returns_none_when_password_missing -x` | ❌ Wave 0 |
| FOUND-02 | Server starts cleanly with all AOS8 secrets absent (regression) | unit | `uv run pytest tests/unit/test_config.py::TestLoadConfig::test_returns_config_with_only_mist_enabled -x` | ✅ (extend unlink list) |
| FOUND-03 | `ENABLE_AOS8_WRITE_TOOLS` env var parsed to `enable_aos8_write_tools` field | unit | extend `tests/unit/test_config.py::TestLoadConfig::test_reads_env_overrides` to include `monkeypatch.setenv("ENABLE_AOS8_WRITE_TOOLS", "true")` | ✅ (extend) |
| FOUND-03 | `aos8_write` tag gated by `enable_aos8_write_tools` | unit | `uv run pytest tests/unit/test_tool_registry.py::TestIsToolEnabled::test_aos8_write_gate -x` | ❌ Wave 0 (extend file) |
| FOUND-03 | `aos8_write_delete` tag gated by `enable_aos8_write_tools` | unit | `uv run pytest tests/unit/test_tool_registry.py::TestIsToolEnabled::test_aos8_write_delete_gate -x` | ❌ Wave 0 (extend file) |
| FOUND-04 | `"aos8"` key present in `REGISTRIES` | unit | `uv run pytest tests/unit/test_tool_registry.py::TestRecordTool::test_records_into_the_right_platform -x` (extend with aos8 case) or new test asserting `"aos8" in REGISTRIES` | ✅ (extend) |
| FOUND-04 | `_WRITE_TAG_BY_PLATFORM["aos8"]` and `_GATE_CONFIG_ATTR["aos8"]` correctly populated | unit | covered by `test_aos8_write_gate` above (any failure means a missing dict entry) | indirect |
| FOUND-05 | `docker-compose.yml` parses as valid YAML and includes 5 AOS8 secrets + env var | unit (or smoke) | `python -c "import yaml; c=yaml.safe_load(open('docker-compose.yml')); assert 'aos8_host' in c['secrets']; assert 'aos8_host' in c['services']['hpe-networking-mcp']['secrets']; assert any('AOS8_WRITE' in e for e in c['services']['hpe-networking-mcp']['environment'])"` | ❌ Wave 0 (or asserted in plan task verification step) |
| FOUND-05 | All 5 `.example` files exist with non-empty content | smoke | `for f in aos8_host aos8_username aos8_password aos8_port aos8_verify_ssl; do test -s secrets/$f.example || exit 1; done` | manual / verification step |

### Sampling Rate
- **Per task commit:** `cd hpe-networking-mcp && uv run pytest tests/unit/test_aos8_config.py tests/unit/test_config.py tests/unit/test_tool_registry.py -x -q` (< 5 sec)
- **Per wave merge:** `cd hpe-networking-mcp && uv run pytest tests/unit -q` (< 60 sec)
- **Phase gate:** `cd hpe-networking-mcp && uv run pytest tests/ -q` plus pre-push checklist (`ruff check . && ruff format --check . && mypy src/ --ignore-missing-imports`) all green before `/gsd:verify-work`.

### Wave 0 Gaps
- [ ] `tests/unit/test_aos8_config.py` — covers FOUND-01, FOUND-02 (10+ test methods, mirroring `test_apstra_config.py`)
- [ ] Extend `tests/unit/test_tool_registry.py` — add `test_aos8_write_gate` and `test_aos8_write_delete_gate` (FOUND-03, FOUND-04)
- [ ] Extend `tests/conftest.py:secrets_dir` fixture — add 5 AOS8 keys
- [ ] Extend `tests/unit/test_config.py::TestLoadConfig::test_returns_config_with_all_platforms_enabled` — assertion to include `"aos8"` (FOUND-01 regression)
- [ ] Extend `tests/unit/test_config.py::TestLoadConfig::test_returns_config_with_only_mist_enabled` — unlink list to include 5 AOS8 files (FOUND-02 regression)
- [ ] Extend `tests/unit/test_config.py::TestLoadConfig::test_reads_env_overrides` — `ENABLE_AOS8_WRITE_TOOLS` env (FOUND-03)
- [ ] Smoke check task — YAML parse of `docker-compose.yml` asserting AOS8 secrets/env present (FOUND-05) — can be a planner verification step rather than a pytest case

*(No framework install needed — pytest infrastructure already in place at `hpe-networking-mcp/tests/`.)*

## Sources

### Primary (HIGH confidence)
- `hpe-networking-mcp/src/hpe_networking_mcp/config.py:73-79` — `ApstraSecrets` dataclass (template for `AOS8Secrets`)
- `hpe-networking-mcp/src/hpe_networking_mcp/config.py:278-323` — `_load_apstra()` (template for `_load_aos8()`)
- `hpe-networking-mcp/src/hpe_networking_mcp/config.py:96-142` — `ServerConfig` (insertion target)
- `hpe-networking-mcp/src/hpe_networking_mcp/config.py:336-413` — `load_config()` (insertion target)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/_common/tool_registry.py:39-81` — Three-dict registry pattern
- `hpe-networking-mcp/docker-compose.yml:1-101` — Service + secrets layout
- `hpe-networking-mcp/secrets/apstra_*.example` — Example file content style
- `hpe-networking-mcp/tests/unit/test_apstra_config.py` — Test template
- `hpe-networking-mcp/tests/unit/test_tool_registry.py` — Write-gate test template
- `hpe-networking-mcp/tests/unit/test_config.py` — Loader test pattern
- `hpe-networking-mcp/tests/conftest.py:47-83` — Shared `secrets_dir` / `patch_secrets_dir` fixtures
- `hpe-networking-mcp/pyproject.toml` — Dependency versions, lint config, pytest config

### Secondary (MEDIUM confidence)
- `hpe-networking-mcp/CLAUDE.md` — Conventions, pre-push checklist, documentation requirements (informational; this phase is too low-level for most of the doc checklist)
- `.planning/REQUIREMENTS.md` — FOUND-01 through FOUND-05 specifications

### Tertiary (LOW confidence)
- None — no WebSearch was needed; every required pattern is verifiable in-tree.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — no new deps; all in-tree code is current
- Architecture: HIGH — direct mechanical adaptation of `apstra` (most recent platform addition pattern)
- Pitfalls: HIGH — discovered by inspecting existing tests; pitfalls 1-3 are concrete failure modes seen in current test code
- Validation: HIGH — pytest infrastructure verified present and working; all required test files identifiable

**Research date:** 2026-04-27
**Valid until:** 2026-05-27 (30 days; codebase is stable, but a new v2.x release that refactors `tool_registry.py` or `config.py` would invalidate the mechanical-copy assumption)
