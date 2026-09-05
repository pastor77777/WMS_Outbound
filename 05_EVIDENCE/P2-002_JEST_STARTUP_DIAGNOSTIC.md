# P2-002 Jest startup diagnostic

**Frozen refs:** Mercato `24d102a41dcfff669cf6b839b369fbaa3620f87d`; accepted P2-001 `8a264fff5c2ca665294d1e02df90c6f37554fe7f`; Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`.

## Database provenance

The approved environment resolved to PostgreSQL host
`aws-1-eu-central-1.pooler.supabase.com:5432`; the username/project component
contains `yzonugcenguvmojwiihb`. No local host, local socket, credential, or
full URL is recorded here.

## Startup probes (VPS tmux)

All commands used `apps/mercato/jest.config.cjs`.

| Probe | Result |
| --- | --- |
| `jest --version` | `30.4.1`, exit `0` |
| `jest --showConfig` | exit `0`; transformer is `scripts/jest-mikroorm-transformer.cjs` |
| `jest --listTests …p2-001-crossdock-eligibility-postgres.integration.test.ts` | returned the exact test path, exit `0` |

The accepted P2-001 and P2-002 `apps/mercato/jest.config.cjs` contents have
the same SHA-256 (`29aabc5d…c6de84`). The relevant compare contains only
`apps/mercato/src/modules/wms_outbound/data/entities.ts`; no Jest config or
setup file changed.

## Exact-test tracer isolation

Bounded command:

```bash
timeout 25s strace -ff -tt -o /tmp/p2-002-jest-trace \
  ./node_modules/.bin/jest --config apps/mercato/jest.config.cjs --runInBand \
  apps/mercato/src/modules/wms_outbound/services/__tests__/p2-001-crossdock-eligibility-postgres.integration.test.ts
```

It exited `1` after reaching the test's `beforeAll` and PostgreSQL driver.
The tracer session intentionally did not inject `/etc/mercato-localhost.env`;
therefore its `ECONNREFUSED 127.0.0.1:5432` result is invalid as PostgreSQL
evidence, but proves Jest transform/module loading is complete.

One subsequent bounded exact-test probe injected the approved Testing
environment. It produced no stdout/stderr and reached timeout exit `124` after
25 seconds. No source or configuration was changed.

## Diagnosis

**Jest/config/transform/module-load startup is not the blocker.** The original
test reaches `beforeAll` and the PostgreSQL driver. The remaining failure is
**UNRESOLVED after startup**: the exact canonical-Testing probe timed out
without output. This record does not claim a regression pass and is not
canonical P2-002 acceptance evidence.
