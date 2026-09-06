# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-07 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.  
Formal progress: **33/37 FINAL PASS / Owner Accepted**.

Latest catalog checkpoint:

- `X-002` — item 33/37 — **Supervisor FINAL PASS / Owner Accepted**.
- Mercato X-002 `4a89a95aad42c476ac206b53fe8ff67f3c8021a9`.
- Scanner accepted base `a2759a29347285dd1dcd14bf51633431fbf2a302`.

## Accepted maintenance baseline

The mandatory post-X-002 raw PostgreSQL SSL maintenance gate is now **Supervisor FINAL PASS / Owner Accepted**.

Accepted Mercato baseline for acceptance work:

`outbound/post-x002-raw-pg-ssl` @ `cfdbc608b22fc1dd44646c309335d2071aafd32c`.

Evidence:

`05_EVIDENCE/POST_X002_RAW_PG_SSL_MAINTENANCE_EVIDENCE.md` at WMS_Outbound `31cc794d40c9f95ea8c81b7b32db5eee7ce515cf`.

Verified final maintenance result: affected PostgreSQL suites **263/263 PASS**, typecheck clean, Scanner unchanged. The discovered P1-009 idempotent-replay stale-read defect was corrected by authoritative refreshed reads; no business rule/API/schema change.

## Active authorization — ACC-001..ACC-003 batch

Owner authorized one fresh executor session for:

`ACC-001 -> ACC-002 -> ACC-003 -> STOP`

Detailed guide:

`06_AGENT_GUIDES/ACC-001_003_BATCH_EXECUTION.md`

**Status:** prepared / Owner-authorized / not launched yet.

This is a one-time exception to the normal Supervisor round-trip between catalog items. It does **not** merge the three items or waive their gates.

Each phase must have:

- canonical deep Testing reset with `RESET_OK` before starting;
- separate checkpoint SHA(s);
- separate durable evidence;
- full internal green completion before the next phase begins.

Evidence files:

- `05_EVIDENCE/ACC-001_EVIDENCE.md`
- `05_EVIDENCE/ACC-002_EVIDENCE.md`
- `05_EVIDENCE/ACC-003_EVIDENCE.md`

Accepted batch bases:

- Mercato `cfdbc608b22fc1dd44646c309335d2071aafd32c`;
- Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302`.

Cumulative branch convention:

- Mercato `outbound/acc-001-003-batch`;
- Scanner same branch only if Scanner changes are required.

## Phase gates

### ACC-001

109/109 requirement-to-test automated coverage, zero orphans, all required automated suites/migration checks/shared regressions green.

### ACC-002

Playwright journeys for Standard Fulfillment + P1 exceptions through normal rendered Mercato/Scanner UI. Fixture APIs/DB may prepare data but cannot replace decisive user actions. Record visible + persisted outcomes. `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

### ACC-003

Playwright Crossdock + Reservation Release + Physical Putback through normal rendered UI, including 1:1/n:n, shortage/damage/empty TU, GR re-evaluation, cancellation recovery and invalid-location loop. Preserve accepted Inbound/GR ownership and INT/CON boundaries.

A true blocker in any phase stops the whole batch. Do not skip forward.

## Final stop boundary

After ACC-003 executor COMPLETE, STOP for independent Supervisor verification of all three phase checkpoints.

Do **not** start `ACC-004`. It remains the separate final Human Verified item and requires explicit Owner authorization later.

Formal progress remains **33/37** until Supervisor verification and explicit Owner acceptance.

## Durable rules

- Executor `COMPLETE` != Supervisor FINAL PASS != Owner Acceptance.
- Owner controls executor/venue/launcher/session mechanics.
- Current Git steering overrides stale chat/session/history.
- Canonical Testing only; no local PostgreSQL; Testing credentials frozen.
- Inbound remains CLOSED / REFERENCE.
- Demo/Prod require explicit Owner authorization.
