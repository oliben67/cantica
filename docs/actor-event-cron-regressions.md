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

## Regression 6 — Copilot 'auto' model never resolves in the UI

**Commit fixed:** `[studio-api, post-49f4ac4]`
**Symptom:** An AI actor configured with provider=`copilot` and model=`auto` shows a spinner indefinitely in the UI. The model badge never shows the actual model name chosen by Copilot.

**Root cause — missing endpoint:** The client polled `GET /v1/runtime/actors/{name}/model` every 2 s to discover the resolved model. This endpoint did not exist, so every poll 404'd. `fetchResolvedModel` in `studio-client.ts` returns `null` on non-OK responses, so the spinner persisted silently.

**Root cause — missing instruct response field:** `instruct_actor` never included `resolved_model` in its response. The `instructActor` client reads `data.resolved_model` from the response, but the server never set it.

**Root cause — `_ActorState` didn't track provider:** `get_resolved_model()` needed a reference to the provider object (where `resolved_model` is set as an instance attribute after `_run_sdk` completes). `_ActorState` had no `provider` field, so there was nowhere to read it.

**Root cause — unhandled `KeyError` in `instruct_actor`:** After adding `get_resolved_model(name)` to `instruct_actor`, the call was placed OUTSIDE the `try/except` block. If the actor stopped between the `instruct` call and the `get_resolved_model` call (race), FastAPI would return 500 — breaking instruction responses entirely.

**Fix:**
- Added `provider: Any | None = None` field to `_ActorState`; `_start_ai_actor` stores `main_provider` there.
- Added `get_resolved_model(name)` method to `ActorRuntime` that reads `getattr(state.provider, "resolved_model", None)`.
- Added `GET /v1/runtime/actors/{name}/model` endpoint; returns `{"name": name, "resolved_model": <str|null>}`.
- `instruct_actor` now reads `get_resolved_model` and includes it in the response when set.
- Wrapped the `get_resolved_model` call in `instruct_actor` in its own `try/except Exception: pass` — resolution failure is non-fatal; the poll endpoint remains as the fallback.

**How resolution actually works:**
1. Actor starts → client calls `pollResolvedModel(name)` (polls `/model` every 2 s for up to 60 s).
2. User sends first instruction → Copilot SDK runs `_run_sdk` → SDK fires `AssistantUsageData` event with `model` field → `captured_model` is set → `provider.resolved_model = captured_model`.
3. `instruct_actor` reads `resolved_model` from the provider and includes it in the response — the webview shows the model immediately.
4. Concurrently, the next poll also finds `resolved_model` set and posts `actorModelResolved`.

**Invariants to preserve:**
- `_ActorState.provider` must always be set to `main_provider` for AI actors (and `None` for code actors).
- Any code path that calls `get_resolved_model()` from an endpoint should wrap it in `try/except` — it's always non-fatal (model badge can update from polling).
- Do NOT call `probe_resolved_model()` at actor startup in a background thread — it opens a concurrent Copilot CLI session alongside the actor's main SDK session, risking duplicate function-call ID errors (see Regression 5).
- The `pollResolvedModel` window is 30 × 2 s = 60 s. The model WILL appear in the `instruct_actor` response after the first successful instruction even if the poll window has closed.

---

## Regression 7 — Server-down leaves actors showing as running with prompting enabled

**Commit fixed:** `[pending]`
**Symptom:** The studio-api process crashes, is stopped, or the container is restarted. All songbook panels still display actors with their previous running state (green badges, active chat panels) and the instruction input remains enabled. Typing a prompt and submitting it either hangs or returns a network error with no visible feedback. The user has no indication the server is down and cannot tell whether their prompt was delivered.

**Root cause:** The extension holds actor state in memory (the actors-panel webview) that is populated when the server is reachable. There is no liveness feedback loop: the 3-second notification poll (`/v1/runtime/notifications`) silently swallows connection errors and does not propagate server-down status to the webview. The webview receives no `actorStatus` message telling it actors are stopped, so it continues rendering them as running. The instruct input is gated only on whether an actor appears running in the local webview state — not on whether the server is actually reachable.

**Fix (required):**
- When any health poll or notification drain returns a network error (connection refused, timeout), emit `{ type: 'studioStatus', health: 'down', url, version: undefined }` immediately — do not wait for the next scheduled health check.
- In the webview, a `studioStatus` with `health: 'down'` must set ALL actors to a stopped/unreachable visual state and disable the prompt input for every actor in every open songbook panel. This state must override any cached running state.
- When the server comes back up (subsequent health check returns `healthy`), re-fetch the actor list and restore the correct running state before re-enabling inputs.
- The notification poll error path must not be silently swallowed — surface the down state immediately rather than waiting for the health poll interval.

**Invariants to preserve:**
- The webview's actor running/stopped state must always be derived from confirmed server state, not from stale cached state.
- Any network error on `/notifications`, `/health`, or any instruct call must trigger an immediate server-down transition in the UI — not just a silent retry.
- Prompting (the instruction textarea and send button) must be disabled whenever `studioStatus.health !== 'healthy'`, regardless of what the local actor state cache says.
- When the server recovers, re-query `/v1/runtime/actors/summary` before re-enabling inputs — do not assume all previously running actors are still running.

