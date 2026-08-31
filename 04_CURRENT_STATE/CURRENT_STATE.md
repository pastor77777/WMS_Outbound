# Current implementation baseline

**Audit date:** 2026-08-31  
**Mode:** read-only source/database analysis before ETAP 2 writes.

This is evidence of current implementation, not business authority.

## 1. WMS_Outbound

At audit start the repository was empty (`size=0`, no established knowledge/implementation structure). ETAP 2 creates only documentation/canon/plan/steering artifacts here.

## 2. Mercato

Observed active `main` during final audit: `fa97bf03906632440613fb16fe25a5e897d0ebe01`.

Active WMS modules include:

- `wms_comms`
- `wms_inbound`
- `wms_inventory`
- `wms_orchestration`
- `wms_putaway`
- `wms_receiving`
- `wms_scanner_api`
- `wms_tu`
- `wms_warehouse`

There is no active `apps/mercato/src/modules/wms_outbound` module and no active Sales/Shipment/Carrier module under that WMS module tree.

### Reusable foundations

**Inventory — REUSE/EXTEND**

`wms_inventory` provides balances, lots/serials, reservations, append-only movement ledger, correlation/idempotency metadata, ATP service and tests. Current reservation rows are quantity records, not target `Allocation` lifecycle objects.

**TU — REUSE/EXTEND/MIGRATE**

`wms_transport_units` is a shared, heavily used model with contents, queue/assignment and transport-task capabilities. Its current `process_status` vocabulary is primarily Inbound. Outbound lifecycle must not overwrite accepted Inbound semantics.

**Warehouse/actor context — REUSE/EXTEND**

Active warehouse selection/assignment APIs exist. WMS ACL features and zone/warehouse APIs provide an existing authorization pattern.

**Record locks/task mechanics — REUSE/EXTEND**

Scanner and backend already use server-side TU record locks, heartbeat/release and task ownership patterns. These are technical primitives; target Outbound single-active-task and assignment guards still require explicit implementation.

**Orchestration/integration — REUSE/EXTEND**

`wms_orchestration` includes durable event/outbox facts, retry-attempt logging and idempotent posting records/services. This is a strong foundation for Outbound ERP/status delivery, but target Shipment posting states and business effects remain to implement.

**Putaway — LEAVE BUSINESS LOGIC UNTOUCHED / REUSE TECHNICAL PRIMITIVES**

Inbound Putaway has rich task, zone, quantity, location, suspend/set-down and placement flows. It is not Outbound Physical Putback. Only generic mechanics/UX patterns may be reused where architect behavior permits.

**Scanner API — SUPERSEDE for Outbound flow**

Current `wms_scanner_api` endpoints are receiving/unloading oriented: gates, gate-scan, buffer-confirm, complete-unloading and shortage settlement. Target Outbound Scanner endpoints are missing.

### Crossdock

Database has `wms_cross_dock_demands`, but no active Mercato code search/module equivalent for target `CrossDockPickTask`. Existing demand/source data is a partial primitive, not a P2 implementation.

## 3. Scanner/RF

Observed `Devaxonic-scanner` `main`: `0feeb88702fd377b0ff5f19471e710038bf691ef`.

### Current Outbound mode

The app exposes an `outbound` mode but routes it to the same receiving-style flow as Inbound:

`mode → gate → unit/ASN list → documents → scan TU → buffer → quantity → complete receiving`.

The application comment explicitly identifies the earlier Outbound pass as UI-only with backend wiring deferred. The Outbound branch therefore does not implement target P1/P2/P4 behavior.

**Classification: SUPERSEDE.**

### Scanner primitives

Potential `REUSE/EXTEND`:

- login/session and reauthentication;
- 401 refresh/retry request wrapper;
- active warehouse selection/persistence;
- common screen/navigation components;
- scan abstraction and barcode sanitization;
- TU record lock + heartbeat/release;
- error/retry UX;
- Putaway/Transport task-oriented UX patterns.

Known implementation issue from audit: persisted session restoration references a locally scoped `session` variable outside its `try` scope. Treat as an implementation defect when Scanner code is next changed.

