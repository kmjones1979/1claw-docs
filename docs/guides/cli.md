---
title: CLI
description: Use the 1Claw CLI for CI/CD, DevOps, and servers. Browser-based login, env pull/push/run, and full API coverage.
sidebar_position: 11
---

# 1Claw CLI

The `@1claw/cli` package provides a full-featured command-line interface for 1Claw. It is designed for CI/CD pipelines, DevOps workflows, and server environments.

## Installation

```bash
npm install -g @1claw/cli
```

Or run with npx:

```bash
npx @1claw/cli login
```

## Authentication

### Browser-based login (recommended)

```bash
1claw login
```

This opens your browser to 1claw.xyz where you approve the login. The CLI polls until you confirm. Your token is stored in `~/.config/1claw/`.

### Email/password

```bash
1claw login --email
```

Supports MFA if enabled on your account.

### CI/CD (non-interactive)

Set environment variables — no login needed:

```bash
export ONECLAW_TOKEN="your-jwt"
# or
export ONECLAW_API_KEY="1ck_..."
export ONECLAW_VAULT_ID="your-vault-uuid"   # optional; required for vault-scoped commands
```

## Main commands

| Area | Commands |
|------|----------|
| **Auth** | `login`, `logout`, `whoami` |
| **Vaults** | `vault list`, `vault create`, `vault get`, `vault link`, `vault delete` |
| **Secrets** | `secret list`, `secret get`, `secret set`, `secret delete`, `secret rotate`, `secret describe` |
| **CI/CD** | `env pull`, `env push`, `env run -- <command>` |
| **Agents** | `agent list`, `agent create`, `agent get`, `agent token` |
| **Policies** | `policy list`, `policy create`, `policy delete` |
| **Sharing** | `share create`, `share list`, `share accept`, `share revoke` |
| **Billing** | `billing status`, `billing credits`, `billing usage` |
| **Audit** | `audit list` |
| **MFA** | `mfa status`, `mfa enable`, `mfa disable` |
| **Config** | `config list`, `config set`, `config get` |

## CI/CD examples

### GitHub Actions

```yaml
- name: Deploy with secrets
  env:
    ONECLAW_TOKEN: ${{ secrets.ONECLAW_TOKEN }}
    ONECLAW_VAULT_ID: ${{ secrets.ONECLAW_VAULT_ID }}
  run: |
    npx @1claw/cli env pull -o .env.production
    npm run deploy
```

### Run a command with secrets injected

```bash
1claw env run -- npm start
```

Secrets from the linked (or `ONECLAW_VAULT_ID`) vault are injected as environment variables for the child process.

## Configuration

Config file: `~/.config/1claw/config.json`.

- `api-url` — API base URL (default: `https://api.1claw.xyz`)
- `output-format` — `table`, `json`, or `plain`
- `default-vault` — Default vault ID for commands that need one

Use `1claw config list` and `1claw config set <key> <value>` to view and update.

## Device authorization flow

When you run `1claw login` (without `--email`), the CLI:

1. Calls `POST /v1/auth/device/code` to get a device code and user code.
2. Opens the dashboard at `https://1claw.xyz/cli/verify?code=<user_code>`.
3. You approve the request in the browser (while logged in to 1Claw).
4. The CLI polls `POST /v1/auth/device/token` until the backend marks the code approved, then receives a JWT and stores it.

This flow does not require typing your password in the terminal.

## See also

- [JavaScript SDK](/docs/sdks/javascript) — Programmatic access from Node.js or browsers
- [MCP Server](/docs/mcp/overview) — AI agents accessing secrets via tools
- [Two-factor authentication](/docs/security/two-factor-auth) — Optional 2FA for human logins
