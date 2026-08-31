# Outbound authority model

## Canonical hierarchy

| Priority | Source | Role |
|---:|---|---|
| 1 | P1–P4 active process prose | Current business behavior and process boundaries |
| 2 | `model_stanow_outbound.md` | Canonical status names, transitions, events and actors; process prose resolves behavioral divergence |
| 3 | `decyzje_outbound_wms.md` | Canonical product decision/reasoning history; later process prose contains the current result |
| 4 | `wymagania_outbound.md` | Derived functional/integration/concurrency requirements |
| 5 | `scenariusze_testowe_outbound.md` | Acceptance/E2E scenario layer |
| 6 | coverage/responsibility matrices | Traceability and responsibility cross-view |
| 7 | `SZABLON_PROCESU` | Documentation helper only |

## Language routing and provenance

The hierarchy above is unchanged by translation.

- Ultimate Architect provenance is the immutable Polish snapshot under `01_ARCHITECT_SOURCE/2026-08-31/`, verified by `01_ARCHITECT_SOURCE/MANIFEST.md`.
- For English-speaking agents, the default detailed Architect reading is the paired English operational mirror under `01_ARCHITECT_TRANSLATIONS/2026-08-31/`; exact PL→EN mapping is recorded in `01_ARCHITECT_TRANSLATIONS/TRANSLATION_MANIFEST.md` and `02_CANON/SOURCE_REGISTRY.md`.
- The English mirror is a faithful representation of the Architect Source, not an independent product-decision authority.
- Read the Polish original only when exact provenance/wording is required, a translation ambiguity or defect is suspected, EN and canon appear inconsistent, or explicit translation validation is requested. Do not read both languages by default.
- If EN is ambiguous, incomplete, mistranslated or semantically different from PL, classify it as a **TRANSLATION DEFECT**. The Polish original wins and EN must be corrected. This is not an architecture conflict.

Conceptually the delivery authority remains:

`immutable Architect Source → faithful representation/canon → implementation as current-state evidence → implementation plan as delivery route`.

The implementation plan may organize delivery of approved requirements; it cannot create or override business requirements.

## Historical material

The active architect documents explicitly mark `propozycja_procesow_outbound.md` as archived and historical. The older `../Archiwum/zlecenie_procesy_outbound_wms.md` is withdrawn. Neither may be used to override the active source set.

There is no active standalone `proces_5`. The former cross-cutting exception concept is represented in P1/P2 exception sections and `FR-P5-*`.

## Non-authoritative implementation sources

Existing code and database prove current implementation state only. They cannot override business target behavior.

`scanner-context` provides scanner/RF/AIDC standards, vendor facts and reference design only. It cannot establish Outbound warehouse rules.

Closed Inbound canon is a reference for knowledge organization and shared implementation regression, not an Outbound business source.

## Change rule

If the architect source changes, capture a new immutable dated snapshot, recompute hashes, update this authority record/canon/traceability, and explicitly identify what was superseded. Never mutate an existing snapshot to make it look current.