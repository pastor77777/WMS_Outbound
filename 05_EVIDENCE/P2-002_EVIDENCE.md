# P2-002 — Crossdock 1:1 implementation evidence

## Scope and revisions

- Mercato: `50b27fdd0c9b495ab612ce458bc90e65428ecb93` (unchanged, clean).
- Scanner: `6796b70aff9ab53d27a0b36d0764ccc83f4b0440`; merge base with Scanner `origin/main`: `b23325aae1c4f83b79d01b3650dbead3486a1041`.
- Canonical Testing PostgreSQL: Supabase project `yzonugcenguvmojwiihb`.
- Evidence class for the rendered journeys below: **PLAYWRIGHT VERIFIED**. This record does not claim Human Verified, Owner Accepted, or final acceptance.

## Runtime recovery

The frozen Mercato source contained both the standard Picking and Crossdock `request-task` routes. Before recovery, the canonical `mercato-localhost.service` served the standard route as HTTP 401 but Crossdock as HTTP 404. A repository-native full `yarn build` (including `yarn generate`) regenerated the API route registry; it contains both routes. Restarting only the canonical service recovered the served build. The service then returned HTTP 200 for `/login` and HTTP 401 (non-404 unauthenticated route-presence proof) for both:

- `POST /api/wms_outbound/picking/request-task`
- `POST /api/wms_outbound/cross-dock/request-task`

Classification: **rebuild-only stale generated/build/runtime routing**. No Mercato source or route-registration change was made.

## Backend evidence retained from the frozen final Mercato revision

- Corrected CON-03: PASS, including its architecture-aligned proof.
- P2-001: 10/10 PASS.
- P2-002 dedicated real PostgreSQL suite: 22/22 PASS.
- Remaining mandatory Mercato regressions: 5 suites / 41 tests PASS.

These unchanged Mercato suites were not rerun because the final Mercato SHA did not change.

## Rendered Scanner proof

Against the recovered canonical runtime and the final Scanner source:

- `e2e/p1-005-real-scanner-assignment.spec.ts`: **1/1 PASS**, zero route mocks.
- `e2e/p2-002-crossdock-assignment.spec.ts`: **1/1 PASS**, zero route mocks.

The P2-002 journey uses real login and waits for the successful warehouse-resolution response before choosing Crossdock. It arms and awaits the real Crossdock `POST /api/wms_outbound/cross-dock/request-task` response before UI assertions.

- Operator A: HTTP 200; received the deterministic first `CD-A-<fixture-id>` task. Scanner visibly showed the task number, source Inbound TU, `5.000000` planned quantity, assigned status through the `StatusBadge` `code` contract, no zone selector/requirement, and the explicit P2-003 boundary text.
- Operator B: HTTP 200; received the distinct deterministic `CD-B-<fixture-id>` task, never operator A's task.
- Blocked operator: HTTP 400; received and visibly displayed the real active-warehouse-task rejection.

The test independently confirms persisted CrossDockPickTask ownership/status/source/quantity for both assignments, two active task locks for A/B, and an organization/tenant-scoped Allocation count unchanged before versus after the fixture. This supports Crossdock planning semantics: no Allocation, lazy target TU, one binding to one OOL to one CrossDockPickTask, and exact planned quantity. It does not exercise P2-003 sorting, scans, target-TU creation, or completion.

The prior P1-005 historical selector mismatch was harness drift, not a P2-002 product regression.

## Final cleanliness

- Mercato worktree: clean.
- Scanner worktree: clean after the committed final corrective.
