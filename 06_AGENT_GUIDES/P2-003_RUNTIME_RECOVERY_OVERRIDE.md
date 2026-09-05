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

## Root-cause hypothesis to verify first

The accepted Mercato app config uses:

`apps/mercato/next.config.ts -> distDir: '.mercato/next'`

but current root `turbo.json` build outputs track:

`["dist/**", ".next/**", "!.next/cache/**"]`

This mismatch can let Turbo report a cached/successful build without restoring the real production Next output under `.mercato/next`.

Treat this as the primary hypothesis. Verify it from the exact local candidate before inventing another workaround.

## Recovery sequence

### 1. Preserve local candidate work

Before cleanup:

- confirm Mercato and Scanner local HEADs and clean/dirty status;
- confirm the reported candidate commits still exist locally and descend from frozen P2-002 bases;
- do not reset, rebase-away, cherry-pick-away, or reconstruct the P2-003 product work.

### 2. Stop serving a broken runtime

Stop `mercato-localhost.service` before deleting generated runtime artifacts.

Inspect port 3009 and process ownership. If a clearly identified ad-hoc/background Mercato process from the executor owns or competes for 3009, terminate that process. Do not kill unrelated/unknown processes.

### 3. Fix the Turbo output mismatch permanently if still present

If exact local `next.config.ts` still uses `.mercato/next` and exact local `turbo.json` still tracks only `.next/**` for the generic `build` task, fix the build outputs so the app production dist is tracked correctly.

Minimum intended correction:

- include `.mercato/next/**` as a build output;
- exclude `.mercato/next/cache/**` as cache noise;
- do not remove `dist/**` needed by packages;
- remove/retain legacy `.next/**` only according to actual repository consumers discovered locally; do not guess.

This is build tooling/runtime plumbing only. Do not change P2-003 business behavior while making this correction.

Commit this tooling fix on the P2-003 Mercato branch only if a source change is needed.

### 4. Aggressive generated/cache cleanup on Testing

It is authorized to delete generated/cache artifacts that can be recreated, including as applicable:

- `apps/mercato/.mercato/next`
- stale `apps/mercato/.next`
- repo/app Turbo cache directories such as `.turbo`

Do not delete source, local P2-003 commits, Testing DB data, credential files, or unrelated persistent application data.

If dependency installation itself is demonstrably corrupt, `yarn install --immutable` is authorized. Do not perform broader host/package-manager surgery without a concrete error requiring it.

### 5. Bypass Turbo for the decisive production build

For this recovery, do not trust a root `yarn build` cache hit as decisive evidence.

From canonical Mercato repo:

1. run `yarn build:packages`;
2. run `yarn generate`;
3. run `yarn build:packages` again;
4. enter `apps/mercato`;
5. run the app's production build directly: `npx cross-env NODE_OPTIONS=--max-old-space-size=8192 next build`.

This direct app build must actually create the configured `.mercato/next` output.

Before restart require non-empty:

- `apps/mercato/.mercato/next/routes-manifest.json`
- `apps/mercato/.mercato/next/required-server-files.json`

Also record the generated build identity available from that output.

If direct `next build` fails, report the actual build error and fix that exact cause. Do not convert a failed build into repeated service restarts.

### 6. Restart canonical service only after valid build exists

Only after both required manifests exist and are non-empty:

- restart `mercato-localhost.service`;
- poll readiness up to 60 seconds;
- require `active (running)`, canonical MainPID owning/listening on 3009, and `/login` HTTP 200.

If it fails, collect status/MainPID/port owner/journal and fix the exact startup error. One generated-build/cache recovery path is already authorized; do not keep blind-restarting.

### 7. Prove the permanent build fix

If `turbo.json` was corrected, after obtaining one healthy direct build, also prove that the repository-native cached build contract is no longer broken:

- remove only the generated production Next output again;
- run the normal repository-native build path in a controlled way;
- prove `.mercato/next` and required manifests are restored/created correctly.

Do this without taking the finally restored service down longer than necessary. The decisive Playwright runtime must end on a known valid final build.

### 8. Finish P2-003, do not stop at runtime recovery

Once Mercato is healthy:

- verify all new P2-003 routes are non-404;
- restart/rebuild canonical Scanner runtime from final Scanner candidate;
- run Journey A real Scanner 1:1, zero route mocks;
- run Journey B real Scanner n:n + physical-full continuation, zero route mocks;
- reconcile persisted quantities/TUs/tasks;
- inspect the `8/8` dedicated suite and explicitly map all 18 required substantive behaviors; add only genuinely missing coverage;
- rerun any gate invalidated by a code/test/build-tooling change;
- push final Mercato `outbound/p2-003` and Scanner `outbound/p2-003` branch heads;
- create `05_EVIDENCE/P2-003_EVIDENCE.md` with exact final SHAs, runtime/build provenance, suite mapping, Playwright and reconciliation;
- end with clean Mercato and Scanner worktrees.

## Scope boundary

No P2-004/P2-005/P2-006/P3/P4, Return Receipt, Demo, Prod, or unrelated product redesign.

The authorized aggression applies to Testing runtime/build recovery and the minimum build-tooling correction required to make the canonical Testing app reproducibly start.

## Final report

Report only:

- final pushed Mercato SHA;
- final pushed Scanner SHA;
- exact build-tooling root cause and fix;
- dedicated P2-003 result/count + 18-behavior mapping status;
- P2-002 regression result;
- relevant reruns;
- Mercato runtime proof (`active`, MainPID/3009 owner, `/login` 200, build identity);
- Journey A result;
- Journey B result;
- WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty;
- if STOPped: one exact remaining blocker.

Then STOP.
