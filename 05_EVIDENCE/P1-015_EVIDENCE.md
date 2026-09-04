# P1-015 Evidence — CarrierManifest Lifecycle and Dispatch Boundaries

**Task:** P1-015 — CarrierManifest lifecycle and dispatch boundaries (Process 1 `STANDARD_FULFILLMENT` Steps 12 & 13, P1 R39, R40, R41, R70, `FR-P1-20`, `FR-P1-44`, `FR-P1-46`, `CON-04`, `CON-05`, `TC-001`, `TC-008`, `TC-128`, `TC-129`, `TC-132`, `TC-133`)  
**Execution guide:** `06_AGENT_GUIDES/P1-015_EXECUTION.md`  
**Evidence date:** 2026-09-04  
**Evidence class:** Candidate evidence — REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED (NOT steering / NOT acceptance)  

## Final Revisions and Runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato accepted P1-014 base | `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6` |
| Mercato final P1-015 head | `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8` on `outbound/p1-015` (pushed) |
| Lineage / compare | Exactly 1 commit ahead of accepted P1-014 base; merge base: `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6` |
| Served Testing Mercato runtime revision | `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8` (served via `mercato-localhost.service` at `https://devaxonic-test.info-start.com.pl`) |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` on `outbound/p1-009` (verified untouched, clean working tree) |
| Devaxonic-WMS steering | `3f46432cc75899c80be38ec9206d61b9a544f416` (untouched) |
| WMS_Outbound base | `0d9b1f88319d2c8b202b6c75a9f21c5c74f53e75` on `main` |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:6543`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `https://devaxonic-test.info-start.com.pl` returned HTTP 200 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Exact Git Compare Statistics

Git compare: `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6..71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8`

```text
 apps/mercato/src/modules/wms_outbound/__integration__/P1-015-manifest-dispatch-ui.spec.ts |  732 +++++++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/api/manifests/[id]/add-shipment/route.ts            |   41 ++
 apps/mercato/src/modules/wms_outbound/api/manifests/[id]/close/route.ts                   |   40 ++
 apps/mercato/src/modules/wms_outbound/api/manifests/[id]/confirm/route.ts                 |   40 ++
 apps/mercato/src/modules/wms_outbound/api/manifests/[id]/handover/route.ts                |   40 ++
 apps/mercato/src/modules/wms_outbound/api/manifests/[id]/route.ts                         |   28 +
 apps/mercato/src/modules/wms_outbound/api/manifests/route.ts                              |   72 +++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/add-to-manifest/route.ts         |   45 ++
 apps/mercato/src/modules/wms_outbound/backend/manifests/[id]/page.tsx                     |  441 +++++++++++++++
 apps/mercato/src/modules/wms_outbound/backend/manifests/page.tsx                         |  263 +++++++++
 apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx                    |   45 +-
 apps/mercato/src/modules/wms_outbound/data/entities.ts                                   |   70 +++
 apps/mercato/src/modules/wms_outbound/data/transitions.ts                                |   18 +
 apps/mercato/src/modules/wms_outbound/data/validators.ts                                 |    8 +
 apps/mercato/src/modules/wms_outbound/di.ts                                               |    7 +
 apps/mercato/src/modules/wms_outbound/migrations/Migration20260904220000_wms_outbound_p1_015.ts |   64 +++
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-015-manifest-lifecycle-postgres.integration.test.ts | 1546 +++++++++++++++++++++++++++++++++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/carrier-selection-service.ts               |   40 +-
 apps/mercato/src/modules/wms_outbound/services/manifest-service.ts                       |  640 +++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/state-transition-service.ts                |   12 +
 20 files changed, 4183 insertions(+), 9 deletions(-)
```

## Schema & Migration Facts (Literal Committed DDL)

Migration: `Migration20260904220000_wms_outbound_p1_015.ts` (applied to Testing PostgreSQL)

### Table: `wms_outbound_carrier_manifests`
- **Columns:**
  - `id` (uuid, PRIMARY KEY, DEFAULT `gen_random_uuid()`)
  - `organization_id` (uuid, NOT NULL)
  - `tenant_id` (uuid, NOT NULL)
  - `warehouse_id` (uuid, NOT NULL)
  - `manifest_number` (text, NOT NULL)
  - `status` (text, NOT NULL, DEFAULT `'OPEN'`)
  - `carrier_id` (text, NULL)
  - `carrier_name` (text, NULL)
  - `closed_at` (timestamptz, NULL)
  - `closed_by` (text, NULL)
  - `handed_over_at` (timestamptz, NULL)
  - `handed_over_by` (text, NULL)
  - `confirmed_at` (timestamptz, NULL)
  - `confirmed_by` (text, NULL)
  - `idempotency_key` (text, NULL)
  - `notes` (text, NULL)
  - `created_at` (timestamptz, NOT NULL, DEFAULT `NOW()`)
  - `updated_at` (timestamptz, NOT NULL, DEFAULT `NOW()`)
