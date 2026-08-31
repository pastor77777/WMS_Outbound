# WMSAI Outbound — Test scenarios

**Project:** WMSAI Outbound  
**Date:** 28.08.2026 | **Author:** Solution Analyst (Solution Architect competencies)  
**Source:** `wymagania_outbound.md`, `proces_1_standard_fulfillment.md` v1.20, `proces_2_outbound_crossdock.md` v1.13, `proces_3_reservation_release.md` v1.2, `proces_4_physical_putback.md` v1.2, `model_stanow_outbound.md` v1.19

## Scope and levels

- **Component tests (K)** — Given/When/Then acceptance criteria attached to every requirement in `wymagania_outbound.md`; they are not duplicated here.
- **Scenario tests (S)** — the E2E flows and cross-cutting cases below: happy path, cardinality, consolidation, exceptions, integration and concurrency.

**Convention:** `TC-nnn` · *Level* · *Related requirements* · *Data* · Gherkin (`Given / When / Then / And`).

---

## 1. P1 — Standard Fulfillment

### TC-001 — Full standard flow
*Level:* E2E · *Requirements:* FR-P1-01/02/03/04/05/06/07/08/09/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/26/27 · *Data:* 1 order, 1 line, full ATP
```gherkin
Scenario: Order goes from import to POSTED
  Given a valid order has full ATP inventory
  When WMS plans, allocates, picks, packs and dispatches the shipment
  Then Shipment transitions to POSTED
  And order and line states reflect completion
```

### TC-002 — No ATP and replanning
*Level:* Process · *Requirements:* FR-P1-03/04/05/06
```gherkin
Scenario: Line without ATP returns to the queue
  Given ATP for the SKU is zero
  When the planning cycle tries to create Allocation
  Then the line does not move to picking
  And remains available for the next cycle
```

### TC-003 — Partial fulfillment
*Level:* Process · *Requirements:* FR-P1-04/05/25/26
```gherkin
Scenario: Available part of the quantity is fulfilled
  Given allowPartial = true and ATP covers part of the request
  When WMS finishes allocation
  Then only the assigned quantity goes to PickTask
  And non-ATP inventory is not reserved
```

### TC-004 — Multi-zone picking
*Level:* Process · *Requirements:* FR-P1-06/07/08/09
```gherkin
Scenario: Several PickTask objects feed one line
  Given inventory is located in several zones
  When operators confirm all tasks
  Then the result is consolidated for the proper line
  And each quantity is settled once
```

### TC-005 — Direct pack
*Level:* Process · *Requirements:* FR-P1-09/10/11
```gherkin
Scenario: Goods go directly into the package
  Given configuration allows direct pack
  When picking is confirmed
  Then WMS creates and closes the package without separate consolidation
```

### TC-006 — Repacking and consolidation
*Level:* Process · *Requirements:* FR-P1-10/11/12/13
```gherkin
Scenario: Multiple TU objects form shipping packages
  Given picking was performed into several TU objects
  When the operator consolidates and packs the contents
  Then packages compliant with packing rules are created
  And linkage to lines remains preserved
```

### TC-007 — Carrier Selection and label
*Level:* Process · *Requirements:* FR-P1-14/15/16/17/18
```gherkin
Scenario: Correct carrier and label
  Given a closed package has complete transport data
  When WMS performs Carrier Selection and generates the label
  Then the package is ready to be assigned to Shipment
```

### TC-008 — Manifest and ERP
*Level:* E2E · *Requirements:* FR-P1-19/20/21/22/23/24
```gherkin
Scenario: Dispatch is manifested and posted
  Given Shipment meets the boundary deadline and manifest conditions
  When the manifest is closed and ERP accepts Shipment POST
  Then Shipment is POSTED
  And CarrierManifest remains closed
```

### TC-009 — State aggregation
*Level:* Process · *Requirements:* FR-P1-25/27
```gherkin
Scenario: Header state follows line states
  Given order lines are in different states
  When one line changes its terminal state
  Then WMS recalculates states of parent objects
```

### TC-010 — Outbound TU number uniqueness
*Level:* Process · *Requirements:* FR-P1-28
```gherkin
Scenario: Active TU number is not duplicated
  Given an active Outbound TU with a given TU_NUMBER exists in the warehouse
  When another Outbound TU is created
  Then it receives a different TU_NUMBER
  And the number is preserved when Picking TU continues in the Packing TU role
  And a TU_NUMBER that is not SSCC is alphanumeric, has at most 20 characters and is encoded in Code 128
```

### TC-104 — Incomplete order is not dispatched
*Level:* Process · *Requirements:* FR-P1-31

```gherkin
Scenario: Incomplete order is not dispatched
  Given CustomerOrder has allowPartialShipment = false
  When at least one required line is missing
  Then the WMS System does not dispatch Shipment
```

### TC-105 — Deadline passes before full packing
*Level:* Process · *Requirements:* FR-P1-32

```gherkin
Scenario: Deadline passes before full packing
  Given CustomerOrder has allowPartialShipment = false and its OutboundOrder has not reached PACKED
  When slaDeadline expires
  Then the Packing TU of this order remains in PACKING_SEALED
  And does not enter any closing Shipment
```

### TC-106 — One customer line goes to two orders
*Level:* Process · *Requirements:* FR-P1-33

```gherkin
Scenario: One customer line goes to two orders
  Given CustomerOrderLine requires more quantity than one order will cover
  When the WMS System plans OutboundOrder
  Then the line is split across multiple OutboundOrderLine objects
```

### TC-107 — Different address blocks joint packing
*Level:* Process · *Requirements:* FR-P1-34

```gherkin
Scenario: Different address blocks joint packing
  Given two OutboundOrder objects have different delivery addresses
  When the Packer tries to repack their lines into one Packing TU
  Then the WMS System does not allow joint packing
```

### TC-108 — Manual removal of flag while cause persists
*Level:* Process · *Requirements:* FR-P1-35

```gherkin
Scenario: Manual removal of flag while cause persists
  Given CustomerOrder has WARNING because full reservation is missing
  When Warehouse Supervisor sets the flag to false while reservation remains incomplete
  Then the next planning run sets WARNING again
  And the order's business behavior does not change
```

