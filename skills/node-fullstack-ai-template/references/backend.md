# Back end (apps/api)

Stack: Fastify, TypeScript strict, Zod, Drizzle + PostgreSQL, Better Auth, `@fastify/swagger`, Pino (built into Fastify), Redis only if chosen.

## Structure by feature

`src/modules/<feature>/{routes.ts,schemas.ts,service.ts,<feature>.test.ts}`. `src/app.ts` builds and returns the Fastify instance (no listening, so tests can use `inject`); `src/server.ts` only reads env and listens. `src/env.ts` is the only reader of `process.env`. Predictable folders and a unique `operationId` per route make the code of any endpoint one search away (`grep -r <operationId> apps/api/src`), which is why the OpenAPI file does not carry source paths.

## Zod as the single source of truth

Use `fastify-type-provider-zod` (set `validatorCompiler`, `serializerCompiler`, and `jsonSchemaTransform` for swagger). Check the installed versions of `zod` and the provider are compatible before writing code. Request/response schemas live in `modules/<feature>/schemas.ts`; schemas shared with the web (error shape, DTOs) live in `packages/shared`.

## OpenAPI contract

- `@fastify/swagger` with the Zod transform, plus `@fastify/swagger-ui` at `/docs` for humans.
- Every route declares: `operationId`, `tags`, `summary`/`description`, `params`/`querystring`/`body`, and a `response` entry per status code it can return (success and errors), each with a body schema.
- `operationId` is mandatory, unique and descriptive (`listTasks`, `createTask`); it is the key an agent greps to find the implementing code. Do not add source file paths to the document: they are hand-maintained, go stale on renames, and a search finds the route faster than they would. Hide the Better Auth catch-all route from the document with `schema: { hide: true }`.
- `yarn api:openapi` boots the app (without listening), calls `app.swagger()` and writes `apps/api/openapi.json`, pretty-printed and with stable key order so diffs are meaningful. `yarn api:openapi --check` exits non-zero if the file on disk differs from the generated one; `yarn verify` and CI use it.
- The file is committed. Agents read it instead of walking the route code; they must never edit it by hand.

## Errors and logs

One error shape for every non-2xx response, declared as a Zod schema in `packages/shared` and reflected in OpenAPI: `{ error: { code: string, message: string, details?: unknown } }`. A single `setErrorHandler` maps validation errors, auth errors and unexpected errors to it. Use Fastify's Pino logger with structured fields (request id, route, status); no `console.log`.

## Database (Drizzle)

- `src/db/schema.ts`, `src/db/client.ts`, `drizzle.config.ts`, migrations in `apps/api/drizzle/`.
- `yarn db:generate` creates a migration from schema changes, `yarn db:migrate` applies. Applied migrations are never edited; create a new one.
- `db:generate` can be `node --env-file=../../.env ../../node_modules/drizzle-kit/bin.cjs generate` (verified to work; `yarn drizzle-kit` is an equivalent). `drizzle.config.ts` reads `process.env.DATABASE_URL`.
- If `drizzle-kit` does not accept `--env-file`, load the root `.env` inside `drizzle.config.ts` (this is the one place `dotenv` is acceptable).

## Auth (Better Auth)

- Drizzle adapter for Postgres (`drizzleAdapter(db, { provider: 'pg', schema })`), email + password enabled. `src/auth.ts` imports the env module, so the Better Auth CLI needs the variables loaded: `cd apps/api && set -a && . ../../.env && set +a && npx @better-auth/cli@latest generate --config src/auth.ts --output src/db/schema.ts -y` (start from an empty `schema.ts`, otherwise it reports missing tables). Then `yarn db:generate`.
- Generate the auth tables with the Better Auth CLI so schema and migrations stay in the normal Drizzle flow.
- Mount Better Auth's handler under `/api/auth/*` in Fastify. Verify against the current Better Auth docs for the Fastify integration (cookie handling, CORS with `credentials: true` for the web origin).
- A small `requireAuth` preHandler that resolves the session and adds the user to the request.

## Seed

`yarn db:seed` is idempotent: creates the test user from `SEED_USER_EMAIL` / `SEED_USER_PASSWORD` (through Better Auth's API so the password is hashed correctly) only if it does not exist. The Playwright setup logs in with it.

## Reference module (copy-me)

Ship these so an agent copies a proven pattern instead of inventing one:

- `health`: `GET /health` (route option `logLevel: 'warn'` so the healthcheck polling does not flood the request logs agents read) checks the database (`select 1`), returns `{ status: 'ok' }`. Used by compose healthchecks, `yarn status`, CI.
- `me`: `GET /me` behind `requireAuth`, returns the current user. Full OpenAPI metadata, all response codes (200, 401), Vitest tests (authenticated and not).

Each has routes, schemas, service where logic exists, tests, and appears correctly in `openapi.json`.

## Redis (optional)

Only when the user chose it (verified end to end with `ioredis`):
- `REDIS_URL: z.string().url().optional()` in `env.ts`; `src/lib/redis.ts` exports `redis = env.REDIS_URL ? new Redis(env.REDIS_URL, { maxRetriesPerRequest: 2 }) : null` (`import { Redis } from 'ioredis'`).
- `/health` does `if (redis) await redis.ping()` after the database check. Observed behaviour: with Redis stopped, `/health` returns 500 and the API container turns unhealthy; with it back, ok again. That is intended, say so in the API `CLAUDE.md`.
- The `health` routes plugin registers `app.addHook('onClose', async () => { await redis?.quit(); })`, otherwise Vitest hangs on the open connection.
- The CI job needs a `redis` service too (see `references/tooling.md`), since `.env.example` carries `REDIS_URL`.
- Do not create code that uses Redis beyond the client.
