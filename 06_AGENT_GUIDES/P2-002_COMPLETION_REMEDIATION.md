# P2-002 — completion remediation

## Frozen baseline

- Mercato implementation SHA: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Accepted P2-001 base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Scanner frozen baseline: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- Approved Testing PostgreSQL only: Supabase project ref `yzonugcenguvmojwiihb` via canonical `/etc/mercato-localhost.env`.
- Local PostgreSQL/local socket/local test DB is forbidden and cannot count as evidence.

Read and obey `06_AGENT_GUIDES/P2-002_EXECUTION.md`. This shot exists only to close the two verified completion gaps below and then run final verification.

## Verified gaps to close

1. The current dedicated P2-002 real-PostgreSQL suite contains only 4 tests, while section 11 requires at least 22 substantive real-PG scenarios. Do not relabel/split trivial assertions to inflate count.
2. Frozen Scanner baseline contains no crossdock module, while P2-002 requires a bounded Scanner crossdock-module assignment journey. Therefore minimal Scanner implementation + Playwright proof is required.

## Allowed implementation

### A. Mercato test/evidence completion

Expand `p2-002-crossdock-planning-postgres.integration.test.ts` (or an equivalent dedicated P2-002 real-PG suite) to cover, as distinct substantive scenarios, all 22 items listed in section 11 of `P2-002_EXECUTION.md`, including:

- positive binding -> immutable CROSSDOCK order/line/task;
- no Allocation;
- exact `plannedQty`;
- replay exactly-once;
- concurrent planner exactly-once;
- channel isolation/immutability;
- grouping compatibility and every incompatibility dimension;
- allowPartial true/false rules;
- exact priority/SLA inheritance;
- source/binding/SKU/customer-line correlation;
- no eager physical outbound TU;
- assignment ordering with no zone input;
- concurrent assignment no double assignment;
- active warehouse task guard;
- org/tenant/warehouse isolation;
- no later task append for later BACKORDERED demand;
- P2-001 source/demand eligibility non-regression.

For material concurrency tests use genuine independent overlapping PostgreSQL transactions/sessions against approved Supabase Testing and capture decisive `pg_stat_activity` / `pg_blocking_pids` or equivalent server-side lock evidence tied to actual participant PIDs.

If a substantive test exposes a real P2-002 product defect, make only the minimum P2-002 product correction needed. No P2-003 behavior.

### B. Scanner bounded crossdock module

Branch Scanner from exact `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` only if changes are required.

Implement the minimum UI path using existing Scanner auth + active warehouse context patterns:

- Warehouse Operator can enter/select a `Crossdock` module;
- no zone selector is shown or required;
- request exactly one next task through the P2-002 backend assignment endpoint/service;
- render assigned task identity, source inbound TU, SKU/item and `plannedQty`;
- if operator already has any active supported warehouse task, show the backend rejection/fail closed;
- a second operator cannot receive the same task;
- stop UI scope before source scan, quantity/quality confirmation, target-TU placement/create, sealing, completion, shortage, or any P2-003 action.

Do not invent a Supervisor assignment pool.

## Mandatory verification order

Use owner-controlled VPS shell/tmux. Each command must have literal stdout/stderr and numeric exit status. No silent wrappers. No blind retries after a red/timeout.

1. Confirm Mercato exact intended final SHA lineage from `24d102...` and Scanner lineage from `f4a404...`.
2. Confirm sanitized DB provenance resolves to Supabase project ref `yzonugcenguvmojwiihb`; do not print credentials/full URL.
3. Run the dedicated P2-002 real-PG suite on approved Supabase Testing. It must contain >=22 substantive scenarios and be fully green.
4. Run accepted P2-001 PostgreSQL regression on the same final Mercato SHA.
5. Run accepted P1-003 grouping/planning regression.
6. Run P1-005 task-assignment/ordering regression and FND-003 shared task-lock/warehouse compatibility regression because P2-002 assignment relies on those semantics.
7. Run accepted Inbound crossdock/shared-boundary regression if any shared TU/Inbound primitive changed.
8. Run relevant Scanner P1 auth/warehouse/task-assignment regressions because Scanner changes are required here.
9. Run mandatory Playwright bounded Scanner crossdock assignment journey against canonical Testing runtime: operator login -> active warehouse -> Crossdock module -> no zone -> next task visible with planned quantity -> second operator cannot receive same task -> active-task operator blocked. Stop before P2-003.

Any failure, timeout, ambiguous provenance, missing lock proof, or missing required scenario => STOP immediately. Do not create canonical evidence.

## Canonical evidence only after all green

Create/update only:

`05_EVIDENCE/P2-002_EVIDENCE.md`

Record:

- exact final Mercato SHA and merge base;
- exact final Scanner SHA and merge base;
- compare/file scope for both repos;
- sanitized Supabase Testing provenance (`yzonugcenguvmojwiihb`);
- dedicated P2-002 suite command and exact scenario/test count (>=22);
- literal CON-03/server lock evidence;
- exact P2-001, P1-003, P1-005, FND-003, Inbound-if-touched and Scanner regression suite names/counts;
- Playwright command/result and exact tested runtime revisions;
- proof no Allocation exists;
- proof target outbound TU remains lazy / no P2-003 scope leakage;
- clean worktree identities.

Do not claim Supervisor PASS, Owner Accepted, FINAL PASS, or update STATE/handover/progress in this shot.

Push required Mercato/Scanner implementation/tests and WMS evidence, then STOP and report only final SHAs plus exact test/regression/Playwright counts.