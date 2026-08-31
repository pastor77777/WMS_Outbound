# WMS Outbound Golden Record

This document is the compact, agent-friendly canonical view of the active architect source. It is not a replacement for the immutable source: every implementation decision must still trace to a concrete source rule/requirement.

## 1. Scope

Outbound v1 contains four processes:

1. P1 `STANDARD_FULFILLMENT`
2. P2 `OUTBOUND_CROSSDOCK`
3. P3 `RESERVATION_RELEASE`
4. P4 `PHYSICAL_PUTBACK`

There is no separate Process 5. Cross-cutting exceptions are embedded in P1/P2 and traced through `FR-P5-*`.

`PickWave` is explicitly outside v1, including speculative extension points.

## 2. Domain

| Object | Canonical responsibility |
|---|---|
| `CustomerOrder` | External/commercial customer demand, SLA/priority/partial-shipment policy and aggregate fulfillment state |
| `CustomerOrderLine` | SKU/quantity demand and line-level fulfillment aggregation |
| `OutboundOrder` | WMS execution order created from grouped/split demand; channel is STANDARD or CROSSDOCK |
| `OutboundOrderLine` | Executable quantity of a CustomerOrderLine in one WMS fulfillment |
| `Allocation` | Hard inventory reservation for a standard OutboundOrderLine |
| `PickTask` | Executable RF picking task, normally scoped by zone |
| `TU` | One physical Technical Unit used in picking/packing/shipping; PickContainer/PackUnit are roles, not separate entities |
| `TUSetup` | TU issue/external-identifier/type thresholds and configuration |
| `Shipment` | Logistics grouping of outbound TUs, carrier selection, label and ERP posting lifecycle |
| `Carrier` | Carrier master/reference |
| `CarrierSetup` | Carrier applicability/configuration by Region and weight/volume ranges plus priority |
| `Region` | Delivery region; distinct from warehouse zone |
| `CarrierManifest` | Carrier handover grouping and final confirmation boundary |
| `Inventory` | Physical stock/ATP/reserved/picked/shipped settlement |
| `PutBackTask` | Physical return of already picked/packed material to available stock |
| `CrossDockPickTask` | RF work moving eligible inbound TU quantity directly to outbound demand |
| `Sequence` | Atomic identifier/number generation where defined by the architect model |

Core ownership split:

`CustomerOrder = demand` → `OutboundOrder = WMS execution plan` → `Shipment = logistics dispatch`.

Do not collapse these objects into one legacy “order”.

## 3. Cardinality and quantity model

- CustomerOrder ↔ OutboundOrder is many-to-many through lines/quantities.
- A CustomerOrderLine can be split across multiple OutboundOrderLine records.
- Multiple CustomerOrders may be grouped into one OutboundOrder only when architect grouping conditions allow it.
- Standard and Crossdock may jointly cover one demand; quantities already covered must not be double-planned.
- `requiredQty`, ATP soft reservation, hard `Allocation`, physically picked quantity and shipped quantity are distinct concepts and must stay traceable.

## 4. Standard Fulfillment P1

High-level flow:

`CustomerOrder receive/validate`
→ `ACCEPTED/REJECTED/ON_HOLD`
→ cyclic ATP/planning
→ `OutboundOrder`
→ hard `Allocation`
→ `PickTask`
→ RF picking into Picking TU
→ direct pack or packing/repack/consolidation
→ `Shipment`
→ Carrier Selection
→ WMS label
→ ERP Shipment POST with retry/error state
→ `CarrierManifest`
→ handover/confirmation
→ final Inventory/Allocation and order aggregation.

Key rules:

- `allowPartialShipment=false` protects CustomerOrder-level shipment completeness.
- partial Allocation does not automatically create another OutboundOrder for the missing quantity.
- a short pick may auto-reallocate only up to `maxAutomaticShortPickReallocations`; then Supervisor decision is required.
- default warehouse retry limit is 1; configured value may differ; 0 disables automatic retry.
- PickTask completion does not automatically close the Picking TU.
- a Picking TU may continue through later PickTasks/zones for the same execution when rules allow.
- external TU issue must satisfy `TUSetup` rules.
- Carrier auto-selection uses Region + weight/volume applicability; tie-break is narrowest volume range, then weight range, then unique Carrier priority.
- manual carrier is fallback when automatic selection has no result.
- WMS generates the label in v1; no external carrier Label API is part of target scope.
- `CarrierManifest.CLOSED` is an irreversible composition/cancellation boundary but is not physical handover.
- final quantity settlement occurs on `CarrierManifest.CONFIRMED`.
- an OutboundOrder is complete only after all contributing manifests are confirmed.

