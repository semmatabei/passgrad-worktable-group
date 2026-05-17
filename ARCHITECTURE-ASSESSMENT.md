# Architecture Assessment — Passgrad Worktable

**Date**: May 17, 2026
**Status**: Updated — Post Review
**Scope**: No-code database engine + workflow engine for `passgrad-worktable-app`

---

## Summary of Decisions

| Area                    | Decision                                                                                              |
| ----------------------- | ----------------------------------------------------------------------------------------------------- |
| Dynamic Schema          | Hybrid: metadata tables + JSONB record store                                                          |
| Multi-tenancy           | Single project, shared tables, `tenant_id` row-level isolation (RLS)                                 |
| Workflow engine         | Activepieces engine (vendored MIT source) + pieces (npm) — runs inside Render API service (Node.js)  |
| Job queue               | pg-boss on Supabase Postgres — no Redis, no separate queue service                                   |
| Automation layer        | Activepieces React UI (copied + API-adapted, MIT) — core product feature + monetization lever        |
| Tech stack              | Vite (React SPA) + Hono on Node.js/Render (monorepo)                                                 |
| API layer               | Hono — REST API for the SPA (databases, records, fields, flows, tasks)                               |
| Auth / DB isolation     | Supabase Auth + RLS with JWT claim pattern; `tenant_id` + `role` in `app_metadata`                   |
| Roles                   | `tenant_members(tenant_id, user_id, role)` — `owner` / `admin` / `member`                           |
| Sharing model           | Lark Base-inspired: base-level roles + view-level public links + public form submission               |
| Row visibility          | `records.created_by` + RLS; members see only their own records, admins see all                       |
| Select options          | IDs in `records.data`, labels in `fields.config.options[].id` — rename without record backfill       |
| Linked records          | Separate `record_links` table — FK integrity + clean cross-DB JOIN for formulas                      |
| Attachments             | Separate `attachments` table — lifecycle, MIME, size, presigned URL route                            |
| Workflow vs Flow        | Distinct: `workflows`/`tasks`/`task_actions` (approval) + `flows`/`flow_versions` (AP automation)   |
| View config shape       | Locked TypeScript type in `shared-types`: `FilterGroup`, `SortItem[]`, `hiddenFields`, `groupBy`     |
| Preset format           | TypeScript files in `apps/api/src/presets/` — versioned with code, seeded via `POST /tenants`        |
| Piece registration      | npm deps at build time; monetization gating as pre-execution check in pg-boss handler                |
| i18n                    | English + Indonesian from day one; `react-i18next`, translation keys in `apps/web/src/i18n/`         |
| Grid                    | AG Grid Community (MIT) — DIY grouping layer; migrate to Enterprise if server-side row model needed  |
| Automation UI           | Copy AP `packages/ui/src/features/flows/` (MIT) — API-adapted to our Hono backend                   |
| Formula engine          | HyperFormula (MIT, ~150 KB) — `{field_name}` → cell address adapter in `packages/core/formula.ts`   |
| Hosting                 | Cloudflare Pages (free) + Render Web Service ($7/mo) + Supabase free tier                            |
| **Total cost at start** | **~$7/mo**                                                                                            |

---

## 1. Dynamic Schema Strategy

### The Problem

User-defined tables (databases) must be stored and queried without pre-defined schemas. There are four approaches:

| Approach            | Description                               | Query perf               | Flexibility       | Complexity                        |
| ------------------- | ----------------------------------------- | ------------------------ | ----------------- | --------------------------------- |
| **DDL per table**   | `CREATE TABLE` per user database          | ✅ Native SQL            | ✅ Any field type | ❌ Schema sprawl, hard migrations |
| **EAV**             | `(record_id, field_id, value)` rows       | ❌ Needs many JOINs      | ✅                | ⚠️ Readable but slow              |
| **Pure JSONB**      | `records(id, data JSONB)`                 | ⚠️ Indexed JSON          | ✅                | ✅ Simple                         |
| **Hybrid** ← chosen | Metadata tables + `data JSONB` per record | ✅ Good with GIN indexes | ✅                | ⚠️ Medium                         |

### Decision: Hybrid

**Why not DDL per table**: 100 tenants × 10 databases each = 1,000 tables. Migrations, RLS policies, foreign keys all become unmanageable. Supabase imposes practical limits.

**Why not EAV**: Filter query like "show all students where status = active AND enrollment_year = 2024" requires multiple self-joins. Performance degrades badly beyond a few thousand records.

**Why Hybrid**: Store field definitions and view configs in structured metadata tables (queryable, indexable, RLS-friendly). Store record values in `data JSONB` on a single `records` table. Use Postgres GIN indexes on JSONB for filter/sort performance. This is the same approach used by Airtable, Teable, and Notion internally.

### Core Schema (simplified)

```sql
-- Tenant-scoped database (what users call a "database" or "table")
CREATE TABLE databases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  name TEXT NOT NULL,
  icon TEXT,
  created_by UUID,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Field definitions per database
CREATE TABLE fields (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  database_id UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  type TEXT NOT NULL, -- 'text' | 'number' | 'single_select' | 'multi_select' | 'date' | 'attachment' | 'person' | 'formula' | 'autonumber'
  config JSONB DEFAULT '{}', -- options list, formula expression, min/max, etc.
  position INTEGER NOT NULL DEFAULT 0,
  is_primary BOOLEAN DEFAULT false, -- the locked default field
  created_at TIMESTAMPTZ DEFAULT now()
);

-- View definitions (grid, form, kanban, calendar, etc.)
CREATE TABLE views (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  database_id UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  type TEXT NOT NULL, -- 'grid' | 'form' | 'kanban' | 'gallery' | 'calendar'
  config JSONB DEFAULT '{}', -- hidden fields, filter state, sort state, group state
  position INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Record storage — the core data
CREATE TABLE records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  database_id UUID NOT NULL REFERENCES databases(id) ON DELETE CASCADE,
  tenant_id UUID NOT NULL, -- denormalized for RLS performance
  data JSONB NOT NULL DEFAULT '{}', -- { [field_id]: value }
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- GIN index for JSONB filtering
CREATE INDEX records_data_gin ON records USING GIN (data jsonb_path_ops);

-- Cross-tenant integrity: composite FK ensures records.tenant_id always matches databases.tenant_id
-- Prevents a buggy INSERT from writing database_id=X with tenant_id=Y where X belongs to a different tenant
ALTER TABLE databases ADD CONSTRAINT databases_id_tenant_uk UNIQUE (id, tenant_id);
ALTER TABLE records   ADD CONSTRAINT records_database_tenant_fk
  FOREIGN KEY (database_id, tenant_id) REFERENCES databases(id, tenant_id);

-- Autonumber state per field
CREATE TABLE field_sequences (
  field_id UUID PRIMARY KEY REFERENCES fields(id) ON DELETE CASCADE,
  next_value BIGINT NOT NULL DEFAULT 1
);
```

### JSONB index strategy

A single GIN index is the baseline but insufficient for performant per-field filter and sort at scale. The field-create handler must emit a typed expression index for every filterable/sortable field alongside the `fields` row insert:

```sql
-- Emitted by the field-create handler when field is marked filterable/sortable
-- Type cast matches the field type ('text', 'numeric', 'timestamptz', etc.)
CREATE INDEX records_<field_id>_<database_id>
  ON records ((data->>'<field_id>')::<type>)
  WHERE database_id = '<database_id>';

-- Dropped by the field-delete handler
DROP INDEX IF EXISTS records_<field_id>_<database_id>;

-- On field-TYPE-CHANGE: drop and recreate with the new cast
-- Leaving a stale e.g. ::numeric index on a field now typed as text will cause index bloat/errors
DROP INDEX IF EXISTS records_<field_id>_<database_id>;
CREATE INDEX records_<field_id>_<database_id>
  ON records ((data->>'<field_id>')::<new_type>)
  WHERE database_id = '<database_id>';
```

| Index type                       | When applied                                                                    |
| -------------------------------- | ------------------------------------------------------------------------------- |
| `GIN (data jsonb_path_ops)`      | Baseline — one per `records` table, always present                              |
| Typed expression index per field | Created when a field is created as filterable/sortable; dropped on field delete |

This is the single highest-leverage code pattern in the platform. The field-create handler is the seam — every field type registers its index DDL there. **Must be built before any field type is implemented**, because retrofitting requires rebuilding indexes across all existing records.

### Field value storage in JSONB

Each record's `data` JSONB is keyed by `field_id`:

