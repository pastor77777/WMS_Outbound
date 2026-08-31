# WMSAI Outbound — Wymagania

**Projekt:** Realizacja zamówień wychodzących (Outbound)
**Data:** 28.08.2026 | **Autor:** Analityk Rozwiązania (kompetencje Architekta Rozwiązania)
**Źródło:** `proces_1_standard_fulfillment.md` v1.20, `proces_2_outbound_crossdock.md` v1.13, `proces_3_reservation_release.md` v1.2, `proces_4_physical_putback.md` v1.2, `model_stanow_outbound.md` v1.19. Wyjątki przekrojowe (`FR-P5-*`) pochodzą z sekcji „Wyjątki i ścieżki alternatywne" plików `proces_1` i `proces_2` — nie istnieje osobny plik procesu dla PROCESU 5.
**Zawartość:** 1a — funkcjonalne · 1b — integracyjne · 1c — niefunkcjonalne (poza B6) · 1d — współbieżność.

## Konwencja zapisu (EARS)

Wymagania w notacji **EARS** (Easy Approach to Requirements Syntax). Wzorce zdań:

- **Uniwersalne:** „System WMS musi <reakcja>."
- **Zdarzeniowe:** „**Gdy** <wyzwalacz>, System WMS musi <reakcja>."
- **Stanowe:** „**Dopóki** <stan>, System WMS musi <reakcja>."
- **Warunkowe (niepożądane):** „**Jeśli** <warunek>, to System WMS musi <reakcja>."
- **Konfiguracyjne:** „**Tam gdzie** <funkcja włączona>, System WMS musi <reakcja>."

Aktor występuje w wyzwalaczu. Każde wymaganie: **ID · Kategoria · Treść (EARS) · Źródło · Kryteria akceptacji (Given/When/Then)**. Kryteria akceptacji w składni Gherkin (PL): `Zakładając / Gdy / Wtedy / Oraz`.

---

# 1a. Wymagania funkcjonalne

## P1 — Standard Fulfillment

### FR-P1-01 — Zerowe ATP i kolejka rezerwacji
*Kategoria:* zdarzeniowe · *Źródło:* P1 R1–P1 R2
**Gdy** dostępność `ATP` dla zaakceptowanej `CustomerOrderLine` wynosi zero, System WMS musi ustawić `ATPReservation = 0`, nadać `BACKORDERED` i umieścić linię w kolejce zgodnej z parametrem magazynu.
```gherkin
Scenariusz: Brak ATP tworzy oczekiwanie
  Zakładając, że `CustomerOrderLine` jest zaakceptowana i nie ma dostępnego ATP
  Gdy System WMS nalicza miękką rezerwację
  Wtedy linia ma `ATPReservation = 0` i status `BACKORDERED`
  Oraz zajmuje miejsce w kolejce zgodnej z konfiguracją magazynu
```

### FR-P1-02 — Przeliczanie kolejki ATP
*Kategoria:* zdarzeniowe · *Źródło:* P1 R3–P1 R4
**Gdy** ERP potwierdzi zaksięgowanie Inbound `TU` dla SKU albo — w wariancie priorytetowym — pojawi się nowe `CustomerOrder ACCEPTED`, System WMS musi przeliczyć kolejkę `ATPReservation` bez naruszania `Allocation RESERVED`.
```gherkin
Scenariusz: Potwierdzenie ERP uruchamia przeliczenie
  Zakładając, że linie oczekują na ATP tego samego SKU
  Gdy ERP potwierdzi `POST` odpowiedniej Inbound `TU`
  Wtedy System WMS przelicza miękkie rezerwacje
  Oraz nie odbiera istniejącej `Allocation RESERVED`
```

### FR-P1-03 — Pomijanie ON_HOLD i pełna rezerwacja
*Kategoria:* stanowe · *Źródło:* P1 R5–P1 R6
**Dopóki** `CustomerOrder` ma `ON_HOLD` albo przy `allowPartialShipment = false` choć jedna jego `CustomerOrderLine` nie jest w pełni pokryta — gdzie pokrycie linii to `ATPReservation` powiększone o sumę `requiredQty` jej `OutboundOrderLine` w `PACKED` albo dalej, niezanulowanych, niezależnie od kanału — System WMS musi pominąć zamówienie w planowaniu i nie tworzyć standardowego `OutboundOrder`; przy pełnym pokryciu cross-dockiem standardowy `OutboundOrder` nie powstaje wcale, a przy pokryciu częściowym obejmuje wyłącznie ilość niepokrytą.
```gherkin
Scenariusz: Zamówienie nie wchodzi do planowania
  Zakładając, że zamówienie jest `ON_HOLD` albo choć jedna jego linia nie jest w pełni pokryta
  Gdy uruchamia się cykliczne planowanie
  Wtedy `OutboundOrder` nie powstaje
  Oraz zamówienie pozostaje `ACCEPTED` z `WARNING`, gdy przyczyną jest brak rezerwacji
```

### FR-P1-04 — Grupowanie zamówień
*Kategoria:* warunkowe · *Źródło:* P1 R7–P1 R8
**Jeśli** linie mają zostać zgrupowane w `OutboundOrder`, to System WMS musi wymagać zgodnego klienta, adresu, priorytetu i `slaDeadline` w tolerancji; dla `allowPartialShipment = false` musi utworzyć dokładnie jeden nieagregowany `OutboundOrder`.
```gherkin
Scenariusz: Niezgodny klient blokuje grupowanie
  Zakładając, że dwie linie należą do różnych klientów
  Gdy System WMS planuje `OutboundOrder`
  Wtedy linie nie są grupowane razem
  Oraz zamówienie bez częściowych wysyłek nie jest agregowane z innym zamówieniem
```

### FR-P1-05 — Atrybuty OutboundOrder i brak realokacji
*Kategoria:* zdarzeniowe · *Źródło:* P1 R9–P1 R10
**Gdy** System WMS tworzy `OutboundOrder`, musi ustawić najpilniejsze `priority`/`slaDeadline` i niezmienny `fulfillmentChannel`, a po utworzeniu `PickTask` nie może realokować zapasu z powodu priorytetu.
```gherkin
Scenariusz: Priorytet nie odbiera rozpoczętej realizacji
  Zakładając, że `PickTask` już istnieje
  Gdy pojawia się zamówienie o wyższym priorytecie
  Wtedy istniejąca realizacja nie jest realokowana
  Oraz nowe zamówienie konkuruje wyłącznie o zapas niezarezerwowany
```

### FR-P1-06 — Transfer ATPReservation i ręczna korekta
*Kategoria:* zdarzeniowe · *Źródło:* P1 R11–P1 R12
**Gdy** powstaje `Allocation RESERVED`, System WMS musi odjąć jej ilość z `ATPReservation` i utrzymać rezerwację twardą do `SHIPPED`; gdy `Warehouse Supervisor` zmniejszy `ATPReservation`, musi zachować status `CustomerOrder`.
```gherkin
Scenariusz: Miękka rezerwacja przechodzi w twardą
  Zakładając, że linia ma dodatnie `ATPReservation`
  Gdy powstaje `Allocation RESERVED`
  Wtedy odpowiadająca ilość znika z `ATPReservation`
  Oraz ręczne zmniejszenie pozostałej rezerwacji nie zmienia statusu zamówienia
```

### FR-P1-07 — PickTask per strefa i kolejność
*Kategoria:* konfiguracyjne · *Źródło:* P1 R13–P1 R14
**Tam gdzie** `OutboundOrder` obejmuje wiele stref, System WMS musi utworzyć wiele `PickTask`, każdy dla jednego operatora, i prezentować je według skonfigurowanej kolejności `slaDeadline`/`priority` z remisem rozstrzyganym kolejnością zgłoszenia.
```gherkin
Scenariusz: Kompletacja wielostrefowa
  Zakładając, że zamówienie wymaga pobrania z dwóch stref
  Gdy System WMS generuje pracę
  Wtedy powstają dwa `PickTask`, po jednym na strefę
  Oraz ich kolejność odpowiada konfiguracji magazynu
```

### FR-P1-08 — Limit TU i deklaracja direct pack
*Kategoria:* zdarzeniowe · *Źródło:* P1 R15–P1 R16
**Gdy** Picker skanuje pierwszą Picking `TU`, System WMS musi umożliwić wiążącą deklarację `directPackDeclared`, a podczas kompletacji blokować przekroczenie limitu masy `TUSetup.maxWeight` i ustawić `PICK_FULL` po osiągnięciu tego limitu.
```gherkin
Scenariusz: Deklaracja jest nieodwracalna
  Zakładając, że Picker zadeklarował direct pack przy pierwszym skanie TU
  Gdy rozpoczął kompletację
  Wtedy deklaracji nie można cofnąć dla tej `TU`
  Oraz przekroczenie limitu masy `TUSetup.maxWeight` jest blokowane
```

### FR-P1-09 — Automatyczne direct pack i odstępstwo
*Kategoria:* warunkowe · *Źródło:* P1 R17–P1 R18
**Jeśli** `directPackDeclared = true` i `TU` spełnia progi wydania, to System WMS musi automatycznie wykonać przejścia do `PACKING_SEALED`/`PACKED`; odstępstwo Packera od sugestii musi wymagać zgody `Warehouse Supervisor`.
```gherkin
Scenariusz: Direct pack bez Packera
  Zakładając, że kompletacja direct pack zakończyła się na wydawalnej TU
  Gdy System WMS oceni progi wydania
  Wtedy TU przechodzi do `PACKING_SEALED`, a linia do `PACKED`
  Oraz Packer nie uczestniczy w tej ścieżce
```

### FR-P1-10 — Kwalifikacja i zgodność pakowania
*Kategoria:* warunkowe · *Źródło:* P1 R19–P1 R20
**Jeśli** Picking `TU` spełnia progi wydania, to System WMS musi zachować jej `TU_NUMBER` jako Packing `TU`; wspólne pakowanie SKU musi podlegać limitowi masy `TUSetup.maxWeight`, bez dodatkowych ograniczeń zgodności w wersji 1.
```gherkin
Scenariusz: Picking TU zostaje Packing TU
  Zakładając, że Picking TU spełnia progi wydania i limit masy `TUSetup.maxWeight`
  Gdy zostanie zakwalifikowana do pakowania
  Wtedy zachowuje `TU_NUMBER`
  Oraz może zawierać wiele SKU bez dodatkowej reguły kompatybilności, jeżeli ich łączna masa nie przekracza `TUSetup.maxWeight`
```

### FR-P1-11 — Repack by SKU i potwierdzenie braku
*Kategoria:* zdarzeniowe · *Źródło:* P1 R21–P1 R22
**Gdy** Packer wykonuje „repack by SKU”, System WMS musi pozostawić mu wybór SKU i kolejności, a ilość mniejszą od `pickedQty` uznać za brak dopiero po ponownym sprawdzeniu i jawnym potwierdzeniu.
```gherkin
Scenariusz: Brak wymaga dwóch działań
  Zakładając, że Packer znalazł mniej sztuk niż `pickedQty`
  Gdy ponownie sprawdzi TU i potwierdzi brak
  Wtedy System WMS uruchamia mechanizm `SHORT_PICKED`
  Oraz przed potwierdzeniem mechanizm nie jest uruchamiany
```

### FR-P1-12 — DAMAGED i SKU nieoczekiwane
*Kategoria:* zdarzeniowe · *Źródło:* P1 R23–P1 R24
**Gdy** podczas przepakowania wykryto `DAMAGED` albo nieoczekiwane SKU, System WMS musi skierować towar na `QC`; dla `DAMAGED` uruchomić mechanizm braku, a dla nieoczekiwanego SKU nie uruchamiać tego mechanizmu.
```gherkin
Scenariusz: Różne skutki QC
  Zakładając, że Packer wykrywa jedną sztukę uszkodzoną i jeden obcy kod SKU
  Gdy rejestruje niezgodności
  Wtedy oba towary trafiają na `QC`
  Oraz tylko `DAMAGED` uruchamia mechanizm `SHORT_PICKED`
```

### FR-P1-13 — Domknięcie przepakowania i grupowanie Shipment
*Kategoria:* warunkowe · *Źródło:* P1 R25–P1 R26
**Jeśli** wszystkie oczekiwane SKU zostały rozliczone, System WMS musi pozwolić zakończyć przepakowanie; grupowanie Packing `TU` w `Shipment` musi wymagać zgodnego klienta, adresu, priorytetu i identycznego `slaDeadline`.
```gherkin
Scenariusz: Nierozliczone SKU blokuje zamknięcie
  Zakładając, że jedno oczekiwane SKU nie ma dyspozycji
  Gdy Packer próbuje zakończyć przepakowanie
  Wtedy System WMS kieruje go do kontynuacji albo zgłoszenia braku
  Oraz nie grupuje TU z innym `slaDeadline`
```

### FR-P1-14 — Agregacja PACKED i READY_FOR_DISPATCH
*Kategoria:* zdarzeniowe · *Źródło:* P1 R27–P1 R28
**Gdy** wszystkie linie i TU `OutboundOrder` są spakowane, System WMS musi ustawić `PACKED` i dalej `READY_FOR_DISPATCH`; `Shipment` musi osiągnąć `READY_FOR_DISPATCH` po gotowości wszystkich kontrybuujących zamówień albo po wspólnym `slaDeadline`.
```gherkin
Scenariusz: Timeout zamyka grupowanie Shipment
  Zakładając, że nie wszystkie planowane zamówienia zdążyły się spakować
  Gdy upłynie wspólny `slaDeadline`
  Wtedy bieżący `Shipment` przechodzi do `READY_FOR_DISPATCH`
  Oraz obejmuje wyłącznie gotowe TU
```

