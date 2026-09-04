# P1-015 — owner-authorized final-runtime closeout

**Execution status:** owner-authorized additional closeout for `P1-015`  
**Catalog item:** `18/37`  
**Scope type:** evidence/runtime closeout only; **NO further product change / NOT steering / NOT acceptance**  
**Workflow:** follow current `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

## Mandatory fresh-session startup

Run this closeout in a **fresh Antigravity session**.

```bash
cd /home/ubuntu/git/Devaxonic-WMS
/home/ubuntu/.local/bin/agy-pl
```

Never use bare `agy`.

Before doing anything, sync current Git and read:

1. `Devaxonic-WMS/AGENTS.md`
2. `Devaxonic-WMS/.ai/STATE.md`
3. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`
4. `Devaxonic-WMS/.ai/TESTING.md`
5. `Devaxonic-WMS/.ai/OPERATIONS.md`
6. `WMS_Outbound/STATE.md`
7. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-04.md`
8. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
9. `WMS_Outbound/06_AGENT_GUIDES/P1-015_EXECUTION.md`
10. `WMS_Outbound/06_AGENT_GUIDES/P1-015_REMEDIATION.md`
11. this closeout guide
12. current `WMS_Outbound/05_EVIDENCE/P1-015_EVIDENCE.md`
13. current P1-015 Mercato branch and exact runtime/service configuration

Do not reconstruct mutable state from old chat.

## Owner authorization

Owner explicitly authorized one additional P1-015 closeout limited to:

- serve exact final remediated P1-015 Mercato runtime,
- rerun fresh P1-015 Playwright on that exact runtime,
- correct final evidence completeness,
- no additional product implementation.

This authorization does **not** authorize P1-016, steering changes, acceptance claims, or product redesign.

## Frozen supervisor refs

These exact refs were independently verified immediately before this guide:

- accepted P1-014 Mercato base: `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`
- final remediated P1-015 Mercato head: `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`
- Mercato branch: `outbound/p1-015`
- Mercato lineage: exactly **2 commits** above accepted P1-014 base; merge base exact `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`
- final Git compare: **20 files changed, 4504 insertions, 11 deletions**
- current WMS evidence head before this guide: `c6c0e1c8b7b8dd0904da210fc371a8e2828cfb08`
- Devaxonic-WMS steering: `3f46432cc75899c80be38ec9206d61b9a544f416`
- frozen Scanner: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` on `outbound/p1-009`
- authoritative progress remains `17/37 FINAL PASS`; P1-015 is not accepted yet.

If any of those refs drift unexpectedly before execution, **STOP** and report the actual refs. Do not merge, rebase, squash, transplant or guess around drift.

## Why this closeout exists

The corrective product implementation passed supervisor review:

- close vs Carrier correction now serializes on real PostgreSQL row locks;
- Test 21 proves the real close/correction race with exact A↔B backend PIDs;
- Tests 6, 10 and 18 now bind PostgreSQL lock evidence to the exact operation/session identities;
- P1-015 PostgreSQL suite is 21/21 and regression aggregate is 171/171.

The remaining blocker is evidence provenance: the current evidence says Playwright 6/6 was executed while `mercato-localhost.service` served pre-remediation SHA `71ba80fbe0ab221ff4484b4ad7f6d2256f57d8b8`.

Because remediation changed runtime transaction/locking behavior, final acceptance requires a fresh rendered run against exact final SHA `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`.

## Hard prohibition: no product changes

Do **not** modify any Mercato source, test, migration, schema, route, UI or configuration tracked in Git.

The Mercato branch must remain exactly at:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

No new Mercato commit is expected or authorized.

If the fresh final-runtime Playwright fails because of a genuine product defect, **STOP** and report the failure. Do not patch it in this closeout.

## Required closeout actions

### 1. Serve exact final Mercato SHA

On the Testing VPS, ensure the canonical `mercato-localhost.service` runtime is built/started from exact Git SHA:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

Use the current documented Testing/Operations procedure; do not improvise Prod/Demo changes.