```json
{
  "field_uuid_1": "Ahmad Fauzi",
  "field_uuid_2": "active",
  "field_uuid_3": ["CS101", "CS201"],
  "field_uuid_4": "2024-09-01"
}
```

This enables: `SELECT * FROM records WHERE data->>'field_uuid_2' = 'active'` with a GIN index hit.

### Formula fields

Formula fields are computed on read (not stored), evaluated server-side in the API layer. Complex cross-database lookups are resolved by joining via the `records` table filtered by `database_id`.

### Scaling consideration

JSONB approach works well up to ~1M records per database on Postgres with proper GIN indexing. For databases exceeding this, partition by `database_id`. This is well beyond MVP scale.

---

## 2. Multi-tenancy Model

### Decision: Single Supabase project, shared tables, `tenant_id` isolation

**Why not schema-per-tenant**: Postgres `SET search_path` on every connection is error-prone. Supabase's PostgREST doesn't handle multi-schema well. Migrations require running against N schemas.

**Shared tables with `tenant_id` vs separate project per use case**:

Both are valid. The engine schema (`databases`, `fields`, `records`, etc.) is identical across all use cases — what differs is preset/seed data and workflow templates. However, separating by Supabase project per use-case deployment is also acceptable and has real advantages:

|                        | Shared project            | Separate project per use case       |
| ---------------------- | ------------------------- | ----------------------------------- |
| Schema migrations      | One migration updates all | Each project migrated independently |
| Data isolation         | Logical (RLS)             | Physical (separate Postgres)        |
| Supabase free tier     | 2 projects total          | 2 projects total (limits apply)     |
| Maintenance            | Single connection string  | N connection strings                |
| Cross-use-case queries | Possible                  | Not possible                        |
| Reasoning about data   | All tenants mixed         | Clean per-product boundary          |

**Decision**: Single project when deploying a single product (e.g., just AMS). Separate project when deploying as a distinct product with its own brand and commercial tier (e.g., AMS product vs. Kiosk product). The codebase is identical — only the Supabase connection config changes per deployment.

**Why shared tables with `tenant_id`** (within a single project):

- One migration to update all tenants
- RLS policies enforce isolation natively in Postgres
- Supabase is well-optimized for this pattern
- Can shard later if a single tenant grows very large (university with 50K students is fine on this model)

### RLS Policy Pattern

All DB queries go server-side through Hono (no direct Supabase access from the browser). RLS is still enforced as a second layer of defense — a missing `WHERE tenant_id = ?` in any route cannot leak cross-tenant data.

**Auth threading pattern (Hono → Supabase)**:

Use the **anon key** (not service role) for all route queries. Pass the user's JWT in the `Authorization` header to each per-request Supabase client. Supabase PostgREST sets `request.jwt.claims` from the header, which `auth.uid()` and RLS policies read from.

```typescript
// apps/api/src/middleware/supabase.ts
import { jwt } from "hono/jwt";
import { createClient } from "@supabase/supabase-js";

// 1. Verify JWT using Supabase JWT secret
app.use("*", (c, next) => jwt({ secret: c.env.SUPABASE_JWT_SECRET })(c, next));

// 2. Per-request Supabase client — forwards JWT so RLS fires
app.use("*", async (c, next) => {
  const supabase = createClient(c.env.SUPABASE_URL, c.env.SUPABASE_ANON_KEY, {
    global: { headers: { Authorization: c.req.header("Authorization") ?? "" } },
    auth: { persistSession: false, autoRefreshToken: false, detectSessionFromUrl: false },
  });
  c.set("supabase", supabase);
  await next();
});
```

**RLS policy — use JWT claim, not a correlated subquery**:

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE databases ENABLE ROW LEVEL SECURITY;
ALTER TABLE records ENABLE ROW LEVEL SECURITY;

-- WRONG — correlated subquery fires on every row during a filter scan
-- CREATE POLICY tenant_isolation ON records
--   USING (tenant_id = (SELECT tenant_id FROM profiles WHERE id = auth.uid()));

-- RIGHT — auth.jwt() is a STABLE function; the planner evaluates it once per query (not once per row)
CREATE POLICY tenant_isolation ON records
  USING (tenant_id = ((auth.jwt() -> 'app_metadata') ->> 'tenant_id')::uuid);
```

**Required companion — Supabase Auth Hook**: A Postgres function must write `tenant_id` and `role` into `raw_app_meta_data` at provisioning time so the JWT claim is populated. This lives in the first migration and must exist before any tenant is created — retrofitting requires resigning all existing JWTs.

**Security constraint: never read `tenant_id` from `raw_user_meta_data`**. That field is set by the client during `signUp({ data: {...} })` — an attacker can put any UUID there, and copying it into `app_metadata` would let them claim membership of any tenant. Instead, all tenant membership flows through a signed invite token:

1. Admin calls `POST /tenants/invites` — server generates a token: `HMAC-SHA256({tenant_id, role, email, exp})`, stores it in `tenant_invites(token_hash, tenant_id, role, email, expires_at, used_at)`.
2. Invite email contains a magic link: `https://app/accept-invite?token=<token>`.
3. User clicks → frontend calls `supabase.auth.signUp({ email, password, data: { invite_token } })` or `signInWithOtp`.
4. Auth Hook fires after insert. It reads `raw_user_meta_data->>'invite_token'`, validates the HMAC server-side, looks up the `tenant_invites` row, writes `tenant_id` + `role` into `raw_app_meta_data`, marks the invite as used.

```sql
-- supabase/migrations/0001_auth_hook.sql
CREATE TABLE public.profiles (
  id         UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  tenant_id  UUID,   -- set by the Auth Hook on invite acceptance; NULL for anonymous form submitters
  locale     TEXT NOT NULL DEFAULT 'id',
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE public.tenant_invites (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  token_hash  TEXT NOT NULL UNIQUE,   -- HMAC-SHA256 hex of the raw token
  tenant_id   UUID NOT NULL,
  role        TEXT NOT NULL DEFAULT 'member',
  email       TEXT NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  used_at     TIMESTAMPTZ,
  created_at  TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
DECLARE
  v_invite   public.tenant_invites;
  v_token    TEXT;
  v_hash     TEXT;
BEGIN
  -- Extract the invite token the user passed during signUp
  v_token := NEW.raw_user_meta_data->>'invite_token';

  IF v_token IS NULL THEN
    -- No invite token — public signup or anonymous form submitter
    -- Do NOT stamp any tenant_id; this user has no tenant membership
    RETURN NEW;
  END IF;

  -- Compute hash and look up the invite row
  v_hash := encode(hmac(v_token, current_setting('app.invite_hmac_secret'), 'sha256'), 'hex');

  SELECT * INTO v_invite
  FROM public.tenant_invites
  WHERE token_hash = v_hash
    AND email      = NEW.email
    AND used_at    IS NULL
    AND expires_at > now();

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Invalid or expired invite token';
  END IF;

  -- Stamp app_metadata with verified tenant_id + role
  UPDATE auth.users
  SET raw_app_meta_data = raw_app_meta_data ||
    jsonb_build_object(
      'tenant_id', v_invite.tenant_id::text,
      'role',      v_invite.role
    )
  WHERE id = NEW.id;

  -- Mark invite consumed
  UPDATE public.tenant_invites SET used_at = now() WHERE id = v_invite.id;

  -- Create profile row
  INSERT INTO public.profiles (id, tenant_id, locale)
  VALUES (NEW.id, v_invite.tenant_id, 'id')
  ON CONFLICT (id) DO NOTHING;

  -- Create tenant_members row
  INSERT INTO public.tenant_members (tenant_id, user_id, role)
  VALUES (v_invite.tenant_id, NEW.id, v_invite.role)
  ON CONFLICT (tenant_id, user_id) DO NOTHING;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

`app.invite_hmac_secret` is a Postgres GUC set in the migration: `ALTER DATABASE postgres SET app.invite_hmac_secret = '<secret>'`. The same secret is used in the Hono `POST /tenants/invites` route to generate tokens. It is never sent to the client.

### Supabase free tier operational notes

- **Auto-pause after 1 week of inactivity**: Add a CF Cron Trigger (free) pinging `/health` on the API to keep dev/staging databases awake.
- **2 active projects max on free**: dev and prod share the free tier. For staging, use a paused project or accept cold starts.
- **500 MB DB, 50K MAU**: sufficient for MVP. Upgrade to Supabase Pro ($25/mo) when either limit approaches.

### Multi-use-case deployment model

Different use cases (AMS, kiosk POS, healthcare) can share the same database engine. They differ in:

- Preset/template databases (seeded schemas)
- Workflow templates
- UI configuration

**Options per use case**:

| Deployment                                     | When to use                                                                              |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Same app, different tenant config              | Same fundamental schema needs (most cases)                                               |
| Separate app deployment, same Supabase project | Different UI/branding, same data model                                                   |
| Separate Supabase project                      | Truly different data isolation requirements (e.g., healthcare with HIPAA considerations) |

Start with single project. Only split when a use case demonstrably needs it.

---

## 3. Workflow & Automation Engine

### Decision: Activepieces engine vendored into the Render API service (Node.js)

The AP engine (`packages/server/engine`) is **not CF Workers compatible** — it targets Node.js 20 explicitly, uses `node:http`, `isolated-vm` (a native Node addon), `socket.io` IPC, and process-level DNS/socket patching. It cannot be bundled into a CF Worker.

**Solution**: The Hono API runs on Render (Node.js) rather than CF Workers. The AP engine and the pg-boss queue consumer run in the **same Node.js process** as the Hono HTTP server. No separate service, no Docker, no separate queue worker — one Render Web Service handles everything.

**What is replaced (the "adapter layer")**:

| Activepieces layer            | Replaced with                                          |
| ----------------------------- | ------------------------------------------------------ |
| Their server API (NestJS)     | Hono API — owns flow definitions + execution state     |
| Their Postgres/Redis          | Supabase (flow JSONB) + pg-boss on Supabase Postgres   |
| Their auth system             | Our existing Supabase auth                             |
| Their app shell (sidebar/nav) | Our React app shell                                    |
| Their Docker container        | Nothing — engine runs in-process inside Render service |

**What is kept as-is**:

| Activepieces layer                                | Status                            |
| ------------------------------------------------- | --------------------------------- |
| Engine execution logic                            | ✅ Copied verbatim (MIT)          |
| Variable interpolation (`{{step1.output.email}}`) | ✅ Engine handles this            |
| Branching / router step                           | ✅ Engine handles this            |
| All 280+ piece connectors                         | ✅ npm packages, import as needed |
| Flow builder React canvas                         | ✅ Copied, API layer swapped      |
| Step config forms                                 | ✅ Copied, API layer swapped      |
| Run history UI                                    | ✅ Copied, API layer swapped      |

### Architecture

```
User action (approve / reject via magic link in email)
  → Hono API (POST /flows/runs/resume?token=<resume_token>&decision=approve)  [Render Node.js]
      ├── Verify resume_token signature
      ├── pg-boss.cancel(escalation_job_id)  ← cancel deadline escalation if pending
      ├── UPDATE flow_runs SET status='running', assigned_to=null
      └── pg-boss.send('execute-flow', { flow_run_id, resume_token, decision })

