# P1-009 — final acceptance override — evidence-only closeout

**Status:** explicit owner override after two-strikes STOP — one narrow additional shot only  
**Item:** P1-009 — Direct Pack declaration and automatic sealing — item 12/37  
**Mode:** **EVIDENCE ONLY**. No product implementation changes are authorized in this shot.  
**Session:** continue the SAME existing Antigravity session only. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

## 1. Frozen implementation refs — independently verified before this guide

Treat these product heads as frozen for this closeout:

- Mercato `outbound/p1-009` = `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`.
- Scanner `outbound/p1-009` = `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- WMS_Outbound `main` before this guide = `2b6e59d7cd9f329ef7e9171f7147ec8c4acc7cc3`.

Do **not** modify or advance Mercato or Scanner in this shot. Do not amend, rebase, reset or force-push either implementation branch.

If any decisive rerun fails because product behavior is wrong, STOP and record the failure truthfully. This override does not authorize a product fix, a third material remediation, or a new implementation branch.

## 2. Read before execution

Refresh these current sources in the same session:

1. `06_AGENT_GUIDES/P1-009_EXECUTION.md`
2. `06_AGENT_GUIDES/P1-009_REMEDIATION.md`
3. `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
4. `05_EVIDENCE/P1-009_EVIDENCE.md`
5. current `.ai/TESTING.md` and `.ai/OPERATIONS.md` from Devaxonic-WMS
6. current installed `wms-outbound`, `fetch_me_prompt`, `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility.

This shot exists only to make the durable evidence exactly match the already-frozen implementation and actual executed commands/output.

## 3. Remaining acceptance blockers to close

### A. Test-name integrity mismatch

The current durable evidence still reports a reconstructed P1-009 backend test list that does not match the checked-in source.

In particular, checked-in Mercato P1-009 test `4C` at the frozen head is:

`4C: Operator issueability override (R65) allows direct pack TU below thresholds to automatically seal on close`

The evidence currently describes `4C` as an OutboundOrder completion test. This is not acceptable.

**Required:** regenerate the P1-009 backend result section from the actual rerun. Copy the exact executed Jest test names / summary as actually emitted. Do not rename, summarize, reorder or reconstruct individual test titles from memory.

### B. Exact commands are missing

The remediation explicitly required exact commands for every reported backend and Playwright run. The current evidence records result summaries but not the verbatim commands used.

**Required:** for every acceptance/regression command executed in this closeout, record:

- working directory;
- exact shell command as executed, including relevant environment-variable names/flags but **never secret values**;
- exact target revision/branch checked out;
- exact pass/fail counts and final summary lines from that run.

Do not replace a real command with a normalized pseudo-command after the fact.

### C. P1-008 Scanner regression was omitted

Supervisor independently verified that the frozen Scanner head contains the accepted real spec:

`e2e/p1-008-real-tu-identity.spec.ts`

Therefore this closeout must run and record it. Do not state that no separate P1-008 Scanner spec exists.

## 4. Mandatory clean-run matrix for this override

All runs below must execute against the frozen heads above. Because the prior evidence document is inconsistent, rerun the decisive matrix rather than merely editing prose.

### 4.1 Mercato / backend — real Testing PostgreSQL

Run and record exact commands + exact outputs for:

1. `p1-009-postgres.integration.test.ts` — decisive P1-009 suite.
2. `p1-006-postgres.integration.test.ts` — regression.
3. `p1-007-postgres.integration.test.ts` — regression.
4. `p1-008-postgres.integration.test.ts` — regression.
5. full `src/modules/wms_outbound` umbrella supported by the current harness.

For P1-009, evidence must include exact test names/counts from the actual executed run and must match the checked-in test source. Do not preserve the stale 4C wording.

### 4.2 Real PostgreSQL identity/provenance

For the decisive P1-009 backend run, add a small non-secret identity proof from the same approved Testing DB environment. Record the exact command/query and output for a safe fingerprint such as:

- current database name;
- PostgreSQL server address/port where available;
- PostgreSQL version.

Do not print `DATABASE_URL`, passwords, tokens, connection strings or other secrets. Refer to the environment variable by name only.

The evidence must make clear that the suite used the approved remote Testing PostgreSQL and not a local/in-memory/fake database.

### 4.3 Scanner — real rendered Playwright

At frozen Scanner head `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`, run and record exact commands + exact outputs for:

1. `e2e/p1-009-real-scanner-direct-pack.spec.ts` — expected four rendered journeys if green:
   - happy path direct pack;
   - same-TU multi-zone no re-prompt;
   - R67 PICK_FULL replacement-TU inheritance no re-prompt;
   - negative non-issueable direct-pack path.
2. `e2e/p1-006-real-scanner-picking.spec.ts`.
3. `e2e/p1-007-real-scanner-short-pick.spec.ts`.
4. `e2e/p1-008-real-tu-identity.spec.ts`.

Record the actual Playwright base URL/runtime provenance used in the final run, including the Scanner URL/port and backend/API target without exposing credentials.

Decisive intended Picker actions must remain real rendered UI actions. DB/API setup is allowed only for deterministic fixtures/persistence verification. Label browser proof only as `PLAYWRIGHT VERIFIED`.

## 5. Evidence rewrite rules

Update only:

`WMS_Outbound/05_EVIDENCE/P1-009_EVIDENCE.md`

The rewritten file must be based on the fresh closeout reruns above and must contain:

1. Frozen exact 40-char implementation heads:
   - Mercato `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`;
   - Scanner `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
2. A clear note that the WMS evidence commit is created *after* file content is written and will be independently verified by the supervisor; do not claim a pre-evidence WMS SHA is the final evidence commit.
3. For every rerun:
   - repo + working directory;
   - exact command;
   - target head;
   - exact pass/fail counts;
   - relevant exact final output lines.
4. P1-009 backend test titles exactly as executed. In particular, test 4C must reflect the actual R65 override test if that is what the frozen source/output contains.
5. Real Testing PostgreSQL fingerprint/provenance without secrets.
6. Scanner P1-009 4-journey Playwright provenance and exact result.
7. P1-006, P1-007 **and P1-008** Scanner regression commands/results.
8. P1-006/P1-007/P1-008 backend regressions and full Outbound umbrella command/results.
9. No `HUMAN VERIFIED`; use `PLAYWRIGHT VERIFIED` for automated browser evidence and truthful DB classification.
10. No invented timestamps, counts, test names, commands or SHAs.

If a run fails, preserve the failure truthfully; do not edit the evidence into PASS.

## 6. Scope discipline

Authorized writes in this shot:

- `WMS_Outbound/05_EVIDENCE/P1-009_EVIDENCE.md` only.

Do **not** change:

- Mercato source/tests/migrations/config;
- Scanner source/tests/config;
- WMS `STATE.md`;
- handovers;
- implementation plan/task catalog;
- P1-010 materials.

The supervisor owns acceptance and durable state advancement after independently verifying the new evidence commit.

## 7. STOP condition

After the clean reruns are complete and the corrected evidence file is committed/pushed to WMS_Outbound `main`:

**STOP.**

Do not start P1-010. Do not ask the owner to paste logs, SHAs or screenshots. The owner will reply only `done`; the supervisor will independently fetch refs, the evidence commit, exact evidence content and frozen implementation lineage.

This owner override authorizes **this one narrow evidence-only closeout shot only**.