### FR-P1-15 — Spóźnione TU i start Carrier Selection
*Kategoria:* zdarzeniowe · *Źródło:* P1 R29–P1 R30
**Gdy** TU spóźnionego `OutboundOrder` stają się gotowe po zamknięciu grupowania, System WMS musi utworzyć nowy `Shipment`; Carrier Selection może rozpocząć dopiero dla `Shipment READY_FOR_DISPATCH` z kompletnym zestawem TU.
```gherkin
Scenariusz: Spóźniona TU tworzy nową wysyłkę
  Zakładając, że poprzedni Shipment zamknięto przez `slaDeadline`
  Gdy kolejna TU staje się gotowa
  Wtedy System WMS grupuje ją w nowy `Shipment`
  Oraz dopiero potem uruchamia Carrier Selection
```

### FR-P1-16 — CarrierSetup i tie-break
*Kategoria:* warunkowe · *Źródło:* P1 R31–P1 R32
**Jeśli** wiele `CarrierSetup` pasuje do `Region` oraz wagi/objętości, to System WMS musi wybrać kolejno najwęższy przedział objętości, najwęższy przedział wagi i unikalny `Carrier.priority`.
```gherkin
Scenariusz: Deterministyczny wybór carriera
  Zakładając, że dwa `CarrierSetup` pasują do Shipment
  Gdy System WMS stosuje tie-break
  Wtedy wybiera rekord z najwęższym przedziałem objętości
  Oraz używa kolejnych kryteriów tylko przy remisie
```

### FR-P1-17 — Ręczna zmiana carriera i etykieta WMS
*Kategoria:* zdarzeniowe · *Źródło:* P1 R33–P1 R34
**Gdy** `Warehouse Supervisor` zmieni wynik Carrier Selection, System WMS musi zapisać nowego `Carrier` bez wymagania powodu; etykietę musi generować lokalnie, bez API i akceptacji przewoźnika.
```gherkin
Scenariusz: Supervisor zmienia wynik
  Zakładając, że automatycznie wybrano przewoźnika
  Gdy Supervisor wskaże innego
  Wtedy Shipment używa nowego `Carrier`
  Oraz etykieta powstaje jako wydruk WMS bez wywołania zewnętrznego
```

### FR-P1-18 — Problem załadunku i bramka ERP
*Kategoria:* warunkowe · *Źródło:* P1 R35–P1 R36
**Jeśli** problem z załadunkiem wystąpi przed `CarrierManifest.CLOSED`, System WMS musi umożliwić zmianę `Carrier` bez ponownego druku; każdy `Shipment` musi osiągnąć `POSTED` przed dodaniem do manifestu.
```gherkin
Scenariusz: Nie można manifestować przed POSTED
  Zakładając, że Shipment ma `POSTING_PENDING`
  Gdy Dispatcher próbuje dodać go do manifestu
  Wtedy System WMS odrzuca operację
  Oraz pozwala na nią dopiero po `POSTED`
```

### FR-P1-19 — Retry ERP i granica anulowania
*Kategoria:* warunkowe · *Źródło:* P1 R37–P1 R38
**Jeśli** Shipment ma `POSTING_ERROR`, System WMS musi wymagać ręcznej decyzji `Warehouse Supervisor` do retry; od `POSTING_PENDING` nie może pozwolić na anulowanie w WMS.
```gherkin
Scenariusz: Retry wymaga decyzji
  Zakładając, że Shipment ma `POSTING_ERROR`
  Gdy brak decyzji Supervisora
  Wtedy ponowienie nie jest wysyłane
  Oraz anulowanie od `POSTING_PENDING` pozostaje niedozwolone
```

### FR-P1-20 — Jeden manifest i nieodwracalne CLOSED
*Kategoria:* stanowe · *Źródło:* P1 R39–P1 R40
**Dopóki** `Shipment` nie należy do manifestu, System WMS musi pozwolić przypisać go do dokładnie jednego `CarrierManifest`; po `CLOSED` musi blokować ponowne otwarcie, anulowanie i zmianę `Carrier`.
```gherkin
Scenariusz: Shipment nie trafia do dwóch manifestów
  Zakładając, że Shipment przypisano do otwartego manifestu
  Gdy Dispatcher próbuje dodać go do drugiego
  Wtedy System WMS odrzuca operację
  Oraz po zamknięciu pierwszego manifestu blokuje jego cofnięcie
```

### FR-P1-21 — Zamknięcie CustomerOrder i SHORT_ALLOCATED false
*Kategoria:* zdarzeniowe · *Źródło:* P1 R41–P1 R42
**Gdy** wszystkie kontrybuujące `OutboundOrder` osiągną `COMPLETED`, System WMS musi zamknąć `CustomerOrder`; przy anulowaniu po `SHORT_ALLOCATED` musi zwolnić alokacje, zachować problemową linię w `BACKORDERED` i pozostałe w `OPEN`.
```gherkin
Scenariusz: Anulowanie SHORT_ALLOCATED zachowuje zamówienie
  Zakładając, że Supervisor anuluje niedoborowy OutboundOrder
  Gdy System WMS rozlicza decyzję
  Wtedy problemowa linia jest `BACKORDERED`, pozostałe `OPEN`
  Oraz CustomerOrder może zostać `CLOSED` dopiero po zakończeniu wszystkich realizujących go zamówień
```

### FR-P1-22 — Automatyczna obsługa niedoborów
*Kategoria:* warunkowe · *Źródło:* P1 R43–P1 R44
**Jeśli** `allowPartialShipment = true`, System WMS musi realizować dostępną ilość i pozostawić brak w `BACKORDERED`; przy `SHORT_PICKED` musi automatycznie tworzyć kolejne `PickTask` do limitu, a potem eskalować.
```gherkin
Scenariusz: Limit realokacji kończy automat
  Zakładając, że linia ma `SHORT_PICKED` i osiągnęła efektywny limit
  Gdy System WMS szuka kolejnego zapasu
  Wtedy nie tworzy kolejnego `PickTask`
  Oraz eskaluje przypadek do `Warehouse Supervisor`
```

### FR-P1-23 — Czekamy albo anulowanie po SHORT_PICKED
*Kategoria:* warunkowe · *Źródło:* P1 R45–P1 R46
**Jeśli** Supervisor wybierze „czekamy”, System WMS musi cofnąć niespakowane linie tego `CustomerOrder`; jeśli wybierze „anulowanie”, musi wymagać edycji ilości `CustomerOrderLine`, a nie tylko anulowania `OutboundOrderLine`.
```gherkin
Scenariusz: Linie PACKED nie są cofane
  Zakładając, że jedna linia jest `PACKED`, a druga ma `SHORT_PICKED`
  Gdy Supervisor wybierze „czekamy”
  Wtedy linia `PACKED` pozostaje bez zmian
  Oraz niespakowane linie są cofane właściwym torem
```

### FR-P1-24 — Trwała zmiana i brak przy pakowaniu
*Kategoria:* zdarzeniowe · *Źródło:* P1 R47–P1 R48
**Gdy** Supervisor trwale ustawi `allowPartialShipment = true`, System WMS musi wymagać powodu i ograniczyć zmianę do wskazanego `CustomerOrder`; brak lub uszkodzenie przy pakowaniu musi obsłużyć jak `SHORT_PICKED`, bez blokady lokalizacji źródłowej.
```gherkin
Scenariusz: Zmiana nie wpływa na inne zamówienia
  Zakładając, że Supervisor zmienił flagę dla jednego CustomerOrder
  Gdy inny CustomerOrder tego klienta ma niedobór
  Wtedy używa własnej niezmienionej flagi
  Oraz brak przy pakowaniu pierwszego zamówienia nie blokuje lokalizacji źródłowej
```

### FR-P1-25 — ON_HOLD i anulowanie ogólne
*Kategoria:* warunkowe · *Źródło:* P1 R49–P1 R50
**Jeśli** `CustomerOrder` jest `ON_HOLD`, System WMS musi pominąć je do zdjęcia blokady; anulowanie ogólne całego zamówienia musi dopuścić wyłącznie, gdy każda linia spełnia swój warunek i manifest nie jest `CLOSED`.
```gherkin
Scenariusz: Jedna blokująca linia odrzuca anulowanie całości
  Zakładając, że jedna linia nie spełnia warunku anulowania
  Gdy OMS żąda anulowania całego CustomerOrder
  Wtedy System WMS odrzuca żądanie ze wskazaniem linii
  Oraz zamówienie `ON_HOLD` wraca do planowania dopiero po odblokowaniu
```

### FR-P1-26 — Brak carriera i zapas ATP
*Kategoria:* warunkowe · *Źródło:* P1 R51–P1 R52
**Jeśli** Carrier Selection nie daje wyniku, System WMS musi eskalować wybór do wskazanych aktorów; `Allocation` musi rezerwować wyłącznie zapas z flagą `ATP`, odrzucając non-ATP.
```gherkin
Scenariusz: Non-ATP nie jest alokowany
  Zakładając, że dostępny fizycznie zapas nie ma flagi ATP
  Gdy System WMS wykonuje Allocation
  Wtedy nie rezerwuje tego zapasu
  Oraz brak wyniku carriera kieruje do ręcznego zatwierdzenia
```

### FR-P1-27 — Ciągła agregacja CustomerOrder
*Kategoria:* stanowe · *Źródło:* P1 Funkcja ciągła F1
**Dopóki** trwa realizacja `CustomerOrder`, System WMS musi wyliczać nagłówek ze statusu najmniej zaawansowanej aktywnej linii, z wyjątkami `PARTIALLY_SHIPPED` i pełnego `BACKORDERED`.
```gherkin
Scenariusz: Pojedyncza linia BACKORDERED nie cofa nagłówka
  Zakładając, że jedna linia jest `BACKORDERED`, a druga jest w realizacji
  Gdy System WMS przelicza nagłówek
  Wtedy nagłówek pozostaje w bieżącym statusie realizacji z `WARNING`
  Oraz `BACKORDERED` otrzymuje dopiero, gdy wszystkie aktywne linie są `BACKORDERED`
```

### FR-P1-28 — Tożsamość i unikalność TU_NUMBER
*Kategoria:* uniwersalne · *Źródło:* P1 R53
System WMS musi nadawać każdej Outbound `TU` wymagany `TU_NUMBER`, unikalny per magazyn wśród aktywnych (nieterminalnych) Outbound `TU`, przy opcjonalnym `SSCC`, który nie musi być równy `TU_NUMBER`, i zachowywać `TU_NUMBER` przy przejściu roli `PickContainer → PackUnit`. `TU_NUMBER`, który nie jest `SSCC`, musi zawierać wyłącznie znaki alfanumeryczne bez znaków specjalnych, mieć maksymalnie 20 znaków i być kodowany w symbolice Code 128.
```gherkin
Scenariusz: Numer aktywnej TU nie jest powielany
  Zakładając, że istnieje aktywna Outbound TU o danym TU_NUMBER
  Gdy powstaje kolejna Outbound TU w tym magazynie
  Wtedy otrzymuje inny TU_NUMBER
  Oraz numer zostaje zachowany przy przejściu roli PickContainer na PackUnit
  Oraz TU_NUMBER, który nie jest SSCC, jest alfanumeryczny, ma maksymalnie 20 znaków i jest kodowany w Code 128
```

### FR-P1-29 — Przydział PickTask przez wybór modułu i strefy
*Kategoria:* zdarzeniowe · *Źródło:* P1 R54, P1 R56
**Gdy** `Warehouse Operator` jest zalogowany do modułu zbierania ze wskazaną strefą i nie ma aktywnego zadania magazynowego żadnego typu, System WMS musi przydzielić mu kolejny `PickTask` tej strefy zgodnie ze skonfigurowaną kolejnością, i nie może przydzielić zadania operatorowi mającemu aktywne zadanie.
```gherkin
Scenariusz: Operator bez zadania dostaje kolejny PickTask swojej strefy
  Zakładając, że operator jest zalogowany do modułu zbierania w strefie A i nie ma aktywnego zadania
  Gdy System WMS wybiera kolejne zadanie do przydziału
  Wtedy operator otrzymuje najwyższy w kolejce PickTask strefy A
Scenariusz: Operator z aktywnym zadaniem nie dostaje kolejnego
  Zakładając, że operator ma już aktywny PickTask
  Gdy w jego strefie oczekuje inny PickTask
  Wtedy System WMS nie przydziela mu kolejnego zadania
```

### FR-P1-30 — Kontynuacja zamówienia w kolejnej strefie
*Kategoria:* zdarzeniowe · *Źródło:* P1 R55
**Gdy** `Warehouse Operator` kończy `PickTask`, System WMS musi umożliwić wybór między zamknięciem Picking `TU` a kontynuacją tego samego `OutboundOrder` w innej strefie do wciąż otwartej `TU`; przy wyborze kontynuacji musi przydzielić wskazany `PickTask` niezależnie od kolejności i strefy oraz zmienić aktywną strefę operatora.
```gherkin
Scenariusz: Operator kontynuuje zamówienie w innej strefie
  Zakładając, że operator zakończył PickTask strefy A dla otwartej Picking TU
  Gdy wybiera kontynuację tego samego OutboundOrder w strefie B
  Wtedy otrzymuje PickTask strefy B tego zamówienia poza kolejnością
  Oraz jego aktywna strefa zmienia się na B
```

