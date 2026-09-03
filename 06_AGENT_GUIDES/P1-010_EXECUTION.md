# P1-010 — Packing, repack, consolidation and discrepancy handling

**Execution status:** first implementation shot — **1/2**  
**Catalog item:** `P1-010` — item **13/37**  
**Mode:** implementation + decisive evidence  
**Session:** continue the **SAME existing Antigravity session only**. Follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

## 0. Mandatory startup

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Run delegated execution from the existing VPS workspace and existing Antigravity session. Use `/home/ubuntu/.local/bin/agy-pl`; never replace the session and never use bare `agy`.

Before changing product code, refresh and follow:

1. `WMS_Outbound/STATE.md`
2. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md`
3. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
4. `WMS_Outbound/07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact `P1-010` section
5. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — current KROK 7/KROK 8 and P1 R19–R26, plus R48/R60/R63/R66 where they govern the same packing actions
6. `WMS_Outbound/03_TRACEABILITY/requirements_index.csv` — `FR-P1-10`, `FR-P1-11`, `FR-P1-12`, `FR-P1-13`, `FR-P5-07`, and preserve already-accepted R60/R63/R66 requirements
7. `WMS_Outbound/03_TRACEABILITY/test_index.csv` — `TC-001`, `TC-005`, `TC-006`, `TC-063`
8. `WMS_Outbound/03_TRACEABILITY/state_event_transitions.csv` — current TU packing/repack transitions
9. current `Devaxonic-WMS/.ai/STATE.md`, `.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-03.md`, `.ai/PLAN.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`
10. current installed `wms-outbound`, `fetch_me_prompt`, `operational-mode`; use `architecture-context` **only** for accepted shared/Inbound Inventory/TU/warehouse/lock/orchestration compatibility.
11. current real-evidence contract used by the workspace for PostgreSQL/concurrency/rollback/UI proof.

Authority for Outbound business behavior remains Architect source/canon first. Current code is implementation evidence, not permission to weaken or redefine the Architect rules.

## 1. Frozen accepted starting point

Supervisor-verified accepted P1-009 implementation heads:

