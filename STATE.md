# WMS Outbound — STATE

**As of:** 2026-09-05  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **21/37 items FINAL PASS**

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

## P2-002 accepted boundary

Preserve these accepted behaviors:

- accepted positive P2-001 binding truth is converted exactly once into CROSSDOCK OutboundOrder/Line + CrossDockPickTask;
- no Allocation exists in crossdock;
- task `plannedQty` is the authoritative quantity lock and exact binding quantity;
- channel is immutable CROSSDOCK and STANDARD/CROSSDOCK lines are not mixed in one OutboundOrder;
- grouping uses customer/address/priority/identical SLA and accepted allowPartial semantics;
- no eager physical target Outbound TU exists before first placement;
- Crossdock Scanner module assigns the next task with no zone selector, same warehouse ordering and one-active-warehouse-task protection;
- rendered Scanner shows task/source/planned quantity/status and stops before P2-003 execution controls;
- canonical Testing runtime may require repository-native `yarn build` including `yarn generate` + restart of `mercato-localhost.service` after route additions; source SHA alone is not proof of served route presence.

Accepted proof in `05_EVIDENCE/P2-002_EVIDENCE.md`:

- corrected architecture-aligned CON-03 PASS; P2-001 regression **10/10**;
- dedicated P2-002 PostgreSQL **22/22**;
- remaining mandatory Mercato regressions **5 suites / 41 tests**;
- current-UI P1-005 Scanner regression **1/1**, zero route mocks;
- dedicated P2-002 Scanner Playwright **1/1**, zero route mocks, real A/B/blocked request-task outcomes and persisted ownership/lock/allocation assertions;
- rebuild-only stale generated/runtime route recovery; no Mercato source change;
- clean final Mercato/Scanner worktrees.

## Current position

Completed and accepted: **21/37**.

Next authorized implementation item:

**P2-003 — RF crossdock sorting into outbound TUs — item 22/37.**

Task grounding:

- objective: implement the normal Scanner crossdock execution path from assigned task through source/SKU/quantity/quality OK placement into one or more target Outbound TUs, including lazy first target creation, physical-full sealing/new target continuation and task completion;
- requirements: `FR-P2-02`, `FR-P2-04`, `FR-P2-05`, `FR-P2-10`, `FR-P2-12`, `FR-P2-19`, `FR-P2-21`, `FR-P2-22`;
- dependencies: `P2-002`, `P1-008` — satisfied;
- source: P2 R3–R4, R7–R10, R19–R20, R23–R24, R34, R39–R40;
- target: Scanner crossdock journey, CrossDockPickTask execution service, Outbound TU placement/sealing;
- acceptance: `TC-020`, `TC-021`, `TC-022`, `TC-023`, `TC-027`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-035`, `TC-037`, `TC-099`, `TC-101`.

Key P2-003 boundaries:

- first real scan transitions task `ASSIGNED → IN_PROGRESS`, line `CREATED → PICKING`, first line start moves order `CREATED → PACKING_IN_PROGRESS`;
- physical target Outbound TU remains lazy until first successful placement;
- 1:1 with valid GS1 source SSCC inherits TU_NUMBER/SSCC; otherwise/n:n creates a new Outbound TU identity using accepted P1-008 TUSetup/Sequence rules;
- physical-full target may be sealed by RF and execution continues into a new target TU under the same task;
- task completion fixes `confirmedQty <= plannedQty` and must be idempotent;
- P2-003 covers normal OK execution only; shortage, DAMAGED, unexpected SKU, empty source TU, in-progress cancellation, residual/finalization exception handling belong to P2-004;
- no GR gate, Shipment/ERP/manifest, P3/P4 implementation in this item.

Authoritative executor guide:

`06_AGENT_GUIDES/P2-003_EXECUTION.md`

## Test/evidence guardrails for all remaining items P2-003 → ACC-004

1. **Exact runtime before Playwright.** Source SHA is not served-runtime proof. For every route/product change, build with the repository-native full build/generate path, restart only canonical Testing services, and prove the intended route/runtime is present before browser assertions.
2. **Real warehouse readiness.** Scanner Playwright must wait for real warehouse resolution/selection before entering a module. Do not use `Choose mode` alone as readiness.
3. **Current UI contract.** Ground assertions in the current tested SHA (`testID`, accessibility labels, current rendered copy). Historical accepted selectors/text are not immutable product requirements.
4. **Regression relevance.** Historical regression specs are evidence only when they still represent the current accepted contract and the new diff touches that surface. If a red spec is on untouched code, first compare current product/UI contract and fixture prerequisites before declaring regression.
5. **Architecture-first concurrency.** Prove the architect-required business effect (serialization, no duplicate assignment/effect, exactly-once state) using genuine overlapping PostgreSQL operations. Do not elevate incidental diagnostics such as one `wait_event` or `pg_blocking_pids()` shape into requirements unless Architect Source explicitly requires them.
6. **Diagnose by layer.** 404/5xx/API-null is diagnosed at route/runtime/API/persistence before changing UI selectors. If API is correct but rendering is wrong, then classify as Scanner/Mercato render defect.
7. **Deterministic fixtures.** Test identities must receive explicit org/tenant/warehouse/role/zone or policy prerequisites. Never rely on ambient Testing cardinality or defaults (for example “only one warehouse”). Preserve/restore shared configuration on every exit path.
8. **Canonical DB only.** Local PostgreSQL remains forbidden. DB-backed Jest/Playwright uses canonical Supabase Testing `yzonugcenguvmojwiihb` through the approved environment source.
9. **Shared regression only when touched.** Inbound/shared regressions are mandatory for actually modified Inventory/TU/warehouse/task-lock/orchestration primitives; do not add broad historical suites merely by habit.
10. **UI evidence designed with implementation.** Human-facing tasks must define their real rendered journey before implementation closeout, with real request/response and persisted reconciliation; do not bolt Playwright onto the end after backend completion.
11. **No evidence relabeling.** X-001/ACC-001..004 must reference exact current executable proof; old accepted evidence may be retained only when the relevant product surface/SHA semantics are unchanged and the evidence class remains truthful.
12. **Two-strikes rule remains.** Two genuine attempts on the same material technical path then STOP unless Owner explicitly authorizes a new narrow path.

## Remaining-plan risk notes

- `P2-003/P2-004`: highest Scanner/quantity risk — require real RF n:n placement and quantity conservation from the start.
- `P2-005/P2-006`: highest runtime/integration risk — route generation/runtime provenance and INT-02/03 correlation must be proven before UI gate assertions.
- `P3-001/P3-003`: route recovery strictly by authoritative `pickedQty`; pre-confirm physical race returns to exact source location and must not create PutBackTask.
- `P4-001..003`: logical cancellation is immediate; physical recovery is later; `pickedQty > 0` only; invalid-location loop has no retry limit/auto escalation.
- `X-001`: executable CON-01..05 proof must test business outcomes, not diagnostic implementation details.
- `ACC-001..004`: freeze exact app/runtime revisions and use normal UI; API/DB may prepare/verify fixtures but may not replace decisive human-facing actions.

## Authority and workflow

For Outbound behavior: Architect Source/Canon → traceability/task docs → current code/DB as implementation evidence → implementation plan as delivery decomposition.

Inbound remains **CLOSED / REFERENCE**. Run targeted regressions only for shared primitives actually touched.

Current authoritative handover: `08_HANDOVER/HANDOVER_CURRENT_2026-09-05.md`.
Current Devaxonic mirror: `.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-05.md`.

Detailed executor instructions live in Git; owner-facing prompts stay microscopic. The Owner controls executor/session mechanics. Testing credentials are designated Testing data; do not rotate or redesign credential handling as part of WMS feature work.