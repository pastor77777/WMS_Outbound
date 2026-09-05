# P2-006 — Canonical Mercato runtime bind diagnostic

Continue the SAME P2-006 item from the preserved current VPS state.

Current checkpoint to preserve:

- durable P2-006 typecheck PASSED;
- durable fresh Mercato build PASSED with fresh non-empty production manifests;
- `mercato-localhost.service` was restarted and reported active with MainPID cwd in the canonical Mercato checkout;
- immediately after restart, port 3009 had no listener and `/login` returned HTTP 000;
- P2-006 backend matrix is already green and must not be rerun unless later product code changes;
- current P2-006 backend + UI spec work remains uncommitted and must be preserved;
- Scanner remains frozen.

This is a runtime-only diagnostic/continuation. Do NOT deep-reset, rebuild, re-run backend suites, edit business code, clean worktrees, rebase, or create a duplicate service.

## Step 1 — inspect the existing service before any restart

Capture all of the following from the current service instance:

```bash
sudo systemctl show mercato-localhost.service --no-pager \
  -p ActiveState -p SubState -p MainPID -p ExecMainPID -p ExecMainCode -p ExecMainStatus -p Result \
  -p NRestarts -p ExecStart -p WorkingDirectory -p EnvironmentFiles

sudo systemctl status mercato-localhost.service --no-pager -l || true
sudo systemctl cat mercato-localhost.service
sudo journalctl -u mercato-localhost.service --since '-15 minutes' --no-pager -n 250

MPID="$(sudo systemctl show mercato-localhost.service -p MainPID --value)"
echo "MainPID=$MPID"
if [[ -n "$MPID" && "$MPID" != "0" && -d "/proc/$MPID" ]]; then
  sudo readlink -f "/proc/$MPID/cwd" || true
  sudo tr '\0' ' ' < "/proc/$MPID/cmdline"; echo
  ps -o pid,ppid,stat,etime,%cpu,%mem,rss,vsz,cmd -p "$MPID" || true
  command -v pstree >/dev/null && pstree -ap "$MPID" || true
  pgrep -P "$MPID" -a || true
fi

sudo ss -ltnp 'sport = :3009' || true
curl -sS -o /dev/null -w 'HTTP=%{http_code}\n' --max-time 5 http://127.0.0.1:3009/login || true
```

Do not print service environment values or secrets.

## Step 2 — bounded observation, no duplicate restart

If the service is still `active` and has a live MainPID but port 3009 is not listening, observe the SAME instance for up to 90 seconds:

```bash
for i in $(seq 1 18); do
  date -Is
  sudo systemctl show mercato-localhost.service --no-pager -p ActiveState -p SubState -p MainPID -p NRestarts
  sudo ss -H -ltnp 'sport = :3009' || true
  curl -sS -o /dev/null -w 'HTTP=%{http_code}\n' --max-time 3 http://127.0.0.1:3009/login || true
  sleep 5
done
```

Then capture the last 150 service journal lines again.

### If port 3009 appears and `/login` becomes HTTP 200

Treat the prior immediate probe as a startup-timing false blocker. Do NOT restart again.

Continue directly with the existing P2-006 durable Playwright gate against this canonical runtime. If Playwright passes, finish the normal P2-006 closeout: commit/push exact final Mercato SHA, keep Scanner frozen, write evidence, STOP for supervisor verification.

### If the service becomes failed/inactive or MainPID changes/restarts

Do not restart repeatedly. Record:

- final `systemctl show/status`;
- `NRestarts`;
- last 250 journal lines;
- kernel OOM evidence from:

```bash
sudo journalctl -k --since '-30 minutes' --no-pager \
  | grep -Ei 'out of memory|oom-kill|killed process|memory cgroup|invoked oom-killer' \
  | tail -n 100 || true
```

STOP and report the exact first causal runtime error.

### If the service stays active with the same MainPID for 90 seconds but never binds 3009

This is a real runtime-startup blocker. Do NOT change P2-006 business code merely to make the port appear.

Use the service journal/process tree to identify the exact startup stage/error. Compare the actual unit `ExecStart`, working directory and canonical build output location with the accepted runtime contract. If the cause is a runtime/unit/helper configuration defect, fix only that runtime plumbing and prove it independently; do not alter business behavior or tests.

If the exact cause is not unambiguous, STOP with:

- unit `ExecStart`/WorkingDirectory summary;
- MainPID command/cwd and process tree;
- whether MainPID was stable;
- `NRestarts`;
- port-3009 observations;
- first relevant journal error and final 80 journal lines.

Do not run Playwright until canonical `/login` is HTTP 200. Do not start P3-001.
