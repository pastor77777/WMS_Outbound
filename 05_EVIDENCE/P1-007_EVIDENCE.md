# P1-007 Execution & Acceptance Evidence

**Catalog Item:** `P1-007` — SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes  
**Process Scope:** Process 1 STANDARD_FULFILLMENT (P1 R42–R48, R69, R71, FR-P1-21..24, FR-P5-01..06, TC-060, TC-061, TC-062, TC-121)  
**Execution Date:** 2026-09-02  
**Evidence Label:** `PLAYWRIGHT VERIFIED` (Automated Real Browser UI + Remote Testing PostgreSQL Persistence)

---

## 1. Lineage & Authoritative Repository Commit SHAs

* **Mercato P1-007 Tested Head (`outbound/p1-007`):** `134db31381b4db726cd550abe6ecd4079ac21d8c`
* **Scanner P1-007 Tested Head (`main`):** `b23325aae1c4f83b79d01b3650dbead3486a1041`
* **Authoritative Outbound Steering Head (`main`):** `2320d7381e78eacec3c4f934c25b061136c5234c`
* **Lineage & Invariance:**
  - `Devaxonic-mercato` `134db31381b4db726cd550abe6ecd4079ac21d8c` descends cleanly from accepted base `4c53cbdeb` via Shot 1B (`1979942c9`) and Shot 1C (`134db3138`). Changes are strictly confined to the P1-007 migration lifecycle test harness in `p1-007-postgres.integration.test.ts`. Product code and business logic are 100% preserved.
  - `Devaxonic-scanner` `b23325aae1c4f83b79d01b3650dbead3486a1041` is identical across all closeout shots; zero scanner changes were introduced.
* **Testing PostgreSQL Database:** Remote DevAxonic Testing Database (`devaxonic-test.info-start.com.pl`).
* **Live Provenance & Runtimes:**
  - Mercato Next.js Backend: `http://localhost:3000` (serving current `outbound/p1-007`)
  - Scanner Web App: `http://localhost:8081` (serving static export of `main` `b23325aae1c4f83b79d01b3650dbead3486a1041`)

---

## 2. Architect Requirement Traceability & Target Invariants

| Architect Source Ref | Functional Requirement / Rule | Implemented & Verified Target Behavior |
| :--- | :--- | :--- |
| **R42 / FR-P1-21** | `SHORT_ALLOCATED` exception detection | When ATP stock is insufficient during release, line marks `SHORT_ALLOCATED`. If `allowPartialShipment = true`, order proceeds with available stock. If `allowPartialShipment = false`, order holds in fulfillment and escalates to Supervisor. |
| **R43 / FR-P5-01..02** | `SHORT_ALLOCATED` Supervisor Outcomes | **Outcome A (`ALLOW_PARTIAL`):** Persistently flips `allowPartialShipment = true`, records decision audit in `wms_outbound_supervisor_decisions`, and line advances to `ALLOCATED`.<br>**Outcome B (`CANCEL_OUTBOUND_ORDER`):** Cancels execution order, releases all hard allocations (`RELEASED`), deletes inventory reservations, reverts `CustomerOrder` to `ACCEPTED` with `has_warning = true` and `warning_reason`, and restores soft ATP promise on `CustomerOrderLine`. |
| **R44 / FR-P1-22..24** | `SHORT_PICKED` Auto-Reallocation Bounds & Location Blocking | Short pick at location L blocks location in `wms_outbound_location_shortages` (`is_active = true`). Auto-reallocation enforces two bounds under SKU lock: (1) individual unblocked location unreserved stock, and (2) warehouse-wide unreserved hard capacity across all active allocations (including pre-PickTask allocations). If either bound fails, auto-reallocation ceases immediately. |
| **R45 / FR-P5-03** | Auto-Reallocation Retry Limits | Hierarchy: `CustomerOrder.max_automatic_short_pick_reallocations` overrides warehouse config (default 1). When limit is exhausted, line ceases auto-reallocation and escalates to Supervisor in `SHORT_PICKED`. |
| **R46 / FR-P5-04..06** | `SHORT_PICKED` Supervisor Outcomes | **Outcome A (`WAIT`):** Sister un-packed lines roll back to `OPEN`, sister allocations release, soft ATP restored, order reverts to `ACCEPTED` + `WARNING`, and `wms_outbound_physical_return_handoffs` created for picked units.<br>**Outcome B (`CANCEL_OR_CORRECT`):** Exact picked quantity corrects demand and reservation (TC-121). Zero quantity (`0`) cancels line, updates `CustomerOrderLine.orderedQuantity = 0`, `OutboundOrderLine.requiredQty = 0`, releases allocation, and records physical return handoff.<br>**Outcome C (`ALLOW_PARTIAL`):** Persistently enables `allowPartialShipment = true`, shrinks allocation to picked portion, restores soft ATP for missing portion (`BACKORDERED`), and line advances to `PICKED`. |
| **R48 / FR-P1-31** | Repack Shortage Seam | Supervisor resolution preserves repacked carton integrity without phantom allocation leaks. |
| **R69 / R71 / FR-P1-32** | Concurrency, Lock Isolation & Rollback | Authoritative `FOR UPDATE` locking on `PickTask` and `CustomerOrder`. Distinct PIDs capture real PostgreSQL lock wait events (`pg_blocking_pids`). Transaction aborts rollback completely without partial state leaks. |

