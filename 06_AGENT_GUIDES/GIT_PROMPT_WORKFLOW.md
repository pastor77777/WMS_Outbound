# WMS Outbound — Git Prompt / Executor Supervisor Workflow

**Status:** current operating protocol  
**Effective:** 2026-09-05  
**Applies to:** WMS Outbound delegated implementation, remediation, evidence and acceptance work

## Purpose

Detailed executor instructions live in Git. The owner should only need to send a microscopic launch message and later return `done` or the executor's short final report.

Do not burden the owner with large prompts, terminal logs, test output, screenshots, SHAs or secrets that the supervisor can verify independently.

## Authority before execution

For every shot:

1. read current `WMS_Outbound/STATE.md` and current handover;
2. ground the exact item in `TASK_CATALOG.md`, Architect/canon, requirements and acceptance scenarios;
3. follow the canonical Devaxonic-WMS execution contract in `Devaxonic-WMS/AGENTS.md`, `.ai/TESTING.md` and `.ai/OPERATIONS.md`;
4. independently verify current implementation/evidence refs before writing the guide.

Current source wins over an older guide or old chat history.

## Canonical rhythm

**SHOT -> RECEIVE -> VERIFY EVIDENCE -> NEXT SHOT**

### 1. Put the detailed shot in Git

Before launch, create or update a bounded task guide under:

`WMS_Outbound/06_AGENT_GUIDES/`

The guide carries all material detail the executor needs locally:

- task/source/acceptance scope;
- known current bases/heads where relevant;
- exact allowed files/behavior and hard exclusions;
- decisive real evidence requirements;
- whether product/test/evidence changes are allowed;
- STOP condition;
- two-strikes/override status where applicable.

Do not rely on chat-session memory for correctness.

Preferred naming remains task-specific, e.g. `<TASK>_EXECUTION.md`, `<TASK>_REMEDIATION.md`, `<TASK>_CLOSEOUT.md`.

### 2. Owner-facing launch prompt is microscopic

If the guide exists in Git, **do not paste the ticket into chat again**.

Normal launch prompt should be 2–4 lines, for example:

```text
Sync WMS_Outbound/main and execute ONLY:
`06_AGENT_GUIDES/<GUIDE>.md`

Push required implementation/evidence, then STOP.
```

Add `continue in the same session` only when session continuity is intentionally useful and the owner asked for that wording. Do not make it a hidden requirement.

**The launch prompt is the final owner-facing content for that shot.** Do not append questions, suggested follow-ups, menus, extra options, automation suggestions, explanations or executor-startup instructions below it unless the owner explicitly asks for them.

### 3. Executor-neutral operation

The owner selects the executor/venue **and controls how executors are started, resumed, organized and switched**.

Codex, Antigravity and Claude may execute the same Git guide when authorized and operating under the canonical Devaxonic-WMS contract.

- Do not tell the owner how to launch, restart, resume, organize or sequence executors unless the owner explicitly asks for those mechanics.
- Do not silently switch executor/model/venue to escape a blocker.
- A fresh executor session is valid when owner-selected; it must bootstrap from Git/state, not old chat memory.
- A same-session continuation is also valid when owner-selected.
- Executor choice must not change business truth, evidence rules or repository paths.
- If the owner says an executor is currently working, do not interfere with that run or redirect it. Supervisor-side verification, steering preparation and handover maintenance may continue independently.

Executor-specific launch mechanics belong in `Devaxonic-WMS/.ai/OPERATIONS.md`, not in owner-facing supervisor messages unless explicitly requested.

### 4. Receive

Normal owner return is simply `done` or the executor's short final report.

Do not ask the owner to paste:

- terminal logs;
- test output;
- branch lists;
- screenshots;
- long evidence documents;
- secrets.

If troubleshooting genuinely requires an owner-side check, give **one precise action/command at a time** and request only the minimal non-secret status needed. Do not flood the owner with alternative command trees.

### 5. Supervisor independently verifies

Executor prose is never acceptance by itself.

Verify as relevant:

- exact branch HEADs / 40-char SHAs;
- lineage from accepted bases;
- compare/diff scope;
- no unrelated changes;
- actual decisive product/test implementation;
- migration path if claimed;
- literal targeted test/evidence outputs in durable evidence;
- real PostgreSQL/concurrency/rollback provenance;
- Playwright runtime/revision and truthful evidence label;
- unaffected frozen repo heads;
- WMS evidence/state/handover commit scope.

### 6. Acceptance

- Automated browser proof is `PLAYWRIGHT VERIFIED`.
- Never relabel automation as `HUMAN VERIFIED`.
- Supervisor verifies evidence first.
- `FINAL PASS / Owner Accepted` occurs only after explicit owner acceptance at the required boundary.
- After owner acceptance, update durable STATE/handover as appropriate.

### 7. Two-strikes / STOP

For the same material path:

- one initial attempt;
- one genuine corrective attempt;
- after materially identical second failure: STOP unless the owner explicitly authorizes a narrow additional shot.

Each explicit override permits only the authorized narrow extra shot.

After STOP, report the exact blocker/current refs. Do not automatically invent another remediation, switch executor or broaden scope.

## Real evidence invariants

The detailed evidence contract lives in Devaxonic-WMS `.ai/TESTING.md` and `WMS_Outbound/05_EVIDENCE/EVIDENCE_STANDARD.md`.

Do not weaken these basics:

- persistence/lock/concurrency/rollback claims use genuine approved Testing PostgreSQL mechanisms;
- concurrency uses independent overlapping operations plus PostgreSQL-side evidence;
- rollback crosses a real write/flush/failure-before-commit and fresh independent read;
- Playwright performs the decisive intended-actor action through the rendered UI;
- fixture APIs/DB may prepare state but cannot substitute the accepted user action.

## Repo roles

- `pastor77777/WMS_Outbound` — Architect/canon/traceability/task/evidence/guide/state/handover.
- `pastor77777/Devaxonic-WMS` — canonical session entrypoint and operational/testing steering.
- `pastor77777/Devaxonic-mercato` — Mercato/backend implementation and tests.
- `pastor77777/Devaxonic-scanner` — Scanner implementation and tests.

## Steering boundary

Durable steering/control changes require explicit owner acceptance before application.

Task guides are execution artifacts under the authorized task scope; changes to this workflow, `AGENTS.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, persistent memory routing or equivalent control files require owner acceptance.

An explicit owner request to persist/update the working rules in steering or memory files counts as acceptance for the specifically requested rule maintenance; do not broaden it into unrelated policy changes.

## Long-chat continuity / memory handoff

When the supervisor chat becomes long enough that continuity risk is material:

1. keep mutable project truth in current Git STATE/handover;
2. persist only durable working rules in canonical steering files, not task-specific chat lore;
3. update the current ChatGPT handover/memory record on Google Drive when owner-authorized;
4. record the exact in-flight task and accepted baseline, but do not mark executor work complete before independent verification;
5. give the owner one short fresh-chat kickoff prompt, with no appended questions or suggestions.

A fresh supervisor must prefer current Git state/handover over stale Drive memory if they conflict.

## Fresh-session rule

A fresh supervisor/executor must read current state/handover and this workflow from Git. Do not hard-code mutable progress, next-task SHAs or checkpoint data into generic startup/operations files.

## Owner-facing communication

Keep it operational:

- verified PASS or exact blocker;
- one microscopic launch prompt when another shot is authorized;
- otherwise STOP.

Do not add post-prompt engagement questions, optional next steps or explanatory tails when the owner asked only for the prompt.