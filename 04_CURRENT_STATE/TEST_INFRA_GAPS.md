# Test-infrastructure gaps — dedicated PostgreSQL suites missing entity registration

**Status:** Gaps 1 and 2 FIXED 2026-09-06 (owner-authorized dedicated test-infra correction, out-of-scope for and separate from the P4-003 item that discovered them). Remaining full-directory sweep failures noted under "Scope note" are still open/undiagnosed.

Each `*-postgres.integration.test.ts` suite in `apps/mercato/src/modules/wms_outbound/services/__tests__/` constructs its own isolated `MikroORM` instance with its own explicit `entities: [...]` array (no shared fixture registry). When a later item extends a shared code path to also query a newer entity, every suite that exercises that path — not just the suites the authoring item's own evidence happened to rerun — must add that entity to its own list, or any test that reaches the new query fails with `MetadataError: Metadata for entity <X> not found`. This is a structural gap in how these suites are composed, not a single one-off typo.

## Gap 1 — `p3-002-postgres.integration.test.ts` missing `WmsOutboundPutBackTask`

- **File:** `apps/mercato/src/modules/wms_outbound/services/__tests__/p3-002-postgres.integration.test.ts`
- **Introduced by:** P4-002 (commit `32c31ac07`, 2026-09-06 12:31 UTC), which extended `pick-task-service.ts`'s shared active-task guard to also check `WmsOutboundPutBackTask` (see `pick-task-service.ts:669` and `:755`, `checkOperatorActiveTask`/task-assignment path).
- **Symptom:** `MetadataError: Metadata for entity WmsOutboundPutBackTask not found`, thrown from `pick-task-service.ts:754` inside `txEm.findOne(WmsOutboundPutBackTask, ...)`, surfacing in at least:
  - `P3-002 ... › 10e. Lifecycle & P3 boundary: first formal pick confirmation transitions OutboundOrderLine to PICKING and thereafter P3 release is blocked/routed to P4`
  - one additional `10e`-adjacent case in the same run (2 failures total observed in a 62-test run of this suite, 2026-09-06, discovered while rerunning `p1-004`/`p1-007`/`p3-001`/`p3-002` as extra P4-003-correction regression coverage).
- **Root cause:** the file's own `entities: [...]` array (starts ~line 61) imports `WmsOutboundPhysicalReturnHandoff` but never `WmsOutboundPutBackTask`, even though it exercises `createPickTaskService`, whose shared-task-guard path was extended by P4-002 to query that entity.
- **Fix:** added `WmsOutboundPutBackTask` to this file's `entities` array, imported from `../../data/entities` alongside the other Outbound entities already imported there.
- **Why not fixed at discovery:** discovered while verifying the P4-003 handoff-resolution correction (`P4-003_COMPLETION_CORRECTION.md`); confirmed pre-existing and unrelated to that diff (predates the P4-003 branch entirely, and P4-002's own accepted evidence never listed `p3-002` in its "regression suites missing the new entity" remediation, unlike `p4-001`/`p3-003`/`p1-005`/`p1-006`/`p2-002`, which it did fix). Fixing it was a one-line, genuinely unrelated test-infra correction outside the P4-003 correction guide's authorized scope, so it was deferred pending separate owner authorization.
- **FIXED:** 2026-09-06, Devaxonic-mercato commit `9a656bf9e` (branch `outbound/p4-003`), as a dedicated owner-authorized test-infra-only correction (no other files touched). Rerun result: `17/17 PASS` against the remote Supabase `DevAxonic_Platform` test DB.

## Gap 2 — `p1-003-detail-api-postgres.integration.test.ts` missing `WmsOutboundPutBackTask`

- **File:** `apps/mercato/src/modules/wms_outbound/services/__tests__/p1-003-detail-api-postgres.integration.test.ts`
- **Introduced by:** P4-002, which added a `putback-tasks-section` read-only observability block to `apps/mercato/src/modules/wms_outbound/api/customer-orders/[id]/route.ts` (`GET` handler), querying `WmsOutboundPutBackTask` by `customerOrderId`.
- **Symptom:** `MetadataError: Metadata for entity WmsOutboundPutBackTask not found`, thrown from `customer-orders/[id]/route.ts:132` (`em.find(WmsOutboundPutBackTask, { customerOrderId: order.id, ... })`), inside this test file's own call to the real `GET` handler.
- **Root cause:** this file's `entities` array (declared inline in `beforeAll`) never includes `WmsOutboundPutBackTask`, even though it imports and directly invokes the real `GET` route handler that P4-002 extended.
- **Fix:** added `WmsOutboundPutBackTask` (from `../../data/entities`) to this file's `entities` array.
- **Why not fixed at discovery:** same reasoning as Gap 1 — discovered incidentally (this time during an initial, later-abandoned attempt to run the entire `wms_outbound/__tests__/` directory as a superset regression check for P4-003; see `05_EVIDENCE/P4-003_EVIDENCE.md` "Note on full-directory run"), pre-existing, unrelated to the diff under test, deferred pending separate owner authorization.
- **FIXED:** 2026-09-06, Devaxonic-mercato commit `9a656bf9e` (branch `outbound/p4-003`), as the same dedicated owner-authorized test-infra-only correction as Gap 1 (no other files touched). Rerun result: `1/1 PASS` against the remote Supabase `DevAxonic_Platform` test DB.

## Scope note

The initial full-directory sweep (`apps/mercato`, `npx jest src/modules/wms_outbound/services/__tests__/`, 2026-09-06) reported **10 failed suites / 32 failed tests out of 572** before it was abandoned in favor of the precedent-matched targeted regression set (see `05_EVIDENCE/P4-002_EVIDENCE.md` and `P4-003_EVIDENCE.md` for the exact targeted-suite lists actually used for acceptance). Only the two gaps above were individually root-caused; the remaining failures from that sweep were not diagnosed and are **not** confirmed to be entity-registration gaps of this same shape — they may include other pre-existing issues. Do not assume the rest are automatically safe to ignore; a future full-directory audit item should enumerate and fix all of them together (with `--forceExit` — see `feedback_jest_postgres_force_exit_mandatory` operating memory) rather than one-at-a-time inside unrelated feature items.
