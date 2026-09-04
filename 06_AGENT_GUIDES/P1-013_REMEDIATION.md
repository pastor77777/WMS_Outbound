# P1-013 — concurrency/evidence remediation guide

**Execution status:** authorized first corrective shot for `P1-013`  
**Catalog item:** `P1-013` — item `16/37`  
**Executor:** owner-selected executor under the canonical `Devaxonic-WMS` contract  
**Scope type:** remediation execution artifact only; **NOT steering**  
**Workflow:** follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md` from current `WMS_Outbound/main`

## Mandatory startup

Start from the canonical Devaxonic-WMS checkout and current Git state. For Antigravity use the canonical wrapper from current `.ai/OPERATIONS.md`; never bypass current session/bootstrap rules.

Before editing:

1. sync current `WMS_Outbound/main` without overwriting unrelated work,
2. read current `Devaxonic-WMS/AGENTS.md`, `.ai/STATE.md`, current Outbound handover, `.ai/TESTING.md`, `.ai/OPERATIONS.md`,
3. read current `WMS_Outbound/STATE.md`, current handover, `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`, `06_AGENT_GUIDES/P1-013_EXECUTION.md`, this remediation guide and `05_EVIDENCE/P1-013_EVIDENCE.md`,
4. independently verify the exact current branch heads and compare output before changing anything.

## Frozen review refs

Supervisor review found the following current facts:

- accepted P1-012 Mercato base: `5019a20be14549ff8cbbf25af5bc61c56888e9e1`,
- current P1-013 Mercato implementation head: `7f8185c3eccaf1b04fd46027616b0286d4c87fd1` on `outbound/p1-013`, exactly one commit above the accepted base,
- current WMS evidence commit actually present on `main`: `b299cd3850d77afc2c90016a9e2a916884120611`,
- Scanner frozen reference: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

The executor's short report contained a mistyped WMS SHA (`b299cd36...`). Do not propagate that typo; always use the exact Git SHA above or the new exact remediation evidence SHA created by this shot.

## Review blockers to remediate

### B1 — missing genuine concurrency proof

`P1-013_EVIDENCE.md` currently claims `REAL PostgreSQL concurrency` and describes a concurrency-safety test, but the committed P1-013 PostgreSQL suite at `7f8185c3...` contains no genuine concurrency test.

This is a material evidence mismatch and violates the project evidence invariant:

- concurrency claims require independent overlapping operations,
- PostgreSQL-side evidence must prove the overlap/lock behavior,
- sequential calls, a unique-constraint-only test, or prose are not substitutes.

P1-013 label generation can create a duplicate durable artifact if concurrency is mishandled, so this claim must be proved rather than removed.

### B2 — evidence file-stat mismatch

The `Files Modified in Implementation` section in `05_EVIDENCE/P1-013_EVIDENCE.md` does not match the actual Git compare for `5019a20be... -> 7f8185c3...`.

Regenerate all file/change statistics directly from Git for the final remediation head. Do not copy stale worktree/intermediate diff counts.

### B3 — evidence must describe the actual final test inventory

The current evidence text assigns test numbers/names that do not match the committed P1-013 test file. Rewrite the evidence test section from literal final-head test output and actual test names/counts.

Do not claim a test exists unless it is present at the final pushed SHA.

## Mercato remediation branch rule

Continue only on:

`outbound/p1-013`

It must currently contain exact head:

`7f8185c3eccaf1b04fd46027616b0286d4c87fd1`

Create a narrow remediation commit on top of that head. Do not rebase, merge, squash, transplant unrelated work or start P1-014.

If the branch head is no longer exactly the reviewed head, STOP and report the new ref instead of guessing.

## Required genuine PostgreSQL concurrency test

Add one decisive P1-013 concurrency test to the existing genuine PostgreSQL suite.

The test must use the approved Testing PostgreSQL environment and prove concurrency on the **same real Shipment** with two independent database/ORM sessions or equivalent independent transactional contexts.

Minimum proof:

1. fixture starts in `CARRIER_SELECTED` with no label row,
2. operation A begins label generation and obtains/holds the Shipment row lock,
3. operation B begins independently against the same Shipment before A commits and is genuinely blocked/serialized by PostgreSQL,
4. PostgreSQL-side evidence proves the overlap, e.g. `pg_stat_activity` / `pg_blocking_pids(...)` / lock wait evidence or an equivalent decisive server-side observation,
5. release A and let both operations settle,
6. final fresh independent reads prove:
   - Shipment is `LABEL_GENERATED`,
   - exactly one `wms_outbound_shipment_labels` row exists for the Shipment,
   - exactly one `ShipmentLabelGenerated` durable transition/event exists,
   - the second operation is a safe replay/serialized non-regressive result rather than a duplicate artifact,
   - no duplicate print fact is introduced.

A bare `Promise.all` without overlap/lock proof is insufficient.

A direct duplicate insert that merely hits the unique constraint is still useful as a schema test but is **not** the required concurrency proof.

If the current implementation fails this genuine test, make only the minimum P1-013 product fix needed for correct serialized/idempotent behavior, then rerun all required evidence. Do not broaden scope.

## Preserve accepted behavior

Do not regress the already-reviewed P1-013 behavior:

- label generation only from `CARRIER_SELECTED`,
- `OWN_TRANSPORT` skips/rejects label generation,
- local WMS-owned label data only,
- no external carrier API/provider acceptance/rejection lifecycle,
- successful generation -> `LABEL_GENERATED`,
- local print/reprint does not create a new business state,
- EXTERNAL missing-`maxVolume` remains blocked until real Supervisor approval,
- real Supervisor may perform the currently enforceable pre-close carrier correction,
- correction does not automatically regenerate/reprint the existing label,
- non-Supervisor correction remains blocked by server-authoritative RBAC,
- no P1-014 ERP posting,
- no P1-015 CarrierManifest lifecycle,
- no P1-016 settlement,
- Scanner remains untouched.

## Required verification after remediation

Run from the exact final remediation head using approved Testing configuration:

1. final P1-013 genuine PostgreSQL suite including the new concurrency test,
2. full P1-012 PostgreSQL regression suite,
3. full P1-011 PostgreSQL regression suite,
4. FND-002 state-transition invariant suite,
5. fresh P1-013 Playwright journeys from the final served product revision.

If the remediation changes only tests/evidence and not runtime product files, still record the exact final Git head and exact served runtime revision; do not imply the browser served a different product revision than it actually did.

Automated browser evidence remains `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

