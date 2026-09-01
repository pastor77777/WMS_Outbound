# P1-005 Execution & Acceptance Evidence

**Catalog Item:** `P1-005` — PickTask creation, zone ordering and operator assignment  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (Step 5 Task Generation, R13, R14, R54, R56 & Step 6 Continuation Primitive FR-P1-30 / R55, CON-02)  
**Execution Date:** 2026-09-01  
**Status:** **FINAL PASS**  
**Final Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-004):** `71b74b5384b3fbfc55d8ed298f1a3715dc477c3c`
* **Mercato P1-005 Commit (`outbound/p1-005`):** `d2709c6fb52ecf869e669bd397296c8261399ace`
* **Scanner P1-005 Commit (`main`):** `ee902f9c2e8643ee67eb83746a5fb5a21e95fbb9`

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
1. `POST http://127.0.0.1:3009/api/auth/login` -> HTTP 200 (Real JWT token issued for operator)
2. `GET http://127.0.0.1:3009/api/wms_warehouse/my-warehouses` -> HTTP 200 (Active warehouse: `DevAxonic Fresh Distribution Centre`)
3. `GET http://127.0.0.1:3009/api/wms_outbound/picking/zones?warehouseId=9b174240-2376-4a5b-8d5a-5f3271e347d7` -> HTTP 200 (Permitted zones: `Ambient Storage`, `Quality/Claims Zone`, `Vendor-Return Staging`, `TEST - Sector B`)
4. `POST http://127.0.0.1:3009/api/wms_outbound/picking/request-task` -> HTTP 200 (Task assigned: `PT-E2E-275944`, status: `ASSIGNED`)
5. `POST http://127.0.0.1:3009/api/wms_outbound/picking/request-task` (Second attempt with active task) -> HTTP 400 (`Operator already has an active warehouse task ...`)

### Execution Output:
```
[WebServer] 127.0.0.1 - - "GET / HTTP/1.1" 200 -
Running 1 test using 1 worker

[P1-005 Setup] Operator: operator-p1005-e2e-275944@devaxonic.com (f5b9548f-5494-432b-9e35-38f36dd8a49b)
[P1-005 Setup] Fixture Task: PT-E2E-275944 (1a6a6afe-25b8-4856-b0dc-248effb17b91)
[API RESP] 200 http://127.0.0.1:3009/api/auth/login
[API RESP] 200 http://127.0.0.1:3009/api/wms_warehouse/my-warehouses
[API RESP] 200 http://127.0.0.1:3009/api/wms_outbound/picking/zones?warehouseId=9b174240-2376-4a5b-8d5a-5f3271e347d7
[API RESP] 200 http://127.0.0.1:3009/api/wms_outbound/picking/request-task
[P1-005 Assertion PASS] Persisted Task: PT-E2E-275944 -> status: ASSIGNED, operator: operator, assignedAt: 2026-09-01T20:58:05.328Z
[API RESP] 200 http://127.0.0.1:3009/api/wms_outbound/picking/zones?warehouseId=9b174240-2376-4a5b-8d5a-5f3271e347d7
[API RESP] 400 http://127.0.0.1:3009/api/wms_outbound/picking/request-task BODY: {"error":"Operator \"operator\" already has an active warehouse task \"1a6a6afe-25b8-4856-b0dc-248effb17b91\" (status: ASSIGNED)."}
[P1-005 Assertion PASS] Active task guard (R56) visibly verified on Scanner UI.
  ✓ 1 Real Scanner operator login, zone fetch, task assignment, and PostgreSQL state verification (11.3s)

  1 passed (13.9s)
```

---

## 3. Persisted PickTask and State Transition Proof

Direct PostgreSQL query executed in the test against Testing PostgreSQL confirms:
* **Table:** `wms_outbound_pick_tasks`
  * `id`: `1a6a6afe-25b8-4856-b0dc-248effb17b91`
  * `task_number`: `PT-E2E-275944`
  * `status`: `ASSIGNED`
  * `operator_id`: `operator`
  * `warehouse_id`: `9b174240-2376-4a5b-8d5a-5f3271e347d7`
  * `zone_id`: `af1103d4-7057-40ba-a1b8-b91d539a37b0`
  * `assigned_at`: `2026-09-01T20:58:05.328Z`
* **Table:** `wms_outbound_state_transition_events`
  * `entity_id`: `1a6a6afe-25b8-4856-b0dc-248effb17b91`
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
* P1-005 implementation and real Playwright evidence remediation is **COMPLETE**.
* P1-006 (RF scanning picking execution) has **NOT** been started.
