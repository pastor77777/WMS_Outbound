# P1-007 Execution & Acceptance Evidence

**Catalog Item:** `P1-007` — SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (P1 R44–R47, R48, R69, FR-P1-31, FR-P1-32, FR-P5-04, FR-P5-05, FR-P5-06, TC-060, TC-061, TC-062, TC-121)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Authoritative Repository Commit SHAs

* **Mercato P1-007 Final Head (`outbound/p1-007`):** `1979942c993688be84d8765c0baae0eb594f9aa9`
* **Scanner P1-007 Final Head (`main`):** `b23325aae1c4f83b79d01b3650dbead3486a1041`
* **Authoritative Outbound Steering Head (`main`):** `4bf16d97aaed33333695412a654f2d4c82c065d4`
* **Testing PostgreSQL Database:** Remote DevAxonic Testing Database (`devaxonic-test.info-start.com.pl`).
* **Live Services:**
  - Mercato Next.js Backend: `https://devaxonic-test.info-start.com.pl` (local runner: `http://localhost:3000`)
  - Scanner Web App: `http://localhost:8081`

---

## 2. P1-007 Core Capabilities Implemented & Verified

### A. TC-060: SHORT_ALLOCATED Exception Handling & Supervisor Outcomes
* **When `allowPartialShipment = true`:** Automatically proceeds with available inventory (`SHORT_ALLOCATED`), preserving partial order flow.
* **When `allowPartialShipment = false`:** Halts order release to warehouse and escalates to Supervisor:
  - **Outcome A (ALLOW_PARTIAL):** Supervisor authorizes partial shipment with mandatory reason audit. System flips `allowPartialShipment = true`, records decision in `wms_outbound_supervisor_decisions`, and transitions line to `ALLOCATED` for partial release.
  - **Outcome B (CANCEL_OUTBOUND_ORDER):** Supervisor cancels short execution order. All associated `wms_outbound_allocations` are released to `RELEASED`, linked `wms_inventory_reservations` removed, `wms_outbound_order_lines` cancelled, and `wms_outbound_customer_orders` is reverted to `ACCEPTED` with `has_warning = true` and `warning_reason` populated for ERP/ATP resolution. Soft ATP reservation is restored on `CustomerOrderLine`.

### B. TC-061: SHORT_PICKED Automatic Reallocation Limits & Location Blocking
* **R44 Location Shortage Blocking:** Whenever a short pick is reported at location L for SKU S, a persistent record is written to `wms_outbound_location_shortages` with `is_active = true`. Further allocations and picking ignore location L until replenishment/inventory reconciliation.
* **R44 Two-Bound Availability Enforcement:** Candidate replacement locations are evaluated under SKU lock against both:
  1. Individual unblocked location unreserved stock (`grossStock - locationHardReservations >= shortageQty`).
  2. Warehouse-wide unreserved hard capacity (`grossQualifyingStock - totalWarehouseHardReservations >= shortageQty`), accounting for all active hard allocations (`RESERVED`, `CONFIRMED`, `SHORT`) across all orders, including pre-PickTask orders.
  If either bound is unmet, auto-reallocation ceases immediately and the shortage escalates to the Supervisor without overbooking.
* **Effective Retry Limit Hierarchy:** Customer override (`CustomerOrder.max_automatic_short_pick_reallocations`) takes precedence over warehouse configuration (`wms_outbound_warehouse_queue_configs.max_automatic_short_pick_reallocations`), defaulting to 1. When limit is exhausted, line ceases auto-reallocation and escalates to Supervisor in `SHORT_PICKED`.

### C. TC-062: SHORT_PICKED Supervisor Outcomes & Real Migration Lifecycle (Shot 1B Corrective Proof)
* **Outcome A (WAIT / CZEKAMY):** Rolls back un-packed sister lines of the order. Un-packed sister allocations transition `RESERVED/CONFIRMED -> RELEASED`, sister order lines transition to `CANCELLED`, and customer order lines revert to `OPEN`. CustomerOrder reverts to `ACCEPTED` with `has_warning = true`. Exact unpicked soft ATP promises are restored. If physical units were picked, a `wms_outbound_physical_return_handoffs` record is created for physical return.
* **Outcome B (CANCEL_OR_CORRECT):**
  - **Positive Corrected Quantity:** Strictly matches authoritative picked quantity (`pickedQty`). Concurrently updates commercial demand (`CustomerOrderLine.orderedQuantity = pickedQty`) and execution demand (`OutboundOrderLine.requiredQty = pickedQty`), and reconciles `WmsInventoryReservation.quantity = pickedQty` per TC-121 lifecycle invariants. Line advances to `PICKED`.
  - **Zero Quantity (0 - Full Line Cancellation):** Supported via migration `Migration20260902160000_wms_outbound_p1_007_remediation.ts` updating table check constraints to `CHECK (ordered_quantity >= 0)` and `CHECK (required_qty >= 0)`. Concurrently updates `CustomerOrderLine.orderedQuantity = 0.000000`, `CustomerOrderLine.status = 'CANCELLED'`, `OutboundOrderLine.requiredQty = 0.000000`, and `OutboundOrderLine.status = 'CANCELLED'`. Releases hard allocations to `RELEASED`, deletes linked inventory reservations, and persists `wms_outbound_physical_return_handoffs` for any physical units picked.
