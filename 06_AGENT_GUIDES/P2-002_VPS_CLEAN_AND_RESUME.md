# P2-002 — VPS Testing cleanup and regression resume

**Owner authorization:** explicit.  
**Venue:** owner-controlled VPS shell/tmux only.  
**Mercato revision is frozen:** `24d102a41dcfff669cf6b839b369fbaa3620f87d`.  
**Scanner remains frozen:** `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## Objective

Clean only orphan test processes / PostgreSQL sessions left by the failed command-execution runtime, then resume the exact P2-002 mandatory regressions on the frozen Mercato SHA. No product or test changes are authorized.

## 1. Preflight

On the VPS:

- work from `/home/ubuntu/git/Devaxonic-mercato`;
- verify `HEAD`/branch is exactly `outbound/p2-002` at `24d102a41dcfff669cf6b839b369fbaa3620f87d`;
- do not pull/rebase/amend/change product or test files;
- verify the Testing DB target is the same approved Testing PostgreSQL used by the accepted suites.

## 2. Clean only proven orphan test activity

Audit running processes for Jest/Yarn/Node test processes from the prior P2 regression attempts.

Terminate only processes demonstrably belonging to those stale test runs. Do not kill unrelated owner tmux sessions, agents, application services, PostgreSQL itself, or unrelated jobs.

Then inspect PostgreSQL activity/locks for the Testing database and identify sessions/transactions blocking the P2 suites. Terminate only sessions demonstrably orphaned from the stale test runs. Do not broadly terminate all Testing connections.

Record before/after process and PostgreSQL blocking-session snapshots in the durable diagnostic output described below.

After cleanup, prove there is no remaining stale test process and no stale blocking transaction from the prior attempts.

## 3. Resume regressions on the frozen SHA

Run the exact mandatory regression commands required by `06_AGENT_GUIDES/P2-002_EXECUTION.md`, in owner-controlled VPS shell/tmux, with durable stdout/stderr and explicit exit status captured for every invocation.

Order:

1. accepted P2-001 PostgreSQL suite;
2. accepted P1-003 grouping/planning suite(s);
3. accepted P1-005 task-assignment/ordering suite(s) only if required by the actual P2-002 diff/touched primitive;
4. FND-003 shared task-lock/TU/warehouse compatibility suite(s) only for touched primitives;
5. accepted Inbound cross-dock result regression only if required by the actual diff;
6. Scanner regressions only if Scanner changed — otherwise record Scanner frozen and do not run invented Scanner work.

If any suite returns non-zero, STOP immediately and record the exact command, exit status and decisive failure text. Do not classify an infrastructure failure as a product failure without evidence.

If all required regressions pass, continue the normal P2-002 evidence completion from `P2-002_EXECUTION.md` using the already-pushed Mercato SHA. Do not modify business code/tests just to improve evidence.

## 4. Git-visible raw result

Create/update:

`05_EVIDENCE/P2-002_VPS_REGRESSION_RUN.md`

This file is a diagnostic/raw-run record, not acceptance by itself. Include:

- exact Mercato SHA and Scanner frozen SHA;
- pre-clean orphan process snapshot;
- pre-clean PostgreSQL blocking-session/lock snapshot;
- exactly what stale process/session was terminated and why it was identified as stale;
- post-clean proof;
- every regression command actually run;
- literal exit status for each;
- concise pass counts or decisive failure excerpt;
- statement that no product/test files changed during cleanup/regression venue work.

Push this WMS file so Supervisor can read the result through Git.

If all mandatory regressions pass and the main execution guide's remaining evidence requirements are satisfied, also create/update the canonical `05_EVIDENCE/P2-002_EVIDENCE.md` and push it. Otherwise do not create a PASS-claiming canonical evidence file.

## 5. STOP

Report exact WMS commit SHA containing the Git-visible run record, regression results, and whether canonical P2-002 evidence was created. Then STOP.
