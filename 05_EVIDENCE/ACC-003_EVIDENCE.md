# ACC-003 — Playwright Crossdock, Reservation Release and Physical Putback Journeys — Evidence

Date: 2026-09-07 UTC
Evidence class: REAL UI / PLAYWRIGHT (Mercato + Scanner, canonical Testing runtime, canonical Testing PostgreSQL). Automated browser proof is `PLAYWRIGHT VERIFIED`, not `HUMAN VERIFIED`. Not Supervisor FINAL PASS, not Owner Accepted.

## Exact revisions and scope

- Mercato batch branch: `outbound/acc-001-003-batch` @ `bc80c989eea2f6b468530a9bd6cd814c8dc35646` (unchanged from ACC-002 — no Mercato diff was required for ACC-003).
- Scanner batch branch: `outbound/acc-001-003-batch`, base `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.
- Scanner ACC-003 checkpoint commit SHA: `f9a98dd8a03078dd9a0e45667927861ab6f79f7b` (test-only diff plus two refreshed screenshot evidence artifacts; no application code changed).
- Item: 36/37 — ACC-003: Playwright Crossdock, Reservation Release and Physical Putback journeys.
- Execution guide: `06_AGENT_GUIDES/ACC-001_003_BATCH_EXECUTION.md`.

## Runtime provenance

- `mercato-localhost.service` rebuilt/restarted from the exact Mercato ACC-002 checkpoint (`bc80c989eea2`) via the durable build probe (`BUILD_OK`); active, `MainPID` cwd `/home/ubuntu/git/Devaxonic-mercato`, port 3009 owned by the canonical runtime, `/login` HTTP 200.
- `scanner-testing.service` active, `MainPID` cwd `/home/ubuntu/git/Devaxonic-scanner`, port 8081 HTTP 200.
- Canonical Testing PostgreSQL confirmed non-loopback before every DB-backed command.

## Journey-to-spec mapping (Evidence Standard journeys 15–20)

| # | Journey | Spec(s) | Result |
|---|---|---|---|
| 15 | Outbound Crossdock 1:1 | Scanner `p2-002-crossdock-assignment.spec.ts`; `p2-003-real-crossdock-execution.spec.ts` Journey A | PASS |
| 16 | Crossdock n:n sorting | Scanner `p2-003-real-crossdock-execution.spec.ts` Journey B | PASS |
| 17 | Crossdock shortage/damage/empty source TU | Scanner `p2-004-real-crossdock-recovery.spec.ts` Journeys A–D | PASS |
| 18 | Goods Receipt gate pending/rejected/re-evaluated | Mercato `P2-005-crossdock-gr-gate-ui.spec.ts` Journeys A–C | PASS |
| 19 | Reservation Release policy paths | Mercato `P3-002-reservation-retention-ui.spec.ts` Journeys A–B (partial release, `AUTO_RELEASE_AFTER_TIME`); Scanner `p3-003-rendered-acceptance.spec.ts` TC-042/043 (P3 pre-pick source-return vs P4 routing, real backend) | PASS |
| 20 | Physical Putback invalid-location loop and successful inventory recovery | Scanner `p4-003-rendered-acceptance.spec.ts` Journey E (exact match: invalid location rejected, then valid location completes and recovers Inventory PICKED → AVAILABLE); `p4-002-rendered-acceptance.spec.ts` Journeys A–D (FIFO assignment, active-task guard, exactly-once race, Mercato Supervisor observability) | PASS |

Downstream integration supporting journeys 15/16: Mercato `P2-006-crossdock-shipment-downstream-ui.spec.ts` Journeys A/B-C (CROSSDOCK continuously joining a common Shipment through carrier/GR/ERP/manifest/settlement).

## Exact commands and results (final checkpoint)

```
# Mercato
npx playwright test P2-005-crossdock-gr-gate-ui.spec.ts P2-006-crossdock-shipment-downstream-ui.spec.ts \
  P3-002-reservation-retention-ui.spec.ts --config .ai/qa/tests/playwright.config.ts --workers=1 --retries=0
