# Roadmap — Setup & provider configuration as in-webview modal forms

Goal: run the first-time setup wizard and provider-key configuration inside the
graph webview as modal forms (same React components everywhere), instead of
VS Code native UI (`showQuickPick` / `showInputBox`). Applies to both the VS Code
extension and the Electron studio-app.

## Where things stand today

| Concern | VS Code extension | Electron app |
|---|---|---|
| Setup wizard (mode, native/container) | `setup-wizard.ts` — QuickPick/InputBox chain, saves to settings + `globalState` | none — only a `setStudioMode` sidebar toggle |
| Provider keys | `provider-keys.ts` — SecretStorage, QuickPick per provider, synced via `client.syncProviderKeys()` | none — relies on host env vars only |
| Modal UI patterns | shared `webview-src/components/*Modal.tsx` (Properties, Cron, Events, Resources, Confirm) | same shared components |

The shared webview already has the modal idiom, the zustand store, and the
`ToWebview`/`FromWebview` message protocol — the work is mostly moving the two
flows onto that rail.

## Phase 1 — Shared UI and protocol (clients/shared)

1. **`SetupModal.tsx`** — steps as a single form: Studio mode (local/remote),
   run mode (native CLI / Docker container), remote URL when applicable.
   Follow `PropertiesModal.tsx` structure and styling.
2. **`ProviderKeysModal.tsx`** — one row per provider (Anthropic, OpenAI,
   Gemini, GitHub token): masked status ("set from env", "set", "not set"),
   password input to replace, clear button, optional "Test" action.
3. **Protocol additions** in `shared/types/index.ts`:
   - `ToWebview`: `setupState { mode, runMode, remoteUrl, keys: Record<provider, 'env'|'stored'|'none'> }`, `openSetup`, `openProviderKeys`.
   - `FromWebview`: `requestSetupState`, `saveSetup {…}`, `saveProviderKey { provider, key }`, `clearProviderKey { provider }`.
   - Never send key material *to* the webview — only presence/masked status.
4. **Store slice** for setup state + modal visibility; keep key values only in
   local component state, cleared on close.

## Phase 2 — VS Code extension host

1. Handle the new messages in `actors-panel.ts` (or a small dedicated
   `settings-host.ts` shared by panels):
   - `saveProviderKey` → SecretStorage (`saveProviderKeys`) → `client.syncProviderKeys()` → push refreshed `setupState`.
   - `saveSetup` → `canticaScores.*` settings + `SETUP_DONE_KEY` in `globalState`; restart local studio when mode/run-mode changed (reuse `StudioManager`).
2. Repoint entry commands: `canticaScores.runSetupWizard` and the configure-keys
   command open (or reveal) the panel and post `openSetup` / `openProviderKeys`.
   Keep command IDs stable so menus/keybindings don't break.
3. First-run: if `!isSetupDone`, open the panel with `openSetup` instead of the
   QuickPick chain.
4. Delete the QuickPick/InputBox flows in `setup-wizard.ts` once the modal path
   is verified (keep `isSetupDone`/`resetSetup`/`publishSetupContext`).

## Phase 3 — Electron app

1. Main-process handlers mirroring Phase 2, with storage the app currently
   lacks: encrypt keys with Electron `safeStorage`, persist in a JSON file
   under `app.getPath('userData')`.
2. On startup and after each save, call `client.syncProviderKeys()` (the app
   never does this today — Copilot/Claude actors in container mode only work
   if host env happens to have the vars).
3. Pass stored keys to `StudioManager.start()` (it already accepts a
   `ProviderApiKeys` argument) so container restarts get the env vars.
4. First-run: missing config file → renderer opens `SetupModal` automatically.

## Phase 4 — Hardening and cleanup

1. Vitest coverage: modal components (render, validation, mask/clear), store
   slice, message round-trips.
2. Manual pass per the deployment checklist (both clients × native/container).
3. Remove dead native-UI code paths; update README/screenshots.

## Notes / risks

- **Secrets in the webview**: inputs are `type=password`, values live only in
  component state and are posted once; the host never echoes keys back. CSP
  already blocks external requests from the webview.
- **Sync semantics**: `syncProviderKeys` seeds the backend DB used by
  `_token_for_provider()`; saving a key while actors run won't retrofit
  already-started actors — surface a hint to restart affected actors.
- **Ordering**: Phase 1+2 ship together (extension parity), Phase 3 can follow
  independently; the app gains functionality it never had rather than losing a
  native flow.