## Evidence correction

Update only the candidate evidence file:

`05_EVIDENCE/P1-013_EVIDENCE.md`

It must include:

- accepted P1-012 base,
- pre-remediation P1-013 head `7f8185c3eccaf1b04fd46027616b0286d4c87fd1`,
- exact final remediation Mercato SHA,
- exact lineage/compare count and merge base,
- exact Git-derived final file stats,
- exact migration/schema impact,
- literal final P1-013 test count/names including genuine concurrency proof,
- PostgreSQL-side blocking/overlap evidence,
- rollback proof,
- exact P1-012/P1-011/FND-002 regression outputs,
- fresh Playwright result and served revision,
- Scanner frozen reference,
- clean git status,
- explicit exclusions/deferred P1-015 boundary,
- no stale `b299cd36...` SHA anywhere.

Evidence status remains candidate/review material. Do **not** mark `FINAL PASS` or `Owner Accepted`.

## WMS / steering boundary

Allowed WMS change for the executor:

- update `05_EVIDENCE/P1-013_EVIDENCE.md` only.

Do not edit:

- `STATE.md`,
- `08_HANDOVER/*`,
- Task Catalog / implementation plan,
- traceability,
- `GIT_PROMPT_WORKFLOW.md`,
- Devaxonic-WMS `.ai/*`,
- Drive handover or other steering/control files.

## Two-strikes rule

This is the one genuine corrective attempt after the initial P1-013 shot.

If the same material concurrency/evidence failure remains after this remediation, STOP and report it. Do not create another remediation or switch strategy without explicit owner authorization.

## STOP boundary

When complete:

1. push the narrow remediation commit to `Devaxonic-mercato/outbound/p1-013`,
2. push corrected `05_EVIDENCE/P1-013_EVIDENCE.md` to `WMS_Outbound/main`,
3. report exact refs,
4. STOP.

Do not merge task branches. Do not advance to P1-014. Do not update steering. Do not claim FINAL PASS.

Owner-facing response should remain microscopic: `done` on success; otherwise at most 5 lines with blocker + exact refs.
