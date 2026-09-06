# P4-003 — RF PutBack Location Validation Loop and Inventory Recovery — Evidence

Date: 2026-09-06 UTC
Evidence class: REAL POSTGRESQL INTEGRATION, REAL CONCURRENCY and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/p4-003`
- Mercato final candidate commit SHA: `6ddec6870` (correction; supersedes `203caba63`)
- Mercato accepted base: `outbound/p4-002` at `d75dccbc7a43b64c2a0ed325d30a7a4b49b57eb1` (FINAL PASS / Owner Accepted)
- Scanner branch: `outbound/p4-003`
- Scanner final candidate commit SHA: `a2759a2` (unchanged by the correction — frozen, no UI regression required)
- Scanner accepted base: `outbound/p4-002` at `7d13e34fc66fe19149b43a6747b5300c2bdcf945` (FINAL PASS / Owner Accepted)
- WMS evidence commit: this commit, on `WMS_Outbound/main`
- Item: 31/37 — P4-003: RF PutBack location validation loop and Inventory recovery.
- Execution guide: `06_AGENT_GUIDES/P4-003_EXECUTION.md` (supervisor-grounded, commit `182cc3c`), executed within this session (Claude Code as executor, per `Devaxonic-WMS/CLAUDE.md`). This session independently drafted its own preliminary guide before discovering the supervisor's grounded version already existed on `origin/main`; reconciled by keeping the supervisor's guide as authoritative and cross-checking this implementation against its explicit invariants (see "Reconciliation" below).
- Correction guide: `06_AGENT_GUIDES/P4-003_COMPLETION_CORRECTION.md` (supervisor-found gap, commit `fad23c5`), executed as a same-item continuation (see "Completion Correction" below). No Testing reset was performed for this continuation, per the correction guide's explicit instruction.

## Authority Chain

`proces_4_physical_putback.md` P4 STEP 4-5, R6-R8 -> `FR-P4-03`, `FR-P4-04` -> `scenariusze_testowe_outbound.md` `TC-050`, `TC-051`, `TC-052`; regression of `TC-100` (`FR-P4-05`, P4-002).

## Schema / Migration

- `Migration20260906173835_wms_outbound_p4_003_location_validation.ts`: adds nullable `validated_location_id uuid` to `wms_outbound_put_back_tasks`. Additive only; no destructive change; applied via `mercato db migrate` against canonical Testing PostgreSQL.
- `PUT_BACK_TASK_TRANSITIONS` (`data/transitions.ts`) already declared `IN_PROGRESS -> LOCATION_VALIDATION`, `LOCATION_VALIDATION -> IN_PROGRESS`, `LOCATION_VALIDATION -> COMPLETED` from P4-002; reused as-is, not redefined.

## Location Validation and Inventory Recovery Mechanism

`put-back-task-service.ts` `submitLocation(scope, { taskId, operatorId, scannedLocation })`, single DB transaction:

1. `PESSIMISTIC_WRITE` lock on the task row; ownership check; idempotent short-circuit if already `COMPLETED`.
2. `IN_PROGRESS -> LOCATION_VALIDATION` transition, audited (`PutBackTaskLocationSubmitted`).
3. Resolves the scanned/indicated destination against `WmsWarehouseLocation` scoped to the task's warehouse, matching `name` or `structuredAddressCode` always, and `id` only when the scanned value is itself a well-formed UUID (a non-UUID value in that comparison position fails at the Postgres type level, not as "no match" — found and fixed during this item's own test development).
4. If no location resolves, or resolves to a location that fails the *existing, unmodified* `wms_inventory` `createInventoryAdjustmentService(...).adjustBalance(...)` reference checks (`LOCATION_NOT_FOUND`, `LOCATION_NOT_STORAGE`, `WAREHOUSE_NOT_FOUND`) — **reject**: `LOCATION_VALIDATION -> IN_PROGRESS`, audited (`PutBackTaskLocationRejected`, with reason), zero Inventory mutation, `proposedLocationId` (the standing System WMS recommendation) left untouched. No attempt counter exists anywhere in the schema or code; the loop is genuinely unbounded.
5. If the location is valid, `adjustBalance` (the shared, unmodified `wms_inventory` primitive — reused, not reimplemented) posts the exact task quantity as a real `WmsInventoryMovement` (`disposition = ADJUSTMENT_IN`) and increments `WmsInventoryBalance.onHandQuantity`/`availableQuantity` at the validated location, in the item's base UOM (`txnUomId === item.baseUomId`, so the UOM-conversion lookup short-circuits to a factor of 1 without requiring a `WmsInventoryUomConversion` row) — **accept**: `LOCATION_VALIDATION -> COMPLETED`, audited (`PutBackTaskCompleted`), `validatedLocationId`/`completedAt` set. All in the same transaction as the balance/movement write, so completion and recovery are atomic.
6. A missing `MasterdataItem` or missing `item.baseUomId` is a hard system error (thrown, not looped as a location rejection) — a data-integrity gap, not something a different scanned location can fix.

API route `apps/mercato/src/modules/wms_outbound/api/returns/submit-location/route.ts`: a rejected location returns **HTTP 200** with `{ accepted: false, reason, task }` — a rejection is a normal business outcome per P4 R7 (no escalation), not a request failure; only a genuine system error returns 400.

Scanner `ReturnsTaskScreen.js`/`api.js`: the `IN_PROGRESS` placeholder banner from P4-002 (explicitly staged as "the P4-003 extension point") is replaced by a real location-scan form (`returns-location-input` + `returns-submit-location-btn`); on rejection the operator sees the reason and can retry immediately with no lockout (`returns-location-rejected`); on acceptance a completion banner is shown (`returns-completed-banner`).

## Dedicated PostgreSQL Suite (8 Tests)

Suite: `apps/mercato/src/modules/wms_outbound/services/__tests__/p4-003-postgres.integration.test.ts`
Result: **8/8 PASSED** on canonical Testing PostgreSQL (Supabase pooler).

1. Rejected location (non-existent) -> task returns to `IN_PROGRESS`, zero `WmsInventoryMovement` rows, `completedAt`/`validatedLocationId` remain null.
2. Rejected location (exists, lacks `STORAGE` capability) -> same zero-movement/zero-balance assertion.
3. Unlimited retry: five consecutive rejections on the same task, then a valid location -> completes; demonstrates no attempt ceiling in code.
4. Valid location -> `COMPLETED`, `validatedLocationId` set, exactly one `WmsInventoryMovement` (`ADJUSTMENT_IN`, quantity matches exactly), `WmsInventoryBalance.onHandQuantity`/`availableQuantity` incremented by exactly the task quantity, shared active-task lock released (`ACTIVE` row absent, `RELEASED` row present).
5. Idempotent re-submission after `COMPLETED` -> zero additional movement/balance mutation, `accepted: true` returned (retry-safe).
6. Real Concurrency: two independent forked `EntityManager`s (two real DB connections/PIDs) racing `submitLocation` on the identical `IN_PROGRESS` task, synchronized via `onTxStart`/`onLockAcquired` hooks so B's transaction is decisively confirmed genuinely blocked (`pg_stat_activity.wait_event_type = 'Lock'` or an ungranted `pg_locks` row for B's real backend PID) on the row lock while A holds it, before A is permitted to proceed — exactly one `WmsInventoryMovement`/one balance increment results (the row lock, not a fresh advisory lock, serializes the pair; B's post-block read sees `COMPLETED` and takes the idempotent short-circuit).
7. Wrong operator cannot submit a location on another operator's task -> rejected with zero mutation (task status/validatedLocationId unchanged, zero Inventory movement).
8. Real rollback proof: `beforeCommitHook` forces a deterministic failure after `adjustBalance`'s real write+flush but before commit -> fresh independent read confirms task status/`completedAt`/`validatedLocationId`, `WmsInventoryMovement`, `WmsInventoryBalance`, and the task lock's `ACTIVE` status are all unchanged (nothing partially committed).

## Self-Repair During This Session

- MikroORM `$or` clauses combine into one SQL statement; comparing a non-UUID scanned string against the `id` column (even inside `$or`) fails at the Postgres type level rather than simply not matching. Fixed `submitLocation` to only include the `id` branch when the scanned value matches a UUID pattern.
- The dedicated test's fixture initially reused P4-002's shared `defaultWarehouseId` (a real warehouse belonging to a different real organization/tenant than the test's own random scope). `adjustBalance`'s `verifyReferences` scopes its `WmsWarehouse` lookup by `organizationId`/`tenantId`, so this produced a spurious `WAREHOUSE_NOT_FOUND` rejection for an otherwise-valid location. Fixed by creating a dedicated `WmsWarehouse` row scoped to each test's own random `organizationId`/`tenantId`.
- `adjustBalance`'s `verifyReferences` also requires a real `MasterdataUom` row for `txnUomId` to exist (independent of the same-UOM conversion-factor short-circuit). Added a `MasterdataUom` fixture row.

## Reconciliation Against the Supervisor-Grounded Guide (Real Defect Found and Fixed)

After this item's initial implementation and test pass (6/6 dedicated, first Playwright pass), this session discovered the supervisor's own grounded `06_AGENT_GUIDES/P4-003_EXECUTION.md` had already been pushed to `origin/main` concurrently (commit `182cc3c`) and was materially more detailed than this session's own preliminary guide. Reconciling against its explicit invariants surfaced one real, previously-untested gap:

**Real defect found and fixed:** invariant #3 ("release the accepted shared active-task ownership/lock for the completed task") was not implemented. `submitLocation`'s `LOCATION_VALIDATION -> COMPLETED` transition left the operator's `WmsOutboundTaskLock` row `ACTIVE` for its full 1-hour TTL, unlike `pick-task-service.ts`'s own `PickTask` completion path, which explicitly releases the lock (`activeLock.status = 'RELEASED'`) at both its `SHORT_PICKED` and `COMPLETED` terminal transitions. Without this fix, an operator who just completed a physical put-back would remain blocked from receiving any new PickTask/CrossDockPickTask/PutBackTask for up to an hour. Fixed by releasing the `ACTIVE` lock in the same transaction as completion, mirroring the exact existing `pick-task-service.ts` pattern (`put-back-task-service.ts` `submitLocation`, immediately after the `COMPLETED` transition, before the final flush).

Added in the same pass: a dedicated wrong-operator test (invariant #6, zero mutation on a foreign operator's submission attempt) and a real rollback-proof test for the completion path (invariant covered by `.ai/TESTING.md` §5 — `beforeCommitHook` forces a deterministic pre-commit failure after `adjustBalance`'s real write+flush; a fresh independent read confirms task status, `completedAt`, `validatedLocationId`, the `WmsInventoryMovement`/`WmsInventoryBalance` rows, and the task lock's `ACTIVE` status are all unchanged — the real write never committed). A lock-release assertion was also added to the existing valid-completion test.

Rerun after this fix: dedicated suite **8/8 PASSED** (was 6/6); full targeted regression **111/111 PASSED** (was 109/109, unchanged elsewhere); typecheck clean; rebuilt/redeployed Testing runtime; rendered Playwright acceptance re-run **5/5 PASSED** (P4-003 Journey E 1/1 + P4-002 4/4, unaffected by this backend-only fix).

## Completion Correction — Physical-Return Handoff Resolution

After the above reconciliation pass, the supervisor independently found and grounded a second, separate real defect via `06_AGENT_GUIDES/P4-003_COMPLETION_CORRECTION.md` (commit `fad23c5`), continuing the same item (no new Testing reset, no restart from an older base):

**Real defect found and fixed:** completing a `PutBackTask` recovered `Inventory` and marked the task `COMPLETED`, but never resolved the P4-001 physical-return handoff (`wms_outbound_physical_return_handoffs`) that had been protecting the picked quantity from ATP/allocation/source availability since cancellation. Three raw-SQL sites unconditionally subtracted every matching handoff's `quantity` regardless of whether its physical put-back had already completed: `atp-reservation-service.ts` `resolveQualifyingAtpSupply`, and `allocation-service.ts` `resolveAvailableEligibleStock` and `resolveLocationEligibleStock` (the latter's subquery appears twice, in both `SELECT` and `HAVING`). Consequently the exact quantity P4-003 had just recovered into ordinary stock could remain logically unavailable indefinitely after a successful physical put-back — contradicting P4 R8 / STEP 5 ("after `PutBackTask` completion, `Inventory` transitions `PICKED -> AVAILABLE`, becoming ordinary available stock").

**Fix:** additive, audit-preserving — added a nullable `resolved_at` column to `WmsOutboundPhysicalReturnHandoff` (`Migration20260906190000_wms_outbound_p4_003_handoff_resolution.ts`) rather than deleting/replacing the handoff row. `submitLocation` now resolves the task's exact source handoff (`resolvedAt = now`) inside the *same* transaction as the `LOCATION_VALIDATION -> COMPLETED` transition, the `wms_inventory` balance/movement write, and the shared task-lock release — atomic, and idempotent (`if (handoff && !handoff.resolvedAt)` guards against a duplicate/replayed completion re-resolving or re-releasing anything). All three query sites now add `AND h.resolved_at IS NULL`, so a resolved handoff's quantity stops being subtracted the instant it resolves, and `reconcilePendingHandoffs` now filters `resolvedAt: null` so it only ever treats genuinely unresolved handoffs as pending recovery work.

**Decisive proof added** (`p4-003-postgres.integration.test.ts`, tests 9-10, plus rollback test 8 extended):
- before completion, the handoff is unresolved and its exact quantity is subtracted from a real ledger-derived ATP figure (`getAvailableAtpForSku` against genuine `PUTAWAY_IN` movement stock seeded through a real POSTED ASN + Transport Unit, mirroring `p1-002`'s own fixture pattern);
- a rejected location leaves the handoff unresolved and the subtraction unchanged;
- valid completion atomically resolves the handoff and the exact recovered quantity becomes available (ATP rises by precisely the task quantity — demonstrated against a real gross-supply baseline, not a zero-clamped figure);
- a duplicate/idempotent re-submission after `COMPLETED` does not re-resolve the handoff (`resolvedAt` timestamp unchanged) and does not raise ATP again (no double release);
- real two-connection concurrency (same row-lock-serialization pattern as the earlier concurrency test) resolves the handoff exactly once, alongside exactly one `WmsInventoryMovement`;
- the rollback-proof test (`beforeCommitHook` forcing a deterministic pre-commit failure) now also asserts the handoff remains unresolved on a fresh independent read, alongside the already-covered task/Inventory/task-lock state.

**Rerun after this correction:** dedicated suite **10/10 PASSED** (was 8/8); combined targeted regression (P4-001/002/003, P3-003, P1-001/002/005/006, P2-002, FND-003) **133/133 PASSED**; additional allocation-service.ts caller regression (P1-004, P1-007, P3-001, P3-002) **60/62 PASSED** — the 2 failures are `p3-002-postgres.integration.test.ts`'s own pre-existing, unrelated `MetadataError` (missing `WmsOutboundPutBackTask` entity registration in that file's own isolated MikroORM instance; predates this branch, introduced by P4-002; documented with full root-cause evidence in `04_CURRENT_STATE/TEST_INFRA_GAPS.md`, not fixed here as outside this correction's authorized scope); typecheck clean; Mercato rebuilt and Testing runtime restarted from the exact correction candidate; rendered Playwright acceptance re-run **5/5 PASSED** (P4-003 Journey E + P4-002 4/4) with Scanner frozen, unmodified (no UI regression required — the correction is entirely backend query/transaction logic).

## Mandatory Regressions

Ran the exact targeted regression set P4-002's own accepted evidence used (P4-001, P3-003, P1-005, P1-006, P2-002, FND-003), plus P4-002/P4-003 and P1-001/P1-002 (also exercise `atp-reservation-service.ts`), together in one `--forceExit` run, after the completion correction:

| Suite | Result |
|---|---|
| P4-001 dedicated PostgreSQL | PASSED |
| P4-002 dedicated PostgreSQL | PASSED (17/17, unchanged) |
| P4-003 dedicated PostgreSQL | PASSED (10/10) |
| P3-003 dedicated PostgreSQL race suite | PASSED |
| P1-001 CustomerOrder/ATP intake | PASSED (`atp-reservation-service.ts` caller) |
| P1-002 ATP reservation queue/recalculation | PASSED (`resolveQualifyingAtpSupply` caller) |
| P1-005 PickTask generation/ordering/assignment/concurrency | PASSED (shared active-task guard) |
| P1-006 RF picking execution & P3-003 concurrency | PASSED |
| P2-002 CrossDockPickTask planning/assignment | PASSED (shared active-task guard) |
| FND-003 shared task-lock/warehouse-context | PASSED |

Combined: **10 suites, 133/133 tests PASSED**, 57.5s, zero failures.

Additionally ran every other `allocation-service.ts` caller not already in the set above (P1-004, P1-007, P3-001, P3-002), since the completion correction also changed `resolveAvailableEligibleStock`/`resolveLocationEligibleStock`: **60/62 PASSED** — the 2 failures are `p3-002-postgres.integration.test.ts`'s own pre-existing `MetadataError` (see `04_CURRENT_STATE/TEST_INFRA_GAPS.md`), confirmed unrelated to this diff.

No product behavior in P4-001, P4-002, P3-003, or the shared PickTask/CrossDockPickTask assignment paths was altered; this item adds a new `submitLocation` method, additive response-field extensions to existing read routes, and (via the completion correction) an additive `resolved_at` filter to three existing availability-calculation queries.

**Note on full-directory run:** An initial attempt to run the *entire* `wms_outbound/__tests__/` directory (37 suites, ~572 tests) as an extra-cautious superset hit two separate environment issues unrelated to this diff: (1) a pre-existing test (`p1-003-detail-api-postgres.integration.test.ts`) with a missing entity-metadata registration leaked two Supabase pooler connections that were never closed, hanging the Jest process for over an hour with zero new output after the actual tests had already finished (`--forceExit` was missing on that first attempt); (2) a second full-directory attempt with `--forceExit` added showed a different, non-deterministic failure pattern (`self-signed certificate in certificate chain`, `statement timeout`) consistent with connection-pool/resource pressure on the shared Testing DB from repeatedly running the whole directory serially, not a code defect. Neither failure mode touches code this item changed. This was abandoned in favor of the precedent-matched targeted regression above, which is unaffected by either issue and green.

## Build / Runtime

- `apps/mercato`: `NODE_OPTIONS=--max-old-space-size=6144 tsc --noEmit` — clean, zero errors.
- `apps/mercato`: `mercato generate` — succeeded, all generators completed.
- `apps/mercato`: `mercato db migrate` — `wms_outbound: 2 migrations applied` (this item's `validated_location_id` migration, plus one prior pending migration) then, after the completion correction, `wms_outbound: 1 migration applied` (`resolved_at`); all other modules `no pending migrations` throughout.
- Root `yarn build` — succeeded; Next.js 16.2.9 production build compiled and typechecked clean.
- Scanner: `npx tsc --noEmit` — clean, zero errors.
- Scanner: `npx expo export --platform web` (via `scanner-testing.service` `ExecStartPre`) — succeeded.
- Testing runtime rebuilt/restarted from these exact candidate revisions: `mercato-localhost.service` and `scanner-testing.service`, both confirmed `active (running)` post-restart, serving `https://devaxonic-test.info-start.com.pl` and `https://scanner.info-start.com.pl` respectively.

