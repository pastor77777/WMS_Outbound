# WMSAI Outbound — Requirements

**Project:** Outbound order fulfillment
**Date:** 28.08.2026 | **Author:** Solution Analyst (Solution Architect competencies)
**Source:** `proces_1_standard_fulfillment.md` v1.20, `proces_2_outbound_crossdock.md` v1.13, `proces_3_reservation_release.md` v1.2, `proces_4_physical_putback.md` v1.2, `model_stanow_outbound.md` v1.19. Cross-cutting exceptions (`FR-P5-*`) come from the “Exceptions and alternative paths” sections of `proces_1` and `proces_2` — there is no separate process file for PROCESS 5.
**Contents:** 1a — functional · 1b — integration · 1c — non-functional (outside B6) · 1d — concurrency.

## Writing convention (EARS)

Requirements use **EARS** (Easy Approach to Requirements Syntax). Sentence patterns:

- **Universal:** “The WMS System must <response>.”
- **Event-driven:** “**When** <trigger>, the WMS System must <response>.”
- **State-driven:** “**While** <state>, the WMS System must <response>.”
- **Conditional (unwanted):** “**If** <condition>, then the WMS System must <response>.”
- **Configurable:** “**Where** <function enabled>, the WMS System must <response>.”

The actor appears in the trigger. Every requirement contains: **ID · Category · Content (EARS) · Source · Acceptance criteria (Given/When/Then)**. Acceptance criteria use Gherkin syntax: `Given / When / Then / And`.

---

# 1a. Functional requirements

## P1 — Standard Fulfillment

### FR-P1-01 — Zero ATP and reservation queue
*Category:* event-driven · *Source:* P1 R1–P1 R2
**When** `ATP` availability for an accepted `CustomerOrderLine` is zero, the WMS System must set `ATPReservation = 0`, assign `BACKORDERED` and place the line in the queue according to the warehouse parameter.
```gherkin
Scenario: No ATP creates waiting
  Given `CustomerOrderLine` is accepted and no ATP is available
  When the WMS System calculates the soft reservation
  Then the line has `ATPReservation = 0` and status `BACKORDERED`
  And it occupies a place in the queue according to warehouse configuration
```

### FR-P1-02 — ATP queue recalculation
*Category:* event-driven · *Source:* P1 R3–P1 R4
**When** ERP confirms posting of an Inbound `TU` for the SKU or — in the priority variant — a new `CustomerOrder ACCEPTED` appears, the WMS System must recalculate the `ATPReservation` queue without disturbing `Allocation RESERVED`.
```gherkin
Scenario: ERP confirmation triggers recalculation
  Given lines are waiting for ATP of the same SKU
  When ERP confirms `POST` of the relevant Inbound `TU`
  Then the WMS System recalculates soft reservations
  And does not take an existing `Allocation RESERVED`
```

### FR-P1-03 — Skipping ON_HOLD and full coverage
*Category:* state-driven · *Source:* P1 R5–P1 R6
**While** a `CustomerOrder` is `ON_HOLD`, or with `allowPartialShipment = false` at least one of its `CustomerOrderLine` objects is not fully covered — where line coverage is `ATPReservation` plus the sum of `requiredQty` of its non-cancelled `OutboundOrderLine` objects in `PACKED` or beyond, regardless of channel — the WMS System must skip the order in planning and must not create a standard `OutboundOrder`; with complete cross-dock coverage no standard `OutboundOrder` is created at all, and with partial coverage it covers only the uncovered quantity.
```gherkin
Scenario: Order does not enter planning
  Given the order is `ON_HOLD` or at least one of its lines is not fully covered
  When cyclic planning starts
  Then `OutboundOrder` is not created
  And the order remains `ACCEPTED` with `WARNING` when the cause is incomplete reservation
```

### FR-P1-04 — Order grouping
*Category:* conditional · *Source:* P1 R7–P1 R8
**If** lines are to be grouped into an `OutboundOrder`, then the WMS System must require compatible customer, address, priority and `slaDeadline` within tolerance; for `allowPartialShipment = false` it must create exactly one non-aggregated `OutboundOrder`.
```gherkin
Scenario: Different customer blocks grouping
  Given two lines belong to different customers
  When the WMS System plans an `OutboundOrder`
  Then the lines are not grouped together
  And an order without partial shipments is not aggregated with another order
```

### FR-P1-05 — OutboundOrder attributes and no reallocation
*Category:* event-driven · *Source:* P1 R9–P1 R10
**When** the WMS System creates an `OutboundOrder`, it must set the most urgent `priority`/`slaDeadline` and immutable `fulfillmentChannel`, and after a `PickTask` is created it must not reallocate inventory because of priority.
```gherkin
Scenario: Priority does not take started fulfillment
  Given a `PickTask` already exists
  When a higher-priority order appears
  Then the existing fulfillment is not reallocated
  And the new order competes only for unreserved inventory
```

### FR-P1-06 — ATPReservation transfer and manual correction
*Category:* event-driven · *Source:* P1 R11–P1 R12
**When** `Allocation RESERVED` is created, the WMS System must subtract its quantity from `ATPReservation` and maintain the hard reservation until `SHIPPED`; when `Warehouse Supervisor` reduces `ATPReservation`, it must preserve `CustomerOrder` status.
```gherkin
Scenario: Soft reservation becomes hard
  Given the line has a positive `ATPReservation`
  When `Allocation RESERVED` is created
  Then the corresponding quantity disappears from `ATPReservation`
  And manual reduction of the remaining reservation does not change order status
```

### FR-P1-07 — PickTask per zone and ordering
*Category:* configurable · *Source:* P1 R13–P1 R14
**Where** an `OutboundOrder` covers multiple zones, the WMS System must create multiple `PickTask` objects, each for one operator, and present them according to configured `slaDeadline`/`priority` ordering with ties resolved by submission order.
```gherkin
Scenario: Multi-zone picking
  Given an order requires picking from two zones
  When the WMS System generates work
  Then two `PickTask` objects are created, one per zone
  And their order follows warehouse configuration
```

### FR-P1-08 — TU limit and direct-pack declaration
*Category:* event-driven · *Source:* P1 R15–P1 R16
**When** the Picker scans the first Picking `TU`, the WMS System must allow a binding `directPackDeclared` declaration, and during picking it must block exceeding `TUSetup.maxWeight` and set `PICK_FULL` when that limit is reached.
```gherkin
Scenario: Declaration is irreversible
  Given the Picker declared direct pack on the first TU scan
  When picking has started
  Then the declaration cannot be revoked for that `TU`
  And exceeding `TUSetup.maxWeight` is blocked
```

### FR-P1-09 — Automatic direct pack and deviation
*Category:* conditional · *Source:* P1 R17–P1 R18
**If** `directPackDeclared = true` and the `TU` meets issuance thresholds, then the WMS System must automatically perform transitions to `PACKING_SEALED`/`PACKED`; a Packer deviation from the suggestion must require `Warehouse Supervisor` approval.
```gherkin
Scenario: Direct pack without Packer
  Given direct-pack picking ended on an issuable TU
  When the WMS System evaluates issuance thresholds
  Then the TU transitions to `PACKING_SEALED`, and the line to `PACKED`
  And the Packer does not participate in this path
```

### FR-P1-10 — Packing qualification and compatibility
*Category:* conditional · *Source:* P1 R19–P1 R20
**If** a Picking `TU` meets issuance thresholds, then the WMS System must preserve its `TU_NUMBER` as a Packing `TU`; joint packing of SKU must be subject to the `TUSetup.maxWeight` weight limit, with no additional compatibility constraints in version 1.
```gherkin
Scenario: Picking TU becomes Packing TU
  Given a Picking TU meets issuance thresholds and the `TUSetup.maxWeight` limit
  When it is qualified for packing
  Then it retains `TU_NUMBER`
  And it may contain multiple SKU without an additional compatibility rule if their combined weight does not exceed `TUSetup.maxWeight`
```

### FR-P1-11 — Repack by SKU and shortage confirmation
*Category:* event-driven · *Source:* P1 R21–P1 R22
**When** the Packer performs “repack by SKU”, the WMS System must leave the choice of SKU and order to them, and treat a quantity lower than `pickedQty` as a shortage only after rechecking and explicit confirmation.
```gherkin
Scenario: Shortage requires two actions
  Given the Packer found fewer units than `pickedQty`
  When they recheck the TU and confirm the shortage
  Then the WMS System starts the `SHORT_PICKED` mechanism
  And before confirmation the mechanism is not started
```

### FR-P1-12 — DAMAGED and unexpected SKU
*Category:* event-driven · *Source:* P1 R23–P1 R24
**When** `DAMAGED` or an unexpected SKU is detected during repacking, the WMS System must route the goods to `QC`; for `DAMAGED` it must start the shortage mechanism, while for unexpected SKU it must not start that mechanism.
```gherkin
Scenario: Different QC effects
  Given the Packer detects one damaged unit and one foreign SKU code
  When they record the discrepancies
  Then both goods go to `QC`
  And only `DAMAGED` starts the `SHORT_PICKED` mechanism
```

### FR-P1-13 — Repacking closure and Shipment grouping
*Category:* conditional · *Source:* P1 R25–P1 R26
**If** all expected SKU have been settled, the WMS System must allow repacking to be completed; grouping Packing `TU` objects into a `Shipment` must require compatible customer, address, priority and identical `slaDeadline`.
```gherkin
Scenario: Unsettled SKU blocks closure
  Given one expected SKU has no disposition
  When the Packer tries to finish repacking
  Then the WMS System routes them to continue or report a shortage
  And does not group TU with another `slaDeadline`
```

### FR-P1-14 — PACKED aggregation and READY_FOR_DISPATCH
*Category:* event-driven · *Source:* P1 R27–P1 R28
**When** all lines and TU objects of an `OutboundOrder` are packed, the WMS System must set `PACKED` and then `READY_FOR_DISPATCH`; a `Shipment` must reach `READY_FOR_DISPATCH` when all contributing orders are ready or at the common `slaDeadline`.
```gherkin
Scenario: Timeout closes Shipment grouping
  Given not all planned orders finished packing in time
  When the common `slaDeadline` expires
  Then the current `Shipment` transitions to `READY_FOR_DISPATCH`
  And includes only ready TU objects
```

