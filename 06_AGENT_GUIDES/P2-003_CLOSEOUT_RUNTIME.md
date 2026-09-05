# P2-003 — Closeout after runtime/readiness stop

## Purpose

This is a **bounded closeout/remediation guide for the already-started P2-003 run only**.

Do not restart P2-003 from scratch. Do not start P2-004 or later work.

The prior executor report stated:

- local Mercato candidate commit: `94577b148bfb39b70de4ea2ce8457617b748e9c1`;
- local Scanner candidate commit: `862cd6fb5bffa86649e2329b5ee4a6d98dc2f06e`;
- P2-003 PostgreSQL suite: `8/8` PASS;
- P2-002 PostgreSQL regression: `22/22` PASS;
- Mercato typecheck PASS;
- Scanner web export PASS;
- final runtime, Playwright, evidence and push incomplete because Mercato port `3009` was not yet accepting connections after canonical service restart.

Those two product SHAs are **not yet present on GitHub** at the time this guide was written. Treat them as local candidate commits, not accepted/pushed truth.

Frozen accepted bases remain:

- Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`;
- Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`;
- P2-002 evidence `2074e2541b7c28cf3c6031cb48ad68901333625a`.

Read and obey `P2-003_EXECUTION.md`, `.ai/TESTING.md` and `.ai/OPERATIONS.md`. This guide only tightens the closeout/runtime sequence; it does not change business scope.

## Independent supervisor findings — canonical DB is already P2-003 schema-forward

Supervisor independently checked canonical Supabase Testing `yzonugcenguvmojwiihb` after the stopped run:

- project status: `ACTIVE_HEALTHY`;
- `mikro_orm_migrations_wms_outbound` records `Migration20260905150000_wms_crossdock_execution` executed at `2026-09-05 15:21:21+00`;
- `wms_outbound_cross_dock_pick_tasks` has `confirmed_qty numeric` and `active_target_tu_id uuid`;
- `wms_outbound_cross_dock_placements` exists with the expected P2-003 placement/correlation columns;
- indexes include organization/tenant/idempotency unique protection and organization/tenant/task lookup;
- after the reported suite, there were no residual placement rows and no recently updated fixture task rows from that run.

Therefore the canonical Testing DB is already **schema-forward relative to the pushed P2-002 product branches**.

Consequences:

1. Do **not** reset/reconstruct the product repos back to P2-002 as a normal continuation strategy.
2. Preserve and recover the reported local candidate commits first; their migration source must remain aligned with the already-applied canonical migration.
3. Do not run a DOWN/downgrade or manually delete P2-003 schema to make Git and DB look aligned.
4. Do not re-apply or invent DDL manually. Verify the local candidate contains the migration matching the applied migration name/semantics.
5. If the local Mercato candidate/migration source is missing, STOP and report this as a source-vs-canonical-DB provenance blocker before any reconstruction.

## 1. Preserve and inspect the existing local candidate work first

Before any reset, rebase, pull-with-reset, rebuild or new implementation edit:

1. In canonical `/home/ubuntu/git/Devaxonic-mercato` and `/home/ubuntu/git/Devaxonic-scanner`, record:
   - `git status --short`;
   - current branch;
   - `git rev-parse HEAD`;
   - `git show --stat --oneline HEAD`.
2. Confirm whether Mercato HEAD is exactly `94577b148bfb39b70de4ea2ce8457617b748e9c1` and Scanner HEAD exactly `862cd6fb5bffa86649e2329b5ee4a6d98dc2f06e`.
3. Prove each candidate is a descendant of its frozen accepted base using merge-base/ancestor checks.
4. Review the full candidate diffs from frozen base to local HEAD before changing anything.
5. Verify no P2-004/P2-005/P2-006/P3/P4/Return Receipt/Demo/Prod leakage.
6. In Mercato, verify the candidate contains `Migration20260905150000_wms_crossdock_execution` (or exact current source identity that produced the recorded migration) and that its DDL matches the already-applied canonical DB shape.
7. Do **not** discard the local candidate commits merely because they are not yet pushed.

