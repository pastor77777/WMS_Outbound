# WMS Outbound — STATE

**As of:** 2026-09-05  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **22/37 items FINAL PASS**

## Architect baseline

Active process set:

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19

Requirements: **109 IDs = 98 FR + 6 INT + 5 CON**.

`PickWave` is out of scope v1. No separate Process 5 exists.

## Accepted implementation checkpoints

1. `FND-001` — FINAL PASS — `a3c8d67f3ec65967fd3405b3da8c9901fb46e192`
2. `FND-002` — FINAL PASS — `d835dc5ec157cb0485cb09cfbc0dfdeedd40c281`
3. `FND-003` — FINAL PASS — `8fa0214d36308b00aacf32b5df369ef486975ae2`
4. `P1-001` — FINAL PASS — `435b51007e5411ebdbb1b3b4d30c984fd770d4c6`
5. `P1-002` — FINAL PASS / Human Verified — `7510ef3f05b6c64c3f9de925a5a85f644913cdfe`
6. `P1-003` — FINAL PASS / Human Verified — `bae31c2c2ad9b1426b868d6df7a05076669ace0d`
7. `P1-004` — FINAL PASS — `71b74b5384b3fbfc55d8ed298f1a3715dc477c3c`
8. `P1-005` — FINAL PASS / Human Verified — Mercato `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975` / Scanner `8199b330cb739a45e2c615a3f2aa3803336be724`
9. `P1-008` — FINAL PASS / Human Verified — Mercato `9512137702a5d5f5b41910c2de97cf03321a1ccd` / Scanner `b5cfb59987c76f39e0ab48af67a52e2e914d9613`
10. `P1-006` — FINAL PASS / Human Verified — Mercato `353a5001cb8f1941971f960e509a8af643e41e5a` / Scanner `7596b7802e7ed55a59dd6dc1f21912ea6331e796` / evidence `43cc7d0e7dd20a48fc00b40150b30275d0c2aa12`
11. `P1-007` — FINAL PASS / Owner Accepted — Mercato `134db31381b4db726cd550abe6ecd4079ac21d8c` / Scanner `b23325aae1c4f83b79d01b3650dbead3486a1041` / evidence `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8`
12. `P1-009` — FINAL PASS / Owner Accepted — Mercato `5d780dabeb605bc657bb521bd2b2fdcc2e516f77` / Scanner `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `6c78a97ece567d90d3cb7d0580bb38669c9f9722`
13. `P1-010` — FINAL PASS / Owner Accepted — Mercato `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b5bb6429717402e0fb6969f7437ddaf673a8a174`
14. `P1-011` — FINAL PASS / Owner Accepted — Mercato `20887f2d74928cf69f447fdd6af20a612f38387c` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `90cc30fc2db15c40d80ef69cb03ffb1e107b51dc`
15. `P1-012` — FINAL PASS / Owner Accepted — Mercato `5019a20be14549ff8cbbf25af5bc61c56888e9e1` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b28f59e7ff41ac6d0a3be4b841410650bc5acd8b`
16. `P1-013` — FINAL PASS / Owner Accepted — Mercato `5e6b70aa81afd28fe3217e4aad216e8a6482a769` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `826b9c477fa86a44a93606265868730e4570ff90`
17. `P1-014` — FINAL PASS / Owner Accepted — Mercato `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b97f1640621b5b01571efec313b7fa0325c1aedf`
18. `P1-015` — FINAL PASS / Owner Accepted — Mercato `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `f201bd0beb2411b4f87f28ca6562a4fc11e6a249`
19. `P1-016` — FINAL PASS / Owner Accepted — Mercato `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `50c5664fda12caf5c2f7bcdb0e9e86c3495a01c2`
20. `P2-001` — FINAL PASS / Owner Accepted — Mercato `8a264fff5c2ca665294d1e02df90c6f37554fe7f` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `941d614966b1d2197d8654b3af924afb6ab14d58`
21. `P2-002` — FINAL PASS / Owner Accepted — Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93` / Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440` / evidence `2074e2541b7c28cf3c6031cb48ad68901333625a`
22. `P2-003` — FINAL PASS / Owner Accepted — Mercato `db0ef671b58ab13c2c0685205fbadcae1e1cf628` / Scanner `2ae72fb00db882fecae659b842e91efed17f949f` / evidence `f985d6099bdff939a0471012a25126baa8e216c2`

## P2-003 accepted boundary

Preserve these accepted behaviors:

