# P2-006 — Crossdock join into common Shipment/dispatch downstream

Date: 2026-09-06 UTC  
Evidence class: REAL POSTGRESQL INTEGRATION and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Revisions and scope

- Mercato branch: `outbound/p2-006`
- Mercato candidate: `4f64641ab14a5359bc22d0685e390b511252b5b5`
- Accepted P2-005 ancestor: `069f02d4c5c9b345b688b838eb685be02206afbd`
- Scanner frozen and clean: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- No schema migration or separate CROSSDOCK Shipment, carrier, label, ERP, CarrierManifest, or settlement model was added.

The product change is limited to common `manifest-service.ts`: CROSSDOCK contributions are settled through the normal manifest confirmation lifecycle while omitting a fabricated Allocation and standard Inventory decrement. Standard contributions retain the accepted Allocation/Inventory path.

## Retained backend and runtime proof

The candidate product change was covered on canonical Testing PostgreSQL before UI stabilization:

- dedicated P2-006 matrix: 20/20 PASS, mapping all required substantive behaviors;
- P2-005 19/19; P1-011 18/18; P1-012 14/14; P1-013 15/15; P1-014 18/18; P1-015 21/21; P1-016 25/25; P2-002 22/22; P2-003 8/8; P2-004 16/16 PASS;
- durable typecheck: PASS (`p2-006-typecheck-20260905T223209Z.service`);
- durable fresh build: PASS with non-empty `routes-manifest.json` and `required-server-files.json`;
- canonical `mercato-localhost.service`: canonical Mercato process bound port 3009 and `/login` returned HTTP 200.

No product source changed after these typecheck/build/runtime proofs; subsequent work was the dedicated UI fixture only.

## Zero-mock rendered UI proof

Durable final unit: `p2-006-playwright-20260906T002556Z.service` — `Result=success`, `ExecMainStatus=0`, **2 tests passed** in 3.6 minutes (`results.json` under Mercato `.ai/qa/test-results/`). The P2-006 spec contains no `page.route` or `.route(` usage.

1. Journey A: a CROSSDOCK Packing TU began `PACKING_SEALED`, unassigned, with persisted source-TU, completed CrossDockPickTask and placement lineage. The rendered common group-waiting-TUs action attached it to the common Shipment. On that same Shipment, rendered carrier selection, WMS label generation/print, visible P2-005 GR blocking and zero blocked-ERP postings preceded real authenticated GR acceptance; rendered ERP and normal CarrierManifest open/add/close/handover/confirm then completed. Persisted reconciliation proved exactly one settlement, terminal Shipment/TU/line/order/customer aggregates, zero Allocation and zero standard Inventory movement for the crossdock contribution.
2. Journey B/C: a compatible mixed STANDARD+CROSSDOCK `allowPartialShipment=false` CustomerOrder kept both sealed TUs outside Shipment while the crossdock line was incomplete. Expiring the identical SLA still could not bypass that rendered grouping guard. After P1 R58 equality completed, the rendered normal grouping action attached both TUs to one common Shipment. Its visible P2-005 gate listed exactly the CROSSDOCK source, not STANDARD content; real GR acceptance unblocked that same Shipment, whose rendered carrier, label, ERP and normal CarrierManifest lifecycle completed. Persisted reconciliation proved exactly-once STANDARD Allocation consumption and one standard Inventory movement, with zero CROSSDOCK Allocation and standard Inventory movement; both channel aggregates closed. A rendered grouping replay preserved the single-Shipment membership invariant.

The real GR ingress used the repository's established `getAuthToken` + `apiRequest` helper for the external producer boundary. Human-facing Shipment and CarrierManifest decisions remained rendered browser actions.

## P2-006 behavior mapping

- Common model/grouping/no fork, grouping key, P2 R43 inheritance, one-TU membership, partial/no-partial and SLA guards: dedicated 20/20 PostgreSQL matrix; continuous rendered Journey A and Journey B/C.
- Common readiness, carrier, label, ERP and P2-005 gate: retained P1/P2 regressions plus both continuous rendered journeys.
- Shared CarrierManifest lifecycle and irreversible UI actions: retained P1-015 plus both continuous rendered journeys.
- Exactly-once final settlement, multi-shipment/replay semantics, standard Allocation/Inventory preservation and Allocation-free/no-standard-inventory CROSSDOCK settlement: dedicated 20/20 matrix, retained P1-016, and both continuous rendered reconciliations.

Supervisor verification is still required before any acceptance state changes.
