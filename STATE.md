# WMS Outbound — STATE

**As of:** 2026-09-06  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **28/37 items FINAL PASS / Owner Accepted**

## Architect baseline

Active process set:

- P1 `STANDARD_FULFILLMENT` v1.20
- P2 `OUTBOUND_CROSSDOCK` v1.13
- P3 `RESERVATION_RELEASE` v1.2
- P4 `PHYSICAL_PUTBACK` v1.2
- state model v1.19

Requirements: **109 IDs = 98 FR + 6 INT + 5 CON**.

`PickWave` is out of scope v1. No separate Process 5 exists.

## Accepted implementation checkpoints

1. `FND-001` — FINAL PASS — `a3c8d67f3ec65967fd3405b3da8c9901fb46e192`
2. `FND-002` — FINAL PASS — `d835dc5ec157cb0485cb09cfbc0dfdeedd40c281`
3. `FND-003` — FINAL PASS — `8fa0214d36308b00aacf32b5df369ef486975ae2`
4. `P1-001` — FINAL PASS — `435b51007e5411ebdbb1b3b4d30c984fd770d4c6`
5. `P1-002` — FINAL PASS / Human Verified — `7510ef3f05b6c64c3f9de925a5a85f644913cdfe`
6. `P1-003` — FINAL PASS / Human Verified — `bae31c2c2ad9b1426b868d6df7a05076669ace0d`
7. `P1-004` — FINAL PASS — `71b74b5384b3fbfc55d8ed298f1a3715dc477c3c`
8. `P1-005` — FINAL PASS / Human Verified — Mercato `0ebc0e8ce44263edf9170293f0c5b0d1a5c54975` / Scanner `8199b330cb739a45e2c615a3f2aa3803336be724`
9. `P1-008` — FINAL PASS / Human Verified — Mercato `9512137702a5d5f5b41910c2de97cf03321a1ccd` / Scanner `b5cfb59987c76f39e0ab48af67a52e2e914d9613`
10. `P1-006` — FINAL PASS / Human Verified — Mercato `353a5001cb8f1941971f960e509a8af643e41e5a` / Scanner `7596b7802e7ed55a59dd6dc1f21912ea6331e796` / evidence `43cc7d0e7dd20a48fc00b40150b30275d0c2aa12`
11. `P1-007` — FINAL PASS / Owner Accepted — Mercato `134db31381b4db726cd550abe6ecd4079ac21d8c` / Scanner `b23325aae1c4f83b79d01b3650dbead3486a1041` / evidence `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8`
12. `P1-009` — FINAL PASS / Owner Accepted — Mercato `5d780dabeb605bc657bb521bd2b2fdcc2e516f77` / Scanner `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `6c78a97ece567d90d3cb7d0580bb38669c9f9722`
13. `P1-010` — FINAL PASS / Owner Accepted — Mercato `19dbf77d9dbf5a36b36adc88a9dbb6debdd15643` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b5bb6429717402e0fb6969f7437ddaf673a8a174`
14. `P1-011` — FINAL PASS / Owner Accepted — Mercato `20887f2d74928cf69f447fdd6af20a612f38387c` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `90cc30fc2db15c40d80ef69cb03ffb1e107b51dc`
15. `P1-012` — FINAL PASS / Owner Accepted — Mercato `5019a20be14549ff8cbbf25af5bc61c56888e9e1` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b28f59e7ff41ac6d0a3be4b841410650bc5acd8b`
16. `P1-013` — FINAL PASS / Owner Accepted — Mercato `5e6b70aa81afd28fe3217e4aad216e8a6482a769` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `826b9c477fa86a44a93606265868730e4570ff90`
17. `P1-014` — FINAL PASS / Owner Accepted — Mercato `bef7c0a3e0995e7ecddb29156bdfa3777463a6b6` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `b97f1640621b5b01571efec313b7fa0325c1aedf`
18. `P1-015` — FINAL PASS / Owner Accepted — Mercato `f9b0b89cbd05d723ca36501c5dfb1dd57ce8a2e4` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `f201bd0beb2411b4f87f28ca6562a4fc11e6a249`
19. `P1-016` — FINAL PASS / Owner Accepted — Mercato `dd5ff1493740ffc99e11ce40e0b5ffc6b646f574` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `50c5664fda12caf5c2f7bcdb0e9e86c3495a01c2`
20. `P2-001` — FINAL PASS / Owner Accepted — Mercato `8a264fff5c2ca665294d1e02df90c6f37554fe7f` / Scanner frozen `f4a404600efb1120cb2f1c5b86383ad148cd1e1a` / evidence `941d614966b1d2197d8654b3af924afb6ab14d58`
21. `P2-002` — FINAL PASS / Owner Accepted — Mercato `50b27fdd0c9b495ab612ce458bc90e65428ecb93` / Scanner `6796b70aff9ab53d27a0b36d0764ccc83f4b0440` / evidence `2074e2541b7c28cf3c6031cb48ad68901333625a`
22. `P2-003` — FINAL PASS / Owner Accepted — Mercato `db0ef671b58ab13c2c0685205fbadcae1e1cf628` / Scanner `2ae72fb00db882fecae659b842e91efed17f949f` / evidence `f985d6099bdff939a0471012a25126baa8e216c2`
23. `P2-004` — FINAL PASS / Owner Accepted — Mercato `9859be5c7dee4fe802d4d00478459a19982eddfe` / Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `9fb9abd33c1ff8318b6339efc9b69cce3a3161ac`
24. `P2-005` — FINAL PASS / Owner Accepted — Mercato `069f02d4c5c9b345b688b838eb685be02206afbd` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `0c7cf142e1723ff80e86cfd0f00d4b12c1e4b777` / supervisor correction `cf399679360d8b7fc071f9f958709c3bb99b7c59`
25. `P2-006` — FINAL PASS / Owner Accepted — Mercato `4f64641ab14a5359bc22d0685e390b511252b5b5` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `9a580b046b5f2aa3bcbf2422eeaf6413248f68db`
26. `P3-001` — FINAL PASS / Owner Accepted — Mercato `9fb32493ed9ec443a18a494aa1a8ec3a1bde6d06` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `15c3ad937a4e81d7b67ff96409bd0b6a65553864`
27. `P3-002` — FINAL PASS / Owner Accepted — Mercato `84274acacfbfe0119e270ca5bfbcb723e47d7723` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`
28. `P4-001` — catalog item 29/37 — FINAL PASS / Owner Accepted — Mercato `66e2e8620041d2db1d10d069e286936083667139` / Scanner frozen `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa` / evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`

## Current position

Completed and accepted: **28/37**.

Because P3-003 depended on P4-001, P4-001 was executed and accepted before catalog item 28. That dependency is now satisfied.

Next authorized implementation item:

**P3-003 — Cancellation race: physical movement before formal confirmation — catalog item 28/37.**

Authoritative executor guide:

`06_AGENT_GUIDES/P3-003_EXECUTION.md`

Guide commit:

`80613b4fc516be66b34d200b3375e4b16217ea4f`

Accepted bases:

- Mercato P4-001 `66e2e8620041d2db1d10d069e286936083667139`;
- Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- P4-001 evidence `6780af113cbc31db26449fa6ca2fde5f238ab801`;
- P3-002 evidence `efe6fec205f5f75baa1822c0c5560fbbdd0c9a14`;
- P3-001 evidence `15c3ad937a4e81d7b67ff96409bd0b6a65553864`.

Grounding:

- P3 R7-R8;
- `FR-P3-04`, `INT-06`;
- `TC-042`, `TC-043`;
- P4-001 accepted discriminator/logical cancellation boundary.

Core boundary:

- pre-confirm race means source location + SKU verified/physically removed while authoritative formal `pickedQty = 0`;
- if P3 cancellation wins, reuse P3-001, cancel eligible task work and show exact-source RF return instruction with zero PutBackTask;
- no formal pick state/quantity is created by the pre-confirm observation itself;
- if formal confirmation wins and `pickedQty > 0`, route to accepted P4-001 and do not emit the P3 exact-source instruction;
- server/PostgreSQL locking decides the winner, not Scanner-local state;
- Scanner is now materially in scope for the RF instruction and pre-confirm observation;
- P4-002/P4-003 remain out of scope.

## Mandatory new-item reset

Before first P3-003 implementation action, perform the canonical reset defined in `Devaxonic-WMS/.ai/OPERATIONS.md` and require a successful `RESET_OK` result.

The reset is separate from executor launch. Do not repeat it for retries/continuations inside P3-003.

## Executor / prompt workflow

Detailed workflow: `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

Prompt skill routing: `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`.

Rules:

- full Task Catalog item is the normal executor unit;
- current Owner-selected executor for this item is Codex; executor choice does not change the guide/business/evidence contract;
- executor owns ordinary implementation/fixture/auth/TLS/tooling/runtime/build/test failures until COMPLETE;
- Testing credentials are frozen per canonical `Devaxonic-WMS/.ai/TESTING.md` and are not an implementation/refactor/rotation topic;
- two-strikes only for the same material unresolved technical path after two genuinely different substantive attempts;
- before writing executor prompts, supervisor refreshes WMS Outbound + Architect/Canon and applies current `wms-outbound`, `scanner-context`, `fetch_me_prompt` + `operational-mode`; `architecture-context` is compatibility-only if shared primitives are touched;
- owner-facing handoff contains prompt content only; launcher/VPN/session-start commands are excluded unless Owner explicitly requests them;
- executor prose is never acceptance;
- Owner acceptance is explicit after independent supervisor verification.

## Authority and continuity

For Outbound behavior: immutable Architect Source -> faithful Canon/translation -> requirements/traceability -> Task Catalog delivery slice -> current code/DB as implementation evidence.

Inbound remains **CLOSED / REFERENCE**.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.