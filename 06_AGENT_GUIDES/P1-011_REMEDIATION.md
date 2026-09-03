# P1-011 — Remediation after first implementation shot

**Execution status:** STRIKE 1 remediation for catalog item P1-011  
**Catalog item:** `P1-011` — item **14/37**  
**Session:** continue the **SAME current Antigravity session** that executed `P1-011_EXECUTION.md`. Do not open another session. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

P1-011 is **not accepted**. The first implementation shot produced useful product surface, but supervisor verification found material product and evidence defects. This remediation is the second allowed shot on the same material path. If any material blocker remains after this shot, STOP applies unless the owner explicitly overrides.

## 0. Mandatory startup and current refs

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Use `/home/ubuntu/.local/bin/agy-pl`; never bare `agy` and never replace the current Antigravity session.

Refresh current installed:

- `wms-outbound`;
- `fetch_me_prompt`;
- `operational-mode`;
- current real-evidence contract;
- `Devaxonic-WMS/.ai/TESTING.md` and `.ai/OPERATIONS.md`;
- `architecture-context` **reference-only** for accepted shared/Inbound compatibility.

Then reread current `STATE.md`, current handover, `GIT_PROMPT_WORKFLOW.md`, `P1-011_EXECUTION.md`, current `P1-011_EVIDENCE.md`, Task Catalog P1-011, Architect Process 1 KROK 9 / R26-R30 / R57-R60, and the current final P1-011 source.

Supervisor-observed refs after shot 1:

- accepted Mercato P1-010 base: `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`;
- current P1-011 Mercato head: `fef4058fbcfe1993589705528be6720674ee1dab`;
- Scanner remains frozen: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- current WMS steering/evidence head before this guide: `9f7483afbcdf4e69a622197dc20554efc91a6f3b`;
- accepted progress remains **13/37 FINAL PASS**.

Verify ancestry before editing. Continue on `outbound/p1-011`; do not rewrite accepted P1-010 history.

Do not update `STATE.md`, handover, task catalog, implementation plan or `.ai` acceptance state.

## 1. Blocker A — CON-04 grouping creation race

The current implementation locks the individual TU and any **already-existing** open Shipment rows. That is insufficient when two independent transactions group **two different compatible TUs** and no matching Shipment exists yet: both transactions can observe zero matching rows and each create a separate Shipment for the same exact R26 grouping key.

The current concurrency test races two actors over the same TU row, which proves one-TU idempotency but does **not** prove stable find-or-create grouping for distinct compatible TUs.

Fix this at the database/transaction level. Use the smallest pattern consistent with current architecture, for example a transaction-scoped database lock keyed by the normalized exact R26 grouping tuple, or an equivalent durable uniqueness/locking seam. Do not use an in-process mutex or in-memory map.

The serialized grouping key must include the exact business dimensions actually used by P1 R26 within tenant/org/warehouse isolation:

- organization;
- tenant;
- warehouse;
- customer;
- delivery address;
- priority;
- exact `slaDeadline` including the null case.

After acquiring the grouping-key lock, re-read the open matching Shipment and create only if none exists.

Required decisive PostgreSQL test:

- create **two different** sealed Packing TUs with the same exact grouping key;
- start independent overlapping transactions/connections so both attempt find-or-create concurrently;
- capture real PostgreSQL participant/wait evidence tied to this operation;
- after both complete, fresh independent DB reads must prove exactly one matching Shipment exists and both distinct TUs reference that same Shipment;
- also preserve the separate same-TU idempotency test.

## 2. Blocker B — `allowPartialShipment=false` must not split after an expired SLA

Current readiness logic can still close a Shipment via `slaElapsed` after the CustomerOrder completeness guard becomes true but before every ready TU belonging to that no-partial CustomerOrder has been attached. Example: two ready sealed TUs, guard now passes, deadline already elapsed; grouping TU1 may create Shipment and immediately close it by SLA, then TU2 is forced into a second Shipment.

That violates P1 R57/R58: `allowPartialShipment=false` means the entire CustomerOrder is released in **one complete Shipment** and SLA does not override that promise.

