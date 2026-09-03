# P1-011 — Fresh Codex Session Remediation

**Execution status:** STRIKE 1 remediation for catalog item `P1-011` — item **14/37**  
**Executor:** a **NEW Codex session**. No Antigravity session continuity is required. Reconstruct current state only from durable Git and current installed skills/contracts.

P1-011 is **not accepted**. Accepted campaign progress remains **13/37 FINAL PASS**. This is the second allowed shot on the same material path; if a material blocker remains after this shot, STOP applies unless the owner explicitly authorizes another narrow override.

## 0. Fresh-session bootstrap

Start from repository state, not chat history.

Refresh/read the current installed project skills and operating contracts available in the Codex environment:

- `wms-outbound` — authoritative routing for Outbound source hierarchy;
- `architecture-context` — **reference-only** for accepted shared/Inbound Inventory, TU, warehouse, locking and orchestration compatibility; it must not redefine Outbound rules;
- `fetch_me_prompt`;
- `operational-mode`;
- current real-evidence contract;
- `Devaxonic-WMS/.ai/TESTING.md`;
- `Devaxonic-WMS/.ai/OPERATIONS.md`.

Then sync/read current Git state:

1. `WMS_Outbound/STATE.md`;
2. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md`;
3. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`;
4. `WMS_Outbound/06_AGENT_GUIDES/P1-011_EXECUTION.md`;
5. `WMS_Outbound/06_AGENT_GUIDES/P1-011_REMEDIATION.md` — use its product/evidence blocker content, but ignore its Antigravity same-session wording;
6. `WMS_Outbound/07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact P1-011 section;
7. `WMS_Outbound/03_TRACEABILITY/requirements_index.csv`;
8. `WMS_Outbound/03_TRACEABILITY/test_index.csv`;
9. `WMS_Outbound/03_TRACEABILITY/state_event_transitions.csv`;
10. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — especially KROK 9, R26-R30, R57-R60 and P5 E17;
11. current Mercato `outbound/p1-011` source, tests, migrations and current durable `P1-011_EVIDENCE.md`.

Authority order for Outbound behavior:

1. current Architect Source / faithful translations;
2. Canon + traceability + exact current task docs;
3. current Mercato/Scanner/DB/runtime as implementation evidence;
4. Implementation Plan as delivery decomposition only.

If `architecture-context` conflicts with an Outbound R-rule, the Outbound Architect source wins.

## 1. Supervisor-observed starting refs

Refresh these before editing and STOP if the current branch is not a clean descendant of the expected base:

- accepted P1-010 Mercato base: `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`;
- current P1-011 Mercato head: `fef4058fbcfe1993589705528be6720674ee1dab`;
- Scanner frozen head: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- WMS main before this guide: `e65801d42057b68abf0c46f17a487667f45d06ea`;
- accepted P1-010 durable evidence: `b5bb6429717402e0fb6969f7437ddaf673a8a174`.

Continue remediation on Mercato branch `outbound/p1-011`. Do not rewrite accepted P1-010 history. Scanner is not a product target and stays frozen unless current authoritative source proves otherwise.

Do not update `STATE.md`, current handover, task catalog, implementation plan or `.ai` acceptance checkpoints. Supervisor owns final acceptance.

## 2. Product blocker A — CON-04 stable find-or-create for distinct compatible TUs

Current code row-locks an individual TU and any already-existing open Shipment rows. This does not protect the empty-set case: two independent transactions grouping **two different compatible TUs** can both observe no matching open Shipment and each create one.

Fix this at the database/transaction boundary. Use a transaction-scoped PostgreSQL lock keyed by the normalized exact R26 grouping tuple, or an equivalent durable database uniqueness/serialization seam. Do not use process-local mutexes or in-memory maps.

The serialized grouping key must include:

- organization;
- tenant;
- warehouse;
- customer;
- delivery address;
- priority;
- exact `slaDeadline`, including the null case.

Acquire the grouping-key lock before the find-or-create decision, then re-read the matching open Shipment and create only if absent. Preserve one-TU idempotency.

Required decisive genuine PostgreSQL test:

- two **different** sealed Packing TUs with the same exact R26 key;
- independent overlapping database participants/connections;
- real DB wait/lock evidence tied to grouping;
- after completion, fresh independent reads prove exactly one matching Shipment and both different TUs reference it;
- keep a separate same-TU replay/idempotency test.

## 3. Product blocker B — no-partial CustomerOrder must remain one complete Shipment even after SLA expiry

P1 R57/R58 is a CustomerOrder-level promise. `allowPartialShipment=false` means the complete CustomerOrder is released in one Shipment, independent of how many OutboundOrders/channels contributed.

Current readiness logic can close a just-created Shipment via expired SLA after the R58 completeness guard becomes true but before all eligible ready TUs for that CustomerOrder have been attached. TU1 can close Shipment A, then TU2 is forced into Shipment B. This is not allowed.

Fix grouping/readiness so, for every contributing no-partial CustomerOrder, Shipment `CREATED -> READY_FOR_DISPATCH` is blocked until **all active eligible TUs belonging to the complete CustomerOrder are attached to the same Shipment**. An elapsed SLA must not split that CustomerOrder. Preserve ordinary R28 SLA closure for `allowPartialShipment=true`.

Required decisive test:

- `allowPartialShipment=false` CustomerOrder;
- at least two ready sealed TUs;
- all active line coverage is complete, so R58 eligibility passes;
- common SLA already expired;
- grouping/reevaluation ends with one Shipment containing all relevant TUs and only then becomes `READY_FOR_DISPATCH`;
- repeated reevaluation remains idempotent.

## 4. Product/evidence blocker C — restore accepted historical migration exactly

The first P1-011 commit modified:

`apps/mercato/src/modules/wms_outbound/migrations/Migration20260902160000_wms_outbound_p1_007_remediation.ts`

That file belongs to an accepted earlier checkpoint and must not be rewritten by P1-011.

Restore it **byte-for-byte** to the accepted P1-010 base version from commit:

`19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`

If a current compiler/runtime compatibility issue motivated the edit, solve it outside the historical migration or document a true blocker and STOP; do not mutate accepted migration history.

## 5. Product/evidence blocker D — no secrets or hardcoded credentials in tracked Playwright source

The first P1-011 Playwright file contains hardcoded login credentials/test secrets. Remove them from tracked source.

Use the current approved environment/config/fixture mechanism for UI authentication. Never commit passwords, tokens or credential hashes. Do not write secrets into WMS evidence.

The final Playwright suite must still run against the real Testing Mercato/DB path and prove rendered UI outcomes.

## 6. Coverage exactness to preserve/fix

The final P1-011 PostgreSQL suite must decisively cover the authoritative P1-011 boundary, including:

- R26 exact grouping key: customer + delivery address + priority + **identical slaDeadline** within tenant/org/warehouse scope;
- incompatible customer/address separation;
- exact-SLA separation;
- stable concurrent find-or-create for distinct compatible TUs;
- one-TU/one-Shipment replay idempotency;
- R57/R58 CustomerOrder completeness guard;
- PLANNED/partial coverage blocking (TC-123);
- inactive/corrected branch handling (TC-124);
- cross-channel completeness (TC-125);
- expired SLA cannot bypass no-partial guard (TC-126);
- P5 E17 completed fragment waits (TC-127);
- R28 all-contributors readiness;
- R28 SLA closure for partial-allowed work;
- R29 late TU creates a follow-up Shipment without mutating the closed one;
- non-regressive repeated reevaluation;
- real rollback with independent fresh read.

Do not expand into P1-012 Carrier/Region/CarrierSetup, labels, ERP posting, manifest, dispatch or settlement.

## 7. Playwright requirements

Retain the normal Mercato operational Shipment/readiness UI, not a test-only page.

Final browser proof must be genuinely executed from the final Mercato head and labelled only `PLAYWRIGHT VERIFIED`.

At minimum preserve/prove:

1. eligible grouping/readiness with visible Shipment identity/state and matching DB membership;
2. `allowPartialShipment=false` blocked banner/reason, DB no-membership while blocked, then genuine server reevaluation unblocks after completion without manual DB repair;
3. SLA closure + late TU displayed as two distinct Shipments for partial-allowed work.

Also rerun the accepted P1-010 Mercato Playwright Packer suite because P1-011 changes the packing handoff surface.

## 8. Final-head regression gates

On the **final new Mercato head**, capture fresh literal output for:

- P1-011 PostgreSQL integration suite;
- P1-010 PostgreSQL suite — the accepted suite is **16 tests**, not 15;
- P1-009 PostgreSQL suite;
- P1-003 planning/grouping regression;
- full `src/modules/wms_outbound` backend umbrella;
- P1-011 Mercato Playwright suite;
- P1-010 Mercato Playwright suite;
- typecheck/build if required by current testing contract.

If a shared Inventory/TU/warehouse/lock/orchestration primitive is materially changed, run the exact shared/Inbound regressions required by current `architecture-context` and evidence contract.

Scanner remains frozen unless an authoritative shared-runtime reason requires a regression run; do not invent Scanner Shipment UI.

## 9. Rebuild durable evidence from final output only

Rewrite/update:

`WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md`

Use actual fresh captures from the final product head. Do not relabel old stdout.

Evidence must include:

- full 40-character accepted P1-010 base SHA;
- full 40-character final P1-011 Mercato SHA;
- frozen Scanner SHA;
- exact WMS guide/ref;
- exact changed-file diff scope;
- confirmation that the historical P1-007 migration matches the accepted base blob exactly;
- safe Testing PostgreSQL identity/provenance;
- exact commands/working directories and literal output captured losslessly (`tee` or equivalent);
- decisive distinct-TU grouping concurrency evidence with PostgreSQL participant/wait details;
- no-partial + expired-SLA one-Shipment proof;
- rollback proof with fresh independent DB read;
- exact final test counts/titles/timings as actually produced;
- exact P1-010 regression result (`16/16` if unchanged and green);
- actual Playwright execution output and visible assertions;
- no credentials/secrets;
- browser evidence labelled `PLAYWRIGHT VERIFIED` only;
- explicit note that P1-012+ remains out of scope.

The first-shot evidence at `9f7483afbcdf4e69a622197dc20554efc91a6f3b` is historical STRIKE 1 evidence, not final proof.

## 10. Stop boundary

Push:

1. the remediated `Devaxonic-mercato/outbound/p1-011` implementation/test commit(s);
2. the rebuilt `WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md` update.

Then **STOP**.

Do not update acceptance/state/handover. Do not start P1-012. Do not ask the owner to paste logs, SHAs, screenshots or test output; the supervisor will independently verify Git state after the owner reports completion.
