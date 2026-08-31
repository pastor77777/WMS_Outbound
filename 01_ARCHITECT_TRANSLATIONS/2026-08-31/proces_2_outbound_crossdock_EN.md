# PROCESS 2: OUTBOUND_CROSSDOCK

**Project:** WMSAI Outbound  
**Version:** 1.13 | **Date:** 2026-08-28 | **Author:** Business Analyst  
**Origin:** this document was created by expanding archived `propozycja_procesow_outbound.md` v1.29 §3.2 (Outbound Crossdock) and §3.5 (P2 part) into the step-by-step format compliant with `SZABLON_PROCESU.md`. The archived document is historical material, not a source of truth — this file is the source of truth (`DEC-A14`).  
**Previous process:** Inbound Process 2 (Unloading) — passes Inbound `TU` (`ELEMENTARY`) in `IN_CROSS_DOCK`  
**Next process:** none in WMS for quantity settled by cross-docking (physical pickup outside boundary); Inbound Process 3 (Putaway) for any residual quantity. PROCESS 1 (Standard Fulfillment) shares steps 9–13 with this process (`Shipment`, Carrier Selection, label, ERP gate, `CarrierManifest`, dispatch).  
**Document scope:** business-process description of transshipping goods received in Inbound directly to dispatch, without storage and without picking from stock. The catalog of states, transitions and domain events is maintained in `model_stanow_outbound.md` — this document describes behavior and does not define new states.

---

## Process objective

Transship goods received in Inbound directly to dispatch by matching contents of Inbound `TU` to `CustomerOrderLine` in `BACKORDERED`, without storage and without creating `Allocation`. Quantity settled through cross-docking joins the standard `Shipment` → Carrier Selection → `CarrierManifest` chain (shared with PROCESS 1), while any residual quantity returns to standard Putaway on the Inbound side.

## Participants

- **Warehouse Operator (Packer)** — picks and checks quality/quantity of `SKU` from source Inbound `TU`, sorts into target Outbound `TU` (step 2).
- **Warehouse Supervisor** — escalation on shortage/damage for `allowPartialShipment = false` (see “Exceptions and alternative paths”); visibility into `grAcceptanceStatus` of source Inbound `TU` blocking the ERP gate for stuck `Shipment` (step 4, **R36**). `Warehouse Operator` is assigned automatically by entering the cross-docking module (**R39**), without `Warehouse Supervisor` involvement.
- **Dispatcher** — participates in shared PROCESS 1 steps 9–13 (see **STEP 4**).
- **System WMS** — generates `CrossDockPickTask`, calculates eligible quantity, manages target Outbound `TU`, settles task, applies ERP gate waiting for `GR_ACCEPTED` of source Inbound `TU`.

## Start event

Inbound `TU` (`ELEMENTARY`) reaches `IN_CROSS_DOCK` (Inbound boundary — see note below). Qualification of `TU` for cross-docking is solely the responsibility of Inbound Process 2; this process takes over a `TU` already qualified.

## Process flow

---

### STEP 1 — Generate `CrossDockPickTask`
**[System WMS]**

- System WMS reads `CustomerOrderLine` in `BACKORDERED`, ordered by the applicable order-priority queue parameterized per warehouse (**R1**).

◇ **Matching topology between source Inbound `TU` and demand?**

**Path A — full 1:1 match → [System WMS]**
- Entire ASN-declared contents of Inbound `TU` complete one `Shipment` after satisfying the full common-destination criterion in **R2**.
- `OutboundOrderLine` in `CREATED` are created for one target Outbound `TU`.

**Path B — n:n sorting → [System WMS]**
- An `OutboundOrderLine` is created per matched line, for a new or open Outbound `TU` matching customer/address/`priority` and identical `slaDeadline`; one Outbound `TU` may collect `SKU` from several Inbound `TU` (**R3**).

- Regardless of path: `CrossDockPickTask CREATED` is created with references to source Inbound `TU` and target `OutboundOrderLine`; `CustomerOrderLine BACKORDERED → PLANNED`.
- At this moment active `OutboundOrderLine` is only a sorting instruction; no `Allocation` is created in cross-docking (**R4**).
- `OutboundOrder` is created with `fulfillmentChannel = CROSSDOCK`, set at creation and immutable; all its `OutboundOrderLine` have the same `fulfillmentChannel` — cross-dock and standard lines are not mixed in one `OutboundOrder` (**R5**).
- Cross-dock eligible quantity for a `CustomerOrderLine` (`crossDockEligibleQty`) is calculated in this step, when `CrossDockPickTask` is generated, as min(`sourceEligibleQty`, `demandEligibleQty`) — full formulas in **R6** (**R6**).
- Topology (1:1 or n:n) is known already here, so `TU_NUMBER`/`SSCC` of target Outbound `TU` is assigned on its first use in **STEP 2**, not only at `PACKING_SEALED`: for full 1:1 match with valid GS1 `SSCC` on source Inbound `TU`, inheritance of `TU_NUMBER` and `SSCC` is mandatory; otherwise and for n:n sorting — new number from `TUSetup`/`Sequence` and new label (**R7**).

---

### STEP 2 — Picking and control
**[Packer]**

- `CrossDockPickTask CREATED → ASSIGNED` (assigned to operator logged into cross-docking module, **R39**), then Packer begins scanning: `CrossDockPickTask ASSIGNED → IN_PROGRESS`.
- Beginning scanning transitions `OutboundOrderLine CREATED → PICKING`, and the first such event in an `OutboundOrder` transitions header `CREATED → PACKING_IN_PROGRESS` (**R8**).
- From this point, general cancellation (unrelated to shortage) is forbidden until `PACKED` — System WMS rejects such request indicating the line is being sorted, to retry after `PACKED` (**R9**).
- Packer scans `SKU`, quantity and quality (`OK`/`DAMAGED`) according to `OutboundOrderLine`.

