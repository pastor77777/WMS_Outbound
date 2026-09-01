# P1-008 Execution & Acceptance Evidence

**Catalog Item:** `P1-008` — Outbound TU identity, TUSetup, numbering, capacity and issueability  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (R53, R63, R64, R65, R66, R68, FR-P1-28, FR-P1-37, FR-P1-38, FR-P1-39, FR-P1-40, FR-P1-42)  
**Execution Date:** 2026-09-01  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-005):** `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975`
* **Mercato P1-008 Remediation Head (`outbound/p1-008`):** `2a8fbb26774c562e6f97398e0f27e3ff1e830607`
* **Scanner P1-008 Remediation Head (`main`):** `b5cfb59987c76f39e0ab48af67a52e2e914d9613`
* **Backend Build/Runtime Identity:** Live Next.js server on `http://127.0.0.1:3009` running directly from Mercato commit `2a8fbb26774c562e6f97398e0f27e3ff1e830607`.

---

## 2. Real Playwright Acceptance Run (Zero Route Mocks)

### Real Rendered UI Journey:
1. **Interactive Operator Login:** Operator signs into Scanner web interface with credentials (`operator-p1008-ui-...@devaxonic.com`).
2. **Mode Navigation:** Selects warehouse and navigates into `outboundTu` mode screen.
3. **TU Creation via Rendered UI:** Selects `TUSetup` chip and clicks rendered **"Create Outbound TU"** button (`page.getByText('Create Outbound TU')`).
4. **Alphanumeric TU_NUMBER Inspection:** Inspects generated `TU_NUMBER` (matching `^TU\d{10}$`) and initial `BELOW_THRESHOLDS` badge on screen.
5. **Item Master Content Addition:** Inputs SKU code and quantity into rendered text fields and clicks **"Add Item to TU"** button. The server authoritatively looks up `MasterdataItem` dimensions (`weight: 5.0 kg`, `volume: 0.05 m³`), computing `currentWeight: 10.000000 kg` and `currentContentVolume: 0.100000 m³` rendered on screen.
6. **Operator Issueability Override:** Types reason (`Urgent order before SLA cutoff`) into rendered field and clicks **"Apply Issueability Override"** button.
7. **UI Confirmation & Operator Binding:** UI transitions to `OPERATOR_OVERRIDE_APPLIED` badge displaying authenticated operator UUID.
8. **Real PostgreSQL Verification:** Independent DB connection confirms:
   - `wms_outbound_transport_units.tu_number === generatedTuNumber`
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

  ✓ 1 [chromium] › e2e/p1-008-real-tu-identity.spec.ts:110:7 › P1-008 Real Playwright Outbound TU Identity, TUSetup & Issueability Suite (Zero Route Mocks) › Real Scanner UI Operator Journey: Login -> Select Outbound TU Mode -> Create TU -> Add SKU Content -> View Mass/Volume -> Apply Override -> DB Verification (7.6s)

  1 passed (10.0s)
```

### Artifact Screenshot:
* **Rendered UI Screenshot:** `/home/ubuntu/p1-008-real-scanner-tu.png`

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
Tests:       15 passed, 15 total
Snapshots:   0 total
Time:        15.343 s
```

### Detailed Acceptance Matrix:
* **1. TUSetup & Exactly-One EXTERNAL Semantics (R68 / FR-P1-40, FR-P1-42):**
  - Valid INTERNAL setup creation with limits and issue thresholds (`maxWeight`, `maxVolume`, `minIssueWeight`, `minIssueVolume`).
  - Automatic nullification of non-applicable attributes for EXTERNAL setups.
  - Rejection of EXTERNAL setup when `externalIssuable = false`.
  - Enforcement of PostgreSQL partial unique index (`wms_outbound_tu_setups_external_uq`): exactly one EXTERNAL setup per warehouse; `ensureWarehouseHasExternalSetup` validation.
* **2. TU_NUMBER & SSCC Identity (R53 / FR-P1-28 / TC-010):**
  - Generates Code 128 compliant alphanumeric `TU_NUMBER` (max 20 chars); permits explicit distinct `SSCC` identifier (`tu.sscc !== tu.tuNumber`).
  - Blocks duplicate active `TU_NUMBER` in same warehouse; allows number reuse after terminal state (`CANCELLED`).
* **3. Item Master Authority (R63 / FR-P1-37 / TC-110, TC-111):**
  - Authoritative resolution from `MasterdataItem` (`masterdata_items` table in PostgreSQL) by `itemId` or `sku`.
  - Calculates `currentWeight = sum(unitWeight * quantity)` and `currentContentVolume = sum(unitVolume * quantity)`.
  - Downstream contract exposes dynamic line sum, never `TUSetup.maxWeight`.
* **4. Issueability Evaluation & Override Rules (R64, R65, R66, R68 / TC-114..TC-120):**
  - **TC-114 (R64 step 2):** INTERNAL `externalIssuable = true` TU passes because CONTENT VOLUME reaches `minIssueVolume` (`0.12 m³ >= 0.10 m³`) while weight (`1.0 kg < 20.0 kg`) does not.
  - **TC-115 (R65):** Below-threshold permitted override persists authenticated operator UUID and reason; issueability transitions to `OPERATOR_OVERRIDE_APPLIED`.
  - **TC-116 (R66):** Repack target sealing guard blocks non-issuable repack target TU (`externalIssuable = false`).
  - **TC-117 (R65):** `externalIssuable = false` is an absolute block that CANNOT be overridden by operator.
  - **TC-120 / R68:** EXTERNAL recognition only via `TUSetup.processUsage = 'EXTERNAL'` -> automatic issueability without lower-threshold evaluation.
* **5. Decisive Real PostgreSQL Concurrency Suite (10 Independent Clients & PIDs):**
  - Spawns 10 independent `Client` connections with distinct `SELECT pg_backend_pid()`.
  - Concurrently executes atomic sequence generation with temporal overlap.
  - Produces 10 collision-free sequential `TU_NUMBER`s (`1..10`); fresh independent connection confirms `last_value = 10`.
* **6. Genuine PostgreSQL Rollback Suite:**
  - Performs writes inside an explicit transaction and executes `ROLLBACK`.
  - Fresh independent connection confirms 0 committed rows.
* **7. Tenant & Warehouse Isolation:**
  - Validates tenant and warehouse boundaries; prevents cross-tenant access and mutation.

---

## 4. Module-Wide Regression Protection

* **Full WMS Outbound Suite:** `15/15` suites passed, `205/205` tests passed.
* **FND-003 Shared Compatibility Suite:** `8/8` passed (`fnd-003-postgres.integration.test.ts`).
* **WMS Receiving Suite:** `10/10` suites passed, `84/84` tests passed.
