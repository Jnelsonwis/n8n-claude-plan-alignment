# Run Claude Code Node — Prompt Design Guide

This file documents how to write effective prompts for the **Run Claude Code** node in this workflow. The prompt is the most impactful thing you can customize — it determines what Claude checks, how deep it looks, and what it outputs.

---

## How the Prompt Gets Passed

The prompt lives inside a JavaScript string in the Code node, then is embedded in a shell command:

```js
const prompt = 'Your prompt here. Use \\\\n for newlines inside JS strings.';
const cmd = `su - YOUR_USER -c '... $CLAUDE_BIN --print "${prompt}" ...'`;
```

**Escaping rules inside the JS string:**
- Newlines → `\\n` (four chars: backslash backslash n)
- Double quotes → `\\"` (avoid them; rephrase to not need quotes in the prompt)
- Single quotes → avoid entirely (they break the shell command wrapper)
- Backticks → avoid or use Unicode equivalent

---

## Anatomy of a Good Prompt

A well-structured prompt has four parts:

```
[1. CONTEXT]     Tell Claude what it's reading and why
[2. CHECKS]      Numbered list of specific things to verify, with exact file paths
[3. INVARIANTS]  Hard rules Claude must flag if the plan violates them
[4. OUTPUT]      Exact section names and format to use in the response
```

### Why this structure works

- **Numbered checks** give Claude a deterministic traversal order — it won't skip steps
- **Exact file paths** prevent Claude from guessing or hallucinating paths
- **Named invariants** (PROTECTED, BLOCKED BY DESIGN, etc.) let the Format Report and Build Gotify Message nodes pattern-match on keywords to grade severity
- **Explicit output sections** produce consistent markdown that downstream nodes can parse

---

## Generic Template

Use this as your starting point, then fill in the paths and rules for your stack:

```
Read .tmp-plan-check.md — a development plan to validate against this codebase.

PHASE 1 — SCHEMA & DATA
1. Read [YOUR_SCHEMA_FILE] — for every table/model/collection the plan mentions: confirm it exists, check all referenced columns/fields exist, flag any that are missing as needing a new migration or schema change.
2. If the plan adds new tables/models, check [YOUR_MIGRATIONS_DIR] for the highest existing migration number and note what the next one should be named.

PHASE 2 — API & ROUTING
3. Grep [YOUR_ROUTES_DIR] for existing endpoint definitions. For every API route the plan mentions: confirm it exists. Flag missing routes as NEW — list the handler file it should go in.
4. Check [YOUR_MIDDLEWARE_FILE] — confirm the plan respects any auth middleware, rate limiting, or request validation already wired up. Flag anything the plan bypasses as PROTECTED.

PHASE 3 — BUSINESS LOGIC INVARIANTS
5. Read [YOUR_CORE_LOGIC_FILE] — identify any hard-coded rules, hooks, or constraints (e.g. auto-populated fields, role restrictions, computed values). If the plan writes to or reads these fields directly, flag as BLOCKED BY DESIGN and explain what the correct approach is.
6. Check [YOUR_CONFIG_FILE] — record key configuration values (feature flags, limits, defaults). Verify the plan does not assume different values than what is configured.

PHASE 4 — DEPENDENCIES & PLACEMENT
7. Verify every new file the plan creates goes in the correct directory per the existing project structure.
8. If the plan adds new packages, grep package.json (or requirements.txt / go.mod / Gemfile) to check they are not already installed. Note any that require a build step or native compilation.

PHASE 5 — CROSS-CUTTING CONCERNS
9. Check [YOUR_ENV_EXAMPLE_FILE] — does the plan require new environment variables that are not already declared? List them.
10. Scan for any plan steps that touch shared infrastructure (queues, caches, storage buckets, cron jobs) — flag these as INFRASTRUCTURE CHANGE: describe what needs to be provisioned.

OUTPUT — use exactly these markdown section headers:
## Aligned
## Warnings
## Misaligned
(table: Issue | Plan Says | Reality | Fix)
## Missing from Plan
## New Infrastructure
## Environment Variables Needed
Be precise. Use ✅ for aligned, ⚠️ for warnings, ❌ for misaligned.
```

