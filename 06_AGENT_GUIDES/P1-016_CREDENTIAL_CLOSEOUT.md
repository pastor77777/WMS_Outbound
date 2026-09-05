# P1-016 — credential hygiene final closeout

**Purpose:** remove the single acceptance blocker found during supervisor verification of the pushed P1-016 closeout, then re-bind final evidence to the resulting exact Mercato SHA.

This is a narrow P1-016 closeout only. Do not change business behavior, steering, Scanner, Demo/Prod, or future-task scope.

## Verified current pushed state

Current pushed Mercato P1-016 head:

`dd5ff1493740ffc99e11ce40e0b5ffc6b646f574`

Current WMS P1-016 evidence commit:

`50c5664fda12caf5c2f7bcdb0e9e86c3495a01c2`

Frozen Scanner:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Current evidence records:

- P1-016 PostgreSQL: 25/25;
- mandatory regressions: 9 suites, 187/187;
- dedicated P1-016 Playwright A–F: 6/6;
- canonical Testing runtime on :3009.

## Exact blocker

The dedicated P1-016 Playwright spec currently contains a literal fallback password value in source:

`apps/mercato/src/modules/wms_outbound/__integration__/P1-016-final-settlement-ui.spec.ts`

This violates the canonical Testing rule in `Devaxonic-WMS/.ai/TESTING.md`: designated Testing credentials may be used by the approved harness, but credential values must not be committed into source/evidence.

Do not print or copy the credential value into evidence or final response.

## Required correction

1. Modify only the minimum test/harness code needed to remove every literal password/secret fallback from the P1-016 Playwright spec.
2. Resolve the supervisor password only through the existing approved Testing environment/harness variables already available on the VPS.
3. Fail closed with a clear non-secret error if no approved credential variable is available.
4. Do not rotate credentials, create a new secret source, change authentication behavior, or modify product business logic.
5. Inspect the final P1-016 diff for any other literal secret/token/password values introduced by P1-016 and remove them if present. Do not expose values while inspecting/reporting.

The resulting commit becomes the new final Mercato P1-016 SHA.

## Final-SHA evidence rebinding

Because the final Mercato SHA changes, rerun final evidence on the resulting exact SHA:

- P1-016 PostgreSQL: expected 25/25 or greater;
- mandatory regressions: expected 9 suites, 187/187 or greater if unchanged suites grow;
- build and serve canonical Testing Mercato from the exact new SHA through `mercato-localhost.service`;
- prove localhost:3009/login HTTP 200 and intended public Testing route HTTP 200;
- run the dedicated P1-016 Playwright A–F spec fresh with one worker and zero retries;
- required result: 6/6 passed, 0 unexpected, 0 flaky (or runner-equivalent clean result).

Do not reuse the prior dd5ff1... final-SHA run as the final acceptance run after changing the spec.

## Evidence update

Update only:

`05_EVIDENCE/P1-016_EVIDENCE.md`

Replace final SHA/provenance and literal final run outputs so evidence truthfully points to the new final Mercato SHA. Preserve the already-proven Race A/Race B/rollback evidence if the underlying product/concurrency code is unchanged, but ensure the final evidence clearly distinguishes preserved decisive DB proof from the newly rerun final-SHA suites.

Evidence must contain no credentials/secrets and must not claim FINAL PASS / Supervisor acceptance / Owner Acceptance.

## Push and STOP

Only after the credential blocker is removed and all final-SHA evidence above passes:

1. push the new final Mercato P1-016 commit to `outbound/p1-016`;
2. push the updated `05_EVIDENCE/P1-016_EVIDENCE.md` to WMS_Outbound;
3. report only:
   - new final Mercato SHA,
   - WMS evidence SHA,
   - P1-016 PostgreSQL count,
   - mandatory regression count,
   - dedicated P1-016 Playwright count,
   - frozen Scanner SHA;
4. STOP.

## Hard exclusions

- no P2/P3/P4 implementation;
- no Return Receipt;
- no external carrier API;
- no Scanner changes;
- no Demo/Prod writes;
- no STATE/handover/canon/catalog/traceability/AGENTS/.ai steering changes;
- no credential rotation or new secret-management architecture.
