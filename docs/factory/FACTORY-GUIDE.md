# Factory Guide — Dalil Agentic Build System

> A complete onboarding reference for contributors. Explains how the pipeline works,
> what every agent and script does, and how to run and recover a build.

---

## 1. Overview

Dalil's backend is built by a **factory**: a supervised multi-agent pipeline that turns a task description + the Functional Specification into tested, committed code. Think of it as an assembly line where each station is a specialized AI agent.

```
  task + FS contract
        │
        ▼
  spec-validator   ←── confirms the behavioral contract (read-only)
        │
        ▼
  schema-agent     ←── writes Prisma schema + migration
        │
        ▼
  test-agent       ←── writes failing (RED) tests from the FS
        │
        ▼
  api-agent        ←── implements the service to make tests pass
        │
        ▼
  quality gate     ←── harness dry-run: typecheck + Jest on a temp branch
        │
        ▼
  review-agent     ←── FS match · test adequacy · conventions · doc drift (read-only)
        │
        ▼
     commit        ←── Husky hooks as final safety net
```

This is **multi-agent orchestration**, not the OOP factory pattern. Each station is a scoped Claude subprocess. They run strictly sequentially in Phase 0 — no parallelism, no shared memory between agents except the output thread that is forwarded from one agent to the next.

The factory lives entirely in `agents/` and `.claude/`. It never ships to production and is never imported by `src/`.

---

## 2. Key Concepts

| Term | Meaning |
|---|---|
| **factory** | The orchestration system: agents, harness, hooks, state. Lives in `agents/` and `.claude/`. Never ships to production. |
| **product** | The Dalil API itself: `src/` and `prisma/`. What the factory builds. |
| **agent** | A scoped Claude subprocess invoked by the harness. Each has a role, a write allowlist, and a turn budget. |
| **spec packet** | The output of spec-validator: FS section, entities, permission key, rules, error codes, edge cases. Every other agent reads this before acting. |
| **knowledge base** | `agents/context/` — machine-readable extracts of the FS and PRD: `fs-index.json`, `prd-index.json`, `error-codes.json`, `terminology.json`, `permission-catalog.json`. These are the SSOT for their domains. Files prefixed `_` (e.g. `_quarantine.json`) are scratch and must be ignored. |
| **harness** | `agents/harness/` — the TypeScript runtime that invokes agents in order, threads output forward, checks signals, manages checkpoints, runs the quality gate, and commits. |
| **blast-radius hook** | A PreToolUse hook (`agents/harness/blast-radius-guard.mjs`) that enforces per-agent write allowlists and a global nuclear-lock denylist. Exits 2 to hard-block violations. Uses the `DALIL_AGENT` env var the harness sets per invocation. |
| **quality gate** | Two stages: (1) the review-agent's FS/test-adequacy check; (2) a harness dry-run (`typecheck + jest`) on a temp `gate/<run-id>` branch before the commit. |
| **`state.json`** | Machine-readable macro state committed to git: phase, module status, ownership, open items, blocked tasks, action log. Written only via the state-module API — never hand-edited (D-017). |
| **run checkpoint** | Per-run step records saved in `agents/harness/runs/<run-id>.json`. Enables resume after a connection drop: already-DONE steps are skipped on re-run. |
| **supervised mode** | The pipeline pauses after each agent for human review. You read the output and type `c` / `r` / `a`. The production way to run builds. |
| **automated mode** | No pauses. Reserved for CI or well-understood rebuilds. Not the default in Phase 0. |
| **run-id** | A stable identifier for a pipeline run (e.g. `rbac-001`). Pass the same run-id to resume from a checkpoint; use a new one for a fresh attempt. |
| **nuclear-locked path** | Paths no agent may ever write, regardless of phase: `.github/**`, `.husky/**`, `.env*`, `.claude/settings.json`, `.claude/hooks/**`, `agents/**`, `prisma/migrations/**`. The hook enforces this. |

---

## 3. The Agents

