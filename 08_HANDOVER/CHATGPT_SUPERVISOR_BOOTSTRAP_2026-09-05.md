# ChatGPT Supervisor Bootstrap — WMS Outbound

**Date:** 2026-09-05 Europe/Warsaw
**Purpose:** durable bootstrap for a fresh ChatGPT supervisor chat while a new Codex session executes the next item.

## Current accepted checkpoint

- Project: WMS Outbound v1
- Progress: **21/37 FINAL PASS**
- Latest accepted item: **P2-002 — Owner Accepted**
- Final Mercato: `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- Final Scanner: `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`
- Evidence: `2074e2541b7c28cf3c6031cb48ad68901333625a`
- Canonical Testing PostgreSQL: Supabase project `yzonugcenguvmojwiihb`
- Local PostgreSQL is forbidden.

P2-002 accepted proof includes: architecture-aligned CON-03, P2-001 10/10, P2-002 dedicated 22/22, remaining Mercato 5 suites/41 tests, P1-005 Scanner current-UI 1/1, dedicated P2-002 Playwright 1/1 zero route mocks, and clean final worktrees.

## Active item

**P2-003 — RF crossdock sorting into outbound TUs — item 22/37.**

Codex must execute only:
`06_AGENT_GUIDES/P2-003_EXECUTION.md`

Frozen accepted bases:
- Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93`
- Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`
- P2-002 evidence `2074e2541b7c28cf3c6031cb48ad68901333625a`

Read also:
- `STATE.md`
- `08_HANDOVER/HANDOVER_CURRENT_2026-09-05.md`
- `08_HANDOVER/REMAINING_TEST_PLAN_AUDIT_2026-09-05.md`
- `05_EVIDENCE/EVIDENCE_STANDARD.md`
- Architect Source for P2/P3/P4 when needed.

## Supervisor/executor protocol

- Owner controls Codex/session mechanics.
- Cadence: `SHOT -> RECEIVE -> VERIFY EVIDENCE -> NEXT SHOT`.
- Do not self-accept executor output. Independently verify Git refs/diffs/evidence before PASS.
- Maximum two genuine attempts on the same material path; a second identical failure means STOP unless Owner explicitly authorizes a distinct narrow path.
- Do not silently switch executor/model/venue.
- Owner-facing executor prompts stay microscopic.
- FINAL PASS / Owner Accepted only after explicit owner acceptance.
- Inbound is CLOSED / REFERENCE; regress only genuinely touched shared primitives.

## Mandatory lessons from P2-002 for all remaining work

1. **Exact served runtime is evidence.** Source SHA alone does not prove what Testing serves. Before Playwright, prove build/generate/export/service state and relevant route presence.
2. Mercato route/product changes: repository-native full build including generate + canonical service restart before UI acceptance.
3. Scanner changes: refresh the canonical `scanner-testing.service` runtime/export before Playwright.
4. Scanner tests must wait for successful real warehouse resolution/selection before entering the module. `Choose mode` alone is not readiness.
5. Fixtures explicitly create org/tenant/warehouse/role/zone/policy/TUSetup/Sequence prerequisites. Never depend on ambient Testing defaults or on there being one warehouse.
6. Current tested SHA's `testID`/accessibility/current UI contract wins over historical selector/copy assumptions unless Architect Source requires exact copy.
7. On 404/5xx/API-null, investigate runtime/route/API/persistence before editing UI assertions. If API is correct but rendering is wrong, then fix UI.
8. Concurrency proof demonstrates Architect-required business outcome with genuine overlapping PostgreSQL operations. Do not turn incidental `pg_stat_activity`, `pg_blocking_pids` or one lock type into a requirement unless Architect Source says so.
9. Local PostgreSQL is forbidden. Use only canonical Supabase Testing.
10. Historical regression specs are not automatically authoritative after later accepted UI evolution. First compare current diff, current UI contract and fixture prerequisites.
11. Shared/Inbound regressions are relevance-based: mandatory only for actually touched Inventory/TU/warehouse/task-lock/orchestration primitives.
12. Design rendered Playwright together with the feature: real UI actions + real request evidence + persisted reconciliation. Do not bolt acceptance on after backend completion.
13. Task Catalog TC IDs are **staged traceability**, not permission to implement the whole full-E2E scenario early. Each item proves only the slice supported by its own FR/Architect references. Later dependent items/ACC close downstream slices.
14. Old evidence may be retained only if the relevant semantics/surface and evidence class remain truthful on the final code line. Never relabel stale evidence.

## Remaining-plan audit summary

Business architecture remains valid through P2-006, P3-001..003, P4-001..003, X-001/X-002 and ACC-001..004. The main correction is staged interpretation of mapped TCs.

Important examples:
- P2-003: TC-020/021 only RF/TU slice now; Shipment/dispatch closes later. TC-023 shortage/Supervisor is P2-004. TC-030 GR is P2-005. TC-035 PutBack/cancellation recovery is later.
- P2-004 owns shortage, DAMAGED, unexpected SKU, empty source, in-progress cancellation, residual/finalization and quantity conservation.
- P2-005 owns GR correlation/re-evaluation and posting gate.
- P2-006 owns common Shipment/carrier/ERP/manifest downstream and mixed-channel completeness.
- P3 authoritative `pickedQty` chooses P3 vs P4. Pre-confirm physical race returns to exact source without PutBackTask.
- P4 logical cancellation is immediate; physical recovery is asynchronous. PutBack only for `pickedQty > 0`. Invalid-location loop has no limit or automatic escalation.
- X-001/X-002 prove business exactly-once/correlation outcomes, not implementation diagnostics.
- ACC stages freeze exact code/runtime revisions; normal UI remains decisive, API/DB only setup/verification.

## P2-003 boundaries

Implement only the normal RF crossdock sorting path:
- task `ASSIGNED -> IN_PROGRESS` on first real scan;
- OOL `CREATED -> PICKING`, first line starts order `CREATED -> PACKING_IN_PROGRESS`;
- scan/confirm normal OK SKU/quantity into target Outbound TU(s);
- target TU created lazily at first placement;
- 1:1 valid GS1 source SSCC inherits TU_NUMBER/SSCC; otherwise/n:n use accepted P1-008 TUSetup/Sequence rules;
- physical-full target can be RF-sealed and same task continues in a new open target TU;
- completion fixes idempotent `confirmedQty <= plannedQty`;
- real Scanner 1:1 and n:n journey required.

Do NOT implement in P2-003:
- shortage/DAMAGED/unexpected SKU/empty source/cancellation recovery (P2-004);
- GR gate (P2-005);
- Shipment/ERP/manifest (P2-006);
- P3/P4;
- Return Receipt;
- Demo/Prod.

## When Codex returns

Do not trust its PASS summary alone. Independently verify:
1. final Mercato/Scanner SHAs and merge bases;
2. diff is bounded to P2-003;
3. no P2-004+ behavior leaked in;
4. dedicated PostgreSQL scenarios and exact counts map the P2-003 requirement slice;
5. real 1:1 + n:n Scanner Playwright uses zero route mocks and real warehouse readiness;
6. runtime provenance: exact served Mercato and Scanner revisions/routes;
7. target TU identity/continuity, lazy creation, full/seal continuation and persisted confirmed quantities;
8. no Allocation and no regression of accepted P2-002 assignment invariants;
9. relevant P1-008/TU/shared regressions if shared TU primitives changed;
10. canonical evidence only after all gates are green;
11. clean worktrees.

If verified, report `SUPERVISOR VERIFIED PASS` and ask Owner for explicit P2-003 acceptance. Do not mark Owner Accepted yourself.
