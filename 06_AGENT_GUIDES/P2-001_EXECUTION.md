# P2-001 — Inbound crossdock boundary and demand/source eligibility

**Catalog item:** 20/37  
**Execution class:** product implementation + real PostgreSQL evidence + accepted Inbound regression  
**Authority:** current Architect Source / Canon / exact Task Catalog  
**Owner-selected executor/venue:** executor-neutral; Owner controls executor/session mechanics.

## 1. Frozen accepted baseline

Start Mercato branch `outbound/p2-001` from exact accepted P1-016 SHA:

`dd5ff1493740ffc99e11ce40e0b5ffc6b646f574`

Accepted P1-016 evidence:

`50c5664fda12caf5c2f7bcdb0e9e86c3495a01c2`

Scanner remains frozen:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Do not rebase/squash/transplant accepted P1 history.

## 2. Mandatory grounding

Read current versions of:

1. `WMS_Outbound/STATE.md`;
2. current WMS handover;
3. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact P2-001 section;
4. `01_ARCHITECT_SOURCE/2026-08-31/proces_2_outbound_crossdock.md` — especially KROK 1, R1–R6, R29–R30, R41–R43;
5. `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P2-01`, `FR-P2-03`, `FR-P2-23`, `FR-P2-24`, `FR-P2-25`, `INT-01`, `CON-03`;
6. acceptance scenarios `TC-020`, `TC-029`, `TC-036`, `TC-092`, `TC-102`, `TC-103`, `TC-119`, `TC-134`;
7. accepted P1-002/P1-003 demand/planning semantics and current target CustomerOrderLine/OutboundOrderLine persistence;
8. accepted Inbound TU lifecycle and the current boundary that exposes an `ELEMENTARY` TU at `IN_CROSS_DOCK`;
9. current shared TU/Inventory/warehouse primitives and any existing `wms_cross_dock_demands` or equivalent legacy/current-state structure.

Current code/DB is implementation evidence only. If it conflicts with Architect Source, preserve Architect behavior and record the gap.

## 3. Exact objective

Implement the Outbound-side boundary that consumes an already-qualified Inbound `TU` `ELEMENTARY` at `IN_CROSS_DOCK`, calculates current source/demand eligibility, and records a concurrency-safe demand/source binding basis without double coverage.

P2-001 establishes eligibility/binding truth only. **Do not implement P2-002 CrossDockPickTask/OutboundOrder/OutboundOrderLine planning in this task.**

## 4. Inbound boundary — INT-01 / R41 / R42

Outbound receives from the accepted Inbound boundary:

- source Inbound `TU` id;
- `SKU`;
- ASN-declared quantity — explicitly not a physical-count confirmation;
- receipt correlation;
- current Inbound TU type/status facts sufficient to prove `ELEMENTARY` + `IN_CROSS_DOCK`.

Hard guards:

- only `ELEMENTARY` may enter Outbound crossdock;
- `AGGREGATE` must be rejected/fail closed;
- do not alter Inbound qualification/deintegration/transport semantics;
- Inbound's earlier qualification is a transport premise only, not a binding demand assignment;
- binding demand eligibility is recalculated when the TU reaches `IN_CROSS_DOCK`.

Do not make Outbound responsible for moving an aggregate TU into crossdock or decomposing it.

## 5. Demand eligibility — R1/R2/R6

Eligible demand is current target-domain `CustomerOrderLine` demand in `BACKORDERED`, ordered according to the accepted warehouse priority queue.

For each candidate line/SKU calculate with exact decimal arithmetic:

`demandEligibleQty = CustomerOrderLine.Quantity - ATPReservation - Σ(requiredQty of all non-CANCELLED OutboundOrderLine for that CustomerOrderLine, regardless of fulfillment channel)`

Rules:

- never use floating-point business arithmetic;
- clamp/reject inconsistent negative computed availability rather than assigning negative quantity;
- both soft coverage (`ATPReservation`) and hard/planned coverage (`requiredQty`) are deducted;
- already-created STANDARD and future CROSSDOCK line coverage participate equally in the `requiredQty` sum once present;
- cancelled OOL coverage does not reduce demand eligibility;
- only current `BACKORDERED` demand is eligible for new P2-001 matching.