| Agent | Role | May write | May not write | Definition |
|---|---|---|---|---|
| **spec-validator** | Locate and confirm the FS behavioral contract. Outputs a spec packet. Read-only. | _(nothing)_ | everything | `.claude/agents/spec-validator.md` |
| **schema-agent** | Translate the FS data model into Prisma entities, relations, enums, and migrations. Runs `prisma migrate dev`. | `prisma/**` | `src/`, anything else | `.claude/agents/schema-agent.md` |
| **test-agent** | Write failing (RED) tests from the FS contract, before any implementation. Owns spec files and stories. | `**/*.spec.ts`, `**/*.test.ts`, `test/**`, `**/*.stories.*` | implementation files | `.claude/agents/test-agent.md` |
| **api-agent** | Implement the NestJS module to make the failing tests pass. Definition of done: tests go green without being weakened. | `src/modules/**`, `src/common/**` | other modules, `prisma/` | `.claude/agents/api-agent.md` |
| **review-agent** | Final gate. Validates FS match, test adequacy, conventions, doc drift. Read-only. | _(nothing)_ | everything | `.claude/agents/review-agent.md` |
| **devops-agent** | CI/CD, Husky quality gates, Prisma migrations in CI, environment scaffolding, deployment. | `Dockerfile`, `docker-compose*.yml`, `.devcontainer/**`, `render.yaml`, `package.json`, `.env.example` | application logic | `.claude/agents/devops-agent.md` |
| **frontend-agent** | Next.js screens, components, Storybook stories. Phase 1+ only — Phase 0 is backend-only. | `src/**`, `app/**`, `components/**` | backend, prisma | `.claude/agents/frontend-agent.md` |

Agents run sequentially. Output from each agent is threaded into the next agent's prompt. Agents never communicate with each other directly — all coordination goes through the harness.

---

## 4. How to Run a Build

### Pre-flight (run once before any pipeline attempt)

```bash
pnpm verify:state      # state.json is valid and not in a broken mid-run state
pnpm verify:hook       # blast-radius hook loads and blocks a known bad write
pnpm verify:pipeline   # pipeline types compile cleanly
pnpm verify:supervised # supervised pause/resume logic is intact
```

All four should exit 0. If any fail, fix the issue before starting.

### The `task:supervised` command

```bash
pnpm task:supervised \
  --task "Build <X> per FS-<section>. Backend only." \
  --module <name> \
  --run-id <name>-001 \
  --timeout-ms 900000
```

**Flags:**

| Flag | Required | Default | Meaning |
|---|---|---|---|
| `--task` | yes | — | Inline task description, or a path to a `.txt` / `.md` file whose contents are used. |
| `--module` | yes | — | The product module being built (e.g. `auth`, `rbac`, `quotation`). Controls ownership claiming and state tracking. |
| `--run-id` | no | `run-<timestamp>` | Stable identifier for this run. Reuse the same id to resume from a checkpoint after a drop. Use a new id for a fresh attempt. |
| `--timeout-ms` | no | `600000` (10 min) | Per-agent subprocess timeout. Increase for complex modules. |
| `--supervised` | implicit | always on via `task:supervised` | Already baked into the `task:supervised` script. You do not need to pass it manually. |

The `task:supervised` script is an alias for `tsx agents/harness/run-task.ts --supervised` — the `--supervised` flag is already included.

### The five supervisor pauses

After each agent completes, the terminal shows:

```
----------------------------------------------------------------------
[SUPERVISOR] Step N/5 — <agent-name> OK
Task: <task-id>
Cost this step: $0.XXXX  |  Cumulative: $0.XXXX
Next: <next-agent>
----------------------------------------------------------------------
<agent output>
----------------------------------------------------------------------
[c]ontinue   [r]etry this agent   [a]bort   (default: c) >
```

**What to look for at each pause:**

