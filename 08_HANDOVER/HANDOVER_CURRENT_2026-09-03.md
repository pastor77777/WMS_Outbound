# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-03 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 implementation/acceptance items**, **109/109 Architect requirements mapped**.

Current progress: **11/37 FINAL PASS**.

Accepted checkpoints:

1. `FND-001` — `a3c8d67f3ec65967fd3405b3da8c9901fb46e192`
2. `FND-002` — `d835dc5ec157cb0485cb09cfbc0dfdeedd40c281`
3. `FND-003` — `8fa0214d36308b00aacf32b5df369ef486975ae2`
4. `P1-001` — `435b51007e5411ebdbb1b3b4d30c984fd770d4c6`
5. `P1-002` — `7510ef3f05b6c64c3f9de925a5a85f644913cdfe` — Human Verified
6. `P1-003` — `bae31c2c2ad9b1426b868d6df7a05076669ace0d` — Human Verified
7. `P1-004` — `71b74b5384b3fbfc55d8ed298f1a3715dc477c3c`
8. `P1-005` — Mercato `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975` / Scanner `8199b330cb739a45e2c615a3f2aa3803336be724` — Human Verified
9. `P1-008` — Mercato `9512137702a5d5f5b41910c2de97cf03321a1ccd` / Scanner `b5cfb59987c76f39e0ab48af67a52e2e914d9613` — Human Verified
10. `P1-006` — Mercato `353a5001cb8f1941971f960e509a8af643e41e5a` / Scanner `7596b7802e7ed55a59dd6dc1f21912ea6331e796` / evidence `43cc7d0e7dd20a48fc00b40150b30275d0c2aa12` — Human Verified
11. `P1-007` — Mercato `134db31381b4db726cd550abe6ecd4079ac21d8c` / Scanner `b23325aae1c4f83b79d01b3650dbead3486a1041` / durable evidence `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8` — FINAL PASS / Owner Accepted based on reviewed `PLAYWRIGHT VERIFIED` + real PostgreSQL evidence.

## P1-007 final accepted proof

Final evidence is `05_EVIDENCE/P1-007_EVIDENCE.md` at WMS evidence commit `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8`.

Accepted evidence surface:

- P1-007 genuine PostgreSQL suite: **20/20**;
- real Mercato Supervisor Playwright: **6/6** outcomes;
- real Scanner P1-007 short-pick/replacement journey: **1/1**;
- P1-004 regression: **11/11**;
- P1-005 regression: **10/10**;
- P1-006 backend: **12/12**;
- P1-006 Scanner retry/picking: **2/2**;
- P1-001: **21/21**;
- P1-003: **15/15**;
- P1-008: **22/22**;
- Inbound/shared compatibility: **17/17**;
- full `wms_outbound`: **245/245**.

Decisive P1-007 migration proof uses the genuine MikroORM `Migrator` with real migration history in an isolated PostgreSQL schema:

- 10 preceding migrations applied through `migrator.up({ to: ... })`;
- remediation UP;
- remediation DOWN;
- remediation re-UP;
- fresh constraint checks at each state;
- incompatible zero-quantity DOWN fails before mutation;
- zero rows remain unchanged and remediation remains recorded as applied;
- no direct migration-class replay, `getQueries()` loops, manual shadow-table DDL or broad cleanup workaround.

Decisive concurrency/atomicity proof includes distinct real PostgreSQL PIDs, `pg_blocking_pids` / lock evidence and real rollback with fresh independent read.

## P1-007 business boundary to preserve

- R42/R43: SHORT_ALLOCATED partial and Supervisor paths.
- R44/R45: source blocking, retry hierarchy and two-bound replacement availability (per-location + warehouse-wide hard reservations including pre-PickTask reservations).
- R46: WAIT, exact-picked correction and true zero cancellation.
- R47: persistent exact-`CustomerOrder` `allowPartialShipment = true` with mandatory reason.
- R48: backend shortage seam only; no P1-010 packing/repack UI.
- P1-007 persists durable physical-return handoff for already picked goods. Actual PutBackTask/RF lifecycle belongs to future P4 and must not be retrofitted into P1-007.
- Supervisor canonical idempotency: exact replay succeeds; same key with different canonical payload fails closed.

