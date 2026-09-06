# P4-003 — RF PutBack location validation loop and Inventory recovery

**Status:** current execution guide
**Effective:** 2026-09-06
**Base:** Mercato `outbound/p4-002` @ `d75dccbc7` (P4-002 accepted) → new branch `outbound/p4-003`
**Base:** Scanner `outbound/p4-002` @ `7d13e34` (P4-002 accepted, frozen) → new branch `outbound/p4-003`

## Scope

Complete P4 STEP 4–5 (`P4 R6`–`R8`) on top of the accepted P4-002 base (`CREATED -> ASSIGNED -> IN_PROGRESS`):

- `FR-P4-03` — location validation before put-away (extends P4-002's task creation).
- `FR-P4-04` — `IN_PROGRESS <-> LOCATION_VALIDATION` loop with no attempt limit / no escalation; `LOCATION_VALIDATION -> COMPLETED` recovers `Inventory PICKED -> AVAILABLE` exactly once.
- Preserve `FR-P4-05` / P4-002 FIFO assignment and single-active-task boundary untouched.

Acceptance: `TC-050`, `TC-051`, `TC-052`, `TC-100` (TC-100 already covered by P4-002; re-run as regression only).

## Architect authority

`WMS_Outbound/01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_4_physical_putback_EN.md` STEP 4–5, R6–R8. Location proposed by System WMS or indicated by operator; must pass WMS validation before put-away. Rejected location returns `LOCATION_VALIDATION -> IN_PROGRESS`, no limit, no escalation, recommendation (`proposedLocationId`, defaults to `sourceLocationId`) stays available. Valid completion: `LOCATION_VALIDATION -> COMPLETED`, `Inventory PICKED -> AVAILABLE` exactly once.

Do not import Inbound Putaway semantics (no `IN_PUTAWAY`, no sector `TRANSIT`, no PutawayTask ownership model). Outbound P4 ends directly in ordinary `Inventory AVAILABLE`.

## Current state (verified in Git before this guide was written)

- `WmsOutboundPutBackTask` (Mercato `apps/mercato/src/modules/wms_outbound/data/entities.ts`) already has `status: OutboundPutBackTaskStatus` including `LOCATION_VALIDATION`/`COMPLETED`, and `proposedLocationId`/`sourceLocationId`. It has **no** `validatedLocationId` column yet — add it.
- `PUT_BACK_TASK_TRANSITIONS` (`data/transitions.ts`) already declares `IN_PROGRESS -> LOCATION_VALIDATION`, `LOCATION_VALIDATION -> IN_PROGRESS`, `LOCATION_VALIDATION -> COMPLETED`. Reuse as-is; do not redefine.
- `put-back-task-service.ts` implements `materializeTaskFromHandoff`, `reconcilePendingHandoffs`, `assignNext`, `startTask`, `getTasks`. It has **no** location-submission/completion method — add `submitLocation`.
- Outbound never mutates `wms_inventory`'s `WmsInventoryBalance` anywhere today (verified by grep — `atp-reservation-service.ts`/`reservation-release-service.ts` only manage the Outbound-local soft `WmsOutboundAtpReservation`, which already recovered ATP at P4 STEP 2/P4-001). The **only** correct place in this codebase to perform the P4 STEP 5 physical `Inventory PICKED -> AVAILABLE` recovery is `wms_inventory`'s shared `WmsInventoryBalance`/`WmsInventoryMovement` via the existing `createInventoryAdjustmentService(...).adjustBalance(...)` (`apps/mercato/src/modules/wms_inventory/services/inventory-adjustment-service.ts`). Reuse it; do not invent a parallel ledger.
- `adjustBalance` already enforces the reusable `STORAGE`-capability location rule (`LOCATION_NOT_STORAGE`) — reuse this exact rule as the location-validity check instead of re-implementing it.
- Scanner `ReturnsTaskScreen.js` already renders an `IN_PROGRESS` banner reading "Staged for destination validation (P4-003)" — this is the extension point.

## Backend design (Mercato)

1. **Migration** `apps/mercato/src/modules/wms_outbound/migrations/` — add nullable `validated_location_id uuid` to `wms_outbound_put_back_tasks`. Follow the exact style of `Migration20260906160000_wms_outbound_p4_002_putback.ts` (idempotent `if not exists` guards where applicable).
2. **Entity** — add `validatedLocationId?: string | null` property (`@Property({ name: 'validated_location_id', type: 'uuid', nullable: true })`) to `WmsOutboundPutBackTask`.
3. **Service** `put-back-task-service.ts`: add `submitLocation(scope, { taskId, operatorId, scannedLocation })`:
   - Lock the task row (`PESSIMISTIC_WRITE`) by id+scope; verify ownership (`task.operatorId === operatorId`).
   - Idempotent retry: if `task.status === 'COMPLETED'`, return `{ task, accepted: true }` without re-mutating inventory.
   - Require current status `IN_PROGRESS` (use `validateStateTransition('PutBackTask', task.status, 'LOCATION_VALIDATION')`); persist the `IN_PROGRESS -> LOCATION_VALIDATION` audit event (`PutBackTaskLocationSubmitted`).
   - Resolve `scannedLocation` against `WmsWarehouseLocation` scoped to `task.warehouseId`, matching `id`, `name`, or (if present) `code`, with `deletedAt: null` — mirror the exact match pattern already used in `pick-task-service.ts` (`loc.name !== input.scannedLocation && loc.code !== input.scannedLocation`).
   - If no matching location is found: reject (see below) with reason `Location not found`.
   - If a location is found: resolve `MasterdataItem` by `{ organizationId, tenantId, sku: task.sku, deletedAt: null }`. Not found or missing `baseUomId` is a **hard system error** (throw) — it is a data-integrity gap, not a retryable location problem; do not loop the operator on it.
   - Call `createInventoryAdjustmentService(tx).adjustBalance(scope, { actorId: operatorId, itemId: item.id, warehouseId: task.warehouseId, locationId: location.id, lotId: null, serialId: null, quantity: task.quantity, txnUomId: item.baseUomId, reasonCode: 'P4_PHYSICAL_PUTBACK_COMPLETION' })` inside the same transaction (pass the current `tx`, not a fresh `em`).
     - `ok: true` → **accept**: transition `LOCATION_VALIDATION -> COMPLETED` (persist `PutBackTaskCompleted` event), set `task.validatedLocationId = location.id`, `task.completedAt = now`. Return `{ task, accepted: true }`.
     - `ok: false` with `code` in `LOCATION_NOT_FOUND` / `LOCATION_NOT_STORAGE` / `WAREHOUSE_NOT_FOUND` → **reject** (see below) with the service's own `reasons[0]`.
     - `ok: false` with any other code (`ITEM_NOT_FOUND`, `ITEM_BASE_UOM_NOT_FOUND`, `TXN_UOM_NOT_FOUND`, `UOM_CONVERSION_NOT_FOUND`, `NEGATIVE_ON_HAND`) → hard system error (throw); these are not location problems.
   - **Reject path** (no location, or `LOCATION_NOT_FOUND`/`LOCATION_NOT_STORAGE`/`WAREHOUSE_NOT_FOUND`): transition `LOCATION_VALIDATION -> IN_PROGRESS` (persist `PutBackTaskLocationRejected` event with the reason), leave `proposedLocationId` unchanged (keeps the standing System WMS recommendation available), perform **zero** inventory mutation. Return `{ task, accepted: false, reason }`.
   - No attempt-count field, no automatic escalation, no attempt limit — callers may invoke `submitLocation` an unbounded number of times while `IN_PROGRESS`.
4. **API route** `apps/mercato/src/modules/wms_outbound/api/returns/submit-location/route.ts`: same auth/operator-id pattern as `start-task`/`request-task`. Body `{ taskId: uuid, scannedLocation: string }`. On rejection return **HTTP 200** with `{ task: {...}, accepted: false, reason }` (a rejected location is a normal business outcome, not a request failure — the Scanner `request()` helper treats any non-2xx as `ok:false`/throw, which would wrongly surface a rejection as a client error). On hard system error, return the usual `400` with `{ error }`.
5. Extend the `tasks` GET route's projection with `validatedLocationId` for Supervisor visibility (trivial, no behavior change).
6. Do not touch `assignNext`/`startTask`/FIFO/single-active-task logic — P4-002 boundary is frozen.

## Scanner design (RF)

1. `src/lib/api.js`: add `submitPutBackLocation(taskId, scannedLocation)` calling `POST /api/wms_outbound/returns/submit-location`. Do **not** throw on `accepted: false` — that is a normal rejected-location result, not a transport error; only throw on `!result.ok` (HTTP/transport failure) per the existing `request()` contract.
2. `src/screens/ReturnsTaskScreen.js`: replace the static `IN_PROGRESS` banner with a location-scan form (mirror `PickingTaskScreen.js`'s `TextInput` + `testID="scan-location-input"` pattern):
   - Show the recommended location (`task.proposedLocationId`/its resolved code — resolve via the existing `sourceLocationCode` pattern from `request-task`, extend `submit-location`'s/`start-task`'s response projection with a resolved `proposedLocationCode` if useful for operator UX).
   - `TextInput testID="returns-location-input"` for the scanned/entered destination.
   - `PrimaryButton testID="returns-submit-location-btn"` calling `submitPutBackLocation`.
   - On `accepted: false`: show the rejection reason (`testID="returns-location-rejected"`), keep the form open, keep status `IN_PROGRESS`, let the operator retry immediately — no limit, no lockout.
   - On `accepted: true` (`task.status === 'COMPLETED'`): show a completion confirmation (`testID="returns-completed-banner"`) with the validated location and confirm `Inventory` recovered to `AVAILABLE`; do not auto-navigate away (Supervisor/operator can start the next return manually via the existing `load()`/request-next flow).
3. Preserve the existing `ASSIGNED -> start` flow untouched.

## Tests (decisive, both repos)

### Mercato — `apps/mercato/src/modules/wms_outbound/services/__tests__/p4-003-postgres.integration.test.ts`

Reuse the exact fixture helper pattern from `p4-002-postgres.integration.test.ts` (`createPostPickFixture`, `WmsOutboundPhysicalReturnHandoff` → `materializeTaskFromHandoff` → `assignNext` → `startTask` to reach `IN_PROGRESS`), extended with:
- a real `WmsWarehouseLocation` row with `STORAGE` capability for the valid destination (fixtures today only use bare `randomUUID()` for `sourceLocationId` — that is **not sufficient** for `submitLocation`, which needs a real, resolvable location row);
- a `MasterdataItem` row for the task's `sku` with `baseUomId` set to a fixed random UUID (no `WmsInventoryUomConversion` row is needed as long as `txnUomId === item.baseUomId`, which the service guarantees — verify `uom-conversion-service.ts`'s same-UOM fast path if extending this).

REAL POSTGRESQL INTEGRATION decisive cases (`.ai/TESTING.md` evidence classes):
1. Rejected location (non-existent id/name) → task returns to `IN_PROGRESS`, zero `WmsInventoryBalance`/`WmsInventoryMovement` rows created, zero `PutBackTask.completedAt`.
2. Rejected location (existing location but missing `STORAGE` capability) → same zero-movement assertion.
3. Unlimited retry: 2+ consecutive rejections on the same task, then a valid location → completes; assert no attempt-count ceiling exists in code (loop N>2 times in the test itself, e.g. 5 rejections, to demonstrate no limit).
4. Valid location → `PutBackTask COMPLETED`, `validatedLocationId` set, `completedAt` set, exactly one `WmsInventoryMovement` row (`disposition = ADJUSTMENT_IN`, `quantity == task.quantity`), `WmsInventoryBalance.onHandQuantity`/`availableQuantity` incremented by exactly `task.quantity` at the validated location (create the balance row if absent, verify delta if a pre-existing balance row exists at that location).
5. Idempotent re-submission after `COMPLETED` (retry/duplicate client call) → no second `WmsInventoryMovement`, no double-counted balance, returns `accepted: true` (exactly-once, `.ai/TESTING.md` §5 rollback/idempotency discipline).
6. Real concurrency: two independent overlapping `submitLocation` calls against the **same** `IN_PROGRESS` task racing to submit two different (both valid) locations — PostgreSQL row lock on the task must serialize them; assert exactly one `WmsInventoryMovement`/one balance increment total (not two), with real backend PID/lock evidence per `.ai/TESTING.md` §5.
7. Regression: P4-002's FIFO/single-active-task/assignment tests continue to PASS unmodified (do not edit `p4-002-postgres.integration.test.ts`; just rerun it).

### Scanner — Jest component test for `ReturnsTaskScreen.js` location-submit UI states (reject → retry → accept), following the existing component test conventions for other RF screens in this repo.

### Playwright (Scanner) — extend/add to the existing P4-002 rendered acceptance suite (`test(p4-002): add rendered Playwright acceptance suite for Returns/PutBack module`): drive the real RF UI through `IN_PROGRESS` → scan an invalid location → observe rejection banner, stay `IN_PROGRESS` → scan a valid location → observe `COMPLETED`. Zero route mocks, real backend, real DB assertions for the persisted `PutBackTask.status`/`validatedLocationId` and the `WmsInventoryBalance` delta (`TC-052` loop, `TC-051`/`TC-050` happy path). This satisfies `PLAYWRIGHT VERIFIED` per `.ai/TESTING.md` §6, not `HUMAN VERIFIED`.

## Definition of Done

- Invalid destination never completes the task and never moves stock (zero balance/movement writes).
- Valid completion recovers exactly `task.quantity` once, moves `PutBackTask` to `COMPLETED`.
- No attempt limit, no automatic escalation anywhere in the implementation.
- P4-002 FIFO/ownership/regression suite still green, unmodified.
- Real PostgreSQL integration evidence (concurrency + rollback/idempotency per `.ai/TESTING.md`), rendered Playwright evidence for `TC-050`/`TC-051`/`TC-052`, `TC-100` regression.
- Mercato generate/typecheck/build green; Scanner build/tests green.
- Push both repos on `outbound/p4-003`; update `Devaxonic-WMS/.ai/STATE.md`, `WMS_Outbound/STATE.md` and current handovers only after independent supervisor verification and explicit Owner acceptance — do not self-declare acceptance.

## Escalation boundary

Two-strikes applies only to the same material blocked path (e.g., a specific PostgreSQL concurrency assertion that fails twice for different root causes attempted). Ordinary fixture/build/typecheck/lint failures are self-repaired in place per `AGENTS.md`/`.ai/OPERATIONS.md`. Do not expand scope into P4-002's FIFO logic, Inbound Putaway semantics, or any wms_inventory manual-adjustment UI.
