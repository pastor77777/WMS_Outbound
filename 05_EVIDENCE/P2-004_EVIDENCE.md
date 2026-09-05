# P2-004 Evidence — Crossdock Shortage, Damage, Empty-TU and Cancellation Recovery

## Tested revisions and environment

- Mercato final pushed SHA: `9859be5c7dee4fe802d4d00478459a19982eddfe` (`outbound/p2-004`, verified in sync with `origin/outbound/p2-004`, descends from frozen accepted base `db0ef671b58ab13c2c0685205fbadcae1e1cf628`).
- Scanner final pushed SHA: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` (`outbound/p2-004`, verified in sync with `origin/outbound/p2-004`, descends from frozen accepted base `2ae72fb00db882fecae659b842e91efed17f949f`).
- Product worktrees: clean (`git status --short` empty on both repositories).
- Migration/schema provenance:
  - `Migration20260905180000_wms_crossdock_exceptions.ts`:
    - Columns added to `wms_outbound_cross_dock_pick_tasks`: `damaged_qty`, `shortage_qty`, `exception_reason`, `supervisor_decision`.
    - Table created: `wms_outbound_cross_dock_finalizations` (with idempotency and source TU indices).
    - Applied via `yarn db:migrate` to canonical Testing PostgreSQL (`DevAxonic_Platform`, Supabase pooler `aws-1-eu-central-1.pooler.supabase.com:6543`). No manual schema changes or rollbacks performed.
- Supervisor role and state authorization architecture (P2-004 corrective):
  - Packer / Scanner execution role boundary:
    - Removed `WAIT` / `CANCEL` / `ALLOW_PARTIAL` decision buttons from Packer `CrossdockTaskScreen` in Scanner.
    - Scanner displays informational waiting banner (`testID="crossdock-escalation-waiting"`): "Shortage confirmed. Warehouse Supervisor decision required (allowPartialShipment=false). Waiting for Supervisor action in Mercato."
    - Removed `case 'supervisor_decision'` from ordinary Packer crossdock execute route (`/api/wms_outbound/cross-dock/execute`, which carries only `wms_outbound.view`).
  - Dedicated Warehouse Supervisor server surface:
    - Endpoint: `/api/wms_outbound/supervisor/cross-dock-decision` (POST).
    - Authorization metadata: `requireAuth: true`, `requireFeatures: ['wms_outbound.manage_orders']`.
    - Authenticated canonical identity: resolves `supervisorId` from `auth.sub || auth.userId` and validates warehouse access.
    - Low-privilege / Packer direct HTTP attempts without `wms_outbound.manage_orders` are rejected with HTTP 403 and create zero side effects.
  - Legal decision state enforcement:
    - `resolveSupervisorDecision` strictly requires `task.status === 'SHORT_PICKED'` with `exceptionReason === 'SHORTAGE_ESCALATED'`.
    - Invocations outside this state (e.g. `ASSIGNED`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`) are rejected with error and create zero side effects.
    - First accepted decision is immutable and replay-safe (replaying identical decision returns `replayed: true` without fact duplication).
  - Exception-state guards:
    - `reportUnexpectedSku` is strictly guarded against non-executable or finalized tasks (`task.status !== 'ASSIGNED' && task.status !== 'IN_PROGRESS'`), preventing late discrepancy facts.
    - `complete` is strictly guarded against non-executable states (`task.status !== 'IN_PROGRESS'`).
  - Rendered Supervisor surface in Mercato:
    - Extended `/api/wms_outbound/supervisor/shortages` to query and return `crossDockCases` (`status: 'SHORT_PICKED'`, `exceptionReason: 'SHORTAGE_ESCALATED'`).
    - Extended `apps/mercato/src/modules/wms_outbound/backend/shortages/page.tsx` with Section 3: "3. Crossdock Shortage Escalations" and modal actions (`Wait (PutBack)`, `Cancel line`, `Allow partial`).
