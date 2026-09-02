# P1-006 Remediation — Explicit Second Override After STOP

Owner explicitly authorizes ONE more narrow remediation shot after STOP.

Use the **same existing Antigravity session**. Do not start another ticket.

## FIRST: synchronize this repository locally

Before reading or executing the remediation:
1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main` without discarding unrelated local work.
3. Verify this local file is the current remote version:
   `06_AGENT_GUIDES/P1-006_REMEDIATION.md`
4. Read the LOCAL file completely.
5. Execute it in the SAME Antigravity session.

Do not reclone repositories and do not replace the current Antigravity session.

## Current remediation heads

- Mercato: `dbc8ddef2873d6babde73a1ecd854c5feb1dff0e`
- Scanner: `7d36f85ab5cf8020746d9a007fe1d8a5795980ae`
- WMS_Outbound evidence base includes remediation evidence commit `489d3a25ef4a5a2c04ce0893a00e54f76da71ece`

## Preserve — already accepted in this shot

Do not redesign/regress:
- real rendered Scanner RF picking and zero-route-mock Playwright;
- formal picked quantity persistence;
- task completion without auto-closing TU;
- real PostgreSQL concurrency lock proof;
- real application rollback;
- warehouse authorization;
- R55 rendered Zone A -> Zone B update;
- P1-009 Direct Pack removed from the P1-006 Scanner journey;
- canonical authenticated operator identity / no synthetic `'operator'` fallback.

Fix **ONLY the four remaining blockers below**.

---

## 1. Idempotency must be mandatory and retry-stable

### Current defect

`idempotencyKey` is optional at the HTTP/service boundary, so a client can bypass the invariant.
Scanner currently creates a fresh key inside each `handleConfirmPick` using time/randomness, so a retry after a committed request but lost response can generate a new key and double-apply a partial pick.

### Required backend contract

For human RF pick confirmation:
- `idempotencyKey` is REQUIRED, non-empty, server-validated.
- No human `confirm-line` request may execute without it.
- Service must fail closed if missing when called through the RF confirmation path.
- Existing durable `wms_outbound_pick_confirmations` unique key remains authoritative.
- Same key + same canonical payload -> replay prior applied confirmation, no second quantity/TU-content write.
- Same key + conflicting payload -> fail closed.

Canonical payload comparison must include every material field needed to distinguish a confirmation, including at minimum warehouse, task, task line, TU, scanned location, SKU, quantity and canonical operator identity.

### Required Scanner retry behavior

A single logical operator confirmation action must own one stable key.

Implement the smallest robust state model:
- generate K when a new pending pick action is created;
- retain K while the request is in flight or outcome is uncertain;
- retry of the SAME pending action reuses K;
- generate a new key only after the prior action has received a decisive success or decisive domain rejection and a genuinely new operator confirmation begins.

Do not generate a fresh key merely because the button handler ran again.

### Required proof

Real PostgreSQL tests:
1. missing key -> rejected; zero DB mutation;
2. partial pick 4/10 with K -> picked=4;
3. sequential exact replay K -> still 4, exactly one logical content/confirmation;
4. same K conflicting material payload -> rejected;
5. two overlapping independent real transactions/connections using the SAME K -> exactly one applied, fresh DB read remains 4;
6. persist/report relevant PIDs and DB-side unique/lock evidence where applicable.

Scanner/Playwright:
- prove one normal confirmation still works;
- add a focused test at API/client level proving a simulated retry reuses the same key rather than generating a new one.

---

## 2. R62 must execute both strategies, not only return `requiresNewTu`

### Current defect

Backend currently reports `strategy` / `requiresNewTu`, but the `SEPARATE_PER_TASK_OR_ZONE` test only checks the flag and then switches configuration back to SHARED before real continuation.
Scanner also effectively consumes only the task list and does not execute separate-TU behavior.

### Required behavior

#### `SHARED_SAME_ORDER_CONSECUTIVE_ZONES`
- completed task may continue into an eligible same-order next-zone task using the SAME open Picking TU;
- prove actual same TU id before/after continuation.

#### `SEPARATE_PER_TASK_OR_ZONE`
- same-order next-zone continuation may still be selected as the next task, but the previous TU MUST NOT be reused for the next task;
- previous TU is closed/transitioned according to the accepted P1-006/P1-008 boundary;
- Scanner requires/selects/creates a new Picking TU before the first pick in the continued task;
- first Zone B pick must be persisted into a DIFFERENT TU id;
- attempting to confirm Zone B into the old Zone A TU must fail closed.

The strategy must be server-authoritative. Do not rely only on a UI hint.

### Required proof

Real PostgreSQL integration tests must execute BOTH complete flows end-to-end:
- SHARED: Zone A task -> continuation -> Zone B pick into same TU;
- SEPARATE: Zone A task -> continuation -> old TU not eligible -> new TU bound -> Zone B pick into different TU.

Scanner must consume returned strategy/`requiresNewTu` and behave accordingly.

Extend Playwright or add one focused genuine Scanner Playwright case for the SEPARATE strategy if practical; at minimum the existing Playwright must continue proving SHARED, while genuine backend integration proves SEPARATE decisively.

---

## 3. R67 switch must be impossible until current TU is durably `PICK_FULL`

### Current defect

`declareTuFull` can persist `PICK_FULL`, but `switchPickingTuForTask` still permits switching an ordinary active TU. Current test switches later TUs without declaring them full first.

### Required invariant

`switchPickingTuForTask` must fail closed unless:
- PickTask is active and owned by the operator;
- current TU belongs to the task/order/warehouse as applicable;
- current TU status is exactly the Architect-eligible persisted full state (`PICK_FULL`) for the R67 capacity-switch path.

Do not allow arbitrary switch from `CREATED` or ordinary `IN_PICKING` for R67.

The Scanner must:
- not present/enable `Switch TU` as an R67 action until the active TU is `PICK_FULL`;
- allow `Declare Full` only while appropriate;
- after durable `PICK_FULL`, allow switch;
- new TU remains on the SAME active PickTask.

### Required proof

1. switch while TU is ordinary `IN_PICKING` -> rejected; task and TU unchanged;
2. durable `declareTuFull` -> fresh DB sees `PICK_FULL`;
3. switch after `PICK_FULL` -> succeeds;
4. new TU is different, active, tied to same PickTask;
5. continue picking and prove authoritative cumulative picked quantity;
6. every additional R67 switch in the test must first put the current TU into `PICK_FULL` — no shortcut calls.

Update Scanner UI tests as needed to prove the button lifecycle.

---

## 4. Add a real additive migration for the P1-006 schema

### Current defect

Mercato entity model adds:
- `wms_outbound_pick_confirmations`;
- `wms_outbound_warehouse_queue_configs.picking_tu_strategy`;

but the remediation commit contains no deployable migration for them.

### Required migration

Add a new forward-only timestamped P1-006 migration after the existing P1-008 migration lineage.

UP must add only P1-006 schema:
- nullable/default-compatible `picking_tu_strategy` column with accepted default semantics;
- `wms_outbound_pick_confirmations` table with required scope fields;
- unique constraint/index enforcing durable idempotency key scope;
- required supporting index(es).

DOWN must safely remove only the P1-006 additions.

Do NOT rewrite old accepted migrations.
Do NOT redefine shared Inbound tables.

### Required migration proof

On a real PostgreSQL schema at the accepted prior migration state:
1. apply P1-006 migration;
2. verify new column/table/constraints/indexes;
3. run focused P1-006 tests;
4. rollback migration and prove only P1-006 additions disappear / prior accepted schema remains;
5. reapply migration and rerun required focused tests.

Persist exact migration filename and command/output evidence.

---

## Regression / durable evidence

After fixes rerun at minimum:
- focused P1-006 PostgreSQL suite;
- genuine P1-006 Scanner Playwright;
- P1-005 assignment regression;
- P1-008 TU regression;
- targeted accepted Inbound shared TU/warehouse regression.

Update and PUSH durable P1-006 evidence with:
- exact Mercato + Scanner SHAs and lineage;
- migration file and apply/down/reapply proof;
- mandatory idempotency + stable retry proof;
- actual SHARED and SEPARATE R62 execution proof;
- R67 rejected-before-full and succeeds-after-full proof;
- test commands/pass counts;
- Playwright result and screenshots/trace where useful;
- regressions;
- remaining gaps.

## Boundary / STOP

- No P1-007.
- No P1-009.
- No unrelated redesign.
- Do NOT self-declare Human Verified or FINAL PASS.
- STOP after implementation + push + durable evidence.