- Mercato `outbound/p1-009` = `5d780dabeb605bc657bb521bd2b2fdcc2e516f77`
- Scanner `outbound/p1-009` = `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- P1-009 durable evidence = `6c78a97ece567d90d3cb7d0580bb38669c9f9722`

Create/use Mercato branch `outbound/p1-010` from exactly the accepted Mercato P1-009 head unless that branch already exists as a clean descendant of that head. Verify ancestry before work. Preserve unrelated local work.

**Scanner is frozen by default.** Do not create a Scanner P1-010 change merely to duplicate a Packer UI. The Task Catalog target is Packing backend + Mercato/Packer UI + TU contents + QC handoff. Only touch Scanner if inspection proves that an already-accepted current product boundary places the decisive Packer action there; if so, document the exact reason and keep scope minimal. Otherwise leave Scanner at the accepted P1-009 head.

Do not edit accepted historical migrations or prior evidence to make P1-010 pass.

## 2. P1-010 business objective

Implement the standard Packer path for TUs that reach `READY_TO_PACK` and were **not** already completed by the accepted P1-009 Direct Pack bypass.

P1-010 owns:

- keep same TU when qualified;
- repack-all;
- repack-by-SKU;
- allowed packing consolidation;
- Packer discrepancy handling for missing, DAMAGED and unexpected/overage SKU;
- completion/accounting of the source Picking TU;
- QC handoff required by those discrepancy outcomes.

Preserve all accepted P1-008 TU identity/issueability behavior and all accepted P1-009 Direct Pack behavior.

## 3. Required functional behavior

### A. Standard `READY_TO_PACK` Packer decision

Provide the normal Packer UI/backend path that starts from a standard `READY_TO_PACK` TU.

The WMS must evaluate the current TU and expose the applicable packing suggestion/decision among keep / repack / consolidation according to current Architect rules and accepted configuration.

If the Packer deviates from the WMS suggestion, reuse the accepted Warehouse Supervisor authorization/audit semantics from P1 R18. Do not invent a separate approval model.

### B. Keep the same TU

When the current TU is eligible/issuable and no wrapper change is required:

- preserve the same `TU_NUMBER` and physical TU identity;
- change its role from `PickContainer` to `PackUnit` through the accepted domain seam;
- execute the valid audited transition chain `READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED`;
- do not create a replacement TU;
- do not duplicate P1-009 Direct Pack logic: this is the standard Packer action for a non-auto-packed TU.

### C. Repack modes

Support the Architect v1 warehouse-configurable modes:

1. **repack all**
   - physical transfer is confirmed as a whole;
   - do not invent per-SKU scanning/counting for this mode;
   - source TU may become `REPACKED` only after its required accounting is complete.

2. **repack by SKU**
   - Packer chooses the SKU/order to process;
   - Packer scans SKU code and quantity;
   - WMS must **not** direct a mandatory next-SKU order;
   - repeated transfers build the target Packing TU(s) while preserving quantity/accounting integrity.

Warehouse configuration may allow both modes or force one. Reuse the current configuration style instead of hard-coding one warehouse behavior.

Every repack target TU must satisfy accepted P1 R66 / P1-008 issueability semantics: a target with `externalIssuable = false` cannot be sealed as the outbound Packing TU.

### D. Source-TU completion gate

The source Picking TU may transition to terminal `REPACKED` only when **every system-expected SKU quantity** is accounted for as one of:

- packed into a valid target Packing TU;
- sent to QC;
- explicitly confirmed missing after the required recheck flow.

If accounting is incomplete, keep the source TU open for continued packing or allow the operator to enter the explicit missing flow. Never auto-generate a shortage merely because an intermediate scanned quantity is lower than `pickedQty`.

### E. Repack-by-SKU shortage / missing quantity

When scanned/accepted quantity is below the system `pickedQty`:

- do **not** automatically create shortage;
- allow the Packer to defer that SKU and return later;
- if the Packer explicitly reports missing quantity, require **recheck + explicit confirmation** before recovery;
- after confirmation, invoke the already-accepted SHORT_PICKED recovery mechanism for the missing quantity;
- **do not block the source picking location** for a packing-stage shortage.

Persist the cause/audit trail and actor context.

### F. DAMAGED quantity

A DAMAGED quantity:

- cannot be packed as good stock;
- is routed to QC through the existing/accepted QC handoff seam;
- is explicitly persisted as `DAMAGED` with actor/reason/audit context;
- invokes the same shortage-recovery mechanism for the missing-good quantity using cause `DAMAGED`;
- must not block the original picking source location.

Do not implement later QC disposition workflows owned by later scope; create/persist only the handoff and packing-side facts required here.

### G. Unexpected / overage SKU

Unexpected or overage SKU discovered during packing:

- route the physical/content discrepancy to QC;
- persist the packing discrepancy fact and actor/audit context;
- **do not** create a shortage for an unexpected/overage SKU merely because it is unexpected.

### H. Consolidation

Packing consolidation may combine content/orders only when current Architect compatibility allows it.

Preserve accepted P1 R60 compatibility at the packing seam: cross-order content in one Packing TU is allowed only under the current same-customer / delivery-address / priority / requested-delivery-window / Ship-To-country compatibility rules. Reject incompatible consolidation.

All content of one Packing TU must remain compatible with belonging to one Shipment. **Do not implement the P1-011 Shipment lifecycle here.** This ticket stops at the packing-side compatibility/handoff seam.

Optimization/limits:

- minimize Packing TU count without overpacking quantities;
- obey the Architect packing `maxWeight` limit;
- `current_content_volume` is informational and must **not** become an invented packing/repack blocking limit;
- do not invent category/temperature compatibility restrictions for v1 packing.

### I. Quantity integrity / overpacking

Never pack more good quantity than is eligible for the order/line contribution being packed. Repack/consolidation must preserve deterministic quantity ownership and must not double-settle picked quantities across source/target TUs.

## 4. Preserve P1-009 Direct Pack exactly

P1-009 is an upstream alternate path and is already owner-accepted.

Do not change these accepted guarantees:

- first-scan immutable `directPackDeclared`;
- same-TU multi-zone continuation without re-prompt;
- R67 replacement TU inheritance without re-prompt;
- qualifying Direct Pack auto `READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED` with PackUnit role and atomic line `PACKED`/`packedQty` without Packer action;
- non-qualifying/issueability-negative path stops at standard packing instead of falsely auto-packing;
- incomplete/SHORT paths do not falsely mark lines packed.

A qualifying P1-009 TU must not be forced back through a P1-010 Packer screen/action.

## 5. Domain / transaction / audit requirements

Use the existing Outbound domain/state-transition/audit infrastructure. Do not build a second state machine and do not bypass an existing audited transition seam with unaudited direct status mutation.

All writes must preserve:

- tenant / organization / warehouse scoping;
- actor identity and authorization;
- idempotency/replay safety for decisive mutations;
- transactionality across TU contents, source accounting, target contents, discrepancy/QC handoff and state transition events;
- locking/concurrency protection on shared mutable TU/order-line accounting.

If schema work is genuinely required, keep it additive/minimal/reversible and prove it against the approved remote Testing PostgreSQL. Never modify an accepted historical migration.

## 6. UI requirement — real Mercato/Packer path

Inspect and extend the existing accepted UI architecture; do not create a parallel product shell.

The decisive Packer actions must be available through rendered UI, including as applicable:

- identify/scan/select a `READY_TO_PACK` TU;
- see the system suggestion and eligibility/issueability facts needed for the decision;
- accept keep-same-TU;
- choose allowed repack mode;
- choose/create/select externally issuable target Packing TU(s);
- repack-all completion action;
- repack-by-SKU operator-selected SKU + quantity scanning;
- defer a low counted SKU without creating shortage;
- missing recheck + explicit confirmation;
- DAMAGED and unexpected/overage QC outcomes;
- consolidation selection and compatibility rejection;
- Supervisor approval when deviating from the WMS suggestion;
- completion state/accounting visible enough to prevent premature source `REPACKED`.

API/DB fixture setup may prepare preconditions, but decisive Packer actions claimed as UI proof must occur through rendered UI.

## 7. Decisive real PostgreSQL test matrix

Add/extend genuine PostgreSQL integration coverage for the real service/application path. At minimum prove:

1. keep-same-TU preserves `TU_NUMBER`, moves role to `PackUnit`, and records the valid `PACK_QUALIFIED -> PACKING_SEALED` path;
2. externally non-issuable repack target cannot be sealed;
3. repack-all does not terminalize source TU before complete accounting;
4. repack-by-SKU persists operator-selected SKU/quantity without a server-imposed next-SKU sequence assumption;
5. lower intermediate by-SKU count can be deferred with **no shortage created**;
6. explicit missing flow requires recheck + explicit confirmation before SHORT_PICKED recovery and does not block the source location;
7. DAMAGED quantity is excluded from good packed quantity, persisted/routed to QC, and triggers the correct shortage recovery without source-location block;
8. unexpected/overage SKU is routed to QC and does **not** create shortage;
9. source TU reaches `REPACKED` only when all expected quantity is accounted as packed/QC/confirmed-missing;
10. compatible consolidation is allowed under current R60 criteria and `maxWeight`;
11. incompatible cross-order consolidation is rejected;
12. content volume alone does not block a valid packing/consolidation decision;
13. deviation from WMS suggestion requires valid Warehouse Supervisor authorization and leaves an auditable reason/actor trail;
14. quantity integrity prevents overpacking/double settlement;
15. real rollback proof crosses a real write boundary, fails before commit, then a fresh independent read proves no partial packing/repack/discrepancy state leaked;
16. real concurrency/locking proof for the shared mutable packing/repack accounting path using independent overlapping transactions/connections and DB-side participant evidence where the operation can race.

Do not substitute mocks/Map/fake entity managers/simulated PostgreSQL errors for these claims.

## 8. Decisive real UI evidence

Use Playwright against the approved Testing runtime and real remote PostgreSQL. Build the smallest deterministic journey set that decisively covers P1-010. It must include at least:

1. **Keep same TU** — standard Packer action, same identity, final sealed PackUnit, fresh DB confirmation.
2. **Repack all** — rendered Packer action, valid externally issuable target, source completion gate and final source/target DB states.
3. **Repack by SKU — missing/defer** — operator controls SKU order; first low count is deferred with no shortage, then explicit missing flow performs recheck + confirmation and persists the correct recovery.
4. **DAMAGED + unexpected/overage** — rendered discrepancy actions; QC handoff persisted; DAMAGED affects missing-good recovery while unexpected/overage does not create shortage.
5. **Consolidation/approval/negative compatibility** — prove a compatible consolidation path and a decisive rejection/authorization guard for incompatible or suggestion-deviation behavior. Split into separate journeys if that makes the evidence more truthful/deterministic.

After decisive UI actions, use fresh independent PostgreSQL reads for persisted facts. Do not call an API directly as a substitute for the actor action you claim to have verified through UI.

Label automated browser evidence only `PLAYWRIGHT VERIFIED`.

## 9. Regression matrix

Because P1-010 touches the TU packing/issueability seam, rerun and record exact commands/output for:

- P1-008 backend TU identity/issueability coverage;
- accepted P1-008 real UI TU identity/issueability Playwright spec if present in the current accepted surface;
- P1-009 backend direct-pack suite (accepted baseline: 15/15);
- P1-009 Scanner direct-pack Playwright 4-journey suite against the frozen accepted Scanner head/runtime;
- P1-006/P1-007 targeted backend/picking regressions where the changed service path overlaps them;
- full `src/modules/wms_outbound` backend umbrella.

Run targeted Inbound/shared regressions **only** if the authorized P1-010 diff touches a shared primitive. Architecture-context is reference-only for deciding that compatibility surface.

If Scanner product code is not changed, keep its accepted implementation head frozen and run its accepted P1-009 regression from that head; do not manufacture a Scanner P1-010 commit merely to record test output.

## 10. Evidence contract

Create/update durable evidence:

`WMS_Outbound/05_EVIDENCE/P1-010_EVIDENCE.md`

Evidence must contain:

- `PLAYWRIGHT VERIFIED` label only for automated browser proof;
- exact final 40-character Mercato implementation head;
- exact final Scanner head, explicitly stating unchanged if no Scanner product change occurred;
- exact accepted bases/lineage and clean diff scope;
- safe real Testing PostgreSQL identity/provenance proof with no credentials/secrets;
- exact working directories and **exact commands actually executed** for every reported backend/Playwright suite;
- exact actual test output/counts and exact test titles matching checked-in source/output;
- runtime URLs/provenance used for Mercato/Packer and Scanner regressions;
- real concurrency evidence tied to actual DB participants if concurrency proof is claimed;
- real rollback evidence with fresh independent read;
- exact UI journeys performed and the persisted facts each proves;
- explicit statement whether any shared primitive changed and therefore whether Inbound regression was required;
- explicit statement that P1-011/P1-012+ scope was not implemented.

Do not call a pre-evidence WMS steering commit the final evidence commit; the supervisor will independently record the evidence commit after push.

Do **not** update `STATE.md`, current handover, or `.ai` steering to FINAL PASS. Acceptance belongs to the supervisor/owner after independent verification.

## 11. Hard scope exclusions

STOP before P1-011.

Do not implement:

- full Shipment creation/lifecycle/readiness or partial-shipment gating;
- Shipment operational UI beyond the minimal existing packing-side handoff/linkage seam if current model already requires one;
- Carrier selection/ranking;
- labels, print/reprint;
- ERP posting;
- manifest logic;
- shipping staging/dispatch confirmation;
- P2 cross-dock process expansion;
- P3 release expansion;
- P4 physical putback/disposition;
- unrelated Inbound changes.

R26 in this ticket is a **packing-to-Shipment compatibility/handoff seam only**. If implementing a requirement would require inventing the P1-011 Shipment lifecycle, STOP at the packing completion boundary and record the dependency instead of pulling P1-011 forward.

## 12. Push / stop contract

Before push, verify branch ancestry and diff scope from the accepted P1-009 bases.

Push:

1. Mercato `outbound/p1-010` with only required P1-010 implementation/tests;
2. Scanner only if a product change was genuinely required by the existing accepted product boundary; otherwise leave Scanner frozen;
3. `WMS_Outbound/main` durable P1-010 evidence.

Record exact current heads in evidence.

Then **STOP**. Do not start P1-011. Do not self-declare `FINAL PASS` or `HUMAN VERIFIED`.

This is execution shot **1/2**. If a material path fails after the pushed evidence, stop and let the supervisor independently verify it. Do not autonomously turn this into an unbounded remediation loop.