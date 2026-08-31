# DevAxonic WMS Outbound

This repository is the canonical knowledge, traceability and implementation-planning repository for **WMS Outbound v1**.

Current status: **COMPLETE OUTBOUND V1 IMPLEMENTATION PLAN PREPARED — PRODUCT CODE NOT STARTED**.

## Start here

1. `AGENTS.md` — mandatory authority and agent routing.
2. `STATE.md` — current project state and next allowed action.
3. `01_ARCHITECT_SOURCE/MANIFEST.md` — immutable architect snapshot and hashes.
4. `02_CANON/OUTBOUND_GOLDEN_RECORD.md` — compact canonical interpretation of the active architect source.
5. `03_TRACEABILITY/README.md` — rule → requirement → test indexes.
6. `04_CURRENT_STATE/CURRENT_STATE.md` and `GAP_ANALYSIS.md` — verified implementation/database baseline.
7. `07_IMPLEMENTATION_PLAN/IMPLEMENTATION_PLAN.md` and `TASK_CATALOG.md` — complete Outbound v1 implementation plan.
8. `05_EVIDENCE/EVIDENCE_STANDARD.md` — Playwright/Human Verified evidence contract.

## Scope boundary

This repository does **not** make Inbound business rules authoritative for Outbound. Inbound repositories are organizational/reference patterns only. The active Outbound architect snapshot is the business authority.

`PickWave` is outside Outbound v1, including speculative technical extension points. Do not add it to the plan without a new architect decision.

## Code repositories

- Mercato/backend/admin UI: `pastor77777/Devaxonic-mercato`
- Scanner/RF: `pastor77777/Devaxonic-scanner`
- Scanner generic technical context: `pastor77777/scanner-context`
- Main steering/router: `pastor77777/Devaxonic-WMS`

Implementation code is expected to be delivered in the owning application repositories; this repository owns the immutable source snapshot, canon, traceability, current-state evidence and implementation plan.
