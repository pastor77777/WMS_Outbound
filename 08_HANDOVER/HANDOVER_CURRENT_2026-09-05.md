# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-05 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Current progress: **23/37 FINAL PASS**.

Latest accepted checkpoint:

`P2-004` — Mercato `9859be5c7dee4fe802d4d00478459a19982eddfe` / Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `9fb9abd33c1ff8318b6339efc9b69cce3a3161ac` — **FINAL PASS / Owner Accepted**.

P2-004 was independently supervisor-verified, corrected for Warehouse Supervisor role/state authorization, re-verified, and then explicitly accepted by Owner.

## Accepted P2-004 proof

- remote Mercato `outbound/p2-004` exactly `9859be5c7dee4fe802d4d00478459a19982eddfe`;
- remote Scanner `outbound/p2-004` exactly `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- both final product heads descend linearly from accepted P2-003 bases, two commits ahead / zero behind;
- dedicated P2-004 canonical PostgreSQL suite **16/16 PASSED**;
- P2-003 PostgreSQL **8/8 PASSED**;
- P2-002 PostgreSQL **22/22 PASSED**;
- P2-001 PostgreSQL **10/10 PASSED**;
- Mercato full generate/build-packages/typecheck/build-app contract PASSED;
- real P2-004 Playwright **4/4 PASSED** with zero route mocks/interception;
- P2-003 Scanner real regression **2/2 PASSED**;
- P2-002 Scanner assignment regression **1/1 PASSED**;
- P1-008 Scanner identity regression **1/1 PASSED**;
- canonical Mercato runtime active/running on port 3009, HTTP 200, non-empty production manifests;
- canonical Scanner runtime active/running on port 8081, HTTP 200;
- final evidence records exact pushed product SHAs and does not self-declare Owner acceptance.

## Accepted P2-004 authorization correction

The first P2-004 closeout was rejected at supervisor gate because the Packer could execute Architect-defined Warehouse Supervisor decisions.

The accepted corrective now guarantees:

- normal Packer `/api/wms_outbound/cross-dock/execute` has no `supervisor_decision` action;
- Scanner Packer UI has no `WAIT`, `CANCEL`, or `ALLOW_PARTIAL` controls;
- Packer sees an escalation/waiting state only;
- dedicated Supervisor endpoint `/api/wms_outbound/supervisor/cross-dock-decision` requires `wms_outbound.manage_orders`;
- Supervisor identity comes from authenticated actor, not client-supplied id;
- Packer direct HTTP attempt is rejected with HTTP 403 and zero side effect;
- Supervisor decision is legal only for `CrossDockPickTask SHORT_PICKED` + `exceptionReason = SHORTAGE_ESCALATED`;
- first accepted decision is immutable/replay-safe;
- late `unexpected SKU` after final task state is rejected with zero discrepancy side effect;
- real role-separated Scanner Packer -> rendered Mercato Warehouse Supervisor flow is covered by Playwright.

This role/state boundary is frozen for all later P2 work.

## Accepted P2-004 functional boundary

Preserve:

- shortage with `allowPartialShipment=true` automatic partial packing/backorder behavior;
- `DAMAGED` segregation to QC without contaminating OK target quantity;
- `allowPartialShipment=false` shortage escalation to Warehouse Supervisor;
- empty source TU cancellation / source `LOST` semantics;
- general cancellation guard while task is `IN_PROGRESS`;
- exact quantity conservation `confirmed + damaged + residual = declared`;
- source finalization and physical-return handoff facts;
- no orphan active target/task after recovery;
- accepted P2-003 normal OK path remains unchanged and Allocation-free.

P2-004 explicitly stops before P2-005 GR correlation/gating and P2-006 automatic Shipment/dispatch joining.

## Active item

**P2-005 — Goods Receipt correlation gate and re-evaluation — item 24/37.**

Authoritative guide:

`06_AGENT_GUIDES/P2-005_EXECUTION.md`

Frozen accepted bases:

- Mercato `9859be5c7dee4fe802d4d00478459a19982eddfe`;
- Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- P2-004 evidence `9fb9abd33c1ff8318b6339efc9b69cce3a3161ac`.

Task Catalog objective:

- correlate Goods Receipt results by source Inbound TU;
- store crossdock GR acceptance status for every contributing source;
- gate ERP Shipment posting until all of that Shipment's contributing crossdock sources have `GR_ACCEPTED`;
- re-evaluate the gate on later GR messages.

Requirements:

`FR-P2-13`, `FR-P2-14`, `FR-P2-17`, `FR-P2-18`, `FR-P5-16`, `INT-02`, `INT-03`.

Acceptance mapping:

`TC-028`, `TC-030`, `TC-031`, `TC-038`, `TC-067`.

## P2-005 Architect truth

The executor must preserve these exact semantics:

1. Correlation is only by `sourceInboundTU` + `GR_SETTLEMENT_SOURCE = CROSSDOCK`; no task id is required.
2. Settlement/message version is not a discriminator between CROSSDOCK and PUTAWAY.
3. A valid GR result updates the same `grAcceptanceStatus` on every CrossDockPickTask for that source TU, even if those tasks feed multiple Shipments.
4. Unknown source TU or non-CROSSDOCK/PUTAWAY result changes zero crossdock GR status.
5. Gate is computed separately per Shipment from all of its own distinct contributing source TUs.
6. One accepted source never unlocks a Shipment that also depends on another pending/rejected source.
7. Residual quantity and later Putaway settlement do not participate in or regress the crossdock GR gate.
8. An initial P1-014 ERP post attempt while the gate is unsatisfied must stop before `POSTING_PENDING` and before posting/attempt/outbox/ERP-adapter side effects.
9. `GR_REJECTED` itself does not set Shipment `POSTING_ERROR`; it only leaves the gate unsatisfied.
10. Later `GR_ACCEPTED` after rejection re-evaluates normally and requires no manual data repair.
11. A Shipment already `POSTING_ERROR` for a true ERP error also re-evaluates on GR messages, but P2-005 must not auto-retry ERP posting; accepted P1-014 Warehouse Supervisor retry remains authoritative.
12. Warehouse Supervisor can inspect contributing source GR statuses; there is no manual GR override/escalation state.
13. Inbound retains responsibility for retrying Goods Receipt messages.
14. P2-005 does not implement P2-006 automatic Shipment formation/joining.

## Mandatory deep reset before P2-005

P2-005 is a **new Task Catalog item**, not a continuation of P2-004.

Before any P2-005 implementation action, run:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

The reset is mandatory new-item Testing hygiene only. It must not:

- downgrade canonical Supabase schema;
- remove accepted P2-003/P2-004 migrations;
- reset Mercato/Scanner to old P2 bases;
- rewrite accepted branch history.

After reset, any changed product branch `outbound/p2-005` starts from the exact frozen P2-004 head above.

## P2-005 test/evidence contract

Dedicated canonical PostgreSQL coverage must explicitly map all **18 substantive behaviors** listed in `P2-005_EXECUTION.md`, including:

- pending/non-accepted initial state;
- accepted/rejected correlation to all tasks of one source;
- reject -> accepted normal re-evaluation;
- duplicate/idempotent GR result;
- unknown-source no-op;
- PUTAWAY no-op;
- version is not settlement-source discriminator;
- per-Shipment multi-source gate;
- one source feeding two Shipments;
- residual/Putaway isolation;
- P1-only Shipment non-regression;
- blocked post has zero P1-014 side effects / ERP adapter calls;
- POSTING_ERROR re-evaluation without auto-retry;
- tenant/org isolation;
- Supervisor read model.

Mandatory regressions:

- P1-014 ERP posting PostgreSQL accepted **18/18** behavior;
- P2-004 PostgreSQL **16/16**;
- P2-003 PostgreSQL **8/8**;
- Mercato repository-native full generate/build/typecheck/build-app contract.

P2-002/P2-001 broader regressions only if their surfaces are touched. Shared/Inbound regressions only if shared primitives are actually modified.

Real Playwright uses canonical runtime and zero route mocks. It must prove:

- two-source Shipment blocked in rendered Mercato UI until all required crossdock GR statuses are accepted;
- blocked rendered posting action produces zero posting side effects;
- later valid GR_ACCEPTED unblocks the gate and normal P1-014 post can proceed;
- POSTING_ERROR case re-evaluates gate but does not auto-retry; existing Supervisor retry remains required;
- PUTAWAY result does not alter crossdock gate/status;
- Supervisor sees exact source statuses/blockers.

Scanner has no direct P2-005 decision. Keep Scanner frozen at accepted P2-004 SHA unless a concrete P2-005 defect proves a Scanner change is necessary.

## Hard exclusions for P2-005

- no P2-006 automatic crossdock Shipment/dispatch joining;
- no CarrierManifest implementation/change;
- no unrelated carrier-selection redesign;
- no ERP payload redesign beyond the required GR precondition;
- no P3;
- no new P4 work;
- no Return Receipt;
- no Inbound GR retry ownership transfer;
- no Demo/Prod;
- no local PostgreSQL.

## Executor/session transition

Owner is using **Antigravity**.

P2-005 should start in a **new Antigravity chat/session** because it is a new Task Catalog item.

Launch prompt stays microscopic; detailed scope is only in `06_AGENT_GUIDES/P2-005_EXECUTION.md`.

## Supervisor protocol

- Owner controls executor selection, launch and session organization;
- executor never self-accepts;
- workflow remains `SHOT -> RECEIVE -> VERIFY EVIDENCE -> NEXT SHOT`;
- detailed instructions live in Git, owner-facing prompts remain short;
- two genuine attempts on one material technical path then STOP unless Owner authorizes a distinct path;
- only supervisor verification followed by explicit Owner acceptance advances the catalog counter;
- do not start P2-006 before P2-005 is supervisor-verified and Owner Accepted.