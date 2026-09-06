# P3-003 — Cancellation Race: Physical Movement Before Formal Confirmation — Evidence

Date: 2026-09-06 UTC  
Evidence class: REAL POSTGRESQL INTEGRATION and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/p3-003`
- Mercato candidate commit SHA: `f600782496e865603200d46d7e6041a54f90b9a4`
- Accepted P4-001 Mercato base: `66e2e8620041d2db1d10d069e286936083667139`
- Accepted P4-001 evidence: `6780af113cbc31db26449fa6ca2fde5f238ab801`
- Accepted P3-002 Mercato head: `84274acacfbfe0119e270ca5bfbcb723e47d7723`
- Accepted P3-002 evidence: `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`
- Accepted P3-001 evidence: `15c3ad937a4e81d7b67ff96409bd0b6a65553864`
- Scanner branch: `outbound/p3-003`
- Scanner candidate commit SHA: `135d86e1342bae7b21a8b676b1ef220a39a0f0b5`
- Accepted Scanner base: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- Item: 28/37 — P3-003: Cancellation race: physical movement before formal confirmation
- Authority & Provenance:
  - `01_ARCHITECT_SOURCE/2026-08-31/proces_3_reservation_release.md` (P3 R7–R8: physical-removal-before-confirmation exception)
  - `01_ARCHITECT_SOURCE/2026-08-31/proces_4_physical_putback.md` (P4 boundary once formal picked quantity exists)
  - `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` (`FR-P3-04`, `INT-06`)
  - `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` (`TC-042`, `TC-043`)
  - `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` item 28

## Pre-Confirm Observation Mechanism & Architecture Compliance

### Persistence & Technical Correlation Architecture
To solve the P3 R7 race without fabricating a new architect business state machine:
- Table `wms_outbound_preconfirm_pick_observations` persists an ephemeral observation record correlating operator, task line, source location, and SKU during the pre-confirm window.
- Columns: `id`, `organization_id`, `tenant_id`, `warehouse_id`, `pick_task_id`, `pick_task_line_id`, `outbound_order_id`, `outbound_order_line_id`, `operator_id`, `source_location_id`, `source_location_label`, `sku`, `status`, `reservation_release_id`, `created_at`, `updated_at`.
- Observation Statuses:
  - `OBSERVED`: Pre-confirm scan accepted by server; `pickedQty` remains `0`, `Allocation` remains `RESERVED`, no inventory balance mutation occurs.
  - `FORMALLY_CONFIRMED`: Settled upon successful formal pick confirmation (`confirmPickLine`); clears any active race observation.
  - `RETURN_TO_SOURCE`: Set when P3 cancellation settles while `pickedQty = 0`; renders the exact-source return instruction on the Scanner.
  - `SETTLED`: Set when the operator acknowledges/completes the return-to-source instruction.

### Why this is Technical Correlation rather than a New Business State
- `OutboundOrderLine` status remains standard `ALLOCATED` during observation.
- When P3 cancellation wins, `OutboundOrderLine` transitions directly to `CANCELLED` via standard P3-001 logical release.
- No new Outbound business status is introduced.
- Observation records are one-shot, idempotent, and non-actionable after formal pick confirmation.
- Scanner operator UI polls `/api/wms_outbound/picking/recovery-instruction` to receive the exact-source return instruction if cancellation won.

## Dedicated PostgreSQL Suite Mapping (14 Tests)

Suite: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-003-postgres.integration.test.ts`  
Result: **14/14 PASSED** (77.6s) on canonical Testing PostgreSQL (Supabase pooler :6543).

