# Tooling: TypeScript, lint, tests, hooks, CI

## TypeScript strict

Pin `typescript@~6.0` (`yarn add -D typescript@~6.0`). The unpinned latest (7.x) is rejected by `typescript-eslint` (`Error: typescript-eslint does not support TS 7.0`; its peer range is `<6.1.0`). Revisit the pin when typescript-eslint supports 7. In TS 6, `baseUrl` is gone: use `paths` with `./`-relative targets (`"@/*": ["./src/*"]`).

`tsconfig.base.json` at the root: `"strict": true`, plus `noUncheckedIndexedAccess`, `noImplicitOverride`, `exactOptionalPropertyTypes` (drop the last one only if a library makes it unworkable, and say so in the README), `"module": "ESNext"`, `"moduleResolution": "Bundler"` (or NodeNext for the api), `"skipLibCheck": true`. Each workspace `extends` it and must not relax `strict`. The same rule is written in the root `CLAUDE.md`.

`yarn typecheck` runs `tsc --noEmit` in each workspace. List `typescript` in every workspace's own `devDependencies` (and `vitest`, `eslint` bins likewise where a workspace script calls them): Yarn Berry only exposes a workspace's own dependency binaries, so `yarn workspace <name> typecheck` otherwise fails with `command not found: tsc` even though root `yarn typecheck` works (observed, it confuses agents that run a single workspace).

## ESLint (flat config) + Prettier

One `eslint.config.js` at the root with blocks per glob (`apps/web/**`, `apps/api/**`, `**/*.test.ts`, `apps/web/e2e/**`).

- Base: `typescript-eslint` with the type-checked preset (needed for `no-floating-promises`), `eslint-config-prettier` (last, to disable conflicting style rules), `eslint-plugin-import-x`, `eslint-plugin-unused-imports`.
- Web: `eslint-plugin-react-hooks`, `eslint-plugin-react-refresh`, `eslint-plugin-jsx-a11y`.
- Tests: `@vitest/eslint-plugin` for unit tests, `eslint-plugin-playwright` for e2e.
- Required rules: `@typescript-eslint/no-explicit-any: error`, `@typescript-eslint/no-floating-promises: error`, `eqeqeq: error`.

Prettier config at the root (`.prettierrc`), `yarn format` writes, `yarn lint` runs ESLint and `prettier --check`.

Config details that failed on a real run: apply `tseslint.configs.disableTypeChecked` to `**/*.{js,mjs}` through `extends: [...]` (spreading it into the block overwrites `languageOptions`); set `@typescript-eslint/require-await: off` (Fastify plugins/handlers are async by contract) and `no-unused-vars` with `argsIgnorePattern: '^_'`; keep `eslint.config.js` in the ignore of type-aware rules via that same JS block. `yarn lint` = `eslint . && prettier --check .`; run `eslint . --fix` and `prettier --write .` once on the generated code before the first verify.

Plugin APIs change between majors; after installing, run `yarn lint` on the generated code and fix config errors rather than trusting remembered config shapes.

## Vitest

Neither Vitest nor Playwright reads the root `.env`. In `apps/api/vitest.config.ts` and `apps/web/playwright.config.ts` call `process.loadEnvFile(<root>/.env)` inside try/catch (CI and containers provide variables directly). Without it the API tests fail with `Invalid environment variables` and Playwright's global setup reports the environment as down.

Unit and integration tests for api and web (`*.test.ts`, `*.test.tsx`). Api tests use Fastify's `app.inject()` against a real test database or a transaction-scoped one; keep at least one DB-backed test in the reference module so the pattern exists. Root `yarn test` runs all workspaces.

## Hooks and commits

- `husky` + `lint-staged`: `.husky/pre-commit` runs `yarn lint-staged`; lint-staged runs `eslint --fix` and `prettier --write` on staged files. `prepare` script installs husky on `yarn install`.
- Commits follow Conventional Commits (`feat(api): ...`, `fix(web): ...`, `chore: ...`). This is documented in the root `CLAUDE.md`; there is no commitlint.

## `.nvmrc` / `.editorconfig`

`.nvmrc` with the current Node LTS and the same range in `engines`. `.editorconfig`: utf-8, LF, 2 spaces, final newline.

## CI (.github/workflows/ci.yml)

Job services must mirror `.env.example`: Postgres mapped to `DB_PORT`, plus a Redis service mapped to `REDIS_PORT` when Redis was chosen (without it `/health` returns 500 and the e2e wait never finishes). Validation done for the generated workflow: `actionlint` clean (run it with `docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:latest`; `act` is usually absent) and a local simulation of the job steps (standalone Postgres/Redis containers, fresh `.env`, `yarn install --immutable`, migrate, seed, verify, `dev:logged`, `wait-on`, e2e) that passed. Not reproducible locally: `playwright install --with-deps` (needs apt on the runner), so it is unverified. On push and PR: checkout, `cp .env.example .env`, setup-node (from `.nvmrc`, with Yarn cache), `corepack enable`, `yarn install --immutable`, `yarn verify`, then e2e: Postgres as a `services:` container, run migrations and seed, start api and web in the background, wait for `/health`, `yarn playwright install --with-deps`, `yarn e2e`. Upload the Playwright report on failure.
