# P1-007 Execution — SHORT_ALLOCATED / SHORT_PICKED Recovery

Owner has accepted `P1-006` as **FINAL PASS / Human Verified**. This is the first implementation shot for `P1-007` only.

Use the **same existing Antigravity session**. Do not restart, replace, or create a second agent session.

## FIRST: synchronize and refresh authority

1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main`, preserving unrelated local work.
3. Read the local current `STATE.md` and this file completely.
4. Refresh locally before implementation:
   - `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — `P1-007`;
   - `07_IMPLEMENTATION_PLAN/IMPLEMENTATION_PLAN.md`;
   - `01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — `R42`–`R48` and the SHORT exception table;
   - `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P1-21..24`, `FR-P5-01..06`;
   - `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — `TC-060`, `TC-061`, `TC-062`, `TC-121` plus mapped TC-001/TC-008 context;
   - current `fetch_me_prompt`, `operational-mode`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, and `REAL_EVIDENCE_CONTRACT.md` if present in the active implementation workspace.
5. Current Architect/Canon wins if anything below conflicts.

## Accepted bases — preserve exactly

- Mercato accepted P1-006 head: `353a5001cb8f1941971f960e509a8af643e41e5a`
- Scanner accepted P1-006 head: `7596b7802e7ed55a59dd6dc1f21912ea6331e796`
- P1-006 durable evidence: `WMS_Outbound` commit `43cc7d0e7dd20a48fc00b40150b30275d0c2aa12`
- P1-006 is Human Verified; accepted progress is `10/37`.

For Mercato, create/use a `P1-007` implementation branch descending from the exact accepted P1-006 head. Do not rewrite the accepted P1-006 branch/history. For Scanner, preserve the accepted head as the base and follow the existing repository branching convention.

## Ticket authority

**P1-007 — SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes**

Objective: shortages, automatic reallocation limits, wait/cancel outcomes and persistent partial-shipment policy changes.

Requirements:
- `FR-P1-21`, `FR-P1-22`, `FR-P1-23`, `FR-P1-24`
- `FR-P5-01`, `FR-P5-02`, `FR-P5-03`, `FR-P5-04`, `FR-P5-05`, `FR-P5-06`

Dependencies `P1-004`, `P1-005`, `P1-006` are satisfied.

Task Catalog acceptance mapping:
- `TC-001`, `TC-008`, `TC-060`, `TC-061`, `TC-062`, `TC-121`

For TC-001/TC-008, prove only the P1-007 slice that is currently executable. Do **not** implement future packing/shipment/ERP/manifest work just to complete a later end-to-end scenario.

## Required product behavior

### 1. SHORT_ALLOCATED — `allowPartialShipment = false`

When real Allocation cannot cover the intended quantity and canonical state becomes `SHORT_ALLOCATED`, persist an explicit shortage case and escalate to `Warehouse Supervisor`.

Supervisor must see the shortage context: CustomerOrder, CustomerOrderLine, OutboundOrder/Line, requested/required quantity, allocated quantity, missing quantity, current `allowPartialShipment`, affected Allocation(s), and reason for escalation.

Two architect outcomes:

**A. Persistent change `allowPartialShipment: false -> true`**
- require a Supervisor reason;
- scope the persistent change to this exact `CustomerOrder` only;
- do not change customer defaults, warehouse defaults, or any other current/future CustomerOrder;
- available quantity stays executable/allocated;
- missing quantity remains `BACKORDERED`/uncovered for later planning;
- process continues without cancelling the available part.

**B. Cancel the shortage OutboundOrder**
- `OutboundOrder -> CANCELLED` through the accepted transition service;
- every affected Allocation in that OutboundOrder is released through the accepted P1-004 path;
- quantity returns to the proper CustomerOrderLine ATPReservation according to the accepted ATP/Allocation contract;
- the line that lost the reservation race becomes/remains `BACKORDERED`;
- other non-problem lines of the same cancelled OutboundOrder return to `OPEN` as Architect requires;
- CustomerOrder returns from `IN_FULFILLMENT` to `ACCEPTED` with `WARNING` where the Architect aggregate rule applies;
- preserve future queue priority/tie ordering; do not silently create a new fulfillment shortcut.

### 2. SHORT_ALLOCATED — `allowPartialShipment = true`

No Supervisor decision is required merely because full quantity is unavailable.

- realize the available allocated quantity;
- keep the missing quantity `BACKORDERED`/uncovered;
- later stock may create a later OutboundOrder through the existing planner;
- do not release or cancel the available portion.

Prove both variants in real PostgreSQL integration tests (`TC-060`).

### 3. Scanner SHORT_PICKED declaration

Add the smallest normal RF action for a Picker to report a genuine shortage after confirming the quantity physically found.

The decisive action must run through the real Scanner UI and real backend.

