# Evidence Standard — Playwright + Human Verified

## Principle

Outbound acceptance is **human behavior acceptance**, not document completion.

For every human-facing requirement/task, the preferred proof chain is:

`fixture/setup`
→ `login as real role`
→ `resolve/select real warehouse`
→ `open the normal Mercato/Scanner module`
→ `click/type/scan through the intended UI`
→ `observe the expected visible state`
→ `verify the authoritative persisted/server state`
→ `continue to the next reachable user action`
→ `Human Verified walkthrough`.

A green API/unit test alone is insufficient for a human-facing architect flow.

## Required evidence classes

### A. Component / domain

Use unit/component tests for:

- state guards and transition services;
- quantity math/ATP/eligibility;
- carrier tie-break;
- aggregation;
- cancellation boundaries;
- policy evaluation.

### B. Integration / persistence

Use integration tests for:

- transactions/row locking;
- idempotency keys;
- unique assignment;
- outbox/retry;
- legacy compatibility;
- inventory ledger exactly-once effects.

### C. Playwright user journey

Required for human-facing Mercato/Scanner behavior.

Playwright must operate the normal app surface. Direct API/DB calls may create deterministic fixtures or verify technical facts, but they must not replace the actual UI action under acceptance.

Each test must record:

- actor/role;
- warehouse;
- task/order/Shipment/TU identifiers;
- architect requirement(s);
- visible expected outcome;
- persisted expected outcome;
- next reachable action.

### D. Human Verified

Final user-facing acceptance requires a person to traverse the same normal application flow using the documented test data and expected results.

Record:

- build/commit under test;
- environment;
- actor;
- date;
- walkthrough path;
- pass/fail;
- defect references if failed.

## Required user journeys

The implementation plan must ultimately provide Playwright + Human Verified coverage for at least these architect journeys:

1. Standard full fulfillment end to end.
2. zero ATP/backorder/replanning.
3. partial allowed vs partial forbidden.
4. multi-zone picking and Picking TU continuation.
5. short pick auto-reallocation and Supervisor decision path.
6. direct pack.
7. repack/consolidation and packing discrepancy.
8. automatic carrier selection and manual fallback.
9. WMS label generation.
10. ERP posting retry/error/recovery.
11. manifest close, handover and confirmation boundaries.
12. general cancellation before pick.
13. cancellation after physical pick → Physical Putback.
14. cancellation after packing before the allowed boundary.
15. Outbound Crossdock 1:1.
16. Crossdock n:n sorting.
17. Crossdock shortage/damage/empty source TU.
18. Goods Receipt gate pending/rejected/re-evaluated.
19. Reservation Release policy paths.
20. Physical Putback invalid-location loop and successful inventory recovery.

## Shared regression

Any modification to shared Inventory, TU, Putaway primitives, warehouse assignment, record locks or orchestration requires regression evidence showing accepted Inbound behavior remains intact.

## Evidence storage convention

Product repositories may hold executable tests. This repository should reference evidence by stable path/commit/test ID in future `05_EVIDENCE/runs/<date>/` records; do not copy secrets or volatile screenshots without a stable evidence purpose.
