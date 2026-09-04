# P1-013 Evidence — Remediation & Closeout Candidate

**Task:** P1-013 — WMS label generation and pre-manifest loading carrier correction (Step 11, P1 R33–R36, FR-P1-17, FR-P1-18, FR-P5-11, TC-001, TC-007, TC-066)  
**Closeout guide:** `06_AGENT_GUIDES/P1-013_CLOSEOUT.md`  
**Remediation guide:** `06_AGENT_GUIDES/P1-013_REMEDIATION.md`  
**Execution guide:** `06_AGENT_GUIDES/P1-013_EXECUTION.md`  
**Evidence date:** 2026-09-04  
**Evidence class:** REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED  

## Final Revisions and Runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato final closeout head | `5e6b70aa81afd28fe3217e4aad216e8a6482a769` on `outbound/p1-013` (pushed) |
| Pre-closeout P1-013 head | `ebc556edd2366a4bb45351924ae2f22a1cafb093` on `outbound/p1-013` |
| Pre-remediation P1-013 head | `7f8185c3eccaf1b04fd46027616b0286d4c87fd1` on `outbound/p1-013` |
| Accepted P1-012 base | `5019a20be14549ff8cbbf25af5bc61c56888e9e1` |
| Lineage / compare | Exactly 3 commits ahead of accepted P1-012 base (`5019a20be -> 7f8185c3e -> ebc556edd -> 5e6b70aa8`); merge base: `5019a20be14549ff8cbbf25af5bc61c56888e9e1` |
| Served Testing Mercato runtime revision | `7f8185c3eccaf1b04fd46027616b0286d4c87fd1` (served via `mercato-localhost.service` at `https://devaxonic-test.info-start.com.pl`; product runtime files are 100% identical to `5e6b70aa8`, as subsequent commits modified only test code) |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` (verified untouched, clean working tree) |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:6543`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `https://devaxonic-test.info-start.com.pl` returned HTTP 200 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Remediation & Closeout Scope Resolved

### 1. Exact A↔B PostgreSQL Lock Contention Proof
- Strengthened test 15 (`15. Genuine PostgreSQL Concurrency: overlapping label generation calls serialize on Shipment row lock with pg_blocking_pids proof`) in `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts`.
- **Exact PID capture & query mechanism:**
  - Operation A opens a transactional session on `emA` and directly queries its backend connection PID via `SELECT pg_backend_pid() AS pid` (`pidA: 2042578`).
  - Operation A invokes `ShipmentLabelService.generateLabel` which acquires the `wms_outbound_shipments` row lock via `LockMode.PESSIMISTIC_WRITE` (`SELECT ... FOR UPDATE`), persists the label and updates status to `LABEL_GENERATED`, but intentionally holds the transaction open before commit.
  - Operation B opens a transactional session on `emB` and directly queries its backend connection PID via `SELECT pg_backend_pid() AS pid` (`pidB: 2042579`).
  - Operation B invokes `generateLabel` on `emB` against the exact same Shipment and becomes blocked by PostgreSQL tuple-lock contention.
  - An independent `observer` client queries `pg_stat_activity` specifically for `pid = $1` with parameter `[pidB]`:
    ```sql
    SELECT pid, pg_blocking_pids(pid) AS blocking_pids, wait_event_type, query
    FROM pg_stat_activity
    WHERE pid = $1 AND cardinality(pg_blocking_pids(pid)) > 0
    ```
  - The observer confirms:
    - `observed.pid === pidB` (`2042579`)
    - `wait_event_type === 'Lock'`
    - `pg_blocking_pids(pidB)` contains `pidA` (`[ 2042578 ]`)
    - `pidA !== pidB` (`2042578 !== 2042579`)
  - Operation A is released and commits.
  - Operation B unblocks, reads the committed `LABEL_GENERATED` shipment, safely replays via the existing label branch, and returns the existing label without creating a duplicate record or transition event.
  - Fresh independent reads prove:
    - Shipment status is `LABEL_GENERATED`
    - Exactly 1 `wms_outbound_shipment_labels` row exists
    - Exactly 1 `ShipmentLabelGenerated` transition event exists
    - Label `printCount === 0` (no unintended print side effect)

