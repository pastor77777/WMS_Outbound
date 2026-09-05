# P2-006 — UI fixture stabilization closeout

Continue the SAME P2-006 item from the preserved canonical VPS state.

## Frozen facts — do not disturb

Preserve all current P2-006 backend/product work and the already-proven green backend matrix. Do not deep-reset, rebuild, restart, rerun broad PostgreSQL suites, or modify business code unless the UI journey exposes a genuine product defect after fixture/auth plumbing is proven correct.

Already proven and retained:

- P2-006 dedicated PostgreSQL 20/20 PASS;
- P2-005 19/19;
- P1-011 18/18;
- P1-012 14/14;
- P1-013 15/15;
- P1-014 18/18;
- P1-015 21/21;
- P1-016 25/25;
- P2-002 22/22;
- P2-003 8/8;
- P2-004 16/16;
- durable typecheck PASS;
- durable fresh build PASS with fresh non-empty manifests;
- canonical Mercato runtime healthy on port 3009 and `/login` HTTP 200;
- Scanner frozen at accepted P2-005 SHA;
- zero route mocks/interception requirement remains absolute.

Current failing artifact is ONLY the new uncommitted P2-006 rendered UI spec.

## Why prior runs failed

The prior failures were fixture/test-plumbing failures before proving the business journey:

1. TLS/base-URL handling;
2. Supervisor DB identity lookup;
3. real GR ingress returned non-OK at Journey A before ERP/manifest reconciliation.

Do not report each fixture defect back to Owner as a separate stop. Own stabilization of this one spec inside this shot.

## Authoritative accepted fixture patterns

Use the accepted P2-005 spec as the primary source of truth for GR/auth fixture behavior:

`apps/mercato/src/modules/wms_outbound/__integration__/P2-005-crossdock-gr-gate-ui.spec.ts`

Key accepted behaviors to reuse literally unless P2-006 needs additional domain setup:

- Supervisor default identity `admin_dev@devaxonic.local`;
- password fallback chain already used by accepted UI specs;
- Supervisor DB lookup by `email = configured email OR established supervisor email_hash`;
- if the configured email does not resolve, use the actually resolved canonical user identity for the login attempt rather than continuing with a nonexistent configured email;
- assert the Supervisor row exists before beginning the journey; do not silently continue on an undefined row;
- derive `organization_id` and `tenant_id` from that resolved row, with only the already accepted canonical fallback values when truly needed;
- `page.request` must share the authenticated browser context/session created by the successful Supervisor login;
- the GR ingress path/body must match accepted P2-005 exactly:
  `/api/wms_outbound/cross-dock/gr-result`
  with `sourceInboundTU` as the source TU NUMBER/identifier, `settlementSource: 'CROSSDOCK'`, `status: 'GR_ACCEPTED'`, version, and idempotency key;
- the route requires auth plus `wms_outbound.edit`; therefore a successful visual/backend login alone is insufficient if the resolved actor lacks that feature.

## Stabilization procedure — one contained effort

Before launching any new Playwright process:

1. Inspect the current uncommitted P2-006 UI spec side-by-side with the accepted P2-005 GR-gate spec and accepted P1-015 manifest UI spec.
2. Fix all fixture/auth/baseURL differences that are not intentional P2-006 business differences.
3. Resolve the actual Supervisor row first and make the UI login use that resolved identity.
4. Verify by DB inspection that the resolved actor has the required outbound feature/role used by the accepted Testing admin path. Do not modify product authorization. If the accepted Testing admin fixture uses wildcard/super-admin semantics, reuse that fixture truth rather than inventing a new role.
5. Make login failure explicit: after login, assert that the page is actually in authenticated `/backend` state before any `page.request` integration call.
6. Make GR ingress failure diagnostic in the spec: if response is non-OK, capture status and response body in the thrown assertion/error so another blind rerun is never needed. This is test diagnostics only, not business behavior.
7. Verify fixture source TU exists in the same org/tenant scope and that the task source correlation uses the exact source TU NUMBER expected by the accepted P2-005 route contract.
8. Keep all fixture setup deterministic and all product actions that the guide requires as UI actions inside the real rendered Mercato UI.

## Run policy for this shot

After static/DB fixture stabilization, run the dedicated P2-006 durable Playwright spec.

You may diagnose and correct fixture-only failures within THIS SAME shot without reporting back after every one. A fixture-only correction means auth/login identity, deterministic test data, TLS/baseURL, selectors matching the already-rendered accepted UI contract, or test diagnostics — not business code.

At most TWO additional durable runs are authorized in this shot after the current failed run history. Before each rerun, inspect the prior result artifacts and make one evidence-based correction. Never launch duplicate concurrent units.

STOP immediately only if:

- a genuine P2-006 product/business defect is proven after fixture/auth plumbing is valid; or
- two evidence-based fixture correction reruns still fail; then report one consolidated blocker with exact status/body/first assertion and why it is no longer a trivial fixture mismatch.

## Success closeout

If the dedicated P2-006 Playwright becomes green:

1. verify zero route mocks/interception;
2. verify persisted reconciliation proves common Shipment/Carrier/label/ERP/CarrierManifest/final-settlement lifecycle and no fake Allocation or standard Inventory decrement for CROSSDOCK;
3. do NOT rerun the already-green full backend matrix or rebuild again unless the final fixture work somehow changed product code (it should not);
4. commit the minimum Mercato diff on `outbound/p2-006`;
5. push exact final Mercato SHA;
6. keep Scanner frozen unless genuinely changed;
7. write `05_EVIDENCE/P2-006_EVIDENCE.md` with exact SHAs, preserved backend counts, durable typecheck/build/runtime proof, UI run proof, zero mocks, clean worktrees, ancestry and 20/20 mapping;
8. evidence must not self-declare FINAL PASS / Owner Accepted / Human Verified;
9. STOP for supervisor verification. Do not start P3-001.
