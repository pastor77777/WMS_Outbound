# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-05 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Current progress: **24/37 FINAL PASS**.

Latest accepted checkpoint:

`P2-005` — Mercato `069f02d4c5c9b345b688b838eb685be02206afbd` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `0c7cf142e1723ff80e86cfd0f00d4b12c1e4b777` / supervisor correction `cf399679360d8b7fc071f9f958709c3bb99b7c59` — **FINAL PASS / Owner Accepted**.

P2-005 was independently supervisor-verified and explicitly accepted by Owner.

## Accepted P2-005 proof

- remote Mercato `outbound/p2-005` exactly `069f02d4c5c9b345b688b838eb685be02206afbd`;
- Mercato final is one commit ahead / zero behind accepted P2-004 base `9859be5c7dee4fe802d4d00478459a19982eddfe`;
- Scanner stayed frozen/clean at accepted P2-004 SHA `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- dedicated P2-005 canonical PostgreSQL **19/19 PASSED** with all 18 required substantive behaviors mapped and quantity/finalization preservation proved;
- P1-014 ERP posting regression **18/18 PASSED**;
- P2-004 recovery regression **16/16 PASSED**;
- P2-003 execution regression **8/8 PASSED**;
- Mercato generate/build-packages/typecheck/build-app contract PASSED;
- real Mercato P2-005 Playwright **3/3 PASSED**, zero route mocks/interception;
- Journey A proved a two-source crossdock Shipment blocked before posting side effects until all required sources became `GR_ACCEPTED`;
- Journey B proved GR re-evaluation while Shipment remained `POSTING_ERROR`, no automatic retry, and successful existing Warehouse Supervisor retry;
- Journey C proved PUTAWAY settlement does not alter the CROSSDOCK GR gate;
- canonical Mercato runtime active/running on 3009 with HTTP 200 and non-empty production manifests;
- canonical Scanner runtime remained healthy on 8081;
- evidence does not self-declare Owner acceptance.

Supervisor correction `cf399679360d8b7fc071f9f958709c3bb99b7c59` fixes a documentation-only typo: the successful posting row is `POSTED`; the posting-attempt outcome is `ACCEPTED`. No product/test/runtime SHA changed.

## Frozen P2-005 business boundary

Preserve exactly:

1. GR identity is `sourceInboundTU` + `GR_SETTLEMENT_SOURCE=CROSSDOCK`; no CrossDockPickTask id is required.
2. Message/version is not a settlement-source discriminator.
3. A valid CROSSDOCK result sets the same `grAcceptanceStatus` on all scoped CrossDockPickTasks of the source TU.
4. Unknown source and non-CROSSDOCK/PUTAWAY results produce zero crossdock GR mutation.
5. The gate is evaluated separately per Shipment from all distinct source Inbound TUs contributing confirmed crossdock quantity.
6. `GR_REJECTED` leaves the gate blocked but does not itself create Shipment `POSTING_ERROR`; later `GR_ACCEPTED` re-evaluates normally.
7. Residual/Putaway settlement does not participate in or regress the CROSSDOCK gate.
8. Both initial ERP posting and P1-014 Supervisor retry are blocked before Phase-1 side effects if the gate is unsatisfied.
9. A real ERP `POSTING_ERROR` remains under accepted P1-014 retry semantics; GR messages never auto-retry ERP posting.
10. P1-only Shipment posting remains unchanged.
11. Warehouse Supervisor can see contributing source GR statuses/blockers in normal Shipment UI.
12. Inbound retains Goods Receipt retry ownership.
13. P2-005 does not create or join automatic crossdock Shipment downstream; that begins only in P2-006.

Inherited P2-004 role boundary also remains frozen: ordinary Packer/Scanner cannot execute Warehouse Supervisor `WAIT` / `CANCEL` / `ALLOW_PARTIAL` decisions.

## Active item

**P2-006 — Crossdock join into common Shipment/dispatch downstream — item 25/37.**

Authoritative guide:

`06_AGENT_GUIDES/P2-006_EXECUTION.md`

Guide commit:

`98fb8988eaa256cb79191bc3091c56e028abd903`

Frozen accepted bases:

- Mercato `069f02d4c5c9b345b688b838eb685be02206afbd`;
- Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- P2-005 evidence `0c7cf142e1723ff80e86cfd0f00d4b12c1e4b777`;
- supervisor evidence correction `cf399679360d8b7fc071f9f958709c3bb99b7c59`.

## P2-006 Architect truth

P2-006 must close P2 by joining crossdock into the already accepted P1 downstream lifecycle — **not by creating a second lifecycle**.

1. Legitimate crossdock Packing TUs in `PACKING_SEALED` use the same persisted Shipment/TU/content/OutboundOrderLine relation as P1.
2. There is no separate crossdock Shipment, carrier, label, ERP posting, CarrierManifest or settlement model.
3. Common Shipment grouping key remains warehouse + customer + delivery address + priority + identical `slaDeadline`.
4. P2 R43 priority/SLA inheritance and grouping restrictions remain authoritative.
5. P1 R57/R58 is CustomerOrder-level and channel-neutral: STANDARD and CROSSDOCK packed coverage is summed together.
6. For `allowPartialShipment=false`, an incomplete active line in either channel keeps ready Packing TUs outside every Shipment; `slaDeadline` cannot bypass this guard.
7. Once complete and compatible, STANDARD + CROSSDOCK Packing TUs of that CustomerOrder may enter one complete Shipment (TC-125).
8. `allowPartialShipment=true` follows ordinary common Shipment grouping/readiness; there is no crossdock-only wait rule.
9. Common P1-012 Carrier Selection and P1-013 label lifecycle consume the combined Shipment normally.
10. P2-005 GR gate remains before P1-014 ERP Phase 1 and only crossdock source TUs create GR requirements.
11. After valid GR acceptance, the same P1-014 posting path proceeds; no crossdock posting service is allowed.
12. POSTED crossdock/mixed Shipment uses the existing P1-015 CarrierManifest lifecycle and its irreversible/idempotent boundaries.
13. P1-016 remains final settlement authority: STANDARD Allocation/Inventory effects settle exactly once; CROSSDOCK creates no fake Allocation and no standard Inventory decrement merely to fit P1, while its TU/line/order/customer states still advance through the common lifecycle.
14. Multi-Shipment/final aggregate and replay behavior must remain accepted P1-016 behavior.
15. Scanner has no second dispatch model and should stay frozen unless a concrete P2-006 defect proves otherwise.

Requirements:

`FR-P2-13`, `FR-P2-18`, `FR-P2-25`, `FR-P1-31`, `FR-P1-32`, `CON-04`, `CON-05`.

Acceptance mapping:

`TC-028`, `TC-030`, `TC-031`, `TC-104`, `TC-105`, `TC-119`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`.

