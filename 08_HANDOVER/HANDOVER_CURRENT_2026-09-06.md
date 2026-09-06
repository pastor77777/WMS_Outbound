# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-06 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Formal progress: **26/37 FINAL PASS / Owner Accepted**.

Latest accepted checkpoint:

**P3-001 — Reservation Release before formal pick — item 26/37 — FINAL PASS / Owner Accepted.**

- Mercato: `9fb32493ed9ec443a18a494aa1a8ec3a1bde6d06`
- Scanner frozen: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- WMS evidence: `15c3ad937a4e81d7b67ff96409bd0b6a65553864`

Supervisor independently verified P3-001 before Owner acceptance.

## Accepted P3-001 boundary

Preserve exactly:

1. P3 release applies only before formal pick (`pickedQty = 0` / no accepted pick confirmation for released quantity).
2. Allocation/hard Inventory reservation/ATP/task/line/order effects settle atomically and exactly once.
3. General cancellation sets affected CustomerOrderLine to `CANCELLED`; shortage release sets it to `BACKORDERED`.
4. Eligible pre-pick PickTask/TaskLine may be cancelled inside dedicated P3 orchestration without weakening generic P1-004 CON-02.
5. True pre-pick release creates zero `PutBackTask`.
6. `INT-06` replay is idempotent; conflicting key reuse fails safely.
7. Crossdock/no-Allocation path does not fabricate Allocation.
8. Formal `pickedQty > 0` is rejected from P3 and remains P4 territory.
9. Automatic/system release can use the same accepted release operation without Supervisor notification dependency.
10. P3-003 still owns the physical-removal-before-confirmation race/exact-source return/P4 handoff.

Accepted proof summary:

- dedicated P3-001 canonical PostgreSQL suite **13/13 PASS** after supervisor-discovered multi-line idempotency remediation;
- multi-line cancellation persists one request-level ReservationRelease and replays with zero duplicate release/ATP restoration;
- P1-004 **11/11**, P1-001 **7/7**, P1-005 **10/10**, P1-016 **25/25** retained green;
- typecheck/build/runtime proof green;
- rendered Mercato Playwright **3/3 PASS**, zero route mocks;
- Scanner remained frozen.

## Active next item

**P3-002 — Reservation retention policy and automatic release timer — item 27/37.**

Authoritative guide:

`06_AGENT_GUIDES/P3-002_EXECUTION.md`

Guide commit:

`b299f9d66ff8438ae2f69a0cc068ac06455b17f7`

Frozen accepted bases:

- Mercato P3-001 `9fb32493ed9ec443a18a494aa1a8ec3a1bde6d06`;
- Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- P3-001 evidence `15c3ad937a4e81d7b67ff96409bd0b6a65553864`.

### P3-002 Architect truth

Grounding:

- P3 R9: warehouse partial-reservation policy has exactly three variants — retain reservation, automatic release after configured time, or Warehouse Supervisor decision;
- P3 R10: configured retention time is independent of `priority` and `slaDeadline`;
- requirements `FR-P3-05`, `FR-P3-06`;
- acceptance `TC-112`, `TC-113`.

Hard boundary:

- P3-002 owns policy/config/timer/Supervisor-decision orchestration only;
- actual release reuses accepted P3-001 rather than duplicating release logic;
- formal picked quantity remains outside P3;
- no P3-003 exact-source RF race behavior and no P4 PutBack behavior;
- no new priority/SLA semantics;
- Scanner has no mapped P3-002 workflow and stays frozen unless authoritative scope proves otherwise.

## Mandatory new-item reset

Before first P3-002 implementation action, perform the canonical new-item Testing reset defined in `Devaxonic-WMS/.ai/OPERATIONS.md` and require `RESET_OK`.

The reset is environment hygiene and is separate from executor launch. Do not repeat it for ordinary P3-002 retries/continuations.

## Prompt-generation and executor mode

Durable prompt routing:

`06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`

current steering commit: `a52587ddb1efcaae5babc0e0bd3a8e3c99f67942`.

Before any executor handoff, supervisor refreshes current WMS/Architect authority and applies current `wms-outbound`, `architecture-context` when shared compatibility is relevant, plus `fetch_me_prompt` + `operational-mode`.

All authorized executors own the **whole authorized item** through implementation/tests/runtime/UI/evidence/push and return only COMPLETE or a true same-material-path two-strikes blocker.

Owner controls executor selection/launch/session organization. Owner-facing handoff contains prompt content only; do not combine it with shell launcher, VPN or session-start commands unless Owner explicitly requests them.

## Fresh supervisor bootstrap

A new supervisor chat must not reconstruct project state from chat memory alone.

Refresh:

1. current Drive handover/memory;
2. current `Devaxonic-WMS` steering/state/testing/operations/handover;
3. current `WMS_Outbound` AGENTS/STATE/handover/prompt workflow/routing;
4. current `wms-outbound` context;
5. exact Architect/Canon for the active item;
6. `architecture-context` for shared/Inbound compatibility only;
7. `scanner-context` only when Scanner is materially relevant;
8. current `fetch_me_prompt` + `operational-mode` before composing executor handoff.

**Git truth overrides stale Drive/chat history.**

## Supervisor protocol

- executor never self-accepts;
- supervisor independently verifies final refs/diff/evidence;
- formal catalog progress advances only after explicit Owner acceptance;
- owner-facing executor prompt stays microscopic because detailed work lives in Git;
- no P3-003 before P3-002 is supervisor-verified and Owner Accepted;
- Demo/Prod and local PostgreSQL remain out of scope.