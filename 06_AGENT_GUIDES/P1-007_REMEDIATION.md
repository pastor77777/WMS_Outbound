# P1-007 Remediation — Sixth Override (Two Blockers Only)

Owner explicitly overrides STOP again for one final **evidence/migration-focused P1-007 remediation**.

Use the **same existing Antigravity session**. Do not restart, replace, or create another session.

## FIRST: synchronize locally

1. Go to the existing local `WMS_Outbound` checkout.
2. Fetch remote and safely fast-forward/pull current `main`, preserving unrelated local work.
3. Verify this LOCAL file is current:
   `06_AGENT_GUIDES/P1-007_REMEDIATION.md`
4. Read this LOCAL file completely.
5. Refresh current Architect/canon plus current `fetch_me_prompt`, `operational-mode`, `REAL_EVIDENCE_CONTRACT.md`, `.ai/TESTING.md`, and `.ai/OPERATIONS.md` where available.
6. Fix **only the two blockers below**.

## Current bases

Preserve these exact current heads as the base:

- Mercato `outbound/p1-007`: `7b3aad79dda997b7ec23c0c69b492feb0af63efe`
- Scanner `main`: `b23325aae1c4f83b79d01b3650dbead3486a1041`
- Current P1-007 evidence: `0342b86c623c11f9ab02d5b30c2c4b0f4082c374`

All product behavior from the fifth override is considered good and must be preserved:

- SHORT_PICKED canonical replay derives authoritative `outboundOrderId` and same-key/same-payload replay succeeds;
- R46 stores true zero commercial/execution quantities and releases hard reservation with physical-return handoff;
- R44 enforces both location and warehouse-wide hard-reservation bounds, including pre-PickTask hard reservations;
- six Mercato Supervisor UI outcomes;
- Scanner replacement continuation;
- strict PostgreSQL concurrency/rollback proof;
- exact current Mercato/Scanner SHA reporting.

Do **not** redesign those paths.

---

## 1. Real reversible migration proof — no copied DDL and no destructive fake DOWN

### Current defect

`Migration20260902160000_wms_outbound_p1_007_remediation.ts` has a real `up()` / `down()`, but the current test does not execute the migration implementation. It manually reproduces the DDL.

Worse, the current test makes its fake DOWN pass by executing broad deletes of every row with `ordered_quantity = 0` / `required_qty = 0`. That is not an acceptable rollback strategy and does not prove the real migration is reversible.

The real `down()` also blindly restores `CHECK (> 0)`, which will fail when legitimate R46-cancelled zero-quantity rows exist. The migration contract already allowed DOWN to restore prior constraints **only when existing data is compatible**, so this must be explicit and non-destructive.

### Required migration behavior

Keep the UP semantics: only these two constraints change from `> 0` to `>= 0`.

Make DOWN safe and explicit:

- before altering constraints, detect whether any incompatible zero/negative rows exist in the affected tables;
- if incompatible rows exist, DOWN must **fail fast before changing constraints**, with a clear migration error explaining that rollback is blocked by legitimate zero-quantity P1-007 data;
- never delete, rewrite, or silently inflate business quantities to make DOWN succeed;
- if data is compatible, DOWN restores the original `> 0` constraints;
- no unrelated schema changes.

### Required decisive proof

Use the **actual migration implementation / MikroORM migrator path**, not copied SQL pretending to be the migration.

Prove both paths:

A. **Compatible-data reversible path**
1. migration UP executes;
2. inspect actual constraints -> `>= 0`;
3. ensure there are no incompatible zero rows in the isolated test fixture/state used for this rollback check;
4. execute actual migration DOWN;
5. inspect constraints -> original `> 0`;
6. execute actual migration UP again;
7. inspect constraints -> `>= 0`.

B. **Incompatible-data fail-safe path**
1. under UP state, create a deterministic P1-007 cancelled zero-quantity fixture (prefer the real R46 path);
2. attempt actual migration DOWN;
3. DOWN fails with the explicit compatibility/precondition error **before constraint mutation**;
4. fresh DB proves the zero business rows are still present and unchanged;
5. constraints remain `>= 0`;
6. clean up only the deterministic test fixture using normal test teardown/fixture cleanup, never broad production-style DELETE statements.

Do not accept a test that merely invokes hand-written ALTER TABLE statements matching the migration text.

---

## 2. Run and persist every required targeted regression gate

### Current defect

The current evidence contains:

- P1-007 PostgreSQL suite;
- P1-007 Mercato Playwright;
- P1-007 Scanner Playwright;
- full `wms_outbound`;
- one Inbound ATP unit/focused test.

It does **not** persist the explicitly required cross-ticket/shared targeted gates from the fifth override.

### Required regression execution

Discover the exact current existing test files/commands from the repos and current testing contract; do not invent names or pass labels.

Run and record exact commands + exact pass counts for:

- P1-007 focused real PostgreSQL suite after migration fix;
- P1-007 Mercato Supervisor Playwright, all 6 outcomes;
- P1-007 Scanner replacement continuation Playwright;
- **P1-004** Allocation / hard-reservation focused regression;
- **P1-005** PickTask creation/assignment/single-active focused regression;
- **P1-006** real RF picking + SAME-key/retry-idempotency focused regressions, including Scanner where applicable;
- **P1-001** CustomerOrder aggregation/policy focused regression;
- **P1-003** planning / requiredQty focused regression;
- **P1-008** TU regression only if this sixth override touches TU code/schema (otherwise explicitly state not touched and no rerun required by this override);
- targeted accepted **Inbound** compatibility for the shared areas affected by P1-007/P1-004 reservation logic: Inventory + location/warehouse + shared TU/entity compatibility where existing suites cover them;
- full `wms_outbound` regression as an additional umbrella gate.

If a requested historical ticket has no single ticket-named test file, select the existing focused suite(s) that actually cover its accepted invariant and state that mapping explicitly in evidence.

Do not treat the umbrella `wms_outbound` run as a substitute for the targeted gates.

### Durable evidence update

Update `05_EVIDENCE/P1-007_EVIDENCE.md` only after the final Mercato commit(s) and test runs.

It must contain:

- exact final 40-char Mercato HEAD from pushed `outbound/p1-007`;
- exact unchanged/current Scanner 40-char HEAD (unless a Scanner change was genuinely necessary);
- exact evidence base/lineage;
- actual migration filename and exact real migration UP/DOWN/re-UP proof;
- incompatible-data DOWN fail-safe proof with no data loss;
- P1-007 20+ focused test result after the final migration change;
- six Mercato Playwright outcomes;
- Scanner replacement Playwright;
- strict concurrency/rollback proof;
- every targeted regression command + pass count listed above;
- any genuine remaining gap, if one exists.

Evidence label may be `PLAYWRIGHT VERIFIED` only. Never self-declare `HUMAN VERIFIED` or `FINAL PASS`.

---

## Hard boundary / STOP

Do not start P1-009, P1-010, P4, packing/repack/QC, Shipment, Carrier, labels, ERP/manifest, or unrelated cleanup.

Do not change accepted P1-006 behavior/history.

Do not change P1-007 product behavior except the minimum migration safety change required above.

Push the minimal Mercato migration/test change + corrected durable evidence, then **STOP**.