Form submission (anonymous or authenticated)
  → Hono API (POST /forms/:token/submit)
      ├── INSERT flow_runs (status='running', submitted_by, trigger_payload)
      └── pg-boss.send('execute-flow', { flow_run_id })

pg-boss consumer (same Render process, background loop):
  ├── Deserialize flow definition from Supabase JSONB
  ├── Call vendored AP engine: engine.executeFlow(flow, context)
  │     └── Engine calls @activepieces/piece-* and @passgrad/piece-* npm packages per step
  ├── On HUMAN_APPROVAL pause:
  │     ├── UPDATE flow_runs SET status='waiting_for_human', assigned_to, resume_token, deadline
  │     ├── pg-boss.sendAfter('escalate-run', { flow_run_id }, deadline)
  │     └── Resend email with approve/reject magic link
  └── On completion: UPDATE flow_runs SET status='completed'|'failed', completed_at

Time-based triggers (reminders, escalations, deadlines):
  → pg-boss built-in cron scheduler (no separate CF Cron needed)
    → Finds overdue runs → enqueues to execute-flow queue
      → Same engine execution path
```

### Process structure on Render

```typescript
// apps/api/src/index.ts
import { serve } from '@hono/node-server'
import { app } from './app'
import { startWorker } from '../../worker/src/index'  // apps/worker — separate workspace

serve(app, (info) => console.log(`API listening on port ${info.port}`))
startWorker() // pg-boss polling loop — runs in same process, non-blocking
```

```typescript
// apps/worker/src/index.ts
import PgBoss from 'pg-boss'
import { executeFlow } from '../../packages/vendor/ap-engine'

const boss = new PgBoss(process.env.DATABASE_URL!)

export async function startWorker() {
  await boss.start()
  await boss.work('execute-flow', async (job) => {
    const { flowId, resumeToken, payload } = job.data
    const version = await loadFlowVersionFromSupabase(flowId)  // SELECT definition FROM flow_versions
    const result = await executeFlow(version.definition, { payload, resumeToken })
    await saveRunResult(result)
  })
}
```

### Job queue: pg-boss on Supabase Postgres

pg-boss (MIT, v12.18.2, actively maintained) stores jobs in a `pgboss` schema inside the existing Supabase Postgres database. No Redis, no separate queue service, no extra cost.

| Feature               | pg-boss support                                                |
| --------------------- | -------------------------------------------------------------- |
| Delayed jobs          | Yes — `sendAfter(name, data, options, future_timestamp)`       |
| Multi-day delays      | Yes — arbitrary future timestamps (no 24h cap)                 |
| Retry + backoff       | Yes — exponential backoff, configurable max attempts           |
| Dead-letter queue     | Yes — explicit DLQ support                                     |
| Cron scheduling       | Yes — built-in cron for time-based triggers                    |
| Exactly-once          | Yes — Postgres `SKIP LOCKED`                                   |
| Transactional enqueue | Yes — enqueue inside a Supabase transaction via Kysely/Drizzle |

**Why not Redis/BullMQ**: Adds $10/mo (Render Key Value Starter) and a second service dependency for no benefit over pg-boss given Supabase Postgres is already present.

**Why not CF Queues**: No longer available — Hono API has moved off CF Workers to Render.

**pg-boss connection string — must be direct or session pooler**: pg-boss uses `LISTEN/NOTIFY` for low-latency job pickup. Supabase's transaction pooler (port 6543) does not support `LISTEN/NOTIFY` or long-lived connections — pg-boss silently falls back to polling (2s interval), degrading latency and increasing DB load. Always use the **direct connection** (port 5432) or **session pooler** (Supavisor, also port 5432) for the `DATABASE_URL` env var passed to pg-boss. The transaction pooler URL (port 6543) is only for short-lived `supabase-js` queries.

### AP flow JSON schema — confirmed

The `FlowVersion.trigger` field is a single self-contained recursive JSON tree containing the entire step graph. It stores and retrieves from Supabase JSONB without transformation. The engine receives it directly.

**Flow schema tables:**

```sql
-- One record per flow (stable identity)
CREATE TABLE flows (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id            UUID NOT NULL,
  status               TEXT NOT NULL DEFAULT 'DISABLED', -- 'ENABLED' | 'DISABLED'
  published_version_id UUID,                              -- FK → flow_versions.id (set on publish)
  created_at           TIMESTAMPTZ DEFAULT now(),
  updated_at           TIMESTAMPTZ DEFAULT now()
);

-- Versioned flow definitions — the step graph lives here as JSONB
CREATE TABLE flow_versions (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  flow_id        UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
  definition     JSONB NOT NULL,           -- full FlowVersion object: trigger + entire step graph
  state          TEXT NOT NULL DEFAULT 'DRAFT', -- 'DRAFT' | 'LOCKED'
  schema_version TEXT DEFAULT '20',        -- AP schema version (currently '20')
  valid          BOOLEAN DEFAULT false,
  created_at     TIMESTAMPTZ DEFAULT now()
);

-- Piece connection credentials (referenced by name in flow step inputs)
CREATE TABLE app_connections (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id  UUID NOT NULL,
  name       TEXT NOT NULL,               -- string referenced in flow step input fields
  piece_name TEXT NOT NULL,               -- e.g. '@activepieces/piece-gmail'
  credentials JSONB NOT NULL,             -- encrypted at rest; access_token, refresh_token, etc.
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (tenant_id, name)
);

