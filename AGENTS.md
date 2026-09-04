# AGENTS.md — WMS Outbound authority and execution rules

## 1. Active scope

The active campaign is **WMS Outbound v1**. This repository is the primary routing target for Outbound analysis, planning, implementation traceability and acceptance evidence.

Do not treat the closed Inbound campaign as current work. Inbound remains historical/reference evidence.

## 2. Mandatory read order

Before changing Outbound behavior, an English-speaking agent normally reads:

1. `STATE.md`
2. `02_CANON/AUTHORITY.md`
3. `02_CANON/SOURCE_REGISTRY.md`
4. `02_CANON/OUTBOUND_GOLDEN_RECORD.md`
5. the specific English Architect mirror file under `01_ARCHITECT_TRANSLATIONS/2026-08-31/` for the behavior being changed
6. `03_TRACEABILITY/requirements_index.csv` and `coverage_matrix.csv`
7. `04_CURRENT_STATE/*` for existing implementation facts
8. the relevant task in `07_IMPLEMENTATION_PLAN/TASK_CATALOG.md`
9. `05_EVIDENCE/EVIDENCE_STANDARD.md`

For Scanner work additionally read `06_AGENT_GUIDES/SCANNER_ROUTING.md`.

### Executor/runtime bootstrap

For implementation, test, runtime or evidence work on the canonical WMS engineering VPS, also read `/home/ubuntu/git/Devaxonic-WMS/AGENTS.md`, `.ai/TESTING.md` and `.ai/OPERATIONS.md` before declaring environment/runtime unavailable.

Canonical VPS Testing facts shared by Codex and Antigravity:

- repos live under `/home/ubuntu/git/`;
- approved Testing DB env source is `/etc/mercato-localhost.env`;
- source that env in the **same shell invocation** as every DB-backed Jest/migration/Playwright command;
- never print `DATABASE_URL` or other secret-bearing env values;
- Testing Mercato is `mercato-localhost.service` on `http://localhost:3009`;
- Testing Scanner is `scanner-testing.service` on `http://localhost:8081`;
- inspect the canonical local Devaxonic-WMS credential instructions before asking the owner for UI credentials.

A fresh executor session on the same VPS is not a fresh machine. Do not rediscover these locations from scratch or claim they are missing until the canonical paths/services have actually been checked.

### Language routing

- The immutable Polish Architect snapshot under `01_ARCHITECT_SOURCE/2026-08-31/` is ultimate provenance. Its hashes are recorded in `01_ARCHITECT_SOURCE/MANIFEST.md`.
- The paired files under `01_ARCHITECT_TRANSLATIONS/2026-08-31/` are the default detailed Architect reading for English-speaking agents. Mapping/review metadata is in `01_ARCHITECT_TRANSLATIONS/TRANSLATION_MANIFEST.md`.
- Do **not** read PL and EN for every implementation task. Read the Polish original only when exact provenance or wording is required, a translation ambiguity/defect is suspected, EN and canon appear inconsistent, or explicit translation validation is requested.
- If EN and PL differ semantically, this is a **TRANSLATION DEFECT**, not an architecture conflict. The Polish original wins and the EN mirror must be corrected.
- The EN mirror is a readable representation, not a new authority layer. Adding translations does not change the business hierarchy below.

## 3. Authority hierarchy

When sources disagree:

1. active process prose `proces_1_standard_fulfillment.md` through `proces_4_physical_putback.md` decides current process behavior;
2. `model_stanow_outbound.md` is the canonical state/status naming and transition catalog, but process prose wins on behavioral conflict;
3. `decyzje_outbound_wms.md` is the canonical product decision/reasoning register; historical entries do not override later active process prose;
4. `wymagania_outbound.md` is a derived requirements layer;
5. `scenariusze_testowe_outbound.md` is the acceptance scenario layer;
6. coverage/responsibility matrices are traceability/cross-view layers;
7. templates, prompts, handovers, memory, old code and generic WMS/scanner knowledge are not business authority.

Never edit `01_ARCHITECT_SOURCE/<snapshot>/` in place.

Conceptually: `immutable Architect Source → faithful representation/canon → implementation as current-state evidence → implementation plan as delivery route`. The implementation plan may sequence delivery of approved requirements; it may not create, broaden or override business requirements.

## 4. Evidence classification

Use these labels explicitly when reasoning:

- `FACT` — directly verified from architect source, source code, database, runtime or acceptance evidence.
- `DERIVED` — logical consequence of verified facts.
- `GAP` — target behavior differs from current state or is missing.
- `CONFLICT` — active authoritative sources contradict each other.
- `OPEN QUESTION` — sources do not answer a required product/architecture decision.

Implementation difficulty, missing code, schema migration and legacy incompatibility are **GAPs**, not product blockers.

## 5. No silent design

Do not add future-proof features, new processes, PickWave, commodity compatibility, carrier label APIs or other functionality without architect authority.

Do not simplify architect behavior to match existing code.

## 6. Shared Inbound assets

`wms_inventory`, `wms_tu`, warehouse assignment, record locks and orchestration are shared foundations. Preserve accepted Inbound behavior. Extend shared models only with explicit compatibility tests.

Inbound Putaway is not Outbound Physical Putback. Reuse technical primitives where valid, not business semantics.

## 7. Human Verified delivery rule

Human-facing behavior is not accepted because a unit/API test is green.

For each human-facing implementation task, evidence must normally include:

`architect source → requirement → task → component test → real application UI → Playwright user journey → persisted/observable result → human walkthrough`

Playwright must click/type/scan through the normal Mercato or Scanner UI as a real role. Direct DB/API setup may prepare fixtures, but must not replace the user action being accepted.

Final acceptance is **Human Verified** for user-facing flows. Technical observability supports the proof; it does not substitute for the UI journey.

## 8. Definition of Done for an implementation task

A task is done only when:

- referenced requirements are implemented without unapproved scope expansion;
- schema/API/UI changes are traceable to source;
- component/integration tests pass;
- concurrency/idempotency tests exist where required;
- human-facing flows have Playwright evidence;
- relevant role/warehouse/session context is used;
- persisted result is verified;
- regression tests protect accepted Inbound/shared behavior where touched;
- evidence location is recorded;
- task DoD is satisfied.

## 9. Product-code boundary of this repository

Documentation/canon/traceability/evidence index updates belong here. Mercato and Scanner product code belongs in their repositories. Database migrations belong with the owning code module and must follow the implementation plan.

Do not start implementation merely because this repository contains a ready plan; `STATE.md` must authorize the implementation phase.