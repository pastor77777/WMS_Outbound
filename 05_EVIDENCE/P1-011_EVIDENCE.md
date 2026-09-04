# P1-011 Evidence — Final Closeout

**Task:** P1-011 — Shipment grouping, readiness closure and partial gate
**Evidence date:** 2026-09-04
**Evidence class:** REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED

## Final revisions and runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato final head | `20887f2d74928cf69f447fdd6af20a612f38387c` on `outbound/p1-011` |
| Accepted P1-010 base | `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643` |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:5432`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `http://localhost:3009/login` returned HTTP 200 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Final authorized UI closeout

The final product/UI corrections are limited to the canonical Shipment detail route behavior:

- Shipment list number links and row actions navigate to `/backend/shipments/<shipmentId>`.
- The Shipment detail page consumes the backend manifest-provided `params.id`; the catch-all router supplies this value. A missing ID now leaves loading cleanly with an error rather than loading indefinitely.

Journey B exercised the rendered canonical route, not a direct detail-page navigation: Shipment list → search → Shipment-number link → `/backend/shipments/<id>` → rendered `shipment-status-badge` showing `READY_FOR_DISPATCH`.

## Decisive real PostgreSQL evidence

P1-011 executed on the approved Testing PostgreSQL with real concurrent connections and transactions.

```text
[P1-011 distinct-TU grouping-key lock contention] {
  blockedPid: 2000125,
  blockingPids: [ 2000121, 2000124 ],
  waitEventType: 'Lock'
}

✓ 16. Real rollback: force a real failure after at least one write/flush within grouping transaction and prove through fresh independent read that no partial membership survives

Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        12.028 s
```

The lock evidence is a real PostgreSQL wait (`Lock`) for the distinct-TU grouping-key contention scenario. The rollback case writes and flushes in the real grouping transaction, forces failure, then confirms from an independent read that no partial membership remains.

## Fresh final-head backend output

```text
P1-011 PostgreSQL
PASS p1-011-postgres.integration.test.ts (11.638 s)
Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Time:        12.028 s

P1-010 PostgreSQL regression
PASS p1-010-postgres.integration.test.ts (6.289 s)
Test Suites: 1 passed, 1 total
Tests:       16 passed, 16 total
Time:        6.643 s

P1-009 PostgreSQL regression
PASS p1-009-postgres.integration.test.ts (10.382 s)
Test Suites: 1 passed, 1 total
Tests:       15 passed, 15 total
Time:        10.806 s

P1-003 PostgreSQL regression
PASS p1-003-postgres.integration.test.ts (8.435 s)
Test Suites: 1 passed, 1 total
Tests:       14 passed, 14 total
Time:        8.842 s

Full wms_outbound umbrella
Test Suites: 20 passed, 20 total
Tests:       294 passed, 294 total
Snapshots:   0 total
Time:        78.93 s
Ran all test suites matching src/modules/wms_outbound.
```

## Fresh final-head rendered UI output

```text
P1-011 Playwright — http://localhost:3009
✓ Journey A (PLAYWRIGHT VERIFIED): Single-shipment wait and unblock under allowPartialShipment=false (P1 R57, R58, P5 E17) (17.4s)
✓ Journey B (PLAYWRIGHT VERIFIED): Dual partial-allowed grouping and Shipment readiness promotion (P1 R26, R27, R28) (16.8s)
✓ Journey C (PLAYWRIGHT VERIFIED): Late TU ready after closure routes to a second Shipment without regressing closed Shipment (P1 R28, R29) (16.6s)
3 passed (3.4m)

P1-010 Playwright regression — http://localhost:3009
✓ Journey 1: Keep same TU — operator selects TU, reviews suggestion, confirms pack as-is, TU seals as PackUnit (8.1s)
✓ Journey 2: Repack All — operator transfers entire contents to a new shipping box, source TU marks REPACKED (7.9s)
✓ Journey 3: Repack by SKU / Defer / Missing — operator defers count with 0 shortage, then reports confirmed missing (8.4s)
✓ Journey 4: Damaged Stock & Unexpected SKU routed to QC (8.3s)
✓ Journey 5: Supervisor Authorization for Suggestion Deviation (15.4s)
✓ Journey 6: Rendered Compatible & Incompatible Packing Consolidation (8.3s)
6 passed (3.4m)
```

The UI evidence is **PLAYWRIGHT VERIFIED**. It is not labeled Human Verified.

## P1-011 acceptance coverage proved

| Journey / proof | Requirements and observable result |
|---|---|
| Journey A | R57/R58: normal Packer Workstation actions show the no-partial block, then rendered Shipment UI grouping releases both sealed TUs into the same Shipment. |
| Journey B | R26/R27/R28: exact TU-specific assignment feedback, shared Shipment, `READY_FOR_DISPATCH`, and rendered canonical list-link-to-detail status proof. |
| Journey C | R28/R29: late TU reaches a second Shipment without regressing the closed first Shipment. |
| PostgreSQL lock | Distinct-TU grouping-key contention produced a real PostgreSQL `Lock` wait. |
| PostgreSQL rollback | Forced post-flush transaction failure left no partial Shipment membership. |

No STATE, handover, Scanner, architecture, credential-policy, migration, or backend Shipment business-rule change is included in this closeout evidence.
