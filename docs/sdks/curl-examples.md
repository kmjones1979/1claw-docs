---
title: curl examples
description: Example curl commands for 1claw auth, vaults, secrets, and agents; works with any environment that has curl.
sidebar_position: 3
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# curl examples

Base URL used: `https://api.1claw.xyz`. Replace with your Cloud Run URL if different.

## Human: get JWT

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -s -X POST https://api.1claw.xyz/v1/auth/token \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"your-password"}'
# Save access_token as $TOKEN
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
import { createClient } from "@1claw/sdk";
const client = createClient({ baseUrl: "https://api.1claw.xyz" });
await client.auth.login({ email: "you@example.com", password: "your-password" });
```

</TabItem>
</Tabs>

## Human: create vault and secret

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
export TOKEN="<access_token>"

# Create vault
curl -s -X POST https://api.1claw.xyz/v1/vaults \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"My Vault","description":"Secrets"}'

# Store secret (use vault id from response)
export VAULT_ID="<vault_id>"
curl -s -X PUT "https://api.1claw.xyz/v1/vaults/$VAULT_ID/secrets/api-keys/openai" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"api_key","value":"sk-proj-..."}'

# Read secret
curl -s "https://api.1claw.xyz/v1/vaults/$VAULT_ID/secrets/api-keys/openai" \
  -H "Authorization: Bearer $TOKEN"
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const client = createClient({
  baseUrl: "https://api.1claw.xyz",
  apiKey: process.env.ONECLAW_API_KEY,
});

const { data: vault } = await client.vault.create({
  name: "My Vault",
  description: "Secrets",
});

await client.secrets.set(vault.id, "api-keys/openai", "sk-proj-...", {
  type: "api_key",
});

const { data: secret } = await client.secrets.get(vault.id, "api-keys/openai");
console.log(secret.value);
```

</TabItem>
</Tabs>

## Agent: get JWT and read secret

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -s -X POST https://api.1claw.xyz/v1/auth/agent-token \
  -H "Content-Type: application/json" \
  -d '{"agent_id":"<uuid>","api_key":"ocv_..."}'
# Save access_token

export TOKEN="<agent_access_token>"
export VAULT_ID="<vault_id>"
curl -s "https://api.1claw.xyz/v1/vaults/$VAULT_ID/secrets/api-keys/openai" \
  -H "Authorization: Bearer $TOKEN"
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const agentClient = createClient({
  baseUrl: "https://api.1claw.xyz",
  agentId: "<uuid>",
  apiKey: "ocv_...",
});

const { data: secret } = await agentClient.secrets.get(VAULT_ID, "api-keys/openai");
```

</TabItem>
</Tabs>

## List secrets (metadata only)

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -s "https://api.1claw.xyz/v1/vaults/$VAULT_ID/secrets" \
  -H "Authorization: Bearer $TOKEN"
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data } = await client.secrets.list(VAULT_ID);
```

</TabItem>
</Tabs>

## Create policy (grant agent read)

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -s -X POST "https://api.1claw.xyz/v1/vaults/$VAULT_ID/policies" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "secret_path_pattern": "**",
    "principal_type": "agent",
    "principal_id": "<agent_uuid>",
    "permissions": ["read"]
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
await client.access.grantAgent({
  vault_id: VAULT_ID,
  secret_path_pattern: "**",
  principal_id: "<agent_uuid>",
  permissions: ["read"],
});
```

</TabItem>
</Tabs>