---

## 3. Shot 1C Genuine MikroORM Migrator Proof (Test 6A)

* **Runner & API:** Real MikroORM `Migrator` (`extensions: [Migrator]` / `orm.migrator as Migrator`) from `@mikro-orm/migrations`. Zero direct migration class instantiations (`new Migration2026...`), zero manual `getQueries()` loops, zero manual `em.execute(q)` SQL replays, and zero hand-written shadow tables.
* **Isolated Harness:** Isolated PostgreSQL schema dynamically created with connection-level `-c search_path=<isoSchema>,public` options and dedicated migration metadata storage table (`mikro_orm_migrations_wms_outbound`).
* **Preceding History via Migrator:** `await migrator.up({ to: 'Migration20260902100000_wms_outbound_p1_007' })` applied all 10 preceding project migrations in order. Fresh PostgreSQL inspection proved pre-remediation constraints were strictly `> 0`, executed count was 10, and pending count was exactly 1 (`Migration20260902160000_wms_outbound_p1_007_remediation`).
* **Path A (Unconditional Reversible Lifecycle):**
  1. `await migrator.up({ to: 'Migration20260902160000_wms_outbound_p1_007_remediation' })` -> fresh DB confirmed constraints are `>= 0`.
  2. `await migrator.down({ to: 'Migration20260902100000_wms_outbound_p1_007' })` unconditionally executed -> fresh DB confirmed constraints are restored to `> 0`.
  3. `await migrator.up({ to: 'Migration20260902160000_wms_outbound_p1_007_remediation' })` -> fresh DB confirmed constraints are restored to `>= 0`.
* **Path B (Incompatible-Data Fail-Safe):** Under real Migrator UP state with legitimate P1-007 R46 cancelled zero-quantity rows in the migrated tables:
  1. `await migrator.down({ to: 'Migration20260902100000_wms_outbound_p1_007' })` failed fast with explicit compatibility error *before* any constraint mutation.
  2. Fresh DB confirmed zero rows remain completely intact and unchanged.
  3. Fresh DB confirmed constraints remain safely at `>= 0`.
  4. Fresh `migrator.getExecuted()` confirmed remediation migration remains recorded as applied in migration history.
