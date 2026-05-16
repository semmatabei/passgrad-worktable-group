# Architecture Assessment — Passgrad Worktable

**Date**: May 16, 2026  
**Status**: Draft — Pending Review  
**Scope**: No-code database engine + workflow engine for `passgrad-worktable-app`

---

## Summary of Decisions

| Area                    | Decision                                                              |
| ----------------------- | --------------------------------------------------------------------- |
| Dynamic Schema          | Hybrid: metadata tables + JSONB record store                          |
| Multi-tenancy           | Single project, shared tables, `tenant_id` row-level isolation        |
| Workflow engine         | Cloudflare Workers + Queues + Cron Triggers (fully CF-native)         |
| Automation layer        | Activepieces (Apache 2.0) — core product feature + monetization lever |
| Tech stack              | Vite (React SPA) + Hono on Cloudflare Workers (monorepo)              |
| Hosting                 | Cloudflare Pages (free) + Workers Paid $5/mo + Supabase free tier     |
| **Total cost at start** | **~$5/mo**                                                            |

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

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE databases ENABLE ROW LEVEL SECURITY;
ALTER TABLE records ENABLE ROW LEVEL SECURITY;

-- Policy: users can only access their own tenant's data
CREATE POLICY tenant_isolation ON records
  USING (tenant_id = (SELECT tenant_id FROM profiles WHERE id = auth.uid()));
```

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

## 3. Workflow Engine

### Current WMS gaps (from code audit)

- Step execution is client-side (`current_step + 1` in React)
- Condition branches defined in schema but never evaluated
- No automated triggers, no notifications, no external calls
- No retry logic, no timeouts, no escalation

### Decision: Cloudflare Workers + Queues + Cron Triggers (fully CF-native, no n8n needed)

Since the API already runs on Cloudflare Workers, the workflow engine lives in the **same codebase and deployment** — no separate service, no n8n, no Supabase Edge Functions required.

**Architecture**:

```
User action (approve/submit step)
  → API Worker (Hono route)
    → Cloudflare Queue: 'workflow-jobs'
      → Queue Consumer Worker: workflow-executor
          ├── Load task + workflow_snapshot from Supabase DB
          ├── Evaluate current step (condition branch, approval, action)
          ├── Assign next owner
          ├── Update DB
          ├── Send notification email (Resend HTTP call)
          └── Fire outbound webhooks to external systems

Time-based triggers (reminders, escalations, deadlines)
  → Cloudflare Cron Trigger (runs a Worker on schedule)
    → Same workflow-executor logic: scan overdue tasks → push to Queue
```

**Why Cloudflare-native over Supabase Edge Functions + pg_cron**:

|             | Supabase Edge Functions + pg_cron             | Cloudflare Workers + Queues + Cron            |
| ----------- | --------------------------------------------- | --------------------------------------------- |
| Language    | Deno (TypeScript, slightly different runtime) | Node-compatible Workers (same TS as your API) |
| Scheduling  | pg_cron → pg_net HTTP call (indirect)         | Cron Triggers (native, built into Workers)    |
| Queue/retry | Manual retry logic                            | Queues handle retry + dead-letter natively    |
| Codebase    | Split across Supabase functions + CF Workers  | **Single codebase, single deploy**            |
| Cold starts | Deno cold start                               | Near-zero (V8 isolates)                       |
| Cost        | Free (but separate platform to manage)        | Included in $5/mo Workers Paid                |

**Why not n8n**: n8n's Sustainable Use License prohibits using it as part of a commercial service. Additionally, hosting it requires a persistent server. Not needed here — outbound HTTP calls from Workers handle external integrations directly.

**Why not Temporal**: Requires a persistent server. Overkill for a state machine that fits in a Queue Consumer. Revisit if workflow complexity grows significantly.

### Cloudflare primitives used

| Primitive         | Role                                          | Cost (Workers Paid)              |
| ----------------- | --------------------------------------------- | -------------------------------- |
| **Workers**       | Workflow execution logic                      | Included                         |
| **Queues**        | Async job queue, automatic retry, dead-letter | 1M ops/mo included, then $0.40/M |
| **Cron Triggers** | Scheduled scans (overdue tasks, reminders)    | Included                         |
| **R2** (optional) | Task attachment storage                       | 10GB free                        |

### Queue Consumer: `workflow-executor`

```ts
// Triggered by: queue message OR cron trigger
async function executeWorkflowStep(taskId: string) {
  // 1. Load task from Supabase
  // 2. Evaluate current step type
  // 3. Resolve condition branch if applicable
  // 4. Advance step: update current_step, current_owner
  // 5. Send email notification via Resend (HTTP fetch)
  // 6. Fire outbound webhooks if configured on step
  // 7. Write to workflow_logs
}

