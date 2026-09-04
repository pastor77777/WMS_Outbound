# P1-015 — CarrierManifest lifecycle and dispatch boundaries

**Execution status:** authorized first implementation shot for `P1-015`  
**Catalog item:** `18/37`  
**Prerequisite:** `P1-014` is `FINAL PASS / Owner Accepted`  
**Scope type:** implementation + evidence; **NOT acceptance steering**  
**Workflow:** follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

## Mandatory fresh-session startup

This task must run in a **fresh Antigravity session**.

Canonical startup:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
/home/ubuntu/.local/bin/agy-pl
```

Never start with bare `agy`.

Before changing anything, refresh and read current Git state in this order:

1. `Devaxonic-WMS/AGENTS.md`
2. `Devaxonic-WMS/.ai/STATE.md`
3. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`
4. `Devaxonic-WMS/.ai/TESTING.md`
5. `Devaxonic-WMS/.ai/OPERATIONS.md`
6. `WMS_Outbound/STATE.md`
7. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-04.md`
8. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
9. this guide
10. current P1 Task Catalog / Architect KROK 12–13 / state model / requirements / acceptance scenarios
11. current Mercato `wms_outbound` Shipment, TU, OutboundOrder, state-transition and P1-013/P1-014 services
12. current Mercato/Scanner/WMS exact Git refs

Do not reconstruct mutable state from old chat history.

## Frozen accepted starting point

Owner accepted `P1-014` after supervisor review.

- accepted P1-014 Mercato head: `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`
- accepted P1-014 branch: `outbound/p1-014`
- accepted P1-014 WMS evidence commit: `b97f1640621b5b01571efec313b7fa0325c1aedf`
- accepted P1-013 Mercato base: `5e6b70aa81afd28fe3217e4aad216e8a6482a769`
- Scanner accepted/frozen reference: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` on `outbound/p1-009`
- Devaxonic-WMS steering after P1-014 acceptance: `3f46432cc75899c80be38ec9206d61b9a544f416`

Before implementation, verify these refs still exist and that the P1-015 branch rule below can be satisfied exactly. If the accepted Mercato head has unexpectedly moved or been rewritten, STOP and report it. Do not rebase around an unexpected ref.

## Mercato branch rule

Create/use:

`outbound/p1-015`

It must start from exact accepted P1-014 Mercato SHA:

`bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`

Expected merge base with the final P1-015 head is exactly that SHA.

Do not merge, squash or rebase accepted P1-014 history. Do not transplant unrelated work.

## Scanner boundary

Default: **Scanner is frozen** at:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Task Catalog says Scanner action is not required unless loading handover is already Scanner-owned in the current accepted product. Inspect ownership before editing anything. Do not invent a Scanner manifest flow merely because physical handover occurs at a dock.

If current authoritative product ownership does not already require Scanner, make no Scanner commit.

## Task Catalog grounding

`P1-015 — CarrierManifest lifecycle and dispatch boundaries`

Objective:

- implement one-manifest Shipment assignment,
- implement `CarrierManifest OPEN → CLOSED → HANDED_OVER → CONFIRMED`,
- make `CLOSED` an irreversible composition boundary,
- keep physical handover distinct from final warehouse confirmation,
- make duplicate/parallel manifest lifecycle commands exactly-once/non-regressive.

Architect/source:

- P1 KROK 12 — `CarrierManifest`
- P1 KROK 13 — physical handover / warehouse confirmation
- P1 R39–R40
- manifest-side trigger context for R70/R72 without absorbing P1-016 settlement
- state model `CarrierManifest`, `Shipment`, `TU`, `OutboundOrder`

Task-traced requirements:

- `FR-P1-20`
- `FR-P1-44`
- `FR-P1-46`
- `CON-04`
- `CON-05`

Dependency:

- `P1-014` — satisfied and accepted.

Target components:

- target `CarrierManifest` persistence/domain service,
- Shipment manifest membership,
- dispatch lifecycle UI in Mercato,
- existing authoritative state-transition/event foundation.

Task Catalog acceptance mapping:

- `TC-001`
- `TC-008`
- `TC-128`
- `TC-129`
- `TC-132`
- `TC-133`

Important decomposition note: `P1-016` is a separate catalog item whose explicit target components are Inventory ledger, Allocation, OutboundOrder/Line and CustomerOrder/Line and whose objective is final quantity/order settlement. P1-015 therefore owns the **manifest-side lifecycle and exactly-once confirmation trigger/context**, not the downstream final settlement consumer. Do not claim the terminal settlement portions of `TC-128/129/132/133` are fully implemented by P1-015; prove the manifest-side prerequisite/boundary now and leave final settlement assertions to P1-016.

## Architect truth to implement

### 1. Target CarrierManifest state machine

Use the exact Architect states/events:

- creation/open: `(none) → OPEN` / `CarrierManifestOpened` / actor Dispatcher
- close: `OPEN → CLOSED` / `CarrierManifestClosed` / actor Dispatcher
- physical handover: `CLOSED → HANDED_OVER` / `CarrierManifestHandedOver` / actor Dispatcher/Carrier boundary
- warehouse confirmation: `HANDED_OVER → CONFIRMED` / `CarrierManifestConfirmed` / actor Dispatcher

`CONFIRMED` is final. No reverse transitions.

Do not invent intermediate states such as `LOADING`, `READY`, `SENT`, `ACKNOWLEDGED`, `CLOSING`, `CONFIRMING` or provider-specific states unless the current Architect source explicitly contains them. It does not authorize an external carrier API in this task.

### 2. Manifest creation and membership

Dispatcher may open a target `CarrierManifest` in `OPEN`.

Only a Shipment in exact state `POSTED` may be added to an `OPEN` manifest.

Successful assignment must:

- establish one durable target manifest membership,
- transition Shipment `POSTED → IN_MANIFEST`,
- emit exactly one `ShipmentAddedToManifest` transition fact,
- preserve organization/tenant/warehouse scope,
- be idempotent for a replay of the same logical assignment,
- reject reassignment of the same Shipment to a second manifest.

One Shipment belongs to exactly one manifest (`FR-P1-20`).

Current `WmsOutboundShipment` already contains a nullable `manifestId`. Inspect it before designing persistence. Prefer making the existing target field authoritative if it is semantically correct rather than creating a second conflicting membership truth. A target CarrierManifest table/entity will still be required if it does not yet exist.

Do **not** make legacy `carrier_shipments` or another provider/legacy table the target CarrierManifest truth merely because it exists.

### 3. `OPEN → CLOSED` is the irreversible composition boundary

Closing is a business boundary, not physical handover.

After a manifest is `CLOSED`:

- no Shipment may be added,
- no Shipment may be removed/reassigned,
- the manifest cannot reopen,
- the manifest cannot be cancelled through a newly invented path,
- Shipment cancellation through WMS must remain blocked at/after this boundary,
- Shipment Carrier change is forbidden at/after this boundary.

The system must not rely only on UI button hiding. Server/domain guards must enforce the boundary.

Do not invent a rule that a manifest must be non-empty before closing unless current Architect source explicitly requires it. Do not invent capacity, route, trailer, dock, license-plate or driver requirements.

### 4. Complete the deferred P1-013 Carrier-correction boundary

P1-013 explicitly deferred the exact post-`CarrierManifest.CLOSED` enforcement to P1-015.

Inspect the accepted P1-013 carrier correction service and preserve:

- real server-authoritative Warehouse Supervisor requirement,
- optional correction reason under the accepted rule,
- no automatic label regeneration or reprint.

P1-015 must make the manifest-aware boundary explicit:

- pre-close correction remains allowed where the Architect rule and current accepted state model make it valid,
- a Shipment associated with an `OPEN` manifest must not be rejected merely because the manifest object now exists if the only reason is the previously missing manifest persistence,
- once its manifest is `CLOSED`, `HANDED_OVER` or `CONFIRMED`, Carrier correction must fail closed,
- do not silently remove/reassign manifest membership as part of Carrier correction,
- do not automatically repost ERP, regenerate label or reprint label; those effects are not authorized by this task.

If enabling the real loading-time pre-close correction requires narrowly extending the accepted P1-013 correction service to `POSTED` / `IN_MANIFEST` with an `OPEN` manifest, that narrow extension is authorized here **only** to implement the existing Architect R35/R40 boundary. Do not broaden to arbitrary Shipment states.

### 5. Physical handover is distinct from close

`CarrierManifest CLOSED → HANDED_OVER` represents physical pickup / own-transport departure.

On a successful handover transaction, for the exact manifest composition:

- manifest becomes `HANDED_OVER`,
- each included Shipment in `IN_MANIFEST` becomes `HANDED_TO_CARRIER` with one `ShipmentHandedToCarrier` transition,
- each included Outbound `TU` in `IN_SHIPMENT` becomes `DISPATCHED` with one `TUDispatched` transition,
- each contributing `OutboundOrder` that is still `READY_FOR_DISPATCH` becomes `DISPATCHED` with one `OutboundOrderDispatched` transition,
- no included physical/business object may regress,
- repeated handover must not duplicate transition/business facts.

Do not mark the manifest `CONFIRMED` automatically in the same action. `HANDED_OVER` and `CONFIRMED` are intentionally distinct.

### 6. Warehouse confirmation is an exactly-once downstream boundary

`CarrierManifest HANDED_OVER → CONFIRMED` is the final warehouse confirmation.

P1-015 must persist/emit an exactly-once durable confirmation boundary sufficient for P1-016 to settle the exact manifest contents later. Use the existing FND-002 transition/event foundation and, where useful, existing orchestration idempotency patterns rather than inventing a parallel generic framework.

The confirmation fact/context must allow deterministic correlation to the manifest and its final frozen Shipment membership. A membership snapshot or equivalent stable relational view is acceptable if it is durable and unambiguous; do not serialize unnecessary secrets or UI-only data.

Duplicate or parallel confirmation must result in:

- one final `CONFIRMED` state,
- one `CarrierManifestConfirmed` transition/business fact,
- no duplicate downstream trigger fact,
- no state regression.

### 7. Hard P1-016 boundary

P1-016, not P1-015, owns the final settlement consumer for:

- `Inventory PICKED → SHIPPED`,
- decrementing `Allocation.reservedQty`,
- `OutboundOrderLine PACKED → SHIPPED`,
- `Allocation CONFIRMED → CONSUMED`,
- `OutboundOrder DISPATCHED → COMPLETED` after all contributing Shipments are confirmed,
- `CustomerOrderLine` / `CustomerOrder` final aggregation,
- final cancellation/settlement boundaries assigned to P1-016.

Therefore P1-015 must **not** add that settlement logic just to make full final-state scenarios pass early.

At P1-015 final head, a confirmation may exist while downstream P1-016 states/quantities remain at their pre-settlement values. Evidence must state this honestly as the authorized delivery decomposition, not as an Architect behavior change.

Do not change the Architect source, requirements, task catalog or traceability to hide this decomposition.

## Persistence guidance

Implement the smallest target model that satisfies the task and existing architecture.

At minimum persist:

- target CarrierManifest identity,
- organization/tenant/warehouse scope needed by current WMS patterns,
- exact lifecycle status,
- durable Shipment membership through one authoritative relation,
- lifecycle timestamps/audit identity where needed by current architecture,
- idempotency/correlation needed to prove exactly-once close/handover/confirm.

Prefer additive/reversible migration.

If using existing `Shipment.manifestId`, add only the integrity constraints/indexes needed by the target model. Do not duplicate membership in an unrelated join table unless the current domain needs it.