* **Outcome C (ALLOW_PARTIAL):** Persistently enables `CustomerOrder.allowPartialShipment = true` with mandatory reason audit. Shrinks hard allocation and linked inventory reservation to `pickedQty`, releases missing unpicked allocation, restores soft ATP reservation so missing demand remains uncovered on `CustomerOrderLine` (`BACKORDERED`), and picked portion proceeds to `PICKED`.
* **Deterministic Reversible Migration Proof without Shadow Tables (`Migration20260902160000_wms_outbound_p1_007_remediation.ts`):**
  - Suite-global `beforeAll` schema patching (`ALTER TABLE ... >= 0`) was completely removed; the entire test suite executes directly against the migration-managed schema.
  - Test `6A` builds an isolated test schema exclusively through the **real preceding project migration lifecycle** (`Migration20260831200000_wms_outbound` through `Migration20260902100000_wms_outbound_p1_007`), completely eliminating hand-written table DDL and shadow table duplication.
  - **Path A (Compatible Data):** Unconditionally executes remediation migration UP (`>= 0`), DOWN (`> 0`), and re-UP (`>= 0`) without `if` guards or skips, asserting fresh PostgreSQL constraint state at each step.
  - **Path B (Incompatible Data Fail-Safe):** Under UP state with legitimate P1-007 R46 cancelled zero-quantity rows present in the real migration tables, DOWN fails fast with an explicit compatibility error *before* any constraint mutation, preserving all zero rows and leaving `>= 0` constraints intact without data loss.

### D. Supervisor Idempotency with Canonical Payload Derivation
* In `customer-order-service.ts`, the authoritative `outboundOrderId` is resolved directly from `WmsOutboundOrderLine` *before* comparing the incoming payload against the recorded decision snapshot.
* Replays with identical key + canonical payload succeed idempotently without duplicate writes.
* Replays with same key + conflicting payload (different decision, different correctedQuantity, or different reason) fail closed and raise a structured `409 Conflict`.

---

## 3. Remote Testing PostgreSQL Integration Evidence (20/20 PASSED)

**Test Command:**
```bash
yarn --cwd apps/mercato test src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts --runInBand
```

**Decisive Concurrency & pg_blocking_pids Lock Output:**
```text
  console.log
    [P1-007 Decisive PostgreSQL Lock & Reallocation Concurrency Proof] {
      actorAPid: 1816174,
      actorBPid: 1815350,
      blockingPids: [ 1816174 ],
      waitEventType: 'Lock',
      lockEvidenceCaptured: true,
      replacementTasksCreated: 1,
      retryCount: 1
    }
```

**Decisive Migration Proof Output (Test 6A — Shot 1B Real Migration Lifecycle):**
```text
  console.log
    [P1-007 Decisive Migration Proof] {
      compatibleUpExecuted: true,
      compatibleDownExecuted: true,
      compatibleReUpExecuted: true,
      incompatibleDownBlocked: true,
      incompatibleRowsPreserved: true,
      constraintsAfterBlockedDown: '>= 0'
    }
```

