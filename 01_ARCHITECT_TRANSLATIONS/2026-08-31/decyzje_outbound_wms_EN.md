# Design decisions — Outbound WMS processes

## 1. Purpose and status of the document

This document summarizes decisions made while preparing the Outbound (Order Fulfillment) processes. It is authoritative over earlier, conflicting entries in `../Archiwum/zlecenie_procesy_outbound_wms.md` (withdrawn 2026-07-18).

**Product status:** discovery.

At this stage, the decisions serve to prepare the first analytical document. They are not yet canonical process documentation or the final state model.

## 2. Documentation rules and sources of truth

1. The guidelines from `AGENTS.md` for Inbound are applied analogously to Outbound.
2. A separate `model_stanow_outbound.md` will be created for Outbound.
3. Object, attribute and status names are written in English. Descriptions and rationales — in Polish.
4. The first result is to be one analytical `.md` document containing a process diagram, state machines and Mermaid sequence diagrams.
5. Only after the analytical document is approved will the content be split into target files compliant with `AGENTS.md` and `SZABLON_PROCESU.md`.
6. A missing fact or unresolved conflict must be marked as an open issue. It must not be filled in by assumption.
7. `PickWave` is entirely out of scope for version 1. Its process or state machine must not be designed.
8. Package cross-docking is included in version 1 as a separate process.
9. **DEC-A09 — design boundaries:** technical architecture, services, integration technologies, route planning, invoicing and customer returns remain outside the scope of Outbound process design.
10. **DEC-A10 — single decision source:** the only canonical register of Outbound product decisions is `decyzje_outbound_wms.md`. Prompts, handover, memory and analytical documents do not establish or override decisions.
11. **DEC-A11 — HISTORICAL, superseded by the current-process-prose rule:** this entry preserves the former “decision log before process” hierarchy. The current source of behavior is now the process prose in `proces_1_standard_fulfillment.md`–`proces_4_physical_putback.md`, while this file preserves history and rationale. The result of every decision must be propagated into the process.
12. **DEC-A12 — temporary prompts:** after a task is completed, relevant prompt content must be classified as decisions, open questions or working rules and moved to the appropriate files. The prompt must then be removed or archived. A prompt is not a source of truth.
13. **DEC-A13 — `PickWave`:** version 1 does not design a process, states or a technical extension point for a future `PickWave`.
14. **DEC-A14 — archiving the analytical document:** `propozycja_procesow_outbound.md` ceased to be a project source and was moved to `../Archiwum/` (2026-08-22, `BACKLOG.md` B16). The source of truth for current behavior is `proces_1_standard_fulfillment.md`–`proces_4_physical_putback.md`, the state canon is `model_stanow_outbound.md`, and the cross-cutting responsibility view is `macierz_odpowiedzialnosci_outbound.md`. No active artifact may require opening the archived document in order to understand current behavior.
15. **DEC-A15 — no separate file for PROCESS 5:** cross-cutting exceptions do not have their own `proces_5` file. Their description lives in the “Exceptions and alternative paths” sections of `proces_1_standard_fulfillment.md` and `proces_2_outbound_crossdock.md` (2026-08-22). Reason: PROCESS 5 has no numbered steps of its own, so a separate file would be an empty pseudo-process.

## 3. Separation of customer order from warehouse execution

### 3.1 Object responsibilities

- `CustomerOrder` represents the customer's commercial demand.
- `CustomerOrderLine` represents the ordered SKU and quantity and has its own state lifecycle.
- `OutboundOrder` represents a warehouse execution scope created by grouping or splitting order lines. It is not a picking task for an operator.
- `OutboundOrderLine` represents the part of a `CustomerOrderLine` assigned to a specific warehouse execution and has its own state lifecycle.
- `Allocation` reserves inventory for an `OutboundOrderLine`.
- `PickTask` is an executable task for one `Warehouse Operator (Picker)`.
- `TU` is the physical transport unit used in Outbound.

Basic dependency chain:

`CustomerOrderLine → OutboundOrderLine → Allocation → PickTask → Picking TU`

### 3.2 `CustomerOrder`–`OutboundOrder` relationships

1. The relationship is many-to-many and is executed and settled at line and quantity level.
2. A `CustomerOrderLine` may be split across multiple `OutboundOrderLine` objects.
3. Several `CustomerOrder` objects may be combined before picking into one `OutboundOrder` if:
   - they belong to the same customer;
   - they have the same delivery address;
   - they have compatible `priority`;
   - their `slaDeadline` values fall within a tolerance configured per warehouse;
   - none of them has `allowPartialShipment = false`.
4. Splitting work by zone does not create separate `OutboundOrder` objects. `PickTask` objects are responsible for that split.
5. One `OutboundOrder` may generate many `PickTask` objects, including for many zones.

### 3.3 When `OutboundOrder` is created

1. After validation, a `CustomerOrder` may remain in `ACCEPTED` without an `OutboundOrder`, waiting for cyclic planning.
2. Cyclic planning skips `CustomerOrder.ON_HOLD`.
3. After `ON_HOLD` is removed, the order automatically returns to the planning queue.
4. **DEC-C05:** the first `OutboundOrder` is created during cyclic fulfillment planning, before `Allocation` is created.
5. **DEC-C06:** partial allocation does not create the first `OutboundOrder`. For `allowPartialShipment = true`, the existing `OutboundOrder` covers the quantity that can be fulfilled and the missing quantity remains `BACKORDERED`. A subsequent `OutboundOrder` is created only after inventory becomes available for the missing quantity.
6. For `allowPartialShipment = false`, exactly one `OutboundOrder` is created and it cannot aggregate other `CustomerOrder` objects.

### 3.4 Closing `CustomerOrder` and state cleanup (`BACKLOG.md` B15)

1. **DEC-L33 — `CustomerOrder SHIPPED→CLOSED` trigger:** header settlement is calculated automatically by the WMS System, not reported manually or by an external ERP signal — `CustomerOrder` transitions from `SHIPPED` to `CLOSED` when every `OutboundOrder` that contributed at least one shipped `OutboundOrderLine` to that `CustomerOrder` has reached `COMPLETED` (`propozycja_procesow_outbound.md` §4.10, `CarrierManifest HANDED_OVER→CONFIRMED`).
   **Rationale:** Darek's decision of 2026-08-18 — symmetry with the existing `OutboundOrder.COMPLETED`←`CarrierManifest` pattern (§4.10); it requires no new actor or integration channel. Rejected variants: manual close by `Warehouse Supervisor` (unnecessary manual step without business justification) and an external ERP/OMS signal (there is currently no equivalent gate for closing `CustomerOrder`, unlike the `POSTED` gate for `Shipment`).
2. **DEC-L34 — removal of `OutboundOrder.ON_HOLD`:** state `ON_HOLD` and transitions `PICKING_IN_PROGRESS↔ON_HOLD` were removed from `OutboundOrder` (`propozycja_procesow_outbound.md` §4.3, diagram). Backward verification (review of §3, §5, §7, §9 and B1–B14 history) found no described trigger, actor or resume condition for this state — it was a working artifact without business coverage. `ON_HOLD` remains only at `CustomerOrder` level (before `OutboundOrder` exists, section 3.3 points 2–3).
   **Rationale:** Darek's decision of 2026-08-18 — the product owner could not recall a use case; keeping an unnamed state in the canon creates a risk of inconsistent implementation. If a real need to suspend an individual `OutboundOrder` appears in the future, it requires a new explicit decision (a new `DEC-*`), not restoration of this state without rationale.

**Rules:** `propozycja_procesow_outbound.md` — point 1: `R85`, §3.1 step 13a; point 2: no new rule (removal).

## 4. Partial shipments, allocation and priorities

### 4.1 `allowPartialShipment`

1. `allowPartialShipment = false` does not limit the number of `PickTask` objects or Packing `TU` objects.
2. Missing even one required line blocks dispatch of the entire `Shipment`.
3. The entire `CustomerOrder` must be dispatched in one complete `Shipment`, which may contain multiple Packing `TU` objects.
4. `Warehouse Supervisor` may, with a reason, change `allowPartialShipment` from `false` to `true` only for the affected `CustomerOrder`.
5. The change applies until the end of that `CustomerOrder` lifecycle: every future shortage is then handled automatically as for `allowPartialShipment = true`, without involving the Supervisor again. It is not limited to one `Shipment`, but does not change customer or warehouse settings and does not affect other current or future `CustomerOrder` objects of that customer. It may be made before any `Shipment` exists for the order (for example during a `SHORT_PICKED` escalation while picking or `SHORT_ALLOCATED` during allocation); it leaves the missing quantity `BACKORDERED`, while the available quantity is fulfilled and dispatched without waiting for the remainder.

6. **DEC-L53 — `allowPartialShipment = false` guard keyed by `CustomerOrder`:**
   **Decision:** a Packing `TU` of an order with `allowPartialShipment = false` is not attached to any `Shipment` until every `CustomerOrderLine` of that `CustomerOrder` satisfies one of two mutually exclusive conditions. Condition (A), inactive line: the line has been successfully cancelled or corrected by `Warehouse Supervisor` and no longer belongs to the required quantity of that `CustomerOrder`; such a line does not block the guard and is not subject to condition (B), in particular it does not require any `OutboundOrderLine` to exist. Condition (B), active line: the sum of required quantities of that line's `OutboundOrderLine` objects that have reached `PACKED` or beyond and are not `CANCELLED` equals the effective `CustomerOrderLine.Quantity` after any correction, regardless of which fulfillment channel fulfilled the quantity. Until the condition is satisfied, the Packing `TU` remains in `PACKING_SEALED`. The guard blocks dispatch of the order in parts; it does not release remaining lines — the order waits as a whole.
   **Rationale:** the promise “the whole order in one complete `Shipment`” is a property of `CustomerOrder`, and a guard keyed by `OutboundOrder` leaks when an order is split between standard and cross-dock channels: channels cannot be mixed in one `OutboundOrder`, so two headers are created and the guard opens prematurely. The condition must be quantitative, not status-based — `CustomerOrderLine` transitions to `PLANNED` as soon as it is assigned to an `OutboundOrderLine`, so a partially covered line (example: `Quantity` 100, 30 covered and packed, remaining 70 with no `OutboundOrderLine`) would satisfy a status-based condition even though the order is incomplete.
   **Rejected alternatives:** none.
   **Date:** 2026-08-25.
   **Rules:** `P1 R57`, `P1 R58`.

7. **DEC-L57 — `requiredQty` attribute on `OutboundOrderLine` and its lifecycle:**
   **Decision:** `OutboundOrderLine` receives the `requiredQty` attribute — the required quantity of that line. It is the currently applicable target quantity, not a historical one. It is set when the line is created: in channel `STANDARD` when `OutboundOrder` is created, in channel `CROSSDOCK` when `CrossDockPickTask` is generated. It changes only together with a `CustomerOrderLine.Quantity` correction by `Warehouse Supervisor`, in two named paths: `P1 R46` for the standard channel and the correction branch of `P2 R15` for the cross-dock channel. No other path changes it; there is no automatic update. The `allowPartialShipment = false` guard compares the sum of `requiredQty` with `CustomerOrderLine.Quantity` using equality.
   **Rationale:** four places in the canon refer to the required quantity of `OutboundOrderLine`, while no object stored it — the only quantity attribute on that line, `pickedQty`, describes the quantity actually picked and explicitly does not apply in the cross-dock channel. Without this attribute, the guard keyed by `CustomerOrder` cannot be expressed quantitatively. The name `requiredQty` states “required quantity” directly, in line with those four locations, and is clearly different from both `pickedQty` and `CustomerOrderLine.Quantity`. A solution with an immutable attribute and replacing the line with a new one conflicts with `P2 R15`, which says the same line continues to `PACKING_SEALED`. A solution with an immutable attribute and inequality comparison weakens the only condition protecting `allowPartialShipment = false`. The consciously accepted cost is narrowing `DEC-L38`, recorded in a separate entry.
   **Rejected alternatives:** none.
   **Date:** 2026-08-26.
   **Rules:** `P1 R46`, `P1 R69`, `P2 R15`.

**Rules:** `propozycja_procesow_outbound.md` — points 2–3: `SP11`, `R5`; points 4–5: `SP12`, `R6`.

8. **DEC-L64 — the `allowPartialShipment = false` gate counts mixed coverage:**
   **Decision:** the planning-entry condition for `allowPartialShipment = false` does not ask for full `ATPReservation`, but for full coverage of each `CustomerOrderLine`: `ATPReservation` plus the sum of `requiredQty` of its `OutboundOrderLine` objects that have reached `PACKED` or beyond and are not `CANCELLED`, regardless of channel. A standard `OutboundOrder` covers only uncovered quantity; if cross-dock coverage is complete, no standard `OutboundOrder` is created at all.
   **Rationale:** cross-dock does not create `ATPReservation`, so the previous wording of the gate never opened for a line covered by that channel — an order forbidding partial shipment could remain in `ACCEPTED` indefinitely even though the goods were physically present. The coverage measure is deliberately the same as in `P1 R58`, so that the body does not contain two definitions of a “covered line”.
   **Rejected alternatives:** counting cross-dock coverage from creation of `CrossDockPickTask` instead of from `PACKED` — rejected because cross-dock quantity is uncertain until physical confirmation and an order forbidding partial shipment must not leave incomplete. This deliberately differs from the measure in `P2 R6`, where everything promised is subtracted; both conditions are conservative, but in opposite directions.
   **Date:** 2026-08-28.
   **Rules:** `P1 R6`.

### 4.2 Shortage during allocation

1. The system reserves the available quantity regardless of `allowPartialShipment`.
2. With `allowPartialShipment = true`, partial allocation leaves the available quantity in the existing `OutboundOrder`; it does not create the first or an additional `OutboundOrder` at allocation time.
3. The missing quantity remains in the same `CustomerOrder` as `BACKORDERED`.
4. With `allowPartialShipment = false`, a partial reservation remains blocked until the whole quantity becomes available or is released according to warehouse policy.
5. The partial-reservation retention policy is configurable and may mean:
   - retain the reservation;
   - automatically release it after a specified time;
   - route it to a `Warehouse Supervisor` decision.
6. Reservation retention time does not depend on `priority` or `slaDeadline`.
7. Automatic reservation release does not require notifying the Supervisor.
8. **DEC-D11:** `SHORT_ALLOCATED` is handled automatically according to `allowPartialShipment` and the reservation-retention policy. `Warehouse Supervisor` participates only when required by the configured policy or when an exceptional decision is needed.
   **Rules:** `propozycja_procesow_outbound.md` — `R33`, `R39`.
9. **DEC-D12:** for a `CustomerOrder` with `allowPartialShipment = false`, cyclic planning does not create an `OutboundOrder` until every `CustomerOrderLine` of that order has a full `ATPReservation` (equal to ordered quantity). `CustomerOrder` remains in `ACCEPTED` as long as necessary. Rationale: without this condition, an `OutboundOrder` could be created with partial reservation (some lines `RESERVED`, some `SHORT`), which with `allowPartialShipment = false` would still allow nothing to ship (even one missing item blocks the whole `Shipment`, D-4.1 point 2) — while requiring later cancellation and cleanup (`Allocation RELEASED`, return to `ATPReservation`) only to start again. Waiting for complete reservations before anything is physically blocked is simpler: `ATPReservation` (soft reservation, decisions section 4.3 points 8–14) already protects available inventory from other orders according to queue rules (points 10–11), so nothing is lost while waiting.
   **Rules:** `propozycja_procesow_outbound.md` — `R49`.
10. **DEC-D13:** despite DEC-D12, `SHORT_ALLOCATED` can still occur for `allowPartialShipment = false`, but rarely — as an edge case: between the moment all lines reach full `ATPReservation` and the moment that reservation is actually converted into a real `Allocation`, the priority queue (variants using priority/`slaDeadline`, decisions section 4.3 point 11b) may take part of the `ATPReservation` from a newly accepted order in favor of a higher-priority order before it becomes a hard reservation. In this case `SHORT_ALLOCATED` and escalation to `Warehouse Supervisor` work exactly as described in R40 (unchanged) — two outcomes: permanent change of `allowPartialShipment` to `true`, or cancellation of the `OutboundOrder`.
   **Rules:** `propozycja_procesow_outbound.md` — `R40`, `R50`.
11. **DEC-D14:** applies only to `allowPartialShipment = false` (continuation of DEC-D13). When `Warehouse Supervisor` chooses cancellation: `OutboundOrder` transitions to `CANCELLED`, every covered `Allocation` (full or `SHORT`) to `RELEASED`, and its corresponding quantity returns to the `ATPReservation` of the relevant `CustomerOrderLine` (R47) — `Allocation` at `SHORT_ALLOCATED` is created only after real availability is known, so it is never overstated, and its whole quantity returns without correction. The `CustomerOrderLine` state after return depends on which line actually lost the `ATPReservation` race described in DEC-D13: a line whose reservation was taken for a higher-priority order transitions to `BACKORDERED` (a real unresolved availability problem — a signal for `Warehouse Supervisor`). The other lines of the same `OutboundOrder` that managed to obtain a real `Allocation` and had no availability issue of their own transition to `OPEN` — they return to the pre-planning state without a shortage mark. `CustomerOrder` returns to `ACCEPTED` with `WARNING` set (DEC-D16) and waits as a whole, just as on the first attempt (DEC-D12), until all lines together regain full reservation.
   **Rules:** `propozycja_procesow_outbound.md` — `R47`, `R50`, `R56`, `R57`, `R60`.
12. **DEC-D15:** header state `BACKORDERED` may occur only when all active `CustomerOrderLine` objects of a given `CustomerOrder` are simultaneously `BACKORDERED` (a real unresolved availability problem on each of them — not merely orderly rollback, DEC-D14/DEC-F15). The rule applies regardless of `allowPartialShipment` (extended 2026-08-02 — closes L7 together with DEC-D18, §4.4). In a typical flow this is rare: usually only one line (the one that actually lost the reservation race at `SHORT_ALLOCATED`, or had a physical shortage at `SHORT_PICKED`) becomes `BACKORDERED`. Other lines: for `allowPartialShipment = false`, when the whole `OutboundOrder` is cancelled (DEC-D14), return to `OPEN` and the header to `ACCEPTED` + `WARNING` (R56); for `allowPartialShipment = true`, the other lines are not moved and remain at their current execution stage, while the header remains `IN_FULFILLMENT` + `WARNING`. Header `BACKORDERED` is therefore an extreme edge case, not a normal waiting state — for either value of `allowPartialShipment`.
   **Rules:** `propozycja_procesow_outbound.md` — `R56`, `R57`, `R60`.
