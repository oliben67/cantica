# Auth Tokens: Opaque API Keys vs JWT

## What Cantica currently uses

Cantica uses **opaque API keys** — random strings generated with `secrets.token_urlsafe(32)`, stored as SHA256 hashes in SQLite, and passed in the `X-API-Key` header. Every request hits the database to validate the key and update `last_used_at`.

---

## The case for JWT

JWTs are self-contained: the server signs the token at login and never stores it. Subsequent requests are verified by checking the signature — no database read required.

**Advantages:**

| Property | JWT |
|---|---|
| Stateless validation | Yes — HMAC/RSA signature check only |
| Horizontal scaling | Easy — any replica can verify |
| Carries claims | Sub, expiry, roles, scopes in the payload |
| Federation / SSO | Natural fit — issuers and audiences in spec |
| Standard ecosystem | `python-jose`, `PyJWT`, many libraries |

A typical flow:
1. `POST /v1/auth/login` → server issues `{"access_token": "<jwt>", "expires_in": 3600}`
2. Client sends `Authorization: Bearer <jwt>`
3. Server decodes header + payload, verifies signature, checks `exp` — no DB touch
4. Revocation via short TTL (15 min access token + refresh token flow)

---

## The case against JWT for Cantica

### 1. Revocation is hard
JWTs are valid until expiry. To revoke a compromised key immediately, you need a blocklist — which is a database lookup on every request, eliminating the statelessness advantage.

### 2. Cantica is primarily a single-server tool
Phase 0–2 assumes a single SQLite instance. Horizontal scaling and replica coordination are Phase 7+ concerns. The stateless scaling argument doesn't apply yet.

### 3. API keys are simpler to manage for machine clients
Cantica's target users are CI pipelines, scripts, and CLI tools. These clients store a long-lived secret (the API key), not a short-lived access token that needs refreshing. Refresh-token flows add complexity for no benefit in this context.

### 4. SQLite lookup is not the bottleneck
An `indexed lookup` on `key_hash` (VARCHAR 64, unique index) in SQLite is ~0.1 ms. At Cantica's expected RPS this is negligible. If it becomes a bottleneck in Phase 6+, the lookup can be cached in-process (an LRU dict keyed by hash).

### 5. Simpler secret rotation
Rotating an opaque API key is: delete the old row, insert a new row. Rotating a JWT signing key requires a key-rollover period where two keys are simultaneously valid — operationally more complex.

---

## When to reconsider JWT

Adopt JWT when Cantica needs any of:

- **Multi-tenant login** — users authenticate interactively via browser (OAuth2/OIDC), then receive short-lived access tokens
- **Federated identity** — Cantica trusts tokens issued by an external IdP (GitHub, Google Workspace, Okta)
- **Fine-grained scopes per request** — e.g., `read:osteck/architect` embedded in the token claim
- **Multiple stateless replicas** that cannot share a database connection

---

## Decision

**Keep opaque API keys for Phase 0–3.** They are simpler, revocable instantly, and perfectly suited to machine-to-machine authentication with a single SQLite backend.

If Phase 5 introduces a browser-based UI with interactive login, add a JWT layer *on top* — the API key system continues to serve CLI/CI clients while the JWT layer handles browser sessions. The two models are complementary, not mutually exclusive.

---

## Implementation note

If JWT is added later, prefer **PyJWT** (simpler, pure Python) over `python-jose` (stale maintenance as of 2024). Use `HS256` for single-server deployments, `RS256` when external parties need to verify tokens. Store the secret in `CANTICA_JWT_SECRET` (env-based, not in the database).
