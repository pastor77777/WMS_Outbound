# P4-002 Antigravity Checkpoint — 2026-09-06T16:09Z

Status: **WIP — DO NOT MERGE**. Stopped at quota. All code committed and pushed. Remaining: test fixes, mandatory regressions, Playwright, evidence, final push.

## Exact Revisions

| Repo | Branch | HEAD SHA |
|---|---|---|
| `pastor77777/Devaxonic-mercato` | `outbound/p4-002` | `32c31ac07` |
| `pastor77777/Devaxonic-scanner` | `outbound/p4-002` | `7891667` |
| `WMS_Outbound` | `main` | `fce5be1` (unchanged) |

## What Was Completed

### Devaxonic-mercato — Backend

1. **Entity + Migration** (`data/entities.ts`, `data/transitions.ts`)
   - Added `WmsOutboundPutBackTask` entity with all P4-002 fields (handoffId, sku, quantity, tu, warehouse, operator, status timestamps)
   - Added `OutboundPutBackTaskStatus` type and `PUT_BACK_TASK_TRANSITIONS` map
   - Migration `Migration20260906160000_wms_outbound_p4_002_putback.ts` — table + all indexes created
   - **DB DDL applied directly to Testing DB** (table exists in Supabase)

2. **PutBackTask Service** (`services/put-back-task-service.ts`)
   - `materializeTaskFromHandoff` — idempotent creation from handoff, unique constraint protection
   - `reconcilePendingHandoffs` — scan unmaterialized handoffs and materialize
   - `assignNext(scope, input, options?)` — transactional advisory lock, shared active-task guard (PickTask + CrossDockPickTask + PutBackTask), strict FIFO `createdAt ASC, id ASC`, idempotent retry returns existing ASSIGNED task, `beforeCommitHook` support for rollback test, creates `WmsOutboundTaskLock`
   - `startTask(scope, input)` — ASSIGNED→IN_PROGRESS, ownership validation, idempotent on retry
   - `getTasks` — supervisor query filter
   - Registered in `di.ts` as `wmsOutboundPutBackTaskService`

3. **Shared active-task guard extended**
   - `pick-task-service.ts`: `checkOperatorActiveTask` and `assignNextPickTask` now check for active `WmsOutboundPutBackTask`
   - `cross-dock-planning-service.ts`: `assignNext` now checks for active `WmsOutboundPutBackTask`

4. **Auto-materialization wired** (`reservation-release-service.ts`)
   - On `releasePostPickReservation`, for each `pickedQty > 0` handoff created, PutBackTask is automatically materialized in the same transaction
   - 3x `createdAt` fixes applied to `WmsOutboundStateTransitionEvent` create calls (required field)

5. **API Routes**
   - `api/returns/request-task/route.ts` — POST, assigns next task from FIFO queue
   - `api/returns/start-task/route.ts` — POST, ASSIGNED→IN_PROGRESS
   - `api/returns/tasks/route.ts` — GET, supervisor list
   - `api/customer-orders/[id]/route.ts` — extended to return `putBackTasks` array
   - Fix: `loc.code` → `loc.name` (WmsWarehouseLocation has `name` not `code`)

6. **Mercato Supervisor UI** (`backend/customer-orders/[id]/page.tsx`)
   - Added `putBackTasks` state and `putback-tasks-section` table showing PutBackTask list

### Devaxonic-scanner

1. `src/constants/strings.js` — `returns` mode labels (EN/PL), `modeReturnsDesc`
2. `src/screens/ModeScreen.js` — returns mode entry (`first: 'returnsTask'`)
3. `src/lib/api.js` — `requestPutBackTask` and `startPutBackTask`
4. `src/screens/ReturnsTaskScreen.js` — full human flow: no zone, FIFO request, active-task error, start ASSIGNED→IN_PROGRESS
5. `App.js` — registered `returnsTask: ReturnsTaskScreen`

**Scanner expo export**: passed successfully (279 modules bundled)

## Checks Run and Results

| Check | Result |
|---|---|
| `npx expo export --platform web` (scanner) | ✅ PASSED — 279 modules, no errors |
| DDL applied to Testing DB | ✅ table `wms_outbound_put_back_tasks` exists |
| `tsc --noEmit` (apps/mercato) | ❌ RUNNING at checkpoint — 5 type errors identified and fixed during session; final run not confirmed |
| P4-002 PostgreSQL test suite (17 tests) | ❌ 10/17 PASS, 7 FAIL — fixes in progress (see below) |
| P4-001 regression | NOT YET RUN in this session |
| P3-003 regression | NOT YET RUN in this session |
| Playwright UI acceptance | NOT STARTED |

## P4-002 Test Suite — Current State (10/17 passing)

File: `apps/mercato/src/modules/wms_outbound/services/__tests__/p4-002-postgres.integration.test.ts`

