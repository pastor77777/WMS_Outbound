# P4-001 — Outbound Post-Pick Cancellation and Logical Settlement — Evidence

Date: 2026-09-06 UTC  
Evidence class: REAL POSTGRESQL INTEGRATION and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/p4-001`
- Mercato candidate commit SHA: `66e2e8620041d2db1d10d069e286936083667139`
- Scanner frozen SHA: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` (confirmed frozen and unmodified; working tree clean)
- Accepted P3-002 base SHA: `84274acacfbfe0119e270ca5bfbcb723e47d7723`
- Accepted P3-002 evidence ref: `05_EVIDENCE/P3-002_EVIDENCE.md`
- Item: 29/37 — P4-001: Outbound post-pick cancellation and logical settlement.
- Authority & Provenance:
  - `proces_4_physical_putback.md` (P4 R1–R4, P4 Step 1: Decision on cancellation of picked / packed position)
  - `wymagania_outbound.md` (`FR-P4-01`, `FR-P4-02`, `INT-06`)
  - `scenariusze_testowe_outbound.md` (`TC-050`, `TC-051`, `TC-053`)
  - `model_stanow_outbound_EN.md` §3 (OutboundOrder PICKED/PACKED -> CANCELLED), §4 (OutboundOrderLine PICKING/SHORT_PICKED/PICKED/PACKED -> CANCELLED), §5 (Allocation CONFIRMED -> RELEASED)
  - `state_event_transitions.csv`
  - `06_AGENT_GUIDES/P4-001_EXECUTION.md`
  - `TASK_CATALOG.md` item 29
  - *Authority note*: Incorrect `FR-P5-14/15/16` attribution removed. Physical task execution (`PutBackTask`, RF scanning, operator put-back) is owned by P4 R5 / P4-002, and physical inventory recovery (`Inventory PICKED -> AVAILABLE`) is owned by P4-003. Neither is claimed as P4-001 behavior.

## Exact P4 Authority & P3/P4 Discriminator Boundary

1. **P3 vs P4 Discriminator**:
   - P3 Reservation Release handles strictly **pre-pick** states where formal picked quantity is 0 (`pickedQty = 0` and lines have not entered physical picking execution: `ALLOCATED`, `SHORT_ALLOCATED`, `CREATED`).
   - P4 handles **post-pick and post-pack** states:
     - Lines in `PICKING` or `SHORT_PICKED` with formal `pickedQty > 0`
     - Lines in `PICKED`
     - Lines in `PACKED`
2. **Supervisor Approval Requirement (P4 R1, FR-P4-01)**:
   - Lines with `PACKED` status or `packedQty > 0` require explicit Warehouse Supervisor approval (`supervisorApproved = true` and supervisor reason). Attempting cancellation of a `PACKED` line without supervisor approval is rejected with zero database mutation.
   - Lines in `PICKING`, `SHORT_PICKED`, or `PICKED` do not require supervisor approval.
3. **Hard Cutoff at POSTING_PENDING (P1 R38, P4 R2, FR-P4-02)**:
   - A `PACKED` cancellation is allowed only while the related Shipment is **before** `POSTING_PENDING`.
   - Once a Shipment enters `POSTING_PENDING`, `POSTED`, `POSTING_ERROR`, `IN_MANIFEST`, or `HANDED_TO_CARRIER`, or when a CarrierManifest enters `CLOSED`, `HANDED_OVER`, or `CONFIRMED`, cancellation is strictly forbidden with zero database mutation and visibly references the **Return Receipt** business path.
4. **Immediate Logical Settlement & Real STANDARD Allocation Lifecycle (P4 R3, INT-06, P1 KROK 6)**:
   - **STANDARD Allocation Lifecycle**: Under P1 KROK 6, an Allocation transitions from `RESERVED` to `CONFIRMED` transactionally upon full pick confirmation (`totalPickedForOol >= requiredQty`). Partial picks and short picks remain `RESERVED`.
   - Upon P4 cancellation:
     - All linked Allocations transition from `CONFIRMED` (or `RESERVED` if partial pick) to `RELEASED`.
     - Hard Inventory Reservation is deleted.
     - Exact quantity returns to `ATPReservation` immediately.
     - Packaging TU attached to the Shipment is detached (`shipment_id = NULL`) and transitioned from `IN_SHIPMENT` to `PACKING_SEALED`.
5. **Physical Stock & Inventory Isolation Boundary (P4 R3, TC-050, TC-051)**:
   - Physical stock remains physically picked in warehouse locations (`wms_inventory_balances` reserved quantity drops, but stock does NOT return to AVAILABLE in P4-001; no fabricated inventory movement).
   - No P4-003 put-back is executed.
   - While a physical-return handoff is pending (`status = 'PENDING'`), the physically picked stock is strictly deducted from available allocation and ATP stock queries. A subsequent allocation cannot reserve the physically picked quantity, guaranteeing inventory isolation until physical putback occurs.
