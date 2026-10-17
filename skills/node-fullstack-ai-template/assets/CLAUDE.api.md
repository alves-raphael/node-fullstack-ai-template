# apps/api

Fastify + TypeScript (strict) + Zod + Drizzle/Postgres + Better Auth. Global rules, commands and the environment runbook are in the root `CLAUDE.md`.

## Contract first
`openapi.json` (this folder) is the machine-readable description of every endpoint: params, bodies, responses per status code, and `x-source-file` pointing at the implementing file. **Read it to understand the API instead of walking the route code.** It is generated: run `yarn api:openapi` after any route or schema change. `yarn verify` fails if it is stale.

## Layout
`src/modules/<feature>/{routes.ts,schemas.ts,service.ts,<feature>.test.ts}`. `src/app.ts` builds the app (testable with `app.inject`), `src/server.ts` listens, `src/env.ts` validates env with Zod. Copy `modules/me` as the reference pattern (auth, schemas, all response codes, tests); `modules/health` is the minimal one.

## Adding an endpoint
Strongly recommended: use the `new-endpoint` skill (`/new-endpoint`), which holds the validated steps and keeps `openapi.json` complete. In short: Zod schemas -> route with `operationId`, `tags`, `summary`, and a `response` entry for every status code returned -> service for logic -> Vitest test -> `yarn api:openapi` -> `yarn verify`.

## Rules
- Validate everything with Zod schemas on the route; never trust `request.body` untyped.
- Errors use the single shape `{ error: { code, message, details? } }` (schema in `packages/shared`). Throw typed errors; the central error handler maps them.
- Logs: use `request.log` / `app.log` (Pino, structured). No `console.log`.
- Database: schema in `src/db/schema.ts`. `yarn db:generate` then `yarn db:migrate`. Never edit an applied migration.
- Auth: use the `requireAuth` preHandler for protected routes. Better Auth is mounted at `/api/auth/*`.
- Tests: Vitest with `app.inject()`; cover success, validation failure and auth failure for each route.