---

## `--allowed-tools` Reference

The `--allowed-tools` flag controls what Claude can do. **Always be as restrictive as possible.**

### Read-only analysis (recommended default)

```
--allowed-tools "Read Bash(grep:*)"
```

Claude can read any file and run grep. Cannot write, delete, or execute anything else.

### Read + specific file inspection

```
--allowed-tools "Read Bash(grep:*) Bash(cat:package.json) Bash(cat:.env.example) Bash(cat:go.mod)"
```

Useful when you need Claude to check specific config files that aren't in your source tree.

### Read + directory listing

```
--allowed-tools "Read Bash(grep:*) Bash(ls:src/**)"
```

Useful when you need Claude to verify file placement and directory structure.

### What to never allow in this workflow

```
# Never allow in a read-only analysis workflow:
Edit, Write, Bash(rm:*), Bash(npm:*), Bash(git:*), WebFetch, WebSearch
```

These turn a passive analysis into an agent that can modify your codebase.

---

## `--model` and `--max-turns` Tradeoffs

| Setting | Use when |
|---------|----------|
| `--model sonnet` | Default — fast, accurate for most plans |
| `--model opus` | Plan touches 10+ files or has complex conditional logic |
| `--max-turns 10` | Small plans, few files to check |
| `--max-turns 20` | Large plans, many files, deep schema checks |
| `--max-turns 30` | Monorepos, multi-service plans |

Higher `--max-turns` increases response time linearly. Set it to the minimum needed for your typical plan size.

---

## Stack-Specific Examples

### TypeScript + Drizzle ORM + Hono/Express

```
Read .tmp-plan-check.md to validate against this TypeScript codebase.\\n
1. Read src/db/schema.ts — check every table and column the plan mentions. Flag missing ones as needing a Drizzle migration. Check src/db/migrations/ for the latest migration number.\\n
2. Grep src/routes/ for existing Hono/Express route definitions. Verify every API path the plan mentions. Flag missing routes.\\n
3. Read src/middleware/auth.ts — check the plan does not bypass JWT validation or role guards already in place. Flag bypasses as PROTECTED.\\n
4. Read src/lib/constants.ts — check the plan does not hardcode values that should come from this file.\\n
5. Verify new files go in the correct src/ subdirectory per the existing structure.\\n
Output sections: Aligned, Warnings, Misaligned (table), Missing from Plan, New Infrastructure. Use ✅ ⚠️ ❌.
```

### Python + Django + PostgreSQL

```
Read .tmp-plan-check.md to validate against this Django codebase.\\n
1. Grep */models.py for model class definitions. For every model the plan mentions: confirm it exists, check all referenced fields are defined, flag missing fields as needing a migration. Note the next migration number from */migrations/.\\n
2. Grep */urls.py and */views.py for URL patterns and view names. Verify every endpoint the plan mentions exists.\\n
3. Read */middleware.py and settings.py MIDDLEWARE list — check the plan does not bypass authentication or permission classes already declared. Flag as PROTECTED if so.\\n
4. Read settings.py — note INSTALLED_APPS, DATABASES config, and any feature flags. Flag plan steps that assume different config.\\n
5. Check requirements.txt — flag any new packages the plan introduces that are not listed. Note packages with C extensions that require system deps.\\n
6. Verify new files go in the correct Django app directory (models, views, serializers, tests).\\n
Output sections: Aligned, Warnings, Misaligned (table), Missing from Plan, New Infrastructure, Environment Variables Needed. Use ✅ ⚠️ ❌.
```

### Python + FastAPI + SQLAlchemy

