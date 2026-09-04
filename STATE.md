# WMS Outbound — STATE

**As of:** 2026-09-04  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **15/37 items FINAL PASS**

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
13. `P1-010` — FINAL PASS / Owner Accepted — Mercato `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643` / Scanner frozen reference `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b5bb6429717402e0fb6969f7437ddaf673a8a174`
14. `P1-011` — FINAL PASS / Owner Accepted — Mercato `20887f2d74928cf69f447fdd6af20a612f38387c` / Scanner frozen reference `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `90cc30fc2db15c40d80ef69cb03ffb1e107b51dc`.
15. `P1-012` — FINAL PASS / Owner Accepted — Mercato `5019a20be14549ff8cbbf25af5bc61c56888e9e1` / Scanner frozen reference `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b28f59e7ff41ac6d0a3be4b841410650bc5acd8b`.

## P1-012 accepted boundary

Preserve these accepted behaviors:

- Carrier Selection starts only from Shipment `READY_FOR_DISPATCH` and uses delivery Region, largest current Packing-TU weight and largest `TUSetup.maxVolume` of the attached TU types.
- Deterministic winner order is narrowest matching volume range, then narrowest matching weight range, then unique `Carrier.priority`; unresolved ambiguity fails closed.
- No match yields `CARRIER_PENDING`; general manual selection and any override require real server-authoritative Warehouse Supervisor authority.
- EXTERNAL TU without positive `maxVolume` never receives fabricated volume: Dispatcher may record a carrier choice but Shipment remains `CARRIER_PENDING` until real Supervisor approval.
- Client-supplied role/approval fields cannot elevate authority; audit identity/role is derived server-side from authenticated RBAC context.
- Supervisor override reason is optional.
- P1-012 does not implement label generation, external carrier API, manifest, ERP posting or settlement.

Accepted proof in `05_EVIDENCE/P1-012_EVIDENCE.md`:

- final Mercato head `5019a20be14549ff8cbbf25af5bc61c56888e9e1`, exactly two commits after accepted P1-011 base `20887f2d74928cf69f447fdd6af20a612f38387c`;
- P1-012 genuine PostgreSQL suite **14/14**;
- P1-011 PostgreSQL regression **18/18**;
- state transition invariant suite **77/77**;
- P1-012 Mercato Playwright **5/5**;
- automated UI evidence remains `PLAYWRIGHT VERIFIED`, not Human Verified.

## Current position

Completed: **15/37**.

Next implementation item:

**P1-013 — WMS label generation and pre-manifest loading carrier correction — item 16/37.**

Fresh Task Catalog grounding:

- objective: generate/print label from WMS-owned data and support architect-permitted carrier correction before manifest close without introducing carrier Label API;
- Architect source/reference: P1 R33–R34, P1 R35–R36, P1 exception — label error / carrier rejection boundary;
- requirements: `FR-P1-17`, `FR-P1-18`, `FR-P5-11`;
- dependencies: `P1-012` — satisfied;
- target components: Shipment label service, Mercato print UI;
- acceptance: `TC-001`, `TC-007`, `TC-066`.

Current execution guide:

`06_AGENT_GUIDES/P1-013_EXECUTION.md`

Do not absorb ERP posting (`P1-014`), CarrierManifest lifecycle (`P1-015`) or final settlement (`P1-016`). Do not introduce an external carrier label API/provider acceptance/rejection lifecycle.

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
