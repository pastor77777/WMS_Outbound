# P1-005 Execution & Acceptance Evidence

**Catalog Item:** `P1-005` — PickTask creation, zone ordering and operator assignment  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (Step 5 Task Generation, R13, R14, R54, R56 & Step 6 Continuation Primitive FR-P1-30 / R55, CON-02)  
**Execution Date:** 2026-09-01  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-004):** `71b74b5384b3fbfc55d8ed298f1a3715dc477c3c`
* **Mercato P1-005 Head (`outbound/p1-005`):** `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975`
* **Scanner P1-005 Head (`main`):** `8199b330cb739a45e2c615a3f2aa3803336be724`
* **Backend Build/Runtime Identity:** Live Next.js server on `http://127.0.0.1:3009` running directly from Mercato commit `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975`.

---

## 2. Playwright Verification (Zero Route Mocks)

### Test Command:
```bash
cd /home/ubuntu/git/Devaxonic-scanner
export PATH="/home/ubuntu/.nvm/versions/node/v24.19.0/bin:$PATH"
export NODE_PATH="/home/ubuntu/git/Devaxonic-mercato/node_modules"
export NODE_TLS_REJECT_UNAUTHORIZED=0
set -a && . /home/ubuntu/git/Devaxonic-mercato/apps/mercato/.env && set +a
/home/ubuntu/git/Devaxonic-mercato/node_modules/.bin/playwright test --config=playwright.config.ts e2e/p1-005-real-scanner-assignment.spec.ts
```

### Real API Path & Network Confirmation:
The browser executed real HTTP calls against the live Next.js backend on `http://127.0.0.1:3009`:
1. `POST http://127.0.0.1:3009/api/auth/login` -> HTTP 200 (Real JWT token issued for operator `operator-p1005-e2e-758475@devaxonic.com` with `sub = 2debb728-e8cc-4f3f-886b-4bfe32c3746e`)
2. `GET http://127.0.0.1:3009/api/wms_warehouse/my-warehouses` -> HTTP 200 (Active warehouse: `DevAxonic Fresh Distribution Centre`)
3. `GET http://127.0.0.1:3009/api/wms_outbound/picking/zones?warehouseId=9b174240-2376-4a5b-8d5a-5f3271e347d7` -> HTTP 200 (Permitted zones: `Ambient Storage`, `Quality/Claims Zone`, `Vendor-Return Staging`, `TEST - Sector B`)
4. `POST http://127.0.0.1:3009/api/wms_outbound/picking/request-task` -> HTTP 200 (Task assigned: `PT-E2E-758475` / `843c5cc9-5171-459d-b4e2-371a57704a57`, status: `ASSIGNED`, `operatorId = 2debb728-e8cc-4f3f-886b-4bfe32c3746e`)
5. `POST http://127.0.0.1:3009/api/wms_outbound/picking/request-task` (Second attempt by same operator) -> HTTP 400 (`Operator "2debb728-e8cc-4f3f-886b-4bfe32c3746e" already has an active warehouse task "843c5cc9-5171-459d-b4e2-371a57704a57" (status: ASSIGNED).`)

### Execution Output:
```
[WebServer] 127.0.0.1 - - "GET / HTTP/1.1" 200 -
Running 1 test using 1 worker

[P1-005 Setup] Operator: operator-p1005-e2e-758475@devaxonic.com (2debb728-e8cc-4f3f-886b-4bfe32c3746e)
[P1-005 Setup] Fixture Task: PT-E2E-758475 (843c5cc9-5171-459d-b4e2-371a57704a57)
[API RESP] 200 http://127.0.0.1:3009/api/auth/login BODY: {"ok":true,"token":"...","sub":"2debb728-e8cc-4f3f-886b-4bfe32c3746e",...}
[API RESP] 200 http://127.0.0.1:3009/api/wms_warehouse/my-warehouses BODY: {"active":{"id":"9b174240-2376-4a5b-8d5a-5f3271e347d7","name":"DevAxonic Fresh Distribution Centre"},...}
[API RESP] 200 http://127.0.0.1:3009/api/wms_outbound/picking/zones?warehouseId=9b174240-2376-4a5b-8d5a-5f3271e347d7 BODY: {"items":[{"id":"af1103d4-7057-40ba-a1b8-b91d539a37b0","name":"Ambient Storage",...},...]}
[API RESP] 200 http://127.0.0.1:3009/api/wms_outbound/picking/request-task BODY: {"task":{"id":"843c5cc9-5171-459d-b4e2-371a57704a57","taskNumber":"PT-E2E-758475","warehouseId":"9b174240-2376-4a5b-8d5a-5f3271e347d7","zoneId":"af1103d4-7057-40ba-a1b8-b91d539a37b0","status":"ASSIGNED","operatorId":"2debb728-e8cc-4f3f-886b-4bfe32c3746e","assignedAt":"2026-09-01T21:22:52.253Z",...}}
[P1-005 Assertion PASS] Persisted Task: PT-E2E-758475 -> status: ASSIGNED, operator: 2debb728-e8cc-4f3f-886b-4bfe32c3746e, assignedAt: Tue Sep 01 2026 21:22:52 GMT+0000
[API RESP] 200 http://127.0.0.1:3009/api/wms_outbound/picking/zones?warehouseId=9b174240-2376-4a5b-8d5a-5f3271e347d7
[API RESP] 400 http://127.0.0.1:3009/api/wms_outbound/picking/request-task BODY: {"error":"Operator \"2debb728-e8cc-4f3f-886b-4bfe32c3746e\" already has an active warehouse task \"843c5cc9-5171-459d-b4e2-371a57704a57\" (status: ASSIGNED)."}
[P1-005 Assertion PASS] Active task guard (R56) visibly verified on Scanner UI.
  ✓ 1 Real Scanner operator login, zone fetch, task assignment, and PostgreSQL state verification (16.7s)

  1 passed (19.8s)
```

