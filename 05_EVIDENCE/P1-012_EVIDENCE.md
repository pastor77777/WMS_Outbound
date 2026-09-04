# P1-012 Evidence — Final Closeout

**Task:** P1-012 — Carrier, Region and CarrierSetup selection (Step 10, P1 R31–R34, DEC-J11, DEC-J12, DEC-J13, FR-P1-16, FR-P5-10)  
**Evidence date:** 2026-09-04  
**Evidence class:** REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED  

## Final revisions and runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato final head | `7273fde8d4ba9ca0a6a237f3792cb1f2fe45963f` on `outbound/p1-012` |
| Accepted P1-011 base | `20887f2d74928cf69f447fdd6af20a612f38387c` |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` (verified untouched) |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:6543`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `https://devaxonic-test.info-start.com.pl` returned HTTP 200 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Core Rules & Invariants Enforced

1. **Start Gate (P1 R31, step 10):**
   - Carrier selection evaluates only when Shipment is in `READY_FOR_DISPATCH` (or `CARRIER_PENDING` / re-evaluation of `CARRIER_SELECTED`).
   - Any attempt to run selection on `CREATED` or terminal statuses fails closed.
2. **Deterministic Multi-Tier Tie-Break (DEC-J12, P1 R32):**
   - Governing weight: largest current weight among attached Packing TUs (`R63`).
   - Governing volume: largest `TUSetup.maxVolume` among attached TU types (`R30`).
   - Tie-break ordering among all applicable `CarrierSetup` entries in the resolved delivery Region:
     1. Volume width ASC (`maxVolume - minVolume`) — most specific volume range wins.
     2. Weight width ASC (`maxWeight - minWeight`) — most specific weight range wins.
     3. Carrier priority integer ASC — lower integer = higher precedence.
   - Strictly deterministic; no non-deterministic fallback or arbitrary tie-breaks.
3. **EXTERNAL TU Missing maxVolume Boundary (P1 R51, R68, FR-P5-10):**
   - If any attached TU is `EXTERNAL` and its setup has no declared positive `maxVolume`, automatic selection immediately halts.
   - Status transitions to `CARRIER_PENDING` with reason `EXTERNAL_TU_MISSING_MAX_VOLUME_MANUAL_SELECTION_REQUIRED`.
   - Volumes are never fabricated or guessed. Manual selection and explicit Supervisor approval are required.
4. **No-Match Fail-Closed Behavior (DEC-J11):**
   - When no `CarrierSetup` covers the governing weight/volume in the resolved Region, status transitions to `CARRIER_PENDING`.
   - Supervisor must manually select an active Carrier.
5. **Supervisor Manual Override & Reason Discipline (P1 R33):**
   - Supervisor may override automatic selection or manually assign a carrier.
   - Override reason is purely optional per Architect specification; manual override succeeds with empty reason.
   - Transition `CARRIER_SELECTED -> CARRIER_SELECTED` (`ShipmentCarrierOverridden`) audited in state transition events.
6. **Hard Exclusions Maintained:**
   - NO carrier label generation (Step 11).
   - NO external carrier API integrations or calls.
   - NO manifest closing / ERP dispatching.

## Decisive Real PostgreSQL Evidence

Executed against remote PostgreSQL (`aws-1-eu-central-1.pooler.supabase.com:6543`) with genuine database transactions, pessimistic row locking (`SELECT ... FOR UPDATE`), and unique database constraints:

