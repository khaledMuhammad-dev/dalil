# Dalil — Master AI Instructions (the permanent operating manual)

> **Read this first, every session, before touching anything.** This file consolidates every rule,
> trap, and verified lesson from the entire build (Phases 0–6.5). It is written so that an AI agent
> with NO prior context can continue the project without repeating a single past mistake.
> Precedence: `agents/context/fs/**` (FS v2.0) + `02-FS-launch.md` (launch amendments) win on
> behavior · `01-PRD-launch.md` on product · `../decisions/api.md` on rulings · this file on process.
> When two sources conflict → STOP and ask Khaled; never silently pick one.

---

## 0. The project in three sentences

Dalil is a multi-tenant subscription SaaS for creative agencies (KSA/GCC, Arabic-first) unifying
CRM → quotation → proposal → project execution → client portal → invoicing → change requests.
The **backend** (`dalil-api`, this repo) is behaviorally complete through Phase 6 with a
demo-ready read surface, OpenAPI at `/api-docs`, and 1943 green tests; infra side-effects
(email/PDF/S3/realtime/payments) are stub adapters behind DI seams. The **frontend** (`dalil-web`,
separate repo, D-011) has not started — building it to demo, then closing the launch gaps in
`03-GAP-ANALYSIS.md`, is the mission.

## 1. Absolute rules (violating any of these is a defect, not a style issue)

1. **Never hand-write product backend code** (`src/**`, `prisma/**`). Every backend feature slice
   goes through the factory harness (D-060):
   ```bash
   cd /workspace && DALIL_AGENT= ./node_modules/.bin/tsx agents/harness/run-task.ts \
     --task task.txt --module <module> --run-id <id> --leave-open --no-commit 2>&1
   ```
   The Orchestrator (you) writes only: `task.txt`, governance docs, session logs, `state.json`
   (via the state API only, never hand-edit — D-017), and root-file one-liners the harness cannot
   touch (e.g. registering a new module in `src/app.module.ts` — D-036).
2. **Nuclear lock** — never Write/Edit: `agents/**`, `.github/**`, `.husky/**`,
   `prisma/migrations/**`, `.env*`, `.claude/settings.json`, `.claude/hooks/**`.
3. **Tenant isolation**: `tenantId` comes ONLY from the JWT; every tenant-scoped query carries it
   (Prisma middleware injects; every new tenant-scoped model must be added to the
   `DIRECTLY_SCOPED` isolation bucket). A scoped query without `tenantId` is a defect.
4. **RBAC by permission key, never role/group name** — backend guards and frontend UI gating alike.
5. **Money is `DECIMAL(12,2)` / string in JSON — never float arithmetic.** Dates stored UTC,
   returned ISO 8601. Append-only tables (Approval, TransitionLog, Payment, AuthToken) get no
   UPDATE/DELETE ever.
6. **Error shape is FLAT** `{ code, message, requestId, field?, details?, errors? }`; every `code`
   from `agents/context/error-codes.json`; validation = HTTP 422. Success = `{ data }` (+ `meta`
   for cursor pagination). Never invent a code; never nest `{ error: {...} }`.
7. **Spec-first** (D-012): business/flow changes are made by editing the governing docs and
   recording a DECISIONS entry — never by chat-driven code drift. `⚠ HUMAN DECISION` triggers
   (spec conflict, load-bearing schema change, twice-failed gate, irreversible action, PRD open
   item, stub-where-MUST) halt work until Khaled rules.
8. **A committed real credential is a security incident.** Placeholders only in tracked files.

## 2. VERIFY, DON'T TRUST (the discipline that caught every real bug)

After ANY backend change, independently re-run the full gate — never trust a green banner:

```bash
npx tsc --noEmit -p tsconfig.src.json          # want exit 0
npx jest --no-coverage 2>&1 | tail -4           # want ALL green, count went UP
npx eslint src --quiet                          # want 0
```

Then **runtime-prove** it: `pnpm build && node dist/src/src/main.js` → login over HTTP → hit the
new endpoints → assert the DB side-effects. Known traps (each bit us at least once):

