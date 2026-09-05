# P2-004 — Crossdock shortage, damage, empty-TU and cancellation recovery

## Status

Authorized next implementation item after explicit Owner acceptance of P2-003 on 2026-09-05.

Progress entering this item: **22/37 FINAL PASS / Owner Accepted**.

This is a **new Task Catalog item**. Before any P2-004 implementation, the executor must perform the mandatory controlled deep Testing reset using the canonical reset script from `Devaxonic-WMS`.

## Frozen accepted bases

- Mercato: `db0ef671b58ab13c2c0685205fbadcae1e1cf628` (`outbound/p2-003`)
- Scanner: `2ae72fb00db882fecae659b842e91efed17f949f` (`outbound/p2-003`)
- WMS P2-003 evidence: `f985d6099bdff939a0471012a25126baa8e216c2`
- P2-003 status: **FINAL PASS / Owner Accepted**

Do not reconstruct P2-003. Do not reset product history to P2-002. P2-004 must descend from these accepted P2-003 product bases.

## Mandatory pre-item Testing reset

Run before implementation:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

The reset is for Testing runtime/generated/cache state. It must not downgrade the canonical Supabase schema or erase accepted product history.

After reset, verify canonical repos/services are in the expected clean baseline state before creating P2-004 branches.

## Objective

Implement the architect-defined crossdock exception/recovery slice for:

- confirmed shortage;
- `DAMAGED` quantity;
- unexpected SKU;
- empty source TU;
- in-progress cancellation;
- residual/finalization rules;
- `allowPartialShipment=false` and `allowPartialShipment=true` outcomes;
- non-regressive source/target TU recovery and Inbound settlement handoff boundaries.

P2-004 extends the accepted P2-003 normal `OK` execution path. It must not weaken or reinterpret the accepted P2-003 quantity, identity, idempotency, task, line, order or target-TU behavior.

## Architect / Task Catalog grounding

Architect source/reference:

- P2 R11–R12
- P2 R13–R14
- P2 R15–R16
- P2 R17, R18, R37
- P2 R19–R20
- P2 R21–R22
- P2 R23–R24
- P2 R28
- P2 R34
- P2 exceptions: shortage or `DAMAGED` with `allowPartialShipment=false`
- P2 exceptions: shortage or `DAMAGED` with `allowPartialShipment=true`
- P2 exceptions: empty TU
- P2 exceptions: general cancellation while `CrossDockPickTask IN_PROGRESS`

Requirements:

`FR-P2-06`, `FR-P2-07`, `FR-P2-08`, `FR-P2-09`, `FR-P2-10`, `FR-P2-11`, `FR-P2-12`, `FR-P2-15`, `FR-P2-19`, `FR-P5-12`, `FR-P5-13`, `FR-P5-14`, `FR-P5-15`.

Acceptance mapping:

`TC-021`, `TC-023`, `TC-024`, `TC-025`, `TC-026`, `TC-027`, `TC-035`, `TC-039`, `TC-067`, `TC-081`, `TC-121`.

Task Catalog Definition of Done:

- `confirmed + damaged + residual` quantities reconcile to declared source quantity;
- cancellation never leaves an orphan active target or task.

## Target components

- Mercato `CrossDockPickTask` exception/finalization service(s)
- accepted P2-003 crossdock execution service extension where architecturally appropriate
- Scanner Crossdock exception UI/actions
- persisted exception/finalization facts
- Inbound settlement handoff boundary only where required by this item

## Required behavior

### 1. Preserve accepted P2-003 normal path

The following remain frozen accepted behavior and must regress green:

- lazy target creation on first successful placement;
- valid 1:1 GS1 source identity inheritance;
- non-GS1 and n:n generated target identity;
- exact decimal placement quantity;
- idempotent placement/completion;
- physical-full seals target while task continues;
- no crossdock Allocation creation;
- `confirmedQty <= plannedQty`;
- real Scanner Journey A/B behavior.

### 2. Quantity conservation

Persist enough authoritative facts to prove the source quantity disposition.

At finalization, the relevant architect-defined quantities must reconcile exactly. At minimum the implementation/evidence must make `confirmed`, `damaged` and `residual` explicit and demonstrate conservation against the declared/source task quantity for each applicable scenario.

Do not hide shortage/damage by mutating `plannedQty` or by retroactively rewriting accepted placement facts.

### 3. Shortage / damage

Implement server-authoritative exception handling for confirmed shortage and `DAMAGED` quantities.

The result must respect `allowPartialShipment` policy. Do not invent a generic single outcome for both policy values.

All state transitions must be idempotent, scoped by tenant/org/warehouse/task and reject impossible or duplicate effects safely.

### 4. Unexpected SKU and empty source TU

Unexpected SKU and an empty source TU are explicit exception paths, not normal `OK` placement.

Scanner must expose the architect-required user action/feedback through the real rendered flow. Backend remains the source of business truth.

### 5. In-progress cancellation

Implement the architect-defined cancellation behavior for `CrossDockPickTask IN_PROGRESS`.

Cancellation/finalization must not leave:

- an active task that should have ended;
- an orphan `active_target_tu_id`;
- an open target TU inconsistent with the final task outcome;
- duplicated settlement/effect rows;
- silently lost confirmed quantity.