### TC-109 — Shared TU variant passing through tasks
*Level:* Process · *Requirements:* FR-P1-36

```gherkin
Scenario: Shared TU variant passing through tasks
  Given the warehouse configured the shared Picking TU variant
  When the operator completes a task and continues the same OutboundOrder in another zone
  Then they pick into the same Picking TU
```

### TC-110 — Weight calculated from contents
*Level:* Process · *Requirements:* FR-P1-37

```gherkin
Scenario: Weight calculated from contents
  Given TU contains 20 units of an SKU with unit weight 2 kg and 10 units of an SKU weighing 1 kg
  When the WMS System determines TU weight
  Then weight is 50 kg
```

### TC-111 — Carrier selection by weight calculated from contents
*Level:* Process · *Requirements:* FR-P1-37

```gherkin
Scenario: Carrier selection by weight calculated from contents
  Given Shipment contains two Packing TU objects with current weights 50 kg and 80 kg, on packaging types with 800 kg load capacity
  When the WMS System starts Carrier Selection
  Then it uses 80 kg, not 800 kg, for matching the weight interval
```

### TC-114 — Light but well-filled TU meets thresholds
*Level:* Process · *Requirements:* FR-P1-38

```gherkin
Scenario: Light but well-filled TU meets thresholds
  Given the `TU` type has `TUSetup.externalIssuable = true` and `TUSetup.processUsage` other than `EXTERNAL`, while current weight has not reached `TUSetup.minIssueWeight`
  When current content volume reaches `TUSetup.minIssueVolume`
  Then the WMS System treats issuance thresholds as satisfied
```

### TC-115 — Last TU of the order below thresholds
*Level:* Process · *Requirements:* FR-P1-39

```gherkin
Scenario: Last TU of the order below thresholds
  Given the `TU` has `TUSetup.externalIssuable = true` and `TUSetup.processUsage` other than `EXTERNAL`, has not reached lower thresholds and is the last `TU` of its `OutboundOrder`
  When `Warehouse Operator` closes it and provides a reason
  Then the WMS System dispatches the `TU` without repacking
  And records the force reason
```

### TC-116 — Repacking into a non-issuable type
*Level:* Process · *Requirements:* FR-P1-40

```gherkin
Scenario: Repacking into a non-issuable type
  Given the Packer repacks goods into a carrier type without externalIssuable
  When they try to seal Packing TU
  Then the WMS System does not allow it to be sealed
```

### TC-117 — Non-issuable type blocks operator force
*Level:* Process · *Requirements:* FR-P1-39

```gherkin
Scenario: Last TU on a non-issuable type cannot be dispatched
  Given the `TU` has `TUSetup.externalIssuable = false`, has not reached lower thresholds and is the last `TU` of its `OutboundOrder`
  When `Warehouse Operator` tries to close it and force issuability
  Then the WMS System does not allow it to be sealed or dispatched
  And the only path is repacking into an issuable type according to `P1 R66`
```

### TC-118 — Continue PickTask in the next Picking TU
*Level:* Process · *Requirements:* FR-P1-41

```gherkin
Scenario: One task uses successive Picking TU objects
  Given `PickTask` has more quantity to pick than fits in the first Picking `TU`
  When the first `TU` reaches `PICK_FULL`, the Picker closes it and scans another Picking `TU`
  Then the first `TU` transitions `PICK_FULL → READY_TO_PACK`, and the second `CREATED → IN_PICKING`
  And the same `PickTask` keeps its identity and remains `IN_PROGRESS`
  And `PickTask` reaches `COMPLETED` only after the full quantity is picked
```

### TC-120 — Recognition and issuability of an externally originated TU
*Level:* Process · *Requirements:* FR-P1-38/FR-P1-39/FR-P1-42

```gherkin
Scenario: Type indicated by tuSetupCode determines TU origin
  Given the catalog contains exactly one `TUSetup` with `processUsage = EXTERNAL` and `externalIssuable = true`
  And `TU.tuSetupCode` points to that type
  When the WMS System recognizes `TU` origin and evaluates issuance thresholds
  Then it recognizes it as external exclusively by `processUsage` of the referenced type, without using `TU_NUMBER` or an additional flag
  And the first branch of `P1 R64` treats thresholds as satisfied without reading `minIssueWeight` or `minIssueVolume`
  And manual forcing under `P1 R65` is not required
```

### TC-121 — requiredQty lifecycle
*Level:* Process · *Requirements:* FR-P1-43

```gherkin
Scenario: TEST A — standard correction
  Given `CustomerOrderLine.Quantity = 100`, and its one `OutboundOrderLine.requiredQty = 100`
  And `SHORT_PICKED` ended with `pickedQty = 30`
  When `Warehouse Supervisor` corrects `CustomerOrderLine.Quantity` to 30 in `P1 R46`
  Then `OutboundOrderLine.requiredQty = 30`

Scenario: TEST B — cross-dock correction
  Given cross-dock `OutboundOrderLine.requiredQty = 100`, and its only `CrossDockPickTask.confirmedQty = 30`
  When `Warehouse Supervisor` corrects `CustomerOrderLine.Quantity` to 30 in the `P2 R15` branch
  Then `OutboundOrderLine.requiredQty = 30`
  And the `PACKED` condition compares `confirmedQty = 30` with `requiredQty = 30`

Scenario: TEST C — no correction and equality condition
  Given `CustomerOrderLine.Quantity = 100`, and the only `OutboundOrderLine` has `requiredQty = 30` and status `PACKED`
  And no `OutboundOrderLine` exists for the remaining quantity
  When the WMS System evaluates the equality condition
  Then the sum `requiredQty = 30` is not equal to `CustomerOrderLine.Quantity = 100`
  And the condition remains unsatisfied
```

### TC-122 — Guard for active line without OutboundOrderLine
*Level:* Process · *Requirements:* FR-P1-32