## 5. Outbound Crossdock P2

Inbound owns qualification of an ELEMENTARY inbound TU into `IN_CROSS_DOCK`. Outbound then:

- evaluates BACKORDERED demand in the configured priority queue;
- creates CROSSDOCK OutboundOrder/lines and `CrossDockPickTask`;
- does **not** create standard `Allocation`;
- calculates eligible quantity without double assignment;
- assigns the task through the RF module;
- moves confirmed quantity from the source inbound TU into an outbound TU;
- accounts for shortage/damage/unexpected/empty-TU outcomes;
- seals/finalizes the outbound TU according to physical-full/SLA/task conditions;
- joins the same Shipment/Carrier/ERP/Manifest downstream path as P1;
- returns residual inbound quantity to Inbound Putaway as defined by the boundary.

The ERP shipment gate must respect Goods Receipt correlation/status of contributing inbound TUs.

## 6. Reservation Release P3

P3 is logical release before physical picking or policy-driven release:

- verify that no confirmed physical pick makes P4 necessary;
- `Allocation RESERVED → RELEASED`;
- Inventory reserved quantity becomes available according to the architect rules;
- line/order state changes according to shortage/general cancellation reason;
- policy may keep reservation, auto-release after configured time, or route to Supervisor.

When the material has already been physically picked, logical cancellation is not sufficient: P4 owns physical recovery.

## 7. Physical Putback P4

After approved cancellation of already picked/packed material:

- logical cancellation/release is immediate;
- create `PutBackTask`;
- assign operator through the RF module;
- identify source material/TU;
- propose/validate destination location;
- reject an invalid location and remain in the placement loop;
- on completion update Inventory so the physical quantity becomes available again.

Inbound Putaway is not P4 business logic even if UI/task/location primitives can be reused.

## 8. State/event canon

The complete machine-readable transition catalog is `03_TRACEABILITY/state_event_transitions.csv`; the authoritative human source is `model_stanow_outbound.md`.

Principal terminal states:

- CustomerOrder: `REJECTED`, `CANCELLED`, `CLOSED`
- CustomerOrderLine: `FULFILLED`, `CANCELLED`
- OutboundOrder: `COMPLETED`, `CANCELLED`
- OutboundOrderLine: `SHIPPED`, `CANCELLED`
- Allocation: `CONSUMED`, `RELEASED`
- PickTask: `COMPLETED`, `SHORT_PICKED`, `CANCELLED`
- Shipment: `HANDED_TO_CARRIER`, `CANCELLED`
- CarrierManifest: `CONFIRMED`
- PutBackTask: `COMPLETED`
- CrossDockPickTask: `COMPLETED`, `CANCELLED`

`TUSetup`, `Carrier`, `CarrierSetup`, `Region` and `Sequence` are configuration/reference objects without their own lifecycle machine.

## 9. Integration contracts

`INT-01..06` cover:

- Inbound → Outbound crossdock boundary;
- Crossdock result back to Inbound;
- Goods Receipt correlation;
- Shipment POST to ERP;
- ERP response/retry;
- cancellation from the ordering system.

Integration implementation must preserve idempotency and observable retry/error state rather than hiding failures.

## 10. Concurrency

`CON-01..05` require protection of:

- atomic ATP/hard reservation behavior;
- immutability of allocation-relevant facts after PickTask creation;
- single assignment of crossdock quantity;
- stable Shipment grouping;
- exactly-once business effects for ERP/manifest transitions.

Database transactions, unique/idempotency keys and locks are implementation mechanisms; they are not substitutes for the behavioral requirement.

## 11. Acceptance model

The architect package contains 109 requirements and 96 named scenario definitions, with coverage mapping from 132 process rules + 17 exception rows to requirements/component criteria/scenarios.

Human-facing behavior must be demonstrated through real Mercato/Scanner user journeys. Playwright evidence proves the normal UI is traversable; final user-facing acceptance is Human Verified.
