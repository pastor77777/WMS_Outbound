# X-002 — Integration correlation, observability and operational recovery

**Status:** current execution guide  
**Effective:** 2026-09-06  
**Item:** Task Catalog 33/37  
**Base:** Mercato `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` (X-001 FINAL PASS / Owner Accepted) -> new branch `outbound/x-002`  
**Frozen by default:** Scanner `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`

## Objective

Audit and close the six Outbound integration boundaries `INT-01..06` so every accepted message/effect is durably correlatable, safely diagnosable and replay-safe without moving business ownership across bounded contexts.

This is an **audit-and-harden** item, not permission to build a new generic integration bus or redesign already accepted P1/P2/P3/P4 flows. Most boundaries already have substantial accepted implementation. Reuse current domain facts, idempotency, transition audit and orchestration primitives; add only what is genuinely missing for X-002 acceptance.

## Authority

Read current project steering, Task Catalog X-002, `wymagania_outbound_EN.md` `INT-01..06`, and exact current process sources before changing code.

Governing boundaries:

- `INT-01` — P2 STEP 1: Inbound exposes an `ELEMENTARY` TU already in `IN_CROSS_DOCK`; Outbound receives TU identity, SKU/item, **ASN-declared** source quantity and receipt correlation. Outbound must not reinterpret that as physically verified Inbound quantity.
- `INT-02` — P2 STEP 3: when source-TU crossdock settlement closes, Outbound supplies `confirmedQty` + `damagedQty` with source correlation. Residual quantity is **not** an extra outbound contract field; Inbound derives it.
- `INT-03` — P2 STEP 4: GR-result correlation is `sourceInboundTU + GR_SETTLEMENT_SOURCE=CROSSDOCK`; GR retry/message transport remains Inbound responsibility. Outbound consumes the result for its Shipment GR gate only.
- `INT-04` — current P1 **STEP 11A**: eligible Shipment POST to ERP is uniquely identifiable and enters `POSTING_PENDING` before ERP response.
- `INT-05` — current P1 **STEP 11A**: explicit ERP rejection -> `POSTING_ERROR` with structured safe error data; retry is a separate Warehouse Supervisor decision; ERP acceptance -> `POSTED`. Timeout/no response is a technical incident and does not fabricate a business rejection/state transition.
- `INT-06` — P3 STEP 1 + P4 STEP 1: cancellation/correction from ordering system correlates to the correct order/line and routes by authoritative formally confirmed picked quantity, including the accepted P3/P4 race handling.

### Traceability correction

Derived `INT-04` / `INT-05` references still say P1 STEP/KROK 13. In current P1 v1.20, ERP posting is STEP **11A**; STEP 13 is physical dispatch/final settlement. Treat the old step number as a traceability-reference defect, not a product blocker. Do not edit immutable Architect snapshots or implement STEP 13 dispatch semantics as the ERP posting contract.

`architecture-context` / WMS-Records is shared/Inbound technical reference only. Inbound remains CLOSED / REFERENCE and its accepted GR delivery/retry ownership must not move into Outbound.

## Current accepted implementation to preserve

Audit these exact final-base paths before adding anything:

### INT-01

`cross-dock-eligibility-service.ts` already:

- accepts `sourceInboundTuId`, `itemId`, `receiptCorrelation`, `idempotencyKey`;
- locks the real shared source TU and requires `ELEMENTARY` + `IN_CROSS_DOCK`;
- derives source quantity from `WmsTuExpectedContent` ASN declaration;
- persists `WmsOutboundCrossDockBinding` with `receiptCorrelation`, `asnDeclaredQty`, source TU/item and idempotency identity.

Preserve this contract. X-002 may add audit/correlation visibility around it but must not change the eligibility formula or Inbound qualification semantics.

### INT-02

`cross-dock-execution-service.ts` already creates a durable `WmsOutboundCrossDockFinalization` once source participation closes, with source TU, declared/confirmed/damaged/residual quantities and idempotency. Quantity conservation is enforced.

The integration contract is `confirmedQty + damagedQty + correlation`; `residualQty` may remain an internal reconciliation fact but must not become an invented outbound-to-Inbound contract field.

### INT-03

`cross-dock-gr-gate-service.ts` / accepted P2-005 already provide:

- durable idempotent `WmsOutboundCrossDockGrResult` records;
- correlation by source Inbound TU + `CROSSDOCK` settlement source;
- all-task status update for the source;
- safe no-op/rejection for wrong settlement source / unknown source;
- Shipment-specific gate evaluation and Supervisor visibility;
- no Outbound ownership of GR retry.

Preserve these semantics.

