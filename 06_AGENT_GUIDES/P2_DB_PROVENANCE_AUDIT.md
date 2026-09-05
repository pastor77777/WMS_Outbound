# P2 DB PROVENANCE AUDIT — mandatory corrective gate

## Owner directive
Local PostgreSQL has been forbidden from the start. Any PostgreSQL test/evidence executed against a local database is INVALID and must not count toward acceptance.

## Scope
Audit ONLY database-test provenance relevant to accepted P2-001 and active P2-002.
Do not change product code, tests, migrations, state, handover, or accepted Git lineage.
Do not rerun broad suites yet.

## Authoritative database venue
Only the approved Supabase Testing PostgreSQL may count as genuine PostgreSQL evidence.
Localhost/local PostgreSQL/docker/local service databases are forbidden and invalid for evidence.

## Required audit
1. Inspect P2-001 evidence, raw logs/records, commands, environment references and any cited PostgreSQL proof.
2. Inspect P2-002 implementation/evidence/raw-run records and all PostgreSQL test claims so far.
3. For every claimed PostgreSQL run classify provenance as exactly one of:
   - VERIFIED SUPABASE TESTING
   - VERIFIED LOCAL / INVALID
   - PROVENANCE UNPROVEN / INVALID
4. Do not infer Supabase from the words "real PostgreSQL", "Testing PostgreSQL", or from a passing result. Require concrete connection provenance sufficient to prove the approved Supabase Testing target.
5. Any local or unproven run must be explicitly marked INVALID and excluded from pass counts/evidence.
6. Determine whether P2-001 still has sufficient valid Supabase Testing proof for every mandatory PostgreSQL acceptance requirement. If not, state that P2-001 DB evidence must be re-established before P2-002 can be accepted.
7. Determine which P2-002 PostgreSQL suites/results must be rerun on approved Supabase Testing.

## Output
Create and push ONLY:
`05_EVIDENCE/P2_DB_PROVENANCE_AUDIT.md`

The audit must include:
- exact evidence/log/file references inspected,
- exact commands/environment provenance where available,
- classification for each PostgreSQL run,
- explicit list of invalidated runs,
- explicit list of required Supabase Testing reruns,
- impact on P2-001 acceptance evidence and P2-002 current status.

Do NOT create/update canonical `P2-002_EVIDENCE.md`.
Do NOT change STATE or handover.
Do NOT claim PASS or acceptance.

STOP after pushing the audit record.