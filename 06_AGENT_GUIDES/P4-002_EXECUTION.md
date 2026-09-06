# P4-002 — PutBackTask model, FIFO assignment and task lifecycle — Execution Guide

**Task Catalog item:** 30/37  
**Scope:** P4-002 only  
**Status:** Owner-authorized after 29/37 FINAL PASS / Owner Accepted  
**Executor:** Owner-selected Antigravity (AGY)  
**Pre-item Testing reset:** Owner reports canonical deep reset completed with `RESET_OK`; do not repeat it for this item or its continuations.

## Accepted starting point — freeze and preserve

Start from the exact accepted post-P3-003 baseline, not stale default-branch assumptions:

- Mercato accepted head: `f600782496e865603200d46d7e6041a54f90b9a4`
- Scanner accepted head: `135d86e1342bae7b21a8b676b1ef220a39a0f0b5`
- WMS accepted evidence/state head: `40d4fd10430ab2b58457f96573ed34af8f57d316`
- P4-001 accepted implementation is already in the Mercato lineage at `66e2e8620041d2db1d10d069e286936083667139`.

Expected implementation branch convention: `outbound/p4-002` in every product repo that changes, based on the exact accepted head above. If a valid P4-002 branch/workspace already exists, continue it; never restart from an older accepted base.

P3-003 is frozen. Do not reopen or redesign it unless P4-002 produces a genuine later regression against its accepted behavior.

## Supervisor execution contract

- Required skills before launch: `fetch_me_prompt` + `operational-mode`.
- `fetch_me_prompt` governs ticket/handoff content; `operational-mode` governs model/launch/supervision/retry/completion mechanics.
- Current Owner-authorized WMS steering defines the **entire P4-002 Task Catalog item as one executor-owned end-to-end unit**. This WMS full-item rule overrides generic micro-shot/shot defaults in shared skills where they conflict.
- Own the whole item through implementation, self-repair, real tests, regressions, build/runtime, rendered UI, evidence and pushes.
- Ordinary implementation/test/fixture/auth/TLS/selector/runtime/build failures remain executor-owned self-repair while a normal in-scope correction exists.
- `incomplete`, `unevidenced`, missing proof, a known fixable regression, execution-window stop or quota stop is not by itself a valid technical blocker.
- Provider/session interruption preserves the exact branch/HEAD/workspace checkpoint. Continue from the first unfinished point; never restart the item.

Failure boundary:

- Two-strikes applies only to the **same material unresolved technical path** after two genuinely different substantive evidence-based attempts.
- After the second materially similar unresolved failure, or when continuing requires an Owner-controlled boundary (scope expansion, destructive action, Demo/Prod, environment/venue/executor switch), STOP and report the exact action, exact failure, repo/branch/HEAD/runtime state, checks already green and exact remaining decision.
- Do not silently switch executor/model/venue, broaden scope or redesign.

Do not self-declare `FINAL PASS`, `Owner Accepted` or `HUMAN VERIFIED`.

## Authority and grounding

Business authority for P4-002, in precedence order:

