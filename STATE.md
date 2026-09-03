# WMS Outbound — STATE

**As of:** 2026-09-03  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **12/37 items FINAL PASS**

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
8. `P1-005` — FINAL PASS / Human Verified — `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975` (Mercato) / `8199b330cb739a45e2c615a3f2aa3803336be724` (Scanner)
9. `P1-008` — FINAL PASS / Human Verified — `9512137702a5d5f5b41910c2de97cf03321a1ccd` (Mercato) / `b5cfb59987c76f39e0ab48af67a52e2e914d9613` (Scanner)
10. `P1-006` — FINAL PASS / Human Verified — `353a5001cb8f1941971f960e509a8af643e41e5a` (Mercato) / `7596b7802e7ed55a59dd6dc1f21912ea6331e796` (Scanner) / evidence `43cc7d0e7dd20a48fc00b40150b30275d0c2aa12`
11. `P1-007` — FINAL PASS / Owner Accepted based on PLAYWRIGHT VERIFIED + real PostgreSQL evidence — `134db31381b4db726cd550abe6ecd4079ac21d8c` (Mercato) / `b23325aae1c4f83b79d01b3650dbead3486a1041` (Scanner) / evidence `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8`
12. `P1-009` — FINAL PASS / Owner Accepted based on PLAYWRIGHT VERIFIED + real PostgreSQL evidence — `5d780dabeb605bc657bb521bd2b2fdcc2e516f77` (Mercato) / `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` (Scanner) / evidence `6c78a97ece567d90d3cb7d0580bb38669c9f9722`

## P1-009 accepted boundary

- `directPackDeclared` is bound at the architect-defined first Picking TU scan and is immutable once picking begins; exact same-intent replay is safe and conflicting mutation fails closed.
- Same-TU multi-zone continuation preserves the declaration without re-prompting; P1 R67 replacement TU after `PICK_FULL` inherits it without re-prompting.
- Qualifying direct-pack TU automatically executes `READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED`, retains TU identity as `PackUnit`, and moves eligible line quantity to `PACKED` with atomic `packedQty` persistence without a Packer action.
- Non-issuable / below-threshold paths remain at `READY_TO_PACK`; incomplete / `SHORT_PICKED` paths do not falsely mark the line PACKED.
- Final proof is labelled `PLAYWRIGHT VERIFIED` and includes genuine remote PostgreSQL identity, real lock contention with distinct PostgreSQL PIDs, real rollback, P1-009 backend 15/15, Scanner direct-pack 4/4, Scanner P1-006/P1-007/P1-008 regressions 3/3, backend P1-006/P1-007/P1-008 regressions 54/54, and full `wms_outbound` 260/260.
- Final implementation diff from accepted bases is clean: the historical P1-007 migration is unchanged; Mercato/Scanner heads remained frozen throughout evidence-only closeout.

## Current position

Completed: **12/37**.

Next implementation item:

**P1-010 — Packing, repack, consolidation and discrepancy handling — item 13/37.**

Task Catalog ownership:

- objective: process `READY_TO_PACK` through Packer keep/repack/consolidate paths and handle packing discrepancies;
- Architect: P1 R19–R26, with inherited compatibility constraints from current KROK 7–8 / R48 / R60 / R63 / R66 where they govern the same packing actions;
- requirements: `FR-P1-10`, `FR-P1-11`, `FR-P1-12`, `FR-P1-13`, `FR-P5-07`;
- dependencies: `P1-006`, `P1-008` — satisfied; preserve accepted P1-009 direct-pack behavior as an upstream alternative path;
- target components: Packing backend, Mercato/Packer UI, TU contents, QC handoff;
- acceptance: `TC-001`, `TC-005`, `TC-006`, `TC-063`.

P1-010 hard boundary:

- Packer evaluates standard `READY_TO_PACK` TU: keep / repack / consolidate; deviation from the WMS suggestion continues to obey accepted R18 Supervisor approval semantics.
- Keep preserves TU identity; repack/consolidate makes the source PickContainer `REPACKED` and prepares one or more Packing TUs.
- Repack v1 has two warehouse-configurable modes: `repack all` and `repack by SKU`; do not invent SKU-guidance order for by-SKU mode.
- Packing consolidation has no category/temperature compatibility rules in v1. Enforce the architect packing mass limit; content volume is not a repack blocking limit. Preserve R60 cross-order compatibility and one-Shipment-per-Packing-TU constraint at the packing seam.
- Repack target must be externally issuable per accepted P1-008/R66; non-issuable target cannot be sealed.
- By-SKU shortage requires recheck + explicit confirmation before invoking the accepted SHORT_PICKED recovery mechanism, with no source-location block.
- DAMAGED quantity goes to QC, is recorded as DAMAGED, and uses the same shortage mechanism; unexpected/overage SKU goes to QC but must not create a shortage.
- Source TU completion requires every expected SKU quantity to be accounted for as packed, QC, or confirmed missing; otherwise continue or explicitly confirm missing after recheck.
- R26 is only the packing-to-Shipment grouping seam. Do **not** absorb P1-011 Shipment lifecycle/readiness/partial-shipment gates, P1-012 Carrier selection, labels, ERP posting, manifest or dispatch work.

## Authority and architecture-context rule

For Outbound business behavior, authority order is:

1. `01_ARCHITECT_SOURCE` / faithful Architect translations and `02_CANON`;
2. traceability and exact current task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Installed `wms-outbound` skill routes Outbound work to this authority chain. Installed `architecture-context` is reference-only for accepted Inbound/shared Inventory/TU/warehouse/lock/orchestration compatibility. It must not redefine Outbound R-rules or expand a ticket into later scope.

Inbound remains **CLOSED / REFERENCE** except targeted regression when an authorized Outbound diff touches shared primitives.

## Current-state documents

`04_CURRENT_STATE/*` remains the pre-implementation audit baseline captured before ETAP 2 writes. Do not rewrite that baseline to masquerade as current runtime truth.

For current implementation state use this file, current Git refs/runtime evidence, `08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md`, and the current Devaxonic-WMS `.ai` handover/state.

## Operating workflow

Canonical prompt/executor workflow is persisted in:

`06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

Key rhythm: **WRITE PROMPT/GUIDE TO GIT -> short owner-facing launch prompt -> SAME Antigravity session -> `done` -> supervisor independently FETCHES/VERIFIES Git refs/evidence -> next shot or STOP.**

The owner should never need to paste Antigravity logs into chat.

## Blockers

No current product/architecture blocker is recorded for starting `P1-010`.

## Handover

Current authoritative Outbound handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md`

Current operational mirror in Devaxonic-WMS:

`.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-03.md`

Fresh supervisor sessions must consume current handover/state and `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md` instead of reconstructing implementation state or operating rules from chat history.