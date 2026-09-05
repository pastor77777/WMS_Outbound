# P1-016 — Playwright evidence closeout

**Purpose:** finish only the missing P1-016 rendered-browser evidence, then produce final durable evidence and push. This is not a new product implementation pass and not a steering task.

## Preserved current state

Preserve current local Mercato checkpoint:

`e423f6e5d6749b73776807a5d832a936f760085e`

Accepted P1-015 base remains:

`f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4`

Scanner remains frozen:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Current reported completed evidence before this blocker:

- canonical `mercato-localhost.service` active;
- `http://localhost:3009/login` HTTP 200;
- public Testing `/login` HTTP 200;
- P1-016 PostgreSQL **25/25**;
- fresh Race A and Race B real PostgreSQL lock proofs;
- rollback proof;
- mandatory regressions: **9 suites, 187/187**;
- existing P1-015 rendered harness: **6/6**, but it is NOT valid P1-016 evidence.

Do not reuse the P1-015 six-journey result as P1-016 evidence. Its Journey E explicitly proves the old P1-015 boundary where P1-016 settlement is absent, so relabeling it would be false.

## Exact remaining blocker

There is no dedicated P1-016 Playwright spec implementing the six P1-016 journeys required by `P1-016_EXECUTION.md`.

The missing work is therefore test-harness/evidence work, not proof that product behavior is absent.

## 1. Discover and reuse the accepted local Playwright harness

In the current local Mercato worktree, locate the actual accepted P1-015 six-test spec and the Playwright config/fixture helpers it uses.

Do not invent a parallel runner if the existing project-native runner can execute P1-016.

Reuse existing helpers for login, deterministic fixture setup, cleanup and DB/API verification where truthful.

Do not modify the P1-015 spec to change its historical assertions. Create a separate P1-016 spec.

## 2. Create one dedicated P1-016 Playwright spec

Add the minimum bounded test-only file(s) needed to cover exactly these six rendered journeys against canonical Testing Mercato:

A. **One-manifest final settlement**
- prepare an eligible one-manifest order;
- perform the decisive final confirmation through the rendered Mercato UI;
- visibly verify final settlement/status;
- persistence verification may confirm Inventory/Allocation/line/order/customer final state.

B. **First of two manifests remains partial/nonterminal**
- prepare one line/order split across two Shipments/manifests;
- confirm the first manifest through rendered UI;
- verify line/order/customer remains correctly nonterminal and partial arithmetic is persisted.

C. **Later manifest completes remaining state**
- continue the same bounded scenario or a deterministic equivalent;
- confirm the remaining manifest through rendered UI;
- verify line/order/customer reaches the correct terminal state and exact settlement arithmetic.

D. **Cancellation blocked at hard boundary**
- through rendered Supervisor UI attempt cancellation at Shipment `POSTING_PENDING`-or-later or CarrierManifest `CLOSED`-or-later boundary;
- verify actionable rejection/blocked reason;
- verify no forbidden cancellation side effect persisted.

E. **Whole-order cancellation is atomic**
- prepare a CustomerOrder with at least one blocking line and another otherwise cancellable line;
- submit whole-order cancellation through rendered UI;
- verify the whole request fails as one unit;
- verify no partial cancellation side effect on other lines.

F. **Duplicate/repeated final confirm is UI-stable and exactly-once**
- perform/repeat the final confirm through rendered UI in the supported way;
- verify stable UI outcome;
- verify persisted business effects remain exactly once (settlement identity, Inventory effect, `reservedQty`, `shippedQty`, terminal events).

Requirements for all six:

- decisive intended-actor action must be through the real rendered UI;
- API/DB may prepare fixtures and verify persisted state but may not substitute for the decisive UI action;
- use canonical Testing only (`localhost:3009` / public Testing route as appropriate);
- no Demo/Prod;
- no Scanner changes;
- no fake/mocked browser evidence;
- do not weaken assertions merely to obtain green tests.

## 3. Final SHA handling

Adding the P1-016 Playwright spec changes the Mercato Git SHA even if product code is unchanged.

Commit the test-harness change as a bounded P1-016 test/evidence commit. The resulting 40-character SHA becomes the new final Mercato SHA.

