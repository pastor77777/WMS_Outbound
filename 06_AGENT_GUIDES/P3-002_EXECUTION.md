# P3-002 — Reservation retention policy and automatic release timer — Execution Guide

**Task Catalog item:** 27/37  
**Scope:** P3-002 only  
**Status:** Owner-authorized next implementation item after P3-001 FINAL PASS / Owner Accepted

## Accepted starting point

Freeze and preserve the accepted P3-001 checkpoint:

- Mercato accepted P3-001: `9fb32493ed9ec443a18a494aa1a8ec3a1bde6d06`
- Scanner frozen accepted baseline: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- WMS P3-001 evidence: `15c3ad937a4e81d7b67ff96409bd0b6a65553864`

Expected Mercato implementation branch: `outbound/p3-002`, created from the exact accepted P3-001 Mercato head above.

Scanner stays frozen unless this item genuinely requires a Scanner change. P3-002 has no Picker/RF workflow in Architect scope.

## Authority

Business truth for this item:

1. `01_ARCHITECT_SOURCE/2026-08-31/proces_3_reservation_release.md` — P3 R9-R10.
2. `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P3-05`, `FR-P3-06`.
3. `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — `TC-112`, `TC-113`.
4. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — P3-002 delivery slice only.
5. Accepted P3-001 implementation/evidence for the actual pre-pick release transaction.

Use `architecture-context` only for shared/config/scheduler compatibility patterns. It does not override WMS Outbound behavior.

Before implementation, apply current `fetch_me_prompt` + `operational-mode` execution discipline together with current repository steering and `.ai/TESTING.md`. Current Owner-authorized WMS Git steering wins over generic skill wording where they differ.

## Objective

Implement warehouse-configured retention behavior for a **partial hard reservation before formal pick**.

The warehouse policy has exactly three Architect variants:

1. **retain reservation**;
2. **automatic release after configured time**;
3. **route to Warehouse Supervisor decision**.

The configured retention time is independent of `CustomerOrder.priority` and `CustomerOrder.slaDeadline`.

P3-002 owns policy/timer/decision orchestration only. The actual release mutation must reuse the accepted P3-001 Reservation Release boundary rather than creating a second release mechanism.

## Core invariants

### 1. Eligibility is partial pre-pick reservation

Policy applies to the partial-reservation condition described by P3 R9-R10 / `FR-P3-05`.

Do not apply retention expiry to ordinary fully allocated work merely because an Allocation exists.

Before any release action, preserve the accepted P3-001 boundary:

- formal `pickedQty = 0` / no accepted pick confirmation for released quantity;
- picked quantity belongs outside this item and must not be forced through P3;
- no `PutBackTask` for true pre-pick release.

### 2. Warehouse-scoped policy

Persist/read the policy from the warehouse configuration boundary using existing configuration patterns where safe.

The supported business variants are exactly:

- `RETAIN` (or an existing equivalent canonical code);
- `AUTO_RELEASE_AFTER_TIME` (or equivalent);
- `SUPERVISOR_DECISION` (or equivalent).

Names in code may follow existing repository conventions, but do not add a fourth business behavior.

For timed automatic release, persist a configured duration using the existing project convention for duration units. The UI/API must make the unit unambiguous.

### 3. Stable timer semantics

For `AUTO_RELEASE_AFTER_TIME`, establish the due timestamp once when the partial reservation enters the retention-policy condition. Persist the resulting policy eligibility/due fact using the narrowest existing suitable model or an additive P3-002 model if necessary.

Do not recompute `dueAt` from "now" on every scheduler run. Repeated scheduler evaluation must not postpone expiry.

`dueAt` must be derived only from the configured retention duration and the persisted eligibility timestamp. It must not use, shorten, extend, sort by, or otherwise depend on:

- `priority`;
- `slaDeadline`.

Do not invent retroactive policy-rewrite behavior for already-due reservations unless an existing accepted configuration framework explicitly defines it.

### 4. Retain variant

`RETAIN` keeps the eligible partial reservation intact.

Scheduler/evaluator runs must not release Allocation, restore ATP, cancel the line, or manufacture a due action for this variant.

### 5. Automatic release variant

When `AUTO_RELEASE_AFTER_TIME` is not yet due, make no business mutation.

When due:

- invoke the accepted P3-001 Reservation Release orchestration with a system/automatic policy reason;
- no Warehouse Supervisor notification/approval dependency is required;
- preserve all P3-001 atomic/idempotent effects and boundaries;
- mark/settle the P3-002 policy work so later scheduler runs are harmless.

Do not duplicate Allocation/Inventory/ATP/line/order release logic inside the scheduler.

### 6. Supervisor-decision variant

`SUPERVISOR_DECISION` must not automatically release the reservation.

Expose the eligible case to Warehouse Supervisor through the normal Mercato operational surface and persist enough decision/audit state to make the case deterministic and non-duplicated.

Provide only the minimum architect-supported decision behavior: the Supervisor may cause the eligible reservation to be released now through the accepted P3-001 release path, or leave/retain the reservation. Do not invent escalation chains, SLA-derived urgency, notification workflows, approval hierarchies or extra policy variants.

Repeated evaluation must not create duplicate pending decision cases.

### 7. Idempotency and concurrency

The scheduler/policy evaluator must be safe under repeated and overlapping execution.

Use real PostgreSQL transaction/locking/idempotency mechanisms appropriate to the owning rows so that two workers cannot cause:

- double Reservation Release;
- double ATP restoration;
- duplicate policy-decision records;
- duplicate audit/event effects.

A release triggered by P3-002 must remain idempotent at the accepted P3-001 boundary as well.

### 8. Separation from adjacent items

Do not implement:

- P3-003 physical-removal-before-confirmation race (`R7-R8`);
- exact-source RF return instruction;
- P4 Physical Putback / `PutBackTask`;
- new priority/SLA queue semantics;
- new cancellation business rules beyond invoking accepted P3-001;
- Scanner workflow unless a real P3-002 product requirement is discovered in authoritative sources (none is currently mapped).

## Reuse first

Inspect and reuse current accepted patterns before adding infrastructure:

- P3-001 `reservation-release-service.ts` and its durable idempotency record;
- existing warehouse configuration/settings ownership;
- existing scheduler/background-job conventions used by Mercato/WMS;
- existing transition/audit conventions;
- existing Supervisor operational UI patterns.

Do not create a parallel generic scheduler framework if the repository already has an accepted one.

## Required real Testing PostgreSQL proof

Use canonical Testing PostgreSQL only, per `Devaxonic-WMS/.ai/TESTING.md`. No local PostgreSQL.

Provide a dedicated P3-002 PostgreSQL integration suite that proves at least:

1. **RETAIN:** eligible partial pre-pick reservation remains reserved across repeated evaluator/scheduler runs; no P3 release mutation occurs.
2. **AUTO before due:** no release before the persisted due time.
3. **AUTO due:** due case invokes P3-001 exactly once; Allocation/Inventory/ATP/line/order effects match accepted P3-001 and no Supervisor actor/notification is required.
4. **AUTO replay:** repeated scheduler runs after successful release cause zero duplicate release, ATP restoration, ReservationRelease record or audit effect.
5. **SUPERVISOR_DECISION:** evaluator creates/exposes one pending decision case and does not auto-release.
6. **Supervisor release decision:** decisive Supervisor action invokes the accepted P3-001 release exactly once and settles the pending decision audit.
7. **Supervisor retain decision:** reservation remains intact and the decision is durably auditable without creating release effects.
8. **Priority/SLA independence (`TC-113`):** two otherwise equivalent eligible reservations with different `priority` and `slaDeadline` receive the same retention duration semantics; those fields do not change the due calculation.
9. **Warehouse scope:** different warehouses can hold different policy variants/configured durations without cross-contamination.
10. **P3 boundary preserved:** a formally picked quantity cannot be released by the P3-002 timer/decision path.
11. **No physical recovery:** zero `PutBackTask` / P3-003 exact-source recovery is created by P3-002.
12. **Real concurrency:** two overlapping scheduler workers against the same due reservation serialize/idempotently to one final release and one policy settlement.
13. **Rollback safety:** after real DB writes/flush but before commit, a deterministic injected failure leaves policy state and P3 business state unchanged on fresh independent read.

Use deterministic clock control/injected `now` in test code where needed; do not make acceptance depend on wall-clock sleeps. Persisted PostgreSQL state must still be real.

## Mandatory regressions

Run the smallest relevant accepted regressions based on touched code, including:

- P3-001 Reservation Release PostgreSQL suite;
- P1-004 Allocation hard reservation lifecycle if Allocation/shared release primitives are touched;
- P1-001 CustomerOrder aggregation if line/order aggregation primitives are touched;
- P1-002 ATP suite if ATP code is touched;
- any existing warehouse-config/scheduler regression directly touched by the implementation;
- shared Inbound regressions only when shared primitives are actually modified.

Scanner remains frozen unless Scanner code changes. Do not ritualistically rerun unrelated P2 suites; state why they remain frozen.

Run native generate/typecheck/build required by the changed application scope.

## Rendered Mercato acceptance

Provide real rendered Playwright proof with zero route mocks for the human-facing behavior.

Minimum Journey A — Supervisor decision:

1. create/prepare a genuine partial pre-pick reservation eligible for `SUPERVISOR_DECISION`;
2. navigate through the normal Mercato Supervisor UI;
3. verify the pending retention decision is visibly attributable to the correct warehouse/order/line;
4. perform a decisive Supervisor action in the rendered UI;
5. for release decision, verify persisted P3-001 release effects and settled decision audit;
6. verify replay/revisit does not present a second active decision or double-release.

Minimum Journey B — policy/timer observability:

- verify configured warehouse policy/duration is visible through the normal application surface as appropriate;
- exercise a due `AUTO_RELEASE_AFTER_TIME` case using real backend/scheduler behavior and verify the rendered resulting order/line state;
- include a `TC-113` proof that differing `priority`/`slaDeadline` do not alter the configured retention duration semantics.

Do not use route interception/mocks as the decisive business proof.

## Evidence

Write durable evidence to:

`05_EVIDENCE/P3-002_EVIDENCE.md`

Evidence must include:

- exact Mercato final SHA and branch;
- Scanner SHA and whether frozen;
- exact P3-001 accepted dependency refs;
- schema/config/scheduler changes;
- dedicated PostgreSQL test names/counts/results;
- concurrency and rollback mechanism, not just labels;
- regression counts;
- generate/typecheck/build/runtime provenance;
- Playwright suite/file, counts and decisive rendered actions;
- explicit statement that `priority` and `slaDeadline` do not affect retention time;
- explicit exclusions preserved (P3-003/P4/Scanner if frozen).

## Mandatory new-item Testing reset

P3-002 is a new Task Catalog item. Before the first P3-002 implementation action, perform the canonical new-item reset defined by `Devaxonic-WMS/.ai/OPERATIONS.md` and require `RESET_OK`.

This reset is separate from executor launch. Do not repeat the deep reset for ordinary P3-002 retries/continuations.

## Completion contract

Own the entire authorized P3-002 item until COMPLETE or a genuine same-material-path two-strikes blocker.

Routine fixture/auth/TLS/test-data/runtime/build/service/test failures are executor-owned and are not new supervisor micro-tickets.

On COMPLETE:

1. push Mercato `outbound/p3-002` with final candidate SHA;
2. keep Scanner frozen unless genuinely changed and report its exact SHA;
3. push `05_EVIDENCE/P3-002_EVIDENCE.md` to `WMS_Outbound/main`;
4. return exact final SHAs and concise decisive test/UI counts;
5. STOP. Do not start P3-003 or any P4 item.

Executor COMPLETE is not acceptance. Supervisor independently verifies; formal progress advances only after explicit Owner acceptance.