If either reported local SHA is missing locally, STOP and report that exact discrepancy before reconstructing work.

## 2. Verify the dedicated suite substantively, not by test count alone

`8/8` is not automatically a failure and not automatically sufficient.

Inspect the actual P2-003 PostgreSQL suite and map it explicitly to all minimum substantive coverage bullets in `P2-003_EXECUTION.md`:

- lazy no-target-before-placement;
- atomic task/line/order start;
- lazy target creation;
- 1:1 valid-GS1 inheritance;
- 1:1 invalid/non-GS1 new identity;
- n:n never inherits source identity;
- exact decimal source/task/SKU placement correlation;
- replay-safe placement;
- no over-confirm above remaining plannedQty;
- wrong task/source/SKU/tenant/warehouse rejection with no committed side effect;
- physical-full seal with task still IN_PROGRESS;
- new target continuation under same task;
- exact multi-target quantity reconciliation;
- replay-safe completion with `confirmedQty <= plannedQty`;
- sibling task independence;
- no Allocation;
- P2-002 assignment/one-active-task non-regression;
- no P2-004 exception effects on normal OK path.

Eight tests are acceptable only if the code/evidence genuinely covers all required behaviors without trivial assertion relabeling. If coverage is missing, add only the missing P2-003 test/implementation slice and rerun invalidated gates.

## 3. Mercato runtime ownership — eliminate terminal-owned/ad-hoc servers

Canonical Testing Mercato runtime is `mercato-localhost.service` on `localhost:3009`.

Before Playwright:

1. Inspect `systemctl show mercato-localhost.service -p MainPID -p ActiveState -p SubState`.
2. Inspect the listener/process on port 3009 (`ss`/`ps` or equivalent).
3. The process serving 3009 must correspond to the canonical systemd service.
4. Do **not** run or leave behind an ad-hoc terminal-owned `yarn dev`, `next dev`, `next start`, `yarn start`, custom webServer or other second Mercato server.
5. If a noncanonical background process from this executor is clearly identified, terminate that process before continuing. Do not kill unknown/unrelated processes.
6. If no process currently owns 3009, diagnose the canonical service rather than starting a replacement server manually.

A background terminal in the executor UI is not runtime provenance. PID/process ownership + systemd MainPID + port ownership are the provenance check.

## 4. Build integrity before restart

The repository-native root command remains `yarn build`; at the accepted base it performs `build:packages -> generate -> build:packages -> build:app`.

Before restarting Mercato against a freshly built candidate, verify the generated Next runtime is complete. In particular, check the canonical build output used by this deployment for non-empty routing/server manifests, including:

- `apps/mercato/.mercato/next/routes-manifest.json`;
- `apps/mercato/.mercato/next/required-server-files.json`.

If the build output path differs on the exact candidate SHA, determine the real configured path from current repository/runtime configuration; do not guess.

If either required manifest is missing/empty or the prior build is demonstrably partial/stale:

1. remove only the generated Next build directory for the canonical Mercato app;
2. rerun the repository-native full `yarn build` from the canonical Mercato repository;
3. verify the required manifests are present/non-empty;
4. only then restart `mercato-localhost.service`.

Do not edit product source to repair an incomplete generated build.

## 5. Readiness is a bounded poll, not one immediate curl

After `sudo systemctl restart mercato-localhost.service`, do **not** fail merely because the first immediate connection attempt is refused.

Use a bounded readiness window of up to **60 seconds**, polling approximately every 2 seconds. Success requires all of:

- `mercato-localhost.service` is `active (running)` and not `activating (auto-restart)`/failed;
- port 3009 is listening from the canonical service process;
- `GET http://localhost:3009/login` returns HTTP `200`.

As soon as these are true, proceed; do not wait out the full window.

If readiness is still not achieved within the bounded window, collect before any further action:

- `systemctl status mercato-localhost.service --no-pager`;
- `systemctl show ... MainPID/ActiveState/SubState`;
- port-3009 listener/process ownership;
- `journalctl -u mercato-localhost.service -n 100 --no-pager`;
- whether the required build manifests exist/non-empty.

