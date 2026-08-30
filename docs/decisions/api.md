# Decision log — `dalil-api` (backend / global)

> **Paths in this file are relative to `dalil-api/`** (the backend repo root), unless
> they start with `../` — those are relative to this file, inside the shared `docs/` tree.

> Every decision the human makes during the build is recorded here, numbered, before
> work resumes. Agents reference decisions by number. When the FS is silent and the
> human rules, that ruling is binding and lives here. Append-only; never rewrite a
> past decision — supersede it with a new numbered entry.

## Format

```
### D-NNN · <short title>
- **Date:** YYYY-MM-DD
- **Context:** what triggered the decision (task, FS sect)
- **Decision:** the ruling, stated unambiguously
- **Consequence:** what this binds going forward
- **Supersedes:** D-NNN (if any)
```

---

## Factory Architecture Decisions (pre-loaded — settled before the build)

### D-001 · MCP access model for Phase 0
- **Date:** 2026-06
- **Decision:** The factory's MCP server uses filesystem + Prisma CLI access in
  Phase 0 (the Dalil API does not yet exist), gated by an enforced path allowlist
  that expands phase by phase. Data operations route through Dalil's own API only
  once those endpoints exist.
- **Consequence:** Stage 5 builds the MCP with a tool-level path allowlist, not raw
  OS access.

### D-002 · Agent placement
- **Date:** 2026-06
- **Decision:** Agents are CLI scripts under `/agents/`, sharing root `node_modules`
  but never imported by `/src/`. The factory never ships to production.
- **Consequence:** Clean factory/product boundary; Option B (in-product AI) is a
  separate future `/src/modules/ai/`.

### D-003 · Quality gate model
- **Date:** 2026-06
- **Decision:** Three-stage gate — agent self-check against the FS contract →
  harness dry-run on a temp branch (one self-correction attempt) → Orchestrator
  commit with Husky hooks as the final safety net. Stage-2 failures return to the
  agent; Stage-3 failures escalate to the human.
- **Consequence:** Built into the harness in Stage 4.

### D-004 · Execution model for Phase 0
- **Date:** 2026-06
- **Decision:** Strictly sequential agent execution. Test-driven: Schema → Test
  (failing tests from the FS) → API (make them pass) → Review. Parallelism deferred
  to later phases.
- **Consequence:** Encoded in CLAUDE.md §6–§7.

### D-005 · Orchestrator state
- **Date:** 2026-06
- **Decision:** `state.json` (Zod-validated, committed to git) for macro state, plus
  per-module session logs for human-readable reasoning.
- **Consequence:** Built in Stage 4.