-- Webhook/polling trigger registrations (separate from flow definition)
CREATE TABLE trigger_sources (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id       UUID NOT NULL,
  flow_id         UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
  flow_version_id UUID NOT NULL REFERENCES flow_versions(id),
  type            TEXT NOT NULL,           -- 'POLLING' | 'WEBHOOK' | 'MANUAL'
  piece_name      TEXT NOT NULL,
  trigger_name    TEXT NOT NULL,
  schedule        JSONB,                   -- { type: 'CRON_EXPRESSION', cronExpression, timezone }
  deleted_at      TIMESTAMPTZ              -- soft delete
);
```

**What is embedded in `definition` JSONB (self-contained):**
- All step names, types, display names, ordering, nesting
- All `input` field values (raw values or `{{expression}}` strings)
- CODE step source (packageJson + code string inline)
- Router branch conditions and loop item expressions
- Piece names and version ranges

**What requires runtime resolution (handled in the `storage/` adapter seam):**
- `AppConnection` credentials — each step's `input.auth` is a connection name string; the engine calls `storage.getConnection(name, tenantId)` to fetch the actual credentials at execution time

**`sampleData` fields** (per-step test output metadata) — not needed for live execution. Skip `File` table for v1; engine ignores missing sample data at runtime.

**Step graph topology:**
```
trigger (FlowTrigger)
  └── nextAction? → step1
        └── nextAction? → step2
              ├── LOOP: firstLoopAction? → (inner chain)
              └── ROUTER: children[] → [branch0_head, branch1_head, null, ...]
```

The engine traverses this recursive structure. The flow builder UI must emit the same shape — a linear chain for simple flows, nested for loops and branches.

Individual `@activepieces/piece-*` `package.json` files lack a `"license"` field. They are MIT by the root AP LICENSE (which explicitly covers all content outside `packages/ee/`). Automated license scanners (FOSSA, license-checker) will flag them as `UNLICENSED`. Add scanner overrides in CI config early.

### Automation as core product feature

Clients configure their own integrations (WhatsApp, Slack, ERP, SIS) via the copied AP flow builder embedded in the product shell. Monetized by gating:

- Number of active automation flows per tenant
- Automation run count/month (e.g., 1K free, 10K on paid tier) — counted inside the pg-boss consumer before invoking the engine (single choke point)
- Access to specific high-value connectors

### Notification layer

**Resend** (free: 3,000 emails/mo) for system-level transactional email (password reset, onboarding). Automation flows handle workflow-driven notifications — clients configure channels themselves via the flow builder.

---

## 4. Tech Stack

### Single-maintainer constraint

One developer maintains the full platform. This favors:

- One language (TypeScript end-to-end)
- One codebase (monorepo)
- Minimal infrastructure to operate
- Easy local development

### Decision: Vite (React SPA) + Hono on Node.js (Render)

**Why not Next.js**: Overkill. The app is fully authenticated — SEO is irrelevant. Highly interactive (spreadsheet grid, drag-and-drop). A SPA with Supabase Realtime handles this better than SSR.

**Why Hono**:

- Runtime-agnostic — runs on Node.js, CF Workers, Bun, Deno with a one-line adapter swap
- TypeScript-first, minimal overhead
- If we ever need to move back to CF Workers (e.g., if AP engine compatibility is resolved in a future version), the swap is one import change

**Why Render over CF Workers for the API**:

The AP engine requires Node.js 20. Running on Render means one service hosts both the HTTP API and the background queue consumer — no separate worker process, no separate deployment. CF Workers is now only used for static asset delivery (CF Pages), which is free.

### Database access from Render

Render Node.js can connect to Supabase Postgres directly via TCP (unlike CF Workers). Two options remain:

| Option                            | How                  | Notes                                                         |
| --------------------------------- | -------------------- | ------------------------------------------------------------- |
| **Supabase REST API (PostgREST)** | `supabase-js` client | Simpler. RLS enforced automatically via JWT header.           |
| **Direct Postgres (pg/Drizzle)**  | TCP connection       | More flexible for complex queries, joins, formula evaluation. |

**Recommendation**: Use `supabase-js` (PostgREST) for standard CRUD (records, fields, views). Use direct Postgres (`pg` + Drizzle ORM or raw SQL) for complex queries: cross-table formula resolution, bulk imports, aggregations. Both can coexist — `supabase-js` for auth/RLS enforcement, `pg` for power queries where you construct the `tenant_id` filter manually.

### Grid: AG Grid Community — with defined migration path

**Decision: AG Grid Community (MIT, v35.3.0).**

AG Grid Community is backed by AG Grid Ltd (commercial company), actively maintained, and has a rich feature set for spreadsheet-like UIs:

- Built-in filter and sort UI
- DOM-based cell renderers — custom field types (person picker, colored selects, attachments, date calendars) are React components, not canvas drawing code
- Inline editing with extensible editor components
- Virtual scrolling, selection, pagination, master/detail

**Row grouping and aggregation are Enterprise features** in AG Grid — `agGroupCellRenderer`, `aggFunc`, tree data, and pivoting all require the Enterprise license. **Row grouping is not in scope for v1.** Ship without it; Airtable-style filtering + sorting without grouping is fully usable. Add grouping in v1.1 using one of these options:

- **Build a DIY grouping layer** (compute grouped structure in a `useGroupedRows()` hook, render group-header rows as a special row type in AG Grid Community). Realistic scope: 800–1,200 LOC for collapsible groups + per-group aggregates + expansion state per view.
- **Upgrade to AG Grid Enterprise** (billing change, zero code change at the `<DatabaseGrid>` wrapper boundary) when a paying tenant needs grouping or pivoting.

TanStack Table v8 (`getGroupedRowModel()`) is an alternative if AG Grid as a whole becomes the wrong tool, but that is a larger swap.

**Migration path**: If AG Grid Community performance becomes a bottleneck at high row counts (tens of thousands of rows with complex rendering), the next step is AG Grid Enterprise (not Glide Data Grid). The `<DatabaseGrid>` abstraction makes this a billing change only.

**Abstraction rule**: No AG Grid types (`ColDef`, `GridApi`, `ICellRendererParams`, etc.) imported outside `apps/web/src/components/database/grid/`. All cell renderers live in `apps/web/src/components/database/cells/`. The rest of the app only sees `<DatabaseGrid columns={...} data={...} onCellEdit={...} />`.

### Monorepo structure

```
passgrad-worktable-app/
├── apps/
│   ├── web/                          ← Vite + React SPA (Cloudflare Pages)
│   │   └── src/
│   │       ├── components/
│   │       │   ├── database/
│   │       │   │   ├── grid/         ← <DatabaseGrid> wrapper (AG Grid, swappable)
│   │       │   │   └── cells/        ← cell renderers per field type (React components)
│   │       │   └── workflow/         ← task UI, approval forms
│   │       ├── features/
│   │       │   └── automation/       ← COPIED from AP packages/ui/src/features/flows/
│   │       │         ├── canvas/     ← React Flow canvas (kept as-is)
│   │       │         ├── api/        ← AP API calls → replaced with Hono endpoints
│   │       │         └── ...         ← step forms, piece selector, run history (kept as-is)
│   │       └── pages/
│   ├── api/                          ← Hono HTTP server (Render Web Service)
│   │   └── src/
│   │       ├── routes/
│   │       │   ├── databases.ts
│   │       │   ├── records.ts
│   │       │   ├── fields.ts
│   │       │   ├── flows.ts          ← AP-compatible flow definition CRUD
│   │       │   ├── workflows.ts
│   │       │   └── tenants.ts        ← tenant onboarding (POST /tenants)
│   │       ├── middleware/
│   │       │   └── supabase.ts       ← JWT verification + per-request Supabase client
│   │       └── index.ts              ← serve(app) + import startWorker from apps/worker
│   └── worker/                       ← pg-boss consumer + AP engine (separate workspace)
│       └── src/
│           └── index.ts              ← startWorker() — imported by apps/api in same process
│                                        Split into separate deployed service later = start-command change only
├── packages/
│   ├── vendor/
│   │   └── ap-engine/                ← COPIED from AP packages/server/engine/src (MIT)
│   │         ├── executor/           ← flow + step execution logic (kept as-is)
│   │         └── storage/            ← IEngineStorage interface → Supabase implementation
│   ├── core/                         ← Built in-house (field types ~200 lines, filter/sort ~400 lines)
│   │   ├── field-types/
│   │   ├── formula/                  ← HyperFormula (MIT, ~150 KB); thin adapter translates {field_name} → cell address
│   │   └── filter/
│   └── shared-types/                 ← Shared TypeScript types (imported by api + worker + web)
└── supabase/
    └── migrations/                   ← DB migrations incl. Auth Hook + RLS policies
