---
title: Crypto Transaction Proxy
description: Let agents sign and broadcast blockchain transactions without ever seeing private keys. Includes the full list of supported chains.
sidebar_position: 3
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Crypto Transaction Proxy

The Crypto Transaction Proxy lets an agent submit on-chain transactions — transfers, swaps, contract calls — while **never having access to the raw private key**. The server signs the transaction using keys stored in the vault and broadcasts it through a dedicated RPC for the target chain.

## How it works

```
Agent                       1claw Vault                  Blockchain
  │                             │                            │
  │  POST /v1/agents/:id/       │                            │
  │    transactions             │                            │
  │  { chain_id, to, value,     │                            │
  │    data, secret_path }      │                            │
  │ ─────────────────────────►  │                            │
  │                             │ 1. Decrypt private key     │
  │                             │    from vault via HSM      │
  │                             │ 2. Build & sign tx         │
  │                             │ 3. Broadcast via RPC  ───► │
  │                             │                            │
  │  ◄───────────────────────── │  tx_hash, status           │
  │  { tx_id, tx_hash, status } │                            │
```

1. The agent calls `POST /v1/agents/:agent_id/transactions` with the chain, recipient, value, calldata, and the vault path to the signing key.
2. The vault decrypts the private key inside the HSM boundary, constructs and signs the transaction, and broadcasts it to the chain's RPC endpoint.
3. The agent receives a `tx_id` and `tx_hash` — it never sees the raw key material.

## Enabling the proxy

Set `crypto_proxy_enabled: true` when registering or updating an agent:

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST "https://api.1claw.xyz/v1/agents" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "DeFi Bot",
    "crypto_proxy_enabled": true
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
import { createClient } from "@1claw/sdk";

const client = createClient({
  baseUrl: "https://api.1claw.xyz",
  apiKey: process.env.ONECLAW_API_KEY,
});

const { data } = await client.agents.create({
  name: "DeFi Bot",
  crypto_proxy_enabled: true,
});
```

</TabItem>
</Tabs>

### What changes when enabled

| Behaviour                        | `crypto_proxy_enabled: false` | `crypto_proxy_enabled: true` |
| -------------------------------- | ----------------------------- | ---------------------------- |
| Read `api_key`, `password`, etc. | Allowed                       | Allowed                      |
| Read `private_key` or `ssh_key`  | Allowed                       | **Blocked (403)**            |
| Submit proxy transactions        | Not available                 | Allowed                      |
| Audit trail per transaction      | N/A                           | Full trace with `tx_id`      |

The enforcement is two-sided: the flag both **grants** access to the transaction endpoints and **blocks** direct reads of signing keys through the standard secrets endpoint. This guarantees the agent can only use keys through the proxy.

## Submitting a transaction

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST "https://api.1claw.xyz/v1/agents/$AGENT_ID/transactions" \
  -H "Authorization: Bearer $AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "chain_id": 1,
    "to": "0xRecipientAddress",
    "value": "1000000000000000000",
    "data": "0x",
    "secret_path": "wallets/hot-wallet"
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data: tx } = await client.agents.submitTransaction(agentId, {
  chain_id: 1,
  to: "0xRecipientAddress",
  value: "1000000000000000000",
  data: "0x",
  secret_path: "wallets/hot-wallet",
});
```

</TabItem>
</Tabs>

### Response

```json
{
    "tx_id": "a7e2c...",
    "tx_hash": "0xabc123...",
    "chain_id": 1,
    "status": "submitted"
}
```

## Querying transactions

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
# List all transactions for this agent
curl "https://api.1claw.xyz/v1/agents/$AGENT_ID/transactions" \
  -H "Authorization: Bearer $AGENT_TOKEN"

# Get a specific transaction
curl "https://api.1claw.xyz/v1/agents/$AGENT_ID/transactions/$TX_ID" \
  -H "Authorization: Bearer $AGENT_TOKEN"
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
// List transactions
const { data: txList } = await client.agents.listTransactions(agentId);

