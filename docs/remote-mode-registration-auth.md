# Remote Mode — Multi-User Registration & Authentication

Design for **remote mode** on the Studio API server and the Cantica API server — referred to
below as the **`api server`**.

- In remote mode, the server accommodates **multiple users**.
- Users are added **per invitation** only.

---

## Registration

### A) General prerequisites

1. **Roles** shall exist on the `api server`.
2. A **null / no-op role** shall exist, called **`limbo`**.
3. An **`e_user_id`** column shall be added to the `users` table to store the user id
   provided by the enterprise infrastructure (nullable).
4. A **flags table** shall be created for users, containing specific marks:

   | Flag | Meaning |
   |---|---|
   | `newbie` | New user, not yet activated, waiting for approval |
   | `warning — abuse` | Abuse warning |
   | `warning — suspicious activities` | Suspicious-activity warning |
   | `warning — no comment` | Unspecified warning |
   | `blocked — abuse` | Blocked for abuse |
   | `blocked — suspicious activities` | Blocked for suspicious activity |
   | `blocked — no comment` | Blocked, unspecified |
   | `pending` | Asked for new roles |
   | `all good` | Nothing is wrong |

   The flag set will evolve, but **`newbie` shall always exist**, and a user can hold
   **several flags simultaneously**.

5. A **JWT key table** shall be created storing the key material received at the end of
   the registration flow (steps B/C 5–8), with columns:

   | Column | Type |
   |---|---|
   | `cantica_user_id` | STRING |
   | `user_id` | UUID (FK → users) |
   | `jwt_key` | binary data |

### B) Enterprise environment

The invitation is **[AD-group-like information × identification]** from the enterprise
infrastructure.

Prerequisites:

1. "AD" groups need to be **mapped to roles** existing on the `api server`, set by the
   server admin ahead of time.
2. **Admin screens** for mapping "AD" groups to `api server` roles need to exist.

Flow:

1. The `api server` retrieves the user's first name, last name, and email from the user
   identification.
2. The `api server` creates the user with the data from step 1, storing the enterprise
   user id in `users.e_user_id`.

### C) Non-enterprise environment

The invitation is **[first name, last name, email × identification]**.

Flow:

1. The user requests an invitation from the server, providing basic information: first
   name, last name, and email.
2. The `api server` creates the user with the data from step 1 and sends back a JWT
   token containing the groups the user will be part of. Either:
   - **a)** the account is created **disabled** and flagged **`newbie`** (see A.4), to be
     reviewed and activated by an admin later; **or**
   - **b)** the account is created **enabled**.

   Prerequisites:

   1. A list of **default roles** shall be defined by the admin, given to any new user.
      If nothing is defined, the role **`limbo`** alone is used.
   2. An **activation screen** for newly registered users shall exist: the form has an
      auto-activated checkbox that, when not populated, lists the newly created users
      (flagged `newbie`) and lets the admin enable them.

### B & C, continued — key enrolment

3. The user requests a **new JWT** from the server, needed for the creation of a
   **key-signed JWT**.
4. The user inserts the invitation into the JWT.
5. The user signs the token with their **private SSH key**, producing a key-signed JWT.
6. The user sends the key-signed JWT back to the `api server`.
7. The `api server` verifies the JWT and extracts the provided data — it **must match**
   the data recorded in B.2 or C.2.
8. If it all checks out:
   - **a)** enterprise user: `e_user_id` is inserted into a new JWT as
     **`cantica_user_id`**; **or**
   - **b)** non-enterprise user: the user's **email** is inserted into a new JWT as
     **`cantica_user_id`**;
   - **c)** the resulting JWT is bound to the SSH key that signed the original key-signed
     JWT, and the key is stored in the **JWT key table** together with
     `cantica_user_id` and the `user_id` from the users table.

---

## Authentication

- **A)** Enterprise user: the user id from the enterprise authentication is used as
  `cantica_user_id`.
- **B)** Non-enterprise user: the user email stored in the system is used as
  `cantica_user_id`.
- **C)** The user generates a new **authentication JWT**, inserts `cantica_user_id`, and
  signs the JWT with their **SSH private key**.
- **D)** The user sends the authentication JWT to the `api server`.
- **E)** The `api server` extracts the key and `cantica_user_id`, retrieves the row for
  `cantica_user_id` from the JWT key table, and **verifies the signature** against the
  stored key.
- **F)** The `api server` looks at the user row (via `user_id`) and returns:

  | User state | Result |
  |---|---|
  | active, not negatively flagged | **authenticated** |
  | active, `warning` flagged | **authenticated, with warning** |
  | active, `blocked` flagged | **not authenticated** + flag info |
  | inactive, `warning` flagged | **not authenticated**, with warning |
  | inactive, `blocked` flagged | **not authenticated** + flag info + disabled-account info |
  | not found | **not authenticated** — "please contact …" |

---

## Current implementation status (audited 2026-07-14)

Legend: ✅ implemented · 🟡 partial · ❌ missing

### studio-api (`cantica-studio/studio-api`)

