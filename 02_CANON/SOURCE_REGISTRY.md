# Source Registry

For English-speaking agents, use the **English operational mirror** column for normal detailed Architect reading. The Polish path remains immutable ultimate provenance and is used for exact provenance/wording, suspected translation defects, EN/canon inconsistencies, or explicit translation validation. Do not read both by default. If PL and EN differ semantically, classify the difference as a **TRANSLATION DEFECT**; PL wins and EN must be corrected.

| ID | Polish immutable source | English operational mirror | Authority role | Active? |
|---|---|---|---|---|
| AS-P1 | `01_ARCHITECT_SOURCE/2026-08-31/proces_1_standard_fulfillment.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_1_standard_fulfillment_EN.md` | Standard Fulfillment behavior | yes |
| AS-P2 | `01_ARCHITECT_SOURCE/2026-08-31/proces_2_outbound_crossdock.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_2_outbound_crossdock_EN.md` | Outbound Crossdock behavior | yes |
| AS-P3 | `01_ARCHITECT_SOURCE/2026-08-31/proces_3_reservation_release.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_3_reservation_release_EN.md` | Reservation Release behavior | yes |
| AS-P4 | `01_ARCHITECT_SOURCE/2026-08-31/proces_4_physical_putback.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/proces_4_physical_putback_EN.md` | Physical Putback behavior | yes |
| AS-STATE | `01_ARCHITECT_SOURCE/2026-08-31/model_stanow_outbound.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/model_stanow_outbound_EN.md` | State/status/event canon | yes |
| AS-DEC | `01_ARCHITECT_SOURCE/2026-08-31/decyzje_outbound_wms.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/decyzje_outbound_wms_EN.md` | Product decision/reasoning register | yes |
| AS-REQ | `01_ARCHITECT_SOURCE/2026-08-31/wymagania_outbound.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/wymagania_outbound_EN.md` | Derived requirements | yes |
| AS-TC | `01_ARCHITECT_SOURCE/2026-08-31/scenariusze_testowe_outbound.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/scenariusze_testowe_outbound_EN.md` | Acceptance scenarios | yes |
| AS-COV | `01_ARCHITECT_SOURCE/2026-08-31/macierz_pokrycia_outbound.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/macierz_pokrycia_outbound_EN.md` | Rule/exception → requirement → test coverage | yes |
| AS-RACI | `01_ARCHITECT_SOURCE/2026-08-31/macierz_odpowiedzialnosci_outbound.md` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/macierz_odpowiedzialnosci_outbound_EN.md` | Responsibility cross-view | yes |
| AS-TPL | `01_ARCHITECT_SOURCE/2026-08-31/SZABLON_PROCESU.md.html` | `01_ARCHITECT_TRANSLATIONS/2026-08-31/SZABLON_PROCESU_EN.md` | Documentation template/helper | helper |

Raw Drive IDs and immutable SHA-256 hashes are in `01_ARCHITECT_SOURCE/MANIFEST.md`. Translation status, review status and the 1:1 PL→EN hash mapping are in `01_ARCHITECT_TRANSLATIONS/TRANSLATION_MANIFEST.md`.