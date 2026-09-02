# P1-006 Remediation — Explicit Final Narrow Override

Owner explicitly authorizes one more narrow P1-006 remediation after STOP.

Use the **same existing Antigravity session**. Do not start another ticket.

## FIRST: synchronize this repository locally

1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main`, preserving unrelated local work.
3. Verify this local file is current:
   `06_AGENT_GUIDES/P1-006_REMEDIATION.md`
4. Read the LOCAL file completely.
5. Execute it in the SAME Antigravity session.

Do not reclone and do not replace the current session.

## Current heads to preserve

- Mercato `outbound/p1-006`: `dc22528c581ae212df5b83b713776b8e8110dfb5`
- Scanner: `6d9c70cad6bd868ba737b0cfbc18fa93f473e46e`
- Current WMS_Outbound evidence commit: `9dde98cf2dacd1b58fbbed1dd6b96a16ca347a7d`

Product behavior from the previous remediation is considered good for this shot:
- mandatory RF pick idempotency key;
- stable Scanner pending key behavior;
- R55 zone update;
- R62 SHARED and SEPARATE server behavior;
- R67 `PICK_FULL` switch guard;
- additive P1-006 migration;
- canonical operator identity;
- P1-009 Direct Pack removed from P1-006.

Do **not** redesign these areas. Fix only the three remaining acceptance/evidence blockers below.

---

## 1. Same-key concurrency proof must use independent real PostgreSQL transactions/connections

### Current defect

The current same-idempotency-key test uses parallel calls through the same service / EntityManager. That does not satisfy the evidence contract for two independent overlapping DB actors.

### Required proof

Add/replace the focused test so the SAME partial confirmation is executed by two independent application actors:

- same warehouse/task/taskLine/TU/location/SKU/operator;
- same `idempotencyKey = K`;
- same partial `pickedQuantity = 4` on a line with remaining quantity >= 10;
- Actor A uses its own fork/transaction/DB backend PID;
- Actor B uses a different fork/transaction/DB backend PID;
- `pidA !== pidB`;
- force genuine overlap using an application-path transaction hook/barrier, not only sleeps;
- while A holds the relevant application lock before commit, B must be observably waiting in PostgreSQL;
- an independent observer connection must capture DB-side proof tied to the real PIDs, preferably `pg_blocking_pids(pidB)` containing `pidA` and `wait_event_type='Lock'`;
- no inferred/fallback blocker PID;
- release A only after decisive DB evidence is captured;
- both application calls must resolve as the same logical confirmation/replay, not double-apply.

Fresh independent DB verification after both finish must prove:
- PickTaskLine `pickedQuantity = 4`, not 8;
- exactly one durable `wms_outbound_pick_confirmations` row for K;
- TU content reflects exactly one logical quantity 4 application;
- no second logical confirmation side effect.

Persist the actual console proof containing at minimum:
- actorAPid;
- actorBPid;
- blockingPids;
- waitEventType;
- waitEvent;
- lockType / lockMode when available;
- idempotencyKey;
- final pickedQuantity;
- confirmation row count;
- TU content quantity/count.

Do not weaken the already accepted over-pick concurrency test. This is an additional decisive SAME-K replay proof.

---

## 2. Add focused Scanner/client retry-stability test

### Current state

`PickingTaskScreen` now keeps `pendingIdempotencyKey` unchanged on error and regenerates only after success. Preserve this behavior.

### Required test

Add the smallest focused Scanner/client test that proves the actual component/client behavior:

1. render/use the picking flow with one pending pick action;
2. capture the idempotency key sent on the first confirmation attempt;
3. simulate an uncertain transport failure / lost response for that attempt;
4. retry the SAME pending operator action;
5. assert the second request uses the exact SAME idempotency key;
6. then simulate decisive success;
7. start the next genuinely new confirmation action;
8. assert it uses a NEW key.

A focused component/client test may stub the transport/API solely to inspect retry-key behavior. This does **not** replace the real zero-route-mock P1-006 Playwright acceptance journey, which must still be rerun and pass.

Do not reintroduce P1-009 scope.

---

## 3. Correct durable evidence and run the missing targeted Inbound regression

### SHA correctness

The current evidence document contains incorrect 40-character Mercato/Scanner SHAs.

After all commits are pushed, obtain the authoritative values directly from each local repo using `git rev-parse HEAD` and verify the remote refs match.

Update `05_EVIDENCE/P1-006_EVIDENCE.md` with the **exact real 40-char SHAs**. Do not guess, synthesize, shorten-then-expand, or manually alter SHA characters.

Record lineage from the current heads above to any new test-only commits.

### Missing regression

Run the required targeted accepted Inbound shared-TU / warehouse regression using the existing accepted Inbound test surface. Do not invent a new fake regression test just for evidence.

Persist:
- exact command;
- suite/test pass count;
- zero regression result;
- which shared TU / warehouse primitive coverage the command exercises.

### Final required reruns

After the narrow changes, rerun at minimum:
- focused P1-006 PostgreSQL suite including the independent SAME-K proof;
- focused Scanner retry-key test;
- genuine zero-route-mock P1-006 Scanner Playwright;
- P1-005 assignment regression;
- P1-008 TU regression;
- targeted accepted Inbound shared-TU / warehouse regression.

Update durable evidence with exact actual outputs/pass counts and exact final SHAs.

## Boundary / STOP

- Prefer test/evidence-only changes. Change product code only if a focused proof exposes a real defect, and then keep the fix minimal.
- No P1-007.
- No P1-009.
- No unrelated redesign.
- Do NOT self-declare Human Verified or FINAL PASS.
- Push implementation/tests/evidence and STOP.