### FR-P1-31 — Wydanie całości przy zakazie wysyłki częściowej
*Kategoria:* warunkowe · *Źródło:* P1 R57

**Jeśli** `CustomerOrder` ma `allowPartialShipment = false`, to System WMS musi wydać całe zamówienie jednym kompletnym `Shipment` i zablokować wydanie, dopóki brakuje choć jednej wymaganej pozycji; jeden `Shipment` może zawierać wiele Packing `TU` tego zamówienia. Obietnica musi obowiązywać niezależnie od tego, iloma `OutboundOrder` i iloma kanałami zamówienie zostało obsłużone.
```gherkin
Scenariusz: Niekompletne zamówienie nie jest wydawane
  Zakładając, że CustomerOrder ma allowPartialShipment = false
  Oraz zamówienie jest obsługiwane przez więcej niż jeden OutboundOrder albo kanał
  Gdy brakuje choć jednej wymaganej pozycji
  Wtedy System WMS nie wydaje Shipment
  Oraz po skompletowaniu wszystkich pozycji wydaje całe CustomerOrder w jednym kompletnym Shipment
```

### FR-P1-32 — Wstrzymanie wpięcia TU do Shipment
*Kategoria:* stanowe · *Źródło:* P1 R58

**Dopóki** każda `CustomerOrderLine` zamówienia z `allowPartialShipment = false` nie spełni jednej z dwóch rozłącznych alternatyw, System WMS nie może wpiąć Packing `TU` tego zamówienia do żadnego `Shipment`: (A) linia nieaktywna — skutecznie anulowana albo skorygowana przez `Warehouse Supervisor` zgodnie z `P1 R46`, nienależąca już do ilości wymaganej i niepodlegająca alternatywie (B); albo (B) linia aktywna — suma `requiredQty` tych jej `OutboundOrderLine`, które osiągnęły `PACKED` albo dalej i nie są `CANCELLED`, równa się `CustomerOrderLine.Quantity` obowiązującej po ewentualnej korekcie, niezależnie od kanału. Sam status `PLANNED` nie może spełniać tego warunku. `TU` musi pozostać w `PACKING_SEALED`, a automatyczne zamknięcie `Shipment` po `slaDeadline` nie może jej objąć, ponieważ nie należy ona do żadnego `Shipment`.
```gherkin
Scenariusz: Termin mija przed pełnym spakowaniem
  Zakładając, że CustomerOrder ma allowPartialShipment = false
  Oraz aktywna CustomerOrderLine ma Quantity = 100
  Oraz jej crossdockowa OutboundOrderLine ma requiredQty = 30 i osiągnęła PACKED
  Oraz dla pozostałej ilości 70 nie istnieje żadna OutboundOrderLine, mimo że CustomerOrderLine ma PLANNED
  Gdy upływa slaDeadline
  Wtedy Packing TU tego zamówienia pozostaje w PACKING_SEALED
  Oraz nie wchodzi do żadnego zamykanego Shipment
  Oraz suma requiredQty = 30 nie jest równa Quantity = 100
```

### FR-P1-33 — Podział pozycji zamówienia klienta
*Kategoria:* zdarzeniowe · *Źródło:* P1 R59

**Gdy** System WMS planuje `OutboundOrder`, musi dopuścić podział jednej `CustomerOrderLine` między wiele `OutboundOrderLine`.
```gherkin
Scenariusz: Jedna pozycja klienta trafia do dwóch zleceń
  Zakładając, że CustomerOrderLine wymaga ilości większej niż jedno zlecenie obejmie
  Gdy System WMS planuje OutboundOrder
  Wtedy pozycja zostaje podzielona między wiele OutboundOrderLine
```

### FR-P1-34 — Warunek wspólnego pakowania pozycji z różnych zleceń
*Kategoria:* warunkowe · *Źródło:* P1 R60

**Jeśli** Packing `TU` ma zawierać pozycje z wielu `OutboundOrder`, to System WMS musi wymagać tego samego klienta, tego samego adresu dostawy, zgodnego priorytetu i zgodnego terminu dostawy źródłowych `CustomerOrder` już przy przepakowaniu; wszystkie pozycje jednej Packing `TU` muszą trafić do jednego `Shipment`.
```gherkin
Scenariusz: Niezgodny adres blokuje wspólne pakowanie
  Zakładając, że dwa OutboundOrder mają różne adresy dostawy
  Gdy Packer próbuje przepakować ich pozycje do jednej Packing TU
  Wtedy System WMS nie pozwala na wspólne pakowanie
```

### FR-P1-35 — Cykl życia flagi WARNING
*Kategoria:* zdarzeniowe · *Źródło:* P1 R61

**Gdy** przyczyna `WARNING` na `CustomerOrder` ustanie, System WMS musi zdjąć flagę automatycznie; **Tam gdzie** `Warehouse Supervisor` zdejmie ją ręcznie mimo trwającej przyczyny, kolejny przebieg cyklicznego planowania musi ustawić ją ponownie, bez zmiany zachowania biznesowego.
```gherkin
Scenariusz: Ręczne zdjęcie flagi przy trwającej przyczynie
  Zakładając, że CustomerOrder ma WARNING z powodu braku pełnej rezerwacji
  Gdy Warehouse Supervisor ustawia flagę na false, a rezerwacja nadal jest niepełna
  Wtedy kolejny przebieg planowania ustawia WARNING ponownie
  Oraz zachowanie biznesowe zamówienia nie zmienia się
```

### FR-P1-36 — Strategia przypisania Picking TU
*Kategoria:* konfiguracyjne · *Źródło:* P1 R62

**Tam gdzie** magazyn konfiguruje strategię przypisania Picking `TU`, System WMS musi udostępnić dwa warianty: osobną Picking `TU` dla każdego zadania albo strefy, oraz wspólną Picking `TU` przechodzącą przez kolejne zadania tego samego `OutboundOrder`.
```gherkin
Scenariusz: Wariant wspólnej TU przechodzącej przez zadania
  Zakładając, że magazyn skonfigurował wariant wspólnej Picking TU
  Gdy operator kończy zadanie i kontynuuje ten sam OutboundOrder w innej strefie
  Wtedy zbiera do tej samej Picking TU
```

### FR-P1-37 — Wyliczanie masy i objętości zawartości `TU`
*Kategoria:* zdarzeniowe · *Źródło:* P1 R63

**Gdy** potrzebne jest wyznaczenie masy albo objętości zawartości `TU`, System WMS musi wyliczyć odpowiednią wartość jako sumę po wszystkich `SKU` w tej `TU` iloczynów odpowiednio wagi albo objętości jednostkowej `SKU` z kartoteki towarowej i liczby sztuk; wyliczoną objętość zawartości musi wykorzystywać wyłącznie do oceny progów wydania, nie do blokady zapełnienia ani limitów przepakowania, a `TUSetup.maxVolume` musi traktować jako kubaturę typu opakowania używaną przy doborze przewoźnika.
```gherkin
Scenariusz: Masa i objętość liczone z zawartości
  Zakładając, że w `TU` leży 20 sztuk `SKU` o wadze 2 kg i objętości 0,01 m³ oraz 10 sztuk `SKU` o wadze 1 kg i objętości 0,02 m³
  Gdy System WMS wyznacza masę i objętość zawartości `TU`
  Wtedy masa wynosi 50 kg, a objętość zawartości 0,4 m³
  Oraz objętość zawartości służy wyłącznie ocenie progów wydania, a nie blokadzie zapełnienia ani limitom przepakowania
  Oraz Carrier Selection używa `TUSetup.maxVolume` jako kubatury typu opakowania
```

### FR-P1-38 — Progi wydania Picking TU
*Kategoria:* warunkowe · *Źródło:* P1 R64

**Jeśli** System WMS ocenia Picking `TU` pod kątem wydania bez przepakowania, to System WMS musi zastosować dwie gałęzie w kolejności: (1) dla `TU` na typie o roli nośnika zewnętrznego (`P1 R68`) uznać progi wydania za spełnione z definicji, bez ich ewaluacji i bez odczytu `TUSetup.minIssueWeight` oraz `TUSetup.minIssueVolume`; (2) dla typów pozostałych uznać progi wydania za spełnione wyłącznie wtedy, gdy typ `TU` ma `TUSetup.externalIssuable = true` oraz osiągnięty jest co najmniej jeden z dwóch dolnych progów: bieżąca masa osiągnęła `TUSetup.minIssueWeight` albo bieżąca objętość zawartości osiągnęła bezwzględny próg `TUSetup.minIssueVolume`.
```gherkin
Scenariusz: Lekka, ale dobrze zapełniona TU spełnia progi
  Zakładając, że typ `TU` ma `TUSetup.externalIssuable = true` i `TUSetup.processUsage` inne niż `EXTERNAL`, a bieżąca masa nie osiągnęła `TUSetup.minIssueWeight`
  Gdy bieżąca objętość zawartości osiągnęła `TUSetup.minIssueVolume`
  Wtedy System WMS uznaje progi wydania za spełnione
```

### FR-P1-39 — Wymuszenie wydawalności przez operatora
*Kategoria:* zdarzeniowe · *Źródło:* P1 R65

**Gdy** `TU` na typie innym niż typ o roli nośnika zewnętrznego (`P1 R68`) nie osiągnęła dolnych progów masy ani objętości, ma `TUSetup.externalIssuable = true` i spełnia co najmniej jeden z dwóch warunków — jest ostatnią `TU` swojego `OutboundOrder` albo zbliża się `slaDeadline` — System WMS musi pozwolić `Warehouse Operator` zamknąć ją i wymusić wydawalność bez przepakowania, z zapisaniem powodu. Wymuszenie musi znosić wyłącznie dolne progi i nie może pozwalać na zapieczętowanie ani wydanie `TU` z `TUSetup.externalIssuable = false`; dla takiej `TU` jedyną drogą musi być przepakowanie na typ wydawalny zgodnie z `P1 R66`. Dla `TU` na typie zewnętrznym System WMS nie może wymagać tego wymuszenia, ponieważ wydawalność wynika z pierwszej gałęzi `P1 R64`.
```gherkin
Scenariusz: Ostatnia TU zamówienia poniżej progów
  Zakładając, że `TU` ma `TUSetup.externalIssuable = true` i `TUSetup.processUsage` inne niż `EXTERNAL`, nie osiągnęła dolnych progów i jest ostatnią `TU` swojego `OutboundOrder`
  Gdy `Warehouse Operator` zamyka ją i podaje powód
  Wtedy System WMS wydaje `TU` bez przepakowania
  Oraz zapisuje powód wymuszenia

Scenariusz: Typ niewydawalny blokuje wymuszenie operatora
  Zakładając, że `TU` ma `TUSetup.externalIssuable = false`, nie osiągnęła dolnych progów i jest ostatnią `TU` swojego `OutboundOrder`
  Gdy `Warehouse Operator` próbuje zamknąć ją i wymusić wydawalność
  Wtedy System WMS nie pozwala jej zapieczętować ani wydać
  Oraz jedyną drogą jest przepakowanie na typ wydawalny zgodnie z `P1 R66`
```

### FR-P1-40 — Wydawalny typ nośnika przy przepakowaniu
*Kategoria:* warunkowe · *Źródło:* P1 R66

**Jeśli** Packing `TU` powstaje przez przepakowanie, to System WMS musi zezwolić na jej zapieczętowanie wyłącznie wtedy, gdy jej typ jest oznaczony jako wydawalny na zewnątrz.
```gherkin
Scenariusz: Przepakowanie do typu niewydawalnego
  Zakładając, że Packer przepakowuje towar do nośnika typu bez flagi externalIssuable
  Gdy próbuje zapieczętować Packing TU
  Wtedy System WMS nie pozwala jej zapieczętować
```

### FR-P1-41 — Kontynuacja PickTask w kolejnej Picking TU
*Kategoria:* zdarzeniowe · *Źródło:* P1 R67

**Gdy** Picking `TU` osiągnie `PICK_FULL`, a w `PickTask` pozostaje ilość do pobrania, System WMS musi umożliwić Pickerowi zamknięcie pełnej `TU`, utworzyć i przy pierwszym skanie powiązać kolejną Picking `TU` z tym samym `PickTask` oraz kontynuować zadanie bez zmiany jego tożsamości; `PickTask` może osiągnąć `COMPLETED` dopiero po pobraniu pełnej ilości, a kolejna `TU` musi odziedziczyć `directPackDeclared` poprzedniej `TU` tego samego `PickTask`, bez ponownego pytania operatora, zgodnie z `P1 R16`.
```gherkin
Scenariusz: Kontynuacja PickTask w kolejnej Picking TU
  Zakładając, że pierwsza Picking `TU` osiągnęła `PICK_FULL`, a w `PickTask` pozostaje ilość do pobrania
  Gdy Picker zamyka pełną `TU` i skanuje kolejną Picking `TU`
  Wtedy pierwsza `TU` przechodzi `PICK_FULL → READY_TO_PACK`, a druga `CREATED → IN_PICKING`
  Oraz System WMS wiąże drugą `TU` z tym samym `PickTask`, który zachowuje tożsamość i pozostaje `IN_PROGRESS`
  Oraz druga `TU` dziedziczy `directPackDeclared` pierwszej, bez ponownego pytania operatora
  Oraz `PickTask` osiąga `COMPLETED` dopiero po pobraniu pełnej ilości
```

