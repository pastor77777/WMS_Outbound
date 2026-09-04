# P1-014 Evidence — ERP Shipment POST, Error State and Safe Retry

**Task:** P1-014 — ERP Shipment POST, error state and safe retry (Process 1 `STANDARD_FULFILLMENT` Step 11A, P1 R35–R38, `FR-P1-18`, `FR-P1-19`, `INT-04`, `INT-05`, `CON-05`, `TC-001`, `TC-007`, `TC-008`)  
**Execution guide:** `06_AGENT_GUIDES/P1-014_EXECUTION.md`  
**Evidence date:** 2026-09-04  
**Evidence class:** REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED  

## Final Revisions and Runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato final P1-014 head | `d20d95a097f9fe4b4459d6829fbd459c80a5efd3` on `outbound/p1-014` (pushed) |
| Accepted P1-013 base | `5e6b70aa81afd28fe3217e4aad216e8a6482a769` |
| Lineage / compare | Exactly 1 commit ahead of accepted P1-013 base; merge base: `5e6b70aa81afd28fe3217e4aad216e8a6482a769` |
| Served Testing Mercato runtime revision | `d20d95a097f9fe4b4459d6829fbd459c80a5efd3` (served via `mercato-localhost.service` at `https://devaxonic-test.info-start.com.pl`) |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` (verified untouched, clean working tree) |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:6543`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `https://devaxonic-test.info-start.com.pl` returned HTTP 200/307 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Exact Git Compare Statistics

Git compare: `5e6b70aa81afd28fe3217e4aad216e8a6482a769..d20d95a097f9fe4b4459d6829fbd459c80a5efd3`

```text
 apps/mercato/src/modules/wms_outbound/__integration__/P1-014-erp-posting-ui.spec.ts  |  419 ++++++++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/cancel-posting-error/route.ts   |   41 +
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/post/route.ts  |   35 +
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/retry-posting/route.ts      |   41 +
 apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx   |  291 ++++++
 apps/mercato/src/modules/wms_outbound/data/entities.ts      |  138 +++
 apps/mercato/src/modules/wms_outbound/di.ts        |   13 +
 apps/mercato/src/modules/wms_outbound/migrations/Migration20260904200000_wms_outbound_p1_014.ts |   95 ++
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-014-erp-posting-postgres.integration.test.ts | 1058 ++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/erp-shipment-posting-adapter.ts       |  130 +++
 apps/mercato/src/modules/wms_outbound/services/shipment-posting-service.ts           |  834 +++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/shipment-service.ts      |   22 +
 12 files changed, 3117 insertions(+)
```

## Schema & Migration Impact

- Migration: `Migration20260904200000_wms_outbound_p1_014.ts` (applied to Testing PostgreSQL)
- **Table: `wms_outbound_shipment_postings`**
  - Columns: `id` (uuid PK), `organization_id` (uuid), `tenant_id` (uuid), `warehouse_id` (uuid), `shipment_id` (uuid), `posting_key` (text), `posting_status` (text: `PENDING`, `ACCEPTED`, `REJECTED`), `attempt_count` (int), `last_attempt_number` (int), `last_status` (text), `last_error_code` (text, nullable), `last_error_message` (text, nullable), `last_error_category` (text, nullable), `last_attempted_at` (timestamptz, nullable), `posted_at` (timestamptz, nullable), `created_at` (timestamptz), `updated_at` (timestamptz)
  - Unique index: `wms_outbound_shipment_postings_org_tenant_shipment_idx` ON `(organization_id, tenant_id, shipment_id)`
  - Unique index: `wms_outbound_shipment_postings_org_tenant_posting_key_idx` ON `(organization_id, tenant_id, posting_key)`
  - FK: `fk_wms_outbound_shipment_postings_shipment_id` -> `wms_outbound_shipments(id)` ON DELETE CASCADE
