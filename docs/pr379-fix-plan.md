# PR #379 Fix Plan — `greenlake_bulk_add_devices`

> Remediation plan for the **Changes requested** review on
> [nowireless4u/hpe-networking-mcp#379](https://github.com/nowireless4u/hpe-networking-mcp/pull/379).
> Generated as a planning artifact — execute via the GSD workflow.

- **Target branch:** `feat/greenlake-bulk-add-devices` (head of PR #379 → `nowireless4u:main`)
- **Key context:** GreenLake is currently read-only. This is the **first** GreenLake
  write tool, so the *entire* write-gating scaffold must be added for the platform,
  not just the tool. Mirror the existing pattern used by mist/central/clearpass/apstra/axis/aos8.

---

## Blocking 1 — Gate the write tool (missing flag + missing elicitation)

1. **`src/hpe_networking_mcp/config.py`**
   - Add field `enable_greenlake_write_tools: bool = False` (next to line ~118).
   - Add env parse: `enable_greenlake_write = os.getenv("ENABLE_GREENLAKE_WRITE_TOOLS", "false").lower() in _truthy` (~line 457).
   - Pass `enable_greenlake_write_tools=enable_greenlake_write` into the config constructor (~line 525).
2. **`src/hpe_networking_mcp/middleware/elicitation.py`** (`ElicitationMiddleware`, ~lines 44–82)
   - Add `greenlake_write = config.enable_greenlake_write_tools`.
   - Include it in the `any_write = …` OR-chain (line ~50).
   - Add `if greenlake_write: await ctx.enable_components(tags={"greenlake_write"}, components={"tool"})`.
3. **`src/hpe_networking_mcp/server.py`** (~line 341)
   - Add:
     ```python
     if not config.enable_greenlake_write_tools:
         mcp.add_transform(Visibility(False, tags={"greenlake_write"}, components={"tool"}))
     ```
4. **`src/hpe_networking_mcp/platforms/greenlake/tools/bulk_add.py`**
   - Register the tool with the write tag + annotations (mirror `aos8/tools/writes.py:30,86`):
     `WRITE = ToolAnnotations(readOnlyHint=False, destructiveHint=False, idempotentHint=False, openWorldHint=True)`
     and tag it `tags={"greenlake_write"}`. Match the GreenLake platform's registration
     mechanism (check `greenlake/tools/__init__.py` vs. a `@tool` decorator).
   - **Replace the self-attestable `acknowledged` bool** (lines ~158–199) with a real
     human-in-the-loop call:
     ```python
     from hpe_networking_mcp.middleware.elicitation import confirm_write
     decision = await confirm_write(ctx, message=f"The LLM wants to bulk-add {n} devices to GreenLake. Do you accept?")
     # bail on decline
     ```
     Drop the `acknowledged` parameter entirely — it is the exact item the reviewer
     flagged as "self-attestable by the AI (no human-in-the-loop)".

## Blocking 2 — Error contract (`ToolError`, not `"Error:"` strings)

The envelope middleware (`response_envelope.py`) wraps a returned string as `data` with
`ok: True`, so error strings read as success.

- In `bulk_add.py`, replace every `return "Error: …"` / `return f"Error: {…}"`
  (lines ~210, 212, 216, 221, 222) with `raise ToolError("…")`.
  Audit the helpers too: `_bulk_assignment.py`, `_bulk_enrichment.py`, `utils/csv_parser.py`.
- `from fastmcp.exceptions import ToolError`.
- Keep **per-row** failures inside the success envelope (legitimate partial-success data);
  only input/precondition failures (no CSV, both args supplied, parse failure, zero valid
  rows) become `ToolError`.

## Blocking 3 — Documentation in the same PR

- **README.md** — flip the GreenLake **Configuration Write (CRUD)** matrix cell, add
  `ENABLE_GREENLAKE_WRITE_TOOLS` to the env-var table, bump the server-wide tool count.
- **CHANGELOG.md** — new version entry (see Blocking 4).
- **INSTRUCTIONS.md** — add the tool + its write-gating note.
- **docs/TOOLS.md** — add `greenlake_bulk_add_devices` to the GreenLake section + counts.

## Blocking 4 — Version bump

`pyproject.toml` is at `3.1.0.4`. Bump per project workflow — a new write tool + new env
flag is more than a patch; likely **`3.2.0.0`** following the UXI precedent (#348), but match
the maintainer's current numbering on `main`. Add the matching CHANGELOG entry and release tag.

---

## Moderate concerns

- **542 → <500 lines:** `bulk_add.py` already has sibling helpers
  (`_bulk_assignment.py`, `_bulk_enrichment.py`). Extract the CSV/return-envelope assembly
  or the polling loop into a `_bulk_core.py` (or into `utils/csv_parser.py`) to get under 500.
- **Double envelope:** remove the manual `"ok": True` dict (line ~522) and return the payload
  `dict` directly — let `response_envelope.py` wrap it. Confirm the tool name is **not** in
  `_NO_ENVELOPE_TOOLS`.
- **`AsyncLimiter` singleton:** the module-level `_device_add_limiter = AsyncLimiter(...)`
  (line ~34) binds to the import-time event loop and warns/breaks across loops. Instantiate
  the limiter **inside** the tool coroutine (or lazily per-call); same for the assignment limiter.
- **Unverified assumptions:** reviewer wants captured live API responses for the location-PATCH
  body shape and async-operation response shapes. Requires **real GreenLake hardware** —
  likely cannot be closed in a code-only pass; flag it explicitly.

## Validation gate (maintainer expects all green)

```
ruff check .
ruff format --check .
mypy
bandit -r src/ -c pyproject.toml
pytest tests/ -q
```

Update affected unit tests: the `acknowledged` → `confirm_write` swap and the `ToolError`
changes will break tests asserting on the old `"Error:"` strings / the `acknowledged` param.

---

## Execution notes

1. The **live-response** moderate item probably cannot be fully closed without hardware;
   everything else is pure code/docs.
2. Fixes must land on **`feat/greenlake-bulk-add-devices`** to update PR #379.
3. The upstream repo (`nowireless4u/hpe-networking-mcp`) is read-only from the web session's
   GitHub scope — review replies must be posted from a session with upstream access.
