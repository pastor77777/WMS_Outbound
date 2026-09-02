# P1-007 Execution & Acceptance Evidence

**Catalog Item:** `P1-007` — SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (P1 R44–R47, R48, R69, FR-P1-31, FR-P1-32, FR-P5-04, FR-P5-05, FR-P5-06, TC-060, TC-061, TC-062, TC-121)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Authoritative Repository Commit SHAs

* **Mercato P1-006 Base:** `353a5001cb8f1941971f960e509a8af643e41e5a`
* **Mercato P1-007 Final Head (`outbound/p1-007`):** `e9ec7d3b84db8e62d665b16fb5265ecb930d6350`
* **Scanner P1-007 Final Head (`main`):** `e45967a544525ea2476e339d332bcba2d287bb7c`
* **Authoritative Outbound Steering Head (`main`):** `ac04783f562356ea7506ebeb739d1981d3df8024`
* **Testing PostgreSQL Database:** Remote DevAxonic Testing Database.
* **Live Services:**
  - Mercato Next.js Backend: `http://localhost:3009`
  - Scanner Web App: `http://localhost:8081`

---

## 2. P1-007 Core Capabilities Implemented & Verified

### A. TC-060: SHORT_ALLOCATED Exception Handling & Supervisor Outcomes
* **When `allowPartialShipment = true`:** Automatically proceeds with available inventory (`SHORT_ALLOCATED`), preserving partial order flow.
* **When `allowPartialShipment = false`:** Halts order release to warehouse and escalates to Supervisor:
  - **Outcome A (ALLOW_PARTIAL):** Supervisor authorizes partial shipment with mandatory reason audit. System flips `allowPartialShipment = true`, records decision in `wms_outbound_supervisor_decisions`, and transitions line to `ALLOCATED` for partial release.
  - **Outcome B (CANCEL_OUTBOUND_ORDER):** Supervisor cancels short execution order. All associated `wms_outbound_allocations` are released to `RELEASED`, `wms_outbound_order_lines` cancelled, and `wms_outbound_customer_orders` is reverted to `ACCEPTED` with `has_warning = true` and `warning_reason` populated for ERP/ATP resolution.

### B. TC-061: SHORT_PICKED Automatic Reallocation Limits & Location Blocking
* **R44 Location Shortage Blocking:** Whenever a short pick is reported at location L for SKU S, a persistent record is written to `wms_outbound_location_shortages` with `is_active = true`. Further allocations and picking ignore location L until replenishment/inventory reconciliation.
* **Automatic Replacement Task Generation:** If alternative unblocked locations with inventory exist and retry count is under `max_automatic_short_pick_reallocations` (configured in `wms_outbound_warehouse_queue_configs`), the short PickTask is terminated in `SHORT_PICKED`, short quantity is reallocated to an unblocked location, and a replacement `PickTask` is queued.
* **Retry Limit Exhaustion:** When `short_pick_reallocations_count >= max_automatic_short_pick_reallocations`, automatic reallocation ceases and the line is escalated to Supervisor in `SHORT_PICKED` state.

### C. TC-062: SHORT_PICKED Supervisor Outcomes (R45, R46, R47, R69, TC-121)
* **Outcome A (WAIT / CZEKAMY):** Rolls back un-packed sister lines of the order. Un-packed sister allocations transition `RESERVED/CONFIRMED -> RELEASED`, sister order lines transition to `CANCELLED`, and customer order lines revert to `OPEN`. CustomerOrder reverts to `ACCEPTED` with `has_warning = true`.
* **Outcome B (CANCEL_OR_CORRECT):** Concurrently updates commercial demand (`CustomerOrderLine.orderedQuantity`) and execution demand (`OutboundOrderLine.requiredQty`) to the actual picked quantity per TC-121 lifecycle invariants. Lines transition to `PICKED`.
* **Outcome C (ALLOW_PARTIAL):** Persistently enables `CustomerOrder.allowPartialShipment = true` with mandatory reason audit. Line transitions to `PICKED` to allow packing and shipping.

### D. R48 Repack Shortage Backend Seam
* Implemented `blockSourceLocation` parameter in `confirmPickLine`. When set to `false` (e.g. during packing shortage detection where source location was not at fault), the task terminates in `SHORT_PICKED` without creating a location shortage block.

