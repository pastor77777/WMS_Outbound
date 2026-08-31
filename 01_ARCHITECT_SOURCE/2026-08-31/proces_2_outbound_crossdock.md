# PROCES 2: OUTBOUND_CROSSDOCK

**Projekt:** WMSAI Outbound
**Wersja:** 1.13 | **Data:** 2026-08-28 | **Autor:** Analityk Biznesowy
**Geneza:** dokument powstał przez rozwinięcie zarchiwizowanego `propozycja_procesow_outbound.md` v1.29 §3.2 (Outbound Crossdock) i §3.5 (część dot. P2) do formatu krok-po-kroku zgodnego z `SZABLON_PROCESU.md`. Zarchiwizowany dokument jest materiałem historycznym, nie źródłem prawdy — źródłem prawdy jest ten plik (`DEC-A14`).
**Poprzedni proces:** Inbound Proces 2 (Unloading) — przekazuje Inbound `TU` (`ELEMENTARY`) w statusie `IN_CROSS_DOCK`
**Następny proces:** brak w WMS dla ilości rozliczonej przez cross-docking (fizyczny odbiór poza granicą); Inbound Proces 3 (Putaway) dla ewentualnej rezydualnej ilości. PROCES 1 (Standard Fulfillment) dzieli z tym procesem kroki 9–13 (`Shipment`, Carrier Selection, etykieta, bramka ERP, `CarrierManifest`, wydanie).
**Zakres dokumentu:** opis procesu biznesowego przeładunku towaru przyjętego w Inbound bezpośrednio na wydanie, bez składowania i bez kompletacji z zapasu. Katalog stanów, przejść i zdarzeń domenowych jest utrzymywany w `model_stanow_outbound.md` — ten dokument opisuje zachowanie, nie definiuje nowych stanów.

---

## Cel procesu

Przeładować towar przyjęty w Inbound bezpośrednio na wydanie, dopasowując zawartość Inbound `TU` do `CustomerOrderLine` w `BACKORDERED`, bez etapu składowania i bez tworzenia `Allocation`. Ilość rozliczona przez cross-docking dołącza do standardowego łańcucha `Shipment` → Carrier Selection → `CarrierManifest` (wspólnego z PROCES 1), a ewentualna rezydualna ilość wraca do standardowego Putaway po stronie Inbound.

## Uczestnicy

