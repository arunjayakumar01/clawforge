# ClawForge — One Dashboard for All Your AI Assistants

ClawForge is the admin layer for [OpenClaw](https://github.com/openclaw/openclaw). It gives you a single control plane to manage authentication, policies, tool and skill governance, audit trails, and a remote kill switch — whether you're a team or a power user running multiple gateways.

## Architecture

```
┌─────────────────────┐       ┌──────────────────────┐       ┌─────────────────────┐
│   OpenClaw Gateway   │       │   Control Plane API   │       │    Admin Console     │
│                     │  HTTP  │                      │  HTTP  │                     │
│  @openclaw/clawforge│◄─────►│  @openclaw/clawforge  │◄─────►│  @openclaw/clawforge │
│     (extension)     │       │       -server         │       │       -admin         │
└─────────────────────┘       └──────────┬───────────┘       └─────────────────────┘
                                         │
                                    PostgreSQL
```

**Three packages:**

| Package | Path | Description |
|---|---|---|
| `@openclaw/clawforge` | `plugin/` | OpenClaw plugin — hooks into the gateway lifecycle |
| `@openclaw/clawforge-server` | `server/` | Fastify control plane API (port 4100) |
| `@openclaw/clawforge-admin` | `admin/` | Next.js admin UI (port 4200) |

## Features

### Authentication
- Email/password login with seed-generated admin user
- Enrollment tokens for onboarding new users
- SSO / OIDC (Authorization Code with PKCE) — Okta, Auth0, Entra ID, etc.
- Token exchange, refresh, and session management
- Self-service password change
- First user in an org auto-promoted to admin
- `/clawforge-login` command in OpenClaw

### Org Policy Management
- Tool allow/deny lists and enforcement profiles
- Skill approval workflows (org-wide or per-user scope)
- Audit level configuration (full / metadata / off)
- Versioned policies with optimistic caching

### Tool Enforcement
- `before_tool_call` hook blocks disallowed tools
- Allow/deny list matching with profile-based overrides
- Blocked calls logged to audit trail

### Skill Governance
- `/clawforge-submit` command packages and submits skills
- Automated security scanning (file count, critical/warn/info findings)
- Admin review workflow: approve (org/self scope) or reject with notes
- Only approved skills loaded into the gateway

### Audit Logging
- Batched async event ingestion to control plane
- Configurable levels: `full` (includes LLM I/O), `metadata`, `off`
- Events: tool calls, session lifecycle, LLM input/output, policy changes
- Queryable by user, event type, tool, outcome, time range

### Kill Switch
- Instant remote disable of all tool calls
- Heartbeat polling with configurable interval and failure threshold
- Custom message displayed to users when active
- Admin toggle in web console

### `/clawforge-status` Command
- Shows auth state, org, policy version, kill switch status, audit level

## Quick Start

### Option A: Docker (Recommended)

The fastest way to get running. One command starts Postgres, runs migrations, seeds an admin user, and launches both the server and admin console.

```bash
git clone https://github.com/openclaw/clawforge.git
cd clawforge
docker compose up --build
```

Once running:

- **Server API:** http://localhost:4100
- **Admin Console:** http://localhost:4200
- **Default login:** `admin@clawforge.local` / `clawforge`

To customize the default admin credentials:

```bash
SUPERADMIN_EMAIL=admin@example.com SUPERADMIN_PASSWORD=s3cure \
  docker compose up --build
```

### Option B: Manual Setup

#### Prerequisites

- Node.js >= 22
- PostgreSQL >= 15
- pnpm

#### 1. Database Setup

```bash
createdb clawforge

cd server
pnpm install
pnpm db:migrate
pnpm db:seed  # Creates default org + admin user
```

The seed creates a superadmin you can log in with immediately. Customize with env vars:

| Variable | Default | Description |
|---|---|---|
| `SUPERADMIN_EMAIL` | `admin@clawforge.local` | Admin email |
| `SUPERADMIN_PASSWORD` | `clawforge` | Admin password |
| `SUPERADMIN_ORG_NAME` | `Default` | Organization name |

#### 2. Start the Control Plane

```bash
cd server

export DATABASE_URL="postgresql://localhost:5432/clawforge"
export JWT_SECRET="your-secret-here"  # Change in production!
export CORS_ORIGIN="http://localhost:4200"

# Development
pnpm dev

# Production
pnpm build && pnpm start
```

The API starts on `http://localhost:4100`. Health check: `GET /health`.

#### 3. Start the Admin Console

```bash
cd admin

export NEXT_PUBLIC_API_URL="http://localhost:4100"

# Development
pnpm dev

# Production
pnpm build && pnpm start
```

The admin UI is available at `http://localhost:4200`.

#### 4. Log In

With the seeded admin user:

```bash
curl -X POST http://localhost:4100/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@clawforge.local", "password": "clawforge", "orgId": "<ORG_ID>"}'
```

The org ID is printed during seed output.

### Configure SSO (Optional)

SSO is not required for getting started. To add OIDC authentication alongside email/password:

1. Register an OIDC application with your IdP
2. Add SSO config to your org record:

```sql
UPDATE organizations
SET sso_config = '{"issuerUrl": "https://your-idp.example.com", "clientId": "your-client-id"}'
WHERE name = 'Default';
```

3. Configure the OpenClaw plugin:

```json
{
  "plugins": {
    "clawforge": {
      "controlPlaneUrl": "http://localhost:4100",
      "orgId": "your-org-uuid",
      "sso": {
        "issuerUrl": "https://your-idp.example.com",
        "clientId": "your-oidc-client-id"
      }
    }
  }
}
```

4. Run `/clawforge-login` from your OpenClaw session to authenticate via browser.

### Connect an OpenClaw Gateway

Add ClawForge to your OpenClaw config (`openclaw.json`):

```json
{
  "plugins": {
    "clawforge": {
      "controlPlaneUrl": "http://localhost:4100",
      "orgId": "your-org-uuid",
      "policyCacheTtlMs": 300000,
      "heartbeatIntervalMs": 60000,
      "heartbeatFailureThreshold": 3,
      "auditBatchSize": 50,
      "auditFlushIntervalMs": 10000
    }
  }
}
```

## Configuration Reference

### Plugin Config (`ClawForgePluginConfig`)

| Key | Type | Default | Description |
|---|---|---|---|
| `controlPlaneUrl` | `string` | — | URL of the ClawForge control plane API |
| `orgId` | `string` | — | Organization UUID (fallback if not in session) |
| `sso.issuerUrl` | `string` | — | OIDC issuer URL (optional — only needed for SSO) |
| `sso.clientId` | `string` | — | OIDC client ID (optional — only needed for SSO) |
| `policyCacheTtlMs` | `number` | — | How long to cache policy locally (ms) |
| `heartbeatIntervalMs` | `number` | — | Kill switch polling interval (ms) |
| `heartbeatFailureThreshold` | `number` | — | Consecutive heartbeat failures before activating local kill switch |
| `auditBatchSize` | `number` | — | Max events per audit flush batch |
| `auditFlushIntervalMs` | `number` | — | Audit event flush interval (ms) |

### Server Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `4100` | HTTP listen port |
| `HOST` | `0.0.0.0` | Bind address |
| `DATABASE_URL` | `postgresql://localhost:5432/clawforge` | PostgreSQL connection string |
| `JWT_SECRET` | `clawforge-dev-secret-change-in-production` | JWT signing secret |
| `CORS_ORIGIN` | `*` | Comma-separated allowed origins |

### Admin Console Environment Variables

| Variable | Default | Description |
|---|---|---|
| `NEXT_PUBLIC_API_URL` | `http://localhost:4100` | Control plane API URL |

## API Reference

All endpoints except those marked "Public" require a `Bearer` token in the `Authorization` header.

### Auth

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Public | Email/password login |
| `POST` | `/api/v1/auth/exchange` | Public | SSO token exchange (OIDC code/token) |
| `POST` | `/api/v1/auth/enroll` | Public | Enroll with enrollment token |
| `GET` | `/api/v1/auth/mode` | Public | Available auth methods |
| `POST` | `/api/v1/auth/change-password` | User | Self-service password change |

**Login body:**
```json
{
  "email": "admin@clawforge.local",
  "password": "clawforge",
  "orgId": "org-uuid"
}
```

**Enrollment body:**
```json
{
  "token": "enrollment-token-string",
  "email": "newuser@example.com",
  "name": "New User"
}
```

**SSO grant types:**
- `authorization_code` — Code + PKCE verifier (requires `X-ClawForge-Org` header)
- `id_token` — Direct id_token validation (requires `orgId` in body)
- `refresh_token` — Refresh an expired session

**Response (all auth endpoints):**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "expiresAt": 1703361600000,
  "userId": "uuid",
  "orgId": "uuid",
  "email": "user@example.com",
  "roles": ["admin"]
}
```

### Enrollment Tokens

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/enrollment-tokens/:orgId` | Admin | Create token (optional: `label`, `expiresAt`, `maxUses`) |
| `GET` | `/api/v1/enrollment-tokens/:orgId` | Admin | List active tokens |
| `DELETE` | `/api/v1/enrollment-tokens/:orgId/:tokenId` | Admin | Revoke a token |

