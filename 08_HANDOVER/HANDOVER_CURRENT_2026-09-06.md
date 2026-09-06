# WMS Outbound — Current Supervisor Handover

**Updated:** 2026-09-06 Europe/Warsaw  
**Project:** WMS Outbound v1  
**Authoritative repository:** `pastor77777/WMS_Outbound`

## Current checkpoint

Plan: **37 items**, **109/109 Architect requirements mapped**.

Formal progress: **25/37 FINAL PASS / Owner Accepted**.

Latest accepted checkpoint:

**P2-006 — Crossdock join into common Shipment/dispatch downstream — item 25/37 — FINAL PASS / Owner Accepted.**

- Mercato: `4f64641ab14a5359bc22d0685e390b511252b5b5`
- Scanner frozen: `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`
- WMS evidence: `9a580b046b5f2aa3bcbf2422eeaf6413248f68db`

Supervisor independently verified P2-006 before Owner acceptance.

## Accepted P2-006 boundary

Preserve exactly:

1. CROSSDOCK Packing TUs use the same common P1 Shipment/TU/content/OutboundOrderLine model; no second crossdock Shipment lifecycle exists.
2. Shipment grouping remains channel-neutral: warehouse + customer + delivery address + priority + identical `slaDeadline`.
3. P2 R43 priority/SLA inheritance remains authoritative.
4. P1 R57/R58 completeness is CustomerOrder-level across STANDARD + CROSSDOCK; `allowPartialShipment=false` blocks every ready TU while any active line is incomplete.
5. `slaDeadline` cannot bypass the no-partial completeness guard.
6. Complete compatible STANDARD + CROSSDOCK contributions of one no-partial CustomerOrder join one common Shipment.
7. Carrier Selection, WMS label, ERP posting and CarrierManifest are the existing P1 common pipeline.
8. P2-005 GR gate remains before ERP Phase 1 and counts only contributing CROSSDOCK source TUs; STANDARD content creates no GR requirement.
9. CROSSDOCK final settlement creates no fake Allocation and no standard Inventory decrement; STANDARD Allocation/Inventory settlement remains exactly once.
10. Crossdock TU/line/order/customer terminal aggregates still advance through the common manifest-confirm lifecycle.
11. Multi-Shipment/final aggregate and replay behavior remains accepted P1-016 behavior.
12. Scanner has no second dispatch model and stayed frozen.

Accepted proof summary:

- dedicated P2-006 canonical PostgreSQL matrix **20/20 PASS**;
- P2-005 **19/19**, P1-011 **18/18**, P1-012 **14/14**, P1-013 **15/15**, P1-014 **18/18**, P1-015 **21/21**, P1-016 **25/25**, P2-002 **22/22**, P2-003 **8/8**, P2-004 **16/16** retained green;
- durable Mercato typecheck/build/runtime proof green;
- final rendered Mercato Playwright **2/2 PASS**, zero route mocks/interception;
- Journey A is continuous: real CROSSDOCK `PACKING_SEALED` TU -> common grouping -> Shipment -> carrier -> label -> GR-blocked ERP -> real GR acceptance -> ERP -> CarrierManifest -> persisted settlement;
- Journey B/C is continuous: mixed STANDARD+CROSSDOCK no-partial guard, SLA non-bypass, one Shipment, crossdock-only GR requirement, common downstream completion, exactly-once STANDARD settlement and allocation-free/no-standard-inventory CROSSDOCK settlement, replay one-Shipment invariant.

## Active next item

**P3-001 — Reservation Release before formal pick — item 26/37.**

Authoritative guide:

`06_AGENT_GUIDES/P3-001_EXECUTION.md`

Guide commit:

`403ec74fb97bd920ca0da8965850101e17b5f40c`

Frozen bases:

- Mercato `4f64641ab14a5359bc22d0685e390b511252b5b5`;
- Scanner `f7817e83babab35dcc2f56c8acf5f21a9e08f1fa`;
- P2-006 evidence `9a580b046b5f2aa3bcbf2422eeaf6413248f68db`.

### P3-001 Architect truth

Grounding:

- P3 R1–R2: pre-pick `Allocation RESERVED -> RELEASED`, stock hard reservation becomes available;
- P3 R3–R4: shortage release cancels pre-pick OOL and leaves unfulfilled CustomerOrderLine `BACKORDERED`;
- P3 R5–R6: general cancellation makes CustomerOrderLine `CANCELLED`; automatic/system release does not require Supervisor notification;
- requirements `FR-P3-01`, `FR-P3-02`, `FR-P3-03`, `INT-06`;
- acceptance `TC-040`, `TC-041`.

Hard boundary:

- P3 is only for **formal `pickedQty = 0`** / no accepted pick confirmation for the released quantity;
- no `PutBackTask` for true pre-pick release;
- formally picked quantity belongs later to P4, not P3;
- P3-002 owns retention policy/timer (`R9-R10`);
- P3-003 owns the exact-source physical-removal-before-confirmation race and P4 handoff (`R7-R8`);
- do not weaken generic accepted P1-004 CON-02; P3 must use a dedicated transactional cancellation orchestration that proves zero formal pick, cancels eligible task work, then releases hard reservation using accepted primitives.

## Mandatory new-item reset

Before launching P3-001 executor work:

```bash
cd /home/ubuntu/git/Devaxonic-WMS
git pull --ff-only
bash scripts/reset-testing-runtime.sh --deep
```

Required result: `RESET_OK`.

This reset is only for the new item boundary; do not repeat it for ordinary P3-001 retries/continuations.

## Operational executor rules — durable

Canonical rule for **Antigravity, local Codex, Codex Cloud, Claude and other authorized executors**:

**whole Task Catalog item -> autonomous implementation/diagnosis/tests/runtime/UI/evidence -> return only COMPLETE or a true blocker.**

Routine fixture/auth/TLS/test-data/selector/tooling/runtime/build failures are executor-owned. Do not create supervisor round-trips after every failure.

Two-strikes applies only to the **same material unresolved technical path** after two genuinely different substantive evidence-based attempts and no normal in-scope next move.

Prompt-generation routing is durable in:

`06_AGENT_GUIDES/PROMPT_SKILL_ROUTING.md`

commit:

`9aa7f0db21c1ffe02f22a2d610b95a5acc1d779d`

Before any new executor guide/ticket, the supervisor must refresh current WMS/Architect authority and use current `fetch_me_prompt` + `operational-mode` guidance. The prompt skill is construction discipline; Architect/Canon/Git remain business authority.

## Fresh supervisor bootstrap

A new ChatGPT supervisor chat must not reconstruct this project from chat memory alone.

Read in this order:

1. current raw Markdown Google Drive fresh-chat handover;
2. `ChatGPT_MEMORY.md` on Google Drive;
3. current `wms-outbound` context/routing;
4. current Architect/Canon context for WMS Outbound; use `architecture-context` for shared/Inbound compatibility, not to rewrite Outbound business truth;
5. `scanner-context` when Scanner is relevant;
6. fresh Git `Devaxonic-WMS/AGENTS.md`, `.ai/STATE.md`, current `.ai/HANDOVER_OUTBOUND_CURRENT_*.md`, `.ai/TESTING.md`, `.ai/OPERATIONS.md`, `.ai/PLAN.md`;
7. fresh Git `WMS_Outbound/AGENTS.md`, `STATE.md`, this handover, `GIT_PROMPT_WORKFLOW.md`, `PROMPT_SKILL_ROUTING.md`, exact Task Catalog item and mapped Architect files;
8. before composing executor prompts, read/apply current `fetch_me_prompt` and `operational-mode` skills.

**Git truth overrides stale Drive/chat history.**

## Supervisor protocol

- Owner controls executor selection/launch/session organization;
- executor never self-accepts;
- supervisor independently verifies final refs/diff/evidence after COMPLETE;
- formal catalog progress advances only after explicit Owner acceptance;
- owner-facing executor prompt remains microscopic because detailed work lives in Git;
- no P3-002 before P3-001 is supervisor-verified and Owner Accepted;
- Demo/Prod and local PostgreSQL remain out of scope.