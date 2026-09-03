# P1-010 — evidence rerun only

**Execution status:** owner-authorized evidence-only closeout shot  
**Catalog item:** `P1-010` — item **13/37**  
**Session:** continue the **SAME existing Antigravity session only**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

This shot is intentionally mechanical. Product code is frozen. The only remaining acceptance blocker is that the durable P1-010 evidence currently re-labels older test output with the new final Mercato SHA instead of recording fresh reruns executed on that exact final head.

If any required rerun fails, **STOP and record the actual failure**. Do not modify Mercato or Scanner to make a rerun pass. Do not start P1-011.

## 0. Mandatory startup and frozen refs

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Refresh current installed `wms-outbound`, `fetch_me_prompt`, and `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility. Re-read current `STATE.md`, current handover, `GIT_PROMPT_WORKFLOW.md`, `P1-010_FINAL_ACCEPTANCE_OVERRIDE_2.md`, current `05_EVIDENCE/P1-010_EVIDENCE.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, and the current real-evidence contract.

Use `/home/ubuntu/.local/bin/agy-pl`; never bare `agy` and never replace the existing Antigravity session.

Supervisor-observed frozen refs before this evidence-only shot:

- final Mercato `outbound/p1-010`: `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6`
- frozen Scanner `outbound/p1-009`: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- WMS evidence head before this guide: `63218bfd80b4cfce37705f5b2a7bc155f506ba02`

Before running tests, independently verify:

- Mercato branch/head is exactly `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6`;
- Scanner branch/head is exactly `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- both product repositories are clean;
- no product commit is created during this shot.

If a product head differs, STOP. Do not rebase, cherry-pick, amend, or create a replacement implementation commit.

## 1. Absolute write boundary

Allowed writes:

- `WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md` only;
- P1-010 evidence screenshots only if the fresh Playwright rerun genuinely regenerates them and the existing test writes them.

Forbidden writes:

- any Mercato source/test/config file;
- any Scanner source/test/config file;
- any migration;
- `STATE.md`, handover, `.ai` checkpoint, Task Catalog, implementation plan;
- P1-011 or later scope.

This is **not** a remediation shot. It is evidence capture only.

## 2. Fresh-output rule — critical

Do not reuse, copy, preserve, or cosmetically re-label any prior timing/output block for a rerun performed in this shot.

For every command below:

1. execute it now on the frozen required head;
2. capture the complete fresh stdout/stderr for that invocation;
3. preserve the exit status (`set -o pipefail` when using `tee`);
4. put the actual new command and actual new output summary into `P1-010_EVIDENCE.md`;
5. if the command fails, evidence must say FAIL with the actual output and then STOP.

Identical counts are expected; identical wall-clock timings are possible but must come from the new run, not from old text.

Use fresh capture files under `/tmp/p1-010-final-rerun-*` so the source of each evidence block is unambiguous.

## 3. Runtime / PostgreSQL provenance — rerun first

From `/home/ubuntu/git/Devaxonic-mercato/apps/mercato`, while checked out exactly at `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6`:

- record `pwd`;
- record `git rev-parse HEAD`;
- record `git status --short`;
- record the safe Testing PostgreSQL identity query used by the current evidence;
- record the real application/Playwright base URL and backend/API target used by this run without credentials/secrets.

Do not assert remote Testing provenance from an older run.

## 4. Fresh Mercato backend reruns on final head

Working directory:

`/home/ubuntu/git/Devaxonic-mercato/apps/mercato`

Run each command separately with fresh output capture. These exact checked-in test files exist on the frozen final head.

### A. P1-010 decisive suite

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 yarn test src/modules/wms_outbound/services/__tests__/p1-010-postgres.integration.test.ts 2>&1 | tee /tmp/p1-010-final-rerun-p1-010-backend.txt'
```

The fresh run must include the final strict Supervisor ACL matrix in test 13, plus existing packing/consolidation/rollback/concurrency coverage.

### B. P1-009 regression

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 yarn test src/modules/wms_outbound/services/__tests__/p1-009-postgres.integration.test.ts 2>&1 | tee /tmp/p1-010-final-rerun-p1-009-backend.txt'
```

### C. P1-008 regression

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 yarn test src/modules/wms_outbound/services/__tests__/p1-008-postgres.integration.test.ts 2>&1 | tee /tmp/p1-010-final-rerun-p1-008-backend.txt'
```

