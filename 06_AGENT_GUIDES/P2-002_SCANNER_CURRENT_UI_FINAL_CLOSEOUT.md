# P2-002 — Scanner current-UI regression correction and final closeout

## Authority / execution boundary

Continue the SAME Codex session. Execute P2-002 only.

Do not change Architect Source, Canon, Task Catalog, STATE, handovers, acceptance state, Mercato product code, or Scanner application/product code in this shot.

Canonical PostgreSQL evidence/runtime is Supabase Testing project `yzonugcenguvmojwiihb`. Local PostgreSQL is forbidden.

Frozen green state entering this shot:
- Mercato final SHA: `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- Scanner product SHA: `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`
- corrected CON-03: 1/1 PASS
- P2-001 PostgreSQL regression: 10/10 PASS
- P2-002 dedicated PostgreSQL: 22/22 PASS
- remaining mandatory Mercato regressions: 5 suites / 41 tests PASS
- Mercato and Scanner product worktrees clean

Do not rerun green Mercato gates. This shot is Scanner test/evidence closeout only.

## 1. Root cause is already established — do not rediscover it

The previous P1-005 Scanner run reached a successful real API assignment, but then failed because the historical acceptance spec asserted an obsolete UI string.

Repository facts:

1. Historical P1-005 Scanner SHA `8199b330cb739a45e2c615a3f2aa3803336be724` rendered:
   - screen title `Assigned Pick Task`;
   - historical test later used a `Back to Zones` button.
2. Current accepted Scanner product at `8ff264e2e03d75abd422f69a7ff1f713f10d32fc` renders after assignment:
   - screen title `RF Picking Execution`;
   - stable task selector `testID="task-number"`;
   - status through current `StatusBadge code={...}`;
   - navigation back through `ScreenHeader` accessibility button `Back` / `Wstecz`.
3. The historical `e2e/p1-005-real-scanner-assignment.spec.ts` is still the old acceptance harness and therefore its UI assertions are stale relative to later accepted RF Scanner evolution.
4. P2-002 Scanner product diff from frozen baseline `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` to `8ff264e...` is only the bounded Crossdock addition (`App.js`, strings, `src/lib/api.js`, `CrossdockTaskScreen.js`, `ModeScreen.js`). It does not modify `PickingZoneScreen`, `PickingTaskScreen`, auth, warehouse selection, or standard `requestPickingTask` behavior.
5. Canonical Testing is multi-warehouse. Never rely on lazy "sole warehouse" assignment for newly-created test users. The test user must receive an explicit `wms_user_warehouses` row for the fixture warehouse.

Classification:

`P1-005 current failure = STALE ACCEPTANCE HARNESS / CURRENT-UI SELECTOR MISMATCH, NOT P2-002 PRODUCT REGRESSION.`

Do not reopen this classification unless the real API/backend behavior fails after the harness is aligned to the current UI contract.

## 2. Authorized Scanner changes — test/e2e only

Authorized files:
- update only `e2e/p1-005-real-scanner-assignment.spec.ts`;
- add one dedicated `e2e/p2-002-crossdock-assignment.spec.ts` if absent.

No Scanner application file may change.
No package dependency may change.
No Mercato file may change.

### 2A. Make P1-005 fixture deterministic

In `p1-005-real-scanner-assignment.spec.ts`:

- retain the unique test user and real auth flow;
- after creating the user/ACL, explicitly create `wms_user_warehouses` for that user with:
  - the existing test `organization_id` / `tenant_id`;
  - fixture warehouse `9b174240-2376-4a5b-8d5a-5f3271e347d7`;
  - `is_default = true`;
- clean that row in `afterAll` before deleting the user;
- do not rely on `ensureDefaultAssignment` because Testing has multiple warehouses;
- do not change product warehouse semantics.

The preceding attempt already reached successful assignment, so do not invent additional fixture changes unless required by an explicit current DB constraint.

### 2B. Align P1-005 assertions to the current accepted Scanner UI

After clicking `Request Next Task`, use the current UI contract:

- assert header `RF Picking Execution` is visible;
- assert `page.getByTestId('task-number')` equals the fixture task number;
- assert status using the current rendered/accessibility contract (`Status: ASSIGNED` or equivalent current `StatusBadge` output), not an obsolete component prop assumption;
- assert the fixture order number is visible;
- assert the fixture SKU is visible in the current Pick Task Lines rendering;
- keep the independent PostgreSQL assertion that the assigned task has the expected operator/status/warehouse/zone;
- keep the persisted transition assertion where the current accepted backend still emits it.

For the active-task guard part:

- DO NOT look for a button named `Back to Zones`;
- use the current `ScreenHeader` Back control (`Back` / `Wstecz`) to return to `PickingZoneScreen`;
- select the same zone and request another task;
- assert the rendered active-task error from the real API.

Delete/replace stale selectors only. Do not weaken business assertions.

Forbidden stale gates after this correction:
- `Assigned Pick Task` as required current title;
- `Back to Zones` as required current navigation control.

## 3. Freeze Scanner test-only revision

Before running final Scanner gates:

1. verify `git diff --name-only` contains only:
   - `e2e/p1-005-real-scanner-assignment.spec.ts`;
   - optionally the new `e2e/p2-002-crossdock-assignment.spec.ts` once created;
2. no app/product file changes;
3. commit the test-only changes on the current P2-002 Scanner branch with a descriptive commit;
4. push and record the new final Scanner SHA.

If any product file is dirty, STOP.

## 4. Run current-UI P1-005 regression once on the final Scanner SHA

Run the corrected existing spec through repository-native Playwright against:
- final Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
- canonical Testing DB/runtime;
- the new final Scanner test-only SHA;
- zero route mocks.

Required:
- real login succeeds;
- warehouse context resolves to the explicit fixture assignment;
- real zone request works;
- real standard PickTask API assigns the fixture task;
- current `RF Picking Execution` screen renders the assigned task;
- persisted PostgreSQL state matches the operator;
- second request is blocked by single-active-task protection;
- 1/1 PASS, exit 0.

If the real API/backend business invariant fails, STOP and report the exact API/status/invariant.
Do not start another selector-hunting loop. All known stale selectors are corrected in §2B.

## 5. Dedicated P2-002 Playwright — build it deterministically

If P1-005 passes, run/create the dedicated P2-002 rendered acceptance spec.

The spec must use the normal Scanner app and real backend with zero route mocks. Direct SQL is allowed only for deterministic fixture creation/cleanup and independent persistence assertions.

### 5A. Deterministic fixture

Use unique run-scoped IDs/emails.

Create three Warehouse Operator test users:
- `operatorA` — normal crossdock assignment;
- `operatorB` — independent second operator;
- `operatorBlocked` — already owns an active warehouse-task lock.

For every test user:
- create normal Testing user + accepted ACL fixture;
- explicitly insert `wms_user_warehouses` with the same fixture warehouse and `is_default = true`;
- never rely on implicit sole-warehouse selection.

Seed two `wms_outbound_cross_dock_pick_tasks` directly in canonical Testing, same org/tenant/warehouse, both `CREATED`:

- Task A must outrank Task B under BOTH supported queue modes: give Task A lower/better priority AND earlier SLA; Task B worse priority AND later SLA.
- use distinct task numbers, source Inbound TU IDs, binding IDs, OOL IDs, COL IDs, item IDs and planned quantities;
- the P2-002 migration has no FK requirement forcing the fixture IDs to reference physical P2-003 objects; do not create unnecessary target TU/sorting data.

For `operatorBlocked`, seed one unexpired `wms_outbound_task_locks` row with `status = ACTIVE`, same org/tenant/warehouse and `actor_id = operatorBlocked`. This is the exact shared active-task primitive checked by `assignNext`.

Clean all fixture rows deterministically in reverse ownership order, including task locks created by successful crossdock assignments.

### 5B. Rendered journey

Use separate real browser contexts/sessions.

#### Operator A
1. normal Scanner login;
2. explicit warehouse context resolves;
3. Choose mode screen appears;
4. enter `Crossdock`;
5. assert `crossdock-module` renders;
6. assert `crossdock-no-zone` / visible no-zone message;
7. assert Task A (the globally better fixture task) is assigned and rendered;
8. assert rendered task number/identity;
9. assert rendered source Inbound TU;
10. assert rendered planned quantity equals DB fixture value;
11. assert there is no P2-003 scan/sort/quantity/quality/target-TU completion UI.

#### Operator B
1. independent browser context and normal login;
2. enter `Crossdock`;
3. assert Operator B does NOT receive Task A;
4. because Task B remains eligible, it may receive Task B — assert task identity differs from Task A;
5. verify persisted Task A remains assigned only to Operator A and Task B only to Operator B.

This proves two operators cannot receive the same task without requiring a fake error when another eligible task exists.

#### OperatorBlocked
1. independent normal login;
2. enter `Crossdock`;
3. assert rendered error from the real backend states that the operator already has an active warehouse task;
4. assert no crossdock task becomes assigned to this operator.

### 5C. Persistence proof

After browser actions, query canonical Testing directly and prove:
- Task A `ASSIGNED` to operatorA only;
- Task B, if consumed, `ASSIGNED` to operatorB only;
- no duplicate assignment of Task A;
- task-lock rows exist for real crossdock assignments;
- operatorBlocked received no CrossDockPickTask;
- no Allocation was created by this crossdock journey;
- no target physical Outbound TU was created by this P2-002 assignment journey.

Required dedicated P2-002 Playwright result: PASS, exit 0.

## 6. Scanner final SHA policy

The final Scanner SHA may differ from `8ff264e...` ONLY because of the test/e2e correction/addition above.

Before evidence:
- compare final Scanner SHA against `8ff264e...`;
- confirm every additional changed line is under `e2e/` only;
- confirm application/product diff remains exactly the original bounded P2-002 Crossdock implementation;
- worktree clean.

## 7. Canonical P2-002 evidence

Only when P1-005 current-UI regression and dedicated P2-002 Playwright are green, create/update exactly:

`05_EVIDENCE/P2-002_EVIDENCE.md`

Record:
- accepted P2-001 base/evidence reference;
- final Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
- final Scanner SHA and merge base;
- explicit distinction between Scanner product commit `8ff264e...` and any later e2e-only harness commit;
- canonical Testing project ref `yzonugcenguvmojwiihb` only, no credentials/full URL;
- corrected CON-03 1/1 PASS;
- P2-001 10/10 PASS;
- P2-002 dedicated PostgreSQL 22/22 PASS;
- remaining Mercato regressions 5 suites / 41 tests PASS;
- P1-005 Scanner current-UI regression 1/1 PASS and exact command;
- root-cause note: historical P1-005 assertion expected `Assigned Pick Task`/`Back to Zones`, while current accepted RF UI renders `RF Picking Execution`/ScreenHeader Back; test-only correction, no product change;
- explicit warehouse fixture because Testing is multi-warehouse;
- dedicated P2-002 Playwright command/result and all three operator outcomes;
- no zone selector for crossdock;
- rendered task/source/plannedQty proof;
- double-assignment prevention;
- active warehouse task block;
- no Allocation;
- lazy/no physical target TU;
- no P2-003 actions;
- clean Mercato/Scanner worktrees.

Do not claim Owner Acceptance, HUMAN VERIFIED, or FINAL PASS.

Commit/push the canonical evidence file to WMS_Outbound/main.

## 8. STOP / report

STOP and report only:
- final Mercato SHA;
- final Scanner SHA;
- P1-005 Scanner current-UI regression count/result;
- dedicated P2-002 Playwright count/result;
- WMS evidence commit SHA;
- Mercato clean/dirty;
- Scanner clean/dirty.
