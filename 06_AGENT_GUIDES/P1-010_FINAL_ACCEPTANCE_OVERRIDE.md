# P1-010 — owner-authorized final acceptance override

**Execution status:** owner-authorized narrow override after normal 2/2 STOP  
**Catalog item:** `P1-010` — item **13/37**  
**Session:** continue the **SAME existing Antigravity session only**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

The owner explicitly authorized **one additional narrow shot** after the normal two-strike STOP. This override exists only to close the remaining P1-010 acceptance blockers listed below. It does **not** reopen P1-010 for redesign or later-scope work.

If any decisive rerun still fails after this shot, or the remaining authorization/consolidation behavior cannot be proven truthfully, **STOP** and record the exact remaining blocker. Do not invent another workaround and do not start P1-011.

## 0. Mandatory startup and frozen observed refs

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Refresh the current installed `wms-outbound`, `fetch_me_prompt`, and `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility. Re-read current `STATE.md`, handover, `GIT_PROMPT_WORKFLOW.md`, `P1-010_EXECUTION.md`, `P1-010_REMEDIATION.md`, P1-010 Architect/traceability sources, and current `.ai` testing/operations guidance.

Use `/home/ubuntu/.local/bin/agy-pl`; never use bare `agy` and never replace the existing Antigravity session.

Supervisor-observed refs before this override:

- accepted Mercato P1-009 base: `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`
- current Mercato `outbound/p1-010`: `323eb55a52b60e25cfb461d906603a99837ac5b4`
- accepted/frozen Scanner P1-009: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- current WMS evidence commit before this override: `726300f05d543117d20b68fb090ecbcf473138da`

Verify refs/ancestry before changing anything. Continue only on `outbound/p1-010` as a clean descendant of the accepted P1-009 Mercato base. Scanner remains frozen.

Do **not** update `STATE.md`, handover, task catalog, implementation plan, or `.ai` checkpoint to FINAL PASS. Supervisor owns acceptance.

## 1. Remaining blocker A — Warehouse Supervisor authority must be real

The current P1-010 route correctly derives the normal operator identity from authenticated request context, but the Supervisor approval boundary is still not acceptable because a client body can supply `supervisorId`, and the route can fall back to the current operator as `supervisorId` when a reason is present.

Close this exact gap.

### Required final behavior

1. **Never trust `supervisorId`, supervisor role, or approval authority from an arbitrary client body.**
2. A normal authenticated Packer/operator must not be able to approve their own deviation merely by sending a reason or identifier.
3. Reuse the **existing accepted Warehouse Supervisor authorization/ACL/decision seam** already present in the product. Do not invent a second role system.
4. The server must establish the approving Supervisor from authenticated/authorized server-side context or from the existing accepted two-actor approval mechanism.
5. Persist the real authorized Supervisor actor and reason in the existing audit/decision model.
6. Prove a normal Packer is rejected for a keep-same-TU deviation when WMS suggestion is `REPACK`.
7. Prove a genuinely authorized Warehouse Supervisor can approve the allowed R18/R65 deviation and the persisted decision contains that authenticated Supervisor identity and reason.
8. Preserve R66 exactly: `externalIssuable=false` is non-overridable, including by Supervisor.
9. Do not weaken accepted P1-008/P1-009 issueability behavior.

If the existing product seam requires separate Packer and Supervisor sessions/actions, implement and prove that actual two-actor flow. Do not replace it with a client-provided fake identity.

## 2. Remaining blocker B — rendered consolidation/compatibility proof is still missing

The current five-journey P1-010 Playwright suite covers keep, repack-all, defer/missing, damaged/unexpected, and Supervisor deviation. It still does **not** execute the required packing consolidation behavior through rendered Packer UI.

Close only this missing P1-010 UI surface; do not implement Shipment lifecycle.

### Required final behavior/proof

Through the real rendered Mercato/Packer UI:

1. select a standard `READY_TO_PACK` source and a target Packing TU / compatible consolidation target using the existing P1-010 model;
2. perform a **compatible consolidation** action under current R60 criteria;
3. use fresh independent PostgreSQL reads to prove the resulting target/source content/accounting state and compatibility ownership;
4. attempt an **incompatible** consolidation (at minimum mismatched customer or delivery address; preserve all currently implemented R60 criteria) and prove it is rejected before invalid mixed content is committed;
5. preserve `maxWeight`; do not invent a volume blocking rule;
6. stop at the packing-side one-Shipment compatibility seam — no P1-011 Shipment creation/readiness/lifecycle.

The decisive consolidation actor action must occur through rendered UI. API/DB setup may create deterministic preconditions only.

You may add one focused Playwright journey or split compatible/incompatible into two if required for truthful deterministic proof. Do not claim a browser path that the checked-in spec does not execute.

## 3. Keep the already-correct P1-010 remediation behavior

Do not rewrite already-passing product areas. Preserve:

- authenticated server-side operator identity for packing mutations;
- keep same TU identity and audited sealing;
- repack-all;
- repack-by-SKU and defer-with-zero-shortage;
- explicit missing/recheck recovery without source-location block;
- DAMAGED -> QC + missing-good recovery;
- unexpected/overage -> QC with no shortage;
- source TU completion/accounting gate;
- quantity integrity;
- real transaction rollback and locking protection;
- accepted P1-009 Direct Pack bypass;
- Scanner implementation head unchanged.

Only make product changes necessary for the remaining Supervisor authorization and rendered consolidation blockers plus tests required to prove them.

## 4. Decisive backend verification on final Mercato head

Rerun the final checked-in P1-010 genuine PostgreSQL suite against the approved remote Testing PostgreSQL after all code changes.

The final suite must decisively include and pass:

- unauthorized normal Packer deviation rejected;
- authorized Warehouse Supervisor deviation accepted with persisted authenticated Supervisor actor + reason;
- R66 non-overridable negative;
- compatible consolidation;
- incompatible consolidation rejection;
- all previously accepted P1-010 packing/discrepancy/rollback/concurrency tests.

Do not reconstruct test names/counts. Evidence must use exact final test source and actual rerun output.

## 5. Decisive rendered Playwright verification on final Mercato head

Rerun the full final P1-010 real Mercato Packer Playwright spec against the approved Testing runtime and remote PostgreSQL.

The final browser set must truthfully cover:

1. keep same TU;
2. repack all;
3. repack-by-SKU + defer + explicit missing/recheck;
4. DAMAGED + unexpected/overage QC handling;
5. real Supervisor authorization/deviation behavior, including a normal Packer not being able to self-approve;
6. compatible consolidation plus incompatible compatibility rejection through rendered UI.

Use separate journeys where required. After decisive actions, use fresh independent PostgreSQL reads for persisted facts.

Label only `PLAYWRIGHT VERIFIED`. Never label automated proof HUMAN VERIFIED.

## 6. Mandatory regression reruns and exact evidence

Because previous evidence still omitted exact commands/output for the reported regression matrix, run and record the required matrix from the final refs. At minimum:

### Mercato/backend — final `outbound/p1-010` head

- final P1-010 PostgreSQL suite;
- P1-009 direct-pack PostgreSQL suite;
- P1-008 TU identity/issueability PostgreSQL suite;
- P1-007 targeted PostgreSQL suite where current P1-010 changes overlap it;
- P1-006 targeted PostgreSQL picking suite where current P1-010 changes overlap it;
- full `src/modules/wms_outbound` backend umbrella.

### Mercato rendered UI — final `outbound/p1-010` head

- final P1-010 Packer Playwright suite including consolidation and Supervisor authority cases;
- accepted P1-008 real UI TU identity/issueability Playwright regression if that accepted spec is present in the Mercato surface.

### Scanner — frozen accepted head `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

