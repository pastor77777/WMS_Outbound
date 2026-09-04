# P1-013 — owner-authorized narrow closeout

**Execution status:** owner-authorized additional closeout shot for `P1-013`  
**Catalog item:** `P1-013` — item `16/37`  
**Authorization boundary:** ONLY exact A↔B PostgreSQL PID lock proof + evidence completeness  
**Scope type:** execution artifact only; **NOT steering**  
**Workflow:** follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md` from current `WMS_Outbound/main`

## Frozen reviewed refs

Before changing anything, verify these exact heads still match Git:

- accepted P1-012 Mercato base: `5019a20be14549ff8cbbf25af5bc61c56888e9e1`,
- current P1-013 Mercato head: `ebc556edd2366a4bb45351924ae2f22a1cafb093` on `outbound/p1-013`,
- current WMS evidence head: `834cda293e72d57551a79e2832d3ca4bfce9363f` on `WMS_Outbound/main`,
- Scanner frozen reference: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

If either Mercato or WMS head differs, STOP and report the actual exact SHA. Do not rebase, merge, transplant, or guess.

## Owner override

The owner explicitly authorized one additional narrow P1-013 closeout after the normal two-strikes stop.

This override permits ONLY:

1. strengthen the existing P1-013 genuine PostgreSQL concurrency test so the observed blocking relationship is tied to the exact backend PIDs of operation A and operation B,
2. update `05_EVIDENCE/P1-013_EVIDENCE.md` so the final evidence contains the exact served runtime revision and clean git-status proof in addition to the corrected PID-bound concurrency proof.

No other remediation, implementation expansion, cleanup, refactor, or next-item work is authorized.

## Allowed Mercato change

Only this file may be changed unless a syntax/import adjustment in the same test file is required:

`apps/mercato/src/modules/wms_outbound/services/__tests__/p1-013-label-generation-postgres.integration.test.ts`

Do not change runtime/product implementation files.

## Required exact A↔B PostgreSQL lock proof

Strengthen test 15 so it proves the lock relationship belongs to the two operations under test, not to an unrelated lock elsewhere in the shared Testing database.

Minimum required structure:

1. Use two independent transactional contexts for the same Shipment: operation A and operation B.
2. Inside operation A, obtain and retain the actual PostgreSQL backend PID from the same transaction/session that will call `generateLabel`, e.g. `SELECT pg_backend_pid()` executed through that exact transaction connection. Call it `pidA`.
3. A invokes `generateLabel` for the fixture Shipment, obtains/holds the Shipment row lock, and deliberately keeps its outer transaction open before commit.
4. Inside operation B, obtain the actual PostgreSQL backend PID from the same transaction/session that will call `generateLabel`. Call it `pidB` **before** B reaches the blocking call.
5. B then invokes `generateLabel` against the same Shipment and becomes blocked by A.
6. An independent observer must query specifically for `pidB`, not for an arbitrary blocked process. The proof must show all of:
   - observed `pid == pidB`,
   - `wait_event_type == 'Lock'`,
   - `pg_blocking_pids(pidB)` contains `pidA`,
   - `pidA != pidB`.
7. Persist/log the exact `pidA`, `pidB`, blocking PID list, and wait event type in the literal test output used for evidence.
8. Release A; both operations settle.
9. Fresh independent reads must again prove exactly:
   - Shipment `LABEL_GENERATED`,
   - exactly one label row for the Shipment,
   - exactly one `ShipmentLabelGenerated` event,
   - second operation is a safe replay/non-regressive result,
   - `printCount == 0` / no duplicate print fact.

A query that searches all of `pg_stat_activity` and takes the first blocked process is NOT sufficient.

A bare `Promise.all`, advisory inference, unique-constraint failure, or timing-only proof is NOT sufficient.

## Evidence completeness requirements

Update only:

`WMS_Outbound/05_EVIDENCE/P1-013_EVIDENCE.md`

The final evidence must include all of the following with exact values from the final run:

- accepted P1-012 base,
- pre-closeout P1-013 head `ebc556edd2366a4bb45351924ae2f22a1cafb093`,
- exact final closeout Mercato SHA,
- exact lineage/compare count and merge base,
- exact Git-derived file stats for the final P1-013 range,
- literal final P1-013 test output/count including `pidA`, `pidB`, `blockingPids`, `waitEventType: 'Lock'`,
- fresh independent one-label/one-event replay proof,
- rollback proof already required by P1-013,
- P1-012 / P1-011 / FND-002 regression results if rerun; do not invent fresh results if not rerun,
- Playwright 4/4 result only if actually rerun; otherwise preserve the prior truthful result and explicitly identify it as prior evidence,
- **exact served Testing Mercato runtime revision SHA** used for the browser evidence, obtained from the actual served checkout/runtime rather than inferred from branch name,
- exact final `git status --short` (or equivalent) for relevant Mercato and WMS worktrees proving clean state after push,
- Scanner frozen SHA,
- explicit exclusions/deferred P1-015 boundary,
- no mistyped/nonexistent WMS SHA such as earlier `834cda24...`.

If the served runtime revision is not the final Mercato closeout SHA because the closeout changes test-only code, state both exact SHAs explicitly and explain that product runtime files are unchanged from the served revision. Do not call a revision "final served" unless it was actually served.

## Verification scope

Because this owner override is test/evidence-only, the minimum required rerun is:

1. final P1-013 PostgreSQL suite with the strengthened PID-bound test,
2. any targeted command needed to prove exact served runtime revision and clean worktrees.

Do not rerun or modify unrelated areas merely to create activity. Existing P1-012/P1-011/FND-002/Playwright evidence may be retained only if clearly labeled with its actual run/revision provenance and still valid because runtime product code was not changed by this closeout.

If the executor chooses to rerun those suites/journeys, record the new literal outputs and exact served revision truthfully.

## Hard exclusions

No:

- runtime/product implementation changes outside the P1-013 test file,
- P1-014 ERP work,
- P1-015 CarrierManifest work,
- P1-016 settlement work,
- Scanner changes,
- Prod/Demo changes,
- STATE/handover/traceability/Task Catalog/plan/workflow edits,
- Devaxonic-WMS `.ai/*` edits,
- branch merge/delete,
- FINAL PASS / Owner Accepted claim by executor.

## STOP boundary

When complete:

1. push the single narrow Mercato closeout commit to `outbound/p1-013`,
2. push corrected `05_EVIDENCE/P1-013_EVIDENCE.md` to `WMS_Outbound/main`,
3. report the exact 40-char Mercato and WMS SHAs,
4. STOP.

Do not advance to P1-014. Do not update steering.

Owner-facing response remains microscopic: `done` on success; otherwise at most 5 lines with blocker + exact refs.