---

## 3. Persisted PickTask and State Transition Proof

Direct PostgreSQL query executed in the test against Testing PostgreSQL confirms:
* **Table:** `wms_outbound_pick_tasks`
  * `id`: `843c5cc9-5171-459d-b4e2-371a57704a57`
  * `task_number`: `PT-E2E-758475`
  * `status`: `ASSIGNED`
  * `operator_id`: `2debb728-e8cc-4f3f-886b-4bfe32c3746e` (Strict equality: `expect(dbTask.operator_id).toBe(testUserId)` -> PASS)
  * `warehouse_id`: `9b174240-2376-4a5b-8d5a-5f3271e347d7`
  * `zone_id`: `af1103d4-7057-40ba-a1b8-b91d539a37b0`
  * `assigned_at`: `2026-09-01T21:22:52.253Z`
* **Table:** `wms_outbound_state_transition_events`
  * `entity_id`: `843c5cc9-5171-459d-b4e2-371a57704a57`
  * `entity_type`: `PickTask`
  * `from_status`: `CREATED`
  * `to_status`: `ASSIGNED`
  * `domain_event`: `PickTaskAssigned`
  * `actor_id`: `System WMS`

---

## 4. Screenshot & Artifact Paths

* **Assignment Screen Screenshot:** `05_EVIDENCE/screenshots/p1-005-real-scanner-assignment.png` (Absolute: `/home/ubuntu/git/WMS_Outbound/05_EVIDENCE/screenshots/p1-005-real-scanner-assignment.png`)
* **Active Task Guard Screen Screenshot:** `05_EVIDENCE/screenshots/p1-005-real-scanner-active-task-block.png` (Absolute: `/home/ubuntu/git/WMS_Outbound/05_EVIDENCE/screenshots/p1-005-real-scanner-active-task-block.png`)
* **E2E Spec:** `Devaxonic-scanner/e2e/p1-005-real-scanner-assignment.spec.ts`

---

## 5. PostgreSQL Integration & Regression Battery

| Test Suite | Result | Details |
|---|---|---|
| `p1-005-postgres.integration.test.ts` | **10 / 10 PASS** | Case A (Multi-Zone Creation), Cases B & C (Queue Ordering Modes), Case D (Advisory Concurrency), Case E (Continuation Primitive), Case F (Single Active Task Guard), Case G (CON-02 Immutability Guard) |
| `p1-004-postgres.integration.test.ts` | **11 / 11 PASS** | Allocation lifecycle, hard inventory reservations, ATP exclusion |
| `p1-003-postgres.integration.test.ts` | **12 / 12 PASS** | Outbound order planning & ATP demand |
| `p1-002-postgres.integration.test.ts` | **15 / 15 PASS** | Customer order acceptance & soft ATP reservations |
| `p1-001-postgres.integration.test.ts` | **7 / 7 PASS** | Core order creation & tenant isolation |
| `wms_inventory/` (Shared Inventory) | **26 / 26 PASS** | Inventory balances, movements, ledger constraints |

---

## 6. Stop Boundary Confirmation
* P1-005 identity and runtime evidence remediation is complete.
* P1-006 (RF scanning picking execution) has **NOT** been started.
