# P1-008 Execution & Acceptance Evidence

**Catalog Item:** `P1-008` — Outbound TU identity, TUSetup, numbering, capacity and issueability  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (R53, R63, R64, R65, R66, R68, FR-P1-28, FR-P1-37, FR-P1-38, FR-P1-39, FR-P1-40, FR-P1-42)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-005):** `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975`
* **Mercato P1-008 Previous Head:** `cb690079ce1e55cf89b4511f0762f0ecbf3dcd86`
* **Mercato P1-008 Final Remediation Head (`outbound/p1-008`):** `b51a201e3a0d861d21fe4a0b5fa5040870fd7933`
* **Scanner P1-008 Head (`main`):** `b5cfb59987c76f39e0ab48af67a52e2e914d9613`
* **Backend Build/Runtime Identity:** Live Next.js server on `http://127.0.0.1:3009` running from Mercato commit `b51a201e3a0d861d21fe4a0b5fa5040870fd7933`.

---

## 2. Decisive Lock-Wait & Blocking Assertion Proof (PostgreSQL Application Path)

### Genuine Overlap & Strict Assertions
* **Application Path Exercised:** Both Actor A and Actor B execute `createOutboundTuService(txEm).createOutboundTu(...)` inside independent `em.transactional(...)` application transactions on separate PostgreSQL backend processes (`pidA` vs `pidB`).
* **Sequence Lock Execution Context:** Fixed `generateTuNumber` sequence upsert to execute directly on `em` rather than raw pooled driver connection, ensuring row-level lock is held for the duration of Actor A's transaction.
* **Strict Observer Assertions:**
  - `expect(pidA).toBeDefined()`
  - `expect(pidB).toBeDefined()`
  - `expect(pidA).not.toBe(pidB)`
  - `expect(lockEvidenceCaptured).toBe(true)`
  - `expect(capturedBlockingPids).toContain(pidA)`
* **Captured PostgreSQL Blocking Proof:**
```json
{
  "actorAPid": 1731190,
  "actorBPid": 1731191,
  "blockingPids": [ 1731190 ],
  "waitEventType": "Lock",
  "waitEvent": "transactionid",
  "lockType": "transactionid",
  "lockMode": "ShareLock",
  "tuANumber": "TU0000000001",
  "tuBNumber": "TU0000000002"
}
```
* **Release & Sequence Resolution:** Only after PostgreSQL captures `pg_blocking_pids(pidB)` containing `pidA` and `wait_event_type = 'Lock'` is Actor A released. Both transactions then commit without deadlock or sequence gap, producing consecutive collision-free numbers `TU0000000001` and `TU0000000002`.
* **Burst Concurrency (10 Actors):** 10 concurrent application service invocations in parallel produce 10 unique sequential numbers (`1..10`), verified with fresh independent DB read (`wms_outbound_tu_sequences.last_value = 10`, `count = 10`).

---

## 3. Warehouse Access Authorization at HTTP Boundary

### Existing Primitive Reused
* Reused existing `UserWarehouseService` / `wms_user_warehouses` assignment primitive through `assertUserWarehouseAccess(userWarehouseService, auth, warehouseId)`.
* Applied to all P1-008 HTTP route entrypoints:
  - `POST /api/wms_outbound/transport-units`
  - `GET /api/wms_outbound/transport-units?warehouseId=...`
  - `POST /api/wms_outbound/tu-setups`
  - `GET /api/wms_outbound/tu-setups?warehouseId=...`
  - `POST /api/wms_outbound/transport-units/[id]/contents`
  - `POST /api/wms_outbound/transport-units/[id]/override-issueability`
  - `GET /api/wms_outbound/transport-units/[id]/issueability`

### Verified Scenarios (Genuine HTTP Route Invocations):
1. **User A assigned to Warehouse A only:**
   - Authorized operations in Warehouse A (`POST/GET tu-setups`, `POST/GET transport-units`, `POST contents`, `GET issueability`, `POST override-issueability`) return **201 / 200**.
   - Unauthorized attempts targeting Warehouse B (`POST/GET tu-setups`, `POST/GET transport-units`, mutating/reading Warehouse B TU `[id]/contents`, `[id]/override-issueability`, `[id]/issueability`) **FAIL CLOSED with 403 Forbidden** (`User is not assigned to warehouse "..."`).
