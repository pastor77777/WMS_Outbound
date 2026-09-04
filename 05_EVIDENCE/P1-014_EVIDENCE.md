# P1-014 Evidence — ERP Shipment POST, Error State and Safe Retry (Remediated)

**Task:** P1-014 — ERP Shipment POST, error state and safe retry (Process 1 `STANDARD_FULFILLMENT` Step 11A, P1 R35–R38, `FR-P1-18`, `FR-P1-19`, `INT-04`, `INT-05`, `CON-05`, `TC-001`, `TC-007`, `TC-008`)  
**Remediation guide:** `06_AGENT_GUIDES/P1-014_REMEDIATION.md`  
**Execution guide:** `06_AGENT_GUIDES/P1-014_EXECUTION.md`  
**Evidence date:** 2026-09-04  
**Evidence class:** Candidate evidence — REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED (NOT steering / NOT acceptance)  

## Final Revisions and Runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato accepted P1-013 base | `5e6b70aa81afd28fe3217e4aad216e8a6482a769` |
| Mercato initial P1-014 head | `d20d95a097f9fe4b4459d6829fbd459c80a5efd3` |
| Mercato final remediated P1-014 head | `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6` on `outbound/p1-014` (pushed) |
| Lineage / compare | Exactly 2 commits ahead of accepted P1-013 base; merge base: `5e6b70aa81afd28fe3217e4aad216e8a6482a769` |
| Served Testing Mercato runtime revision | `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6` (served via `mercato-localhost.service` at `https://devaxonic-test.info-start.com.pl`) |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` (verified untouched, clean working tree) |
| Devaxonic-WMS steering | `a1d9348826d911b92e9ff74cf44ddbf0d6fa51a1` (untouched) |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:6543`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `https://devaxonic-test.info-start.com.pl` returned HTTP 200 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Exact Git Compare Statistics

Git compare: `5e6b70aa81afd28fe3217e4aad216e8a6482a769..bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`

```text
 apps/mercato/src/modules/wms_outbound/__integration__/P1-014-erp-posting-ui.spec.ts        |  560 ++++++++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/cancel-posting-error/route.ts     |   41 ++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/post/route.ts                    |   35 ++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/retry-posting/route.ts            |   41 ++
 apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx                     |  302 ++++++++++++++
 apps/mercato/src/modules/wms_outbound/data/entities.ts                                    |  138 +++++++
 apps/mercato/src/modules/wms_outbound/di.ts                                              |   13 +
 apps/mercato/src/modules/wms_outbound/migrations/Migration20260904200000_wms_outbound_p1_014.ts |   95 +++++
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-014-erp-posting-postgres.integration.test.ts | 1103 +++++++++++++++++++++++++++++++++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/erp-shipment-posting-adapter.ts             |  130 ++++++
 apps/mercato/src/modules/wms_outbound/services/shipment-posting-service.ts                 |  875 ++++++++++++++++++++++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/shipment-service.ts                        |   22 +
 12 files changed, 3355 insertions(+)
```

Remediation diff on top of pre-remediation head `d20d95a09..bef7c0a3e`:
```text
 apps/mercato/src/modules/wms_outbound/__integration__/P1-014-erp-posting-ui.spec.ts        | 143 +++++++++++++++++++++-
 apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx                     |  17 +--
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-014-erp-posting-postgres.integration.test.ts | 319 ++++++++++++++++++++++++++++++---------------------
 apps/mercato/src/modules/wms_outbound/services/shipment-posting-service.ts                 |  47 +++++++-
 4 files changed, 382 insertions(+), 144 deletions(-)
```

## Schema & Migration Facts (Literal Committed DDL)

Migration: `Migration20260904200000_wms_outbound_p1_014.ts` (applied to Testing PostgreSQL)

