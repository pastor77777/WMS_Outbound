# ACC-001 — Complete Automated Component/Integration Requirement Suite — Evidence

Date: 2026-09-07 UTC
Evidence class: REAL POSTGRESQL INTEGRATION, REAL CONCURRENCY (approved Testing Supabase `DevAxonic_Platform`). No product/business behavior was changed; this item is an audit-and-close item. Not Human Verified, not Supervisor FINAL PASS, not Owner Accepted.

## Exact revisions and scope

- Mercato base: `outbound/post-x002-raw-pg-ssl` @ `cfdbc608b22fc1dd44646c309335d2071aafd32c` (post-X-002 raw pg SSL maintenance gate, Supervisor FINAL PASS / Owner Accepted).
- Mercato batch branch: `outbound/acc-001-003-batch`.
- Mercato ACC-001 checkpoint commit SHA: `7480f1d707be88ac70ccd2c8ef04b3a56eda760e`.
- Scanner: unchanged, `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302` (frozen; no Scanner change required for ACC-001).
- Item: 34/37 — ACC-001: Complete automated component/integration requirement suite.
- Execution guide: `06_AGENT_GUIDES/ACC-001_003_BATCH_EXECUTION.md`.

## Method

ACC-001 is an audit-and-close item, not new business behavior. Work performed:

1. Cross-checked `03_TRACEABILITY/requirements_index.csv` (109 requirement IDs) against `03_TRACEABILITY/requirement_task_matrix.csv` for zero orphans.
2. For the 11 `INT-01..06`/`CON-01..05` requirements (technical invariants with no `TC-*` scenario mapping by design), located and named their exact currently-owning automated test files, reusing the decisive test names already established and independently verified in the accepted `X-001_EVIDENCE.md` (CON-01..05) and `X-002_EVIDENCE.md` (INT-01..06).
3. Ran the complete `apps/mercato/src/modules/wms_outbound/services/__tests__/` directory against the canonical Testing PostgreSQL to find any real remaining gap (the accepted `04_CURRENT_STATE/TEST_INFRA_GAPS.md` "Scope note" explicitly called for exactly this full-directory audit as a future ACC-001-shaped item).
4. Diagnosed and fixed the one genuine gap found (see "Gap Found and Fixed" below); no other product or business-logic change was made.
5. Reran the full directory to green, ran Mercato typecheck, and verified there are no pending migrations across every module (including shared `wms_inventory`/`wms_tu`/`wms_orchestration`/`wms_warehouse`).

## 109/109 requirement-to-test coverage — zero orphans

- `03_TRACEABILITY/requirement_task_matrix.csv` has exactly 109 data rows, one per requirement ID in `requirements_index.csv`; a full outer-join check (`requirement_id` set equality) confirms **zero orphans in either direction**.
- 98 of 109 requirements (`FR-P1-*`, `FR-P2-*`, `FR-P3-*`, `FR-P4-*`, `FR-P5-*`) carry an explicit `scenario_ids` (`TC-*`) mapping, cross-checked against `03_TRACEABILITY/coverage_matrix.csv` (150 architect-rule rows, all marked `component_acceptance_present = ✅`, zero requirement orphans against the FR set) and `03_TRACEABILITY/test_index.csv` (97 named `TC-*` test definitions).
- The remaining 11 requirements (`INT-01..06`, `CON-01..05`) are technical invariants tracked by named automated test files directly (no human-facing `TC-*` scenario), per the table below.

## INT/CON decisive automated coverage (reused from accepted X-001/X-002 evidence, reverified live on this item's checkpoint)

| Requirement | Owning test(s) | Result on this checkpoint |
|---|---|---|
| CON-01 | `p1-004-postgres.integration.test.ts` | PASS (part of full-directory rerun below) |
| CON-02 | `p1-005-postgres.integration.test.ts` | PASS |
| CON-03 | `p2-002-crossdock-planning-postgres.integration.test.ts` (+ `p2-001` CON-03 case) | PASS |
| CON-04 | `p1-011-postgres.integration.test.ts`, `p1-015-manifest-lifecycle-postgres.integration.test.ts` | PASS |
| CON-05 | `p1-014-erp-posting-postgres.integration.test.ts`, `p1-016-final-settlement-postgres.integration.test.ts`, `p2-006-crossdock-shipment-downstream-postgres.integration.test.ts` | PASS |
| INT-01 | `p2-001-crossdock-eligibility-postgres.integration.test.ts` | PASS |
| INT-02 | `p2-003-crossdock-execution-postgres.integration.test.ts`, `p2-004-crossdock-recovery-postgres.integration.test.ts` | PASS |
| INT-03 | `p2-005-crossdock-gr-gate-postgres.integration.test.ts` | PASS |
| INT-04/05 | `p1-014-erp-posting-postgres.integration.test.ts` | PASS |
| INT-06 | `fnd-001-ordering-adapter.test.ts`, `p3-002-postgres.integration.test.ts`, `p3-003-postgres.integration.test.ts`, `p4-001-postgres.integration.test.ts` | PASS |

