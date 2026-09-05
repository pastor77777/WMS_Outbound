# P2-002 — dedicated-suite harness corrective

## Purpose

Continue the already-authorized P2-002 completion remediation after the dedicated real-PostgreSQL suite reached 19/22 with three harness-only failures.

This shot is restricted to fixing those harness defects and rerunning ONLY the dedicated P2-002 PostgreSQL suite.

## Frozen / preserved state

- Preserve all current uncommitted Mercato and Scanner product changes from the P2-002 completion remediation exactly as they are.
- Do not discard, reset, rebase, stash, or rewrite the current worktree.
- The additive P2-002 migration has already been applied to canonical Testing Supabase; do not reapply, rollback, or alter it unless the existing dedicated suite itself requires normal test cleanup.
- Database: canonical approved Supabase Testing only. Local PostgreSQL is forbidden.

## Allowed changes

Change ONLY the dedicated P2-002 PostgreSQL test harness needed to fix the three reported failures:

1. missing MikroORM metadata for the Allocation assertion;
2. missing MikroORM metadata for the TU-registry / lazy-target-TU assertion;
3. unsupported `persistAndFlush` usage.

Use the minimum harness-only correction:

- add/import/register only the exact existing entity metadata required by those assertions;
- replace unsupported `persistAndFlush` with the repository-supported explicit `persist(...)` + `flush()` pattern (or exact equivalent already used by accepted suites);
- do not weaken assertions, skip tests, change scenario meaning, reduce concurrency, change timeout to mask failures, or change product code to satisfy the harness.

## Forbidden

Do NOT:

- change P2-002 product/business logic;
- change migrations/schema;
- change Scanner product code;
- change shared primitives;
- change accepted P2-001/P1/FND tests;
- run broad regressions;
- create canonical `05_EVIDENCE/P2-002_EVIDENCE.md` yet;
- update STATE/handover;
- claim acceptance.

## Execution

1. Confirm the three failing dedicated-suite cases are exactly harness failures matching the report above.
2. Apply the minimum test-only correction.
3. Run ONLY the dedicated P2-002 PostgreSQL integration suite against canonical Testing Supabase, same final worktree, `--runInBand`, no retries.
4. Require exactly 22 substantive scenarios to execute.

## Stop rules

- If any of the 22 tests is red, timed out, skipped unexpectedly, or ambiguous: STOP. Do not commit/push Mercato or Scanner. Report exact failing test names and errors.
- If 22/22 pass: keep the corrected test harness with the current uncommitted P2-002 worktree, report the literal suite summary and exact changed-file list, then STOP.
- Do not proceed to regressions or evidence in this shot.