- accepted P1-009 Scanner direct-pack 4-journey Playwright regression;
- do not modify Scanner product/tests merely to record results.

Run targeted Inbound/shared regressions only if the final authorized diff touches a shared primitive. State explicitly whether it did.

## 7. Evidence exactness contract

Update only the durable P1-010 evidence under:

`WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md`

and add/replace only P1-010 evidence screenshots/artifacts genuinely produced by this final override when needed.

The final evidence must contain, for **every reported acceptance/regression run**:

- exact working directory;
- exact shell command actually executed, including env variable **names** and flags but no secret values;
- exact repo/branch/revision under test;
- exact actual final output summary/counts;
- exact test titles when titles are presented;
- exact Playwright base URL / application URL / runtime provenance and backend/API target without credentials;
- safe remote Testing PostgreSQL identity/provenance proof;
- exact final 40-character Mercato head;
- exact frozen 40-character Scanner head;
- truthful statement of changed-file scope and whether shared primitives changed;
- explicit P1-011/P1-012+ non-goal statement.

Do not present a regression matrix containing PASS rows unless the corresponding exact command and actual output are also recorded in the evidence.

Do not call a pre-evidence WMS commit the final evidence commit. The supervisor will independently verify the evidence commit after push.

## 8. Authorized write boundary

Authorized writes in this override are limited to:

- minimal P1-010 Mercato product/auth/UI code required for real Supervisor authorization and rendered consolidation;
- P1-010 Mercato tests needed to prove those exact fixes;
- `WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md`;
- P1-010 evidence screenshots/artifacts produced by the final rerun.

Do not modify Scanner. Do not modify accepted historical migrations. Do not start P1-011. Do not update durable checkpoint state/handover to FINAL PASS.

After pushing final Mercato implementation/test changes and WMS evidence, **STOP**. Do not ask the owner for logs, SHAs, screenshots, or test output. The owner replies only `done`; the supervisor independently verifies current Git refs and evidence.