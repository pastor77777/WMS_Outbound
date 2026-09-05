# WMS Outbound — STATE

**As of:** 2026-09-05  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **20/37 items FINAL PASS**

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

## P2-001 accepted boundary

Preserve these accepted behaviors:

- Outbound consumes only Inbound `TU` type `ELEMENTARY` already at `IN_CROSS_DOCK`;
- binding demand is recalculated at that boundary against current `BACKORDERED` demand;
- `demandEligibleQty` deducts ATPReservation and all non-CANCELLED OutboundOrderLine `requiredQty` across channels;
- source capacity uses ASN declaration and keeps `ACTIVE`, `CONSUMED` and `DAMAGED` binding quantity deducted; only `RELEASED` returns unexecuted capacity;
- positive binding uses exact decimal arithmetic and deterministic idempotency;
- zero match records deterministic residual truth without creating P2-002 work;
- concurrency is serialized in real PostgreSQL with source/demand locks and no double assignment;
- accepted Inbound automatic cross-dock result harness explicitly controls/restores its `AUTOMATIC` prerequisite without changing Inbound business semantics;
- Scanner remained frozen.

Accepted proof in `05_EVIDENCE/P2-001_EVIDENCE.md`:

- dedicated P2-001 PostgreSQL **10/10**, substantively mapping all 20 required guide scenarios;
- literal PostgreSQL CON-03 backend PID / lock-wait evidence;
- focused regressions **4 suites / 38/38** including Inbound **3/3** on final head;
- accepted P1-016 base reproduced Inbound **1/3** under ambient `MANUAL`, proving the original failure was a test-harness prerequisite rather than P2-001 regression;
- durable evidence `941d614966b1d2197d8654b3af924afb6ab14d58`.

## Current position

Completed and accepted: **20/37**.

Next authorized implementation item:

**P2-002 — Crossdock OutboundOrder/Line and CrossDockPickTask planning — item 21/37.**

Task grounding:

- objective: create CROSSDOCK OutboundOrder/Line and exactly-once CrossDockPickTask assignments from accepted P2-001 binding truth, without Allocation;
- requirements: `FR-P2-02`, `FR-P2-03`, `FR-P2-04`, `FR-P2-16`, `FR-P2-21`, `FR-P2-22`, `FR-P2-25`, `CON-03`;
- dependencies: `P2-001`, `FND-002` — satisfied;
- source: P2 R3–R8, R29–R30, R39–R40, R43;
- target: CrossDock planner, CrossDockPickTask, OutboundOrder/Line and bounded RF module assignment;
- acceptance: `TC-020`, `TC-021`, `TC-022`, `TC-029`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-036`, `TC-092`, `TC-099`, `TC-101`, `TC-119`, `TC-134`.

Key P2-002 boundaries:

- `OutboundOrder.fulfillmentChannel = CROSSDOCK` is immutable and never mixed with STANDARD lines;
- grouping requires same customer/address/priority and identical `slaDeadline`; `allowPartialShipment = false` keeps grouping inside one CustomerOrder;
- every crossdock OutboundOrderLine is created with exactly one CrossDockPickTask; task `plannedQty` is the authoritative source quantity lock;
- replay/concurrency cannot double-plan P2-001 binding/source quantity;
- no `Allocation` exists in crossdock;
- physical target Outbound TU creation/numbering remains lazy until first placement and therefore belongs to P2-003;
- P2-002 may expose the Scanner crossdock module only through assignment/visibility: no zone selector, same warehouse ordering as PickTask, one active warehouse task per operator;
- P2-002 must not implement RF sorting/scans, quantity/quality confirmation, target TU placement/sealing, task completion, shortage/QC, GR gate or later P2 work.

Current executor guide:

`06_AGENT_GUIDES/P2-002_EXECUTION.md`

P2-002 Mercato branch starts from accepted P2-001 head:

`8a264fff5c2ca665294d1e02df90c6f37554fe7f`

Scanner starts from accepted frozen head if bounded assignment UI work is required:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

## Authority and workflow

For Outbound behavior: Architect Source/Canon → traceability/task docs → current code/DB as implementation evidence → implementation plan as delivery decomposition.

Inbound remains **CLOSED / REFERENCE**. P2-002 may consume accepted P2-001 truth and run targeted regressions, but must not reinterpret accepted Inbound qualification/deintegration/transport semantics.

Current authoritative handover: `08_HANDOVER/HANDOVER_CURRENT_2026-09-05.md`.
Current Devaxonic mirror: `.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-05.md`.

Detailed executor instructions live in Git; owner-facing prompts stay microscopic. The Owner controls executor/session mechanics. Testing credentials are designated Testing data; do not rotate or redesign credential handling as part of WMS feature work.