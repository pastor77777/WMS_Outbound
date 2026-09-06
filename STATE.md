# WMS Outbound — STATE

**As of:** 2026-09-06  
**Campaign:** WMS Outbound v1  
**Architecture:** implementation-ready; no unresolved product/architecture blocker recorded  
**Current phase:** product implementation  
**Implementation progress:** **25/37 items FINAL PASS / Owner Accepted**

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

## Current position

Completed and accepted: **25/37**.

Next authorized implementation item:

**P3-001 — Reservation Release before formal pick — item 26/37.**

Authoritative executor guide:

`06_AGENT_GUIDES/P3-001_EXECUTION.md`

Guide commit:

`403ec74fb97bd920ca0da8965850101e17b5f40c`

Frozen bases:

- Mercato `4f64641ab14a5359bc22d0685e390b511252b5b5`;
- Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- P2-006 evidence `9a580b046b5f2aa3bcbf2422eeaf6413248f68db`.

Grounding:

- `FR-P3-01`, `FR-P3-02`, `FR-P3-03`, `INT-06`;
- P3 R1–R6 and P3/P4 process-entry boundary;
- `TC-040`, `TC-041`.

Core boundary:

- P3 applies only before formal pick (`pickedQty = 0` / no accepted pick confirmation for the released quantity);
- true pre-pick release returns hard reserved stock to availability and ATP exactly once, cancels line/task state consistently, and creates no `PutBackTask`;
- shortage -> CustomerOrderLine `BACKORDERED`; general cancellation -> CustomerOrderLine `CANCELLED`;
- P3-002 owns retention policy/timer R9–R10;
- P3-003 owns the physical-removal-before-confirmation race / exact-source return / P4 handoff R7–R8;
- P4 owns physically picked quantity (`pickedQty > 0`).

## Mandatory new-item reset

Before first P3-001 implementation action:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

Required result: `RESET_OK`.

Do not repeat deep reset for retries/continuations inside the same item.

## Executor / prompt workflow

Detailed workflow: `06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md`.

Prompt skill routing: `06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md` (`9aa7f0db21c1ffe02f22a2d610b95a5acc1d779d`).

Rules:

- full Task Catalog item is the normal executor unit;
- executor owns ordinary implementation/fixture/auth/TLS/tooling/runtime/build/test failures until COMPLETE;
- two-strikes only for the same material unresolved technical path after two genuinely different substantive attempts;
- all executor venues use the same Git/business/evidence contract;
- before writing executor prompts, supervisor refreshes WMS Outbound + Architect/Canon and applies current `fetch_me_prompt` + `operational-mode` guidance;
- executor prose is never acceptance;
- Owner acceptance is explicit after independent supervisor verification.

## Authority and continuity

For Outbound behavior: immutable Architect Source -> faithful Canon/translation -> requirements/traceability -> Task Catalog delivery slice -> current code/DB as implementation evidence.

Inbound remains **CLOSED / REFERENCE**.

Current authoritative handover:

`08_HANDOVER/HANDOVER_CURRENT_2026-09-06.md`

Git truth overrides stale Drive/chat history.