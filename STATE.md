# WMS Outbound — STATE

**As of:** 2026-09-07  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** acceptance batch prepared  
**Formal implementation progress:** **33/37 items FINAL PASS / Owner Accepted**

## Architect baseline

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19
- requirements: **109 IDs = 98 FR + 6 INT + 5 CON**

Inbound remains **CLOSED / REFERENCE**. `PickWave` is out of scope v1. No separate Process 5 exists.

## Latest catalog checkpoint

`X-002 — Integration correlation, observability and operational recovery` — item **33/37** — **Supervisor FINAL PASS / Owner Accepted**.

- Mercato X-002: `outbound/x-002` @ `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`.
- Scanner accepted/frozen base: `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.

## Accepted non-catalog maintenance baseline

`Post-X-002 raw PostgreSQL SSL maintenance gate` — **Supervisor FINAL PASS / Owner Accepted** on 2026-09-07.

Accepted Mercato maintenance head:

`outbound/post-x002-raw-pg-ssl` @ `cfdbc608b22fc1dd44646c309335d2071aafd32c`.

Durable evidence:

`05_EVIDENCE/POST_X002_RAW_PG_SSL_MAINTENANCE_EVIDENCE.md` at WMS_Outbound `31cc794d40c9f95ea8c81b7b32db5eee7ce515cf`.

Accepted maintenance outcome:

- one canonical test-only raw `pg.Client` observer helper;
- all equivalent Outbound raw-client observer call sites migrated/justified;
- P1-009 stale idempotent-replay response defect corrected with authoritative refreshed reads;
- maintenance affected PostgreSQL suites **263/263 PASS**;
- Mercato typecheck clean;
- Scanner unchanged at `a2759a29347285dd1dcd14bf51633431fbf2a302`.

This gate is non-catalog and does not change the 37-item count.

## Current authorized execution scope — ACC-001..ACC-003 batch

The Owner explicitly authorized a **one-time batch exception** for the next three standard Task Catalog items in one fresh executor session:

`ACC-001 -> ACC-002 -> ACC-003 -> STOP`

Detailed guide:

`06_AGENT_GUIDES/ACC-001_003_BATCH_EXECUTION.md`

**Status:** grounded / prepared / Owner-authorized; **not launched yet**.

The three items remain separate acceptance units. Each requires:

- its own canonical deep Testing reset (`RESET_OK`) before the phase;
- its own exact product checkpoint SHA(s);
- its own evidence file;
- a fully green internal gate before moving to the next phase.

No Supervisor round-trip is required between the three phases for this one authorized batch. A true blocker stops the whole batch; later phases must not be skipped into.

Accepted product bases for the batch:

- Mercato `cfdbc608b22fc1dd44646c309335d2071aafd32c`;
- Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302`.

Cumulative batch branch convention:

- Mercato `outbound/acc-001-003-batch`;
- Scanner `outbound/acc-001-003-batch` only if Scanner changes are required.

Expected evidence:

- `05_EVIDENCE/ACC-001_EVIDENCE.md`
- `05_EVIDENCE/ACC-002_EVIDENCE.md`
- `05_EVIDENCE/ACC-003_EVIDENCE.md`

`ACC-004` is explicitly excluded from this batch.

## Batch acceptance boundaries

- `ACC-001`: 109/109 automated requirement coverage, zero orphans, all required automated suites/migrations/shared regressions green.
- `ACC-002`: normal rendered UI Playwright Standard Fulfillment + P1 exception journeys; automation is `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.
- `ACC-003`: normal rendered UI Playwright Crossdock/P3/P4 journeys with authoritative persisted reconciliation; preserve accepted Inbound/GR ownership and INT/CON boundaries.

Executor `COMPLETE` != Supervisor FINAL PASS != Owner Acceptance. Formal progress remains **33/37** until the batch is independently verified and the Owner explicitly accepts the catalog items.

## Durable operating rules

- Owner controls executor/venue/launcher/session mechanics.
- Current Git steering wins over stale chat/session/history.
- Canonical Testing only; no local PostgreSQL; Testing credentials frozen.
- Mandatory deep reset applies before each new Task Catalog item, including all three phases of this batch.
- Demo/Prod require separate Owner authorization.
- `architecture-context` is shared/Inbound reference only; Outbound Architect/Canon remains business authority.
- `ACC-004` starts only after ACC-002 and ACC-003 are independently verified/accepted and the Owner explicitly authorizes it.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`
