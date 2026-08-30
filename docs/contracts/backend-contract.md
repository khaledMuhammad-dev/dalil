# Backend contract — how `dalil-web` consumes the API

> Pointer doc. **Do not duplicate shapes here** — the authoritative contract is the live
> OpenAPI spec; the human-readable overview is maintained in the backend repo.

## Sources of truth (in order)

1. **`GET /api-docs-json`** — the OpenAPI spec (92 paths at last count). Generate the typed
   client from it: `pnpm openapi-typescript http://localhost:8080/api-docs-json -o src/lib/api/schema.d.ts`
   (wire as `pnpm api:types`; regenerate after every backend change; check the file in).
2. **`frontend-contract.md`** — the maintained human-readable API overview
   (base URL, tenant resolution, auth flows, envelope, pagination, error codes).
3. **`../../dalil-api/agents/context/error-codes.json`** — the error-code catalog (map every code to an
   `errors.<CODE>` i18n key).
4. **`../../dalil-api/fable-5/05-FRONTEND-START.md` §4–§5** — the client-layer rules (envelope unwrapping,
   error→i18n, token lifecycles, query conventions, permission gating).

## The five facts you re-need constantly

| Fact | Value |
|---|---|
| Base URL | `http://localhost:8080/api/v1` (OpenAPI endpoints sit at server root, not under the prefix) |
| Tenant header | `X-Tenant-Slug: demo` on every tenant/client call |
| Success envelope | `{ data }` · paginated `{ data, meta: { nextCursor, hasMore } }` |
| Error envelope | FLAT `{ code, message, requestId, field?, errors?, details? }` — validation is HTTP 422 |
| Sessions | agency: access token 15 min held in **memory**; refresh in an **httpOnly cookie** (7-day rotation, **D-063**) · portal Client-JWT 15 min, no refresh |

## Agency auth flow (D-063 — refresh via httpOnly cookie)

The refresh token is **never** in a response body or readable by JS — it is set/read **only** as
an httpOnly, `SameSite=Strict`, path-scoped (`/api/v1/auth`) cookie. The access token is returned
in the body and held in **memory** (Zustand, never `localStorage`/`persist`). Send
`credentials: "include"` on every call so the cookie flows.

| Call | Request | Success (200) body |
|---|---|---|
| `POST /auth/login` | `{ email, password }` | `{ data: { accessToken, expiresIn, user: { id, email, name } } }` — sets the refresh cookie |
| `POST /auth/refresh` | **no body** (cookie only) | same shape, cookie rotated |
| `POST /auth/logout` | cookie only | `{ data: null }`, cookie cleared |

- Missing/expired/rotated refresh cookie → `401 INVALID_REFRESH_TOKEN`; bad login → `401 INVALID_CREDENTIALS`.
- **Silent bootstrap:** on load/hard-refresh the in-memory access token is gone but the cookie
  survives — call `/auth/refresh` once at startup (200 → seed session; 401 → show login).
- CSRF is covered by `SameSite=Strict` + path-scoping (no CSRF token needed). Dev backend must
  run `COOKIE_SECURE=false` (else `Secure` blocks the cookie over http localhost).

## Backend gaps

Found a missing endpoint / wrong shape / missing code while building? Do **not** patch the
backend from here — log it and route it through the dalil-api factory harness
(`.claude/rules/WORKFLOW.md` §3).