### 2. File-Stat Mismatch & Literal Test Inventory Resolved
- Regenerated exact file stats directly from Git compare `5019a20be14549ff8cbbf25af5bc61c56888e9e1..5e6b70aa81afd28fe3217e4aad216e8a6482a769`.
- Literal test inventory rewritten from actual final-head test execution (15/15 passed).

## Exact Git Compare Statistics

Git compare: `5019a20be14549ff8cbbf25af5bc61c56888e9e1..5e6b70aa81afd28fe3217e4aad216e8a6482a769`

```text
 apps/mercato/src/modules/wms_outbound/__integration__/P1-013-label-generation-ui.spec.ts | 466 ++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/generate-label/route.ts     |  35 ++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/print-label/route.ts        |  29 ++
 apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx       | 171 ++++++-
 apps/mercato/src/modules/wms_outbound/data/entities.ts                      |  58 +++
 apps/mercato/src/modules/wms_outbound/data/transitions.ts                   |   1 +
 apps/mercato/src/modules/wms_outbound/di.ts                                 |   7 +
 apps/mercato/src/modules/wms_outbound/migrations/Migration20260904190000_wms_outbound_p1_013.ts  |  46 ++
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts     |   5 +-
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts | 909 ++++++++++++++++++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/carrier-selection-service.ts |  21 +-
 apps/mercato/src/modules/wms_outbound/services/shipment-label-service.ts    | 296 ++++++++++++
 apps/mercato/src/modules/wms_outbound/services/shipment-service.ts          |   8 +
 13 files changed, 2041 insertions(+), 11 deletions(-)
```

## Schema & Migration Impact

- Migration: `Migration20260904190000_wms_outbound_p1_013.ts`
- Table: `wms_outbound_shipment_labels`
- Columns: `id` (uuid, PK), `organization_id` (uuid), `tenant_id` (uuid), `warehouse_id` (uuid), `shipment_id` (uuid), `carrier_code` (text), `label_payload` (jsonb), `status` (text), `print_count` (int), `last_printed_at` (timestamptz, nullable), `generated_by` (text, nullable), `created_at` (timestamptz), `updated_at` (timestamptz)
- Unique Index: `(organization_id, tenant_id, shipment_id)`
- FK Constraint: `fk_wms_outbound_shipment_labels_shipment_id` -> `wms_outbound_shipments(id)` ON DELETE CASCADE

## Genuine PostgreSQL Verification (15/15 PASSED)

Executed against remote Testing PostgreSQL (`aws-1-eu-central-1.pooler.supabase.com:6543`, database `postgres`, PostgreSQL 17.6):

```text
  console.log
    [P1-013 label-generation exact A<->B lock contention] {
      pidA: 2042578,
      pidB: 2042579,
      blockedPid: 2042579,
      blockingPids: [ 2042578 ],
      waitEventType: 'Lock'
    }

      at Object.<anonymous> (src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts:871:15)

PASS src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts (25.6s)
  P1-013 Genuine PostgreSQL Label Generation & Carrier Correction Suite
    ✓ 1. Start gate: non-selected or pending shipment cannot generate label (P1 R34)
    ✓ 2. Gate: CARRIER_PENDING shipment cannot generate label (P1 R34)
    ✓ 3. OWN_TRANSPORT skips/rejects label generation per Architect specification (P1 step 10/11, P1 R34)
    ✓ 4. Generated label is based strictly on WMS-owned persisted Shipment/Packing TU/Carrier/address data (P1 R34)
    ✓ 5. Successful generation persists authoritative label evidence and transitions to LABEL_GENERATED (P1 step 11, R34)
    ✓ 6. Repeated generation / retry is non-regressive and idempotent (P1 R34)
    ✓ 7. Local print / reprint does not call external API and does not regress status (P1 R34)
    ✓ 8. EXTERNAL missing-maxVolume path cannot generate until Supervisor approval completed (P1 R51, FR-P5-10, TC-066)
    ✓ 9. Real Supervisor carrier correction is allowed at pre-close boundary (P1 R35, FR-P1-18)
    ✓ 10. Carrier correction does not automatically regenerate or reprint existing label (P1 R35, FR-P5-11)
    ✓ 11. Unauthorized non-supervisor carrier correction is rejected (P1 R33, R35)
    ✓ 12. Architecture invariant: no provider rejection or invented label error state (FR-P5-11)
    ✓ 13. Real rollback test: failure after write/flush in transaction rolls back completely with fresh independent read proof
    ✓ 14. PostgreSQL Unique constraint prevents duplicate label records per shipment
    ✓ 15. Genuine PostgreSQL Concurrency: overlapping label generation calls serialize on Shipment row lock with pg_blocking_pids proof

Test Suites: 1 passed, 1 total
Tests:       15 passed, 15 total
Snapshots:   0 total
Time:        25.613 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts.
```

