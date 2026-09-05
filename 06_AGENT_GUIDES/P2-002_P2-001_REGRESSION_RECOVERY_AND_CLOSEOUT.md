# P2-002 — P2-001 regression recovery and final closeout

## Purpose

Recover the single remaining P2-002 closeout blocker: accepted P2-001 PostgreSQL regression hangs as a whole with exit 124/no Jest output, while P2-002 dedicated 22/22, raw Supabase connectivity, native-Jest MikroORM init, and a full first P2-001 test path have already passed on canonical Testing Supabase.

This shot MUST diagnose P2-001 at test-case granularity, make at most a test-harness-only correction if a specific P2-001 test is proven to hang for harness synchronization reasons, then continue the already-defined P2-002 final closeout only if P2-001 becomes green.

## Frozen implementation identities entering this shot

- Mercato P2-002: `92b50ca0eeeb41adc71800f1da036e73562552c6`
- Scanner P2-002: `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`
- Accepted P2-001 base/reference: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Canonical Testing Supabase project reference in approved env: `yzonugcenguvmojwiihb`
- Dedicated P2-002 suite already passed 22/22 on canonical Testing.

Do not change P2-002 product behavior unless a new product regression is literally proven. Do not use local PostgreSQL under any circumstances.

## Hard database rule

All PostgreSQL work in this shot MUST use the approved environment from `/etc/mercato-localhost.env`, whose sanitized DATABASE_URL provenance must resolve to the Supabase Testing pooler/project ref `yzonugcenguvmojwiihb`.

Never print credentials/full DATABASE_URL. Fail immediately if host/project provenance is not canonical Testing Supabase.

## 1. Preflight

1. Sync repositories without rebasing/squashing accepted history.
2. Mercato must be exactly `92b50ca0eeeb41adc71800f1da036e73562552c6` and clean before any temporary diagnostic edit.
3. Scanner must be exactly `8ff264e2e03d75abd422f69a7ff1f713f10d32fc` and clean.
4. Confirm no stale Jest/Yarn/Node test process from earlier shots remains.
5. Load approved Testing env and record only sanitized host/port/project-ref presence.

## 2. Isolate all ten accepted P2-001 tests individually

Target file only:

`apps/mercato/src/modules/wms_outbound/services/__tests__/p2-001-crossdock-eligibility-postgres.integration.test.ts`

Run each existing test by exact `--testNamePattern` one at a time with `--runInBand`, using an outer `timeout 45s`, canonical Testing env, and literal numeric exit status. Do not edit the test before this isolation pass.

The ten existing cases are:

1. `TC-020/102: accepts only ELEMENTARY at IN_CROSS_DOCK and persists ASN declaration/correlation`
2. `section 11 #3: rejects an ELEMENTARY source before IN_CROSS_DOCK`
3. `section 11 #5/#13: ASN declaration is the source basis and min(source,demand) is exact decimal arithmetic`
4. `TC-029/036: ATP and non-cancelled execution coverage leave no double-covered demand`
5. `section 11 #8/#9: CROSSDOCK coverage deducts demand while CANCELLED coverage does not`
6. `section 11 #10/#11/#12: active, consumed and damaged source facts all remain deducted; RELEASED alone returns capacity`
7. `TC-092/103: zero match creates a durable residual handoff fact and replay is idempotent`
8. `section 11 #15: demand disappearing after Inbound qualification produces the deterministic zero-match fact`
9. `section 11 #19: organization and tenant scope cannot bind a foreign source or demand`
10. `CON-03/TC-134: independent PostgreSQL sessions expose real source-row lock waiting and serialize competing bindings`

For each case record: exact name, PASS/FAIL/TIMEOUT, exit status, Jest summary if available, elapsed time.

### Decision gate A

- If tests 1–9 are not all PASS: STOP. Do not change product code. Record which exact case failed.
- If test 10 also PASS: proceed directly to section 4.
- If tests 1–9 PASS and only test 10 times out/fails: proceed to section 3.

## 3. CON-03 harness-only recovery, only if isolated test 10 is the sole blocker

The accepted semantic requirement MUST remain unchanged: two independent PostgreSQL transactions/sessions; actor A holds the authoritative source-row/serialization lock; actor B demonstrably waits on actor A; `pg_stat_activity`/`pg_blocking_pids` or equivalent decisive PostgreSQL-side evidence identifies the actual participants; after release exactly one positive binding consumes capacity and no overbooking occurs.