### FR-P1-15 — Late TU and start of Carrier Selection
*Category:* event-driven · *Source:* P1 R29–P1 R30
**When** TU objects of a late `OutboundOrder` become ready after grouping has closed, the WMS System must create a new `Shipment`; Carrier Selection may start only for a `Shipment READY_FOR_DISPATCH` with a complete TU set.
```gherkin
Scenario: Late TU creates a new shipment
  Given the previous Shipment was closed by `slaDeadline`
  When another TU becomes ready
  Then the WMS System groups it into a new `Shipment`
  And only then starts Carrier Selection
```

### FR-P1-16 — CarrierSetup and tie-break
*Category:* conditional · *Source:* P1 R31–P1 R32
**If** multiple `CarrierSetup` records match `Region` and weight/volume, then the WMS System must select, in order, the narrowest volume interval, the narrowest weight interval and the unique `Carrier.priority`.
```gherkin
Scenario: Deterministic carrier selection
  Given two `CarrierSetup` records match the Shipment
  When the WMS System applies the tie-break
  Then it selects the record with the narrowest volume interval
  And uses subsequent criteria only on a tie
```

### FR-P1-17 — Manual carrier change and WMS label
*Category:* event-driven · *Source:* P1 R33–P1 R34
**When** `Warehouse Supervisor` changes the Carrier Selection result, the WMS System must store the new `Carrier` without requiring a reason; it must generate the label locally, without carrier API or carrier approval.
```gherkin
Scenario: Supervisor changes the result
  Given a carrier was selected automatically
  When the Supervisor indicates another one
  Then the Shipment uses the new `Carrier`
  And the label is produced as a WMS printout without an external call
```

### FR-P1-18 — Loading problem and ERP gate
*Category:* conditional · *Source:* P1 R35–P1 R36
**If** a loading problem occurs before `CarrierManifest.CLOSED`, the WMS System must allow changing `Carrier` without reprinting; every `Shipment` must reach `POSTED` before being added to a manifest.
```gherkin
Scenario: Cannot manifest before POSTED
  Given Shipment has `POSTING_PENDING`
  When the Dispatcher tries to add it to the manifest
  Then the WMS System rejects the operation
  And allows it only after `POSTED`
```

### FR-P1-19 — ERP retry and cancellation boundary
*Category:* conditional · *Source:* P1 R37–P1 R38
**If** a Shipment has `POSTING_ERROR`, the WMS System must require a manual `Warehouse Supervisor` decision for retry; from `POSTING_PENDING` onward it must not allow cancellation in WMS.
```gherkin
Scenario: Retry requires a decision
  Given Shipment has `POSTING_ERROR`
  When no Supervisor decision exists
  Then retry is not sent
  And cancellation from `POSTING_PENDING` remains forbidden
```

### FR-P1-20 — One manifest and irreversible CLOSED
*Category:* state-driven · *Source:* P1 R39–P1 R40
**While** a `Shipment` does not belong to a manifest, the WMS System must allow assigning it to exactly one `CarrierManifest`; after `CLOSED` it must block reopening, cancellation and `Carrier` change.
```gherkin
Scenario: Shipment does not enter two manifests
  Given Shipment was assigned to an open manifest
  When the Dispatcher tries to add it to a second one
  Then the WMS System rejects the operation
  And after closing the first manifest it blocks rollback
```

### FR-P1-21 — Closing CustomerOrder and SHORT_ALLOCATED false
*Category:* event-driven · *Source:* P1 R41–P1 R42
**When** all contributing `OutboundOrder` objects reach `COMPLETED`, the WMS System must close `CustomerOrder`; when cancelling after `SHORT_ALLOCATED`, it must release allocations, keep the problematic line in `BACKORDERED` and the remaining lines in `OPEN`.
```gherkin
Scenario: SHORT_ALLOCATED cancellation preserves the order
  Given the Supervisor cancels a shortage OutboundOrder
  When the WMS System settles the decision
  Then the problematic line is `BACKORDERED`, the remaining lines `OPEN`
  And CustomerOrder may become `CLOSED` only after all orders fulfilling it are completed
```

### FR-P1-22 — Automatic shortage handling
*Category:* conditional · *Source:* P1 R43–P1 R44
**If** `allowPartialShipment = true`, the WMS System must fulfill the available quantity and leave the shortage `BACKORDERED`; at `SHORT_PICKED` it must automatically create further `PickTask` objects up to the limit and then escalate.
```gherkin
Scenario: Reallocation limit ends automation
  Given the line has `SHORT_PICKED` and reached the effective limit
  When the WMS System searches for further inventory
  Then it does not create another `PickTask`
  And escalates the case to `Warehouse Supervisor`
```

### FR-P1-23 — Wait or cancel after SHORT_PICKED
*Category:* conditional · *Source:* P1 R45–P1 R46
**If** the Supervisor selects “wait”, the WMS System must roll back unpacked lines of that `CustomerOrder`; if they select “cancellation”, it must require editing `CustomerOrderLine` quantity rather than only cancelling `OutboundOrderLine`.
```gherkin
Scenario: PACKED lines are not rolled back
  Given one line is `PACKED`, and another has `SHORT_PICKED`
  When the Supervisor chooses “wait”
  Then the `PACKED` line remains unchanged
  And unpacked lines are rolled back through the proper path
```

### FR-P1-24 — Permanent change and shortage during packing
*Category:* event-driven · *Source:* P1 R47–P1 R48
**When** the Supervisor permanently sets `allowPartialShipment = true`, the WMS System must require a reason and limit the change to the indicated `CustomerOrder`; a shortage or damage during packing must be handled like `SHORT_PICKED`, without blocking the source location.
```gherkin
Scenario: Change does not affect other orders
  Given the Supervisor changed the flag for one CustomerOrder
  When another CustomerOrder of the same customer has a shortage
  Then it uses its own unchanged flag
  And a packing shortage on the first order does not block the source location
```

### FR-P1-25 — ON_HOLD and general cancellation
*Category:* conditional · *Source:* P1 R49–P1 R50
**If** `CustomerOrder` is `ON_HOLD`, the WMS System must skip it until the block is removed; general cancellation of the whole order must be allowed only when every line meets its condition and the manifest is not `CLOSED`.
```gherkin
Scenario: One blocking line rejects whole-order cancellation
  Given one line does not meet the cancellation condition
  When OMS requests cancellation of the entire CustomerOrder
  Then the WMS System rejects the request and indicates the line
  And an `ON_HOLD` order returns to planning only after unblocking
```

### FR-P1-26 — No carrier and ATP inventory
*Category:* conditional · *Source:* P1 R51–P1 R52
**If** Carrier Selection returns no result, the WMS System must escalate selection to the indicated actors; `Allocation` must reserve only inventory with the `ATP` flag, rejecting non-ATP inventory.
```gherkin
Scenario: Non-ATP is not allocated
  Given physically available inventory does not have the ATP flag
  When the WMS System performs Allocation
  Then it does not reserve that inventory
  And no carrier result is routed to manual approval
```

### FR-P1-27 — Continuous CustomerOrder aggregation
*Category:* state-driven · *Source:* P1 Continuous function F1
**While** fulfillment of a `CustomerOrder` continues, the WMS System must calculate the header from the least advanced active line status, with exceptions `PARTIALLY_SHIPPED` and full `BACKORDERED`.
```gherkin
Scenario: One BACKORDERED line does not roll the header back
  Given one line is `BACKORDERED` and another is in fulfillment
  When the WMS System recalculates the header
  Then the header remains in its current fulfillment status with `WARNING`
  And receives `BACKORDERED` only when all active lines are `BACKORDERED`
```

### FR-P1-28 — TU_NUMBER identity and uniqueness
*Category:* universal · *Source:* P1 R53
The WMS System must assign every Outbound `TU` a required `TU_NUMBER`, unique per warehouse among active (non-terminal) Outbound `TU` objects, with optional `SSCC` which need not equal `TU_NUMBER`, and preserve `TU_NUMBER` when the role transitions `PickContainer → PackUnit`. A `TU_NUMBER` that is not an `SSCC` must contain only alphanumeric characters without special characters, have a maximum of 20 characters and be encoded using Code 128 symbology.
```gherkin
Scenario: Active TU number is not duplicated
  Given an active Outbound TU exists with a given TU_NUMBER
  When another Outbound TU is created in that warehouse
  Then it receives another TU_NUMBER
  And the number is preserved when the role changes from PickContainer to PackUnit
  And a TU_NUMBER that is not SSCC is alphanumeric, has at most 20 characters and is encoded in Code 128
```

### FR-P1-29 — PickTask assignment by module and zone selection
*Category:* event-driven · *Source:* P1 R54, P1 R56
**When** `Warehouse Operator` is logged into the picking module with a selected zone and has no active warehouse task of any type, the WMS System must assign the next `PickTask` from that zone according to configured order, and must not assign a task to an operator who already has an active task.
```gherkin
Scenario: Operator without a task gets next PickTask in their zone
  Given the operator is logged into the picking module in zone A and has no active task
  When the WMS System selects the next task to assign
  Then the operator receives the highest queued PickTask of zone A
Scenario: Operator with an active task gets no next task
  Given the operator already has an active PickTask
  When another PickTask is waiting in their zone
  Then the WMS System does not assign another task
```

### FR-P1-30 — Continuing an order in the next zone
*Category:* event-driven · *Source:* P1 R55
**When** `Warehouse Operator` completes a `PickTask`, the WMS System must allow a choice between closing the Picking `TU` and continuing the same `OutboundOrder` in another zone into the still-open `TU`; if continuation is selected, it must assign the indicated `PickTask` regardless of queue order and zone and change the operator's active zone.
```gherkin
Scenario: Operator continues the order in another zone
  Given the operator completed a zone-A PickTask for an open Picking TU
  When they choose to continue the same OutboundOrder in zone B
  Then they receive that order's zone-B PickTask outside the queue order
  And their active zone changes to B
```

