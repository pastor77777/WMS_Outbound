# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-06 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Formal progress: **31/37 FINAL PASS / Owner Accepted**.

Latest accepted checkpoints:

- `P4-001` — catalog item 29/37 — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`.
- `P3-003` — catalog item 28/37 — Mercato `f600782496e865603200d46d7e6041a54f90b9a4` / Scanner `135d86e1342bae7b21a8b676b1ef220a39a0f0b5` / evidence `40d4fd10430ab2b58457f96573ed34af8f57d316`.
- `P4-002` — catalog item 30/37 — Mercato `d75dccbc7a43b64c2a0ed325d30a7a4b49b57eb1` / Scanner `7d13e34fc66fe19149b43a6747b5300c2bdcf945` / evidence `0961a7fe2e08395cd1b2522e30770d62c2fb2841`.
- `P4-003` — catalog item 31/37 — **FINAL PASS / Owner Accepted** — Mercato product candidate `6ddec6870b5498901224b3457f5955101204a2f0` / Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8a916dfc9387b09d92b9e9143010cc34145108bd`.

## P4-003 accepted truth

- PutBack destination submission performs the authoritative server-side `IN_PROGRESS -> LOCATION_VALIDATION` transition.
- Invalid/non-storage destination returns to `IN_PROGRESS` with zero Inventory movement; retry is unlimited and there is no automatic escalation.
- Valid completion atomically performs `LOCATION_VALIDATION -> COMPLETED`, exact physical Inventory recovery, shared task-lock release and resolution of the corresponding P4-001 physical-return handoff.
- The handoff remains unresolved/protecting quantity before completion and on reject/error/rollback; `resolved_at` disables that temporary subtraction only once completion succeeds.
- Duplicate completion is idempotent; real concurrent completion is serialized by PostgreSQL locking; forced rollback leaves task, Inventory, task lock and handoff resolution unchanged.
- P4-002 FIFO/no-zone/single-active-task behavior remains preserved.
- No Inbound Putaway business states/flows were imported.
- Dedicated PostgreSQL: **10/10 PASS**.
- Mandatory targeted regressions: **133/133 PASS**.
- Rendered P4-003 + P4-002 regression: **5/5 PLAYWRIGHT VERIFIED**, zero route mocks.

## Post-acceptance maintenance completed

The two concrete test-infrastructure gaps discovered during P4-003 verification were fixed separately after Owner authorization:

- Mercato `outbound/p4-003` maintenance commit `9a656bf9e5a42e29b6493f1b7bed5d62b7bf562c` adds `WmsOutboundPutBackTask` to the isolated MikroORM entity registries in exactly:
  - `p3-002-postgres.integration.test.ts` — **17/17 PASS**;
  - `p1-003-detail-api-postgres.integration.test.ts` — **1/1 PASS**.
- WMS tracking record updated in `04_CURRENT_STATE/TEST_INFRA_GAPS.md` @ `6cde5bde465a1a4e10d41d9e04ccff443720c72c`.
- No product code changed in this maintenance fix.

## Exact next item — grounded, not launched

**X-001 — Enforce CON-01..05 concurrency and exactly-once business effects** — catalog item **32/37**.

Catalog objective: apply/verify explicit transactions, row/advisory locks, uniqueness and idempotency at every authoritative boundary covered by `CON-01..05`. Dependencies `P1-004`, `P1-011`, `P1-014`, `P1-015`, `P2-002` are already accepted.

Grounded behavior:

1. `CON-01` — parallel ATP/allocation competition cannot reserve more than available ATP.
2. `CON-02` — existence of a PickTask freezes its assigned quantity against reallocation.
3. `CON-03` — one physical source Cross-Dock TU/SKU quantity cannot enter two active/completed assignments.
4. `CON-04` — concurrent Shipment grouping uses a stable deadline/boundary and one Packing TU/package cannot enter two Shipments.
5. `CON-05` — duplicate/concurrent ERP and manifest effects settle once and never regress terminal state.

## Existing accepted implementation evidence to preserve

X-001 is a hardening/audit item, not a rewrite:

- CON-01: current P1-004 has genuine two-transaction limited-stock competition with PostgreSQL advisory-lock wait proof and `sum(reserved) <= stock`.
- CON-02: current P1-005 has a server-authoritative PickTask immutability guard. X-001 must audit whether the existing proof is a genuine overlapping race; harden only if it is not.
- CON-03: current P2-002 contains corrected quantity-lock/unique-assignment PostgreSQL coverage. Verify the exact overlap/evidence quality before changing product code.
- CON-04: P1-011 has real grouping-key lock contention and rollback proof; P1-015 has one-manifest and add-vs-close races.
- CON-05: P1-014 has real in-flight/duplicate ERP posting exactly-once proof; P1-015 has duplicate/parallel manifest-confirm exactly-once proof. Include P1-016/P2-006 settlement side effects in the audit where their final effect can duplicate.

The Definition of Done is not “rewrite all concurrency”. Each of the five CON requirements must finish with a decisive executable race/idempotency test at the owning DB/server boundary. Existing accepted proofs may satisfy that requirement if they are genuinely decisive; add or repair only missing guards/evidence. A stronger test exposing a real bug makes that bug in-scope self-repair.

## Grounding defect: stale source-number references

The derived requirements/index currently points `CON-04` to P1 R37–R41 and `CON-05` to P1 R43–R46. Those references are stale after later P1 renumbering and must not drive implementation literally.

Authority hierarchy resolves this without a product decision:

- current Shipment grouping behavior is in P1 STEP 9 / R26–R29, with R39–R40 covering singular manifest membership/irreversible close where applicable;
- ERP/manifest exactly-once concern belongs to the current posting/manifest/final-settlement flow around P1 R37–R40 and R70–R72, while the explicit `CON-05` requirement supplies the concurrency/duplicate constraint;
- current R43–R46 are SHORT_ALLOCATED/SHORT_PICKED rules and are not CON-05 behavior.

Classification: **traceability-reference defect, not architecture/product blocker**. Do not mutate immutable Architect snapshots to conceal it.

## Architecture-context / shared compatibility boundary

`architecture-context` / WMS-Records remains Inbound/shared reference only. Allowed reuse is technical: transaction shape, record/advisory locks, idempotency, uniqueness, warehouse context and rollback patterns. No Inbound business state/process semantics may be imported into Outbound.

If X-001 changes shared Inventory/TU/warehouse/record-lock/orchestration primitives, run the relevant accepted Inbound regressions. Do not introduce a new generic lock subsystem if accepted primitives already satisfy the boundary.

## Expected execution surface

- Primary repo: Mercato.
- Scanner stays frozen by default; touch it only if a real product-visible conflict/retry defect is found.
- No new UI journey is required merely because X-001 exists. Use Playwright only when X-001 changes user-visible conflict/retry behavior. Real PostgreSQL overlap/idempotency/rollback proof is the decisive default evidence.
- No new external integration semantics.

Execution bases:

- Mercato `outbound/p4-003` @ `9a656bf9e5a42e29b6493f1b7bed5d62b7bf562c`.
- Scanner `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.

## Next execution boundary

Grounding is complete. **X-001 has not been launched.**

Before first implementation action after Owner authorization:

1. refresh current Git;
2. for a fresh Claude session load current `fetch_me_prompt` + `operational-mode` first, then project steering + `wms-outbound` + `architecture-context` (`scanner-context` only if Scanner becomes relevant);
3. run the canonical new-item Testing reset from `Devaxonic-WMS/.ai/OPERATIONS.md` and require `RESET_OK`;
4. write/update the detailed X-001 Git execution guide;
5. use the Owner-selected executor with a microscopic handoff;
6. executor owns ordinary in-scope defects/tests/build/evidence through self-repair;
7. supervisor independently verifies; Owner Acceptance remains separate.

Do not start X-002 automatically.

## Durable operating rules

- Executor `COMPLETE` != Supervisor PASS != Owner Acceptance.
- Owner controls executor/venue/launcher/session mechanics.
- Provider/session/quota interruption preserves branch/HEAD/workspace checkpoint.
- Testing credentials are frozen; do not audit/rotate/refactor them during implementation.
- Inbound remains CLOSED / REFERENCE.
- Demo/Prod remain out of scope unless explicitly authorized.

**Git truth overrides stale Drive/chat history.**