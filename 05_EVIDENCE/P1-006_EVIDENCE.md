# P1-006 Execution & Acceptance Evidence

**Catalog Item:** `P1-006` — RF Scanner picking, PickTaskLine confirmation, continuation across consecutive zones and PickTask completion without auto-closing TU  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (P1 R15–R16, R55, R62, R67, FR-P1-08, FR-P1-30, FR-P1-36, FR-P1-41)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-008):** `9512137702a5d5f5b41910c2de97cf03321a1ccd`
* **Mercato P1-006 Final Head (`outbound/p1-006`):** `4274403d674a9481d0edc3c851989da49090aaed`
* **Scanner P1-006 Head (`main`):** `9022d6979a92c196e1333d97a979d9726a46e5e7`
* **Backend Build/Runtime Identity:** Live Next.js server on `http://127.0.0.1:3009` running from Mercato commit `4274403d674a9481d0edc3c851989da49090aaed`.

---

## 2. Decisive Lock-Wait & Blocking Assertion Proof (PostgreSQL Application Path)

### Genuine Overlap & Strict Assertions
* **Application Path Exercised:** Both Actor A and Actor B execute `createPickTaskService(txEm).confirmPickLine(...)` inside independent `em.transactional(...)` application transactions on separate PostgreSQL backend processes (`pidA` vs `pidB`).
* **Row-Level Write Locks:** `confirmPickLine` applies `LockMode.PESSIMISTIC_WRITE` (`SELECT ... FOR UPDATE`) to `WmsOutboundPickTask` and `WmsOutboundPickTaskLine`, ensuring strict serializable pick allocation confirmation semantics.
* **Strict Observer Condition:** Decisive proof is accepted ONLY when PostgreSQL `pg_blocking_pids(pidB)` directly reports `pidA` (`blocking.includes(pidA) === true`) AND `stat.wait_event_type === 'Lock'` (`lockMode: 'ShareLock'`, `lockType: 'transactionid'`). Zero manufactured fallbacks or PID inferences.
* **Strict Observer Assertions:**
  - `expect(pidA).toBeDefined()`
  - `expect(pidB).toBeDefined()`
  - `expect(pidA).not.toBe(pidB)`
  - `expect(lockEvidenceCaptured).toBe(true)`
  - `expect(capturedBlockingPids).toContain(pidA)`
  - `expect(capturedWaitEventType).toBe('Lock')`
  - `expect(actorBError).toContain('is in status "COMPLETED" and cannot accept picks')`

### Actual PostgreSQL Blocking Console Output (Verbatim):
```json
{
  "actorAPid": 1748672,
  "actorBPid": 1749458,
  "blockingPids": [ 1748672 ],
  "waitEventType": "Lock",
  "waitEvent": "transactionid",
  "lockType": "transactionid",
  "lockMode": "ShareLock",
  "actorAResult": "fulfilled",
  "actorBError": "PickTask \"2829e72c-3a2e-4868-8b30-f068dc72a79f\" is in status \"COMPLETED\" and cannot accept picks."
}
```

* **Outcome:** Actor A completes the picking line and transitions task to `COMPLETED`. When Actor A releases lock upon commit, Actor B immediately fails with domain invariant guard (`PickTask ... is in status "COMPLETED"`), preventing duplicate confirmation and over-picking.

---

## 3. Real Application Rollback Proof

* **Tested Scenario:** Application transaction failure (simulated error inside transaction block after incrementing picked quantities and writing TU content).
* **Proof Verified:**
  - `pickTask.status` remains `ASSIGNED`.
  - `pickTaskLine.pickedQuantity` remains `0.000000`.
  - `wms_outbound_tu_contents` table has 0 rows for the TU.
  - Zero partial state leak in remote Testing PostgreSQL.

---

## 4. Fail-Closed HTTP Warehouse Access Authorization Suite

* **Existing Primitive Reused:** `UserWarehouseService` / `wms_user_warehouses` assignment primitive via `assertUserWarehouseAccess(userWarehouseService, auth, warehouseId)`.
* **Endpoints Guarded:**
  - `POST /api/wms_outbound/picking/confirm-line`
  - `POST /api/wms_outbound/picking/close-tu`
  - `POST /api/wms_outbound/picking/switch-tu`
  - `GET /api/wms_outbound/picking/continuation-tasks?warehouseId=...&outboundOrderId=...`
  - `POST /api/wms_outbound/picking/continue-task`
* **Verified Scenarios (Genuine Route Invocations):**
  1. Operator assigned to Warehouse A only: Allowed in Warehouse A; **FAIL CLOSED with 403 Forbidden** in Warehouse B.
  2. Operator assigned to both Warehouse A and B: Allowed in both.
  3. Super Admin: Permitted across warehouses.
  4. Missing/invalid token: Fails with 401 Unauthorized.

