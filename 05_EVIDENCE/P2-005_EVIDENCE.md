# P2-005 Evidence — Goods Receipt Correlation Gate & Re-evaluation

## Tested revisions and environment

- Mercato final pushed SHA: `069f02d4c5c9b345b688b838eb685be02206afbd` (`outbound/p2-005`, verified in sync with `origin/outbound/p2-005`, descends from frozen accepted base `9859be5c7dee4fe802d4d00478459a19982eddfe`).
- Scanner frozen accepted SHA: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` (`outbound/p2-004`, clean, untouched; zero modifications made).
- Product worktrees: clean (`git status --short` empty on both repositories).
- Migration/schema provenance:
  - `Migration20260905210000_wms_crossdock_gr_gate.ts`:
    - Column added to `wms_outbound_cross_dock_pick_tasks`: `gr_acceptance_status text not null default 'GR_PENDING'`.
    - Table created: `wms_outbound_cross_dock_gr_results` (`id`, `organization_id`, `tenant_id`, `source_inbound_tu_id`, `settlement_source`, `status`, `version`, `idempotency_key`, `created_at`) with unique index on `(organization_id, tenant_id, idempotency_key)` and performance index on `(organization_id, tenant_id, source_inbound_tu_id)`.
    - Applied via `yarn db:migrate` to canonical Testing PostgreSQL (`DevAxonic_Platform`, Supabase pooler `aws-1-eu-central-1.pooler.supabase.com:6543`). No manual schema changes or rollbacks performed.
- Core services & architecture:
  - `CrossDockGrGateService` (`apps/mercato/src/modules/wms_outbound/services/cross-dock-gr-gate-service.ts`):
    - `correlateGrResult`: correlates incoming GR results by `sourceInboundTU` (UUID, identifier, or SSCC) in tenant/org scope; verifies `settlementSource === 'CROSSDOCK'` (ignores `PUTAWAY` without mutation); updates `grAcceptanceStatus` on all tasks of that source; records idempotent `WmsOutboundCrossDockGrResult`.
    - `evaluateShipmentGrGate`: aggregates contributing sources where `confirmedQty > 0`; evaluates whether all sources are `GR_ACCEPTED`; returns gate readiness (`ready: boolean`), blocking sources list, and detailed breakdown. Returns `{ ready: true, requiresGate: false }` for P1-only shipments with zero crossdock tasks.
  - Precondition hook in `ShipmentPostingService`:
    - Initial `postShipmentToErp` and supervisor `retryPosting`: strictly gated on `evaluateShipmentGrGate`. Rejects with clear message and HTTP 400 before Phase 1 writes if gate is not satisfied.
  - Integration Ingress route:
    - Registered at `/api/wms_outbound/cross-dock/gr-result` with `requireAuth: true` and feature `wms_outbound.edit`.
  - Supervisor rendered surface:
    - Rendered in `apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx`: Section "Goods Receipt Correlation Gate (Crossdock)", showing badge (`GR GATE READY` or `GR BLOCKED`), warning banner listing blocking sources and statuses, and contributing sources breakdown table.
- Mercato canonical runtime: `mercato-localhost.service` active/running; MainPID `3456763`; listening on port `3009`; `https://devaxonic-test.info-start.com.pl/login` returned HTTP `200`; non-empty `.mercato/next/routes-manifest.json` (4.7 KB) and `.mercato/next/required-server-files.json` (9.5 KB) verified.
- Scanner canonical runtime: `scanner-testing.service` active/running; MainPID `3462525`; listening on port `8081`; `https://scanner.info-start.com.pl/` returned HTTP `200`.
- Zero route mocks / network interception: all Playwright journeys executed against the canonical running services with real HTTP calls via live Cloudflare endpoints; direct PostgreSQL client access used only for deterministic fixture preparation, background ingress triggers, and persisted-state reconciliation.

## Gate verification results

