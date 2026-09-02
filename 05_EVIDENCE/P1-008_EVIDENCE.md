# P1-008 Execution & Acceptance Evidence

**Catalog Item:** `P1-008` — Outbound TU identity, TUSetup, numbering, capacity and issueability  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (R53, R63, R64, R65, R66, R68, FR-P1-28, FR-P1-37, FR-P1-38, FR-P1-39, FR-P1-40, FR-P1-42)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Repository Commit SHAs

* **Mercato Accepted Base (P1-005):** `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975`
* **Mercato P1-008 Previous Head:** `2a8fbb26774c562e6f97398e0f27e3ff1e830607`
* **Mercato P1-008 Final DB-Remediation Head (`outbound/p1-008`):** `cb690079ce1e55cf89b4511f0762f0ecbf3dcd86`
* **Scanner P1-008 Head (`main`):** `b5cfb59987c76f39e0ab48af67a52e2e914d9613`
* **Backend Build/Runtime Identity:** Live Next.js server on `http://127.0.0.1:3009` running from Mercato commit `cb690079ce1e55cf89b4511f0762f0ecbf3dcd86`.

---

## 2. Real Playwright Acceptance Run (Preserved & Zero Route Mocks)

> **Note:** The decisive user-facing acceptance was executed through real rendered Scanner UI interactions and verified against Testing PostgreSQL. It was not replaced by DB-only evidence.

### Real Rendered UI Journey:
1. **Interactive Operator Login:** Operator logs in through the rendered form (`operator-p1008-ui-...@devaxonic.com`).
2. **Mode Navigation:** Selects warehouse and clicks into `outboundTu` mode screen.
3. **TU Creation via Rendered UI:** Selects `TUSetup` chip and clicks **"Create Outbound TU"** button (`page.getByText('Create Outbound TU')`).
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

## 3. Genuine PostgreSQL Integration Test Battery (Application Paths)

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
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        17.065 s
```

### Detailed Acceptance & Remediation Matrix:

#### 1. App-Path Real Concurrency & Lock Barrier
* **Application Method Exercised:** `createOutboundTuService(txEm).createOutboundTu(scope, { warehouseId, tuSetupCode, role: 'PickContainer' })`
* **Real Connection & Transaction Isolation:**
  - Actor A starts `emA.transactional(...)` (DB PID `1727112`), invokes application `createOutboundTu`, and holds transaction open.
  - Actor B starts `emB.transactional(...)` on separate connection (DB PID `1727108`), concurrently invoking application `createOutboundTu` for the same warehouse.
  - Observer connection queries PostgreSQL `pg_stat_activity` / `pg_locks` and captures lock evidence: Actor B is blocked at PostgreSQL waiting on sequence row lock held by Actor A (`lockEvidenceCaptured: true`).
  - Actor A transaction commits and releases lock; Actor B immediately unblocks and commits.
  - Generates distinct, collision-free sequential numbers: `tuANumber: 'TU0000000001'`, `tuBNumber: 'TU0000000002'`.
* **Application Concurrency Burst:**
  - 10 concurrent actors invoke application `createOutboundTu` via separate EntityManager forks in parallel.
  - Produces 10 unique sequential numbers (`1..10`).
  - Fresh independent PostgreSQL connection confirms `wms_outbound_tu_sequences.last_value = 10` and 10 rows in `wms_outbound_transport_units`.

#### 2. App-Path Real Rollback & Clean Abort
* **Application Method Exercised:** `em.transactional(async (txEm) => { ... })` invoking `service.createTuSetup`, `service.createOutboundTu`, and `service.addTuContent`.
* **Failure & Rollback Behavior:** After flushing application entities through MikroORM, an error is deliberately thrown before commit.
* **Fresh Independent Read:** Fresh PostgreSQL connection queries `wms_outbound_tu_setups`, `wms_outbound_transport_units`, and `wms_outbound_tu_contents` and confirms **0 rows committed**.

#### 3. Real Warehouse Isolation & Tenant Scoping
* **Setup Scoping:** Warehouse A setups are completely isolated from Warehouse B (`BOX-WH-A` is not visible or accessible in Warehouse B).
* **Active TU_NUMBER Uniqueness is Warehouse-Scoped (Architect R53):**
  - In Warehouse A, creating an active TU with `tuNumber = 'TUSHARED0001'` succeeds.
  - Duplicate `TUSHARED0001` in Warehouse A is blocked (`already in use by an active Outbound TU`).
  - In Warehouse B, creating an active TU with the **same** `tuNumber = 'TUSHARED0001'` **succeeds** independently without collision.
* **Mutation Guard:** Attempting to create a TU in Warehouse B using a `tuSetupCode` belonging to Warehouse A fails closed (`not found in warehouse`).

#### 4. Item Master Authority & Metric Calculation (R63 / FR-P1-37 / TC-110, TC-111)
* Authoritatively resolves `dimensions.weight` and `dimensions.volume` from `masterdata_items` table in PostgreSQL.
* Calculates `currentWeight` and `currentContentVolume` from line contents; exposes line sum (never `TUSetup.maxWeight`).

#### 5. Issueability Evaluation Order & Override Gates (R64, R65, R66, R68 / TC-114..TC-120)
* **TC-114 (R64 step 2):** INTERNAL `externalIssuable = true` TU passes when content volume reaches `minIssueVolume` (`0.12 m³ >= 0.10 m³`) while weight (`1.0 kg < 20.0 kg`) does not.
* **TC-115 (R65):** Below-threshold override persists authenticated operator UUID and reason, transitioning to `OPERATOR_OVERRIDE_APPLIED`.
* **TC-116 (R66):** Repack target sealing guard blocks non-issuable repack target TU (`externalIssuable = false`).
* **TC-117 (R65):** `externalIssuable = false` is an absolute block that cannot be overridden.
* **TC-120 / R68:** EXTERNAL recognition only via `TUSetup.processUsage = 'EXTERNAL'` -> automatic issueability without reading lower thresholds.
* **R68 DB Constraint:** Exactly one EXTERNAL setup per warehouse enforced by PostgreSQL partial unique index `wms_outbound_tu_setups_external_uq`.

---

## 4. Module-Wide Regression Protection

* **Full WMS Outbound Suite:** `15/15` suites passed, `208/208` tests passed.
* **FND-003 Shared Compatibility Suite:** `8/8` passed (`fnd-003-postgres.integration.test.ts`).
* **WMS Receiving Suite:** `10/10` suites passed, `84/84` tests passed.

---

## 5. Remaining Gaps & Stop Boundary

* **Remaining Gaps:** None for P1-008.
* **Stop Boundary:** P1-008 execution is complete and verified. No work on P1-006 has been started. Execution has halted.
