# P2-006 — Crossdock join into common Shipment/dispatch downstream

## Status and authority

This is the authoritative executor guide for **P2-006 — Crossdock join into common Shipment/dispatch downstream — item 25/37**.

P2-005 is **FINAL PASS / Owner Accepted**. Preserve the accepted P2-005 product boundary exactly.

Frozen accepted bases for this item:

- Mercato: `069f02d4c5c9b345b688b838eb685be02206afbd`
- Scanner: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` (frozen unless a real P2-006 defect proves Scanner work is necessary)
- P2-005 evidence: `0c7cf142e1723ff80e86cfd0f00d4b12c1e4b777`
- P2-005 supervisor evidence correction: `cf399679360d8b7fc071f9f958709c3bb99b7c59`

Authority order remains:

1. immutable Architect Source / Canon;
2. traceability and Task Catalog;
3. current accepted product/DB truth as implementation evidence;
4. this execution guide as the bounded delivery contract.

Executor never self-accepts. `FINAL PASS`, `Owner Accepted`, and `Human Verified` are supervisor/Owner states only.

## Mandatory new-item Testing reset

P2-006 is a **new Task Catalog item**. Before any P2-006 implementation action, run the canonical deep Testing reset:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

Record `RESET_OK` or the exact blocker.

This reset is runtime/worktree hygiene only. It must **not**:

- downgrade or roll back canonical Supabase schema;
- delete accepted P2-003/P2-004/P2-005 schema;
- reconstruct product repositories from older P2 bases;
- rewrite accepted product history.

After reset, verify product heads/branches. Create/switch the intended `outbound/p2-006` branch from the exact accepted P2-005 Mercato head in every product repository that actually changes. Do not rebase accepted history.

## Task Catalog objective

Ensure `CROSSDOCK` lines and Packing TUs join the **same** Shipment, carrier, label, ERP posting, CarrierManifest and final settlement pipeline already accepted for P1, while preserving priority/SLA grouping, the CustomerOrder-level mixed-channel completeness guard, the P2-005 Goods Receipt gate and P1-016 exactly-once final settlement.

Task Catalog references:

- P2 R25–R26;
- P2 R33;
- P2 R43;
- P1 R57–R58;
- P1 R37–R41;
- P1 R43–R46;
- `FR-P2-13`, `FR-P2-18`, `FR-P2-25`, `FR-P1-31`, `FR-P1-32`, `CON-04`, `CON-05`;
- acceptance scenarios `TC-028`, `TC-030`, `TC-031`, `TC-104`, `TC-105`, `TC-119`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`.

Dependencies already accepted:

- P2-005 — crossdock GR correlation and ERP gate;
- P1-011 — Shipment grouping and CustomerOrder completeness guard;
- P1-012 — Carrier Selection;
- P1-013 — WMS label lifecycle used by normal Shipment flow;
- P1-014 — ERP posting and Supervisor retry;
- P1-015 — CarrierManifest lifecycle;
- P1-016 — final line/order/inventory settlement.

## Architect invariants — implement literally

### 1. There is one logistics lifecycle, not a crossdock fork

A crossdock Packing TU that is ready for downstream handling must use the same persisted P1 objects and services as standard fulfillment:

`Packing TU -> Shipment -> Carrier Selection -> WMS label -> ERP Shipment posting -> CarrierManifest -> handover/confirm -> final settlement`.

Do **not** create a second crossdock Shipment table, manifest table, posting lifecycle, carrier lifecycle, dispatch state machine, or duplicate dispatch UI.

### 2. Crossdock Packing TU joins through the common Shipment relationship

A completed crossdock flow produces normal outbound Packing TU(s) in `PACKING_SEALED`. P2-006 must make those TUs eligible for the accepted common Shipment grouping mechanism.

The canonical relation remains the accepted P1 Shipment/TU/content/OutboundOrderLine relation. One Packing TU belongs to at most one active Shipment.

Do not copy crossdock quantities into a parallel Shipment ownership model.

### 3. Exact Shipment grouping key remains channel-neutral

The common Shipment grouping key remains:

- warehouse;
- customer;
- delivery address;
- priority;
- **identical** `slaDeadline`.

STANDARD and CROSSDOCK Packing TUs may enter the same Shipment only when this common key matches.

Crossdock `OutboundOrder.priority` and `slaDeadline` remain inherited from the parent `CustomerOrder` per P2 R43 / FR-P2-25. P2-006 must not recalculate or normalize them differently.

