# node-fullstack-ai-template

An agent skill that scaffolds a blank full stack Node.js monorepo built for AI-assisted development: layered `CLAUDE.md` files, a machine-readable OpenAPI contract, one-command environment setup and a spec-driven workflow (`/implement-spec`).

The generated project includes:

- Yarn workspaces: `apps/api` (Fastify, Zod, Drizzle, Better Auth), `apps/web` (Vite, React, shadcn/ui), `packages/shared`
- `apps/api/openapi.json` generated from Zod schemas, with the source file of each endpoint
- Strict TypeScript, ESLint + Prettier, Vitest, Playwright, husky + lint-staged, Conventional Commits
- Docker Compose for the whole stack, `yarn setup` / `yarn status` / `yarn verify`
- CI (GitHub Actions) and a Playwright MCP config

## Install

With GitHub CLI (preview feature):

```sh
gh skill install <owner>/node-fullstack-ai-template node-fullstack-ai-template
```

Or copy `skills/node-fullstack-ai-template/` into `~/.claude/skills/` (Claude Code user scope) or `<project>/.claude/skills/`.

## Use

Ask your agent for a new Node.js/TypeScript full stack project (for example "create a new fullstack project with Fastify and Vite"). The skill asks for the project name, whether to include Redis and the ports, generates the monorepo and verifies it by running setup, `yarn verify` and the e2e smoke test.

Requirements on the machine: Node (current LTS), `corepack`, Docker with Compose, about 2 GB of free disk.

## Layout of this repository

```
skills/node-fullstack-ai-template/
├─ SKILL.md          entry point and workflow
├─ references/       detailed specs the agent reads on demand
└─ assets/           static files copied into generated projects
DESIGN.md            why the template is the way it is (for maintainers)
```

Operational details live in `references/`. `DESIGN.md` records the decisions and their reasons; read it before changing the template's stack.

## License

MIT, see `LICENSE`.

## Validation status

The template was generated and executed end to end on Linux (setup from zero and idempotent re-run, `yarn verify`, OpenAPI check, e2e with seed login, Redis variant, pre-commit hook, CI steps simulated locally). Not verified: the GitHub Actions workflow on a real runner (`playwright install --with-deps`), and Windows/macOS for the setup scripts.
