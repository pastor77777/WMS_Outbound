# P2-002 — Crossdock OutboundOrder/Line and CrossDockPickTask planning

**Catalog item:** 21/37  
**Execution class:** product implementation + real PostgreSQL evidence + bounded Scanner assignment proof  
**Authority:** current Architect Source / Canon / exact Task Catalog  
**Owner-selected executor/venue:** executor-neutral; Owner controls executor/session mechanics.

## 1. Frozen accepted baseline

Start Mercato branch `outbound/p2-002` from exact accepted P2-001 SHA:

`8a264fff5c2ca665294d1e02df90c6f37554fe7f`

Accepted P2-001 evidence:

`941d614966b1d2197d8654b3af924afb6ab14d58`

Scanner accepted/frozen baseline entering P2-002:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

If Scanner changes are required for the bounded crossdock-module assignment journey, branch from that exact SHA. Do not absorb P2-003 sorting/scanning.

Do not rebase/squash/transplant accepted history.

## 2. Mandatory grounding

Read current versions of:

1. `WMS_Outbound/STATE.md`;
2. current WMS handover;
3. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact P2-002 section;
4. `01_ARCHITECT_SOURCE/2026-08-31/proces_2_outbound_crossdock.md` — KROK 1 and KROK 2 boundary, especially R3–R8, R29–R30, R39–R40, R43;
5. `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P2-02`, `FR-P2-03`, `FR-P2-04`, `FR-P2-16`, `FR-P2-21`, `FR-P2-22`, `FR-P2-25`, `CON-03`;
6. acceptance scenarios `TC-020`, `TC-021`, `TC-022`, `TC-029`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-036`, `TC-092`, `TC-099`, `TC-101`, `TC-119`, `TC-134`;
7. accepted P2-001 `WmsOutboundCrossDockBinding`/eligibility service and exact persistence/idempotency semantics;
8. accepted P1-003 OutboundOrder/Line grouping and status primitives;
9. accepted P1-005/FND-003 warehouse task ordering, single-active-warehouse-task guard and record-lock primitives;
10. current Outbound TU/TUSetup/Sequence primitives and Scanner module/task-assignment architecture.

Current code/DB is implementation evidence only. If it conflicts with Architect Source, preserve Architect behavior and record the gap.

## 3. Exact objective

Consume authoritative positive P2-001 crossdock binding facts and create the target planning objects for crossdock execution:

- `OutboundOrder` with immutable `fulfillmentChannel = CROSSDOCK`;
- exactly one `OutboundOrderLine` for each planned crossdock demand allocation represented by the accepted P2-001 boundary;
- exactly one `CrossDockPickTask` per crossdock `OutboundOrderLine`, carrying the authoritative source quantity lock;
- operator assignment through the Scanner crossdock module using the accepted warehouse task-ordering and single-active-task rules.

P2-002 owns planning and task assignment only. **Do not implement P2-003 RF sorting, quantity/quality confirmation, first placement, physical target-TU creation, sealing, completion or shortage handling.**

## 4. Planning source — accepted P2-001 truth

Use the accepted P2-001 binding seam as the only pre-planning crossdock reservation truth.

Hard rules:

- plan only positive active binding quantity;
- do not re-evaluate historical Inbound qualification as a substitute for P2-001;
- do not independently reserve the same source quantity in a second ad-hoc structure;
- planning must atomically convert/associate the accepted binding quantity to one planned task without allowing duplicate planning on replay/concurrency;
- `plannedQty` on `CrossDockPickTask` becomes the authoritative P2 quantity lock described by R29/FR-P2-16;
- terminal future `confirmedQty`/`damagedQty` accounting must remain compatible with the P2-001 source-capacity seam (`CONSUMED`/`DAMAGED` stay deducted; only an unexecuted released plan may return capacity).

No `Allocation` is created anywhere in crossdock planning.

## 5. One-shot generation — R29/R30 / FR-P2-16

Generation for a source Inbound TU is one-shot at the `IN_CROSS_DOCK` boundary represented by accepted P2-001 binding facts.

For each binding planned by P2-002:

- create at most one crossdock `OutboundOrderLine`;
- create exactly one `CrossDockPickTask` with that line;
- store exact source Inbound TU, SKU/item, plannedQty, target line and binding correlation;
- prevent replay from creating another task or another line;
- never append a second task later to the same crossdock `OutboundOrderLine`;
- never auto-increase its `requiredQty` because later BACKORDERED demand appeared;
- later demand after source-TU finalization is standard fulfillment territory.

The quantity lock is on `CrossDockPickTask`, not on `OutboundOrderLine`.

## 6. Crossdock OutboundOrder/Line grouping — R3/R5/R43

`OutboundOrder.fulfillmentChannel` must be `CROSSDOCK` at creation and immutable.

Never mix STANDARD and CROSSDOCK lines in one `OutboundOrder`.

Group crossdock lines together only when all required destination facts are compatible:

- same customer;
- same delivery address;
- same `priority`;
- identical `slaDeadline`;
- when `allowPartialShipment = false`, only lines from one `CustomerOrder` may share the order;
- when `allowPartialShipment = true`, compatible lines may span multiple `CustomerOrder` for the same customer.

The crossdock `OutboundOrder` inherits `priority` and `slaDeadline` from its source `CustomerOrder` facts. Do not replace them with a newly computed looser tolerance.

Each crossdock `OutboundOrderLine.requiredQty` equals the quantity planned for that line by this one-shot generation and remains traceable to its `CustomerOrderLine` and P2-001 binding.

Planning must move the covered `CustomerOrderLine` from `BACKORDERED` to the accepted planned state only according to existing authoritative state-transition rules; do not invent new statuses.

## 7. n:n topology and destination continuity — R3/R40

P2-002 must persist enough destination/topology identity for later P2-003 sorting so that compatible tasks may feed a shared outbound destination flow.

However physical target Outbound TU creation is **lazy**:

- do not eagerly create a physical Outbound TU merely because a plan/task exists;
- first real target TU creation/numbering occurs at first placement in P2-003 under R7/R40;
- a task may carry durable target grouping/continuity metadata before the physical TU exists;
- when P2-003 performs first placement, lack of an open target TU must be resolvable atomically by creating one;
- do not pre-implement the placement action, TU scan, TU_NUMBER/SSCC generation or label printing in this task.

Preserve the future rules:

- full 1:1 + valid GS1 SSCC will inherit source TU_NUMBER/SSCC;
- otherwise/n:n will use new TUSetup/Sequence numbering and label;
- those physical-numbering effects are P2-003 execution behavior, not P2-002 planning behavior.

## 8. CrossDockPickTask persistence

Introduce/reuse the minimum additive task model needed for later RF execution.

Retain at least:

- organization/tenant/warehouse;
- source Inbound TU;
- SKU/item;
- source P2-001 binding/idempotency correlation;
- `OutboundOrderLine` target;
- `plannedQty` exact decimal;
- task status with at least the states required now for planning/assignment (`CREATED`, `ASSIGNED`) and compatibility with later P2 states;
- assigned operator/owner and assignment timestamps where existing task infrastructure expects them;
- deterministic task identity / uniqueness constraints sufficient for one task per planned binding/line;
- timestamps/audit correlation.

Do not duplicate existing shared task-lock truth if FND-003 primitives already own operator exclusivity.

Migration must be additive/reversible and must not reinterpret legacy Inbound crossdock tables.

## 9. Assignment — R39 / FR-P2-21

Provide the bounded Scanner crossdock-module assignment journey.

When a Warehouse Operator enters/selects the crossdock module:

- no zone selector is shown/required;
- if the operator has any active warehouse task of any supported warehouse-task type, assignment is rejected/fails closed;
- otherwise assign exactly one next eligible `CrossDockPickTask`;
- ordering follows the same configured warehouse ordering used by accepted PickTask semantics;
- parallel operators cannot receive the same task;
- the assignment transition is server/DB authoritative and idempotent;
- no Supervisor task assignment pool is introduced.

P2-002 Scanner/UI scope ends after the operator receives/observes the assigned task. No source scan, SKU scan, quantity/quality capture, target TU scan/create, completion or exception decision belongs here.

## 10. CON-03 — planning concurrency

Use genuine approved Testing PostgreSQL with independent overlapping transactions.

Prove at minimum:

- the same positive P2-001 binding/source quantity cannot produce two active `CrossDockPickTask` plans;
- two concurrent planner calls for the same source/binding are replay-safe/exactly-once;
- concurrent bindings competing for compatible grouping cannot create duplicate line/task coverage;
- `SUM(plannedQty)` for active planned tasks plus terminal source-use facts never exceeds accepted P2-001 source capacity;
- demand coverage including ATPReservation, all non-CANCELLED OOL requiredQty and active P2 binding/task coverage never exceeds CustomerOrderLine demand;
- independent concurrent operator assignment cannot assign one task to two operators;
- operator with an already-active warehouse task cannot receive a crossdock task.

Capture decisive PostgreSQL-side lock/serialization evidence tied to the actual participants for the material race(s).

## 11. Required real PostgreSQL scenarios

Create a dedicated P2-002 PostgreSQL integration suite covering at minimum:

1. positive P2-001 binding creates CROSSDOCK OutboundOrder/Line + one CrossDockPickTask;
2. no `Allocation` is created;
3. `plannedQty` equals the accepted binding quantity exactly;
4. one binding/line creates exactly one task under replay;
5. concurrent planner calls cannot double-plan one binding/source quantity;
6. CROSSDOCK channel is immutable and not mixed with STANDARD lines;
7. compatible customer/address/priority/slaDeadline grouping succeeds;
8. incompatible customer prevents grouping;
9. incompatible delivery address prevents grouping;
10. incompatible priority prevents grouping;
11. non-identical slaDeadline prevents grouping;
12. `allowPartialShipment = false` prevents multi-CustomerOrder grouping;
13. compatible `allowPartialShipment = true` grouping across CustomerOrders of same customer succeeds;
14. OutboundOrder inherits exact source `priority`/`slaDeadline`;
15. source TU/binding/SKU/CustomerOrderLine correlation persists correctly;
16. task is created without eager physical Outbound TU creation;
17. assignment through crossdock module uses warehouse ordering and no zone input;
18. concurrent operator assignment gives one task to at most one operator;
19. operator with active warehouse task is blocked from crossdock assignment;
20. organization/tenant/warehouse isolation;
21. later BACKORDERED demand does not append a task to an already-generated crossdock line;
22. accepted P2-001 source/demand eligibility remains non-regressive after planning.

Do not split trivial assertions merely to inflate count.

## 12. Mandatory regressions

Run on exact final P2-002 revisions:

- accepted P2-001 PostgreSQL suite;
- accepted P1-003 grouping/planning suite(s);
- accepted P1-005 task-assignment/ordering suite(s) for any shared assignment primitive touched;
- FND-003 shared task-lock/TU/warehouse compatibility suites for touched primitives;
- accepted Inbound cross-dock result regression if shared TU/Inbound boundary primitives are touched;
- relevant Scanner P1 task-assignment/auth/warehouse-context regressions if Scanner changes.

Record exact suite names and counts in evidence.

Do not rerun unrelated dispatch/manifest/final-settlement suites unless the actual diff touches their owning primitives.

## 13. Playwright / user-facing proof

P2-002 has a bounded user-facing Scanner assignment assertion, so Playwright is mandatory if Scanner/module assignment changes are required.

Decisive rendered journey:

1. Warehouse Operator logs in to the canonical Testing Scanner;
2. selects/enters the crossdock module;
3. does **not** select a zone;
4. receives the next eligible CrossDockPickTask according to configured ordering;
5. rendered task identity/source/expected planned quantity is visible;
6. a second operator cannot receive the same task;
7. an operator with another active warehouse task is blocked.

Stop before any P2-003 sorting interaction.

Automation is `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED` unless the Owner explicitly performs/accepts a human walkthrough.

## 14. Evidence

Create/update only:

`05_EVIDENCE/P2-002_EVIDENCE.md`

Record:

- accepted P2-001 base and exact final Mercato SHA;
- final Scanner SHA if changed, otherwise frozen baseline;
- merge bases and compare scope/files for every changed repo;
- migration/schema/unique/idempotency constraints;
- exact P2-001-binding → OutboundOrderLine → CrossDockPickTask mapping;
- exact `plannedQty` and quantity-lock semantics;
- grouping/inheritance proof for customer/address/priority/slaDeadline/allowPartialShipment;
- proof that no `Allocation` exists;
- proof that physical target TU creation remains lazy/not implemented;
- dedicated real PostgreSQL P2-002 suite command/result;
- literal CON-03 and operator-assignment serialization evidence;
- exact regression suite names/counts;
- exact Playwright runtime/revisions and result if UI changed;
- clean final worktree identities;
- explicit P2-003+ exclusions;
- no secrets;
- no `FINAL PASS`, Supervisor acceptance or Owner Acceptance claim.

## 15. Repository write boundary

Authorized product writes:

- Mercato branch `outbound/p2-002` from exact accepted P2-001 SHA `8a264fff5c2ca665294d1e02df90c6f37554fe7f`;
- additive planner/task/schema/API/tests needed by P2-002;
- Scanner branch from `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` only if required for crossdock-module assignment visibility, and only for that bounded assignment scope.

Authorized evidence write:

- `WMS_Outbound/05_EVIDENCE/P2-002_EVIDENCE.md`.

Do not change:

- Inbound business semantics/qualification/deintegration/transport behavior;
- `STATE.md` or handovers during execution;
- Task Catalog / Canon / Architect Source / traceability;
- Devaxonic-WMS `AGENTS.md`, credentials or unrelated `.ai/*` steering;
- Prod/Demo.

## 16. STOP boundary

After complete implementation and verification:

1. push final Mercato P2-002 commit(s);
2. if Scanner changed, push the bounded P2-002 Scanner commit(s);
3. push `05_EVIDENCE/P2-002_EVIDENCE.md`;
4. report exact final Mercato SHA, final/frozen Scanner SHA, WMS evidence SHA, P2-002 PostgreSQL count, concurrency/assignment result, regression counts and Playwright count/status if applicable;
5. STOP.

Do not start P2-003 and do not claim acceptance.