◇ **Scan/control result for the `SKU`?**

**Path A — quantity matches, quality `OK` → [Packer]**
- Quantity goes into target Outbound `TU`. If no open target Outbound `TU` exists for this task at that moment, System WMS opens a new one and assigns `TU_NUMBER` (**R40**, numbering by **R7**).
- 🔁 If the `TU` becomes physically full before task completion, Packer closes it (`TU → PACKING_SEALED`) by RF scan and continues sorting remaining quantity into a new open Outbound `TU` within the same `CrossDockPickTask` — normal process step, not escalation (**R10**).

**Path B — lower quantity (missing) or `DAMAGED`, detected during picking or as `confirmedQty < plannedQty` at task completion → [Packer / System WMS]**
- Lower quantity requires (1) recheck and (2) Packer confirmation (**R11**); `DAMAGED` is routed to `QC` (Quality Control) (**R12**).
- Only quantity reported as `DAMAGED` is recorded as task `damagedQty` — lower quantity detected as missing is not included in `damagedQty` (**R13**).

  ◇ **`allowPartialShipment`?**

  **`= true` → [System WMS]** — automatically: sorted part `OutboundOrderLine → PACKED`, missing `CustomerOrderLine → BACKORDERED`.

  **`= false` → [Warehouse Supervisor]** — escalation, three outcomes analogous to `SHORT_PICKED` in standard picking (see **“Exceptions and alternative paths”**):
  - **“wait”** — `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, remainder waits; if `confirmedQty > 0`, System WMS creates `PutBackTask` (PROCESS 4) for that quantity — after `COMPLETED` quantity becomes ordinary `Inventory AVAILABLE` (`ATP`), never returns to Inbound `TU`/`IN_PUTAWAY` (**R14**).
  - **“cancel”** — `Warehouse Supervisor` cancels full required `CustomerOrderLine` quantity → if `confirmedQty > 0`, `PutBackTask` as in “wait”; or corrects `CustomerOrderLine` to `confirmedQty` → `OutboundOrderLine → PACKED` for sorted quantity, continues normally to `PACKING_SEALED`, without `PutBackTask` (**R15**).
  - **“permanently change `allowPartialShipment` to `true`”** — continue like automatic variant.
- In `PICKING → CANCELLED` mode (put-back/cancellation after sorting started), `OutboundOrderLine` transitions from `PICKING`, not `CREATED` (**R16**).

**Path C — unexpected `SKU` → [Packer / System WMS]**
- Goes to `QC` (**R17**).

**Path D — `TU` empty before picking (task still `ASSIGNED`, `OutboundOrderLine` still `CREATED`) → [System WMS]**
- `CrossDockPickTask ASSIGNED → CANCELLED`, related `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, Inbound `TU → LOST` (**R18**).
- As on every cancellation path: if all `OutboundOrderLine` of that `OutboundOrder` therefore reach `CANCELLED` with none `PACKED`, `OutboundOrder → CANCELLED` — general criterion, not specific to this path (**R37**).

- **Recovery after cancellation after `PACKED` (general cancellation, outside shortage).** The same recovery rule (`PutBackTask → Inventory AVAILABLE`) applies to general cancellation of a cross-dock `OutboundOrderLine` after `PACKED` — without `Allocation RELEASED`, because cross-docking creates no `Allocation` (**R19**).
- Continue to **STEP 3**.

---

### STEP 3 — Complete `CrossDockPickTask`
**[System WMS]**

- `CrossDockPickTask IN_PROGRESS → COMPLETED` when Packer reports task completion; task `confirmedQty` is then fixed and does not exceed `plannedQty`. Completion of one task does not depend on remaining tasks of the same source Inbound `TU` or whether all its ASN-declared contents are already settled — regardless of 1:1, 1:n, n:1 or n:n topology (**R20**).

◇ **After all related `CrossDockPickTask` complete, does residual unassigned physical `SKU` quantity remain?**

**Path A — NO → [System WMS]**
- Inbound `TU → CROSS_DOCKED` (Inbound state, outside Outbound canon).

**Path B — YES → [System WMS]**
- Residual is passed to Inbound Process 3; `TU` moves on sector `TRANSIT` and receives `IN_PUTAWAY` when that transport completes (**R21**); then standard Putaway without changes to this mechanism.

