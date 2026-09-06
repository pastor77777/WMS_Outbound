# Post-X-002 raw PostgreSQL SSL maintenance gate

**Status:** grounded / prepared, not launched  
**Effective:** 2026-09-07  
**Position:** mandatory non-catalog maintenance gate after X-002 and before ACC-001  
**Catalog impact:** none — does not change the 37-item count  
**Launch prerequisite:** X-002 Supervisor FINAL PASS **and explicit Owner Acceptance**  
**Mercato base after prerequisite is satisfied:** `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9` -> new branch `outbound/post-x002-raw-pg-ssl`  
**Frozen:** Scanner `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`

## Objective

Remove duplicated hand-written raw `pg.Client` SSL normalization from WMS Outbound PostgreSQL observer/concurrency tests and replace it with one canonical **test-only** helper/factory that is safe against the current Testing `DATABASE_URL` containing `sslmode=require`.

This gate is test infrastructure only. It must not alter Outbound business behavior, application database configuration, credentials, Architect contracts, Demo/Prod, or Scanner.

## Authority and current problem

Primary maintenance plan:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

The accepted Testing environment injects `DATABASE_URL`. With the current `pg@8.22.0` / `pg-connection-string@2.14.0`, connection-string parsing can let `sslmode=require` override an explicit raw-client `ssl: { rejectUnauthorized: false }`, causing `self-signed certificate in certificate chain` before concurrency assertions run.

The X-001 correction removed `sslmode` locally at four known raw observer call sites, but the logic stayed duplicated. X-002 added a fifth equivalent observer construction in its decisive INT-02 concurrency proof.

## Known final-head call sites to migrate

Audit and replace the equivalent hand-written raw observer-client construction in at least:

1. `p1-011-postgres.integration.test.ts`
2. `p1-014-erp-posting-postgres.integration.test.ts`
3. `p1-015-manifest-lifecycle-postgres.integration.test.ts`
4. `p1-016-final-settlement-postgres.integration.test.ts`
5. `p2-004-crossdock-recovery-postgres.integration.test.ts`

Before editing, search the complete WMS Outbound test tree for equivalent patterns, including imports/usages of `Client` from `pg`, `new Client(...)`, `pg.Client`, and manual `sslmode` stripping. Route equivalent Testing observer connections through the helper. If a raw `pg.Client` use is genuinely different and should remain direct, document why in evidence.

## Required helper contract

Create one narrow test-only helper/factory in the WMS Outbound test-support area. Exact file/API naming is implementation choice, but the final helper must satisfy all of the following:

- consume the runtime-injected Testing `DATABASE_URL`; never hard-code credentials;
- fail clearly if the required environment URL is absent;
- centralize removal/normalization of connection-string SSL parameters that can clobber the explicit approved Testing SSL behavior;
- apply the explicit Testing raw-client SSL option in one place;
- never print, snapshot, persist or include the full connection string, password, auth material or secret-bearing config in errors/evidence;
- preserve each existing suite's connection-routing semantics. Do not silently change direct-vs-pooler port behavior merely to centralize the helper;
- remain test-only; do not move this helper into application/runtime DB configuration;
- return/use ordinary `pg.Client` behavior needed by the existing PostgreSQL observer proofs without weakening their concurrency assertions.

Do not solve this by globally disabling TLS verification in application code or by changing Testing credentials/certificates.

## Implementation boundary

Allowed:

- one shared test-only raw PostgreSQL observer helper/factory;
- minimal import/construction edits in equivalent WMS Outbound PostgreSQL tests;
- a focused helper integration proof;
- minimal fixture/test-support corrections required to keep the directly affected observer suites decisive.

Not allowed:

- product/business logic changes;
- application DB config refactor;
- MikroORM SSL changes unless a new, independently proven defect requires Owner escalation;
- credential rotation or secret handling redesign;
- local PostgreSQL;
- Demo/Prod;
- Scanner changes;
- ACC-001 work or any broad acceptance sweep.