1. `01_ARCHITECT_SOURCE/2026-08-31/proces_4_physical_putback.md` — P4 STEP 3–4, especially R5, R6 and R9; process prose wins on behavior.
2. `01_ARCHITECT_SOURCE/2026-08-31/model_stanow_outbound.md` — canonical `PutBackTask` lifecycle/state vocabulary.
3. `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P4-03`, `FR-P4-05`.
4. `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — `TC-050`, `TC-051`, `TC-052`, `TC-100`.
5. `03_TRACEABILITY/state_event_transitions.csv`, `requirements_index.csv`, `test_index.csv`.
6. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — P4-002 delivery slice only; it does not create business requirements.
7. Accepted implementation/evidence from P4-001/P3-003 as current-state evidence.

Use:

- `wms-outbound` for project routing and exact Outbound authority;
- `scanner-context` for generic RF/session/work-discovery/concurrency guidance only, especially `[SCN-SEC-050]`, `[SCN-SEC-051]`, `[SCN-SEC-073]`; Outbound authority wins;
- `architecture-context` only for shared/Inbound compatibility of task ownership/locks/warehouse/session primitives. Reuse technical foundations; do not import Inbound Putaway business states or behavior into P4.

Also read current `WMS_Outbound/06_AGENT_GUIDES/SCANNER_ROUTING.md` before Scanner changes.

## Accepted P4-001 handoff contract to consume, not replace

P4-001 already provides the durable physical-return source fact:

`wms_outbound_physical_return_handoffs`

For `pickedQty > 0`, the accepted cancellation path persists one unresolved handoff carrying at least the cancelled line correlation, warehouse, SKU, exact recovery quantity, TU correlation and source-location information. The existence of the unresolved handoff keeps the physically picked quantity excluded from new availability/allocation until later physical recovery.

P4-002 must **materialize exactly one PutBackTask from that accepted handoff** and make it discoverable/assignable through the RF returns module. Do not invent a second cancellation/recovery source of truth.

For handoffs that already exist when the P4-002 code is deployed, provide an idempotent reconciliation/materialization path so they are not stranded. For new P4 cancellations after P4-002 is installed, ensure the accepted handoff leads to exactly one PutBackTask without duplicate business effects.

Do not delete, resolve or otherwise make the physical-return handoff stop protecting stock merely because a PutBackTask was created or assigned. P4-003 owns physical completion and `Inventory PICKED -> AVAILABLE`.

## Objective

Implement the P4 `PutBackTask` model and its RF work-assignment boundary so an eligible operator can enter the **returns module**, receive the oldest submitted PutBackTask, take ownership atomically and begin the task.

P4-002 stops before destination-location validation/placement and physical Inventory recovery.

## Core invariants

### 1. One eligible physical return = exactly one PutBackTask

For every accepted unresolved P4 physical-return handoff with positive recovery quantity:

- exactly one `PutBackTask` exists;
- duplicate cancellation replay, materializer replay, retry or concurrent creation must not create another task;
- the task remains traceable to the original handoff and cancelled line/material/TU;
- task quantity equals the authoritative P4 recovery quantity, never recalculated from current ATP or a mutable UI value;
- zero-picked cancellation continues to create zero PutBackTask.

Prefer an explicit durable uniqueness relationship between handoff and PutBackTask rather than an application-only duplicate check.

### 2. Canonical PutBackTask lifecycle

The canonical full state machine is:

`CREATED -> ASSIGNED -> IN_PROGRESS -> LOCATION_VALIDATION -> COMPLETED`

with the P4-003 rejection loop:

`LOCATION_VALIDATION -> IN_PROGRESS`.

P4-002 owns the model and the executable initial lifecycle needed for assignment/start:

- creation in `CREATED`;
- atomic assignment `CREATED -> ASSIGNED`;
- operator start `ASSIGNED -> IN_PROGRESS`.

It may define the full canonical status vocabulary/schema now if that is the smallest compatible model, but **do not implement the P4-003 location submission/rejection/completion behavior** in this item.

Illegal/regressive transitions must be rejected server-side and audited through the accepted transition mechanism where applicable.

### 3. FIFO assignment means task-arrival order

P4 R9 is explicit:

- operator selects the returns module;
- if the operator has no active warehouse task of any type, WMS assigns the next PutBackTask in submission/arrival order;
- there is **no zone selection**;
- PutBackTask carries neither `priority` nor `slaDeadline`.

Implement deterministic FIFO on the authoritative server/DB side. Use `created_at` / submission time as the business ordering key with a stable deterministic tie-break if necessary. Customer priority or SLA must not reorder PutBackTask assignment.

### 4. Single active warehouse task guard is shared and authoritative

Do not create a PutBack-specific silo lock if the accepted shared task-ownership/record-lock primitive can be extended safely.

An operator already owning any active warehouse task must not receive a PutBackTask. At minimum preserve compatibility with accepted Outbound PickTask and CrossDockPickTask ownership, and preserve accepted Inbound/shared task behavior for every shared primitive touched.

The server/DB decides ownership atomically. Scanner local state such as `isLocked` is never authority.

Two operators racing for one task: exactly one wins. The loser receives a clear non-500 result and must not obtain duplicate ownership.

### 5. Warehouse/session boundary

Assignment is scoped to the operator's active authorized warehouse context.

- never assign a PutBackTask from another warehouse;
- warehouse identity remains explicit in the Scanner operational context;
- do not infer warehouse from scanned material/TU;
- preserve accepted auth/session/warehouse rules.

### 6. RF returns-module behavior

Scanner must expose the normal human flow required by P4 R9:

- operator enters/selects the returns module;
- no zone selector is presented for PutBack work;
- the system requests/receives the next eligible task;
- task screen shows human-operational identifiers/context needed to execute the task (task identity, relevant material/SKU/quantity, TU/source context where architect/current model provides it), not raw UUID instructions;
- an assigned task can be started, persisting `IN_PROGRESS`;
- duplicate/retry/start-after-timeout is idempotent and human-readable;
- if no task is available or operator already has active work, show a clear actionable state.

Do not implement destination-location scan/validation in this item.

### 7. Mercato visibility is observational only

Provide the Task Catalog-required Supervisor visibility for PutBackTask state/assignee/correlation using existing operational patterns where possible.

Do not give Mercato a second assignment authority and do not invent Supervisor reassignment/prioritization unless Architect authority explicitly requires it.

### 8. Physical inventory remains untouched

P4-002 must not:

- execute `Inventory PICKED -> AVAILABLE`;
- clear the unresolved physical-return protection merely because a task exists/was assigned/started;
- create placement/inventory movements;
- fabricate a validated destination.

P4-003 owns location validation and physical settlement.

### 9. No borrowed Inbound Putaway semantics

Shared locks, warehouse context, task-discovery patterns and UI primitives may be reused.

Do not reuse Inbound `PutawayTask` business statuses, routing rules, zones, quantity settlement or location semantics as P4 truth. `PutBackTask` remains a distinct Outbound task with P4 states and FIFO/no-zone assignment.

## Expected implementation surfaces

Inspect current accepted code first and make the smallest integrated change. Likely surfaces include, but are not limited to:

### Mercato/backend

- Outbound data entity + migration for `PutBackTask` if not already present as a real model;
- physical-return-handoff -> PutBackTask materialization/reconciliation service;
- task lifecycle/assignment service;
- extension of accepted shared active-task ownership/locking primitive where required;
- API/server actions for returns-module task acquisition/start;
- Mercato Supervisor read visibility;
- state transition/audit wiring;
- focused product/integration tests.

### Scanner

- returns/PutBack module entry;
- task acquisition via authoritative API;
- assigned-task screen and start action/state;
- no zone selection;
- clear no-work/already-active/owned-by-other/retry feedback;
- reuse accepted warehouse/session/scanning primitives without changing unrelated Scanner flows.

Do not perform broad navigation or Scanner architecture refactors.

## Migration / data rules

Any schema change must be additive/reversible and preserve accepted data.

If a new PutBackTask table/entity is required, it must support at least the architect/Task Catalog facts needed for P4-002 and later P4-003: source handoff/material/TU correlation, warehouse, quantity, optional proposed destination if already derivable, operator/owner, canonical status and timestamps.

Do not add `priority` or `slaDeadline` to PutBackTask.

Existing unresolved physical-return handoffs must be safely materializable into tasks exactly once. Do not fabricate task history for already resolved/invalid data.

## Required REAL POSTGRESQL acceptance

Canonical Testing PostgreSQL only per `Devaxonic-WMS/.ai/TESTING.md`; no local PostgreSQL.

Provide a dedicated P4-002 real PostgreSQL suite proving at least:

1. one positive unresolved P4-001 physical-return handoff materializes exactly one PutBackTask in `CREATED`;
2. materializer/reconciliation replay produces no duplicate task;
3. concurrent materialization of the same handoff produces one task through a real DB uniqueness/locking mechanism;
4. a new accepted P4 cancellation with `pickedQty > 0` yields one handoff and one task end-to-end, without changing P4-001 logical settlement semantics;
5. `pickedQty = 0` yields zero PutBackTask;
6. task carries exact handoff/line/SKU/quantity/TU/warehouse correlation needed for later execution;
7. FIFO: oldest `CREATED` task in the active warehouse is assigned first;
8. different CustomerOrder priority / `slaDeadline` values do not alter FIFO order;
9. no-zone semantics: assignment does not require or use zone selection;
10. operator with an already-active warehouse task receives no PutBackTask and no ownership mutation;
11. two independent operators/connections racing for one PutBackTask produce exactly one `ASSIGNED` winner, with genuine overlapping PostgreSQL operations/transactions and decisive DB-side lock/serialization evidence;
12. warehouse isolation: operator cannot receive another warehouse's task;
13. retry of the same acquisition operation is idempotent and does not consume a second task;
14. `ASSIGNED -> IN_PROGRESS` start is server-authoritative, idempotent on retry and rejects wrong owner/stale task;
15. illegal/regressive lifecycle transitions fail with zero mutation;
16. creating/assigning/starting a PutBackTask does not execute Inventory recovery and does not resolve the P4-001 physical-return stock-protection boundary;
17. real rollback proof: after task/ownership write+flush, deterministic failure before commit leaves task/owner/shared-lock state unchanged on a fresh independent read.

If current implementation architecture requires additional decisive cases, add them; do not weaken the minimum above.

## Mandatory regressions

Run the smallest relevant accepted regressions according to touched primitives.

At minimum when the corresponding surface is touched:

- P4-001 dedicated PostgreSQL cancellation/logical-settlement suite;
- P3-003 dedicated race suite because its post-confirm P4 handoff must remain accepted;
- P1-005/P1-006 PickTask assignment/picking tests for shared active-task ownership;
- CrossDockPickTask assignment tests for shared active-task ownership if that mechanism is touched;
- FND-003 shared task-lock/warehouse-context regressions;
- accepted Inbound/shared task/warehouse/record-lock regressions when shared primitives change;
- Scanner accepted picking/P3-003 regressions for changed shared Scanner navigation/session/task primitives.

Do not rerun unrelated broad suites merely for volume. Preserve already-green proof unless a product diff invalidates it.

## Build/runtime requirements

For every changed product repo run the native generate/typecheck/build/lint checks required by that repo and exact changed surface.

Rebuild/restart canonical Testing runtime from the exact candidate revisions as needed for real UI acceptance. Do not use Demo/Prod.

Record exact runtime revisions and environment identity in evidence. Testing credentials are frozen per canonical steering; use the designated harness and do not audit/rotate/refactor credential handling.

## Rendered UI acceptance — zero route mocks

Human-facing P4-002 behavior requires real rendered application proof.

### Scanner Journey A — FIFO returns-module assignment

1. Prepare at least two genuine unresolved P4 handoffs/tasks in one warehouse with distinct creation order and deliberately opposing CustomerOrder priority/SLA values.
2. Log into the canonical Testing Scanner with an eligible operator/warehouse context.
3. Enter the normal **returns / PutBack module** through rendered UI.
4. Verify there is no zone-selection step.
5. Acquire work through the real UI/API path.
6. Verify the oldest PutBackTask is assigned despite priority/SLA differences.
7. Verify task context is rendered using human-operational identifiers.
8. Start the task and verify persisted `ASSIGNED -> IN_PROGRESS`.

### Scanner Journey B — active-task protection

1. Give the operator a genuine active warehouse task through an accepted task path.
2. Enter the returns module.
3. Verify no PutBackTask is assigned and the UI explains the active-work block clearly.
4. Verify PostgreSQL ownership/task state is unchanged.

### Scanner Journey C — atomic race / ownership outcome

Use two independent rendered Scanner sessions/operators against the same oldest available PutBackTask when practical. The decisive race itself must still have real PostgreSQL concurrency proof. Rendered evidence must show one operator receives the task and the other receives next/no work/owned state without generic failure or duplicate assignment.

### Mercato Journey — Supervisor visibility

Through the normal Mercato UI, show the created PutBackTask and its relevant status/assignee/correlation as read-only operational visibility. Do not add reassignment/priority controls absent from Architect authority.

Automated browser evidence is `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

