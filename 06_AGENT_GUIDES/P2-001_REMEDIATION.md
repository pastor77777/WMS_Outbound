# P2-001 — verification remediation

**Shot:** first corrective attempt after initial P2-001 execution  
**Scope:** close only the objective verification/evidence gaps below; do not broaden P2-001.

## Frozen inputs

- Accepted P1-016 Mercato base: `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574`.
- Current P2-001 Mercato head to preserve and amend from: `a262b135617c6f84d125e6539c59ae1586ca4ae3` on `outbound/p2-001`.
- Current P2-001 evidence commit: `dad4121a5cb554723669433f135bef7571df8237`.
- Scanner remains frozen: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.
- Authoritative execution contract remains `06_AGENT_GUIDES/P2-001_EXECUTION.md`; this file narrows the corrective shot only.

Do not rebase/squash/transplant accepted history. Do not start P2-002.

## Supervisor findings to close

The initial implementation lineage/scope is clean, but evidence is insufficient for acceptance.

### A. Required PostgreSQL scenario coverage is incomplete

`P2-001_EXECUTION.md` section 11 requires a dedicated real PostgreSQL suite covering all 20 listed scenarios. The current suite has four test blocks and does not prove all mandatory cases.

Close every missing case and map each required scenario explicitly in the suite/evidence. In particular, prove at minimum:

- wrong source status rejected;
- ASN declaration is the source basis, not a physical-count confirmation;
- non-CANCELLED coverage from another/future fulfillment channel reduces demand eligibility;
- CANCELLED OOL requiredQty does not reduce demand eligibility;
- source-eligibility treatment for active planned, completed confirmed and completed damaged quantities is compatible with the future P2 task lifecycle and never returns confirmed/damaged quantity to eligibility;
- exact `min(sourceEligibleQty, demandEligibleQty)` arithmetic;
- demand disappearing between earlier Inbound qualification and `IN_CROSS_DOCK` gives deterministic zero match;
- organization/tenant isolation;
- accepted P1 ATP/planning semantics remain non-regressive.

Do not inflate counts with trivial assertions; scenario coverage must be substantive.

### B. Source lifecycle seam must not re-eligible consumed/damaged quantity

The current implementation uses a binding seam with statuses including `ACTIVE`, `RELEASED`, `CONSUMED`. Reconcile this seam against the authoritative formula from the execution guide:

`sourceEligibleQty = ASN declared qty - active plannedQty - completed confirmedQty - completed damagedQty`.

Prove by implementation + real PostgreSQL tests that future completion/consumption cannot cause already-confirmed or damaged source quantity to become eligible again. If the current `ACTIVE`-only deduction permits that, correct the model/service minimally now. Do not create P2-002 CrossDockPickTask objects.

### C. CON-03 evidence is not yet decisive enough

Keep the real independent-session race, but additionally capture PostgreSQL-side serialization/lock evidence tied to the actual competing participants, as required by section 10 of the execution guide.

Evidence must show the authoritative DB mechanism, not only the final application result. Record exact participant/session/lock facts sufficient to demonstrate serialization and no double coverage.

### D. Mandatory Inbound regression is currently red

Current evidence records the accepted Inbound cross-dock result integration suite as `1/3 passed, 2 failed`.

Determine the truth without changing accepted Inbound business semantics:

1. reproduce the exact suite on the accepted P1-016 base and on the current/final P2-001 head under the same approved Testing conditions;
2. if P2-001 causes the regression, fix P2-001;
3. if the failure is identical on the accepted base, prove that fact explicitly and identify the stale fixture/runtime/test-harness cause;
4. where a non-business test-fixture/harness correction is clearly required to restore the accepted regression, keep it minimal and explain why it does not change Inbound semantics;
5. final P2-001 evidence must not call a red mandatory regression PASS.

Do not modify Inbound qualification/deintegration/transport behavior.

### E. Evidence precision

Update only `05_EVIDENCE/P2-001_EVIDENCE.md` with:

- new final Mercato SHA and compare scope from the accepted P1-016 base;
- exact dedicated P2-001 PostgreSQL scenario count and mapping to all mandatory section-11 cases;
- literal CON-03 PostgreSQL serialization evidence;
- exact focused P1-002/P1-003/FND-003 suite result counts, not only declared case counts;
- exact Inbound base-vs-final regression result and disposition;
- frozen Scanner SHA;
- explicit exclusions and clean worktree identities;
- no FINAL PASS / Supervisor acceptance / Owner acceptance claim.

## Hard exclusions

- no P2-002 planning/task/order objects;
- no Scanner changes;
- no Inbound business-semantics changes;
- no GR gate, P3/P4, Return Receipt, Prod/Demo;
- no STATE/handover/Task Catalog/Canon/Architect/traceability changes;
- no `AGENTS.md`, credentials or unrelated `.ai/*` changes.

## STOP

After closing the above gaps:

1. push amended `outbound/p2-001`;
2. push updated `05_EVIDENCE/P2-001_EVIDENCE.md`;
3. report exact Mercato SHA, evidence SHA, dedicated P2-001 PostgreSQL count, CON-03 result, exact regression counts, Inbound base-vs-final disposition and frozen Scanner SHA;
4. STOP.

Do not claim acceptance and do not start P2-002.