Fix the readiness/grouping seam so a no-partial CustomerOrder cannot reach Shipment `READY_FOR_DISPATCH` until all eligible active TUs contributing to that complete CustomerOrder are attached to the same Shipment. An expired SLA may not split a no-partial CustomerOrder after the guard has become satisfied.

Do not weaken normal P1 R28 SLA closure for `allowPartialShipment=true`.

Required decisive test:

- `allowPartialShipment=false` CustomerOrder with at least two ready Packing TUs;
- all active line coverage is complete, so the R58 eligibility guard passes;
- common SLA is already expired;
- execute the normal grouping/reevaluation path;
- fresh DB proof: both TUs end in the **same** Shipment, there is no second Shipment for the same CustomerOrder promise, and readiness occurs only after complete same-Shipment membership.

Keep the existing TC-126 case where the completeness guard itself is still false; both cases are required because they prove different boundaries.

## 3. Blocker C — validate the complete TU grouping identity, not only `primaryOrder`

Current `groupPackingTuIntoShipment` derives customer/address/priority/SLA from `contributingOrders[0]`. If corrupted or legacy data puts multiple contributing OutboundOrders with incompatible Shipment identities into one Packing TU, the service silently chooses the first order's key.

P1-011 must fail closed rather than silently grouping incompatible contents. P1 R60 is primarily enforced at packing time, but P1-011 owns the one-Shipment membership seam and must not manufacture a Shipment key from only one contributor when the TU contains several.

Before grouping, validate all contributing OutboundOrders represented by TU content against the effective Shipment identity required by current source. At minimum reject mismatches in:

- customer;
- delivery address;
- priority;
- exact `slaDeadline`.

Do not invent additional category/temperature/carrier dimensions.

Required tests:

- same customer/priority/SLA but **different delivery address** must not share a Shipment;
- a single TU whose contributing order data is internally incompatible must fail closed with no Shipment/TU state mutation;
- keep the exact-SLA negative test already present.

## 4. Preserve accepted historical files and remove unrelated churn

The first P1-011 commit modified the accepted historical migration:

`apps/mercato/src/modules/wms_outbound/migrations/Migration20260902160000_wms_outbound_p1_007_remediation.ts`

Restore that file **byte-for-byte** to the accepted P1-010 version. Supervisor-observed accepted blob is `38a3dad7f356021b1e6b67bf70ef8d9829e9ed71`.

Do not modify historical accepted migrations to solve TypeScript/compiler issues. If a current compiler typing issue exists, solve it in current code or a new additive migration without rewriting already-accepted migration history.

Review the broad `packing-service.ts` churn from shot 1. Revert semantic changes unrelated to P1-011 unless they are strictly required for the current P1-011 behavior. In particular:

- do not refactor all accepted P1-010 lock calls merely for style;
- do not alter accepted Direct Pack/Packer semantics;
- if a minimal field default is genuinely required for a new P1-011-created object, isolate and test it without broad accepted-code rewrites.

Final diff from accepted P1-010 must be explainable as P1-011 only.

## 5. Remove committed credentials/secrets from Playwright source

The first P1-011 Playwright spec committed concrete authentication credentials in source. This is not acceptable.

Remove all hard-coded passwords, tokens and reusable account credentials from the repository and from evidence. Use the current approved test authentication/fixture/environment mechanism already used by the project. Fail clearly if required test credentials are unavailable; do not fall back to embedded secrets.

Do not print credentials to durable evidence or terminal captures committed to Git.

## 6. Complete the decisive PostgreSQL coverage

Keep the valid first-shot tests and add/fix coverage so the final P1-011 suite directly proves the original execution guide, including:

1. compatible R26 grouping happy path;
2. exact SLA mismatch -> separate Shipment;
3. delivery-address mismatch -> separate Shipment;
4. one-TU/one-Shipment replay;
5. **distinct compatible TU concurrent find-or-create -> exactly one Shipment** with real DB-side contention;
6. incomplete no-partial guard;
7. TC-123 partial `requiredQty` despite PLANNED status;
8. TC-124 inactive/corrected-line branches;
9. TC-125 cross-channel completeness;
10. TC-126 expired SLA cannot bypass an incomplete CustomerOrder guard;
11. P5 E17 packed fragment waits for complete CustomerOrder;
12. R28 all-contributors-ready closure;
13. R28 SLA closure for partial-allowed work;
14. R29 late TU creates follow-up Shipment for partial-allowed work;
15. **expired SLA + now-complete `allowPartialShipment=false` with multiple ready TUs stays one complete Shipment**;
16. incompatible mixed-content TU fails closed;
17. non-regressive replay;
18. genuine rollback after real write/flush with fresh independent read.

Record the **actual** final test count and titles. Do not force the suite to remain 16 tests if correct decisive coverage needs more.

## 7. Mercato Playwright proof

Preserve the normal Mercato UI — no test-only page.

Final Playwright must run on the final P1-011 head and prove through rendered UI + fresh DB assertions:

- Journey A: no-partial blocked banner, no Shipment membership while incomplete, then unblock through the supported server/UI path and one complete Shipment membership;
- Journey B: eligible grouping/readiness and visible Shipment identity/status;
- Journey C: partial-allowed SLA closure + late TU shown under a second Shipment while the first remains unchanged.

Add rendered coverage for the remediated no-partial expired-SLA case if the existing Journey A does not decisively prove it. The user-visible contract must never imply that an expired SLA overrides `allowPartialShipment=false`.

Use the approved environment/auth fixture. No hard-coded credentials.

Evidence label remains `PLAYWRIGHT VERIFIED` only.

## 8. Regression gates on the final head

Rerun with literal output captured via `tee`/lossless capture:

- final P1-011 PostgreSQL suite;
- P1-010 PostgreSQL suite — the accepted source has **16 tests**; record the actual final output, never `15` by assumption;
- P1-009 Direct Pack PostgreSQL suite;
- P1-003 planning/grouping suite;
- full `src/modules/wms_outbound` backend umbrella;
- final P1-011 Mercato Playwright suite;
- accepted P1-010 Mercato Playwright Packer suite;
- typecheck/build only as current contract requires;
- shared/Inbound regressions only if a shared primitive is actually touched.

Scanner remains frozen unless a real shared runtime/API change requires a regression; do not implement Shipment workflow in Scanner.

## 9. Evidence must be rebuilt factually

Update only:

`WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md`

The current evidence is not acceptable because it contains abbreviated refs, an incorrect P1-010 regression count, incomplete exact-output provenance and a browser-verification statement that does not demonstrate an actual final-head Playwright run.

Final evidence must contain:

- full 40-character accepted P1-010 base ref;
- full 40-character final P1-011 Mercato ref;
- full 40-character frozen Scanner ref;
- full 40-character current WMS remediation-guide/pre-evidence ref;
- exact final changed-file scope, explicitly proving the historical P1-007 migration matches the accepted base again;
- safe Testing PostgreSQL identity;
- exact cwd and commands;
- literal fresh stdout/stderr from final-head P1-011 backend suite;
- actual current test titles/counts/timings as emitted — never reconstructed;
- real distinct-TU grouping-key contention evidence;
- real rollback output/evidence;
- literal current P1-010 regression output showing its actual count;
- literal full Outbound umbrella output;
- literal Playwright command/output with actual test count and titles on the final head;
- runtime/base URL;
- truthful rendered assertions and fresh DB assertions;
- no credential values;
- no statement such as “full browser execution requires the instance to be active” while simultaneously claiming `PLAYWRIGHT VERIFIED`; either the browser run actually happened and its literal output is recorded, or the evidence must say it did not pass.

Do not guess the final WMS evidence SHA.

## 10. Stop boundary

Push the minimal Mercato remediation and rebuilt durable evidence, then **STOP**.

Do not advance `STATE.md`, handover or `.ai` acceptance state. Do not start P1-012.

Owner replies only `done`; supervisor independently verifies Git refs, lineage, diff scope, historical migration restoration, concurrency behavior, no-partial/SLA behavior, UI evidence and exact regressions.
