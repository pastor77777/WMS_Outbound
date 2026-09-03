# P1-009 — Direct Pack declaration and automatic sealing — Antigravity execution guide

**Status:** first authorized execution shot  
**Item:** 12/37  
**Authoritative project:** `pastor77777/WMS_Outbound`  
**Operating mode:** SAME existing Antigravity session; follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`; push implementation/evidence and STOP.

## 1. Mandatory startup

Before changing code, refresh the current local checkouts and read the current durable steering, not old chat history:

1. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md`
2. `WMS_Outbound/STATE.md`
3. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
4. `WMS_Outbound/07_IMPLEMENTATION_PLAN/TASK_CATALOG.md` — exact `P1-009` section
5. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` — P1 R15–R18 plus directly referenced R55/R62/R64/R65/R67/R68 where needed to interpret the current path
6. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` — `FR-P1-08`, `FR-P1-09`
7. `WMS_Outbound/01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` — `TC-001`, `TC-004`, `TC-005`
8. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-03.md`, `.ai/STATE.md`, `.ai/PLAN.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`
9. current installed `wms-outbound`, `architecture-context`, `fetch_me_prompt`, `operational-mode`; use `architecture-context` only for accepted shared/Inbound compatibility
10. current real-evidence contract before any DB/UI/runtime acceptance claim.

Current independently verified refs at guide creation:

- WMS_Outbound `main`: `bdd12ba35f7299a3faade0b69993eea17e260c89` before this guide commit.
- Devaxonic-WMS `main`: `76bd655361bd80d2f871cdf4a6cbe7794a27ae63`.
- Mercato accepted cumulative Outbound base: `outbound/p1-007` = `134db31381b4db726cd550abe6ecd4079ac21d8c`; this branch is 41 commits ahead of legacy `main` `60638c5812b493352c5e987e904230581309a096`, so do not branch P1-009 from legacy Mercato `main`.
- Scanner accepted P1-007 head: `b23325aae1c4f83b79d01b3650dbead3486a1041`; current Scanner `main` is exactly this commit.
- Accepted P1-007 durable WMS evidence commit remains `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8`.

If any current remote ref has advanced when you start, inspect lineage and preserve accepted cumulative Outbound work; do not silently reset or discard unrelated work.

## 2. Exact P1-009 authority

### Task Catalog objective

Implement immutable `directPackDeclared` at the architect-defined first scan and automatic qualification/sealing/line `PACKED` behavior when issue criteria allow.

Mapped authority:

- Architect: P1 R15–R18.
- Requirements: `FR-P1-08`, `FR-P1-09`.
- Acceptance: `TC-001`, `TC-004`, `TC-005`.
- Dependencies already accepted: `P1-006`, `P1-008`.
- Target: TU, `OutboundOrderLine`, Scanner picking.

### Normative rules to implement

1. **R16 / FR-P1-08 — first-scan declaration is binding.** When Picker scans the first Picking TU, the flow must allow the direct-pack declaration at that architect-defined first-scan point. Once picking begins, `directPackDeclared` for that TU cannot be changed. It remains binding when that same Picking TU continues through later PickTasks/zones. The operator is not asked again for the same TU.
2. **R16 + R67 — PICK_FULL continuation inherits declaration.** When the current Picking TU reaches `PICK_FULL` before the PickTask quantity is complete, the next Picking TU created/scanned for that same PickTask inherits the previous TU's `directPackDeclared`; do not ask again. Preserve the already accepted P1-006 continuation semantics and PickTask identity.
3. **R15 — preserve mass-capacity behavior.** P1-009 must not regress the accepted max-weight block / `PICK_FULL` behavior. Do not redesign TU capacity; P1-008 owns TU capacity/issueability primitives.
4. **R17 / FR-P1-09 — automatic direct-pack closure.** When `directPackDeclared = true` and the completed TU meets the architect issue thresholds, System WMS must automatically perform `TU READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED` and the contributing `OutboundOrderLine PICKED -> PACKED` transition without a Packer action or hidden manual packing step.
5. **Issue qualification is the already accepted P1-008 semantic.** Reuse current R64/R68 evaluation: external carrier type by `TUSetup.processUsage = EXTERNAL` qualifies by definition; otherwise the type must be `externalIssuable = true` and meet at least one lower threshold (`minIssueWeight` or `minIssueVolume`). Do not invent a second direct-pack-specific issueability algorithm.
6. **R65 remains the below-threshold operator exception where applicable.** For a non-EXTERNAL TU with `externalIssuable = true` that is below lower thresholds, the existing operator-force rule may apply only under its architect conditions (last TU for its OutboundOrder or near `slaDeadline`) and requires a persisted reason. It never overrides `externalIssuable = false`. Preserve this accepted P1-008 behavior; integrate direct-pack without duplicating/redefining it.
7. **R18 is not an invitation to build P1-010.** The Packer suggestion/override path belongs to the normal packing evaluation surface. P1-009 must preserve the rule that a Packer deviation from a system suggestion requires Warehouse Supervisor approval, but do not build repack/consolidation/discrepancy UX in this ticket.

Current source text wins over historical changelog prose or old guides if there is any discrepancy.

## 3. Hard scope boundary

IN SCOPE:

- persisted direct-pack declaration on the correct Outbound TU domain object;
- declaration capture at the first Picking TU scan in the real Scanner picking journey;
- server-authoritative immutability after picking begins;
- same-TU continuation without re-prompt;
- R67 next-TU inheritance for the same PickTask after `PICK_FULL`;
- direct-pack automatic qualification/sealing and `OutboundOrderLine` `PACKED` transition when the already accepted issue criteria pass;
- idempotent/replay-safe handling consistent with accepted picking correlation patterns;
- evidence and targeted regressions required below.

OUT OF SCOPE — STOP rather than absorb these:

- P1-010 repack, consolidate, discrepancy-resolution UI or full Packer workflow;
- new Shipment, Carrier, label, ERP POST, manifest or dispatch behavior;
- redesign of P1-006 picking, task assignment, zone continuation or shortage behavior;
- redesign of P1-008 TU identity, capacity, external-origin recognition or issueability thresholds;
- future P4 physical PutBack lifecycle;
- any Inbound behavior change. Inbound is CLOSED / REFERENCE; only targeted compatibility regression is allowed if a shared primitive is touched.

## 4. Implementation approach constraints

1. Start from the accepted cumulative Outbound implementation, not legacy Mercato `main`.
2. Inspect the accepted P1-006 Scanner/backend paths and P1-008 TU/issueability implementation before writing new logic. Extend the existing server-authoritative picking/TU transition seam; do not create a parallel direct-pack subsystem.
3. `directPackDeclared` must be durable and authoritative on the TU. If schema already contains an equivalent field from accepted work, reuse it. If a new column/migration is genuinely required, make it additive/reversible and follow the project's approved MikroORM migration path.
4. Do not trust a client-only disabled control as immutability. Backend/service rules must reject an attempted post-start change, including crafted/replayed requests.
5. Exact replay of the same first-scan intent may be idempotent; a conflicting later value must fail closed and leave the persisted declaration unchanged.
6. Automatic packing transitions must use the accepted transition/state service or equivalent authoritative seam; do not directly mutate statuses in a bypass that skips audit/domain-event/idempotency protections.
7. Only mark an OutboundOrderLine `PACKED` for the quantity/line contribution actually covered by the sealed direct-pack TU, respecting current multi-TU/multi-task relationships. Do not collapse unrelated lines or future shipment aggregation.
8. Preserve `TU_NUMBER`, accepted TU identity and current contents links when Picking TU becomes the sealed Packing TU.
9. Preserve P1-007 shortage paths. A shortage/non-complete path must not accidentally auto-seal or mark a line `PACKED` before the architect completion conditions are met.

## 5. Required tests and evidence in this first shot

This ticket includes persistence and real Scanner UI behavior, so the first shot must carry decisive real evidence rather than defer it.

### Backend / DB

Use the approved Testing PostgreSQL/runtime only, following `.ai/TESTING.md` and the current real-evidence contract.

Prove at minimum:

- first scan persists `directPackDeclared = true` and `false` correctly;
- after picking begins, a conflicting attempt to change the declaration is rejected and a fresh independent read shows the original value unchanged;
- same-value replay is safe/idempotent;
- same TU across the accepted multi-zone continuation is not re-declared and keeps the original value;
- after R67 `PICK_FULL` continuation, the newly created/scanned TU for the same PickTask inherits the declaration;
- qualifying direct pack performs the exact automatic transition chain to `PACKING_SEALED` and sets the relevant line to `PACKED` without a Packer action;
- non-qualifying / `externalIssuable = false` TU does not bypass accepted issueability guards;
- incomplete/SHORT_PICKED paths do not falsely auto-pack.

If a migration is required, prove it through the genuine project-approved MikroORM `Migrator` path. No hand-written shadow DDL, no fake EntityManager/Map, no simulated PostgreSQL error.

### Scanner / real UI

Add or extend Playwright against the real rendered Testing Scanner. The decisive Picker action must occur through the UI.

Required visible journeys:

1. Picker opens/receives an eligible picking task, scans the first Picking TU, chooses direct pack, performs the normal pick, and the completed qualifying TU reaches the automatic sealed result with persisted backend state. The UI must not insert a Packer step.
2. The same Picking TU continues across another accepted task/zone path without asking for direct pack again.
3. `PICK_FULL` continuation to another TU for the same PickTask inherits the declaration without asking again, if the current deterministic fixture can exercise this without destabilizing the suite.
4. At least one negative UI-visible guard: a path that is not issueable must not present a false successful seal/packed result.

Label automated browser evidence **PLAYWRIGHT VERIFIED**, never HUMAN VERIFIED.

### Regression matrix

Run the smallest decisive matrix that protects accepted dependencies and shared compatibility:

- P1-006 backend picking regression;
- P1-006 Scanner picking/multi-zone regression;
- P1-008 backend TU identity/capacity/issueability regression;
- P1-008 Scanner regression if the changed Scanner seam overlaps its accepted UI;
- P1-007 shortage regression for any touched picking completion/status seam;
- targeted shared Inbound/TU compatibility only if shared TU/entity/migration primitives changed;
- full `wms_outbound` umbrella if the project test harness supports it after targeted suites are green.

Record exact commands/counts and 40-char implementation heads in durable evidence.

## 6. Git / branch expectations

Create/continue bounded P1-009 implementation branches from accepted cumulative bases while preserving unrelated local work.

Preferred branch names:

- Mercato: `outbound/p1-009`, based on accepted cumulative Outbound head `134db31381b4db726cd550abe6ecd4079ac21d8c` unless a newer verified cumulative Outbound descendant exists when execution starts.
- Scanner: `outbound/p1-009`, based on accepted Scanner head `b23325aae1c4f83b79d01b3650dbead3486a1041` unless a newer verified cumulative Outbound descendant exists.

Do not reset legacy `main` branches to these refs. Do not force-push accepted branches.

Push implementation and tests. Then add/update durable P1-009 evidence in `WMS_Outbound/05_EVIDENCE/` with exact refs, commands, counts, real DB identity proof where applicable, and Playwright provenance.

## 7. Definition of Done for this shot

This first shot is complete only when all of the following are true:

- `directPackDeclared` is captured at the correct first-scan point and persisted;
- it is immutable after picking starts, including against direct backend calls;
- same-TU continuation and R67 next-TU inheritance behave exactly as current R16 requires;
- successful direct pack reaches `READY_TO_PACK -> PACK_QUALIFIED -> PACKING_SEALED` and relevant line `PICKED -> PACKED` automatically;
- no Packer/manual packing step is required for the qualifying direct-pack path;
- accepted R64/R65/R68 issueability semantics are reused, not duplicated or weakened;
- P1-006, P1-007 and P1-008 affected regressions are green;
- any shared TU change has targeted accepted Inbound compatibility proof;
- real Scanner evidence is PLAYWRIGHT VERIFIED;
- exact branch heads and evidence are pushed.

Do not self-declare FINAL PASS. Supervisor will independently fetch refs/diffs/evidence after owner returns `done`.

## 8. STOP condition

After pushing the P1-009 implementation/tests/evidence required above, STOP.

Do not start P1-010, carrier/shipment/manifest work, broad cleanup or opportunistic refactors. Do not ask the owner to paste logs. The owner should only need to reply `done`; the supervisor will independently verify Git refs, diffs and evidence.
