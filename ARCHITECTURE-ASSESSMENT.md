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
| Auth / DB isolation     | Supabase Auth + RLS with anon key + JWT forwarding through Hono                                       |
| Grid                    | AG Grid Community (MIT) — migrate to Glide Data Grid if performance limits hit                       |
| Hosting                 | Cloudflare Pages (free) + Render Starter ($7/mo) + Supabase free tier                                |
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
CREATE INDEX records_data_gin ON records USING GIN (data);

-- Autonumber state per field
CREATE TABLE field_sequences (
  field_id UUID PRIMARY KEY REFERENCES fields(id) ON DELETE CASCADE,
  next_value BIGINT NOT NULL DEFAULT 1
);
```

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
import { jwt } from 'hono/jwt'
import { createClient } from '@supabase/supabase-js'

// 1. Verify JWT using Supabase JWT secret
app.use('*', (c, next) =>
  jwt({ secret: c.env.SUPABASE_JWT_SECRET })(c, next)
)

// 2. Per-request Supabase client — forwards JWT so RLS fires
app.use('*', async (c, next) => {
  const supabase = createClient(c.env.SUPABASE_URL, c.env.SUPABASE_ANON_KEY, {
    global: { headers: { Authorization: c.req.header('Authorization') ?? '' } },
    auth: { persistSession: false, autoRefreshToken: false, detectSessionFromUrl: false },
  })
  c.set('supabase', supabase)
  await next()
})
```

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE databases ENABLE ROW LEVEL SECURITY;
ALTER TABLE records ENABLE ROW LEVEL SECURITY;

-- Policy: users can only access their own tenant's data
CREATE POLICY tenant_isolation ON records
  USING (tenant_id = (SELECT tenant_id FROM profiles WHERE id = auth.uid()));
```

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

| Activepieces layer            | Replaced with                                            |
| ----------------------------- | -------------------------------------------------------- |
| Their server API (NestJS)     | Hono API — owns flow definitions + execution state       |
| Their Postgres/Redis          | Supabase (flow JSONB) + pg-boss on Supabase Postgres     |
| Their auth system             | Our existing Supabase auth                               |
| Their app shell (sidebar/nav) | Our React app shell                                      |
| Their Docker container        | Nothing — engine runs in-process inside Render service   |

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
User action (submit step / approve / reject)
  → Hono API (POST /tasks/:id/action)  [Render Node.js]
      ├── Write action to Supabase (audit trail)
      ├── pg-boss.send('execute-flow', { flow_id, payload })
      └── Return 200 immediately

pg-boss consumer (same Render process, background loop):
  ├── Deserialize flow definition from Supabase JSONB
  ├── Call vendored AP engine: engine.executeFlow(flow, context)
  │     └── Engine calls @activepieces/piece-* npm packages per step
  ├── Write run result + step outputs to Supabase (run_logs table)
  └── PATCH /tasks/:id/state (internal call or direct Supabase write)

Human-in-the-loop (approval steps):
  → Engine writes { status: 'waiting_for_human', resume_token } to Supabase
  → pg-boss job completes (acknowledged — no re-delivery)
  → Optional escalation: pg-boss.sendAfter('escalate', payload, deadline_timestamp)
  → Human approves → POST /tasks/:id/approve
      ├── pg-boss.cancel(escalation_job_id)  ← cancel escalation if pending
      └── pg-boss.send('execute-flow', { flow_id, resume_token, decision })

Time-based triggers (reminders, escalations, deadlines):
  → pg-boss built-in cron scheduler (no separate CF Cron needed)
    → Finds overdue tasks → enqueues to execute-flow queue
      → Same engine execution path
```

### Process structure on Render

```typescript
// apps/api/src/index.ts
import { serve } from '@hono/node-server'
import { app } from './app'
import { startWorker } from './queues/worker'

serve(app, (info) => console.log(`API listening on port ${info.port}`))
startWorker() // pg-boss polling loop — runs in same process, non-blocking
```

```typescript
// apps/api/src/queues/worker.ts
import PgBoss from 'pg-boss'
import { executeFlow } from '../../../packages/vendor/ap-engine'

const boss = new PgBoss(process.env.DATABASE_URL)

export async function startWorker() {
  await boss.start()
  await boss.work('execute-flow', async (job) => {
    const { flowId, resumeToken, payload } = job.data
    const flow = await loadFlowFromSupabase(flowId)
    const result = await executeFlow(flow, { payload, resumeToken })
    await saveRunResult(result)
  })
}
```

