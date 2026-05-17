# Passgrad Worktable — Architecture Review

**Source**: BRIEF.md, ARCHITECTURE-ASSESSMENT.md, HANDOFF.md
**Date**: May 17, 2026
**Status**: No code yet

| Count | Category                                  |
| ----- | ----------------------------------------- |
| 1     | Blocker (factual error vs spec)           |
| 4     | High-risk decisions                       |
| 5     | Better alternatives worth weighing        |
| 6     | Decisions that hold up under scrutiny     |

---

## Headline Finding

The architecture's AG Grid Community claim is factually wrong: row grouping is **Enterprise-only** ($999/dev/yr). BRIEF.md lists "group" as a core per-view capability. Either swap the grid, rewrite the spec, or budget for the Enterprise license. This is the only spec-vs-architecture contradiction; the rest of the review is risk and optimization.

---

## 1 · Blocker — fix before implementation

### AG Grid Community does not include row grouping

ARCHITECTURE-ASSESSMENT §4 ("Grid: AG Grid Community — with defined migration path") states: _"Full row grouping + aggregation (built-in, Community edition)"_. This is incorrect.

In AG Grid v32+, row grouping, aggregation, pivoting, and tree data are gated behind `ag-grid-enterprise`. Community ships filter, sort, virtualization, basic editing, column resize/reorder — but not grouping.

BRIEF.md §1 lists "Filter, sort, **group**, and hide fields per view" as a core capability. The spec and the grid choice are inconsistent.

**Three viable options:**

| Option                                       | Effort               | Cost          | Tradeoff                                                                                                                                                                       |
| -------------------------------------------- | -------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| AG Grid Enterprise license                   | Zero code change     | $999/dev/yr   | Kills the $7/mo cost story; perpetual license but yearly support fees                                                                                                          |
| **TanStack Table v8 + custom virtualizer**   | 2–3 weeks initial    | $0 (MIT)      | Headless — you build the renderer. Grouping is a few hundred LOC against TanStack's group-row model. Much lighter bundle (~30 KB vs ~500 KB)                                   |
| Glide Data Grid (already noted as fallback)  | 1–2 weeks initial    | $0 (MIT)      | Canvas-rendered, fast. Grouping not built in either — same custom work as TanStack. Cell renderers harder than React components                                                |
| Tabulator (MIT)                              | 1–2 weeks initial    | $0 (MIT)      | Has tree/grouping in MIT build. Less React-native than TanStack. Lower momentum                                                                                                |

**Recommendation**: Go TanStack Table v8 + TanStack Virtual. The `<DatabaseGrid>` wrapper in the doc already enforces isolation; you'll get grouping, sort, filter, virtualization, and column reordering for a fraction of the bundle weight, and the dependency surface stays inside the React Query / TanStack family you already use.

---

## 2 · High-risk decisions worth challenging

### 2.1 RLS uses correlated subquery → per-row profile lookup

The doc's policy reads:

```sql
tenant_id = (SELECT tenant_id FROM profiles WHERE id = auth.uid())
```

This subquery runs against `profiles` on every row visibility check. Under filter scans across thousands of records the planner caches it, but the pattern is the #1 RLS performance footgun on Supabase forums.

**Fix — put `tenant_id` in the JWT:**

```sql
tenant_id = ((auth.jwt() -> 'app_metadata') ->> 'tenant_id')::uuid
```

Set `tenant_id` in `app_metadata` at signup via a Supabase hook (or the onboarding edge function). No row lookup, planner inlines the literal, GIN index scans stay cheap.

### 2.2 AP engine vendoring scope is optimistic

The doc frames it as _"copy `executor/`, swap `storage/`"_ with a `VENDOR.md` quarterly sync. Real coupling in `packages/server/engine`:

| Coupling point                  | Reality                                                                              |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| `@activepieces/shared` types    | Flow, Trigger, Action, PieceMetadata — pulled into your code permanently             |
| Error class hierarchy           | `ActivepiecesError` thrown across boundaries; engine consumers must match            |
| Logger / sandbox interfaces     | Engine assumes pino + their sandboxing contract                                      |
| Piece registry + version resolver | Engine resolves pieces by hash; you maintain the resolver in your storage layer    |
| `isolated-vm` native addon      | Per-platform binaries; Render Linux container fine, but local Mac dev needs prebuilt + `xcode-select` |

