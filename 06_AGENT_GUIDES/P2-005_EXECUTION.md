# P2-005 — Goods Receipt correlation gate and re-evaluation

## Status and authority

This is the authoritative executor guide for **P2-005 — Goods Receipt correlation gate and re-evaluation — item 24/37**.

P2-004 is **FINAL PASS / Owner Accepted**. Preserve the accepted P2-004 product boundary exactly.

Frozen accepted bases for this item:

- Mercato: `9859be5c7dee4fe802d4d00478459a19982eddfe`
- Scanner: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- P2-004 evidence: `9fb9abd33c1ff8318b6339efc9b69cce3a3161ac`

Authority order remains:

1. immutable Architect Source / Canon;
2. traceability and Task Catalog;
3. current accepted product/DB truth as implementation evidence;
4. this execution guide as the bounded delivery contract.

Executor never self-accepts. `FINAL PASS`, `Owner Accepted`, and `Human Verified` are supervisor/Owner states only.

## Mandatory new-item Testing reset

P2-005 is a **new Task Catalog item**. Before any P2-005 implementation action, run the canonical deep Testing reset:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

Record `RESET_OK` or the exact blocker.

This reset is runtime/worktree hygiene only. It must **not**:

- downgrade or roll back canonical Supabase schema;
- delete accepted P2-003/P2-004 schema;
- reconstruct product repositories from older P2 bases;
- rewrite accepted product history.

After the reset, verify product heads/branches. Create/switch the intended `outbound/p2-005` branch from the exact accepted P2-004 head in each product repository that actually changes. If a branch already exists, verify ancestry before using it. Do not rebase accepted history.

## Task Catalog objective

Correlate Goods Receipt results by source Inbound TU, persist the cross-dock GR acceptance status for every contributing source, gate ERP Shipment posting on that status, and re-evaluate the gate on later GR messages.

Task Catalog references:

- P2 R25–R26;
- P2 R31–R33;
- P2 R35–R36;
- P2 KROK 3 / KROK 4;
- `FR-P2-13`, `FR-P2-14`, `FR-P2-17`, `FR-P2-18`, `FR-P5-16`, `INT-02`, `INT-03`;
- acceptance scenarios `TC-028`, `TC-030`, `TC-031`, `TC-038`, `TC-067`.

Dependencies already accepted:

- P2-004 — crossdock execution/recovery/finalization;
- P1-014 — ERP Shipment POST / POSTING_ERROR / safe Supervisor retry.

## Architect invariants — implement literally

### 1. Correlation identity is source TU + settlement source

A GR result for cross-docking is correlated **only** by:

- `sourceInboundTU` (the source elementary Inbound TU identifier), and
- `GR_SETTLEMENT_SOURCE = CROSSDOCK`.

Do **not** require or trust a CrossDockPickTask id in the GR message.

Do **not** use settlement/message version number to infer CROSSDOCK vs PUTAWAY. Versioning is independent inside each settlement source.

### 2. One source result updates all tasks for that source

When a valid CROSSDOCK GR result matches a source Inbound TU, apply the same `grAcceptanceStatus` (`GR_ACCEPTED` or `GR_REJECTED`) to **all** `CrossDockPickTask` rows in the same org/tenant whose `sourceInboundTU` is that TU, regardless of which Shipment each task eventually contributes to.

The accepted task model currently has no `grAcceptanceStatus`; P2-005 may add the minimum additive field/index/migration required by Architect truth.

Initial status for a task/source that has not received a valid result must be explicit and non-accepted (normally `GR_PENDING` if repository conventions allow). Do not treat NULL/absence as accepted.

### 3. Wrong source or wrong settlement source causes zero mutation

Reject/ignore safely with an explicit outcome when:

- `sourceInboundTU` matches no CrossDockPickTask in the scoped org/tenant; or
- `GR_SETTLEMENT_SOURCE != CROSSDOCK` (including PUTAWAY).

Such a message must change **zero** crossdock task GR statuses and must not change Shipment posting state.

### 4. Shipment gate is per Shipment, not global

