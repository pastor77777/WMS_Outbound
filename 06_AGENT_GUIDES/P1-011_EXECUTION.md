# P1-011 — Shipment grouping, closure and partial-shipment gates

**Execution status:** first implementation shot for catalog item P1-011  
**Catalog item:** `P1-011` — item **14/37**  
**Session:** this is a **NEW Antigravity session**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md` from current `WMS_Outbound/main`.

The previous P1-010 Antigravity session is closed. Start P1-011 from durable Git state only; do not reconstruct mutable state from chat history.

## 0. Mandatory fresh-session startup

FIRST sync the local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Use `/home/ubuntu/.local/bin/agy-pl`; never bare `agy`.

Refresh and read the current installed skills/contracts before implementation:

- `wms-outbound` — authoritative routing for Outbound business/source hierarchy;
- `architecture-context` — **reference-only** for accepted shared/Inbound Inventory, TU, warehouse, locking and orchestration compatibility; it must not redefine Outbound rules or expand P1-011;
- `fetch_me_prompt`;
- `operational-mode`;
- current real-evidence contract;
- `Devaxonic-WMS/.ai/TESTING.md` and `.ai/OPERATIONS.md`.

Then read, in current Git state:

1. `WMS_Outbound/STATE.md`;
2. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md`;
3. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`;
4. `WMS_Outbound/07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact `P1-011` section;
5. `WMS_Outbound/03_TRACEABILITY/requirements_index.csv` for P1-011 requirements;
6. `WMS_Outbound/03_TRACEABILITY/test_index.csv` for P1-011 acceptance scenarios;
7. `WMS_Outbound/03_TRACEABILITY/state_event_transitions.csv` for Shipment / TU / OutboundOrder transitions;
8. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — especially KROK 9 and rules R26–R30, R57–R60 plus the P5 E17 exception context;
9. current `Devaxonic-mercato` P1-010 implementation and migrations/entities/services/UI before designing any new surface.

Authority order for Outbound behavior:

1. current Architect Source / faithful translations;
2. Canon + traceability + exact current task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. Implementation Plan as delivery decomposition only.

If `architecture-context` conflicts with an Outbound R-rule, the Outbound Architect source wins. Use architecture context only to preserve accepted shared primitives.

## 1. Frozen accepted starting point

Supervisor-observed durable state before this guide:

- accepted progress: **13/37 FINAL PASS**;
- accepted P1-010 Mercato head: `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`;
- frozen Scanner head: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- accepted P1-010 durable evidence: `b5bb6429717402e0fb6969f7437ddaf673a8a174`;
- WMS steering head before this guide: `dff28103d60ef2ee91031c165f9aac0e33904811`.

Before editing product code, independently refresh remote refs and verify ancestry.

### Mercato branch

Create or use `outbound/p1-011` from the exact accepted P1-010 head `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`.

If `outbound/p1-011` already exists, verify it is a clean descendant of that head and has no unrelated work before continuing. Do not silently rebase unrelated changes into this task.

### Scanner