### FR-P1-31 — Whole-order dispatch when partial shipment is forbidden
*Category:* conditional · *Source:* P1 R57

**If** `CustomerOrder` has `allowPartialShipment = false`, then the WMS System must dispatch the entire order in one complete `Shipment` and block dispatch while even one required line is missing; one `Shipment` may contain multiple Packing `TU` objects of the order. The promise must apply regardless of how many `OutboundOrder` objects and fulfillment channels handled the order.
```gherkin
Scenario: Incomplete order is not dispatched
  Given CustomerOrder has allowPartialShipment = false
  And the order is handled by more than one OutboundOrder or channel
  When at least one required line is missing
  Then the WMS System does not dispatch the Shipment
  And after all lines are complete it dispatches the entire CustomerOrder in one complete Shipment
```

### FR-P1-32 — Holding TU attachment to Shipment
*Category:* state-driven · *Source:* P1 R58

**While** every `CustomerOrderLine` of an order with `allowPartialShipment = false` does not satisfy one of two mutually exclusive alternatives, the WMS System must not attach a Packing `TU` of that order to any `Shipment`: (A) inactive line — successfully cancelled or corrected by `Warehouse Supervisor` according to `P1 R46`, no longer belonging to required quantity and not subject to alternative (B); or (B) active line — the sum of `requiredQty` of those `OutboundOrderLine` objects that reached `PACKED` or beyond and are not `CANCELLED` equals the effective `CustomerOrderLine.Quantity` after any correction, regardless of channel. Status `PLANNED` alone must not satisfy this condition. The `TU` must remain in `PACKING_SEALED`, and automatic `Shipment` close after `slaDeadline` must not include it because it belongs to no `Shipment`.
```gherkin
Scenario: Deadline passes before full packing
  Given CustomerOrder has allowPartialShipment = false
  And an active CustomerOrderLine has Quantity = 100
  And its cross-dock OutboundOrderLine has requiredQty = 30 and reached PACKED
  And for the remaining quantity 70 no OutboundOrderLine exists, even though CustomerOrderLine has PLANNED
  When slaDeadline expires
  Then Packing TU of this order remains in PACKING_SEALED
  And does not enter any closing Shipment
  And the sum of requiredQty = 30 is not equal to Quantity = 100
```

### FR-P1-33 — Splitting a customer-order line
*Category:* event-driven · *Source:* P1 R59

**When** the WMS System plans an `OutboundOrder`, it must allow one `CustomerOrderLine` to be split across multiple `OutboundOrderLine` objects.
```gherkin
Scenario: One customer line goes to two orders
  Given CustomerOrderLine requires more quantity than one order will cover
  When the WMS System plans OutboundOrder
  Then the line is split across multiple OutboundOrderLine objects
```

### FR-P1-34 — Condition for joint packing of lines from different orders
*Category:* conditional · *Source:* P1 R60

**If** a Packing `TU` is to contain lines from multiple `OutboundOrder` objects, then the WMS System must require the same customer, the same delivery address, compatible priority and compatible delivery deadline of the source `CustomerOrder` objects already during repacking; all lines in one Packing `TU` must go to one `Shipment`.
```gherkin
Scenario: Different address blocks joint packing
  Given two OutboundOrder objects have different delivery addresses
  When the Packer tries to repack their lines into one Packing TU
  Then the WMS System does not allow joint packing
```

### FR-P1-35 — WARNING flag lifecycle
*Category:* event-driven · *Source:* P1 R61

**When** the cause of `WARNING` on a `CustomerOrder` ceases, the WMS System must remove the flag automatically; **Where** `Warehouse Supervisor` removes it manually while the cause persists, the next cyclic-planning run must set it again, without changing business behavior.
```gherkin
Scenario: Manual removal while the cause persists
  Given CustomerOrder has WARNING due to incomplete reservation
  When Warehouse Supervisor sets the flag to false while reservation is still incomplete
  Then the next planning run sets WARNING again
  And the order's business behavior does not change
```

### FR-P1-36 — Picking TU assignment strategy
*Category:* configurable · *Source:* P1 R62

**Where** the warehouse configures the Picking `TU` assignment strategy, the WMS System must offer two variants: a separate Picking `TU` for each task or zone, and a shared Picking `TU` passing through subsequent tasks of the same `OutboundOrder`.
```gherkin
Scenario: Shared TU variant across tasks
  Given the warehouse configured the shared Picking TU variant
  When the operator completes a task and continues the same OutboundOrder in another zone
  Then they pick into the same Picking TU
```

### FR-P1-37 — Calculating `TU` content weight and volume
*Category:* event-driven · *Source:* P1 R63

**When** the weight or volume of `TU` contents must be determined, the WMS System must calculate the corresponding value as the sum across all `SKU` in that `TU` of unit weight or unit volume from the warehouse item master multiplied by number of units; it must use calculated content volume only to evaluate issuance thresholds, not to block filling or repacking limits, and it must treat `TUSetup.maxVolume` as packaging-type cubic capacity used for carrier selection.
```gherkin
Scenario: Weight and volume calculated from contents
  Given the `TU` contains 20 units of an `SKU` weighing 2 kg and occupying 0.01 m³ and 10 units of an `SKU` weighing 1 kg and occupying 0.02 m³
  When the WMS System determines `TU` content weight and volume
  Then weight is 50 kg and content volume is 0.4 m³
  And content volume is used only for issuance-threshold evaluation, not filling blocks or repacking limits
  And Carrier Selection uses `TUSetup.maxVolume` as packaging-type cubic capacity
```

### FR-P1-38 — Picking TU issuance thresholds
*Category:* conditional · *Source:* P1 R64

**If** the WMS System evaluates a Picking `TU` for issue without repacking, then it must apply two branches in order: (1) for a `TU` whose type has the external-carrier role (`P1 R68`), treat issuance thresholds as satisfied by definition, without evaluating them and without reading `TUSetup.minIssueWeight` or `TUSetup.minIssueVolume`; (2) for other types, treat issuance thresholds as satisfied only when the `TU` type has `TUSetup.externalIssuable = true` and at least one of two lower thresholds is reached: current weight reaches `TUSetup.minIssueWeight` or current content volume reaches the absolute threshold `TUSetup.minIssueVolume`.
```gherkin
Scenario: Light but well-filled TU meets thresholds
  Given the `TU` type has `TUSetup.externalIssuable = true` and `TUSetup.processUsage` other than `EXTERNAL`, and current weight has not reached `TUSetup.minIssueWeight`
  When current content volume reaches `TUSetup.minIssueVolume`
  Then the WMS System treats issuance thresholds as satisfied
```

### FR-P1-39 — Operator forcing issuability
*Category:* event-driven · *Source:* P1 R65

**When** a `TU` on a type other than the external-carrier role type (`P1 R68`) has not reached lower weight or volume thresholds, has `TUSetup.externalIssuable = true`, and meets at least one of two conditions — it is the last `TU` of its `OutboundOrder` or `slaDeadline` is approaching — the WMS System must allow `Warehouse Operator` to close it and force issuability without repacking, recording a reason. The force must waive only lower thresholds and must not allow sealing or dispatching a `TU` with `TUSetup.externalIssuable = false`; for such a `TU`, the only path must be repacking into an issuable type according to `P1 R66`. For a `TU` on the external type, the WMS System must not require this force because issuability follows from the first branch of `P1 R64`.
```gherkin
Scenario: Last TU of an order below thresholds
  Given the `TU` has `TUSetup.externalIssuable = true` and `TUSetup.processUsage` other than `EXTERNAL`, has not reached lower thresholds and is the last `TU` of its `OutboundOrder`
  When `Warehouse Operator` closes it and provides a reason
  Then the WMS System dispatches the `TU` without repacking
  And records the force reason

Scenario: Non-issuable type blocks operator force
  Given the `TU` has `TUSetup.externalIssuable = false`, has not reached lower thresholds and is the last `TU` of its `OutboundOrder`
  When `Warehouse Operator` tries to close it and force issuability
  Then the WMS System does not allow it to be sealed or dispatched
  And the only path is repacking to an issuable type according to `P1 R66`
```

### FR-P1-40 — Issuable carrier type during repacking
*Category:* conditional · *Source:* P1 R66

**If** a Packing `TU` is created by repacking, then the WMS System must allow it to be sealed only if its type is marked externally issuable.
```gherkin
Scenario: Repacking into a non-issuable type
  Given the Packer repacks goods into a carrier type without externalIssuable
  When they try to seal the Packing TU
  Then the WMS System does not allow it to be sealed
```

### FR-P1-41 — Continuing PickTask in the next Picking TU
*Category:* event-driven · *Source:* P1 R67

**When** a Picking `TU` reaches `PICK_FULL` while quantity remains to be picked in `PickTask`, the WMS System must allow the Picker to close the full `TU`, create and on first scan link another Picking `TU` to the same `PickTask`, and continue the task without changing its identity; `PickTask` may reach `COMPLETED` only after the full quantity is picked, and the next `TU` must inherit `directPackDeclared` from the previous `TU` of the same `PickTask` without asking the operator again, according to `P1 R16`.
```gherkin
Scenario: Continue PickTask in the next Picking TU
  Given the first Picking `TU` reached `PICK_FULL` and quantity remains in the `PickTask`
  When the Picker closes the full `TU` and scans another Picking `TU`
  Then the first `TU` transitions `PICK_FULL → READY_TO_PACK`, and the second `CREATED → IN_PICKING`
  And the WMS System links the second `TU` to the same `PickTask`, which keeps its identity and remains `IN_PROGRESS`
  And the second `TU` inherits `directPackDeclared` from the first without asking the operator again
  And `PickTask` reaches `COMPLETED` only after full quantity is picked
```

### FR-P1-42 — External carrier type
*Category:* universal · *Source:* P1 R68

