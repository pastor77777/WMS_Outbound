# WMS Outbound — STATE

**As of:** 2026-09-07  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation / pre-acceptance maintenance  
**Formal implementation progress:** **32/37 items FINAL PASS / Owner Accepted**

## Architect baseline

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19
- requirements: **109 IDs = 98 FR + 6 INT + 5 CON**

Inbound remains **CLOSED / REFERENCE**. `PickWave` is out of scope v1. No separate Process 5 exists.

## Latest Owner-Accepted checkpoint

- `X-001` — catalog item 32/37 — **FINAL PASS / Owner Accepted** — Mercato `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` / Scanner frozen `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8fcd97942545d9e49bcd819f2898dd8887843165`.

Earlier accepted checkpoints remain unchanged in Git history/evidence.

## X-002 — Supervisor FINAL PASS, Owner Acceptance pending

`X-002 — Integration correlation, observability and operational recovery` — catalog item **33/37**.

Supervisor FINAL PASS verified on remote Git:

- Mercato `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`;
- exact lineage from accepted X-001 base `b56515ecffba729c828f5fe9ca0ec6471edc4c43`;
- Scanner frozen `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`;
- evidence `05_EVIDENCE/X-002_EVIDENCE.md` at WMS_Outbound `71e8fd8c805357952fff83a0f1d7b5d915efc953`;
- P2 regression set **99/99**, P1/P3/P4/fnd set **80/80**, full p2-004 **18/18**, typecheck clean;
- INT-02 hardened for real overlapping source-TU finalization and has decisive PostgreSQL-side `pg_blocking_pids`/lock proof;
- explicit Outbound -> Inbound settlement contract is exactly `sourceInboundTuId + confirmedQty + damagedQty`; `residualQty` remains internal only;
- no GR retry ownership transfer, automatic ERP retry, generic integration bus, Scanner change, or ACC work.

**Formal count remains 32/37 until explicit Owner Acceptance of X-002.** Executor COMPLETE and Supervisor FINAL PASS do not substitute for Owner Acceptance.

## Exact next execution scope — raw pg SSL maintenance gate grounded/prepared, not launched

Mandatory non-catalog gate:

`Post-X-002 raw PostgreSQL SSL maintenance gate`

Detailed execution guide:

`06_AGENT_GUIDES/POST_X002_RAW_PG_SSL_MAINTENANCE_EXECUTION.md` @ grounding commit `cf159a799ffa428decb645264492aa50413de39d`

Primary maintenance plan:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

Launch prerequisite:

- X-002 must have Supervisor FINAL PASS **and explicit Owner Acceptance**.

Execution base after that prerequisite is satisfied:

- Mercato `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9` -> `outbound/post-x002-raw-pg-ssl`;
- Scanner remains frozen at `a2759a29347285dd1dcd14bf51633431fbf2a302`.

### Grounded maintenance scope

Test infrastructure only:

- create one canonical test-only raw `pg.Client` observer helper/factory;
- consume runtime Testing `DATABASE_URL`, normalize SSL parameters centrally, never expose credentials;
- preserve existing direct/pooler routing semantics per suite;
- migrate the four X-001-known call sites plus the fifth observer introduced by X-002:
  - `p1-011-postgres.integration.test.ts`
  - `p1-014-erp-posting-postgres.integration.test.ts`
  - `p1-015-manifest-lifecycle-postgres.integration.test.ts`
  - `p1-016-final-settlement-postgres.integration.test.ts`
  - `p2-004-crossdock-recovery-postgres.integration.test.ts`
- search the full WMS Outbound test tree for any other equivalent raw observer construction/manual `sslmode` normalization and route equivalent cases through the helper;
- add the smallest real Testing PostgreSQL helper connection proof;
- rerun all migrated directly affected concurrency suites and preserve their decisive PostgreSQL-side evidence;
- run Mercato typecheck;
- write `05_EVIDENCE/POST_X002_RAW_PG_SSL_MAINTENANCE_EVIDENCE.md`.

No business/product logic, application DB config, credential rotation, local PostgreSQL, Demo/Prod, Scanner, Playwright, or ACC-001 belongs in this gate.

### Testing hygiene

This gate is explicitly **non-catalog**. The deep reset rule in `Devaxonic-WMS/.ai/OPERATIONS.md` applies before every new WMS Outbound **Task Catalog item**; do not invent an automatic deep reset requirement for this maintenance gate.

Canonical Testing only. DB-backed commands must source the canonical Testing environment in the same invocation without printing secrets.

## Completion boundary

Executor completion of this gate still requires independent Supervisor verification of remote refs, diff, helper implementation, post-change search, exact Testing evidence and frozen Scanner head.

After eventual Supervisor FINAL PASS for the maintenance gate, STOP. Do not start `ACC-001` without explicit Owner authorization.

Required sequence remains:

`X-002 -> raw pg SSL maintenance gate -> ACC-001 -> ACC-002 -> ACC-003 -> ACC-004`

The maintenance gate does not change the 37-item count.

## Durable operating rules

- Executor `COMPLETE` != Supervisor FINAL PASS != Owner Acceptance.
- Owner controls executor/venue/launcher/session mechanics.
- Full authorized scope is the execution objective; ordinary in-scope failures are executor self-repair.
- Provider/session/quota interruption preserves exact branch/HEAD/workspace checkpoint.
- Testing credentials remain frozen.
- Canonical Testing only; no local PostgreSQL; Demo/Prod require separate Owner authorization.
- `architecture-context` is shared/Inbound technical reference only; Outbound Architect/Canon remains business authority.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.
