# Data access and schema migrations: Prisma

Status: accepted (Gunnar, 2026-10-13).

## Context

The backend is Node.js/TypeScript on Postgres (RDS, IAM authentication, see
[rds-iam-authentication.md](rds-iam-authentication.md)). The alternatives considered were Drizzle,
Kysely and plain `pg`, with Flyway for migrations.

## Decision

- **Data access:** Prisma Client, connected through `@prisma/adapter-pg` with our own `pg` `Pool`.
- **Migrations:** Prisma Migrate. `schema.prisma` is the source of truth for tables and types;
  Migrate writes SQL migration files that are reviewed and committed.
- **Hand-written SQL** goes through `$queryRaw`, not a second library.

The adapter is not a workaround we accept reluctantly: it is how Prisma 7 connects, and it is
what makes an IAM token usable as a password.

## Consequences

- The schema lives in Prisma's own DSL, and a `prisma generate` step belongs in the build and CI.
- Request validation is still a separate choice (Prisma generates model types, not request
  schemas).
- **Migrations need a token in the URL.** In Prisma 7 `prisma.config.ts` has no `adapter`
  property; Migrate uses `datasource.url`, which is a static string and cannot be computed. An
  IAM token lasts 15 minutes, so a migration run must be given a freshly generated token, for
  example by a small script that signs a token and exports `DATABASE_URL` for that one command.
  Migrations are applied by a human, which fits this.
- The token has to be URL-encoded in the connection string, and the CA bundle passed to Migrate
  (for example `sslmode=verify-full&sslrootcert=...`).

## What was verified

- Checked (2026-10-13) against the Prisma 7 config reference: `datasource.url` is required and
  static; the `adapter` property was removed in v7, and "migrations for driver adapters work
  automatically".
- **Not verified:** the token-in-URL migration flow against RDS, including URL encoding and the
  TLS parameters. Try it before relying on it.
