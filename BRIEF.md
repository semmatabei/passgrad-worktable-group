# Project Brief — Passgrad Worktable

## What We Are Building

A no-code platform that combines a **flexible database engine** (like Airtable) with a **workflow automation engine** (like n8n), or Lark Base + Anycross, packaged as a white-label B2B SaaS product. Clients use it to manage structured data and automate processes without writing code.

Primary initial market: Indonesian campus administration (AMS — Academic Management System). Extensible to other verticals (healthcare, kiosk/POS, government) using the same codebase with different preset templates.

---

## Core Capabilities

### 1. Database Engine

- Tenants create their own tables ("databases") with custom fields: text, number, single/multi-select, date, attachment, person, formula, autonumber
- Multiple view types per table: grid (spreadsheet), form, kanban, calendar, gallery
- Filter, sort, group, and hide fields per view — saved per view, per user
- Formula fields computed server-side (referencing other fields or linked tables)
- CSV import/export

### 2. Workflow Engine

- Tenants define multi-step approval workflows (e.g., enrollment, leave request, purchase order)
- Steps have assignees, deadlines, conditions, and branching logic
- Human-in-the-loop: tasks wait for approval/rejection before advancing
- Automated actions on step transitions: send email, notify via WhatsApp/Slack, update a field, create a linked record

### 3. Automation Builder

- Visual flow builder (drag-and-drop canvas) embedded in the product shell
- Trigger types: record created, field changed, date reached, manual, webhook
- 280+ connectors (Gmail, Slack, WhatsApp, HTTP webhook, Google Sheets, etc.)
- Non-technical users configure integrations themselves — no code required
- Monetization lever: gated by automation run count per billing tier

---

## Who Uses It

| Role                                   | What they do                                                    |
| -------------------------------------- | --------------------------------------------------------------- |
| **Platform admin** (us)                | Deploys the platform, creates tenant accounts, seeds presets    |
| **Tenant admin** (e.g., university IT) | Creates databases, defines workflows, manages users             |
| **End users** (e.g., students, staff)  | Submits forms, completes workflow tasks, views records they own |

---

## Tech Stack (summary)

| Layer                  | Technology                                                               |
| ---------------------- | ------------------------------------------------------------------------ |
| Frontend               | Vite + React SPA, hosted on Cloudflare Pages                             |
| API                    | Hono on Cloudflare Workers                                               |
| Database               | Supabase (Postgres + Auth), hybrid JSONB schema                          |
| Automation engine      | Activepieces engine + pieces (MIT, vendored/npm) — runs inside CF Worker |
| Automation UI          | Activepieces React flow builder (MIT, copied + API-adapted)              |
| File storage           | Cloudflare R2                                                            |
| Email                  | Resend                                                                   |
| **Total hosting cost** | **~$5/mo**                                                               |

The stack is fully TypeScript. Single maintainer. No separate services or Docker containers — everything runs inside Cloudflare Workers.

---

## What Is Copied vs Built

This project maximizes reuse of MIT-licensed open source code and only builds the glue layer:

| Source                                               | What is reused                                          | What is replaced                                       |
| ---------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------ |
| **Activepieces** (`packages/server/engine`)          | Flow execution logic, variable interpolation, branching | Storage interface → Supabase; auth → our Supabase auth |
| **Activepieces** (`@activepieces/piece-*`)           | 280+ connectors as npm packages                         | Nothing — used as-is                                   |
| **Activepieces** (`packages/web/src/features/flows`) | React flow canvas, step config forms, run history UI    | API calls → our Hono endpoints; app shell → our shell  |
| **Teable** (`packages/core`, MIT)                    | Field types, formula engine, filter/sort logic          | Nothing — vendored as-is                               |
| **AG Grid Community** (MIT)                          | Spreadsheet grid with virtual scroll, inline editing    | Nothing — used as-is                                   |

---

## Deployment Model

One codebase. Deployed as:

- A single multi-tenant app (`app.passgrad.com/[tenant-slug]`), **or**
- Separate branded deployments per vertical (`ams.passgrad.com`, `clinic.passgrad.com`) — same code, different env vars and preset data

Each new tenant is onboarded in ~30 seconds via an automated Edge Function that seeds their preset databases, field schemas, and workflow templates.

---

## Architecture Reference

Full architecture decisions, schema, and rationale: see `ARCHITECTURE-ASSESSMENT.md`.