### 4. `allowPartialShipment = false` is a CustomerOrder-level mixed-channel promise

The accepted P1 R57/R58 guard is authoritative **regardless of channel**.

For `allowPartialShipment = false`, no Packing TU of that CustomerOrder — STANDARD or CROSSDOCK — may be attached to a Shipment until every active `CustomerOrderLine` is complete by the canonical rule:

- inactive/cancelled/supervisor-corrected line is excluded as defined by P1 R58; otherwise
- sum of `requiredQty` of all non-cancelled `OutboundOrderLine` rows for that CustomerOrderLine that are `PACKED` or beyond must equal the current `CustomerOrderLine.Quantity`;
- STANDARD and CROSSDOCK `OutboundOrderLine` contributions are summed together.

A status such as `PLANNED` alone never satisfies this guard.

### 5. Mixed STANDARD + CROSSDOCK must produce one complete Shipment when required

For one `CustomerOrder allowPartialShipment=false` legitimately split across STANDARD and CROSSDOCK:

- while either channel is incomplete, both channels' ready Packing TUs stay outside Shipment as required by P1 R58;
- once all active lines are fully covered, the compatible STANDARD and CROSSDOCK Packing TUs may enter **one** complete Shipment;
- a shared `slaDeadline` expiring cannot bypass the CustomerOrder completeness guard.

This is the core acceptance case from TC-122..TC-126.

### 6. `allowPartialShipment = true` preserves normal common Shipment behavior

For `allowPartialShipment=true`, an otherwise eligible crossdock Packing TU may join the normal Shipment flow without waiting for unrelated future demand, subject to the same grouping key and accepted P1 readiness rules.

Do not introduce a crossdock-only partial-shipment rule.

### 7. Common downstream services must remain channel-neutral

Once the Packing TU is in Shipment, reuse the accepted P1 services and state machine for:

- Shipment readiness;
- Carrier Selection;
- WMS label generation;
- ERP posting;
- CarrierManifest membership and lifecycle;
- final physical handover/confirmation.

Channel may be visible as provenance, but it must not select a second business pipeline.

### 8. P2-005 GR gate remains inserted before ERP posting

For a Shipment containing any confirmed crossdock contribution:

- all distinct source Inbound TUs that contributed confirmed crossdock quantity to **that Shipment** must have `GR_ACCEPTED` from `GR_SETTLEMENT_SOURCE=CROSSDOCK` before ERP posting can begin;
- STANDARD content creates no GR requirement;
- a mixed Shipment is blocked only by its unresolved/rejected crossdock sources;
- a P1-only Shipment with no crossdock contribution remains unaffected;
- PUTAWAY settlement never changes this crossdock gate.

Do not move the P2-005 gate later into manifest handling and do not weaken its pre-Phase-1 no-side-effect behavior.

### 9. Carrier Selection and label use the complete common Shipment

Carrier Selection must inspect the common Shipment/TU set exactly as accepted P1 does. CROSSDOCK TUs are ordinary Shipment Packing TUs for this purpose.

WMS label generation remains the accepted P1-013 internal label lifecycle. Do not add a crossdock-specific label API/provider lifecycle.

### 10. CarrierManifest is shared and retains irreversible boundaries

Only a normal eligible/posted Shipment may enter the accepted CarrierManifest flow.

Crossdock or mixed Shipment must obey the same rules:

- one Shipment belongs to one manifest;
- composition cannot change after `CarrierManifest CLOSED`;
- `CLOSED -> HANDED_OVER -> CONFIRMED` retains accepted P1-015 semantics;
- duplicate confirmation remains idempotent.

No crossdock manifest object is allowed.

### 11. Final settlement distinguishes provenance without forking the lifecycle

P1-016 remains the final settlement authority.

For STANDARD contributions, preserve accepted Allocation/Inventory settlement exactly.

For CROSSDOCK contributions:

- no `Allocation` exists and none may be manufactured merely to satisfy the standard settlement path;
- confirmed crossdock quantity came through `CrossDockPickTask`/Inbound GR and must not be decremented from standard warehouse `Inventory` as if it had been allocated from storage;
- the common manifest confirmation must still advance the relevant crossdock `OutboundOrderLine`, `OutboundOrder`, Packing TU, Shipment and CustomerOrder/Line aggregate states consistently with accepted P1 terminal semantics;
- mixed Shipment settlement must settle STANDARD inventory/allocation exactly once while advancing CROSSDOCK business states without a fake standard-stock effect.

