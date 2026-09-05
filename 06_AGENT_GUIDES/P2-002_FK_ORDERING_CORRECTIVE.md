# P2-002 — narrow corrective shot: FK persistence ordering

**Authorization:** Owner-approved narrow shot after the required two-strikes STOP.  
**Scope:** fix only the deterministic PostgreSQL FK persistence-ordering blocker already observed in the in-progress local `outbound/p2-002` work, then rerun only the dedicated P2-002 PostgreSQL suite and STOP.

## Frozen baseline

- Accepted P2-001 Mercato base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`.
- Active Mercato branch: `outbound/p2-002`.
- Authoritative full contract remains `06_AGENT_GUIDES/P2-002_EXECUTION.md`.
- Existing uncommitted P2-002 work from the stopped session is the working state to inspect; do not discard/rebuild it unless strictly required to repair this blocker.

## Observed blocker

The dedicated real PostgreSQL P2-002 suite failed twice because a newly-created `wms_outbound_order_lines` row is being persisted/flushed before its newly-created parent `wms_outbound_orders` header, causing a deterministic FK ordering failure.

## Authorized correction only

Inspect the exact planner/unit-of-work path producing the new CROSSDOCK `OutboundOrder` + `OutboundOrderLine` and make the smallest correct persistence-ordering fix.

Required behavior:

- the parent `OutboundOrder` must be durably insertable before any child `OutboundOrderLine` that references it;
- preserve the same transaction boundary and atomic planning semantics;
- do not weaken/remove the FK;
- do not bypass integrity with raw SQL, deferred constraints, disabled checks, retries masking the defect, or test-only sequencing hacks;
- do not change grouping, quantity, idempotency, concurrency, status, assignment, Scanner, Inbound, or P2-003 behavior;
- do not add unrelated refactors;
- preserve all existing local P2-002 work not implicated in this blocker.

Use the ORM relation/unit-of-work mechanism or an explicit minimal parent-first flush only if that is the correct production behavior for the existing model. The fix must apply to real application planning, not merely the test fixture.

## Verification for this shot

After the minimal fix:

1. run typecheck only if the edit requires it or compilation changed;
2. rerun the same dedicated real PostgreSQL P2-002 integration suite that failed;
3. record exact result/count and whether the FK failure is gone.

Do **not** proceed to the remaining P2-002 regressions, Playwright, evidence creation, commits, pushes, STATE/handover changes, or P2-003 in this shot.

## STOP

Report only:

- exact file(s) changed for the FK-ordering correction;
- exact mechanism used to guarantee parent-before-child persistence;
- dedicated P2-002 PostgreSQL result/count;
- any remaining first failure if the suite is still red.

Then STOP and return control to the Owner/Supervisor.
