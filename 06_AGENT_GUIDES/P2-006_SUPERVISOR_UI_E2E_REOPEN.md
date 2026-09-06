# P2-006 — supervisor reopen: complete the mandatory rendered E2E proof

## Target and authority

Continue the SAME P2-006 item from the preserved pushed candidate.

Canonical repositories:

- WMS steering/evidence: `/home/ubuntu/git/WMS_Outbound`
- Mercato: `/home/ubuntu/git/Devaxonic-mercato`
- Scanner: `/home/ubuntu/git/Devaxonic-scanner`

Current pushed candidate under supervisor review:

- Mercato branch `outbound/p2-006`: `2c50a186ade68ec2e73ec479654a8ec989c88eb4`
- accepted P2-005 parent: `069f02d4c5c9b345b688b838eb685be02206afbd`
- Scanner frozen: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- current evidence commit: `a6118bce415a5b52cd0cbd452120459a70819049`

Read and obey current:

1. `/home/ubuntu/git/Devaxonic-WMS/AGENTS.md`
2. `/home/ubuntu/git/Devaxonic-WMS/.ai/TESTING.md`
3. `/home/ubuntu/git/Devaxonic-WMS/.ai/OPERATIONS.md`
4. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
5. `WMS_Outbound/06_AGENT_GUIDES/P2-006_EXECUTION.md`
6. this reopen guide.

The full-item autonomy/two-strikes workflow is authoritative. Do not return after ordinary fixture/auth/runtime failures. Diagnose and fix them inside this item. STOP only after the item is genuinely complete or the same material blocker survives two substantive evidence-based attempts with no normal in-scope next move.

## Why supervisor reopened P2-006

The pushed product diff and backend matrix are acceptable so far, but the final rendered Playwright proof does not yet satisfy the mandatory P2-006 E2E contract from `P2-006_EXECUTION.md`.

The current spec `P2-006-crossdock-shipment-downstream-ui.spec.ts` splits the proof in a way that leaves required rendered E2E steps unproven:

### Gap A — Journey A does not prove crossdock joins Shipment through the common action

Current Journey A directly creates:

- a Shipment already in `READY_FOR_DISPATCH`;
- the crossdock Packing TU already in `IN_SHIPMENT` with `shipment_id` assigned.

That bypasses mandatory Journey A steps 1-2 from the authoritative guide:

1. genuine crossdock TU starts as `PACKING_SEALED` with persisted CrossDockPickTask/placement/source lineage;
2. that TU is attached through the real common Shipment action/service, not by direct fixture assignment.

Journey A must become one continuous real crossdock `PACKING_SEALED -> common Shipment -> carrier -> label -> GR gate -> ERP -> manifest -> confirm -> persisted settlement` path.

### Gap B — Journey B does not finish the mandatory mixed-channel downstream path

Current Journey B/C proves blocking, later common grouping, same-Shipment membership, gate visibility and replay idempotency, but stops there.

The authoritative Journey B additionally requires:

- expired `slaDeadline` cannot bypass the incomplete CustomerOrder guard;
- P2-005 gate requires only the CROSSDOCK source(s), not STANDARD content;
- after GR acceptance the SAME mixed Shipment completes the common Carrier Selection + label + ERP + CarrierManifest path;
- persisted reconciliation proves exactly-once STANDARD Allocation/Inventory settlement and Allocation-free/no-standard-Inventory CROSSDOCK settlement.

Those must be proved in the rendered mixed journey, not merely inferred from separate backend tests or from the pure-crossdock Journey A.

Journey C replay may remain folded into Journey B if the one-Shipment invariant stays explicit.

## Preserve already-verified work

Do NOT deep-reset. Do NOT reconstruct/rebase/clean away P2-006 history.

Treat these as retained unless a subsequent PRODUCT CODE change invalidates them:

- dedicated P2-006 PostgreSQL: 20/20 PASS;
- P2-005 19/19;
- P1-011 18/18;
- P1-012 14/14;
- P1-013 15/15;
- P1-014 18/18;
- P1-015 21/21;
- P1-016 25/25;
- P2-002 22/22;
- P2-003 8/8;
- P2-004 16/16;
- durable typecheck PASS;
- durable fresh build PASS;
- canonical runtime/build proof;
- Scanner frozen.

Do not rerun that full backend matrix merely because the UI spec changes.

If the corrected rendered journey exposes a genuine PRODUCT defect and product source must change, fix only the concrete P2-006 defect, then rerun the dedicated P2-006 suite plus only directly invalidated accepted regressions and refresh typecheck/build/runtime as required.

## Mandatory UI correction