## Evidence

Write durable evidence to:

`05_EVIDENCE/P4-002_EVIDENCE.md`

Evidence must include:

- exact Mercato final branch/SHA and accepted base lineage;
- exact Scanner final branch/SHA and accepted base lineage;
- WMS evidence commit SHA;
- exact authority chain: P4 R5/R6/R9 -> `FR-P4-03`/`FR-P4-05` -> `TC-050`/`TC-051`/`TC-052`/`TC-100`;
- schema/migration details and uniqueness mechanism for one handoff -> one task;
- exact task state/assignment mechanism and shared active-task guard;
- dedicated PostgreSQL suite names/counts/results;
- genuine concurrency and rollback mechanism/provenance;
- exact regression suites/counts;
- build/runtime revisions;
- rendered Scanner/Mercato Playwright suite names/counts and decisive actions;
- explicit proof that PutBackTask has no priority/SLA and RF flow has no zone selection;
- explicit proof that Inventory remains physically unrecovered and unresolved handoff protection remains until P4-003;
- explicit exclusion of P4-003 destination validation/completion and any unrelated work.

Do not mark evidence `HUMAN VERIFIED`, `FINAL PASS` or `Owner Accepted`.

## Hard exclusions

Do not implement or modify beyond what is necessary for P4-002:

