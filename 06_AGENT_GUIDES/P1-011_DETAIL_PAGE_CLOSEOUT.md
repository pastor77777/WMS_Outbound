# P1-011 Detail Page Closeout

## Scope

Final narrow closeout for P1-011 after UI evidence reached 2/3 Playwright journeys.

Do not change steering, policy, architecture, STATE, handover, Scanner, credentials/auth handling, backend Shipment business rules, entities, migrations, or APIs.

Current Mercato head before this shot:
`1046a7b5b86664041be4c5bdfb65ddc3e5d78d1a`

Testing runtime:
`http://localhost:3009`

Use the existing canonical Testing environment on the VPS, including `/etc/mercato-localhost.env` where required. Do not print credential values.

## Independently verified root cause

The backend UI is served through the catch-all route:
`apps/mercato/src/app/(backend)/backend/[...slug]/page.tsx`

That router resolves the manifest route and renders the module page as:
`<Component params={match.params} />`

The P1-011 Shipment detail page:
`apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx`

currently ignores that `params` prop and instead calls `useParams()` and reads `params?.id`.

Under the backend catch-all, Next `useParams()` exposes the catch-all `slug`, not the module manifest's matched `id`. Therefore `shipmentId` is undefined. `loadData()` exits at `if (!shipmentId) return` before issuing `/api/wms_outbound/shipments/<id>`, leaving `loading=true` forever. This exactly matches the Playwright trace: canonical detail URL reached, `Loading shipment details...` remains, and no detail API request is emitted.

The detail API route itself is present and correct:
`apps/mercato/src/modules/wms_outbound/api/shipments/[id]/route.ts`

## Authorized product fix

Modify ONLY:
`apps/mercato/src/modules/wms_outbound/backend/shipments/[id]/page.tsx`

Use the backend manifest-provided route params prop instead of `useParams()`.

Required behavior:
- remove dependency on `useParams` for Shipment ID;
- accept the route params prop provided by the backend catch-all, e.g. a narrow typed prop containing `params?: { id?: string }`;
- derive `shipmentId` from that prop;
- preserve all existing detail-page behavior and API path;
- fail cleanly instead of remaining permanently in loading state if the ID is unexpectedly missing.

Do not change the catch-all router, route generator, API route, Shipment service, or business lifecycle.

Commit and push this one-file product UI fix to `outbound/p1-011`.

## Verification

Rebuild/restart only the canonical Testing Mercato runtime if required so port 3009 serves the new final head.

First rerun P1-011 Playwright against:
`PLAYWRIGHT_TEST_BASE_URL=http://localhost:3009`

Required result:
`3/3 passed`

Journey B must prove the normal rendered flow:
Shipment list -> Shipment-number link -> `/backend/shipments/<id>` -> detail API request -> visible `shipment-status-badge` with `READY_FOR_DISPATCH`.

Do not bypass the detail page with direct DB-only evidence.

If P1-011 Playwright is green, rerun the complete final-head gate:
- P1-011 PostgreSQL
- P1-010 PostgreSQL
- P1-009 PostgreSQL
- P1-003 PostgreSQL
- full `wms_outbound` backend umbrella
- P1-011 Playwright
- P1-010 Playwright

Previous successful baselines are reference only; record fresh final-head outputs:
- P1-011 PostgreSQL: 18/18
- P1-010 PostgreSQL: 16/16
- P1-009 PostgreSQL: 15/15
- P1-003 PostgreSQL: 14/14
- Outbound umbrella: 20 suites / 294 tests
- P1-011 Playwright target: 3/3
- P1-010 Playwright: 6/6

## Durable evidence

When all final-head gates pass, rebuild:
`WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md`

Use the NEW final 40-character Mercato SHA.

Evidence must include:
- accepted P1-010 base;
- final P1-011 Mercato SHA;
- frozen Scanner SHA;
- genuine Testing PostgreSQL identity;
- distinct-TU real lock/wait proof;
- rollback proof;
- literal fresh backend outputs;
- literal fresh P1-011 Playwright 3/3 output;
- literal fresh P1-010 Playwright 6/6 output;
- Testing Mercato runtime provenance on port 3009;
- Shipment list -> canonical detail page rendered proof;
- `PLAYWRIGHT VERIFIED` only;
- no credential values.

Push ONLY the rebuilt P1-011 evidence to `WMS_Outbound/main`.

Do not update STATE or handover.

Then STOP and report:
- final Mercato SHA;
- WMS evidence SHA;
- final test counts;
- remaining blocker only if one genuinely remains.
