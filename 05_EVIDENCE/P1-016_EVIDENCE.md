# P1-016 — final settlement evidence

## Scope and identities

- Accepted P1-015 base: `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`.
- Final Mercato commit: `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574` on `outbound/p1-016`.
- Merge base: `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`.
- The closeout diff from preserved runtime checkpoint `e423f6e5d6749b73776807a5d832a936f760085e` is test-only: one dedicated 78-line Playwright spec, `apps/mercato/src/modules/wms_outbound/__integration__/P1-016-final-settlement-ui.spec.ts`.
- Frozen Scanner identity: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## PostgreSQL settlement and concurrency proof

On the final Mercato SHA, canonical Testing PostgreSQL ran:

```text
yarn workspace @open-mercato/app test --runInBand \
  src/modules/wms_outbound/services/__tests__/p1-016-final-settlement-postgres.integration.test.ts

Test Suites: 1 passed, 1 total
Tests:       25 passed, 25 total
```

The same final-SHA run recorded genuine PostgreSQL lock evidence:

```text
Race A: pidA=2083059, pidB=2083061, waitEventType=Lock, blockers=[2083059]
Race B: pidA=2083061, pidB=2083059, waitEventType=Lock, blockers=[2083061]
```

Race A proves distinct backend sessions with the replay session blocked on the manifest lock. Race B proves the two-manifest shared line/allocation lock, including the no-lost-update assertions. Test 23 in this suite is the rollback proof: a fresh independent read after an intentional post-flush failure finds no settlement or inventory movement, unchanged allocation quantity, a `PACKED` line and `HANDED_OVER` manifest.

## Mandatory final-SHA regressions

The following final-SHA command completed against canonical Testing PostgreSQL:

```text
yarn workspace @open-mercato/app test --runInBand \
  p1-015-manifest-lifecycle-postgres.integration.test.ts \
  p1-014-erp-posting-postgres.integration.test.ts \
  p1-013-label-generation-postgres.integration.test.ts \
  p1-012-carrier-selection-postgres.integration.test.ts \
  p1-011-postgres.integration.test.ts \
  fnd-002-state-transitions.test.ts \
  fnd-002-transaction-simulation.test.ts \
  fnd-003-postgres.integration.test.ts \
  fnd-003-shared-compatibility.test.ts

Test Suites: 9 passed, 9 total
Tests:       187 passed, 187 total
```

Included required counts are P1-015 21/21, P1-014 18/18, P1-013 15/15, P1-012 14/14, P1-011 18/18, FND-002 state transitions 77/77, FND-002 transaction simulation 8/8, plus the accepted shared Allocation/Inventory/FND-003 regression suites in the same 187/187 aggregate.

## Canonical Testing runtime

The final SHA was production-built before the runtime check. The production artifact includes `apps/mercato/.mercato/next/routes-manifest.json`. Canonical `mercato-localhost.service` was restarted through systemd and was `active`; the process listened on `:3009`.

```text
Mercato HEAD: dd5ff1493740ffc99e11ce40e0b5ffc6b646f574
localhost:3009/login HTTP 200
https://devaxonic-test.info-start.com.pl/login HTTP 200
```

No Demo/port-3000 runtime was used.

## Dedicated rendered-browser evidence

Final run, against `http://127.0.0.1:3009` with the project-native Playwright config:

```text
npx playwright test \
  apps/mercato/src/modules/wms_outbound/__integration__/P1-016-final-settlement-ui.spec.ts \
  --config .ai/qa/tests/playwright.config.ts --workers=1 --retries=0

6 passed (3.0m)
```

The dedicated spec is separate from P1-015 and covers six rendered Supervisor journeys:

- A — final confirmation of one manifest, visibly `CONFIRMED`, then persisted settlement, negative inventory movement, allocation consumption, shipped line, completed order and closed customer order.
- B — rendered confirmation of the first split manifest, with persisted partial quantities and nonterminal line/order/customer states.
- C — rendered confirmation of the later split manifest, with exact final quantities and terminal aggregation.
- D — rendered cancellation eligibility attempt at a dispatched/shipped hard boundary, visibly blocked with no cancellation side effect.
- E — rendered whole-order cancellation evaluation with a blocking line, visibly atomically rejected and both lines unchanged.
- F — repeated rendered final confirmation with stable `CONFIRMED` UI and exactly one settlement and inventory movement.

The run used one worker, zero retries, and had 6/6 passed; therefore it recorded 0 unexpected and 0 flaky results. This is `PLAYWRIGHT VERIFIED` rendered-browser evidence, not human-verification evidence.

## Exclusions

No P2/P3/P4, Returns, external carrier API, Scanner product, Demo/Prod, steering/control files, or P1-015 evidence relabeling is included.
