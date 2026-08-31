# WMS Outbound v1 — Complete Implementation Plan

This plan implements the active Architect Source; it does not establish new business requirements. `01_ARCHITECT_SOURCE` and `02_CANON/AUTHORITY.md` remain authoritative.

## Readiness baseline

- Requirements: **109 = 98 FR + 6 INT + 5 CON**.
- Product/architecture blockers: **0** at ETAP 2.
- Current implementation contains reusable Inventory/TU/warehouse/lock/orchestration primitives plus incompatible legacy Outbound/SO shortcuts.
- Implementation must be additive/reversible around existing test data until cutover is proven.
- Human-facing acceptance requires real UI Playwright and final **Human Verified** walkthrough.

## Delivery invariants

1. Do not reinterpret architecture to match current code/DB.
2. Do not introduce PickWave or external carrier Label API in v1.
3. Preserve accepted Inbound behavior when shared Inventory/TU/Putaway/warehouse/lock/orchestration code is touched.
4. Server/DB remains authoritative for quantities, state, ownership, concurrency and idempotency.
5. Playwright must perform the actual accepted user action through Mercato/Scanner; API/DB may only prepare deterministic fixtures or verify persisted results.
6. Final user-facing acceptance is Human Verified.

## Work packages

### 0 — Foundations

- `FND-001` — **Establish target Outbound domain ownership and persistence boundaries** — depends on nothing.
- `FND-002` — **Build authoritative state/event transition and audit foundation** — depends on FND-001.
- `FND-003` — **Protect shared Inventory, TU, task-lock and warehouse compatibility** — depends on FND-001, FND-002.

### 1 — Standard demand, reservation, planning and picking

- `P1-001` — **CustomerOrder intake, validation, hold, warning and continuous aggregation** — depends on FND-001, FND-002.
- `P1-002` — **ATP soft reservation queue and recalculation** — depends on P1-001, FND-003.
- `P1-003` — **Cyclic planning, grouping and OutboundOrder/Line creation** — depends on P1-001, P1-002.
- `P1-004` — **Allocation hard reservation lifecycle** — depends on P1-003, FND-003.
- `P1-005` — **PickTask creation, zone ordering and operator assignment** — depends on P1-004, FND-003.
- `P1-006` — **RF picking and multi-zone Picking TU continuation** — depends on P1-005, P1-008.
- `P1-007` — **SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes** — depends on P1-004, P1-005, P1-006.
- `P1-008` — **Outbound TU identity, TUSetup, numbering, capacity and issueability** — depends on FND-003.

### 2 — Packing, logistics and settlement

- `P1-009` — **Direct Pack declaration and automatic sealing** — depends on P1-006, P1-008.
- `P1-010` — **Packing, repack, consolidation and discrepancy handling** — depends on P1-006, P1-008.
- `P1-011` — **Shipment grouping, closure and partial-shipment gates** — depends on P1-003, P1-009, P1-010.
- `P1-012` — **Carrier, Region and CarrierSetup selection** — depends on P1-011, P1-008.
- `P1-013` — **WMS label generation and pre-manifest loading carrier correction** — depends on P1-012.
- `P1-014` — **ERP Shipment POST, error state and safe retry** — depends on P1-011, P1-013, FND-002.
- `P1-015` — **CarrierManifest lifecycle and dispatch boundaries** — depends on P1-014.
- `P1-016` — **Final line/order/inventory settlement and cancellation boundaries** — depends on P1-015, P1-004, P1-001.

### 3 — Outbound Crossdock

- `P2-001` — **Inbound crossdock boundary and demand/source eligibility** — depends on FND-001, FND-003, P1-002, P1-003.
- `P2-002` — **Crossdock OutboundOrder/Line and CrossDockPickTask planning** — depends on P2-001, FND-002.
- `P2-003` — **RF crossdock sorting into outbound TUs** — depends on P2-002, P1-008.
- `P2-004` — **Crossdock shortage, damage, empty-TU and cancellation recovery** — depends on P2-003.
- `P2-005` — **Goods Receipt correlation gate and re-evaluation** — depends on P2-004, P1-014.
- `P2-006` — **Crossdock join into common Shipment/dispatch downstream** — depends on P2-005, P1-011, P1-012, P1-014, P1-015, P1-016.

### 4 — Cancellation recovery

- `P3-001` — **Reservation Release before formal pick** — depends on P1-004, P1-001, P1-016.
- `P3-002` — **Reservation retention policy and automatic release timer** — depends on P3-001.
- `P3-003` — **Cancellation race: physical movement before formal confirmation** — depends on P3-001, P4-001.
- `P4-001` — **Post-pick/post-pack cancellation approval and logical effects** — depends on P1-014, P1-015, P1-016.
- `P4-002` — **PutBackTask model, FIFO assignment and task lifecycle** — depends on P4-001, FND-003.
- `P4-003` — **RF PutBack location validation loop and Inventory recovery** — depends on P4-002.

### 5 — Cross-cutting correctness

- `X-001` — **Enforce CON-01..05 concurrency and exactly-once business effects** — depends on P1-004, P1-011, P1-014, P1-015, P2-002.
- `X-002` — **Integration correlation, observability and operational recovery** — depends on FND-002, P1-014, P2-005, P3-001, P4-001.

### 6 — Acceptance

- `ACC-001` — **Complete automated component/integration requirement suite** — depends on FND-001, FND-002, FND-003, P1-001, P1-002, P1-003, P1-004, P1-005, P1-006, P1-007, P1-008, P1-009, P1-010, P1-011, P1-012, P1-013, P1-014, P1-015, P1-016, P2-001, P2-002, P2-003, P2-004, P2-005, P2-006, P3-001, P3-002, P3-003, P4-001, P4-002, P4-003, X-001, X-002.
- `ACC-002` — **Playwright Standard Fulfillment + P1 exception journeys** — depends on ACC-001.
- `ACC-003` — **Playwright Crossdock, Reservation Release and Physical Putback journeys** — depends on ACC-001, P2-006, P3-003, P4-003.
- `ACC-004` — **Final Human Verified Outbound v1 walkthrough and acceptance record** — depends on ACC-002, ACC-003.

## Migration/cutover strategy

- Treat `wms_sales_order.so_header` and `wms_outbound.outbound_delivery` as legacy current-state evidence, not target domain truth.
- Introduce target entities additively; retain legacy data until mapping, dual-read/controlled adapter behavior and regression evidence prove safe cutover.
- Remove/disable the legacy `outbound_delivery.SHIPPED → so_header.SHIPPED` shortcut from active target behavior only in an implementation task with migration/backout evidence.
- Extend shared `wms_transport_units` without redefining accepted Inbound status semantics; prefer explicit Outbound role/lifecycle ownership.
- Reuse `wms_inventory` balances/ledger/ATP and orchestration idempotency primitives, but implement target `Allocation`, Shipment/Manifest and crossdock semantics explicitly.

## Evidence gates

A work package is not accepted on document completion. Component/integration tests prove technical rules; Playwright proves normal application traversal; Human Verified is the final acceptance layer for user-facing behavior. Evidence requirements are defined in `05_EVIDENCE/EVIDENCE_STANDARD.md`.

## Completion rule

Plan readiness requires all 109 requirements to map to one or more tasks and all architect scenario mappings to remain reachable. See `03_TRACEABILITY/IMPLEMENTATION_TRACEABILITY.md` and `requirement_task_matrix.csv`. Product implementation starts only after steering/owner authorization.