Honest estimate: **2–4 weeks** initial integration for a solo dev, ongoing maintenance ~1 day per AP release (~26 releases/year). Quarterly sync ≈ 6 versions of drift per merge — painful.

> **Cheaper path worth weighing**: For the AMS MVP, you need ~5–10 connectors (email, WhatsApp, Slack, HTTP webhook, internal record actions, Sheets, maybe Notion). Hand-rolled mini-engine: state machine over typed step definitions, ~1.5K LOC, zero `isolated-vm`, zero vendor drift. You lose the "280 connectors" marketing point — but the brief says non-technical users configure their own integrations, and the actual integration long-tail on a campus is small. Keep AP as a Phase-2 trigger when a paying customer needs a specific connector you don't have.

### 2.3 Render Starter 512 MB is tight for HTTP + worker + isolated-vm

Baseline residents inside one Node process:

| Component                                                  | Approx RSS |
| ---------------------------------------------------------- | ---------: |
| Node 20 + V8                                               |      80 MB |
| Hono + supabase-js + pg + Drizzle                          |      50 MB |
| pg-boss polling worker                                     |      30 MB |
| AP engine baseline                                         |     ~80 MB |
| isolated-vm V8 isolate (per concurrent code step)          |  ~50 MB ea |
| Headroom for JSONB record buffers under list endpoints     |          ? |

Two concurrent Code steps + a 1k-record list endpoint and you're at the 512 MB ceiling with no slack. Render OOM-kills the process, pg-boss recovers but users see 502s.

**Mitigations, ranked:**

1. **Disable Code steps in v1** — drop `isolated-vm` entirely. Most workflow builders never use Code. Re-enable when you can afford Standard ($25/mo).
2. **Switch host to Hetzner CX22** (€4.51/mo, 2 vCPU, 4 GB RAM, 40 GB disk) — same money, 8× the RAM. Coolify or Dokploy on top gives you a Render-like push-to-deploy UX. Tradeoff: you maintain Linux + backups.
3. **Budget for the upgrade trigger** and ship on Render Starter knowing the first real workload moves you to Standard ($25). The true v1 floor cost is $7, not the steady-state cost.

### 2.4 Supabase free tier 500 MB will not last a real customer

Quick capacity math for one mid-sized AMS tenant (5k students):

| Source                                          |    Rows | Avg JSONB size |    Total |
| ----------------------------------------------- | ------: | -------------: | -------: |
| records (students × 9 preset DBs)               |    ~80k |         1.5 KB |   120 MB |
| `records.data` GIN index                        |       — |              — |   ~40 MB |
| run_logs (one flow per enrollment × 5k)         |    ~25k |           3 KB |    75 MB |
| pg-boss tables (jobs + archive)                 |  varies |              — | 30–80 MB |
| Postgres overhead, WAL, autovacuum slack        |       — |              — |  ~100 MB |

You hit 500 MB before you finish onboarding the second customer. The $7/mo headline cost is honest for a demo/dev environment, not for first paying tenant.

> **Real production floor cost** = Render Starter $7 + Supabase Pro $25 + Resend Pro $20 (3k emails dies on first batch enrollment) = **~$52/mo**. That's still very competitive — but the doc framing buries it.

---

## 3 · Better alternatives worth weighing

| Area                            | Current decision                                  | Alternative                                                      | When it wins                                                                                                                                       |
| ------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Workflow engine                 | Vendor AP engine + 280 pieces                     | Hand-rolled state machine + 5–10 connectors                      | AMS-only v1 with known integration list — saves ~3 weeks initial + all upstream drift                                                              |
| Flow builder UI                 | Copy AP `packages/ui/src/features/flows` + swap `api/` | Build directly on React Flow / `@xyflow/react` with shadcn step config | If you've already chosen TanStack/shadcn for the rest of the app, AP's UI is Mantine + Redux + their hook system — fewer dependencies to drag in   |
| Formula engine                  | Vendor Teable `packages/core` (ANTLR4 grammar)    | HyperFormula (MIT, Excel-compatible, ~150 KB)                    | Smaller dep, better-known surface, formula syntax users already understand                                                                         |
| Tenant onboarding               | Supabase Edge Function (Deno)                     | `POST /tenants` route on the existing Hono API (Node)            | Removes second runtime; one less environment to test/deploy/debug                                                                                  |
| Host                            | Render Starter $7                                 | Hetzner CX22 €4.51 + Coolify                                     | Same money, 4× CPU, 8× RAM, same git-push UX. Trade: you patch the OS.                                                                             |
| JSONB sort/filter on typed fields | Single GIN index, accept full-scan ORDER BY     | Expression index per field as user adds it (migration runs in onboarding flow) | Sort by number/date stays fast at >10k records. Adds index management — explicit in field-create handler.                                          |

