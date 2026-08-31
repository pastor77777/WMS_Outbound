# WMSAI Outbound — Scenariusze testowe

**Projekt:** WMSAI Outbound  
**Data:** 28.08.2026 | **Autor:** Analityk Rozwiązania (kompetencje Architekta Rozwiązania)  
**Źródło:** `wymagania_outbound.md`, `proces_1_standard_fulfillment.md` v1.20, `proces_2_outbound_crossdock.md` v1.13, `proces_3_reservation_release.md` v1.2, `proces_4_physical_putback.md` v1.2, `model_stanow_outbound.md` v1.19

## Zakres i poziomy

- **Testy komponentowe (K)** — kryteria akceptacji Given/When/Then przy każdym wymaganiu w `wymagania_outbound.md`; nie są tu duplikowane.
- **Testy scenariuszowe (S)** — poniższe przebiegi E2E i przypadki przekrojowe: happy path, kardynalność, konsolidacja, wyjątki, integracja i współbieżność.

**Konwencja:** `TC-nnn` · *Poziom* · *Powiązane wymagania* · *Dane* · Gherkin (`Zakładając / Gdy / Wtedy / Oraz`).

---

## 1. P1 — Standard Fulfillment

### TC-001 — Pełny przebieg standardowy
*Poziom:* E2E · *Wymagania:* FR-P1-01/02/03/04/05/06/07/08/09/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/26/27 · *Dane:* 1 zamówienie, 1 linia, pełny ATP
```gherkin
Scenariusz: Zamówienie przechodzi od importu do POSTED
  Zakładając, że poprawne zamówienie ma pełny zapas ATP
  Gdy WMS zaplanuje, zaalokuje, skompletuje, spakuje i wyda wysyłkę
  Wtedy Shipment przechodzi do POSTED
  Oraz statusy zamówienia i linii odzwierciedlają zakończenie
```

### TC-002 — Brak ATP i ponowne planowanie
*Poziom:* Proces · *Wymagania:* FR-P1-03/04/05/06
```gherkin
Scenariusz: Linia bez ATP wraca do kolejki
  Zakładając, że ATP dla SKU wynosi zero
  Gdy cykl planowania próbuje utworzyć Allocation
  Wtedy linia nie przechodzi do kompletacji
  Oraz pozostaje dostępna dla kolejnego cyklu
```

### TC-003 — Częściowa realizacja
*Poziom:* Proces · *Wymagania:* FR-P1-04/05/25/26
```gherkin
Scenariusz: Dostępna część ilości jest realizowana
  Zakładając, że allowPartial = true i ATP pokrywa część żądania
  Gdy WMS kończy alokację
  Wtedy tylko przydzielona ilość trafia do PickTask
  Oraz zapas non-ATP nie jest rezerwowany
```

### TC-004 — Kompletacja wielostrefowa
*Poziom:* Proces · *Wymagania:* FR-P1-06/07/08/09
```gherkin
Scenariusz: Kilka PickTask zasila jedną linię
  Zakładając, że zapas znajduje się w kilku strefach
  Gdy operatorzy potwierdzą wszystkie zadania
  Wtedy wynik jest skonsolidowany dla właściwej linii
  Oraz każda ilość jest rozliczona jednokrotnie
```

### TC-005 — Direct pack
*Poziom:* Proces · *Wymagania:* FR-P1-09/10/11
```gherkin
Scenariusz: Towar trafia bezpośrednio do paczki
  Zakładając, że konfiguracja dopuszcza direct pack
  Gdy kompletacja zostaje potwierdzona
  Wtedy WMS tworzy i zamyka paczkę bez osobnej konsolidacji
```

### TC-006 — Repakowanie i konsolidacja
*Poziom:* Proces · *Wymagania:* FR-P1-10/11/12/13
```gherkin
Scenariusz: Wiele TU tworzy paczki wysyłkowe
  Zakładając, że kompletacja powstała w kilku TU
  Gdy operator konsoliduje i pakuje zawartość
  Wtedy powstają paczki zgodne z regułami pakowania
  Oraz powiązanie z liniami pozostaje zachowane
```

### TC-007 — Carrier Selection i etykieta
*Poziom:* Proces · *Wymagania:* FR-P1-14/15/16/17/18
```gherkin
Scenariusz: Poprawny przewoźnik i etykieta
  Zakładając, że zamknięta paczka ma komplet danych przewozowych
  Gdy WMS wykona Carrier Selection i wygeneruje etykietę
  Wtedy paczka jest gotowa do przypisania do Shipment
```

### TC-008 — Manifest i ERP
*Poziom:* E2E · *Wymagania:* FR-P1-19/20/21/22/23/24
```gherkin
Scenariusz: Wydanie zostaje zmanifestowane i zaksięgowane
  Zakładając, że Shipment spełnia termin graniczny i warunki manifestu
  Gdy manifest zostanie zamknięty, a ERP zaakceptuje Shipment POST
  Wtedy Shipment ma POSTED
  Oraz CarrierManifest pozostaje zamknięty
```

### TC-009 — Agregacja statusów
*Poziom:* Proces · *Wymagania:* FR-P1-25/27
```gherkin
Scenariusz: Status nagłówka wynika ze statusów linii
  Zakładając, że linie zamówienia są w różnych stanach
  Gdy jedna linia zmieni stan końcowy
  Wtedy WMS przelicza statusy nadrzędnych obiektów
```

### TC-010 — Unikalność numeru Outbound TU
*Poziom:* Proces · *Wymagania:* FR-P1-28
```gherkin
Scenariusz: Numer aktywnej TU nie jest powielany
  Zakładając, że w magazynie istnieje aktywna Outbound TU o danym TU_NUMBER
  Gdy powstaje kolejna Outbound TU
  Wtedy otrzymuje inny TU_NUMBER
  Oraz numer zostaje zachowany, gdy Picking TU pełni dalej rolę Packing TU
  Oraz TU_NUMBER, który nie jest SSCC, jest alfanumeryczny, ma maksymalnie 20 znaków i jest kodowany w Code 128
```

### TC-104 — Niekompletne zamówienie nie jest wydawane
*Poziom:* Proces · *Wymagania:* FR-P1-31

```gherkin
Scenariusz: Niekompletne zamówienie nie jest wydawane
  Zakładając, że CustomerOrder ma allowPartialShipment = false
  Gdy brakuje choć jednej wymaganej pozycji
  Wtedy System WMS nie wydaje Shipment
```

### TC-105 — Termin mija przed pełnym spakowaniem
*Poziom:* Proces · *Wymagania:* FR-P1-32

```gherkin
Scenariusz: Termin mija przed pełnym spakowaniem
  Zakładając, że CustomerOrder ma allowPartialShipment = false, a jego OutboundOrder nie osiągnął PACKED
  Gdy upływa slaDeadline
  Wtedy Packing TU tego zamówienia pozostaje w PACKING_SEALED
  Oraz nie wchodzi do żadnego zamykanego Shipment
```

### TC-106 — Jedna pozycja klienta trafia do dwóch zleceń
*Poziom:* Proces · *Wymagania:* FR-P1-33

```gherkin
Scenariusz: Jedna pozycja klienta trafia do dwóch zleceń
  Zakładając, że CustomerOrderLine wymaga ilości większej niż jedno zlecenie obejmie
  Gdy System WMS planuje OutboundOrder
  Wtedy pozycja zostaje podzielona między wiele OutboundOrderLine
```

### TC-107 — Niezgodny adres blokuje wspólne pakowanie
*Poziom:* Proces · *Wymagania:* FR-P1-34

```gherkin
Scenariusz: Niezgodny adres blokuje wspólne pakowanie
  Zakładając, że dwa OutboundOrder mają różne adresy dostawy
  Gdy Packer próbuje przepakować ich pozycje do jednej Packing TU
  Wtedy System WMS nie pozwala na wspólne pakowanie
```

### TC-108 — Ręczne zdjęcie flagi przy trwającej przyczynie
*Poziom:* Proces · *Wymagania:* FR-P1-35