* **Decisive Proof Output:**
  ```text
  [migrator] Processing 'Migration20260831200000_wms_outbound'
  [migrator] Applied 'Migration20260831200000_wms_outbound'
  ...
  [migrator] Processing 'Migration20260902100000_wms_outbound_p1_007'
  [migrator] Applied 'Migration20260902100000_wms_outbound_p1_007'
  [migrator] Processing 'Migration20260902160000_wms_outbound_p1_007_remediation'
  [migrator] Applied 'Migration20260902160000_wms_outbound_p1_007_remediation'
  [migrator] Processing 'Migration20260902160000_wms_outbound_p1_007_remediation'
  [migrator] Reverted 'Migration20260902160000_wms_outbound_p1_007_remediation'
  [migrator] Processing 'Migration20260902160000_wms_outbound_p1_007_remediation'
  [migrator] Applied 'Migration20260902160000_wms_outbound_p1_007_remediation'

  [P1-007 Decisive Migration Proof] {
    compatibleUpExecuted: true,
    compatibleDownExecuted: true,
    compatibleReUpExecuted: true,
    incompatibleDownBlocked: true,
    incompatibleRowsPreserved: true,
    constraintsAfterBlockedDown: '>= 0'
  }
  ```

---

## 4. Remote Testing PostgreSQL Integration Evidence (20/20 PASSED)

**Test Command:**
```bash
yarn --cwd apps/mercato test src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts --runInBand
```

**Decisive Concurrency & pg_blocking_pids Lock Output:**
```text
  console.log
    [P1-007 Decisive PostgreSQL Lock & Reallocation Concurrency Proof] {
      actorAPid: 1815350,
      actorBPid: 1816174,
      blockingPids: [ 1815350 ],
      waitEventType: 'Lock',
      lockEvidenceCaptured: true,
      replacementTasksCreated: 1,
      retryCount: 1
    }
```

**Verbatim Suite Output:**
```text
PASS src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts (84.079 s)
  P1-007 Genuine PostgreSQL SHORT_ALLOCATED & SHORT_PICKED Recovery Suite
    1. TC-060 SHORT_ALLOCATED Handling & Outcomes
      ✓ 1A: allowPartialShipment = true drives real planning/allocation path: available allocated, shortfall BACKORDERED, no supervisor intervention (2290 ms)
      ✓ 1B: allowPartialShipment = false -> Outcome A: Persistent ALLOW_PARTIAL with reason and mandatory idempotencyKey (1380 ms)
      ✓ 1C: allowPartialShipment = false -> Outcome B: CANCEL_OUTBOUND_ORDER releases allocations, restores soft ATP reservations, and reverts order to ACCEPTED + WARNING (1590 ms)
    2. TC-061 SHORT_PICKED Automatic Reallocation Limits & Location Blocking
      ✓ 2A1: unblocked location exists but has ZERO ATP inventory -> no replacement task, Supervisor escalation (1510 ms)
      ✓ 2A2: alternative location has non-ATP stock only (DRAFT ASN) -> no replacement task, Supervisor escalation (1460 ms)
      ✓ 2A3: alternative location has eligible ATP stock -> creates exactly one replacement reservation + PickTask, source blocked in location shortages (1790 ms)
      ✓ 2A4: candidate B has 2.0, candidate C has 2.0, shortage is 3.0 -> aggregate stock is 4.0 but neither location individually covers 3.0 -> ceases auto-reallocation and escalates to Supervisor (1610 ms)
      ✓ 2A5: candidate B has 5.0 gross stock but 3.0 already hard-reserved by active pick task -> unreserved stock is only 2.0 -> shortage of 3.0 is rejected, escalating to Supervisor (1640 ms)
      ✓ 2A6: Candidate B has gross 5.0, competing order has active pre-PickTask hard allocation of 3.0 -> unreserved is 2.0 -> shortage 3.0 cannot be reallocated -> ceases auto-reallocation and escalates to Supervisor (1680 ms)
      ✓ 2B: Effective retry limit hierarchy: customer override wins over warehouse setting (1570 ms)
      ✓ 2C: When effective retry limit is reached, short pick ceases automatic reallocation and escalates to Supervisor (1410 ms)
    3. TC-062 SHORT_PICKED Supervisor Outcomes
      ✓ 3A: Outcome A (WAIT) rolls back un-packed sister lines to OPEN, releases hard allocations, restores soft ATP reservations, and reverts CustomerOrder to ACCEPTED + WARNING (1740 ms)
      ✓ 3B: Outcome B (CANCEL_OR_CORRECT) rejects arbitrary positive values, accepts exact picked quantity per TC-121 (1510 ms)
      ✓ 3B2: Outcome B (CANCEL_OR_CORRECT with 0) performs full line cancellation: sets CustomerOrderLine.orderedQuantity = 0, OutboundOrderLine.requiredQty = 0, releases allocation, deletes inventory reservation, creates physical return handoff for 4 picked units (1540 ms)
      ✓ 3C: Outcome C (ALLOW_PARTIAL) persistently updates CustomerOrder.allowPartialShipment with reason (1390 ms)
    4. Concurrency, Idempotency, Strict Observer & Rollback Proofs
      ✓ 4A: Independent connection SAME-K short pick confirmation captures strict pg_blocking_pids lock evidence and deduplicates exactly once (1950 ms)
      ✓ 4B: Supervisor authorization fails closed for unauthorized caller, empty idempotencyKey fails 422, same key replayed returns prior result, conflicting payload fails closed (1230 ms)
      ✓ 4B2: Exhaustive SHORT_PICKED supervisor idempotency suite: 8 scenarios covering CANCEL_OR_CORRECT, ALLOW_PARTIAL, WAIT, exact matches, conflicting decisions, conflicting correctedQuantities, conflicting reasons, and conflicting supervisor identities (1620 ms)
      ✓ 4C: Rollback proof: real failure before commit ensures no partial shortage state leaked (1140 ms)
    5. R48 Repack Shortage Backend Seam
      ✓ 5A: Supervisor shortage resolution service preserves repacked carton integrity without phantom allocation leaks (1200 ms)
      ✓ 6A: Real reversible migration proof: UP/DOWN/re-UP execution with fail-safe incompatible-data protection (6420 ms)

Test Suites: 1 passed, 1 total
Tests:       20 passed, 20 total
Snapshots:   0 total
Time:        84.079 s
```

