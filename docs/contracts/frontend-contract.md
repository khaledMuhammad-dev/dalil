# Dalil API — Frontend Contract

> **See also `../../dalil-api/fable-5/05-FRONTEND-START.md`** — the full frontend build guide (scaffold, auth/RBAC
> wiring, slice-by-slice screen order, demo acceptance walk). This file remains the quick API
> reference; the fable-5 pack is the launch briefing.
>
> Companion doc for `dalil-web`. The **live OpenAPI explorer is at `/api-docs`** (spec JSON at
> `/api-docs-json` — generate your typed client from it). This doc is the human-readable overview;
> the spec is authoritative for exact shapes. Everything here is stable and safe to build against.
>
> ⚠ **Run the built server, not `tsx`.** Start the API with `pnpm build && node dist/src/src/main.js`
> (or `pnpm start`); it listens on `PORT` (default 8080). `npx tsx src/main.ts` fails to boot due to an
> esbuild circular-import quirk between the permission guard and the RBAC service; the compiled
> CommonJS build (what prod uses) is unaffected. See `../../dalil-api/carry-farword.md` → "Known gotchas".

## Base

- **Base URL:** `http://localhost:8080/api/v1` (all routes are under the `api/v1` global prefix). The
  port is the `PORT` env var, default **8080** when unset.
- **OpenAPI:** `GET /api-docs` (Scalar reference UI) · `GET /api-docs-json` (spec, 92 paths). Not under the
  `api/v1` prefix — they sit at the server root (e.g. `http://localhost:8080/api-docs`).
- **CORS:** enabled. Dev reflects any origin; prod uses `CORS_ORIGIN` (comma-separated allowlist).
  The custom header `X-Tenant-Slug` and `Authorization` are allowed; `credentials: true`.

## Tenant resolution (required on every tenant/agency + client call)

Send the tenant slug on **every** request as a header:

```
X-Tenant-Slug: demo
```

(Subdomain resolution also works in prod, but use the header in dev — it takes priority.) Missing/
unknown slug → `404 TENANT_NOT_FOUND`; suspended tenant → `403 TENANT_SUSPENDED`.

## Auth

### Agency app (tenant users) — full session support
- `POST /auth/login` `{ email, password }` (+ `X-Tenant-Slug`) → `{ data: { accessToken, refreshToken } }`
- `POST /auth/refresh` `{ refreshToken }` → new `{ accessToken, refreshToken }` (**rotation** — the old
  refresh token is revoked; store the new one). Access token TTL **15 min**; refresh **7 days**.
- `POST /auth/logout` — send the **refresh** token in `Authorization: Bearer <refreshToken>` to revoke it.
- `POST /auth/forgot-password`, `POST /auth/reset-password`.
- Send the **access token** as `Authorization: Bearer <accessToken>` on authenticated calls.
- The JWT payload is only `{ sub, tenantId, type: 'tenant_user', iat, exp }` — **no user profile in it**.
  Call **`GET /me`** for `{ id, email, name, tenantId, isActive, groups[],
  permissions: string[] }` to render the user + gate UI by permission key.

### Client portal (contacts) — Client-JWT, no refresh
- Onboarding: `POST /auth/onboarding/send {contactId}` → (emailed link, dev: read token from the
  `auth_tokens` table) → `GET /auth/onboarding/verify/:token` → `{ accessToken }` → `POST /auth/client/set-password`.
- Returning: magic link `POST /auth/client-login/send {email}` → `GET /auth/client-login/verify/:token`
  → `{ accessToken }`; OR password `POST /auth/client/login {email,password}` → `{ accessToken }`.
- Client-JWT TTL **15 min, no refresh** — session ends on expiry (re-auth). Send as `Authorization: Bearer`.

## Response & error envelope (stable)

- **Success:** `{ "data": <value> }`. Paginated lists: `{ "data": [...], "meta": { "nextCursor": string|null,
  "hasMore": boolean } }` (cursor pagination being rolled out this pass; controllers take `?cursor=&limit=`).
- **Error (flat, never nested):**
  ```json
  { "code": "MACHINE_CODE", "message": "english", "requestId": "uuid",
    "field?": "…", "errors?": [{ "field": "…", "message": "…" }], "details?": { } }
  ```
  - Map **`code`** → your i18n strings (the API is locale-agnostic; `message` is English, unlocalized).
  - Validation failures are **HTTP 422** `VALIDATION_ERROR` with `errors: [{ field, message }]` (not 400).
  - Common: `401 UNAUTHENTICATED`, `403 FORBIDDEN` (+`requiredPermission`), `404 NOT_FOUND`,
    `422 <business code>`. Every response carries `requestId`.

## Permissions

Guarded endpoints require a permission **key** (e.g. `quotations.create`, `invoices.view`). Fetch the
current user's effective keys from `GET /me` and show/hide UI accordingly. The full catalog is at
`GET /permission-groups/permissions`.

