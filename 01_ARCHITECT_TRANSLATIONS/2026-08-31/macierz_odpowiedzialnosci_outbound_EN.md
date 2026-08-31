# Outbound Responsibility Matrix

**Project:** WMSAI Outbound  
**Version:** 1.0 | **Date:** 2026-08-22 | **Author:** Business Analyst  
**Origin:** moved from archived `propozycja_procesow_outbound.md` v1.31 §8. The archived document is historical material, not a source of truth — this file is the source of truth (`DEC-A14`).  
**Scope:** cross-process responsibility view for P1–P5; it does not replace `proces_N`

---

## Responsibility matrix

“System WMS” is indicated only for automatic actions. An empty “Approves exception” column = no exception requiring escalation at this stage.

| Stage | Performs | Approves exception | Object | Business result |
|---|---|---|---|---|
| Order validation | System WMS | — | `CustomerOrder` | `ACCEPTED` / `REJECTED` / `ON_HOLD` |
| Cyclic planning | System WMS | — | `OutboundOrder` | `OutboundOrder` created |
| Allocation | System WMS | Supervisor only according to policy/exception | `Allocation` | `ATP` stock reserved or shortage handled automatically |
| `PickTask` creation | System WMS | — | `PickTask` | tasks for operators |
| Picking | Picker / System WMS (automatic reallocation) | Supervisor after no automatic option remains or the limit is reached | Picking `TU`, `OutboundOrderLine` | goods in Picking `TU` or a new `PickTask` |
| Assessment/packing | Packer | Supervisor (deviation from suggestion) | Outbound `TU` (Packing) | Packing `TU` ready |
| `Shipment` creation | Packer / System WMS | — | `Shipment` | `Shipment` with Packing `TU` |
| Carrier Selection | System WMS | Supervisor (no result) | `Shipment` | carrier determined |
| Manual carrier selection (external `TU`) | Dispatcher | Supervisor | `Shipment` | carrier approved |
| Labeling | System WMS | — | `Shipment` | label generated |
| Manifest and dispatch | Dispatcher | — | `CarrierManifest` | closing, handover |
| Reservation release (P3) | System WMS | Supervisor (according to policy) | `Allocation` | stock `AVAILABLE` |
| Physical put-back (P4) | Operator | Supervisor (after packing) | `PutBackTask`, `Inventory` | goods back to `AVAILABLE` |
| Outbound Crossdock | Packer / System WMS (automatic matching and classification) | Supervisor (shortage/`DAMAGED` with `allowPartialShipment = false` — `P2 R14`/`P2 R15`; visibility into `grAcceptanceStatus` when the ERP gate is held by `GR_REJECTED` — `P2 R36`) | `CrossDockPickTask`, `OutboundOrderLine`, Outbound `TU` | Outbound `TU` in `PACKING_SEALED`, ready for `Shipment` |

## Change history

- **No version change (2026-08-22)** — batch 3/5 closing the V3-CD audit: metric-header label “Source” reworded to “Origin” (`DEC-A14`); “Approves exception” column completed in the Outbound Crossdock row — Supervisor for shortage/`DAMAGED` with `allowPartialShipment = false` (`P2 R14`/`P2 R15`) and visibility into `grAcceptanceStatus` when the ERP gate is held by `GR_REJECTED` (`P2 R36`, option C1 from batch 1).
- **1.0 (2026-08-22)** — Cross-process P1–P5 responsibility matrix moved from `propozycja_procesow_outbound.md` §8 while implementing B16.