| Pause | Agent | "Good" looks like |
|---|---|---|
| 1 | spec-validator | A clear spec packet: FS section cited, entities named, permission key, error codes from the catalogue, edge cases. No ⚠ HUMAN DECISION signal. |
| 2 | schema-agent | Prisma model added or updated, `prisma migrate dev` ran without errors, Prisma client regenerated, types compile. |
| 3 | test-agent | Tests written, all RED (fail because the service doesn't exist yet — not because of compile errors). Cases cover the FS edge cases and all listed error codes. |
| 4 | api-agent | Tests now GREEN. Implementation matches the spec packet. No tests weakened or deleted. Flat error shape with codes from the catalogue. |
| 5 | review-agent | `VERDICT: PASS`. FS contract ✓, test adequacy ✓, conventions ✓, no doc drift or drift flagged for your review. |

**Your choices at each pause:**

- **`c` (continue)** — output looks good, advance to the next agent.
- **`r` (retry)** — re-run this agent (uses the same session; up to 3 retries before auto-abort).
- **`a` (abort)** — stop the run immediately. The checkpoint is saved; you can investigate and re-run with the same `--run-id` to resume from where you stopped.

If the agent emits a `⚠ HUMAN DECISION` or `⚠ DOC DRIFT` signal, the harness halts automatically — you do not need to abort manually. The task is recorded in `state.json` under `blockedOn`.

---

## 5. The Scripts

| Script | Command | What it does | When to use | Cost |
|---|---|---|---|---|
| `build` | `tsc` | TypeScript compile (no emit check) | Before shipping; in CI | Free |
| `typecheck` | `tsc --noEmit` | Type-check without output files | Quick type check during dev | Free |
| `lint` | _(not yet configured)_ | Placeholder | N/A | Free |
| `test` | `jest --config jest.config.js` | Run the full Jest suite | After any code change | Free |
| `verify:state` | `tsx agents/harness/state/verify-state.ts` | Zod-validate `state.json`; check for broken mid-run state | Before every pipeline run | Free |
| `verify:hook` | `node agents/harness/audit/verify-hook.mjs` | Load the blast-radius hook and assert it blocks a known bad write | Before every pipeline run; after any hook change | Free |
| `verify:pipeline` | `tsx agents/harness/pipeline/verify-pipeline.ts` | Compile-check the pipeline driver and its types | Before every pipeline run | Free |
| `verify:supervised` | `tsx agents/harness/pipeline/verify-supervised.ts` | Verify the supervisor pause/resume logic end-to-end (uses a mock runner) | Before every pipeline run | Free |
| `verify:runner` | `tsx agents/harness/runner/verify-runner.ts` | Smoke-test the live Claude runner with a real API call | Only when debugging the runner itself | **~$0.10 real API call** |
| `task:supervised` | `tsx agents/harness/run-task.ts --supervised` | Run the full supervised pipeline for a module | Building a feature | API cost per run |

---

## 6. The File Map

Paths below are relative to `dalil-api/` (the backend repo root) unless shown
under the monorepo root block at the end.

```
dalil-api/
│
├── THE FACTORY (never ships to production)
│   ├── CLAUDE.md                   Orchestration brain: rules, pipeline, agents, uncertainty triggers
│   ├── CHANGELOG.md                Operational log of completed work
│   │
│   ├── .claude/
│   │   ├── agents/                 Agent definitions (one .md per agent)
│   │   │   ├── spec-validator.md
│   │   │   ├── schema-agent.md
│   │   │   ├── test-agent.md
│   │   │   ├── api-agent.md
│   │   │   ├── review-agent.md
│   │   │   ├── devops-agent.md
│   │   │   └── frontend-agent.md
│   │   ├── rules/
│   │   │   ├── CONVENTIONS.md      Naming, error shape, money, dates, module structure
│   │   │   └── WORKFLOW.md         Handoff protocol and the worked example
│   │   ├── hooks/
│   │   │   ├── blast-radius-guard.mjs    PreToolUse hook — enforces write allowlists
│   │   │   └── blast-radius.config.json  Per-agent allowlists + nuclear-lock denylist
│   │   └── settings.json           Claude Code settings (nuclear-locked; do not edit)
│   │
│   ├── agents/
│   │   ├── state.json              Macro state: phases, modules, ownership, action log (Zod-validated, committed)
│   │   ├── context/                Knowledge base (FS/PRD extracts)
│   │   │   ├── fs-index.json       FS section index (machine-readable)
│   │   │   ├── prd-index.json      PRD section index
│   │   │   ├── error-codes.json    All error codes (SSOT)
│   │   │   ├── terminology.json    Canonical terms + neverUse list
│   │   │   ├── permission-catalog.json   All permission keys
│   │   │   ├── fs/                 FS section markdown mirrors (human-readable)
│   │   │   └── prd/                PRD section markdown mirrors
│   │   └── harness/                The TypeScript runtime
│   │       ├── run-task.ts         CLI entry point (parse flags → runTask)
│   │       ├── pipeline/
│   │       │   ├── driver.ts       Pipeline orchestration: 5-agent sequence, gate, commit
│   │       │   ├── supervisor.ts   Human pause/resume (ReadlineSupervisor)
│   │       │   ├── signals.ts      Detect ⚠ HUMAN DECISION / ⚠ DOC DRIFT in agent output
│   │       │   ├── breaker.ts      Cost/token circuit breaker
│   │       │   └── run-checkpoint.ts   Per-run step persistence (enables resume)
│   │       ├── runner/             Agent subprocess runner (invokes claude -p)
│   │       ├── gate/               Quality gate: typecheck + Jest dry-run
│   │       ├── git/                Committer (stages files, creates commit)
│   │       ├── state/              state.json read/write API + Zod schema
│   │       └── audit/              verify-hook.mjs
│   │
│   └── _ai-context/
│       └── modules/                Per-module session logs (human-readable reasoning history)
│
├── THE PRODUCT (what ships)
│   ├── src/
│   │   ├── modules/                NestJS feature modules (one dir per domain)
│   │   └── common/                 Shared guards, filters, error codes, Prisma middleware
│   └── prisma/
│       ├── schema.prisma           DB source of truth
│       └── migrations/             Applied migration SQL (never hand-written)
│
├── package.json                    Scripts + dependencies
├── jest.config.js                  Jest configuration
└── tsconfig.json                   TypeScript configuration

../                                 THE MONOREPO ROOT (shared, outside both repos)
├── docs/
│   ├── product/                    PRD v1.6 + Functional Specification v2.0 (the source of truth)
│   ├── contracts/                  frontend-contract.md · backend-contract.md
│   ├── decisions/
│   │   ├── api.md                  Every human ruling during the backend build; append-only
│   │   └── web.md                  Frontend rulings (FE-D-xxx)
│   └── factory/
│       └── FACTORY-GUIDE.md        This file
└── dalil-web/                      The frontend repo (separate git history)
```

---

## 7. Recovery & Troubleshooting

### Connection drop mid-run

The harness saves a checkpoint after each agent step (in `agents/harness/runs/<run-id>.json`). Re-run with the **same `--run-id`** and already-DONE steps are skipped automatically:

```bash
pnpm task:supervised \
  --task "same task as before" \
  --module <name> \
  --run-id <same-run-id> \
  --timeout-ms 900000
```

### Module stuck "CLAIMED" / ownership conflict

If a prior run died without releasing the module, `state.json` still shows it as owned. Fix via the state-module API — never hand-edit `state.json` (D-017):

```bash
npx tsx -e "
import { releaseModule } from './agents/harness/state/state';
releaseModule('<module-name>', '<run-id-that-owns-it>');
console.log('released');
"
```

Then verify: `pnpm verify:state`.

### `state.json` looks wrong

Always verify first:

```bash
pnpm verify:state
```

If validation fails, write a small repair script (like `agents/harness/repair-rbac-002.ts`) that calls the state-module API to correct the values. Never open `state.json` and type in it directly.

### An agent overreaches (writes outside its allowlist)

The blast-radius hook fires and exits 2 — the write is hard-blocked. The pipeline surfaces this as an error. Check `verify:hook` passes:

```bash
pnpm verify:hook
```

If the hook itself is broken, do not bypass it. Fix the hook and re-verify.

### A spec conflict or ambiguous FS section

The agent emits `⚠ HUMAN DECISION` and the harness halts with `status: HALTED_HUMAN`. The task is recorded under `blockedOn` in `state.json`. Write the resolution to `../decisions/api.md`, then re-run with a new `--run-id`.

### Review FAIL on second attempt (GATE_TWICE)

If the review-agent FAILs twice on the same module, the harness halts and escalates. Read both review outputs carefully. The issue is usually a missing edge case in tests or an incorrect FS interpretation. Fix, then re-run fresh.

### Cost breaker triggers

The harness tracks cumulative cost per session. If it exceeds the configured limit, it halts with `status: HALTED_BREAKER`. Check the run summary for actual vs limit values. To continue, start a new session with a new `--run-id`.

---

## 8. How to Contribute / Extend

### Adding a new module build

1. Pick a task description that references a specific FS section.
2. Run pre-flight: `pnpm verify:state && pnpm verify:hook && pnpm verify:pipeline && pnpm verify:supervised`.
3. Launch: `pnpm task:supervised --task "..." --module <new-module> --run-id <new-module>-001 --timeout-ms 900000`.
4. Review each of the five pauses carefully. Read the spec packet (pause 1) before approving anything else.
5. After commit, update `_ai-context/modules/<module>.session-log.md` with what was built and any deviations.

### The spec is the source of truth (D-012)

If a FS section is missing or unclear, the fix is to update the spec and re-index — not to patch the code with a workaround. When the spec changes:
1. Update the relevant section in `agents/context/fs/`.
2. Re-index (update `agents/context/fs-index.json`).
3. Rebuild the affected module.

### Where decisions get recorded

Every human ruling goes in `../decisions/api.md` as a numbered entry before work resumes. Agents reference decisions by number (e.g. D-017). The file is append-only — supersede old decisions with new numbered entries, never rewrite history.

### Agent write scope is enforced, not trusted

Each agent runs under `--permission-mode bypassPermissions` (D-016) to avoid interactive prompts in unattended subprocesses. The blast-radius hook is the real safety boundary. Any change to an agent's allowlist requires updating `agents/harness/blast-radius.config.json` and running `pnpm verify:hook`.

---

## 9. Where Things Stand

### Factory stages (all complete)

The factory itself was built in six stages before any product code:

| Stage | What was built |
|---|---|
| 1 | Repo scaffold, tsconfig, Prisma, Jest |
| 2 | Knowledge base (`agents/context/`) indexed from FS/PRD |
| 3 | Agent definitions (`.claude/agents/*.md`), CONVENTIONS.md, WORKFLOW.md |
| 4 | `state.json` schema + state-module API |
| 5 | Blast-radius PreToolUse hook + `bypassPermissions` runner (D-014, D-016) |
| 6 | Live pipeline: driver, supervisor, gate, committer, `task:supervised` |

### Product phases

| Phase | What ships | Status |
|---|---|---|
| 0 | Core backend: auth, RBAC, tenants, catalog, quotations, projects, billing | **In progress** — RBAC complete |
| 1+ | Frontend (Next.js), client portal, analytics, advanced billing | Not started |

**Phase 0 module status (from `state.json`):**

- `rbac` — COMPLETED (effective-permissions resolution, admin protection invariants — FS-2.4.1/2.4.2, D-018)

See [`../decisions/api.md`](../decisions/api.md) for the full decision history (D-001 through D-018 at time of writing).

---

## 10. Glossary

| Term | Definition |
|---|---|
| **factory** | The entire orchestration system (`agents/`, `.claude/`) that builds the product. Never ships. |
| **product** | The Dalil API (`src/`, `prisma/`). What the factory produces. |
| **agent** | A scoped Claude subprocess invoked by the harness for one job (spec, schema, tests, implementation, review). |
| **spec packet** | The structured output of the spec-validator: FS section, entities, permission key, validation rules, error codes, edge cases. The contract all other agents work from. |
| **knowledge base** | `agents/context/` — machine-readable FS/PRD extracts. SSOT for error codes, terminology, permission keys, and FS/PRD sections. |
| **harness** | `agents/harness/` — the TypeScript runtime that drives the pipeline: invokes agents, threads output, manages checkpoints, runs the gate, commits. |
| **hook** | `agents/harness/blast-radius-guard.mjs` — a PreToolUse hook that fires before every agent file/bash operation and exits 2 to block out-of-allowlist writes. |
| **gate** | Two-stage quality check: (1) review-agent FS + test-adequacy check; (2) harness dry-run (typecheck + Jest) on a temp branch. Both must pass before commit. |
| **blast radius** | The set of paths an agent is allowed to write. Defined per-agent in `blast-radius.config.json`. Violations are hard-blocked by the hook. |
| **supervised mode** | Pipeline pauses after each agent for human review. Type `c` / `r` / `a` at each pause. |
| **run-id** | A stable identifier for a pipeline run. Same id = resume from checkpoint. New id = fresh attempt. |
| **nuclear-locked path** | Paths no agent may ever write: `.github/**`, `.husky/**`, `.env*`, `.claude/settings.json`, `.claude/hooks/**`, `agents/**`, `prisma/migrations/**`. |
