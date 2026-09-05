# P2-002 — Owner-authorized VPS regression venue

**Purpose:** complete only the blocked mandatory regression verification for P2-002 after the command-execution runtime was proven to terminate the whole test process group before Bash/Jest could emit a result.

**Owner authorization:** explicit authorization to change venue for these regressions only.

## Frozen product revisions

Mercato P2-002 final candidate:

`24d102a41dcfff669cf6b839b369fbaa3620f87d`

Accepted P2-001 base:

`8a264fff5c2ca665294d1e02df90c6f37554fe7f`

Scanner remains frozen:

`f4a404600efb1120cb2f1c5b86383ad148cd1e1a`

Do not amend, rebase, squash, transplant or otherwise modify the Mercato candidate while executing this guide.

## Venue

Run the regressions directly in the Owner-controlled VPS shell/tmux environment against the exact Mercato SHA above.

Do not run them through the previously diagnosed command-execution runtime/process wrapper that killed the process group.

Repository path on VPS:

`/home/ubuntu/git/Devaxonic-mercato`

## Hard write boundary

This is a verification-only venue switch.

Do not modify:

- product/business code;
- migrations;
- test files or fixtures;
- Scanner;
- STATE, handovers, Task Catalog, Canon, Architect Source or traceability;
- AGENTS.md, credentials or unrelated `.ai/*` files;
- Prod/Demo.

The only permitted write after successful regression verification is the already-authorized durable evidence file:

`WMS_Outbound/05_EVIDENCE/P2-002_EVIDENCE.md`

If any mandatory regression is red, do not change code or tests under this guide. Record the exact failing command/result in the report and STOP without creating evidence that claims PASS.

## Mandatory preflight

Before tests:

1. verify the Mercato worktree is clean;
2. verify `git rev-parse HEAD` is exactly `24d102a41dcfff669cf6b839b369fbaa3620f87d`;
3. verify branch/ref identity is `outbound/p2-002` or detached at that exact SHA;
4. use the same approved Testing PostgreSQL/database environment used by P2-001/P2-002 verification;
5. do not apply new migrations or mutate product source as part of this guide.

## Regression set

Run the exact mandatory regression suites required by `06_AGENT_GUIDES/P2-002_EXECUTION.md` for the actual final diff:

1. accepted P2-001 PostgreSQL suite;
2. accepted P1-003 grouping/planning suite(s);
3. FND-003/shared compatibility suites required by the touched warehouse/task primitives;
4. P1-005 task-assignment/ordering regressions only if the final P2-002 diff actually reuses/touches the corresponding shared primitive;
5. accepted Inbound cross-dock-result regression only if required by the touched shared TU/Inbound boundary primitive;
6. Scanner regressions only if Scanner changed — Scanner is frozen here, so do not invent Scanner execution work.

Use the exact existing test files/commands from the repository and the P2-002 execution guide. Do not substitute approximate or newly-created tests.

## Result capture

For every invoked suite, capture durably in the shell/session output:

- exact command;
- exact exit status;
- Jest suite/test counts (`passed/failed/skipped` where reported);
- final SHA under test.

A suite is PASS only with an observed normal test completion and exit status `0`.

Do not infer PASS from silence, lack of a remaining process, or prior successful runs on a different SHA.

## Existing P2-002 proof to preserve

The dedicated P2-002 real PostgreSQL suite was already verified as **4/4 passed** after the FK-ordering correction. Do not change product/test code to improve or inflate this count.

Final Mercato candidate remains:

`24d102a41dcfff669cf6b839b369fbaa3620f87d`

## Evidence completion — only if all mandatory regressions PASS

If and only if all mandatory regressions complete normally with exit status `0`, create/update only:

`05_EVIDENCE/P2-002_EVIDENCE.md`

The evidence must truthfully record:

- accepted P2-001 base `8a264fff5c2ca665294d1e02df90c6f37554fe7f`;
- final Mercato `24d102a41dcfff669cf6b839b369fbaa3620f87d`;
- Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a`;
- compare scope/files;
- dedicated P2-002 PostgreSQL **4/4** result;
- exact mandatory regression suite commands and literal pass counts/exit statuses from this VPS venue;
- that the command-execution runtime was diagnosed as externally terminating the process group and the Owner explicitly authorized VPS shell/tmux only for regression completion;
- all already-required P2-002 mapping/concurrency/no-Allocation/lazy-target-TU facts actually supported by the final code/test evidence;
- clean worktrees and explicit P2-003+ exclusions;
- no FINAL PASS / Supervisor acceptance / Owner acceptance claim.

Push the evidence commit to `WMS_Outbound/main` and report its exact SHA.

## STOP

Report:

- Mercato SHA under test;
- each mandatory regression suite and exact result count/exit status;
- whether all mandatory regressions are green;
- WMS evidence SHA if evidence was validly created;
- Scanner frozen SHA;
- confirmation that no product/test files changed during this venue switch.

Then STOP.