# P2-003 — Playwright Fixture Closeout

## Purpose

Close only the remaining P2-003 acceptance blocker: the missing real Scanner Playwright execution fixture for Journey A (1:1) and Journey B (n:n + physical-full), with persisted reconciliation.

Do not reopen runtime root-cause work unless a current service check proves it is actually broken again.

## Preserve current local candidates

The latest executor reported clean local candidate commits:

- Mercato `0f4ab9d0605fae7596c5c603a92af9f12d79a2e1`
- Scanner `a70c744b927fba620768e72740128cab8ae57b65`

These commits are not yet on GitHub. Preserve them exactly; do not reset, rebase-away, reconstruct from P2-002, or discard local work.

Frozen accepted bases remain:

- Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`

Before editing, prove each local candidate descends from its frozen accepted base and record current branch/status.

## Existing evidence/runtime to retain unless invalidated

- Supervisor host build probe already produced a real Mercato `BUILD_OK` with non-empty `.mercato/next/routes-manifest.json` and `.mercato/next/required-server-files.json`.
- P2-003 dedicated PostgreSQL: 8/8 PASS, but final evidence still requires explicit mapping of all 18 substantive behaviors.
- P2-002 PostgreSQL regression: 22/22 PASS.
- Existing P2-003 assignment/UI-only browser coverage is not sufficient for acceptance.

Do not rerun unrelated gates merely by habit. Rerun only what this fixture/test change or a discovered product fix invalidates.

## Accepted test patterns to reuse

Use the existing Scanner tests as patterns, not as acceptance substitutes:

- `e2e/p2-002-crossdock-assignment.spec.ts` — deterministic real PostgreSQL fixture, user/ACL/warehouse setup, task assignment, zero route mocks, persisted reconciliation.
- `e2e/p1-009-real-scanner-direct-pack.spec.ts` — larger deterministic real PostgreSQL setup/cleanup and full rendered Scanner journey with post-action DB reconciliation.

Do not copy obsolete UI text blindly. Use current testIDs/accessibility/current copy from the exact local P2-003 Scanner candidate.

## Required new fixture/spec

Create a dedicated current P2-003 real execution spec, preferably:

`e2e/p2-003-real-crossdock-execution.spec.ts`

It may use DB/API only for deterministic setup and post-action verification. The decisive business execution must occur through the real rendered Scanner UI.

Hard rule: zero Playwright route mocks / request interception that substitutes product behavior.

The fixture must explicitly create/restore its own relevant org/tenant/warehouse/operator/ACL prerequisites and all P2-003 business records it needs. Do not rely on ambient Testing cardinality/defaults.

Use canonical Testing PostgreSQL only via the existing Testing env contract. No local PostgreSQL.

## Journey A — real 1:1 execution

Build one deterministic 1:1 crossdock scenario whose source TU has a valid GS1 source SSCC and whose quantity/order structure allows the P2-003 inheritance rule to apply.

Through the real Scanner UI, perform the full intended operator flow from task acquisition/start through real SKU/quantity placement and task completion.

The journey must prove, through rendered UI actions plus independent persisted reconciliation as applicable:

1. task starts from assignment into real execution;
2. source/order-line/order execution states move according to P2-003 rules;
3. correct SKU and OK quantity are accepted;
4. target TU remains lazy until first placement;
5. first placement creates/activates the target TU;
6. valid 1:1 GS1 source identity is inherited according to P2-003/P1-008 rules, including TU_NUMBER + SSCC semantics;
7. placement persistence is real;
8. task completion is real and `confirmedQty <= plannedQty`;
9. final persisted task/placement/TU/order-line/order state reconciles to the UI actions;
10. no Allocation-based crossdock behavior is introduced.

Do not pull P2-004 shortage/damage/empty/cancel behavior into this journey.

## Journey B — real n:n + physical-full continuation

Build one deterministic n:n scenario that requires creation of new target identity via accepted P1-008 TUSetup/Sequence behavior and forces a target TU to become physically full before the crossdock task is complete.

Through the real Scanner UI, prove:

1. n:n does not inherit source TU identity;
2. first target identity is generated through the accepted TUSetup/Sequence path;
3. real placements persist against the first target TU;
4. operator can mark/seal the physically full target TU through the intended rendered flow;
5. sealing the target does not complete/cancel the crossdock task;
6. the same task continues into a new/open target TU;
7. the next target identity is valid/new according to the accepted identity rules;
8. remaining quantity is placed through the real UI;
9. task completes only after the planned quantity is satisfied, with `confirmedQty <= plannedQty`;
10. persisted placements, target TUs, seal/open states, quantities and task/order states reconcile exactly.

This journey is still P2-003 only. No shortage, damage, unexpected SKU, empty/cancellation, GR, Shipment/ERP/manifest, P3/P4 or Return Receipt.

## Runtime contract before Playwright

Before running either journey:

- verify canonical `mercato-localhost.service` is active/running;
- verify its MainPID owns/listens on port 3009;
- verify `/login` returns HTTP 200;
- verify canonical `scanner-testing.service` is serving the freshly exported exact Scanner candidate on port 8081;
- do not launch an ad-hoc Playwright webServer that masks canonical runtime identity.

If runtime is not healthy, collect the exact current service/port/journal error and fix only that concrete defect. Do not repeat the old blind build-recovery loop.

## Product-fix boundary

The expected work is fixture/test coverage. If a real Journey A/B exposes a deterministic P2-003 product defect, fix only the minimum P2-003 product code necessary, rebuild/restart the affected canonical runtime, and rerun the invalidated gates.

Do not broaden into P2-004+ or unrelated refactors.

## 18-behavior mapping

Before final evidence, inspect the dedicated P2-003 PostgreSQL suite and map all 18 required substantive behaviors explicitly.

Eight test cases are acceptable only if their assertions genuinely cover all 18 behaviors. If any behavior is not substantively covered, add the minimum missing dedicated test assertion/case and rerun the dedicated suite.

Do not claim mapping complete from headings or prose alone; tie each behavior to a concrete test/assertion/evidence result.

## Final closeout

Only after Journey A and Journey B are green and persisted reconciliation is green:

1. rerun any directly invalidated dedicated/regression gate;
2. ensure final Mercato and Scanner worktrees are clean;
3. put the final Mercato work on/push `outbound/p2-003`;
4. put the final Scanner work on/push `outbound/p2-003`;
5. verify pushed heads equal the exact tested finals;
6. create/update `05_EVIDENCE/P2-003_EVIDENCE.md` with exact final product SHAs, runtime provenance, dedicated-suite count, explicit 18-behavior mapping, Journey A/B results, zero-route-mock statement and persisted reconciliation;
7. push WMS evidence and report its SHA.

Evidence must not self-declare `FINAL PASS`, `Owner Accepted` or `HUMAN VERIFIED`.

## Final report

Report only:

- final pushed Mercato SHA;
- final pushed Scanner SHA;
- P2-003 dedicated result/count and 18/18 behavior mapping status;
- P2-002 regression result if rerun/retained and why;
- Mercato canonical runtime proof;
- Scanner canonical runtime proof;
- Journey A result;
- Journey B result;
- persisted reconciliation summary;
- WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty;
- if STOPped: one exact remaining blocker.

Then STOP.