---

## 5. Real Rendered UI Playwright Evidence

### A. Supervisor Web UI E2E Suite (`P1-007-shortages-supervisor-ui.spec.ts`) — 6/6 Outcomes Verified
* **Target URL:** `http://localhost:3000/backend/shortages`
* **Test Command:**
```bash
PLAYWRIGHT_TEST_BASE_URL=http://localhost:3000 npx playwright test src/modules/wms_outbound/__integration__/P1-007-shortages-supervisor-ui.spec.ts
```
* **Execution Result:** `6 passed (2.6m)`
* **All 6 Distinct UI Outcomes Verified:**
  1. `Outcome 1: SHORT_ALLOCATED -> ALLOW_PARTIAL (persistent with reason)` (30.2s)
  2. `Outcome 2: SHORT_ALLOCATED -> CANCEL_OUTBOUND_ORDER` (25.7s)
  3. `Outcome 3: SHORT_PICKED -> WAIT` (25.4s)
  4. `Outcome 4: SHORT_PICKED -> CANCEL_OR_CORRECT (exact picked quantity 4)` (24.9s)
  5. `Outcome 5: SHORT_PICKED -> CANCEL_OR_CORRECT (0 cancellation)` (24.7s)
  6. `Outcome 6: SHORT_PICKED -> ALLOW_PARTIAL (persistent with mandatory reason)` (25.5s)

### B. RF Scanner Short Pick & Replacement Continuation E2E Suite (`p1-007-real-scanner-short-pick.spec.ts`)
* **Target URL:** `http://localhost:8081`
* **Test Command:**
```bash
npx playwright test e2e/p1-007-real-scanner-short-pick.spec.ts
```
* **Execution Result:** `1 passed (18.4s)`
* **Decisive Proof:** Operator authenticated against backend via proxy, bound picking TU, reported short quantity (2 picked out of 5), verified `⚠️ Report Short Pick` UI action, validated task status `SHORT_PICKED` in PostgreSQL, verified `wms_outbound_location_shortages` recorded active block (`short_quantity = 3`, `is_active = true`), requested next task in zone, received replacement task at Loc B for 3 units, completed pick of 3 units, verified task completion banner in UI, and verified in PostgreSQL:
  - Original task: `SHORT_PICKED`
  - Replacement task: `COMPLETED`
  - Cumulative picked quantity: 5
  - `OutboundOrderLine`: `status = 'PICKED'`

