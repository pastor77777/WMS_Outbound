# X-001 — Enforce CON-01..05 Concurrency and Exactly-Once Business Effects — Evidence

Date: 2026-09-06 UTC
Evidence class: REAL POSTGRESQL INTEGRATION, REAL CONCURRENCY (approved Testing Supabase `DevAxonic_Platform`, real `pg_stat_activity`/`pg_locks`/`pg_blocking_pids` lock-wait proof, real overlapping transactions on separate PostgreSQL backend PIDs). This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/x-001`
- Mercato final candidate commit SHA: `b56515ecf` (acceptance correction; supersedes `f7af85651` for CON-03 decisiveness — see "Acceptance Correction" below)
- Mercato accepted base: `outbound/p4-003` @ `9a656bf9e5a42e29b6493f1b7bed5d62b7bf562c` (P4-003 FINAL PASS / Owner Accepted, plus post-acceptance test-infra maintenance)
- Scanner: untouched, still `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302` (frozen; no visible retry/conflict UI defect exists in this item's scope, so no Scanner branch was created)
- WMS evidence/guide commit: this commit, on `WMS_Outbound/main`
- Item: 32/37 — X-001: Enforce CON-01..05 concurrency and exactly-once business effects
- Execution guide: `06_AGENT_GUIDES/X-001_EXECUTION.md` (owner-authored, commit `ac1a883`). This session independently grounded X-001 against Architect source and current accepted evidence and began implementation before discovering the owner's own guide had landed on `origin/main` concurrently (commit `ac1a883`, timestamped during this session's work). Reconciled by keeping the owner's guide as authoritative and cross-checking this implementation against its exact acceptance matrix and completion contract (see per-CON sections below) — the independently-derived grounding matched the owner's guide on every material point (bases, CON-01..05 authority mapping including the same stale-reference correction, and the "audit first, harden only on a real gap" strategy), so no rework was needed.

## Authority Chain

`proces_1_standard_fulfillment_EN.md` R4–R6, R10, R26–R29, R39–R40, R52, R70–R72; `proces_2_outbound_crossdock_EN.md` R6, R29–R30 -> derived `CON-01`..`CON-05` -> `TC-090`..`TC-094` (concurrency scenario universe). `03_TRACEABILITY/requirements_index.csv`'s `CON-04`/`CON-05` source-number mapping is stale (resolves `CON-05` to shortage-handling rules R43–R46) — classified as a traceability-reference defect during this item's grounding, not a product blocker; current Architect process prose (cited above) was used as governing authority instead, consistent with the accepted handover note.

## Scope and Method

X-001 is an audit-and-harden item, not a new business workflow. Every `CON-01`..`CON-05` was audited against its exact current owning implementation and exact current test file (not evidence prose) before any change was made. Per the item's own DoD: existing genuinely decisive evidence counts as-is; only a real proof or code gap is hardened. No green flow was refactored for uniformity.

## Per-CON Audit Outcome

### CON-01 — Atomic ATP/allocation reservation (P1 R4–R6, R52)

**Audited, reused, no change.** `allocation-service.ts::allocateOrder`, tested by `p1-004-postgres.integration.test.ts`:
- Case G: real overlapping transactions on the same OutboundOrderLine, `pg_advisory_xact_lock` contention proven via `pg_locks`/`pg_stat_activity` on the waiting backend PID before the holder commits.
- Case H: independent competing-demand transactions for limited stock; `sum(reserved) <= stock` proven; decisive `pg_stat_activity` advisory-lock wait proof on the second PID.
- Case I: real rollback — flushed write, forced failure before commit, fresh independent read proves zero partial commit.
- Result: **11/11 PASSED** (unchanged by this item; rerun as regression, see below).

### CON-02 — PickTask immutability (P1 R10)

**Gap found in proof, not in code — hardened.** `allocation-service.ts::releaseAllocation` and `pick-task-service.ts::generatePickTasks` both take a `PESSIMISTIC_WRITE` row lock on the owning `WmsOutboundAllocation` before acting, which already correctly serializes a concurrent release-vs-create race. However, the only existing test for this guard (`p1-005-postgres.integration.test.ts` Case G) exercised it **sequentially** (generate, then attempt release), not as a genuine overlapping transaction — insufficient to meet the `.ai/TESTING.md` §4/§5 "REAL CONCURRENCY" bar.

**Change made:**
- `allocation-service.ts::releaseAllocation` — added optional `onTxStart`/`onLockAcquired` test-support hooks (additive; mirrors the existing `allocateOrder` hook pattern in the same file; behavior is unchanged for every call site that omits `options`, including all production call sites).
- `pick-task-service.ts::generatePickTasks` — **no change**; its existing `beforeCommitHook` (fires after the allocation row is locked and the PickTask/PickTaskLine are flushed, before commit) was sufficient to hold the lock open for the test.
- New test `p1-005-postgres.integration.test.ts` **Case G2**: launches `generatePickTasks` (Tx1) and holds it open post-flush via `beforeCommitHook`, then launches `releaseAllocation` (Tx2) concurrently on the same Allocation. Observer polls `pg_stat_activity`/`pg_blocking_pids` and proves Tx2 is genuinely blocked (`wait_event_type = 'Lock'`, blocked by Tx1's real backend PID) before Tx1 is released to commit. Outcome: Tx1's PickTask creation wins the row lock and commits; Tx2's release then correctly rejects with the existing CON-02 error once it proceeds. Fresh DB read confirms the Allocation remains `RESERVED` with its PickTask intact — never released mid-flight.
- Result: **11/11 PASSED** (10 pre-existing + 1 new Case G2), 6.7s.
- No new rollback proof was added for this hardening: the change adds no new transaction boundary (the two existing transactions in `generatePickTasks`/`releaseAllocation` and their commit points are unchanged); the pre-existing `p1-005-postgres.integration.test.ts` Case I (real rollback on `assignNextPickTask`) and `p1-004-postgres.integration.test.ts` Case I (real rollback on `allocateOrder`) already cover rollback behavior on the same transactional primitives this hardening exercises.

### CON-03 — Single cross-dock assignment (P2 R6, R29–R30)

**Corrected in the acceptance-correction pass — see "Acceptance Correction" below for the decisive replacement evidence.** The original audit (superseded) accepted the pre-existing `Promise.all`-only tests as sufficient; the supervisor found those non-decisive (no forced transaction overlap, no PostgreSQL-side lock proof) and required genuine overlap evidence, which is now in place.

### CON-04 — Stable Shipment grouping / single manifest membership (P1 STEP 9, R26–R29, R39–R40)

**Audited, reused, no change.** `shipment-grouping-service.ts`, tested by `p1-011-postgres.integration.test.ts`:
- Test 5 "Parallel grouping / CON-04": genuine `pg_advisory_xact_lock` on the grouping key, real overlapping `Promise.all` participants, decisive `pg_stat_activity` lock-wait proof, converges on one Shipment.
- Test 16: real rollback proof.

`carrier-manifest-service.ts`, tested by `p1-015-manifest-lifecycle-postgres.integration.test.ts`:
- Test 6 "Real concurrent assignment to two manifests is exactly-one with decisive PostgreSQL overlap proof".
- Test 10 "Real add-versus-close race cannot attach a Shipment after durable close".
- Test 21 "Real race: Supervisor Carrier correction vs closeManifest serializes deterministically".

All use real overlapping transactions with row-lock contention proven via `pg_blocking_pids`/`pg_stat_activity` against actual participant PIDs. Result: unchanged by this item; included in regression (see below).

### CON-05 — Exactly-once ERP/manifest effects (P1 R37–R38, R70–R72)

**Audited, reused, no change.** `erp-posting-service.ts`, tested by `p1-014-erp-posting-postgres.integration.test.ts`:
- Test 14 "Genuine PostgreSQL Concurrency: overlapping initial posting serialize with pg_blocking_pids proof".
- Test 15 "Duplicate in-flight and post-settlement ERP posting requests are exactly-once and idempotent".
- Test 16: real rollback proof.

`carrier-manifest-service.ts`, tested by `p1-015-manifest-lifecycle-postgres.integration.test.ts`:
- Test 18 "Duplicate/parallel confirm is exactly-once with real database proof".

`final-settlement-service.ts`, tested by `p1-016-final-settlement-postgres.integration.test.ts`:
- Test 23: real flushed-settlement-failure rollback leaves every P1-016 business effect unchanged.
- Test 24 "real parallel duplicate has distinct pids and PostgreSQL Lock wait" — two concurrent confirm attempts on the same manifest, `pg_blocking_pids` proof, exactly one settlement fact persists.
- Test 25 "real two-manifest race locks shared line/allocation without lost update" — two concurrent confirms on different manifests contributing to the same OutboundOrderLine/Allocation, `pg_blocking_pids` proof, no lost update (final `reservedQty`/`shippedQty`/status correct, two distinct settlement facts).

`p2-006-crossdock-shipment-downstream-postgres.integration.test.ts` test 20 confirms the same shared `final-settlement-service.ts` path (proven above) is reused unmodified for crossdock/mixed Shipments — decisive by inheritance; no separate crossdock-specific settlement-concurrency mechanism exists to test independently.

Result: unchanged by this item; included in regression (see below).

## Test Results (superseded by "Acceptance Correction" final-head reruns below)

Dedicated suite (includes new Case G2):

| Suite | Result |
|---|---|
| P1-005 PickTask generation/ordering/assignment/concurrency (CON-02) | **11/11 PASSED** (6.7s) |

Targeted regression — every other `allocation-service.ts` caller, since `releaseAllocation`'s signature gained additive optional hooks:

| Suite | Result |
|---|---|
| P1-004 Allocation hard reservation (CON-01) | PASSED |
| P1-007 | PASSED |
| P3-001 Reservation release | PASSED |
| P3-002 | PASSED |
| P4-001 | PASSED |
| P4-003 | PASSED |

Combined: **6 suites, 90/90 tests PASSED**, 54.4s, zero failures.

CON-01/CON-03/CON-04/CON-05 owning suites (`p1-004`, `p2-002`, `p1-011`, `p1-014`, `p1-015`, `p1-016`, `p2-006`) were read in full and independently judged decisive against current code (see per-CON sections above); `p1-004-postgres.integration.test.ts` was additionally re-run live as part of the regression set above and remains green. The others were not re-run in this item because no product or shared-signature code they call was touched — per `.ai/OPERATIONS.md` "Preserve already-proven green areas. Do not rerun broad suites after every fixture-only correction unless a product change invalidates those proofs," and no such invalidating change was made to their owning services (`crossdock-planning-service.ts`, `shipment-grouping-service.ts`, `carrier-manifest-service.ts`, `erp-posting-service.ts`, `final-settlement-service.ts`).

**Supervisor correction:** this "read in full, judged decisive, not re-run" reasoning was rejected for CON-03 specifically (the CON-03 tests were not actually decisive — Promise.all timing, not forced overlap) and the guide additionally required literally rerunning every claimed suite on the final X-001 head regardless. See "Acceptance Correction" below for the compliant replacement evidence.

## Acceptance Correction (`X-001_ACCEPTANCE_CORRECTION.md`, same-item continuation, no Testing reset)

Mercato final SHA for this correction: **`b56515ecf`** on `outbound/x-001` (parent `f7af85651`).

### Gap A — CON-03 made decisive

`cross-dock-planning-service.ts` gained additive, production-transparent test-support hooks (`onTxStart`, `onLockAcquired`, `beforeCommitHook`) on `planBinding` and `assignNext`, mirroring the existing hook pattern already used in `allocation-service.ts`/`pick-task-service.ts`. Every production call site omits `options`; behavior is unchanged (confirmed: `di.ts` is the only non-test caller and passes no options).

Two new decisive tests in `p2-002-crossdock-planning-postgres.integration.test.ts`:

1. **"X-001 CON-03 hardening: REAL CONCURRENCY — two overlapping planBinding transactions on the same binding serialize via PESSIMISTIC_WRITE, decisive DB lock wait proven, exactly one task/line durable"**
   - TxA acquires the binding's `PESSIMISTIC_WRITE` row lock, then holds open post-flush (pre-commit) via `beforeCommitHook`.
   - TxB launches concurrently against the same not-yet-planned binding.
   - Observer polls `pg_stat_activity`/`pg_blocking_pids` until TxB is genuinely blocked with `wait_event_type = 'Lock'` and `blockers` containing TxA's real backend PID.
   - Live-run PIDs: `pidA` / `pidB` distinct, decisive lock wait observed (`observedWaitEventType = 'Lock'`, `observedBlockers` contains `pidA`) — confirmed by a passing run of this exact assertion set.
   - TxA commits first (`replayed: false`); TxB, released to proceed after, correctly observes the already-created task (`replayed: true`).
   - Fresh independent read: exactly one `WmsOutboundCrossDockPickTask` for the binding, exactly one `WmsOutboundOrderLine` for the customer line, `plannedQty` intact (`5.000000`) — no duplicated planned quantity.

2. **"X-001 CON-03 hardening: REAL CONCURRENCY — two operators racing assignNext for the same warehouse serialize via warehouse-scoped advisory lock, decisive DB lock wait proven, task assigned exactly once"**
   - Operator A acquires the warehouse-scoped `pg_advisory_xact_lock` for assignment, then holds open post-flush via `beforeCommitHook`.
   - Operator B launches concurrently for the same warehouse.
   - Same `pg_stat_activity`/`pg_blocking_pids` decisive-wait proof (`wait_event_type = 'Lock'`, blockers contains Operator A's real PID).
   - Operator A's assignment commits first and wins the only `CREATED` task; Operator B, proceeding after, correctly gets `null` (no task left).
   - Fresh independent read: exactly one `ASSIGNED` `WmsOutboundCrossDockPickTask` for the binding, owned by `op-a`.

No product/business-logic change was required for CON-03 — the stronger race did not expose a defect; the existing `PESSIMISTIC_WRITE` binding lock and warehouse-scoped advisory lock were already correct, only the proof was insufficient before this correction.

Result: `p2-002-crossdock-planning-postgres.integration.test.ts` **24/24 PASSED** (22 pre-existing + 2 new decisive tests), 13.3s.

### Gap B — mandatory final-head dedicated-suite reruns

Every suite whose proof is claimed for X-001 was individually rerun on the final Mercato head (`b56515ecf`) against the approved Testing Supabase `DevAxonic_Platform`:

| Suite (CON area) | Result |
|---|---|
| `p1-004-postgres.integration.test.ts` (CON-01) | **11/11 PASSED** |
| `p1-005-postgres.integration.test.ts` (CON-02, incl. Case G2) | **11/11 PASSED** |
| `p2-002-crossdock-planning-postgres.integration.test.ts` (CON-03, incl. new hardening) | **24/24 PASSED** |
| `p1-011-postgres.integration.test.ts` (CON-04 grouping) | **18/18 PASSED** |
| `p1-015-manifest-lifecycle-postgres.integration.test.ts` (CON-04/CON-05 manifest races) | **21/21 PASSED** |
| `p1-014-erp-posting-postgres.integration.test.ts` (CON-05 ERP posting) | **18/18 PASSED** |
| `p1-016-final-settlement-postgres.integration.test.ts` (CON-05 final settlement) | **25/25 PASSED** |
| `p2-006-crossdock-shipment-downstream-postgres.integration.test.ts` (CON-05 shared crossdock downstream settlement) | **20/20 PASSED** |

Combined: **8 suites, 148/148 tests PASSED**, zero failures, on the final X-001 head.

**Test-tooling defect found and fixed during these reruns (not a product/business defect):** `p1-011`, `p1-014`, `p1-015` and `p1-016` each open a raw `pg.Client` observer connection for their PostgreSQL lock-wait proof, e.g. `new Client({ connectionString: dbUrl, ssl: { rejectUnauthorized: false } })`. In this environment's currently installed `pg@8.22.0`/`pg-connection-string@2.14.0`, when `connectionString` also carries `sslmode=require` (as the approved Testing `DATABASE_URL` does), `pg`'s `ConnectionParameters` constructor merges `parse(connectionString)` **over** the explicitly-passed `ssl` object (`sslmode=require` now aliases `verify-full`), silently discarding `rejectUnauthorized: false` and causing `self-signed certificate in certificate chain` on `.connect()`. This reproduced identically on the unmodified `outbound/x-001` base (`f7af85651`, confirmed via `git stash` before any correction code was applied) — a pre-existing environment/dependency-drift defect, not something this item's CON-03 change introduced, and not a business-logic gap.

Fix: strip `sslmode` from the connection string immediately before constructing each affected raw `pg.Client` (`dbUrl.replace(/[?&]sslmode=[^&]*/i, '')`), so the explicit `ssl: { rejectUnauthorized: false }` option survives the merge unclobbered. This is test-support-only; no product code, no MikroORM connection config (which was never affected — it already passed), and no business behavior changed. Applied in `p1-011-postgres.integration.test.ts`, `p1-014-erp-posting-postgres.integration.test.ts`, `p1-015-manifest-lifecycle-postgres.integration.test.ts`, `p1-016-final-settlement-postgres.integration.test.ts`. `p2-006` does not use a raw `pg.Client` and needed no change.

### Build

- `apps/mercato`: `NODE_OPTIONS=--max-old-space-size=6144 npx tsc --noEmit` — clean, zero errors, exit 0.
- No migration, schema, route/API or UI change in this correction.

### Reconciliation with the original per-CON audit above

CON-01, CON-02, CON-04, CON-05 sections above are unaffected by this correction and remain accurate as written (their owning suites are now additionally proven live on the final head per the table above, not merely read/judged). Only the CON-03 section's original "Promise.all is sufficient" conclusion was wrong and is superseded by this section.

## Build

- `apps/mercato`: `NODE_OPTIONS=--max-old-space-size=6144 npx tsc --noEmit` — clean, zero errors, exit 0. (A first attempt without the increased heap size OOM-crashed at the default Node heap limit — the same known environment constraint recorded in `P4-003_EVIDENCE.md`; not a code defect. Retried with the heap flag and passed clean.)
- No migration, no schema change, no route/API change, no UI change in this item — no `mercato generate`, `mercato db migrate`, or Testing runtime rebuild/restart was required or performed.

## Not Implemented / Explicitly Out of Scope

- No Playwright evidence: no visible retry/conflict UI behavior changed in this item (CON-01..05 hardening is entirely backend transaction/lock logic); per the guide and `.ai/STATE.md` grounding, Playwright is required only if visible behavior changes.
- Scanner repository: untouched.
- The stale `CON-04`/`CON-05` source-number mapping in `03_TRACEABILITY/requirements_index.csv` was not corrected — out of this item's product/test scope; recorded here and in the guide as a known traceability-reference defect for a future documentation-only correction.
