# Handoff — Passgrad Worktable Implementation Session

**Date**: May 19, 2026
**Purpose**: Context transfer for continuing implementation with a new agent
**Previous handoff**: `HANDOFF.md` (architecture decisions, schema, rationale)
**Codebase**: `/Volumes/personal/passgrad-worktable-app`

---

## Project Summary

Passgrad Worktable — white-label B2B no-code platform combining Airtable-style database engine with AP-style workflow automation. Primary market: Indonesian campus administration (AMS). Single developer, ~$7/mo target cost at launch.

**Stack**: React 19, Vite 5, TanStack Router (code-based), TanStack Query v5, Tailwind v4 (`@tailwindcss/vite`), shadcn/ui canary (Base UI), React Hook Form + Zod, sonner, lucide-react, TypeScript strict + `exactOptionalPropertyTypes: true`, pnpm monorepo.

**Never use**: axios, Redux, Emotion, CSS modules, inline `style={{}}`, class components, `react-router-dom`, Radix direct imports.

---

## What Is Complete

### Infrastructure & Schema
- Monorepo scaffolded at `/Volumes/personal/passgrad-worktable-app`
- 4 migrations: `0001_auth.sql`–`0004_flow_engine.sql`
- `packages/ap-engine` — lightweight in-process executor with allowlist pieces; `AP_EXECUTION_MODE=UNSANDBOXED`
- `packages/shared-types` — `ViewConfig`, `FieldType`, `FilterGroup`, `SortItem`, `FormFieldConfig`, resume token utils (`signResumeToken`, `verifyResumeToken`)
- `packages/core` — typecheck clean
- `apps/worker/src/jobs/execute-flow.ts` — fully typed, uses `AnyClient` cast for untyped Supabase, `resumeDecision`/`pausedStepName` use conditional spreads for `exactOptionalPropertyTypes`

### Web App — All Pages Refactored (full Tailwind, useQuery/useMutation, no inline styles)
- `AuthPage` — shadcn Card/Input/Button + useMutation
- `OnboardingPage` — shadcn Card + useMutation + preset selector
- `AcceptInvitePage` — multi-mode (choose/signup/signin) with useMutation
- `DatabasesPage` — useQuery + `NewDatabaseDialog` + useMutation; navigate on create
- `DatabasePage` — full rewrite: useQuery/useMutation for all data, Tailwind, view tabs, toolbar toggles (Filter/Sort/Fields), FieldPanel slide-in, all view types wired
- `FlowsPage` — useQuery/useMutation + ShowConfirm for delete; opens FlowBuilder inline
- `TasksPage` — useQuery/useMutation + Badge with statusVariant/statusLabel maps
- `SubmissionsPage` — useQuery + Badge + accordion expand for step outputs
- `SettingsPage` — useMutation for invite + copy-to-clipboard
- `FormPage` — useQuery + useMutation submit + field type renderer

### Web App — All Components Refactored
**Database cells** (all Tailwind, no inline styles):
- `TextCell`, `CheckboxCell`, `NumberCell`, `SingleSelectCell`, `AttachmentCell`, `LinkedRecordCell`

**Database views** (all Tailwind):
- `KanbanView`, `GalleryView`, `CalendarView`
- `FieldPanel` — create/edit field, all types including formula + autonumber, shadcn primitives
- `FilterBar` — conjunction toggle, add/remove/edit conditions
- `SortBar` — add/remove/reorder sort items
- `FormViewBuilder` — field toggle/label/required, share link, success message
- `DatabaseGrid` — AG Grid wrapper, wrapper div converted to Tailwind

**Grid internals**:
- `FieldHeader` — inline rename on double-click, sort indicator, edit menu, all Tailwind

**Automation**:
- `FlowBuilder` — full rewrite: Tailwind + shadcn Button/Input/Label/Textarea, `alert()` → `toast`, `PiecePicker` grid, `StepConfig` form; all 3 CSSProperties objects (`card`, `btn`, `ghostBtn`) removed

**Base components** (cleaned in prior session):
- `components/base/dialog/` — Zustand store + `<DialogPortal>`, imperative `ShowDialog`/`ShowConfirm`/`ShowSimpleDialog`
- `components/base/form/ControlledField*` — `asChild` → `render={}`, `exactOptionalPropertyTypes` fixes
- `components/ui/badge.tsx` — added `success` variant (`bg-green-100 text-green-800`)

**Custom primitives**:
- `src/components/ui/field.tsx`
- `src/components/ui/empty.tsx`

**App shell**:
- `App.tsx` — TanStack Router code-based route tree, QueryClientProvider, Toaster
- `AppShell.tsx` — nav shell, full Tailwind

### Instruction Files (all rewritten to reflect actual stack)
All 7 files in `.github/instructions/`: `architecture`, `architecture-deep`, `style-guide`, `ui-patterns`, `forms-and-data`, `forms-and-data-deep`, `routing-pages`

### Cleanup
- `old-controlled-field.tsx` — deleted
- `react-router-dom` — removed from `package.json`
- `shared-types` — rebuilt dist with `@types/node` added; `signResumeToken` now resolves in worker

### Typecheck Status
**0 non-legacy errors** across all packages. Legacy errors (53, intentionally deferred) are all in:
- `apps/web/src/components/base/grid-table/` — `cell-content.ts`, `draw-fns.ts`, `grid-view.tsx`, `use-grid-columns.ts`, `use-grid.ts`
- `apps/web/src/components/base/table/` — `column-factory.tsx`, `filter-creator.tsx`, `use-view-table.ts`, `view-table.tsx`

These are legacy grid components that will be replaced or deleted when the new grid is fully wired.

