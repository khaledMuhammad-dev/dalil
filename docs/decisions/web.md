# Decision log — `dalil-web` (frontend)

> **Paths in this file are relative to `dalil-web/`** (the frontend repo root), unless
> they start with `../` — those are relative to this file, inside the shared `docs/` tree.

> Frontend-scoped decisions, numbered `FE-D-xxx`. Backend/global rulings live in
> `api.md` and remain binding here where applicable (notably D-007.x code quality,
> D-011 topology, D-012 spec-first). When the FS is silent and Khaled rules, the answer is
> recorded here and becomes binding.

---

## Inherited rulings (binding, recorded for visibility)

- **D-007.3 / D-007.4 / D-007.5** — reusability mandate, predictable module shape, clean-code
  rules. Applied to the frontend via `.claude/rules/CONVENTIONS.md`.
- **D-007.8** — every shared component ships Storybook stories for all 8 states with `play()`
  tests; `@storybook/test-runner` is a gate.
- **D-011** — `dalil-web` is a separate repository, separate git history, separate deploy;
  the API contract is the only coupling.
- **D-012** — spec-first: behavior changes happen in governing docs + a DECISIONS entry, never
  by chat-driven drift.
- **Stack lock** (CLAUDE.md §2 / PRD) — Next.js App Router, TS strict, Tailwind + Shadcn,
  TanStack Query, Zustand, RHF + Zod, i18next, Storybook. Never substitute.

---

## FE-D-001 — Physical placement of `dalil-web` (2026-07-09)

**Context:** D-011 defines `dalil-web` as a sibling repo at `dalil/dalil-web/` beside
`dalil/dalil-api/`. This devcontainer mounts only `/workspace` (= `dalil-api`), so a true
sibling path is not writable.

