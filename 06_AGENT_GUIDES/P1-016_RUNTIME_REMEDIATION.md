# P1-016 — Testing runtime remediation and final closeout

**Purpose:** resolve only the remaining Testing Mercato runtime blocker for the already-implemented and DB-proven P1-016, then finish fresh Playwright, final evidence and push.

This is **not** a new implementation pass and **not** a steering task.

## Preserved local checkpoint

Preserve current local Mercato checkpoint exactly:

`4ba4e38cb40c4c7219b4f4837295a563289a796e`

Accepted P1-015 base remains:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

Scanner remains frozen:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

WMS_Outbound baseline for this remediation is the current `main` containing the prior P1-016 execution/closeout/concurrency guides.

## Already completed before this blocker

Preserve as completed unless a later product-code change invalidates final-SHA evidence:

- P1-016 dedicated PostgreSQL suite: **25/25**;
- Race A: real PostgreSQL concurrency/locking proof completed;
- Race B: real PostgreSQL concurrency/locking proof completed;
- real rollback proof completed;
- final Mercato build for `4ba4e38cb40c4c7219b4f4837295a563289a796e` completed successfully.

Do not restart product implementation or redesign the concurrency harness.

## Exact blocker

After successful build of exact local Mercato SHA:

`4ba4e38cb40c4c7219b4f4837295a563289a796e`

canonical Testing Mercato did not come back on `localhost:3009` after restart:

- no serving process;
- port 3009 unavailable;
- observed startup log empty.

This blocks fresh P1-016 Playwright 6/6, therefore final evidence and push are also blocked.

## Canonical runtime authority

Follow current `Devaxonic-WMS/.ai/TESTING.md` and `.ai/OPERATIONS.md`.

Canonical Testing Mercato is:

`http://localhost:3009` -> `https://devaxonic-test.info-start.com.pl`

Existing VPS service route is:

`mercato-localhost.service`

Do **not** substitute Demo, port 3000, an ad-hoc second server, another executor/runtime environment, or a different branch to manufacture browser evidence.

## 1. Diagnose the real systemd failure first

Before changing product code, inspect the existing canonical service and capture enough non-secret detail to identify why it exits or fails to start:

- confirm exact Mercato worktree HEAD is `4ba4e38cb40c4c7219b4f4837295a563289a796e`;
- inspect `systemctl status mercato-localhost.service --no-pager`;
- inspect `systemctl cat mercato-localhost.service` to confirm `WorkingDirectory`, `ExecStart`, environment-file routing and user;
- inspect the service journal for the failed start (`journalctl -u mercato-localhost.service` for the relevant current boot/start window);
- inspect whether any process is actually listening on 3009;
- verify the successful build artifacts expected by the service still exist in the canonical Mercato worktree;
- do not print secret env values.

An empty application log is not sufficient diagnosis if systemd/journal contains the real failure.

## 2. Fix classification

Classify the cause before modifying anything.

### A. Operational/runtime wiring defect

Examples: stale service process state, wrong working directory, missing expected build path after restart, service command mismatch, permissions/ownership of generated build files, environment-file not being loaded, stale PID/process condition.

If this is the cause:

- fix only the minimum canonical Testing runtime/service issue;
- do not change P1-016 product behavior;
- do not introduce a replacement launcher/server;
- do not touch Demo/Prod;
- restart `mercato-localhost.service` through the existing canonical route;
- prove service is active and `http://localhost:3009/login` returns HTTP 200.

### B. Genuine product/runtime defect introduced by P1-016

If the exact final SHA itself cannot boot under the canonical service because of a real product defect:

- make only the smallest P1-016 product correction required to boot correctly;
- create a new local final Mercato commit;
- this new SHA becomes the only final SHA;
- rerun all final-SHA-sensitive P1-016 evidence invalidated by the code change, including P1-016 PostgreSQL and all mandatory regressions affected by the change, before browser evidence;
- do not broaden scope.

### C. External canonical Testing dependency genuinely unavailable

If the service cannot start because a required canonical Testing dependency is unavailable and the executor cannot correct it within the authorized environment, STOP with exact non-secret blocker evidence. Do not substitute another environment.