### INT-04 / INT-05

`shipment-posting-service.ts` + `erp-shipment-posting-adapter.ts` already provide:

- typed ERP Shipment POST request containing Shipment/TU/item facts and a correlation ID;
- deterministic posting idempotency identity;
- durable `WmsOutboundShipmentPosting` + append-only attempt rows;
- shared orchestration event/retry facts for Shipment posting;
- in-flight/POSTED replay protection;
- explicit `POSTING_ERROR` with structured category/code/message/details on ERP rejection;
- manual Supervisor retry with preserved attempt history;
- technical timeout/error remaining a technical incident while Shipment remains `POSTING_PENDING`.

Do not replace this accepted posting aggregate with another generic mechanism.

### INT-06

`ordering-adapter-service.ts` already:

- correlates external/source order and line identifiers;
- evaluates cancellation against posting/manifest boundaries;
- routes pre-pick to P3 and formally picked/packed to P4;
- requires Supervisor approval for packed cancellation;
- re-evaluates after the accepted P3-003 race when formal pick wins between initial evaluation and authoritative P3 lock;
- passes idempotency and concurrency hooks to accepted release services.

Preserve the P3/P4 business logic. X-002 may add durable inbound-request/correlation audit if missing, but must not create a third cancellation process.

## Required X-002 outcome

For every `INT-01..06`, final evidence must answer all of the following from the **final X-002 head**:

1. **Direction and owner** — inbound-to-Outbound, Outbound-to-Inbound, or Outbound-to-ERP; which bounded context owns retry/business decision.
2. **Native correlation identity** — use the architect/current accepted identity for that boundary; do not collapse all six integrations into one invented global business key.
3. **Durable fact/effect** — exact persisted row/fact that lets an operator/test trace the message to its business object/effect after process restart.
4. **Idempotency/replay behavior** — duplicate input or retry cannot create a second business effect.
5. **Safe diagnostic state** — accepted/rejected/pending/technical outcome can be understood without exposing secrets or raw stack traces.
6. **Business-state protection** — diagnostic/audit changes do not create new business states or move responsibility between Outbound, Inbound and ERP/ordering system.

A single physical table is **not** required for all six boundaries. Prefer existing authoritative domain rows where they already provide durable correlation. Use existing `wms_orchestration` facts where semantically appropriate; do not misuse an outbound-delivery/outbox fact as an inbound-message audit merely for uniformity. If a small Outbound-local audit/read-model helper is required, keep it additive and narrowly scoped.

## Implementation strategy

1. Branch Mercato `outbound/x-002` from exact accepted X-001 head above.
2. Build a six-row implementation audit matrix from current code and accepted evidence before editing.
3. Mark each boundary `SUFFICIENT` or a concrete gap such as:
   - missing durable correlation after restart;
   - missing idempotent inbound-request record;
   - missing safe error/result persistence;
   - missing Supervisor-required visibility;
   - inconsistent correlation between accepted domain fact and adapter payload.
4. Change only real gaps. Do not refactor green boundaries merely to make file names or tables look uniform.
5. Preserve each boundary's native correlation:
   - INT-01: receipt/source correlation + source TU/SKU/item;
   - INT-02: source TU/finalization correlation;
   - INT-03: source Inbound TU + settlement source `CROSSDOCK`;
   - INT-04/05: Shipment posting correlation + posting/idempotency identity;
   - INT-06: source system + external order/line identity (or accepted internal IDs where explicitly supported) + request idempotency.
6. Persist only safe diagnostic data. Never store/print credentials, auth headers, connection strings or raw secret-bearing external payloads in evidence/UI.
7. Supervisor diagnostics may expose architect-required operational correlation/status/error data. Scanner must show only actionable user-flow errors; do not expose integration internals on RF.
8. Do not create automatic ERP retry: retry from `POSTING_ERROR` remains explicit Supervisor action. Do not create Outbound GR retry: that remains Inbound.
9. Do not invent real external ERP/OMS endpoints or credentials. Use the accepted typed adapter/testing seam for executable contract proof.
10. If shared `wms_orchestration` **implementation/schema** is changed, run the directly relevant accepted Inbound orchestration/GR regressions. Merely reading/reusing existing shared entities from Outbound does not justify a broad Inbound sweep.

## Decisive tests

Create a focused real-PostgreSQL X-002 contract/integration suite, or extend the smallest owning suites if that produces clearer evidence. Do not duplicate entire historical fixtures unnecessarily.

At minimum prove:

### INT-01

