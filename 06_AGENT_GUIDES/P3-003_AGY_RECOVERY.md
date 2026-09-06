# P3-003 — AntiGravity recovery / finish from current checkpoint

**Status:** Owner-authorized same-item executor handoff  
**Item:** P3-003 — Cancellation race: physical movement before formal confirmation  
**Executor:** AntiGravity, Owner-selected  
**Purpose:** finish the existing P3-003 work without restarting or discarding Codex progress

## Supervisor execution contract

Before acting, load/apply the current:

- `fetch_me_prompt`
- `operational-mode`
- `wms-outbound`
- `scanner-context`
- `architecture-context` only if a shared Inventory/TU/warehouse/locking primitive is actually touched

Current WMS Git steering and the canonical P3-003 guide override generic skill wording where they differ.

This is a **same-item executor handoff**, not a new item:

- **DO NOT run the Testing reset again.** The P3-003 reset requirement was already satisfied earlier in this item.
- **DO NOT recreate branches from accepted bases.**
- **DO NOT restart the implementation from P4-001 / Scanner accepted base.**
- **DO NOT discard, reset, clean, overwrite or otherwise destroy local in-scope work that may be newer than remote.**
- Own the whole remaining P3-003 item through proof, self-repair, rendered acceptance, evidence and push.
- Ordinary implementation/test/fixture/auth/TLS/runtime/build/UI failures are executor-owned self-repair while a normal in-scope correction exists.
- Terminal `BLOCKER` is valid only after the same material path survives two genuinely different substantive attempts, or an Owner-controlled boundary is required.
- Return only `COMPLETE` with final pushed SHAs/test counts or a true blocker.

## Canonical authority

Read and execute the full existing item contract:

`06_AGENT_GUIDES/P3-003_EXECUTION.md`

Business/acceptance authority remains exactly the sources named there: P3 R7-R8, P4 handoff boundary, `FR-P3-04`, `INT-06`, `TC-042`, `TC-043`, Task Catalog P3-003 and accepted P3-001/P3-002/P4-001 behavior.

Do not start P4-002/P4-003.

## Durable checkpoint already pushed

As of this recovery handoff, the minimum known remote checkpoint is:

### Mercato

- branch: `outbound/p3-003`
- remote HEAD: `65f89c375daa738069d6638c9f7af5675da87b6e`
- accepted base / merge base: `66e2e8620041d2db1d10d069e286936083667139`
- current branch is a straight descendant of that accepted base
- current remote work already includes the P3 pre-confirm correlation/API/migration, cancellation/formal-confirm arbitration work, shared per-line PostgreSQL advisory serialization and later P4 reroute corrections

### Scanner

- branch: `outbound/p3-003`
- remote HEAD: `2e8531db42e1a7abd5e08b13e1d01126d9621f91`
- accepted base / merge base: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- remote already contains `e2e/p3-003-preconfirm-recovery.spec.ts` plus the P3 pre-confirm/recovery UI/API changes

### WMS_Outbound

- `05_EVIDENCE/P3-003_EVIDENCE.md` is **not yet present on main** at handoff time.

### Known remote gap

The Mercato remote compare from accepted P4-001 base through the checkpoint above does **not** show a dedicated P3-003 PostgreSQL integration test file. Treat the mandatory dedicated real-PostgreSQL P3-003 suite as unresolved unless the local workspace contains newer uncommitted/unpushed work that supplies it.

## First action: recover current workspace, do not overwrite it

Before pulling/resetting/checking out anything, inspect the current Mercato, Scanner and WMS_Outbound workspaces:

- current branch and HEAD;
- `git status` / working-tree dirt;
- local commits ahead of remote;
- uncommitted in-scope files/diffs;
- any locally-created P3-003 test/evidence/browser artifacts from the prior Codex run.

If local state contains newer P3-003 work than the remote checkpoint, **preserve and continue from it**. Do not throw it away merely to match the remote HEADs above.

If the worktree is already clean and matches/is behind the remote P3-003 branches, fast-forward to the current remote branch state and continue.