### D-006 · Uncertainty triggers
- **Date:** 2026-06
- **Decision:** Six hard-stop triggers as listed in CLAUDE.md §8. Any trigger eSION`
  and halts the task until answered here.
- **Consequence:** Every agent definition (Stage 3) embeds these triggers.

---

## Pending — PRD Open Items (await human ruling before the dependent code is built)

These are flagged in PRD v1.6 §16.2. They are NOT yet decided. When the human rules,
add a numbered D-entry above and mark the item resolved here.

- **A** — Stripe payout support for Saudi bank accounts in SAR / local gateway fallback.
- **B** — Trademark check for "Dalil" and the buslahub.com umbrella brand.
- **C** — Monitoring stack (Sentry + Datadog/BetterStack) and deployment target (ECS / Railway / Render).
- **D** — Validate SAR pricing tiers with the first tenant cohort.
- **E** — Pressure-test the strict exact-match blueprint rule against a real agency's
  service matrix; adopt a fallback only if explosion is confirmed.
- **F** — Confirm "Saudi Air" is a licensed, web-deliverable typeface, or pick an alternative.

---

## Build Decisions (added as the human rules during the yet)_

### D-007 · Engineering standards binding on `CONVENTIONS.md` and the Definition of Done
- **Date:** 2026-06
- **Context:** Human-mandated code-quality standards must govern all product code.
  Captured now so they are binding before `CONVENTIONS.md` is authored in Stage 3.
- **Decision:** `CONVENTIONS.md` (Stage 3) and the Definition of Done (`CLAUDE.md`
  §10) MUST enforce ach is a hard requirement; an agent
  violating one is a defect caught in review.

  1. **Single source of truth in code.** Every shared concept has exactly one
     definition: error codes (mirroring FS §18) live in one module and are imported,
     never re-declared; status enums are defined once and shared; Zod schemas and
     DTOs are mirrored from a single declaration, not duplicated by hand. No copy of a
     definition anywhere.

  2. **Code-splitting rules.** Explicit boundaries: code shared across modules lives
     in `/src/common/**`; module-specific code lives in `/src/modules/{module}/**`
     and never reaches across into another module. Define a maximum file size / single
     responsibility rule so services and components are split before they sprawl.

  3. **Reusability mandate.** Before writing new code, an agent MUST check whether a
     suitable shared utility, service, or component already exists, and reuse or
     extend it rather than duplicating. Duplicated logic that a shared helpe already
     covers is a review failure. This is the primary defense against AI-generated bloat.

  4. **Consistent intra-module structure.** Every backend module follows the same
     internal shape (controller / service / repository / dto / module file), so the
     codebase is navigable and predictable across all modules.

  5. **Clean-code standards (concrete, enforceable).** "Clean" is defined as testable
     rules, not taste: no magic numbers/strings (named constants), bounded function
     length and single responsibility, descriptive names per the naming convention,
     no commented-out or dead code, no silent `any`, no stubs where the FS mandates
     behavior (reaffirms `CLAUDE.md` §8.6).

  6. **Database seed script.** A `prisma db seed` script (wired in `package.json`)
     seeds the permission catalog, the eight default Permission Groups, a Platform
     Owner, and a demo tenant — so a fresh environment is immediately usable. Built in
     Phase 0; required, not optional.

  7. **Secrets dling — credentials NEVER in any tracked file.** A tracked
     `.env.example` lists every required environment variable **name** with a
     placeholder value only. Real values live exclusively in an untracked `.env`
     (gitignored) and in CI secrets (GitHub Actions). Putting any real credential in
     `CLAUDE.md`, `.env.example`, a seed file, or any other tracked file is a
     forbidden anti-pattern and a hard defect.

  8. **Storybook for reusable frontend components.** Every reusable component (the
     shared/design-system layer — not one-off page compositions) ships with:
     - **Stories** covering each required state: default + variants + loading + empty
       + empty-on-filter + error + RTL + dark mode. The stories ARE the proof these
       required states exist.
     - **Story tests** — interaction tests via Storybook `play()` and the
       `@storybook/test-runner` gate; every story must render without error and pass
       its interactions.
     Ownership: the **Frontend Agent** writ component and its stories; the
     **Test Agent** writes/runs the story tests and MAY write the story interaction
     tests first (test-driven, mirroring the Schema→Test→Implement order). The Review
     Agent verifies a reusable component has stories for all required states before DoD.

- **Consequence:** Stage 3 authors `CONVENTIONS.md` to spell out each rule, expands
  `CLAUDE.md` §10 (Definition of Done) to include them, and encodes the Storybook
  ownership in the Frontend/Test agent files and `WORKFLOW.md`. Storybook +
  `@storybook/test-runner` are added to the locked frontend stack (this patch, Edit 1).

### D-008 · Canonical API response & error shape (FS wins over CLAUDE.md draft)
- **Date:** 2026-06
- **Context:** Drafting `CONVENTIONS.md` surfaced a conflict: `CLAUDE.md` §3.10 (adapted
  from a reference project) specified a nested failure shape `{ error: { … } }` and
  `{ data, meta }` success, while FS §16.1/§16.2 specify a **flat** failure shape and
  `{ data }` success.
- **Decision:** The FS is authoritative (`CLAUDE.md` §0, §13). Canonical shapes are:
  success `{ data }` (+ `meta` for paginated lists, cursor-based); failure flat
  `{ code, message, requestId, field?, details?, errors? }`, no wrapper. Error codes come
  from `error-codes.json`.
- **Consequence:** `CLAUDE.md` §3.10 corrected; `CONVENTIONS.md` §8 encodes it; all API
  work follows this shape. Supersedes the earlier nested phrasing in `CLAUDE.md` §3.10.

### D-009 · DevOps Agent vs. nuclear-locked files — resolution
- **Date:** 2026-06
- **Context:** `CLAUDE.md` §11 forbids any agent from writing `/.github/**`, `/.husky/**`,
  `/.env*`, yet the DevOps Agent's role includes CI and Husky.
- **Decision:** The DevOps Agent owns non-locked infra directly (Dockerfile, devcontainer,
  deploy config, `package.json` scripts, `.env.example`). For nuclear-locked files it produces
  a proposed diff that the **human applies**. The §11 lock stands; autonomous edits to the
  quality gate and secrets are never permitted.
- **Consequence:** Encoded in `devops-agent.md`.

### D-010 · Frontend Agent target repo — OPEN, resolve before Phase 1
- **Date:** 2026-06
- **Context:** The factory lives in `dalil-api`; the frontend is the separate `dalil-web` repo.
  Cross-repo orchestration was never decided (the six factory decisions were backend/Phase-0 scoped).
- **Status:** **UNRESOLVED.** Phase 0 is backend-only, so this does not block now. The Frontend
  Agent file flags it. MUST be decided before Phase 1 frontend work begins.
- **Options to weigh later:** (a) embed a second factory in `dalil-web`; (b) one cross-repo
  orchestrator driving both; (c) a monorepo restructure.

### D-011 · Repository topology — separate repos under a parent
- **Date:** 2026-06
- **Context:** Backend and frontend will be separate repositories for scalability, not a monorepo.
- **Decision:** Parent container `dalil/` holds sibling repos: `dalil-api/` (backend, this repo)
  and a future `dalil-web/` (frontend). Each repo has its own git history and deploys independently
  (backend → AWS Middle East; frontend → web host). The factory currently lives in `dalil-api` and
  governs the backend only.
- **Consequence:** `CLAUDE.md` structure references updated. D-010 (Frontend Agent target repo)
  remains open and is now narrowed: the Frontend Agent will operate in the separate `dalil-web` repo;
  the cross-repo orchestration mechanism is still to be decided before Phase 1.

### D-012 · Business/flow changes are spec-first, never chat-driven
- **Date:** 2026-06
- **Context:** How do business-logic or flow changes reach the built code, given agents do the
  implementation?
- **Decision:** All business, flow, or behavioral changes are made to the **source of truth first**
  (PRD v1.6 / FS v2.0), then propagated: (1) edit the PRD/FS; (2) re-run the Stage-2 indexing so
  `agents/context/` regenerates (with the three-way parity check); (3) record the change in
  `DECISIONS.md`; (4) the factory rebuilds the affected modules against the new contract; (5) the
  §14 stewardship loop updates any governing docs. An agent is NEVER instructed to "change the
  logic" conversationally and trusted to propagate it — the spec is the input, the code is the output.
- **Consequence:** The knowledge base remains the single source of truth. A future capability —
  a "spec diff → affected modules" map — will identify which modules must be rebuilt when a given
  FS section changes; built when the first spec amendment is needed.

### D-015 · Under bypass-permissions mode, the PreToolUse hook is the sole hard gate
- **Date:** 2026-06
- **Context:** The factory runs Claude Code in bypass-permissions mode for unattended automation.
  In bypass mode, `settings.json` `deny` rules do NOT enforce, but PreToolUse hooks DO still fire
  (exit 2 still blocks).
- **Decision:** The `blast-radius-guard` PreToolUse hook is the SOLE reliable security boundary for
  path blast-radius. The `settings.json` `deny` block is defense-in-depth for non-bypass/interactive
  sessions only and must never be the primary gate. The hook's correctness is non-optional and must
  stay exhaustively verified.
- **Consequence:** Any change to the hook requires a green `verify:hook` before commit. Recorded so
  this is not forgotten when the factory runs unattended.

### D-014 · Path-level blast-radius enforced by PreToolUse hook, not MCP
- **Date:** 2026-06
- **Context:** Building the harness clarified that it drives `claude -p`, and Claude Code does file
  ops with built-in tools. An MCP server adds tools but cannot gate built-in Write/Edit/Bash. The
  reliable native mechanism is a PreToolUse hook (exit 2 blocks; works in all permission modes).
- **Decision:** Path-level blast radius (CLAUDE.md §11) is enforced by a **PreToolUse hook** that
  reads the current agent from a `DALIL_AGENT` env var the harness sets per invocation, checks the
  target path/command against that agent's allowlist plus a global nuclear-lock denylist, and exits
  2 to hard-block violations. A complementary `deny` block in `.claude/settings.json` provides
  defense-in-depth on the most critical paths. MCP is **reserved for Phase-0 scoped DB access**, its
  correct use, and is NOT built now.
- **Consequence:** Supersedes the "MCP allowlist, built in Stage 5" phrasing in CLAUDE.md §11.
  The hook script and `.claude/settings.json` are themselves security-critical and added to the
  nuclear lock. Tool-level scoping (read-only vs read-write via `--tools`, from 4b-i) remains the
  first layer; the hook is the second; settings `deny` is the third; OS sandbox is an optional future layer.

### D-013 · Factory engine extracted as a shared package post-Phase-0, with a version-enforcement gate
- **Date:** 2026-06
- **Context:** The backend (`dalil-api`) and frontend (`dalil-web`) are separate repos. How is the
  AI factory reused across them without copying or merging repos?
- **Decision:** Post-Phase 0 — once the harness has proven itself building real backend code — the
  **domain-agnostic engine** (harness: runner, driver, gate, breaker; state module; pipeline logic)
  is extracted into a shared, versioned package (e.g. `@dalil/factory-core`) in its own repo. Each
  product repo **installs** it as a dependency and keeps its own **config** locally: `CLAUDE.md`,
  `.claude/agents/*`, `CONVENTIONS.md`, `DECISIONS.md`, and its gate commands. The knowledge base
  (`agents/context/`) is shared (packaged or synced). Engine = shared; config = per-repo.
- **Timing:** NOT before Phase 0. The shared-vs-local seam is drawn from experience after the engine
  has built real features — not guessed at now. Extracting prematurely would draw the line wrong.
- **Build requirement (record now so it is designed in, not bolted on):** the shared package MUST
  ship a **version-enforcement gate** that plugs into Husky/CI and **blocks a push/merge when the
  installed engine version is stale** relative to the latest published version (e.g. "factory-core
  X.Y.Z available; you have A.B.C — update first"). This is distinct from spec/code drift (D-012):
  a stale *engine* fails the push; stale *code vs spec* is caught by the Review Agent's FS-contract
  check and a future "spec-drift" CI check.
- **Consequence:** D-010 (Frontend Agent target repo) is further narrowed: the frontend factory in
  `dalil-web` will be mostly per-repo config installing the shared `factory-core`. The three earlier
  options (separate factories / one shared factory / shared core) resolve toward the **shared-core**
  model, to be confirmed when the package is extracted post-Phase-0.

### D-016 · Factory agents run under `bypassPermissions`; the PreToolUse hook is the enforcement
- **Date:** 2026-06
- **Context:** Unattended `claude -p` agents spawned with `stdio:["ignore",...]` hang on the default
  interactive tool-approval prompt (no stdin to answer it). Observed as a full-duration timeout with
  zero output and zero cost.
- **Decision:** The pipeline runner invokes every agent with `--permission-mode bypassPermissions`.
  Safety is enforced by the Stage-5 PreToolUse blast-radius hook (D-014/D-015), which fires in all
  permission modes and hard-blocks out-of-allowlist/nuclear writes. Bypass mode removes only the
  unanswerable interactive prompt, not the security boundary.
- **Consequence:** Agents never hang on approval; the hook remains the sole hard gate and must stay
  green (`verify:hook`). Any future move away from bypass mode must restore an answerable approval
  path before doing so.

### D-017 · `state.json` is written only via the state-module API; never hand-edited
- **Date:** 2026-06
- **Context:** During the first live rbac run, a connection drop plus manual edits to `state.json`
  (to silence Zod validation errors) desynced the orchestrator records from the actual on-disk work
  (a correct, applied schema existed but state referenced the wrong run and lacked a checkpoint).
- **Decision:** `state.json` MUST only be mutated through the state-module API
  (`setModuleStatus`/`claimModule`/`releaseModule`/`recordAction`/`saveState`). Manual edits are
  prohibited. If a required correction cannot be expressed via the API, that is a missing state-module
  capability to be added — not a license to hand-edit.
- **Consequence:** Reconciliation after any future desync is done via a small API-driven script
  (see `repair-rbac-002.ts`), keeping every write schema-validated and atomic.

### D-018 · Administrator-protection uses four invariant-specific 422 error codes
- **Date:** 2026-06
- **Context:** FS §2.4.2 mandated a single `ADMIN_PROTECTION_VIOLATION`; FS §18 listed overlapping
  specific codes for the same scenarios. Two agents read the spec and disagreed — a genuine spec
  defect surfaced by the factory at a supervised pause.
- **Decision (LOCKED):** The four Administrator-protection invariants each return a dedicated 422 code:
  `SYSTEM_PERMISSION_GROUP_IMMUTABLE`, `LAST_ADMIN_REMOVAL_FORBIDDEN`,
  `LAST_ADMIN_DEACTIVATION_FORBIDDEN`, `CRITICAL_ADMIN_PERMISSIONS_REQUIRED`. The generic
  `ADMIN_PROTECTION_VIOLATION` and the redundant `CANNOT_DELETE_SYSTEM_GROUP` /
  `CANNOT_DEACTIVATE_LAST_ADMIN` are removed from this contract. The policy keeps the name
  "Administrator Protection" (no rename). FS §2.4.2 and §18 are aligned to this; PRD unchanged (it
  specifies no codes).
- **Rationale:** Distinct user-facing failures need distinct codes (UX, logging, QA, future
  i18n). `SYSTEM_PERMISSION_GROUP_IMMUTABLE` covers future rename/archive, not just delete;
  `CRITICAL_ADMIN_PERMISSIONS_REQUIRED` reflects that only 3 critical perms are mandatory, not all.
- **Consequence:** This contract is final. Spec is the source of truth (D-012): fixed here,
  re-indexed, not patched in code. Future admin safeguards add new codes without changing these four.

### Backlog · Spec-validator should Read markdown mirrors, not parse JSON extracts via Bash
- **Observed (rbac-003):** spec-validator spent ~14 turns running `cat extract.json | python3` to
  reverse-engineer the shape of `error-codes.json` / `terminology.json`, nearly exhausting its turn
  budget before emitting a packet.
- **Improvement (later, not now):** steer the spec-validator to `Read` the `agents/context/fs/*.md`
  and `prd/*.md` mirrors (clean, human-readable) for content, using the JSON extracts only as direct
  key lookups — not as things to parse with Bash. Likely a small instruction tweak in
  `.claude/agents/spec-validator.md` and/or `buildPrompt`.
- **Also noted:** under `bypassPermissions`, non-allowlisted tools (Bash) still run despite
  `allowedTools: Read Grep Glob`. The allowlist does not bind in bypass mode; the PreToolUse hook is
  the real guard (D-016). If we ever want hard tool restriction for read-only agents, revisit the
  permission mode for those agents specifically.
- **Priority:** low — a turn-budget bump (5→15) unblocks the immediate run; this is an efficiency/clarity
  improvement to make later.

## D-019 — Prisma 7 client instantiation & seed wiring

**Status:** Locked
**Date:** 2026-06-29
**Context:** Tenant Bootstrap (tbs-002). The factory's Schema Agent generated a
seed file using the Prisma 5/6 constructor pattern, which fails under Prisma 7.
Cost a four-error debugging loop (missing barrel import → datasources option →
adapter required). Recording so the Auth build doesn't repeat it.

**Decision — the Prisma 7 pattern for this repo:**

1. **Client generation:** `generator client` uses `provider = "prisma-client"`
   (v7), output to `../generated/prisma`. This output has **no barrel `index.ts`** —
   import from the explicit entry point: `from "../generated/prisma/client"`,
   NOT `from "../generated/prisma"`.

2. **Client instantiation:** Prisma 7 requires the **driver adapter**, not a
   connection-string option. Correct:
```ts
   import { PrismaPg } from "@prisma/adapter-pg";
   const prisma = new PrismaClient({
     adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL }),
   });
```
   WRONG (Prisma 5/6, fails at runtime): `new PrismaClient()` with no args, or
   `new PrismaClient({ datasources: { db: { url } } })`.

3. **Dependency:** `@prisma/adapter-pg` is a production dependency.

4. **Seed wiring:** lives in **`prisma.config.ts`** under
   `migrations.seed`, NOT in `package.json`:
```ts
   migrations: {
     path: "prisma/migrations",
     seed: "tsx ./prisma/seed.ts",
   },
```

**Action item (factory):** bake a one-line Prisma-7 note into the Schema Agent's
definition so generated seeds/clients use this pattern from the start. Until then,
the human verifies seed instantiation at Pause 2.

## D-020 — AUTH_TOKEN_INVALID covers password-reset token failures

**Status:** Locked · **Date:** 2026-06-29 · **Context:** auth (auth-002)
FS §18 has no dedicated PASSWORD_RESET_TOKEN_INVALID code. `AUTH_TOKEN_INVALID`
(trigger: "AuthToken not found, already used, or expired") is the canonical code
for PasswordResetToken validation failures (not-found / used / expired). It also
covers client-portal AuthToken failures when that module lands. Do not introduce
a separate reset-token code.

## D-021 — forgot-password does not gate on isActive

**Status:** Locked · **Date:** 2026-06-29 · **Context:** auth (auth-002)
`forgotPassword` issues a reset token regardless of User.isActive. An inactive
user can reset their password but still cannot log in (login enforces isActive).
FS is silent; this is the accepted behavior — no security risk, no enumeration.
Decision: ALLOW (current behavior). Do not add an isActive gate to forgot-password.

## D-022 — NestJS requires decorator compiler flags in tsconfig.src.json

**Status:** Locked · **Date:** 2026-06-30 · **Context:** http-foundation (hf-004)
Any module using NestJS decorators (`@Module`, `@Catch`, `@Injectable`, etc.)
requires `experimentalDecorators: true` AND `emitDecoratorMetadata: true` in
`tsconfig.src.json` (the config ts-jest uses). Without both, NestJS code fails to
compile under Jest.

**Why:** The four service-only modules (rbac, tenant-bootstrap, auth, client-auth)
used plain interfaces and never exercised decorators, so the flags were absent. The
first real NestJS module (http-foundation) failed in the api-agent until the flags
were added. The api-agent cannot fix this itself — `tsconfig` is outside its write
scope — so it surfaces as a hard environment gap.

## D-023 — Prisma 7 import paths for the Jest/CommonJS environment (extends D-019)

**Status:** Locked (extends D-019) · **Date:** 2026-06-30 · **Context:** http-foundation (hf-004)
In any file compiled under Jest (ts-jest / `tsconfig.src.json`):
- Import the PrismaClient CLASS from `generated/prisma/internal/class` — NOT from
  `generated/prisma/client.ts` (which uses `import.meta.url` / `fileURLToPath`, is
  ESM-only, and crashes Jest's CommonJS transform).
- Import the `Prisma` NAMESPACE (types/enums) from
  `generated/prisma/internal/prismaNamespace`.
- Despite the "do not import directly" header warning in those internal files, these
  are the mandated paths for the Jest environment.

**Why:** Agents defaulted to the standard `@prisma/client` / `generated/prisma/client`
paths, causing repeated compile failures. Confirmed by reading the generated files:
`client.ts` has the ESM poison; `internal/class` and `internal/prismaNamespace` are
CJS-safe. Supersedes the D-019 addendum backlog note below.

### Backlog · HTTP/controller layer deferred across four Phase-0 modules
- **Observed (client-auth-cauth-001):** the review agent flagged "HTTP layer never
  built" as CRITICAL — no controller, module, or DTOs. But this is the consistent
  shape of every Phase-0 module: RBAC, Tenant Bootstrap, Auth, and client-auth all
  ship service-layer-only (service.ts + spec.ts). Controllers/DTOs/NestJS wiring
  were deferred throughout.
- **Resolution (FS-3.1 assembly):** the multi-tenancy middleware / app-assembly
  module wires the HTTP layer for all four prior modules at once, alongside the
  @UseGuards permission guard and tenant-from-subdomain resolution. Each needs a
  controller + Zod DTOs + `{ data }` response shape + correct guard (tenant-user
  routes guarded; client/public routes not). Endpoints owed:
  RBAC (permission-groups/users/overrides), Tenant Bootstrap (POST /tenants),
  Auth (login/refresh/logout/logout-all/forgot/reset), client-auth (6 portal routes).
- **Priority:** high — FS-3.1 is a large integration module; scope it accordingly.

### Backlog · client-auth setPassword contact-existence guard (MEDIUM)
- **Observed (client-auth-cauth-001):** `setPassword` calls
  `contact.update({ where: { id: contactId } })` without confirming the contact
  exists. Invalid contactId → Prisma P2025 bubbles as an unstructured 500 instead of
  a clean VALIDATION_ERROR 422. Same bug class as the null-guard already fixed in
  `sendOnboardingToken` during this run.
- **Improvement (FS-3.1):** add a `findUnique` guard before `update` + a test, when
  the client-auth controller/DTO layer is built. Low real-world risk (contactId
  comes from a validated JWT), but a genuine crash path.
- **Priority:** medium.

---

## D-024 — setPassword P2025 guard throws AUTH_TOKEN_INVALID at 401 (not 422)

**Context:** `ClientAuthService.setPassword` let a Prisma `P2025` (record-not-found, from a JWT
carrying a contactId with no matching Contact) surface as a raw 500. The cpa-001 task string asked
for a 422; the spec-validator flagged a SPEC_CONFLICT because `AUTH_TOKEN_INVALID` is mapped to
**401** in the SSOT (`error-codes.json` / FS-18).

**Decision:** The guard catches `P2025` and throws `DalilError(AUTH_TOKEN_INVALID, 401)`. We keep
**one code → one HTTP status**: `AUTH_TOKEN_INVALID` is always 401, everywhere. We do NOT introduce a
422 variant or a second code for this path. (Rejected: option B = 422 same code; option C = new
`CONTACT_NOT_FOUND` 422 code.) The triggering condition — an invalid client identity — is
semantically an auth-token failure, so 401 is correct anyway.

**Enforced by:** the cpa-002 test `P2025 → 401 AUTH_TOKEN_INVALID through full filter stack`.

---

## D-025 — supertest is imported as a default import

**Context:** The api-agent hit an interop wall: `import * as request from 'supertest'` does not produce
a callable under this TS/ts-jest config.

**Decision:** Use `import request from 'supertest'` (default import). `esModuleInterop: true` is set, so
the default-import form is correct and callable. Any e2e-lite / supertest spec uses this form. Namespace
import (`* as request`) is the wrong form here and must not be reintroduced.

---

## D-026 — DataEnvelopeInterceptor wraps void/undefined as { data: null }

**Context:** 200-returning routes whose handlers resolve `undefined` (e.g. logout, void service calls)
were emitting an empty body, breaking the universal success-envelope contract (FS-16.1: `data` key
always present).

**Decision:** `DataEnvelopeInterceptor` maps `value === undefined` → `{ data: null }` before the normal
`{ data: value }` wrap. The `data` key is therefore always present on every success response, including
void routes. (Paginated/pre-shaped responses still pass through untouched per the existing guard above
this branch.)

**Enforced by:** the cpa-002 e2e-lite requestId test (which exercised a void route).

---

## D-027 — Real crypto lives in src/common/crypto/; adapters thin-delegate

**Context:** During aad-001 the crypto/JWT logic was extracted into shared modules rather than living
inside each adapter.

**Decision:** The real cryptographic primitives live in `src/common/crypto/`:
- `bcrypt-crypto.ts` — `hashPassword` (bcryptjs, **cost 12**), `verifyPassword` (bcrypt.compare,
  constant-time, false on malformed hash), `generateOpaqueToken` (`crypto.randomBytes(32)` → base64url),
  `hashToken` (SHA-256 hex). Per FS-16.4 + FS-2.5.
- `rs256-jwt.ts` — `resolveRs256Keypair()` loads `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY` from env; if
  absent, an **ephemeral 2048-bit RSA keypair** at runtime with a loud `[DEV-ONLY]` warning
  (never hardcoded, never committed — consistent with D-007.7). `signRs256Token()` produces a
  proper `header.body.sig` RS256 JWT via Node `crypto.createSign('SHA256')`.

The module adapters (`*/adapters/crypto.adapter.ts`, `jwt.adapter.ts`) are thin `@Injectable()` wrappers
that delegate to these shared functions. bcryptjs (pure-JS) is the chosen impl over native `bcrypt` to
avoid native-build friction in Docker/devcontainer.

---

## Backlog · Factory empty-output escalation
When an agent step returns `ok: false, output: ""` (empty), the pipeline routes it
identically to a real FAIL verdict and escalates. Observed on mt-001 (after a
timeout) and mt-003 (clean halt: api-retry + second review both returned empty
output). Empty output is pipeline noise — a dropped connection or aborted turn —
not a code verdict. Fix: distinguish empty-output from a genuine FAIL; treat
empty-output as a retryable transport failure, not a quality gate failure.

## Backlog · Spec-validator substring-scanner false-positive halt
The driver's signal detector scans agent output text for the literal phrase
`⚠ HUMAN DECISION` and fires HALTED_HUMAN. This false-fires whenever the
spec-validator writes the phrase in EXPLANATORY text (e.g. "No ⚠ HUMAN DECISION
triggers"). Hit on Bootstrap once, then mt-002 and mt-003. Workaround: manual-continue
past it; the spec packet is valid. Partial fix may already exist (see commits
`9e651cf` / `11803a3` — structured SIGNAL sentinel + negation-aware detector);
confirm whether this is fully resolved or still scanning narrative text.

## Backlog · Production extension factory untested by pure-logic tests
In multitenancy, the Test Agent covered the pure function `applyTenantContext`
thoroughly (33 tests) but did NOT exercise the production `$extends` / 
`createTenantExtension` wiring — so a broken production factory passed all tests.
The real review caught it; the human fix added 3 wiring tests. Lesson for the
Test Agent: when a module ships a pure helper PLUS a thin production-wiring shell,
the shell needs at least one test that drives the real factory, not just the helper.

## Backlog · D-019 addendum — generated client is ESM-only, breaks jest
`generated/prisma/client.ts` uses `import.meta.url` (ESM-only), which the jest
CommonJS transform cannot parse — importing it from a `.spec.ts` crashes the suite.
Import the `Prisma` namespace from `generated/prisma/internal/prismaNamespace`
instead (still D-019-compliant: generated client, not stock `@prisma/client/runtime`).
This is the exact source `client.ts` re-exports `Prisma` from. The schema-agent
should bake this in so the next test-importing module doesn't rediscover it.

### D-028 — Boot check is `tsc → node`, never `tsx`
Boot verification compiles then runs Node against the emitted entry, NEVER `tsx src/main.ts`.
tsx/esbuild does not emit `emitDecoratorMetadata`, so every type-based DI param resolves
undefined and Nest throws "can't resolve dependencies … index [0]" regardless of correct
wiring — a wrong-probe artifact, not a bug. Correct recipe: `npx tsc -p tsconfig.src.json`
then `node "$(find dist -name main.js | head -1)"`. NOTE the real emit path is
`dist/src/src/main.js` (double-`src`-nested, per this tsconfig's outDir/rootDir) — locate it
dynamically, do not hardcode `dist/src/main.js`. Seeded tenant slug for real-request probes is
`demo` (`test` → 404 TENANT_NOT_FOUND before the guard). A guarded route with no token → 401
proves the JwtAuthGuard→PermissionGuard chain resolves and rejects.

### D-029 — RBAC assign route is `PUT /users/:id/groups/:groupId`
Group assignment is idempotent, so PUT is the correct verbot pin the verb.
Ratifies the drift from the B2b kickoff's original POST.

### D-030 — PrismaService exposes model delegates via Proxy-forwarding, not `extends`
PrismaService forwards unknown property access to the private runtime client via a Proxy
(binding functions for correct `this`), rather than extending the client. Extends breaks because
the `getPrismaClientClass()` Jest mock is a constructor that RETURNS AN OBJECT, so `super()`
replaces `this` and strips lifecycle methods / breaks `instanceof`. A real-boot regression test
(prisma.service.spec.ts) guards it: asserts delegates like `svc.tenant.findFirst` are functions.
Verified holding against the live PrismaPg driver-adapter client (Claude Code, this session).

### D-031 — main.ts loads `.env`; boot check must supply DB env
The compiled server entry (`main.ts`) imports `dotenv/config` before any env read. Without it,
booting via bare `node dist/src/src/main.js` leaves `DATABASE_URL` undefined → `PrismaPg('')` →
every query throws `SASL: client password must be a string`, which the tenant middleware's catch
collapses to 404 — an intermittent-looking failure invisible to the 741-green suite (specs +
tsx probes load dotenv; the server binary did not). Lesson: a swallowed DB error is only visible
because Fix 2 (3bc4f98) logs it to stderr — keep pre-auth failures loud. Boot-check recipe stands
(tsc → node dist/src/src/main.js), now with `.env` loaded by main.ts itself. NOTE: `.env` must
carry real `JWT_PRIVATE_KEY`/`JWT_PUBLIC_KEY` for token continuity across restarts (else ephemeral
dev keypair per boot) — a deploy/DevOps requirement, not dev-blocking.

### D-037 — Phase 3 (Quotation & Proposal, FS §6) is sliced; slice 1 = quotation core through DRAFT
**Status:** Locked · **Date:** 2026-07-03 · **Context:** quotation kickoff (Khaled ruling) · **Triggers:** CLAUDE.md §8.1 (PRD-15 vs FS-17.3 conflict), §8.2 (load-bearing scope)

FS §6 is built in slices, not one run. **Slice 1 scope:** create quotation → exact-match blueprint
filter (FS-6.2) → total computation (FS-6.3) → milestone-grouping validation (FS-6.4.*), tenant-side
(`quotations.create`), through Quotation status **DRAFT**. This is the WORKFLOW.md §6 canonical
teaching feature and is fully unblocked today.

**Explicitly deferred out of slice 1 (each with a named precondition):**
- **Proposal lifecycle** (generate/send/amend/accept/reject, FS-6.6): the client-facing
  amend/accept/reject endpoints are `[Client actor]` (Client JWT) and **no Client-JWT auth guard
  exists in the repo yet** (`client-auth` issues tokens but has no session guard). Building those
  endpoints is an auth-infra precondition → later slice.
- **Accept → PROJECT_STARTED + blueprint snapshot** (Snapshot-on-Start → `ProjectPhase`,
  CLAUDE.md §3.9): **conflict resolved here.** PRD-15 lists "accept/start + snapshot" under Phase 3,
  but **FS-17.3 assigns the ProjectPhase snapshot to Project Start (§8.2)** — a later phase — and
  neither `Project` nor `ProjectPhase` exists yet (`blueprint.service.ts` `countLinkedProjects()` is
  a stub returning 0). **FS wins on behavior (CLAUDE.md §13):** the snapshot is Project-execution
  work, deferred to the phase that introduces `Project`/`ProjectPhase`. Slice 1 does NOT stub a
  snapshot (would violate §8.6).

Milestones in slice 1 group **live blueprint `Phase` rows** at quote time (`MilestonePhase.phaseId`
→ `Phase.id`); the snapshot to `ProjectPhase` is a separate, later concern.

### D-038 — MILESTONE_BASED milestone triggerType literal is `ON_MILESTONE_COMPLETION`
**Status:** Locked · **Date:** 2026-07-03 · **Context:** quotation kickoff (Khaled ruling) · **Trigger:** CONVENTIONS §1 (enum value absent from the FS index → escalate, never invent)

The FS names milestone `triggerType` literals only for the one-time cases — `ON_PROJECT_START`
(ONE_TIME_BEFORE_START) and `ON_COMPLETION` (ONE_TIME_AFTER_FINISH), FS-11. The literal for the
**MILESTONE_BASED** case (a milestone that fires when its own phase set completes) is **not named
anywhere in `fs-index.json`**. Per CONVENTIONS §1 an agent may not invent an enum value; Khaled
ruled the value. **Binding:** `MilestoneTriggerType = ON_PROJECT_START | ON_COMPLETION |
ON_MILESTONE_COMPLETION`. `ON_MILESTONE_COMPLETION` is used for every milestone under a
MILESTONE_BASED payment model. If the FS is later revised to name this literal differently, this is
the reconciliation point.

### D-039 — Quotation modeling & reuse defaults (conservative, PRD-9 + house precedent)
**Status:** Locked · **Date:** 2026-07-03 · **Context:** quotation kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.2 (load-bearing schema), §8.5 (PRD open item E)

FS §6 has **no entity-field table** (FS-19.1/19.2 are future-engineering stubs); field-level
authority is PRD-9. Where PRD-9 is ambiguous, these conservative defaults are adopted:
1. **Selected services** persist via a **`QuotationService` link table carrying `tenantId`**
   (mirrors the `PackageService` precedent, `schema.prisma:296-308`). Not a scalar array.
2. **`MilestonePhase` carries `tenantId`** (same `PackageService` precedent — link tables get
   `tenantId` and join `DIRECTLY_SCOPED`), with a unique constraint scoping one-milestone-per-phase
   **per quotation** (`phaseId` recurs across quotations).
3. **Drop `oneTimeTiming`** — redundant with `paymentModel`'s two `ONE_TIME_*` variants
   (`ONE_TIME_BEFORE_START` / `ONE_TIME_AFTER_FINISH` already encode the timing).
4. **Quotation ↔ Proposal is 1–N**, not 1–1 (PRD-9 says 1–1, but FS-6.7.6 versioning — sequential
   `versionNo`, at most one actionable proposal — requires 1–N). FS wins. (Proposal itself is out of
   slice 1; this fixes the model direction now so slice 2 doesn't remodel.)
5. **Dependency-closed check reuses graph logic via a NEW `src/common/graph/` reachability helper.**
   `blueprint/phase-graph.ts` provides topo-sort/cycle detection but **not** pairwise reachability,
   and lives in another module (CONVENTIONS §4 forbids cross-module internal imports). A fresh
   `src/common/graph/reachability.ts` (transitive closure / path-set over `dependsOnPhaseIds`) serves
   the dependency-closed milestone check. Optionally lift `phase-graph.ts` to `common/` later; not
   required for slice 1.
6. **Exact-match filter = strict set-equality** of the selected service set vs `blueprintServiceSet`
   (no subset/superset). **Flagged provisional** — PRD open item E (design-partner validation) lands
   here (CLAUDE.md §8.5); conservative reading adopted until validated.

### D-040 — `QuotationMilestone` has NO `status` field; it is a stateless billing checkpoint
**Status:** Locked · **Date:** 2026-07-04 · **Context:** quotation-1 run, schema-agent HUMAN_DECISION halt (Khaled ruling) · **Triggers:** CONVENTIONS §1 (enum absent from index), CLAUDE.md §8.2 (shared-entity schema)

The schema-agent halted before writing the migration because `QuotationMilestone.status` (listed in
PRD-9) has **no enum values anywhere** in the FS/PRD/index, and a status enum on a shared entity
(read by the future Phase-6 invoicing module) is a load-bearing §8.2 decision it may not guess.
Investigation (Claude Code, verbatim FS pulled) resolved it decisively: **a milestone has no
lifecycle of its own — every apparent state is DERIVED:**
- *complete?* → FS-11.2 step 107: all `ProjectPhase` rows in the milestone (via `MilestonePhase`)
  have `status = COMPLETED`. Reads **ProjectPhase**, not the milestone.
- *fired/invoiced?* → FS-11.2 step 109 + 11.7.4: an `Invoice WHERE milestoneId = :id` exists. Reads
  **Invoice existence**.
- *paid?* → FS-08 (project COMPLETED = "all milestones paid") + glossary ("Invoice paid status →
  PAID"). Reads **Invoice.status = PAID**.

The authoritative FS glossary lists a status enum for every stateful entity (Invoice, Phase, Task,
Project…) and defines `QuotationMilestone` only as a **"Billing checkpoint"** — no milestone status
enum exists by design. PRD-9 lists `status` as a bare word with **no enumeration**, whereas the same
PRD-9 table enumerates `Invoice.status (UNPAID|PARTIALLY_PAID|PAID|OVERDUE)` — i.e. PRD-9 realized
the statuses it meant, and left this one an unrealized placeholder that the FS (which wins on
behavior, §13) superseded by derivation.

**Ruling:** `QuotationMilestone` carries **no `status` column** and there is **no `MilestoneStatus`
enum**. Fields: `{ id, tenantId, quotationId, name, order, amount DECIMAL(12,2), triggerType }`. If a
future billing slice proves it needs a stored milestone status, that slice defines the FSM + enum
from a concrete spec then — no premature shared-enum lock-in now. (Consistent with D-039.3 dropping
the unenumerated `oneTimeTiming` placeholder for the same reason.)

### D-041 — Quotation slice 2 = tenant-side proposal generate + send (DRAFT → SENT)
**Status:** Locked · **Date:** 2026-07-04 · **Context:** quotation slice-2 kickoff (Khaled ruling) · **Triggers:** CLAUDE.md §8.2 (scope), precondition analysis

FS-6.6 is built in slices. **Slice 2 scope:** the tenant-authored half of the proposal lifecycle —
`POST /quotations/:id/generate-proposal` (`quotations.create`) and `POST /quotations/:id/send-proposal`
(`quotations.send`). Creates `Proposal` (versionNo max+1), enforces the FS-6.7.6 "at most one
actionable proposal per quotation" invariant, transitions `Proposal.status DRAFT→SENT` and
`Quotation.status DRAFT→SENT`.

**Explicitly deferred (each with a named, verified precondition):**
- **Client operations** amend/accept/reject (FS-6.6.3–6.6.5): `[Client actor]` endpoints. No
  `ClientJwtGuard` exists, AND the global `JwtAuthGuard` (`APP_GUARD` in multitenancy) actively
  rejects `type:'client'` tokens. Building the client auth surface is a later slice.
- **Accept's invoice enqueue** (FS-6.6.4 step 67): needs BullMQ + `Invoice` model + Project Start —
  none exist. Cross-phase, deferred with accept.

### D-042 — Proposal PDF / S3 / email are Phase-0 STUB ADAPTERS behind interfaces
**Status:** Locked · **Date:** 2026-07-04 · **Context:** quotation slice-2 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.6 (no-stub rule) — dispositioned here

FS-6.6.1/6.6.2 mandate PDF generation, S3 storage, and an email to the contact. None of that infra
exists and §2 names no PDF library. **Ruling:** these three EXTERNAL INTEGRATIONS are implemented as
stub adapters behind injected interfaces + tokens (`PROPOSAL_PDF_ADAPTER`, `PROPOSAL_STORAGE_ADAPTER`,
`PROPOSAL_EMAIL_ADAPTER`), mirroring the already-shipped `StubClientEmailAdapter` pattern
(`client-auth/adapters/email.adapter.ts`). Real SES/S3/PDF land in a Phase-1 infra slice.

**This is NOT a §8.6 violation:** the FS-mandated BUSINESS behavior — Proposal record, versionNo
sequencing, one-actionable-proposal invariant, both status transitions, persisting a deterministic
`pdfS3Key` — is FULLY implemented and tested. Only the byte-level side effects (rendering PDF bytes,
uploading to S3, delivering email) are stubbed, behind a real seam (interface + token + stub impl),
never a bare `// TODO`. The storage stub returns the FS-6.6.1 deterministic key
`tenant/{tenantId}/proposals/{proposalId}/proposal_v{versionNo}.pdf`; `STORAGE_UNAVAILABLE` (already
in the catalogue) is the failure code. Consistent with the Phase-0 email stub that already shipped.

