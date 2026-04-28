# Domain Pitfalls — AOS8 / Mobility Conductor MCP Module

**Domain:** Aruba OS 8 (AOS8) / Mobility Conductor REST API integration as a 7th platform module in the existing `hpe-networking-mcp` FastMCP server
**Researched:** 2026-04-27
**Overall confidence:** HIGH (sourced from AOS8 API Guide, existing codebase patterns in `apstra/client.py` and `middleware/retry.py`, and CONCERNS.md observations)

---

## Critical Pitfalls

These cause session lockouts, silent config drift, security incidents, or rewrites. Address every one of these in the client/middleware design phase before writing any tool.

### Pitfall C1: Re-login per request → Conductor session exhaustion

**What goes wrong:** Each tool invocation calls `POST /v1/api/login` to get a fresh `UIDARUBA` cookie/token instead of reusing a cached one. After tens to a few hundred logins the Conductor refuses new sessions or evicts existing ones; legitimate operators get logged out of the GUI/CLI.

**Why it happens:** The "obvious" implementation pattern from REST tutorials is "auth in the request helper." The existing `ApstraClient` (`platforms/apstra/client.py`) gets this right by caching `self._token` behind an `asyncio.Lock` and refreshing only on 401 — that pattern must be copied. Authors who skip studying the Apstra client and instead "just write a quick wrapper" land here.

**Consequences:**
- Conductor session table fills up; `MAX_SESSIONS_REACHED` errors begin
- Legitimate admins kicked out
- Audit log spam (every tool call = login + logout entry)
- AOS8 docs explicitly warn against this — vendor-acknowledged anti-pattern

**Prevention:**
1. Single `AOS8Client` instance per server lifespan (constructed in `server.py:lifespan()` and stored in `ctx.lifespan_context["aos8_client"]`).
2. `_token: str | None` cached on the instance, guarded by `asyncio.Lock` (mirror `ApstraClient._ensure_token` / `_refresh_token` exactly).
3. Refresh only on 401 response, not preemptively.
4. Implement `aclose()` that calls `POST /v1/api/logout` on shutdown — register in lifespan teardown so we don't leak a session on container stop.
5. Do NOT log out per request — only on lifespan close.

**Detection (warning signs):**
- Login count in audit log grows linearly with tool calls
- Conductor returns HTTP 503 "Max sessions" after sustained use
- `UIDARUBA` value changes between calls in debug logs

**Phase:** Phase 1 (client foundation). Must be correct before any tool is written.

---

### Pitfall C2: UIDARUBA token leaked to logs / error messages

**What goes wrong:** The session token is logged at INFO/DEBUG, included in `format_http_error()` output, or surfaced in tool responses. A token captured from logs grants full Conductor admin until logout.

**Why it happens:** Apstra's `_login_locked()` uses `mask_secret(token)` (`utils/logging.py`) — easy to forget to apply this in a new client. AOS8's token also rides in **query strings** (`?UIDARUBA=...`) on showcommand calls, which means request-URL logging at DEBUG level will capture it verbatim.

**Consequences:**
- Token in logs persists indefinitely (until logout) and grants full admin on the Conductor
- Bandit/secret-scanning tools may not catch it because it doesn't match a known credential pattern
- Tokens captured from MCP error responses leak to LLM transcripts

**Prevention:**
1. Use `mask_secret()` on every token reference in log statements.
2. Strip `UIDARUBA` from query params before logging request URLs — wrap any URL log call with a sanitizer.
3. In `format_http_error()`, redact `UIDARUBA` from `response.text` and `response.url` before returning the body — `mask_error_details=True` is set on the FastMCP server but the dict we return bypasses that.
4. Never put the token in tool responses (no "session info" tool that exposes it).
5. Add a unit test that asserts `UIDARUBA=` does not appear in any captured log record.

**Phase:** Phase 1 (client foundation). Token redaction must ship with the first login implementation.

