# P4-002 — PutBackTask Model, FIFO Assignment and Task Lifecycle — Evidence

Date: 2026-09-06 UTC
Evidence class: REAL POSTGRESQL INTEGRATION, REAL CONCURRENCY and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/p4-002`
- Mercato final candidate commit SHA: `a73a4353cf094d67b31529448592fc03ffd9c9dd`
- Mercato accepted base lineage: P4-001 accepted at `66e2e8620041d2db1d10d069e286936083667139`; P3-003 accepted at `f600782496e865603200d46d7e6041a54f90b9a4`
- Scanner branch: `outbound/p4-002`
- Scanner final candidate commit SHA: `7d13e34fc66fe19149b43a6747b5300c2bdcf945`
- Scanner accepted base lineage: `135d86e1342bae7b21a8b676b1ef220a39a0f0b5`
- WMS evidence commit: this commit, on `WMS_Outbound/main`
- Item: 30/37 — P4-002: PutBackTask model, FIFO assignment and task lifecycle.
- Executor handoff: Antigravity (AGY) reached quota mid-item and produced `06_AGENT_GUIDES/P4-002_EXECUTION.md` and `08_HANDOVER/P4-002_AGY_CHECKPOINT.md` at Mercato `32c31ac07` / Scanner `7891667`. Claude Code continued from that exact checkpoint with no reset, per Owner-authorized executor switch.

## Authority Chain

`proces_4_physical_putback.md` P4 R5, R6, R9 -> `FR-P4-03`, `FR-P4-05` -> `scenariusze_testowe_outbound.md` `TC-050`, `TC-051`, `TC-052`, `TC-100`.

## Schema / Migration and Uniqueness Mechanism

- Table `wms_outbound_put_back_tasks` (`Migration20260906160000_wms_outbound_p4_002_putback.ts`): `id`, `organization_id`, `tenant_id`, `warehouse_id`, `task_number`, `handoff_id`, `customer_order_id`, `outbound_order_id`, `outbound_order_line_id`, `sku`, `quantity`, `tu_id`, `source_location_id`, `proposed_location_id`, `status`, `operator_id`, `assigned_at`, `started_at`, `completed_at`, `created_at`, `updated_at`.
- No `priority` or `slaDeadline` column exists on this entity (explicit exclusion per P4 R9).
- Durable uniqueness: `UNIQUE (organization_id, tenant_id, handoff_id)` — one accepted P4-001 physical-return handoff produces exactly one PutBackTask, enforced at the database, not only application-side.
- Additional `UNIQUE (organization_id, tenant_id, task_number)`.

## Task Lifecycle and FIFO Assignment Mechanism

- `put-back-task-service.ts`:
  - `materializeTaskFromHandoff` — idempotent `CREATED` materialization from an accepted P4-001 handoff, protected by the DB unique constraint.
  - `reconcilePendingHandoffs` — scans unmaterialized positive handoffs and materializes them; zero-picked handoffs (`quantity <= 0`) never materialize.
  - `assignNext` — single DB transaction: `pg_advisory_xact_lock` scoped per `organization/tenant/warehouse`, then `PESSIMISTIC_WRITE` reads of `WmsOutboundPickTask`, `WmsOutboundCrossDockPickTask`, `WmsOutboundPutBackTask`, and `WmsOutboundTaskLock` for the requesting operator (shared single-active-task guard extended, not a PutBack-only silo). Candidates are queried by `(organization, tenant, warehouse, status = CREATED)` and sorted strictly by `createdAt` ascending with an `id` tie-break — no `priority`/`slaDeadline` column exists to read, so FIFO cannot be reordered by either. Idempotent retry on the same operator/task returns the existing `ASSIGNED` row without consuming a second task.
  - `startTask` — `ASSIGNED -> IN_PROGRESS`, ownership-validated, idempotent on same-operator retry, rejects wrong owner and illegal/regressive transitions with zero mutation via `validateStateTransition`.
- `pick-task-service.ts` (`checkOperatorActiveTask`, `assignNextPickTask`) and `cross-dock-planning-service.ts` (`assignNext`) extended to also treat an active `WmsOutboundPutBackTask` (`ASSIGNED`/`IN_PROGRESS`) as blocking, preserving one shared cross-task-type ownership invariant.

## Self-Repair During This Session (root-caused, not worked around)

Continuing from the AGY checkpoint (10/17 P4-002 tests passing, 7 failing), the actual failures diverged from the checkpoint's own diagnosis on inspection:

1. **`state-transition-service.ts`** — both `WmsOutboundStateTransitionEvent` create sites (`transition()` and `recordTransition()`) omitted an explicit client-side `id`, unlike every other create call for this entity in the module. When a P4-002 PutBackTask audit event (explicit `id`) and one of these id-less events were flushed together in the same `reservation-release-service` transaction, MikroORM's batch insert (`persistNewEntitiesBatch`) produced a row/index mismatch (`Cannot read properties of undefined (reading 'id')`). Fixed by adding `id: randomUUID()` to both sites, matching the rest of the codebase's convention. This bug was latent in shared, previously-accepted code and only surfaced once P4-002 introduced a second same-transaction creator of this entity type.
2. **Test/service argument-order mismatch** — `p4-002-postgres.integration.test.ts` called `materializeTaskFromHandoff(handoff, scope)` / `reconcilePendingHandoffs(warehouseId, scope)`, the reverse of the service's own signature and its own internal usage (`reconcilePendingHandoffs` calling `this.materializeTaskFromHandoff(scope, handoff, tx)`). Fixed the test call sites to match the service, not the reverse.
3. **Test fixture bugs** — missing `externalOrderId` on the FIFO-priority fixture's two `CustomerOrder` rows (now-required field); a `taskNumber` assertion regex that didn't match the service's actual `PBT-<base36>-<uuid4>` format; a `WmsInventoryBalance` fixture/assertion using non-existent property names (`onHand`/`reserved` instead of the entity's actual `onHandQuantity`/`reservedQuantity`), which silently produced `undefined` and made the physical-inventory-untouched assertion (invariant #16) pass for the wrong reason. Corrected to assert the balance is genuinely unchanged (`onHandQuantity`/`reservedQuantity` both remain at their fixture value).
4. **Regression suites missing the new entity** — `p4-001-postgres.integration.test.ts`, `p3-003-postgres.integration.test.ts`, `p1-005-postgres.integration.test.ts`, `p1-006-postgres.integration.test.ts`, and `p2-002-crossdock-planning-postgres.integration.test.ts` each construct their own isolated `MikroORM` instance and now exercise a shared code path (`checkOperatorActiveTask` / `cross-dock-planning-service.assignNext`) that queries `WmsOutboundPutBackTask`. Registered that entity in each suite's own entity list.

## Dedicated PostgreSQL Suite (17 Tests)

Suite: `apps/mercato/src/modules/wms_outbound/services/__tests__/p4-002-postgres.integration.test.ts`
Result: **17/17 PASSED** on canonical Testing PostgreSQL (Supabase pooler).

Covers, at minimum: single-handoff materialization; replay/reconciliation dedup; concurrent materialization race resolved by the DB unique constraint; end-to-end cancellation -> handoff -> task; zero-picked cancellation yields zero task; exact handoff/line/SKU/quantity/TU/warehouse correlation; strict `createdAt` FIFO ignoring CustomerOrder priority/SLA; no-zone semantics; single-active-task guard (no ownership mutation on block); real two-connection concurrency race with exactly one `ASSIGNED` winner and DB-side lock evidence; warehouse isolation; idempotent retry (acquisition and start); illegal/regressive transition rejection with zero mutation; PutBackTask lifecycle does not execute Inventory recovery and does not resolve the P4-001 handoff protection; real rollback proof (write+flush, forced failure before commit, fresh independent read unchanged).

## Mandatory Regressions

| Suite | Result |
|---|---|
| P4-001 dedicated PostgreSQL (`p4-001-postgres.integration.test.ts`) | 18/18 PASSED |
| P3-003 dedicated PostgreSQL race suite (`p3-003-postgres.integration.test.ts`) | 14/14 PASSED |
| P1-005 PickTask generation/ordering/assignment/concurrency | PASSED (shared active-task guard) |
| P1-006 RF picking execution & P3-003 concurrency | PASSED |
| P2-002 CrossDockPickTask planning/assignment | PASSED (shared active-task guard) |
| FND-003 shared task-lock/warehouse-context | PASSED |

Combined P1-005/P1-006/P2-002/FND-003 run: **54/54 PASSED**.

No product behavior in P4-001, P3-003, or the shared PickTask/CrossDockPickTask assignment paths was altered; only each suite's own isolated MikroORM entity registration was extended to reflect the entity their exercised code path now touches.

## Build / Runtime

- `apps/mercato`: `npx tsc --noEmit` — clean, zero errors.
- Root `yarn build` (`build:packages` -> `generate` -> `build:packages` -> `build:app`) — succeeded; Next.js production build compiled and typechecked clean.
- Scanner: `npx expo export --platform web` — succeeded, 277 modules bundled, zero errors.
- Testing runtime rebuilt/restarted from these exact candidate revisions: `mercato-localhost.service` (Mercato, `https://devaxonic-test.info-start.com.pl`) and `scanner-testing.service` (Scanner, `https://scanner.info-start.com.pl`), both confirmed `active (running)` post-restart.

