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

- **Dynamic schema**: Hybrid JSONB — metadata tables (`databases`, `fields`, `views`) + `records.data JSONB` with GIN indexes. Same approach as Airtable/Teable/Notion.
- **Multi-tenancy**: Single Supabase project, shared tables, `tenant_id` RLS isolation. Separate Supabase project per vertical deployment is also acceptable.
- **Stack**: Vite + React SPA (Cloudflare Pages) + Hono API (Render Node.js). TypeScript end-to-end. Single monorepo.
- **API hosting**: Hono runs on **Render** (Node.js), NOT Cloudflare Workers. The AP engine requires Node.js 20 — CF Workers incompatible.
- **Automation engine**: Activepieces engine (`packages/server/engine`) vendored as `packages/vendor/ap-engine/` and run inside the **same Render process** as the Hono API. No separate service.
- **Job queue**: **pg-boss** (MIT, Postgres-native) running against the existing Supabase Postgres. No Redis. No separate queue service. Zero extra cost.
- **Human-in-the-loop**: Handled natively by pg-boss — `sendAfter()` supports arbitrary future timestamps. No 24h cap. Engine writes `waiting_for_human` state + `resume_token` to Supabase; new job enqueued when human approves/rejects.
- **Automation UI**: Activepieces React flow builder (`packages/ui/src/features/flows/`) copied into `apps/web/src/features/automation/`. Only the `api/` layer is replaced to point at our Hono backend.
- **Grid UI**: **AG Grid Community** (MIT). Abstracted behind `<DatabaseGrid>` wrapper — migration path to Glide Data Grid defined if performance limits hit.
- **Field types / formula / filter-sort**: Vendored from Teable `packages/core` (MIT). ANTLR4 formula engine confirmed pure JS — CF Workers safe (though moot since we're on Node.js now).
- **Auth / RLS**: Supabase Auth + RLS. Hono uses anon key + forwards user JWT to `supabase-js` per-request client. RLS fires in Postgres automatically. Service role key is never used in routes.
- **File storage**: Cloudflare R2 (10GB free, no egress). Accessed from Render via AWS S3-compatible API (`@aws-sdk/client-s3`).
- **Email**: Resend (3,000 emails/mo free) for system transactional email only.
- **Total hosting cost**: ~$7/mo (Render $7 + everything else free).

---

## What Was Investigated and Rejected/Updated

| Original decision | Finding | Updated decision |
|---|---|---|
| AP engine in CF Worker | Engine uses `node:http`, `isolated-vm` (native addon), `socket.io` IPC — 100% Node.js only, zero CF Workers compatibility | AP engine on Render Node.js |
| CF Queues for job dispatch | CF Queues `delaySeconds` caps at 24h — cannot handle multi-day approval waits | pg-boss on Supabase Postgres |
| CF Workers Paid ($5/mo) | Not needed once API moves to Render — no CF Workers usage beyond Pages | Dropped, saving $5/mo |
| DO Alarms for human-in-the-loop | pg-boss `sendAfter()` handles arbitrary future timestamps natively | DO Alarms not needed |
| Glide Data Grid | Row grouping not built-in; maintenance risk (no stable release since Feb 2024); canvas renderers harder for rich field types | AG Grid Community first, Glide as migration path |

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

The 5 risks from the previous handoff have been resolved:

1. ~~AP Engine + CF Workers Node.js Compatibility~~ → **Resolved**: Engine is CF Workers incompatible. Decision: run on Render Node.js.
2. ~~Human-in-the-Loop Queue Pattern~~ → **Resolved**: pg-boss `sendAfter()` with arbitrary future timestamps. State machine documented in ARCHITECTURE-ASSESSMENT.md §3.
3. ~~Hono + Supabase RLS Auth Threading~~ → **Resolved**: Anon key + JWT forwarding pattern documented in ARCHITECTURE-ASSESSMENT.md §2.
4. ~~AP Flow JSON Schema Review~~ → **Still open**: Need to read `packages/shared` types to confirm JSONB storage works without transformation. Low risk — standard JSON store.
5. ~~Teable Core Packages Dependency Audit~~ → **Resolved**: `antlr4ts` is pure JS. All runtime deps are isomorphic. `axios` works natively on Node.js (no fetch adapter needed). Safe to vendor.

One new open item:
- **AP pieces `package.json` missing `"license"` field** → MIT by root LICENSE, but will trip automated license scanners. Add scanner overrides in CI config.

---

## Recommended Next Steps

1. **Read AP flow JSON schema** — open `activepieces/packages/shared` and document the flow definition type. Confirm it stores/retrieves as Supabase JSONB without transformation. Last remaining risk before implementation.
2. **Scaffold the monorepo** — `pnpm` workspace, `apps/web`, `apps/api`, `packages/vendor/ap-engine`, `packages/core`, `packages/shared-types`.
3. **Establish Hono + Supabase middleware** — implement the JWT + per-request client pattern from §2 as the first route middleware.
4. **First Supabase migration** — `tenants`, `profiles`, `databases`, `fields`, `views`, `records`, `field_sequences`.
5. **Wire pg-boss** — `startWorker()` in `apps/api/src/index.ts`, stub `execute-flow` job handler.

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
│   │       │   └── workflow/         ← task UI, approval forms
│   │       ├── features/
│   │       │   └── automation/       ← COPIED from AP packages/ui/src/features/flows/
│   │       │         ├── canvas/     ← kept as-is
│   │       │         ├── api/        ← replaced → Hono endpoints
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
│   │         ├── executor/           ← kept as-is
│   │         └── storage/            ← replaced → Supabase implementation
│   ├── core/                         ← Vendored from Teable packages/core (MIT)
│   └── shared-types/
└── supabase/
    ├── functions/
    └── migrations/
```

---

## Related Files in This Repo

- `BRIEF.md` — product brief (what we're building, for whom, capabilities)
- `ARCHITECTURE-ASSESSMENT.md` — full decisions with rationale, schema, open questions
- `HANDOFF.md` — this file
