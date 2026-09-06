# P4-001 — Post-pick/post-pack cancellation approval and logical effects — Execution Guide

**Task Catalog item:** 29/37  
**Execution order note:** P3-003 is catalog item 28/37 but has a hard dependency on P4-001. Therefore P4-001 is the next executable item; after P4-001 acceptance, return to P3-003.  
**Scope:** P4-001 only  
**Status:** Owner-authorized next implementation item after P3-002 FINAL PASS / Owner Accepted

## Accepted starting point

Freeze and preserve the accepted P3-002 checkpoint:

- Mercato accepted P3-002: `84274acacfbfe0119e270ca5bfbcb723e47d7723`
- Scanner frozen accepted baseline: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- WMS P3-002 evidence: `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`

Expected Mercato implementation branch: `outbound/p4-001`, created from the exact accepted P3-002 Mercato head above.

Scanner stays frozen. P4-001 owns cancellation approval/boundary and immediate logical settlement in Mercato/backend; Scanner receives PutBackTask only in later P4-002/P4-003 work.

## Supervisor execution contract

- Required skills before launch: `fetch_me_prompt` + `operational-mode`.
- `fetch_me_prompt` governs ticket content; `operational-mode` governs model, launch, supervision, retries/fail-stop, notifications and completion handling.

Failure boundary:

- The same action/path may fail at most twice.
- After the second materially identical failure: STOP and report the exact action, exact error/failure, repo/branch/HEAD/runtime state, checks already run, and the exact remaining item/decision needed.
- No third retry, workaround route, model/tool/executor switch, scope expansion or redesign without supervisor instruction.

Project steering and Owner instructions override generic executor mechanics where they differ.

## Authority

Business truth for this item, in precedence order:

1. `01_ARCHITECT_SOURCE/2026-08-31/proces_4_physical_putback.md` — P4 KROK 1–2, R1–R4.
2. `01_ARCHITECT_SOURCE/2026-08-31/proces_3_reservation_release.md` — P3 KROK 1 / R8 handoff boundary.
3. `01_ARCHITECT_SOURCE/2026-08-31/model_stanow_outbound.md` — canonical state/event vocabulary.
4. `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P4-01`, `FR-P4-02`, `INT-06`.
5. `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — `TC-050`, `TC-051`, `TC-053`.
6. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — P4-001 delivery slice only.
7. Accepted P3-001/P3-002 cancellation routing and idempotency implementation as implementation evidence, never as business authority.

Use `architecture-context` only for shared/Inbound compatibility patterns. It must not override WMS Outbound behavior. Scanner context is not required unless an unexpected shared Scanner primitive is touched; no Scanner product change is authorized here.

## Objective

Implement the authoritative P4 cancellation boundary after formal pick / after pack, with immediate logical settlement when cancellation is allowed.

P4-001 must make the existing INT-06 cancellation router capable of executing the P4 logical path instead of only recognizing it and throwing a P4 handoff error.

This item does **not** implement physical put-back execution. It must preserve enough recovery quantity/correlation for later P4-002/P4-003, but it must not create or execute the RF PutBackTask workflow.

## Core invariants

### 1. P3/P4 discriminator remains authoritative

Preserve the accepted boundary:

- true pre-pick quantity remains P3;
- `PICKING`, `SHORT_PICKED`, `PICKED`, `PACKED` and/or formally confirmed picked quantity route to P4;
- P3 must never release a P4 quantity;
- P4-001 must not weaken the corrected lifecycle in which `generatePickTasks()` leaves the line `ALLOCATED` and first formal `confirmPickLine()` starts `PICKING`.

P3-003 will later own the physical-removal-before-formal-confirmation race. Do not implement that race here.

### 2. Cancellation before PACKED does not require Supervisor approval

For an eligible formally picked line in `PICKING` / `SHORT_PICKED` / `PICKED` that is not yet `PACKED`, P4 KROK 1 routes directly to logical settlement without a separate Warehouse Supervisor approval gate.

Do not invent an approval step for these states.

### 3. PACKED requires Warehouse Supervisor approval

For `OutboundOrderLine PACKED`:

- cancellation requires an explicit Warehouse Supervisor approval;
- approval/audit correlation must be durable and idempotent;
- where the line is attached to a Packing TU / Shipment, the approved logical cancellation may detach/withdraw the affected Packing TU from the Shipment as required by the existing model;
- label invalidation or equivalent must follow an existing accepted project mechanism if one exists; do not invent an external carrier cancellation integration.

### 4. POSTING_PENDING is the hard P4 cutoff

A `PACKED` cancellation is allowed only while the related Shipment is **before** `POSTING_PENDING`.

At `POSTING_PENDING` or later:

- reject P4 cancellation;
- perform zero cancellation mutation;
- expose a clear blocked reason that points to Return Receipt as the later business path;
- Return Receipt itself is out of scope.

Preserve the already-accepted stronger general cancellation boundaries around closed/later CarrierManifest states. Do not weaken P1 cancellation protection.

### 5. Immediate logical settlement does not wait for physical return

Once P4 cancellation is valid/approved:

- `CustomerOrderLine` cancellation semantics settle immediately according to the authoritative general cancellation path;
- affected `OutboundOrderLine -> CANCELLED` immediately;
- Standard-channel `Allocation CONFIRMED -> RELEASED` immediately;
- corresponding quantity returns to the correct `ATPReservation` immediately;
- parent order/header aggregation is recalculated atomically;
- physical Inventory remains physically picked until later P4 physical put-back completion; do **not** fabricate `Inventory PICKED -> AVAILABLE` in P4-001;
- logical settlement must not wait for P4-002/P4-003.

For CROSSDOCK, no standard Allocation exists. Never fabricate one.

### 6. Preserve recovery quantity for later P4 task

For `pickedQty > 0`, persist or expose one authoritative recovery quantity/correlation sufficient for P4-002 to create exactly one PutBackTask later.

P4-001 must not create the PutBackTask itself unless existing code already creates a narrow placeholder that is explicitly required by the Task Catalog boundary. The expected implementation boundary is logical settlement + durable recovery fact only.

For `pickedQty = 0`, no physical recovery quantity exists and no future PutBackTask should be implied (`TC-053`). This zero-picked path must remain compatible with P3/P3-003 boundaries rather than manufacturing physical work.

### 7. INT-06 idempotency and conflict safety

Cancellation correlation/replay must remain exactly-once:

- identical request/key replays with zero duplicate status/ATP/release/audit/recovery mutation;
- conflicting key/payload fails safely;
- whole-order cancellation remains atomic across all targeted lines;
- if any target line is past an authoritative cancellation boundary, do not partially mutate the rest of a whole-order request unless current accepted INT-06 semantics explicitly allow a line-scoped request.

### 8. Separation from adjacent items

Do not implement:

- P3-003 physical-removal race or exact-source RF return instruction;
- P4-002 PutBackTask model/assignment/FIFO/RF module;
- P4-003 destination-location validation loop or `Inventory PICKED -> AVAILABLE` physical recovery;
- Return Receipt;
- new carrier cancellation integration;
- Scanner UI changes;
- Demo/Prod changes.

## Reuse first

Inspect and reuse accepted implementation paths before adding new infrastructure:

- `ordering-adapter-service.ts` INT-06 correlation/routing;
- P3-001 durable cancellation idempotency patterns where compatible;
- existing CustomerOrder/OutboundOrder aggregation services;
- existing Shipment/CarrierManifest boundary checks from accepted P1-014/P1-015/P1-016;
- existing state-transition/audit mechanisms;
- existing Mercato Supervisor cancellation/shortage operational patterns.

Do not duplicate cancellation routing or create a second incompatible status engine.

## Required real Testing PostgreSQL proof

Use canonical Testing PostgreSQL only, per `Devaxonic-WMS/.ai/TESTING.md`. No local PostgreSQL.

Provide a dedicated P4-001 PostgreSQL integration suite proving at least:

1. `PICKING` with formal `pickedQty > 0`: cancellation routes to P4 and performs immediate logical settlement without a Supervisor approval requirement.
2. `SHORT_PICKED` with picked quantity: same P4 logical path; no P3 mutation.
3. `PICKED`: same immediate logical settlement.
4. `PACKED` before `POSTING_PENDING`: without Supervisor approval, cancellation is rejected with zero mutation.
5. `PACKED` before `POSTING_PENDING`: with explicit Supervisor approval, logical settlement succeeds exactly once.
6. `PACKED` with Shipment `POSTING_PENDING`: cancellation rejected with zero mutation and explicit Return Receipt direction.
7. later Shipment/CarrierManifest boundary remains blocked; accepted P1-014/P1-016 cancellation protection is not weakened.
8. Standard Allocation transitions from the P4-owned active state to `RELEASED` exactly once; hard/physical stock is not falsely returned to AVAILABLE in this item.
9. ATPReservation restoration is exact and occurs immediately with logical cancellation, once only.
10. CustomerOrderLine / OutboundOrderLine / parent aggregation settle atomically and match Architect semantics.
11. recovery quantity/correlation for `pickedQty > 0` is durable and exact for later P4-002, with no duplicate recovery fact on replay.
12. `pickedQty = 0` creates no recovery quantity / PutBackTask implication and remains outside the P4 physical-return path (`TC-053`).
13. CROSSDOCK cancellation does not fabricate standard Allocation.
14. identical INT-06 replay is idempotent; conflicting key/payload fails safely.
15. real rollback: after writes/flush and before commit, injected failure leaves all P4 logical state/recovery/audit unchanged on fresh independent read.
16. real concurrency: two overlapping cancellation approvals/executions against the same line serialize to one logical settlement and one recovery fact.

## Mandatory regressions

Run the smallest relevant accepted regressions based on touched code, including:

- P3-001 reservation release suite;
- P3-002 retention suite if shared cancellation routing/discriminator is touched;
- P1-001 CustomerOrder aggregation if header/line settlement is touched;
- P1-004 Allocation lifecycle if Allocation release primitives are touched;
- P1-014 Shipment POST boundary;
- P1-015/P1-016 boundary/manifest regressions when their cancellation/Shipment/manifest primitives are touched;
- P1-006 picking regression because the P3/P4 discriminator and formal pick lifecycle must remain intact;
- shared Inbound regressions only if shared Inventory/TU/warehouse/lock/orchestration primitives are modified.

Scanner stays frozen unless Scanner code changes, which is not expected/authorized.

Run native generate/typecheck/build required by the changed Mercato application scope.

## Rendered Mercato acceptance

Provide real rendered Playwright proof with zero route mocks for the human-facing behavior.

Minimum Journey A — PACKED cancellation allowed:

1. prepare a genuine PACKED line attached to a Shipment that is still before `POSTING_PENDING`;
2. navigate through the normal Mercato Supervisor UI;
3. attempt cancellation and observe the required Supervisor approval interaction;
4. approve decisively;
5. verify rendered cancelled state and persisted immediate logical effects;
6. revisit/replay and prove no duplicate settlement/recovery fact.

Minimum Journey B — PACKED cancellation blocked:

1. prepare a genuine PACKED line whose Shipment is `POSTING_PENDING`;
2. navigate through the same normal application surface;
3. attempt cancellation;
4. verify the UI visibly rejects it with the correct boundary/Return Receipt direction;
5. verify fresh PostgreSQL state is unchanged.

A picked-but-not-packed P4 path must also be covered at least by genuine backend/PostgreSQL acceptance; add rendered UI coverage if the normal current cancellation surface exposes it without inventing new UX.

## Evidence

Write durable evidence to:

`05_EVIDENCE/P4-001_EVIDENCE.md`

Evidence must include:

- exact Mercato final SHA and branch;
- Scanner SHA and confirmation that it stayed frozen;
- accepted P3-002 base/evidence refs;
- exact P4 authority refs and P3/P4 discriminator boundary;
- schema/audit/recovery-fact changes, if any;
- dedicated PostgreSQL test names/counts/results;
- approval and POSTING_PENDING boundary proof;
- idempotency/concurrency/rollback mechanism, not just labels;
- regression counts;
- generate/typecheck/build/runtime provenance;
- Playwright suite/file, counts and decisive rendered actions;
- explicit statement that P4-002/P4-003, P3-003 and Return Receipt remain excluded.

Do not mark evidence Human Verified, FINAL PASS or Owner Accepted.

## Mandatory new-item Testing reset

P4-001 is a new Task Catalog item. Before the first P4-001 implementation action, perform the canonical new-item reset defined by `Devaxonic-WMS/.ai/OPERATIONS.md` and require successful completion.

This reset is separate from executor launch. Do not repeat the deep reset for ordinary P4-001 retries/continuations.

## Completion contract

Own the entire authorized P4-001 item until COMPLETE or a genuine same-material-path two-strikes blocker.

Routine fixture/auth/TLS/test-data/runtime/build/service/test failures are executor-owned and are not supervisor micro-tickets.

On COMPLETE:

1. push Mercato `outbound/p4-001` with final candidate SHA;
2. keep Scanner frozen and report exact SHA unless an unexpected authorized need changes that;
3. push `05_EVIDENCE/P4-001_EVIDENCE.md` to `WMS_Outbound/main`;
4. return exact final SHAs and concise decisive DB/UI counts;
5. STOP. Do not start P3-003, P4-002 or any later item.

Executor COMPLETE is not acceptance. Supervisor independently verifies; formal progress advances only after explicit Owner acceptance.
