# Phase 2: API Client - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-28
**Phase:** 02-api-client
**Areas discussed:** Health check depth, Logout behavior, server.py scope, TDD sequencing

---

## Health Check Depth

| Option | Description | Selected |
|--------|-------------|----------|
| Richer: hostname + version | GET controller version via show command — health tool shows controller hostname and AOS version string alongside other platforms. Matches what Mist/Central surface. | ✓ |
| Reachable-only | Just verify login succeeds — health tool reports reachable/unreachable. Simpler but gives operators less at a glance. | |

**User's choice:** Richer: hostname + version
**Notes:** None provided — selected recommended option.

---

## Logout Behavior on Shutdown

| Option | Description | Selected |
|--------|-------------|----------|
| Explicit logout | POST to AOS8 logout endpoint on shutdown — releases the session on the Conductor, prevents session exhaustion over repeated server restarts. Errors logged as WARNING and swallowed. | ✓ |
| Close-only, no logout | Just close the httpx client — matches Apstra pattern. AOS8 sessions expire naturally. Simpler, but orphaned sessions accumulate if the server restarts frequently. | |

**User's choice:** Explicit logout
**Notes:** None provided — selected recommended option.

---

## server.py Scope for Phase 2

| Option | Description | Selected |
|--------|-------------|----------|
| Full wiring | Lifespan + health probe + write-gate visibility transform + `_register_aos8_tools()` stub. Phase 3 only needs to add tools to the stub — no server.py changes needed. | ✓ |
| Minimal: lifespan + health only | Wire lifespan and health probe now. Defer write-gate visibility and `_register_aos8_tools()` to Phase 3 when tools actually exist. Slightly cleaner per-phase boundary but Phase 3 touches server.py. | |

**User's choice:** Full wiring
**Notes:** None provided — selected recommended option.

---

## TDD Sequencing

| Option | Description | Selected |
|--------|-------------|----------|
| TDD alongside client | Write client tests in Phase 2 — same pattern as Phase 1 (01-01-PLAN was test scaffolding first). Client tests pass in CI on phase completion. Phase 7 adds tool-level and regression tests on top. | ✓ |
| Defer all tests to Phase 7 | Follow the requirements traceability table strictly — TEST-01 is mapped to Phase 7. Phase 2 ships client code only; Phase 7 tests everything. Risk: client bugs discovered late. | |

**User's choice:** TDD alongside client
**Notes:** None provided — selected recommended option.

---

## Claude's Discretion

- Exact AOS8 show-command path used by `health_check()` (researcher to verify)
- Exact AOS8 logout endpoint path (researcher to verify)
- Whether `health_check()` returns a dict or raises on unreachable
- Internal variable naming within `client.py`

## Deferred Ideas

None — discussion stayed within phase scope.
