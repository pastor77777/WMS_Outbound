# ACC-001 -> ACC-002 -> ACC-003 — one-time Owner-authorized acceptance batch

**Status:** current execution guide  
**Effective:** 2026-09-07  
**Scope:** Task Catalog items 34/37, 35/37, 36/37 in one executor session  
**Owner authorization:** one-time batch exception explicitly approved; no Supervisor round-trip between the three items  
**Accepted Mercato base:** `outbound/post-x002-raw-pg-ssl` @ `cfdbc608b22fc1dd44646c309335d2071aafd32c`  
**Accepted Scanner base:** `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`  
**Hard stop:** do not start `ACC-004`

## Purpose

Execute exactly these three standard Task Catalog items in one fresh Owner-selected executor session:

1. `ACC-001` — Complete automated component/integration requirement suite.
2. `ACC-002` — Playwright Standard Fulfillment + P1 exception journeys.
3. `ACC-003` — Playwright Crossdock, Reservation Release and Physical Putback journeys.

This is a **batch execution exception only**. The three catalog items remain separate acceptance units with separate evidence and separate internal completion gates. Executor COMPLETE for any phase is not Supervisor FINAL PASS or Owner Acceptance.

The normal dependency is preserved:

`ACC-001 -> ACC-002 -> ACC-003 -> STOP`

`ACC-003` does not technically depend on `ACC-002`, but this authorized batch deliberately executes them sequentially. Do not skip a failed phase and continue to a later one.

## Fresh-session bootstrap

Start from `/home/ubuntu/git/Devaxonic-WMS`.

Before doing work:

1. fast-forward all canonical repositories from origin:
   - `/home/ubuntu/git/Devaxonic-WMS`
   - `/home/ubuntu/git/WMS_Outbound`
   - `/home/ubuntu/git/Devaxonic-mercato`
   - `/home/ubuntu/git/Devaxonic-scanner`
2. read current Devaxonic-WMS `AGENTS.md`, `.ai/STATE.md`, `.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-06.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`;
3. read current WMS_Outbound `AGENTS.md`, `STATE.md`, current handover, `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md`, `05_EVIDENCE/EVIDENCE_STANDARD.md` and this guide;
4. load current `fetch_me_prompt` + `operational-mode`, then `wms-outbound` + `architecture-context`; load `scanner-context` because ACC-002/003 include Scanner acceptance;
5. verify the accepted base refs above before branching.

Current Git steering wins over stale chat/session/history.

## Batch branch strategy

Use one cumulative batch branch in each product repository that actually changes:

- Mercato: `outbound/acc-001-003-batch` from exact accepted Mercato base `cfdbc608b22fc1dd44646c309335d2071aafd32c`.
- Scanner: `outbound/acc-001-003-batch` from exact accepted Scanner base `a2759a29347285dd1dcd14bf51633431fbf2a302` if Scanner changes are required.

At the end of **each phase**, record the exact product HEAD(s) in that phase's evidence before continuing. Later phases may advance the same cumulative branch; the recorded checkpoint SHA is the durable item boundary.

Do not rewrite or squash away already-recorded phase checkpoints.

## Mandatory Testing hygiene between phases

The one-time batch exception does **not** waive `.ai/OPERATIONS.md` pre-item Testing hygiene.

Before **each** catalog item, run the canonical deep reset and require `RESET_OK`:

```bash
cd /home/ubuntu/git/Devaxonic-WMS && git pull --ff-only && bash scripts/reset-testing-runtime.sh --deep
```

Therefore this batch has three reset boundaries:

- before ACC-001;
- after ACC-001 internal gate and before ACC-002;
- after ACC-002 internal gate and before ACC-003.

Do not reset source branches/commits or Supabase. Canonical Testing only. No local PostgreSQL. Never print secrets.

# PHASE 1 — ACC-001

## Objective

Provide executable automated coverage for all **109/109 requirements** across Mercato, Scanner, DB integration, contract, race/concurrency and migration/shared-regression surfaces.

The machine-readable requirement-to-test matrix must have zero orphans and every claimed automated suite must pass on the exact phase checkpoint.

## Authority / grounding

Use Task Catalog `ACC-001`, current Architect/canon and traceability. Inbound remains CLOSED / REFERENCE; use `architecture-context` only for shared Inventory/TU/warehouse/lock/orchestration compatibility.

Do not weaken accepted X-001/X-002 concurrency/integration evidence or the accepted raw-PG SSL helper. Preserve the `confirmPickLine` authoritative idempotent-replay behavior from the accepted maintenance base.

## Required work

Audit current automated coverage before adding tests. Reuse valid existing evidence/tests; do not duplicate green coverage merely to create new filenames.

Close only actual gaps needed to prove:

- 109/109 requirement IDs have named executable automated coverage;
- state/quantity/transaction/idempotency/integration requirements are covered at the correct layer;
- `INT-01..06` and `CON-01..05` remain decisively automated;
- required migrations/constraints are verified;
- accepted Inbound/shared behavior has regression evidence wherever shared Inventory/TU/warehouse/lock/orchestration primitives are materially touched;
- Mercato and Scanner type/build/component/integration checks required by changed surfaces are green.

If ACC-001 reveals a genuine implementation defect, fix it only when the required behavior is already architect-authorized and the fix stays inside ACC-001 acceptance closure. Do not invent new product behavior.

## ACC-001 evidence

