# P1-007 Remediation — First Shot

This is the **first remediation shot** for `P1-007` after independent acceptance review.

Use the **same existing Antigravity session**. Do not restart, replace, or create a second agent session.

## FIRST: synchronize locally

1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main`, preserving unrelated local work.
3. Verify this local file exists and is current:
   `06_AGENT_GUIDES/P1-007_REMEDIATION.md`
4. Read this LOCAL file completely.
5. Refresh the same authority listed in `P1-007_EXECUTION.md`, especially current Architect P1 R42–R48, FR-P1-21..24, FR-P5-01..06, TC-060/061/062/121, current evidence contract/testing/operations docs.
6. Then remediate in the SAME Antigravity session.

## Current heads

Preserve these as the remediation bases:

- Mercato `outbound/p1-007`: `e9ec7d3b839923def1ecd1f0a1896c2767e9a8f8`
- Scanner: `e45967ab6d57631a2214df3a93aa57762f235ee4`
- Current P1-007 evidence: `WMS_Outbound` commit `8b4b83a27b6db6a99cb498ab98c22d1c4ae18f9b`
- Accepted P1-006 bases remain immutable.

Preserve what is already good:
- explicit RF Short Pick action exists;
- short task can terminate `SHORT_PICKED`;
- rendered Mercato shortage/recovery surface exists;
- additive P1-007 migration exists;
- R48 `blockSourceLocation=false` backend seam exists;
- no P1-009/P1-010/P4 UI scope expansion.

Fix **ONLY** the acceptance blockers below.

---

## 1. R44 reallocation must use REAL eligible ATP stock, not any unblocked location

### Current defect

Current `confirmPickLine` finds all `WmsWarehouseLocation` rows, excludes blocked/source locations, and selects the first remaining location. It does **not** prove that the location contains qualified ATP inventory for the SKU or enough eligible quantity.

Current code also directly rewrites/creates `WmsOutboundAllocation` instead of executing the accepted P1-004 reservation path.

This can create a replacement PickTask against an empty location and can bypass hard-reservation/ATP invariants.

### Required behavior

For automatic SHORT_PICKED reallocation:

- locate only real inventory for the same SKU in the same warehouse;
- stock must be ATP-eligible under the accepted Inventory contract;
- exclude the short/blocked source location and every active blocked location;
- use only actually available/unreserved quantity;
- reserve exactly the unresolved shortage quantity or the exact Architect-permitted available part;
- perform the hard reservation through the accepted P1-004 Allocation/Inventory application primitive, not ad-hoc `status/reservedQty` mutation;
- replacement PickTask must reference the resulting authoritative Allocation and exact eligible location;
- no phantom candidate location may create a replacement task.

If no eligible ATP stock exists, escalate to Supervisor and create no replacement Allocation/PickTask.

### Required proof

Real PostgreSQL integration:

A. another unblocked location exists but has **zero/no ATP inventory** -> no replacement task, Supervisor escalation;
B. another location has non-ATP stock only -> no replacement task;
C. another location has eligible ATP stock -> exactly one replacement reservation + PickTask for the missing quantity;
D. fresh DB read proves the reservation/Inventory facts are consistent with accepted P1-004 invariants.

Update Scanner Playwright fixture so the alternative location contains **real eligible ATP stock** before proving auto-retry.

---

## 2. R44 effective retry limit hierarchy is incomplete

### Current defect

Current implementation resolves only:

`warehouse.maxAutomaticShortPickReallocations ?? 1`

Architect requires:

1. customer-specific override when it is not `null`;
2. otherwise warehouse setting;
3. otherwise default `1`.

### Required behavior

Search for the existing canonical customer-level configuration first. Reuse it if present.

If no canonical field exists, add the smallest additive configuration needed in the correct customer/config ownership surface. Do not invent a duplicate concept on an unrelated execution row.

Snapshot/use the effective limit consistently for the unresolved shortage case and prove:

- customer override wins over warehouse;
- null customer override uses warehouse;
- null/unconfigured warehouse uses default 1.

---

## 3. SHORT_PICKED reallocation concurrency / exactly-once proof is missing

### Current defect

The focused P1-007 suite contains no independent real-connection concurrency proof for automatic reallocation. There is no strict `pg_blocking_pids` evidence for the same unresolved shortage and no parallel limit-boundary proof.

### Required behavior and proof

Use the real application service path with two independent MikroORM EntityManagers/connections and distinct PostgreSQL PIDs.

Prove:

A. two overlapping attempts to process the **same unresolved SHORT_PICKED case** create at most one replacement Allocation and one replacement PickTask;
B. retry counter increments exactly once;
C. at `maxAutomaticShortPickReallocations - 1`, parallel attempts cannot advance beyond the limit or create a second retry;
D. fresh independent DB read proves exact final count/rows;
E. when serialization blocks, capture strict observer evidence only if:
   - `pg_blocking_pids(pidB)` contains `pidA`, AND
   - `wait_event_type = 'Lock'`.

No manufactured blocker fallback and no raw-SQL-only business proof.

Also prove duplicate/replayed explicit SHORT_PICKED declaration does not double-apply picked quantity, duplicate location-shortage facts, or duplicate replacement work.

---

## 4. Supervisor authorization and idempotency must fail closed

### Current defect

Both Supervisor decision routes currently:

- require only `wms_outbound.view`;
- allow `idempotencyKey` to be omitted;
- use `auth.sub || auth.userId || 'SUPERVISOR'`.

The Mercato UI generates `Date.now()` idempotency keys inside each click handler, so an uncertain/lost response followed by retry creates a new key and can apply a second decision.

### Required behavior

For both SHORT_ALLOCATED and SHORT_PICKED Supervisor decisions:

- enforce the accepted Warehouse Supervisor permission/role contract, not generic read-only access;
- resolve canonical authenticated human user UUID;
- if canonical Supervisor identity/permission is unavailable, fail closed 401/403;
- never persist synthetic `'SUPERVISOR'` or another fallback actor;
- `idempotencyKey` is mandatory and non-empty at HTTP + service boundary;
- one logical pending Supervisor action owns one stable key;
- retry after uncertain transport outcome reuses the same key;
- generate a fresh key only for a genuinely new decision action;
- same key + same canonical decision payload returns prior committed result without duplicate audit/business writes;
- same key + conflicting material payload fails closed.

Material payload comparison must include the affected order/line ids, decision, correctedQuantity when applicable, reason where material, and canonical Supervisor identity.

Add focused HTTP/real-service proof for unauthorized/read-only/non-Supervisor caller and missing canonical actor.

---

## 5. R42 / R45 reservation return must use accepted ATP/Allocation recovery semantics

### Current defect

Current shortage decision service directly marks Allocations `RELEASED`, but the inspected SHORT_ALLOCATED cancel / SHORT_PICKED WAIT paths do not execute the authoritative ATP restoration flow required by Architect and the accepted P1-004/P1-002 contract.

### Required behavior

For SHORT_ALLOCATED cancellation and SHORT_PICKED WAIT/cancel recovery:

- release Allocation through the accepted reservation/release application primitive;
- restore exactly the Architect-defined quantity to the proper CustomerOrderLine ATPReservation/soft-reservation state;
- do not double-return quantities already physically/formally picked;
- do not mutate Inventory/ATP with ad-hoc field writes if an accepted service exists;
- preserve hard-reservation ledger/exactly-once invariants.

Specific Architect outcomes must be proved:

- SHORT_ALLOCATED cancel: failed line -> BACKORDERED; sister lines -> OPEN; released quantity returns correctly to ATPReservation; CustomerOrder aggregate outcome as R42;
- SHORT_PICKED WAIT: short line unresolved quantity returned correctly; other rolled-back ALLOCATED/PICKED non-PACKED lines returned through the same accepted recovery path; PACKED untouched.

Fresh independent DB reads must prove exact quantities, not only statuses.

---

## 6. R46 CANCEL_OR_CORRECT currently accepts arbitrary positive quantity

### Current defect

Current service accepts any `correctedQuantity >= 0` and, for any positive value, rewrites `CustomerOrderLine.orderedQuantity`, `OutboundOrderLine.requiredQty`, and Allocation quantity.

Architect allows only two named outcomes:

A. commercial quantity becomes `0` -> full cancellation/release + physical-return handoff for already picked goods;
B. quantity is corrected to the **authoritative quantity actually picked** -> no PutBack for that picked quantity, and `CustomerOrderLine.Quantity` + `OutboundOrderLine.requiredQty` change together.

### Required behavior

- `0` is allowed for full cancellation;
- any positive corrected quantity must equal the authoritative persisted actual picked quantity for the affected execution scope;
- reject arbitrary positive values, stale values and values larger/smaller than authoritative picked quantity;
- never trust the UI default as the invariant;
- do not directly rewrite Allocation reservedQty to an arbitrary commercial correction.

Prove TC-121 with fresh DB reads and explicit rejection tests for invalid positive corrections.

---

## 7. Persist the physical-return handoff required by R45/R46 without implementing P4

### Current defect

WAIT and full cancellation can involve already-picked physical stock, but the inspected service has no durable PutBack/physical-return handoff fact for later P4 consumption.

### Required behavior

Do **not** implement P4 PutBackTask lifecycle or RF PutBack.

Persist/reuse the smallest accepted durable recovery handoff/event required to say:

- physical stock has already moved;
- logical Allocation/execution is being cancelled/released;
- a later P4 physical-return process is required;
- exact CustomerOrder/OutboundOrder/Line/SKU/quantity/TU/source context and correlation are retained where available.

Use FND-002 transition/audit/outbox primitives if they are the canonical seam; otherwise add only the smallest additive P1-007 recovery-handoff persistence required.

Prove no immediate fake Inventory return occurs for physically picked goods.

---

## 8. TC-060 `allowPartialShipment=true` must execute a real application path

### Current defect

The current `1A` test manually seeds an already-ALLOCATED OutboundOrderLine/Allocation and then asserts those seeded values. It does not execute the shortage/allocation business operation that is supposed to produce the Architect outcome.

### Required proof

Drive the real application Allocation/planning/shortage path from a state where required quantity exceeds eligible hard-reservable stock and `allowPartialShipment=true`.

Prove the application itself produces:

- available portion executable/allocated;
- missing portion remains uncovered/BACKORDERED;
- no Supervisor decision/audit row;
- later planning remains possible for missing quantity.

No fixture-only assertion as decisive proof.

---

## 9. Playwright coverage is incomplete

### Scanner

Current Scanner Playwright reports short and proves replacement generation, but acceptance requires the real auto-retry continuation.

With real eligible ATP in the alternative location:

1. report SHORT_PICKED through rendered Scanner UI;
2. prove old task is SHORT_PICKED;
3. receive/continue the actual replacement task through normal Scanner flow;
4. rendered task/location changes to the eligible alternative location;
5. perform the missing pick through UI;
6. fresh DB verifies exact cumulative picked quantity and no duplicate work.

No decisive route mocks.

### Mercato Supervisor

Current Playwright executes only SHORT_ALLOCATED `ALLOW_PARTIAL`.

Add genuine rendered Supervisor UI coverage for distinct SHORT_PICKED outcomes:

- WAIT;
- CANCEL_OR_CORRECT to authoritative picked quantity (and full cancel path where practical/required by fixture split);
- persistent ALLOW_PARTIAL with reason.

Also cover SHORT_ALLOCATED cancellation or otherwise provide decisive normal-UI coverage for both TC-060 Supervisor branches.

Fixtures may prepare state; decisive decisions must go through rendered UI and real backend.

---

## 10. Regression and evidence must be complete and exact

Current durable evidence has incorrect/nonexistent head SHAs and omits required regression runs.

After remediation rerun and persist exact command/pass counts for:

- P1-007 real PostgreSQL suite;
- P1-007 Scanner Playwright;
- P1-007 Mercato Supervisor Playwright;
- P1-004 Allocation regression;
- P1-005 assignment regression;
- P1-006 RF + idempotency regression;
- P1-001 CustomerOrder aggregation/policy regression;
- P1-003 planning/requiredQty regression;
- P1-008/TU targeted regression if TU code remains touched;
- accepted Inbound Inventory/location/warehouse/TU compatibility regression for every shared primitive touched.

Update `05_EVIDENCE/P1-007_EVIDENCE.md` with:

- exact actual 40-char Mercato + Scanner SHAs and clean lineage;
- correct P1-007 scope including R42–R48 / FR-P1-21..24 / FR-P5-01..06;
- real ATP candidate/reallocation proof;
- customer->warehouse->default retry hierarchy proof;
- independent concurrent reallocation PIDs/lock evidence;
- stable Supervisor retry idempotency and authorization proof;
- exact ATP/Allocation return quantities;
- R46 0 vs exact-picked correction proof;
- physical-return handoff proof without P4 implementation;
- real TC-060 partial-true application path;
- all real Playwright results;
- all regressions;
- remaining gaps, if any.

Evidence labels may say `PLAYWRIGHT VERIFIED`; never self-declare `HUMAN VERIFIED` or `FINAL PASS`.

## Hard boundary / STOP

- No P1-009.
- No P1-010 packing UI/repack/QC.
- No Shipment/Carrier/label/ERP/manifest work.
- No P4 PutBackTask model/FIFO/RF implementation.
- No unrelated redesign.
- Do not rewrite accepted P1-006 history.

Push remediation implementation + durable evidence, then **STOP**.