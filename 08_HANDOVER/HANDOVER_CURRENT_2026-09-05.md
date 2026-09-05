# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-05 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Current progress: **21/37 FINAL PASS**.

Latest accepted checkpoint:

`P2-002` — Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93` / Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440` / evidence `2074e2541b7c28cf3c6031cb48ad68901333625a` — **FINAL PASS / Owner Accepted**.

Accepted proof:

- architecture-aligned CON-03 PASS and P2-001 regression **10/10**;
- P2-002 dedicated canonical PostgreSQL **22/22**;
- remaining mandatory Mercato regressions **5 suites / 41 tests**;
- current-UI P1-005 Scanner regression **1/1**, zero route mocks;
- dedicated P2-002 Scanner Playwright **1/1**, zero route mocks;
- real Crossdock request-task outcomes: operator A HTTP 200 expected task, operator B HTTP 200 distinct task, blocked operator HTTP 400 active-task rejection;
- rendered task number/source/plannedQty/status/no-zone/P2-003 boundary plus persisted ownership, two task locks and unchanged Allocation count;
- canonical Mercato 404 was classified as stale generated/build/runtime routing and recovered by repository-native full `yarn build` including `yarn generate` plus canonical `mercato-localhost.service` restart; no Mercato source change;
- final Mercato and Scanner worktrees clean.

## Accepted P2-002 boundary

Preserve:

- positive accepted P2-001 binding becomes exactly one CROSSDOCK OOL + CrossDockPickTask;
- no Allocation in crossdock;
- exact `plannedQty` source quantity lock;
- immutable CROSSDOCK channel and accepted grouping/allowPartial semantics;
- no target Outbound TU before first placement;
- Crossdock RF entry/assignment has no zone selector and respects warehouse queue + one-active-task guard;
- P2-002 stops before sorting, quality/quantity confirmation, target placement/sealing, completion and shortage handling.

## Active item

**P2-003 — RF crossdock sorting into outbound TUs — item 22/37.**

Authoritative guide:

`06_AGENT_GUIDES/P2-003_EXECUTION.md`

Frozen accepted bases:

- Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
- Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`;
- P2-002 evidence `2074e2541b7c28cf3c6031cb48ad68901333625a`.

Requirements:

`FR-P2-02`, `FR-P2-04`, `FR-P2-05`, `FR-P2-10`, `FR-P2-12`, `FR-P2-19`, `FR-P2-21`, `FR-P2-22`.

Acceptance mapping:

`TC-020`, `TC-021`, `TC-022`, `TC-023`, `TC-027`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-035`, `TC-037`, `TC-099`, `TC-101`.

**Important staged-mapping rule:** these TC IDs are traceability mappings, not permission to close every future step of each full scenario inside P2-003. P2-003 proves only the slice supported by its own requirements/Architect refs. In particular: Shipment/dispatch portions of TC-020/021 remain P2-006/ACC; shortage/Supervisor TC-023 remains P2-004; residual/finalization exception portions of TC-027 remain P2-004; GR portions of TC-030 remain P2-005; cancellation/PutBack portions of TC-035 remain P2-004/P4. Later tasks/ACC close the full scenario.

Authoritative behavior:

- first real scan starts execution: task `ASSIGNED → IN_PROGRESS`, OOL `CREATED → PICKING`, first line start moves order `CREATED → PACKING_IN_PROGRESS`;
- scan/confirm normal `OK` SKU/quantity into target Outbound TU(s);
- target TU is created lazily at first placement, not during P2-002 planning;
- 1:1 with valid GS1 source SSCC inherits TU_NUMBER/SSCC; otherwise and for n:n use accepted P1-008 TUSetup/Sequence identity generation;
- physically full target can be RF-sealed and the same task continues into a new open target TU;
- task completion fixes an idempotent `confirmedQty <= plannedQty`;
- P2-003 normal path must support both 1:1 and n:n through the real Scanner UI;
- shortage/DAMAGED/unexpected SKU/empty source/cancellation/residual exception logic remains P2-004.

