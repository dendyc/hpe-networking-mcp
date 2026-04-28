# Phase 5: Write Tools - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-28
**Phase:** 05-write-tools
**Areas discussed:** File organization, write_memory response contract, Write payload design, AOS8 delete mechanics

---

## File Organization

| Option | Description | Selected |
|--------|-------------|----------|
| One writes.py | All 12 tools in a single file (~350-400 lines, under 500-line limit). Mirrors Phase 4 differentiators.py approach. | ✓ |
| Split: config + operational | config_writes.py (WRITE-01..09) + operational_writes.py (WRITE-10..12). Two files to wire and test. | |
| You decide | Claude picks based on line count after implementation. | |

**User's choice:** One writes.py
**Notes:** Single file keeps it simple; mirrors Phase 4 differentiators.py precedent.

---

## write_memory Response Contract

| Option | Description | Selected |
|--------|-------------|----------|
| Always include the field | Every write tool returns `requires_write_memory_for` — config writes have the config_path, operational tools have empty list. AI can always trust the field is present. | ✓ |
| Conditional — only when needed | Config writes include it; operational tools omit it. Less predictable for AI callers. | |
| Include + check pending changes | Config writes return field AND make a follow-up call to check for existing pending changes. | |

**User's choice:** Always include the field
**Notes:** Unconditional presence simplifies AI caller logic. WRITE-12 (write_memory itself) is the one exception — no circular `requires_write_memory_for` in its own response.

---

## Write Payload Design

| Option | Description | Selected |
|--------|-------------|----------|
| Freeform payload: dict | `payload: dict` + `config_path` + `action_type`. Matches Central pattern. Schema knowledge in docstrings. | ✓ |
| Typed params per tool | Named parameters per tool for common fields. More verbose, harder to maintain. | |
| Hybrid: common typed + extras: dict | Required fields typed, optional fields in extras dict. More discoverable but more complex. | |

**User's choice:** Freeform payload: dict
**Notes:** Consistent with Central's `central_manage_wlan_profile` pattern. Researcher documents key payload fields per object type in the plan.

---

## AOS8 Delete Mechanics

| Option | Description | Selected |
|--------|-------------|----------|
| Researcher finds the endpoint | Researcher verifies the AOS8 API delete path per object type. Most likely POST with body flag or dedicated path. | ✓ |
| Delete via config-object DELETE body convention | Assume POST to config-object endpoint with `{"_action": "delete"}` body for all 9 types. | |
| Separate aos8_delete_X tools | Split manage_X (create/update) from delete — more tools (18 vs 9) but cleaner tag separation. | |

**User's choice:** Researcher finds the endpoint
**Notes:** AOS8 is POST-only so researcher must confirm delete mechanism for all 9 config object types. If a single pattern covers all, a shared helper can be used; if patterns differ, each function handles its own delete path.

---

## Claude's Discretion

- Exact AOS8 REST endpoint paths per object type (researcher to verify)
- Whether AOS8 uses one delete pattern or per-object delete paths
- JSON fixture file names for write tool test mocks
- `ToolAnnotations` constant naming in writes.py
- Whether `aos8_disconnect_client` and `aos8_reboot_ap` are idempotent

## Deferred Ideas

None — discussion stayed within phase scope.
