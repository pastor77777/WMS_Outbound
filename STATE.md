# WMS Outbound — STATE

**As of:** 2026-09-06  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **32/37 items FINAL PASS / Owner Accepted**

## Architect baseline

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19
- requirements: **109 IDs = 98 FR + 6 INT + 5 CON**

Inbound remains **CLOSED / REFERENCE**. `PickWave` is out of scope v1. No separate Process 5 exists.

## Latest accepted checkpoints

Earlier accepted checkpoints remain unchanged in Git history.

- `P3-003` — catalog item 28/37 — Mercato `f600782496e865603200d46d7e6041a54f90b9a4` / Scanner `135d86e1342bae7b21a8b676b1ef220a39a0f0b5` / evidence `40d4fd10430ab2b58457f96573ed34af8f57d316` — FINAL PASS / Owner Accepted.
- `P4-001` — catalog item 29/37 — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801` — FINAL PASS / Owner Accepted.
- `P4-002` — catalog item 30/37 — Mercato `d75dccbc7a43b64c2a0ed325d30a7a4b49b57eb1` / Scanner `7d13e34fc66fe19149b43a6747b5300c2bdcf945` / evidence `0961a7fe2e08395cd1b2522e30770d62c2fb2841` — FINAL PASS / Owner Accepted.
- `P4-003` — catalog item 31/37 — Mercato product candidate `6ddec6870b5498901224b3457f5955101204a2f0` / Scanner `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8a916dfc9387b09d92b9e9143010cc34145108bd` — FINAL PASS / Owner Accepted.
- `X-001` — catalog item 32/37 — **FINAL PASS / Owner Accepted** — Mercato `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` / Scanner frozen `a2759a29347285dd1dcd14bf51633431fbf2a302` / evidence `8fcd97942545d9e49bcd819f2898dd8887843165`.

## X-001 accepted boundary

- `CON-01..05` now have decisive executable final-head evidence.
- `CON-02` was hardened with a genuine overlapping PostgreSQL race for PickTask creation versus Allocation release.
- `CON-03` was hardened with genuine overlapping PostgreSQL plan/assignment races, distinct backend PIDs and `pg_blocking_pids`/lock-wait proof.
- `CON-01`, `CON-04`, `CON-05` existing authoritative guards remained unchanged and their owning suites were rerun on final head.
- Final dedicated X-001 acceptance set: **148/148 PASS**; Mercato typecheck clean.
- X-001 changed no UI/API/schema business behavior. Scanner remained frozen.
- Raw `pg.Client` SSL breakage discovered during final reruns was corrected at the four known test call sites only; durable centralization is a separate mandatory post-X-002 maintenance gate, not part of X-001.

## Exact next item — X-002 grounded and prepared, not launched

**X-002 — Integration correlation, observability and operational recovery** — catalog item **33/37**.

Execution guide:

`06_AGENT_GUIDES/X-002_EXECUTION.md` @ `0fff59ebfeddc7104e6396c3882d8defeaf8145e`

Execution bases:

- Mercato accepted base: `outbound/x-001` @ `b56515ecffba729c828f5fe9ca0ec6471edc4c43` -> future `outbound/x-002`.
- Scanner frozen by default: `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302`.

X-002 is **audit-and-harden**, not a new integration bus. Existing accepted boundaries already provide substantial implementation:

- `INT-01`: cross-dock eligibility consumes `ELEMENTARY` `IN_CROSS_DOCK` source TU, ASN-declared quantity, `receiptCorrelation` and idempotency.
- `INT-02`: source-TU finalization durably records declared/confirmed/damaged/residual reconciliation; outbound contract remains confirmed + damaged + source correlation, while Inbound derives residual.
- `INT-03`: accepted P2-005 GR correlation is source Inbound TU + `GR_SETTLEMENT_SOURCE=CROSSDOCK`, durable/idempotent, Shipment-gate scoped; GR retry remains Inbound-owned.
- `INT-04/05`: accepted Shipment posting has typed ERP request, durable posting + attempts, correlation/idempotency, orchestration fact/retry history, safe structured rejection diagnostics, manual Supervisor retry and technical-timeout separation.
- `INT-06`: ordering adapter correlates external order/line, routes by formal picked quantity to accepted P3/P4 and preserves the accepted P3-003 race re-evaluation.

X-002 must identify concrete correlation/audit/diagnostic gaps at those boundaries and add only missing durable proof/visibility. It must not redesign accepted business flows or transfer ownership between Outbound, Inbound, ERP or the ordering system.

### X-002 traceability correction

Derived `INT-04` / `INT-05` references still point to P1 STEP/KROK 13. Current P1 v1.20 governs: ERP Shipment POST and retry are **STEP 11A**; STEP 13 is dispatch/final settlement. This is a traceability-reference defect, not a product blocker. Do not implement STEP 13 semantics as INT-04/05.

## Mandatory execution boundary

X-002 grounding/guide preparation does **not** replace the new-item Testing hygiene gate.

Before first X-002 implementation action:

1. refresh current Git;
2. run the canonical new-item Testing reset from `Devaxonic-WMS/.ai/OPERATIONS.md` and require `RESET_OK`;
3. for a fresh Claude session load current `fetch_me_prompt` + `operational-mode` first, then current steering + `wms-outbound` + `architecture-context`; load `scanner-context` only if Scanner becomes materially relevant;
4. execute only `06_AGENT_GUIDES/X-002_EXECUTION.md`;
5. independently verify remote Git/evidence before Supervisor FINAL PASS; Owner Acceptance remains explicit.

Do not run the full Outbound acceptance sweep during X-002.

## Mandatory post-X-002 pre-ACC gate

After X-002 receives Supervisor FINAL PASS and explicit Owner Acceptance, execute the non-catalog maintenance gate before `ACC-001`:

`07_IMPLEMENTATION_PLAN/POST_X002_PRE_ACC_RAW_PG_SSL_MAINTENANCE.md`

Required sequence:

`X-002 -> raw pg SSL maintenance gate -> ACC-001 -> ACC-002 -> ACC-003 -> ACC-004`

This gate does not change the 37-item count.

## Durable operating rules

- Executor `COMPLETE` != Supervisor PASS != Owner Acceptance.
- Owner controls executor/venue/launcher/session mechanics.
- Full authorized Task Catalog item is the normal execution objective; ordinary in-scope failures are executor self-repair.
- Provider/session/quota interruption preserves exact branch/HEAD/workspace checkpoint.
- Testing credentials remain frozen during implementation.
- Canonical Testing only; no local PostgreSQL; Demo/Prod require separate Owner authorization.
- `architecture-context` is shared/Inbound technical reference only; Outbound Architect/Canon remains business authority.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.
