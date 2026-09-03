# P1-009 — Direct Pack declaration and automatic sealing — bounded remediation

**Status:** corrective shot 2/2 for the same material path  
**Item:** P1-009 — item 12/37  
**Rule:** SAME existing Antigravity session only. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`. After this shot, push implementation/evidence and STOP. If the same material path still fails, two-strikes STOP applies; do not invent a third workaround without explicit owner override.

## 1. Current independently verified refs

Use the current remote refs, not executor prose:

- Mercato accepted base: `134db31381b4db726cd550abe6ecd4079ac21d8c`.
- Mercato current P1-009 head: `outbound/p1-009` = `5c95a7334f071cb22e7dd566a4cd9ffbd657a704` — exactly one commit ahead of the accepted base.
- Scanner accepted base: `b23325aae1c4f83b79d01b3650dbead3486a1041`.
- Scanner current P1-009 head: `outbound/p1-009` = `6edba157e35653b1f833270ae33b3e5f0c216a12` — exactly one commit ahead of the accepted base.
- WMS_Outbound current `main` before this remediation guide: `b6eacf53f82e315f3e46b597c7253ad6a99b8f30`.
- Current first-shot evidence: `05_EVIDENCE/P1-009_EVIDENCE.md` at WMS commit `b6eacf53f82e315f3e46b597c7253ad6a99b8f30`.

Continue the existing P1-009 branches. Do not create replacement implementation branches and do not reset accepted bases.

## 2. Authority and scope remain unchanged

Read/refresh the original execution guide first:

`06_AGENT_GUIDES/P1-009_EXECUTION.md`

The same authority remains binding:

- P1 R15–R18, with directly referenced R55/R62/R64/R65/R67/R68 only as needed;
- `FR-P1-08`, `FR-P1-09`;
- `TC-001`, `TC-004`, `TC-005`;
- accepted P1-006 picking continuation and P1-008 TU identity/capacity/issueability semantics.

Do not absorb P1-010 packing/repack/consolidation/discrepancy UI, Shipment/Carrier/label/ERP/manifest scope, future P4, or unrelated refactors.

## 3. Why the first shot is not accepted

The first-shot lineage is clean, but supervisor verification found evidence and implementation inconsistencies that prevent acceptance.

### A. Durable evidence is not exact enough and conflicts with checked-in artifacts

`05_EVIDENCE/P1-009_EVIDENCE.md` currently records shortened implementation heads (`5c95a7334...`, `6edba15...`) instead of required 40-character SHAs and records the pre-evidence WMS head rather than the evidence commit itself.

More importantly, the written 15/15 result summary does not exactly match the checked-in test source. For example, the evidence describes test `4C` as an OutboundOrder completion test, while the checked-in `p1-009-postgres.integration.test.ts` test `4C` is the R65 operator issueability override case. Evidence must be regenerated from actual commands/results, not reconstructed from memory or intended behavior.

### B. The checked-in Scanner Playwright spec does not cover the required P1-009 UI matrix

Current `e2e/p1-009-real-scanner-direct-pack.spec.ts` contains one happy-path journey only:

- declare direct pack;
- create/bind TU;
- complete one-zone pick;
- close TU;
- DB verify sealed/PACKED result.

It does **not** provide the required rendered-UI proof for:

1. same Picking TU continuing into a second accepted task/zone without asking for direct pack again;
2. R67 `PICK_FULL` -> replacement TU inheritance without re-prompt, although the current UI exposes declare-full/switch-TU actions and this path is deterministic enough to test now;
3. a negative non-issueable direct-pack UI path that does not falsely present a successful seal/packed result.

Because `src/screens/PickingTaskScreen.js` is the same Scanner seam used by accepted P1-006/P1-007/P1-008 behavior, the first-shot evidence also omitted required overlapping Scanner regressions.

### C. Browser evidence is not durably proved as an executed passing run

The evidence describes a Playwright journey, but does not record the exact Playwright command, target runtime/base URL/revision and pass count/output. A checked-in spec plus narrative is not proof that the real rendered Scanner run passed.

### D. `packedQty` evidence is inconsistent with the checked-in direct-pack service

The current Scanner Playwright spec asserts after the UI journey:

- `outbound_order_line.status = 'PACKED'`;
- `picked_qty = 4`;
- `packed_qty = 4`.

However the checked-in P1-009 `processDirectPackClosure` path in `pick-task-service.ts` transitions the line status to `PACKED` but does not visibly update `WmsOutboundOrderLine.packedQty` when calculating sealed TU contribution. That makes the current durable claim/assertion impossible to accept without an independently proven mechanism outside this service.

Resolve this inconsistency in the product path, not by weakening or deleting the decisive persistence assertion. When a direct-pack line becomes `PACKED`, persist `packedQty` consistently with the sealed quantity covered by the authoritative TU contents/required quantity. Keep multi-TU behavior correct: do not mark the line packed early; update quantity only when the line has actually satisfied the same sealed-contribution condition used for the status transition.

### E. Unrelated accepted P1-007 migration file was modified

The current Mercato P1-009 diff changes:

`apps/mercato/src/modules/wms_outbound/migrations/Migration20260902160000_wms_outbound_p1_007_remediation.ts`

only to alter a TypeScript cast in the already accepted P1-007 migration. This is outside P1-009 and changes an accepted historical migration artifact. Revert that file exactly to the accepted base `134db31381b4db726cd550abe6ecd4079ac21d8c` version. P1-009 must not carry this unrelated diff.

## 4. Required remediation work

### 4.1 Mercato product correction

On `outbound/p1-009`:

1. Revert the P1-007 migration file above to the accepted-base content so it disappears from the P1-009 diff.
2. Keep the current R16 immutability and R67 inheritance behavior, but strengthen tests where needed.
3. Make the direct-pack `OutboundOrderLine` persistence internally consistent when the line reaches `PACKED`:
   - derive the packed quantity from authoritative sealed TU contents for that line;
   - do not count unsealed/noncontributing TU contents;
   - do not mark/update the line as fully packed before all required contribution is sealed;
   - persist `packedQty` atomically in the same transaction as the `PICKED -> PACKED` transition;
   - preserve replay/concurrency safety and accepted transition/audit behavior.
4. Do not create a parallel packing service or P1-010 workflow.

### 4.2 Strengthen genuine PostgreSQL P1-009 tests

Keep real Testing PostgreSQL only.

At minimum ensure the suite proves with fresh reads:

- true/false first declaration persistence;
- conflicting post-start change rejected and original declaration unchanged;
- same-value replay safe;
- same-TU multi-zone continuation actually performs the next-zone pick into the same TU, not merely assigns the continuation task, while retaining the declaration;
- R67 replacement TU inherits the declaration; cover both `true` and `false` if straightforward;
- qualifying direct pack seals through `READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED`;
- line reaches `PACKED` only after all required contributing TU quantity is sealed;
- `packedQty` equals the authoritative packed quantity at that point;
- non-issueable and below-threshold direct pack stop at `READY_TO_PACK`;
- non-direct-pack closes to `READY_TO_PACK` without auto-seal;
- SHORT/incomplete path never auto-packs;
- real rollback/concurrency proof remains genuine if retained.

Do not fabricate expected console output. Record the actual final command and actual test names/counts.

### 4.3 Expand real Scanner Playwright acceptance

Use the real rendered Testing Scanner and perform the decisive Picker actions through UI. API/DB may prepare deterministic fixtures and verify persistence, but may not replace the user action.

P1-009 Scanner suite must now prove these rendered journeys:

1. **Happy path direct pack:** declare at first TU binding/scan point, pick, close, automatic seal; fresh DB confirms `directPackDeclared`, `PACKING_SEALED`, PackUnit role, line `PACKED` and consistent `packedQty`.
2. **Same-TU multi-zone:** complete first zone, continue the same OutboundOrder into the next eligible zone using the same TU, prove the Scanner does not show the direct-pack declaration prompt again, complete the next pick and confirm the original declaration remains authoritative.
3. **R67 PICK_FULL inheritance:** through rendered Scanner actions, declare/current TU full, switch to replacement TU for the same active PickTask and prove no direct-pack prompt appears again and the replacement TU inherited the declaration.
4. **Negative issueability:** use a deterministic non-issueable (`externalIssuable = false`) or below-threshold direct-pack TU; perform the close action through UI and prove the UI does not claim successful automatic sealing while fresh DB remains at `READY_TO_PACK` and line is not falsely `PACKED`.

Label this `PLAYWRIGHT VERIFIED` only.

Do not rely on a screenshot stored only under an Antigravity brain/session directory as durable acceptance evidence. Screenshots are optional; the decisive proof is the real rendered action plus exact run output and persisted assertions.

### 4.4 Required regression matrix after correction

Because P1-009 changes the shared Scanner picking screen and backend picking completion seam, run and record exact commands/counts for:

- P1-006 backend PostgreSQL suite;
- P1-007 backend PostgreSQL suite;
- P1-008 backend PostgreSQL suite;
- P1-006 real Scanner picking/multi-zone Playwright regression;
- P1-007 real Scanner short-pick/replacement Playwright regression;
- P1-008 real Scanner TU issueability/capacity regression if its accepted spec exists on the current branch/runtime; if there is no separate accepted P1-008 Scanner spec, state exactly what accepted Scanner surface is reused and why rather than inventing a result;
- full `wms_outbound` umbrella after targeted backend suites are green (the harness is known to exist from prior accepted evidence).

Inbound/shared compatibility does not need ceremonial rerun if the final diff touches no shared Inbound primitive. If the remediation changes a shared TU/entity/migration primitive, run the targeted accepted shared regression required by `architecture-context`/testing contract.

## 5. Evidence rewrite requirements

Update `WMS_Outbound/05_EVIDENCE/P1-009_EVIDENCE.md` from actual final artifacts and actual executed output.

It must contain:

- exact 40-char final Mercato and Scanner branch heads;
- exact WMS evidence commit handling: the evidence file cannot know its own future commit SHA before commit, so record the implementation heads in-file and let the supervisor independently record/verify the WMS evidence commit after push; do not label a pre-evidence steering head as the "final evidence head";
- exact commands and exact pass/fail counts for every reported backend and Playwright run;
- target URLs/runtime provenance for real Scanner Playwright;
- real PostgreSQL identity proof for the P1-009 integration suite;
- exact P1-009 test names that match the checked-in source/output;
- truthful description of which required Scanner journeys executed;
- no `HUMAN VERIFIED` label;
- no invented/stale counts or abbreviated SHAs.

If a test was not executed, do not describe it as passed.

## 6. Diff discipline

Final Mercato diff from accepted base should contain only P1-009 product/tests/API changes genuinely required for this item. The historical P1-007 migration cast edit must be gone.

Final Scanner diff should remain limited to P1-009 Scanner behavior/tests plus any narrowly necessary correction to the same picking screen/API seam.

Do not touch WMS state/handover to declare FINAL PASS. Only update the P1-009 evidence file. Supervisor owns acceptance/state advancement after independent verification.

## 7. Definition of Done for remediation

This corrective shot is complete only when:

- the unrelated P1-007 migration diff is removed;
- direct-pack line status and `packedQty` persistence are mutually consistent and atomically proved;
- backend multi-zone proof actually performs the second-zone pick into the same TU;
- real Scanner P1-009 happy, multi-zone no-reprompt, R67 inheritance and negative issueability journeys pass through rendered UI;
- overlapping P1-006/P1-007/P1-008 Scanner regressions required above are run and truthfully recorded;
- targeted backend regressions and full `wms_outbound` umbrella are green;
- evidence contains exact 40-char heads, commands and real counts matching checked-in tests/output;
- all pushes are complete.

Then STOP. Do not start P1-010. Do not ask the owner to paste logs; the owner will reply only `done` and the supervisor will independently fetch/verify current refs and evidence.