Before final browser evidence:

1. confirm diff from `e423f6e5d6749b73776807a5d832a936f760085e` is test/harness-only unless a genuine product defect is discovered;
2. build the new exact final SHA;
3. restart canonical `mercato-localhost.service`;
4. prove the service is active and `localhost:3009/login` returns HTTP 200;
5. prove the served runtime is built from the exact new final SHA.

Do not bypass canonical systemd/runtime routing.

## 4. Rebind mandatory final-SHA regression evidence

Because the Mercato SHA changes when the new Playwright spec is committed, rerun the mandatory final-SHA suites required by P1-016 on that exact final SHA, even if the only diff is test-only.

Required final counts must include at least:

- P1-016 PostgreSQL: **25/25 or greater**;
- P1-015 PostgreSQL: **21/21**;
- P1-014 PostgreSQL: **18/18**;
- P1-013 PostgreSQL: **15/15**;
- P1-012 PostgreSQL: **14/14**;
- P1-011 PostgreSQL: **18/18**;
- FND-002 state transitions: **77/77**;
- FND-002 transaction simulation: **8/8**;
- relevant accepted shared Allocation/Inventory/FND-003 suites required by the execution guide.

Preserve the already-obtained Race A/Race B/rollback proof if the final diff is strictly Playwright-test-only and does not touch product/concurrency/transaction code. If any such product code changes, regenerate the affected decisive DB evidence on the new final SHA.

## 5. Run the dedicated P1-016 rendered suite

Run only the new P1-016 six-journey spec against the canonical exact-final-SHA Testing runtime.

Required result:

- **6/6 passed**;
- **0 unexpected**;
- **0 flaky**;

or the runner's equally explicit clean equivalent.

Do not report the P1-015 6/6 as this result.

If a genuine P1-016 product defect is exposed by the new spec, make only the smallest product correction, create a new final SHA, then rerun all final-SHA-sensitive evidence required by the execution guide before proceeding.

## 6. Durable final evidence

After clean P1-016 Playwright 6/6, create/update only:

`05_EVIDENCE/P1-016_EVIDENCE.md`

It must include all required evidence from `P1-016_EXECUTION.md`, `P1-016_CLOSEOUT.md`, concurrency remediation and runtime remediation, including:

- accepted base and exact final Mercato SHA;
- merge base and compare scope;
- literal P1-016 PostgreSQL **25/25 or greater** result;
- literal Race A exact-PID PostgreSQL lock proof;
- literal Race B exact-PID PostgreSQL lock/no-lost-update proof;
- rollback fresh-independent-read proof;
- mandatory regression commands/counts on exact final SHA;
- relevant shared Allocation/Inventory regression counts;
- canonical runtime/service proof tied to exact final SHA;
- exact path/name of the dedicated P1-016 Playwright spec;
- literal P1-016 Playwright **6/6**, 0 unexpected, 0 flaky result;
- frozen Scanner SHA;
- clean final worktree identities;
- explicit exclusions;
- no secrets;
- no `FINAL PASS`, Supervisor acceptance or Owner Acceptance claim.

## 7. Push and STOP

Only after all final evidence is complete:

1. push final Mercato P1-016 commit(s), including the dedicated P1-016 Playwright spec, to the authorized P1-016 branch;
2. push `05_EVIDENCE/P1-016_EVIDENCE.md` to WMS_Outbound;
3. report only:
   - final Mercato SHA,
   - WMS evidence SHA,
   - P1-016 PostgreSQL count,
   - mandatory regression aggregate/counts,
   - P1-016 Playwright count,
   - frozen Scanner SHA;
4. STOP.

Do not stop after merely creating the spec or after merely getting 6/6. Continue through final evidence and push in the same run unless a genuine blocker prevents it.

## Hard exclusions

- no P2/P3/P4 implementation;
- no Return Receipt;
- no external carrier API;
- no Scanner product change;
- no Demo/Prod writes;
- no STATE/handover/canon/catalog/traceability/AGENTS/.ai steering changes;
- no Codex permission/approval configuration changes in this shot;
- no relabeling of P1-015 Playwright evidence as P1-016 evidence.
