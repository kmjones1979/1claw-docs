---
title: Using 1claw with OpenClaw
description: Add the 1claw skill to your OpenClaw gateway so your AI agent can store and retrieve secrets from the vault.
sidebar_position: 7
---

# Using 1claw with OpenClaw

[OpenClaw](https://docs.openclaw.ai) is a self-hosted gateway that connects chat apps (WhatsApp, Telegram, Discord, iMessage, and more) to AI coding agents. The **1claw skill** lets your OpenClaw agent use the 1Claw vault to store, retrieve, rotate, and share secrets — so the agent can use API keys and credentials at runtime without them ever living in chat or context.

## What you get

With the 1claw skill installed, your OpenClaw agent can:

- List and fetch secrets from a 1Claw vault by path
- Store new secrets or rotate existing ones
- Share secrets with users or other agents
- Create vaults and grant access (subject to policies)
- Sign or simulate EVM transactions via the Intents API (if enabled for the agent)

Secrets are encrypted in the vault and fetched just-in-time; they are not persisted in the conversation.

## Setup options

There are two ways to connect your OpenClaw agent to 1Claw:

### Option A: Self-enrollment (recommended)

The agent can register itself. Run this from your deployment script or during `clawhub install`:

```bash
curl -s -X POST https://api.1claw.xyz/v1/agents/enroll \
  -H "Content-Type: application/json" \
  -d '{"name":"my-openclaw-agent","human_email":"you@example.com"}'
```

1Claw looks up your account by email, creates the agent, and **emails the credentials** (Agent ID + API key) to you. The API key is not returned in the response — only the human receives it.

Once you get the email, configure the credentials in your OpenClaw environment (see [Configure credentials](#configure-credentials) below), then grant the agent access to a vault from the [dashboard](https://1claw.xyz/agents).

### Option B: Manual registration

1. **1Claw account** — Sign up at [1claw.xyz](https://1claw.xyz).
2. **A vault** — Create a vault in the dashboard (or via API).
3. **An agent** — Register an agent in the dashboard (Vaults → select vault → Agents, or from the Agents page). Copy the **Agent ID** and the one-time **API key** (`ocv_...`); store the key securely.
4. **Access for the agent** — Grant the agent read (and optionally write) access to the vault via a policy. See [Give an agent access](/docs/guides/give-agent-access).

## Install the 1claw skill

Install the skill using the ClawHub CLI:

```bash
clawhub install 1claw
```

This installs the 1claw skill so your OpenClaw gateway can use 1Claw for secret management.

## Configure credentials

The 1claw skill expects these environment variables (or equivalent in your OpenClaw config):

| Variable | Description |
| -------- | ----------- |
| `ONECLAW_AGENT_ID` | Your 1Claw agent's UUID (from the enrollment email or dashboard). |
| `ONECLAW_AGENT_API_KEY` | The agent's API key (`ocv_...`). Sent to the human's email during enrollment, or shown once during manual registration. |
| `ONECLAW_VAULT_ID` | The vault UUID the agent will read from and write to. |

Set them in your OpenClaw environment or in the config your gateway uses when running the 1claw MCP server (the skill uses the [1Claw MCP server](https://www.npmjs.com/package/@1claw/mcp) under the hood).

## How it works

The 1claw skill teaches your agent how to call 1Claw's MCP tools (e.g. `list_secrets`, `get_secret`, `put_secret`, `create_vault`, `share_secret`). When the agent needs a credential, it calls the tool; the MCP server authenticates to the 1Claw API with your agent credentials and returns the secret value. The agent uses it for the task and does not store it in long-term context.

- **MCP server** — The skill uses `@1claw/mcp`. You can run it as a local process (stdio) or use the hosted MCP at `https://mcp.1claw.xyz/mcp` if your OpenClaw setup supports remote MCP.
- **Permissions** — The agent's 1Claw policies control what it can read and write. Restrict by path pattern (e.g. `prod/*` only) and use the dashboard to revoke or rotate access anytime.
- **Sharing with your human** — The agent can share secrets back to the human who registered it using `recipient_type: "creator"` in the `share_secret` tool. The human will see the shared secret in their dashboard under Sharing → Inbound.

## Next steps

- [OpenClaw documentation](https://docs.openclaw.ai) — Gateway setup, channels, and configuration.
- [MCP Server](/docs/mcp/overview) — 1Claw MCP tools and setup in detail.
- [Give an agent access](/docs/guides/give-agent-access) — Create policies so your agent can access the right vault and paths.
- [Skill repository](https://github.com/1clawAI/1claw/tree/main/skill) — Skill source (SKILL.md, EXAMPLES.md, CONFIG.md) and validation script.
