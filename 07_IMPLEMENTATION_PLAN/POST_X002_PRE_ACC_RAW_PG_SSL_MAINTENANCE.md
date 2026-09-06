# Post-X-002 pre-ACC maintenance gate — raw PostgreSQL test client SSL

**Status:** planned / mandatory before `ACC-001`  
**Position:** execute **after X-002** and **before the first full acceptance item (`ACC-001`)**.  
**Catalog impact:** none — this is a test-infrastructure maintenance gate, not a new Task Catalog item and does not change the 37-item count.

## Situation

During the X-001 acceptance-correction rerun, four existing concurrency suites that use raw `pg.Client` observer connections failed before their business assertions with:

`self-signed certificate in certificate chain`

The affected suites were:

- `p1-011-postgres.integration.test.ts`
- `p1-014-erp-posting-postgres.integration.test.ts`
- `p1-015-manifest-lifecycle-postgres.integration.test.ts`
- `p1-016-final-settlement-postgres.integration.test.ts`

The failure was reproduced independently of the X-001 product changes. Current Testing `DATABASE_URL` contains `sslmode=require`; with the currently installed `pg@8.22.0` / `pg-connection-string@2.14.0`, parsing that connection string can override the explicitly supplied `ssl: { rejectUnauthorized: false }` on a raw `pg.Client`, effectively turning the observer connection back into certificate verification and causing the self-signed-chain failure.

MikroORM-backed application/test connections were not affected because they use their own driver SSL configuration.

## Immediate X-001 correction already applied

To unblock the mandatory final-head X-001 dedicated-suite reruns, the four known raw `pg.Client` call sites were corrected locally in their tests by removing `sslmode` from the connection string before passing the explicit test SSL option.

That correction is verified on Mercato `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43`; the X-001 final dedicated set passed **148/148**.

This fixes the known current call sites, but duplicated hand-written raw-client construction remains a maintenance risk: a future concurrency test can accidentally reintroduce the old pattern.

## Mandatory maintenance after X-002

Before `ACC-001` starts:

1. Create one canonical **test-only** helper/factory for raw PostgreSQL observer clients used by WMS Outbound concurrency tests.
2. The helper must consume the existing runtime-injected Testing `DATABASE_URL`; never hard-code, print or persist credentials.
3. Centralize the SSL normalization currently duplicated in the four X-001-corrected suites so `sslmode` cannot clobber the explicit approved Testing SSL option.
4. Replace the existing hand-written raw `pg.Client` observer construction in the affected Outbound suites with the helper.
5. Search the WMS Outbound test tree for any other raw `pg.Client` construction and route equivalent Testing observer connections through the same helper where applicable. Do not touch unrelated application DB configuration.
6. Add a focused test/proof for the helper against the canonical Testing PostgreSQL path sufficient to prove it connects with the approved environment without exposing secrets.
7. Rerun only the directly affected raw-client concurrency suites plus typecheck; do **not** perform the full Outbound acceptance sweep here. The full automated acceptance starts immediately afterward under `ACC-001`.

## Boundary

- Test infrastructure only; no business behavior change.
- No credential rotation/refactor; Testing credential handling remains frozen.
- No local PostgreSQL.
- No Demo/Prod.
- Do not convert this gate into a new product requirement or Task Catalog item.
- Completion of X-002 does **not** authorize skipping this gate or starting `ACC-001` before it is complete and supervisor-verified.

## Required sequence

`X-002 -> raw pg SSL maintenance gate -> ACC-001 -> ACC-002 -> ACC-003 -> ACC-004`