- Mercato canonical runtime: `mercato-localhost.service` active/running; MainPID `3418492`; child process PID `3418554`; listening on port `3009`; `https://devaxonic-test.info-start.com.pl/login` returned HTTP `200`; non-empty `.mercato/next/routes-manifest.json` (4.7 KB) and `.mercato/next/required-server-files.json` (9.6 KB) verified.
- Scanner canonical runtime: `scanner-testing.service` active/running; MainPID `3421125`; listening on port `8081` with fresh web export built from exact final Scanner SHA `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`; `https://scanner.info-start.com.pl/` returned HTTP `200`.
- Zero route mocks / network interception: all Playwright journeys executed against the canonical running services with real HTTP calls via live Cloudflare endpoints; direct PostgreSQL client access used only for deterministic setup, teardown, and persisted-state reconciliation.

## Gate verification results

- Dedicated P2-004 PostgreSQL suite: `p2-004-crossdock-recovery-postgres.integration.test.ts`: **16/16 PASSED** (62.2s) on canonical Testing PostgreSQL (including non-SHORT_PICKED decision rejection, post-final unexpected SKU rejection, decision immutability replay, and supervisor route RBAC metadata).
- P2-003 PostgreSQL regression: `p2-003-crossdock-execution-postgres.integration.test.ts`: **8/8 PASSED** (27.8s).
- P2-002 PostgreSQL regression: `p2-002-crossdock-planning-postgres.integration.test.ts`: **22/22 PASSED** (50.2s).
- P2-001 PostgreSQL regression: `p2-001-crossdock-eligibility-postgres.integration.test.ts`: **10/10 PASSED** (21.1s).
- Mercato full build/generate contract:
  - `yarn generate`: **PASSED** (24.8s), generated all registries and registered `/wms_outbound/supervisor/cross-dock-decision`.
  - `yarn build:packages`: **19/19 packages PASSED** (26.8s).
  - `yarn typecheck`: **19/19 packages PASSED** (2m45s).
  - `yarn build:app`: **PASSED** (3m4s), compiled Next.js production build cleanly.
- Dedicated P2-004 Scanner real Playwright suite: `e2e/p2-004-real-crossdock-recovery.spec.ts`: **4/4 PASSED** (27.1s):
  - Journey A: Shortage, DAMAGED, and unexpected SKU auto-packing (TC-024, TC-025, TC-026, TC-081) - PASSED.
  - Journey B: Role-separated shortage escalation (Packer waiting banner -> 403 unauthorized direct attempt -> Rendered Mercato Supervisor `/backend/shortages` decision -> DB reconciliation) - PASSED.
  - Journey C: Empty source TU before picking (TC-021, FR-P2-11, R19, R20) - PASSED.
  - Journey D: In-progress cancellation guard & completion (FR-P5-15, R9, R37) - PASSED.
- P2-003 Scanner real Playwright regression: `e2e/p2-003-real-crossdock-execution.spec.ts`: **2/2 PASSED**.
- P2-002 Scanner real Playwright regression: `e2e/p2-002-crossdock-assignment.spec.ts`: **1/1 PASSED**.
- P1-008 Scanner real Playwright regression: `e2e/p1-008-real-tu-identity.spec.ts`: **1/1 PASSED**.

## Explicit substantive behavior mapping

