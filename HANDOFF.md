# Handoff — Passgrad Worktable Architecture Session

**Date**: May 17, 2026
**Purpose**: Context transfer for continuing with a new agent
**Existing docs**: `BRIEF.md` (what we're building), `ARCHITECTURE-ASSESSMENT.md` (all decisions + rationale)

---

## What This Session Covered

Architecture review and revision. All major decisions from the previous session were challenged against facts, and several were updated. No code has been written yet — still a pure architecture/planning session.

---

## Key Decisions Made (summary)

Full rationale is in `ARCHITECTURE-ASSESSMENT.md`. Short version:

- **Dynamic schema**: Hybrid JSONB — metadata tables (`databases`, `fields`, `views`) + `records.data JSONB` with GIN indexes + typed expression indexes per filterable field.
- **Multi-tenancy**: Single Supabase project, shared tables, `tenant_id` RLS isolation. Separate Supabase project per vertical deployment is also acceptable.
- **RLS policy**: JWT claim pattern (`auth.jwt() -> 'app_metadata' ->> 'tenant_id'`) — not a correlated subquery. Auth Hook writes `tenant_id` + `role` into `raw_app_meta_data` at provisioning. Must be in the first migration.
- **Roles**: `tenant_members(tenant_id, user_id, role)` — `owner` / `admin` / `member`. Role in JWT claim drives all RLS policies.
- **Row visibility**: `records.created_by` set server-side on insert. Members see only their own records (RLS). Admins see all.
- **Sharing model**: Lark Base-inspired. Three levels: (1) base-level role access via `tenant_members`, (2) view read-only public link, (3) public form submission (no auth). Implemented via `view_shares` table + unauthenticated routes.
- **Select options**: IDs stored in `records.data`, labels in `fields.config.options[].id`. Rename = one `fields` row update, zero record backfill.
- **Linked records**: Separate `record_links` table — FK integrity + clean JOIN for cross-DB formula references.
- **Attachments**: Separate `attachments` table — lifecycle, MIME, size. Presigned R2 URL route. No inline JSONB.
- **Unified flow model** (workflows dropped): `workflows`/`tasks`/`task_actions` tables dropped entirely. Approval = `@passgrad/piece-approval` node in a flow. Form submission = `@passgrad/piece-database-form` trigger. Run state tracked in `flow_runs` table (new). "My Tasks" = `flow_runs WHERE status='waiting_for_human' AND assigned_to=me`. "My Submissions" = `flow_runs WHERE submitted_by=me`. No separate approval state machine.
- **View config shape**: Locked TypeScript type `ViewConfig` in `shared-types` — `FilterGroup`, `SortItem[]`, `hiddenFields`, `groupBy`, `kanbanStackBy`, etc.
- **Presets**: TypeScript files in `apps/api/src/presets/` — versioned with code, seeded via `POST /tenants`. Indonesian names in preset data.
- **i18n**: English + Indonesian from day one. `react-i18next`. All UI copy uses `t('key')` from the first component. Indonesian is the default locale (`profiles.locale DEFAULT 'id'`). Preset data stores Indonesian names directly.
- **JSONB index strategy**: Field-create handler emits typed expression index per filterable/sortable field; field-delete drops it. Must be built before any field type.
- **Monorepo structure**: `apps/worker` is a separate workspace package from `apps/api`. Both run in one process today; split = start-command change only.
- **Tenant onboarding**: `POST /tenants` on Hono — not a Supabase Edge Function.
- **pg-boss payload discipline**: References only in job payloads. Configure pruning from day one.
- **API hosting**: Hono on **Render** (Node.js). AP engine requires Node.js 20 — CF Workers incompatible.
- **Automation engine**: AP engine vendored as `packages/vendor/ap-engine/`, runs in same Render process as Hono. `AP_EXECUTION_MODE=UNSANDBOXED` for v1 — disables `isolated-vm`, Code steps hidden from UI. Re-enable on Developer tier.
- **Job queue**: pg-boss on Supabase Postgres. No Redis. Zero extra cost.
- **Human-in-the-loop**: pg-boss `sendAfter()` with arbitrary future timestamps. Engine writes `waiting_for_human` + `resume_token`; new job on approve/reject.
- **Automation UI**: AP React flow builder copied into `apps/web/src/features/automation/`. Only `api/` layer replaced.
- **Grid**: AG Grid Community (MIT) behind `<DatabaseGrid>` wrapper. DIY grouping layer. No AG Grid types outside `components/database/grid/`.
- **Formula engine**: HyperFormula (MIT, ~150 KB). `{field_name}` → cell address adapter in `packages/core/formula.ts`.
- **Field types / filter-sort**: Built in-house — small surface area (~600 lines total). Teable core not vendored.
- **File storage**: Cloudflare R2. Accessed via `@aws-sdk/client-s3` from Render.
- **Email**: Resend (3,000 emails/mo free).
- **Total hosting cost**: ~$7/mo.

---

## What Was Investigated and Rejected/Updated

| Original decision | Finding | Updated decision |
|---|---|---|
| AP engine in CF Worker | Engine uses `node:http`, `isolated-vm` (native addon), `socket.io` IPC — 100% Node.js only, zero CF Workers compatibility | AP engine on Render Node.js |
| CF Queues for job dispatch | CF Queues `delaySeconds` caps at 24h — cannot handle multi-day approval waits | pg-boss on Supabase Postgres |
| CF Workers Paid ($5/mo) | Not needed once API moves to Render — no CF Workers usage beyond Pages | Dropped, saving $5/mo |
| DO Alarms for human-in-the-loop | pg-boss `sendAfter()` handles arbitrary future timestamps natively | DO Alarms not needed |
| Glide Data Grid | Row grouping not built-in; maintenance risk (no stable release since Feb 2024); canvas renderers harder for rich field types | AG Grid Community; Glide as escalation path only |
| RLS correlated subquery | `SELECT tenant_id FROM profiles WHERE id = auth.uid()` fires per-row during scans | JWT claim inlined as literal by planner |
| Single GIN index on `records.data` | Insufficient for per-field filter/sort at scale | Typed expression index per filterable field from field-create handler |
| `apps/worker` inside `apps/api` | Couples HTTP server and queue consumer — hard to split later | Separate workspace package; split = start-command change |
| Tenant onboarding via Supabase Edge Function | Second runtime for a 30-second operation | `POST /tenants` on Hono |
| Teable packages/core vendored | Small surface area; ANTLR4 overhead; HyperFormula is simpler for formula | HyperFormula for formulas; field types + filter/sort built in-house |
| `isolated-vm` enabled by default | Lazy-loaded, ~2–3 MB resident, V8 isolate spin-up per `{{}}` expression; native addon compile risk on Render | `AP_EXECUTION_MODE=UNSANDBOXED` for v1; Code step hidden from UI; re-enable on Developer tier |
| Single `profiles.role` enum | Can't model "admin of tenant A, member of tenant B" | `tenant_members(tenant_id, user_id, role)` join table |
| Separate `workflows`/`tasks`/`task_actions` tables | AP already has form trigger + human approval pause + router — everything needed to model approval as a flow. Separate state machine tables add schema surface for no benefit. | Dropped. Unified flow model: approval is a node, form is a trigger, `flow_runs` tracks state. |
| Labels in `records.data` for select options | Rename requires backfilling every record | Option IDs in `records.data`, labels in `fields.config` |
| Linked records inline in `records.data` | Cross-DB formula queries become JSONB scans; no FK integrity | Separate `record_links` table |
| Attachments inline in `records.data` | No lifecycle, no MIME/size, R2 orphans accumulate | Separate `attachments` table |
| No i18n planned | Primary market is Indonesian; English needed for extensibility | `react-i18next`, English + Indonesian from day one, `profiles.locale DEFAULT 'id'` |

---

## Copy-and-Adapt Strategy (important)

The core philosophy: **copy as much as possible from MIT-licensed open source, only replace the infrastructure/API adapter layer**.

| Source | Copy | Replace |
|---|---|---|
| `activepieces/packages/server/engine` | Execution logic, branching, variable interpolation | `storage/` interface → Supabase |
| `activepieces/packages/ui/src/features/flows` | Canvas, step config forms, piece selector, run history | `api/` calls → Hono endpoints |
| `activepieces/packages/piece-*` (npm) | All 280+ connectors, used as-is | Nothing |
| `teable/packages/core` (MIT) | Field types, formula engine, filter/sort | Nothing |

Rule: only touch files in designated adapter seams. Leave execution logic and UI untouched to keep upstream diffs minimal.

---

## Options Evaluated and Rejected

Documented here so the next agent doesn't re-litigate them:

| Option | Rejected because |
|---|---|
| NocoDB | RLS, SSO, white-labeling all behind paid Enterprise tier |
| Teable (fork full app) | `apps/` directory is AGPL. Only `packages/` are MIT — those we vendor. |
| n8n | "Sustainable Use License" explicitly prohibits use as part of a commercial service |
| Windmill | CE license explicitly prohibits selling as managed service; Rust+Svelte wrong stack |
| Full Activepieces Docker stack on Render | ~$43–71/mo; runs infra we don't need |
| AP engine in CF Worker | Incompatible — uses `node:http`, `isolated-vm`, `socket.io` IPC, Node.js process primitives |
| CF Queues | Not viable for multi-day approval waits (24h delay cap); also moot since API moved off Workers |
| BullMQ + Redis | Adds $10/mo Render Redis for no benefit over pg-boss given Supabase Postgres already present |
| Durable Object Alarms | Unnecessary — pg-boss `sendAfter()` handles arbitrary-future timestamps natively |
| Glide Data Grid (primary) | No built-in row grouping; maintenance risk; canvas renderers hard for rich field types |
| Cloudflare D1 / Turso | No RLS, no Auth, no Realtime — would require rebuilding tenant isolation in app layer |
| DDL-per-tenant-table | Schema sprawl at scale, Supabase practical limits |
| EAV schema | Multi-join performance degrades beyond a few thousand records |
| Schema-per-tenant (Postgres) | PostgREST doesn't handle multi-schema well, migration complexity |

---

## Open Risks (resolved from previous session)

All 5 risks from the original session are now resolved:

1. ~~AP Engine + CF Workers Node.js Compatibility~~ → **Resolved**: Engine is CF Workers incompatible. Decision: run on Render Node.js.
2. ~~Human-in-the-Loop Queue Pattern~~ → **Resolved**: pg-boss `sendAfter()` with arbitrary future timestamps. State machine documented in ARCHITECTURE-ASSESSMENT.md §3.
3. ~~Hono + Supabase RLS Auth Threading~~ → **Resolved**: Anon key + JWT claim pattern documented in ARCHITECTURE-ASSESSMENT.md §2.
4. ~~AP Flow JSON Schema Review~~ → **Resolved**: `FlowVersion.trigger` is a self-contained recursive JSON tree. Stores as `flow_versions.definition JSONB`, passed directly to engine. Only `AppConnection` credentials need runtime lookup via the `storage/` adapter. Tables: `flows`, `flow_versions`, `app_connections`, `trigger_sources` — schema in §3.
5. ~~Teable Core Packages Dependency Audit~~ → **Resolved**: `antlr4ts` is pure JS. All runtime deps are isomorphic. Safe to vendor.

Remaining open items (non-blocking):
- **AP pieces `package.json` missing `"license"` field** → MIT by root LICENSE, but will trip automated license scanners. Add scanner overrides in CI config.
- **Formula engine choice** (Teable ANTLR4 vs HyperFormula) — decide when implementing formula fields. Not needed for scaffolding.

---

## Recommended Next Steps

**All architecture decisions confirmed. Ready to scaffold.**

1. **Scaffold the monorepo** — `pnpm` workspace, `apps/web`, `apps/api`, `apps/worker`, `packages/vendor/ap-engine`, `packages/core`, `packages/shared-types`.
2. **Write the first migration** — in order: Auth Hook + RLS JWT-claim policies first, then `tenants`, `profiles`, `tenant_members`, `databases`, `fields`, `views`, `records`, `record_links`, `attachments`, `field_sequences`, `flows`, `flow_versions`, `flow_runs`, `app_connections`, `trigger_sources`, `view_shares`, `database_collaborators`. The Auth Hook must exist before any tenant is created.
3. **Establish Hono + Supabase middleware** — JWT + per-request client pattern from §2.
4. **Wire pg-boss** — `startWorker()` in `apps/worker/src/index.ts`, imported by `apps/api/src/index.ts`. Stub `execute-flow` job handler. Configure `archiveCompletedAfterSeconds` and `deleteAfterDays`.
5. **Build field-create handler with index DDL** — before implementing any field type, establish the pattern that emits a typed expression index on field create and drops it on field delete.

---

## Monorepo Structure (agreed)

```
passgrad-worktable-app/
├── apps/
│   ├── web/                          ← Vite + React SPA (Cloudflare Pages)
│   │   └── src/
│   │       ├── components/
│   │       │   ├── database/
│   │       │   │   ├── grid/         ← <DatabaseGrid> wrapper (AG Grid, swappable)
│   │       │   │   └── cells/        ← cell renderers per field type (React components)
│   │       │   └── flows/                ← task inbox, approval forms, "My Tasks", "My Submissions"
│   │       ├── features/
│   │       │   └── automation/       ← COPIED from AP packages/ui/src/features/flows/
│   │       │         ├── canvas/     ← kept as-is
│   │       │         ├── api/        ← replaced → Hono endpoints
│   │       │         └── ...         ← step forms, piece selector, run history (kept as-is)
│   │       └── pages/
│   ├── api/                          ← Hono HTTP server (Render Web Service)
│   │   └── src/
│   │       ├── routes/
│   │       │   ├── databases.ts
│   │       │   ├── records.ts
│   │       │   ├── fields.ts
│   │       │   ├── flows.ts          ← flow CRUD + flow_runs routes (resume, My Tasks, My Submissions)
│   │       │   ├── forms.ts          ← POST /forms/:token/submit (no auth middleware)
│   │       │   └── tenants.ts        ← tenant onboarding (POST /tenants)
│   │       ├── middleware/
│   │       │   └── supabase.ts       ← JWT verification + per-request Supabase client
│   │       └── index.ts              ← serve(app) + import startWorker from apps/worker
│   └── worker/                       ← pg-boss consumer + AP engine (separate workspace)
│       └── src/
│           └── index.ts              ← startWorker() exported; no imports from apps/api
├── packages/
│   ├── vendor/
│   │   └── ap-engine/                ← COPIED from AP packages/server/engine/src (MIT)
│   │         ├── executor/           ← kept as-is
│   │         └── storage/            ← replaced → Supabase implementation
│   ├── core/                         ← in-house field types, filter/sort, formula adapter (HyperFormula)
│   ├── pieces/
│   │   ├── database-form/            ← @passgrad/piece-database-form (form trigger)
│   │   └── approval/                 ← @passgrad/piece-approval (human approval node)
│   └── shared-types/
└── supabase/
    └── migrations/                   ← incl. Auth Hook + RLS JWT-claim policies
```

---

## Related Files in This Repo

- `BRIEF.md` — product brief (what we're building, for whom, capabilities)
- `ARCHITECTURE-ASSESSMENT.md` — full decisions with rationale, schema, open questions
- `HANDOFF.md` — this file
