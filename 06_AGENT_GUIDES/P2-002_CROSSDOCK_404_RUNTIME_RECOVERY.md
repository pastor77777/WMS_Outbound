# P2-002 — Crossdock HTTP 404 canonical runtime recovery

## Purpose
Resolve the current P2-002 blocker directly: the real Scanner request to `POST /api/wms_outbound/cross-dock/request-task` returned HTTP 404 although the route source exists in final Mercato.

Continue in the SAME Codex session. Do not start P2-003.

## Frozen state entering this guide
- Mercato source SHA: `50b27fdd0c9b495ab612ce458bc90e65428ecb93` — clean, pushed.
- Scanner HEAD: `c5f65189c832ebdbd312ae0f5fc6930c74d43da0`.
- Scanner working tree is intentionally DIRTY only with the already-authorized current corrective changes from the immediately preceding guide. Preserve them exactly. Do not reset/stash/rebase them.
- P1-005 current-UI Scanner regression: PASS 1/1.
- P2-002 dedicated PostgreSQL: PASS 22/22.
- P2-001: PASS 10/10.
- Remaining Mercato regressions: PASS 5 suites / 41 tests.
- P2-002 Playwright current blocker: operator A real Crossdock request returned HTTP 404; B/blocked were not executed.
- WMS evidence does not yet exist.

Canonical Testing only:
- Mercato: `http://localhost:3009` via `mercato-localhost.service`.
- Scanner: `http://localhost:8081`.
- DB: remote Supabase Testing project `yzonugcenguvmojwiihb` via `/etc/mercato-localhost.env`.
- Local PostgreSQL is forbidden.

Read before acting:
- `/home/ubuntu/git/Devaxonic-WMS/.ai/TESTING.md`
- `/home/ubuntu/git/Devaxonic-WMS/.ai/OPERATIONS.md`

## Verified source facts — do not rediscover from scratch
At Mercato SHA `50b27fdd0c9b495ab612ce458bc90e65428ecb93` the source file exists:

`apps/mercato/src/modules/wms_outbound/api/cross-dock/request-task/route.ts`

It exports `POST` and, when reached without valid auth, returns a non-404 auth/input response. Therefore HTTP 404 from the real Scanner request means the currently served Testing runtime did not expose this route.

The existing STANDARD route `POST /api/wms_outbound/picking/request-task` is the control route and already works in the P1-005 rendered flow.

Root Mercato `yarn build` includes package build + `yarn generate` + application build. Do not use an app-only build that can leave generated route artifacts stale.

## 1. Diagnose the served runtime, not the Playwright selector
Do not edit Scanner or Playwright before completing this section.

Capture only non-secret facts:

1. Mercato worktree:
   - `git -C /home/ubuntu/git/Devaxonic-mercato rev-parse HEAD`
   - require exact `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
   - require clean worktree.

2. Canonical service identity:
   - `systemctl status mercato-localhost.service --no-pager`
   - `systemctl cat mercato-localhost.service`
   - `systemctl show mercato-localhost.service -p MainPID -p ActiveState -p SubState -p ExecStart -p WorkingDirectory`
   - for a non-zero MainPID, record `/proc/<pid>/cwd` and command line without environment values.
   - verify the service is actually the canonical Testing service on port 3009.

3. Source route presence:
   - prove the Crossdock route file exists in the exact Mercato worktree;
   - prove the existing STANDARD picking request-task route file exists too.

4. Runtime route probe, unauthenticated and with a valid UUID body. Do not print credentials/tokens:
   - POST `http://127.0.0.1:3009/api/wms_outbound/picking/request-task`
   - POST `http://127.0.0.1:3009/api/wms_outbound/cross-dock/request-task`
   - capture HTTP status + short non-secret body.

Expected classification:
- the control STANDARD route must be non-404;
- the Crossdock route must also be non-404 if the current runtime contains the route;
- `401`, `403`, `400`, or `422` without auth is acceptable route-presence proof; `404` is not.

5. Inspect generated/build routing artifacts in the canonical Mercato worktree without assuming a hardcoded manifest path:
   - locate generated API route registry/manifest and Next build manifests used by this app;
   - search them for both `picking/request-task` and `cross-dock/request-task`;
   - record only whether each route is present and relevant artifact timestamps/identity, not giant file contents.

Do not rerun Playwright yet.

## 2. Primary recovery — regenerate + rebuild exact Mercato SHA and restart canonical service
If source route exists at exact SHA, STANDARD runtime route is non-404, and Crossdock runtime route is 404 or absent from generated/build artifacts, classify this as stale generated/build/runtime routing.

Then, with Mercato still exactly `50b27fdd0c9b495ab612ce458bc90e65428ecb93` and clean:

1. use Node/Yarn versions already required by the repo/VPS;
2. source `/etc/mercato-localhost.env` in the same shell invocation where required by generation/build;
3. from `/home/ubuntu/git/Devaxonic-mercato`, run the repository-native FULL build:
   - `yarn build`
   - this intentionally includes `yarn generate`; do not substitute `yarn workspace @open-mercato/app build` alone;