// Dead-letter handling: if step fails after N retries → alert admin
```

### Cron Trigger for time-based events

```toml
# wrangler.toml
[triggers]
crons = ["0 * * * *"]  # every hour
# Worker checks: overdue tasks, pending reminders, escalation deadlines
```

### External integrations — Activepieces as core product feature

**Decision: Activepieces is a first-class product feature, not a future add-on.**

Automation extensibility is a selling point and a pricing lever. Clients configure their own integrations (WhatsApp, Slack, ERP, SIS, payment gateways) via a visual builder embedded in the platform. You monetize by gating:

- Number of active automation flows per tenant
- Number of workflow runs/month (e.g., 1K runs free, 10K on paid tier)
- Access to premium connectors

This mirrors how Zapier, Make, and n8n Cloud price — clients already understand the model.

**Why Activepieces over n8n**: n8n's Sustainable Use License prohibits embedding in a commercial product. Activepieces is Apache 2.0 — copy, modify, embed, monetize freely.

**Architecture with Activepieces**:

```
Two layers of automation:

[Internal] Cloudflare Workers + Queues
  → Workflow state machine (step routing, approvals, conditions)
  → Not user-configurable — powers the WMS engine

[External] Activepieces (self-hosted, Fly.io)
  → User-configurable integrations ("when student enrolled → notify WhatsApp")
  → Triggered by: Workers emitting a webhook event to Activepieces
  → Connects to: 200+ pre-built connectors (Slack, WhatsApp, Google Sheets, etc.)
  → Exposed in product UI as "Automation" section
  → Metered per tenant for billing
```

**Hosting Activepieces**: Cloudflare Containers — runs a Docker container alongside your Workers, managed from the same `wrangler.toml`, same dashboard, same billing. Included in the $5/mo Workers Paid plan (25 GiB-hours/month free). Activepieces publishes an official Docker image so this works out of the box.

```ts
// Worker routes /automation/* requests to the Activepieces container
import { Container, getContainer } from "@cloudflare/containers";

export class ActivepiecesContainer extends Container {
  defaultPort = 3000;
  sleepAfter = "10m"; // sleeps when idle — saves compute on free allotment
}

export default {
  async fetch(request, env) {
    if (new URL(request.url).pathname.startsWith("/automation")) {
      const ap = getContainer(env.ACTIVEPIECES, "singleton");
      return ap.fetch(request);
    }
    return app.fetch(request, env); // Hono API handles everything else
  },
};
```

The `sleepAfter: "10m"` means the container stops when idle — perfect for early customers with infrequent automation runs. Wakes in seconds when a trigger fires.

**Integration point**: When the workflow executor completes a step, it fires a webhook to Activepieces with the event payload. Activepieces routes to the client's configured flows. Decoupled — Activepieces going down doesn't break the core workflow engine.

### Notification layer

**Resend** (free: 3,000 emails/mo) for system notifications (step assigned, task completed). One `fetch()` call from the Worker. Activepieces handles client-configured notification channels (WhatsApp, Slack, etc.).

### Workflow engine cost

- Cloudflare Workers + Queues + Cron: **included in $5/mo Paid plan**
- Activepieces on Fly.io: **~$3/mo**
- Resend: **$0** up to 3K emails/mo
- **Total: ~$3/mo additional beyond the base $5/mo**

---

## 4. Tech Stack

### Single-maintainer constraint

One developer maintains the full platform. This favors:

- One language (TypeScript end-to-end)
- One codebase (monorepo)
- Minimal infrastructure to operate
- Easy local development

### Decision: Vite (React SPA) + Hono on Cloudflare Workers

**Why not Next.js**: Next.js on Cloudflare Workers requires `@opennextjs/cloudflare` adapter, which adds complexity and has quirks. Overkill for a single maintainer who is already comfortable with Vite.

**Why not full server-rendered**: The app is highly interactive (spreadsheet-like grid, drag-and-drop, real-time). A SPA with Supabase Realtime handles this better than SSR. SEO is irrelevant for an authenticated B2B app.

**Why Hono**:

- Runs identically on Cloudflare Workers, Node.js, Bun, and Deno — same codebase, different runtime adapter
- TypeScript-first, minimal overhead
- Easy to move from Workers → dedicated server: swap `import { serve } from '@hono/node-server'`
- Great for building REST APIs that are called by the React SPA

### Monorepo structure

```
passgrad-worktable-app/
├── apps/
│   ├── web/              ← Vite + React SPA
│   │   └── src/
│   │       ├── components/
│   │       │   ├── database/   ← grid, views, field editors
│   │       │   └── workflow/   ← workflow builder, task UI
│   │       └── pages/
│   └── api/              ← Hono app (runs on Workers or Node)
│       └── src/
│           ├── routes/
│           │   ├── databases.ts
│           │   ├── records.ts
│           │   ├── fields.ts
│           │   └── workflows.ts
│           └── index.ts  ← Hono app entry
├── packages/
│   ├── core/             ← Vendored from Teable MIT packages
│   │   ├── field-types/
│   │   ├── formula/
│   │   └── filter/
│   └── shared-types/     ← Shared TypeScript types
└── supabase/
    ├── functions/        ← Edge Functions (workflow-executor, etc.)
    └── migrations/