### Table: `wms_outbound_shipment_postings`
- **Columns:**
  - `id` (uuid, PRIMARY KEY, DEFAULT `gen_random_uuid()`)
  - `organization_id` (uuid, NOT NULL)
  - `tenant_id` (uuid, NOT NULL)
  - `warehouse_id` (uuid, NOT NULL)
  - `shipment_id` (uuid, NOT NULL)
  - `correlation_id` (text, NOT NULL)
  - `idempotency_key` (text, NOT NULL)
  - `status` (text, NOT NULL, DEFAULT `'POSTING_PENDING'`)
  - `attempt_count` (integer, NOT NULL, DEFAULT `1`)
  - `last_attempt_at` (timestamptz, NOT NULL, DEFAULT `NOW()`)
  - `posted_at` (timestamptz, NULL)
  - `last_error_category` (text, NULL)
  - `last_error_code` (text, NULL)
  - `last_error_message` (text, NULL)
  - `last_error_details` (jsonb, NULL)
  - `created_at` (timestamptz, NOT NULL, DEFAULT `NOW()`)
  - `updated_at` (timestamptz, NOT NULL, DEFAULT `NOW()`)
- **Indexes:**
  - `CREATE UNIQUE INDEX "wms_outbound_shipment_postings_shipment_uq" ON "wms_outbound_shipment_postings" ("organization_id", "tenant_id", "shipment_id")`
  - `CREATE UNIQUE INDEX "wms_outbound_shipment_postings_correlation_uq" ON "wms_outbound_shipment_postings" ("organization_id", "tenant_id", "correlation_id")`
  - `CREATE UNIQUE INDEX "wms_outbound_shipment_postings_idempotency_uq" ON "wms_outbound_shipment_postings" ("organization_id", "tenant_id", "idempotency_key")`
  - `CREATE INDEX "wms_outbound_shipment_postings_org_tenant_idx" ON "wms_outbound_shipment_postings" ("organization_id", "tenant_id")`
  - `CREATE INDEX "wms_outbound_shipment_postings_status_idx" ON "wms_outbound_shipment_postings" ("organization_id", "tenant_id", "status")`

### Table: `wms_outbound_shipment_posting_attempts`
- **Columns:**
  - `id` (uuid, PRIMARY KEY, DEFAULT `gen_random_uuid()`)
  - `organization_id` (uuid, NOT NULL)
  - `tenant_id` (uuid, NOT NULL)
  - `posting_id` (uuid, NOT NULL)
  - `shipment_id` (uuid, NOT NULL)
  - `attempt_number` (integer, NOT NULL)
  - `status` (text, NOT NULL)
  - `request_payload` (jsonb, NOT NULL)
  - `response_payload` (jsonb, NULL)
  - `error_category` (text, NULL)
  - `error_code` (text, NULL)
  - `error_message` (text, NULL)
  - `error_details` (jsonb, NULL)
  - `actor_id` (text, NULL)
  - `actor_role` (text, NULL)
  - `attempted_at` (timestamptz, NOT NULL, DEFAULT `NOW()`)
  - `completed_at` (timestamptz, NULL)
- **Indexes:**
  - `CREATE UNIQUE INDEX "wms_outbound_shipment_posting_attempts_posting_attempt_uq" ON "wms_outbound_shipment_posting_attempts" ("posting_id", "attempt_number")`
  - `CREATE INDEX "wms_outbound_shipment_posting_attempts_shipment_idx" ON "wms_outbound_shipment_posting_attempts" ("organization_id", "tenant_id", "shipment_id")`

*(Note: No database-level foreign key constraints are generated by the migration; relational integrity is enforced through MikroORM entity definitions and application transactional services.)*

## ERP Adapter & Error Category Vocabulary

- **Adapter Seam:** `ErpShipmentPostingAdapter` interface with contract: `postShipment(payload: ErpShipmentPostingPayload): Promise<ErpShipmentPostingResult>`.
- **Implementation:** `TestingErpShipmentPostingAdapter` provides deterministic test execution controls without fabricating external third-party ERP connectivity (Testing stub/contract; never claimed as REAL ERP).
- **Canonical Rejection Categories:**
  - `ERP_CONFIGURATION` — external/ERP master data or configuration error; retriable unchanged once ERP is reconfigured.
  - `WMS_DATA_DEFECT` — WMS payload / data defect; requires WMS data correction before retry.