```

**`apps/worker` isolation rule**: `apps/worker` imports from `packages/` only — never from `apps/api`. `apps/api` imports `startWorker` from `apps/worker` for the single-process deployment. When the time comes to split, the start commands diverge:

```bash
# Today — one process
node apps/api/dist/index.js  # runs serve(app) + startWorker()

# Later — two processes, zero code change
node apps/api/dist/index.js    # HTTP only
node apps/worker/dist/index.js # queue consumer only
```

---

## 5. Hosting Map

### Decision: Cloudflare Pages (free) + Render Starter ($7/mo) + Supabase Free

| Service                | What it hosts                                       | Cost       | Notes                                  |
| ---------------------- | --------------------------------------------------- | ---------- | -------------------------------------- |
| **Cloudflare Pages**   | React SPA (static assets)                           | **$0**     | Unlimited static asset requests        |
| **Render Web Service** | Hono API + pg-boss worker + AP engine (one service) | **$7/mo**  | 512 MB RAM, 0.5 CPU, always-on         |
| **Supabase Free**      | Postgres DB + Auth + pg-boss schema + flow JSONB    | **$0**     | 500MB DB, 50K MAU, 2 active projects   |
| **Cloudflare R2**      | File/attachment storage                             | **$0**     | 10GB storage, no egress fees           |
| **Resend**             | System email notifications                          | **$0**     | 3,000 emails/mo                        |
| **Cloudflare Workers** | Not used (API moved to Render)                      | **$0**     | CF Pages does not require Workers plan |
| **Total**              |                                                     | **~$7/mo** |                                        |

### When to upgrade

| Trigger                         | Action                                            | New cost |
| ------------------------------- | ------------------------------------------------- | -------- |
| DB > 500MB or > 50K MAU         | Supabase Pro                                      | +$25/mo  |
| API memory pressure / slow jobs | Render Standard (2 GB RAM)                        | +$18/mo  |
| High automation job volume      | Second Render worker service (dedicated consumer) | +$7/mo   |
| Attachments > 10GB              | R2 $0.015/GB-mo beyond free tier                  | +small   |

### Render operational notes

- **No scale-to-zero**: Render Web Services are always-on. The $7/mo is fixed regardless of traffic. Acceptable at this scale.
- **pg-boss recovery on restart**: If the Render service restarts, pg-boss recovers in-flight jobs cleanly after the visibility timeout. No data loss.
- **pg-boss storage discipline**: Store only a reference (e.g. `flow_id`, `run_id`) in job payloads — never the full flow definition or record data. Full payloads in the `pgboss` schema eat into the 500 MB Supabase free tier fast. Configure `archiveCompletedAfterSeconds` and `deleteAfterDays` from day one.
- **Health check**: Render requires an HTTP health check endpoint for Web Services. Hono serves `GET /health → 200`. Use a free uptime monitor (e.g., UptimeRobot free tier) to ping this endpoint every 5 minutes — this also keeps the Supabase project from auto-pausing.
- **Local development**: `@hono/node-server` + pg-boss against a local Postgres (or Supabase local dev) — no emulators or special tooling needed. Run with `pnpm dev` in `apps/api/`.

### File storage

Use **Cloudflare R2** from day one. 10GB free, zero egress fees. Access control is handled by Hono: verify auth → generate a short-lived R2 presigned URL → return to client. R2 is accessed via the AWS S3-compatible API from Node.js using `@aws-sdk/client-s3`. Supabase Storage not used.

---

## 6. Use-Case Deployment Pattern

```
One codebase (passgrad-worktable-app)
  │
  ├── Option A: Single deployment, all use cases
  │     app.passgrad.com/[tenant-slug]
  │     → Same Supabase project, tenant_id isolation
  │     → Preset templates loaded on tenant creation
  │
  └── Option B: Separate deployment per use case (also fine)
        ├── ams.passgrad.com     → own Supabase project, AMS presets
        ├── kiosk.passgrad.com   → own Supabase project, POS presets
        └── clinic.passgrad.com  → own Supabase project, healthcare presets
              Same Render + Pages deployment, different env vars per deployment
              Each Supabase project stays within free tier (500MB) for early customers
```

**Which to choose**: Start with Option A (simpler). Move to Option B per use-case only when you need clean product separation, different pricing tiers, or compliance requirements. The codebase never changes — just environment variables.

Tenant onboarding for a new university (AMS):

1. Create tenant record in `tenants` table
2. Run preset migration: seed 9 databases (Mahasiswa, Pegawai, etc.) with predefined field schemas
3. Seed default workflow templates (bulk enrollment, course registration)
4. Create admin user

This is a ~30-second operation via `POST /tenants` on the Hono API — same Node.js process, same `supabase-js`, same types. No Supabase Edge Function / second runtime needed.

---

## 7. Permissions, Sharing & i18n

### 7.1 Roles within a tenant

```sql
CREATE TABLE tenant_members (
  tenant_id  UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  user_id    UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  role       TEXT NOT NULL DEFAULT 'member', -- 'owner' | 'admin' | 'member'
  created_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (tenant_id, user_id)
);
```

Three roles for MVP:

| Role | Capabilities |
| ------- | ------------ |
| `owner` | Full access; manages billing; can delete tenant |
| `admin` | Create/edit databases, workflows, flows; manage members; sees all records |
| `member` | Submit forms; complete tasks; sees only their own records |

The Auth Hook (migration 0001) writes both `tenant_id` and `role` into `raw_app_meta_data` at invite time. RLS policies read both from the JWT claim — no extra DB lookup per request. When a member's role changes, revoke their session to force a token refresh.

### 7.2 Row visibility model

`records` table carries a `created_by` column set server-side on insert (never trusted from the client):

```sql
ALTER TABLE records ADD COLUMN created_by UUID REFERENCES auth.users(id);
```

Two RLS policies coexist on `records`:

```sql
-- Admins and owners see all records in their tenant
CREATE POLICY records_admin ON records FOR ALL
  USING (
    tenant_id = ((auth.jwt() -> 'app_metadata') ->> 'tenant_id')::uuid
    AND ((auth.jwt() -> 'app_metadata') ->> 'role') IN ('owner', 'admin')
  );

-- Members see only their own records
CREATE POLICY records_member ON records FOR ALL
  USING (
    tenant_id = ((auth.jwt() -> 'app_metadata') ->> 'tenant_id')::uuid
    AND ((auth.jwt() -> 'app_metadata') ->> 'role') = 'member'
    AND created_by = auth.uid()
  );
```

### 7.3 Sharing model (Lark Base-inspired)

Three sharing levels, each with its own access scope:

| Level | What is shared | Who can access | Implementation |
| ----- | -------------- | -------------- | -------------- |
| **Base** | Entire database | Invited tenant members (by role) | `tenant_members` table + JWT RLS |
| **View (read-only link)** | One view, read-only | Anyone with the link (no login required) | `view_shares` table + public route |
| **Form (public submit)** | One form view | Anyone with the link, submit only | `view_shares` table + `POST /forms/:token/submit` (no auth) |

```sql
CREATE TABLE view_shares (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id  UUID NOT NULL,
  view_id    UUID NOT NULL REFERENCES views(id) ON DELETE CASCADE,
  token      TEXT NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(24), 'base64url'),
  type       TEXT NOT NULL, -- 'readonly' | 'form'
  enabled    BOOLEAN NOT NULL DEFAULT true,
  expires_at TIMESTAMPTZ,   -- null = never expires
  created_by UUID NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

**Read-only public view**: `GET /share/:token` — bypasses auth middleware, loads view config + records filtered by the view's saved filter, returns read-only data. No editing, no record creation.

**Public form submit**: `POST /forms/:token/submit` — bypasses auth middleware, validates payload against field schema, inserts record with `created_by = NULL` and `tenant_id` resolved from `view_shares.tenant_id`. Returns 200 with a configurable success message.

**What is NOT supported (intentional scope limit for v1):**
- Per-column (field-level) permissions — add via `view_shares.hidden_fields JSONB` when needed
- Record-level sharing (share one specific row)
- External collaborator invites with edit rights (members must be provisioned by a tenant admin)

### 7.4 Data model contracts

**Linked records** — separate table, not inline JSONB:

