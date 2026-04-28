# Phase 6: Guided Prompts & Documentation - Research

**Researched:** 2026-04-28
**Domain:** FastMCP `@mcp.prompt` registration + project documentation update
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** All prompts follow the **fully explicit** style — numbered steps with specific `aos8_*` tool names and key parameter examples. Matches Central's `prompts.py` depth.
- **D-02:** Every prompt ends with a **summary + next steps** section.
- **D-03:** Contextual prompts accept **typed parameters** where it improves workflow specificity:
  - `aos8_triage_client(mac_address: str)`
  - `aos8_triage_ap(ap_name: str)`
  - `aos8_rf_analysis(config_path: str = "/md")`
  - `aos8_compare_md_config(md_path_1: str, md_path_2: str)`
  - `aos8_client_flood(config_path: str = "/md")`
  - `aos8_pre_change_check(config_path: str)` — required (no default)
  - Parameterless: `aos8_health_check`, `aos8_audit_change`, `aos8_wlan_review`
- **D-04:** Implement **all 9 prompts** (PROMPT-01..09). No deferrals.
- **D-05:** `prompts.py` in `src/hpe_networking_mcp/platforms/aos8/tools/` with a `register(mcp)` function. Registered in `aos8/__init__.py` in a `try/except` block after the meta-tools call — exactly mirrors `central/__init__.py`.
- **D-06:** Create a **new `INSTRUCTIONS.md` at repo root** (same level as `README.md`) covering:
  1. `config_path` semantics (Conductor hierarchy, `/md`, `/md/<device>`, `/md/<ap-group>`, standalone)
  2. `write_memory` contract — explicit, never auto-called, `requires_write_memory_for` field
  3. `show_command` passthrough — `_meta` stripping
  4. Conductor vs. standalone behavior
  5. **Guided prompt index** — list all 9 prompts with purpose and parameters
- **D-07:** **2-plan structure** (no TDD Wave 0):
  - **Plan 06-01:** `prompts.py` (all 9 prompts + `register()`) + `aos8/__init__.py` wiring + `tests/unit/test_aos8_prompts.py` smoke test
  - **Plan 06-02:** All documentation — `INSTRUCTIONS.md` (new), `README.md` (AOS8 row + secrets + tool count), `docs/TOOLS.md` (AOS8 tool reference), `CHANGELOG.md` (new version entry), `pyproject.toml` (version bump)
- **D-08:** **Registration smoke test only** — assert all 9 prompts are registered and return non-empty strings. No content assertions.

### Claude's Discretion

- Exact parameter names/defaults for parameterized vs. parameterless prompts (research below)
- Precise version number to bump (research below)
- Order of sections in `INSTRUCTIONS.md` (research below)
- Whether `README.md` has an existing AOS8 row stub or needs a fresh row (research below — needs fresh insertion)

### Deferred Ideas (OUT OF SCOPE)