6. **Durable Recovery Fact for P4-002 (P4 R4, TC-050, TC-051, TC-053)**:
   - For `pickedQty > 0`, a durable record is persisted in `wms_outbound_physical_return_handoffs` recording `customer_order_id`, `outbound_order_line_id`, `sku`, `quantity`, and `tu_id`.
   - For `pickedQty = 0`, zero recovery handoff is created (`TC-053`).

## Schema, Audit & Recovery Fact Changes

- Table `wms_outbound_physical_return_handoffs`:
  - Columns: `id`, `organization_id`, `tenant_id`, `warehouse_id`, `customer_order_id`, `outbound_order_id`, `outbound_order_line_id`, `sku`, `quantity`, `tu_id`, `source_location_id`, `reason`, `status`, `created_at`, `updated_at`.
- Entity `WmsOutboundReservationRelease`:
  - Extended to record `releaseType = 'POST_PICK'` with actor role `SUPERVISOR` or `SYSTEM`, tracking affected lines, recovery handoffs, released allocations, and withdrawn TUs.
- State transitions in `transitions.ts`:
  - Added `OUTBOUND_ORDER_TRANSITIONS`: `PICKED -> CANCELLED` and `PACKED -> CANCELLED` (actor: System WMS/Supervisor, architectRef: P4).
- Allocation lifecycle in `pick-task-service.ts`:
  - Wires `confirmPickLine` to transition `Allocation RESERVED -> CONFIRMED` with `AllocationConfirmed` domain event upon full pick completion.

## Dedicated PostgreSQL Suite Mapping (18 Tests)

Suite: `apps/mercato/src/modules/wms_outbound/services/__tests__/p4-001-postgres.integration.test.ts`  
Result: **18/18 PASSED** (63.7s) on canonical Testing PostgreSQL (Supabase pooler).

1. **TEST 1 (PICKING with pickedQty > 0)**: Immediate logical settlement without supervisor approval. Allocation `RELEASED`, ATP restored, line `CANCELLED`, handoff created.
2. **TEST 2 (SHORT_PICKED with pickedQty > 0)**: Settles immediately and stages handoff without supervisor approval requirement.
3. **TEST 3 (PICKED)**: Settles immediately and creates durable physical return handoff.
4. **TEST 4 (PACKED without supervisor approval)**: Rejected with zero database mutation; order, lines, allocations, and inventory reservations remain unchanged.
5. **TEST 5 (PACKED with supervisor approval)**: Settles immediately, detaches TU from Shipment (`shipment_id = null`, status `PACKING_SEALED`), creates durable handoff.
6. **TEST 6 (PACKED with Shipment POSTING_PENDING)**: Rejected with zero database mutation, referencing Return Receipt.
7. **TEST 7 (CarrierManifest CLOSED boundary)**: Closed manifest blocks cancellation and references Return Receipt.
8. **TEST 8 (Allocation and Inventory Settlement)**: Standard Allocation transitions to `RELEASED`, inventory reservation deleted, physical stock remains picked.
9. **TEST 9 (ATP Reservation Restoration)**: ATPReservation restoration is exact and occurs once without double-crediting.
10. **TEST 10 (Multi-line Atomic Settlement)**: Multi-line order with mixed statuses (PICKED and PICKING) settles atomically, generating separate recovery handoffs for each line.
11. **TEST 11 (Durable Recovery Fact)**: Durable recovery quantity recorded in `wms_outbound_physical_return_handoffs` for `pickedQty > 0`.
12. **TEST 12 (Zero Picked Quantity / TC-053)**: `pickedQty = 0` creates zero recovery handoff.
13. **TEST 13 (CROSSDOCK Channel)**: CROSSDOCK cancellation does not fabricate standard Allocation; creates physical return handoff for picked quantity.
14. **TEST 14 (Idempotent Replay)**: Replay with identical idempotency key returns saved result with `isReplay: true` and zero duplicate mutations or handoffs.
15. **TEST 15 (Rollback Proof)**: Simulated failure before commit aborts transaction leaving zero database mutations on independent read.
16. **TEST 16 (Real Concurrency Proof)**: Two overlapping cancellation requests serialize via PostgreSQL row locks to one execution and one replay.
17. **TEST 17 (STANDARD Allocation RESERVED -> CONFIRMED Lifecycle & Events)**:
    - Verifies partial pick (qty 4 of 10) leaves Allocation in `RESERVED`.
    - Verifies completing pick (qty 6 of 10) transitions Allocation from `RESERVED` to `CONFIRMED` with domain event `AllocationConfirmed`.
    - Verifies subsequent P4 post-pick cancellation transitions the confirmed allocation from `CONFIRMED` to `RELEASED` with event `AllocationReleased`.
18. **TEST 18 (P4 Logical Settlement Inventory Boundary & Pending Return Handoff Lockout)**:
    - Confirms physical picked quantity is NOT returned to AVAILABLE in `wms_inventory_balances`.
    - Confirms zero P4-003 putback tasks are created.
    - Confirms with no other stock for the SKU, a second allocation request cannot reserve the physically picked quantity while the return handoff is pending, settling the second allocation to `SHORT` (`SHORT_ALLOCATED`).