# 7 passed

# Scanner
npx playwright test e2e/p2-002-crossdock-assignment.spec.ts e2e/p2-003-real-crossdock-execution.spec.ts \
  e2e/p2-004-real-crossdock-recovery.spec.ts --workers=1 --retries=0
# 7 passed

npx playwright test e2e/p3-003-rendered-acceptance.spec.ts --workers=1 --retries=0
# 2 passed

npx playwright test e2e/p4-002-rendered-acceptance.spec.ts e2e/p4-003-rendered-acceptance.spec.ts --workers=1 --retries=0
# 5 passed
```

Mercato total: **7/7 passed**. Scanner total: **14/14 passed**. Combined ACC-003 decisive Playwright coverage: **21/21 passed**, zero failures.

Mercato `git status --porcelain` was empty for ACC-003 (no product/test diff required); typecheck therefore stayed at the clean ACC-002 result.

## Gap found and fixed (test-only; no product/business behavior changed)

**sslmode overriding the explicit ssl option in five Scanner raw-pg clients.** `p2-002`, `p2-003`, `p2-004`, `p4-002`, and `p4-003` each construct a raw `pg.Client`/`Pool` directly against `process.env.DATABASE_URL` with an explicit `ssl: { rejectUnauthorized: false }`. With `sslmode=require` present in the canonical Testing `DATABASE_URL`, this pg/pg-connection-string version merges the string-parsed SSL mode over the explicit option, discarding `rejectUnauthorized: false` and failing with `self-signed certificate in certificate chain` — the exact defect class the Mercato post-X-002 raw pg SSL maintenance gate already fixed in Mercato's own Jest tree, here recurring in Scanner's separate e2e tree (never in scope of that earlier gate). Fixed by stripping `sslmode=...` from the connection string before construction in all five files, matching the same fix shape.

## Documented exception (not fixed; superseded, not required for this item)

`p3-003-preconfirm-recovery.spec.ts` fully route-mocks the backend and seeds a fake auth token (`{ token: 'test', email: 'operator@test' }`) via `localStorage`. The app now enforces a real session-validity check that this fake token fails, surfacing a "Session expired" reauth modal that permanently intercepts the next click (180s test timeout). This is the same shape of staleness as the `p1-005-picking-scanner-assignment.spec.ts` mocked draft noted in `ACC-001_EVIDENCE.md`/`ACC-002_EVIDENCE.md`: a route-mocked test overtaken by later real auth requirements. `p3-003-rendered-acceptance.spec.ts`'s TC-042 ("Operator Pre-confirms Source & SKU -> Supervisor executes P3 Cancellation -> Scanner receives exact-source return instruction with 0 PutBackTask & 0 P4 handoff") already decisively proves the identical business scenario against the real backend, real auth, and real PostgreSQL — a strictly stronger evidence class per `.ai/TESTING.md` §4. Left unmodified; not run for acceptance purposes.

## Architecture boundaries preserved

- Inbound remained CLOSED / REFERENCE; no Inbound data, GR retry ownership, or Putaway business semantics were touched.
- `INT-01..03/06` and P3/P4 concurrency/idempotency semantics were exercised as-is; no correctness fix was required (all failures found were test-harness-only: SSL construction and a stale mocked-auth draft).
- No destructive Inbound data changes; all fixtures used were additive/isolated.

## Regression scope

Zero product/application code changed in either repository for ACC-003. All fixes are to Scanner test files only (connection-string SSL construction). No shared Inventory/TU/warehouse/lock/orchestration primitive was touched, so no additional Inbound regression sweep is required.

## Completion statement

All 6 required Evidence Standard journeys (15–20) have decisive `PLAYWRIGHT VERIFIED` coverage on the final checkpoint: Mercato 7/7, Scanner 14/14, zero failures. One superseded mocked draft is documented and intentionally excluded per the stronger real-backend evidence already available for the same scenario. No product code was changed. Evidence and implementation are pushed. `ACC-004` has not started.

This is executor-prepared evidence, not Supervisor FINAL PASS or Owner Acceptance.
