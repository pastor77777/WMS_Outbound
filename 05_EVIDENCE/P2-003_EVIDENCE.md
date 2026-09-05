# P2-003 evidence — normal crossdock execution

## Tested revisions and environment

- Mercato final pushed SHA: `db0ef671b58ab13c2c0685205fbadcae1e1cf628` (`outbound/p2-003`, verified in sync with `origin/outbound/p2-003`, descends from frozen accepted base `50b27fdd0c9b495ab612ce458bc90e65428ecb93`).
- Scanner final pushed SHA: `2ae72fb00db882fecae659b842e91efed17f949f` (`outbound/p2-003`, verified in sync with `origin/outbound/p2-003`, descends from frozen accepted base `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`).
- Product worktrees: clean (`git status --short` empty on both repositories).
- Narrow build-tooling fixes:
  - `turbo.json`: updated `build` outputs to include `.mercato/next/**` while excluding `.mercato/next/cache/**` and preserving required package `dist/**` artifacts, ensuring reproducible build artifact tracking without bloated caching.
  - `apps/mercato/package.json`: added explicit dependency on `cross-env` because the workspace `build` script directly invokes it.
- Testing migration provenance: `Migration20260905150000_wms_crossdock_execution.ts` (`wms_outbound_cross_dock_placements` table, `active_target_tu_id` FK column on `wms_outbound_cross_dock_pick_tasks`), confirmed already applied in canonical Testing PostgreSQL (`DevAxonic_Platform`, Supabase pooler `aws-1-eu-central-1.pooler.supabase.com`). No manual schema alteration or rollback performed.
- Mercato canonical runtime: `mercato-localhost.service` active/running; MainPID `3367908`; Next child process PID `3367958` listening on port `3009`; `http://localhost:3009/login` returned HTTP `200`; non-empty `.mercato/next/routes-manifest.json` (4.7 KB) and `.mercato/next/required-server-files.json` (9.5 KB) verified.
- Scanner canonical runtime: `scanner-testing.service` active/running; MainPID `3356876`; listening on port `8081` with fresh web export built from current Scanner working copy; `http://localhost:8081` returned HTTP `200`.
- Zero route mocks / network interception: all Playwright journeys executed against the canonical running services with real HTTP calls; direct PostgreSQL access used only for deterministic setup, teardown, and persisted-state reconciliation.

## Gate verification results

- Dedicated P2-003 PostgreSQL suite: `p2-003-crossdock-execution-postgres.integration.test.ts`: **8/8 PASSED** (26.8s) on canonical Testing PostgreSQL.
- P2-002 PostgreSQL regression: `p2-002-crossdock-planning-postgres.integration.test.ts`: **22/22 PASSED** (50.0s).
- P2-001 PostgreSQL regression: `p2-001-crossdock-eligibility-postgres.integration.test.ts`: **10/10 PASSED** (21.5s).
- P1-008 PostgreSQL regression: `p1-008-postgres.integration.test.ts`: **22/22 PASSED** (24.6s).
- P1-008 Scanner Playwright regression: `p1-008-real-tu-identity.spec.ts`: **1/1 PASSED** (6.0s).
- P2-002 Scanner Playwright regression: `p2-002-crossdock-assignment.spec.ts`: **1/1 PASSED** (5.1s).
- Mercato typecheck: `yarn typecheck` in Devaxonic-mercato: **19/19 packages PASSED** (58.4s).
- Real Scanner Playwright fixture: `e2e/p2-003-real-crossdock-execution.spec.ts`: **2/2 PASSED** (11.0s).

## Explicit 18 substantive-behavior mapping