### Job queue: pg-boss on Supabase Postgres

pg-boss (MIT, v12.18.2, actively maintained) stores jobs in a `pgboss` schema inside the existing Supabase Postgres database. No Redis, no separate queue service, no extra cost.

| Feature              | pg-boss support                                               |
| -------------------- | ------------------------------------------------------------- |
| Delayed jobs         | Yes — `sendAfter(name, data, options, future_timestamp)`      |
| Multi-day delays     | Yes — arbitrary future timestamps (no 24h cap)                |
| Retry + backoff      | Yes — exponential backoff, configurable max attempts          |
| Dead-letter queue    | Yes — explicit DLQ support                                    |
| Cron scheduling      | Yes — built-in cron for time-based triggers                   |
| Exactly-once         | Yes — Postgres `SKIP LOCKED`                                  |
| Transactional enqueue| Yes — enqueue inside a Supabase transaction via Kysely/Drizzle|

**Why not Redis/BullMQ**: Adds $10/mo (Render Key Value Starter) and a second service dependency for no benefit over pg-boss given Supabase Postgres is already present.

**Why not CF Queues**: No longer available — Hono API has moved off CF Workers to Render.

### AP pieces license note

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

| Option                            | How                  | Notes                                                              |
| --------------------------------- | -------------------- | ------------------------------------------------------------------ |
| **Supabase REST API (PostgREST)** | `supabase-js` client | Simpler. RLS enforced automatically via JWT header.               |
| **Direct Postgres (pg/Drizzle)**  | TCP connection       | More flexible for complex queries, joins, formula evaluation.     |

**Recommendation**: Use `supabase-js` (PostgREST) for standard CRUD (records, fields, views). Use direct Postgres (`pg` + Drizzle ORM or raw SQL) for complex queries: cross-table formula resolution, bulk imports, aggregations. Both can coexist — `supabase-js` for auth/RLS enforcement, `pg` for power queries where you construct the `tenant_id` filter manually.

### Grid: AG Grid Community — with defined migration path

**Decision: AG Grid Community (MIT, v35.3.0).**

AG Grid Community is backed by AG Grid Ltd (commercial company), actively maintained, and has the richest out-of-the-box feature set for a spreadsheet-like no-code product:

- Full row grouping + aggregation (built-in, Community edition)
- Built-in filter and sort UI
- DOM-based cell renderers — custom field types (person picker, colored selects, attachments, date calendars) are React components, not canvas drawing code
- Inline editing with extensible editor components

**Migration path to Glide Data Grid**: If AG Grid Community performance becomes a bottleneck at high row counts (tens of thousands of rows with complex rendering), migrate to `@glideapps/glide-data-grid`. This is bounded by the `<DatabaseGrid>` abstraction layer — all grid-specific code lives inside `apps/web/src/components/database/grid/` and cell renderer files. The swap cost is 2–5 days given clean isolation. Row grouping and filter/sort UI will need custom implementation in Glide (not built-in).

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
│   └── api/                          ← Hono app + pg-boss worker (Render Node.js)
│       └── src/
│           ├── routes/
│           │   ├── databases.ts
│           │   ├── records.ts
│           │   ├── fields.ts
│           │   ├── flows.ts          ← AP-compatible flow definition CRUD
│           │   └── workflows.ts
│           ├── queues/
│           │   └── worker.ts         ← pg-boss consumer + AP engine calls (same process)
│           ├── middleware/
│           │   └── supabase.ts       ← JWT verification + per-request Supabase client
│           └── index.ts              ← serve(app) + startWorker()
├── packages/
│   ├── vendor/
│   │   └── ap-engine/                ← COPIED from AP packages/server/engine/src (MIT)
│   │         ├── executor/           ← flow + step execution logic (kept as-is)
│   │         └── storage/            ← IEngineStorage interface → Supabase implementation
│   ├── core/                         ← Vendored from Teable packages/core (MIT)
│   │   ├── field-types/
│   │   ├── formula/                  ← ANTLR4-based (antlr4ts — pure JS, no Node-only deps)
│   │   └── filter/
│   └── shared-types/                 ← Shared TypeScript types
└── supabase/
    ├── functions/                    ← Edge Functions (tenant onboarding, health ping)
    └── migrations/
