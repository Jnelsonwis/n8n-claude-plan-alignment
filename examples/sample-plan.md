# Sample Plan Payload

Send this via `curl` to test your workflow:

```bash
curl -X POST https://YOUR_N8N_HOST/webhook/plan-check \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "feature-user-invites",
    "plan": "## Feature: User Invitations\n\n### Overview\nAdd ability for workspace owners to invite new users via email.\n\n### Database Changes\n- Add `invitations` table with columns: id, workspace_id, email, token, expires_at, accepted_at, created_by\n- Add `invited_by` column to `users` table\n\n### API Routes\n- POST /api/invitations - create invitation, send email\n- GET /api/invitations/:token - validate token\n- POST /api/invitations/:token/accept - accept invite, create user\n- GET /api/invitations - list pending invitations for workspace\n\n### Auth Considerations\n- Invite acceptance creates a new user account via Better Auth createUser\n- Role should default to `member`, never inherit from invitation creator\n- Token expires after 48 hours\n\n### New Files\n- src/routes/invitations.ts - route handlers\n- src/lib/email/invite-template.tsx - email template\n- src/db/migrations/0012_invitations.sql - migration\n\n### Dependencies\n- No new native modules required\n- Uses existing resend integration for email"
  }'
```

## Response Structure

The webhook returns a markdown document:

```
# Plan Alignment Report

**Plan:** feature-user-invites
**Checked:** March 18, 2026, 10:30 AM
**Engine:** Claude Code (Max) → Sonnet
**Project:** /opt/stacks/your-project

---

## Aligned
- ✅ POST /api/invitations route pattern matches existing route structure
- ✅ src/routes/ is the correct directory for new route handlers
- ✅ src/db/migrations/ is the correct directory for SQL migrations

## Warnings
- ⚠️ `invited_by` column addition to `users` table requires a migration
- ⚠️ Email template in src/lib/email/ — verify resend integration supports TSX templates

## Misaligned
| Issue | Plan Says | Reality | Fix |
|-------|-----------|---------|-----|
| **invitations table** | id, workspace_id, email... | Does not exist | Create migration 0012 |

## Missing from Plan
- No mention of rate limiting on invite creation endpoint
- Token generation strategy not specified (UUID vs signed JWT)

## New Infrastructure
- New DB table: invitations (migration required before deploy)

## MOBILE CONFIG STATUS
Safe — no mobile config files touched by this plan
```
