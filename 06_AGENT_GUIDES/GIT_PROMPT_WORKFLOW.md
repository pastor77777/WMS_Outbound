# WMS Outbound — Git Prompt / Antigravity Supervisor Workflow

**Status:** current operating protocol  
**Effective:** 2026-09-03  
**Applies to:** WMS Outbound delegated implementation, remediation, evidence and acceptance work

## Purpose

This file records the working method agreed with the owner so a fresh supervisor chat does not need to rediscover it.

The owner must not be asked to copy/paste long Antigravity logs or evidence into chat. Prompt content, acceptance constraints and remediation instructions are persisted in Git first; Antigravity reads them locally; after the owner says `done`, the supervisor independently retrieves and verifies current Git refs/evidence.

## Authority before execution

For every Outbound shot:

1. Load current installed `wms-outbound` skill for Outbound routing/authority.
2. Load current installed `architecture-context` only for accepted Inbound/shared-foundation compatibility context. It must not redefine Outbound business rules.
3. Load current `fetch_me_prompt` + `operational-mode` before delegated execution.
4. For DB/concurrency/integration/UI work, also refresh current `REAL_EVIDENCE_CONTRACT.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`.
5. Ground exact current task in `TASK_CATALOG.md`, Architect Source/faithful translation, requirements and acceptance scenarios.
6. Current source wins if stricter than an older guide or handover.

## Canonical rhythm

**SHOT -> RECEIVE -> VERIFY EVIDENCE -> NEXT SHOT**

### 1. Supervisor writes the shot to Git first

The supervisor creates or updates a bounded file in `WMS_Outbound/06_AGENT_GUIDES/` before asking the owner to launch Antigravity.

Preferred naming:

- initial execution: `<TASK>_EXECUTION.md`;
- bounded remediation: `<TASK>_REMEDIATION.md` or a clearly numbered closeout/override file;
- final evidence-only gate: explicit `FINAL_ACCEPTANCE` / `CLOSEOUT` file.

The guide must contain enough context that Antigravity can execute from the local checkout without receiving a giant chat prompt.

It must specify:

- exact authoritative task/rules/acceptance mapping;
- exact known current bases/heads where relevant;
- exact scope and hard exclusions;
- real evidence requirements;
- whether product changes are allowed or evidence-only;
- STOP condition;
- two-strikes status where applicable.

### 2. Owner receives only a short launch prompt

After the guide is pushed, the owner gets a compact prompt similar to:

```text
Continue <TASK> in the SAME Antigravity session.

FIRST sync the existing local `WMS_Outbound` checkout with remote `main`, preserving unrelated local work.

Then read and execute ONLY:
`06_AGENT_GUIDES/<GUIDE>.md`

Push the required implementation/evidence, then STOP.
```

Do not paste the full ticket into chat if it is already in Git.

### 3. Same Antigravity session only

- Continue the existing valuable interactive Antigravity session.
- Never restart/replace it or silently launch a second session.
- Preferred wrapper is `/home/ubuntu/.local/bin/agy-pl`; never bare `agy`.
- Preserve unrelated local work while synchronizing `WMS_Outbound` main.

### 4. Owner replies only `done`

The normal return message from the owner is simply `done` / `już`.

Do **not** ask the owner to paste:

- Antigravity terminal logs;
- test output;
- commit SHAs;
- evidence documents;
- screenshots;
- branch lists.

### 5. Supervisor independently fetches current state

After `done`, the supervisor must independently inspect connected current sources, normally GitHub and current repo documents.

At minimum verify as relevant:

- exact branch HEADs and 40-char SHAs;
- parent/lineage from accepted bases;
- compare/diff scope;
- no unrelated product changes;
- exact production/service/test implementation for decisive claims;
- evidence file consistency with current refs;
- migration implementation and real migrator path when migration proof is claimed;
- targeted regression commands/counts;
- Playwright provenance and evidence label;
- Scanner/Mercato heads and shared compatibility gates;
- WMS steering/evidence commit.

Executor prose is never acceptance by itself.

If a remote branch/report is unexpectedly missing immediately after Antigravity work, consider IPS/sync delay before concluding absence.

### 6. Acceptance language

- Automated browser proof is `PLAYWRIGHT VERIFIED`, never automatically `HUMAN VERIFIED`.
- Real DB proof is classified according to what genuinely executed.
- A final owner acceptance can close an item based on reviewed automated evidence when the owner explicitly accepts that basis; the underlying evidence label remains truthful (`PLAYWRIGHT VERIFIED`).
- Never self-declare `FINAL PASS` before supervisor verification and required owner acceptance boundary.

### 7. Two-strikes / STOP

For the same material path:

- attempt 1 may receive one corrective retry;
- after a materially identical second failure: STOP;
- do not invent a third workaround, alternate fake proof, executor switch or scope expansion without explicit owner override (`dalej` / equivalent);
- each explicit owner override permits one narrow additional shot only.

When STOP is active, report the exact remaining blocker and current refs. Do not auto-generate the next remediation.

## Real evidence rules that must survive fresh chats

- Real PostgreSQL means real MikroORM/PostgreSQL against the approved Testing DB; no fake/in-memory EntityManager, JavaScript `Map`, simulated SQL/PG errors or fake locks.
- Real concurrency requires independent overlapping DB operations/transactions and decisive PostgreSQL-side evidence tied to actual participants/PIDs.
- Real rollback requires a real write/flush inside a transaction, deterministic failure before commit, and a fresh independent read proving no partial commit.
- A migration lifecycle claim uses the project-approved real migration path/MikroORM `Migrator` when available. Do not replace it with hand-written DDL, shadow tables, `getQueries() -> em.execute()` replay or broad data deletion.
- Playwright must perform the decisive intended-actor action through the real rendered UI; API/DB may establish deterministic preconditions or verify persistence but cannot replace the user action.
- Inbound is CLOSED / REFERENCE. Run only targeted shared compatibility regressions when shared Inventory/TU/warehouse/lock/orchestration primitives are impacted.

## Architecture-context boundary

`wms-outbound` defines where current Outbound business authority is read from. `architecture-context` contributes accepted shared/Inbound architecture compatibility knowledge only.

Examples of what this prevents:

- treating future P4 PutBackTask/RF lifecycle as an intrinsic P1-007 implementation requirement when the P1-007 execution boundary only requires a durable physical-return handoff;
- changing Outbound R-rules to fit old Inbound behavior;
- accepting shared primitive regressions merely because Outbound tests pass.

## Repo roles

- `pastor77777/WMS_Outbound` — authoritative Outbound Architect/canon/traceability/task/evidence/guide/state/handover repository.
- `pastor77777/Devaxonic-mercato` — Mercato/backend implementation and tests.
- `pastor77777/Devaxonic-scanner` — Scanner implementation and Playwright tests.
- `pastor77777/Devaxonic-WMS` — operational steering/testing/operations mirror and fresh-session handover.

## Fresh-chat startup rule

A fresh supervisor should first read:

1. `WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md` (or later current handover);
2. `WMS_Outbound/STATE.md`;
3. this file;
4. `Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-03.md` (or later);
5. `Devaxonic-WMS/.ai/STATE.md`, `.ai/PLAN.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`;
6. exact next task Architect/requirements/acceptance sources.

Then independently verify current Git refs before creating the next guide.

## Owner-facing communication style

Keep owner-facing messages short and operational:

- what was verified;
- PASS / exact blocker;
- one short Antigravity launch prompt if another shot is authorized.

Do not burden the owner with executor logs that the supervisor can fetch independently.