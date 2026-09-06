# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-06 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Formal progress: **29/37 FINAL PASS / Owner Accepted**.

Latest accepted checkpoints:

- `P3-001` — Mercato `9fb32493ed9ec443a18a494aa1a8ec3a1bde6d06` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `15c3ad937a4e81d7b67ff96409bd0b6a65553864`.
- `P3-002` — Mercato `84274acacfbfe0119e270ca5bfbcb723e47d7723` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`.
- `P4-001` — catalog item 29/37, executed early as P3-003 dependency — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`.
- `P3-003` — catalog item 28/37 — Mercato `f600782496e865603200d46d7e6041a54f90b9a4` / Scanner `135d86e1342bae7b21a8b676b1ef220a39a0f0b5` / evidence `40d4fd10430ab2b58457f96573ed34af8f57d316` — **FINAL PASS / Owner Accepted**.

## P3-003 accepted truth

- R7 race window is after source-location + SKU verification / possible physical removal but before formal Picking TU + quantity confirmation.
- Pre-confirm observation is technical correlation only, not a new Outbound business state machine.
- Before formal confirmation `pickedQty = 0` and Allocation remains `RESERVED`.
- Cancellation winning first executes accepted P3-001 release exactly once, cancels eligible PickTask work, changes Allocation `RESERVED -> RELEASED`, and Scanner renders exact-original-source return instruction. No PutBackTask and no P4 physical-return handoff are created.
- Formal confirmation winning first settles the observation, creates authoritative `pickedQty > 0`, follows accepted formal-pick Allocation lifecycle, and later cancellation routes to accepted P4-001 with zero P3 exact-source instruction.
- Server/PostgreSQL arbitration is authoritative. Dedicated concurrency A/B proof uses independent overlapping transactions, distinct PostgreSQL PIDs and PostgreSQL-side advisory-lock blocking proof.
- Dedicated rollback proof writes/flushes, fails before commit, then proves unchanged state using a fresh independent read.
- Dedicated PostgreSQL P3-003 suite: **14/14 PASS**.
- Decisive/regression integration total: **128/128 PASS**.
- Rendered normal human-flow acceptance: **TC-042 + TC-043 = 2/2 PLAYWRIGHT VERIFIED**, zero route mocks.
- Corrected rendered lifecycle: TC-042 `RESERVED -> RELEASED`; TC-043 `RESERVED -> CONFIRMED -> RELEASED`.
- P4-002/P4-003 were not fabricated by P3-003.

## Executor incident lessons now durable

P3-003 exposed repeated invalid Codex terminal responses such as ordinary regression blockers, `incomplete/unevidenced` status-only stops and execution-window stops. These are now durable steering, not chat-only lessons.

Authoritative project routing:

- `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`
- `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

Shared prompt skill:

- `pastor77777/CLAUDE-SKILLS/fetch_me_prompt/SKILL.md`
- full-item override update commit: `a94a97c868e79e5bd1177754b818f994bbf01dea`

Required behavior for future full-item execution when project steering defines the Task Catalog item as the executor unit:

1. Whole authorized item is one end-to-end goal: implementation -> self-repair -> real tests -> regressions -> build/runtime -> rendered UI -> evidence -> push.
2. Codex should use supported `/goal` long-horizon mode for such items when available.
3. `incomplete`, `unevidenced`, missing tests/evidence, ordinary in-scope regression, or execution-window/quota interruption is not a valid technical blocker by itself.
4. Ordinary failures stay in the executor self-repair loop while a normal in-scope correction exists.
5. A true blocker requires the same material unresolved path after two genuinely different substantive attempts, or an Owner-controlled boundary.
6. Provider/session/quota interruption means preserve checkpoint and continue; never restart the item from accepted base.
7. Codex <-> Antigravity switch is same-item continuation from exact branch/HEAD/workspace state, not restart/redesign.
8. Executor prose is never acceptance. Supervisor independently verifies remote Git/evidence.
9. Owner controls launch/session mechanics. Never invent or replace the Owner's executor launcher command.
10. Testing credential hygiene is out of implementation scope; do not audit/rotate/refactor credentials or create blockers on that basis. Final rotation is Owner-controlled production cutover.

## Fresh-chat boundary

**No next implementation item is authorized in this old chat.**

The fresh supervisor chat must:

1. read current Drive handover + `ChatGPT_MEMORY.md`;
2. refresh current Git steering in Devaxonic-WMS and WMS_Outbound;
3. load current `wms-outbound`, and `scanner-context`/`architecture-context` only when relevant;
4. before any executor prompt, load current `fetch_me_prompt + operational-mode`;
5. inspect current Task Catalog/Architect sources and determine the exact next unaccepted item from Git truth;
6. keep P3-003 accepted/frozen at the SHAs above;
7. perform the canonical new-item Testing reset only after the next item is identified/authorized, separately from executor launch;
8. then prepare the next item's detailed Git guide and microscopic Owner-facing handoff.

Do not reconstruct the next item from this old conversation. Do not start implementation from Drive memory alone.

## Acceptance semantics

- Executor `COMPLETE` != accepted.
- Supervisor `FINAL PASS` != Owner Accepted.
- Only explicit Owner acceptance increments the formal count.
- Current formal accepted count is **29/37**.
- Inbound remains CLOSED / REFERENCE.
- Demo/Prod remain outside implementation unless explicitly authorized.

**Git truth overrides stale Drive/chat history.**