# Passgrad Worktable — Architecture Review (No-Throwaway Revision)

**Source**: BRIEF.md, ARCHITECTURE-ASSESSMENT.md, HANDOFF.md
**Date**: May 17, 2026
**Posture**: Stay under **$8/mo** hosting budget, but pick the **scalable** library and architecture choices so no code/deps need to be thrown away when traffic, connectors, or features grow. CI / observability / backups are out of scope for this budget.
**Status**: No code yet; fold this into ARCHITECTURE-ASSESSMENT.md before scaffolding.

---

## The throwaway test

The single question this revision asks of every decision:

| Outcome at scale                                                | Verdict     |
| --------------------------------------------------------------- | ----------- |
| Forces a code or dependency rewrite                             | **Throwaway — replace now** |
| Forces a tier / billing upgrade with zero code change           | **Acceptable — note the trigger** |
| Forces only an infra topology change (e.g. split one process into two), no library change | **Acceptable if monorepo is shaped for it from day 1** |

Anything in the "throwaway" column is unacceptable even if it's cheaper today, because the rewrite cost eats the budget many times over.

---

## Headline summary

| Count | Category                                                                     |
| ----- | ---------------------------------------------------------------------------- |
| 1     | Blocker — current grid choice is throwaway                                   |
| 5     | Decisions revised under the no-throwaway lens                                |
| 6     | Decisions that hold up and stay unchanged                                    |
| 4     | Code patterns built right from day 1 at zero $ cost                          |

---

## 1 · Blocker — AG Grid Community is throwaway

AG Grid Community lacks row grouping (Enterprise-only since v32). BRIEF.md §1 lists "group" as a core view capability. The throwaway test:

| Path                                  | Outcome at the moment grouping is needed                                                                    | Verdict     |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ----------- |
| AG Grid Community                     | Rip out, swap library — every cell renderer, every column def, every Grid event wiring rewritten            | **Throwaway** |
| AG Grid Enterprise ($999/dev/yr)      | Zero rewrite, but $999 > $8 × 12 = $96/yr ceiling                                                           | Out of budget |
| **TanStack Table v8 + TanStack Virtual** | Same library forever; grouping is a one-time `getGroupedRowModel` wiring + render | **Right call** |

### Why TanStack Table v8 is the no-throwaway choice

