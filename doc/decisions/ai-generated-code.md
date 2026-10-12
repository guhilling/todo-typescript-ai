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
  read-only plan with the exact flags (global rule).

## Consequences

- More up-front setup (strict TypeScript, linter, test harness) before features.
- Review effort goes into the domain model, migrations and tests rather than into line-by-line
  reading of boilerplate.
