# P1-013 — WMS label generation and pre-manifest loading carrier correction — execution guide

**Execution status:** authorized first implementation shot for `P1-013`  
**Catalog item:** `P1-013` — item `16/37`  
**Executor:** NEW Antigravity session from canonical `Devaxonic-WMS` checkout  
**Scope type:** execution artifact only  
**Workflow:** follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md` from current `WMS_Outbound/main`

## Mandatory startup

```bash
cd /home/ubuntu/git/Devaxonic-WMS
/home/ubuntu/.local/bin/agy-pl
```

Never launch bare `agy`.

Before implementation:

1. sync `WMS_Outbound/main` without overwriting unrelated work,
2. read current `Devaxonic-WMS/AGENTS.md`, `.ai/STATE.md`, current Outbound handover, `.ai/TESTING.md`, `.ai/OPERATIONS.md`,
3. read current `WMS_Outbound/STATE.md`, current handover, `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`, Task Catalog and this guide,
4. read exact P1 Architect/source, requirements and acceptance scenarios for P1-013,
5. inspect current `Devaxonic-mercato/outbound/p1-012` and current label/print primitives before editing.

## Frozen accepted starting point

Owner accepted `P1-012` after supervisor review.

- accepted P1-012 Mercato head: `5019a20be14549ff8cbbf25af5bc61c56888e9e1`,
- accepted P1-012 WMS evidence commit: `b28f59e7ff41ac6d0a3be4b841410650bc5acd8b`,
- accepted P1-011 base: `20887f2d74928cf69f447fdd6af20a612f38387c`,
- Scanner accepted/frozen reference: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`,
- Devaxonic-WMS steering head before this acceptance write: `680f20c66a5aad9dc02e60bbf7843b725906100f`,
- WMS_Outbound head before this P1-013 guide: `b28f59e7ff41ac6d0a3be4b841410650bc5acd8b`.

Do not reinterpret or re-open accepted P1-012 behavior unless a genuine P1-013 regression requires a bounded fix and the evidence makes that explicit.

## Mercato branch rule

Create/use task branch:

`outbound/p1-013`

It must start from exact accepted P1-012 Mercato SHA:

`5019a20be14549ff8cbbf25af5bc61c56888e9e1`

If an existing `outbound/p1-013` branch has the wrong base or unrelated work, STOP and report it. Do not silently rebase, merge or transplant unrelated changes.

## Scanner boundary

No Scanner implementation is authorized for P1-013 unless the current authoritative UI ownership proves the required dispatch/print action is already Scanner-owned. Default target is Mercato. Do not create a Scanner label flow merely because a printer/scanner device exists.

## Task Catalog grounding

`P1-013 — WMS label generation and pre-manifest loading carrier correction`

Objective:

- generate/print label from WMS-owned data,
- support Architect-permitted carrier correction before manifest close,
- do not introduce carrier Label API.

Architect/reference:

- P1 R33–R34,
- P1 R35–R36,
- P1 exception: label error / carrier rejection boundary.

Requirements:

- `FR-P1-17`,
- `FR-P1-18`,
- `FR-P5-11`.

Acceptance scenarios:

- `TC-001`,
- `TC-007`,
- `TC-066`.

Dependency:

- `P1-012` — satisfied and accepted.

Target components:

- Shipment label service,
- Mercato print UI.

## Architect truth to implement

### Label generation gate

For external-carrier flow:

- label generation occurs only after the Shipment has an approved carrier and is `CARRIER_SELECTED`,
- successful generation advances `Shipment CARRIER_SELECTED -> LABEL_GENERATED`,
- an EXTERNAL TU / missing-`maxVolume` case may generate a label only after the Dispatcher choice has received real Supervisor approval and the Shipment reached `CARRIER_SELECTED`,
- `OWN_TRANSPORT` skips label generation.

### Label ownership and data

The label is WMS-owned and generated locally from data WMS already has, including the relevant Shipment / Packing TU information, Carrier description and delivery address.

Do not call any external carrier API. Do not create provider acceptance, electronic rejection, provider label lifecycle, tracking acquisition, webhook, polling or external retry semantics.

Prefer existing internal rendering/printing primitives when they are suitable. Keep business authority in WMS Outbound rather than in a generic carrier-provider integration.

### Version-1 error boundary

Architect v1 explicitly has no technical carrier-label failure/rejection mode.

- Do not invent `LABEL_ERROR`, provider rejection, carrier acceptance, retry queue, dead-letter flow or external error state.
- Ordinary local UI/print failures may be surfaced as ordinary technical/UI errors without creating a new business state machine.
- Do not claim a simulated provider rejection as Architect evidence.

### Print / reprint

Provide the normal Mercato user action required by the Task Catalog to generate/print the WMS label.

If the current local printing surface supports reprint, reprint must remain a local print action and must not create a new carrier/provider lifecycle or regress Shipment status.

### Carrier correction before manifest close

Architect rule R35:

- if a loading problem is discovered before `CarrierManifest.CLOSED`, a real `Warehouse Supervisor` may change the Shipment Carrier,
- reason is not mandatory under the accepted carrier override rule,
- the carrier correction must not automatically regenerate or reprint the existing label,
- after `CarrierManifest.CLOSED`, carrier change is forbidden.

P1-015 owns the full `CarrierManifest` lifecycle and is NOT part of this task. Therefore:

- reuse current Shipment/state/manifest boundary data that already exists,
- implement only the narrow correction guard that can be truthfully enforced with current persisted state,
- do not create/close manifests or absorb P1-015,
- if the exact post-CLOSED boundary cannot yet be decisively exercised because P1-015 persistence does not exist, prove the pre-close correction behavior now and leave the full post-CLOSED integration guard to P1-015; do not fabricate a parallel manifest model.

