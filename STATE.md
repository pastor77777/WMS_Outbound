# WMS Outbound — STATE

**As of:** 2026-09-06  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **31/37 items FINAL PASS / Owner Accepted**

## Architect baseline

Active process set:

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19

Requirements: **109 IDs = 98 FR + 6 INT + 5 CON**.

`PickWave` is out of scope v1. No separate Process 5 exists.

## Latest accepted implementation checkpoints

Earlier accepted checkpoints 1–26 remain unchanged in Git history.

27. `P3-002` — FINAL PASS / Owner Accepted — Mercato `84274acacfbfe0119e270ca5bfbcb723e47d7723` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`
28. `P4-001` — catalog item 29/37 — FINAL PASS / Owner Accepted — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`
29. `P3-003` — catalog item 28/37 — FINAL PASS / Owner Accepted — Mercato `f600782496e865603200d46d7e6041a54f90b9a4` / Scanner `135d86e1342bae7b21a8b676b1ef220a39a0f0b5` / evidence `40d4fd10430ab2b58457f96573ed34af8f57d316`
30. `P4-002` — catalog item 30/37 — FINAL PASS / Owner Accepted — Mercato `d75dccbc7a43b64c2a0ed325d30a7a4b49b57eb1` / Scanner `7d13e34fc66fe19149b43a6747b5300c2bdcf945` / evidence `0961a7fe2e08395cd1b2522e30770d62c2fb2841`
31. `P4-003` — catalog item 31/37 — **FINAL PASS / Owner Accepted** — Mercato product candidate `6ddec6870b5498901224b3457f5955101204a2f0` / Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8a916dfc9387b09d92b9e9143010cc34145108bd`.

## P4-003 accepted boundary

- RF PutBack supports destination submission and server validation: `IN_PROGRESS -> LOCATION_VALIDATION`.
- Invalid/non-storage destination returns `LOCATION_VALIDATION -> IN_PROGRESS`, with zero Inventory mutation, unlimited retry and no automatic escalation.
- Valid destination completes exactly once: `LOCATION_VALIDATION -> COMPLETED`, exact physical recovery into ordinary Inventory availability, shared active-task lock release, and physical-return handoff resolution in the same transaction.
- P4-001 protection remains active while the handoff is unresolved; after completion `resolved_at` removes that exact quantity from the temporary protection subtraction used by ATP/allocation/source availability.
- Duplicate completion is idempotent; genuine concurrent completion serializes on PostgreSQL row locking; rollback leaves task, Inventory, lock and handoff resolution unchanged.
- No Inbound Putaway business semantics were imported.
- Dedicated PostgreSQL P4-003: **10/10 PASS**.
- Mandatory targeted regression set: **133/133 PASS**.
- Rendered Scanner acceptance plus P4-002 rendered regression: **5/5 PLAYWRIGHT VERIFIED**, zero route mocks.

## Post-acceptance test-infra maintenance

Owner-authorized maintenance fixed the two known `WmsOutboundPutBackTask` MikroORM registration gaps discovered during P4-003 verification:

- Mercato maintenance commit `9a656bf9e5a42e29b6493f1b7bed5d62b7bf562c`.
- `p3-002-postgres.integration.test.ts`: **17/17 PASS**.
- `p1-003-detail-api-postgres.integration.test.ts`: **1/1 PASS**.
- Tracking record: `04_CURRENT_STATE/TEST_INFRA_GAPS.md` @ `6cde5bde465a1a4e10d41d9e04ccff443720c72c`.

This maintenance does not change the accepted P4-003 product behavior or Task Catalog count.

## Current position

Completed and accepted: **31/37**.

The next Task Catalog item has **not** been grounded or launched in this checkpoint. Ground it fresh from current Git and required skills before authorization.

## Executor / prompt workflow — durable rules

Detailed workflow: `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

Prompt skill routing: `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`.

- full Task Catalog item is the normal Owner-authorized executor unit when project steering says so;
- ordinary in-scope implementation/test/runtime/build/evidence failures remain executor self-repair;
- provider/session/quota interruption preserves exact checkpoint;
- executor prose is never acceptance; supervisor independently verifies remote Git/evidence;
- Owner acceptance is explicit after Supervisor FINAL PASS;
- Owner controls executor launch/session mechanics;
- before a fresh Claude execution session, load current `fetch_me_prompt` + `operational-mode` and then project steering/skills;
- designated Testing credential handling remains frozen during implementation.

## Authority and continuity

For Outbound behavior: immutable Architect Source -> faithful Canon/translation -> requirements/traceability -> Task Catalog delivery slice -> current code/DB as implementation evidence.

Inbound remains **CLOSED / REFERENCE**.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.