### D. P1-007 regression

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 yarn test src/modules/wms_outbound/services/__tests__/p1-007-postgres.integration.test.ts 2>&1 | tee /tmp/p1-010-final-rerun-p1-007-backend.txt'
```

### E. P1-006 regression

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 yarn test src/modules/wms_outbound/services/__tests__/p1-006-postgres.integration.test.ts 2>&1 | tee /tmp/p1-010-final-rerun-p1-006-backend.txt'
```

### F. Full Outbound backend umbrella

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 yarn test src/modules/wms_outbound 2>&1 | tee /tmp/p1-010-final-rerun-outbound-umbrella.txt'
```

Do not summarize any row as PASS unless its corresponding fresh capture exists and the command exited successfully.

## 5. Fresh Mercato rendered UI rerun on final head

Still from `/home/ubuntu/git/Devaxonic-mercato/apps/mercato`, run the final checked-in P1-010 Playwright spec exactly on Mercato head `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6`:

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 npx playwright test src/modules/wms_outbound/__integration__/P1-010-packer-workstation-ui.spec.ts 2>&1 | tee /tmp/p1-010-final-rerun-playwright.txt'
```

The checked-in spec currently contains six journeys. The fresh run must exercise the actual current source, including:

- Journey 5 normal-Packer rejection followed by genuine authorized Supervisor session/approval;
- Journey 6 rendered compatible/incompatible consolidation.

Record the actual fresh journey titles, counts, timing summary, base URL/application runtime provenance, and any screenshots genuinely regenerated by this run.

Label automated browser proof only `PLAYWRIGHT VERIFIED`.

There is no P1-008 Mercato UI spec in the current `__integration__` directory on this final head, so do not invent one for this evidence-only shot.

## 6. Fresh frozen Scanner regression

Scanner must remain exactly at:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

From `/home/ubuntu/git/Devaxonic-scanner`:

- record `pwd`;
- record `git rev-parse HEAD`;
- record `git status --short`;
- run the existing accepted Scanner P1-009 direct-pack Playwright suite with fresh capture:

```bash
bash -lc 'set -o pipefail; NODE_TLS_REJECT_UNAUTHORIZED=0 npx playwright test e2e/p1-009-real-scanner-direct-pack.spec.ts 2>&1 | tee /tmp/p1-010-final-rerun-scanner-p1-009.txt'
```

Do not change Scanner for any reason in this shot.

## 7. Rewrite durable evidence from the fresh captures

Update only:

`WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md`

Required final evidence content:

- final Mercato full 40-char head: `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6`;
- frozen Scanner full 40-char head: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- this guide/pre-evidence WMS ref using a full 40-char SHA after syncing it locally;
- safe fresh PostgreSQL identity/provenance from this shot;
- exact fresh working directory + command + actual output for **every** regression row reported;
- exact fresh P1-010 backend test titles as emitted by this run if titles are included;
- exact fresh P1-010 Playwright journey titles/counts as emitted by this run;
- exact Scanner command/output from this rerun;
- exact full umbrella command/output from this rerun;
- truthful statement that Mercato and Scanner product refs were frozen and no product files were modified in this evidence-only shot;
- truthful shared-primitive statement;
- P1-011/P1-012+ non-goal statement;
- evidence label `PLAYWRIGHT VERIFIED`, never HUMAN VERIFIED and never FINAL PASS.

### Remove stale evidence

Delete or replace every old timing/output block that did not come from this shot. Do not leave old values next to new values in a way that makes provenance ambiguous.

The regression summary table must be generated from the new captured outputs. No PASS row may exist without a matching exact fresh command/output section in the same evidence document.

## 8. Git and stop rule

After updating evidence:

1. verify Mercato still equals `d2a703e4103a2e1ea7fdfe1b1ce51ee43ae9afb6` and is clean;
2. verify Scanner still equals `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` and is clean;
3. commit/push only the WMS evidence update (plus genuinely regenerated P1-010 screenshots only if changed by the fresh run);
4. do **not** update `STATE.md` or handover;
5. STOP.

Do not self-declare P1-010 FINAL PASS. The supervisor independently fetches the new evidence commit and decides acceptance after the owner replies only `done`.