For each Shipment, derive its own complete set of source Inbound TUs that contributed **confirmed crossdock quantity** to that Shipment.

The gate is satisfied only when **every distinct contributing source Inbound TU for that Shipment** has crossdock `GR_ACCEPTED`.

Do not let acceptance of source X globally unlock a Shipment that also contains source Y.

The same source X may contribute to two different Shipments. One GR result for X updates all X tasks; each Shipment then evaluates its own full source set independently.

### 5. Do not implement P2-006 Shipment formation

P2-006 owns automatic joining of crossdock lines/TUs into the common Shipment/dispatch pipeline.

P2-005 may inspect the **already persisted** Shipment/TU/content/OutboundOrderLine/CrossDockPickTask lineage needed to determine contributing source TUs. For deterministic tests, fixtures may establish legitimate persisted Shipment membership directly using accepted P1 shipment/TU structures.

Do **not** broaden P2-005 into:

- automatic crossdock Shipment creation/grouping;
- carrier selection changes;
- manifest behavior;
- dispatch/release behavior;
- mixed-channel completion logic owned by P2-006.

Prefer deriving contribution lineage from existing persisted relationships (Shipment -> Outbound TU -> TU content / OutboundOrderLine -> CrossDockPickTask -> `sourceInboundTU`) rather than inventing a second ownership model, unless current repository truth proves a smaller canonical relation is necessary.

### 6. Only crossdock-settled quantity participates in this GR gate

Residual quantity passed back toward Inbound Putaway is **not** part of the P2-005 Shipment gate.

A later PUTAWAY GR result for the same physical source TU must not regress, replace, or reinterpret the already stored CROSSDOCK `grAcceptanceStatus` and must not make an already satisfied Shipment gate unsatisfied.

### 7. Gate P1-014 before ERP posting side effects

Integrate the gate with the accepted P1-014 `shipment-posting-service` at the **precondition boundary before Phase 1 posting side effects**.

For a Shipment with crossdock contribution where the gate is unsatisfied:

- Shipment must remain in its eligible pre-post state (`LABEL_GENERATED` or `OWN_TRANSPORT`) when an initial post is attempted;
- it must **not** transition to `POSTING_PENDING`;
- create zero `wms_outbound_shipment_postings` rows;
- create zero posting-attempt rows;
- create zero `SHIPMENT_POSTING` orchestration facts/retry attempts;
- invoke the ERP posting adapter zero times;
- return a clear gate-blocked result/error that can be rendered to operations.

Standard P1-only Shipments with no crossdock contribution must preserve accepted P1-014 behavior unchanged.

### 8. GR_REJECTED is an unsatisfied precondition, not ERP posting failure

A valid CROSSDOCK `GR_REJECTED` updates source/task GR status to rejected and re-evaluates affected Shipment gates, but **must not itself** set any Shipment to `POSTING_ERROR`.

The gate simply remains unsatisfied until a later valid `GR_ACCEPTED` for that same source.

`GR_REJECTED -> later GR_ACCEPTED` is normal re-evaluation. It requires no manual data repair, no special recovery object, and no reset of Shipment data.

### 9. Re-evaluate on every valid CROSSDOCK GR result

Every valid `GR_ACCEPTED` or `GR_REJECTED` for a relevant source causes re-evaluation of all affected Shipment gates.

The implementation may expose/persist an observable gate assessment if useful, but do not invent a new Architect state machine state or escalation object.

### 10. Preserve P1-014 POSTING_ERROR and retry semantics

If a Shipment is already `POSTING_ERROR` for an actual ERP posting error unrelated to GR, later GR messages still re-evaluate the GR gate.

However P2-005 must **not** silently auto-retry ERP posting or rewrite accepted P1-014 semantics.

Expected behavior:

- `GR_REJECTED`: gate becomes/remains unsatisfied; Shipment remains `POSTING_ERROR`;
- later `GR_ACCEPTED`: gate becomes satisfied if all required sources are accepted; Shipment remains `POSTING_ERROR` until the accepted P1-014 Warehouse Supervisor retry action is used;
- that existing retry must then be allowed to proceed without manual data repair if no other blocker remains.