Classify the failure before retrying:

- still booting / slow start;
- crash-loop;
- incomplete/stale build;
- port conflict / wrong process owner;
- application startup error;
- other concrete journal error.

Do not perform repeated blind restarts. A readiness recheck after the same restart is diagnosis, not a new product implementation attempt.

Historical DevAxonic runtime evidence already records that systemd process launch may precede Next.js port readiness and that partial `.mercato/next` output can crash-loop until a clean generated rebuild is performed. Apply that known runtime behavior here instead of treating an immediate connection refusal as a product blocker.

## 6. Route/runtime provenance before Scanner Playwright

Once `/login` is HTTP 200:

1. prove Mercato local HEAD is the intended candidate SHA;
2. record non-secret build identity available from the generated runtime;
3. probe every new P2-003 API route so route presence/non-404 is known before UI assertions;
4. if API returns 404/5xx, diagnose route/build/runtime/API first — do not alter Scanner selectors to hide it.

For Scanner:

1. prove local HEAD is the intended Scanner candidate SHA;
2. restart only canonical `scanner-testing.service` after final Scanner source changes;
3. let its canonical fresh web export complete;
4. use a bounded readiness check for port 8081 rather than reusing a stale Playwright server;
5. prove Scanner backend wiring targets canonical Testing Mercato.

## 7. Final P2-003 acceptance execution

Only after exact runtime provenance is green:

1. run Journey A — real Scanner 1:1 normal path, zero route mocks;
2. run Journey B — real Scanner n:n + physical-full continuation, zero route mocks;
3. reconcile persisted task/TU/content quantities against rendered outcomes;
4. rerun any backend/regression gate invalidated by a product/test change made during closeout;
5. do not implement any future-scope exception path.

## 8. Push only after candidate verification

After candidate diff + tests + exact runtime + both Playwright journeys are green:

- push the verified Mercato candidate on the P2-003 outbound branch convention (`outbound/p2-003`);
- push the verified Scanner candidate on `outbound/p2-003`;
- verify the pushed remote branch heads equal the exact tested local SHAs.

If a new product commit becomes necessary during closeout, the pushed SHA must be that final rerun-tested commit, not the earlier reported candidate.

Do not push these Outbound candidate commits to product `main` merely to make them visible.

## 9. Evidence and cleanliness

Only after all final gates are green create/update:

`WMS_Outbound/05_EVIDENCE/P2-003_EVIDENCE.md`

It must include everything required by `P2-003_EXECUTION.md`, plus:

- local-candidate-to-pushed-remote provenance;
- explicit mapping showing how the dedicated suite covers all 18 substantive minimum behaviors even if the suite has fewer than 18 test cases;
- canonical DB migration provenance for `Migration20260905150000_wms_crossdock_execution`;
- Mercato systemd MainPID/port/runtime readiness proof;
- build-manifest integrity and any clean-rebuild recovery performed;
- explicit statement that no terminal-owned/ad-hoc Mercato server was used for decisive Playwright evidence.

Then verify Mercato and Scanner worktrees clean.

Evidence must not self-declare Supervisor PASS, Owner Accepted or Human Verified.

## Hard STOP

If the bounded runtime diagnostic produces a concrete non-readiness error that cannot be repaired by the one documented generated-build/runtime recovery path without changing unrelated infrastructure, STOP with the exact journal/status classification.

Do not broaden into infrastructure redesign, Demo/Prod, another executor, another application server or P2-004+.

## Final report

Report only:

- final pushed Mercato SHA;
- final pushed Scanner SHA;
- dedicated P2-003 result/count + confirmation all 18 substantive behaviors are mapped;
- P2-002 `22/22` regression result;
- any other relevant regressions actually rerun;
- canonical DB migration/source match;
- Mercato runtime readiness (`active`, MainPID/port owner, `/login` HTTP 200, build identity);
- Playwright Journey A result;
- Playwright Journey B result;
- WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty;
- if STOPped: one exact classified blocker.

Then STOP.