## 6. Source eligibility — R6

For each source Inbound TU/SKU calculate:

`sourceEligibleQty = ASN declared qty - Σ(plannedQty of active CrossDockPickTask) - Σ(confirmedQty of completed CrossDockPickTask) - Σ(damagedQty of completed CrossDockPickTask)`

P2-001 must support the formula even though P2-002+ creates future tasks. Use persisted eligibility/assignment facts or an explicit seam compatible with future task rows; do not invent duplicate quantity truth.

Rules:

- completed damaged quantity never returns to eligible source quantity;
- confirmed quantity never returns to eligible source quantity;
- active planned quantity reserves source eligibility;
- source eligibility belongs to exact source TU + SKU + organization/tenant/warehouse context;
- no local/in-memory-only reservation may be the authoritative concurrency guard.

## 7. Crossdock eligibility

`crossDockEligibleQty = min(sourceEligibleQty, demandEligibleQty)`

Only positive quantity may be bound/returned as eligible.

For zero current demand match:

- create no P2-002 work/task/order/line;
- persist/return a deterministic zero-match outcome sufficient for audit/correlation;
- the full ASN-declared quantity is residual for the Inbound settlement handoff;
- do not directly implement Inbound Putaway movement/state transitions; preserve the accepted Inbound handoff boundary.

## 8. 1:1 destination eligibility facts — R2 / R43

P2-001 may calculate/record the facts needed for future P2-002 topology decisions, but must not create the future order/task objects.

Full 1:1 matching is allowed only when the entire declared source content can be associated to compatible demand with:

- same customer;
- same delivery address;
- compatible `priority` as defined by current Architect rule;
- identical `slaDeadline`;
- with `allowPartialShipment = false`, demand from one CustomerOrder only;
- with `allowPartialShipment = true`, compatible demand may span multiple CustomerOrders for the same customer.

Any future crossdock OutboundOrder must inherit source CustomerOrder `priority`/`slaDeadline`; do not implement that order in P2-001.

## 9. Persistence and idempotency

Persist only the minimum additive/reversible facts needed to make the boundary authoritative and future P2-002 planning safe.

The model must retain at least:

- source Inbound TU correlation;
- source SKU and ASN-declared quantity/correlation;
- organization/tenant/warehouse identity;
- evaluated demand line identity where a positive binding/reservation fact is created;
- eligible/bound quantity and deterministic idempotency identity;
- audit timestamps/status sufficient to distinguish active vs released/consumed future assignment facts if such a reservation seam is introduced.

If existing `wms_cross_dock_demands` or another current structure is reused, record the exact mapping and prove it does not redefine accepted Inbound semantics.

No destructive migration and no legacy-data reinterpretation.

## 10. CON-03 — real PostgreSQL concurrency

Mock concurrency is insufficient.

Create a real PostgreSQL race using independent sessions/transactions where multiple eligible `BACKORDERED` lines compete for the same source TU/SKU quantity.

Prove:

- exact source quantity is assigned/bound at most once;
- warehouse priority ordering determines the winner/allocation sequence where demand exceeds supply;
- no lost update and no double coverage;
- final summed active/bound quantity never exceeds `sourceEligibleQty`;
- demand coverage never exceeds each line's `demandEligibleQty`;
- exact source and demand rows/facts are locked/serialized at the authoritative server/DB boundary;
- replay of the same inbound boundary/correlation is idempotent.

Capture decisive PostgreSQL-side lock/serialization evidence tied to the actual participants where the implementation uses blocking serialization.

## 11. Required PostgreSQL test scenarios

Create a dedicated real PostgreSQL P2-001 integration suite covering at minimum:

1. ELEMENTARY + IN_CROSS_DOCK accepted;
2. AGGREGATE rejected;
3. wrong source status rejected;
4. INT-01 source id/SKU/ASN qty/receipt correlation persisted correctly;
5. ASN quantity is used as declared source basis, not invented physical confirmation;
6. `demandEligibleQty` deducts ATPReservation;
7. `demandEligibleQty` deducts non-CANCELLED STANDARD OOL requiredQty;
8. `demandEligibleQty` deducts non-CANCELLED CROSSDOCK-compatible/future-channel coverage when present;
9. CANCELLED OOL requiredQty does not reduce eligible demand;
10. `sourceEligibleQty` deducts active plannedQty;
11. `sourceEligibleQty` deducts completed confirmedQty;
12. `sourceEligibleQty` deducts completed damagedQty;
13. crossDockEligibleQty is exact min(source,demand);
14. zero demand returns zero/no work and full residual handoff fact;
15. demand disappearing between Inbound qualification and IN_CROSS_DOCK yields zero-match behavior;
16. warehouse priority ordering is respected;
17. concurrent two-line competition for one source quantity cannot double assign (real PostgreSQL);
18. duplicate/replayed boundary message/correlation is idempotent;
19. organization/tenant isolation;
20. existing P1 ATP/requiredQty planning semantics remain non-regressive.

Do not split trivial assertions merely to inflate count.

## 12. Mandatory regressions

Run on exact final P2-001 Mercato SHA:

- accepted P1-002 ATP/soft reservation suite(s);
- accepted P1-003 planning/grouping suite(s);
- FND-003/shared TU/Inventory/warehouse compatibility suites for every shared primitive touched;
- accepted Inbound tests covering TU type/aggregation/deintegration and the `IN_CROSS_DOCK` handoff boundary;
- any existing current crossdock-demand regression affected by the migration/service seam.

Record exact suite names and counts in evidence.

No need to rerun unrelated P1-011..P1-016 suites unless the actual diff touches their owning primitives.

## 13. UI / Scanner boundary

P2-001 has Supervisor visibility only if needed to inspect the new eligibility facts.

- no Scanner product change;
- no CrossDock RF module/task assignment — P2-002/P2-003;
- no mandatory Playwright journey unless P2-001 introduces/changes user-facing Mercato behavior; if UI is changed, prove that exact UI change through rendered Playwright.

## 14. Evidence

Create/update only:

`05_EVIDENCE/P2-001_EVIDENCE.md`

Record:

- accepted P1-016 base and exact final Mercato SHA;
- merge base and compare scope/files;
- migration/schema/unique constraints/idempotency identity;
- exact INT-01 mapping;
- exact formulas and representative arithmetic for source/demand/crossdock eligibility;
- real PostgreSQL P2-001 suite command/result;
- real CON-03 race/serialization evidence;
- exact accepted Inbound regression suites/counts;
- P1-002/P1-003 and shared-primitives regression names/counts;
- frozen Scanner SHA;
- clean final worktree identities;
- explicit scope exclusions;
- no secrets;
- no `FINAL PASS`, Supervisor acceptance or Owner Acceptance claim.

## 15. Repository write boundary

Authorized product writes:

- Mercato branch `outbound/p2-001` from exact accepted P1-016 SHA;
- additive schema/service/tests and minimal Supervisor visibility strictly required by P2-001.

Authorized evidence write:

- `WMS_Outbound/05_EVIDENCE/P2-001_EVIDENCE.md`.

Do not change:

- Scanner;
- Inbound business semantics/qualification rules;
- `STATE.md` or handovers during execution;
- Task Catalog / Canon / Architect Source / traceability;
- Devaxonic-WMS `AGENTS.md` or unrelated `.ai/*` steering;
- Prod/Demo.

## 16. STOP boundary

After complete implementation and verification:

1. push final Mercato P2-001 commit(s);
2. push `05_EVIDENCE/P2-001_EVIDENCE.md`;
3. report exact final Mercato SHA, WMS evidence SHA, P2-001 PostgreSQL count, concurrency result, regression counts and frozen Scanner SHA;
4. STOP.

Do not start P2-002 and do not claim acceptance.