# P1-014 — ERP Shipment POST, error state and safe retry

**Execution status:** authorized first implementation shot for `P1-014`  
**Catalog item:** `17/37`  
**Prerequisite:** `P1-013` is `FINAL PASS / Owner Accepted`  
**Scope type:** implementation + evidence; **NOT acceptance steering**  
**Workflow:** follow `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`

## Mandatory fresh-session startup

This task must run in a **fresh Antigravity session**.

Canonical startup:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
/home/ubuntu/.local/bin/agy-pl
```

Never start with bare `agy`.

Before changing anything, refresh and read current Git state in this order:

1. `Devaxonic-WMS/AGENTS.md`
2. `Devaxonic-WMS/.ai/STATE.md`
3. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-04.md`
4. `Devaxonic-WMS/.ai/TESTING.md`
5. `Devaxonic-WMS/.ai/OPERATIONS.md`
6. `WMS_Outbound/STATE.md`
7. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-04.md`
8. `WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`
9. this guide
10. current P1 Task Catalog / Architect KROK 11A / state transitions / requirements / acceptance scenarios
11. current Mercato `wms_outbound` and reusable `wms_orchestration` implementation
12. current Mercato/Scanner/WMS exact Git refs

If current steering does not say **16/37 FINAL PASS** with `P1-013` Owner Accepted and `P1-014` next, STOP.

## Frozen accepted refs

Preserve these exact accepted inputs:

- P1-013 final Mercato base: `5e6b70aa81afd28fe3217e4aad216e8a6482a769`
- P1-013 durable evidence: `826b9c477fa86a44a93606265868730e4570ff90`
- Scanner frozen reference: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Create/use Mercato branch:

`outbound/p1-014`

It must start from the exact accepted P1-013 Mercato SHA above. Before implementation prove the merge base and parentage. Do not rebase, merge, squash or transplant unrelated work.

Scanner is not authorized for P1-014.

## Product objective

Implement the P1 ERP Shipment posting gate with durable, observable and idempotent behavior:

- eligible Shipment enters `POSTING_PENDING`,
- accepted ERP response advances to `POSTED`,
- explicit ERP rejection advances to `POSTING_ERROR`,
- real Warehouse Supervisor can explicitly retry `POSTING_ERROR → POSTING_PENDING`,
- explicit Supervisor give-up from `POSTING_ERROR → CANCELLED` remains available,
- duplicate/retried requests and responses cannot double-apply WMS business effects.

The normal flow must remain compatible with P1-013 label/own-transport behavior and stop before P1-015 manifest lifecycle.

## Authority grounding

Business behavior comes from current Architect/canon, not from legacy code or this guide if wording conflicts.

Primary grounding:

- P1 KROK 11A / R35–R38
- `FR-P1-18`
- `FR-P1-19`
- `INT-04`
- `INT-05`
- `CON-05`
- state transitions:
  - `LABEL_GENERATED` / `OWN_TRANSPORT → POSTING_PENDING` — `ShipmentPostingRequested`
  - `POSTING_PENDING → POSTED` — `ShipmentPosted`
  - `POSTING_PENDING → POSTING_ERROR` — `ShipmentPostingRejected`
  - `POSTING_ERROR → POSTING_PENDING` — `ShipmentPostingRetried`
  - `POSTING_ERROR → CANCELLED` — `ShipmentCancelled`

Acceptance scope:

- `TC-001`
- `TC-007`
- `TC-008`

## Required business behavior

### 1. Posting eligibility and gate

Every Shipment must pass the ERP posting gate before P1-015 may add it to a CarrierManifest.

Only these P1-014 start states are eligible:

- `LABEL_GENERATED`
- `OWN_TRANSPORT`

Initial posting is a System WMS orchestration action, not a new Supervisor approval step.

Starting the durable posting intent must advance the Shipment to `POSTING_PENDING` through the authoritative transition mechanism and record a durable correlation/idempotency identity.

Do not implement `POSTED → IN_MANIFEST`; that belongs to P1-015.

### 2. Accepted ERP response

Only a positive accepted ERP response may advance:

`POSTING_PENDING → POSTED`

The response must correlate to the exact Shipment/posting intent. Duplicate acceptance/replay must be harmless and must not create duplicate business transition facts, duplicate successful posting effects or duplicate attempt history entries for the same logical response.

### 3. Explicit ERP rejection

A **real explicit rejection response** from the ERP adapter may advance:

`POSTING_PENDING → POSTING_ERROR`

Persist an operations-safe structured reason/correlation sufficient to distinguish at least:

- a content/data rejection attributable to ERP-side data/configuration, where a later retry can reuse unchanged WMS content after ERP repair;
- a content/data rejection attributable to WMS-side data, where correction must precede retry.

Do not expose secrets, raw credentials, stack dumps or unrestricted remote payloads as operational error details.

The packed physical TU content and `OutboundOrderLine PACKED` facts must not be mutated by rejection/retry handling.

### 4. Timeout / no response boundary

Architect explicitly leaves timeout/no-response as a technical incident outside the defined P1 business path.

Therefore:

- do **not** silently map timeout/network failure/no response to `POSTING_ERROR`;
- do not invent a new Shipment business state;
- do not invent a new product-visible automatic cancellation path;
- if existing shared orchestration has generic transport retry/technical failure metadata, it may be reused additively, but it must not change the Shipment business state contrary to the Architect source.

Evidence must distinguish an ERP business rejection from a transport/timeout incident.

### 5. Manual retry

Retry from `POSTING_ERROR` is a separate, real Warehouse Supervisor decision:

`POSTING_ERROR → POSTING_PENDING`

Authorization must be server-authoritative. Client/request-body role flags must not elevate permissions.

Persist a new durable attempt while preserving previous attempt history. The Shipment business identifier/correlation model must make retries idempotent and auditable.

### 6. Supervisor give-up / cancellation boundary

The specifically defined exception is allowed:

`POSTING_ERROR → CANCELLED`

and requires real Warehouse Supervisor authority.

Do **not** generalize this into cancellation from `POSTING_PENDING` or `POSTED`.

The accepted cancellation boundary is:

- ordinary Shipment cancellation is possible only before first entry to `POSTING_PENDING`,
- once posting has started, direct WMS cancellation is blocked,
- the explicit post-rejection `POSTING_ERROR → CANCELLED` Supervisor path remains the named exception,
- Return Receipt/correction after this boundary is outside P1-014.

### 7. Integration contract honesty

Inspect current Mercato integrations and approved Testing configuration before designing the ERP adapter.

If a real approved ERP Testing endpoint/contract already exists and is authorized, use it according to current operational rules.

If no such endpoint/credential/wire contract exists:

- implement a typed ERP adapter boundary and deterministic Testing/test implementation sufficient to exercise accepted/rejected/technical outcomes;
- do **not** fabricate a vendor URL, secret, ERP credential or undocumented wire protocol;
- do **not** claim `REAL ERP` evidence;
- label results honestly as adapter/contract/stub verification while PostgreSQL persistence remains real.

Do not use `wms_orchestration_mock_postings` as authoritative Shipment ERP business truth merely because it exists for another integration concern. Reuse the durable event-fact/retry foundation where semantically valid, add P1-014-specific persistence where required, and keep ownership explicit.

## Persistence and orchestration

Current code already contains reusable `wms_orchestration` durable event facts and append-only retry attempts with idempotency constraints. Prefer additive reuse over creating a parallel generic retry framework.

P1-014 must durably represent, at minimum:

- exact Shipment identity and organization/tenant scope,
- logical posting correlation/idempotency key,
- posting request intent/state,
- attempt number/history,
- accepted/rejected/technical outcome metadata,
- safe structured rejection details,
- timestamps sufficient for operations/evidence.

Any new migration must be additive/reversible and applied only to approved Testing PostgreSQL.

No Production/Demo changes.

## Exactly-once and concurrency requirements (`CON-05`)

Schema-level uniqueness alone is not enough if concurrent operations can create duplicate durable posting facts or duplicate state transitions.

The final PostgreSQL suite must include genuine overlapping operations with independent database/ORM sessions where concurrency is relevant.

At minimum prove:

### A. Concurrent initial posting request

From one real eligible Shipment (`LABEL_GENERATED` or `OWN_TRANSPORT`):

1. operation A starts posting and holds the relevant Shipment/idempotency transaction boundary open;
2. independent operation B attempts the same logical posting before A commits;
3. prove actual PostgreSQL-side serialization/contention where locking is the mechanism (`pg_stat_activity` / `pg_blocking_pids` or equivalent exact-session proof), or prove the alternative exact database mechanism if implementation does not use blocking;
4. after both settle, fresh reads prove:
   - Shipment is non-regressively `POSTING_PENDING` (or later accepted state only if the test intentionally includes response handling),
   - one logical durable initial posting intent exists,
   - one `ShipmentPostingRequested` business transition exists,
   - no duplicate business posting effect exists.

### B. Duplicate/concurrent ERP acceptance

For the same correlated posting intent, process duplicate acceptance/replay safely and prove by fresh database reads:

- Shipment reaches `POSTED` once,
- exactly one `ShipmentPosted` transition fact exists,
- no duplicate successful business effect is recorded,
- duplicate response is replay/no-op rather than regression/error.

If true overlap is possible in the response handler, exercise it with independent overlapping operations and database-side evidence.

### C. Retry history

After one explicit rejection:

- Shipment is `POSTING_ERROR`,
- rejection reason is persisted safely,
- a real Supervisor retry produces a new attempt and returns to `POSTING_PENDING`,
- after acceptance the Shipment becomes `POSTED`,
- previous rejected attempt remains append-only/auditable,
- retry does not duplicate the initial Shipment business identity or packed contents.

## Transaction rollback

Include a real PostgreSQL rollback test that fails after a meaningful write/flush inside the P1-014 transaction and proves with a fresh independent read that no partial posting transition/attempt/outbox state survives.

Do not simulate rollback only with mocks.

## Authorization / tenancy

Prove server-authoritative organization/tenant isolation for posting records and Supervisor actions.

At minimum:

- non-Supervisor cannot trigger `POSTING_ERROR → POSTING_PENDING` retry;
- non-Supervisor cannot trigger the explicit `POSTING_ERROR → CANCELLED` give-up path;
- request-body role spoofing cannot elevate authority;
- cross-tenant Shipment/attempt access fails closed.

## Minimum PostgreSQL behavior suite

Final-head tests must cover at least:

1. `LABEL_GENERATED → POSTING_PENDING` creates one durable correlated posting intent.
2. `OWN_TRANSPORT → POSTING_PENDING` follows the same ERP gate without label requirement.
3. invalid starting Shipment status rejects posting.
4. accepted correlated ERP response yields `POSTED` and `ShipmentPosted` exactly once.
5. explicit structured ERP rejection yields `POSTING_ERROR` without touching packed physical contents.
6. ERP-side content rejection can be retried unchanged after external repair simulation.
7. WMS-side content rejection requires correction-before-retry path consistent with existing authorized correction capability; do not invent broad data-edit powers.
8. timeout/no-response does not become business `POSTING_ERROR`.
9. non-Supervisor retry fails closed.
10. Supervisor retry `POSTING_ERROR → POSTING_PENDING` creates a new append-only attempt.
11. non-Supervisor cancellation from `POSTING_ERROR` fails closed.
12. Supervisor give-up `POSTING_ERROR → CANCELLED` succeeds only from that exact state.
13. direct cancellation from `POSTING_PENDING` / `POSTED` is rejected.
14. concurrent duplicate initial posting is exactly-once.
15. duplicate/concurrent accepted response is exactly-once.
16. real rollback leaves no partial durable posting state.
17. organization/tenant isolation for posting intent/attempt/response.
18. `POSTED → IN_MANIFEST` is not implemented by P1-014.

This list is a minimum behavior inventory, not permission to fake tests around implementation details.

## Regression requirements

At final Mercato head rerun:

- P1-014 PostgreSQL suite,
- P1-013 PostgreSQL **15/15**,
- P1-012 PostgreSQL **14/14**,
- P1-011 PostgreSQL **18/18**,
- FND-002 state-transition invariant **77/77** or the exact current accepted count if legitimate P1-014 transition coverage increases it; explain any changed count from committed test diff,
- existing `wms_orchestration` tests affected by shared changes.

If P1-014 modifies shared orchestration behavior rather than only adding a P1-specific consumer, identify and run the relevant accepted Inbound/shared regressions required by project rules.

Do not reduce existing assertions/counts to make the task pass.

## Minimum rendered UI / Playwright evidence

Use the canonical Testing Mercato URL and record the **exact served Mercato Git revision**.

At minimum exercise normal product surfaces for:

### Journey A — happy posting

- start with eligible external-carrier `LABEL_GENERATED` Shipment,
- posting enters `POSTING_PENDING`,
- approved Testing ERP adapter response accepts,
- UI reflects `POSTED`,
- no manifest behavior is introduced.

### Journey B — explicit ERP rejection

- eligible Shipment posts,
- deterministic explicit rejection is returned,
- UI surfaces `POSTING_ERROR` plus safe structured reason,
- packed Shipment/TU facts remain intact.

### Journey C — Supervisor retry

- from Journey B state, real authorized Supervisor performs retry,
- Shipment returns to `POSTING_PENDING`,
- accepted response advances to `POSTED`,
- attempt history/retry result is visible enough for the architect-required operational behavior.

### Journey D — authorization/boundary

- unauthorized user cannot perform retry/give-up,
- no user action can add the Shipment to a manifest in this task,
- no timeout/provider fiction is presented as ERP business rejection.

Automated browser evidence is `PLAYWRIGHT VERIFIED`, never `HUMAN VERIFIED` unless the owner explicitly performs/accepts a human walkthrough.

## Evidence file

Executor may create/update only:

`WMS_Outbound/05_EVIDENCE/P1-014_EVIDENCE.md`

Do not edit WMS steering/state/handover/traceability/task catalog/workflow.

Evidence must contain literal final-head facts, not reconstructed summaries:

- exact accepted P1-013 base and exact final P1-014 Mercato SHA,
- exact branch/parent/merge-base/commit count,
- Git-derived final file stats,
- migration/schema impact,
- exact adapter mode/provenance: real approved ERP vs explicit Testing stub/contract seam,
- request/response correlation and idempotency design,
- literal P1-014 PostgreSQL test inventory/count,
- decisive PostgreSQL exactly-once/concurrency evidence with exact session/PID or equivalent database proof,
- exact rollback proof,
- literal regression outputs/counts,
- literal Playwright journeys and count,
- exact served Testing Mercato runtime SHA,
- exact Scanner frozen SHA,
- exact final clean `git status --short` for relevant worktrees,
- explicit exclusions: no P1-015 manifest lifecycle, no P1-016 settlement, no Scanner, no Prod/Demo, no Return Receipt,
- no `FINAL PASS` / `Owner Accepted` claim by executor.

Never put secrets in evidence.

## Scope exclusions

Do not implement:

- P1-015 `CarrierManifest` lifecycle or `POSTED → IN_MANIFEST`,
- P1-016 line/order/Inventory settlement,
- Crossdock-specific GR gating (`P2-005`),
- Return Receipt / post-dispatch correction process,
- external carrier API behavior,
- Scanner changes,
- Production/Demo changes,
- unrelated refactors or cleanup.

## Git / STOP boundary

1. Work only on `outbound/p1-014` from exact accepted base `5e6b70aa81afd28fe3217e4aad216e8a6482a769`.
2. Push the P1-014 implementation/test commit(s).
3. Push `05_EVIDENCE/P1-014_EVIDENCE.md` to `WMS_Outbound/main` only after final-head verification.
4. Report exact 40-char Mercato, WMS evidence and frozen Scanner SHAs.
5. STOP.

Do not merge/delete branch. Do not start P1-015. Do not update steering. Do not claim acceptance.

Owner-facing result on success should remain microscopic: `done` plus exact refs/test counts only.