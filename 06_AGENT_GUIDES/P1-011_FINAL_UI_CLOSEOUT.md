# P1-011 Final UI Closeout

## Scope

This is a narrow final closeout for P1-011. Do not change steering, policy, architecture, STATE, handover, Scanner, authentication handling, backend Shipment business rules, entities, migrations, or APIs.

Current Mercato head before this shot:
`73898b053b85e0372f23f7823e997c7d04ebb053`

Testing runtime:
`http://localhost:3009`

Use the existing Testing environment on the VPS, including `/etc/mercato-localhost.env` where required. Do not print credential values.

## Authorized changes

Only these files may change unless a directly proven same-path route/timing defect requires another line in the same P1-011 Playwright spec:

1. `apps/mercato/src/modules/wms_outbound/backend/shipments/page.tsx`
2. `apps/mercato/src/modules/wms_outbound/__integration__/P1-011-shipment-grouping-ui.spec.ts`

### 1. Fix Shipment detail navigation

The canonical routes are:
- Shipment list: `/backend/shipments`
- Shipment detail: `/backend/shipments/<shipmentId>`

In `backend/shipments/page.tsx`, correct both stale detail navigations still using `/backend/wms_outbound/shipments/<id>`:
- Shipment-number link
- RowActions `View Details`

Do not change application routing elsewhere.

### 2. Make P1-011 Playwright lifecycle synchronization deterministic

Use realistic real-UI test timeout consistent with the accepted P1-010 harness, e.g. `test.setTimeout(90_000)` per journey. Do not use arbitrary sleeps as the synchronization mechanism.

#### Journey A

Preserve proof after packing TU1:
- blocked banner visible;
- reason proves `allowPartialShipment=false` guard;
- TU1 remains `PACKING_SEALED`;
- TU1 `shipment_id IS NULL`.

Pack TU2.

Wait until rendered `shipment-assigned-info` contains TU2's exact `tuNumber`; visibility alone is insufficient.

Then navigate through rendered UI to `/backend/shipments` and click `[data-testid="group-waiting-tus-btn"]` to perform the normal waiting-TU grouping action. Do not replace this decisive action with a direct API call.

Use deterministic polling/fresh DB reads to prove:
- TU1 = `IN_SHIPMENT`;
- TU2 = `IN_SHIPMENT`;
- both `shipment_id` values are non-null;
- both TUs share exactly the same Shipment.

#### Journey B

After packing TU1, require `shipment-assigned-info` to contain TU1's exact `tuNumber`.

After packing TU2, require the same rendered element to update and contain TU2's exact `tuNumber`. Do not accept stale visibility from TU1.

Then prove by fresh DB reads:
- both Shipment IDs are non-null;
- both are identical;
- Shipment reaches `READY_FOR_DISPATCH`.

From the rendered Shipment list, locate/search the actual Shipment and click its Shipment-number link. Do not bypass the production navigation with direct `page.goto()` to the detail URL. Prove the corrected canonical link reaches the detail page and rendered status is `READY_FOR_DISPATCH`.

#### Journey C

After packing TU1, wait until `shipment-assigned-info` contains TU1's exact `tuNumber`.

After packing late TU2, wait until it contains TU2's exact `tuNumber`.

Then preserve the existing persisted proof that TU2 belongs to a second Shipment and the first closed Shipment did not regress.

## Commit and verification

Commit and push the narrow Mercato changes to `outbound/p1-011`.

First run P1-011 Playwright on the canonical Testing runtime with:
`PLAYWRIGHT_TEST_BASE_URL=http://localhost:3009`

Required: `3/3 passed`.

Only after that passes, rerun the complete final-head gate against the new Mercato SHA:
- P1-011 PostgreSQL
- P1-010 PostgreSQL
- P1-009 PostgreSQL
- P1-003 PostgreSQL
- full `wms_outbound` backend umbrella
- P1-011 Playwright
- P1-010 Playwright

Previous baselines are reference only; record fresh final-head output:
- P1-011 PostgreSQL: 18/18
- P1-010 PostgreSQL: 16/16
- P1-009 PostgreSQL: 15/15
- P1-003 PostgreSQL: 14/14
- Outbound umbrella: 20 suites / 294 tests
- P1-010 Playwright: 6/6
- P1-011 Playwright target: 3/3

Do not fabricate or reuse counts if fresh output differs.

## Durable evidence

When all final-head gates are green, rebuild:
`WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md`

Include:
- final 40-character Mercato SHA;
- accepted P1-010 base;
- frozen Scanner SHA;
- real Testing PostgreSQL identity;
- distinct-TU real PostgreSQL lock/wait proof;
- rollback proof;
- literal fresh backend outputs;
- literal P1-011 Playwright output;
- literal P1-010 Playwright output;
- Testing Mercato runtime provenance on port 3009;
- canonical Shipment UI route proof;
- `PLAYWRIGHT VERIFIED` only;
- no credential values.

Push only the rebuilt P1-011 evidence to `WMS_Outbound/main`.

Do not update STATE or handover.

Then STOP and report only:
- final Mercato SHA;
- WMS evidence SHA;
- final test counts;
- remaining blocker, if any.
