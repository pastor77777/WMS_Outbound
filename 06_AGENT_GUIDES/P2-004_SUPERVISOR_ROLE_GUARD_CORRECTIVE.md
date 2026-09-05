# P2-004 corrective — Supervisor role/state authorization

## Status

P2-004 remains **INCOMPLETE / NOT PASS** after the first Antigravity closeout. This is a continuation of the same Task Catalog item, not a new item.

Do **not** run the deep Testing reset for this corrective. Do not reset/rebase/reconstruct product history or roll back the canonical Testing schema.

Preserve the current pushed P2-004 candidates:

- Mercato: `0e75119a75c0032aa8806d27e467b949e37b510b` (`outbound/p2-004`)
- Scanner: `8f975f4545f0761f91df3a681a9ee13df0c5e193` (`outbound/p2-004`)
- provisional WMS evidence: `cf771268a5883e76115af07ad5755fa53bca0c25`

Frozen accepted P2-003 bases remain:

- Mercato: `db0ef671b58ab13c2c0685205fbadcae1e1cf628`
- Scanner: `2ae72fb00db882fecae659b842e91efed17f949f`

## Exact supervisor blocker

The current P2-004 implementation violates the Architect role boundary for `allowPartialShipment=false` shortage/DAMAGED escalation.

Architect truth: `WAIT`, `CANCEL`, and `ALLOW_PARTIAL` are **Warehouse Supervisor decisions**. The Packer/Scanner operator reports the exception and waits for the Supervisor decision; the Packer must not be able to approve the escalation itself.

Current defect to repair:

1. Scanner `CrossdockTaskScreen` exposes Supervisor decision actions directly in the Packer crossdock execution screen.
2. `/api/wms_outbound/cross-dock/execute` accepts `supervisor_decision` under the ordinary crossdock execution authorization (`wms_outbound.view` + warehouse/operator access), so an assigned Packer can invoke a Supervisor decision.
3. `resolveSupervisorDecision` must be state-constrained to the real escalated shortage case; it must not accept a Supervisor decision on arbitrary non-final crossdock tasks.
4. Audit `reportUnexpectedSku` and equivalent exception actions for explicit legal task-state guards. A completed/cancelled task must not accept new discrepancy effects.

This is an authorization/state-machine defect, not a reason to redo already-green P2-004 exception work.

## Existing repository pattern to reuse

Use the already accepted Supervisor API pattern in Mercato as the model:

`apps/mercato/src/modules/wms_outbound/api/supervisor/short-picked-decision/route.ts`

That surface requires `wms_outbound.manage_orders` and treats the authenticated actor as the Supervisor. Reuse/parallel this established pattern for crossdock; do not invent a new RBAC model.

## Required minimum repair

### 1. Separate Packer execution from Supervisor decision

- Remove `WAIT` / `CANCEL` / `ALLOW_PARTIAL` controls from the normal Packer `CrossdockTaskScreen`, or otherwise make them unavailable to the Packer role.
- The Packer Scanner flow may report shortage/DAMAGED/empty/unexpected SKU and show a waiting/escalated state, but must not execute the Warehouse Supervisor decision.
- Do not expose `supervisor_decision` through the ordinary Packer crossdock execute route with only `wms_outbound.view` authorization.

### 2. Add/use a Supervisor-authorized server surface

Implement the smallest repository-aligned crossdock Supervisor decision surface, following the accepted `api/supervisor/short-picked-decision` pattern.

Minimum authorization:

- authenticated Supervisor actor;
- `wms_outbound.manage_orders` (or the already-established stronger equivalent if current repo truth requires it);
- correct tenant/org/warehouse scope.

Do not trust an operator/supervisor id supplied by the client when the authenticated actor identity is available.

### 3. Enforce exact decision state

A crossdock Supervisor decision is legal only for the Architect-defined escalated shortage/DAMAGED path that actually requires Supervisor intervention.

At minimum prove and enforce:

- task is `SHORT_PICKED`;
- the exception/finalization context is the `allowPartialShipment=false` escalation requiring a decision;
- `WAIT`, `CANCEL`, `ALLOW_PARTIAL` are rejected outside that state/context;
- the first accepted decision is immutable / replay-safe according to current accepted semantics;
- no duplicate or contradictory effects are created;
- Packer direct attempts are rejected with no side effect.

