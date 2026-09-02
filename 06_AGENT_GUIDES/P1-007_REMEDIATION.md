# P1-007 Remediation — Fifth Override (Four Blockers Only)

Owner explicitly overrides STOP again for one more **extremely narrow P1-007 remediation**.

Use the **same existing Antigravity session**. Do not restart, replace, or create another session.

## FIRST: synchronize locally

1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main`, preserving unrelated local work.
3. Verify this LOCAL file is current:
   `06_AGENT_GUIDES/P1-007_REMEDIATION.md`
4. Read this LOCAL file completely.
5. Refresh current Architect/canon plus current `fetch_me_prompt`, `operational-mode`, `REAL_EVIDENCE_CONTRACT.md`, `.ai/TESTING.md`, and `.ai/OPERATIONS.md` where available.
6. Remediate **only** the four remaining blockers below.

## Current bases

Preserve these exact current heads as the base for this override:

- Mercato `outbound/p1-007`: `6e0ffab5403d79cba2f4e417af50550c946440ae`
- Scanner `main`: `b23325aae1c4f83b79d01b3650dbead3486a1041`
- Current P1-007 evidence: `58eb8a9ee69fb7d1a39ec30a593d34502009120f`
- Accepted P1-006 heads remain immutable.

Preserve what is already good unless one of the four blockers requires a surgical change:

- six distinct Mercato Supervisor UI outcome tests exist;
- Scanner replacement-task continuation is implemented and already reaches cumulative picked quantity;
- customer -> warehouse -> default `1` retry hierarchy;
- strict two-connection PostgreSQL lock proof for SAME-K short-pick confirmation;
- canonical Supervisor route auth / `wms_outbound.manage_orders` / mandatory HTTP idempotency key;
- physical-return handoff;
- R48 seam;
- `reconcileHardReservationInTx` shared recovery primitive;
- migration history before this override.

Do not broaden scope.

---

## 1. SHORT_PICKED Supervisor idempotent replay must use a canonical payload that can actually replay

### Current defect

`ResolveShortPickedDecisionInput` contains `customerOrderId`, `outboundOrderLineId`, decision, correctedQuantity/reason, Supervisor identity and key, but **does not contain `outboundOrderId`**.

Current code builds `incomingPayload` directly from the input, so its `outboundOrderId` is `null`. After the first committed decision, the persisted `WmsOutboundSupervisorDecision.outboundOrderId` is a concrete UUID. The replay comparator then compares:

- incoming `outboundOrderId = null`
- existing `outboundOrderId = <UUID>`

and can reject a genuine same-key/same-user-payload retry as `409 Conflict`.

The existing replay test only proves SHORT_ALLOCATED, so this defect is currently untested.

### Required implementation

Use one canonical payload model whose fields are identical on first execution and replay.

For SHORT_PICKED, derive server-owned material identifiers **before canonical comparison** from the authoritative target line under the same tenant/org scope:

- load the target `OutboundOrderLine` by `outboundOrderLineId`;
- derive its canonical `outboundOrderId` server-side;
- build incoming canonical payload using that derived value;
- persist the same canonical material payload (or deterministic hash) with the committed Supervisor decision;
- on replay, rebuild/compare the same canonical fields.

Do not make the client invent or trust `outboundOrderId` merely to repair the comparison unless current API canon explicitly requires it.

Canonical material fields remain:

- canonical Supervisor UUID;
- CustomerOrder id;
- derived OutboundOrder id where applicable;
- OutboundOrderLine id where applicable;
- decision;
- correctedQuantity normalized to fixed form or explicit `null`;
- normalized reason according to route contract.

Rules:

- same key + exact SHORT_PICKED canonical payload -> return prior committed result, zero duplicate business/audit writes;
- same key + any material difference -> 409;
- explicit `null <-> value` remains a conflict when the field is part of that decision contract;
- missing/empty key remains rejected at HTTP boundary;
- UI pending-action stable key behavior remains unchanged.

### Required focused proof

Add real service/HTTP tests for SHORT_PICKED specifically:

1. first `CANCEL_OR_CORRECT` exact-picked commit with key K;
2. replay the **same user request with the same K** -> successful prior-result replay;
3. fresh DB -> exactly one Supervisor decision and no duplicate reservation/status/handoff writes;
4. same K + different correctedQuantity -> conflict;
5. same K + different reason -> conflict;
6. same K + different Supervisor actor -> conflict;
7. same K + different target line/order -> conflict;
8. same K + decision requiring `null` correctedQuantity vs a value -> conflict where applicable.

Do not accept a test that only covers SHORT_ALLOCATED.

---

## 2. R46 full cancellation must implement the canonical quantity `0` outcome, including schema compatibility

### Current defect

Architect/acceptance requires `CANCEL_OR_CORRECT` with `correctedQuantity = 0` to represent full commercial cancellation of the affected line.

Current code now cancels statuses and releases the hard reservation, but deliberately keeps positive `CustomerOrderLine.orderedQuantity` / `OutboundOrderLine.requiredQty` because existing DB check constraints require values `> 0`.

That means the implementation does not match the required R46 quantity outcome.

### Required behavior

For `correctedQuantity = 0`, in one authoritative transaction:

- `CustomerOrderLine.orderedQuantity = 0`;
- `CustomerOrderLine.status = CANCELLED`;
- `OutboundOrderLine.requiredQty = 0`;
- `OutboundOrderLine.status = CANCELLED`;
- hard Allocation is fully released through the shared reconciliation primitive;
- linked `WmsInventoryReservation` is removed/zeroed according to accepted hard-reservation semantics;
- already-picked physical quantity is represented by the durable physical-return handoff;
- no fake immediate Inventory return;
- no stale hard block survives.

### Schema requirement

If the current database constraints (`ordered_quantity > 0`, `required_qty > 0`, or equivalent) prevent the canonical R46 zero outcome:

- add the **smallest additive/reversible P1-007 migration after the current latest migration** to make zero valid for cancelled commercial/execution lines;
- prefer constraint semantics compatible with normal positive lines while allowing `0` only as a legal stored quantity state (for example `>= 0` if consistent with canon/domain validation);
- do not rewrite old migrations;
- do not weaken unrelated quantity fields/tables;
- DOWN restores the prior constraints only if existing data is compatible, with a clear reversible strategy;
- do not change accepted Inbound/shared schemas unless truly required.

Service/API validation must still reject negative quantities and arbitrary positive correction values.

### Required proof

Real PostgreSQL:

1. start with `requiredQty=10`, `pickedQty=4`, hard reservation `10`;
2. Supervisor full cancel `correctedQuantity=0`;
3. fresh DB: COL ordered quantity `0`, OOL required quantity `0`, both statuses cancelled;
4. Allocation released qty `0`;
5. no linked hard Inventory reservation remains;
6. physical-return handoff qty exactly `4`;
7. no immediate fake stock increment/putback;
8. migration UP -> inspect constraints -> test -> DOWN -> prior constraint restored -> re-UP -> test green.

Mercato Playwright Outcome 5 must fresh-read and assert the zero quantities, not only cancelled statuses.

---

## 3. R44 replacement availability must account for ALL active hard reservations, including pre-PickTask reservations

### Current defect

`resolveLocationEligibleStock()` currently subtracts quantities from active `PickTaskLine`s at the target location. That covers hard reservations which already have an active task/location claim.

It does **not** account for active hard `Allocation/WmsInventoryReservation` quantities that exist before a PickTask is created or otherwise are not represented by an active PickTaskLine.

Therefore a warehouse/location can still be over-reserved. Example:

- eligible physical stock at the only candidate location B = `5`;
- another active hard Allocation/InventoryReservation already owns `3`, but no PickTask exists yet;
- new shortage wants `3`;
- current location query sees `5` and may reserve another `3`, exceeding warehouse hard capacity.

### Required invariant

The shared P1-004 reservation primitive must enforce **both**:

1. target-location physical eligibility/capacity; and
2. warehouse/SKU hard-reservation availability under the accepted P1-004 lock/ledger semantics.

If pre-task hard reservations are warehouse-scoped and cannot be authoritatively attributed to a specific location, do not invent location ownership. Instead enforce a conservative two-bound check under the same SKU/allocation lock:

- target location eligible physical stock minus authoritative location-attributed active claims must cover the requested replacement quantity; **and**
- warehouse eligible stock minus **all active hard Allocation/WmsInventoryReservation commitments** for the SKU must cover the requested replacement quantity.

The final reservable quantity is bounded by the stricter of those two authoritative limits.

If the accepted data model already has a better canonical mapping from hard reservation -> location, use it. Do not add a duplicate shadow ledger merely for this test.

### Required implementation

- keep reservation creation inside the shared allocation primitive;
- under the accepted SKU/advisory/transaction lock, calculate all active hard commitments (`RESERVED`, `SHORT`, `CONFIRMED`, and any accepted active equivalent) for the SKU/warehouse, not only active PickTasks;
- prevent double-counting the same commitment when it is also represented by a PickTaskLine;
- preserve the exact replacement Allocation and target PickTask location relationship;
- no direct fallback reservation writes in `pick-task-service.ts`.

### Required real PostgreSQL proof

Add a decisive case distinct from current 2A5:

A. candidate B physical eligible stock = `5`;
B. another active hard Allocation + linked `WmsInventoryReservation` reserves `3` for same SKU/warehouse;
C. **that other allocation has no PickTask/PickTaskLine**;
D. shortage = `3`;
E. replacement must be rejected/escalated because warehouse unreserved hard capacity is only `2`;
F. fresh DB proves zero new replacement Allocation/PickTask and original competing hard reservation unchanged.

Also retain:

- existing active-PickTask reservation case;
- `2+2 < 3` per-location case;
- eligible single location succeeds when both location and warehouse unreserved bounds cover quantity;
- strict concurrency/lock proof remains green.

---

## 4. Final durable evidence must use exact real heads and record the decisive proofs

### Current defect

Latest durable evidence again contains non-existent/mismatched full SHAs.

Actual current heads before this override are:

- Mercato: `6e0ffab5403d79cba2f4e417af50550c946440ae`
- Scanner: `b23325aae1c4f83b79d01b3650dbead3486a1041`

Do not manually type guessed SHAs into evidence. Resolve exact `git rev-parse HEAD` / remote pushed commit IDs after the final commits and copy those exact 40-char values.

### Required final test/evidence gate

After blockers 1–3 are fixed, rerun and persist exact commands + pass counts for at least:

- focused P1-007 real PostgreSQL suite, including:
  - SHORT_PICKED same-key/same-payload successful replay;
  - conflict variants;
  - pre-PickTask hard-reservation overbooking rejection;
  - R46 true zero quantity fresh DB proof;
  - migration UP/DOWN/re-UP if schema changed;
  - strict concurrency and rollback;
- P1-007 Mercato Supervisor Playwright all six outcomes, with Outcome 5 asserting stored zero quantities;
- P1-007 Scanner replacement continuation Playwright;
- P1-004 Allocation/hard-reservation regression;
- P1-005 assignment regression;
- P1-006 RF + SAME-key/retry-idempotency regression;
- P1-001 CustomerOrder aggregation/policy regression;
- P1-003 planning/requiredQty regression;
- targeted P1-008 TU regression only if TU code/schema is touched;
- targeted accepted Inbound Inventory/location/warehouse/TU compatibility regressions for shared primitives touched;
- full `wms_outbound` regression as an additional gate.

Update `05_EVIDENCE/P1-007_EVIDENCE.md` with:

- exact final real 40-char Mercato SHA;
- exact final real 40-char Scanner SHA;
- exact evidence commit lineage/base;
- migration name + UP/DOWN/re-UP output if added;
- SHORT_PICKED retry replay proof showing exactly one decision/business application;
- null/value conflict proof;
- R46 stored `0` quantity proof;
- pre-PickTask hard-reservation R44 rejection proof;
- six Mercato UI outcomes;
- Scanner replacement continuation;
- strict PG concurrency/rollback;
- all required targeted regressions including Inbound;
- any remaining gap, if one exists.

Evidence label may say `PLAYWRIGHT VERIFIED`; never self-declare `HUMAN VERIFIED` or `FINAL PASS`.

---

## Hard boundary / STOP

Do not start P1-009, P1-010, P4, Shipment, Carrier, labels, ERP/manifest, packing/repack/QC, or unrelated redesign.

Do not rewrite accepted P1-006 history.

This override is only for the four blocker groups above.

Push implementation + tests + corrected durable evidence, then **STOP**.