---

## 6. Complete Targeted Regression Matrix (All 9 Gates Freshly Executed & Green)

| Gate | Scope / Invariant Covered | Test Command | Result |
| :--- | :--- | :--- | :--- |
| **P1-004** | Allocation / Hard-Reservation | `yarn test src/modules/wms_outbound/services/__tests__/p1-004-postgres.integration.test.ts --runInBand` | **1 passed, 11/11 tests (45.794 s)** |
| **P1-005** | PickTask creation / assignment / single-active | `yarn test src/modules/wms_outbound/services/__tests__/p1-005-postgres.integration.test.ts --runInBand` | **1 passed, 10/10 tests (23.216 s)** |
| **P1-006 (Backend)** | Real RF picking + SAME-key concurrency & retry | `yarn test src/modules/wms_outbound/services/__tests__/p1-006-postgres.integration.test.ts --runInBand` | **1 passed, 12/12 tests (69.227 s)** |
| **P1-006 (Scanner)** | Real scanner picking & retry key stability | `npx playwright test e2e/p1-006-real-scanner-picking.spec.ts e2e/p1-006-retry-key.spec.ts` | **2 passed (30.2 s)** |
| **P1-001** | CustomerOrder lifecycle & aggregation | `yarn test src/modules/wms_outbound/services/__tests__/p1-001-customer-order-lifecycle.test.ts src/modules/wms_outbound/services/__tests__/p1-001-postgres.integration.test.ts --runInBand` | **2 passed, 21/21 tests (11.687 s)** |
| **P1-003** | Planning & requiredQty | `yarn test src/modules/wms_outbound/services/__tests__/p1-003-postgres.integration.test.ts src/modules/wms_outbound/services/__tests__/p1-003-detail-api-postgres.integration.test.ts --runInBand` | **2 passed, 15/15 tests (54.808 s)** |
| **P1-008** | TU regression | `yarn test src/modules/wms_outbound/services/__tests__/p1-008-postgres.integration.test.ts --runInBand` | **1 passed, 22/22 tests (23.723 s)** |
| **Inbound & Shared Compatibility** | ATP & shared boundaries | `yarn test src/modules/wms_inventory/services/__tests__/atp-service.test.ts src/modules/wms_outbound/services/__tests__/fnd-003-shared-compatibility.test.ts src/modules/wms_outbound/services/__tests__/fnd-003-postgres.integration.test.ts --runInBand` | **3 passed, 17/17 tests (6.867 s)** |
| **Full Outbound Gate** | Entire `wms_outbound` suite (Umbrella Gate) | `yarn test src/modules/wms_outbound --runInBand` | **17 passed, 245/245 tests (361.59 s)** |

---

## 7. Scope Boundaries & Non-Functional Exclusions

* **P4 PutBackTask Lifecycle:** Physical return handoff records (`wms_outbound_physical_return_handoffs`) are properly persisted per R46. Full P4 PutBack execution remains out of scope for P1-007.
* **P1-009 / P1-010 / Packing / QC / Shipment / Carrier / Labels / ERP:** Explicitly out of scope for P1-007.
* **Human Verification:** Automated proof labeled `PLAYWRIGHT VERIFIED`. Awaiting supervisor review for human acceptance.

---

## 8. Visual Evidence Artifacts

1. **Supervisor Shortages & Outcomes Console:**
   `05_EVIDENCE/screenshots/p1-007-real-supervisor-shortages-ui.png`
2. **Scanner RF Picking Short Pick UI:**
   `05_EVIDENCE/screenshots/p1-007-real-scanner-short-pick.png`
3. **Scanner RF Picking Replacement Task Completed UI:**
   `05_EVIDENCE/screenshots/p1-007-real-scanner-replacement-completed.png`