Use deterministic DB/API fixture preparation only for prerequisites that are not themselves the accepted user action. Do not directly manufacture the Shipment membership or downstream state that the journey is supposed to prove.

Use repository-proven auth/API helpers (`getAuthToken`, `apiRequest`) for external-system ingress or non-human fixture preparation where appropriate. Human-facing Shipment/Carrier/Manifest actions must remain real rendered UI actions. No Playwright route mocks/interception/substitution.

### Journey A — one continuous crossdock E2E

The final rendered test must prove, on the SAME persisted chain:

1. CROSSDOCK Packing TU exists `PACKING_SEALED` with source TU + CrossDockPickTask + placement lineage;
2. real normal/common Shipment grouping action attaches it to the common Shipment model;
3. normal Shipment UI shows it and normal readiness is reached;
4. rendered common Carrier Selection succeeds;
5. rendered common WMS label generation/print succeeds;
6. P2-005 GR gate is visibly blocked for its unresolved crossdock source;
7. rendered ERP posting while blocked produces zero posting side effects;
8. real accepted external CROSSDOCK GR ingress marks the source accepted;
9. SAME Shipment UI becomes GR-ready and rendered ERP posting succeeds;
10. rendered normal CarrierManifest open/add/close/handover/confirm completes;
11. persistence proves terminal Shipment/TU/OOL/OO/CustomerOrder states, exactly one manifest settlement, zero fabricated CROSSDOCK Allocation and zero standard Inventory movement for that crossdock contribution.

### Journey B/C — one continuous mixed no-partial E2E

The final rendered test must prove, on the SAME mixed CustomerOrder:

1. legitimate STANDARD + CROSSDOCK compatible contributions with identical customer/address/priority/slaDeadline;
2. while CROSSDOCK is incomplete, ready STANDARD TU remains `PACKING_SEALED` outside Shipment (and/or symmetric channel block as appropriate);
3. make `slaDeadline` expired while still incomplete and prove the normal grouping action still cannot attach either blocked TU;
4. complete P1 R58 equality for both active lines;
5. rendered/common grouping attaches both TUs to ONE common Shipment;
6. normal Shipment UI shows the one mixed Shipment;
7. GR gate lists/requires only CROSSDOCK source TU(s), not STANDARD content;
8. real CROSSDOCK GR acceptance unblocks that same Shipment;
9. rendered common Carrier Selection + label + ERP posting succeeds on that mixed Shipment;
10. rendered normal CarrierManifest open/add/close/handover/confirm completes;
11. persistence proves exactly-once STANDARD Allocation consumption + standard Inventory movement, while CROSSDOCK creates no Allocation/standard Inventory decrement; both channel business states and final CustomerOrder aggregates advance correctly;
12. replay/negative boundary proves the two TUs remain members of only that same Shipment and no second Shipment is created.

Do not satisfy these by two disconnected fixture chains where one proves grouping and another proves downstream. The acceptance point is the continuous common logistics lifecycle.

## Execution discipline

Own the entire correction in this execution. Ordinary issues such as auth session plumbing, TLS, deterministic fixture data, selectors, service timing or durable runner handling are executor-owned and are NOT reasons to return control.

Use durable host-side test execution if needed. Never launch duplicate Playwright units while one is still active.

No arbitrary rerun-count limit. Apply the canonical two-strikes rule only to the SAME material blocker after two genuinely different substantive attempts.

## Final gates

Before reporting COMPLETE:

- corrected dedicated P2-006 Playwright spec passes all required continuous Journey A and Journey B/C tests against canonical runtime;
- zero `page.route`, `.route(`, request interception or response substitution;
- final spec/source review confirms no direct DB assignment substitutes the required common grouping/downstream UI action;
- if product code unchanged from `2c50a186...`, retain existing backend/typecheck/build proof and do not waste time rerunning it;
- if Mercato changes are test-only, commit the minimum spec correction on `outbound/p2-006` and push;
- if product source changes, perform the directly invalidated test/build/runtime gates described above before push;
- update `05_EVIDENCE/P2-006_EVIDENCE.md` with the new exact Mercato SHA, exact final Playwright unit/result, continuous Journey A/B proof, retained-vs-rerun gates, Scanner frozen SHA, and clean worktree status;
- push WMS evidence;
- evidence must not self-declare FINAL PASS / Owner Accepted / Human Verified;
- STOP. Do not start P3-001.

## Final report

Return only:

- `COMPLETE` plus final Mercato SHA, Scanner frozen SHA, WMS evidence SHA and final rendered Journey A / Journey B/C counts; or
- one true two-strikes material blocker with the two substantive attempts and exact remaining decision needed.
