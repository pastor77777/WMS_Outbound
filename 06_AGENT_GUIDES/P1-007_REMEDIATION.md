# P1-007 Remediation — Third Override (Narrow)

Owner explicitly overrides the second-strike STOP for one more **narrow P1-007 remediation**.

Use the **same existing Antigravity session**. Do not restart, replace, or create another agent session.

## FIRST: synchronize locally

1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main`, preserving unrelated local work.
3. Verify this LOCAL file is current:
   `06_AGENT_GUIDES/P1-007_REMEDIATION.md`
4. Read this LOCAL file completely.
5. Refresh current Architect/canon and current evidence/testing/operations contract before editing.
6. Then remediate only the blockers below.

## Current remediation bases

Preserve these exact current product heads as the base for this override:

- Mercato `outbound/p1-007`: `f841950643cf67147e7ec3d4b690fc333b36a841`
- Scanner `main`: `404b2fff9534dc4fef42fdb64488b9bd88e0b55a`
- Current P1-007 evidence commit: `30dab894197c631768390aeb4fb4e10766f7e71c`
- Accepted P1-006 heads remain immutable.

Preserve what is already good and **do not rework it without necessity**:

- customer override -> warehouse -> default `1` retry hierarchy;
- canonical Supervisor UUID / fail-closed route actor;
- `wms_outbound.manage_orders` route permission;
- mandatory HTTP idempotency key;
- stable UI action key per open Supervisor action;
- physical-return handoff persistence;
- strict two-connection PostgreSQL lock proof for SAME-K short-pick replay;
- R48 `blockSourceLocation=false` seam;
- current P1-007 migration additions.

Fix only the six remaining acceptance blockers below.

---

## 1. R44: replacement location must individually cover the reserved quantity and use the accepted P1-004 reservation path

### Current defect

Current `confirmPickLine` collects eligible ATP rows, computes `totalAvailableStock` across **all** candidate locations, checks whether the total covers the shortage, then selects `candidateLocs[0]` for the replacement PickTask.

This is invalid. Example: shortage `3`, location B has `2`, location C has `2`. Aggregate stock is `4`, so current code proceeds, but the generated task may point to B for `3` even though B only has `2`.

Current reallocation also still mutates/creates `WmsOutboundAllocation` and `WmsInventoryReservation` directly inside `confirmPickLine` instead of reusing the accepted P1-004 reservation/conversion primitive.

### Required behavior

For one replacement PickTask:

- choose a **specific eligible unblocked location** that can authoritatively support the exact quantity reserved for that task;
- if the Architect/current accepted allocation contract allows splitting the unresolved shortage across multiple allocations/tasks, implement that only if current canon explicitly requires it; otherwise require one candidate location to cover the unresolved replacement quantity;
- never use aggregate quantity from several locations to justify one task against a single insufficient location;
- account for hard reservations / already reserved stock using the accepted P1-004 semantics;
- perform replacement hard reservation through the existing accepted Allocation/ATP application primitive, or refactor the smallest reusable primitive from that accepted service and call it from both paths;
- do not duplicate reservation rules in `confirmPickLine`;
- replacement PickTask must reference the exact authoritative Allocation and exact location that backs it;
- if no single eligible location can support the required replacement quantity, do not create a phantom task; escalate according to R44.

### Required proof

Real PostgreSQL tests:

A. shortage `3`, candidate B=`2`, candidate C=`2` -> **no single 3-unit task** may be created;
B. shortage `3`, B=`3+` -> exactly one replacement reservation/task on B for `3`;
C. blocked source with eligible stock is excluded;
D. non-ATP/non-POSTED stock is excluded;
E. fresh DB verifies Allocation + hard Inventory reservation quantity/location consistency and no over-reservation.

Keep existing strict concurrency proof and make sure it still passes after using the accepted reservation primitive.

---

## 2. R47 SHORT_PICKED `ALLOW_PARTIAL`: release the unresolved hard reservation and leave missing demand correctly uncovered

### Current defect

Current SHORT_PICKED `ALLOW_PARTIAL` only:

- sets `CustomerOrder.allowPartialShipment = true`;
- moves the affected `OutboundOrderLine` to `PICKED` when `pickedQty > 0`;
- records the Supervisor decision.

It does **not** settle the unresolved missing part of the hard Allocation/Inventory reservation.

### Required behavior

For persistent `ALLOW_PARTIAL` after SHORT_PICKED:

- picked quantity remains executable and may proceed;
- unresolved missing quantity is no longer held by a stale hard reservation;
- shrink/release the hard Allocation/Inventory reservation using the accepted P1-004/ATP recovery primitive, not ad-hoc field writes;
- keep the still-uncovered commercial quantity available for later planning/backorder handling according to Architect R47;
- do **not** silently shrink `CustomerOrderLine.orderedQuantity` in this outcome;
- future shortages for this CustomerOrder use the persisted `allowPartialShipment=true` without another Supervisor approval;
- fresh DB must show exact picked quantity, exact remaining commercial demand, exact soft/uncovered quantity, and no stale hard reservation for missing stock.

Add a focused real-PG test for this exact quantity lifecycle.

---

## 3. R46 `CANCEL_OR_CORRECT`: finish the commercial + hard-reservation quantity contract

### Current defects

#### A. `correctedQuantity = 0`
Current full-cancel path cancels statuses and releases Allocation, but does not set `CustomerOrderLine.orderedQuantity = 0` as required by the commercial correction decision.

#### B. positive correction == authoritative picked quantity
Current code correctly rejects arbitrary positive values, but then changes `Allocation.reservedQty` to picked quantity without reconciling the linked `WmsInventoryReservation` through the accepted reservation primitive.

### Required behavior

For `correctedQuantity = 0`:

- set the affected `CustomerOrderLine.orderedQuantity = 0` in the same authoritative operation;
- cancel/release the execution quantity;
- release hard reservation using accepted P1-004/ATP recovery semantics;
- preserve physical-return handoff for already-picked goods;
- do not fake physical Inventory return.

For positive correction:

- only authoritative persisted `pickedQty` is allowed;
- update `CustomerOrderLine.orderedQuantity` and `OutboundOrderLine.requiredQty` together;
- hard Allocation and linked `WmsInventoryReservation` must reconcile consistently to the quantity that remains legitimately committed;
- no stale excess hard reservation may survive;
- no arbitrary direct rewrite that bypasses the accepted reservation ledger.

Required tests must fresh-read:

- commercial ordered quantity;
- execution required quantity;
- Allocation qty/status;
- linked hard Inventory reservation qty/status/existence;
- physical-return handoff for zero-cancel;
- rejection of positive quantity different from authoritative picked qty.

---

## 4. Supervisor idempotency: same key must compare the full canonical material payload

### Current defect

The service-level replay check is still too weak.

SHORT_ALLOCATED currently compares only CustomerOrder + decision.
SHORT_PICKED compares CustomerOrder + OutboundOrderLine + decision.

Therefore the same key can be reused with a materially changed payload of the same decision type and be treated as a successful replay.

### Required behavior

Persist and compare a canonical material decision payload (or deterministic hash) for every Supervisor decision.

At minimum compare as applicable:

- canonical Supervisor user UUID;
- CustomerOrder id;
- OutboundOrder id when applicable;
- OutboundOrderLine id when applicable;
- decision type;
- correctedQuantity when applicable;
- normalized reason when reason is material/required.

Rules:

- same key + exact canonical material payload -> return prior committed result exactly once;
- same key + any material difference -> fail closed with conflict;
- concurrent same-key decision cannot duplicate business writes/audit row;
- missing/empty key remains rejected at HTTP boundary;
- UI retry of one pending modal action continues to reuse the same key.

Add focused tests for:

- same key/same payload replay;
- same key/different reason;
- same key/different correctedQuantity;
- same key/different target order/line;
- same key/different Supervisor actor;
- concurrent same-key decision exactly once where practical under current service locking/unique constraint.

---

## 5. Complete the decisive Playwright acceptance paths

### Scanner Playwright

Current test proves Short Pick and DB-side replacement creation, but it stops before the operator consumes the replacement task.

Extend the real rendered Scanner journey, with no decisive route mocks:

1. login / warehouse / Picking / zone;
2. receive original task;
3. bind TU;
4. report explicit SHORT_PICKED through UI;
5. verify visible shortage result;
6. receive/continue the **actual generated replacement PickTask** through the normal Scanner flow;
7. rendered location/task changes to the specific eligible alternative location;
8. pick the unresolved quantity through UI;
9. fresh DB proves exact cumulative picked quantity, old task remains SHORT_PICKED, replacement task completes correctly, and no duplicate replacement work exists.

Use a fixture with **one specific alternative location that individually has enough real ATP stock** so this also proves blocker #1.

### Mercato Supervisor Playwright

Current test only executes SHORT_ALLOCATED `ALLOW_PARTIAL`.

Add real rendered UI coverage for the distinct outcomes needed for TC-060/TC-062:

- SHORT_ALLOCATED `ALLOW_PARTIAL`;
- SHORT_ALLOCATED `CANCEL_OUTBOUND_ORDER`;
- SHORT_PICKED `WAIT`;
- SHORT_PICKED `CANCEL_OR_CORRECT` to authoritative picked qty;
- SHORT_PICKED full cancel (`0`) if best represented as a separate deterministic fixture;
- SHORT_PICKED persistent `ALLOW_PARTIAL` with mandatory reason.

Fixtures may seed deterministic state, but every decisive Supervisor action must be clicked/submitted through the rendered Mercato UI and real backend.

For each path assert the visible result plus decisive fresh DB quantity/state facts.

---

## 6. Durable evidence + required regressions must match the actual final heads

### Current defect

Current `05_EVIDENCE/P1-007_EVIDENCE.md` still reports the **old first-shot heads**:

- Mercato `e9ec...`
- Scanner `e459...`

Actual remediation bases are already:

- Mercato `f841950643cf67147e7ec3d4b690fc333b36a841`
- Scanner `404b2fff9534dc4fef42fdb64488b9bd88e0b55a`

The evidence also does not contain the complete required regression record.

### Required final evidence

After fixing blockers 1–5, update durable P1-007 evidence with:

- exact actual final 40-char Mercato and Scanner SHAs;
- clean lineage from accepted P1-006 via all P1-007 commits;
- exact changed files/migrations;
- R44 per-location ATP proof and accepted Allocation/Inventory reservation path proof;
- customer->warehouse->default retry hierarchy proof (preserve existing good proof);
- strict concurrency/PID/`pg_blocking_pids` proof (preserve existing good proof);
- R47 missing-hard-reservation release proof;
- R46 zero vs exact-picked correction with hard reservation reconciliation;
- full canonical Supervisor idempotency conflict proof;
- real Scanner replacement-task continuation Playwright;
- real Mercato UI outcomes listed above;
- rollback proof;
- physical-return handoff proof;
- R48 proof;
- exact test commands and pass counts.

Run and record at minimum:

- focused P1-007 real PostgreSQL suite;
- P1-007 Scanner Playwright;
- P1-007 Mercato Supervisor Playwright;
- P1-004 Allocation regression;
- P1-005 assignment regression;
- P1-006 RF + idempotency regression;
- P1-001 CustomerOrder aggregation/policy regression;
- P1-003 planning/requiredQty regression;
- targeted P1-008/TU regression if TU code remains touched;
- targeted accepted Inbound Inventory/location/warehouse/TU compatibility regressions for shared primitives touched.

Evidence may say **PLAYWRIGHT VERIFIED**. Never self-declare `HUMAN VERIFIED` or `FINAL PASS`.

---

## Hard boundary / STOP

Do not start or implement:

- P1-009;
- P1-010 packing/repack/QC UI;
- Shipment / Carrier / labels / ERP / manifest;
- P4 PutBackTask lifecycle/FIFO/RF;
- unrelated cleanup/redesign;
- accepted P1-006 history rewrites.

Do not broaden this override beyond the six blockers above.

Push product/test changes + corrected durable evidence, then **STOP**.