# P1-015 — bounded remediation: close-vs-carrier race + PID-bound concurrency evidence

**Execution status:** authorized first corrective shot for `P1-015`  
**Catalog item:** `18/37`  
**Scope type:** remediation execution artifact only; **NOT steering / NOT acceptance**  
**Workflow:** follow current `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

## Mandatory fresh-session startup

Run this corrective shot in a **fresh Antigravity session**.

```bash
cd /home/ubuntu/git/Devaxonic-WMS
/home/ubuntu/.local/bin/agy-pl
```

Never use bare `agy`.

Before changing anything, sync and read current Git state, then read:

1. `Devaxonic-WMS/AGENTS.md`
2. `Devaxonic-WMS/.ai/STATE.md`
3. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`
4. `Devaxonic-WMS/.ai/TESTING.md`
5. `Devaxonic-WMS/.ai/OPERATIONS.md`
6. `WMS_Outbound/STATE.md`
7. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-04.md`
8. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
9. `WMS_Outbound/06_AGENT_GUIDES/P1-015_EXECUTION.md`
10. this remediation guide
11. current `WMS_Outbound/05_EVIDENCE/P1-015_EVIDENCE.md`
12. exact current P1-015 Mercato code/tests and Git refs

Do not reconstruct mutable state from old chat.

## Frozen review refs

Supervisor independently reviewed these exact refs:

- accepted P1-014 Mercato base: `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`
- current P1-015 Mercato head before remediation: `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8`
- branch: `outbound/p1-015`
- lineage: exactly one commit above accepted P1-014 base
- current P1-015 WMS evidence head before this remediation guide: `2302b29315560ab4020a5d9febcf4c3d56b5514c`
- frozen Scanner ref: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` on `outbound/p1-009`
- Devaxonic-WMS steering: `3f46432cc75899c80be38ec9206d61b9a544f416`
- authoritative WMS progress remains `17/37 FINAL PASS`; P1-015 is **not accepted** yet.

If Mercato `outbound/p1-015` no longer contains exact head `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8` before your corrective commit, or if WMS/steering refs unexpectedly moved, **STOP** and report actual refs. Do not rebase, merge, squash, transplant or guess around drift.

## Why remediation is required

Initial P1-015 implementation is close, but supervisor found two material blockers plus one evidence-completeness issue.

### B1 — `CarrierManifest.CLOSED` does not currently serialize against a concurrent Supervisor Carrier correction

The committed P1-015 Carrier correction path locks the `Shipment`, then reads its `CarrierManifest` status without a manifest row lock. `closeManifest()` locks the manifest but does not lock its member Shipments before committing `OPEN -> CLOSED`.

That allows this real race:

1. Supervisor correction locks an `IN_MANIFEST` Shipment;
2. correction reads manifest as `OPEN` and passes the pre-close guard;
3. concurrently Dispatcher `closeManifest()` locks the manifest and commits `CLOSED` because it does not wait on that Shipment;
4. the still-running correction can then persist a new Carrier after the manifest is already durably `CLOSED`.

That violates the exact P1 R40 boundary P1-015 was required to complete.

#### Required behavior

A Carrier change must never commit after the Shipment's manifest has become durably `CLOSED`, `HANDED_OVER` or `CONFIRMED`.

Preserve all accepted behavior:

- pre-close Warehouse Supervisor correction remains allowed at the Architect-valid loading boundary (`POSTED` / `IN_MANIFEST` while manifest is `OPEN`);
- no automatic label regeneration/reprint;
- no ERP repost;
- no membership removal/reassignment;
- non-Supervisor correction remains blocked.

Implement the narrowest deterministic synchronization consistent with current P1-015 lock ordering.

A preferred safe shape is to make `closeManifest()` serialize on the frozen composition before it writes `CLOSED`, e.g. lock the manifest and its exact current member Shipment rows in a deterministic order inside the same transaction, so an in-progress correction must finish before close can commit and a correction starting after close sees the closed state and fails. An equivalent solution is acceptable if it proves the same invariant and does not introduce deadlocks or broaden scope.

Do **not** solve this with UI-only disabling, sleeps, retries hiding errors, or a database trigger that duplicates the domain state machine.

### B1 required real PostgreSQL race test — Carrier correction vs close

Add a genuine overlapping test using **two real service operations**:

- one real Warehouse Supervisor Carrier correction on an `IN_MANIFEST` Shipment in an `OPEN` manifest;
- one real Dispatcher `closeManifest()` on that same manifest.

The test must deterministically overlap the operations and prove database-side serialization. Use independent ORM/database sessions.

Required proof:

1. start with real `OPEN` manifest + attached `IN_MANIFEST` Shipment + an existing generated/printed label artifact;
2. operation A begins real Carrier correction and holds the relevant Shipment/domain lock long enough to overlap;
3. operation B begins real `closeManifest()` before A commits;
4. capture exact backend PID/session identity for both operations from the same sessions that execute the real business calls;
5. prove the blocked operation is the exact expected PID with `wait_event_type = 'Lock'` and `pg_blocking_pids(exactBlockedPid)` containing the exact blocker PID, or equivalent decisive PostgreSQL-side evidence;
6. settle both operations;
7. fresh independent reads prove only an Architect-valid serialization occurred:
   - if correction wins first, Carrier changed while manifest was still `OPEN`, then close commits `CLOSED` with that Carrier frozen;
   - if close wins first, correction fails and Carrier remains unchanged;
   - forbidden: Carrier changes after durable `CLOSED`;
8. existing label identity/payload/print count remains unchanged — no auto regeneration/reprint;
9. manifest membership remains unchanged.

Prefer a deterministic controlled winner so the literal output is stable. A timing-only `Promise.all()` is insufficient.

### B2 — current PID evidence does not bind the observed blocked backend to the exact operation B session

The committed concurrency tests currently discover `pidB` by querying for any backend whose `pg_blocking_pids(...)` contains `pidA`, then assign that observed row's PID to the variable named `pidB`.

That is not the exact operation/session identity proof required by the execution guide, especially for duplicate/parallel confirmation where the guide explicitly says:

> exact A↔B backend PID contention tied to those operations, not a global unrelated lock query.

The functional outcomes are strong, but the PostgreSQL-side identity proof must be made exact.

#### Test 6 — same Shipment assigned to two manifests

Keep two real `addShipmentToManifest()` calls.

Capture:

- `pidA` from operation A's exact transaction/session while it holds the Shipment lock;
- `pidB` independently from operation B's exact transaction/session **before B blocks on the Shipment lock**.

A convenient existing seam is that B locks its distinct target manifest before it attempts the shared Shipment lock; its exact backend PID can be captured there. Other exact same-session methods are acceptable.

Observer must query **specifically for `pidB`** and assert/log:

- `pidA != pidB`
- observed `pid == pidB`
- `wait_event_type == 'Lock'`
- `pg_blocking_pids(pidB)` contains exact `pidA`.

Do not infer `pidB` from the observer row after the fact.

Retain fresh-read proof of one membership and one `ShipmentAddedToManifest` fact.

#### Test 18 — duplicate/parallel confirmation

Keep two real `confirmManifest()` operations.

Capture:

- exact `pidA` from A's transaction/session while holding the manifest lock;
- exact `pidB` from B's same transaction/session **before** B attempts the blocking manifest lock.

If useful, start B inside an explicit transaction-bound EM, run `SELECT pg_backend_pid()` there, then call the real service using that exact transaction-bound EM; after B eventually acquires the manifest lock, assert any hook-reported PID still equals the pre-captured `pidB`.

Observer must query specifically for exact `pidB` and assert/log:

- `pidA != pidB`
- observed blocked PID equals `pidB`
- `wait_event_type == 'Lock'`
- `pg_blocking_pids(pidB)` contains exact `pidA`.

Then settle both and retain exactly-one `CarrierManifestConfirmed` proof.

#### Test 10 — add versus close

The existing functional outcome is acceptable, but if the chosen remediation touches its hooks/locking, strengthen it to exact blocked-session identity too. Do not regress the existing decisive outcome: no Shipment may attach after durable close.

### B3 — evidence completeness / final-head status

Correct `05_EVIDENCE/P1-015_EVIDENCE.md` from final Git/runtime facts.

Current evidence records the WMS worktree as clean at the **pre-evidence base** `0d9b1f88319d2c8b202b6c75a9f21c5c74f53e75`, even though the evidence itself was pushed as `2302b29315560ab4020a5d9febcf4c3d56b5514c`. That is not a final post-push WMS worktree identity.

The corrected evidence must include:

- accepted P1-014 base `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`;
- pre-remediation P1-015 head `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8`;
- exact final remediation Mercato SHA on `outbound/p1-015`;
- exact lineage / merge base / commit count;
- exact Git-derived final compare/file stats;
- literal final P1-015 PostgreSQL test inventory/count;
- literal exact PID output for Test 6 and Test 18, and for the new Carrier-correction-vs-close race;
- exact `pidA`, `pidB`, `blockedPid`, `blockingPids`, `waitEventType` with the test asserting those identities, not prose inference;
- fresh-read exactly-once event/membership counts;
- the new close-vs-carrier serialization result and unchanged label identity/print evidence;
- real rollback proof;
- literal regression outputs/counts actually rerun;
- fresh rendered Playwright result from the exact **final remediated served Mercato SHA** because this remediation changes runtime locking behavior;
- exact served Testing Mercato runtime SHA;
- exact Scanner frozen SHA/branch;
- exact Devaxonic-WMS steering ref untouched;
- exact final `git status --short` / clean worktree evidence **after all pushes** for Mercato and WMS at their final pushed heads;
- P1-016 settlement exclusion remains explicit and unchanged;
- no `FINAL PASS` / `Owner Accepted` claim.