---

## 3. Remote Testing PostgreSQL Integration Evidence (11/11 PASSED)

**Test Command:**
```bash
npx jest src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts --runInBand
```

**Verbatim Output:**
```text
PASS src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts (29.107 s)
  P1-007 Genuine PostgreSQL SHORT_ALLOCATED & SHORT_PICKED Recovery Suite
    1. TC-060 SHORT_ALLOCATED Handling & Outcomes
      ✓ 1A: allowPartialShipment = true -> auto-proceeds with available quantity without halting order (1214 ms)
      ✓ 1B: allowPartialShipment = false -> Outcome A: Persistent ALLOW_PARTIAL with reason (1102 ms)
      ✓ 1C: allowPartialShipment = false -> Outcome B: CANCEL_OUTBOUND_ORDER releases allocations and reverts order to ACCEPTED + WARNING (1420 ms)
    2. TC-061 SHORT_PICKED Automatic Reallocation Limits & Location Blocking
      ✓ 2A: Automatic short pick creates replacement task in alternative location and blocks source location (1680 ms)
      ✓ 2B: When retry limit reached, short pick escalates to Supervisor without creating new task (1240 ms)
    3. TC-062 SHORT_PICKED Supervisor Outcomes
      ✓ 3A: Outcome A (WAIT) rolls back un-packed sister lines to OPEN and reverts CustomerOrder to ACCEPTED + WARNING (1550 ms)
      ✓ 3B: Outcome B (CANCEL_OR_CORRECT) concurrently updates CustomerOrderLine.orderedQuantity AND OutboundOrderLine.requiredQty (TC-121 lifecycle) (1380 ms)
      ✓ 3C: Outcome C (ALLOW_PARTIAL) persistently updates CustomerOrder.allowPartialShipment with reason (1290 ms)
    4. Concurrency, Idempotency & Rollback Proofs
      ✓ 4A: Idempotent replay of Supervisor decision returns identical result without duplicate audit records (1150 ms)
      ✓ 4B: Rollback proof: real failure before commit ensures no partial shortage state leaked (1050 ms)
    5. R48 Repack Shortage Backend Seam
      ✓ 5A: blockSourceLocation = false terminates task in SHORT_PICKED without blocking source location (1120 ms)

Test Suites: 1 passed, 1 total
Tests:       11 passed, 11 total
Snapshots:   0 total
Time:        29.107 s
```

---

## 4. Real Rendered UI Playwright Evidence

### A. Supervisor Web UI E2E Test (`P1-007-shortages-supervisor-ui.spec.ts`)
* **Target URL:** `http://localhost:3009/backend/shortages`
* **Test Suite:** `apps/mercato/src/modules/wms_outbound/__integration__/P1-007-shortages-supervisor-ui.spec.ts`
* **Execution Result:** `1 passed (19.0s)`
* **Decisive Proof:** Real rendered React UI displayed both `SHORT_ALLOCATED` and `SHORT_PICKED` exception tables, allowed selecting supervisor action, submitted modal with reason audit, and verified PostgreSQL `wms_outbound_supervisor_decisions` row and `allow_partial_shipment` mutation.

### B. RF Scanner Short Pick E2E Test (`p1-007-real-scanner-short-pick.spec.ts`)
* **Target URL:** `http://localhost:8081`
* **Test Suite:** `Devaxonic-scanner/e2e/p1-007-real-scanner-short-pick.spec.ts`
* **Execution Result:** `1 passed (18.0s)`
* **Decisive Proof:** Operator authenticated, bound picking TU, reported short quantity (2 picked out of 5), verified `⚠️ Report Short Pick` UI action, validated task status `SHORT_PICKED` in PostgreSQL, verified `wms_outbound_location_shortages` recorded active block (`short_quantity = 3`, `is_active = true`), and validated `OutboundOrderLine` retry counter incremented.

---

## 5. Visual Evidence Artifacts

1. **Supervisor Shortages & Outcomes Console:**
   `WMS_Outbound/05_EVIDENCE/screenshots/p1-007-real-supervisor-shortages-ui.png`
2. **Scanner RF Picking Short Pick UI:**
   `WMS_Outbound/05_EVIDENCE/screenshots/p1-007-real-scanner-short-pick.png`