```gherkin
Scenariusz: Ręczne zdjęcie flagi przy trwającej przyczynie
  Zakładając, że CustomerOrder ma WARNING z powodu braku pełnej rezerwacji
  Gdy Warehouse Supervisor ustawia flagę na false, a rezerwacja nadal jest niepełna
  Wtedy kolejny przebieg planowania ustawia WARNING ponownie
  Oraz zachowanie biznesowe zamówienia nie zmienia się
```

### TC-109 — Wariant wspólnej TU przechodzącej przez zadania
*Poziom:* Proces · *Wymagania:* FR-P1-36

```gherkin
Scenariusz: Wariant wspólnej TU przechodzącej przez zadania
  Zakładając, że magazyn skonfigurował wariant wspólnej Picking TU
  Gdy operator kończy zadanie i kontynuuje ten sam OutboundOrder w innej strefie
  Wtedy zbiera do tej samej Picking TU
```

### TC-110 — Masa liczona z zawartości
*Poziom:* Proces · *Wymagania:* FR-P1-37

```gherkin
Scenariusz: Masa liczona z zawartości
  Zakładając, że w TU leży 20 sztuk SKU o wadze jednostkowej 2 kg i 10 sztuk SKU o wadze 1 kg
  Gdy System WMS wyznacza masę TU
  Wtedy masa wynosi 50 kg
```

### TC-111 — Dobór przewoźnika po masie policzonej z zawartości
*Poziom:* Proces · *Wymagania:* FR-P1-37

```gherkin
Scenariusz: Dobór przewoźnika po masie policzonej z zawartości
  Zakładając, że Shipment zawiera dwie Packing TU o bieżącej masie 50 kg i 80 kg, na typach opakowania o udźwigu 800 kg
  Gdy System WMS uruchamia Carrier Selection
  Wtedy do dopasowania przedziału wagi używa 80 kg, a nie 800 kg
```

### TC-114 — Lekka, ale dobrze zapełniona TU spełnia progi
*Poziom:* Proces · *Wymagania:* FR-P1-38

```gherkin
Scenariusz: Lekka, ale dobrze zapełniona TU spełnia progi
  Zakładając, że typ `TU` ma `TUSetup.externalIssuable = true` i `TUSetup.processUsage` inne niż `EXTERNAL`, a bieżąca masa nie osiągnęła `TUSetup.minIssueWeight`
  Gdy bieżąca objętość zawartości osiągnęła `TUSetup.minIssueVolume`
  Wtedy System WMS uznaje progi wydania za spełnione
```

### TC-115 — Ostatnia TU zamówienia poniżej progów
*Poziom:* Proces · *Wymagania:* FR-P1-39

```gherkin
Scenariusz: Ostatnia TU zamówienia poniżej progów
  Zakładając, że `TU` ma `TUSetup.externalIssuable = true` i `TUSetup.processUsage` inne niż `EXTERNAL`, nie osiągnęła dolnych progów i jest ostatnią `TU` swojego `OutboundOrder`
  Gdy `Warehouse Operator` zamyka ją i podaje powód
  Wtedy System WMS wydaje `TU` bez przepakowania
  Oraz zapisuje powód wymuszenia
```

### TC-116 — Przepakowanie do typu niewydawalnego
*Poziom:* Proces · *Wymagania:* FR-P1-40

```gherkin
Scenariusz: Przepakowanie do typu niewydawalnego
  Zakładając, że Packer przepakowuje towar do nośnika typu bez flagi externalIssuable
  Gdy próbuje zapieczętować Packing TU
  Wtedy System WMS nie pozwala jej zapieczętować
```

### TC-117 — Typ niewydawalny blokuje wymuszenie operatora
*Poziom:* Proces · *Wymagania:* FR-P1-39

```gherkin
Scenariusz: Ostatnia TU na typie niewydawalnym nie może zostać wydana
  Zakładając, że `TU` ma `TUSetup.externalIssuable = false`, nie osiągnęła dolnych progów i jest ostatnią `TU` swojego `OutboundOrder`
  Gdy `Warehouse Operator` próbuje zamknąć ją i wymusić wydawalność
  Wtedy System WMS nie pozwala jej zapieczętować ani wydać
  Oraz jedyną drogą jest przepakowanie na typ wydawalny zgodnie z `P1 R66`
```

### TC-118 — Kontynuacja PickTask w kolejnej Picking TU
*Poziom:* Proces · *Wymagania:* FR-P1-41

```gherkin
Scenariusz: Jedno zadanie korzysta z kolejnych Picking TU
  Zakładając, że `PickTask` ma do pobrania ilość większą niż mieści pierwsza Picking `TU`
  Gdy pierwsza `TU` osiąga `PICK_FULL`, Picker zamyka ją i skanuje kolejną Picking `TU`
  Wtedy pierwsza `TU` przechodzi `PICK_FULL → READY_TO_PACK`, a druga `CREATED → IN_PICKING`
  Oraz ten sam `PickTask` zachowuje tożsamość i pozostaje `IN_PROGRESS`
  Oraz `PickTask` osiąga `COMPLETED` dopiero po pobraniu pełnej ilości
```

### TC-120 — Rozpoznanie i wydawalność TU pochodzenia zewnętrznego
*Poziom:* Proces · *Wymagania:* FR-P1-38/FR-P1-39/FR-P1-42

```gherkin
Scenariusz: Typ wskazany przez tuSetupCode rozstrzyga pochodzenie TU
  Zakładając, że katalog zawiera dokładnie jeden `TUSetup` o `processUsage = EXTERNAL` i `externalIssuable = true`
  Oraz `TU.tuSetupCode` wskazuje ten typ
  Gdy System WMS rozpoznaje pochodzenie `TU` i ocenia progi wydania
  Wtedy rozpoznaje ją jako zewnętrzną wyłącznie po `processUsage` wskazanego typu, bez użycia `TU_NUMBER` ani dodatkowej flagi
  Oraz pierwsza gałąź `P1 R64` uznaje progi za spełnione bez odczytu `minIssueWeight` i `minIssueVolume`
  Oraz ręczne wymuszenie z `P1 R65` nie jest wymagane
```

### TC-121 — Cykl życia requiredQty
*Poziom:* Proces · *Wymagania:* FR-P1-43

```gherkin
Scenariusz: TEST A — korekta standardowa
  Zakładając, że `CustomerOrderLine.Quantity = 100`, a jej jedna `OutboundOrderLine.requiredQty = 100`
  Oraz `SHORT_PICKED` zakończył się z `pickedQty = 30`
  Gdy `Warehouse Supervisor` koryguje `CustomerOrderLine.Quantity` do 30 w `P1 R46`
  Wtedy `OutboundOrderLine.requiredQty = 30`

Scenariusz: TEST B — korekta crossdockowa
  Zakładając, że crossdockowa `OutboundOrderLine.requiredQty = 100`, a jej jedyny `CrossDockPickTask.confirmedQty = 30`
  Gdy `Warehouse Supervisor` koryguje `CustomerOrderLine.Quantity` do 30 w gałęzi `P2 R15`
  Wtedy `OutboundOrderLine.requiredQty = 30`
  Oraz warunek `PACKED` porównuje `confirmedQty = 30` z `requiredQty = 30`

Scenariusz: TEST C — brak korekty i warunek równości
  Zakładając, że `CustomerOrderLine.Quantity = 100`, a jedyna `OutboundOrderLine` ma `requiredQty = 30` i status `PACKED`
  Oraz dla pozostałej ilości nie istnieje żadna `OutboundOrderLine`
  Gdy System WMS ocenia warunek równości
  Wtedy suma `requiredQty = 30` nie jest równa `CustomerOrderLine.Quantity = 100`
  Oraz warunek pozostaje niespełniony
```

### TC-122 — Guard dla aktywnej linii bez OutboundOrderLine
*Poziom:* Proces · *Wymagania:* FR-P1-32

