# P1-010 — final acceptance override 2

**Execution status:** owner-authorized narrow override after prior override STOP  
**Catalog item:** `P1-010` — item **13/37**  
**Session:** continue the **SAME existing Antigravity session only**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

The owner explicitly authorized **one additional very narrow shot**. This shot is limited to the two remaining acceptance blockers only:

1. Warehouse Supervisor authority must use the real existing server-side ACL/role authorization seam without email/name heuristics or arbitrary client identity.
2. Durable evidence must satisfy the exact-ref / exact-command / exact-output contract.

Everything else in P1-010 is frozen. Do not redesign packing, repack, discrepancies, consolidation, Direct Pack, Scanner, or later scope.

If this shot does not truthfully close both blockers, **STOP**. Do not create another workaround and do not start P1-011.

## 0. Mandatory startup and frozen observed refs

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Refresh the current installed `wms-outbound`, `fetch_me_prompt`, and `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility. Re-read current `STATE.md`, handover, `GIT_PROMPT_WORKFLOW.md`, prior P1-010 execution/remediation/override guides, current Architect/traceability sources, and `.ai` testing/operations guidance.

Use `/home/ubuntu/.local/bin/agy-pl`; never bare `agy` and never replace the existing Antigravity session.

Supervisor-observed refs before this override:

- accepted Mercato P1-009 base: `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`
- current Mercato `outbound/p1-010`: `d072ce47b790ac2731f6a0a21360f9cd0fe9d7cf`
- accepted/frozen Scanner P1-009: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- current WMS evidence head before this guide: `62660f3d8466eb885724fd0ea0b5bbd55e4fd061`

Verify refs and ancestry before changing anything. Continue only on Mercato `outbound/p1-010` as a clean descendant of the accepted P1-009 base. Scanner remains frozen.

Do **not** update `STATE.md`, handover, task catalog, implementation plan, or `.ai` checkpoint to FINAL PASS. Supervisor owns acceptance.

## 1. Frozen accepted P1-010 areas — do not reopen

Preserve the currently working behavior and tests for:

- keep same TU;
- repack all;
- repack by SKU;
- defer + explicit missing/recheck;
- DAMAGED and unexpected/overage QC handling;
- source TU accounting/completion gate;
- quantity integrity;
- rollback and real PostgreSQL contention;
- rendered compatible/incompatible consolidation;
- accepted P1-009 Direct Pack behavior;
- Scanner at the accepted P1-009 head.

Only touch code/tests directly required to fix Supervisor authorization and to prove it.

## 2. Remaining blocker A — remove all Supervisor authority heuristics

The current implementation is still unacceptable because it can treat a user as Supervisor based on heuristic properties such as:

- email containing `admin` or `supervisor`;
- broad/general role names such as `manager` without the required WMS authorization;
- any client-provided `supervisorId` or equivalent identity assertion.

Remove those authorization shortcuts from the P1-010 decision path.

### Required final authorization model

1. **Never trust Supervisor identity or authority from the request body.** Client fields may carry a reason and, only if the established product flow requires it, credentials that are validated server-side; they may not directly establish `supervisorId`, role, privilege, or authorization.
2. The normal Packer/operator identity continues to come from the authenticated request context.
3. Determine Warehouse Supervisor authority using the **existing accepted server-side auth/ACL seam**. Inspect the current auth feature/ACL helpers and reuse them rather than hand-rolling a parallel role model.
4. `wms_outbound.manage_orders` is the explicit current Outbound management feature available in the module. Accept only the actual product-defined super-admin/global-wildcard semantics or the exact authorized WMS management feature/role mapping established by the existing ACL framework.
5. **Do not authorize by email substring, display name, or arbitrary generic role name.** A user named `supervisor@example...` with no required ACL must be rejected.
6. Tenant / organization boundaries must be enforced for both the Packer and approving Supervisor. A valid Supervisor from another tenant/org must not approve this mutation.
7. If the product supports a same-session Supervisor action, the current authenticated user may approve only when the server ACL check proves that user has the required Warehouse Supervisor authority.
8. If a normal Packer uses a two-actor approval flow, verify the secondary Supervisor through the existing secure authentication mechanism, then perform the same server-side ACL check on that authenticated candidate. Never accept only an email/id string.
9. Persist `overrideBy` / SupervisorDecision supervisor identity from the server-validated authorized Supervisor and persist the reason.
10. Preserve R66 exactly: `externalIssuable=false` is non-overridable even by a valid Supervisor.

Do not invent a new `warehouse_supervisor` system if the existing ACL feature framework already defines the accepted authority.

## 3. Required decisive authorization tests

Strengthen the final checked-in P1-010 PostgreSQL/service/API tests so the following are independently decisive:

1. **Normal Packer, no approval** -> below-threshold `REPACK` suggestion cannot be overridden.
2. **Fake Supervisor by email/name only** -> a user whose email/name contains `supervisor` or `admin` but lacks the required ACL is rejected.
3. **Generic non-authorized role** -> a user with a broad role that does not grant the required WMS management feature is rejected.
4. **Client identity spoof** -> arbitrary body `supervisorId`/role/authority cannot create approval.
5. **Cross-tenant/cross-org Supervisor** -> rejected.
6. **Valid authorized Warehouse Supervisor** -> accepted only after real server-side auth/ACL validation; fresh DB read proves `overrideBy` and SupervisorDecision contain that exact authorized user and reason.
7. **R66** -> `externalIssuable=false` remains rejected even with valid Supervisor authority.

Use real current auth/ACL entities/helpers and remote Testing PostgreSQL. Do not simulate authorization with a boolean flag or test-only bypass.

## 4. Required rendered UI proof — Supervisor path only

Do not rebuild the existing six-journey UI suite. Change only what is necessary to make the Supervisor journey truthful.

The final rendered Supervisor journey must demonstrate the actual product behavior:

- below-threshold TU shows `REPACK` suggestion;
- a normal Packer cannot self-approve merely by entering a reason;
- the real accepted Supervisor authorization flow occurs through rendered UI / existing auth interaction;
- the server validates the approving Supervisor's real authority;
- successful approval persists the validated Supervisor actor + reason;
- a negative authorization path is visible/decisive before the successful authorized path.

If the current supported product model is that only a currently logged-in Supervisor can approve, run the negative Packer journey under a genuine Packer session and the positive journey under a genuine authorized Supervisor session. If the accepted UI supports two-actor credential approval, exercise that actual path instead.

Do not claim a two-actor flow when the checked-in Playwright spec does not perform it.

Keep the already-proven consolidation journey unchanged except for unavoidable fixture/helper adjustments.

## 5. Evidence exactness blocker

Update `WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md` only after all final reruns.

The current evidence is invalid because it records abbreviated refs such as `d072ce47b` and `b0d7c11` despite the explicit 40-character requirement.

### Mandatory final evidence rules

- exact final **40-character** Mercato `outbound/p1-010` head;
- exact frozen **40-character** Scanner head;
- exact relevant WMS pre-evidence/guide ref using full 40 characters;
- do not call an abbreviated hash an authoritative commit SHA;
- exact working directory and exact command actually executed for every reported acceptance/regression run;
- exact actual final output summary/counts;
- when individual test titles are printed, they must match actual checked-in source/output;
- exact Playwright base URL / app runtime / backend/API target provenance without secrets;
- safe remote Testing PostgreSQL identity;
- truthful changed-file scope;
- no PASS row in a summary table unless its exact command and actual output are recorded in the same evidence document;
- `PLAYWRIGHT VERIFIED` only; never HUMAN VERIFIED or FINAL PASS;
- no final WMS evidence commit SHA may be guessed before push — supervisor will fetch it independently.

Do not rewrite correct prior evidence claims merely for style. Fix exactness and update only results affected by the final Supervisor authorization change/reruns.

## 6. Mandatory final reruns

After the minimal authorization fix, rerun on the **final Mercato head**:

### Backend / PostgreSQL

- P1-010 genuine PostgreSQL suite with decisive Supervisor ACL negatives/positive;
- P1-009 direct-pack suite;
- P1-008 TU identity/issueability suite;
- P1-007 targeted suite where overlap remains;
- P1-006 targeted suite where overlap remains;
- full `src/modules/wms_outbound` backend umbrella.

### Mercato Playwright

- full final P1-010 Packer UI suite, with the corrected truthful Supervisor journey and existing consolidation journey;
- accepted P1-008 real UI regression if present and required by the current P1-010 overlap.

### Scanner

From frozen accepted Scanner head `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`:

- accepted P1-009 Scanner direct-pack 4-journey Playwright regression.

Do not modify Scanner.

Run targeted Inbound/shared regressions only if this narrow final diff touches a shared primitive; state explicitly whether it did.

## 7. Authorized write boundary

Authorized Mercato writes are limited to the smallest set required for:

- real Supervisor ACL/authorization enforcement on the P1-010 deviation path;
- the P1-010 tests/Playwright journey proving that exact behavior.

Authorized WMS writes are limited to:

- `05_EVIDENCE/P1-010_EVIDENCE.md`;
- P1-010 screenshots/artifacts genuinely regenerated by the final run if needed.

Do not modify Scanner. Do not edit accepted historical migrations. Do not touch P1-011 or later scope. Do not advance durable state/handover.

After pushing the final Mercato changes and WMS evidence, **STOP**. Do not ask the owner for logs, SHAs, screenshots, or test output. The owner replies only `done`; the supervisor independently verifies Git refs, diff scope, code, and evidence.