- **Indexes:**
  - `CREATE UNIQUE INDEX "wms_outbound_carrier_manifests_num_uq" ON "wms_outbound_carrier_manifests" ("organization_id", "tenant_id", "manifest_number")`
  - `CREATE INDEX "wms_outbound_carrier_manifests_org_tenant_idx" ON "wms_outbound_carrier_manifests" ("organization_id", "tenant_id")`
  - `CREATE INDEX "wms_outbound_carrier_manifests_scope_wh_idx" ON "wms_outbound_carrier_manifests" ("organization_id", "tenant_id", "warehouse_id")`
  - `CREATE INDEX "wms_outbound_carrier_manifests_status_idx" ON "wms_outbound_carrier_manifests" ("organization_id", "tenant_id", "status")`

### Table: `wms_outbound_shipments` (Membership Index)
- `CREATE INDEX "wms_outbound_shipments_manifest_idx" ON "wms_outbound_shipments" ("organization_id", "tenant_id", "manifest_id")`
- Shipment-to-manifest membership is singular and authoritative on `wms_outbound_shipments.manifest_id` (`FR-P1-20`).

## Target CarrierManifest Lifecycle & Dispatch Boundaries

1. **State Machine (`CARRIER_MANIFEST_TRANSITIONS`):**
   - `(initial) -> OPEN` via `CarrierManifestOpened` (Actor: Dispatcher, P1 step 12)
   - `OPEN -> CLOSED` via `CarrierManifestClosed` (Actor: Dispatcher, P1 step 12, P1 R40)
   - `CLOSED -> HANDED_OVER` via `CarrierManifestHandedOver` (Actor: Dispatcher/Carrier, P1 step 13)
   - `HANDED_OVER -> CONFIRMED` via `CarrierManifestConfirmed` (Actor: Dispatcher, P1 step 13, P1 R70)
   - `CONFIRMED` is final/terminal; no reverse transitions exist.

2. **One-Manifest Membership Boundary (`FR-P1-20`):**
   - Only Shipments in exact state `POSTED` may be added to an `OPEN` manifest.
   - Assignment transitions Shipment `POSTED -> IN_MANIFEST` and records one `ShipmentAddedToManifest` transition.
   - Replay of same Shipment to same manifest is idempotent (`replayed: true`).
   - Reassigning an already-manifested Shipment to a second manifest fails closed under row lock.

3. **Irreversible Composition Boundary (`P1 R40`):**
   - `OPEN -> CLOSED` freezes composition.
   - Adding or removing shipments after `CLOSED` is strictly rejected server-side.
   - Reopening or cancelling a closed manifest is impossible (no such transition exists).
   - Race conditions between add and close are deterministically serialized via `FOR UPDATE` locking on manifest and shipment.

4. **Deferred P1-013 Carrier-Correction Boundary Completed:**
   - Pre-close carrier correction by Warehouse Supervisor remains available on `POSTED` or `IN_MANIFEST` shipments while manifest is `OPEN`.
   - Once manifest is `CLOSED`, `HANDED_OVER`, or `CONFIRMED`, carrier correction fails closed server-side (`P1 R40`).
   - No automatic label reprint or regeneration is performed.

5. **Physical Handover vs. Confirmation Distinction (`P1 step 13, R41, R70`):**
   - Physical handover (`CLOSED -> HANDED_OVER`):
     - Transitions included Shipments (`IN_MANIFEST -> HANDED_TO_CARRIER`).
     - Transitions included TUs (`IN_SHIPMENT -> DISPATCHED`).
     - Transitions contributing OutboundOrders (`READY_FOR_DISPATCH -> DISPATCHED`).
     - Manifest remains in `HANDED_OVER`.
   - Warehouse confirmation (`HANDED_OVER -> CONFIRMED`):
     - Separate user action.
     - Persists a durable snapshot of manifest composition for downstream processing.
     - Duplicate/parallel confirmation calls serialize idempotently (`replayed: true`), emitting exactly one `CarrierManifestConfirmed` transition event.

6. **Server-Authoritative RBAC Authorization:**
   - Manifest lifecycle actions require Dispatcher authority (`checkDispatcherAuthority` / `wms_outbound.edit` or `wms_outbound.manage_orders`).
   - Non-authorized operators (e.g. `packer_operator_p10@devaxonic.local`) fail closed.