## 3. Exact-runtime proof before Playwright

Before final browser evidence, prove all of the following:

- canonical Mercato worktree HEAD = exact final 40-character P1-016 SHA;
- the build being served belongs to that exact SHA;
- `mercato-localhost.service` is active/running;
- port 3009 is listening;
- `http://localhost:3009/login` returns HTTP 200;
- public Testing route `https://devaxonic-test.info-start.com.pl` reaches the same intended Testing runtime;
- no Demo/port-3000 substitution occurred.

Record only non-secret runtime identity/provenance in evidence.

## 4. Fresh P1-016 Playwright 6/6

Once exact final runtime is healthy, run the existing required P1-016 rendered-browser acceptance against canonical Testing.

Required journeys remain:

A. one-manifest final settlement;
B. first of two manifests remains nonterminal;
C. later manifest completes remaining line/order/customer state;
D. cancellation blocked at post/manifest boundary with actionable reason;
E. whole-order cancellation is atomically rejected when one line blocks;
F. duplicate/repeated final confirm is UI-stable and exactly-once in persistence.

Required clean result:

- **6/6 passed**;
- **0 unexpected**;
- **0 flaky**;

or the runner's equally explicit clean six-journey equivalent.

The decisive user actions must occur through the rendered Mercato UI; API/DB may only prepare fixtures and verify persisted outcomes.

## 5. Mandatory final regression completeness

Before final evidence/push, ensure the exact final SHA has the complete mandatory regression set required by `P1-016_EXECUTION.md` / `P1-016_CLOSEOUT.md`.

If all mandatory regressions were already run on the unchanged exact final SHA, preserve them. If not, run the missing ones now.

At minimum final evidence must contain exact counts for:

- P1-016 PostgreSQL: current reported **25/25** or greater;
- P1-015: **21/21**;
- P1-014: **18/18**;
- P1-013: **15/15**;
- P1-012: **14/14**;
- P1-011: **18/18**;
- FND-002 state transitions: **77/77**;
- FND-002 transaction simulation: **8/8**;
- required shared Allocation/Inventory/FND-003 regressions for primitives touched by P1-016.

A product-code change after any final regression invalidates affected final-SHA evidence and requires rerun.

## 6. Final evidence and push

Only after runtime proof + clean fresh Playwright + complete final regression evidence:

Create/update only:

`05_EVIDENCE/P1-016_EVIDENCE.md`

Include all requirements from the prior P1-016 guides, especially:

- accepted base, final Mercato SHA and lineage/compare scope;
- P1-016 **25/25** (or final greater count);
- Race A exact-PID PostgreSQL proof;
- Race B exact-PID PostgreSQL proof;
- rollback fresh-read proof;
- final mandatory regression counts;
- exact canonical runtime/service proof tied to final SHA;
- literal fresh Playwright 6/6 result;
- frozen Scanner SHA;
- clean final worktree identities;
- explicit exclusions;
- no secrets;
- no FINAL PASS / Supervisor acceptance / Owner Acceptance claim.

Then:

1. push final Mercato P1-016 commit(s) to the authorized P1-016 branch;
2. push P1-016 evidence to WMS_Outbound;
3. report only final Mercato SHA, WMS evidence SHA, P1-016 PostgreSQL count, mandatory regression counts, Playwright count and frozen Scanner SHA;
4. STOP.

## Early-stop rule

Do not stop merely after repairing the service or getting HTTP 200. Continue through Playwright, evidence and push in this same run.

Stop early only if a genuine blocker still prevents canonical Testing runtime or required evidence after one focused remediation attempt. Preserve the exact local SHA and report the exact non-secret blocker.

## Hard exclusions

- no P2/P3/P4 implementation;
- no Return Receipt;
- no external carrier API;
- no Scanner product changes;
- no Demo/Prod writes;
- no STATE/handover/canon/catalog/traceability/AGENTS/.ai steering changes;
- no alternate ad-hoc Testing server used to bypass `mercato-localhost.service`;
- no Codex permission/approval configuration changes in this shot.
