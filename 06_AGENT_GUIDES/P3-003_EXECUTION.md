# P3-003 — Cancellation race: physical movement before formal confirmation

**Status:** owner-authorized execution guide  
**Catalog position:** item 28/37  
**Active after:** P3-001, P3-002 and P4-001 accepted  
**Executor-neutral:** Codex / Antigravity / other Owner-selected executor under canonical Devaxonic-WMS steering

## Supervisor execution contract

- Required skills before launch: `fetch_me_prompt` + `operational-mode`.
- `fetch_me_prompt` governs ticket content; `operational-mode` governs model/launch/supervision/retries/fail-stop/notifications/completion handling.
- Current Owner-authorized WMS Git steering overrides generic skill wording if they differ.
- The executor owns this whole authorized Task Catalog item through implementation, real tests, runtime/UI proof, evidence and push.
- Two-strikes applies only to the same material unresolved technical path after two genuinely different substantive attempts. Routine fixture/auth/TLS/test-data/selector/tooling/runtime/build/test issues remain executor-owned while a normal in-scope correction exists.
- Return only `COMPLETE` or a true two-strikes / Owner-controlled blocker. Never self-declare `FINAL PASS`, `Owner Accepted` or `Human Verified`.

## Canonical bases

Start from the exact accepted product boundary unless current Git proves a later Owner-accepted descendant:

- Mercato accepted P4-001 head: `66e2e8620041d2db1d10d069e286936083667139`.
- P4-001 evidence: `6780af113cbc31db26449fa6ca2fde5f238ab801`.
- Accepted P3-002 Mercato head already contained in that lineage: `84274acacfbfe0119e270ca5bfbcb723e47d7723`.
- P3-002 evidence: `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`.
- Accepted P3-001 evidence: `15c3ad937a4e81d7b67ff96409bd0b6a65553864`.
- Scanner accepted/frozen base: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`.

Expected implementation branches:

- Mercato: `outbound/p3-003`, created from the exact accepted Mercato head above.
- Scanner: use a P3-003 branch from the exact Scanner base if Scanner changes are required by this item. Scanner changes are expected because P3 R7 requires an RF exact-source return instruction.

Do not rewrite accepted branch history. Do not start P4-002.

## Authority

Business authority, in order:

1. `01_ARCHITECT_SOURCE/2026-08-31/proces_3_reservation_release.md` — P3 R7–R8 and the physical-removal-before-confirmation exception.
2. `01_ARCHITECT_SOURCE/2026-08-31/proces_4_physical_putback.md` — P4 boundary once formal picked quantity exists; reuse accepted P4-001 logical cancellation only.
3. `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P3-04`, `INT-06`.
4. `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — `TC-042`, `TC-043`.
5. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — P3-003 delivery slice.
6. Accepted P3-001 / P3-002 / P4-001 implementation and evidence as current-state implementation facts.

For Scanner technique only, use `scanner-context` as reference guidance. Outbound Architect behavior wins. Preserve the server-authoritative/idempotent/concurrency model (`SCN-SEC-034`, `SCN-SEC-054`, `SCN-SEC-072`–`074`).

Use `architecture-context` only if a shared Inventory/TU/warehouse/lock/orchestration primitive is actually touched; it cannot override Outbound behavior.

## Exact business behavior

### 1. Pre-confirm physical-removal race belongs to P3

P3 R7 defines one narrow race window:

- operator has verified/scanned the source location and SKU and may already have physically removed the SKU from that source;
- the destination Picking TU + quantity has **not** yet been formally confirmed;
- authoritative formal `pickedQty` is still `0`;
- Allocation remains `RESERVED` until the cancellation decision settles.

If cancellation/release wins in this window:

- execute accepted P3-001 logical release exactly once;
- cancel the affected PickTask/PickTaskLine as allowed by P3-001;
- provide the affected operator an RF instruction to return the physically removed SKU to the **exact source location from which it was taken**;
- create **no `PutBackTask`**;
- do not ask the operator to choose or validate a new location;
- do not create an Inventory `PICKED -> AVAILABLE` movement because formal pick was never recorded; P3-001 owns the logical reservation release;
- do not invent a new OutboundOrderLine business status for this race.

Do not invent an ungrounded returned quantity if it has not been formally confirmed. The RF instruction must communicate the authoritative task/SKU/exact-source context needed by the operator; quantity handling may only use information already authoritatively available from the task/scan contract.

### 2. Formal confirmation belongs to P4

If formal pick confirmation wins first so that authoritative `pickedQty > 0` (or the line has entered the accepted formal pick states), P3 must not release that picked quantity and must not emit the P3 exact-source race instruction.

Route/settle cancellation through the already accepted P4-001 path. P4-002/P4-003 physical put-back execution remains outside this item.

### 3. The discriminator is server/DB authoritative

The cancellation decision must be atomic with the same authoritative rows that formal pick confirmation uses. UI ordering, local Scanner state, a green barcode decode or a client-only flag cannot decide the winner.