- Dedicated P2-005 PostgreSQL suite: `p2-005-crossdock-gr-gate-postgres.integration.test.ts`: **19/19 PASSED** (42.5s) on canonical Testing PostgreSQL (Supabase pooler).
- P1-014 ERP posting regression: `p1-014-erp-posting-postgres.integration.test.ts`: **18/18 PASSED** (34.3s).
- P2-004 crossdock recovery regression: `p2-004-crossdock-recovery-postgres.integration.test.ts`: **16/16 PASSED** (48.7s).
- P2-003 crossdock execution regression: `p2-003-crossdock-execution-postgres.integration.test.ts`: **8/8 PASSED** (26.1s).
- Mercato full build/generate contract:
  - `yarn generate`: **PASSED**, registered `/api/wms_outbound/cross-dock/gr-result`.
  - `yarn build:packages`: **19/19 packages PASSED**.
  - `yarn typecheck`: **19/19 packages PASSED** (2m22s).
  - `yarn build:app`: **PASSED** (3m7s), compiled Next.js production build cleanly.
- Dedicated P2-005 real Mercato Playwright suite: `apps/mercato/src/modules/wms_outbound/__integration__/P2-005-crossdock-gr-gate-ui.spec.ts`: **3/3 PASSED** (1.0m, zero retries):
  - Journey A: Two-source crossdock gate blocks then unblocks upon GR_ACCEPTED (P2 R25-R26, R31-R33, TC-028, TC-038) — PASSED (22.3s).
  - Journey B: POSTING_ERROR re-evaluation without auto-retry, Supervisor retry succeeds (P2 R35-R36, TC-030, TC-031) — PASSED (29.1s).
  - Journey C: Non-crossdock settlement source (PUTAWAY) has zero effect on crossdock gate (TC-067, INT-02) — PASSED (2.9s).

## Explicit substantive behavior mapping