### 11. Warehouse Supervisor visibility only

Expose the blocking contributing source TUs and their `grAcceptanceStatus` on an existing appropriate Mercato Supervisor/Shipment surface.

Minimum useful visibility:

- source Inbound TU identifier;
- `GR_PENDING` / `GR_ACCEPTED` / `GR_REJECTED` status;
- whether the Shipment GR gate is satisfied;
- which sources still block it.

Do not create a new business state, escalation workflow, or manual GR override. The Supervisor observes the gate; the GR result remains integration truth.

### 12. Inbound owns GR retry

`INT-02` / `INT-03` boundary must remain intact:

- Outbound provides/retains the crossdock finalization correlation and the confirmed/damaged quantities needed by the accepted Inbound settlement boundary;
- Outbound consumes the resulting GR outcome for gate purposes;
- retrying the Goods Receipt message itself remains **Inbound responsibility**;
- do not move Inbound retry/alert ownership into Outbound.

If current accepted shared orchestration/Inbound code already provides an ingress/result pattern, reuse it. Do not invent duplicate retry infrastructure.

## Expected implementation shape

Use current repository truth. The exact file names below are guidance, not permission to ignore a smaller repository-native design.

Likely Mercato changes are bounded to:

- additive `CrossDockPickTask.grAcceptanceStatus` persistence/migration;
- a P2-005 GR-result correlation service / repository-aligned ingress;
- a Shipment crossdock-GR gate evaluator;
- a narrow precondition hook in accepted P1-014 shipment posting/retry flow;
- Supervisor shipment/status read surface;
- dedicated PostgreSQL integration coverage and real Mercato Playwright.

Scanner impact is **none directly** by Task Catalog. Keep Scanner frozen at accepted P2-004 SHA unless a real P2-005 defect proves a Scanner change is necessary. Do not add a Scanner GR decision flow.

## Canonical Testing and integration boundary

Canonical Testing database remains Supabase project `yzonugcenguvmojwiihb` / PostgreSQL 17. Local PostgreSQL is forbidden.

Use deterministic fixtures against canonical Testing PostgreSQL. Never write secrets to evidence.

A deterministic Testing integration seam for receiving a GR result is acceptable if it is repository-aligned and clearly identified as Testing/contract execution. Do not claim real external ERP connectivity if no real ERP endpoint exists.

Do not modify Demo/Prod.

## Required substantive PostgreSQL proof

Create a dedicated suite, preferably:

`p2-005-crossdock-gr-gate-postgres.integration.test.ts`

Test-case count may be lower than the list below only if the evidence maps all behaviors to substantive assertions. **All 18 behaviors below must be explicitly covered.**

1. New/unresolved crossdock source is non-accepted (`GR_PENDING` or repository-equivalent), never implicitly accepted.
2. Valid `CROSSDOCK + GR_ACCEPTED` correlates by `sourceInboundTU` without task id and updates every task of that source.
3. Valid `CROSSDOCK + GR_REJECTED` updates every task of that source and does not create Shipment `POSTING_ERROR`.
4. Later `GR_ACCEPTED` after `GR_REJECTED` replaces the gate status normally and requires no repair path.
5. Duplicate/replayed identical GR result is idempotent and creates no duplicate durable effect/fact.
6. Unknown `sourceInboundTU` is rejected with zero task/Shipment mutation.
7. `GR_SETTLEMENT_SOURCE = PUTAWAY` for the same TU is rejected/ignored with zero crossdock-status mutation.
8. Settlement/message version cannot be used as a substitute for `GR_SETTLEMENT_SOURCE`.
9. One Shipment with two contributing sources remains blocked when only one is `GR_ACCEPTED`.
10. One Shipment with a `GR_REJECTED` contributing source remains outside `POSTING_PENDING` and creates zero P1-014 posting side effects.
11. The same Shipment becomes gate-ready when all distinct contributing sources are `GR_ACCEPTED`.
12. One source TU contributing to two Shipments is updated by one GR result, while each Shipment evaluates its own complete source set separately.
13. Residual/PUTAWAY settlement does not block or regress a gate already satisfied by CROSSDOCK acceptance.
14. P1-only Shipment with no crossdock contribution retains accepted P1-014 posting behavior.
15. Gate rejection occurs before P1-014 Phase 1 side effects: zero posting row, attempt, outbox/retry fact, state transition, and ERP adapter call.
16. Shipment already `POSTING_ERROR` for a real ERP rejection re-evaluates on later GR status changes without auto-retry; once gate-ready, existing P1-014 Supervisor retry can proceed.
17. Org/tenant isolation: a GR result in one scope cannot update tasks or gate state in another scope even with matching/colliding identifiers.
18. Supervisor read model reports exact contributing source statuses and blocking set without changing business state.

