# State / Event Catalog

The authoritative human source is `01_ARCHITECT_SOURCE/2026-08-31/model_stanow_outbound.md`. The exhaustive machine-readable transition rows are in `03_TRACEABILITY/state_event_transitions.csv`.

## Stateful objects

1. CustomerOrder
2. CustomerOrderLine
3. OutboundOrder
4. OutboundOrderLine
5. Allocation
6. PickTask
7. TU
8. Shipment
9. CarrierManifest
10. Inventory
11. PutBackTask
12. CrossDockPickTask

## Stateless configuration/reference objects

- TUSetup
- Carrier
- CarrierSetup
- Region
- Sequence

## Event rule

Events are consequences/evidence of architect-defined transitions. Event names, actors and conditions must be taken from the state model/transition index; implementation code must not create a competing lifecycle vocabulary. Process prose wins on behavioral divergence, while the state model remains the status/name catalog as defined in `AUTHORITY.md`.
