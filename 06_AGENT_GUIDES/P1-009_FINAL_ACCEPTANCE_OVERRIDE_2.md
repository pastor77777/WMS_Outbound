# P1-009 — final acceptance override 2 — Scanner regression evidence integrity only

**Status:** explicit owner override after the previous evidence-only closeout failed the evidence-integrity gate. This authorizes **one narrow additional shot only**.  
**Item:** P1-009 — Direct Pack declaration and automatic sealing — item 12/37.  
**Mode:** **EVIDENCE ONLY**. No Mercato or Scanner product/test-source changes are authorized.  
**Session:** continue the SAME existing Antigravity session only. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

## 1. Frozen refs — independently verified before this guide

Do not advance, amend, rebase, reset, force-push or otherwise modify these implementation refs:

- Mercato `outbound/p1-009` = `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`.
- Scanner `outbound/p1-009` = `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- WMS_Outbound `main` before this guide = `241421f83db9c1fde838f2ae0e256224d35f5ee8`.

The current product diff has already been supervisor-checked as clean for P1-009. This shot must not touch product code.

## 2. Read before execution

Refresh in the same Antigravity session:

1. `06_AGENT_GUIDES/P1-009_EXECUTION.md`
2. `06_AGENT_GUIDES/P1-009_REMEDIATION.md`
3. `06_AGENT_GUIDES/P1-009_FINAL_ACCEPTANCE_OVERRIDE.md`
4. `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
5. `05_EVIDENCE/P1-009_EVIDENCE.md`
6. current `.ai/TESTING.md` and `.ai/OPERATIONS.md` from Devaxonic-WMS
7. current installed `wms-outbound`, `fetch_me_prompt`, `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility.

## 3. The only remaining blocker

The current `05_EVIDENCE/P1-009_EVIDENCE.md` labels the Scanner regression block as **Exact Output Received**, but two displayed test titles are not verbatim titles from the frozen Scanner specs.

At frozen Scanner head `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`:

- `e2e/p1-007-real-scanner-short-pick.spec.ts` contains the test title:
  `Scanner operator reports short pick, verifies location shortage block & automatic replacement task generation`
- `e2e/p1-008-real-tu-identity.spec.ts` contains the test title:
  `Real Scanner UI Operator Journey: Login -> Select Outbound TU Mode -> Create TU -> Add SKU Content -> View Mass/Volume -> Apply Override -> DB Verification`

The evidence currently shows reconstructed/summarized alternatives. That is not acceptable under the durable evidence contract.

Do **not** simply replace those strings from this guide and call it evidence. Rerun the exact Scanner regression command below and copy the actual terminal output verbatim.

## 4. Required rerun — Scanner regressions only

Use the frozen Scanner head. Verify `git rev-parse HEAD` is exactly:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Do not commit anything in Devaxonic-scanner.

From `/home/ubuntu/git/Devaxonic-scanner`, run the real rendered Scanner regression matrix against the approved Testing runtime/database:

```bash
DATABASE_URL=$(grep ^DATABASE_URL ../Devaxonic-mercato/apps/mercato/.env | cut -d= -f2-) npx playwright test e2e/p1-006-real-scanner-picking.spec.ts e2e/p1-007-real-scanner-short-pick.spec.ts e2e/p1-008-real-tu-identity.spec.ts
```

Requirements:

- real rendered UI;
- no route mocks replacing decisive operator actions;
- same approved Scanner/Mercato runtime used by current accepted evidence;
- real remote Testing PostgreSQL;
- capture the **literal command** and the **literal Playwright terminal output**;
- do not paraphrase, normalize, shorten, rename or reconstruct test titles/timings/counts.

If the command fails, record the failure truthfully in evidence and STOP. Do not alter Scanner/Mercato code or tests.

## 5. Evidence update — tightly bounded

Update only:

`WMS_Outbound/05_EVIDENCE/P1-009_EVIDENCE.md`

Primary required edit: replace the Scanner regression subsection with the exact rerun command/output from this shot.

Before committing, perform an evidence-integrity check:

1. Every line presented under an `Exact Output`, `Exact Output Received`, or equivalent label must be copied from actual command output, not reconstructed prose.
2. Scanner regression test titles shown in evidence must exactly match the rerun output and the checked-in frozen specs.
3. Keep exact 40-character frozen Mercato and Scanner heads unchanged.
4. Keep P1-009 evidence label `PLAYWRIGHT VERIFIED`; do not use `HUMAN VERIFIED` or declare `FINAL PASS`.
5. Do not edit `STATE.md`, handover files, task catalog, requirements, Architect sources or Devaxonic-WMS steering state.
6. Do not alter previously valid backend or P1-009 Playwright result blocks merely to make formatting uniform.

The only expected repository change after this guide is the evidence document itself.

## 6. Final Git discipline

After the evidence correction:

- commit/push only the WMS_Outbound evidence update to `main`;
- Mercato head must remain `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`;
- Scanner head must remain `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- STOP immediately after push;
- do not start P1-010;
- do not update acceptance/checkpoint state yourself.

The owner will reply only `done`; the supervisor will independently fetch current refs, diff and evidence.
