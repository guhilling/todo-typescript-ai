# RDS IAM authentication from Node.js

Status: proposed. Carried over from TaskFest, where the Quarkus backend uses the same mechanism.
The data access library is decided in [data-access-and-migrations.md](data-access-and-migrations.md).

## Finding

IAM authentication to RDS works from Node.js and needs no infrastructure change beyond what
TaskFest already has. The token is signed locally from the task role's credentials (no call to
AWS) and is used as the password. TaskFest's Java counterpart is
`RdsUtilities.generateAuthenticationToken` in `RdsIamCredentialsProvider.java`.

## Decision

Connect through one `pg` `Pool` whose `password` is a function returning a fresh token, and hand
that pool to Prisma's `pg` driver adapter:

```ts
import { Signer } from '@aws-sdk/rds-signer'
import { PrismaPg } from '@prisma/adapter-pg'
import { readFileSync } from 'node:fs'
import { Pool } from 'pg'
import { PrismaClient } from './generated/prisma/client' // output path is set in schema.prisma

const signer = new Signer({ hostname: DB_HOST, port: 5432, username: DB_USER, region: 'eu-central-1' })

const pool = new Pool({
  host: DB_HOST,
  port: 5432,
  user: DB_USER,
  database: DB_NAME,
  // pg accepts a function: each new connection gets a fresh token (a token is valid 15 minutes,
  // but is only checked at connect, so open connections outlive it).
  password: () => signer.getAuthToken(),
  // IAM authentication requires TLS; verify against the RDS CA bundle.
  ssl: { ca: readFileSync('global-bundle.pem', 'utf8'), rejectUnauthorized: true },
})

export const prisma = new PrismaClient({ adapter: new PrismaPg(pool) })
```

## Consequences

- **AWS side, same as TaskFest:** the task role needs `rds-db:connect` on
  `arn:aws:rds-db:<region>:<account>:dbuser:<DbiResourceId>/<user>`, and the database user has
  `GRANT rds_iam TO <user>`. Nothing is specific to Java.
- **CA bundle:** `https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`, baked into
  the image.
- **Use a pool, not one connection per request:** AWS recommends IAM authentication for fewer
  than about 200 new connections a second.
- **The adapter does not close the pool by default.** `PrismaPg` takes `disposeExternalPool`,
  default `false`; set it to `true` if `prisma.$disconnect()` should also end the pool.
- **Migrations cannot use the pool.** See
  [data-access-and-migrations.md](data-access-and-migrations.md).

## What was verified

- Checked (2026-10-13) in the type declarations of `@prisma/adapter-pg` 7.10.0: the `PrismaPg`
  constructor takes `pg.Pool | pg.PoolConfig | string`, so an existing pool with a password
  function is accepted.
- **Not yet run:** the snippet end to end against RDS, and the generated client's import path.