## Regression Verification

### P1-012 Carrier Selection PostgreSQL Suite (14/14 PASSED)

```text
PASS src/modules/wms_outbound/services/__tests__/p1-012-carrier-selection-postgres.integration.test.ts (32.6s)
Test Suites: 1 passed, 1 total
Tests:       14 passed, 14 total
Snapshots:   0 total
Time:        32.636 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/p1-012-carrier-selection-postgres.integration.test.ts.
```

### P1-011 Shipment Grouping PostgreSQL Suite (18/18 PASSED)

```text
  console.log
    [P1-011 distinct-TU grouping-key lock contention] {
      blockedPid: 2041713,
      blockingPids: [ 2041714 ],
      waitEventType: 'Lock'
    }

      at Object.<anonymous> (src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts:590:15)

PASS src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts (71.9s)
Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        71.902 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts.
```

### State Transitions Invariant Suite (77/77 PASSED)

```text
PASS src/modules/wms_outbound/services/__tests__/fnd-002-state-transitions.test.ts (2.1s)
Test Suites: 1 passed, 1 total
Tests:       77 passed, 77 total
Snapshots:   0 total
Time:        2.127 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/fnd-002-state-transitions.test.ts.
```

## Fresh Final-Head Rendered UI Evidence (Playwright)

Executed using Playwright against canonical testing URL `https://devaxonic-test.info-start.com.pl` through real Chromium browser automation from the served revision `7f8185c3eccaf1b04fd46027616b0286d4c87fd1`:

```text
Running 4 tests using 1 worker

  ✓  1 Journey 1 (PLAYWRIGHT VERIFIED): Normal label generation and print from CARRIER_SELECTED (P1 R34) (1.0m)
  ✓  2 Journey 2 (PLAYWRIGHT VERIFIED): Pending approval gate blocks label generation until Supervisor approval (P1 R51, TC-066) (14.7s)
  ✓  3 Journey 3 (PLAYWRIGHT VERIFIED): Supervisor carrier correction after label generation before manifest close (P1 R35) (14.2s)
  ✓  4 Journey 4 (PLAYWRIGHT VERIFIED): Architecture boundary: no provider rejection or external API mode (FR-P5-11) (12.6s)

  4 passed (1.9m)
```

*(Note: Playwright automated test output; not claimed as HUMAN VERIFIED per project standards).*

## Worktree Cleanliness Proof

```bash
$ cd /home/ubuntu/git/Devaxonic-mercato && git status --short
(clean - 0 modified, 0 untracked)

$ cd /home/ubuntu/git/WMS_Outbound && git status --short
(clean - 0 modified, 0 untracked)
```

## Explicit Exclusions & Deferred Scope

- External carrier Label API: Not implemented / rejected per Architect specification.
- Carrier acceptance/rejection mode: Not implemented / rejected per FR-P5-11.
- Invented `LABEL_ERROR` state machine: Not introduced.
- ERP POST / `POSTING_PENDING` / `POSTED` (P1-014): Deferred to authorized P1-014.
- `CarrierManifest` lifecycle & post-CLOSED boundary (P1-015): Deferred to authorized P1-015.
- Order settlement & final inventory deduction (P1-016): Deferred to authorized P1-016.
- Scanner application: Untouched and frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- Production / Demo environments: Untouched.
