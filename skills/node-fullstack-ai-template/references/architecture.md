# Architecture, scripts, env and Docker

## Layout

```
/
├─ package.json            workspaces + root scripts
├─ .yarnrc.yml             nodeLinker: node-modules
├─ docker-compose.yml
├─ README.md  CLAUDE.md
├─ .env.example (versioned)  .env (local, gitignored)
├─ .nvmrc  .editorconfig  .gitignore
├─ .mcp.json
├─ .claude/                settings.json, skills/{implement-spec,new-endpoint}/, agents/ (optional)
├─ .github/workflows/ci.yml
├─ .husky/
├─ docs/specs/             _template.md + one file per feature
├─ scripts/                setup.mjs, status.mjs, dev-logged.mjs
├─ packages/shared/        Zod schemas + types used by api and web
└─ apps/
   ├─ api/   src/{server.ts,app.ts,env.ts,db/,modules/<feature>/{routes,schemas,service,test}}  CLAUDE.md
   └─ web/   src/, e2e/, playwright.config.ts  CLAUDE.md
```

Use `apps/*` and `packages/*` as workspaces. Yarn Berry setup, in this order (corepack's default is Yarn 1, which has no `-W` and cannot `set version` without a manifest): write a minimal root `package.json` first, run `corepack enable` then `yarn set version stable` (this adds `packageManager`), then write `.yarnrc.yml` yourself with `nodeLinker: node-modules` (the command does not create it). Set `"type": "module"` in the root `package.json` (ESLint config and scripts are ESM). In Berry, add root dev dependencies with plain `yarn add -D ...` and workspace ones with `yarn workspace <name> add ...`; quote `pkg@workspace:*` specs so zsh does not glob them. Name packages `@<scope>/api`, `@<scope>/web`, `@<scope>/shared`. `shared` is consumed via workspace dependency and must build or be consumed from source in a way both Vite and the API understand (simplest: export TS source and let each consumer's bundler/tsx compile it).

## Root scripts (package.json)

`dev`, `dev:logged`, `build`, `lint`, `format`, `typecheck`, `test`, `e2e`, `db:generate`, `db:migrate`, `db:seed`, `api:openapi`, `setup`, `status`, `verify`, plus `prepare` (husky).

- `verify` = `typecheck` + `lint` + `test` + `api:openapi --check` (fails if `apps/api/openapi.json` is stale). One command so an agent never has to guess what "checks pass" means. CI runs the same.
- `dev:logged` = run `yarn dev` with output duplicated to `.logs/dev.log` (`scripts/dev-logged.mjs` or `yarn dev 2>&1 | tee .logs/dev.log`). `.logs/` is gitignored. Exists so an agent can read logs of a process running in the user's own terminal.

## Environment variables

- `.env.example` is versioned, lists every variable with a safe dev value and a comment. `.env` is gitignored.
- Load with Node's native support: `node --env-file=.env`, `tsx --env-file=.env`. Vite loads `.env` by itself; only `VITE_*` reaches the browser. This is why `dotenv` is not a dependency. Add it only if a tool (e.g. `drizzle-kit`) refuses `--env-file`, and note why in the README.
- Scripts run from workspace folders must point at the root file (`--env-file=../../.env`).
- Also put `COMPOSE_PROJECT_NAME=<project>` in `.env.example`, so container/volume names are explicit and a second checkout can override them.
- Never assume the default ports are free: before choosing ports check `ss -ltn`; the user's own stacks often hold 5173/5432.
- Minimum variables: `NODE_ENV`, `API_PORT`, `WEB_PORT`, `WEB_URL`, `API_URL`, `DATABASE_URL`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `SEED_USER_EMAIL`, `SEED_USER_PASSWORD`, `VITE_API_URL`, and `REDIS_URL` when Redis is included.
- `apps/api/src/env.ts` parses `process.env` with Zod at boot and fails listing every missing or invalid variable. No other file reads `process.env`.

## Docker Compose

Services: `db` (postgres), `redis` (only if chosen; omit the service entirely otherwise), `migrate`, `api`, `web`.

When Redis is chosen (verified end to end): add `redis:7-alpine` with `healthcheck: redis-cli ping` and `ports: '${REDIS_PORT:-6379}:6379'`; add `REDIS_PORT` and `REDIS_URL=redis://localhost:<REDIS_PORT>` to `.env.example`; set `REDIS_URL: redis://redis:6379` in the shared `x-app` environment block (like the database host); make `api` `depends_on` redis `service_healthy`; add a `redis` row to the URLs table in `scripts/lib.mjs`, README and root `CLAUDE.md`. Details of the code side are in `references/backend.md`.

