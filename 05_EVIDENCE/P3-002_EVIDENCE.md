# P3-002 — Reservation Retention Policy and Automatic Release Timer — Evidence

Date: 2026-09-06 UTC  
Evidence class: REAL POSTGRESQL INTEGRATION and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/p3-002`
- Mercato candidate commit SHA: `84274acacfbfe0119e270ca5bfbcb723e47d7723`
- Accepted P3-001 Mercato base: `9fb32493ed9ec443a18a494aa1a8ec3a1bde6d06`
- Scanner frozen SHA: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` (unmodified and clean)
- P3-001 evidence SHA: `15c3ad937a4e81d7b67ff96409bd0b6a65553864`
- Item: 27/37 — P3-002: Reservation retention policy and automatic release timer.
- Authority: `proces_3_reservation_release.md` (P3 R9–R10), `wymagania_outbound.md` (`FR-P3-05`, `FR-P3-06`), `scenariusze_testowe_outbound.md` (`TC-112`, `TC-113`), `TASK_CATALOG.md` item 27.

## Migration & Schema Provenance

- Applied Migration: `Migration20260906080000_wms_outbound_p3_002_reservation_retention.ts`
- Table `wms_outbound_warehouse_queue_configs` extended:
  - `partial_reservation_retention_policy`: text not null default `'RETAIN'`, restricted to `'RETAIN' | 'AUTO_RELEASE_AFTER_TIME' | 'SUPERVISOR_DECISION'`.
  - `partial_reservation_retention_minutes`: integer not null default `60`.
- Table `wms_outbound_reservation_retentions`:
  - Columns: `id`, `organization_id`, `tenant_id`, `warehouse_id`, `customer_order_id`, `customer_order_line_id`, `outbound_order_id`, `outbound_order_line_id`, `allocation_id`, `sku`, `required_quantity`, `reserved_quantity`, `policy_variant`, `retention_duration_minutes`, `eligible_at`, `due_at`, `status`, `supervisor_decision`, `supervisor_reason`, `decided_by`, `decided_at`, `released_at`, `reservation_release_id`, `idempotency_key`, `created_at`, `updated_at`.
  - Unique Constraint: `(organization_id, tenant_id, outbound_order_line_id)` (`wms_outbound_res_retentions_ool_uq`) ensuring exactly one active retention tracking record per outbound line.
  - Unique Idempotency Constraint: `(organization_id, tenant_id, idempotency_key)` (`wms_outbound_res_retentions_idemp_uq`) ensuring idempotent decision submissions.
  - Query Index: `(organization_id, tenant_id, warehouse_id, policy_variant, status, due_at)` (`wms_outbound_res_retentions_wh_due_idx`) for efficient timer evaluation.

## Dedicated PostgreSQL Suite Mapping (17 Tests)

