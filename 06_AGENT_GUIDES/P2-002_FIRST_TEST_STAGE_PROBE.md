# P2-002 — first P2-001 test stage probe

## Purpose

Find the exact stage where the accepted P2-001 regression stalls on canonical Supabase Testing.

Diagnostic-only. Do not change product code, canonical tests, config, dependencies, Scanner, state, handover, or canonical P2-002 evidence.

## Frozen refs

- Mercato: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Accepted P2-001: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Scanner: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

## Database boundary

Use only canonical Supabase Testing project ref `yzonugcenguvmojwiihb`.
Load `/etc/mercato-localhost.env` without printing or modifying `DATABASE_URL`.
Print only sanitized scheme/host/port plus boolean `usernameHasProjectRef`.
If host is local or project ref mismatches, STOP.

## Preconditions

In `/home/ubuntu/git/Devaxonic-mercato`:
- HEAD exactly `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- clean working tree
- Scanner remains frozen

If not, STOP.

## One permitted temporary probe

Create one temporary untracked Jest spec:
`apps/mercato/src/modules/wms_outbound/services/__tests__/p2-002-first-test-stage-probe.test.ts`

Use normal `apps/mercato/jest.config.cjs`.
Mirror only the FIRST test path from accepted:
`p2-001-crossdock-eligibility-postgres.integration.test.ts`

Use the exact P2-001 imports/entity set and exact `MikroORM.init` options.
Use fresh random IDs.

Run only:
1. MikroORM.init
2. create `WmsTransportUnit` ELEMENTARY / IN_CROSS_DOCK and `WmsTuExpectedContent` qty `5.000000`; flush
3. create `WmsOutboundCustomerOrder` BACKORDERED; flush
4. create `WmsOutboundCustomerOrderLine` qty `5.000000`; flush
5. call `createCrossDockEligibilityService(orm.em.fork()).evaluateAndBind(...)` with the same semantics as the first P2-001 test
6. assert `boundQty === '5.000000'`
7. clean only probe rows
8. close ORM

Mandatory UTC markers immediately before/after each stage:
- `IMPORTS_COMPLETE`
- `BEFORE_MIKROORM_INIT`
- `AFTER_MIKROORM_INIT`
- `BEFORE_SOURCE_FLUSH`
- `AFTER_SOURCE_FLUSH`
- `BEFORE_ORDER_FLUSH`
- `AFTER_ORDER_FLUSH`
- `BEFORE_LINE_FLUSH`
- `AFTER_LINE_FLUSH`
- `BEFORE_EVALUATE_AND_BIND`
- `AFTER_EVALUATE_AND_BIND`
- `BEFORE_CLEANUP`
- `AFTER_CLEANUP`
- `AFTER_ORM_CLOSE`

Also capture numeric exit status.
Bound the single Jest command with one 60-second outer timeout, `--runInBand`, zero retries.

If it hangs, the last emitted marker is the classification boundary. Do not retry.

Afterward delete the temporary probe and verify Mercato working tree is clean.

## Record

Write only:
`05_EVIDENCE/P2-002_FIRST_TEST_STAGE_PROBE.md`

Include:
- frozen refs
- sanitized Supabase provenance
- exact marker sequence with timestamps
- literal Jest result or timeout exit
- classification as exactly one of:
  - `SOURCE_FLUSH`
  - `ORDER_FLUSH`
  - `LINE_FLUSH`
  - `EVALUATE_AND_BIND`
  - `CLEANUP`
  - `FIRST_TEST_PASS`
- confirmation temporary probe removed and Mercato/Scanner unchanged

Commit/push only this diagnostic record to WMS_Outbound/main, then STOP.

Do not run the full P2-001 regression and do not create canonical P2-002 evidence.