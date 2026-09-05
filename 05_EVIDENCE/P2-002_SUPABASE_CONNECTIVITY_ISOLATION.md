# P2-002 — Supabase connectivity isolation

Date: 2026-09-05

## Frozen references

- Mercato: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Accepted P2-001: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Scanner: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

## Approved Testing provenance

The canonical Mercato environment was loaded from `/etc/mercato-localhost.env`.
The resulting `DATABASE_URL` was not printed or altered. Sanitized provenance:

```json
{"host":"aws-1-eu-central-1.pooler.supabase.com","port":"5432","scheme":"postgresql","usernameHasProjectRef":true}
```

The username contains the approved Supabase Testing project reference
`yzonugcenguvmojwiihb`.

## Raw `pg` probe

The existing installed `pg` package used the unchanged `DATABASE_URL`, a 10 second
connection timeout, and TLS with certificate verification disabled (matching the
P2-001 test driver behavior).

| Event | UTC timestamp |
| --- | --- |
| Before connect | `2026-09-05T09:56:43.071Z` |
| After connect | `2026-09-05T09:56:43.142Z` |
| After `select 1 as ok, current_database() as db` | `2026-09-05T09:56:43.155Z` (`ok=1`, `db=postgres`) |
| After close | `2026-09-05T09:56:43.160Z` |

Exit status: `0`.

## MikroORM probe

One bounded (45 second) stdin-only Node diagnostic was run with the P2-001 entity
set and the test's `PostgreSqlDriver`, `clientUrl`, and TLS options. It exited
before `MikroORM.init` because the required TypeScript entity module is ESM while
the available `ts-node/register` stdin loader attempted a CommonJS `require`:

```text
Error [ERR_REQUIRE_ESM]: Must use import to load ES Module:
apps/mercato/src/modules/wms_tu/data/entities.ts
```

No `before_mikroorm_init` timestamp was emitted, no connection/query was attempted
by this probe, and its exit status was `1`. No diagnostic process remained after
the bounded attempt.

## Single-test result

Not run. The guide requires stopping when the one MikroORM probe does not complete
decisively; no Jest retry or broader P2-001 suite was started.

## Classification

`MIKROORM_INIT` — raw VPS-to-Supabase PostgreSQL connectivity and a trivial query
succeeded. The sole permitted MikroORM isolation attempt was blocked by the
diagnostic loader's ESM/CommonJS incompatibility before ORM initialization, so this
shot does not establish a P2-001 test-body stall.
