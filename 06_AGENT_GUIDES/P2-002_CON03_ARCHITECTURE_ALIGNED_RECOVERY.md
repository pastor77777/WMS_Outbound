# P2-002 — architecture-aligned CON-03 recovery and closeout

## Authority / owner authorization

Owner explicitly authorized continuation after review of `architecture-context` and `wms-outbound` authority.

This is a NEW Codex session shot. Execute P2-002 only.

Do not start P2-003. Do not modify Architect Source, Canon, Task Catalog, STATE, handovers, or acceptance state.

Hard DB rule: all PostgreSQL evidence/regressions MUST use approved canonical Supabase Testing project `yzonugcenguvmojwiihb` through the existing Testing `DATABASE_URL`. Local PostgreSQL is forbidden and invalid evidence.

Current product revisions entering this shot:
- Mercato P2-002: `92b50ca0eeeb41adc71800f1da036e73562552c6`
- Scanner P2-002: `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`
- accepted P2-001 base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- accepted P2-001 evidence: `941d614966b1d2197d8654b3af924afb6ab14d58`

Known verified state:
- P2-002 dedicated PostgreSQL suite: 22/22 PASS on the current implementation;
- P2-001 cases 1–9 pass individually;
- old P2-001 CON-03 harness is the blocker;
- product code is not currently implicated by that failure.

## 1. Mandatory grounding before changing anything

Read only the exact relevant sources:

1. `CLAUDE-SKILLS/wms-outbound/SKILL.md` authority rules if available in the executor context;
2. `WMS_Outbound/06_AGENT_GUIDES/P2-002_EXECUTION.md` §10 CON-03 and §12–§14;
3. `WMS_Outbound/05_EVIDENCE/EVIDENCE_STANDARD.md` integration/persistence evidence rules;
4. Architect Outbound `proces_2_outbound_crossdock.md` R6, R29, R30;
5. Architect Outbound `wymagania_outbound.md` CON-03;
6. Architect scenario TC-134;
7. current Mercato test:
   `apps/mercato/src/modules/wms_outbound/services/__tests__/p2-001-crossdock-eligibility-postgres.integration.test.ts`.

Authority conclusion to preserve:

- Outbound CON-03 requires the same source quantity to be assigned/planned at most once under concurrency.
- R29 makes `CrossDockPickTask.plannedQty` the quantity lock.
- R30 requires one-shot generation.
- `TC-134` is NOT the concurrency scenario; it proves hard `OutboundOrderLine.requiredQty` remains in `demandEligibleQty` after ATP transfer.
- `pg_blocking_pids()`, `pg_stat_activity.wait_event_type = 'Lock'`, or any particular PostgreSQL observer query is NOT an Architect requirement.
- Those observer assertions were historical execution-harness choices and must not become an acceptance requirement.

## 2. Authorized change — test harness only

Change ONLY the P2-001 PostgreSQL regression test harness unless a compile-only import adjustment is strictly necessary.

Do NOT change:
- `cross-dock-eligibility-service.ts`;
- P2-002 planner product code;
- migrations/schema;
- Scanner;
- business behavior.

Make these exact semantic corrections:

### A. Correct scenario naming/mapping

The existing demand coverage test that proves ATP + non-CANCELLED OOL coverage must explicitly include `TC-134` in its test name/mapping.

The final concurrency test must be named/mapped only to `CON-03` (not `TC-134`).

### B. Remove non-authoritative observer gate

Remove the pass/fail dependency on:
- `pg_blocking_pids()`;
- `pg_stat_activity.wait_event_type = 'Lock'`;
- relation-lock visibility from an observer connection;
- fixed sleeps whose only purpose is to catch an ephemeral PostgreSQL wait-state snapshot.

Do not replace it with another brittle PostgreSQL catalog snapshot requirement.

### C. Keep genuine PostgreSQL concurrency proof

