# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-04 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 implementation/acceptance items**, **109/109 Architect requirements mapped**.

Current progress: **17/37 FINAL PASS**.

Latest accepted checkpoint:

`P1-014` — Mercato `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6` / Scanner frozen reference `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / durable evidence `b97f1640621b5b01571efec313b7fa0325c1aedf` — **FINAL PASS / Owner Accepted** after independent review of the remediation: genuine two-service PostgreSQL concurrency, in-flight duplicate idempotency, literal final schema evidence and `PLAYWRIGHT VERIFIED` 5/5 rendered UI evidence.

Earlier accepted checkpoints remain recorded in `STATE.md`.

## P1-014 accepted boundary

Preserve the accepted ERP-posting behavior:

- initial posting starts only from `Shipment LABEL_GENERATED` or `OWN_TRANSPORT` and durably enters `POSTING_PENDING`;
- the implementation uses a typed deterministic Testing ERP adapter/contract seam and does **not** claim real external ERP connectivity;
- explicit accepted response advances to `POSTED`; explicit structured rejection advances to `POSTING_ERROR`;
- timeout/no-response is a technical incident and leaves Shipment in `POSTING_PENDING`, not business `POSTING_ERROR`;
- a real server-authoritative Warehouse Supervisor is required for retry `POSTING_ERROR → POSTING_PENDING` and give-up `POSTING_ERROR → CANCELLED`;
- non-Supervisor retry/give-up fails closed; direct cancellation from `POSTING_PENDING` or `POSTED` is rejected;
- duplicate initial request while Phase 2 is in flight returns an explicit idempotent in-flight replay and does not create another posting intent, attempt, transition or adapter invocation;
- duplicate call after `POSTED` remains a safe replay;
- P1-014 did not implement CarrierManifest membership/lifecycle, final settlement, Scanner work, Return Receipt or external carrier API behavior.

Accepted evidence records:

- final Mercato head `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`, exactly two commits above accepted P1-013 base `5e6b70aa81afd28fe3217e4aad216e8a6482a769`;
- P1-014 PostgreSQL **18/18**, including exact row-lock contention between two real `postShipment()` service calls and fresh-read exactly-one posting/attempt/`ShipmentPostingRequested`/`ShipmentPosted` plus exactly one adapter call;
- P1-013 regression **15/15**, P1-012 **14/14**, P1-011 **18/18**, FND-002 state transitions **77/77**, FND-002 transaction simulation **8/8**;
- P1-014 Playwright **5/5** against served runtime `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`;
- rendered Journey E proves real non-Supervisor fail-closed, absence of manifest action in P1-014 and timeout staying outside business `POSTING_ERROR`;
- relevant worktrees recorded clean; Scanner remained frozen.

## Next item

Next catalog item: **P1-015 — CarrierManifest lifecycle and dispatch boundaries — item 18/37**.

Fresh Task Catalog/source summary:

- objective: implement one-manifest assignment and `CarrierManifest OPEN → CLOSED → HANDED_OVER → CONFIRMED`, with irreversible `CLOSED` composition boundary and final warehouse-confirmation boundary;
- Architect: P1 KROK 12–13 / R39–R40 plus the manifest-side state/event boundaries feeding later settlement;
- requirements traced by the task: `FR-P1-20`, `FR-P1-44`, `FR-P1-46`, `CON-04`, `CON-05`;
- acceptance: `TC-001`, `TC-008`, `TC-128`, `TC-129`, `TC-132`, `TC-133`;
- dependency: `P1-014` — satisfied;
- target: CarrierManifest persistence, Shipment membership and Mercato Dispatcher/Supervisor dispatch UI.

Architect behavior to preserve for P1-015:

- Dispatcher opens a manifest in `OPEN`;
- only `Shipment POSTED` may be added; successful assignment advances Shipment `POSTED → IN_MANIFEST`;
- a Shipment may belong to exactly one manifest;
- `OPEN → CLOSED` is irreversible and freezes composition; after close, no reopen/add/remove, Shipment cancellation or Carrier correction is allowed;
- `CLOSED → HANDED_OVER` is physical carrier/self-transport handover and advances included Shipment `IN_MANIFEST → HANDED_TO_CARRIER`;
- `HANDED_OVER → CONFIRMED` is the final warehouse confirmation and must be idempotent/exactly-once;
- final settlement semantics in Architect (`Inventory PICKED→SHIPPED`, `Allocation.reservedQty`, `OutboundOrderLine SHIPPED`, `OutboundOrder COMPLETED`, CustomerOrder aggregation) are owned by the separate next task `P1-016`; P1-015 must expose a durable deterministic confirmation event/membership snapshot sufficient for that downstream settlement, but must **not** implement P1-016 prematurely;
- duplicate/parallel close/handover/confirm operations must not duplicate business facts or regress final states;
- do not equate existing legacy `carrier_shipments` with the target `CarrierManifest` model;
- no external carrier API is required;
- Scanner remains frozen unless inspection of current accepted product proves the manifest handover action is already Scanner-owned; do not invent Scanner ownership.

Current executor guide:

`06_AGENT_GUIDES/P1-015_EXECUTION.md`

P1-015 Mercato branch must start from exact accepted P1-014 SHA:

`bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`

Do not absorb P1-016 final settlement/cancellation logic, Crossdock GR gating, Return Receipt, external carrier APIs, unrelated Scanner work or Prod/Demo changes.

## Source and evidence routing

For Outbound business behavior:

1. `01_ARCHITECT_SOURCE` / faithful Architect translations;
2. `02_CANON`, traceability and exact task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Inbound remains **CLOSED / REFERENCE** except targeted regression when an authorized Outbound diff touches accepted shared primitives.

Evidence requirements remain: real PostgreSQL for persistence/concurrency/rollback/exactly-once claims; decisive rendered UI actions for UI claims; automated browser evidence is `PLAYWRIGHT VERIFIED`; executor prose is not acceptance by itself.

## Operational rules

- every P1-015 executor run uses a **fresh Antigravity session**;
- canonical start: `/home/ubuntu/git/Devaxonic-WMS`, then `/home/ubuntu/.local/bin/agy-pl`; never bare `agy`;
- detailed task/remediation content lives in Git and owner-facing launch prompts remain microscopic;
- owner normally returns only `done`; supervisor verifies refs/diffs/evidence independently;
- Testing credentials are designated test data; use Testing configuration and do not rotate Testing credentials unless the owner changes that rule;
- do not ask owner to paste long logs, screenshots, SHAs or secrets;
- no Prod/Demo changes.

## Fresh session startup

Read/refresh in this order:

1. `Devaxonic-WMS/AGENTS.md`;
2. `Devaxonic-WMS/.ai/STATE.md` and `.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`;
3. `.ai/TESTING.md` and `.ai/OPERATIONS.md`;
4. `WMS_Outbound/STATE.md` and this handover;
5. exact P1-015 Task Catalog / Architect KROK 12–13 / state model / requirements / acceptance scenarios;
6. `WMS_Outbound/06_AGENT_GUIDES/P1-015_EXECUTION.md`;
7. current Mercato CarrierManifest/Shipment/TU/Allocation/Inventory primitives and exact Git refs.

Do not reconstruct mutable project state from old chat history.