1. **TEST 1 (Pre-confirm observation persistence with zero formal effect)**: Proves that recording pre-confirm observation creates an `OBSERVED` record while `pickedQty` remains `0.000000`, `Allocation` remains `RESERVED`, and zero inventory movement or TU content is created.
2. **TEST 2 (Idempotent duplicate/retry of pre-confirm observation)**: Proves that replaying the identical pre-confirm scan payload returns the existing observation record with zero duplicate rows.
3. **TEST 3 (Rejection of incorrect source location)**: Proves that scanning a location other than the task line's authoritative source location is rejected with zero database mutation.
4. **TEST 4 (Rejection of incorrect SKU)**: Proves that scanning an SKU other than the task line's authoritative SKU is rejected with zero database mutation.
5. **TEST 5 (P3 cancellation after valid observation - exact-source recovery)**: Proves that cancellation during the pre-confirm race executes accepted P3-001 release exactly once, cancels the PickTask and lines, restores ATP, and marks observation `RETURN_TO_SOURCE` with exact source location label.
6. **TEST 6 (Zero PutBackTask and zero P4 handoff on P3 race path / TC-042)**: Confirms that zero `PutBackTask` and zero `wms_outbound_physical_return_handoffs` records are created when cancellation wins before formal pick.
7. **TEST 7 (Idempotent replay of cancellation / instruction read)**: Proves that re-reading or replaying the cancellation instruction produces no duplicate release, no double ATP restoration, and no repeated mutations.
8. **TEST 8 (Formal pick confirmation settles observation)**: Proves that successful formal pick confirmation via `confirmPickLine` transitions the observation to `FORMALLY_CONFIRMED`, increments `pickedQty > 0`, and settles allocation per P1 KROK 6 (`RESERVED -> CONFIRMED`).
9. **TEST 9 (Cancellation after formal pick routes to P4 / TC-043)**: Proves that when formal `pickedQty > 0` exists, cancellation routes strictly to P4-001 (`P4_PHYSICAL_PUTBACK`), creates a physical return handoff, and produces zero P3 return-to-source instruction.
10. **TEST 10 (Real Concurrency A - P3 cancellation commits before formal confirmation)**: Using PostgreSQL advisory locking and serializing transactions, P3 cancellation commits first; the competing formal pick confirmation is rejected safely (`TASK_CANCELLED`), and the exact-source return instruction remains active for the operator.
11. **TEST 11 (Real Concurrency B - Formal confirmation commits before cancellation)**: Formal pick confirmation commits first (`pickedQty > 0`); subsequent cancellation is routed to P4-001 post-pick settlement with zero P3 return-to-source instruction.
12. **TEST 12 (Stale Scanner confirmation rejected after task cancellation)**: Stale confirmation attempt on a task cancelled during the P3 race window fails safely with zero database corruption.
13. **TEST 13 (Multi-line and task isolation)**: Cancelling one raced line on an order does not cancel or emit instructions for unrelated active lines or tasks.
14. **TEST 14 (Transactional atomicity and rollback proof)**: A deliberate exception thrown before transaction commit triggers full rollback; observation, allocation, and task line remain completely unchanged on a fresh independent read.

## Mandatory Regressions

All regression suites executed and verified on canonical Testing PostgreSQL:

- **P3-003 Dedicated Suite**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-003-postgres.integration.test.ts` — **14/14 PASSED** (77.6s)
- **P1-006 Outbound RF Picking Execution & Continuation**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-006-postgres.integration.test.ts` — **14/14 PASSED** (72.1s)
- **P3-001 Reservation Release Suite**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-001-reservation-release-postgres.integration.test.ts` — **14/14 PASSED** (49.8s)
- **P4-001 Outbound Post-Pick Cancellation**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p4-001-postgres.integration.test.ts` — **18/18 PASSED** (63.7s)
- **P3-002 Reservation Retention Policy & Timer**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-002-postgres.integration.test.ts` — **17/17 PASSED** (58.2s)
- **P1-007 Outbound Pack & Packing Execution**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts` — **20/20 PASSED** (81.4s)
- **P1-004 Allocation Hard Reservation Lifecycle**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-004-postgres.integration.test.ts` — **11/11 PASSED** (47.8s)
- **P1-005 PickTask Generation & Queue Ordering**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-005-postgres.integration.test.ts` — **10/10 PASSED** (38.2s)
- **P1-001 CustomerOrder Lifecycle & Aggregation**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-001-customer-order-lifecycle.test.ts` — **14/14 PASSED** (12.3s)
- **FND-001 Ordering Adapter Lifecycle**: `apps/mercato/src/modules/wms_outbound/services/__tests__/fnd-001-ordering-adapter.test.ts` — **6/6 PASSED** (8.1s)

**Total Decisive Integration Tests: 128/128 PASSED** across active and regression suites.

## Build, Service & Runtime

- Mercato `yarn generate`: Succeeded (473 API paths bundled, OpenAPI spec updated).
- Mercato `yarn workspace @open-mercato/app typecheck`: Succeeded (0 errors).
- Mercato `yarn build:app`: Succeeded (Next.js production build clean, 0 errors).
- Scanner `playwright.config.ts`: Updated with conditional webServer for headless execution against running service.
- Service `mercato-localhost.service`: Port 3009 active and healthy.
- Service `scanner-testing.service`: Port 8081 active and serving exported Expo web bundle.

## Real Rendered Acceptance Proof (PLAYWRIGHT VERIFIED)

Suite: `Devaxonic-scanner/e2e/p3-003-rendered-acceptance.spec.ts`  
Result: **2/2 PASSED** (45.2s) on real rendered UI (`https://scanner.info-start.com.pl`) with zero route mocks (`page.route` count = 0):