- In both cases System WMS passes Inbound, alongside confirmed quantity, the sum of `damagedQty` of all related `CrossDockPickTask` for that `TU` (**R22**).
- `OK` `SKU` quantity already confirmed by completed `CrossDockPickTask` (the quantity that went to Outbound `TU`) is settled separately from any residual quantity: cross-docking does not wait for physical put-away of the residual before treating the completed portion as settled (**R23**). Any later settlement of residual quantity after standard Putaway of the same Inbound `TU` is an independent event outside this process.
- Criterion for entering `CROSS_DOCKED` is only absence of residual quantity, not number of target Outbound `TU` (**R24**). `TU_NUMBER`/`SSCC` retention is a separate axis — determined only by 1:1 topology (**STEP 1**), independently of this decision.
- Source Inbound `TU` is finalized when no active `CrossDockPickTask` remains for that `TU`, ending its cross-dock participation; demand submitted after finalization does not create another cross-dock task for that `TU` (**R28**).
- Target Outbound `TU` enters `PACKING_SEALED` only under **R10** (Packer closure or automatic closure after `slaDeadline` with no active/planned tasks). One task may have used more than one Outbound `TU` during sorting, and one Outbound `TU` may have been fed by several tasks from different source Inbound `TU` (**R3**). An Outbound `TU` that loses all confirmed quantity through `PutBackTask` recovery and remains empty transitions `CREATED → CANCELLED` instead of `PACKING_SEALED` (**R34**).
- `OutboundOrderLine → PACKED` independently of target Outbound `TU` state when its quantitative settlement is closed: quantity confirmed by its single `CrossDockPickTask` (**R30**) equals `OutboundOrderLine.requiredQty` (with no missing or `DAMAGED` in progress — paths **B**/**C** separately handled in **STEP 2**). This quantitative settlement is the only `PACKED` condition; it does not depend on whether the Outbound `TU` holding the quantity has already reached `PACKING_SEALED`.
- **Note:** cross-dock Packing `TU` for an order with `allowPartialShipment = false` waits in `PACKING_SEALED` for the complete `CustomerOrder`; the channel-independent condition is `P1 R58`.
- Continue to **STEP 4**.

---

### STEP 4 — `Shipment` and onward
**[Packer / System WMS / Dispatcher]**

- Shared with PROCESS 1 (Standard Fulfillment) steps 9–13: create/extend `Shipment`, Carrier Selection, label generation, report dispatch readiness to ERP, `CarrierManifest`, dispatch — see `proces_1_standard_fulfillment.md` **STEP 9**–**STEP 13**.
- **Note:** cross-dock Packing `TU` for an order with `allowPartialShipment = false` does not enter `Shipment`; it waits in `PACKING_SEALED` for complete `CustomerOrder`, according to `P1 R58`.
- **Cross-dock-specific exception at ERP posting gate (counterpart of PROCESS 1 STEP 11A):** for a cross-dock `Shipment`, posting to ERP waits until for every source Inbound `TU` contributing confirmed `CrossDockPickTask` quantity to this `Shipment`, System WMS receives `GR_ACCEPTED` for the quantity settled through cross-docking from that `TU` (**R25**).
- Residual quantity of source Inbound `TU` (waiting for standard Putaway, **STEP 3** Path B) and any later settlement are not part of this wait and do not affect `Shipment` readiness (**R26**).
- System WMS correlates GR result only by source Inbound `TU` identifier (`sourceInboundTU`) and settlement source `GR_SETTLEMENT_SOURCE = CROSSDOCK`, without task identifier, and sets the same `grAcceptanceStatus` on all `CrossDockPickTask` of that Inbound `TU` in the system, independently of `Shipment` (**R31**). A signal matching no `sourceInboundTU` or concerning putaway settlement is rejected without changing `grAcceptanceStatus` (**R32**).
- Gate condition is calculated separately for each `Shipment`, from the full set of its own source Inbound `TU` (**R33**).
- Gate is reevaluated on every GR message, including while `Shipment` is already `POSTING_ERROR` for another reason; explicit `GR_REJECTED` does not by itself move `Shipment` to `POSTING_ERROR` — gate remains unsatisfied until `GR_ACCEPTED` for that `TU` (**R35**).
- `grAcceptanceStatus` of source Inbound `TU` blocking the gate is visible to `Warehouse Supervisor` (**R36**).

---

## End event

Dispatch through closed `CarrierManifest`; `OutboundOrderLine → SHIPPED`; `CustomerOrderLine → FULFILLED` (aggregation through PROCESS 1 Continuous Function F1 to `PARTIALLY_SHIPPED`/`SHIPPED` at `CustomerOrder` level). Alternatively, exits without dispatch — Inbound `TU → CROSS_DOCKED`/`IN_PUTAWAY`/`LOST`, return `CustomerOrderLine → BACKORDERED`.

---

## Sequence diagram

### 6.8 Outbound Crossdock

```mermaid
sequenceDiagram
    participant INB as Inbound Process
    participant WMS as System WMS
    participant PA as Packer
    participant ERP as ERP
    participant DI as Dispatcher
    INB->>WMS: Inbound TU in IN_CROSS_DOCK
    WMS->>WMS: match CustomerOrderLine BACKORDERED
    alt Case 1 - full 1:1 match
        WMS->>WMS: CrossDockPickTask + OutboundOrderLine CREATED
    else Case 2 - n:n sorting
        WMS->>WMS: CrossDockPickTask + OutboundOrderLine CREATED per line
    end
    alt E - Inbound TU empty before picking
        WMS->>WMS: CrossDockPickTask and lines CANCELLED, CustomerOrderLine BACKORDERED
        WMS->>INB: Inbound TU LOST
    else Inbound TU contains quantity to pick
        WMS->>PA: CrossDockPickTask to pick
        PA->>WMS: scan SKU, quantity and OK/DAMAGED control
        alt A - quantity matches and OK
            WMS->>WMS: SKU to target Outbound TU (TU_NUMBER/SSCC assigned here already, R7)
            Note over WMS: PACKING_SEALED - Packer closure (full) or automatic after slaDeadline with no active/planned tasks (R10)
        else B - lower quantity after confirmation
            WMS->>WMS: OutboundOrderLine CANCELLED, CustomerOrderLine BACKORDERED
        else C - DAMAGED
            PA->>WMS: place on QC
            WMS->>WMS: OutboundOrderLine CANCELLED, CustomerOrderLine BACKORDERED
        else D - unexpected SKU
            PA->>WMS: place on QC
        end
        WMS->>WMS: CrossDockPickTask COMPLETED
        WMS->>WMS: OutboundOrderLine PACKED for sorted quantities
        alt Path A - no residual (1:1, 1:n, n:1 or n:n without residual)
        WMS->>INB: Inbound TU CROSS_DOCKED, + damagedQty
            WMS->>WMS: Outbound TU PACKING_SEALED (number assigned earlier during picking)
            INB->>ERP: GR (full settlement, one version)
        else Path B - physical residual remains
            WMS->>WMS: Outbound TU PACKING_SEALED (number assigned earlier, R7)
        WMS->>INB: residual to TRANSIT, Inbound TU IN_PUTAWAY, + damagedQty
            INB->>ERP: GR (cross-dock settlement - quantity already confirmed by CrossDockPickTask)
            Note over INB: putaway settlement of residual quantity occurs later, after standard Putaway - outside this process, does not affect Shipment
        end
        WMS->>WMS: Shipment, Carrier Selection, label
        Note over WMS,ERP: ERP gate waits for GR_ACCEPTED of cross-dock settlement for each source Inbound TU (R25)
        Note over WMS: GR_REJECTED does not change Shipment state - gate remains unsatisfied until GR_ACCEPTED (R35); grAcceptanceStatus visible to Warehouse Supervisor (R36)
        WMS->>ERP: POST Shipment
        WMS->>DI: Shipment ready for CarrierManifest
        DI->>WMS: CarrierManifest CLOSED, dispatch
    end
```

## Data objects

| Object | Key fields | Status(es) |
|---|---|---|
| Inbound `TU` (`ELEMENTARY`) | — (Inbound canon, outside `model_stanow_outbound.md`) | `IN_CROSS_DOCK → CROSS_DOCKED`; alternatively `IN_CROSS_DOCK → IN_PUTAWAY` (residual) or `→ LOST` (empty `TU`) |
| `CrossDockPickTask` | `sourceInboundTU`, `SKU`, `plannedQty`, `confirmedQty`, `damagedQty`, `targetOutboundOrderLine`, `grAcceptanceStatus` | `CREATED → ASSIGNED → IN_PROGRESS → COMPLETED`; alternative terminal: `CANCELLED` |
| `CustomerOrderLine` | `crossDockEligibleQty` | `BACKORDERED → PLANNED → PARTIALLY_FULFILLED`/`FULFILLED`; returns to `BACKORDERED` on shortage |
| `OutboundOrder` | `fulfillmentChannel = CROSSDOCK`, `priority`, `slaDeadline` (inherited from parent `CustomerOrder`, **R43**) | `CREATED → PACKING_IN_PROGRESS → PACKED → READY_FOR_DISPATCH → DISPATCHED → COMPLETED`; alternative terminal: `CANCELLED` (no `ALLOCATION_IN_PROGRESS`/`PICKING_IN_PROGRESS` phase — skipped in cross-docking) |
| `OutboundOrderLine` | `requiredQty`; `pickedQty` (not applicable — no formal `PickTask`) | `CREATED → PICKING → PACKED → SHIPPED`; alternative terminal: `CANCELLED` |
| Outbound `TU` (`PackUnit`) | `TU_NUMBER`, `SSCC` | `CREATED → PACKING_SEALED` (directly, skips `IN_PICKING`/`READY_TO_PACK`/`PACK_QUALIFIED`) `→ IN_SHIPMENT → DISPATCHED`; alternative terminal: `CANCELLED` |
| `Allocation` | — | **not created** in cross-docking (**R4**) |
| `Inventory` (Outbound scope) | — | outside standard reservation mechanism — quantity settled directly through `CrossDockPickTask`/Inbound GR, without `Allocation RESERVED` |

---

## Exceptions and alternative paths (PROCESS 5 — Cross-cutting exception handling)

This section implements PROCESS 5 — cross-cutting exceptions shared by Outbound processes, without its own process file — `SHORT_ALLOCATED`, `SHORT_PICKED`, `ON_HOLD`, cancellation, no Carrier Selection result, label error; see `proces_1_standard_fulfillment.md` for remainder.

| Condition | Behavior | Effect |
|---|---|---|
| Shortage/`DAMAGED` during picking, `allowPartialShipment = false` (**STEP 2**, Path B) | Escalate to `Warehouse Supervisor`, three results: “wait” (`PutBackTask` for `confirmedQty > 0`), “cancel” (full or correction to `confirmedQty`), “permanently change `allowPartialShipment` to `true`” | See **R14**–**R15** |
| Shortage/`DAMAGED`, `allowPartialShipment = true` | Automatically: sorted part `→ PACKED`, missing `CustomerOrderLine → BACKORDERED` | No escalation |
| Empty `TU` before picking (**STEP 2**, Path D) | `CrossDockPickTask → CANCELLED`, `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, Inbound `TU → LOST` (**R18**) | If all `OutboundOrder` lines `CANCELLED` with none `PACKED` → `OutboundOrder → CANCELLED` — general criterion (**R37**) |
| General cancellation attempt (unrelated to shortage) while sorting (`CrossDockPickTask IN_PROGRESS`) | System WMS **rejects** the request indicating the line is being sorted | May retry after `PACKED` (then as standard general cancellation, PROCESS 1 “Exceptions and alternative paths”) (**R9**) |
| Explicit `GR_REJECTED` for source Inbound `TU` at ERP gate (**STEP 4**) | Gate **R25** remains unsatisfied; `Shipment` does not change state for this reason | `grAcceptanceStatus` visible to `Warehouse Supervisor` (**R36**); gate becomes satisfied after `GR_ACCEPTED` for that `TU` (**R35**) |

---

## Business rules

- **R1** — System WMS generates `CrossDockPickTask` from `CustomerOrderLine BACKORDERED`, ordered according to applicable order-priority queue (warehouse parameter).
- **R2** — Full 1:1 match occurs when all ASN-declared contents of source Inbound `TU` cover demand of `CustomerOrderLine` in `BACKORDERED` jointly satisfying common-dispatch conditions: same customer, same delivery address, compatible `priority`, identical `slaDeadline`. With `allowPartialShipment = true`, demand may come from several `CustomerOrder` of the same customer; with `allowPartialShipment = false`, only from one. Source Inbound `TU` may contain multiple `SKU` — commonality concerns destination, not contents.
- **R3** — In n:n sorting, one Outbound `TU` may collect `SKU` from several Inbound `TU`, for `OutboundOrderLine` matching customer/address/`priority` and identical `slaDeadline` (`slaDeadline` match is strict, not “close” — required to bind unambiguously to automatic closure threshold **R10**).
- **R4** — Cross-docking creates no `Allocation` — active `OutboundOrderLine` is only a sorting instruction.
- **R5** — `OutboundOrder.fulfillmentChannel = CROSSDOCK` is set at creation and immutable; cross-dock and standard lines are not mixed in one `OutboundOrder`.
- **R6** — `crossDockEligibleQty` for a `CustomerOrderLine` = min(`sourceEligibleQty`, `demandEligibleQty`), calculated when generating `CrossDockPickTask`. `sourceEligibleQty` = ASN-declared quantity of source Inbound `TU`/`SKU` minus sum of `plannedQty` of active `CrossDockPickTask` on that `TU`/`SKU`, minus sum of `confirmedQty` of completed `CrossDockPickTask` on that `TU`/`SKU`, minus sum of `damagedQty` of completed `CrossDockPickTask` on that `TU`/`SKU`. `demandEligibleQty` = `CustomerOrderLine.Quantity` minus its `ATPReservation`, minus sum of `requiredQty` of all its `OutboundOrderLine` not `CANCELLED`, regardless of channel. Both forms of demand protection are subtracted — soft (`ATPReservation`) and hard (`requiredQty`) — because `P1 R11` transfers quantity from first to second, and `P1 R59` allows splitting one `CustomerOrderLine` across multiple `OutboundOrderLine`. Cross-dock `OutboundOrderLine` are created together with their `CrossDockPickTask` (**R30**), so they are already included in that sum and are not subtracted separately.
- **R7** — `TU_NUMBER`/`SSCC` of target Outbound `TU` is assigned on first use in **STEP 2**, not only at `PACKING_SEALED`, because 1:1/n:n topology is known when task is generated. In full 1:1 match, when source Inbound `TU` `TU_NUMBER` is valid GS1 `SSCC`, target Outbound `TU` must inherit `TU_NUMBER` and `SSCC`. If source Inbound `TU` `TU_NUMBER` is not valid GS1 `SSCC`, target Outbound `TU` receives a new number according to `TUSetup`/`Sequence` and a new label. In n:n sorting, target Outbound `TU` always receives a new number from `Sequence` referenced by its `TUSetup`.
- **R8** — Beginning scanning (`CrossDockPickTask → IN_PROGRESS`) transitions `OutboundOrderLine → PICKING`; first such event in an `OutboundOrder` transitions header `CREATED → PACKING_IN_PROGRESS`.
- **R9** — From start of sorting until `PACKED`, general cancellation (unrelated to shortage) is forbidden; System WMS rejects request to retry after `PACKED`.
- **R10** — Target Outbound `TU` transitions to `PACKING_SEALED` in one of two cases: (1) Packer closes it by RF scan when physically full before task completion and continues sorting into a new open `TU` in same `CrossDockPickTask` — normal step, not escalation; (2) System WMS closes it automatically when `slaDeadline` is reached (or passed) and no active or planned `CrossDockPickTask` points to it as target. Before `slaDeadline`, absence of active/planned tasks alone does not close `TU` — cross-dock goods for same customer/address/`priority`/`slaDeadline` (**R3**) may still arrive in another Inbound delivery the same or following days, so early closure would reduce `TU` fill; `slaDeadline` is the hard waiting boundary. Because one Outbound `TU` may be fed by several tasks from different source Inbound `TU` (**R3**), completion of one task does not by itself close it.
- **R11** — Quantity lower than planned requires rechecking and Packer confirmation before being registered as missing.
- **R12** — `DAMAGED` goods go to `QC`.
- **R13** — Only quantity reported as `DAMAGED` enters task `damagedQty`; missing lower quantity does not enter `damagedQty`.
- **R14** — “Wait” result on shortage (`allowPartialShipment = false`): `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`; for `confirmedQty > 0`, create `PutBackTask` (PROCESS 4), after which quantity becomes `Inventory AVAILABLE`, never returns to Inbound `TU`/`IN_PUTAWAY`.
- **R15** — “Cancel” result on shortage: full `CustomerOrderLine` cancellation (with `PutBackTask` for `confirmedQty > 0`) or `Warehouse Supervisor`-controlled correction of `CustomerOrderLine.Quantity` to `confirmedQty` together with `OutboundOrderLine.requiredQty` to same value (`P1 R69`), no `PutBackTask`, continuing to `PACKING_SEALED`.
- **R16** — When cancelling a cross-dock `OutboundOrderLine` after sorting starts, transition to `CANCELLED` is from `PICKING`, not `CREATED`.
- **R17** — Unexpected `SKU` in source `TU` goes to `QC`, without starting shortage mechanism.
- **R18** — Empty `TU` before picking: `CrossDockPickTask → CANCELLED`, related `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, Inbound `TU → LOST`. Whole-`OutboundOrder` closure criterion in this case — see **R37** (general, not path-specific).
- **R19** — `PutBackTask → Inventory AVAILABLE` recovery also applies to general cancellation of cross-dock `OutboundOrderLine` after `PACKED`, without `Allocation RELEASED` (because `Allocation` does not exist in cross-docking).
- **R20** — `CrossDockPickTask → COMPLETED` when Packer reports task completion; `confirmedQty` is then fixed and does not exceed `plannedQty`. Condition concerns only this task — completion does not depend on other `CrossDockPickTask` of same source Inbound `TU` or whether all ASN-declared contents are settled, regardless of topology.
- **R21** — Residual unassigned quantity after all related `CrossDockPickTask` complete is passed to Inbound Process 3 (Putaway); `TU` moves on sector `TRANSIT` and receives `IN_PUTAWAY` when that transport completes (**R28**), then standard Putaway. Residual quantity per `SKU` is determined by Inbound Process 3 (Putaway) — definition and quantity control belong there — from components supplied by this process: ASN-declared source Inbound `TU` quantity minus sum of `confirmedQty` **and** sum of `damagedQty` of all completed `CrossDockPickTask` of that `TU` (**R22**). Damaged quantity reduces residual because it physically leaves source `TU` to `QC` (**R12**) and is no longer expected there; omitting it overstates residual expected at Putaway. Note formula similarity: when residual is determined, i.e. after source-`TU` finalization with no active tasks (**R28**), sum of `plannedQty` of active tasks is zero and this quantity numerically matches `sourceEligibleQty` (**R6**) — at other times formulas differ and are not interchangeable.
- **R22** — System WMS passes Inbound the sum of `damagedQty` of all related `CrossDockPickTask` for a `TU`, whether `TU` reaches `CROSS_DOCKED` or `IN_PUTAWAY`.
- **R23** — `SKU OK` quantity already confirmed by completed `CrossDockPickTask` is settled separately from any residual quantity — process does not wait for physical put-away of residual.
- **R24** — Criterion for Inbound `TU` transition to `CROSS_DOCKED` is only absence of residual quantity, not number of target Outbound `TU`.
- **R25** — ERP-posting gate for cross-dock `Shipment` waits for `GR_ACCEPTED` for each source Inbound `TU` contributing confirmed `CrossDockPickTask` quantity to that `Shipment`.
- **R26** — Residual quantity of source Inbound `TU` and its later settlement do not affect `Shipment` readiness for ERP posting.
- **R28** — Source Inbound `TU` finalization occurs when no active `CrossDockPickTask` remains for that `TU`, ending its cross-dock participation. With zero residual, finalization sets `CROSS_DOCKED`. With nonzero residual, finalization is notification to pass `TU` to Inbound Process 3 (Putaway); this process does not set `IN_PUTAWAY` — Inbound sets it when its own sector-`TRANSIT` transport completes, so until then `TU` remains `IN_CROSS_DOCK`. Demand submitted after finalization does not create another `CrossDockPickTask` for that `TU` — it is handled normally from stock after Putaway completes.
- **R29** — The same physical quantity of source Inbound `TU`/`SKU` may be `planned` or `confirmed` in at most one active `CrossDockPickTask`; `plannedQty` reserves it for confirmation. Cross-dock quantity lock is on `CrossDockPickTask`, not `OutboundOrderLine`.
- **R30** — Generation of `CrossDockPickTask` and `OutboundOrderLine` for a source Inbound `TU` is a one-time event triggered by its transition to `IN_CROSS_DOCK` (**STEP 1**), matching only `CustomerOrderLine BACKORDERED` existing at that time. There is no mechanism to generate additional `CrossDockPickTask` for an already-created `OutboundOrderLine` or automatically update its `requiredQty` after creation — also in the window between `IN_CROSS_DOCK` and finalization of same source `TU` (**R28**). Explicit controlled correction of `CustomerOrderLine.Quantity` and `OutboundOrderLine.requiredQty` in **R15** remains allowed according to `P1 R69`. Every cross-dock `OutboundOrderLine` is created with exactly one `CrossDockPickTask`; quantity lock remains on `CrossDockPickTask`, not `OutboundOrderLine` (**R29**).
- **R31** — For a GR message received from ERP for one elementary Inbound `TU`, System WMS correlates only by source Inbound `TU` identifier (`sourceInboundTU`) and settlement source `GR_SETTLEMENT_SOURCE = CROSSDOCK`, without task identifier. After matching, System WMS sets same `grAcceptanceStatus` (`GR_ACCEPTED` or `GR_REJECTED`) on all `CrossDockPickTask` in system whose `sourceInboundTU` matches that Inbound `TU` — independently of which `Shipment` individual tasks contributed `confirmedQty` to, including when one Inbound `TU` fed tasks belonging to several different `Shipment`.
- **R32** — System WMS rejects GR signal and changes no task `grAcceptanceStatus` when indicated Inbound `TU` matches no `CrossDockPickTask.sourceInboundTU` or signal concerns settlement other than cross-dock. Cross-dock versus putaway settlement is distinguished only by settlement source (`GR_SETTLEMENT_SOURCE`), never by settlement version number — version numbering is separate per source, so “version 1” does not identify cross-dock settlement.
- **R33** — System WMS calculates gate **R25** separately for each `Shipment`, from full set of its own source Inbound `TU`. `GR_ACCEPTED` for one source Inbound `TU` is insufficient if same `Shipment` contains contributions from other source Inbound `TU`.
- **R34** — When a target Outbound `TU` created during active `CrossDockPickTask` loses all confirmed quantity through `PutBackTask` recovery and remains empty without any other confirmed line, while no active or planned `CrossDockPickTask` points to it as target, it transitions `CREATED → CANCELLED` and never enters `PACKING_SEALED`. While such a task points to it, `TU` remains `CREATED` as empty open target. Unlike closure into `PACKING_SEALED` (**R10**), cancellation does not wait for `slaDeadline` — an empty `TU` has nothing to hold, so this asymmetry versus **R10** is intentional.
- **R35** (new) — Gate **R25** is reevaluated on every GR message (`GR_ACCEPTED` or `GR_REJECTED`) for any required source Inbound `TU`, regardless of previous result for that `TU` or `Shipment` state — including when `Shipment` is in `POSTING_ERROR` for another reason (for example ERP rejection, PROCESS 1 **STEP 11A**). Explicit `GR_REJECTED` does not by itself move `Shipment` into `POSTING_ERROR` — gate remains unsatisfied (analogous to distinction between unsatisfied precondition and actual posting error, Inbound model `D54`); when same `TU` later receives `GR_ACCEPTED`, gate is satisfied like any other, without a separate recovery path.
- **R36** (new) — `grAcceptanceStatus` of source Inbound `TU` blocking gate **R25** is visible to `Warehouse Supervisor` (read at `CrossDockPickTask`/waiting `Shipment` level), without introducing a new state, object or separate escalation mechanism — exposure of existing field only.
- **R37** (new) — `OutboundOrder → CANCELLED` when all its `OutboundOrderLine` reach `CANCELLED` with none `PACKED` — regardless of path cancelling individual lines (empty `TU` before picking — **R18**; shortage with “wait” or full “cancel” — **R14**/**R15**; general cancellation during sorting is blocked until `PACKED` — **R9** — so it cannot itself lead to this state before `PACKED`).
- **R39** (new) — `Warehouse Operator` selects cross-docking module on RF terminal. System WMS assigns next `CrossDockPickTask` in same order as `PickTask` (warehouse parameter, `P1 R14`) when operator has no active warehouse task of any type. Cross-docking module has no work-location selection analogous to zone in picking — process does not model physical sorting location.
- **R40** (new) — System WMS guarantees active `CrossDockPickTask` always has an available target for placing confirmed quantity — no state exists in which active task has nowhere to place goods. Mechanism: if at placement time no open target Outbound `TU` exists for the task — regardless of cause: physically full `TU` closure (**R10**), empty-`TU` cancellation after full recovery (**R34**), or no `TU` at start of sorting — System WMS opens a new Outbound `TU` at that exact moment and assigns `TU_NUMBER` according to **R7**. Creation is therefore lazy: on first `SKU` placement into that `TU`, not pre-emptively when prior one is closed or cancelled.
- **R41** (new) — Only Inbound `TU` type `ELEMENTARY` enters this process; sorting always uses an `ELEMENTARY` `TU`. `AGGREGATE` `TU` never reaches `IN_CROSS_DOCK`: when SKU needed by `BACKORDERED` line is declared in a `TU` attached to an aggregate, Inbound Process 2 forces disintegration of aggregate before transport and passes only `ELEMENTARY` `TU`, each by its own route. This process has no path for `AGGREGATE` `TU`.
- **R42** (new) — Qualification by Inbound Process 2 is a transport precondition, not a binding allocation. Binding match to `CustomerOrderLine BACKORDERED` is created only when source `TU` transitions to `IN_CROSS_DOCK` (**R30**), using priority queue current at that moment — demand may disappear during physical transport to cross-dock area. When **R30** finds no match, System WMS creates no `CrossDockPickTask`, and full ASN-declared source `TU` quantity is residual: `TU` finalizes immediately (**R28**) and is passed to Inbound Process 3 with residual equal to full declaration; it receives `IN_PUTAWAY` only after sector-`TRANSIT` transport completes, so remains `IN_CROSS_DOCK` until then (**R21**). Recovery after demand loss when a task was already created (**R14**) does not cover this case.
- **R43** — Cross-dock `OutboundOrder` inherits `priority` and `slaDeadline` from parent `CustomerOrder`. Grouping lines into one cross-dock `OutboundOrder` is allowed only for matching customer, matching delivery address, matching `priority` and identical `slaDeadline`. With `allowPartialShipment = false`, grouping includes only lines of one `CustomerOrder`.