- One shared dev image (`Dockerfile.dev`, `node:24-alpine`) used by `migrate`, `api` and `web`. Only `migrate` has the `build:` section; all three set the same `image:` tag and `pull_policy: never` on the other two. Declaring `build:` on all three builds the image three times in parallel (registry sockets get dropped and disk usage balloons). Do not set `pull_policy: build` (it rebuilds on every `up`); `yarn setup` runs `docker compose build migrate` explicitly, and a plain `docker compose up` still builds the image if it is missing.
- The image only contains dependencies: copy the four `package.json` files + `yarn.lock` + `.yarnrc.yml`, then `yarn workspaces focus <api> <web>` (skips root-only tooling such as ESLint, which keeps the image smaller). Set `ENV YARN_NETWORK_CONCURRENCY=4 YARN_HTTP_RETRY=5` in the Dockerfile; Yarn's default parallelism fails with `ERR_SOCKET_CLOSED_BEFORE_CONNECTION` inside Docker. Everything else (source, `tsconfig*.json`, `vite.config.ts`, `drizzle.config.ts`, `index.html`, `drizzle/` migrations, `packages/shared/src`) is bind-mounted per service, so editing config never requires a rebuild. `tsx` must be an `apps/api` dependency (not root-only) because the container runs it.
- Container commands call binaries through `yarn workspace <name> exec ...` and do not use the package scripts that pass `--env-file` (the container has no `.env` file; values come from `env_file:`).
- `db`: named volume, `healthcheck: pg_isready`.
- `migrate`: one-shot, runs migrations then the idempotent seed; `depends_on: db: condition: service_healthy`.
- `api`: `depends_on: migrate: condition: service_completed_successfully`; healthcheck hits `GET /health` using `http://127.0.0.1:<port>` (on Alpine `localhost` resolves to `::1` first while Fastify listens on IPv4, so the check fails and the container is reported unhealthy). Source mounted as a volume with watch mode for hot reload; keep `node_modules` in a named volume so the host's do not leak in.
- `web`: Vite dev server, healthcheck on `http://127.0.0.1:<port>/`. The port comes from the root `.env` through `vite.config.ts` (`loadEnv(mode, envDir, '')` -> `server: { host: true, port: Number(env.WEB_PORT), strictPort: true }`), never hardcoded in a script, so host mode, Docker and Playwright agree.
- Ports come from `.env` (`${API_PORT:-3000}:3000`).
- Inside containers the API reaches Postgres at host `db`, so `DATABASE_URL` differs between container and host mode. Provide both (`DATABASE_URL` for host, override in the compose `environment:` block).

Result: `docker compose up` yields a ready database and running apps with no manual steps.

## `yarn setup` (scripts/setup.mjs), idempotent

**Bootstrap caveat (verified):** on a fresh clone `yarn setup` fails with `Couldn't find the node_modules state file`, because Yarn Berry refuses to run any script before the first install. So the documented entry point for a fresh clone is `yarn install && yarn setup` (or `node scripts/setup.mjs`, which runs the install itself). Say this in the README and in the root `CLAUDE.md` runbook; after the first install `yarn setup` works normally.

1. Create `.env` from `.env.example` if missing (never overwrite).
2. `yarn install`.
3. `docker compose build migrate`, `docker compose up -d` and wait on healthchecks (poll `docker compose ps` health or `/health`; never `sleep`). Print the failing service's logs on timeout.
4. Migrations and seed (done by the `migrate` service; in hybrid mode run `db:migrate` and `db:seed`).
5. `yarn playwright install chromium` run inside `apps/web` (where `@playwright/test` is installed) for the browsers.
6. Print the URLs table (same as `yarn status`).

Safe to run repeatedly; each step must be a no-op when already done. Put shared helpers (dotenv parsing, `ping`, `composeServices`, `printUrls`) in `scripts/lib.mjs`; scripts are plain ESM `.mjs` so they run before any dependency exists. While waiting, also detect a failed `migrate` service (exited non-zero) and print its logs instead of waiting for the timeout.

## `yarn status` (scripts/status.mjs)

Prints containers and health (`docker compose ps`), whether the ports are listening, the result of `GET /health`, and the URLs table. Exit code 0 only if everything is healthy, so an agent can branch on it. Do not start anything.

## Ports and URLs table

| Service | URL | Logs |
|---|---|---|
| web | `http://localhost:5173` | `docker compose logs web` |
| api | `http://localhost:3000` | `docker compose logs api` |
| OpenAPI docs | `http://localhost:3000/docs` | |
| Postgres | `localhost:5432` | `docker compose logs db` |
| Redis | `localhost:6379` | `docker compose logs redis` |

Put this table in the root `CLAUDE.md` and README, with the real ports used.

## README

Short: what the project is, "worth mentioning" notes (stack choices, why OpenAPI is committed, spec workflow), requirements (Node version, Docker, corepack), how to run (`yarn setup`, URLs, seed user), main scripts.
