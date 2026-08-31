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

## Historical material

The active architect documents explicitly mark `propozycja_procesow_outbound.md` as archived and historical. The older `../Archiwum/zlecenie_procesy_outbound_wms.md` is withdrawn. Neither may be used to override the active source set.

There is no active standalone `proces_5`. The former cross-cutting exception concept is represented in P1/P2 exception sections and `FR-P5-*`.

## Non-authoritative implementation sources

Existing code and database prove current implementation state only. They cannot override business target behavior.

`scanner-context` provides scanner/RF/AIDC standards, vendor facts and reference design only. It cannot establish Outbound warehouse rules.

Closed Inbound canon is a reference for knowledge organization and shared implementation regression, not an Outbound business source.

## Change rule

If the architect source changes, capture a new immutable dated snapshot, recompute hashes, update this authority record/canon/traceability, and explicitly identify what was superseded. Never mutate an existing snapshot to make it look current.