You may modify ONLY the P2-001 PostgreSQL test harness and ONLY to make synchronization deterministic/observable on the canonical Supabase pooler. Allowed examples:

- replace fixed sleeps with bounded polling for both backend PIDs and the actual lock wait;
- use a deferred/latch with explicit timeout/rejection so a failed callback cannot leave actor A held forever;
- use an independent observer connection/EntityManager for `pg_stat_activity` / `pg_blocking_pids`;
- ensure `releaseFirst()` executes in `finally` on assertion/observer failure;
- add bounded diagnostic markers inside this one test.

Forbidden:

- weakening/removing the independent-session requirement;
- accepting simulated/mock concurrency;
- deleting the lock-wait assertion;
- changing P2-001 product/service semantics;
- switching DB venue;
- using local PostgreSQL.

After any harness-only correction:

1. run isolated CON-03 with timeout 60s;
2. it must PASS with literal participant PID + PostgreSQL wait/block evidence;
3. run the entire P2-001 file with timeout 150s;
4. it must report `10 passed, 10 total`, exit 0.

Commit/push the harness correction on the existing Mercato P2-002 branch only after both isolated CON-03 and full P2-001 are green. Record the new Mercato SHA. If the isolated CON-03 cannot be made decisively green by one narrow harness correction, STOP without product changes.

## 4. Full accepted P2-001 regression

If no correction was needed, run the full P2-001 file now with canonical Testing env and `timeout 150s`.

Required result: `10 passed, 10 total`, exit 0.

If it still times out despite all ten individual cases passing, STOP and record that exact contradiction; do not invent PASS.

## 5. Resume P2-002 final closeout from the first uncompleted regression

Once full P2-001 is green, continue the mandatory P2-002 closeout from `06_AGENT_GUIDES/P2-002_FINAL_CLOSEOUT.md` using the final Mercato SHA (either `92b50ca...` unchanged or the harness-only successor) and Scanner `8ff264e...`.

Run and record exact suite names/counts for all remaining mandatory regressions required by the authoritative P2-002 execution guide, including:

- accepted P1-003 grouping/planning regressions;
- accepted P1-005 task assignment/ordering regressions because the shared assignment semantics are used;
- FND-003 shared task-lock/TU/warehouse compatibility regressions for touched/shared primitives;
- accepted Inbound cross-dock/shared-boundary regressions where required by touched shared entities;
- relevant Scanner P1 auth/warehouse/task-assignment regressions because Scanner changed.

Fail fast at the first genuine regression failure. No canonical evidence on red.

## 6. Scanner Playwright

Because Scanner changed, run the required rendered P2-002 crossdock assignment journey on canonical Testing runtime/revisions:

- operator enters Crossdock module;
- no zone selection;
- next eligible CrossDockPickTask is shown with task/source/planned quantity;
- a second operator cannot receive the same task;
- an operator already holding another active warehouse task is blocked;
- STOP before all P2-003 sorting/scanning/placement behavior.

Use zero route mocks for the decisive journey. Record literal Playwright count/result and exact Scanner/Mercato SHAs.

## 7. Final evidence and pushes

Only after all required backend regressions and Scanner Playwright are green:

1. ensure Mercato and Scanner implementation/test changes are committed and pushed;
2. verify clean worktrees and exact final SHAs/merge bases;
3. create/update ONLY canonical:
   `05_EVIDENCE/P2-002_EVIDENCE.md`
4. include all fields required by `P2-002_EXECUTION.md`: 22/22 dedicated PG result, no Allocation, exact binding→OOL→CrossDockPickTask mapping, exact plannedQty, immutable CROSSDOCK channel, grouping/allowPartial rules, lazy physical target TU, assignment semantics, literal CON-03 evidence, exact regressions, exact Playwright result, final repository identities, canonical Testing Supabase provenance.
5. Do NOT claim Owner acceptance or FINAL PASS in the evidence.
6. Push WMS evidence and report its commit SHA.

Then STOP.

## Final response format

Return only concise facts:

- final Mercato SHA;
- final Scanner SHA;
- P2-002 dedicated result;
- P2-001 regression result;
- remaining regression aggregate/counts;
- Scanner Playwright result;
- WMS evidence commit SHA;
- PASS/STOP classification.