```

### Serverless now, server later

The key decision: **use Hono**. Hono is runtime-agnostic.

```ts
// Today: Cloudflare Workers
export default app; // workers entry

// Later: Node.js server (same app code)
import { serve } from "@hono/node-server";
serve(app); // swap one import, done
```

Moving from Workers to a dedicated server is a ~1 hour change when the time comes, not a rewrite.

### Database access from Workers

Workers cannot use Supabase's Postgres connection directly (TCP not allowed). Two options:

| Option                            | How                    | Notes                                                                                     |
| --------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------- |
| **Supabase REST API (PostgREST)** | `supabase-js` client   | ✅ Works natively from Workers. No connection pooling needed.                             |
| **Cloudflare Hyperdrive**         | TCP proxy for Postgres | ✅ $0 with Workers Paid plan. Direct SQL from Workers. More flexible for complex queries. |

**Recommendation**: Start with Supabase REST (`supabase-js`). Migrate to Hyperdrive for complex queries (formula evaluation, cross-table aggregations) when needed.

---

## 5. Hosting Map

### Decision: Cloudflare Pages + Workers Paid ($5/mo) + Supabase Free

| Service                     | What it hosts                                           | Cost       | Free limit                                           |
| --------------------------- | ------------------------------------------------------- | ---------- | ---------------------------------------------------- |
| **Cloudflare Pages**        | React SPA (static assets)                               | **$0**     | Unlimited static asset requests                      |
| **Cloudflare Workers Paid** | Hono API + Workflow engine + Cron + Queues + Containers | **$5/mo**  | 10M requests, 1M Queue ops, 25 GiB-hrs Containers/mo |
| **Supabase Free**           | Postgres DB + Auth + Storage                            | **$0**     | 500MB DB, 1GB storage                                |
| **Resend**                  | System email notifications                              | **$0**     | 3,000 emails/mo                                      |
| **Total**                   |                                                         | **~$5/mo** |                                                      |

### When to upgrade

| Trigger                              | Action                                              | New cost |
| ------------------------------------ | --------------------------------------------------- | -------- |
| DB > 500MB or > 50K MAU              | Supabase Pro                                        | +$25/mo  |
| API > 10M requests/mo                | Already included in Workers Paid; just CPU billing  | +small   |
| Activepieces > 25 GiB-hrs/mo         | Containers billed at $0.0000025/GiB-sec (very low)  | +small   |
| Workflow complexity needs durability | Replace Queue Consumer with Temporal on a Container | +small   |

### Cloudflare Workers limitations to know

- **No persistent TCP connections**: Can't run a long-running Node.js process. Stateless per request.
- **CPU time limit**: 30 seconds per invocation on Paid plan (more than enough for API calls).
- **No filesystem access**: Use R2 for file storage, not local disk.
- **Cold starts**: Near zero on Workers (V8 isolates, not containers). Not an issue.

### File storage

Use **Supabase Storage** (1GB free) initially. If attachment storage grows, switch to **Cloudflare R2** (10GB free, no egress fees).

---

## 6. Use-Case Deployment Pattern

For different verticals (AMS, kiosk POS, healthcare):

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
              Same Workers codebase, different SUPABASE_URL env var per deployment
              Cloudflare allows multiple Pages deployments from same repo
              Each Supabase project stays within free tier (500MB) for early customers
```

**Which to choose**: Start with Option A (simpler). Move to Option B per use-case only when you need clean product separation, different pricing tiers, or compliance requirements. The codebase never changes — just the environment variables.

Tenant onboarding for a new university (AMS):

1. Create tenant record in `tenants` table
2. Run preset migration: seed 9 databases (Mahasiswa, Pegawai, etc.) with predefined field schemas
3. Seed default workflow templates (bulk enrollment, course registration)
4. Create admin user

This is a ~30-second operation via an Edge Function, not a manual process.

---

## Open Questions (for later)

1. **Formula engine**: Use Teable's MIT `packages/core/formula` vendored in, or build a simpler expression evaluator? Teable's uses ANTLR grammar — powerful but heavy. Assess when implementing formula field.
2. **Real-time collaboration**: Supabase Realtime (free) for live record updates. Sufficient for initial use cases.
3. **Offline support**: Not in scope for MVP. Note for healthcare use case (clinics with spotty internet).
4. **Compliance**: Healthcare deployments may require separate Supabase project for data isolation. Decide per client contract, not upfront.
5. **Bulk import performance**: CSV import of 10K student records via JSONB insert — benchmark before launch. May need batching + background processing via Queue.
6. **Activepieces billing metering**: Decide how to count and expose automation run usage per tenant. Options: proxy all Activepieces webhook triggers through the API (count there), or use Activepieces' own execution logs via API. Assess when building the billing layer.
7. **Activepieces UI embedding**: Decide whether to deep-link clients to Activepieces' own UI, or embed it in an iframe within the product shell. Iframe is simpler; deep-link requires less maintenance.
