# P2-002 — OWNER-AUTHORIZED EXTRA CON-03 SHOT

## Owner authorization
The Owner explicitly authorized exactly one additional material attempt for the remaining P2-001 CON-03 regression blocker.

This guide is ONLY that extra shot. Do not broaden scope.

## Frozen product revisions
- Mercato product head entering this shot: `92b50ca0eeeb41adc71800f1da036e73562552c6`
- Scanner: `8ff264e2e03d75abd422f69a7ff1f713f10d32fc`
- P2-002 dedicated suite is already `22/22 PASS` and MUST NOT be rerun here.

Use only the approved canonical Supabase Testing PostgreSQL environment. Local PostgreSQL is forbidden.
Sanitized target must still resolve to Supabase project ref `yzonugcenguvmojwiihb`.
Never print credentials or full `DATABASE_URL`.

## Proven state entering shot
- P2-001 cases 1–9 pass individually.
- Only P2-001 `CON-03/TC-134` remains red.
- Raw VPS->Supabase connectivity passes.
- Native Jest + exact MikroORM init passes.
- First normal P2-001 functional path passes.
- Current service acquires transaction-scoped advisory locks BEFORE the pessimistic source-row lock.
- Therefore the second participant may legitimately wait on the advisory lock before it can reach the row lock. The harness must observe the actual PostgreSQL blocker, not assume the exact lock subtype in advance.

## Strict scope
Allowed change:
- ONLY the P2-001 PostgreSQL integration test harness for `CON-03/TC-134`.

Forbidden:
- product/service/business-logic changes;
- entity or migration changes;
- Scanner changes;
- changing P2-001 semantics or weakening exactly-once/source-capacity assertions;
- deleting the concurrency proof;
- replacing genuine overlapping transactions with sequential calls;
- local DB;
- broad regressions, P2-002 dedicated rerun, Playwright, canonical evidence, STATE or handover.

## Required correction method
Keep two genuine concurrent `evaluateAndBind` calls on independent PostgreSQL sessions.

Use the existing instrumentation hooks to capture the actual participant backend PIDs:
- Actor A PID from `onSerializationAcquired`; hold Actor A there with an explicit promise/barrier.
- Actor B PID from `onTransactionStarted`.

Do NOT use a fixed 50/100 ms sleep as the decisive proof.

Open a THIRD independent observer connection using the already-installed `pg` package and the unchanged canonical `DATABASE_URL`.
The observer must be separate from both actor EntityManagers.

Poll for up to 10 seconds at a short bounded interval (for example 50–100 ms) for Actor B to be observably waiting in PostgreSQL.
For each poll, query PostgreSQL using the captured Actor A/B PIDs and inspect at minimum:

```sql
SELECT pid, wait_event_type, wait_event, state
FROM pg_stat_activity
WHERE pid = ANY($1::int[]);
```

and:

```sql
SELECT pg_blocking_pids($1::int) AS blockers;
```

Also inspect participant lock rows so the proof does not depend on only one catalog view:

```sql
SELECT pid, locktype, mode, granted, classid, objid, objsubid
FROM pg_locks
WHERE pid = ANY($1::int[])
ORDER BY pid, locktype, granted;
```

Accept the lock-wait observation only when ALL are true before releasing Actor A:
1. Actor A PID > 0 and Actor B PID > 0 and they are distinct.
2. Actor B has `wait_event_type = 'Lock'` OR an ungranted lock row.
3. `pg_blocking_pids(actorB)` contains Actor A OR lock-catalog evidence ties Actor A granted lock to Actor B's ungranted wait on the same lock identity.
4. Actor A is still held by the explicit barrier when the observation is captured.

The harness may report whether the actual wait is advisory or row/transaction lock. Do not require one subtype if the service's authoritative locking order makes another PostgreSQL lock the first serialization point.

After decisive observation:
- release Actor A;
- await both calls;
- retain the original business assertions that total active bound quantity is exactly source capacity, only one active positive binding remains for the winning highest-priority demand, and results are `0.000000` + `5.000000` as applicable;
- close the observer in `finally` so no connection leaks.

## Execution
Run ONLY the single CON-03 test from:
`src/modules/wms_outbound/services/__tests__/p2-001-crossdock-eligibility-postgres.integration.test.ts`

Use native project Jest config, `--runInBand`, exact canonical Testing env, and an outer deterministic timeout no greater than 90 seconds.
Capture literal stdout/stderr and numeric exit.

## Success path
If and only if CON-03 passes with the genuine PostgreSQL wait proof and all original serialization/business assertions intact:
1. commit ONLY the P2-001 test-harness correction in Mercato;
2. push the existing P2-002 branch;
3. report the new Mercato SHA and literal CON-03 result;
4. STOP.

Do NOT run remaining regressions or Playwright in this shot.
Do NOT create `05_EVIDENCE/P2-002_EVIDENCE.md`.

## Failure path
If the single corrected CON-03 still fails, times out, cannot observe a genuine PostgreSQL blocker, or would require product/service changes:
- do not make another correction;
- leave product code unchanged;
- do not create canonical evidence;
- report the exact observed PostgreSQL participant/lock facts and numeric exit;
- STOP.