| # | Behavior | Verification Evidence |
|---|---|---|
| 1 | No physical target TU exists before first placement | Dedicated test 1 (`keeps target lazy...`); Journey A pre-placement DB check: `SELECT count(*) FROM wms_outbound_transport_units WHERE outbound_order_id=$1` returned `0`. |
| 2 | First OK placement atomically starts task, line, and order states | Dedicated test 1: task `ASSIGNED -> IN_PROGRESS`, line `CREATED -> PICKING`, order `CREATED -> PACKING_IN_PROGRESS`. |
| 3 | First placement lazily creates/opens a target TU | Dedicated test 1; Journey A response and DB: first placement creates target TU in `CREATED` status with `PackUnit` role. |
| 4 | Valid 1:1 GS1 source inherits TU_NUMBER and SSCC | Dedicated test 2 (`inherits valid 1:1 GS1 source identity`); Journey A UI (`crossdock-target-tu`) and DB verify target inherits source SSCC `123456789012345675`. |
| 5 | 1:1 invalid/non-GS1 source creates new distinct TU identity | Dedicated test 3: target TU number does not match source prefix and is distinct. |
| 6 | n:n topology always creates new target identity and never inherits source identity | Journey B: two-source-task scenario; first placement creates generated `TU##########` with `sscc: null`. |
| 7 | Confirmed placement uses exact decimal quantity and source/task/SKU correlation | Dedicated tests 4–5; Journey A (`3.000000`) and Journey B (`2.000000` + `3.000000`) placement records match exactly. |
| 8 | Replay/duplicate does not double-confirm quantity or duplicate TU contents | Dedicated test 4 (`replay does not double confirm or duplicate placement`): `replayed: true`, placement count remains 1, confirmed quantity unchanged. |
| 9 | Placement cannot exceed remaining planned quantity | Dedicated test 5: attempting quantity exceeding planned quantity is rejected (`exceeds`) without side effect. |
| 10 | Wrong task, source, SKU, tenant, or warehouse rejected without side effects | Dedicated test 5: rejects invalid source TU (`source TU`), wrong SKU (`SKU`), wrong warehouse (`not assigned`), and wrong tenant (`not assigned`); placement count remains 0. |
| 11 | Physical-full seal closes current target while task remains IN_PROGRESS | Dedicated test 6; Journey B UI action `Seal physically full TU`: target moves to `PACKING_SEALED`, task remains `IN_PROGRESS`. |
| 12 | Next placement after physical-full creates/uses a new target under same task | Dedicated test 6; Journey B: subsequent placement creates distinct target TU (`id !== firstId`, status `CREATED`), task remains `IN_PROGRESS`. |
| 13 | Multi-target quantities reconcile exactly to task confirmed quantity | Journey B DB verification: target 1 has `2.000000`, target 2 has `3.000000`, task has `confirmed_qty = 5.000000`, line has `packed_qty = 5.000000`. |
| 14 | Task completion fixes confirmed quantity and is replay-safe | Dedicated test 7; Journey A (`3.000000`) and Journey B (`5.000000`): task moves to `COMPLETED`, replay returns `replayed: true`. |
| 15 | Completing one task does not require sibling task completion | Journey B: sibling task sharing source TU remains untouched while task B completes. |
| 16 | No Allocation is created by P2-003 normal crossdock path | Dedicated test 8; Journey B pre- vs post-execution DB query: `SELECT count(*) FROM wms_outbound_allocations` remains unchanged (`allocationsBefore`). |
| 17 | Accepted P2-002 assignment / one-active-task behavior remains non-regressive | P2-002 dedicated suite: 22/22 PASSED; P2-002 Playwright: 1/1 PASSED with one-active-task rejection verified. |
| 18 | Normal OK path produces no P2-004 exception effects | Dedicated test 8; Journey A and Journey B operate strictly through OK placement, seal, and completion without shortage, damage, cancellation, or PutBack effects. |

## Real UI persisted reconciliation

- **Journey A (1:1 valid GS1 inheritance)**:
  - Scanner operator logged in and requested Crossdock task; received `CD-A-<stamp>` (`3.000000` planned).
  - Pre-placement check proved 0 target TUs existed.
  - Operator confirmed OK placement of `3.000000`; target TU was lazily created with `tuNumber = '123456789012345675'` and `sscc = '123456789012345675'`, matching the valid source GS1 SSCC.
  - Scanner UI displayed the inherited target TU number.
  - Operator completed the task.
  - Persisted DB reconciliation:
    - Task: `status = 'COMPLETED'`, `confirmed_qty = '3.000000'`.
    - Target TU: `status = 'CREATED'`, `tu_number = '123456789012345675'`, `sscc = '123456789012345675'`.
    - Line: `status = 'PACKED'`, `packed_qty = '3.000000'`.
    - Order: `status = 'PACKING_IN_PROGRESS'`.
    - Placements: single placement row with `quantity = '3.000000'`.
    - Allocations: zero allocation increase.

- **Journey B (n:n multi-target continuation with physical-full seal)**:
  - Scanner operator logged in and received `CD-B-<stamp>` (`5.000000` planned) under multi-task source TU context.
  - Confirmed first OK placement of `2.000000`; non-SSCC target TU generated (`TU##########`, `sscc: null`).
  - Operator triggered "Seal physically full TU"; target transitioned to `PACKING_SEALED`; task remained `IN_PROGRESS`.
  - Operator confirmed second OK placement of `3.000000`; new target TU generated (`id !== firstTargetId`, `sscc: null`, status `CREATED`).
  - Operator completed task.
  - Persisted DB reconciliation:
    - Task: `status = 'COMPLETED'`, `confirmed_qty = '5.000000'`, `confirmed_qty <= planned_qty`.
    - Target TUs: exactly two targets; first `PACKING_SEALED`, second `CREATED`, distinct numbers.
    - Placements: two rows (`2.000000` on first target, `3.000000` on second target).
    - Line: `status = 'PACKED'`, `packed_qty = '5.000000'`.
    - Order: `status = 'PACKING_IN_PROGRESS'`.
    - Allocations: count unchanged before vs after.
