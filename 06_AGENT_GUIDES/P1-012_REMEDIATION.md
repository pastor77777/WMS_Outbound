# P1-012 — Carrier Selection — remediation guide

**Execution status:** authorized bounded remediation for the already executed `P1-012` only  
**Catalog item:** `P1-012` — item `15/37`  
**Executor:** NEW Antigravity session from canonical `Devaxonic-WMS` checkout  
**Scope type:** remediation + evidence only; **NOT steering**  
**Workflow:** follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md` from current `WMS_Outbound/main`

## Mandatory startup

```bash
cd /home/ubuntu/git/Devaxonic-WMS
/home/ubuntu/.local/bin/agy-pl
```

Never launch bare `agy`.

Before touching implementation:

1. sync `WMS_Outbound/main` without overwriting unrelated work,
2. read current `Devaxonic-WMS/AGENTS.md`, `.ai/STATE.md`, current Outbound handover, `.ai/TESTING.md`, `.ai/OPERATIONS.md`,
3. read current `WMS_Outbound/STATE.md`, current handover, `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`, `06_AGENT_GUIDES/P1-012_EXECUTION.md`, and this remediation guide,
4. inspect current `Devaxonic-mercato/outbound/p1-012` before editing.

## Frozen review facts

The supervisor independently verified:

- accepted P1-011 Mercato base: `20887f2d74928cf69f447fdd6af20a612f38387c`,
- current pushed P1-012 branch head at review time: `7273fde8d47686812d099f99a6cdcfa323045826`,
- current `outbound/p1-012` lineage: exactly one commit ahead of the accepted P1-011 base, merge base exactly `20887f2d74928cf69f447fdd6af20a612f38387c`,
- current WMS evidence commit: `226298140163b8c7ca6c32beb34032da21966b60`,
- Scanner remains frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

Continue the existing `Devaxonic-mercato/outbound/p1-012` branch from its actual current head. Do not rebase it, merge unrelated work into it, or recreate it from another base.

## Supervisor review blockers to remediate

### B1 — Supervisor authority is client-spoofable

Current manual-carrier API accepts client-controlled `actorRole` and `isSupervisorApproval`, then the service treats either as proof of Supervisor authority. The route itself only requires authentication plus `wms_outbound.edit`.

This is incompatible with the Architect boundary:

- no automatic match: `Warehouse Supervisor` performs `CARRIER_PENDING -> CARRIER_SELECTED`,
- `Warehouse Supervisor` may always override automatic or manual Carrier Selection without a mandatory reason (`R33`),
- EXTERNAL TU with missing `TUSetup.maxVolume`: Dispatcher may make the manual carrier choice, but approval must come from a real `Warehouse Supervisor` before downstream continuation (`R51`).

#### Required correction

Make actor authority server-authoritative.

- Never trust request-body `actorRole` as authorization or audit truth.
- Never trust request-body `isSupervisorApproval` as proof that the caller is a Supervisor.
- Resolve the authenticated caller's real authorization/role using the existing Mercato auth/feature/role mechanism.
- If an existing server-authoritative role/permission helper exists, reuse it rather than inventing a parallel role model.
- Keep audit actor identity derived from authenticated context.

Required behavior:

1. automatic Carrier Selection remains available only to the already authorized WMS operation boundary;
2. general no-match manual selection (`CARRIER_PENDING -> CARRIER_SELECTED`) is allowed only to a real `Warehouse Supervisor`;
3. any override of `CARRIER_SELECTED` is allowed only to a real `Warehouse Supervisor`; reason remains optional;
4. for the EXTERNAL/missing-`maxVolume` exception:
   - a real Dispatcher may record the manual carrier choice but Shipment must remain `CARRIER_PENDING`,
   - only a real Warehouse Supervisor approval may advance it to `CARRIER_SELECTED`,
   - a caller must not obtain Supervisor behavior by posting role/approval flags;
5. fail closed when caller authority cannot be proven.

Do not widen permissions merely to make tests pass.

### B2 — Evidence identifies the wrong Mercato commit

`05_EVIDENCE/P1-012_EVIDENCE.md` currently records this Mercato final head:

`7273fde8d4ba9ca0a6a237f3792cb1f2fe45963f`

That SHA is not the pushed P1-012 commit. The independently verified pushed branch head before remediation is:

`7273fde8d47686812d099f99a6cdcfa323045826`

After remediation the final head will change again. Evidence must therefore be updated only after the remediation commit(s) are pushed and must contain the exact final pushed Mercato SHA verified from Git, not a copied/constructed value.

### B3 — Testing credential is committed in the Playwright spec

The new `P1-012-carrier-selection-ui.spec.ts` contains a literal Testing login password in repository source.

Required correction:

- remove the literal credential from source,
- use the repository's existing approved environment/secret mechanism for Testing UI credentials,
- do not write credential values into evidence, logs committed to Git, or this guide,
- if the committed value corresponds to a still-valid account credential, treat it as exposed and rotate it using the project's normal secret-management practice; do not record the replacement value in Git.

Do not solve this by replacing one hardcoded password with another.

## Required focused tests

Use the approved Testing PostgreSQL environment (`/etc/mercato-localhost.env`) where DB evidence is required. No SQLite or ad-hoc substitute. No Prod/Demo.

Add/adjust decisive tests proving at minimum:

1. authenticated non-Supervisor with ordinary WMS edit capability cannot move a general no-match Shipment from `CARRIER_PENDING` to `CARRIER_SELECTED`;
2. the same caller cannot override a `CARRIER_SELECTED` Shipment;
3. posting `actorRole = Warehouse Supervisor` does not elevate a non-Supervisor;
4. posting `isSupervisorApproval = true` does not elevate a non-Supervisor;
5. a real Warehouse Supervisor can manually select after no-match;
6. a real Warehouse Supervisor can override an existing selection without a reason;
7. EXTERNAL missing-`maxVolume`: real Dispatcher choice remains `CARRIER_PENDING` awaiting approval;
8. EXTERNAL missing-`maxVolume`: non-Supervisor cannot self-approve by request fields;
9. EXTERNAL missing-`maxVolume`: real Supervisor approval advances to `CARRIER_SELECTED`;
10. audit actor identity/role reflects server-authoritative authenticated context, not spoofed request data;
11. existing P1-012 deterministic selection tests remain green;
12. P1-011 PostgreSQL regression remains green.

Run fresh Playwright proof from the final served remediation revision for the relevant manual-selection/override/EXTERNAL journeys. Do not claim HUMAN VERIFIED.

## Credential hygiene proof

Before closeout, prove the new P1-012 diff no longer contains literal login passwords or newly introduced credentials. Do not print secret values in the evidence artifact.

## Evidence update

Update existing:

`05_EVIDENCE/P1-012_EVIDENCE.md`

Do not mark P1-012 FINAL PASS and do not update progress/steering.

Evidence must include:

- exact accepted P1-011 base,
- exact pre-remediation P1-012 head `7273fde8d47686812d099f99a6cdcfa323045826`,
- exact final remediation Mercato head after push,
- exact lineage/compare proof,
- files changed by remediation,
- server-authoritative authorization design used,
- negative anti-spoof tests and positive Supervisor/Dispatcher tests,
- genuine PostgreSQL evidence where applicable,
- fresh Playwright evidence from the final served revision,
- P1-011 regression result,
- proof Scanner remains frozen at `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`,
- proof no literal Testing credential remains in the P1-012 test source,
- clean git status and explicit exclusions.

Correct the previously wrong Mercato SHA; do not preserve it as the claimed final revision.

## Hard exclusions

No:

- label generation/printing,
- external carrier API,
- manifest work,
- ERP work,
- Scanner changes,
- unrelated refactors,
- Prod/Demo changes,
- `STATE.md` updates,
- handover updates,
- Task Catalog / Plan / traceability changes,
- `Devaxonic-WMS/.ai` steering changes,
- Drive handover changes,
- workflow changes,
- merge/delete of task branches,
- claim of FINAL PASS.

## Two-strikes rule

If the same failure class occurs twice, STOP and report the blocker rather than widening scope.

## STOP boundary

When remediation is complete:

1. push the corrected `Devaxonic-mercato/outbound/p1-012`,
2. update/push the P1-012 evidence artifact,
3. report exact refs,
4. STOP.

Do not merge. Do not update steering. Do not claim acceptance.

Owner-facing response must remain microscopic: `done` on success; otherwise at most 5 lines containing blocker + refs.
