# WMS Outbound — Prompt Skill Routing

**Status:** durable supervisor/executor prompt-generation steering  
**Effective:** 2026-09-06  
**Owner-authorized:** yes

## Purpose

Prevent executor-specific prompt drift and reactive micro-ticketing. Antigravity, local Codex, Codex Cloud, Claude and other authorized executors must receive the same business truth, full-item ownership contract and evidence boundary.

## Mandatory supervisor bootstrap before writing any executor ticket/guide

Before preparing a new item guide, corrective guide or materially changing an executor instruction, the supervisor must first refresh current authority:

1. current `Devaxonic-WMS/AGENTS.md`, `.ai/STATE.md`, current `.ai/HANDOVER_OUTBOUND_CURRENT_*.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, `.ai/PLAN.md`;
2. current `WMS_Outbound/AGENTS.md`, `STATE.md`, current `08_HANDOVER/HANDOVER_CURRENT_*.md`, `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`;
3. current `wms-outbound` routing/context skill;
4. Architect/Canon for the exact item: source/translation, state model, requirements, acceptance scenarios and Task Catalog slice;
5. current accepted implementation/evidence for dependencies;
6. `architecture-context` when shared/Inbound compatibility is materially relevant;
7. `scanner-context` when Scanner behavior is materially relevant.

No new/fresh chat may write an executor prompt from old chat memory alone.

## Mandatory prompt-generation skills

For every delegated implementation/test/remediation handoff:

1. read/apply the current installed **`fetch_me_prompt`** skill;
2. read/apply current **`operational-mode`** for long-running executor ownership/execution behavior;
3. for DB/runtime/concurrency/UI acceptance use the current real-evidence contract referenced by `fetch_me_prompt` plus canonical `.ai/TESTING.md`;
4. then write/update one bounded **full-item** Git guide under `WMS_Outbound/06_AGENT_GUIDES/`.

If skill wording conflicts with current Owner-authorized WMS Git steering, current WMS Git steering wins. The skill is a prompt-construction discipline, not business authority.

## Executor-neutral output contract

The generated guide must be executable by Antigravity, local Codex, Codex Cloud or Claude without changing business scope or evidence rules.

Normal unit of work is the entire authorized Task Catalog item, not a one-failure micro-ticket.

The guide must tell the executor to:

- own implementation through tests/build/runtime/UI/evidence/push;
- solve ordinary fixture, auth, TLS, test-data, selector, service-startup, terminal-lifecycle and tooling issues autonomously;
- treat implementation mistakes, compile/type errors, failing assertions and regressions caused by the executor's own in-scope changes as **executor-owned self-repair work, not blockers while a normal in-scope correction exists**;
- on such a failure, continue the loop `diagnose -> patch -> rerun the smallest relevant check -> continue the item` without returning control to the Owner/supervisor;
- preserve already-green proof unless later product changes invalidate it;
- return only `COMPLETE` or a true escalation blocker;
- apply two-strikes only to the **same material unresolved technical path** after two genuinely different substantive attempts;
- never self-declare `FINAL PASS`, `Owner Accepted` or `Human Verified`.

A `BLOCKER` response is invalid when it only reports an ordinary implementation/test regression and does not identify either:

1. two genuinely different substantive failed attempts on the same material technical path, with the path still unresolved and no normal in-scope move remaining; or
2. a genuine Owner-controlled boundary such as scope expansion, destructive action, Demo/Prod access, environment/venue/executor switch or a missing product decision.

Do not generate `STOP after first failure` or artificial rerun-count limits for ordinary fixture/tooling/implementation failures.

## Codex long-horizon execution mode

For a full Task Catalog item executed in Codex, use Codex **`/goal` long-running mode** rather than a normal one-turn prompt whenever the installed Codex surface supports it.

Rationale: `/goal` is the Codex mechanism intended for a durable objective with a verifiable stopping condition across long-running work. A normal Codex turn must not be treated as the default execution container for an item expected to include implementation, repeated self-repair, PostgreSQL proof, regressions, rendered UI acceptance and evidence.

The supervisor must construct one goal for the **whole item**, with:

- one objective: complete the exact authorized Git guide;
- one stopping condition: all implementation/tests/build/runtime/UI/evidence are complete and pushed;
- the exact guide path as the authority to execute;
- explicit instruction to continue through checkpoints and ordinary self-repair without Owner round-trips;
- terminal output only after the goal's verifiable completion condition or a true two-strikes / Owner-controlled blocker.

Do not split a healthy item into repeated normal Codex turns merely because one turn ends. Do not replace `/goal` with repeated `continue`, `resume`, status prompts or micro-tickets when long-horizon goal mode is available.

If `/goal` is not present in the current Codex slash-command list, the Owner may enable the Codex goals feature using the current Codex-supported goals setting/command, then launch the same full-item goal. This is executor capability setup, not a change to WMS business scope.

Antigravity and other executors continue to use their own long-running execution mechanisms; `/goal` is Codex-specific execution transport only and does not change business truth, guide content, evidence requirements or acceptance authority.

## Owner-facing prompt

After the detailed Git guide exists, owner-facing executor text remains microscopic.

For **Codex full-item execution**, prefer a single long-horizon goal:

```text
/goal Sync WMS_Outbound/main and execute ONLY `06_AGENT_GUIDES/<GUIDE>.md`. Complete the entire authorized item end-to-end, including implementation, self-repair, required real tests, regressions, build/runtime, rendered UI acceptance, evidence and pushes. Do not stop for progress/status/incomplete reports or ordinary in-scope failures. Stop only when the guide's COMPLETE condition is fully satisfied or a true two-strikes / Owner-controlled blocker exists.
```

For executors without Codex `/goal`, the normal microscopic handoff remains:

```text
Sync WMS_Outbound/main and execute ONLY:
`06_AGENT_GUIDES/<GUIDE>.md`

Finish the entire item. Ordinary in-scope implementation/test regressions are not blockers: fix and continue. Return only COMPLETE or a true two-strikes / Owner-controlled blocker.
```

Owner-facing handoff contains prompt content only. Do not combine it with shell/launcher/VPN/session-start commands unless Owner explicitly requests them. Pre-item reset is a separate operation and must not be conflated with executor launch.

Do not paste the detailed ticket into owner chat. Do not append menus/explanations unless requested.

## Fresh supervisor chat rule

A fresh ChatGPT supervisor session must first load current Drive handover/memory and then refresh Git authority and the contexts above. Its first executor prompt for the next item must be created only after this bootstrap and `fetch_me_prompt`/`operational-mode` routing.

Git truth overrides stale Drive/chat history.