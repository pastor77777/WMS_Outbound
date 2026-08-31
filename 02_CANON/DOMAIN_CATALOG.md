# Domain Catalog

The canonical 17-object domain catalog is summarized below. Attribute-level implementation must be traced to source requirements/process/state model rather than invented here.

| Object | Stateful | Owning business role | Current-state implementation note |
|---|---|---|---|
| CustomerOrder | yes | customer demand header | No confirmed target-equivalent; legacy `wms_sales_order.so_header` is header-only and conflicting; public Sales is a candidate input model |
| CustomerOrderLine | yes | SKU/qty demand | public `sales_order_lines` is a candidate source, currently empty test data |
| OutboundOrder | yes | WMS execution | legacy `wms_outbound.outbound_delivery` must be superseded/migrated |
| OutboundOrderLine | yes | execution quantity | target-equivalent missing |
| Allocation | yes | hard reservation | Inventory reservation foundation exists but lacks target lifecycle/line ownership |
| PickTask | yes | RF picking work | target object/flow missing; generic task primitives exist |
| TU | yes | physical unit | shared TU exists; requires compatible Outbound extension |
| TUSetup | no | TU policy/config | target config missing/partial |
| Shipment | yes | logistics dispatch | public Sales shipment is only a partial candidate; target WMS lifecycle missing |
| Carrier | no | carrier master | carrier reference/provider models exist, need mapping |
| CarrierSetup | no | selection applicability | target applicability model missing |
| Region | no | delivery region | target mapping/config missing |
| CarrierManifest | yes | handover grouping/confirmation | target object missing |
| Inventory | yes | stock settlement | strong reusable Inventory foundation exists |
| PutBackTask | yes | physical recovery task | target object missing; Inbound Putaway is not equivalent |
| CrossDockPickTask | yes | RF crossdock work | target object missing; only demand/source primitives exist |
| Sequence | no | atomic numbering | PostgreSQL sequence patterns exist; target scopes need explicit implementation |