## Rendered UI Acceptance — Zero Route Mocks

Suite: `Devaxonic-scanner/e2e/p4-002-rendered-acceptance.spec.ts`
Result: **4/4 PASSED** against `https://scanner.info-start.com.pl` and `https://devaxonic-test.info-start.com.pl`, with real seeded PostgreSQL fixtures (real users/operators via `bcrypt` + real warehouse assignment, real CustomerOrders, real physical-return handoffs, real PutBackTask rows) and zero route/API mocking.

- **Journey A**: operator enters the Returns module; no zone-selection step is rendered (`returns-no-zone` visible, no zone-card UI present); the older, lower-priority-CO task is assigned ahead of a newer, much-higher-priority-CO task (FIFO by arrival, not business priority); human-operational task number/SKU/quantity are rendered; operator starts the task and PostgreSQL confirms `ASSIGNED -> IN_PROGRESS` with the correct `operator_id`; the untouched higher-priority task remains `CREATED`.
- **Journey B**: an operator with a genuine active `PickTask` (`ASSIGNED`, seeded through the accepted task path) enters the Returns module and receives a clear active-work block; PostgreSQL confirms the one remaining available task is untouched (`CREATED`, no `operator_id`).
- **Journey C**: two independent real Scanner browser sessions/operators enter the Returns module concurrently against the single remaining candidate task; exactly one operator's UI renders the assigned task, the other renders no-task/empty state; PostgreSQL confirms exactly one `ASSIGNED` winner.
- **Journey D**: Mercato Supervisor logs in, opens the Customer Order detail page, and the `putback-tasks-section` read-only table renders the task's number, SKU, and current status (`IN_PROGRESS`, reflecting Journey A). No reassignment/priority control is present — observational only, per invariant #7.