```gherkin
Scenario: TEST 1 — set of OutboundOrderLine objects for line A is empty
  Given `CustomerOrder.allowPartialShipment = false` and it has lines A and B
  And line B was fulfilled by cross-dock and its `OutboundOrderLine` has `requiredQty` equal to line B `Quantity` and reached `PACKED`
  And line A is `BACKORDERED`, has no `OutboundOrderLine` and has `Quantity > 0`
  When the WMS System evaluates alternative (B) from `P1 R58` for line A
  Then the sum of `requiredQty` of the empty set is 0 and is not equal to line A `Quantity`
  And Packing `TU` of line B remains in `PACKING_SEALED` outside every `Shipment`
```

### TC-123 — Guard for partially covered PLANNED line
*Level:* Process · *Requirements:* FR-P1-32

```gherkin
Scenario: TEST 2 — PLANNED status does not open the guard with partial coverage
  Given `CustomerOrder.allowPartialShipment = false`, and `CustomerOrderLine.Quantity = 100`
  And `crossDockEligibleQty = 30`
  And one cross-dock `OutboundOrderLine.requiredQty = 30` reached `PACKED`
  And `CustomerOrderLine` is `PLANNED`, while no `OutboundOrderLine` exists for the remaining 70
  When the WMS System evaluates alternative (B) from `P1 R58`
  Then the sum `requiredQty = 30` is not equal to `Quantity = 100`
  And Packing `TU` remains in `PACKING_SEALED` outside every `Shipment`
```

### TC-124 — Guard after correction and with multiple lines
*Level:* Process · *Requirements:* FR-P1-32

```gherkin
Scenario: TEST 3 — STANDARD correction opens the guard
  Given `CustomerOrderLine.Quantity = 100`, and one `OutboundOrderLine.requiredQty = 100`
  And the line reached `SHORT_PICKED` with `pickedQty = 30`
  When `Warehouse Supervisor`, according to `P1 R46`, corrects `CustomerOrderLine.Quantity` and `OutboundOrderLine.requiredQty` to 30, and the line reaches `PACKED`
  Then the sum `requiredQty = 30` equals `Quantity = 30`
  And alternative (B) from `P1 R58` is satisfied

Scenario: TEST 3b — CROSSDOCK correction opens the guard
  Given the cross-dock line was planned for 100 and `confirmedQty = 30`
  When `Warehouse Supervisor`, according to `P2 R15`, corrects `CustomerOrderLine.Quantity` and `OutboundOrderLine.requiredQty` to 30, and the line reaches `PACKED`
  Then the sum `requiredQty = 30` equals `Quantity = 30`
  And alternative (B) from `P1 R58` is satisfied

Scenario: TEST 3c — two OutboundOrderLine objects together cover the line
  Given `CustomerOrderLine.Quantity = 100`
  And two non-cancelled `OutboundOrderLine` objects in `PACKED` have `requiredQty` 30 and 70 respectively
  When the WMS System evaluates alternative (B) from `P1 R58`
  Then the sum `requiredQty = 30 + 70 = 100` equals `Quantity = 100`
  And alternative (B) is satisfied
```

### TC-125 — One Shipment for two channels
*Level:* Process · *Requirements:* FR-P1-31/FR-P1-32

```gherkin
Scenario: TEST 4 — complete lines from two channels enter one Shipment
  Given `CustomerOrder.allowPartialShipment = false` has one line fulfilled through `STANDARD` and another through `CROSSDOCK`
  And for both lines the sum of `requiredQty` of non-cancelled `OutboundOrderLine` objects in `PACKED` equals their `Quantity`
  And both channels have identical `slaDeadline`
  When the WMS System evaluates `P1 R57` and `P1 R58`
  Then both Packing `TU` objects enter one complete `Shipment`
```

### TC-126 — slaDeadline does not bypass CustomerOrder guard
*Level:* Process · *Requirements:* FR-P1-32

```gherkin
Scenario: TEST 5 — deadline does not dispatch an order in parts
  Given `CustomerOrder.allowPartialShipment = false` has a Packing `TU` in `PACKING_SEALED`
  And at least one active `CustomerOrderLine` does not satisfy alternative (B) from `P1 R58`
  When the common `slaDeadline` expires
  Then the Packing `TU` remains outside every `Shipment`
  And automatic close of an existing `Shipment` according to `P1 R28` cannot include it
  And the order does not leave the warehouse in part
```

### TC-128 — First confirmed manifest does not complete OutboundOrder
*Level:* Process · *Requirements:* FR-P1-44 · *Data:* 1 OutboundOrder, 2 Packing TU, 2 Shipment

```gherkin
Scenario: Order with TU objects in two Shipment objects remains DISPATCHED
  Given `Shipment` A was closed automatically by `slaDeadline` with the first Packing `TU` of the order (`P1 R28`)
  And the second Packing `TU` of the same `OutboundOrder` created `Shipment` B (`P1 R29`)
  And the order reached `READY_FOR_DISPATCH` and then `DISPATCHED`
  When the `CarrierManifest` containing `Shipment` A reaches `CONFIRMED`
  Then `OutboundOrder` remains in `DISPATCHED`
  And `OutboundOrderLine` objects dispatched by `Shipment` A transition `PACKED → SHIPPED`, while lines from `Shipment` B are not changed
  And `CustomerOrder` does not reach `CLOSED`
```

### TC-129 — Last confirmed manifest completes OutboundOrder
*Level:* Process · *Requirements:* FR-P1-44 · *Data:* as TC-128, both manifests confirmed

```gherkin
Scenario: Confirmation of the last Shipment closes the order
  Given the state after `TC-128`: `Shipment` A confirmed, `Shipment` B unconfirmed
  When the `CarrierManifest` containing `Shipment` B reaches `CONFIRMED`
  Then `OutboundOrder` transitions `DISPATCHED → COMPLETED`

Scenario: Reverse confirmation order gives the same result
  Given the same `OutboundOrder` has Packing `TU` objects in `Shipment` A and B
  When first the manifest containing `Shipment` B reaches `CONFIRMED`, and only then the manifest containing `Shipment` A
  Then `OutboundOrder` remains in `DISPATCHED` after the first confirmation
  And transitions `DISPATCHED → COMPLETED` only after the second
```

### TC-130 — SHORT allocation blocks only quantity actually reserved
*Level:* Process · *Requirements:* FR-P1-45 · *Data:* `requiredQty` 100, available 30

