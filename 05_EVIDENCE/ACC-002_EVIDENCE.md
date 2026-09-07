# ACC-002 — Playwright Standard Fulfillment + P1 Exception Journeys — Evidence

Date: 2026-09-07 UTC
Evidence class: REAL UI / PLAYWRIGHT (Mercato + Scanner, canonical Testing runtime, canonical Testing PostgreSQL). Automated browser proof is `PLAYWRIGHT VERIFIED`, not `HUMAN VERIFIED`. Not Supervisor FINAL PASS, not Owner Accepted.

## Exact revisions and scope

- Mercato batch branch: `outbound/acc-001-003-batch`.
- Mercato ACC-001 checkpoint: `7480f1d707be88ac70ccd2c8ef04b3a56eda760e`.
- Mercato ACC-002 checkpoint commit SHA: `bc80c989eea2f6b468530a9bd6cd814c8dc35646` (test-only diff; no product code touched; Mercato typecheck stayed clean throughout, so the served runtime did not need a rebuild for these test-file-only changes).
- Scanner batch branch: `outbound/acc-001-003-batch`, base `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.
- Scanner ACC-002 checkpoint commit SHA: `fbd4f45915d2cfbb5280c3df7f0a5b8bcf32949e` (test-only diff; Scanner remains functionally frozen — no application code changed, so the already-accepted Scanner web build continued serving unchanged).
- Item: 35/37 — ACC-002: Playwright Standard Fulfillment + P1 exception journeys.
- Execution guide: `06_AGENT_GUIDES/ACC-001_003_BATCH_EXECUTION.md`.

## Runtime provenance

- `mercato-localhost.service` built and restarted from Mercato checkpoint `7480f1d707be` via the durable `scripts/probe-mercato-testing-build.sh` probe (`BUILD_OK`, fresh non-empty `routes-manifest.json`/`required-server-files.json`); confirmed active, `MainPID` cwd `/home/ubuntu/git/Devaxonic-mercato`, port 3009 owned by the canonical Next runtime, `/login` HTTP 200.
- `scanner-testing.service` started (already built/frozen at the accepted head, no rebuild needed); confirmed active, `MainPID` cwd `/home/ubuntu/git/Devaxonic-scanner`, port 8081 HTTP 200, serving `a2759a293472`.
- Canonical Testing PostgreSQL (`aws-1-eu-central-1.pooler.supabase.com`) confirmed non-loopback before every DB-backed command.

## Journey-to-spec mapping (Evidence Standard journeys 1–14)

| # | Journey | Spec(s) | Result |
|---|---|---|---|
| 1 | Standard full fulfillment end to end | Composite proof across the already-accepted P1-001→P1-016 chain (intake → ATP → planning → picking → pack → group → carrier → label → ERP post → manifest → settlement); `P1-001` test 1 anchors intake→detail→HOLD/RELEASE | PASS |
| 2 | Zero ATP / backorder / replanning | `P1-002-atp-reservations-ui.spec.ts` Flow A | PASS |
| 3 | Partial allowed vs partial forbidden | `P1-011-shipment-grouping-ui.spec.ts` Journey A (forbidden, waits) / Journey B (allowed, promotes); `P1-007` outcomes 1/6 | PASS |
| 4 | Multi-zone picking and Picking TU continuation | Scanner `p1-006-real-scanner-picking.spec.ts` (Zero Route Mocks) | PASS |
| 5 | Short pick auto-reallocation and Supervisor decision path | `P1-007-shortages-supervisor-ui.spec.ts` (6 outcomes); Scanner `p1-007-real-scanner-short-pick.spec.ts` | PASS |
| 6 | Direct pack | `P1-010-packer-workstation-ui.spec.ts` Journey 1; Scanner `p1-009-real-scanner-direct-pack.spec.ts` Journey 1 | PASS |
| 7 | Repack/consolidation and packing discrepancy | `P1-010-packer-workstation-ui.spec.ts` Journeys 2–6 | PASS |
| 8 | Automatic carrier selection and manual fallback | `P1-012-carrier-selection-ui.spec.ts` Journeys 1–5 | PASS |
| 9 | WMS label generation | `P1-013-label-generation-ui.spec.ts` Journeys 1–4 | PASS |
| 10 | ERP posting retry/error/recovery | `P1-014-erp-posting-ui.spec.ts` Journeys A–E | PASS |
| 11 | Manifest close, handover and confirmation boundaries | `P1-015-manifest-dispatch-ui.spec.ts` Journeys A–F | PASS |
| 12 | General cancellation before pick | `P3-001-reservation-release-ui.spec.ts` Journey A | PASS |
| 13 | Cancellation after physical pick → Physical Putback (routing decision only; FR-P4 execution itself belongs to ACC-003) | `P4-001-post-pick-cancellation-ui.spec.ts` Journey C | PASS |
| 14 | Cancellation after packing before the allowed boundary | `P4-001-post-pick-cancellation-ui.spec.ts` Journey A (allowed) + Journey B (boundary enforced) | PASS |

Partial/multi-shipment exceptions: `P1-011` Journeys A/B/C; `P1-003` Flows A/B/C; `P1-007` all 6 outcomes.

## Exact commands and results (final checkpoint)

All commands sourced the canonical Testing DB env in the same shell; Mercato runs used `PLAYWRIGHT_TEST_BASE_URL=http://localhost:3009` and `TEST_SUPERVISOR_PASSWORD=<designated Testing password, not printed>`.