None.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PROMPT-01 | `aos8_triage_client` — find client → AP health → auth events → root cause | Tools available: `aos8_find_client`, `aos8_get_client_detail`, `aos8_get_ap_detail`, `aos8_get_alarms`, `aos8_get_events`, `aos8_get_client_history` |
| PROMPT-02 | `aos8_triage_ap` — radio state, clients, alarms, events for an AP | `aos8_get_ap_detail`, `aos8_get_radio_summary`, `aos8_get_bss_table`, `aos8_get_alarms`, `aos8_get_arm_history`, `aos8_get_events` |
| PROMPT-03 | `aos8_health_check` — network-wide: APs, clients, alarms, firmware | `aos8_get_controllers`, `aos8_get_active_aps`, `aos8_get_ap_database`, `aos8_get_clients`, `aos8_get_alarms`, `aos8_get_version`, `aos8_get_controller_stats` |
| PROMPT-04 | `aos8_audit_change` — recent audit trail, summarize, flag risk | `aos8_get_audit_trail`, `aos8_get_events` |
| PROMPT-05 | `aos8_rf_analysis(config_path)` — channels, co-channel, ARM history | `aos8_get_radio_summary`, `aos8_get_arm_history`, `aos8_get_rf_monitor`, `aos8_get_bss_table` |
| PROMPT-06 | `aos8_wlan_review` — SSID/VAP/AP-group consistency | `aos8_get_ssid_profiles`, `aos8_get_virtual_aps`, `aos8_get_ap_groups`, `aos8_get_user_roles` |
| PROMPT-07 | `aos8_client_flood(config_path)` — high client counts, failed connections | `aos8_get_clients`, `aos8_get_active_aps`, `aos8_get_radio_summary`, `aos8_get_events`, `aos8_get_alarms` |
| PROMPT-08 | `aos8_compare_md_config(md_path_1, md_path_2)` — side-by-side config | `aos8_show_command` (with `config_path` for each MD), `aos8_get_ssid_profiles`, `aos8_get_virtual_aps`, `aos8_get_ap_groups` per scope |
| PROMPT-09 | `aos8_pre_change_check(config_path)` — health, pending changes, write_memory state | `aos8_get_alarms`, `aos8_get_controller_stats`, `aos8_get_audit_trail`, `aos8_show_command` (`show configuration committed`/`show configuration pending`), `aos8_write_memory` reminder |
| DOCS-01 | `INSTRUCTIONS.md` AOS8 section | New file at repo root per D-06 |
| DOCS-02 | `README.md` AOS8 row, secrets, auto-disable, tool count | Insert AOS8 column in capability table; add §Aruba OS 8 secrets; update tool count to 38 + 9 prompts; add aos8 to auto-disable example |
| DOCS-03 | `CHANGELOG.md` new version entry | Add `## [2.4.0.0] - 2026-04-28` block (minor bump — adds platform) |
| DOCS-04 | `docs/TOOLS.md` AOS8 reference | New `## Aruba OS 8 (38 tools + 9 prompts)` section after Axis with grouped subsections matching the 6 TOOLS dict keys |
| DOCS-05 | `pyproject.toml` version bump | `2.3.0.1` → `2.4.0.0` |
</phase_requirements>

## Summary

Phase 6 has zero new technical risk. The `@mcp.prompt` registration pattern, `register(mcp)` shim, and `try/except` wiring in `__init__.py` are already proven across two reference implementations (`central/tools/prompts.py` with 12 prompts + `mist/tools/prompts.py` with 2 prompts). All 38 AOS8 tools the prompts reference are already implemented and tested (737 tests green at end of Phase 5). The only research-meaningful questions concern documentation conventions and version semantics.

The doc work has more surface area than the code work but follows existing patterns with one small wrinkle: `INSTRUCTIONS.md` does **not** currently exist at the repo root — only at `src/hpe_networking_mcp/INSTRUCTIONS.md`. CONTEXT.md decision D-06 explicitly says "create a new `INSTRUCTIONS.md` at repo root". The planner must decide whether this is a *new file* alongside the existing in-package one, or whether the user intent was to amend the existing in-package file. **Recommendation: create the new file at `hpe-networking-mcp/INSTRUCTIONS.md` per D-06's verbatim language; leave the in-package one untouched (the in-package one is loaded into the MCP `instructions=` field by `server.py` and serves a different purpose — it's the LLM-facing system prompt, not human-facing operator docs).** Flag this for explicit confirmation in the plan.

**Primary recommendation:** Mirror Central's `prompts.py` line-for-line in structure. For docs, mirror the existing AOS8-adjacent platform precedent (Axis was added in v2.2.x — its CHANGELOG entry is a good shape template). Bump to **2.4.0.0** (minor: adds a platform).

## Standard Stack

### Core (already in place — no new deps)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `fastmcp[code-mode]` | >= 3.1.1 | `@mcp.prompt` decorator | Already used by Central + Mist prompts |
| `loguru` | >= 0.7.3 | `logger.debug/warning` for prompt registration logging | Pattern from `central/__init__.py` |
| `pytest` + `pytest-asyncio` | 8.4 / 1.2 | Smoke test framework | Project default |

**No new dependencies.** Phase 6 adds zero packages to `pyproject.toml`.

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Per-prompt `@mcp.prompt` decorator | Skill markdown file in `skills/` | Skills are runbooks for the AI to *load on demand*. Prompts are first-class MCP primitives clients surface in their UI (Claude Desktop shows them in the slash-command menu). PROMPT-01..09 must be **prompts**, not skills, per the requirement language. |
| `register(mcp)` shim | Direct module-import side effects via `_registry.mcp` | Central uses `register(mcp)` for prompts because prompts are not auto-imported via the `TOOLS` dict loop. Mirror Central. |

## Architecture Patterns

### Recommended Project Structure

