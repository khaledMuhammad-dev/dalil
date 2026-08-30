# Dalil — Frontend Start Guide (`dalil-web`)

> **Audience:** the AI agent (or developer) building the Dalil frontend from zero.
> **Status of the backend:** phases 0–6 behaviorally complete, read surface + CORS + OpenAPI live,
> demo-walk proven end-to-end. 1943 tests green. The API is ready to build against **today**.
> **Goal:** ship a demo-quality frontend covering the full lifecycle (lead → quotation → proposal →
> project → invoice → payment) against the live API, then harden to launch quality.
>
> Read together with: `04-AI-INSTRUCTIONS.md` (binding rules), `02-FS-launch.md` (behavior),
> `01-PRD-launch.md` (product), `03-GAP-ANALYSIS.md` (why decisions were made).

---

## 1. Non-negotiable context

- `dalil-web` is a **separate repository** (D-011) living at `dalil/dalil-web/` beside `dalil/dalil-api/`.
  Separate git history, separate deploy. Never import backend code; the API contract is the only coupling.
- **Stack (LOCKED — never substitute):** Next.js (App Router) · TypeScript strict · TailwindCSS +
  Shadcn UI · TanStack Query (ALL server state) · Zustand (client-only state: UI, locale, session) ·
  React Hook Form + Zod · i18next (Arabic + English) · Storybook + `@storybook/test-runner`.
- **Arabic-first**: default locale is Arabic (RTL). Every component is built RTL-first with logical CSS
  properties ONLY (`ps-`/`pe-`/`ms-`/`me-`/`start-`/`end-`; never `pl-`/`pr-`/`ml-`/`mr-`/`left-`/`right-`).
- Three distinct UIs in one Next.js app (route groups), one design system:
  1. **Agency app** `(agency)` — tenant users, RBAC-gated. The main product.
  2. **Client portal** `(portal)` — the agency's clients. Separate auth (Client-JWT), separate nav, white-label surface.
  3. **Platform console** `(platform)` — Dalil's own operator. Small; last priority.
- **The design is decided — implement the Dalil Experience System (DXS).** Tokens (colors,
  type, spacing, radius, elevation, motion, grid), fonts, and ten screen designs live in the
  Claude Design project `https://claude.ai/design/p/a299c726-a791-4824-b931-1776a2cd9204`.
  The implementation contract is `08-UI-DESIGN-BRIEF.md` §13 — read it before Slice 0. In
  short: warm cream `#F2EDDF` + navy ink `#232E52` + terracotta `#DE7356` (never blue/indigo/
  gray SaaS defaults), semantic CSS custom properties only, borders-not-shadows, 8pt spacing,
  dark mode via `data-theme="dark"`. Never hardcode a hex/px that exists as a token.

## 2. Run the backend locally (before any frontend code)

```bash
cd ../dalil-api
pnpm install
# .env needs DATABASE_URL (Postgres) + JWT keys; see .env.example
npx prisma migrate dev && npx prisma db seed
pnpm build && node dist/src/src/main.js     # ⚠ built server ONLY — `tsx src/main.ts` fails to boot
# API at http://localhost:8080/api/v1 · docs at http://localhost:8080/api-docs
```

Seeded logins:

| Surface | How |
|---|---|
| Agency app | header `X-Tenant-Slug: demo` + `POST /auth/login` `{ "email": "admin@demo.local", "password": "Demo@123456" }` |
| Platform console | `POST /platform/auth/login` `{ "email": "owner@dalil.platform", "password": "Platform@123456" }` |
| Client portal | no seeded contact with password — convert a lead (YES) in the agency app, then read the onboarding token from the `auth_tokens` table (email is stubbed in dev) |

## 3. Scaffold

```bash
pnpm create next-app@latest dalil-web --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd dalil-web
pnpm dlx shadcn@latest init
pnpm add @tanstack/react-query zustand react-hook-form zod @hookform/resolvers i18next react-i18next axios
pnpm add -D storybook @storybook/nextjs @storybook/test-runner openapi-typescript prettier
```