**PASSING (10):**
1 (materialize from handoff), 2 (replay dedup), 3 (concurrent materialize uniqueness), 7 (FIFO oldest first), 9 (no-zone), 10 (active-task block), 11 (race winner), 12 (warehouse isolation), 13 (idempotent retry), 14 (start auth + idempotent)

**FAILING (7) — root causes identified:**
- Tests 4, 5, 6, 16: `releasePostPickReservation` needs entity registration context — fails because `customer-order-service` internal state transition references entity not registered in test ORM init list (needs `WmsOutboundCrossDockPickTask`, `WmsOutboundLocationShortage`, `WmsOutboundSupervisorDecision`, `WmsOutboundPreConfirmPickObservation`, `WmsWarehouse`, `WmsUserWarehouse`, `WmsWarehouseZone`, `WmsTransportUnit`, `WmsTuExpectedContent` — compare to p4-001 and p1-007 entity lists)
- Test 8: `externalOrderId` required on `WmsOutboundCustomerOrder` — add it to minimal CO fixture in test
- Tests 14, 15: error message pattern `not assigned to` — actual message is `is not assigned to PutBackTask` — test regex already matches but `expect(...).rejects.toThrow(/not assigned to/)` — ACTUALLY PASSES, check mismatch in output suggests fixture TU field issue
- Test 17: `WmsOutboundTaskLock.actorId` query — fixed in latest commit (was `operatorId`)

## Exact Remaining Work

### 1. Fix test failures (self-repair)

**a) Tests 4, 5, 6, 16** — add missing entities to ORM init list in test file:
```
WmsOutboundTuSequence, WmsOutboundLocationShortage, WmsOutboundSupervisorDecision,
WmsOutboundPreConfirmPickObservation, WmsWarehouse, WmsUserWarehouse,
WmsWarehouseZone, WmsTransportUnit, WmsTuExpectedContent
```
(Compare p4-001 test `beforeAll` entity list at lines 56-86 with p1-007 at lines 89-118)

**b) Test 8** — add `externalOrderId: \`EXT-HIGH-${randomUUID().slice(0,8)}\`` and `externalOrderId: \`EXT-LOW-${randomUUID().slice(0,8)}\`` to the two CustomerOrder creates.

### 2. Typecheck

Run `npx tsc --noEmit` in `apps/mercato`. Expected clean — all 5 known type errors were fixed. If any remain, fix and confirm.

### 3. Mandatory Regressions

```bash
set -a && source /etc/mercato-localhost.env && set +a && export NODE_TLS_REJECT_UNAUTHORIZED=0
yarn jest src/modules/wms_outbound/services/__tests__/p4-001-postgres.integration.test.ts --runInBand
yarn jest src/modules/wms_outbound/services/__tests__/p3-003-postgres.integration.test.ts --runInBand
```
Expected: 18/18, 18/18 — no regressions.

### 4. P4-002 Suite — Full 17/17

After fixing (a) and (b) above, re-run:
```bash
yarn jest src/modules/wms_outbound/services/__tests__/p4-002-postgres.integration.test.ts --runInBand
```
Target: 17/17 PASS.

### 5. Mercato service restart (if needed)

If mercato dev server is running, restart to pick up new routes. Test URL: `https://devaxonic-test.info-start.com.pl`

### 6. Scanner service restart

```bash
scanner-testing restart
```

### 7. Playwright UI Acceptance

Run against `https://scanner.info-start.com.pl` and `https://devaxonic-test.info-start.com.pl`:

- **Journey A (Scanner)**: Returns module → no zone selector → FIFO task assigned → start task → status IN_PROGRESS
- **Journey B (Scanner)**: Operator already active → clear no-task feedback
- **Journey C (Scanner)**: Concurrent race → one winner
- **Journey D (Mercato)**: Customer order detail → putBackTasks section visible with CREATED/ASSIGNED/IN_PROGRESS task

Evidence class: PLAYWRIGHT VERIFIED.

### 8. Evidence File

Write `WMS_Outbound/05_EVIDENCE/P4-002_EVIDENCE.md` following `EVIDENCE_STANDARD.md`.

### 9. Final Clean Commit + Push

Replace WIP commit with final commit:
```bash
git commit --amend --no-edit   # or squash WIP and create clean commit
git push origin outbound/p4-002 --force-with-lease
```
Also push evidence to `WMS_Outbound/main`.

## Key Technical Notes

- `WmsOutboundTaskLock` uses `actorId` (not `operatorId`) — all code and tests now corrected
- `WmsWarehouseLocation` has `name` field (not `code`) — route fixed
- `WmsOutboundStateTransitionEvent.createdAt` is required — all 3 create sites in put-back-task-service + 1 in reservation-release-service now include `createdAt: new Date()`/`now`
- `assignNext` signature: `assignNext(scope, input, options?)` — test file uses correct order
- Service method `releasePostPickReservation` (not `executePostPickCancellation`) — test file uses correct method name
- DB table `wms_outbound_put_back_tasks` already created in Testing DB via direct DDL
