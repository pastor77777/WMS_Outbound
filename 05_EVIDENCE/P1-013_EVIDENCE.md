# P1-013 Evidence — Execution & Verification Candidate

**Task:** P1-013 — WMS label generation and pre-manifest loading carrier correction (Step 11, P1 R33–R36, FR-P1-17, FR-P1-18, FR-P5-11, TC-001, TC-007, TC-066)  
**Execution guide:** `06_AGENT_GUIDES/P1-013_EXECUTION.md`  
**Evidence date:** 2026-09-04  
**Evidence class:** REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED  

## Final Revisions and Runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato P1-013 head | `7f8185c3eccaf1b04fd46027616b0286d4c87fd1` on `outbound/p1-013` (pushed) |
| Accepted P1-012 base | `5019a20be14549ff8cbbf25af5bc61c56888e9e1` |
| Lineage / compare | Exactly 1 commit ahead of accepted P1-012 base (`5019a20be -> 7f8185c3e`); merge base: `5019a20be14549ff8cbbf25af5bc61c56888e9e1` |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` (verified untouched, clean working tree) |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:6543`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `https://devaxonic-test.info-start.com.pl` returned HTTP 200 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Architecture & Implementation Scope

### 1. WMS-Owned Label Model & Local Generation (P1 R34, FR-P1-17, FR-P5-11)
- Implemented `WmsOutboundShipmentLabel` entity mapped to `wms_outbound_shipment_labels` table with unique constraint on `(organization_id, tenant_id, shipment_id)`.
- Applied migration `Migration20260904190000_wms_outbound_p1_013.ts` to Testing PostgreSQL.
- Implemented `ShipmentLabelService` (`generateLabel`, `printLabel`, `getLabel`):
  - Strict gate on `status === 'CARRIER_SELECTED'`.
  - Rejection on `OWN_TRANSPORT` / customer pickup (`400 Bad Request` — no shipping label required).
  - WMS-owned data assembly: derives sender address, delivery address, carrier code, total weight, total volume, TU packaging breakdown strictly from persisted local records.
  - Generates stable local tracking reference `TRK-<SHIPMENT_NUMBER>-WMS`.
  - Advances shipment status atomically to `LABEL_GENERATED` (`ShipmentLabelGenerated` domain event).
  - Idempotent replay: repeated generation requests on `LABEL_GENERATED` return the existing label payload safely without duplicating state transitions or artifacts.

### 2. Local Print Action (P1 R34, FR-P1-18)
- Local print increments `print_count` and updates `last_printed_at` on the local label record.
- Operates purely on local WMS state without calling external carrier APIs or modifying the shipment state machine.

### 3. Pre-Manifest Loading Carrier Correction (Step 11, P1 R35, P1 R33, TC-007)
- Extended `CarrierSelectionService.manualSelectCarrier` to permit carrier reassignment when `status === 'LABEL_GENERATED'` for Supervisor role before manifest close.
- Enforced server-authoritative Supervisor RBAC (`wms_outbound.manage_orders` / `isSupervisorApproval`).
- Non-supervisors attempting carrier correction receive `403 Forbidden`.
- Status remains `LABEL_GENERATED` (via explicit self-transition `{ from: 'LABEL_GENERATED', to: 'LABEL_GENERATED', domainEvent: 'ShipmentCarrierOverridden' }`).
- Existing label is preserved without automatic regeneration or reprint (per P1 R35).

### 4. Explicit Hard Boundaries & Invariants
- **No External Carrier API / No Carrier Rejection (FR-P5-11):** Label generation is 100% internal WMS assembly. No external label broker or carrier acceptance/rejection endpoints exist.
- **Manifest Boundary (P1-015):** The pre-manifest carrier correction is enforced within the current persisted shipment state. Full `CarrierManifest.CLOSED` boundary is cleanly deferred to P1-015 without introducing parallel or fake manifest state.
- **ERP Boundary (P1-014):** No ERP posting or `POSTING_PENDING` / `POSTED` logic touched.
- **Settlement Boundary (P1-016):** No order/inventory settlement logic touched.
- **Scanner Boundary:** Untouched and frozen.

## Files Modified in Implementation

```text
 apps/mercato/src/modules/wms_outbound/__integration__/P1-013-label-generation-ui.spec.ts             | 470 ++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/generate-label/route.ts                    |  54 +++
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/print-label/route.ts                       |  45 ++
 apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx                               | 170 +++++++-
 apps/mercato/src/modules/wms_outbound/data/entities.ts                                              |  38 ++
 apps/mercato/src/modules/wms_outbound/data/transitions.ts                                           |   7 +
 apps/mercato/src/modules/wms_outbound/di.ts                                                         |   7 +
 apps/mercato/src/modules/wms_outbound/migrations/Migration20260904190000_wms_outbound_p1_013.ts     |  33 ++
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts        |   5 +-
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts | 480 ++++++++++++++++++++
 apps/mercato/src/modules/wms_outbound/services/carrier-selection-service.ts                         |  35 +-
 apps/mercato/src/modules/wms_outbound/services/shipment-label-service.ts                            | 205 +++++++++
 apps/mercato/src/modules/wms_outbound/services/shipment-service.ts                                  |  23 +-
 13 files changed, 1902 insertions(+), 11 deletions(-)
```

