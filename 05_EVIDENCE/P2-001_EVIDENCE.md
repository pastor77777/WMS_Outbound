# P2-001 — Inbound crossdock boundary and demand/source eligibility

**Execution evidence only.** This document makes no FINAL PASS, Supervisor acceptance, or Owner acceptance claim.

## Revision and scope

- Accepted base: Mercato `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574` (P1-016).
- Final Mercato revision: `8a264fff5c2ca665294d1e02df90c6f37554fe7f` on `outbound/p2-001`.
- Compare scope: additive P2-001 entity, migration, eligibility/binding service, dedicated PostgreSQL integration suite, and a minimal accepted-Inbound test-harness prerequisite/restore only.
- Scanner frozen and not modified: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## Implemented boundary

`WmsOutboundCrossDockBinding` is a new Outbound-owned pre-planning seam. It does not reuse or reinterpret Inbound's `wms_cross_dock_demands`, which remains the bounded qualification read model.

- requires source `WmsTransportUnit.tuType = ELEMENTARY` and `processStatus = IN_CROSS_DOCK`;
- retains source Inbound TU, SKU/item, ASN-declared quantity, receipt correlation, organization/tenant/warehouse, optional eligible CustomerOrderLine, binding amount, state and idempotency identity;
- records `ZERO_MATCH` with zero bound quantity when no current BACKORDERED demand is available. It creates no CrossDockPickTask, OutboundOrder or OutboundOrderLine; the complete ASN declaration remains available to the existing Inbound residual handoff;
- uses integer micro-units (`bigint`) for exact decimal arithmetic;
- calculates source availability as ASN declared quantity minus `ACTIVE`, `CONSUMED`, and `DAMAGED` persisted bindings. `RELEASED` is the sole unexecuted plan status that returns capacity. P2-002 will own task creation and the transition of its linked binding to the terminal source-use statuses; P2-001 creates no task;
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

Result: **10/10 passed** against real PostgreSQL.

The substantive scenario mapping is:

| Guide §11 scenarios | PostgreSQL proof |
| --- | --- |
| 1–5 | ELEMENTARY/`IN_CROSS_DOCK` acceptance; AGGREGATE and wrong-status rejection; INT-01 source/SKU/receipt persistence; ASN declaration as the exact 2.125000 source basis. |
| 6–9 | ATP, STANDARD OOL, CROSSDOCK OOL and CANCELLED OOL coverage arithmetic. |
| 10–13 | `ACTIVE` planned, `CONSUMED` confirmed and `DAMAGED` completed source facts remain deducted; `RELEASED` alone returns capacity; exact `min(source,demand)`. |
| 14–16 | zero-match/full residual, demand disappearance before `IN_CROSS_DOCK`, warehouse priority winner. |
| 17–19 | independent-session source race, idempotent replay, and foreign organization/tenant isolation. |
| 20 | accepted P1 ATP/planning regression command below. |

### CON-03 literal PostgreSQL serialization evidence

The race starts two independent MikroORM forks. The first service transaction holds the actual source `WmsTransportUnit` with `PESSIMISTIC_WRITE`; its PostgreSQL backend PID has a granted relation lock. The second service transaction reports its own backend PID before lock acquisition; `pg_stat_activity.wait_event_type = 'Lock'` is asserted for that PID while the first transaction is intentionally held. After release, the two calls complete with exactly one active `5.000000` binding against the `5.000000` ASN source, assigned to the higher-priority line; the other call yields `0.000000`. This captures the PostgreSQL-side lock mechanism and actual participants, not just a final in-memory outcome.

## Regression execution

The following focused command completed successfully on the final source state:

```bash
yarn workspace @open-mercato/app typecheck --pretty false
yarn workspace @open-mercato/app test --runInBand \
  apps/mercato/src/modules/wms_outbound/services/__tests__/p1-002-postgres.integration.test.ts \
  apps/mercato/src/modules/wms_outbound/services/__tests__/p1-003-postgres.integration.test.ts \
  apps/mercato/src/modules/wms_outbound/services/__tests__/fnd-003-shared-compatibility.test.ts
```

Result: **4 suites / 38 passed**: P1-002 PostgreSQL 13/13; P1-003 PostgreSQL 14/14; FND-003 shared compatibility 8/8; accepted Inbound cross-dock result integration 3/3.

### Inbound base-versus-final disposition

The exact accepted P1-016 base `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574` was reproduced in a detached worktree under the same Testing environment: the suite was **1/3 passed, 2 failed**. Both failures had `locationDeterminationMode = MANUAL` and therefore correctly returned no automatic Cross-Dock-to-sector `TransportTask`, while the test asserted the automatic journey. The P2-001 diff did not modify the Inbound service or this test before remediation.

The minimal harness correction explicitly switches the existing tenant location-determination setting to `AUTOMATIC` for this automatic-journey suite and restores/deletes it in `afterAll`. It does not alter Inbound qualification, deintegration, transport, or application business semantics. On final P2-001 it is **3/3 passed**.

## Explicit exclusions

No P2-002 planning objects, CrossDockPickTask, Scanner/RF work, GR gate, Inbound qualification/deintegration/transport behavior, P3/P4, Return Receipt, external carrier behavior, Prod, or Demo changes were made. Final Mercato branch and WMS evidence worktrees were clean after their commits; no secrets were recorded.