Honeywell hardware integration remains a stub; manual/simulated scan works. Hardware-specific acceptance must not be claimed without real-device evidence.

### Missing target flows

- PickTask queue/assignment and picking journey;
- multi-zone continuation and Picking TU behavior;
- short-pick/reallocation/Supervisor outcomes;
- packing/direct-pack/repack/consolidation;
- CrossDockPickTask journey;
- Physical PutBack journey;
- target single-active-warehouse-task enforcement/feedback;
- target-specific cancellation/exception UX.

## 4. scanner-context

`scanner-context` is a technical knowledge source for scanner/RF/AIDC architecture, standards/vendor facts, idempotency/concurrency/offline guidance and testing/observability.

It explicitly places project architecture above generic scanner knowledge and current implementation facts in source code.

Routing for Outbound:

1. Outbound Architect Source / Canon = business truth.
2. `scanner-context` = generic technical guidance only.
3. `Devaxonic-scanner` = current implementation/evidence only.

## 5. Database — Supabase DevAxonic_Platform

Project ref audited read-only: `yzonugcenguvmojwiihb`.

### Legacy Sales Order / Outbound

`wms_sales_order.so_header`:

- header-only schema; no line table exists under `wms_sales_order`;
- 320 rows at final audit query;
- current statuses: `DRAFT`, `CONFIRMED`, `RECEIVED`, `SHIPPED`, `CANCELLED`;
- generated internal SO numbering via PostgreSQL sequence;
- document-registry/warehouse constraints.

`wms_outbound.outbound_delivery`:

- 182 rows;
- statuses `DRAFT`, `CONFIRMED`, `PICKED`, `PACKED`, `SHIPPED`, `CANCELLED`;
- FK to `wms_sales_order.so_header`;
- legacy outbound internal sequence;
- trigger path marks the legacy SO header `SHIPPED` when outbound delivery becomes `SHIPPED`.

This shortcut is incompatible with the target line/Shipment/CarrierManifest aggregation and final settlement at `CarrierManifest.CONFIRMED`.

**Classification: SUPERSEDE/MIGRATE.**

### Public Sales candidates

Present but empty in test data:

- `sales_orders`: 0
- `sales_order_lines`: 0
- `sales_shipments`: 0
- `sales_shipment_items`: 0

This model is much richer than legacy `so_header` and includes customer/order snapshots, quantities, fulfillment fields and shipment relations. It is a candidate integration/source model, not automatically the target CustomerOrder domain.

### Inventory

- `wms_inventory_balances`: 25 rows
- `wms_inventory_reservations`: 0 rows
- movement/lot/serial structures exist.

Target `Allocation` needs explicit lifecycle/state, source OutboundOrderLine ownership and atomic transition semantics on top of/alongside this foundation.

### TU/tasks

- `wms_transport_units`: 2305 rows
- `wms_transport_tasks`: 30 rows
- `wms_putaway_tasks`: 478 rows

Transport Unit current statuses are predominantly Inbound-oriented. Shared-table compatibility is a major regression concern.

### Crossdock

- `wms_cross_dock_demands`: 3 rows, all `BACKORDERED`
- one current TU was observed in `IN_CROSS_DOCK` during status audit.

This is not a complete target P2 implementation.

### Carrier/manifest

`carrier_shipments` exists as provider/tracking/label integration storage but has 0 rows and is not the target WMS `CarrierManifest`.

No target `CarrierManifest` table/model was found in the active audited database.

### Sequence

PostgreSQL sequences already exist for legacy internal SO/outbound numbering and many framework migrations. Target `Sequence` should use an atomic implementation appropriate to architect-defined numbering while avoiding accidental reuse of legacy business meaning.

## 6. Current-state conclusion

Outbound is not greenfield because Inventory, TU, warehouse/task/orchestration and legacy Sales/Outbound data exist. It is also not an extension of one coherent legacy Outbound module. The implementation must introduce the architect domain while deliberately reusing shared foundations and superseding incompatible legacy lifecycle shortcuts.