```
npx playwright test P1-001 P1-002 P1-003 P1-012 P1-013 --config .ai/qa/tests/playwright.config.ts --workers=1 --retries=0
# 19 passed

npx playwright test P1-007 P1-010 P1-011 --config .ai/qa/tests/playwright.config.ts --workers=1 --retries=0
# 15 passed

npx playwright test P1-014 P1-015 P1-016 P4-001 --config .ai/qa/tests/playwright.config.ts --workers=1 --retries=0
# 20 passed

npx playwright test P3-001 --config .ai/qa/tests/playwright.config.ts --workers=1 --retries=0
# 3 passed
```

Mercato total: **57/57 passed**, zero failures.

```
cd Devaxonic-scanner
npx playwright test e2e/p1-005-real-scanner-assignment.spec.ts e2e/p1-006-real-scanner-picking.spec.ts \
  e2e/p1-006-retry-key.spec.ts e2e/p1-007-real-scanner-short-pick.spec.ts \
  e2e/p1-009-real-scanner-direct-pack.spec.ts --workers=1 --retries=0
# 9 passed
```

Scanner total: **9/9 passed**, zero failures.

`npx tsc --noEmit` (apps/mercato, `NODE_OPTIONS=--max-old-space-size=6144` under this VPS's current shared memory pressure): clean, zero errors.

## Gaps found and fixed (test-only; no product/business behavior changed)

1. **E2BIG in P1-001/002/003 runtime seams.** Their MikroORM seam helper bundled a script with esbuild and executed it via `node -e <code>`; the bundled script has grown large enough to exceed the OS argv size limit. Fixed by writing the bundle to a temp `.cjs` file and executing that file instead — the file must live inside the repo root (not `os.tmpdir()`), since Node's CommonJS resolver looks for `node_modules` relative to the file's own directory, and `/tmp` has none.
2. **Stale packer test-user password (P1-010).** The packer test-user upsert only inserted a row if none existed by email/hash, silently reusing whatever password hash a prior DB state left behind instead of syncing it to the current designated `testPackerPassword` every run. Fixed to always update the hash.
3. **Wrong column name (P3-001 Journey A).** A verification query referenced `order_line_id`; the actual column is `outbound_order_line_id`.
4. **Stale test expectation, not a product defect (P3-001 Journey B2).** The test expected a direct API cancel to return 422 for a merely-PICKED (not PACKED) line. The already-accepted `P4-001` Journey C evidence establishes that such a line requires no Supervisor approval and auto-settles through Physical Putback (200 OK) — `requiresSupervisorApproval` in `ordering-adapter-service.ts` is gated on `hasPacked`, confirmed correct by direct code read. Updated the assertion to expect success and verify the resulting `WmsOutboundPutBackTask` row, keeping the test's actual intent (proving P3 immediate release is blocked and correctly routes to P4) intact via the UI-level assertions already in the same test.
5. **Missing preconfirm gate in Scanner P1-006/007/009 specs.** `P3-003` added a source/SKU preconfirm gate to `PickingTaskScreen` (Confirm Pick and each subsequent pick action stay disabled until `preconfirm-source-sku-btn` is accepted; the gate resets after every successful confirm). The P1-006/007/009 specs predate that gate and never clicked it, leaving Confirm Pick permanently disabled — a cross-item regression in test-only code caused by a later item extending a shared screen, matching the precedent already documented in `WMS_Outbound/04_CURRENT_STATE/TEST_INFRA_GAPS.md`. Added the missing preconfirm step before every Confirm Pick action across all three files (6 call sites in P1-009 alone).

One redundant, pre-real mocked draft (`p1-005-picking-scanner-assignment.spec.ts`, fully route-mocked, superseded by the "Zero Route Mocks" `p1-005-real-scanner-assignment.spec.ts`) was left unmodified and unrun for acceptance purposes — the real spec already provides decisive Journey-4-adjacent coverage for task assignment, and TESTING.md's evidence hierarchy places REAL UI/Playwright above a mocked simulation.

## Regression scope

Zero product/application code changed in either repository. All fixes are to test files only (temp-file execution, fixture password sync, a query column name, one assertion, and missing UI interaction steps). No shared Inbound/Inventory/TU/warehouse/lock/orchestration primitive was touched, so no additional Inbound regression sweep is required.

## Completion statement

All 14 required Evidence Standard journeys (plus partial/multi-shipment exceptions) have decisive `PLAYWRIGHT VERIFIED` coverage on the final checkpoint. Mercato 57/57 and Scanner 9/9 Playwright tests pass; Mercato typecheck is clean. Every fix found and applied was test-harness-only; no business logic was altered. Evidence and implementation are pushed. ACC-003 was not started before this internal gate was confirmed green.

This is executor-prepared evidence, not Supervisor FINAL PASS or Owner Acceptance.