On short confirmation:
- authoritative picked quantity is the quantity actually found, never client-trusted derived totals;
- current PickTask terminates in canonical `SHORT_PICKED`;
- the source location is blocked using the canonical warehouse/location blocking primitive required by Architect;
- persist missing quantity and a durable shortage/reallocation identity;
- preserve exact TU contents/picked quantity already confirmed;
- `OutboundOrder` must not advance to packing while unresolved shortage recovery remains.

Do not reinterpret a normal partial RF confirmation as SHORT_PICKED unless the operator performs the explicit shortage action required by the UI flow.

### 4. Automatic SHORT_PICKED reallocation — R44 / TC-061

Compute effective `maxAutomaticShortPickReallocations` exactly:
- customer-specific override when it is not `null`;
- otherwise warehouse setting;
- warehouse default is `1` when not configured.

Reuse an existing canonical config field if present. If not present, add the smallest additive configuration needed. Do not duplicate configuration concepts.

The counter identity is for the same `OutboundOrderLine` + same unresolved missing quantity/case, not a global task counter.

If all conditions hold:
- qualified ATP stock exists in a **different, unblocked** location;
- effective retry limit has not been reached;
- current shortage case is unresolved;

then atomically:
- create/adjust the replacement hard reservation using accepted P1-004 primitives;
- create exactly one replacement PickTask for exactly the unresolved quantity using accepted P1-005 task semantics;
- never revive or mutate the old SHORT_PICKED task back to active;
- replacement task may use only eligible stock and must not violate ATP/non-ATP or hard-reservation rules.

If no eligible stock exists or the effective retry limit is reached, stop automatic retries and escalate the unresolved shortage to Warehouse Supervisor.

No retry loop may create duplicate Allocation/PickTask rows or exceed the configured limit under replay/concurrency.

### 5. Supervisor SHORT_PICKED outcomes — `allowPartialShipment = false`

Expose a real Mercato Supervisor shortage/recovery surface. Decisions are server-authoritative, role-gated, audited and idempotent.

#### A. `WAIT / BACKORDERED` — R45 / FR-P5-04

- short-picked OutboundOrderLine -> canonical cancelled/recovery state required by Architect;
- release its remaining Allocation through accepted reservation primitives;
- return the unresolved amount to ATPReservation according to the accepted quantity contract;
- all other lines of the same CustomerOrder currently in `ALLOCATED` or `PICKED` and **not PACKED** are rolled back through the same logical recovery path;
- lines already `PACKED` are untouched;
- short line becomes `BACKORDERED`;
- other rolled-back lines become `OPEN`;
- `CustomerOrder` returns to `ACCEPTED + WARNING` only if no packed execution remains; otherwise it stays `IN_FULFILLMENT + WARNING`;
- if the affected OutboundOrder has no PACKED line left it becomes `CANCELLED`; if PACKED lines remain, it continues only for that packed part.

Where Architect requires physical return of already-picked stock, do **not** implement future P4 PutBackTask lifecycle/RF assignment in this ticket. Persist/use the accepted durable recovery handoff/event needed for later P4 and do not fake an immediate physical inventory move.

#### B. `CANCEL / quantity correction` — R46 / FR-P5-05 / TC-121

A plain OutboundOrderLine cancel button is insufficient.

Require Warehouse Supervisor to edit `CustomerOrderLine.Quantity` as part of the decision.

Supported Architect outcomes:
- set required commercial quantity to zero: cancel/release the execution and record the physical-return handoff for already-picked goods; or
- correct quantity to the quantity actually picked: no PutBack is required for that picked quantity and execution continues in the reduced commercial quantity.

When correcting to actual picked quantity, update the affected `OutboundOrderLine.requiredQty` in the same authoritative operation, preserving the accepted FR-P1-43/R69 requiredQty lifecycle (`TC-121`).

Reject ambiguous or stale conflicting decisions. Do not silently change commercial quantity from an execution-only cancel.

#### C. Persistent `allowPartialShipment = true` — R47 / FR-P5-06

- require a Supervisor reason;
- persist `allowPartialShipment = true` on this exact CustomerOrder only;
- current missing execution part is cancelled/released as required while already picked available quantity proceeds;
- the still-uncovered commercial quantity remains available for later planning/backorder handling; do not silently shrink CustomerOrderLine.Quantity;
- all future shortages for this CustomerOrder follow partial-shipment behavior automatically without another Supervisor approval;
- do not modify customer or warehouse defaults.

Prove all three distinct decision outcomes (`TC-062`).

### 6. R48 packing shortage reuse — backend seam only

Architect requires a confirmed shortage/damage during later `repack by SKU` to use the same SHORT_PICKED recovery mechanism but **without blocking the source picking location**.

P1-010 packing UI is not implemented here.