- **In-Flight Concurrency & Idempotency Resolution (B1):**
  - When concurrent or repeated calls arrive while an initial posting request is in Phase 2 (`shipment.status === 'POSTING_PENDING'`), `postShipment()` Phase 1 detects the pending state under row lock and returns `{ success: true, replayed: true, inFlight: true, outcome: 'IN_FLIGHT' }`.
  - It does **not** throw an invalid-start-state error.
  - It does **not** create a second posting intent row.
  - It does **not** create a second `ShipmentPostingRequested` transition.
  - It does **not** create a second attempt row.
  - It does **not** call the ERP adapter a second time.
  - Later requests arriving after settlement (`POSTED`) return `{ success: true, replayed: true, inFlight: false, outcome: 'ACCEPTED' }`.

## Genuine PostgreSQL Verification (18/18 PASSED)

Executed against remote Testing PostgreSQL (`aws-1-eu-central-1.pooler.supabase.com:6543`, database `postgres`, PostgreSQL 17.6):

```text
  console.log
    [P1-014 erp-posting exact A<->B lock contention] {
      pidA: 2051333,
      pidB: 2051359,
      blockedPid: 2051359,
      blockingPids: [ 2051333 ],
      waitEventType: 'Lock'
    }

Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        69.624 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/p1-014-erp-posting-postgres.integration.test.ts.
```

### Complete Test Inventory (18/18)

1. `1. Initial posting from LABEL_GENERATED creates one durable correlated posting intent in POSTING_PENDING (P1 R35, TC-001)`
2. `2. Initial posting from OWN_TRANSPORT follows identical ERP gate without label requirement (P1 R35, FR-P1-18)`
3. `3. Invalid starting Shipment status rejects initial posting (P1 R35)`
4. `4. Accepted correlated ERP response yields POSTED and ShipmentPosted transition exactly once (P1 R35, TC-001)`
5. `5. Explicit structured ERP rejection yields POSTING_ERROR without touching packed physical contents (P1 R36, TC-007)`
6. `6. ERP-side content rejection (ERP_CONFIGURATION) can be retried unchanged after external repair (P1 R37, TC-007)`
7. `7. WMS-side content rejection (WMS_DATA_DEFECT) requires correction before retry (P1 R37, TC-007)`
8. `8. Technical timeout / unreachable response does NOT become business POSTING_ERROR (P1 step 11a boundary)`
9. `9. Non-Supervisor retry fails closed (P1 R37, server-authoritative RBAC)`
10. `10. Supervisor retry POSTING_ERROR -> POSTING_PENDING creates a new append-only attempt (P1 R37, TC-007)`
11. `11. Non-Supervisor cancellation from POSTING_ERROR fails closed (P1 step 11a, TC-008)`
12. `12. Supervisor give-up POSTING_ERROR -> CANCELLED succeeds only from that exact state (P1 step 11a, TC-008)`
13. `13. Direct cancellation from POSTING_PENDING or POSTED is strictly rejected (P1 step 11a, TC-008)`
14. `14. Genuine PostgreSQL Concurrency: overlapping actual service calls serialize on Shipment row lock with pg_blocking_pids proof (P1 R35, B2 remediation)`
15. `15. Duplicate / concurrent request arriving while in-flight or post-settlement is idempotent and invokes adapter exactly once (P1 R35, CON-05, B1 remediation)`
16. `16. Real PostgreSQL rollback test: failure after write/flush leaves no partial durable state (CON-05)`
17. `17. Organization / tenant isolation for posting intent, attempt, and response (INT-04, tenancy invariant)`
18. `18. POSTED -> IN_MANIFEST is NOT implemented by P1-014 (strict scope boundary, P1-015 exclusion)`

### Decisive Actual-Service Concurrency & In-Flight Proof (B1 & B2 Remediation)

