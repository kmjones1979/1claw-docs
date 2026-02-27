# Phase 1 Security Audit: Authentication & Authorization

**Scope:** Human auth, agent auth, MCP auth, API keys.  
**Method:** Read actual code in vault, dashboard, packages/mcp, packages/sdk where relevant.

---

## 1.1 Human Authentication

### How humans authenticate
- **Email/password:** `POST /v1/auth/token` (handlers/auth.rs `create_token`). Password verified via Argon2 against `user.password_hash`. No separate “login” rate limit beyond global rate limit.
- **Google OAuth:** `POST /v1/auth/google` (handlers/auth.rs `google_auth`). ID token verified with `https://oauth2.googleapis.com/tokeninfo?id_token=...`; audience and `email_verified` are checked.
- **Device flow (CLI):** `POST /v1/auth/device/code` → user approves via dashboard `POST /v1/auth/device/approve` (authenticated) → `POST /v1/auth/device/token` returns JWT. Device codes expire in 10 minutes.

### JWT issuance
- **Lifetime:** User JWT: `jwt_access_token_expiry_secs` (default 900). MFA challenge token: 300s. Device-flow JWT: 86400s hardcoded (auth.rs:738).
- **Refresh:** `POST /v1/auth/refresh` returns `BadRequest("Refresh tokens not yet implemented")` (auth.rs:297–302).
- **Revocation:** `DELETE /v1/auth/token` returns `{"revoked": true}` but does not add the token to any revocation list (auth.rs:305–308). JWTs remain valid until expiry.

### Password requirements
- Enforced in `validate_password_strength` (auth.rs:417–441): ≥12 chars, one uppercase, one lowercase, one digit, one special, no 3+ repeating, no 3+ sequential. Used on signup and change-password.

### Rate limit and lockout
- **Login:** No dedicated rate limit for `/v1/auth/token` or `/v1/auth/google`. Only the global `rate_limit_middleware` (rate_limit.rs) applies (default 100 req/s, burst 200 per IP).
- **Account lockout:** No lockout after N failed logins. Failed login returns generic `Unauthorized("Invalid credentials")`.

### IDOR (vault access by ID)
- **Checked:** vaults.rs `get_vault`, `delete_vault`, enable/disable CMEK; secrets.rs all handlers; policies, agents, sharing.
- **Enforcement:** After loading vault/secret, every handler checks `vault.org_id != caller.org_id` → `Forbidden("Access denied")`. Caller’s `org_id` comes from JWT or API key (auth middleware). Agents additionally restricted by `enforce_vault_binding` when `vault_ids` is non-empty.
- **Finding:** No IDOR: one user cannot access another org’s vaults by ID.

### CSRF on state-changing endpoints
- **Checked:** All state-changing endpoints require `Authorization: Bearer <token>` or `Bearer 1ck_...`. No cookie-based session for the API; dashboard uses its own auth (token in memory/localStorage typical for SPA).
- **Finding:** No CSRF finding for the vault API; state-changing operations require Bearer/API key and are not cookie-driven.

---

### Findings (1.1 Human Auth)

| # | Location | Category | Severity | Description | Impact | Remediation |
|---|----------|----------|----------|-------------|--------|-------------|
| 1.1.1 | vault/src/api/handlers/auth.rs:305–308 | Auth / Revocation | **MEDIUM** | `revoke_token` returns success but does not revoke the JWT; no revocation list or short expiry implemented. | Stolen or leaked JWTs remain valid until `exp`. | Implement token revocation (e.g. short-lived access tokens + refresh tokens with revocation, or a revocation store and check in auth middleware). |
| 1.1.2 | vault/src/api/handlers/auth.rs:297–302 | Auth / Refresh | **LOW** | Refresh tokens are not implemented; long-lived usage relies on a single access token. | Users must re-authenticate when token expires; no way to rotate without re-login. | Implement refresh token flow and use it for long-lived sessions. |
| 1.1.3 | vault/src/config.rs, rate_limit.rs | Auth / Brute force | **MEDIUM** | No per-identity (e.g. per-email) or per-endpoint rate limit for login; no account lockout after failed attempts. | Credential stuffing and brute force on passwords are easier. | Add a stricter rate limit for `POST /v1/auth/token` (and optionally Google) per IP and/or per email, and consider lockout or backoff after N failures. |
| 1.1.4 | vault/src/api/handlers/auth.rs:797–812 | Info disclosure | **LOW** | `GET /v1/auth/device/code/{user_code}` is unauthenticated; anyone can poll status (pending/approved/denied/expired). | Enumerating or confirming device codes. | Consider requiring a minimal proof (e.g. device_code in body/query for same client) or rate limiting this endpoint heavily. |

---

## 1.2 Agent Authentication

