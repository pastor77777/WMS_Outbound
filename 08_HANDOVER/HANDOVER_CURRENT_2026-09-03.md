# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-03 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 implementation/acceptance items**, **109/109 Architect requirements mapped**.

Current progress: **12/37 FINAL PASS**.

Accepted checkpoints:

1. `FND-001` — `a3c8d67f3ec65967fd3405b3da8c9901fb46e192`
2. `FND-002` — `d835dc5ec157cb0485cb09cfbc0dfdeedd40c281`
3. `FND-003` — `8fa0214d36308b00aacf32b5df369ef486975ae2`
4. `P1-001` — `435b51007e5411ebdbb1b3b4d30c984fd770d4c6`
5. `P1-002` — `7510ef3f05b6c64c3f9de925a5a85f644913cdfe` — Human Verified
6. `P1-003` — `bae31c2c2ad9b1426b868d6df7a05076669ace0d` — Human Verified
7. `P1-004` — `71b74b5384b3fbfc55d8ed298f1a3715dc477c3c`
8. `P1-005` — Mercato `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975` / Scanner `8199b330cb739a45e2c615a3f2aa3803336be724` — Human Verified
9. `P1-008` — Mercato `9512137702a5d5f5b41910c2de97cf03321a1ccd` / Scanner `b5cfb59987c76f39e0ab48af67a52e2e914d9613` — Human Verified
10. `P1-006` — Mercato `353a5001cb8f1941971f960e509a8af643e41e5a` / Scanner `7596b7802e7ed55a59dd6dc1f21912ea6331e796` / evidence `43cc7d0e7dd20a48fc00b40150b30275d0c2aa12` — Human Verified
11. `P1-007` — Mercato `134db31381b4db726cd550abe6ecd4079ac21d8c` / Scanner `b23325aae1c4f83b79d01b3650dbead3486a1041` / evidence `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8` — FINAL PASS / Owner Accepted based on reviewed `PLAYWRIGHT VERIFIED` + real PostgreSQL evidence.
12. `P1-009` — Mercato `5d780dabeb605bc657bb521bd2b2fdcc2e516f77` / Scanner `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / durable evidence `6c78a97ece567d90d3cb7d0580bb38669c9f9722` — FINAL PASS / Owner Accepted based on reviewed `PLAYWRIGHT VERIFIED` + real PostgreSQL evidence.

## P1-009 final accepted proof

Final evidence is `05_EVIDENCE/P1-009_EVIDENCE.md` at WMS evidence commit `6c78a97ece567d90d3cb7d0580bb38669c9f9722`.

Accepted evidence surface:

- genuine remote PostgreSQL identity proof;
- P1-009 PostgreSQL suite **15/15** with exact checked-in test names;
- real PostgreSQL concurrency with distinct PIDs / DB-side blocking evidence and real rollback;
- real Scanner P1-009 direct-pack Playwright **4/4** journeys: happy path, same-TU multi-zone, R67 replacement inheritance, negative issueability;
- Scanner P1-006/P1-007/P1-008 regression specs **3/3**;
- backend P1-006/P1-007/P1-008 regression matrix **54/54**;
- full `wms_outbound` umbrella **260/260**;
- final Mercato/Scanner implementation heads remained frozen through evidence-only closeout;
- final implementation diff is clean and does not modify the accepted historical P1-007 migration.

## P1-009 business boundary to preserve

- First-scan `directPackDeclared` is immutable after picking starts; replay of the same intent is safe and conflicting intent fails closed.
- Same-TU multi-zone continuation does not re-prompt; R67 replacement TU inherits the declaration.
- A qualifying Direct Pack TU auto-transitions `READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED`, keeps TU identity as `PackUnit`, and atomically moves eligible line quantity to `PACKED`/`packedQty` without Packer action.
- Non-issuable/below-threshold and incomplete/SHORT paths do not falsely auto-pack.

## Next item

Next planned implementation item:

**P1-010 — Packing, repack, consolidation and discrepancy handling — item 13/37.**

Task Catalog:

- objective: implement standard `READY_TO_PACK` processing, qualification, repack-all/repack-by-SKU, consolidation and packing discrepancy outcomes;
- Architect core: P1 R19–R26;
- requirements: `FR-P1-10`, `FR-P1-11`, `FR-P1-12`, `FR-P1-13`, `FR-P5-07`;
- dependencies: `P1-006`, `P1-008` — satisfied; preserve accepted P1-009 upstream Direct Pack behavior;
- target components: Packing backend, Mercato/Packer UI, TU contents, QC handoff;
- acceptance: `TC-001`, `TC-005`, `TC-006`, `TC-063`;
- DoD: Packer completes allowed keep/repack/consolidate modes through normal UI; packing shortage requires recheck + explicit confirmation; damage/unexpected SKU behavior matches current Architect source.

Grounded P1-010 constraints that must not be lost:

- Standard Packer path begins from `READY_TO_PACK`. Keep on an issuable/qualified TU preserves `TU_NUMBER`; repack/consolidate terminalizes the source PickContainer as `REPACKED` and prepares one or more Packing TUs.
- `repack all`: no per-SKU quantity control; `repack by SKU`: Packer chooses SKU/order, scans code and quantity, WMS does not direct the next SKU.
- Quantity below `pickedQty` does not become a shortage automatically. Packer may defer; reporting shortage requires recheck and explicit confirmation, then uses accepted SHORT_PICKED recovery without source-location blocking.
- DAMAGED quantity cannot be packed; it goes to QC, records `DAMAGED`, and invokes the same shortage recovery. Unexpected/overage SKU goes to QC but must not invoke shortage.
- Completion of a source TU requires every system-expected SKU quantity accounted for as Packed, QC or confirmed missing; otherwise continue or explicitly confirm missing after recheck.
- Repack target must be externally issuable (`P1 R66`). Packing v1 has no category/temperature compatibility restriction; enforce the Architect packing mass limit, not an invented content-volume packing limit (`P1 R20`, `P1 R63`).
- Preserve R60 cross-order compatibility at the packing seam: one Packing TU can mix orders only under the allowed same-customer/address/priority/delivery compatibility and all its content must belong to one Shipment.
- R26 defines the grouping seam only. Full Shipment creation/readiness/partial-shipment gating is P1-011. Carrier selection, labels, ERP, manifests and dispatch are later scope.

## Architecture/source routing

For Outbound business behavior:

1. current `01_ARCHITECT_SOURCE` / faithful Architect translations;
2. `02_CANON` and exact traceability/task sources;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Installed `wms-outbound` skill routes Outbound work to this chain.

Installed `architecture-context` is **reference-only** for accepted Inbound/shared Inventory/TU/warehouse/lock/orchestration compatibility. It does not override Outbound R-rules or justify expanding a ticket into later scope.

Inbound remains **CLOSED / REFERENCE** except targeted shared regression when a current Outbound diff touches shared primitives.

## Mandatory operating workflow

Read and follow:

`06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