## Decisive proof

### 1. Helper proof against canonical Testing PostgreSQL

Add the smallest real PostgreSQL test/proof that constructs the helper using the runtime Testing environment, successfully connects, performs a harmless non-secret query such as `SELECT 1` / `SELECT pg_backend_pid()`, and closes cleanly.

The proof must not log the connection string or credentials.

### 2. Preserve real concurrency evidence

Rerun every migrated raw-client concurrency/observer suite. Their decisive PostgreSQL-side assertions must remain intact. Centralization is not allowed to replace `pg_blocking_pids`, backend PID, lock-wait, observer-connection, or other accepted real-concurrency evidence with timing-only assertions.

Known directly affected suites on the X-002 final head:

- `p1-011-postgres.integration.test.ts`
- `p1-014-erp-posting-postgres.integration.test.ts`
- `p1-015-manifest-lifecycle-postgres.integration.test.ts`
- `p1-016-final-settlement-postgres.integration.test.ts`
- `p2-004-crossdock-recovery-postgres.integration.test.ts`

If the initial search finds additional equivalent Outbound raw observer clients and they are migrated, rerun their owning suites too.

### 3. Static completion check

After migration, search the scoped WMS Outbound test tree again and prove there is no remaining duplicated equivalent hand-written raw observer construction / `sslmode` stripping. Any intentional exception must be listed with exact file and rationale.

### 4. Typecheck

Run current Mercato typecheck for the changed test/support surface.

No Playwright, Scanner, runtime rebuild, or full Outbound acceptance sweep is required unless the maintenance change unexpectedly touches a user-visible/runtime surface; such expansion requires Supervisor review first.

## Testing hygiene

This gate is explicitly **non-catalog**. The mandatory deep Testing reset in `Devaxonic-WMS/.ai/OPERATIONS.md` applies before every new WMS Outbound **Task Catalog item**, not automatically before this maintenance gate.

Do not invent a deep reset requirement for this gate. Use the current canonical Testing environment and source `/etc/mercato-localhost.env` in the same invocation for DB-backed commands as required by `.ai/TESTING.md`. Never print secrets.

## Evidence

Write:

`WMS_Outbound/05_EVIDENCE/POST_X002_RAW_PG_SSL_MAINTENANCE_EVIDENCE.md`

Include:

- exact Mercato branch/head and base `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`;
- frozen Scanner head;
- helper path and its exact contract;
- complete pre-change search result for equivalent raw observer call sites;
- exact migrated files;
- any intentional exception with rationale;
- focused real-PostgreSQL helper proof command/result;
- exact directly affected concurrency-suite commands/results on final head;
- typecheck result;
- post-change static search proving duplication is removed;
- explicit statement: test infrastructure only, no business/application DB config/credential/Scanner change, no ACC-001 started.

Do not claim Supervisor FINAL PASS or Owner Acceptance.

## Completion contract

Return COMPLETE only when:

- one canonical test-only raw `pg.Client` observer helper/factory exists;
- all equivalent known/current Outbound observer call sites are migrated or explicitly justified;
- canonical Testing connection proof is green without secret exposure;
- directly affected real-concurrency suites are green on final head with decisive PostgreSQL-side evidence preserved;
- Mercato typecheck is clean;
- maintenance evidence is pushed;
- Mercato `outbound/post-x002-raw-pg-ssl` is pushed;
- Scanner remains exactly frozen;
- no business/application DB config/credential behavior changed;
- no ACC-001 work has started.

Ordinary in-scope test/fixture/tooling/runtime failures are executor-owned self-repair under the existing two-strikes rule.

## Sequence boundary

After this gate later receives independent Supervisor FINAL PASS, STOP. Do not start `ACC-001` until the Owner explicitly authorizes it.

Required project sequence remains:

`X-002 -> raw pg SSL maintenance gate -> ACC-001 -> ACC-002 -> ACC-003 -> ACC-004`