### Policies

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/v1/policies/:orgId/effective` | User | Get effective policy for authenticated user |
| `GET` | `/api/v1/policies/:orgId` | Admin | Get raw org policy |
| `PUT` | `/api/v1/policies/:orgId` | Admin | Update org policy |
| `PUT` | `/api/v1/policies/:orgId/kill-switch` | Admin | Toggle kill switch |

**Update policy body:**
```json
{
  "toolsConfig": {
    "allow": ["web_search", "read", "write"],
    "deny": ["exec"],
    "profile": "restricted"
  },
  "skillsConfig": {
    "requireApproval": true,
    "approved": [
      { "name": "weather", "key": "weather-v1", "scope": "org" }
    ]
  },
  "auditLevel": "full"
}
```

**Kill switch body:**
```json
{
  "active": true,
  "message": "Tool access suspended pending security review."
}
```

### Skills

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/skills/:orgId/submit` | User | Submit a skill for review |
| `GET` | `/api/v1/skills/:orgId/review` | Admin | List pending submissions |
| `PUT` | `/api/v1/skills/:orgId/review/:id` | Admin | Approve or reject a submission |
| `GET` | `/api/v1/skills/:orgId/approved` | User | List approved skills |

**Review body:**
```json
{
  "status": "approved-org",
  "reviewNotes": "Reviewed, no issues found.",
  "approvedForUser": "optional-user-uuid"
}
```

