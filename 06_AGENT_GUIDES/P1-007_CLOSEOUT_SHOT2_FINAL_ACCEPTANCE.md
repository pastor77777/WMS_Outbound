# P1-007 Final Closeout — Shot 2: Final Acceptance & Evidence Gate

Owner explicitly authorized closing P1-007 with all remaining problems resolved, without exceptions.

Shot 1C migration proof has now been independently accepted by the supervisor. This is the **FINAL EVIDENCE-ONLY CLOSEOUT SHOT** for P1-007.

Use the SAME existing Antigravity session. Do not restart or replace it.

## Hard boundary

This shot is verification/evidence only.

- Do **not** change P1-007 product services, API behavior, migrations, Mercato UI, Scanner UI, business rules, shared Inventory/TU/warehouse semantics, or accepted prior-ticket implementation.
- Do **not** implement P1-009/P1-010, P4 PutBackTask lifecycle, packing/QC, Shipment, Carrier, labels, ERP, manifest, or unrelated cleanup.
- If any required gate fails, STOP and report the exact failing command, exact failure, current repo/branch/HEAD/runtime revision, and evidence gap. Do not fix the product in this shot.

## Required authority refresh

Before execution, refresh locally:

- installed `wms-outbound` skill;
- installed `architecture-context` skill, reference-only for accepted Inbound/shared foundations;
- `fetch_me_prompt` + `operational-mode`;
- current `.ai/TESTING.md`, `.ai/OPERATIONS.md`, `REAL_EVIDENCE_CONTRACT.md` if present;
- `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — P1-007;
- `01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — R42–R48, R69, R71;
- `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — FR-P1-21..24, FR-P5-01..06;
- `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — TC-060/061/062/121 and mapped TC-001/008 context;
- `05_EVIDENCE/EVIDENCE_STANDARD.md`.

Current authority wins if stricter.

## Current independently verified heads

- Mercato `outbound/p1-007`: `134db31381b4db726cd550abe6ecd4079ac21d8c`
- Scanner `main`: `b23325aae1c4f83b79d01b3650dbead3486a1041`
- Current P1-007 evidence commit before this shot: `8a19b8a1bcdfb9fb7b6bf9a3e9623c3832ce8111`

Mercato Shot 1C is a single clean descendant of prior head `1979942c993688be84d8765c0baae0eb594f9aa9` and changes only the P1-007 PostgreSQL migration-proof test harness.

Scanner must remain unchanged.

## Already accepted in Shot 1C — do not redesign

Migration test `6A` now uses genuine MikroORM `Migrator` with real migration history in an isolated PostgreSQL schema:

- preceding migrations 1..10 applied by `migrator.up({ to: ... })`;
- remediation UP by Migrator;
- remediation DOWN by Migrator;
- remediation re-UP by Migrator;
- fresh PostgreSQL constraint checks at each state;
- incompatible zero-quantity DOWN fails before mutation;
- zero rows remain intact;
- remediation remains recorded as applied after blocked DOWN;
- no direct `new Migration...`, no `getQueries()` replay, no hand-written shadow-table DDL.

Do not reopen this unless the final rerun itself fails.

## Exact final acceptance gate

Run and persist the **exact command and exact pass/fail counts** for every item below on the current heads.

### 1. P1-007 real PostgreSQL suite

Run current P1-007 genuine PostgreSQL integration suite, including:

- TC-060 SHORT_ALLOCATED both partial/non-partial outcomes;
- TC-061 automatic retry, effective limit, different/unblocked location, per-location and warehouse-wide hard reservation bounds including pre-PickTask reservations;
- TC-062 WAIT / exact-picked correction / zero cancellation / persistent ALLOW_PARTIAL;
- SAME-key real PostgreSQL concurrency with distinct PIDs and DB-side lock evidence;
- Supervisor idempotent replay/conflict behavior;
- real rollback;
- R48 backend seam;
- genuine Migrator 6A proof.

Expected test inventory is currently 20 tests. Record actual result, not expectation.

### 2. Mercato real UI / Playwright

Run the real rendered P1-007 Supervisor journey with no decisive route mocks and record actual runtime/revision provenance before claiming it.

All six distinct outcomes must pass through the real UI:

1. SHORT_ALLOCATED -> ALLOW_PARTIAL with mandatory reason;
2. SHORT_ALLOCATED -> CANCEL_OUTBOUND_ORDER;
3. SHORT_PICKED -> WAIT;
4. SHORT_PICKED -> CANCEL_OR_CORRECT to authoritative picked quantity;
5. SHORT_PICKED -> CANCEL_OR_CORRECT to 0 with stored COL/OOL zero quantity proof;
6. SHORT_PICKED -> persistent ALLOW_PARTIAL with reason.

