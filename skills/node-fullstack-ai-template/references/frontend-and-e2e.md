# Front end (apps/web) and e2e

## Web

- Vite + React + TypeScript (strict) + shadcn/ui (Tailwind v4 via `@tailwindcss/vite`). Path alias `@/` to `src/` in both `tsconfig.json` (`"./src/*"`, no `baseUrl`) and `vite.config.ts`. shadcn CLI works non-interactively (verified with shadcn 4.21): run it from `apps/web` **inside the Yarn workspace** (outside it the CLI falls back to npm and dies on `workspace:*`): `npx --yes shadcn@latest init -t vite -b radix -p nova -y --no-monorepo --cwd . < /dev/null`. Without `-p <preset>` it stops at an interactive preset menu. It detects Vite, Tailwind v4 and the `@/` alias, writes `components.json`, `src/lib/utils.ts`, `src/components/ui/button.tsx`, edits `src/index.css`, and installs dependencies with Yarn. Components: `npx shadcn@latest add input card -y`. Generated code passes strict `tsc`, but not the lint setup: it breaks `import-x/order`, `react-refresh/only-export-components` and Prettier. Treat `src/components/ui` as vendored: add `'apps/web/src/components/ui/**'` to the ESLint `ignores` and the folder to `.prettierignore`, and run `prettier --write` once on `src/index.css` and `src/lib/utils.ts` after init. Testing Library needs `@testing-library/dom` installed next to `@testing-library/react`.
- Data access in one place (`src/lib/api.ts`): a typed fetch wrapper using `VITE_API_URL`, `credentials: 'include'`. Optional but recommended: types generated from `apps/api/openapi.json` with `openapi-typescript` (script `yarn web:api-types`), so the front consumes the same contract the agent reads.
- Better Auth client (`better-auth/react`) for sign-in/session.
- Reference page: a login page and a protected page that shows the current user from `GET /me`. Components carry accessible labels/roles (they also serve as stable Playwright locators).
- Vitest + Testing Library for at least one component test.

## Playwright (apps/web/e2e)

- `playwright.config.ts`: load the root `.env` first (`process.loadEnvFile`), then `baseURL` from `WEB_URL` (default `http://localhost:5173`), no `webServer` block. The tests run against the environment that is already up. A `globalSetup` pings `${WEB_URL}` and the API `/health`; if unreachable it throws: "Environment is not running. Run `yarn setup` (or `yarn status` to inspect)." Starting a second instance would collide with ports and the user's terminal.
- Auth: a `setup` project logs in once with the seed user (`SEED_USER_EMAIL` / `SEED_USER_PASSWORD`) and saves `storageState`; other projects reuse it. Also keep one test that exercises the login form itself, unauthenticated.
- Smoke test: log in, reach the protected page, see the user's email.
- Prefer role/label locators; no `waitForTimeout`. `eslint-plugin-playwright` enforces this.
- `yarn e2e` runs everything; `yarn e2e <file>` runs a single flow. Browsers are installed by `yarn setup` (`playwright install`).
- Rule documented in root and web `CLAUDE.md`: any change to a user flow must be validated with Playwright (create or update the test and run it before calling the task done).
