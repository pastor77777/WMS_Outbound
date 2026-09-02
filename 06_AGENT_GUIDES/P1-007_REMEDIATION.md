# P1-007 Remediation — Fourth Override (Final Narrow Set)

Owner explicitly overrides STOP again for one more **extremely narrow P1-007 remediation**.

Use the **same existing Antigravity session**. Do not restart, replace, or create another session.

## FIRST: synchronize locally

1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main`, preserving unrelated local work.
3. Verify this LOCAL file is current:
   `06_AGENT_GUIDES/P1-007_REMEDIATION.md`
4. Read this LOCAL file completely.
5. Refresh current Architect/canon plus current evidence/testing/operations contract.
6. Remediate only the remaining blockers below.

## Current bases

Preserve these exact current heads as the base for this override:

- Mercato `outbound/p1-007`: `378d7a2d1996ccfbe3bc1184a37cddb376d55a2f`
- Scanner `main`: `0f1ca673ca8bbc1de8b77c99ca91e9d8ae00a6df`
- Current P1-007 evidence: `7b68eaf4ff38cae82695ebd622f52e6af6a02706`
- Accepted P1-006 heads remain immutable.

Preserve what is already good:

- per-location `2 + 2 < 3` guard now exists;
- real ATP excludes non-POSTED/non-storage stock;
- retry hierarchy customer -> warehouse -> default 1;
- strict SAME-K two-connection PostgreSQL lock proof;
- canonical Supervisor route auth + `wms_outbound.manage_orders` + mandatory HTTP key;
- R46 zero sets commercial/execution quantity to 0;
- R46 positive correction rejects anything other than authoritative pickedQty;
- R47 preserves commercial ordered quantity and restores uncovered demand conceptually;
- physical-return handoff;
- Scanner replacement-task continuation to cumulative picked quantity;
- migration and R48 seam.

Do not rework these unless required by the blockers below.

---

## 1. R44 / R46 / R47: use one accepted hard-reservation/recovery primitive — no ad-hoc Allocation or InventoryReservation writes

### Current defect

The current code still directly mutates/creates `WmsOutboundAllocation` and `WmsInventoryReservation` in shortage recovery paths.

In R44 it also calculates a location's `available_qty` from eligible inventory movements but does **not subtract existing hard reservations that already consume the same stock**. Therefore a location can appear to cover the shortage while its available unreserved stock does not.

This is the remaining core product blocker.

### Required implementation

Refactor the **smallest reusable P1-004 application primitive** from the accepted allocation/reservation logic, then use that primitive from both the normal P1-004 path and P1-007 recovery paths.

The primitive must own the invariant, not `confirmPickLine` / Supervisor service code. It must support the exact operations P1-007 needs:

### A. Reserve replacement quantity at one specific location

Given scope + warehouse + SKU/item + target location + quantity + Allocation identity/context:

- serialize using the accepted allocation/SKU locking semantics;
- calculate **eligible unreserved stock**, including subtraction/accounting for active hard reservations;
- validate that the specific target location individually supports the requested quantity;
- create/update authoritative Allocation and linked hard Inventory reservation atomically;
- perform any accepted soft-ATP -> hard conversion needed for the actual business case;
- return the authoritative reservation result;
- fail atomically when capacity/stock is insufficient.

Do not leave a second duplicate reservation implementation in `pick-task-service.ts`.

### B. Shrink/release an existing hard reservation

Given Allocation + target committed quantity:

- reconcile Allocation quantity/status and linked `WmsInventoryReservation` atomically;
- when reducing hard quantity, restore only the correct released/unpicked quantity to soft ATP when Architect requires it;
- when target committed quantity is zero, fully release/remove the hard reservation through the same primitive;
- preserve exactly-once/replay safety under surrounding transaction locks.

Use this in:

- R44 replacement reallocation;
- R46 full cancellation and positive exact-picked correction;
- R47 ALLOW_PARTIAL hard-reservation shrink/release;
- R45/WAIT release paths where the same accepted recovery operation is applicable.

No business-path raw SQL mutation and no direct service-level `allocation.reservedQty = ...` / `inventoryReservation.quantity = ...` as the authoritative write path outside the primitive.

### Required real PostgreSQL proof

1. Location B has physical eligible `5`, but another active hard reservation consumes `3`; shortage is `3` -> B has only `2` unreserved -> **no replacement task/reservation** from B.
2. Location B has enough eligible **unreserved** stock -> exactly one replacement Allocation/hard reservation/PickTask for shortage qty.
3. Existing blocked source remains excluded.
4. R46 exact-picked correction leaves Allocation + hard Inventory reservation exactly at picked quantity through the shared primitive.
5. R46 zero leaves no hard reservation and creates required physical-return handoff.
6. R47 leaves only pickedQty hard committed, restores exact missing quantity to the correct soft/uncovered state, and commercial orderedQuantity remains unchanged.
7. Fresh independent DB reads prove all quantities and no over-reservation.
8. Existing strict concurrency proof remains green after this refactor.

---

## 2. Supervisor idempotency: canonical payload comparison must be null-safe and exact

### Current defect

Current comparisons still allow material null/value mismatches:

- SHORT_ALLOCATED compares `outboundOrderId` only when both sides are non-null;
- SHORT_PICKED compares corrected quantity only when both sides have a value.

Thus the same key can be replayed with `null -> value` or `value -> null` material changes and be treated as the same operation.

### Required implementation

Use one explicit canonical decision payload or deterministic hash stored with the Supervisor decision.

Normalize and compare exact material fields, including explicit nulls:

- supervisorId;
- customerOrderId;
- outboundOrderId (`null` is a real canonical value);
- outboundOrderLineId (`null` where not applicable);
- decision;
- correctedQuantity normalized to fixed numeric form or explicit `null`;
- normalized reason string or explicit empty/null according to route contract.

Rules:

- exact canonical payload + same key -> replay prior committed result;
- **any** canonical field difference, including `null <-> value`, -> 409 conflict;
- missing/empty key still fails HTTP boundary;
- UI stable action key behavior remains intact.

Required focused tests:

- same key/same payload;
- outboundOrderId null -> UUID conflict and UUID -> null conflict;
- correctedQuantity null -> pickedQty conflict and pickedQty -> null conflict;
- different reason;
- different Supervisor actor;
- different target line/order;
- no duplicate audit/business writes.

---

## 3. Mercato Playwright: actually execute every required Supervisor outcome named by the suite

### Current defect

The test names claim broader coverage than the code executes.

Current rendered UI actually performs only:

- SHORT_ALLOCATED -> `ALLOW_PARTIAL`;
- SHORT_PICKED -> `CANCEL_OR_CORRECT` exact-picked.

It does **not** execute the following required normal-UI decisions:

- SHORT_ALLOCATED -> `CANCEL_OUTBOUND_ORDER`;
- SHORT_PICKED -> `WAIT`;
- SHORT_PICKED -> persistent `ALLOW_PARTIAL`;
- SHORT_PICKED -> full cancel `CANCEL_OR_CORRECT` with quantity `0`.

### Required Playwright

Use separate deterministic fixtures where needed. Every decisive action must go through rendered Mercato UI and real backend; DB seeding is allowed only to prepare the pre-decision state.

Cover through UI:

1. SHORT_ALLOCATED -> ALLOW_PARTIAL.
2. SHORT_ALLOCATED -> CANCEL_OUTBOUND_ORDER.
3. SHORT_PICKED -> WAIT.
4. SHORT_PICKED -> CANCEL_OR_CORRECT to authoritative picked quantity.
5. SHORT_PICKED -> CANCEL_OR_CORRECT `0`.
6. SHORT_PICKED -> persistent ALLOW_PARTIAL with mandatory reason.

For each, assert visible success and fresh DB business facts, especially quantities/reservation state for outcomes 3–6.

Keep Scanner P1-007 replacement Playwright green; do not broaden Scanner behavior beyond P1-007.

---

## 4. Run and persist the required targeted regressions; evidence must describe what actually ran

### Current defect

Current evidence records focused P1-007 tests and full `wms_outbound`, but does not persist all targeted shared/cross-ticket regressions explicitly required by the remediation contract.

### Required regression gate

Run and record exact commands + pass counts for:

- focused P1-007 real PostgreSQL suite;
- P1-007 Mercato Supervisor Playwright with all six outcomes above;
- P1-007 Scanner replacement continuation Playwright;
- P1-004 Allocation / hard-reservation focused regression;
- P1-005 PickTask assignment focused regression;
- P1-006 real RF picking + SAME-key/retry-idempotency focused regression;
- P1-001 CustomerOrder aggregation/policy focused regression;
- P1-003 planning / requiredQty focused regression;
- targeted P1-008 TU regression if TU files are touched in this override;
- targeted **Inbound** Inventory/location/warehouse/TU compatibility tests because shared Inventory reservation/allocation primitives are touched;
- full `wms_outbound` regression may remain as an additional gate but is not a substitute for the shared Inbound regression.

Use the current testing contract to choose the exact existing suites; do not invent fake pass labels.

### Durable evidence

Update `05_EVIDENCE/P1-007_EVIDENCE.md` after final implementation with:

- exact final 40-char Mercato and Scanner SHAs;
- evidence commit lineage;
- exact shared reservation primitive introduced/reused and why it preserves P1-004 semantics;
- hard-reservation-aware per-location R44 proof;
- R46/R47 exact reservation reconciliation proof;
- null-safe canonical Supervisor idempotency proof;
- all six real Mercato Supervisor Playwright outcomes;
- real Scanner replacement continuation result;
- strict concurrency/rollback proof;
- all targeted regressions including Inbound compatibility;
- any remaining gap, if one exists.

Evidence label may be `PLAYWRIGHT VERIFIED` only. Never self-declare `HUMAN VERIFIED` or `FINAL PASS`.

---

## Hard boundary / STOP

Do not start P1-009, P1-010, P4, Shipment, Carrier, labels, ERP/manifest, or unrelated redesign.

Do not rewrite accepted P1-006 history.

This override is only for the four blocker groups above.

Push implementation + tests + corrected evidence, then **STOP**.