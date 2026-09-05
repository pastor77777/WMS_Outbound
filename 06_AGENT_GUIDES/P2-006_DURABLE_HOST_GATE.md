# P2-006 — durable host gate after silent executor terminations

Continue the SAME P2-006 item from the current canonical VPS state.

Preserve all current local Mercato work, including the uncommitted P2-006 backend compatibility change and `P2-006-crossdock-shipment-downstream-ui.spec.ts`. Scanner remains frozen at accepted P2-005 SHA.

Do **not** deep-reset, checkout, clean, rebase, reconstruct, rerun the already-green backend matrix, or edit business code because a heavy command disappears from the executor terminal.

## Known green backend checkpoint

Preserve as already proven unless a later product change invalidates it:

- P2-006 dedicated PostgreSQL 20/20;
- P2-005 19/19;
- P1-011 18/18;
- P1-012 14/14;
- P1-013 15/15;
- P1-014 18/18;
- P1-015 21/21;
- P1-016 25/25;
- P2-002 22/22;
- P2-003 8/8;
- P2-004 16/16.

Current backend product scope remains the narrow crossdock-safe manifest settlement behavior: CROSSDOCK settlement advances normal business states without manufacturing a fake Allocation or decrementing standard warehouse Inventory.

## Exact blocker being resolved

Direct executor-terminal invocations of both:

- heap-allocated Mercato app typecheck; and
- the dedicated P2-006 Playwright spec

terminated without usable runner output/result/log content.

Treat this as host/executor process-lifecycle plumbing until durable evidence proves otherwise.

## Durable runner

First sync only the operational helper repo so the new runner is available:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
```

The helper is:

`/home/ubuntu/git/Devaxonic-WMS/scripts/p2-006-durable-gates.sh`

It launches heavy gates as transient systemd services outside the executor terminal lifecycle and records:

- service Result / ExecMainStatus;
- MemoryPeak / MemoryMax;
- journal output;
- recent kernel/cgroup OOM-kill signals;
- fresh Playwright JSON result proof.

If the Codex command itself disappears while the transient service is running, do **not** start another copy. Recover with:

```bash
bash /home/ubuntu/git/Devaxonic-WMS/scripts/p2-006-durable-gates.sh --status
bash /home/ubuntu/git/Devaxonic-WMS/scripts/p2-006-durable-gates.sh --logs
```

## Closeout order

### 1. Durable typecheck only

Run:

```bash
bash /home/ubuntu/git/Devaxonic-WMS/scripts/p2-006-durable-gates.sh --typecheck
```

If `TYPECHECK=OK`, continue.

If it reports `OOM_OR_SIGKILL`, `SIGTERM`, a nonzero exit, or another exact classification, STOP and report only that durable classification plus MemoryPeak and final relevant journal lines. Do not modify product code to address a host resource/process-lifecycle failure.

### 2. Fresh canonical build

Only after durable typecheck passes, run the already-established durable Mercato build probe:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
bash scripts/probe-mercato-testing-build.sh --run
```

If the caller disappears, recover with its existing `--status` / `--logs` actions rather than starting a second build.

Require `BUILD_OK` and fresh non-empty:

- `apps/mercato/.mercato/next/routes-manifest.json`;
- `apps/mercato/.mercato/next/required-server-files.json`.

### 3. Restart and prove exact canonical runtime

After `BUILD_OK`:

```bash
sudo systemctl restart mercato-localhost.service
```

Prove before Playwright:

- service active/running;
- MainPID cwd is `/home/ubuntu/git/Devaxonic-mercato`;
- port 3009 is owned by canonical Next/Mercato runtime;
- `/login` returns HTTP 200;
- fresh production manifests remain non-empty.

Do not claim UI proof against an older/stale runtime.

### 4. Durable P2-006 Playwright only

With the fresh canonical runtime serving the current local P2-006 source/build, run:

```bash
bash /home/ubuntu/git/Devaxonic-WMS/scripts/p2-006-durable-gates.sh --playwright
```

Require `PLAYWRIGHT=OK` plus a fresh non-empty `.ai/qa/test-results/results.json`.

The dedicated local spec must remain zero-route-mock/interception. DB use is allowed only for deterministic fixture setup/cleanup and persisted reconciliation; required product actions remain rendered Mercato UI + real backend behavior.

If Playwright produces a genuine assertion/product failure, fix only that concrete failure and rerun only P2-006 plus directly invalidated regression(s). Do not reopen the full already-green backend matrix unless product code actually changes.

If Playwright instead reports OOM/SIGKILL/process-plumbing classification, STOP with the durable host evidence; do not rewrite UI/business behavior to hide it.

### 5. Commit/push/evidence only after all durable gates are green

Then and only then:

- commit the minimum Mercato diff, including the dedicated P2-006 UI spec;
- push `outbound/p2-006`;
- verify linear ancestry from accepted P2-005 Mercato `069f02d4c5c9b345b688b838eb685be02206afbd`;
- keep Scanner frozen at `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- write `05_EVIDENCE/P2-006_EVIDENCE.md` with exact product SHA, already-green backend matrix, 20/20 behavior mapping, durable typecheck/build/runtime/Playwright proof, zero-mock statement, clean worktree and ancestry;
- evidence must not self-declare FINAL PASS / Owner Accepted / Human Verified;
- STOP for supervisor verification. Do not start P3-001.
