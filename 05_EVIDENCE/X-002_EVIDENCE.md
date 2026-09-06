# X-002 — Integration Correlation, Observability and Operational Recovery — Evidence

Date: 2026-09-06 UTC
Evidence class: REAL POSTGRESQL INTEGRATION, REAL CONCURRENCY (approved Testing Supabase `DevAxonic_Platform`; genuine overlapping-transaction proof for the one hardened boundary). This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/x-002`
- Mercato final commit SHA: `4a89a95aad42c476ac206b53fe8ff67f3c8021a9` (acceptance correction of `c04bb4f836198d5385ccb21bbf53c371c3bd64ae` — see "Acceptance Correction" below)
- Mercato base: `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` (X-001 FINAL PASS / Owner Accepted)
- Scanner: untouched, `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302` (frozen; no RF user-flow defect was found in this item's scope)
- Item: 33/37 — X-002: Integration correlation, observability and operational recovery
- Execution guide: `06_AGENT_GUIDES/X-002_EXECUTION.md` @ `0fff59ebfeddc7104e6396c3882d8defeaf8145e`

## Scope and Method

X-002 is an audit-and-harden item. Each of `INT-01..06` was audited against its exact current owning service and exact current test file before any change was made:

- `cross-dock-eligibility-service.ts` (INT-01), owned by `p2-001-crossdock-eligibility-postgres.integration.test.ts`
- `cross-dock-execution-service.ts` (INT-02), owned by `p2-003-crossdock-execution-postgres.integration.test.ts` and `p2-004-crossdock-recovery-postgres.integration.test.ts`
- `cross-dock-gr-gate-service.ts` (INT-03), owned by `p2-005-crossdock-gr-gate-postgres.integration.test.ts`
- `shipment-posting-service.ts` + `erp-shipment-posting-adapter.ts` (INT-04/05), owned by `p1-014-erp-posting-postgres.integration.test.ts`
- `ordering-adapter-service.ts` (INT-06), owned by `fnd-001-ordering-adapter.test.ts`, `p3-002-postgres.integration.test.ts`, `p3-003-postgres.integration.test.ts`, `p4-001-postgres.integration.test.ts`

No boundary was refactored for uniformity. Only INT-02 had a real gap; the other five were already decisive and were left unchanged.

## INT-01..06 Matrix

| # | Authority | Direction / owner | Correlation identity | Durable fact | Idempotency/replay | Safe diagnostics | Owning tests | Code changed |
|---|---|---|---|---|---|---|---|---|
| INT-01 | P2 STEP 1 | Inbound → Outbound; Inbound owns TU qualification | `sourceInboundTuId` + `itemId` + `receiptCorrelation` | `WmsOutboundCrossDockBinding` (ASN-declared qty, source TU/item, receipt correlation) | Replay via `idempotencyKey:*` prefix returns existing bindings, zero new rows (`p2-001` test "TC-092/103") | Rejection messages name only ELEMENTARY/IN_CROSS_DOCK/scope violations, no internals | `p2-001-crossdock-eligibility-postgres.integration.test.ts` (10/10) | No |
| INT-02 | P2 STEP 3 | Outbound → Inbound; Inbound derives residual | `sourceInboundTuId` | `WmsOutboundCrossDockFinalization`; explicit `toInboundCrossDockSettlementContract()` mapping exposes exactly `sourceInboundTuId` + `confirmedQty` + `damagedQty` — `residualQty` stays a real persisted column but is asserted absent from the contract | **Gap found and fixed** — see below | Quantity-conservation violation throws a plain domain error, no internals | `p2-003-crossdock-execution-postgres.integration.test.ts` (8/8), `p2-004-crossdock-recovery-postgres.integration.test.ts` (18/18, incl. new decisive concurrency test and new contract test) | **Yes** |
| INT-03 | P2 STEP 4 | Inbound → Outbound (GR result); GR retry stays Inbound-owned | `sourceInboundTU` + `settlementSource = CROSSDOCK` | `WmsOutboundCrossDockGrResult` | Idempotency-key replay returns the existing result with `tasksUpdated: 0`; wrong source/settlement source is a zero-mutation no-op (`p2-005` tests 4–8) | Outcome is `matched`/`replayed`/`reason` only | `p2-005-crossdock-gr-gate-postgres.integration.test.ts` (19/19) | No |
| INT-04/05 | current P1 STEP 11A | Outbound → ERP; retry is a separate Supervisor decision | Shipment `correlationId` (`SHIP-POST-<shipmentNumber>`) + posting `idempotencyKey` | `WmsOutboundShipmentPosting` + append-only `WmsOutboundShipmentPostingAttempt` rows | Duplicate/in-flight calls are exactly-once (`p1-014` test 15); genuine overlapping-transaction proof (`p1-014` test 14, `pg_blocking_pids`) | `POSTING_ERROR` carries `errorCategory`/`errorCode`/`errorMessage`/`errorDetails` only; technical timeout never becomes a fabricated business rejection | `p1-014-erp-posting-postgres.integration.test.ts` (18/18) | No |
| INT-06 | P3 STEP 1 + P4 STEP 1 | Ordering system → Outbound; routes by formally confirmed picked quantity | `sourceSystem` + `externalOrderId` + `externalLineId` + request `idempotencyKey` | `WmsOutboundReservationRelease` (both P3 pre-pick and P4 post-pick paths already persist `sourceSystem`/`externalOrderId`/`externalLineId`/`idempotencyKey`) | Idempotency-key collision on a different order/line/allocation is rejected; P3-003 race re-evaluation preserved (`ordering-adapter-service.ts::executeCancellation` catch path) | Rejection reasons are plain domain strings (`blockingReason`) | `fnd-001-ordering-adapter.test.ts` (6/6), `p3-002-postgres.integration.test.ts` (17/17, exercises `executeCancellation`), `p3-003-postgres.integration.test.ts` (14/14, incl. the accepted race tests 10/11), `p4-001-postgres.integration.test.ts` (18/18) | No |

## INT-02 — Gap Found and Fixed

**Gap:** `cross-dock-execution-service.ts::checkAndFinalizeSourceTu` read the task/finalization snapshot (`allTasks`, `existing`) **before** taking the `PESSIMISTIC_WRITE` lock on the source `WmsTransportUnit` row. When a source TU's last two remaining `WmsOutboundCrossDockPickTask` rows complete in two genuinely overlapping transactions, each transaction's own read of the *other* task's status reflects only what that other transaction has already **committed** — under READ COMMITTED, an in-flight (not-yet-committed) sibling completion is invisible. Both transactions could therefore each see the sibling task as still active and both return `null` (skip finalization) — silently losing the durable `confirmedQty`/`damagedQty`/`residualQty` fact that INT-02 requires to exist at all after both commit.

**Fix:** move the source TU's `PESSIMISTIC_WRITE` lock acquisition to the first statement in `checkAndFinalizeSourceTu`, before the task/finalization reads. This serializes the two completions on the source TU row: whichever transaction acquires the row lock second is forced to wait for the first to commit, then re-reads a **fresh, fully-committed** task/finalization snapshot — guaranteeing exactly one finalization is created with the correct combined totals, regardless of which of the two overlapping completions physically wins the row lock first. The change is additive to the existing code path (same reads, same writes, only the lock's position moved) and does not alter any existing accepted `p2-003`/`p2-004` assertion.

**Decisive test:** `p2-004-crossdock-recovery-postgres.integration.test.ts` test 17 — "genuine PostgreSQL concurrency of the last two tasks completing for one source TU serializes on the source-TU lock and produces exactly one durable finalization with the combined confirmed quantity":
- Two independent `WmsOutboundCrossDockPickTask` rows, sharing only the source TU (fully independent customer orders/lines/outbound orders/lines/bindings — so the source-TU row is the *only* lock the two completions can contend on).
- Transaction A (`serviceA.complete(task1)`) is held open via an `onSourceTuLocked` instrumentation hook (mirrors the existing `onTransactionStarted`/`onSerializationAcquired`/`onPhase1Locked` seams already used by `cross-dock-eligibility-service.ts` and `shipment-posting-service.ts`) once it has genuinely acquired the source-TU row lock.
- Transaction B (`serviceB.complete(task2)`) is launched concurrently on an independent `EntityManager`/connection.
- A is released; both complete; fresh independent read proves exactly one `WmsOutboundCrossDockFinalization` row for the source, with `confirmedQty: '10.000000'` (both tasks' 5.000000 combined), `damagedQty: '0.000000'`, `residualQty: '0.000000'`, `status: 'CROSS_DOCKED'`, and the source TU's `processStatus` at `CROSS_DOCKED`.

## Acceptance Correction — Real PostgreSQL Blocking Proof and Explicit Contract Assertion

Supervisor review of the original test 17 required two corrections before FINAL PASS, both closed in this same item on the same branch:

**1. Real concurrency proof (was timing-only).** The original test 17 proved contention only via "B's promise has not settled yet" — a timing-only signal, not decisive PostgreSQL-side evidence. Corrected: the test now captures both transactions' real backend PIDs and, while A holds the source-TU row lock, queries `pg_stat_activity`/`pg_blocking_pids()` from an independent observer connection until it finds a real backend pid genuinely blocked by A's pid, asserting `wait_event_type = 'Lock'` and that the blocked pid is distinct from A's. A's held lock is now released in a `finally` block so a failed blocking assertion can never leave an open transaction hanging the suite's cleanup.

Fixing this decisively surfaced a real bug in the `onSourceTuLocked` instrumentation seam itself: it captured the backend pid via `tx.getConnection().execute('SELECT pg_backend_pid()...')`, which does **not** reliably resolve through the same pinned transactional connection that `tx.findOneOrFail(...)` used to take the row lock — the captured pid consistently did not match either of the two real, independently-observable overlapping connections in `pg_stat_activity`. Switched to `tx.execute('SELECT pg_backend_pid()...')`, matching the already-proven, already-accepted pattern in `shipment-posting-service.ts`'s `onPhase1Locked` hook (CON-05, X-001 evidence). After the fix, the captured pid correctly and reproducibly matches the real backend holding the lock; the already-accepted `p1-014` CON-05 concurrency test was independently re-verified still green throughout this investigation, confirming the bug was specific to this new instrumentation call, not an environment regression.

**2. Explicit external contract assertion (was implicit).** Added `toInboundCrossDockSettlementContract()` — an explicit, additive mapping from the internal `WmsOutboundCrossDockFinalization` row to the exact Outbound → Inbound settlement contract. New test 18 — "the Outbound -> Inbound settlement contract exposes exactly source correlation + confirmedQty + damagedQty; residualQty is an internal-only reconciliation fact that Inbound derives itself" — proves:
- the contract's key set is exactly `{sourceInboundTuId, confirmedQty, damagedQty}` (`Object.keys(...).sort()` equality check);
- `residualQty` is absent from the contract object (`'residualQty' in contract === false`);
- the internal persisted `WmsOutboundCrossDockFinalization.residualQty` column is unchanged/not removed;
- Inbound can derive residual itself: `declaredQty - confirmedQty - damagedQty` (computed independently in the test from only what Inbound already knows plus what the contract actually transmits) equals the internal `residualQty` exactly.

## Exact Test Commands and Results (final Mercato head `4a89a95a`)

All commands run with the canonical Testing DB environment sourced in the same shell (`.ai/TESTING.md` §2):

```bash
set -a && source /etc/mercato-localhost.env && set +a
npx jest --config jest.config.cjs --testPathPatterns "p2-00[1-6].*postgres" --forceExit
# Test Suites: 6 passed, 6 total
# Tests:       99 passed, 99 total

