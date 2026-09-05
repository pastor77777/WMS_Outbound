# P2-002 — regression-result capture corrective

**Scope:** one narrow corrective shot authorized by Owner after the P2-002 implementation was pushed but mandatory regression invocations ended without durable result/exit-status output.

## Frozen inputs

- Accepted P2-001 Mercato base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`.
- Current pushed P2-002 Mercato head: `24d102a41dcfff669cf6b839b369fbaa3620f87d` on `outbound/p2-002`.
- Scanner remains frozen: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- Authoritative P2-002 execution contract remains `06_AGENT_GUIDES/P2-002_EXECUTION.md`.

Do not alter business code, schema, migrations, tests, Scanner, STATE, handovers, Architect/Canon/traceability, `AGENTS.md`, credentials, runtime configuration, or unrelated files in this corrective shot.

## Objective

Run the mandatory P2-002 regression suites from the exact current P2-002 Mercato head and capture, for every invocation:

1. the exact command;
2. complete stdout/stderr in a temporary log outside the repository;
3. explicit process exit status;
4. final Jest/test summary and exact suite/test counts when emitted;
5. enough tail output to diagnose any non-zero result.

A silent/empty invocation is not PASS.

## Required regressions

Use the exact accepted regression suites/commands required by `06_AGENT_GUIDES/P2-002_EXECUTION.md` and the corresponding accepted evidence/implementation files. At minimum this includes:

- accepted P2-001 PostgreSQL suite;
- accepted P1-003 grouping/planning regression suite(s);
- accepted P1-005 task-assignment/ordering regression suite(s) only if the P2-002 diff reuses/touches their shared assignment primitive;
- FND-003 shared task-lock/TU/warehouse compatibility suite(s) for touched primitives;
- accepted Inbound cross-dock result regression only if the P2-002 diff touches shared TU/Inbound boundary primitives.

Do not invent unrelated regressions merely to increase counts.

## Mandatory capture pattern

Run suites **one command at a time**, not as a compound chain whose earlier output can disappear.

For each exact regression command, use an equivalent shell wrapper that preserves the real exit code while always printing a result marker. Example pattern:

```bash
log="/tmp/p2-002-regression-<short-name>.log"
rm -f "$log"
set +e
<EXACT_REGRESSION_COMMAND> >"$log" 2>&1
rc=$?
set -e
printf '\n=== P2-002 REGRESSION RESULT: <short-name> ===\n'
printf 'exit_status=%s\n' "$rc"
printf 'log=%s bytes=%s\n' "$log" "$(wc -c <"$log")"
tail -n 160 "$log"
printf '=== END RESULT: <short-name> ===\n'
test "$rc" -eq 0
```

Requirements for this wrapper:

- do not pipe the test command through `tee` unless `set -o pipefail` is explicitly active and the captured status is the test process status;
- do not background the command;
- do not rely on terminal rendering as the only result source;
- keep logs under `/tmp` (or another non-repository temporary location); do not commit raw logs;
- if the command exits non-zero, preserve the log and report the exact exit status + failure summary;
- if the command exits zero but the log has no credible test summary/count, do **not** call it PASS; inspect the log/command invocation and report the ambiguity.

## No implementation repair in this shot

This corrective is diagnostic/evidence capture only.

If any regression genuinely fails:

- do not modify product/test code in this shot;
- report the exact failing suite/test, command, exit status and relevant failure summary;
- STOP.

If a regression cannot produce a trustworthy result even with explicit file capture and exit-status handling:

- report that exact infrastructure/tooling blocker;
- STOP.

## Evidence continuation

Only if every required regression has a trustworthy passing result may you resume the **evidence-only remainder** of `P2-002_EXECUTION.md`:

- create/update only `05_EVIDENCE/P2-002_EVIDENCE.md`;
- record exact commands, suite/test counts, final Mercato SHA `24d102a41dcfff669cf6b839b369fbaa3620f87d`, frozen Scanner SHA, dedicated P2-002 PostgreSQL `4/4`, and the captured regression results;
- push the evidence commit;
- do not change Mercato or Scanner code;
- do not claim FINAL PASS / Supervisor acceptance / Owner acceptance.

## STOP

Report either:

- all required regression names + exact counts + exit status 0 and the resulting WMS evidence SHA; **or**
- the first trustworthy failing/ambiguous regression with its exact exit status and summary.

Then STOP. Do not start P2-003.
