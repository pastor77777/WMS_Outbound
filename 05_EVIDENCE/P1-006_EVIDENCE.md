# P1-006 Execution & Acceptance Evidence (Remediation Complete)

**Catalog Item:** `P1-006` — RF Scanner picking, PickTaskLine confirmation, continuation across consecutive zones and PickTask completion without auto-closing TU  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (P1 R15–R16, R55, R62, R67, FR-P1-08, FR-P1-30, FR-P1-36, FR-P1-41)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-008):** `9512137702a5d5f5b41910c2de97cf03321a1ccd`
* **Mercato P1-006 Remediation Head (`outbound/p1-006`):** `dbc8ddef2873d6babde73a1ecd854c5feb1dff0e`
* **Scanner P1-006 Remediation Head (`main`):** `7d36f85ab5cf8020746d9a007fe1d8a5795980ae`
* **Authoritative Outbound Head (`main`):** `WMS_Outbound` synchronized with remote `main`.
* **Backend Runtime:** Live Next.js server on `http://127.0.0.1:3009` running from Mercato commit `dbc8ddef2873d6babde73a1ecd854c5feb1dff0e`.

---

## 2. Remediation Defects Addressed & Proved

### Defect 1: Server-Authoritative Pick Idempotency
* **Database Constraint:** Table `wms_outbound_pick_confirmations` created with unique index `(organization_id, tenant_id, idempotency_key)`.
* **Behavior:**
  - Sequential replay with identical idempotency key returns the previously committed confirmation snapshot without second DB write, double quantity increment, or duplicate `WmsOutboundTuContent` rows.
  - Conflicting payload with the same key fails closed with 422 / error (`Idempotency key ... already used with conflicting payload`).
  - Concurrent overlapping transactions are serialized and deduplicated at the database constraint boundary.

### Defect 2: R55 Operator Active Zone Dynamic Update
* **Scanner UI State:** Upon clicking the same-order multi-zone continuation offer (Zone A -> Zone B), `currentZone` in state dynamically updates to Zone B (`Chilled Storage`).
* **Rendered UI & Pick Execution:** The active header and form display Zone B, and subsequent picks are executed in Zone B into the retained open TU.

### Defect 3: R62 Configurable Picking-TU Strategy
* **Configuration:** Added `picking_tu_strategy` (`SHARED_SAME_ORDER_CONSECUTIVE_ZONES` vs `SEPARATE_PER_TASK_OR_ZONE`) to `wms_outbound_warehouse_queue_configs`.
* **Validation:** Genuine PostgreSQL integration tests prove:
  - `SHARED_SAME_ORDER_CONSECUTIVE_ZONES`: Continuation retains the open TU (`requiresNewTu: false`).
  - `SEPARATE_PER_TASK_OR_ZONE`: Continuation requires creating / binding a separate TU (`requiresNewTu: true`).

### Defect 4: R67 Capacity-Driven TU Switch
* **Explicit Full State:** When a TU reaches physical/mass capacity or when the operator determines no more items fit, `declareTuFull` / `POST /api/wms_outbound/picking/declare-tu-full` transitions the TU to `PICK_FULL` durably.
* **Switch Action:** `switchPickingTuForTask` / `POST /api/wms_outbound/picking/switch-tu` closes the full TU to `READY_TO_PACK` and creates a new TU for the same active PickTask.

### Defect 5: Direct Pack (P1-009) Scope Removed
* Removed Direct Pack toggle and related scope from the P1-006 Scanner UI and test assertions.

### Defect 6: Canonical Operator Identity Enforcement
* Removed all `'operator'` fallback strings.
* Endpoints (`confirm-line`, `close-tu`, `switch-tu`, `continue-task`, `request-task`, `declare-tu-full`) fail closed with 401 Unauthorized if `auth.sub || auth.userId` is missing or resolves to `'operator'`.

---

## 3. Decisive PostgreSQL Concurrency & Lock Proof

### Strict Observer Proof (PostgreSQL Application Path)
* **Application Path:** Concurrently executing `createPickTaskService(txEm).confirmPickLine(...)` across two distinct database connections/processes (`pidA` vs `pidB`).
* **Strict Observer Assertions:**
  - `expect(pidA).toBeDefined()`
  - `expect(pidB).toBeDefined()`
  - `expect(pidA).not.toBe(pidB)`
  - `expect(lockEvidenceCaptured).toBe(true)`
  - `expect(capturedBlockingPids).toContain(pidA)`
  - `expect(capturedWaitEventType).toBe('Lock')`
  - `expect(actorBError).toMatch(/(is in status "COMPLETED"|already fully picked)/)`

### Actual PostgreSQL Concurrency Proof Log (Verbatim):
```json
{
  "actorAPid": 1758829,
  "actorBPid": 1758775,
  "blockingPids": [ 1758829 ],
  "waitEventType": "Lock",
  "waitEvent": "transactionid",
  "lockType": "transactionid",
  "lockMode": "ShareLock",
  "actorAResult": "fulfilled",
  "actorBError": "PickTask \"52a4bdf9-f8f4-400f-879d-677ba3560763\" is in status \"COMPLETED\" and cannot accept picks."
}
```