```gherkin
Scenario: Partial reservation does not occupy full required quantity
  Given `OutboundOrderLine.requiredQty` is 100 and `AVAILABLE` inventory is 30
  When the WMS System performs `Allocation PENDING → SHORT`
  Then `Allocation.reservedQty` is 30
  And inventory quantity occupied by this allocation is 30
  And after inventory replenishment and transition `SHORT → RESERVED`, `reservedQty` is 100
```

### TC-131 — RELEASED and CONSUMED do not block inventory
*Level:* Process · *Requirements:* FR-P1-45 · *Data:* one `Allocation` in a full lifecycle

```gherkin
Scenario: Release and dispatch zero occupied quantity
  Given `Allocation` is in `CONFIRMED` with `reservedQty` equal to 100
  When the allocation transitions to `CONSUMED` on dispatch or to `RELEASED` on cancellation
  Then `reservedQty` returns to 0
  And the allocation contributes nothing to inventory quantity considered occupied
  And an allocation in `PENDING` also contributes nothing
```

### TC-132 — First dispatch does not terminalize line or Allocation
*Level:* Process · *Requirements:* FR-P1-46 · *Data:* one `OutboundOrderLine` 100, two Outbound `TU` objects of 50, two `Shipment` objects

```gherkin
Scenario: Line split across two Shipment objects — first manifest
  Given `OutboundOrderLine.requiredQty` is 100 and `Allocation.reservedQty` is 100
  And TU-A with 50 units belongs to `Shipment` A and TU-B with 50 units to `Shipment` B
  When the manifest of `Shipment` A reaches `CONFIRMED` while `Shipment` B is not yet confirmed
  Then `Inventory` settles 50 units as `SHIPPED`
  And `Allocation.reservedQty` is 50, while `Allocation` remains in `CONFIRMED`
  And `OutboundOrderLine` remains in `PACKED`
  And `OutboundOrder` remains in `DISPATCHED` according to `P1 R70`
```

### TC-133 — Final dispatch closes line, Allocation and order
*Level:* Process · *Requirements:* FR-P1-46 · *Data:* state after TC-132

```gherkin
Scenario: Second manifest closes settlement
  Given the state after `TC-132`: `Shipment` A confirmed, `Shipment` B unconfirmed
  When the manifest of `Shipment` B reaches `CONFIRMED`
  Then `Inventory` settles the remaining 50 units as `SHIPPED`
  And `Allocation.reservedQty` reaches 0 and `Allocation` transitions `CONFIRMED → CONSUMED`
  And `OutboundOrderLine` transitions `PACKED → SHIPPED`
  And if this was the last unconfirmed `Shipment` of the order, `OutboundOrder` transitions `DISPATCHED → COMPLETED` according to `P1 R70`

Scenario: Reverse confirmation order gives the same result
  Given the same line has TU-A in `Shipment` A and TU-B in `Shipment` B
  When first the manifest of `Shipment` B reaches `CONFIRMED`, and only then the manifest of `Shipment` A
  Then after the first confirmation the line remains in `PACKED` and `Allocation` in `CONFIRMED`
  And both terminal transitions occur only after the second confirmation
```

### TC-135 — Mixed coverage opens the STEP 2A gate
*Level:* Process · *Requirements:* FR-P1-03 · *Data:* `allowPartialShipment = false`, line 100, cross-dock covers 40

```gherkin
Scenario: Line covered by cross-dock does not block planning
  Given `CustomerOrder.allowPartialShipment = false` and its only `CustomerOrderLine` has `Quantity` 100
  And a cross-dock `OutboundOrderLine` with `requiredQty` 40 reached `PACKED`, while line `ATPReservation` is 60
  When cyclic planning starts and the WMS System evaluates the `STEP 2A` gate
  Then the line is considered fully covered because 60 plus 40 equals 100
  And a standard `OutboundOrder` is created only for the uncovered quantity, which is 60

Scenario: Full cross-dock coverage creates no standard order
  Given a cross-dock `OutboundOrderLine` with `requiredQty` 100 reached `PACKED`, while `ATPReservation` is 0
  When the WMS System evaluates the `STEP 2A` gate
  Then the line is fully covered
  And no standard `OutboundOrder` is created at all

Scenario: Missing coverage still stops the order
  Given `ATPReservation` is 60 and no cross-dock `OutboundOrderLine` reached `PACKED`
  When the WMS System evaluates the `STEP 2A` gate
  Then the order remains in `ACCEPTED` with `WARNING`
  And no standard `OutboundOrder` is created
```

## 2. P2 — Outbound Crossdock

### TC-020 — Cross-dock 1:1, full flow
*Level:* E2E · *Requirements:* FR-P2-01/02/03/04
```gherkin
Scenario: Full one-to-one match
  Given the entire declared contents of one source Inbound TU cover demand of the same customer and address, with compatible priority and identical slaDeadline
  When the TU reaches IN_CROSS_DOCK and the Packer confirms sorting of the entire declared contents
  Then one CrossDockPickTask, one target Outbound TU and one Shipment are created
  And OutboundOrder has fulfillmentChannel CROSSDOCK and no Allocation is created
```

### TC-021 — n:n sorting
*Level:* E2E · *Requirements:* FR-P2-02/03/12
```gherkin
Scenario: Several source TU objects feed several lines
  Given two source Inbound TU objects contain SKU for three compatible lines
  When the Packer sorts the entire declared contents of both TU objects
  Then each line receives confirmed quantity and both source TU objects reach CROSS_DOCKED
  And the CROSS_DOCKED criterion depends on no residual quantity, not on the number of target TU objects
```

### TC-022 — Full target TU during an active task
*Level:* Process · *Requirements:* FR-P2-05
```gherkin
Scenario: Full target TU does not complete the task
  Given CrossDockPickTask has plannedQty 80 and the target Outbound TU holds 50
  When the Packer closes the full TU by RF scan
  Then that TU transitions to PACKING_SEALED and the task continues sorting the remaining 30 into a new TU
  And general cancellation submitted during this time is rejected
```