## Endpoint catalog (current)

> `[key]` = required permission; `[Public]` = no auth; `[Client-JWT]` = portal; `[me]` = any authed user.
> Items marked **(+)** were the read surface added in the frontend-enablement pass — **all now
> landed** (FE-1…FE-6). Cursor pagination (`?cursor=&limit=`, `meta.nextCursor`/`meta.hasMore`) is
> live on the growth-prone lists: leads, contacts, projects, quotations, users, permission-groups.
> Bounded lists (invoices, change-requests, notifications) return a plain `{ data: [...] }` array.

**auth** — see Auth above (all `[Public]`).
**me** — `GET /me` **(+)** `[me]`.
**catalog** — `GET/POST/PATCH/DELETE /services` `[services.*]`, `GET/POST/PATCH/DELETE /packages` `[packages.manage]` (list+detail ✓).
**crm** — `GET/PATCH /sales-flow/stages` `[crm.*]`; `POST /leads` `[crm.create]`; `PATCH /leads/:id/stage|convert` `[crm.edit]`; `GET /contacts/:id/history` `[contacts.view]`; **(+)** `GET /leads` (board, `?stageId`/`?q`), `GET /contacts`, `GET /contacts/:id`.
**blueprint** — `GET/POST/PATCH/DELETE /blueprints` + `/task-boards` (list+detail ✓) `[blueprints.*]`.
**quotation** — `POST /quotations`, `PATCH /quotations/:id`, `POST /quotations/:id/generate-proposal|send-proposal|reopen`, `POST /quotations/blueprint-matches` `[quotations.*]`; client `POST /proposals/:id/amend|accept|reject` `[Client-JWT]`; **(+)** `GET /quotations`, `GET /quotations/:id`, `GET /quotations/:id/proposals`, `GET /proposals/:id`.
**project** — `POST /projects/:id/start` `[projects.start]`, `GET /projects/:id` `[projects.view]`, `PATCH /projects/:id/archive`, `PATCH /projects/phases/:id/hold|resume`, `GET /projects/:id/board` `[tasks.view]`, `PATCH /tasks/:id`, `PATCH /tasks/:id/assignee`, `GET /approvals/queue` + `POST /approvals` `[approvals.review]`, client `POST /feedback` + `POST /phases/:id/approve` `[Client-JWT]`; **(+)** `GET /projects` (staff list, `?status`).
**client-portal** — `GET /portal/projects`, `GET /portal/projects/:id`, `POST /portal/project-requests` `[Client-JWT]`; `GET /project-requests` `[projects.view]`.
**invoicing** — `POST /invoices/:id/payments` `[invoices.record_payment]`, `GET /invoices` (`?projectId`) `[invoices.view]`, `GET /invoices/:id/pdf` (stub URL), `GET /portal/invoices` `[Client-JWT]`; **(+)** `GET /invoices/:id` (JSON), `GET /invoices/:id/payments`.
**change-requests** — `POST /projects/:id/change-requests` `[change_requests.manage]` (+ portal `POST /portal/projects/:id/change-requests` `[Client-JWT]`), `POST /change-requests/:id/approve|reject` `[change_requests.approve]`, `GET /change-requests` (`?projectId`); **(+)** `GET /change-requests/:id`.
**notifications** — `GET /notifications` `[me]` (poll this; no realtime yet), `PATCH /notifications/:id/read`.
**rbac** — `GET /users` + invite/activate/deactivate/groups/overrides `[users.*]`; `GET /permission-groups` + CRUD + `GET /permission-groups/permissions` `[permission-groups.manage]`; **(+)** `GET /users/:id`, `GET /permission-groups/:id`.
**platform** (separate platform-owner console, not the tenant app) — `POST /platform/auth/login`, `GET /platform/me`, plans + tenants CRUD `[Platform]`.

## Known stubs / deferrals (demo runs on these; deploy-phase for real)

- **Real-time (Socket.io):** not served — **poll** `GET /notifications` and lists.
- **File upload/download (S3):** no presign endpoint — file features unavailable in the demo.
- **PDF:** `GET /invoices/:id/pdf` returns a stub URL (unreachable host); no proposal-PDF endpoint.
- **Email:** stub — onboarding/magic-link/reset/invoice tokens are **not delivered**; read them from
  the `auth_tokens` / `password_reset_tokens` DB tables in dev.
- **Notifications:** only the *invoice-generated* event creates a notification so far; other triggers
  (payment, change-request, phase-overdue) + the §13.4 preferences endpoint are not wired yet.
- **Online payment (Stripe):** none — payments are recorded manually via `POST /invoices/:id/payments`.
- **Analytics/dashboards (Phase 7):** no endpoints yet.
