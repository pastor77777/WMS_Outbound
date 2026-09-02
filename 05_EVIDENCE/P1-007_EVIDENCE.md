# P1-007 Execution & Acceptance Evidence

**Catalog Item:** `P1-007` — SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (P1 R44–R47, R48, R69, FR-P1-31, FR-P1-32, FR-P5-04, FR-P5-05, FR-P5-06, TC-060, TC-061, TC-062, TC-121)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Authoritative Repository Commit SHAs

* **Mercato P1-006 Base:** `353a5001cb8f1941971f960e509a8af643e41e5a`
* **Mercato P1-007 Final Head (`outbound/p1-007`):** `378d7a2d1996ccfbe3bc1184a37cddb376d55a2f`
* **Scanner P1-007 Final Head (`main`):** `0f1ca673ca8bbc1de8b77c99ca91e9d8ae00a6df`
* **Authoritative Outbound Steering Head (`main`):** `a1e79e60f04f21bbbe8a3a0e633d76e73c883ea5`
* **Testing PostgreSQL Database:** Remote DevAxonic Testing Database.
* **Live Services:**
  - Mercato Next.js Backend: `https://devaxonic-test.info-start.com.pl`
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
* **R44 Individual Location Coverage & Accepted ATP/Reservation Path:** Replacement task is created only if an unblocked storage location individually possesses `availableQty >= shortageQty`. If no single location covers the shortage, auto-reallocation ceases and escalates to Supervisor. Hard allocation and linked `wms_inventory_reservations` are created through the accepted primitive.
* **Effective Retry Limit Hierarchy:** Customer override (`CustomerOrder.max_automatic_short_pick_reallocations`) takes precedence over warehouse configuration (`wms_outbound_warehouse_queue_configs.max_automatic_short_pick_reallocations`), defaulting to 1. When limit is exhausted, line ceases auto-reallocation and escalates to Supervisor in `SHORT_PICKED`.

### C. TC-062: SHORT_PICKED Supervisor Outcomes (R45, R46, R47, R69, TC-121)
* **Outcome A (WAIT / CZEKAMY):** Rolls back un-packed sister lines of the order. Un-packed sister allocations transition `RESERVED/CONFIRMED -> RELEASED`, sister order lines transition to `CANCELLED`, and customer order lines revert to `OPEN`. CustomerOrder reverts to `ACCEPTED` with `has_warning = true`. Soft ATP promises are restored. If physical units were picked, a `wms_outbound_physical_return_handoffs` record is created for physical return.
* **Outcome B (CANCEL_OR_CORRECT):**
  - **Positive Corrected Quantity:** Strictly matches authoritative picked quantity (`pickedQty`). Concurrently updates commercial demand (`CustomerOrderLine.orderedQuantity = pickedQty`) and execution demand (`OutboundOrderLine.requiredQty = pickedQty`), and reconciles `WmsInventoryReservation.quantity = pickedQty` per TC-121 lifecycle invariants. Line advances to `PICKED`.
  - **Zero Quantity (0 - Full Cancel):** Sets `CustomerOrderLine.orderedQuantity = 0`, `CustomerOrderLine.status = 'CANCELLED'`, `OutboundOrderLine.requiredQty = 0`, `OutboundOrderLine.status = 'CANCELLED'`, releases hard allocations and deletes linked inventory reservations. If `pickedQty > 0`, persists `wms_outbound_physical_return_handoffs` to ensure picked stock is returned.
* **Outcome C (ALLOW_PARTIAL):** Persistently enables `CustomerOrder.allowPartialShipment = true` with mandatory reason audit. Shrinks hard allocation and linked inventory reservation to `pickedQty`, releases missing unpicked allocation, restores soft ATP reservation so missing demand remains uncovered on `CustomerOrderLine` (`BACKORDERED`), and picked portion proceeds to `PICKED`.

### D. Supervisor Idempotency with Canonical Material Payload Comparison
* Supervisor endpoints compare full canonical material payload (`supervisorId`, `customerOrderId`, `outboundOrderId`, `outboundOrderLineId`, `decision`, `correctedQuantity`, and normalized `reason`).
* Replay with identical key + identical payload returns prior result without duplicate writes.
* Replay with same key + conflicting payload throws 409 Conflict.

---

## 3. Remote Testing PostgreSQL Integration Evidence (15/15 PASSED)

**Test Command:**
```bash
yarn --cwd apps/mercato test src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts --runInBand
```

**Decisive Concurrency & pg_blocking_pids Lock Output:**
```text
  console.log
    [P1-007 Decisive PostgreSQL Lock & Reallocation Concurrency Proof] {
      actorAPid: 1784552,
      actorBPid: 1788735,
      blockingPids: [ 1784552 ],
      waitEventType: 'Lock',
      lockEvidenceCaptured: true,
      replacementTasksCreated: 1,
      retryCount: 1
    }
```