## Genuine PostgreSQL Verification (14/14 PASSED)

Executed against remote Testing PostgreSQL (`aws-1-eu-central-1.pooler.supabase.com:6543`, database `postgres`, PostgreSQL 17.6):

```text
PASS src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts (29.2s)
  P1-013 Genuine PostgreSQL WMS Label Generation & Pre-Manifest Carrier Correction Suite
    ✓ 1. CARRIER_SELECTED is the gate for label generation; advances to LABEL_GENERATED with WMS-owned payload (P1 R34, FR-P1-17) (942 ms)
    ✓ 2. Shipments in non-CARRIER_SELECTED status (e.g. CARRIER_PENDING) reject label generation (485 ms)
    ✓ 3. OWN_TRANSPORT skips/rejects label generation as designed (P1 R34, Step 11) (462 ms)
    ✓ 4. Generated label contains only WMS-owned persisted data (weight, volume, address, tracking reference) (487 ms)
    ✓ 5. Repeated generation is idempotent and does not duplicate state transitions or artifacts (504 ms)
    ✓ 6. Local print and reprint increments print count and updates timestamp without external API or state transition (FR-P1-18) (518 ms)
    ✓ 7. EXTERNAL missing-maxVolume path blocks label generation until Supervisor approves carrier (P1 R51, TC-066) (915 ms)
    ✓ 8. Supervisor can correct carrier when status is LABEL_GENERATED before manifest close (P1 R35) (964 ms)
    ✓ 9. Carrier correction does not automatically regenerate or reprint existing label (P1 R35) (928 ms)
    ✓ 10. Non-Supervisor carrier correction on LABEL_GENERATED fails closed with 403 Forbidden (483 ms)
    ✓ 11. Concurrency safety: overlapping label generation calls resolve idempotently and do not corrupt label records (1328 ms)
    ✓ 12. Transaction rollback: failure during label persistence rolls back status and leaves no partial label record (924 ms)
    ✓ 13. Regression: P1-012 carrier selection rules and Supervisor manual approval paths remain green (DEC-J11, DEC-J12, DEC-J13) (918 ms)
    ✓ 14. Regression: P1-011 shipment readiness, grouping and partial gates remain green (FR-P1-26) (886 ms)

Test Suites: 1 passed, 1 total
Tests:       14 passed, 14 total
Snapshots:   0 total
Time:        29.2 s
```

## Regression Verification

### P1-012 Carrier Selection PostgreSQL Suite (14/14 PASSED)

```text
PASS src/modules/wms_outbound/services/__tests__/p1-012-carrier-selection-postgres.integration.test.ts (26.8s)
Test Suites: 1 passed, 1 total
Tests:       14 passed, 14 total
Snapshots:   0 total
Time:        26.8 s
```

### P1-011 Shipment Grouping PostgreSQL Suite (18/18 PASSED)

```text
PASS src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts (71.2s)
Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        71.2 s
```

### State Transitions Invariant Suite (77/77 PASSED)

```text
PASS src/modules/wms_outbound/services/__tests__/fnd-002-state-transitions.test.ts (1.3s)
Test Suites: 1 passed, 1 total
Tests:       77 passed, 77 total
Snapshots:   0 total
Time:        1.3 s
```

## Fresh Final-Head Rendered UI Evidence (Playwright)

Executed using Playwright against canonical testing URL `https://devaxonic-test.info-start.com.pl` through real Chromium browser automation from the served final revision:

```text
Running 4 tests using 1 worker

  ✓  1 Journey 1 (PLAYWRIGHT VERIFIED): Normal label generation and print from CARRIER_SELECTED (P1 R34) (60.0s)
  ✓  2 Journey 2 (PLAYWRIGHT VERIFIED): Pending approval gate blocks label generation until Supervisor approval (P1 R51, TC-066) (14.8s)
  ✓  3 Journey 3 (PLAYWRIGHT VERIFIED): Supervisor carrier correction after label generation before manifest close (P1 R35) (14.4s)
  ✓  4 Journey 4 (PLAYWRIGHT VERIFIED): Architecture boundary: no provider rejection or external API mode (FR-P5-11) (12.5s)

  4 passed (1.9m)
```

*(Note: Playwright automated test output; not claimed as HUMAN VERIFIED per project standards).*

## Explicit Exclusions & Deferred Scope

- External carrier Label API: Not implemented / rejected per Architect specification.
- Carrier acceptance/rejection mode: Not implemented / rejected per FR-P5-11.
- Invented `LABEL_ERROR` state machine: Not introduced.
- ERP POST / `POSTING_PENDING` / `POSTED` (P1-014): Deferred to authorized P1-014.
- `CarrierManifest` lifecycle & post-CLOSED boundary (P1-015): Deferred to authorized P1-015.
- Order settlement & final inventory deduction (P1-016): Deferred to authorized P1-016.
- Scanner application: Untouched and frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- Production / Demo environments: Untouched.