Canonical behavior:

1. Supervisor grounds the current item and writes the detailed Antigravity execution/remediation guide to `WMS_Outbound/06_AGENT_GUIDES/` **and pushes it first**.
2. Owner receives only a short launch prompt telling the SAME Antigravity session to sync `WMS_Outbound/main`, read the local guide, execute/push and STOP.
3. Owner normally returns only `done`.
4. Supervisor independently fetches current Mercato/Scanner/WMS refs, diffs, tests/evidence and verifies them. Never ask the owner to paste long executor logs.
5. `SHOT -> RECEIVE -> VERIFY EVIDENCE -> NEXT SHOT`.
6. Same material path: two strikes then STOP unless the owner explicitly overrides. Each override permits one narrow further shot.
7. Existing Antigravity session only; preferred wrapper `/home/ubuntu/.local/bin/agy-pl`, never bare `agy`.

## Evidence rules that must not be lost

- real PostgreSQL only for DB claims; no fake EM/Map/simulated PG errors;
- real concurrency = independent overlapping operations/transactions plus DB-side evidence tied to actual participants;
- real rollback = real write/flush/failure-before-commit + fresh independent read;
- real UI claim = decisive actor action through rendered UI;
- automated browser evidence label = `PLAYWRIGHT VERIFIED`;
- executor narrative is never acceptance evidence by itself;
- exact commands, exact current 40-char implementation heads and exact test titles/output must be recorded in durable evidence.

## Fresh-session startup

A fresh supervisor session should read/refresh, in this order:

1. this handover;
2. `STATE.md`;
3. `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`;
4. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact P1-010 section;
5. exact P1-010 Architect source / requirements / test scenarios;
6. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-03.md`;
7. `Devaxonic-WMS/.ai/STATE.md`, `.ai/PLAN.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`;
8. current installed `wms-outbound`, `architecture-context`, `fetch_me_prompt`, `operational-mode` and current real-evidence contract.

Then independently verify current Git refs before writing the first P1-010 guide.

Do not reconstruct mutable state or operating rules from old chat history.