# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-06 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.  
Formal progress: **32/37 FINAL PASS / Owner Accepted**.

Latest accepted:

- `P4-002` — item 30/37 — Mercato `d75dccbc7a43b64c2a0ed325d30a7a4b49b57eb1` / Scanner `7d13e34fc66fe19149b43a6747b5300c2bdcf945` / evidence `0961a7fe2e08395cd1b2522e30770d62c2fb2841`.
- `P4-003` — item 31/37 — Mercato product candidate `6ddec6870b5498901224b3457f5955101204a2f0` / Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8a916dfc9387b09d92b9e9143010cc34145108bd`.
- `X-001` — item 32/37 — **FINAL PASS / Owner Accepted** — Mercato `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` / Scanner frozen `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8fcd97942545d9e49bcd819f2898dd8887843165`.

## X-001 accepted truth

- All `CON-01..05` have decisive executable final-head evidence.
- CON-02: genuine overlapping `generatePickTasks` vs `releaseAllocation` PostgreSQL race proves Allocation row serialization and immutability after PickTask creation.
- CON-03: genuine overlapping cross-dock `planBinding` and `assignNext` races prove real PostgreSQL row/advisory-lock waits and exactly-one durable assignment/planning.
- CON-01/04/05 accepted guards remain unchanged and were rerun on final head.
- Required dedicated X-001 set: **148/148 PASS**; typecheck clean.
- No business UI/API/schema change; Scanner frozen.
- Raw `pg.Client` SSL compatibility issue discovered during final reruns was fixed at the four known test call sites. Centralization is deliberately scheduled after X-002 and before ACC-001.

## Exact next item — X-002 grounded/prepared, not launched

**X-002 — Integration correlation, observability and operational recovery** — item **33/37**.

Detailed execution guide:

`06_AGENT_GUIDES/X-002_EXECUTION.md` @ `0fff59ebfeddc7104e6396c3882d8defeaf8145e`

Mercato base: `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` -> future `outbound/x-002`.  
Scanner: frozen by default at `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.

### X-002 governing integration boundaries

- `INT-01` — P2 STEP 1: accept only Inbound-qualified `ELEMENTARY` TU in `IN_CROSS_DOCK`; source quantity is ASN-declared; retain TU/SKU/item + receipt correlation.
- `INT-02` — P2 STEP 3: source settlement passes Inbound `confirmedQty` + `damagedQty` with source correlation; residual is derived by Inbound, not an extra contract field.
- `INT-03` — P2 STEP 4: GR outcome correlation is `sourceInboundTU + GR_SETTLEMENT_SOURCE=CROSSDOCK`; Outbound owns its gate consumption only; GR retry/transport remains Inbound-owned.
- `INT-04` — current P1 STEP 11A: uniquely identifiable Shipment POST to ERP.
- `INT-05` — current P1 STEP 11A: explicit rejection -> durable safe diagnostics + `POSTING_ERROR`; retry is explicit Supervisor action; acceptance -> `POSTED`; timeout remains technical incident, not business rejection.
- `INT-06` — P3 STEP 1 + P4 STEP 1: external cancellation/correction correlates to proper line and routes by formally confirmed picked quantity, including accepted P3-003 race handling.

### Existing implementation to preserve

X-002 is audit-and-harden, not a new bus:

- INT-01 already has `receiptCorrelation`, ASN-declared source quantity, source TU/item and idempotent binding in `cross-dock-eligibility-service.ts`.
- INT-02 already has durable source `WmsOutboundCrossDockFinalization` with quantity conservation in `cross-dock-execution-service.ts`.
- INT-03 already has durable/idempotent `WmsOutboundCrossDockGrResult`, Shipment gate and Supervisor visibility in accepted P2-005.
- INT-04/05 already have typed ERP adapter, posting aggregate/attempt history, correlation/idempotency, shared orchestration fact/retry rows, safe rejection detail and Supervisor retry in `shipment-posting-service.ts`.
- INT-06 already has deterministic external/internal order-line correlation and P3/P4 routing in `ordering-adapter-service.ts`.

Executor must first audit each boundary and change only concrete missing durable correlation/audit/diagnostic gaps. A single universal table or correlation key is not required and must not be invented merely for uniformity.

### Traceability defect

Derived requirement/catalog references for INT-04/05 still say P1 STEP/KROK 13. Current P1 v1.20 puts ERP posting in **STEP 11A**; STEP 13 is dispatch/final settlement. Current P1 process prose wins. Classification: traceability-reference defect, not product blocker.

## Testing / execution boundary

X-002 has **not been implemented/launched** yet.

Before first implementation action:

1. canonical new-item Testing reset from `Devaxonic-WMS/.ai/OPERATIONS.md` -> required `RESET_OK`;
2. fresh Claude session, if used: load current `fetch_me_prompt` + `operational-mode` first, then current WMS steering + `wms-outbound` + `architecture-context`; `scanner-context` only if Scanner becomes relevant;
3. execute only `06_AGENT_GUIDES/X-002_EXECUTION.md`;
4. no broad full Outbound acceptance sweep in X-002; only focused INT contract proof + directly affected regressions/build/UI if applicable;
5. supervisor independently verifies remote Git/diff/evidence; Owner Acceptance remains explicit.

If shared `wms_orchestration` implementation/schema changes, require the relevant accepted Inbound regressions. Reusing existing shared entities without changing shared code does not justify a broad Inbound sweep.

## Mandatory post-X-002 gate

After X-002 is FINAL PASS + Owner Accepted, before ACC-001:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

Sequence is fixed:

`X-002 -> raw pg SSL maintenance gate -> ACC-001 -> ACC-002 -> ACC-003 -> ACC-004`

The SSL maintenance gate is non-catalog and does not change the 37-item count.

## Durable operating rules

- Executor `COMPLETE` != Supervisor PASS != Owner Acceptance.
- Owner controls executor/venue/launcher/session mechanics.
- Detailed logic belongs in Git guide; Owner-facing executor prompt stays microscopic.
- Ordinary in-scope implementation/test/runtime/build/evidence failures are executor-owned self-repair.
- Provider/session/quota interruption preserves exact checkpoint; never restart from accepted base.
- Testing credentials frozen; canonical Testing only; no local PostgreSQL.
- Inbound remains CLOSED / REFERENCE; Demo/Prod out of scope without explicit Owner authorization.

**Git truth overrides stale Drive/chat history.**