Carrier correction after label generation must not automatically regenerate/reprint the label.

## Minimum persistence

Persist only what is needed to make label generation/printing authoritative and observable, for example a WMS-owned label artifact/payload or label-generation metadata and local print metadata appropriate to the existing architecture.

The persisted model must make it possible to prove:

- which Shipment the label belongs to,
- that it was generated locally by WMS,
- the generation state/timestamp/revision or equivalent durable fact,
- the normal print action where required,
- repeated operations are non-regressive/idempotent where the user action can be retried.

Do not make legacy/provider `carrier_shipments.label_url` / `label_data` the business authority unless the exact current architecture proves it is already the correct WMS-owned target. Prefer a narrow adapter over duplicating an existing valid local print primitive.

## Minimum backend tests

Use the approved Testing PostgreSQL environment from `/etc/mercato-localhost.env` whenever persistence/transaction claims are made. No SQLite or ad-hoc substitute. No Prod/Demo.

Prove at minimum:

1. `CARRIER_SELECTED` is the external-carrier label generation gate,
2. non-selected / pending carrier cannot generate label,
3. `OWN_TRANSPORT` skips/rejects label generation as designed,
4. generated label is based only on WMS-owned persisted Shipment/Packing TU/Carrier/address data,
5. successful generation persists authoritative label evidence and transitions to `LABEL_GENERATED`,
6. repeated generation/retry is non-regressive and does not duplicate irreversible business effects,
7. local print/reprint does not call an external carrier API and does not create a new business state,
8. EXTERNAL missing-`maxVolume` path cannot generate until Supervisor approval completed P1-012,
9. real Supervisor carrier correction is allowed at the currently enforceable pre-close boundary,
10. carrier correction does not automatically regenerate/reprint an existing label,
11. unauthorized/non-Supervisor carrier correction remains blocked by the accepted server-authoritative P1-012 RBAC path,
12. no new carrier rejection / label-error business state exists,
13. P1-012 carrier-selection regression remains green,
14. P1-011 Shipment readiness/grouping regression remains green.

If a multi-write transaction is introduced, include a genuine post-write/flush failure and fresh independent read proving rollback.

If concurrency can create duplicate label artifacts or duplicate durable print facts, include real overlapping PostgreSQL operations and database-side evidence. Do not manufacture concurrency claims when the operation is purely local/read-only.

## Minimum Playwright evidence

Use the final served `outbound/p1-013` revision and normal rendered Mercato UI.

At minimum:

### Journey A — normal label generation/print

- start from a real `CARRIER_SELECTED` Shipment,
- use the visible UI action,
- generate/print the local WMS label,
- verify visible label/print result,
- verify fresh persisted `LABEL_GENERATED` / label evidence.

### Journey B — pending/manual approval gate

- use the P1-012 EXTERNAL missing-`maxVolume` path or equivalent real pending selection,
- prove label action is unavailable/rejected before Supervisor approval,
- complete the real Supervisor approval,
- then prove label generation is available.

### Journey C — carrier correction after label generation

- generate the label,
- as a real Supervisor change the carrier at the currently enforceable pre-close boundary,
- prove the Carrier changed,
- prove no automatic label regeneration/reprint occurred.

### Journey D — no provider rejection mode

Prove the UI contains no invented provider-accept/reject workflow. Do not simulate an external carrier rejection API.

Automated UI evidence label: `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED`.

## Evidence artifact

Create/update candidate:

`WMS_Outbound/05_EVIDENCE/P1-013_EVIDENCE.md`

Do not mark progress FINAL PASS. Do not update STATE/handover from the executor.

Evidence must include:

- exact accepted P1-012 base `5019a20be14549ff8cbbf25af5bc61c56888e9e1`,
- exact final pushed P1-013 Mercato SHA,
- lineage/compare proof,
- files changed and schema/migration impact,
- exact local label ownership/rendering design,
- proof there is no external carrier API/provider acceptance/rejection lifecycle,
- decisive PostgreSQL results for persistence/rollback/concurrency claims actually made,
- fresh Playwright results from the served final revision,
- P1-012 and P1-011 regression results,
- Scanner status/reference if inspected,
- clean git status,
- explicit exclusions and any deferred P1-015 boundary that cannot yet be truthfully exercised.

Testing credentials are designated test data. Use the project's existing Testing configuration. Do not rotate Testing credentials and do not create credential-broker detours.

## Hard exclusions

No:

- external carrier Label API,
- provider acceptance/rejection workflow,
- tracking-number acquisition unless already purely local and explicitly required for the label data by current source,
- ERP POST / `POSTING_PENDING` / `POSTED` implementation (P1-014),
- CarrierManifest creation/closure/handover implementation (P1-015),
- final inventory/order settlement (P1-016),
- Scanner changes by default,
- Prod/Demo changes,
- unrelated refactors,
- merge/delete of task branches,
- progress/state/handover edits by executor,
- claim of FINAL PASS.

## Two-strikes rule

If the same material failure class occurs twice, STOP and report the blocker instead of widening scope.

## STOP boundary

When the bounded implementation is complete:

1. push `Devaxonic-mercato/outbound/p1-013`,
2. push candidate `P1-013_EVIDENCE.md`,
3. report exact refs,
4. STOP.

Do not merge. Do not advance to P1-014. Do not claim acceptance.

Owner-facing response must remain microscopic: `done` on success; otherwise at most 5 lines with blocker + refs.
