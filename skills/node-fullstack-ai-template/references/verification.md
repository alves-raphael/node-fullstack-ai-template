# Verification: the template is done only when this passes

Before starting, check free disk (`df -h /`; the Docker image plus `node_modules` need about 2 GB) and that the chosen ports are free (`ss -ltn`). Never stop or reuse the user's existing containers; use a unique `COMPOSE_PROJECT_NAME`.

Run in a scratch copy or the fresh target directory. Stop at the first failure, fix the template (and these references if the cause was a wrong instruction), and rerun from the top.

1. **Clean install**: remove every `node_modules` (`find . -name node_modules -prune -exec rm -rf {} +`; a zsh glob that matches nothing aborts the whole `rm`), then `yarn install --immutable` succeeds; husky hooks are installed.
2. **Setup from zero**: no `.env`, no containers, then `yarn install && yarn setup` (plain `yarn setup` before the first install fails, see architecture) completes with no manual step and prints the URLs table. Run it a second time: it must succeed and change nothing (idempotent).
3. **Status**: `yarn status` exits 0 and shows every service healthy; `curl $API_URL/health` returns ok.
4. **Env validation**: temporarily remove a required variable and confirm the api fails at boot naming that variable; restore it.
5. **Database**: migrations applied, seed user exists; running `yarn db:seed` again does not duplicate.
6. **Verify**: `yarn verify` passes (typecheck, ESLint + Prettier, Vitest, `openapi.json` up to date). Change a route's schema without regenerating and confirm `api:openapi --check` fails, then regenerate.
7. **OpenAPI content**: `apps/api/openapi.json` lists `/health` and `/me` with operationIds, all response codes, bodies and `x-source-file`.
8. **E2E**: `yarn e2e` passes logged in as the seed user. With the environment stopped it fails with the "run `yarn setup`" message instead of hanging.
9. **Logs**: `docker compose logs api` shows structured request logs. In hybrid mode, `yarn dev:logged` writes `.logs/dev.log`.
10. **Pre-commit**: stage a file with a lint error and confirm the commit is blocked.
11. **Redis variant** (only if chosen): `redis` is healthy, `/health` ok, 500 with Redis stopped and ok after starting it again; `yarn verify` passes.
12. **Agent files**: root, api and web `CLAUDE.md`, `.claude/settings.json`, `.claude/commands/*`, `.mcp.json`, `docs/specs/_template.md` exist with no `{{placeholder}}` left (`grep -r '{{' .`).

## Frequent breakages to look for

- `Couldn't find the node_modules state file` on a fresh clone: run `yarn install` first.
- `command not found: tsc` when running one workspace's script: `typescript` missing from that workspace's devDependencies.
- shadcn output failing ESLint/Prettier: ignore `src/components/ui`.
- With Redis chosen: `/health` 500 and CI never ready when the CI job has no Redis service.
- `typescript-eslint does not support TS 7.0` (pin `typescript@~6.0`).
- Healthcheck failing on Alpine because it used `localhost` (use `127.0.0.1`).
- `ERR_SOCKET_CLOSED_BEFORE_CONNECTION` during `yarn install` in Docker (throttle with `YARN_NETWORK_CONCURRENCY`; build the shared image once).
- Vitest/Playwright not finding env variables (they do not read `.env`).
- Better Auth CLI failing with `Invalid environment variables` (load `.env` first).

- Container cannot reach Postgres: wrong host (`db` inside Docker, `localhost` on the host).
- `--env-file` path wrong when a script runs from a workspace folder.
- `node_modules` from the host shadowing the container's through a bind mount.
- Zod / `fastify-type-provider-zod` version mismatch causing schema transform errors in swagger.
- Better Auth cookies not sent: CORS origin and `credentials` mismatch between web and api.
- Playwright browsers missing on a fresh machine.
- ESLint type-checked rules failing because `parserOptions.projectService`/`project` does not cover test and config files.

Report to the user which checks passed and which could not be run (for example no Docker available).
