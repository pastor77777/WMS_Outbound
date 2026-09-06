# Post-X-002 raw PostgreSQL SSL maintenance gate — Executor Evidence

**Status:** Executor COMPLETE. Not Supervisor FINAL PASS, not Owner Accepted.

## Scope confirmation

Test infrastructure only. No business/application DB configuration, credential
handling, Scanner, Playwright, runtime rebuild, or ACC-001 work was performed.

## Revisions

- Mercato base: `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`.
- Mercato branch: `outbound/post-x002-raw-pg-ssl` @ `4d6033b900958b2717d4c4c16c357acf7b7b5687` (pushed to `origin`).
- Scanner: unchanged, frozen at `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302` (verified identical before and after).

## Helper

`apps/mercato/src/modules/wms_outbound/services/__tests__/support/pg-observer-client.ts`

Contract:

- `getTestingDatabaseUrl()` — returns runtime `process.env.DATABASE_URL`, throws `DATABASE_URL is required for Genuine PostgreSQL acceptance tests.` if absent. Never hard-codes credentials.
- `createPgObserverClient(connectionString)` — strips `sslmode=...` from the caller-supplied connection string via regex (matching the pattern already used at the majority of known-good X-001/X-002 call sites), then constructs `new pg.Client({ connectionString, ssl: { rejectUnauthorized: false } })`. Never prints/logs the connection string.
- Callers pass their own connection string (already possibly port-swapped for pooler routing); the helper never rewrites host/port, preserving each suite's existing direct-vs-pooler semantics.
- Test-only: lives under `__tests__/support/`, not imported by any application/runtime source.

## Pre-change search (complete WMS Outbound test tree)

Searched `apps/mercato/src/modules/wms_outbound` for `new Client(`, `import { Client } from 'pg'`, and manual `sslmode` handling.

Found raw `pg.Client` observer construction with equivalent hand-written SSL/`sslmode` normalization in **14 files** (5 known from the maintenance guide + 9 additional discovered by the full-tree search):

Known (from guide):
1. `p1-011-postgres.integration.test.ts`
2. `p1-014-erp-posting-postgres.integration.test.ts`
3. `p1-015-manifest-lifecycle-postgres.integration.test.ts`
4. `p1-016-final-settlement-postgres.integration.test.ts`
5. `p2-004-crossdock-recovery-postgres.integration.test.ts`

Additional, discovered by this gate's required full-tree search:
6. `p1-006-postgres.integration.test.ts` (4 call sites)
7. `p1-007-postgres.integration.test.ts` (local `testingObserverClient()` helper, 9 call sites)
8. `p1-008-postgres.integration.test.ts` (3 call sites; did not strip `sslmode` — pre-existing latent risk)
9. `p1-009-postgres.integration.test.ts`
10. `p1-010-postgres.integration.test.ts` (3 clients; did not strip `sslmode` — pre-existing latent risk)
11. `p1-013-label-generation-postgres.integration.test.ts`
12. `p3-001-reservation-release-postgres.integration.test.ts`
13. `p3-002-postgres.integration.test.ts`
14. `p3-003-postgres.integration.test.ts` (2 call sites)

Also found and removed: an **unused** `import { Client } from 'pg'` in `p1-012-carrier-selection-postgres.integration.test.ts` — no `new Client(...)` construction existed in that file; the import was dead code, not a real observer.

## Migrated files

