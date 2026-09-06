# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-07 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.  
Formal progress: **32/37 FINAL PASS / Owner Accepted**.

Latest Owner-Accepted checkpoint:

- `X-001` — item 32/37 — **FINAL PASS / Owner Accepted** — Mercato `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` / Scanner frozen `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8fcd97942545d9e49bcd819f2898dd8887843165`.

Earlier accepted checkpoints remain durable in Git/evidence history.

## X-002 — Supervisor FINAL PASS, Owner Acceptance pending

`X-002 — Integration correlation, observability and operational recovery` — Task Catalog item **33/37**.

Supervisor independently verified:

- Mercato `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`;
- exact lineage from accepted X-001 base `b56515ecffba729c828f5fe9ca0ec6471edc4c43`;
- only the intended INT-02 service/test surface changed across X-002;
- Scanner remains exactly frozen at `a2759a29347285dd1dcd14bf51633431fbf2a302`;
- evidence `05_EVIDENCE/X-002_EVIDENCE.md` at WMS_Outbound `71e8fd8c805357952fff83a0f1d7b5d915efc953`;
- P2 **99/99**, P1/P3/P4/fnd **80/80**, p2-004 **18/18**, typecheck clean;
- INT-02 source-TU completion race is serialized by the source-TU row lock before state re-read;
- decisive real PostgreSQL concurrency evidence uses participant backend PIDs, `pg_stat_activity`, `pg_blocking_pids()` and `wait_event_type = 'Lock'`;
- explicit Outbound -> Inbound settlement mapper/test proves the external contract is exactly `sourceInboundTuId + confirmedQty + damagedQty`; `residualQty` remains internal reconciliation only;
- no GR retry ownership transfer, automatic ERP retry, generic integration bus, Scanner change, or ACC work.

**Formal progress remains 32/37 until explicit Owner Acceptance of X-002.**

## Exact next execution scope — raw pg SSL maintenance gate grounded/prepared

Mandatory non-catalog gate:

`Post-X-002 raw PostgreSQL SSL maintenance gate`

Detailed guide:

`06_AGENT_GUIDES/POST_X002_RAW_PG_SSL_MAINTENANCE_EXECUTION.md` @ grounding commit `cf159a799ffa428decb645264492aa50413de39d`

Primary plan:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

Launch prerequisite: X-002 Supervisor FINAL PASS **and explicit Owner Acceptance**.

After that prerequisite, execute from:

- Mercato base `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9` -> `outbound/post-x002-raw-pg-ssl`;
- Scanner frozen `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.

### Maintenance objective

Create one canonical **test-only** raw PostgreSQL observer helper/factory so Testing `DATABASE_URL` SSL parameters cannot reintroduce the `self-signed certificate in certificate chain` failure by overriding the explicit approved Testing SSL option.

Known final-head raw observer call sites to migrate:

1. `p1-011-postgres.integration.test.ts`
2. `p1-014-erp-posting-postgres.integration.test.ts`
3. `p1-015-manifest-lifecycle-postgres.integration.test.ts`
4. `p1-016-final-settlement-postgres.integration.test.ts`
5. `p2-004-crossdock-recovery-postgres.integration.test.ts` — added by the X-002 concurrency correction.

Executor must also search the full WMS Outbound test tree for equivalent `pg.Client`/`new Client`/manual `sslmode` patterns and migrate all equivalent Testing observer constructions or document a concrete intentional exception.

The helper must:

- use runtime Testing `DATABASE_URL` only;
- centralize SSL normalization without printing/persisting secrets;
- preserve each suite's current direct-vs-pooler routing semantics;
- remain test-only;
- not alter application DB config or business behavior.

### Required proof

- smallest real canonical Testing PostgreSQL helper connection proof;
- all migrated directly affected concurrency suites green with their decisive PostgreSQL-side evidence preserved;
- post-change static search proves duplicated equivalent raw observer construction/manual SSL normalization is gone or explicitly justified;
- Mercato typecheck clean;
- evidence in `05_EVIDENCE/POST_X002_RAW_PG_SSL_MAINTENANCE_EVIDENCE.md`.

No full Outbound acceptance sweep belongs here. No ACC-001, UI/Playwright, Scanner, credential rotation, local PostgreSQL, Demo/Prod, application DB config or business logic change.

### Testing hygiene

This gate is non-catalog. `.ai/OPERATIONS.md` mandates the deep Testing reset before a new WMS Outbound **Task Catalog item**; do not automatically apply that reset to this maintenance gate.

Canonical Testing environment and secret-handling rules still apply.

## Completion boundary

Executor COMPLETE is not Supervisor FINAL PASS. Supervisor must independently verify remote branch/head lineage, diff scope, helper implementation, static search, exact Testing evidence, typecheck, evidence file and frozen Scanner.

After this maintenance gate later receives Supervisor FINAL PASS, STOP. `ACC-001` starts only after explicit Owner authorization.

Required sequence:

`X-002 -> raw pg SSL maintenance gate -> ACC-001 -> ACC-002 -> ACC-003 -> ACC-004`

The gate is non-catalog and does not change the 37-item count.

## Durable operating rules

- Executor `COMPLETE` != Supervisor FINAL PASS != Owner Acceptance.
- Owner controls executor/venue/launcher/session mechanics.
- Detailed execution logic belongs in Git guide; owner-facing launcher prompt stays microscopic.
- Ordinary in-scope implementation/test/runtime/tooling failures are executor-owned self-repair.
- Provider/session/quota interruption preserves exact checkpoint; never restart from stale accepted base.
- Testing credentials frozen; canonical Testing only; no local PostgreSQL.
- Inbound remains CLOSED / REFERENCE; Demo/Prod require explicit Owner authorization.

**Git truth overrides stale Drive/chat history.**
