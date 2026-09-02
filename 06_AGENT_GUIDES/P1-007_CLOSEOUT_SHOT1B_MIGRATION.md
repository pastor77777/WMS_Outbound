# P1-007 Final Closeout — Corrective Shot 1B: Real Migration Harness, No Shadow Tables

Owner explicitly authorized closing P1-007 with all remaining problems resolved, without exceptions.

This is the ONE corrective retry for the migration-proof path after Shot 1. Stay in the SAME existing Antigravity session. Do not execute the later final evidence-closeout shot in this turn.

## Supervisor execution contract

- Required skills before launch: `fetch_me_prompt` + `operational-mode`.
- Also load `wms-outbound` for Outbound authority/routing and `architecture-context` only for accepted Inbound/shared-foundation compatibility context.
- `fetch_me_prompt` governs ticket content; `operational-mode` governs model, launch, supervision, retries/fail-stop, notifications and completion handling.

## Failure boundary

- This is retry 2 for this exact migration-proof path.
- If this materially fails again: STOP and report exact action, error/failure, branch/HEAD/runtime, checks run, and remaining blocker. No third workaround or alternate fake proof.

## FIRST: synchronize locally

1. Use the SAME Antigravity session.
2. Sync local `WMS_Outbound` from remote `main`, preserving unrelated local work.
3. Read this LOCAL file completely.
4. Refresh current installed `wms-outbound`, `architecture-context`, `fetch_me_prompt`, `operational-mode`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, and `REAL_EVIDENCE_CONTRACT.md` if present.
5. Work only in Mercato `outbound/p1-007` and the smallest WMS evidence scope required by this corrective shot.

## Current verified state

- Mercato P1-007 HEAD after failed Shot 1: `4c53cbdeb9a7b1a1169bbb84aba08bb909d9db75`.
- Parent/base for Shot 1: `aa7b8e57c685b6ac4de28dfeccbea0b2be79db8f`.
- Scanner remains unchanged: `b23325aae1c4f83b79d01b3650dbead3486a1041`.
- WMS evidence commit after Shot 1: `dfdc6e1319416544e5bb9c250373a5b632df9f3b`.

All previously verified P1-007 product behavior remains out of scope for redesign.

## Exact Shot 1 defect

Shot 1 correctly removed the suite-global masking `ALTER TABLE` and removed the `if (existingIncompatible === 0)` false-green.

However test 6A then created hand-written copies of:

- `wms_outbound_customer_order_lines`
- `wms_outbound_order_lines`

inside a temporary schema using manual `CREATE TABLE ...` DDL and ran the migration class queries against those copies.

That violates the Shot 1 isolation rule:

> Do not invent a shadow migration, duplicate production tables just to make assertions easy.

It also weakens the deployment claim because the test is proving behavior against a hand-maintained approximation of the target schema rather than the project-approved migration/schema lifecycle.

The current durable evidence therefore overstates Shot 1 and must be corrected after the real proof exists.

## Required correction

Replace ONLY the migration-proof harness. Do not change P1-007 product services/UI/Scanner.

### 1. Discover and use the closest existing project-approved migration test pattern

Before editing, inspect only the relevant Mercato database/migration configuration and existing migration tests/scripts.

Use the project's real MikroORM migration configuration/runner (`Migrator`/CLI/project helper as actually used in this repo) wherever available.

Do NOT hand-create production-table definitions and do NOT manually reproduce migration DDL in the test.

Allowed isolation mechanisms, in order:

1. an existing disposable test database/harness already used by this project;
2. an existing schema-per-test migration harness where the schema is produced by the REAL project migration/schema lifecycle, not hand-written `CREATE TABLE` copies;
3. a transactionally isolated approved project migration-test mechanism against the real target tables, provided it does not rewrite/delete unrelated shared Testing data and all state is safely restored.

If none exists, STOP and report that exact infrastructure gap rather than inventing another shadow schema.

### 2. Compatible path must be real and unconditional

Using the approved real migration harness and actual migration implementation, prove all of:

- deterministic pre-migration state with the real target table model and `> 0` constraints;
- actual P1-007 migration UP;
- fresh PostgreSQL inspection => both constraints `>= 0`;
- actual migration DOWN with no `if`, skip, conditional success or broad data cleanup;
- fresh PostgreSQL inspection => both constraints `> 0`;
- actual migration re-UP;
- fresh PostgreSQL inspection => both constraints `>= 0`.

The test must fail if any step did not execute.

Do not replace the real migration runner with `migration.up(); getQueries(); em.execute(...)` if the project has a real Migrator/CLI/helper capable of exercising this migration lifecycle. Use the actual project path.

### 3. Incompatible-data fail-safe must be separate and real

Under the real UP state:

- create exact-ID scoped legitimate zero-quantity cancelled data through the R46 service path where practical; a minimal fixture insert is allowed only when required by the migration harness and must use real target tables, not duplicates;
- attempt actual DOWN through the same migration path;
- assert explicit compatibility failure BEFORE constraint mutation;
- fresh DB proves zero rows remain unchanged;
- fresh DB proves constraints remain `>= 0`;
- clean only exact fixture IDs if the chosen approved harness requires cleanup.

No broad `DELETE`, no quantity rewrite to make DOWN pass, no manual `ALTER TABLE` to reset the world.

### 4. Keep normal P1-007 suite honest

The normal P1-007 PostgreSQL suite must continue without the removed suite-global hand-patch. If the expected migration is absent in the Testing schema, fail visibly.

### 5. Evidence must state what ACTUALLY ran

Update the migration subsection of `05_EVIDENCE/P1-007_EVIDENCE.md` only after the real proof passes.

Record:

- exact 40-char Mercato HEAD;
- exact command;
- exact migration runner/harness used;
- actual UP -> DOWN -> re-UP executed with no skip;
- separate incompatible DOWN blocked pre-mutation with zero rows preserved;
- normal suite no longer hand-patches constraints.

Remove/replace any Shot 1 wording that claims the hand-written shadow tables were an acceptable real migration lifecycle proof.

Do not self-declare `HUMAN VERIFIED` or `FINAL PASS`.

## Verification for this corrective shot

Run only:

1. corrected deterministic migration proof;
2. focused P1-007 real PostgreSQL suite;
3. migration-specific typecheck/lint/check required by current project guidance.

Do not rerun the full regression matrix yet. That remains the final closeout shot after supervisor independently accepts this migration proof.

## Hard boundary / STOP

Do not change Scanner.
Do not redesign R42-R48/R69/R71 behavior.
Do not implement P4 PutBackTask lifecycle.
Do not touch P1-009/P1-010, packing/QC, Shipment, Carrier, labels, ERP, manifest, or unrelated cleanup.

Push minimal Mercato migration-test/harness correction + bounded evidence correction, then STOP.