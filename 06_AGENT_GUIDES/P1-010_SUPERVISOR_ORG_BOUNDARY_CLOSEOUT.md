# P1-010 — Supervisor organization-boundary closeout

**Execution status:** owner-authorized ultra-narrow product closeout shot  
**Catalog item:** `P1-010` — item **13/37**  
**Session:** continue the **SAME existing Antigravity session only**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

This shot has exactly one product blocker to close: a Warehouse Supervisor from the same tenant but a different organization must not be able to approve a P1-010 packing deviation for the current organization.

Do not broaden scope. Do not redesign Supervisor approval, packing, repack, consolidation, discrepancies, Direct Pack, Scanner, or later tasks.

If the organization boundary cannot be closed truthfully with the existing auth/RBAC model, **STOP**. Do not invent a parallel authorization system and do not start P1-011.

## 0. Mandatory startup and observed refs

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Refresh the current installed `wms-outbound`, `fetch_me_prompt`, and `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility. Re-read current `STATE.md`, current handover, `GIT_PROMPT_WORKFLOW.md`, all P1-010 closeout guides, `05_EVIDENCE/P1-010_EVIDENCE.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, and the current real-evidence contract.

Use `/home/ubuntu/.local/bin/agy-pl`; never bare `agy` and never replace the existing Antigravity session.

Supervisor-observed refs before this shot:

- accepted Mercato P1-009 base: `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`
- current Mercato `outbound/p1-010`: `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6`
- frozen Scanner `outbound/p1-009`: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- current WMS evidence head before this guide: `f10aeb32124ef6bd7702096382183578cbf4a7cb`
- durable accepted progress remains **12/37 FINAL PASS**; P1-010 is not accepted yet.

Verify refs and ancestry before editing. Mercato must continue on `outbound/p1-010` as a clean descendant of `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6`. Scanner remains frozen.

Do **not** update `STATE.md`, handover, task catalog, implementation plan, or `.ai` acceptance checkpoint. Supervisor owns final acceptance.

## 1. Exact remaining product defect

Current P1-010 Supervisor authorization correctly uses the authenticated actor plus server-side `RbacService` feature checking, but the helper currently rejects a different tenant explicitly while relying on RBAC organization visibility for organization scope.

The accepted P1-010 boundary is stricter for this approval mutation:

> The approving Warehouse Supervisor must belong to the **same tenant and the same organization** as the packing mutation. A Supervisor from another tenant or another organization must not approve it.

A same-tenant user whose own `User.organizationId` differs from `scope.organizationId` must therefore be rejected even if that user has a broad `wms_outbound.manage_orders`, `wms_outbound.*`, `*`, or super-admin-capable ACL that would otherwise satisfy feature matching.

This is an approval actor-ownership rule at the P1-010 mutation boundary, not a change to global RBAC semantics.

## 2. Required minimal implementation

Change only the smallest P1-010 authorization surface required to enforce the organization boundary.

`checkSupervisorAuthority(...)` (or the single equivalent helper actually used by the final route/service) must require all of the following before returning authorized:

1. Supervisor user exists and is active/not deleted.
2. `user.tenantId === scope.tenantId` under the current product data model.
3. `user.organizationId === scope.organizationId` under the current product data model.
4. The real server-side `RbacService.userHasAllFeatures(...)` check succeeds for `wms_outbound.manage_orders` in the same `{ tenantId, organizationId }` scope, including only the wildcard/super-admin semantics already provided by `RbacService`.

Do not authorize from email/name strings, generic role names, body `supervisorId`, client-supplied role/authority flags, or test-only booleans.

Do not weaken the already-correct two-actor credential validation or authenticated-caller Supervisor path.

Do not change global `RbacService` behavior to solve this P1-010-specific actor ownership requirement unless current installed architecture explicitly proves that the shared service itself is defective. Prefer the narrow packing authorization helper.

Preserve R66 exactly: `externalIssuable=false` remains non-overridable.

## 3. Decisive PostgreSQL test matrix

Extend the existing P1-010 genuine PostgreSQL authorization matrix minimally. Preserve all existing cases and add a decisive new case:

### Same tenant, different organization — MUST FAIL

Create a real user with:

- the **same tenant** as the packing fixture;
- a **different organizationId** from the packing fixture;
- a real role/RoleAcl that would otherwise satisfy `wms_outbound.manage_orders` (use a deliberately broad grant, including `organizationsJson = null` / unrestricted organization visibility if supported by the current schema, so the test proves the P1-010 actor ownership boundary rather than merely RBAC organization filtering).

Then attempt the below-threshold keep-same-TU deviation with that user as the Supervisor candidate.

Required result:

- operation is rejected as unauthorized for this Warehouse Supervisor approval;
- TU remains unchanged (`READY_TO_PACK`, no override applied);
- no `WmsOutboundSupervisorDecision` for that attempted foreign-organization approval is persisted.

Also preserve and rerun the existing decisive cases:

- normal Packer, no approval -> reject;
- fake Supervisor by email/name only -> reject;
- generic non-authorized role -> reject;
- client/self identity spoof -> reject;
- cross-tenant Supervisor -> reject;
- same-tenant/different-org Supervisor -> reject;
- `externalIssuable=false` -> reject even with valid Supervisor;
- valid same-tenant/same-org Warehouse Supervisor with real ACL -> succeed and persist exact actor + reason.

Use fresh independent DB reads for the negative organization-boundary assertion and the positive audit assertion.

## 4. UI scope

Do **not** redesign the UI. The current rendered P1-010 Supervisor Journey 5 already proves normal-Packer rejection followed by a genuine authorized Supervisor success path.

Rerun the complete existing P1-010 Playwright suite on the final Mercato head to prove this narrow backend authorization change did not regress rendered behavior. Do not add a new browser journey solely for cross-organization authorization unless the current UI naturally exposes a safe, deterministic way to select a foreign-organization Supervisor. The decisive same-tenant/different-org boundary may be proven in the genuine PostgreSQL/service integration suite.

Keep rendered consolidation unchanged.

## 5. Mandatory final reruns on the new final Mercato head

Because this shot changes Mercato product code, all Mercato PASS evidence used by the P1-010 final gate must be freshly rerun on the **new final Mercato head** and recorded literally.

From `/home/ubuntu/git/Devaxonic-mercato/apps/mercato` run and record the exact command + actual output for:

1. P1-010 PostgreSQL suite — including the new same-tenant/different-org rejection case.
2. P1-009 PostgreSQL regression.
3. P1-008 PostgreSQL regression.
4. P1-007 PostgreSQL regression.
5. P1-006 PostgreSQL regression.
6. Full `src/modules/wms_outbound` backend umbrella.
7. Full P1-010 Mercato Playwright Packer suite (all existing journeys).

Use `tee` or another reliable local capture so the evidence is copied from the actual fresh stdout/stderr rather than reconstructed from memory. Preserve secrets out of Git.

### Scanner

Scanner code must remain frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

Rerun the accepted P1-009 Scanner direct-pack 4-journey Playwright regression against the final runtime/API environment and record its fresh output. Do not modify Scanner.

### PostgreSQL / runtime provenance

Record fresh safe database identity and fresh lock-contention participant evidence if the P1-010 suite emits it. Record exact app/base URL and backend target provenance without secrets.

## 6. Evidence correction rules

Update only `WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md` after the implementation is committed and every required rerun has completed on the final refs.

Mandatory corrections:

- exact full 40-character final Mercato head;
- exact frozen Scanner head;
- exact WMS pre-evidence/guide head;
- actual fresh test output from this shot — never reuse times/PIDs/line references from the previous final head;
- exact commands and working directories for every PASS row;
- P1-010 test title/count updated truthfully if the authorization matrix test count/title changes;
- accurate description of Supervisor authority: the final rule is **same tenant + same organization + server-side `RbacService` authorization for `wms_outbound.manage_orders` (including only the wildcard/super-admin semantics implemented by RbacService)**. Do not claim authorization by role names such as `warehouse_supervisor`, `admin`, or `manager` unless the actual checked-in authorization code explicitly uses those names;
- explicitly state same-tenant/different-org Supervisor rejection is covered and passed;
- evidence label remains `PLAYWRIGHT VERIFIED` only — never `HUMAN VERIFIED` or `FINAL PASS`;
- do not guess the final WMS evidence commit SHA before push.

If any rerun fails, record the real failure and STOP. Do not relabel old green output as final-head output.

## 7. Absolute write boundary

Authorized Mercato writes are limited to:

- the minimal P1-010 Supervisor organization-boundary authorization helper/service;
- the existing P1-010 PostgreSQL integration test needed to prove the new negative case;
- unavoidable formatting/import adjustments directly caused by those changes.

Authorized WMS writes are limited to:

- `05_EVIDENCE/P1-010_EVIDENCE.md`.

Do not modify Scanner. Do not modify historical migrations. Do not change packing behavior outside Supervisor authorization. Do not touch P1-011 or later scope. Do not advance durable state/handover.

Push the final Mercato implementation and fresh WMS evidence, then **STOP**. Do not ask the owner for logs, SHAs, screenshots, or test output. The owner replies only `done`; the supervisor independently verifies Git refs, ancestry, diff scope, code, and evidence.
