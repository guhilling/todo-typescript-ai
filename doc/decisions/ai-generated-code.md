# Working with AI-generated code

Status: accepted (Gunnar, 2026-10-13).

## Context

Claude generates most of this project's code. Gunnar is an experienced backend developer (Java,
Quarkus) and reviews all of it, but is newer to Node.js and frontend, so a large generated diff
is harder to judge than in Java. "AI" here is about how the project is built, not a runtime LLM
dependency; no LLM provider is called at runtime.

## Decisions

- **Tests are the specification.** TDD: a failing test first, then the minimal code, then
  refactor. An existing test is never changed, deleted or skipped without asking.
- **A strict, mechanical gate runs on every change:** `tsc --noEmit` with `strict`, Biome and
  Vitest, as one command. The command is recorded in `CLAUDE.md` once a `package.json` exists.
  Generation is fast, so feedback has to be fast and mechanical, not depend on review alone.
- **Explicit over implicit.** Prefer plain functions and visible wiring to decorators and
  containers (see [technology-stack.md](technology-stack.md)).
- **Conventional, widely used libraries.** Newer or fast-moving ones are used only with their
  version-specific API checked against current documentation, not from memory. What was verified
  is recorded in `doc/decisions/`. Prisma 7 is the example: its adapter and migration behaviour
  had to be checked (see [data-access-and-migrations.md](data-access-and-migrations.md)).
- **Generated SQL migrations are reviewed by Gunnar before they are applied.** Destructive ones
  (drops, type changes) are called out explicitly in the pull request description.
- **Small, reviewable pull requests.** Node-specific idioms are explained in the pull request, not
  only in chat.
- **Infrastructure is never generated and applied by Claude.** It is applied by Gunnar, after a
  read-only plan with the exact flags (global rule). This is enforced, not only written down:
  `.claude/settings.json` denies `tofu`/`terraform` apply, destroy, import and state changes,
  and AWS CLI calls with mutating verbs (`create-*`, `delete-*`, `update-*`, `put-*` and so on,
  plus `s3 cp/mv/rm/sync`). Read-only calls (`describe-*`, `list-*`, `get-*`, `tofu plan`) are
  not denied.

## Limits of the enforcement

- The deny rules are pattern matches on the command, not a sandbox. They stop the straightforward
  cases; a determined detour (a script, another tool) is not caught. They guard against mistakes,
  not against intent.
- AWS CLI verbs are matched by prefix. A mutating verb that doesn't start with one of the listed
  prefixes is not denied; extend the list when one turns up.
- They bind Claude Code sessions only. CI and humans are not affected, which is as intended
  (humans apply infrastructure; CI never does, see the global rule).

## Consequences

- More up-front setup (strict TypeScript, linter, test harness) before features.
- Review effort goes into the domain model, migrations and tests rather than into line-by-line
  reading of boilerplate.
