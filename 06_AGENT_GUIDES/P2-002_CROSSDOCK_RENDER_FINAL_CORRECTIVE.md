# P2-002 — final Crossdock rendered-flow corrective

## Scope
Continue P2-002 only in the SAME Codex session. Do not start P2-003. Do not touch STATE, handovers, Architect Source, Canon, Task Catalog, or acceptance state.

Frozen green backend state:
- Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93` — clean and MUST remain unchanged.
- Scanner entering SHA `c5f65189c832ebdbd312ae0f5fc6930c74d43da0` — clean.
- P1-005 current-UI Scanner regression: 1/1 PASS.
- corrected CON-03: PASS.
- P2-001: 10/10 PASS.
- P2-002 dedicated PostgreSQL: 22/22 PASS.
- remaining mandatory Mercato regressions: 5 suites / 41 tests PASS.
- canonical PostgreSQL Testing only: Supabase project `yzonugcenguvmojwiihb`. Local PostgreSQL is forbidden.

This corrective exists because the dedicated P2-002 Playwright reached the rendered Crossdock module but did not observe `crossdock-task-number`.

## 1. Ground the exact current code before editing
Read only:
- Scanner `src/screens/CrossdockTaskScreen.js`
- Scanner `src/components/StatusBadge.js`
- Scanner `src/screens/LoginScreen.js`
- Scanner `App.js` warehouse-resolution path
- Scanner `e2e/p2-002-crossdock-assignment.spec.ts`
- Mercato `/api/wms_outbound/cross-dock/request-task` route at the frozen Mercato SHA.

Preserve these verified conclusions unless repo facts differ:
1. `CrossdockTaskScreen` already exposes `testID="crossdock-task-number"` and displays `task.taskNumber || task.id`.
2. `requestCrossdockTask()` already returns `result.task || null`.
3. `LoginScreen` invokes async `onAuthed(...)` without awaiting its warehouse-resolution work; therefore `Choose mode` can render before the active warehouse state is fully resolved.
4. The dedicated spec currently clicks Crossdock immediately after seeing `Choose mode`; this can race warehouse resolution.
5. `CrossdockTaskScreen` currently passes `status={task.status}` to `StatusBadge`, while `StatusBadge` accepts `code`; this is a real one-line rendered-status defect.

Do not alter business assignment semantics, planner ordering, database schema, migrations, Mercato, auth logic, or warehouse service.

## 2. Authorized minimal Scanner correction
Only these Scanner changes are authorized.

### A. Product UI — fix StatusBadge prop
In `src/screens/CrossdockTaskScreen.js`, change only the incorrect status-badge prop so the assigned status is visibly rendered through the current StatusBadge contract:
- from `status={task.status}`
- to `code={task.status}`

No other Crossdock product behavior change.

### B. Dedicated P2-002 Playwright — remove login/warehouse race and make failure evidence decisive
Update only `e2e/p2-002-crossdock-assignment.spec.ts` as needed.

For each login helper/session:
1. sign in normally through rendered Scanner UI;
2. wait for the real `/api/wms_warehouse/my-warehouses` resolution to complete successfully OR wait for the rendered active warehouse context to contain the fixture warehouse name/identity;
3. do not click Crossdock merely because `Choose mode` is visible;
4. after warehouse context is resolved, arm a `page.waitForResponse(...)` for the real `POST /api/wms_outbound/cross-dock/request-task` BEFORE clicking Crossdock;
5. click the rendered Crossdock mode tile;
6. await the real response, require HTTP 200, parse JSON, and record/sanity-check its `task` or `error/message` result;
7. if response contains a task, require its `taskNumber` to equal the deterministic fixture expected for that operator before asserting the rendered element;
8. then assert `crossdock-task-number`, source TU, planned quantity, no-zone text, and P2-003 boundary text in the rendered UI;
9. if response is `task:null` or error, fail with that exact API classification instead of generic "element not exposed";
10. zero route mocks remain mandatory.

Do not increase arbitrary sleeps. Use network/UI readiness signals.

For operator A/B, preserve actual assignment ordering and exactly-once checks. For blocked operator, await the real request-task response and assert the real active-task error plus rendered `crossdock-error`.

Correct the DB assertions at the end if needed so they assert actual counts/rows, not merely that a query result object is truthy. In particular, if asserting no Allocation creation for this P2-002 fixture, scope the assertion to fixture-related rows/identifiers or use a before/after count; never assert the entire shared Testing database has zero allocations.

## 3. First gate — current P1-005 Scanner regression
Because one shared Scanner product file changed, rerun exactly the already-corrected current-UI P1-005 Playwright regression once.

Required: 1/1 PASS, zero route mocks.
If it fails, STOP with exact failure. Do not modify P1 product behavior.

## 4. Decisive gate — dedicated P2-002 Playwright
Run only `e2e/p2-002-crossdock-assignment.spec.ts` against:
- final Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
- canonical Testing `yzonugcenguvmojwiihb`;
- the corrected Scanner working tree.

Required rendered proof:
- real login;
- active warehouse resolved before Crossdock entry;
- Crossdock module entry;
- no zone selector/requirement;
- real POST request-task, no route mock;
- operator A receives expected first task;
- visible task number;
- visible source Inbound TU;
- visible plannedQty;
- visible assigned status through correct StatusBadge contract;
- operator B receives the other task, not A's task;
- blocked operator receives the real active-warehouse-task rejection;
- persisted task ownership/status matches rendered/API result;
- no P2-003 sorting/scan/target-TU/completion controls are exercised.

Required: PASS, exit 0.

If the API response itself is correct (expected taskNumber/source/plannedQty) but the same data still does not render, STOP and report this as a genuine Scanner render bug with the exact API payload classification and rendered state. Do not invent another harness correction.

If the API response is null/error, STOP and report the exact response and persisted fixture/task state; do not change product assignment logic in this guide.

## 5. Commit Scanner only if both gates green
If P1-005 and dedicated P2-002 Playwright are green:
- verify Scanner diff contains only:
  - one-line Crossdock StatusBadge prop fix;
  - dedicated P2-002 test synchronization/evidence correction;
- commit and push on current P2-002 Scanner branch;
- record final Scanner SHA;
- require clean Scanner worktree.

Mercato remains exactly `50b27fdd0c9b495ab612ce458bc90e65428ecb93` and clean.

## 6. Canonical P2-002 evidence
Only if both Scanner gates are green, create/update exactly:
`WMS_Outbound/05_EVIDENCE/P2-002_EVIDENCE.md`

Include the already-green backend evidence without rerunning unchanged Mercato suites:
- final Mercato SHA `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
- final Scanner SHA and merge base;
- canonical Testing provenance `yzonugcenguvmojwiihb` (sanitized, no credentials/full URL);
- corrected CON-03 PASS and architecture-aligned proof;
- P2-001 10/10;
- P2-002 PostgreSQL 22/22;
- remaining Mercato 5 suites / 41 tests;
- P1-005 current-UI Scanner 1/1 PASS on final Scanner SHA;
- dedicated P2-002 Playwright PASS with real request-task HTTP evidence, expected task identities, rendered task/source/plannedQty/status, no-zone, second-operator separation, active-task block, zero route mocks;
- explicit statement that the old P1-005 historical selector mismatch was harness drift, not P2-002 product regression;
- no Allocation for Crossdock planning semantics, lazy target TU, one binding -> one OOL -> one CrossDockPickTask, exact plannedQty;
- no P2-003 scope leakage;
- clean final worktrees.

Do not claim Owner Accepted / Human Verified / FINAL PASS.
Commit/push only the canonical evidence file to WMS_Outbound/main.

## 7. STOP/report
Report only:
- final Mercato SHA;
- final Scanner SHA;
- P1-005 current-UI result;
- dedicated P2-002 Playwright result;
- key request-task result classification (operator A/B/blocked);
- WMS evidence commit SHA;
- Mercato clean/dirty;
- Scanner clean/dirty.
Then STOP.