---

## Regression 8 — `task build:image` produced the wrong image tag, silently leaving the old container running

**Commit fixed:** `[studio-api/Taskfile.yml, 2026-06-24]`
**Symptom:** After running `task build:image` + `task down` + `task up`, the container continues to exhibit bugs that were supposedly fixed in the new image. Code changes to the studio-api runtime, wheel, or Dockerfile appear to have no effect.

**Root cause:** `task build:image` built the image tagged `cantica-studio-api:latest`. The `docker-compose.yml` declares `image: studio-api:latest`. These are two different image names. `docker compose up -d` (`task up`) starts from `studio-api:latest` — the old image. The freshly built `cantica-studio-api:latest` is never used by the compose stack. Every rebuild silently produced the wrong tag, and the running container was never updated.

**Fix:** Changed `task build:image` to tag the image as `studio-api:latest` (matching the compose `image:` field). Now `task build:image` + `task up` correctly updates the running container.

**Invariants to preserve:**

- The image tag in `task build:image` (`-t <name>`) must exactly match the `image:` field in `docker-compose.yml`. Keep them in sync whenever either file is changed.
- After any Dockerfile or wheel change, always verify the new image is actually running: `docker inspect studio-api --format='{{.Created}}'` should show a time after your build.
- If the compose stack has a `build:` section, `docker compose up --build` also rebuilds correctly and is the safest way to guarantee the latest image is used.

---

## Regression 9 — Copilot actor start fails with 500 due to unguarded keyring call and no GITHUB_TOKEN in container

**Commit fixed:** `[studio-api/docker-compose.yml + actor_ai wheel rebuild, 2026-06-24]`
**Symptom:** Starting any actor with `provider=copilot` returns HTTP 500: `{"detail":"No recommended backend was available. Install a recommended 3rd party backend package; or, install the keyrings.alt package if you want to use the non-recommended backends."}`. No actor is created. No traceback appears in container logs (FastAPI catches and serialises the exception as the 500 detail).

**Root cause — missing GITHUB_TOKEN:** The container has no `GITHUB_TOKEN` environment variable and no OS keyring backend (headless Linux, no D-Bus/secretstorage). `_resolve_github_token(None)` exhausts all four sources (explicit key → env var → `gh auth token` → keyring) and reaches `keyring.get_password()`. The `keyring` package raises `NoKeyringError` when no backend is installed.

**Root cause — unguarded keyring call in installed wheel:** The installed `actor_ai` wheel at the time had `keyring.get_password()` called OUTSIDE a `try/except` block at the bottom of `_token_from_keyring()`. The `except Exception:` wrapper was present in the source and the rebuilt wheel, but the container had been built from an OLDER wheel (via Docker's build cache, compounded by Regression 8 so the image was never actually updated). The unguarded call let `NoKeyringError` propagate all the way through `_make_provider` → `_start_ai_actor` → `start_actor` endpoint → `raise HTTPException(status_code=500, detail=str(exc))`.

**Fix:**

1. Added `GITHUB_TOKEN: "${GITHUB_TOKEN:-}"` to `docker-compose.yml` so the host's GitHub token is forwarded into the container. `_resolve_github_token` returns the token from the env var before reaching the keyring — the keyring is never called.
2. Rebuilt the image with `--no-cache` so the updated wheel (with `except Exception:` around `keyring.get_password()`) is installed.

**Invariants to preserve:**

- `GITHUB_TOKEN` must always be available in the container for Copilot actors to authenticate. Keep `GITHUB_TOKEN: "${GITHUB_TOKEN:-}"` in `docker-compose.yml` and ensure the host env var is set before running `docker compose up`.
- `_token_from_keyring()` in `actor_ai/providers/openai.py` must wrap `keyring.get_password()` in `try/except Exception: pass`. Any edit to that function must preserve this guard — the keyring backend is never guaranteed to be present.
- When rebuilding the actor-ai wheel (or any dependency wheel), rebuild the Docker image with `--no-cache` or `docker compose up --build` to ensure the new wheel is installed. Docker build cache can preserve old wheel installs even if the wheel file on disk has changed.

---

## Quick checklist before touching actor/event/cron code

- [ ] Does anything call `stop_all()` outside the process-exit lifespan? → Don't. Use `stop(name)` per actor.
- [ ] Does the change touch `_register_crons` or `_register_code_crons`? → Ensure the `if not self._scheduler.running: self._scheduler.start()` guard is present.
- [ ] Does `_start_ai_actor` or `start()` do any LLM call? → Remove it. No LLM calls on start.
- [ ] Does a cron `_run` closure call `proxy.instruct()` or `fire_event()`? → It must also write to `_forwarded_log` under `_forwarded_log_lock`.
- [ ] Does a new `@tool` implementation call `provider.run()`? → Use `_event_provider` to avoid concurrent-session corruption.
- [ ] Does the MCP `fire_event` or any MCP tool return a `dict`? → Return a `str` (or `list`). Extract `["output"]` from `runtime.fire_event()`.
- [ ] Adding a new actor lifecycle method that stops actors? → Make sure it doesn't call `stop_all()` or touch the scheduler unless it's a process-exit handler.