Scanner is **not an implementation target by default** for P1-011. Keep `Devaxonic-scanner` frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` unless current product architecture and exact P1-011 source prove that a minimal handoff-state display must change there. Do not create a duplicate Shipment workflow in Scanner.

Do not update `STATE.md`, handover, task catalog, implementation plan or `.ai` acceptance checkpoint. Supervisor owns final acceptance.

## 2. Authoritative P1-011 scope

Task Catalog objective:

> Create stable Shipment grouping from packed TUs and enforce SLA / `allowPartialShipment` / CustomerOrder completeness guards, including late-TU follow-up shipments.

Requirements:

- `FR-P1-13` — repack completion and Shipment grouping (P1 R25–R26);
- `FR-P1-14` — PACKED aggregate and READY_FOR_DISPATCH (P1 R27–R28);
- `FR-P1-15` — late TU and Carrier Selection start boundary (P1 R29–R30);
- `FR-P1-31` — full release when partial shipment is forbidden (P1 R57);
- `FR-P1-32` — hold TU out of Shipment until the CustomerOrder completeness guard passes (P1 R58);
- `FR-P1-34` — cross-order packing compatibility seam (P1 R60);
- `FR-P5-17` — completed fragment waits for the complete CustomerOrder where partial shipment is forbidden;
- `CON-04` — stable Shipment grouping under repeated/concurrent execution.

Dependencies `P1-003`, `P1-009`, `P1-010` are accepted and must remain non-regressive.

Target components:

- target WMS Outbound `Shipment` persistence/lifecycle;
- Shipment ↔ Packing TU / contributing order-line relationships;
- CustomerOrder completeness guard;
- idempotent grouping/readiness service;
- Mercato operational Shipment/readiness/blocked-reason surface.

## 3. Architect behavior that must be implemented exactly

### 3.1 Shipment grouping key — P1 R26

After the first eligible Packing TU is ready, create or supplement a `Shipment` using the Architect grouping boundary:

- same customer;
- same delivery address;
- compatible priority as defined by the current source/data model;
- **identical `slaDeadline`** — no tolerance window at Shipment grouping;
- same tenant / organization / warehouse scope as required by product data isolation.

Do not invent category, temperature, carrier, label, manifest or other grouping dimensions.

One Packing TU belongs to exactly one Shipment.

P1 R60 is checked at packing time, not invented as a second Shipment grouping algorithm: one Packing TU may contain multiple OutboundOrders only when the accepted R60 packing compatibility was valid, and all contents of that Packing TU must remain in one Shipment.

### 3.2 `allowPartialShipment = false` CustomerOrder guard — P1 R57/R58 + P5 E17

For a CustomerOrder with `allowPartialShipment = false`, **do not attach any Packing TU to any Shipment** until every CustomerOrderLine satisfies exactly one branch:

A. **Inactive line** — successfully cancelled or Supervisor-corrected so it no longer contributes required quantity; or

B. **Active line** — the sum of `requiredQty` across its non-cancelled OutboundOrderLines that are `PACKED` or later equals the current CustomerOrderLine quantity after any authorized correction, independent of fulfillment channel.

Important consequences:

- line status alone is insufficient; a partially covered `PLANNED` line must still block;
- a completed fragment must wait for the whole CustomerOrder when partial shipment is forbidden;
- while blocked, the Packing TU stays `PACKING_SEALED` and is not assigned to a Shipment;
- `slaDeadline` expiration **cannot bypass this guard**, because blocked TUs are not yet members of a Shipment;
- when the final blocking condition becomes satisfied, reevaluation must attach eligible waiting TU(s) without manual data repair.

Do not implement this guard at OutboundOrder-only granularity. It is a CustomerOrder-level promise across all contributing OutboundOrders/channels.

### 3.3 TU membership transition

On successful attachment:

`Packing TU: PACKING_SEALED -> IN_SHIPMENT`

Use the existing server-authoritative transition/audit foundation; do not mutate status only in client code.

Membership must be idempotent and non-regressive:

- repeating the same grouping operation must not create duplicate links or another active Shipment for the same TU;
- parallel grouping attempts must not double-assign one TU;
- incompatible existing Shipment membership must fail closed.

### 3.4 OutboundOrder PACKED -> READY_FOR_DISPATCH — P1 R27

Preserve the Architect aggregate:

- OutboundOrder becomes `PACKED` only when all its OutboundOrderLines and all contributing TUs are packed according to the accepted state model;
- from `PACKED`, it moves immediately to `READY_FOR_DISPATCH` as the signal consumed by Shipment readiness.

Do not falsely move an OutboundOrder forward because only one TU or one line is packed.

### 3.5 Shipment closure/readiness — P1 R28

A `Shipment` closes its TU membership and reaches `READY_FOR_DISPATCH` from `CREATED` when either:

1. all OutboundOrders currently contributing to that Shipment have reached their own `READY_FOR_DISPATCH`; or
2. the common exact `slaDeadline` has elapsed, even if not all expected TUs for partial-allowed work were ready.

This readiness transition must be server-authoritative, idempotent and safe under concurrent reevaluation.

Once a Shipment is `READY_FOR_DISPATCH`, P1-011 must treat its membership as closed for further grouping.

### 3.6 Late TUs — P1 R29

Packing TUs not ready before an SLA-driven closure do **not** enter the already-closed Shipment.

When a late TU becomes ready:

- create or join a new compatible `Shipment` under the same R26 grouping rules;
- the earlier TU remains in the earlier Shipment;
- the same OutboundOrder may therefore contribute TUs to multiple Shipments.

Do not regress this into one-OutboundOrder/one-Shipment logic.

### 3.7 P1 R30 boundary only

P1-011 ends at the stable `Shipment READY_FOR_DISPATCH` boundary.

Carrier Selection may start only after that state, but **do not implement P1-012 Carrier/Region/CarrierSetup selection here**.

Also do not implement:

- label generation / printing (P1-013);
- ERP posting/retry (later task);
- CarrierManifest lifecycle;
- physical dispatch / handover;
- downstream completion/settlement behavior owned by later tasks.

P1 R37–R41 / CON-04 references may constrain stability/immutability expected by later stages, but they do not authorize pulling those later workflows into P1-011.

## 4. Current-state audit before implementation

Before creating entities or migrations, inspect current Mercato at the P1-010 base and document in the final evidence what already exists:

- any target WMS Outbound Shipment entity/table/service;
- any legacy `public.sales_shipments`, outbound delivery, shipment-like Sales or fulfillment objects;
- existing TU↔Shipment fields/relations;
- state-transition support for `Shipment`, `OutboundTU`, and `OutboundOrder`;
- existing scheduler/reevaluation/orchestration patterns suitable for SLA readiness;
- existing Mercato operational routes/pages that should host Shipment visibility.

Do not treat `public.sales_shipments` or another legacy Sales object as authoritative P1 Shipment lifecycle without an explicit adapter/mapping justified by current architecture.

Prefer additive target WMS Outbound persistence consistent with accepted FND-001/FND-002 boundaries.

If an existing target Shipment implementation already satisfies part of P1-011, reuse it and add only missing behavior; do not create a parallel model.

## 5. Required backend behavior and transaction boundaries

Implement the smallest coherent server-side surface for P1-011. Exact function names may follow current module conventions, but the behavior must support:

1. evaluate CustomerOrder completeness for Shipment eligibility;
2. find-or-create compatible open Shipment by exact R26 key;
3. atomically attach an eligible Packing TU and transition it to `IN_SHIPMENT`;
4. reevaluate OutboundOrder `PACKED -> READY_FOR_DISPATCH` where the accepted aggregate is satisfied;
5. reevaluate Shipment `CREATED -> READY_FOR_DISPATCH` by all-contributors-ready or SLA elapsed;
6. ensure a ready/closed Shipment cannot accept later TU membership;
7. route late TU to a new compatible Shipment;
8. safely replay the same operation without duplicate Shipment/link/transition effects.

Use genuine database constraints and row/transaction locking where needed. Do not solve concurrency only with process-local mutexes or in-memory maps.

If a scheduler is needed for SLA expiry, reuse the current scheduler/orchestration pattern. Do not invent a new infrastructure subsystem. The readiness service must also be directly testable deterministically without long sleeps.

## 6. Decisive genuine PostgreSQL tests

Create a focused P1-011 genuine PostgreSQL integration suite using the approved Testing database and independent reads.

At minimum prove:

1. **R26 grouping happy path:** two compatible sealed Packing TUs join one CREATED Shipment; each TU transitions to `IN_SHIPMENT` and is linked once.
2. **Exact SLA key:** same customer/address/priority but different `slaDeadline` does not share a Shipment.
3. **Incompatible grouping:** different customer or address cannot join the same Shipment; preserve the accepted R60 seam.
4. **One-TU/one-Shipment:** replay is idempotent and a TU cannot be assigned to a second active Shipment.
5. **Parallel grouping / CON-04:** independent overlapping PostgreSQL transactions cannot double-create/double-attach the same TU; capture DB-side participant/locking evidence tied to the real operation.
6. **`allowPartialShipment=false` blocked:** one active CustomerOrderLine incomplete -> TU stays `PACKING_SEALED`, no Shipment membership, explicit blocked reason/result.
7. **PLANNED is not enough / TC-123:** partial `requiredQty` coverage still blocks despite line status.
8. **Inactive/corrected line / TC-124:** cancelled or authorized corrected line is evaluated according to the R58 A/B branches, not as an automatic blocker.
9. **Cross-channel completeness / TC-125:** CustomerOrder completeness is evaluated across contributing channels/OutboundOrders without implementing the P2 workflow itself. Use existing persisted channel semantics only.
10. **SLA cannot bypass false-partial guard / TC-126:** expired SLA does not attach or dispatch a TU that R58 still keeps out of Shipment.
11. **P5 E17 / TC-127:** already packed fragment waits for the complete CustomerOrder.
12. **All contributors ready / R28:** Shipment becomes `READY_FOR_DISPATCH` when all contributing OutboundOrders are ready.
13. **SLA closure / TC-105:** for partial-allowed work, expired common SLA closes the open Shipment even when a later TU is not ready.
14. **Late TU / R29:** TU becoming ready after closure goes to a second Shipment; earlier membership is unchanged; the same OutboundOrder may contribute to both.
15. **Non-regressive replay:** repeated planner/reevaluation after readiness creates no duplicate links, Shipment, or reverse state transitions.
16. **Real rollback:** force a real failure after at least one write/flush within the grouping transaction and prove through a fresh independent read that no partial membership/state survives.

Use actual final source/test names in evidence. Do not pre-write expected timings or counts.

## 7. Mercato UI / Playwright acceptance

Task Catalog requires a Shipment operational view and blocked reason plus Playwright Shipment readiness/blocking.

Use the **normal existing Mercato operational flow**, not a test-only page.

The rendered UI proof must make at least these outcomes visible and decisive:

### Journey A — eligible Shipment grouping/readiness

- operator reaches the normal packing/Shipment handoff surface with a real sealed eligible TU;
- UI shows the Shipment identity / membership after grouping;
- persisted DB proves the same TU is `IN_SHIPMENT` and linked exactly once;
- when readiness criteria are met, UI shows `READY_FOR_DISPATCH` (or the existing authoritative wording mapped to that exact state).

### Journey B — `allowPartialShipment=false` blocked

- real CustomerOrder with an incomplete active line;
- Packer completes/seals an otherwise eligible TU;
- rendered UI visibly reports why Shipment handoff is blocked / waiting for complete CustomerOrder;
- DB proves TU remains `PACKING_SEALED` and has no Shipment membership;
- after the final required coverage is satisfied through the normal supported setup/action path, reevaluation unblocks and the UI reflects Shipment membership/readiness without manual DB repair.

### Journey C — SLA closure + late TU

Prove through rendered Mercato Shipment view, where practical and deterministic:

- an open partial-allowed Shipment reaches readiness after its deadline;
- a later Packing TU does not mutate the closed Shipment and is shown under a new Shipment.

If exact wall-clock scheduling is unsuitable for browser determinism, fixture the deadline in the past through legitimate test setup and trigger the normal server reevaluation path; do not fake the UI state client-side.

Preserve P1-010 Packer journeys and rendered consolidation.

Evidence label for automated browser proof is **`PLAYWRIGHT VERIFIED`** only. Never self-declare `HUMAN VERIFIED` or `FINAL PASS`.

## 8. Regression gates

On the final P1-011 Mercato head, rerun and record literal output for:

- focused P1-011 PostgreSQL integration suite;
- P1-010 PostgreSQL packing suite;
- P1-009 Direct Pack PostgreSQL suite;
- P1-003 planning/grouping regression relevant to CustomerOrder/OutboundOrder grouping;
- full `src/modules/wms_outbound` backend umbrella;
- P1-011 Mercato Playwright Shipment readiness/blocking suite;
- accepted P1-010 Mercato Playwright Packer suite.

If the implementation touches shared Inventory/TU/warehouse/lock/orchestration primitives, run the exact accepted shared/Inbound regressions required by current `architecture-context` and the real-evidence contract.

If Scanner remains untouched, keep its accepted ref frozen and do not invent a Scanner Shipment workflow. Run Scanner regression only if a shared API/runtime change can materially affect the accepted Scanner path or current evidence contract requires it.

## 9. Evidence contract

Create/update:

`WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md`

Evidence must include:

- full 40-character accepted P1-010 base and final P1-011 Mercato refs;
- frozen Scanner ref (or truthful changed ref only if exact scope justified it);
- exact WMS guide/steering ref;
- exact changed-file scope and migration scope;
- current-state audit result for any pre-existing Shipment/legacy Sales shipment objects;
- safe real Testing PostgreSQL identity;
- exact working directories and commands;
- literal fresh terminal output captured from the final product head (use `tee`/lossless capture; do not reconstruct timings);
- genuine concurrency evidence with distinct PostgreSQL participants tied to the grouping mutation;
- genuine rollback proof with fresh independent DB read;
- exact Playwright titles/output, runtime/base URLs and visible journey assertions;
- explicit mapping to Task Catalog acceptance cases `TC-001`, `TC-006`, `TC-007`, `TC-104`, `TC-105`, `TC-107`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`, `TC-127`, `TC-128` where P1-011 owns or protects the boundary;
- truthful note that P1-012+ Carrier/label/ERP/manifest/dispatch behavior remains out of scope;
- label browser evidence only as `PLAYWRIGHT VERIFIED`.

Do not include secrets or test passwords.

## 10. Absolute stop boundary

Push the final P1-011 Mercato implementation branch and the durable WMS evidence, then **STOP**.

Do not update `STATE.md`, current handover, task catalog, implementation plan or `.ai` acceptance checkpoint. Do not start P1-012.

Do not ask the owner for logs, SHAs, screenshots or test output. The owner replies only `done`; the supervisor independently verifies Git refs, ancestry, diff scope, source and evidence.