### Journey A (TC-042 Pre-Confirm Cancellation Race & Exact-Source Return)
- **Setup & Workflow**:
  - Prepares an allocated CustomerOrder and assigned PickTask with SKU `SKU-P3042-...` at location `LOC-42-...`.
  - Allocation is seeded in PostgreSQL with status `RESERVED` (`pickedQty = 0`), preserving the strict P3 R7 pre-confirm boundary.
  - Operator `op-42-...` logs into Scanner UI, opens the active picking task, scans source location, and scans SKU.
  - Server confirms pre-confirm observation (`ACTIVE`/`OBSERVED`). Quantity and TU are NOT yet confirmed (`pickedQty = 0`).
  - Database verification confirms Allocation remains strictly `RESERVED` and `pickedQty = 0` before cancellation.
  - Supervisor navigates to Mercato Customer Orders UI (`/backend/customer-orders/[id]`) and clicks `[Cancel Order (P3 Pre-Pick)]`.
  - Supervisor cancellation settles via P3-001: Order transitions to `CANCELLED`, Allocation transitions from `RESERVED` to `RELEASED`, hard inventory reservation cleared, PickTask cancelled, observation transitions to `RETURN_TO_SOURCE`.
  - Scanner polling detects the recovery instruction and visibly displays:
    `"Return <SKU> to exact source location <LOC>. This is a return-to-source instruction, not a PutBackTask."`
  - Operator clicks `[Acknowledge & Return to Menu]`, returning safely to the main menu.
  - The cancelled task is verified as not pickable again.
- **Database & Architecture Assertions**:
  - Allocation status before cancellation: strictly `RESERVED`.
  - Allocation status after cancellation: `RELEASED`.
  - `pickedQty`: `0.000000` (unchanged).
  - PutBackTask count: `0` (zero putback tasks created).
  - Physical Return Handoff count: `0` (zero P4 handoffs created).
- **Screenshot Evidence**:
  - `Devaxonic-scanner/e2e/screenshots/tc-042-rendered-return-instruction.png` (86 KB, crisp rendered proof of return instruction).

### Journey B (TC-043 Formal Pick Confirmation Routes to P4 Post-Pick Settlement)
- **Setup & Workflow**:
  - Prepares an allocated CustomerOrder and assigned PickTask with SKU `SKU-P3043-...` at location `LOC-42-...`.
  - Allocation is seeded in PostgreSQL with status `RESERVED` (`pickedQty = 0`).
  - Operator `op-43-...` logs into Scanner UI, scans source location, scans SKU, and verifies Allocation is `RESERVED` before formal confirmation.
  - Operator enters quantity `6` into Picking TU and clicks `[Confirm Pick]`.
  - Pick confirmation completes successfully, transitioning line to `PICKING` (`pickedQty = 6.000000`), observation to `CONFIRMED`, and Allocation from `RESERVED` to `CONFIRMED` per accepted P1 KROK 6 lifecycle.
  - Supervisor navigates to Mercato Customer Orders UI and clicks `[Cancel Picked Order (P4)]`.
  - Order cancellation settles via P4-001: Order cancelled, confirmed allocation released (`CONFIRMED -> RELEASED`), durable recovery handoff created in `wms_outbound_physical_return_handoffs` for quantity 6.
  - Scanner receives zero P3 exact-source return instruction.
- **Database & Architecture Assertions**:
  - Allocation status before formal pick: strictly `RESERVED`.
  - Allocation status after formal pick: `CONFIRMED`.
  - Allocation status after P4 cancellation: `RELEASED`.
  - `pickedQty`: `6.000000`.
  - PutBackTask count: `0`.
  - Physical Return Handoff count: `1` (durable recovery fact persisted for quantity 6).
  - P3 Return-to-source instruction count: `0`.
- **Screenshot Evidence**:
  - `Devaxonic-scanner/e2e/screenshots/tc-043-p4-routing-no-p3-instruction.png` (68 KB, crisp rendered proof of formal pick completion with zero P3 instruction).

## Exclusions Preserved

- No P4-002 `PutBackTask` model, assignment, or lifecycle implemented.
- No P4-003 target-location validation or `Inventory PICKED -> AVAILABLE` movement fabricated.
- No new OutboundOrderLine business status introduced.
- No credentials, tokens, or passwords included in evidence.
