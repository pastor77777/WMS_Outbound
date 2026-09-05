# P2-003 — RF crossdock sorting into outbound TUs

## Purpose

Implement and verify only item **22/37: P2-003** from the accepted P2-002 checkpoint.

This guide is written for a **new Codex session**. Git is the bootstrap source; no prior chat/session context is required.

Do not start P2-004 or later work.

## Frozen accepted inputs

- Mercato: `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- Scanner: `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`
- P2-002 evidence: `2074e2541b7c28cf3c6031cb48ad68901333625a`
- Canonical Testing PostgreSQL: Supabase `DevAxonic_Platform`, project `yzonugcenguvmojwiihb`
- Local PostgreSQL: **forbidden**

Before editing, sync all canonical repos and prove the Mercato/Scanner worktrees are clean and at the frozen accepted inputs (or exact authorized descendants created in this run).

## Authority

Read before implementation:

1. `STATE.md`
2. `08_HANDOVER/HANDOVER_CURRENT_2026-09-05.md`
3. `01_ARCHITECT_SOURCE/2026-08-31/proces_2_outbound_crossdock.md`
4. `01_ARCHITECT_SOURCE/2026-08-31/model_stanow_outbound.md`
5. `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md`
6. `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — P2-003 row only
7. `05_EVIDENCE/EVIDENCE_STANDARD.md`
8. `Devaxonic-WMS/.ai/TESTING.md` and `.ai/OPERATIONS.md`

Architect Source wins over implementation decomposition or historical test wording.

## Acceptance-mapping rule — avoid future-scope leakage