Also verify quantity/finalization correlation is not silently changed: P2-004 confirmed/damaged/residual facts remain exact.

## Mandatory regressions

Because P2-005 touches CrossDockPickTask persistence and P1-014 posting preconditions, the minimum mandatory regressions are:

- P2-005 dedicated suite — all mapped behaviors green;
- P1-014 ERP posting PostgreSQL suite — accepted **18/18** behavior remains green, updated only where the new crossdock precondition legitimately adds coverage;
- P2-004 crossdock recovery PostgreSQL — accepted **16/16** remains green;
- P2-003 crossdock execution PostgreSQL — accepted **8/8** remains green;
- repository-native Mercato generate/build/typecheck/build-app contract.

Run P2-002/P2-001 or broader P1 suites only if their surfaces are actually touched.

If shared `wms_orchestration`, Inbound GR, Inventory, TU, warehouse, or lock primitives are modified, run the directly relevant accepted Inbound/shared regressions. Inbound remains CLOSED/REFERENCE; do not reinterpret its behavior.

## Real rendered Playwright proof — zero route mocks

Use canonical running services and **zero Playwright route mocks/interception/substitution of product behavior**.

A dedicated Mercato spec is expected, preferably:

`P2-005-crossdock-gr-gate-ui.spec.ts`

At minimum prove these real journeys:

### Journey A — two-source gate blocks then unblocks

1. Deterministically prepare a legitimate Shipment in accepted P1-014 initial posting state (`LABEL_GENERATED` or `OWN_TRANSPORT`) with persisted crossdock contribution from two distinct source Inbound TUs.
2. Source A = `GR_ACCEPTED`; source B = pending/rejected.
3. Warehouse Supervisor opens the real Mercato Shipment/Supervisor surface and sees both source IDs/statuses and the blocked gate.
4. Through the real rendered posting action, attempt ERP posting.
5. Live backend rejects it at the GR precondition; Shipment remains in its pre-post state; persisted reconciliation shows zero P1-014 posting/attempt/outbox side effects and zero ERP adapter call.
6. Deliver a valid CROSSDOCK `GR_ACCEPTED` result for source B through the real repository-aligned integration ingress/contract (external-system message delivery is not a user UI action and may be driven directly by the test).
7. Refresh real Mercato UI; gate shows ready/all accepted.
8. Use the real rendered posting action again; accepted P1-014 flow proceeds normally using its Testing ERP adapter contract.
9. Persisted DB reconciliation proves exact source statuses, one posting intent/attempt, and non-duplicated transitions.

### Journey B — POSTING_ERROR re-evaluation without auto-retry

1. Prepare a Shipment with all required crossdock sources initially accepted and make a genuine P1-014 ERP posting attempt return the accepted deterministic **ERP rejection** path, producing `POSTING_ERROR`.
2. Deliver a later valid CROSSDOCK `GR_REJECTED` for one required source.
3. Prove Shipment stays `POSTING_ERROR`, gate is unsatisfied, no automatic retry/extra posting attempt occurs, and Supervisor can see the rejected source.
4. Deliver a later valid CROSSDOCK `GR_ACCEPTED` for the same source.
5. Prove gate re-evaluates to ready while Shipment still remains `POSTING_ERROR` and no automatic retry occurs.
6. Use the existing rendered P1-014 Warehouse Supervisor retry action; it proceeds normally and preserves append-only attempt history.
7. Persisted DB reconciliation proves no duplicate/contradictory posting facts.

