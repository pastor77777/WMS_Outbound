# P1-012 — Carrier Selection — execution guide

**Execution status:** authorized first implementation shot for catalog item `P1-012`  
**Catalog item:** `P1-012` — item **15/37**  
**Executor:** **NEW Antigravity session** from canonical `Devaxonic-WMS` checkout.  
**Scope type:** execution artifact only; **NOT steering**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md` from current `WMS_Outbound/main`.

Start from durable Git state only. Do not reconstruct mutable state from chat history.

## 0. Mandatory fresh-session startup

Start Antigravity only from the canonical checkout and wrapper:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
/home/ubuntu/.local/bin/agy-pl
```

Never use bare `agy`.

FIRST sync the local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work. Then read current Git state in this order:

1. `Devaxonic-WMS/AGENTS.md`;
2. `Devaxonic-WMS/.ai/STATE.md`;
3. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`;
4. `Devaxonic-WMS/.ai/TESTING.md`;
5. `Devaxonic-WMS/.ai/OPERATIONS.md`;
6. `WMS_Outbound/STATE.md`;
7. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-04.md`;
8. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`;
9. `WMS_Outbound/07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact `P1-012` section;
10. `WMS_Outbound/03_TRACEABILITY/requirements_index.csv`, `test_index.csv`, `rule_catalog.csv`, and `state_event_transitions.csv` for P1-012 Carrier Selection boundaries;
11. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — P1 R31–R34 and R51–R52;
12. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P1-16`, `FR-P1-17`, `FR-P1-26`, `FR-P5-10`;
13. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — `TC-001`, `TC-003`, `TC-007`, `TC-066`;
14. current `Devaxonic-mercato` accepted P1-011 implementation, migrations, entities, services and Supervisor UI before designing any new surface.

Authority order for Outbound behavior:

1. current Architect Source / faithful translations;
2. Canon + traceability + exact current task docs;
3. current Mercato / DB / runtime as implementation evidence;
4. Implementation Plan / Task Catalog as delivery decomposition.

Do not let generic architecture context redefine exact Outbound R-rules. Preserve accepted shared primitives only.

## 1. Frozen accepted starting point — verify before editing

Supervisor-observed durable state before this guide:

- accepted progress: **14/37 FINAL PASS**;
- accepted P1-011 Mercato head: `20887f2d74928cf69f447fdd6af20a612f38387c`;
- prior P1-010 Mercato head: `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`;
- frozen Scanner head: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- accepted P1-011 durable WMS evidence commit: `90cc30fc2db15c40d80ef69cb03ffb1e107b51dc`;
- Devaxonic-WMS steering head observed before this guide: `680f20c66a5aad9dc02e60bbf7843b725906100f`;
- WMS_Outbound steering/workflow head observed before this guide: `91eb6750450528f75ddc515eed6e073809c5b866`.

Before editing product code, independently refresh remote refs and verify 40-character SHAs and ancestry.

### Mercato branch

Create or use `outbound/p1-012` from the **exact accepted P1-011 head**:

`20887f2d74928cf69f447fdd6af20a612f38387c`

If `outbound/p1-012` already exists, verify it is a clean descendant of that exact accepted head and contains no unrelated work. If the branch exists with a wrong base or unrelated lineage, **STOP and report**; do not silently rebase, merge, or repair scope.

### Scanner

`Devaxonic-scanner` is **frozen** for P1-012 at:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

No Scanner-owned carrier rules or Scanner changes are authorized.

### Steering boundary

Do **not** update `STATE.md`, handovers, Task Catalog, Implementation Plan, traceability, Devaxonic-WMS `.ai` steering, Drive handover, or this workflow contract. Supervisor owns verification and owner-acceptance updates.

## 2. Exact authorized objective

Implement **only P1-012 Carrier Selection**:

- Region resolution for the Shipment delivery destination;
- Carrier / CarrierSetup applicability by Region, weight and volume;
- deterministic tie-break;
- automatic Carrier result when exactly/deterministically selectable;
- `CARRIER_PENDING` when no automatic result exists;
- Supervisor manual selection / override;
- the EXTERNAL TU missing-`maxVolume` manual-approval boundary required by Architect;
- only the minimum persistence/config/UI needed to make the above real and testable.

Dependencies P1-011 and P1-008 are accepted and must remain non-regressive.

Target components are limited to:

- current carrier master adapter / existing `ref_carrier` or provider mapping seam;
- Region and CarrierSetup persistence/configuration;
- Carrier Selection service at the Shipment readiness boundary;
- Mercato Supervisor operational UI for automatic result, no-result/manual path and override;
- minimum schema constraints needed for deterministic configuration.

## 3. Architect behavior — implement exactly

### 3.1 Start boundary

Carrier Selection may execute **only after the Shipment is `READY_FOR_DISPATCH`**.

Do not move P1-011 grouping/readiness semantics into this task and do not select Carrier for an open/not-ready Shipment.

Repeated evaluation after readiness must be idempotent and non-regressive.

### 3.2 Inputs

Resolve Carrier applicability using the Architect inputs:

1. delivery **Region** for the Shipment destination;
2. the **largest current weight** among the Shipment's relevant Packing TUs / current TU contents, using accepted source semantics;
3. the **largest `TUSetup.maxVolume`** among the relevant TU types/configurations for the Shipment.

Do not substitute averages, totals, UI-estimated values, client-only state, or unrelated Sales/provider heuristics for these exact inputs.

### 3.3 Candidate matching

A CarrierSetup candidate must be applicable to:

- the resolved Region;
- the governing weight input;
- the governing volume input.

Reuse current carrier master data through a narrow adapter where possible. Map existing `ref_carrier` / provider data without importing provider-label lifecycle into P1-012.

Configuration must fail closed when it cannot produce the Architect-required deterministic semantics. Add only the minimum genuine DB constraints/validation needed for this task.

### 3.4 Deterministic tie-break

When several CarrierSetup candidates match, choose in this exact order:

1. **narrowest matching volume range**;
2. then **narrowest matching weight range**;
3. then the unique **`Carrier.priority`** tie-break.

The same persisted inputs and configuration must always yield the same Carrier.

Do not invent additional hidden tie-break dimensions. If configuration still leaves an Architect-invalid ambiguity, fail closed with an explicit operational reason instead of choosing arbitrarily.

### 3.5 Result states and manual fallback

- one deterministic result -> persist/display `CARRIER_SELECTED` with the selected Carrier;
- no automatic matching result -> persist/display `CARRIER_PENDING` and expose the authorized manual path;
- Supervisor can manually choose a Carrier and reach `CARRIER_SELECTED`;
- Supervisor may change/override an automatic or manual selection **without requiring a reason**, per Architect.

All decisive state changes must be server-authoritative and persisted; UI-only selection state is not acceptance evidence.

### 3.6 EXTERNAL TU missing `maxVolume`

When an EXTERNAL TU has no usable `maxVolume`, do **not** invent a volume or force automatic selection.

Follow the Architect manual boundary:

- Dispatcher/manual Carrier choice;
- Supervisor approval;
- only then may downstream label work occur.

For P1-012, prove the selection/approval boundary only. If the current UI lacks an unrelated Dispatcher surface and adding it would expand into another task, expose/record the smallest existing operational manual handoff that preserves the rule, document the bounded gap, and **STOP rather than expanding scope**.

### 3.7 Existing ATP semantics

`FR-P1-26` also carries accepted ATP behavior. P1-012 does **not** reimplement ATP. Preserve it by regression evidence only.

## 4. Hard exclusions

P1-012 does **not** authorize:

- carrier label generation, printing, ZPL/PDF, provider-label lifecycle or label status;
- external Carrier API integration;
- manifest lifecycle;
- ERP posting/retry;
- physical dispatch / handover / shipment completion;
- later settlement or downstream tasks;
- Scanner changes;
- Prod or Demo changes;
- steering/state/handover/task-catalog/traceability edits;
- unrelated refactors or shared-platform redesign.

`FR-P1-17` and `TC-007` include downstream label language in their broader source scenario. For this catalog item, implement and prove **Carrier Selection only**; label generation belongs to the later authorized item.

## 5. Current-state audit before implementation

Before adding entities/migrations, inspect accepted Mercato P1-011 state and record in evidence what already exists:

- carrier/provider reference tables and adapters (`ref_carrier` or current equivalent);
- delivery-address/Region representation and tenant/org/warehouse boundaries;
- target Shipment entity/table and `READY_FOR_DISPATCH` transition from P1-011;
- Packing TU weight and TU/TUSetup volume sources;
- existing audit/state-transition primitives suitable for Carrier selection;
- existing Mercato Supervisor routes/pages/components that should host the result/manual path.

Reuse existing accepted target-WMS primitives when they satisfy part of P1-012. Do not create parallel carrier, Shipment, TU, audit or authorization models merely for this task.

## 6. Required backend behavior and transaction boundaries

Implement the smallest coherent server-side surface that can:

1. reject/defer selection before `Shipment READY_FOR_DISPATCH`;
2. resolve Region and governing Shipment weight/volume inputs from persisted authoritative data;
3. obtain applicable CarrierSetup candidates;
4. deterministically rank them by volume-range width, then weight-range width, then unique Carrier priority;
5. persist the automatic result or `CARRIER_PENDING` atomically with audit/state-transition semantics;
6. support Supervisor manual selection and override without reason;
7. support/represent EXTERNAL missing-`maxVolume` as manual/approval-required instead of fabricated automatic input;
8. replay safely without duplicate transition/audit side effects;
9. preserve tenant/organization/warehouse isolation and P1-011 Shipment readiness invariants.

Use genuine PostgreSQL constraints/transactions where configuration uniqueness, consistency or concurrent update safety requires them. Do not solve data invariants with process-local memory only.

## 7. Decisive genuine PostgreSQL evidence

Use the approved Testing database and canonical DB environment (`/etc/mercato-localhost.env`). No SQLite substitute. No local ad-hoc PostgreSQL. Do not expose secrets in logs/evidence.

At minimum prove with focused tests and independent reads:

1. **start gate:** not-ready Shipment cannot run/commit Carrier Selection;
2. **single match:** Region + governing weight + governing volume selects the sole applicable Carrier and persists `CARRIER_SELECTED`;
3. **volume tie-break:** multiple candidates -> narrowest matching volume range wins;
4. **weight tie-break:** equal volume specificity -> narrowest matching weight range wins;
5. **priority tie-break:** equal range specificity -> unique Carrier priority wins;
6. **determinism:** repeated evaluation with unchanged persisted inputs/config yields the same Carrier and no duplicate audit/transition effects;
7. **no match:** persists `CARRIER_PENDING` with a useful no-result/manual reason;
8. **manual fallback:** Supervisor manual choice transitions/persists to `CARRIER_SELECTED`;
9. **override:** Supervisor changes an automatic or previous manual result without a required reason; persisted result and audit reflect the new Carrier;
10. **EXTERNAL missing maxVolume:** automatic selection fails closed into the required manual/approval boundary; no fabricated volume;
11. **configuration validation:** Architect-required priority/determinism conflict is rejected or fails closed decisively;
12. **concurrent/replay safety:** overlapping/repeated selection/override attempts do not corrupt the persisted selection or duplicate irreversible state effects;
13. **P1-011 regression:** Shipment readiness/grouping/late-TU accepted behavior remains intact;
14. **ATP regression:** accepted ATP behavior referenced by `FR-P1-26` remains intact without being reimplemented.

Where a transaction writes more than one related row/state, include a real rollback test with an injected failure after a genuine write/flush and prove via fresh independent read that no partial selection/config state survives.

Report actual test names/counts and observed outputs. Do not pre-write expected timings/counts as if already observed.

## 8. Mercato Playwright acceptance

Use the normal rendered Mercato operational flow. Playwright evidence must identify the app runtime revision/commit actually serving the UI and independently prove persisted server state after the decisive action.

At minimum prove:

### Journey A — automatic selection

- open a real Shipment that is `READY_FOR_DISPATCH`;
- rendered Supervisor UI shows the Carrier Selection result produced from persisted configuration;
- persisted DB/server read confirms the same selected Carrier and `CARRIER_SELECTED` state;
- refresh/revisit shows the persisted result, not client-only state.

### Journey B — no match -> manual Supervisor selection

- use a real ready Shipment for which no CarrierSetup matches;
- rendered UI shows `CARRIER_PENDING` / no-result reason and manual action;
- Supervisor selects a Carrier through the real UI;
- server/DB confirms persisted `CARRIER_SELECTED` and selected Carrier;
- refresh/revisit preserves it.

### Journey C — Supervisor override

- start from an automatic or already-manual selection;
- Supervisor changes Carrier through the rendered UI with no mandatory reason field;
- server/DB confirms the replacement Carrier and auditable persisted change.

### Journey D — EXTERNAL/no-maxVolume boundary

- where the existing supported operational surface permits, demonstrate that missing `maxVolume` does not auto-select and routes to manual/approval handling;
- if completing a separate Dispatcher UX would require scope expansion, capture the backend/manual-boundary evidence and report that exact bounded UI blocker instead of implementing a later surface.

Evidence label is **PLAYWRIGHT VERIFIED**, not HUMAN VERIFIED.

## 9. Required durable evidence

Create/update candidate evidence under the existing convention:

`WMS_Outbound/05_EVIDENCE/P1-012_EVIDENCE.md`

Do not mark it FINAL PASS and do not update accepted progress. The Supervisor/Owner owns acceptance.

Evidence must include at least:

- exact 40-character implementation SHA and branch;
- exact accepted before-ref and verified lineage;
- WMS evidence commit SHA after push;
- concise changed-file/diff summary and scope-exclusion check;
- migration/schema/config details and rollback notes where applicable;
- exact genuine PostgreSQL test commands/results plus independent-read proof;
- Playwright routes/actions/outcomes plus app runtime revision and persisted server/DB proof;
- P1-011 and ATP regression results;
- frozen Scanner SHA confirmation;
- clean/understood Git status;
- explicit statement that label generation, external Carrier API, manifest/ERP/dispatch and steering were not changed;
- any bounded blocker, with no speculative scope expansion.

Screenshots, when useful, belong under `05_EVIDENCE/screenshots/` and must be tied to the real served revision and action.

## 10. Two-strikes rule

If the same failure class occurs twice, **STOP**. Record the exact failing command/action, observed error, current refs and the smallest grounded blocker. Do not start speculative rewrites, environment churn or scope expansion.

## 11. STOP boundary

When the authorized shot is complete:

1. push the task-scoped Mercato `outbound/p1-012` implementation commit(s);
2. push the WMS candidate evidence only under the existing evidence convention;
3. report exact refs/evidence briefly;
4. **STOP**.

Do **not** merge implementation branches, delete branches, update `STATE.md`, handovers, Task Catalog, traceability, Drive handover or any steering file. Do not claim FINAL PASS or owner acceptance.

Executor response to owner should be microscopic: `done` on successful push, otherwise at most five lines describing the exact blocker and refs.