The WMS System must maintain exactly one `TUSetup` with `processUsage = EXTERNAL` and `externalIssuable = true` and recognize an Outbound `TU` of external origin exclusively by `processUsage` of the type referenced by its `tuSetupCode`, never by `TU_NUMBER` or an additional flag; for such a `TU`, the first branch of `P1 R64` must treat issuance thresholds as satisfied without reading `minIssueWeight` and `minIssueVolume`, and `P1 R65` must not be required.
```gherkin
Scenario: Recognition and issuability of an externally originated TU
  Given the catalog contains exactly one `TUSetup` with `processUsage = EXTERNAL` and `externalIssuable = true`
  And `TU.tuSetupCode` points to that type
  When the WMS System recognizes the `TU` origin and evaluates issuance thresholds
  Then it recognizes it as external exclusively by `processUsage` of the referenced type, without using `TU_NUMBER` or an additional flag
  And the first branch of `P1 R64` treats thresholds as satisfied without reading `minIssueWeight` or `minIssueVolume`
  And manual forcing under `P1 R65` is not required
```

### FR-P1-43 — requiredQty lifecycle
*Category:* event-driven · *Source:* P1 R69

**When** the WMS System creates an `OutboundOrderLine`, it must set its `requiredQty` as the current target quantity — during `OutboundOrder` planning in channel `STANDARD` or generation of `CrossDockPickTask` in channel `CROSSDOCK`; **when** `Warehouse Supervisor` corrects `CustomerOrderLine.Quantity` in `P1 R46` or in the correction branch of `P2 R15`, the WMS System must change the `requiredQty` of the relevant line together with it. Outside these two paths, `requiredQty` must not change automatically.
```gherkin
Scenario: Standard correction changes requiredQty
  Given `CustomerOrderLine.Quantity = 100`, and its one `OutboundOrderLine.requiredQty = 100` and `pickedQty = 30`
  When `Warehouse Supervisor` corrects `CustomerOrderLine.Quantity` to 30 in `P1 R46`
  Then the WMS System changes `OutboundOrderLine.requiredQty` to 30

Scenario: Cross-dock correction changes requiredQty
  Given cross-dock `OutboundOrderLine.requiredQty = 100`, and its only `CrossDockPickTask.confirmedQty = 30`
  When `Warehouse Supervisor` corrects `CustomerOrderLine.Quantity` to 30 in the correction branch of `P2 R15`
  Then the WMS System changes `OutboundOrderLine.requiredQty` to 30
  And `confirmedQty = requiredQty`, so the line may continue to `PACKED`

Scenario: Without correction requiredQty does not change automatically
  Given `CustomerOrderLine.Quantity = 100`, and the only `OutboundOrderLine` has `requiredQty = 30` and status `PACKED`
  When no correction was performed in `P1 R46` or `P2 R15`
  Then `requiredQty` remains 30
  And the sum of `requiredQty` is 30 and is not equal to `CustomerOrderLine.Quantity = 100`
```

### FR-P1-44 — Completion of OutboundOrder after all its Shipment objects
*Category:* event-driven · *Source:* P1 R70
**When** `CarrierManifest` reaches `CONFIRMED`, the WMS System must perform `OutboundOrder DISPATCHED → COMPLETED` only for those orders for which every `Shipment` containing their Outbound `TU` belongs to a `CarrierManifest` in `CONFIRMED`; **while** at least one such `Shipment` is not confirmed, the order must remain in `DISPATCHED`, regardless of manifest-confirmation order.
```gherkin
Scenario: First confirmed manifest does not complete the order
  Given `OutboundOrder` has Packing `TU` objects in `Shipment` A and `Shipment` B
  When the `CarrierManifest` containing `Shipment` A reaches `CONFIRMED` while `Shipment` B is not yet confirmed
  Then `OutboundOrder` remains in `DISPATCHED`

Scenario: Last confirmed manifest completes the order
  Given `Shipment` A of this `OutboundOrder` is already confirmed
  When the `CarrierManifest` containing `Shipment` B reaches `CONFIRMED`
  Then `OutboundOrder` transitions `DISPATCHED → COMPLETED`

Scenario: One Shipment per order preserves behavior
  Given all Packing `TU` objects of the order belong to one `Shipment`
  When that `Shipment`'s `CarrierManifest` reaches `CONFIRMED`
  Then `OutboundOrder` transitions `DISPATCHED → COMPLETED` at the same moment as before
```

### FR-P1-45 — Quantity blocked by Allocation
*Category:* event-driven · *Source:* P1 R71

**When** the WMS System creates an `Allocation` or changes its state, it must maintain `reservedQty` as the quantity actually blocked: `0` in `PENDING`, partial quantity in `SHORT`, `OutboundOrderLine.requiredQty` on entry into `RESERVED` and `CONFIRMED`, `0` in `RELEASED` and `CONSUMED`; **while** an allocation is not in `SHORT`, `RESERVED` or `CONFIRMED`, it must contribute nothing to the quantity of inventory considered occupied. Reducing `reservedQty` during `CONFIRMED` on partial dispatch is described by `FR-P1-46`.
```gherkin
Scenario: Partial reservation blocks only what was reserved
  Given `OutboundOrderLine.requiredQty` is 100
  When the WMS System reserves 30 and sets `Allocation SHORT`
  Then `reservedQty` is 30
  And the quantity occupied by this allocation is 30, not 100

Scenario: Full reservation and picking do not change occupied quantity
  Given `Allocation` reached `RESERVED` with `reservedQty` equal to `requiredQty`
  When `Allocation` transitions `RESERVED → CONFIRMED`
  Then `reservedQty` remains unchanged
  And the inventory is still considered occupied

Scenario: Terminal states do not block inventory
  Given `Allocation` reached `RELEASED` or `CONSUMED`
  When the WMS System calculates occupied inventory quantity
  Then this allocation contributes 0
  And `PENDING` also contributes 0
```

### FR-P1-46 — Settlement of a line dispatched in multiple Shipment objects
*Category:* event-driven · *Source:* P1 R72

**When** `CarrierManifest` reaches `CONFIRMED`, the WMS System must settle `Inventory PICKED → SHIPPED` for quantity contained in the Outbound `TU` objects of that manifest and reduce `Allocation.reservedQty` by that quantity; **while** at least one Outbound `TU` contributing quantity to a given `OutboundOrderLine` does not belong to a `Shipment` with a `CONFIRMED` manifest, the WMS System must not perform `OutboundOrderLine PACKED → SHIPPED` or `Allocation CONFIRMED → CONSUMED`, regardless of manifest-confirmation order.
```gherkin
Scenario: Partial dispatch does not terminalize line or allocation
  Given `OutboundOrderLine.requiredQty` is 100 and its goods are in two Outbound `TU` objects of 50 each, in two different `Shipment` objects
  When the manifest of the first `Shipment` reaches `CONFIRMED`
  Then `Inventory` settles 50 units as `SHIPPED`, and `Allocation.reservedQty` falls from 100 to 50
  And `OutboundOrderLine` remains in `PACKED`, and `Allocation` in `CONFIRMED`

Scenario: Final dispatch closes line and allocation
  Given one unconfirmed Outbound `TU` of the line remains
  When the manifest of its `Shipment` reaches `CONFIRMED`
  Then `Allocation.reservedQty` reaches 0
  And `OutboundOrderLine` transitions `PACKED → SHIPPED`, and `Allocation` `CONFIRMED → CONSUMED`

Scenario: One TU and one Shipment preserve behavior
  Given the entire line quantity is in one Outbound `TU`, in one `Shipment`
  When that `Shipment`'s manifest reaches `CONFIRMED`
  Then `OutboundOrderLine` transitions `PACKED → SHIPPED`, and `Allocation` `CONFIRMED → CONSUMED` at the same moment as before
```

## P2 — Outbound Crossdock

### FR-P2-01 — Demand queue and 1:1 matching
*Category:* event-driven · *Source:* P2 R1–P2 R2
**When** an Inbound `TU` enters `IN_CROSS_DOCK`, the WMS System must generate `CrossDockPickTask` for `BACKORDERED` lines according to the warehouse queue, and route a full 1:1 match to one `Shipment` only when the entire declared contents of the source `TU` cover demand of the same customer and delivery address, with compatible `priority` and identical `slaDeadline`; with `allowPartialShipment = true`, demand may come from several `CustomerOrder` objects of the same customer, with `allowPartialShipment = false` — from one only, and the source `TU` may contain multiple `SKU`.
```gherkin
Scenario: Full 1:1 match
  Given the entire declared contents of the source TU cover demand of the same customer and address, with compatible priority and identical slaDeadline
  And the source TU contains multiple SKU, and with allowPartialShipment = true demand comes from two CustomerOrder objects of the same customer
  When the WMS System generates the cross-dock task
  Then the whole quantity feeds one target TU and one Shipment
  And demand was selected according to the queue

Scenario: No combining orders without permission for partial shipment
  Given demand with compatible customer, address, priority and identical slaDeadline comes from two CustomerOrder objects, one of which has allowPartialShipment = false
  When the WMS System evaluates a full 1:1 match
  Then it does not qualify combined demand from both CustomerOrder objects as a full 1:1 match
```

### FR-P2-02 — n:n sorting without Allocation
*Category:* universal · *Source:* P2 R3–P2 R4
The WMS System must allow one Outbound `TU` to be fed from multiple Inbound `TU` objects for `OutboundOrderLine` objects compatible in customer/address/`priority` and with identical `slaDeadline` (strict match, not “similar”), and must not create `Allocation` in cross-docking.
```gherkin
Scenario: Two source TU objects feed one target
  Given two Inbound TU objects contain matching SKU for one shipment
  When the Packer sorts their contents
  Then one Outbound TU may collect quantity from both sources
  And no `Allocation` is created
```