If one new PostgreSQL behavior test is added, the final P1-015 count will naturally increase from `20/20`; record the real count. Do not delete or relabel required tests to preserve `20/20`.

## Allowed corrective product scope

Only the narrow files genuinely needed for B1/B2 are authorized. Expected candidates are:

- `apps/mercato/src/modules/wms_outbound/services/manifest-service.ts`
- `apps/mercato/src/modules/wms_outbound/services/carrier-selection-service.ts` only if the chosen correct synchronization requires a narrow change there
- `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-015-manifest-lifecycle-postgres.integration.test.ts`
- P1-015 Playwright spec only if required to re-prove the remediated final runtime boundary; do not add unrelated journeys.

Do not change schema/migration unless the runtime invariant genuinely cannot be implemented correctly without it. The current blocker is transactional synchronization, not a request for new business data.

No unrelated refactor.

## Required final verification

On the exact final Mercato remediation head, rerun:

1. full P1-015 PostgreSQL suite including:
   - exact-PID Test 6,
   - existing add-vs-close race,
   - new real Carrier-correction-vs-close race,
   - exact-PID duplicate/parallel confirm,
   - rollback,
   - P1-016 non-settlement boundary;
2. P1-014 PostgreSQL `18/18`;
3. P1-013 PostgreSQL `15/15`;
4. P1-012 PostgreSQL `14/14`;
5. P1-011 PostgreSQL `18/18`;
6. FND-002 state-machine accepted count (`77/77` unless legitimately additive and explained);
7. FND-002 transaction simulation `8/8` if still applicable;
8. affected targeted shared regression required by current `AGENTS.md` / Testing rules;
9. fresh P1-015 Playwright against canonical Testing Mercato served from the exact final remediation SHA. Existing six journeys may remain six if they still fully cover the UI contract; do not inflate count without a real journey.

Record literal outputs and counts. No invented aggregate.

## Preserve accepted/working P1-015 behavior

Do not regress:

- only `POSTED` Shipment enters an `OPEN` manifest;
- one Shipment belongs to one manifest;
- same Shipment→same manifest replay is idempotent;
- add-vs-close composition race cannot attach after close;
- `OPEN -> CLOSED` is irreversible;
- physical handover remains separate `CLOSED -> HANDED_OVER`;
- handover transitions Shipment/TU/OutboundOrder exactly once;
- `HANDED_OVER -> CONFIRMED` remains a separate exactly-once confirmation boundary;
- confirmation snapshot remains deterministic and sufficient for P1-016;
- non-dispatch operator actions fail closed;
- no external carrier API;
- no automatic label regeneration/reprint on carrier correction;
- no P1-016 settlement.

## Hard exclusions

No:

- P1-016 Inventory/Allocation/OOL/OutboundOrder/CustomerOrder settlement;
- P1-016 cancellation logic;
- P1-016 execution guide/start;
- Crossdock GR gating;
- Return Receipt;
- external carrier provider API/webhook;
- Scanner changes;
- Prod/Demo changes;
- STATE/handover/traceability/task catalog/implementation plan/workflow edits;
- Devaxonic-WMS `.ai/*` edits;
- merge/delete/rebase/squash;
- acceptance claim.

WMS corrective write is limited to updated `05_EVIDENCE/P1-015_EVIDENCE.md` after the supervisor-authored remediation guide already present on `main`.

## Two-strikes rule

This guide is the **one authorized corrective shot** for P1-015.

If the same material close-boundary or exact-session concurrency-evidence failure remains after this attempt, **STOP**. Do not self-authorize a second remediation or a wider redesign. Owner authorization would be required for any additional shot.

## STOP boundary

When complete:

1. push the narrow corrective commit(s) to `Devaxonic-mercato/outbound/p1-015` on top of exact `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8`;
2. update/push only `WMS_Outbound/05_EVIDENCE/P1-015_EVIDENCE.md` on top of this remediation guide;
3. report exact 40-char Mercato final SHA, WMS evidence SHA, frozen Scanner SHA and literal key test counts;
4. STOP.

Do not start P1-016. Do not update steering. Do not claim FINAL PASS or Owner Accepted.

Owner-facing success response: `done` plus exact refs/test counts only. Blocker response: at most 5 lines.