- P4-003 location validation loop, completion or `Inventory PICKED -> AVAILABLE`;
- P3-003 accepted race semantics;
- P4-001 cancellation rules except the minimal integration required to generate exactly one PutBackTask from its accepted handoff;
- Return Receipt;
- new Warehouse Transfer business logic;
- Inbound Putaway business semantics;
- PickWave;
- carrier/ERP behavior;
- Demo/Prod;
- credential cleanup/rotation/refactor;
- shared steering/skill files.

## Completion contract

P4-002 is COMPLETE only when the whole authorized item is done end-to-end:

1. implementation is pushed from exact accepted lineage in every changed product repo;
2. dedicated canonical Testing PostgreSQL acceptance is green, including real concurrency/rollback;
3. required accepted regressions are green;
4. native build/typecheck/generate/runtime checks are green;
5. normal rendered Scanner returns-module flow and Mercato visibility are `PLAYWRIGHT VERIFIED` with zero route mocks;
6. durable `05_EVIDENCE/P4-002_EVIDENCE.md` is pushed to `WMS_Outbound/main`;
7. final report contains only exact repo SHAs plus concise decisive DB/UI counts, or a true two-strikes / Owner-controlled blocker;
8. STOP. Do not start P4-003, X-001 or any later Task Catalog item.

Executor `COMPLETE` is not acceptance. Supervisor independently verifies remote Git/diff/tests/evidence; formal count advances only after explicit Owner Acceptance.