### FR-P2-03 — CROSSDOCK channel and eligibleQty
*Category:* event-driven · *Source:* P2 R5–P2 R6
**When** a cross-dock `OutboundOrder` is created, the WMS System must set immutable `fulfillmentChannel = CROSSDOCK`, must not mix channels within one `OutboundOrder`, and must calculate `crossDockEligibleQty` as min(`sourceEligibleQty`, `demandEligibleQty`), where `sourceEligibleQty` reduces declared (ASN) quantity of source Inbound `TU`/`SKU` by `plannedQty` of active tasks and by `confirmedQty` and `damagedQty` of completed `CrossDockPickTask` objects, while `demandEligibleQty` reduces `CustomerOrderLine.Quantity` by its assigned `ATPReservation` and the sum of `requiredQty` of all its `OutboundOrderLine` objects that are not `CANCELLED`, regardless of channel.
```gherkin
Scenario: Damaged quantity does not return to eligible pool
  Given ASN declaration is 100, active tasks have plannedQty 30, and completed tasks have 40 confirmedQty and 10 damagedQty
  When the WMS System calculates sourceEligibleQty
  Then the result is 20
  And channel CROSSDOCK remains immutable for the whole OutboundOrder
```

### FR-P2-04 — Target TU number and start of sorting
*Category:* event-driven · *Source:* P2 R7–P2 R8
**When** a target Outbound `TU` is used for the first time, the WMS System must assign its `TU_NUMBER`/`SSCC` — for a full 1:1 match with a valid GS1 `SSCC` on the source Inbound `TU`, by mandatory inheritance of `TU_NUMBER` and `SSCC`, and otherwise and for n:n sorting by assigning a new number from `TUSetup`/`Sequence` together with a new label; start of scanning must move the task and line to `IN_PROGRESS`/`PICKING`, and the header to `PACKING_IN_PROGRESS`.
```gherkin
Scenario: Number is inherited only for 1:1 with valid GS1
  Given the source Inbound TU has a TU_NUMBER that is a valid GS1 SSCC and a full 1:1 match
  When the target Outbound TU is created
  Then it inherits TU_NUMBER and SSCC without a new label
  And for n:n sorting it receives a new number from Sequence and a new label
```

### FR-P2-05 — Cancellation block and closing target TU
*Category:* conditional · *Source:* P2 R9–P2 R10
**If** sorting is in progress, the WMS System must reject general cancellation until `PACKED`; it must close the target Outbound `TU` into `PACKING_SEALED` in one of two cases: (1) the Packer closes it as physically full during a task and continues sorting into a new `TU` within the same `CrossDockPickTask`; (2) the WMS System closes it automatically when its `slaDeadline` is reached (or passed) and no active or planned `CrossDockPickTask` points to it as target — before `slaDeadline`, the mere absence of such tasks must not close the `TU`.
```gherkin
Scenario: Completion of one task does not close a shared TU
  Given the target Outbound TU is fed by tasks from two source Inbound TU objects
  When the first of those tasks reaches COMPLETED
  Then the TU remains open because the second task still targets it
  And general cancellation is rejected during this time
```

### FR-P2-06 — Confirmed shortage and DAMAGED
*Category:* event-driven · *Source:* P2 R11–P2 R12
**When** the Packer finds a quantity lower than planned, the WMS System must require rechecking and confirmation; detected `DAMAGED` must be routed to `QC`.
```gherkin
Scenario: Shortage does not arise after first reading
  Given a quantity lower than planned was read
  When the Packer does not confirm a recheck
  Then the shortage is not recorded
  And goods marked `DAMAGED` go to `QC`
```

### FR-P2-07 — damagedQty and wait decision
*Category:* conditional · *Source:* P2 R13–P2 R14
**If** quantity is reported as `DAMAGED`, the WMS System must include it in `damagedQty`, not ordinary shortage; with a “wait” decision it must cancel the line, return confirmed quantity through P4 and leave demand in `BACKORDERED`.
```gherkin
Scenario: Ordinary shortage does not increase damagedQty
  Given part of the quantity is missing but not damaged
  When the task is settled
  Then `damagedQty` does not include the missing quantity
  And with a “wait” decision confirmed quantity goes to PutBack
```

### FR-P2-08 — Shortage cancellation and correct source state
*Category:* event-driven · *Source:* P2 R15–P2 R16
**When** the Supervisor chooses shortage cancellation, the WMS System must cancel the full `CustomerOrderLine` or correct it to `confirmedQty`; after sorting starts, `OutboundOrderLine` must transition to `CANCELLED` from `PICKING`.
```gherkin
Scenario: Correction to confirmedQty continues shipment
  Given part of planned quantity was confirmed
  When the Supervisor corrects CustomerOrderLine to `confirmedQty`
  Then no PutBackTask is created for that quantity
  And the line may continue to `PACKING_SEALED`
```

### FR-P2-09 — Unexpected SKU and empty TU
*Category:* event-driven · *Source:* P2 R17, P2 R18, P2 R37
**When** the Packer detects an unexpected SKU, the WMS System must route it to `QC` without the shortage mechanism; when the source TU is empty before picking, it must cancel the task/line, set demand `BACKORDERED` and Inbound `TU → LOST`; when all `OutboundOrderLine` objects of that `OutboundOrder` thereby reach `CANCELLED` with none `PACKED`, the `OutboundOrder` header must transition to `CANCELLED` — a general criterion (`P2 R37`), not specific to this path.
```gherkin
Scenario: Empty source TU cancels an empty OutboundOrder
  Given no line of OutboundOrder reached `PACKED`
  When the source TU proves empty
  Then the task and line are `CANCELLED`, and demand `BACKORDERED`
  And the OutboundOrder header transitions to `CANCELLED`
```

### FR-P2-10 — PutBack without Allocation and task completeness
*Category:* event-driven · *Source:* P2 R19–P2 R20
**When** confirmed cross-dock quantity loses demand after `PACKED`, the WMS System must recover it through `PutBackTask` without an `Allocation RELEASED` step; `CrossDockPickTask` must reach `COMPLETED` when the Packer reports completion, regardless of remaining tasks of the same source Inbound `TU` and whether its entire declared (ASN) contents have been settled.
```gherkin
Scenario: Task completes independently of the remaining tasks
  Given one source TU with declaration 100 has three tasks with plannedQty 30, 20 and 10
  When the Packer reports completion of the first task
  Then that task reaches COMPLETED and the source TU remains in cross-docking
  And no Allocation release step is created
```

### FR-P2-11 — Remainder and damagedQty aggregation
*Category:* event-driven · *Source:* P2 R21–P2 R22
**When** residual quantity remains after cross-docking, the WMS System must route it to `TRANSIT` and set Inbound `TU → IN_PUTAWAY`; it must pass Inbound the sum of `confirmedQty` and the sum of `damagedQty` of all tasks of that `TU` as separate components, so residual quantity per `SKU` — declared (ASN) quantity minus both components — can be determined by Inbound Process 3, which owns its definition and quantitative control.
```gherkin
Scenario: Remainder returns to Putaway
  Given part of the declared quantity was not assigned
  When all tasks of the source TU are completed
  Then the remainder goes to `TRANSIT`, and the TU to `IN_PUTAWAY`
  And Inbound receives aggregated `damagedQty`

Scenario: Damaged quantity reduces residual quantity
  Given ASN declaration of the source TU is 100, and completed tasks produced total confirmedQty 50 and damagedQty 10
  When the WMS System finalizes the source TU and passes the result to Inbound
  Then the passed components determine residual quantity 40, not 50
```

### FR-P2-12 — Independent settlement and CROSS_DOCKED
*Category:* conditional · *Source:* P2 R23–P2 R24
**If** all declared contents have been settled by cross-docking, the WMS System must set Inbound `TU → CROSS_DOCKED` regardless of topology; it must settle confirmed OK quantity without waiting for Putaway of the remainder.
```gherkin
Scenario: No remainder gives CROSS_DOCKED
  Given n:n topology settled the entire declared quantity
  When the tasks are completed
  Then Inbound TU has `CROSS_DOCKED`
  And settlement does not depend on the number of target TU objects
```

### FR-P2-13 — GR gate for Shipment
*Category:* state-driven · *Source:* P2 R25–P2 R26
**While** every source Inbound `TU` feeding a `Shipment` does not have `GR_ACCEPTED` for cross-dock settlement, the WMS System must hold reporting the Shipment to ERP; later settlement of the remainder must not affect readiness.
```gherkin
Scenario: Putaway of remainder does not block Shipment
  Given cross-dock settlement has `GR_ACCEPTED` and the remainder waits for Putaway
  When the WMS System evaluates the Shipment gate
  Then the source TU satisfies the gate condition
  And later settlement of the remainder does not change readiness
```

### FR-P2-14 — GR gate re-evaluation and Supervisor visibility
*Category:* event-driven · *Source:* P2 R35–P2 R36
**When** a GR message (`GR_ACCEPTED` or `GR_REJECTED`) arrives for any required source Inbound `TU`, the WMS System must re-evaluate the `FR-P2-13`/`P2 R25` gate regardless of the previous result for that `TU` or the current `Shipment` state, including when `Shipment` is in `POSTING_ERROR` for another reason; explicit `GR_REJECTED` must not by itself transition `Shipment → POSTING_ERROR` — the gate remains unsatisfied until `GR_ACCEPTED`. `grAcceptanceStatus` of source `TU` objects blocking the gate must be visible to `Warehouse Supervisor`.
```gherkin
Scenario: Rejected GR does not move Shipment to POSTING_ERROR
  Given Shipment depends on two source TU objects
  When one receives explicit GR_REJECTED
  Then Shipment does not transition to POSTING_ERROR and the gate remains unsatisfied
  And Warehouse Supervisor sees grAcceptanceStatus of that source TU
```

### FR-P2-15 — Source TU finalization
*Category:* event-driven · *Source:* P2 R28
**When** no active `CrossDockPickTask` remains for a source Inbound `TU`, the WMS System must finalize that `TU` — `CROSS_DOCKED` if there is no residual quantity or `IN_PUTAWAY` with remainder at `TRANSIT` — and must not create another cross-dock task for it.
```gherkin
Scenario: New demand does not resume cross-docking
  Given all tasks of the source TU finished and residual quantity remained
  When a new BACKORDERED line for that SKU appears after finalization
  Then the WMS System does not create another CrossDockPickTask for that TU
  And demand is handled through standard flow after Putaway is complete
```

