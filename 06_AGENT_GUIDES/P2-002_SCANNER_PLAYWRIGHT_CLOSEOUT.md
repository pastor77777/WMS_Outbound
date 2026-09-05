# P2-002 — Scanner regression correction + Playwright closeout

## Authority / boundary

Continue P2-002 only. This shot exists solely because the prior closeout stopped on an invalid Scanner regression-runner assumption.

Do not start P2-003. Do not modify Architect Source, Canon, Task Catalog, STATE, handovers, or acceptance state.

Hard database rule: any DB setup/assertion must use canonical Supabase Testing project `yzonugcenguvmojwiihb`. Local PostgreSQL is forbidden and invalid evidence.

Frozen green product/test state entering this shot:
- final Mercato: `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- current Scanner: `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`
- corrected CON-03: 1/1 PASS
- full P2-001 PostgreSQL regression: 10/10 PASS
- P2-002 dedicated PostgreSQL: 22/22 PASS
- remaining mandatory Mercato regressions: 5 suites / 41 tests PASS

Do not rerun those green Mercato gates unless this shot changes Mercato (it must not).

## 1. Correct the Scanner test-runner assumption

Read exact Scanner `package.json`, `package-lock.json`, `playwright.config.*` if present, and `e2e/` contents at SHA `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`.

Required conclusion to preserve unless repository facts contradict it:
- Scanner has no separate Jest/unit regression runner configured in `package.json`;
- Scanner's native automated acceptance runner is Playwright via `npm run test:e2e` / repository-equivalent command;
- `@playwright/test` is already declared in `devDependencies` and represented in `package-lock.json`;
- therefore the prior STOP caused by an attempted absent regression package is a harness/command-selection error, not a product dependency gap.

Do NOT add Jest, Vitest, Testing Library, or any other test dependency merely to manufacture a separate Scanner regression command.
Do NOT modify package.json/package-lock.json unless the repository facts above are false.

Record this classification for final evidence as:
`SCANNER_SEPARATE_UNIT_RUNNER_NOT_APPLICABLE — repository-native Scanner regression/acceptance runner is Playwright`.

## 2. Preflight

Verify:
- Mercato exact HEAD `50b27fdd0c9b495ab612ce458bc90e65428ecb93`, clean;
- Scanner exact HEAD `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`, clean;
- WMS_Outbound synced to main;
- canonical Testing runtime identity is known and no Demo/Prod runtime is used.

If either product repo has unrelated dirty changes, STOP.

## 3. Scanner regression through repository-native Playwright

Inspect existing Scanner `e2e/` specs and select the smallest existing accepted regression set that actually covers the shared surfaces touched by P2-002 Scanner changes:
- authentication/session entry;
- warehouse-context selection/visibility;
- mode selection/navigation;
- existing Outbound/picking task entry or assignment behavior if such an accepted spec exists.

Run those existing specs using the repository-native Playwright command. Do not invent a nonexistent unit-test package.

Requirements:
- normal rendered Scanner application;
- no route mocks for decisive assignment behavior;
- exact Scanner revision recorded;
- literal spec names, test count, exit code recorded.

If an existing selected regression fails due to a real behavioral regression, STOP.
If no pre-existing spec exists for one of the listed shared surfaces, record `NO PRE-EXISTING SPEC` for that surface; do not fail merely because a nonexistent historical spec cannot be run. The mandatory dedicated P2-002 journey below must still prove the changed behavior.

## 4. Dedicated P2-002 Playwright journey

Create the minimum dedicated Playwright spec if none already exists. This is test/evidence code only; do not change Scanner product behavior.

The decisive journey must use the normal rendered Scanner application against canonical Testing and prove:

1. Warehouse Operator authenticates through the normal Scanner flow;
2. correct warehouse context is selected/resolved;
3. operator enters/selects the `Crossdock` module;
4. no zone selector is shown or required;
5. operator receives exactly one next eligible `CrossDockPickTask` according to configured warehouse ordering;
6. rendered task identity/task number is visible;
7. rendered source Inbound TU is visible;
8. rendered `plannedQty` is visible and matches persisted task data;
9. second independent operator/session cannot receive that same task;
10. an operator/session that already owns another active warehouse task is blocked from receiving a crossdock task;
11. no sorting/source scan/SKU scan/quantity-quality entry/target-TU placement or completion controls are exercised — stop at P2-002 boundary.

### Fixture/setup rules

Direct API/DB fixture setup is permitted only to create deterministic prerequisites and verify persisted facts. It must not replace the rendered user actions above.

Use unique org/tenant/warehouse/order/binding/task/operator identifiers so this run cannot collide with old Testing data.

No route mocking of:
- authentication result used by the decisive flow;
- `/api/wms_outbound/cross-dock/request-task`;
- active warehouse-task guard;
- task assignment result.

The backend must be final Mercato SHA `50b27fdd0c9b495ab612ce458bc90e65428ecb93`.

If current canonical Testing runtime is not actually serving that exact Mercato SHA and exact Scanner source/build under test, update/restart only the owner-controlled Testing runtime as already permitted by project execution conventions; never use Prod/Demo. Record runtime identity.

## 5. Scanner test-only commit policy

If a new/changed dedicated Playwright spec was required:
- verify Scanner diff contains test/e2e code only;
- commit/push that test-only change on the current P2-002 Scanner branch;
- record the resulting final Scanner SHA.

Do not change application product files in this shot.

If no test file change was required, final Scanner remains `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`.

Any application/product-code change requirement => STOP and report; not authorized in this corrective.

## 6. Final invariant/evidence review

Do not rerun green Mercato suites. Reuse the immediately preceding verified results on Mercato `50b27fdd...` because this shot must not change Mercato.

Before canonical evidence, confirm:
- CON-03 corrected proof 1/1 PASS;
- P2-001 10/10 PASS;
- P2-002 dedicated 22/22 PASS;
- remaining Mercato regressions 5 suites / 41 tests PASS;
- Scanner native regression Playwright result is PASS, with `NO PRE-EXISTING SPEC` allowed only where truly absent;
- dedicated P2-002 Playwright is PASS;
- Mercato clean at exact `50b27fdd...`;
- Scanner clean at its final SHA;
- no P2-003 scope leakage.

## 7. Canonical P2-002 evidence

Only if the dedicated P2-002 Playwright journey passes and there is no real Scanner regression failure, create/update exactly:

`WMS_Outbound/05_EVIDENCE/P2-002_EVIDENCE.md`

Record:
- accepted P2-001 base/evidence reference;
- final Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
- final Scanner SHA;
- canonical Supabase Testing sanitized provenance `yzonugcenguvmojwiihb`;
- corrected CON-03 1/1 proof and architecture-aligned semantics;
- P2-001 10/10;
- P2-002 dedicated 22/22;
- remaining Mercato regressions 5 suites / 41 tests;
- Scanner regression classification `SCANNER_SEPARATE_UNIT_RUNNER_NOT_APPLICABLE` plus actual repository-native Playwright regression specs/counts;
- package evidence that `@playwright/test` is already in Scanner package.json and package-lock.json;
- dedicated P2-002 Playwright spec/command/count/runtime and rendered assertions;
- final Mercato/Scanner SHAs and clean worktrees;
- no Allocation, lazy target-TU, one-binding→one-OOL→one-task, exact plannedQty, assignment/no-zone/active-task/double-assignment invariants;
- explicit P2-003+ exclusions.

Do NOT claim Owner Acceptance, Human Verified, or FINAL PASS.

Commit/push the canonical evidence file to WMS_Outbound/main.

## 8. STOP/report

STOP and report only:
- final Mercato SHA;
- final Scanner SHA;
- Scanner native regression spec count/result;
- dedicated P2-002 Playwright count/result;
- WMS evidence commit SHA;
- Mercato clean/dirty;
- Scanner clean/dirty.