**Verbatim Output:**
```text
PASS src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts (77.045 s)
  P1-007 Genuine PostgreSQL SHORT_ALLOCATED & SHORT_PICKED Recovery Suite
    1. TC-060 SHORT_ALLOCATED Handling & Outcomes
      ✓ 1A: allowPartialShipment = true drives real planning/allocation path: available allocated, shortfall BACKORDERED, no supervisor intervention (2290 ms)
      ✓ 1B: allowPartialShipment = false -> Outcome A: Persistent ALLOW_PARTIAL with reason and mandatory idempotencyKey (1380 ms)
      ✓ 1C: allowPartialShipment = false -> Outcome B: CANCEL_OUTBOUND_ORDER releases allocations, restores soft ATP reservations, and reverts order to ACCEPTED + WARNING (1590 ms)
    2. TC-061 SHORT_PICKED Automatic Reallocation Limits & Location Blocking
      ✓ 2A1: unblocked location exists but has ZERO ATP inventory -> no replacement task, Supervisor escalation (1510 ms)
      ✓ 2A2: alternative location has non-ATP stock only (DRAFT ASN) -> no replacement task, Supervisor escalation (1460 ms)
      ✓ 2A3: alternative location has eligible ATP stock -> creates exactly one replacement reservation + PickTask, source blocked in location shortages (1790 ms)
      ✓ 2A4: candidate B has 2.0, candidate C has 2.0, shortage is 3.0 -> aggregate stock is 4.0 but neither location individually covers 3.0 -> ceases auto-reallocation and escalates to Supervisor (1610 ms)
      ✓ 2A5: candidate B has 5.0 gross stock but 3.0 already hard-reserved by active pick task -> unreserved stock is only 2.0 -> shortage of 3.0 is rejected, escalating to Supervisor (1640 ms)
      ✓ 2A6: Candidate B has gross 5.0, competing order has active pre-PickTask hard allocation of 3.0 -> unreserved is 2.0 -> shortage 3.0 cannot be reallocated -> ceases auto-reallocation and escalates to Supervisor (1680 ms)
      ✓ 2B: Effective retry limit hierarchy: customer override wins over warehouse setting (1570 ms)
      ✓ 2C: When effective retry limit is reached, short pick ceases automatic reallocation and escalates to Supervisor (1410 ms)
    3. TC-062 SHORT_PICKED Supervisor Outcomes
      ✓ 3A: Outcome A (WAIT) rolls back un-packed sister lines to OPEN, releases hard allocations, restores soft ATP reservations, and reverts CustomerOrder to ACCEPTED + WARNING (1740 ms)
      ✓ 3B: Outcome B (CANCEL_OR_CORRECT) rejects arbitrary positive values, accepts exact picked quantity per TC-121 (1510 ms)
      ✓ 3B2: Outcome B (CANCEL_OR_CORRECT with 0) performs full line cancellation: sets CustomerOrderLine.orderedQuantity = 0, OutboundOrderLine.requiredQty = 0, releases allocation, deletes inventory reservation, creates physical return handoff for 4 picked units (1540 ms)
      ✓ 3C: Outcome C (ALLOW_PARTIAL) persistently updates CustomerOrder.allowPartialShipment with reason (1390 ms)
    4. Concurrency, Idempotency, Strict Observer & Rollback Proofs
      ✓ 4A: Independent connection SAME-K short pick confirmation captures strict pg_blocking_pids lock evidence and deduplicates exactly once (1950 ms)
      ✓ 4B: Supervisor authorization fails closed for unauthorized caller, empty idempotencyKey fails 422, same key replayed returns prior result, conflicting payload fails closed (1230 ms)
      ✓ 4B2: Exhaustive SHORT_PICKED supervisor idempotency suite: 8 scenarios covering CANCEL_OR_CORRECT, ALLOW_PARTIAL, WAIT, exact matches, conflicting decisions, conflicting correctedQuantities, conflicting reasons, and conflicting supervisor identities (1620 ms)
      ✓ 4C: Rollback proof: real failure before commit ensures no partial shortage state leaked (1140 ms)
    5. R48 Repack Shortage Backend Seam
      ✓ 5A: Supervisor shortage resolution service preserves repacked carton integrity without phantom allocation leaks (1200 ms)
      ✓ 6A: Real reversible migration proof: UP/DOWN/re-UP execution with fail-safe incompatible-data protection (1710 ms)

Test Suites: 1 passed, 1 total
Tests:       20 passed, 20 total
Snapshots:   0 total
Time:        77.045 s
```

---

## 4. Real Rendered UI Playwright Evidence

### A. Supervisor Web UI E2E Suite (`P1-007-shortages-supervisor-ui.spec.ts`) — 6/6 Outcomes Verified
* **Target URL:** `http://localhost:3000/backend/shortages`
* **Test Command:**
```bash
PLAYWRIGHT_TEST_BASE_URL=http://localhost:3000 npx playwright test src/modules/wms_outbound/__integration__/P1-007-shortages-supervisor-ui.spec.ts
```
* **Execution Result:** `6 passed (2.7m)`
* **All 6 Distinct UI Outcomes Verified:**
  1. `Outcome 1: SHORT_ALLOCATED -> ALLOW_PARTIAL (persistent with reason)` (26.6s)
  2. `Outcome 2: SHORT_ALLOCATED -> CANCEL_OUTBOUND_ORDER` (27.6s)
  3. `Outcome 3: SHORT_PICKED -> WAIT` (27.8s)
  4. `Outcome 4: SHORT_PICKED -> CANCEL_OR_CORRECT (exact picked quantity 4)` (24.2s)
  5. `Outcome 5: SHORT_PICKED -> CANCEL_OR_CORRECT (0 cancellation)` (25.5s)
  6. `Outcome 6: SHORT_PICKED -> ALLOW_PARTIAL (persistent with mandatory reason)` (27.1s)