Do not add carrier-provider payloads, tracking APIs, external webhooks, route planning, trailer planning or settlement tables owned by P1-016.

Apply migrations only to approved Testing PostgreSQL. No Prod/Demo.

## Authorization and tenancy

Use server-authoritative current authentication/RBAC context. Do not trust request-body role strings to authorize manifest actions.

Architect actor is Dispatcher for open/close/final confirmation and Dispatcher/Carrier boundary for physical handover. Map this to the current real product role/permission model; do not invent a new role string if the product already has an appropriate dispatch capability.

At minimum prove:

- authorized dispatch user can perform required actions,
- ordinary unauthorized warehouse operator cannot close/handover/confirm a manifest,
- cross-organization/tenant Manifest/Shipment access fails closed,
- membership cannot cross tenant/org/warehouse scope where current domain requires warehouse scope consistency.

Supervisor visibility/actions must not silently bypass Dispatcher restrictions unless current accepted RBAC explicitly grants the same capability.

## Concurrency / exactly-once requirements

Schema uniqueness alone is insufficient. Use real PostgreSQL and independent sessions for races that can change composition or lifecycle.

### A. Same Shipment assigned to two manifests

Create two real `OPEN` manifests and one real `POSTED` Shipment.

Run two independent actual manifest-assignment service operations concurrently, each trying to assign that Shipment to a different manifest.

Prove real overlap/database serialization using exact backend PIDs + `pg_stat_activity` / `pg_blocking_pids` if row locking is the mechanism, or equivalent decisive database-side proof for the chosen mechanism.

After both settle, fresh independent reads must prove:

- Shipment has exactly one manifest membership,
- Shipment is `IN_MANIFEST`,
- exactly one `ShipmentAddedToManifest` transition exists,
- exactly one assignment business effect exists,
- the losing operation fails deterministically or returns a safe idempotent result; it must never reassign the Shipment.

### B. Add-versus-close composition race

Use one real `OPEN` manifest and a second eligible `POSTED` Shipment.

Overlap an actual add operation and an actual close operation. Both operations must use a consistent lock order / transaction boundary so the result is serializable and deterministic.

Acceptable final outcomes are only:

1. add commits first and the Shipment is part of the frozen composition before close, or
2. close commits first and add fails without changing membership.

Forbidden outcome: a Shipment becomes newly attached after the manifest is already durably `CLOSED`.

Fresh reads must prove final manifest status, exact membership and transition counts.

### C. Duplicate/parallel confirmation

From one real `HANDED_OVER` manifest, run two independent actual confirm operations with overlap.

Prove:

- one final `CONFIRMED`,
- exactly one `CarrierManifestConfirmed` transition/business fact,
- no duplicate durable downstream trigger,
- no final-state regression.

If locking is used, record exact A↔B backend PID contention tied to those operations, not a global unrelated lock query.

Do not use a synthetic test where operation A manually updates rows while only operation B calls the real service.

## Transaction rollback

P1-015 includes multi-entity handover changes, so include a real PostgreSQL rollback test.

Fail deterministically after meaningful write/flush inside the handover transaction and prove with fresh independent reads that no partial state survives, including at minimum:

- manifest did not partially advance,
- Shipments did not partially become `HANDED_TO_CARRIER`,
- TUs did not partially become `DISPATCHED`,
- OutboundOrders did not partially become `DISPATCHED`,
- no partial transition/event facts remain.

Do not simulate rollback only with mocks.

## Minimum PostgreSQL behavior suite

Final-head Testing PostgreSQL tests must cover at least:

1. authorized Dispatcher creates a target manifest in `OPEN` with `CarrierManifestOpened` once;
2. only `POSTED` Shipment may be added to an `OPEN` manifest;
3. successful add sets one membership, `POSTED → IN_MANIFEST`, one `ShipmentAddedToManifest`;
4. replay of the same Shipment→same manifest assignment is idempotent/non-regressive;
5. assignment of one Shipment to a second manifest fails closed;
6. real concurrent assignment to two manifests is exactly-one with decisive PostgreSQL overlap proof;
7. cross-tenant/org membership fails closed;
8. `OPEN → CLOSED` produces exactly one `CarrierManifestClosed` and freezes composition;
9. reopen/remove/add after `CLOSED` is impossible;
10. real add-versus-close race cannot attach a Shipment after durable close;
11. accepted pre-close Supervisor Carrier correction remains usable at the Architect-valid boundary and does not regenerate/reprint label;
12. Carrier correction after `CLOSED`/later states fails closed;
13. `CLOSED → HANDED_OVER` is the only handover start state;
14. handover transitions included Shipments to `HANDED_TO_CARRIER`, TUs to `DISPATCHED` and contributing `OutboundOrder READY_FOR_DISPATCH → DISPATCHED` exactly once;
15. handover is distinct from confirmation: manifest remains `HANDED_OVER` until explicit confirm;
16. handover rollback leaves no partial multi-entity state/event facts;
17. `HANDED_OVER → CONFIRMED` creates one durable `CarrierManifestConfirmed` boundary/fact;
18. duplicate/parallel confirm is exactly-once with real database proof;
19. non-authorized operator lifecycle actions fail closed under server-authoritative RBAC;
20. P1-016 settlement is not implemented: confirm does not prematurely consume Allocation, ship OOL quantities, complete OutboundOrder or close CustomerOrder.

This inventory is a minimum behavior contract, not permission to fake tests around implementation internals. If final implementation requires additional tests, add them and record the literal final count.

## Regression requirements

At exact final Mercato P1-015 head rerun:

- full P1-015 PostgreSQL suite,
- P1-014 PostgreSQL **18/18**,
- P1-013 PostgreSQL **15/15**,
- P1-012 PostgreSQL **14/14**,
- P1-011 PostgreSQL **18/18**,
- FND-002 state-transition suite **77/77** or exact legitimate additive count if committed transition tests increase it; explain any count change,
- FND-002 transaction simulation **8/8** if still applicable,
- any shared `wms_orchestration`, Inventory/TU or warehouse regressions touched by the P1-015 diff.

Because P1-015 touches TU/OutboundOrder handover and potentially shared transition primitives, run the targeted accepted shared regressions required by current `AGENTS.md`/Testing rules. Do not reduce assertions/counts to make the task pass.

## Minimum rendered Mercato / Playwright evidence

Use the canonical Testing Mercato URL and record the **exact served final Mercato SHA**.

Use normal authenticated product surfaces. Automated evidence is `PLAYWRIGHT VERIFIED`, not `HUMAN VERIFIED`.

At minimum cover:

### Journey A — open manifest and add POSTED Shipment

- authenticate as the real authorized dispatch role/capability,
- open a new manifest,
- add a real `POSTED` Shipment through the visible product action,
- UI shows Manifest `OPEN` and Shipment `IN_MANIFEST`,
- fresh DB read proves exact membership and one transition.

### Journey B — uniqueness and invalid membership boundary

- attempt to add a non-`POSTED` Shipment and show fail-closed behavior,
- attempt to assign an already-manifested Shipment to another manifest and show fail-closed/idempotent behavior,
- persisted membership remains singular.

### Journey C — close and irreversible composition / carrier boundary

- close an `OPEN` manifest through UI,
- UI shows `CLOSED`,
- prove add/remove/reopen action is unavailable or rejected server-side,
- exercise the accepted Supervisor Carrier-correction surface at the relevant boundary and prove post-close correction is blocked,
- prove no automatic label regeneration/reprint is introduced.

### Journey D — physical handover

- from `CLOSED`, perform the visible handover action,
- UI shows manifest `HANDED_OVER`, included Shipment `HANDED_TO_CARRIER`,
- persisted included TU(s) are `DISPATCHED`, contributing OutboundOrder is `DISPATCHED`,
- prove manifest is **not** yet `CONFIRMED`.

