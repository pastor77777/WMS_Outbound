# WMS Outbound — STATE

**As of:** 2026-09-06  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **31/37 items FINAL PASS / Owner Accepted**

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
31. `P4-003` — catalog item 31/37 — **FINAL PASS / Owner Accepted** — Mercato product candidate `6ddec6870b5498901224b3457f5955101204a2f0` / Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8a916dfc9387b09d92b9e9143010cc34145108bd`.

## P4-003 accepted boundary

- RF PutBack supports destination submission and server validation: `IN_PROGRESS -> LOCATION_VALIDATION`.
- Invalid/non-storage destination returns `LOCATION_VALIDATION -> IN_PROGRESS`, with zero Inventory mutation, unlimited retry and no automatic escalation.
- Valid destination completes exactly once: `LOCATION_VALIDATION -> COMPLETED`, exact physical recovery into ordinary Inventory availability, shared active-task lock release, and physical-return handoff resolution in the same transaction.
- P4-001 protection remains active while the handoff is unresolved; after completion `resolved_at` removes that exact quantity from the temporary protection subtraction used by ATP/allocation/source availability.
- Duplicate completion is idempotent; genuine concurrent completion serializes on PostgreSQL row locking; rollback leaves task, Inventory, lock and handoff resolution unchanged.
- No Inbound Putaway business semantics were imported.
- Dedicated PostgreSQL P4-003: **10/10 PASS**.
- Mandatory targeted regression set: **133/133 PASS**.
- Rendered Scanner acceptance plus P4-002 rendered regression: **5/5 PLAYWRIGHT VERIFIED**, zero route mocks.

## Post-acceptance test-infra maintenance

Owner-authorized maintenance fixed the two known `WmsOutboundPutBackTask` MikroORM registration gaps discovered during P4-003 verification:

- Mercato maintenance commit `9a656bf9e5a42e29b6493f1b7bed5d62b7bf562c`.
- `p3-002-postgres.integration.test.ts`: **17/17 PASS**.
- `p1-003-detail-api-postgres.integration.test.ts`: **1/1 PASS**.
- Tracking record: `04_CURRENT_STATE/TEST_INFRA_GAPS.md` @ `6cde5bde465a1a4e10d41d9e04ccff443720c72c`.

This maintenance does not change the accepted P4-003 product behavior or Task Catalog count.

## Current position — X-001 grounded, not launched

Completed and accepted: **31/37**.

Exact next Task Catalog item: **X-001 — Enforce CON-01..05 concurrency and exactly-once business effects** (catalog item **32/37**).

Grounded target:

- `CON-01` — concurrent ATP/allocation competition must never reserve more than available stock.
- `CON-02` — once a `PickTask` exists, its assigned quantity cannot be reallocated.
- `CON-03` — the same source Cross-Dock TU/SKU quantity may be planned/confirmed at most once.
- `CON-04` — concurrent Shipment grouping must use one stable grouping boundary and each Packing TU/package may enter at most one Shipment.
- `CON-05` — duplicate/concurrent ERP posting response and manifest close/confirm paths must produce one business effect and never regress terminal state.

Dependencies listed by the catalog (`P1-004`, `P1-011`, `P1-014`, `P1-015`, `P2-002`) are already accepted.

This item is **hardening/audit of existing authoritative DB/server boundaries**, not permission to redesign accepted flows. Reuse accepted transactions, row/advisory locks, uniqueness and idempotency mechanisms. Do not refactor green code merely for uniformity; change product code only where a decisive race/idempotency test exposes a real gap.

Existing accepted evidence already covers substantial parts of the target:

- CON-01: P1-004 has genuine competing-allocation PostgreSQL overlap with advisory-lock wait and `sum(reserved) <= stock` proof.
- CON-02: P1-005 has the accepted PickTask immutability guard; X-001 must verify whether its evidence is a decisive real overlap race and harden only if needed.
- CON-03: P2-002 has corrected PostgreSQL quantity-lock/unique-assignment coverage; X-001 must verify the exact overlap proof rather than reimplement it.
- CON-04: P1-011 has real Shipment grouping lock contention/rollback evidence; P1-015 has add-vs-close and singular-membership races.
- CON-05: P1-014 has genuine ERP-posting duplicate/in-flight exactly-once proof; P1-015 has duplicate/parallel manifest-confirm and add/close serialization proof. P1-016/P2-006 settlement effects must be included in the final audit where relevant.

### Grounding note — stale traceability references

The current derived requirement/index layer carries stale source-number references for `CON-04`/`CON-05` after later P1 renumbering. This is a **traceability-reference defect, not a product blocker**. Active P1 process prose remains higher authority:

- Shipment grouping behavior for CON-04 is currently anchored in STEP 9 / P1 R26–R29 (with one-manifest boundary R39–R40 where relevant).
- ERP/manifest single-effect behavior for CON-05 is anchored in the current ERP/manifest/settlement flow around P1 R37–R40 and R70–R72, while the explicit CON-05 requirement defines the duplicate/concurrent exactly-once constraint.

Do not edit immutable Architect snapshots to hide the mismatch and do not implement SHORT_PICKED rules R43–R46 as concurrency behavior merely because the stale derived reference points there.

## X-001 compatibility / scope boundary

- `architecture-context` / WMS-Records is shared/Inbound reference only. Reuse generic transaction, lock, idempotency and warehouse-context patterns technically; do not import Inbound business states/process rules into Outbound.
- Mercato is the primary expected product repo. Keep Scanner frozen unless a real X-001 conflict-response defect requires a change.
- No new human workflow or UI is implied by X-001. Playwright is required only if product-visible conflict/retry behavior changes; DB concurrency/idempotency proof is the decisive default evidence class.
- Any touched shared Inventory/TU/warehouse/record-lock/orchestration primitive requires the corresponding accepted Inbound regression evidence.
- No new external integration behavior; make duplicates/concurrency safe at existing bounded-context seams.

Continuation bases for future execution:

- Mercato `outbound/p4-003` @ `9a656bf9e5a42e29b6493f1b7bed5d62b7bf562c` (accepted P4-003 product lineage plus test-infra-only maintenance).
- Scanner `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.

Grounding does **not** launch X-001. Before first implementation action, explicit Owner authorization and the canonical new-item Testing reset from `Devaxonic-WMS/.ai/OPERATIONS.md` are required; reset must end `RESET_OK`.

Do not start X-002 automatically after X-001.

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

## Authority and continuity

For Outbound behavior: immutable Architect Source -> faithful Canon/translation -> requirements/traceability -> Task Catalog delivery slice -> current code/DB as implementation evidence.

Inbound remains **CLOSED / REFERENCE**.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.