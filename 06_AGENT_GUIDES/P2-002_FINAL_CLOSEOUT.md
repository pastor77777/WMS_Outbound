# P2-002 — final closeout after dedicated 22/22

## Authority / boundary

Owner authorized continuation of P2-002 only.

Use ONLY the current retained P2-002 Mercato and Scanner work produced by the completion remediation + harness corrective.

Hard database rule: PostgreSQL evidence/regressions MUST use the approved canonical Supabase Testing project `yzonugcenguvmojwiihb` through the existing `/etc/mercato-localhost.env` `DATABASE_URL`. Local PostgreSQL is forbidden and invalid evidence.

Do not start P2-003. Do not update STATE/handover/acceptance. Do not alter architect/canon files.

Frozen accepted inputs:
- accepted P2-001 Mercato base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- original P2-002 Mercato implementation checkpoint: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Scanner accepted baseline entering P2-002: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Dedicated P2-002 real-PG result already obtained in the immediately preceding corrective shot: **22 passed / 22 total**. The current retained working trees contain the tested harness/product corrections and Scanner module work.

## 1. Preflight and commit exact retained implementation

In Mercato and Scanner:
1. record branch, HEAD, `git status --short`, and full changed-file list;
2. verify there are no unrelated changes;
3. verify Mercato changes are confined to P2-002 implementation/migration/dedicated suite and required shared code only;
4. verify Scanner changes are confined to the bounded P2-002 crossdock module/assignment journey;
5. do not pull/rebase/squash/amend.

Commit the exact retained changes before the final test battery so every final command runs on immutable final SHAs.

Use normal descriptive commits and push the current P2-002 branches. Record exact Mercato and Scanner commit SHAs and merge bases.

If unrelated changes are present, STOP before committing.

## 2. Re-run dedicated P2-002 on final Mercato SHA

On canonical Supabase Testing only, run the exact dedicated suite:

`apps/mercato/src/modules/wms_outbound/services/__tests__/p2-002-crossdock-planning-postgres.integration.test.ts`

Run in-band with the repository-native Jest config/command pattern.

Required: **22/22 PASS, exit 0**.

If not 22/22, STOP. Do not continue to regressions/evidence.

## 3. Mandatory Mercato regressions on the same final SHA

Run only the exact mandatory regressions required by `06_AGENT_GUIDES/P2-002_EXECUTION.md`, using canonical Supabase Testing where PostgreSQL is involved:

1. accepted P2-001 PostgreSQL suite:
   `p2-001-crossdock-eligibility-postgres.integration.test.ts`
2. accepted P1-003 grouping/planning PostgreSQL suite(s) used by the accepted P1-003 evidence;
3. accepted P1-005 assignment/ordering suite because P2-002 relies on the warehouse ordering / active-task primitive;
4. FND-003 shared task-lock / warehouse compatibility regression(s) required by the actual touched primitives;
5. accepted Inbound cross-dock regression only if the final Mercato diff touches shared TU/Inbound boundary primitives; otherwise explicitly record `NOT REQUIRED — untouched`;
6. no unrelated dispatch/manifest/final-settlement suites.

Capture literal command, numeric exit code, suites/tests pass counts, and decisive failure text if any.

On the first non-zero/ambiguous result: STOP. Do not create canonical P2-002 evidence.

## 4. Scanner regressions + mandatory P2-002 Playwright

Because Scanner changed, run the relevant accepted Scanner auth/warehouse-context/task-assignment regressions required by the actual changed paths.

Then run a dedicated rendered Playwright P2-002 journey against canonical Testing runtime with the final Mercato + final Scanner SHAs. It must prove, with zero route mocks:

1. Warehouse Operator authenticates;
2. enters/selects **Crossdock** module;
3. no zone selector is required/shown for crossdock assignment;
4. receives exactly one next eligible CrossDockPickTask according to configured ordering;
5. rendered task identity, source Inbound TU and planned quantity are visible;
6. a second operator cannot receive the same task;
7. an operator with another active warehouse task is blocked;
8. stop before any P2-003 scan/sort/placement/quantity/quality/target-TU action.

Record exact command, runtime URLs, final repo SHAs, pass count and exit code.

Any failure/route mock/ambiguous runtime identity => STOP, no canonical evidence.

## 5. Final invariants to verify from code + real PG results

Before evidence, explicitly verify and record:
- no `Allocation` created by P2-002;
- positive ACTIVE P2-001 binding -> exactly one OOL -> exactly one CrossDockPickTask;
- `plannedQty` exactly equals accepted binding quantity;
- replay/concurrency cannot append a second task/line;
- CROSSDOCK channel immutable and never mixed with STANDARD;
- grouping requires exact customer/address/priority/slaDeadline and correct allowPartial semantics;
- exact source TU/binding/SKU/CustomerOrderLine correlation persists;
- physical target outbound TU remains lazy/not created in P2-002;
- assignment has no zone input, follows accepted ordering, enforces one active warehouse task, and prevents double assignment;
- CON-03 uses genuine overlapping independent PostgreSQL transactions on canonical Supabase Testing;
- org/tenant/warehouse isolation holds;
- no P2-003 scope leakage.

## 6. Canonical evidence

Only if every required final-SHA check above passes, create/update exactly:

`05_EVIDENCE/P2-002_EVIDENCE.md`

Evidence must include:
- accepted P2-001 SHA/evidence reference;
- exact final Mercato SHA and merge base;
- exact final Scanner SHA and merge base;
- exact changed-file scope for both repos;
- canonical Supabase Testing sanitized provenance (`yzonugcenguvmojwiihb`; never print credentials/full URL);
- migration/schema/uniqueness/idempotency/channel-immutability details;
- 22/22 dedicated P2-002 real-PG command/result;
- literal CON-03 evidence;
- exact mandatory regression commands and counts;
- exact Scanner regression commands/counts;
- exact Playwright command/runtime/result;
- explicit no-Allocation and lazy-target-TU proofs;
- clean final worktree identities.

Do NOT claim Owner acceptance or FINAL PASS. Use a neutral supervisor-verification-ready conclusion only.

Commit/push the WMS evidence file to `WMS_Outbound/main`.

## 7. Stop / report

STOP and report only:
- final Mercato SHA;
- final Scanner SHA;
- dedicated P2-002 count;
- mandatory regression counts;
- Playwright count;
- WMS evidence commit SHA;
- clean/dirty status of both product repos.