- **Table: `wms_outbound_shipment_posting_attempts`**
  - Columns: `id` (uuid PK), `organization_id` (uuid), `tenant_id` (uuid), `warehouse_id` (uuid), `posting_id` (uuid), `shipment_id` (uuid), `attempt_number` (int), `correlation_id` (text), `idempotency_key` (text), `outcome` (text: `ACCEPTED`, `REJECTED`), `error_category` (text, nullable: `ERP_CONFIGURATION`, `WMS_DATA_CORRECTION_REQUIRED`), `error_code` (text, nullable), `error_message` (text, nullable), `payload_snapshot` (jsonb), `response_snapshot` (jsonb), `attempted_by` (text, nullable), `created_at` (timestamptz), `updated_at` (timestamptz)
  - Unique index: `wms_outbound_shipment_posting_attempts_org_tenant_posting_attempt_idx` ON `(organization_id, tenant_id, posting_id, attempt_number)`
  - Unique index: `wms_outbound_shipment_posting_attempts_org_tenant_idempotency_idx` ON `(organization_id, tenant_id, idempotency_key)`
  - FK: `fk_wms_outbound_shipment_posting_attempts_posting_id` -> `wms_outbound_shipment_postings(id)` ON DELETE CASCADE
  - FK: `fk_wms_outbound_shipment_posting_attempts_shipment_id` -> `wms_outbound_shipments(id)` ON DELETE CASCADE

## ERP Adapter & Idempotency Design

- **Adapter Seam:** `ErpShipmentPostingAdapter` interface defines `postShipment(payload: ErpShipmentPostingPayload): Promise<ErpShipmentPostingResult>`.
- **Implementation:** `TestingErpShipmentPostingAdapter` implements the adapter seam with deterministic test controls (via payload markers or injection) for happy path, structured ERP configuration rejection, WMS-side data correction rejection, and timeout/unreachable technical exceptions.
- **Idempotency & Concurrency:**
  - Posting intent is locked at the database boundary via `SELECT ... FOR UPDATE` on `wms_outbound_shipments`.
  - Exactly-once posting record ensured by unique constraint on `(organization_id, tenant_id, shipment_id)` and `(organization_id, tenant_id, posting_key)`.
  - Attempts are append-only with unique `(posting_id, attempt_number)` and `idempotency_key`.
  - Replay of duplicate calls or concurrent requests is safe and returns the existing result without side effects.

## Genuine PostgreSQL Verification (18/18 PASSED)

Executed against remote Testing PostgreSQL (`aws-1-eu-central-1.pooler.supabase.com:6543`, database `postgres`, PostgreSQL 17.6):

```text
  console.log
    [P1-014 erp-posting exact A<->B lock contention] {
      pidA: 2046498,
      pidB: 2046463,
      blockedPid: 2046463,
      blockingPids: [ 2046498 ],
      waitEventType: 'Lock'
    }

 PASS  apps/mercato/src/modules/wms_outbound/services/__tests__/p1-014-erp-posting-postgres.integration.test.ts (28.452 s)
  P1-014 ERP Shipment Posting PostgreSQL Integration Tests
    ✓ 1. Initial posting from LABEL_GENERATED creates one durable correlated posting intent in POSTING_PENDING (P1 R35, TC-001) (678 ms)
    ✓ 2. Initial posting from OWN_TRANSPORT follows identical ERP gate without label requirement (P1 R35, FR-P1-18) (512 ms)
    ✓ 3. Invalid starting Shipment status rejects initial posting (P1 R35) (421 ms)
    ✓ 4. Accepted correlated ERP response yields POSTED and ShipmentPosted transition exactly once (P1 R35, TC-001) (589 ms)
    ✓ 5. Explicit structured ERP rejection yields POSTING_ERROR without touching packed physical contents (P1 R36, TC-007) (612 ms)
    ✓ 6. ERP-side content rejection (ERP_CONFIGURATION) can be retried unchanged after external repair (P1 R37, TC-007) (704 ms)
    ✓ 7. WMS-side content rejection (WMS_DATA_CORRECTION_REQUIRED) requires correction before retry (P1 R37, TC-007) (641 ms)
    ✓ 8. Technical timeout / unreachable response does NOT become business POSTING_ERROR (P1 step 11a boundary) (502 ms)
    ✓ 9. Non-Supervisor retry fails closed (P1 R37, server-authoritative RBAC) (432 ms)
    ✓ 10. Supervisor retry POSTING_ERROR -> POSTING_PENDING creates a new append-only attempt (P1 R37, TC-007) (698 ms)
    ✓ 11. Non-Supervisor cancellation from POSTING_ERROR fails closed (P1 step 11a, TC-008) (441 ms)
    ✓ 12. Supervisor give-up POSTING_ERROR -> CANCELLED succeeds only from that exact state (P1 step 11a, TC-008) (623 ms)
    ✓ 13. Direct cancellation from POSTING_PENDING or POSTED is strictly rejected (P1 step 11a, TC-008) (489 ms)
    ✓ 14. Genuine PostgreSQL Concurrency: overlapping initial post calls serialize on Shipment row lock with pg_blocking_pids proof (P1 R35, exactly-once) (1842 ms)
    ✓ 15. Duplicate / concurrent accepted ERP response handling is idempotent and exactly-once (P1 R35, CON-05) (611 ms)
    ✓ 16. Real PostgreSQL rollback leaves no partial durable posting state (P1 R35, CON-05) (528 ms)
    ✓ 17. Organization / tenant isolation for posting intent, attempt, and response (INT-04, tenancy invariant) (587 ms)
    ✓ 18. POSTED -> IN_MANIFEST is NOT implemented by P1-014 (strict scope boundary, P1-015 exclusion) (398 ms)

Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        28.452 s
```

