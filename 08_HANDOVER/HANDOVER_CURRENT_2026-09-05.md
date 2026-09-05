# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-05 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 implementation/acceptance items**, **109/109 Architect requirements mapped**.

Current progress: **18/37 FINAL PASS**.

Latest accepted checkpoint:

`P1-015` — Mercato `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / durable evidence `f201bd0beb2411b4f87f28ca6562a4fc11e6a249` — **FINAL PASS / Owner Accepted**.

Accepted proof:

- P1-015 PostgreSQL **21/21**;
- prior integrated regression aggregate **171/171**;
- exact-session PostgreSQL lock proof for assignment, add-vs-close, duplicate confirm and carrier-correction-vs-close;
- fresh rendered Playwright **6/6** against canonical Testing served from exact final runtime `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`;
- closeout changed evidence only; Scanner stayed frozen.

## P1-015 accepted boundary

Preserve:

- `Shipment POSTED → IN_MANIFEST` only into one `OPEN` manifest;
- `OPEN → CLOSED` irreversible composition freeze;
- no post-close add/remove/reopen, Shipment cancellation or Carrier correction;
- physical `CLOSED → HANDED_OVER` separately advances Shipment/TU/OutboundOrder dispatch state;
- `HANDED_OVER → CONFIRMED` is distinct final warehouse confirmation, idempotent/exactly-once, with durable deterministic membership snapshot;
- P1-015 deliberately does not perform final Inventory/Allocation/line/order/customer settlement;
- no external carrier API and no Scanner change.

## Active item — execution in flight

Current catalog item: **P1-016 — Final line/order/inventory settlement and cancellation boundaries — item 19/37**.

Owner reported on 2026-09-05 that an owner-selected **Codex executor is currently executing** `06_AGENT_GUIDES/P1-016_EXECUTION.md`.

This records execution status only. **P1-016 remains unverified and unaccepted until the executor finishes and the supervisor independently verifies current refs/diff/evidence.** Do not advance STATE or acceptance from this in-flight fact.

The owner controls executor selection, launch, resume, session organization and sequencing. Do not issue executor-startup/session-management instructions unless explicitly requested, and do not redirect an in-flight executor.

Task grounding:

- objective: settle `OutboundOrderLine`/`Allocation`/`Inventory` quantities per confirmed contributing manifests, complete orders only after all contributing Shipment manifests are confirmed, continuously aggregate customer line/header state, and enforce cancellation boundaries;
- Architect: P1 R37–R38, R41–R42, R49–R50, continuous F1, R70–R72, general cancellation, plus P3/P4 entry routing;
- requirements: `FR-P1-19`, `FR-P1-21`, `FR-P1-25`, `FR-P1-27`, `FR-P1-44`, `FR-P1-45`, `FR-P1-46`, `FR-P5-09`, `INT-06`, `CON-05`;
- dependencies: P1-015, P1-004, P1-001 — satisfied;
- target components: Inventory ledger, Allocation, OutboundOrder/Line, CustomerOrder/Line, cancellation correlation/orchestration;
- acceptance: `TC-001`, `TC-003`, `TC-008`, `TC-009`, `TC-065`, `TC-128`, `TC-129`, `TC-130`, `TC-131`, `TC-132`, `TC-133`.

## P1-016 authoritative behavior

- a confirmed manifest settles only the quantity represented by its outbound TUs;
- `Inventory PICKED → SHIPPED` and `Allocation.reservedQty` decrement happen exactly once per settled quantity;
- partial multi-manifest settlement keeps `OutboundOrderLine PACKED` and `Allocation CONFIRMED`;
- only after all contributing outbound TUs belong to Shipments on `CONFIRMED` manifests may the line become `SHIPPED` and Allocation become `CONSUMED` with `reservedQty = 0`;
- `OutboundOrder DISPATCHED → COMPLETED` only after every Shipment containing any TU from the order is covered by a `CONFIRMED` manifest;
- CustomerOrderLine and CustomerOrder aggregation follows Architect F1 and must not terminalize early;
- duplicate/parallel same-manifest settlement and different-manifest settlement of one line/allocation must be exactly-once and arithmetically correct;
- cancellation from Shipment `POSTING_PENDING` onward is forbidden; after `CarrierManifest.CLOSED` general cancellation is forbidden;
- incoming cancellation/correction must correlate to the correct line and select P3 vs P4 using formal `pickedQty`; this task establishes the routing/boundary, not the future P3/P4 recovery implementation;
- Return Receipt remains outside scope.

Current executor guide:

`06_AGENT_GUIDES/P1-016_EXECUTION.md`

P1-016 Mercato branch baseline:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

## Hard scope exclusions

Do not absorb:

- P2 Crossdock implementation;
- P3/P4 physical recovery/PutBack execution;
- Return Receipt;
- external carrier APIs;
- unrelated Scanner work;
- Prod/Demo changes.

Scanner remains frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` unless a separately authorized future task requires it.

## Source and evidence routing

Authority order:

1. `01_ARCHITECT_SOURCE` / faithful Architect translations;
2. `02_CANON`, traceability and exact task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. `07_IMPLEMENTATION_PLAN` as delivery decomposition only.

Real PostgreSQL is mandatory for persistence/concurrency/rollback/exactly-once claims. Rendered application actions are mandatory for UI claims. Executor prose is never acceptance.

## Owner-facing supervisor protocol

Canonical rules are in `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

Preserve these explicit owner preferences:

- detailed work belongs in Git guides; owner-facing executor prompts stay microscopic;
- the owner decides executor/session mechanics;
- do not append questions, suggestions, menus or explanatory tails below a launch prompt unless explicitly requested;
- do not ask for long logs/screenshots/SHAs/secrets that can be independently verified;
- if an owner-side diagnostic is unavoidable, give one precise action/command at a time and request only minimal non-secret status.

## Steering boundary

P1-016 execution may change product code and its own evidence only within its guide. It must not update `STATE.md`, handovers, catalog, canon or Devaxonic `.ai/*`. Steering advances only after independent supervisor verification and explicit Owner Acceptance.
