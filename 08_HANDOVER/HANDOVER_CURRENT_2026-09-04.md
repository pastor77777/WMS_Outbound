# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-04 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 implementation/acceptance items**, **109/109 Architect requirements mapped**.

Current progress: **15/37 FINAL PASS**.

Latest accepted checkpoint:

`P1-012` — Mercato `5019a20be14549ff8cbbf25af5bc61c56888e9e1` / Scanner frozen reference `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / durable evidence `b28f59e7ff41ac6d0a3be4b841410650bc5acd8b` — **FINAL PASS / Owner Accepted** based on reviewed genuine PostgreSQL + `PLAYWRIGHT VERIFIED` evidence.

Earlier accepted checkpoints remain recorded in `STATE.md`.

## P1-012 accepted boundary

Preserve the accepted Carrier Selection behavior:

- selection starts only after Shipment `READY_FOR_DISPATCH`;
- selection inputs are delivery Region, largest current Packing-TU weight and largest `TUSetup.maxVolume` among attached TU types;
- deterministic tie-break: narrowest volume range, then narrowest weight range, then unique `Carrier.priority`;
- no match -> `CARRIER_PENDING`;
- general manual selection and override require real server-authoritative Warehouse Supervisor RBAC; client role/approval flags cannot elevate;
- EXTERNAL TU without positive `maxVolume` never gets fabricated volume; Dispatcher may choose Carrier while Shipment remains pending, and only real Supervisor approval advances to `CARRIER_SELECTED`;
- Supervisor override reason remains optional;
- no label generation/provider API/manifest/ERP/settlement was absorbed into P1-012.

Accepted P1-012 evidence records:

- Mercato final `5019a20be14549ff8cbbf25af5bc61c56888e9e1`, lineage exactly from accepted P1-011 base;
- P1-012 PostgreSQL **14/14**;
- P1-011 regression **18/18**;
- state-transition invariant **77/77**;
- P1-012 Playwright **5/5**;
- automated browser proof is `PLAYWRIGHT VERIFIED`, not Human Verified.

## Next item

Next catalog item: **P1-013 — WMS label generation and pre-manifest loading carrier correction — item 16/37**.

Fresh Task Catalog/source summary:

- objective: generate/print label from WMS-owned data and support architect-permitted carrier correction before manifest close without introducing carrier Label API;
- Architect: P1 R33–R34, P1 R35–R36 and the v1 label-error/carrier-rejection exception boundary;
- requirements: `FR-P1-17`, `FR-P1-18`, `FR-P5-11`;
- acceptance: `TC-001`, `TC-007`, `TC-066`;
- dependency: `P1-012` — satisfied;
- target: Shipment label service + Mercato print UI.

Architect behavior to preserve for P1-013:

- external-carrier label generation only from `Shipment CARRIER_SELECTED`, then `-> LABEL_GENERATED`;
- label is generated locally by WMS from existing Shipment/Packing-TU/Carrier/address data;
- no external carrier label API, no provider acceptance or electronic rejection workflow in v1;
- `OWN_TRANSPORT` skips label generation;
- loading problem before `CarrierManifest.CLOSED` permits real Supervisor carrier correction without automatic label reprint; full CarrierManifest lifecycle remains P1-015.

Current executor guide:

`06_AGENT_GUIDES/P1-013_EXECUTION.md`

P1-013 Mercato branch must start from exact accepted P1-012 SHA:

`5019a20be14549ff8cbbf25af5bc61c56888e9e1`

Do not absorb P1-014 ERP posting, P1-015 manifest lifecycle or P1-016 final settlement.

## Source and evidence routing

For Outbound business behavior:

1. `01_ARCHITECT_SOURCE` / faithful Architect translations;
2. `02_CANON`, traceability and exact task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Inbound remains **CLOSED / REFERENCE** except targeted regression when an authorized Outbound diff touches accepted shared primitives.

Evidence requirements remain: real PostgreSQL for persistence/concurrency/rollback claims; decisive rendered UI action for UI claims; automated browser evidence is `PLAYWRIGHT VERIFIED`; executor prose is not acceptance by itself.

## Operational rules

- fresh executor/supervisor starts from canonical Devaxonic-WMS checkout;
- detailed task/remediation content lives in Git and owner-facing launch prompts remain microscopic;
- owner normally returns only `done`; supervisor verifies refs/diffs/evidence independently;
- Testing credentials are designated test data; use Testing configuration and do not rotate Testing credentials unless the owner changes that rule;
- do not ask owner to paste long logs, screenshots, SHAs or secrets.

## Fresh session startup

Read/refresh in this order:

1. `Devaxonic-WMS/AGENTS.md`;
2. `Devaxonic-WMS/.ai/STATE.md` and `.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`;
3. `.ai/TESTING.md` and `.ai/OPERATIONS.md`;
4. `WMS_Outbound/STATE.md` and this handover;
5. exact P1-013 Task Catalog / Architect / requirements / acceptance scenarios;
6. `WMS_Outbound/06_AGENT_GUIDES/P1-013_EXECUTION.md`;
7. current Mercato/Scanner/WMS refs.

Do not reconstruct mutable project state from old chat history.