```
Read .tmp-plan-check.md to validate against this FastAPI codebase.\\n
1. Grep app/models/ for SQLAlchemy model definitions. Check every table/column the plan references exists. Flag missing as needing Alembic migrations. Check alembic/versions/ for the latest revision.\\n
2. Grep app/routers/ for FastAPI router and endpoint definitions (@router.get, @router.post, etc.). Verify every route the plan mentions.\\n
3. Read app/dependencies.py — check the plan respects existing Depends() injected auth and permission checks. Flag any route that skips required dependencies as PROTECTED.\\n
4. Read app/core/config.py — check plan does not hardcode values managed by Pydantic Settings. List any new env vars the plan needs as ENVIRONMENT VARIABLES NEEDED.\\n
5. Read pyproject.toml or requirements.txt — check new packages are not already present. Flag packages that require a new Docker build layer.\\n
Output sections: Aligned, Warnings, Misaligned (table), Missing from Plan, New Infrastructure, Environment Variables Needed.
```

### Ruby on Rails

```
Read .tmp-plan-check.md to validate against this Rails codebase.\\n
1. Grep db/schema.rb for table and column definitions. Check every table/column the plan mentions. Flag missing as needing a new migration. Check db/migrate/ for the latest timestamp.\\n
2. Grep config/routes.rb and app/controllers/ for route and action definitions. Verify every route the plan mentions exists.\\n
3. Read app/models/application_record.rb and any relevant model files — check for before_action callbacks, validations, and scopes the plan might conflict with. Flag as BLOCKED BY DESIGN if the plan writes directly to fields managed by callbacks.\\n
4. Read config/initializers/ — check the plan does not conflict with configured gems (Devise, Pundit, etc.).\\n
5. Read Gemfile — check new gems the plan introduces are not already present. Flag gems that require bundle install + server restart.\\n
6. Check config/credentials.yml.enc references — list any new credentials the plan needs.\\n
Output sections: Aligned, Warnings, Misaligned (table), Missing from Plan, New Infrastructure, Credentials Needed.
```

### Go + Chi/Gin + SQLC or GORM

```
Read .tmp-plan-check.md to validate against this Go codebase.\\n
1. Grep internal/db/ or sqlc/ for query files and struct definitions. Check every table/struct the plan references. Flag missing as needing a new SQL migration. Check migrations/ for the latest version number.\\n
2. Grep internal/handler/ or internal/routes/ for HTTP handler registrations. Verify every endpoint the plan mentions is registered.\\n
3. Read internal/middleware/ — check the plan does not bypass auth middleware or rate limiter already applied to route groups. Flag as PROTECTED.\\n
4. Read internal/config/config.go — check the plan does not hardcode values that should come from env/config. List new env vars needed.\\n
5. Read go.mod — check new packages the plan introduces. Flag packages with CGO requirements.\\n
6. Verify new files follow Go package conventions (one package per directory, no circular imports between packages the plan creates).\\n
Output sections: Aligned, Warnings, Misaligned (table), Missing from Plan, New Infrastructure, Environment Variables Needed.
```

### Next.js App Router + Prisma + NextAuth

```
Read .tmp-plan-check.md to validate against this Next.js App Router codebase.\\n
1. Read prisma/schema.prisma — check every model and field the plan mentions. Flag missing as needing prisma migrate. Check prisma/migrations/ for the latest migration.\\n
2. Grep app/api/ for route.ts files. Verify every API route the plan mentions (GET/POST/etc. exports). Flag missing routes.\\n
3. Read lib/auth.ts or app/api/auth/[...nextauth]/route.ts — check the plan respects session shape, JWT callbacks, and any role fields stored on the session. Flag as PROTECTED if the plan reads session fields that do not exist.\\n
4. Grep app/ for page.tsx and layout.tsx files — check the plan does not create pages in locations that would conflict with existing layouts or parallel routes.\\n
5. Read next.config.js — check for rewrites, redirects, or experimental flags the plan might conflict with.\\n
6. Read package.json — check new packages. Flag any that require next.config.js changes or have known Next.js App Router incompatibilities.\\n
7. MOBILE/PWA: If next-pwa or a manifest is present, check the plan does not alter public/manifest.json or service worker config.\\n
Output sections: Aligned, Warnings, Misaligned (table), Missing from Plan, New Infrastructure, Environment Variables Needed.
```