### D-043 — Proposal modeling defaults (slice 2)
**Status:** Locked · **Date:** 2026-07-04 · **Context:** quotation slice-2 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.2 (load-bearing schema)

1. **`ProposalStatus` is a NEW enum, distinct from `QuotationStatus`** (CONVENTIONS §3 — one concept,
   one enum). Define all 6 PRD-9 values verbatim ahead-of-need (like `QuotationStatus` was in slice 1,
   avoiding a later breaking migration): `DRAFT, SENT, AMENDMENT_REQUESTED, ACCEPTED, REJECTED,
   REOPENED`. (Enum breadth is cheap and spec-grounded — unlike D-040, these values ARE in PRD-9.)
2. **`Proposal`** `{ id, tenantId, quotationId→Quotation (onDelete Cascade), versionNo Int,
   status ProposalStatus @default(DRAFT), sentAt DateTime?, pdfS3Key String?, timestamps }`;
   `@@unique([quotationId, versionNo])` (enforces FS-6.7.6 sequencing); `@@index([tenantId])`;
   `@@map("proposals")`. `Quotation` gains `proposals Proposal[]`. `pdfS3Key` is nullable (PRD-9
   omits it but FS-6.6.1's S3 path requires persisting the key).
3. **`Proposal` joins `DIRECTLY_SCOPED`** (tenant-scoped, house pattern c).
4. **DEFER the `ProposalAmendment` child table** to the amend slice — nothing writes it in slice 2a
   (amend is deferred). Building it now means shipping an untested, unused tenant-scoped table
   (D-040 / D-039.3 precedent: don't build unused structure prematurely). The full enum is defined
   now; the amendment STORAGE is not.
5. **Quotation ↔ Proposal is 1–N** (D-039.4, already ruled).

### D-044 — send-proposal with no actionable proposal → `NOT_FOUND` (404)
**Status:** Locked · **Date:** 2026-07-04 · **Context:** quotation-2 run, review-agent caught a null-deref → 500 (Khaled ruling) · **Triggers:** CONVENTIONS §1 (code absent from spec), CLAUDE.md §8.2

The review-agent found a real defect: `POST /quotations/:id/send-proposal` on a quotation that has
**no proposal generated yet** null-derefs the active-proposal lookup → `TypeError` → 500. FS-6.6.2 /
FS-18.3 assign **no tenant-side error code** for this case (`PROPOSAL_NOT_ACTIONABLE` is client-scoped
per the catalogue), so the api-agent could not pick a code without a ruling.

**Ruling:** `sendProposal` guards the missing active proposal and throws **`NOT_FOUND` (404)** —
message "No proposal to send; generate a proposal first." Rationale: the proposal sub-resource does
not exist yet → 404 is the correct REST semantics; it mirrors the same file's missing-quotation
handling (`proposal.service.ts` already throws `NOT_FOUND` 404 for a missing quotation); it keeps the
client-scoped `PROPOSAL_NOT_ACTIONABLE` out of tenant territory (CONVENTIONS §3); recovery is clear
(call generate-proposal first). Test coverage for send-with-no-proposal is added as part of the fix.

### D-045 — Quotation slice 3 = ClientJwtGuard + client ops (amend/accept/reject)
**Status:** Locked · **Date:** 2026-07-05 · **Context:** quotation slice-3 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.2 (scope)

FS-6.6.3–6.6.5 client operations, built in one slice on top of a new **`ClientJwtGuard`** (D-046):
`POST /proposals/:id/amend {comment}`, `.../accept`, `.../reject {reason?}` — all `[Client actor]`
(Client JWT, NO RBAC key; NOT in the permission catalog). amend → Proposal + Quotation
`AMENDMENT_REQUESTED` + append `ProposalAmendment`; accept → both `ACCEPTED` (idempotent); reject →
both `REJECTED`. Both status flips per op in ONE `$transaction`.

**Deferred (named cross-phase, absent preconditions — dispositioned, NOT bare `// TODO`, per §8.6):**
- **Accept's invoice enqueue** (FS-6.6.4 step 67, ONE_TIME_BEFORE_START): needs BullMQ + `Invoice` +
  Project Start — all absent. Fires when Project Start lands (D-041 already deferred this).
- **All notifications** (FS steps 65/68/70): no `Notification` model, no client-notification email
  infra. Deferred to the notification/infra phase (FS-13).
- **Tenant-side `REJECTED → REOPENED` "revise" transition** (FS-6.5 step 71): not a client op, not yet
  implemented anywhere — a separate future tenant-side micro-op.

### D-046 — ClientJwtGuard design (Client-JWT auth for the client surface)
**Status:** Locked · **Date:** 2026-07-05 · **Context:** quotation slice-3 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §3.1 (tenantId from JWT), §8.2

The repo has no client-session guard, and the global `JwtAuthGuard` (APP_GUARD) rejects
`type!=='tenant_user'`. Build `src/common/guards/client-jwt.guard.ts`:
1. Client routes are `@Public()` (escape the global tenant guard) **and** `@UseGuards(ClientJwtGuard)`.
2. Guard: require `Bearer` (else `UNAUTHENTICATED 401`); `verifyRs256Token` (reuse
   `src/common/crypto/rs256-jwt.ts`; `AUTH_TOKEN_INVALID 401` on bad sig/expiry); assert
   `payload.type === 'client'`; **reconcile `payload.tenantId === req.tenantId`** (the middleware-
   resolved slug tenant) else `AUTH_TOKEN_INVALID 401` — this is the §3.1 enforcement point (a client
   token for tenant A presented on tenant B's subdomain must be rejected, else Prisma scopes to B);
   set `req.contactId = payload.sub`.
3. New `src/common/types/request-with-client.ts` → `RequestWithClient { contactId, tenantId }`.
4. `QuotationModule` provides `JWT_PUBLIC_KEY` (useFactory `resolveRs256Keypair().publicKey`, mirroring
   `multitenancy.module.ts`) — it is not exported from MultitenancyModule — and registers
   `ClientJwtGuard`. Guard ships with a spec (rejects tenant_user / bad-sig / expired / tenant-mismatch;
   accepts valid client token, sets contactId).

### D-047 — Slice-3 modeling + behavior defaults
**Status:** Locked · **Date:** 2026-07-05 · **Context:** quotation slice-3 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.2

1. **`ProposalAmendment`** `{ id, tenantId, proposalId → Proposal (onDelete Cascade), contactId,
   comment, createdAt }`; `@@index([tenantId])`, `@@index([proposalId])`, `@@map("proposal_amendments")`.
   `Proposal` gains `amendments ProposalAmendment[]`. **Append-only by convention** (create-only; no
   UPDATE/DELETE endpoints) — note it is NOT bound by the CLAUDE.md §3.4 hard-immutable list
   (Approval/TransitionLog/Payment/AuthToken), but the amendment audit intent makes it create-only.
   `contactId` (the acting client, from `req.contactId`) is recorded for audit.
2. **`ProposalAmendment` joins `DIRECTLY_SCOPED`** + an isolation-membership test (extend
   `tenant-scoped-prisma.quotation.spec.ts`). FRONT-LOADED — this is the exact class of miss that
   slipped through for `Proposal` in slice 2 (fixed by hand there).
3. **reject `reason` is PERSISTED** as a new nullable `Proposal.rejectionReason String?` — honors the
   FS `{reason?}` contract and captures client feedback (superior to accepting-then-discarding input).
4. **Client-actionable status sets:** amend requires `{SENT, AMENDMENT_REQUESTED}` (allows re-amend);
   accept requires `{SENT}` and is **idempotent** (already `ACCEPTED` → return 200, NO side effects —
   the re-check short-circuits BEFORE the actionable throw, FS-6.7.3/FS-3.6); reject requires `{SENT}`.
   Off-set → `PROPOSAL_NOT_ACTIONABLE 422`. Missing proposal → `NOT_FOUND 404`.
5. Slice 3 does **not** generate proposal versions (that is slice-2 machinery) — it only flips statuses
   and records amendments on the existing actionable proposal.

### D-048 — Quotation slice 4 = narrow "accept→start" vertical slice (opens Project Execution / FS §8)
**Status:** Locked · **Date:** 2026-07-05 · **Context:** quotation slice-4 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.2 (load-bearing schema referenced by >1 module; work crosses into a NEW `project` module)

1. **Scope (narrow vertical slice).** Build the minimum path from an accepted proposal to a running
   project: (a) new `Project`, `ProjectPhase`, `Task` models; (b) **on client accept, create a
   `Project` in `PENDING_START`** (backfills the gap — Slice 3's accept only flipped statuses, never
   provisioned a project, because the model did not exist); (c) `POST /api/v1/projects/:id/start`
   `[PK: projects.start]` implementing FS-8.2 steps 76–80,82: validate `project.status=PENDING_START`
   + `quotation.status=ACCEPTED` (else `PROJECT_NOT_STARTABLE 422`), entitlement check
   (`active project count < max_projects` else `PLAN_LIMIT_EXCEEDED 422`), blueprint snapshot (FS-7.4),
   set `ACTIVE`+`startedAt`, unlock dependency-free phases → `ProjectPhase.status=ACTIVE`+`unlockedAt`.
2. **`SELECT FOR UPDATE`** on the `Project` row across the Start transaction (FS-16.7 lists Project
   Start; unlike the proposal ops in Slice 3, this one IS in the lock table). The whole Start (snapshot
   ProjectPhase + Task rows + set ACTIVE) executes in one transaction (FS-17.3) — partial completion is
   a defect.
3. **DEFERRED to later slices/phases** (NOT built now): task-board movement + phase FSM transitions
   (IN_REVIEW/CLIENT_PENDING/COMPLETED, FS-8.3/8.5), admin approval + client feedback (FS-8.6/8.7),
   project archive/cancel (FS-8.8.7). Slice 4 gets a project *started* with its phase graph unlocked;
   downstream execution is its own module work.
4. **Entitlement count** = `COUNT(Project WHERE tenantId AND status NOT IN [COMPLETED,ARCHIVED,CANCELLED])`
   (FS-15 modeling). A `PENDING_START` project does NOT count until it goes `ACTIVE` (FS-8.1). Query the
   `Entitlement` rows directly; the Redis entitlement cache (`entitlement:{tenantId}`, TTL 300s) is
   deferred with the rest of the caching layer — correctness first.

### D-049 — Slice-4 deferred infra (invoice-on-start + WebSocket events), named
**Status:** Locked · **Date:** 2026-07-05 · **Context:** quotation slice-4 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.5 (absent infra), §8.6 (no silent stubs)

1. **Invoice-on-start (FS-8.2 step 81, `ONE_TIME_BEFORE_START`)** is DEFERRED — needs BullMQ + Invoice
   (absent), same posture as D-045. The Start service records the intent as a prose comment citing this
   decision (NOT a `// TODO`, per §8.6); the invoice fires when the invoicing phase lands.
2. **WebSocket emits (FS-8.2 step 83 `project:started`, FS-8.3.1 step 87 `phase:unlocked`)** are
   DEFERRED — Socket.io is not wired. Chosen over a stub-emitter seam: unlike the Slice-2 PDF/email
   stubs (which had real byte-level *business* effects to fake), these are pure notifications with no
   behavioral contract to satisfy now, so a stub would be dead code. Re-added when the realtime layer is
   built. Both deferrals are logged here so they are not silent gaps.

### D-050 — Slice-4 modeling (Project / ProjectPhase / Task) + cross-module wiring
**Status:** Locked · **Date:** 2026-07-05 · **Context:** quotation slice-4 kickoff (Khaled ruling) · **Trigger:** CLAUDE.md §8.2

1. **`Project`** `{ id, tenantId, quotationId → Quotation, blueprintId (origin ref only — never read for
   live structure, CLAUDE.md §3.9), status ProjectStatus @default(PENDING_START), startedAt DateTime?,
   createdAt, updatedAt }`; `@@index([tenantId])`, `@@map("projects")`. Enum `ProjectStatus {
   PENDING_START, ACTIVE, COMPLETED, ARCHIVED, CANCELLED }` (FS-8.1, verbatim).
2. **`ProjectPhase`** copies `{ name, serviceId, boardId, dependsOnPhaseIds, order }` from the blueprint
   Phase plus `{ id, tenantId, projectId → Project (onDelete Cascade), status PhaseStatus @default(PENDING),
   progress Int @default(0), deadline DateTime?, unlockedAt DateTime?, completedAt DateTime? }`;
   `@@map("project_phases")`. Enum `PhaseStatus { PENDING, ACTIVE, IN_REVIEW, CLIENT_PENDING, COMPLETED,
   ON_HOLD }` (FS-8.3). `dependsOnPhaseIds` stored as the snapshot copy so live unlock logic never reads
   Blueprint.Phase (FS-7.4 step 74).
3. **`Task`** `{ id, tenantId, projectPhaseId → ProjectPhase (onDelete Cascade), title, description?,
   columnId, status TaskStatus @default(TODO), createdAt, updatedAt }`; `@@map("tasks")`. Enum
   `TaskStatus { TODO, IN_PROGRESS, DONE }`. Created at snapshot from `TaskTemplate` rows on the phase's
   board (FS-7.4 step 73); if a board has no templates, zero Task rows (valid — FS-8.8.4 zero-task phase).
4. **All three models join `DIRECTLY_SCOPED`** + isolation-membership tests — FRONT-LOADED (this is the
   exact class of miss that slipped for `Proposal` in Slice 2). Grep-verify each post-run.
5. **New `project` module** (`src/modules/project/**`) owns Project/ProjectPhase/Task logic + the Start
   endpoint. **Cross-module dependency direction: `quotation` → `project`.** The Slice-3 accept flow
   (`proposal.service.ts`) calls `ProjectService` through its **public interface** to provision the
   `PENDING_START` project (CONVENTIONS §4 — never import another module's internals). `QuotationModule`
   imports `ProjectModule`; `ProjectModule` does not import quotation.

---

### D-051 — Phase-3 finish, part 1: tenant "revise → REOPENED" via a dedicated reopen endpoint

Governs the final quotation slice (slice 5). The FS-6.5 FSM mandates `REJECTED → REOPENED` ("admin
revises") → `SENT` (on resend), but nothing writes REOPENED today, so a REJECTED quotation is terminal.

**Ruling:** add a dedicated tenant route `POST /api/v1/quotations/:id/reopen`
(`@RequirePermission("quotations.create")`, `@HttpCode(200)`, on `ProposalsController`) →
`ProposalService.reopen(tenantId, quotationId)`: load quotation; missing → `NOT_FOUND` (404); if
`status !== REJECTED` → `QUOTATION_NOT_EDITABLE` (422, `{ status }`); else flip quotation → `REOPENED`
in one `$transaction`. The old highest-version proposal stays REJECTED (historical); reopen creates NO
new proposal row (that is `generate`'s job). The existing `generate` (v_max+1 DRAFT) + `send` (REOPENED
is already sendable) complete the loop to SENT. No new Quotation row — versioning lives on
`Proposal.versionNo` (FS-6.7.1, FS-6.6.5 step 71).

**Rejected alternatives:** reopen-on-generate (overloads generate with a status-dependent side effect,
hides the FSM edge); reopen-on-PATCH (illegal — PATCH on REJECTED must 422 per D-053).

### D-052 — Phase-3 finish, part 2: one-actionable-proposal invariant enforced at write time (FS-6.7.6)

FS-6.7.6 is a MUST ("system MUST NOT present two live actionable proposals"). Today it is only a
read-time convention; `generateProposal` has no status guard and client actions load a proposal by id
without checking it is the active version — so a stale v1 stays actionable after v2 exists.

**Ruling (no new status, no demotion — invariant holds by construction):**
- The active proposal ≝ `max(versionNo)` for the quotation, unconditionally.
- `generateProposal` gains the D-053 `REVISABLE` quotation-status gate — the real enforcement: you
  cannot mint v2 while a v1 is live-SENT/ACCEPTED.
- Client actions (`amend`/`accept`/`reject`) assert the loaded proposal IS the active (max-versionNo)
  proposal; if a higher `versionNo` exists → `PROPOSAL_NOT_ACTIONABLE` (422, `{ status }`). `accept`
  keeps its idempotent-ACCEPTED short-circuit FIRST, then max-version assert, then status assert.
- `sendProposal` gains a send-time proposal-status guard (latent-bug fix): `send` picks max(versionNo)
  but never checked its status, so after a reopen (active = REJECTED v1) a naive send would resurrect
  it. Add `SENDABLE_PROPOSAL_STATUSES = [DRAFT, AMENDMENT_REQUESTED]`; else `PROPOSAL_NOT_ACTIONABLE`
  (422). Keeps the existing no-proposal→404 first. This forces a `generate` after `reopen`.

**FS-silent micro-rulings (resolved strict, human-approved):** (a) drafting a revision while a proposal
is still live-SENT is BLOCKED (a drafting-while-live workflow would need a SUPERSEDED status the schema
lacks — do not invent one; that would be a §8 escalation); (b) resending an already-SENT proposal →
`PROPOSAL_NOT_ACTIONABLE` (strict; FS silent on idempotent re-email); (c) `reopen` from a non-REJECTED
status → `QUOTATION_NOT_EDITABLE` (no `QUOTATION_NOT_ACTIONABLE` code exists).

### D-053 — Phase-3 finish, part 3: quotation "revisable" editability set (FS-6.7.1)

`updateQuotation` allowed only DRAFT; the revise loop needs AMENDMENT_REQUESTED and REOPENED editable.

**Ruling:** define one constant `REVISABLE_QUOTATION_STATUSES = [DRAFT, AMENDMENT_REQUESTED, REOPENED]`
(collapses the existing `SENDABLE_QUOTATION_STATUSES`, same tuple). It gates `updateQuotation`,
`generateProposal`, and `sendProposal`. Forbidden-to-revise = {SENT, ACCEPTED, REJECTED,
PROJECT_STARTED} → `QUOTATION_NOT_EDITABLE` (422, `{ status }`). REJECTED must be `reopen`ed first
(D-051). Matches FS-6.7.1 ("SENT & ACCEPTED MUST NOT be directly edited").

### D-054 — Phase-3 finish, part 4: BLUEPRINT_DELETED guard at project Start (FS-6.7.2)

The `BLUEPRINT_DELETED` code exists but is unused; `ProjectService.start` never checks whether the
referenced blueprint was soft-deleted, so Start would snapshot a deleted blueprint.

**Ruling (project module, slice 6):** in `start()`, after the quotation-ACCEPTED / PROJECT_NOT_STARTABLE
validation and BEFORE `enforceProjectLimit` + snapshot (step 79), read the blueprint (`where { id:
quotation.blueprintId, tenantId }`, NO `isDeleted` filter so a soft-deleted row surfaces); if null or
`isDeleted === true` → `BLUEPRINT_DELETED` (422, `{ blueprintId }`). Inside the existing `start()`
`$transaction`, after the FOR UPDATE lock (FS-17.3). Requires adding a `blueprint: { findFirst }`
delegate to `ProjectDataClient` + `PrismaProjectDataClient` (project module scope). Remediation per
FS-6.7.2: admin creates a new quotation.

### D-055 — Phase execution FSM: phases are BORN ACTIVE at Start, not transitioned (FS-8.2/8.3)

**Backfill (recorded now; the ruling shipped in project slice C, `feat(project): add phase fsm, unlock
cascade, progress read`, and lived only in `phase.service.ts` code comments — CLAUDE §14 governance debt
closed here).** CLAUDE §3.4 requires a `TransitionLog` on every phase-status change, which could be read
to include the dependency-free unlock at Start (FS-8.2 step 82, PENDING → ACTIVE).

**Ruling:** the initial dependency-free unlock at Start is *snapshot initialization*, not a runtime
transition — the phase is **born ACTIVE**, so it is NOT logged. Every **runtime** phase-status change
(cascade unlock on COMPLETED, IN_REVIEW, CLIENT_PENDING, COMPLETED, revert, ON_HOLD, resume) goes through
`PhaseService.writePhaseTransition`, which writes the append-only `TransitionLog` (entityType `PHASE`).

### D-056 — Terminal board column identified structurally (highest `order`), not by a status flag (FS-8.5.1)

**Backfill (project slice D, `feat(project): task board move, assign and shared board read`).** FS-8.5.1
references a "Done column" but `TaskColumn` has no `isTerminal`/`isDone` flag.

**Ruling:** the terminal ("Done") column is the **highest-`order`** column on the phase's board. Moving a
task there → `Task.status = DONE`; moving it out reverts (FS-8.8.3). No schema flag is added; column order
is the single source of truth for terminality.

### D-057 — Reuse `PHASE_ALREADY_REVIEWED` as the generic non-actionable-phase code; split task permission keys (FS-8.6/8.8.1)

**Backfill (project slices D & E).** The FS defines one phase-state-conflict code, `PHASE_ALREADY_REVIEWED`
(FS-8.8.1), and no generic "phase not in the required state" code.

**Ruling (two parts):** (a) `PHASE_ALREADY_REVIEWED` (422, `{ currentStatus }`) is reused for *every*
non-actionable phase state across approval, client feedback/approve, and (D-059) hold/resume — the caller
always sees where the phase actually is via `currentStatus`; no new code is minted. (b) Task endpoints
carry one permission key each: move → `tasks.manage` (FS-8.5.1), assign → `tasks.assign` (FS-8.8.2).

### D-058 — Client phase-approve served via the Client-JWT scheme, not a literal `/portal/:token` path (FS-8.7.2)

**Backfill (project slice F, `feat(project): client feedback, approve and project completion`).** FS-8.7.2
writes the client approve endpoint as `POST /portal/:token/approve`, but the platform already has a
first-class client authentication scheme (`ClientJwtGuard`, from quotation slice 3).

**Ruling:** the client approve/feedback endpoints are served at `POST /phases/:id/approve` and
`POST /feedback` behind `@Public()` + `ClientJwtGuard` (contact identity + tenantId from the client JWT,
FS-3.1), rather than a bespoke `/portal/:token` route. The FS's `/portal/:token` is honored in spirit (a
token-authenticated client actor); the transport is unified with the existing client-auth scheme. No RBAC
permission key (client actor, not a tenant user).

### D-059 — Phase-4 finish: project archive (FS-8.8.7) + phase ON_HOLD pause/resume (FS-8.3)

**Context:** FS §8 was ~90% implemented across slices C–F, but two behavioral pieces were unbuilt:
FS-8.8.7 archive (the `ARCHIVED`/`CANCELLED` enum values, `projects.archive` key, and `PROJECT_NOT_ACTIVE`
code were defined but wired to nothing) and the FS-8.3 `ON_HOLD` phase state (enum present, no code set or
cleared it). The FS ambiguities were resolved under Khaled's standing "choose the recommended,
business-informed option" directive (this session) in lieu of the CLAUDE §8 hard-stop, and the slice was
built through the **factory pipeline** (spec-validator → schema-agent → test-agent → api-agent →
review-agent, run `project-3`, review verdict **PASS**), then independently verified + runtime-proven.

**Ruling (four parts):**
1. **Archive only; CANCELLED deferred.** `PATCH /api/v1/projects/:id/archive` [PK `projects.archive`]
   transitions ACTIVE/COMPLETED → ARCHIVED in one `$transaction` with `SELECT ... FOR UPDATE` on the
   projects row (FS-16.7). FS-8.1 lists `CANCELLED` but FS-8.8.7 documents no request/trigger for it, so
   CANCELLED is **not** implemented — deferred until the FS specifies its trigger.
2. **Invalid archive state → reuse `PROJECT_NOT_ACTIVE`** (422, `{ status }`) for status ∉ {ACTIVE,
   COMPLETED} — closest existing catalog code, none minted. Entitlement needs no change:
   `enforceProjectLimit` counts only ACTIVE, so archiving frees a slot (FS-8.8.7). BullMQ invoice-job
   cancellation on archive is a Phase-6 infra-gated named deferral (D-049) — the state transition is fully
   implemented, not stubbed.
3. **ON_HOLD toggle guarded by `projects.start`.** FS-8.3 says "manual toggle by a `projects.*` user"; the
   catalog has only `projects.{view,start,archive}`. `projects.start` is the chosen reading of "projects.*",
   reused rather than minting a new key. Endpoints: `PATCH /api/v1/projects/phases/:phaseId/hold` and
   `.../resume`. hold: ACTIVE → ON_HOLD; resume: ON_HOLD → ACTIVE; any other source → 422
   `PHASE_ALREADY_REVIEWED { currentStatus }` (D-057). One `$transaction` + `SELECT ... FOR UPDATE` on the
   project_phases row (FS-16.7); the write goes through `PhaseService.writePhaseTransition` so an
   append-only PHASE `TransitionLog` row is recorded (CLAUDE §3.4). Resume does not re-stamp `unlockedAt`.
   `ProjectService` now injects `PhaseService`.
4. **Still Phase-6 infra-gated (unchanged, D-049):** invoice-on-start, the milestone-completion check on
   phase COMPLETED, notifications, and the WebSocket emits remain deferred — not FS §8 behavioral gaps.

With D-059 shipped, FS §8 is behaviorally complete and the `project` module moves to COMPLETED.

### D-060 — Standing rule: harness-per-slice; no solo implementation

**Context:** in this session the Orchestrator (Claude Code) first implemented D-059 **directly** (authoring
the tests + code + review itself), bypassing the factory pipeline and, critically, the independent Review
Agent PASS the DoD (§10) requires before commit. Khaled flagged the pattern break; the work was reverted
and re-run through the harness (run `project-3`, review **PASS**).

**Ruling:** every substantive feature slice is built through the factory harness, never solo. The
Orchestrator authors the slice spec in `task.txt` (house section format, with any FS-ambiguity rulings
pre-baked as "orchestrator-approved decisions" so the pipeline does not hard-stop), then runs
`DALIL_AGENT= tsx agents/harness/run-task.ts --task task.txt --module <m> --run-id <id> --leave-open
--no-commit` (non-supervised; supervised needs an interactive prompt the Orchestrator cannot drive). The
harness owns test-authoring (independent of implementation) and the Review Agent verdict. The Orchestrator
then **independently** re-verifies out-of-band (tsc + full suite + eslint + a bespoke runtime proof
against real Postgres) and hand-commits (`feat` + `chore(state)`). The only Orchestrator-authored code is
governance/state/session-log/docs, never product `src/**`. Rationale: the factory's non-replicable value
is an independent Test Agent + Review Agent; collapsing roles to save tokens trades away the exact
integrity the factory exists to provide, and it is affordable (a slice runs ~$4–9).

### D-061 — Phase-5: the Client Portal read surface as a standalone `client-portal` module (FS §10)

**Context:** FS §10 ("Client Portal Module") defines a `/portal/...` client-facing surface. The "Approval"
half of Phase 5 was already built inside `ProjectModule` (admin approval queue/decision, client
phase-approve + feedback) and the proposal amend/accept/reject actions live in `QuotationModule` — all on
the shared Client-JWT scheme (D-046/D-058). The genuine Phase-5 remainder was the portal **read** surface
(FS-10.2 project list, FS-10.4 project tracking) plus FS-10.6 project-requests. Built through the factory
harness (run `client-portal-1`, review verdict **PASS**, $8.09), then independently verified +
runtime-proven against real Postgres. FS ambiguities were pre-baked as orchestrator-approved decisions
under Khaled's standing "choose the recommended, business-informed option" directive (no §8 hard-stop).

**Ruling (six parts):**
1. **New standalone `client-portal` module**, not folded into `project`. It is a distinct `/portal` actor
   surface spanning domains and reads project / projectPhase / quotationMilestone / projectRequest through
   its OWN tenant-scoped Prisma facade (`PortalDataClient` + adapter + `PORTAL_DATA_CLIENT` token, D-030) —
   the established cross-module read pattern (like quotation reading catalog/blueprint tables); it imports
   no other module's services (CONVENTIONS §4). Registered in `src/app.module.ts` by the Orchestrator
   (D-036 root-file step, api-agent cannot edit it).
2. **Ownership resolves via `Quotation.contactId`, not a `Project.contactId`** (the Project model has no
   contact column). Portal list/detail scope by the related quotation:
   `where: { quotation: { contactId: req.contactId } }`; `tenantId` is auto-injected on the top-level
   Project query. `getProject` null → `404 NOT_FOUND` (covers missing + not-owned + cross-contact).
3. **`ProjectRequest` is a lightweight record with NO status FSM** (FS-10.6 defines none; it is not a
   Project and cannot self-activate): `{ id, tenantId, contactId, description?, serviceSuggestions[],
   createdAt }`, tenant-scoped (added to `DIRECTLY_SCOPED`). Create is not auto-scoped
   (`WHERE_INJECTION_SKIP_OPERATIONS`), so the service writes `tenantId`+`contactId` from the JWT into
   `data` explicitly. Agency reads it via `GET /api/v1/project-requests` [PK `projects.view`] so the record
   is not write-only (avoids a §8.6 stub).
4. **`GET /portal/projects/:id` state machine** (honors FS-10.4 full-tracking-only-when-ACTIVE + FS-10.8.1):
   ACTIVE → full tracking (phases + client display status + milestone summary); PENDING_START → 'Awaiting
   Start' holding view; ARCHIVED/CANCELLED → read-only summary (`readOnly: true`, no phase detail).
   Project→client label: PENDING_START→'Awaiting Start', ACTIVE→'Active', COMPLETED/ARCHIVED→'Completed',
   CANCELLED→'Closed'.
5. **Phase→client display mapping** over FS-10.4's closed set {Completed, Active, Pending, In Review}:
   IN_REVIEW & CLIENT_PENDING → 'In Review'; COMPLETED → 'Completed'; ACTIVE → 'Active'; PENDING → 'Pending';
   **ON_HOLD → 'Pending'** (baked — ON_HOLD is absent from the FS closed set; a paused phase maps
   conservatively in-set). **Milestone completion is derived** (QuotationMilestone has no status column):
   a milestone is complete ⇔ every ProjectPhase whose `originPhaseId` ∈ that milestone's
   `MilestonePhase.phaseId` set is COMPLETED.
6. **Named Phase-6 infra deferrals (D-049 style, not stubs):** FS-10.7 invoice access (no `Invoice` model),
   FS-10.5 deliverable listing + FS-10.8.4 presigned-URL regeneration (no `Asset` model + needs S3), and
   the invoice-status field on the milestone summary. Each is a one-line comment at the relevant site; no
   phantom endpoints or placeholder fields. With D-061 shipped, FS §10 is behaviorally complete (the portal
   read surface + project-requests) and the `client-portal` module moves to COMPLETED.

### D-062 — Phase 6 architecture: behavioral-first, transport/provider deferred (FS §11–§13)

**Context.** Phase 6 (Invoicing/Payments/Change Requests/Notifications, FS §11–§13) requires external
infrastructure that does not exist in this environment and is gated on external accounts: Redis + BullMQ
(the devcontainer runs Postgres only), Socket.io transport, a real PDF renderer, S3 (`@aws-sdk` presign),
SES email, and a Stripe/SAR payment gateway (PRD open items A & C — §2/§8 escalations). Standing up that
infra cannot be runtime-proven this phase and is a deploy-phase concern.

**Ruling (orchestrator-approved; the same posture as D-042 stub-adapters and D-049 named deferrals).**
Build the full FS §11–§13 **behavioral, money-correct surface** now, DB-backed and runtime-proven, with the
transport/provider layer behind real seams:

1. **All models + endpoints + money-correctness are REAL:** Invoice, Payment, ChangeRequest, Notification
   records; the derived invoice status + `amountPaid` (never stored — D-040 precedent); the
   `milestoneId` XOR `changeRequestId` source invariant (FS-11.1, CLAUDE §3.8); the `OVERPAYMENT` guard;
   Payment append-only (CLAUDE §3.4); the ChangeRequest REQUESTED→APPROVED→INVOICED / REJECTED FSM;
   Notification creation on triggers.
2. **Invoice generation runs INLINE** behind an exported `InvoiceGenerationService` — synchronous, dedup by
   a `@unique` `milestoneId`/`changeRequestId` column + an existence check (the dedup invariant is what
   matters for correctness; BullMQ async/retry/dead-letter is deferred resilience). The planted
   D-041/D-045/D-049/D-059 hooks in `project`/`quotation` are replaced with real calls to this service.
3. **External I/O behind injectable ports + tokens with dev stubs** (D-042 pattern — `INVOICE_PDF_ADAPTER`,
   `INVOICE_STORAGE_ADAPTER`, `INVOICE_EMAIL_ADAPTER`, later `NOTIFICATION_WS_ADAPTER`): only byte-level /
   transport side effects are stubbed; the business logic is real.
4. **Deferred to a deploy-phase infra pass — NAMED, not silent** (a one-line comment citing D-062 at each
   site, never a bare `// TODO`, CLAUDE §8.6): real Redis + BullMQ worker, Socket.io transport, real
   SES/S3/PDF renderer, and the Stripe/SAR payment gateway (PRD-A). WebSocket emits stay D-049-style prose
   deferrals until the notifications slice wires the stub emitter (a real Notification *record* is created
   now; only the transport is deferred).

**FS-silence rulings (conservative, flagged provisional — resolved per CLAUDE §8.5, not FS conflicts):**
(a) no human-readable/sequential invoice number — key by `invoiceId` (FS §11 defines none);
(b) `Payment.method` is a free string (FS enumerates no method values — invent no enum);
(c) `Notification` fields inferred from the FS-13.1 trigger table (no explicit schema in the FS);
(d) minimal payloads for the WS events the FS leaves unspecified (`approval:submitted`, `notification:new`).

**Slicing (one harness run each, D-060):** A = `invoicing` (Invoice + Payment, FS §11); B = wire the
invoice-generation hooks into `project` (FS-8.2 start, FS-11.2 milestone-completion); C = `change-requests`
(FS §12); D = `notifications` (FS §13 + WS emitter stub). Each closes with the D-036 `app.module.ts`
registration for new modules and an independent tsc/jest/eslint/runtime-proof gate. Slice A also closes the
D-061.6 FS-10.7 portal invoice-access deferral (`GET /portal/invoices`).

### D-063 — Refresh token via httpOnly cookie; access token in memory

**Overrides** FS-2.2.1 step 6 (`Return { accessToken, refreshToken, expiresIn }`) and FS-2.2.2 step 7
(`Client posts { refreshToken }`). The refresh token is delivered/accepted **only** as an httpOnly,
`SameSite=Strict`, path-scoped (`/api/v1/auth`) cookie — never in the response body or JS-readable —
mitigating XSS token theft (OWASP). Login/refresh bodies return `{ accessToken, expiresIn, user }`; `user`
(`{ id, email, name }`) is added beyond FS-2.2.1 to save a `/me` round-trip. New env: `COOKIE_DOMAIN`,
`COOKIE_SECURE`, `COOKIE_SAMESITE`. Cookie attributes are env-configurable; prod = shared parent domain +
`Secure`. Then update `../contracts/frontend-contract.md` + `../contracts/backend-contract.md`.

### D-064 — List endpoints sort via `sort` + `order` query params

FS §19.3 names "sorting parameters" for the list API but the retrievable spec index defines no
concrete shape — resolved per CLAUDE §8.5 (FS-silence, conservative). **Human ruling:** every
cursor-paginated list endpoint accepts two optional query params, `sort` (a per-resource
whitelisted column) + `order` (`asc|desc`, default `desc`), e.g. `GET /services?sort=name&order=asc`.

- **Backward-compatible:** omitting the params yields the historical `createdAt desc` ordering.
- **Stable cursors:** `paginate()` appends the row `id` as a same-direction tiebreaker so the
  (column, id) pair is a total order, required for correct id-cursor pagination under any sort.
- **Whitelist discipline:** `sort` is a `z.enum([...])` of real **scalar** columns only. Relations
  and DERIVED columns are excluded — e.g. Invoice `status` / `amountPaid` are computed (D-040), not
  Prisma columns; sorting on them would raise `PrismaClientValidationError` → 500. An unlisted value
  is rejected with `422 VALIDATION_ERROR` at the DTO (the FE-2 lesson: never let an unvalidated
  string reach a Prisma `orderBy`).
- **Scope:** applied to all 18 list endpoints across 12 modules in one pass. Shared primitives
  (`SortOrderParam`, `toSortSpec`, `SortSpec`) live in `src/common/pagination/sort.ts`.
- **OpenAPI:** the params surface automatically from the `createZodDto` DTOs via `cleanupOpenApiDoc`
  (no committed spec file; generated at boot). Verified present with correct per-resource enums.

Recorded in `CONVENTIONS.md` §8.

### D-065 — List endpoints add offset pagination (`page` + `total`) alongside cursor

FS §19.3 specifies **cursor-based** pagination. The frontend list/table views additionally need a
page-numbered control (page N of M, jump-to-page), which cursor pagination cannot serve (no `total`,
no random access). **Human ruling:** every list endpoint gains OFFSET pagination *in addition to*
cursor mode — the two coexist on one endpoint, selected per request. This does not replace or weaken
the FS §19.3 cursor contract (so it is an additive extension, not a §3 non-negotiable defect).

- **Mode selection:** a request carrying `?cursor=` uses cursor mode (FS §19.3, unchanged). Otherwise
  the endpoint is OFFSET-paginated by `page` (1-based) + `limit`, returning that window at
  `OFFSET (page-1)*limit LIMIT limit` over the same filters + `sort`/`order`.
- **`meta.total`:** offset responses add `{ total, page, limit }` to `meta` (the count matching the
  filter *before* pagination), so the FE derives `totalPages = ceil(total / limit)`. `total` comes
  from a `count(where)` issued with the SAME `where` as the page query.
- **Additive, non-breaking:** offset `meta` still carries `nextCursor` + `hasMore` (tail-row cursor,
  `skip + rows < total`), so a returning cursor consumer keeps working. The shared
  `PaginationMetaSchema` gained the three fields as **optional**, so it stays the single meta SSOT.
- **Default page size → 10 (was 20).** Shared `PageParam` (default 1) + `LimitParam` (default 10,
  cap 100) live in `src/common/pagination/pagination.query.ts`; every list DTO imports them (the
  default is defined once, per CONVENTIONS §3/§5). `paginate()` handles offset when given `page` + a
  delegate `count` fn; each `*DataClient` model gained a `count` method (Prisma-native).
- **Scope:** applied to all 18 list endpoints across 12 modules in one pass, mirroring D-064.

Recorded in `CONVENTIONS.md` §8.

### D-066 — Package deletion blocks when referenced by a quotation (`PACKAGE_IN_USE`); usage count on read

FS-4.2 specifies package create/update/expansion but is **silent on deletion**. FS-4.1.3 gives the
Service pattern (in-use → block with `SERVICE_IN_USE`, else hard delete). `Quotation.packageId` is a FK
with `onDelete: Restrict`, so an in-use hard delete currently surfaces a raw Prisma P2003. **Human
ruling (this session):** mirror the Service pattern — deleting a package referenced by ≥1 quotation is
blocked with a **new `422 PACKAGE_IN_USE`**; additionally package reads expose a `quotationCount` so the
UI can show "used by N quotations".

- **New error code:** `PACKAGE_IN_USE` added to the code SSOT `src/common/errors/error-codes.ts` ONLY,
  tagged `// D-066`. The FS §18 mirror `agents/context/error-codes.json` is nuclear-locked (CLAUDE.md
  §11) and stays FS-only; `error-codes.ts` is already a documented superset (FS §16.1 infra codes) and no
  test asserts set-equality between the two.
- **Count mechanism — Prisma `_count` on the package's own back-relation.** `Package` declares
  `quotations Quotation[]`; both the usage count and the guard read `_count: { select: { quotations:
  true } }` through the **existing** `package` delegate. No new `quotation` facade delegate, no `groupBy`,
  no N+1, no NestJS module import or circular dependency — the count stays within the catalog module's
  current data access via the declared relation. (Tenant-consistent: a quotation's `packageId` is within
  the same tenant as the package.)
- **Guard:** `deletePackage` (and `bulkDeletePackages`, per id) reads the package via `findFirst({ where:
  { id, tenantId }, include: { _count: { select: { quotations: true } } } })` inside its transaction;
  null → `NOT_FOUND` (deliberate correctness improvement — deleting a nonexistent package previously
  succeeded silently); `_count.quotations > 0` → `PACKAGE_IN_USE` (422, `details { quotationCount }`);
  else hard delete proceeds (packageService links cascade as today).
- **Usage count on read:** `mapPackage` gains `quotationCount = row._count?.quotations ?? 0`;
  `PACKAGE_INCLUDE` gains the `_count` select so detail / list / create-reload / update-reload all carry
  it. `PackageSchema` gains `quotationCount: z.number().int()`.

### D-067 — Packages gain archive/unarchive (`isArchived`), mirroring Service (FS-4.1.3 / FS-3.1)

FS-3.1's lifecycle table and FS-4.1.3 grant an Archive action to **Service** only; the FS is silent on
package archiving and `Package` has no `isArchived`. **Human ruling (this session):** packages gain the
same archive capability as services.

- **Schema (Schema Agent):** `Package.isArchived Boolean @default(false)`; migration via `prisma migrate
  dev` (never hand-written SQL, CLAUDE.md §11).
- **Endpoints:** `POST /packages/:id/archive` + `/unarchive` under `packages.manage`; list gains
  `includeArchived` (default hides archived), mirroring `listServices`.
- **Read shape:** `isArchived` added to `mapPackage` + `PackageSchema`.
- **Independent of D-066:** archive and delete-block are separate policies — an in-use package is still
  BLOCKED on delete (D-066); archiving is a distinct explicit action and does not bypass the guard.
- **Distinct from `hasArchivedService`** (FS-4.3.2, about *constituent services* being archived):
  `isArchived` is the package's own lifecycle state.

**Sequencing:** D-066 implemented first (no migration); D-067 follows. Per the §11 token checkpoint, D-067
may land in a subsequent session.

### D-068 — Archived packages are blocked from quotation selection (`PACKAGE_ARCHIVED`)

Extends **D-067** (packages gained `isArchived`). FS-4.3.2 blocks selecting a package whose *member
services* are archived (`PACKAGE_CONTAINS_ARCHIVED_SERVICE`) but is silent on selecting a package that is
*itself* archived — a state that did not exist before D-067. FS-4.1 / FS-3.1 establish that archived
*services* are "excluded from new selections"; by analogy and **human ruling (this session):** selecting
an archived package in the quotation builder is blocked with a new `422 PACKAGE_ARCHIVED`.

- **New error code:** `PACKAGE_ARCHIVED` → code SSOT `src/common/errors/error-codes.ts` ONLY, tagged
  `// D-068`, 422, `details { packageId }`. The FS-18 mirror `agents/context/error-codes.json` is
  nuclear-locked (CLAUDE.md §11) and stays FS-only (same stance as D-066).
- **Guard:** `QuotationService.resolveSelectedServiceIds` (the create path — the ONLY place a package is
  selected; `updateQuotation` re-selects nothing) fetches the package when `dto.packageId` is set:
  missing → `NOT_FOUND`; `isArchived` → `PACKAGE_ARCHIVED`. Runs regardless of `totalOverride` (which
  bypasses `computeTotal`), closing the prior hole where an override skipped package validation.
- **Distinct from** `PACKAGE_CONTAINS_ARCHIVED_SERVICE` (member services archived) — this is the
  package's own lifecycle state. No schema change (isArchived exists from D-067).