Do not implement P3/P4 physical putback semantics here unless the Architect Source explicitly makes a P2-004 effect part of this task. P4 remains a separate Task Catalog slice.

### 6. Inbound ownership boundary

Inbound remains **CLOSED / REFERENCE**.

P2-004 may produce only the required Outbound-side handoff/facts for Inbound settlement. Do not move Inbound GR retry/ownership logic into Outbound and do not reinterpret accepted Inbound TU process semantics.

If shared Inventory/TU/warehouse/task-lock/orchestration primitives are modified, run targeted shared/Inbound regressions. If they are not modified, do not run broad Inbound suites by habit.

## DB / migration rules

Canonical Testing DB only:

- Supabase project: `yzonugcenguvmojwiihb`
- local PostgreSQL is forbidden

Schema changes must be additive, migration-backed and source-controlled. No manual canonical DDL, no DOWN/downgrade of the accepted P2-003 migration, and no destructive reset of accepted P2-003 schema.

Persist confirmed/damaged/residual and cancellation/finalization facts in a way that supports idempotency and deterministic reconciliation.

## Runtime rules

Mercato source/route changes:

1. repository-native full build including generation;
2. verify non-empty `.mercato/next/routes-manifest.json` and `required-server-files.json`;
3. restart only canonical `mercato-localhost.service`;
4. prove MainPID/runtime owns port 3009 and `/login` returns 200;
5. prove new/changed routes are served before Playwright.

Scanner source changes:

1. restart canonical `scanner-testing.service` so its fresh Expo web export is built;
2. prove service owns port 8081 and HTTP 200;
3. no stale Playwright `webServer` reuse as runtime proof.

## Tests required before evidence

### Dedicated P2-004 PostgreSQL coverage

Create/extend a real canonical PostgreSQL suite that substantively covers the P2-004 business behaviors, including at minimum:

- shortage with `allowPartialShipment=false`;
- shortage with `allowPartialShipment=true`;
- `DAMAGED` quantity handling;
- unexpected SKU exception;
- empty source TU;
- cancellation from `IN_PROGRESS`;
- exact confirmed/damaged/residual conservation;
- idempotent duplicate/replay of exception/finalization operations;
- no orphan active target/task after cancellation/finalization;
- correct state outcomes for task/line/order/target TU;
- no P2-005 GR gate implementation and no P2-006 shipment/ERP effect.

A smaller test count is acceptable only when evidence maps every required behavior substantively to assertions.

### Mandatory regressions

- P2-003 dedicated PostgreSQL suite: **8/8** baseline must remain green unless the accepted suite is deliberately expanded without weakening assertions;
- P2-003 Scanner real Journey A/B: **2/2**, zero route mocks;
- P2-002 dedicated PostgreSQL: **22/22**;
- P1-008 identity regression relevant to target-TU identity;
- P2-001 eligibility regression if binding/source eligibility surfaces are touched;
- shared/Inbound regression only when shared primitives are actually modified.

### Real Scanner Playwright

P2-004 is user-facing. Provide real Scanner Playwright against canonical services, with zero route mocks/interception, for the decisive exception flows.

At minimum the combined journeys must demonstrate through rendered UI and persisted DB reconciliation:

- shortage/damage policy outcome(s);
- empty/unexpected-SKU handling where user-facing;
- in-progress cancellation/finalization;
- quantity conservation and no orphan active target/task.

DB/API may create deterministic fixtures and verify final persisted truth; decisive user actions must be performed through the rendered Scanner/Mercato UI where the architect assigns them to a role.

## Hard exclusions

Do not implement in P2-004:

- P2-005 GR gate / GR accepted-pending-rejected lifecycle;
- P2-006 Shipment creation, ERP/manifest/carrier dispatch integration;
- P3 reservation release workflow;
- P4 PutBack workflow beyond any explicitly required P2 cancellation handoff fact;
- Return Receipt;
- Demo/Prod;
- unrelated refactors or architecture redesign.

## Branch / commit rules

Create new P2-004 product branches from the frozen accepted P2-003 heads:

- Mercato `outbound/p2-004` from `db0ef671b58ab13c2c0685205fbadcae1e1cf628`
- Scanner `outbound/p2-004` from `2ae72fb00db882fecae659b842e91efed17f949f`

Do not force-rewrite accepted P2-003 branches.

Keep worktrees clean at closeout. Push final product SHAs before WMS evidence.

## Evidence / stop condition

Create `05_EVIDENCE/P2-004_EVIDENCE.md` only after final product commits, required tests, canonical runtime proof and real Playwright are green.

Evidence must contain exact:

- Mercato final pushed SHA;
- Scanner final pushed SHA;
- migration/schema provenance if changed;
- dedicated behavior mapping;
- regression results;
- runtime/service/port proof;
- zero-route-mock Playwright results;
- persisted quantity/state reconciliation;
- clean-worktree proof.

Evidence must **not** self-declare `FINAL PASS`, `Owner Accepted` or `Human Verified`.

Then STOP and report exact SHAs/results/blockers for supervisor verification.

The executor never self-accepts P2-004.