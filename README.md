# Validate Development Plans Against Your Codebase Using Claude Code and n8n
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

### Why Docker-direct instead of an HTTP request to Supergateway?

Claude Code takes 30–90 seconds to analyze a project. **Cloudflare tunnels have a 120-second proxy read timeout** (Enterprise-only to increase). If your n8n instance and Supergateway are behind Cloudflare, any request that takes longer than 120s will fail with a 524 error — silently from n8n's perspective.

The `Run Claude Code` node bypasses this entirely by talking directly to `supergateway-filesystem:8000` on the shared Docker network via a persistent SSE connection. No Cloudflare, no proxy timeout. This is why Supergateway must be on the **same Docker network as n8n**.

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

### Inline plan

```bash
curl -s -X POST http://localhost:5678/webhook/plan-check \
  -H "Content-Type: application/json" \
  -d '{"filename": "my-feature", "plan": "## Feature: ...\n\n### DB Changes\n..."}' \
  | jq -r .report
```

### From a plan file

```bash
curl -s -X POST http://localhost:5678/webhook/plan-check \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
    --arg p "$(cat plans/my-feature.md)" \
    --arg n "$(basename plans/my-feature.md .md)" \
    '{plan: $p, filename: $n}')" \
  | jq -r .report
```

### Shell alias (add to `~/.bashrc` or `~/.zshrc`)

```bash
plancheck() {
  local file="${1:?Usage: plancheck <plan.md>}"
  curl -s -X POST http://localhost:5678/webhook/plan-check \
    -H "Content-Type: application/json" \
    -d "$(jq -n \
      --arg p "$(cat "$file")" \
      --arg n "$(basename "$file" .md)" \
      '{plan: $p, filename: $n}')" \
    | jq -r .report
}
```

Then just: `plancheck plans/my-feature.md`

### From inside Claude Code

```
Read plans/my-feature.md and POST it to http://localhost:5678/webhook/plan-check
as JSON {"plan": "<content>", "filename": "my-feature"}, then display the report.
```

### Via external URL

```bash
curl -s -X POST https://YOUR_N8N_HOST/webhook/plan-check \
  -H "Content-Type: application/json" \
  -d '{"filename": "my-feature", "plan": "..."}' \
  | jq -r .report
```

> **Note:** If your n8n is behind Cloudflare, use `localhost` for plans that take >90s to analyze. See [Why Docker-direct](#why-docker-direct-instead-of-an-http-request-to-supergateway) above.

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

## Troubleshooting

**524 timeout / "Workflow execution failed" from external URL**
Cloudflare's proxy read timeout is 120s. Claude Code analysis often takes longer. Use `localhost:5678` directly, or ensure Supergateway is on the same Docker network as n8n so the `Run Claude Code` node bypasses Cloudflare entirely.

**"No sessionId" / SSE connection never starts**
The `supergateway-filesystem` container is down or not on the right Docker network. Check:
```bash
docker ps | grep supergateway
docker network inspect YOUR_NETWORK --format '{{range .Containers}}{{.Name}} {{end}}' | grep supergateway
```

**Claude Code auth errors**
The auth token has expired. Re-authenticate on the server:
```bash
su - YOUR_USER -c 'claude auth login'
```

**Empty report or "No result found"**
Claude Code timed out (>170s). The plan may be too large for the current `--max-turns` setting. Test with a shorter plan to confirm the pipeline is working, then increase `--max-turns` in the Run Claude Code node.

**Claude Code binary not found after VS Code update**
The VS Code extension auto-updates and moves the binary. The command in the Run Claude Code node uses a glob to find the latest binary automatically:
```bash
ls -d /home/YOUR_USER/.vscode-server/extensions/anthropic.claude-code-*-linux-x64/resources/native-binary/claude | sort -V | tail -1
```
If Claude Code stops responding after an extension update, verify the path resolves:
```bash
su - YOUR_USER -c 'ls ~/.vscode-server/extensions/anthropic.claude-code-*/resources/native-binary/claude'
```

---

## Removing Gotify (Notification-Free Mode)

If you don't want push notifications:

1. Delete the **Build Gotify Message** and **Send Gotify** nodes
2. Connect **Run Claude Code** directly to **Format Report**
3. The webhook response still returns the full report

---

## Future Enhancements

Ideas that have come up in production use — good starting points for contributors:

- **Save reports to disk** — Add nodes after Format Report to write the report to a `docs/alignment-reports/` directory and clean up `.tmp-plan-check.md`
- **Auto-gate before execution** — Trigger plancheck automatically before Claude Code begins executing any plan; block on CRITICAL results
- **Schema snapshot validation** — Add your ORM's migration snapshot (e.g. `drizzle/meta/_journal.json`) as context to catch drift between the schema file and what's actually been migrated
- **Frontend component validation** — Extend the prompt to grep `src/components/` and `src/pages/` so UI references in the plan are validated against real component files
- **Multi-project support** — Accept a `project` field in the request body and route to different project directories and prompts based on it
- **GitHub Actions trigger** — Fire plancheck on PR creation using a GitHub webhook into n8n, post results as a PR comment

---

## License

MIT