4. require build exit 0;
5. restart only `mercato-localhost.service` through systemd;
6. require service active/running and port 3009 listening;
7. require `http://127.0.0.1:3009/login` HTTP 200;
8. repeat the unauthenticated route probes.

Hard gate after restart:
- STANDARD request-task: non-404;
- Crossdock request-task: non-404.

If Crossdock becomes non-404, do not change Mercato source code. Proceed to section 4.

## 3. If a clean full build still produces Crossdock 404
Only enter this section if section 2 completed successfully but the exact rebuilt/restarted runtime still returns 404 for Crossdock.

This is now a route-discovery/registration defect, not a browser-test defect.

Inspect the Open Mercato module API generation/discovery mechanism by comparing the working sibling:

`apps/mercato/src/modules/wms_outbound/api/picking/request-task/route.ts`

with:

`apps/mercato/src/modules/wms_outbound/api/cross-dock/request-task/route.ts`

and the generated route registry/build manifest.

Determine the exact reason the generator includes the first and excludes the second. Do not guess from folder naming alone.

Authorized correction if and only if the reason is proven:
- make the smallest Mercato route-registration/source-layout correction required for the generator to expose exactly `/api/wms_outbound/cross-dock/request-task`;
- preserve the endpoint contract and all P2-002 business semantics;
- no planner/service/schema/migration behavior changes;
- no alternate URL in Scanner to bypass the intended route;
- no proxy/mock/ad-hoc server.

Then:
1. commit/push the minimal Mercato correction on the existing P2-002 branch;
2. record new final Mercato SHA;
3. run full `yarn build` on that exact SHA;
4. restart `mercato-localhost.service`;
5. prove both STANDARD and Crossdock runtime route probes are non-404;
6. because Mercato SHA changed, rerun final-SHA-sensitive gates before UI closeout:
   - P2-002 dedicated PostgreSQL: 22/22;
   - P2-001: 10/10;
   - remaining previously-required Mercato regressions: same 5 suites / 41 tests or their exact current equivalent;
7. STOP immediately if any backend gate becomes red.

If the route-discovery cause cannot be proven after this one focused inspection, STOP and report the generated-registry difference and service/build facts. Do not edit Playwright again.

## 4. Finish Scanner gates against the verified exact runtime
Once the canonical runtime Crossdock probe is non-404:

1. preserve the already-authorized dirty Scanner corrective changes;
2. rebuild the Scanner web `dist` only as required so it contains the current working-tree correction and canonical Testing API base;
3. rerun P1-005 current-UI Playwright once — require 1/1 PASS;
4. rerun only `e2e/p2-002-crossdock-assignment.spec.ts`.

P2-002 Playwright must use zero route mocks and prove through real rendered flow:
- warehouse context resolved before Crossdock entry;
- operator A real POST Crossdock request returns HTTP 200 with expected task A;
- task number visible;
- source Inbound TU visible;
- planned quantity visible;
- assigned status visible using the current `StatusBadge` contract;
- no zone selector/requirement;
- operator B receives the other task, never A's task;
- blocked operator receives real active-warehouse-task rejection;
- DB ownership/status matches API/rendered result;
- no P2-003 sorting/scan/target-TU/completion action is available/exercised.

Required: PASS, exit 0.

If the verified runtime route is non-404 but the dedicated Playwright fails, STOP with the exact HTTP response and rendered/DB mismatch. Do not return to runtime guessing.

## 5. Commit Scanner corrective only after green
If both Scanner gates are green:
- inspect Scanner diff from `c5f65189c832ebdbd312ae0f5fc6930c74d43da0`;
- it may contain only the already-authorized final P2-002 harness synchronization and the one-line Crossdock `StatusBadge` prop correction;
- commit/push on the existing Scanner P2-002 branch;
- record final Scanner SHA;
- require clean worktree.

## 6. Canonical P2-002 evidence and STOP
Only after all gates are green, create/update exactly:

`05_EVIDENCE/P2-002_EVIDENCE.md`

Include:
- exact final Mercato SHA and whether runtime recovery was rebuild-only or required a minimal route-registration correction;
- exact final Scanner SHA and merge base;
- canonical Testing provenance `yzonugcenguvmojwiihb` without secrets;
- proof canonical `mercato-localhost.service` served the exact final Mercato build and Crossdock route was non-404 before Playwright;
- corrected CON-03 PASS;
- P2-001 10/10;
- P2-002 PostgreSQL 22/22;
- remaining Mercato 5 suites / 41 tests (rerun only if final Mercato SHA changed as required above);
- P1-005 current-UI 1/1 PASS;
- dedicated P2-002 Playwright PASS with A/B/blocked real request classifications and required rendered facts;
- zero route mocks;
- no Allocation for Crossdock semantics;
- no P2-003 scope leakage;
- clean final worktrees.

Do not claim Owner Accepted, HUMAN VERIFIED, FINAL PASS or Supervisor acceptance.

Commit/push only the canonical evidence file to WMS_Outbound/main.

Final response only:
- final Mercato SHA;
- runtime route recovery classification;
- final Scanner SHA;
- P1-005 result;
- P2-002 Playwright result;
- A/B/blocked request classifications;
- WMS evidence SHA;
- Mercato/Scanner cleanliness;
- PASS or STOP.