```gherkin
Scenariusz: TEST 1 — zbiór OutboundOrderLine linii A jest pusty
  Zakładając, że `CustomerOrder.allowPartialShipment = false` i ma linie A oraz B
  Oraz linia B została zrealizowana cross-dockiem, a jej `OutboundOrderLine` ma `requiredQty` równą `Quantity` linii B i osiągnęła `PACKED`
  Oraz linia A ma `BACKORDERED`, nie ma żadnej `OutboundOrderLine` i ma `Quantity > 0`
  Gdy System WMS ocenia alternatywę (B) z `P1 R58` dla linii A
  Wtedy suma `requiredQty` pustego zbioru wynosi 0 i nie jest równa `Quantity` linii A
  Oraz Packing `TU` linii B pozostaje w `PACKING_SEALED` poza każdym `Shipment`
```

### TC-123 — Guard dla częściowego pokrycia linii PLANNED
*Poziom:* Proces · *Wymagania:* FR-P1-32

```gherkin
Scenariusz: TEST 2 — status PLANNED nie otwiera guardu przy częściowym pokryciu
  Zakładając, że `CustomerOrder.allowPartialShipment = false`, a `CustomerOrderLine.Quantity = 100`
  Oraz `crossDockEligibleQty = 30`
  Oraz jedna crossdockowa `OutboundOrderLine.requiredQty = 30` osiągnęła `PACKED`
  Oraz `CustomerOrderLine` ma `PLANNED`, a dla pozostałych 70 nie istnieje żadna `OutboundOrderLine`
  Gdy System WMS ocenia alternatywę (B) z `P1 R58`
  Wtedy suma `requiredQty = 30` nie jest równa `Quantity = 100`
  Oraz Packing `TU` pozostaje w `PACKING_SEALED` poza każdym `Shipment`
```

### TC-124 — Guard po korekcie i przy wielu liniach
*Poziom:* Proces · *Wymagania:* FR-P1-32

```gherkin
Scenariusz: TEST 3 — korekta STANDARD otwiera guard
  Zakładając, że `CustomerOrderLine.Quantity = 100`, a jedna `OutboundOrderLine.requiredQty = 100`
  Oraz linia osiągnęła `SHORT_PICKED` z `pickedQty = 30`
  Gdy `Warehouse Supervisor` zgodnie z `P1 R46` koryguje `CustomerOrderLine.Quantity` i `OutboundOrderLine.requiredQty` do 30, a linia osiąga `PACKED`
  Wtedy suma `requiredQty = 30` jest równa `Quantity = 30`
  Oraz alternatywa (B) z `P1 R58` jest spełniona

Scenariusz: TEST 3b — korekta CROSSDOCK otwiera guard
  Zakładając, że crossdockowa linia była planowana na 100, a `confirmedQty = 30`
  Gdy `Warehouse Supervisor` zgodnie z `P2 R15` koryguje `CustomerOrderLine.Quantity` i `OutboundOrderLine.requiredQty` do 30, a linia osiąga `PACKED`
  Wtedy suma `requiredQty = 30` jest równa `Quantity = 30`
  Oraz alternatywa (B) z `P1 R58` jest spełniona

Scenariusz: TEST 3c — dwie OutboundOrderLine razem pokrywają linię
  Zakładając, że `CustomerOrderLine.Quantity = 100`
  Oraz dwie nieanulowane `OutboundOrderLine` w `PACKED` mają `requiredQty` odpowiednio 30 i 70
  Gdy System WMS ocenia alternatywę (B) z `P1 R58`
  Wtedy suma `requiredQty = 30 + 70 = 100` jest równa `Quantity = 100`
  Oraz alternatywa (B) jest spełniona
```

### TC-125 — Jedno Shipment dla dwóch kanałów
*Poziom:* Proces · *Wymagania:* FR-P1-31/FR-P1-32

```gherkin
Scenariusz: TEST 4 — kompletne linie z dwóch kanałów wchodzą do jednego Shipment
  Zakładając, że `CustomerOrder.allowPartialShipment = false` ma jedną linię zrealizowaną kanałem `STANDARD` i drugą kanałem `CROSSDOCK`
  Oraz dla obu linii suma `requiredQty` nieanulowanych `OutboundOrderLine` w `PACKED` jest równa ich `Quantity`
  Oraz oba kanały mają identyczny `slaDeadline`
  Gdy System WMS ocenia `P1 R57` i `P1 R58`
  Wtedy obie Packing `TU` wchodzą do jednego kompletnego `Shipment`
```

### TC-126 — slaDeadline nie omija guardu CustomerOrder
*Poziom:* Proces · *Wymagania:* FR-P1-32

```gherkin
Scenariusz: TEST 5 — termin nie wydaje zamówienia w częściach
  Zakładając, że `CustomerOrder.allowPartialShipment = false` ma Packing `TU` w `PACKING_SEALED`
  Oraz co najmniej jedna aktywna `CustomerOrderLine` nie spełnia alternatywy (B) z `P1 R58`
  Gdy upływa wspólny `slaDeadline`
  Wtedy Packing `TU` pozostaje poza każdym `Shipment`
  Oraz automatyczne zamknięcie istniejącego `Shipment` zgodnie z `P1 R28` nie może jej objąć
  Oraz zamówienie nie opuszcza magazynu w części
```

### TC-128 — Pierwszy potwierdzony manifest nie kończy OutboundOrder
*Poziom:* Proces · *Wymagania:* FR-P1-44 · *Dane:* 1 OutboundOrder, 2 Packing TU, 2 Shipment

```gherkin
Scenariusz: Zlecenie z TU w dwóch Shipment zostaje w DISPATCHED
  Zakładając, że `Shipment` A zamknięto automatycznie przez `slaDeadline` z pierwszą Packing `TU` zlecenia (`P1 R28`)
  Oraz druga Packing `TU` tego samego `OutboundOrder` utworzyła `Shipment` B (`P1 R29`)
  Oraz zlecenie osiągnęło `READY_FOR_DISPATCH`, a następnie `DISPATCHED`
  Gdy `CarrierManifest` obejmujący `Shipment` A osiągnie `CONFIRMED`
  Wtedy `OutboundOrder` pozostaje w `DISPATCHED`
  Oraz `OutboundOrderLine` wydane `Shipment` A przechodzą `PACKED → SHIPPED`, a linie ze `Shipment` B nie są ruszane
  Oraz `CustomerOrder` nie osiąga `CLOSED`
```

### TC-129 — Ostatni potwierdzony manifest kończy OutboundOrder
*Poziom:* Proces · *Wymagania:* FR-P1-44 · *Dane:* jak TC-128, oba manifesty potwierdzone

```gherkin
Scenariusz: Potwierdzenie ostatniego Shipment domyka zlecenie
  Zakładając, że stan po `TC-128`: `Shipment` A potwierdzony, `Shipment` B niepotwierdzony
  Gdy `CarrierManifest` obejmujący `Shipment` B osiągnie `CONFIRMED`
  Wtedy `OutboundOrder` przechodzi `DISPATCHED → COMPLETED`

Scenariusz: Odwrotna kolejność potwierdzania daje ten sam wynik
  Zakładając, że ten sam `OutboundOrder` ma Packing `TU` w `Shipment` A i B
  Gdy najpierw `CONFIRMED` osiąga manifest ze `Shipment` B, a dopiero potem manifest ze `Shipment` A
  Wtedy `OutboundOrder` pozostaje w `DISPATCHED` po pierwszym potwierdzeniu
  Oraz przechodzi `DISPATCHED → COMPLETED` dopiero po drugim
```

### TC-130 — Alokacja SHORT blokuje wyłącznie ilość faktycznie zarezerwowaną
*Poziom:* Proces · *Wymagania:* FR-P1-45 · *Dane:* `requiredQty` 100, dostępne 30

```gherkin
Scenariusz: Rezerwacja częściowa nie zajmuje pełnej ilości wymaganej
  Zakładając, że `OutboundOrderLine.requiredQty` wynosi 100, a zapas `AVAILABLE` wynosi 30
  Gdy System WMS wykona `Allocation PENDING → SHORT`
  Wtedy `Allocation.reservedQty` wynosi 30
  Oraz ilość zapasu zajęta przez tę alokację wynosi 30
  Oraz po uzupełnieniu zapasu i przejściu `SHORT → RESERVED` `reservedQty` wynosi 100
```

