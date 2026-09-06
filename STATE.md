# WMS Outbound — STATE

**As of:** 2026-09-07  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** pre-acceptance test-infrastructure maintenance  
**Formal implementation progress:** **33/37 items FINAL PASS / Owner Accepted**

## Architect baseline

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19
- requirements: **109 IDs = 98 FR + 6 INT + 5 CON**

Inbound remains **CLOSED / REFERENCE**. `PickWave` is out of scope v1. No separate Process 5 exists.

## Latest Owner-Accepted checkpoint

- `X-002 — Integration correlation, observability and operational recovery` — catalog item **33/37** — **Supervisor FINAL PASS / Owner Accepted** on 2026-09-07.
- Mercato `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`.
- Accepted base lineage: X-001 `b56515ecffba729c828f5fe9ca0ec6471edc4c43` -> X-002 final head above.
- Scanner frozen `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.
- X-002 evidence: `05_EVIDENCE/X-002_EVIDENCE.md` at WMS_Outbound `71e8fd8c805357952fff83a0f1d7b5d915efc953`.
- Final claimed/recorded checks: P2 **99/99**, P1/P3/P4/fnd **80/80**, p2-004 **18/18**, Mercato typecheck clean.
- Supervisor independently verified the two decisive acceptance corrections: real PostgreSQL-side blocking proof for INT-02 and an explicit three-field Outbound -> Inbound settlement contract excluding `residualQty`.

Earlier accepted checkpoints remain unchanged in Git/evidence history.

## Active execution scope — raw pg SSL maintenance gate

Mandatory non-catalog gate:

`Post-X-002 raw PostgreSQL SSL maintenance gate`

**Status:** grounded / Owner-authorized to execute; not yet executor-COMPLETE or Supervisor FINAL PASS.

Detailed execution guide:

`06_AGENT_GUIDES/POST_X002_RAW_PG_SSL_MAINTENANCE_EXECUTION.md`

Primary maintenance plan:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

Execution base:

- Mercato `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9` -> `outbound/post-x002-raw-pg-ssl`;
- Scanner remains frozen at `a2759a29347285dd1dcd14bf51633431fbf2a302`.

### Exact maintenance scope

Test infrastructure only:

- create one canonical test-only raw `pg.Client` observer helper/factory;
- consume runtime Testing `DATABASE_URL`, normalize SSL parameters centrally, never expose credentials;
- preserve existing direct/pooler routing semantics per suite;
- migrate at least the five known final-head observer call sites:
  - `p1-011-postgres.integration.test.ts`
  - `p1-014-erp-posting-postgres.integration.test.ts`
  - `p1-015-manifest-lifecycle-postgres.integration.test.ts`
  - `p1-016-final-settlement-postgres.integration.test.ts`
  - `p2-004-crossdock-recovery-postgres.integration.test.ts`
- search the complete WMS Outbound test tree for equivalent raw observer construction/manual `sslmode` normalization and migrate equivalent cases or document a concrete intentional exception;
- add the smallest real canonical Testing PostgreSQL connection proof for the helper;
- rerun all migrated directly affected concurrency suites with their decisive PostgreSQL-side evidence preserved;
- run Mercato typecheck;
- write `05_EVIDENCE/POST_X002_RAW_PG_SSL_MAINTENANCE_EVIDENCE.md`.

No product/business logic, application DB configuration, credential rotation, local PostgreSQL, Demo/Prod, Scanner, UI/Playwright, broad acceptance sweep or ACC-001 belongs in this gate.

### Testing/session boundary

This gate is explicitly **non-catalog**. The deep reset rule in `Devaxonic-WMS/.ai/OPERATIONS.md` applies before new WMS Outbound **Task Catalog items**, so there is no automatic deep reset before this maintenance gate.

A **fresh Claude session is appropriate and preferred** because this is a separate maintenance work unit rather than a continuation of X-002. The Owner controls the actual executor/session/launcher. A fresh Claude session must load current `fetch_me_prompt` + `operational-mode`, current steering, `wms-outbound`, and the maintenance guide; `scanner-context` is not required unless Scanner unexpectedly becomes materially relevant.

Canonical Testing only. DB-backed commands source the canonical Testing environment in the same invocation without printing secrets.

## Completion boundary

Executor COMPLETE for this maintenance gate is not Supervisor FINAL PASS. Supervisor independently verifies remote branch/head lineage, diff scope, helper implementation, pre/post static search, real Testing PostgreSQL proof, all affected concurrency suites, typecheck, maintenance evidence and frozen Scanner.

After this maintenance gate receives Supervisor FINAL PASS, STOP. Do not start `ACC-001` without explicit Owner authorization.

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
