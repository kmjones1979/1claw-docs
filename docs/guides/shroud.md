---
title: Shroud — TEE LLM Proxy
description: TEE-protected LLM proxy and transaction signing for AI agents. Secret redaction, prompt injection defense, and key isolation inside AMD SEV-SNP confidential memory.
sidebar_position: 12
---

# Shroud — TEE LLM Proxy & Transaction Signing

Shroud is a Rust service running inside Google Cloud Confidential GKE (AMD SEV-SNP). It sits between AI agents and LLM providers, performing real-time security inspection of all traffic while also handling on-chain transaction signing with keys that never leave the TEE.

## Architecture

```
Agent
  │
  ├── LLM requests ──► shroud.1claw.xyz (GKE TEE) ──► LLM Providers
  │                         │
  │                         ├── Secret redaction
  │                         ├── PII scrubbing
  │                         ├── Prompt injection detection
  │                         ├── Policy enforcement
  │                         └── Audit logging
  │
  └── Transaction requests ──► shroud.1claw.xyz
                                   │
                                   ├── POST /v1/agents/:id/transactions → TEE signing
                                   └── GET/simulate/bundle → Proxied to api.1claw.xyz
```

## Endpoints

**Shroud-hosted (TEE signing):**

| Method | Path | Description |
| --- | --- | --- |
| POST | `/v1/agents/:id/transactions` | Sign and broadcast a transaction inside the TEE |

**Proxied to Vault API:**

| Method | Path | Description |
| --- | --- | --- |
| GET | `/v1/agents/:id/transactions` | List agent transactions |
| GET | `/v1/agents/:id/transactions/:tx_id` | Get a specific transaction |
| POST | `/v1/agents/:id/transactions/simulate` | Simulate a transaction (Tenderly) |
| POST | `/v1/agents/:id/transactions/simulate-bundle` | Simulate a bundle |

**Health/ops (port 8080):**

| Method | Path | Description |
| --- | --- | --- |
| GET | `/healthz` | Liveness probe |
| GET | `/readyz` | Readiness probe |
| GET | `/livez` | Deadlock detection |

**Metrics (port 9090):**

| Method | Path | Description |
| --- | --- | --- |
| GET | `/metrics` | Prometheus metrics |

## LLM Proxy

Agents send LLM requests directly to `shroud.1claw.xyz` with two required headers:

```bash
curl -X POST https://shroud.1claw.xyz/v1/chat/completions \
  -H "Authorization: Bearer $AGENT_TOKEN" \
  -H "X-Shroud-Provider: openai" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

| Header | Required | Description |
| --- | --- | --- |
| `Authorization` | Yes | Agent JWT (from `POST /v1/auth/agent-token`) |
| `X-Shroud-Provider` | Yes | LLM provider: `openai`, `anthropic`, `google`, `mistral`, `cohere` |
| `X-Shroud-Api-Key` | Optional | Fallback LLM API key (used if vault lookup fails) |

### Inspection pipeline

Every request passes through Shroud's inspection pipeline before reaching the LLM:

1. **Secret redaction** — Detects and masks secrets (API keys, tokens, passwords) that may have leaked into prompts
2. **PII scrubbing** — Identifies and removes personally identifiable information
3. **Prompt injection detection** — Scores requests for injection attempts
4. **Context injection defense** — Detects injected instructions in assistant/system context
5. **Hidden content stripping** — Removes invisible Unicode, zero-width characters, and encoded payloads
6. **Policy enforcement** — Checks agent budget, token limits, and allowed providers/models

## Transaction signing

See the [Intents API guide](/docs/guides/intents-api#shroud-tee-signing) for details on TEE-based transaction signing.

Key differences from Vault API signing:

- Private keys are decrypted inside AMD SEV-SNP confidential memory
- Intent validation uses LLM conversation context (velocity, drainer patterns, origin analysis)
- In-memory nonce management per agent session

## Authentication

Shroud authenticates agents using the same JWT as the Vault API. Obtain a token via:

```bash
TOKEN=$(curl -s -X POST https://api.1claw.xyz/v1/auth/agent-token \
  -H "Content-Type: application/json" \
  -d "{\"agent_id\":\"$AGENT_ID\",\"api_key\":\"$API_KEY\"}" | jq -r .access_token)
```

Use this token as `Authorization: Bearer $TOKEN` for all Shroud requests.

## Deployment

Shroud runs on a dedicated GKE cluster with:

- **DNS**: `shroud.1claw.xyz` → A record to GKE static IP
- **TLS**: Google-managed certificate via ManagedCertificate resource
- **Health checks**: HTTP on port 8080 (`/healthz`)
- **Proxy**: Port 8443
- **Metrics**: Port 9090 (Prometheus)

## API parity

Both `api.1claw.xyz` and `shroud.1claw.xyz` serve the complete Intents API. When the dashboard middleware has `SHROUD_URL` configured, all transaction routes are automatically routed through Shroud. Agents can also call Shroud directly for both LLM and transaction operations.
