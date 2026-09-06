# WMS Outbound — STATE

**As of:** 2026-09-06  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **30/37 items FINAL PASS / Owner Accepted**

## Architect baseline

Active process set:

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19

Requirements: **109 IDs = 98 FR + 6 INT + 5 CON**.

`PickWave` is out of scope v1. No separate Process 5 exists.

## Latest accepted implementation checkpoints

Earlier accepted checkpoints 1–26 remain unchanged in Git history.

27. `P3-002` — FINAL PASS / Owner Accepted — Mercato `84274acacfbfe0119e270ca5bfbcb723e47d7723` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`
28. `P4-001` — catalog item 29/37 — FINAL PASS / Owner Accepted — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`
29. `P3-003` — catalog item 28/37 — FINAL PASS / Owner Accepted — Mercato `f600782496e865603200d46d7e6041a54f90b9a4` / Scanner `135d86e1342bae7b21a8b676b1ef220a39a0f0b5` / evidence `40d4fd10430ab2b58457f96573ed34af8f57d316`
30. `P4-002` — catalog item 30/37 — FINAL PASS / Owner Accepted — Mercato `d75dccbc7a43b64c2a0ed325d30a7a4b49b57eb1` / Scanner `7d13e34fc66fe19149b43a6747b5300c2bdcf945` / evidence `0961a7fe2e08395cd1b2522e30770d62c2fb2841`

## P4-002 accepted boundary

- One positive unresolved P4-001 physical-return handoff materializes exactly one durable `PutBackTask`; replay and genuine concurrent materialization remain idempotent through DB uniqueness plus winner re-read.
- `PutBackTask` carries exact warehouse/order-line/SKU/quantity/TU/source correlation and has no `priority`/`slaDeadline`.
- RF returns module has no zone selection. Assignment is strict FIFO by task arrival with stable tie-break and shared no-active-warehouse-task protection.
- Two-operator assignment is PostgreSQL-authoritative; accepted proof uses separate backend PIDs and live advisory-lock wait evidence.
- Executable lifecycle accepted in this item: `CREATED -> ASSIGNED -> IN_PROGRESS`; wrong owner/illegal transitions reject with zero business mutation.
- P4-002 does not validate destination, complete task, move physical Inventory to AVAILABLE, or resolve the P4-001 handoff protection. Those belong to P4-003.
- Dedicated PostgreSQL: **17/17 PASS**, stable across 3 reruns after concurrency hardening.
- Regressions: P4-001 **18/18**, P3-003 **14/14**, P1-005/P1-006/P2-002/FND-003 **54/54**.
- Rendered Scanner/Mercato acceptance: **4/4 PLAYWRIGHT VERIFIED**, zero route mocks.

## Current position

Completed and accepted: **30/37**.

Exact next unaccepted implementation item from current Task Catalog: **P4-003 — RF PutBack location validation loop and Inventory recovery**.

Grounded authority for P4-003:

- P4 STEP 4–5, especially `P4 R6`, `P4 R7`, `P4 R8`; preserve `P4 R9` assignment/no-zone behavior from accepted P4-002.
- Requirements: `FR-P4-03`, `FR-P4-04`, `FR-P4-05`.
- Acceptance scenarios: `TC-050`, `TC-051`, `TC-052`, `TC-100`.
- Exact target: proposed/operator location -> server validation -> unlimited rejection loop with no auto escalation -> valid physical completion -> exact `Inventory PICKED -> AVAILABLE` once.
- Inbound/WMS-Records is reference only: shared location/Inventory/locking primitives may be reused technically, but Inbound Putaway statuses, zone routing, sector `TRANSIT`, `TransportTask`, settlement/progress and GR semantics must not be imported into Outbound PutBack.

**Grounding does not itself launch P4-003.** Owner authorization and the canonical new-item Testing reset are required before first implementation action.

## Executor / prompt workflow — durable rules

Detailed workflow: `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

Prompt skill routing: `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`.

- full Task Catalog item is the normal Owner-authorized executor unit when project steering says so;
- ordinary in-scope implementation/test/runtime/build/evidence failures remain executor self-repair;
- provider/session/quota interruption preserves exact checkpoint;
- executor prose is never acceptance; supervisor independently verifies remote Git/evidence;
- Owner acceptance is explicit after Supervisor FINAL PASS;
- Owner controls executor launch/session mechanics;
- before a fresh Claude execution session, load current `fetch_me_prompt` + `operational-mode` and then project steering/skills;
- designated Testing credential handling remains frozen during implementation.

## Mandatory new-item reset

Before first implementation action of newly authorized P4-003, perform the canonical reset from `Devaxonic-WMS/.ai/OPERATIONS.md` and require `RESET_OK`.

Do not repeat the reset for retries/continuations of the same item.

## Authority and continuity

For Outbound behavior: immutable Architect Source -> faithful Canon/translation -> requirements/traceability -> Task Catalog delivery slice -> current code/DB as implementation evidence.

Inbound remains **CLOSED / REFERENCE**.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.
