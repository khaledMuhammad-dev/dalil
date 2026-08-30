# Dalil — Monorepo

Multi-tenant SaaS for creative agencies. Two applications, one workspace.

| Path | App | Stack | Dev port |
| --- | --- | --- | --- |
| [`dalil-api/`](dalil-api/) | Backend API | NestJS 11 · Prisma 7 · PostgreSQL | `8080` |
| [`dalil-web/`](dalil-web/) | Frontend | Next.js 16 · React 19 · Tailwind 4 | `3000` |

All product, contract and decision documents live in [`docs/`](docs/) — see the
[docs index](docs/README.md).

---

## Getting started

```bash
pnpm install          # root orchestration tooling only
pnpm install:all      # dependencies for both apps
pnpm dev              # runs the API and the web app together
```

`pnpm dev` starts both apps side by side with prefixed, colour-coded output
(`api` cyan, `web` magenta). To run one on its own use `pnpm dev:api` or
`pnpm dev:web`.

### Environment

Each app owns its own environment file — the root has none.

- `dalil-api/.env` — `DATABASE_URL`, `JWT_PRIVATE_KEY`, `JWT_PUBLIC_KEY`, optional `PORT` (default `8080`) and `CORS_ORIGIN`.
- `dalil-web/.env.local` — copy from `dalil-web/.env.example`. `NEXT_PUBLIC_API_URL` must point at the API **server root** (`http://localhost:8080`); the generated OpenAPI paths already carry `/api/v1`.

---

## Scripts

Every root script delegates into an app with `pnpm --dir`; the apps keep their
own lockfiles and `node_modules`, so anything you can run inside an app you can
also run from here.

| Script | What it does |
| --- | --- |
| `pnpm dev` | Both apps in watch mode (see the API caveat below) |
| `pnpm build` | `build:api` then `build:web` |
| `pnpm start` | Both apps from their production builds |
| `pnpm test` | Jest suites for both apps |
| `pnpm test:e2e` | Playwright suite (`dalil-web`) |
| `pnpm lint` / `pnpm typecheck` | ESLint / `tsc --noEmit` across both apps |
| `pnpm storybook` | Storybook for `dalil-web` on port `6006` |
| `pnpm api:types` | Regenerate the web app's typed API client from the running API's `/api-docs-json` |
| `pnpm clean` | Remove `dalil-api/dist` and `dalil-web/.next` |

Append `:api` or `:web` to `dev`, `build`, `test`, `lint` or `typecheck` to
target a single app.

> **API watch caveat.** `dev:api` compiles and then runs the server; it does not
> hot-reload on source changes (a pre-existing limitation of `dalil-api` — Nest
> needs `emitDecoratorMetadata`, which the repo's `tsx` loader does not emit).
> Restart `dev:api` after backend edits, or run `pnpm --dir dalil-api build`
> in a second terminal.

---

## Repository layout

The two apps remain **separate git repositories** with independent histories and
independent deploys; this root directory is the shared workspace and document
home, not a git repo of its own.

```
dalil/
├── docs/          Product specs, API contracts, decision logs  ← shared
├── dalil-api/     Backend repo (own .git)
└── dalil-web/     Frontend repo (own .git)
```

The only coupling between the apps is the HTTP contract: the API publishes its
OpenAPI spec at `/api-docs-json`, and the web app generates its typed client
from it (`pnpm api:types`). Never import backend code from the frontend.