| # | Behavior | Verification Evidence |
|---|---|---|
| 1 | Shortage with `allowPartialShipment=true` auto-packs confirmed quantity, backorders customer line, records shortage discrepancy | Dedicated test 1 (`shortage with allowPartialShipment=true automatically packs...`); Playwright Journey A (`AUTOMATIC_PACKED`, line `PACKED` with `6.000000`, col `BACKORDERED`, discrepancy `MISSING` with `2.000000`). |
| 2 | `DAMAGED` quantity separated from confirmed OK stock, routed to QC location | Dedicated test 2; Playwright Journey A (`crossdock-damaged-btn` records `2.000000`, target TU contains only OK stock `6.000000`, `DAMAGED` discrepancy recorded with `qcLocationId = 'QC'`). |
| 3 | Shortage with `allowPartialShipment=false`: WAIT decision triggers physical return handoff, clears active target, cancels empty target TU, cancels order | Dedicated test 3; Playwright Journey B (`SUPERVISOR_ESCALATION_REQUIRED`, supervisor chooses `WAIT`, task `CANCELLED`, `active_target_tu_id = null`, line `CANCELLED`, col `BACKORDERED`, handoff created for `6.000000`, target TU `CANCELLED` per R34, order `CANCELLED` per R37). |
| 4 | Shortage with `allowPartialShipment=false`: CANCEL full decision cancels line and customer line | Dedicated test 4 (`CANCEL decision with full cancel`: col `CANCELLED`, line `CANCELLED`, physical return handoff created for confirmed units). |
| 5 | Shortage with `allowPartialShipment=false`: CANCEL corrected decision rescales line and customer line to confirmed quantity | Dedicated test 5 (`CANCEL decision with correction to confirmedQty`: col `orderedQuantity` and line `requiredQty` updated to confirmed, line `PACKED`, col stays `CANCELLED` remainder). |
| 6 | Shortage with `allowPartialShipment=false`: ALLOW_PARTIAL decision overrides policy, packs confirmed quantity, backorders remainder | Dedicated test 6; service resolves `ALLOW_PARTIAL`, order policy updated, line `PACKED`, task `COMPLETED`. |
| 7 | Unexpected SKU routed to QC without affecting target TU or planned quantity | Dedicated test 7; Playwright Journey A (`crossdock-unexpected-sku-btn`, discrepancy `UNEXPECTED_SKU` recorded with `qcLocationId = 'QC'`, zero impact on target TU contents). |
| 8 | Empty source TU before picking cancels task/lines, backorders customer order line, marks source TU `LOST` | Dedicated test 8; Playwright Journey C (`crossdock-empty-source-btn`, task `CANCELLED` with `exception_reason = 'EMPTY_SOURCE_TU'`, source TU `processStatus = 'LOST'`, 0 target TUs created). |
| 9 | General order cancellation blocked while task is `IN_PROGRESS` (P2 R9, FR-P5-15) | Dedicated test 9 (`orderingAdapter.requestCancellation` returns `eligible: false`, `blockingReason` cites P2 R9); Playwright Journey D (cancel attempt during `IN_PROGRESS` rejected with 422 and R9 feedback). |
| 10 | Exact quantity conservation: confirmed + damaged + residual = declared source quantity | Dedicated test 10; Playwright Journey A (`declared (10.000000) = confirmed (6.000000) + damaged (2.000000) + residual (2.000000)` persisted in `wms_outbound_cross_dock_finalizations`). |
| 11 | Cancellation and recovery never leave orphan active target TU or orphan active task | Dedicated test 11; Playwright Journeys B, C, D (verified via database queries: `active_target_tu_id` is null on cancelled tasks, target TUs are cancelled or properly packed, zero hanging locks). |
| 12 | Idempotent replay of exception operations does not duplicate facts | Dedicated test 12 (`replayed: true` returned on duplicate damage, unexpected SKU, empty source TU, complete, decision; placement and discrepancy counts remain unchanged). |
| 13 | Preservation of accepted P2-003 normal OK path | P2-003 dedicated suite 8/8 PASSED; P2-003 Playwright 2/2 PASSED; normal OK execution remains completely non-regressive. |
| 14 | Inbound settlement handoff boundary respected | Outbound writes finalization facts (`WmsOutboundCrossDockFinalization`, `WmsOutboundPhysicalReturnHandoff`) and updates source TU status (`CROSS_DOCKED`, `IN_CROSS_DOCK`, or `LOST`) without altering Inbound internal GR logic. |

## Quantity conservation demonstration

- **Scenario A (Partial packing with damage and shortage)**:
  - Declared ASN source quantity: `10.000000`
  - Confirmed OK placement: `6.000000`
  - Damaged quantity (routed to QC): `2.000000`
  - Residual / shortage quantity (remains on source TU / backordered): `2.000000`
  - Reconciled in `wms_outbound_cross_dock_finalizations`:
    - `declared_qty`: `10.000000`
    - `confirmed_qty`: `6.000000`
    - `damaged_qty`: `2.000000`
    - `residual_qty`: `2.000000`
    - `status`: `TRANSIT_PUTAWAY` (residual `2.000000` available for inbound putaway settlement)
    - Equation: `6.000000 + 2.000000 + 2.000000 == 10.000000` (exact match).

