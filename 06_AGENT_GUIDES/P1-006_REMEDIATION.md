# P1-006 Remediation — First Shot

Use the **same existing Antigravity session**. Do not start another ticket.

## Current heads

- Mercato P1-006: `4274403d674a9481d0edc3c851989da49090aaed`
- Scanner P1-006: `9022d6979a92c196e1333d97a979d9726a46e5e7`

Preserve what is already good:
- real rendered Scanner RF picking;
- formal pickedQuantity persistence;
- task completion without auto-closing TU;
- real application rollback;
- genuine PostgreSQL lock concurrency proof;
- warehouse authorization;
- real R55 same-order continuation concept.

Fix only the blockers below.

## 1. Real pick idempotency

Current `confirmPickLine` has no durable confirmation/idempotency identity. A sequential or concurrent replay of the same partial confirmation must never increment pickedQty/TU contents twice.

Implement a durable server-authoritative idempotency contract:
- Scanner creates one stable confirmation key for one user pick action;
- network retries reuse the same key;
- backend persists the key in PostgreSQL using an existing suitable idempotency primitive or the smallest additive P1-006 persistence structure;
- unique scope prevents duplicate application;
- same key + same payload returns the already-applied result without second quantity/content write;
- same key + conflicting payload fails closed;
- client cannot bypass the invariant.

Prove with real PostgreSQL:
- partial pick 4/10 with key K -> picked=4;
- exact sequential replay K -> still picked=4, one logical TU-content confirmation only;
- two overlapping real transactions using K -> exactly one applied;
- fresh independent DB read proves quantity exactly once.

Use real connections/PIDs and DB-side blocking/unique evidence where material.

## 2. R55 operator zone change

After same-order continuation Zone A -> Zone B:
- Scanner/operator active zone must actually become Zone B;
- rendered UI must show Zone B;
- subsequent actions/next-task context must use Zone B, not stale route params.

Extend Playwright:
- assert Zone A before continuation;
- click continuation;
- assert Zone B after continuation;
- same open TU remains;
- perform Zone B pick.

## 3. R62 configurable Picking-TU strategy

Implement/reuse warehouse configuration for both Architect strategies:

### A. `SEPARATE_PER_TASK_OR_ZONE`
- completed task does not automatically reuse its TU for next-zone task;
- continuation requires/selects a separate Picking TU according to configured behavior.

### B. `SHARED_SAME_ORDER_CONSECUTIVE_ZONES`
- eligible same-order continuation reuses the same open TU.

Do not hardcode shared-TU behavior.

Use the smallest appropriate warehouse-level configuration and additive migration only if no existing canonical field exists.

Prove both strategies with real PostgreSQL integration tests. Playwright may cover the configured strategy used by the acceptance fixture, but backend tests must prove both.

## 4. R67 capacity-driven TU switch

Current UI exposes New TU at any time and an over-capacity attempt mutates `PICK_FULL` then throws, so the status rolls back.

Correct the lifecycle:
- switching for R67 is available because current TU reached the Architect-permitted operational full/capacity condition, not arbitrarily;
- capacity/full state must be durable and server-authoritative;
- do not rely on state mutation rolled back with a rejected pick;
- task remains active;
- next TU is created through accepted P1-008 primitives;
- subsequent pick continues on the same PickTask.

Prove TC-118 from an actual persisted capacity/full condition.

## 5. Remove P1-009 Direct Pack scope

P1-006 must not implement P1-009 directPack behavior.

Remove from the P1-006 Scanner journey:
- Declare Direct Pack toggle;
- `directPackDeclared` submission from `confirm-line`;
- P1-006 assertions requiring directPack inheritance.

Do not remove the already-accepted underlying P1-008-compatible field/schema if later work needs it; simply do not implement/exercise P1-009 behavior in P1-006.

## 6. Canonical operator identity

No `'operator'` fallback on human picking endpoints.

For:
- confirm-line;
- close-tu;
- switch-tu;
- continue-task;

resolve canonical authenticated user UUID using the accepted platform identity contract.

If canonical human operator UUID is unavailable, fail closed with 401/403. Never persist a synthetic actor string for these operator actions.

Add focused HTTP proof.

## 7. Regression and durable evidence

Rerun:
- focused P1-006 PostgreSQL suite;
- real P1-006 Scanner Playwright;
- P1-005 assignment regression;
- P1-008 TU regression;
- targeted accepted Inbound shared TU / warehouse regression.

Update durable P1-006 evidence with:
- exact Mercato + Scanner SHAs/lineage;
- durable idempotency proof;
- Zone A -> Zone B rendered state proof;
- both R62 strategies;
- persisted R67 capacity/full proof;
- canonical operator UUID;
- rollback/concurrency;
- exact commands/pass counts;
- screenshots/trace;
- regressions.

## Boundary

- No P1-007.
- No P1-009.
- Do not self-declare Human Verified or FINAL PASS.
- Stop after push.