- real Scanner normal `OK` crossdock execution starts task/line/order state on first successful placement;
- target Outbound TU is lazy until first successful placement;
- valid 1:1 GS1 source SSCC inherits TU_NUMBER + SSCC;
- invalid/non-GS1 1:1 and all n:n cases use generated outbound TU identity;
- placement quantity is exact decimal and idempotent;
- `confirmedQty` never exceeds `plannedQty`;
- physical-full seals the current target while the same task remains `IN_PROGRESS` and may continue into a new target;
- completion is replay-safe and packs the line without creating Allocation;
- Journey A 1:1 and Journey B n:n/physical-full passed through real Scanner UI with zero route mocks and persisted reconciliation;
- P2-003 does not implement shortage, damage, empty source, cancellation recovery, GR gate, Shipment/ERP, P3 or P4.

Accepted P2-003 proof:

- dedicated canonical PostgreSQL suite **8/8**;
- all **18 substantive behaviors** explicitly mapped and asserted;
- P2-002 regression **22/22**;
- P1-008 PostgreSQL **22/22** and real Scanner identity **1/1**;
- P2-001 regression **10/10**;
- P2-002 Scanner assignment regression **1/1**;
- Mercato typecheck **19/19 packages**;
- real P2-003 Scanner Playwright **2/2**, zero route mocks;
- final canonical Mercato runtime on port 3009 with `/login` HTTP 200 and non-empty production manifests;
- final canonical Scanner runtime on port 8081 HTTP 200;
- clean final product worktrees and pushed `outbound/p2-003` branches.

## Current position

Completed and accepted: **22/37**.

Next authorized implementation item:

**P2-004 — Crossdock shortage, damage, empty-TU and cancellation recovery — item 23/37.**

Authoritative executor guide:

`06_AGENT_GUIDES/P2-004_EXECUTION.md`

Frozen accepted bases for P2-004:

- Mercato `db0ef671b58ab13c2c0685205fbadcae1e1cf628`;
- Scanner `2ae72fb00db882fecae659b842e91efed17f949f`;
- P2-003 evidence `f985d6099bdff939a0471012a25126baa8e216c2`.

P2-004 objective:

- confirmed shortage;
- `DAMAGED` quantity;
- unexpected SKU;
- empty source TU;
- in-progress cancellation;
- residual/finalization rules;
- `allowPartialShipment` policy outcomes;
- exact `confirmed + damaged + residual` conservation and no orphan active target/task.

Requirements:

`FR-P2-06`, `FR-P2-07`, `FR-P2-08`, `FR-P2-09`, `FR-P2-10`, `FR-P2-11`, `FR-P2-12`, `FR-P2-15`, `FR-P2-19`, `FR-P5-12`, `FR-P5-13`, `FR-P5-14`, `FR-P5-15`.

Acceptance mapping:

`TC-021`, `TC-023`, `TC-024`, `TC-025`, `TC-026`, `TC-027`, `TC-035`, `TC-039`, `TC-067`, `TC-081`, `TC-121`.

## Mandatory new-item deep Testing reset

Before the first P2-004 implementation action, run the canonical deep reset:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

This is required before each **new** Task Catalog item. It is not used for retries/continuations inside one item. The reset must not downgrade canonical Supabase schema or rewrite accepted product history.

## Test/evidence guardrails for all remaining items

1. Exact served runtime/revision before Playwright; source SHA alone is not runtime proof.
2. Mercato route/product changes require repository-native full build/generate, canonical service restart, production-manifest proof and route probe.
3. Scanner changes require fresh canonical `scanner-testing.service` export/restart and port 8081 proof.
4. Scanner Playwright waits for real warehouse readiness and uses the current rendered UI contract.
5. Diagnose route/API/persistence before selectors on 404/5xx/API-null failures.
6. Fixtures explicitly set org/tenant/warehouse/role/zone/policy prerequisites and restore shared state.
7. Local PostgreSQL is forbidden; canonical DB is Supabase Testing project `yzonugcenguvmojwiihb`.
8. Shared/Inbound regressions are required only for actually touched shared primitives.
9. Real user-facing flows use zero route mocks/interception and persisted reconciliation.
10. Executor evidence never self-declares FINAL PASS / Owner Accepted / Human Verified.
11. Two genuine attempts on one material technical path then STOP unless Owner authorizes a distinct path.
12. Demo/Prod remains out of scope unless explicitly authorized.

## Authority and workflow

For Outbound behavior: Architect Source/Canon → traceability/task docs → current code/DB as implementation evidence → implementation plan as delivery decomposition.

Inbound remains **CLOSED / REFERENCE**.

Current authoritative handover: `08_HANDOVER/HANDOVER_CURRENT_2026-09-05.md`.

Detailed executor instructions live in Git; owner-facing launch prompts stay microscopic. The Owner controls executor selection, launch and session organization.