# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-06 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Formal progress: **31/37 FINAL PASS / Owner Accepted**.

Latest accepted checkpoints:

- `P4-001` — catalog item 29/37 — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`.
- `P3-003` — catalog item 28/37 — Mercato `f600782496e865603200d46d7e6041a54f90b9a4` / Scanner `135d86e1342bae7b21a8b676b1ef220a39a0f0b5` / evidence `40d4fd10430ab2b58457f96573ed34af8f57d316`.
- `P4-002` — catalog item 30/37 — Mercato `d75dccbc7a43b64c2a0ed325d30a7a4b49b57eb1` / Scanner `7d13e34fc66fe19149b43a6747b5300c2bdcf945` / evidence `0961a7fe2e08395cd1b2522e30770d62c2fb2841`.
- `P4-003` — catalog item 31/37 — **FINAL PASS / Owner Accepted** — Mercato product candidate `6ddec6870b5498901224b3457f5955101204a2f0` / Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8a916dfc9387b09d92b9e9143010cc34145108bd`.

## P4-003 accepted truth

- PutBack destination submission performs the authoritative server-side `IN_PROGRESS -> LOCATION_VALIDATION` transition.
- Invalid/non-storage destination returns to `IN_PROGRESS` with zero Inventory movement; retry is unlimited and there is no automatic escalation.
- Valid completion atomically performs `LOCATION_VALIDATION -> COMPLETED`, exact physical Inventory recovery, shared task-lock release and resolution of the corresponding P4-001 physical-return handoff.
- The handoff remains unresolved/protecting quantity before completion and on reject/error/rollback; `resolved_at` disables that temporary subtraction only once completion succeeds.
- Duplicate completion is idempotent; real concurrent completion is serialized by PostgreSQL locking; forced rollback leaves task, Inventory, task lock and handoff resolution unchanged.
- P4-002 FIFO/no-zone/single-active-task behavior remains preserved.
- No Inbound Putaway business states/flows were imported.
- Dedicated PostgreSQL: **10/10 PASS**.
- Mandatory targeted regressions: **133/133 PASS**.
- Rendered P4-003 + P4-002 regression: **5/5 PLAYWRIGHT VERIFIED**, zero route mocks.

## Post-acceptance maintenance completed

The two concrete test-infrastructure gaps discovered during P4-003 verification were fixed separately after Owner authorization:

- Mercato `outbound/p4-003` maintenance commit `9a656bf9e5a42e29b6493f1b7bed5d62b7bf562c` adds `WmsOutboundPutBackTask` to the isolated MikroORM entity registries in exactly:
  - `p3-002-postgres.integration.test.ts` — **17/17 PASS**;
  - `p1-003-detail-api-postgres.integration.test.ts` — **1/1 PASS**.
- WMS tracking record updated in `04_CURRENT_STATE/TEST_INFRA_GAPS.md` @ `6cde5bde465a1a4e10d41d9e04ccff443720c72c`.
- No product code changed in this maintenance fix.

The undiagnosed failures from the earlier abandoned full-directory sweep remain only as a recorded future integration-test concern; they were not part of this narrowly authorized maintenance correction.

## Current execution boundary

**Do not start the next Task Catalog item automatically.**

Before the next item:

1. refresh current Git;
2. load current `fetch_me_prompt` + `operational-mode` for a fresh Claude execution session;
3. load current project steering and relevant skills (`wms-outbound`, `architecture-context`, plus `scanner-context` when Scanner work requires it);
4. identify and ground the exact next Task Catalog item from current authority;
5. after Owner authorization, perform the canonical new-item Testing reset and require `RESET_OK`;
6. write/update the detailed execution guide and use a microscopic Owner-facing handoff.

## Durable operating rules

- Executor `COMPLETE` != Supervisor PASS != Owner Acceptance.
- Owner controls executor/venue/launcher/session mechanics.
- Provider/session/quota interruption preserves branch/HEAD/workspace checkpoint.
- Testing credentials are frozen; do not audit/rotate/refactor them during implementation.
- Inbound remains CLOSED / REFERENCE.
- Demo/Prod remain out of scope unless explicitly authorized.

**Git truth overrides stale Drive/chat history.**