The CON-03 test MUST still use the real service and genuine independent overlapping PostgreSQL transactions on canonical Testing.

Use the existing service instrumentation hooks only as synchronization/evidence seams:

1. transaction A calls the real `evaluateAndBind` through its own `orm.em.fork()`;
2. capture A's real PostgreSQL backend PID from the existing in-transaction `SELECT pg_backend_pid()` hook;
3. hold A only after `onSerializationAcquired` fires — at that point A has passed the service's PostgreSQL advisory serialization and pessimistic source-row lock acquisition;
4. transaction B calls the same real service through a separate `orm.em.fork()` with a competing idempotency key;
5. capture B's distinct real PostgreSQL backend PID from `onTransactionStarted`;
6. synchronize on explicit Promises/hooks, not arbitrary sleep;
7. while A is intentionally held and B has definitely started, assert B has NOT settled/completed yet;
8. assert both PIDs are > 0 and different;
9. release A;
10. await both real service calls;
11. assert final persisted source usage is exactly one ACTIVE binding totaling the 5.000000 source capacity and attached to the higher-priority line;
12. assert the two results are exactly one `5.000000` and one `0.000000` outcome;
13. assert no double source/demand coverage is persisted.

This is the decisive CON-03 evidence: two distinct PostgreSQL transaction participants overlap against the real authoritative boundary, the competing call cannot complete while the first serialization owner is held, and the persisted/final result is exactly-once after release.

The test must have a bounded timeout and must always release the held Promise in `finally`/failure cleanup so a failed assertion cannot strand the suite.

## 3. First gate — run only corrected CON-03

Run only the corrected CON-03 test from the P2-001 file, in-band, canonical Supabase Testing.

Required:
- exit 0;
- one test PASS;
- two distinct positive PostgreSQL backend PIDs captured;
- explicit overlap proof (`B started` + `B not settled before A release`);
- after release exactly one active source binding / exactly one 5.000000 winner.

If this fails because business/result invariants fail, STOP and report the exact invariant failure.

If this fails only because the test harness cannot deterministically synchronize its own hooks, one harness-only correction within this shot is allowed. No product-code correction is authorized.

## 4. Second gate — full accepted P2-001 regression

If corrected CON-03 passes, run the complete:

`apps/mercato/src/modules/wms_outbound/services/__tests__/p2-001-crossdock-eligibility-postgres.integration.test.ts`

Required: **10/10 PASS, exit 0** on canonical Supabase Testing.

Confirm TC-134 is separately represented by the hard-demand-coverage assertion and CON-03 by the concurrency assertion.

If not 10/10, STOP.

## 5. Commit harness correction, then freeze final Mercato SHA

If the full P2-001 suite is green:

- verify diff contains only the authorized P2-001 test-harness correction;
- commit it on the current P2-002 Mercato branch with a descriptive test-only commit;
- push;
- record the new final Mercato SHA.

Scanner remains `8ff264e2e03d75abd422f69a7ff1f713f10d32fc` unless an unrelated dirty state is found; do not modify Scanner in this correction.

If unrelated changes are present, STOP.

## 6. Re-run P2-002 dedicated suite on the new final Mercato SHA

Run:

`apps/mercato/src/modules/wms_outbound/services/__tests__/p2-002-crossdock-planning-postgres.integration.test.ts`

Required: **22/22 PASS, exit 0** on canonical Supabase Testing.

If not 22/22, STOP.

## 7. Complete remaining mandatory Mercato regressions

On the same final Mercato SHA, run the exact mandatory regressions required by `P2-002_EXECUTION.md`:

1. accepted P1-003 grouping/planning PostgreSQL suite(s);
2. accepted P1-005 assignment/ordering regression because P2-002 uses accepted warehouse ordering / active-task semantics;
3. FND-003 shared task-lock / warehouse compatibility suite(s) for the touched primitives;
4. accepted Inbound cross-dock regression only if the actual final Mercato diff touches shared TU/Inbound boundary primitives; otherwise record `NOT REQUIRED — untouched`.