### FR-P2-16 — Quantity lock and one-time task generation
*Category:* universal · *Source:* P2 R29–P2 R30
The WMS System must maintain a quantity lock on `CrossDockPickTask`: the same physical source Inbound `TU`/`SKU` quantity may be `planned` or `confirmed` in at most one active task. Generation of `CrossDockPickTask` and `OutboundOrderLine` for a given source Inbound `TU` is a one-time event triggered by its transition to `IN_CROSS_DOCK`, matching only `CustomerOrderLine BACKORDERED` objects existing at that moment; there is no mechanism to generate more tasks for an already created `OutboundOrderLine` or update its required quantity after creation. Each cross-dock `OutboundOrderLine` is created together with exactly one `CrossDockPickTask`.
```gherkin
Scenario: Same quantity does not enter two tasks
  Given an active task reserves plannedQty on an SKU of the source TU
  When another task is created for the same SKU of that TU
  Then it cannot include quantity already reserved
  And no mechanism generates additional tasks for the already created OutboundOrderLine
```

### FR-P2-17 — GR result correlation and rejection
*Category:* event-driven · *Source:* P2 R31–P2 R32
**When** the WMS System receives a `Goods Receipt` result from ERP, it must correlate it exclusively by `sourceInboundTU` and `GR_SETTLEMENT_SOURCE = CROSSDOCK`, without a task identifier, and set the same `grAcceptanceStatus` on all `CrossDockPickTask` objects of that Inbound `TU`; a signal matching no `sourceInboundTU` or concerning another settlement source must be rejected without changing `grAcceptanceStatus`.
```gherkin
Scenario: Putaway settlement does not move the cross-dock gate
  Given tasks of the source TU already have GR_ACCEPTED from source CROSSDOCK
  When a GR result with source PUTAWAY arrives for the same TU
  Then grAcceptanceStatus of those tasks remains unchanged
  And settlement version number is not used as the source discriminator
```

### FR-P2-18 — Gate determined per Shipment
*Category:* state-driven · *Source:* P2 R33
**While** at least one source Inbound `TU` feeding a given `Shipment` does not have `GR_ACCEPTED` for cross-dock settlement, the WMS System must hold reporting of that `Shipment`, determining the condition separately for each `Shipment` from the complete set of its own source `TU` objects.
```gherkin
Scenario: One pallet feeds two Shipment objects
  Given source TU X fed a task of Shipment 1 and a task of Shipment 2
  When TU X receives GR_ACCEPTED for cross-dock settlement
  Then tasks of both Shipment objects have updated grAcceptanceStatus
  And each Shipment checks its own source-TU set separately
```

### FR-P2-19 — Empty target TU after recovery
*Category:* conditional · *Source:* P2 R34
**If** a target Outbound `TU` created during an active `CrossDockPickTask` loses all confirmed quantity through `PutBackTask` recovery and remains empty, while no active or planned `CrossDockPickTask` still points to it as target, then the WMS System must transition it `CREATED → CANCELLED` and must not allow it into `PACKING_SEALED`. While such a task points to it, the WMS System must leave it in `CREATED` as an empty open target unit. The WMS System must not make this cancellation dependent on reaching `slaDeadline`.
```gherkin
Scenario: Empty target TU with no references is cancelled
  Given all confirmed quantity of the target TU returned through PutBackTask
  When the TU has no other confirmed item and no active or planned task points to it as target
  Then it transitions to CANCELLED without waiting for slaDeadline
  And never enters PACKING_SEALED

Scenario: Empty target TU referenced by active task is not cancelled
  Given all confirmed quantity of the target TU returned through PutBackTask
  When another active CrossDockPickTask still points to this TU as target
  Then the TU remains in CREATED as an empty open target unit
```

### FR-P2-21 — CrossDockPickTask assignment by module selection
*Category:* event-driven · *Source:* P2 R39
**When** `Warehouse Operator` is logged into the cross-dock module and has no active warehouse task of any type, the WMS System must assign the next `CrossDockPickTask` according to configured warehouse order.
```gherkin
Scenario: Operator without a task gets next CrossDockPickTask
  Given the operator is logged into the cross-dock module and has no active task
  When the WMS System selects the next task to assign
  Then the operator receives the highest queued CrossDockPickTask
```

### FR-P2-22 — Target continuity of active CrossDockPickTask
*Category:* universal · *Source:* P2 R40
The WMS System must ensure that an active `CrossDockPickTask` always has an available target into which the operator can place confirmed quantity; if no open target Outbound `TU` of that task exists when the next item is placed, the WMS System must open a new Outbound `TU` and assign its `TU_NUMBER`, regardless of why the previous one is missing.
```gherkin
Scenario: Missing open TU at placement opens a new one
  Given an active CrossDockPickTask has no open target Outbound TU because the previous one was closed as physically full or cancelled as empty
  When the Packer places the next SKU item within that task
  Then the WMS System opens a new Outbound TU and assigns TU_NUMBER
  And creation occurs only at that placement, not in advance when the previous TU was closed
```

### FR-P2-23 — Only ELEMENTARY TU in cross-docking
*Category:* universal · *Source:* P2 R41
The WMS System must accept into cross-docking only Inbound `TU` of type `ELEMENTARY`; `TU` `AGGREGATE` must not reach `IN_CROSS_DOCK`, and when an `SKU` needed for a `BACKORDERED` line is declared in a `TU` attached to an aggregate, only the `TU` `ELEMENTARY` enters the process.
```gherkin
Scenario: Child, not aggregate, enters cross-docking
  Given an SKU needed for a BACKORDERED line is declared in an ELEMENTARY TU attached to an AGGREGATE TU
  When Inbound qualifies the goods for cross-docking
  Then the ELEMENTARY TU enters the Outbound Crossdock process
  And the AGGREGATE TU does not reach IN_CROSS_DOCK
```

### FR-P2-24 — Zero match at IN_CROSS_DOCK
*Category:* conditional · *Source:* P2 R42
**If** no matching `CustomerOrderLine` in `BACKORDERED` exists when the source Inbound `TU` transitions to `IN_CROSS_DOCK`, then the WMS System must not create any `CrossDockPickTask` and must finalize that `TU` with the entire declared quantity as residual, moving it to `TRANSIT` and toward `IN_PUTAWAY`.
```gherkin
Scenario: Demand disappeared during transport to the cross-dock area
  Given Inbound qualified the TU for cross-docking and before its transition to IN_CROSS_DOCK all matching CustomerOrderLine objects ceased to be BACKORDERED
  When the TU reaches IN_CROSS_DOCK
  Then the WMS System creates no CrossDockPickTask
  And the entire declared TU quantity is residual quantity
  And the TU is handed to Putaway with the full residual quantity
```

### FR-P2-25 — Inheritance of priority and slaDeadline in cross-docking
*Category:* event-driven · *Source:* P2 R43

**When** the WMS System creates a cross-dock `OutboundOrder`, it must inherit its `priority` and `slaDeadline` from the parent `CustomerOrder` and group lines into it only when customer, delivery address, `priority` and `slaDeadline` are compatible and the `slaDeadline` is identical; with `allowPartialShipment = false`, grouping must include only lines of one `CustomerOrder`.
```gherkin
Scenario: Inheritance of cross-dock OutboundOrder parameters
  Given the parent CustomerOrder has priority 2 and slaDeadline 2026-08-28T16:00:00
  When the WMS System creates a cross-dock OutboundOrder for it
  Then OutboundOrder receives priority 2 and slaDeadline 2026-08-28T16:00:00

Scenario: Grouping requires identical deadline
  Given two lines have compatible customer, address and priority, but different slaDeadline
  When the WMS System groups lines into a cross-dock OutboundOrder
  Then the lines do not enter one OutboundOrder
```

## P3 — Reservation Release

### FR-P3-01 — Release Allocation and Inventory
*Category:* event-driven · *Source:* P3 R1–P3 R2
**When** a reservation is released before physical picking, the WMS System must perform `Allocation RESERVED → RELEASED` and `Inventory RESERVED → AVAILABLE`.
```gherkin
Scenario: Release without physical work
  Given goods are reserved but not picked
  When the WMS System releases the reservation
  Then Allocation is `RELEASED`
  And Inventory is `AVAILABLE`
```

### FR-P3-02 — Cancel line on shortage
*Category:* conditional · *Source:* P3 R3–P3 R4
**If** release results from `SHORT_ALLOCATED`, the WMS System must cancel `OutboundOrderLine` in a pre-pick state and set the unfulfilled part of `CustomerOrderLine → BACKORDERED`.
```gherkin
Scenario: Shortage preserves demand
  Given Allocation has a shortage before picking
  When the reservation is released
  Then OutboundOrderLine is `CANCELLED`
  And CustomerOrderLine is `BACKORDERED`
```

### FR-P3-03 — General cancellation and auto-release
*Category:* configurable · *Source:* P3 R5–P3 R6
**When** the cause of release is general cancellation, the WMS System must set `CustomerOrderLine → CANCELLED`; automatic release according to policy must occur without notifying the Supervisor.
```gherkin
Scenario: Auto-release without escalation
  Given warehouse policy requires automatic release
  When the policy condition is met
  Then the reservation is released and the relevant line is cancelled
  And the Supervisor receives no mandatory escalation
```

### FR-P3-04 — Race window and handoff to P4
*Category:* conditional · *Source:* P3 R7–P3 R8
**If** release occurs after physical picking but before its confirmation, the WMS System must cancel `PickTask` and instruct return to the source location without `PutBackTask`; formally confirmed quantity must be routed to P4.
```gherkin
Scenario: Return to source without PutBackTask
  Given the operator physically picked the SKU but did not confirm the TU and quantity
  When Allocation is released
  Then PickTask is cancelled and RF instructs return to source
  And no formal PutBackTask is created
```

### FR-P3-05 — Reservation-retention policy variants
*Category:* configurable · *Source:* P3 R9