```sql
CREATE TABLE record_links (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id        UUID NOT NULL,
  field_id         UUID NOT NULL REFERENCES fields(id) ON DELETE CASCADE,
  source_record_id UUID NOT NULL REFERENCES records(id) ON DELETE CASCADE,
  target_record_id UUID NOT NULL REFERENCES records(id) ON DELETE CASCADE,
  created_at       TIMESTAMPTZ DEFAULT now(),
  UNIQUE (field_id, source_record_id, target_record_id)
);

CREATE INDEX record_links_source ON record_links (source_record_id, field_id);
CREATE INDEX record_links_target ON record_links (target_record_id);

-- Cross-tenant integrity: both source and target records must belong to the same tenant as the link
ALTER TABLE records ADD CONSTRAINT records_id_tenant_uk UNIQUE (id, tenant_id);
ALTER TABLE record_links ADD CONSTRAINT record_links_source_tenant_fk
  FOREIGN KEY (source_record_id, tenant_id) REFERENCES records(id, tenant_id);
ALTER TABLE record_links ADD CONSTRAINT record_links_target_tenant_fk
  FOREIGN KEY (target_record_id, tenant_id) REFERENCES records(id, tenant_id);
```

`records.data` for a linked field is `null` or omitted. Links live entirely in `record_links`. Cross-database `LOOKUP`/`ROLLUP` formulas resolve via a JOIN on this table — no JSONB scan.

**Attachments** — separate table, not inline JSONB:

```sql
CREATE TABLE attachments (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id  UUID NOT NULL,
  record_id  UUID NOT NULL REFERENCES records(id) ON DELETE CASCADE,
  field_id   UUID NOT NULL REFERENCES fields(id) ON DELETE CASCADE,
  r2_key     TEXT NOT NULL,        -- 'tenants/{tenant_id}/attachments/{uuid}.{ext}'
  filename   TEXT NOT NULL,
  mime_type  TEXT NOT NULL,
  size_bytes BIGINT NOT NULL,
  created_by UUID NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX attachments_record_field ON attachments (record_id, field_id);

-- Cross-tenant integrity: attachment must belong to the same tenant as its record
ALTER TABLE attachments ADD CONSTRAINT attachments_record_tenant_fk
  FOREIGN KEY (record_id, tenant_id) REFERENCES records(id, tenant_id);
```

`records.data` for an attachment field is `null`. Presigned URL: `GET /attachments/:id/url` — verifies auth → generates R2 signed URL → returns it. Delete: `DELETE /attachments/:id` — deletes R2 object then row. R2 key format ensures no cross-tenant path collisions.

**Select options** — store IDs, not labels:

```json
{ "field_uuid": "opt_uuid_active" }
```

`fields.config` holds option definitions:

```json
{
  "options": [
    { "id": "opt_uuid_active",   "label": "Active",   "color": "green" },
    { "id": "opt_uuid_inactive", "label": "Inactive", "color": "red"   }
  ]
}
```

Renaming a label is a single `UPDATE fields SET config = ... WHERE id = ?`. Zero record backfill. The field-create handler validates incoming option IDs against `fields.config.options` before writing.

**View config shape** — locked TypeScript type in `packages/shared-types/`:

```typescript
// packages/shared-types/src/view.ts
export type FilterCondition = {
  fieldId: string
  operator: 'eq' | 'neq' | 'contains' | 'gt' | 'lt' | 'gte' | 'lte' | 'is_empty' | 'is_not_empty'
  value: unknown
}

export type FilterGroup = {
  conjunction: 'and' | 'or'
  conditions: (FilterCondition | FilterGroup)[]
}

export type SortItem = {
  fieldId: string
  direction: 'asc' | 'desc'
}

export type ViewConfig = {
  hiddenFields:     string[]       // field UUIDs
  filters:          FilterGroup    // recursive filter tree
  sort:             SortItem[]
  groupBy?:         string         // field UUID
  kanbanStackBy?:   string         // field UUID (single_select)
  calendarDateField?: string       // field UUID (date)
  rowHeight?:       'short' | 'medium' | 'tall'
}
```

Default: `{ hiddenFields: [], filters: { conjunction: 'and', conditions: [] }, sort: [] }`.

### 7.5 Unified flow model (approval = node, form = trigger)

**Decision: drop `workflows`/`tasks`/`task_actions`. Everything is a flow.**

AP already has all the primitives:
- Form submission → a trigger piece scoped to a `databases`/`views` row (built in-house, see §7.5.1)
- Human approval pause → `@passgrad/piece-approval` (our own AP piece, see §7.5.2)
- Deadline escalation → `pg-boss.sendAfter()` from within the approval piece handler
- Approve/reject branching → AP ROUTER step

A "workflow" is just a flow whose definition happens to contain one or more approval nodes. There is no separate concept.

**What survives:**

| Before | After |
|---|---|
| `flows` + `flow_versions` | unchanged |
| `workflows` table | **dropped** |
| `tasks` table | **dropped** — "My Tasks" is a query over `flow_runs` |
| `task_actions` table | **dropped** — step resolution events stored in `flow_run_steps` |
| `automations` as a separate sidebar section | **dropped** — one "Flows" section covers both |

**Sidebar:**
```
📁 Databases
⚡ Flows          ← single editor; templates distinguish approval vs automation
✅ My Tasks       ← derived view: flow_runs WHERE status='waiting_for_human' AND assigned_to=me
```

**`flow_runs` table** (new — covers both approval pauses and automation runs):

```sql
CREATE TABLE flow_runs (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id        UUID NOT NULL,
  flow_id          UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
  flow_version_id  UUID NOT NULL REFERENCES flow_versions(id),
  status           TEXT NOT NULL DEFAULT 'running',
  -- 'running' | 'waiting_for_human' | 'completed' | 'failed' | 'cancelled'
  trigger_payload  JSONB,          -- form submission data / record snapshot / webhook body
  current_step_id  TEXT,           -- node id within definition step graph
  assigned_to      UUID REFERENCES auth.users(id), -- set when status='waiting_for_human'
  deadline         TIMESTAMPTZ,    -- set by the approval piece; triggers pg-boss escalation job
  resume_token     TEXT UNIQUE,    -- signed token sent in approval email link
  step_outputs     JSONB DEFAULT '{}', -- accumulated outputs keyed by step id
  submitted_by     UUID REFERENCES auth.users(id), -- original submitter (for "My Submissions" view)
  created_at       TIMESTAMPTZ DEFAULT now(),
  updated_at       TIMESTAMPTZ DEFAULT now(),
  completed_at     TIMESTAMPTZ
);

CREATE INDEX flow_runs_tenant_status ON flow_runs (tenant_id, status);
CREATE INDEX flow_runs_assigned_to   ON flow_runs (assigned_to) WHERE status = 'waiting_for_human';
CREATE INDEX flow_runs_submitted_by  ON flow_runs (submitted_by);

-- Cross-tenant integrity: flow_run must belong to the same tenant as its flow
ALTER TABLE flows ADD CONSTRAINT flows_id_tenant_uk UNIQUE (id, tenant_id);
ALTER TABLE flow_runs ADD CONSTRAINT flow_runs_flow_tenant_fk
  FOREIGN KEY (flow_id, tenant_id) REFERENCES flows(id, tenant_id);

-- RLS: members see only their own runs (as submitter or assignee); admins see all in tenant
CREATE POLICY flow_runs_admin ON flow_runs FOR ALL
  USING (
    tenant_id = ((auth.jwt() -> 'app_metadata') ->> 'tenant_id')::uuid
    AND ((auth.jwt() -> 'app_metadata') ->> 'role') IN ('owner', 'admin')
  );

CREATE POLICY flow_runs_member ON flow_runs FOR SELECT
  USING (
    tenant_id = ((auth.jwt() -> 'app_metadata') ->> 'tenant_id')::uuid
    AND (submitted_by = auth.uid() OR assigned_to = auth.uid())
  );
```

**Derived views (no extra tables):**

```sql
-- "My Tasks" inbox
CREATE VIEW my_tasks AS
  SELECT * FROM flow_runs
  WHERE status = 'waiting_for_human'
    AND assigned_to = auth.uid();

-- "My Submissions" (what I started)
CREATE VIEW my_submissions AS
  SELECT * FROM flow_runs
  WHERE submitted_by = auth.uid();
```

**Human-in-the-loop revised flow:**