Suite: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-002-postgres.integration.test.ts`  
Result: **17/17 PASSED** (77.2s) on canonical Testing PostgreSQL (Supabase pooler :6543).

1. **TEST 1 (TC-112 Variant 1 - RETAIN)**: Warehouse policy `RETAIN` keeps partial reservation intact indefinitely during evaluation. Allocation remains `RESERVED` / `SHORT`, inventory reservation unchanged, zero P3 release triggered.
2. **TEST 2 (TC-112 Variant 2 - AUTO_RELEASE_AFTER_TIME timer not expired)**: When `due_at` has not arrived (`NOW() < due_at`), evaluation preserves reservation intact and triggers zero release.
3. **TEST 3 (TC-112 Variant 2 & TC-113 - AUTO_RELEASE_AFTER_TIME timer expired)**: When `due_at <= NOW()`, evaluation releases reservation atomically via P3-001 with reason `AUTOMATIC_POLICY`, restoring ATP, setting line to `CANCELLED`, and marking retention record `RELEASED`.
4. **TEST 4 (TC-113 - Configured retention time independence from Priority and SLA)**: Establishes that retention duration is strictly governed by configured `retention_duration_minutes` regardless of high `priority` (e.g. 999) or tight `sla_deadline`.
5. **TEST 5 (TC-112 Variant 3 - SUPERVISOR_DECISION queued without auto-release)**: Partial reservation routed to `SUPERVISOR_DECISION` leaves reservation intact during automatic evaluation and appears in supervisor decision queue.
6. **TEST 6 (Supervisor Decisive Action - RELEASE)**: Supervisor submits decisive `RELEASE` action; reservation is released atomically via P3-001 with reason `GENERAL_CANCELLATION`, actor `SUPERVISOR`, and retention record settled as `RELEASED`.
7. **TEST 7 (Supervisor Decisive Action - RETAIN)**: Supervisor submits decisive `RETAIN` action; reservation remains intact (`RESERVED`), zero P3 release occurs, retention record settled as `RETAINED`.
8. **TEST 8 (Pre-pick Discriminator Guard - pickedQty > 0 blocks P3 release)**: Partial allocation with confirmed picked quantity (`pickedQty > 0`) rejects supervisor release attempt with explicit error directing to P4 Physical Putback (`TC-043`).
9. **TEST 9 (Pre-pick Discriminator Guard - PICKING status with pickedQty = 0 blocks P3 release)**: Partial allocation with line in `PICKING` status and `pickedQty = 0` rejects supervisor release, auto-release timer, and direct release, directing to P4 Physical Putback (`P3 R8`, `TC-043`, `TC-112`, `TC-113`).
10. **TEST 10 (Pre-pick Discriminator Guard - SHORT_PICKED status with pickedQty = 0 blocks P3 release)**: Partial allocation with line in `SHORT_PICKED` status and `pickedQty = 0` rejects supervisor release, auto-release timer, and direct release, directing to P4 Physical Putback (`P3 R8`, `TC-043`, `TC-112`, `TC-113`).
11. **TEST 10d (Lifecycle & P3 Eligibility - Pre-pick PickTask Cancellation)**: Proves that `generatePickTasks()` and `assignNextPickTask()` leave `OutboundOrderLine` in `ALLOCATED` status with zero formal pick (`pickedQty = 0.000000`). Line remains fully P3-eligible, and P3 release cancels the PickTask and its task lines while releasing the reservation and restoring ATP.
12. **TEST 10e (Lifecycle & P3 Boundary - Pick Confirmation Blocks P3 Release)**: Proves that first formal pick confirmation via `confirmPickLine()` transitions `OutboundOrderLine` to `PICKING` (pickedQty > 0), and thereafter P3 release is strictly blocked across direct invocation and ordering adapter cancellation (`overallRouting: P4_PHYSICAL_PUTBACK`).
13. **TEST 11 (No Physical Recovery - Zero PutBackTask Created)**: Confirms that zero `PutBackTask` or physical recovery records are generated by P3-002 policy release.
14. **TEST 12 (Idempotency - Replay of identical decision)**: Replay of supervisor decision with identical idempotency key returns `replayed: true` with zero duplicate mutations or releases.
15. **TEST 13 (Idempotency - Conflicting key reuse)**: Attempt to reuse idempotency key for conflicting action rejects safely with `Idempotency key collision`.
16. **TEST 14 (Stable Timer Semantics)**: `registerOrGetRetentionRecord` called repeatedly returns existing `due_at` without recomputing or extending duration from current time.
17. **TEST 15 (Concurrency & Deterministic Locking)**: Two concurrent evaluation / decision transactions acquire `PESSIMISTIC_WRITE` locks, serialize deterministically via PostgreSQL lock manager, resulting in exactly one release.
18. **TEST 16 (Transactional Atomicity & Rollback Proof)**: Simulated failure before commit completely rolls back retention record update, allocation release, and inventory unreservation.

## Mandatory Regressions

All regression suites executed and verified on canonical Testing PostgreSQL:

- **P1-005 PickTask Generation & Queue Ordering**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-005-postgres.integration.test.ts` — **10/10 PASSED** (23.6s).
- **P1-006 Outbound RF Picking Execution & Continuation**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-006-postgres.integration.test.ts` — **12/12 PASSED** (68.8s).
- **P3-001 Reservation Release Suite**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-001-reservation-release-postgres.integration.test.ts` — **13/13 PASSED** (76.9s).
- **P1-004 Allocation Hard Reservation Lifecycle (CON-02 Intact)**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-004-postgres.integration.test.ts` — **11/11 PASSED** (46.8s).
- **P1-001 CustomerOrder Lifecycle & Aggregation**: `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-001-postgres.integration.test.ts` — **7/7 PASSED** (12.5s).

## Build, Service & Runtime

- `yarn generate`: Succeeded (471 API routes generated, OpenAPI spec generated, cache purged).
- `yarn typecheck`: Succeeded (exit code 0 with `--max-old-space-size=4096`).
- `yarn build:app`: Succeeded (Turbo build completed, Next.js production bundle created).
- Service: `mercato-localhost.service` restarted and active/running.
- HTTP Health: `https://devaxonic-test.info-start.com.pl/login` returned HTTP 200.

## Real Rendered Playwright UI Proof

Suite: `apps/mercato/src/modules/wms_outbound/__integration__/P3-002-reservation-retention-ui.spec.ts`  
Result: **2/2 PASSED** (38.5s) on real rendered Mercato UI with zero route mocks (`page.route` count = 0):

- **Journey A (Supervisor Decision in Shortages UI)**: Supervisor logs in, navigates to `/backend/shortages`, verifies Section 4 "Partial Reservation Retention Decisions" displays the pending partial reservation with policy badge `SUPERVISOR_DECISION`. Clicks `[Release Reservation]`, confirms in modal (`retention-decision-modal`), modal closes, and database verification confirms Allocation is `RELEASED`, Inventory Reservation cleared, `WmsOutboundReservationRelease` created with `reason: 'GENERAL_CANCELLATION'`, `actor_role: 'SUPERVISOR'`, and retention record settled as `RELEASED`.
- **Journey B (Policy Observability & AUTO_RELEASE_AFTER_TIME / TC-113)**: Seeds an expired partial reservation (`due_at <= NOW()`) on an order with high priority (999) and tight SLA, alongside a future non-expired partial reservation. Triggers policy evaluation via `/api/wms_outbound/retention-policy/evaluate`. Verifies that the expired case is released atomically in PostgreSQL with `AUTOMATIC_POLICY` and actor `SYSTEM_WMS` strictly independent of Priority/SLA (`TC-113`), while the future case remains intact (`RESERVED` and `PENDING`).

## Exclusions Preserved

- No P3-003 RF physical-removal race window.
- Zero `PutBackTask` or physical recovery records created.
- Formally picked quantity (`pickedQty > 0`) or lines in physical picking status (`PICKING`, `SHORT_PICKED`, `PICKED`, `PACKED`, `SHIPPED`) even with `pickedQty = 0` cannot be released by P3 (routes to P4).
- P3-002 owns policy/timer/decision orchestration only; reuses accepted P3-001 pre-pick release mechanism.
- Scanner remains frozen at baseline `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`.