- **Scenario B (Full shortage with allowPartialShipment=false and WAIT)**:
  - Declared source quantity: `10.000000`
  - Confirmed OK placement: `6.000000`
  - Shortage: `4.000000`
  - Supervisor WAIT decision triggers PutBack of confirmed `6.000000` into `wms_outbound_physical_return_handoffs`.
  - Empty target TU cancelled (R34); OutboundOrder cancelled (R37).
  - Source TU finalization: `declared_qty = 10.000000`, `confirmed_qty = 0.000000`, `residual_qty = 10.000000`.

- **Scenario C (Empty source TU)**:
  - Declared source quantity: `10.000000`
  - Source TU inspected empty before picking: confirmed `0.000000`, damaged `0.000000`, residual `0.000000`.
  - Source TU status marked `LOST`.
  - Target TUs created: 0.

## Real UI persisted reconciliation

- **Journey A (Shortage, DAMAGED, unexpected SKU, allowPartialShipment=true)**:
  - Scanner operator logged in and received task `CD-A-<stamp>` (`10.000000` planned).
  - Operator reported unexpected SKU `UNEXPECTED-SKU-X` -> UI showed feedback, packing discrepancy created.
  - Operator confirmed `6.000000` OK placement -> Target TU created, confirmed `6.000000`.
  - Operator recorded `2.000000` DAMAGED -> UI updated damaged qty to `2.000000`, QC discrepancy created.
  - Operator clicked Complete -> UI prompted R11 re-check requirement.
  - Operator confirmed shortage -> Task completed with `AUTOMATIC_PACKED`.
  - Persisted DB reconciliation:
    - Task: `status = 'COMPLETED'`, `confirmed_qty = 6.000000`, `damaged_qty = 2.000000`, `shortage_qty = 2.000000`.
    - Line: `status = 'PACKED'`, `packed_qty = 6.000000`.
    - Customer line: `status = 'BACKORDERED'`.
    - Target TU: contains exactly `6.000000` OK units.
    - Finalization: `confirmed (6) + damaged (2) + residual (2) == declared (10)`.

- **Journey B (Shortage with allowPartialShipment=false, WAIT & PutBack)**:
  - Scanner operator placed `6.000000` OK, then confirmed shortage on complete.
  - UI presented supervisor escalation requirement.
  - Operator selected "Wait (PutBack)".
  - Persisted DB reconciliation:
    - Task: `status = 'CANCELLED'`, `active_target_tu_id = null`.
    - Line: `status = 'CANCELLED'`.
    - Customer line: `status = 'BACKORDERED'`.
    - Target TU: `status = 'CANCELLED'`.
    - OutboundOrder: `status = 'CANCELLED'`.
    - Physical return handoff: 1 record with `quantity = 6.000000`.
    - Zero orphan active targets or tasks.

- **Journey C (Empty source TU before picking)**:
  - Scanner operator received `ASSIGNED` task and clicked "Report empty source TU".
  - UI displayed feedback: "Source TU reported empty and marked LOST. Task cancelled."
  - Persisted DB reconciliation:
    - Task: `status = 'CANCELLED'`, `exception_reason = 'EMPTY_SOURCE_TU'`.
    - Line: `status = 'CANCELLED'`.
    - Customer line: `status = 'BACKORDERED'`.
    - Source TU: `process_status = 'LOST'`.
    - Target TUs: exactly 0 created.

- **Journey D (In-progress cancellation guard & completion)**:
  - Scanner operator placed `3.000000` units; task moved to `IN_PROGRESS`.
  - Operator attempted "Cancel task" -> rejected with P2 R9 message: "General cancellation is rejected while CrossDockPickTask is IN_PROGRESS (R9). Item is currently being sorted; retry after PACKED."
  - Operator completed task normally per R9 rules.
  - Persisted DB reconciliation:
    - Task: `status = 'COMPLETED'`, `confirmed_qty = 3.000000`.
    - Line: `status = 'PACKED'`, `packed_qty = 3.000000`.
    - Target TU: `status = 'CREATED'`.
    - Zero orphan active targets or tasks.