13. The rule from point 12 (header `BACKORDERED` only when all active `CustomerOrderLine` objects are `BACKORDERED`) applies regardless of `allowPartialShipment` — the general line→header aggregation rule for `CustomerOrder`, together with the `PARTIALLY_SHIPPED` exception and the `WARNING` trigger for mixed cases, described in DEC-D18 (§4.4).
14. **DEC-D16:** `CustomerOrder` receives a new `WARNING` feature — one flag with a text description (not a list of many independent warnings). The WMS System sets it in four situations: (a) an order with `allowPartialShipment = false` remains in `ACCEPTED` without full `ATPReservation` on all lines (DEC-D12); (b) `SHORT_ALLOCATED` occurs and requires a Supervisor decision (DEC-D13); (c) `SHORT_PICKED` occurs and requires a Supervisor decision (section 5, DEC-F12); (d) with `allowPartialShipment = true`, at least one `CustomerOrderLine` is `BACKORDERED` but not all order lines are — the header remains `IN_FULFILLMENT`, not `BACKORDERED` (DEC-D15/DEC-D18), but requires Supervisor review. Purpose: `Warehouse Supervisor` has one place (`CustomerOrder` monitoring panel) to review orders requiring a decision instead of searching for them separately in every object type.
   **Rules:** `propozycja_procesow_outbound.md` — `R51`, `R60`.
15. **DEC-D17:** `WARNING` disappears in two cases: (a) manually, when `Warehouse Supervisor` sets the flag to `false` — if the cause has not actually been resolved, the next cyclic-planning run sets it back to `true` after detecting the same shortage; (b) automatically, when the underlying problem is actually resolved — cyclic planning creates an `OutboundOrder` for an order from DEC-D12, or a `SHORT_PICKED`/`SHORT_ALLOCATED` case is settled (successful pick/reallocation, or a Supervisor decision closing the case).
   **Rules:** `propozycja_procesow_outbound.md` — `R51`.

16. **DEC-L61:** `Allocation` receives its own quantity attribute `reservedQty` — the quantity of inventory actually blocked by it. Inventory is occupied only in states `SHORT`, `RESERVED` and `CONFIRMED`, in the amount of `reservedQty`; `PENDING`, `RELEASED` and `CONSUMED` contribute nothing. Variant A was selected from three considered variants: (A) own attribute on `Allocation`; (B) as A, but `PENDING` blocks full `requiredQty`; (C) no attribute, occupied quantity calculated from `Inventory` records. Variant B was rejected — it would show not-yet-identified inventory as occupied and the `PENDING` window is short, so the benefit would be theoretical. Variant C was rejected — it would require first expanding `Inventory`, which is described only fragmentarily in Outbound, i.e. a real scope extension. Reason for the decision: the previous sentence “reservation quantity is identical to `requiredQty`” is false for `SHORT`, which exists precisely when reservation is partial. Formula `P2 R6` is not changed by this decision — channel exclusivity remains open as `BACKLOG.md` B22.
    **Rules:** `proces_1_standard_fulfillment.md` — `R71`, STEP 4; `model_stanow_outbound.md` — section 5 `Allocation`.

### 4.3 `priority` and `slaDeadline`

1. `CustomerOrder` has both `priority` and `slaDeadline`.
2. An `OutboundOrder` aggregating multiple orders considers both criteria and takes the most urgent values.
3. Priority affects the order of `ATPReservation` assignment (allocation order, see points 8–14) and the order of `PickTask` execution (see point 7).
4. Priority applies only to inventory not yet reserved.
5. Existing `Allocation.RESERVED` must not be taken from a lower-priority order.
6. Priority reallocation is not possible after `PickTask` is created or after physical picking starts.
7. The order in which the Warehouse Operator executes `PickTask` objects (detail of “picking” from point 3) is a warehouse parameter: (a) `slaDeadline` → `priority` as tie-break — default; (b) `priority` → `slaDeadline` as tie-break. A tie on both criteria is resolved by `PickTask` submission order — the task goes to the next free Warehouse Operator.
8. `CustomerOrderLine` has an `ATPReservation` feature (beside `Quantity`) — the amount of `ATP` inventory softly reserved for that line before a real `Allocation` exists. Availability for new reservations on a given SKU = `Inventory.AVAILABLE` (with the `ATP` flag) − sum of active `ATPReservation` on that SKU. `Allocation` is not subtracted separately — its effect is already reflected in state `AVAILABLE`. The sum of active `ATPReservation` on a given SKU never exceeds `ATP`; therefore `ATPReservation` may be non-zero only when `ATP > 0` (necessary, not sufficient condition).
9. `ATPReservation` for `CustomerOrderLine` is set when `CustomerOrder` reaches `ACCEPTED`, regardless of `allowPartialShipment`. If availability for the SKU is 0 at that moment, the new line receives `ATPReservation = 0` and waits in the queue.
10. The order of `ATPReservation` assignment among waiting `CustomerOrderLine` objects for the same SKU is a warehouse parameter: (a) submission order only, without priority or `slaDeadline` — default; (b) priority → `slaDeadline` as tie-break; (c) `slaDeadline` → priority as tie-break.
11. The queue is recalculated on every event changing its composition for a given SKU: (a) `Inbound TU POSTED` for an ASN containing that SKU — applies to all three variants: in variant (a), newly available goods go in sequence to waiting lines in submission order, among those for which `ATPReservation` + quantity covered by a real non-released `Allocation` < `Quantity`; in variants (b)/(c), the queue is additionally re-sorted by priority/`slaDeadline` before assignment. (b) a new `CustomerOrder` reaches `ACCEPTED` — applies only to variants (b)/(c): it may take `ATPReservation` previously assigned to lower-priority/further-back lines provided those lines do not yet have a real `Allocation`; in variant (a), the new order simply joins the end of the submission queue and does not disturb anyone's existing reservation.
12. A real `Allocation` in `RESERVED` is never taken by later or higher-priority `CustomerOrder` objects — per point 5. The queuing limitation (point 11) applies only to soft `ATPReservation`, never to a real `Allocation`.
13. As a real `Allocation` is created for `CustomerOrderLine`, the corresponding quantity is subtracted from that line's `ATPReservation` (transition from soft to hard reservation, maintained through the rest of the lifecycle — `PICKING`, `PICKED`, `PACKED`, until `SHIPPED`). If an `OutboundOrderLine` is cancelled before `SHIPPED` for a reason unrelated to a record-vs-reality discrepancy (for example a Supervisor decision to roll back a sibling line, DEC-F15), the entire quantity covered by that `Allocation` returns to that line's `ATPReservation` — `Allocation` is never overstated versus real availability in such cases. If cancellation occurs due to confirmed `SHORT_PICKED` (a physical shortage detected during picking even though `Allocation` had previously been created for the full quantity), only `Allocation quantity − confirmed missing quantity` returns to `ATPReservation` — the missing part remains blocked for inventory checking (DEC-F10) and is not real `ATP` until that check resolves it.
14. `Warehouse Supervisor` may manually reduce or remove `ATPReservation` for a `CustomerOrderLine` without affecting `CustomerOrder` status and without providing a reason.

**Rules:** `propozycja_procesow_outbound.md` — points 5–6: `SP13`, `R11`; point 8: `R42`; point 9: `R43`; point 10: `R44`; point 11: `R45`; point 12: `R46`; point 13: `R47`; point 14: `R48`.

### 4.4 Line→header state aggregation (`CustomerOrder`, L1/L7)

1. **DEC-D18:** `CustomerOrder` header status relative to its `CustomerOrderLine` statuses follows the rule: header = status of the least advanced active line, with two exceptions: (a) when at least one line reaches `SHIPPED` (part of the order dispatched), the header becomes `PARTIALLY_SHIPPED` regardless of remaining lines; (b) the header becomes `BACKORDERED` only when all active lines are `BACKORDERED` — for any other combination containing at least one `BACKORDERED`, the header remains in its current fulfillment status (`IN_FULFILLMENT`, or for `allowPartialShipment = false` after cancellation of the entire `OutboundOrder` — `ACCEPTED`) with `WARNING` set (DEC-D15, DEC-D16). This applies only to `CustomerOrder`/`CustomerOrderLine` — `OutboundOrder` does not need the rule: it has no `BACKORDERED` state (`SHORT_ALLOCATED` after creation ends in `CANCELLED`, DEC-D14) and no intermediate “partially shipped” state (`COMPLETED` only after all its `Shipment` objects are confirmed, `propozycja_procesow_outbound.md` SP4); its aggregation is already fully covered by SP2–SP4 and R58. Closes L1 (`OutboundOrder`, already covered by SP2–SP4/R58) and L7 (`CustomerOrder`, this rule).
   **Rationale:** Darek's decision of 2026-08-02 — a conservative rule (the header does not “promise” more than the weakest line), with partial shipment explicitly singled out as the only “positive” exception; `BACKORDERED` is limited to the case when fulfillment has genuinely stopped entirely — a single shortage line while the remainder is in progress must not suggest that the entire order is waiting (the signal for that case is `WARNING`, not a header-state change). It also avoids expanding the vocabulary with a full set of `PARTIALLY_PICKED`/`PARTIALLY_PACKED`, etc.
   **Rules:** `propozycja_procesow_outbound.md` — `SP1`, `SP2`–`SP4`, `R58`, `R60`.
   **Behavior detail:** `propozycja_procesow_outbound.md` §3.1, Continuous Function F1; §4.1 (`CustomerOrder` diagram, edge `IN_FULFILLMENT↔BACKORDERED`).

## 5. `PickTask` and shortage handling

1. One `PickTask` is assigned to one operator.
2. One `OutboundOrder` may generate multiple `PickTask` objects for different zones.
3. The Picking `TU` assignment strategy is configurable per warehouse:
   - separate Picking `TU` objects for tasks or zones;
   - shared Picking `TU` objects passing through successive tasks.
4. One `PickTask` may use several Picking `TU` objects after the previous one becomes full.
5. Several `PickTask` objects may work on the same Picking `TU` concurrently only within one `OutboundOrder`.
6. Before adding goods, the operator must scan the Picking `TU`.
7. The system blocks exceeding the `TU` weight or volume limit.
8. `SHORT_PICKED` ends the original `PickTask`.
9. Reallocation of the missing quantity creates a new `PickTask` inheriting the source task's priority.
10. The location where the shortage was reported is automatically blocked for inventory checking.
11. **DEC-F10:** `SHORT_PICKED` ends the original `PickTask` and automatically blocks the source location for inventory checking.
    **Rules:** `propozycja_procesow_outbound.md` — `R17`, `R47`.
12. **DEC-F11:** if qualified `ATP` inventory exists in another unblocked location and the effective attempt limit has not been reached, the WMS System automatically creates a new `PickTask` for the missing quantity. The new task inherits the source task's priority.
13. **DEC-F12:** the case goes to `Warehouse Supervisor` when no qualified inventory exists, the automatic reallocation limit has been reached, or a decision is needed concerning `BACKORDERED`, cancellation or deviation from `allowPartialShipment = false`.
14. **DEC-F13:** parameter `maxAutomaticShortPickReallocations` is a warehouse setting with default value `1`. In customer master data it has default value `null`. A customer value other than `null` overrides the warehouse setting; `null` means inheritance. Value `0` means no automatic reallocation.
15. **DEC-F14:** the automatic-reallocation counter applies to a specific case defined by `OutboundOrderLine` and missing quantity. Successive automatically created `PickTask` objects continue the same counter. A successful pick closes the case.
16. **DEC-F15 — Outcome 1 — “wait” (`BACKORDERED`).** Applies only to `allowPartialShipment = false`. `Warehouse Supervisor` decides: keep the customer's original request; the entire shipment will leave together only when complete. Effects, object by object:
    - the `OutboundOrderLine` of the short-picked line transitions to `CANCELLED` (same mechanism as any other cancellation, existing edge `SHORT_PICKED → CANCELLED`, §4.4).
    - its `Allocation` transitions to `RELEASED`. The quantity returned to that `CustomerOrderLine`'s `ATPReservation` is the `Allocation` quantity **minus the confirmed missing quantity** — not the whole `Allocation` (R47 correction): the missing part is blocked for inventory checking (DEC-F10) and is not actually available `ATP` until the check resolves it. That `CustomerOrderLine` transitions to `BACKORDERED` — it has a real unresolved availability issue visible to `Warehouse Supervisor` in monitoring.
    - for quantity already physically picked (`pickedQty > 0`), a `PutBackTask` (P4) is created — goods return to the shelf.
    - **all remaining lines of the same `CustomerOrder` that are in `ALLOCATED` or `PICKED` (but not `PACKED`) are rolled back through the same path** — `OutboundOrderLine → CANCELLED`, `Allocation → RELEASED`, the **entire** `Allocation` quantity returns to `ATPReservation` (no reduction — this is ordinary cancellation without a record/reality discrepancy), and a `PutBackTask` is created for what was already picked. These lines (with no availability problem of their own) transition to `OPEN` — they return to the pre-planning state. This happens automatically as the mechanical consequence of one Supervisor decision — no separate decision per line is required. Rationale: because the whole shipment cannot leave until the missing quantity is found, holding reserved/picked goods from other lines needlessly blocks them from other orders.
    - **lines already `PACKED` are not moved** — they remain physically in the consolidation area and wait; their `CustomerOrderLine` remains `PLANNED`. Rationale: unpacking an already packed unit is operationally costly and would require Supervisor approval anyway (D-8.4/R32), so it makes no sense to do it automatically.
    - `CustomerOrder` returns to `ACCEPTED` with `WARNING` (DEC-D16) **only if no `OutboundOrderLine` of the order remains `PACKED`.** If at least one is `PACKED` and waiting, `CustomerOrder` **remains in `IN_FULFILLMENT`** with `WARNING` set — an active `OutboundOrder` still covers the packed part, so the header cannot formally return to its pre-planning state. In both cases, a new `OutboundOrder` for the missing/rolled-back part is created only when the required lines together regain full reservation (DEC-D12).
    **Rules:** `propozycja_procesow_outbound.md` — `R47`, `R52`, `R56`, `R57`.
17. **DEC-F16 — Outcome 2 — “cancellation”.** Key fact: cancelling the `OutboundOrderLine` alone is **not sufficient** to allow anything to leave — `allowPartialShipment = false` lives on `CustomerOrderLine`/`CustomerOrder`, not on `OutboundOrderLine`, so even after cancelling the warehouse task, the required quantity on the customer order does not change and shipment remains blocked (D-4.1 point 2). The only effective solution is **editing quantity on `CustomerOrderLine` by `Warehouse Supervisor`** — WS decides how much to cancel:
    - **WS cancels the full originally ordered quantity** from `CustomerOrderLine` (e.g. 100 of 100) → this removes the entire SKU from the order. `OutboundOrderLine` transitions to `CANCELLED`, `Allocation` to `RELEASED`, `ATPReservation` returns to zero for the line (the line is no longer needed by anyone), and a `PutBackTask` is created for the full quantity already picked.
    - **WS corrects `CustomerOrderLine` to the quantity actually picked** (e.g. from 100 to 60) → the order is fulfilled in that reduced form because the “required quantity” is now 60 and 60 was fully picked. No `PutBackTask` is needed for those 60 — what was picked exactly matches the corrected order.
    - This mechanism partially addresses `BACKLOG.md` B3 (“cancellation criteria for `CustomerOrder`/`CustomerOrderLine`, full and partial triggers”) — exactly for this trigger (`SHORT_PICKED` escalation). Other B3 aspects (general triggers unrelated to shortage) remain open. The distinction from sales correction is resolved by the mechanism that reports dispatch readiness to ERP — see `propozycja_procesow_outbound.md` §3.1 step 11a and §4.9 (`Shipment POSTING_PENDING`/`POSTED`/`POSTING_ERROR` states).
    **Rules:** `propozycja_procesow_outbound.md` — `R53`.
18. **DEC-F17 — Outcome 3 — “permanent change of `allowPartialShipment` to `true`”.** Simpler than outcome 2 because WS has agreed partial fulfillment with the customer — no need to edit `CustomerOrderLine` quantity. The `OutboundOrderLine` for the missing part simply transitions to `CANCELLED` (same mechanism as outcome 1, R47), the available quantity continues and is dispatched as a separate shipment. From then on, every future shortage on that `CustomerOrder` is handled automatically as with `allowPartialShipment = true` (R6), without involving the Supervisor again.
19. **DEC-F18 — missing goods detected during packing (“repack by SKU” check, DEC-G12):** two detection moments — (a) while counting one SKU, when counted quantity is lower than that line's `pickedQty`; (b) when finishing repacking the source `TU`, when the WMS System determines that not all system-expected SKU have been accounted for. In both cases the shortage is **not recorded automatically** — at (a) the Packer may defer the decision and return to the line later (before finishing the `TU`); reporting a shortage in either case requires (1) rechecking the `TU` contents against that line and (2) explicit confirmation of the report by the Packer. Only that confirmation starts the mechanism: the WMS System corrects `pickedQty` to the actually confirmed quantity, `OutboundOrderLine` transitions `PICKED → SHORT_PICKED` for the missing part, and then exactly the same mechanism as for `SHORT_PICKED` detected during picking applies (DEC-F11–DEC-F17): automatic reallocation within `maxAutomaticShortPickReallocations`, after exhaustion — escalation to `Warehouse Supervisor` with three outcomes. **Deviation from DEC-F10:** the source location of the original `PickTask` is not blocked — there is no certainty whether the cause is operator error during `Pick` or a real shortage at the location. The event is always recorded as a discrepancy in the context of the original `PickTask`, deferred for investigation by `Warehouse Supervisor`; the investigation mechanism itself (including settlement of missing quantity in `Inventory`) is out of scope — `WMSAI_OUTBOUND/BACKLOG.md` B9.
    **Rationale:** Darek's decision of 2026-08-07 — reuse the proven `SHORT_PICKED` mechanism instead of creating a parallel mechanism; confirmation prevents premature shortage reporting by an operator who has deliberately not searched the entire `TU`; not blocking the location prevents falsely blocking a valid location due to potential operator error.