2. **User AB assigned to both Warehouse A and Warehouse B:**
   - Operations in Warehouse B return **201 / 200**.
3. **Super Admin User:**
   - Permitted across warehouses without explicit individual `wms_user_warehouses` assignment row.
4. **Tenant/Org Isolation:**
   - Cross-tenant/unauthenticated attempts fail with 401/403.

---

## 4. Real Scanner UI Playwright Acceptance (Zero Route Mocks)

### Rendered Operator Journey:
1. **Interactive Login:** Operator logs in through the rendered form (`operator-p1008-ui-...@devaxonic.com`).
2. **Mode Selection:** Selects active warehouse and navigates to `outboundTu` mode screen.
3. **TU Creation via Rendered UI:** Selects `TUSetup` chip and clicks **"Create Outbound TU"** button.
4. **Alphanumeric TU_NUMBER Inspection:** Inspects generated `TU_NUMBER` (`TU0000000006`) and initial `BELOW_THRESHOLDS` badge on screen.
5. **Item Master Content Addition:** Inputs SKU code and quantity into rendered text fields and clicks **"Add Item to TU"** button. The server authoritatively looks up `MasterdataItem` dimensions (`weight: 5.0 kg`, `volume: 0.05 m³`), computing `currentWeight: 10.000000 kg` and `currentContentVolume: 0.100000 m³` rendered on screen.
6. **Operator Issueability Override:** Types reason (`Urgent order before SLA cutoff`) into rendered field and clicks **"Apply Issueability Override"** button.
7. **UI Confirmation & Operator Binding:** UI transitions to `OPERATOR_OVERRIDE_APPLIED` badge displaying authenticated operator UUID.
8. **Real PostgreSQL Verification:** Independent DB connection confirms:
   - `wms_outbound_transport_units.tu_number === 'TU0000000006'`
   - `wms_outbound_transport_units.override_by === testUserId` (authenticated operator UUID, not trusted from request body)
   - `wms_outbound_transport_units.override_reason === 'Urgent order before SLA cutoff'`
   - `wms_outbound_tu_contents.unit_weight === 5.000000`, `unit_volume === 0.050000` (exact item master correlation).

### Test Command:
```bash
cd /home/ubuntu/git/Devaxonic-scanner
export PATH="/home/ubuntu/.nvm/versions/node/v24.19.0/bin:$PATH"
export NODE_PATH="/home/ubuntu/git/Devaxonic-mercato/node_modules"
export NODE_TLS_REJECT_UNAUTHORIZED=0
set -a && . /home/ubuntu/git/Devaxonic-mercato/apps/mercato/.env && set +a
/home/ubuntu/git/Devaxonic-mercato/node_modules/.bin/playwright test e2e/p1-008-real-tu-identity.spec.ts
```

### Test Result:
```
Running 1 test using 1 worker

  ✓ 1 [chromium] › e2e/p1-008-real-tu-identity.spec.ts:110:7 › P1-008 Real Playwright Outbound TU Identity, TUSetup & Issueability Suite (Zero Route Mocks) › Real Scanner UI Operator Journey: Login -> Select Outbound TU Mode -> Create TU -> Add SKU Content -> View Mass/Volume -> Apply Override -> DB Verification (14.2s)

  1 passed (17.0s)
```

---

## 5. Test Verification Matrix & Pass Counts

| Test Suite | Command | Result | Pass Count |
|---|---|---|---|
| **P1-008 PostgreSQL Suite** | `yarn workspace @open-mercato/app test p1-008` | PASSED | **22 / 22** |
| **Real Scanner UI Playwright** | `playwright test e2e/p1-008-real-tu-identity.spec.ts` | PASSED | **1 / 1** |
| **Outbound Module Battery** | `yarn workspace @open-mercato/app test wms_outbound` | PASSED | **212 / 212** (15 suites) |
| **FND-003 Compatibility** | `yarn workspace @open-mercato/app test fnd-003` | PASSED | **16 / 16** (2 suites) |
| **WMS Receiving Regression** | `yarn workspace @open-mercato/app test wms_receiving` | PASSED | **84 / 84** (10 suites) |

---

## 6. Remaining Gaps & Stop Boundary

* **Remaining Gaps:** None for P1-008.
* **Stop Boundary:** P1-008 narrow remediation is complete and verified. No work on P1-006 has been started. Execution has halted cleanly.
