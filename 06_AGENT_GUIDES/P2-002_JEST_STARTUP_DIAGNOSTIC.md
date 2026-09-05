# P2-002 — Jest startup diagnostic

## Scope

Owner-authorized narrow diagnostic shot for the current P2-002 blocker only.

Frozen implementation under diagnosis:
- Mercato: `24d102a41dcfff669cf6b839b369fbaa3620f87d`
- Scanner: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Do **not** amend/reset/rebase/transplant either ref.
Do **not** change product code, tests, migrations, Jest config, package scripts, runtime config, STATE, handover, or canonical P2-002 evidence.

## Database boundary — mandatory

Local PostgreSQL is forbidden and invalid evidence.

Any PostgreSQL-backed command in this shot may use only the canonical Supabase Testing project:
- Supabase project name: `DevAxonic_Platform`
- project ref: `yzonugcenguvmojwiihb`

Before any test command, inspect the loaded `DATABASE_URL` **without printing credentials/full URL** and record only enough sanitized provenance to prove it is not local and belongs to the canonical Testing project. Accept direct Supabase host/project-ref identity or a Supabase pooler identity whose username/project component contains `yzonugcenguvmojwiihb`.

If the loaded DB target is `localhost`, `127.0.0.1`, `::1`, a local socket, or cannot be tied to `yzonugcenguvmojwiihb`, STOP immediately and report DB provenance failure. Do not run PostgreSQL tests.

## Known symptom

The exact P2-001 regression command on Mercato `24d102...` produced no stdout/stderr for 80+ seconds. Process tree was `bash -> yarn -> jest`. During the hang there was no active Testing PostgreSQL session/lock. The test imports `wms_outbound/data/entities.ts` before `beforeAll`, and P2-002 changed that file.

Therefore this shot diagnoses Jest/config/transform/module-loading startup only. Do not rerun the full regression blindly.

## Diagnostic sequence

Use owner-controlled VPS shell/tmux. Capture stdout, stderr, timestamps, PID/PGID, exit status, and bounded timeout for every probe. Ensure no orphan Jest/Yarn process remains after each probe.

Run the smallest probes in this order and stop as soon as the blocking layer is proven:

1. Jest binary/config startup only:
   - resolve/version
   - `--showConfig`
2. Exact test discovery only:
   - `--listTests` scoped to `p2-001-crossdock-eligibility-postgres.integration.test.ts`
3. Transformer/module-load isolation without changing source:
   - inspect CPU/state/process tree while the exact single test starts
   - if available, use a bounded external tracer (`strace` or equivalent) on the Jest PID/process group to identify the file/module/syscall where progress stops
   - do not install packages or alter repository/runtime config
4. Compare the same startup probes on the accepted P2-001 ref `8a264fff5c2ca665294d1e02df90c6f37554fe7f` versus P2-002 ref `24d102...`, using separate clean worktrees or read-only checkout mechanics that do not mutate the P2-002 branch.

The comparison must answer one concrete question: does the startup hang reproduce only after P2-002, and if so, at which config/transform/module/file boundary does divergence first appear?

Do not execute the full PostgreSQL regression unless the startup layer is proven healthy and the previously hanging exact test now reaches `beforeAll` / opens a canonical Testing DB session. If it does reach the DB naturally, STOP and report that boundary; do not continue the broader regression battery in this diagnostic shot.

## Output

Create one raw diagnostic record only:
`05_EVIDENCE/P2-002_JEST_STARTUP_DIAGNOSTIC.md`

It must contain:
- sanitized DB provenance (no secrets)
- exact frozen refs
- exact probe commands
- literal outputs / exit statuses / timeout markers
- process/tracer observations
- accepted-P2-001 vs P2-002 comparison
- narrow diagnosis, or explicit `UNRESOLVED` if no layer is proven

Push only that WMS diagnostic record if created. No Mercato/Scanner changes. No `P2-002_EVIDENCE.md`.

Then STOP.