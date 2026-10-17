---
description: Implement a spec from docs/specs, run the project and validate it with tests
argument-hint: <spec-name>
---

Implement the spec `docs/specs/$ARGUMENTS.md` end to end.

1. Read the spec and list its acceptance criteria. If something is ambiguous or missing, ask before coding.
2. Run `yarn status`. If the environment is not healthy, run `yarn setup`. Never start a second instance.
3. Implement following the reference modules (`apps/api/src/modules/me`, the web reference page) and `/new-endpoint` for API work. Respect the rules in the root `CLAUDE.md`.
4. Write tests from the acceptance criteria: Vitest for API/business rules, Playwright for user flows.
5. Run `yarn api:openapi` if the API changed, then `yarn verify`, then `yarn e2e` for the affected flows. If anything fails, read the logs (`docker compose logs --since 5m api`, or `.logs/dev.log`) before changing code.
6. Report criterion by criterion: what was validated, by which test, and anything you could not validate and why.
