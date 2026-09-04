# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-04 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 implementation/acceptance items**, **109/109 Architect requirements mapped**.

Current progress: **16/37 FINAL PASS**.

Latest accepted checkpoint:

`P1-013` — Mercato `5e6b70aa81afd28fe3217e4aad216e8a6482a769` / Scanner frozen reference `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / durable evidence `826b9c477fa86a44a93606265868730e4570ff90` — **FINAL PASS / Owner Accepted** based on independently reviewed genuine PostgreSQL concurrency/rollback evidence plus `PLAYWRIGHT VERIFIED` UI evidence.

Earlier accepted checkpoints remain recorded in `STATE.md`.

## P1-013 accepted boundary

Preserve the accepted label/correction behavior:

- external-carrier label generation is gated by `Shipment CARRIER_SELECTED`; success advances to `LABEL_GENERATED`;
- `OWN_TRANSPORT` skips label generation;
- label/tracking payload is generated from persisted WMS-owned Shipment/Packing-TU/Carrier/address data only;
- v1 has no external Carrier Label API, provider acceptance step, electronic carrier rejection state or invented `LABEL_ERROR` lifecycle;
- repeated generation is idempotent and yields one durable label plus one `ShipmentLabelGenerated` event;
- print/reprint is a local action and does not change Shipment business state or call a carrier API;
- P1-012 EXTERNAL-TU missing-`maxVolume` approval semantics remain intact;
- before the future `CarrierManifest.CLOSED` boundary, real server-authoritative Warehouse Supervisor correction may change Carrier while Shipment remains `LABEL_GENERATED`; it must not automatically regenerate/reprint the label;
- full CarrierManifest lifecycle/post-CLOSED guard remains P1-015.

Accepted P1-013 evidence records:

- final Mercato closeout `5e6b70aa81afd28fe3217e4aad216e8a6482a769`, exactly three commits above accepted P1-012 base `5019a20be14549ff8cbbf25af5bc61c56888e9e1`;
- P1-013 PostgreSQL **15/15**, including exact two-session backend PID lock proof and fresh one-label/one-event replay proof;
- P1-012 regression **14/14**;
- P1-011 regression **18/18**;
- state-transition invariant **77/77**;
- P1-013 Playwright **4/4** against served product revision `7f8185c3eccaf1b04fd46027616b0286d4c87fd1`; later P1-013 commits were test-only;
- clean Mercato/WMS worktrees and frozen Scanner `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## Next item

Next catalog item: **P1-014 — ERP Shipment POST, error state and safe retry — item 17/37**.

Fresh Task Catalog/source summary:

- objective: implement `Shipment LABEL_GENERATED/OWN_TRANSPORT → POSTING_PENDING`, ERP acceptance → `POSTED`, explicit ERP rejection → `POSTING_ERROR`, durable attempt evidence and idempotent safe manual retry;
- Architect: P1 KROK 11A / R35–R38, with downstream exactly-once boundary from `CON-05`;
- requirements: `FR-P1-18`, `FR-P1-19`, `INT-04`, `INT-05`, `CON-05`;
- acceptance: `TC-001`, `TC-007`, `TC-008`;
- dependencies: `P1-011`, `P1-013`, `FND-002` — satisfied;
- target: Shipment posting service, `wms_orchestration` outbox/retry, ERP adapter and Mercato Supervisor error/retry UI.

Architect behavior to preserve for P1-014:

- every Shipment must pass the ERP posting gate before it can enter a CarrierManifest;
- valid start states are `LABEL_GENERATED` and `OWN_TRANSPORT`;
- entering posting creates durable/observable `POSTING_PENDING` intent;
- only an accepted ERP response advances to `POSTED`;
- an explicit structured ERP rejection advances to `POSTING_ERROR` and surfaces a safe operational reason;
- retry from `POSTING_ERROR` is a separate real Warehouse Supervisor decision and returns to `POSTING_PENDING`;
- the explicit `POSTING_ERROR → CANCELLED` Supervisor give-up path is permitted, but no general post-`POSTING_PENDING` WMS cancellation path may be invented;
- packed TU physical contents and `OutboundOrderLine PACKED` are unchanged by posting failure/retry;
- timeout/no response is a technical incident outside the defined P1 business path and must not be silently reclassified as a business rejection/`POSTING_ERROR`;
- duplicate/retried request/response handling must preserve exactly-once WMS business effects;
- P1-015 owns `POSTED → IN_MANIFEST` and all CarrierManifest lifecycle behavior.

Current executor guide:

`06_AGENT_GUIDES/P1-014_EXECUTION.md`

P1-014 Mercato branch must start from exact accepted P1-013 SHA:

`5e6b70aa81afd28fe3217e4aad216e8a6482a769`

Scanner remains frozen unless a later explicitly authorized task needs it.

Do not absorb P1-015 manifest lifecycle, P1-016 settlement, Crossdock GR gate work, Return Receipt, Scanner work or any invented external ERP behavior.

## Integration/evidence honesty

Existing `wms_orchestration` event-fact/retry infrastructure is a reusable technical foundation, not automatic proof of a real ERP integration.

Executor must inspect current approved ERP/integration seams. If no real approved Testing ERP endpoint/credential exists, use an explicit adapter boundary with deterministic Testing/test behavior as needed and label that evidence honestly as contract/stub verification. Never invent an ERP endpoint, credential or claim real external ERP verification that did not occur.

## Source and evidence routing

For Outbound business behavior:

1. `01_ARCHITECT_SOURCE` / faithful Architect translations;
2. `02_CANON`, traceability and exact task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Inbound remains **CLOSED / REFERENCE** except targeted regression when an authorized Outbound diff touches accepted shared primitives.

Evidence requirements remain: real PostgreSQL for persistence/concurrency/rollback/exactly-once claims; decisive rendered UI actions for UI claims; automated browser evidence is `PLAYWRIGHT VERIFIED`; executor prose is not acceptance by itself.

## Operational rules

- every P1-014 executor run uses a **fresh Antigravity session**;
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
5. exact P1-014 Task Catalog / Architect KROK 11A / requirements / transition catalog / acceptance scenarios;
6. `WMS_Outbound/06_AGENT_GUIDES/P1-014_EXECUTION.md`;
7. current Mercato/Scanner/WMS refs and current reusable `wms_orchestration` implementation.

Do not reconstruct mutable project state from old chat history.