**Where** the warehouse configures a partial-reservation retention policy, the WMS System must offer three variants: retain the reservation, automatically release after a specified time, and route the case to a `Warehouse Supervisor` decision.
```gherkin
Scenario: Automatic-release variant
  Given the warehouse configured automatic release after a time
  When the retention time expires
  Then the WMS System releases Allocation without Warehouse Supervisor participation
```

### FR-P3-06 — Independence of reservation-retention time
*Category:* state-driven · *Source:* P3 R10

**While** a partial reservation is retained, the WMS System must not make its retention time depend on `priority` or `slaDeadline` of `CustomerOrder`.
```gherkin
Scenario: Priority does not shorten or lengthen time
  Given two CustomerOrder objects have different priority and different slaDeadline
  When both have partial reservation under the same warehouse policy
  Then reservation-retention time is identical for both
```

## P4 — Physical Put-back

### FR-P4-01 — Approval and POSTING_PENDING boundary
*Category:* conditional · *Source:* P4 R1–P4 R2
**If** the cancelled line is `PACKED`, the WMS System must require `Warehouse Supervisor` approval; cancellation must be allowed only before `Shipment POSTING_PENDING`.
```gherkin
Scenario: PACKED after POSTING_PENDING is not handled by P4
  Given the line is PACKED and Shipment has `POSTING_PENDING`
  When the Supervisor tries to approve cancellation
  Then the WMS System rejects the operation
  And indicates Return Receipt as the subsequent path
```

### FR-P4-02 — Immediate logical effect
*Category:* event-driven · *Source:* P4 R3–P4 R4
**When** cancellation is approved, the WMS System must immediately cancel the line, release `Allocation` and return quantity to `ATPReservation`, without waiting for physical put-back.
```gherkin
Scenario: Statuses do not wait for the operator
  Given cancellation of picked quantity was approved
  When PutBackTask remains incomplete
  Then OutboundOrderLine is `CANCELLED` and Allocation `RELEASED`
  And physical Inventory still waits to be put back
```

### FR-P4-03 — PutBackTask and location validation
*Category:* event-driven · *Source:* P4 R5–P4 R6
**When** a cancelled line has `pickedQty > 0`, the WMS System must create a separate `PutBackTask`; it must validate the indicated or proposed location before put-back.
```gherkin
Scenario: Picked quantity creates a task
  Given a cancelled line has positive `pickedQty`
  When the WMS System settles cancellation
  Then it creates PutBackTask for the picked quantity
  And does not allow placement into an unvalidated location
```

### FR-P4-04 — Location loop and Inventory recovery
*Category:* state-driven · *Source:* P4 R7–P4 R8
**While** the location is rejected, the WMS System must maintain the `IN_PROGRESS ↔ LOCATION_VALIDATION` loop without a limit and without escalation; after `PutBackTask COMPLETED` it must perform `Inventory PICKED → AVAILABLE`.
```gherkin
Scenario: Subsequent attempt completes PutBack
  Given the first location was rejected
  When the operator indicates another valid location and puts the goods back
  Then PutBackTask is completed
  And Inventory transitions to `AVAILABLE`
```

### FR-P4-05 — PutBackTask assignment by module selection
*Category:* event-driven · *Source:* P4 R9
**When** `Warehouse Operator` is logged into the returns module and has no active warehouse task of any type, the WMS System must assign the next `PutBackTask` in submission order.
```gherkin
Scenario: Operator without a task gets next PutBackTask
  Given the operator is logged into the returns module and has no active task
  When the WMS System selects the next task to assign
  Then the operator receives the oldest submitted PutBackTask
```

## P5 — Cross-cutting exceptions

### FR-P5-01 — SHORT_ALLOCATED without permission for partial shipment
*Category:* conditional · *Source:* P1 Exceptions — `SHORT_ALLOCATED`, `allowPartialShipment = false`
**If** a race for soft reservation causes `SHORT_ALLOCATED` with `allowPartialShipment = false`, the WMS System must escalate to `Warehouse Supervisor` the choice of permanent permission for partial shipment or cancellation of `OutboundOrder`, and leave the losing line in `BACKORDERED`.
```gherkin
Scenario: Supervisor resolves SHORT_ALLOCATED
  Given the line lost the race for part of the reservation
  When the Supervisor chooses to cancel OutboundOrder
  Then its Allocation objects are released and the losing line is BACKORDERED
  And CustomerOrder returns to waiting according to the state of the remaining lines
```

### FR-P5-02 — SHORT_ALLOCATED with permission for partial shipment
*Category:* conditional · *Source:* P1 Exceptions — `SHORT_ALLOCATED`, `allowPartialShipment = true`
**If** `SHORT_ALLOCATED` occurs with `allowPartialShipment = true`, the WMS System must fulfill the available `OutboundOrder` and leave the shortage in `BACKORDERED` until inventory appears.
```gherkin
Scenario: Partial quantity is fulfilled without escalation
  Given allowPartialShipment = true
  When Allocation covers only part of the quantity
  Then the existing OutboundOrder fulfills the available part
  And the shortage remains BACKORDERED
```

### FR-P5-03 — Automatic retry after SHORT_PICKED
*Category:* conditional · *Source:* P1 Exceptions — `SHORT_PICKED`
**When** a `PickTask` ends in `SHORT_PICKED`, the WMS System must block the location and create a new `PickTask` for the missing quantity while qualified ATP inventory exists in another location and the effective limit is not exhausted; otherwise it must escalate.
```gherkin
Scenario: New PickTask uses another location
  Given the first PickTask has SHORT_PICKED
  When ATP exists in an unblocked location and the limit has not been exhausted
  Then WMS creates a new PickTask for the missing quantity
  And OutboundOrder does not yet transition to packing
```

### FR-P5-04 — “Wait” outcome after SHORT_PICKED
*Category:* conditional · *Source:* P1 Exceptions — `SHORT_PICKED`, `allowPartialShipment = false`, “wait” outcome
**If** the Supervisor chooses the “wait” outcome, the WMS System must roll back unpacked lines of that `CustomerOrder`, release their allocations and keep already `PACKED` lines unchanged.
```gherkin
Scenario: Waiting rolls back only unpacked lines
  Given some lines are PACKED and some ALLOCATED or PICKED
  When the Supervisor chooses “wait”
  Then unpacked lines are rolled back and their Allocation objects released
  And PACKED lines remain unchanged
```

### FR-P5-05 — “Cancellation” outcome after SHORT_PICKED
*Category:* conditional · *Source:* P1 Exceptions — `SHORT_PICKED`, `allowPartialShipment = false`, “cancellation” outcome
**If** the Supervisor chooses cancellation, the WMS System must require editing `CustomerOrderLine` quantity: to zero with cancellation and return, or to the actually picked quantity without `PutBackTask`.
```gherkin
Scenario: Cancelling the execution line alone is insufficient
  Given allowPartialShipment = false and SHORT_PICKED occurred
  When the Supervisor chooses cancellation
  Then they must indicate a new CustomerOrderLine quantity
  And WMS selects the return according to that quantity
```

### FR-P5-06 — Permanent permission for partial shipment after SHORT_PICKED
*Category:* conditional · *Source:* P1 Exceptions — permanent change of `allowPartialShipment`
**If** the Supervisor permanently sets `allowPartialShipment = true`, the WMS System must require a reason, cancel the missing part and limit the effect to the current `CustomerOrder`.
```gherkin
Scenario: Permanent change has limited scope
  Given the Supervisor provided a reason
  When they approve allowPartialShipment = true
  Then the available quantity continues and the missing part is cancelled
  And other customer orders and warehouse parameters remain unchanged
```

### FR-P5-07 — Discrepancy during packing
*Category:* conditional · *Source:* P1 Exceptions — shortage or damage during “repack by SKU”
**When** a shortage or damage is detected during packing, the WMS System must start the same mechanism as for `SHORT_PICKED`, but without blocking the source location.
```gherkin
Scenario: Packing shortage uses SHORT_PICKED handling
  Given the operator counts SKU during repack by SKU
  When they detect a shortage or damage
  Then WMS starts SHORT_PICKED resolution
  And does not block the source location
```

### FR-P5-08 — ON_HOLD
*Category:* conditional · *Source:* P1 Exceptions — `CustomerOrder.ON_HOLD`
**While** `CustomerOrder` is `ON_HOLD`, the WMS System must skip it in cyclic planning; after the block is removed it must return it to the queue.
```gherkin
Scenario: Removing the block restores planning
  Given CustomerOrder is ON_HOLD
  When the block is removed
  Then the order returns to the planning queue
```

### FR-P5-09 — General cancellation
*Category:* conditional · *Source:* P1 Exceptions — cancellation request for `CustomerOrder` or `CustomerOrderLine`
**When** OMS/ERP or `Warehouse Supervisor` requests cancellation, the WMS System must evaluate each affected line according to `OutboundOrderLine` status, reject whole-order cancellation when at least one line blocks it, and must not allow cancellation after `CarrierManifest.CLOSED`.
```gherkin
Scenario: One blocking line rejects whole-order cancellation
  Given CustomerOrder has at least one line that does not meet the cancellation condition
  When a request to cancel the whole order arrives
  Then WMS rejects the request and indicates the blocking line
```

### FR-P5-10 — No Carrier Selection result
*Category:* conditional · *Source:* P1 Exceptions — no Carrier Selection result
**If** Carrier Selection returns no result, the WMS System must escalate to `Warehouse Supervisor`; for a `TU` on the external-carrier role type (`P1 R68`) whose `TUSetup.maxVolume` has no value, it must require manual selection by `Dispatcher` and approval before label creation.
```gherkin
Scenario: Manual selection precedes the label
  Given the `TU` is on the external-carrier role type (`P1 R68`) whose `TUSetup.maxVolume` has no value, and Carrier Selection returned no result
  When the Dispatcher chooses a carrier and the Supervisor approves it
  Then WMS may generate the label
```

### FR-P5-11 — Limited handling of label errors
*Category:* conditional · *Source:* P1 Exceptions — label error or carrier rejection
**If** a label error or electronic rejection is reported, the WMS System in version 1 must not start a separate failure mode; a loading problem before `CarrierManifest.CLOSED` must be handled by a manual `Carrier` change without reprinting the label.
```gherkin
Scenario: Version 1 has no electronic rejection
  Given a loading problem occurred before manifest close
  When the Supervisor manually changes Carrier
  Then WMS does not reprint the label
```