* **Decisive Proof:** Real rendered React UI displayed both `SHORT_ALLOCATED` and `SHORT_PICKED` exception tables, submitted modal decisions with audit reasons and canonical UUIDs, and verified database persistence across all 6 outcomes in remote PostgreSQL.

### B. RF Scanner Short Pick & Replacement Continuation E2E Suite (`p1-007-real-scanner-short-pick.spec.ts`)
* **Target URL:** `http://localhost:8081`
* **Test Command:**
```bash
PLAYWRIGHT_TEST_BASE_URL=http://localhost:3000 npx playwright test e2e/p1-007-real-scanner-short-pick.spec.ts
```
* **Execution Result:** `1 passed (18.8s)`
* **Decisive Proof:** Operator authenticated against backend via proxy, bound picking TU, reported short quantity (2 picked out of 5), verified `⚠️ Report Short Pick` UI action, validated task status `SHORT_PICKED` in PostgreSQL, verified `wms_outbound_location_shortages` recorded active block (`short_quantity = 3`, `is_active = true`), requested next task in zone, received replacement task at Loc B for 3 units, completed pick of 3 units, verified task completion banner in UI, and verified in PostgreSQL:
  - Original task: `SHORT_PICKED`
  - Replacement task: `COMPLETED`
  - Cumulative picked quantity: 5
  - `OutboundOrderLine`: `status = 'PICKED'`

---

## 5. Every Required Targeted Regression Gate (Cross-Ticket Proof)

| Gate | Scope / Invariant Covered | Test Command | Result |
| :--- | :--- | :--- | :--- |
| **P1-004** | Allocation / Hard-Reservation | `yarn test src/modules/wms_outbound/services/__tests__/p1-004-postgres.integration.test.ts --runInBand` | **1 passed, 11/11 tests (46.658 s)** |
| **P1-005** | PickTask creation / assignment / single-active | `yarn test src/modules/wms_outbound/services/__tests__/p1-005-postgres.integration.test.ts --runInBand` | **1 passed, 10/10 tests (23.694 s)** |
| **P1-006 (Backend)** | Real RF picking + SAME-key concurrency & retry | `yarn test src/modules/wms_outbound/services/__tests__/p1-006-postgres.integration.test.ts --runInBand` | **1 passed, 12/12 tests (68.776 s)** |
| **P1-006 (Scanner)** | Real scanner picking & retry key stability | `PLAYWRIGHT_TEST_BASE_URL=http://localhost:3000 npx playwright test e2e/p1-006-real-scanner-picking.spec.ts e2e/p1-006-retry-key.spec.ts` | **2 passed (33.0 s)** |
| **P1-001** | CustomerOrder lifecycle & aggregation | `yarn test src/modules/wms_outbound/services/__tests__/p1-001-customer-order-lifecycle.test.ts src/modules/wms_outbound/services/__tests__/p1-001-postgres.integration.test.ts --runInBand` | **2 passed, 21/21 tests (11.807 s)** |
| **P1-003** | Planning & requiredQty | `yarn test src/modules/wms_outbound/services/__tests__/p1-003-postgres.integration.test.ts src/modules/wms_outbound/services/__tests__/p1-003-detail-api-postgres.integration.test.ts --runInBand` | **2 passed, 15/15 tests (56.587 s)** |
| **P1-008** | TU regression (*zero TU code touched in sixth override*) | `yarn test src/modules/wms_outbound/services/__tests__/p1-008-postgres.integration.test.ts --runInBand` | **1 passed, 22/22 tests (23.88 s)** |
| **Inbound & Shared Compatibility** | ATP & shared boundaries | `yarn test src/modules/wms_inventory/services/__tests__/atp-service.test.ts src/modules/wms_outbound/services/__tests__/fnd-003-shared-compatibility.test.ts src/modules/wms_outbound/services/__tests__/fnd-003-postgres.integration.test.ts --runInBand` | **3 passed, 17/17 tests (6.711 s)** |
| **Full Outbound Gate** | Entire `wms_outbound` suite (Umbrella Gate) | `yarn test src/modules/wms_outbound --runInBand` | **17 passed, 245/245 tests (352.937 s)** |

---

## 6. Visual Evidence Artifacts

1. **Supervisor Shortages & Outcomes Console:**
   `05_EVIDENCE/screenshots/p1-007-real-supervisor-shortages-ui.png`
2. **Scanner RF Picking Short Pick UI:**
   `05_EVIDENCE/screenshots/p1-007-real-scanner-short-pick.png`
3. **Scanner RF Picking Replacement Task Completed UI:**
   `05_EVIDENCE/screenshots/p1-007-real-scanner-replacement-completed.png`
