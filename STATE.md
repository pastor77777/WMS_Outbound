# WMS Outbound — STATE

**As of:** 2026-09-06  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **29/37 items FINAL PASS / Owner Accepted**

## Architect baseline

Active process set:

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19

Requirements: **109 IDs = 98 FR + 6 INT + 5 CON**.

`PickWave` is out of scope v1. No separate Process 5 exists.

## Accepted implementation checkpoints

Accepted checkpoints 1–25 remain unchanged in Git history and prior STATE revisions.

26. `P3-001` — FINAL PASS / Owner Accepted — Mercato `9fb32493ed9ec443a18a494aa1a8ec3a1bde6d06` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `15c3ad937a4e81d7b67ff96409bd0b6a65553864`
27. `P3-002` — FINAL PASS / Owner Accepted — Mercato `84274acacfbfe0119e270ca5bfbcb723e47d7723` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`
28. `P4-001` — catalog item 29/37 — FINAL PASS / Owner Accepted — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`
29. `P3-003` — catalog item 28/37 — FINAL PASS / Owner Accepted — Mercato `f600782496e865603200d46d7e6041a54f90b9a4` / Scanner `135d86e1342bae7b21a8b676b1ef220a39a0f0b5` / evidence `40d4fd10430ab2b58457f96573ed34af8f57d316`

## P3-003 accepted boundary

- Pre-confirm race is after source location + SKU verification / possible physical removal but before formal Picking TU + quantity confirmation.
- Pre-confirm observation is technical durable correlation only; it does not create a new business state machine.
- Before formal confirmation authoritative `pickedQty = 0` and Allocation remains `RESERVED`.
- If P3 cancellation wins, accepted P3-001 release executes exactly once, eligible task work is cancelled, Allocation becomes `RELEASED`, Scanner shows return to the exact original source location, and zero PutBackTask / zero P4 physical-return handoff is created.
- If formal confirmation wins, observation becomes non-actionable/confirmed, `pickedQty > 0`, Allocation follows accepted formal-pick lifecycle, and cancellation routes to accepted P4-001 with no P3 exact-source instruction.
- Concurrency is server/PostgreSQL authoritative. Dedicated proof uses independent overlapping real transactions and PostgreSQL-side advisory-lock blocking evidence for both winner orders.
- Real rollback proof performs write/flush, deterministic failure before commit, then fresh independent read proving unchanged state.
- Rendered TC-042/TC-043 use normal Scanner + Mercato UI with zero route mocks. Corrected acceptance proves Allocation lifecycle `RESERVED -> RELEASED` for TC-042 and `RESERVED -> CONFIRMED -> RELEASED` for TC-043.
- P4-002/P4-003 remain outside P3-003.

Evidence summary:

- dedicated P3-003 PostgreSQL: **14/14 PASS**;
- decisive/regression integration total: **128/128 PASS**;
- rendered TC-042/TC-043: **2/2 PLAYWRIGHT VERIFIED**;
- Mercato generate/typecheck/build and Testing runtime proof green per evidence.

## Current position

Completed and accepted: **29/37**.

**No next implementation item is authorized in this chat.** The fresh supervisor chat must determine the exact next Task Catalog item from current Git/Architect authority, then obtain/recognize Owner authorization for that item before implementation handoff.

Do not infer the next item from old chat memory alone.

## Executor / prompt workflow — durable rules

Detailed workflow: `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

Prompt skill routing: `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`.

Current durable corrections from P3-003:

- full Task Catalog item is the normal Owner-authorized executor unit when project steering says so;
- Codex full-item work should use supported long-horizon `/goal` mode when available, with one verifiable COMPLETE condition;
- `incomplete`, `unevidenced`, ordinary regression, missing test/evidence, or execution-window/quota interruption is not by itself a valid technical blocker;
- ordinary in-scope failure stays in executor self-repair: diagnose -> patch -> rerun relevant check -> continue;
- provider/session/quota interruption preserves a durable checkpoint; it does not restart the item from accepted base;
- switching Codex <-> Antigravity is continuation from exact branch/HEAD/workspace state, never a product restart;
- executor prose is never acceptance; supervisor independently verifies remote Git/evidence;
- Owner acceptance is explicit after Supervisor FINAL PASS;
- Owner controls executor launch/session mechanics; owner-facing handoff contains prompt content only unless Owner explicitly asks for launcher details;
- designated Testing credential handling is frozen during implementation and is not a review/blocker topic; final rotation is Owner-controlled production cutover only.

Shared prompt skill correction is persisted in `pastor77777/CLAUDE-SKILLS/fetch_me_prompt/SKILL.md` commit `a94a97c868e79e5bd1177754b818f994bbf01dea`.

Project prompt routing correction is persisted in `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`; current Git truth must be refreshed by the fresh chat.

## Mandatory new-item reset

For the next newly authorized item, perform the canonical reset from `Devaxonic-WMS/.ai/OPERATIONS.md` and require `RESET_OK` before its first implementation action.

Reset is separate from executor launch and is not repeated for same-item retries/continuations.

## Authority and continuity

For Outbound behavior: immutable Architect Source -> faithful Canon/translation -> requirements/traceability -> Task Catalog delivery slice -> current code/DB as implementation evidence.

Inbound remains **CLOSED / REFERENCE**.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.