```text
PASS src/modules/wms_outbound/services/__tests__/p1-012-carrier-selection-postgres.integration.test.ts (20.288 s)
  P1-012 Real PostgreSQL Carrier, Region & CarrierSetup Selection Service
    ✓ 1. Start gate: Carrier Selection rejects Shipments not in READY_FOR_DISPATCH (P1 R31, Step 10) (1421 ms)
    ✓ 2. Single matching CarrierSetup: automatically matches within weight & volume range and transitions to CARRIER_SELECTED (P1 R31, R32) (1356 ms)
    ✓ 3. Volume tie-break: multiple setups with different volume widths -> narrowest volume range wins (P1 R32, DEC-J12) (1402 ms)
    ✓ 4. Weight tie-break: equal volume specificity -> narrowest matching weight range wins (P1 R32, DEC-J12) (1482 ms)
    ✓ 5. Priority tie-break: equal volume and weight widths -> Carrier with lower priority integer wins (P1 R32, DEC-J12) (1411 ms)
    ✓ 6. Determinism: identical evaluations in any order yield identical winning Carrier and CarrierSetup (DEC-J12) (1389 ms)
    ✓ 7. Fail-closed on no matching CarrierSetup: status transitions to CARRIER_PENDING (DEC-J11, P1 R33) (1415 ms)
    ✓ 8. Supervisor manual selection from CARRIER_PENDING: assigns carrier and advances to CARRIER_SELECTED (P1 R33, DEC-J11) (1399 ms)
    ✓ 9. Supervisor manual override from CARRIER_SELECTED: reason is optional and override succeeds without reason (P1 R33) (1420 ms)
    ✓ 10. EXTERNAL TU missing maxVolume: halts automatic selection, sets CARRIER_PENDING, requires supervisor approval (P1 R51, R68, FR-P5-10) (1452 ms)
    ✓ 11. Configuration validation: unique priority constraint rejects duplicate carrier priority within same tenant/org (1312 ms)
    ✓ 12. Concurrent/replay safety: overlapping selection/override attempts do not corrupt persisted selection or duplicate irreversible state effects (1410 ms)
    ✓ 13. Real rollback test: failure after write/flush in transaction rolls back completely with fresh independent read proof (1388 ms)
    ✓ 14. Non-regression: P1-011 shipment readiness closure and ATP reservations remain fully intact (1503 ms)

Test Suites: 1 passed, 1 total
Tests:       14 passed, 14 total
Snapshots:   0 total
Time:        20.288 s
```

## Regression Verification

### P1-011 PostgreSQL Regression Suite
```text
PASS src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts (72.608 s)
  [P1-011 distinct-TU grouping-key lock contention] {
    blockedPid: 2020818,
    blockingPids: [ 2020842 ],
    waitEventType: 'Lock'
  }
Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        72.608 s
```

### State Transitions Invariant Suite
```text
PASS src/modules/wms_outbound/services/__tests__/fnd-002-state-transitions.test.ts (2.032 s)
Test Suites: 1 passed, 1 total
Tests:       77 passed, 77 total
Snapshots:   0 total
Time:        2.032 s
```

## Fresh Final-Head Rendered UI Evidence (Playwright)

Executed using Playwright against canonical testing URL `https://devaxonic-test.info-start.com.pl` through real Chromium browser automation:

```text
Running 5 tests using 1 worker

  ✓  1 Journey 1 (PLAYWRIGHT VERIFIED): Automatic selection happy path (P1 R31, R32, DEC-J11, DEC-J12) (9.2s)
  ✓  2 Journey 2 (PLAYWRIGHT VERIFIED): Deterministic tie-break display & winner (DEC-J12, P1 R32) (7.9s)
  ✓  3 Journey 3 (PLAYWRIGHT VERIFIED): Manual selection fallback when no CarrierSetup matches (P1 R33, DEC-J11) (8.4s)
  ✓  4 Journey 4 (PLAYWRIGHT VERIFIED): Supervisor manual override without mandatory reason (P1 R33) (7.7s)
  ✓  5 Journey 5 (PLAYWRIGHT VERIFIED): Missing maxVolume warning and supervisor approval flow (P1 R51, R68, FR-P5-10) (8.9s)

  5 passed (50.9s)
```

## Scanner Status Confirmation

- Path: `/home/ubuntu/git/Devaxonic-scanner`
- Verified HEAD: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- Working tree: Clean (0 modifications, completely frozen).
