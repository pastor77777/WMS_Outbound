# P2-003 — Runtime Recovery Override

## Owner authorization

Owner explicitly authorizes one additional narrow recovery override after the prior STOPs. This is a **Testing-only** recovery. It may perform aggressive cleanup of generated build/cache artifacts and may make the minimum permanent build-tooling fix required to stop the repeated Mercato runtime failure.

Do not touch Demo or Prod. Do not reset/recreate P2-003 business work from P2-002. Preserve the existing local P2-003 candidate commits and canonical Testing DB state.

## Known state to preserve

Prior local candidate commits reported by executor:

- Mercato `94577b148bfb39b70de4ea2ce8457617b748e9c1`
- Scanner `862cd6fb5bffa86649e2329b5ee4a6d98dc2f06e`

Canonical Testing DB already has P2-003 schema migration `Migration20260905150000_wms_crossdock_execution` applied. Therefore do not roll repositories back to P2-002 or attempt destructive DB rollback.

Existing green evidence to retain unless invalidated:

- P2-003 dedicated PostgreSQL: `8/8` PASS, but still map all 18 substantive required behaviors explicitly;
- P2-002 PostgreSQL regression: `22/22` PASS;
- Mercato typecheck PASS;
- Scanner web export PASS.

Read `P2-003_EXECUTION.md`, `P2-003_CLOSEOUT_RUNTIME.md`, `.ai/TESTING.md`, and `.ai/OPERATIONS.md` first.

## Supervisor-proven runtime/build facts — 2026-09-05

Owner ran the canonical host-side durable build probe after the Testing reset. This is now proven and must be treated as current environment evidence, not re-diagnosed from scratch.

Observed/proven:

- Mercato local HEAD before the probe remained `94577b148bfb39b70de4ea2ce8457617b748e9c1` with the P2-003 business candidate preserved;
- local branch name was still `outbound/p2-002`; do **not** reset because of the stale branch name — preserve the HEAD and create/move the intended `outbound/p2-003` branch pointer from the existing candidate state before final push;
- `turbo.json` is dirty with the intended `.mercato/next/**` tracking correction;
- `yarn install --immutable` completed successfully and did not repair the `cross-env` issue, proving dependency installation was not corrupt;
- `apps/mercato/package.json` runs `cross-env NODE_OPTIONS=--max-old-space-size=8192 next build` but that workspace does not itself declare `cross-env`; root `package.json` does declare `cross-env`;
- direct workspace `cross-env` execution therefore fails under Yarn 4 workspace isolation even after a clean immutable install;
- the decisive direct Next build, bypassing that wrapper, succeeded through the durable systemd probe;
- fresh non-empty production manifests now exist:
  - `apps/mercato/.mercato/next/routes-manifest.json` — mtime `2026-09-05 16:25:31 UTC`;
  - `apps/mercato/.mercato/next/required-server-files.json` — mtime `2026-09-05 16:25:26 UTC`;
- probe result: `BUILD_OK`;
- no OOM/kernel kill was observed during the successful direct Next build.

The previously repeated “build disappears before manifests” symptom is therefore classified as build-tooling/runtime plumbing, not a P2-003 business-code failure.

## Confirmed build-tooling defects to fix permanently

Two independent defects are now confirmed:

1. **Turbo output tracking mismatch**
   - Next app is configured with `distDir: '.mercato/next'`;
   - generic Turbo build outputs tracked `.next/**` rather than the real `.mercato/next/**` output;
   - this can produce false cache success / missing production output.

2. **Workspace build dependency mismatch**
   - `apps/mercato/package.json` uses `cross-env` in its `build` script;
   - `apps/mercato` does not declare `cross-env` itself;
   - root has `cross-env`, but Yarn 4 workspace execution does not make that undeclared binary reliably available to the app workspace;
   - `yarn install --immutable` completing successfully while workspace `cross-env` is still missing proves this is a manifest/build-script defect, not a broken install.

Minimum permanent correction is authorized on the P2-003 Mercato branch:

- keep the already-prepared Turbo `.mercato/next/**` output tracking fix;
- make the app build self-contained by either:
  - declaring `cross-env` in `apps/mercato` devDependencies at the same compatible version used by the repo and updating the lockfile normally; **preferred if cross-platform behavior is intentionally required**, or
  - replacing the app build wrapper with an equivalent repository-approved direct invocation that does not depend on an undeclared binary;
- do not invent a broader package-manager redesign.

After the permanent correction, prove the native app/repository build path works without relying on the supervisor probe workaround.

