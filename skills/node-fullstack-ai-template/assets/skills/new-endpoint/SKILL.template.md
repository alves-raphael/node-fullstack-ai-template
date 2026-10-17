---
name: new-endpoint
description: Adds an API endpoint to apps/api following the project's contract-first pattern (Zod schemas, OpenAPI metadata for every status code, service, Vitest test, regenerated openapi.json). Use when the user asks to add, create or change an API endpoint or route, or invokes /new-endpoint <feature> <METHOD /route>.
argument-hint: <feature> <METHOD /route>
---

# Add an API endpoint

Endpoint to create: $ARGUMENTS

Following this pattern keeps `openapi.json` accurate and complete, which is what other agents read to understand the API. Adapt a step when needed and mention it in your summary.

1. Read `apps/api/CLAUDE.md` and `apps/api/src/modules/me` as the reference.
2. In `apps/api/src/modules/<feature>/`: write the Zod schemas (`schemas.ts`), the route (`routes.ts`) with `operationId`, `tags`, `summary`, and a `response` entry for every status code it can return, using the shared error schema for non-2xx; put logic in `service.ts`; register the module in `src/app.ts`.
3. Protect it with `requireAuth` unless the spec says it is public.
4. If the database changes: edit `src/db/schema.ts`, then `yarn db:generate` and `yarn db:migrate`. Never edit an applied migration.
5. Write the Vitest test (`app.inject`): success, validation failure, auth failure.
6. Run `yarn api:openapi` and check the new operation appears with its unique `operationId` and a response for every status code.
7. Run `yarn verify`.
