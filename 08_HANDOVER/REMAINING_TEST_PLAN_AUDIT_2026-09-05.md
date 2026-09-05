# WMS Outbound — Remaining Test Plan Audit (P2-003 → ACC-004)

**Date:** 2026-09-05  
**Purpose:** durable supervisor audit of the remaining test/evidence plan after P2-002 Owner Acceptance.  
**Authority:** Architect Source remains immutable truth. This audit clarifies staged execution/evidence only; it does not rewrite business requirements or Task Catalog ownership.

## Global corrections for every remaining item

1. **Task Catalog TC IDs are staged traceability.** A task proves only the portion of a mapped scenario supported by that task's own FR/Architect references. A mapped full E2E scenario may span later items and is not permission to implement future scope early.
2. **Exact served runtime is a gate.** Source SHA alone is insufficient. Mercato route/product changes require repository-native full build/generate + canonical service restart + route presence proof. Scanner changes require fresh canonical `scanner-testing.service` export/runtime proof.
3. **Scanner warehouse readiness is explicit.** Browser tests wait for successful real warehouse resolution/selection before module entry.
4. **Current UI contract wins over historical selectors.** Test current `testID`/accessibility/current copy. Historical accepted text is not immutable unless Architect Source requires it.
5. **Fixture prerequisites are explicit.** org/tenant/warehouse/role/zone/policy/TUSetup/Sequence prerequisites are created deterministically and restored. Never depend on ambient Testing cardinality/defaults.
6. **Concurrency evidence proves business effects.** Genuine overlapping PostgreSQL participants are required where concurrency is claimed; incidental observer shapes (`wait_event`, `pg_blocking_pids`, one lock type) are not requirements unless Architect Source says so.
7. **Diagnose by layer.** 404/5xx/API-null → runtime/route/API/persistence before UI selectors. Correct API + wrong render → UI defect.
8. **Regression is relevance-based.** Accepted Inbound/shared suites are mandatory when the current diff actually modifies shared Inventory/TU/warehouse/task-lock/orchestration primitives; do not run broad historical regressions by habit.
9. **Canonical Testing only.** Local PostgreSQL remains forbidden; canonical Supabase Testing project is `yzonugcenguvmojwiihb`.
10. **Human-facing evidence is designed with the feature.** Real Playwright journey + real request evidence + persisted reconciliation must be defined before closeout, not bolted on after backend completion.
11. **Owner Accepted != Human Verified.** Human Verified is used only after a person actually traverses the normal UI. ACC-004 is the final human walkthrough gate.
12. **No evidence relabeling.** Old evidence may be retained only where the relevant semantics/surface and evidence class remain truthful on the final code line.

## Item-by-item audit

### P2-003 — RF crossdock sorting into outbound TUs

**Plan status:** business scope OK; acceptance mapping needs staged interpretation.

Corrections:

- `TC-020/021` are full E2E scenarios and extend into Shipment/dispatch; P2-003 proves only RF/TU sorting slices. P2-006/ACC closes downstream slices.
- `TC-023` shortage/Supervisor is P2-004, not P2-003 implementation.
- residual/finalization exception portions of `TC-027` are P2-004.
- `TC-030` GR correlation is P2-005.
- `TC-035` cancellation/PutBack recovery is P2-004/P4.
- P2-003 must ship real 1:1 + n:n Scanner Playwright in the same item, with lazy target creation and physical-full continuation.
- Relevant dependency regression: P2-002 22/22 and P1-008 TU identity/numbering; shared Inbound only if shared code is modified.

### P2-004 — Crossdock shortage, damage, empty-TU and cancellation recovery

**Plan status:** business scope OK; staged TC boundaries must be explicit.

Corrections:

- P2-004 owns shortage recheck/confirmation, DAMAGED/QC, unexpected SKU, empty source TU, allowPartial decisions, in-progress cancellation and source residual/finalization exception behavior.
- When P2-004 creates a `PutBackTask` as required by Architect Source, it proves task creation/recovery handoff only; PutBack assignment/location/inventory completion remains P4.
- `TC-021` normal n:n mechanics are regression from P2-003, not new implementation.
- `TC-035` physical PutBack completion must not be pulled forward.
- `TC-067` is a composite exception scenario including GR_REJECTED; P2-004 proves only shortage/empty/cancellation slices. GR slice is P2-005.
- Quantity evidence must reconcile declared = confirmed + damaged + residual with unexpected SKU handled separately as Architect Source states.

### P2-005 — Goods Receipt correlation gate and re-evaluation

**Plan status:** OK with runtime/integration guardrails.

Corrections:

- Prove INT-02/INT-03 source correlation and GR source classification before UI assertions.
- `GR_REJECTED` leaves gate unsatisfied but does not itself force Shipment `POSTING_ERROR`.
- Re-evaluation on later GR message must be idempotent and source-specific; PUTAWAY GR must not mutate CROSSDOCK acceptance state.
- Full ERP/Shipment lifecycle after gate satisfaction remains P2-006; P2-005 proves gate behavior and visibility only.
- Any new integration/API route must pass exact generated-route/runtime preflight before Playwright.

### P2-006 — Crossdock join into common Shipment/dispatch downstream

**Plan status:** OK; this is where full downstream slices are finally allowed.

Corrections:

- Reuse the accepted P1 Shipment/carrier/ERP/manifest/settlement pipeline; do not create a Crossdock-specific logistics lifecycle.
- Mixed STANDARD+CROSSDOCK completeness and allowPartial=false guard must be tested on the same canonical pipeline.
- Downstream slices of `TC-020/021/028/030/031` may close here only after P2-005 gate behavior is satisfied.
- Regression should target the P1 downstream services actually reused/modified, with final exact-runtime Playwright.

### P3-001 — Reservation Release before formal pick

**Plan status:** OK.

Corrections:

- Authoritative selector is formal `pickedQty = 0` / no confirmed pick.
- Exactly-once Allocation release and Inventory `RESERVED → AVAILABLE` are the core integration proof.
- No PutBackTask for true pre-pick release.
- If physical movement happened before formal confirmation, only exact-source return instruction is allowed; full race execution is P3-003.

### P3-002 — Reservation retention policy and automatic release timer

**Plan status:** OK; test mechanics need deterministic time.

Corrections:

- Test configured due timestamps/clock deterministically; do not use long real sleeps as decisive proof.
- Cover all three warehouse policy variants explicitly.
- Prove configured retention time is independent of priority/SLA.
- Repeated scheduler execution must be idempotent/exactly-once.
- Fixture must explicitly set and restore warehouse policy.

### P3-003 — Cancellation race: physical movement before formal confirmation

**Plan status:** OK; concurrency proof must remain architecture-first.

Corrections:

- Real overlapping operations are required for the race; do not require a particular PostgreSQL observer diagnostic shape.
- `pickedQty = 0` race path: cancel PickTask and render exact source-location return instruction; no PutBackTask.
- `pickedQty > 0`: route to P4; do not execute P4 physical recovery in this item.
- Scanner UI must prove the recovery instruction through current UI contract.

### P4-001 — Post-pick/post-pack cancellation approval and logical effects

**Plan status:** OK; full TC-050 is staged.

Corrections:

- This item owns approval/boundary + immediate logical effects only.
- `PACKED` cancellation requires Supervisor and is blocked at `Shipment POSTING_PENDING`/later architect boundary.
- OOL cancellation / Allocation release happen immediately and do not wait for PutBack completion.
- `TC-050` end-to-end inventory return is not fully closed here; task/lifecycle/physical completion belong P4-002/P4-003.

### P4-002 — PutBackTask model, FIFO assignment and lifecycle

**Plan status:** OK; staged UI boundary required.

Corrections:

- Create exactly one PutBackTask for eligible `pickedQty > 0` cancellation.
- FIFO assignment, no zone selector, one-active-warehouse-task guard and task states are this item.
- No location-validation/inventory completion implementation yet; that is P4-003.
- Scanner assignment tests use explicit warehouse fixture + real warehouse readiness, not historical text or ambient single-warehouse assumptions.

