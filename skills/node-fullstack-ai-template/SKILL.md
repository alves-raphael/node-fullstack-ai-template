---
name: node-fullstack-ai-template
license: MIT
description: Scaffolds a blank full stack Node.js monorepo (Yarn workspaces, Fastify + Zod + Drizzle + Better Auth API with a generated OpenAPI contract, Vite/React/shadcn web, Playwright, Vitest, Docker Compose) designed for AI-assisted development, with layered CLAUDE.md files, a one-command environment setup and a spec-driven workflow. Use this skill whenever the user wants to start a new Node.js/TypeScript full stack project, a monorepo with api and web, a project skeleton or boilerplate "ready for Claude/AI coding agents", or mentions Fastify, Drizzle, Better Auth, Vite plus a backend together, even if they do not say "template" or "scaffold".
---

# Node.js full stack template for AI-assisted development

Generate a blank but fully working monorepo whose main goal is to let a coding agent go from "implement spec X, run it and test it" to a validated result with no guessing. Everything below exists to serve that: context is layered in `CLAUDE.md` files, the API contract is machine-readable (OpenAPI), the environment comes up with one command, and every acceptance criterion maps to a test.

## Inputs

Ask only what is missing; use the default otherwise.

| Input | Default |
|---|---|
| Project name (kebab-case, also the package scope, e.g. `@name/api`) | required |
| Target directory | `./<project-name>` |
| Include Redis | no |
| Ports (web / api / db / redis) | 5173 / 3000 / 5432 / 6379 |

The default mode is "everything in Docker". The hybrid mode (db in Docker, api/web on the host) is supported through `yarn dev:logged`.

## Workflow

1. **Check prerequisites**: Node (version in the template `.nvmrc`), Docker with Compose, `corepack`, roughly 2 GB of free disk, and that the chosen ports are free (`ss -ltn`; pick others if the defaults are taken and write them into `.env.example`). If something is missing, tell the user instead of working around it. Never touch the user's running containers.
2. **Read the references you need, in this order** (they hold the details; this file only orchestrates):
   - `references/architecture.md`: layout, root scripts, env handling, Docker Compose, `setup`/`status`.
   - `references/tooling.md`: strict TypeScript, ESLint/Prettier, Vitest, husky/lint-staged, CI.
   - `references/backend.md`: Fastify, Zod, OpenAPI, Drizzle, Better Auth, reference module.
   - `references/frontend-and-e2e.md`: Vite/React/shadcn and Playwright.
3. **Create the structure and files.** Copy the static files from `assets/` and substitute placeholders (`{{PROJECT_NAME}}`, `{{SCOPE}}`, ports). Write the code files following the references. Install dependencies with the package manager (`yarn add`) rather than hardcoding versions from memory, so the project gets current, mutually compatible versions. The markdown assets are not Prettier-formatted when copied (tables get re-aligned), and `yarn verify` runs `prettier --check`, so run `yarn format` once after copying everything and before the first `yarn verify`.
4. **Verify against `references/verification.md`.** This is not optional: the template is only done when a clean run of `yarn setup`, `yarn verify` and the e2e smoke test passes. Fix what breaks. Generated projects that were never executed are the main failure mode of scaffolding skills.
5. **Report** to the user: what was created, how to start (`yarn install && yarn setup`), the URLs table, and anything that could not be verified.

## Assets (copy, then fill placeholders)

- The three layered context files. Root holds global rules; api/web hold only what is specific to them. The `.root`/`.api`/`.web` suffixes only tell the templates apart here; in the generated project every file must be named exactly `CLAUDE.md`, because that is the only name Claude Code loads:
  - `assets/CLAUDE.root.md` -> `CLAUDE.md`
  - `assets/CLAUDE.api.md` -> `apps/api/CLAUDE.md`
  - `assets/CLAUDE.web.md` -> `apps/web/CLAUDE.md`
- `assets/spec-template.md` -> `docs/specs/_template.md`.
- `assets/skills/implement-spec/SKILL.template.md` -> `.claude/skills/implement-spec/SKILL.md`, and `assets/skills/new-endpoint/SKILL.template.md` -> `.claude/skills/new-endpoint/SKILL.md` (note the rename: the `.template` suffix keeps tools such as `gh skill` from treating these assets as installable skills of this repository). They are project skills: they travel with the generated repo (nobody installs them separately), the model can trigger them from natural language, and they can also be invoked as `/implement-spec <name>` and `/new-endpoint ...`.
- `assets/claude-settings.json` -> `.claude/settings.json` (pre-approved safe commands).
- `assets/mcp.json` -> `.mcp.json` (Playwright MCP).

## Principles that guide judgment calls

- **The agent is the user of this template.** When a choice is unclear, pick what makes the project more legible and verifiable to an agent: explicit over implicit, one place per concern, errors that say what to do next.
- **Zod is the single source of truth** for validation, types and the OpenAPI document. Never let the three drift.
- **Strict everywhere.** TypeScript `strict`, no `any`, no floating promises. Do not relax rules to make a check pass; fix the cause.
- **No second instance.** Setup and status scripts detect a running environment instead of starting another one.
- **Do not add what is not in the spec** (no Turborepo, no `dotenv`, no commitlint). Mention it in the report if the user asks; they can add it later.
