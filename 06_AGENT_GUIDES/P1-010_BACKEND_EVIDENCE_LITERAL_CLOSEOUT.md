# P1-010 — backend evidence literal closeout

**Execution status:** owner-authorized evidence-only micro-shot  
**Catalog item:** `P1-010` — item **13/37**  
**Session:** continue the **SAME existing Antigravity session only**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

This shot has **zero product scope**. The P1-010 product implementation is frozen. The only remaining blocker is durable evidence exactness for the final P1-010 PostgreSQL backend run plus one inaccurate prose statement about the foreign-organization rejection message.

If the fresh backend rerun fails, record the real failure and **STOP**. Do not modify Mercato or Scanner. Do not start P1-011.

## 0. Mandatory startup and frozen refs

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Refresh the current installed `wms-outbound`, `fetch_me_prompt`, and `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility. Re-read current `STATE.md`, current handover, `GIT_PROMPT_WORKFLOW.md`, `P1-010_SUPERVISOR_ORG_BOUNDARY_CLOSEOUT.md`, current `05_EVIDENCE/P1-010_EVIDENCE.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, and the current real-evidence contract.

Use `/home/ubuntu/.local/bin/agy-pl`; never bare `agy` and never replace the existing Antigravity session.

Supervisor-observed frozen refs before this evidence-only micro-shot:

- final Mercato `outbound/p1-010`: `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`
- frozen Scanner `outbound/p1-009`: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- current WMS evidence head before this guide: `bcccffe9d4ee1953af1d6add2a75d14dd931004d`
- durable accepted progress remains **12/37 FINAL PASS** until supervisor acceptance.

Before running anything, verify:

- Mercato branch/head is exactly `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`;
- Scanner branch/head is exactly `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- both product repositories are clean;
- no product commit is created during this shot.

If either product ref differs, STOP.

## 1. Absolute write boundary

Allowed write:

- `WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md` only.

Forbidden:

- any Mercato file;
- any Scanner file;
- screenshots;
- `STATE.md`;
- handover files;
- `.ai` acceptance state;
- task catalog / implementation plan;
- P1-011 or later scope.

## 2. Fresh literal backend rerun

From exactly:

`/home/ubuntu/git/Devaxonic-mercato/apps/mercato`

run the final P1-010 backend suite on the frozen final Mercato head and capture the **literal fresh stdout/stderr** using `tee` (or an equivalent lossless local capture):

```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 yarn test src/modules/wms_outbound/services/__tests__/p1-010-postgres.integration.test.ts 2>&1 | tee /tmp/p1-010-final-literal-backend.log
```

Do not manually reconstruct per-test durations. Do not copy per-test duration lines from any older evidence block. Do not normalize, estimate, or rewrite individual test timing values.

The durable evidence must reproduce the actual captured test output exactly enough to prove the final run, including:

- PASS line;
- current suite/test title lines emitted by Jest;
- each test line and its actual runtime if Jest emits it;
- suite/test counts;
- snapshots count;
- total time;
- final `Ran all test suites matching ...` line;
- fresh PostgreSQL contention PIDs/wait event emitted by the suite if present.

If the captured output does not contain individual test durations, do not invent them.

## 3. Correct the inaccurate foreign-organization error prose

The current evidence contains an inaccurate statement claiming that the final implementation returns a special foreign-organization message such as:

`User <id> is not authorized ... in organization <orgId> (must belong to the same organization).`

Do **not** change product code to make that prose true.

Correct the evidence to match the checked-in implementation:

- `checkSupervisorAuthority` rejects the candidate because `user.organizationId !== scope.organizationId` before the RBAC feature check;
- `packKeepSameTu` then exposes the existing generic unauthorized Warehouse Supervisor error for a failed authority check;
- the decisive proof of the organization boundary is the server-side equality check plus the genuine PostgreSQL negative test and independent DB assertions, not a special response string.

Do not quote an error message unless it is actually emitted by the final code/run.

## 4. Preserve already-valid final-head evidence

Because this shot changes **no product code**, the already-fresh final-head evidence from the immediately preceding shot remains valid for:

- P1-010 Playwright 6/6;
- P1-009/P1-008/P1-007/P1-006 backend regressions;
- Scanner P1-009 Playwright 4/4;
- full `src/modules/wms_outbound` umbrella 276/276;
- runtime/base URL provenance;
- rendered consolidation proof;
- Supervisor same-tenant/same-org positive path;
- same-tenant/different-org negative implementation/test source.

Do not rerun or alter those sections unless needed to remove a contradiction created by the two corrections above. Do not replace their fresh output with older values.

## 5. Evidence exactness rules

Update `05_EVIDENCE/P1-010_EVIDENCE.md` only after the fresh backend run finishes.

Mandatory final state:

- full 40-character Mercato final ref: `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`;
- full 40-character Scanner frozen ref: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- full 40-character WMS pre-evidence/guide ref from this shot;
- exact working directory and command for the rerun;
- literal fresh P1-010 backend output from `/tmp/p1-010-final-literal-backend.log` (or equivalent captured file);
- fresh contention PID values if emitted;
- accurate Supervisor organization-boundary prose matching the checked-in code;
- evidence label remains `PLAYWRIGHT VERIFIED` only;
- no `HUMAN VERIFIED`, no `FINAL PASS` self-declaration;
- no guessed final WMS commit SHA.

If anything conflicts with the final checked-in source, source truth wins and evidence must be corrected.

## 6. Stop boundary

Commit and push the evidence-only WMS change, then **STOP**.

Do not modify product repositories. Do not advance `STATE.md` or handover. The owner replies only `done`; the supervisor independently verifies refs, diff scope, literal output replacement, and evidence/source consistency.