// Get transaction
const { data: tx } = await client.agents.getTransaction(agentId, txId);
```

</TabItem>
</Tabs>

## Transaction simulation (Tenderly) {#simulation}

Every transaction can be simulated before signing. Simulation executes the full transaction against the current chain state in a sandboxed environment, returning decoded traces, balance changes, gas estimates, and human-readable error messages — without consuming real gas.

### Standalone simulation

Call the simulate endpoint to preview a transaction without committing:

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST "https://api.1claw.xyz/v1/agents/$AGENT_ID/transactions/simulate" \
  -H "Authorization: Bearer $AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "chain": "base",
    "to": "0xRecipientAddress",
    "value": "0.5",
    "data": "0x",
    "signing_key_path": "wallets/hot-wallet"
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data: sim } = await client.agents.simulateTransaction(agentId, {
  chain: "base",
  to: "0xRecipientAddress",
  value: "0.5",
  data: "0x",
  signing_key_path: "wallets/hot-wallet",
});
```

</TabItem>
</Tabs>

The response includes:

```json
{
  "simulation_id": "sim_a7e2c...",
  "status": "success",
  "gas_used": 21000,
  "balance_changes": [
    { "address": "0xSender...", "token": "ETH", "before": "2.5", "after": "1.99", "change": "-0.51" },
    { "address": "0xRecipient...", "token": "ETH", "before": "0.0", "after": "0.5", "change": "+0.5" }
  ],
  "tenderly_dashboard_url": "https://dashboard.tenderly.co/..."
}
```

### Simulate-then-sign (single call)

Add `"simulate_first": true` to the standard transaction submission. The server simulates first; if the simulation reverts, it returns HTTP 422 and does **not** sign or broadcast:

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST "https://api.1claw.xyz/v1/agents/$AGENT_ID/transactions" \
  -H "Authorization: Bearer $AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "chain": "base",
    "to": "0xRecipientAddress",
    "value": "0.5",
    "simulate_first": true
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data: tx } = await client.agents.submitTransaction(agentId, {
  chain: "base",
  to: "0xRecipientAddress",
  value: "0.5",
  simulate_first: true,
});
```

</TabItem>
</Tabs>

### Bundle simulation

Simulate multiple transactions sequentially (e.g. ERC-20 approve followed by a swap):

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST "https://api.1claw.xyz/v1/agents/$AGENT_ID/transactions/simulate-bundle" \
  -H "Authorization: Bearer $AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "transactions": [
      { "chain": "base", "to": "0xToken", "value": "0", "data": "0xapprove..." },
      { "chain": "base", "to": "0xRouter", "value": "0", "data": "0xswap..." }
    ]
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data: bundle } = await client.agents.simulateBundle(agentId, {
  transactions: [
    { chain: "base", to: "0xToken", value: "0", data: "0xapprove..." },
    { chain: "base", to: "0xRouter", value: "0", data: "0xswap..." },
  ],
});
```

</TabItem>
</Tabs>

### Enforcing simulation

Org admins can require simulation for all agent transactions by setting the `crypto_proxy.require_simulation` org setting to `"true"` via `PUT /v1/admin/settings/crypto_proxy.require_simulation`. When enabled, any transaction submitted without `simulate_first: true` will be automatically simulated, and reverts will block signing.

### EIP-1559 (Type 2) transactions

Set `max_fee_per_gas` and `max_priority_fee_per_gas` instead of `gas_price` to use EIP-1559 fee mode:

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST "https://api.1claw.xyz/v1/agents/$AGENT_ID/transactions" \
  -H "Authorization: Bearer $AGENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "chain": "base",
    "to": "0xRecipientAddress",
    "value": "0.1",
    "max_fee_per_gas": "30000000000",
    "max_priority_fee_per_gas": "1500000000",
    "simulate_first": true
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data: tx } = await client.agents.submitTransaction(agentId, {
  chain: "base",
  to: "0xRecipientAddress",
  value: "0.1",
  max_fee_per_gas: "30000000000",
  max_priority_fee_per_gas: "1500000000",
  simulate_first: true,
});
```

</TabItem>
</Tabs>

## MCP tools

The MCP server provides two transaction tools:

**`simulate_transaction`** — simulate without signing:
```
Tool: simulate_transaction
Args:
  chain: "base"
  to: "0xRecipientAddress"
  value: "0.5"
  signing_key_path: "wallets/hot-wallet"