## Rendered UI Acceptance — Zero Route Mocks

Suite: `Devaxonic-scanner/e2e/p4-003-rendered-acceptance.spec.ts`
Result: **1/1 PASSED** against `https://scanner.info-start.com.pl`, with real seeded PostgreSQL fixtures (real operator via `bcrypt` + real warehouse assignment, real `MasterdataItem`/`MasterdataUom`, real `WmsWarehouseLocation` with `STORAGE` capability, real physical-return handoff, real `PutBackTask` row) and zero route/API mocking.

- **Journey E**: operator receives and starts the task (`IN_PROGRESS`); scans a non-existent location -> rejection banner rendered with reason, task confirmed still `IN_PROGRESS` in PostgreSQL, zero `wms_inventory_movements` rows for the item; scans the valid `STORAGE` location (no attempt limit, same session, same form) -> completion banner rendered, PostgreSQL confirms `PutBackTask COMPLETED` with `validated_location_id` set, exactly one `ADJUSTMENT_IN` movement, and `wms_inventory_balances.on_hand_quantity`/`available_quantity` both equal to the task's exact quantity at the validated location.

Regression: `Devaxonic-scanner/e2e/p4-002-rendered-acceptance.spec.ts` re-run against this exact build — **4/4 PASSED**, both suites together **5/5 PASSED**, re-confirmed a final time after the completion correction and its Mercato rebuild/redeploy (29.6s, Scanner unchanged/frozen). Journey A's assertion on the P4-002 `returns-in-progress-banner` placeholder (explicitly documented in that item's own evidence as "P4-003 destination-location validation... not implemented in this item... `LOCATION_VALIDATION`/`COMPLETED` remain schema-only vocabulary") was updated to assert the real `returns-location-form` it was staged to become; no DB/business assertion in that spec changed.

