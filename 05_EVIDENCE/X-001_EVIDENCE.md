# X-001 — Enforce CON-01..05 Concurrency and Exactly-Once Business Effects — Evidence

Date: 2026-09-06 UTC
Evidence class: REAL POSTGRESQL INTEGRATION, REAL CONCURRENCY (approved Testing Supabase `DevAxonic_Platform`, real `pg_stat_activity`/`pg_locks`/`pg_blocking_pids` lock-wait proof, real overlapping transactions on separate PostgreSQL backend PIDs). This is not Human Verified, FINAL PASS, or Owner Accepted.

## Exact Revisions and Scope

- Mercato branch: `outbound/x-001`
- Mercato final candidate commit SHA: `f7af85651`
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

**Audited, reused, no change.** `crossdock-planning-service.ts`, tested by `p2-002-crossdock-planning-postgres.integration.test.ts`:
- "is replay-safe and concurrent calls produce one line/task for the binding" — two independent service instances call `planBinding` concurrently via real `Promise.all`; post-condition asserts exactly one `WmsOutboundCrossDockPickTask` and one `WmsOutboundOrderLine` exist.
- "concurrent operators can receive one created task at most once" — same pattern for `assignNext`.
- Result: unchanged by this item; included in regression (see below).

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

## Test Results

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

## Build

- `apps/mercato`: `NODE_OPTIONS=--max-old-space-size=6144 npx tsc --noEmit` — clean, zero errors, exit 0. (A first attempt without the increased heap size OOM-crashed at the default Node heap limit — the same known environment constraint recorded in `P4-003_EVIDENCE.md`; not a code defect. Retried with the heap flag and passed clean.)
- No migration, no schema change, no route/API change, no UI change in this item — no `mercato generate`, `mercato db migrate`, or Testing runtime rebuild/restart was required or performed.

## Not Implemented / Explicitly Out of Scope

- No Playwright evidence: no visible retry/conflict UI behavior changed in this item (CON-01..05 hardening is entirely backend transaction/lock logic); per the guide and `.ai/STATE.md` grounding, Playwright is required only if visible behavior changes.
- Scanner repository: untouched.
- The stale `CON-04`/`CON-05` source-number mapping in `03_TRACEABILITY/requirements_index.csv` was not corrected — out of this item's product/test scope; recorded here and in the guide as a known traceability-reference defect for a future documentation-only correction.
