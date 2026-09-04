# WMS Outbound — STATE

**As of:** 2026-09-04  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **17/37 items FINAL PASS**

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

## P1-014 accepted boundary

Preserve these accepted behaviors:

- initial ERP posting is allowed only from Shipment `LABEL_GENERATED` or `OWN_TRANSPORT` and durably enters `POSTING_PENDING`;
- the approved implementation uses an explicit deterministic Testing ERP adapter/contract seam; it does **not** claim a real external ERP endpoint;
- explicit adapter acceptance advances `POSTING_PENDING → POSTED` and explicit structured business rejection advances `POSTING_PENDING → POSTING_ERROR`;
- timeout/no-response is a technical incident and leaves the Shipment in `POSTING_PENDING`; it is not reclassified as business `POSTING_ERROR`;
- real server-authoritative Warehouse Supervisor authority is required for `POSTING_ERROR → POSTING_PENDING` retry and `POSTING_ERROR → CANCELLED` give-up; non-Supervisor actions fail closed;
- direct cancellation from `POSTING_PENDING` or `POSTED` is forbidden;
- repeated/concurrent initial posting is idempotent: a duplicate arriving while the original call is in Phase 2 returns an explicit in-flight replay and does not create a second posting row, attempt, transition or ERP adapter invocation;
- post-settlement duplicate calls return a safe replay without state regression;
- P1-014 does not implement `POSTED → IN_MANIFEST`, CarrierManifest lifecycle, final inventory/order settlement, Scanner changes, Return Receipt or Prod/Demo behavior.

Accepted proof in `05_EVIDENCE/P1-014_EVIDENCE.md`:

- final Mercato head `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`, exactly two commits after accepted P1-013 base `5e6b70aa81afd28fe3217e4aad216e8a6482a769`;
- P1-014 PostgreSQL **18/18**, including two real `postShipment()` service calls, exact PostgreSQL lock contention, one posting row, one attempt, one `ShipmentPostingRequested`, one `ShipmentPosted` and exactly one Testing adapter call;
- P1-013 regression **15/15**, P1-012 **14/14**, P1-011 **18/18**, FND-002 state transitions **77/77**, FND-002 transaction simulation **8/8**;
- P1-014 Playwright **5/5** against served runtime `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`, including non-Supervisor fail-closed, no manifest action and timeout/non-business boundary;
- final worktrees recorded clean and Scanner remained frozen.

## Current position

Completed: **17/37**.

Next implementation item:

**P1-015 — CarrierManifest lifecycle and dispatch boundaries — item 18/37.**

Fresh Task Catalog grounding:

- objective: implement one-manifest assignment and `CarrierManifest OPEN → CLOSED → HANDED_OVER → CONFIRMED`, with irreversible `CLOSED` composition boundary and final warehouse-confirmation event;
- Architect source: P1 KROK 12–13, R39–R40 and manifest-related state/event rules;
- requirements traced by the task: `FR-P1-20`, `FR-P1-44`, `FR-P1-46`, `CON-04`, `CON-05`;
- dependency: `P1-014` — satisfied;
- target components: `CarrierManifest`, Shipment membership and Mercato dispatch/Supervisor UI;
- acceptance mapping: `TC-001`, `TC-008`, `TC-128`, `TC-129`, `TC-132`, `TC-133`;
- task DoD: `CLOSED` is irreversible and distinct from physical handover; duplicate confirmation has one business effect.

Authoritative P1 behavior for this boundary:

- only `Shipment POSTED` may be added to an `OPEN` manifest; add advances Shipment `POSTED → IN_MANIFEST`;
- one Shipment may belong to exactly one manifest;
- `OPEN → CLOSED` is irreversible and freezes composition; after `CLOSED` no add/remove/reopen, Shipment cancellation or Carrier correction may be permitted;
- `CLOSED → HANDED_OVER` represents physical carrier/self-transport handover and advances included Shipments `IN_MANIFEST → HANDED_TO_CARRIER`;
- `HANDED_OVER → CONFIRMED` is the final warehouse confirmation and must be idempotent/exactly-once;
- P1-015 must create a durable, deterministic confirmation boundary/event suitable for downstream settlement without performing the final `Inventory`/`Allocation`/`OutboundOrderLine`/`CustomerOrder` settlement owned by `P1-016`;
- do not use legacy `carrier_shipments` as the target CarrierManifest truth merely because it exists;
- no external carrier API is required;
- Scanner remains frozen unless current product inspection proves the authorized manifest handover action is already scanner-owned; do not move it to Scanner by invention.

Current executor guide:

`06_AGENT_GUIDES/P1-015_EXECUTION.md`

P1-015 Mercato branch must start from exact accepted P1-014 SHA:

`bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`

Do not absorb `P1-016` final settlement/cancellation work, Crossdock GR gating, Return Receipt, external carrier APIs, unrelated Scanner work or Prod/Demo changes.

## Authority and architecture-context rule

For Outbound business behavior, authority order is:

1. `01_ARCHITECT_SOURCE` / faithful Architect translations and `02_CANON`;
2. traceability and exact current task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Inbound remains **CLOSED / REFERENCE** except targeted regression when an authorized Outbound diff touches shared primitives.

## Current-state documents

`04_CURRENT_STATE/*` remains the pre-implementation audit baseline captured before ETAP 2 writes. Do not rewrite that baseline to masquerade as current runtime truth.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-04.md`

Current operational mirror in Devaxonic-WMS:

`.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`

## Operating workflow

The persisted workflow remains:

`06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

Detailed executor instructions live in Git; owner-facing launch prompts stay microscopic; supervisor independently verifies refs/diffs/evidence. Testing uses designated Testing credentials/configuration; do not rotate Testing credentials unless the owner explicitly changes that rule.