---

## 5. Real Scanner UI Playwright Acceptance (Zero Route Mocks)

### Rendered Operator Journey:
1. **Interactive Login:** Operator logs in through the rendered form (`operator-p1006-ui-...@devaxonic.com`).
2. **Mode & Zone Selection:** Selects active warehouse and navigates to `Picking Module` -> `Ambient Storage (Zone A)`.
3. **Request Next Task:** Clicks **"Request Next Task"** button; receives `PT-A-...` in `ASSIGNED` status.
4. **Picking TU Creation (R53/R16):** Selects `TUSetup` chip, declares `directPackDeclared`, and clicks **"Create & Bind Picking TU"** button. TU transitions `CREATED -> IN_PICKING`.
5. **Real RF Item Scan & Confirmation (TC-001/004):** Enters/scans Location ID, SKU code, and quantity `10`, then clicks **"Confirm Pick"** button.
6. **R55 Invariant Verified:**
   - PickTask A transitions `IN_PROGRESS -> COMPLETED`.
   - Outbound TU **MUST NOT** auto-close. TU status remains `IN_PICKING` with mass updated to `20.00 kg`.
7. **R55 Next-Zone Continuation Offer (TC-109):** Scanner displays continuation banner: `Continue Order in Chilled Storage ... (1 lines)`.
8. **Continuation Action:** Operator clicks continuation button; seamlessly assigned to PickTask B in Zone B into the **SAME** open TU (`TU0000000008`).
9. **Zone B RF Confirmation:** Scans Location B, SKU B, and quantity `6`, then clicks **"Confirm Pick"**.
10. **Task B Completion & Cumulative TU Metrics:** Task B completes; cumulative TU mass updates to `29.00 kg` (20.00 kg + 6 * 1.5 kg).
11. **Explicit Operator TU Closure:** Operator clicks **"Close TU & Return to Zones"**; TU transitions `IN_PICKING -> READY_TO_PACK`, and UI returns to Picking Module zones.

### Screenshots Captured:
* Completed Zone A task with R55 open TU and continuation offer:  
  `WMS_Outbound/05_EVIDENCE/screenshots/p1-006-real-scanner-picking.png`
* Final state after closing TU and returning to zones:  
  `WMS_Outbound/05_EVIDENCE/screenshots/p1-006-real-scanner-continuation-closed.png`

### Test Command:
```bash
cd /home/ubuntu/git/Devaxonic-scanner
export PATH="/home/ubuntu/.nvm/versions/node/v24.19.0/bin:$PATH"
export NODE_TLS_REJECT_UNAUTHORIZED=0
export PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH="/home/ubuntu/.cache/ms-playwright/chromium-1228/chrome-linux64/chrome"
set -a && . /home/ubuntu/git/Devaxonic-mercato/apps/mercato/.env && set +a
npx playwright test e2e/p1-006-real-scanner-picking.spec.ts
```

### Playwright Output (Verbatim):
```
Running 1 test using 1 worker

     1 … persistence, Multi-Zone Continuation, and TU Closure (Zero Route Mocks)
[P1-006 Setup] Operator: operator-p1006-ui-428393@devaxonic.com (540fc47f-76fe-4816-a0f6-e1b8625e31e9)
[P1-006 Setup] Tasks: PT-A-428393 (Zone A) & PT-B-428393 (Zone B)
[P1-006 Assertion PASS] Real Scanner UI RF Picking & Multi-Zone Continuation verified end-to-end.
  ✓  1 …ence, Multi-Zone Continuation, and TU Closure (Zero Route Mocks) (13.6s)

  1 passed (17.7s)
```

---

## 6. PostgreSQL Integration & Regression Test Battery

### Test Commands:
```bash
cd /home/ubuntu/git/Devaxonic-mercato
export PATH="/home/ubuntu/.nvm/versions/node/v24.19.0/bin:$PATH"
export NODE_TLS_REJECT_UNAUTHORIZED=0
set -a && . apps/mercato/.env && set +a

# P1-006 Genuine PostgreSQL Suite (7 tests)
corepack yarn workspace @open-mercato/app test p1-006-postgres

# Full WMS Outbound Regression Battery (16 suites, 219 tests)
corepack yarn workspace @open-mercato/app test wms_outbound
```

### Results Summary:
* **`p1-006-postgres.integration.test.ts`:** 7 passed, 7 total (100% PASS)
* **`wms_outbound` Regression Battery:** 16 passed, 16 test suites total (219 passed, 219 total, 100% PASS)
