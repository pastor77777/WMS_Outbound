# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-05 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Current progress: **19/37 FINAL PASS**.

Latest accepted checkpoint:

`P1-016` — Mercato `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `50c5664fda12caf5c2f7bcdb0e9e86c3495a01c2` — **FINAL PASS / Owner Accepted**.

Accepted proof:

- P1-016 PostgreSQL **25/25**;
- real PostgreSQL Race A/B lock evidence + rollback proof;
- mandatory regressions **9 suites / 187/187**;
- dedicated P1-016 Playwright A–F **6/6** against canonical Testing on exact final Mercato SHA;
- Scanner unchanged.

## Accepted P1-016 boundary

Preserve:

- exact-once settlement per confirmed manifest contribution;
- no early terminalization under partial multi-manifest coverage;
- exact Allocation reservedQty / OutboundOrderLine shippedQty arithmetic;
- OutboundOrder completion only after all relevant Shipment manifests are confirmed;
- CustomerOrderLine/CustomerOrder aggregation per Architect F1;
- cancellation hard boundaries and atomic whole-order rejection;
- real PostgreSQL serialization for same-manifest replay and different-manifest shared-line settlement.

## Active item

**P2-001 — Inbound crossdock boundary and demand/source eligibility — item 20/37.**

Grounding:

- requirements: `FR-P2-01`, `FR-P2-03`, `FR-P2-23`, `FR-P2-24`, `FR-P2-25`, `INT-01`, `CON-03`;
- dependencies satisfied: FND-001, FND-003, P1-002, P1-003;
- source: P2 R1–R2, R5–R6, R41–R43, KROK 1, R29–R30;
- acceptance: `TC-020`, `TC-029`, `TC-036`, `TC-092`, `TC-102`, `TC-103`, `TC-119`, `TC-134`;
- exact Mercato base: `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574`.

Authoritative behavior:

- only Inbound `TU` `ELEMENTARY` already at `IN_CROSS_DOCK` enters Outbound crossdock;
- Inbound qualification is only a transport premise; binding demand matching happens at `IN_CROSS_DOCK` against current `BACKORDERED` demand;
- `sourceEligibleQty = ASN qty - active plannedQty - completed confirmedQty - completed damagedQty`;
- `demandEligibleQty = CustomerOrderLine.Quantity - ATPReservation - sum(requiredQty of all non-CANCELLED OutboundOrderLines, all channels)`;
- `crossDockEligibleQty = min(sourceEligibleQty, demandEligibleQty)`;
- concurrent matching cannot assign any quantity twice and follows warehouse priority order;
- zero match creates no crossdock work and returns the full declared quantity as residual to the Inbound handoff;
- P2-001 does not create P2-002 CrossDockPickTask/OutboundOrder planning and does not implement Scanner crossdock sorting.

Guide:

`06_AGENT_GUIDES/P2-001_EXECUTION.md`

## Hard exclusions

- no P2-002 task/order planning;
- no P2-003 Scanner sorting;
- no GR gate (P2-005);
- no P3/P4 execution;
- no Return Receipt;
- no external carrier API;
- no Prod/Demo;
- no reinterpretation of accepted Inbound qualification/TU semantics.

## Supervisor protocol

- Owner controls executor selection, launch, resume and session organization;
- launch prompts are microscopic and contain no appended questions/suggestions;
- executor prose is not acceptance; supervisor verifies refs/diff/evidence independently;
- Testing credentials are designated Testing data and are not a feature-work target;
- do not modify `AGENTS.md` or unrelated `.ai/*` policy files as part of feature delivery.