| # | Behavior | Verification Evidence |
|---|---|---|
| 1 | New/unresolved crossdock source is non-accepted (`GR_PENDING`), never implicitly accepted | Dedicated test 1 (`unresolved crossdock source defaults to GR_PENDING and blocks gate`); migration default `GR_PENDING`; Playwright Journey A. |
| 2 | Valid `CROSSDOCK + GR_ACCEPTED` correlates by `sourceInboundTU` without task id and updates every task of that source | Dedicated test 2 (`correlateGrResult updates every task for source TU`); verified multiple tasks for same source TU all updated to `GR_ACCEPTED`. |
| 3 | Valid `CROSSDOCK + GR_REJECTED` updates every task of that source and does not create Shipment `POSTING_ERROR` | Dedicated test 3 (`correlateGrResult with GR_REJECTED sets tasks to GR_REJECTED without mutating shipment status`); shipment remains in `LABEL_GENERATED`. |
| 4 | Later `GR_ACCEPTED` after `GR_REJECTED` replaces the gate status normally and requires no repair path | Dedicated test 4 (`later GR_ACCEPTED replaces GR_REJECTED normally`); gate evaluates to `ready: true`, blocking sources list cleared. |
| 5 | Duplicate/replayed identical GR result is idempotent and creates no duplicate durable effect/fact | Dedicated test 5 (`duplicate/replayed identical GR result is idempotent`); returns `replayed: true`, `wms_outbound_cross_dock_gr_results` row count unchanged. |
| 6 | Unknown `sourceInboundTU` is rejected with zero task/Shipment mutation | Dedicated test 6 (`unknown sourceInboundTU is rejected with zero task/shipment mutation`); returns `matched: false, reason: 'UNKNOWN_SOURCE_INBOUND_TU'`, 0 tasks updated. |
| 7 | `GR_SETTLEMENT_SOURCE = PUTAWAY` for the same TU is rejected/ignored with zero crossdock-status mutation | Dedicated test 7; Playwright Journey C; returns `matched: false, reason: 'IGNORED_SETTLEMENT_SOURCE'`, task `grAcceptanceStatus` stays `GR_ACCEPTED`. |
| 8 | Settlement/message version cannot be used as a substitute for `GR_SETTLEMENT_SOURCE` | Dedicated test 8 (`version cannot substitute for settlementSource`); high version with non-CROSSDOCK rejected with zero mutation. |
| 9 | One Shipment with two contributing sources remains blocked when only one is `GR_ACCEPTED` | Dedicated test 9; Playwright Journey A (`source A GR_ACCEPTED, source B GR_PENDING -> evaluateShipmentGrGate returns ready: false, blocking: [source B]`). |
| 10 | One Shipment with a `GR_REJECTED` contributing source remains outside `POSTING_PENDING` and creates zero P1-014 posting side effects | Dedicated test 10 (`GR_REJECTED contributing source blocks posting attempt before Phase 1 writes`); rejects with HTTP 400, 0 postings, 0 attempts. |
| 11 | The same Shipment becomes gate-ready when all distinct contributing sources are `GR_ACCEPTED` | Dedicated test 11; Playwright Journey A (`delivering GR_ACCEPTED for source B causes evaluateShipmentGrGate to return ready: true`). |
| 12 | One source TU contributing to two Shipments is updated by one GR result, while each Shipment evaluates its own complete source set separately | Dedicated test 12 (`one source TU contributing to two shipments`); Shipment 1 (sources 1+2) blocked, Shipment 2 (source 1 only) unblocked. |
| 13 | Residual/PUTAWAY settlement does not block or regress a gate already satisfied by CROSSDOCK acceptance | Dedicated test 13 (`residual/PUTAWAY settlement does not regress gate satisfied by CROSSDOCK`); gate remains `ready: true`. |
| 14 | P1-only Shipment with no crossdock contribution retains accepted P1-014 posting behavior | Dedicated test 14; P1-014 regression suite 18/18 PASSED; `evaluateShipmentGrGate` returns `{ ready: true, requiresGate: false }`. |
| 15 | Gate rejection occurs before P1-014 Phase 1 side effects: zero posting row, attempt, outbox/retry fact, state transition, and ERP adapter call | Dedicated test 15; Playwright Journey A (`attempt posting while gate blocked throws before Phase 1 writes: zero rows in postings, zero attempt rows, zero status changes`). |
| 16 | Shipment already `POSTING_ERROR` for a real ERP rejection re-evaluates on later GR status changes without auto-retry; once gate-ready, existing P1-014 Supervisor retry can proceed | Dedicated test 16; Playwright Journey B (`shipment in POSTING_ERROR; GR_REJECTED updates gate to unready without auto-retry; later GR_ACCEPTED makes gate ready without auto-retry; supervisor retry action succeeds`). |
| 17 | Org/tenant isolation: a GR result in one scope cannot update tasks or gate state in another scope even with matching/colliding identifiers | Dedicated test 17 (`cross-tenant/org isolation`); GR result for tenant B does not update tenant A tasks with identical TU identifier. |
| 18 | Supervisor read model reports exact contributing source statuses and blocking set without changing business state | Dedicated test 18; Playwright Journey A and B (`getShipmentDetail` returns `crossDockGrGate` with `sources`, `blockingSources`, `ready`; rendered on UI). |

## Correlation and quantity conservation preservation

- **Correlation preservation**:
  - `correlateGrResult` resolves `sourceInboundTU` through `wms_transport_units` (or direct UUID) within `(organizationId, tenantId)` scope.
  - Updates `wms_outbound_cross_dock_pick_tasks.gr_acceptance_status` on all tasks matching `source_inbound_tu_id`.
  - Durably records the correlation fact in `wms_outbound_cross_dock_gr_results` with idempotency guard.
  - Dedicated test 19 explicitly confirms:
    - `WmsOutboundCrossDockFinalization` declared, confirmed, damaged, and residual quantities remain exact (`confirmed + damaged + residual == declared`).
    - `WmsOutboundPhysicalReturnHandoff` records remain exact and intact.
    - Zero interference between GR gate evaluation and P2-004 exception handling.

## Real UI persisted reconciliation