### Agent-token flow
- **Endpoint:** `POST /v1/auth/agent-token` with `{ agent_id, api_key }` (auth.rs `create_agent_token`).
- **Validation:** Agent loaded by ID; `is_active` and `expires_at` checked. For `auth_method == "api_key"`, `api_key_hash` is verified with Argon2 (PasswordHash::new + Argon2::default().verify_password). Key is **ocv_**-prefixed; stored value is Argon2 hash only (agent_repo stores `api_key_hash`).
- **JWT claims:** Built in auth.rs:163–168: `sub: "agent:{id}"`, `org`, `scopes`, `vault_ids`, `exp`, `iat`, `jti`, `crypto_proxy_enabled`. Signed with EdDSA (HSM), header `{"alg":"EdDSA","typ":"JWT"}`.
- **Algorithm:** EdDSA (GCP KMS or SoftHSM), not HMAC; verification in auth middleware via `JwtValidator` (DecodingKey::from_ed_der, Validation::new(Algorithm::EdDSA)).
- **Lifetime:** `token_ttl_seconds` per agent or default `jwt_agent_token_expiry_secs` (e.g. 3600).

### Policy enforcement when agent requests secret
- **Order:** Vault loaded → `vault.org_id == caller.org_id` → `enforce_vault_binding` (agent’s `vault_ids` if set) → `enforce_scope_access` (at least one JWT scope glob-matches path) → policies loaded for (vault, principal_type, principal_id) → `PolicyEngine::check_access(policies, caller, path, "read"|"write", client_ip)` → owner bypass (vault creator) → then DB/secret service.
- **Path traversal:** Secret path is validated on **write** in `secret_service::validate_path`: PATH_REGEX allows only `[a-zA-Z0-9_\-/]`, no `..` or `.`. On **read**, path is not re-validated; it is used as-is for scope glob and DB lookup. So `foo/../bar` is not storable; if sent on read, scope check uses path segments (e.g. `foo`, `..`, `bar`). Pattern `foo/*` matches segment `*` to `..`, so scope check can pass for `foo/../bar`; DB lookup is for literal `foo/../bar`, which typically does not exist → 404. So no cross-path read, but scope semantics are loose for path traversal strings.
- **Glob:** `domain/policy_engine.rs` `glob_match`: `*` = one segment, `**` = zero or more segments; segments split by `/`. No normalization of `..` in path; pattern is applied to raw segments.
- **Enumeration / timing:** get_secret returns 403 for “Insufficient permissions” vs 404 for “Secret not found” (from secret_service). So 403 vs 404 can leak whether a path exists in the vault. List_secrets filters by policy/scope and returns only allowed paths, so it does not leak other paths.

### Findings (1.2 Agent Auth)

| # | Location | Category | Severity | Description | Impact | Remediation |
|---|----------|----------|----------|-------------|--------|-------------|
| 1.2.1 | vault/src/domain/secret_service.rs, vault/src/api/handlers/secrets.rs | Path / Scope | **LOW** | Read path is not normalized and not validated with the same PATH_REGEX as write. Scope check can match paths containing `..` (e.g. `foo/*` matches `foo/../x`). | Confusing scope behavior; no direct escape to another path because DB key is literal. | Normalize path on read (reject or collapse `..` and `.`) and/or run read path through the same validate_path rules as write. |
| 1.2.2 | vault/src/api/handlers/secrets.rs (get_secret, delete_secret, etc.) | Info disclosure | **LOW** | 403 “Insufficient permissions” vs 404 “not found” allows inferring existence of a secret path. | Agents (or users) can enumerate paths by response type. | Use a single generic error for “no access or not found” (e.g. 404 with same message), or ensure 403 is only for “authenticated but not allowed” and 404 for “no such resource” with no distinction for “no such path in this vault”. |

---

### No finding (1.2)
- **ocv_ key validation:** Done with Argon2 in auth.rs (create_agent_token) and hash stored in agents.api_key_hash (agent_repo).
- **JWT claims and algorithm:** EdDSA, required claims (sub, org, exp, iat, jti), scopes and vault_ids enforced in middleware and handlers.
- **Deactivated agents:** create_agent_token returns Unauthorized when `!agent.is_active`.
- **Policy enforcement:** get_secret, put_secret, delete_secret, list_secrets, list_versions all use PolicyEngine (or owner bypass) before returning secret data; path and vault are constrained by org, vault_ids, and scopes.

---

## 1.3 MCP Auth

### Bearer and X-Vault-ID
- **stdio mode:** Single `OneClawClient` at startup with `ONECLAW_VAULT_ID` and either `ONECLAW_AGENT_ID` + `ONECLAW_AGENT_API_KEY` or `ONECLAW_AGENT_TOKEN`. Every tool call uses `resolveClient()` → same client. Client sends Bearer (JWT from agent-token exchange or static token) on every request (client.ts `headers()` → `ensureToken()` then `Authorization: Bearer ${this.token}`). Vault ID is in the URL path (`/v1/vaults/${this.vaultId}/...`), not as a separate header to the API.
- **httpStream mode:** `authenticate` callback (index.ts:84–101) requires `Authorization` (Bearer) and `X-Vault-ID`; rejects if missing. Session is `{ token, vaultId }`. Each tool’s execute receives `context.session` and `resolveClient(context.session)` builds a client with that token and vaultId; that client is used for all API calls (same Bearer + vault in path). So Bearer is validated by the Vault API on every request; the MCP server does not validate the token itself, it forwards it. X-Vault-ID is only used by the MCP layer to choose which vault to call; the API then enforces org and vault binding.

