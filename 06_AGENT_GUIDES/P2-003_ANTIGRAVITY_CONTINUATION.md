# P2-003 — Antigravity continuation after Codex time limit

## Purpose

Continue the already-implemented P2-003 work on the canonical VPS. Do **not** reconstruct P2-003 from accepted P2-002 and do **not** repeat the prior runtime-recovery investigation from scratch.

This is the same P2-003 item (22/37), so the mandatory deep Testing reset for a **new** catalog item does not apply here.

## Authoritative local context to preserve

Canonical repos:
- `/home/ubuntu/git/Devaxonic-WMS`
- `/home/ubuntu/git/Devaxonic-mercato`
- `/home/ubuntu/git/Devaxonic-scanner`

Frozen accepted bases:
- Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`

Previously reported clean local candidates before the final Codex continuation:
- Mercato `0f4ab9d0605fae7596c5c603a92af9f12d79a2e1`
- Scanner `a70c744b927fba620768e72740128cab8ae57b65`

Important: Codex continued after those hashes and hit its 5h limit. Therefore the **current local HEAD/worktree on the VPS is authoritative** and may be newer than these candidate hashes. Inspect exact HEAD/status first. Do not reset to the hashes above.

## Facts already established by supervisor

1. Canonical Testing DB already has `Migration20260905150000_wms_crossdock_execution` applied. No rollback/down/manual schema recreation.
2. Mercato host-side direct Next production build is proven healthy. A durable probe produced non-empty:
   - `apps/mercato/.mercato/next/routes-manifest.json`
   - `apps/mercato/.mercato/next/required-server-files.json`
3. Build-tooling defects already identified and addressed in local P2-003 work:
   - Turbo build outputs must include `.mercato/next/**` while excluding `.mercato/next/cache/**` and retaining required package `dist/**` outputs.
   - `apps/mercato/package.json` now declares `cross-env` because its workspace `build` script directly invokes it.
4. Mercato P2-003 implementation exists locally: execution route/service/entities/migration/dedicated PostgreSQL suite.
5. Scanner P2-003 implementation exists locally: execution API client and `CrossdockTaskScreen` execution controls.
6. Contrary to the earlier STOP report, the final VPS snapshot shows that Codex **already created** `e2e/p2-003-real-crossdock-execution.spec.ts` before its time limit.
7. That spec already contains both required real journeys and no Playwright route mocks were found:
   - Journey A: real 1:1 Scanner placement, valid GS1 identity inheritance, completion, persisted reconciliation.
   - Journey B: real n:n Scanner flow, physical-full sealing, same task continues onto a newly generated target TU, completion, persisted reconciliation.

Do not delete or rewrite that fixture wholesale unless an exact failing assertion proves it is wrong. First run it as-is.

## First actions — inspect, do not mutate

Record for Mercato and Scanner:
- `git rev-parse HEAD`
- current branch
- `git status --short`
- `git log --oneline --decorate -8`
- diff/name-status from frozen accepted base to current HEAD
- any uncommitted diff

Confirm the current local changes are still P2-003 plus the narrow authorized build-tooling fixes only. No P2-004+ scope.

If current work is uncommitted, preserve it. Do not checkout/reset/clean/rebase.

## Decisive closeout sequence

### 1. Runtime health

Use the existing final local product state. Do not start another broad runtime recovery.

Require:
- canonical `mercato-localhost.service` active;
- its MainPID owns/listens on port 3009;
- `http://localhost:3009/login` returns HTTP 200;
- required `.mercato/next` manifests are non-empty;
- canonical `scanner-testing.service` active on 8081 with a fresh export from the current Scanner state.

If a service is unhealthy, collect its exact status/journal/port owner and fix only that concrete cause.

### 2. Run the existing P2-003 real Scanner Playwright fixture first

Target existing file:
`/home/ubuntu/git/Devaxonic-scanner/e2e/p2-003-real-crossdock-execution.spec.ts`

Use the canonical Scanner runtime and real Mercato backend. Zero route mocks.

The fixture must execute through rendered Scanner UI controls and real HTTP calls. Direct PostgreSQL is allowed only for deterministic fixture setup, teardown, and persisted-state reconciliation.

Journey A must prove at minimum:
- task assigned through real Scanner flow;
- no target outbound TU exists before first placement (lazy target);
- first OK placement creates target;
- 1:1 valid GS1 source inherits TU number + SSCC;
- completion succeeds;
- task `COMPLETED`, exact confirmed qty, placement fact persisted;
- outbound line/order quantities/status reconcile.

Journey B must prove at minimum:
- n:n context forces generated target identity;
- first placement creates generated non-SSCC target;
- physical-full seals that target;
- task remains `IN_PROGRESS` after physical-full;
- next placement creates a different generated target;
- completion succeeds;
- exactly two target TUs and corresponding placement quantities reconcile;
- `confirmedQty <= plannedQty`;
- no outbound allocation count mutation.

If either journey fails, diagnose the exact first real failure and make the minimum P2-003 fix. Do not replace the journey with API-only proof and do not mock routes.

### 3. Dedicated/regression gates

After any change that can invalidate them, run:
- P2-003 dedicated PostgreSQL suite and explicitly verify/map all 18 required substantive behaviors from `P2-003_EXECUTION.md`;
- P2-002 mandatory PostgreSQL regression `22/22`;
- relevant P1-008 TU identity regression;
- Mercato typecheck/build gates affected by code/tooling changes;
- Scanner fresh web export after Scanner changes.

Eight dedicated test cases are acceptable only if the evidence explicitly maps all 18 required business behaviors to substantive assertions/coverage.

### 4. Final product commits and push

Only after all gates and both journeys pass:
- commit any remaining intended Mercato changes on `outbound/p2-003`;
- commit any remaining intended Scanner changes on `outbound/p2-003`;
- push both final branch heads;
- prove each final SHA descends from its frozen accepted base;
- leave both product worktrees clean.

Do not push an intermediate/failing fixture state as the final candidate.

### 5. Evidence

Create/update `WMS_Outbound/05_EVIDENCE/P2-003_EVIDENCE.md` only after final product SHAs are fixed and tested.

Evidence must record:
- exact final pushed Mercato SHA;
- exact final pushed Scanner SHA;
- narrow build-tooling fixes and why they were needed;
- migration source identity matching the already-applied canonical migration;
- P2-003 dedicated result/count and explicit 18-behavior mapping;
- P2-002 `22/22`;
- relevant P1-008/relevance-based regressions;
- Mercato runtime proof (service/MainPID/3009/login/build manifests/identity);
- Scanner canonical runtime/fresh export;
- Journey A PASS evidence and persisted reconciliation;
- Journey B PASS evidence and persisted reconciliation;
- explicit zero route mocks;
- clean product worktrees.

Evidence must **not** self-declare `FINAL PASS`, `Owner Accepted`, `Human Verified`, or equivalent. Supervisor/Owner acceptance remains external.

## Hard scope boundary

No P2-004 shortage/damage/empty/cancel, P2-005 GR, P2-006 Shipment/ERP/manifest, P3/P4, Return Receipt, Demo, Prod, or unrelated redesign.

## STOP conditions

STOP and report one exact blocker only if:
- an existing real journey exposes a product defect that cannot be safely fixed within P2-003;
- canonical runtime cannot be restored after diagnosing the exact concrete cause;
- an unavoidable schema mismatch is found (do not downgrade canonical Testing DB);
- final push is impossible for a concrete Git/auth reason.

Do not STOP merely because a fixture did not previously exist: it exists now and must be run first.

## Final report format

Report only:
- current/final Mercato SHA and pushed remote SHA;
- current/final Scanner SHA and pushed remote SHA;
- clean/dirty status for both;
- Mercato runtime proof;
- Scanner runtime proof;
- P2-003 dedicated result + 18-behavior mapping status;
- P2-002 result;
- relevant P1-008/relevance regressions;
- Journey A result;
- Journey B result;
- zero-route-mocks confirmation;
- WMS evidence SHA;
- if STOPped, one exact remaining blocker.

Then STOP.