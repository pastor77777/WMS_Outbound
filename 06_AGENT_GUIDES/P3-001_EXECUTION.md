# P3-001 — Reservation Release before formal pick

## Status and authority

This is the authoritative executor guide for **P3-001 — Reservation Release before formal pick — item 26/37**.

P2-006 is **FINAL PASS / Owner Accepted** and closes the P2 implementation slice.

Frozen accepted bases for P3-001:

- Mercato: `4f64641ab14a5359bc22d0685e390b511252b5b5`
- Scanner: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- P2-006 evidence: `9a580b046b5f2aa3bcbf2422eeaf6413248f68db`

Authority order:

1. immutable Architect Source / faithful Architect translation and Canon;
2. requirements / acceptance scenarios / Task Catalog;
3. accepted implementation and Testing DB as implementation evidence;
4. this guide as bounded delivery contract.

Executor never self-accepts. `FINAL PASS`, `Owner Accepted`, and `Human Verified` are supervisor/Owner states only.

## Mandatory bootstrap

Before implementation, read the canonical Devaxonic-WMS execution contract and current WMS Outbound authority:

- `/home/ubuntu/git/Devaxonic-WMS/AGENTS.md`
- `/home/ubuntu/git/Devaxonic-WMS/.ai/STATE.md`
- current `.ai/HANDOVER_OUTBOUND_CURRENT_*.md`
- `/home/ubuntu/git/Devaxonic-WMS/.ai/TESTING.md`
- `/home/ubuntu/git/Devaxonic-WMS/.ai/OPERATIONS.md`
- `/home/ubuntu/git/WMS_Outbound/AGENTS.md`
- `/home/ubuntu/git/WMS_Outbound/STATE.md`
- current `08_HANDOVER/HANDOVER_CURRENT_*.md`
- `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
- Task Catalog P3-001
- `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_3_reservation_release_EN.md`
- relevant state model / requirements / TC-040 and TC-041.

Current executor mode is **full-item ownership**. Antigravity, local Codex, Codex Cloud and Claude use the same contract: continue through ordinary fixture/auth/tooling/runtime/test failures autonomously; STOP only on COMPLETE, a true two-strikes material blocker, or an Owner-controlled boundary.

## Mandatory new-item Testing reset

P3-001 is a **new Task Catalog item**. Before any P3-001 implementation action:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

Required result: `RESET_OK`.

Do not repeat this deep reset for retries/continuations inside P3-001.

After reset, create/use `outbound/p3-001` from the exact accepted P2-006 Mercato head `4f64641ab14a5359bc22d0685e390b511252b5b5`. Scanner starts frozen at `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` and changes only if the required P3-001 user feedback genuinely needs Scanner source changes.

## Task Catalog objective

Implement Reservation Release for quantity that is still only reserved and has **not been formally picked**.

Task Catalog grounding:

- Architect: P3 R1–R2, R3–R4, R5–R6; P3 KROK 1 / P4 KROK 1 boundary;
- requirements: `FR-P3-01`, `FR-P3-02`, `FR-P3-03`, `INT-06`;
- dependencies: accepted P1-004 Allocation, P1-001 CustomerOrder/Line aggregation/cancellation, P1-016 final settlement foundations;
- acceptance: `TC-040`, `TC-041`;
- DoD: released stock becomes available exactly once; true pre-pick release creates no `PutBackTask`; line/order state is consistent.

P3-002 owns the reservation-retention policy variants/timer (`P3 R9-R10`). P3-003 owns the physical-removal-before-formal-confirmation race and P4 handoff (`P3 R7-R8`). Do not pull those later items forward.

## Architect invariants — implement literally

### 1. P3 is selected only before formal pick

The authoritative discriminator is formal picked quantity/state, not UI intent.

P3 applies only when the affected Standard `OutboundOrderLine` has no formally confirmed picked quantity (`pickedQty = 0` / no accepted pick confirmation for the released quantity).

If formally confirmed picked quantity is greater than zero, P3 must reject/route out without releasing it as a pre-pick reservation. That quantity belongs to P4 later. P3-001 must prove this boundary but must not implement P4.

### 2. Release is one atomic business operation

For the quantity released by P3:

- `Allocation RESERVED -> RELEASED` (or the current accepted pre-pick hard-reservation equivalent allowed by P3 semantics);
- linked hard Inventory reservation is removed/released so stock becomes available again;
- released quantity returns to the relevant `ATPReservation` / demand coverage exactly once;
- no physical inventory decrement/put-back movement is fabricated;
- no `PutBackTask` is created.

The release, task cancellation where applicable, line/order status update, ATP restoration and audit/idempotency effects must commit atomically.

### 3. Do not weaken accepted P1-004 CON-02 globally

Accepted `allocation-service.releaseAllocation()` currently protects a hard Allocation once a PickTask exists. P3 must **not** remove or weaken that generic invariant.

If P3 cancellation needs to release an Allocation with a pre-pick `PickTask`, implement/use a P3-specific authoritative transaction that:

1. locks the line/allocation/task rows;
2. proves formal picked quantity is zero;
3. cancels eligible not-completed PickTask/TaskLine work as part of the same P3 operation;
4. then uses the accepted hard-reservation reconciliation primitive to release the Allocation and restore soft ATP.

This is an explicit cancellation exception, not a general permission to reallocate active picking work.

### 4. Shortage and general cancellation have different demand outcomes

For pre-pick release caused by shortage / `SHORT_ALLOCATED` cancellation:

- released `OutboundOrderLine -> CANCELLED`;
- unfulfilled `CustomerOrderLine -> BACKORDERED`;
- parent order/header aggregation remains consistent with accepted P1 rules.

For general cancellation unrelated to shortage:

- released `OutboundOrderLine -> CANCELLED`;
- affected `CustomerOrderLine -> CANCELLED`;
- parent aggregation follows accepted P1 continuous aggregation/cancellation semantics.

Do not silently convert one reason into the other.

### 5. Automatic-release business effect exists, timer policy does not

P3 R6 requires that an automatic release path does not require Warehouse Supervisor notification.

P3-001 may expose/implement the same authoritative release operation with an automatic/system reason and prove it does not depend on a Supervisor actor.

Do **not** implement:

- retain / auto-release-after-time / Supervisor-decision warehouse policy;
- timer scheduling;
- retention duration;
- priority/SLA independence logic.

Those belong to P3-002.

### 6. INT-06 correlation must be idempotent

If cancellation enters through the accepted ordering-system integration boundary, repeated delivery of the same cancellation correlation must not double-release Allocation/Inventory/ATP or duplicate transitions.

A conflicting reuse of an idempotency/correlation key must fail safely rather than mutate a second target.

### 7. Crossdock is not converted into Allocation

CROSSDOCK lines have no standard Allocation. P3-001 must not manufacture one or apply Standard reservation-release stock semantics to a crossdock contribution.

P2 accepted behavior remains frozen.

### 8. P3-003 race remains future scope

P3-001 proves the boundary at `pickedQty = 0` versus formally picked quantity.

Do not implement the detailed RF race window, exact-source return instruction, or P4 handoff mechanics here. Those are P3-003. If current data exposes a physical-removal marker without formal confirmation, preserve it and fail/route according to current authoritative boundary without inventing a PutBackTask.

## Current implementation facts to reuse

Inspect current accepted code before adding anything. Existing reusable primitives include:

- `allocation-service.ts` hard reservation / `reconcileHardReservationInTx()`;
- `customer-order-service.ts` existing SHORT_ALLOCATED cancellation logic and aggregation;
- `pick-task-service.ts` PickTask / PickTaskLine states and picked quantity persistence;
- state-transition service / durable audit events;
- current ordering/cancellation adapter path for `INT-06`.

Prefer one narrow Reservation Release orchestration service around accepted primitives rather than duplicating Allocation, ATP or task logic.

No separate P3 inventory model is allowed.

## Required substantive PostgreSQL proof

Create a dedicated canonical Testing PostgreSQL suite for P3-001. Test count may vary, but evidence must map every behavior below with substantive assertions:

1. normal pre-pick general cancellation with `pickedQty=0` releases the hard Allocation exactly once;
2. linked hard Inventory reservation disappears / available quantity is restored exactly once;
3. released hard quantity is restored to ATP/demand coverage exactly once;
4. affected Standard OOL becomes `CANCELLED`;
5. general cancellation sets the relevant CustomerOrderLine to `CANCELLED` and preserves correct parent aggregation;
6. shortage release sets the relevant CustomerOrderLine to `BACKORDERED`, not `CANCELLED`;
7. eligible pre-pick PickTask/TaskLine work is cancelled transactionally without weakening generic P1-004 release protection;
8. no `PutBackTask` or P4 physical recovery record is created for true pre-pick release;
9. replay of the same correlation/idempotency key creates zero duplicate release/ATP/audit effects;
10. conflicting key/payload reuse fails safely with no second mutation;
11. formal `pickedQty > 0` / accepted pick confirmation blocks P3 release and leaves Allocation/physical quantity for later P4 handling;
12. automatic/system reason can execute the same release without a Supervisor actor/notification dependency, but no P3-002 timer/policy is implemented;
13. CROSSDOCK/no-Allocation target is rejected or no-ops according to the current contract and creates no fake Allocation;
14. rollback proof: deterministic failure after real writes/flush but before commit leaves Allocation, Inventory reservation, ATP, task and line state unchanged on a fresh read;
15. concurrency proof: two overlapping release attempts for the same line serialize/idempotently produce one final release and no double ATP restoration.

Use canonical Testing PostgreSQL only. Local PostgreSQL is forbidden.

## Mandatory regressions

Run at least the full accepted suites for every directly touched accepted surface:

- P1-004 Allocation hard reservation lifecycle;
- P1-001 CustomerOrder/Line intake, cancellation and aggregation behavior relevant to changed code;
- P1-002 ATPReservation if ATP restore/recalculation code is touched;
- P1-005 PickTask if task cancellation/ownership code is touched;
- P1-016 if final aggregate/settlement primitives are modified rather than merely reused.

Run shared Inventory / warehouse / task-lock / orchestration / accepted Inbound regressions only if those shared primitives are actually modified.

Do not ritualistically rerun P2 suites unless a shared primitive or common service changed in a way that can affect accepted P2 behavior. If not rerun, evidence states why P2 remains frozen.

Run repository-native Mercato generation/typecheck/build contract after final product code.

## Real rendered UI proof

Use the exact canonical Testing runtime/revision and zero Playwright route mocks/interception/substitution of product behavior.

### Journey A — general pre-pick cancellation

Through deterministic setup plus real rendered Mercato operations:

1. create a legitimate STANDARD CustomerOrder/Line, OutboundOrder/Line, hard Allocation and hard Inventory reservation;
2. formal picked quantity is zero;
3. if a pre-pick PickTask exists, it has no accepted picked quantity;
4. Warehouse Supervisor uses the normal cancellation action/feedback surface;
5. rendered UI shows the cancellation/release outcome;
6. persisted reconciliation proves Allocation `RELEASED`, hard reservation removed, ATP restored once, OOL/COL/parent states correct, eligible task cancelled, and zero PutBackTask/physical put-back effect;
7. replay through the real accepted action/integration boundary proves no duplicate release.

### Journey B — shortage vs P4 boundary

Prove, through the most natural real surface plus persisted reconciliation:

- SHORT_ALLOCATED pre-pick cancellation leaves demand `BACKORDERED` while releasing hard reservation; and
- a fixture with formally picked quantity greater than zero cannot execute P3 release and is left for later P4 handling.

The later P3-003 exact-source RF race is not required here and must not be implemented as P3-001.

If Scanner source changes solely to show cancellation feedback for a pre-pick task, add a narrow real Scanner proof. If no Scanner source change is needed, keep Scanner frozen and state that explicitly in evidence.

## Runtime contract

For every changed product repo:

- build/generate using repository-native commands;
- restart the canonical Testing service from the exact final candidate;
- prove exact runtime provenance and HTTP/route health before Playwright;
- preserve non-empty required production manifests where applicable.

Do not use stale built output as acceptance evidence.

## Hard exclusions

P3-001 must not implement:

- P3-002 reservation retention policy/timer;
- P3-003 exact-source physical-removal race workflow;
- any P4 PutBackTask creation/location validation/physical put-away;
- Return Receipt;
- crossdock Allocation fabrication;
- changes to accepted P2 shipment/GR/manifest semantics;
- Demo/Prod;
- local PostgreSQL;
- unrelated refactors or new warehouse business policy.

## Evidence and push

When all gates are green:

1. push exact tested Mercato final to `outbound/p3-001`;
2. keep Scanner at frozen accepted SHA unless genuinely changed; if changed, push exact tested Scanner final to `outbound/p3-001`;
3. write `05_EVIDENCE/P3-001_EVIDENCE.md` with exact product SHAs, migration/schema provenance, dedicated behavior mapping, mandatory regressions, runtime proof, Playwright proof, idempotency/rollback/concurrency evidence, frozen unaffected repos and clean worktrees;
4. push WMS evidence to `main`;
5. evidence must not self-declare `FINAL PASS`, `Owner Accepted` or `Human Verified`.

## Executor completion behavior

Own the **entire P3-001 item** in one run/session context. Ordinary implementation, fixture, auth, test-data, TLS, runtime, build and test failures are yours to diagnose and solve inside scope.

Do not STOP after the first failed test or ordinary harness issue.

The two-strikes rule applies only when the **same material blocker** survives two genuinely different substantive evidence-based attempts and there is no normal in-scope next move.

Return only:

- `COMPLETE` with final Mercato SHA, Scanner final/frozen SHA, WMS evidence SHA and decisive test/UI counts; or
- one exact true two-strikes / Owner-controlled blocker with preserved current state.

Then STOP for supervisor verification. Do not start P3-002.