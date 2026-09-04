# P1-014 — bounded remediation: real concurrent posting + literal evidence

**Execution status:** authorized first corrective shot for `P1-014`  
**Catalog item:** `17/37`  
**Scope type:** remediation execution artifact only; **NOT steering / NOT acceptance**  
**Workflow:** `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

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
9. `WMS_Outbound/06_AGENT_GUIDES/P1-014_EXECUTION.md`
10. this remediation guide
11. current `05_EVIDENCE/P1-014_EVIDENCE.md`
12. exact current Mercato P1-014 code/tests and Git refs

Do not reconstruct mutable state from old chat.

## Frozen review refs

Supervisor review found the following exact refs:

- accepted P1-013 Mercato base: `5e6b70aa81afd28fe3217e4aad216e8a6482a769`
- current P1-014 Mercato head before remediation: `d20d95a097f9fe4b4459d6829fbd459c80a5efd3`
- branch: `outbound/p1-014`
- lineage: exactly one commit above accepted P1-013 base
- current P1-014 evidence commit before this remediation guide: `06ec7d451e96f3a0972c1aa5806ad74eb053dac0`
- frozen Scanner ref: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- Devaxonic-WMS steering remains at `a1d9348826d911b92e9ff74cf44ddbf0d6fa51a1`

If `outbound/p1-014` no longer contains exact head `d20d95a097f9fe4b4459d6829fbd459c80a5efd3` before your corrective commit, **STOP** and report the changed ref. Do not rebase/merge/squash around an unexpected head.

## Why remediation is required

The initial P1-014 implementation/evidence is **not accepted**. Supervisor found three material blockers.

### B1 — actual concurrent `postShipment()` calls are not proven and current code exposes an in-flight duplicate race

The committed test named `Genuine PostgreSQL Concurrency` does not run two real P1-014 initial posting operations.

Its operation A manually:

- locks the Shipment row,
- sets Shipment directly to `POSTED`,
- manually inserts a posting record,
- manually inserts an accepted attempt,
- holds that transaction open.

Only operation B calls `createShipmentPostingService(...).postShipment(...)`.

That proves a PostgreSQL row can block, but it does **not** prove two real P1-014 initial posting requests are safe.

More importantly, current production logic has a real race:

1. real call A enters Phase 1 and commits `LABEL_GENERATED/OWN_TRANSPORT -> POSTING_PENDING` plus posting intent before the ERP adapter call;
2. real call B can then acquire the Shipment lock while A is still in-flight;
3. current `postShipment()` treats only `POSTED` as replay and otherwise permits initial start only from `LABEL_GENERATED`/`OWN_TRANSPORT`;
4. therefore B sees `POSTING_PENDING` and throws an invalid-start-state error instead of handling the same logical in-flight posting idempotently.

This must be fixed narrowly inside P1-014.

#### Required behavior

For the same tenant/org + Shipment + logical posting identity:

- concurrent/repeated initial requests must not create a second posting intent;
- must not create a second `ShipmentPostingRequested` transition;
- must not create a second initial attempt for the same logical request;
- must not call the ERP adapter a second time merely because a duplicate request arrives while the first is already `POSTING_PENDING`;
- must not regress state;
- a duplicate arriving while the original request is in-flight must be handled as an explicit idempotent in-flight/replay result, or another equally safe deterministic result — **not** as the current generic invalid-start-state failure;
- a later duplicate after `POSTED` must remain safe replay.

Do not solve this by holding a real external/network ERP call inside a long PostgreSQL transaction unless current architecture explicitly requires that and the tradeoff is justified by existing project patterns. Prefer a narrow idempotent in-flight representation/response around the already durable posting aggregate.

Do not invent new business states.

### B2 — concurrency test must exercise the actual service path

Replace the synthetic/manual concurrency proof with a genuine test using **two independent real P1-014 `postShipment()` calls** against the same real eligible Shipment.

The test must prove real overlap and PostgreSQL-side behavior using independent DB/ORM sessions.

At minimum:

1. create one real `LABEL_GENERATED` or `OWN_TRANSPORT` Shipment with no posting intent;
2. run actual service operation A;
3. start actual service operation B before A's logical posting has fully settled;
4. prove the two operations genuinely overlap; when row locking is involved, capture exact PostgreSQL backend PIDs and `pg_stat_activity` / `pg_blocking_pids` (or equivalent exact database-side proof);
5. use deterministic adapter control so the test can keep A in-flight long enough to exercise B without fabricating a vendor API;
6. after both settle, use fresh independent reads to prove:
   - Shipment is non-regressive and reaches only the expected final/in-flight state,
   - exactly one `wms_outbound_shipment_postings` row exists for the Shipment,
   - exactly one initial posting attempt exists for the logical initial request,
   - exactly one `ShipmentPostingRequested` transition exists,
   - if acceptance is included, exactly one `ShipmentPosted` transition exists,
   - the ERP Testing adapter was invoked exactly once for that logical initial posting,
   - no duplicate orchestration/business effect exists,
   - operation B is handled idempotently, not by generic invalid-start-state failure.

A test where operation A manually mutates Shipment/posting/attempt rows is insufficient.

A bare `Promise.all()` without overlap/database proof is insufficient.

A unique-constraint-only test is insufficient.

### B3 — evidence and rendered UI proof are not literal final-head facts

#### Evidence schema mismatch

`05_EVIDENCE/P1-014_EVIDENCE.md` currently describes schema that is not the committed migration.

Examples that must be corrected from actual final Git/migration rather than copied from the old evidence:

- evidence currently names fields such as `posting_key`, `posting_status`, `last_attempt_number`, `last_status`, `last_attempted_at`; committed migration instead uses the actual `correlation_id`, `idempotency_key`, `status`, `attempt_count`, `last_attempt_at`, etc.;
- evidence claims FK constraints that the committed migration does not create;
- evidence claims an attempt idempotency unique index that the committed migration does not create;
- evidence uses `WMS_DATA_CORRECTION_REQUIRED` while the committed adapter type uses `WMS_DATA_DEFECT`.

After remediation, write only what the **final committed migration/entities/code actually contain**. If the remediation legitimately changes schema, evidence must describe the changed final schema exactly and include the migration/diff that made it true.

Do not add schema solely to make old evidence wording true; change product schema only if independently required by the P1-014 behavior/integrity contract.

#### Rendered UI Journey D gap

The original P1-014 guide required rendered UI/boundary proof that:

- unauthorized user cannot perform retry/give-up;
- no user action can add Shipment to a manifest in P1-014;
- timeout/provider fiction is not presented as ERP business rejection.

Current Playwright Journey D only proves authorized Supervisor give-up from `POSTING_ERROR`.

Correct the P1-014 Playwright suite so final rendered evidence explicitly covers the missing authorization/boundary behavior. You may keep the existing Supervisor give-up journey and add a separate journey, or restructure Journey D, but final evidence must state the literal final count and what each journey proves.

Use real authenticated product roles; do not prove UI authorization only by calling service methods directly.

## Preserve accepted P1-014 behavior

Do not regress these initial-shot behaviors while fixing the blockers:

- `LABEL_GENERATED/OWN_TRANSPORT -> POSTING_PENDING` only;
- explicit ERP acceptance -> `POSTED`;
- explicit structured ERP rejection -> `POSTING_ERROR`;
- timeout/no response remains a technical incident and does **not** become business `POSTING_ERROR`;
- retry from `POSTING_ERROR` is a real server-authoritative Warehouse Supervisor decision;
- Supervisor give-up is only `POSTING_ERROR -> CANCELLED`;
- non-Supervisor retry/give-up fails closed;
- packed physical contents remain untouched by posting/retry/rejection;
- Testing adapter/contract seam remains honest — no claim of `REAL ERP`;
- no `POSTED -> IN_MANIFEST` in P1-014.

## Mercato corrective branch rule

Continue only on:

`outbound/p1-014`

Starting head must be exactly:

`d20d95a097f9fe4b4459d6829fbd459c80a5efd3`

Create the narrowest corrective commit(s) directly on top.

No rebase, merge, squash, branch replacement, unrelated cleanup or P1-015 work.

## Required final-head verification

After the correction, rerun on the exact final Mercato head:

- full P1-014 PostgreSQL suite, including the corrected genuine actual-service concurrency proof;
- P1-013 PostgreSQL regression — expected accepted baseline `15/15` unless current legitimate committed count differs and is explained;
- P1-012 PostgreSQL regression — `14/14`;
- P1-011 PostgreSQL regression — `18/18`;
- FND-002 state-transition invariant — accepted `77/77` unless a legitimate additive P1-014 transition test changes only the test inventory and is explained;
- FND-002 transaction suite — current accepted `8/8` if still applicable;
- any shared `wms_orchestration` regression affected by the corrective implementation;
- fresh P1-014 Playwright against canonical Testing Mercato, served from the exact final Mercato Git revision.

If the final P1-014 test count changes from `18/18`, record the exact final count and literal test inventory. Do not preserve `18/18` by deleting/relabeling required coverage.

## Corrected evidence

Update only:

`WMS_Outbound/05_EVIDENCE/P1-014_EVIDENCE.md`

Evidence must be rebuilt from final Git/runtime facts and include:

- exact accepted P1-013 base;
- exact pre-remediation `d20d95a...` and final remediation Mercato head;
- exact branch/parent/merge-base/commit count;
- exact Git-derived final compare/file stats;
- exact committed migration/schema/index/FK facts — no stale names or invented constraints;
- exact adapter mode and final rejection category vocabulary;
- exact idempotency/in-flight duplicate behavior;
- literal final P1-014 PostgreSQL test inventory/count;
- actual-service concurrency proof with exact PIDs/database-side overlap evidence and fresh-read exactly-once assertions;
- exact `ShipmentPostingRequested` / `ShipmentPosted` transition counts from the concurrency/acceptance proof;
- exact adapter invocation count for the duplicate initial request proof;
- real rollback proof;
- literal regression outputs/counts;
- literal final Playwright journeys/count, including unauthorized rendered UI boundary;
- exact served Testing Mercato SHA;
- frozen Scanner SHA;
- final clean worktree statuses;
- explicit exclusions and honest `Testing stub/contract`, never `REAL ERP` unless an independently approved real ERP endpoint is actually authorized and evidenced.

Evidence remains **candidate evidence only**. Do not write `FINAL PASS`, `Owner Accepted`, or otherwise change steering.

## WMS_Outbound allowed change

For the executor corrective shot, the only WMS_Outbound file it may modify is:

`05_EVIDENCE/P1-014_EVIDENCE.md`

This supervisor-created remediation guide is already present before executor start.

Do **not** edit:

- `STATE.md`,
- any handover,
- task catalog / implementation plan,
- traceability,
- workflow,
- Devaxonic-WMS `.ai/*`,
- Drive steering.

## Scanner / environment boundary

- Scanner remains frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- No Scanner branch/change for P1-014.
- Testing PostgreSQL / Testing Mercato only.
- No Production/Demo changes.
- No secrets in evidence.

## Scope exclusions remain hard

Do not implement:

- P1-015 CarrierManifest lifecycle,
- P1-016 settlement,
- P2 crossdock GR gating,
- Return Receipt,
- external carrier behavior,
- unrelated orchestration refactors,
- Scanner changes.

## Two-strikes / STOP

This is the **one genuine corrective attempt** for the material P1-014 concurrency/evidence path.

After implementation + corrected evidence are pushed, **STOP**.

Do not start P1-015, do not merge, do not update steering, do not claim acceptance.

If the same material concurrency/evidence failure remains after this corrective shot, STOP under the two-strikes rule. No further remediation without explicit owner authorization.

Owner-facing result should remain microscopic: `done` plus exact refs/counts only.
