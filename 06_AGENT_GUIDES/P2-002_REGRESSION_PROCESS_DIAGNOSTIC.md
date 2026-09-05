# P2-002 — regression process diagnostic

**Scope:** diagnostic only. Determine why required regression commands terminate without output/exit marker. Do not change business code or test logic.

## Frozen refs

- Mercato `outbound/p2-002`: `24d102a41dcfff669cf6b839b369fbaa3620f87d`.
- Accepted P2-001 base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`.
- Scanner frozen: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## Objective

Diagnose the silent disappearance observed while invoking required regression suites: empty log, no result marker, no exit status, and no remaining test process.

Do not attempt the whole regression matrix until the failure mode is identified.

## Allowed actions

- inspect shell/process environment, wrapper behavior, parent/child process tree, signals, kernel/system logs available to the user, resource limits, OOM evidence, package-manager/Jest behavior, terminal/session teardown behavior, and command redirection semantics;
- run minimal non-mutating control commands and the exact first affected P2-001 regression command under instrumentation;
- capture PID/PPID/PGID/session, timestamps, stdout/stderr separately, wrapper exit trap/marker, and process termination signal/status when obtainable;
- use standard diagnostics such as `set -x`, `trap`, `ps`, `/proc`, `ulimit`, `dmesg`/journal reads if permitted, `time`, or equivalent read-only tooling;
- if the problem is the wrapper/command invocation itself, prove a corrected invocation that reliably returns an exit status and preserves the exact test target.

## Forbidden actions

- no changes to Mercato/Scanner business code;
- no changes to regression test source files or expectations;
- no DB/schema changes;
- no dependency upgrades or package-lock changes;
- no STATE/handover/evidence writes;
- no claim that a regression passed unless the actual test command completes with captured result and exit status.

## Diagnostic sequence

1. Confirm exact Mercato SHA and clean worktree.
2. Reproduce a trivial control command through the same wrapper/redirection path and prove marker + exit status are captured.
3. Run the exact first P2-001 regression invocation with instrumentation that records:
   - wrapper PID/PPID/PGID/session;
   - child test PID(s);
   - start/end timestamps;
   - stdout and stderr separately;
   - `EXIT`/signal trap output where possible;
   - final shell exit status.
4. If it disappears again, immediately inspect for:
   - OOM kill/kernel kill evidence;
   - process/session termination signal;
   - shell `errexit`/pipeline behavior;
   - parent tmux/session teardown;
   - package manager spawning/detaching behavior;
   - file descriptor/redirection failure;
   - resource/ulimit exhaustion.
5. State one evidence-backed root cause, or if root cause is not yet proven, the narrowest remaining hypothesis with captured facts.
6. If a corrected invocation is proven reliable, run only the same P2-001 regression once with that invocation and report its real result + exit status. Do not proceed to the rest of the regression matrix in this diagnostic shot.

## STOP

Report:
- exact root cause or narrowest proven diagnosis;
- diagnostic commands/facts;
- whether a reliable invocation was established;
- if established, exact P2-001 regression result and exit status;
- confirm no business/test-source changes.

Then STOP. Do not create P2-002 evidence and do not resume full verification in this shot.