| Spec item | Status | Notes |
|---|---|---|
| Multiple users in remote mode | ✅ | `users` table, JWT login (`/v1/auth/login`), API tokens; `local_mode` bypasses auth entirely |
| Roles exist (A.1) | ✅ | Full RBAC: `roles`, `permissions`, `user_roles`; seeded `admin` / `operator` / `viewer` |
| `limbo` role (A.2) | ❌ | No null role seeded; new users get no role fallback |
| `users.e_user_id` (A.3) | ❌ | No column; `groups.external_id` exists but maps directories to **groups**, not users |
| User flags table (A.4) | ❌ | Only boolean `users.is_active`; no `newbie`/`warning`/`blocked` flags, no multi-flag support |
| JWT key table (A.5) | ❌ | No table. `clients/shared/auth-core.ts` generates a client key pair and POSTs to `/v1/auth/register` — **that endpoint does not exist server-side**; registration always fails and is silently swallowed by `_syncCredentials` |
| AD-group → role mapping (B.pre) | 🟡 | `ldap_group_map` / `oidc_group_map` settings exist but map to **groups**, and both backends (`ldap_backend.py`, `oidc_backend.py`) are stubs |
| Admin mapping screens (B.pre.2) | ❌ | No UI |
| Enterprise user auto-creation (B.1–2) | ❌ | Backends are stubs; no user provisioning from directory data |
| Invitation flow (C.1–2) | ❌ | No invite endpoints; users are created by admins via `/v1/users` |
| Default roles for new users (C.pre.1) | ❌ | Not configurable |
| Activation screen (C.pre.2) | ❌ | `users.is_active` is toggleable via the users API, but there is no newbie-review UI |
| Key enrolment (B&C 3–8) | ❌ | No key-signed JWT support; JWTs are HMAC-signed with the server's `jwt_secret` only |
| SSH-key authentication (Auth A–E) | ❌ | Bearer JWT (HS) or opaque API token only |
| Flag-aware auth outcomes (Auth F) | 🟡 | Only `is_active` is checked (API-token path returns 401 "Account inactive"); no warning/blocked distinctions |

### cantica-api (`cantica-api`)

| Spec item | Status | Notes |
|---|---|---|
| Multiple users | ✅ | Users with role enum (`admin`/`user`/`readonly`/`anonymous`), JWT sessions, static API keys (SHA-256-hashed) |
| Invitation flow (C) | 🟡 | **Closest match in the codebase**: `/v1/invites/{token}` validate + `/v1/invites/{token}/accept` — email invites (SMTP mailer + QR), single-use, expiring; creates user + returns JWT. But accounts are created **active immediately** (no C.2a newbie/approval path) and the invite JWT does not carry groups |
| `limbo` role | ❌ | Role enum is closed; no null role |
| `e_user_id` | ❌ | Not present |
| Flags table | ❌ | Only `is_active` |
| JWT key table / SSH-key JWTs | ❌ | HMAC JWT + password (bcrypt) + API keys only |
| AD mapping | ❌ | Not present |

### Overall

The **spec is not yet implemented** on either server. Existing building blocks to reuse:
studio-api's RBAC + users/groups/roles admin API, its (stub) LDAP/OIDC backends and
group-map settings, the client-side key-pair + assertion machinery in
`clients/shared/auth-core.ts`, and cantica-api's invite token flow and mailer.

---

## Security review notes on this design

Items to resolve before implementation — the flow is sound overall, but wording must be
tightened so the implementation doesn't take it literally:

1. **Never transmit or store the private key.** Steps 5–8 and the "JWT Private Key
   table" must be read as: the client signs with the **private** SSH key and the server
   receives, verifies against, and stores only the **public** key (`jwt_key` = public key
   bytes). If the server ever receives private-key material the whole scheme is void.
2. **Step 8c as written says the server signs a JWT "with the extracted SSH key".** The
   server cannot (and must not) sign with the user's key. Either the server signs the
   confirmation JWT with **its own** key, or step 8c is only the *binding* of the user's
   public key to `cantica_user_id` in the key table (recommended reading).
3. **Invitation JWT (C.2) needs replay protection**: short expiry, single-use (`jti`
   tracked server-side), and audience/issuer claims — same properties cantica-api's
   invite tokens already have (single-use + expiry).
4. **Authentication JWTs (Auth C–E) need freshness**: require `iat`/`exp` with a short
   window and a nonce/`jti`, otherwise a captured token can be replayed for its lifetime.
5. **`cantica_user_id` = email for non-enterprise users** ties identity to a mutable
   attribute. Store the immutable `user_id` alongside (the key table already does) and
   treat email strictly as a lookup alias.
6. **Auth F leaks account state to unauthenticated callers** ("content of flag info",
   distinguishable "not found" message). Distinguishable errors enable **user
   enumeration** and disclose moderation state. Recommend: identical generic failure for
   `blocked` / `inactive` / `not found` externally, with the detailed reason available to
   admins (audit log) and, at most, delivered out-of-band to the account owner.
7. **`limbo` must be a real deny-by-default role** (no permissions), not the absence of a
   check — every endpoint must require an explicit permission (`require_permission` in
   studio-api already works this way).
8. **Flags must be evaluated on every request**, not only at login — a `blocked` flag has
   to invalidate live JWTs/API tokens (check flags in `get_current_user`, or keep a
   token-revocation list keyed by user).