### TC-131 — RELEASED i CONSUMED nie blokują zapasu
*Poziom:* Proces · *Wymagania:* FR-P1-45 · *Dane:* jedna `Allocation` w cyklu pełnym

```gherkin
Scenariusz: Zwolnienie i wydanie zerują ilość zajętą
  Zakładając, że `Allocation` jest w `CONFIRMED` z `reservedQty` równym 100
  Gdy alokacja przejdzie do `CONSUMED` przy wydaniu albo do `RELEASED` przy anulowaniu
  Wtedy `reservedQty` wraca do 0
  Oraz alokacja nie wnosi nic do ilości zapasu uznanej za zajętą
  Oraz alokacja w `PENDING` również nie wnosi nic
```

### TC-132 — Pierwsze wydanie nie terminalizuje linii ani Allocation
*Poziom:* Proces · *Wymagania:* FR-P1-46 · *Dane:* jedna `OutboundOrderLine` 100, dwie Outbound `TU` po 50, dwa `Shipment`

```gherkin
Scenariusz: Podział linii na dwa Shipment — pierwszy manifest
  Zakładając, że `OutboundOrderLine.requiredQty` wynosi 100, a `Allocation.reservedQty` wynosi 100
  Oraz TU-A z 50 sztukami należy do `Shipment` A, a TU-B z 50 sztukami do `Shipment` B
  Gdy manifest `Shipment` A osiągnie `CONFIRMED`, a `Shipment` B nie jest jeszcze potwierdzony
  Wtedy `Inventory` rozlicza 50 sztuk jako `SHIPPED`
  Oraz `Allocation.reservedQty` wynosi 50, a `Allocation` pozostaje w `CONFIRMED`
  Oraz `OutboundOrderLine` pozostaje w `PACKED`
  Oraz `OutboundOrder` pozostaje w `DISPATCHED` zgodnie z `P1 R70`
```

### TC-133 — Ostatnie wydanie domyka linię, Allocation i zlecenie
*Poziom:* Proces · *Wymagania:* FR-P1-46 · *Dane:* stan po TC-132

```gherkin
Scenariusz: Drugi manifest domyka rozliczenie
  Zakładając, że stan po `TC-132`: `Shipment` A potwierdzony, `Shipment` B niepotwierdzony
  Gdy manifest `Shipment` B osiągnie `CONFIRMED`
  Wtedy `Inventory` rozlicza pozostałe 50 sztuk jako `SHIPPED`
  Oraz `Allocation.reservedQty` osiąga 0, a `Allocation` przechodzi `CONFIRMED → CONSUMED`
  Oraz `OutboundOrderLine` przechodzi `PACKED → SHIPPED`
  Oraz jeżeli był to ostatni niepotwierdzony `Shipment` zlecenia, `OutboundOrder` przechodzi `DISPATCHED → COMPLETED` zgodnie z `P1 R70`

Scenariusz: Odwrotna kolejność potwierdzania daje ten sam wynik
  Zakładając, że ta sama linia ma TU-A w `Shipment` A i TU-B w `Shipment` B
  Gdy najpierw `CONFIRMED` osiąga manifest `Shipment` B, a dopiero potem manifest `Shipment` A
  Wtedy po pierwszym potwierdzeniu linia pozostaje w `PACKED`, a `Allocation` w `CONFIRMED`
  Oraz oba przejścia terminalne następują dopiero po drugim potwierdzeniu
```

### TC-135 — Pokrycie mieszane otwiera bramkę KROKU 2A
*Poziom:* Proces · *Wymagania:* FR-P1-03 · *Dane:* `allowPartialShipment = false`, linia 100, cross-dock pokrywa 40

```gherkin
Scenariusz: Linia pokryta cross-dockiem nie blokuje planowania
  Zakładając, że `CustomerOrder.allowPartialShipment = false`, a jego jedyna `CustomerOrderLine` ma `Quantity` 100
  Oraz crossdockowa `OutboundOrderLine` o `requiredQty` 40 osiągnęła `PACKED`, a `ATPReservation` linii wynosi 60
  Gdy uruchamia się cykliczne planowanie i System WMS ocenia bramkę `KROKU 2A`
  Wtedy linia jest uznana za w pełni pokrytą, bo 60 plus 40 równa się 100
  Oraz powstaje standardowy `OutboundOrder` wyłącznie na ilość niepokrytą, czyli 60

Scenariusz: Pełne pokrycie cross-dockiem nie tworzy standardowego zlecenia
  Zakładając, że crossdockowa `OutboundOrderLine` o `requiredQty` 100 osiągnęła `PACKED`, a `ATPReservation` wynosi 0
  Gdy System WMS ocenia bramkę `KROKU 2A`
  Wtedy linia jest w pełni pokryta
  Oraz standardowy `OutboundOrder` nie powstaje wcale

Scenariusz: Brak pokrycia nadal zatrzymuje zamówienie
  Zakładając, że `ATPReservation` wynosi 60, a żadna crossdockowa `OutboundOrderLine` nie osiągnęła `PACKED`
  Gdy System WMS ocenia bramkę `KROKU 2A`
  Wtedy zamówienie zostaje w `ACCEPTED` z `WARNING`
  Oraz standardowy `OutboundOrder` nie powstaje
```

## 2. P2 — Outbound Crossdock

### TC-020 — Cross-dock 1:1, pełny przebieg
*Poziom:* E2E · *Wymagania:* FR-P2-01/02/03/04
```gherkin
Scenariusz: Pełne dopasowanie jeden do jednego
  Zakładając, że cała zadeklarowana zawartość jednej źródłowej Inbound TU pokrywa zapotrzebowanie tego samego klienta i adresu, ze zgodnym priority i identycznym slaDeadline
  Gdy TU osiągnie IN_CROSS_DOCK i Packer potwierdzi rozsortowanie całej zadeklarowanej zawartości
  Wtedy powstaje jeden CrossDockPickTask, jedna docelowa Outbound TU i jeden Shipment
  Oraz OutboundOrder ma fulfillmentChannel CROSSDOCK, a Allocation nie powstaje
```

### TC-021 — Rozsortowanie n:n
*Poziom:* E2E · *Wymagania:* FR-P2-02/03/12
```gherkin
Scenariusz: Kilka źródłowych TU zasila kilka linii
  Zakładając, że dwie źródłowe Inbound TU zawierają SKU dla trzech zgodnych linii
  Gdy Packer rozsortuje całą zadeklarowaną zawartość obu TU
  Wtedy każda linia otrzymuje potwierdzoną ilość, a obie źródłowe TU osiągają CROSS_DOCKED
  Oraz kryterium CROSS_DOCKED zależy od braku ilości rezydualnej, nie od liczby docelowych TU
```

### TC-022 — Pełna docelowa TU w trakcie aktywnego zadania
*Poziom:* Proces · *Wymagania:* FR-P2-05
```gherkin
Scenariusz: Zapełniona docelowa TU nie kończy zadania
  Zakładając, że CrossDockPickTask ma plannedQty 80, a docelowa Outbound TU mieści 50
  Gdy Packer zamknie pełną TU skanem RF
  Wtedy ta TU przechodzi w PACKING_SEALED, a zadanie kontynuuje rozsortowanie pozostałych 30 do nowej TU
  Oraz ogólne anulowanie zgłoszone w tym czasie jest odrzucone
```

### TC-023 — Niedobór bez zgody na część
*Poziom:* Proces · *Wymagania:* FR-P2-05/06/07/08
```gherkin
Scenariusz: Eskalacja i wynik czekamy
  Zakładając, że allowPartialShipment jest false, a Packer potwierdził ilość mniejszą od planowanej
  Gdy Warehouse Supervisor wybierze wynik czekamy
  Wtedy OutboundOrderLine przechodzi w CANCELLED z PICKING, a CustomerOrderLine w BACKORDERED
  Oraz dla confirmedQty większego od zera powstaje PutBackTask
```

