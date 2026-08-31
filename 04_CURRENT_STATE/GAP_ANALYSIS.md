# Target vs current GAP analysis

Legend: `REUSE`, `EXTEND`, `MIGRATE`, `SUPERSEDE`, `LEAVE_UNTOUCHED`.

| Target area | Current state | Classification | Required direction |
|---|---|---|---|
| CustomerOrder | legacy header-only SO + empty richer public Sales model | `MIGRATE/EXTEND` | define architect CustomerOrder ownership/integration without treating legacy SO as target |
| CustomerOrderLine | public Sales lines exist, legacy WMS SO has no lines | `EXTEND/MIGRATE` | persist line demand/status/quantity traceability needed by target |
| OutboundOrder | legacy `outbound_delivery` incompatible lifecycle | `SUPERSEDE/MIGRATE` | introduce target execution order and migrate/retire legacy active use |
| OutboundOrderLine | no target equivalent | `EXTEND` | implement quantity-bearing execution line |
| ATPReservation | ATP service exists, soft reservation queue target missing | `EXTEND` | implement queue/priority/recalculation behavior |
| Allocation | quantity reservation primitive exists | `EXTEND/MIGRATE` | implement target state machine + OOL ownership + atomic transitions |
| PickTask | generic tasks/locks exist | `EXTEND` | implement target PickTask model, zone ordering, assignment and guards |
| Picking RF | current Outbound Scanner is receiving flow | `SUPERSEDE` | build target picking journey |
| TU | shared production TU foundation exists | `REUSE/EXTEND/MIGRATE` | add Outbound role/lifecycle compatibility without breaking Inbound |
| TUSetup | partial/generic TU config only | `EXTEND` | implement external issuability/type/threshold policy |
| Packing | no target flow | `EXTEND` | implement direct pack/repack/consolidation and TU qualification |
| Shipment | Sales shipment partial candidate | `EXTEND/MIGRATE` | implement target WMS Shipment lifecycle and TU/line traceability |
| Carrier | references/provider integration exist | `REUSE/EXTEND` | map carrier master cleanly |
| CarrierSetup | missing target applicability model | `EXTEND` | Region + weight/volume ranges + priority/tie-break |
| Region | no confirmed target delivery-region model | `EXTEND` | implement carrier-selection region domain |
| WMS label | provider label fields exist but target says WMS-generated | `SUPERSEDE/EXTEND` | implement WMS label behavior, do not introduce carrier Label API |
| ERP posting | reusable orchestration/outbox/retry foundation | `REUSE/EXTEND` | implement target Shipment posting lifecycle/idempotency |
| CarrierManifest | target object absent | `EXTEND` | implement OPEN/CLOSED/HANDED_OVER/CONFIRMED and final settlement |
| Inventory settlement | reusable ledger/balances | `REUSE/EXTEND` | connect RESERVED/PICKED/SHIPPED/release/putback exactly once |
| Crossdock demand | primitive demand table exists | `REUSE/EXTEND` | map source demand safely |
| CrossDockPickTask | absent | `EXTEND` | implement P2 task, locks, qty/damage/source-TU guards |
| Crossdock RF | absent; no separate target work area allowed | `EXTEND` | implement module-entry assignment without invented selector |
| Reservation Release | target lifecycle absent | `EXTEND` | implement P3 policy/race/line/order effects |
| PutBackTask | absent | `EXTEND` | implement P4 task and physical recovery |
| Inbound Putaway | accepted, different process | `LEAVE_UNTOUCHED` | reuse primitives only; protect with regression |
| Cancellation | legacy shortcuts exist | `SUPERSEDE/EXTEND` | implement correct pre/post-pick/packed/posting/manifest boundaries |
| Concurrency/idempotency | locks/idempotency primitives exist | `REUSE/EXTEND` | implement CON-01..05 at authoritative server/DB boundaries |
| Scanner hardware | Honeywell bridge stub | `EXTEND` | only required hardware behavior proven by scope; never claim untested hardware |
| Human acceptance | mature Playwright evidence exists mainly for Inbound | `REUSE/EXTEND` | create Outbound user journeys and Human Verified walkthrough evidence |

## Migration safety

Legacy Outbound has real test data and shared TU has substantial state. Supersede work must be additive/reversible until validation proves cutover. Do not destructively remove legacy structures in the first migration.

## Product blockers

None identified. All rows above are implementation work.