7. **Hard Scope Boundary — P1-016 Settlement Exclusion:**
   - P1-015 does **not** implement downstream settlement.
   - Upon confirmation, Inventory remains `PICKED` (not `SHIPPED`), Allocation `reservedQty` remains decremented only by pick, OutboundOrderLine remains `PACKED` (not `SHIPPED`), Allocation remains `CONFIRMED` (not `CONSUMED`), and OutboundOrder remains `DISPATCHED` (not `COMPLETED`).
   - Final settlement is explicitly deferred to P1-016 (`TC-128`, `TC-129`, `TC-132`, `TC-133`).

## Genuine PostgreSQL Verification (20/20 PASSED)

Executed against remote Testing PostgreSQL (`aws-1-eu-central-1.pooler.supabase.com:6543`, database `postgres`, PostgreSQL 17.6):

```text
Test Suites: 1 passed, 1 total
Tests:       20 passed, 20 total
Snapshots:   0 total
Time:        93.425 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/p1-015-manifest-lifecycle-postgres.integration.test.ts.
```

### Complete Test Inventory (20/20)

1. `1. Authorized Dispatcher creates a target manifest in OPEN with CarrierManifestOpened once`
2. `2. Only POSTED Shipment may be added to an OPEN manifest`
3. `3. Successful add sets one membership, POSTED -> IN_MANIFEST, one ShipmentAddedToManifest`
4. `4. Replay of same Shipment -> same manifest assignment is idempotent/non-regressive`
5. `5. Assignment of one Shipment to a second manifest fails closed`
6. `6. Real concurrent assignment to two manifests is exactly-one with decisive PostgreSQL overlap proof`
7. `7. Cross-tenant/org membership fails closed`
8. `8. OPEN -> CLOSED produces exactly one CarrierManifestClosed and freezes composition`
9. `9. Reopen/remove/add after CLOSED is impossible`
10. `10. Real add-versus-close race cannot attach a Shipment after durable close`
11. `11. Accepted pre-close Supervisor Carrier correction remains usable at the Architect-valid boundary and does not regenerate/reprint label`
12. `12. Carrier correction after CLOSED/later states fails closed`
13. `13. CLOSED -> HANDED_OVER is the only handover start state`
14. `14. Handover transitions included Shipments to HANDED_TO_CARRIER, TUs to DISPATCHED and contributing OutboundOrder READY_FOR_DISPATCH -> DISPATCHED exactly once`
15. `15. Handover is distinct from confirmation: manifest remains HANDED_OVER until explicit confirm`
16. `16. Handover rollback leaves no partial multi-entity state/event facts`
17. `17. HANDED_OVER -> CONFIRMED creates one durable CarrierManifestConfirmed boundary/fact with snapshot payload`
18. `18. Duplicate/parallel confirm is exactly-once with real database proof`
19. `19. Non-authorized operator lifecycle actions fail closed under server-authoritative RBAC`
20. `20. P1-016 settlement is not implemented: confirm does not prematurely consume Allocation, ship OOL quantities, complete OutboundOrder or close CustomerOrder`

### Decisive Actual-Service Concurrency Proofs

- **Test 6 (Real Overlapping Assignment to Two Manifests):**
  - Two independent service instances on separate connections attempt to assign the same `POSTED` shipment to `manifest1` and `manifest2`.
  - Contention query on `pg_stat_activity` / `pg_blocking_pids` confirms backend lock contention:
    - `wait_event_type`: `'Lock'`
    - `blocking_pids`: contains `pidA`
  - Winner succeeds: shipment becomes `IN_MANIFEST` with `manifestId = manifest1.id`.
  - Loser fails deterministically with rejection error: `Shipment ... is already assigned to manifest`.
  - Fresh read confirms singular membership and exactly 1 `ShipmentAddedToManifest` transition event.

- **Test 10 (Add-vs-Close Composition Race Proof):**
  - Manifest close operation holds row lock during close transaction.
  - Concurrent add operation blocks on manifest row lock:
    - Contention query confirms `wait_event_type = 'Lock'` with `blocking_pids` containing `pidClose`.
  - Close operation completes, transitioning manifest to `CLOSED`.
  - Add operation unblocks and fails closed: `Cannot add shipment to manifest in status "CLOSED"`.
  - Fresh read confirms shipment remains `POSTED` with `manifest_id = NULL`.

