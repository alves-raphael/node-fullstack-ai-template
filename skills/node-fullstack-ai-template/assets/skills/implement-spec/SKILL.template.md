---
name: implement-spec
description: Implements a feature from a spec in docs/specs, runs the project and validates every acceptance criterion with tests (Vitest for API and business rules, Playwright for user flows). Use when the user asks to implement, build or develop a spec or feature described in docs/specs, for example "implement the XPTO spec, run it and test it", or invokes /implement-spec <spec-name>.
argument-hint: <spec-name>
---

# Implement a spec

Spec to implement: `docs/specs/$ARGUMENTS.md` (if no name was given, ask which spec in `docs/specs/`).

This flow exists so that "done" always means the same thing: every acceptance criterion has a passing test and the environment was actually exercised. Skipping steps is what produces work that compiles but was never run. Adapt a step when the situation really calls for it, and say so in the final report.

1. Read the spec and list its acceptance criteria. If something is ambiguous or missing, ask before coding.
2. Run `yarn status`. If the environment is not healthy, run `yarn setup` (on a fresh clone, `yarn install && yarn setup`). Never start a second instance.
3. Implement following the reference modules (`apps/api/src/modules/me`, the web reference page) and the `new-endpoint` skill for API work. Respect the rules in the root `CLAUDE.md`.
4. Write tests from the acceptance criteria: Vitest for API and business rules, Playwright for user flows.
5. Run `yarn api:openapi` if the API changed, then `yarn format` (so new files do not fail the Prettier check), then `yarn verify`, then `yarn e2e` for the affected flows. If anything fails, read the logs (`docker compose logs --since 5m api`, or `.logs/dev.log`) before changing code.
6. Report criterion by criterion: what was validated, by which test, and anything you could not validate and why.
