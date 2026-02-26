---
title: Quickstart for agents
description: Exchange an agent API key for a JWT, then list and fetch secrets the agent is allowed to access.
sidebar_position: 1
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Quickstart for agents

An **agent** is registered by a human; when registered, it receives an **API key** (`ocv_...`). The agent (or its runtime) exchanges this key for a short-lived JWT, then calls the same API to list and fetch secrets. Access is enforced by **policies** created by the human.

## 1. Get an agent token

You need the **agent ID** (UUID) and the **API key** that was returned when the agent was registered (or rotated).

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST https://api.1claw.xyz/v1/auth/agent-token \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "ec7e0226-30f0-4dda-b169-f060a3502603",
    "api_key": "ocv_W3_eYj0BSdTjChKwCKRYuZJacmmhVn4ozWIxHV-zlEs"
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
import { createClient } from "@1claw/sdk";

// Agent credentials — the SDK exchanges them for a JWT automatically
// and refreshes it before expiry
const client = createClient({
  baseUrl: "https://api.1claw.xyz",
  agentId: "ec7e0226-30f0-4dda-b169-f060a3502603",
  apiKey: "ocv_W3_eYj0BSdTjChKwCKRYuZJacmmhVn4ozWIxHV-zlEs",
});
```

</TabItem>
</Tabs>

**Response:**

```json
{
  "access_token": "eyJhbGciOiJFZERTQSIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

Use this as `Authorization: Bearer <access_token>` for subsequent requests. Token is short-lived (e.g. 1 hour).

## 2. List secrets you can access

With the agent JWT you can list secrets in a vault (metadata only). You only see secrets for vaults and paths your policies allow.

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
export TOKEN="<agent access_token>"
export VAULT_ID="ae370174-9aee-4b02-ba7c-d1519930c709"

curl -s "https://api.1claw.xyz/v1/vaults/$VAULT_ID/secrets" \
  -H "Authorization: Bearer $TOKEN"
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const VAULT_ID = "ae370174-9aee-4b02-ba7c-d1519930c709";

const { data } = await client.secrets.list(VAULT_ID);
for (const s of data.secrets) {
  console.log(`${s.path} (${s.type}, v${s.version})`);
}
```

</TabItem>
</Tabs>

**Response:** `{ "secrets": [ { "id", "path", "type", "version", "metadata", "created_at", "expires_at" }, ... ] }`

## 3. Fetch a secret value

Request a secret by vault ID and path. The server checks policy; if the agent has read access, it decrypts and returns the value.

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -s "https://api.1claw.xyz/v1/vaults/$VAULT_ID/secrets/api-keys/openai" \
  -H "Authorization: Bearer $TOKEN"
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data: secret } = await client.secrets.get(VAULT_ID, "api-keys/openai");
// Use the value for the intended call — don't log or persist it
console.log(`Retrieved ${secret.path} (${secret.type}, v${secret.version})`);
```

</TabItem>
</Tabs>

**Response (200):** Includes `value` (plaintext) and metadata. Use the value only for the intended call; don't log or persist it.

If the agent has no read permission for that path, or the secret is expired/deleted, you get **403** or **404/410**.

## Important

- **Store the API key securely** — In the agent's config or secrets store, not in code or prompts.
- **Refresh the JWT** before it expires — Call `POST /v1/auth/agent-token` again when `expires_in` has passed. The TypeScript SDK handles this automatically when you pass `agentId` + `apiKey`.
- **Same API as humans** — Same base URL and paths; only the way you get the JWT (agent-token vs email/password or Google) and the permissions (policies) differ.

## Next steps

- [Agent API overview](/docs/agent-api/overview) — Auth and endpoints in one place.
- [Give an agent access](/docs/guides/give-agent-access) — How a human registers an agent and creates a policy.
- [Fetch secret](/docs/agent-api/fetch-secret) — Full request/response and errors.