npx jest --config jest.config.cjs --testPathPatterns "p1-014|p3-002|p3-003|p4-001|fnd-001" --forceExit
# Test Suites: 6 passed, 6 total
# Tests:       80 passed, 80 total

npx jest --config jest.config.cjs src/modules/wms_outbound/services/__tests__/p2-004-crossdock-recovery-postgres.integration.test.ts --forceExit
# Test Suites: 1 passed, 1 total
# Tests:       18 passed, 18 total   (test 17 rerun 3x standalone for flakiness after the pid fix; all green)
```

`npx tsc --noEmit` (apps/mercato): clean, no errors.

## Regression Scope

Only `cross-dock-execution-service.ts` (INT-02) changed product code (plus its owning test file). Directly affected regressions rerun: all six P2 crossdock suites (`p2-001`..`p2-006`, 99/99), P1 ERP posting (`p1-014`, 18/18, included above), and P3/P4 cancellation (`p3-002`, `p3-003`, `p4-001`, `fnd-001`, included above). No shared `wms_orchestration` implementation/schema was touched, so no broader Inbound regression sweep was required per the guide.

No new Playwright evidence was added: no Mercato/Scanner user-visible behavior changed (the fix and its test are backend-only, exercising the service layer directly against real PostgreSQL). Scanner remains frozen — no real RF user-flow defect was found or required a change.

## Traceability Note

`INT-04`/`INT-05` current authority is P1 **STEP 11A** (ERP Shipment posting), not the stale derived `STEP 13` reference that appears in older traceability indexes; STEP 13 is physical dispatch/final settlement. This item did not implement any STEP 13 dispatch semantics.

## Boundary Statement

This item did not move GR retry ownership into Outbound (INT-03 remains a pure consumer of the GR gate result), did not create automatic ERP retry (INT-04/05 retry remains an explicit Supervisor action via `retryPosting`), and did not invent any external ERP/OMS endpoint, credential, or new generic integration bus. INT-06's accepted P3-003 race-handling re-evaluation path in `ordering-adapter-service.ts::executeCancellation` was preserved unchanged.

## Completion Statement

All six `INT-01..06` have decisive final-head executable contract/correlation evidence. The one real gap found (INT-02 duplicate/lost-finalization race) is fixed and covered by a genuine overlapping-PostgreSQL-transaction test. Directly affected regressions and typecheck are green. Mercato `outbound/x-002` and this evidence are pushed. Scanner remains frozen. No post-X-002 SSL maintenance and no `ACC-001` work has started.

This is Supervisor-prepared evidence, not Supervisor FINAL PASS or Owner Acceptance.
