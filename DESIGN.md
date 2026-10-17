# Design notes

Why the template looks the way it does. This is for people maintaining or adapting the skill. Commands and implementation details are in `skills/node-fullstack-ai-template/references/`, which is the source of truth for the agent; this file only records decisions and reasons.

## Goal

A blank project from which a coding agent can go from "implement spec X, run it and test it" to a validated result without guessing. Every decision below serves that: legible context, a verifiable contract, an environment that starts with one command, and checks that tell the agent when it is done.

## Decisions

**Layered `CLAUDE.md` (root, api, web).** Each file holds only what is specific to its location, so the agent loads the minimum relevant context. Global rules (strict TypeScript, Playwright for user flows, Conventional Commits, pitfalls) live in the root.

**OpenAPI committed at `apps/api/openapi.json`.** It sits in the api workspace because it describes the api, and it is committed so an agent can read the whole contract (params, bodies, every status code, auth) without walking route code. It is generated, never edited, and `yarn verify` fails when stale. It deliberately carries no source file paths: they would be hand-maintained and go stale, and a unique `operationId` already lets an agent find the code with one search.

**Zod as the single source of truth.** Validation, TypeScript types and the OpenAPI document come from the same schemas, so they cannot drift.

**`apps/*` + `packages/*` layout.** Executables are separated from shared code, and new apps (worker, admin) can be added without reorganizing.

**One verification command, `yarn verify`.** The agent does not have to guess what "checks pass" means. CI runs the same command.

**Strict everywhere.** TypeScript `strict`, `no-explicit-any`, `no-floating-promises`. Rules are not to be relaxed to make a check pass. Agents tend to take the shortest path to green; the lint config removes the shortcut.

**Idempotent `yarn setup` and read-only `yarn status`.** The agent checks before starting anything and never starts a second instance. Waiting is done on healthchecks, not `sleep`. A fresh clone needs `yarn install` first because Yarn Berry cannot run scripts before its first install.

**Docker by default, hybrid mode optional.** `docker compose up` brings up everything with migrations and seed done. The user's own terminal cannot be read by an agent, but container logs can (`docker compose logs`), which is why Docker is the default. Hybrid mode (apps on the host) writes logs to `.logs/dev.log` through `yarn dev:logged` for the same reason.

**Playwright runs against the already-running environment.** No `webServer` block: starting another instance would collide with ports and the user's session. If nothing is up, the test fails telling the agent to run `yarn setup`. Login uses a seeded user and `storageState`.

**Specs in `docs/specs/` with a template, and `/implement-spec`.** Acceptance criteria are written so each becomes a test (Vitest for API/business rules, Playwright for flows), and the agent reports criterion by criterion what was validated.

**Native `--env-file`, no `dotenv`.** Node loads `.env` natively and Vite handles its own. `dotenv` is only acceptable inside `drizzle.config.ts` if `drizzle-kit` cannot take `--env-file`. Vitest and Playwright do not read `.env`, so they call `process.loadEnvFile`.

**ESLint + Prettier, not Biome.** Chosen for the plugin ecosystem the template depends on: type-checked rules, react-hooks, jsx-a11y, Vitest and Playwright plugins.

**husky + lint-staged, Conventional Commits, no commitlint.** Quality is enforced automatically on staged files; commit message format is a documented convention, not a gate, to avoid friction.

**TypeScript pinned to `~6.0`.** `typescript-eslint` did not support TS 7 when validated. Revisit when it does.

**Generated shadcn code is vendored.** `components/ui` is excluded from ESLint and Prettier.

## Deliberately left out

- **Turborepo:** Yarn workspaces cover the essentials for two apps and one package. Adding it later costs a `turbo.json` and changing root scripts. Reconsider with long CI times or several more workspaces.
- **commitlint:** see above.
- **`dotenv`:** see above.
- **Redis usage code:** Redis is optional and off by default; only the client and the health check are scaffolded, nothing that uses it.

## Known gaps

- The GitHub Actions workflow has not run on a real runner.
- Setup and status scripts were tested on Linux only.
- The shadcn CLI path was exercised non-interactively once; the first run of a new shadcn version may need adjustment.