If local in-scope changes are dirty, inspect them first and retain any valid work. Resolve/commit/push them only after verification. Never use destructive cleanup to make the repository look clean.

The checkpoint SHAs above are a **floor**, not permission to overwrite newer valid workspace state.

## Recovery objective

Do not re-implement behavior that is already correctly present. Perform a gap-driven closeout against `P3-003_EXECUTION.md`:

1. Verify the current product implementation against the authoritative P3/P4 race semantics.
2. Preserve already-correct implementation and Scanner flow.
3. Fix only genuine defects exposed by the required proof.
4. Complete every missing mandatory proof and evidence artifact.
5. Push final Mercato/Scanner/WMS revisions.

## Mandatory remaining proof / closeout

The item is not complete until the full P3-003 guide is satisfied, including at minimum:

### Real PostgreSQL P3-003 suite

Add/run the dedicated real PostgreSQL P3-003 suite covering all required guide cases, including:

- pre-confirm persistence with zero formal pick effect;
- idempotent observation;
- wrong source / wrong SKU zero mutation;
- cancellation-wins exact-source path;
- zero `PutBackTask` and zero P4 physical-return handoff on P3 race;
- cancellation/instruction replay idempotency;
- normal formal-pick settlement;
- short-pick settlement;
- pickedQty > 0 routes P4;
- real concurrency A: cancellation commits first;
- real concurrency B: formal confirmation commits first;
- stale Scanner confirmation rejection;
- isolation;
- real rollback: write+flush -> deterministic failure before commit -> fresh independent read unchanged.

Concurrency must use independent overlapping real PostgreSQL operations plus PostgreSQL-side proof. No wall-clock-sleep fake concurrency.

### Scanner

Use the existing P3-003 Scanner implementation/tests as a starting point, not something to recreate. Run and complete the dedicated P3-003 behavior coverage required by the guide. Repair only if the tests or rendered flow expose a genuine defect.

### Regressions / build / runtime

Run every regression required by `P3-003_EXECUTION.md`, including the accepted P3/P4 and picking/allocation/customer-order boundaries touched by the final diff. Because Mercato received additional fixes after earlier partial reports, do not rely on stale pre-fix green results for affected paths.

Run required generate/typecheck/build/runtime checks for both changed applications.

### Rendered acceptance — actual human application flow

Run the required zero-route-mock rendered acceptance through the normal application surfaces:

- **TC-042:** Scanner operator performs source-location + SKU pre-confirm step -> Mercato Supervisor performs normal cancellation -> Scanner visibly receives the exact-source return instruction; persisted result remains pickedQty=0 with P3 release exactly once and no PutBackTask/P4 handoff.
- **TC-043:** Scanner operator formally confirms positive pick -> Mercato Supervisor performs normal cancellation -> accepted P4 path is selected and no P3 source-return instruction exists for that picked quantity.

The decisive user actions must be performed through rendered Scanner/Mercato UI. Fixture API/DB setup may prepare state but cannot substitute those human actions. Label automation `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

### Evidence

Create and push:

`05_EVIDENCE/P3-003_EVIDENCE.md`

It must contain the exact final 40-char Mercato/Scanner SHAs, accepted lineage, P3/P4 authority mapping, dedicated PostgreSQL case results, real concurrency/rollback provenance, Scanner results, mandatory regressions, build/runtime results, rendered TC-042/TC-043 results, explicit zero PutBackTask/P4 handoff proof on TC-042 and explicit P4 routing after formal pick.

Do not include Testing credential values. Existing Testing credential handling is Owner-controlled and out of scope for cleanup/refactor/rotation.

## Completion

`COMPLETE` is valid only after:

1. Mercato final P3-003 branch is pushed;
2. Scanner final P3-003 branch is pushed;
3. `05_EVIDENCE/P3-003_EVIDENCE.md` is pushed to `WMS_Outbound/main`;
4. all mandatory proof in `P3-003_EXECUTION.md` is green and truthfully evidenced;
5. final report includes exact remote Mercato SHA, Scanner SHA, WMS evidence SHA and decisive test counts.

Then STOP. Do not start another Task Catalog item.