### 12. Multi-Shipment and idempotency rules remain P1 rules

If an OutboundOrder/line legitimately contributes through more than one Shipment/manifest, do not complete it early. Preserve accepted P1-016 multi-manifest aggregation.

Repeated grouping, posting, manifest add/close/handover/confirm, and settlement retries must not duplicate membership, posting effects, manifest settlement or terminal state transitions.

### 13. Scanner has no second dispatch model

P2-006 has no new RF dispatch business model. Scanner should remain frozen unless a concrete product defect proves a Scanner source change is required.

Do not implement manifest/ERP/carrier decisions in Scanner simply because the source quantity originated from crossdock RF.

## Expected implementation shape

Use current repository truth and **extend/reuse accepted P1 services**, especially current repository-native equivalents of:

- `shipment-service.ts`;
- `carrier-selection-service.ts`;
- `shipment-label-service.ts`;
- `shipment-posting-service.ts` plus accepted P2-005 GR gate;
- `manifest-service.ts`;
- P1-016 final settlement logic;
- normal Shipment/Mercato operational surfaces.

Likely P2-006 work is a narrow compatibility/join layer and any minimum fixes required so accepted common services treat legitimate CROSSDOCK/mixed TUs correctly.

Do not create a crossdock duplicate of those services.

### Schema posture

The Task Catalog says **use the same Shipment/TU/line relation and settlement records** and **no separate crossdock Shipment schema**.

Prefer zero schema change if current accepted common relations can represent the required lineage. If a persisted relation is genuinely missing, use the smallest additive common-model change and prove why existing accepted relations cannot represent it. No destructive migration and no parallel crossdock dispatch schema.

## Canonical Testing

Canonical Testing database remains Supabase project `yzonugcenguvmojwiihb` / PostgreSQL 17. Local PostgreSQL is forbidden.

Use deterministic fixtures against canonical Testing PostgreSQL. Never write secrets to evidence.

Do not modify Demo/Prod.

## Required substantive PostgreSQL proof

Create a dedicated suite, preferably:

`p2-006-crossdock-shipment-downstream-postgres.integration.test.ts`

Test-case count may be lower than the list below only if evidence maps every behavior to substantive assertions. **All 20 behaviors below must be explicitly covered.**

1. A valid crossdock `PACKING_SEALED` TU can join the existing common Shipment model/service; no crossdock-specific Shipment object is created.
2. Exact grouping key for crossdock uses warehouse/customer/address/priority/identical `slaDeadline` and rejects mismatches.
3. Crossdock downstream preserves inherited `priority`/`slaDeadline` from the parent CustomerOrder and does not regroup incompatible P2 R43 orders.
4. One Packing TU cannot become a member of two Shipments; replayed grouping is idempotent/non-regressive.
5. `allowPartialShipment=false`: an incomplete STANDARD line blocks an otherwise ready CROSSDOCK TU from entering any Shipment.
6. `allowPartialShipment=false`: an incomplete CROSSDOCK line blocks an otherwise ready STANDARD TU from entering any Shipment.
7. Complete compatible STANDARD + CROSSDOCK lines of the same CustomerOrder enter one common Shipment (TC-125).
8. Partially covered crossdock `PLANNED` line does not satisfy the P1 R58 equality guard (TC-123).
9. Active CustomerOrderLine with no OutboundOrderLine has sum `requiredQty=0` and blocks the guard (TC-122).
10. Supervisor-corrected/inactive lines are handled exactly by the accepted P1 R58 alternatives; no stale pre-correction quantity blocks a complete order (TC-124).
11. `slaDeadline` expiry cannot pull a blocked `allowPartialShipment=false` Packing TU into Shipment or close around it (TC-105/TC-126).
12. `allowPartialShipment=true` crossdock TU follows normal common grouping/readiness without a crossdock-only wait rule.
13. Shipment readiness and late-TU behavior use accepted P1-011 rules for crossdock/mixed contributors; channel does not fork readiness semantics.
14. Common P1-012 Carrier Selection consumes crossdock/mixed Shipment TUs with the same deterministic rules and no crossdock carrier path.
15. Common P1-013 WMS label lifecycle works for a crossdock/mixed Shipment without provider-specific crossdock label behavior.
16. P2-005 GR gate blocks initial ERP posting of crossdock/mixed Shipment until every contributing CROSSDOCK source is accepted; STANDARD-only content creates no GR blocker and blocked posting has zero P1-014 side effects.
17. After all required CROSSDOCK sources become `GR_ACCEPTED`, accepted P1-014 posting/retry works normally; PUTAWAY result remains irrelevant to the gate.
18. A posted crossdock/mixed Shipment enters the existing CarrierManifest exactly once and obeys P1-015 `OPEN -> CLOSED -> HANDED_OVER -> CONFIRMED` composition/irreversibility rules.
19. Manifest confirmation settles a mixed Shipment exactly once: STANDARD allocation/inventory effects remain correct while CROSSDOCK contribution creates no fake Allocation or standard Inventory decrement; crossdock TU/line/order/customer aggregates advance correctly.
20. Multi-Shipment/final aggregate and retry semantics remain accepted P1-016 behavior: no line/order completes before all required contributing manifests are confirmed, and duplicate confirm/replay creates zero duplicate settlement.