```
hpe-networking-mcp/
├── INSTRUCTIONS.md                            # NEW — repo-root operator-facing AOS8 docs (D-06)
├── README.md                                  # MODIFY — capability table, secrets, auto-disable, tool count
├── CHANGELOG.md                               # MODIFY — add 2.4.0.0 entry
├── pyproject.toml                             # MODIFY — version bump 2.3.0.1 → 2.4.0.0
├── docs/
│   └── TOOLS.md                               # MODIFY — add AOS8 section after Axis
├── src/hpe_networking_mcp/
│   └── platforms/aos8/
│       ├── __init__.py                        # MODIFY — add prompts try/except after build_meta_tools
│       └── tools/
│           └── prompts.py                     # NEW — 9 prompts + register(mcp)
└── tests/unit/
    └── test_aos8_prompts.py                   # NEW — 9 smoke tests
```

### Pattern 1: Prompt Module File Structure

```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/prompts.py
"""AOS8 guided prompts for operator workflows."""


def register(mcp):

    @mcp.prompt
    def aos8_health_check() -> str:
        """Network-wide AOS8 health assessment."""
        return """
Provide a network-wide AOS8 health overview by following these steps:

1. Call `aos8_get_controllers` to enumerate every controller/MD with role and status.
2. Call `aos8_get_version` to capture firmware versions across the Conductor and MDs;
   flag any version drift.
3. Call `aos8_get_active_aps` to count currently-up APs and association totals.
4. Call `aos8_get_alarms` to pull current active alarms; group by severity and category.
5. Call `aos8_get_controller_stats` for any MD whose role is master or standby — capture
   CPU, memory, and session counts.

Summarize: total controller count by role, AP up/down counts, client total, alarm
breakdown by severity, firmware drift if any, and one recommended next action per
risk area.
        """.strip()

    # ... 8 more prompts ...
```

**Key conventions copied from Central:**
- Module docstring at top
- `register(mcp)` is a plain `def`, NOT `async def`
- Each prompt is a plain `def` (NOT `async def`) returning `str`
- One-line docstring on each prompt — surfaces as the prompt's description in MCP clients
- Prompt body is a `"""\n...\n        """.strip()` triple-quoted string with leading/trailing newline stripped
- Numbered steps with explicit tool names in backticks
- Parameters interpolated with f-strings: `f'... mac_address="{mac_address}"'`
- End with `Summarize: ...` line listing what to report and recommended next steps

### Pattern 2: `__init__.py` Wiring (mirror Central)

```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/__init__.py lines 125-133
# Insertion point in aos8/__init__.py: after the for-loop that imports TOOLS modules,
# before the `if mode == "dynamic":` block.

try:
    from hpe_networking_mcp.platforms.aos8.tools import prompts

    prompts.register(mcp)
    logger.debug("AOS8: loaded prompts")
except Exception as e:
    logger.warning("AOS8: failed to load prompts -- {}", e)
```

**Why `try/except`:** prompts are non-critical to tool surface. If prompts.py has a bug, tools still register and the platform is still usable. Central uses the same pattern.

### Pattern 3: Smoke Test Structure

```python
# Source: hpe-networking-mcp/tests/unit/test_aos8_init.py (structure pattern)
"""Smoke tests for AOS8 guided prompts registration."""

from __future__ import annotations
from unittest.mock import MagicMock

import pytest

pytestmark = pytest.mark.unit

EXPECTED_PROMPTS = {
    "aos8_triage_client",
    "aos8_triage_ap",
    "aos8_health_check",
    "aos8_audit_change",
    "aos8_rf_analysis",
    "aos8_wlan_review",
    "aos8_client_flood",
    "aos8_compare_md_config",
    "aos8_pre_change_check",
}


def test_register_attaches_all_nine_prompts():
    """register(mcp) must decorate exactly the 9 PROMPT-01..09 functions."""
    from hpe_networking_mcp.platforms.aos8.tools import prompts

    captured: dict[str, callable] = {}

    def fake_prompt(fn):
        captured[fn.__name__] = fn
        return fn

    fake_mcp = MagicMock()
    fake_mcp.prompt = fake_prompt

    prompts.register(fake_mcp)

    assert set(captured.keys()) == EXPECTED_PROMPTS
    for name, fn in captured.items():
        # parameter-less and parameter-bearing prompts both return str
        sig_params = fn.__code__.co_varnames[: fn.__code__.co_argcount]
        # call with placeholder strings for any required params
        kwargs = {p: "x" for p in sig_params}
        result = fn(**kwargs)
        assert isinstance(result, str) and result, f"{name} returned empty"
```