- **Test 18 (Duplicate/Parallel Confirmation Proof):**
  - Two concurrent `confirmManifest()` operations execute against a `HANDED_OVER` manifest.
  - Operation A acquires lock, transitions manifest to `CONFIRMED`, and creates snapshot.
  - Operation B blocks on manifest lock; upon acquiring lock, detects `status === 'CONFIRMED'` and idempotently returns `{ success: true, replayed: true }`.
  - Lock contention query confirms:
    - `wait_event_type`: `'Lock'`
    - `blocking_pids`: contains `pidA`
  - Fresh read verifies exactly 1 `CarrierManifestConfirmed` transition event exists.

- **Test 16 (Multi-Entity Handover Rollback Proof):**
  - Handover transaction simulates deterministic failure after writing and flushing updates across manifest, shipments, TUs, orders, and transition events.
  - Independent database inspection proves 100% rollback:
    - Manifest remains `CLOSED` (not `HANDED_OVER`).
    - Shipments remain `IN_MANIFEST` (not `HANDED_TO_CARRIER`).
    - TUs remain `IN_SHIPMENT` (not `DISPATCHED`).
    - OutboundOrders remain `READY_FOR_DISPATCH` (not `DISPATCHED`).
    - 0 handover-related transition events persisted.

## Regression Test Results

| Suite | Result | Details |
|---|---|---|
| **P1-015 PostgreSQL Suite** | **20/20 PASSED** | Manifest lifecycle, irreversible composition, handover, exactly-once confirmation |
| **P1-014 PostgreSQL Suite** | **18/18 PASSED** | ERP shipment posting, rejection categorisation, safe retry, in-flight idempotency |
| **P1-013 PostgreSQL Suite** | **15/15 PASSED** | Label generation, carrier selection correction, lock contention proof |
| **P1-012 PostgreSQL Suite** | **14/14 PASSED** | Carrier auto-assignment, manual override, audit history |
| **P1-011 PostgreSQL Suite** | **18/18 PASSED** | Shipment grouping, aggregation, concurrency proof |
| **FND-002 State Machine Suite** | **77/77 PASSED** | Core outbound state machine transition rules |
| **FND-002 Transaction Suite** | **8/8 PASSED** | Real PostgreSQL transaction simulation invariants |
| **Total Automated Regression** | **170/170 PASSED** | Zero regressions across all integrated outbound modules |

## Playwright Rendered UI Verification (6/6 PASSED)

Executed against `https://devaxonic-test.info-start.com.pl` (served via `mercato-localhost.service` on commit `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8`):

```text
Running 6 tests using 1 worker

  ✓  1 Journey A (PLAYWRIGHT VERIFIED): Authorized Dispatcher opens new manifest and adds POSTED shipment (P1 R39, FR-P1-20, TC-001) (23.6s)
  ✓  2 Journey B (PLAYWRIGHT VERIFIED): Uniqueness and invalid membership boundary fails closed (P1 R39, FR-P1-20) (13.3s)
  ✓  3 Journey C (PLAYWRIGHT VERIFIED): Close freezes composition & carrier correction is blocked post-close without label reprint (P1 R35, P1 R40, FR-P1-20) (20.3s)
  ✓  4 Journey D (PLAYWRIGHT VERIFIED): Physical handover transitions manifest to HANDED_OVER, shipments to HANDED_TO_CARRIER, TUs to DISPATCHED (P1 step 13, R41) (13.8s)
  ✓  5 Journey E (PLAYWRIGHT VERIFIED): Final warehouse confirmation establishes exactly-once boundary without P1-016 downstream settlement (P1 step 13, R70, P1-016 boundary) (15.4s)
  ✓  6 Journey F (PLAYWRIGHT VERIFIED): Non-authorized operator lifecycle actions fail closed under server-authoritative RBAC (13.5s)

  6 passed (1.8m)
```

### Rendered Journey Details (6/6)

- **Journey A (PLAYWRIGHT VERIFIED):** Authorized Dispatcher opens new manifest and adds `POSTED` shipment (`P1 R39`, `FR-P1-20`, `TC-001`)
  - Logged in as authorized Dispatcher (`admin_dev@devaxonic.local`).
  - Created new CarrierManifest via UI `/backend/manifests` modal (`carrier: DPD`, `warehouse: Main Warehouse`).
  - Added real `POSTED` shipment via manifest detail action.
  - Manifest displays status `OPEN`, assigned shipments count 1.
  - Shipment detail page reflects status `IN_MANIFEST` with manifest number linked.
  - Direct database check verifies 1 `CarrierManifestOpened` and 1 `ShipmentAddedToManifest` transition event.