```

**`submit_transaction`** — sign and broadcast (simulation on by default):
```
Tool: submit_transaction
Args:
  chain: "base"
  to: "0xRecipientAddress"
  value: "0.5"
  signing_key_path: "wallets/hot-wallet"
  simulate_first: true
```

---

## Supported chains {#supported-chains}

The proxy can broadcast transactions to any chain in the registry. All mainnet chains below are configured with dedicated dRPC endpoints for reliable transaction delivery.

:::tip Querying chains via API
You can always fetch the live list with `GET /v1/chains`. The response includes `chain_id`, `rpc_url`, `explorer_url`, and `native_currency` for every chain.
:::

### Mainnet chains (28)

| Chain             | Chain ID | Native token | Explorer                                                           |
| ----------------- | -------- | ------------ | ------------------------------------------------------------------ |
| Ethereum          | 1        | ETH          | [etherscan.io](https://etherscan.io)                               |
| Optimism          | 10       | ETH          | [optimistic.etherscan.io](https://optimistic.etherscan.io)         |
| Cronos            | 25       | CRO          | [cronoscan.com](https://cronoscan.com)                             |
| BNB Smart Chain   | 56       | BNB          | [bscscan.com](https://bscscan.com)                                 |
| Gnosis            | 100      | xDAI         | [gnosisscan.io](https://gnosisscan.io)                             |
| Polygon           | 137      | POL          | [polygonscan.com](https://polygonscan.com)                         |
| Sonic             | 146      | S            | [sonicscan.org](https://sonicscan.org)                             |
| Fantom            | 250      | FTM          | [ftmscan.com](https://ftmscan.com)                                 |
| zkSync Era        | 324      | ETH          | [explorer.zksync.io](https://explorer.zksync.io)                   |
| World Chain       | 480      | ETH          | [worldscan.org](https://worldscan.org)                             |
| Metis             | 1088     | METIS        | [andromeda-explorer.metis.io](https://andromeda-explorer.metis.io) |
| Polygon zkEVM     | 1101     | ETH          | [zkevm.polygonscan.com](https://zkevm.polygonscan.com)             |
| Moonbeam          | 1284     | GLMR         | [moonscan.io](https://moonscan.io)                                 |
| Sei               | 1329     | SEI          | [seitrace.com](https://seitrace.com)                               |
| Mantle            | 5000     | MNT          | [mantlescan.xyz](https://mantlescan.xyz)                           |
| Kaia              | 8217     | KAIA         | [kaiascan.io](https://kaiascan.io)                                 |
| Base              | 8453     | ETH          | [basescan.org](https://basescan.org)                               |
| Mode              | 34443    | ETH          | [modescan.io](https://modescan.io)                                 |
| Arbitrum One      | 42161    | ETH          | [arbiscan.io](https://arbiscan.io)                                 |
| Arbitrum Nova     | 42170    | ETH          | [nova.arbiscan.io](https://nova.arbiscan.io)                       |
| Celo              | 42220    | CELO         | [celoscan.io](https://celoscan.io)                                 |
| Avalanche C-Chain | 43114    | AVAX         | [snowtrace.io](https://snowtrace.io)                               |
| Linea             | 59144    | ETH          | [lineascan.build](https://lineascan.build)                         |
| Berachain         | 80094    | BERA         | [berascan.com](https://berascan.com)                               |
| Blast             | 81457    | ETH          | [blastscan.io](https://blastscan.io)                               |
| Taiko             | 167000   | ETH          | [taikoscan.io](https://taikoscan.io)                               |
| Scroll            | 534352   | ETH          | [scrollscan.com](https://scrollscan.com)                           |
| Zora              | 7777777  | ETH          | [explorer.zora.energy](https://explorer.zora.energy)               |

### Testnet chains

| Chain        | Chain ID | Native token |
| ------------ | -------- | ------------ |
| Sepolia      | 11155111 | ETH          |
| Base Sepolia | 84532    | ETH          |

### Adding a chain

Admins can add new chains via the admin API:

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X POST "https://api.1claw.xyz/v1/admin/chains" \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-chain",
    "display_name": "My Chain",
    "chain_id": 12345,
    "rpc_url": "https://rpc.mychain.io",
    "explorer_url": "https://explorer.mychain.io",
    "native_currency": "MCH"
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
// Admin chain management requires direct API calls
const response = await fetch("https://api.1claw.xyz/v1/admin/chains", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${adminToken}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    name: "my-chain",
    display_name: "My Chain",
    chain_id: 12345,
    rpc_url: "https://rpc.mychain.io",
    explorer_url: "https://explorer.mychain.io",
    native_currency: "MCH",
  }),
});
```