### FR-P1-42 — Typ nośnika zewnętrznego
*Kategoria:* uniwersalne · *Źródło:* P1 R68

System WMS musi utrzymywać dokładnie jeden `TUSetup` o `processUsage = EXTERNAL` i `externalIssuable = true` oraz rozpoznawać Outbound `TU` pochodzenia zewnętrznego wyłącznie po `processUsage` typu wskazanego przez jej `tuSetupCode`, nigdy po `TU_NUMBER` ani po dodatkowej fladze; dla takiej `TU` pierwsza gałąź `P1 R64` musi uznawać progi wydania za spełnione bez odczytu `minIssueWeight` i `minIssueVolume`, a `P1 R65` nie może być wymagana.
```gherkin
Scenariusz: Rozpoznanie i wydawalność TU pochodzenia zewnętrznego
  Zakładając, że katalog zawiera dokładnie jeden `TUSetup` o `processUsage = EXTERNAL` i `externalIssuable = true`
  Oraz `TU.tuSetupCode` wskazuje ten typ
  Gdy System WMS rozpoznaje pochodzenie `TU` i ocenia progi wydania
  Wtedy rozpoznaje ją jako zewnętrzną wyłącznie po `processUsage` wskazanego typu, bez użycia `TU_NUMBER` ani dodatkowej flagi
  Oraz pierwsza gałąź `P1 R64` uznaje progi za spełnione bez odczytu `minIssueWeight` i `minIssueVolume`
  Oraz ręczne wymuszenie z `P1 R65` nie jest wymagane
```

### FR-P1-43 — Cykl życia requiredQty
*Kategoria:* zdarzeniowe · *Źródło:* P1 R69

**Gdy** System WMS tworzy `OutboundOrderLine`, musi ustawić jej `requiredQty` jako obowiązującą ilość docelową — podczas planowania `OutboundOrder` w kanale `STANDARD` albo generowania `CrossDockPickTask` w kanale `CROSSDOCK`; **gdy** `Warehouse Supervisor` koryguje `CustomerOrderLine.Quantity` w `P1 R46` albo gałęzi korekty `P2 R15`, System WMS musi zmienić razem z nią `requiredQty` właściwej linii. Poza tymi dwiema ścieżkami `requiredQty` nie może się zmienić automatycznie.
```gherkin
Scenariusz: Korekta standardowa zmienia requiredQty
  Zakładając, że `CustomerOrderLine.Quantity = 100`, a jej jedna `OutboundOrderLine.requiredQty = 100` i `pickedQty = 30`
  Gdy `Warehouse Supervisor` koryguje `CustomerOrderLine.Quantity` do 30 w `P1 R46`
  Wtedy System WMS zmienia `OutboundOrderLine.requiredQty` do 30

Scenariusz: Korekta crossdockowa zmienia requiredQty
  Zakładając, że crossdockowa `OutboundOrderLine.requiredQty = 100`, a jej jedyny `CrossDockPickTask.confirmedQty = 30`
  Gdy `Warehouse Supervisor` koryguje `CustomerOrderLine.Quantity` do 30 w gałęzi `P2 R15`
  Wtedy System WMS zmienia `OutboundOrderLine.requiredQty` do 30
  Oraz `confirmedQty = requiredQty`, więc linia może kontynuować do `PACKED`

Scenariusz: Bez korekty requiredQty nie zmienia się automatycznie
  Zakładając, że `CustomerOrderLine.Quantity = 100`, a jedyna `OutboundOrderLine` ma `requiredQty = 30` i status `PACKED`
  Gdy nie wykonano korekty w `P1 R46` ani `P2 R15`
  Wtedy `requiredQty` pozostaje równe 30
  Oraz suma `requiredQty` wynosi 30 i nie jest równa `CustomerOrderLine.Quantity = 100`
```

### FR-P1-44 — Zakończenie OutboundOrder po wszystkich jego Shipment
*Kategoria:* zdarzeniowe · *Źródło:* P1 R70

**Gdy** `CarrierManifest` osiągnie `CONFIRMED`, System WMS musi wykonać `OutboundOrder DISPATCHED → COMPLETED` wyłącznie dla tych zleceń, których każdy `Shipment` zawierający ich Outbound `TU` należy do `CarrierManifest` w stanie `CONFIRMED`; **dopóki** choć jeden taki `Shipment` nie jest potwierdzony, zlecenie musi pozostać w `DISPATCHED`, niezależnie od kolejności potwierdzania manifestów.
```gherkin
Scenariusz: Pierwszy potwierdzony manifest nie kończy zlecenia
  Zakładając, że `OutboundOrder` ma Packing `TU` w `Shipment` A i w `Shipment` B
  Gdy `CarrierManifest` z `Shipment` A osiągnie `CONFIRMED`, a `Shipment` B nie jest jeszcze potwierdzony
  Wtedy `OutboundOrder` pozostaje w `DISPATCHED`

Scenariusz: Ostatni potwierdzony manifest kończy zlecenie
  Zakładając, że `Shipment` A tego `OutboundOrder` jest już potwierdzony
  Gdy `CarrierManifest` z `Shipment` B osiągnie `CONFIRMED`
  Wtedy `OutboundOrder` przechodzi `DISPATCHED → COMPLETED`

Scenariusz: Jedno Shipment na zlecenie bez zmiany zachowania
  Zakładając, że wszystkie Packing `TU` zlecenia należą do jednego `Shipment`
  Gdy `CarrierManifest` tego `Shipment` osiągnie `CONFIRMED`
  Wtedy `OutboundOrder` przechodzi `DISPATCHED → COMPLETED` w tym samym momencie co dotychczas
```

### FR-P1-45 — Ilość zablokowana przez Allocation
*Kategoria:* zdarzeniowe · *Źródło:* P1 R71

**Gdy** System WMS tworzy `Allocation` albo zmienia jej stan, musi utrzymywać `reservedQty` jako ilość faktycznie zablokowaną: `0` w `PENDING`, ilość częściową w `SHORT`, `OutboundOrderLine.requiredQty` przy wejściu w `RESERVED` i `CONFIRMED`, `0` w `RELEASED` i `CONSUMED`; **dopóki** alokacja nie jest w `SHORT`, `RESERVED` albo `CONFIRMED`, nie może wnosić nic do ilości zapasu uznanej za zajętą. Pomniejszanie `reservedQty` w trakcie `CONFIRMED` przy częściowym wydaniu opisuje `FR-P1-46`.
```gherkin
Scenariusz: Rezerwacja częściowa blokuje tylko to, co zarezerwowano
  Zakładając, że `OutboundOrderLine.requiredQty` wynosi 100
  Gdy System WMS zarezerwuje 30 i ustawi `Allocation SHORT`
  Wtedy `reservedQty` wynosi 30
  Oraz ilość zajęta przez tę alokację wynosi 30, nie 100

Scenariusz: Pełna rezerwacja i pobranie nie zmieniają ilości zajętej
  Zakładając, że `Allocation` osiągnęła `RESERVED` z `reservedQty` równym `requiredQty`
  Gdy `Allocation` przejdzie `RESERVED → CONFIRMED`
  Wtedy `reservedQty` pozostaje bez zmian
  Oraz zapas nadal jest uznany za zajęty

Scenariusz: Stany terminalne nie blokują zapasu
  Zakładając, że `Allocation` osiągnęła `RELEASED` albo `CONSUMED`
  Gdy System WMS wylicza ilość zapasu zajętą
  Wtedy ta alokacja wnosi 0
  Oraz `PENDING` również wnosi 0
```

### FR-P1-46 — Rozliczenie linii wydawanej wieloma Shipment
*Kategoria:* zdarzeniowe · *Źródło:* P1 R72

**Gdy** `CarrierManifest` osiągnie `CONFIRMED`, System WMS musi rozliczyć `Inventory PICKED → SHIPPED` dla ilości znajdującej się w Outbound `TU` tego manifestu i pomniejszyć o nią `Allocation.reservedQty`; **dopóki** choć jedna Outbound `TU` wnosząca ilość danej `OutboundOrderLine` nie należy do `Shipment` z manifestem `CONFIRMED`, System WMS nie może wykonać `OutboundOrderLine PACKED → SHIPPED` ani `Allocation CONFIRMED → CONSUMED`, niezależnie od kolejności potwierdzania manifestów.
```gherkin
Scenariusz: Częściowe wydanie nie terminalizuje linii ani alokacji
  Zakładając, że `OutboundOrderLine.requiredQty` wynosi 100, a jej towar jest w dwóch Outbound `TU` po 50, w dwóch różnych `Shipment`
  Gdy manifest pierwszego `Shipment` osiągnie `CONFIRMED`
  Wtedy `Inventory` rozlicza 50 sztuk jako `SHIPPED`, a `Allocation.reservedQty` spada ze 100 do 50
  Oraz `OutboundOrderLine` pozostaje w `PACKED`, a `Allocation` w `CONFIRMED`

Scenariusz: Ostatnie wydanie domyka linię i alokację
  Zakładając, że pozostała jedna niepotwierdzona Outbound `TU` tej linii
  Gdy manifest jej `Shipment` osiągnie `CONFIRMED`
  Wtedy `Allocation.reservedQty` osiąga 0
  Oraz `OutboundOrderLine` przechodzi `PACKED → SHIPPED`, a `Allocation` `CONFIRMED → CONSUMED`

Scenariusz: Jedna TU i jedno Shipment bez zmiany zachowania
  Zakładając, że cała ilość linii jest w jednej Outbound `TU`, w jednym `Shipment`
  Gdy manifest tego `Shipment` osiągnie `CONFIRMED`
  Wtedy `OutboundOrderLine` przechodzi `PACKED → SHIPPED`, a `Allocation` `CONFIRMED → CONSUMED` w tym samym momencie co dotychczas
```

## P2 — Outbound Crossdock

### FR-P2-01 — Kolejka demand i dopasowanie 1:1
*Kategoria:* zdarzeniowe · *Źródło:* P2 R1–P2 R2
**Gdy** Inbound `TU` wejdzie w `IN_CROSS_DOCK`, System WMS musi generować `CrossDockPickTask` dla linii `BACKORDERED` według kolejki magazynu, a pełne dopasowanie 1:1 skierować do jednego `Shipment` wyłącznie wtedy, gdy cała zadeklarowana zawartość źródłowej `TU` pokrywa zapotrzebowanie tego samego klienta i adresu dostawy, ze zgodnym `priority` i identycznym `slaDeadline`; przy `allowPartialShipment = true` zapotrzebowanie może pochodzić z kilku `CustomerOrder` tego samego klienta, przy `allowPartialShipment = false` — wyłącznie z jednego, a źródłowa `TU` może zawierać wiele `SKU`.
```gherkin
Scenariusz: Pełne dopasowanie 1:1
  Zakładając, że cała zadeklarowana zawartość źródłowej TU pokrywa zapotrzebowanie tego samego klienta i adresu, ze zgodnym priority i identycznym slaDeadline
  Oraz źródłowa TU zawiera wiele SKU, a przy allowPartialShipment = true zapotrzebowanie pochodzi z dwóch CustomerOrder tego samego klienta
  Gdy System WMS generuje zadanie cross-dock
  Wtedy cała ilość zasila jedną docelową TU i jeden Shipment
  Oraz demand wybrano zgodnie z kolejką

Scenariusz: Brak łączenia zamówień bez zgody na wysyłkę częściową
  Zakładając, że zapotrzebowanie o zgodnym kliencie, adresie, priority i identycznym slaDeadline pochodzi z dwóch CustomerOrder, z których jeden ma allowPartialShipment = false
  Gdy System WMS ocenia pełne dopasowanie 1:1
  Wtedy nie kwalifikuje połączonego zapotrzebowania obu CustomerOrder jako pełnego dopasowania 1:1
```