### Journey E — final warehouse confirmation / exactly-once UI boundary

- from `HANDED_OVER`, execute final confirmation,
- UI shows `CONFIRMED`,
- repeat/reload/retry the user action where product surface permits and prove non-regression/no duplicate business fact,
- prove P1-016 final settlement is not exposed/claimed as already completed in this task.

### Journey F — unauthorized role

- authenticate as a real non-dispatch warehouse operator,
- prove close/handover/confirm controls are absent or server-side rejected,
- database state remains unchanged.

You may combine B/F or split journeys differently if the final UI makes that clearer. Evidence must record literal final count and what each journey proves.

Do not manipulate final business state only through direct SQL and then call that a rendered journey. SQL fixture setup is allowed; the decisive action under test must go through the normal application surface.

## Evidence artifact

Create/update only:

`WMS_Outbound/05_EVIDENCE/P1-015_EVIDENCE.md`

Do not edit WMS steering/state/handover/traceability/task catalog/workflow from the executor.

Evidence must contain literal final-head facts:

- accepted P1-014 base `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`,
- exact final P1-015 Mercato SHA and branch,
- parent/merge-base/commit count,
- Git-derived compare/file stats,
- exact target CarrierManifest persistence/schema/migration and use of Shipment membership,
- exact lifecycle/status/domain-event implementation,
- exact server-authoritative authorization model used by manifest actions,
- literal final PostgreSQL test inventory/count,
- decisive concurrent same-Shipment assignment proof,
- decisive add-versus-close race proof,
- decisive duplicate/parallel confirm proof with exact operation/session identity,
- exact fresh-read transition/event counts,
- exact handover rollback proof,
- literal regression outputs/counts,
- literal Playwright journeys/count,
- exact served Testing Mercato SHA,
- exact Scanner frozen SHA and branch,
- exact clean `git status --short` for relevant worktrees,
- explicit statement that P1-016 final settlement is intentionally not implemented/claimed,
- explicit exclusions: no external carrier API, no Crossdock GR gate, no Return Receipt, no Prod/Demo, no unrelated Scanner,
- no `FINAL PASS` / `Owner Accepted` claim by executor.

Where Task Catalog acceptance mapping includes `TC-128`, `TC-129`, `TC-132`, `TC-133`, evidence must distinguish the manifest-side prerequisite/boundary proven in P1-015 from the final quantity/order settlement assertions deferred to P1-016. Do not falsely claim the full terminal scenarios are complete early.

Never record secrets.

## Hard exclusions

No:

- P1-016 final Inventory/Allocation/OOL/OutboundOrder/CustomerOrder settlement,
- P1-016 cancellation completion logic,
- Crossdock-specific GR gating,
- Return Receipt / post-dispatch correction process,
- external carrier API/webhook/provider workflow,
- route/trailer/dock planning invented for manifest,
- Scanner changes by default,
- Production/Demo changes,
- unrelated refactors/cleanup,
- branch merge/delete,
- STATE/handover/traceability/task-plan/workflow edits by executor,
- acceptance claim.

## Two-strikes rule

This is the first implementation shot for P1-015.

If supervisor later authorizes one corrective shot, the same material failure class after that corrective attempt means STOP. Do not self-authorize repeated remediation.

## Git / STOP boundary

1. Work only on `outbound/p1-015` from exact accepted base `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`.
2. Push P1-015 implementation/tests.
3. Push `05_EVIDENCE/P1-015_EVIDENCE.md` to `WMS_Outbound/main` only after final-head verification.
4. Report exact 40-char Mercato, WMS evidence and frozen Scanner SHAs plus literal key test counts.
5. STOP.

Do not merge/delete branch. Do not start P1-016. Do not update steering. Do not claim acceptance.

Owner-facing success response should remain microscopic: `done` plus exact refs/test counts only.