- **The `--no-commit` harness banner lies** (`✓ COMPLETED [COMMITTED]` while committing nothing).
  `Commit: no-commit` in the footer is the truth. The Review Agent's PASS is not the gate either —
  it has raised false halts contradicting its own verdict; rule from the source code.
- **Boot the BUILT server only.** `npx tsx src/main.ts` fails (circular import makes `RbacService`
  undefined in `PermissionGuard`). Prod path `node dist/src/src/main.js` (note DOUBLE `src`) works.
  Rebuild `dist` before any runtime proof. Kill stale servers by pid-filtering
  `comm==node && /dist.*main\.js/` — never `pkill -f`, never match `/main\.js/` alone.
- **New module = manual `app.module.ts` registration** (D-036). A 404 on a real route means
  unmounted; a 401 means mounted and guarded. `grep <NewModule> src/app.module.ts` before booting.
- **Runtime-prove every `@Optional()`/cross-module DI wire** — green mocked tests do not prove the
  wire fires. Seed → login → hit endpoint → assert the side-effect row exists.
- **Client-JWT proofs**: mint the token with the SAME `JWT_PRIVATE_KEY` the app boots with — run
  both minter and server under `node --env-file=.env`. A `[DEV-ONLY] … ephemeral` warning in the
  boot log = wrong key = guaranteed 401.
- **Jest**: parallel is reliable (`maxWorkers:'50%'` capped). A red run may be OOM/orphan noise —
  two clean runs at the expected count is the truth. `UniqueConstraintViolation` on
  `*_isolation_test` rows = a previous run was killed before `afterAll`; delete the orphans
  (permission_groups by tenantId first, then tenants, `slug LIKE '%_isolation_test'`). It is NOT a
  code defect. `prisma migrate reset` requires Khaled's explicit consent.
- **Enum-validate every list `?filter` in a Zod query DTO** — a raw string reaching a Prisma enum
  WHERE throws → 500 instead of 422.
- **zod v4 + OpenAPI**: `z.coerce.date()` cannot serialize to JSON Schema and crashes `/api-docs`
  generation for the whole app — use `z.iso.datetime()` for date inputs.
- The literal `::uuid` must not appear anywhere in source, even comments (a scan test forbids it).

## 3. Commit discipline

- Conventional Commits + agent tag: `feat(quotation): add milestone grouping [api-agent]`,
  `chore(state): record fe-6 [orchestrator]`. Subject lowercase, ≤72 chars, imperative, no trailing
  period, **no uppercase tokens** (`FS`, `API`, `CI`, `OpenAPI` all get rejected by commitlint).
  Body lines ≤100 cols. Trailer: `Co-Authored-By:` line for the model in use.
- lint-staged reformats and re-stages — re-verify tsc + suite AFTER committing.
- Governing-doc edits ride in the same commit as the work that caused them (CLAUDE.md §14.2), but
  only after Khaled approves the diff. Commit or push only when asked.

## 4. Backend slice recipe (when a gap requires new API work)

1. Read the module session log + FS contract + `02-FS-launch.md` amendment for the area.
2. Read-only survey of the real surface (controllers, schema, error codes that already exist).
3. Author `task.txt` in the house format (`MODULE / MANDATORY HOUSE PATTERNS / SCHEMA / ENDPOINTS /
   TESTS / SIGNALS`), pre-baking orchestrator-approved rulings so agents don't hard-stop. Name up
   front: the DATA-ACCESS facade/adapter/token pattern (mirror catalog/rbac), `imports: [RbacModule]`
   for guarded controllers (D-035), the isolation-bucket edit for each new tenant-scoped model, the
   D-036 app.module registration reminder, and the exact FS §18 error codes to reuse.
4. `resetSessionCost()` via the state API, then run the harness (see §1.1).
5. Independent gate + runtime proof (§2). New tenant-scoped models: verify migration applied,
   client regenerated, `model.count()` queryable.
6. Hand-commit `feat(...) [api-agent]` + `chore(state) [orchestrator]`; update the session log.
7. If a run dies half-way (`--leave-open`): `git checkout agents/state.json`, `resetSessionCost()`,
   re-run the SAME task.txt with a fresh `--run-id`.

## 5. Frontend rules (dalil-web — see `05-FRONTEND-START.md` for the full guide)

