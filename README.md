# Dynatrace MCP + Claude Code Setup Guide

Connect your Dynatrace environment to Claude Code CLI for live DQL queries, problem investigation, log analysis, and AI-assisted observability workflows.

---

## Table of Contents

1. [Why Claude.ai Doesn't Work (and Claude Code Does)](#1-why-claudeai-doesnt-work-and-claude-code-does)
2. [What You're Installing](#2-what-youre-installing)
3. [Prerequisites](#3-prerequisites)
4. [Step 1 — Create a Platform Token](#4-step-1--create-a-platform-token)
5. [Step 2 — Find Your MCP Gateway URL](#5-step-2--find-your-mcp-gateway-url)
6. [Step 3 — Register the MCP Server in Claude Code](#6-step-3--register-the-mcp-server-in-claude-code)
7. [Step 4 — Install the Domain Skills Plugin](#7-step-4--install-the-domain-skills-plugin)
8. [Step 5 — Verify & Test](#8-step-5--verify--test)
9. [What You Can Do Now](#9-what-you-can-do-now)
10. [Tips & Gotchas](#10-tips--gotchas)
11. [Further Reading](#11-further-reading)

---

## 1. Why Claude.ai Doesn't Work (and Claude Code Does)

If you've tried adding the Dynatrace MCP server through claude.ai and seen "Needs authentication" — that's expected, and here's why.

### The short answer: authentication protocol mismatch

| | Claude Code CLI | claude.ai (web) |
|---|---|---|
| MCP auth method | Bearer token (any header) | OAuth 2.0 only |
| Dynatrace auth method | Platform Token (Bearer) | Platform Token (Bearer) |
| Works? | **Yes** | **No — not yet** |

**Claude Code CLI** supports HTTP MCP servers with arbitrary `Authorization` headers. You can pass a Dynatrace Platform Token directly as a Bearer token and it connects immediately.

**Claude.ai** requires MCP servers to implement OAuth 2.0 with a public discovery endpoint. Dynatrace's MCP gateway currently uses Platform Token (Bearer) authentication, not OAuth — so claude.ai can't complete the auth handshake.

> Until Dynatrace adds OAuth 2.0 support to the MCP gateway, or Anthropic adds user-supplied Bearer token support to claude.ai MCP, this only works in **Claude Code CLI**.

---

## 2. What You're Installing

There are two separate components, and both are needed for the full experience:

```
Claude Code CLI
    │
    ├── HTTP MCP server: dynatrace-mcp
    │   ├── Cloud-hosted on your Dynatrace tenant
    │   ├── Auth: Platform Token (Bearer)
    │   └── Provides: live DQL execution, problems, logs, metrics,
    │                 traces, K8s events, security vulnerabilities, Davis AI
    │
    └── Plugin: dynatrace@dynatrace-for-ai
        ├── Installed locally via claude plugin install
        └── Provides: Dynatrace domain skills — DQL patterns,
                      observability workflows, query guidance
```

The **MCP server** gives Claude live access to your tenant data. The **plugin** gives Claude Dynatrace-specific knowledge and query patterns so it can reason about that data intelligently.

---

## 3. Prerequisites

- A **Dynatrace SaaS environment** (any production or trial tenant)
- **Node.js 18+** installed ([nodejs.org](https://nodejs.org))
- **Claude Code CLI** installed with an active Claude Pro, Team, or Enterprise subscription:
  ```bash
  npm install -g @anthropic-ai/claude-code
  ```
- A terminal and basic comfort running shell commands

---

## 4. Step 1 — Create a Platform Token

1. In your Dynatrace tenant, navigate to **Settings > Access tokens** (or search "Access tokens" in the main menu)
2. Click **Generate new token**
3. Give it a descriptive name (e.g., `claude-code-mcp`)
4. Add the following token scopes:

   **MCP Gateway (required — without these the server won't connect)**
   | Scope | Purpose |
   |-------|---------|
   | `mcp-gateway:servers:invoke` | Invoke the MCP server |
   | `mcp-gateway:servers:read` | Read MCP server metadata |

   **Storage / Observability Data**
   | Scope | Purpose |
   |-------|---------|
   | `storage:metrics:read` | Metric queries |
   | `storage:logs:read` | Log queries |
   | `storage:events:read` | Event queries |
   | `storage:bizevents:read` | Business event queries |
   | `storage:spans:read` | Trace/span queries |
   | `storage:entities:read` | Entity data |
   | `events:read` | Platform events |
   | `document:documents:read` | Notebook/document access |

   **Davis AI (required for AI-powered analysis tools)**
   | Scope | Purpose |
   |-------|---------|
   | `davis:analyzers:execute` | Run Davis analyzers (anomaly detection, forecasting) |
   | `davis:analyzers:read` | Read Davis analyzer results |
   | `davis-copilot:conversations:execute` | Davis Copilot conversations |
   | `davis-copilot:dql2nl:execute` | Translate DQL to natural language |
   | `davis-copilot:nl2dql:execute` | Translate natural language to DQL |
   | `davis-copilot:document-search:execute` | Davis document search |

   > **Note:** Always verify the complete and current scope list against the [official Dynatrace MCP Server documentation](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp) — scopes may change with platform updates.

5. Click **Generate token** and copy it immediately — it won't be shown again
6. The token format looks like: `dt0s16.XXXX.XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`

> **Security tip:** Store the token in a password manager or your OS keyring. Never paste it directly into shell commands — use environment variables instead (shown in Step 3).

---

## 5. Step 2 — Find Your MCP Gateway URL

Your Dynatrace tenant exposes an MCP gateway at a fixed path. The URL is derived from your tenant's base URL:

```
https://<your-env-id>.apps.dynatracelabs.com/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp
```

**To find your `env-id`:** look at your browser's URL bar when logged into Dynatrace. It's the subdomain before `.apps.dynatracelabs.com`.

Example: if your tenant URL is `https://abc12345.apps.dynatracelabs.com`, then your MCP gateway URL is:
```
https://abc12345.apps.dynatracelabs.com/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp
```

---

## 6. Step 3 — Register the MCP Server in Claude Code

Set your credentials as environment variables first to keep them out of your shell history:

```bash
export DT_PLATFORM_TOKEN="dt0s16.YOUR_TOKEN_HERE"
export DT_ENV_URL="https://your-env-id.apps.dynatracelabs.com"
```

Then register the MCP server:

```bash
claude mcp add dynatrace-mcp \
  --transport http \
  --scope user \
  "${DT_ENV_URL}/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp" \
  --header "Authorization: Bearer ${DT_PLATFORM_TOKEN}"
```

- `--transport http` — Dynatrace MCP is a remote HTTP server, not a local process
- `--scope user` — registers at user scope, available in **all** your Claude Code projects (omit for local/project-only scope)
- `--header` — passes your Platform Token as a Bearer token on every request

---

## 7. Step 4 — Install the Domain Skills Plugin

The plugin adds Dynatrace-specific knowledge and reasoning skills. It's separate from the MCP server and doesn't require tenant credentials.

**First, register the Dynatrace plugin marketplace:**

```bash
claude plugin marketplace add dynatrace-for-ai --source github:dynatrace/dynatrace-for-ai
```

**Then install the plugin:**

```bash
claude plugin install dynatrace@dynatrace-for-ai
```

This installs at user scope and provides skills like DQL query patterns, observability troubleshooting workflows, and Dynatrace-specific reasoning.

---

## 8. Step 5 — Verify & Test

**Check that the MCP server connected:**

```bash
claude mcp list
```

You should see:
```
dynatrace-mcp: https://your-env.apps.dynatracelabs.com/... (HTTP) - ✓ Connected
```

If it shows `! Needs authentication`, double-check that your Platform Token has the correct scopes and hasn't expired.

**Test it in Claude Code:**

Start Claude Code in any directory:
```bash
claude
```

Then try a prompt like:
> "Fetch the last 10 log entries from my Dynatrace environment using DQL."

Or to test problem data:
> "Are there any open problems in my Dynatrace environment right now?"

A successful response means live data is flowing from your tenant through the MCP server to Claude.

---

## 9. What You Can Do Now

Once connected, Claude Code can work directly with your tenant data. Capabilities include:

| What you can ask | Underlying tool |
|---|---|
| Run any DQL query | `execute-dql` |
| Query open/recent problems | `query-problems` |
| Look up entities by name or ID | `get-entity-name`, `get-entity-id` |
| Fetch security vulnerabilities | `get-vulnerabilities` |
| Get Kubernetes cluster events | `get-events-for-kubernetes-cluster` |
| Forecast metric trends | `timeseries-forecast` |
| Detect anomalies in timeseries | `timeseries-novelty-detection`, `adaptive-anomaly-detector` |
| Ask questions about Dynatrace docs | `ask-dynatrace-docs` |
| Find troubleshooting guides | `find-troubleshooting-guides` |

Run `claude mcp get dynatrace-mcp` to see the full list of available tools.

---

## 10. Tips & Gotchas

---

**`mcp-gateway:servers:invoke` is the most commonly missed scope.**
Without it, the token will authenticate but the MCP server won't actually be invocable — tools will silently fail or return permission errors.

---

**Token scopes are additive — start minimal, add as needed.**
If a specific tool category stops working, check whether its corresponding scope group (`davis:*`, `storage:*:read`, etc.) is included in your token.

---

**The `--scope user` flag matters.**
Without it, the MCP server is registered with local scope (only available in the current project directory). Use `--scope user` to make it available everywhere.

---

**Platform Tokens can expire.**
If Claude Code suddenly loses connection to your tenant, run `claude mcp list` to check status. If it shows `! Needs authentication`, your token may have expired — generate a new one and re-run the `claude mcp add` command.

---

**Re-registering an existing server.**
If you need to update the token or URL, remove the old registration first:
```bash
claude mcp remove dynatrace-mcp -s user
```
Then re-run the `claude mcp add` command with the updated values.

---

**The plugin and MCP server are independent.**
You can use the domain skills plugin without the MCP server (for offline DQL writing/guidance), or the MCP server without the plugin. Both together give the best experience.

---

## 11. Further Reading

- [Dynatrace MCP Server — Official Documentation](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp)
- [Claude Code CLI — MCP Setup](https://docs.anthropic.com/en/docs/claude-code/mcp)
- [Dynatrace Platform Tokens](https://docs.dynatrace.com/docs/manage/identity-access-management/access-tokens-and-oauth-clients/access-tokens)
- [DQL Reference](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language)

---

> **Disclaimer:** This guide is AI-assisted and intended for reference and learning purposes only. It may contain inaccuracies, incomplete information, or content that has drifted from current product behavior — always consult the [official Dynatrace documentation](https://docs.dynatrace.com) for authoritative guidance. This is not an official Dynatrace resource.
