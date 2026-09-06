# P3-003 — Rendered acceptance fixture correction

**Status:** same-item supervisor correction after independent review  
**Item:** P3-003 — Cancellation race: physical movement before formal confirmation  
**Do not reset Testing. Do not restart P3-003.**

## Current durable checkpoint

- Mercato `outbound/p3-003`: `f600782496e865603200d46d7e6041a54f90b9a4`
- Scanner `outbound/p3-003`: `e39352100d33d1db52b2076d25cb7a7310fc5b41`
- WMS evidence commit: `098db61eda991db2172248ae4fb2048b4d1f2678`
- Existing dedicated real PostgreSQL P3-003 suite: 14/14 PASS in evidence.
- Existing rendered TC-042/TC-043 suite: 2/2 PASS in evidence, but fixture state needs correction below.

## Exact review finding

`Devaxonic-scanner/e2e/p3-003-rendered-acceptance.spec.ts` currently seeds both TC-042 and TC-043 allocations with:

`wms_outbound_allocations.status = 'CONFIRMED'`

before any formal pick confirmation.

That does not reproduce the required P3 R7 pre-confirm boundary. Before formal confirmation, authoritative Allocation must still be `RESERVED`; formal full-pick confirmation owns the accepted `RESERVED -> CONFIRMED` transition.

The dedicated PostgreSQL suite already proves the correct RESERVED pre-confirm behavior, but the decisive rendered human journey must start from the same architecturally valid state.

## Required correction

1. In the rendered TC-042 and TC-043 fixtures, seed the relevant Allocation rows as `RESERVED`, not `CONFIRMED`.
2. Preserve `pickedQty = 0`, assigned/active PickTask flow and all existing P3-003 business behavior before formal confirmation.
3. Rerun the real zero-route-mock rendered TC-042 journey through normal Scanner operator UI and Mercato Supervisor UI.
4. Rerun the real zero-route-mock rendered TC-043 journey through normal Scanner operator UI and Mercato Supervisor UI.
5. Verify TC-042 still proves:
   - pre-confirm observation active before cancellation;
   - Allocation remains RESERVED before cancellation;
   - cancellation selects P3;
   - exact-source return instruction renders;
   - pickedQty remains 0;
   - zero PutBackTask;
   - zero P4 physical-return handoff.
6. Verify TC-043 still proves:
   - Allocation is RESERVED before formal confirmation;
   - formal positive pick confirmation performs the accepted Allocation lifecycle effect;
   - cancellation then selects P4-001;
   - no P3 exact-source return instruction is produced;
   - P4 handoff exists as accepted.
7. Replace/update the two screenshots if the rerun changes them.
8. Update `05_EVIDENCE/P3-003_EVIDENCE.md` so its fixture description and assertions truthfully state the RESERVED pre-confirm allocation state and record the corrected rendered results.
9. Push Scanner and WMS evidence changes. Mercato must remain at the current head unless the corrected real UI proof exposes an actual product defect requiring an in-scope repair.
10. If product code changes are required, rerun the dedicated P3-003 PostgreSQL suite and any directly affected mandatory regressions before completion.

## Failure / completion boundary

This is ordinary same-item correction work. A failing rendered run, fixture mismatch, selector issue, build/runtime issue or in-scope regression is self-repair work, not a terminal blocker while a normal correction exists.

Return only:

`COMPLETE` + final remote Mercato SHA + Scanner SHA + WMS evidence SHA + corrected rendered counts

or a genuine two-strikes / Owner-controlled blocker.

Do not start another Task Catalog item.