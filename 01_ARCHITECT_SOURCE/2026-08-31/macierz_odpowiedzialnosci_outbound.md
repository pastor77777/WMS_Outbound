# Macierz odpowiedzialności Outbound

**Projekt:** WMSAI Outbound
**Wersja:** 1.0 | **Data:** 2026-08-22 | **Autor:** Analityk Biznesowy
**Geneza:** przeniesione z zarchiwizowanego `propozycja_procesow_outbound.md` v1.31 §8. Zarchiwizowany dokument jest materiałem historycznym, nie źródłem prawdy — źródłem prawdy jest ten plik (`DEC-A14`).
**Zakres:** przekrojowy widok odpowiedzialności P1–P5, nie zastępuje `proces_N`

---

## Macierz odpowiedzialności

„System WMS" wskazano tylko dla działań automatycznych. Kolumna „Zatwierdza wyjątek" pusta = brak wyjątku wymagającego eskalacji na tym etapie.

| Etap | Wykonuje | Zatwierdza wyjątek | Obiekt | Rezultat biznesowy |
|---|---|---|---|---|
| Walidacja zamówienia | System WMS | — | `CustomerOrder` | `ACCEPTED` / `REJECTED` / `ON_HOLD` |
| Cykliczne planowanie | System WMS | — | `OutboundOrder` | utworzony `OutboundOrder` |
| Alokacja | System WMS | Supervisor tylko wg polityki/wyjątku | `Allocation` | zapas `ATP` zarezerwowany albo automatyczna obsługa niedoboru |
| Tworzenie `PickTask` | System WMS | — | `PickTask` | zadania dla operatorów |
| Kompletacja | Picker / System WMS (auto-realokacja) | Supervisor po braku automatu lub osiągnięciu limitu | Picking `TU`, `OutboundOrderLine` | towar w Picking `TU` albo nowy `PickTask` |
| Ocena/pakowanie | Packer | Supervisor (odstępstwo od sugestii) | Outbound `TU` (Packing) | Packing `TU` gotowe |
| Tworzenie `Shipment` | Packer / System WMS | — | `Shipment` | `Shipment` z Packing `TU` |
| Carrier Selection | System WMS | Supervisor (brak wyniku) | `Shipment` | przewoźnik ustalony |
| Ręczny wybór carriera (zewn. `TU`) | Dispatcher | Supervisor | `Shipment` | przewoźnik zatwierdzony |
| Etykietowanie | System WMS | — | `Shipment` | etykieta wygenerowana |
| Manifest i wydanie | Dispatcher | — | `CarrierManifest` | zamknięcie, przekazanie |
| Zwolnienie rezerwacji (P3) | System WMS | Supervisor (wg polityki) | `Allocation` | zapas `AVAILABLE` |
| Fizyczny put-back (P4) | Operator | Supervisor (po zapakowaniu) | `PutBackTask`, `Inventory` | towar z powrotem `AVAILABLE` |
| Outbound Crossdock | Packer / System WMS (automatyczne dopasowanie i klasyfikacja) | Supervisor (niedobór/`DAMAGED` przy `allowPartialShipment = false` — `P2 R14`/`P2 R15`; wgląd w `grAcceptanceStatus` przy bramce ERP wstrzymanej przez `GR_REJECTED` — `P2 R36`) | `CrossDockPickTask`, `OutboundOrderLine`, Outbound `TU` | Outbound `TU` w `PACKING_SEALED`, gotowe do `Shipment` |

## Historia zmian

- **Bez zmiany wersji (2026-08-22)** — partia 3/5 domknięcia audytu V3-CD: nagłówek metryki „Źródło" przeredagowany na „Geneza" (`DEC-A14`); uzupełniona kolumna „Zatwierdza wyjątek" w wierszu Outbound Crossdock — Supervisor przy niedoborze/`DAMAGED` dla `allowPartialShipment = false` (`P2 R14`/`P2 R15`) oraz wgląd w `grAcceptanceStatus` przy bramce ERP wstrzymanej przez `GR_REJECTED` (`P2 R36`, opcja C1 z partii 1).
- **1.0 (2026-08-22)** — Przeniesienie przekrojowej macierzy odpowiedzialności P1–P5 z `propozycja_procesow_outbound.md` §8 przy realizacji B16.