```

**Copy-and-adapt rule**: When copying from Activepieces source, only touch files in designated adapter seams (`api/`, `storage/`). Leave execution logic, UI canvas, and step forms untouched to keep upstream diffs minimal.

**AP upstream sync**: Maintain a `VENDOR.md` log recording which AP git commit each vendored copy was taken from. Review AP releases quarterly for engine bug fixes and new pieces.

---

## 5. Hosting Map

### Decision: Cloudflare Pages (free) + Render Starter ($7/mo) + Supabase Free

| Service                  | What it hosts                                        | Cost       | Notes                                   |
| ------------------------ | ---------------------------------------------------- | ---------- | --------------------------------------- |
| **Cloudflare Pages**     | React SPA (static assets)                            | **$0**     | Unlimited static asset requests         |
| **Render Web Service**   | Hono API + pg-boss worker + AP engine (one service)  | **$7/mo**  | 512 MB RAM, 0.5 CPU, always-on          |
| **Supabase Free**        | Postgres DB + Auth + pg-boss schema + flow JSONB     | **$0**     | 500MB DB, 50K MAU, 2 active projects    |
| **Cloudflare R2**        | File/attachment storage                              | **$0**     | 10GB storage, no egress fees            |
| **Resend**               | System email notifications                           | **$0**     | 3,000 emails/mo                         |
| **Cloudflare Workers**   | Not used (API moved to Render)                       | **$0**     | CF Pages does not require Workers plan  |
| **Total**                |                                                      | **~$7/mo** |                                         |

### When to upgrade

| Trigger                        | Action                                          | New cost   |
| ------------------------------ | ----------------------------------------------- | ---------- |
| DB > 500MB or > 50K MAU        | Supabase Pro                                    | +$25/mo    |
| API memory pressure / slow jobs| Render Standard (2 GB RAM)                      | +$18/mo    |
| High automation job volume     | Second Render worker service (dedicated consumer)| +$7/mo    |
| Attachments > 10GB             | R2 $0.015/GB-mo beyond free tier                | +small     |

### Render operational notes

- **No scale-to-zero**: Render Web Services are always-on. The $7/mo is fixed regardless of traffic. Acceptable at this scale.
- **pg-boss recovery on restart**: If the Render service restarts, pg-boss recovers in-flight jobs cleanly after the visibility timeout. No data loss.
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

This is a ~30-second operation via a Supabase Edge Function, not a manual process.

---

## Open Questions (for later)

1. **Formula engine**: Teable's `packages/core/formula` uses ANTLR4 (`antlr4ts` — pure JS, runs in any environment). Confirmed viable for vendoring. Assess complexity vs. a simpler expression evaluator when implementing formula fields.
2. **Real-time collaboration**: Supabase Realtime (free, 200 concurrent connections) for live record updates. Sufficient for initial use cases.
3. **Offline support**: Not in scope for MVP. Note for healthcare use case (clinics with spotty internet).
4. **Compliance**: Healthcare deployments may require separate Supabase project for data isolation. Decide per client contract, not upfront.
5. **Bulk import performance**: CSV import of 10K student records via JSONB insert — benchmark before launch. May need batching through the pg-boss queue (enqueue as a background job, stream results).
6. **Automation run metering**: Count runs inside the pg-boss consumer before invoking the engine (single choke point). Write `run_count` increments to Supabase per tenant per billing period.
7. **AP upstream sync strategy**: Maintain `VENDOR.md` logging the AP git commit each vendor copy was taken from. Review quarterly. Only sync `executor/` — never sync `storage/` (that is our adapter layer).
8. **Direct Postgres vs PostgREST**: For complex formula resolution and cross-table aggregations, `pg` + raw SQL is more flexible. Establish which query patterns warrant direct Postgres vs `supabase-js` per route.
9. **Render service split**: If job processing becomes CPU-heavy (many concurrent AP flows), split into two Render services: one for the Hono HTTP API, one for the pg-boss consumer. Same codebase, different start commands. Cost: +$7/mo.
10. **Grid migration trigger**: Document the specific AG Grid Community limitations that would trigger a Glide Data Grid migration (e.g., >50K visible rows, rendering lag measured at Xms). Do not migrate speculatively.
