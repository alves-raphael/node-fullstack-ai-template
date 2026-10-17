---
description: Add an API endpoint following the project's contract-first pattern
argument-hint: <feature> <METHOD /route>
---

Create the endpoint described by: $ARGUMENTS

1. Read `apps/api/CLAUDE.md` and `apps/api/src/modules/me` as the reference.
2. In `apps/api/src/modules/<feature>/`: write the Zod schemas (`schemas.ts`), the route (`routes.ts`) with `operationId`, `tags`, `summary`, and a `response` entry for every status code it can return, using the shared error schema for non-2xx; put logic in `service.ts`; register the module in `src/app.ts`.
3. Protect it with `requireAuth` unless the spec says it is public.
4. If the database changes: edit `src/db/schema.ts`, then `yarn db:generate` and `yarn db:migrate`. Never edit an applied migration.
5. Write the Vitest test (`app.inject`): success, validation failure, auth failure.
6. Run `yarn api:openapi` and check the new operation appears with its `x-source-file`.
7. Run `yarn verify`.