---

### Pitfall C3: Write Memory not called → config silently lost on reload

**What goes wrong:** A write tool successfully POSTs a config object change. The runtime config updates and the change appears to take effect. The Conductor (or MD) reboots — power outage, planned maintenance, ArubaOS upgrade — and the change is **gone** because `write memory` was never called for that `config_path`.

**Why it happens:** Unlike most REST APIs where PUT/POST is durable, AOS8 separates **runtime** from **startup** config. Without an explicit `POST /v1/configuration/object/write_memory?config_path=<path>` the change lives only in running-config. Operators (and LLM agents) used to other platforms assume "the API returned 200, we're done."

**Consequences:**
- Config drift discovered after reboot — often weeks later
- Difficult to diagnose: the original change worked, the post-reboot device "lost" the setting
- Compliance / change-control nightmare — audit logs show the change happened, the device has reverted

**Prevention:**
1. Make `aos8_write_memory` an explicit, separate tool tagged `aos8_write` (NOT auto-called).
2. Every write tool's response MUST include a `config_persisted: false` and a `requires_write_memory_for: [<config_path>]` field, surfacing the obligation to the LLM/operator.
3. Consider a per-client tracker: after any successful POST that mutates config, record the `config_path`; the `aos8_write_memory` tool defaults its parameter from that pending set.
4. Document this prominently in `INSTRUCTIONS.md` for the AOS8 section so the LLM knows the contract.
5. Optionally: add an `aos8_pending_changes` read tool that lists config_paths with unpersisted changes (Conductor exposes a "pending" indicator on each MD).
6. Do NOT auto-call write_memory inside write tools — operator intent must drive it (matches the "Write Memory as explicit tool" decision in PROJECT.md, and the staged-write pattern from Axis).

**Detection (warning signs):**
- Devices "lose" config after reboot
- Conductor "pending changes" indicator persists across sessions
- `show running-config` differs from `show startup-config`

**Phase:** Phase 1 (write-tool contract design) and Phase 4 (config write tools).

---

### Pitfall C4: Auto-retry on POST writes → duplicate / partial state changes

**What goes wrong:** The existing `RetryMiddleware` correctly skips 5xx retries on tools tagged `*_write` / `*_write_delete`. A new contributor adds an AOS8 write tool, forgets the tag, and the middleware happily replays a POST that modifies config. The first attempt actually succeeded (the 5xx came from a downstream timeout or proxy), and the retry creates a duplicate object — or worse, partial changes on a multi-step operation.

**Why it happens:** `_is_write_tool()` reads tool tags at call time (`retry.py:134-154`). If the tag is missing or misspelled (`aos8_writes` instead of `aos8_write`), the tool is classified as a read and gets retried.