The implementation must make these competing operations deterministic under real PostgreSQL locking:

- pre-confirm removal observation / correlation;
- P3 cancellation/release;
- formal pick confirmation.

Exactly one business path wins. A stale/late competing request must receive a clear non-mutating result consistent with the committed winner.

## Current implementation facts to reuse

At the accepted bases:

- `reservation-release-service.ts` already performs P3-001 atomic logical release and can cancel zero-picked PickTaskLine/PickTask work.
- P4-001 already owns formal post-pick cancellation and logical settlement.
- `pick-task-service.ts` makes formal pick confirmation authoritative and, after the accepted lifecycle fix, changes `OutboundOrderLine` to `PICKING` only on actual confirmation; full pick confirms Allocation `RESERVED -> CONFIRMED`.
- Scanner `PickingTaskScreen.js` currently submits source location + SKU + quantity/TU together through `confirmPickLineApi`; therefore current code has no durable server-visible representation of the P3 R7 window before formal confirmation.

Extend these paths; do not duplicate cancellation or picking engines.

## Required implementation shape

Implement the smallest architecture-compliant correlation mechanism that makes the P3 R7 race real and testable.

It must:

1. Persist a durable, tenant/organization/warehouse/task-line/operator-correlated observation that source location + SKU were verified for the active pick before formal quantity/TU confirmation.
2. Preserve the exact source location from the authoritative PickTaskLine and validate the scanned source/SKU against that task context.
3. Be idempotent for Scanner retry/ambiguous timeout.
4. Not increment `pickedQty`, not create TU content, not confirm Allocation, and not transition the OutboundOrderLine into a formal pick state merely because the pre-confirm observation exists.
5. Be cleared/settled or otherwise made non-actionable when formal confirmation succeeds, without leaving an instruction that can later be misread as an unresolved P3 race.
6. When P3 cancellation wins, persist enough durable recovery/instruction correlation for the Scanner to render the exact-source return instruction even if the original cancellation response is not the Scanner's active request.
7. Keep the instruction one-shot/idempotent at the business-effect level; repeated reads/retries must not repeat inventory/release mutations.
8. Avoid creating a new business state machine. Technical correlation fields/record status needed solely to distinguish unresolved/settled instruction state are allowed only if they do not invent a new architect business state.

Choose the smallest existing persistence/API pattern after inspecting accepted code. Do not create a generic notification framework, workflow engine or new task type.

## Scanner behavior

The normal Scanner picking journey must expose the architect sequence rather than faking the race entirely in test setup:

`active PickTask -> source location -> SKU -> pre-confirm server acceptance -> Picking TU / quantity formal confirmation`

The precise existing Scanner interaction may be adapted minimally, but the decisive P3 R7 source/SKU observation must reach the server before formal pick confirmation.

When P3 cancellation wins after that observation:

- Scanner must visibly stop further normal confirmation for the cancelled line/task;
- show a human-readable instruction containing the SKU/task context and **exact source location**;
- make clear that this is a return-to-source instruction, not a `PutBackTask` destination-selection flow;
- after the operator leaves/acknowledges the instruction according to the smallest existing UI pattern, the cancelled work must not become pickable again.

If formal confirmation won first, Scanner must not show the P3 return-to-source instruction for that quantity; subsequent cancellation follows P4.

Do not expose UUIDs as the operator-facing instruction when business identifiers exist.

## Mercato behavior

The existing Supervisor cancellation surface / INT-06 router must show the recovery path actually chosen by the authoritative server decision.

For a zero-picked P3 race, visible feedback may identify P3/source-return behavior but must not claim a `PutBackTask` exists.

For formal picked quantity, preserve the accepted P4-001 behavior and boundaries.

Do not add new Supervisor approvals for P3 R7.

## Hard exclusions

This item does **not** authorize:

- P4-002 `PutBackTask` model/assignment/lifecycle;
- P4-003 target-location validation or `Inventory PICKED -> AVAILABLE` recovery;
- a new general notification/event bus;
- a new Outbound business status solely for the race marker;
- changing P3-002 retention policy semantics;
- changing priority/SLA behavior;
- changing accepted dispatch/posting/manifest boundaries;
- broad Scanner redesign or unrelated navigation cleanup;
- changes to shared Inbound business semantics;
- Testing credential audit/refactor/rotation/replacement. Follow canonical `Devaxonic-WMS/.ai/TESTING.md`; designated Testing credential handling is frozen during implementation.

## Required real PostgreSQL proof

Add a dedicated P3-003 real PostgreSQL suite on canonical Testing PostgreSQL. Cover at least:

