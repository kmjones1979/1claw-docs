# Security Audit — Phases 6–10 (x402, MCP, Dependencies, DoS, Checklist)

**Scope:** x402 & on-chain, MCP, dependencies, DoS, and Phase 10 checklist.  
**Codebase:** 1claw (vault, dashboard, packages/mcp, packages/sdk, root).  
**Format:** Location | Severity | Description | Impact | Remediation. Where no finding, what was checked is stated.

---

## Phase 6 — x402 & on-chain

### 6.1 x402: payment verification, replay, race, front-run/revert

| ID   | Location | Severity | Description | Impact | Remediation |
|------|----------|----------|-------------|--------|-------------|
| 6.1.1 | `vault/src/api/middleware/x402.rs` | **MEDIUM** | **Race between verification and secret delivery:** Payment proof replay is checked (`is_payment_proof_used`), then facilitator `verify`, then `mark_payment_proof_used` is called with `INSERT ... ON CONFLICT DO NOTHING`. Two concurrent requests with the same proof can both pass the replay check (both see no row yet), both pass verification, then the first INSERT succeeds and the second INSERT does nothing (no error). Both proceed to `next.run(request)` and receive the secret. One payment proof can thus be spent twice. | Double-spend of a single x402 payment; revenue loss and protocol abuse. | Make proof consumption atomic and fail the second request: e.g. `INSERT INTO x402_payment_proofs (proof_hash, org_id) VALUES ($1, $2) ON CONFLICT (proof_hash) DO NOTHING RETURNING proof_hash`; only proceed to verify + downstream if a row is returned. If 0 rows, return 402 (replay). |
| — | Same file | Checked | **Replay of payment proof (single-request):** Proof is SHA-256 hashed and stored in `x402_payment_proofs`; duplicate proof is checked *before* facilitator verification. Replay is rejected with 402. | — | No change. |
| — | Same file | Checked | **Verification model:** Payment is verified via **facilitator** (`verify_payment_with_facilitator` → POST to `{facilitator_url}/verify`), not on-chain by the vault. Facilitator is CDP (Coinbase). | — | Document that trust is in the facilitator; on-chain verification is out of scope for the vault. |
| — | Same file | Checked | **Front-run/revert:** After verify + mark_used, the request is served. Settle is called only after a successful response. There is no “revert” path that unmarks the proof; once marked used, the proof cannot be reused. | — | No change. |

### 6.2 CDP: integration security, webhooks, fallback

| ID   | Location | Severity | Description | Impact | Remediation |
|------|----------|----------|-------------|--------|-------------|
| — | `vault/src/api/middleware/cdp_auth.rs` | Checked | **CDP integration:** Facilitator requests use Ed25519 JWT (method + path scoped, 120s expiry). Key secret is base64-decoded and used as Ed25519 seed; no raw secrets in logs. | — | No change. |
| — | `vault/src/api/handlers/billing_v2.rs` | Checked | **Webhook signatures:** Stripe webhook requires `Stripe-Signature` header; `verify_stripe_signature` uses HMAC-SHA256 with `STRIPE_WEBHOOK_SECRET`. Rejects missing/invalid signature and timestamps &gt;5 min old. Idempotency via `stripe_events` (event_id) before processing. | — | No change. |
| 6.2.1 | `vault/src/api/middleware/x402.rs` | **LOW** | **Fallback if payment network down:** If the facilitator is unreachable or returns an error, `verify_payment_with_facilitator` returns false and the request gets 402. There is no fallback (e.g. credits or queue). | Legitimate payers cannot complete requests during CDP/facilitator outage. | Document behavior; consider optional fallback (e.g. temporary credit or retry-after) for availability, with clear operational guidance. |

---

## Phase 7 — MCP

### 7.1 Tool injection: prompt injection, server-side validation, malicious secret names

| ID   | Location | Severity | Description | Impact | Remediation |
|------|----------|----------|-------------|--------|-------------|
| — | `packages/mcp/src/tools/*.ts`, `vault/src/api/handlers/secrets.rs`, `vault/src/domain/secret_service.rs` | Checked | **Prompt injection via secret names:** Tool arguments (e.g. `secret_path`, `vault_id`) are passed to the vault API. The API does not interpret path as instructions; path is used as an identifier and in parameterized DB queries. A path like `"; DROP TABLE"` or `"Ignore previous instructions"` is just a string. | — | No change. |
| — | `vault/src/domain/secret_service.rs` | Checked | **Server-side validation:** Path is validated on **write** with `validate_path` and `PATH_REGEX` (`^[a-zA-Z0-9][a-zA-Z0-9_\-/]*[a-zA-Z0-9]$|^[a-zA-Z0-9]$`). Invalid characters (including `;`, spaces, `..`) are rejected. DB access uses `sqlx` with bound parameters; no string concatenation into SQL. | — | No change. |
| — | Same | Checked | **Secret name like "; DROP TABLE":** Such a path fails `PATH_REGEX` on write. On read, path is only used in parameterized queries; no SQL injection. | — | No change. |