- **Warehouse Operator (Packer)** — pobranie i kontrola jakości/ilości `SKU` ze źródłowej Inbound `TU`, rozsortowanie do docelowej Outbound `TU` (krok 2).
- **Warehouse Supervisor** — eskalacja przy niedoborze/uszkodzeniu dla `allowPartialShipment = false` (patrz „Wyjątki i ścieżki alternatywne"); wgląd w `grAcceptanceStatus` źródłowych Inbound `TU` blokujących bramkę ERP dla utkniętych `Shipment` (krok 4, **R36**). `Warehouse Operator` przydzielany jest zadaniom automatycznie przez wejście do modułu crossdockingu (**R39**), bez udziału `Warehouse Supervisor`.
- **Dispatcher** — udział we wspólnych krokach 9–13 z PROCES 1 (patrz **KROK 4**).
- **System WMS** — generowanie `CrossDockPickTask`, wyznaczanie kwalifikowalnej ilości, zarządzanie docelową Outbound `TU`, rozliczanie zadania, bramka ERP uwzględniająca oczekiwanie na `GR_ACCEPTED` źródłowych Inbound `TU`.

## Zdarzenie startowe

Inbound `TU` (`ELEMENTARY`) osiąga status `IN_CROSS_DOCK` (granica z Inbound — patrz nota niżej). Kwalifikacja `TU` do cross-dockingu jest wyłącznie odpowiedzialnością Inbound Procesu 2; ten proces przejmuje `TU` już zakwalifikowaną.

## Przebieg procesu

---

### KROK 1 — Generowanie `CrossDockPickTask`
**[System WMS]**

- System WMS odczytuje `CustomerOrderLine` w `BACKORDERED`, uporządkowane według obowiązującej kolejki priorytetowej zamówień, parametryzowanej per magazyn (**R1**).

◇ **Topologia dopasowania między źródłową Inbound `TU` a zapotrzebowaniem?**

**Ścieżka A — pełne dopasowanie 1:1 → [System WMS]**
- Cała zadeklarowana (ASN) zawartość Inbound `TU` domyka jeden `Shipment` po spełnieniu pełnego kryterium wspólnego przeznaczenia z **R2**.
- Powstają `OutboundOrderLine` w `CREATED` dla jednej docelowej Outbound `TU`.

**Ścieżka B — rozsortowanie n:n → [System WMS]**
- Powstaje `OutboundOrderLine` per dopasowana linia, dla nowej albo otwartej Outbound `TU` zgodnej w kliencie/adresie/`priority` oraz o identycznym `slaDeadline`; jedna Outbound `TU` może zbierać `SKU` z kilku Inbound `TU` (**R3**).

- Niezależnie od ścieżki: `CrossDockPickTask CREATED` powstaje z referencją do źródłowej Inbound `TU` i docelowej `OutboundOrderLine`; `CustomerOrderLine BACKORDERED → PLANNED`.
- Aktywna `OutboundOrderLine` jest w tym momencie wyłącznie instrukcją sortowania; `Allocation` nie powstaje w cross-dockingu (**R4**).
- `OutboundOrder` powstaje z atrybutem `fulfillmentChannel = CROSSDOCK`, ustawianym przy tworzeniu i niezmiennym; wszystkie jego `OutboundOrderLine` mają ten sam `fulfillmentChannel` — nie miesza się linii cross-dockowych ze standardowymi w jednym `OutboundOrder` (**R5**).
- Ilość kwalifikowalna do cross-dockingu dla danej `CustomerOrderLine` (`crossDockEligibleQty`) jest wyliczana w tym kroku, przy generowaniu `CrossDockPickTask`, jako min(`sourceEligibleQty`, `demandEligibleQty`) — pełny wzór obu składników w **R6** (**R6**).
- Topologia (1:1 czy n:n) jest znana już w tym momencie, więc `TU_NUMBER`/`SSCC` docelowej Outbound `TU` jest przydzielany już przy jej pierwszym użyciu w **KROK 2**, nie dopiero przy `PACKING_SEALED`: przy pełnym dopasowaniu 1:1 z poprawnym `SSCC` GS1 na źródłowej Inbound `TU` — obowiązkowe dziedziczenie `TU_NUMBER` i `SSCC`; w przeciwnym razie oraz przy rozsortowaniu n:n — nowy numer z `TUSetup`/`Sequence` i nowa etykieta (**R7**).

---

### KROK 2 — Pobranie i kontrola
**[Packer]**

- `CrossDockPickTask CREATED → ASSIGNED` (przydział operatorowi zalogowanemu do modułu crossdockingu, **R39**), następnie Packer rozpoczyna skanowanie: `CrossDockPickTask ASSIGNED → IN_PROGRESS`.
- Rozpoczęcie skanowania przenosi `OutboundOrderLine CREATED → PICKING`, a pierwsze takie zdarzenie w ramach `OutboundOrder` przenosi jego nagłówek `CREATED → PACKING_IN_PROGRESS` (**R8**).
- Od tego momentu ogólne anulowanie (niezwiązane z niedoborem) jest niedozwolone do `PACKED` — System WMS odrzuca takie żądanie ze wskazaniem, że pozycja jest w trakcie rozsortowania, do ponowienia po `PACKED` (**R9**).
- Packer skanuje `SKU`, ilość i jakość (`OK`/`DAMAGED`) według `OutboundOrderLine`.

◇ **Wynik skanu/kontroli danego `SKU`?**

**Ścieżka A — ilość zgodna, jakość `OK` → [Packer]**
- Ilość trafia do docelowej Outbound `TU`. Jeżeli w tym momencie nie istnieje otwarta docelowa Outbound `TU` dla tego zadania, System WMS otwiera nową i nadaje jej `TU_NUMBER` (**R40**, numeracja wg **R7**).
- 🔁 Gdy ta `TU` zapełni się fizycznie przed zakończeniem zadania, Packer zamyka ją (`TU → PACKING_SEALED`) przez skan RF i kontynuuje rozsortowanie pozostałej ilości do nowej, otwartej Outbound `TU` w ramach tego samego `CrossDockPickTask` — normalny krok procesu, nie eskalacja (**R10**).

**Ścieżka B — ilość mniejsza (brak) albo `DAMAGED`, wykryte w trakcie pobrania lub jako `confirmedQty < plannedQty` przy zakończeniu zadania → [Packer / System WMS]**
- Ilość mniejsza wymaga (1) ponownego sprawdzenia i (2) potwierdzenia przez Packera (**R11**); `DAMAGED` kierowana jest na `QC` (Quality Control) (**R12**).
- Wyłącznie ilość zgłoszona jako `DAMAGED` jest rejestrowana jako `damagedQty` zadania — ilość mniejsza wykryta jako brak nie wchodzi do `damagedQty` (**R13**).

  ◇ **`allowPartialShipment`?**

  **`= true` → [System WMS]** — automatycznie: rozsortowana część `OutboundOrderLine → PACKED`, brakująca `CustomerOrderLine → BACKORDERED`.

  **`= false` → [Warehouse Supervisor]** — eskalacja, trzy wyniki analogiczne do `SHORT_PICKED` standardowej kompletacji (patrz **„Wyjątki i ścieżki alternatywne"**):
  - **„czekamy"** — `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, reszta czeka; jeśli `confirmedQty > 0`, System WMS tworzy `PutBackTask` (PROCES 4) dla tej ilości — po `COMPLETED` ilość staje się zwykłym `Inventory AVAILABLE` (`ATP`), nigdy nie wraca do Inbound `TU`/`IN_PUTAWAY` (**R14**).
  - **„anulowanie"** — `Warehouse Supervisor` anuluje pełną wymaganą ilość `CustomerOrderLine` → jeśli `confirmedQty > 0`, `PutBackTask` jak w wariancie „czekamy"; albo koryguje `CustomerOrderLine` do `confirmedQty` → `OutboundOrderLine → PACKED` dla rozsortowanej ilości, kontynuuje normalnie do `PACKING_SEALED`, bez `PutBackTask` (**R15**).
  - **„trwała zmiana `allowPartialShipment` na `true`"** — dalej jak wariant automatyczny.
- W trybie `PICKING → CANCELLED` (put-back/anulowanie po rozpoczęciu rozsortowania), `OutboundOrderLine` przechodzi z `PICKING`, nie z `CREATED` (**R16**).

**Ścieżka C — `SKU` nieoczekiwane → [Packer / System WMS]**
- Trafia na `QC` (**R17**).

**Ścieżka D — `TU` pusta przed pobraniem (zadanie jeszcze `ASSIGNED`, `OutboundOrderLine` jeszcze `CREATED`) → [System WMS]**
- `CrossDockPickTask ASSIGNED → CANCELLED`, powiązane `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, Inbound `TU → LOST` (**R18**).
- Jak w każdej ścieżce anulowania: jeśli w ten sposób wszystkie `OutboundOrderLine` danego `OutboundOrder` osiągną `CANCELLED` bez żadnej `PACKED`, `OutboundOrder → CANCELLED` — kryterium ogólne, nie specyficzne dla tej ścieżki (**R37**).

- **Odzysk po anulowaniu po `PACKED` (ogólne anulowanie, poza niedoborem).** Ta sama zasada odzysku (`PutBackTask → Inventory AVAILABLE`) obowiązuje przy ogólnym anulowaniu cross-dockowej `OutboundOrderLine` po `PACKED` — bez kroku `Allocation RELEASED`, bo cross-docking `Allocation` nie tworzy (**R19**).
- Przejście do **KROK 3**.

---

### KROK 3 — Zakończenie `CrossDockPickTask`
**[System WMS]**

- `CrossDockPickTask IN_PROGRESS → COMPLETED`, gdy Packer zgłosi zakończenie zadania; `confirmedQty` zadania jest wtedy ustalone i nie przekracza jego `plannedQty`. Zakończenie jednego zadania nie zależy od pozostałych zadań tej samej źródłowej Inbound `TU` ani od tego, czy cała jej zadeklarowana (ASN) zawartość została już rozliczona — niezależnie od topologii 1:1, 1:n, n:1 czy n:n (**R20**).

◇ **Czy po zakończeniu wszystkich powiązanych `CrossDockPickTask` pozostaje rezydualna, nieprzypisana fizyczna ilość `SKU`?**

**Ścieżka A — NIE → [System WMS]**
- Inbound `TU → CROSS_DOCKED` (stan Inbound, poza kanonem Outbound).

**Ścieżka B — TAK → [System WMS]**
- Resztka zostaje przekazana do Inbound Procesu 3; `TU` przechodzi na `TRANSIT` sektora i otrzymuje `IN_PUTAWAY` przy zakończeniu tego przewozu (**R21**); dalej standardowy Putaway bez zmian w tym mechanizmie.

- W obu przypadkach System WMS przekazuje Inbound, obok potwierdzonej ilości, sumę `damagedQty` wszystkich powiązanych `CrossDockPickTask` tej `TU` (**R22**).
- Ilość `SKU` `OK` już potwierdzona przez zakończone `CrossDockPickTask` (ta, która trafiła do Outbound `TU`) jest rozliczana odrębnie od ewentualnej rezydualnej ilości: cross-docking nie czeka na fizyczne rozłożenie resztki, żeby uznać już wykonaną część za rozliczoną (**R23**). Ewentualne dalsze rozliczenie rezydualnej ilości po zakończeniu standardowego Putaway tej samej Inbound `TU` jest zdarzeniem niezależnym, poza tym procesem.
- Kryterium przejścia w `CROSS_DOCKED` jest wyłącznie brak rezydualnej ilości, nie liczba docelowych Outbound `TU` (**R24**). Zachowanie `TU_NUMBER`/`SSCC` jest osobną osią — zależy wyłącznie od topologii 1:1 (**KROK 1**), niezależnie od tego rozstrzygnięcia.
- Finalizacja źródłowej Inbound `TU` następuje, gdy nie pozostaje już żaden aktywny `CrossDockPickTask` tej `TU`, i kończy jej udział w cross-dockingu; zapotrzebowanie zgłoszone po finalizacji nie powoduje utworzenia kolejnego zadania cross-dockowego dla tej `TU` (**R28**).
- Docelowa Outbound `TU` wchodzi w `PACKING_SEALED` wyłącznie wg **R10** (zamknięcie przez Packera albo automatyczne po `slaDeadline` bez aktywnych/zaplanowanych zadań). Jedno zadanie mogło w toku rozsortowania wykorzystać więcej niż jedną Outbound `TU`, a jedna Outbound `TU` mogła być zasilana przez kilka zadań z różnych źródłowych Inbound `TU` (**R3**). Outbound `TU`, która przez odzysk `PutBackTask` straciła całą dotąd potwierdzoną ilość i pozostała pusta, przechodzi `CREATED → CANCELLED` zamiast `PACKING_SEALED` (**R34**).
- `OutboundOrderLine → PACKED` niezależnie od stanu docelowej Outbound `TU`, gdy jej rozliczenie ilościowe jest zamknięte: ilość potwierdzona przez jej jedyny `CrossDockPickTask` (**R30**) zrównoważy `OutboundOrderLine.requiredQty` (bez braku ani `DAMAGED` w toku — ścieżki **B**/**C** obsłużone osobno w **KROK 2**). To rozliczenie ilościowe jest jedynym warunkiem `PACKED`; nie zależy ono od tego, czy Outbound `TU`, do której trafiła ilość, osiągnęła już `PACKING_SEALED`.
- **Nota:** crossdockowa Packing `TU` zamówienia z `allowPartialShipment = false` czeka w `PACKING_SEALED` na komplet całego `CustomerOrder`; warunek wspólny dla kanałów określa `P1 R58`.
- Przejście do **KROK 4**.

---

### KROK 4 — `Shipment` i dalej
**[Packer / System WMS / Dispatcher]**

- Wspólne z PROCES 1 (Standard Fulfillment) kroki 9–13: utworzenie/uzupełnienie `Shipment`, Carrier Selection, generowanie etykiety, zgłoszenie gotowości wydania do ERP, `CarrierManifest`, wydanie — patrz `proces_1_standard_fulfillment.md` **KROK 9**–**KROK 13**.
- **Nota:** crossdockowa Packing `TU` zamówienia z `allowPartialShipment = false` nie wchodzi do `Shipment`, lecz czeka w `PACKING_SEALED` na komplet całego `CustomerOrder`, zgodnie z `P1 R58`.
- **Odstępstwo specyficzne dla cross-dockingu w bramce zgłoszenia do ERP (odpowiednik KROK 11A PROCES 1):** dla `Shipment` cross-dockowego wysyłka do ERP czeka, aż dla każdej źródłowej Inbound `TU` zasilającej ten `Shipment` ilością potwierdzoną przez `CrossDockPickTask`, System WMS otrzyma `GR_ACCEPTED` dla ilości rozliczonej przez cross-docking tej `TU` (**R25**).
- Rezydualna ilość źródłowej Inbound `TU` (czekająca na standardowy Putaway, **KROK 3** Ścieżka B) i jej ewentualne późniejsze rozliczenie nie są częścią tego oczekiwania i nie wpływają na gotowość `Shipment` (**R26**).
- Wynik komunikatu GR System WMS koreluje wyłącznie po identyfikatorze źródłowej Inbound `TU` (`sourceInboundTU`) i po wskazaniu źródła rozliczenia `GR_SETTLEMENT_SOURCE = CROSSDOCK`, bez identyfikatora zadania, i ustawia ten sam `grAcceptanceStatus` na wszystkich `CrossDockPickTask` tej Inbound `TU` w systemie, niezależnie od `Shipment` (**R31**). Sygnał niepasujący do żadnego `sourceInboundTU` albo dotyczący rozliczenia putawayowego jest odrzucany bez zmiany `grAcceptanceStatus` (**R32**).
- Warunek bramki jest wyznaczany osobno dla każdego `Shipment`, z pełnego zbioru jego własnych źródłowych Inbound `TU` (**R33**).
- Bramka jest ponownie ewaluowana przy każdym komunikacie GR, w tym gdy `Shipment` jest już w `POSTING_ERROR` z innej przyczyny; jawny `GR_REJECTED` nie przenosi samodzielnie `Shipment` w `POSTING_ERROR` — bramka pozostaje niespełniona do czasu `GR_ACCEPTED` tej `TU` (**R35**).
- `grAcceptanceStatus` źródłowych Inbound `TU` blokujących bramkę jest widoczny dla `Warehouse Supervisor` (**R36**).

---

## Zdarzenie końcowe

Wydanie przez zamknięty `CarrierManifest`; `OutboundOrderLine → SHIPPED`; `CustomerOrderLine → FULFILLED` (agregacja przez Funkcję ciągłą F1 PROCES 1 na poziomie `CustomerOrder` do `PARTIALLY_SHIPPED`/`SHIPPED`). Alternatywnie: wyjścia z procesu bez wydania — Inbound `TU → CROSS_DOCKED`/`IN_PUTAWAY`/`LOST`, powrót `CustomerOrderLine → BACKORDERED`.

---

## Diagram sekwencji

### 6.8 Outbound Crossdock

```mermaid
sequenceDiagram
    participant INB as Proces Inbound
    participant WMS as System WMS
    participant PA as Packer
    participant ERP as ERP
    participant DI as Dispatcher
    INB->>WMS: Inbound TU w IN_CROSS_DOCK
    WMS->>WMS: dopasowanie CustomerOrderLine BACKORDERED
    alt Przypadek 1 - pelne dopasowanie 1:1
        WMS->>WMS: CrossDockPickTask + OutboundOrderLine CREATED
    else Przypadek 2 - rozsortowanie n:n
        WMS->>WMS: CrossDockPickTask + OutboundOrderLine CREATED per linia
    end
    alt E - Inbound TU pusta przed pobraniem
        WMS->>WMS: CrossDockPickTask i linie CANCELLED, CustomerOrderLine BACKORDERED
        WMS->>INB: Inbound TU LOST
    else Inbound TU zawiera ilosc do pobrania
        WMS->>PA: CrossDockPickTask do pobrania
        PA->>WMS: skan SKU, ilosc i kontrola OK/DAMAGED
        alt A - ilosc zgodna i OK
            WMS->>WMS: SKU do docelowej Outbound TU (TU_NUMBER/SSCC przydzielony juz tutaj, R7)
            Note over WMS: PACKING_SEALED - zamkniecie przez Packera (zapelnienie) albo automatycznie po slaDeadline bez aktywnych/zaplanowanych zadan (R10)
        else B - ilosc mniejsza po potwierdzeniu
            WMS->>WMS: OutboundOrderLine CANCELLED, CustomerOrderLine BACKORDERED
        else C - DAMAGED
            PA->>WMS: odlozenie na QC
            WMS->>WMS: OutboundOrderLine CANCELLED, CustomerOrderLine BACKORDERED
        else D - SKU nieoczekiwane
            PA->>WMS: odlozenie na QC
        end
        WMS->>WMS: CrossDockPickTask COMPLETED
        WMS->>WMS: OutboundOrderLine PACKED dla ilosci rozsortowanych
        alt Sciezka A - brak resztki (1:1, 1:n, n:1 lub n:n bez resztki)
        WMS->>INB: Inbound TU CROSS_DOCKED, + damagedQty
            WMS->>WMS: Outbound TU PACKING_SEALED (numer przydzielony wczesniej w kroku pobrania)
            INB->>ERP: GR (rozliczenie pelne, jedna wersja)
        else Sciezka B - resztka fizyczna pozostala
            WMS->>WMS: Outbound TU PACKING_SEALED (numer przydzielony wczesniej, R7)
        WMS->>INB: resztka na TRANSIT, Inbound TU IN_PUTAWAY, + damagedQty
            INB->>ERP: GR (rozliczenie crossdockowe - ilosc juz potwierdzona przez CrossDockPickTask)
            Note over INB: rozliczenie putawayowe rezydualnej ilosci nastapi pozniej, przy zakonczeniu standardowego Putaway - poza tym procesem, nie wplywa na Shipment
        end
        WMS->>WMS: Shipment, Carrier Selection, etykieta
        Note over WMS,ERP: bramka ERP czeka na GR_ACCEPTED rozliczenia crossdockowego kazdej zrodlowej Inbound TU (R25)
        Note over WMS: GR_REJECTED nie zmienia stanu Shipment - brama pozostaje niespelniona do GR_ACCEPTED (R35); grAcceptanceStatus widoczny dla Warehouse Supervisor (R36)
        WMS->>ERP: POST Shipment
        WMS->>DI: Shipment gotowy do CarrierManifest
        DI->>WMS: CarrierManifest CLOSED, wydanie
    end
```

## Obiekty danych

| Obiekt | Kluczowe pola | Status(y) |
|---|---|---|
| Inbound `TU` (`ELEMENTARY`) | — (kanon Inbound, poza `model_stanow_outbound.md`) | `IN_CROSS_DOCK → CROSS_DOCKED`; alternatywnie `IN_CROSS_DOCK → IN_PUTAWAY` (resztka) albo `→ LOST` (`TU` pusta) |
| `CrossDockPickTask` | `sourceInboundTU`, `SKU`, `plannedQty`, `confirmedQty`, `damagedQty`, `targetOutboundOrderLine`, `grAcceptanceStatus` | `CREATED → ASSIGNED → IN_PROGRESS → COMPLETED`; końcowe alternatywne: `CANCELLED` |
| `CustomerOrderLine` | `crossDockEligibleQty` | `BACKORDERED → PLANNED → PARTIALLY_FULFILLED`/`FULFILLED`; powrót do `BACKORDERED` przy niedoborze |
| `OutboundOrder` | `fulfillmentChannel = CROSSDOCK`, `priority`, `slaDeadline` (dziedziczone z macierzystego `CustomerOrder`, **R43**) | `CREATED → PACKING_IN_PROGRESS → PACKED → READY_FOR_DISPATCH → DISPATCHED → COMPLETED`; końcowe alternatywne: `CANCELLED` (brak fazy `ALLOCATION_IN_PROGRESS`/`PICKING_IN_PROGRESS` — pomijane w cross-dockingu) |
| `OutboundOrderLine` | `requiredQty`; `pickedQty` (nie dotyczy — brak formalnego `PickTask`) | `CREATED → PICKING → PACKED → SHIPPED`; końcowe alternatywne: `CANCELLED` |
| Outbound `TU` (`PackUnit`) | `TU_NUMBER`, `SSCC` | `CREATED → PACKING_SEALED` (bezpośrednio, pomija `IN_PICKING`/`READY_TO_PACK`/`PACK_QUALIFIED`) `→ IN_SHIPMENT → DISPATCHED`; końcowe alternatywne: `CANCELLED` |
| `Allocation` | — | **nie powstaje** w cross-dockingu (**R4**) |
| `Inventory` (zakres Outbound) | — | poza mechanizmem standardowej rezerwacji — ilość rozliczana bezpośrednio przez `CrossDockPickTask`/Inbound GR, bez `Allocation RESERVED` |

---

## Wyjątki i ścieżki alternatywne (PROCES 5 — Obsługa wyjątków przekrojowych)

Ta sekcja realizuje PROCES 5 — wyjątki przekrojowe wspólne dla procesów Outbound, bez własnego pliku procesu — `SHORT_ALLOCATED`, `SHORT_PICKED`, `ON_HOLD`, anulowanie, brak wyniku Carrier Selection, błąd etykiety; patrz też `proces_1_standard_fulfillment.md` dla pozostałej części.

| Warunek | Zachowanie | Skutek |
|---|---|---|
| Niedobór/`DAMAGED` przy pobraniu, `allowPartialShipment = false` (**KROK 2**, Ścieżka B) | Eskalacja do `Warehouse Supervisor`, trzy wyniki: „czekamy" (`PutBackTask` dla `confirmedQty > 0`), „anulowanie" (pełne albo korekta do `confirmedQty`), „trwała zmiana `allowPartialShipment` na `true`" | Patrz **R14**–**R15** |
| Niedobór/`DAMAGED`, `allowPartialShipment = true` | Automatycznie: rozsortowana część `→ PACKED`, brakująca `CustomerOrderLine → BACKORDERED` | Bez eskalacji |
| `TU` pusta przed pobraniem (**KROK 2**, Ścieżka D) | `CrossDockPickTask → CANCELLED`, `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, Inbound `TU → LOST` (**R18**) | Jeśli wszystkie linie `OutboundOrder` `CANCELLED` bez żadnej `PACKED` → `OutboundOrder → CANCELLED` — kryterium ogólne (**R37**) |
| Próba ogólnego anulowania (niezwiązanego z niedoborem) w trakcie rozsortowania (`CrossDockPickTask IN_PROGRESS`) | System WMS **odrzuca** żądanie ze wskazaniem, że pozycja jest w trakcie rozsortowania | Można ponowić po osiągnięciu `PACKED` (dalej jak ogólne anulowanie standardowe, PROCES 1 „Wyjątki i ścieżki alternatywne") (**R9**) |
| Jawny `GR_REJECTED` dla źródłowej Inbound `TU` w bramce ERP (**KROK 4**) | Bramka **R25** pozostaje niespełniona; `Shipment` nie zmienia stanu z tego powodu | `grAcceptanceStatus` widoczny dla `Warehouse Supervisor` (**R36**); bramka spełniona po `GR_ACCEPTED` tej `TU` (**R35**) |

---

## Reguły biznesowe

- **R1** — System WMS generuje `CrossDockPickTask` z `CustomerOrderLine BACKORDERED`, uporządkowanych według obowiązującej kolejki priorytetowej zamówień (parametr magazynu).
- **R2** — Pełne dopasowanie 1:1 zachodzi, gdy cała zadeklarowana (ASN) zawartość źródłowej Inbound `TU` pokrywa zapotrzebowanie `CustomerOrderLine` w `BACKORDERED` spełniające łącznie warunki wspólnego wydania: ten sam klient, ten sam adres dostawy, zgodny `priority` i identyczny `slaDeadline`. Przy `allowPartialShipment = true` zapotrzebowanie może pochodzić z kilku `CustomerOrder` tego samego klienta; przy `allowPartialShipment = false` — wyłącznie z jednego. Źródłowa Inbound `TU` może zawierać wiele `SKU` — wspólność dotyczy przeznaczenia, nie zawartości.
- **R3** — Przy rozsortowaniu n:n jedna Outbound `TU` może zbierać `SKU` z kilku Inbound `TU`, dla `OutboundOrderLine` zgodnych w kliencie/adresie/`priority` oraz o identycznym `slaDeadline` (dopasowanie po `slaDeadline` jest ścisłe, nie „zbliżone" — wymagane do jednoznacznego związania z progiem automatycznego zamknięcia **R10**).
- **R4** — Cross-docking nie tworzy `Allocation` — aktywna `OutboundOrderLine` jest wyłącznie instrukcją sortowania.
- **R5** — `OutboundOrder.fulfillmentChannel = CROSSDOCK` jest ustawiany przy tworzeniu i niezmienny; nie miesza się linii cross-dockowych ze standardowymi w jednym `OutboundOrder`.
- **R6** — `crossDockEligibleQty` dla danej `CustomerOrderLine` = min(`sourceEligibleQty`, `demandEligibleQty`), wyliczana przy generowaniu `CrossDockPickTask`. `sourceEligibleQty` = zadeklarowana (ASN) ilość źródłowej Inbound `TU`/`SKU` minus suma `plannedQty` aktywnych `CrossDockPickTask` na tej `TU`/`SKU`, minus suma `confirmedQty` zakończonych `CrossDockPickTask` na tej `TU`/`SKU`, minus suma `damagedQty` zakończonych `CrossDockPickTask` na tej `TU`/`SKU`. `demandEligibleQty` = `CustomerOrderLine.Quantity` minus jej `ATPReservation`, minus suma `requiredQty` wszystkich jej `OutboundOrderLine`, które nie są `CANCELLED`, niezależnie od kanału. Odejmowane są obie formy zabezpieczenia popytu — miękka (`ATPReservation`) i twarda (`requiredQty`) — ponieważ `P1 R11` przenosi ilość z pierwszej do drugiej, a `P1 R59` dopuszcza podział jednej `CustomerOrderLine` między wiele `OutboundOrderLine`. Cross-dockowe `OutboundOrderLine` powstają razem ze swoim `CrossDockPickTask` (**R30**), więc są już objęte tą sumą i nie są odejmowane osobno.
- **R7** — `TU_NUMBER`/`SSCC` docelowej Outbound `TU` jest przydzielany już przy jej pierwszym użyciu w **KROK 2**, nie dopiero przy `PACKING_SEALED`, bo topologia 1:1/n:n jest znana już przy generowaniu zadania. Przy pełnym dopasowaniu 1:1, gdy `TU_NUMBER` źródłowej Inbound `TU` jest poprawnym `SSCC` GS1, docelowa Outbound `TU` musi odziedziczyć `TU_NUMBER` i `SSCC`. Jeżeli `TU_NUMBER` źródłowej Inbound `TU` nie jest poprawnym `SSCC` GS1, docelowa Outbound `TU` otrzymuje nowy numer zgodnie z `TUSetup`/`Sequence` oraz nową etykietę. Przy rozsortowaniu n:n docelowa Outbound `TU` zawsze otrzymuje nowy numer z `Sequence` wskazanej przez jej `TUSetup`.
- **R8** — Rozpoczęcie skanowania (`CrossDockPickTask → IN_PROGRESS`) przenosi `OutboundOrderLine → PICKING`; pierwsze takie zdarzenie w ramach `OutboundOrder` przenosi jego nagłówek `CREATED → PACKING_IN_PROGRESS`.
- **R9** — Od rozpoczęcia rozsortowania do `PACKED` ogólne anulowanie (niezwiązane z niedoborem) jest niedozwolone; System WMS odrzuca żądanie do ponowienia po `PACKED`.
- **R10** — Docelowa Outbound `TU` przechodzi w `PACKING_SEALED` w jednym z dwóch przypadków: (1) Packer zamyka ją skanem RF, gdy zapełni się fizycznie przed zakończeniem zadania, i kontynuuje rozsortowanie do nowej, otwartej `TU` w ramach tego samego `CrossDockPickTask` — normalny krok, nie eskalacja; (2) System WMS zamyka ją automatycznie, gdy jej `slaDeadline` zostanie osiągnięty (albo minięty) i żaden aktywny ani zaplanowany `CrossDockPickTask` nie wskazuje jej już jako celu. Przed osiągnięciem `slaDeadline` sam brak aktywnych/zaplanowanych zadań nie zamyka `TU` — towar cross-dockowy dla tego samego klienta/adresu/`priority`/`slaDeadline` (**R3**) może wciąż nadejść kolejną Inbound dostawą tego samego dnia albo w kolejnych dniach, więc przedwczesne zamknięcie obniżałoby wypełnienie `TU`; `slaDeadline` wyznacza twardą granicę tego oczekiwania. Ponieważ jedna Outbound `TU` może być zasilana przez kilka zadań z różnych źródłowych Inbound `TU` (**R3**), zakończenie pojedynczego zadania nie zamyka jej samo w sobie.
- **R11** — Ilość mniejsza niż planowana wymaga ponownego sprawdzenia i potwierdzenia przez Packera, zanim zostanie zarejestrowana jako brak.
- **R12** — Towar `DAMAGED` kierowany jest na `QC`.
- **R13** — Wyłącznie ilość zgłoszona jako `DAMAGED` wchodzi do `damagedQty` zadania; brak wykryty jako ilość mniejsza nie wchodzi do `damagedQty`.
- **R14** — Wynik „czekamy" przy niedoborze (`allowPartialShipment = false`): `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`; dla `confirmedQty > 0` powstaje `PutBackTask` (PROCES 4), po którym ilość staje się `Inventory AVAILABLE`, nigdy nie wraca do Inbound `TU`/`IN_PUTAWAY`.
- **R15** — Wynik „anulowanie" przy niedoborze: pełne anulowanie `CustomerOrderLine` (z `PutBackTask` dla `confirmedQty > 0`) albo nadzorowana przez `Warehouse Supervisor` korekta `CustomerOrderLine.Quantity` do `confirmedQty` wraz ze zmianą `OutboundOrderLine.requiredQty` do tej samej wartości (`P1 R69`), bez `PutBackTask`, z kontynuacją do `PACKING_SEALED`.
- **R16** — Przy anulowaniu cross-dockowej `OutboundOrderLine` po rozpoczęciu rozsortowania przejście do `CANCELLED` następuje z `PICKING`, nie z `CREATED`.
- **R17** — `SKU` nieoczekiwane w źródłowej `TU` trafia na `QC`, bez uruchamiania mechanizmu braku.
- **R18** — `TU` pusta przed pobraniem: `CrossDockPickTask → CANCELLED`, powiązana `OutboundOrderLine → CANCELLED`, `CustomerOrderLine → BACKORDERED`, Inbound `TU → LOST`. Kryterium zamknięcia całego `OutboundOrder` w tym przypadku — patrz **R37** (ogólne, nie specyficzne dla tej ścieżki).
- **R19** — Odzysk `PutBackTask → Inventory AVAILABLE` obowiązuje też przy ogólnym anulowaniu cross-dockowej `OutboundOrderLine` po `PACKED`, bez kroku `Allocation RELEASED` (bo `Allocation` nie istnieje w cross-dockingu).
- **R20** — `CrossDockPickTask → COMPLETED`, gdy Packer zgłosi zakończenie zadania; `confirmedQty` zadania jest wtedy ustalone i nie przekracza jego `plannedQty`. Warunek dotyczy wyłącznie tego zadania — jego zakończenie nie zależy od pozostałych `CrossDockPickTask` tej samej źródłowej Inbound `TU` ani od tego, czy cała jej zadeklarowana (ASN) zawartość została już rozliczona, niezależnie od topologii.
- **R21** — Rezydualna, nieprzypisana ilość po zakończeniu wszystkich powiązanych `CrossDockPickTask` jest przekazywana do Inbound Procesu 3 (Putaway); `TU` przechodzi na `TRANSIT` sektora i otrzymuje `IN_PUTAWAY` przy zakończeniu tego przewozu (**R28**), dalej standardowy Putaway. Ilość rezydualną per `SKU` wyznacza Inbound Proces 3 (Putaway) — tam należy jej definicja i kontrola ilościowa — ze składników dostarczonych przez ten proces: zadeklarowanej (ASN) ilości źródłowej Inbound `TU` pomniejszonej o sumę `confirmedQty` **oraz** o sumę `damagedQty` wszystkich zakończonych `CrossDockPickTask` tej `TU` (**R22**). Ilość uszkodzona pomniejsza ilość rezydualną, bo fizycznie opuszcza źródłową `TU` na `QC` (**R12**) i nie jest już w niej oczekiwana; pominięcie tego składnika zawyża resztkę oczekiwaną przy Putaway. Uwaga na podobieństwo wzorów: w momencie wyznaczania ilości rezydualnej, czyli po finalizacji źródłowej `TU`, gdy nie pozostaje już żadne aktywne zadanie (**R28**), suma `plannedQty` aktywnych zadań wynosi zero i wielkość ta pokrywa się liczbowo z `sourceEligibleQty` (**R6**) — poza tym momentem oba wzory dają różne wyniki i nie są zamienne.
- **R22** — System WMS przekazuje Inbound sumę `damagedQty` wszystkich powiązanych `CrossDockPickTask` danej `TU`, niezależnie od tego, czy `TU` osiąga `CROSS_DOCKED` czy `IN_PUTAWAY`.
- **R23** — Ilość `SKU OK` już potwierdzona przez zakończone `CrossDockPickTask` jest rozliczana odrębnie od ewentualnej rezydualnej ilości — proces nie czeka na fizyczne rozłożenie resztki.
- **R24** — Kryterium przejścia Inbound `TU` w `CROSS_DOCKED` jest wyłącznie brak rezydualnej ilości, nie liczba docelowych Outbound `TU`.
- **R25** — Bramka zgłoszenia do ERP dla `Shipment` cross-dockowego czeka na `GR_ACCEPTED` dla każdej źródłowej Inbound `TU` zasilającej ten `Shipment` ilością potwierdzoną przez `CrossDockPickTask`.
- **R26** — Rezydualna ilość źródłowej Inbound `TU` i jej późniejsze rozliczenie nie wpływają na gotowość `Shipment` do zgłoszenia w ERP.
- **R28** — Finalizacja źródłowej Inbound `TU` następuje, gdy nie pozostaje żaden aktywny `CrossDockPickTask` tej `TU`, i kończy jej udział w cross-dockingu. Przy zerowej ilości rezydualnej finalizacja nadaje `CROSS_DOCKED`. Przy niezerowej ilości rezydualnej finalizacja jest zgłoszeniem przekazania `TU` do Inbound Procesu 3 (Putaway); statusu `IN_PUTAWAY` ten proces nie nadaje — nadaje go Inbound przy zakończeniu własnego przewozu `TU` na `TRANSIT` sektora, więc do tego momentu `TU` pozostaje w `IN_CROSS_DOCK`. Zapotrzebowanie zgłoszone po finalizacji nie powoduje utworzenia kolejnego `CrossDockPickTask` dla tej `TU` — jest obsługiwane standardowo z zapasu po zakończeniu Putaway.
- **R29** — Ta sama fizyczna ilość źródłowej Inbound `TU`/`SKU` może być jednocześnie `planned` albo `confirmed` w co najwyżej jednym aktywnym `CrossDockPickTask`; `plannedQty` rezerwuje tę ilość do potwierdzenia. Zamek ilościowy cross-dockingu znajduje się na `CrossDockPickTask`, nie na `OutboundOrderLine`.
- **R30** — Generowanie `CrossDockPickTask` i `OutboundOrderLine` dla danej źródłowej Inbound `TU` jest zdarzeniem jednorazowym, wyzwalanym przez jej przejście w `IN_CROSS_DOCK` (**KROK 1**), dopasowującym wyłącznie `CustomerOrderLine BACKORDERED` istniejące w tym momencie. Nie istnieje mechanizm dogenerowania kolejnych `CrossDockPickTask` do już utworzonej `OutboundOrderLine` ani automatycznej aktualizacji jej `requiredQty` po utworzeniu — również w oknie między `IN_CROSS_DOCK` a finalizacją tej samej źródłowej `TU` (**R28**). Jawna, nadzorowana korekta `CustomerOrderLine.Quantity` i `OutboundOrderLine.requiredQty` w gałęzi **R15** pozostaje dopuszczalna zgodnie z `P1 R69`. Każda cross-dockowa `OutboundOrderLine` powstaje razem z dokładnie jednym `CrossDockPickTask`; zamek ilościowy pozostaje na `CrossDockPickTask`, nie na `OutboundOrderLine` (**R29**).
- **R31** — Wynik komunikatu GR otrzymany z ERP dla jednej elementarnej Inbound `TU` System WMS koreluje wyłącznie po identyfikatorze źródłowej Inbound `TU` (`sourceInboundTU`) oraz po wskazaniu źródła rozliczenia `GR_SETTLEMENT_SOURCE = CROSSDOCK`, bez identyfikatora zadania. Po dopasowaniu System WMS ustawia ten sam `grAcceptanceStatus` (`GR_ACCEPTED` albo `GR_REJECTED`) na wszystkich `CrossDockPickTask` w systemie, których `sourceInboundTU` odpowiada wskazanej Inbound `TU` — niezależnie od tego, do którego `Shipment` poszczególne zadania zasiliły `confirmedQty`, także gdy jedna Inbound `TU` zasiliła zadania należące do kilku różnych `Shipment`.
- **R32** — System WMS odrzuca sygnał GR i nie zmienia `grAcceptanceStatus` żadnego zadania, gdy wskazana Inbound `TU` nie odpowiada `sourceInboundTU` żadnego `CrossDockPickTask` albo gdy sygnał dotyczy rozliczenia innego niż crossdockowe. Rozróżnienie rozliczenia crossdockowego od putawayowego opiera się wyłącznie na wskazaniu źródła rozliczenia (`GR_SETTLEMENT_SOURCE`), nigdy na numerze wersji rozliczenia — numeracja wersji liczy się osobno w obrębie każdego źródła, więc „wersja 1" nie identyfikuje rozliczenia crossdockowego.
- **R33** — Warunek bramki **R25** System WMS wyznacza osobno dla każdego `Shipment`, z pełnego zbioru jego własnych źródłowych Inbound `TU`. Wynik `GR_ACCEPTED` jednej źródłowej Inbound `TU` nie wystarcza, gdy ten sam `Shipment` zawiera wkład z innych źródłowych Inbound `TU`.
- **R34** — Gdy docelowa Outbound `TU` utworzona w trakcie aktywnego `CrossDockPickTask` traci przez odzysk `PutBackTask` całą dotąd potwierdzoną ilość i pozostaje pusta bez żadnej innej potwierdzonej pozycji, a jednocześnie żaden aktywny ani zaplanowany `CrossDockPickTask` nie wskazuje jej już jako celu, przechodzi `CREATED → CANCELLED` i nigdy nie wchodzi w `PACKING_SEALED`. Dopóki takie zadanie ją wskazuje, `TU` pozostaje w `CREATED` jako pusta, otwarta jednostka docelowa. W odróżnieniu od zamknięcia w `PACKING_SEALED` (**R10**) anulowanie nie czeka na `slaDeadline` — pusta `TU` nie ma czego przetrzymywać, więc ta asymetria względem **R10** jest zamierzona.
- **R35** (nowa) — Bramka **R25** jest ponownie ewaluowana przy każdym komunikacie GR (`GR_ACCEPTED` lub `GR_REJECTED`) dotyczącym którejkolwiek wymaganej źródłowej Inbound `TU`, niezależnie od poprzedniego wyniku dla tej `TU` czy stanu `Shipment` — w tym gdy `Shipment` znajduje się w `POSTING_ERROR` z innej przyczyny (np. odrzucenie przez ERP, PROCES 1 **KROK 11A**). Jawny `GR_REJECTED` nie przenosi samodzielnie `Shipment` w `POSTING_ERROR` — bramka pozostaje niespełniona (analogicznie do rozróżnienia niespełnionego warunku wstępnego od faktycznego błędu zgłoszenia, model Inbound `D54`); gdy dla tej samej `TU` nadejdzie następnie `GR_ACCEPTED`, bramka jest spełniona jak każda inna, bez odrębnej ścieżki odzyskania.
- **R36** (nowa) — `grAcceptanceStatus` źródłowych Inbound `TU` blokujących bramkę **R25** jest widoczny dla `Warehouse Supervisor` (odczyt na poziomie `CrossDockPickTask`/`Shipment` oczekującego), bez wprowadzania nowego stanu, obiektu ani osobnego mechanizmu eskalacji — wyłącznie ekspozycja istniejącego pola.
- **R37** (nowa) — `OutboundOrder → CANCELLED`, gdy wszystkie jego `OutboundOrderLine` osiągną `CANCELLED` bez żadnej `PACKED` — niezależnie od ścieżki prowadzącej do anulowania poszczególnych linii (`TU` pusta przed pobraniem — **R18**; niedobór z wynikiem „czekamy" albo pełne „anulowanie" — **R14**/**R15**; ogólne anulowanie w trakcie rozsortowania jest zablokowane do `PACKED` — **R9** — więc nie może samo prowadzić do tego stanu przed `PACKED`).
- **R39** (nowa) — `Warehouse Operator` wybiera na terminalu RF moduł crossdockingu. System WMS przydziela mu kolejny `CrossDockPickTask` według tej samej kolejności co `PickTask` (parametr magazynu, `P1 R14`), gdy operator nie ma aktywnego zadania magazynowego żadnego typu. Moduł crossdockingu nie ma wskazania miejsca pracy analogicznego do strefy w zbieraniu — proces nie modeluje fizycznego umiejscowienia rozsortowania.
- **R40** (nowa) — System WMS gwarantuje, że aktywny `CrossDockPickTask` zawsze ma dostępny cel, do którego może odłożyć potwierdzoną ilość — nie istnieje stan, w którym aktywne zadanie nie ma gdzie odłożyć towaru. Mechanizm: jeżeli w momencie odłożenia kolejnej pozycji nie istnieje otwarta docelowa Outbound `TU` dla tego zadania — niezależnie od przyczyny jej braku: zamknięcia fizycznie pełnej `TU` (**R10**), anulowania pustej `TU` po pełnym odzysku (**R34**), albo braku jakiejkolwiek `TU` na początku rozsortowania — System WMS otwiera w tym właśnie momencie nową Outbound `TU` i nadaje jej `TU_NUMBER` zgodnie z **R7**. Utworzenie jest więc leniwe: następuje przy pierwszym odłożeniu `SKU` do tej `TU`, nie z wyprzedzeniem w momencie zamknięcia albo anulowania poprzedniej.
- **R41** (nowa) — Do tego procesu trafia wyłącznie Inbound `TU` typu `ELEMENTARY`; rozsortowanie zawsze odbywa się z `TU` `ELEMENTARY`. `TU` `AGGREGATE` nigdy nie osiąga `IN_CROSS_DOCK`: gdy `SKU` potrzebne do linii `BACKORDERED` jest zadeklarowane w `TU` przyczepionej do zbiorczej, Inbound Proces 2 wymusza dezintegrację zbiorczej przed transportem i przekazuje wyłącznie `TU` `ELEMENTARY`, każdą własną trasą. Ten proces nie przewiduje żadnej ścieżki dla `TU` `AGGREGATE`.
- **R42** (nowa) — Kwalifikacja dokonana przez Inbound Proces 2 jest przesłanką transportu, nie wiążącym przydziałem. Wiążące dopasowanie do `CustomerOrderLine` `BACKORDERED` powstaje dopiero przy przejściu źródłowej `TU` w `IN_CROSS_DOCK` (**R30**), na kolejce priorytetowej aktualnej w tym momencie — zapotrzebowanie może zniknąć w czasie fizycznego transportu do strefy cross-dock. Gdy **R30** nie znajdzie ani jednego dopasowania, System WMS nie tworzy żadnego `CrossDockPickTask`, a cała zadeklarowana ilość źródłowej `TU` jest ilością rezydualną: `TU` finalizuje się natychmiast (**R28**) i zostaje przekazana do Inbound Procesu 3 z ilością rezydualną równą pełnej deklaracji; `IN_PUTAWAY` otrzymuje dopiero przy zakończeniu przewozu na `TRANSIT` sektora, więc do tego momentu pozostaje w `IN_CROSS_DOCK` (**R21**). Odzysk po utracie demand przy już utworzonym zadaniu (**R14**) tego przypadku nie obejmuje.
- **R43** — Crossdockowy `OutboundOrder` dziedziczy `priority` i `slaDeadline` z macierzystego `CustomerOrder`. Grupowanie linii w jeden crossdockowy `OutboundOrder` jest dopuszczalne wyłącznie przy zgodnym kliencie, zgodnym adresie dostawy, zgodnym `priority` i identycznym `slaDeadline`. Przy `allowPartialShipment = false` grupowanie obejmuje wyłącznie linie jednego `CustomerOrder`.

## Uwagi projektowe (do weryfikacji)

- **Etap 2 (poza wersją 1):** konsolidacja Outbound `TU` po `PACKED` dla cross-dockingu (ścieżka `READY_TO_PACK`) — pozostaje otwarte w `BACKLOG.md` B12.

## Powiązanie z procesami sąsiednimi

- **Poprzedni:** Inbound Proces 2 (Unloading) — przekazuje Inbound `TU` (`ELEMENTARY`) w stanie `IN_CROSS_DOCK`.
- **Następny:** brak procesu WMS następującego wprost dla ilości rozliczonej (fizyczny odbiór poza granicą, wspólny z PROCES 1 od **KROK 4** w dół); Inbound Proces 3 (Putaway) przejmuje ewentualną rezydualną ilość (Inbound `TU → IN_PUTAWAY`).

## Historia zmian

- **1.13 (2026-08-28)** — Zamknięcie `BACKLOG.md` B22: człon `demandEligibleQty` w **R6** odejmuje teraz obie formy zabezpieczenia popytu — `ATPReservation` i sumę `requiredQty` niezanulowanych `OutboundOrderLine` tej linii. Dotychczasowe brzmienie odejmowało wyłącznie `ATPReservation`, więc ilość przeniesiona do rezerwacji twardej przez `P1 R11` znikała ze wzoru i mogła zostać przyznana drugi raz kanałem crossdockowym. Usunięto też niezdefiniowaną frazę „brakująca ilość `CustomerOrderLine` w `BACKORDERED`" oraz osobny człon o przypisaniach innych `CrossDockPickTask`, wchłonięty przez sumę `requiredQty`. Bez zmian w `sourceEligibleQty` i w pozostałych regułach.
- **1.12 (2026-08-28)** — WP-09: w **KROKU 3** i **KROKU 4** dodano notę, że crossdockowa Packing `TU` zamówienia z `allowPartialShipment = false` czeka w `PACKING_SEALED` na komplet całego `CustomerOrder` zgodnie ze wspólnym guardem `P1 R58`; bez powielania reguły i bez nowego wiersza wyjątku.
- **1.11 (2026-08-27)** — Wprowadzono `OutboundOrderLine.requiredQty` do „Obiektów danych” i przepięto na niego warunek `PACKED` w **KROKU 3** oraz **R30**. **R15** jawnie aktualizuje `requiredQty` razem z nadzorowaną korektą `CustomerOrderLine.Quantity`; **R30** nadal wyklucza dogenerowanie zadań i automatyczną aktualizację, zgodnie z `P1 R69`.
- **1.10 (2026-08-27)** — Zgodnie z `DEC-L52` przywrócono pełne kryterium dopasowania 1:1 jako wspólność przeznaczenia całej zadeklarowanej zawartości źródłowej Inbound `TU`, z dopuszczeniem wielu `SKU` i regułami `allowPartialShipment`; w KROKU 1 przywrócono liczbę mnogą `OutboundOrderLine`. Nowa **R43**, zgodna z `DEC-L54`, wskazuje dziedziczenie `priority` i `slaDeadline` crossdockowego `OutboundOrder` z macierzystego `CustomerOrder` oraz ścisłe warunki grupowania.
- **1.9 (2026-08-23)** — Rozdzielenie finalizacji źródłowej Inbound `TU` od nadania jej statusu `IN_PUTAWAY`, po zgłoszeniu rozbieżności przez sesję Inbound. **R28**, **R21** i **R42** mówiły dotąd skrótem, że finalizacja jest przejściem w `IN_PUTAWAY`; w rzeczywistości finalizacja przy niezerowej ilości rezydualnej jest zgłoszeniem przekazania `TU` do Inbound Procesu 3, a `IN_PUTAWAY` nadaje Inbound przy zakończeniu własnego przewozu `TU` na `TRANSIT` sektora — do tego momentu `TU` pozostaje w `IN_CROSS_DOCK`. Doprecyzowano też prozę KROKU 3. Bez nowych reguł, wymagań i scenariuszy.
- **1.8 (2026-08-23)** — Domknięcie granicy Inbound→Outbound Crossdock po zmianach zgłoszonych przez sesję Inbound. Nowa **R41**: do procesu trafia wyłącznie Inbound `TU` `ELEMENTARY`, `TU` `AGGREGATE` nigdy nie osiąga `IN_CROSS_DOCK` (Inbound wymusza dezintegrację zbiorczej przed transportem). Nowa **R42**: kwalifikacja Inbound jest przesłanką transportu, a nie wiążącym przydziałem — wiążące dopasowanie powstaje przy `IN_CROSS_DOCK` (**R30**), a przy zerowym dopasowaniu `TU` finalizuje się natychmiast do `IN_PUTAWAY` z całą ilością rezydualną. Uzupełniono kwalifikator `(ELEMENTARY)` w nagłówku "Poprzedni proces".
- **1.7 (2026-08-23)** — Domknięcie pozycji audytu V3-CD dotyczącej ilości rezydualnej (Faza 1c pkt C). **R21** uzupełnione o składniki ilości rezydualnej — zadeklarowana (ASN) ilość źródłowej Inbound `TU` minus suma `confirmedQty` i suma `damagedQty` zakończonych zadań — wraz ze wskazaniem, że definicja i kontrola ilościowa należą do Inbound Proces 3, a ten proces wyłącznie dostarcza składniki. Dotąd termin „ilość rezydualna" był używany w **R21**, **R23**, **R24** i w **KROK 3** bez żadnej definicji w tym korpusie, mimo że **R24** opiera na nim kryterium przejścia źródłowej `TU` w `CROSS_DOCKED`. Dopisano ostrzeżenie o niezamienności z `sourceEligibleQty` (**R6**): oba wzory zbiegają się wyłącznie w momencie finalizacji `TU`, gdy `plannedQty` aktywnych zadań wynosi zero. Bez zmiany zachowania — uzupełnienie samowystarczalności zapisu.
- **1.6 (2026-08-23)** — Domknięcie pozycji audytu V3-CD-02 (PutBack vs współdzielona docelowa Outbound `TU`), niezamkniętej we wcześniejszej rundzie. **R34** uzupełnione o warunek braku aktywnych/zaplanowanych `CrossDockPickTask` wskazujących `TU` jako cel — dotychczasowy guard był wyłącznie ilościowy, przez co odzysk `PutBackTask` mógł anulować `TU`, do której nadal kierowało inne zadanie (dangling target reference). Zapisano jawnie, że brak warunku `slaDeadline` przy anulowaniu jest zamierzoną asymetrią wobec **R10**. Nowa **R40** — niezmiennik ciągłości celu: System WMS gwarantuje, że aktywny `CrossDockPickTask` zawsze ma dostępny cel do odłożenia towaru — przy braku otwartej `TU` otwiera nową i nadaje `TU_NUMBER` (**R7**); mechanizm istniał dotąd wyłącznie w gałęzi zamknięcia pełnej `TU` w **R10** (**KROK 2** uzupełniony o to odwołanie), teraz obowiązuje niezależnie od przyczyny braku otwartej `TU`. Numer R39 jest już zajęty przez zasady podejmowania zadań w module RF (wpis wyżej) — stąd kolejny wolny numer. Nagłówek metryki „Źródło" przeredagowany na „Geneza" (`DEC-A14`).
- **1.5 (2026-08-23)** — Zasady podejmowania zadań magazynowych (`BACKLOG.md` B10): nowa **R39** — operator wybiera moduł crossdockingu na terminalu RF, System WMS przydziela kolejny `CrossDockPickTask` tą samą kolejnością co `PickTask`, bez wskazania strefy (świadoma decyzja negatywna). Zaktualizowano sekcję „Uczestnicy" i **KROK 2**. Uzasadnienie pełne w `decyzje_outbound_wms.md` `DEC-L42`/`DEC-L44`.
- **Bez zmiany wersji (2026-08-23)** — wycofanie konceptu puli operatorów (decyzja właściciela produktu): usunięto regułę **R38** przywróconą 2026-08-22, fragment o przypisaniu zadania z puli w sekcji „Uczestnicy" oraz odwołanie do puli zadań w KROK 2. Numer **R38** pozostaje celowo nieużyty. Mechanizm przydziału zadania zostanie opisany osobno.
- **1.4 (2026-08-22)** — Naprawa luki kompletności migracji (nie luka spójności): reguła `R75` z globalnego katalogu `propozycja_procesow_outbound.md` §7 nigdy nie została przeniesiona do żadnego pliku procesowego — przywrócona jako nowa lokalna **R38** (pierwszeństwo `CrossDockPickTask` nad `PickTask` przy przypisaniu operatora do obu pul). Zaktualizowano sekcję „Uczestnicy" (rola `Warehouse Supervisor`) o odwołanie do **R38**.
- **1.3 (2026-08-22)** — Domknięcie audytu V3-CD (owner decisions #1–#3): **R30** przeredagowane — usunięty błędny zapis o zasilaniu jednej `OutboundOrderLine` przez wiele `CrossDockPickTask` z różnych źródłowych `TU` (generowanie jest jednorazowe, per źródłowa `TU`, KROK 1); dodany zamknięty zakres generowania. **R10** rozszerzone o twarde ograniczenie czasowe (`slaDeadline`) dla automatycznego zamknięcia Outbound `TU` — zapobiega przedwczesnemu zamknięciu przy towarze cross-dockowym nadchodzącym kolejnymi dostawami Inbound tego samego adresu w ciągu dnia lub kilku dni; **R3** zaostrzone do identycznego `slaDeadline` dla jednoznacznego dopasowania. **KROK 3** rozdzielone: zamknięcie Outbound `TU` (`PACKING_SEALED`, **R10**) i przejście `OutboundOrderLine → PACKED` (wyłącznie rozliczenie ilościowe, niezależne od stanu `TU`) opisane osobno. **R27** usunięte — jawny `GR_REJECTED` nie przenosi już `Shipment → POSTING_ERROR`; nowe **R35** (ponowna ewaluacja bramki **R25** przy każdym komunikacie GR, także po wcześniejszym `POSTING_ERROR`) i **R36** (widoczność `grAcceptanceStatus` dla `Warehouse Supervisor`, opcja C1). **R18** zawężone do samego skutku dla `TU` pustej przed pobraniem; ogólne kryterium `OutboundOrder → CANCELLED` (wszystkie linie `CANCELLED`, brak `PACKED`) wydzielone do nowego **R37**, obowiązującego niezależnie od ścieżki anulowania (zgodnie z `DEC-L13`). Zaktualizowano diagram sekwencji §6.8, sekcję „Uczestnicy" (wgląd `Warehouse Supervisor` w `grAcceptanceStatus`) oraz tabelę „Wyjątki i ścieżki alternatywne".
- **1.2 (2026-08-22)** — Naprawa luk propagacji ujawnionych audytem spójności: `R20` przywrócone do właściwego podmiotu (zakończenie zadania zgłasza Packer; kryterium całej zadeklarowanej zawartości dotyczy źródłowej Inbound `TU`, nie zadania); `R6` uzupełnione o pełny wzór `crossDockEligibleQty` ze składnikami `sourceEligibleQty`/`demandEligibleQty`; `R7` uzupełnione o reguły dziedziczenia i przenumerowania `TU_NUMBER`/`SSCC`; `R10` rozszerzone o drugi wyzwalacz `PACKING_SEALED`. Nowe `R28`–`R34`: finalizacja źródłowej `TU` i brak ponownego cross-dockingu po niej, zamek ilościowy na zadaniu, wiele zadań na jedną `OutboundOrderLine`, kontrakt korelacji GR po `sourceInboundTU` i `GR_SETTLEMENT_SOURCE`, odrzucanie sygnału niepasującego lub putawayowego, bramka wyznaczana per `Shipment`, pusta Outbound `TU` po pełnym `PutBackTask`. Diagram sekwencji przepięty z globalnej na lokalną numerację reguł.
- **Bez zmiany wersji (2026-08-22)** — redakcyjna konsolidacja P5 w sekcji „Wyjątki i ścieżki alternatywne (PROCES 5 — Obsługa wyjątków przekrojowych)”, bez osobnego pliku i bez zmiany numeracji `Rn`.
- **1.1 (2026-08-22)** — Przeniesiono diagram sekwencji §6.8 z `propozycja_procesow_outbound.md` przy archiwizacji B16.
- **1.0 (2026-08-18)** — Wersja bazowa. Rozwinięcie `propozycja_procesow_outbound.md` v1.29 §3.2 (kroki 1–4) i części §3.5 dotyczącej P2 do formatu krok-po-kroku zgodnego z `SZABLON_PROCESU.md`, z lokalną numeracją `R1`–`R27`. Realizacja `BACKLOG.md` B5.