## Test/evidence lessons — mandatory from P2-003 through ACC-004

1. Before every Playwright run, prove the exact intended Testing runtime/revision. Source SHA alone is not enough; route generation/build/service state must match.
2. For Mercato route/product changes, use repository-native full build/generate and canonical `mercato-localhost.service` restart; probe new routes before browser assertions.
3. For Scanner source changes, restart canonical `scanner-testing.service`; its `ExecStartPre` performs fresh `npx expo export --platform web`. Do not let Playwright `reuseExistingServer` silently reuse stale exported UI.
4. Scanner browser tests wait for real warehouse resolution/selection before module entry. `Choose mode` alone is not readiness.
5. Assertions use the current tested SHA's UI contract (`testID`, accessibility, current copy), not historical accepted text.
6. A historical regression spec is not automatically authoritative after later accepted UI evolution. On red, first compare the new diff, current product contract and fixture prerequisites.
7. Concurrency proof targets the Architect-required business outcome using genuine overlapping PostgreSQL operations. Do not make incidental `pg_stat_activity`/`pg_blocking_pids` shapes into requirements unless Architect Source says so.
8. Diagnose failures by layer: 404/5xx/API-null → runtime/route/API/persistence first; correct API + wrong UI → render defect.
9. Fixtures explicitly set org/tenant/warehouse/role/zone/policy prerequisites and restore shared configuration. Never depend on ambient Testing defaults or cardinality.
10. Local PostgreSQL is forbidden. Canonical DB is Supabase Testing project `yzonugcenguvmojwiihb` through the approved environment source.
11. Shared/Inbound regression is required only for actually touched Inventory/TU/warehouse/task-lock/orchestration primitives; do not run broad historical suites by habit.
12. Design the real rendered Playwright journey together with implementation, including real request evidence and persisted reconciliation; do not bolt it on at the end.
13. X/ACC stages may retain prior proof only if the relevant surface/semantics and evidence class remain truthful; never relabel stale evidence.
14. Per-item Owner Accepted is not automatically `Human Verified`; only an actual human traversal may carry that label. ACC-004 remains the final human walkthrough gate.
15. Two genuine attempts on one material technical path then STOP unless Owner authorizes a distinct narrow path.

## Remaining-plan hotspots

- **P2-003/P2-004:** real RF 1:1+n:n sorting, target-TU identity/continuity and quantity conservation are the main risk.
- **P2-005/P2-006:** integration correlation + generated route/runtime provenance are the main risk; prove route presence before UI gate assertions.
- **P3:** authoritative `pickedQty` determines P3 vs P4; pre-confirm physical race returns to exact source and never creates PutBackTask.
- **P4:** logical cancellation is immediate; physical recovery is asynchronous; PutBack only for `pickedQty > 0`; invalid-location loop has no retry limit or auto escalation.
- **X-001/X-002:** test business exactly-once/correlation outcomes, not implementation diagnostics.
- **ACC-001..004:** freeze exact Mercato/Scanner/runtime revisions; normal UI actions remain decisive, while API/DB is setup/verification only.

## Hard exclusions for P2-003

- no P2-004 shortage/damage/empty/cancellation recovery;
- no GR gate or P2-005;
- no Shipment/ERP/manifest P2-006 work;
- no P3/P4 implementation;
- no Return Receipt;
- no Demo/Prod;
- no reinterpretation of accepted Inbound semantics.

## Supervisor protocol

- next executor run should use a **new Codex session** bootstrapped only from current Git state/guide;
- Owner controls executor selection, launch and session organization;
- launch prompts stay microscopic;
- executor prose is not acceptance; supervisor independently verifies refs/diff/evidence;
- Testing credentials are designated Testing data and are not a feature-work target;
- do not modify Architect Source/Canon/Task Catalog/AGENTS/.ai steering unless separately authorized.