Evidence class: PLAYWRIGHT VERIFIED, not HUMAN VERIFIED.

## Explicit Exclusions Confirmed

- No `priority` or `slaDeadline` field exists on `WmsOutboundPutBackTask`; no zone-selection UI step exists in the returns flow (Journey A).
- Inventory remains physically unrecovered: creating/assigning/starting a PutBackTask does not execute `Inventory PICKED -> AVAILABLE` and does not resolve the P4-001 unresolved physical-return handoff protection (dedicated test #16, `WmsInventoryBalance.onHandQuantity`/`reservedQuantity` both verified unchanged).
- P4-003 destination-location validation, rejection loop, completion, and physical Inventory settlement are not implemented in this item. `LOCATION_VALIDATION`/`COMPLETED` remain schema-only vocabulary, not executable behavior.
- No unrelated work: P3-003 accepted race semantics untouched (regression green); P4-001 cancellation rules unmodified except the already-accepted minimal PutBackTask-materialization integration point; no Return Receipt, Warehouse Transfer, Inbound Putaway, PickWave, or carrier/ERP changes.

## Completion Statement

Item 30/37 (P4-002) implementation is pushed from the exact accepted lineage in both changed product repos; dedicated PostgreSQL acceptance (17/17) including real concurrency and rollback is green; required regressions (P4-001 18/18, P3-003 14/14, P1-005/P1-006/P2-002/FND-003 54/54) are green; native build/typecheck/generate/runtime checks are green; rendered Scanner/Mercato Playwright acceptance (4/4) is PLAYWRIGHT VERIFIED with zero route mocks; this evidence is pushed to `WMS_Outbound/main`.

This is executor `COMPLETE`, not Owner Acceptance. Supervisor independently verifies remote Git/diff/tests/evidence before advancing the Task Catalog count.

Executor does not proceed to P4-003 or any later item.
