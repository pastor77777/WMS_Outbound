# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-05 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Current progress: **20/37 FINAL PASS**.

Latest accepted checkpoint:

`P2-001` — Mercato `8a264fff5c2ca665294d1e02df90c6f37554fe7f` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `941d614966b1d2197d8654b3af924afb6ab14d58` — **FINAL PASS / Owner Accepted**.

Accepted proof:

- dedicated P2-001 PostgreSQL **10/10**, mapping all 20 required scenarios;
- literal real PostgreSQL CON-03 backend-PID/lock-wait evidence;
- focused regressions **4 suites / 38/38**;
- accepted Inbound cross-dock result suite final **3/3** after explicit test-harness `AUTOMATIC` prerequisite with restoration;
- base P1-016 reproduced Inbound **1/3** under ambient `MANUAL`, proving the original red regression was not introduced by P2-001;
- Scanner unchanged.

## Accepted P2-001 boundary

Preserve:

- only Inbound `TU ELEMENTARY` already at `IN_CROSS_DOCK` is eligible;
- binding demand is recalculated at boundary time;
- exact source/demand/crossdock quantity arithmetic and all-channel demand coverage deduction;
- `ACTIVE`, `CONSUMED`, `DAMAGED` source binding quantity remains deducted; only `RELEASED` returns unexecuted capacity;
- deterministic zero-match residual fact and replay idempotency;
- real PostgreSQL source/demand serialization with no double assignment;
- no P2-002 work was created by P2-001.

## Active item

**P2-002 — Crossdock OutboundOrder/Line and CrossDockPickTask planning — item 21/37.**

Grounding:

- requirements: `FR-P2-02`, `FR-P2-03`, `FR-P2-04`, `FR-P2-16`, `FR-P2-21`, `FR-P2-22`, `FR-P2-25`, `CON-03`;
- dependencies satisfied: P2-001, FND-002;
- source: P2 R3–R8, R29–R30, R39–R40, R43;
- acceptance: `TC-020`, `TC-021`, `TC-022`, `TC-029`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-036`, `TC-092`, `TC-099`, `TC-101`, `TC-119`, `TC-134`;
- exact Mercato base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`;
- Scanner baseline if bounded assignment UI is needed: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

Authoritative behavior:

- consume accepted positive P2-001 binding truth; do not create a second reservation truth;
- create immutable `fulfillmentChannel = CROSSDOCK` OutboundOrder/Line and exactly one CrossDockPickTask per planned line;
- `CrossDockPickTask.plannedQty` is the authoritative quantity lock; replay/concurrency cannot double-plan source quantity;
- no `Allocation` exists for crossdock;
- grouping requires same customer/address/priority and identical `slaDeadline`; `allowPartialShipment = false` keeps grouping inside one CustomerOrder;
- physical target Outbound TU creation/numbering remains lazy until first placement and therefore is not performed by P2-002;
- Scanner scope is bounded to entering/selecting the crossdock module and receiving the next task without a zone selector, under the accepted single-active-warehouse-task and queue-ordering rules;
- no P2-003 source/SKU/quantity/quality scans, target placement, sealing, completion or shortage/QC handling.

Guide:

`06_AGENT_GUIDES/P2-002_EXECUTION.md`

## Hard exclusions

- no P2-003 RF sorting/execution;
- no eager physical target Outbound TU creation/numbering/labeling;
- no Allocation;
- no GR gate or later P2 shipment/ERP work;
- no P3/P4 execution;
- no Return Receipt;
- no Prod/Demo;
- no reinterpretation of accepted Inbound semantics.

## Supervisor protocol

- Owner controls executor selection, launch, resume and session organization;
- launch prompts are microscopic and contain no appended questions/suggestions;
- executor prose is not acceptance; supervisor verifies refs/diff/evidence independently;
- Testing credentials are designated Testing data and are not a feature-work target;
- do not modify `AGENTS.md` or unrelated `.ai/*` policy files as part of feature delivery.
