# P2-002 — First P2-001 test stage probe

Date: 2026-09-05

## Frozen references

- Mercato: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Accepted P2-001: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Scanner: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

## Canonical Testing provenance

The approved Mercato environment was loaded from `/etc/mercato-localhost.env`.
`DATABASE_URL` was not printed or altered. Sanitized provenance:

```json
{"host":"aws-1-eu-central-1.pooler.supabase.com","port":"5432","scheme":"postgresql","usernameHasProjectRef":true}
```

The username includes the approved Testing project reference `yzonugcenguvmojwiihb`.

## Temporary probe intent

One temporary untracked native-Jest test mirrored only the accepted P2-001 first-test
path: exact ORM initialization and entity set, fresh source-TU/content persistence,
customer-order persistence, customer-order-line persistence, one
`evaluateAndBind`, a `boundQty === '5.000000'` assertion, probe-scope cleanup, and
ORM close. It was run once with `--runInBand` and a 60-second outer timeout.

## Literal markers and result

```text
2026-09-05T10:07:15.096Z IMPORTS_COMPLETE
2026-09-05T10:07:15.098Z BEFORE_MIKROORM_INIT
2026-09-05T10:07:15.196Z AFTER_MIKROORM_INIT
2026-09-05T10:07:15.196Z BEFORE_SOURCE_FLUSH
2026-09-05T10:07:15.393Z AFTER_SOURCE_FLUSH
2026-09-05T10:07:15.393Z BEFORE_ORDER_FLUSH
2026-09-05T10:07:15.452Z AFTER_ORDER_FLUSH
2026-09-05T10:07:15.452Z BEFORE_LINE_FLUSH
2026-09-05T10:07:15.516Z AFTER_LINE_FLUSH
2026-09-05T10:07:15.516Z BEFORE_EVALUATE_AND_BIND
2026-09-05T10:07:15.858Z AFTER_EVALUATE_AND_BIND
2026-09-05T10:07:15.860Z BEFORE_CLEANUP
2026-09-05T10:07:16.073Z AFTER_CLEANUP
2026-09-05T10:07:16.074Z AFTER_ORM_CLOSE
PASS src/modules/wms_outbound/services/__tests__/p2-002-first-test-stage-probe.test.ts
Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Time:        2.697 s
```

Numeric exit status: `0`.

## Classification

`FIRST_TEST_PASS` — each required stage completed against canonical Supabase
Testing, including the first P2-001 test's persistence sequence and
`evaluateAndBind`. No broader P2-001 suite ran.

## Cleanup confirmation

The temporary diagnostic file was deleted. Mercato is clean at
`24d102a41dcfff669cf6b839b369fbaa3620f87d`; Scanner is clean at
`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`. No probe process remained.