- Separate repo; the OpenAPI spec (`/api-docs-json`) is the only contract. Regenerate types, never
  hand-mirror DTOs.
- Stack locked: Next.js App Router · TS strict · Tailwind + Shadcn · TanStack Query (server state) ·
  Zustand (client state only) · RHF + Zod · i18next · Storybook + test-runner.
- Arabic (RTL) is the default locale; **logical CSS properties only** (`ps-/pe-/ms-/me-`); every
  string through i18next; every data view handles loading/empty/empty-on-filter/error; every shared
  component ships all 8 story states (default/variants/loading/empty/empty-on-filter/error/RTL/dark)
  with `play()` tests (D-007.8). Gate UI by permission keys from `GET /me`.
- Respect backend stubs: poll notifications (no WS), no file upload UI, PDF/email buttons behind a
  "available at deploy" seam, payments recorded manually.
- **The visual design is decided — the Dalil Experience System (DXS).** Tokens, fonts (custom
  "Dalil"/SaudiaSans EN + IBM Plex Sans Arabic AR), and ten screen designs live in the Claude
  Design project `https://claude.ai/design/p/a299c726-a791-4824-b931-1776a2cd9204`; the
  implementation contract is `08-UI-DESIGN-BRIEF.md` §4 + §13. Warm cream/navy/terracotta —
  never generic blue/indigo/gray. Consume semantic CSS tokens only; never invent a color, size,
  radius, shadow, or duration outside the tokens; dark mode = `data-theme="dark"` re-pointing
  the token layer.

## 6. Environment facts (memorize; they don't change)

| Fact | Value |
|---|---|
| API base | `http://localhost:8080/api/v1` (`PORT` env, default 8080) |
| OpenAPI | `/api-docs` UI · `/api-docs-json` (server root, not under api/v1) |
| Boot | `pnpm build && node dist/src/src/main.js` — never tsx |
| Tenant header | `X-Tenant-Slug: demo` on every tenant/client call |
| Agency demo login | `admin@demo.local` / `Demo@123456` |
| Platform login | `owner@dalil.platform` / `Platform@123456` |
| Tokens in dev | email is stubbed — read from `auth_tokens` / `password_reset_tokens` / `user_invite_tokens` tables |
| Auth TTLs | tenant access 15 min + refresh 7 d (rotating) · Client-JWT 15 min, no refresh |
| Suite baseline | 1943 tests / 99 suites / tsc 0 / eslint 0 (2026-07-07) — counts only go UP |
| Branch | `phase0/request-context` (stale name, cosmetic — rename only if Khaled asks) |
| Design system (DXS) | Claude Design project `https://claude.ai/design/p/a299c726-a791-4824-b931-1776a2cd9204` — `styles.css` + `tokens/*.css` + `assets/fonts/` + `*.dc.html` screens; contract in `08-UI-DESIGN-BRIEF.md` §13 |

## 7. Current mission order (post-fable-5 pack)

1. **Frontend demo build** — follow `05-FRONTEND-START.md` slices 0→8 in order; demo-walk (§9 there)
   is the acceptance test.
2. **Launch-gap backend slices** — work `03-GAP-ANALYSIS.md` register in priority order
   (each launch-blocking gap has its resolution spelled out; run each through the harness per §4).
3. **Deploy-phase infra pass** — real email (SES) / S3 presign / PDF / Redis+BullMQ / Socket.io /
   payment gateway per the launch checklist (`06-LAUNCH-CHECKLIST.md`). Needs Khaled's provider
   picks + accounts; stubs stay until then.
4. **Never mention post-launch roadmap items** (`07-POST-LAUNCH-ROADMAP.md`) inside PRD/FS scope —
   launch docs stay launch-focused.

## 8. When to stop and ask Khaled

Spec conflicts · schema/API-shape decisions referenced by multiple modules · a gate failing twice on
the same file · irreversible/destructive actions (`migrate reset`, data deletion, force-push) ·
anything touching the PRD open items (A Stripe/gateway, B trademark, C monitoring/deploy stack,
D pricing, E exact-match rule, F font licence) · publishing/deploying anything externally ·
governing-doc edits (propose the exact diff, wait for approval). Everything else: proceed and log.