Do not run unrelated dispatch/manifest/final-settlement suites.

Capture literal commands, suite/test counts and exit codes.

First non-zero result => STOP, no canonical P2-002 evidence.

## 8. Scanner regressions + mandatory P2-002 Playwright

Scanner changed in P2-002, so run relevant accepted Scanner auth/warehouse-context/task-assignment regressions on exact Scanner SHA `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`.

Then run the bounded P2-002 Playwright journey against canonical Testing with the final Mercato SHA + exact Scanner SHA.

It must prove through the normal rendered application, with zero route mocks:

1. Warehouse Operator authenticates;
2. enters/selects Crossdock module;
3. no zone selector is required/shown;
4. receives exactly one next eligible CrossDockPickTask in configured ordering;
5. task identity, source Inbound TU and planned quantity are visible;
6. a second operator cannot receive the same task;
7. an operator with another active warehouse task is blocked;
8. STOP before any P2-003 sorting/source/SKU/quantity/quality/target-TU action.

Any failure, route mock, wrong runtime identity, or ambiguous result => STOP.

## 9. Final invariants

Before evidence, independently verify from final code + real-PG results:

- no `Allocation` is created;
- positive active P2-001 binding maps to exactly one OOL and exactly one CrossDockPickTask;
- `plannedQty` equals the accepted binding quantity exactly;
- replay/concurrency cannot append a second task or line;
- source capacity and customer-line demand cannot be over-covered;
- CROSSDOCK channel is immutable and never mixed with STANDARD;
- grouping uses exact customer/address/priority/slaDeadline and correct allowPartial rules;
- source TU/binding/SKU/CustomerOrderLine correlation persists;
- no eager physical target outbound TU exists in P2-002;
- assignment has no zone input, uses accepted ordering, blocks any active warehouse task, and prevents double assignment;
- CON-03 proof uses genuine overlapping independent PostgreSQL transactions on canonical Supabase Testing without requiring ephemeral PostgreSQL observer state;
- org/tenant/warehouse isolation holds;
- no P2-003 scope leakage.

## 10. Canonical evidence

Only if every gate above is green, create/update exactly:

`WMS_Outbound/05_EVIDENCE/P2-002_EVIDENCE.md`

Record:

- accepted P2-001 base and evidence SHA;
- exact final Mercato SHA and merge base;
- exact Scanner SHA and merge base;
- changed-file scope;
- sanitized canonical Supabase Testing provenance (`yzonugcenguvmojwiihb` only; no credentials/full URL);
- migration/schema/uniqueness/idempotency/channel-immutability details;
- P2-002 dedicated **22/22** result;
- P2-001 regression **10/10** result;
- corrected CON-03 literal proof: two distinct backend PIDs, controlled overlap, B not settled before A release, exactly-one persisted winner after release;
- explicit statement that `pg_blocking_pids` / `wait_event_type='Lock'` are not Architect acceptance requirements and are not used as the gate;
- TC-134's correct hard-demand-coverage mapping;
- remaining Mercato regression suite names/counts;
- Scanner regression names/counts;
- Playwright command/runtime/result;
- no-Allocation and lazy-target-TU proof;
- clean final worktree identities;
- explicit P2-003+ exclusions.

Do NOT claim Owner acceptance, Human Verified, FINAL PASS, or update project progress.

Commit/push only the canonical WMS evidence file to `WMS_Outbound/main`.

## 11. STOP / report

STOP and report only:

- final Mercato SHA;
- Scanner SHA;
- corrected CON-03 result;
- P2-001 full count;
- P2-002 dedicated count;
- remaining mandatory regression counts;
- Scanner regression counts;
- Playwright count/status;
- WMS evidence commit SHA;
- final clean/dirty status for Mercato and Scanner.