Status values: `approved-org`, `approved-self`, `rejected`

### Audit

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/api/v1/audit/:orgId/events` | User | Ingest audit events (batched from plugin) |
| `GET` | `/api/v1/audit/:orgId/query` | Admin | Query audit logs |

**Query parameters:** `userId`, `eventType`, `toolName`, `outcome`, `from`, `to`, `limit`, `offset`

### Heartbeat

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/v1/heartbeat/:orgId/:userId` | User | Client heartbeat — returns kill switch state + policy version |

### Users

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/v1/users/:orgId` | Admin | List org users |

## Database Schema

8 tables managed by Drizzle ORM:

| Table | Purpose |
|---|---|
| `organizations` | Org registry with optional SSO config (issuer, client ID, audience) |
| `users` | Org members with role (`admin` / `user`) and optional password hash |
| `policies` | Versioned org policies (tools, skills, audit level, kill switch) |
| `skill_submissions` | Skill review queue with security scan results |
| `approved_skills` | Approved skills per org, with optional per-user scope |
| `audit_events` | Tool calls, session lifecycle, LLM I/O events |
| `client_heartbeats` | Last heartbeat timestamp per user per org |
| `enrollment_tokens` | Admin-generated tokens for user onboarding |

## How It Works

### Startup Flow

1. Gateway loads the ClawForge extension
2. Extension checks for a saved session (`~/.clawforge/session.json`)
3. If expired → refresh via control plane; if missing → unauthenticated mode
4. Fetches org policy (cache → API → stale cache fallback)
5. Applies skill filter to OpenClaw config
6. Registers `before_tool_call` / `after_tool_call` / session / LLM hooks
7. Starts heartbeat polling for kill switch
8. On `gateway_stop` → flushes audit buffer, stops heartbeat

### Policy Enforcement

```
User invokes tool
    │
    ▼
before_tool_call hook fires
    │
    ├── Kill switch active? → BLOCK + audit "kill_switch_activated"
    │
    ├── Tool in deny list? → BLOCK + audit "tool_call_attempt" (blocked)
    │
    ├── Allow list exists & tool not in it? → BLOCK
    │
    └── ALLOW → audit "tool_call_attempt" (allowed)
         │
         ▼
    Tool executes
         │
         ▼
    after_tool_call hook → audit "tool_call_result"
```

### Policy Caching

```
On startup:
  1. Check local cache (within TTL) → use it, refresh in background
  2. Cache miss/expired → fetch from API, save to cache
  3. API unreachable → use stale cache as fallback

On heartbeat:
  - If server indicates new policy version → refresh immediately
```

## Development

```bash
# Install dependencies (from repo root)
pnpm install

# Run everything in dev mode
pnpm --filter @openclaw/clawforge-server dev   # API on :4100
pnpm --filter @openclaw/clawforge-admin dev    # UI on :4200

# Database management
cd server
pnpm db:generate   # Generate migration from schema changes
pnpm db:migrate    # Apply migrations
pnpm db:seed       # Seed default org + admin user
pnpm db:studio     # Open Drizzle Studio (visual DB browser)
```

## Known Gaps

- **No tests** — unit and integration tests needed for all three packages
- **No user role management API** — users are created via seed, enrollment tokens, or SSO; no role-change or remove API yet
- **No secret/key management** — no vault integration for API keys or credentials
- **No audit export** — no CSV/JSON export or retention policy management
- **No real-time push** — kill switch propagates on next heartbeat, not instantly
- **No multi-org management UI** — schema supports it, but no org creation flow

## License

MIT
