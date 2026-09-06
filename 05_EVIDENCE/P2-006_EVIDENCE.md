# P2-006 — Crossdock join into common Shipment/dispatch downstream

Date: 2026-09-06 UTC  
Evidence class: REAL POSTGRESQL INTEGRATION and PLAYWRIGHT VERIFIED. This is not Human Verified, FINAL PASS, or Owner Accepted.

## Revisions and scope

- Mercato branch: `outbound/p2-006`
- Mercato candidate: `2c50a186ade68ec2e73ec479654a8ec989c88eb4`
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

Durable final unit: `p2-006-playwright-20260906T000808Z.service` — exit status 0, **2 tests passed** (`results.json` under Mercato `.ai/qa/test-results/`). The P2-006 spec contains no `page.route` or `.route(` usage.

1. Journey A: a real CROSSDOCK contribution used the rendered common Shipment UI for carrier selection, WMS label generation/print, visible P2-005 GR gate and blocked posting; a real authenticated external GR ingress accepted the source; rendered ERP posting then succeeded; the normal rendered CarrierManifest open/add/close/handover/confirm lifecycle completed. Persisted reconciliation proved exactly one settlement, line/order/customer terminal aggregates, zero Allocation for the crossdock line, and zero standard Inventory movement.
2. Journey B/C: a mixed STANDARD+CROSSDOCK `allowPartialShipment=false` CustomerOrder remained outside Shipment while the crossdock line was incomplete; the rendered normal group-waiting-TUs action joined both sealed TUs into one common Shipment only after equality was satisfied; the rendered Shipment detail exposed the crossdock GR gate; replaying the normal grouping action preserved the one-Shipment membership invariant.

The real GR ingress used the repository's established `getAuthToken` + `apiRequest` helper for the external producer boundary. Human-facing Shipment and CarrierManifest decisions remained rendered browser actions.

## P2-006 behavior mapping

- Common model/grouping/no fork, grouping key, P2 R43 inheritance, one-TU membership, partial/no-partial and SLA guards: dedicated 20/20 PostgreSQL matrix; rendered mixed Journey B/C.
- Common readiness, carrier, label, ERP and P2-005 gate: retained P1/P2 regressions plus rendered Journey A.
- Shared CarrierManifest lifecycle and irreversible UI actions: retained P1-015 plus rendered Journey A.
- Exactly-once final settlement, multi-shipment/replay semantics, standard Allocation/Inventory preservation and Allocation-free/no-standard-inventory CROSSDOCK settlement: dedicated 20/20 matrix, retained P1-016, and rendered Journey A reconciliation.

Supervisor verification is still required before any acceptance state changes.