The dedicated proof must explicitly state whether any new schema was required and demonstrate that there is still exactly one Shipment/manifest/posting/settlement model.

## Mandatory regressions

Because P2-006 deliberately joins multiple accepted P1 surfaces, run the full relevant accepted suites on canonical Testing PostgreSQL:

- P2-006 dedicated suite — all 20 mapped behaviors green;
- P2-005 GR gate PostgreSQL — accepted **19/19** remains green;
- P1-011 Shipment grouping PostgreSQL — full accepted suite green;
- P1-012 Carrier Selection PostgreSQL — full accepted suite green;
- P1-013 label PostgreSQL — full accepted suite green;
- P1-014 ERP posting PostgreSQL — accepted **18/18** remains green;
- P1-015 CarrierManifest PostgreSQL — full accepted suite green;
- P1-016 final settlement PostgreSQL — full accepted suite green;
- P2-002 crossdock planning PostgreSQL — accepted **22/22** remains green for P2 R43 priority/SLA/channel inheritance;
- P2-004 recovery **16/16** and P2-003 execution **8/8** if implementation touches crossdock execution/finalization/TU-state code; otherwise record why they are retained rather than rerun.

Run P2-001 or broader P1 suites if their surfaces are actually touched.

If shared Inventory/TU/warehouse/lock/orchestration primitives are modified, run directly relevant accepted shared/Inbound regressions. Inbound remains CLOSED/REFERENCE; do not reinterpret its behavior.

Run repository-native Mercato generation/build/typecheck/build-app contract after final code.

## Real rendered Playwright proof — zero route mocks

Use canonical running services and **zero Playwright route mocks/interception/substitution of product behavior**.

A dedicated Mercato spec is expected, preferably:

`P2-006-crossdock-shipment-downstream-ui.spec.ts`

### Journey A — crossdock joins the full common downstream pipeline

Through deterministic setup plus real rendered Mercato operations, prove a genuine crossdock Packing TU:

1. exists as `PACKING_SEALED` with persisted CrossDockPickTask/placement/source lineage;
2. is attached through the real common Shipment action/service, not direct fake crossdock Shipment state;
3. appears in the normal Shipment operational UI;
4. reaches normal Shipment readiness;
5. uses normal Carrier Selection and WMS label flow;
6. shows the P2-005 GR gate in the same Shipment UI;
7. rendered ERP posting is blocked with zero side effects while a required crossdock source is pending/rejected;
8. a valid CROSSDOCK `GR_ACCEPTED` integration result unblocks the same Shipment;
9. rendered normal ERP posting succeeds;
10. Dispatcher uses the normal rendered CarrierManifest flow to add the Shipment, close, hand over and confirm;
11. persisted reconciliation proves final Shipment/TU/OutboundOrderLine/OutboundOrder/CustomerOrder state and no crossdock Allocation/standard Inventory settlement was invented.

External-system GR result delivery may be triggered through the real accepted integration ingress; it is not a user UI action. Do not route-mock it.

### Journey B — one `allowPartialShipment=false` CustomerOrder across STANDARD + CROSSDOCK

Prove the core mixed-channel contract:

1. one CustomerOrder has at least one legitimate STANDARD contribution and one legitimate CROSSDOCK contribution with compatible customer/address/priority/identical `slaDeadline`;
2. while either active line is incomplete, a ready Packing TU from the other channel remains `PACKING_SEALED` outside every Shipment;
3. expiry of `slaDeadline` does not bypass this guard;
4. after both channel contributions satisfy the P1 R58 equality rule, use the normal rendered/common grouping flow;
5. both Packing TUs belong to the **same** Shipment;
6. normal Shipment UI shows one combined Shipment, not two channel-specific shipments;
7. P2-005 gate requires only the crossdock source TU(s), not standard content;
8. after GR acceptance, the same common Carrier/label/ERP/manifest pipeline completes;
9. persisted reconciliation proves exactly-once standard Inventory/Allocation settlement and Allocation-free crossdock settlement.

