# P1-012 Evidence — Remediation & Verification Closeout

**Task:** P1-012 — Carrier, Region and CarrierSetup selection (Step 10, P1 R31–R34, DEC-J11, DEC-J12, DEC-J13, FR-P1-16, FR-P5-10)  
**Remediation guide:** `06_AGENT_GUIDES/P1-012_REMEDIATION.md`  
**Evidence date:** 2026-09-04  
**Evidence class:** REAL PostgreSQL integration / REAL PostgreSQL concurrency / PLAYWRIGHT VERIFIED  

## Final Revisions and Runtime

| Subject | Verified revision / identity |
|---|---|
| Mercato final remediation head | `5019a20be14549ff8cbbf25af5bc61c56888e9e1` on `outbound/p1-012` (pushed) |
| Pre-remediation P1-012 head | `7273fde8d47686812d099f99a6cdcfa323045826` on `outbound/p1-012` |
| Accepted P1-011 base | `20887f2d74928cf69f447fdd6af20a612f38387c` |
| Lineage / compare | Exactly 2 commits ahead of accepted P1-011 base (`20887f2d7 -> 7273fde8d -> 5019a20be`); merge base: `20887f2d74928cf69f447fdd6af20a612f38387c` |
| Frozen Scanner head | `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` (verified untouched, clean working tree) |
| Testing database | `postgres` on `aws-1-eu-central-1.pooler.supabase.com:6543`; PostgreSQL 17.6 |
| Testing Mercato runtime | `mercato-localhost.service` active; `https://devaxonic-test.info-start.com.pl` returned HTTP 200 |

All database-backed commands sourced `/etc/mercato-localhost.env` in their executing shell. No credential values are recorded here.

## Remediation Scope & Review Blockers Resolved

### B1 — Server-Authoritative Supervisor / Dispatcher Authority & Anti-Spoofing

- **Root cause:** Initial implementation accepted `actorRole` and `isSupervisorApproval` from the client request body, allowing potential client-side role spoofing.
- **Architectural correction:**
  - Request body schema in `apps/mercato/src/modules/wms_outbound/api/shipments/[id]/manual-carrier/route.ts` stripped of `actorRole` and `isSupervisorApproval`.
  - Caller identity resolved strictly from authenticated session context (`auth.userId || auth.sub`).
  - Authority resolution delegates to PostgreSQL role/feature mapping:
    - `checkSupervisorAuthority(txEm, scope, actorId)` verifies feature `wms_outbound.manage_orders` in tenant/organization scope.
    - `checkDispatcherAuthority(txEm, scope, actorId)` verifies feature `wms_outbound.edit` in tenant/organization scope.
  - Fail-closed enforcement:
    - Non-supervisors attempting manual selection from general no-match `CARRIER_PENDING` receive `403 Forbidden`.
    - Non-supervisors attempting to override `CARRIER_SELECTED` receive `403 Forbidden`.
    - On EXTERNAL TU missing `maxVolume`:
      - Dispatchers can submit carrier selection, which persists carrier and keeps status `CARRIER_PENDING` with audit `actorRole: 'Dispatcher'`.
      - Only a verified Warehouse Supervisor can approve and advance to `CARRIER_SELECTED` with `manualApprovedBy: supervisorId` and audit `actorRole: 'Warehouse Supervisor'`.
      - Non-supervisors attempting approval fail with `403 Forbidden`.
  - Audit logging records real server-determined role and authenticated caller identity.

### B2 — Exact Mercato Commit SHA in Evidence

- Pre-remediation P1-012 head: `7273fde8d47686812d099f99a6cdcfa323045826`.
- Final remediation head pushed to `origin/outbound/p1-012`: `5019a20be14549ff8cbbf25af5bc61c56888e9e1`.
- The previously miscopied commit SHA has been removed and replaced with the verified Git SHA.

### B3 — Credential Hygiene in Test Suite

- Removed hardcoded test password `'Devax7adm#'` from `apps/mercato/src/modules/wms_outbound/__integration__/P1-012-carrier-selection-ui.spec.ts`.
- Replaced with `process.env.TEST_SUPERVISOR_PASSWORD || ...` with fail-closed validation if not set.
- Diff verification confirms 0 literal passwords or credentials in the P1-012 changeset.

## Files Modified in Remediation

```text
 apps/mercato/src/modules/wms_outbound/__integration__/P1-012-carrier-selection-ui.spec.ts            |  12 +-
 apps/mercato/src/modules/wms_outbound/api/shipments/[id]/manual-carrier/route.ts                     |  17 +-
 apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx                               |  10 +-
 apps/mercato/src/modules/wms_outbound/services/__tests__/p1-012-carrier-selection-postgres.integration.test.ts | 275 ++++++++++++++++++++-
 apps/mercato/src/modules/wms_outbound/services/carrier-selection-service.ts                          | 125 ++++++++--
 apps/mercato/src/modules/wms_outbound/services/packing-service.ts                                   |  34 +++
 6 files changed, 426 insertions(+), 47 deletions(-)
```

## Decisive Real PostgreSQL Evidence

Executed against remote PostgreSQL pooler (`aws-1-eu-central-1.pooler.supabase.com:6543`) with genuine database transactions, pessimistic row locking (`SELECT ... FOR UPDATE`), and unique database constraints:

```text
Test Suites: 1 passed, 1 total
Tests:       14 passed, 14 total
Snapshots:   0 total
Time:        33.821 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/p1-012-carrier-selection-postgres.integration.test.ts.
```

### Test Suite Breakdown:

1. `Start gate: not-ready Shipment cannot run/commit Carrier Selection (P1 Step 10)` — PASS
2. `Single match: Region + governing weight + governing volume selects sole applicable Carrier and persists CARRIER_SELECTED (P1 R31, DEC-J11)` — PASS
3. `Volume tie-break: multiple candidates -> narrowest matching volume range wins (P1 R32, DEC-J12)` — PASS
4. `Weight tie-break: equal volume specificity -> narrowest matching weight range wins (P1 R32, DEC-J12)` — PASS
5. `Priority tie-break: equal range specificity -> unique Carrier priority wins (P1 R32, DEC-J12)` — PASS
6. `Determinism: repeated evaluation with unchanged persisted inputs/config yields same Carrier and no duplicate transition effects (P1 R31)` — PASS
7. `No match: persists CARRIER_PENDING with useful reason and manual path (P1 R31, R51, FR-P5-10)` — PASS
8. `Manual selection after no-match: server-authoritative supervisor enforcement and anti-spoofing (P1 R33, DEC-J11, DEC-J13, FR-P1-17)` — PASS:
   - Proves authenticated non-Supervisor with `wms_outbound.edit` is rejected (403 Forbidden).
   - Proves body-spoofed `actorRole: 'Warehouse Supervisor'` and `isSupervisorApproval: true` are ignored/rejected.
   - Proves verified Warehouse Supervisor successfully advances status to `CARRIER_SELECTED`.
   - Proves audit trail records authentic `actorRole: 'Warehouse Supervisor'`.
9. `Override: Supervisor authority enforcement, anti-spoofing, and optional reason (P1 R33, DEC-J13)` — PASS:
   - Proves non-Supervisor cannot override existing selection even with spoofed body flags (403 Forbidden).
   - Proves verified Warehouse Supervisor can override with or without reason.
   - Proves transition event `ShipmentCarrierOverridden` is properly recorded.
10. `EXTERNAL missing maxVolume: Dispatcher choice remains CARRIER_PENDING, anti-spoofing, and Supervisor approval (P1 R51, R68, FR-P5-10, TC-066)` — PASS:
    - Proves Dispatcher choice sets carrier while preserving `CARRIER_PENDING`.
    - Proves non-Supervisor cannot approve missing-volume shipment (403 Forbidden).
    - Proves verified Supervisor approval transitions shipment to `CARRIER_SELECTED` with `manualApprovedBy`.
11. `Configuration validation: duplicate priority in carrier dictionary is rejected by PostgreSQL unique constraint (DEC-J11, DEC-J12)` — PASS
12. `Concurrent/replay safety: overlapping selection/override attempts do not corrupt persisted selection or duplicate irreversible state effects` — PASS
13. `Real rollback test: failure after write/flush in transaction rolls back completely with fresh independent read proof` — PASS
14. `P1-011 and ATP regression: Shipment grouping, readiness and ATP reservation primitives remain intact (FR-P1-26)` — PASS

## Regression Verification

### P1-011 PostgreSQL Regression Suite

```text
  [P1-011 distinct-TU grouping-key lock contention] {
    blockedPid: 2024132,
    blockingPids: [ 2024133 ],
    waitEventType: 'Lock'
  }

Test Suites: 1 passed, 1 total
Tests:       18 passed, 18 total
Snapshots:   0 total
Time:        70.5 s, estimated 72 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/p1-011-postgres.integration.test.ts.
```

### State Transitions Invariant Suite

```text
Test Suites: 1 passed, 1 total
Tests:       77 passed, 77 total
Snapshots:   0 total
Time:        1.272 s, estimated 2 s
Ran all test suites matching src/modules/wms_outbound/services/__tests__/fnd-002-state-transitions.test.ts.
```

## Fresh Final-Head Rendered UI Evidence (Playwright)

Executed using Playwright against canonical testing URL `https://devaxonic-test.info-start.com.pl` through real Chromium browser automation from the served remediation revision:

```text
Running 5 tests using 1 worker

  ✓  1 Journey 1 (PLAYWRIGHT VERIFIED): Automatic selection happy path (P1 R31, R32, DEC-J11, DEC-J12) (11.7s)
  ✓  2 Journey 2 (PLAYWRIGHT VERIFIED): Deterministic tie-break display & winner (DEC-J12, P1 R32) (8.1s)
  ✓  3 Journey 3 (PLAYWRIGHT VERIFIED): Manual selection fallback when no CarrierSetup matches (P1 R33, DEC-J11) (8.8s)
  ✓  4 Journey 4 (PLAYWRIGHT VERIFIED): Supervisor manual override without mandatory reason (P1 R33) (8.1s)
  ✓  5 Journey 5 (PLAYWRIGHT VERIFIED): Missing maxVolume warning and supervisor approval flow (P1 R51, R68, FR-P5-10) (8.8s)

  5 passed (48.0s)
```

*(Note: Playwright automated test output; not claimed as HUMAN VERIFIED per project standards).*

## Scanner Status Confirmation

- Path: `/home/ubuntu/git/Devaxonic-scanner`
- Verified HEAD: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- Working tree: Clean (0 modifications, completely frozen).

## Clean Git Status and Explicit Exclusions

- `Devaxonic-mercato`: `outbound/p1-012` is clean and pushed to `origin/outbound/p1-012` (`5019a20be14549ff8cbbf25af5bc61c56888e9e1`). Not merged to `main`.
- `Devaxonic-scanner`: clean and frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- Hard exclusions maintained:
  - NO label generation / printing.
  - NO external carrier API calls.
  - NO manifest work.
  - NO ERP dispatching.
  - NO Scanner changes.
  - NO `STATE.md`, roadmap, or task catalog updates.
  - NO claim of FINAL PASS.