Automation label: **PLAYWRIGHT VERIFIED**, never HUMAN VERIFIED.

### 3. Scanner real UI / Playwright

Run real Scanner P1-007 journey on current Scanner head:

- assigned task;
- Picking TU;
- source + SKU + actual short quantity;
- explicit SHORT_PICKED action;
- original task remains SHORT_PICKED;
- source block persisted;
- replacement task comes from different eligible location when allowed;
- replacement task completes missing quantity;
- resulting cumulative picked quantity/state is correct.

Record actual runtime/revision provenance before claiming it.

### 4. Required targeted regressions

Run and persist exact commands/results for all required accepted dependencies/invariants:

- **P1-004** Allocation / hard-reservation regression — current known suite `p1-004-postgres.integration.test.ts`;
- **P1-005** PickTask creation/assignment/single-active regression — current known suite `p1-005-postgres.integration.test.ts`;
- **P1-006 backend** real RF picking + SAME-key/retry idempotency — current known suite `p1-006-postgres.integration.test.ts`;
- **P1-006 Scanner** real scanner picking + retry-key stability — current known specs `p1-006-real-scanner-picking.spec.ts` and `p1-006-retry-key.spec.ts`;
- **P1-001** CustomerOrder lifecycle/aggregation/policy regression — current accepted focused suites;
- **P1-003** planning + requiredQty regression — current accepted focused suites;
- **P1-008** TU regression because P1-007 uses shared TU/picking continuity and prior evidence already included this gate;
- **Inbound/shared compatibility** targeted Inventory/location/warehouse/TU compatibility for the primitives touched by P1-007, using the currently accepted focused shared-compatibility/ATP suites;
- **full `wms_outbound` umbrella suite** as additional gate.

Do not substitute a single broad suite for the named targeted gates. Do not omit targeted gates because the full suite is green.

### 5. Revision and lineage reconciliation

Persist exact 40-char SHAs and independently check:

- Mercato branch head exactly matches tested implementation;
- Scanner `main` head exactly matches tested Scanner;
- lineage from the accepted P1-006 bases remains intact;
- no unrelated product changes entered during closeout;
- final evidence WMS commit points to the exact tested heads.

If a real UI runtime is not actually serving the tested revision, do not claim UI acceptance. Record the mismatch and STOP rather than inventing provenance or silently rebuilding/restarting shared infrastructure without authorization.

## Final durable evidence repair

Update `05_EVIDENCE/P1-007_EVIDENCE.md` so it is self-contained and does not lose previously required evidence.

It must contain, at minimum:

1. exact current Mercato/Scanner/WMS SHAs and lineage;
2. concise Architect mapping for R42–R48/R69/R71 and FR-P1-21..24 / FR-P5-01..06;
3. TC-060/061/062/121 proof summary tied to real DB/UI evidence;
4. Shot 1C genuine MikroORM Migrator UP/DOWN/re-UP + incompatible DOWN proof;
5. strict PG concurrency proof with real distinct PIDs / blocking evidence;
6. rollback proof;
7. Supervisor canonical idempotency/replay/conflict proof;
8. Mercato 6-outcome Playwright result;
9. Scanner short-pick/replacement continuation Playwright result;
10. **complete targeted regression matrix** for P1-004/P1-005/P1-006/P1-001/P1-003/P1-008/shared Inbound compatibility/full Outbound, with exact commands and counts;
11. evidence labels must remain `PLAYWRIGHT VERIFIED` / real PostgreSQL classes as appropriate, never `HUMAN VERIFIED` unless the owner later performs/accepts the human walkthrough;
12. any remaining gap must be explicit. Do not write `FINAL PASS` yourself.

The current evidence commit accidentally removed the targeted regression matrix during Shot 1C. Restore it from actual fresh results in this shot; do not merely copy stale prose without executing the required gates.

## Completion

If and only if all gates are green:

- push only the evidence update to `WMS_Outbound`;
- report exact final Mercato SHA, Scanner SHA, WMS evidence SHA and all pass counts;
- STOP.

Do **not** update `STATE.md` to accepted progress and do not self-declare FINAL PASS/HUMAN VERIFIED. The supervisor will independently verify this final evidence and then perform the acceptance/state update if warranted.
