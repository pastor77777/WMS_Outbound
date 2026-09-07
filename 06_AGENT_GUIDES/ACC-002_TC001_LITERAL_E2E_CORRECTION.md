# ACC-002 — TC-001 literal Standard Fulfillment E2E correction

**Status:** current corrective execution guide  
**Effective:** 2026-09-07  
**Scope:** ACC-002 only — close the missing literal `TC-001` Standard Fulfillment end-to-end Playwright proof  
**Preserve:** ACC-001 Supervisor FINAL PASS; existing ACC-003 implementation/evidence checkpoint unless this correction changes product behavior  
**Mercato current batch head:** `outbound/acc-001-003-batch` @ `bc80c989eea2f6b468530a9bd6cd814c8dc35646`  
**Scanner current batch head:** `outbound/acc-001-003-batch` @ `f9a98dd8a03078dd9a0e45667927861ab6f79f7b`  
**Hard stop:** do not start ACC-004

## Why this correction exists

Supervisor review found one real ACC-002 acceptance gap:

`TC-001 / Evidence Standard journey 1 — Standard full fulfillment end to end`

The current `ACC-002_EVIDENCE.md` represents Journey 1 as a composite citation across separate P1 stage tests. Executor discovery then confirmed there is no single literal test that carries one Standard Fulfillment order continuously from intake through final settlement. The separate tests seed independent fixtures, so they do not prove one continuous business identity through the complete chain.

This is an acceptance-test completeness gap, not currently a demonstrated product defect.

## Fixed design decision — do not redesign this

Implement the missing journey as **one Mercato-repository Playwright orchestrator spec controlling both real applications**.

Create:

`apps/mercato/src/modules/wms_outbound/__integration__/TC-001-standard-fulfillment-e2e.spec.ts`

Use the existing Mercato Playwright runner/config. A Playwright test is not restricted to its configured `baseURL`; the same test may open a second page/context and navigate to the canonical Scanner URL explicitly. Therefore no new cross-repo runner, IPC protocol, shared orchestration framework, or separate coordinating process is needed.

Use:

- Mercato surface: canonical Testing Mercato (`http://localhost:3009` / configured Testing base URL);
- Scanner surface: canonical Testing Scanner (`http://localhost:8081` / canonical Testing Scanner URL);
- one Playwright test process;
- one continuous order/warehouse identity ledger across both surfaces.

Do not create an abstract general-purpose E2E framework for this correction.

## Identity contract

One business thread must be preserved from beginning to end. Record and assert, as they become available:

- organization / tenant;
- warehouse id/code;
- CustomerOrder id + order number/external id;
- CustomerOrderLine id;
- OutboundOrder / OutboundOrderLine id;
- allocation/reservation identity where applicable;
- PickTask / PickTaskLine id;
- Picking/Shipping TU id(s);
- Shipment id;
- Carrier selection / label correlation needed to prove the selected shipment;
- ERP posting correlation/attempt identity;
- CarrierManifest id;
- final Shipment / order / quantity state.

A later stage must consume state created by the earlier stage. **Do not reseed a new order, shipment, TU, manifest or substitute business chain mid-test.**

Multiple real roles may legitimately participate in the full fulfillment journey. Keep the same business order + warehouse and explicitly record every actor/role transition instead of pretending one role owns the entire warehouse process.

## Fixture boundary

Direct DB/API/test seams may create only deterministic prerequisites that are not the decisive accepted business action, for example:

- test users/roles;
- warehouse/masterdata;
- starting Inventory/stock;
- external adapter response/stub state where the Architect contract has no human UI.

They may also read authoritative state for assertions.

They must **not** manufacture completed intermediate business stages merely to jump forward in TC-001.

For each human-facing stage, use the normal rendered Mercato or Scanner UI and the same accepted selectors/actions already proven by the existing P1 Playwright specs.

## Reuse strategy

Do not invent new product behavior or duplicate large fixture frameworks.

Read and reuse the smallest proven setup/action/assertion patterns from the existing stage specs, especially the currently green ACC-002 sources for:

- P1-001 — intake/order visibility;
- P1-002/P1-003 — ATP/planning;
- Scanner P1-005/P1-006 — assignment/picking;
- P1-009/P1-010 — direct-pack or pack workstation as appropriate for the chosen simple Standard Fulfillment fixture;
- P1-011 — Shipment grouping/readiness;
- P1-012 — carrier selection;
- P1-013 — WMS label;
- P1-014 — ERP posting;
- P1-015 — manifest close/handover boundaries;
- P1-016 — final settlement.

Local test helpers may be extracted/refactored when that materially reduces duplication, but keep the correction narrow. Do not alter product code unless the literal flow exposes a genuine architect-authorized product defect.

## Required literal flow

Use the smallest deterministic Standard Fulfillment shape: one order, one line, full ATP, no shortage, no crossdock, no exception branch.

The one test must traverse the accepted chain continuously:

`Customer Order/intake`
→ `ATP/reservation/planning`
→ `task assignment`
→ `Scanner picking`
→ `packing/direct-pack completion appropriate to the fixture`
→ `Shipment grouping/readiness`
→ `carrier selection`
→ `WMS label generation/print action`
→ `ERP posting`
→ `manifest close/handover/confirmation boundary`
→ `final settlement`.

Where an architecturally external system response has no normal human UI, use the already accepted deterministic adapter/test seam, but keep the same correlation identity and return to the normal UI for the next human action. Do not replace a human-facing action with direct API/DB mutation.

## Evidence Standard fields

For TC-001 the test/evidence must explicitly record:

- actor/role for every role transition;
- warehouse;
- stable order/task/Shipment/TU/manifest identifiers;
- visible expected outcome at each human stage;
- authoritative persisted/server outcome after that stage;
- next reachable user action.

Emit one sanitized final identity summary in the test output (IDs/statuses only, no credentials/secrets) so the continuous chain is auditable from one run.

## Existing ACC-002 journeys 2–14

Do not rerun or rewrite green journeys merely for ceremony.

Update `05_EVIDENCE/ACC-002_EVIDENCE.md` so journeys 2–14 explicitly contain the Evidence Standard fields that were previously summarized too loosely: actor/role, warehouse, stable identifiers available from the owning spec, visible outcome, persisted/server outcome, and next reachable action.

These details may be grounded from the already-passing exact spec bodies/artifacts. Rerun a journey only if its required evidence cannot be established from the existing final-checkpoint proof or if this correction changes code/helper behavior that can affect it.

## Execution / test scope

This is a continuation/correction inside ACC-002, not a new Task Catalog item. Do not invent another mandatory deep reset solely for this correction.

Before execution, fast-forward current steering repos and verify the exact batch heads above.

Minimum required final proof when only a new test spec/evidence changes:

1. new literal TC-001 spec PASS on canonical Testing;
2. rerun TC-001 at least once from a clean deterministic fixture lifecycle to prove repeatability;
3. Mercato typecheck clean if TypeScript test code changed;
4. no product-code diff unless a genuine product defect was found;
5. ACC-002 evidence updated with the literal TC-001 chain and required per-journey fields;
6. relevant Mercato branch/evidence pushed;
7. Scanner branch remains at `f9a98dd8...` unless a real Scanner test/product change is actually required.

If shared helpers used by other ACC-002 specs are modified, rerun the directly affected specs. If product behavior is modified, rerun all impacted ACC-001/ACC-002 evidence and any ACC-003 journey whose behavior is touched.

## Completion contract

Return `COMPLETE` only when:

- one literal continuous TC-001 Standard Fulfillment Playwright journey exists and passes;
- one order/warehouse business identity is proven continuously through final settlement;
- no intermediate business stage is replaced by a freshly seeded unrelated fixture;
- required Evidence Standard fields are explicit for TC-001 and journeys 2–14 in `ACC-002_EVIDENCE.md`;
- required final proof is green and pushed;
- ACC-001 remains preserved;
- ACC-003 existing checkpoint is preserved or explicitly reverified if genuinely impacted;
- ACC-004 has not started.

Then STOP for Supervisor verification. Do not claim Supervisor FINAL PASS, Owner Acceptance or HUMAN VERIFIED.