### TC-023 — Shortage without permission for partial shipment
*Level:* Process · *Requirements:* FR-P2-05/06/07/08
```gherkin
Scenario: Escalation and wait outcome
  Given allowPartialShipment is false and the Packer confirmed a quantity lower than planned
  When Warehouse Supervisor chooses the wait outcome
  Then OutboundOrderLine transitions to CANCELLED from PICKING and CustomerOrderLine to BACKORDERED
  And for confirmedQty greater than zero a PutBackTask is created
```

### TC-024 — Shortage with permission for partial shipment
*Level:* Process · *Requirements:* FR-P2-06/07
```gherkin
Scenario: Part of the quantity continues automatically
  Given allowPartialShipment is true and confirmed quantity is lower than planned
  When the task is completed
  Then the sorted part of OutboundOrderLine transitions to PACKED without escalation
  And the missing quantity returns to BACKORDERED
```

### TC-025 — DAMAGED, unexpected SKU and damagedQty
*Level:* Process · *Requirements:* FR-P2-06/07/09/11
```gherkin
Scenario: Damaged goods do not return to source TU
  Given during picking a DAMAGED quantity and an SKU outside the ASN declaration were detected
  When the Packer confirms both discrepancies
  Then both goods go physically to QC and do not feed the target Outbound TU
  And only damagedQty count returns to Inbound, without the unexpected SKU and without physical return of goods
```

### TC-026 — Empty source TU before picking
*Level:* Process · *Requirements:* FR-P2-09
```gherkin
Scenario: No contents before picking starts
  Given the task is ASSIGNED and OutboundOrderLine is CREATED
  When the source Inbound TU proves empty
  Then the task and line transition to CANCELLED, demand returns to BACKORDERED and Inbound TU receives LOST
  And this case does not include a TU correctly emptied after sorting, which reaches CROSS_DOCKED
```

### TC-027 — Multiple tasks on one pallet and residual quantity
*Level:* Process · *Requirements:* FR-P2-10/11/12/15
```gherkin
Scenario: Sixty-to-forty settlement with three tasks
  Given the source TU has ASN declaration 100 and three tasks with plannedQty 30, 20 and 10
  When the Packer reports completion of each one in turn
  Then each task reaches COMPLETED separately and the source TU remains in cross-docking until the last task is completed
  And after all complete, cross-dock settlement covers 60, remainder 40 goes to TRANSIT, the TU transitions to IN_PUTAWAY and receives no further cross-dock task

Scenario: Damaged quantity does not enter the remainder expected in TU
  Given the source TU has ASN declaration 100 and completed tasks produced total confirmedQty 50 and damagedQty 10
  When the WMS System finalizes the source TU and passes the result to Inbound
  Then residual quantity is 40, not 50 — damaged quantity left the TU for QC and is not expected in it
  And Inbound receives confirmedQty 50 and damagedQty 10 as separate components
```

### TC-028 — Goods Receipt gate and rejection
*Level:* Integration · *Requirements:* FR-P2-13/14
```gherkin
Scenario: Shipment waits for cross-dock settlement, rejected GR does not block permanently
  Given Shipment depends on two source Inbound TU objects
  When the first receives GR_ACCEPTED for cross-dock settlement and the second has not yet done so
  Then Shipment does not transition to POSTING_PENDING and remains in LABEL_GENERATED or OWN_TRANSPORT
  And explicit GR_REJECTED for the second TU does not move Shipment to POSTING_ERROR, while a later GR_ACCEPTED for the same TU satisfies the gate without a separate recovery path
```

### TC-029 — Eligible quantity with active tasks
*Level:* Process · *Requirements:* FR-P2-03/16
```gherkin
Scenario: Damaged quantity does not return to the eligible pool
  Given ASN declaration is 100, active tasks have plannedQty 30, and completed tasks have 40 confirmedQty and 10 damagedQty
  When the WMS System calculates sourceEligibleQty for a new task
  Then the result is 20
  And the same physical quantity cannot be planned or confirmed in two active tasks
```

### TC-030 — One source pallet, two Shipment objects
*Level:* Integration · *Requirements:* FR-P2-02/05/16/17/18
```gherkin
Scenario: One GR result updates tasks of both shipments
  Given source TU X fed task A of shipment 1 and task B of shipment 2
  When TU X receives one GR_ACCEPTED for cross-dock settlement
  Then grAcceptanceStatus of both tasks is set, without a separate signal per Shipment
  And each shipment checks its own source-TU set separately, while target Outbound TU objects of tasks A and B are independent — each belongs to exactly one Shipment
```

### TC-031 — Putaway settlement after cross-dock settlement
*Level:* Integration · *Requirements:* FR-P2-13/17
```gherkin
Scenario: Signal from PUTAWAY source does not move the cross-dock gate
  Given tasks of the source TU have GR_ACCEPTED from source CROSSDOCK and Shipment has already been reported
  When a GR result from source PUTAWAY, accepted or rejected, arrives for the same TU
  Then grAcceptanceStatus of cross-dock tasks remains unchanged
  And Shipment readiness is not regressed
```

### TC-032 — Numbering at 1:1 with valid GS1
*Level:* Process · *Requirements:* FR-P2-04
```gherkin
Scenario: Number and SSCC inheritance
  Given TU_NUMBER of the source Inbound TU is a valid GS1 SSCC and a full 1:1 match occurs
  When the target Outbound TU is created
  Then it inherits TU_NUMBER and SSCC
  And no new label is printed
```

### TC-033 — Numbering at 1:1 without valid GS1
*Level:* Process · *Requirements:* FR-P2-04
```gherkin
Scenario: Renumbering and new label
  Given TU_NUMBER of the source Inbound TU is not a valid GS1 SSCC despite a full 1:1 match
  When the target Outbound TU is created
  Then it receives a new number according to TUSetup and Sequence
  And a new label is printed
```

### TC-034 — Numbering during n:n sorting
*Level:* Process · *Requirements:* FR-P2-04
```gherkin
Scenario: New number regardless of source number
  Given n:n sorting occurs and one source TU has a valid GS1 SSCC
  When the target Outbound TU is created
  Then it receives a new number from the Sequence referenced by its TUSetup
  And no source number is inherited
```

