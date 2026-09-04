# P1-011 — Codex VPS Evidence Closeout Override

**Execution status:** owner-authorized narrow override after STRIKE 2 / environment-discovery stop  
**Catalog item:** `P1-011` — item **14/37**  
**Executor:** continue on the **same current VPS** in Codex. Session continuity is optional; durable Git + canonical VPS operations are authoritative.

The product remediation is already pushed. This shot is **EVIDENCE-ONLY**. Do not change product source, tests, migrations, configuration, Scanner, task catalog, STATE, handover, or `.ai` acceptance checkpoints.

## 0. Fixed refs and write boundary

Supervisor-observed starting refs:

- accepted P1-010 Mercato base: `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643`;
- frozen final P1-011 product head: `c76871b038012f8f9558da0b3d25d02e6c8fe3f1` on `Devaxonic-mercato/outbound/p1-011`;
- frozen Scanner head: `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- WMS `main` before this guide: `216b79b179e6e59ee6424557b4a9b830f32c4dab`;
- accepted campaign state remains **13/37 FINAL PASS**.

Allowed durable write:

- `WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md` only.

Temporary `/tmp` logs are allowed and should be used for literal capture, but do not commit them.

If `Devaxonic-mercato/outbound/p1-011` is not exactly `c76871b038012f8f9558da0b3d25d02e6c8fe3f1`, STOP and report the observed ref. Do not repair or advance product code in this shot.

## 1. Canonical VPS environment — do not rediscover from scratch

This work runs on the approved WMS engineering VPS.

Canonical paths:

- `/home/ubuntu/git/WMS_Outbound`
- `/home/ubuntu/git/Devaxonic-WMS`
- `/home/ubuntu/git/Devaxonic-mercato`
- `/home/ubuntu/git/Devaxonic-scanner`

Do not use stale `/home/ubuntu/devaxonic-wms` and do not use Demo paths.

Read first:

- `/home/ubuntu/git/Devaxonic-WMS/.ai/TESTING.md`
- `/home/ubuntu/git/Devaxonic-WMS/.ai/OPERATIONS.md`
- current real-evidence contract
- `/home/ubuntu/git/WMS_Outbound/06_AGENT_GUIDES/P1-011_EXECUTION.md`
- `/home/ubuntu/git/WMS_Outbound/06_AGENT_GUIDES/P1-011_REMEDIATION.md`
- `/home/ubuntu/git/WMS_Outbound/06_AGENT_GUIDES/P1-011_CODEX_FRESH_SESSION_REMEDIATION.md`
- current `P1-011_EVIDENCE.md`

The approved Testing DB environment source on this VPS is:

`/etc/mercato-localhost.env`

Do **not** ask the owner for `DATABASE_URL`. Do **not** print it.

For every Jest / migration / DB-backed command that needs Testing DB variables, source the environment in the **same shell invocation** as the command, e.g. the pattern:

```bash
cd /home/ubuntu/git/Devaxonic-mercato && \
  set -a && source /etc/mercato-localhost.env && set +a && \
  <command>
```

Do not source it in one shell and run tests in another.

Before DB tests, perform a safe preflight without exposing credentials:

1. `test -r /etc/mercato-localhost.env`;
2. source it in the same shell;
3. verify `DATABASE_URL` is non-empty;
4. print only safe derived identity such as DB hostname / `current_database()` / PostgreSQL version, never user/password/token/query secrets;
5. confirm it is the approved remote Testing PostgreSQL, not a local database.

Historical accepted VPS operation uses the remote Supabase Testing DB and the transaction pooler where applicable. Follow `.ai/TESTING.md` exactly.

## 2. Canonical Testing Mercato runtime

Testing Mercato is the real host service:

- URL: `http://localhost:3009`
- hosted route: `https://devaxonic-test.info-start.com.pl`
- systemd unit: `mercato-localhost.service`

Preflight:

```bash
systemctl is-active mercato-localhost.service
curl -fsS -o /dev/null -w '%{http_code}\n' http://localhost:3009/login
```

The owner explicitly authorizes bringing the **Testing** Mercato runtime up for this P1-011 evidence closeout if it is inactive or stale. This does not authorize Demo or Production changes.

The browser evidence must exercise the build corresponding to frozen Mercato head:

`c76871b038012f8f9558da0b3d25d02e6c8fe3f1`

If the service is not running a build from that frozen head, use the existing approved Testing build/restart procedure from current `.ai/OPERATIONS.md` / repository scripts, build the frozen head, and restart **only `mercato-localhost.service`**. Record safe runtime provenance (Git head, build result/identifier if available, service active timestamp, `/login` HTTP result) in evidence.

Do not modify product source or runtime configuration to make tests pass.

## 3. UI authentication — use canonical local sources, never chat

Do not ask the owner to paste UI credentials.

Before Playwright authentication, inspect the canonical local operational credential instructions on the VPS, including when present:

- `/home/ubuntu/git/Devaxonic-WMS/.ai/CODEX-TEST-CREDENTIALS.md`
- `/home/ubuntu/git/Devaxonic-WMS/.ai/OPERATIONS-CREDENTIAL-OVERRIDE.md`

Use the current approved Testing account/environment mechanism described there and in the existing integration auth helpers. Required env overrides such as `OM_INIT_ADMIN_EMAIL`, `OM_INIT_ADMIN_PASSWORD`, `OM_INIT_SUPERADMIN_EMAIL`, `OM_INIT_SUPERADMIN_PASSWORD`, or task-specific Playwright variables must be injected from approved local sources into the **same shell invocation** as the test.