---

## Adding Custom Invariant Checks

Invariants are rules specific to your codebase that the plan must not violate. Define them as numbered checks with a named flag Claude must emit when violated.

**Pattern:**

```
N. Read [file] — [what to look for]. If the plan [violation condition], flag as [KEYWORD]: [explanation of why and what to do instead].
```

**Examples:**

```
# Billing protection
6. Read src/lib/billing.ts — identify all functions that increment usage counters or
   charge credits. If the plan calls these functions without first checking the user's
   current plan limits, flag as BILLING_BYPASS: list the unguarded calls and the
   limit-check function that must wrap them.

# Audit log requirement
7. Read src/lib/audit.ts — check the logAuditEvent() function signature. If the plan
   adds, updates, or deletes any records in the users, workspaces, or billing tables
   without calling logAuditEvent(), flag as AUDIT_MISSING: list every unlogged
   write operation.

# Tenant isolation
8. Read src/middleware/tenant.ts — verify tenant scoping is applied via
   withTenantScope(). If the plan adds any database queries that access records
   across tenants without explicit tenant filtering, flag as TENANT_LEAK: list
   every unscoped query.

# Rate limit coverage
9. Grep src/routes/ for rateLimit() middleware usage. If the plan adds any public
   POST/PUT/DELETE endpoints without rate limiting, flag as RATE_LIMIT_MISSING:
   list each unprotected endpoint.
```

The keywords (`BILLING_BYPASS`, `AUDIT_MISSING`, etc.) can then be added to the severity detection regex in the **Build Gotify Message** node:

```js
const hasCritical = /CRITICAL|BLOCKED BY DESIGN|MOBILE_INTERFERENCE|BILLING_BYPASS|AUDIT_MISSING|TENANT_LEAK/i.test(analysis);
```

---

## Output Section Design

The sections you define in your prompt are parsed downstream. Keep them consistent:

| Section | Purpose | Downstream impact |
|---------|---------|-------------------|
| `## Aligned` | Items that match the codebase | Positive signal, no action |
| `## Warnings` | Things to double-check, not blockers | Increments `warningCount` in Gotify node |
| `## Misaligned` | Hard conflicts, structured as a table | Increments `misalignedCount`, drives severity |
| `## Missing from Plan` | Things the plan should have covered | Informational |
| `## New Infrastructure` | Net-new tables, services, queues, etc. | Informs deployment checklist |
| `## Environment Variables Needed` | New env vars to add to `.env` / secrets manager | Informs ops checklist |
| `## MOBILE CONFIG STATUS` | `Safe` or `Needs Review` | Triggers `MOBILE_INTERFERENCE` flag if violated |

If you rename sections, update the regex patterns in **Build Gotify Message** and **Format Report** to match.

---

## Debugging Prompt Issues

If Claude returns partial or unexpected output:

1. **Check escaping** — run the `cmd` string through `console.log` in the node and inspect it. Unescaped quotes will break the shell invocation silently.
2. **Increase `--max-turns`** — if the output cuts off mid-analysis, Claude hit the turn limit.
3. **Narrow `--allowed-tools`** — if Claude is spending turns trying to use tools it doesn't have, it wastes turns. Be explicit.
4. **Split large plans** — prompts work best on plans under ~2,000 words. For large plans, split into sections and call the webhook multiple times.
5. **Test headlessly first** — SSH to the server and run the `claude --print "..."` command directly to validate the prompt before wiring it into n8n.