- **Test 14 (Real Concurrency & Lock Contention Proof):**
  - Two independent service instances (`serviceA` on `emA`, `serviceB` on `emB`) call `postShipment()` on the same real `LABEL_GENERATED` shipment.
  - Operation A enters Phase 1 and holds row lock via `SELECT ... FOR UPDATE`.
  - Operation B enters `postShipment()` concurrently and blocks on the Shipment row lock.
  - Diagnostic query on `pg_stat_activity` proves exact backend PID contention:
    - `pidA`: `2051333`
    - `pidB`: `2051359`
    - `blockedPid`: `2051359`
    - `blockingPids`: `[ 2051333 ]`
    - `waitEventType`: `'Lock'`
  - Both operations complete cleanly; fresh independent read verifies exactly 1 posting record and non-regressive state.

- **Test 15 (In-Flight Duplicate & Settlement Idempotency Proof):**
  - Operation A starts `postShipment()` and enters Phase 2 (adapter execution pauses deterministically).
  - Duplicate Operation B calls `postShipment()` while Shipment is in `POSTING_PENDING`.
  - Operation B receives `{ success: true, replayed: true, inFlight: true, outcome: 'IN_FLIGHT' }` without throwing an error and without calling the ERP adapter.
  - Operation A's adapter pause is released, advancing Shipment to `POSTED`.
  - Subsequent Operation C calls `postShipment()` after settlement; returns `{ success: true, replayed: true, outcome: 'ACCEPTED' }`.
  - Fresh independent database reads prove:
    - Exactly 1 row in `wms_outbound_shipment_postings`
    - Exactly 1 row in `wms_outbound_shipment_posting_attempts`
    - Exactly 1 `ShipmentPostingRequested` transition event
    - Exactly 1 `ShipmentPosted` transition event
    - Total ERP adapter calls across calls A, B, and C: **exactly 1**.

- **Test 16 (Real Transaction Rollback Proof):**
  - Phase 1 failure after write/flush triggers rollback.
  - Independent database inspection proves 0 posting rows and 0 attempt rows remained.

## Regression Test Results

| Suite | Result | Details |
|---|---|---|
| **P1-014 PostgreSQL Suite** | **18/18 PASSED** | Remediated actual-service concurrency and in-flight duplicate handling |
| **P1-013 PostgreSQL Suite** | **15/15 PASSED** | Label generation, carrier correction, lock contention proof (`pidA: 2051333`, `pidB: 2051359`) |
| **P1-012 PostgreSQL Suite** | **14/14 PASSED** | Carrier auto-assignment, manual override, audit history |
| **P1-011 PostgreSQL Suite** | **18/18 PASSED** | Shipment grouping, aggregation, concurrency proof |
| **FND-002 State Machine Suite** | **77/77 PASSED** | Core outbound state machine transition rules |
| **FND-002 Transaction Suite** | **8/8 PASSED** | Real PostgreSQL transaction simulation invariants |

## Playwright Rendered UI Verification (5/5 PASSED)

Executed against `https://devaxonic-test.info-start.com.pl` (served via `mercato-localhost.service` on commit `bef7c0a3e`):

```text
Running 5 tests using 1 worker

     1 …ppy ERP posting path advances LABEL_GENERATED to POSTED (P1 R35, TC-001)
  ✓  1 …posting path advances LABEL_GENERATED to POSTED (P1 R35, TC-001) (17.5s)
     2 …tion surfaces POSTING_ERROR and structured error banner (P1 R36, TC-007)
  ✓  2 …faces POSTING_ERROR and structured error banner (P1 R36, TC-007) (13.5s)
     3 …tries POSTING_ERROR to POSTED with full attempt history (P1 R37, TC-007)
  ✓  3 …STING_ERROR to POSTED with full attempt history (P1 R37, TC-007) (13.9s)
     4 …Give-Up / Cancellation boundary from POSTING_ERROR (P1 step 11a, TC-008)
  ✓  4 …/ Cancellation boundary from POSTING_ERROR (P1 step 11a, TC-008) (14.1s)
     5 …does not become business POSTING_ERROR (P1 R37, INT-04, P1-015 boundary)
  ✓  5 … become business POSTING_ERROR (P1 R37, INT-04, P1-015 boundary) (20.2s)
  5 passed (1.5m)
```