- **Journey B (PLAYWRIGHT VERIFIED):** Uniqueness and invalid membership boundary fails closed (`P1 R39`, `FR-P1-20`)
  - Attempt to add non-`POSTED` shipment (`LABEL_GENERATED`) via UI is rejected with error banner: *"Only POSTED shipments may be added to a manifest"*.
  - Attempt to assign an already-manifested shipment to a second manifest is rejected: *"already assigned to manifest"*.
  - Database verifies membership remains singular.

- **Journey C (PLAYWRIGHT VERIFIED):** Close freezes composition & carrier correction is blocked post-close without label reprint (`P1 R35`, `P1 R40`, `FR-P1-20`)
  - Dispatcher clicks "Close Manifest" on `/backend/manifests/[id]`.
  - Manifest status updates to `CLOSED`; "Add Shipment" controls are removed.
  - Direct attempt to add shipment to closed manifest returns structured error: *"Cannot add shipment to manifest in status CLOSED"*.
  - Supervisor carrier correction on a closed shipment fails closed with error banner: *"Cannot correct carrier for shipment in manifest status CLOSED"*.
  - No new shipment label rows created; reprint is not triggered.

- **Journey D (PLAYWRIGHT VERIFIED):** Physical handover transitions manifest to `HANDED_OVER`, shipments to `HANDED_TO_CARRIER`, TUs to `DISPATCHED` (`P1 step 13`, `R41`)
  - Dispatcher clicks "Handover to Carrier".
  - UI displays manifest status `HANDED_OVER`.
  - Included shipment detail displays `HANDED_TO_CARRIER`.
  - Database verification confirms:
    - Included TU status = `DISPATCHED` (`WmsOutboundTransportUnit`).
    - Contributing OutboundOrder status = `DISPATCHED` (`WmsOutboundOrder`).
    - Manifest is NOT `CONFIRMED` (distinct lifecycle boundary).

- **Journey E (PLAYWRIGHT VERIFIED):** Final warehouse confirmation establishes exactly-once boundary without P1-016 downstream settlement (`P1 step 13`, `R70`, `P1-016 boundary`)
  - Dispatcher clicks "Confirm Dispatch (Final)".
  - Manifest status updates to `CONFIRMED`.
  - Retry/re-click action is idempotent; UI remains stable in `CONFIRMED`.
  - Database inspection verifies exactly 1 `CarrierManifestConfirmed` transition event with frozen composition snapshot.
  - Downstream settlement audit proves:
    - Inventory ledger remains `PICKED` (not `SHIPPED`).
    - Allocation `reservedQty` is NOT decremented.
    - OutboundOrderLine remains `PACKED` (not `SHIPPED`).
    - OutboundOrder remains `DISPATCHED` (not `COMPLETED`).
    - CustomerOrder remains `IN_FULFILLMENT` (not `CLOSED`).
    - P1-016 settlement boundary is strictly preserved.

- **Journey F (PLAYWRIGHT VERIFIED):** Non-authorized operator lifecycle actions fail closed under server-authoritative RBAC
  - Logged in as warehouse operator (`packer_operator_p10@devaxonic.local`, lacking `wms_outbound.manage_orders` / dispatch capability).
  - Attempted close, handover, and confirm actions via UI are rejected with structured server error: *"is not authorized to perform this manifest action (Dispatcher authority required)"*.
  - Manifest remains in its prior state; no unauthorized state progression.

## Worktree Status

```text
Devaxonic-mercato: clean (outbound/p1-015 @ 71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8)
Devaxonic-scanner: clean (outbound/p1-009 @ f4a404600efb1120cb2f1c5b86383ad148cd1e1a)
Devaxonic-WMS:     clean (main @ 3f46432cc75899c80be38ec9206d61b9a544f416)
WMS_Outbound:      clean (main @ 0d9b1f88319d2c8b202b6c75a9f21c5c74f53e75)
```

## Explicit Scope Exclusions Obeyed

- **NO P1-016 Downstream Settlement:** Final consumption of Allocations, shipping of OutboundOrderLines/Inventory, and completion of OutboundOrders/CustomerOrders is intentionally excluded from P1-015 and deferred to P1-016.
- **NO External Carrier API:** No external carrier dispatch webservices, tracking webhooks, or trailer/dock scheduling introduced.
- **NO Crossdock / P2-005:** Preserved for P2.
- **NO Return Receipt / Post-Dispatch Corrections:** Preserved for future phases.
- **NO Scanner Changes:** Scanner repository frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- **NO Prod/Demo Changes:** All execution strictly on Testing environment.
- **NO Acceptance Claims:** Evidence is candidate evidence only (`PLAYWRIGHT VERIFIED` and `REAL PostgreSQL INTEGRATION VERIFIED`). Steering files (`STATE.md`, handovers) remain untouched for Owner acceptance.
