# P1-010 — corrective remediation

**Execution status:** corrective implementation/evidence shot — **2/2**  
**Catalog item:** `P1-010` — item **13/37**  
**Session:** continue the **SAME existing Antigravity session only**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

This is the one normal corrective retry permitted after the first failed P1-010 shot. If the same material implementation/evidence path is still not proven after this shot, **STOP**. Do not invent a third workaround without explicit owner override.

## 0. Mandatory startup and frozen refs

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Refresh current installed `wms-outbound`, `fetch_me_prompt`, `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility. Re-read current `STATE.md`, handover, `GIT_PROMPT_WORKFLOW.md`, this guide, P1-010 Task Catalog/Architect/traceability sources, and current `.ai` state/testing/operations.

Use `/home/ubuntu/.local/bin/agy-pl`; never bare `agy` and never replace the existing Antigravity session.

Supervisor-observed first-shot refs:

- accepted Mercato P1-009 base: `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`
- first-shot Mercato `outbound/p1-010`: `ed8627da76c367c448106764f8e864b8d2e84191`
- accepted/frozen Scanner P1-009: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- first-shot WMS evidence commit: `9ca5c64b0c5b536a987bedf15c527e7e0919b772`

Verify refs/ancestry before work. Continue `outbound/p1-010` as a clean descendant of the accepted P1-009 base. Scanner stays frozen unless a newly proven product requirement genuinely forces a Scanner code change; do not create one merely for evidence.

Do **not** update `STATE.md`, handover, or `.ai` checkpoint to FINAL PASS. Supervisor owns acceptance.

## 1. Why shot 1 failed

The first shot is **not accepted**. Correct all of the following, not only the Markdown evidence.

### A. Durable evidence is not exact

Current evidence records Mercato head `ed8627da7bfa2b3a165fc3521b34a6e138a0c201`, while the actual pushed branch head observed by the supervisor is `ed8627da76c367c448106764f8e864b8d2e84191`.

The evidence claims a 16/16 backend suite but its displayed grouped output/titles do not match the checked-in numbered P1-010 source and even enumerate 17 checks. Replace invented/reconstructed output with the **actual rerun output** from the final checked-in test source.

The evidence also omits exact commands/output for required regressions, the full Outbound umbrella, and the accepted Scanner P1-009 regression. Every reported pass must have the exact command actually executed and actual count/output.

### B. Decisive browser proof is incomplete

The checked-in P1-010 Playwright spec currently has only three journeys. The original execution guide required decisive rendered-UI proof for at least these behavior groups:

1. keep same TU;
2. repack all;
3. repack-by-SKU with operator-selected SKU/quantity, a low count deferred with **no shortage**, then explicit missing/recheck confirmation;
4. DAMAGED **and** unexpected/overage QC handling with their different shortage effects;
5. consolidation plus decisive compatibility/authorization negative behavior.

Current Journey 3 is titled as if it proves damage routing, but the checked-in actions only execute the missing/recheck path. Do not claim a UI action that the browser did not perform.

Expand the rendered Packer journey set so the decisive actions above really happen through UI. Split journeys when needed for deterministic proof. After each decisive action use fresh independent PostgreSQL reads for the persisted facts being claimed.

### C. Actor / Supervisor authorization is not trustworthy yet

Current Packer UI sends placeholder actor/supervisor values (for example a synthetic packer actor and synthetic supervisor id), and current API behavior permits a client body `operatorId` to override the authenticated user. That cannot be the authority/audit model for P1-010.

Fix the P1-010 packing mutation boundary so:

- server-side actor identity comes from authenticated request context / the existing accepted auth seam, not an arbitrary client-supplied actor id;
- a normal Packer cannot self-assert a Warehouse Supervisor identity by posting a string;
- deviation from the WMS suggestion reuses the accepted R18 Warehouse Supervisor authorization/audit seam and persists the real authorized actor + reason;
- `externalIssuable=false` remains non-overridable under R66;
- any below-threshold exception/deviation path stays consistent with accepted R65 and P1-008/P1-009 issueability behavior.

Inspect existing accepted supervisor decision/authorization surfaces before changing code. Do not invent a second approval system.

### D. The current “Supervisor deviation” test is not decisive

The checked-in test marked as the supervisor-deviation case starts from the default issueable fixture and merely toggles `isOverrideApplied`; it does not prove a real WMS-suggestion deviation and does not decisively assert a persisted valid supervisor decision.

Replace/strengthen it with a genuine case: externally issuable TU that does **not** meet normal issue threshold / produces the non-keep suggestion, then prove unauthorized deviation is rejected and valid Warehouse Supervisor authorization allows the intended exception with persisted actor/reason audit. Also prove `externalIssuable=false` cannot be bypassed.

## 2. Functional remediation boundary

Keep all already-correct P1-010 behavior, but close missing acceptance paths rather than rewriting the module.

Required final behavior still includes:

- normal `READY_TO_PACK` Packer evaluation;
- keep same TU preserving identity and audited `READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED`;
- repack-all;
- repack-by-SKU with operator-controlled SKU order and quantity;
- defer low count without shortage;
- explicit missing only after recheck + confirmation, using accepted SHORT_PICKED recovery and no source-location block;
- DAMAGED -> QC + missing-good recovery, no source-location block;
- unexpected/overage -> QC and **no shortage**;
- source TU `REPACKED` only after every expected quantity is accounted packed/QC/confirmed-missing;
- externally issuable target enforcement;
- quantity integrity/no double settlement;
- compatible consolidation only under current R60 criteria and packing maxWeight; incompatible consolidation rejected; content volume alone is not a blocking limit;
- real R18 Warehouse Supervisor authorization for deviation from system suggestion;
- accepted P1-009 Direct Pack path remains untouched as an upstream bypass.

Do not implement P1-011 Shipment lifecycle, carrier selection, labels, ERP/manifest, dispatch, or P1-012+ scope.

## 3. Required real PostgreSQL matrix

Rerun the P1-010 genuine PostgreSQL suite from the **final** remediation head and make checked-in tests truthfully cover all original 16 acceptance points. Preserve the existing 1–16 source naming if practical so evidence and source are easy to compare.

At minimum verify:

1. keep-same identity/role/transition chain;
2. non-issuable target rejection;
3. source completion gate for repack-all;
4. operator-controlled repack-by-SKU;
5. defer low count with zero shortage;
6. explicit missing requires recheck/confirmation and no location block;
7. DAMAGED persistence/QC/recovery/no location block;
8. unexpected/overage QC with zero shortage;
9. source `REPACKED` only after complete accounting;
10. compatible R60 consolidation and maxWeight;
11. incompatible consolidation rejection;
12. content volume alone not a blocker;
13. **real** R18 suggestion-deviation authorization + persisted actor/reason, including unauthorized rejection;
14. overpacking/double-settlement prevention;
15. real rollback across a write boundary + fresh independent read;
16. real overlapping PostgreSQL concurrency/locking with DB-side participant evidence.

If remediation adds a separate test for an important negative (for example R66 non-overridable vs R65 supervisor-authorized deviation), report the new actual total honestly. Never massage source/output to a predetermined count.

## 4. Required real Playwright journeys

Run against the approved Testing runtime + remote PostgreSQL. The final checked-in P1-010 Packer Playwright set must prove through rendered UI:

- **Journey A — Keep Same TU:** operator selects standard `READY_TO_PACK` TU, reviews suggestion, confirms keep, same TU becomes sealed `PackUnit`; fresh DB confirmation.
- **Journey B — Repack All:** operator performs repack-all into a valid external target; source completion gate and final source/target facts confirmed.
- **Journey C — Repack by SKU / defer / missing:** operator chooses SKU/quantity; first low count/defer produces zero shortage; later explicit missing requires recheck + confirmation and persists SHORT_PICKED recovery/no source-location block.
- **Journey D — DAMAGED + unexpected/overage:** perform the actual rendered damage action and actual rendered unexpected/overage action; fresh DB proves QC facts and proves only DAMAGED affects missing-good recovery while unexpected/overage creates no shortage.
- **Journey E — Consolidation + authorization/negative:** prove a compatible consolidation operation and a decisive rejection for incompatible consolidation; also prove the real Supervisor authorization UI/flow for a WMS-suggestion deviation (or split into Journey F if cleaner).

Do not rename a journey to imply actions that were not performed. API/DB fixture setup may create preconditions only; the decisive actor actions must be browser actions.

Automated browser proof label remains `PLAYWRIGHT VERIFIED` only.

## 5. Regression matrix — execute and record exact commands

After final product changes, run and record exact commands + actual output/counts for:

- P1-010 PostgreSQL suite;
- P1-010 Mercato/Packer Playwright suite;
- P1-008 backend TU identity/issueability coverage;
- accepted P1-008 real UI TU identity/issueability Playwright spec if it exists on the accepted current surface; if no separate accepted spec exists, state exactly what accepted surface is reused and why — do not invent a pass;
- P1-009 backend direct-pack suite (accepted baseline was 15/15 before this ticket);
- accepted P1-009 Scanner 4-journey Playwright regression from frozen Scanner head/runtime;
- P1-006/P1-007 targeted backend regressions overlapping changed seams;
- full `src/modules/wms_outbound` backend umbrella;
- targeted Inbound/shared regressions only if this remediation actually changes a shared primitive.

If a prior accepted regression source must change only because a new additive migration changes an expected migration count, keep that edit minimal and explain it explicitly in evidence. Do not weaken assertions to make a failure disappear.

## 6. Durable evidence contract

Replace/update `WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md` only from final, actually executed results.

It must contain:

- exact final 40-character Mercato P1-010 head;
- exact final Scanner head and explicit statement that it remained unchanged if true;
- accepted bases + clean ancestry/diff scope;
- safe Testing PostgreSQL identity/provenance, no credentials/secrets;
- exact runtime/base URLs/provenance for Mercato/Packer and Scanner;
- exact working directories;
- **exact commands actually executed** for every reported backend/Playwright suite;
- exact actual counts and exact test titles matching final checked-in source/output;
- real rollback and concurrency details;
- exact rendered UI journeys and fresh persisted facts each proves;
- explicit shared-primitive/Inbound-regression decision;
- explicit statement that P1-011/P1-012+ was not implemented.

Do not write fake command output, reconstructed durations, invented SHAs, or claimed browser actions that are absent from the final spec. Do not label `HUMAN VERIFIED` or `FINAL PASS`.

## 7. Push and STOP

Push the final Mercato `outbound/p1-010` remediation commit(s) and the durable P1-010 evidence update to `WMS_Outbound/main`.

Leave Scanner unchanged unless a genuine authorized product change was required and justified.

Then **STOP**. Do not start P1-011 and do not update acceptance state. The supervisor will independently verify Git refs, source/evidence exactness, scope, and lineage after the owner replies `done`.