Evidence class: PLAYWRIGHT VERIFIED, not HUMAN VERIFIED.

## Explicit Exclusions Confirmed

- No attempt-count field exists anywhere in the schema or code; the `IN_PROGRESS <-> LOCATION_VALIDATION` loop is genuinely unbounded (P4 R7) — demonstrated directly by dedicated test #3 (five rejections, still completes).
- No automatic escalation to a Supervisor or any other role exists on repeated rejection.
- The System WMS recommendation (`proposedLocationId`, defaulted to `sourceLocationId` at P4-002 materialization) is never cleared by a rejection and remains available to the operator throughout the loop.
- Inbound Putaway business semantics were not imported: no `IN_PUTAWAY` status, no sector `TRANSIT` handling, no PutawayTask ownership model. Completion goes directly to ordinary `Inventory AVAILABLE` via the shared `wms_inventory` balance/movement primitives, unchanged from their existing implementation.
- P4-002's accepted FIFO/no-zone/single-active-task boundary is untouched; `assignNext`/`startTask` were not modified.
- No unrelated work: no Return Receipt, Warehouse Transfer, PickWave, or carrier/ERP changes. The completion correction's query changes are additive filter clauses on three existing, already-accepted availability calculations — no new business rule, no P4 state change, no FIFO/Inbound/Scanner-UX change.
- The two pre-existing, unrelated test-infra entity-registration gaps found while verifying this item (`p3-002`, `p1-003-detail-api`) are documented in `04_CURRENT_STATE/TEST_INFRA_GAPS.md` with exact root-cause evidence and are explicitly not fixed by this item, per its authorized scope.

## Completion Statement

Item 31/37 (P4-003) implementation is pushed from the accepted P4-002 lineage in both changed product repos, at final Mercato candidate `6ddec6870` and Scanner candidate `a2759a2`; dedicated PostgreSQL acceptance (10/10) including real concurrency, idempotency, wrong-operator rejection, rollback, and the physical-return-handoff resolution/availability-release proof is green; the precedent-matched targeted regression set (133/133 across 10 suites, plus 60/62 on the remaining `allocation-service.ts` callers with the 2 failures confirmed pre-existing/unrelated) is green; native build/typecheck/generate/runtime checks are green; rendered Scanner Playwright acceptance for this item (1/1) plus the re-verified P4-002 suite (4/4, one legitimate locator update) are PLAYWRIGHT VERIFIED with zero route mocks; this evidence is pushed to `WMS_Outbound/main`.

This is executor `COMPLETE`, not Owner Acceptance. Supervisor independently verifies remote Git/diff/tests/evidence before advancing the Task Catalog count.

Executor does not proceed to any later item.