### Rendered Journey Details (5/5)

- **Journey A (PLAYWRIGHT VERIFIED):** Happy ERP posting path advances `LABEL_GENERATED` to `POSTED` (P1 R35, TC-001)
  - Real user clicks "Post to ERP".
  - Shipment transitions from `LABEL_GENERATED` to `POSTED`.
  - ERP Posting Details card displays `status: POSTED`, attempt count 1, and no manifest action exists.
- **Journey B (PLAYWRIGHT VERIFIED):** Explicit ERP rejection surfaces `POSTING_ERROR` and structured error banner (P1 R36, TC-007)
  - Simulated ERP rejection transitions shipment to `POSTING_ERROR`.
  - Structured rejection banner renders with category `ERP_CONFIGURATION`.
  - Packed TU contents and weights remain completely intact.
- **Journey C (PLAYWRIGHT VERIFIED):** Authorized Supervisor retries `POSTING_ERROR` to `POSTED` with full attempt history (P1 R37, TC-007)
  - Supervisor clicks "Retry ERP Posting".
  - Shipment returns to `POSTING_PENDING` and advances to `POSTED`.
  - Attempts audit history shows Attempt #1 (`REJECTED`) and Attempt #2 (`ACCEPTED`).
- **Journey D (PLAYWRIGHT VERIFIED):** Supervisor Give-Up / Cancellation boundary from `POSTING_ERROR` (P1 step 11a, TC-008)
  - Supervisor clicks "Cancel Shipment (Give-Up)".
  - Shipment transitions from `POSTING_ERROR` to `CANCELLED`.
  - Re-attempt buttons disappear; `CANCELLED` state is final.
- **Journey E (PLAYWRIGHT VERIFIED):** Unauthorized operator role fails closed, manifest action absence, and timeout boundary (P1 R37, INT-04, P1-015 boundary)
  - Authenticated as real operator (`packer_operator_p10@devaxonic.local`, role `WMS Standard Packer Role` with `wms_outbound.view` + `wms_outbound.edit`, lacking `wms_outbound.manage_orders`).
  - Attempted retry from `POSTING_ERROR` is blocked by server-authoritative RBAC; surfaces structured UI error banner: *"Only a Warehouse Supervisor is authorized to retry ERP posting from POSTING_ERROR (P1 step 11a, P1 R37)."*
  - Shipment remains in `POSTING_ERROR` in PostgreSQL.
  - Shipment detail page in `POSTED` status is inspected: no button or UI action matching "manifest" or "add to manifest" exists.
  - Technical timeout / unreachable response leaves shipment in `POSTING_PENDING`; database inspection proves status does **not** become business `POSTING_ERROR`.

## Worktree Status

```text
Devaxonic-mercato: clean (outbound/p1-014 @ bef7c0a3e0995e7ecddb29156bdfa3777463a6b6)
Devaxonic-scanner: clean (main @ f4a404600efb1120cb2f1c5b86383ad148cd1e1a)
WMS_Outbound:      clean (main)
Devaxonic-WMS:     clean (main @ a1d9348826d911b92e9ff74cf44ddbf0d6fa51a1)
```

## Explicit Scope Exclusions Obeyed

- **NO P1-015 Manifest Lifecycle:** `POSTED → IN_MANIFEST` is not implemented in P1-014 (verified by Playwright Journey E and automated test #18).
- **NO P1-016 Settlement:** No order/line/inventory settlement logic introduced.
- **NO Crossdock / P2-005:** Preserved for P2.
- **NO Scanner Changes:** Scanner repository frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- **NO Prod/Demo Changes:** All execution occurred strictly on Testing environment.
- **NO Acceptance Claims:** Evidence is candidate evidence only (`PLAYWRIGHT VERIFIED` and `REAL PostgreSQL INTEGRATION VERIFIED`). Steering files (`STATE.md`, handovers) remain untouched for Owner acceptance.