## Next item

Next planned implementation item:

**P1-009 — Direct Pack declaration and automatic sealing — item 12/37.**

Task Catalog:

- objective: immutable directPack declaration at the first architect-defined scan and automatic qualification/sealing when issue criteria allow;
- Architect: P1 R15–R18;
- requirements: `FR-P1-08`, `FR-P1-09`;
- dependencies: `P1-006`, `P1-008` — satisfied;
- target components: TU, OutboundOrderLine, Scanner picking;
- acceptance: `TC-001`, `TC-004`, `TC-005`;
- DoD: declaration cannot change later; successful direct pack reaches sealed/PACKED states without hidden manual packing step.

Hard boundary for P1-009:

- do not absorb P1-010 packing/repack/consolidation/discrepancy UI;
- preserve accepted P1-006 normal picking and P1-008 TU issueability/capacity/identity semantics;
- no Shipment/Carrier/labels/ERP/manifest work.

## Architecture/source routing

For Outbound business behavior:

1. current `01_ARCHITECT_SOURCE` / faithful Architect translations;
2. `02_CANON` and exact traceability/task sources;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Installed `wms-outbound` skill routes Outbound work to this chain.

Installed `architecture-context` is **reference-only** for accepted Inbound/shared Inventory/TU/warehouse/lock/orchestration compatibility. It does not override Outbound R-rules or justify expanding a ticket into later scope.

Inbound remains **CLOSED / REFERENCE** except targeted shared regression when a current Outbound diff touches shared primitives.

## Mandatory operating workflow

Read and follow:

`06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

Canonical behavior:

1. Supervisor grounds the current item and writes the detailed Antigravity execution/remediation guide to `WMS_Outbound/06_AGENT_GUIDES/` **and pushes it first**.
2. Owner receives only a short launch prompt telling the SAME Antigravity session to sync `WMS_Outbound/main`, read the local guide, execute/push and STOP.
3. Owner normally returns only `done`.
4. Supervisor independently fetches current Mercato/Scanner/WMS refs, diffs, tests/evidence and verifies them. Never ask the owner to paste long executor logs.
5. `SHOT -> RECEIVE -> VERIFY EVIDENCE -> NEXT SHOT`.
6. Same material path: two strikes then STOP unless the owner explicitly overrides. Each override permits one narrow further shot.
7. Existing Antigravity session only; preferred wrapper `/home/ubuntu/.local/bin/agy-pl`, never bare `agy`.

## Evidence rules that must not be lost

- real PostgreSQL only for DB claims; no fake EM/Map/simulated PG errors;
- real concurrency = independent overlapping operations/transactions plus DB-side evidence tied to actual participants;
- real rollback = real write/flush/failure-before-commit + fresh independent read;
- real migration claim = project-approved MikroORM Migrator path when available;
- real UI claim = decisive action through rendered UI;
- automated browser evidence label = `PLAYWRIGHT VERIFIED`;
- executor narrative is never acceptance evidence by itself.

## Fresh-session startup

A fresh supervisor session should read/refresh, in this order:

1. this handover;
2. `STATE.md`;
3. `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`;
4. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact P1-009 section;
5. exact P1-009 Architect source / requirements / test scenarios;
6. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-03.md`;
7. `Devaxonic-WMS/.ai/STATE.md`, `.ai/PLAN.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`;
8. current installed `wms-outbound`, `architecture-context`, `fetch_me_prompt`, `operational-mode` and current real-evidence contract.

Then independently verify current Git refs before writing the first P1-009 guide.

Do not reconstruct mutable state or operating rules from old chat history.