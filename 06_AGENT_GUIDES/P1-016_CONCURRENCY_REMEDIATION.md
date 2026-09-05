# P1-016 — concurrency harness remediation + final closeout

**Purpose:** preserve the current local P1-016 implementation and repair only the real-PostgreSQL concurrency/rollback evidence harness that blocked closeout, then continue through the remaining P1-016 closeout phases in the same run.

This is not a new implementation pass.

## Preserved local checkpoint

Current owner-reported local Mercato checkpoint:

`ae24afcdd72d0eb4af611909865106278b01ab46`

Preserve it. Do not reset/rebase/restart P1-016.

Accepted P1-015 base remains:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

Reported P1-016 dedicated PostgreSQL suite already passes:

`22/22`

Scanner remains frozen:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Source authority remains:

- `06_AGENT_GUIDES/P1-016_EXECUTION.md`
- `06_AGENT_GUIDES/P1-016_CLOSEOUT.md`

## Exact blocker to remediate

The closeout stopped because the required real-concurrency proof hung on the test barrier. The failure is in the test/evidence orchestration unless inspection proves a genuine product deadlock.

Do not weaken the concurrency requirement. Do not replace it with mocks, fake overlap, same-session calls, or timing-only assertions.

## Required harness pattern

For both required races, avoid a symmetric barrier that can leave both test workers waiting forever while one transaction already owns the row lock needed by the other.

Use deterministic one-way orchestration:

1. create two independent real PostgreSQL service-call sessions/EntityManagers and one independent observer session;
2. capture `pidA` and `pidB` from the exact service-call sessions before contention and prove `pidA != pidB`;
3. start operation A and hold it only after it has entered the real transaction and acquired the decisive business row lock/serialization point;
4. signal the test harness externally that A is at the lock-holding point;
5. only then start operation B on the second independent session;
6. observer polls `pg_stat_activity` / `pg_blocking_pids(pidB)` (or the exact waiter PID) until the real blocked relationship is visible;
7. record exact waiter/blocker PIDs and decisive lock wait evidence;
8. release A from the test-only orchestration gate;
9. allow both real service calls to finish naturally;
10. verify persisted arithmetic/idempotency with a fresh independent read.

The orchestration gate must be test-only/deterministic and must not alter production business semantics. Do not use arbitrary sleep duration as the concurrency proof.

If current code exposes no safe deterministic point after the decisive lock is acquired, add the smallest test-only hook/seam required to observe that point. Do not add a production feature or change business behavior merely to make the test easy.

If inspection proves the product itself deadlocks under the required lock order, make only the smallest P1-016 correction required and rerun all final-SHA evidence affected by that correction.

## Race A — same manifest duplicate/parallel settlement

Produce the exact proof required by the source guide:

- two independent real service calls;
- exact `pidA`, `pidB`;
- real waiter/blocker relationship from PostgreSQL;
- one settlement contribution only;
- one Inventory effect only;
- one Allocation decrement only;
- one line `shippedQty` increment only;
- no duplicate terminal transition/event;
- safe replay/loser outcome.

## Race B — two manifests settle the same line/allocation

Use the same deterministic holder/waiter/observer pattern against one shared mutable line/allocation split across two manifest contributions.

Prove:

- exact independent PIDs and real overlap;
- actual PostgreSQL lock wait/serialization;
- no lost update;
- exact final `reservedQty` arithmetic;
- exact final `shippedQty` arithmetic;
- each manifest contribution exactly once;
- correct terminalization only after complete coverage;
- result independent of winner order.

## Rollback proof

After concurrency harness remediation, complete the required rollback proof separately:

- real transaction;
- deterministic injected failure after at least one meaningful P1-016 mutation has been flushed/staged and before successful completion;
- rollback;
- fresh independent session/EntityManager read;
- prove no partial settlement identity, Inventory, Allocation, line/order/customer state or event remains.

Do not use a same-session cached read as rollback evidence.

## Continue the entire closeout in this same run

Once concurrency + rollback proof succeeds, do not stop. Continue all remaining items from `P1-016_CLOSEOUT.md`:

1. final-SHA mandatory regressions;
2. serve canonical Testing from exact final Mercato SHA;
3. fresh rendered P1-016 Playwright `6/6` with `0 unexpected` and `0 flaky` (or equivalent explicit clean result);
4. create/update only `05_EVIDENCE/P1-016_EVIDENCE.md` with literal decisive evidence;
5. push final Mercato P1-016 commit(s);
6. push WMS evidence;
7. report final Mercato SHA, evidence SHA, P1-016 PostgreSQL count, mandatory regression counts, Playwright count and frozen Scanner SHA;
8. STOP.

## Early-stop rule

Stop before completion only if, after this one genuine harness remediation attempt, there is a material blocker that prevents required evidence from being produced. If so, preserve the current local SHA and report the exact technical blocker. Do not claim completion and do not broaden scope.

## Hard exclusions

- no STATE/handover/AGENTS/.ai/steering changes;
- no P2/P3/P4 implementation;
- no Scanner product changes;
- no Prod/Demo writes;
- no external carrier APIs;
- no Return Receipt;
- no weakening or faking of PostgreSQL concurrency/rollback evidence.