## Mandatory deep reset before P2-006

P2-006 is a **new Task Catalog item**.

Before any P2-006 implementation action:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

The reset is Testing/worktree hygiene only. It must not downgrade Supabase schema, remove accepted P2 migrations, reset products to older bases, or rewrite accepted history.

After reset, any changed product branch `outbound/p2-006` starts from the exact frozen accepted P2-005 head above.

## P2-006 test/evidence contract

Dedicated canonical PostgreSQL proof must explicitly map all **20 substantive behaviors** in `P2-006_EXECUTION.md`, including:

- common Shipment model / no crossdock fork;
- exact grouping key and one-Shipment TU membership;
- P2 R43 priority/SLA preservation;
- both directions of mixed-channel `allowPartialShipment=false` blocking;
- TC-122/123/124 equality/correction branches;
- TC-125 one complete mixed STANDARD+CROSSDOCK Shipment;
- slaDeadline cannot bypass the guard;
- allowPartial true normal common grouping;
- common P1 readiness/carrier/label behavior;
- P2-005 gate in the combined Shipment;
- common ERP posting;
- common CarrierManifest lifecycle;
- provenance-correct exactly-once final settlement with no fake crossdock Allocation/Inventory decrement;
- multi-Shipment/replay idempotency.

Mandatory regressions:

- P2-005 GR gate **19/19**;
- full accepted P1-011 Shipment grouping suite;
- full P1-012 Carrier Selection suite;
- full P1-013 label suite;
- P1-014 ERP posting **18/18**;
- full P1-015 CarrierManifest suite;
- full P1-016 final settlement suite;
- P2-002 planning **22/22** for P2 R43 inheritance;
- P2-004 **16/16** and P2-003 **8/8** if execution/finalization/TU-state surfaces are touched; otherwise record why they remain retained rather than rerun;
- broader/shared/Inbound regressions only for actually touched shared primitives.

Real canonical Mercato Playwright uses zero route mocks and must prove:

- **Journey A:** real crossdock Packing TU -> common Shipment -> Carrier/label -> GR-blocked ERP -> GR_ACCEPTED -> ERP POSTED -> common CarrierManifest close/handover/confirm -> persisted final reconciliation;
- **Journey B:** one `allowPartialShipment=false` CustomerOrder split STANDARD + CROSSDOCK; incomplete channel blocks all TUs, deadline cannot bypass, completion joins both TUs into one Shipment, then common downstream completes;
- **Journey C:** a real grouping incompatibility or replay/second-Shipment negative boundary.

Scanner should remain frozen unless proven necessary.

## Hard exclusions for P2-006

- no second crossdock Shipment schema/lifecycle;
- no second crossdock carrier/label/ERP/manifest/settlement model;
- no external carrier provider API additions;
- no P3 implementation;
- no new P4 work;
- no Return Receipt;
- no Inbound behavior/retry ownership changes;
- no Demo/Prod;
- no local PostgreSQL;
- no unrelated refactors.

## Executor/session transition

Owner is using **Antigravity**.

P2-006 must start in a **new Antigravity chat/session** because it is a new Task Catalog item.

Launch prompt stays microscopic; detailed scope is only in `06_AGENT_GUIDES/P2-006_EXECUTION.md`.

## Supervisor protocol

- Owner controls executor selection, launch and session organization;
- executor never self-accepts;
- workflow remains `SHOT -> RECEIVE -> VERIFY EVIDENCE -> NEXT SHOT`;
- detailed instructions live in Git, owner-facing prompts remain short;
- two genuine attempts on one material technical path then STOP unless Owner authorizes a distinct path;
- only supervisor verification followed by explicit Owner acceptance advances the catalog counter;
- do not start P3-001 before P2-006 is supervisor-verified and Owner Accepted.