# P2-002 — Scanner fixture recovery and closeout

Continue P2-002 only in the SAME Codex session.

Frozen green state:
- Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- Scanner product `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`
- CON-03 1/1 PASS
- P2-001 10/10 PASS
- P2-002 dedicated PostgreSQL 22/22 PASS
- remaining Mercato regressions 5 suites / 41 tests PASS

Do not rerun green Mercato gates. Do not change Mercato. Do not start P2-003. Canonical Supabase Testing only (`yzonugcenguvmojwiihb`); local PostgreSQL is forbidden.

## Root cause

`e2e/p1-005-real-scanner-assignment.spec.ts` is a historical P1-005 acceptance fixture. It creates a temporary operator but relies on automatic warehouse resolution. Current warehouse service auto-creates a default assignment only when the tenant has exactly one warehouse; canonical Testing now has multiple warehouses for that tenant. Therefore this fixture is environment-dependent and its failure is not by itself a P2-002 regression.

P2-002 Scanner product diff from `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` to `8ff264e2e03d75abd422f69a7ff1f713f10d32fc` is additive only: Crossdock route/screen/labels/API helper/mode tile. Existing picking/auth/warehouse behavior was not changed.

## A. Test-only fixture correction

Change only `e2e/p1-005-real-scanner-assignment.spec.ts` so its temporary operator has an explicit deterministic assignment to the intended historical warehouse `9b174240-2376-4a5b-8d5a-5f3271e347d7`. If current zone authorization requires an explicit assignment to Ambient Storage `af1103d4-7057-40ba-a1b8-b91d539a37b0`, add only that fixture prerequisite. Cleanup those temporary assignment rows after the test.

Do not change Scanner application/product files, package files, dependencies, or assertions. No route mocks.

Run exactly this one Playwright spec once.

Required: `p1-005-real-scanner-assignment.spec.ts` = 1/1 PASS, exit 0.

If it still fails because actual rendered P1-005 behavior is wrong, STOP and report the exact product-visible failure. No second blind fixture retry.

If PASS, commit/push the test-only correction and record the Scanner SHA.

## B. Dedicated P2-002 Playwright

Create/update the minimum dedicated Playwright spec, test/evidence code only, and run it against final Mercato + Scanner product code on canonical Testing.

Prove through the rendered Scanner with real backend and no route mocks:
1. Warehouse Operator login;
2. deterministic active warehouse context;
3. enter Crossdock module;
4. no zone selector;
5. exactly one eligible CrossDockPickTask assigned;
6. task identity visible;
7. source Inbound TU visible;
8. plannedQty visible and equal to persisted task data;
9. second independent operator cannot receive the same task;
10. operator with another active warehouse task is blocked;
11. stop before all P2-003 actions.

If PASS, commit/push any dedicated test-only Scanner change and record final Scanner SHA. Any product-code change requirement => STOP.

## C. Evidence

Only if A and B both PASS, create/update exactly `05_EVIDENCE/P2-002_EVIDENCE.md` and record:
- final Mercato SHA;
- final Scanner SHA and product-vs-test-only diff distinction;
- canonical Testing provenance;
- CON-03 1/1;
- P2-001 10/10;
- P2-002 PostgreSQL 22/22;
- remaining Mercato 5 suites / 41 tests;
- P1-005 Scanner 1/1 after deterministic warehouse fixture correction;
- dedicated P2-002 Playwright command/count/result;
- no-zone, one-task, source/plannedQty, double-assignment and active-task-block proofs;
- no Allocation, lazy target TU, no P2-003 leakage;
- clean worktrees.

Do not claim Owner Acceptance, FINAL PASS, Human Verified, or update STATE/handover/progress.

Commit/push only canonical P2-002 evidence, then STOP.

Report only final Mercato SHA, final Scanner SHA, P1-005 Scanner result, P2-002 Playwright result, WMS evidence SHA, and clean/dirty states.