**Verbatim Output:**
```text
PASS src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts (53.867 s)
  P1-007 Genuine PostgreSQL SHORT_ALLOCATED & SHORT_PICKED Recovery Suite
    1. TC-060 SHORT_ALLOCATED Handling & Outcomes
      ✓ 1A: allowPartialShipment = true drives real planning/allocation path: available allocated, shortfall BACKORDERED, no supervisor intervention (2310 ms)
      ✓ 1B: allowPartialShipment = false -> Outcome A: Persistent ALLOW_PARTIAL with reason and mandatory idempotencyKey (1410 ms)
      ✓ 1C: allowPartialShipment = false -> Outcome B: CANCEL_OUTBOUND_ORDER releases allocations, restores soft ATP reservations, and reverts order to ACCEPTED + WARNING (1620 ms)
    2. TC-061 SHORT_PICKED Automatic Reallocation Limits & Location Blocking
      ✓ 2A1: unblocked location exists but has ZERO ATP inventory -> no replacement task, Supervisor escalation (1520 ms)
      ✓ 2A2: alternative location has non-ATP stock only (DRAFT ASN) -> no replacement task, Supervisor escalation (1480 ms)
      ✓ 2A3: alternative location has eligible ATP stock -> creates exactly one replacement reservation + PickTask, source blocked in location shortages (1810 ms)
      ✓ 2A4: candidate B has 2.0, candidate C has 2.0, shortage is 3.0 -> aggregate stock is 4.0 but neither location individually covers 3.0 -> ceases auto-reallocation and escalates to Supervisor (1630 ms)
      ✓ 2B: Effective retry limit hierarchy: customer override wins over warehouse setting (1590 ms)
      ✓ 2C: When effective retry limit is reached, short pick ceases automatic reallocation and escalates to Supervisor (1420 ms)
    3. TC-062 SHORT_PICKED Supervisor Outcomes
      ✓ 3A: Outcome A (WAIT) rolls back un-packed sister lines to OPEN, releases hard allocations, restores soft ATP reservations, and reverts CustomerOrder to ACCEPTED + WARNING (1760 ms)
      ✓ 3B: Outcome B (CANCEL_OR_CORRECT) rejects arbitrary positive values, accepts 0 (with physical return handoff) and exact picked quantity (updating CustomerOrderLine.orderedQuantity AND OutboundOrderLine.requiredQty per TC-121) (1530 ms)
      ✓ 3C: Outcome C (ALLOW_PARTIAL) persistently updates CustomerOrder.allowPartialShipment with reason (1410 ms)
    4. Concurrency, Idempotency, Strict Observer & Rollback Proofs
      ✓ 4A: Independent connection SAME-K short pick confirmation captures strict pg_blocking_pids lock evidence and deduplicates exactly once (1980 ms)
      ✓ 4B: Supervisor authorization fails closed for unauthorized caller, empty idempotencyKey fails 422, same key replayed returns prior result, conflicting payload fails closed (1250 ms)
      ✓ 4C: Rollback proof: real failure before commit ensures no partial shortage state leaked (1160 ms)

Test Suites: 1 passed, 1 total
Tests:       15 passed, 15 total
Snapshots:   0 total
Time:        53.867 s
```

---

## 4. Full Outbound Regression Gate (17/17 Suites, 240/240 PASSED)

**Test Command:**
```bash
yarn --cwd apps/mercato test src/modules/wms_outbound
```

**Verbatim Output:**
```text
Test Suites: 17 passed, 17 total
Tests:       240 passed, 240 total
Snapshots:   0 total
Time:        163.09 s
Ran all test suites matching src/modules/wms_outbound.
```

---

## 5. Real Rendered UI Playwright Evidence

### A. Supervisor Web UI E2E Suite (`P1-007-shortages-supervisor-ui.spec.ts`)
* **Target URL:** `https://devaxonic-test.info-start.com.pl/backend/shortages`
* **Test Suite:** `apps/mercato/src/modules/wms_outbound/__integration__/P1-007-shortages-supervisor-ui.spec.ts`
* **Execution Result:** `2 passed (18.7s)`
* **Decisive Proof:** Real rendered React UI displayed both `SHORT_ALLOCATED` and `SHORT_PICKED` exception tables, allowed selecting supervisor actions (`ALLOW_PARTIAL`, `CANCEL_OR_CORRECT`), submitted modal with reason audit and canonical user UUID, and verified PostgreSQL `wms_outbound_supervisor_decisions` rows, `CustomerOrderLine.orderedQuantity = 4`, `OutboundOrderLine.requiredQty = 4`, and `allow_partial_shipment` mutation.

### B. RF Scanner Short Pick & Replacement Continuation E2E Suite (`p1-007-real-scanner-short-pick.spec.ts`)
* **Target URL:** `http://localhost:8081`
* **Test Suite:** `Devaxonic-scanner/e2e/p1-007-real-scanner-short-pick.spec.ts`
* **Execution Result:** `1 passed (19.4s)`
* **Decisive Proof:** Operator authenticated, bound picking TU, reported short quantity (2 picked out of 5), verified `⚠️ Report Short Pick` UI action, validated task status `SHORT_PICKED` in PostgreSQL, verified `wms_outbound_location_shortages` recorded active block (`short_quantity = 3`, `is_active = true`), requested next task in zone, received replacement task at Loc B for 3 units, completed pick of 3 units, verified task completion banner in UI, and verified in PostgreSQL:
  - Original task: `SHORT_PICKED`
  - Replacement task: `COMPLETED`
  - Cumulative picked quantity: 5
  - `OutboundOrderLine`: `status = 'PICKED'`

---

## 6. Visual Evidence Artifacts

1. **Supervisor Shortages & Outcomes Console:**
   `05_EVIDENCE/screenshots/p1-007-real-supervisor-shortages-ui.png`
2. **Scanner RF Picking Short Pick UI:**
   `05_EVIDENCE/screenshots/p1-007-real-scanner-short-pick.png`
3. **Scanner RF Picking Replacement Task Completed UI:**
   `05_EVIDENCE/screenshots/p1-007-real-scanner-replacement-completed.png`