### TC-035 — Empty target TU after full recovery
*Level:* Process · *Requirements:* FR-P2-10/19
```gherkin
Scenario: Recovery of all quantity cancels target TU
  Given the target Outbound TU contains only quantity that lost demand
  When PutBackTask recovers all of that quantity into Inventory AVAILABLE
  And no active or planned task points to this TU as target
  Then the target TU transitions from CREATED to CANCELLED without waiting for slaDeadline
  And no Allocation release step or entry into PACKING_SEALED occurs

Scenario: Another task pointing to the same TU prevents cancellation
  Given task A placed quantity into target TU X and task B is active and also points to TU X as target
  When PutBackTask recovers all quantity placed by task A and TU X remains empty
  Then TU X remains in CREATED and does not transition to CANCELLED
  And task B retains a valid target reference
```

### TC-036 — demandEligibleQty as limiting factor
*Level:* Process · *Requirements:* FR-P2-03
```gherkin
Scenario: demandEligibleQty limits eligible quantity
  Given sourceEligibleQty of the source TU is 50 and CustomerOrderLine Quantity minus ATPReservation and the sum of requiredQty of its non-cancelled OutboundOrderLine objects is 20
  When the WMS System calculates crossDockEligibleQty
  Then the result is 20, lower than sourceEligibleQty
  And fulfillmentChannel of OutboundOrder remains CROSSDOCK
```

### TC-134 — Hard reservation does not disappear from the demandEligibleQty formula
*Level:* Process · *Requirements:* FR-P2-03 · *Data:* `Quantity` 100, one standard `OutboundOrderLine` 30, `ATPReservation` 0

```gherkin
Scenario: Quantity moved from ATPReservation to an order still reduces the cross-dock pool
  Given `CustomerOrderLine.Quantity` is 100
  And planning created a standard `OutboundOrderLine` with `requiredQty` 30, while `P1 R11` reduced `ATPReservation` to 0
  And the missing 70 remains `BACKORDERED`
  When the source Inbound `TU` reaches `IN_CROSS_DOCK` and the WMS System calculates `demandEligibleQty` of this line
  Then the result is 70, not 100
  And the sum of quantities assigned to both channels does not exceed `Quantity`

Scenario: Cancelled OutboundOrderLine does not reduce the pool
  Given the same standard `OutboundOrderLine` became `CANCELLED` and its quantity returned to `ATPReservation`
  When the WMS System recalculates `demandEligibleQty`
  Then the cancelled line is not subtracted
  And the result is not changed by double subtraction of the same quantity
```

### TC-037 — Automatic close after slaDeadline
*Level:* Process · *Requirements:* FR-P2-05
```gherkin
Scenario: Automatic close after slaDeadline and joining a late delivery before deadline
  Given the target Outbound TU no longer has active or planned CrossDockPickTask objects and its slaDeadline has not yet arrived
  When another Inbound TU arrives on the same day for the same compatible customer/address/priority/slaDeadline before slaDeadline passes
  Then the new task may still feed the same open TU, which does not close prematurely
  And when slaDeadline is reached with no further active/planned tasks, the WMS System automatically closes the TU into PACKING_SEALED
```

### TC-038 — Recovering the gate from POSTING_ERROR unrelated to GR
*Level:* Integration · *Requirements:* FR-P2-14
```gherkin
Scenario: Gate recovers Shipment from POSTING_ERROR caused by another reason
  Given Shipment is in POSTING_ERROR due to ERP rejection unrelated to the GR gate and one source TU still lacks GR_ACCEPTED
  When that source TU receives GR_ACCEPTED
  Then the gate is re-evaluated regardless of the POSTING_ERROR cause
  And Warehouse Supervisor could read grAcceptanceStatus of that TU throughout
```

### TC-039 — General OutboundOrder cancellation criterion outside empty-TU path
*Level:* Process · *Requirements:* FR-P2-08/09
```gherkin
Scenario: OutboundOrder cancelled by wait outcome, not by empty TU
  Given the only line of OutboundOrder reaches CANCELLED through a wait decision on shortage, without passing through an empty source TU
  When this is the only line of that OutboundOrder and no line reached PACKED
  Then OutboundOrder transitions to CANCELLED
  And the header-close criterion is general (P2 R37), independent of the path leading to line cancellation
```

## 3. P3 — Reservation Release

### TC-040 — Release before picking
*Level:* Process · *Requirements:* FR-P3-01/02/03
```gherkin
Scenario: Cancellation releases reservation
  Given pickedQty = 0
  When the line is cancelled
  Then Allocation and ATPReservation are released
  And no PutBackTask is created
```

### TC-041 — Automatic release
*Level:* Process · *Requirements:* FR-P3-01/02/03
```gherkin
Scenario: Expiration starts P3
  Given the automatic-release condition is met
  When WMS settles Allocation
  Then reservation is released and line state is updated
```

### TC-042 — Race window
*Level:* Concurrency · *Requirements:* FR-P3-04
```gherkin
Scenario: Physical pick without confirmation
  Given goods were physically picked but PickTask was not confirmed
  When release occurs concurrently
  Then WMS cancels PickTask and instructs return to source without PutBackTask
```

### TC-043 — Confirmed pick moves to P4
*Level:* Process · *Requirements:* FR-P3-04
```gherkin
Scenario: P3 does not return formally picked quantity
  Given pickedQty > 0
  When the line is cancelled
  Then the case is handed to P4
```

### TC-112 — Automatic-release variant
*Level:* Process · *Requirements:* FR-P3-05

```gherkin
Scenario: Automatic-release variant
  Given the warehouse configured automatic release after a time
  When reservation-retention time expires
  Then the WMS System releases Allocation without Warehouse Supervisor participation
```

### TC-113 — Priority does not shorten or lengthen time
*Level:* Process · *Requirements:* FR-P3-06

```gherkin
Scenario: Priority does not shorten or lengthen time
  Given two CustomerOrder objects have different priority and different slaDeadline
  When both have partial reservation under the same warehouse policy
  Then reservation-retention time is identical for both
```

## 4. P4 — Physical Put-back

