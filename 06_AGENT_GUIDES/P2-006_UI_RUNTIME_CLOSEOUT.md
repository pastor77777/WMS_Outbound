# P2-006 — UI/runtime closeout after backend matrix green

Continue the SAME P2-006 item from the current canonical VPS state. Do not reset/reconstruct/rebase/clean Mercato or Scanner. Preserve the current green backend implementation and test state.

Current executor checkpoint to preserve:

- P2-006 dedicated PostgreSQL: 20/20 PASSED;
- P2-005: 19/19 PASSED;
- P1-011: 18/18 PASSED;
- P1-012: 14/14 PASSED;
- P1-013: 15/15 PASSED;
- P1-014: 18/18 PASSED;
- P1-015: 21/21 PASSED;
- P1-016: 25/25 PASSED;
- P2-002: 22/22 PASSED;
- P2-003: 8/8 PASSED;
- P2-004: 16/16 PASSED;
- Scanner remains frozen at accepted P2-005 SHA `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- current product change is the narrow crossdock-safe manifest settlement compatibility fix; do not broaden it.

## This shot has only four goals

1. Create the missing dedicated real rendered Mercato Playwright spec required by `P2-006_EXECUTION.md`, preferably `P2-006-crossdock-shipment-downstream-ui.spec.ts`.
2. Run the required P2-006 journeys against the canonical Mercato runtime with zero Playwright route mocks/interception/substitution.
3. Complete repository-native Mercato typecheck/build and canonical runtime proof.
4. Only after 1-3 are green: commit/push the final Mercato `outbound/p2-006`, keep Scanner frozen unless a concrete defect proves otherwise, write `05_EVIDENCE/P2-006_EVIDENCE.md`, and STOP for supervisor verification.

## UI proof discipline

Use deterministic DB setup/cleanup only for fixture creation and persisted reconciliation. Product actions that the guide requires as rendered UI proof must be performed through the real Mercato UI and real HTTP/backend behavior.

Reuse the accepted P1/P2 UI patterns where possible; do not invent a second crossdock dispatch UI.

The dedicated spec must prove the main-guide P2-006 rendered journeys, including the common Shipment/dispatch pipeline and the mixed STANDARD + CROSSDOCK `allowPartialShipment=false` guard. It must include persisted reconciliation proving the same common Shipment/CarrierManifest/posting/settlement objects are used and that crossdock settlement creates no fake Allocation or standard Inventory decrement.

No `.route(` / `page.route` / response substitution is allowed.

If the new UI spec exposes a real product defect, fix only that concrete defect and rerun only P2-006 plus the directly invalidated accepted suite(s). Do not rerun or edit the already-green full backend matrix unless a subsequent product change actually invalidates it.

## Typecheck OOM handling

The prior plain command `yarn workspace @open-mercato/app typecheck` exhausted the Node heap. Treat this first as execution/resource plumbing, not as a product failure.

After the UI spec is stable, retry typecheck once with the repository unchanged and an explicit heap allowance:

```bash
cd /home/ubuntu/git/Devaxonic-mercato
NODE_OPTIONS=--max-old-space-size=8192 corepack yarn workspace @open-mercato/app typecheck
```

If this succeeds, continue normally.

If it still fails with heap/OOM symptoms, do not edit product code or tests to address it. Preserve the exact log and report the resource/tooling blocker; use the existing WMS host-probe pattern only if needed, without changing business scope.

For build, use the repository-native full generate/build contract and the existing canonical Mercato build/runtime procedure. Do not rely on a stale Turbo cache without fresh `.mercato/next` production manifests.

## Final runtime proof

Before Playwright final acceptance proof, establish the exact canonical runtime:

- `mercato-localhost.service` active/running;
- exact final Mercato revision served from canonical repo;
- port 3009 listener belongs to canonical Mercato/Next process;
- `/login` HTTP 200;
- `.mercato/next/routes-manifest.json` and `.mercato/next/required-server-files.json` exist and are non-empty.

Scanner stays untouched/frozen unless a concrete P2-006 defect requires otherwise.

## Commit/evidence gate

Do not push incomplete work and do not write acceptance evidence before real UI/runtime proof is green.

Once green:

- commit the minimum final Mercato diff;
- push exact `outbound/p2-006` head;
- verify ancestry from accepted P2-005 Mercato SHA `069f02d4c5c9b345b688b838eb685be02206afbd`;
- keep Scanner at `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` unless genuinely changed;
- write exact final SHAs, test counts, 20/20 mapping, zero-mock UI proof, runtime proof and clean worktree status into `05_EVIDENCE/P2-006_EVIDENCE.md`;
- evidence must not self-declare FINAL PASS / Owner Accepted / Human Verified;
- STOP. Do not start P3-001.