Provide only the reusable backend/domain entry seam necessary so later packing can invoke the same shortage engine with `blockSourceLocation = false`, and prove the behavioral distinction with focused integration tests.

Do not build Packer screens, repack flow, QC workflow, Shipment, or packing completion in P1-007.

## Persistence / migration

Persist only the smallest authoritative P1-007 data required, for example shortage case identity/state, unresolved quantity, automatic retry count/limit snapshot, escalation/decision outcome, canonical actor, reason where required, timestamps/idempotency/correlation, and any missing canonical config field.

Prefer additive relations/tables/columns. Do not rewrite accepted migrations.

Use accepted FND-002 transition/audit facts rather than inventing a parallel audit subsystem.

Do not redefine Inbound location/TU semantics.

## Concurrency, idempotency and rollback — decisive evidence required

Use real MikroORM/PostgreSQL application paths. No in-memory/fake EntityManager and no raw-SQL-only business implementation proof.

Prove at minimum:

1. Duplicate/replayed `SHORT_PICKED` declaration cannot create two shortage cases or double-change picked quantity.
2. Two independent overlapping real DB transactions trying to auto-reallocate the same unresolved shortage create at most one replacement Allocation/PickTask and increment the retry counter exactly once.
3. Capture strict DB-side lock evidence with distinct real PostgreSQL PIDs and `pg_blocking_pids`/`wait_event_type = Lock` when the implementation uses a blocking serialization path.
4. At limit boundary, parallel workers cannot exceed `maxAutomaticShortPickReallocations`.
5. Duplicate Supervisor decision with the same idempotency identity is exactly-once; conflicting second decision fails closed.
6. Real rollback: application transaction performs writes/flushes, forced failure occurs before commit, independent fresh DB read proves no partial shortage/reallocation/decision state escaped.

## Real UI / Playwright acceptance

### Scanner

Real rendered Scanner, no decisive route mocks:
- login / warehouse / Picking module / zone;
- receive an assigned task;
- bind Picking TU;
- scan source + SKU + actual quantity less than remaining;
- execute explicit SHORT_PICKED action;
- visible short state;
- when fixture provides eligible stock elsewhere and retry limit allows, receive/continue the replacement PickTask for the missing quantity from a different eligible location.

### Mercato Supervisor

Real rendered Mercato, no decisive route mocks:
- open shortage/recovery context;
- see quantities, retry count/effective limit, affected order/line/location and current allowPartialShipment;
- execute real Supervisor decision through UI;
- visible resulting states and persisted DB correlation.

Playwright coverage must decisively cover the auto-retry path and the Supervisor outcomes needed to prove wait/cancel/persistent-partial behavior. Fixtures may prepare deterministic state but must not replace the decisive UI action.

Human-facing automation evidence label is **PLAYWRIGHT VERIFIED**, never HUMAN VERIFIED.

## Tests / regression

Run and persist exact command/pass counts for:
- focused P1-007 real PostgreSQL integration suite;
- P1-007 Scanner Playwright;
- P1-007 Mercato Supervisor Playwright;
- P1-004 Allocation regression;
- P1-005 task assignment regression;
- P1-006 RF/idempotency regression;
- P1-001 CustomerOrder aggregation/policy regression;
- P1-003 requiredQty/planning regression;
- targeted P1-008/TU regression if TU code is touched;
- targeted accepted Inbound Inventory/location/warehouse/TU compatibility regression for every shared primitive touched.

Task Catalog Definition of Done:
- automatic retries stop exactly at the configured limit;
- wait/cancel/partial paths produce Architect-defined states and a valid next UI action.

## Evidence

Push durable P1-007 evidence to `WMS_Outbound` with:
- exact 40-char Mercato and Scanner SHAs + lineage from accepted P1-006 heads;
- exact files/migrations changed;
- shortage state/quantity lifecycle tables for SHORT_ALLOCATED and SHORT_PICKED;
- retry-limit hierarchy and counter identity;
- independent PostgreSQL concurrency output with PIDs/DB wait proof;
- rollback proof;
- all Supervisor outcomes and requiredQty correction proof;
- R48 backend-seam proof without packing scope creep;
- exact test commands/pass counts;
- real Playwright screenshots/trace identifiers and visible order/task/TU/shortage ids;
- regression results and any remaining gaps.

Do not self-declare Human Verified or FINAL PASS.

## Hard boundary / STOP

Do not start or implement:
- P1-009 Direct Pack;
- P1-010 full packing/repack/Packer UI/QC;
- Shipment / Carrier Selection / labels / ERP POST / manifest;
- P4 PutBackTask model, FIFO assignment, RF PutBack execution or physical return workflow;
- P1-008 redesign;
- unrelated legacy cleanup.

Implement P1-007 + decisive evidence only, push, then **STOP**.