### TC-050 — Cancel PACKED before boundary
*Level:* E2E · *Requirements:* FR-P4-01/02/03/04
```gherkin
Scenario: Supervisor approves put-back
  Given the line is PACKED and Shipment does not have POSTING_PENDING
  When the Supervisor approves cancellation
  Then logical effects are immediate and PutBackTask leads Inventory to AVAILABLE
```

### TC-051 — PutBackTask happy path
*Level:* Process · *Requirements:* FR-P4-02/03/04
```gherkin
Scenario: Goods return to a valid location
  Given a PutBackTask exists for pickedQty > 0
  When the operator confirms a valid location and placement
  Then PutBackTask is COMPLETED and Inventory is AVAILABLE
```

### TC-052 — Location-validation loop
*Level:* Process · *Requirements:* FR-P4-03/04
```gherkin
Scenario: Second location is valid
  Given the first location was rejected
  When the operator provides a valid next location
  Then the task completes without escalation and without an attempt limit
```

### TC-053 — Zero pickedQty
*Level:* Process · *Requirements:* FR-P4-02
```gherkin
Scenario: No physical task
  Given pickedQty = 0
  When cancellation is settled
  Then no PutBackTask is created
```

## 5. P5 — Cross-cutting exceptions

### TC-060 — SHORT_ALLOCATED
*Level:* Cross-cutting · *Requirements:* FR-P5-01/02
```gherkin
Scenario: Partial-fulfillment flag selects outcome
  Given Allocation does not cover the full quantity
  When allowPartial is false and then true
  Then WMS respectively releases all or retains the assigned part
```

### TC-061 — SHORT_PICKED: automatic retry and escalation
*Level:* Cross-cutting · *Requirements:* FR-P5-03
```gherkin
Scenario: Limit determines transition to Supervisor
  Given the first PickTask has SHORT_PICKED
  When another ATP location is available and then the retry limit is exhausted
  Then WMS first creates a new PickTask and later escalates the unresolved shortage
```

### TC-062 — SHORT_PICKED resolution
*Level:* Cross-cutting · *Requirements:* FR-P5-04/05/06
```gherkin
Scenario: Each decision has a distinct effect
  Given SHORT_PICKED requires a Supervisor decision
  When they choose in turn “wait”, cancellation or permanent permission for partial shipment
  Then WMS respectively rolls back unpacked lines, requires quantity editing, or fulfills the part with a provided reason
```

### TC-063 — Shortage or damage during packing
*Level:* Cross-cutting · *Requirements:* FR-P5-07
```gherkin
Scenario: Packing uses SHORT_PICKED mechanism
  Given the operator detected a shortage or damage during repack by SKU
  When they confirm the discrepancy
  Then WMS starts SHORT_PICKED handling without blocking the source location
```

### TC-064 — ON_HOLD and resume
*Level:* Cross-cutting · *Requirements:* FR-P5-08
```gherkin
Scenario: Block stops execution
  Given the object has ON_HOLD
  When the block is removed after an earlier execution attempt
  Then the process returns to the proper point of the main path
```

### TC-065 — General cancellation
*Level:* Cross-cutting · *Requirements:* FR-P5-09
```gherkin
Scenario: Blocking line rejects whole-order cancellation
  Given one CustomerOrder line does not meet the cancellation condition
  When OMS requests cancellation of the whole order
  Then WMS rejects the whole request and indicates the blocking line
```

### TC-066 — Carrier Selection and label limitation
*Level:* Cross-cutting · *Requirements:* FR-P5-10/11
```gherkin
Scenario: Manual selection without a separate label-failure mode
  Given Carrier Selection returned no result for a `TU` on a type with `processUsage = EXTERNAL` whose `TUSetup.maxVolume` has no value
  When the Dispatcher chooses Carrier and the Supervisor approves the choice
  Then a label may be created and version 1 does not start an electronic-rejection mode
```

### TC-067 — Cross-dock exceptions
*Level:* Cross-cutting · *Requirements:* FR-P5-12/13/14/15/16
```gherkin
Scenario: Five exceptions have explicit outcomes
  Given cross-dock encounters a shortage, empty TU, cancellation in progress or GR_REJECTED
  When WMS settles the relevant condition
  Then it chooses escalation or partial fulfillment, cancels empty TU, rejects cancellation in progress, or leaves Shipment before POSTING_PENDING until GR_ACCEPTED
```

### TC-127 — Completed fragment waits for complete CustomerOrder
*Level:* Cross-cutting · *Requirements:* FR-P5-17
```gherkin
Scenario: P5 E17 is not shortage E12 or E13
  Given `CustomerOrder.allowPartialShipment = false`
  And some `CustomerOrderLine` satisfies neither alternative (A) nor (B) from `P1 R58`
  And other lines of the same `CustomerOrder` already have `OutboundOrderLine` in `PACKED`
  When the WMS System evaluates whether `Shipment` can be created or supplemented
  Then Packing `TU` objects of packed lines remain in `PACKING_SEALED` in the consolidation area and outside every `Shipment`
  And `OutboundOrder` is not cancelled and remaining lines are not automatically released into separate fulfillment
  And waiting ends only by completing the order or by a `Warehouse Supervisor` path from `P1 R46` or `P1 R47`
```

## 6. Integrations

### TC-080 — Inbound–Outbound boundary
*Level:* Integration · *Requirements:* INT-01
```gherkin
Scenario: Quantity basis comes from ASN declaration
  Given Inbound publishes TU data without physical verification of its contents
  When state reaches IN_CROSS_DOCK
  Then Outbound receives SKU, quantity declared in ASN and receipt correlation
  And that quantity is the basis for calculating sourceEligibleQty
```

### TC-081 — Cross-dock result returns to Inbound
*Level:* Integration · *Requirements:* INT-02/03
```gherkin
Scenario: Return contract includes two quantities
  Given cross-dock settlement of the source TU was closed
  When P2 publishes the result to Inbound
  Then Inbound receives confirmedQty and damagedQty correlated by TU, without a separate residual-quantity field
  And correlation by sourceInboundTU lasts until the ERP result arrives, while retrying the Goods Receipt message belongs to Inbound
```