**Consequences:**
- Duplicate VAP/SSID profiles (many AOS8 endpoints aren't idempotent on POST)
- "Object already exists" errors masking the original successful write
- Partial config across MDs in a Conductor hierarchy if the retry hits a different MD's runtime
- Write Memory called twice — usually safe, but adds load and audit noise

**Prevention:**
1. Centralize tag application: factor a `register_aos8_write_tool()` helper that asserts the tag is present and well-formed.
2. Add a unit test that imports every AOS8 tool and asserts each `@mcp.tool` either has `aos8_read` OR (`aos8_write` / `aos8_write_delete`) — no missing tags, no read-and-write tagged together.
3. Verify the middleware classification by hitting a mock 503 for a write tool in tests and asserting **no** retry occurred.
4. Documentation in the platform README reinforcing that 429 retries on writes ARE intended (always safe — server is rate-limiting) but 5xx are not.

**Phase:** Phase 1 (tagging convention) and ongoing in every write-tool phase.

---

### Pitfall C5: SSL verify=False default → silent MITM exposure

**What goes wrong:** Because self-signed certs are common in AOS8 deployments, the developer flips the default to `verify_ssl=False` "to make it work out of the box." The Conductor is now trusted unconditionally; a network-positioned attacker can intercept admin credentials and tokens.

**Why it happens:** Self-signed certs are genuinely common, and the friction of teaching every operator to install a corporate CA is real. The path of least resistance is to default-disable verification.

**Consequences:**
- UIDARUBA token captured by anyone on-path
- Admin password captured at login
- Indistinguishable from a working production deployment in tests — the failure is silent

**Prevention:**
1. **Default `verify_ssl=True`** (matches PROJECT.md constraint and the Apstra precedent at `client.py:50`).
2. Operator must affirmatively create `secrets/aos8_verify_ssl` containing `false` to disable.
3. **Log a WARNING at startup** when `verify_ssl=False` is detected — CONCERNS.md flags this as currently missing for ClearPass/Apstra, so AOS8 should set the better example. Recommendation: add the warning to the AOS8 module and backfill it for the others in a follow-up.
4. Include the warning text in operator docs ("If you see `AOS8: SSL verification disabled` in startup logs and you didn't intend it, fix it").
5. Provide a separate `aos8_ca_bundle_path` secret for operators who want to pin a corporate CA without disabling verification entirely.

**Phase:** Phase 1 (config + client foundation).

---

### Pitfall C6: Hardcoding port 4343

**What goes wrong:** The base URL is built as `f"https://{host}:4343"` instead of `f"https://{host}:{port}"`. Any operator running the Conductor on a different port (custom NAT, reverse proxy, change-controlled non-default port) gets connection refused with no path to fix it short of editing the source.

**Why it happens:** Port 4343 is the documented default and appears in every example. It feels harmless to hardcode.

**Consequences:**
- Deployment fails for any non-default port without code changes
- Reverse proxy / load balancer setups (8443, 443, etc.) blocked
- Operator confusion: "the port secret is set, why isn't it being used?"

**Prevention:**
1. Mirror `ApstraSecrets` exactly: `host` and `port` as separate fields, `port` defaulting to 4343 when the secret file is absent or empty.
2. Build base URL as `f"https://{config.host}:{config.port}"` — never literal `4343`.
3. Validate port is in `1..65535` at config load.
4. Add a unit test that constructs the client with a non-default port and asserts the base URL is correct.

**Phase:** Phase 1 (config + client).

---

### Pitfall C7: Wrong `config_path` scope → unintended global change

**What goes wrong:** A tool defaults `config_path` to `/md` (the root of the managed-device hierarchy). An operator asks the LLM to "disable that test SSID on building-3-controller." The LLM calls the write tool with the SSID name but **doesn't override `config_path`**. The change is applied at `/md`, propagating to **every** managed device, not just building-3.

**Why it happens:**
- `/md` is the Conductor-wide scope; `/md/<device-or-group>` is targeted.
- The tool signature has `config_path: str = "/md"` as a default, which is correct for reads but dangerous for writes.
- LLMs default to omitting optional params.

**Consequences:**
- Production-wide change instead of targeted change
- "I only meant to fix one site" incidents
- Rollback requires identifying every MD that received the propagation
- Worse on writes than reads — read at `/md` returns merged view (annoying but recoverable); write at `/md` mutates global config (incident).

**Prevention:**
1. **Reads:** default `config_path` to `/md` (sensible Conductor-wide view).
2. **Writes:** require `config_path` as a non-default parameter — NO default. The LLM must supply it explicitly.
3. Validate `config_path` against a known shape (`/md`, `/md/<name>`, `/mm`, `/mm/mynode`); reject empty/`/` to prevent accidental root writes.
4. Tool description must explain scope-of-effect: "Setting config_path=/md applies to ALL managed devices. Use /md/<device-name> for a single device."
5. Elicitation middleware confirmation message must include the resolved `config_path` and a plain-English scope hint: "This will change config on **all 47 managed devices** under /md. Confirm?"
6. Consider a separate `aos8_list_config_paths` read tool that enumerates valid paths so the LLM can resolve operator intent ("building-3-controller") to a path before writing.

**Phase:** Phase 1 (write-tool design contract) and Phase 4 (config writes).

---

### Pitfall C8: Conductor vs direct-MD writes diverge

**What goes wrong:** Some tools point at the Conductor (`config_path=/md/...`) and the change propagates correctly. Another tool, written by a different contributor, points directly at the MD's IP/hostname (bypassing the Conductor). The MD now has runtime config that doesn't match what the Conductor thinks it pushed; on next Conductor sync, the local change is overwritten — or worse, an inconsistency stays latent.

**Why it happens:** Both forms work in isolation. The AOS8 docs even show direct-MD examples for standalone scenarios. Mixing them in a Conductor-managed deployment is the silent killer.

**Consequences:**
- Config divergence between Conductor's view and MD reality
- Conductor sync overwrites operator changes
- Audit trail split across two systems
- Hard to diagnose: GUI shows one config, CLI shows another

**Prevention:**
1. **One client = one target.** Operator's secrets point at either a Conductor OR a standalone — not both. Document this explicitly.
2. At startup, query `show switches` (or equivalent) to determine if the host is acting as a Conductor or a standalone — log it, store the role on the client.
3. If role is `Conductor`: refuse direct-MD config_paths in writes (only allow `/md/...` hierarchy).
4. If role is `standalone`: allow `/mm` paths only.
5. Tool docs reinforce: "Always write through the Conductor in Conductor-managed deployments."

**Phase:** Phase 1 (role detection) and Phase 4 (write tools).

---

## Moderate Pitfalls

### Pitfall M1: showcommand `_meta` field treated as data → LLM confusion

**What goes wrong:** `/v1/configuration/showcommand` responses include a top-level `_meta` field describing the schema/available fields. A tool returns the raw response to the LLM, which then treats `_meta` as substantive data ("the device has a meta property of...") or worse, includes its keys in summaries.

**Why it happens:** No selective field stripping — the response is forwarded as-is. Some commands return only `_meta` and a single data field; others return rich structures.

**Prevention:**
1. Strip `_meta` from showcommand responses before returning to the LLM (or move it to a sibling `_schema` key the LLM is told to ignore).
2. If `_meta` is useful for tool implementers (it lists available fields), expose it through a separate `aos8_show_command_schema` tool, not on the main response.
3. Document the response shape in tool docstrings.

**Phase:** Phase 2 (show command passthrough).

---

### Pitfall M2: showcommand returns unstructured text for some commands

**What goes wrong:** A tool assumes every showcommand returns parseable JSON. Some commands (especially older `show ap ...` variants) return a `_data` array of free-text lines. The tool's downstream parser (e.g., `[item["name"] for item in resp["items"]]`) crashes or returns garbage.

**Prevention:**
1. Whitelist: maintain a curated set of show commands the tool exposes, each with a known response shape and a parser tested against captured fixtures.
2. For an "advanced/passthrough" tool, document the response is unstructured and return `{"raw": ..., "structured": null}` shape.
3. Fixture-based tests for every supported show command using captured real responses (mocked, no live system per PROJECT.md constraint).

**Phase:** Phase 2 (read tools, show command surface).

---

### Pitfall M3: "Delete via POST with delete flag" implemented inconsistently

**What goes wrong:** AOS8 has no DELETE verb. Object deletion is a POST to the same endpoint with `_action: "delete"` (or similar — varies by object). Tools end up with a mix of:
- `aos8_delete_xyz` that POSTs `{"_action": "delete", ...}`
- Other tools that POST a "tombstone" object
- Some that try `DELETE` and fail

Tag confusion follows: should it be `aos8_write` or `aos8_write_delete`?

**Prevention:**
1. Centralize delete in the client: `client.delete_object(object_name, payload, config_path)` that does the correct POST + flag for each object type. Maintain a per-object dispatch table.
2. **All deletes tagged `aos8_write_delete`** — surfaces as the "destructive" set in the elicitation message and to enable separate enable-flags down the line.
3. Tool naming: `aos8_delete_<object>` exclusively for delete-flag POSTs. Never reuse a `set` tool path with a flag parameter.
4. Document the deletion pattern in the platform README.

**Phase:** Phase 4 (write/delete tools).

---

### Pitfall M4: Concurrent write tools racing on the same config_path

**What goes wrong:** Two parallel tool calls (e.g., two LLM threads, or `asyncio.gather` from a guided prompt) modify the same `config_path` simultaneously. AOS8 doesn't transactionally lock — last-write-wins, with intermediate state observable.

**Prevention:**
1. Per-`config_path` `asyncio.Lock` map on the client (cleaned up periodically). All writes acquire the lock for their `config_path` (and any prefix path being written transitively).
2. Document that the AOS8 client serializes writes per config_path; reads are concurrent.
3. Note in INSTRUCTIONS.md that guided prompts should not parallelize writes.

**Phase:** Phase 1 (client) and Phase 4 (write tools).

---

### Pitfall M5: Login error responses trigger 5xx retry middleware loops

**What goes wrong:** If `POST /v1/api/login` itself returns 503 (Conductor under load at startup), the retry middleware sees a 5xx and retries. The login is a "read-shape" tool from the middleware's perspective (no `_write` tag because login isn't a tool — it's an internal client call). But the client's own retry of login under the lock can compound: the first call's lock holder retries, the second waiter eventually wakes and also retries.

**Prevention:**
1. Login happens **inside** `AOS8Client._login_locked()`, not via a FastMCP tool — the retry middleware never sees it.
2. Inside `_login_locked()`, implement a small explicit retry (2-3 attempts with backoff) for connection errors and 503 only. Do NOT retry on 401/403 (credential failure) — those should fail fast.
3. Fail fast on 401: raise `AOS8AuthError` so the lifespan startup fails loudly rather than spinning at boot.

**Phase:** Phase 1 (client).

---

### Pitfall M6: Health probe at startup blocks lifespan

**What goes wrong:** Following the GreenLake pattern (CONCERNS.md "TokenManager calls `_generate_new_token()` synchronously in `__init__`"), the AOS8 client logs in synchronously in its constructor or in lifespan startup with no timeout. If the Conductor is unreachable or slow, the entire MCP server fails to start — blocking all six other platforms.

**Prevention:**
1. Construct the client without logging in. Defer login to first use (lazy, behind the lock — matches Apstra).
2. Lifespan startup: kick off an `aos8_client.health_check()` with an explicit short timeout (5-10s). On timeout, **log a warning and continue** — disable AOS8 tools dynamically rather than failing the whole server.
3. Mirror `apstra/client.py:health_check()` (which calls `_ensure_token()` without other work).
4. Add config knob `AOS8_STARTUP_TIMEOUT` (default 10s).

**Phase:** Phase 1 (lifespan integration).

---

### Pitfall M7: Auto-disable behavior diverges from existing platforms

**What goes wrong:** The platform module defines its enablement check differently from the existing six. Operators who removed AOS8 secrets to disable the platform find that AOS8 tools still register (and fail at call-time with a confusing error), or vice versa.

**Prevention:**
1. Follow the established pattern: `_read_secret("aos8_host")`, `_read_secret("aos8_username")`, `_read_secret("aos8_password")` — if any required secret is empty/absent, return `None` from the config loader and skip `register_tools()` in `server.py`.
2. Mirror Apstra's `ApstraSecrets` Pydantic model exactly.
3. Add a config-loading unit test that asserts: missing host → AOS8 disabled, all tools absent.

**Phase:** Phase 1 (config).

---

### Pitfall M8: `0.0.0.0` host default extends to AOS8 too

**What goes wrong:** Not AOS8-specific but worth flagging — CONCERNS.md notes the server binds to `0.0.0.0` by default. Adding AOS8 with write tools enabled and `ENABLE_AOS8_WRITE_TOOLS=true` running on a developer laptop without a firewall exposes Conductor admin to the local network.

**Prevention:**
1. Document in the AOS8 README that enabling write tools should be paired with non-`0.0.0.0` binding or network isolation.
2. Optionally log a higher-severity warning when (`ENABLE_AOS8_WRITE_TOOLS=true` AND `MCP_HOST=0.0.0.0` AND not running in Docker).
3. This is a server-wide concern; flag it but don't try to solve it solely in the AOS8 module.

**Phase:** Phase 4 (write tools).

---

## Minor Pitfalls

### Pitfall m1: `httpx` cookie jar vs explicit header

**What goes wrong:** Some AOS8 endpoints accept the token in a cookie (`UIDARUBA=<token>`); some require it as a query param (`?UIDARUBA=...` for showcommand). Using only one style causes 401s on the other endpoint family.

**Prevention:** Client supports both — store the token, attach the cookie via `httpx.AsyncClient` cookie jar, AND add `UIDARUBA` to query params on showcommand calls. Test both code paths.

**Phase:** Phase 1.

---

### Pitfall m2: Tool count drift in README

**What goes wrong:** CONCERNS.md highlights several "documentation drift" findings. Adding AOS8 without updating the comparison table in README.md, INSTRUCTIONS.md tool categories, and `docs/TOOLS.md` will produce yet another stale-doc PR-review cycle.

**Prevention:** Follow the Documentation Checklist in CLAUDE.md religiously — update README, CHANGELOG, INSTRUCTIONS, docs/TOOLS, pyproject.toml version in the same PR.

**Phase:** Every phase that adds tools.

---

### Pitfall m3: `_template` platform's `mcp = None` antipattern reused as-is

**What goes wrong:** Copying from `platforms/_template/_registry.py` carries over the `mcp: FastMCP = None` with `type: ignore` (CONCERNS.md technical debt). Tool code that imports `mcp` before `register_tools()` is called gets `None` and crashes at decoration time.

**Prevention:** Follow the pattern but understand the ordering: `server.py` MUST call `register_tools(mcp, config)` before any tool module is imported. `register_tools` imports tool modules via `importlib` — adding direct top-level imports anywhere else will break. Document this in the platform's `__init__.py` docstring.

**Phase:** Phase 1.

---

### Pitfall m4: `assert` for None-narrowing replicated in AOS8 client

**What goes wrong:** The Apstra client uses `assert self._token is not None` after the lock. CONCERNS.md flags `assert` use as a Python `-O` hazard. Copying the pattern propagates the issue.

**Prevention:** Replace asserts with explicit `if self._token is None: raise AOS8AuthError(...)` in the AOS8 client. Slightly more code, no `-O` footgun, type checker still satisfied.

**Phase:** Phase 1.

---

### Pitfall m5: 401 retry loop on truly bad credentials

**What goes wrong:** `request()` refreshes the token once on 401 (Apstra pattern). If credentials are actually wrong, the refresh succeeds at re-login (or hits 401 on login), and the second request hits 401 again — but the Apstra implementation doesn't loop. AOS8 tokens can also be invalidated by a Conductor admin manually; the refresh-once pattern handles this. Issue arises if you generalize to "refresh on every 401" — that becomes infinite under bad creds.

**Prevention:** Refresh exactly once per request (mirror Apstra). After the second 401, raise. Test both: (a) token expiry → refresh-once succeeds; (b) bad password → refresh fails fast at login, raised as `AOS8AuthError`.

**Phase:** Phase 1.

---

### Pitfall m6: Docker secrets file missing trailing newline

**What goes wrong:** Operators creating `secrets/aos8_password` with a text editor get a trailing newline appended; `_read_secret` returns the value with `\n`. The login POST sends the wrong password.

**Prevention:** `_read_secret()` must `.strip()` the result (existing helper likely already does — verify and follow the same pattern).

**Phase:** Phase 1.

---

### Pitfall m7: AOS8 returns 200 OK with embedded error

**What goes wrong:** Some AOS8 endpoints return HTTP 200 with a JSON body like `{"_global_result": {"status": 1, "status_str": "Some error"}}`. `response.raise_for_status()` is happy; the tool returns "success" with garbage data.

**Prevention:** After `raise_for_status()`, parse the body and check `_global_result.status` — if non-zero, raise an `AOS8APIError` with `status_str`. Apply uniformly in `client.request()`. Add unit test fixtures for both shapes.

**Phase:** Phase 1.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Phase 1 — Client foundation | C1 (re-login), C2 (token leak), C5 (SSL default), C6 (port hardcode), M5/M6 (login retry / startup), m1 (cookie vs query), m4 (asserts), m7 (200-with-error) | Build the client by mirroring `ApstraClient` line-by-line, then layer AOS8 specifics. Code review checklist: token reuse, redaction, verify_ssl default, configurable port, lazy login, both auth surfaces handled, `_global_result` checked. |
| Phase 1 — Config / Secrets | M7 (auto-disable), m6 (trailing newline) | Mirror Apstra `_read_secret()` chain; unit test missing-secret → platform disabled. |
| Phase 2 — Read tools | M1 (`_meta`), M2 (unstructured showcommand) | Strip `_meta`; whitelist commands with known shapes; capture fixtures from a real Conductor for tests. |
| Phase 3 — Health / inventory | M6 (lifespan blocking), m2 (doc drift) | Health check with timeout; update README counts. |
| Phase 4 — Write / config tools | C3 (write_memory), C4 (retry on writes), C7 (config_path scope), C8 (Conductor vs MD), M3 (delete via POST), M4 (concurrent writes), M8 (0.0.0.0 + writes) | `aos8_write_memory` as explicit tool; per-tool tag tests; `config_path` no-default on writes; role detection at startup; per-path locks; centralized delete dispatch. |
| Phase 5 — Dynamic mode / meta-tools | Same dynamic-mode gotchas as other platforms (no AOS8-specific issue here) | Follow Apstra's dynamic-mode pattern. |
| Phase 6 — Cross-platform tools (site_health_check etc.) | None AOS8-specific; existing fan-out / rate-limit concerns from CONCERNS.md apply | Not in scope for the initial AOS8 module per PROJECT.md. |

---

## Sources

- **AOS8 API Guide** (Aruba documentation, multiple versions 8.6+) — authoritative for endpoint behavior, UIDARUBA semantics, write_memory, showcommand `_meta`, GET/POST-only constraint. Confidence: HIGH.
- **Existing codebase: `platforms/apstra/client.py`** — proven pattern for token caching, lock, refresh-on-401, verify_ssl honored. Direct evidence of what to mirror. Confidence: HIGH.
- **Existing codebase: `middleware/retry.py`** — confirms 5xx-on-writes is intentionally skipped via `*_write` tag detection; clarifies the contract new tools must follow. Confidence: HIGH.
- **`.planning/codebase/CONCERNS.md`** — surfaces existing technical debt to avoid replicating: SSL warning gap, `assert` for None-narrowing, GreenLake synchronous init, host=`0.0.0.0`, doc drift. Confidence: HIGH.
- **`.planning/PROJECT.md`** — explicit constraints: GET/POST only, UIDARUBA reuse, write_memory required, verify_ssl default behavior, config_path scope. Confidence: HIGH.
- Community discussions / GitHub issues on `aruba-cli` and `pyaoscx`-style libraries (general ecosystem awareness, no specific URL load-bearing) — Confidence: MEDIUM. Used to corroborate the "session exhaustion on repeated login" warning.
