# P2-006 — Full-item autonomous recovery and closeout

Target repo roots (exact, mandatory):

- WMS steering/evidence: `/home/ubuntu/git/WMS_Outbound`
- canonical runtime/tooling: `/home/ubuntu/git/Devaxonic-WMS`
- Mercato product/tests: `/home/ubuntu/git/Devaxonic-mercato`
- Scanner product/tests: `/home/ubuntu/git/Devaxonic-scanner`

This is the SAME P2-006 Task Catalog item. Current local VPS work is authoritative and must be preserved.

Before any product mutation, read in full:

1. `/home/ubuntu/git/Devaxonic-WMS/AGENTS.md`
2. `/home/ubuntu/git/Devaxonic-WMS/.ai/TESTING.md`
3. `/home/ubuntu/git/Devaxonic-WMS/.ai/OPERATIONS.md`
4. `/home/ubuntu/git/WMS_Outbound/AGENTS.md`
5. `/home/ubuntu/git/WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
6. `/home/ubuntu/git/WMS_Outbound/06_AGENT_GUIDES/P2-006_EXECUTION.md`
7. current WMS `STATE.md` and current handover

Also read and apply the current local `fetch_me_prompt` and `operational-mode` skills before continuing execution. The execution model for this ticket is full-item autonomy, not a chain of focused micro-tickets.

## Current preserved checkpoint — do not throw this away

Already proven in this same P2-006 item:

- P2-006 dedicated PostgreSQL acceptance matrix: 20/20 PASS.
- P2-005: 19/19 PASS.
- P1-011: 18/18 PASS.
- P1-012: 14/14 PASS.
- P1-013: 15/15 PASS.
- P1-014: 18/18 PASS.
- P1-015: 21/21 PASS.
- P1-016: 25/25 PASS.
- P2-002: 22/22 PASS.
- P2-003: 8/8 PASS.
- P2-004: 16/16 PASS.
- durable Mercato app typecheck: PASS, about 1.9 GB peak memory.
- durable fresh Mercato build: PASS with fresh non-empty production manifests.
- canonical `mercato-localhost.service`: healthy on port 3009 and `/login` HTTP 200 after bounded runtime recovery.
- Scanner remains clean/frozen at accepted SHA `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`.
- current P2-006 product change is narrow crossdock-safe manifest settlement: no fake Allocation and no standard Inventory decrement for crossdock quantity.
- current uncommitted P2-006 rendered UI spec exists at:
  `apps/mercato/src/modules/wms_outbound/__integration__/P2-006-crossdock-shipment-downstream-ui.spec.ts`.
- no Playwright route interception/mocks have been used.

Preserve all current Mercato work. Do NOT deep-reset, checkout, rebase, clean, reconstruct from P2-005, or repeat the new-item reset. Do NOT rerun the already-green full backend matrix unless a later product-code change genuinely invalidates one of those proofs.

## Exact current unfinished boundary

The latest durable P2-006 Playwright run reached real rendered Supervisor Shipment UI and successfully exercised normal carrier selection, label generation, and the blocked P2-005 GR gate.

It then failed at the real CROSSDOCK GR acceptance ingress with:

`HTTP 401 {"error":"Unauthorized"}`

The failure is currently an acceptance-fixture/auth-path problem, not a proven P2-006 business-code defect.

The endpoint is the real production route:

`POST /api/wms_outbound/cross-dock/gr-result`

That route requires authenticated tenant/org context and `wms_outbound.edit`.

The accepted P2-005 UI proof used the same real ingress contract after rendered UI login. The repository also contains a proven generic integration-auth mechanism in:

`apps/mercato/src/modules/wms_putaway/__integration__/FIFTH-CORRECTION-REOPENED-ui.spec.ts`

which imports:

- `apiRequest`
- `getAuthToken`

from:

`@open-mercato/core/helpers/integration/api`

Use proven repository auth helpers/patterns; do not hand-roll a new auth protocol.

## Goal

Finish the WHOLE P2-006 item from this preserved checkpoint and return only when:

A. P2-006 is fully green, committed, pushed and evidenced for supervisor verification; or
B. one real material blocker has survived two substantive evidence-based attempts and there is no normal in-scope next move.

Ordinary fixture/auth/TLS/runtime/test-data failures are executor-owned work and are NOT reasons to return control after one failure.

## Required autonomous execution

### 1. Inspect current local truth first

Record without mutating:

- Mercato HEAD, branch, `git status --short`, last 10 commits;
- current diff vs accepted P2-005 Mercato SHA `069f02d4c5c9b345b688b838eb685be02206afbd`;
- current P2-006 UI spec in full;
- current durable Playwright `results.json`, trace/error context from the 401 run;
- Scanner HEAD/status and confirm frozen state.

Do not discard any current local P2-006 work.

### 2. Stabilize the full P2-006 UI fixture before launching the next expensive run

Do not fix one line and immediately spend another durable run.

Review the entire P2-006 spec against accepted repository patterns, especially:

- P2-005 crossdock GR gate UI spec;
- P1-011 Shipment grouping UI spec;
- P1-012 carrier selection UI spec;
- P1-013 label generation UI spec;
- P1-014 ERP posting UI spec;
- P1-015 manifest dispatch UI spec;
- P1-016 final settlement UI spec;
- `FIFTH-CORRECTION-REOPENED-ui.spec.ts` for proven integration API authentication.

Fix all deterministic fixture/auth/session/test-plumbing defects you can identify in one pass before rerunning.

For the GR-result ingress specifically:

- keep it a REAL call to `/api/wms_outbound/cross-dock/gr-result`;
- no route mocks, interception, response substitution or direct service invocation;
- use a proven authenticated API mechanism that supplies valid tenant/org context and an identity with `wms_outbound.edit`;
- it is acceptable to use the repository's proven `getAuthToken` + `apiRequest` integration helper for this external ingress because the Architect/P2-006 proof requires real ingress, not a rendered human action for the external GR producer;
- keep the rendered Supervisor UI session for human-facing Shipment/Carrier/label/ERP/manifest actions;
- do not fabricate auth cookies or invent a new login endpoint contract;
- on any non-2xx ingress, capture status plus sanitized response body before diagnosis.

### 3. Own ordinary failures inside this same execution

Within P2-006 scope, autonomously diagnose and correct routine issues such as:

- auth/session mismatch;
- missing fixture row or wrong deterministic lookup;
- TLS/self-signed Testing behavior;
- stale selector/test data;
- incorrect request helper usage;
- late runtime readiness;
- test cleanup/idempotency defects;
- durable runner recovery.

Do not return to the Owner after each such failure.

Use the smallest relevant rerun while stabilizing. Preserve proven green backend/build/runtime evidence unless changed code invalidates it.

### 4. P2-006 rendered acceptance must finish all required journeys

The final dedicated zero-route-mock Mercato spec must prove the P2-006 main-guide journeys, not merely GR ingress.

#### Journey A — CROSSDOCK common downstream

Prove through real persisted state and rendered Mercato UI as applicable:

1. crossdock `PACKING_SEALED` TU with real task/placement/source lineage;
2. attach through the common Shipment model/service, not a parallel crossdock Shipment lifecycle;
3. normal Shipment UI/readiness;
4. normal Carrier Selection and label lifecycle;
5. P2-005 GR gate rendered and blocking;
6. ERP posting attempt blocked with zero Phase-1 posting side effects while GR unresolved;
7. valid real CROSSDOCK `GR_ACCEPTED` ingress;
8. normal ERP posting succeeds afterward;
9. existing Dispatcher CarrierManifest UI add/close/handover/confirm path;
10. persistence proves final crossdock states and no fake Allocation or standard Inventory decrement.

#### Journey B — mixed STANDARD + CROSSDOCK allowPartialShipment=false

Prove:

1. one CustomerOrder with legitimate compatible STANDARD and CROSSDOCK contributions;
2. while either active line is incomplete, the ready TU remains `PACKING_SEALED` outside Shipment;
3. slaDeadline cannot bypass the P1 R58 completeness guard;
4. once both channel contributions satisfy requiredQty equality, both Packing TUs enter one common Shipment;
5. normal UI shows one combined Shipment;
6. GR gate counts only required crossdock sources, not standard content;
7. common Carrier/label/ERP/manifest pipeline completes;
8. persistence proves standard Allocation/Inventory effects exactly once and allocation-free crossdock settlement.

#### Journey C — grouping/idempotency incompatibility

Explicitly prove either:

- incompatible customer/address/priority/slaDeadline does not merge; or
- replay cannot attach one TU to a second Shipment.

This may be folded into A/B if the assertion and persistence proof are explicit.

### 5. Product defect handling

If the stabilized fixture exposes a genuine P2-006 product defect:

- diagnose root cause;
- fix the smallest correct product code within P2-006 scope;
- rerun dedicated P2-006 plus only directly invalidated accepted suite(s);
- rerun typecheck/build/runtime only if the product change requires fresh proof;
- continue the item to completion.

Do NOT automatically stop merely because product code needs a legitimate P2-006 fix.

STOP for Owner decision only if fixing the defect would require business-scope expansion, destructive action, Demo/Prod access, a different executor/venue, or an Architect decision not already answered by canon.

### 6. Two-strikes rule — exact meaning

Two-strikes applies only to the SAME MATERIAL BLOCKER.

For one material path:

1. identify root cause and make one substantive evidence-based attempt;
2. if still blocked, make one genuinely different substantive correction;
3. only if the same material blocker remains after both and there is no ordinary in-scope next move, STOP and report it.

Do not count trivial fixture corrections, syntax mistakes, missing rows, selector fixes, TLS setup or unrelated later failures as consuming the same two strikes.

There is no arbitrary global rerun limit for the whole item.

## Evidence/build/runtime discipline

Do not rebuild or rerun typecheck merely because a fixture changed.

Current typecheck/build/runtime proofs may be retained if only the Playwright spec/fixture changes and runtime remains the same healthy built revision.

If product source changes after those proofs, refresh the affected build/typecheck/runtime evidence as required by the main P2-006 guide.

Use durable host-side execution for heavy commands so executor-terminal lifecycle cannot destroy auditability.

Final Playwright must produce durable `results.json` and artifacts and must run with zero route interception/mocks.

## Hard exclusions

- no separate crossdock Shipment schema/lifecycle;
- no separate crossdock carrier/label/ERP/manifest/settlement model;
- no external carrier API/provider additions;
- no P3 work;
- no P4 work;
- no Return Receipt;
- no Inbound business changes or GR retry ownership changes;
- no Demo/Prod;
- no local PostgreSQL;
- no unrelated refactors;
- Scanner remains frozen unless a concrete P2-006 defect demonstrably requires a Scanner change.

## Definition of done — no exceptions

P2-006 is ready for supervisor verification only when all are true:

1. P2-006 20/20 backend acceptance remains valid.
2. Required accepted regression matrix remains valid, with any genuinely invalidated suites rerun after later product changes.
3. Mercato typecheck/build proof is valid for the final product revision.
4. canonical Mercato runtime serves the final product revision on port 3009 with `/login` HTTP 200 and fresh non-empty manifests.
5. dedicated P2-006 zero-route-mock Playwright journeys A/B/C PASS.
6. persisted reconciliation proves common Shipment/carrier/posting/manifest lifecycle and correct mixed settlement semantics.
7. no fake crossdock Allocation or standard Inventory decrement exists.
8. Scanner is still at accepted SHA unless a justified change was required and fully proven.
9. minimum final Mercato diff is committed and pushed on `outbound/p2-006`.
10. `05_EVIDENCE/P2-006_EVIDENCE.md` is written with exact final SHAs, test counts, 20/20 mapping, retained/rerun proof rationale, runtime proof, zero-mock Playwright proof, ancestry and clean worktrees.
11. evidence does NOT self-declare `FINAL PASS`, `Owner Accepted` or `Human Verified`.
12. STOP before P3-001.

## Mandatory final Git step

After all acceptance gates are green:

1. inspect final Mercato diff and ensure only P2-006-required files are staged;
2. commit with a clear P2-006 message;
3. push exact `outbound/p2-006`;
4. confirm ancestry from accepted P2-005 Mercato SHA `069f02d4c5c9b345b688b838eb685be02206afbd`;
5. confirm Scanner exact accepted SHA `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` unless explicitly justified otherwise;
6. write/commit/push `05_EVIDENCE/P2-006_EVIDENCE.md` in WMS_Outbound;
7. confirm clean worktrees and exact 40-char final SHAs;
8. STOP for supervisor verification. Do not start P3-001.

## Final response

Return only one of these two outcomes:

### COMPLETE

- exact final Mercato SHA/branch;
- exact Scanner SHA;
- exact WMS evidence commit;
- P2-006 Playwright PASS count/journeys;
- retained/rerun backend/build/runtime proof summary;
- clean worktree/push status;
- any material risk that remains.

### TRUE TWO-STRIKES BLOCKER

- one exact blocker;
- exact first root-cause diagnosis and attempt;
- exact second materially different attempt;
- evidence showing the same blocker remains;
- current preserved refs/state;
- why there is no normal in-scope next move.

Do not return ordinary intermediate fixture/test failures as the final response.