### TC-024 — Niedobór ze zgodą na część
*Poziom:* Proces · *Wymagania:* FR-P2-06/07
```gherkin
Scenariusz: Część ilości przechodzi dalej automatycznie
  Zakładając, że allowPartialShipment jest true, a potwierdzona ilość jest mniejsza od planowanej
  Gdy zadanie zostanie zakończone
  Wtedy rozsortowana część OutboundOrderLine przechodzi w PACKED bez eskalacji
  Oraz brakująca ilość wraca do BACKORDERED
```

### TC-025 — DAMAGED, SKU nieoczekiwane i damagedQty
*Poziom:* Proces · *Wymagania:* FR-P2-06/07/09/11
```gherkin
Scenariusz: Towar uszkodzony nie wraca do źródłowej TU
  Zakładając, że w trakcie pobrania wykryto ilość DAMAGED oraz SKU spoza deklaracji ASN
  Gdy Packer potwierdzi obie rozbieżności
  Wtedy oba towary trafiają fizycznie na QC i nie zasilają docelowej Outbound TU
  Oraz do Inbound wraca wyłącznie liczba damagedQty, bez SKU nieoczekiwanego i bez fizycznego zwrotu towaru
```

### TC-026 — Pusta źródłowa TU przed pobraniem
*Poziom:* Proces · *Wymagania:* FR-P2-09
```gherkin
Scenariusz: Brak zawartości przed rozpoczęciem pobrania
  Zakładając, że zadanie jest ASSIGNED, a OutboundOrderLine jest CREATED
  Gdy okaże się, że źródłowa Inbound TU jest pusta
  Wtedy zadanie i linia przechodzą w CANCELLED, demand wraca do BACKORDERED, a Inbound TU otrzymuje LOST
  Oraz przypadek ten nie obejmuje TU prawidłowo opróżnionej po rozsortowaniu, która osiąga CROSS_DOCKED
```

### TC-027 — Wiele zadań na jednej palecie i ilość resztowa
*Poziom:* Proces · *Wymagania:* FR-P2-10/11/12/15
```gherkin
Scenariusz: Rozliczenie sześćdziesiąt na czterdzieści przy trzech zadaniach
  Zakładając, że źródłowa TU ma deklarację ASN 100 i trzy zadania o plannedQty 30, 20 i 10
  Gdy Packer kolejno zgłosi zakończenie każdego z nich
  Wtedy każde zadanie osiąga COMPLETED z osobna, a źródłowa TU pozostaje w cross-dockingu aż do zakończenia ostatniego
  Oraz po zakończeniu wszystkich rozliczenie crossdockowe obejmuje 60, resztka 40 trafia na TRANSIT, TU przechodzi w IN_PUTAWAY i nie otrzymuje już kolejnego zadania cross-dockowego

Scenariusz: Uszkodzona ilość nie wchodzi do resztki oczekiwanej w TU
  Zakładając, że źródłowa TU ma deklarację ASN 100, a zakończone zadania dały łącznie confirmedQty 50 i damagedQty 10
  Gdy System WMS finalizuje źródłową TU i przekazuje wynik do Inbound
  Wtedy ilość rezydualna wynosi 40, a nie 50 — ilość uszkodzona opuściła TU na QC i nie jest w niej oczekiwana
  Oraz Inbound otrzymuje confirmedQty 50 i damagedQty 10 jako osobne składniki
```

### TC-028 — Bramka Goods Receipt i odrzucenie
*Poziom:* Integracja · *Wymagania:* FR-P2-13/14
```gherkin
Scenariusz: Shipment czeka na rozliczenie crossdockowe, odrzucone GR nie blokuje trwale
  Zakładając, że Shipment zależy od dwóch źródłowych Inbound TU
  Gdy pierwsza otrzyma GR_ACCEPTED rozliczenia crossdockowego, a druga jeszcze nie
  Wtedy Shipment nie przechodzi do POSTING_PENDING i pozostaje w LABEL_GENERATED albo OWN_TRANSPORT
  Oraz jawny GR_REJECTED drugiej TU nie przenosi Shipment w POSTING_ERROR, a późniejszy GR_ACCEPTED tej samej TU spełnia bramkę bez odrębnej ścieżki odzyskania
```

### TC-029 — Ilość kwalifikowalna przy trwających zadaniach
*Poziom:* Proces · *Wymagania:* FR-P2-03/16
```gherkin
Scenariusz: Uszkodzona ilość nie wraca do puli kwalifikowalnej
  Zakładając, że deklaracja ASN wynosi 100, aktywne zadania mają plannedQty 30, a zakończone 40 confirmedQty i 10 damagedQty
  Gdy System WMS wylicza sourceEligibleQty dla nowego zadania
  Wtedy wynik wynosi 20
  Oraz ta sama fizyczna ilość nie może być planned ani confirmed w dwóch aktywnych zadaniach
```

### TC-030 — Jedna paleta źródłowa, dwa Shipmenty
*Poziom:* Integracja · *Wymagania:* FR-P2-02/05/16/17/18
```gherkin
Scenariusz: Jeden wynik GR aktualizuje zadania obu wysyłek
  Zakładając, że źródłowa TU X zasiliła zadanie A wysyłki 1 i zadanie B wysyłki 2
  Gdy TU X otrzyma jeden GR_ACCEPTED rozliczenia crossdockowego
  Wtedy grAcceptanceStatus obu zadań zostaje ustawiony, bez osobnego sygnału per Shipment
  Oraz każda wysyłka sprawdza własny zbiór źródłowych TU osobno, a docelowe Outbound TU zadań A i B są od siebie niezależne — każda należy do dokładnie jednego Shipmentu
```

### TC-031 — Rozliczenie putawayowe po crossdockowym
*Poziom:* Integracja · *Wymagania:* FR-P2-13/17
```gherkin
Scenariusz: Sygnał ze źródła PUTAWAY nie rusza bramki crossdockowej
  Zakładając, że zadania źródłowej TU mają GR_ACCEPTED ze źródła CROSSDOCK, a Shipment został już zgłoszony
  Gdy dla tej samej TU dotrze wynik GR ze źródłem PUTAWAY, akceptujący albo odrzucający
  Wtedy grAcceptanceStatus zadań crossdockowych pozostaje bez zmiany
  Oraz gotowość Shipment nie zostaje cofnięta
```

### TC-032 — Numeracja przy 1:1 z poprawnym GS1
*Poziom:* Proces · *Wymagania:* FR-P2-04
```gherkin
Scenariusz: Dziedziczenie numeru i SSCC
  Zakładając, że TU_NUMBER źródłowej Inbound TU jest poprawnym SSCC GS1 i zachodzi pełne dopasowanie 1:1
  Gdy powstaje docelowa Outbound TU
  Wtedy dziedziczy TU_NUMBER i SSCC
  Oraz nie jest drukowana nowa etykieta
```

### TC-033 — Numeracja przy 1:1 bez poprawnego GS1
*Poziom:* Proces · *Wymagania:* FR-P2-04
```gherkin
Scenariusz: Przenumerowanie i nowa etykieta
  Zakładając, że TU_NUMBER źródłowej Inbound TU nie jest poprawnym SSCC GS1, mimo pełnego dopasowania 1:1
  Gdy powstaje docelowa Outbound TU
  Wtedy otrzymuje nowy numer zgodnie z TUSetup i Sequence
  Oraz zostaje wydrukowana nowa etykieta
```

### TC-034 — Numeracja przy rozsortowaniu n:n
*Poziom:* Proces · *Wymagania:* FR-P2-04
```gherkin
Scenariusz: Nowy numer niezależnie od numeru źródłowego
  Zakładając, że zachodzi rozsortowanie n:n, a jedna ze źródłowych TU ma poprawny SSCC GS1
  Gdy powstaje docelowa Outbound TU
  Wtedy otrzymuje nowy numer z Sequence wskazanej przez jej TUSetup
  Oraz żaden numer źródłowy nie jest dziedziczony
```

