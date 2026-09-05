# P2-002 — Supabase connectivity isolation

Status: narrow diagnostic only

## Frozen refs

- Mercato: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Accepted P2-001: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Scanner frozen: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

## Canonical database rule

All PostgreSQL diagnostics/tests in this shot MUST use the approved Supabase Testing project `DevAxonic_Platform` / project ref `yzonugcenguvmojwiihb` through the approved environment. Local PostgreSQL, localhost DB, local socket, SQLite, fallback DB, or altered DATABASE_URL is forbidden.

Do not print or persist credentials/full DATABASE_URL.

## Only objective

Determine whether the current P2-001 regression stall is at:
1. VPS -> Supabase `pg` connection/query;
2. MikroORM initialization against the same Supabase database; or
3. execution of the P2-001 test body after successful initialization.

## Hard boundaries

- NO product/business code changes.
- NO test source changes.
- NO Jest/harness/config changes.
- NO migrations/schema changes.
- NO Scanner changes.
- NO dependency changes.
- NO canonical `05_EVIDENCE/P2-002_EVIDENCE.md`.
- NO STATE/handover changes.
- Use owner-controlled VPS shell/tmux only.
- One bounded attempt per probe; no blind retries.

## Procedure

1. Sync `WMS_Outbound/main`.
2. Verify Mercato branch `outbound/p2-002`, clean worktree, exact HEAD `24d102a41dcfff669cf6b839b369fbaa3620f87d`.
3. Load the approved Testing environment exactly as used by canonical Mercato runtime. Record only sanitized provenance proving Supabase host/project ref `yzonugcenguvmojwiihb`; never record secrets.
4. Run a bounded raw Node `pg` probe using the existing installed `pg` package and the unchanged `DATABASE_URL`:
   - connect timeout <= 10s;
   - execute `select 1 as ok, current_database() as db`;
   - print timestamps before connect, after connect, after query, after close;
   - capture stdout/stderr and numeric exit status.
5. If raw `pg` does not connect/query decisively, STOP. Do not run Jest.
6. If raw `pg` succeeds, run one bounded ephemeral Node diagnostic (heredoc/stdin only; do not create repo source files) that imports the same MikroORM driver and exact entity set used by `p2-001-crossdock-eligibility-postgres.integration.test.ts`, then:
   - print timestamp immediately before `MikroORM.init`;
   - initialize using unchanged `DATABASE_URL` and the same SSL behavior as the test;
   - print timestamp immediately after init;
   - execute one trivial `select 1` through the initialized ORM;
   - close ORM;
   - capture stdout/stderr and numeric exit status.
7. If MikroORM init/query does not complete decisively, STOP. Do not run Jest.
8. If both raw `pg` and MikroORM probes succeed, run ONLY the first named P2-001 test using Jest `--runInBand` plus a test-name filter for:
   `TC-020/102: accepts only ELEMENTARY at IN_CROSS_DOCK and persists ASN declaration/correlation`
   Use the unchanged canonical Testing environment. Bound the process generously enough for remote PostgreSQL (up to 90s). Capture literal stdout/stderr and numeric exit status.
9. Do not run the whole P2-001 suite in this shot.
10. After every probe, verify no orphan Jest/Yarn/Node diagnostic process remains.

## Result record

Create/update ONLY a raw diagnostic record under `05_EVIDENCE/` named `P2-002_SUPABASE_CONNECTIVITY_ISOLATION.md` containing:
- frozen SHA;
- sanitized Supabase provenance;
- raw `pg` timings/result/exit status;
- MikroORM timings/result/exit status if reached;
- single-test result/exit status if reached;
- narrow classification: `PG_CONNECT`, `MIKROORM_INIT`, `TEST_BODY`, or `NO_STALL_REPRODUCED`.

Push only that diagnostic record to `WMS_Outbound/main`, then STOP.

Do not claim P2-002 PASS or acceptance.