Directory shape (mirror the backend's predictability mandate):

```
src/
  app/
    (agency)/            # tenant app: /login, /leads, /contacts, /quotations, /projects, /invoices, ...
    (portal)/portal/     # client portal: /portal/login, /portal/projects, /portal/invoices, ...
    (platform)/platform/ # operator console
    layout.tsx           # html dir= + lang= from locale; providers
  components/ui/         # shadcn primitives (generated)
  components/shared/     # design-system components — EVERY one has stories (D-007.8)
  features/<module>/     # module-scoped components + hooks + api slices (leads, quotations, ...)
  lib/api/               # generated types + client + envelope/error handling + auth
  lib/i18n/              # i18next init; locales/ar/*.json, locales/en/*.json
  lib/permissions/       # usePermission(), <RequirePermission>
  stores/                # zustand: session, ui
  styles/                # DXS: styles.css entry + tokens/*.css (copied from the design project)
public/fonts/            # SaudiaSans woff2/woff (the "Dalil" face) — self-hosted, 4 weights
```

**Install the design system as part of the scaffold:** copy `styles.css`, `tokens/` (8 files)
and `assets/fonts/` (8 SaudiaSans files) out of the Claude Design project into the paths above,
import the entry stylesheet in the root layout, and map the Tailwind theme onto the CSS custom
properties (`colors`, `spacing`, `borderRadius`, `fontFamily` all reference `var(--…)`). Full
rules: `08-UI-DESIGN-BRIEF.md` §13.

## 4. API client layer (build this first — everything sits on it)

1. **Generate types from the live spec:**
   `pnpm openapi-typescript http://localhost:8080/api-docs-json -o src/lib/api/schema.d.ts`
   (check the generated file in; regenerate on backend change; add a `pnpm api:types` script).
2. **Envelopes.** Success is `{ data }`; paginated lists are `{ data: [...], meta: { nextCursor, hasMore } }`
   (cursor pagination on: leads, contacts, projects, quotations, users, permission-groups — pass
   `?cursor=&limit=`). Errors are FLAT: `{ code, message, requestId, field?, errors?, details? }`.
   Unwrap `data` centrally; surface a typed `ApiError`.
3. **Error → i18n.** `message` is unlocalized English. Map `code` (40 codes, `error-codes.json`) to
   i18n keys: `errors.OVERPAYMENT`, `errors.VALIDATION_ERROR`, … with a generic fallback that shows
   `requestId` for support. Validation failures are **HTTP 422** with `errors: [{ field, message }]` —
   map them onto RHF field errors via a shared helper.
4. **Headers on every request:** `X-Tenant-Slug` (dev: `demo`; prod: derived from subdomain) and
   `Authorization: Bearer <accessToken>`.
5. **Token lifecycle (agency):** access 15 min / refresh 7 days with **rotation** — on refresh the old
   refresh token is dead; always store the newly returned pair. Single-flight the refresh (one refresh
   promise shared by concurrent 401s), queue and replay failed requests, logout on refresh failure.
   Store the refresh token in memory + `localStorage` (v1; move to httpOnly cookie via a BFF route
   post-demo — noted in GAP-ANALYSIS).
6. **Token lifecycle (portal):** Client-JWT, 15 min, **no refresh** — on expiry show the "session
   expired, re-enter email" screen. Keep portal and agency sessions in separate storage keys.
7. **TanStack Query:** one `queryKey` convention `[module, entity, params]`; `staleTime` 30s default;
   invalidate on mutation success by module key. **Poll** `GET /notifications` every 30–60s (no
   WebSocket in the demo) and re-poll active lists (project board 15s) — realtime lands post-demo.

## 5. Auth + permission gating

- `GET /me` after login → `{ id, email, name, tenantId, isActive, groups[], permissions: string[] }`.
  The JWT has **no profile claims** — `/me` is the single source for identity + effective permissions.
- Build `usePermission(key)` + `<RequirePermission key="quotations.create">` and gate every action
  button, nav item, and route segment with **permission keys, never group names** (mirror of the
  backend's rule). `403 FORBIDDEN` responses carry `requiredPermission` — show the standard 403 screen.
- The permission catalog (33 keys, 16 categories) is at `GET /permission-groups/permissions` — use it
  to render the group editor; never hardcode the key list.
- Route guards: `(agency)` requires tenant session; `(portal)` requires client session; `(platform)`
  requires platform session. `401 UNAUTHENTICATED` → redirect to the surface's login.

## 6. i18n / RTL / theming (from day one, not retrofitted)

- `i18next` with namespaces per module. **No literal user-facing string in JSX** — every string via `t()`.
- `<html lang="ar" dir="rtl">` default; toggle to `en/ltr`. Persist per user (locale in Zustand +
  `localStorage`; server pref endpoint is post-demo).
- Fonts (per DXS `tokens/fonts.css`): English = the custom **"Dalil"** face (SaudiaSans files,
  self-hosted, weights 400/500/600/700, `font-display: swap`); Arabic = **IBM Plex Sans Arabic**
  (Google Fonts for demo; self-host at deploy). Switch by locale via `--font-english` /
  `--font-arabic`. ⚠ Licence for Saudia Sans is still open item F (fallback: Inter) — keep the
  font behind the token so a swap is one line.
- Numbers/currency: `Intl.NumberFormat` with `SAR`; **money is a string from the API** (DECIMAL) — never
  `parseFloat` for arithmetic, only for display formatting. Dates: ISO 8601 UTC from API → render local
  with `Intl.DateTimeFormat` (both `ar` and `en`); calendar UI must handle a Sunday–Thursday work week.
- Dark mode: toggle `data-theme="dark"` on `<html>` — the DXS semantic tokens re-point and
  component code changes nothing (wire Tailwind's `dark:` variant to that attribute). Every
  shared component ships light+dark, RTL+LTR stories.

## 7. Storybook mandate (D-007.8 — binding)

Every **shared/design-system** component ships stories covering: default · variants · loading · empty ·
empty-on-filter · error · RTL · dark mode — with `play()` interaction tests run by
`@storybook/test-runner` in CI. Feature screens are composed from these proven components. The stories
ARE the proof the states exist; a data view without loading/empty/empty-on-filter/error handling is a
review failure.

## 8. Build order (screens mapped to live endpoints)

Build in vertical slices along the demo path; each slice is shippable and demo-able.
`[key]` = permission key gating the UI. Full screen inventory: PRD §8 (60 screens).

**Slice 0 — Foundation (no screens):** DXS tokens + fonts installed and mapped into Tailwind
(§3 / `08-UI-DESIGN-BRIEF.md` §13) + API client + auth + refresh + `/me` + permission gate + i18n +
layout shells (sidebar 240/68px, header 68px — from `tokens/grid.css`) + error screens (404/403/500)
+ notification poller + `PageState` (loading/empty/error) component. **Definition of done: login →
authenticated shell → logout works in AR and EN, rendered in the DXS cream/navy/terracotta theme,
light and dark.**

**Slice 1 — CRM (the demo opens here):**
- Leads kanban board — `GET /sales-flow/stages` + `GET /leads?stageId&q&cursor` `[crm.view]`; drag
  between stages → `PATCH /leads/:id/stage` (optimistic update, rollback on error).
- New lead `POST /leads` `[crm.create]` · Lead detail + convert dialog (`YES/NO`) →
  `PATCH /leads/:id/convert` `[crm.edit]` · reopen `PATCH /leads/:id`.
- Sales-flow configurator — `PATCH /sales-flow/stages` `[crm.edit]`; handle `STAGE_HAS_ACTIVE_LEADS`
  by re-submitting with `force: true` after an explicit confirm dialog.
- Contacts list/detail/history — `GET /contacts`, `GET /contacts/:id`, `GET /contacts/:id/history` `[contacts.view]`.

**Slice 2 — Catalog + Blueprints (setup screens):**
- Services CRUD `[services.view|manage]`; archive flow on `SERVICE_IN_USE`. Packages CRUD
  `[packages.manage]` (min 2 services; surface `PACKAGE_CONTAINS_ARCHIVED_SERVICE`).
- Blueprint library + editor `[blueprints.view|manage]` — phases (name, service, order,
  `dependsOnPhaseIds`), DAG editing with client-side cycle check (server confirms with
  `CIRCULAR_PHASE_DEPENDENCY`), board assignment per phase.
- Task-board template editor — boards, columns (order matters: **highest order = Done column**), task templates.

**Slice 3 — Quotation & Proposal (the revenue path):**
- Quotations list/detail `[quotations.view]` — status chips for the 7-state FSM.
- Quotation builder `[quotations.create]` — contact → services XOR package → `POST /quotations/blueprint-matches`
  (exact-match; render the `NO_BLUEPRINT_FOUND` fallback with a "create blueprint" link) → payment model →
  **milestone grouping step** (group phases; amounts must sum to total ±0.01; surface
  `MILESTONE_NOT_DEPENDENCY_CLOSED` / `PHASE_NOT_IN_MILESTONE` / `PHASE_IN_MULTIPLE_MILESTONES` /
  `MILESTONE_AMOUNTS_DONT_SUM` inline on the grouping UI, not as toasts).
- Generate proposal → send `[quotations.send]` → track versions (`GET /quotations/:id/proposals`);
  amendment review; reopen on rejection.

**Slice 4 — Projects & Task board:**
- Projects list `GET /projects?status` `[projects.view]` · project detail (phase list, per-phase progress,
  computed project progress) · Start screen `POST /projects/:id/start` `[projects.start]` (handle
  `PROJECT_NOT_STARTABLE`, `PLAN_LIMIT_EXCEEDED`, `BLUEPRINT_DELETED`).
- Kanban board `GET /projects/:id/board?phaseId` `[tasks.view]` — shared-board phase filter + phase badge
  per card; move `PATCH /tasks/:id` `[tasks.manage]`; assign `PATCH /tasks/:id/assignee` `[tasks.assign]`
  (handle `USER_INACTIVE`). Phase hold/resume `[projects.start]`.
- Approval queue + review `[approvals.review]` — `GET /approvals/queue`, `POST /approvals`
  (APPROVE / REQUEST_CHANGES + comment; handle `PHASE_ALREADY_REVIEWED` = someone else got there first).

**Slice 5 — Invoicing & Change requests:**
- Invoices list `GET /invoices?projectId` + detail `GET /invoices/:id` `[invoices.view]` — derived status
  (UNPAID/PARTIALLY_PAID/PAID/OVERDUE), embedded payment ledger.
- Record payment `POST /invoices/:id/payments` `[invoices.record_payment]` (handle `OVERPAYMENT` with the
  `maximum` detail). PDF button: dev returns a stub URL — show "PDF available at deploy phase" until real.
- Change requests: create (staff) `[change_requests.manage]`, approve/reject `[change_requests.approve]`
  (approve auto-generates the invoice — reflect it immediately), list/detail.

**Slice 6 — Team & RBAC:**
- Users list/detail/invite/deactivate `[users.*]` — invite flow ends at a dev-visible token note (email stubbed).
- Group assignment + per-user overrides `[users.manage]`; Permission Groups CRUD `[permission-groups.manage]`
  — surface all four admin-protection 422s verbatim as friendly, specific messages.

**Slice 7 — Client portal (second persona of the demo):**
- Portal auth: onboarding verify → set password; magic-link login; password login. 15-min session, no refresh.
- Portal home `GET /portal/projects`; project detail (client-mapped phase statuses + milestone summary);
  proposal review → **amend / accept / reject** (accept → "project awaiting start" holding state);
  phase approve / feedback; invoices `GET /portal/invoices`; new-project request form.

**Slice 8 — Platform console (thin):** platform login, tenants list/provision/suspend/reactivate,
plans + entitlements editor.

**Deliberately NOT built yet (backend stubs — design the UI seams, hide the buttons):** file
upload/download (S3), real PDFs, real email, WebSocket realtime (poll instead), online payment/checkout,
analytics dashboards (no endpoints), notification preferences.

## 9. The demo walk (acceptance script — must pass end-to-end in the UI)

1. Login as `admin@demo.local` (AR default; switch to EN and back).
2. Configure: create 2 services → a blueprint with 3 phases (one dependency) → a board with templates.
3. CRM: create lead → move across stages → convert **YES**.
4. (Dev) read onboarding token from DB → open portal → set password as the client.
5. Quotation: builder → exact-match blueprint → milestone grouping (2 milestones) → generate + send proposal.
6. Portal: client requests amendment → agency revises + resends → client **accepts**.
7. Start project → board work → move tasks to Done → phase IN_REVIEW → admin approves → client approves phase.
8. Invoice auto-generates for the completed milestone → record a partial payment → PARTIALLY_PAID → full → PAID.
9. Change request from portal → approve → second invoice appears.
10. Team: invite a user into Sales group → login as them → verify UI is permission-trimmed.

## 10. Quality gates (every slice, before moving on)

- `tsc --noEmit` 0 · eslint 0 · Storybook test-runner green · stories for every new shared component
  (all 8 required states) · both locales render (no missing keys, no layout breakage in RTL) ·
  loading/empty/empty-on-filter/error handled on every data view · all mapped error codes have i18n
  strings · demo-walk steps touched by the slice re-verified against the live API ·
  **token discipline: no hardcoded hex/px where a DXS token exists, semantic aliases only**
  (`08-UI-DESIGN-BRIEF.md` §13 — same defect class as a physical `pl-`/`pr-`).