### TC-035 — Pusta docelowa TU po pełnym odzysku
*Poziom:* Proces · *Wymagania:* FR-P2-10/19
```gherkin
Scenariusz: Odzysk całej ilości anuluje docelową TU
  Zakładając, że docelowa Outbound TU zawiera wyłącznie ilość, która utraciła demand
  Gdy PutBackTask odzyska całą tę ilość do Inventory AVAILABLE
  Oraz żadne aktywne ani zaplanowane zadanie nie wskazuje tej TU jako celu
  Wtedy docelowa TU przechodzi z CREATED w CANCELLED, nie czekając na slaDeadline
  Oraz nie powstaje krok zwolnienia Allocation ani wejście w PACKING_SEALED

Scenariusz: Drugie zadanie wskazujące tę samą TU wstrzymuje anulowanie
  Zakładając, że zadanie A odłożyło ilość do docelowej TU X, a zadanie B jest aktywne i również wskazuje TU X jako cel
  Gdy PutBackTask odzyska całą ilość odłożoną przez zadanie A i TU X pozostaje pusta
  Wtedy TU X pozostaje w CREATED i nie przechodzi w CANCELLED
  Oraz zadanie B zachowuje ważne wskazanie celu
```

### TC-036 — demandEligibleQty jako czynnik ograniczający
*Poziom:* Proces · *Wymagania:* FR-P2-03
```gherkin
Scenariusz: demandEligibleQty ogranicza kwalifikowalną ilość
  Zakładając, że sourceEligibleQty źródłowej TU wynosi 50, a Quantity CustomerOrderLine pomniejszone o ATPReservation i o sumę requiredQty jej niezanulowanych OutboundOrderLine wynosi 20
  Gdy System WMS wylicza crossDockEligibleQty
  Wtedy wynik wynosi 20, mniejszy niż sourceEligibleQty
  Oraz fulfillmentChannel OutboundOrder pozostaje CROSSDOCK
```

### TC-134 — Rezerwacja twarda nie znika ze wzoru demandEligibleQty
*Poziom:* Proces · *Wymagania:* FR-P2-03 · *Dane:* `Quantity` 100, jedna standardowa `OutboundOrderLine` 30, `ATPReservation` 0

```gherkin
Scenariusz: Ilość przeniesiona z ATPReservation do zlecenia nadal pomniejsza pulę crossdockową
  Zakładając, że `CustomerOrderLine.Quantity` wynosi 100
  Oraz planowanie utworzyło standardową `OutboundOrderLine` z `requiredQty` 30, a `P1 R11` zmniejszyło `ATPReservation` do 0
  Oraz brakujące 70 pozostaje `BACKORDERED`
  Gdy źródłowa Inbound `TU` osiąga `IN_CROSS_DOCK` i System WMS wylicza `demandEligibleQty` tej linii
  Wtedy wynik wynosi 70, nie 100
  Oraz suma ilości przyznanych obu kanałom nie przekracza `Quantity`

Scenariusz: Anulowana OutboundOrderLine nie pomniejsza puli
  Zakładając, że ta sama standardowa `OutboundOrderLine` została `CANCELLED`, a jej ilość wróciła do `ATPReservation`
  Gdy System WMS ponownie wylicza `demandEligibleQty`
  Wtedy anulowana linia nie jest odejmowana
  Oraz wynik nie zmienia się przez podwójne odjęcie tej samej ilości
```

### TC-037 — Automatyczne zamknięcie po slaDeadline
*Poziom:* Proces · *Wymagania:* FR-P2-05
```gherkin
Scenariusz: Automatyczne zamknięcie po slaDeadline i dołączenie spóźnionej dostawy przed terminem
  Zakładając, że docelowa Outbound TU nie ma już aktywnych ani zaplanowanych CrossDockPickTask, a jej slaDeadline jeszcze nie nadszedł
  Gdy nadejdzie kolejna Inbound TU tego samego dnia dla zgodnego klienta/adresu/priority/slaDeadline, zanim slaDeadline minie
  Wtedy nowe zadanie może nadal zasilić tę samą otwartą TU, która nie zamyka się przedwcześnie
  Oraz gdy slaDeadline zostanie osiągnięty bez kolejnych aktywnych/zaplanowanych zadań, System WMS automatycznie zamyka TU w PACKING_SEALED
```

### TC-038 — Odzyskanie bramki z POSTING_ERROR niezwiązanego z GR
*Poziom:* Integracja · *Wymagania:* FR-P2-14
```gherkin
Scenariusz: Bramka odzyskuje Shipment z POSTING_ERROR spowodowanego inną przyczyną
  Zakładając, że Shipment jest w POSTING_ERROR z powodu odrzucenia przez ERP niezwiązanego z bramką GR, a jedna źródłowa TU nadal nie ma GR_ACCEPTED
  Gdy ta źródłowa TU otrzyma GR_ACCEPTED
  Wtedy bramka jest ponownie ewaluowana niezależnie od przyczyny POSTING_ERROR
  Oraz Warehouse Supervisor mógł przez cały czas odczytać grAcceptanceStatus tej TU
```

### TC-039 — Ogólne kryterium anulowania OutboundOrder poza ścieżką pustej TU
*Poziom:* Proces · *Wymagania:* FR-P2-08/09
```gherkin
Scenariusz: OutboundOrder anulowany przez wynik czekamy, nie przez pustą TU
  Zakładając, że jedyna linia OutboundOrder osiąga CANCELLED przez decyzję czekamy przy niedoborze, bez przejścia przez pustą źródłową TU
  Gdy to jest jedyna linia tego OutboundOrder i żadna linia nie osiągnęła PACKED
  Wtedy OutboundOrder przechodzi w CANCELLED
  Oraz kryterium zamknięcia nagłówka jest ogólne (P2 R37), niezależne od ścieżki prowadzącej do anulowania linii
```

## 3. P3 — Reservation Release

### TC-040 — Zwolnienie przed pobraniem
*Poziom:* Proces · *Wymagania:* FR-P3-01/02/03
```gherkin
Scenariusz: Anulowanie zwalnia rezerwację
  Zakładając, że pickedQty = 0
  Gdy linia zostanie anulowana
  Wtedy Allocation i ATPReservation są zwolnione
  Oraz PutBackTask nie powstaje
```

### TC-041 — Automatyczne zwolnienie
*Poziom:* Proces · *Wymagania:* FR-P3-01/02/03
```gherkin
Scenariusz: Wygaśnięcie uruchamia P3
  Zakładając, że spełniono warunek automatycznego zwolnienia
  Gdy WMS rozlicza Allocation
  Wtedy rezerwacja zostaje zwolniona, a stan linii zaktualizowany
```

### TC-042 — Okno wyścigu
*Poziom:* Współbieżność · *Wymagania:* FR-P3-04
```gherkin
Scenariusz: Pobranie fizyczne bez potwierdzenia
  Zakładając, że towar pobrano, lecz nie potwierdzono PickTask
  Gdy równolegle nastąpi zwolnienie
  Wtedy WMS anuluje PickTask i nakazuje zwrot na źródło bez PutBackTask
```

### TC-043 — Potwierdzone pobranie przechodzi do P4
*Poziom:* Proces · *Wymagania:* FR-P3-04
```gherkin
Scenariusz: P3 nie zwraca formalnie pobranej ilości
  Zakładając, że pickedQty > 0
  Gdy linia zostaje anulowana
  Wtedy przypadek zostaje przekazany do P4
```

### TC-112 — Wariant automatycznego zwolnienia
*Poziom:* Proces · *Wymagania:* FR-P3-05

```gherkin
Scenariusz: Wariant automatycznego zwolnienia
  Zakładając, że magazyn skonfigurował automatyczne zwolnienie po czasie
  Gdy czas utrzymania rezerwacji upływa
  Wtedy System WMS zwalnia Allocation bez udziału Warehouse Supervisor
```

### TC-113 — Priorytet nie skraca ani nie wydłuża czasu
*Poziom:* Proces · *Wymagania:* FR-P3-06

```gherkin
Scenariusz: Priorytet nie skraca ani nie wydłuża czasu
  Zakładając, że dwa CustomerOrder mają różny priority i różny slaDeadline
  Gdy oba mają częściową rezerwację przy tej samej polityce magazynu
  Wtedy czas utrzymania rezerwacji jest dla obu identyczny
```