20. **DEC-F19 — damaged goods detected during packing:** on the “repack by SKU” path (DEC-G12), damaged goods must not be repacked into a Packing `TU`. The Packer reports the SKU as `DAMAGED`; the WMS System instructs them to place it at the designated `QC` location. For that quantity, exactly the same mechanism as DEC-F18 starts (`PICKED → SHORT_PICKED`, reallocation/escalation, no location block) — without DEC-F18's two-step confirmation requirement (damage is a direct physical observation, without uncertainty whether the operator searched the entire `TU`). The event is marked as `DAMAGED` (not a generic shortage) in the discrepancy register — `WMSAI_OUTBOUND/BACKLOG.md` B9.
    **Rationale:** Darek's decision of 2026-08-07 — the fulfillment consequence is the same as for missing goods (the goods will not reach the customer), but the cause (damage, not loss/mistake) is a separate fact worth marking for future investigation.

### 5.1 Picking directly into Outbound `TU`

21. **DEC-L35 — alternative picking path bypassing the Packer:** at the first scan of a Picking `TU` for a given `PickTask`, the Picker may declare picking directly into an Outbound `TU` (new attribute `TU.directPackDeclared = true`, `propozycja_procesow_outbound.md` §4.7, §3.1 step 6a) — a binding and irreversible decision for that task. When, after picking is complete, the `TU` meets issuance conditions, the WMS System automatically performs `READY_TO_PACK→PACK_QUALIFIED→PACKING_SEALED` (§4.7) and `OutboundOrderLine PICKED→PACKED` (§4.4) — without Packer participation. When the `TU` **does not** meet issuance conditions despite the declaration, the WMS System routes it to standard Packer evaluation (§3.1 step 7, keep/repack/consolidate) — exactly as for a `TU` without the declaration; the Packer remains the second, fallback verification point. No changes to the `TU`/`OutboundOrderLine` state diagrams (no new states/edges) — only the actor and timing of the existing evaluation change.
    **Rationale:** Darek's decision of 2026-08-18 — today's evaluation “does the `TU` meet issuance conditions” (step 7) already exists as a mechanism; the new path lets an experienced Picker declare this up front (when the task's nature indicates the `TU` will go directly to dispatch), reducing execution time without creating a new evaluation mechanism. The risk of a wrong declaration is limited by the same issuance condition as today — if the declaration proves wrong, the `TU` still goes to the Packer, so there is no unverified path.
    **Rules:** `propozycja_procesow_outbound.md` — `R86`, §3.1 step 6a/7, §4.7, §4.4.

22. **DEC-L65 — inheritance of `directPackDeclared` between successive Picking `TU` objects of the same `PickTask`:**
   **Decision:** the next Picking `TU`, created after the previous one reaches `PICK_FULL` within the same `PickTask`, inherits its `directPackDeclared`. The operator is not asked again. This narrows `DEC-L43` in the part that speaks about a separate declaration for each `TU`.
   **Rationale:** the declaration concerns how one task is executed, not one carrier; asking the operator again in the middle of the same task is unnecessary and permits an inconsistent result — one part of the task picked directly into an Outbound `TU`, another part not.
   **Rejected alternatives:** keeping a separate declaration per `TU` — rejected as operationally burdensome and allowing inconsistency within one task.
   **Date:** 2026-08-28.
   **Rules:** `P1 R16`, `P1 R67`.

## 6. Outbound `TU` model

### 6.1 Identity and numbering

1. Inbound `TU` and Outbound `TU` are separate objects with separate state lifecycles.
2. Internal software identification of objects is outside the process-layer scope.
3. `TU_NUMBER` is the required business number of an Outbound `TU` and is unique within the warehouse among active (non-terminal) Outbound `TU` objects.
4. `SSCC` is an optional GS1-compliant identifier. An internal number must not be called `SSCC`.
5. `TUSetup` is a closed dictionary of `TU` types. It defines `externalIssuable`, `maxWeight`, `maxVolume`, `processUsage`, `outbound_tu_number_standard`, `numberFormatMask` and the `sequenceCode` reference for internally created Outbound `TU` objects.
6. `Sequence` is an independent system counter with `sequenceId`, a unique code, name and next free number. Multiple `TUSetup` objects may reference the same `Sequence` (N:1 relationship).
7. An external `TU` does not receive a number from `Sequence`, but is assigned a `TUSetup` identifying external origin.
8. **DEC-G07:** a valid `SSCC` may simultaneously be the business `TU_NUMBER` if it satisfies the `TU_NUMBER` uniqueness rule.
9. **DEC-G08:** a value that does not comply with SSCC rules must not be stored in field `SSCC`; the field remains empty.
10. **DEC-G09:** a unique external number that is not a valid SSCC may be stored as `TU_NUMBER`.
11. **DEC-G10:** a `TU_NUMBER` conflict requires explicit handling. Silent overwrite, automatic suffix addition and storing an ordinary number in the `SSCC` field are forbidden.
12. **DEC-G14 — `SSCC` validation:** `SSCC` is valid only if it has exactly 18 digits and a correct GS1 check digit calculated using the mod-10 algorithm. When generating an ordinary (non-cross-dock) Outbound `TU`, the WMS System does not verify the company prefix against an external GS1 registry: it generates `SSCC` only from its own configured prefix and does not accept a foreign `SSCC`. The exception is `SSCC` inheritance in 1:1 cross-docking (`DEC-G17`, `R27`).
13. **DEC-G15 — non-`SSCC` `TU_NUMBER` format:** a `TU_NUMBER` that is not `SSCC` uses Code 128 symbology, contains only alphanumeric characters without special characters and has a maximum of 20 characters.
14. **DEC-G16 — uniqueness and reuse of `TU_NUMBER`:** `TU_NUMBER` is unique only among active (non-terminal) Outbound `TU` objects. Reuse will practically occur only after a `Sequence` is exhausted and manually reset. Resetting `Sequence` is technical administration, analogous to monitoring certificates, disks or CPU, and remains outside business-process descriptions.
15. **DEC-G17 — no later correction or reclassification:** `TU_NUMBER` and `SSCC` are determined deterministically when an Outbound `TU` is created: by generation compliant with `TUSetup`/`Sequence`, or by inheritance from an Inbound `TU` only in 1:1 cross-docking when creation-time validation confirms GS1. There is no later correction stage, storage of a raw scan or number reclassification, therefore a mechanism analogous to Inbound `ORIGINAL_CODE` does not apply. See `K10` in `propozycja_procesow_outbound.md`.
16. **DEC-G18 — number conflict:** a `TU_NUMBER`/`SSCC` conflict between different physical units is impossible when GS1 is followed. If it nevertheless occurs, it indicates a configuration or data error handled in service mode, outside business processes and without escalation to `Warehouse Supervisor`. See `K11` in `propozycja_procesow_outbound.md`.
17. **DEC-G19 — `Sequence`/`TUSetup` model:** `Sequence` is a universal system counter with `sequenceId`, unique code, name and next free number. `TUSetup` is a closed dictionary of `TU` types with type code as the key, `externalIssuable`, `maxWeight`/`maxVolume` limits, `processUsage`, `outbound_tu_number_standard` (`GS1`/`OWN`), `numberFormatMask` and `sequenceCode` pointing to `Sequence`. Multiple `TUSetup` objects may reference the same `Sequence`; every Outbound `TU` stores a reference to `TUSetup`.

18. **DEC-L51 — external carrier type for Outbound `TU`:**
    **Decision:** an Outbound `TU` of external origin receives the `tuSetupCode` of the external type. The warehouse configures exactly one such type. Such a `TU` is issuable without meeting issuance thresholds — approval is part of the process, not a separate acceptance step for each `TU`. The semantic contract of the external type comprises four determinations: (1) the external type is excluded from `P1 R64` threshold evaluation, which gets a priority branch — a `TU` whose type has the external-carrier role satisfies issuance thresholds by definition and lower thresholds are not evaluated for it; (2) issuability follows from `externalIssuable = true` on the external type and the branch in point 1, so no numeric comparison with an undefined value occurs; (3) `P1 R65`, manual forcing of issuability by the operator, does not participate on this path — it concerns a `TU` on an internal type that has not reached lower thresholds; (4) threshold and limit attributes of the external type have no value, and the catalog records this as “not applicable” — never `0`, never a boundary value, never a substitute value. Applicability of `TUSetup` attributes for the external type: `tuSetupCode` required; `externalIssuable` required, value `true`; `processUsage` required, and the value identifying this type is established by `DEC-L59`; `maxWeight`, `maxVolume`, `minIssueWeight`, `minIssueVolume`, `outbound_tu_number_standard`, `numberFormatMask` and `sequenceCode` — not applicable, no value. Missing `maxVolume` means Carrier Selection has no value with which to match a volume interval, so a `Shipment` containing such a `TU` goes to manual carrier selection (`P1 R51`) — an intended effect.
    **Rationale:** the term “external `TU`”, used by `P1 R51`, had no object definition in the body. A carrier originating outside the warehouse has no reliable maximum weight or dimensional limits, so limits and thresholds of internal types do not apply. Manual carrier selection is a consciously accepted consequence of the decision, not a gap.
    **Rejected alternatives:** none.
    **Date:** 2026-08-25.
    **Rules:** `P1 R51`, `P1 R64`, `P1 R65`, `P1 R68`.

19. **DEC-L59 — `TUSetup.processUsage` value identifying the external type:**
    **Decision:** `TUSetup.processUsage` — an enum that previously had no defined value — receives exactly one defined value: `EXTERNAL`, meaning a carrier of external origin. The enum remains open; the remaining carrier roles and closing the dictionary are outside the scope of this decision. An Outbound `TU` of external origin is recognized exclusively by its type's `processUsage`, never by the `TU` number. No boolean flag duplicating `processUsage` is created.
    **Rationale:** without a discriminator, the sentence “the warehouse configures exactly one external type” cannot be represented because the WMS System would not know how to identify it. Recognition by `TU` number is excluded — `P2 R7` allows both inheritance of `TU_NUMBER`/`SSCC` from the source and assignment of a number from `Sequence`, so the number carries no information about carrier role. A separate boolean flag beside an empty `processUsage` would introduce a second carrier-role representation and would need to be removed when the dictionary is closed. Defining a full closed dictionary immediately goes beyond what the external-type definition itself requires.
    **Rejected alternatives:** none.
    **Date:** 2026-08-26.
    **Rules:** `P1 R68`.

### 6.2 Outbound Crossdock

1. The process begins when an Inbound `TU` reaches `IN_CROSS_DOCK`. The WMS System generates `CrossDockPickTask` objects and related `OutboundOrderLine` objects in `CREATED` for matched `CustomerOrderLine` objects in `BACKORDERED`; the `CustomerOrderLine` objects transition to `PLANNED`. `CrossDockPickTask` is performed by the Packer, who scans SKU, quantity and quality `OK`/`DAMAGED`.
2. **DEC-L03 — Inbound/Outbound boundary:**
   [HISTORICAL — clarified by DEC-L29] “task settlement” in this entry proved ambiguous — see DEC-L29 for the current exact moment when Outbound `TU` is created.
   **Decision:** during `IN_CROSS_DOCK`, no Outbound `TU` exists; `CrossDockPickTask` operates on the source Inbound `TU`, and the Outbound `TU` is created only when the task is settled.
   **Rationale:** this separates process responsibilities without maintaining two parallel `TU` objects for the same physical unit before the sorting result.
   **Rejected alternatives:** an Outbound `TU` existing in parallel with a non-terminal Inbound `TU`.
   **Date:** 2026-08-10.
   **Rules:** no new rule — behavior described in `propozycja_procesow_outbound.md` §3.2 step 1 and step 3.
3. **DEC-L04 — two settlement cases:**
   **Decision:** with a full 1:1 match, the entire contents of the Inbound `TU` close one `Shipment` according to `SP14`/`R20`; the Outbound `TU` retains `TU_NUMBER`/`SSCC`. In n:n sorting, new Outbound `TU` objects are created with numbers from the type sequence, and one Outbound `TU` may collect SKU from several Inbound `TU` objects under `SP14`/`R20` criteria.
   **Rationale:** retaining the number is correct only when the physical unit remains unsplit; n:n sorting creates new business units.
   **Rejected alternatives:** retaining source `TU_NUMBER`/`SSCC` after n:n sorting or assigning a new number also in full 1:1.
   **Date:** 2026-08-10.
   **Rules:** `R72`.
4. **DEC-L05 — `OutboundOrderLine` as sorting instruction and lock:**
   **Decision:** `OutboundOrderLine` is created in `CREATED` before sorting, points to the target Outbound `TU` and prevents reuse of the same quantity; no `Allocation` is created.
   **Rationale:** cross-dock execution needs an instruction before Packer work, but the quantity does not come from `Inventory` and cannot be reserved by `Allocation`.
   **Rejected alternatives:** creating `OutboundOrder` before picking as in P1 — infeasible because `SP2` requires `Allocation`, which cross-dock does not have.
   **Date:** 2026-08-10.
   **Rules:** `R71`, `R73`, `SP18`.
5. Missing goods, `DAMAGED`, or an empty Inbound `TU` cancel the corresponding `OutboundOrderLine` objects and return `CustomerOrderLine` to `BACKORDERED`; unexpected SKU and `DAMAGED` go to `QC`. On completion of full 1:1, Inbound `TU` transitions to `CROSS_DOCKED`; with remainder or n:n, the remainder goes to `TRANSIT` and Inbound `TU` to `IN_PUTAWAY`. In both cases the Outbound `TU` enters `PACKING_SEALED` directly and `OutboundOrderLine` transitions to `PACKED`.
6. **DEC-L06 — version-1 ERP gate:**
   [HISTORICAL — superseded by DEC-L23/DEC-L24] The “version 1” gate (waiting for the whole ASN to be `POSTED`) and deferring GR split to B11 are obsolete. Current behavior: `Shipment` waits only for `GR_ACCEPTED` of the cross-dock settlement of every source Inbound `TU` (`DEC-L23`), and GR settlement is already split into cross-dock/putaway parts (`DEC-L24`–`DEC-L26`, closes B11).
   **Decision:** a cross-dock `Shipment` is sent to ERP only after the entire source ASN reaches `POSTED`; until then `Shipment` remains in `POSTING_PENDING`. Splitting GR into cross-docked and putaway parts was deferred to `BACKLOG.md` B11.
   **Rationale:** the current Inbound model posts GR for the entire ASN and does not expose a separate confirmation of cross-docked quantity.
   **Rejected alternatives:** sending `Shipment` before GR is posted or treating a nonexistent partial GR as a version-1 fact.
   **Date:** 2026-08-10.
   **Rules:** `R54` (extension).
7. **DEC-L07 — cross-dock quantity outside the ATP model:**
   **Decision:** quantity handled by `CrossDockPickTask` never creates an `Inventory` record, therefore it does not enter the `ATP` pool and requires no new ATP rule.
   **Rationale:** goods pass directly from Inbound `TU` to Outbound `TU` without a storage stage or availability as stock.
   **Rejected alternatives:** temporarily creating `Inventory`/`ATPReservation` solely for cross-docking.
   **Date:** 2026-08-10.
   **Rules:** no new rule — cross-dock quantity never creates `Inventory`.
   Exception: see `DEC-L21` (recovery path after demand loss).
8. **DEC-L08 — task precedence:**
   [HISTORICAL — withdrawn by `DEC-L41` (2026-08-23)] The operator-pool concept referenced by this decision was withdrawn in full. Current mechanism: the operator chooses a work module on the RF terminal, see `DEC-L41`–`DEC-L45`.
   **Decision:** an operator assigned simultaneously to the `CrossDockPickTask` and `PickTask` pools receives `CrossDockPickTask` with precedence. The full operator-pool configuration mechanism was deferred to `BACKLOG.md` B10.
   **Rationale:** cross-docking has a short operating window and loses value when the task waits behind standard picking.
   **Rejected alternatives:** equal priority of both pools or full design of the pool mechanism within P2.
   **Date:** 2026-08-10.
   **Rules:** `R75`.
9. After `Shipment` is created, the process uses Carrier Selection, label, ERP gate, `CarrierManifest` and dispatch as in P1.
10. **DEC-L29 — moment Outbound `TU` is created in cross-docking:**
   **Decision:** the moment Outbound `TU` is created in cross-docking was clarified; the full current description is in `proces_2_outbound_crossdock.md` `R7` and STEP 2. This clarifies `DEC-L03`: “task settlement” in that entry was imprecise — see `R7` for the exact moment.
   **Rationale:** without a previously existing object there is nothing to hold placed quantity or `TU_NUMBER` to scan when confirming repacking of `SKU` (§3.2 step 2) — and `DEC-L16` already assumes that the operator closes and opens several Outbound `TU` objects during one still-active `CrossDockPickTask`, which requires the object to exist before task settlement.
   **Rejected alternatives:** creation only at `PACKING_SEALED` (previous §4.7 wording) — rejected because it does not explain what the operator scans/fills during the task; creation only on settlement of the whole task (literal wording of `DEC-L03`) — rejected for the same reason and additionally infeasible with multiple Outbound `TU` per task (`DEC-L16`).
   **Date:** 2026-08-15.
   **Rules:** `propozycja_procesow_outbound.md` — §4.7, §3.2 step 1 (clarification).
11. **DEC-L30 — terminology alignment of `sourceEligibleQty`/`R77`: “declared (ASN)” instead of “physically identified”:**
   **Decision:** quantity-base terminology in `R76`, `R77` and §3.2 step 3 was changed from “physically identified quantity” to “declared (ASN) quantity”, according to D42 (`../MEMORY.md`); the full current description is in `proces_2_outbound_crossdock.md` `R6` and `R21`. Naming change only — value and formula unchanged.
   **Rationale:** D42 explicitly defines this quantity as “not physically verified by Inbound” (Inbound does not open the `TU` before handoff to cross-docking) — the previous name contradicted the established Inbound-Outbound contract.
   **Rejected alternatives:** retaining “physically identified” — rejected as misleading terminology conflicting with D42.
   **Date:** 2026-08-15.
   **Rules:** `propozycja_procesow_outbound.md` — `R76`, `R77` (terminology revision); in this file `DEC-L18`, `DEC-L20`, `DEC-L24`, `DEC-L28` (terminology, see Task 17).
12. **DEC-L31 — empty Outbound `TU` after full `PutBackTask` in cross-docking:**
   [HISTORICAL — extended by `DEC-L46` (2026-08-23)] The guard below is quantity-only: it predates `DEC-L36`, which allowed one target Outbound `TU` to be fed by several tasks, so it does not account for another active or planned task still targeting that `TU`. Current full cancellation condition: `DEC-L46`.
   **Decision:** when an Outbound `TU` created during an active `CrossDockPickTask` (`DEC-L29`) loses, through `PutBackTask` recovery (`DEC-L21`), all quantity confirmed so far and remains empty with no other confirmed item, it transitions `CREATED`→`CANCELLED`; it never enters `PACKING_SEALED`.
   **Rationale:** the object exists from the first placement of `SKU` (`DEC-L29`), so an empty Outbound `TU` after full recovery is possible and requires an explicit end of lifecycle, analogously to the existing `CREATED`→`CANCELLED` edge in standard flow (put-back/VOID).
   **Rejected alternatives:** leaving an empty Outbound `TU` without formal state closure — rejected because the object would remain permanently suspended in `CREATED` with no exit path.
   **Date:** 2026-08-17.
   **Rules:** `propozycja_procesow_outbound.md` — new `R84`, §4.7 (diagram, transition table).
13. **DEC-L32 — `grAcceptanceStatus` updated system-wide, not per `Shipment`:**
   **Decision:** after a GR result is recorded for an Inbound `TU`, the WMS System sets the same `grAcceptanceStatus` on all `CrossDockPickTask` objects in the system whose `sourceInboundTU` matches that Inbound `TU` — regardless of which `Shipment` each task contributed `confirmedQty` to, including when one Inbound `TU` supplied tasks belonging simultaneously to several different `Shipment` objects (n:n case). Gate condition `R54` is still calculated separately per `Shipment`, from the complete set of its own source Inbound `TU` objects.
   **Rationale:** limiting the update only to tasks feeding one currently checked `Shipment` would require repeated, independent processing of the same GR signal for every later `Shipment` fed by the same Inbound `TU` — a system-wide update after one ERP confirmation is more consistent operationally and removes the risk of divergent `grAcceptanceStatus` values among tasks for the same Inbound `TU`.
   **Rejected alternatives:** updating `grAcceptanceStatus` only for tasks feeding the `Shipment` whose `R54` gate is currently being checked — considered and rejected (risk of repeated processing of the same signal, inconsistent `grAcceptanceStatus` across tasks for the same Inbound `TU` in different `Shipment` objects).
   **Date:** 2026-08-17.
   **Rules:** `propozycja_procesow_outbound.md` — `R81` (update-scope revision).
14. **DEC-L36 — triggers for closing a cross-dock Outbound `TU`:**
   [HISTORICAL — superseded by `DEC-L39` (2026-08-22)] The criterion below (absence of active/planned tasks alone) is incomplete — it lacks the requirement to reach `slaDeadline`; see `DEC-L39` for the current full criterion.
   **Decision:** target Outbound `TU` transitions to `PACKING_SEALED` in two cases: when the Packer closes it by RF scan as physically full during an active `CrossDockPickTask`, or when no active or planned `CrossDockPickTask` points to it as a target. Completion of one task does not by itself close the `TU`.
   **Rationale:** `DEC-L04` allows one Outbound `TU` to be fed by tasks from several source Inbound `TU` objects, and task generation routes it to a new or already open Outbound `TU`. Closing the `TU` when each task is settled would invalidate the concept of an open `TU` and close it before the next task could add goods. Previous text described only the reverse case — one task filling several `TU` objects in sequence (`DEC-L16`).
   **Rejected alternatives:** closing only by Packer decision, also for an incomplete `TU` — rejected because an incomplete `TU` could remain open indefinitely; narrowing `DEC-L04` to one task per one target `TU` — rejected because it loses consolidation from several inbound pallets into one package.
   **Date:** 2026-08-22.
   **Rules:** `proces_2_outbound_crossdock.md` — `R10` (extension); `model_stanow_outbound.md` §7 (two `CREATED→PACKING_SEALED` transition rows).
15. **DEC-L37 — source Inbound `TU` finalization takes precedence over new demand:**
   **Decision:** source Inbound `TU` is finalized when none of its active `CrossDockPickTask` objects remain, and this ends that `TU`'s participation in cross-docking. Demand submitted after finalization does not create another cross-dock task for that `TU` — it is handled through standard inventory after Putaway is complete.
   **Rationale:** `sourceEligibleQty` still shows residual quantity as available to plan and matching against `CustomerOrderLine BACKORDERED` runs cyclically — without this rule there is a race between finalization and generation of another task. The deterministic variant is consistent with literal `DEC-L18`, where the criterion is absence of residual quantity, not absence of demand.
   **Rejected alternatives:** giving new demand precedence while the pallet physically remains in the cross-dock area — rejected because the pallet would block the area and delay GR settlement; a time window as a warehouse parameter — rejected because it introduces a new configuration parameter and waiting state without a real benefit.
   **Date:** 2026-08-22.
   **Rules:** `proces_2_outbound_crossdock.md` — `R28` (new).
16. **DEC-L38 — one-time closed generation of `CrossDockPickTask`/`OutboundOrderLine` (narrowing `DEC-L19`):**
   [NARROWED — see `DEC-L58` (2026-08-26)] The prohibition on “updating required quantity of `OutboundOrderLine` after creation” applies to automatic generation of additional tasks and automatic quantity updates; explicit supervised correction by `Warehouse Supervisor` in the `P2 R15` correction branch is permitted.
   **Decision:** generation of `CrossDockPickTask` and `OutboundOrderLine` for a given source Inbound `TU` is a one-time event triggered by its transition to `IN_CROSS_DOCK`, matching only `CustomerOrderLine BACKORDERED` objects that exist at that moment. There is no mechanism to generate further `CrossDockPickTask` objects for an already created `OutboundOrderLine` or to update its required quantity after creation — including in the window between `IN_CROSS_DOCK` and finalization of the same source `TU` (`DEC-L37`). Each cross-dock `OutboundOrderLine` is created together with exactly one `CrossDockPickTask`. The quantity lock remains on `CrossDockPickTask` (`DEC-L19`, unchanged) — only the statement that one `OutboundOrderLine` aggregates multiple tasks from different source `TU` objects is narrowed.
   **Rationale:** literal `DEC-L19` allowed further tasks to be generated for an existing `OutboundOrderLine`, which conflicts with the one-time event nature of generation described in §3.2 step 1 and with `DEC-L37` (finalization ends participation of the source `TU`, new demand follows the standard path). A closed generation scope removes ambiguity around when exactly an `OutboundOrderLine` is “complete”.
   **Rejected alternatives:** retaining an open additional-generation mechanism — rejected because there is no triggering event beyond `IN_CROSS_DOCK` and no criterion for when generation should stop.
   **Date:** 2026-08-22.
   **Rules:** `proces_2_outbound_crossdock.md` — `R30` (clarification).
17. **DEC-L39 — automatic close of target Outbound `TU` after `slaDeadline` (extension of `DEC-L36`):**
   **Decision:** target Outbound `TU` transitions to `PACKING_SEALED` in one of two cases: (1) the Packer closes it by RF scan as physically full during an active `CrossDockPickTask` (`DEC-L36`, unchanged); (2) the WMS System closes it automatically when its `slaDeadline` is reached (or passed) and no active or planned `CrossDockPickTask` points to it as target. Before `slaDeadline`, the mere absence of such tasks does not close the `TU` — cross-dock goods for the same customer/address/`priority`/`slaDeadline` (`DEC-L04`) may still arrive in another Inbound delivery on the same day or later days.
   **Rationale:** the previous `DEC-L36` criterion (absence of active/planned tasks alone) would close the `TU` too early, before another compatible delivery could join it, reducing `TU` utilization and increasing small shipments. `slaDeadline` defines a hard waiting boundary already present in the model (`DEC-L10` — analogous mechanism for `Shipment`).
   **Rejected alternatives:** keeping only the `DEC-L36` criterion — rejected for the reason above; a new separate time parameter instead of reusing `slaDeadline` — rejected as unnecessary complexity.
   **Date:** 2026-08-22.
   **Rules:** `proces_2_outbound_crossdock.md` — `R10` (extension), `R3` (tightened to identical `slaDeadline`).
18. **DEC-L40 — re-evaluate the GR gate per message, `GR_REJECTED` does not cause `POSTING_ERROR`, Supervisor visibility (narrowing `DEC-L23`):**
   **Decision:** the gate for reporting `Shipment` to ERP (`DEC-L23`) is re-evaluated on every GR message (`GR_ACCEPTED` or `GR_REJECTED`) concerning any required source Inbound `TU`, regardless of the previous result for that `TU` or the current `Shipment` state — including when `Shipment` is in `POSTING_ERROR` for another reason (e.g. ERP rejection unrelated to GR, `DEC-L22`). Explicit `GR_REJECTED` does not by itself move `Shipment` to `POSTING_ERROR` — the gate remains unsatisfied; if `GR_ACCEPTED` later arrives for the same `TU`, the gate is satisfied like any other, with no separate recovery path. `grAcceptanceStatus` of source `TU` objects blocking the gate is visible to `Warehouse Supervisor`, without a new state, object or escalation mechanism.
   **Rationale:** the sentence in `DEC-L23` treating `GR_REJECTED` as a permanent explicit rejection analogous to ERP rejection (`DEC-L22`) confused an unsatisfied gate precondition with an actual posting error — ERP had not yet seen the `Shipment`, so it could not have rejected it. Supervisor visibility provides operational insight without a new escalation mechanism.
   **Rejected alternatives:** keeping `GR_REJECTED → POSTING_ERROR` with manual Supervisor retry — rejected because it confuses an unsatisfied precondition with a posting failure; a new separate `Shipment` state for waiting after GR rejection — rejected as unnecessary.
   **Date:** 2026-08-22.
   **Rules:** `proces_2_outbound_crossdock.md` — `R35`, `R36` (new).

19. **DEC-L46 — cancellation guard for an empty target Outbound `TU` and target-continuity invariant (extension of `DEC-L31`):**
   **Decision:** cancellation of an empty target Outbound `TU` (`DEC-L31`) requires, in addition to the existing quantity condition, that no active or planned `CrossDockPickTask` still points to that `TU` as target; while such a task points to it, the `TU` remains in `CREATED` as an empty open target unit. Cancellation does not wait for `slaDeadline` — the asymmetry versus closing into `PACKING_SEALED` (`DEC-L39`) is intentional and must be stated explicitly in the rule so a later audit does not report it as inconsistency. Independently, the WMS System guarantees that an active `CrossDockPickTask` always has an available target: if, at the moment of placing the next item, there is no open target Outbound `TU` for that task — regardless of why it is missing — the WMS System opens a new one and assigns its `TU_NUMBER`, lazily at that exact `SKU` placement (`DEC-L29`), not in advance.
   **Rationale:** `DEC-L31` is not wrong — it became incomplete in light of a later fact. It was created on 2026-08-17, while `DEC-L36` (2026-08-22) allowed one target Outbound `TU` to be fed by several tasks from different source Inbound `TU` objects. A quantity-only guard therefore became incomplete: `PutBackTask` recovery could cancel a `TU` still targeted by another active task, leaving that task pointing at a terminal object (dangling target reference), and no retargeting mechanism exists in any active file. This follows the same pattern by which `DEC-L39` extended `DEC-L36` — extension of an earlier decision, not reversal. The second part generalizes a mechanism that previously existed only in one `DEC-L36`/`R10` branch (the Packer closes a physically full `TU` and continues into a new one) into an invariant independent of why the open `TU` is missing — therefore no state exists in which an active task has nowhere to place confirmed quantity.
   **Rejected alternatives:** waiting to cancel until `slaDeadline` for full symmetry with `DEC-L39` — considered and rejected by the owner: an empty `TU` holds nothing and the physical container is freed faster in the cross-dock area; the consciously accepted cost is loss of the assigned `TU_NUMBER` (numbers return to circulation only after `Sequence` exhaustion, `DEC-G16`) and the need to open a new `TU` if another compatible delivery arrives before the deadline. The task-only condition without the target-continuity invariant — rejected because it does not cover a case where an active task loses its target for a reason other than cancellation of an empty `TU`.
   **Date:** 2026-08-23.
   **Rules:** `proces_2_outbound_crossdock.md` — `R34` (revision), `R40` (new).

20. **DEC-L47 — Inbound qualification is a transport prerequisite, not a binding assignment:**
   **Decision:** qualification of a `TU` for cross-docking by Inbound Process 2 at `IN_BUFFER` is a prerequisite for transport to the cross-dock area, not a binding assignment of goods to specific demand. Binding matching to `CustomerOrderLine` `BACKORDERED` is created only when the source `TU` transitions to `IN_CROSS_DOCK` (`R30`), using the priority queue current at that moment. The timing divergence between the two matches, equal to physical transport time, is consciously accepted. If all matching demand disappears during that window, no `CrossDockPickTask` is created, and the `TU` is immediately finalized with the entire declared quantity as residual and transitions to `IN_PUTAWAY`. It is also recorded as a boundary invariant that only `TU` `ELEMENTARY` enters this process.
   **Rationale:** matching at `IN_CROSS_DOCK` works on fresher data than Inbound qualification, so the time gap improves allocation accuracy rather than harming it. Its only cost is a possible empty transport run — cheaper than reserving demand during transport. The zero-match path previously existed only as a degenerate reading of `R21` and `R28` (empty set of related tasks), without its own rule, requirement and scenario. The `ELEMENTARY` invariant was previously only a qualifier in prose, not a rule.
   **Rejected alternatives:** reserving `CustomerOrderLine` during transport — rejected because it blocks demand without physical coverage; generating `CrossDockPickTask` after `IN_CROSS_DOCK` — explicitly excluded by `R30`; leaving the zero-match path undocumented as a degenerate reading of `R21`/`R28` — rejected because a future audit would read it as a gap.
   **Date:** 2026-08-23.
   **Rules:** `proces_2_outbound_crossdock.md` — `R41` (new), `R42` (new).

21. **DEC-L48 — separate source-`TU` finalization from assignment of `IN_PUTAWAY` (clarification of `DEC-L47`):**
   **Decision:** finalization of the source Inbound `TU` in this process and assignment of its `IN_PUTAWAY` state are two different events in two different processes. Finalization with zero residual quantity assigns `CROSS_DOCKED` and is an event of this process. Finalization with non-zero residual quantity is only a handoff notification to Inbound Process 3 (Putaway); Inbound assigns `IN_PUTAWAY` when its own transport of the `TU` to the sector `TRANSIT` is completed, and until then the `TU` remains in `IN_CROSS_DOCK`.
   **Rationale:** `R28` previously equated finalization directly with transition to `CROSS_DOCKED` or `IN_PUTAWAY`, and `R21` and `R42` repeated the shortcut. After the Inbound transport-model change, `IN_PUTAWAY` is assigned when transport is completed, not at handoff — read literally, the previous content meant a `TU` in `IN_PUTAWAY` physically standing in the cross-dock area, conflicting with the Inbound canon. The discrepancy was reported by the parallel Inbound session; it was corrected in the normative layer rather than by adding an exception sentence beside existing rules, so the file would not describe two mechanisms simultaneously.
   **Rejected alternatives:** correcting only `R42` while leaving `R21` and `R28` unchanged — rejected because it would create an internal contradiction in one file; moving the timing of `IN_PUTAWAY` assignment into prose instead of rules — rejected because prose summarizes rules and is not the normative layer; describing the Inbound transport task in this process — rejected because it crosses the process boundary and duplicates the Inbound canon.
    **Date:** 2026-08-23.
    **Rules:** `proces_2_outbound_crossdock.md` — `R21` (revision), `R28` (revision), `R42` (revision).

22. **DEC-L52 — definition of full 1:1 match in cross-docking:**
   **Decision:** full 1:1 match means destination homogeneity, not content homogeneity: the source Inbound `TU` may contain multiple `SKU`. Full match occurs when the entire declared (ASN) content of the source Inbound `TU` covers `CustomerOrderLine` demand in `BACKORDERED` that jointly satisfies the common-dispatch conditions: same customer, same delivery address, compatible `priority` and identical `slaDeadline`. With `allowPartialShipment = true`, demand may come from several `CustomerOrder` objects of the same customer; with `allowPartialShipment = false` — from one only.
   **Rationale:** the 1:1 criterion in the body used the term “`TU` content homogeneity rule”, which the body never defines. Content homogeneity is also the wrong criterion — common dispatch depends on destination, not SKU composition. Contextual finding: Inbound qualifies a `TU` for cross-docking only by `SKU` and quantity, so homogeneity is entirely an Outbound criterion and requires no Inbound change.
   **Rejected alternatives:** none.
   **Date:** 2026-08-25.
   **Rules:** `P2 R2`.

23. **DEC-L54 — `priority` and `slaDeadline` of a cross-dock `OutboundOrder`:**
   **Decision:** a cross-dock `OutboundOrder` inherits `priority` and `slaDeadline` from its parent `CustomerOrder`. Grouping lines into one cross-dock `OutboundOrder` is allowed only for compatible customer, delivery address, `priority` and identical `slaDeadline`. For `allowPartialShipment = false` — only lines of one `CustomerOrder`.
   **Rationale:** the rule establishing the source of `priority`/`slaDeadline` applied to both fulfillment channels, while the body had an equivalent only for the standard channel. Both attributes are operative in cross-docking: `CrossDockPickTask` assignment orders by `slaDeadline` and `priority`, automatic close of target Outbound `TU` refers to its `slaDeadline`, and attaching Packing `TU` to `Shipment` requires identical `slaDeadline`. Without inheritance from the same `CustomerOrder`, the two halves of an order with `allowPartialShipment = false` could have different values and fail to enter one `Shipment`.
   **Rejected alternatives:** an aggregate like the standard channel, taking the most urgent values among grouped orders — rejected because cross-docking already requires identical `slaDeadline` at line level, so the aggregate would resolve a nonexistent tie. Leaving the boundary undefined — rejected because rules depending on these attributes would have no defined value source.
   **Date:** 2026-08-25.
   **Rules:** `P2 R43`.

24. **DEC-L55 — `sourceInboundTU` does not return as an Outbound `TU` attribute:**
   **Decision:** attribute `sourceInboundTU` is not added to Outbound `TU`. Provenance remains where it is today — on `CrossDockPickTask`. The archived clause of a rule requiring this reference is consciously not reintroduced; the item is closed by this decision, not by restoring text into the body.
   **Rationale:** GR completeness checking per source Inbound `TU` does not require this attribute. `CrossDockPickTask` carries both `sourceInboundTU` and `grAcceptanceStatus`, and `DEC-L32` sets the same `grAcceptanceStatus` on all tasks with a given `sourceInboundTU` — system-wide, independent of `Shipment`, also in n:n topology. Aggregating tasks by `sourceInboundTU` gives the complete answer in one step, and `P2 R36` exposes this field to `Warehouse Supervisor`. Traversal through `targetOutboundOrderLine` is unnecessary for this purpose. `DEC-L31` and `DEC-L32` are rationale for this decision, never evidence of earlier supersession — neither decided whether the attribute belongs on Outbound `TU`.
   **What this decision does not close:** the input to the `P2 R25`/`R33` gate remains underspecified. Determining the set of source Inbound `TU` objects for a `Shipment` requires traversing `Shipment` → Outbound `TU` → contents → `OutboundOrderLine` → `CrossDockPickTask` → `sourceInboundTU`, while the relation between Outbound `TU` contents and `OutboundOrderLine` does not exist in the data catalog. The same gap affects `P1 R27`, `P1 R58` and `P1 R60`. The issue is moved to a follow-up item in `WMSAI_OUTBOUND/BACKLOG.md`.
   **Rejected alternatives:** none.
   **Date:** 2026-08-26.
   **Rules:** no rules — decision not to introduce a change.

25. **DEC-L58 — narrowing `DEC-L38`: explicit Supervisor correction is not covered by the quantity-update prohibition:**
   **Decision:** the prohibition from `DEC-L38`, worded as “there is no mechanism to generate further `CrossDockPickTask` objects for an already created `OutboundOrderLine` nor update its required quantity after creation”, applies to automatic task generation and automatic quantity update, not to explicit supervised correction by `Warehouse Supervisor` in the `P2 R15` correction branch. Correction of required quantity on a cross-dock `OutboundOrderLine`, made together with correction of `CustomerOrderLine.Quantity` by `Warehouse Supervisor`, is permitted. The remaining content of `DEC-L38` stays unchanged.
   **Rationale:** `DEC-L38` was created to close an open additional-task generation mechanism for which there was no triggering event or stop criterion. Explicit Supervisor correction has both: the trigger is the Supervisor's decision and the scope is that one line. `P2 R15` already provides for continuation of the same line to `PACKING_SEALED` after correction. Without this narrowing, correctability of required quantity in the `P2 R15` branch would conflict with the active register.
   **Rejected alternatives:** none.
   **Date:** 2026-08-26.
   **Rules:** `P2 R15`, `P2 R30`, `P1 R69`.

26. **DEC-L63 — channel exclusivity: `demandEligibleQty` subtracts both forms of demand protection:**
   **Decision:** `demandEligibleQty` for `CustomerOrderLine` = `Quantity` minus `ATPReservation`, minus the sum of `requiredQty` of all its `OutboundOrderLine` objects that are not `CANCELLED`, regardless of channel. Replaces the `demandEligibleQty` term from `DEC-L20`. `sourceEligibleQty` is unchanged.
   **Rationale:** `P1 R11` moves quantity from soft to hard reservation, causing the previous formula — which subtracted only `ATPReservation` — to stop seeing quantity already promised. With `Quantity` 100, `ATPReservation` 0 and a standard `OutboundOrderLine` with `requiredQty` 30, it returned 100 instead of 70, assigning thirty units a second time through cross-dock. Subtracting both forms closes the gap without a new attribute. It also removes the undefined body phrase “missing quantity of `CustomerOrderLine` in `BACKORDERED`” and the separate term for assignments of other `CrossDockPickTask` objects, absorbed by the `requiredQty` sum — a cross-dock `OutboundOrderLine` is created together with its task (`P2 R30`).
   **Rejected alternatives:** subtracting the sum of `reservedQty` of allocations in `SHORT`/`RESERVED`/`CONFIRMED` instead of `requiredQty` — rejected because an allocation in `SHORT` holds less than the line promises, so cross-dock would take quantity the standard channel is supposed to recover after replenishment; it would be the same error in the opposite direction. Adding an `Allocation` term alongside existing `ATPReservation` — rejected because it would retain two measures of the same protection.
   **Date:** 2026-08-28.
   **Rules:** `P2 R6`.

### 6.2a Shipment — order of `READY_FOR_DISPATCH` and cancellation boundary

1. **DEC-L09 — `READY_FOR_DISPATCH` before Carrier Selection:**
   **Decision:** `Shipment.READY_FOR_DISPATCH` is moved before Carrier Selection (instead of after `LABEL_GENERATED`) and means closure of `TU` grouping in that `Shipment`. Carrier Selection starts only after this state is reached.
   **Rationale:** R61 calculates `maxWeight`/`maxVolume` from the Packing `TU` objects of that `Shipment` — until the `TU` set is closed, the result may become stale if another `TU` arrives after Carrier Selection starts.
   **Rejected alternatives:** leaving `READY_FOR_DISPATCH` after `LABEL_GENERATED` (current version at the time) — rejected because it does not guarantee a complete `TU` set before weight/volume calculation; renaming `POSTED` to `READY_FOR_DISPATCH` — considered and withdrawn during discussion, `POSTED` keeps its name and meaning.
   **Date:** 2026-08-11.
   **Rules:** SP8, R61 (extension), §4.9.
2. **DEC-L10 — `Shipment` consolidation requires identical `slaDeadline`, closes by timeout:**
   **Decision:** consolidation of multiple `OutboundOrder` objects in one `Shipment` (SP14/R20) requires identical `slaDeadline` for all contributing `OutboundOrder` objects, with no tolerance. `Shipment.READY_FOR_DISPATCH` is also reached automatically when that common `slaDeadline` expires, even if not all contributing `OutboundOrder` objects have finished packing — those that did not finish create a new `Shipment`.
   **Rationale:** without a common `slaDeadline` and timeout mechanism, a `Shipment` could wait indefinitely for a slower `OutboundOrder`, blocking dispatch of those already ready.
   **Rejected alternatives:** `slaDeadline` tolerance as in R1 — rejected because this is closing the dispatch window, not planning; waiting with no time limit — rejected as operational risk.
   **Date:** 2026-08-11.
   **Rules:** SP8, SP14 (extension).
3. **DEC-L11 — `Shipment` cancellation boundary after `POSTING_PENDING`:**
   **Decision:** `Shipment` may be cancelled only before entering `POSTING_PENDING` (from `CREATED`/`READY_FOR_DISPATCH`/`CARRIER_SELECTED`/`OWN_TRANSPORT`/`CARRIER_PENDING`/`LABEL_GENERATED`) or from `POSTING_ERROR`. From `POSTING_PENDING` onward, WMS cancellation (including the R65 `PACKED` branch) is impossible; further handling is an ordinary goods return (Return Receipt) within a future post-dispatch process, not correction of the `Shipment` document in WMS.
   **Rationale:** ERP is the ordering system (see R54/step 11a rationale) — after reporting to ERP, correction should follow an analogous path to other post-dispatch events, not WMS's internal cancellation mechanism.
   **Rejected alternatives:** correcting the `Shipment` document directly in ERP with a new integration channel to WMS — rejected at this stage as premature; the mechanism will be designed within a future Return Receipt process.
   **Date:** 2026-08-11.
   **Rules:** R55, R65 (narrowing of the `PACKED` branch).
4. **DEC-L12 — `OutboundOrder.PACKED` as full aggregate:**
   **Decision:** `OutboundOrder.PACKED` means full aggregate — all its `OutboundOrderLine` objects and all their `TU` objects are packed, consistent with SP3 (`PICKED`). `PACKED`→`READY_FOR_DISPATCH` is immediate, with no additional condition, and serves as a signal consumed by `Shipment` (SP8).
   **Rationale:** consistency with the existing `OutboundOrder` aggregation pattern (SP2–SP4) — the aggregate state reflects complete closure, not partial progress.
   **Rejected alternatives:** `PACKED` at the first ready `TU` (partial progress) — rejected as inconsistent with SP3 and misleading to consumers of the status.
   **Date:** 2026-08-11.
   **Rules:** §4.3, SP8 (dependency).
5. **DEC-L22 — `POSTING_ERROR` from content discrepancy, two repair paths:**
   **Decision:** when `Shipment POSTING_ERROR` results from a content-data discrepancy (structured ERP rejection reason distinguishable from a technical communication failure), the cause is on one of two sides: (a) ERP side (e.g. missing item price, ATP-state divergence between WMS and ERP) — retry with unchanged content succeeds after repair in ERP; (b) WMS side (incorrect `CustomerOrder`/`Shipment` data) — correction precedes retry, entered manually by `Warehouse Supervisor` or by OMS/ERP through webservice (as in `R65`). In both cases, re-submission is a separate manual `Warehouse Supervisor` decision (`POSTING_ERROR→POSTING_PENDING`, `R54`); physical contents of the packed `TU` and the `OutboundOrderLine`/`PACKED` state are untouched in either variant.
   **Rationale:** ERP never sees the physical contents of a `TU` — every rejection is by definition a WMS↔ERP data discrepancy, not a finding of physical packing error; distinguishing the responsible side (ERP vs WMS) determines whether retry requires prior data correction.
   **Rejected alternatives:** modeling a separate path for a “physical discrepancy” detected at `POSTING_ERROR` — rejected because this gate structurally cannot detect such a discrepancy (it would require physically opening a sealed `TU`, outside the scope of this mechanism).
   **Date:** 2026-08-12.
   **Rules:** `propozycja_procesow_outbound.md` — `R80`, §3.1 step 11a, §4.9.

6. **DEC-L23 — GR gate per Inbound `TU` for a cross-dock `Shipment` (ICR-05):**
   [HISTORICAL — clarified by `R81`/`DEC-L25` (2026-08-14)] The sentence in “Decision” below listing a “`CrossDockPickTask` identifier” as required GR-signal content is obsolete — `R81` correlates the signal only by `sourceInboundTU`/`GR_SETTLEMENT_SOURCE`, without task identifier (see `R81`).
   [HISTORICAL — narrowed by `DEC-L40` (2026-08-22)] The sentence “`GR_REJECTED` for any required Inbound `TU` moves `Shipment` to `POSTING_ERROR` as an explicit rejection in the existing gate; retry remains a manual `Warehouse Supervisor` decision” in “Decision” below is obsolete — the gate is re-evaluated per GR message and `GR_REJECTED` does not independently cause `POSTING_ERROR`, see `DEC-L40`.
   **Decision:** the ERP gate of a cross-dock `Shipment` stops waiting for the whole source ASN to be `POSTED`. A `Shipment` in `POSTING_PENDING` sends `POST` only after `GR_ACCEPTED` has been recorded for every source Inbound `TU` that, through a `CrossDockPickTask`, contributed `confirmedQty` to any Outbound `TU` of that `Shipment`. In n:n, GR acceptance of one directly linked `CrossDockPickTask` is insufficient: `Shipment` waits for the complete set of source Inbound `TU` objects contributing quantity. WMS stores `grAcceptanceStatus` on `CrossDockPickTask` with values `GR_PENDING`, `GR_ACCEPTED` and `GR_REJECTED`. The signal consumed by Outbound must indicate: Inbound `TU` identifier, `CrossDockPickTask` identifier and acceptance/rejection result. A mismatch between the indicated Inbound `TU` and the task's `sourceInboundTU` is rejected as inconsistent. `GR_REJECTED` for any required Inbound `TU` moves `Shipment` to `POSTING_ERROR` as an explicit rejection in the existing gate; retry remains a manual `Warehouse Supervisor` decision.

   **Rationale:** an elementary Inbound `TU` settled by cross-docking has an independent GR result in ERP, so waiting for Putaway and GR of the remaining `TU` objects in the same ASN no longer reflects the state of quantity shipped by `Shipment`. Requiring the complete source-`TU` set preserves accounting control also when sorting combines multiple sources into one Outbound `TU` or splits one source across multiple Outbound `TU` objects.

   **Rejected alternatives:** waiting for the whole ASN to be `POSTED` — rejected because it eliminates the cross-dock timing advantage; releasing `Shipment` after `GR_ACCEPTED` for only one Inbound `TU`/one `CrossDockPickTask` — rejected because in n:n it permits `POST` before GR acceptance for part of the physical `Shipment` contents; releasing after `CrossDockPickTask` completion alone — rejected because that does not confirm ERP GR acceptance.

   **Date:** 2026-08-12.

   **Rules:** `propozycja_procesow_outbound.md` — `R54`, `R81`, §4.13 (`CrossDockPickTask.grAcceptanceStatus`).

7. **DEC-L24 — two-stage GR for an Inbound `TU` partially settled by cross-docking:**
   **Decision:** when an Inbound `TU` settled by cross-docking leaves residual quantity and transitions to `IN_PUTAWAY` (`R77`, unchanged), GR settlement of that `TU` occurs in two independent steps: cross-dock settlement (quantity already confirmed by completed `CrossDockPickTask`, at transition from `IN_CROSS_DOCK`) and putaway settlement (residual quantity, when standard Putaway of the same `TU` is completed). Together the two settlements cover the entire declared (ASN) `OK` quantity of that `TU`, with no gap or overlap. The `Shipment` gate (`R54`) waits only for the cross-dock settlement result.
   **Rationale:** forcing full physical sorting of the `TU` before any GR (an earlier considered variant with a new logistics unit for the remainder) solves the same SLA problem at the cost of extra operator work at the cross-dock station and a new model object. Splitting the GR settlement itself into two independent steps for the same Inbound `TU` achieves the same effect without changing the physical sorting process — `Warehouse Operator` works exactly as today. The `PZ` document already aggregates many `TU` objects into one document for an `ASN` (Inbound); extending it with another settlement version for the same `TU` is the same aggregation mechanism, with more messages but no new integration logic.
   **Rejected alternatives:** fully physically sorting the source `TU` before each GR with a new logistics unit (`InternalTU`) for residual quantity — rejected as heavier both operationally (extra `Warehouse Operator` work at cross-dock) and architecturally (new object, new `TU_ID` identity in Inbound) with no advantage over splitting only the GR settlement.
   **Date:** 2026-08-13.
   **Rules:** `propozycja_procesow_outbound.md` — `R54` (clarification), §3.2 step 3/4.
8. **DEC-L25 — GR signal contract with settlement discriminator:**
   **Decision:** the `GR_ACCEPTED`/`GR_REJECTED` signal consumed by Outbound must unambiguously indicate that it concerns the cross-dock settlement of the given Inbound `TU`, not its possible later putaway settlement. The WMS System reacts only to a signal concerning cross-dock settlement; a signal concerning putaway settlement of the same `TU` is not associated with any `CrossDockPickTask` and does not change `grAcceptanceStatus`.
   **Rationale:** without this distinction, the WMS System could not distinguish two independent GR results for the same Inbound `TU` — risking association of a putaway signal with a task that had long since contributed to a dispatched `Shipment`.
   **Rejected alternatives:** relying on signal arrival order (first received = cross-dock) — rejected as unsafe because message delivery order is not guaranteed.
   **Date:** 2026-08-13.
   **Rules:** `propozycja_procesow_outbound.md` — `R81` (extension; clarified 2026-08-14 after Inbound answer `D48` — discriminator is `GR_SETTLEMENT_SOURCE`, not version number, because versions are counted separately per settlement source).
9. **DEC-L26 — no change to `R77`/§3.2 step 3, no new logistics unit:**
   **Decision:** residual quantity left after cross-docking still goes to `TRANSIT`, and the source Inbound `TU` still transitions to `IN_PUTAWAY` (`R77`, unchanged), from where it follows standard Putaway without modifying that process. No new logistics unit is introduced for residual quantity.
   **Rationale:** splitting GR settlement (`DEC-L24`) already solves the SLA problem; maintaining the same `TU_ID` through the lifecycle simplifies traceability versus the new-unit variant and does not change `Warehouse Operator` work at cross-dock.
   **Rejected alternatives:** see `DEC-L24`.
   **Date:** 2026-08-13.
   **Rules:** no new rule — `R77` remains unchanged.

10. **DEC-L27 — `damagedQty` field in the cross-dock contract with Inbound:**
   **Decision:** `CrossDockPickTask` gains attribute `damagedQty` — quantity of an SKU consistent with the TU/SKU declaration, detected as `DAMAGED` during picking (`R67`) and placed at `QC`, excluded from `confirmedQty`. The sum of `damagedQty` of all related `CrossDockPickTask` objects for a given Inbound `TU` is passed to Inbound at the same moment and through the same channel as the current cross-dock settlement confirmation (`R77`, §3.2 step 3). It does not include unexpected `SKU` (`R68`).
   **Rationale:** `DAMAGED` goods detected during cross-docking physically leave the Inbound `TU` (go to `QC`) but were previously invisible in any field sent to Inbound — Inbound's residual-quantity formula after cross-docking (`declaredQty − confirmedQty`) falsely counted that quantity as missing (`MISSING`) during quantitative Putaway checking. Reported as an integration request from the parallel Inbound session (`D50`, `../MEMORY.md`, 2026-08-15), verified directly against our files before acceptance.
   **Rejected alternatives:** none — a purely corrective extension of the existing contract (analogous to `confirmedQty`/`grAcceptanceStatus`), not a new model.
   **Date:** 2026-08-15.
   **Rules:** `propozycja_procesow_outbound.md` — new `R83`, §4.13, §3.2 step 2/3, §6.8.

### 6.2b Outbound Crossdock — fulfillmentChannel, PICKING, multiple Outbound TU

1. **DEC-L13 — `fulfillmentChannel` + SP19/SP20 + `OutboundOrder` header diagram:**
   **Decision:** `OutboundOrder` receives `fulfillmentChannel` (`STANDARD`/`CROSSDOCK`), set at creation and immutable; all `OutboundOrderLine` objects of one `OutboundOrder` have the same `fulfillmentChannel` (SP19). For `CROSSDOCK`, the header skips `ALLOCATION_IN_PROGRESS`/`ALLOCATED`/`PICKING_IN_PROGRESS`/`PICKED` and enters `PACKING_IN_PROGRESS` directly after the first `CrossDockPickTask.IN_PROGRESS` (SP20); exit to `PACKED` is shared with standard flow. A new `PACKING_IN_PROGRESS`→`CANCELLED` edge exists when all cross-dock `OutboundOrderLine` objects were `CANCELLED` by R66/R67 and none reached `PACKED` (extension of R58).
   **Rationale:** SP18 covered only absence of `Allocation` at line level; the header had no cross-dock path in §4.3. `PACKING_IN_PROGRESS` was chosen instead of `PICKING_IN_PROGRESS` because cross-dock sorting is a form of packing into the target shipping `TU`, not collecting into a `TU` that will later be packed. The edge to `CANCELLED` is limited only to the R66/R67 case (already controlled, with no goods-location risk) — not general cancellation, which remains blocked in this state (DEC-L17).
   **Rejected alternatives:** no `CANCELLED` edge from `PACKING_IN_PROGRESS` at all — rejected because the header would remain permanently stuck when all lines were zeroed; `PICKING_IN_PROGRESS` as the cross-dock state instead of `PACKING_IN_PROGRESS` — rejected in favor of more accurate packing semantics.
   **Date:** 2026-08-11.
   **Rules:** SP19, SP20, R58 (extension), §4.3.
2. **DEC-L14 — `crossDockEligibleQty`:**
   **Decision:** cross-dock eligible quantity for `CustomerOrderLine` (`crossDockEligibleQty`) = min(quantity available in Inbound `TU`, missing quantity of `CustomerOrderLine` in `BACKORDERED`); calculated when generating `CrossDockPickTask` (§3.2 step 1). Recorded as R76 with an “Invariant:” annotation, without a new document section.
   **Rationale:** a simple mathematical invariant resulting from the already described mechanism (R70, step 1), not a new process decision.
   **Rejected alternatives:** a separate “Quantity invariants” section in the document — rejected as premature for a single formula.
   **Date:** 2026-08-11.
   **Rules:** R76 (new).
3. **DEC-L15 — partial settlement of `CrossDockPickTask`:**
   **Decision:** when `CrossDockPickTask` ends with `confirmedQty < plannedQty` (missing/`DAMAGED` detected during picking or at completion, R66/R67): for `allowPartialShipment=true` — automatically, sorted part of `OutboundOrderLine`→`PACKED`, missing `CustomerOrderLine`→`BACKORDERED`. For `allowPartialShipment=false` — escalation to `Warehouse Supervisor`, three outcomes analogous to `SHORT_PICKED` (§3.5): “wait” (`BACKORDERED`), “cancellation” (sorted part [HISTORICAL — superseded by DEC-L21: does not return to Inbound `TU`/`IN_PUTAWAY`, but `PutBackTask`→`Inventory AVAILABLE`], entire line `CANCELLED`), “permanent change of `allowPartialShipment` to `true`”.
   **Rationale:** cross-dock has no inventory to reallocate (no `Inventory`, DEC-L07), so automatic reallocation from `SHORT_PICKED` has no equivalent — only the escalation part is transferred. The “cancellation” variant reuses the existing n:n remainder mechanism (R72) instead of `PutBackTask`, because there is no source location to return to [HISTORICAL — superseded by `DEC-L21`: recovery `PutBackTask`→`Inventory AVAILABLE`, see “Decision” above].
   **Rejected alternatives:** automatically keeping the sorted part as `PACKED` regardless of `allowPartialShipment` — rejected because with `allowPartialShipment=false` the customer did not agree to partial fulfillment.
   **Date:** 2026-08-11.
   **Rules:** R66/R67 (extension with `allowPartialShipment` branch), §3.2 step 2.
4. **DEC-L16 — multiple Outbound `TU` per `CrossDockPickTask`:**
   **Decision:** one `CrossDockPickTask` may use several Outbound `TU` objects after the previous one becomes full — analogously to R14 for `PickTask`/Picking `TU`. `Warehouse Operator` closes a full `TU` (`PACKING_SEALED`) by RF scan and continues the same task into a new open Outbound `TU`. This is a normal process step (§3.2 step 2), not Supervisor escalation.
   **Rationale:** R14 already establishes exactly this pattern for standard flow — extending it instead of creating a new rule is more consistent. The actor is `Warehouse Operator` (RF scan), not `Warehouse Supervisor` — this is a physical container-capacity constraint, not an exception.
   **Rejected alternatives:** manual `TU` close by `Warehouse Supervisor` with audit fields (`closedBy`/`closedAt`/`closeReason`) and return of unfulfilled quantity as outstanding demand — rejected as unnecessary complexity.
   **Date:** 2026-08-11.
   **Rules:** R14 (extension), §3.2 step 2, §4.13.
5. **DEC-L17 — `PICKING` state for cross-dock `OutboundOrderLine` and cancellation block:**
   **Decision:** a cross-dock `OutboundOrderLine` receives intermediate state `PICKING` between `CREATED` and `PACKED`: `CREATED`→`PICKING` when `CrossDockPickTask`→`IN_PROGRESS`, `PICKING`→`PACKED` after sorting. `CREATED`→`CANCELLED` remains only for an empty `TU` before picking; missing/`DAMAGED` during work transitions `PICKING`→`CANCELLED` (R66/R67). General cancellation (R65) is not allowed in `PICKING` — the WMS System rejects the request, which may be repeated after `PACKED`.
   **Rationale:** R73 hid physical progress inside `CREATED` — R65 keyed to `OutboundOrderLine` state could not distinguish “task unassigned” from “Packer scanning in progress”. During sorting, it is impossible to determine unambiguously where the SKU physically is — rollback in progress is risky, therefore a block is used instead of automatic put-back, analogous to the limitation already accepted for `PACKED`.
   **Rejected alternatives:** extending the same block to standard flow (`PICKING`/`PICKED`) — considered and withdrawn; this applies only to cross-dock. Supervisor approval + reuse of `IN_PUTAWAY` for cancellation during `PICKING` (analogous to DEC-L15) — rejected in favor of a simpler hard block because the trigger is different (external request, not a confirmed shortage).
   **Date:** 2026-08-11.
   **Rules:** R65 (new cross-dock branch), R73 (rewriting), §3.5 point 5 (new), §4.4.
6. **DEC-L18 — `CROSS_DOCKED`/`IN_PUTAWAY` criterion based on residual quantity:**
   **Decision:** Inbound `TU` transitions to `CROSS_DOCKED` when all declared (ASN) content of the `TU` has been settled by cross-docking, regardless of 1:1, 1:n, n:1 or n:n topology. If, after all related `CrossDockPickTask` objects are completed, residual unassigned physical quantity of `SKU` remains, the remainder goes to `TRANSIT` and the Inbound `TU` transitions to `IN_PUTAWAY`.
   **Rationale:** the number of target Outbound `TU` objects describes sorting topology, not the state of physically unsettled quantity of the source `TU`.
   **Rejected alternatives:** a criterion based only on 1:1 or n:n topology — rejected because it causes `IN_PUTAWAY` despite no residual quantity after full settlement in 1:n, n:1 or n:n.
   **Date:** 2026-08-11.
   **Rules:** `R77`.
7. **DEC-L19 — quantity lock on `CrossDockPickTask`:**
   [HISTORICAL — narrowed by `DEC-L38` (2026-08-22)] The sentence “One `OutboundOrderLine` may aggregate quantity from multiple `CrossDockPickTask` objects originating from different Inbound `TU` objects” in “Decision” below is obsolete — generation is one-time and closed, see `DEC-L38`. The quantity lock on `CrossDockPickTask` remains unchanged.
   **Decision:** the cross-dock quantity lock is on `CrossDockPickTask`, not on `OutboundOrderLine`. `plannedQty` reserves source Inbound `TU`/`SKU` quantity for confirmation, and the same physical quantity must not simultaneously be `planned` or `confirmed` in another active `CrossDockPickTask`. One `OutboundOrderLine` may aggregate quantity from multiple `CrossDockPickTask` objects originating from different Inbound `TU` objects.
   **Rationale:** `OutboundOrderLine` aggregates demand and confirmations, but without a lock on the source `TU`/`SKU` it does not protect against double assignment of the same physical quantity.
   **Rejected alternatives:** active `OutboundOrderLine` as the only quantity lock — rejected because it does not distinguish two tasks competing for the same source quantity.
   **Date:** 2026-08-11.
   **Rules:** `R71` (revision), `R78`, `R79`.
8. **DEC-L20 — revision of `crossDockEligibleQty`:**
   **Decision:** `crossDockEligibleQty` for `CustomerOrderLine` = min(`sourceEligibleQty`, `demandEligibleQty`). `sourceEligibleQty` reduces the declared (ASN) quantity of source Inbound `TU`/`SKU` by quantities already actively planned and completed-confirmed; `demandEligibleQty` reduces the missing quantity of `CustomerOrderLine` in `BACKORDERED` by quantities already assigned by `CrossDockPickTask` and by `ATPReservation` from standard flow. Replaces the earlier formula from `DEC-L14`.
   **Rationale:** the formula must account for already assigned quantities on both source and demand sides to prevent over-assignment.
   **Rejected alternatives:** min(full quantity available in Inbound `TU`, full shortage of `CustomerOrderLine`) — rejected because it ignores existing `CrossDockPickTask` assignments and `ATPReservation`.
   **Date:** 2026-08-11.
   **Rules:** `R76` (revision).
   **Note (2026-08-28):** the `demandEligibleQty` term of this decision was replaced by `DEC-L63`; remaining content unchanged. The entry remains historical and is not rewritten.
9. **DEC-L21 — recovery of `confirmedQty` after demand loss:**
   **Decision:** when confirmed quantity `confirmedQty` of a cross-dock `CrossDockPickTask` loses demand in the “wait” path or full cancellation with `allowPartialShipment = false`, the WMS System creates `PutBackTask`; after its `COMPLETED`, the quantity goes to a validated location as ordinary `Inventory AVAILABLE` (`ATP`). This quantity does not return to Inbound `TU` or `IN_PUTAWAY`. This is a narrow exception to `DEC-L07`, limited only to the recovery path after demand loss; happy path `DEC-L07` remains unchanged.
   **Rationale:** after sorting and demand loss, goods require physical traceable recovery into standard inventory; putting them back into Inbound `TU` would not reflect their real location.
   **Rejected alternatives:** leaving the quantity in Outbound `TU` without a recovery task or returning it to Inbound `TU`/`IN_PUTAWAY` — rejected because these do not provide physical settlement after demand loss.
   **Date:** 2026-08-11.
   **Rules:** `R65`, `R71`, `SP18`, §3.2 step 2, §4.12.

10. **DEC-L28 — `sourceEligibleQty` includes `damagedQty`:**
   **Decision:** formula `sourceEligibleQty` from `R76` subtracts from declared (ASN) Inbound `TU`/`SKU` quantity, alongside `plannedQty` of active `CrossDockPickTask` objects, the sum of `confirmedQty` and `damagedQty` of completed `CrossDockPickTask` objects — not only `confirmedQty` as before. Replaces the formula from `DEC-L20`.
   **Rationale:** `DAMAGED` quantity of a completed `CrossDockPickTask` physically leaves the source Inbound `TU` (goes to `QC`, `damagedQty`, introduced by `DEC-L27`), but the previous `sourceEligibleQty` formula did not subtract it — the WMS System could incorrectly count that quantity as still available for a new `CrossDockPickTask` even though it is physically gone. This is the same type of error reported by the parallel Inbound session (`D50`) for their own formula, found by Outbound during consistency verification after `DEC-L27` was introduced.
   **Rejected alternatives:** none — a purely corrective clarification of an existing formula, not a new model.
   **Date:** 2026-08-15.
   **Rules:** `propozycja_procesow_outbound.md` — `R76` (revision).

### 6.2c Taking warehouse tasks — withdrawal of operator pools (PickTask, CrossDockPickTask, PutBackTask)

1. **DEC-L41 — withdrawal of the operator-pool concept and `WMSAI_ADM` module:**
   **Decision:** the concept of a configurable operator pool per warehouse-task type (`WarehouseTaskTypeOperatorAssignment` object, task-type priority ranking, operator assignment by `Warehouse Supervisor`) is withdrawn in full. Module `WMSAI_ADM` together with `decyzje_adm.md` (`DEC-ADM-01`–`DEC-ADM-08`) and `wymagania_operator_pool.md` goes to `../Archiwum/` and ceases to be a source. Withdrawn: `DEC-L08`, `P2 R38`, `FR-P2-20`, `TC-095`.
   **Rationale:** the product owner judged the concept non-operational. The problem it was meant to solve — which task type takes precedence for an operator eligible for several — proved artificial: it disappears when the operator chooses the work type by entering the proper RF terminal module. Configuration complicated the algorithm instead of simplifying it.
   **Rejected alternatives:** keeping the pool in reduced scope; retaining only `CrossDockPickTask` precedence over `PickTask` in another form — irrelevant because the operator is in one module at a time.
   **Date:** 2026-08-23.

2. **DEC-L42 — taking tasks by selecting a work module in RF:**
   **Decision:** `Warehouse Operator` chooses work type by entering a module on the RF terminal (picking / cross-docking / returns). In the picking module the operator additionally selects one zone. The WMS System assigns the next task of that type according to the applicable ordering, only if the operator has no active warehouse task of any type. A task is created in `CREATED` and waits for an operator. The operator must complete the task before leaving the module.
   **Rationale:** assignment without configuration and without `Warehouse Supervisor`; work-type choice belongs to the operator who knows where they physically are and what they are doing.
   **Rejected alternatives:** selecting several zones at once (the system would send the operator across the warehouse); condition “no active task of this type” instead of “no task of any type” (operator with two open containers); suspending a task when leaving the module — deferred to shared `BACKLOG.md`, Stage 2.
   **Date:** 2026-08-23.
   **Rules:** `proces_1_standard_fulfillment.md` — `R54`, `R56`; `proces_2_outbound_crossdock.md` — `R39`; `proces_4_physical_putback.md` — `R9`.

3. **DEC-L43 — Picking `TU` independent of an individual `PickTask`:**
   **Decision:** a Picking `TU` may collect goods from successive `PickTask` objects of the same `OutboundOrder` across different zones. Completing a `PickTask` does not close the `TU` — it is closed by operator decision or by reaching the limit (`PICK_FULL`). Choosing to continue the order in another zone assigns the indicated task regardless of priority order and operator zone, and the operator's zone becomes that task's zone. Declaration `directPackDeclared` is binding for the whole `TU`, not for an individual `PickTask`.
   **Rationale:** when picking directly into shipping packaging, closing the container at a zone boundary would be pointless. Separating the `TU` lifecycle from the task lifecycle removes coupling that existed only because one `TU` previously corresponded to one task.
   **Date:** 2026-08-23.
   **Rules:** `proces_1_standard_fulfillment.md` — `R13`, `R16`, `R55`.

4. **DEC-L44 — no workplace selection in the cross-dock module (negative decision):**
   **Decision:** the cross-dock module deliberately has no equivalent of a zone. The operator enters the module and receives the next task without narrowing to a sorting location.
   **Rationale:** Outbound does not model the physical location of sorting — `IN_CROSS_DOCK` belongs to the Inbound canon, while `TRANSIT` appears in `proces_2_outbound_crossdock.md` only as the handoff location for a remainder. Introducing a location restriction would require a new entity and reaching into Inbound data (boundary `D41`) to solve a problem nobody reported.
   **Unresolved scope:** a warehouse with multiple sorting locations requires a separate explicit decision.
   **Date:** 2026-08-23.
   **Rules:** `proces_2_outbound_crossdock.md` — `R39`.

5. **DEC-L45 — `PutBackTask` covered by the same model:**
   **Decision:** `PutBackTask` is taken by entering its own returns module in RF, without selecting a zone, in task submission order.
   **Rationale:** the third warehouse-task type in Outbound had the same unnamed gap; leaving it without a mechanism would create asymmetry reported by every later audit. Submission order rather than priority because `PutBackTask` carries neither `priority` nor `slaDeadline` (`model_stanow_outbound.md` §12), and the line it concerns is already cancelled.
   **Rejected alternatives:** including `PutBackTask` in the picking module — it would restore the “which task type has precedence” problem eliminated by `DEC-L42`.
   **Date:** 2026-08-23.
   **Rules:** `proces_4_physical_putback.md` — `R9`.

### 6.3 Picking `TU` and Packing `TU`

1. `PickContainer` and `PackUnit` are roles of an Outbound `TU` in the process.
2. Every pick is performed into a Picking `TU` (`PickContainer`).
3. The `TU` type defines the basic possibility of using the unit for dispatch.
4. The system additionally evaluates the specific unit and suggests keeping, repacking or consolidating it.
5. The final decision is made by `Warehouse Operator (Packer)`.
6. Deviating from the system suggestion requires `Warehouse Supervisor` approval.
7. Weight and volume-use thresholds are configured per `TU` type. [NARROWED — see `DEC-L50`] The volume threshold is an absolute threshold of content volume (`TUSetup.minIssueVolume`), not a share of volume utilization.
8. If a Picking `TU` meets issuance conditions, the same physical `TU` may serve as the Packing `TU` and keeps `TU_NUMBER`.
9. If a Picking `TU` does not meet issuance conditions, its contents are repacked into one or more Packing `TU` objects permitted for issue.
10. Consolidation is many-to-many:
    - one Packing `TU` may contain contents from multiple Picking `TU` objects;
    - contents of one Picking `TU` may be split across multiple Packing `TU` objects;
    - every transfer is tracked at SKU and quantity level.
11. One Packing `TU` may contain lines from multiple `OutboundOrder` objects if their source `CustomerOrder` objects have the same customer, delivery address, compatible priority and delivery deadline.
12. All lines in one Packing `TU` must belong to one `Shipment`.
13. **DEC-G11 (L6):** version 1 introduces no goods compatibility/incompatibility rules for joint packing (e.g. chemicals vs food, temperature requirements) — consolidation is subject only to the conditions in point 11 (customer/address/priority/deadline).
14. **DEC-G12 — “repack all” / “repack by SKU” modes:** when repacking/consolidating into Packing `TU` (points 9–10), two modes are available: “repack all” — no quantitative check per SKU, discrepancy risk accepted; “repack by SKU” — the Packer scans and enters quantity for each SKU, and the WMS System checks consistency with `pickedQty`. Mode is configurable per warehouse: an administrator may disable one mode for the entire warehouse (forcing the other) or leave the Packer a choice for each `TU`.
    **Rationale:** Darek's decision of 2026-08-07 — operational flexibility (speed of “repack all” vs control of “repack by SKU”) with the ability for an administrator to enforce one policy where discrepancy risk is undesirable.
15. **DEC-G13 — unexpected SKU in source `TU` during packing:** on the “repack by SKU” path (DEC-G12), when the Packer encounters goods in a source Picking `TU` that are unexpected relative to content recorded in WMS for that `TU` — excess quantity of an already expected SKU or a completely different unassigned code (handled identically) — the WMS System informs the Packer that the item does not belong to that `TU`; the Packer places it at the designated `QC` (Quality Control) location. The event does not start `SHORT_PICKED` — it is recorded as a discrepancy for later investigation within future Inventory Management processes (`WMSAI_OUTBOUND/BACKLOG.md` B9).
     **Rationale:** Darek's decision of 2026-08-07 — settling the cause (how the goods got into that `TU`) requires an investigation mechanism not designed today; physically separating it to `QC` protects the goods until clarification without stopping packing.

16. **DEC-L49 — `externalIssuable = false` is an absolute block:**
    **Decision:** `P1 R65` waives only lower issuance thresholds — weight and volume. It does not waive type issuability. A `TU` whose type has `externalIssuable = false` cannot be sealed or dispatched; the only path is repacking into an issuable type (`P1 R66`). With `directPackDeclared = true` and a non-issuable type, existing Path B of STEP 7 in `proces_1_standard_fulfillment.md` applies.
    **Rationale:** the active `P1 R65` allowed bypassing the type-issuability condition, which was not what that rule was intended to waive — forcing must waive only quantity thresholds. Without this separation, a path exists by which goods leave the warehouse on a carrier not permitted for issue.
    **Rejected alternatives:** none.
    **Date:** 2026-08-25.
    **Rules:** `P1 R65`, `P1 R66`.

17. **DEC-L50 — `minIssueVolumeUtilization` replaced by absolute threshold `minIssueVolume`:**
    **Decision:** attribute `TUSetup.minIssueVolumeUtilization` is renamed to `TUSetup.minIssueVolume` and changes meaning from share of volume utilization to an absolute content-volume threshold, fully analogous to `minIssueWeight`. Attribute type in the catalog changes from “number (share)” to “number”. `P1 R64` therefore no longer depends on `TUSetup.maxVolume`.
    **Rationale:** a threshold expressed as a share tied issuability assessment to `maxVolume`, i.e. carrier cubic capacity, which is not a limit enforced during packing. An absolute threshold measures the same thing as `minIssueWeight` — achieved content — without requiring a second attribute for calculation.
    **Rejected alternatives:** none.
    **Date:** 2026-08-25.
    **Rules:** `P1 R64`.

18. **DEC-L56 — ratification of `P1 R63`–`R66` (issuance thresholds):**
    **Decision:** the four rules `P1 R63`–`R66`, which entered the current prose of `proces_1_standard_fulfillment.md` on 2026-08-24 (versions 1.6–1.8) without a register entry, are ratified by one collective entry covering them as one change to issuance-threshold semantics. The ratification concerns the state of these rules before the B16 audit repair packages.
    **Rationale:** the four rules close one mechanism missing from the body after migration even though many places referred to it. `R63` gives `TU` content weight and volume one calculable source — the sum by `SKU` from the warehouse item master; without this, “`TU` limit” was undefined and `TUSetup.maxWeight` mixed two meanings: type load capacity and actual fill. `R64` defines “issuance thresholds”, previously used many times without definition: the type must be externally issuable and at least one lower threshold must be reached; separating type issuability from fill threshold is intentional — they are two independent conditions, not one. `R65` ensures a fill threshold does not block a physically ready order: the operator may force issuability below thresholds when the `TU` is the last for its `OutboundOrder` or `slaDeadline` is approaching, with a recorded reason; forcing waives only quantity thresholds. `R66` closes the path through which repacking into a non-issuable type was previously unchecked; relative to `R64` it is an alternative, not an additional condition — repacking into a proper type replaces threshold evaluation.
    **Rejected alternatives:** issuance thresholds as one condition combining type issuability and fill — rejected because forcing under `R65` would then also waive type issuability, enabling dispatch on a carrier that must not be used externally. Content volume as a second fill-blocking condition (`R15`) equal to weight — rejected because fill blocking is measurable by weight and the operator decides volume fill by closing the `TU`. `TUSetup.maxVolume` as an enforced limit — rejected because it is carrier cubic capacity, an input to carrier selection (`R30`), not a packing boundary. Leaving repacking without type control — rejected because it is a real gap allowing goods out on a non-issuable carrier.
    **Date note:** the rules entered `proces_1_standard_fulfillment.md` prose on 2026-08-24 (versions 1.6–1.8) before the register entry. This entry is a ratification from 2026-08-26, not a record of a decision from 2026-08-24.
    **Scope note:** the entry ratifies the mechanism. Volume threshold `R64` is separately changed to absolute threshold `TUSetup.minIssueVolume` by the decision in `DEC-L50`; the ratification does not quote the pre-change wording of `R64` as current.
    **Date:** 2026-08-26.
    **Rules:** `P1 R63`–`P1 R66`.

**Rules:** `propozycja_procesow_outbound.md` — points 11–12: `SP6`, `SP14`, `R20`.

## 7. `Shipment`, Carrier Selection and dispatch

### 7.1 Creating `Shipment`

1. `Shipment` is created after Packing `TU` is prepared.
2. `Shipment` groups one or more ready Packing `TU` objects.
3. One Packing `TU` belongs to exactly one `Shipment`.
4. `Shipment` relationships with `CustomerOrder` and `OutboundOrder` follow from Packing `TU` contents.

**Rules:** `propozycja_procesow_outbound.md` — `SP7`, `R21`.

### 7.2 Carrier Selection

1. Order of actions: Packing `TU` → `Shipment` → Carrier Selection → label.
2. External-carrier selection uses final shipment parameters, address and deadline.
3. Own transport is indicated earlier and does not participate in Carrier Selection.
4. No result from Carrier Selection rules requires a `Warehouse Supervisor` decision.
5. A label is created only after the carrier is approved.
6. **DEC-J10:** GS1 AI `(00)` may be used on the printed label only for a valid `SSCC`, never for an ordinary `TU_NUMBER`.
7. **DEC-J11 — Carrier Selection rule mechanism (L3):** external-carrier selection is based on two configured criteria, not hard-coded rules:
   - **`Region`** — new configuration object representing a delivery area (freely defined per warehouse: postal code, range, group of voivodeships, etc.); independent of `Zone` (internal warehouse topology).
   - **`CarrierSetup`** — new configuration object linking `Carrier`, `Region`, a weight interval (`minWeight`–`maxWeight`) and a volume interval (`minVolume`–`maxVolume`). One `Carrier` may have many `CarrierSetup` objects (different regions and/or weight-volume variants).
   - `Carrier` becomes a formal master-data object (today in the document it appears only as an actor “outside the WMS boundary”, §3.1 P1) and receives a new `priority` attribute — a unique value in the carrier dictionary used only as the final tie-break (DEC-J12).
   **Rules:** `propozycja_procesow_outbound.md` — `R61`.
8. **DEC-J12 — matching algorithm and tie-break:** during Carrier Selection, the WMS System calculates `maxWeight` and `maxVolume` as maximum values among **all** Packing `TU` objects belonging to the given `Shipment` — independently, i.e. they may come from different `TU` objects (for example the heaviest `TU` and separately the largest-by-volume `TU`). `CarrierSetup` matches when: delivery `Region` matches AND `maxWeight` is in `[minWeight, maxWeight]` of that `CarrierSetup` AND `maxVolume` is in `[minVolume, maxVolume]` of that `CarrierSetup`. When more than one `CarrierSetup` matches for the same `Region` (overlapping intervals), resolve in order: (a) narrowest volume interval (`maxVolume - minVolume`); (b) on tie — narrowest weight interval (`maxWeight - minWeight`); (c) on further tie — `Carrier.priority` (always resolves uniquely because the value is unique in the `Carrier` dictionary).
   **Rules:** `propozycja_procesow_outbound.md` — `SP17`, `R61`, `R62`.
9. **DEC-J13:** no matching `CarrierSetup` for a `Shipment` → manual carrier selection by `Warehouse Supervisor` (consistent with point 4). `Warehouse Supervisor` may also always change the Carrier Selection result — automatic or manual — without providing a reason (distinguished from permanent change of `allowPartialShipment`, R6, which requires a reason).
   **Rules:** `propozycja_procesow_outbound.md` — `R6`, `SP17`, `R63`.
10. The whole `Shipment` travels with one `Carrier` — no splitting one `Shipment`'s lines among multiple carriers (consistent with D-7.1/SP7, `propozycja_procesow_outbound.md`).
    **Rationale (DEC-J11-J13):** Darek's decision of 2026-08-02 — start with a simple two-parameter mechanism (region + weight/volume), fully configurable without code changes, with an unambiguous tie-break (`Carrier.priority` guarantees resolution) and full freedom for Supervisor correction. Closes L3.
11. **DEC-J14 — L4 not applicable in version 1:** the label is a document printed by the WMS System from data it already has (Packing `TU` data, selected `Carrier` description, delivery address) — without calling an external carrier API and without a confirmation/approval step on the carrier side. Consequently: (a) there is no technical label-generation failure mode requiring a separate exception path; (b) there is no electronic shipment rejection by the carrier. The real equivalent: a loading problem discovered **before** `CarrierManifest.CLOSED` — `Warehouse Supervisor` manually changes `Carrier` (DEC-J13), without reprinting the label. Boundary unchanged: `Carrier` cannot be changed after `CarrierManifest.CLOSED` (DEC-K10). Closes L4.
    **Rationale:** Darek's decision of 2026-08-02 — version 1 assumes no integration with a carrier system (neither sending data nor receiving confirmation/approval), so both original L4 scenarios (technical API error, electronic rejection) have no anchor in this version; the only real exception is an operational WS decision at loading, already covered by DEC-J13.

### 7.3 `CarrierManifest`

1. The same `CarrierManifest` object supports an external carrier and own transport.
2. For an external carrier, a manifest concerns one carrier and one specific pickup.
3. For own transport, a manifest groups `Shipment` objects for a specific run received from an external process.
4. Vehicle and route planning remains outside WMS scope.
5. One `Shipment` may belong to only one `CarrierManifest`.
6. The manifest is closed by `Dispatcher`.
7. **DEC-K08:** `CarrierManifest.CLOSED` means the manifest composition is locked and ready for physical handover. It does not yet mean shipments were handed over.
8. **DEC-K09:** only physical handover causes `CarrierManifest.HANDED_OVER` and the corresponding dispatch state of linked `Shipment` and `OutboundOrder` objects.
9. **DEC-K10:** cancellation is not allowed from the moment `CarrierManifest.CLOSED` is reached, even if physical handover occurs later. A closed manifest cannot be reopened.
10. **DEC-L60:** `OutboundOrder` reaches `COMPLETED` only after confirmation (`CarrierManifest CONFIRMED`) of all `Shipment` objects containing its Outbound `TU` objects. Confirmation of the first of several such manifests leaves the order in `DISPATCHED`; confirmation order does not matter. Settlement of `OutboundOrderLine PACKED → SHIPPED`, `Allocation CONFIRMED → CONSUMED` and `Inventory PICKED → SHIPPED` is limited to lines dispatched by that manifest. Reason: the invariant lived in the archived analytical document as `SP4` and was not transferred during B16 migration, even though `DEC-D18` still referred to it as the active premise for the absence of a “partially shipped” `OutboundOrder` state. An order may have Packing `TU` objects in several `Shipment` objects when some are sealed after automatic grouping close by `slaDeadline`.
    **Rules:** `proces_1_standard_fulfillment.md` — `R70`, STEP 13; `model_stanow_outbound.md` — section 3, `DISPATCHED→COMPLETED` row.

11. **DEC-L62:** `CarrierManifest` confirmation settles `Inventory` quantitatively — only for quantity contained in the Outbound `TU` objects of that manifest — and reduces `Allocation.reservedQty` by that quantity. Terminal states `OutboundOrderLine SHIPPED` and `Allocation CONSUMED` are reached only when every Outbound `TU` contributing quantity of that line belongs to a `Shipment` with a `CONFIRMED` manifest. Reason: `P1 R27` allows multiple Outbound `TU` objects on one `OutboundOrderLine`, and `P1 R28` and `P1 R29` may split them among different `Shipment` objects; previous wording terminalized the whole line and whole allocation on the first confirmed manifest despite the physically unshipped remainder. Partial inventory settlement remains allowed — the block applies only to terminal transitions. No new states were introduced; `P1 R70` at `OutboundOrder` level is unchanged. The rule relies on the Outbound `TU` contents ↔ `OutboundOrderLine` relationship, like existing `P1 R27`, `P1 R58` and `P1 R60`; the target representation of that relationship belongs to `BACKLOG.md` B24 and will cover all four rules at once.
    **Rules:** `proces_1_standard_fulfillment.md` — `R72`, `R71`, STEP 13; `model_stanow_outbound.md` — section 4 `PACKED→SHIPPED` row, section 5 `CONFIRMED→CONSUMED` row.

**Rules:** `propozycja_procesow_outbound.md` — point 5: `SP9`, `R29`; point 9: `SP10`.

## 8. Cancellation and put-back

1. Reservation release and physical put-back are separate variants:
   - before picking, the system releases `Allocation` without physical work;
   - after picking, a physical task to return goods is created.
2. Physical put-back creates a separate task for the operator.
3. The target location is proposed by the system or indicated by the operator when they know the storage location. The location must pass WMS validation.
4. Cancellation after packing requires `Warehouse Supervisor` approval and may include invalidating the label, removing Packing `TU` from `Shipment` and physical put-back.
5. The actual cancellation boundary is closure of `CarrierManifest`.
6. After the manifest is closed, cancellation is impossible even if physical handover has not yet happened.
7. **DEC-L01 — `PutBackTask` with no attempt limit (L5, INFO #8):** loop `LOCATION_VALIDATION↔IN_PROGRESS` (`propozycja_procesow_outbound.md` §4.12) has no attempt limit and no automatic escalation to `Warehouse Supervisor` — deliberately, because rejection risk applies practically only to the manual path (operator selects the location themselves, point 3); the WMS System recommendation remains available to the operator throughout the task as a fallback. When a self-selected location is rejected, the operator either continues looking for a valid one or accepts the system recommendation — without hard enforcement after N attempts. Closes INFO #8 (B2 check).
   **Rationale:** Darek's decision of 2026-08-02 — the system's primary task is to indicate a put-back location while avoiding rejection risk (system recommendation); a limit and escalation are unnecessary when the operator always has a safe exit.
8. **DEC-L02 — general cancellation of `CustomerOrder`/`CustomerOrderLine` unrelated to shortage:** a cancellation request is accepted from two independent channels — an external system (OMS/ERP through webservice) and manually by `Warehouse Supervisor` in WMS. Cancelling an entire `CustomerOrder` is allowed only when each of its `CustomerOrderLine` objects meets the line-cancellation condition; if even one does not, the entire header remains blocked. The condition for an individual `CustomerOrderLine` depends on the related `OutboundOrderLine` status: no `OutboundOrderLine` or `CREATED`/`ALLOCATED`/`SHORT_ALLOCATED` — release `Allocation` without physical work; `PICKING` in progress — cancel `PickTask` (existing R59 mechanism); `PICKED`/`SHORT_PICKED` — an `RF` message stops packing and a `PutBackTask` is created for picked quantity; `PACKED` — outside this mechanism, requires a separate `Warehouse Supervisor` decision (D-8.4/R32: cancel packed `TU`, remove Packing `TU` from `Shipment`) — only then does the line return to the normal cancellation path. General boundary unchanged — `CarrierManifest` close (R30).
   **Rationale:** separating the self-service cancellation channel (webservice/Supervisor, no additional approval) from the heavier path requiring a Supervisor decision when goods are already packed prevents uncontrolled unpacking without a conscious decision.

**Rules:** `propozycja_procesow_outbound.md` — points 5–6: `SP10`, `R30`.

## 9. Withdrawn decisions

The following earlier assumptions must not be used:

1. `OutboundOrder` as a direct picking task for the operator.
2. Creating a separate `OutboundOrder` for every warehouse zone.
3. Requiring multiple `OutboundOrder` objects for `allowPartialShipment = false` because of zone split.
4. Allowing cancellation until `HANDED_TO_CARRIER`. The final boundary is closure of `CarrierManifest`.
5. Treating `TU_ID` and `SSCC` as the same concept.
6. Treating `CarrierManifest` and `Load` as two objects in version 1.
7. **WD-06:** storing an invalid SSCC or any external number in field `SSCC`.
8. **WD-08:** designing `PickWave` as part of version 1 or as a technical extension point in version 1.
9. **WD-09:** a Supervisor exception as temporary, not changing `allowPartialShipment`, limited to one Shipment. Superseded by a permanent flag change (section 4.1 points 4–5, 2026-07-18).

## 10. Open issues

The following issues must not be presented as approved decisions. Variants and recommendations must be marked as proposals:

1. ~~Full catalog of states and transitions in `model_stanow_outbound.md`.~~ → **resolved:** `model_stanow_outbound.md` v1.0 (2026-08-18, `BACKLOG.md` B5).
2. ~~Business events emitted on transitions and actors responsible for individual stages.~~ → **resolved:** as above.

No other open issues (status as of 2026-08-18).

## 11. Change history
- **2026-08-28 (closure of B21 and B23):** added `DEC-L64` in section 4.1 (gate counts mixed coverage) and `DEC-L65` in section 5.1 (`directPackDeclared` inheritance). Changed `P1 R6`, `P1 R16`, `P1 R67`, prose of `STEP 2A` and `STEP 6`, `FR-P1-03`, `FR-P1-41`; new `TC-135`. `proces_1_standard_fulfillment.md` → 1.20, `model_stanow_outbound.md` → 1.19. Local-rule count unchanged (132).
- **2026-08-28 (closure of B22):** added `DEC-L63` in section 6.2 — `demandEligibleQty` subtracts `ATPReservation` and sum of `requiredQty` of non-cancelled `OutboundOrderLine` objects instead of only `ATPReservation`. Added a narrowing note at `DEC-L20` without rewriting its content. Changed `P2 R6`, `FR-P2-03`, `TC-036`; new `TC-134`; `proces_2_outbound_crossdock.md` → 1.13. Local-rule count unchanged (132).
- **2026-08-28 (multi-stage line settlement):** added `DEC-L62` in section 7.3 — `OutboundOrderLine SHIPPED` and `Allocation CONSUMED` only after all Outbound `TU` objects contributing line quantity are dispatched; `Inventory` settled quantitatively at each manifest. New `P1 R72`, supplemented `P1 R71`, `FR-P1-46`, `TC-132`/`TC-133`; `proces_1_standard_fulfillment.md` → 1.19, `model_stanow_outbound.md` → 1.17, coverage matrix 132 local rules. `P1 R70` unchanged.
- **2026-08-28 (`BACKLOG.md` B20, `Allocation` part):** added `DEC-L61` in section 4.2 — `reservedQty` attribute and contribution of six allocation states to occupied quantity, variant A. New `P1 R71`, `FR-P1-45`, `TC-130`/`TC-131`; `proces_1_standard_fulfillment.md` → 1.18, `model_stanow_outbound.md` → 1.16, coverage matrix 131 local rules. `P2 R6` unchanged — B22 remains open.
- **2026-08-28 (B16 migration loss — archived `SP4`):** added `DEC-L60` in section 7.3 — `OutboundOrder COMPLETED` only after confirmation of all its `Shipment` objects. New `P1 R70`, `FR-P1-44`, `TC-128`/`TC-129`; `proces_1_standard_fulfillment.md` → 1.16, `model_stanow_outbound.md` → 1.15, coverage matrix 130 local rules.
- **2026-08-28 (WP-10 package of the B16 audit-repair plan):** filled the **Rules:** fields in all eleven entries `DEC-L49`–`DEC-L59` with actual body references after WP-01–WP-09; `DEC-L55` received explicit confirmation of the decision not to introduce a change. Entry contents are consistent with the current body. No `DEC-L` entry was created.
- **2026-08-26 (WP-00 package of the B16 audit-repair plan):** created eleven entries `DEC-L49`–`DEC-L59` for owner decisions of 2026-08-25 and 2026-08-26, forming the register basis for behavior changes propagated by later plan packages. Section 4.1: `DEC-L53` (`allowPartialShipment = false` guard keyed by `CustomerOrder`) and `DEC-L57` (`requiredQty` attribute and lifecycle). Section 6.1: `DEC-L51` (external Outbound `TU` carrier type) and `DEC-L59` (`EXTERNAL` value of `TUSetup.processUsage`). Section 6.2: `DEC-L52` (full 1:1 match definition), `DEC-L54` (`priority`/`slaDeadline` of cross-dock `OutboundOrder`), `DEC-L55` (`sourceInboundTU` does not return on Outbound `TU`) and `DEC-L58` (narrowing `DEC-L38`). Section 6.3: `DEC-L49` (`externalIssuable = false` as absolute block), `DEC-L50` (`minIssueVolume` as absolute threshold) and `DEC-L56` (ratification of `P1 R63`–`R66`). The **Rules:** fields of all eleven entries remain empty until WP-10, which fills them with actual rule numbers assigned in WP-01–WP-09. Two narrowing notes were added without rewriting the affected content: at `DEC-L38` (reference to `DEC-L58`) and section 6.3 point 7 (reference to `DEC-L50`). `DEC-L29` unchanged. No process or model file was changed in this package.
- **2026-08-23 (completion of exchange with Inbound):** added `DEC-L48` — separation of source Inbound `TU` finalization from assignment of `IN_PUTAWAY`, after a discrepancy was reported by the Inbound session. Revision of `R21`, `R28` and `R42` in `proces_2_outbound_crossdock.md` → 1.9. No new rules, requirements or scenarios.
- **2026-08-23 (latest):** added `DEC-L47` — closed the Inbound→Outbound Crossdock boundary after changes reported by the Inbound session: Inbound qualification is a transport prerequisite, binding matching occurs at `IN_CROSS_DOCK` (`R30`); described the zero-match path (`TU` finalizes immediately toward `IN_PUTAWAY` with the full quantity residual) and the boundary invariant (only `TU` `ELEMENTARY`). `proces_2_outbound_crossdock.md` → 1.8, new `R41`, `R42`.
- **2026-08-23 (later):** added `DEC-L46` — closed audit item V3-CD-02 (PutBack vs shared target Outbound `TU`): empty-`TU` cancellation guard supplemented with absence of active and planned `CrossDockPickTask` objects targeting it (removes dangling target reference), with explicitly intended asymmetry against `DEC-L39` — cancellation does not wait for `slaDeadline`; added active-task target-continuity invariant. Marked `[HISTORICAL]` on `DEC-L31` with reference to `DEC-L46`. Results propagated to `proces_2_outbound_crossdock.md` 1.6 (`R34` revision, `R40` new), `model_stanow_outbound.md` 1.5, `wymagania_outbound.md` (`FR-P2-19` revision, new `FR-P2-22`), `scenariusze_testowe_outbound.md` (`TC-035` extended, new `TC-101`), `macierz_pokrycia_outbound.md` (new `P2 R40` row, P2 count 37 → 38, total 110 → 111).
- **2026-08-23:** withdrew the operator-pool concept in full (`DEC-L41`) — `WMSAI_ADM` archived, `DEC-L08` marked historical. New `DEC-L42`–`DEC-L45`: rules for taking warehouse tasks by selecting an RF work module, for `PickTask`/`CrossDockPickTask`/`PutBackTask`. Execution of `BACKLOG.md` B10 (closed by rejection of the original assumption, not by implementing it).
- **2026-08-22 (later):** added `DEC-L38` (one-time closed generation of `CrossDockPickTask`/`OutboundOrderLine`), `DEC-L39` (automatic close of target Outbound `TU` after `slaDeadline`) and `DEC-L40` (GR gate re-evaluation per message, `GR_REJECTED` does not cause `POSTING_ERROR`, `grAcceptanceStatus` visible to Supervisor) — three owner decisions closing audit V3-CD. Marked the corresponding fragments as historical in `DEC-L19` (narrowed), `DEC-L23` (narrowed) and `DEC-L36` (superseded). Results propagated to `proces_2_outbound_crossdock.md` 1.3, `model_stanow_outbound.md` 1.4, `wymagania_outbound.md`, `scenariusze_testowe_outbound.md`, `macierz_pokrycia_outbound.md` (parts 1–4 of audit V3-CD).
- **2026-08-22:** added `DEC-L36` (two triggers for target Outbound `TU` `PACKING_SEALED` in cross-docking) and `DEC-L37` (source Inbound `TU` finalization takes precedence over new demand) — owner decisions made while repairing propagation gaps after B5/B16 migration. Added `DEC-A14` (archiving the analytical document) and `DEC-A15` (no separate file for PROCESS 5) — recording the 2026-08-22 arrangements in the canonical decision register per `DEC-A10`. Results propagated to `proces_1_standard_fulfillment.md` 1.2, `proces_2_outbound_crossdock.md` 1.2, `model_stanow_outbound.md` 1.3, `wymagania_outbound.md`, `scenariusze_testowe_outbound.md` and `macierz_pokrycia_outbound.md`.
- **2026-08-18 (even later):** added `DEC-L35` — alternative picking path: Picker declares picking directly into Outbound `TU` (`TU.directPackDeclared`), WMS System automatically closes `PackUnit` role and moves `OutboundOrderLine PICKED→PACKED` when issuance conditions are met; otherwise standard Packer path as fallback verification. Updated `propozycja_procesow_outbound.md` → 1.29 (`R86`, §3.1 step 6a/7, §4.7, §4.4) and `model_stanow_outbound.md` → 1.2.
- **2026-08-18 (later):** closed `BACKLOG.md` B15. Added `DEC-L33` (`CustomerOrder SHIPPED→CLOSED` trigger — calculated automatically when all contributing `OutboundOrder` objects are `COMPLETED`) and `DEC-L34` (removal of unused `OutboundOrder.ON_HOLD`, no coverage in any process step or earlier decision). Updated `propozycja_procesow_outbound.md` → 1.28 (`R85`, §3.1 step 13a, §4.1, §4.3, §9 L-INV) and `model_stanow_outbound.md` → 1.1.
- **2026-08-18:** closed §10 points 1–2 — created `model_stanow_outbound.md` v1.0 (`BACKLOG.md` B5): catalog of states, transitions, domain events (`PascalCase` EN) and actors for 17 objects from §4 of `propozycja_procesow_outbound.md` v1.27, with column and section format aligned to Inbound standard `model_stanow.md`. During this work, two documentation gaps with no described process step in §3 were discovered and reported to `BACKLOG.md` (B15): `CustomerOrder SHIPPED→CLOSED` and `OutboundOrder PICKING_IN_PROGRESS↔ON_HOLD`/resume.
- **2026-08-13:** closed `BACKLOG.md` B4. Added `DEC-G14`–`DEC-G19`: `SSCC` validation, `TU_NUMBER` format and uniqueness, no later reclassification, conflict handling in service mode and `Sequence`/`TUSetup` model. Updated `propozycja_procesow_outbound.md` to version 1.21 (`R26`, `R27`, `R82`, §4.7, §4.8, §4.17, `K10`, `K11`).
- **2026-07-18:** moved key findings from the control prompt into project sources; approved separation of planning from partial allocation, `CarrierManifest.CLOSED`/`HANDED_OVER` semantics, hybrid `SHORT_PICKED` handling, warehouse/customer configuration of reallocation limit, SSCC rules and knowledge management without prompts as sources.
- **2026-07-18:** replaced one-time temporary Supervisor exception (`allowPartialShipment`) with a permanent flag change with reason — see section 4.1 points 4–5 and WD-09.
- **2026-07-18:** resolved L2 — `PickTask` execution ordering as a warehouse parameter (`slaDeadline`→`priority` by default, or `priority`→`slaDeadline`), tie resolved by `PickTask` submission order. See section 4.3 point 7.
- **2026-07-18:** resolved B3a — ATP reservation mechanism at `CustomerOrderLine` level (`ATPReservation`), added section 4.3 points 8–14, updated point 3.
- **2026-07-19:** completed MEDIUM #3 (B2 check, continuation): corrected DEC-D14 and DEC-D15 (explicit `allowPartialShipment = false` scope, distinction between `CustomerOrderLine` `BACKORDERED`/`OPEN` after `SHORT_ALLOCATED` cancellation), DEC-F15 (same distinction for `SHORT_PICKED`, correction of `ATPReservation` return by confirmed missing quantity, `PACKED` exception for returning header to `ACCEPTED`) and §4.3 point 13 (R47 source — analogous correction).
- **2026-08-02:** resolved L1/L7 — line→header status aggregation. L1 (`OutboundOrder`): already covered by SP2–SP4/R58 in `propozycja_procesow_outbound.md`, no new decision. L7 (`CustomerOrder`): header = least advanced active `CustomerOrderLine`, exception `PARTIALLY_SHIPPED` when at least one is `SHIPPED`, exception `BACKORDERED` only when all active lines are `BACKORDERED`. Added DEC-D18 (new section 4.4); extended DEC-D15 to both `allowPartialShipment` variants and added trigger (d) to DEC-D16 (§4.2 points 12/13/14); removed from §10 Open issues.
- **2026-08-02 (later, L3):** resolved L3 — Carrier Selection mechanism. New configuration objects `Region` and `CarrierSetup` (delivery region + weight/volume interval per `Carrier`); match against `maxWeight`/`maxVolume` calculated from Packing `TU` objects in `Shipment`; tie-break: narrowest volume → narrowest weight → `Carrier.priority` (new attribute, unique in `Carrier` dictionary); no match → manual `Warehouse Supervisor` selection, who may always change the result without providing a reason. Added DEC-J11–DEC-J13 (§7.2); removed L3 from §10 Open issues.
- **2026-08-02 (even later, L4):** resolved L4 — not applicable in version 1. Label is a WMS System printout without external API and without carrier approval step — no failure mode, no electronic rejection. Real mechanism: loading problem before `CarrierManifest.CLOSED` → manual `Carrier` change by `Warehouse Supervisor` (DEC-J13), without reprinting label; `CLOSED` boundary unchanged (DEC-K10). Added DEC-J14 (§7.2).
- **2026-08-02 (next, L5):** resolved L5, put-back part — DEC-L01 (§8): `PutBackTask` with no attempt limit in `LOCATION_VALIDATION↔IN_PROGRESS` loop, system recommendation always available as exit; closes INFO #8 (B2 check). The “blocked-location check” part (`StockCheckTask`, DEC-F10) was deferred to a future Inventory Management process group, parked in `../MEMORY.md`. Removed from §10 Open issues points 5 (label, resolved L4 — missed in previous edit) and 6 (put-back).
- **2026-08-03 (L6):** resolved L6 — SKU/package compatibility for joint packing. Version 1 introduces no goods compatibility/incompatibility rules; consolidation is subject only to customer/address/priority/deadline conditions already described in §6.3 point 11. Added DEC-G11 (§6.3 point 13); removed point 4 from §10 Open issues.
- **2026-08-07:** resolved general cancellation triggers (BACKLOG B3), DEC-L02; updated `propozycja_procesow_outbound.md` (version 1.10), `BACKLOG.md` and `handover_outbound_wms.md`.
- **2026-08-07:** resolved the last B3 point — missing/damaged/excess goods detected during packing; DEC-F18, DEC-F19, DEC-G12, DEC-G13; `propozycja_procesow_outbound.md` version 1.12; `BACKLOG.md`, `handover_outbound_wms.md`.
- **2026-08-08:** completed decision→rule traces (**Rules:** lines) for decisions referenced so far in §5/§7 of `propozycja_procesow_outbound.md` — prerequisite for BACKLOG B8.
- **2026-08-11:** removed from §10 Open issues two obsolete points — priority/`slaDeadline` (resolved 2026-07-18, §4.3 point 7) and full package cross-dock flow (resolved 2026-08-10, Outbound Crossdock §3.2/§6.2); renumbered the list.
- **2026-08-11:** corrected DEC-L04 (citation R1/R4→SP14/R20, editorial correction). Added DEC-L09–DEC-L12: `READY_FOR_DISPATCH` before Carrier Selection in `Shipment`, consolidation requires identical `slaDeadline` with timeout close, `Shipment` cancellation boundary = `POSTING_PENDING` (then Return Receipt, BACKLOG B13), `OutboundOrder.PACKED` as full aggregate. `propozycja_procesow_outbound.md` → 1.15.
- **2026-08-11 (Group B):** added DEC-L13–DEC-L17 (§6.2b Outbound Crossdock): `fulfillmentChannel`/SP19/SP20 and cross-dock `OutboundOrder` header diagram; R76 (`crossDockEligibleQty`); `allowPartialShipment` branch for missing/DAMAGED on `CrossDockPickTask` completion; R14 extension (multiple Outbound `TU` per task); `PICKING` state for cross-dock `OutboundOrderLine` and general-cancellation block until `PACKED`. `propozycja_procesow_outbound.md` → 1.16.
- **2026-08-11 (Crossdock follow-up):** added `DEC-L18`–`DEC-L21` (`CROSS_DOCKED`/`IN_PUTAWAY` criterion, `CrossDockPickTask` quantity lock, revision of `crossDockEligibleQty`, `confirmedQty` recovery mechanism); added reference to `DEC-L21` in `DEC-L07`. `propozycja_procesow_outbound.md` → 1.17.
- **2026-08-12:** added DEC-L22 — `Shipment POSTING_ERROR` correction mechanism for content discrepancy (two paths: ERP-side cause vs WMS-side cause); closes `WMSAI_OUTBOUND/BACKLOG.md` B3c. `propozycja_procesow_outbound.md` → 1.19, `R80`.
- **2026-08-12:** added `DEC-L23` — ICR-05: ERP gate of cross-dock `Shipment` waits for `GR_ACCEPTED` of every source Inbound `TU` contributing quantity to the `Shipment`, instead of `POSTED` of the whole ASN; added `grAcceptanceStatus` and `R81`. `propozycja_procesow_outbound.md` → 1.20.
- 2026-08-13: added `DEC-L24`–`DEC-L26` — GR settlement for Outbound Crossdock split into cross-dock settlement and putaway settlement of the same Inbound `TU`; `Shipment` waits only for the cross-dock settlement. `propozycja_procesow_outbound.md` → 1.22 (`R54`, `R81` clarified; §3.2 step 3/4, §4.13, §6.8, §9 L-CD updated).
- 2026-08-14: supplemented DEC-L25 (Rules) after Inbound D48 answer to ICR-06–ICR-08 — the cross-dock settlement discriminator is `GR_SETTLEMENT_SOURCE`, not version number. `propozycja_procesow_outbound.md` → 1.23 (`R81` clarified).
- 2026-08-15: added `DEC-L27` — `damagedQty` in the cross-dock contract with Inbound; `propozycja_procesow_outbound.md` → 1.24 (`R83`, §4.13, §3.2 step 2/3, §6.8).
- 2026-08-15: added `DEC-L28` — correction to `R76`: `sourceEligibleQty` also subtracts `damagedQty` of completed `CrossDockPickTask` objects; `propozycja_procesow_outbound.md` → 1.25.
- 2026-08-15: added `DEC-L29`–`DEC-L30` — Outbound Crossdock consistency audit (8 propagation gaps): clarified when Outbound `TU` is created in cross-docking (at first placement of `SKU`, not at `PACKING_SEALED`, `DEC-L29`); aligned quantity-base terminology in `R76`/`R77`/§3.2 from “physically identified” to “declared (ASN)”, according to D42 (`DEC-L30`); marked `[HISTORICAL]` on `DEC-L03`, `DEC-L06` and a fragment of `DEC-L15`. `propozycja_procesow_outbound.md` → 1.26.
- 2026-08-17: continuation of Outbound Crossdock consistency audit — added `DEC-L31` (Outbound `TU` empty after full `PutBackTask` → `CANCELLED`, new `R84`) and `DEC-L32` (`grAcceptanceStatus` updated system-wide by `sourceInboundTU`, not per `Shipment`, revision of `R81`); corrected `R27` (reference to source, not “completed”, Inbound `TU`); corrected state diagram/transition table §4.7 (new cross-dock edge `CREATED`→`PACKING_SEALED`, clarification of `CREATED`→`CANCELLED`); corrected sequence diagram §6.8 (assignment of `TU_NUMBER`/`SSCC` moved from `PACKING_SEALED` to first `SKU` placement, according to §3.2 step 1); clarified sentence in process overview (moment Outbound `TU` is created); marked `[HISTORICAL]` on `DEC-L23` (task-identifier fragment in GR signal) and a fragment of `DEC-L15` (Rationale). `propozycja_procesow_outbound.md` → 1.27.