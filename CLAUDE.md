# CLAUDE.md — Dalil monorepo root

This directory is the **workspace root**, not an application and not a git repo. It
owns the shared documents and the scripts that run both apps. The two applications
underneath are independent git repositories, each with its own governing brain.

## Precedence

| Working in | Governing document |
| --- | --- |
| `dalil-api/**` | `dalil-api/CLAUDE.md` — the backend factory brain (agents, harness, blast radius) |
| `dalil-web/**` | `dalil-web/CLAUDE.md` — the frontend brain (slice pipeline, quality gates) |
| `docs/**` or root config | this file |

An app's own `CLAUDE.md` wins inside that app. Nothing here relaxes a rule there.

## Layout

```
dalil/
├── docs/          Shared: product specs, contracts, decision logs, fable-5 launch pack
├── dalil-api/     Backend — NestJS · Prisma · PostgreSQL (own .git)
└── dalil-web/     Frontend — Next.js · React · Tailwind (own .git)
```

Start from [`docs/README.md`](docs/README.md) for the document index.

## Rules that hold across both repos

- **The docs tree is shared and load-bearing.** Both repos reference it as
  `../docs/...`, resolved from that repo's root. Moving or renaming anything under
  `docs/` means updating those references in the same change.
- **Decision logs are append-only.** `docs/decisions/api.md` (`D-xxx`, backend and
  global) and `docs/decisions/web.md` (`FE-D-xxx`, frontend). Supersede a ruling
  with a new numbered entry; never rewrite a past one. Global `D-xxx` rulings bind
  the frontend where applicable.
- **The HTTP contract is the only coupling.** `dalil-web` never imports backend
  code. The API's OpenAPI spec at `/api-docs-json` is authoritative for shapes;
  regenerate the typed client with `pnpm api:types` after every backend change.
- **The apps stay separately versioned.** Do not merge the two git histories, add a
  root `pnpm-workspace.yaml`, or hoist dependencies — each app installs and
  deploys on its own. Root scripts delegate with `pnpm --dir`.
- **Spec conflicts are a hard stop.** When PRD v1.6 and FS v2.0 disagree, or either
  is silent on needed behavior, escalate to Khaled and record the ruling — never
  resolve it by silently picking an interpretation.

## Running things

`pnpm dev` runs both apps; `pnpm dev:api` / `pnpm dev:web` run one. `build`, `test`,
`lint` and `typecheck` follow the same pattern. See [`README.md`](README.md).
