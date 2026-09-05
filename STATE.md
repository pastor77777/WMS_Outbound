# WMS Outbound — STATE

**As of:** 2026-09-05  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **18/37 items FINAL PASS**

## Architect baseline

Active process set:

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19

Requirements: 109 IDs = 98 FR + 6 INT + 5 CON.

`PickWave` is out of scope v1. No separate Process 5 exists; cross-cutting exceptions live in P1/P2 and are exposed as `FR-P5-*`.

## Implementation plan

The complete implementation plan remains in `07_IMPLEMENTATION_PLAN/`: 37 tasks with 109/109 requirements mapped.

The plan is delivery decomposition only. Architect Source/Canon remains business authority.

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

## P1-015 accepted boundary

Preserve these accepted behaviors:

- only `Shipment POSTED` may enter an `OPEN` `CarrierManifest`; successful assignment advances `POSTED → IN_MANIFEST` and one Shipment belongs to exactly one manifest;
- `CarrierManifest OPEN → CLOSED` is irreversible and freezes composition; no post-close add/remove/reopen, Shipment cancellation or Carrier correction;
- close serializes against member Shipment mutations with real PostgreSQL row locks;
- `CLOSED → HANDED_OVER` is physical handover and advances included Shipments `IN_MANIFEST → HANDED_TO_CARRIER`, TUs `IN_SHIPMENT → DISPATCHED`, and contributing OutboundOrders `READY_FOR_DISPATCH → DISPATCHED`;
- `HANDED_OVER → CONFIRMED` is a distinct final warehouse-confirmation boundary with exactly one `CarrierManifestConfirmed` fact and deterministic membership snapshot under duplicate/parallel calls;
- P1-015 does not perform final `Inventory`/`Allocation`/`OutboundOrderLine`/`OutboundOrder`/`CustomerOrder` settlement; that boundary is P1-016;
- no external carrier API and no Scanner changes were introduced.

Accepted proof in `05_EVIDENCE/P1-015_EVIDENCE.md`:

- final Mercato head `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`, exactly two commits above accepted P1-014 base `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`;
- P1-015 PostgreSQL **21/21**, including exact-session lock proof for assignment, add-vs-close, duplicate confirm and carrier-correction-vs-close;
- post-remediation regression aggregate **171/171**;
- fresh P1-015 Playwright **6/6** against canonical Testing served from exact final runtime `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`;
- durable final evidence `f201bd0beb2411b4f87f28ca6562a4fc11e6a249`; Scanner remained frozen.

## Current position

Completed and accepted: **18/37**.

Active implementation item — **execution in flight, not yet verified or accepted**:

**P1-016 — Final line/order/inventory settlement and cancellation boundaries — item 19/37.**

Owner reported on 2026-09-05 that an owner-selected Codex executor is executing `06_AGENT_GUIDES/P1-016_EXECUTION.md`. Do not infer completion from this fact. Steering/acceptance advances only after the executor finishes, supervisor independently verifies refs/diff/evidence, and the Owner explicitly accepts the result.

Fresh Task Catalog grounding:

- objective: settle `Inventory`/`Allocation`/`OutboundOrderLine` quantities per confirmed contributing manifest, terminalize lines and orders only after all contributing Shipment/manifest coverage is confirmed, continuously aggregate CustomerOrderLine/CustomerOrder, and enforce cancellation boundaries;
- Architect source: P1 R37–R38, R41–R42, R49–R50, F1, R70–R72, general cancellation; P3/P4 entry routing;
- requirements: `FR-P1-19`, `FR-P1-21`, `FR-P1-25`, `FR-P1-27`, `FR-P1-44`, `FR-P1-45`, `FR-P1-46`, `FR-P5-09`, `INT-06`, `CON-05`;
- dependencies: `P1-015`, `P1-004`, `P1-001` — satisfied;
- target: Inventory ledger, Allocation, OutboundOrder/Line, CustomerOrder/Line, cancellation correlation/orchestration;
- acceptance: `TC-001`, `TC-003`, `TC-008`, `TC-009`, `TC-065`, `TC-128`, `TC-129`, `TC-130`, `TC-131`, `TC-132`, `TC-133`.

Authoritative P1-016 boundaries:

- each `CONFIRMED` manifest settles only the quantities physically represented by its Outbound TUs: `Inventory PICKED → SHIPPED`, `Allocation.reservedQty` decreases by exactly that quantity, and line `shippedQty` advances exactly once;
- partial multi-manifest issue leaves `OutboundOrderLine PACKED` and `Allocation CONFIRMED`; only full contributing-TU confirmation allows `OutboundOrderLine → SHIPPED` and `Allocation → CONSUMED` with `reservedQty = 0`;
- `OutboundOrder DISPATCHED → COMPLETED` only when every Shipment containing any TU of that order is on a `CONFIRMED` manifest; confirmation order is irrelevant;
- CustomerOrderLine and CustomerOrder aggregation follows Architect F1; no premature `SHIPPED`/`CLOSED`;
- duplicate/parallel settlement must have one business effect; cross-manifest concurrent settlement of one line/allocation must not lose or double arithmetic;
- cancellation remains impossible from Shipment `POSTING_PENDING` onward and after `CarrierManifest.CLOSED`; Return Receipt remains outside scope;
- incoming cancellation/correction correlation chooses P3 vs P4 from formal `pickedQty`; P1-016 establishes the boundary/routing but must not implement future P3/P4 physical recovery workflows;
- Scanner remains frozen; no P2 Crossdock implementation, Return Receipt, external carrier APIs or Prod/Demo changes.

Current executor guide:

`06_AGENT_GUIDES/P1-016_EXECUTION.md`

P1-016 Mercato branch must start from exact accepted P1-015 SHA:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

## Authority and architecture-context rule

For Outbound business behavior, authority order is:

1. `01_ARCHITECT_SOURCE` / faithful Architect translations and `02_CANON`;
2. traceability and exact current task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Inbound remains **CLOSED / REFERENCE** except targeted regression when an authorized Outbound diff touches shared primitives.

## Current-state documents

`04_CURRENT_STATE/*` remains the pre-implementation audit baseline captured before ETAP 2 writes.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-05.md`

Current operational mirror in Devaxonic-WMS:

`.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-05.md`

## Operating workflow

The persisted workflow remains:

`06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

Detailed executor instructions live in Git; owner-facing launch prompts stay microscopic and stand alone without appended questions/suggestions. The Owner controls executor launch/resume/session organization. Supervisor independently verifies refs/diffs/evidence. Testing uses designated Testing credentials/configuration; do not rotate Testing credentials unless the owner explicitly changes that rule.