**Why one combined test, not 9:** D-08 says "smoke test only". A single registration probe covers all 9 names + the non-empty contract. Splitting into 9 tests adds noise without coverage value. If the planner prefers 9 separate tests for trace clarity, parametrize:

```python
@pytest.mark.parametrize("prompt_name", sorted(EXPECTED_PROMPTS))
def test_prompt_registered(prompt_name): ...
```

### Pattern 4: Documentation Section Order

**`INSTRUCTIONS.md` recommended section order** (no existing structure precedent at repo root, so adopt logical operator-onboarding order):

1. **Overview** — what AOS8 module does, link to README for setup
2. **Connecting to Conductor vs. Standalone** — how to connect to each, conceptual difference
3. **`config_path` semantics** — `/md`, `/md/<device>`, `/md/<ap-group>`, hierarchy rules
4. **The `write_memory` contract** — explicit operator action, `requires_write_memory_for`, never implicit
5. **Using `aos8_show_command`** — passthrough, `_meta` stripping, structured JSON
6. **Guided prompts index** — table of all 9 prompts with parameters and one-line purpose

This order matches how Central documentation is structured (concepts → tools → workflows).

### Anti-Patterns to Avoid

- **Don't `async def` prompt functions** — Central and Mist both use plain `def`. FastMCP unwraps differently for async, and there's no I/O happening inside a prompt body (it's a string template).
- **Don't reference tools by partial name** — always use the full `aos8_<name>` for clarity to the AI. Pattern from Central uses `central_get_site_health`, not `get_site_health`.
- **Don't inline parameter dicts** — write `param=value` style in prose, not `{"param": "value"}`. The AI is being instructed in English, not given JSON.
- **Don't auto-call `aos8_write_memory` inside prompts** — per WRITE-12 contract, `write_memory` is *always* an explicit operator action. Prompts may *remind* the operator to call it, never instruct the AI to call it without confirmation.
- **Don't bump major version** — adding a platform is additive, not breaking. Use 2.4.0.0 (minor) not 3.0.0.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Prompt registration | Custom decorator or registry | `@mcp.prompt` | FastMCP handles MCP-protocol prompt advertisement |
| Prompt invocation | Pre-rendered prompt strings stored elsewhere | f-string return inside the decorated function | Standard FastMCP pattern; lets MCP clients pass parameters at invocation time |
| Multi-step orchestration | A "prompt runner" tool that calls other tools | Just describe steps in plain English | LLM follows the numbered steps; no code needed |
| Prompt content tests | Asserting on substring presence | Smoke test only (per D-08) | String content is brittle and high-maintenance |

**Key insight:** Prompts are templates, not code. Every test that asserts on string content becomes a maintenance tax with negative value. Trust the LLM to follow the numbered steps — that's the contract.

## Common Pitfalls

### Pitfall 1: Prompts Don't Register Because the Module Was Never Imported

**What goes wrong:** Adding `prompts.py` to `aos8/tools/` is not enough; the module must be explicitly imported by `aos8/__init__.py:register_tools()` because it's NOT in the `TOOLS` dict (TOOLS is a tools-only registry; prompts are a different MCP primitive).

**Why it happens:** The TOOLS dict + importlib loop only imports modules listed in TOOLS. `prompts.py` would be silently skipped.

**How to avoid:** Add the explicit `try/except: from ... import prompts; prompts.register(mcp)` block as in Pattern 2 above. Mirror Central exactly.

**Warning signs:** No `AOS8: loaded prompts` debug log line at startup; MCP client doesn't show the prompts in its slash-command menu.

### Pitfall 2: f-string Quoting Inside Prompt Bodies

**What goes wrong:** Prompts that mention dict-like syntax (e.g. `payload={...}`) inside an f-string break because of unbalanced braces.

**Why it happens:** f-strings interpret single `{` / `}` as expression delimiters.

**How to avoid:** Use double-braces: `payload={{key: value}}`. See Mist's `provision_site_from_template` prompt for the canonical pattern.

**Warning signs:** `SyntaxError: f-string` at import time, or curly braces silently disappearing from the rendered prompt text.

### Pitfall 3: `INSTRUCTIONS.md` Naming Collision

**What goes wrong:** The repo *already* has `src/hpe_networking_mcp/INSTRUCTIONS.md` (541 lines, loaded as MCP `instructions=` for the LLM). D-06 says create a new `INSTRUCTIONS.md` at repo root. If the planner edits the in-package file by mistake, it bloats the LLM system prompt; if they create the new one without realizing the in-package one exists, they may duplicate setup content.

**Why it happens:** The in-package file is in a non-obvious location.

**How to avoid:**
- The new repo-root file is **operator-facing** (human reading) — concepts, semantics, prompt index
- The in-package file is **AI-facing** (system prompt) — tool catalog, discovery rules, ID-resolution patterns
- Do not edit the in-package file in this phase. The AOS8 entries inside it can be added in a future v2.4.x patch if the AI needs explicit AOS8 routing guidance.

**Warning signs:** Planner proposes editing `src/hpe_networking_mcp/INSTRUCTIONS.md`; reviewer should redirect to repo-root location.

### Pitfall 4: Tool Count Drift in README

**What goes wrong:** README lists tool counts in three places — capability table footer, "default tool surface" sentence, and ASCII architecture diagram. Updating one and forgetting the others creates drift.

**How to avoid:** When bumping README, grep for the current count strings (`35 + 2 prompts`, `73 + 12 prompts`, `25 underlying tools`, etc.) and update *every* occurrence. Add the AOS8 column to all three places: capability table row, default-tool-surface sentence, and ASCII diagram box.

**Warning signs:** Reviewer sees "5 platforms × 3 = 15 meta-tools" in one paragraph but "6 platforms" in another.

### Pitfall 5: CHANGELOG Version Format

**What goes wrong:** Project uses 4-segment SemVer-ish versioning (`2.3.0.1`, `2.3.0.0`, `2.2.0.4`). Picking a 3-segment number breaks the table format.

**How to avoid:** Use `2.4.0.0` (full 4-segment, third+fourth digit zero). Cross-reference: 2.3.0.0 was "added Skills system"; 2.3.0.1 was "added one skill + bug docs"; 2.4.0.0 = "added AOS8 platform".

## Code Examples

### Example 1: Parameterless Prompt

```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/prompts.py:151-165
@mcp.prompt
def aos8_health_check() -> str:
    """Network-wide AOS8 health assessment across Conductor and MDs."""
    return """
Provide a network-wide AOS8 health overview:

1. Call `aos8_get_controllers` to enumerate all controllers and MDs with role and status.
2. Call `aos8_get_version` to capture firmware on Conductor and each MD.
3. Call `aos8_get_active_aps` to count up-APs and association totals.
4. Call `aos8_get_alarms` to pull active alarms; group by severity (Critical > Major > Minor > Warning).
5. Call `aos8_get_controller_stats` for the master and standby Conductors — note CPU, memory, sessions.

Summarize: controller count by role, AP up/down totals, client total, alarm breakdown
by severity, firmware drift if any, and one recommended next action per risk area.
    """.strip()
```

### Example 2: Parameterized Prompt with Required Param

```python
# Source: hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/prompts.py:38-52 (analogue)
@mcp.prompt
def aos8_triage_client(mac_address: str) -> str:
    """Triage a wireless client connectivity problem by MAC address."""
    return f"""
Triage client connectivity for MAC "{mac_address}":

1. Call `aos8_find_client` with mac_address="{mac_address}" to locate the client and its
   currently associated AP, BSSID, SSID, VLAN, and role.
2. If found, capture the AP name from the response and call `aos8_get_ap_detail`
   with that AP name — note radio state, channel, power, and last-reboot time.
3. Call `aos8_get_client_detail` for the same MAC for full auth/association/encryption
   details and any failed-auth counters.
4. Call `aos8_get_client_history` for the same MAC over the recent window — look for
   association flapping, repeated auth failures, or roam thrash.
5. Call `aos8_get_alarms` filtered to the AP's scope to surface AP- or controller-level
   issues that may be the upstream cause.

Summarize: client current state, AP health, auth/association event timeline, likely
root cause (auth failure, RF issue, AP outage, roaming), and the next operator action.
    """.strip()
```

### Example 3: Parameterized Prompt with Default

```python
@mcp.prompt
def aos8_rf_analysis(config_path: str = "/md") -> str:
    """RF environment report for an AOS8 scope: channels, co-channel, ARM history."""
    return f"""
Produce an RF environment report scoped to config_path "{config_path}":

1. Call `aos8_get_radio_summary` with config_path="{config_path}" — collect channel,
   bandwidth, TX power, channel utilization, and noise floor per radio per AP.
2. Group radios by band (2.4 / 5 / 6 GHz) and by channel; flag any channel with >3 APs
   in the same scope (co-channel cluster).
3. Call `aos8_get_arm_history` with config_path="{config_path}" to see recent
   ARM-driven channel and power changes; flag radios that have changed >3 times in 24h.
4. Call `aos8_get_rf_monitor` with config_path="{config_path}" for interference and
   rogue detection signal.
5. Call `aos8_get_bss_table` with config_path="{config_path}" to see BSSID density per AP.

Summarize: channel-distribution histogram per band, co-channel clusters, top
oscillating radios, top interferers/rogues, and recommended channel-plan or
power-adjustment actions.
    """.strip()
```

### Example 4: Pre-change Check (most operator-safety-critical — D-09 spec)

```python
@mcp.prompt
def aos8_pre_change_check(config_path: str) -> str:
    """Pre-maintenance checklist before any AOS8 change at a config_path."""
    return f"""
Pre-change checklist for config_path "{config_path}". Run BEFORE any planned
configuration change or maintenance window:

1. Call `aos8_get_alarms` and confirm there are no active Critical or Major alarms
   at or below "{config_path}". If there are, recommend the operator address those
   first or proceed with explicit acknowledgement.
2. Call `aos8_get_controller_stats` for the master Conductor and any MD under
   "{config_path}" — confirm CPU < 80%, memory < 80%, and uptime is reasonable.
3. Call `aos8_get_audit_trail` for "{config_path}" over the last 24h — surface any
   recent changes that the operator may not be aware of (someone else mid-change?).
4. Call `aos8_show_command` with command="show configuration committed" and
   config_path="{config_path}" to capture the *current* committed config baseline.
5. Call `aos8_show_command` with command="show configuration pending" and
   config_path="{config_path}" — if there are pending changes, WARN the operator:
   the planned change will be persisted alongside someone else's staged work when
   `aos8_write_memory` is called.

Summarize: GO / NO-GO recommendation, current alarm state, controller resource state,
recent change history, pending-change warning if applicable, and a reminder that
after the change the operator MUST call `aos8_write_memory` with the affected
config_path(s) — `write_memory` is never called automatically.
    """.strip()
```

### Example 5: `__init__.py` Insertion (the diff)

```python
# In src/hpe_networking_mcp/platforms/aos8/__init__.py, after the existing
# `for category, names in TOOLS.items(): ... importlib.import_module(...)` loop
# and BEFORE the `mode = config.tool_mode` line.

    try:
        from hpe_networking_mcp.platforms.aos8.tools import prompts

        prompts.register(mcp)
        logger.debug("AOS8: loaded prompts")
    except Exception as e:
        logger.warning("AOS8: failed to load prompts -- {}", e)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Prompts hand-coded into the LLM system prompt | `@mcp.prompt` first-class MCP primitive | MCP spec ~2024 | Prompts are now discoverable by clients (Claude Desktop slash menu, etc.) and parameterizable |
| Single `INSTRUCTIONS.md` blob | Two-tier docs: in-package (AI-facing) + repo-root (operator-facing) | This phase introduces the split | Operator docs no longer bloat the LLM context; AI sees only what it needs |

**Deprecated/outdated:** None applicable — FastMCP `@mcp.prompt` is the current standard pattern.

## Open Questions

1. **Should the new repo-root `INSTRUCTIONS.md` link to or summarize the existing in-package one?**
   - What we know: in-package file at `src/hpe_networking_mcp/INSTRUCTIONS.md` is loaded by `server.py` as the MCP `instructions=` field for AI consumption.
   - What's unclear: whether the operator reading repo-root INSTRUCTIONS.md should be told "the AI also gets a copy of these rules in its system prompt at runtime — see src/hpe_networking_mcp/INSTRUCTIONS.md".
   - Recommendation: add a one-line cross-reference at the top of the new file: *"For the AI-facing system prompt that ships inside the container, see [src/hpe_networking_mcp/INSTRUCTIONS.md](src/hpe_networking_mcp/INSTRUCTIONS.md)."* — this resolves Pitfall 3 confusion for future maintainers.

2. **Tool count math for README footer.**
   - What we know: current README says `35 + 2 prompts | 73 + 12 prompts | 10 | 140 | 19 | 25` for Mist/Central/GreenLake/ClearPass/Apstra/Axis. Phase 6 adds an AOS8 column with 38 tools + 9 prompts.
   - What's unclear: nothing operationally — just confirm the planner adds `38 + 9 prompts` exactly.
   - Recommendation: planner Plan 06-02 should grep README for every count string and update each to include AOS8.

3. **Whether to add AOS8 to the `Guided Prompts` row in the capability table.**
   - What we know: that row currently shows `✅` for Mist (2 prompts) and Central (12 prompts), `—` for the rest.
   - Recommendation: yes — set AOS8 column to `✅` since this phase delivers 9 prompts.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| FastMCP `@mcp.prompt` | All 9 prompts + register block | ✓ | 3.1.1+ (per `pyproject.toml`) | — |
| `loguru` | logger.debug/warning in __init__.py | ✓ | 0.7.3+ | — |
| `pytest` + `pytest-asyncio` | Smoke test | ✓ | 8.4 / 1.2 (already installed; 737 tests passing) | — |

**Missing dependencies with no fallback:** None.

**Missing dependencies with fallback:** None.

Phase 6 is purely code + docs. No external services, no new tools, no runtime dependencies beyond what's already in place.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `pytest` 8.4.0+ with `pytest-asyncio` 1.2.0+ (`asyncio_mode = "auto"`) |
| Config file | `pyproject.toml` `[tool.pytest.ini_options]` |
| Quick run command | `uv run pytest tests/unit/test_aos8_prompts.py -x` |
| Full suite command | `uv run pytest tests/ -q` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PROMPT-01 | `aos8_triage_client` registered + returns non-empty str | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k triage_client` | ❌ Plan 06-01 creates it |
| PROMPT-02 | `aos8_triage_ap` registered + returns non-empty str | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k triage_ap` | ❌ Plan 06-01 |
| PROMPT-03 | `aos8_health_check` registered | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k health_check` | ❌ Plan 06-01 |
| PROMPT-04 | `aos8_audit_change` registered | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k audit_change` | ❌ Plan 06-01 |
| PROMPT-05 | `aos8_rf_analysis` registered + accepts default config_path | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k rf_analysis` | ❌ Plan 06-01 |
| PROMPT-06 | `aos8_wlan_review` registered | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k wlan_review` | ❌ Plan 06-01 |
| PROMPT-07 | `aos8_client_flood` registered + accepts default config_path | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k client_flood` | ❌ Plan 06-01 |
| PROMPT-08 | `aos8_compare_md_config` registered + 2 required params | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k compare_md_config` | ❌ Plan 06-01 |
| PROMPT-09 | `aos8_pre_change_check` registered + required config_path | unit (smoke) | `pytest tests/unit/test_aos8_prompts.py -x -k pre_change_check` | ❌ Plan 06-01 |
| DOCS-01..05 | Documentation present | manual-only | reviewer reads diffs | ✅ existing files; `INSTRUCTIONS.md` is new |

**Manual-only justification for DOCS-01..05:** doc completeness/accuracy is human judgement. Automating "the README has an AOS8 row" is doable but provides no signal beyond a grep — which a reviewer does naturally. No test infrastructure planned.

### Sampling Rate

- **Per task commit:** `uv run pytest tests/unit/test_aos8_prompts.py -x` (< 2 sec)
- **Per wave merge:** `uv run pytest tests/ -q` (full 737+9=746 test suite)
- **Phase gate:** Full suite green before `/gsd:verify-work`; reviewer eyeball pass on each doc diff

### Wave 0 Gaps

- [ ] `tests/unit/test_aos8_prompts.py` — covers PROMPT-01..09 (created in Plan 06-01)

*(No framework install needed — pytest infrastructure has been operational since Phase 1.)*

## Project Constraints (from CLAUDE.md)

- **File size:** max 500 lines per file. `prompts.py` for 9 prompts at ~20 lines each = ~180 lines. Well under. `INSTRUCTIONS.md` is markdown, not code, so the 500-line rule doesn't apply, but keep it focused.
- **Function size:** max 50 lines per function. Each prompt body is one `def` returning a triple-quoted string; bodies will hit ~15-25 lines each. Under.
- **Style:** ruff lint + format, mypy type-checked. Each prompt has `def name(...) -> str:` with proper return type annotation. Run `uv run ruff format src/hpe_networking_mcp/platforms/aos8/tools/prompts.py` before commit.
- **Docstrings:** Google-style, but for `@mcp.prompt` functions a single one-line docstring is sufficient and matches Central — the docstring becomes the prompt's MCP-protocol description, so it should be a single sentence describing the workflow.
- **Compatibility:** must not break any existing platform module or test. Confirmed: prompts.py is additive; `__init__.py` change is additive (adds a try/except block).
- **GSD workflow:** This phase is being executed under `/gsd:plan-phase` → `/gsd:execute-phase`. Direct edits outside of this workflow are not permitted.
- **Branch strategy from project CLAUDE.md (parent repo):** `feature/*` branches; main is protected. The `.planning/config.json` has `"branching_strategy": "none"` which **overrides** the project default for GSD work — Phase 6 commits to current branch, no new branch needed.
- **Documentation Checklist (project CLAUDE.md):** EVERY code PR must update README, CHANGELOG, INSTRUCTIONS, docs/TOOLS.md, pyproject.toml. Phase 6 satisfies this in Plan 06-02 — it IS the documentation update. The CLAUDE.md rule means: when Plan 06-01 is committed without docs, Plan 06-02 must follow before any merge to main.
- **Commit message format:** "Never include claude code, or written by claude code in commit messages." (project CLAUDE.md). Honor this when running `gsd-tools.cjs commit`.

## Sources

### Primary (HIGH confidence)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/tools/prompts.py` — canonical prompt module, 12 prompts implementing the exact pattern Phase 6 needs to replicate
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/central/__init__.py` lines 125-133 — `try/except prompts.register(mcp)` block to mirror in `aos8/__init__.py`
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/mist/tools/prompts.py` — second reference for the `register(mcp)` pattern, validates Central isn't a one-off
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/__init__.py` — current state of file to modify (lines 86-99 show the existing for-loop and mode-switch structure)
- `hpe-networking-mcp/src/hpe_networking_mcp/platforms/aos8/tools/` directory listing — confirms all 38 tools exist; prompts.py is the only file to add
- `hpe-networking-mcp/README.md` — current capability table, ASCII diagram, secrets section to update
- `hpe-networking-mcp/CHANGELOG.md` lines 1-60 — established CHANGELOG format (Keep a Changelog; 4-segment versioning; section header + intro paragraph + subheaders)
- `hpe-networking-mcp/pyproject.toml` line 7 — current version is `2.3.0.1`
- `hpe-networking-mcp/docs/TOOLS.md` — section structure to extend with AOS8 (after Axis at the bottom)
- `hpe-networking-mcp/src/hpe_networking_mcp/INSTRUCTIONS.md` — confirms the in-package AI-facing instructions file already exists; new repo-root file is distinct
- `hpe-networking-mcp/tests/unit/test_aos8_init.py` — test file structure pattern to mirror in `test_aos8_prompts.py`
- `.planning/phases/06-guided-prompts-documentation/06-CONTEXT.md` — locked decisions D-01..D-08
- `.planning/REQUIREMENTS.md` §PROMPT and §DOCS — PROMPT-01..09 and DOCS-01..05 specifications
- `.planning/STATE.md` — confirms 737 tests green at end of Phase 5, 38 AOS8 tools wired
- `./CLAUDE.md` (project) — file size, style, GSD workflow, commit message conventions

### Secondary (MEDIUM confidence)
- None — this phase is documented entirely from in-repo primary sources.

### Tertiary (LOW confidence)
- None.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — pattern proven across 2 implementations in this exact repo
- Architecture: HIGH — `register(mcp)` + `try/except` wiring is verified in `central/__init__.py`
- Pitfalls: HIGH — Pitfall 3 (INSTRUCTIONS.md naming) discovered during research by reading filesystem; others are derived from reading the existing pattern files
- Documentation conventions: HIGH — README, CHANGELOG, TOOLS.md formats inspected directly
- Version-bump choice: HIGH — 4-segment SemVer with minor-bump-for-platform precedent established by Axis (v2.2.x added Axis platform)

**Research date:** 2026-04-28
**Valid until:** 2026-05-28 (FastMCP API surface for prompts is stable; only doc state could drift if other PRs land before this phase executes)