- The headless table primitive behind shadcn/ui's official `data-table` component (so it fits your stack natively); confirmed production use at Cal.com (public repo, `tanstack-table` GitHub topic).
- Headless: you own the renderer, no canvas indirection, cell renderers are plain React components (matches the brief's "person picker, colored selects, attachments, date calendars" requirement).
- Bundle ~15 KB vs AG Grid Community ~500 KB (per TanStack Table homepage spec).
- Same TanStack family (`@tanstack/react-query`) you already use — single dep family.
- Built-in row models for grouping, aggregation, sorting, filtering, pagination, expansion, virtualisation — all opt-in. Grouping ships when a tenant asks: a `getGroupedRowModel` wiring change, not a library swap.
- Server-side row model = plain TanStack Query infinite query — no library coupling.
- Migration path to AG Grid Enterprise stays open: the `<DatabaseGrid columns data onCellEdit>` wrapper in ARCHITECTURE-ASSESSMENT.md §4 is what makes it reversible. Keep it.

The work TanStack asks of you (writing the grouping/virtualisation glue) is **product investment you own forever**, not throwaway labour. The work AG Grid Community asks of you (a forced library swap later) **is** throwaway.

---

## 2 · Decisions revised under the no-throwaway lens

### 2.1 Vendor the Activepieces engine — keep it in one process, but shape the repo as if it were two

Hand-rolling 5–10 connectors fails the throwaway test: at the moment a tenant asks for the 11th, you've rewritten the connector contract once. AP gives you the 280-piece library on day 1 and forces zero contract changes when usage grows.

**Where this revision differs from the prior "scale" version**: do **not** spin up a second Render/Hetzner service for the worker. One Linux process runs Hono + pg-boss + AP engine. Keep it under the $8/mo ceiling. What you cannot skip is the **monorepo shape**:

```
apps/
├── api/                     ← Hono routes, middleware
│   └── src/index.ts         ← serve(app) + startWorker() in the SAME process for now
└── worker/                  ← pg-boss handlers + AP engine entrypoint
    └── src/startWorker.ts   ← exported; imported by apps/api/src/index.ts
packages/
├── vendor/ap-engine/        ← copied AP engine (MIT)
├── shared-types/            ← shared between api + worker
└── core/                    ← Teable core (if vendored) OR HyperFormula wrapper
```

`apps/worker` is its own workspace package from day 1. It imports nothing from `apps/api`. Splitting it into a second deployed service later is a **start-command change**, not a code refactor.

**Risk mitigation for fitting AP engine in 4 GB / 2 vCPU**: disable Code steps in v1 (drops `isolated-vm` entirely — re-enable later as a tier upgrade trigger). Most users never write a Code step; the connector library handles their needs.

### 2.2 Flow builder UI — build directly on React Flow, do **not** copy AP's UI

The prior assessments treated "vendor AP engine" and "copy AP UI" as one decision. They are separable. Under the no-throwaway lens, the UI is where AP costs you the most:

| Option                                              | Throwaway risk                                                                                                  |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Copy AP's `packages/ui/src/features/flows`          | Drags in Mantine + Redux + AP's hook stack. Your app is already shadcn + Tailwind + TanStack Query. The day you decide to ditch AP UI (theme drift, perf, customisation) you rewrite. **Throwaway.** |
| **Build directly on `@xyflow/react` (React Flow)** | Owns ~1–2 KLOC of flow-builder UI in your own shadcn idiom. The engine still consumes AP-compatible flow JSON — your UI just emits it. **Not throwaway.** |

Engine and UI are independent: the engine consumes a JSON flow definition; nothing in the engine cares whether the JSON came from AP's React app or yours. AP's flow schema is public; emit the same shape.

### 2.3 Hosting — Hetzner CX22 (~$5) over Render Starter ($7)

Both fit the $8 ceiling. The difference is runway before the first forced upgrade:

| Host                          | Cost      | vCPU    | RAM    | Disk   | Runway before forced upgrade                                  |
| ----------------------------- | --------- | ------- | ------ | ------ | ------------------------------------------------------------- |
| Render Starter                | $7/mo     | 0.5     | 512 MB | —      | First real workload; AP engine baseline alone uses ~180 MB    |
| **Hetzner CX22** + Coolify    | €4.51 (~$5) | 2 (shared) | 4 GB | 40 GB SSD | Months. 8× the RAM at lower cost; you patch the OS yourself.  |

Both are Docker / Node deployments. Code is identical on either host. Migrating Hetzner → Render or Render → Hetzner is `docker build && docker push` away. **Choosing one over the other is not a throwaway commitment.**

Recommendation: **Hetzner CX22 + Coolify**. Same money, 8× the RAM, single VPS for Hono + pg-boss + AP engine. Coolify gives push-to-deploy UX without managing nginx/systemd by hand. The user has already opted out of CI / backups / observability scope, so the "managed PaaS" premium isn't buying anything.

### 2.4 Supabase Free — keep, but treat it as a tier-upgrade trigger

Free tier limits:

| Limit                         | Value     | Throwaway? | Mitigation                                                                                    |
| ----------------------------- | --------- | ---------- | --------------------------------------------------------------------------------------------- |
| 500 MB DB                     | hard cap  | No — upgrade is billing only, zero code change | Aggressive pg-boss pruning (`archiveCompletedAfterSeconds`, `deleteAfterDays`); don't store full payloads in jobs (store reference to a Supabase row). |
| 50 K MAU                      | hard cap  | No                                          | Upgrade trigger; no auth code changes between tiers.                                          |
| 2 active projects             | hard cap  | No                                          | Use Supabase CLI local Postgres for dev; staging shares the second project.                   |
| Auto-pause after 1 week idle  | behaviour | No                                          | Free uptime ping (UptimeRobot, cron-job.org) on `GET /health` — included in $8 ceiling.       |
| No PITR / no daily backups    | gap       | No                                          | Out of scope per posture; accept until first paying customer.                                 |

Code on free is identical to code on Pro: same `supabase-js`, same migrations, same RLS policies, same Auth Hooks. Tier upgrade is **not** throwaway.

### 2.5 Formula engine — HyperFormula over vendoring Teable ANTLR4

The original doc plans to vendor Teable's `packages/core/formula` (ANTLR4 grammar). The throwaway test:

| Option                                              | Bundle weight | Drift risk                       | Surface area you own forever                                  |
| --------------------------------------------------- | ------------- | -------------------------------- | ------------------------------------------------------------- |
| Vendor Teable ANTLR4 formula                        | ~200 KB       | Cross-references into Teable's `apps/` (AGPL) need audit | A vendored ANTLR4 grammar; debugging is your problem.         |
| **HyperFormula** (MIT, Excel-compatible, ~150 KB)   | ~150 KB       | None — independent project       | A typed function call: `hf.calculate(formula, scope)`.        |

HyperFormula gives Excel-syntax formulas which users already understand. If you ever outgrow it, swap is one module — `packages/core/formula.ts` is your seam. Not throwaway either way, but HyperFormula starts you with less code to maintain.

---

## 3 · Decisions that hold up and stay unchanged

| Decision                                       | Why it stays                                                                                                                 |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| pg-boss for the job queue                      | Free, runs on the Postgres you're already paying $0 for. Arbitrary `sendAfter()` for HITL. Inngest Hobby's 7-day sleep cap fails for AMS approvals. No throwaway when scaling — pg-boss scales with Supabase Pro's pool. |
| Hybrid JSONB schema                            | What Airtable / Teable / Notion run. With §4.2's per-field expression-index handler built day 1, scales to millions of records per tenant without schema rewrite. |
| Hono over Next.js                              | Authed SPA doesn't need SSR. Hono is portable across Node / Bun / Workers; deploy target is replaceable later with no code change. |
| Render Node.js or VPS over CF Workers          | AP engine is Node-only (`isolated-vm`, `node:http`, socket.io). Workers' 50 ms CPU + ephemeral connections preclude long-running flows.   |
| Single Supabase project + `tenant_id` RLS within a vertical | The pattern Supabase optimises for. Combined with §4.1's JWT-claim policy, scales to ~1000 tenants per project. Per-vertical projects later = env-var change, not code change. |
| Cloudflare R2 + Resend                         | R2 zero-egress beats S3 / Supabase Storage for downloads. Resend transactional remains best-in-class. Both have free tiers within budget. |

---

## 4 · Code patterns built right from day 1 (zero $ cost, max future leverage)

These are not optional even at $8/mo. They cost no money and prevent future throwaway:

### 4.1 RLS via JWT claim — never the correlated subquery

```sql
-- WRONG (current doc) — subquery per row check, degrades on large filter scans
CREATE POLICY tenant_isolation ON records
  USING (tenant_id = (SELECT tenant_id FROM profiles WHERE id = auth.uid()));

-- RIGHT — JWT claim, planner inlines as a literal
CREATE POLICY tenant_isolation ON records
  USING (tenant_id = ((auth.jwt() -> 'app_metadata') ->> 'tenant_id')::uuid);
```

Companion: a Supabase Auth Hook (Postgres function) that writes `tenant_id` into `raw_app_meta_data` at provisioning. Lives in the first migration set. Retrofitting later means resigning every existing JWT.

### 4.2 JSONB index strategy — handler-driven, not schema-only

A single GIN on `records.data` is insufficient. Build into the field-create handler from day 1:

| Pattern                                                                                                            | When applied                                                                          |
| ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| `CREATE INDEX … ON records ((data->>'<field_id>')::<type>) WHERE database_id = '<...>'`                          | When a field is marked filterable/sortable at create time.                            |
| `CREATE INDEX … USING GIN (data jsonb_path_ops)`                                                                  | Baseline, one per `records` table (or partition).                                     |
| `pg_partman` partition on `database_id` once any single database crosses ~100 K rows                              | Triggered by a background check — not enabled day 1, but the schema must allow it.    |

The field-create handler emits the matching `CREATE INDEX` migration alongside the `fields` row insert. Drop on field delete. This is the single most leverage-per-line code pattern in the platform.

### 4.3 Monorepo shape — `apps/api` + `apps/worker` separate from day 1

Even though one process runs both, they are two workspace packages with no cross-imports. Split into two services later = start-command change only. See §2.1.

### 4.4 Tenant onboarding lives in Hono, not a Supabase Edge Function

The doc puts onboarding in a Deno Edge Function. Two runtimes for a 30-second operation is throwaway dual-maintenance. Move to a `POST /tenants` route on Hono. Same Node, same `supabase-js`, same types. One runtime, one logger, one error surface.

---

## 5 · Real cost reality at no-throwaway, $8 ceiling

| Item                                       | Monthly |
| ------------------------------------------ | ------- |
| Hetzner CX22 (2 vCPU, 4 GB, 40 GB SSD)     | ~$5     |
| Coolify (self-hosted on the same box)      | $0      |
| Supabase Free (prod) + local CLI (dev)     | $0      |
| Cloudflare Pages (web SPA)                 | $0      |
| Cloudflare R2 (10 GB free)                 | $0      |
| Resend (3 K emails/mo free)                | $0      |
| UptimeRobot or cron-job.org keep-alive ping | $0      |
| Domain                                     | ~$1     |
| **Total**                                  | **~$6/mo** |

Tier-upgrade triggers, all zero-code:

| Trigger                                                | Action               | Marginal cost  |
| ------------------------------------------------------ | -------------------- | -------------- |
| DB > 500 MB or > 50 K MAU                              | Supabase Pro         | +$25/mo        |
| Render → Hetzner if VPS ops becomes annoying           | Render Standard      | +$25/mo (-$5)  |
| AP engine memory pressure (Code steps re-enabled)      | Hetzner CCX13 (dedicated, 8 GB) | +$9/mo |
| Email volume > 3 K/mo                                   | Resend Pro           | +$20/mo        |
| Attachments > 10 GB                                     | R2 ($0.015/GB-mo)    | small linear   |

None require code changes.

---

## 6 · Recommended next-step order (no-throwaway)

1. **Replace AG Grid Community with TanStack Table v8 + TanStack Virtual** in ARCHITECTURE-ASSESSMENT.md §4. Keep the `<DatabaseGrid>` wrapper as the abstraction boundary. Drop the "AG Grid migration path to Glide" sentence; the migration path is now "TanStack to AG Grid Enterprise" if it's ever needed.
2. **Decide on the Hono + flow-builder split**: vendor AP engine for connector breadth, build the flow-builder UI on `@xyflow/react` (do not copy AP's UI). Update HANDOFF.md's monorepo tree to reflect `apps/worker` as a separate workspace package even though deployed in-process.
3. **Write the first migration with JWT-claim RLS + the Auth Hook** that writes `tenant_id` into `raw_app_meta_data`. Cheap, non-negotiable, prevents retrofit pain.
4. **Bake the JSONB index handler into the field-create code path** before any field-type code exists. The handler signature determines the field types' DDL contract.
5. **Move tenant onboarding to a Hono route**. Delete the Edge Function plan.
6. **Pick HyperFormula** for formula evaluation; remove the Teable ANTLR4 vendoring plan from §3.
7. **Provision Hetzner CX22 + Coolify**; deploy a "hello Hono + pg-boss" smoke service before any product code. Verify the AP engine baseline RAM cost in-place before designing around it.
8. **Read AP's flow JSON schema** (open §5 risk in HANDOFF.md). The schema is the contract between your React Flow UI and the vendored engine — confirm shape before writing either.

---

_Review covers ARCHITECTURE-ASSESSMENT.md §§1–6 and the Summary table under the constraint "stay under $8/mo, no throwaway code or dependencies". Source claims cross-checked: AG Grid v32+ Community feature matrix, Hetzner CX22 specs (~$5/mo, 2 vCPU shared, 4 GB), Render Starter specs ($7/mo, 0.5 vCPU, 512 MB), TanStack Table v8 docs, HyperFormula MIT package, pg-boss v12._