### Per-tool authorization
- All 11 tools (list_secrets, get_secret, put_secret, delete_secret, describe_secret, create_vault, list_vaults, grant_access, share_secret, simulate_transaction, submit_transaction) plus rotate_and_store and get_env_bundle call `resolveClient(context.session)` (or shared client). No tool runs without a client; in httpStream, no client without passing `authenticate`. So every tool is gated by the same auth: Bearer + (in httpStream) X-Vault-ID. The Vault API then enforces org, vault_ids, scopes, and policies per endpoint.

### Finding (1.3)

| # | Location | Category | Severity | Description | Impact | Remediation |
|---|----------|----------|----------|-------------|--------|-------------|
| 1.3.1 | packages/mcp/src/index.ts | Auth / Validation | **INFO** | MCP server does not verify the Bearer token; it forwards it to the Vault API. In httpStream, X-Vault-ID is not cross-checked against the token server-side (only used to build request URLs). | If the API is misconfigured or bypassed, MCP could send requests to the wrong vault. Relies entirely on API enforcement. | Optional: have MCP call a lightweight “validate token + vault” endpoint that returns 403 if token’s vault_ids do not include the given vault, to fail fast before tool execution. |

---

### No finding (1.3)
- Bearer is sent on every tool-originated request (client always calls `ensureToken()` and adds Authorization).
- All 11 tools (plus rotate_and_store and get_env_bundle) require a resolved client (session or shared); no unauthenticated tool path.
- Vault ID is enforced by the API (org_id and enforce_vault_binding) when the request hits the backend.

---

## 1.4 API Keys

### Generation (1ck_ and ocv_)
- **1ck_ (user API keys):** api_keys.rs `generate_user_api_key()`: 32 bytes from `rand::rngs::OsRng`, then `base64::engine::general_purpose::URL_SAFE_NO_PAD.encode`; prefix `"1ck_"` (api_keys.rs:29–35).
- **ocv_ (agent API keys):** agents.rs `generate_api_key()`: same method, 32 bytes OsRng, URL_SAFE_NO_PAD, prefix `"ocv_"` (agents.rs:49–55). Sufficient entropy for keys.

### Storage (hashing)
- **User keys:** `hash_api_key` in api_keys.rs: Argon2::default(), SaltString::generate(OsRng), hash_password; stored in user_api_keys.api_key_hash. Only key_prefix (first 12 chars) stored in plaintext for lookup (api_keys.rs:60–76, user_api_key_repo).
- **Agent keys:** Same pattern in agents.rs `hash_api_key`; stored in agents.api_key_hash (agent_repo). No plaintext key stored.

### Rotation
- **Agent key rotation:** agents.rs `rotate_agent_key`: generates new key, `hash_api_key`, then `agent_repo::update_api_key_hash(&state.db, agent_id, &new_hash)`. Single UPDATE; old hash replaced. Old key invalid on next request (create_agent_token verifies against current hash).
- **User API keys:** No rotation endpoint; user revokes and creates a new key. Revoke sets `is_active = false` (user_api_key_repo revoke_key).

### Deactivated agents
- create_agent_token checks `if !agent.is_active { return Err(AppError::Unauthorized("Agent is deactivated")) }` (auth.rs:75–77). Deactivated agents cannot obtain a new JWT. Existing JWTs remain valid until exp (no revocation list).

### Finding (1.4)

| # | Location | Category | Severity | Description | Impact | Remediation |
|---|----------|----------|----------|-------------|--------|-------------|
| 1.4.1 | vault/src/api/handlers/auth.rs:75–77 | Auth / Revocation | **LOW** | Deactivating an agent does not invalidate already-issued JWTs; they remain valid until expiry. | Revoked agent can continue to access until token expires. | Implement token revocation or very short agent token TTL when revocation is required; or maintain a blocklist of agent IDs and check in auth middleware. |

---

### No finding (1.4)
- **1ck_ and ocv_ generation:** Crypto-secure (OsRng, 32 bytes, base64).
- **Storage:** Both hashed with Argon2 (salt + default params) before storage; only prefix stored for user keys for lookup.
- **Rotation:** Agent key rotation replaces hash in DB; old key invalid immediately. User keys: revoke = is_active false; no reuse.

---

## Summary

| Severity | Count |
|----------|--------|
| CRITICAL | 0 |
| HIGH     | 0 |
| MEDIUM   | 2 (1.1.1 token revocation, 1.1.3 login rate limit / lockout) |
| LOW      | 5 (1.1.2 refresh, 1.1.4 device code status, 1.2.1 path/scope, 1.2.2 enumeration, 1.4.1 deactivated agent JWT) |
| INFO     | 1 (1.3.1 MCP token/vault validation) |

**Recommended priorities:** Implement token revocation and refresh (1.1.1, 1.1.2), add login-specific rate limiting and optional lockout (1.1.3), then harden path handling and error consistency (1.2.1, 1.2.2), and consider agent JWT invalidation on deactivate (1.4.1).
