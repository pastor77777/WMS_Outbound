# Architect Translation Manifest

**Snapshot date:** 2026-08-31  
**Polish ultimate provenance:** `01_ARCHITECT_SOURCE/2026-08-31/`  
**English operational mirror:** `01_ARCHITECT_TRANSLATIONS/2026-08-31/`

## Language authority policy

- The Polish originals under `01_ARCHITECT_SOURCE/2026-08-31/` are immutable and remain the ultimate provenance for Architect source material.
- The English files under `01_ARCHITECT_TRANSLATIONS/2026-08-31/` are faithful operational readable representations and are the default detailed Architect reading for English-speaking agents.
- The English mirror is not an independent product-decision authority and does not change the existing business authority hierarchy.
- A semantic discrepancy, ambiguity, omission, or mistranslation between an English mirror and its Polish original is a **TRANSLATION DEFECT**, not an architecture conflict.
- In any PL↔EN semantic discrepancy, the Polish original wins and the English translation must be corrected.
- English-speaking agents should not read both languages by default. Read the Polish original only for provenance, exact-wording validation, suspected translation defects, or explicit translation verification.

## 1:1 source-to-translation mapping

Original SHA-256 values below are copied from the immutable `01_ARCHITECT_SOURCE/MANIFEST.md` and were reverified against the raw Polish snapshot bytes during ETAP 2.5 acceptance.

| Polish original path | Original SHA-256 | English translation path | Source version | Source date | Translation status | Review status |
|---|---|---|---|---|---|---|
| `01_ARCHITECT_SOURCE/2026-08-31/SZABLON_PROCESU.md.html` | `30b31ed0d70209dad054114a6e2f902aded2ca7d5bcc5d88d83a79626bd3db29` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/SZABLON_PROCESU_EN.md` | 1.0 | — | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/decyzje_outbound_wms.md` | `e93f57024c2c939563a14dfab80a9a0d73e886e9b8cca87232f72d6b3356f550` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/decyzje_outbound_wms_EN.md` | — | 2026-08-25 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/macierz_odpowiedzialnosci_outbound.md` | `373a3c9e288b733063bf9b96b61424a5671d54e45b19e28ed5112294f303f147` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/macierz_odpowiedzialnosci_outbound_EN.md` | 1.0 | 2026-08-22 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/macierz_pokrycia_outbound.md` | `b567c92ed43038ed21a4b1c57f1f48236043917046ccca2c3772d84d5e3638d1` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/macierz_pokrycia_outbound_EN.md` | — | 28.08.2026 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/model_stanow_outbound.md` | `eef03bf68531564379d120b2b9a6ccffb06af7d78c23e4e539fe79b3efb9dea8` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/model_stanow_outbound_EN.md` | 1.19 | 2026-08-28 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` | `e3b46af2c9961560a2f33e574706c6e9fb796f09da67f0fca98f3bb73c861519` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_1_standard_fulfillment_EN.md` | 1.20 | 2026-08-28 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/proces_2_outbound_crossdock.md` | `544ff95ae80ba60177df38dd7c11b0009f2cb92b1b7aba7c5dcde2a286696b5e` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_2_outbound_crossdock_EN.md` | 1.13 | 2026-08-28 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/proces_3_reservation_release.md` | `29a872bac4eb77d91a570335c1287818c0db4ef9c2331f80c77930a5c3ccee76` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_3_reservation_release_EN.md` | 1.2 | 2026-08-24 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/proces_4_physical_putback.md` | `b4f9852893be72f15e3752fa33ad8e6d252c7e7405fcfdf8d9edfbd20e47de63` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_4_physical_putback_EN.md` | 1.2 | 2026-08-23 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` | `d1aac304f05de56a58588d8f04a683c17cb1ca098f7b04cbd23a47a8be6443ca` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/scenariusze_testowe_outbound_EN.md` | — | 28.08.2026 | COMPLETE | PASS — 2026-08-31 |
| `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` | `ddb0eba86f2506c91a01d41e564a2a761c93f5ec315c6de65fd902f5a99f7b0e` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/wymagania_outbound_EN.md` | — | 28.08.2026 | COMPLETE | PASS — 2026-08-31 |

## ETAP 2.5 acceptance verification

- Polish source count: **11/11**.
- English operational mirror count: **11/11**.
- Recomputed Polish SHA-256 values matching `01_ARCHITECT_SOURCE/MANIFEST.md`: **11/11**; mismatches: **0**.
- Immutable Polish source files modified during ETAP 2.5: **0**.
- Protected PL↔EN identifier consistency: **PASS** — decision IDs, requirement IDs, integration IDs, concurrency IDs, test-case IDs, structured P1–P4 rule IDs, canonical status tokens, canonical domain object names and machine-identifiable domain event names are preserved. Historical withdrawn identifiers remain historical and are not reactivated.
- Focused semantic QA of `decyzje_outbound_wms`, `model_stanow_outbound`, P1, P2, P3 and P4: **PASS**. No unresolved translation defect remains.

## Acceptance rule

The English mirror is accepted only while all 11 Polish source hashes match the immutable source manifest, all 11 English translations exist, protected PL/EN identifier sets remain equal, and focused semantic QA remains green. If a later defect is found, correct EN only; never mutate the immutable PL snapshot.