P2-003 Task Catalog references `TC-020`, `TC-021`, `TC-022`, `TC-023`, `TC-027`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-035`, `TC-037`, `TC-099`, `TC-101`.

These IDs are **traceability mappings, not permission to implement every later step of the full TC in P2-003**.

For this item, execute only the slice supported by P2-003 requirements/Architect refs:

- `FR-P2-02`, `FR-P2-04`, `FR-P2-05`, `FR-P2-10`, `FR-P2-12`, `FR-P2-19`, `FR-P2-21`, `FR-P2-22`;
- P2 R3–R4, R7–R10, R19–R20, R23–R24, R34, R39–R40.

Explicitly **do not** pull later scenario steps forward:

- `TC-020/021` Shipment/dispatch completion belongs to P2-006/ACC;
- `TC-023` shortage + Supervisor decision belongs to P2-004;
- residual/finalization exception slices of `TC-027` belong to P2-004;
- `TC-030` GR correlation/gate belongs to P2-005;
- `TC-035` cancellation/PutBack recovery belongs to P2-004/P4;
- later GR/Shipment/ERP/manifest assertions belong to P2-005/P2-006.

P2-003 may preserve already accepted P2-002 assignment behavior (`TC-099`) as regression, not reimplement it.

## P2-003 business scope

### 1. Start normal RF execution

From an already assigned `CrossDockPickTask`:

- first real execution scan transitions task `ASSIGNED → IN_PROGRESS`;
- its `OutboundOrderLine CREATED → PICKING`;
- the first active line in its `OutboundOrder` moves header `CREATED → PACKING_IN_PROGRESS`;
- general cancellation during active sorting remains blocked until the architect-defined later boundary; do not implement P2-004 cancellation recovery here.

### 2. Normal OK scan/placement

Packer uses the normal Scanner Crossdock module to scan/confirm:

- source task/source TU context;
- SKU;
- quantity;
- quality `OK`;
- target Outbound TU placement.

Server remains authoritative for task/source/SKU/quantity/target validation.

### 3. Lazy target Outbound TU

Before first successful placement there is no physical target Outbound TU.

At first successful `OK` placement, if no open target exists for the task, create/open it atomically.

Identity rule:

- full 1:1 topology + valid source GS1 SSCC → inherit source `TU_NUMBER` and SSCC;
- 1:1 without valid GS1 SSCC → new number using accepted P1-008 `TUSetup`/`Sequence` rules + new label identity;
- n:n topology → always new target identity using accepted P1-008 rules; do not inherit a source number merely because one source has valid GS1.

Do not change P1-008 global TU semantics unless a genuine P2-003 requirement forces a minimal compatible extension.

### 4. Multi-target continuation

When the active target becomes physically full before the task is complete:

- Packer seals/closes it through the real RF flow (`PACKING_SEALED` as defined by architect state model);
- the `CrossDockPickTask` remains `IN_PROGRESS`;
- the next successful placement under the same task opens/selects the next target TU according to the same identity rules;
- task identity is unchanged;
- no quantity is lost or double confirmed.

### 5. Task completion

Normal completion:

- fixes `confirmedQty` for the task;
- `confirmedQty <= plannedQty` always;
- completion is idempotent/exactly once;
- one task completing does not require other tasks of the same source TU to complete;
- do not implement P2-004 shortage/damaged/residual exception outcomes here.

## Hard exclusions

Do not implement in P2-003:

- shortage confirmation/recheck;
- `DAMAGED` quantity handling/QC;
- unexpected SKU exception handling beyond safe rejection/no side effect;
- empty source TU cancellation/LOST;
- `allowPartialShipment` shortage decisions;
- in-progress cancellation recovery or PutBackTask;
- residual source-TU finalization exception logic;
- GR_PENDING/GR_ACCEPTED/GR_REJECTED gate;
- Shipment, ERP, CarrierManifest or final settlement;
- P3/P4;
- Return Receipt;
- Demo/Prod;
- local PostgreSQL.

## Implementation constraints

- Prefer additive/bounded P2-003 entities/fields/services/routes.
- Reuse accepted P2-002 `CrossDockPickTask` and P1-008 TU identity services/contracts; do not create parallel truth.
- No Allocation.
- Every mutation must be organization/tenant/warehouse scoped.
- Placement/completion must have an idempotency/correlation identity sufficient to make duplicate RF submission safe.
- Do not move business truth into Scanner local state.
- Do not reinterpret Inbound process status ownership.

## Dedicated real-PostgreSQL suite

Create one dedicated P2-003 canonical PostgreSQL integration suite. Use `DATABASE_URL` from `/etc/mercato-localhost.env` in the same shell invocation. Run `--runInBand`.

Minimum substantive coverage (do not inflate count with trivial assertion splits):

1. no physical target TU exists before first placement;
2. first OK placement atomically starts task/line/order states;
3. first placement lazily creates/opens a target TU;
4. 1:1 + valid GS1 source inherits TU_NUMBER/SSCC;
5. 1:1 + invalid/non-GS1 source creates new TU identity;
6. n:n always creates new target identity and never inherits source identity;
7. confirmed placement uses exact decimal quantity and source/task/SKU correlation;
8. duplicate/replayed placement does not double-confirm quantity or duplicate TU content;
9. placement cannot exceed remaining `plannedQty`;
10. wrong task/source/SKU/tenant/warehouse is rejected with no committed side effect;
11. physical-full seal closes the current target while task remains `IN_PROGRESS`;
12. next placement after physical-full creates/uses a new target under the same task;
13. quantity across multiple target TUs reconciles exactly to task confirmed quantity;
14. normal task completion fixes `confirmedQty <= plannedQty` and is replay-safe;
15. completing one task does not require sibling task completion;
16. no Allocation is created by P2-003;
17. accepted P2-002 assignment/one-active-task behavior remains non-regressive;
18. P2-004 exception states/effects are not produced by the normal OK path.

If the implementation adds a distinct architect-required normal SLA sealing mechanism within P2-003, add a substantive test for `TC-037`'s P2-003 slice without implementing later Shipment behavior.

## Regression plan — relevance first

Mandatory on the final Mercato SHA:

- P2-003 dedicated suite;
- P2-002 dedicated PostgreSQL **22/22**;
- P2-001 **10/10** if P2-001 binding/source fields or eligibility semantics are touched;
- relevant P1-008 TU identity/numbering tests because P2-003 consumes those accepted rules.

Run FND-003 / Inbound shared TU / warehouse / task-lock regressions **only if the final diff actually modifies those shared primitives**. If not touched, evidence must state that they were inspected and unchanged rather than running broad historical suites by habit.

If a historical regression fails on untouched product code, classify current fixture/UI contract before any product change.

## Scanner implementation and real Playwright

P2-003 is human-facing and must ship with its real rendered acceptance journey in the same implementation shot, not as a later afterthought.

Add/extend only the Crossdock Scanner flow required for the normal P2-003 path.

Required rendered proof with **zero route mocks**:

### Journey A — 1:1 normal path

- real login;
- wait for real successful warehouse resolution/selection;
- enter rendered Crossdock module;
- real request-task returns the deterministic assigned task;
- start execution through rendered Scanner action;
- scan/enter source/SKU/quantity as normal UI;
- confirm quality `OK`;
- first placement creates target lazily;
- verify visible target identity follows 1:1 GS1 inheritance rule for the fixture;
- complete task through rendered UI;
- verify visible completed/confirmed result and persisted task/TU/quantity state.

### Journey B — n:n / physical-full continuation

- deterministic fixture with n:n topology and planned quantity greater than first target capacity;
- real assignment and warehouse context;
- first target identity is newly generated, not inherited;
- place part of quantity;
- physically-full seal via rendered RF control;
- same task remains active;
- next placement creates/uses a new target;
- finish the task;
- persisted target contents sum exactly to confirmed quantity and do not exceed planned quantity.

Do not use P2-004 shortage/damage/empty/cancellation actions in these journeys.

Use current SHA `testID`/accessibility contract. Do not assert historical copy unless it is part of the current product requirement.

## Runtime provenance — mandatory before Playwright

### Mercato

If any route/product source changes:

1. commit the intended Mercato implementation checkpoint;
2. prove final Mercato HEAD;
3. run repository-native full `yarn build` (which includes generation);
4. restart only canonical `mercato-localhost.service`;
5. prove service active, port 3009 listening and `/login` HTTP 200;
6. probe every new P2-003 API route unauthenticated/authenticated as appropriate so route presence is known before Scanner UI assertions;
7. a 404 is a runtime/route/build problem first, not a Playwright selector problem.

### Scanner

After final Scanner source changes:

1. commit the intended Scanner checkpoint;
2. restart canonical `scanner-testing.service` so its `ExecStartPre` performs fresh `npx expo export --platform web` from the exact worktree;
3. prove canonical Scanner port 8081 is serving and backend wiring targets canonical Mercato Testing;
4. prove the intended final Scanner revision/build marker or equivalent non-secret identity before Playwright;
5. do not let Playwright `reuseExistingServer` silently reuse a stale exported UI.

## Fixture rules

- explicit org, tenant, warehouse, user/role and warehouse assignment;
- explicit TUSetup/Sequence and target-capacity prerequisites required by each scenario;
- explicit topology/GS1 fixture data;
- no dependence on ambient “single warehouse”, default policy or existing queue content;
- isolate identifiers per test run;
- restore every shared configuration mutation in `finally`/cleanup;
- do not print credentials or full DATABASE_URL.

## Commit order

1. implement bounded Mercato/Scanner P2-003 work;
2. run dedicated backend suite and relevant regressions;
3. commit/push final product/test changes;
4. rebuild/restart exact final canonical runtimes;
5. run final rendered Playwright journeys on those exact SHAs;
6. only if all required gates are green, create `05_EVIDENCE/P2-003_EVIDENCE.md`;
7. commit/push evidence only;
8. STOP.

If a final product change occurs after a supposedly final regression/Playwright run, rerun the invalidated final-SHA evidence.

## Evidence requirements

`05_EVIDENCE/P2-003_EVIDENCE.md` must include:

- exact accepted bases and final Mercato/Scanner SHAs;
- compare/merge-base scope;
- canonical Testing provenance `yzonugcenguvmojwiihb` without secrets;
- dedicated P2-003 real-PG command/result/count;
- P2-002 **22/22** regression and any other actually-required relevant regression counts;
- explicit shared-regression touched/not-touched classification;
- exact runtime build/generate/service proof for Mercato and fresh Scanner export/service proof;
- Playwright Journey A and B result, zero route mocks, real warehouse resolution and real API request classifications;
- persisted quantity/TU/task reconciliation;
- explicit statement that full future slices of TC-020/021/023/027/030/035 were not pulled into P2-003;
- clean final worktrees;
- no `FINAL PASS`, no Supervisor acceptance, no Owner Accepted claim.

## Failure discipline

Diagnose by layer:

1. route/runtime status;
2. API response and persisted fixture state;
3. Scanner render/state;
4. selector/assertion.

Do not change selectors to hide a non-200 API or stale runtime.

Same material technical path gets at most two genuine attempts. After the second identical/material failure: STOP unless Owner authorizes a distinct narrow path.

## Final report

Report only:

- final Mercato SHA;
- final Scanner SHA;
- P2-003 dedicated result/count;
- P2-002 regression result;
- other relevant regression counts actually run;
- Playwright 1:1 result;
- Playwright n:n/physical-full result;
- WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty.

Then STOP.