---

## 4 · Decisions that hold up under scrutiny

These were reviewed against alternatives and remain the right call for this scope. Don't re-litigate.

### pg-boss over Inngest / BullMQ

Inngest free durable-sleep caps at 7 days; AMS approvals routinely run multi-week. Inngest Pro is $75/mo. BullMQ needs Redis ($10/mo). pg-boss runs free on the Postgres you already pay for and supports arbitrary future `sendAfter()`. **Right call.**

### Hybrid JSONB schema

DDL-per-table sprawls; EAV joins die at scale; pure JSONB skips metadata. The hybrid matches what Airtable / Teable / Notion run internally. **Right call** — just plan for the per-field expression-index strategy noted above.

### Hono over Next.js

Fully-authed SPA with heavy interactive surfaces (grid, flow canvas) doesn't need SSR. Hono gives you runtime portability if you ever move workers around. **Right call.**

### Render Node.js over CF Workers

AP engine uses `isolated-vm`, `node:http`, socket.io IPC — 100% incompatible with Workers. Even ignoring AP, Workers' 50 ms CPU + no long-lived connections kills the worker-in-same-process design. **Right call.**

### Single Supabase project + tenant_id RLS

Schema-per-tenant is a PostgREST anti-pattern; project-per-tenant breaks Supabase's project quota. **Right call** — fix the JWT-claim subquery noted in §2.1 and it's production-grade.

### R2 for attachments, Resend for email

R2 zero-egress beats S3 / Supabase Storage on cost for any user-facing downloads. Resend transactional is best-in-class for the price. **Right call.**

---

## 5 · Missing from the doc

Sections the next agent will need before writing code:

| Topic                       | Why it matters                                                                                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CI/CD pipeline              | GitHub Actions, lint + typecheck + e2e, Render auto-deploy hook, CF Pages preview deploys. Currently undocumented.                                            |
| Local dev setup             | `supabase` CLI for local Postgres, pg-boss against same, `isolated-vm` prebuilt for arm64 macOS. Footguns deserve a section.                                  |
| Backup + DR                 | Supabase Pro = nightly PITR. Free tier = none. State which tier is acceptable for which environment.                                                          |
| License audit pipeline      | Doc flags AP pieces missing license field; needs FOSSA / license-checker config committed before first vendor copy.                                           |
| Observability               | Render gives basic logs; consider Logtail / Better Stack free tier for log aggregation. Sentry free tier for FE+BE errors.                                    |
| Rate limiting & abuse       | Single tenant can DOS the engine by enqueueing 10k jobs. pg-boss per-queue concurrency caps + tenant-level throttle in Hono middleware needed.                |
| Migration story             | Schema migrations across N tenants with `tenant_id` is fine, but JSONB schema changes (renaming a field across all `records.data`) need a documented backfill pattern. |

---

## 6 · Recommended next-step order

Re-ordered relative to HANDOFF.md's list to address blockers first:

1. **Resolve the grid blocker** — decide TanStack / Glide / Tabulator / Enterprise license. Document the choice in ARCHITECTURE-ASSESSMENT.md §4 and update the cost table if it changes.
2. **Decide vendor-AP vs hand-rolled engine for v1.** This single choice changes the monorepo layout, the queue payload shape, and the scope of `packages/vendor/ap-engine/`. Don't scaffold the monorepo until this is decided.
3. **Read the AP flow JSON schema** (open §5 risk) — even if you don't vendor the engine, the schema is a decent reference for your own workflow JSON.
4. **Rewrite the RLS policy** to JWT-claim form and add the auth hook that writes `tenant_id` into `app_metadata`.
5. **Scaffold monorepo** (pnpm), Hono middleware, first migration, pg-boss `startWorker()` — proceed as HANDOFF.md describes once (1) and (2) are settled.

---

_Review covers ARCHITECTURE-ASSESSMENT.md §§1–6 and the Summary table. Source claims cross-checked: AG Grid v32+ Community feature matrix (ag-grid.com pricing page), Inngest Hobby tier limits (inngest.com/pricing), Render Starter specs ($7 / 0.5 CPU / 512 MB), pg-boss v12 documented API._