Record literal proof of the served revision using the current canonical mechanism from `.ai/TESTING.md` / `.ai/OPERATIONS.md` plus Git HEAD.

Required fact in evidence:

```text
Served Testing Mercato runtime revision: f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4
```

The canonical URL must remain:

`https://devaxonic-test.info-start.com.pl`

### 2. Fresh P1-015 Playwright — exact final runtime

Rerun the existing committed P1-015 Playwright suite against the canonical Testing URL while exact `f9b0b89c...` is being served.

Expected suite remains **6 real journeys** unless the committed spec itself says otherwise; do not add journeys or edit tests merely to alter count.

All existing journeys A–F must pass:

- A — open manifest + add POSTED Shipment;
- B — invalid/duplicate membership boundary;
- C — close freezes composition + post-close Carrier correction blocked + no label regeneration/reprint;
- D — physical handover;
- E — final warehouse confirmation / exactly-once UI boundary with no P1-016 settlement;
- F — unauthorized role fail-closed.

Record the literal command output and exact final count.

If any journey fails, **STOP**. No product fix is authorized in this closeout.

### 3. Do not rerun unrelated suites unless needed

The already completed post-remediation PostgreSQL/regression evidence may be retained if its provenance remains truthful:

- P1-015 PostgreSQL `21/21`;
- P1-014 `18/18`;
- P1-013 `15/15`;
- P1-012 `14/14`;
- P1-011 `18/18`;
- FND-002 state machine `77/77`;
- FND-002 transaction `8/8`;
- aggregate `171/171`.

Do not claim a rerun if you did not rerun it.

### 4. Correct `P1-015_EVIDENCE.md` from final facts

Update only:

`WMS_Outbound/05_EVIDENCE/P1-015_EVIDENCE.md`

Required corrections/completeness:

- final Mercato SHA `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4` on `outbound/p1-015`;
- exact accepted base `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6`;
- exact lineage: 2 commits, exact merge base;
- final compare stats: **20 files changed, 4504 insertions, 11 deletions**;
- retain literal post-remediation Test 6/10/18/21 exact PID proof;
- retain literal 21/21 and 171/171 outputs as prior post-remediation verification unless rerun;
- replace stale served-runtime claim with exact final served SHA `f9b0b89c...`;
- replace stale Playwright provenance with the **fresh final-runtime** run and literal result;
- exact canonical Testing URL;
- frozen Scanner exact SHA/branch;
- Devaxonic-WMS steering exact ref untouched;
- P1-016 settlement exclusion unchanged;
- no Prod/Demo changes;
- no external carrier API;
- no acceptance claim.

### 5. Final clean-worktree evidence after all pushes

After updating/pushing the evidence commit, record final clean state from the actual pushed heads.

At minimum:

- Mercato `outbound/p1-015` remains exact `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`, clean;
- Scanner remains exact `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`, clean;
- Devaxonic-WMS remains exact `3f46432cc75899c80be38ec9206d61b9a544f416`, clean;
- WMS_Outbound is clean at the new evidence commit that you just pushed.

Do not write the pre-push WMS head as though it were the final post-push head.

## WMS write boundary

The only Git write authorized in this closeout is the corrected:

`05_EVIDENCE/P1-015_EVIDENCE.md`

on `WMS_Outbound/main`, on top of this supervisor-authored closeout guide.

Do not edit:

- `STATE.md`;
- handovers;
- traceability;
- Task Catalog;
- implementation plan;
- workflow;
- Devaxonic-WMS `.ai/*`;
- Mercato product/tests;
- Scanner.

## STOP boundary

When complete:

1. verify Mercato still exact `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4` with no new commit;
2. push corrected `05_EVIDENCE/P1-015_EVIDENCE.md` to `WMS_Outbound/main`;
3. report exact WMS evidence SHA, exact served Mercato SHA, fresh Playwright count and frozen Scanner SHA;
4. STOP.

Do not start P1-016. Do not update steering. Do not claim `FINAL PASS` or `Owner Accepted`.

Owner-facing success response: `done` plus exact refs and fresh final-runtime Playwright count only.