### Journey C — grouping incompatibility / one-Shipment invariant

Prove at least one real rendered/common negative boundary:

- incompatible `slaDeadline`, priority, customer or address does not merge into the same Shipment; **or**
- replaying the same TU grouping cannot attach it to a second Shipment.

If this is fully and visibly proved in Journey A/B plus substantive PostgreSQL assertions, a separate test case is optional.

## Runtime contract

After final Mercato code changes:

- run repository-native generation/build contract;
- prove non-empty production manifests;
- restart canonical `mercato-localhost.service` from the exact final Mercato candidate;
- prove service active/running, MainPID/child owns port 3009, `/login` HTTP 200;
- prove build/runtime provenance for every changed/new P2-006 route/action.

If Scanner source remains unchanged, do **not** create noise by rebuilding/changing Scanner. Verify frozen Scanner SHA/worktree remains clean. If Scanner is changed for a proven reason, fresh canonical `scanner-testing.service` export/restart and relevant real regression become mandatory.

Do not let Playwright `webServer` / `reuseExistingServer` mask stale canonical runtime.

## Hard exclusions

P2-006 must not implement:

- a separate crossdock Shipment schema/lifecycle;
- a separate crossdock Carrier/label/manifest/posting model;
- new external carrier API/provider-label behavior;
- P3 Reservation Release;
- P4 Physical PutBack beyond already accepted P2-004 facts;
- Return Receipt;
- new Inbound behavior or GR retry ownership;
- Demo/Prod;
- local PostgreSQL;
- unrelated refactors.

No schema downgrade or destructive reset.

## Push and evidence

When all required gates are green:

1. Ensure Mercato worktree is clean.
2. If Scanner is unchanged, prove it remains exactly accepted P2-004 SHA `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` and clean; do not manufacture a Scanner commit.
3. Push exact tested Mercato final to `outbound/p2-006`.
4. If Scanner genuinely changed, push exact tested Scanner final to `outbound/p2-006`; otherwise report it as frozen accepted SHA.
5. Create `05_EVIDENCE/P2-006_EVIDENCE.md` recording:
   - exact final Mercato SHA;
   - exact Scanner frozen/final SHA;
   - migration/schema provenance and explicit statement that no separate crossdock Shipment/manifest model exists;
   - dedicated result/count;
   - explicit **20/20 substantive behavior mapping**;
   - P2-005 + P1-011/012/013/014/015/016 + required P2 regressions;
   - mixed STANDARD+CROSSDOCK CustomerOrder completeness proof;
   - grouping key and one-Shipment membership proof;
   - carrier/label common-pipeline proof;
   - P2-005 GR gate inside the common Shipment pipeline;
   - CarrierManifest common lifecycle proof;
   - final mixed settlement proof, including no fake crossdock Allocation/Inventory decrement;
   - zero-route-mock statement;
   - canonical runtime provenance;
   - clean/dirty worktree status.
6. Push WMS evidence to `main` and report its SHA.
7. Evidence must not self-declare `FINAL PASS`, `Owner Accepted`, or `Human Verified`.

## Final executor report only

Report exactly:

- final pushed Mercato SHA;
- Scanner final/frozen SHA and whether Scanner changed;
- P2-006 dedicated result/count and 20/20 mapping status;
- P2-005 regression result;
- P1-011/012/013/014/015/016 regression results;
- any P2-002/P2-003/P2-004 regression results required/rerun and why;
- mixed STANDARD+CROSSDOCK `allowPartialShipment=false` guard result;
- common Shipment grouping result;
- Carrier Selection + label result;
- P2-005 GR gate result in the common pipeline;
- ERP posting result;
- CarrierManifest lifecycle result;
- final mixed settlement / no-fake-crossdock-Allocation result;
- Journey A result;
- Journey B result;
- Journey C result or where the negative boundary is substantively proved;
- zero-route-mocks confirmation;
- Mercato runtime proof;
- WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty;
- if STOPped: one exact blocker.

Then STOP for supervisor verification. Do not start P3-001.