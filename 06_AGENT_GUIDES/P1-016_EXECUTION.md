# P1-016 — Final line/order/inventory settlement and cancellation boundaries

**Catalog item:** 19/37  
**Execution class:** product implementation + real PostgreSQL evidence + rendered Playwright evidence  
**Authority:** current Architect Source / Canon / traceability / exact Task Catalog  
**Owner-selected executor/venue:** executor-neutral; the Owner decides the executor and launch mechanism.

Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

## 1. Frozen accepted baseline

Before any write, sync and verify exact current refs.

Accepted P1-015 baseline:

- Mercato branch base: `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`
- accepted P1-015 evidence: `f201bd0beb2411b4f87f28ca6562a4fc11e6a249`
- Scanner frozen: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` on `outbound/p1-009`
- accepted P1-015 PostgreSQL: `21/21`
- accepted integrated regression aggregate: `171/171`
- accepted P1-015 fresh Playwright: `6/6` against exact served runtime `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

P1-016 implementation branch:

`outbound/p1-016`

It must start from exact merge base:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

Do not rebase/squash/transplant accepted P1-015 history. If the baseline has drifted unexpectedly, STOP and report actual refs.

## 2. Mandatory grounding before implementation

Read current versions of:

1. `Devaxonic-WMS/AGENTS.md`
2. `Devaxonic-WMS/.ai/STATE.md`
3. current Devaxonic Outbound handover
4. `Devaxonic-WMS/.ai/TESTING.md`
5. `Devaxonic-WMS/.ai/OPERATIONS.md`
6. `WMS_Outbound/STATE.md`
7. current WMS Outbound handover
8. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
9. `WMS_Outbound/07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact P1-016 section
10. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — especially R37–R38, R41–R42, R49–R50, R70–R72 and continuous F1
11. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — exact P1-016 requirements
12. state model / transition canon and relevant acceptance scenarios
13. accepted P1-015 evidence and current P1-015 final code
14. current shared Inventory ledger/balance/reservation primitives and accepted Outbound Allocation semantics

Do not reconstruct mutable state from old chat.

## 3. Exact P1-016 scope

### Objective

Settle final Outbound quantities from each `CarrierManifest CONFIRMED` boundary, terminalize lines/orders only when all required contributing Shipment/manifest coverage is confirmed, continuously aggregate customer state, and establish the formal cancellation boundary/routing.

### Requirements

Implement and prove:

- `FR-P1-19`
- `FR-P1-21`
- `FR-P1-25`
- `FR-P1-27`
- `FR-P1-44`
- `FR-P1-45`
- `FR-P1-46`
- `FR-P5-09`
- `INT-06`
- `CON-05`

Acceptance mapping:

- `TC-001`
- `TC-003`
- `TC-008`
- `TC-009`
- `TC-065`
- `TC-128`
- `TC-129`
- `TC-130`
- `TC-131`
- `TC-132`
- `TC-133`

## 4. Accepted P1-015 seam — preserve it

Current accepted P1-015 confirmation already provides:

- `CarrierManifest HANDED_OVER → CONFIRMED`;
- one durable `CarrierManifestConfirmed` transition under duplicate/parallel calls;
- deterministic confirmation snapshot containing immutable manifest membership context;
- physical handover states already applied before confirmation:
  - Shipment `IN_MANIFEST → HANDED_TO_CARRIER`;
  - TU `IN_SHIPMENT → DISPATCHED`;
  - OutboundOrder `READY_FOR_DISPATCH → DISPATCHED`.

P1-016 must attach settlement to this accepted final confirmation boundary without regressing P1-015 lock ordering, idempotency, RBAC, membership or snapshot behavior.

A successful confirmation/settlement operation must not leave a durable `CONFIRMED` manifest with only a partially applied P1-016 settlement.

## 5. Settlement persistence — mandatory

Introduce additive, reversible persistence for exactly-once settlement facts.

Minimum durable identity must prevent duplicate settlement for the same business quantity. Use a normalized unique key at least equivalent to:

- manifest;
- OutboundOrderLine;
- Allocation where applicable;
- settled business quantity/provenance component where one line may be represented by multiple TU contributions.

The exact schema may be improved after current-code inspection, but it must satisfy all of these:

1. one confirmed manifest contribution cannot settle the same line/allocation quantity twice;
2. two different manifests may legitimately settle different portions of the same line/allocation;
3. retries and parallel calls remain replay-safe;
4. settlement facts retain correlation to manifest, Shipment/TU provenance and affected line/allocation;
5. migration/backout is explicit and does not destroy accepted legacy data.

Evidence must record literal final DDL/index/unique-key facts.

## 6. Inventory settlement — R72 / FR-P1-46

For every newly settled quantity represented by outbound TUs in the confirmed manifest:

- settle only that exact quantity;
- perform the accepted semantic `Inventory PICKED → SHIPPED` effect using existing shared Inventory truth/ledger conventions;
- produce append-only Inventory movement/audit evidence with deterministic idempotency identity;
- never decrement or ship quantity from a TU/Shipment not covered by this confirmed manifest;
- never duplicate the movement on replay/concurrency;
- preserve organization/tenant/warehouse/item/location/lot/serial provenance required by the existing Inventory model.

If current Inventory representation does not store a literal status column `PICKED/SHIPPED`, implement the Architect semantic using the existing accepted ledger/balance mechanism rather than inventing a parallel stock truth. Evidence must explain the exact mapping.

Any touched shared Inventory primitive requires targeted accepted Inbound/shared regression.

## 7. Allocation arithmetic — R71/R72 / FR-P1-45/46

`Allocation.reservedQty` is authoritative blocked quantity.

For a partial confirmed-manifest settlement:

`new reservedQty = old reservedQty - exactly newly settled quantity`

Rules:

- decimal arithmetic must be exact; do not use floating-point business arithmetic;
- never allow negative `reservedQty`;
- replay must not decrement twice;
- different manifests settling one Allocation concurrently must not lose an update;
- while line coverage is incomplete: Allocation remains `CONFIRMED` with remaining `reservedQty > 0`;
- when all contributing outbound TU quantity for the line is confirmed and `reservedQty = 0`: `Allocation CONFIRMED → CONSUMED` exactly once;
- `CONSUMED` contributes zero reserved quantity.

## 8. OutboundOrderLine settlement — R72

For each line:

- advance `shippedQty` by exactly the newly settled confirmed-manifest contribution;
- do not double increment on replay/concurrency;
- while any outbound TU contributing quantity to that line belongs to a Shipment whose manifest is not `CONFIRMED`, keep line `PACKED`;
- only when all contributing outbound TUs are covered by confirmed manifests may the line transition `PACKED → SHIPPED`;
- emit `OutboundOrderLineShipped` exactly once at terminalization;
- line terminalization and Allocation `CONFIRMED → CONSUMED` must agree transactionally.

The order of manifest confirmations must not change the final result.

## 9. OutboundOrder completion — R70 / FR-P1-44

An `OutboundOrder` may contribute TUs to more than one Shipment.

Rules:

- after physical handover it is already `DISPATCHED` from P1-015;
- confirmation of only some relevant Shipments must leave it `DISPATCHED`;
- transition `DISPATCHED → COMPLETED` only when every Shipment containing any outbound TU from that order belongs to a `CONFIRMED` manifest;
- emit `OutboundOrderCompleted` exactly once;
- confirmation order is irrelevant;
- a replay must not create another completion effect.

## 10. CustomerOrderLine and CustomerOrder aggregation

Recompute from persisted target-domain truth, not from legacy Sales Order shortcuts.

### CustomerOrderLine

Based on total actually issued/settled quantity across non-cancelled target OutboundOrderLines:

- partial issued coverage -> `PARTIALLY_FULFILLED`;
- full required coverage -> `FULFILLED`;
- transitions/events must be non-regressive and exactly once.

### CustomerOrder continuous F1

Preserve Architect continuous aggregation:

- one issued/fulfilled portion while other active lines remain -> `PARTIALLY_SHIPPED` where defined by F1;
- all required active lines issued -> `SHIPPED`;
- `SHIPPED → CLOSED` only when every OutboundOrder that contributed at least one issued line has reached `COMPLETED` (R41);
- do not close CustomerOrder merely because one manifest or one Shipment is confirmed;
- do not regress accepted BACKORDERED/WARNING behavior.

## 11. Transactionality and lock discipline

P1-016 is a multi-entity settlement transaction.

Use deterministic row-lock ordering for all shared mutable settlement state. At minimum reason explicitly about locking of:

1. CarrierManifest / confirmation boundary;
2. settlement identity rows or equivalent uniqueness guard;
3. relevant Shipments/TUs used as immutable provenance;
4. OutboundOrderLines;
5. Allocations;
6. shared Inventory rows/reservations/balances as required by the existing model;
7. OutboundOrders;
8. CustomerOrderLines / CustomerOrders.

Avoid lock cycles with accepted P1-015 lock order.

A deterministic injected failure after some in-transaction settlement mutations but before completion must roll back all business effects.

## 12. Required real PostgreSQL concurrency proofs

Mock concurrency is insufficient.

### A. Duplicate/parallel settlement of the same manifest

Use two real service calls on independent PostgreSQL sessions against the same eligible manifest confirmation/settlement operation.

Evidence must capture exact backend identities before blocking:

- `pidA`
- `pidB`

Use an independent observer to query `pg_stat_activity` / `pg_blocking_pids` for the exact blocked PID.

Prove:

- `pidA != pidB`;
- one real session blocks on the other where serialization is expected;
- exact `wait_event_type = Lock` or equivalent decisive PostgreSQL lock evidence;
- one settlement fact per unique business contribution;
- one Inventory movement/business effect;
- one `reservedQty` decrement;
- one shippedQty increment;
- at most one terminal transition per entity;
- loser/replay returns safely without regression.

### B. Two different manifests settle the same line/allocation concurrently

Construct one OutboundOrderLine / Allocation whose outbound TU quantity is split across two Shipments in two different manifests.

Run two real confirmation/settlement operations concurrently on independent PostgreSQL sessions.

Capture exact `pidA` and `pidB` and decisive lock proof tied to those actual operations.

Prove no lost update:

- final `reservedQty = initial reservedQty - settlementA - settlementB`;
- final `shippedQty = prior shippedQty + settlementA + settlementB`;
- no negative quantity;
- each manifest contribution exists exactly once;
- line/Allocation terminalize only after complete coverage;
- final result is identical regardless of which manifest wins first.

## 13. Required rollback proof

Use a real transaction and deterministic injected failure after at least one meaningful settlement mutation has been staged/flushed but before final completion.

After rollback, inspect from an independent/fresh EntityManager/session and prove no partial durable business state, including as applicable:

- manifest confirmation/settlement boundary;
- settlement rows;
- Inventory movement/balance effect;
- Allocation `reservedQty`/status;
- OutboundOrderLine `shippedQty`/status;
- OutboundOrder status;
- CustomerOrderLine status;
- CustomerOrder status;
- state-transition events.

The exact expected manifest state depends on where settlement is transactionally attached, but there must never be a durable success-shaped `CONFIRMED` state with missing required settlement.

## 14. Cancellation boundary and routing

P1-016 owns the final cancellation boundary/orchestration needed by its Task Catalog. It does **not** own future P3/P4 physical recovery implementation.

Implement one canonical server-authoritative cancellation request path for CustomerOrder/CustomerOrderLine correlation.

### Correlation / INT-06

Incoming OMS/ERP or Supervisor request must resolve the exact target customer line and current target-domain execution facts.

Route based on formal confirmed `pickedQty` / current formal line state:

- before physical pick -> P3 route classification;
- after physical pick / packed physical stock -> P4 route classification.

Persist correlation/audit sufficient for downstream P3/P4 tasks.

Do not create P3 Reservation Release timers, P4 PutBackTask execution, Scanner putback UI or other future recovery workflow in this task.

### Hard cancellation blocks

Preserve and enforce:

- Shipment `POSTING_PENDING` or later: cancellation in WMS is forbidden (R38 / FR-P1-19);
- `CarrierManifest CLOSED` or later: general cancellation is forbidden (R50 / FR-P5-09);
- correction after the post/dispatch boundary is Return Receipt territory and remains out of scope.

### Whole-order atomicity

A request to cancel an entire CustomerOrder must inspect every affected line first.

If one line is blocked:

- reject the whole-order cancellation;
- return/persist an actionable blocked reason identifying the blocking line/boundary;
- do not partially cancel other lines as a side effect.

Mercato Supervisor-facing UI must expose the final status/routing/blocked reason required by the Task Catalog.

## 15. Legacy Sales Order shortcut

Inspect the current legacy mechanism that can mark legacy SO `SHIPPED` from legacy `outbound_delivery.SHIPPED`.

P1-016 must supersede/neutralize that shortcut for the target Outbound flow so target `CustomerOrder`/`OutboundOrder` truth is driven only by target settlement aggregation.

Requirements:

- additive/reversible migration or guarded compatibility mechanism;
- no destructive deletion of legacy data;
- no fake synchronization that lets legacy rows become authoritative target truth;
- literal evidence of the exact legacy trigger/function/config discovered and the exact final behavior after P1-016.

## 16. Minimum PostgreSQL test inventory

Create a dedicated P1-016 real PostgreSQL suite with **at least 22 meaningful tests**. More is acceptable; do not split trivial assertions to inflate count.

Minimum scenarios:

1. one confirmed manifest fully settles one line/allocation/inventory quantity;
2. settlement quantity comes only from outbound TUs in that manifest;
3. settlement fact unique/idempotent;
4. replay same manifest creates no second Inventory effect;
5. partial first manifest decrements reservedQty exactly and leaves Allocation CONFIRMED;
6. partial first manifest increments shippedQty exactly and leaves line PACKED;
7. later second manifest completes line -> SHIPPED;
8. later second manifest consumes Allocation -> CONSUMED with reservedQty 0;
9. reverse confirmation order produces identical final result;
10. one OutboundOrder across two Shipments remains DISPATCHED after first confirmation;
11. same order becomes COMPLETED only after all relevant Shipments confirmed;
12. CustomerOrderLine partial aggregation;
13. CustomerOrderLine full aggregation;
14. CustomerOrder PARTIALLY_SHIPPED aggregation;
15. CustomerOrder SHIPPED then CLOSED only after contributing orders complete;
16. duplicate/parallel same-manifest real-session exact-PID proof;
17. different-manifest same-allocation concurrent exact-PID/no-lost-update proof;
18. deterministic rollback leaves no partial settlement;
19. whole-order cancellation rejected atomically when one line blocks;
20. cancellation blocked at Shipment POSTING_PENDING-or-later;
21. cancellation blocked at CarrierManifest CLOSED-or-later;
22. INT-06 correlation chooses correct P3-vs-P4 route from formal pickedQty;
23. tenant/org isolation if not already included in the above;
24. legacy shortcut cannot prematurely mark target-domain completion.

## 17. Mandatory regressions on exact final P1-016 head

At minimum rerun on exact final implementation SHA:

- P1-015 PostgreSQL: **21/21**;
- P1-014 PostgreSQL: **18/18**;
- P1-013 PostgreSQL: **15/15**;
- P1-012 PostgreSQL: **14/14**;
- P1-011 PostgreSQL: **18/18**;
- FND-002 state transitions: **77/77**;
- FND-002 transaction simulation: **8/8**.

Additionally run the accepted relevant Allocation and shared Inventory/FND-003 regression suites for every shared primitive touched by P1-016. Record exact test names and counts rather than claiming a generic pass.

Any regression failure is a STOP until corrected within the authorized first/corrective-attempt policy.

## 18. Rendered Playwright acceptance

Serve canonical Testing from the exact final P1-016 Mercato SHA before the final browser run.

Use existing accepted UI where possible; add the minimum bounded UI needed for P1-016 Supervisor visibility.

Minimum **6 real rendered journeys**:

A. final confirmation of a one-manifest order visibly results in final settlement/statuses;

B. first of two manifests confirms a partial line/order and visibly leaves it nonterminal;

C. second/later manifest confirms remaining quantity and line/order/customer reaches correct terminal state;

D. attempted cancellation at the post/manifest hard boundary is rejected with actionable blocked reason;

E. whole-order cancellation with one blocking line fails as one unit with no partial cancellation side effects;

F. duplicate/repeated final confirm is UI-stable and persisted business effects remain exactly once.

Playwright must perform the accepted user action through the real application UI. API/DB may prepare deterministic fixtures and verify persisted results, but may not substitute for the decisive UI action.

Evidence must contain the literal final command/result and exact served 40-character Mercato SHA.

## 19. Evidence file

Create/update only P1-016 evidence in WMS_Outbound:

`05_EVIDENCE/P1-016_EVIDENCE.md`

Required evidence:

- exact accepted base and final Mercato SHA;
- exact merge base and Git compare stats/files;
- final migration/DDL/unique settlement identity;
- exact discovered legacy shortcut and final neutralization behavior;
- real Testing PostgreSQL provenance/version;
- literal P1-016 test output/count;
- literal exact-PID concurrency output for both required races;
- rollback fresh-read proof;
- exact settlement arithmetic for partial and complete examples;
- mandatory regression commands/counts;
- shared Inventory/Allocation regression names/counts;
- exact served final Mercato SHA;
- literal final Playwright output and journey count;
- frozen Scanner SHA;
- final worktree identities;
- explicit scope exclusions;
- no secrets;
- no `FINAL PASS`, Supervisor acceptance or Owner Acceptance claim.

## 20. Hard scope exclusions

Do not implement or modify unless strictly required by the exact P1-016 target behavior:

- P2 Crossdock business implementation;
- P3 Reservation Release workflow/timer;
- P4 PutBackTask/RF physical recovery workflow;
- Return Receipt/post-dispatch correction;
- external carrier API/tracking/webhooks;
- Scanner product behavior;
- Prod/Demo;
- unrelated Inbound behavior;
- steering/canon/catalog/traceability documents.

Future P2 must be able to reuse the canonical settlement pipeline, but do not implement P2 in P1-016.

## 21. Repository write boundary

Authorized product writes:

- Mercato on `outbound/p1-016`, starting exactly from accepted P1-015 SHA;
- schema/migration/services/API/UI/tests strictly needed for P1-016.

Authorized evidence write:

- `WMS_Outbound/05_EVIDENCE/P1-016_EVIDENCE.md`.

Do not change:

- `WMS_Outbound/STATE.md`;
- handovers;
- Task Catalog / implementation plan;
- Canon / Architect Source / traceability;
- Devaxonic-WMS `.ai/*`;
- Scanner.

## 22. STOP boundary

After implementation and verification:

1. push final Mercato P1-016 commit(s);
2. ensure final Testing runtime is the exact final Mercato SHA;
3. complete all required PostgreSQL/regression/Playwright evidence;
4. push `05_EVIDENCE/P1-016_EVIDENCE.md` to WMS_Outbound;
5. report only exact final Mercato SHA, exact evidence SHA, test counts, Playwright count and frozen Scanner SHA;
6. STOP.

Do not start P2-001. Do not update steering. Do not claim acceptance.