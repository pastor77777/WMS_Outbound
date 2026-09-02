# P1-007 Final Closeout — Explicit Override Shot 1C: Real MikroORM Migrator Only

Owner explicitly overrides the prior STOP for **one final narrow migration-proof shot only**.

This shot exists because Shot 1B removed shadow-table DDL but still executed migration classes manually via `migration.up() -> getQueries() -> em.execute()`, even though the Mercato project has a real MikroORM migration runner.

Use the SAME existing Antigravity session. Do not execute any later final evidence-closeout work in this turn.

## Supervisor execution contract

- Required skills before launch: `fetch_me_prompt` + `operational-mode`.
- Also load `wms-outbound` for Outbound authority/routing and `architecture-context` only for accepted Inbound/shared-foundation compatibility.
- `fetch_me_prompt` governs ticket content; `operational-mode` governs execution/supervision/fail-stop.

## Failure boundary

- This is an owner-authorized override after two failed migration-proof attempts.
- Exactly one narrow implementation attempt is permitted.
- If this attempt does not produce a genuine real-Migrator proof, STOP. No further workaround, alternate harness, manual SQL lifecycle, shadow schema approximation, or scope expansion.

## Current verified bases

- Mercato `outbound/p1-007`: `1979942c993688be84d8765c0baae0eb594f9aa9`
- Scanner `main`: `b23325aae1c4f83b79d01b3650dbead3486a1041` — DO NOT CHANGE
- Current durable P1-007 evidence: WMS_Outbound `2dfb89f5fd533a4c40bb85f8a2394cb0bae0787e`

All P1-007 product semantics/UI/Scanner/concurrency behavior are out of scope for redesign.

## Grounded project migration authority

Mercato already exposes the project database migration path:

- `apps/mercato/package.json`: `db:migrate = mercato db migrate`.
- `packages/cli/src/lib/db/commands.ts`: `dbMigrate()` initializes MikroORM with `@mikro-orm/migrations`, obtains `const migrator = orm.migrator as Migrator`, uses `migrator.getPending()` and executes migrations with `migrator.up(...)`.

Therefore this proof MUST exercise the actual MikroORM `Migrator` lifecycle. A migration class instantiated directly and its `getQueries()` executed through `em.execute()` is NOT sufficient for Shot 1C.

## Exact defect to remove

Current test 6A still imports a list of migration classes and manually does roughly:

```ts
const m = new MigClass(driver, orm.config)
await m.up()
for (const q of m.getQueries()) await em.execute(q)
```

and similarly for remediation UP/DOWN/re-UP.

Remove this manual migration lifecycle from the decisive migration proof.

## Required implementation

### 1. Use a genuine MikroORM `Migrator`

Build the smallest deterministic migration-test harness using the same migration configuration semantics as `dbMigrate()`:

- real `MikroORM.init<PostgreSqlDriver>(...)`;
- `@mikro-orm/migrations` Migrator extension/path;
- actual committed `wms_outbound` migration files discovered by the migration runner;
- an isolated, exact-scope migration metadata table for this harness where needed;
- an isolated PostgreSQL schema/database mechanism only if it is supported by the real runner and genuinely routes the migration connection there.

The decisive lifecycle operations must be calls equivalent to:

```ts
const migrator = migrationOrm.migrator as Migrator
await migrator.up(...)
await migrator.down(...)
await migrator.up(...)
```

NOT `new MigrationClass`, NOT `getQueries()`, NOT manual replay of production migration SQL.

If a helper from the project can create/configure the migration ORM safely, reuse it. Otherwise mirror only the configuration shape necessary to instantiate the real Migrator; do not copy migration SQL or target-table DDL.

### 2. Real history, real target model

The isolated pre-remediation state must be produced by the real migration runner applying the actual committed preceding `wms_outbound` migrations in order, so the migration metadata/history agrees with the schema.

Do not hand-create the target tables.
Do not directly instantiate preceding migration classes.
Do not insert fake rows into the MikroORM migrations metadata table to pretend migrations ran.

Fresh PostgreSQL inspection must prove that immediately before remediation:

- the two real target constraints are `> 0`;
- the remediation migration is pending according to the real Migrator.

### 3. Compatible lifecycle — mandatory, no skip

Through the real `Migrator` only:

1. Apply remediation migration UP.
2. Fresh DB => both constraints `>= 0`.
3. Execute real Migrator DOWN of the remediation migration.
4. Fresh DB => both constraints `> 0`.
5. Execute real Migrator UP again.
6. Fresh DB => both constraints `>= 0`.
7. Migrator history/pending state is consistent after every boundary.

No `if`, no skip, no broad cleanup.

### 4. Incompatible-data fail-safe — separate real Migrator path

Under the real UP state:

1. Create exact-ID scoped legitimate cancelled zero-quantity data in the actual migrated isolated target tables. Prefer real R46 service path when practical; exact fixture inserts are acceptable only as precondition setup.
2. Call real `migrator.down(...)` for the remediation migration.
3. Assert explicit compatibility failure.
4. Fresh DB proves zero rows unchanged.
5. Fresh DB proves both constraints remain `>= 0`.
6. Fresh Migrator state proves remediation remains applied / was not falsely rolled back in migration history.
7. Clean only exact fixture IDs or drop the disposable isolated schema/harness at the end.

### 5. Prove this is not the manual path

The decisive test must make it reviewable that:

- no `Migration2026...` class is instantiated directly for lifecycle execution;
- no `getQueries()` is used for decisive migration execution;
- no hand-written `CREATE TABLE` reproduces production tables;
- no hand-written `ALTER TABLE` reproduces remediation UP/DOWN;
- actual `Migrator.up/down/getPending/getExecuted` (or equivalent real API) drives the lifecycle.

A static review of the final test should be enough to distinguish it from Shots 1/1B.

## Verification for Shot 1C only

Run and persist:

1. the corrected deterministic migration proof using real Migrator;
2. focused P1-007 real PostgreSQL suite with no suite-global schema hand-patch;
3. any migration-specific typecheck/lint required by current project guidance.

Do not rerun the full cross-ticket matrix in this shot.

## Evidence correction

Update only the migration-proof subsection of `05_EVIDENCE/P1-007_EVIDENCE.md` after proof passes.

Record truthfully:

- exact final Mercato 40-char HEAD;
- exact test command;
- exact real migration runner/API used;
- isolated harness mechanism;
- real Migrator preceding-migration setup;
- remediation real Migrator UP -> DOWN -> re-UP;
- incompatible real Migrator DOWN blocked pre-mutation with data and migration-history state preserved;
- confirmation that direct migration-class `getQueries()/em.execute()` is no longer the decisive proof.

Do not self-declare HUMAN VERIFIED or FINAL PASS.

## Hard boundary / STOP

Do not change Scanner.
Do not change P1-007 product services/business semantics unless compilation requires a purely mechanical import/test seam change.
Do not change UI.
Do not implement P4 PutBackTask.
Do not touch P1-009/P1-010, packing/QC, Shipment, Carrier, labels, ERP, manifest, or unrelated cleanup.

Push only the minimal Mercato migration-test/harness correction + bounded evidence correction, then STOP.