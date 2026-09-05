# P2-006 — Codex continuation after Antigravity quota stop

Continue the SAME P2-006 item from the current canonical VPS state. Antigravity stopped mid-work because of quota. Current VPS state is authoritative.

## First action: inspect, do not mutate

Before any checkout/reset/clean/pull in Mercato or Scanner, record:

- current HEAD, branch, `git status --short`, last 10 commits;
- diff vs accepted P2-005 Mercato base `069f02d4c5c9b345b688b838eb685be02206afbd`;
- diff vs accepted Scanner base `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` if Scanner is dirty;
- untracked files, especially P2-006 tests/specs/migrations;
- whether `RESET_OK` from the P2-006 new-item start is already present in shell/log/handoff evidence if available.

Do NOT run another deep reset. Do NOT reset/checkout/rebase/clean product repos. Do NOT reconstruct from P2-005. Preserve all local Antigravity work.

## Scope discipline

The authoritative full contract remains `06_AGENT_GUIDES/P2-006_EXECUTION.md`.

P2-006 is a compatibility/join into the already accepted common downstream pipeline. Do not broaden into a general Outbound cleanup.

Accepted regression tests are evidence, not targets to rewrite until green. If an accepted P1/P2 regression fails:

1. first assume product regression caused by P2-006 and diagnose the product code/change;
2. modify an existing accepted test only if its fixture or expectation is genuinely invalidated by a documented, architect-authorized P2-006 common-model change;
3. if such a test change is necessary, keep it minimal and record exactly why the old fixture/expectation became invalid;
4. never weaken assertions, skip cases, replace real behavior with mocks, or mass-edit historical tests simply to make the suite pass.

Do not run broad unrelated suites while code is still changing. Run the smallest failing/affected test first. Run the full mandatory P2-006 regression matrix only after final product code is stable.

## Continuation order

1. Inspect and preserve current Antigravity local work.
2. Identify exactly what P2-006 implementation/tests already exist and the first current failing test or unfinished behavior.
3. Continue from that point with the minimum product/test change.
4. Keep Scanner frozen unless current local Scanner changes are demonstrably required by a real P2-006 defect; otherwise preserve/revert only with explicit evidence and without destructive loss.
5. Complete all 20 P2-006 substantive behaviors from the main guide.
6. Then run final mandatory regressions from the main guide: P2-005, P1-011 through P1-016, P2-002, and P2-003/P2-004 only when their touched surfaces require them.
7. Run repository-native Mercato generate/build/typecheck/build-app only after final code.
8. Prove canonical Mercato runtime and real zero-route-mock Mercato Playwright journeys.
9. Push final `outbound/p2-006` only after green; keep Scanner at accepted P2-005 SHA unless genuinely changed and proven.
10. Write `05_EVIDENCE/P2-006_EVIDENCE.md` with exact final SHAs, test counts, 20/20 behavior mapping, runtime proof, zero-mock UI proof, clean worktrees and ancestry.
11. Evidence must not self-declare FINAL PASS / Owner Accepted / Human Verified.
12. STOP for supervisor verification. Do not start P3-001.

If stopped, report one exact blocker and preserve the current local state.