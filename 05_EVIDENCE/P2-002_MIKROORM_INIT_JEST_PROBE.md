# P2-002 — MikroORM.init isolation via native Jest transform

Date: 2026-09-05

## Frozen references

- Mercato implementation: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Accepted P2-001 base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Scanner: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

## Canonical Testing provenance

The approved Mercato environment was loaded from `/etc/mercato-localhost.env`.
`DATABASE_URL` was neither printed nor changed. Sanitized provenance was:

```json
{"host":"aws-1-eu-central-1.pooler.supabase.com","port":"5432","scheme":"postgresql","usernameHasProjectRef":true}
```

The username includes the approved Supabase Testing project reference
`yzonugcenguvmojwiihb`.

## Temporary probe

One untracked Jest test was created under the Mercato Jest `rootDir`, executed with
the normal `apps/mercato/jest.config.cjs` transformer, then deleted. It imported
exactly the accepted P2-001 `MikroORM`, PostgreSQL driver, and nine-entity set. Its
initialization options were the P2-001 options: unchanged `DATABASE_URL`,
`PostgreSqlDriver`, the exact entity list, disabled empty-entity warning, and TLS
with `rejectUnauthorized: false`.

The sole command was bounded by one 45-second outer timeout and used `--runInBand`.

## Literal result

```text
2026-09-05T10:02:48.248Z IMPORTS_COMPLETE
2026-09-05T10:02:48.249Z BEFORE_MIKROORM_INIT
2026-09-05T10:02:48.334Z AFTER_MIKROORM_INIT
2026-09-05T10:02:48.334Z BEFORE_ORM_SELECT1
2026-09-05T10:02:48.464Z AFTER_ORM_SELECT1
2026-09-05T10:02:48.466Z AFTER_ORM_CLOSE
PASS src/modules/wms_outbound/services/__tests__/p2-002-mikroorm-init-jest-probe.test.ts
Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Time:        1.899 s
```

The ORM `select 1 as ok, current_database() as db` returned `ok=1` and
`db=postgres`. Numeric exit status: `0`.

## Classification

`MIKROORM_INIT_PASS` — native Jest transformation, exact P2-001 ORM initialization,
and a post-init ORM query completed successfully against the approved Supabase
Testing project. No P2-001 regression test was run in this shot.

## Cleanup

The temporary diagnostic test was deleted. Mercato remains clean at
`24d102a41dcfff669cf6b839b369fbaa3620f87d`; Scanner remains clean at
`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`. No probe process remained.