### Decisive Concurrency & Contention Proof
- **Exact PIDs captured:**
  - Operation A connection backend PID: `2046498`
  - Operation B connection backend PID: `2046463`
- **Observed `pg_stat_activity` query:**
  - Blocked PID: `2046463`
  - Blocking PIDs: `[ 2046498 ]`
  - Wait event type: `'Lock'`
- Both operations targeted the same `wms_outbound_shipments` row; Operation B blocked until Operation A committed; Operation B unblocked and safely returned the already-posted state without duplicate attempts or duplicate transitions.

## Regression Test Results

| Suite | Result | Details |
|---|---|---|
| **P1-014 PostgreSQL Suite** | **18/18 PASSED** | All ERP posting, retry, give-up, concurrency, tenant isolation tests passed |
| **P1-013 PostgreSQL Suite** | **15/15 PASSED** | Label generation, carrier correction, genuine concurrency proof passed |
| **P1-012 PostgreSQL Suite** | **14/14 PASSED** | Carrier auto-assignment, manual override, audit history passed |
| **P1-011 PostgreSQL Suite** | **18/18 PASSED** | Shipment grouping, aggregation, concurrency proof passed |
| **FND-002 State Machine Suite** | **77/77 PASSED** | Core state machine transition rules intact |
| **FND-002 Transaction Suite** | **8/8 PASSED** | Real PostgreSQL transaction simulation invariants passed |

## Playwright Rendered UI Verification (4/4 PASSED)

Executed against `https://devaxonic-test.info-start.com.pl` (Mercato localhost service):

```text
Running 4 tests using 1 worker

     1 …ppy ERP posting path advances LABEL_GENERATED to POSTED (P1 R35, TC-001)
  ✓  1 …posting path advances LABEL_GENERATED to POSTED (P1 R35, TC-001) (14.0s)
     2 …tion surfaces POSTING_ERROR and structured error banner (P1 R36, TC-007)
  ✓  2 …faces POSTING_ERROR and structured error banner (P1 R36, TC-007) (13.7s)
     3 …tries POSTING_ERROR to POSTED with full attempt history (P1 R37, TC-007)
  ✓  3 …STING_ERROR to POSTED with full attempt history (P1 R37, TC-007) (14.1s)
     4 …Give-Up / Cancellation boundary from POSTING_ERROR (P1 step 11a, TC-008)
  ✓  4 …/ Cancellation boundary from POSTING_ERROR (P1 step 11a, TC-008) (14.0s)
  4 passed (1.1m)
```

- **Journey A (PLAYWRIGHT VERIFIED):** Happy ERP posting path advances `LABEL_GENERATED` to `POSTED` (P1 R35, TC-001)
  - Shipment in `LABEL_GENERATED` posts to ERP via "Post to ERP" button.
  - Transitions to `POSTED`.
  - ERP Posting Details card displays status `ACCEPTED`, attempt count 1, and no manifest actions.
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

## Worktree Status

```text
Devaxonic-mercato: clean (git status --short: empty)
Devaxonic-scanner: clean (git status --short: empty)
WMS_Outbound:      clean (git status --short: empty)
```

## Explicit Scope Exclusions Obeyed

- **NO P1-015 Manifest Lifecycle:** `POSTED → IN_MANIFEST` is not implemented in P1-014 (verified by automated test #18).
- **NO P1-016 Settlement:** No order/line/inventory settlement logic introduced.
- **NO Crossdock / P2-005:** Preserved for P2.
- **NO Scanner Changes:** Scanner repository frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- **NO Prod/Demo Changes:** All execution occurred strictly on Testing environment.
- **NO Acceptance Claims:** Evidence is `PLAYWRIGHT VERIFIED` and `REAL PostgreSQL INTEGRATION VERIFIED`. Steering files (`STATE.md`, handovers) remain untouched for Owner acceptance.
