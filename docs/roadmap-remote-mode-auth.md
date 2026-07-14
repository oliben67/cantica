# Roadmap — Remote-Mode Multi-User Registration & Authentication

Implementation plan for [remote-mode-registration-auth.md](remote-mode-registration-auth.md).
Grounded in the current code: studio-api's RBAC (`orm/models.py`, `auth/deps.py`,
`api/v1/{auth,users}.py`), its stub LDAP/OIDC backends, the client key machinery in
`clients/shared/auth-core.ts` (RS256 assertions with `iss/sub/aud/iat/exp/jti`, already
POSTing to a nonexistent `/v1/auth/register`), and cantica-api's invite flow
(`api/v1/endpoints/invites.py`).

Primary target is **studio-api**; Phase 7 aligns cantica-api. Local mode is untouched
throughout (`settings.local_mode` keeps bypassing everything).

> **Status (2026-07-14): Phases 0–4 implemented in studio-api** — schema + startup
> migrations (`orm/migrate.py`), `limbo` seed, flag gate (`auth/flags.py`) wired into
> `get_current_user`, flags/activation/keys admin APIs, `POST /v1/auth/invitations`,
> `POST /v1/auth/register` (key enrolment; legacy local-mode body still accepted as a
> no-op), `POST /v1/auth/assert`, and client helpers
> (`createEnrolmentAssertion` / `createAuthAssertion`, `StudioClient.requestInvitation` /
> `enrollKey` / `assertAuth`). Covered by `tests/test_remote_auth.py` (32 tests).
> **Phases 5–7 implemented (2026-07-14):**
> Phase 5 — `directory_group_roles` mapping table + `/v1/directory/mappings` CRUD,
> real OIDC backend (JWKS validation, `POST /v1/auth/oidc`), real LDAP backend
> (service-account search + user bind, `ldap3` behind the `[ldap]` extra), and
> directory provisioning (`auth/provision.py`: e_user_id, group→role mapping with
> replace-on-login semantics, limbo+newbie fallback) — `tests/test_directory.py`.
> Phase 6 — webview admin screens on the shared modal rail: AdminUsersModal
> (spec-style auto-activated checkbox + newbie review + flags management) and
> DirectoryMappingsModal, wired through both hosts; `X-Cantica-Warning` surfaces
> as a toolbar badge.
> Phase 7 — cantica-api aligned: `user_flags`/`jwt_keys`/`used_jtis` +
> `users.e_user_id` (Atlas migration `20260714125307_remote_auth`), per-request
> flag gate in `get_current_user`, `auto_activate_users` newbie path on invite
> acceptance, and `/v1/auth/register` + `/v1/auth/assert` with claim formats
> shared with studio-api — `tests/api/test_keyauth*.py`.

---

## Phase 0 — Schema & seeds (foundations)

**Migrations first.** `main.py` uses `Base.metadata.create_all`, which creates missing
*tables* but never adds *columns* to existing ones — adding `users.e_user_id` silently
won't apply to existing databases. Introduce Alembic (or a minimal startup migration
runner) before any schema change below.

New schema (`orm/models.py`):