## Recovery / closeout sequence from the current proven artifact

### 1. Preserve candidate state and establish correct branch names

- confirm Mercato HEAD `94577b148bfb39b70de4ea2ce8457617b748e9c1` and Scanner candidate `862cd6fb5bffa86649e2329b5ee4a6d98dc2f06e` still exist locally;
- preserve all candidate work;
- do not reset/rebase-away/reconstruct from P2-002;
- if Mercato is still on stale branch name `outbound/p2-002`, create/switch the intended `outbound/p2-003` branch at the existing candidate HEAD before committing tooling changes;
- do the analogous branch normalization for Scanner only if needed, without changing its candidate commit contents.

### 2. Use the already-proven production artifact first

Do **not** delete `.mercato/next` and do not rebuild before the first runtime proof.

The current artifact is freshly proven `BUILD_OK`. Immediately:

- confirm both required manifests are still non-empty;
- start/restart `mercato-localhost.service`;
- poll readiness up to 60 seconds;
- require `active (running)`;
- require canonical systemd MainPID to own/listen on port 3009;
- require `/login` HTTP 200;
- record build identity/revision provenance;
- prove new P2-003 routes are non-404.

If startup fails with valid manifests present, diagnose only the concrete service/journal error. Do not rebuild blindly.

### 3. Commit the permanent build-tooling corrections

Once runtime proves the current artifact is valid:

- commit the `turbo.json` output correction;
- fix the app workspace `cross-env` build dependency mismatch using the minimum coherent change described above;
- update lockfile only if the chosen package-manifest fix requires it;
- keep these changes strictly build-tooling/runtime plumbing.

### 4. Prove the final native build contract

After the permanent tooling fix is committed locally:

- stop Mercato cleanly;
- remove only generated production output/cache needed for a decisive rebuild;
- run the normal repository-native production build path from the final local code line;
- require non-empty `.mercato/next/routes-manifest.json` and `required-server-files.json`;
- prove a second run/cache path does not lose those outputs;
- restart `mercato-localhost.service` from that **final** build;
- again require active/running, canonical MainPID on 3009, `/login` 200, route presence and final build identity.

The decisive Playwright run must use this final proven runtime, not the earlier supervisor probe artifact if tooling changes have since altered the code line.

### 5. Finish P2-003 acceptance work

Once final Mercato runtime is healthy:

- restart/rebuild canonical Scanner runtime from the final Scanner candidate;
- prove canonical `scanner-testing.service` and port 8081 against the intended backend;
- run Journey A real Scanner 1:1, zero route mocks;
- run Journey B real Scanner n:n + physical-full continuation, zero route mocks;
- reconcile persisted quantities/TUs/tasks;
- inspect the dedicated `8/8` suite and explicitly map all 18 required substantive behaviors;
- add only genuinely missing dedicated coverage; do not inflate count mechanically;
- rerun P2-002 regression `22/22` after final code/build-tool changes;
- rerun P1-008 identity regression if identity/TUSetup/Sequence behavior is touched by any final code/test adjustment;
- run any other relevance-based regression only if its primitive was actually touched.

### 6. Push and evidence

Only after all final runtime/tests/Playwright are green:

- push final Mercato `outbound/p2-003` branch;
- push final Scanner `outbound/p2-003` branch;
- verify remote SHAs equal the tested local finals;
- create `05_EVIDENCE/P2-003_EVIDENCE.md` with exact final SHAs, migration provenance, 18-behavior mapping, regressions, runtime proof, Playwright A/B and persisted reconciliation;
- evidence must not self-declare `FINAL PASS`, `Owner Accepted` or `HUMAN VERIFIED`;
- end with clean Mercato and Scanner worktrees.

## Scope boundary

No P2-004/P2-005/P2-006/P3/P4, Return Receipt, Demo, Prod, or unrelated product redesign.

The authorized aggression applies only to Testing runtime/build recovery and the minimum build-tooling correction required to make the canonical Testing app reproducibly build and start.

## Final report

Report only:

- final pushed Mercato SHA;
- final pushed Scanner SHA;
- exact build-tooling root causes and permanent fixes;
- dedicated P2-003 result/count + explicit 18-behavior mapping status;
- P2-002 regression result;
- relevant identity/other reruns;
- Mercato final runtime proof (`active`, MainPID/3009 owner, `/login` 200, routes, build identity);
- Scanner final runtime proof;
- Journey A result;
- Journey B result;
- persisted reconciliation result;
- WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty;
- if STOPped: one exact remaining blocker.

Then STOP.