### 7.2 Credential scope: blast radius, multi-vault, session isolation

| ID   | Location | Severity | Description | Impact | Remediation |
|------|----------|----------|-------------|--------|-------------|
| — | `packages/mcp/src/index.ts`, `vault` JWT scopes | Checked | **Max blast radius:** Agent access is bounded by (1) `vault_ids` in JWT when set (vault binding), (2) scopes (path globs), and (3) per-vault policies. One agent token cannot access another org’s vaults. | — | No change. |
| — | `packages/mcp/src/index.ts` | Checked | **Multi-vault access:** In stdio mode a single `ONECLAW_VAULT_ID` is used. In `httpStream` mode, each request supplies `X-Vault-ID` and Bearer token; the server resolves the client per request. An agent can have policies on multiple vaults but each MCP session/client is typically configured with one vault. | — | Document that multi-vault usage is via multiple clients or session headers. |
| — | Same | Checked | **Session isolation:** `httpStream` uses per-request auth (Bearer + X-Vault-ID). No server-side session store; each request is authenticated independently. Stdio uses a single shared client. | — | No change. |

---

## Phase 8 — Dependencies

| ID   | Location | Severity | Description | Impact | Remediation |
|------|----------|----------|-------------|--------|-------------|
| 8.1 | Root `pnpm audit` | **HIGH** | **Rollup (dashboard):** “Rollup 4 has Arbitrary File Write via Path Traversal” — path `dashboard>@vitejs/plugin-react>vite>rollup`, vulnerable &gt;=4.0.0 &lt;4.59.0. | Build/dev dependency; could affect local or CI builds. | Upgrade rollup (via Vite/plugin) to &gt;=4.59.0 or patch. |
| 8.2 | Root `pnpm audit` | **HIGH** | **minimatch (dashboard, eslint):** ReDoS (matchOne combinatorial backtracking; nested *() extglobs). Vulnerable &gt;=10.0.0 &lt;10.2.3. Root `package.json` has `pnpm.overrides` for `minimatch` (e.g. `">=10.2.1"`). | Override may already resolve; confirm resolved version is &gt;=10.2.3 in lockfile. | Ensure override is `>=10.2.3` and run `pnpm install` + `pnpm audit`. |
| 8.3 | Root `pnpm audit` | **HIGH** | **Koa (packages/mcp > fastmcp > mcp-proxy > pipenet > koa):** “Host Header Injection via ctx.hostname”, &gt;=3.0.0 &lt;3.1.2. | MCP server may be exposed; host header abuse could affect redirects or logging. | Upgrade koa (via fastmcp/mcp-proxy) to &gt;=3.1.2 or patch; if not possible, constrain MCP server exposure and reverse-proxy validation. |
| 8.4 | Root `pnpm audit` | **MODERATE** | **js-yaml (packages/sdk > openapi-typescript > @redocly/openapi-core):** Prototype pollution in merge (<<), &gt;=4.0.0 &lt;4.1.1. | Used in OpenAPI tooling; low exposure if only used at build/generate time. | Upgrade to &gt;=4.1.1 in dependency chain or override. |
| — | Repo root, `docs/` | Checked | **Lockfiles:** `pnpm-lock.yaml` (root) and `docs/pnpm-lock.yaml` are present and should be committed. | — | Keep lockfiles committed. |
| — | Root and key packages `package.json` | Checked | **Wildcards:** No unbounded wildcards (e.g. `*` or `x.x.x`) in version ranges in the scanned packages; ranges use `^` or `~`. | — | No change. |
| — | Vault (Rust) | Checked | **Supabase/JWT/crypto libs:** Vault uses `sqlx` (Postgres), `jsonwebtoken`, `argon2`, `ed25519-dalek`, `sha2`, etc. No Supabase client in vault. JWT and crypto are standard crates. | — | Continue to track Rust advisory DB and Cargo audit for vault. |

---

## Phase 9 — DoS

