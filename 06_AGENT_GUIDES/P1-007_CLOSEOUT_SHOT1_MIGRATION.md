# P1-007 Final Closeout — Shot 1: Deterministic Migration Proof

Owner explicitly requested that P1-007 be closed with **all remaining problems resolved, without exceptions**.

This is **SHOT 1 ONLY** of the final closeout. Do not execute the later evidence-closeout shot in the same turn.

## Supervisor execution contract

- Required skills before launch: `fetch_me_prompt` + `operational-mode`.
- Also load `wms-outbound` for Outbound authority/routing and `architecture-context` only for accepted Inbound/shared-foundation compatibility context.
- `fetch_me_prompt` governs ticket content; `operational-mode` governs model, launch, supervision, retries/fail-stop, notifications and completion handling.

## Failure boundary

- The same action/path may fail at most twice.
- After the second materially identical failure: STOP and report the exact action, exact error/failure, repo/branch/HEAD/runtime state, checks already run, and the exact remaining item/decision needed.
- No third retry, workaround route, model/tool/executor switch, scope expansion or redesign without supervisor instruction.

## FIRST: synchronize locally

1. Use the **same existing Antigravity session**.
2. Sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.
3. Read this LOCAL file completely.
4. Refresh the locally installed skills named above and the current `.ai/TESTING.md`, `.ai/OPERATIONS.md`, and `REAL_EVIDENCE_CONTRACT.md` if available.
5. Work only in the current P1-007 Mercato branch and the smallest WMS evidence/doc scope required by this shot.

## Authority / why this shot exists

`wms-outbound` establishes that Outbound business behavior comes from the current `WMS_Outbound` Architect/canon chain, while `architecture-context` is only shared Inbound/reference compatibility context.

For this shot, business behavior is already closed. Do **not** reinterpret R42–R48.

Relevant fixed authority:

- `STATE.md` — P1-007 is the current item, item 11/37.
- `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_1_standard_fulfillment_EN.md` — R42–R48 and SHORT exception table.
- `01_ARCHITECT_TRANSLATIONS/2026-08-31/wymagania_outbound_EN.md` — FR-P1-21..24 and FR-P5-01..06.
- `01_ARCHITECT_TRANSLATIONS/2026-08-31/scenariusze_testowe_outbound_EN.md` — TC-060/061/062/121.
- `05_EVIDENCE/EVIDENCE_STANDARD.md` — real executable evidence, no document-only acceptance.
- `06_AGENT_GUIDES/P1-007_EXECUTION.md` — P1-007 decomposition and hard boundary, including durable P4 recovery handoff instead of implementing the future P4 PutBackTask lifecycle here.

## Current verified bases — preserve

- Mercato `outbound/p1-007`: `aa7b8e57c685b6ac4de28dfeccbea0b2be79db8f`
- Scanner `main`: `b23325aae1c4f83b79d01b3650dbead3486a1041`
- Current durable P1-007 evidence: WMS_Outbound commit `290f9f7de7d67fd54a2e4bca10c675d3581fbd8d`

The following P1-007 areas are independently considered product-correct and are **not** to be redesigned in this shot:

- SHORT_ALLOCATED partial/non-partial outcomes;
- R44 effective retry hierarchy and same unresolved shortage identity;
- per-location + warehouse-wide hard-reservation bounds, including pre-PickTask reservations;
- strict SAME-K PostgreSQL concurrency proof;
- SHORT_PICKED canonical Supervisor idempotent replay;
- Supervisor role/canonical actor/idempotency boundary;
- R45 WAIT logical recovery and durable physical-return handoff;
- R46 exact-picked correction and true zero commercial/execution cancellation;
- R47 persistent per-`CustomerOrder` allowPartial behavior;
- R48 backend seam;
- six Mercato Supervisor UI outcomes;
- Scanner short-pick -> replacement-task continuation.

## The remaining acceptance defect

Current test `6A` claims real migration UP/DOWN/re-UP proof, but the compatible DOWN/re-UP branch is guarded by:

```ts
if (existingIncompatible === 0) {
  // DOWN -> >0 -> re-UP
}
```

Earlier P1-007 tests can legitimately create `ordered_quantity = 0` / `required_qty = 0` rows. Therefore the compatible DOWN/re-UP branch may be skipped while the test still passes.

Also, the P1-007 suite `beforeAll` currently hand-applies the `>= 0` constraints using raw `ALTER TABLE`. That can mask migration/deployment drift and weakens the claim that the real migration path was proven.

The migration implementation itself already has the correct intended safety policy:

- UP: `> 0 -> >= 0` for only the two P1-007 quantity constraints;
- DOWN: fail fast **before mutation** when incompatible zero/negative rows exist;
- compatible database: restore `> 0`;
- no data deletion or quantity rewriting.

## Required fix — deterministic, no skip, no fake DDL

Create the smallest deterministic migration-test harness that proves the **actual migration implementation** in isolation.

### A. Compatible-data path MUST execute

The test must unconditionally execute all of these using the actual migration class / project migrator path:

1. Start from a deterministic database state compatible with the old `> 0` constraints.
2. Execute actual migration **UP**.
3. Fresh DB inspection proves both constraints are `>= 0`.
4. Execute actual migration **DOWN** — **no `if`, no skip, no conditional early success**.
5. Fresh DB inspection proves both constraints are restored to `> 0`.
6. Execute actual migration **UP again**.
7. Fresh DB inspection proves both constraints are `>= 0` again.

This path must fail the test if any of steps 1–7 did not actually execute.

### B. Incompatible-data fail-safe path MUST execute separately

In a separately isolated fixture/state under UP:

1. Create deterministic legitimate cancelled P1-007 zero-quantity rows using the real R46 service path where practical; if the migration harness needs a minimal fixture insert, keep it exact-ID scoped and explain why.
2. Execute actual migration **DOWN**.
3. Assert the explicit compatibility error.
4. Assert zero ALTER/DROP constraint mutation was applied after the compatibility check.
5. Fresh DB proves the zero rows remain intact and unchanged.
6. Fresh DB proves constraints remain `>= 0`.
7. Clean up only the exact deterministic fixture IDs.

### C. Isolation requirement

Do not make compatible-path success depend on whatever historical test rows happen to exist in the shared remote DB.

Use the closest **existing project-approved migration testing pattern** discovered from current testing/operations guidance. Preferred order:

1. existing disposable test DB / isolated migration harness already used by the project;
2. existing isolated database/schema mechanism that genuinely executes this migration against the target table model;
3. another existing project pattern that guarantees deterministic preconditions without touching unrelated rows.

Do **not** invent a shadow migration, duplicate production tables just to make assertions easy, or broad-delete existing zero rows.

If the shared test database cannot provide a clean compatible state, the test must create/use an approved isolated DB/harness rather than `if/skip` the proof.

### D. Remove masking setup

Remove the suite-global manual `ALTER TABLE ... >= 0` preparation from P1-007 `beforeAll` **unless** the current project testing contract explicitly requires it for a separate non-migration reason.

The normal P1-007 suite should rely on the schema being brought to the current migration state by the real migration/test setup. If the expected migration is absent, fail visibly instead of silently patching the schema in test setup.

## Decisive proof markers

Persist concise proof that makes a false-green impossible. The migration test output/evidence must expose at least equivalent facts to:

```text
compatibleUpExecuted=true
compatibleDownExecuted=true
compatibleReUpExecuted=true
incompatibleDownBlocked=true
incompatibleRowsPreserved=true
constraintsAfterBlockedDown=>=0
```

The exact logging format may follow project conventions; these facts must be independently inferable from the test and fresh DB assertions.

## Verification for this shot

Run only what is needed to verify this migration-focused change:

1. the deterministic migration proof test;
2. the focused P1-007 real PostgreSQL suite after removing any masking setup;
3. any migration-specific lint/typecheck/check required by current testing guidance.

Do **not** spend this shot rerunning the full cross-ticket regression matrix; that is the separate final evidence-closeout shot after the supervisor independently verifies this one.

## Evidence update for this shot

Update only the migration-proof subsection of `05_EVIDENCE/P1-007_EVIDENCE.md` if needed to record:

- exact final Mercato 40-char HEAD;
- exact migration test command;
- compatible UP/DOWN/re-UP actually executed with no skip;
- incompatible DOWN fail-safe actually executed with rows preserved;
- confirmation that the suite no longer hand-patches the constraints in `beforeAll` (or the exact current-contract reason if a project-approved setup remains).

Do not self-declare `HUMAN VERIFIED` or `FINAL PASS`.

## Hard boundary / STOP

Do not change Scanner.
Do not redesign P1-007 product behavior.
Do not implement P4 PutBackTask lifecycle.
Do not touch P1-009/P1-010, Shipment, Carrier, labels, ERP, manifest, packing/QC, or unrelated cleanup.

Push the minimal Mercato migration/test change and the bounded evidence update, then **STOP**.

The supervisor will independently verify this shot before issuing the separate final evidence-closeout shot.