```
User submits form → POST /forms/:token/submit
  → insert flow_runs row (status='running', submitted_by=user)
  → pg-boss.send('execute-flow', { flow_run_id })

pg-boss consumer executes flow until approval node:
  → engine pauses at HUMAN_APPROVAL step
  → UPDATE flow_runs SET status='waiting_for_human', assigned_to=<resolved>, resume_token=<signed>, deadline=<computed>
  → pg-boss.sendAfter('escalate-run', { flow_run_id }, deadline)   ← optional escalation
  → Resend email to assignee with approve/reject link containing resume_token

Assignee clicks approve link → POST /flows/runs/resume?token=<resume_token>&decision=approve
  → pg-boss.cancel(escalation_job_id)
  → UPDATE flow_runs SET status='running', assigned_to=null
  → pg-boss.send('execute-flow', { flow_run_id, resume_token, decision })
  → engine continues from next step
```

#### 7.5.1 Custom form trigger piece

AP's built-in Forms trigger is generic. Build `@passgrad/piece-database-form` — a trigger that binds to a specific `view_id` (form view in our database engine):

```typescript
// packages/pieces/database-form/src/index.ts
export const databaseFormTrigger = createTrigger({
  name: 'on_form_submit',
  displayName: 'Form Submitted',
  type: TriggerStrategy.WEBHOOK,
  props: {
    view_id: Property.Dropdown({
      displayName: 'Form View',
      description: 'Select a form view from your databases',
      required: true,
      refreshers: [],
      options: async (ctx) => {
        // calls GET /views?type=form to populate dropdown
      },
    }),
  },
  async onEnable(ctx) { /* register webhook in trigger_sources */ },
  async onDisable(ctx) { /* deregister */ },
  async run(ctx) { return [ctx.payload.body] },
})
```

`POST /forms/:token/submit` resolves the view's flow, fires the trigger payload into the pg-boss queue. Anonymous submissions set `submitted_by = null`.

#### 7.5.2 Approval piece

**Decision: use AP's built-in `@activepieces/piece-approval`** — it covers the v1 case (user X clicks approve/reject via a link). No custom piece needed.

The built-in piece:
- Pauses the flow and returns a `{ approveUrl, rejectUrl }` pair
- The URLs contain a signed resume token that calls back to our Hono endpoint
- The step before it (e.g. a Send Email step) delivers the links to the assignee

**Assignee resolution** happens in the step *before* the approval node — a Send Email step addressed to `{{trigger.output.advisor_email}}` or a static address. The approval piece itself does not need to know who the assignee is; it just pauses and waits.

This maps directly to `flow_runs.assigned_to` being set by our pg-boss consumer when it detects the pause state — we read the `assignee` from the Send Email step's resolved input and write it to `flow_runs`.

**What is deferred to v2**: role-based fan-out (send to all admins, first to respond wins), delegation, and custom approval forms. These require a custom `@passgrad/piece-approval`. The hook point is already designed — just swap the piece when needed.

#### 7.5.3 Template wizard

The wizard is a constrained editor over `flow_versions.definition` JSONB. It writes AP-compatible flow JSON. The AP canvas opens the same row — switching is lossless.

**v1 wizard templates:**
1. **Linear approval** — Form trigger → Send Email (to assignee) → AP Approval node → branch on approve/reject → Send Email on complete
2. **Scheduled reminder** — Schedule trigger → Send email/WhatsApp
3. **Record webhook** — Record created/updated trigger → HTTP request or Slack message

The wizard emits these shapes; the canvas lets admins extend or customize. Both write to `flow_versions.definition`.

**"My Tasks" and "My Submissions" are separate UI surfaces** — both read from `flow_runs` with different filters, presented as distinct pages:
- **My Tasks** (`/tasks`): `flow_runs WHERE status='waiting_for_human' AND assigned_to=me` — action required by me
- **My Submissions** (`/submissions`): `flow_runs WHERE submitted_by=me` — status tracking of things I started

### 7.6 Preset format

TypeScript files in `apps/api/src/presets/`, versioned with code. `POST /tenants` calls `seedPreset(tenantId, preset)` — typed, diffable, no separate UI needed.

```typescript
// apps/api/src/presets/ams.ts
export const AMS_PRESET = {
  databases: [
    { name: 'Mahasiswa', icon: '🎓', fields: [ /* ... */ ] },
    { name: 'Pegawai',   icon: '👤', fields: [ /* ... */ ] },
  ],
  flows: [
    // Pre-built flow_versions.definition JSON for common AMS patterns
    { name: 'Pendaftaran Mahasiswa Baru', template: 'linear_approval', steps: 3 },
    { name: 'Pengajuan Cuti',             template: 'linear_approval', steps: 2 },
  ],
}
```

Preset changes apply to new tenants only. Migrate to a `presets` table when platform admins need to edit templates in a UI.

### 7.7 Internationalisation (i18n)

**English + Indonesian from day one.** All UI copy uses translation keys — no hardcoded strings in components.

Stack: `react-i18next` + `i18next`. Two locale files:

```
apps/web/src/i18n/
  en.json   ← English (default)
  id.json   ← Indonesian (Bahasa Indonesia)
```

Rules:
- All component copy uses `t('key')` from the first component written — no retrofitting
- Preset data (database names, field labels, workflow names) is stored localised in the database — the `ams.ts` preset seeds Indonesian names directly into `fields.name`, `databases.name`, etc.
- Date/number formatting uses `Intl.DateTimeFormat` / `Intl.NumberFormat` with the user's locale preference stored in `profiles.locale` (`'en'` or `'id'`)
- The locale switcher is a single dropdown in the app shell; preference persisted to `profiles.locale` + `localStorage` fallback for unauthenticated form pages

```sql
ALTER TABLE profiles ADD COLUMN locale TEXT NOT NULL DEFAULT 'id';
```

Indonesian is the default locale (primary market). English is the fallback.

---

## Open Questions (for later)

1. **Formula engine**: ~~Teable ANTLR4 vendoring~~ — **decided: HyperFormula** (MIT). Adapter in `packages/core/formula.ts`. **v1 scope: single-record arithmetic only** — `SUM`, `IF`, `CONCAT`, date math, etc. No `LOOKUP`/`ROLLUP` cross-record formulas. Those require loading referenced record sets into HF workbooks per request (real latency + memory cost); deferred to v2 with a caching strategy.
2. **Real-time collaboration**: Supabase Realtime (free tier: **200 concurrent connections**, not 200 MAU). Limit bites at busy-hours concurrent users, not total users. Throttle Realtime to admin-facing screens for v1; budget for Supabase Pro at first paying tenant if a university hits this with students on dashboards concurrently.
3. **Offline support**: Not in scope for MVP. Note for healthcare use case (clinics with spotty internet).
4. **Compliance**: Healthcare deployments may require separate Supabase project for data isolation. Decide per client contract, not upfront.
5. **Bulk import performance**: CSV import of 10K student records via JSONB insert — benchmark before launch. May need batching through the pg-boss queue (enqueue as a background job, stream results).
6. **Automation run metering**: Count runs inside the pg-boss consumer before invoking the engine (single choke point). Write `run_count` increments to Supabase per tenant per billing period.
7. **AP upstream sync strategy**: Maintain `VENDOR.md` logging the AP git commit each vendor copy was taken from. Review quarterly. Only sync `executor/` — never sync `storage/` (that is our adapter layer).
8. **Direct Postgres vs PostgREST**: For complex formula resolution and cross-table aggregations, `pg` + raw SQL is more flexible. Establish which query patterns warrant direct Postgres vs `supabase-js` per route.
9. **Render service split**: If job processing becomes CPU-heavy (many concurrent AP flows), split `apps/api` and `apps/worker` into two Render services. Same codebase, different start commands (`serve(app)` vs `startWorker()`). Cost: +$7/mo.
10. **Grid grouping (v1.1)**: Not in v1. Two implementation paths when needed: (a) DIY grouping layer in `useGroupedRows()` hook (~1,000 LOC), (b) upgrade to AG Grid Enterprise (billing only, zero wrapper code change). Decision deferred until a paying tenant requests it.
11. ~~**AP flow JSON schema**~~ — **Resolved**.
12. ~~**Approval piece — custom vs built-in**~~ — **Resolved**: AP built-in for v1.
13. ~~**Template wizard scope**~~ — **Resolved**: 3 templates for v1.
14. ~~**Assignee expression syntax**~~ — **Resolved**: built-in approval piece, assignee via Send Email step.
15. ~~**My Tasks vs My Submissions**~~ — **Resolved**: separate pages, both query `flow_runs`.

---

## 8. Security, Operational, and Multi-tenant Concerns

### 8.1 AP pieces — trust model and allowlist

**`AP_EXECUTION_MODE=UNSANDBOXED` means every imported piece runs in the same Node process as Hono**, with access to `process.env`, outbound HTTP, and the filesystem. The service role key (if present) would be readable by any piece.

