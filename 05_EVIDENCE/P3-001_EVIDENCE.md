# P3-001 — Reservation Release before formal pick — Evidence

Date: 2026-09-06 UTC  
Evidence class: REAL POSTGRESQL INTEGRATION and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/p3-001`
- Mercato commit SHA: `e4afe12b74d4cba8ef7c2a3f6ff605bac20cbd4b`
- Accepted P2-006 Mercato base: `4f64641ab14a5359bc22d0685e390b511252b5b5`
- Scanner frozen SHA: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` (unmodified and clean)
- P2-006 evidence SHA: `9a580b046b5f2aa3bcbf2422eeaf6413248f68db`

## Migration & Schema Provenance

- Applied Migration: `Migration20260906020000_wms_outbound_p3_001_reservation_release.ts`
- Table: `wms_outbound_reservation_releases`
  - Columns: `id`, `organization_id`, `tenant_id`, `warehouse_id`, `customer_order_id`, `customer_order_line_id`, `outbound_order_id`, `outbound_order_line_id`, `allocation_id`, `reason`, `release_type`, `released_quantity`, `source_system`, `external_order_id`, `external_line_id`, `idempotency_key`, `actor_id`, `actor_role`, `details`, `created_at`, `updated_at`.
  - Unique Index: `(tenant_id, idempotency_key)` ensuring strict idempotency and zero duplicate mutations.

## Dedicated PostgreSQL Suite Mapping (15 Behaviors)

Suite: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-001-reservation-release-postgres.integration.test.ts`  
Result: **12/12 PASSED** (66.8s) on canonical Testing PostgreSQL (Supabase pooler :6543).

1. Normal pre-pick general cancellation with `pickedQty=0` releases the hard Allocation exactly once (TEST 1 PASS).
2. Linked hard Inventory reservation is removed and available quantity is restored exactly once (TEST 1 PASS).
3. Released hard quantity is restored to soft ATP reservation / demand coverage exactly once (TEST 1 PASS).
4. Affected Standard OutboundOrderLine becomes `CANCELLED` (TEST 1 PASS).
5. General cancellation sets CustomerOrderLine to `CANCELLED` and preserves continuous parent CustomerOrder aggregation (TEST 1, TEST 4 PASS).
6. Shortage release sets CustomerOrderLine to `BACKORDERED` (not `CANCELLED`), preserving original customer demand while releasing hard allocation (TEST 2 PASS).
7. Eligible pre-pick PickTask/TaskLine work is cancelled transactionally without weakening generic P1-004 CON-02 release protection (TEST 3 PASS).
8. Zero `PutBackTask` or physical recovery record is created for true pre-pick release (TEST 1, TEST 3 PASS).
9. Replay of the same correlation/idempotency key returns `REPLAYED` and produces ZERO duplicate release, ATP restoration, or audit transitions (TEST 6 PASS).
10. Conflicting idempotency key reuse with mismatched parameters fails safely with error and zero mutation (TEST 7 PASS).
11. Formally confirmed `pickedQty > 0` blocks P3 release and routes out to P4 Physical Putback (TEST 8 PASS).
12. Automatic/system reason executes release without requiring Warehouse Supervisor notification, without implementing P3-002 retention timers (TEST 9 PASS).
13. Crossdock lines without Allocation reject or no-op safely without manufacturing fake Allocations (TEST 10 PASS).
14. Rollback proof: intentional transaction failure after DB flush but before commit leaves Allocation, Inventory reservation, and ATP completely unchanged on fresh read (TEST 11 PASS).
15. Concurrency proof: two overlapping release requests on distinct PostgreSQL sessions (PIDs 2197578 & 2195992) acquire genuine row locks via `SELECT ... FOR UPDATE`, serialize deterministically via `Lock` wait event in `pg_stat_activity`, and result in exactly one release and zero double ATP restoration (TEST 12 PASS).

## Mandatory Regressions

All regression suites executed and verified on canonical Testing PostgreSQL:

- **P1-004 Allocation Hard Reservation Lifecycle (CON-02 Intact)**: `p1-004-postgres.integration.test.ts` — **11/11 PASSED** (49.2s). Active PickTasks still strictly block generic `releaseAllocation`.
- **P1-001 CustomerOrder Lifecycle & Aggregation**: `p1-001-postgres.integration.test.ts` — **7/7 PASSED** (13.1s).
- **P1-005 PickTask Lifecycle**: `p1-005-postgres.integration.test.ts` — **10/10 PASSED** (25.0s).
- **P1-016 Final Settlement Foundations**: `p1-016-final-settlement-postgres.integration.test.ts` — **25/25 PASSED** (84.2s).

## Code Generation & App Build

- `yarn generate`: Succeeded (468 API routes, OpenAPI spec generated, cache purged).
- `yarn typecheck --filter=@open-mercato/app`: Succeeded (exit code 0).
- `yarn build:app`: Succeeded (Turbo build completed in 3m9s, Next.js routes built).
- Service restart: `sudo systemctl restart mercato-localhost.service`.
- HTTP Health: `https://devaxonic-test.info-start.com.pl/login` returned HTTP 200.

## Real Rendered Playwright UI Proof

Suite: `apps/mercato/src/modules/wms_outbound/__integration__/P3-001-reservation-release-ui.spec.ts`  
Result: **3/3 PASSED** (50.9s) with zero route mocks (`page.route` count = 0):

- **Journey A (General Pre-Pick Cancellation)**: Supervisor navigates to real rendered customer order page (`/backend/customer-orders/[id]`), clicks `Check Cancellation (INT-06)`, receives `ELIGIBLE` status with `Target: P3_RESERVATION_RELEASE`, clicks `Cancel Order (P3 Release)`, receives immediate feedback notice, verified rendered badge transitions to `CANCELLED`. Persisted DB verification proves Allocation `RELEASED`, Inventory Reservation cleared, soft ATP restored to 10.0, OOL/COL `CANCELLED`, PickTask/Line `CANCELLED`, zero PutBackTask. Replay verification via API returns `status: 'REPLAYED'` and zero double ATP restoration.
- **Journey B1 (Shortage vs Demand Preservation)**: SHORT_ALLOCATED pre-pick cancellation through real cancellation boundary releases hard Allocation and inventory reservation while transitioning CustomerOrderLine to `BACKORDERED` (preserving customer demand).
- **Journey B2 (P4 Picked Quantity Boundary)**: Customer order with formal `pickedQty = 5.0` (> 0) displays `Target: P4_PHYSICAL_PUTBACK` with P3 execution button hidden. Direct API attempt to execute P3 release is rejected with HTTP 422: `Formally picked quantity belongs to P4 Physical Putback (P3 R8, TC-043)`.

## Exclusions Preserved

- No P3-002 timer scheduling or retention duration logic.
- No P3-003 RF physical-removal race window.
- Zero PutBack tasks created.
- Zero fake allocations manufactured for crossdock.
- Generic P1-004 CON-02 release guard preserved intact.