### FR-P5-12 — Cross-dock shortage without permission for partial shipment
*Category:* conditional · *Source:* P2 Exceptions — shortage or `DAMAGED`, `allowPartialShipment = false`
**If** a shortage or `DAMAGED` occurs during cross-dock picking with `allowPartialShipment = false`, the WMS System must escalate three outcomes: wait with `PutBackTask`, cancellation, or permanent permission for partial shipment.
```gherkin
Scenario: Supervisor selects one of three paths
  Given confirmedQty is lower than demand
  When the Supervisor chooses “wait”
  Then WMS creates PutBackTask for confirmedQty greater than zero
```

### FR-P5-13 — Cross-dock shortage with permission for partial shipment
*Category:* conditional · *Source:* P2 Exceptions — shortage or `DAMAGED`, `allowPartialShipment = true`
**If** a shortage or `DAMAGED` occurs during cross-dock picking with `allowPartialShipment = true`, the WMS System must automatically pass the sorted part to `PACKED` and leave the missing part in `BACKORDERED`.
```gherkin
Scenario: Partial cross-dock without escalation
  Given allowPartialShipment = true
  When picking ends with a shortage
  Then the sorted part is PACKED and the missing part BACKORDERED
```

### FR-P5-14 — Empty TU before picking
*Category:* conditional · *Source:* P2 Exceptions — empty `TU`
**When** a source `TU` proves empty before picking, the WMS System must cancel `CrossDockPickTask` and `OutboundOrderLine`, set `CustomerOrderLine BACKORDERED` and Inbound `TU LOST`.
```gherkin
Scenario: Empty TU cancels the task
  Given the TU is empty before picking
  When the operator confirms that state
  Then CrossDockPickTask and OutboundOrderLine are CANCELLED
  And CustomerOrderLine is BACKORDERED and Inbound TU is LOST
```

### FR-P5-15 — Cancellation during sorting
*Category:* conditional · *Source:* P2 Exceptions — general cancellation at `CrossDockPickTask IN_PROGRESS`
**If** general cancellation arrives during `CrossDockPickTask IN_PROGRESS`, the WMS System must reject the request and allow retry only after `PACKED` is reached.
```gherkin
Scenario: Cancellation is rejected during sorting
  Given CrossDockPickTask is IN_PROGRESS
  When general cancellation arrives
  Then WMS rejects the request and indicates the reason
```

### FR-P5-16 — Goods Receipt gate re-evaluation
*Category:* conditional · *Source:* P2 Exceptions — `P2 R35`–`P2 R36`
**When** any required source Inbound `TU` receives a GR result (`GR_ACCEPTED` or `GR_REJECTED`), the WMS System must re-evaluate the gate for reporting `Shipment` to ERP; explicit `GR_REJECTED` must not independently move `Shipment` to `POSTING_ERROR` — the gate remains unsatisfied until `GR_ACCEPTED`, and `grAcceptanceStatus` must be visible to `Warehouse Supervisor`.
```gherkin
Scenario: GR_REJECTED does not post Shipment to POSTING_ERROR
  Given a source TU feeds Shipment
  When its result is GR_REJECTED
  Then Shipment does not transition to POSTING_ERROR and the gate remains unsatisfied
```

### FR-P5-17 — Completed fragment waits for complete order
*Category:* state-driven · *Source:* P1 Exceptions — P5 E17
**While** a `CustomerOrder` with `allowPartialShipment = false` has some `CustomerOrderLine` that satisfies neither alternative (A) nor (B) of `P1 R58`, while other lines of the same `CustomerOrder` already have `OutboundOrderLine` in `PACKED`, the WMS System must leave the Packing `TU` objects of packed lines in `PACKING_SEALED` in the consolidation area, outside every `Shipment`, without cancelling `OutboundOrder` and without automatically releasing remaining lines into separate fulfillment. Waiting may end only by completing the order or by an existing `Warehouse Supervisor` path under `P1 R46` or `P1 R47`.
```gherkin
Scenario: Completed fragment waits before Shipment creation
  Given CustomerOrder has allowPartialShipment = false
  And some CustomerOrderLine satisfies neither alternative A nor B of P1 R58
  And other lines of this CustomerOrder have OutboundOrderLine in PACKED
  When the WMS System evaluates whether Shipment can be created or supplemented
  Then Packing TU objects of packed lines remain in PACKING_SEALED outside every Shipment
  And OutboundOrder is not cancelled
  And remaining lines are not automatically released into separate fulfillment
```

# 1b. Integration requirements

### INT-01 — Inbound boundary for cross-dock
*Source:* P2 STEP 1
**When** Inbound exposes quantity with state `IN_CROSS_DOCK`, the WMS System must accept the `TU` identifier, `SKU`, quantity declared in ASN — not physically verified by Inbound — and receipt correlation.
```gherkin
Scenario: Quantity basis comes from ASN declaration
  Given Inbound confirmed IN_CROSS_DOCK without opening the TU
  When the message reaches Outbound
  Then the base quantity is the quantity declared in ASN
  And P2 receives TU data, SKU and correlation
```

### INT-02 — Cross-dock result to Inbound
*Source:* P2 STEP 3
**When** cross-dock settlement of a source Inbound `TU` is closed, the WMS System must pass Inbound `confirmedQty` and `damagedQty` of that `TU` while preserving correlation; it does not pass residual quantity as a separate contract field — Inbound calculates it.
```gherkin
Scenario: Contract contains two quantities
  Given cross-dock settlement of the TU was closed
  When P2 publishes the result
  Then Inbound receives confirmedQty and damagedQty with TU correlation
  And calculates the remainder on its side as declaration minus confirmedQty minus damagedQty
```

### INT-03 — Goods Receipt correlation
*Source:* P2 STEP 4
**While** the result of the dependent `Goods Receipt` has not arrived from ERP, the WMS System must maintain correlation by `sourceInboundTU` and `GR_SETTLEMENT_SOURCE`; retrying the `Goods Receipt` message itself belongs to Inbound and is not an Outbound responsibility.
```gherkin
Scenario: Correlation persists until the result
  Given cross-dock settlement was passed to Inbound
  When the ERP result has not yet arrived
  Then correlation by sourceInboundTU remains active
  And Outbound does not retry the Goods Receipt message
```

### INT-04 — Shipment POST to ERP
*Source:* P1 STEP 13
**When** a shipment meets publication conditions, the WMS System must send ERP a `Shipment POST` message with a unique identifier.
```gherkin
Scenario: Shipment publication
  Given Shipment is ready for posting
  When the WMS System publishes Shipment POST
  Then ERP receives an unambiguously identifiable message
```

### INT-05 — ERP response and retry
*Source:* P1 STEP 13
**If** ERP rejects `Shipment POST`, the WMS System must set `POSTING_ERROR`, preserve error data and allow a safe retry; after acceptance it must set `POSTED`.
```gherkin
Scenario: Rejection and successful retry
  Given Shipment has POSTING_PENDING
  When ERP first rejects and then accepts the retry
  Then Shipment transitions through POSTING_ERROR to POSTED
```

### INT-06 — Cancellation from the ordering system
*Source:* P3 STEP 1, P4 STEP 1
**When** the ordering system sends a cancellation or correction, the WMS System must correlate the request with the proper line and select P3 or P4 according to formally confirmed `pickedQty`.
```gherkin
Scenario: Integration cancellation selects the process
  Given a request arrived for an existing line
  When the WMS System reads pickedQty
  Then it routes zero to P3 and a positive value to P4
```

# 1c. Non-functional requirements

Not applicable within B6 scope.

# 1d. Concurrency requirements

### CON-01 — Atomic ATP reservation
*Source:* P1 R4–P1 R6, P1 R52
**When** parallel planning runs compete for the same ATP inventory, the WMS System must atomically create at most one successful `ATPReservation` and corresponding `Allocation` for a given quantity.
```gherkin
Scenario: Two allocations do not reserve the same quantity
  Given two orders concurrently request the same inventory
  When both perform allocation
  Then the sum of successful reservations does not exceed available ATP
```

### CON-02 — Immutability after PickTask creation
*Source:* P1 R10
**While** a `PickTask` exists, the WMS System must not reallocate the quantity assigned to it.
```gherkin
Scenario: Parallel reallocation attempt is rejected
  Given PickTask already exists
  When another run tries to reallocate that quantity
  Then the assignment remains unchanged
```

### CON-03 — Single cross-dock assignment
*Source:* P2 R6, P2 R29–P2 R30
**When** multiple lines concurrently wait for the same cross-dock `SKU`, the WMS System must assign every unit of quantity at most once according to priority, determining `sourceEligibleQty` after subtracting `plannedQty` of active tasks and `confirmedQty` and `damagedQty` of completed `CrossDockPickTask` objects.
```gherkin
Scenario: Damaged quantity is not planned again
  Given TU declaration is 100, 30 is in progress, and completed tasks produced 40 confirmedQty and 10 damagedQty
  When two runs concurrently try to plan another task
  Then together they may cover at most 20
  And no unit of quantity enters two tasks
```

### CON-04 — Stable Shipment grouping
*Source:* P1 R37–P1 R41
**When** multiple packages concurrently reach the grouping boundary, the WMS System must apply a consistent boundary deadline and assign each package to at most one `Shipment`.
```gherkin
Scenario: Package belongs to one Shipment
  Given packages are being closed around the boundary deadline
  When grouping is performed
  Then each package enters at most one Shipment
```

### CON-05 — Single effects of ERP and manifest
*Source:* P1 R43–P1 R46
**When** an ERP response or manifest close is delivered again or concurrently, the WMS System must preserve a single business effect and must not regress a terminal state.
```gherkin
Scenario: Duplicate response does not duplicate the effect
  Given the response has already been settled
  When the same message arrives again
  Then the business state remains unchanged
```