| ID   | Location | Severity | Description | Impact | Remediation |
|------|----------|----------|-------------|--------|-------------|
| — | `vault/src/domain/quota.rs`, `vault/src/domain/billing.rs` | Checked | **Exhaust free-tier for others:** Quota is **per-org** (usage_repo, tier_limits). Free tier: 1,000 req/mo, 3 vaults, 50 secrets, 2 agents. One org cannot consume another org’s quota. | — | No change. |
| — | Same | Checked | **Thousands of vaults/secrets:** Enforced by `enforce_vault_limit`, `enforce_secret_limit`, `enforce_agent_limit` at creation time; limits are tier-based (free/pro/business/enterprise). | — | No change. |
| 9.1 | `vault` | **LOW** | **Slowloris / slow-read:** No specific request-body read timeout or slow-read mitigation in the codebase. Relies on Axum/default TCP behavior and any upstream proxy (e.g. Cloud Run). | Prolonged connections could tie up workers. | Prefer timeouts and limits at reverse proxy/load balancer; consider body size and read timeouts in Axum if needed. |
| — | `vault` crypto + KMS | Checked | **Excessive KMS calls:** Each secret read/write does envelope encrypt/decrypt (one or two KMS calls per operation). No caching of DEKs across requests. Rate is naturally limited by request rate and rate_limit_middleware. | — | No change unless scaling requires DEK caching (with clear invalidation). |
| — | `vault/src/api/middleware/rate_limit.rs` | Checked | **MCP / connection exhaustion:** Global `rate_limit_middleware` (token bucket per IP) applies to all routes. MCP over HTTP uses the same API; no separate connection limit in app code. | — | Rely on rate limit and infra; add connection/request limits at proxy if needed. |

---

## Phase 10 — Checklist

| Item | Status | Notes |
|------|--------|------|
| **Secrets never logged** | OK | `audit_middleware` logs only method, path, status, caller; explicitly does not log secret values. Secrets handler does not log request/response bodies or secret values. |
| **Secrets never in URLs** | Partial | Secret **value** is never in URL (value is in PUT body only). Secret **path** is in the URL as path segment for GET/PUT/DELETE `/v1/vaults/{id}/secrets/{*path}`. This is an identifier, not the credential. |
| **All endpoints require auth** | OK | Public: auth endpoints, device code, health. Webhook: unauthenticated but verified with Stripe signature. Share access: unauthenticated, protected by share token + optional passphrase/IP. All other routes go through `auth_middleware`. |
| **Admin not public** | OK | Admin routes live under `authenticated_routes`; `verify_platform_admin` in admin handlers enforces `platform_org_id` and user role (owner/admin). |
| **DB backups encrypted** | Doc | Docs (e.g. customer-managed-keys) state that database backup theft risk is mitigated because backups contain only encrypted data. No code change; operational backup encryption is infra-specific. |
| **Key rotation documented** | OK | Changelog and SECURITY-AUDIT-PHASE1-AUTH mention agent key rotation and user key revoke. |
| **Incident plan** | Partial | Compliance/audit docs mention using the audit log for “incident response”; no formal incident runbook found in repo. |
| **GDPR delete** | OK | User can self-delete via `DELETE /v1/auth/me`; admin can delete users via `DELETE /v1/admin/users/{user_id}` with cascades. |
| **ToS / Privacy consistent** | Not in repo | No ToS or Privacy policy files in the repo; dashboard has a privacy page (content only). Consistency with product is not verifiable from codebase. |
| **Changelog security changes** | OK | Changelog documents security-related changes (rate limiting, replay protection, CORS, CSP, auth enforcement, dependency overrides). |

---

## Summary

- **Phase 6:** One **MEDIUM** (x402 proof race allowing double-spend) and one **LOW** (no fallback when facilitator is down). Replay check and CDP/Stripe security are in good shape.
- **Phase 7:** No issues; tool inputs are validated server-side, path regex and parameterized queries prevent injection.
- **Phase 8:** **4 HIGH/MODERATE** from `pnpm audit` (rollup, minimatch x2, koa, js-yaml); lockfiles present; no wildcard versions; vault uses standard Rust crypto/JWT.
- **Phase 9:** DoS mitigated by per-org quotas and tier limits; optional improvement for slowloris/slow-read at proxy or Axum.
- **Phase 10:** Checklist largely satisfied; secrets not logged, auth and admin correctly gated, GDPR delete and changelog in place. Partial: secret path in URL (by design), incident runbook and ToS/Privacy not in repo.

**Recommended priority:** Fix x402 proof race (6.1.1), then address dependency advisories (8.1–8.4), then consider x402 facilitator fallback (6.2.1) and slowloris/slow-read (9.1).