### P4-003 — RF PutBack location validation loop and Inventory recovery

**Plan status:** OK.

Corrections:

- Real Scanner invalid → invalid → valid journey; no retry limit and no automatic escalation.
- Invalid location must commit no Inventory movement/completion.
- Valid completion moves exact quantity `PICKED → AVAILABLE` once.
- Persisted quantity/location/task state must reconcile after rendered action.
- This is where the physical completion slices of TC-050/051/052 close.

### X-001 — CON-01..05 concurrency / exactly-once

**Plan status:** conceptually OK; evidence design needs tightening.

Corrections:

- Build an explicit CON-01..05 outcome matrix against the final implementation line.
- Each proof states the business invariant that would fail without the guard.
- Do not infer correctness from lock implementation or diagnostic shape alone.
- Earlier P1/P2 concurrency evidence can be referenced for lineage, but final X-001 executable proof must still be valid against the final relevant code semantics; rerun changed surfaces.

### X-002 — Integration correlation, observability and operational recovery

**Plan status:** OK with route/runtime and ownership discipline.

Corrections:

- INT-01..06 correlation identities must be explicit and testable without exposing secrets/raw unnecessary payloads.
- Use real application adapters/contracts; deterministic test stubs/endpoints may stand in for external systems only where the contract allows.
- Inbound keeps GR retry ownership; Outbound must not absorb it.
- New diagnostic/integration routes require generated-route/runtime preflight.

### ACC-001 — Complete automated requirement suite

**Plan status:** requires a stronger coverage-verification rule.

Corrections:

- 109/109 automated coverage must be validated by explicit requirement IDs/metadata, not inferred from historical test names or accidental TC labels.
- This avoids repeats of the `CON-03/TC-134` naming error.
- Staged TC mappings are resolved at requirement-slice level; a TC name alone cannot claim coverage of all its requirements.
- Freeze final Mercato/Scanner/migration/runtime identities and rerun suites invalidated by later code changes.
- Local PostgreSQL is never accepted evidence.

### ACC-002 — Final P1 normal-UI journeys

**Plan status:** OK; historical UI drift must be neutralized.

Corrections:

- Run final journeys on final exact revisions/current UI, not old accepted selectors/spec assumptions.
- Historical P1 Playwright may be reused only after confirming its fixtures and current UI contract remain valid.
- Real warehouse readiness and exact runtime/build provenance are mandatory.
- API/DB only prepare/verify; decisive user actions remain rendered UI.

### ACC-003 — Final P2/P3/P4 normal-UI journeys

**Plan status:** OK; must be a final rerun, not evidence aggregation only.

Corrections:

- Execute final journeys 15–20 on exact final Mercato/Scanner/runtime revisions.
- Do not substitute a collection of old item-level Playwright passes for the final end-to-end journey.
- Include 1:1+n:n Crossdock, exception paths, GR gate re-evaluation, Reservation Release and invalid-location PutBack loop.
- Preserve real warehouse/runtime preflight and zero mock substitution of decisive UI actions.

### ACC-004 — Final Human Verified walkthrough

**Plan status:** OK; label discipline required.

Corrections:

- Human Verified requires an actual person traversing the normal UI on the frozen final revisions/environment.
- Earlier Owner acceptance of automated evidence does not become Human Verified retroactively.
- No manual DB repair is allowed to make the walkthrough pass.
- Record exact Mercato SHA, Scanner SHA, runtime/environment, actor, date, journey and defects.

## Conclusion

No remaining business requirement requires architectural redesign. The main plan correction is **execution semantics**:

- mapped TC IDs are staged slices;
- runtime identity is a first-class acceptance gate;
- current UI/explicit fixtures replace historical assumptions;
- concurrency proof targets business invariants;
- final ACC work reruns exact final journeys rather than relabeling old evidence.

P2-003 may proceed under `06_AGENT_GUIDES/P2-003_EXECUTION.md`.