### Journey C — wrong settlement source does nothing

Using the same real application/integration surface, deliver a PUTAWAY GR result for a source with accepted CROSSDOCK status and prove:

- crossdock `grAcceptanceStatus` stays unchanged;
- Shipment gate stays unchanged;
- no posting state/fact is created or regressed.

If Journey C can be substantively included in A/B without hiding assertions, a separate Playwright test case is not mandatory.

## Runtime contract

After final Mercato code changes:

- run repository-native generation/build contract;
- prove non-empty production manifests;
- restart canonical `mercato-localhost.service` from the exact final Mercato candidate;
- prove service active/running, MainPID/child owns port 3009, `/login` HTTP 200;
- prove generated route/runtime provenance for any new P2-005 ingress/read routes.

If Scanner source remains unchanged, do **not** create noise by rebuilding/changing Scanner. Verify the frozen accepted Scanner SHA/worktree remains clean. If Scanner is changed for a proven reason, a fresh canonical `scanner-testing.service` export/restart and relevant real regression become mandatory.

Do not let Playwright `webServer` / `reuseExistingServer` mask stale canonical runtime.

## Scope exclusions

P2-005 must not implement:

- P2-006 automatic crossdock Shipment/dispatch joining;
- new Carrier Selection behavior;
- ERP payload redesign unrelated to the GR precondition;
- CarrierManifest or issue/dispatch changes;
- P3 Reservation Release;
- P4 Physical PutBack beyond already accepted P2-004 handoff facts;
- Return Receipt;
- Demo/Prod;
- local PostgreSQL;
- Inbound GR retry ownership.

No schema downgrade or destructive reset.

## Push and evidence

When all required gates are green:

1. Ensure Mercato worktree is clean.
2. If Scanner is unchanged, prove it remains exactly accepted P2-004 SHA `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` and clean; do not manufacture a Scanner commit.
3. Push exact tested Mercato final to `outbound/p2-005`.
4. If Scanner genuinely changed, push exact tested Scanner final to `outbound/p2-005`; otherwise report it as frozen accepted SHA.
5. Create `05_EVIDENCE/P2-005_EVIDENCE.md` recording:
   - exact final Mercato SHA;
   - exact Scanner frozen/final SHA;
   - migration/schema provenance;
   - dedicated result/count;
   - explicit **18/18 substantive behavior mapping**;
   - P1-014 / P2-004 / P2-003 regression results;
   - exact GR correlation contract (`sourceInboundTU` + `GR_SETTLEMENT_SOURCE=CROSSDOCK`);
   - wrong-source / PUTAWAY no-side-effect proof;
   - per-Shipment aggregation proof;
   - no-posting-side-effects while gate blocked;
   - POSTING_ERROR re-evaluation and preserved P1-014 Supervisor retry semantics;
   - Supervisor UI visibility;
   - zero-route-mock statement;
   - canonical runtime provenance;
   - clean/dirty worktree status.
6. Push WMS evidence to `main` and report its SHA.
7. Evidence must not self-declare `FINAL PASS`, `Owner Accepted`, or `Human Verified`.

## Final executor report only

Report exactly:

- final pushed Mercato SHA;
- Scanner final/frozen SHA and whether Scanner changed;
- P2-005 dedicated result/count and 18/18 mapping status;
- P1-014 regression result;
- P2-004 regression result;
- P2-003 regression result;
- GR correlation result (`sourceInboundTU` + CROSSDOCK source);
- unknown-source / PUTAWAY negative results;
- per-Shipment gate aggregation result;
- blocked-post zero-side-effect result;
- `GR_REJECTED -> GR_ACCEPTED` re-evaluation result;
- POSTING_ERROR/no-auto-retry + Supervisor retry result;
- Supervisor rendered visibility result;
- zero-route-mocks confirmation;
- Mercato runtime proof;
- WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty;
- if STOPped: one exact blocker.

Then STOP for supervisor verification. Do not start P2-006.