# P2-001 — Inbound crossdock boundary and demand/source eligibility

**Execution evidence only.** This document makes no FINAL PASS, Supervisor acceptance, or Owner acceptance claim.

## Revision and scope

- Accepted base: Mercato `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574` (P1-016).
- Final Mercato revision: `a262b135617c6f84d125e6539c59ae1586ca4ae3` on `outbound/p2-001`.
- Compare scope: additive P2-001 entity, migration, eligibility/binding service, and dedicated PostgreSQL integration suite only.
- Scanner frozen and not modified: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## Implemented boundary

`WmsOutboundCrossDockBinding` is a new Outbound-owned pre-planning seam. It does not reuse or reinterpret Inbound's `wms_cross_dock_demands`, which remains the bounded qualification read model.

- requires source `WmsTransportUnit.tuType = ELEMENTARY` and `processStatus = IN_CROSS_DOCK`;
- retains source Inbound TU, SKU/item, ASN-declared quantity, receipt correlation, organization/tenant/warehouse, optional eligible CustomerOrderLine, binding amount, state and idempotency identity;
- records `ZERO_MATCH` with zero bound quantity when no current BACKORDERED demand is available. It creates no CrossDockPickTask, OutboundOrder or OutboundOrderLine; the complete ASN declaration remains available to the existing Inbound residual handoff;
- uses integer micro-units (`bigint`) for exact decimal arithmetic;
- calculates source availability as ASN declared quantity minus active persisted bindings. This is the P2-001 authoritative planned-reservation seam; later P2 task completed/confirmed/damaged facts are deliberately not created by this item;
- calculates line demand as ordered quantity minus active warehouse ATP reservations, non-CANCELLED OutboundOrderLine required quantity (all channels), and active pre-planning bindings;
- uses `min(sourceRemaining, demandRemaining)`, clamps unavailable demand to zero, and orders candidates by the configured warehouse queue rule (priority/SLA or SLA/priority);
- serializes source and competing demand through transaction-scoped PostgreSQL advisory keys plus `PESSIMISTIC_WRITE` locks on the actual source TU and BACKORDERED demand rows.

Schema migration: `Migration20260905130000_wms_outbound_p2_001` creates `wms_outbound_cross_dock_bindings`, source/demand indexes, and a tenant-scoped idempotency unique index. It is additive; its down migration drops only this table. The migration was applied to approved Testing PostgreSQL successfully.

## Real PostgreSQL evidence

Command (with approved Testing environment injected in the same shell):

```bash
yarn workspace @open-mercato/app test --runInBand \
  apps/mercato/src/modules/wms_outbound/services/__tests__/p2-001-crossdock-eligibility-postgres.integration.test.ts
```

Result: **4/4 passed** against real PostgreSQL.

The suite proves ELEMENTARY/IN_CROSS_DOCK acceptance and AGGREGATE rejection; persisted INT-01 source/SKU/ASN declaration/receipt correlation; ATP plus non-CANCELLED STANDARD OOL demand deductions; zero-match plus idempotent replay; and a `Promise.all` race through independent ORM forks. The race finishes with exactly one active 5.000000 binding from a 5.000000 source, assigned to the higher-priority CustomerOrderLine. The implementation uses PostgreSQL transaction locks named above, not an in-memory guard.

## Regression execution

The following focused command completed successfully on the final source state:

```bash
yarn workspace @open-mercato/app typecheck --pretty false
yarn workspace @open-mercato/app test --runInBand \
  apps/mercato/src/modules/wms_outbound/services/__tests__/p1-002-postgres.integration.test.ts \
  apps/mercato/src/modules/wms_outbound/services/__tests__/p1-003-postgres.integration.test.ts \
  apps/mercato/src/modules/wms_outbound/services/__tests__/fnd-003-shared-compatibility.test.ts
```

Focused suites: P1-002 (13 declared cases), P1-003 (14 declared cases), FND-003 shared compatibility (8 declared cases).

The accepted Inbound cross-dock result integration suite was also run. It currently reports **1/3 passed, 2 failed** because its existing expected `Cross-Dock-to-sector TransportTask` is `null` while its `PutawayTask` is created. No P2-001 code or Inbound code was changed to address this unrelated fixture/runtime assertion; it is recorded here rather than represented as a passing regression.

## Explicit exclusions

No P2-002 planning objects, CrossDockPickTask, Scanner/RF work, GR gate, Inbound qualification/deintegration semantics, P3/P4, Return Receipt, external carrier behavior, Prod, or Demo changes were made.
