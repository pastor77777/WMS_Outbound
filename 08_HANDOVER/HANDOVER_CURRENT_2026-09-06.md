# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-06 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Formal progress: **28/37 FINAL PASS / Owner Accepted**.

Latest accepted implementation checkpoints:

- `P3-002` — FINAL PASS / Owner Accepted — Mercato `84274acacfbfe0119e270ca5bfbcb723e47d7723` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`.
- `P4-001` — catalog item 29/37 — FINAL PASS / Owner Accepted — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`.

P4-001 was intentionally executed before P3-003 because Task Catalog declares `P4-001` as a dependency of `P3-003`. That dependency is now satisfied.

## Accepted boundary to preserve

### P3 pre-pick

- P3-001 release applies only before formal pick; accepted P3-001 evidence is `15c3ad937a4e81d7b67ff96409bd0b6a65553864`.
- P3-002 retention policy reuses P3-001 and does not change the formal-pick discriminator.

### P4 formal pick

Accepted P4-001 boundary:

- formal pick begins only on actual pick confirmation; task generation alone does not put `OutboundOrderLine` in `PICKING`;
- partial formal pick keeps standard Allocation `RESERVED`; full pick transitions Allocation `RESERVED -> CONFIRMED`;
- post-pick/post-pack cancellation performs immediate logical settlement and preserves physical-return correlation;
- physically picked stock awaiting later P4 recovery is excluded from ATP/allocation by the unresolved physical-return handoff;
- P4-001 creates no P4-002 PutBackTask execution and no P4-003 physical `PICKED -> AVAILABLE` recovery;
- late Shipment/CarrierManifest cancellation boundaries remain intact.

P4-001 accepted proof includes dedicated PostgreSQL **18/18**, P1-014 **18/18**, P1-015 **21/21**, P1-016 **25/25**, P1-006/P1-004/P3-001/P3-002 regressions green, and rendered Mercato Playwright **3/3**.

## Active item

**P3-003 — Cancellation race: physical movement before formal confirmation — catalog item 28/37.**

Authoritative guide:

`06_AGENT_GUIDES/P3-003_EXECUTION.md`

Guide commit:

`80613b4fc516be66b34d200b3375e4b16217ea4f`

Accepted bases:

- Mercato `66e2e8620041d2db1d10d069e286936083667139`;
- Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- P4-001 evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`;
- P3-002 evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`;
- P3-001 evidence `15c3ad937a4e81d7b67ff96409bd0b6a65553864`.

### P3-003 Architect truth

Grounding:

- P3 R7-R8;
- requirement `FR-P3-04` and `INT-06`;
- acceptance `TC-042`, `TC-043`;
- P4 process boundary and accepted P4-001 implementation once formal `pickedQty > 0` exists.

Hard behavior:

1. Race window is after source-location + SKU verification/physical removal but before formal Picking TU + quantity confirmation.
2. In that window authoritative `pickedQty` remains `0`; pre-confirm observation itself must not create formal pick state, TU content or Allocation confirmation.
3. If cancellation wins, reuse accepted P3-001, cancel eligible PickTask work and instruct the Scanner operator to return the SKU to the **exact original source location**, with no PutBackTask and no target-location selection/validation.
4. If formal confirmation wins first and `pickedQty > 0`, no P3 exact-source instruction is created; cancellation routes to accepted P4-001.
5. Server/PostgreSQL locking determines the winner; Scanner-local state cannot.
6. Persist only the minimum technical correlation needed for the race/instruction; do not invent a new business state machine.
7. Scanner is materially in scope: current code submits source location/SKU/quantity together, so P3-003 must expose a real pre-confirm server-visible step and rendered RF return instruction.
8. P4-002/P4-003 remain forbidden in this item.

Scanner reference guidance is limited to server authority, idempotency, ordered context-sensitive operations and human-readable task/location/SKU feedback. Outbound Architect truth wins.

## Testing / credentials

Canonical Testing contract is `Devaxonic-WMS/.ai/TESTING.md`.

Testing credential handling is frozen for implementation. Do not audit, refactor, relocate, redact, rotate, replace or redesign designated Testing credentials or create a blocker on that basis. Final credential rotation is a separate Owner-controlled production-cutover activity.

P3-003 requires real PostgreSQL race/concurrency/rollback proof and real rendered Scanner + Mercato acceptance as defined in the guide.

## New-item reset

Before first P3-003 implementation action, perform the canonical new-item Testing reset defined in `Devaxonic-WMS/.ai/OPERATIONS.md` and require successful `RESET_OK`.

The reset is separate from executor launch and is not repeated for ordinary continuations within P3-003.

## Executor selection and prompt routing

Owner selected **Codex** for P3-003 because Antigravity is temporarily unavailable by quota. This changes only the executor venue, never business scope/evidence rules.

Before the handoff, supervisor refreshed current `wms-outbound`, `scanner-context`, `fetch_me_prompt` and `operational-mode`, plus current Devaxonic-WMS/WMS steering and exact Architect sources.

Durable routing:

- `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
- `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`

Owner-facing prompt remains microscopic and contains no launcher/VPN/session-start command.

## Supervisor protocol

- executor owns the whole P3-003 item and ordinary execution failures;
- two-strikes only for the same material unresolved technical path after two substantive attempts;
- executor never self-accepts;
- supervisor independently verifies final Mercato/Scanner/WMS refs, diff scope and evidence;
- only explicit Owner acceptance advances progress;
- do not start P4-002 or another item before P3-003 verification/acceptance;
- Demo/Prod and local PostgreSQL remain out of scope.

**Git truth overrides stale Drive/chat history.**