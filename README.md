# N8N Claude Code Plan Alignment Checker - (Guardrails for your developement) (Works with Claude Code Pro, Max or API Subscriptions) 
The result catches mismatches between what a plan *describes* and what the codebase *actually has* before any code is written.
Don't even have to leave your Claude Code session!
An n8n workflow that will use **Claude Code subscription** (running headlessly via [Supergateway](https://github.com/supercorp-ai/supergateway)) to validate a development plan against your actual codebase before you start coding.

Send a plan document to a webhook inside Claude Code chat — get back a structured markdown report identifying what aligns, what conflicts, what's missing, and whether any infrastructure changes are needed.

> **Usage & Policy Note:** This workflow uses `claude --print` (Claude Code's official headless mode), which is designed for non-interactive, scripted use. Claude Code with `--print` is a supported pattern for developer automation. It uses Claude subscription, which Anthropic covers for Claude Code usage — this is fine for personal dev workflows. If you're scaling this up or embedding it in a product serving other users, switch to the [Claude API](https://docs.anthropic.com/en/api) instead.

---

## What This Does

When you POST a development plan to the webhook, the workflow:

1. **Writes** the plan to a temp file on your server via an MCP filesystem server
2. **Invokes Claude Code** headlessly with read + grep tools against your project directory
3. **Analyzes** the plan against your actual code — checking schemas, routes, auth hooks, file placement, and mobile config (if applicable)
4. **Sends a push notification** via Gotify with a severity-graded summary
5. **Returns** a full markdown alignment report in the webhook response

The result catches mismatches between what a plan *describes* and what the codebase *actually has* before any code is written.

---

## Architecture

```
POST /webhook/plan-check
        │
        ▼
┌─────────────────┐
│   Webhook       │  Receives { filename, plan } JSON body
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Parse Input   │  Extracts plan text, builds MCP write_file payload
└────────┬────────┘
         │
         ▼
┌──────────────────────┐
│  Write Plan to Server │  Writes .tmp-plan-check.md via MCP HTTP API
└────────┬─────────────┘
         │
         ▼
┌─────────────────┐
│  Run Claude Code │  Opens SSE to Supergateway, runs `claude --print`
│                 │  with Read + Bash(grep) tools against your codebase
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌──────────┐  ┌──────────────┐
│  Gotify  │  │ Format Report │
│  Notify  │  │  (webhook     │
│ (async)  │  │   response)   |
|          |  | Back to your  |
|          |  |    current    |
|          |  |Claude Session !| 
└──────────┘  └──────────────┘
```

**Node summary:**

| Node | Purpose |
|------|---------|
| Webhook | Entry point, accepts POST with `filename` and `plan` fields |
| Parse Input | Builds MCP `write_file` JSON-RPC payload |
| Write Plan to Server | Calls your MCP filesystem server to write the temp plan file |
| Run Claude Code | Connects to Supergateway via SSE, runs Claude Code headlessly |
| Build Gotify Message | Parses output and grades severity for push notification |
| Send Gotify | POSTs markdown notification to Gotify |
| Format Report | Formats final markdown report returned in the webhook response |

---

## Prerequisites

| Component | Purpose | Notes |
|-----------|---------|-------|
| [n8n](https://n8n.io) | Workflow engine | Self-hosted recommended |
| [Claude Code](https://claude.ai/code) | AI analysis engine | Requires Claude Max subscription |
| [Supergateway](https://github.com/supercorp-ai/supergateway) | MCP-over-HTTP bridge | Runs as Docker container on same host as your project |
| [Gotify](https://gotify.net) | Push notifications | Optional — remove the Gotify nodes if not needed |
| Cloudflare Access | Auth for MCP/Gotify endpoints | Optional — remove CF headers if your services are on a private network |

### Why Supergateway?

Claude Code runs as a CLI on your server. Supergateway wraps it in an HTTP+SSE server so n8n can invoke it remotely. The `Run Claude Code` node opens a persistent SSE connection to Supergateway's `/sse` endpoint and sends commands via `/message`.

---

## Setup

### 1. Run Supergateway

Add this to your Docker Compose on the same host as your project:

```yaml
supergateway-filesystem:
  image: supercorp/supergateway:latest
  container_name: supergateway-filesystem
  restart: unless-stopped
  command: >
    --stdio "npx -y @modelcontextprotocol/server-filesystem /host"
    --port 8000
    --baseUrl http://supergateway-filesystem:8000
  volumes:
    - /:/host:rw
  networks:
    - n8n_default  # must be on the same Docker network as n8n
```

> **Security note:** The filesystem server mounts `/` read-write. Scope this to only the directories you need, e.g. `/opt/stacks/your-project:/host:rw`.

### 2. Install Claude Code on the server

```bash
npm install -g @anthropic-ai/claude-code
# Authenticate
claude auth login
```

Claude Code must be authenticated with an account that has a **Claude Max** subscription for the `sonnet` model to work in headless (`--print`) mode.

### 3. Import the Workflow

1. In n8n, go to **Workflows → Import**
2. Upload `workflow/plan-alignment-checker.json`
3. The workflow imports with all nodes pre-wired

### 4. Configure the Placeholders

Search for and replace every `YOUR_*` placeholder in the imported nodes:

| Placeholder | Where | What to set |
|-------------|-------|-------------|
| `YOUR_MCP_SERVER` | Write Plan to Server node | URL of your MCP HTTP server (e.g. `https://mcp.example.com`) |
| `YOUR_CF_ACCESS_CLIENT_ID` | Write Plan to Server, Send Gotify | Cloudflare Access service token client ID (omit header if not using CF) |
| `YOUR_CF_ACCESS_CLIENT_SECRET` | Write Plan to Server, Send Gotify | Cloudflare Access service token secret |
| `YOUR_PROJECT` | Parse Input, Format Report | Directory name of your project under `/opt/stacks/` |
| `YOUR_USER` | Run Claude Code | Linux username that owns the project directory |
| `YOUR_GOTIFY_SERVER` | Send Gotify | Base URL of your Gotify instance |
| `YOUR_GOTIFY_APP_TOKEN` | Send Gotify | App token from Gotify → Apps |
| `Your/Timezone` | Build Gotify Message, Format Report | IANA timezone string, e.g. `America/New_York` |
| `YOUR_N8N_INSTANCE_ID` | workflow JSON meta | Your n8n instance ID (cosmetic only) |

### 5. Activate the Workflow

Toggle the workflow to **Active** in n8n. The webhook path will be:

```
https://YOUR_N8N_HOST/webhook/plan-check
```

---

## How to Use

Send a POST request with your plan as a string:

```bash
curl -X POST https://YOUR_N8N_HOST/webhook/plan-check \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "my-feature-plan",
    "plan": "## Feature: ...\n\n### Database Changes\n..."
  }'
```

The response is a `text/markdown` document containing the full alignment report.

See [examples/sample-plan.md](examples/sample-plan.md) for a complete example request and response.

---

## Response Format

The returned markdown report contains these sections:

| Section | Description |
|---------|-------------|
| **Aligned** | Plan items that match what exists in the codebase (✅) |
| **Warnings** | Things that need attention but aren't blockers (⚠️) |
| **Misaligned** | Table of conflicts: what the plan says vs. what exists vs. how to fix |
| **Missing from Plan** | Things the plan should have addressed but didn't mention |
| **New Infrastructure** | New tables, services, or dependencies the plan requires |
| **MOBILE CONFIG STATUS** | `Safe` or `Needs Review` with a list of any mobile config keys the plan would affect |

---

## Notifications

If Gotify is configured, a push notification is sent immediately after analysis with a severity grade:

| Severity | Priority | Trigger |
|----------|----------|---------|
| ✅ All Clear | 1 (low) | No issues or warnings found |
| 📋 Warnings Only | 3 | Warnings present, no hard conflicts |
| ⚠️ Issues Found | 5 | One or more misaligned items |
| 🚨 CRITICAL | 8 | 4+ misaligned items, or auth/mobile config interference detected |

Notifications render as markdown in Gotify clients that support it.

---

## Customizing the Claude Prompt

The analysis prompt lives inside the **Run Claude Code** node's JavaScript. The default prompt is designed for a TypeScript + Drizzle ORM + Better Auth + Expo stack. Adapt it for your project:

**What to customize:**
- File paths checked (e.g. `src/db/schema.ts`, `src/routes/*.ts`, `src/lib/auth.ts`)
- Framework-specific hooks or constraints to validate
- The `--allowed-tools` flag — add or remove `Bash(cat:...)` entries for config files you want Claude to inspect
- The `--model` flag — `sonnet` is the default; use `opus` for more thorough analysis on complex plans
- `--max-turns` — increase for larger codebases

**Prompt structure:**

```
Read .tmp-plan-check.md → analyze against:
  1. Database schema (check tables/columns exist)
  2. API routes (check endpoints exist)
  3. Auth/framework hooks (check plan respects invariants)
  4. File placement (check new files go in right dirs)
  5. Mobile config (check plan doesn't break Expo config)
→ Output structured markdown report
```

**To disable mobile config checking** (non-mobile projects), remove step 5 from the prompt and the `MOBILE CONFIG STATUS` section from the Format Report node.

---

## Security Notes

- **Cloudflare Access headers** in Write Plan to Server and Send Gotify are only needed if those services sit behind a Cloudflare Access policy. Remove them if your services are on a private network.
- **The MCP filesystem server** has write access to your server. Restrict its volume mount to the minimum necessary path.
- **Claude Code authentication** is stored per-user on the server. The `su - YOUR_USER` in the command runs Claude Code as that user, scoped to their auth token.
- **The temp plan file** (`.tmp-plan-check.md`) is overwritten on each run and is never committed — it's just a bridge between n8n and Claude Code.
- Never commit real credentials to the workflow JSON. Use n8n's built-in **Credentials** system or environment variables for production.

---

## Removing Gotify (Notification-Free Mode)

If you don't want push notifications:

1. Delete the **Build Gotify Message** and **Send Gotify** nodes
2. Connect **Run Claude Code** directly to **Format Report**
3. The webhook response still returns the full report

---

## License

MIT