## 4. P4 — Physical Put-back

### TC-050 — Anulowanie PACKED przed granicą
*Poziom:* E2E · *Wymagania:* FR-P4-01/02/03/04
```gherkin
Scenariusz: Supervisor zatwierdza put-back
  Zakładając, że linia jest PACKED, a Shipment nie ma POSTING_PENDING
  Gdy Supervisor zatwierdzi anulowanie
  Wtedy skutki logiczne są natychmiastowe, a PutBackTask prowadzi Inventory do AVAILABLE
```

### TC-051 — PutBackTask happy path
*Poziom:* Proces · *Wymagania:* FR-P4-02/03/04
```gherkin
Scenariusz: Towar wraca do poprawnej lokalizacji
  Zakładając, że istnieje PutBackTask dla pickedQty > 0
  Gdy operator potwierdzi poprawną lokalizację i odłożenie
  Wtedy PutBackTask ma COMPLETED, a Inventory ma AVAILABLE
```

### TC-052 — Pętla walidacji lokalizacji
*Poziom:* Proces · *Wymagania:* FR-P4-03/04
```gherkin
Scenariusz: Druga lokalizacja jest poprawna
  Zakładając, że pierwszą lokalizację odrzucono
  Gdy operator poda poprawną kolejną lokalizację
  Wtedy zadanie kończy się bez eskalacji i limitu prób
```

### TC-053 — Zerowe pickedQty
*Poziom:* Proces · *Wymagania:* FR-P4-02
```gherkin
Scenariusz: Brak fizycznego zadania
  Zakładając, że pickedQty = 0
  Gdy anulowanie zostanie rozliczone
  Wtedy PutBackTask nie powstaje
```

## 5. P5 — Wyjątki przekrojowe

### TC-060 — SHORT_ALLOCATED
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-01/02
```gherkin
Scenariusz: Flaga częściowej realizacji wybiera wynik
  Zakładając, że Allocation nie pokrywa pełnej ilości
  Gdy allowPartial ma kolejno false i true
  Wtedy WMS odpowiednio zwalnia całość albo zachowuje przydzieloną część
```

### TC-061 — SHORT_PICKED: automatyczne ponowienie i eskalacja
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-03
```gherkin
Scenariusz: Limit wyznacza przejście do Supervisora
  Zakładając, że pierwszy PickTask ma SHORT_PICKED
  Gdy dostępna jest inna lokalizacja ATP, a następnie limit ponowień zostaje wyczerpany
  Wtedy WMS najpierw tworzy nowy PickTask, a później eskaluje nierozliczony brak
```

### TC-062 — Rozstrzygnięcie SHORT_PICKED
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-04/05/06
```gherkin
Scenariusz: Każda decyzja ma odrębny skutek
  Zakładając, że SHORT_PICKED wymaga decyzji Supervisora
  Gdy wybierze kolejno „czekamy”, anulowanie albo trwałą zgodę na część
  Wtedy WMS odpowiednio cofa niepakowane linie, wymaga edycji ilości albo realizuje część z podanym powodem
```

### TC-063 — Niedobór lub uszkodzenie przy pakowaniu
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-07
```gherkin
Scenariusz: Pakowanie używa mechanizmu SHORT_PICKED
  Zakładając, że operator wykrył brak lub uszkodzenie podczas repack by SKU
  Gdy potwierdzi rozbieżność
  Wtedy WMS uruchamia obsługę SHORT_PICKED bez blokady lokalizacji źródłowej
```

### TC-064 — ON_HOLD i wznowienie
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-08
```gherkin
Scenariusz: Blokada zatrzymuje wykonanie
  Zakładając, że obiekt ma ON_HOLD
  Gdy blokada zostanie zdjęta po wcześniejszej próbie wykonania
  Wtedy proces wraca do właściwego miejsca ścieżki głównej
```

### TC-065 — Anulowanie ogólne
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-09
```gherkin
Scenariusz: Blokująca linia odrzuca anulowanie całości
  Zakładając, że jedna linia CustomerOrder nie spełnia warunku anulowania
  Gdy OMS zażąda anulowania całego zamówienia
  Wtedy WMS odrzuca całość i wskazuje blokującą linię
```

### TC-066 — Carrier Selection i ograniczenie etykiety
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-10/11
```gherkin
Scenariusz: Ręczny wybór bez osobnego trybu awarii etykiety
  Zakładając, że Carrier Selection nie zwrócił wyniku dla `TU` na typie o `processUsage = EXTERNAL`, którego `TUSetup.maxVolume` nie ma wartości
  Gdy Dispatcher wybierze Carrier, a Supervisor zatwierdzi wybór
  Wtedy etykieta może powstać, a wersja 1 nie uruchamia elektronicznego trybu odrzucenia
```

### TC-067 — Wyjątki cross-dock
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-12/13/14/15/16
```gherkin
Scenariusz: Pięć wyjątków ma jawny wynik
  Zakładając, że cross-dock napotyka niedobór, puste TU, anulowanie w toku albo GR_REJECTED
  Gdy WMS rozliczy właściwy warunek
  Wtedy wybiera eskalację lub część, anuluje puste TU, odrzuca anulowanie w toku albo pozostawia Shipment poza POSTING_PENDING do czasu GR_ACCEPTED
```

### TC-127 — Skompletowany fragment czeka na komplet CustomerOrder
*Poziom:* Przekrojowy · *Wymagania:* FR-P5-17
```gherkin
Scenariusz: P5 E17 nie jest niedoborem E12 ani E13
  Zakładając, że `CustomerOrder.allowPartialShipment = false`
  Oraz część `CustomerOrderLine` nie spełnia ani alternatywy (A), ani (B) z `P1 R58`
  Oraz inne linie tego samego `CustomerOrder` mają już `OutboundOrderLine` w `PACKED`
  Gdy System WMS ocenia możliwość utworzenia albo uzupełnienia `Shipment`
  Wtedy Packing `TU` linii spakowanych pozostają w `PACKING_SEALED` w strefie konsolidacji i poza każdym `Shipment`
  Oraz `OutboundOrder` nie jest anulowany, a linie pozostałe nie są automatycznie uwalniane do osobnej realizacji
  Oraz oczekiwanie kończy wyłącznie skompletowanie zamówienia albo ścieżka `Warehouse Supervisor` z `P1 R46` lub `P1 R47`
```

## 6. Integracje

### TC-080 — Granica Inbound–Outbound
*Poziom:* Integracja · *Wymagania:* INT-01
```gherkin
Scenariusz: Baza ilościowa pochodzi z deklaracji ASN
  Zakładając, że Inbound publikuje dane TU bez fizycznej weryfikacji jej zawartości
  Gdy status osiąga IN_CROSS_DOCK
  Wtedy Outbound otrzymuje SKU, ilość zadeklarowaną w ASN i korelację przyjęcia
  Oraz ta ilość jest bazą do wyliczenia sourceEligibleQty
```

### TC-081 — Wynik cross-dock wraca do Inbound
*Poziom:* Integracja · *Wymagania:* INT-02/03
```gherkin
Scenariusz: Kontrakt zwrotny obejmuje dwie ilości
  Zakładając, że rozliczenie crossdockowe źródłowej TU zostało zamknięte
  Gdy P2 publikuje wynik do Inbound
  Wtedy Inbound otrzymuje confirmedQty i damagedQty z korelacją po TU, bez osobnego pola ilości rezydualnej
  Oraz korelacja po sourceInboundTU trwa do nadejścia wyniku z ERP, a ponawianie komunikatu Goods Receipt należy do Inbound
```

### TC-082 — ERP akceptuje Shipment POST
*Poziom:* Integracja · *Wymagania:* INT-04/05
```gherkin
Scenariusz: Poprawne księgowanie
  Zakładając, że Shipment jest gotowy
  Gdy ERP zaakceptuje Shipment POST
  Wtedy Shipment przechodzi z POSTING_PENDING do POSTED
```

