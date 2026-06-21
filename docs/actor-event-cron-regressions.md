# Actor Event & Cron Regression Reference

**Purpose:** Document every known regression in the Studio API's actor/event/cron system, why each happened, and what invariants must hold to prevent recurrence. Consult this before touching `runtime.py`, `actor.py`, `mcp_server.py`, or any actor lifecycle code.

---

## System architecture (brief)

```
ActorRuntime
  ├── _actors: dict[name, _ActorState]        # running actors (pykka refs)
  ├── _scheduler: BackgroundScheduler          # APScheduler, one per process, starts in __init__
  ├── _forwarded_log: list[dict]               # cron/event output drain for the frontend
  └── _forwarded_log_lock: threading.Lock
```

**Lifecycle methods:**

| Method | When called | Does what |
|---|---|---|
| `stop(name)` | `DELETE /actors/{name}` | Stops one actor. Never touches scheduler. |
| `stop_all()` | HTTP `DELETE /actors`, process exit | Stops all actors. In `studio-api/`: also shuts down scheduler — safe at process exit only. |
| `shutdown()` (api/ only) | Process exit lifespan only | Calls `stop_all()` then shuts down scheduler. In `studio-api/` this distinction is encoded by only calling `stop_all()` from the lifespan handler. |

**Event/cron flow:**

```
Cron fires (_run closure) → proxy.instruct(prompt) → actor LLM response
    → if targetActor/targetEvent: fire_event(ta, te, output)  OR  instruct(ta, output)
    → output appended to _forwarded_log
    → frontend drains _forwarded_log via /notifications and shows actorOutput
```

---

## Regression 1 — fire_event triggers on actor start

**Commit fixed:** `37bb456` (`studio-api/`)
**Symptom:** Newly started AI actor is unresponsive for 120 s. After timeout the actor appears to start but cron/event output is wrong or missing.

**Root cause:** During `_start_ai_actor()`, the code called `proxy.instruct(system_prompt)` as a "warm-up" to initialise the LLM session. Because `fire_event` is a `@tool`, an LLM whose system prompt contains "call fire_event after every response" would immediately invoke `fire_event` during this warm-up call. `fire_event` then made a nested LLM call that blocked for 120 s (the timeout), making the actor appear hung. The warm-up was also redundant — `system_prompt` is passed as the `system=` argument on every `instruct()` call anyway.

**Fix:** Removed the warm-up `proxy.instruct(system_prompt)` call entirely. `start()` now always returns `None`. Session restore still works (restore_session is called before the first user instruct).

**Invariant to preserve:**
- Never call `instruct()` or `provider.run()` during actor start-up.
- `start()` / `_start_ai_actor()` must return without doing any LLM call.
- If you need session state on start, use `proxy.restore_session(...)`.

---

## Regression 2 — Cron events stop firing after actors are restarted

**Commit fixed:** `c6bbbc5` (`api/` only — `studio-api/` avoids the trigger differently)
**Symptom:** Stop all actors via the UI or `DELETE /actors`, then restart them. Cron jobs never fire again. No error is visible in logs (the `SchedulerNotRunningError` was silently caught).

**Root cause:** `stop_all()` called `self._scheduler.shutdown(wait=False)`. APScheduler's `shutdown()` is **permanent** — the scheduler object is dead and cannot be restarted or accept new jobs. When actors were restarted, `_register_crons()` called `scheduler.add_job()` on a dead scheduler. APScheduler raised `SchedulerNotRunningError`. The exception was caught by the broad `except Exception: pass` in `_register_crons`, so cron jobs were never registered and never fired. The actors appeared to be running normally but were cron-dead.

**Fix (api/ version):**
- Split `stop_all()` from `shutdown()`.
- `stop_all()` stops actors only; it no longer touches the scheduler.
- `shutdown()` does `stop_all()` + `scheduler.shutdown()`. Only the FastAPI lifespan handler calls `shutdown()`.
- Added defence-in-depth guard in `_register_crons`: `if not self._scheduler.running: self._scheduler.start()`.

**How studio-api avoids the trigger:** The HTTP endpoint `DELETE /actors` calls `rt.stop(name)` per actor in a loop, never `rt.stop_all()`. `stop_all()` is only called from the lifespan handler on process exit, where shutting down the scheduler is correct. So the scheduler is never killed mid-session.

**Invariants to preserve:**
- `_scheduler` must stay alive for the entire process lifetime.
- Only shut down the scheduler in the process-exit lifespan handler.
- HTTP endpoints that stop actors must call `stop(name)` per actor, never `stop_all()`.
- Any new code path that stops all actors outside the lifespan handler must NOT call `stop_all()` if it needs crons to re-register afterwards — call `stop(name)` for each actor instead.
- Always add the guard `if not self._scheduler.running: self._scheduler.start()` at the top of `_register_crons` and `_register_code_crons` as defence-in-depth. This is present in `api/` but must be kept whenever those methods are touched in either codebase.

---

## Regression 3 — Cron job output not appearing in webview

**Commit fixed:** `7e4abf3` (`studio-api/`)
**Symptom:** Cron jobs fire (verified via logs) but actor chat panels in the VS Code extension / Electron app never show the output. The frontend appears dead while the backend is working.