- `users.e_user_id` — `String(255) | None`, indexed, unique-when-not-null.
- `user_flags` — `id`, `user_id` FK (CASCADE), `flag` (string code), `comment`
  (String(500), default ""), `created_by` (user id), `created_at`. Multiple rows per
  user. Flag codes as a checked vocabulary (constants, not a closed DB enum, since "flags
  will evolve"): `newbie`, `warning:abuse`, `warning:suspicious`, `warning:none`,
  `blocked:abuse`, `blocked:suspicious`, `blocked:none`, `pending:roles`, `ok`.
- `jwt_keys` — `id`, `cantica_user_id` (String, unique, indexed), `user_id` FK,
  `public_key` (Text, PEM — **public key only**, per security note 1), `created_at`,
  `last_used_at`, `revoked_at | None`.
- `used_jtis` — `jti` (PK), `purpose` (`invite` | `enrol` | `auth`), `expires_at`
  (pruned periodically) — replay protection for every client-signed JWT.

Seeds (`orm/seed.py`):

- `limbo` role — description "No access (default for unassigned users)",
  **zero permissions** (deny-by-default works because every endpoint already goes
  through `require_permission`).
- Settings (`config.py`): `default_roles: list[str] = ["limbo"]`,
  `auto_activate_users: bool = False`, `invite_expire_minutes`, `assertion_max_age_seconds`.

Tests: model round-trips, seed idempotency, migration on a pre-existing DB.
**Exit criteria:** schema present on fresh + upgraded DBs; `limbo` seeded.

## Phase 1 — Flag-aware authentication core (Auth F)

- `get_current_user` (`auth/deps.py`): load flags + `is_active` for **both** credential
  paths on every request (the JWT path currently never checks `is_active` — fix). Outcome
  mapping, with security note 6 applied:
  - active + no negative flags → authenticated;
  - active + `warning:*` → authenticated; warnings exposed as `CurrentUser.warnings`
    and an `X-Cantica-Warning` response header (middleware);
  - `blocked:*` or inactive or unknown → **401 with one generic body** (no state
    disclosure); the specific reason (blocked/inactive/not-found + flag comments) goes to
    a structured audit log and is visible to admins via the flags API.
- Flags API (`api/v1/users.py`): `GET/POST/DELETE /v1/users/{id}/flags`
  (`users:read`/`users:write`); flags included in user list/detail responses; adding any
  `blocked:*` flag takes effect on the next request (no token revocation list needed
  because flags are re-read per request — measure, and add per-request caching if hot).
- Activation endpoint: `POST /v1/users/{id}/activate` → sets `is_active=True`, removes
  `newbie`.

Tests: per-row truth table of Auth F; token issued before a `blocked` flag stops working.
**Exit criteria:** the six Auth-F outcomes observable (admin view) while external
responses stay uniform.

## Phase 2 — Invitations & registration, non-enterprise (spec C)

- `POST /v1/auth/invitations` (public, rate-limited): `{first_name, last_name, email}` →
  creates the user (`is_active` per `auto_activate_users`, `newbie` flag when disabled,
  roles = `default_roles`) and returns the **invitation JWT**: HS256-signed by the server,
  claims `{sub: user_id, email, roles, purpose: "invite", jti, exp}` (C.2 "groups the
  user will be part of" = the assigned role names). Single-use via `used_jtis`.
  - Email-ownership proof: deliver the invitation JWT by email (reuse cantica-api's
    mailer pattern) rather than in the HTTP response, at least when SMTP is configured.
  - Duplicate email → same generic "invitation sent" response (no enumeration).
- Newbie review API: `GET /v1/users?flag=newbie` filter (backs the activation screen).
- Admin-initiated invites (`users:write`) share the same issuance path.

Tests: newbie vs auto-activate paths, default-roles fallback to `limbo`, invite expiry,
single-use, rate limiting, no-enumeration responses.
**Exit criteria:** a user can be invited, exist as `newbie`+`limbo`, and be activated by
an admin.

## Phase 3 — Key enrolment (spec B&C 3–8)

Builds on `auth-core.ts` as-is (RS256, 5-min assertions with `jti`).

- `POST /v1/auth/register` — **make the endpoint the client already calls real.**
  Request: `{assertion: <JWS signed with the user's private key>, public_key_pem}` where
  the assertion's payload embeds the invitation JWT (step 4) plus `iat/exp/jti`.
  Server steps (spec 7–8):
  1. verify the embedded invitation JWT (server HS256, unexpired, `jti` unused);
  2. verify the outer signature **with the supplied public key** (RS256);
  3. match invitation identity against the stored user (B.2/C.2);
  4. derive `cantica_user_id` = `e_user_id` (enterprise) or email (non-enterprise);
  5. store the **public key** in `jwt_keys` (`cantica_user_id`, `user_id`), burn both
     `jti`s, return a confirmation JWT signed by the **server** (security note 2).
- Re-enrolment (new machine/key): allowed while authenticated, or via a fresh admin
  invite; multiple active keys per user supported by the table (revoke via
  `DELETE /v1/users/{id}/keys/{key_id}` + `revoked_at`).
- Client updates (`auth-core.ts` / `studioManager` flow): send the enrolment payload
  above instead of the bare `{client_id, public_key_pem}`; surface enrolment failures
  instead of the current silent catch in `_syncCredentials`.

Tests: happy path, identity mismatch, replayed invitation, replayed assertion, garbage
keys, private-key material rejected (PEM type check).
**Exit criteria:** studio client enrols end-to-end against a remote server; the silent
`/v1/auth/register` failure loop is gone.

## Phase 4 — Key-based authentication (spec Auth A–E)

- `POST /v1/auth/assert`: body = authentication JWT signed with the user's private key;
  unverified header/payload carries `cantica_user_id` (as `iss`/`sub`).
  Server: look up `jwt_keys` by `cantica_user_id` (non-revoked) → verify RS256 signature
  → enforce `iat/exp` window (`assertion_max_age_seconds`) + `jti` single-use → run the
  Phase-1 flag gate → issue the existing short-lived HS256 **access token**
  (`create_access_token`), so every downstream endpoint keeps working unchanged.
- `makeCachedAssertion` already produces exactly this JWT; point it at
  `/v1/auth/assert` and use the returned access token as the Bearer credential (today the
  raw assertion is sent as the Bearer token and never verified).
- Keep `/v1/auth/login` (password) as a parallel path for the web UI; both funnel into
  the same flag gate.

Tests: signature/`exp`/`jti` failures, revoked key, blocked-user assert, wrong-key
cross-user attempts, clock-skew tolerance.
**Exit criteria:** remote-mode studio client authenticates purely via its key pair.

## Phase 5 — Enterprise environment (spec B)

- New mapping table `directory_group_roles` (`external_group` → `role_id`), admin CRUD at
  `/v1/directory/mappings` (`roles:write`) — the spec maps AD groups to **roles**; the
  existing `groups.external_id` / `*_group_map` settings keep handling group membership
  separately.
- Implement the `ldap_backend.py` / `oidc_backend.py` stubs: authenticate/validate,
  pull first/last/email + directory groups, then **provision**: create-or-update the user
  with `e_user_id`, assign roles from the mapping (fallback `default_roles` → `limbo`),
  flag `newbie` when unmapped.
- Enterprise users then continue through Phase 3 enrolment with
  `cantica_user_id = e_user_id`.

Tests: fake OIDC token / LDAP fixture provisioning, mapping precedence, unmapped-group
fallback.
**Exit criteria:** an OIDC/LDAP identity yields a provisioned, role-mapped user without
admin intervention.

## Phase 6 — Admin screens

Where: the studio webview already has the modal/store/protocol rail (SetupModal pattern)
for VS Code + Electron parity; cantica-web is the alternative host if these become
server-admin rather than studio-admin concerns. Decide once Phase 1–2 APIs are stable.

1. **User activation screen** (spec C.pre.2): auto-activated checkbox that, when
   unchecked, lists `newbie` users with an Enable action.
2. **Flags management**: per-user flag list, add/remove with comment.
3. **Directory mapping screen** (spec B.pre.2): CRUD for `external_group → role`.
4. Warning surfacing in clients: show `X-Cantica-Warning` as a toolbar/status notice.

## Phase 7 — cantica-api alignment

Reuse its existing invite flow; close the gaps against the same spec:

- Add `user_flags` + `newbie` review path to `/v1/invites/{token}/accept`
  (`auto_activate` setting; today accounts activate immediately).
- Add `e_user_id`, `jwt_keys`, `used_jtis` tables and the `/v1/auth/assert` +
  `/v1/auth/register` endpoints, sharing claim formats with studio-api so one client key
  pair can serve both (consider extracting the JWT/claims code into a small shared
  package, e.g. `actor-ai`-style wheel or a `cantica-auth` module vendored into both).
- Apply the Phase-1 flag gate inside its `get_current_user`.

## Cross-cutting

- **Threat tests** (every phase): user enumeration probes, replay, revocation latency,
  private-key-upload rejection.
- **Audit log**: structured entries for register/enrol/assert/flag changes — required by
  the generic-error policy (admins need the real reason somewhere).
- **Backfill**: existing users get no flags (treated as `all good`); existing password
  login continues to work throughout, so rollout is additive.
- **Sequencing**: 0 → 1 → 2 → 3 → 4 are strictly ordered; 5 and 6 can proceed in
  parallel after 2; 7 any time after 4.