**Ruling (per Khaled's instruction):** `dalil-web` lives nested at `/workspace/dalil-web/`
with its **own git repository** (separate history preserved), and is ignored by `dalil-api`
via a `.gitignore` entry so the api repo never tracks it. D-011's separate-history/deploy
intent is preserved; only the on-disk placement differs. If later extracted to a true sibling
checkout, copy the `fable-5/` pack in and update the `../` references in the governing docs.

## FE-D-003 — API client: openapi-react-query replaces axios; GSAP for animation (2026-07-10)

**Context:** CONVENTIONS.md §3 mandated one axios instance. Khaled ruled to switch the API
layer to `openapi-react-query` (on `openapi-fetch` + the `openapi-typescript` generated
types), giving fully-typed TanStack Query hooks with no hand-written query functions.

**Ruling (Khaled, 2026-07-10):**
- `axios` is removed. The API layer is `openapi-fetch` (one configured client) wrapped by
  `openapi-react-query`. The axios interceptor duties move to `openapi-fetch` middleware:
  `X-Tenant-Slug` + `Authorization` injection, typed `ApiError` from the flat error shape,
  and the single-flight 401 refresh with queue+replay. The `{ data }` envelope remains in the
  generated types — unwrap via a small shared layer (or accept `query.data.data`), decided at
  Slice 0 implementation.
- Animation: **GSAP only for now** (no Framer Motion). Map DXS motion tokens per
  `../../dalil-api/fable-5/08-UI-DESIGN-BRIEF.md` §13.11.
- Also installed per the same instruction: `@tanstack/react-table`, `socket.io-client`
  (post-demo realtime; polling remains the demo transport), `@playwright/test`, `jest` (+
  jsdom env), and Shadcn UI initialized (`components.json` with `rtl: true` so generated
  components use logical properties).
- CONVENTIONS.md §3 updated to match on 2026-07-11 (Khaled re-confirmed the stack during the
  FE-D-006 orchestration setup); CLAUDE.md §2 aligned in the same pass.

## FE-D-005 — Date/time picker built on date-fns + Intl (calendar-lib ruling, 2026-07-11)

**Context:** The DXS date/time picker was deferred from §02 Forms pending a calendar-library
ruling (the component needs Hijri Umm al-Qura + Gregorian, an antd-style days/months/years
drill, single/range/multi selection, disabling, and a time picker).

**Ruling (Khaled, 2026-07-11):** Build on **date-fns v4** (installed as the app's date
library). date-fns has no Hijri calendar, so the Hijri projection derives from the
platform's `Intl.DateTimeFormat` with `calendar: "islamic-umalqura"` — zero extra
dependencies, and it matches the KSA civil calendar. Consequences:

- All picker values are plain Gregorian `Date`s; the calendar system is display projection
  only (`src/components/ui/date-picker/calendar-system.ts`).
- All labels/digits come from `Intl` (`DateTimeFormat`, `DisplayNames`, `NumberFormat`,
  `ListFormat`) in the `<html lang>` locale — Arabic-Indic numerals and Hijri month names
  with no hardcoded strings.
- Caveat discovered during verification: Umm al-Qura month lengths can differ ±1 day
  between ICU builds (Node vs Chrome) — tests must not hardcode Hijri month boundaries.
- jest (via `next/jest`, no new deps) now covers pure logic (`*.test.ts` under `src/`);
  Storybook play tests remain the UI gate.

## FE-D-004 — Design tokens are rem-based in code (2026-07-10)

**Context:** The DXS token SSOT in the claude.ai design project (`tokens/*.css`) declares
lengths in px. While wiring the tokens into `dalil-web`, Khaled ruled that the code-side
tokens use rem.

**Ruling (Khaled, 2026-07-10):** All length tokens in `src/styles/tokens/` are expressed in
rem (@16px base), keeping the px design reference in a comment next to each value. This also
aligns the token layer with Tailwind's rem-based utilities and respects user font-size
preferences. Exceptions: `--radius-pill` uses `calc(infinity * 1px)` (a cap, not a length),
and ms-based motion tokens are unchanged. The px→rem conversion is a mechanical mapping —
the design project remains the SSOT for the *values*; re-syncs convert on the way in until
the design project itself moves to rem (proposed upstream).

## FE-D-006 — Frontend sub-agent pipeline (supersedes FE-D-002's build-directly model, 2026-07-11)

**Context:** FE-D-002 (provisional) ruled the frontend is built directly with no agent
roster. Before starting slice implementation, Khaled instructed setting up orchestration:
dedicated sub-agents for requirements/spec → TDD → implement → verify → enhancement loop
(max 3) → accessibility/CLS/localization/RTL audit, plus binding optimistic-update,
code-splitting, and reuse/DRY mandates.

**Ruling (Khaled, 2026-07-11):**
- The frontend runs a **five-agent sequential pipeline** per slice/feature, orchestrated by
  the main session (which never implements directly): `fe-spec-analyst` (spec packet) →
  `fe-test-designer` (RED tests) → `fe-implementer` (green, unweakened) → `fe-verifier`
  (spec match + test adequacy + live-API wire proof; PASS/FAIL/ENHANCE) →
  `fe-quality-auditor` (axe 0 · unexpected CLS 0 · 0 missing keys in both locales ·
  LTR + RTL walk). Agent definitions live in `.claude/agents/`; the protocol in
  `.claude/rules/WORKFLOW.md`.
- **Enhancement loop:** FAIL/ENHANCE findings route to the named owner and re-verify —
  hard cap **3 iterations**, then escalate to Khaled. ENHANCE never adds scope.
- Agents end output with a machine-readable `SIGNAL:` line (`NONE` / `HUMAN_DECISION` /
  `FAIL` / `ENHANCE` / `BACKEND_GAP` / `DOC_DRIFT`), mirroring the backend sentinel.
- Mandates recorded as non-negotiables (CLAUDE.md §3.10): optimistic updates with rollback
  as the default for visible-mutation UX; `next/dynamic` splitting for heavy/conditional
  components with parity-sized fallbacks; reuse-before-write / one-helper-per-concern DRY.
- Unchanged from FE-D-002: no `state.json`/hook harness here (session logs carry the loop
  counter), the `fable-5/` pack stays referenced at `../../dalil-api/fable-5/`, and backend gaps route to
  the dalil-api factory — never fixed from this repo.

## FE-D-002 — Cross-repo orchestration model (resolves D-010 for the frontend, 2026-07-09) — SUPERSEDED by FE-D-006 (pipeline); topology/state points remain in force

**Context:** D-010 left cross-repo orchestration open. The carry-forward recommended a simple
mirror: thin governance in `dalil-web`, factory harness stays backend-only.

**Ruling (provisional until Khaled confirms):** No factory harness, no agent roster, no
`state.json` in `dalil-web`. The frontend is built directly, governed by `CLAUDE.md` +
`.claude/rules/*` here, with the Storybook/test-runner gate + the demo walk as the quality
bar. The `fable-5/` pack is **referenced** at `../../dalil-api/fable-5/` (not copied) to avoid doc rot
while the repos share a workspace. Operational state = `_ai-context/SLICES.md` + per-slice
session logs. Backend gaps found during frontend work route to the dalil-api factory harness
(WORKFLOW.md §3), never fixed by hand from here.

## FE-D-007 — Session hardening: refresh in httpOnly cookie, access token in memory (adopts backend D-063, 2026-07-13)

**Context:** Khaled hardened agency session handling on the backend (`dalil-api` D-063, commit
`f3d07ac`): the refresh token is delivered/accepted **only** as an httpOnly, `SameSite=Strict`,
path-scoped (`/api/v1/auth`) cookie — never in a response body — and the access token is
returned in the body to be held in memory. This overrides FS-2.2.1/2.2.2 and supersedes the
earlier "v1 stores refresh in `localStorage`; httpOnly-cookie move is post-demo" plan.

**Ruling (Khaled, 2026-07-13):**
- The access token + `user` live in **memory only** (Zustand — never `localStorage`/`persist`);
  the refresh token is the httpOnly cookie and is never read by JS. This is the XSS defense.
- The one `openapi-fetch` client sends **`credentials: "include"`**; `POST /auth/refresh` takes
  **no body** (the cookie carries it); `/auth/login` returns `{ accessToken, expiresIn, user }`.
- **Silent bootstrap:** on load / hard-refresh, call `/auth/refresh` once (200 → seed session;
  401 → login), since the in-memory token is gone but the cookie survives.
- CSRF is covered by `SameSite=Strict` + path-scoping (no CSRF token). Dev backend runs
  `COOKIE_SECURE=false`; prod uses a shared parent domain + `Secure`.
- Responses are now typed at `/api-docs-json` (backend `SessionResponseDto`) — `res.data` on the
  auth endpoints is fully typed after `pnpm api:types`; no hand-written response types.
- Governing docs updated in the same change: `CLAUDE.md` §3.6, `.claude/rules/CONVENTIONS.md`
  §3/§4, `../contracts/backend-contract.md` (+ its `ds-bundle` copy). Slice 0 builds to this contract.

## FE-D-008 — Storybook freeze: agents never modify Storybook files (2026-07-13)

**Context:** Khaled ruled that agents must never change anything Storybook-related — the
config, the story files, or the build output. Enforcement is mechanical, not advisory: a
`PreToolUse` hook (`.claude/hooks/protect-paths.sh`, wired in `.claude/settings.json`)
denies `Edit`/`Write`/`NotebookEdit` on protected paths and denies `Bash` commands that
appear to mutate them.

**Ruling (Khaled, 2026-07-13):**
- **Frozen (hard deny):** `.storybook/**`, every `*.stories.*` file, `storybook-static/**`,
  `debug-storybook.log`. Read-only access (and running `pnpm storybook` / `storybook:test`)
  stays allowed.
- Also denied by the same hook: hand-edits to `src/lib/api/schema.d.ts` (generated only —
  `pnpm api:types`).
- Governing docs (`CLAUDE.md`, `.claude/rules/*`, `DECISIONS.md`) prompt for explicit
  approval instead of editing silently (mechanizes the §8 stewardship loop).
- **Open tension with D-007.8:** the Storybook mandate requires new shared components to
  ship 8-state stories, which agents can no longer write. Until Khaled amends D-007.8 or
  lifts the freeze per-task, new-component stories are authored by Khaled or the freeze is
  explicitly relaxed for that task; the pipeline flags the gap instead of writing stories.
- The `storybook*` scripts inside `package.json` cannot be path-blocked (shared file);
  agents treat them as frozen by convention.

## FE-D-009 — Backend gap: live auth is pre-D-063; frontend still builds to D-063 (2026-07-13)

**Context:** during Slice 0.A a live probe (backend reachable from the devcontainer at
`http://host.docker.internal:8080`) showed the running backend implements the old
body-token flow: `POST /auth/login` (201) returns `{ data: { accessToken, refreshToken } }`
with **no `Set-Cookie`** and no `expiresIn`/`user`; `POST /auth/refresh` rejects the
D-063 no-body shape (422) and returned 500 on the body shape. `GET /me` matches the
contract. The live OpenAPI spec is byte-identical to the checked-in `schema.d.ts`.
Full probe evidence: `_ai-context/modules/slice-0-foundation.session-log.md`.

**Ruling (Khaled, 2026-07-13):** the frontend builds to **D-063 / FE-D-007** as written
(httpOnly-cookie refresh, in-memory access token, no-body refresh, single-flight +
queue/replay, silent bootstrap). Consequences:

- Auth flows are wire-proven against contract-exact route mocks until the cookie-session
  slice lands in `dalil-api`; `/me` and other non-auth endpoints verify against the live
  backend. A live auth re-verification pass is owed after the backend catches up.
- The gap (cookie session + login response `expiresIn`/`user` + refresh 500) is routed to
  the `dalil-api` factory harness as a backend slice, per `WORKFLOW.md` §4.
- `schema.d.ts` stays as-is (it matches the live spec); it is regenerated via
  `pnpm api:types` only after the backend change lands.

## FE-D-010 — Locale lives in the URL path prefix (`/ar/*`, `/en/*`), overriding CONVENTIONS §5 (2026-07-15)

**Context:** Slice 0.B shipped locale via a persisted Zustand + `localStorage` store
(`CONVENTIONS.md §5`: "`dir`/`lang` … persisted (`localStorage`; server pref is post-demo)").
Khaled asked for locale to be driven by the URL (`en/*`, `ar/*`) so screens are
shareable/canonical and the language is unambiguous from the address.

**Ruling (Khaled, 2026-07-15, via the orchestrator's clarifying question):** adopt a
**`[locale]` path prefix**. Consequences:

- Routes live under `src/app/[locale]/…`; a `middleware.ts` redirects any bare path to the
  remembered (`dalil.locale` cookie) or default (`ar`) locale, and echoes the active locale
  in an `x-app-locale` header so the server root layout stamps `<html lang dir>` at SSR (no
  post-hydration re-stamp; the old localStorage first-paint script is retired).
- The URL is the **single source of truth** (`src/lib/i18n/use-locale.ts` reads the route
  param). The Zustand locale store is retained ONLY as a non-React mirror the api client
  reads for `Accept-Language`; Providers sync it from the URL. Every locale switch (top-nav
  language action + login switcher) navigates the URL and writes the cookie.
- Every internal link/redirect is locale-prefixed via `localePath(locale, path)`.
- **This supersedes the `localStorage`-locale wording in `CONVENTIONS.md §5`** for the
  active-locale mechanism (the store's `localStorage` persistence is now vestigial). A
  `CONVENTIONS.md §5` amendment to match is proposed for Khaled's stewardship approval
  (governing-doc edit, `CLAUDE.md §8`) — recorded here as the interim authority.
- Companion change (#4b): a reusable `useQueryParam` hook makes search/filter state a URL
  query param (the top-nav global search writes `?q=`); list filters adopt it in slices 1+.

## FE-D-011 — Read discipline: the orchestrator directs what each agent reads; finished work is archived out of the read path (2026-07-15)

**Context:** feature builds were slow because pipeline agents intake far more context than they
use — every stage re-read whole governing docs, the full slice session log, and 30–40 KB spec
packets before doing any work. The `_ai-context/modules/` logs are append-only and never
pruned (~170 KB), and a 1.2 MB standalone component-system HTML was tracked in git, riding
along in every search.

**Ruling (Khaled, 2026-07-15):** intake is the bottleneck; cut it without cutting coverage.

- **Orchestrator-directed reading.** Every delegation carries an explicit `READ:` list — the
  exact files/sections that stage needs. Agents read only `PIPELINE-ENV.md` + their stage's
  default reads + what the handoff names; they never open other governing docs end-to-end,
  sibling modules, `_ai-context/modules/archive/**`, or historical spec packets unless routed.
  A missing fact is asked upward, not trawled. (`.claude/rules/WORKFLOW.md` §1a; per-stage read
  map in `_ai-context/PIPELINE-ENV.md`.)
- **Tiered CLAUDE.md §1.** Governing docs are cited from base context, not re-read; the only
  always-reads are `SLICES.md` + the active slice log; everything else is a load-on-demand map.
- **Archive.** Completed-slice session logs and consumed spec packets move to
  `_ai-context/modules/archive/` (out of the read path, read only when a handoff routes there).
  The 1.2 MB `DXS-07 - Component System (standalone).html` is untracked + gitignored (kept on
  disk; superseded by the `ui-components` skill).
- **Governing-doc edits approved.** This records Khaled's stewardship approval (`CLAUDE.md` §8)
  of the accompanying edits to `CLAUDE.md` §1 and `WORKFLOW.md` §1a.

## FE-D-012 — Optional simple pipeline (TDD → implement → verify); orchestrator offers the mode at task intake (2026-07-16)

**Context:** the full five-stage pipeline (spec-analyst → test-designer → implementer → verifier
→ quality-auditor) plus orchestrator deliberation makes each task slow; Khaled finds the latency
too high for small, well-scoped UI work.

**Ruling (Khaled, 2026-07-16):** offer a lighter path, chosen per task.

- **Two modes.** FULL = the FE-D-006 five stages. SIMPLE = three agents —
  **fe-test-designer (TDD) → fe-implementer → fe-verifier** — dropping the standalone
  fe-spec-analyst (the orchestrator folds the requirements/reuse/error-code notes into the
  test-designer handoff) and the standalone fe-quality-auditor (fe-verifier does the
  a11y / l10n / RTL spot-check inline).
- **Choice at intake.** When handed a task the orchestrator asks Khaled which mode, unless he has
  set a session default. SIMPLE is the recommended default for focused UI/feature tweaks; FULL for
  whole slices, contract/schema changes, auth/permission/security-touching work, or anything risky.
- **Bar unchanged.** Either mode still meets the §6 gates (tsc 0 · eslint 0 · jest green · both
  locales · loading/empty/empty-on-filter/error · touched demo-walk steps). SIMPLE folds stages 1
  and 5 into 2 and 4; it does not lower the bar. The 3-iteration enhancement cap still applies.
- **Governing-doc edits approved.** Records Khaled's stewardship approval (`CLAUDE.md` §8) of the
  accompanying notes in `CLAUDE.md` §5 and `WORKFLOW.md` §1.