- real accepted `IN_CROSS_DOCK` source uses ASN-declared quantity and receipt/source correlation;
- replay with the same accepted idempotency identity creates no second binding/effect;
- persisted fact can be correlated back to source TU + demand after a fresh EntityManager/read.

### INT-02

- source finalization durably exposes exactly `confirmedQty` + `damagedQty` with source correlation;
- duplicate finalization/replay does not publish/create a second effect;
- contract assertion explicitly proves residual is derived on Inbound side and is not introduced as a third integration field.

### INT-03

- valid CROSSDOCK GR result correlates to the source and updates accepted gate state idempotently;
- wrong settlement source and unknown source produce zero business mutation;
- duplicate result creates no second durable effect;
- evidence explicitly proves Outbound did **not** create GR retry ownership.

### INT-04 / INT-05

- unique Shipment POST correlation/idempotency persists through request/attempt/result;
- explicit rejection preserves safe structured diagnostics and `POSTING_ERROR`;
- Supervisor retry appends one new attempt and can reach `POSTED` without duplicating accepted business effect;
- technical timeout/error remains distinct from rejection and does not fabricate `POSTING_ERROR`;
- duplicate/in-flight calls remain exactly-once using the accepted X-001/P1-014 protection.

### INT-06

- external order/line cancellation correlation selects the correct line;
- `pickedQty = 0` routes P3; formally confirmed picked quantity routes P4;
- accepted P3-003 race remains correct when formal pick wins after initial evaluation;
- duplicate request/idempotency does not release/put-back twice;
- posting/manifest cancellation boundaries remain unchanged.

### Regression scope

Rerun every owning dedicated suite whose implementation/evidence is materially changed or explicitly claimed as the final proof for X-002. At minimum include directly affected P2 crossdock boundary/GR-gate, P1 ERP posting and P3/P4 cancellation suites when their paths are touched.

Do **not** run the full WMS Outbound acceptance sweep in X-002. That is `ACC-001`, and before `ACC-001` there is an Owner-mandated non-catalog raw-PG SSL maintenance gate.

If no user-visible behavior changes, no new Playwright is required merely to prove backend correlation. If Mercato Supervisor diagnostics are changed, add/rerun the smallest rendered Mercato journey that proves the new visible status/error/correlation surface through real UI + persisted state. Scanner remains frozen unless an actual RF user-flow defect is exposed.

Run current required Mercato typecheck/build/generate steps appropriate to the changed surface. Rebuild/restart Testing only when changed runtime/API/UI requires it.

## Evidence

Write `WMS_Outbound/05_EVIDENCE/X-002_EVIDENCE.md` containing:

- exact Mercato branch/head and frozen Scanner head;
- six-row `INT-01..06` matrix: authority, direction/owner, correlation identity, durable fact, idempotency behavior, safe diagnostics, owning tests, code changed yes/no;
- exact PostgreSQL test commands/results on final head;
- exact changed files and why each change was necessary;
- directly affected regressions;
- shared Inbound regressions only if shared implementation/schema was changed;
- typecheck/build/runtime/UI evidence as actually applicable;
- explicit traceability note that INT-04/05 current authority is P1 STEP 11A, not stale STEP 13;
- explicit statement that Outbound did not take over Inbound GR retry or invent external integration behavior.

Do not claim Supervisor FINAL PASS or Owner Acceptance.

## Completion contract

Return COMPLETE only when:

- all six `INT-01..06` have decisive final-head executable contract/correlation evidence;
- every real gap exposed by the audit is fixed within scope;
- duplicate/retry paths create no duplicate business effect;
- directly affected regressions and required build/typecheck are green;
- evidence is pushed;
- Mercato `outbound/x-002` is pushed;
- Scanner remains frozen unless a real in-scope RF defect required change;
- no post-X-002 SSL maintenance and no `ACC-001` work has started.

Ordinary in-scope test/fixture/tooling/runtime/build/evidence failures are executor-owned self-repair. Stop only at the normal two-strikes same-material-path boundary or an Owner-controlled boundary.

For a fresh Claude Code session: load current `fetch_me_prompt` + `operational-mode` first, then current project steering + `wms-outbound` + `architecture-context`; load `scanner-context` only if Scanner becomes materially relevant. Desktop Commander MCP remains only the approved Claude transport fallback for otherwise-authorized operations, never a permission escalation.

## Mandatory boundary after X-002 acceptance

After X-002 receives Supervisor FINAL PASS **and explicit Owner Acceptance**, do not start `ACC-001` immediately. The Owner-mandated non-catalog maintenance gate must run first:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

Required sequence:

`X-002 -> raw pg SSL maintenance gate -> ACC-001 -> ACC-002 -> ACC-003 -> ACC-004`