All 14 files above were migrated to import and call `createPgObserverClient(...)` (and `p1-007`'s dead local helper was deleted). `p1-012`'s unused `Client` import was removed. Exact diff: `outbound/post-x002-raw-pg-ssl` vs `outbound/x-002` in `apps/mercato/src/modules/wms_outbound/services/__tests__/`.

## Intentional exception

`apps/mercato/src/modules/wms_outbound/__integration__/P2-006-crossdock-shipment-downstream-ui.spec.ts` (Playwright UI spec) performs its own manual `sslmode` stripping, but constructs a `pg.Pool` (not `pg.Client`) for a single one-time role-lookup query in `test.beforeAll`. This is a different construct outside the helper's documented `pg.Client`-only contract, and this maintenance gate's scope explicitly excludes Playwright/UI test changes unless they unexpectedly require them. Left unmodified. Rationale recorded here per the guide's requirement to document any intentional exception.

## Decisive proof

### 1. Helper proof against canonical Testing PostgreSQL

New file: `apps/mercato/src/modules/wms_outbound/services/__tests__/support/pg-observer-client-postgres.integration.test.ts`

Command (canonical Testing env sourced in the same invocation, no secrets echoed):

```
set -a && source /etc/mercato-localhost.env && set +a && npx jest --config jest.config.cjs --forceExit \
  src/modules/wms_outbound/services/__tests__/support/pg-observer-client-postgres.integration.test.ts
```

Result: `Test Suites: 1 passed, 1 total` / `Tests: 2 passed, 2 total` — verified live connection to `aws-1-eu-central-1.pooler.supabase.com:6543` (remote Supabase pooler, confirmed non-loopback host before running), `SELECT 1`/`pg_backend_pid()` succeeded, clean `.end()`, and the "DATABASE_URL is required" path throws without needing a real connection. No connection string or credential value was logged.

### 2. Real-concurrency suites preserved on final head

All directly affected migrated suites rerun against canonical Testing PostgreSQL:

```
p1-006, p1-007, p1-008           -> 3 suites, 56/56 PASS
p1-009, p1-010, p1-011, p1-013   -> 4 suites, 63/64 PASS (1 pre-existing failure, see below)
p1-014, p1-015, p1-016           -> 3 suites, 64/64 PASS
p2-004, p3-001, p3-002, p3-003   -> 4 suites, 63/63 PASS
p1-012 (dead-import removal only) -> 1 suite, 14/14 PASS
```

Total: **15 suites, 260/261 PASS** (pooled). All decisive PostgreSQL-side proofs (`pg_blocking_pids`, backend PID, `wait_event_type='Lock'`, real overlapping connections) remained intact and unweakened — centralization changed only client construction, not assertions.

**Pre-existing failure, confirmed not caused by this gate:** `p1-009-postgres.integration.test.ts` test `"6A: Distinct PostgreSQL connections capture real lock contention during concurrent pick confirmation & TU closure"` fails a post-lock business assertion (`resultB.pickTaskLine.pickedQuantity` expected `'5.000000'`, received `'0.000000'`) — unrelated to SSL/observer-client construction. Reproduced identically by stashing this gate's changes and rerunning the same test against the unmodified `outbound/x-002` head (`4a89a95aad42c476ac206b53fe8ff67f3c8021a9`): same failure, same assertion, same values. This is a pre-existing product/test defect outside this maintenance gate's test-infrastructure-only scope; not corrected here. Flagging for Supervisor/Owner triage as a separate item.

### 3. Static completion check (post-migration)

```
grep -rn "new Client(" apps/mercato/src/modules/wms_outbound --include="*.ts" | grep -v "/support/"
  -> no results

grep -rln "sslmode" apps/mercato/src/modules/wms_outbound --include="*.ts" | grep -v "/support/"
  -> apps/mercato/src/modules/wms_outbound/__integration__/P2-006-crossdock-shipment-downstream-ui.spec.ts
     (documented exception above — pg.Pool, not pg.Client)

grep -rn "^import { Client } from 'pg'" apps/mercato/src/modules/wms_outbound --include="*.ts" | grep -v "/support/"
  -> no results
```

No remaining duplicated hand-written raw `pg.Client` SSL/`sslmode` normalization in the WMS Outbound test tree outside the shared helper and the one documented `pg.Pool` exception.

### 4. Typecheck

```
npm run typecheck --workspace apps/mercato
```

Result: clean (`tsc --noEmit`, no errors), run twice — once before test suite reruns and once after all edits were finalized.

## Explicit statement

Test infrastructure only. No business/application DB configuration, credential rotation, MikroORM SSL configuration, Scanner code, or UI/Playwright behavior was changed. No ACC-001 work was started. Scanner head verified unchanged (`a2759a29347285dd1dcd14bf51633431fbf2a302`) before and after this gate.

## Completion boundary

This is executor COMPLETE evidence only. Supervisor FINAL PASS and explicit Owner Acceptance remain separate, subsequent steps. Do not start `ACC-001` until the Owner explicitly authorizes it after this gate is accepted.