**Root cause — backend:** The `_run` closure inside `_register_crons` called `proxy.instruct(p)` and (for forwarding) `self.fire_event(ta, te, output)` / `self.instruct(ta, output)` directly. These code paths bypassed `_forwarded_log`: no output was ever written to the ring buffer. The frontend drains `_forwarded_log` via `/notifications`, so it saw no activity.

**Root cause — frontend:** `pushNotifications()` in `actors-panel.ts` was only called after explicit user interactions (button click, instruct call). There was no background polling, so even if `_forwarded_log` had content, the frontend would never receive it unless the user did something.

**Fix:**
- Backend: After `proxy.instruct(p)` in `_run`, append `{"name": actor_name, "prompt": p, "output": output}` to `_forwarded_log` under `_forwarded_log_lock`. After `fire_event` or `instruct` to a target actor, extend/append forwarded entries the same way.
- Frontend: Added a `setInterval` (3 s) in `actors-panel.ts` that calls `pushNotifications()` while the panel is visible. Cleared in `dispose()`.

**Invariants to preserve:**
- Every code path that calls `instruct()` or `fire_event()` on behalf of the runtime (crons, internal forwarding) must write to `_forwarded_log` under `_forwarded_log_lock`.
- Never call `proxy.instruct()` in a cron or event handler without also logging to `_forwarded_log`.
- `_instruct_actor` (the function injected into AI actors for LLM-triggered forwarding) already writes to `_forwarded_log` — use it for actor-initiated forwards. Direct `proxy.instruct()` calls in runtime code need manual logging.

---

## Regression 4 — fire_event MCP tool returned wrong content to LLM

**Symptom:** An actor whose system prompt says "call fire_event after every response" would call the MCP `fire_event` tool and receive its own event prompt text back as the tool result. The LLM interpreted this as a new instruction, causing a loop or unexpected behaviour.

**Root cause:** The `fire_event` tool in `actor.py` (path: `sendResponse=False`, no target actors) returned `output` which equals `instruction` — the raw event prompt string. The LLM calling `fire_event` as an MCP tool received its own prompt as the tool result and thought it needed to act on it.

**Additionally:** When `runtime.fire_event()` was changed to return `{"output": ..., "forwarded": [...]}` (a dict), the MCP server's `fire_event` tool initially returned `str(dict)` — a raw JSON blob as the tool result. The LLM couldn't interpret this as success and retried.

**Fix:** `mcp_server.py fire_event` extracts only the string: `return _get_rt().fire_event(actor_name, event_name, context)["output"]`. The `actor.py fire_event` returns human-readable confirmation strings ("Event 'X' fired successfully. Forwarded to: Y.") instead of the raw prompt in forwarding cases.

**Invariants to preserve:**
- The MCP `fire_event` tool must always return a plain `str`, never a `dict` or complex type.
- `runtime.fire_event()` returns `dict` (internal). `mcp_server.fire_event` must extract `["output"]` before returning to the LLM.
- `actor.py fire_event` must return a human-readable confirmation (not the instruction text) in the `sendResponse=False` + forwarded case, so the calling LLM understands the tool succeeded.

---

## Regression 5 — Duplicate function-call IDs for Copilot actors with sendResponse events

**Commit fixed:** `4797bdb`
**Symptom:** Copilot-backed actors with `sendResponse=True` events produced HTTP 400 "duplicate function-call ID" errors when the event fired. The event and the outer instruct session corrupted each other.

**Root cause:** `provider.run()` inside `fire_event` (triggered by a tool dispatch that itself runs inside `run_in_executor` inside the SDK session loop) created a second concurrent Copilot CLI session. Two sessions with overlapping call IDs caused the 400 error.

**Fix:** `_start_ai_actor` creates a separate `_event_provider` — a stateless HTTP (non-SDK) clone of the main provider. `actor.py fire_event` uses `self._event_provider` (falling back to `self.provider`). The non-SDK HTTP provider doesn't share session state with the SDK loop.

**Invariants to preserve:**
- Never call `provider.run()` from within a pykka tool dispatch if the provider is SDK-based (Copilot). Use `_event_provider` (HTTP-only) for any nested LLM call that might run concurrently with the outer session.
- When adding a new `@tool` that needs an LLM call, follow the same `_event_provider` pattern.

---

## Quick checklist before touching actor/event/cron code

- [ ] Does anything call `stop_all()` outside the process-exit lifespan? → Don't. Use `stop(name)` per actor.
- [ ] Does the change touch `_register_crons` or `_register_code_crons`? → Ensure the `if not self._scheduler.running: self._scheduler.start()` guard is present.
- [ ] Does `_start_ai_actor` or `start()` do any LLM call? → Remove it. No LLM calls on start.
- [ ] Does a cron `_run` closure call `proxy.instruct()` or `fire_event()`? → It must also write to `_forwarded_log` under `_forwarded_log_lock`.
- [ ] Does a new `@tool` implementation call `provider.run()`? → Use `_event_provider` to avoid concurrent-session corruption.
- [ ] Does the MCP `fire_event` or any MCP tool return a `dict`? → Return a `str` (or `list`). Extract `["output"]` from `runtime.fire_event()`.
- [ ] Adding a new actor lifecycle method that stops actors? → Make sure it doesn't call `stop_all()` or touch the scheduler unless it's a process-exit handler.