</TabItem>
</Tabs>

See the [Admin API reference](/docs/reference/api-reference#admin) for update and delete endpoints.

---

## Transaction guardrails

Per-agent controls can be set when registering or updating an agent to limit what transactions the proxy will sign:

| Field | Type | Description |
| ----- | ---- | ----------- |
| `tx_allowed_chains` | `string[]` | Restrict to specific chain names (e.g. `["ethereum", "base"]`). Empty = all chains allowed. |
| `tx_to_allowlist` | `string[]` | Restrict recipient addresses. Empty = any address allowed. |
| `tx_max_value_eth` | `string` | Maximum value per transaction in ETH (e.g. `"1.0"`). Null = no per-tx limit. |
| `tx_daily_limit_eth` | `string` | Rolling 24-hour spend limit in ETH. Null = no daily limit. |

<Tabs groupId="code-examples">
<TabItem value="curl" label="curl">

```bash
curl -X PATCH "https://api.1claw.xyz/v1/agents/$AGENT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tx_allowed_chains": ["ethereum", "base"],
    "tx_to_allowlist": ["0xSafeAddress1", "0xSafeAddress2"],
    "tx_max_value_eth": "0.5",
    "tx_daily_limit_eth": "5.0"
  }'
```

</TabItem>
<TabItem value="typescript" label="TypeScript">

```typescript
const { data: agent } = await client.agents.update(agentId, {
  tx_allowed_chains: ["ethereum", "base"],
  tx_to_allowlist: ["0xSafeAddress1", "0xSafeAddress2"],
  tx_max_value_eth: "0.5",
  tx_daily_limit_eth: "5.0",
});
```

</TabItem>
</Tabs>

When a transaction violates any guardrail, the proxy returns **403 Forbidden** with a descriptive `detail` message.

---

## Security model

- **Keys never leave the HSM boundary** — the vault decrypts the key, signs the transaction, and zeroes the memory. The plaintext key is never returned to the caller.
- **Full audit trail** — every transaction is logged with the agent ID, chain, recipient, value, and resulting `tx_hash`.
- **Policy enforcement** — the agent still needs a policy granting access to the vault path that holds the signing key. The proxy doesn't bypass access control.
- **Transaction guardrails** — per-agent chain allowlists, recipient allowlists, per-tx caps, and daily spend limits enforced server-side before signing.
- **Rate limiting** — standard rate limits apply to transaction endpoints.

## Best practices

1. **One key per agent** — give each agent its own signing key in its own vault path so you can revoke independently.
2. **Set `expires_at`** — register agents with an expiry so leaked API keys have a bounded blast radius.
3. **Use scoped policies** — grant the agent access only to the specific vault path containing its signing key, not the entire vault.
4. **Monitor transactions** — query `GET /v1/agents/:id/transactions` regularly or set up audit webhooks.
5. **Use testnets first** — store a testnet key (Sepolia, Base Sepolia) and verify the flow before moving to mainnet.