Write/update:

`05_EVIDENCE/ACC-001_EVIDENCE.md`

Include at minimum:

- exact Mercato and Scanner phase checkpoint SHAs;
- machine-readable 109/109 coverage result with zero requirement orphans;
- exact automated suites/commands/results actually used;
- migration/constraint verification;
- decisive INT/CON test references, including genuine PostgreSQL evidence where required;
- shared Inbound regressions if shared code was touched;
- exact changed files and why;
- clean type/build results relevant to changed surfaces;
- explicit statement that ACC-002/003 were not started before the ACC-001 internal gate was green.

## ACC-001 internal gate

Do **not** proceed to ACC-002 until all of these are true:

- 109/109 automated requirement coverage, zero orphans;
- all required automated suites green on the recorded checkpoint;
- migration/constraint checks green;
- required shared regressions green;
- evidence pushed;
- product branch checkpoint(s) pushed.

If the same material blocker survives two substantive attempts, STOP the whole batch. Do not skip to ACC-002.

# PHASE 2 — ACC-002

After ACC-001 internal gate is green, perform the second mandatory deep reset and require `RESET_OK`.

## Objective

Prove Standard Fulfillment and P1 exception/recovery paths through the **normal rendered Mercato/Scanner UI as real roles**.

Required scope is Task Catalog `ACC-002` plus Evidence Standard **journeys 1–14**, including partial/multi-shipment exception paths specified by the catalog.

## Playwright evidence rules

For every accepted journey:

- fixture/API/DB may prepare deterministic state only;
- the decisive user action must be click/type/scan through the normal app surface;
- use the intended actor/role and explicit warehouse context;
- record stable order/task/Shipment/TU identifiers;
- assert the visible expected outcome;
- verify authoritative persisted/server state after the UI action;
- record the next reachable human action;
- never label automation `HUMAN VERIFIED`.

Fix stale selectors, auth/session setup, deterministic fixtures and ordinary Testing runtime issues autonomously. If a genuine architect-authorized UI/backend defect blocks a required journey, fix it within ACC-002 rather than weakening the journey.

## ACC-002 evidence

Write/update:

`05_EVIDENCE/ACC-002_EVIDENCE.md`

Include:

- exact cumulative Mercato/Scanner checkpoint SHAs after ACC-002;
- required journey list with PASS/FAIL and mapped architect TC/requirements;
- actor, warehouse and stable identifiers;
- rendered visible assertion + persisted assertion for each journey;
- exact Playwright commands/results and stable artifact references where useful;
- any product/test changes made to close real ACC-002 gaps;
- explicit `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

## ACC-002 internal gate

Do **not** proceed to ACC-003 until every required ACC-002 journey is green on the recorded checkpoint and evidence is pushed.

If a true two-strikes blocker remains, STOP the whole batch. Do not skip to ACC-003.

# PHASE 3 — ACC-003

After ACC-002 internal gate is green, perform the third mandatory deep reset and require `RESET_OK`.

## Objective

Prove P2/P3/P4 through normal rendered UI, including Crossdock, Reservation Release and Physical Putback.

Required scope is Task Catalog `ACC-003` plus Evidence Standard **journeys 15–20**, including:

- Crossdock 1:1;
- Crossdock n:n sorting;
- shortage/damage/empty source TU;
- Goods Receipt pending/rejected/accepted re-evaluation;
- Reservation Release policy paths;
- cancellation race/recovery;
- Physical Putback invalid-location loop and successful inventory recovery.

## Architecture boundaries to preserve

- Inbound stays CLOSED / REFERENCE.
- Outbound consumes/produces the already accepted Crossdock contracts; do not move GR retry/business ownership into Outbound.
- `INT-01..03/06` and P3/P4 concurrency/idempotency semantics remain unchanged unless a genuine acceptance defect requires an architect-authorized correctness fix.
- Controlled source-TU/stock fixtures are allowed; no destructive Inbound data changes.
- deterministic stubs/test endpoints may prepare integration state, but cannot replace the decisive UI action.

## ACC-003 evidence

Write/update:

`05_EVIDENCE/ACC-003_EVIDENCE.md`

Include:

- exact cumulative Mercato/Scanner checkpoint SHAs after ACC-003;
- every required journey with mapped architect TC/requirements;
- actor, warehouse and stable identifiers;
- visible UI outcome + authoritative persisted quantities/states;
- exact Playwright commands/results and stable artifact references where useful;
- integration boundary/result correlation where relevant;
- any product/test changes made to close real ACC-003 gaps;
- explicit `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

# Final batch completion / STOP

Return batch `COMPLETE` only when:

- ACC-001 internal gate is fully green and `ACC-001_EVIDENCE.md` is pushed;
- ACC-002 internal gate is fully green and `ACC-002_EVIDENCE.md` is pushed;
- ACC-003 internal gate is fully green and `ACC-003_EVIDENCE.md` is pushed;
- exact final cumulative Mercato/Scanner branch HEADs are pushed;
- no unrelated scope drift exists;
- no Demo/Prod access occurred;
- no secrets were logged;
- `ACC-004` has **not** started.

Then STOP for Supervisor verification of all three item checkpoints.

Do not claim Supervisor FINAL PASS, Owner Acceptance or `HUMAN VERIFIED` for ACC-001/002/003. Formal progress remains 33/37 until independent Supervisor verification and explicit Owner acceptance are completed.
