# Execution guide

## Before a task

- open the task ID in `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md`;
- resolve every listed requirement through `requirements_index.csv`;
- inspect the underlying architect rule/process/state source;
- inspect current code/DB in the owning application;
- classify each touched existing component as REUSE/EXTEND/MIGRATE/SUPERSEDE/LEAVE_UNTOUCHED.

## During implementation

- keep server/domain authoritative for inventory, locks, task ownership and state transitions;
- use transactions/idempotency for CON requirements;
- preserve shared Inbound behavior;
- do not create hidden duplicate domain ownership between legacy and target models;
- keep migration/cutover explicit and observable.

## Testing

Use component/integration tests first for fast feedback, then the normal UI Playwright journey for human-facing behavior. The Playwright journey must prove the application can actually be traversed; final acceptance is Human Verified.

## After a task

Record code commit(s), migration, executable tests, Playwright test IDs/artifacts, persisted evidence and Human Verified status where applicable. Update task/evidence traceability without altering architect source.