### TC-083 — ERP odrzuca i przyjmuje ponowienie
*Poziom:* Integracja · *Wymagania:* INT-05
```gherkin
Scenariusz: Bezpieczne ponowienie
  Zakładając, że ERP zwróciło błąd
  Gdy ten sam Shipment POST zostanie poprawnie ponowiony
  Wtedy stan przechodzi z POSTING_ERROR do POSTED bez podwójnego skutku
```

### TC-084 — Anulowanie z systemu zamówień
*Poziom:* Integracja · *Wymagania:* INT-06
```gherkin
Scenariusz: Korelacja żądania z linią
  Zakładając, że system zamówień wysyła anulowanie
  Gdy WMS odnajdzie linię i jej pickedQty
  Wtedy wybiera P3 albo P4 dla tej samej linii
```

## 7. Współbieżność

### TC-090 — Konkurencja o ATP
*Poziom:* Współbieżność · *Wymagania:* CON-01
```gherkin
Scenariusz: Brak podwójnej rezerwacji
  Zakładając, że dwa zlecenia konkurują o jedną ilość ATP
  Gdy alokacje wykonają się równolegle
  Wtedy suma ATPReservation nie przekracza zapasu ATP
```

### TC-091 — Realokacja po PickTask
*Poziom:* Współbieżność · *Wymagania:* CON-02
```gherkin
Scenariusz: Istniejący PickTask chroni przypisanie
  Zakładając, że PickTask został utworzony
  Gdy równoległy przebieg próbuje realokować ilość
  Wtedy WMS odrzuca zmianę przypisania
```

### TC-092 — Konkurencja o ilość cross-dock
*Poziom:* Współbieżność · *Wymagania:* CON-03
```gherkin
Scenariusz: Uszkodzona ilość nie jest ponownie planowana
  Zakładając, że deklaracja TU wynosi 100, w toku jest 30, a zakończone zadania dały 40 confirmedQty i 10 damagedQty
  Gdy dwa przebiegi równolegle próbują zaplanować kolejne zadanie
  Wtedy łącznie mogą objąć najwyżej 20
  Oraz żadna jednostka ilości nie trafia do dwóch zadań
```

### TC-093 — Termin graniczny Shipment
*Poziom:* Współbieżność · *Wymagania:* CON-04
```gherkin
Scenariusz: Paczka nie trafia do dwóch wysyłek
  Zakładając, że paczki zamykają się wokół terminu granicznego
  Gdy grupowanie wykonuje się równolegle
  Wtedy każda paczka należy do najwyżej jednego Shipment
```

### TC-094 — Duplikat ERP i manifestu
*Poziom:* Współbieżność · *Wymagania:* CON-05
```gherkin
Scenariusz: Powtórzony komunikat ma jeden skutek
  Zakładając, że odpowiedź ERP lub zamknięcie manifestu już rozliczono
  Gdy ten sam komunikat przyjdzie ponownie
  Wtedy stan końcowy nie cofa się i skutek nie jest dublowany
```

### TC-096 — Przydział PickTask wg strefy i kolejności
*Poziom:* Proces · *Wymagania:* FR-P1-29
```gherkin
Scenariusz: Operator bez zadania dostaje kolejny PickTask swojej strefy
  Zakładając, że operator jest zalogowany do modułu zbierania w strefie A i nie ma aktywnego zadania
  Gdy System WMS wybiera kolejne zadanie do przydziału
  Wtedy operator otrzymuje najwyższy w kolejce PickTask strefy A
```

### TC-097 — Operator z aktywnym zadaniem nie dostaje kolejnego
*Poziom:* Proces · *Wymagania:* FR-P1-29
```gherkin
Scenariusz: Operator z aktywnym zadaniem nie dostaje kolejnego
  Zakładając, że operator ma już aktywny PickTask
  Gdy w jego strefie oczekuje inny PickTask
  Wtedy System WMS nie przydziela mu kolejnego zadania
```

### TC-098 — Kontynuacja zamówienia w kolejnej strefie do otwartej TU
*Poziom:* Proces · *Wymagania:* FR-P1-30
```gherkin
Scenariusz: Operator kontynuuje zamówienie w innej strefie
  Zakładając, że operator zakończył PickTask strefy A dla otwartej Picking TU
  Gdy wybiera kontynuację tego samego OutboundOrder w strefie B
  Wtedy otrzymuje PickTask strefy B tego zamówienia poza kolejnością
  Oraz jego aktywna strefa zmienia się na B
```

### TC-099 — Przydział CrossDockPickTask przez wybór modułu
*Poziom:* Proces · *Wymagania:* FR-P2-21
```gherkin
Scenariusz: Operator bez zadania dostaje kolejny CrossDockPickTask
  Zakładając, że operator jest zalogowany do modułu crossdockingu i nie ma aktywnego zadania
  Gdy System WMS wybiera kolejne zadanie do przydziału
  Wtedy operator otrzymuje najwyższy w kolejce CrossDockPickTask
```

### TC-100 — Przydział PutBackTask w kolejności zgłoszenia
*Poziom:* Proces · *Wymagania:* FR-P4-05
```gherkin
Scenariusz: Operator bez zadania dostaje kolejny PutBackTask
  Zakładając, że operator jest zalogowany do modułu zwrotów i nie ma aktywnego zadania
  Gdy System WMS wybiera kolejne zadanie do przydziału
  Wtedy operator otrzymuje najstarszy zgłoszony PutBackTask
```

### TC-101 — Otwarcie nowej docelowej TU w trakcie aktywnego zadania
*Poziom:* Proces · *Wymagania:* FR-P2-22
```gherkin
Scenariusz: Zamknięcie albo anulowanie docelowej TU nie zostawia zadania bez celu
  Zakładając, że CrossDockPickTask jest aktywny, a jego docelowa Outbound TU została zamknięta jako fizycznie pełna albo anulowana jako pusta
  Gdy Packer odkłada kolejną pozycję SKU w ramach tego samego zadania
  Wtedy System WMS otwiera nową Outbound TU i nadaje jej TU_NUMBER
  Oraz zadanie kontynuuje rozsortowanie bez eskalacji do Warehouse Supervisor
```

### TC-102 — Do cross-dockingu trafia wyłącznie TU ELEMENTARY
*Poziom:* Proces · *Wymagania:* FR-P2-23
```gherkin
Scenariusz: Do cross-dockingu trafia dziecko, nie zbiorcza
  Zakładając, że SKU potrzebne do linii BACKORDERED jest zadeklarowane w TU ELEMENTARY przyczepionej do TU AGGREGATE
  Gdy Inbound kwalifikuje ten towar do cross-dockingu
  Wtedy do procesu Outbound Crossdock trafia TU ELEMENTARY
  Oraz TU AGGREGATE nie osiąga IN_CROSS_DOCK
```

### TC-103 — Zerowe dopasowanie przy IN_CROSS_DOCK kończy się Putaway
*Poziom:* Proces · *Wymagania:* FR-P2-24
```gherkin
Scenariusz: Popyt zniknął w czasie transportu do strefy cross-dock
  Zakładając, że Inbound zakwalifikował TU do cross-dockingu, a przed jej przejściem w IN_CROSS_DOCK wszystkie pasujące CustomerOrderLine przestały być BACKORDERED
  Gdy TU osiąga IN_CROSS_DOCK
  Wtedy System WMS nie tworzy żadnego CrossDockPickTask
  Oraz cała zadeklarowana ilość TU jest ilością rezydualną
  Oraz TU zostaje przekazana do Putaway z pełną ilością rezydualną
```

### TC-119 — Dziedziczenie i grupowanie crossdockowego OutboundOrder
*Poziom:* Proces · *Wymagania:* FR-P2-25

```gherkin
Scenariusz: Crossdockowy OutboundOrder dziedziczy parametry zamówienia
  Zakładając, że CustomerOrder ma priority 2, slaDeadline 2026-08-28T16:00:00 i allowPartialShipment = false
  Gdy System WMS tworzy crossdockowy OutboundOrder dla części jego linii
  Wtedy OutboundOrder otrzymuje priority 2 i slaDeadline 2026-08-28T16:00:00
  Oraz nie grupuje linii żadnego innego CustomerOrder
```
