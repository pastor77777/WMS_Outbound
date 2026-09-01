# P1-008 Execution & Acceptance Evidence

**Catalog Item:** `P1-008` — Outbound TU identity, TUSetup, numbering, capacity and issueability  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (R53, R63, R64, R65, R66, R68, FR-P1-28, FR-P1-37, FR-P1-38, FR-P1-39, FR-P1-40, FR-P1-42)  
**Execution Date:** 2026-09-01  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-005):** `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975`
* **Mercato P1-008 Head (`outbound/p1-008`):** `bc8f66a12e98d52c9abe776c446484e957984464`
* **Scanner P1-008 Head (`main`):** `b32197b04aa00f94266340022eff369bae2ba269`
* **Backend Build/Runtime Identity:** Live Next.js server on `http://127.0.0.1:3009` running directly from Mercato commit `bc8f66a12e98d52c9abe776c446484e957984464`.

---

## 2. Playwright Verification (Zero Route Mocks)

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

  ✓ 1 [chromium] › e2e/p1-008-real-tu-identity.spec.ts:49:7 › P1-008 Real Playwright Outbound TU Identity, TUSetup & Issueability Suite (Zero Route Mocks) › P1-008 End-to-End Real Scanner & Backend Flow: UI Login -> TUSetup -> Outbound TU -> Contents -> Mass/Volume -> Issueability -> Override (8.6s)

  1 passed (10.7s)
```

### Real API Paths Exercised:
1. `POST /api/auth/login` (User interactive login on Scanner UI returning valid session token);
2. `POST /api/wms_outbound/tu-setups` (Creation of standard box and single EXTERNAL pallet setup);
3. `POST /api/wms_outbound/transport-units` (Atomic sequential generation of alphanumeric `TU_NUMBER` per warehouse);
4. `GET /api/wms_outbound/transport-units/[id]/issueability` (Authoritative issueability check evaluating R64 step 1 / step 2);
5. `POST /api/wms_outbound/transport-units/[id]/contents` (Dynamic content line addition and automatic mass/volume recalculation);
6. `POST /api/wms_outbound/transport-units/[id]/override-issueability` (Operator issueability override persisting reason and operator ID).

---

## 3. Genuine PostgreSQL Integration Test Battery

### Test Command:
```bash
cd /home/ubuntu/git/Devaxonic-mercato
export PATH="/home/ubuntu/.nvm/versions/node/v24.19.0/bin:$PATH"
export NODE_TLS_REJECT_UNAUTHORIZED=0
set -a && . apps/mercato/.env && set +a
corepack yarn workspace @open-mercato/app test p1-008
```

### Test Result:
```
Test Suites: 1 passed, 1 total
Tests:       12 passed, 12 total
Snapshots:   0 total
Time:        14.375 s
```

### Test Matrix & Acceptance Mapping:
* **TC-010 (R53 / FR-P1-28):** Generates unique Code 128 compliant alphanumeric `TU_NUMBER` (max 20 chars); blocks active duplicate numbers; concurrency proof with 10 parallel creations producing 10 unique sequential numbers without race conditions; enables number reuse after previous TU reaches terminal status (`DISPATCHED`).
* **TC-110, TC-111 (R63 / FR-P1-37):** Calculates `currentWeight = sum(unitWeight * quantity)` and `currentContentVolume = sum(unitVolume * quantity)` from line contents; persists decimal metrics on `wms_outbound_transport_units`.
* **TC-114 (R64 step 1 / R68 / FR-P1-38, FR-P1-40):** EXTERNAL type automatically satisfies issue thresholds by definition (`EXTERNAL_TYPE_AUTO_ISSUABLE`); lower thresholds are not read or evaluated.
* **TC-115 (R64 step 2 / FR-P1-38, FR-P1-42):** Standard types evaluate `externalIssuable = true` and lower thresholds (`currentWeight >= minIssueWeight OR currentContentVolume >= minIssueVolume`).
* **TC-116 (R66):** Packing TU created through repack is blocked from sealing when `TUSetup.externalIssuable = false`.
* **TC-117 (R65 / FR-P1-39):** Operator override is strictly blocked when `TUSetup.externalIssuable = false` (absolute block).
* **TC-120 (R65 / FR-P1-39):** Operator override is not applicable for EXTERNAL TU types.
* **R68 DB Constraint:** Warehouse permits exactly one `TUSetup` with `process_usage = 'EXTERNAL'`, enforced via PostgreSQL partial unique index `wms_outbound_tu_setups_external_uq`.

---

## 4. Inbound & Shared Compatibility Regression

* **Full WMS Outbound Suite:** `15/15` suites passed, `202/202` tests passed.
* **FND-003 Shared Compatibility Suite:** `8/8` passed (`fnd-003-postgres.integration.test.ts`).
* **WMS Receiving Suite:** `10/10` suites passed, `84/84` tests passed.
* **Shared TU Entity Integrity:** `wms_transport_units` table and Inbound `process_status` semantics remain unmodified.