### FR-P2-02 — Rozsortowanie n:n bez Allocation
*Kategoria:* uniwersalne · *Źródło:* P2 R3–P2 R4
System WMS musi umożliwiać zasilanie jednej Outbound `TU` z wielu Inbound `TU` dla `OutboundOrderLine` zgodnych w kliencie/adresie/`priority` i o identycznym `slaDeadline` (dopasowanie ścisłe, nie „zbliżone") i nie może tworzyć `Allocation` w cross-dockingu.
```gherkin
Scenariusz: Dwie źródłowe TU zasilają jedną docelową
  Zakładając, że dwie Inbound TU zawierają zgodne SKU dla jednej wysyłki
  Gdy Packer rozsortuje ich zawartość
  Wtedy jedna Outbound TU może zebrać ilość z obu źródeł
  Oraz nie powstaje `Allocation`
```

### FR-P2-03 — Kanał CROSSDOCK i eligibleQty
*Kategoria:* zdarzeniowe · *Źródło:* P2 R5–P2 R6
**Gdy** powstaje cross-dockowy `OutboundOrder`, System WMS musi ustawić niezmienny `fulfillmentChannel = CROSSDOCK`, nie mieszać kanałów w jednym `OutboundOrder` i wyliczyć `crossDockEligibleQty` jako min(`sourceEligibleQty`, `demandEligibleQty`), gdzie `sourceEligibleQty` pomniejsza zadeklarowaną (ASN) ilość źródłowej Inbound `TU`/`SKU` o `plannedQty` aktywnych oraz o `confirmedQty` i `damagedQty` zakończonych `CrossDockPickTask`, a `demandEligibleQty` pomniejsza `CustomerOrderLine.Quantity` o przyznane jej `ATPReservation` oraz o sumę `requiredQty` wszystkich jej `OutboundOrderLine`, które nie są `CANCELLED`, niezależnie od kanału.
```gherkin
Scenariusz: Uszkodzona ilość nie wraca do puli kwalifikowalnej
  Zakładając, że deklaracja ASN wynosi 100, aktywne zadania mają plannedQty 30, a zakończone 40 confirmedQty i 10 damagedQty
  Gdy System WMS wylicza sourceEligibleQty
  Wtedy wynik wynosi 20
  Oraz kanał CROSSDOCK pozostaje niezmienny dla całego OutboundOrder
```

### FR-P2-04 — Numer docelowej TU i start rozsortowania
*Kategoria:* zdarzeniowe · *Źródło:* P2 R7–P2 R8
**Gdy** docelowa Outbound `TU` zostanie użyta po raz pierwszy, System WMS musi nadać jej `TU_NUMBER`/`SSCC` — przy pełnym dopasowaniu 1:1 z poprawnym `SSCC` GS1 na źródłowej Inbound `TU` przez obowiązkowe dziedziczenie `TU_NUMBER` i `SSCC`, a w przeciwnym razie oraz przy rozsortowaniu n:n przez nadanie nowego numeru z `TUSetup`/`Sequence` wraz z nową etykietą; start skanowania musi przenieść zadanie i linię do `IN_PROGRESS`/`PICKING`, a nagłówek do `PACKING_IN_PROGRESS`.
```gherkin
Scenariusz: Dziedziczenie numeru tylko przy 1:1 z poprawnym GS1
  Zakładając, że źródłowa Inbound TU ma TU_NUMBER będący poprawnym SSCC GS1 i pełne dopasowanie 1:1
  Gdy powstaje docelowa Outbound TU
  Wtedy dziedziczy TU_NUMBER i SSCC bez nowej etykiety
  Oraz przy rozsortowaniu n:n otrzymuje nowy numer z Sequence i nową etykietę
```

### FR-P2-05 — Blokada anulowania i zamknięcie docelowej TU
*Kategoria:* warunkowe · *Źródło:* P2 R9–P2 R10
**Jeśli** rozsortowanie trwa, System WMS musi odrzucać ogólne anulowanie do `PACKED`; docelową Outbound `TU` musi zamykać w `PACKING_SEALED` w jednym z dwóch przypadków: (1) Packer zamyka ją jako fizycznie pełną w trakcie zadania i kontynuuje rozsortowanie do nowej `TU` w ramach tego samego `CrossDockPickTask`; (2) System WMS zamyka ją automatycznie, gdy jej `slaDeadline` zostanie osiągnięty (albo minięty) i żaden aktywny ani zaplanowany `CrossDockPickTask` nie wskazuje jej już jako celu — przed osiągnięciem `slaDeadline` sam brak takich zadań nie zamyka `TU`.
```gherkin
Scenariusz: Zakończenie jednego zadania nie zamyka współdzielonej TU
  Zakładając, że docelowa Outbound TU jest zasilana przez zadania z dwóch źródłowych Inbound TU
  Gdy pierwsze z tych zadań osiągnie COMPLETED
  Wtedy TU pozostaje otwarta, bo drugie zadanie nadal ma ją jako cel
  Oraz anulowanie ogólne w tym czasie jest odrzucone
```

### FR-P2-06 — Potwierdzony brak i DAMAGED
*Kategoria:* zdarzeniowe · *Źródło:* P2 R11–P2 R12
**Gdy** Packer stwierdzi ilość mniejszą od planowanej, System WMS musi wymagać ponownego sprawdzenia i potwierdzenia; wykryte `DAMAGED` musi skierować na `QC`.
```gherkin
Scenariusz: Niedobór nie powstaje po pierwszym odczycie
  Zakładając, że odczytano ilość mniejszą od planowanej
  Gdy Packer nie potwierdzi ponownego sprawdzenia
  Wtedy brak nie jest rejestrowany
  Oraz towar oznaczony `DAMAGED` trafia na `QC`
```

### FR-P2-07 — damagedQty i decyzja czekamy
*Kategoria:* warunkowe · *Źródło:* P2 R13–P2 R14
**Jeśli** ilość została zgłoszona jako `DAMAGED`, System WMS musi włączyć ją do `damagedQty`, bez zwykłego braku; przy decyzji „czekamy” musi anulować linię, zwrócić potwierdzoną ilość przez P4 i pozostawić demand w `BACKORDERED`.
```gherkin
Scenariusz: Zwykły brak nie zwiększa damagedQty
  Zakładając, że część ilości jest brakująca, lecz nie uszkodzona
  Gdy zadanie zostanie rozliczone
  Wtedy `damagedQty` nie obejmuje brakującej ilości
  Oraz przy decyzji „czekamy” potwierdzona ilość trafia do PutBack
```

### FR-P2-08 — Anulowanie niedoboru i poprawny stan źródłowy
*Kategoria:* zdarzeniowe · *Źródło:* P2 R15–P2 R16
**Gdy** Supervisor wybierze anulowanie niedoboru, System WMS musi anulować pełną `CustomerOrderLine` albo skorygować ją do `confirmedQty`; po rozpoczęciu sortowania `OutboundOrderLine` musi przejść do `CANCELLED` z `PICKING`.
```gherkin
Scenariusz: Korekta do confirmedQty kontynuuje wysyłkę
  Zakładając, że potwierdzono część planowanej ilości
  Gdy Supervisor koryguje CustomerOrderLine do `confirmedQty`
  Wtedy nie powstaje PutBackTask dla tej ilości
  Oraz linia może kontynuować do `PACKING_SEALED`
```

### FR-P2-09 — SKU nieoczekiwane i pusta TU
*Kategoria:* zdarzeniowe · *Źródło:* P2 R17, P2 R18, P2 R37
**Gdy** Packer wykryje nieoczekiwane SKU, System WMS musi skierować je na `QC` bez mechanizmu braku; gdy źródłowa TU jest pusta przed pobraniem, musi anulować zadanie/linię, ustawić demand `BACKORDERED` i Inbound `TU → LOST`; gdy w ten sposób wszystkie `OutboundOrderLine` danego `OutboundOrder` osiągną `CANCELLED` bez żadnej `PACKED`, nagłówek `OutboundOrder` musi przejść w `CANCELLED` — kryterium ogólne (`P2 R37`), nie specyficzne dla tej ścieżki.
```gherkin
Scenariusz: Pusta źródłowa TU anuluje pusty OutboundOrder
  Zakładając, że żadna linia OutboundOrder nie osiągnęła `PACKED`
  Gdy źródłowa TU okaże się pusta
  Wtedy zadanie i linia są `CANCELLED`, a demand `BACKORDERED`
  Oraz nagłówek OutboundOrder przechodzi do `CANCELLED`
```

### FR-P2-10 — PutBack bez Allocation i kompletność zadania
*Kategoria:* zdarzeniowe · *Źródło:* P2 R19–P2 R20
**Gdy** potwierdzona ilość cross-dockowa traci demand po `PACKED`, System WMS musi odzyskać ją przez `PutBackTask` bez kroku `Allocation RELEASED`; `CrossDockPickTask` musi osiągać `COMPLETED` w momencie zgłoszenia zakończenia przez Packera, niezależnie od pozostałych zadań tej samej źródłowej Inbound `TU` i od tego, czy cała jej zadeklarowana (ASN) zawartość została rozliczona.
```gherkin
Scenariusz: Zadanie kończy się niezależnie od pozostałych
  Zakładając, że jedna źródłowa TU o deklaracji 100 ma trzy zadania o plannedQty 30, 20 i 10
  Gdy Packer zgłosi zakończenie pierwszego zadania
  Wtedy to zadanie osiąga COMPLETED, a źródłowa TU pozostaje w cross-dockingu
  Oraz nie powstaje krok zwolnienia Allocation
```

### FR-P2-11 — Resztka i agregacja damagedQty
*Kategoria:* zdarzeniowe · *Źródło:* P2 R21–P2 R22
**Gdy** po cross-dockingu pozostaje rezydualna ilość, System WMS musi skierować ją na `TRANSIT` i ustawić Inbound `TU → IN_PUTAWAY`; do Inbound musi przekazać sumę `confirmedQty` oraz sumę `damagedQty` wszystkich zadań tej `TU` jako osobne składniki, tak aby ilość rezydualna per `SKU` — zadeklarowana (ASN) ilość pomniejszona o oba te składniki — dała się wyznaczyć po stronie Inbound Proces 3, do którego należy jej definicja i kontrola ilościowa.
```gherkin
Scenariusz: Resztka wraca do Putaway
  Zakładając, że część deklarowanej ilości nie została przypisana
  Gdy zakończą się wszystkie zadania źródłowej TU
  Wtedy resztka trafia na `TRANSIT`, a TU do `IN_PUTAWAY`
  Oraz Inbound otrzymuje zagregowane `damagedQty`

Scenariusz: Uszkodzona ilość pomniejsza ilość rezydualną
  Zakładając, że deklaracja ASN źródłowej TU wynosi 100, a zakończone zadania dały łącznie confirmedQty 50 i damagedQty 10
  Gdy System WMS finalizuje źródłową TU i przekazuje wynik do Inbound
  Wtedy przekazane składniki wyznaczają ilość rezydualną 40, a nie 50
```

### FR-P2-12 — Niezależne rozliczenie i CROSS_DOCKED
*Kategoria:* warunkowe · *Źródło:* P2 R23–P2 R24
**Jeśli** cała deklarowana zawartość została rozliczona przez cross-docking, System WMS musi ustawić Inbound `TU → CROSS_DOCKED` niezależnie od topologii; potwierdzoną ilość OK rozliczyć bez oczekiwania na Putaway resztki.
```gherkin
Scenariusz: Brak resztki daje CROSS_DOCKED
  Zakładając, że topologia n:n rozliczyła całą deklarowaną ilość
  Gdy zadania zostaną zakończone
  Wtedy Inbound TU ma `CROSS_DOCKED`
  Oraz rozliczenie nie zależy od liczby docelowych TU
```

### FR-P2-13 — Bramka GR dla Shipment
*Kategoria:* stanowe · *Źródło:* P2 R25–P2 R26
**Dopóki** każda źródłowa Inbound `TU` zasilająca `Shipment` nie ma `GR_ACCEPTED` dla rozliczenia cross-dockowego, System WMS musi wstrzymać zgłoszenie Shipment do ERP; późniejsze rozliczenie resztki nie może wpływać na gotowość.
```gherkin
Scenariusz: Putaway resztki nie blokuje Shipment
  Zakładając, że rozliczenie cross-dockowe ma `GR_ACCEPTED`, a resztka czeka na Putaway
  Gdy System WMS ocenia bramkę Shipment
  Wtedy źródłowa TU spełnia warunek bramki
  Oraz późniejsze rozliczenie resztki nie zmienia gotowości
```

### FR-P2-14 — Ponowna ewaluacja bramki GR i widoczność dla Supervisora
*Kategoria:* zdarzeniowe · *Źródło:* P2 R35–P2 R36
**Gdy** nadejdzie komunikat GR (`GR_ACCEPTED` albo `GR_REJECTED`) dla którejkolwiek wymaganej źródłowej Inbound `TU`, System WMS musi ponownie ewaluować bramkę `FR-P2-13`/`P2 R25` niezależnie od poprzedniego wyniku dla tej `TU` czy bieżącego stanu `Shipment`, w tym gdy `Shipment` znajduje się w `POSTING_ERROR` z innej przyczyny; jawny `GR_REJECTED` nie może samodzielnie przenieść `Shipment → POSTING_ERROR` — bramka pozostaje niespełniona do czasu `GR_ACCEPTED`. `grAcceptanceStatus` źródłowych `TU` blokujących bramkę musi być widoczny dla `Warehouse Supervisor`.
```gherkin
Scenariusz: Odrzucone GR nie przenosi Shipment w POSTING_ERROR
  Zakładając, że Shipment zależy od dwóch źródłowych TU
  Gdy jedna z nich otrzyma jawny GR_REJECTED
  Wtedy Shipment nie przechodzi w POSTING_ERROR, a bramka pozostaje niespełniona
  Oraz Warehouse Supervisor widzi grAcceptanceStatus tej źródłowej TU
```

### FR-P2-15 — Finalizacja źródłowej TU
*Kategoria:* zdarzeniowe · *Źródło:* P2 R28
**Gdy** nie pozostaje żaden aktywny `CrossDockPickTask` źródłowej Inbound `TU`, System WMS musi sfinalizować tę `TU` — `CROSS_DOCKED` przy braku ilości rezydualnej albo `IN_PUTAWAY` z resztką na `TRANSIT` — i nie może utworzyć dla niej kolejnego zadania cross-dockowego.
```gherkin
Scenariusz: Nowe zapotrzebowanie nie wznawia cross-dockingu
  Zakładając, że wszystkie zadania źródłowej TU zakończyły się i pozostała ilość rezydualna
  Gdy po finalizacji pojawi się nowa linia BACKORDERED na to SKU
  Wtedy System WMS nie tworzy nowego CrossDockPickTask dla tej TU
  Oraz zapotrzebowanie jest obsługiwane standardowo po zakończeniu Putaway
```

### FR-P2-16 — Zamek ilościowy i jednorazowe generowanie zadania
*Kategoria:* uniwersalne · *Źródło:* P2 R29–P2 R30
System WMS musi utrzymywać zamek ilościowy na `CrossDockPickTask`: ta sama fizyczna ilość źródłowej Inbound `TU`/`SKU` może być `planned` albo `confirmed` w co najwyżej jednym aktywnym zadaniu. Generowanie `CrossDockPickTask` i `OutboundOrderLine` dla danej źródłowej Inbound `TU` jest zdarzeniem jednorazowym, wyzwalanym przez jej przejście w `IN_CROSS_DOCK`, dopasowującym wyłącznie `CustomerOrderLine BACKORDERED` istniejące w tym momencie; nie istnieje mechanizm dogenerowania kolejnych zadań do już utworzonej `OutboundOrderLine` ani aktualizacji jej wymaganej ilości po utworzeniu. Każda cross-dockowa `OutboundOrderLine` powstaje razem z dokładnie jednym `CrossDockPickTask`.
```gherkin
Scenariusz: Ta sama ilość nie trafia do dwóch zadań
  Zakładając, że aktywne zadanie rezerwuje plannedQty na danym SKU źródłowej TU
  Gdy powstaje kolejne zadanie na to samo SKU tej TU
  Wtedy nie może objąć ilości już zarezerwowanej
  Oraz nie powstaje mechanizm dogenerowania kolejnych zadań do już utworzonej OutboundOrderLine
```

### FR-P2-17 — Korelacja i odrzucanie wyniku GR
*Kategoria:* zdarzeniowe · *Źródło:* P2 R31–P2 R32
**Gdy** System WMS otrzyma z ERP wynik komunikatu `Goods Receipt`, musi skorelować go wyłącznie po `sourceInboundTU` i po `GR_SETTLEMENT_SOURCE = CROSSDOCK`, bez identyfikatora zadania, i ustawić ten sam `grAcceptanceStatus` na wszystkich `CrossDockPickTask` tej Inbound `TU`; sygnał niepasujący do żadnego `sourceInboundTU` albo dotyczący innego źródła rozliczenia musi odrzucić bez zmiany `grAcceptanceStatus`.
```gherkin
Scenariusz: Rozliczenie putawayowe nie rusza bramki crossdockowej
  Zakładając, że zadania źródłowej TU mają już GR_ACCEPTED ze źródła CROSSDOCK
  Gdy przyjdzie wynik GR ze źródłem PUTAWAY dla tej samej TU
  Wtedy grAcceptanceStatus tych zadań pozostaje bez zmiany
  Oraz numer wersji rozliczenia nie jest używany jako dyskryminator źródła
```

### FR-P2-18 — Bramka wyznaczana per Shipment
*Kategoria:* stanowe · *Źródło:* P2 R33
**Dopóki** choć jedna źródłowa Inbound `TU` zasilająca dany `Shipment` nie ma `GR_ACCEPTED` rozliczenia crossdockowego, System WMS musi wstrzymywać zgłoszenie tego `Shipment`, wyznaczając warunek osobno dla każdego `Shipment` z pełnego zbioru jego własnych źródłowych `TU`.
```gherkin
Scenariusz: Jedna paleta zasila dwa Shipmenty
  Zakładając, że źródłowa TU X zasiliła zadanie Shipmentu 1 i zadanie Shipmentu 2
  Gdy TU X otrzyma GR_ACCEPTED rozliczenia crossdockowego
  Wtedy zadania obu Shipmentów mają zaktualizowany grAcceptanceStatus
  Oraz każdy Shipment sprawdza własny zbiór źródłowych TU osobno
```

### FR-P2-19 — Pusta docelowa TU po odzysku
*Kategoria:* warunkowe · *Źródło:* P2 R34
**Jeśli** docelowa Outbound `TU` utworzona w trakcie aktywnego `CrossDockPickTask` straci przez odzysk `PutBackTask` całą potwierdzoną ilość i pozostanie pusta, a jednocześnie żaden aktywny ani zaplanowany `CrossDockPickTask` nie wskazuje jej już jako celu, to System WMS musi przenieść ją `CREATED → CANCELLED` i nie może dopuścić jej do `PACKING_SEALED`. Dopóki takie zadanie ją wskazuje, System WMS musi pozostawić ją w `CREATED` jako pustą, otwartą jednostkę docelową. System WMS nie może uzależniać tego anulowania od osiągnięcia `slaDeadline`.
```gherkin
Scenariusz: Pusta docelowa TU bez wskazań jest anulowana
  Zakładając, że cała potwierdzona ilość docelowej TU wróciła przez PutBackTask
  Gdy TU pozostaje bez żadnej innej potwierdzonej pozycji i żadne aktywne ani zaplanowane zadanie nie wskazuje jej jako celu
  Wtedy przechodzi do CANCELLED, nie czekając na slaDeadline
  Oraz nigdy nie wchodzi w PACKING_SEALED

Scenariusz: Pusta docelowa TU ze wskazaniem aktywnego zadania nie jest anulowana
  Zakładając, że cała potwierdzona ilość docelowej TU wróciła przez PutBackTask
  Gdy inny aktywny CrossDockPickTask nadal wskazuje tę TU jako cel
  Wtedy TU pozostaje w CREATED jako pusta, otwarta jednostka docelowa
```

### FR-P2-21 — Przydział CrossDockPickTask przez wybór modułu
*Kategoria:* zdarzeniowe · *Źródło:* P2 R39
**Gdy** `Warehouse Operator` jest zalogowany do modułu crossdockingu i nie ma aktywnego zadania magazynowego żadnego typu, System WMS musi przydzielić mu kolejny `CrossDockPickTask` zgodnie ze skonfigurowaną kolejnością magazynu.
```gherkin
Scenariusz: Operator bez zadania dostaje kolejny CrossDockPickTask
  Zakładając, że operator jest zalogowany do modułu crossdockingu i nie ma aktywnego zadania
  Gdy System WMS wybiera kolejne zadanie do przydziału
  Wtedy operator otrzymuje najwyższy w kolejce CrossDockPickTask
```

### FR-P2-22 — Ciągłość celu aktywnego CrossDockPickTask
*Kategoria:* uniwersalne · *Źródło:* P2 R40
System WMS musi zapewnić, że aktywny `CrossDockPickTask` zawsze ma dostępny cel, do którego operator może odłożyć potwierdzoną ilość; jeżeli w momencie odłożenia kolejnej pozycji nie istnieje otwarta docelowa Outbound `TU` tego zadania, System WMS musi otworzyć nową Outbound `TU` i nadać jej `TU_NUMBER`, niezależnie od przyczyny braku poprzedniej.
```gherkin
Scenariusz: Brak otwartej TU przy odkładaniu otwiera nową
  Zakładając, że aktywny CrossDockPickTask nie ma otwartej docelowej Outbound TU, bo poprzednia została zamknięta jako fizycznie pełna albo anulowana jako pusta
  Gdy Packer odkłada kolejną pozycję SKU w ramach tego zadania
  Wtedy System WMS otwiera nową Outbound TU i nadaje jej TU_NUMBER
  Oraz utworzenie następuje dopiero przy tym odłożeniu, nie z wyprzedzeniem przy zamknięciu poprzedniej TU
```

### FR-P2-23 — Wyłącznie TU ELEMENTARY w cross-dockingu
*Kategoria:* uniwersalne · *Źródło:* P2 R41
System WMS musi przyjmować do cross-dockingu wyłącznie Inbound `TU` typu `ELEMENTARY`; `TU` `AGGREGATE` nie może osiągnąć `IN_CROSS_DOCK`, a gdy `SKU` potrzebne do linii `BACKORDERED` jest zadeklarowane w `TU` przyczepionej do zbiorczej, do procesu trafia wyłącznie `TU` `ELEMENTARY`.
```gherkin
Scenariusz: Do cross-dockingu trafia dziecko, nie zbiorcza
  Zakładając, że SKU potrzebne do linii BACKORDERED jest zadeklarowane w TU ELEMENTARY przyczepionej do TU AGGREGATE
  Gdy Inbound kwalifikuje ten towar do cross-dockingu
  Wtedy do procesu Outbound Crossdock trafia TU ELEMENTARY
  Oraz TU AGGREGATE nie osiąga IN_CROSS_DOCK
```

### FR-P2-24 — Zerowe dopasowanie przy IN_CROSS_DOCK
*Kategoria:* warunkowe · *Źródło:* P2 R42
**Jeśli** przy przejściu źródłowej Inbound `TU` w `IN_CROSS_DOCK` nie istnieje żadna pasująca `CustomerOrderLine` w `BACKORDERED`, to System WMS nie może utworzyć żadnego `CrossDockPickTask` i musi zafinalizować tę `TU` z całą zadeklarowaną ilością jako rezydualną, przenosząc ją na `TRANSIT` i do `IN_PUTAWAY`.
```gherkin
Scenariusz: Popyt zniknął w czasie transportu do strefy cross-dock
  Zakładając, że Inbound zakwalifikował TU do cross-dockingu, a przed jej przejściem w IN_CROSS_DOCK wszystkie pasujące CustomerOrderLine przestały być BACKORDERED
  Gdy TU osiąga IN_CROSS_DOCK
  Wtedy System WMS nie tworzy żadnego CrossDockPickTask
  Oraz cała zadeklarowana ilość TU jest ilością rezydualną
  Oraz TU zostaje przekazana do Putaway z pełną ilością rezydualną
```

### FR-P2-25 — Dziedziczenie priority i slaDeadline w cross-dockingu
*Kategoria:* zdarzeniowe · *Źródło:* P2 R43

**Gdy** System WMS tworzy crossdockowy `OutboundOrder`, musi odziedziczyć jego `priority` i `slaDeadline` z macierzystego `CustomerOrder` oraz grupować w nim linie wyłącznie przy zgodnym kliencie, adresie dostawy, `priority` i identycznym `slaDeadline`; przy `allowPartialShipment = false` grupowanie musi obejmować wyłącznie linie jednego `CustomerOrder`.
```gherkin
Scenariusz: Dziedziczenie parametrów crossdockowego OutboundOrder
  Zakładając, że macierzysty CustomerOrder ma priority 2 i slaDeadline 2026-08-28T16:00:00
  Gdy System WMS tworzy dla niego crossdockowy OutboundOrder
  Wtedy OutboundOrder otrzymuje priority 2 i slaDeadline 2026-08-28T16:00:00

Scenariusz: Grupowanie wymaga identycznego terminu
  Zakładając, że dwie linie mają zgodnego klienta, adres i priority, ale różne slaDeadline
  Gdy System WMS grupuje linie w crossdockowy OutboundOrder
  Wtedy linie nie trafiają do jednego OutboundOrder
```

## P3 — Reservation Release

### FR-P3-01 — Zwolnienie Allocation i Inventory
*Kategoria:* zdarzeniowe · *Źródło:* P3 R1–P3 R2
**Gdy** rezerwacja zostaje zwolniona przed fizycznym pobraniem, System WMS musi wykonać `Allocation RESERVED → RELEASED` i `Inventory RESERVED → AVAILABLE`.
```gherkin
Scenariusz: Zwolnienie bez pracy fizycznej
  Zakładając, że towar jest zarezerwowany, lecz niepobrany
  Gdy System WMS zwolni rezerwację
  Wtedy Allocation ma `RELEASED`
  Oraz Inventory ma `AVAILABLE`
```

### FR-P3-02 — Anulowanie linii przy niedoborze
*Kategoria:* warunkowe · *Źródło:* P3 R3–P3 R4
**Jeśli** zwolnienie wynika z `SHORT_ALLOCATED`, System WMS musi anulować `OutboundOrderLine` w stanie przed pobraniem i ustawić niezrealizowaną część `CustomerOrderLine → BACKORDERED`.
```gherkin
Scenariusz: Niedobór zachowuje demand
  Zakładając, że Allocation ma niedobór przed pobraniem
  Gdy rezerwacja zostanie zwolniona
  Wtedy OutboundOrderLine ma `CANCELLED`
  Oraz CustomerOrderLine ma `BACKORDERED`
```

### FR-P3-03 — Anulowanie ogólne i auto-release
*Kategoria:* konfiguracyjne · *Źródło:* P3 R5–P3 R6
**Gdy** przyczyną zwolnienia jest anulowanie ogólne, System WMS musi ustawić `CustomerOrderLine → CANCELLED`; auto-zwolnienie według polityki musi wykonać bez powiadomienia Supervisora.
```gherkin
Scenariusz: Auto-release bez eskalacji
  Zakładając, że polityka magazynu nakazuje auto-zwolnienie
  Gdy spełni się warunek polityki
  Wtedy rezerwacja jest zwolniona, a właściwa linia anulowana
  Oraz Supervisor nie otrzymuje obowiązkowej eskalacji
```

### FR-P3-04 — Okno wyścigu i przekazanie do P4
*Kategoria:* warunkowe · *Źródło:* P3 R7–P3 R8
**Jeśli** zwolnienie nastąpi po fizycznym pobraniu, lecz przed jego potwierdzeniem, System WMS musi anulować `PickTask` i polecić zwrot na lokalizację źródłową bez `PutBackTask`; formalnie potwierdzoną ilość musi skierować do P4.
```gherkin
Scenariusz: Zwrot na źródło bez PutBackTask
  Zakładając, że operator fizycznie pobrał SKU, lecz nie potwierdził TU i ilości
  Gdy Allocation zostanie zwolniona
  Wtedy PickTask jest anulowany, a RF nakazuje zwrot na źródło
  Oraz formalny PutBackTask nie powstaje
```

### FR-P3-05 — Warianty polityki utrzymywania rezerwacji
*Kategoria:* konfiguracyjne · *Źródło:* P3 R9

**Tam gdzie** magazyn konfiguruje politykę utrzymywania częściowej rezerwacji, System WMS musi udostępnić trzy warianty: utrzymanie rezerwacji, automatyczne zwolnienie po określonym czasie oraz skierowanie sprawy do decyzji `Warehouse Supervisor`.
```gherkin
Scenariusz: Wariant automatycznego zwolnienia
  Zakładając, że magazyn skonfigurował automatyczne zwolnienie po czasie
  Gdy czas utrzymania rezerwacji upływa
  Wtedy System WMS zwalnia Allocation bez udziału Warehouse Supervisor
```

### FR-P3-06 — Niezależność czasu utrzymania rezerwacji
*Kategoria:* stanowe · *Źródło:* P3 R10

**Dopóki** trwa utrzymanie częściowej rezerwacji, System WMS nie może uzależniać jej czasu od `priority` ani od `slaDeadline` `CustomerOrder`.
```gherkin
Scenariusz: Priorytet nie skraca ani nie wydłuża czasu
  Zakładając, że dwa CustomerOrder mają różny priority i różny slaDeadline
  Gdy oba mają częściową rezerwację przy tej samej polityce magazynu
  Wtedy czas utrzymania rezerwacji jest dla obu identyczny
```

## P4 — Physical Put-back

### FR-P4-01 — Zgoda i granica POSTING_PENDING
*Kategoria:* warunkowe · *Źródło:* P4 R1–P4 R2
**Jeśli** anulowana linia jest `PACKED`, System WMS musi wymagać zgody `Warehouse Supervisor`; anulowanie musi dopuścić wyłącznie przed `Shipment POSTING_PENDING`.
```gherkin
Scenariusz: PACKED po POSTING_PENDING nie podlega P4
  Zakładając, że linia jest PACKED, a Shipment ma `POSTING_PENDING`
  Gdy Supervisor próbuje zatwierdzić anulowanie
  Wtedy System WMS odrzuca operację
  Oraz wskazuje Return Receipt jako dalszą ścieżkę
```

### FR-P4-02 — Natychmiastowy skutek logiczny
*Kategoria:* zdarzeniowe · *Źródło:* P4 R3–P4 R4
**Gdy** anulowanie zostanie zatwierdzone, System WMS musi natychmiast anulować linię, zwolnić `Allocation` i zwrócić ilość do `ATPReservation`, nie czekając na fizyczne odłożenie.
```gherkin
Scenariusz: Statusy nie czekają na operatora
  Zakładając, że anulowanie pobranej ilości zostało zatwierdzone
  Gdy PutBackTask pozostaje niezakończony
  Wtedy OutboundOrderLine ma `CANCELLED`, a Allocation `RELEASED`
  Oraz fizyczny Inventory nadal czeka na odłożenie
```

### FR-P4-03 — PutBackTask i walidacja lokalizacji
*Kategoria:* zdarzeniowe · *Źródło:* P4 R5–P4 R6
**Gdy** anulowana pozycja ma `pickedQty > 0`, System WMS musi utworzyć osobny `PutBackTask`; wskazaną lub proponowaną lokalizację musi zwalidować przed odłożeniem.
```gherkin
Scenariusz: Pobrana ilość tworzy zadanie
  Zakładając, że anulowana linia ma dodatnie `pickedQty`
  Gdy System WMS rozlicza anulowanie
  Wtedy tworzy PutBackTask dla pobranej ilości
  Oraz nie pozwala odłożyć do niezwalidowanej lokalizacji
```

### FR-P4-04 — Pętla lokalizacji i odzysk Inventory
*Kategoria:* stanowe · *Źródło:* P4 R7–P4 R8
**Dopóki** lokalizacja jest odrzucana, System WMS musi utrzymywać pętlę `IN_PROGRESS ↔ LOCATION_VALIDATION` bez limitu i eskalacji; po `PutBackTask COMPLETED` musi wykonać `Inventory PICKED → AVAILABLE`.
```gherkin
Scenariusz: Kolejna próba kończy PutBack
  Zakładając, że pierwsza lokalizacja została odrzucona
  Gdy operator wskaże kolejną poprawną lokalizację i odłoży towar
  Wtedy PutBackTask zostaje zakończony
  Oraz Inventory przechodzi do `AVAILABLE`
```

### FR-P4-05 — Przydział PutBackTask przez wybór modułu
*Kategoria:* zdarzeniowe · *Źródło:* P4 R9
**Gdy** `Warehouse Operator` jest zalogowany do modułu zwrotów i nie ma aktywnego zadania magazynowego żadnego typu, System WMS musi przydzielić mu kolejny `PutBackTask` w kolejności zgłoszenia.
```gherkin
Scenariusz: Operator bez zadania dostaje kolejny PutBackTask
  Zakładając, że operator jest zalogowany do modułu zwrotów i nie ma aktywnego zadania
  Gdy System WMS wybiera kolejne zadanie do przydziału
  Wtedy operator otrzymuje najstarszy zgłoszony PutBackTask
```

## P5 — Wyjątki przekrojowe

### FR-P5-01 — SHORT_ALLOCATED bez zgody na część
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = false`
**Jeśli** wyścig o miękką rezerwację spowoduje `SHORT_ALLOCATED` przy `allowPartialShipment = false`, System WMS musi eskalować do `Warehouse Supervisor` wybór trwałej zgody na część albo anulowania `OutboundOrder`, a przegraną linię pozostawić w `BACKORDERED`.
```gherkin
Scenariusz: Supervisor rozstrzyga SHORT_ALLOCATED
  Zakładając, że linia przegrała wyścig o część rezerwacji
  Gdy Supervisor wybierze anulowanie OutboundOrder
  Wtedy jego Allocation są zwolnione, a przegrana linia ma BACKORDERED
  Oraz CustomerOrder wraca do oczekiwania zgodnie ze stanem pozostałych linii
```

### FR-P5-02 — SHORT_ALLOCATED ze zgodą na część
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = true`
**Jeśli** wystąpi `SHORT_ALLOCATED` przy `allowPartialShipment = true`, System WMS musi realizować dostępny `OutboundOrder`, a brak pozostawić w `BACKORDERED` do pojawienia się zapasu.
```gherkin
Scenariusz: Część jest realizowana bez eskalacji
  Zakładając, że allowPartialShipment = true
  Gdy Allocation pokrywa tylko część ilości
  Wtedy istniejący OutboundOrder realizuje dostępną część
  Oraz brak pozostaje BACKORDERED
```

### FR-P5-03 — Automatyczne ponowienie po SHORT_PICKED
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — `SHORT_PICKED`
**Gdy** `PickTask` zakończy się w `SHORT_PICKED`, System WMS musi zablokować lokalizację i tworzyć nowy `PickTask` dla brakującej ilości, dopóki istnieje kwalifikowany zapas ATP w innej lokalizacji i nie wyczerpano efektywnego limitu; w przeciwnym razie musi eskalować.
```gherkin
Scenariusz: Nowy PickTask korzysta z innej lokalizacji
  Zakładając, że pierwszy PickTask ma SHORT_PICKED
  Gdy istnieje ATP w niezablokowanej lokalizacji i limit nie został wyczerpany
  Wtedy WMS tworzy nowy PickTask dla brakującej ilości
  Oraz OutboundOrder nie przechodzi jeszcze do pakowania
```

### FR-P5-04 — Wynik „czekamy” po SHORT_PICKED
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „czekamy”
**Jeśli** Supervisor wybierze wynik „czekamy”, System WMS musi cofnąć niepakowane linie tego `CustomerOrder`, zwolnić ich alokacje i zachować już `PACKED` linie bez zmian.
```gherkin
Scenariusz: Czekanie cofa tylko niepakowane linie
  Zakładając, że część linii jest PACKED, a część ALLOCATED lub PICKED
  Gdy Supervisor wybierze „czekamy”
  Wtedy niepakowane linie są cofnięte i ich Allocation zwolnione
  Oraz linie PACKED pozostają bez zmian
```

### FR-P5-05 — Wynik „anulowanie” po SHORT_PICKED
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „anulowanie”
**Jeśli** Supervisor wybierze anulowanie, System WMS musi wymagać edycji ilości `CustomerOrderLine`: do zera z anulowaniem i zwrotem albo do ilości faktycznie pobranej bez `PutBackTask`.
```gherkin
Scenariusz: Sama anulowana linia wykonawcza nie wystarcza
  Zakładając, że allowPartialShipment = false i wystąpił SHORT_PICKED
  Gdy Supervisor wybierze anulowanie
  Wtedy musi wskazać nową ilość CustomerOrderLine
  Oraz WMS dobiera zwrot według tej ilości
```

### FR-P5-06 — Trwała zgoda na część po SHORT_PICKED
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — trwała zmiana `allowPartialShipment`
**Jeśli** Supervisor trwale ustawi `allowPartialShipment = true`, System WMS musi wymagać powodu, anulować brakującą część i ograniczyć skutek do bieżącego `CustomerOrder`.
```gherkin
Scenariusz: Trwała zmiana ma ograniczony zakres
  Zakładając, że Supervisor podał powód
  Gdy zatwierdzi allowPartialShipment = true
  Wtedy dostępna ilość jedzie dalej, a brakująca część jest anulowana
  Oraz inne zamówienia klienta i parametry magazynu nie zmieniają się
```

### FR-P5-07 — Rozbieżność podczas pakowania
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — brak lub uszkodzenie przy „repack by SKU”
**Gdy** podczas pakowania zostanie wykryty brak albo uszkodzenie, System WMS musi uruchomić ten sam mechanizm co dla `SHORT_PICKED`, ale bez blokowania lokalizacji źródłowej.
```gherkin
Scenariusz: Brak przy pakowaniu używa obsługi SHORT_PICKED
  Zakładając, że operator liczy SKU podczas repack by SKU
  Gdy wykryje brak lub uszkodzenie
  Wtedy WMS uruchamia rozstrzygnięcie SHORT_PICKED
  Oraz nie blokuje lokalizacji źródłowej
```

### FR-P5-08 — ON_HOLD
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — `CustomerOrder.ON_HOLD`
**Dopóki** `CustomerOrder` ma `ON_HOLD`, System WMS musi pomijać go w cyklicznym planowaniu; po zdjęciu blokady musi zwrócić go do kolejki.
```gherkin
Scenariusz: Zdjęcie blokady przywraca planowanie
  Zakładając, że CustomerOrder ma ON_HOLD
  Gdy blokada zostanie zdjęta
  Wtedy zamówienie wraca do kolejki planowania
```

### FR-P5-09 — Anulowanie ogólne
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — żądanie anulowania `CustomerOrder` lub `CustomerOrderLine`
**Gdy** OMS/ERP albo `Warehouse Supervisor` zażąda anulowania, System WMS musi ocenić każdą objętą linię według statusu `OutboundOrderLine`, odrzucić anulowanie całości przy choć jednej blokującej linii i nie dopuścić anulowania po `CarrierManifest.CLOSED`.
```gherkin
Scenariusz: Jedna blokująca linia odrzuca anulowanie całości
  Zakładając, że CustomerOrder ma co najmniej jedną linię niespełniającą warunku anulowania
  Gdy przychodzi żądanie anulowania całego zamówienia
  Wtedy WMS odrzuca żądanie i wskazuje blokującą linię
```

### FR-P5-10 — Brak wyniku Carrier Selection
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — brak wyniku Carrier Selection
**Jeśli** Carrier Selection nie zwróci wyniku, System WMS musi eskalować do `Warehouse Supervisor`; dla `TU` na typie o roli nośnika zewnętrznego (`P1 R68`), którego `TUSetup.maxVolume` nie ma wartości, musi wymagać ręcznego wyboru przez `Dispatcher` i zatwierdzenia przed etykietą.
```gherkin
Scenariusz: Ręczny wybór poprzedza etykietę
  Zakładając, że `TU` jest na typie o roli nośnika zewnętrznego (`P1 R68`), którego `TUSetup.maxVolume` nie ma wartości, i brak wyniku Carrier Selection
  Gdy Dispatcher wybierze przewoźnika, a Supervisor go zatwierdzi
  Wtedy WMS może wygenerować etykietę
```

### FR-P5-11 — Ograniczenie obsługi błędu etykiety
*Kategoria:* warunkowe · *Źródło:* P1 Wyjątki — błąd etykiety lub odrzucenie przez przewoźnika
**Jeśli** zgłoszono błąd etykiety albo elektroniczne odrzucenie, System WMS w wersji 1 nie może uruchamiać osobnego trybu awarii; problem z załadunkiem przed `CarrierManifest.CLOSED` musi obsłużyć ręczną zmianą `Carrier` bez ponownego druku etykiety.
```gherkin
Scenariusz: Wersja 1 nie ma elektronicznego odrzucenia
  Zakładając, że problem z załadunkiem wystąpił przed zamknięciem manifestu
  Gdy Supervisor ręcznie zmieni Carrier
  Wtedy WMS nie drukuje ponownie etykiety
```

### FR-P5-12 — Niedobór cross-dock bez zgody na część
*Kategoria:* warunkowe · *Źródło:* P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = false`
**Jeśli** podczas pobrania cross-dock wystąpi niedobór albo `DAMAGED` przy `allowPartialShipment = false`, System WMS musi eskalować trzy wyniki: czekanie z `PutBackTask`, anulowanie lub trwałą zgodę na część.
```gherkin
Scenariusz: Supervisor wybiera jedną z trzech ścieżek
  Zakładając, że confirmedQty jest mniejsze od zapotrzebowania
  Gdy Supervisor wybierze „czekamy”
  Wtedy WMS tworzy PutBackTask dla confirmedQty większego od zera
```

### FR-P5-13 — Niedobór cross-dock ze zgodą na część
*Kategoria:* warunkowe · *Źródło:* P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = true`
**Jeśli** podczas pobrania cross-dock wystąpi niedobór albo `DAMAGED` przy `allowPartialShipment = true`, System WMS musi automatycznie przekazać rozsortowaną część do `PACKED`, a brak pozostawić w `BACKORDERED`.
```gherkin
Scenariusz: Częściowy cross-dock bez eskalacji
  Zakładając, że allowPartialShipment = true
  Gdy pobranie kończy się niedoborem
  Wtedy rozsortowana część ma PACKED, a brakująca BACKORDERED
```

### FR-P5-14 — Puste TU przed pobraniem
*Kategoria:* warunkowe · *Źródło:* P2 Wyjątki — puste `TU`
**Gdy** źródłowe `TU` okaże się puste przed pobraniem, System WMS musi anulować `CrossDockPickTask` i `OutboundOrderLine`, ustawić `CustomerOrderLine BACKORDERED` oraz Inbound `TU LOST`.
```gherkin
Scenariusz: Puste TU anuluje zadanie
  Zakładając, że TU jest puste przed pobraniem
  Gdy operator potwierdzi ten stan
  Wtedy CrossDockPickTask i OutboundOrderLine mają CANCELLED
  Oraz CustomerOrderLine ma BACKORDERED, a Inbound TU ma LOST
```

### FR-P5-15 — Anulowanie podczas sortowania
*Kategoria:* warunkowe · *Źródło:* P2 Wyjątki — ogólne anulowanie przy `CrossDockPickTask IN_PROGRESS`
**Jeśli** ogólne anulowanie nadejdzie podczas `CrossDockPickTask IN_PROGRESS`, System WMS musi odrzucić żądanie i dopuścić ponowienie dopiero po osiągnięciu `PACKED`.
```gherkin
Scenariusz: Anulowanie jest odrzucone podczas sortowania
  Zakładając, że CrossDockPickTask ma IN_PROGRESS
  Gdy przychodzi ogólne anulowanie
  Wtedy WMS odrzuca żądanie ze wskazaniem przyczyny
```

### FR-P5-16 — Ponowna ewaluacja bramki Goods Receipt
*Kategoria:* warunkowe · *Źródło:* P2 Wyjątki — `P2 R35`–`P2 R36`
**Gdy** dowolna wymagana źródłowa Inbound `TU` otrzyma wynik GR (`GR_ACCEPTED` albo `GR_REJECTED`), System WMS musi ponownie ewaluować bramkę zgłoszenia `Shipment` do ERP; jawny `GR_REJECTED` nie może samodzielnie przenieść `Shipment` do `POSTING_ERROR` — bramka pozostaje niespełniona do czasu `GR_ACCEPTED`, a `grAcceptanceStatus` musi być widoczny dla `Warehouse Supervisor`.
```gherkin
Scenariusz: GR_REJECTED nie księguje Shipment do POSTING_ERROR
  Zakładając, że źródłowa TU zasila Shipment
  Gdy jej wynik ma GR_REJECTED
  Wtedy Shipment nie przechodzi do POSTING_ERROR, a bramka pozostaje niespełniona
```

### FR-P5-17 — Oczekiwanie skompletowanego fragmentu na komplet zamówienia
*Kategoria:* stanowe · *Źródło:* P1 Wyjątki — P5 E17
**Dopóki** `CustomerOrder` z `allowPartialShipment = false` ma część `CustomerOrderLine`, która nie spełnia ani alternatywy (A), ani (B) z `P1 R58`, a inne linie tego samego `CustomerOrder` mają już `OutboundOrderLine` w `PACKED`, System WMS musi pozostawić Packing `TU` linii spakowanych w `PACKING_SEALED` w strefie konsolidacji, poza każdym `Shipment`, bez anulowania `OutboundOrder` i bez automatycznego uwolnienia linii pozostałych do osobnej realizacji. Oczekiwanie może zakończyć wyłącznie skompletowanie zamówienia albo istniejąca ścieżka `Warehouse Supervisor` z `P1 R46` lub `P1 R47`.
```gherkin
Scenariusz: Skompletowany fragment czeka przed utworzeniem Shipment
  Zakładając, że CustomerOrder ma allowPartialShipment = false
  Oraz część CustomerOrderLine nie spełnia ani alternatywy A, ani B z P1 R58
  Oraz inne linie tego CustomerOrder mają OutboundOrderLine w PACKED
  Gdy System WMS ocenia możliwość utworzenia albo uzupełnienia Shipment
  Wtedy Packing TU linii spakowanych pozostają w PACKING_SEALED poza każdym Shipment
  Oraz OutboundOrder nie jest anulowany
  Oraz linie pozostałe nie są automatycznie uwalniane do osobnej realizacji
```

# 1b. Wymagania integracyjne

### INT-01 — Granica Inbound dla cross-dock
*Źródło:* P2 KROK 1
**Gdy** Inbound udostępni ilość ze statusem `IN_CROSS_DOCK`, System WMS musi przyjąć identyfikator `TU`, `SKU`, ilość zadeklarowaną w ASN — niezweryfikowaną fizycznie po stronie Inbound — oraz korelację przyjęcia.
```gherkin
Scenariusz: Baza ilościowa pochodzi z deklaracji ASN
  Zakładając, że Inbound potwierdził IN_CROSS_DOCK bez otwierania TU
  Gdy komunikat trafia do Outbound
  Wtedy ilością bazową jest ilość zadeklarowana w ASN
  Oraz P2 otrzymuje dane TU, SKU i korelację
```

### INT-02 — Wynik cross-dock do Inbound
*Źródło:* P2 KROK 3
**Gdy** rozliczenie crossdockowe źródłowej Inbound `TU` zostanie zamknięte, System WMS musi przekazać Inbound `confirmedQty` i `damagedQty` tej `TU` z zachowaniem korelacji; ilości rezydualnej nie przekazuje jako osobnego pola kontraktu — wylicza ją Inbound.
```gherkin
Scenariusz: Kontrakt obejmuje dwie ilości
  Zakładając, że rozliczenie crossdockowe TU zostało zamknięte
  Gdy P2 publikuje wynik
  Wtedy Inbound otrzymuje confirmedQty i damagedQty z korelacją TU
  Oraz resztę wylicza po swojej stronie jako deklaracja minus confirmedQty minus damagedQty
```

### INT-03 — Korelacja Goods Receipt
*Źródło:* P2 KROK 4
**Dopóki** wynik zależnego `Goods Receipt` nie dotrze z ERP, System WMS musi utrzymywać korelację po `sourceInboundTU` i `GR_SETTLEMENT_SOURCE`; ponawianie samego komunikatu `Goods Receipt` należy do Inbound i nie jest odpowiedzialnością Outbound.
```gherkin
Scenariusz: Korelacja trwa do wyniku
  Zakładając, że rozliczenie crossdockowe zostało przekazane Inbound
  Gdy wynik z ERP jeszcze nie dotarł
  Wtedy korelacja po sourceInboundTU pozostaje aktywna
  Oraz Outbound nie ponawia komunikatu Goods Receipt
```

### INT-04 — Shipment POST do ERP
*Źródło:* P1 KROK 13
**Gdy** wysyłka spełnia warunki publikacji, System WMS musi wysłać do ERP komunikat `Shipment POST` z jednoznacznym identyfikatorem.
```gherkin
Scenariusz: Publikacja wysyłki
  Zakładając, że Shipment jest gotowy do księgowania
  Gdy System WMS publikuje Shipment POST
  Wtedy ERP otrzymuje jednoznacznie identyfikowalny komunikat
```

### INT-05 — Odpowiedź ERP i ponowienie
*Źródło:* P1 KROK 13
**Jeśli** ERP odrzuci `Shipment POST`, System WMS musi ustawić `POSTING_ERROR`, zachować dane błędu i umożliwić bezpieczne ponowienie; po akceptacji musi ustawić `POSTED`.
```gherkin
Scenariusz: Odrzucenie i skuteczne ponowienie
  Zakładając, że Shipment ma POSTING_PENDING
  Gdy ERP najpierw odrzuci, a następnie zaakceptuje ponowienie
  Wtedy Shipment przechodzi przez POSTING_ERROR do POSTED
```

### INT-06 — Anulowanie z systemu zamówień
*Źródło:* P3 KROK 1, P4 KROK 1
**Gdy** system zamówień przekaże anulowanie lub korektę, System WMS musi skorelować żądanie z właściwą linią i wybrać P3 albo P4 według formalnie potwierdzonego `pickedQty`.
```gherkin
Scenariusz: Integracyjne anulowanie wybiera proces
  Zakładając, że przyszło żądanie dla istniejącej linii
  Gdy System WMS odczyta pickedQty
  Wtedy kieruje zero do P3, a wartość dodatnią do P4
```

# 1c. Wymagania niefunkcjonalne

Nie dotyczy w zakresie B6.

# 1d. Wymagania współbieżności

### CON-01 — Atomowa rezerwacja ATP
*Źródło:* P1 R4–P1 R6, P1 R52
**Gdy** równoległe planowania konkurują o ten sam zapas ATP, System WMS musi atomowo utworzyć najwyżej jedną skuteczną `ATPReservation` i odpowiadającą jej `Allocation` dla danej ilości.
```gherkin
Scenariusz: Dwie alokacje nie rezerwują tej samej ilości
  Zakładając, że dwa zlecenia równolegle żądają tego samego zapasu
  Gdy oba wykonują alokację
  Wtedy suma skutecznych rezerwacji nie przekracza dostępnego ATP
```

### CON-02 — Niezmienność po utworzeniu PickTask
*Źródło:* P1 R10
**Dopóki** istnieje `PickTask`, System WMS nie może realokować przypisanej do niego ilości.
```gherkin
Scenariusz: Równoległa próba realokacji jest odrzucona
  Zakładając, że PickTask już istnieje
  Gdy inny przebieg próbuje realokować tę ilość
  Wtedy przypisanie pozostaje bez zmian
```

### CON-03 — Jednokrotne przypisanie cross-dock
*Źródło:* P2 R6, P2 R29–P2 R30
**Gdy** wiele linii równolegle oczekuje na ten sam `SKU` cross-dock, System WMS musi przypisać każdą jednostkę ilości najwyżej raz zgodnie z priorytetem, wyznaczając `sourceEligibleQty` z odjęciem `plannedQty` aktywnych oraz `confirmedQty` i `damagedQty` zakończonych `CrossDockPickTask`.
```gherkin
Scenariusz: Uszkodzona ilość nie jest ponownie planowana
  Zakładając, że deklaracja TU wynosi 100, w toku jest 30, a zakończone zadania dały 40 confirmedQty i 10 damagedQty
  Gdy dwa przebiegi równolegle próbują zaplanować kolejne zadanie
  Wtedy łącznie mogą objąć najwyżej 20
  Oraz żadna jednostka ilości nie trafia do dwóch zadań
```

### CON-04 — Stabilne grupowanie Shipment
*Źródło:* P1 R37–P1 R41
**Gdy** wiele paczek równolegle osiąga granicę grupowania, System WMS musi zastosować spójny termin graniczny i przypisać każdą paczkę do najwyżej jednego `Shipment`.
```gherkin
Scenariusz: Paczka należy do jednego Shipment
  Zakładając, że paczki są zamykane wokół terminu granicznego
  Gdy wykonywane jest grupowanie
  Wtedy każda paczka trafia do najwyżej jednego Shipment
```

### CON-05 — Jednokrotne skutki ERP i manifestu
*Źródło:* P1 R43–P1 R46 
**Gdy** odpowiedź ERP albo zamknięcie manifestu zostanie dostarczone ponownie lub równolegle, System WMS musi zachować jednokrotny skutek biznesowy i nie może cofnąć stanu końcowego.
```gherkin
Scenariusz: Duplikat odpowiedzi nie dubluje skutku
  Zakładając, że odpowiedź została już rozliczona
  Gdy ten sam komunikat przychodzi ponownie
  Wtedy stan biznesowy pozostaje bez zmian
```