### 4. Tighten exception-state guards

Audit P2-004 exception commands, especially `reportUnexpectedSku`, so no new exception/discrepancy fact can be recorded after a task is already final (`COMPLETED`, `CANCELLED`, or another non-executable final state). Add the minimum explicit guard/test if missing.

Do not broaden into P2-005/P2-006/P3/P4.

## Required executable proof

### Dedicated PostgreSQL

Update/add minimum tests proving:

1. Supervisor decision succeeds for the correct `SHORT_PICKED` + escalation context.
2. Packer/non-Supervisor authorization cannot execute the Supervisor endpoint/action and creates no state/fact side effect.
3. Supervisor decision outside the correct `SHORT_PICKED` escalation state is rejected with no side effect.
4. Decision immutability/replay behavior remains correct.
5. `reportUnexpectedSku` (and any equivalently exposed exception action found in this narrow audit) is rejected after final task state.
6. Existing P2-004 quantity conservation/finalization tests remain green.

Retain/rerun directly invalidated P2-004 and P2-003 regression gates. Rerun broader shared/Inbound regressions only if the corrective actually touches shared primitives.

### Real rendered UI — zero route mocks

The decisive role split must be proven through real UI and canonical services:

1. **Packer Scanner** reports the shortage/DAMAGED escalation and cannot execute `WAIT`, `CANCEL`, or `ALLOW_PARTIAL`.
2. **Warehouse Supervisor** performs the required decision through a rendered Mercato Supervisor surface using the authorized Supervisor account/feature set. API/DB may prepare and verify fixtures, but may not replace the decisive human-facing decision.
3. Persisted DB reconciliation proves the correct task/order/line/exception/finalization result after the Supervisor action.
4. A low-privilege/Packer direct HTTP attempt against the Supervisor decision surface is rejected (`403` or repository-standard equivalent) and leaves no side effect.
5. Zero Playwright route mocks/interception/substitution of product behavior.

If an existing Mercato Supervisor page can host the crossdock decision with a narrow additive change, use it. Do not invent a parallel admin application.

## Runtime contract

After final code changes:

- Mercato repository-native full build/generate contract must pass.
- `mercato-localhost.service` must be active/running from the exact final Mercato SHA; MainPID/child must own port `3009`; `/login` HTTP 200; required `.mercato/next` manifests non-empty.
- Scanner source changes require a fresh `scanner-testing.service` restart/export from the exact final Scanner SHA; service owns `8081`; HTTP 200.
- Do not let Playwright webServer/reuseExistingServer hide stale canonical runtime.

## Push and evidence

When all corrective gates are green:

1. Keep both product worktrees clean.
2. Push exact tested finals to `outbound/p2-004` in Mercato and Scanner.
3. Update `05_EVIDENCE/P2-004_EVIDENCE.md` so it records the new exact final product SHAs and supersedes the provisional role model.
4. Evidence must explicitly record:
   - Packer cannot execute Supervisor decisions;
   - Supervisor authorization feature/route used;
   - correct `SHORT_PICKED` escalation-state guard;
   - low-privilege rejection with zero side effect;
   - rendered Packer -> Supervisor role-separated journey;
   - persisted reconciliation;
   - zero route mocks;
   - final canonical runtime provenance;
   - relevant dedicated/regression counts.
5. Push updated WMS evidence and report its SHA.
6. Do **not** write `FINAL PASS`, `Owner Accepted`, or `Human Verified` into executor evidence.

## Final executor report only

Report exactly:

- final pushed Mercato SHA;
- final pushed Scanner SHA;
- dedicated P2-004 result/count;
- directly invalidated regression results;
- Packer authorization-negative result;
- Supervisor rendered decision journey result;
- invalid-state decision-negative result;
- post-final-state unexpected-SKU negative result;
- zero-route-mocks confirmation;
- Mercato runtime proof;
- Scanner runtime proof;
- persisted reconciliation summary;
- updated WMS evidence SHA;
- Mercato clean/dirty;
- Scanner clean/dirty;
- if stopped: one exact blocker.

Then STOP for supervisor verification.
