# X-001 — acceptance correction

**Status:** same-item correction, X-001 remains not accepted
**Effective:** 2026-09-06
**Mercato current candidate:** `outbound/x-001` @ `f7af856519edfd3b3bb461c5ff23a949b7974654`
**Scanner:** frozen @ `a2759a29347285dd1dcd14bf51633431fbf2a302`
**Current evidence:** `05_EVIDENCE/X-001_EVIDENCE.md` @ WMS `40236ecd538e91b5c36a35dadd44ae8a0bc8b483`

Do not reset Testing. Continue the same X-001 item only.

## Acceptance gap A — CON-03 is not yet decisive

The current P2-002 tests claimed for CON-03 use ordinary `Promise.all` plus final row-count assertions:

- `is replay-safe and concurrent calls produce one line/task for the binding`
- `concurrent operators can receive one created task at most once`

Those tests do not force genuine transaction overlap and do not record PostgreSQL-side lock/serialization evidence. This does not satisfy `X-001_EXECUTION.md`, which explicitly requires genuinely independent PostgreSQL sessions/transactions and actual lock/wait proof, not Promise timing alone.

The current product service already has relevant DB protection (`PESSIMISTIC_WRITE` on the binding plus `pg_advisory_xact_lock` for plan; warehouse-scoped advisory lock for assignment). Preserve business behavior unless the stronger test exposes a real defect.

Correction requirement:

- harden the authoritative CON-03 planning/quantity race so two independent PostgreSQL sessions are forced to overlap while competing for the same source binding / source TU-SKU quantity;
- capture distinct `pg_backend_pid()` values and decisive `pg_stat_activity` / `pg_blocking_pids` / `pg_locks` evidence that the loser is blocked/serialized by the intended PostgreSQL lock;
- fresh independent reads must prove exactly one durable CrossDockPickTask / one corresponding OOL and no duplicated planned quantity;
- also prove the relevant assignment race with genuine DB overlap if the planning proof alone does not cover the guide's claimed single-assignment boundary;
- add only minimal test-support hooks if needed. Do not alter product behavior unless the stronger race exposes a defect.

## Acceptance gap B — mandatory final-head dedicated-suite reruns were skipped

`X-001_EXECUTION.md` says: **"At minimum rerun every dedicated suite whose accepted proof is claimed for X-001 on the final X-001 head."**

Current evidence explicitly says the owning suites for CON-03/04/05 were read but not rerun. That is not compliant with the guide.

After the CON-03 correction, rerun on the final X-001 Mercato head every dedicated suite whose proof is claimed in X-001 evidence:

- P1-004 — CON-01
- P1-005 — CON-02
- P2-002 cross-dock planning/assignment — CON-03
- P1-011 — CON-04 grouping
- P1-015 — CON-04/CON-05 manifest races
- P1-014 — CON-05 ERP posting
- P1-016 — CON-05 final settlement
- P2-006 — CON-05 shared cross-dock downstream settlement

Do not add unrelated broad sweeps. Rerun directly affected caller regressions only if product/shared signatures change further.

## Evidence correction

Update `05_EVIDENCE/X-001_EVIDENCE.md` so it no longer describes Promise-only CON-03 tests as decisive. Record:

- exact final Mercato SHA;
- exact hardened CON-03 test names;
- participant PostgreSQL PIDs and lock/wait evidence;
- fresh durable state assertions;
- exact results of all mandatory dedicated-suite reruns above;
- any product/test-support change and why it was necessary.

Typecheck remains required after final code change. No UI/Scanner/Playwright work unless the correction changes user-visible behavior.

## Stop boundary

Push Mercato `outbound/x-001` and updated WMS evidence, then STOP. Do not update accepted count/state/handover. Do not start X-002.
