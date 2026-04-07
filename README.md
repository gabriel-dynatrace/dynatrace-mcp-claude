# Dynatrace MCP Setup Guide — Claude Desktop & Claude Code

Connect your Dynatrace environment to Claude for live DQL queries, problem investigation, log analysis, and AI-assisted observability workflows. This guide covers two supported clients: the **Claude Desktop app** and **Claude Code CLI**.

---

## Table of Contents

1. [Which Claude Client Should I Use?](#1-which-claude-client-should-i-use)
2. [Why Claude.ai Web Doesn't Work](#2-why-claudeai-web-doesnt-work)
3. [Platform Token Scopes](#3-platform-token-scopes)
4. [Option A — Claude Desktop App (Extension)](#4-option-a--claude-desktop-app-extension)
5. [Option B — Claude Code CLI](#5-option-b--claude-code-cli)
   - [Step 1 — Find Your MCP Gateway URL](#step-1--find-your-mcp-gateway-url)
   - [Step 2 — Register the MCP Server](#step-2--register-the-mcp-server)
   - [Step 3 — Install the Domain Skills Plugin](#step-3--install-the-domain-skills-plugin)
   - [Step 4 — Verify & Test](#step-4--verify--test)
6. [What You Can Do Now](#6-what-you-can-do-now)
7. [Tips & Gotchas](#7-tips--gotchas)
8. [Further Reading](#8-further-reading)

---

## 1. Which Claude Client Should I Use?

| | Claude Desktop App | Claude Code CLI |
|---|---|---|
| Who it's for | Anyone using the Claude desktop app | Developers / technical users in the terminal |
| Setup method | Extensions UI (no config files) | `claude mcp add` command |
| Auth supported | Platform Token **or** OAuth | Platform Token (Bearer header) |
| Works on claude.ai web? | No — see below | No — see below |

Use **Claude Desktop** if you want a point-and-click setup. Use **Claude Code CLI** if you want tenant access available across all your coding projects in the terminal.

---

## 2. Why Claude.ai Web Doesn't Work

If you've tried adding Dynatrace through claude.ai in a browser and seen "Needs authentication" — that's expected.

| | Claude Desktop App | Claude Code CLI | claude.ai (web) |
|---|---|---|---|
| Auth method | Platform Token or OAuth | Bearer token (header) | OAuth 2.0 only |
| Works? | **Yes** | **Yes** | **No — not yet** |

**Claude.ai web** requires MCP servers to complete an OAuth 2.0 handshake with a public discovery endpoint. The Dynatrace MCP gateway does not currently expose a compatible OAuth endpoint for browser-based sessions — so the connection stalls at "Needs authentication."

The **Desktop app** and **Claude Code CLI** both support Platform Token (Bearer) authentication directly, which is why they work.

> This limitation is on the Dynatrace MCP gateway side, not Claude's. Until Dynatrace adds a compatible OAuth 2.0 endpoint, the web client cannot connect.

---

## 3. Platform Token Scopes

Regardless of which client you use, you'll need a **Platform Token** with the scopes listed below.

Platform Tokens are managed in **Account Management > Platform tokens**. That page lists all tokens created by users in your account, but you cannot create one directly there.

To create your own token:
1. Go to **Account Management > Platform tokens**
2. Click the **user profile** link in the message: *"If you want to create a platform token for yourself, please go to your user profile"*
3. In your user profile, click **Create token**, name it (e.g. `claude-mcp`), and add the scopes below

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

> Copy the generated token immediately — it won't be shown again. Token format: `dt0s16.XXXX.XXXX...`

> **Note:** Always verify the complete and current scope list against the [official Dynatrace MCP Server documentation](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp) — scopes may change with platform updates.

---

## 4. Option A — Claude Desktop App (Extension)

This is the easiest setup path — no terminal required.

### Prerequisites
- [Claude Desktop app](https://claude.ai/download) installed (Mac or Windows)
- An active Claude Pro, Team, or Enterprise subscription
- A Dynatrace Platform Token (see [Section 3](#3-platform-token-scopes))

### Steps

**1. Open Extensions**

In the Claude Desktop app, go to **Settings > Extensions** (under the "Desktop app" section of the sidebar).

**2. Browse and install**

Click **Browse extensions**, search for `dynatrace`, and select **Dynatrace MCP Server**. Click to install it.

**3. Configure**

Click **Configure** next to the installed extension and fill in the fields:

| Field | Value |
|-------|-------|
| **Dynatrace Environment URL** *(Required)* | Your tenant base URL: `https://your-env-id.apps.dynatrace.com/` — do not use classic URLs |
| **Dynatrace Platform Token** *(Optional)* | Your `dt0s16.*` token — recommended over OAuth for simplicity |
| **OAuth Client ID** *(Optional)* | Leave blank unless using OAuth-based auth |
| **OAuth Client Secret** *(Optional)* | Leave blank unless using OAuth-based auth |
| **Grail Query Budget (GB)** | Max GB scanned per session (default: `1000`). Set to `-1` to disable the limit |
| **Disable Telemetry** | Toggle on to stop usage events being sent to Dynatrace |

**4. Save and enable**

Click **Save**. The toggle at the top of the configuration page should show **Enabled**.

**5. Test it**

Start a new conversation in the Claude Desktop app and try:
> "Are there any open problems in my Dynatrace environment right now?"

If Claude queries your tenant and returns results, the extension is working.

---

## 5. Option B — Claude Code CLI

Claude Code CLI requires a terminal and Node.js, but gives you Dynatrace access automatically across all your coding projects.

### Prerequisites
- **Node.js 18+** ([nodejs.org](https://nodejs.org))
- **Claude Code CLI** with an active Claude Pro, Team, or Enterprise subscription:
  ```bash
  npm install -g @anthropic-ai/claude-code
  ```
- A Dynatrace Platform Token (see [Section 3](#3-platform-token-scopes))

### Step 1 — Find Your MCP Gateway URL

Your tenant exposes the MCP gateway at a fixed path — just substitute your environment ID:

```
https://<your-env-id>.apps.dynatrace.com/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp
```

Your `env-id` is the subdomain in your browser's URL bar when logged into Dynatrace (e.g. `abc12345` from `https://abc12345.apps.dynatrace.com`).

### Step 2 — Register the MCP Server

Set your credentials as environment variables, then run the registration command. The shell expands the variables at runtime, so the **actual token value** (not the variable name) gets written into `~/.claude/settings.json`.

```bash
export DT_PLATFORM_TOKEN="dt0s16.YOUR_TOKEN_HERE"
export DT_ENV_URL="https://your-env-id.apps.dynatrace.com"

claude mcp add dynatrace-mcp \
  --transport http \
  --scope user \
  "${DT_ENV_URL}/platform-reserved/mcp-gateway/v0.1/servers/dynatrace-mcp/mcp" \
  --header "Authorization: Bearer ${DT_PLATFORM_TOKEN}"
```

- `--transport http` — Dynatrace MCP is a remote HTTP server, not a local process
- `--scope user` — makes it available in **all** your Claude Code projects (omit for current project only)
- `--header` — passes your Platform Token as a Bearer token on every request

> The environment variables are only needed during setup. Once registered, the token is stored in `~/.claude/settings.json` and you do not need to set them again.

### Step 3 — Install the Domain Skills Plugin

The plugin adds Dynatrace-specific knowledge and DQL patterns. It's separate from the MCP server and doesn't require credentials.

```bash
claude plugin marketplace add dynatrace-for-ai --source github:dynatrace/dynatrace-for-ai
claude plugin install dynatrace@dynatrace-for-ai
```

### Step 4 — Verify & Test

```bash
claude mcp list
```

You should see:
```
dynatrace-mcp: https://your-env.apps.dynatrace.com/... (HTTP) - ✓ Connected
```

If it shows `! Needs authentication`, double-check that your token has the correct scopes and hasn't expired.

Start Claude Code and test:
```bash
claude
```
> "Are there any open problems in my Dynatrace environment right now?"

---

## 6. What You Can Do Now

Once connected (via either client), Claude can work directly with your tenant data:

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

---

## 7. Tips & Gotchas

---

**`mcp-gateway:servers:invoke` is the most commonly missed scope.**
Without it, the token authenticates but the MCP server can't actually be invoked — tools will silently fail or return permission errors.

---

**Desktop app: use your base tenant URL, not the full gateway path.**
The extension only needs `https://your-env-id.apps.dynatrace.com/`. The Claude Code CLI needs the full `/platform-reserved/mcp-gateway/...` path.

---

**Claude Code: the `--scope user` flag matters.**
Without it, the MCP server is only available in the current directory. Use `--scope user` to make it available everywhere.

---

**Platform Tokens can expire.**
If your connection drops, run `claude mcp list` (CLI) or check **Settings > Extensions** (Desktop). Regenerate the token via your user profile in Account Management and re-enter it if expired.

---

**Grail Query Budget (Desktop app).**
The default budget is 1000 GB per session. For general use this is fine. Set to `-1` to disable the limit entirely.

---

**Re-registering the CLI server.**
To update your token or URL, remove the old entry first:
```bash
claude mcp remove dynatrace-mcp -s user
```
Then re-run the `claude mcp add` command.

---

## 8. Further Reading

- [Dynatrace MCP Server — Official Documentation](https://docs.dynatrace.com/docs/dynatrace-intelligence/dynatrace-mcp)
- [Claude Desktop App — Extensions](https://claude.ai/download)
- [Claude Code CLI — MCP Setup](https://docs.anthropic.com/en/docs/claude-code/mcp)
- [Dynatrace Platform Tokens](https://docs.dynatrace.com/docs/manage/identity-access-management/access-tokens-and-oauth-clients/access-tokens)
- [DQL Reference](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language)

---

> **Disclaimer:** This guide is AI-assisted and intended for reference and learning purposes only. It may contain inaccuracies, incomplete information, or content that has drifted from current product behavior — always consult the [official Dynatrace documentation](https://docs.dynatrace.com) for authoritative guidance. This is not an official Dynatrace resource.
