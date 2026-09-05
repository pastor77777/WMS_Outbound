# P1-016 — bounded closeout

**Purpose:** finish the already-implemented P1-016 work from the current local Mercato state without restarting the implementation from scratch.

This is a **closeout only**. The product implementation already progressed through two local checkpoints and the dedicated P1-016 PostgreSQL suite is reported **22/22**. Do not discard, rebase away, or restart that work.

## Current reported local checkpoint

Preserve the current Mercato local state:

`4cc2658f25795ff0d17a8ac7f67fd0e9325ffe45`

Earlier local checkpoint:

`450328a8fddf488189467e5d3d26cdb99215b769`

Accepted P1-015 base remains:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

Scanner remains frozen:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

WMS_Outbound P1-016 source guide used by the executor:

`06_AGENT_GUIDES/P1-016_EXECUTION.md`

The source guide remains the authority for P1-016 behavior. This file only narrows the remaining work.

## Already reported as completed

Do not rerun/rewrite merely for ceremony unless a later final-SHA change requires it:

- dedicated P1-016 PostgreSQL suite: **22/22**;
- P1-015 PostgreSQL regression: **21/21**;
- local Mercato implementation preserved at the checkpoint above.

If a product defect is discovered while producing the missing decisive evidence, make only the smallest P1-016 correction required and then bind all final evidence to the resulting new final SHA.

## Remaining mandatory work

Complete **all** items below before finishing the executor response.

### 1. Real PostgreSQL concurrency proof A

Duplicate/parallel settlement of the same manifest using two independent real PostgreSQL sessions/service calls.

Record in durable evidence:

- exact `pidA` and `pidB`;
- `pidA != pidB`;
- independent observer output tied to the exact blocked PID using `pg_stat_activity` / `pg_blocking_pids` or equivalent decisive PostgreSQL evidence;
- real lock wait/serialization evidence;
- one settlement contribution/business effect only;
- no duplicate Inventory effect;
- no duplicate `reservedQty` decrement;
- no duplicate `shippedQty` increment;
- no duplicate terminal transition/event.

### 2. Real PostgreSQL concurrency proof B

Two different manifests concurrently settling the same line/allocation using independent real PostgreSQL sessions/service calls.

Record:

- exact `pidA`, `pidB` and independent observer proof;
- genuine overlap/locking;
- no lost update;
- exact final arithmetic for `reservedQty` and `shippedQty`;
- each manifest contribution exactly once;
- correct nonterminal/terminal behavior independent of winner order.

### 3. Real rollback proof

Use a real transaction with deterministic injected failure **after a meaningful P1-016 settlement mutation has been staged/flushed and before successful completion**.

Then use a fresh independent EntityManager/session/read and prove no partial durable business state remains across the affected settlement identity, Inventory, Allocation, line/order/customer states and events.

Do not substitute mocked transaction behavior or same-session cached reads.

### 4. Mandatory regressions on the exact final Mercato SHA

Run and record exact literal command/results for at least:

- P1-016 PostgreSQL: **22/22 or greater**;
- P1-015 PostgreSQL: **21/21**;
- P1-014 PostgreSQL: **18/18**;
- P1-013 PostgreSQL: **15/15**;
- P1-012 PostgreSQL: **14/14**;
- P1-011 PostgreSQL: **18/18**;
- FND-002 state transitions: **77/77**;
- FND-002 transaction simulation: **8/8**;
- relevant accepted Allocation/shared Inventory/FND-003 suites for primitives actually touched by P1-016.

Any product correction after a test run invalidates that run as final evidence; rerun affected final-SHA evidence.

### 5. Fresh rendered Playwright 6/6

Serve canonical Testing from the **exact final 40-character Mercato SHA** and run the P1-016 rendered browser suite.

Minimum journeys remain the six required by the execution guide:

A. full one-manifest final settlement;
B. first of two manifests leaves nonterminal partial state;
C. later manifest completes remaining line/order/customer state;
D. post/manifest-boundary cancellation rejection with actionable reason;
E. whole-order atomic cancellation rejection with no partial side effect;
F. duplicate/repeated final confirm remains UI-stable and persisted business effects remain exactly once.

API/DB may prepare fixtures and verify persistence, but the decisive accepted user actions must go through the rendered application UI.

Required final result: **6/6 passed, 0 unexpected, 0 flaky** (or an equally explicit clean six-journey Playwright result if the runner formats it differently).

### 6. Final evidence

Create/update only:

`05_EVIDENCE/P1-016_EVIDENCE.md`

It must include the exact final Mercato SHA and all decisive literal evidence required by `P1-016_EXECUTION.md`, especially:

- accepted base and final SHA / merge base / compare scope;
- P1-016 test count;
- both exact-PID concurrency proofs;
- rollback fresh-read proof;
- settlement arithmetic;
- all mandatory regression counts;
- relevant shared Inventory/Allocation regression counts;
- exact served final SHA;
- final Playwright 6/6 literal result;
- frozen Scanner SHA;
- clean final worktree identities;
- scope exclusions;
- no secrets;
- no `FINAL PASS`, Supervisor acceptance or Owner Acceptance claim.

## Push boundary

Do **not** push an incomplete P1-016 completion claim.

Once every item above is complete:

1. push the final Mercato P1-016 commit(s) to the authorized P1-016 branch;
2. push `05_EVIDENCE/P1-016_EVIDENCE.md` to WMS_Outbound;
3. report only:
   - final Mercato SHA,
   - WMS evidence SHA,
   - P1-016 PostgreSQL count,
   - mandatory regression counts,
   - Playwright count,
   - frozen Scanner SHA;
4. STOP.

## Early-stop rule

Do **not** end the executor response merely because one remaining closeout phase completed.

Continue through all remaining phases in this guide in the same execution run.

Stop early only for a **genuine blocking condition that prevents further work**, such as an unavailable required Testing dependency, an unresolvable failing regression/product defect, or inability to produce required real PostgreSQL/runtime evidence after a genuine corrective attempt. If that happens, report the exact blocker and preserved local SHA; do not claim completion and do not broaden scope.

## Hard exclusions

- no P2/P3/P4 implementation;
- no Return Receipt;
- no external carrier API;
- no Scanner product changes;
- no Prod/Demo writes;
- no STATE/handover/canon/catalog/traceability/AGENTS/.ai steering changes.