## Mandatory Regressions

All regression suites executed and verified on canonical Testing PostgreSQL:

- **P1-006 Outbound RF Picking Execution & Continuation Suite**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-006-postgres.integration.test.ts` — **12/12 PASSED** (71.3s).
- **P1-004 Outbound Wave Allocation Suite**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-004-postgres.integration.test.ts` — **11/11 PASSED** (47.8s).
- **P3-001 Reservation Release Suite**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-001-reservation-release-postgres.integration.test.ts` — **13/13 PASSED** (80.7s).
- **P3-002 Reservation Retention Suite**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-002-postgres.integration.test.ts` — **17/17 PASSED** (81.5s).
- **P1-001 CustomerOrder Lifecycle & Aggregation**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-001-customer-order-lifecycle.test.ts` — **10/10 PASSED** (1.9s).
- **FND-001 Ordering Adapter & Routing**: `apps/mercato/src/modules/wms_outbound/services/__tests__/fnd-001-ordering-adapter.test.ts` — **10/10 PASSED** (1.9s).

## Build, Service & Runtime Provenance

- `yarn generate`: Succeeded (471 API routes generated, OpenAPI spec generated, cache purged).
- `yarn workspace @open-mercato/app typecheck`: Succeeded (exit code 0, 0 errors).
- `yarn build:app`: Succeeded (Next.js 16.2.9 production build compiled successfully).
- Service: `mercato-localhost.service` restarted and active/running (`systemctl restart mercato-localhost.service`).
- HTTP Health: `https://devaxonic-test.info-start.com.pl/login` returned HTTP 200.

## Real Rendered Playwright UI Proof

Suite: `apps/mercato/src/modules/wms_outbound/__integration__/P4-001-post-pick-cancellation-ui.spec.ts`  
Result: **3/3 PASSED** (3.1m) on real rendered Mercato UI with zero route mocks:

- **Journey A (PACKED cancellation allowed with Supervisor approval)**:
  1. Seeds a genuine PACKED line into a Transport Unit attached to a Shipment in `DRAFT` (before `POSTING_PENDING`), with allocation in `CONFIRMED`.
  2. Navigates to `/backend/customer-orders/[id]`.
  3. Clicks `Check Cancellation (INT-06)`; evaluation panel displays `ELIGIBLE` with target `P4_PHYSICAL_PUTBACK`.
  4. Observes required Supervisor approval section (`p4-cancellation-section`) with checkbox (`supervisor-approval-checkbox`) and disabled action button (`execute-p4-cancellation-button`).
  5. Enters supervisor reason, checks approval checkbox, button becomes enabled.
  6. Clicks `Approve & Settle Cancellation (P4)`.
  7. Observes success feedback banner, page reloads with order `CANCELLED`.
  8. Database verification confirms: CustomerOrder and lines `CANCELLED`, Allocation `RELEASED`, Inventory Reservation cleared, TU detached from Shipment (`shipment_id = null`, status `PACKING_SEALED`), durable handoff created in `wms_outbound_physical_return_handoffs`.
- **Journey B (PACKED cancellation blocked when Shipment is POSTING_PENDING)**:
  1. Seeds a genuine PACKED line attached to a Shipment in `POSTING_PENDING`, with allocation in `CONFIRMED`.
  2. Navigates to `/backend/customer-orders/[id]`.
  3. Clicks `Check Cancellation (INT-06)`.
  4. Evaluation panel displays `BLOCKED` and visibly references Return Receipt direction.
  5. Zero cancellation buttons are rendered.
  6. Database verification confirms: order remains `IN_FULFILLMENT`, allocations `CONFIRMED`, zero physical return handoffs.
- **Journey C (PICKED line cancellation without supervisor approval requirement)**:
  1. Seeds a genuine PICKED line (pickedQty > 0, not packed), with allocation in `CONFIRMED`.
  2. Navigates to `/backend/customer-orders/[id]`.
  3. Clicks `Check Cancellation (INT-06)`.
  4. Evaluation panel displays `ELIGIBLE` with target `P4_PHYSICAL_PUTBACK`.
  5. Observes no supervisor approval checkbox; execute button is directly enabled.
  6. Clicks `Approve & Settle Cancellation (P4)`.
  7. Success feedback received; order settles to `CANCELLED`; durable physical return handoff created in database.

## Exclusions Preserved

- No P3-003 RF physical-removal race window or exact-source RF return instruction.
- Zero `PutBackTask` created (P4-002 owns PutBackTask model/assignment/FIFO/RF).
- No `Inventory PICKED -> AVAILABLE` physical recovery or destination-location loop fabricated (P4-003 scope).
- Return Receipt remains separate scope.
- No new carrier cancellation integration invented.
- Scanner UI remains frozen at baseline `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`.
- Demo/Prod environments unchanged.
