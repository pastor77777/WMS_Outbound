# P2-002 VPS regression run — raw diagnostic record

**Mercato candidate:** `24d102a41dcfff669cf6b839b369fbaa3620f87d` (`outbound/p2-002`)

**Scanner:** frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## Clean-up preflight

The VPS process audit found no stale Jest/Yarn test process before this run.
The approved Testing PostgreSQL activity and blocking-lock queries returned no
matching session, waiting transaction, or blocker. No PostgreSQL session was
terminated.

An earlier orphaned composite Jest process from the prior command-execution
venue was identified by its command line and 20-minute elapsed time, and was
terminated before this guide's clean preflight. Post-clean process and
PostgreSQL queries were empty for the scoped test/lock predicates.

## P2-001 regression attempt

Exact command, executed in a detached owner-controlled VPS `tmux` session:

```bash
set -a && source /etc/mercato-localhost.env && set +a && \
yarn workspace @open-mercato/app test --runInBand \
  src/modules/wms_outbound/services/__tests__/p2-001-crossdock-eligibility-postgres.integration.test.ts
```

The command produced no stdout/stderr and no Jest summary for more than 80
seconds. Its process tree was verified as `bash → yarn → jest`; it had no
active Testing PostgreSQL session or lock at the time of the hang. To prevent
another orphaned test process, only that exact tmux test process group was
terminated. Therefore no normal test exit status or pass count exists for this
attempt, and it is **not PASS**.

Post-termination scoped process audit and Testing PostgreSQL lock/activity
query were empty.

## Boundary

No Mercato or Scanner product/test file, migration, configuration, state or
handover was changed by this cleanup/regression venue work. Canonical
`P2-002_EVIDENCE.md` was not created because mandatory regression completion
was not obtained.
