# P4-003 completion correction — resolve physical-return protection

**Same item:** P4-003 only. This is a continuation, not a new Task Catalog item. **No new Testing reset.**

## Exact continuation state

- Mercato `outbound/p4-003` @ `203caba6365428bb32679634d5e6f573c03e67c1`
- Scanner `outbound/p4-003` @ `a2759a29347285dd1dcd14bf51633431fbf2a302` — freeze unless a real UI regression requires a change
- Current P4-003 evidence @ `e7f1dd7b7d100b7fd4f5fa601c49b39fc0659a2b`
- Accepted P4-002 base remains unchanged.

## Supervisor-found acceptance gap

P4-001 introduced `wms_outbound_physical_return_handoffs` as a temporary protection boundary: pending handoff quantity is subtracted from ATP/allocation/source availability while physically picked goods are outside ordinary storage.

Current P4-003 completion adds the recovered quantity to `wms_inventory`, marks `PutBackTask COMPLETED`, and releases the shared task lock, but it does **not** resolve the physical-return handoff. Current availability code still subtracts every matching handoff unconditionally. Therefore after successful physical put-back the newly recovered stock can remain logically unavailable, contradicting P4 R8 / STEP 5: after completion goods become ordinary `Inventory AVAILABLE` stock.

This is a real P4-003 correctness gap and must be fixed before Supervisor FINAL PASS.

## Required correction

Implement the smallest additive auditable resolution mechanism for `WmsOutboundPhysicalReturnHandoff` rather than deleting audit history. Prefer a nullable technical `resolvedAt` / `resolved_at` marker (or an equally small existing-compatible technical marker if inspection proves one already exists).

On successful `submitLocation` completion, inside the **same transaction** as:

1. validated-location acceptance,
2. Inventory adjustment,
3. `PutBackTask LOCATION_VALIDATION -> COMPLETED`, and
4. shared task-lock release,

resolve exactly the task's source handoff. The handoff must remain unresolved on rejected location, hard failure, rollback, wrong operator, and before completion.

Update **every** P4-001 pending-return protection query to subtract only unresolved handoffs. Search all product code for `wms_outbound_physical_return_handoffs` / `WmsOutboundPhysicalReturnHandoff`; do not patch only one service. At minimum re-check ATP reservation, Allocation availability and PickTask/source availability logic. `reconcilePendingHandoffs` must treat only unresolved handoffs as pending recovery work.

Do not change P4 business states, FIFO, no-zone assignment, Inbound semantics, or Scanner UX.

## Decisive proof

Extend the real PostgreSQL P4-003 suite with proof that:

- before valid completion, the handoff is unresolved and its quantity is protected from ATP/allocation;
- rejected locations leave it unresolved and protected;
- valid completion atomically sets the handoff resolution marker and exposes exactly the recovered task quantity as ordinary available stock;
- a subsequent allocation/ATP calculation can consume that recovered quantity once, not zero and not double;
- duplicate completion remains idempotent and does not re-resolve or double-add stock;
- concurrent completion still creates one Inventory movement and one handoff resolution;
- forced rollback after real writes leaves task, Inventory, task lock **and handoff resolution** unchanged.

Run P4-003 dedicated PostgreSQL plus P4-001, P4-002, P3-003 and the same shared assignment/Inventory regressions already required by the main P4-003 guide. Re-run the P4-003 rendered Playwright journey against the rebuilt exact runtime; re-run P4-002 rendered regression if the runtime/backend delta invalidates its prior proof.

Update `05_EVIDENCE/P4-003_EVIDENCE.md` with exact final SHAs and the resolved-handoff proof. Push changed repos and STOP. This remains executor COMPLETE only; do not mark Owner Accepted and do not start the next item.