1. source-location/SKU pre-confirm observation persists with `pickedQty = 0`, Allocation still `RESERVED`, no TU content/formal pick effect;
2. duplicate/retry of the same pre-confirm observation is idempotent;
3. wrong source location is rejected with zero mutation;
4. wrong SKU is rejected with zero mutation;
5. cancellation after valid pre-confirm observation executes accepted P3 release exactly once, cancels eligible task work and produces exact-source RF recovery correlation;
6. P3 race path creates zero `PutBackTask` and zero P4 physical-return handoff;
7. replay of cancellation/instruction read creates no duplicate release/ATP/inventory/task effect;
8. formal pick confirmation settles the pre-confirm observation and produces the accepted formal `pickedQty`/state effect;
9. cancellation after formal `pickedQty > 0` routes to accepted P4-001 and produces no P3 exact-source instruction;
10. **real concurrency A:** independent overlapping transactions where P3 cancellation commits before formal confirmation -> one P3 release, confirmation loses safely, exact-source instruction remains;
11. **real concurrency B:** independent overlapping transactions where formal confirmation commits before cancellation -> P4 path wins, no P3 source-return instruction;
12. stale Scanner confirmation after task cancellation is rejected with zero mutation and clear reason;
13. multi-line/task isolation: cancelling one raced line does not cancel or instruct unrelated active lines/tasks;
14. real rollback: after persistence/write+flush, deterministic failure before commit leaves observation/release/instruction/task state unchanged on a fresh independent read.

Use deterministic transaction hooks; do not use wall-clock sleeps as concurrency proof.

## Mandatory regressions

At minimum rerun the accepted suites whose product paths are touched:

- P3-001 Reservation Release — accepted 13-case boundary must stay green;
- P3-002 retention release if the shared P3 release entry/router is touched;
- P4-001 post-pick cancellation — accepted 18-case boundary must stay green;
- P1-006 RF picking execution/continuation;
- P1-005 PickTask generation/queue if task lifecycle/generation primitives are touched;
- P1-004 Allocation lifecycle if Allocation/hard reservation primitives are touched;
- P1-001 CustomerOrder aggregation if cancellation aggregation is touched.

Scanner: run the existing relevant Scanner test/lint/build checks plus dedicated P3-003 Scanner behavior tests.

If any shared Inventory/TU/warehouse/record-lock/orchestration primitive is modified, run the targeted accepted Inbound/shared regressions required by `05_EVIDENCE/EVIDENCE_STANDARD.md` and canonical Testing steering.

Run native generate/typecheck/build required by changed Mercato/Scanner scope.

## Rendered acceptance

Provide real rendered acceptance with zero route mocks for the decisive human actions.

### Journey A — TC-042 race / exact-source return

1. Prepare a real allocated/assigned PickTask and operator/session in canonical Testing.
2. Through the normal Scanner UI, enter the active picking task and perform the actual source-location + SKU step so the server records the pre-confirm observation; do **not** formally confirm TU/quantity yet.
3. Through the normal Mercato Supervisor cancellation surface, perform the cancellation/INT-06 action while `pickedQty = 0`.
4. Return to/continue the Scanner journey and visibly receive the exact-source return instruction for the same task/SKU/location.
5. Verify rendered flow does not offer P4 target-location selection / `PutBackTask` behavior.
6. Verify persisted result: P3 logical release settled exactly once, task work cancelled as applicable, formal `pickedQty` remained `0`, no PutBackTask/P4 handoff was created.

### Journey B — TC-043 formal confirmation routes to P4

1. Through normal Scanner picking, perform source/SKU and then formally confirm positive quantity into Picking TU.
2. Through normal Mercato cancellation, cancel the formally picked line within P4-001's allowed boundary.
3. Verify P4-001 is the selected path and no P3 exact-source race instruction is shown/created for the formally picked quantity.

Record exact Mercato and Scanner revisions under test, actor/warehouse/task/order/SKU/location identifiers, visible results and persisted assertions. Automated UI proof is `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

## Evidence

Write:

`05_EVIDENCE/P3-003_EVIDENCE.md`

Evidence must record:

- exact Mercato and Scanner branches/40-char SHAs;
- accepted bases and lineage;
- exact authority `P3 R7–R8`, `FR-P3-04`, `INT-06`, `TC-042`, `TC-043`;
- persistence/API mechanism chosen for the pre-confirm correlation and why it is technical correlation rather than a new business state;
- dedicated PostgreSQL case mapping/results;
- decisive real concurrency and rollback mechanism/results;
- mandatory regression counts;
- real rendered Mercato + Scanner journey results and runtime revision identity;
- explicit zero `PutBackTask` / zero P4 handoff on the P3 race path;
- explicit P4-001 routing after formal picked quantity;
- Scanner final SHA and clean/pushed state;
- truthful evidence labels only.

Do not copy secrets or Testing credential values into evidence.

## Completion

On COMPLETE:

1. Push Mercato P3-003 branch.
2. Push Scanner P3-003 branch if changed.
3. Push `05_EVIDENCE/P3-003_EVIDENCE.md` to `WMS_Outbound/main`.
4. Return only `COMPLETE` with final Mercato SHA, Scanner SHA, WMS evidence SHA and decisive test counts, or a true blocker satisfying the two-strikes/Owner-controlled boundary.
5. STOP. Do not start P4-002 or any later item.
