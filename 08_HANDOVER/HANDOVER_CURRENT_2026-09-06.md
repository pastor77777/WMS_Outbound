# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-07 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.  
Formal progress: **33/37 FINAL PASS / Owner Accepted**.

Latest Owner-Accepted checkpoint:

- `X-002 — Integration correlation, observability and operational recovery` — item **33/37** — **Supervisor FINAL PASS / Owner Accepted**.
- Mercato `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`.
- Scanner frozen `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.
- X-002 evidence `05_EVIDENCE/X-002_EVIDENCE.md` at WMS_Outbound `71e8fd8c805357952fff83a0f1d7b5d915efc953`.
- Supervisor independently verified final lineage/diff and the two corrected INT-02 acceptance gaps: decisive PostgreSQL lock-blocking proof and explicit `sourceInboundTuId + confirmedQty + damagedQty` contract with no external `residualQty` field.

Earlier accepted checkpoints remain durable in Git/evidence history.

## Exact active execution scope — raw pg SSL maintenance gate

Mandatory non-catalog maintenance before returning to Task Catalog acceptance work:

`Post-X-002 raw PostgreSQL SSL maintenance gate`

**Status:** grounded and Owner-authorized; execution may start in a fresh Owner-selected Claude session. It is not yet executor COMPLETE or Supervisor FINAL PASS.

Detailed execution guide:

`06_AGENT_GUIDES/POST_X002_RAW_PG_SSL_MAINTENANCE_EXECUTION.md`

Primary maintenance plan:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

Execution base:

- Mercato `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9` -> `outbound/post-x002-raw-pg-ssl`;
- Scanner frozen at `a2759a29347285dd1dcd14bf51633431fbf2a302`.

### Maintenance objective

Create one canonical **test-only** raw PostgreSQL observer helper/factory so Testing `DATABASE_URL` SSL parameters cannot reintroduce the `self-signed certificate in certificate chain` failure by overriding the explicit approved Testing SSL behavior.

Known final-head raw observer call sites to migrate:

1. `p1-011-postgres.integration.test.ts`
2. `p1-014-erp-posting-postgres.integration.test.ts`
3. `p1-015-manifest-lifecycle-postgres.integration.test.ts`
4. `p1-016-final-settlement-postgres.integration.test.ts`
5. `p2-004-crossdock-recovery-postgres.integration.test.ts`

Executor must also search the complete WMS Outbound test tree for equivalent `Client` from `pg`, `new Client(...)`, `pg.Client`, and manual `sslmode` stripping. Equivalent Testing observer clients must use the helper; any intentional exception must be documented with exact rationale.

The helper must use runtime Testing `DATABASE_URL`, never expose credentials, preserve each suite's existing direct/pooler routing semantics, remain test-only and not modify application DB configuration.

### Required proof

- smallest real canonical Testing PostgreSQL helper connection proof;
- all migrated directly affected concurrency suites green with decisive PostgreSQL-side assertions preserved;
- post-change search proving duplicated equivalent raw observer construction/manual SSL normalization is removed or explicitly justified;
- Mercato typecheck clean;
- durable evidence in `05_EVIDENCE/POST_X002_RAW_PG_SSL_MAINTENANCE_EVIDENCE.md`.

No full acceptance sweep, ACC-001, Playwright/UI, Scanner, credential rotation, application DB config, local PostgreSQL or Demo/Prod belongs in this gate.

### Session/testing hygiene

A **fresh Claude session is the preferred execution boundary** because this gate is a separate maintenance work unit after accepted X-002. The Owner still controls the actual executor/session/launcher.

For a fresh Claude session: load current `fetch_me_prompt` + `operational-mode` first, then current steering + `wms-outbound` and this maintenance guide. `scanner-context` only if Scanner unexpectedly becomes materially relevant.

This gate is explicitly **non-catalog**. `.ai/OPERATIONS.md` mandates the deep Testing reset before a new WMS Outbound **Task Catalog item**; do not automatically apply that reset here. Canonical Testing and secret-handling rules remain mandatory.

## Completion / stop boundary

Executor COMPLETE is not Supervisor FINAL PASS. Supervisor must independently verify remote branch/head lineage, diff scope, helper contract, pre/post search, exact Testing DB evidence, directly affected concurrency suites, typecheck, maintenance evidence and frozen Scanner.

After this gate receives Supervisor FINAL PASS, STOP. `ACC-001` starts only on explicit Owner authorization.

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