- **Journey A (Two-source gate blocks then unblocks upon GR_ACCEPTED, TC-028, TC-038)**:
  - Shipment prepared in `LABEL_GENERATED` with 2 distinct source Inbound TUs (`TU-INB-A1-*`, `TU-INB-A2-*`).
  - Source A initialized with `GR_ACCEPTED`; Source B with `GR_PENDING`.
  - Supervisor views shipment detail at `https://devaxonic-test.info-start.com.pl/backend/shipments/[id]`:
    - Gate status displays `GR BLOCKED` badge.
    - Blocking banner appears: "ERP posting is blocked by Goods Receipt correlation gate. Unaccepted source TUs: TU-INB-A2-* (GR_PENDING)".
    - Sources table displays Source A as `GR_ACCEPTED` and Source B as `GR_PENDING`.
  - Supervisor clicks "Post to ERP (Step 11A)":
    - Live backend rejects request: `Shipment ERP posting is blocked by Goods Receipt correlation gate: contributing source TU(s) TU-INB-A2-* (GR_PENDING) are not accepted.`
    - Error displayed in `[data-testid="shipment-action-error"]`.
    - Shipment status remains `LABEL_GENERATED`.
    - DB proof: exactly 0 posting records (`count = 0`), zero attempt records, zero outbox side effects.
  - Valid CROSSDOCK `GR_ACCEPTED` delivered for Source B via `/api/wms_outbound/cross-dock/gr-result`.
  - Page reloaded: gate displays `GR GATE READY` badge; blocking banner is removed; Source B shows `GR_ACCEPTED`.
  - Supervisor clicks "Post to ERP (Step 11A)":
    - ERP posting succeeds cleanly via Testing ERP adapter.
    - Shipment status updates to `POSTED`.
    - DB proof: exactly 1 posting record (`status = 'ACCEPTED'`), exactly 1 attempt record (`outcome = 'ACCEPTED'`).
- **Journey B (POSTING_ERROR re-evaluation without auto-retry, Supervisor retry succeeds, TC-030, TC-031)**:
  - Shipment prepared with all sources `GR_ACCEPTED`.
  - Initial ERP posting triggered with configured rejection scenario -> status transitions to `POSTING_ERROR`.
  - Subsequent `GR_REJECTED` delivered for contributing source TU:
    - Gate status re-evaluates to `GR BLOCKED`.
    - Shipment remains in `POSTING_ERROR` — no automatic retry occurs.
    - DB proof: exactly 1 attempt record (the original rejection).
  - Subsequent `GR_ACCEPTED` delivered for the same source:
    - Gate status re-evaluates to `GR GATE READY`.
    - Shipment still remains in `POSTING_ERROR` — no automatic retry occurs.
    - DB proof: still exactly 1 attempt record.
  - Supervisor clicks "Supervisor Retry Posting (R37)":
    - Precondition passes; retry proceeds and succeeds.
    - Shipment transitions to `POSTED`.
    - DB proof: exactly 2 attempt records (attempt 1 REJECTED, attempt 2 ACCEPTED).
- **Journey C (PUTAWAY settlement source has zero effect, TC-067, INT-02)**:
  - Source TU in `GR_ACCEPTED` receives a subsequent GR result with `settlementSource: 'PUTAWAY'` and `status: 'GR_REJECTED'`.
  - Ingress returns `ok: true`, `matched: false`, `reason: 'IGNORED_SETTLEMENT_SOURCE'`.
  - Task `grAcceptanceStatus` remains strictly `GR_ACCEPTED`.
  - Shipment gate remains `GR GATE READY`.

## Inbound settlement boundary

- Boundaries `INT-02` and `INT-03` are preserved:
  - Outbound provides and retains crossdock finalization quantities and source TU correlation facts (`wms_outbound_cross_dock_finalizations`).
  - Outbound consumes the resulting GR acceptance/rejection outcome solely for gate evaluation.
  - Goods Receipt retries, alerts, and message transport remain strictly Inbound/integration responsibility.
  - No Inbound internal logic or retry infrastructure was modified or moved.

## Clean worktree verification

- Mercato:
  - Branch: `outbound/p2-005`
  - Commit: `069f02d4c5c9b345b688b838eb685be02206afbd`
  - Push: up to date with `origin/outbound/p2-005`
  - `git status --short`: clean (0 modified, 0 untracked).
- Scanner:
  - Branch: `outbound/p2-004`
  - Commit: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
  - Push: up to date with `origin/outbound/p2-004`
  - `git status --short`: clean (0 modified, 0 untracked; completely frozen and untouched).
- Verification Status: **PLAYWRIGHT VERIFIED / Awaiting Supervisor Verification**.