---

## 4. Real Application Rollback Proof

* **Tested Scenario:** Application transaction failure (simulated error inside transaction block after incrementing picked quantities and writing TU content).
* **Proof Verified:**
  - `pickTask.status` remains `ASSIGNED`.
  - `pickTaskLine.pickedQuantity` remains `0.000000`.
  - `wms_outbound_tu_contents` table has 0 rows for the TU.
  - Zero partial state leak in remote Testing PostgreSQL.

---

## 5. Real Scanner UI Playwright Acceptance (Zero Route Mocks)

### Rendered Operator Journey:
1. **Interactive Login:** Operator logs in through the rendered form (`operator-p1006-ui-...@devaxonic.com`).
2. **Mode & Zone Selection:** Selects active warehouse and navigates to `Picking Module` -> `Ambient Storage (Zone A)`.
3. **Request Next Task:** Clicks **"Request Next Task"** button; receives `PT-A-...` in `ASSIGNED` status.
4. **Picking TU Creation (R53):** Selects `TUSetup` chip and clicks **"Create & Bind Picking TU"** button. TU transitions `CREATED -> IN_PICKING`.
5. **Real RF Item Scan & Confirmation (TC-001/004):** Enters/scans Location ID, SKU code, and quantity `10`, then clicks **"Confirm Pick"** button.
6. **R55 Invariant Verified:**
   - PickTask A transitions `IN_PROGRESS -> COMPLETED`.
   - Outbound TU **MUST NOT** auto-close. TU status remains `IN_PICKING` with mass updated to `20.00 kg`.
7. **R55 Next-Zone Continuation Offer (TC-109):** Scanner displays continuation banner: `Continue Order in Chilled Storage ...`.
8. **Continuation Action:** Operator clicks continuation button; seamlessly assigned to PickTask B in Zone B into the **SAME** open TU (`TU0000000013`), and active zone updates to `Chilled Storage`.
9. **Zone B RF Confirmation:** Scans Location B, SKU B, and quantity `6`, then clicks **"Confirm Pick"**.
10. **Task B Completion & Cumulative TU Metrics:** Task B completes; cumulative TU mass updates to `29.00 kg` (20.00 kg + 6 * 1.5 kg).
11. **Explicit Operator TU Closure:** Operator clicks **"Close TU & Return to Zones"**; TU transitions `IN_PICKING -> READY_TO_PACK`, and UI returns to Picking Module zones.

### Screenshots:
* **Completed Zone A task with R55 open TU and continuation offer:**  
  `05_EVIDENCE/screenshots/p1-006-real-scanner-picking.png`

* **Closed TU returning to zones:**  
  `05_EVIDENCE/screenshots/p1-006-real-scanner-continuation-closed.png`

---

## 6. Test Suite Results

1. **P1-006 Genuine PostgreSQL Test Suite:**
   - Command: `corepack yarn workspace @open-mercato/app test p1-006-postgres`
   - Result: **9 passed, 9 total (100% pass)**
2. **P1-006 Zero-Route-Mock Playwright E2E Test:**
   - Command: `npx playwright test e2e/p1-006-real-scanner-picking.spec.ts`
   - Result: **1 passed, 1 total (100% pass)**
3. **Full WMS Outbound Regression Battery:**
   - Command: `corepack yarn workspace @open-mercato/app test wms_outbound`
   - Result: **16 test suites passed, 221 tests passed (100% pass)**

---

## 7. Traceability Matrix

| Requirement / Rule | Verification Point | Status |
| :--- | :--- | :--- |
| **P1 R15 (TU Capacity)** | Mass/volume validation on item add, capacity rejection, and TU full handling | **VERIFIED** |
| **P1 R16 (Direct Pack)** | Direct pack toggle removed from P1-006 scope; deferred to P1-009 | **VERIFIED** |
| **R55 (Multi-Zone Continuation)** | Same-order continuation across consecutive zones in same open TU | **VERIFIED** |
| **R62 (TU Strategy)** | Warehouse-configurable picking TU strategy (`SHARED` vs `SEPARATE`) | **VERIFIED** |
| **R67 (Capacity-Driven Switch)** | Durable `PICK_FULL` declaration and switch to new TU for active task | **VERIFIED** |
| **FR-P1-08 / FR-P1-30** | RF Picking task execution & multi-zone continuation | **VERIFIED** |
| **FR-P1-36 / FR-P1-41** | Pick line confirmation & TU assignment | **VERIFIED** |
| **Pick Idempotency** | Server-authoritative idempotency key unique constraint & replay | **VERIFIED** |
| **Canonical Security** | Fail-closed warehouse access & strict UUID operator check | **VERIFIED** |