---

## Key Architecture Decisions (critical context)

- **shadcn canary Button** — no `asChild`; use `<Link className={buttonVariants({variant})} />` for link-buttons
- **shadcn canary Popover/Select** — `render={}` prop on Trigger; no `onCloseAutoFocus`
- **`calendar.tsx`** — `// @ts-nocheck`; react-day-picker + `exactOptionalPropertyTypes` conflict — do not remove
- **Route nesting**: `DatabasePage` under `appRoute` (id `'app'`) → `useParams({ from: '/app/databases/$id' })`
- **Router**: TanStack Router code-based routes in `App.tsx`; no file-based codegen
- **Tailwind v4**: CSS-native config, `@tailwindcss/vite`, no `tailwind.config.ts`
- **`exactOptionalPropertyTypes: true`**: workspace-wide; optional props need `T | undefined`; use conditional spreads: `...(val !== undefined ? { key: val } : {})`
- **`components/base/dialog/`**: Zustand store + `<DialogPortal>` in root — imperative `ShowDialog`/`ShowConfirm`/`ShowSimpleDialog`
- **Status badge pattern**: two maps per status enum — `xxxStatusBadgeVariant(status)` + `xxxStatusLabel(status)` — no inline ternaries. Badge has `success` variant added.
- **`flowUtils.ts`** — `buildDefinition`/`emptyFlowDefinition` use `as unknown as` casts for `exactOptionalPropertyTypes`
- **`ap-engine/types.ts`** — `PauseMetadata.assigneeEmail?: string | undefined` (explicit `| undefined`)
- **Worker Supabase client** — uses `ReturnType<typeof createClient<any>>` aliased as `AnyClient`; no generated DB types yet
- **pg-boss**: direct (5432) or session pooler only — NOT transaction pooler (6543); `batchSize: 2`
- **RLS**: JWT claim pattern; Auth Hook writes `tenant_id`+`role` into `raw_app_meta_data`
- **Resume token**: HMAC-SHA256 over `{flowRunId}:{exp}`; 7-day TTL; single-use (NULLed on consume)
- **Formula engine**: HyperFormula (MIT); single-record arithmetic v1; LOOKUP/ROLLUP deferred
- **`FlowsPage`** — `FlowBuilder` renders inline (no separate route); delete via `ShowConfirm`

---

## Relevant Files

```
apps/web/src/
  App.tsx                                    — route tree + QueryClientProvider + Toaster
  index.css                                  — @import "tailwindcss" + CSS vars
  components/AppShell.tsx                    — nav shell
  pages/                                     — all 10 pages, fully refactored
  components/database/
    cells/                                   — all cell components refactored
    KanbanView.tsx, GalleryView.tsx,
    CalendarView.tsx                         — full Tailwind
    FieldPanel.tsx                           — create/edit field panel
    FilterBar.tsx, SortBar.tsx               — toolbar bars
    FormViewBuilder.tsx                      — form view config + share link
    grid/DatabaseGrid.tsx                    — AG Grid wrapper
    grid/FieldHeader.tsx                     — inline rename + sort indicator
  components/automation/FlowBuilder.tsx      — full Tailwind rewrite
  components/base/dialog/                    — imperative dialog system
  components/ui/badge.tsx                    — includes 'success' variant

packages/shared-types/src/index.ts          — ViewConfig, FormFieldConfig, FilterGroup, token utils
packages/ap-engine/src/types.ts             — ExecutionContext, FlowDefinition, PauseMetadata
apps/worker/src/jobs/execute-flow.ts        — flow run handler, AnyClient, conditional spreads
.github/instructions/                        — all 7 instruction files, reflect actual stack
```

---

## Immediate Next Steps

1. **Fill `.env` files** — `apps/web/.env`, `apps/api/.env`, `apps/worker/.env`. Required vars:
   - `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`
   - `APPROVAL_TOKEN_SECRET` (for `signResumeToken`)
   - `APPROVAL_TOKEN_TTL_DAYS` (default 7)
   - `RESEND_API_KEY`, `RESEND_FROM_EMAIL`
   - `WEB_URL` (for approval email links)
   - `R2_BUCKET`, `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`

2. **Run migrations** against Supabase to make dev bootable:
   ```bash
   supabase db push
   # or
   supabase migration up
   ```

3. **Audit `apps/web/src/lib/api.ts`** — verify all API stubs have real implementations or are clearly marked as stubs. Key methods: `api.databases.*`, `api.fields.*`, `api.views.*`, `api.records.*`, `api.flows.*`

4. **Build `packages/core` API handlers** (if not yet wired) — Supabase CRUD for databases, fields, views, records with filter/sort application and typed expression indexes

5. **Wire up `apps/api` Hono routes** — check which routes are stubs vs implemented

6. **Delete legacy grid files** once new grid is confirmed working — `base/grid-table/` and `base/table/` (the 53 deferred typecheck errors live here)

---

## Known Issues / Deferred

- **53 typecheck errors** — all in legacy `base/grid-table/` and `base/table/` — intentionally deferred, will be deleted when new grid is fully wired
- **`flowUtils.ts`** — uses `as unknown as` casts; acceptable workaround for `exactOptionalPropertyTypes` with AP engine types
- **No generated Supabase DB types** — worker uses `createClient<any>`; web uses untyped queries. Generate types with `supabase gen types typescript` once schema is stable
- **`calendar.tsx`** — `// @ts-nocheck` due to react-day-picker incompatibility with `exactOptionalPropertyTypes`; do not remove
- **AP pieces `package.json` missing `"license"` field** — MIT by root LICENSE; add scanner overrides in CI
