---
title: Parts of 1claw
description: The API, Dashboard, MCP server, CLI, and SDK — what each is for and when to use it.
sidebar_position: 4
---

# Parts of 1claw

1claw is one product with several ways to use it. Pick the right one for your role and environment.

## Vault API

**What it is:** The REST API that everything else talks to. Base URL: `https://api.1claw.xyz`. Handles auth, vaults, secrets, policies, agents, sharing, billing, and audit.

**When to use it:** When you're integrating 1claw into your own app, script, or service. You send HTTP requests with a Bearer token (JWT or API key). Use the [API reference](/docs/reference/api-reference) and the [OpenAPI spec](https://github.com/1clawAI/1claw-openapi-spec) for full details.

**You need:** A user JWT (from login or device flow) or a personal API key (`1ck_`), or an agent JWT (from `POST /v1/auth/agent-token` with an agent API key `ocv_`).

---

## Dashboard

**What it is:** The web UI at [1claw.xyz](https://1claw.xyz). Sign in with email/password or Google, then manage vaults, secrets, agents, policies, sharing, audit log, API keys, billing, and team.

**When to use it:** For day-to-day setup and management — creating vaults, storing secrets, registering agents, granting access, viewing audit logs, upgrading or managing billing. Most humans use the dashboard as the main way to interact with 1claw.

**You need:** An account (sign up at 1claw.xyz). Optional: MFA and API keys for extra security.

---

## MCP Server

**What it is:** A [Model Context Protocol](https://modelcontextprotocol.io) server that exposes 1claw as tools (e.g. `list_secrets`, `get_secret`, `put_secret`, `create_vault`, `share_secret`, `simulate_transaction`, `submit_transaction`). AI assistants (Claude, Cursor, GPT, etc.) call these tools so they can use secrets at runtime without you pasting credentials.

**When to use it:** When you want an AI agent to read or write secrets, create vaults, or sign transactions through 1claw. Configure your AI tool to use the 1claw MCP server (hosted at `mcp.1claw.xyz` or run locally). The agent uses its own API key; you control what it can do via policies.

**You need:** An agent registered in the dashboard (or via API), with policies that grant the agent access to the vaults and paths it needs. See [MCP Setup](/docs/mcp/setup) and [Give an agent access](/docs/guides/give-agent-access).

---

## CLI

**What it is:** A command-line tool (`@1claw/cli`) for CI/CD, servers, and local scripts. Log in via browser (device flow) or with email/password, then run commands for vaults, secrets, agents, policies, and shares. Can inject secrets into env or run a command with secrets loaded.

**When to use it:** For scripts, cron jobs, deploy pipelines, or any environment where you want to pull secrets or run a process with secrets without building API calls yourself. Use `1claw env run -- your-command` to run a command with vault secrets as environment variables.

**You need:** Node.js 20+; install with `npm i -g @1claw/cli`. Then `1claw login` (device flow) or set `ONECLAW_TOKEN` / `ONECLAW_API_KEY`. See [CLI guide](/docs/guides/cli).

---

## SDK

**What it is:** TypeScript/JavaScript client (`@1claw/sdk`) that wraps the REST API. Methods for auth, vaults, secrets, policies, agents, sharing, billing, audit, chains, and (for agents) transaction simulation and submission.

**When to use it:** When you're writing an app or service in Node/TS and want typed, high-level calls instead of raw `fetch`. Same auth as the API (user or agent token, or API key). Supports x402 payment flow if you need to pay per request.

**You need:** `npm i @1claw/sdk` or `pnpm add @1claw/sdk`. Configure with `baseUrl` and either a token or credentials to obtain one. See [JavaScript / TypeScript SDK](/docs/sdks/javascript).

---

## How they fit together

| You are…                    | Best starting point        |
| --------------------------- | -------------------------- |
| A human setting things up   | **Dashboard**              |
| A human in a script/CI      | **CLI** or **SDK**         |
| An AI agent (MCP client)    | **MCP Server** (hosted or local) |
| A custom app or backend     | **SDK** or **Vault API**   |

All of them talk to the same Vault API and the same data. Create a vault in the dashboard, store a secret via the API or CLI, and an agent using MCP can read it — as long as you’ve granted that agent access with a policy.

For definitions of terms (vault, secret, policy, agent, etc.), see the [Glossary](/docs/reference/glossary). For common errors and fixes, see [Troubleshooting](/docs/guides/troubleshooting).
