# P2-002 — MikroORM.init isolation via native Jest transform

## Purpose
Isolate the remaining P2-002 regression blocker at the exact `MikroORM.init(...)` used by accepted P2-001, while preserving canonical Supabase Testing provenance and avoiding the invalid `ts-node/register` ESM/CommonJS path.

## Frozen refs
- Mercato implementation: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Accepted P2-001 base: `8a264fff5c2ca665294d1e02df90c6f37554fe7f`
- Scanner: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`
- Database: canonical Supabase Testing only, project ref `yzonugcenguvmojwiihb`

## Hard prohibitions
- NO local PostgreSQL, localhost DB, local socket DB, SQLite, mock DB, or replacement database.
- NO product/business-code changes.
- NO committed Mercato/Scanner test, config, harness, dependency, schema, migration, state, handover, or evidence-acceptance changes.
- Do NOT run the full P2-001 suite or any P2-002 suite in this shot.
- Do NOT change `DATABASE_URL` or print credentials/full URL.

## Execution venue
Owner-controlled VPS shell/tmux. Work in `/home/ubuntu/git/Devaxonic-mercato` at exact frozen Mercato SHA.

Load the same approved canonical Testing environment used by Mercato. Before any DB-bearing probe, print only sanitized provenance proving host `aws-1-eu-central-1.pooler.supabase.com`, port `5432`, PostgreSQL scheme, and project ref presence `yzonugcenguvmojwiihb`; never print password/full URL.

## Probe
Create ONE temporary, untracked diagnostic Jest test under the existing Mercato Jest `rootDir`, then delete it before finishing. The temporary diagnostic must use the repository's normal `apps/mercato/jest.config.cjs` and therefore the existing Jest MikroORM transformer. Do not use `ts-node/register`.

The temporary diagnostic must import exactly:
- `MikroORM` from `@mikro-orm/core`
- `PostgreSqlDriver` from `@mikro-orm/postgresql`
- `WmsTransportUnit`, `WmsTuExpectedContent`
- `WmsOutboundCrossDockBinding`, `WmsOutboundCustomerOrder`, `WmsOutboundCustomerOrderLine`, `WmsOutboundOrder`, `WmsOutboundOrderLine`, `WmsOutboundAtpReservation`, `WmsOutboundWarehouseQueueConfig`

Use the exact accepted P2-001 init options:
- `driver: PostgreSqlDriver`
- unchanged canonical `clientUrl: process.env.DATABASE_URL`
- exact entity list above
- `discovery: { warnWhenNoEntities: false }`
- `driverOptions: { ssl: { rejectUnauthorized: false } }`

The probe must emit flushed timestamp markers to stderr/stdout for exactly these boundaries:
1. `IMPORTS_COMPLETE`
2. `BEFORE_MIKROORM_INIT`
3. `AFTER_MIKROORM_INIT`
4. `BEFORE_ORM_SELECT1`
5. `AFTER_ORM_SELECT1`
6. `AFTER_ORM_CLOSE`

After successful init, execute `select 1 as ok, current_database() as db` through the MikroORM connection, verify `ok=1`, then close ORM cleanly.

Run ONLY this one temporary Jest test, `--runInBand`, with one outer timeout of 45 seconds. Capture literal stdout/stderr and numeric exit status durably.

## Classification / stop rules
- If `AFTER_MIKROORM_INIT` appears and ORM `SELECT 1` succeeds: classify `MIKROORM_INIT_PASS`, delete temp test, record result, STOP. Do NOT run P2-001 yet.
- If `BEFORE_MIKROORM_INIT` appears but `AFTER_MIKROORM_INIT` does not before timeout: classify `MIKROORM_INIT_HANG`, delete temp test, record last marker + timeout status, STOP.
- If init returns an actual error: classify `MIKROORM_INIT_ERROR`, capture exact sanitized error/stack, delete temp test, STOP.
- If imports fail under Jest: classify `JEST_IMPORT_ERROR`, capture exact error, delete temp test, STOP.
- If init succeeds but ORM `SELECT 1` hangs/fails: classify `ORM_QUERY_PATH`, capture exact last marker/error, delete temp test, STOP.

No retries in this shot.

## Record
Create only a diagnostic record at:
`05_EVIDENCE/P2-002_MIKROORM_INIT_JEST_PROBE.md`

Include frozen refs, sanitized canonical DB provenance, exact temporary probe intent, literal marker sequence, numeric status, classification, and confirmation that the temporary diagnostic file was deleted and `git status` shows no Mercato/Scanner tracked changes from this shot.

Push only the WMS_Outbound diagnostic record, then STOP.