### TC-082 — ERP accepts Shipment POST
*Level:* Integration · *Requirements:* INT-04/05
```gherkin
Scenario: Correct posting
  Given Shipment is ready
  When ERP accepts Shipment POST
  Then Shipment transitions from POSTING_PENDING to POSTED
```

### TC-083 — ERP rejects and accepts retry
*Level:* Integration · *Requirements:* INT-05
```gherkin
Scenario: Safe retry
  Given ERP returned an error
  When the same Shipment POST is correctly retried
  Then state transitions from POSTING_ERROR to POSTED without a duplicate effect
```

### TC-084 — Cancellation from ordering system
*Level:* Integration · *Requirements:* INT-06
```gherkin
Scenario: Request correlation with line
  Given the ordering system sends cancellation
  When WMS finds the line and its pickedQty
  Then it selects P3 or P4 for the same line
```

## 7. Concurrency

### TC-090 — Competition for ATP
*Level:* Concurrency · *Requirements:* CON-01
```gherkin
Scenario: No double reservation
  Given two orders compete for one ATP quantity
  When allocations execute concurrently
  Then the sum of ATPReservation does not exceed ATP inventory
```

### TC-091 — Reallocation after PickTask
*Level:* Concurrency · *Requirements:* CON-02
```gherkin
Scenario: Existing PickTask protects assignment
  Given PickTask was created
  When a concurrent run tries to reallocate the quantity
  Then WMS rejects the assignment change
```

### TC-092 — Competition for cross-dock quantity
*Level:* Concurrency · *Requirements:* CON-03
```gherkin
Scenario: Damaged quantity is not planned again
  Given TU declaration is 100, 30 is in progress, and completed tasks produced 40 confirmedQty and 10 damagedQty
  When two runs concurrently try to plan another task
  Then together they may cover at most 20
  And no unit of quantity enters two tasks
```

### TC-093 — Shipment boundary deadline
*Level:* Concurrency · *Requirements:* CON-04
```gherkin
Scenario: Package does not enter two shipments
  Given packages close around the boundary deadline
  When grouping executes concurrently
  Then each package belongs to at most one Shipment
```

### TC-094 — Duplicate ERP and manifest
*Level:* Concurrency · *Requirements:* CON-05
```gherkin
Scenario: Repeated message has one effect
  Given the ERP response or manifest close has already been settled
  When the same message arrives again
  Then the terminal state does not regress and the effect is not duplicated
```

### TC-096 — PickTask assignment by zone and order
*Level:* Process · *Requirements:* FR-P1-29
```gherkin
Scenario: Operator without a task gets next PickTask in their zone
  Given the operator is logged into the picking module in zone A and has no active task
  When the WMS System selects the next task to assign
  Then the operator receives the highest queued PickTask of zone A
```

### TC-097 — Operator with active task receives no next task
*Level:* Process · *Requirements:* FR-P1-29
```gherkin
Scenario: Operator with active task receives no next task
  Given the operator already has an active PickTask
  When another PickTask waits in their zone
  Then the WMS System does not assign another task
```

### TC-098 — Continue order in next zone into open TU
*Level:* Process · *Requirements:* FR-P1-30
```gherkin
Scenario: Operator continues order in another zone
  Given the operator completed a zone-A PickTask for an open Picking TU
  When they choose to continue the same OutboundOrder in zone B
  Then they receive the zone-B PickTask of this order outside queue order
  And their active zone changes to B
```

### TC-099 — CrossDockPickTask assignment by module selection
*Level:* Process · *Requirements:* FR-P2-21
```gherkin
Scenario: Operator without a task gets next CrossDockPickTask
  Given the operator is logged into the cross-dock module and has no active task
  When the WMS System selects the next task to assign
  Then the operator receives the highest queued CrossDockPickTask
```

### TC-100 — PutBackTask assignment in submission order
*Level:* Process · *Requirements:* FR-P4-05
```gherkin
Scenario: Operator without a task gets next PutBackTask
  Given the operator is logged into the returns module and has no active task
  When the WMS System selects the next task to assign
  Then the operator receives the oldest submitted PutBackTask
```

### TC-101 — Opening new target TU during an active task
*Level:* Process · *Requirements:* FR-P2-22
```gherkin
Scenario: Closing or cancelling target TU does not leave task without a target
  Given CrossDockPickTask is active and its target Outbound TU was closed as physically full or cancelled as empty
  When the Packer places another SKU item within the same task
  Then the WMS System opens a new Outbound TU and assigns its TU_NUMBER
  And the task continues sorting without escalation to Warehouse Supervisor
```

### TC-102 — Only ELEMENTARY TU enters cross-docking
*Level:* Process · *Requirements:* FR-P2-23
```gherkin
Scenario: Child, not aggregate, enters cross-docking
  Given an SKU needed for a BACKORDERED line is declared in an ELEMENTARY TU attached to an AGGREGATE TU
  When Inbound qualifies those goods for cross-docking
  Then the ELEMENTARY TU enters the Outbound Crossdock process
  And the AGGREGATE TU does not reach IN_CROSS_DOCK
```

### TC-103 — Zero match at IN_CROSS_DOCK ends in Putaway
*Level:* Process · *Requirements:* FR-P2-24
```gherkin
Scenario: Demand disappeared during transport to cross-dock area
  Given Inbound qualified the TU for cross-docking and before its transition to IN_CROSS_DOCK all matching CustomerOrderLine objects ceased to be BACKORDERED
  When the TU reaches IN_CROSS_DOCK
  Then the WMS System creates no CrossDockPickTask
  And the entire declared TU quantity is residual quantity
  And the TU is handed to Putaway with full residual quantity
```

### TC-119 — Inheritance and grouping of cross-dock OutboundOrder
*Level:* Process · *Requirements:* FR-P2-25

```gherkin
Scenario: Cross-dock OutboundOrder inherits order parameters
  Given CustomerOrder has priority 2, slaDeadline 2026-08-28T16:00:00 and allowPartialShipment = false
  When the WMS System creates a cross-dock OutboundOrder for part of its lines
  Then OutboundOrder receives priority 2 and slaDeadline 2026-08-28T16:00:00
  And does not group lines from any other CustomerOrder
```