## Design notes (for verification)

- **Stage 2 (outside version 1):** consolidation of Outbound `TU` after `PACKED` for cross-docking (`READY_TO_PACK` path) — remains open in `BACKLOG.md` B12.

## Relationship to adjacent processes

- **Preceding:** Inbound Process 2 (Unloading) — passes Inbound `TU` (`ELEMENTARY`) in `IN_CROSS_DOCK`.
- **Following:** no direct WMS process for settled quantity (physical pickup outside boundary, shared with PROCESS 1 from **STEP 4** onward); Inbound Process 3 (Putaway) takes any residual quantity (Inbound `TU → IN_PUTAWAY`).

## Change history

- **1.13 (2026-08-28)** — Closed `BACKLOG.md` B22: `demandEligibleQty` term in **R6** now subtracts both forms of demand protection — `ATPReservation` and sum of `requiredQty` of non-cancelled `OutboundOrderLine` of the line. Previous wording subtracted only `ATPReservation`, so quantity transferred to hard reservation by `P1 R11` disappeared from formula and could be granted again through cross-dock. Removed undefined phrase “missing quantity of `CustomerOrderLine` in `BACKORDERED`” and separate term for assignments of other `CrossDockPickTask`, absorbed by `requiredQty` sum. No changes to `sourceEligibleQty` or other rules.
- **1.12 (2026-08-28)** — WP-09: added note in **STEP 3** and **STEP 4** that cross-dock Packing `TU` of `allowPartialShipment = false` order waits in `PACKING_SEALED` for complete `CustomerOrder` according to shared guard `P1 R58`; no duplicated rule and no new exception row.
- **1.11 (2026-08-27)** — Added `OutboundOrderLine.requiredQty` to “Data objects” and rewired `PACKED` condition in **STEP 3** and **R30** to it. **R15** explicitly updates `requiredQty` together with controlled `CustomerOrderLine.Quantity` correction; **R30** still excludes additional task generation and automatic update, consistent with `P1 R69`.
- **1.10 (2026-08-27)** — According to `DEC-L52`, restored full 1:1 matching criterion as common destination of all declared contents of source Inbound `TU`, allowing multiple `SKU` and `allowPartialShipment` rules; restored plural `OutboundOrderLine` in STEP 1. New **R43**, consistent with `DEC-L54`, defines inheritance of `priority` and `slaDeadline` from parent `CustomerOrder` and strict grouping conditions.
- **1.9 (2026-08-23)** — Separated source Inbound `TU` finalization from assigning `IN_PUTAWAY`, after discrepancy reported by Inbound session. **R28**, **R21**, **R42** had abbreviated finalization as transition to `IN_PUTAWAY`; in fact with nonzero residual, finalization is notification to pass `TU` to Inbound Process 3, while Inbound assigns `IN_PUTAWAY` when its own sector-`TRANSIT` transport completes — until then `TU` remains `IN_CROSS_DOCK`. STEP 3 prose clarified. No new rules, requirements or scenarios.
- **1.8 (2026-08-23)** — Closed Inbound→Outbound Crossdock boundary after changes reported by Inbound session. New **R41**: only Inbound `TU` `ELEMENTARY` enters process, `AGGREGATE` never reaches `IN_CROSS_DOCK` (Inbound forces disintegration before transport). New **R42**: Inbound qualification is transport precondition, not binding allocation — binding match occurs at `IN_CROSS_DOCK` (**R30**), and with zero match `TU` finalizes immediately toward `IN_PUTAWAY` with entire quantity residual. Added `(ELEMENTARY)` qualifier to “Previous process” header.
- **1.7 (2026-08-23)** — Closed V3-CD audit item concerning residual quantity (Phase 1c item C). **R21** supplemented with residual components — ASN-declared source Inbound `TU` quantity minus sum `confirmedQty` and `damagedQty` of completed tasks — while stating definition/quantity control belong to Inbound Process 3 and this process supplies components. Previously “residual quantity” was used in **R21**, **R23**, **R24** and STEP 3 without definition despite **R24** basing `CROSS_DOCKED` transition on it. Added warning that it is not interchangeable with `sourceEligibleQty` (**R6**): formulas converge only at `TU` finalization when active-task `plannedQty` sum is zero. No behavior change — self-contained wording completed.
- **1.6 (2026-08-23)** — Closed V3-CD-02 (PutBack vs shared target Outbound `TU`). **R34** supplemented with condition of no active/planned `CrossDockPickTask` pointing to `TU` — prior guard was quantity-only and could cancel a `TU` still targeted by another task (dangling target reference). Explicitly recorded that no `slaDeadline` condition on cancellation is intentional asymmetry versus **R10**. New **R40** target-continuity invariant: System WMS guarantees active `CrossDockPickTask` always has a target — if no open `TU`, opens new and assigns `TU_NUMBER` (**R7**); mechanism previously existed only for full-`TU` closure branch of **R10** (**STEP 2** updated), now applies regardless of cause. R39 already used by RF task take-up, hence next free number. Metric header “Source” reworded to “Origin” (`DEC-A14`).
- **1.5 (2026-08-23)** — Warehouse-task take-up rules (`BACKLOG.md` B10): new **R39** — operator selects cross-docking module on RF terminal, System WMS assigns next `CrossDockPickTask` in same order as `PickTask`, without zone selection (deliberate negative decision). Updated “Participants” and **STEP 2**. Full rationale in `decyzje_outbound_wms.md` `DEC-L42`/`DEC-L44`.
- **No version change (2026-08-23)** — operator-pool concept withdrawn (product-owner decision): removed **R38** restored on 2026-08-22, pool-assignment fragment from “Participants”, and task-pool reference from STEP 2. Number **R38** intentionally remains unused. Task assignment mechanism to be described separately.
- **1.4 (2026-08-22)** — Fixed migration completeness gap (not consistency gap): global `R75` from `propozycja_procesow_outbound.md` §7 had never been moved into any process file — restored as new local **R38** (precedence of `CrossDockPickTask` over `PickTask` when operator assigned to both pools). Updated “Participants” (`Warehouse Supervisor`) with **R38** reference.
- **1.3 (2026-08-22)** — Closed V3-CD audit (owner decisions #1–#3): **R30** rewritten — removed incorrect statement about feeding one `OutboundOrderLine` by multiple `CrossDockPickTask` from different source `TU` (generation is one-time, per source `TU`, STEP 1); added closed generation scope. **R10** extended with hard time limit (`slaDeadline`) for automatic Outbound `TU` closure — prevents early closure when cross-dock goods arrive in later Inbound deliveries to same address over one or several days; **R3** tightened to identical `slaDeadline`. **STEP 3** split: Outbound `TU` closure (`PACKING_SEALED`, **R10**) and `OutboundOrderLine → PACKED` (quantitative settlement only, independent of `TU` state) described separately. **R27** removed — explicit `GR_REJECTED` no longer transitions `Shipment → POSTING_ERROR`; new **R35** (reevaluate **R25** gate on every GR message, including after earlier `POSTING_ERROR`) and **R36** (`grAcceptanceStatus` visibility for `Warehouse Supervisor`, option C1). **R18** narrowed to empty-before-pick effect; general `OutboundOrder → CANCELLED` criterion (all lines `CANCELLED`, no `PACKED`) separated into new **R37**, applying regardless of cancellation path (`DEC-L13`). Updated sequence diagram §6.8, “Participants” (`grAcceptanceStatus` visibility) and exception table.
- **1.2 (2026-08-22)** — Fixed propagation gaps revealed by consistency audit: `R20` restored to correct subject (Packer reports task completion; full declared-content criterion concerns source Inbound `TU`, not task); `R6` completed with full `crossDockEligibleQty` formula using `sourceEligibleQty`/`demandEligibleQty`; `R7` completed with `TU_NUMBER`/`SSCC` inheritance/renumbering; `R10` extended with second `PACKING_SEALED` trigger. New `R28`–`R34`: source-`TU` finalization and no recrossdock after it, task-level quantity lock, multiple tasks for one `OutboundOrderLine`, GR correlation by `sourceInboundTU` and `GR_SETTLEMENT_SOURCE`, reject unmatched/putaway signal, gate calculated per `Shipment`, empty Outbound `TU` after full `PutBackTask`. Sequence diagram moved from global to local rule numbering.
- **No version change (2026-08-22)** — editorial consolidation of P5 into “Exceptions and alternative paths (PROCESS 5 — Cross-cutting exception handling)”, without separate file and without changing `Rn` numbering.
- **1.1 (2026-08-22)** — Moved sequence diagram §6.8 from `propozycja_procesow_outbound.md` during B16 archival.
- **1.0 (2026-08-18)** — Baseline version. Expanded `propozycja_procesow_outbound.md` v1.29 §3.2 (steps 1–4) and P2 part of §3.5 into step-by-step format compliant with `SZABLON_PROCESU.md`, with local numbering `R1`–`R27`. Implementation of `BACKLOG.md` B5.