Never print, echo, commit, summarize, or copy passwords/tokens/hashes into WMS evidence.

If a canonical credential file documents where the value is retrieved rather than containing the value itself, follow that local source. Do not substitute generic seed credentials.

## 4. Product is frozen — verify blockers from source, then execute evidence

Do not edit the Mercato branch.

Verify from frozen source that remediation contains:

- PostgreSQL transaction-scoped serialization for the exact R26 grouping tuple, protecting the empty-set find-or-create race for **different compatible TUs**;
- the tuple includes organization, tenant, warehouse, customer, delivery address, priority and exact nullable `slaDeadline`;
- no-partial CustomerOrder readiness prevents SLA from splitting a complete `allowPartialShipment=false` CustomerOrder across multiple Shipments;
- the accepted historical P1-007 migration matches the accepted P1-010 base byte-for-byte;
- any typing workaround is external to that historical migration;
- P1-011 Playwright source contains no embedded credentials/secrets.

## 5. Fresh final-head DB evidence

Run the focused P1-011 genuine PostgreSQL integration suite from frozen head with `/etc/mercato-localhost.env` sourced in the same shell. Capture stdout/stderr literally with `set -o pipefail` + `tee` to a fresh `/tmp` file.

Evidence must include the actual final test count/titles/timing produced by this run. Do not reconstruct or reuse first-shot output.

The run must decisively prove at least:

- exact R26 grouping including exact SLA and address separation;
- **two different compatible TUs** racing through independent real DB participants converge to one Shipment;
- real PostgreSQL advisory/lock wait provenance tied to that grouping-key operation;
- same-TU replay remains idempotent;
- R57/R58 blocked incomplete CustomerOrder;
- TC-123/124/125/126/127 boundaries;
- `allowPartialShipment=false` + already-expired SLA + multiple ready TUs produces one complete Shipment before readiness;
- partial-allowed SLA closure remains valid;
- R29 late TU creates follow-up Shipment;
- genuine transaction rollback with a fresh independent read.

Record safe DB identity (`current_database()`, version, safe host) and fresh participant PIDs/wait event where produced.

## 6. Fresh final-head regressions

With the same approved env mechanism, rerun and capture literal final-head output for:

- P1-010 PostgreSQL suite — expected accepted source count is **16 tests** if unchanged;
- P1-009 PostgreSQL suite;
- P1-003 relevant planning/grouping regression;
- full `src/modules/wms_outbound` backend umbrella.

Run typecheck/build if required by the current testing/evidence contract. A previous typecheck before runtime evidence may be noted but does not replace required final-head test evidence.

Do not rerun shared/Inbound ceremonially unless current `architecture-context` says this frozen remediation touched a shared primitive requiring targeted protection.

## 7. Fresh Playwright evidence

Run from the frozen product head against the real Testing Mercato runtime on `http://localhost:3009` (or the exact approved Testing route required by current test config), using canonical local auth/env sources.

Capture literal Playwright output with `tee`.

Required P1-011 journeys:

1. eligible Shipment grouping/readiness with visible Shipment identity/state and matching DB membership;
2. `allowPartialShipment=false` blocked visible reason, DB proves no membership while blocked, then normal server reevaluation unblocks without manual DB repair;
3. partial-allowed SLA closure + late TU shown as two distinct Shipments without mutating the closed Shipment.

Also rerun accepted P1-010 Mercato Playwright Packer regression because P1-011 modified the packing handoff UI.

Browser evidence label is **PLAYWRIGHT VERIFIED** only.

If Playwright runs in a container/MCP environment, obey the current container networking rule: verify the browser actually reaches the VPS host Testing service, not its own container localhost.

## 8. Rebuild durable evidence

Replace the historical STRIKE 1 contents of:

`/home/ubuntu/git/WMS_Outbound/05_EVIDENCE/P1-011_EVIDENCE.md`

with final-head evidence only.

It must contain:

- full 40-char accepted P1-010 base;
- full 40-char frozen final P1-011 head `c76871b038012f8f9558da0b3d25d02e6c8fe3f1`;
- frozen Scanner ref;
- exact WMS guide/ref used for this override;
- exact product diff scope and confirmation no product edits occurred in this shot;
- P1-007 historical migration blob equality to accepted base;
- safe runtime + DB provenance;
- exact commands/working directories but with secrets redacted/not rendered;
- literal fresh test/Playwright output from `/tmp` captures;
- distinct-TU concurrency PIDs/wait evidence;
- no-partial expired-SLA one-Shipment proof;
- genuine rollback proof;
- exact regression counts, including P1-010 16/16 if that is the actual final output;
- `PLAYWRIGHT VERIFIED` only;
- explicit P1-012+ out-of-scope statement.

Do not include credential values or secret-bearing env output.

## 9. Push and STOP

Commit/push **only** the rebuilt `05_EVIDENCE/P1-011_EVIDENCE.md` to `WMS_Outbound/main`.

Do not update `STATE.md`, handover, task catalog, implementation plan, `.ai` acceptance mirrors, Mercato source, or Scanner.

Then STOP. Do not ask the owner for logs, SHAs, passwords, screenshots, test output or manual Git commands. The supervisor will independently verify remote Git state.