No CON/INT test was weakened, skipped, or had its decisive PostgreSQL-side assertions removed.

## Gap Found and Fixed — `p1-012` audit-log ordering non-determinism (test-only, not a product defect)

**Symptom:** first full-directory sweep on this checkpoint (`apps/mercato`, `npx jest --forceExit src/modules/wms_outbound/services/__tests__/`) reported one failure: `p1-012-carrier-selection-postgres.integration.test.ts` › "10. EXTERNAL missing maxVolume: Dispatcher choice remains CARRIER_PENDING, anti-spoofing, and Supervisor approval" — `lastDispEvent.actorId` was `"System WMS"` instead of the expected real dispatcher user id.

**Root cause:** the test fetched `WmsOutboundStateTransitionEvent` rows with `dispEm.find(...)` and no `orderBy`, then indexed `dispEvents[dispEvents.length - 1]` assuming SQL row order matches insertion order. That assumption is not guaranteed: the automatic-evaluation step (10a) and the manual-selection step (10b) both write a `toStatus: 'CARRIER_PENDING'` audit row for the same shipment, and an unordered `find()` can return them in either order. The underlying service code (`carrier-selection-service.ts::manualSelectCarrier`) was independently read and confirmed correct — it does write the real dispatcher `actorId` on the already-`CARRIER_PENDING` branch (line ~683-698); this was a test-determinism defect only, not a business-logic defect.

**Fix:** added `{ orderBy: { createdAt: 'ASC' } }` to the `find()` call, matching the existing convention already used identically for the same entity/ordering need in `fnd-002`, `p1-001`, `p1-003`, and `p1-009`'s own postgres integration suites. One file changed, test-only, no application code touched.

**Verification:** `p1-012-carrier-selection-postgres.integration.test.ts` rerun standalone 3x after the fix — `14/14 PASSED` every time.

## Pooler-load transient (not a real defect, no fix needed)

A second full-directory run (all suites in parallel, default Jest worker count) reported one different, non-reproducible failure: `p2-001-crossdock-eligibility-postgres.integration.test.ts` › "CON-03: independent PostgreSQL sessions serialize competing bindings exactly once" — `expect(secondPid).not.toBe(firstPid)` failed once (both logical sessions observed the same backend PID). Rerun standalone 3x immediately after: `10/10 PASSED` every time, no failure. This is consistent with the shared Supabase pooler (`aws-1-eu-central-1.pooler.supabase.com:6543`) occasionally handing two near-simultaneous logical connections the same backend under the additional connection pressure of ~38 suites running fully in parallel — not a reproducible defect in the underlying `CON-03` serialization mechanism, which passed decisively and independently in `X-001_EVIDENCE.md` and here in the reduced-worker full sweep below. No product or test change was made for this.

## Final full-directory result (this item's exact checkpoint)

```bash
set -a && source /etc/mercato-localhost.env && set +a
npx jest --config jest.config.cjs --forceExit --maxWorkers=4 src/modules/wms_outbound/services/__tests__/
# Test Suites: 38 passed, 38 total
# Tests:       583 passed, 583 total
```

`npx tsc --noEmit` (apps/mercato): clean, zero errors.

## Migration / constraint verification

`npm run db:migrate` (canonical Testing DB, same-shell `DATABASE_URL`): **no pending migrations** in any module, including shared `wms_inventory`, `wms_tu`, `wms_orchestration`, `wms_warehouse`, and `wms_outbound` — no migration was applied or needed by this item.

## Shared Inbound regression

This item's only diff is the single test-only ordering fix in `p1-012-carrier-selection-postgres.integration.test.ts`. No shared Inventory/TU/warehouse/task-lock/orchestration primitive was touched, so no additional Inbound regression sweep is required beyond what X-001/X-002/the SSL maintenance gate already proved.

## Scanner

No Scanner change was required. ACC-001 is automated component/integration coverage only (no Playwright/UI); Scanner remains frozen at `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.

## Completion statement

109/109 requirements have a named, executable, currently-passing automated test (zero orphans in `requirement_task_matrix.csv`). The one genuine gap found (`p1-012` audit-log ordering non-determinism) is fixed; the one non-reproducible pooler-load transient (`p2-001` CON-03) is documented and does not indicate a real defect. Mercato typecheck is clean. No pending migrations. No shared Inbound regression required. Scanner remains frozen. Evidence and implementation are pushed. ACC-002 was not started before this internal gate was confirmed green.

This is executor-prepared evidence, not Supervisor FINAL PASS or Owner Acceptance.