**Mitigations — all must be in place before first piece is added:**

1. **No service role key in the API process.** Use the anon key + per-request JWT for all tenant-scoped queries. Cross-tenant operations (tenant onboarding, run metering, orphan cleanup) go through `SECURITY DEFINER` Postgres functions, not service-role Node calls.

2. **Piece allowlist.** Only install pieces that have been reviewed. v1 allowlist:
   ```
   @activepieces/piece-approval
   @activepieces/piece-http           (generic HTTP request)
   @activepieces/piece-gmail
   @activepieces/piece-slack
   @activepieces/piece-schedule       (cron trigger)
   @activepieces/piece-webhook        (webhook trigger)
   @passgrad/piece-database-form      (our own)
   ```
   Adding a new piece requires a PR with: license check, `pnpm audit`, and explicit approval. No `@activepieces/piece-shell`, `piece-script`, or `piece-code` in production.

3. **Pin exact versions.** No `^` or `~` in `packages/vendor/ap-engine/package.json`. Run `pnpm audit --audit-level=moderate` in CI on every PR.

4. **Block filesystem access.** Render Web Service runs under a non-root user by default. Don't mount any volumes. Pieces that attempt filesystem writes will fail silently or with a permission error — acceptable.

### 8.2 JWT TTL and role-change procedure

**JWT TTL: set to 15 minutes** (Supabase Dashboard → Auth → JWT expiry). Refresh tokens remain long-lived (default 7 days). Worst-case stale role window = 15 minutes.

**Role-change procedure:**
1. Update `tenant_members SET role = ?` WHERE ...
2. Call `supabase.auth.admin.signOut(user_id, { scope: 'global' })` — invalidates all refresh tokens. The user's next API call will fail with 401, forcing re-login and a fresh JWT with the new role.
3. Note: the current access token (up to 15 min remaining) is still valid for in-flight requests. This is acceptable given the TTL.

### 8.3 Multi-tenant user switching

A user can belong to multiple tenants (`tenant_members` is many-to-many), but the JWT carries a single `tenant_id`. Switching tenants requires a new JWT.

**Flow: `POST /sessions/switch-tenant`**

```typescript
// apps/api/src/routes/sessions.ts
app.post('/sessions/switch-tenant', async (c) => {
  const { tenant_id } = await c.req.json()
  const user_id = c.get('jwtPayload').sub

  // Verify membership via a SECURITY DEFINER function (no service role key in Node)
  const { data } = await c.get('supabase').rpc('get_user_tenant_role', {
    p_user_id: user_id,
    p_tenant_id: tenant_id,
  })
  if (!data?.role) return c.json({ error: 'not a member' }, 403)

  // Re-issue JWT via Supabase Admin API (requires service role — called from a Postgres function)
  // Alternative: client calls supabase.auth.refreshSession() after server updates app_metadata
  // Practical v1 implementation: update raw_app_meta_data in a SECURITY DEFINER function,
  // return 200, client calls supabase.auth.refreshSession() to pick up new claims.
  await c.get('supabase').rpc('switch_user_tenant', {
    p_user_id: user_id,
    p_tenant_id: tenant_id,
    p_role: data.role,
  })
  return c.json({ ok: true, message: 'Refresh your session to activate new tenant context.' })
})
```

```sql
-- SECURITY DEFINER function — writes new tenant_id + role into app_metadata
-- No service role key needed in Node; this runs with elevated DB privileges only
CREATE OR REPLACE FUNCTION public.switch_user_tenant(
  p_user_id   UUID,
  p_tenant_id UUID,
  p_role      TEXT
) RETURNS void AS $$
BEGIN
  UPDATE auth.users
  SET raw_app_meta_data = raw_app_meta_data ||
    jsonb_build_object('tenant_id', p_tenant_id::text, 'role', p_role)
  WHERE id = p_user_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

After the server call succeeds, the client calls `supabase.auth.refreshSession()`. The refresh token exchange returns a new access JWT with the updated `tenant_id`/`role` claims.

### 8.4 resume_token signing

The `resume_token` in approval emails is the only thing standing between an attacker and approving any pending flow run. Spec:

- **Algorithm**: HMAC-SHA256 over `{flow_run_id}:{exp_unix_ts}` with `APPROVAL_TOKEN_SECRET` from env.
- **Expiry**: 7 days default (configurable via `APPROVAL_TOKEN_TTL_DAYS` env).
- **Single-use**: on first valid redemption, set `flow_runs.resume_token = NULL` (or a `consumed_at` timestamp). A second request with the same token returns 409.
- **Rotation**: if `APPROVAL_TOKEN_SECRET` is rotated, all pending tokens are immediately invalidated. Acceptable — pending approvals will receive a "link expired" error and can request a resend. Document this in runbooks.

```typescript
// apps/api/src/lib/resume-token.ts
import { createHmac } from 'crypto'

export function signResumeToken(flowRunId: string, expUnix: number): string {
  const payload = `${flowRunId}:${expUnix}`
  const sig = createHmac('sha256', process.env.APPROVAL_TOKEN_SECRET!)
    .update(payload).digest('hex')
  return Buffer.from(`${payload}:${sig}`).toString('base64url')
}

export function verifyResumeToken(token: string): { flowRunId: string } | null {
  try {
    const decoded = Buffer.from(token, 'base64url').toString()
    const [flowRunId, expStr, sig] = decoded.split(':')
    const exp = parseInt(expStr, 10)
    if (Date.now() / 1000 > exp) return null
    const expected = createHmac('sha256', process.env.APPROVAL_TOKEN_SECRET!)
      .update(`${flowRunId}:${expStr}`).digest('hex')
    if (sig !== expected) return null
    return { flowRunId }
  } catch { return null }
}
```

### 8.5 Attachment orphan cleanup

When a record is deleted, `attachments` rows are cascade-deleted by FK — but the R2 objects are not. A pg-boss cron job runs nightly to sweep orphaned objects:

```typescript
// apps/worker/src/jobs/attachment-sweeper.ts
// Registered as: boss.schedule('attachment-sweeper', '0 3 * * *')  ← 3 AM daily
export async function sweepOrphanedAttachments(supabase, r2) {
  // Find attachments soft-deleted or whose record no longer exists
  const { data: orphans } = await supabase
    .from('attachments')
    .select('id, r2_key')
    .is('record_id', null)   // after ON DELETE SET NULL variant, or use a deleted_at column

  for (const orphan of orphans ?? []) {
    await r2.send(new DeleteObjectCommand({ Bucket: R2_BUCKET, Key: orphan.r2_key }))
    await supabase.from('attachments').delete().eq('id', orphan.id)
  }
}
```

Alternative: add `attachments.deleted_at TIMESTAMPTZ` (soft delete), sweep rows where `deleted_at < now() - interval '1 day'`.

### 8.6 Inbound webhook route

`trigger_sources` rows with `type = 'WEBHOOK'` need an inbound URL. Route:

```
POST /webhooks/:trigger_source_id
```

No auth middleware on this route (webhooks are called by external services). Payload validation is the piece's responsibility. The route:
1. Looks up `trigger_sources` by ID (public, no tenant auth needed).
2. Validates that `deleted_at IS NULL` and `flows.status = 'ENABLED'`.
3. Enqueues `pg-boss.send('execute-flow', { flow_id, trigger_source_id, payload: req.body })`.
4. Returns `200 { received: true }` immediately (do not wait for execution).

### 8.7 Render RAM budget

Render Starter: 512 MB RAM. Realistic steady-state estimate for this process:

| Component | ~RAM |
|---|---|
| Node.js runtime | 80 MB |
| Hono + supabase-js + AWS SDK + Resend | 100 MB |
| pg-boss + active job buffer | 40 MB |
| AP engine + installed pieces (~8 packages) | 60 MB |
| Request headroom (JSONB buffers, per-request HF) | 80 MB |
| **Total** | **~360 MB** |

Headroom: ~150 MB. This is adequate for normal load but thin for bulk import spikes.

**Guardrails from day one:**
- Set `boss.work('execute-flow', { teamSize: 2 })` — max 2 concurrent flow executions. Prevents runaway memory on burst.
- Add Render memory alert at 420 MB.
- Upgrade trigger: sustained P95 memory > 450 MB → upgrade to Render Standard (2 GB, $18/mo).
- Bulk CSV import goes through pg-boss queue with a single-concurrency job type to prevent OOM.
