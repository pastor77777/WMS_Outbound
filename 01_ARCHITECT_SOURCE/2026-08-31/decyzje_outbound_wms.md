# Decyzje projektowe — procesy Outbound WMS

## 1. Cel i status dokumentu

Dokument podsumowuje decyzje podjęte podczas przygotowania procesów Outbound (Order Fulfillment). Stanowi źródło nadrzędne wobec wcześniejszych, sprzecznych zapisów w `../Archiwum/zlecenie_procesy_outbound_wms.md` (wycofany 2026-07-18).

**Status produktowy:** odkrywanie.

Na tym etapie decyzje służą do przygotowania pierwszego dokumentu analitycznego. Nie są jeszcze kanoniczną dokumentacją procesową ani ostatecznym modelem stanów.

## 2. Zasady dokumentacji i źródła prawdy

1. Wytyczne z `AGENTS.md` dla Inbound stosujemy analogicznie do Outbound.
2. Dla Outbound powstanie osobny `model_stanow_outbound.md`.
3. Nazwy obiektów, atrybutów i statusów zapisujemy po angielsku. Opisy i uzasadnienia — po polsku.
4. Pierwszy rezultat ma być jednym dokumentem analitycznym `.md`, zawierającym diagram procesu, maszyny stanów oraz diagramy sekwencji w Mermaid.
5. Dopiero po zatwierdzeniu dokumentu analitycznego treść zostanie rozdzielona na pliki docelowe zgodne z `AGENTS.md` i `SZABLON_PROCESU.md`.
6. Brakujący fakt lub nierozstrzygnięta sprzeczność ma zostać oznaczona jako otwarta kwestia. Nie wolno jej uzupełniać domysłem.
7. `PickWave` jest całkowicie poza zakresem wersji 1. Nie należy projektować jego procesu ani maszyny stanów.
8. Cross-docking paczkowy wchodzi do wersji 1 jako osobny proces.
9. **DEC-A09 — granice projektowania:** architektura techniczna, usługi, technologie integracyjne, planowanie tras, fakturowanie i zwroty od klienta pozostają poza zakresem projektowania procesowego Outbound.
10. **DEC-A10 — pojedyncze źródło decyzji:** jedynym kanonicznym rejestrem decyzji produktowych Outbound jest `decyzje_outbound_wms.md`. Prompty, handover, pamięć i dokumenty analityczne nie ustanawiają ani nie nadpisują decyzji.
11. **DEC-A11 — HISTORYCZNE, zastąpione zasadą aktualnej prozy procesu:** wpis zachowuje dawną hierarchię „decision log przed procesem". Aktualnie źródłem bieżącego zachowania jest proza procesu w plikach `proces_1_standard_fulfillment.md`–`proces_4_physical_putback.md`, a niniejszy plik zachowuje historię i uzasadnienie. Rezultat każdej decyzji musi zostać naniesiony do procesu.
12. **DEC-A12 — prompty tymczasowe:** po wykonaniu zadania istotne treści promptu należy sklasyfikować jako decyzje, pytania otwarte albo zasady pracy i przenieść do właściwych plików. Następnie prompt należy usunąć albo zarchiwizować. Prompt nie jest źródłem prawdy.
13. **DEC-A13 — `PickWave`:** w wersji 1 nie projektuje się procesu, stanów ani technicznego punktu rozszerzenia dla przyszłego `PickWave`.
14. **DEC-A14 — archiwizacja dokumentu analitycznego:** `propozycja_procesow_outbound.md` przestał być źródłem projektu i został przeniesiony do `../Archiwum/` (2026-08-22, `BACKLOG.md` B16). Źródłem prawdy o aktualnym zachowaniu są pliki `proces_1_standard_fulfillment.md`–`proces_4_physical_putback.md`, kanonem stanów `model_stanow_outbound.md`, a przekrojowym widokiem odpowiedzialności `macierz_odpowiedzialnosci_outbound.md`. Żaden aktywny artefakt nie może wymagać otwarcia zarchiwizowanego dokumentu, żeby zrozumieć aktualne zachowanie.
15. **DEC-A15 — brak osobnego pliku dla PROCESU 5:** wyjątki przekrojowe nie mają własnego pliku `proces_5`. Ich opis żyje w sekcjach „Wyjątki i ścieżki alternatywne" plików `proces_1_standard_fulfillment.md` i `proces_2_outbound_crossdock.md` (2026-08-22). Powód: PROCES 5 nie ma własnych numerowanych kroków, więc osobny plik byłby pustym pseudo-procesem.

## 3. Rozdzielenie zamówienia klienta od realizacji magazynowej

### 3.1 Odpowiedzialność obiektów

- `CustomerOrder` reprezentuje handlowe zapotrzebowanie klienta.
- `CustomerOrderLine` reprezentuje zamówione SKU i ilość oraz ma własny cykl stanów.
- `OutboundOrder` reprezentuje magazynowy zakres realizacji powstały przez grupowanie lub dzielenie pozycji zamówień. Nie jest zadaniem kompletacji dla operatora.
- `OutboundOrderLine` reprezentuje część `CustomerOrderLine` przekazaną do konkretnej realizacji magazynowej i ma własny cykl stanów.
- `Allocation` rezerwuje zapas dla `OutboundOrderLine`.
- `PickTask` jest wykonywalnym zadaniem dla jednego `Warehouse Operator (Picker)`.
- `TU` jest fizyczną jednostką transportową używaną w Outbound.

Podstawowy łańcuch zależności:

`CustomerOrderLine → OutboundOrderLine → Allocation → PickTask → Picking TU`

### 3.2 Relacje `CustomerOrder`–`OutboundOrder`

1. Relacja jest wiele-do-wielu, realizowana i rozliczana na poziomie pozycji oraz ilości.
2. `CustomerOrderLine` może zostać podzielona między wiele `OutboundOrderLine`.
3. Kilka `CustomerOrder` może zostać połączonych przed kompletacją w jeden `OutboundOrder`, jeśli:
   - należą do tego samego klienta;
   - mają ten sam adres dostawy;
   - mają zgodne `priority`;
   - ich `slaDeadline` mieszczą się w tolerancji konfigurowanej per magazyn;
   - żadne z nich nie ma `allowPartialShipment = false`.
4. Podział pracy według stref nie tworzy osobnych `OutboundOrder`. Odpowiadają za niego `PickTask`.
5. Jeden `OutboundOrder` może generować wiele `PickTask`, również dla wielu stref.

### 3.3 Moment tworzenia `OutboundOrder`

1. `CustomerOrder` po walidacji może pozostawać w `ACCEPTED` bez `OutboundOrder`, oczekując na cykliczne planowanie.
2. Cykliczne planowanie pomija `CustomerOrder.ON_HOLD`.
3. Po zdjęciu `ON_HOLD` zamówienie automatycznie wraca do kolejki planowania.
4. **DEC-C05:** pierwszy `OutboundOrder` powstaje podczas cyklicznego planowania realizacji, przed utworzeniem `Allocation`.
5. **DEC-C06:** częściowa alokacja nie tworzy pierwszego `OutboundOrder`. Dla `allowPartialShipment = true` istniejący `OutboundOrder` obejmuje ilość możliwą do realizacji, a brakująca ilość pozostaje `BACKORDERED`. Kolejny `OutboundOrder` powstaje dopiero po pojawieniu się dostępnego zapasu dla brakującej ilości.
6. Dla `allowPartialShipment = false` powstaje dokładnie jeden `OutboundOrder`, który nie może agregować innych `CustomerOrder`.

### 3.4 Zamknięcie `CustomerOrder` i porządkowanie stanów (`BACKLOG.md` B15)

1. **DEC-L33 — Wyzwalacz `CustomerOrder SHIPPED→CLOSED`:** rozliczenie nagłówka jest wyliczane automatycznie przez System WMS, nie zgłaszane ręcznie ani przez sygnał zewnętrzny z ERP — `CustomerOrder` przechodzi z `SHIPPED` w `CLOSED`, gdy każdy `OutboundOrder`, który wniósł choć jedną wydaną `OutboundOrderLine` do tego `CustomerOrder`, osiągnął `COMPLETED` (`propozycja_procesow_outbound.md` §4.10, `CarrierManifest HANDED_OVER→CONFIRMED`).
   **Uzasadnienie:** decyzja Darka 2026-08-18 — symetria z istniejącym wzorcem `OutboundOrder.COMPLETED`←`CarrierManifest` (§4.10); nie wymaga nowego aktora ani nowego kanału integracyjnego. Odrzucone warianty: ręczne zamknięcie przez `Warehouse Supervisor` (zbędny krok ręczny bez uzasadnienia biznesowego) i sygnał zewnętrzny z ERP/OMS (nie istnieje dziś żaden odpowiednik takiej bramki dla zamknięcia `CustomerOrder`, w odróżnieniu od bramki `POSTED` dla `Shipment`).
2. **DEC-L34 — Usunięcie `OutboundOrder.ON_HOLD`:** stan `ON_HOLD` i przejścia `PICKING_IN_PROGRESS↔ON_HOLD` usunięte z `OutboundOrder` (`propozycja_procesow_outbound.md` §4.3, diagram). Weryfikacja wsteczna (przegląd §3, §5, §7, §9 i historii B1–B14) nie znalazła żadnego opisanego wyzwalacza, aktora ani warunku wznowienia dla tego stanu — artefakt roboczy bez pokrycia biznesowego. `ON_HOLD` pozostaje wyłącznie na poziomie `CustomerOrder` (przed powstaniem `OutboundOrder`, sekcja 3.3 pkt 2-3).
   **Uzasadnienie:** decyzja Darka 2026-08-18 — brak przypomnienia sytuacji użycia po stronie właściciela produktu; utrzymywanie nienazwanego stanu w kanonie tworzy ryzyko niespójnej implementacji. Jeśli w przyszłości pojawi się realna potrzeba wstrzymania pojedynczego `OutboundOrder`, wymaga to nowej, świadomej decyzji (nowy `DEC-*`), nie przywrócenia tego stanu bez uzasadnienia.

**Reguły:** `propozycja_procesow_outbound.md` — pkt 1: `R85`, §3.1 krok 13a; pkt 2: brak nowej reguły (usunięcie).

## 4. Wysyłki częściowe, alokacja i priorytety

### 4.1 `allowPartialShipment`

1. `allowPartialShipment = false` nie ogranicza liczby `PickTask` ani liczby Packing `TU`.
2. Brak choć jednej wymaganej pozycji blokuje wydanie całego `Shipment`.
3. Całe `CustomerOrder` musi zostać wydane w jednym kompletnym `Shipment`, który może zawierać wiele Packing `TU`.
4. Warehouse Supervisor może, z podaniem powodu, zmienić `allowPartialShipment` z `false` na `true` wyłącznie dla `CustomerOrder`, którego dotyczy.
5. Zmiana obowiązuje do końca cyklu życia tego `CustomerOrder`: każdy jego przyszły niedobór jest obsługiwany automatycznie jak przy `allowPartialShipment = true`, bez ponownego angażowania Supervisora. Nie jest ograniczona do jednego `Shipment`, ale nie zmienia ustawień klienta ani magazynu i nie dotyczy innych obecnych ani przyszłych `CustomerOrder` tego klienta. Może zostać podjęta zanim jeszcze powstał jakikolwiek `Shipment` dla tego zamówienia (np. przy eskalacji `SHORT_PICKED` w trakcie kompletacji lub `SHORT_ALLOCATED` przy alokacji); pozostawia brakującą ilość w `BACKORDERED`, a dostępna ilość jest realizowana i wydawana bez oczekiwania na resztę.

6. **DEC-L53 — guard `allowPartialShipment = false` kluczowany po `CustomerOrder`:**
   **Decyzja:** Packing `TU` zamówienia z `allowPartialShipment = false` nie jest wpinana do żadnego `Shipment`, dopóki każda `CustomerOrderLine` tego `CustomerOrder` nie spełnia jednego z dwóch rozłącznych warunków. Warunek (A), linia nieaktywna: linia została skutecznie anulowana albo skorygowana przez `Warehouse Supervisor` i nie należy już do ilości wymaganej tego `CustomerOrder`; taka linia guardu nie blokuje i nie podlega warunkowi (B), w szczególności nie wymaga się od niej istnienia żadnej `OutboundOrderLine`. Warunek (B), linia aktywna: suma ilości wymaganych tych `OutboundOrderLine` tej linii, które osiągnęły `PACKED` albo dalej i nie są `CANCELLED`, równa się `CustomerOrderLine.Quantity` obowiązującej po ewentualnej korekcie, niezależnie od kanału realizacji, który daną ilość zrealizował. Do czasu spełnienia warunku Packing `TU` pozostaje w `PACKING_SEALED`. Guard blokuje wyjście zamówienia w częściach; nie uwalnia linii pozostałych do realizacji — zamówienie czeka jako całość.
   **Uzasadnienie:** obietnica „całość jednym kompletnym `Shipment`" jest własnością `CustomerOrder`, a guard kluczowany po `OutboundOrder` przecieka, gdy zamówienie zostaje rozdzielone na kanał standardowy i crossdockowy: kanałów nie wolno mieszać w jednym `OutboundOrder`, więc powstają dwa nagłówki i guard otwiera się przedwcześnie. Warunek musi być ilościowy, nie statusowy — `CustomerOrderLine` przechodzi w `PLANNED` już przy przypisaniu do `OutboundOrderLine`, więc linia pokryta częściowo (przykład: `Quantity` 100, pokryte i spakowane 30, pozostałe 70 bez żadnej `OutboundOrderLine`) spełniałaby warunek statusowy, mimo że zamówienie nie jest kompletne.
   **Odrzucone alternatywy:** brak.
   **Data:** 2026-08-25.
   **Reguły:** `P1 R57`, `P1 R58`.

7. **DEC-L57 — atrybut `requiredQty` na `OutboundOrderLine` i jego cykl życia:**
   **Decyzja:** `OutboundOrderLine` otrzymuje atrybut `requiredQty` — ilość wymaganą tej linii. Jest to ilość docelowa obowiązująca, nie historyczna. Ustawiana jest przy powstaniu linii: w kanale `STANDARD` przy tworzeniu `OutboundOrder`, w kanale `CROSSDOCK` przy generowaniu `CrossDockPickTask`. Zmienia się wyłącznie razem z korektą `CustomerOrderLine.Quantity` przez `Warehouse Supervisor`, w dwóch nazwanych ścieżkach: `P1 R46` dla kanału standardowego i gałęzi korekty `P2 R15` dla kanału crossdockowego. Żadna inna ścieżka jej nie zmienia; nie istnieje automatyczna aktualizacja. Guard `allowPartialShipment = false` porównuje sumę `requiredQty` z `CustomerOrderLine.Quantity` równością.
   **Uzasadnienie:** cztery miejsca kanonu powołują się na ilość wymaganą `OutboundOrderLine`, a żaden obiekt jej nie przechowuje — jedyny atrybut ilościowy tej linii, `pickedQty`, opisuje ilość faktycznie pobraną i w kanale crossdockowym wprost nie dotyczy. Bez tego atrybutu guard kluczowany po `CustomerOrder` nie da się zapisać ilościowo. Nazwa `requiredQty` mówi wprost „ilość wymagana", zgodnie z brzmieniem tych czterech miejsc, i jest wyraźnie różna zarówno od `pickedQty`, jak i od `CustomerOrderLine.Quantity`. Rozwiązanie polegające na atrybucie niezmiennym i zastąpieniu linii nową jest sprzeczne z `P2 R15`, który mówi o kontynuacji tej samej linii do `PACKING_SEALED`. Rozwiązanie z atrybutem niezmiennym i porównaniem nierównościowym osłabia jedyny warunek chroniący `allowPartialShipment = false`. Ceną przyjętą świadomie jest zawężenie `DEC-L38`, zapisane osobnym wpisem.
   **Odrzucone alternatywy:** brak.
   **Data:** 2026-08-26.
   **Reguły:** `P1 R46`, `P1 R69`, `P2 R15`.

**Reguły:** `propozycja_procesow_outbound.md` — pkt 2–3: `SP11`, `R5`; pkt 4–5: `SP12`, `R6`.

8. **DEC-L64 — bramka `allowPartialShipment = false` liczy pokrycie mieszane:**
   **Decyzja:** Warunek wejścia do planowania dla `allowPartialShipment = false` nie pyta o pełne `ATPReservation`, tylko o pełne pokrycie każdej `CustomerOrderLine`: `ATPReservation` powiększone o sumę `requiredQty` jej `OutboundOrderLine`, które osiągnęły `PACKED` albo dalej i nie są `CANCELLED`, niezależnie od kanału. Standardowy `OutboundOrder` obejmuje wyłącznie ilość niepokrytą; przy pełnym pokryciu cross-dockiem nie powstaje wcale.
   **Uzasadnienie:** cross-dock nie tworzy `ATPReservation`, więc poprzednie brzmienie bramki nigdy się nie otwierało dla linii pokrytej tym kanałem — zamówienie z zakazem wysyłki częściowej mogło zostać w `ACCEPTED` bezterminowo, mimo że towar fizycznie był. Miara pokrycia jest celowo ta sama co w `P1 R58`, żeby w korpusie nie powstały dwie definicje „linii pokrytej".
   **Odrzucone alternatywy:** liczenie strony crossdockowej od utworzenia `CrossDockPickTask`, a nie od `PACKED` — odrzucone, bo ilość crossdockowa jest niepewna do fizycznego potwierdzenia, a zamówienie z zakazem wysyłki częściowej nie może wyjechać niekompletne. Świadomie różni się to od miary w `P2 R6`, gdzie odejmowane jest wszystko obiecane; oba warunki są zachowawcze, ale w przeciwne strony.
   **Data:** 2026-08-28.
   **Reguły:** `P1 R6`.

### 4.2 Niedobór podczas alokacji

1. System rezerwuje dostępną ilość niezależnie od `allowPartialShipment`.
2. Przy `allowPartialShipment = true` częściowa alokacja pozostawia dostępną ilość w istniejącym `OutboundOrder`; nie tworzy pierwszego ani dodatkowego `OutboundOrder` w chwili alokacji.
3. Brakująca ilość pozostaje w tym samym `CustomerOrder` jako `BACKORDERED`.
4. Przy `allowPartialShipment = false` częściowa rezerwacja pozostaje zablokowana do czasu dostępności całości albo zwolnienia zgodnie z polityką magazynu.
5. Polityka utrzymywania częściowej rezerwacji jest konfigurowalna i może oznaczać:
   - utrzymanie rezerwacji;
   - automatyczne zwolnienie po określonym czasie;
   - skierowanie do decyzji `Warehouse Supervisor`.
6. Czas utrzymania rezerwacji nie zależy od `priority` ani `slaDeadline`.
7. Automatyczne zwolnienie rezerwacji nie wymaga powiadomienia Supervisor.
8. **DEC-D11:** `SHORT_ALLOCATED` jest obsługiwany automatycznie według `allowPartialShipment` i polityki utrzymania rezerwacji. `Warehouse Supervisor` uczestniczy wyłącznie wtedy, gdy wskazuje go skonfigurowana polityka albo wymagana jest decyzja wyjątkowa.
   **Reguły:** `propozycja_procesow_outbound.md` — `R33`, `R39`.
9. **DEC-D12:** Dla `CustomerOrder` z `allowPartialShipment = false`, cykliczne planowanie nie tworzy `OutboundOrder`, dopóki każda `CustomerOrderLine` tego zamówienia nie ma pełnego `ATPReservation` (równego zamówionej ilości). `CustomerOrder` zostaje w `ACCEPTED` tak długo, jak trzeba. Uzasadnienie: bez tego warunku `OutboundOrder` mógłby powstać z częściową rezerwacją (część linii `RESERVED`, część `SHORT`), co przy `allowPartialShipment = false` i tak nie pozwala niczego wysłać (brak choćby jednej pozycji blokuje cały `Shipment`, D-4.1 pkt 2) — a jednocześnie wymagałoby później anulowania i sprzątania (`Allocation RELEASED`, zwrot do `ATPReservation`) tylko po to, żeby zacząć od nowa. Czekanie na komplet rezerwacji, zanim cokolwiek się zablokuje fizycznie, jest prostsze: `ATPReservation` (miękka rezerwacja, decyzje sekcja 4.3 pkt 8-14) już chroni dostępny zapas przed innymi zamówieniami zgodnie z regułami kolejki (pkt 10-11), więc nic nie ginie w czasie oczekiwania.
   **Reguły:** `propozycja_procesow_outbound.md` — `R49`.
10. **DEC-D13:** Mimo DEC-D12, `SHORT_ALLOCATED` nadal może wystąpić dla `allowPartialShipment = false`, ale rzadko — jako przypadek brzegowy: między momentem, gdy wszystkie linie osiągną pełne `ATPReservation`, a momentem faktycznego przekształcenia tej rezerwacji w prawdziwą `Allocation`, kolejka priorytetowa (warianty z priorytetem/`slaDeadline`, decyzje sekcja 4.3 pkt 11b) może odebrać część `ATPReservation` nowo przyjętemu zamówieniu o wyższym priorytecie, zanim zdążyła się przekuć w twardą rezerwację. W tym przypadku `SHORT_ALLOCATED` i eskalacja do `Warehouse Supervisor` działają dokładnie tak, jak opisano w R40 (bez zmian) — dwa wyniki: trwała zmiana `allowPartialShipment` na `true`, albo anulowanie `OutboundOrder`.
   **Reguły:** `propozycja_procesow_outbound.md` — `R40`, `R50`.
11. **DEC-D14:** Dotyczy wyłącznie `allowPartialShipment = false` (kontynuacja przypadku z DEC-D13). Gdy `Warehouse Supervisor` wybiera anulowanie: `OutboundOrder` przechodzi w `CANCELLED`, każda objęta nim `Allocation` (pełna lub `SHORT`) w `RELEASED`, a odpowiadająca jej ilość wraca do `ATPReservation` właściwej `CustomerOrderLine` (R47) — `Allocation` przy `SHORT_ALLOCATED` jest tworzona już po ujawnieniu prawdziwej dostępności, więc nigdy nie jest zawyżona, i cała jej ilość wraca bez korekty. Status `CustomerOrderLine` po powrocie zależy od tego, która linia faktycznie przegrała wyścig o `ATPReservation` opisany w DEC-D13: linia, której zabrano rezerwację na rzecz zamówienia o wyższym priorytecie, przechodzi w `BACKORDERED` (realny, nierozwiązany problem z dostępnością — sygnał dla `Warehouse Supervisor`). Pozostałe linie tego samego `OutboundOrder`, które zdążyły uzyskać rzeczywistą `Allocation` i nie miały własnego problemu z dostępnością, przechodzą w `OPEN` — wracają do stanu sprzed planowania, bez oznaczenia niedoboru. `CustomerOrder` wraca do `ACCEPTED` z ustawionym `WARNING` (DEC-D16) i czeka jako całość, tak samo jak przy pierwszym podejściu (DEC-D12), aż wszystkie linie razem odzyskają pełną rezerwację.
   **Reguły:** `propozycja_procesow_outbound.md` — `R47`, `R50`, `R56`, `R57`, `R60`.
12. **DEC-D15:** Status nagłówka `BACKORDERED` może wystąpić wyłącznie wtedy, gdy wszystkie aktywne `CustomerOrderLine` danego `CustomerOrder` są jednocześnie `BACKORDERED` (realny, nierozwiązany problem z dostępnością na każdej z nich — nie samo wycofanie porządkowe, DEC-D14/DEC-F15). Zasada obowiązuje niezależnie od `allowPartialShipment` (rozszerzone 2026-08-02 — zamyka L7 razem z DEC-D18, §4.4). W typowym przebiegu to rzadko zachodzi: zwykle tylko jedna linia (ta, która faktycznie przegrała wyścig o rezerwację przy `SHORT_ALLOCATED`, albo miała fizyczny brak przy `SHORT_PICKED`) trafia do `BACKORDERED`. Pozostałe linie: dla `allowPartialShipment = false`, gdy cały `OutboundOrder` zostaje anulowany (DEC-D14), wracają do `OPEN`, a nagłówek do `ACCEPTED` + `WARNING` (R56); dla `allowPartialShipment = true` pozostałe linie nie są ruszane i zostają na swoim bieżącym etapie realizacji, a nagłówek zostaje w `IN_FULFILLMENT` + `WARNING`. `BACKORDERED` na nagłówku jest więc skrajnym przypadkiem brzegowym, nie typowym stanem oczekiwania — dla żadnej wartości `allowPartialShipment`.
   **Reguły:** `propozycja_procesow_outbound.md` — `R56`, `R57`, `R60`.
13. Reguła z pkt 12 (`BACKORDERED` na nagłówku tylko, gdy wszystkie aktywne `CustomerOrderLine` są `BACKORDERED`) obowiązuje niezależnie od `allowPartialShipment` — ogólna zasada agregacji pozycja→nagłówek dla `CustomerOrder`, wraz z wyjątkiem `PARTIALLY_SHIPPED` i wyzwalaczem `WARNING` dla przypadków mieszanych, opisana w DEC-D18 (§4.4).
14. **DEC-D16:** `CustomerOrder` dostaje nową cechę `WARNING` — pojedynczą flagę z opisem tekstowym (nie listę wielu niezależnych ostrzeżeń). System WMS ustawia ją w czterech sytuacjach: (a) zamówienie `allowPartialShipment = false` pozostaje w `ACCEPTED` bez pełnego `ATPReservation` na wszystkich liniach (DEC-D12); (b) wystąpił `SHORT_ALLOCATED` wymagający decyzji Supervisora (DEC-D13); (c) wystąpił `SHORT_PICKED` wymagający decyzji Supervisora (sekcja 5, DEC-F12); (d) przy `allowPartialShipment = true`, choć jedna `CustomerOrderLine` jest `BACKORDERED`, a nie wszystkie linie zamówienia — nagłówek zostaje `IN_FULFILLMENT`, nie `BACKORDERED` (DEC-D15/DEC-D18), ale wymaga przeglądu Supervisora. Cel: `Warehouse Supervisor` ma jedno miejsce (panel monitoringu `CustomerOrder`) do przeglądania zamówień wymagających jego decyzji, zamiast szukać ich osobno w każdym typie obiektu.
   **Reguły:** `propozycja_procesow_outbound.md` — `R51`, `R60`.
15. **DEC-D17:** `WARNING` znika w dwóch przypadkach: (a) ręcznie, gdy `Warehouse Supervisor` ustawia flagę na `false` — jeśli jednak przyczyna nie została faktycznie rozwiązana, kolejny przebieg cyklicznego planowania ponownie ją ustawi na `true`, wykrywając ten sam brak; (b) automatycznie, gdy problem, który ją wywołał, faktycznie się rozwiąże — cykliczne planowanie tworzy `OutboundOrder` dla zamówienia z DEC-D12, albo przypadek `SHORT_PICKED`/`SHORT_ALLOCATED` zostaje rozliczony (udane pobranie/realokacja, albo decyzja Supervisora zamykająca sprawę).
   **Reguły:** `propozycja_procesow_outbound.md` — `R51`.

16. **DEC-L61:** `Allocation` otrzymuje własny atrybut ilości `reservedQty` — ilość zapasu faktycznie przez nią zablokowaną. Zapas jest zajęty wyłącznie w stanach `SHORT`, `RESERVED` i `CONFIRMED`, w wysokości `reservedQty`; `PENDING`, `RELEASED` i `CONSUMED` nie wnoszą nic. Wybrany wariant A z trzech rozważanych: (A) własny atrybut na `Allocation`; (B) jak A, ale `PENDING` blokuje pełne `requiredQty`; (C) brak atrybutu, ilość zajęta wyliczana z rekordów `Inventory`. Wariant B odrzucony — pokazywałby jako zajęty zapas jeszcze niezidentyfikowany, a okno `PENDING` jest krótkie, więc zysk byłby teoretyczny. Wariant C odrzucony — wymagałby wcześniejszej rozbudowy `Inventory`, opisanego w Outbound szczątkowo, czyli realnego rozszerzenia zakresu. Powód decyzji: dotychczasowe zdanie „ilość rezerwacji jest tożsama z `requiredQty`" jest nieprawdziwe dla `SHORT`, który powstaje właśnie przy rezerwacji częściowej. Wzór `P2 R6` nie jest tą decyzją zmieniany — rozłączność kanałów pozostaje otwarta jako `BACKLOG.md` B22.
    **Reguły:** `proces_1_standard_fulfillment.md` — `R71`, KROK 4; `model_stanow_outbound.md` — sekcja 5 `Allocation`.

### 4.3 `priority` i `slaDeadline`

1. `CustomerOrder` ma zarówno `priority`, jak i `slaDeadline`.
2. `OutboundOrder` agregujący wiele zamówień uwzględnia oba kryteria i przyjmuje najpilniejsze wartości.
3. Priorytet wpływa na kolejność przydziału ATPReservation (kolejność alokacji, patrz pkt 8-14) oraz na kolejność kompletacji PickTask (patrz pkt 7).
4. Priorytet działa tylko na zapas jeszcze niezarezerwowany.
5. Istniejące `Allocation.RESERVED` nie mogą zostać odebrane zamówieniu o niższym priorytecie.
6. Realokacja priorytetowa nie jest możliwa po utworzeniu `PickTask` ani po rozpoczęciu fizycznej kompletacji.
7. Kolejność wykonywania PickTask przez Warehouse Operatora (uszczegółowienie „kompletacji" z pkt 3) jest parametrem magazynu: (a) slaDeadline → priority jako remis — domyślnie; (b) priority → slaDeadline jako remis. Remis na obu kryteriach rozstrzyga kolejność zgłoszenia PickTask — zadanie trafia do kolejnego wolnego Warehouse Operatora.
8. `CustomerOrderLine` ma cechę `ATPReservation` (obok `Quantity`) — ilość zapasu `ATP` miękko zarezerwowaną dla tej linii, zanim jeszcze powstanie rzeczywista `Allocation`. Dostępność dla nowych rezerwacji na danym SKU = `Inventory.AVAILABLE` (z flagą `ATP`) − suma aktywnych `ATPReservation` na tym SKU. `Allocation` nie jest odejmowana osobno — jej efekt jest już ujęty w stanie `AVAILABLE`. Suma aktywnych `ATPReservation` na danym SKU nigdy nie przekracza `ATP`; z tego wynika, że `ATPReservation` może być niezerowe tylko, gdy `ATP > 0` (warunek konieczny, nie wystarczający).
9. `ATPReservation` dla `CustomerOrderLine` jest ustawiane w momencie, gdy `CustomerOrder` osiąga status `ACCEPTED`, niezależnie od wartości `allowPartialShipment`. Jeżeli dostępność na danym SKU wynosi 0 w tym momencie, nowa linia otrzymuje `ATPReservation = 0` i oczekuje w kolejce.
10. Kolejność przydziału `ATPReservation` między oczekującymi `CustomerOrderLine` na tym samym SKU jest parametrem magazynu: (a) wyłącznie kolejność zgłoszenia, bez priorytetu i `slaDeadline` — domyślnie; (b) priorytet → `slaDeadline` jako remis; (c) `slaDeadline` → priorytet jako remis.
11. Kolejka jest przeliczana przy każdym zdarzeniu zmieniającym jej skład na danym SKU: (a) `Inbound TU POSTED` dla ASN zawierającego dany SKU — dotyczy wszystkich trzech wariantów: w wariancie (a) nowo dostępny towar trafia po kolei do czekających linii w kolejności zgłoszenia, spośród tych, dla których `ATPReservation` + ilość objęta rzeczywistą nieuwolnioną `Allocation` < `Quantity`; w wariantach (b)/(c) kolejka jest dodatkowo ponownie sortowana wg priorytetu/`slaDeadline` przed przydziałem. (b) nowe `CustomerOrder` osiąga `ACCEPTED` — dotyczy wyłącznie wariantów (b)/(c): może przejąć `ATPReservation` przydzielone wcześniej liniom o niższym priorytecie/dalszym miejscu w kolejce, o ile te linie nie mają jeszcze rzeczywistej `Allocation`; w wariancie (a) nowe zamówienie po prostu dołącza na koniec kolejki zgłoszeń i nie narusza niczyjej istniejącej rezerwacji.
12. Rzeczywista `Allocation` w stanie `RESERVED` nigdy nie jest odbierana przez późniejsze lub wyżej priorytetowe `CustomerOrder` — zgodnie z pkt 5. Ograniczenie kolejkowania (pkt 11) dotyczy wyłącznie miękkiej rezerwacji `ATPReservation`, nigdy rzeczywistej `Allocation`.
13. W miarę jak dla `CustomerOrderLine` powstaje rzeczywista `Allocation`, odpowiadająca ilość jest odejmowana z `ATPReservation` tej linii (przejście z rezerwacji miękkiej na twardą, utrzymywane przez całą resztę cyklu — `PICKING`, `PICKED`, `PACKED`, aż do `SHIPPED`). Jeżeli `OutboundOrderLine` zostaje anulowana przed `SHIPPED` z powodu niezwiązanego z rozbieżnością zapis/rzeczywistość (np. decyzja Supervisora o wycofaniu linii-siostry, DEC-F15), cała ilość objęta tą `Allocation` wraca do `ATPReservation` tej linii — `Allocation` nigdy nie jest zawyżona względem realnej dostępności w takich przypadkach. Jeżeli anulowanie następuje z powodu potwierdzonego `SHORT_PICKED` (fizyczny brak wykryty przy kompletacji, mimo że `Allocation` powstała wcześniej na pełną ilość), do `ATPReservation` wraca wyłącznie `ilość Allocation − potwierdzona ilość brakująca` — brakująca część pozostaje zablokowana do kontroli zapasu (DEC-F10) i nie jest realnym `ATP`, dopóki kontrola jej nie rozstrzygnie.
14. `Warehouse Supervisor` może ręcznie zmniejszyć lub usunąć `ATPReservation` dla `CustomerOrderLine` bez wpływu na status `CustomerOrder` i bez konieczności podania powodu.

**Reguły:** `propozycja_procesow_outbound.md` — pkt 5–6: `SP13`, `R11`; pkt 8: `R42`; pkt 9: `R43`; pkt 10: `R44`; pkt 11: `R45`; pkt 12: `R46`; pkt 13: `R47`; pkt 14: `R48`.

### 4.4 Agregacja statusu pozycja→nagłówek (`CustomerOrder`, L1/L7)

1. **DEC-D18:** Status nagłówka `CustomerOrder` wobec statusów jego `CustomerOrderLine` wyznacza zasada: nagłówek = status najmniej zaawansowanej aktywnej pozycji, z dwoma wyjątkami: (a) gdy choć jedna pozycja osiągnęła `SHIPPED` (część zamówienia wydana), nagłówek przyjmuje `PARTIALLY_SHIPPED`, niezależnie od stanu pozostałych pozycji; (b) nagłówek przyjmuje `BACKORDERED` wyłącznie, gdy wszystkie aktywne pozycje są `BACKORDERED` — dla dowolnej innej kombinacji z co najmniej jedną `BACKORDERED` nagłówek zostaje w swoim bieżącym statusie realizacji (`IN_FULFILLMENT`, albo dla `allowPartialShipment = false` po anulowaniu całego `OutboundOrder` — `ACCEPTED`) z ustawionym `WARNING` (DEC-D15, DEC-D16). Dotyczy wyłącznie `CustomerOrder`/`CustomerOrderLine` — `OutboundOrder` nie potrzebuje tej zasady: nie ma stanu `BACKORDERED` (`SHORT_ALLOCATED` po utworzeniu kończy się `CANCELLED`, DEC-D14) ani stanu pośredniego „częściowo wydany" (`COMPLETED` dopiero po potwierdzeniu wszystkich swoich `Shipment`, `propozycja_procesow_outbound.md` SP4); jego agregacja jest już w pełni pokryta SP2–SP4 i R58. Zamyka L1 (`OutboundOrder`, już pokryte SP2–SP4/R58) i L7 (`CustomerOrder`, ta reguła).
   **Uzasadnienie:** decyzja Darka 2026-08-02 — konserwatywna reguła (nagłówek nie „obiecuje" więcej niż najsłabsza pozycja), z jawnym wyróżnieniem częściowej wysyłki jako jedynego „pozytywnego" wyjątku; `BACKORDERED` ograniczone do przypadku, gdy realizacja faktycznie stoi w całości — pojedyncza linia z niedoborem przy reszcie w toku nie powinna sugerować, że całe zamówienie czeka (sygnał dla tego przypadku to `WARNING`, nie zmiana statusu nagłówka). Unika też rozrostu słownika o pełny zestaw `PARTIALLY_PICKED`/`PARTIALLY_PACKED` itd.
   **Reguły:** `propozycja_procesow_outbound.md` — `SP1`, `SP2`–`SP4`, `R58`, `R60`.
   **Szczegół zachowania:** `propozycja_procesow_outbound.md` §3.1, Funkcja ciągła F1; §4.1 (diagram `CustomerOrder`, krawędź `IN_FULFILLMENT↔BACKORDERED`).

## 5. `PickTask` i obsługa braków

1. Jeden `PickTask` jest przypisany do jednego operatora.
2. Jeden `OutboundOrder` może generować wiele `PickTask` dla różnych stref.
3. Strategia przypisywania Picking `TU` jest konfigurowalna per magazyn:
   - osobne Picking `TU` dla zadań lub stref;
   - wspólne Picking `TU` przechodzące przez kolejne zadania.
4. Jeden `PickTask` może korzystać z kilku Picking `TU` po zapełnieniu poprzedniego.
5. Kilka `PickTask` może równocześnie pracować na tym samym Picking `TU` wyłącznie w ramach jednego `OutboundOrder`.
6. Przed dodaniem towaru operator musi zeskanować Picking `TU`.
7. System blokuje przekroczenie limitu masy lub objętości `TU`.
8. `SHORT_PICKED` kończy pierwotny `PickTask`.
9. Realokacja brakującej ilości tworzy nowy `PickTask`, który dziedziczy priorytet zadania źródłowego.
10. Lokalizacja, w której wykazano brak, zostaje automatycznie zablokowana do kontroli zapasu.
11. **DEC-F10:** `SHORT_PICKED` kończy pierwotny `PickTask` i automatycznie blokuje lokalizację źródłową do kontroli zapasu.
    **Reguły:** `propozycja_procesow_outbound.md` — `R17`, `R47`.
12. **DEC-F11:** jeżeli istnieje kwalifikowany zapas `ATP` w innej niezablokowanej lokalizacji i nie osiągnięto efektywnego limitu prób, System WMS automatycznie tworzy nowy `PickTask` dla brakującej ilości. Nowe zadanie dziedziczy priorytet zadania źródłowego.
13. **DEC-F12:** przypadek trafia do `Warehouse Supervisor`, gdy nie istnieje kwalifikowany zapas, osiągnięto limit automatycznych realokacji albo potrzebna jest decyzja dotycząca `BACKORDERED`, anulowania lub odstępstwa od `allowPartialShipment = false`.
14. **DEC-F13:** parametr `maxAutomaticShortPickReallocations` jest ustawieniem magazynu z wartością domyślną `1`. W master data klienta ma wartość domyślną `null`. Wartość klienta różna od `null` nadpisuje ustawienie magazynu; `null` oznacza dziedziczenie. Wartość `0` oznacza brak automatycznej realokacji.
15. **DEC-F14:** licznik automatycznych realokacji dotyczy konkretnego przypadku określonego przez `OutboundOrderLine` i brakującą ilość. Kolejne automatycznie utworzone `PickTask` kontynuują ten sam licznik. Udane pobranie zamyka przypadek.
16. **DEC-F15 — Wynik 1 — "czekamy" (`BACKORDERED`).** Dotyczy wyłącznie `allowPartialShipment = false`. `Warehouse Supervisor` decyduje: trzymamy się pierwotnego życzenia klienta, cała wysyłka pojedzie razem, dopiero gdy będzie kompletna. Skutki, obiekt po obiekcie:
    - `OutboundOrderLine` krótko pobranej linii przechodzi w `CANCELLED` (ten sam mechanizm co każde inne anulowanie, istniejąca krawędź `SHORT_PICKED → CANCELLED`, §4.4).
    - Jej `Allocation` przechodzi w `RELEASED`. Do `ATPReservation` tej `CustomerOrderLine` wraca ilość `Allocation` **pomniejszona o potwierdzoną ilość brakującą** — nie cała `Allocation` (poprawka R47): brakująca część jest zablokowana do kontroli zapasu (DEC-F10) i nie jest realnie dostępnym `ATP`, dopóki kontrola jej nie rozstrzygnie. Ta `CustomerOrderLine` przechodzi w `BACKORDERED` — ma realny, nierozwiązany problem z dostępnością, widoczny dla `Warehouse Supervisor` w panelu monitoringu.
    - Dla ilości już fizycznie pobranej (`pickedQty > 0`) powstaje `PutBackTask` (P4) — towar wraca na półkę.
    - **Wszystkie pozostałe linie tego samego `CustomerOrder`, które są w stanie `ALLOCATED` lub `PICKED` (ale nie `PACKED`), zostają cofnięte tym samym torem** — `OutboundOrderLine → CANCELLED`, `Allocation → RELEASED`, **cała** ilość `Allocation` wraca do `ATPReservation` (bez pomniejszenia — to zwykłe anulowanie bez rozbieżności zapis/rzeczywistość), `PutBackTask` dla tego, co już pobrano. Te linie (bez własnego problemu z dostępnością) przechodzą w `OPEN` — wracają do stanu sprzed planowania. To dzieje się automatycznie, jako mechaniczny skutek jednej decyzji Supervisora — nie wymaga osobnej decyzji per linia. Uzasadnienie: skoro cała wysyłka i tak nie pojedzie, dopóki brakująca ilość się nie znajdzie, trzymanie zarezerwowanego/pobranego towaru innych linii blokuje go bez potrzeby innym zamówieniom.
    - **Linie już `PACKED` nie są ruszane** — zostają fizycznie w strefie konsolidacji, czekają; ich `CustomerOrderLine` zostaje w `PLANNED`. Uzasadnienie: rozpakowanie już zapakowanej jednostki jest kosztowne operacyjnie i wymagałoby zgody Supervisora tak czy inaczej (D-8.4/R32), więc nie ma sensu tego robić automatycznie.
    - `CustomerOrder` wraca do `ACCEPTED` z `WARNING` (DEC-D16) **wyłącznie, jeśli żadna `OutboundOrderLine` tego zamówienia nie pozostała `PACKED`.** Jeśli choć jedna jest `PACKED` i czeka, `CustomerOrder` **zostaje w `IN_FULFILLMENT`** z ustawionym `WARNING` — nadal istnieje aktywny `OutboundOrder` obejmujący tę spakowaną część, więc nagłówek nie może formalnie wrócić do stanu sprzed planowania. W obu przypadkach nowy `OutboundOrder` dla brakującej/cofniętej części powstaje dopiero, gdy potrzebne linie razem odzyskają pełną rezerwację (DEC-D12).
    **Reguły:** `propozycja_procesow_outbound.md` — `R47`, `R52`, `R56`, `R57`.
17. **DEC-F16 — Wynik 2 — "anulowanie".** Kluczowy fakt: samo anulowanie `OutboundOrderLine` **nie wystarcza**, żeby cokolwiek mogło wyjechać — `allowPartialShipment = false` żyje na `CustomerOrderLine`/`CustomerOrder`, nie na `OutboundOrderLine`, więc nawet po anulowaniu zadania magazynowego wymagana ilość na zamówieniu klienta się nie zmienia i wysyłka pozostaje zablokowana (D-4.1 pkt 2). Jedynym skutecznym rozwiązaniem jest **edycja ilości na `CustomerOrderLine` przez `Warehouse Supervisor`** — WS rozstrzyga, ile anulować:
    - **WS anuluje pełną, pierwotnie zamówioną ilość** z `CustomerOrderLine` (np. 100 z 100) → to kasuje całe SKU na tym zamówieniu. `OutboundOrderLine` przechodzi w `CANCELLED`, `Allocation` w `RELEASED`, `ATPReservation` wraca do zera dla tej linii (linia nie jest już nikomu potrzebna), `PutBackTask` dla całej ilości, która została już pobrana.
    - **WS koryguje `CustomerOrderLine` do ilości faktycznie pobranej** (np. ze 100 na 60) → zamówienie zostaje zrealizowane w tej okrojonej formie, bo "wymagana ilość" to teraz 60, a 60 zostało w pełni skompletowane. Nie potrzeba `PutBackTask` dla tych 60 — to, co pobrano, dokładnie odpowiada skorygowanemu zamówieniu.
    - Ten mechanizm odpowiada częściowo na temat z `BACKLOG.md` B3 ("kryteria anulowania `CustomerOrder`/`CustomerOrderLine`, wyzwalacze całościowe i częściowe") — dokładnie dla tego wyzwalacza (eskalacja `SHORT_PICKED`). Pozostałe aspekty B3 (ogólne wyzwalacze niezwiązane z niedoborem) zostają otwarte. Rozróżnienie od korekty sprzedaży rozwiązuje mechanizm zgłoszenia gotowości wydania do ERP — patrz `propozycja_procesow_outbound.md` §3.1 krok 11a i §4.9 (stany `Shipment POSTING_PENDING`/`POSTED`/`POSTING_ERROR`).
    **Reguły:** `propozycja_procesow_outbound.md` — `R53`.
18. **DEC-F17 — Wynik 3 — "trwała zmiana `allowPartialShipment` na `true`".** Prostszy niż wynik 2, bo WS uzgodnił z klientem częściową realizację — nie trzeba edytować ilości na `CustomerOrderLine`. `OutboundOrderLine` brakującej części po prostu przechodzi w `CANCELLED` (ten sam mechanizm co wynik 1, R47), dostępna ilość jedzie dalej i zostaje wydana jako osobna wysyłka. Od tej chwili każdy przyszły niedobór na tym `CustomerOrder` jest obsługiwany automatycznie jak przy `allowPartialShipment = true` (R6), bez ponownego angażowania Supervisora.
19. **DEC-F18 — Brak towaru wykryty podczas pakowania (kontrola „repack by SKU", DEC-G12):** dwa momenty wykrycia — (a) podczas liczenia pojedynczego SKU, gdy policzona ilość jest mniejsza niż `pickedQty` tej pozycji; (b) przy zakończeniu przepakowania źródłowego `TU`, gdy System WMS stwierdza, że nie wszystkie systemowo oczekiwane SKU zostały rozliczone. W obu przypadkach brak **nie jest rejestrowany automatycznie** — Packer może przy (a) odłożyć decyzję i wrócić do pozycji później (przed zakończeniem `TU`); zgłoszenie braku w obu przypadkach wymaga (1) ponownego sprawdzenia zawartości `TU` względem tej pozycji i (2) jawnego potwierdzenia zgłoszenia przez Packera. Dopiero to potwierdzenie uruchamia mechanizm: System WMS koryguje `pickedQty` do faktycznie potwierdzonej ilości, `OutboundOrderLine` przechodzi `PICKED → SHORT_PICKED` dla brakującej części, dalej dokładnie ten sam mechanizm co `SHORT_PICKED` wykryty przy kompletacji (DEC-F11–DEC-F17): automatyczna realokacja w granicach `maxAutomaticShortPickReallocations`, po wyczerpaniu — eskalacja do `Warehouse Supervisor` z trzema wynikami. **Odstępstwo od DEC-F10:** lokalizacja źródłowa pierwotnego `PickTask` nie zostaje zablokowana — nie ma pewności, czy przyczyną jest błąd operatora podczas `Pick` czy rzeczywisty brak na lokalizacji. Zdarzenie zawsze rejestrowane jako niezgodność w kontekście pierwotnego `PickTask`, odłożona do wyjaśnienia przez `Warehouse Supervisor`; sam mechanizm wyjaśniania (w tym rozliczenie brakującej ilości w `Inventory`) poza zakresem — `WMSAI_OUTBOUND/BACKLOG.md` B9.
    **Uzasadnienie:** decyzja Darka 2026-08-07 — ponowne użycie sprawdzonego mechanizmu `SHORT_PICKED` bez tworzenia równoległego mechanizmu; wymóg potwierdzenia zapobiega przedwczesnemu zgłoszeniu braku przez operatora, który świadomie nie przeszukał całej `TU`; brak blokady lokalizacji chroni przed fałszywym zablokowaniem sprawnej lokalizacji z powodu potencjalnego błędu operatora.
20. **DEC-F19 — Towar uszkodzony wykryty podczas pakowania:** na ścieżce „repack by SKU" (DEC-G12) towar uszkodzony nie może zostać przepakowany do Packing `TU`. Packer zgłasza SKU jako `DAMAGED`; System WMS każe odłożyć go na wyznaczone miejsce `QC`. Dla tej ilości uruchamia się dokładnie ten sam mechanizm co DEC-F18 (`PICKED → SHORT_PICKED`, realokacja/eskalacja, bez blokady lokalizacji) — bez wymogu dwuetapowego potwierdzenia z DEC-F18 (uszkodzenie to bezpośrednia obserwacja fizyczna, bez niepewności czy operator przeszukał całą `TU`). Zdarzenie oznaczone jest jako `DAMAGED` (nie ogólny brak) w rejestrze niezgodności — `WMSAI_OUTBOUND/BACKLOG.md` B9.
    **Uzasadnienie:** decyzja Darka 2026-08-07 — skutek dla realizacji zamówienia jest identyczny jak dla braku (towar i tak nie trafi do klienta), ale przyczyna (uszkodzenie, nie zgubienie/pomyłka) jest odrębnym faktem wartym oznaczenia dla przyszłego dochodzenia.

### 5.1 Zbieranie bezpośrednio do Outbound `TU`

21. **DEC-L35 — Alternatywna ścieżka kompletacji z pominięciem Packera:** Picker może przy pierwszym skanie Picking `TU` dla danego `PickTask` zadeklarować zbieranie bezpośrednio do Outbound `TU` (nowy atrybut `TU.directPackDeclared = true`, `propozycja_procesow_outbound.md` §4.7, §3.1 krok 6a) — decyzja wiążąca i nieodwracalna dla tego zadania. Gdy po zakończeniu kompletacji `TU` spełnia warunki wydania, System WMS automatycznie wykonuje `READY_TO_PACK→PACK_QUALIFIED→PACKING_SEALED` (§4.7) oraz `OutboundOrderLine PICKED→PACKED` (§4.4) — bez udziału Packera. Gdy `TU` **nie** spełnia warunków wydania mimo deklaracji, System WMS kieruje do standardowej oceny Packera (§3.1 krok 7, keep/repack/consolidate) — dokładnie tak, jak dla `TU` bez deklaracji; Packer pozostaje drugim, zapasowym punktem weryfikacji. Bez zmian w diagramach stanów `TU`/`OutboundOrderLine` (żadnych nowych stanów/krawędzi) — zmienia się wyłącznie aktor i moment wykonania istniejącej oceny.
    **Uzasadnienie:** decyzja Darka 2026-08-18 — dzisiejsza ocena „czy `TU` spełnia warunki wydania" (krok 7) już istnieje jako mechanizm; nowa ścieżka pozwala doświadczonemu Pickerowi zadeklarować to z góry (gdy z charakteru zadania wynika, że `TU` i tak trafi bezpośrednio do wysyłki), skracając czas realizacji bez tworzenia nowego mechanizmu oceny. Ryzyko błędnej deklaracji jest ograniczone tym samym warunkiem wydania co dziś — jeśli deklaracja okaże się nietrafiona, `TU` i tak trafia do Packera, więc nie ma ścieżki bez weryfikacji.
    **Reguły:** `propozycja_procesow_outbound.md` — `R86`, §3.1 krok 6a/7, §4.7, §4.4.

22. **DEC-L65 — dziedziczenie `directPackDeclared` między kolejnymi Picking `TU` tego samego `PickTask`:**
   **Decyzja:** Kolejna Picking `TU`, utworzona po osiągnięciu `PICK_FULL` przez poprzednią w ramach tego samego `PickTask`, dziedziczy jej `directPackDeclared`. Operator nie jest pytany ponownie. Zawęża `DEC-L43` w części mówiącej o własnej deklaracji każdej `TU`.
   **Uzasadnienie:** deklaracja dotyczy sposobu prowadzenia jednego zadania, nie pojedynczego nośnika; pytanie operatora drugi raz w środku tego samego zadania jest zbędnym krokiem i pozwala na niespójny wynik — część zadania zebrana bezpośrednio do Outbound `TU`, część nie.
   **Odrzucone alternatywy:** utrzymanie osobnej deklaracji per `TU` — odrzucone jako operacyjnie uciążliwe i dopuszczające niespójność w obrębie jednego zadania.
   **Data:** 2026-08-28.
   **Reguły:** `P1 R16`, `P1 R67`.

## 6. Model Outbound `TU`

### 6.1 Tożsamość i numeracja

1. Inbound `TU` i Outbound `TU` są oddzielnymi obiektami i mają oddzielne cykle stanów.
2. Wewnętrzna identyfikacja obiektów w oprogramowaniu jest poza zakresem warstwy procesowej.
3. `TU_NUMBER` jest wymaganym numerem biznesowym Outbound `TU` i jest unikalny w ramach magazynu wśród aktywnych (nieterminalnych) Outbound `TU`.
4. `SSCC` jest opcjonalnym identyfikatorem zgodnym z GS1. Numeru wewnętrznego nie należy nazywać `SSCC`.
5. `TUSetup` jest zamkniętym słownikiem typów `TU`. Określa `externalIssuable`, `maxWeight`, `maxVolume`, `processUsage`, `outbound_tu_number_standard`, `numberFormatMask` i referencję `sequenceCode` dla Outbound `TU` tworzonych wewnętrznie.
6. `Sequence` jest niezależnym, systemowym licznikiem z `sequenceId`, unikalnym kodem, nazwą i kolejnym wolnym numerem. Wiele `TUSetup` może wskazywać tę samą `Sequence` (relacja N:1).
7. Zewnętrzne `TU` nie otrzymuje numeru z `Sequence`, ale ma przypisany `TUSetup` wskazujący pochodzenie zewnętrzne.
8. **DEC-G07:** poprawny `SSCC` może być równocześnie biznesowym `TU_NUMBER`, jeżeli spełnia regułę unikalności `TU_NUMBER`.
9. **DEC-G08:** wartość niezgodna z regułami SSCC nie może zostać zapisana w polu `SSCC`; pole pozostaje puste.
10. **DEC-G09:** unikalny numer zewnętrzny niebędący poprawnym SSCC może zostać zapisany jako `TU_NUMBER`.
11. **DEC-G10:** konflikt `TU_NUMBER` wymaga jawnej obsługi. Zabronione są ciche nadpisanie, automatyczne dodanie sufiksu oraz zapisanie zwykłego numeru w polu `SSCC`.
12. **DEC-G14 — walidacja `SSCC`:** `SSCC` jest poprawny wyłącznie, gdy ma dokładnie 18 cyfr i poprawną cyfrę kontrolną GS1 wyliczoną algorytmem mod-10. Przy generowaniu zwykłej (niecrossdockowej) Outbound `TU` System WMS nie weryfikuje prefiksu firmy względem zewnętrznego rejestru GS1: generuje `SSCC` wyłącznie z własnego, skonfigurowanego prefiksu i nie odbiera obcego `SSCC`. Wyjątkiem jest dziedziczenie `SSCC` przy cross-dockingu 1:1 (`DEC-G17`, `R27`).
13. **DEC-G15 — format `TU_NUMBER` spoza `SSCC`:** `TU_NUMBER`, który nie jest `SSCC`, używa symboliki Code 128, zawiera wyłącznie znaki alfanumeryczne bez znaków specjalnych i ma maksymalnie 20 znaków.
14. **DEC-G16 — unikalność i ponowne użycie `TU_NUMBER`:** `TU_NUMBER` jest unikalny wyłącznie wśród aktywnych (nieterminalnych) Outbound `TU`. Ponowne użycie wystąpi praktycznie tylko po wyczerpaniu `Sequence` i jej ręcznym resecie. Reset `Sequence` jest administracją techniczną, analogiczną do monitoringu certyfikatów, dysków lub CPU, i pozostaje poza opisem procesów biznesowych.
15. **DEC-G17 — brak późniejszej korekty lub reklasyfikacji:** `TU_NUMBER` i `SSCC` są ustalane deterministycznie przy tworzeniu Outbound `TU`: przez wygenerowanie zgodne z `TUSetup`/`Sequence` albo przez dziedziczenie z Inbound `TU` wyłącznie przy cross-dockingu 1:1, gdy walidacja przy tworzeniu potwierdza GS1. Nie istnieje etap późniejszej korekty, przechowania surowego skanu ani reklasyfikacji numeru, dlatego mechanizm analogiczny do Inbound `ORIGINAL_CODE` nie ma zastosowania. Patrz `K10` w `propozycja_procesow_outbound.md`.
16. **DEC-G18 — konflikt numeru:** konflikt `TU_NUMBER`/`SSCC` między różnymi fizycznymi jednostkami jest niemożliwy przy zachowaniu standardu GS1. Jeżeli mimo to wystąpi, oznacza błąd konfiguracji lub danych obsługiwany w trybie serwisowym, poza procesami biznesowymi i bez eskalacji do `Warehouse Supervisor`. Patrz `K11` w `propozycja_procesow_outbound.md`.
17. **DEC-G19 — model `Sequence`/`TUSetup`:** `Sequence` jest uniwersalnym, systemowym licznikiem z `sequenceId`, unikalnym kodem, nazwą i kolejnym wolnym numerem. `TUSetup` jest zamkniętym słownikiem typów `TU` z kodem typu jako kluczem, `externalIssuable`, limitami `maxWeight`/`maxVolume`, `processUsage`, `outbound_tu_number_standard` (`GS1`/`OWN`), `numberFormatMask` oraz `sequenceCode` wskazującym `Sequence`. Wiele `TUSetup` może wskazywać tę samą `Sequence`; każda Outbound `TU` przechowuje referencję do `TUSetup`.

18. **DEC-L51 — typ zewnętrzny nośnika Outbound `TU`:**
    **Decyzja:** Outbound `TU` pochodzenia zewnętrznego otrzymuje `tuSetupCode` typu zewnętrznego. Magazyn konfiguruje dokładnie jeden taki typ. Taka `TU` jest wydawalna bez spełnienia progów wydania — zgoda jest elementem procesu, nie osobnym krokiem akceptacji dla każdej `TU`. Kontrakt semantyczny typu zewnętrznego obejmuje cztery rozstrzygnięcia: (1) typ zewnętrzny jest wyłączony z oceny progów `P1 R64`, która otrzymuje gałąź pierwszeństwa — `TU` na typie o roli nośnika zewnętrznego spełnia progi wydania z definicji, a dolne progi nie są dla niej ewaluowane; (2) uznanie za wydawalną wynika z `externalIssuable = true` na typie zewnętrznym oraz z gałęzi z punktu 1, więc nie powstaje porównanie liczbowe z wartością niezdefiniowaną; (3) `P1 R65`, czyli ręczne wymuszenie wydawalności przez operatora, w tej ścieżce nie uczestniczy — dotyczy `TU` na typie wewnętrznym, która nie osiągnęła dolnych progów; (4) atrybuty progowe i limitowe typu zewnętrznego nie mają wartości, a katalog zapisuje to jako „nie dotyczy" — nigdy `0`, nigdy wartość graniczna, nigdy wartość zastępcza. Zastosowalność atrybutów `TUSetup` dla typu zewnętrznego: `tuSetupCode` wymagany; `externalIssuable` wymagany, wartość `true`; `processUsage` wymagany, a wartość identyfikującą ten typ ustala `DEC-L59`; `maxWeight`, `maxVolume`, `minIssueWeight`, `minIssueVolume`, `outbound_tu_number_standard`, `numberFormatMask` i `sequenceCode` — nie dotyczy, bez wartości. Brak `maxVolume` oznacza, że Carrier Selection nie ma czym dopasować przedziału objętości, więc `Shipment` z taką `TU` trafia na ręczny wybór przewoźnika (`P1 R51`) — skutek zamierzony.
    **Uzasadnienie:** pojęcie „zewnętrzne `TU`", używane przez `P1 R51`, nie miało w korpusie definicji obiektu, do którego się odwołuje. Nośnik spoza magazynu nie ma wiarygodnej masy ani wymiarów granicznych, więc limity i progi typów wewnętrznych nie mają dla niego zastosowania. Ręczny wybór przewoźnika jest świadomie przyjętym skutkiem tej decyzji, nie luką.
    **Odrzucone alternatywy:** brak.
    **Data:** 2026-08-25.
    **Reguły:** `P1 R51`, `P1 R64`, `P1 R65`, `P1 R68`.

19. **DEC-L59 — wartość `TUSetup.processUsage` identyfikująca typ zewnętrzny:**
    **Decyzja:** `TUSetup.processUsage` — enum, który dotąd nie miał ani jednej wartości — otrzymuje dokładnie jedną zdefiniowaną wartość: `EXTERNAL`, oznaczającą nośnik pochodzenia zewnętrznego. Enum pozostaje otwarty; pozostałe role nośnika i domknięcie słownika pozostają poza zakresem tej decyzji. Rozpoznanie Outbound `TU` pochodzenia zewnętrznego następuje wyłącznie po `processUsage` jej typu, nigdy po numerze `TU`. Nie powstaje żadna flaga logiczna dublująca `processUsage`.
    **Uzasadnienie:** bez dyskryminatora nie da się zapisać zdania „magazyn konfiguruje dokładnie jeden typ zewnętrzny", bo nie wiadomo, po czym System WMS ten typ rozpoznaje. Rozpoznawanie po numerze `TU` jest wykluczone — `P2 R7` dopuszcza zarówno dziedziczenie `TU_NUMBER`/`SSCC` ze źródła, jak i nadanie numeru z `Sequence`, więc numer nie niesie informacji o roli nośnika. Osobna flaga logiczna obok pustego `processUsage` wprowadzałaby drugi nośnik roli typu i trzeba by ją usuwać przy domykaniu słownika. Pełny, zamknięty słownik od razu wykracza poza to, czego wymaga sama definicja typu zewnętrznego.
    **Odrzucone alternatywy:** brak.
    **Data:** 2026-08-26.
    **Reguły:** `P1 R68`.

### 6.2 Outbound Crossdock

1. Proces zaczyna się, gdy Inbound `TU` osiąga `IN_CROSS_DOCK`. System WMS generuje `CrossDockPickTask` i powiązane `OutboundOrderLine` w `CREATED` dla dopasowanych `CustomerOrderLine` w `BACKORDERED`; `CustomerOrderLine` przechodzą do `PLANNED`. `CrossDockPickTask` jest wykonywany przez Packera, który skanuje SKU, ilość i jakość `OK`/`DAMAGED`.
2. **DEC-L03 — granica Inbound/Outbound:**
   [HISTORYCZNE — doprecyzowane przez DEC-L29] „Rozliczenie zadania" w tym wpisie okazało się niejednoznaczne — patrz DEC-L29 dla aktualnego, dokładnego momentu powstania Outbound `TU`.
   **Decyzja:** podczas `IN_CROSS_DOCK` nie istnieje Outbound `TU`; `CrossDockPickTask` operuje na źródłowej Inbound `TU`, a Outbound `TU` powstaje dopiero przy rozliczeniu zadania.
   **Uzasadnienie:** rozdziela odpowiedzialność procesową bez utrzymywania dwóch równoległych obiektów `TU` dla tej samej fizycznej jednostki przed wynikiem sortowania.
   **Odrzucone alternatywy:** Outbound `TU` istniejące równolegle z nieterminalną Inbound `TU`.
   **Data:** 2026-08-10.
   **Reguły:** brak nowej reguły — zachowanie opisane w `propozycja_procesow_outbound.md` §3.2 krok 1 i krok 3.
3. **DEC-L04 — dwa przypadki rozliczenia:**
   **Decyzja:** przy pełnym dopasowaniu 1:1 cała zawartość Inbound `TU` domyka jeden `Shipment` zgodnie z `SP14`/`R20`; Outbound `TU` zachowuje `TU_NUMBER`/`SSCC`. Przy rozsortowaniu n:n powstają nowe Outbound `TU` z numerami z sekwencji typu, a jedna Outbound `TU` może zbierać SKU z kilku Inbound `TU` przy kryteriach `SP14`/`R20`.
   **Uzasadnienie:** zachowanie numeru jest poprawne tylko wtedy, gdy fizyczna jednostka pozostaje niepodzielona; sortowanie n:n tworzy nowe jednostki biznesowe.
   **Odrzucone alternatywy:** zachowanie źródłowego `TU_NUMBER`/`SSCC` po rozsortowaniu n:n albo nadawanie nowego numeru także przy pełnym 1:1.
   **Data:** 2026-08-10.
   **Reguły:** `R72`.
4. **DEC-L05 — `OutboundOrderLine` jako instrukcja sortowania i zamek:**
   **Decyzja:** `OutboundOrderLine` powstaje w `CREATED` przed rozsortowaniem, wskazuje docelową Outbound `TU` i blokuje ponowne użycie tej samej ilości; `Allocation` nie powstaje.
   **Uzasadnienie:** wykonanie cross-dockingu wymaga instrukcji przed pracą Packera, ale ilość nie pochodzi z zapasu `Inventory` i nie może być rezerwowana przez `Allocation`.
   **Odrzucone alternatywy:** `OutboundOrder` tworzony przed pickingiem na wzór P1 — niewykonalne, ponieważ `SP2` wymaga `Allocation`, której w cross-dockingu nie ma.
   **Data:** 2026-08-10.
   **Reguły:** `R71`, `R73`, `SP18`.
5. Brak, `DAMAGED` albo pusta Inbound `TU` powodują anulowanie odpowiednich `OutboundOrderLine` i powrót `CustomerOrderLine` do `BACKORDERED`; SKU nieoczekiwane i `DAMAGED` trafiają na `QC`. Przy zakończeniu pełnego 1:1 Inbound `TU` przechodzi do `CROSS_DOCKED`; przy resztce albo n:n resztka trafia na `TRANSIT`, a Inbound `TU` do `IN_PUTAWAY`. Outbound `TU` w obu przypadkach wchodzi wprost w `PACKING_SEALED`, a `OutboundOrderLine` przechodzi do `PACKED`.
6. **DEC-L06 — bramka ERP wersji 1:**
   [HISTORYCZNE — zastąpione przez DEC-L23/DEC-L24] Bramka „wersji 1" (czekanie na `POSTED` całego ASN) i odłożenie rozbicia GR do B11 są nieaktualne. Aktualne zachowanie: `Shipment` czeka wyłącznie na `GR_ACCEPTED` rozliczenia crossdockowego każdej źródłowej Inbound `TU` (`DEC-L23`), a rozliczenie GR jest już rozbite na crossdockowe/putawayowe (`DEC-L24`–`DEC-L26`, zamyka B11).
   **Decyzja:** wysłanie cross-dockowego `Shipment` do ERP następuje dopiero po osiągnięciu przez cały źródłowy ASN statusu `POSTED`; do tego czasu `Shipment` pozostaje w `POSTING_PENDING`. Rozbicie GR na część cross-dockowaną i putawayową odłożono do `BACKLOG.md` B11.
   **Uzasadnienie:** bieżący model Inbound księguje GR dla całego ASN i nie udostępnia osobnego potwierdzenia ilości cross-dockowanej.
   **Odrzucone alternatywy:** wysłanie `Shipment` przed zaksięgowaniem GR albo przyjęcie nieistniejącego częściowego GR jako faktu wersji 1.
   **Data:** 2026-08-10.
   **Reguły:** `R54` (rozszerzenie).
7. **DEC-L07 — ilość cross-dockowa poza modelem ATP:**
   **Decyzja:** ilość obsługiwana przez `CrossDockPickTask` nigdy nie tworzy rekordu `Inventory`, dlatego nie wchodzi do puli `ATP` i nie wymaga nowej reguły ATP.
   **Uzasadnienie:** towar przechodzi bezpośrednio z Inbound `TU` do Outbound `TU`, bez etapu składowania i udostępnienia jako zapas.
   **Odrzucone alternatywy:** chwilowe utworzenie `Inventory`/`ATPReservation` wyłącznie na potrzeby cross-dockingu.
   **Data:** 2026-08-10.
   **Reguły:** brak nowej reguły — ilość cross-dockowa nigdy nie tworzy `Inventory`.
   Wyjątek: patrz `DEC-L21` (ścieżka odzysku po utracie demand).
8. **DEC-L08 — pierwszeństwo zadań:**
   [HISTORYCZNE — wycofane przez `DEC-L41` (2026-08-23)] Koncept puli operatorów, do którego odwołuje się ta decyzja, został wycofany w całości. Aktualny mechanizm: operator wybiera moduł pracy na terminalu RF, patrz `DEC-L41`–`DEC-L45`.
   **Decyzja:** operator przypisany jednocześnie do puli `CrossDockPickTask` i `PickTask` otrzymuje `CrossDockPickTask` z pierwszeństwem. Pełny mechanizm konfiguracji puli operatorów odłożono do `BACKLOG.md` B10.
   **Uzasadnienie:** cross-docking ma krótki czas operacyjny i traci wartość, gdy zadanie czeka za standardowym pickingiem.
   **Odrzucone alternatywy:** równy priorytet obu pul albo pełne projektowanie mechanizmu puli w ramach P2.
   **Data:** 2026-08-10.
   **Reguły:** `R75`.
9. Po utworzeniu `Shipment` proces korzysta z Carrier Selection, etykiety, bramki ERP, `CarrierManifest` i wydania jak P1.
10. **DEC-L29 — moment powstania Outbound `TU` w cross-dockingu:**
   **Decyzja:** doprecyzowano moment powstania Outbound `TU` w cross-dockingu; pełny, aktualny opis w `proces_2_outbound_crossdock.md` `R7` i KROK 2. Doprecyzowuje `DEC-L03`: „rozliczenie zadania" w tamtym wpisie było zapisem nieprecyzyjnym — patrz `R7` dla dokładnego momentu.
   **Uzasadnienie:** bez wcześniej istniejącego obiektu nie ma nic, co trzymałoby odłożoną ilość ani `TU_NUMBER` do zeskanowania przy potwierdzeniu przepakowania `SKU` (§3.2 krok 2) — a `DEC-L16` już zakłada, że operator zamyka i otwiera kilka Outbound `TU` w trakcie jednego, wciąż aktywnego `CrossDockPickTask`, co wymaga istnienia obiektu przed rozliczeniem zadania.
   **Odrzucone alternatywy:** powstanie dopiero przy `PACKING_SEALED` (dotychczasowy zapis §4.7) — odrzucone, bo nie tłumaczy, co operator skanuje/zapełnia w trakcie zadania; powstanie dopiero przy rozliczeniu całego zadania (dosłowne brzmienie `DEC-L03`) — odrzucone z tego samego powodu, dodatkowo niewykonalne przy wielu Outbound `TU` na jedno zadanie (`DEC-L16`).
   **Data:** 2026-08-15.
   **Reguły:** `propozycja_procesow_outbound.md` — §4.7, §3.2 krok 1 (doprecyzowanie).
11. **DEC-L30 — ujednolicenie terminologii `sourceEligibleQty`/`R77`: „zadeklarowana (ASN)" zamiast „fizycznie zidentyfikowana":**
   **Decyzja:** zmieniono nazewnictwo bazy ilościowej w `R76`, `R77` i §3.2 krok 3 z „fizycznie zidentyfikowana ilość" na „zadeklarowana (ASN) ilość", zgodnie z D42 (`../MEMORY.md`); pełny, aktualny opis w `proces_2_outbound_crossdock.md` `R6` i `R21`. Wyłącznie zmiana nazwy — wartość i formuła bez zmian.
   **Uzasadnienie:** D42 wprost określa tę ilość jako „niezweryfikowaną fizycznie przez Inbound" (Inbound nie otwiera `TU` przed przekazaniem do cross-dockingu) — poprzednia nazwa zaprzeczała własnemu ustaleniu kontraktu Inbound-Outbound.
   **Odrzucone alternatywy:** pozostawienie nazwy „fizycznie zidentyfikowana" — odrzucone, myląca terminologia sprzeczna z D42.
   **Data:** 2026-08-15.
   **Reguły:** `propozycja_procesow_outbound.md` — `R76`, `R77` (rewizja nazewnictwa); w tym pliku `DEC-L18`, `DEC-L20`, `DEC-L24`, `DEC-L28` (nazewnictwo, patrz Task 17).
12. **DEC-L31 — Outbound `TU` pusta po pełnym `PutBackTask` w cross-dockingu:**
   [HISTORYCZNE — rozszerzone przez `DEC-L46` (2026-08-23)] Guard poniżej jest wyłącznie ilościowy: powstał przed `DEC-L36`, która dopuściła zasilanie jednej docelowej Outbound `TU` przez kilka zadań, więc nie uwzględnia przypadku, w którym inne aktywne albo zaplanowane zadanie nadal wskazuje tę `TU` jako cel. Aktualny, pełny warunek anulowania: `DEC-L46`.
   **Decyzja:** gdy Outbound `TU` utworzona w trakcie aktywnego `CrossDockPickTask` (`DEC-L29`) traci, przez odzysk `PutBackTask` (`DEC-L21`), całą dotąd potwierdzoną ilość i pozostaje pusta bez żadnej innej potwierdzonej pozycji, przechodzi `CREATED`→`CANCELLED`; nigdy nie wchodzi w `PACKING_SEALED`.
   **Uzasadnienie:** obiekt istnieje od pierwszego odłożenia `SKU` (`DEC-L29`), więc scenariusz pustej Outbound `TU` po pełnym odzysku jest możliwy i wymaga jawnego zakończenia cyklu życia obiektu, analogicznie do istniejącej krawędzi `CREATED`→`CANCELLED` w standardowym flow (put-back/VOID).
   **Odrzucone alternatywy:** pozostawienie pustej Outbound `TU` bez formalnego zamknięcia stanu — odrzucone, obiekt pozostawałby trwale zawieszony w `CREATED` bez ścieżki wyjścia.
   **Data:** 2026-08-17.
   **Reguły:** `propozycja_procesow_outbound.md` — nowa `R84`, §4.7 (diagram, tabela przejść).
13. **DEC-L32 — `grAcceptanceStatus` aktualizowany systemowo, nie per `Shipment`:**
   **Decyzja:** po zarejestrowaniu wyniku GR dla Inbound `TU`, System WMS ustawia ten sam `grAcceptanceStatus` na wszystkich `CrossDockPickTask` w systemie, których `sourceInboundTU` odpowiada wskazanej Inbound `TU` — niezależnie od tego, do którego `Shipment` poszczególne zadania zasiliły `confirmedQty`, także gdy jedna Inbound `TU` zasiliła zadania należące jednocześnie do kilku różnych `Shipment` (przypadek n:n). Warunek bramki `R54` nadal liczy się osobno per `Shipment`, z pełnego zbioru jego własnych źródłowych Inbound `TU`.
   **Uzasadnienie:** ograniczenie aktualizacji wyłącznie do zadań zasilających jeden, aktualnie sprawdzany `Shipment` wymagałoby wielokrotnego, niezależnego przetwarzania tego samego sygnału GR przy każdym kolejnym `Shipment` zasilonym przez tę samą Inbound `TU` — aktualizacja systemowa po jednym potwierdzeniu z ERP jest spójniejsza wydajnościowo i eliminuje ryzyko rozjechania `grAcceptanceStatus` między zadaniami tej samej Inbound `TU`.
   **Odrzucone alternatywy:** aktualizacja `grAcceptanceStatus` wyłącznie dla zadań zasilających `Shipment`, dla którego akurat trwa sprawdzanie bramki `R54` — rozważone i odrzucone (ryzyko wielokrotnego przetwarzania tego samego sygnału, niespójność `grAcceptanceStatus` między zadaniami tej samej Inbound `TU` w różnych `Shipment`).
   **Data:** 2026-08-17.
   **Reguły:** `propozycja_procesow_outbound.md` — `R81` (rewizja zakresu aktualizacji).
14. **DEC-L36 — wyzwalacze zamknięcia crossdockowej Outbound `TU`:**
   [HISTORYCZNE — zastąpione przez `DEC-L39` (2026-08-22)] Kryterium poniżej (sam brak aktywnych/zaplanowanych zadań) jest niekompletne — brakuje wymogu osiągnięcia `slaDeadline`, patrz `DEC-L39` dla aktualnego, pełnego kryterium.
   **Decyzja:** docelowa Outbound `TU` przechodzi w `PACKING_SEALED` w dwóch przypadkach: gdy Packer zamknie ją skanem RF jako fizycznie pełną w trakcie aktywnego `CrossDockPickTask`, albo gdy żaden aktywny ani zaplanowany `CrossDockPickTask` nie wskazuje jej już jako celu. Zakończenie pojedynczego zadania nie zamyka `TU` samo w sobie.
   **Uzasadnienie:** `DEC-L04` dopuszcza zasilanie jednej Outbound `TU` przez zadania z kilku źródłowych Inbound `TU`, a generowanie zadania kieruje je do nowej albo już otwartej Outbound `TU`. Zamykanie `TU` przy rozliczeniu każdego zadania unieważniałoby pojęcie otwartej `TU` i zamykało ją, zanim kolejne zadanie zdąży do niej dołożyć towar. Dotychczasowe zapisy opisywały wyłącznie przypadek odwrotny — jedno zadanie zapełniające kolejno kilka `TU` (`DEC-L16`).
   **Odrzucone alternatywy:** zamykanie wyłącznie decyzją Packera, także dla `TU` niepełnej — odrzucone, niepełna `TU` mogłaby pozostać otwarta bez końca; zawężenie `DEC-L04` do jednego zadania na jedną docelową `TU` — odrzucone, traci konsolidację towaru z kilku palet przyjęciowych w jedną paczkę.
   **Data:** 2026-08-22.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R10` (rozszerzenie); `model_stanow_outbound.md` §7 (dwa wiersze przejścia `CREATED→PACKING_SEALED`).
15. **DEC-L37 — pierwszeństwo finalizacji źródłowej Inbound `TU` przed nowym zapotrzebowaniem:**
   **Decyzja:** finalizacja źródłowej Inbound `TU` następuje, gdy nie pozostaje żaden jej aktywny `CrossDockPickTask`, i kończy udział tej `TU` w cross-dockingu. Zapotrzebowanie zgłoszone po finalizacji nie powoduje utworzenia kolejnego zadania cross-dockowego dla tej `TU` — jest obsługiwane standardowo z zapasu po zakończeniu Putaway.
   **Uzasadnienie:** wzór `sourceEligibleQty` nadal wykazuje ilość rezydualną jako dostępną do zaplanowania, a dopasowywanie do `CustomerOrderLine BACKORDERED` chodzi cyklicznie — bez tej reguły powstaje wyścig między finalizacją a wygenerowaniem kolejnego zadania. Wariant deterministyczny jest zgodny z dosłownym brzmieniem `DEC-L18`, gdzie kryterium jest brak ilości rezydualnej, nie brak zapotrzebowania.
   **Odrzucone alternatywy:** pierwszeństwo nowego zapotrzebowania, dopóki paleta fizycznie stoi w strefie cross-dock — odrzucone, paleta blokowałaby strefę i opóźniała rozliczenie GR; okno czasowe jako parametr magazynu — odrzucone, wprowadza nowy parametr konfiguracyjny i nowy stan oczekiwania bez realnej przewagi.
   **Data:** 2026-08-22.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R28` (nowa).
16. **DEC-L38 — jednorazowe, zamknięte generowanie `CrossDockPickTask`/`OutboundOrderLine` (zawężenie `DEC-L19`):**
   [ZAWĘŻONE — patrz `DEC-L58` (2026-08-26)] Zakaz „aktualizacji wymaganej ilości `OutboundOrderLine` po utworzeniu" dotyczy automatycznego dogenerowania zadań i automatycznej aktualizacji ilości; jawna, nadzorowana korekta przez `Warehouse Supervisor` w gałęzi korekty `P2 R15` jest dopuszczalna.
   **Decyzja:** Generowanie `CrossDockPickTask` i `OutboundOrderLine` dla danej źródłowej Inbound `TU` jest zdarzeniem jednorazowym, wyzwalanym przez jej przejście w `IN_CROSS_DOCK`, dopasowującym wyłącznie `CustomerOrderLine BACKORDERED` istniejące w tym momencie. Nie istnieje mechanizm dogenerowania kolejnych `CrossDockPickTask` do już utworzonej `OutboundOrderLine` ani aktualizacji jej wymaganej ilości po utworzeniu — również w oknie między `IN_CROSS_DOCK` a finalizacją tej samej źródłowej `TU` (`DEC-L37`). Każda cross-dockowa `OutboundOrderLine` powstaje razem z dokładnie jednym `CrossDockPickTask`. Zamek ilościowy pozostaje na `CrossDockPickTask` (`DEC-L19`, bez zmian) — zawężeniu podlega wyłącznie twierdzenie o agregacji jednej `OutboundOrderLine` z wielu zadań różnych źródłowych `TU`.
   **Uzasadnienie:** dosłowne brzmienie `DEC-L19` dopuszczało dogenerowanie kolejnych zadań do już istniejącej `OutboundOrderLine`, co jest niespójne z jednorazowym, zdarzeniowym charakterem generowania opisanym w §3.2 krok 1 i z `DEC-L37` (finalizacja kończy udział źródłowej `TU`, nowe zapotrzebowanie idzie standardowym torem). Zamknięty zakres generowania eliminuje niejednoznaczność, kiedy dokładnie `OutboundOrderLine` jest „kompletna".
   **Odrzucone alternatywy:** utrzymanie otwartego mechanizmu dogenerowania — odrzucone, brak zdarzenia wyzwalającego poza `IN_CROSS_DOCK` i brak kryterium, kiedy dogenerowanie miałoby się zatrzymać.
   **Data:** 2026-08-22.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R30` (doprecyzowanie).
17. **DEC-L39 — automatyczne zamknięcie docelowej Outbound `TU` po `slaDeadline` (rozszerzenie `DEC-L36`):**
   **Decyzja:** docelowa Outbound `TU` przechodzi w `PACKING_SEALED` w jednym z dwóch przypadków: (1) Packer zamyka ją skanem RF jako fizycznie pełną w trakcie aktywnego `CrossDockPickTask` (`DEC-L36`, bez zmian); (2) System WMS zamyka ją automatycznie, gdy jej `slaDeadline` zostanie osiągnięty (albo minięty) i żaden aktywny ani zaplanowany `CrossDockPickTask` nie wskazuje jej już jako celu. Przed osiągnięciem `slaDeadline` sam brak takich zadań nie zamyka `TU` — towar cross-dockowy dla tego samego klienta/adresu/`priority`/`slaDeadline` (`DEC-L04`) może wciąż nadejść kolejną Inbound dostawą tego samego dnia albo w kolejnych dniach.
   **Uzasadnienie:** dotychczasowe kryterium `DEC-L36` (sam brak aktywnych/zaplanowanych zadań) zamykałoby `TU` przedwcześnie, zanim kolejna zgodna dostawa zdąży do niej dołączyć, obniżając wypełnienie `TU` i zwiększając liczbę drobnych wysyłek. `slaDeadline` wyznacza twardą, już istniejącą w modelu granicę tego oczekiwania (`DEC-L10` — analogiczny mechanizm dla `Shipment`).
   **Odrzucone alternatywy:** pozostawienie wyłącznie kryterium `DEC-L36` — odrzucone z powyższego powodu; nowy, osobny parametr czasowy zamiast reużycia `slaDeadline` — odrzucone jako zbędna komplikacja.
   **Data:** 2026-08-22.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R10` (rozszerzenie), `R3` (zaostrzenie do identycznego `slaDeadline`).
18. **DEC-L40 — ponowna ewaluacja bramki GR per komunikat, `GR_REJECTED` nie powoduje `POSTING_ERROR`, widoczność dla Supervisora (zawężenie `DEC-L23`):**
   **Decyzja:** bramka zgłoszenia `Shipment` do ERP (`DEC-L23`) jest ponownie ewaluowana przy każdym komunikacie GR (`GR_ACCEPTED` lub `GR_REJECTED`) dotyczącym którejkolwiek wymaganej źródłowej Inbound `TU`, niezależnie od poprzedniego wyniku dla tej `TU` czy stanu `Shipment` — w tym gdy `Shipment` znajduje się w `POSTING_ERROR` z innej przyczyny (np. odrzucenie ERP niezwiązane z GR, `DEC-L22`). Jawny `GR_REJECTED` nie przenosi samodzielnie `Shipment` w `POSTING_ERROR` — bramka pozostaje niespełniona; gdy dla tej samej `TU` nadejdzie następnie `GR_ACCEPTED`, bramka jest spełniona jak każda inna, bez odrębnej ścieżki odzyskania. `grAcceptanceStatus` źródłowych `TU` blokujących bramkę jest widoczny dla `Warehouse Supervisor`, bez nowego stanu, obiektu ani mechanizmu eskalacji.
   **Uzasadnienie:** zdanie `DEC-L23` traktujące `GR_REJECTED` jako trwałe, jawne odrzucenie analogiczne do odrzucenia ERP (`DEC-L22`) myliło niespełniony warunek wstępny bramki z faktycznym błędem zgłoszenia — ERP jeszcze nie widziało `Shipment`, więc nie mogło go odrzucić. Widoczność dla Supervisora daje operacyjny wgląd bez nowego mechanizmu eskalacji.
   **Odrzucone alternatywy:** utrzymanie `GR_REJECTED → POSTING_ERROR` z ręcznym retry Supervisora — odrzucone, myli niespełniony warunek wstępny z błędem zgłoszenia; nowy, osobny stan `Shipment` dla oczekiwania po odrzuceniu GR — odrzucone jako zbędny.
   **Data:** 2026-08-22.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R35`, `R36` (nowe).

19. **DEC-L46 — guard anulowania pustej docelowej Outbound `TU` i niezmiennik ciągłości celu (rozszerzenie `DEC-L31`):**
   **Decyzja:** anulowanie pustej docelowej Outbound `TU` (`DEC-L31`) wymaga, obok dotychczasowego warunku ilościowego, żeby żaden aktywny ani zaplanowany `CrossDockPickTask` nie wskazywał już tej `TU` jako celu; dopóki takie zadanie ją wskazuje, `TU` pozostaje w `CREATED` jako pusta, otwarta jednostka docelowa. Anulowanie nie czeka na `slaDeadline` — asymetria względem zamknięcia w `PACKING_SEALED` (`DEC-L39`) jest zamierzona i ma być zapisana jawnie w regule, żeby kolejny audyt nie zgłosił jej jako niespójności. Niezależnie od powyższego System WMS gwarantuje, że aktywny `CrossDockPickTask` zawsze ma dostępny cel: jeżeli w momencie odłożenia kolejnej pozycji nie istnieje otwarta docelowa Outbound `TU` tego zadania — niezależnie od przyczyny jej braku — System WMS otwiera nową i nadaje jej `TU_NUMBER`, leniwie, przy tym właśnie odłożeniu `SKU` (`DEC-L29`), nie z wyprzedzeniem.
   **Uzasadnienie:** `DEC-L31` nie jest błędna — jest niepełna wobec faktu ustalonego później. Powstała 2026-08-17, a `DEC-L36` (2026-08-22) dopuściła zasilanie jednej docelowej Outbound `TU` przez kilka zadań z różnych źródłowych Inbound `TU`. Guard wyłącznie ilościowy stał się przez to niekompletny: odzysk `PutBackTask` mógł anulować `TU`, do której nadal kierowało inne aktywne zadanie, zostawiając je ze wskazaniem na obiekt terminalny (dangling target reference), a mechanizmu retargetowania nie ma w żadnym aktywnym pliku. To ten sam wzorzec, w którym `DEC-L39` rozszerzył `DEC-L36` — rozszerzenie wcześniejszej decyzji, nie jej odwrócenie. Druga część decyzji uogólnia mechanizm, który dotąd istniał wyłącznie w jednej gałęzi `DEC-L36`/`R10` (Packer zamyka fizycznie pełną `TU` i kontynuuje do nowej), do niezmiennika obowiązującego niezależnie od przyczyny braku otwartej `TU` — dzięki temu nie istnieje stan, w którym aktywne zadanie nie ma gdzie odłożyć potwierdzonej ilości.
   **Odrzucone alternatywy:** czekanie z anulowaniem do `slaDeadline`, dla pełnej symetrii z `DEC-L39` — rozważone i odrzucone przez właściciela: pusta `TU` nie ma czego przetrzymywać, a fizyczny pojemnik zwalnia się w strefie cross-dock szybciej; świadomie przyjęty koszt to utrata nadanego `TU_NUMBER` (numery wracają do obiegu dopiero po wyczerpaniu `Sequence`, `DEC-G16`) i konieczność otwarcia nowej `TU`, gdyby kolejna zgodna dostawa nadeszła jeszcze przed terminem. Sam warunek o zadaniach, bez niezmiennika ciągłości celu — odrzucone, bo nie pokrywa przypadku, w którym aktywne zadanie zostaje bez celu z innej przyczyny niż anulowanie pustej `TU`.
   **Data:** 2026-08-23.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R34` (rewizja), `R40` (nowa).

20. **DEC-L47 — kwalifikacja Inbound jako przesłanka transportu, nie wiążący przydział:**
   **Decyzja:** kwalifikacja `TU` do cross-dockingu dokonana przez Inbound Proces 2 przy `IN_BUFFER` jest przesłanką transportu do strefy cross-dock, a nie wiążącym przydziałem towaru do konkretnego zapotrzebowania. Wiążące dopasowanie do `CustomerOrderLine` `BACKORDERED` powstaje dopiero przy przejściu źródłowej `TU` w `IN_CROSS_DOCK` (`R30`), na kolejce priorytetowej aktualnej w tym momencie. Rozjazd czasowy między tymi dwoma dopasowaniami, równy czasowi fizycznego przewozu, jest akceptowany świadomie. Gdy w tym oknie zniknie cały pasujący popyt, nie powstaje żaden `CrossDockPickTask`, a `TU` finalizuje się natychmiast z całą zadeklarowaną ilością jako rezydualną i przechodzi do `IN_PUTAWAY`. Zapisano też jako niezmiennik granicy, że do procesu trafia wyłącznie `TU` `ELEMENTARY`.
   **Uzasadnienie:** dopasowanie przy `IN_CROSS_DOCK` operuje na świeższych danych niż kwalifikacja Inbound, więc luka czasowa działa na korzyść trafności przydziału, nie przeciw niej. Jej jedynym kosztem jest możliwy pusty przebieg transportu — tańszy niż rezerwowanie zapotrzebowania na czas przewozu. Ścieżka zerowego dopasowania istniała dotąd wyłącznie jako zdegenerowane odczytanie `R21` i `R28` (zbiór powiązanych zadań pusty), bez własnej reguły, wymagania i scenariusza. Niezmiennik `ELEMENTARY` był dotąd zapisany tylko jako kwalifikator w prozie, nie jako reguła.
   **Odrzucone alternatywy:** rezerwowanie `CustomerOrderLine` na czas transportu — odrzucone, wprowadza blokadę zapotrzebowania bez fizycznego pokrycia; dogenerowywanie `CrossDockPickTask` po `IN_CROSS_DOCK` — wprost wykluczone przez `R30`; pozostawienie ścieżki zerowego dopasowania bez zapisu, na zdegenerowanym odczytaniu `R21`/`R28` — odrzucone, przyszły audyt odczytałby to jako lukę.
   **Data:** 2026-08-23.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R41` (nowa), `R42` (nowa).

21. **DEC-L48 — rozdzielenie finalizacji źródłowej `TU` od nadania statusu `IN_PUTAWAY` (doprecyzowanie `DEC-L47`):**
   **Decyzja:** finalizacja źródłowej Inbound `TU` w tym procesie i nadanie jej statusu `IN_PUTAWAY` to dwa różne zdarzenia w dwóch różnych procesach. Finalizacja przy zerowej ilości rezydualnej nadaje `CROSS_DOCKED` i jest zdarzeniem tego procesu. Finalizacja przy niezerowej ilości rezydualnej jest wyłącznie zgłoszeniem przekazania `TU` do Inbound Procesu 3 (Putaway); `IN_PUTAWAY` nadaje Inbound przy zakończeniu własnego przewozu `TU` na `TRANSIT` sektora, a do tego momentu `TU` pozostaje w `IN_CROSS_DOCK`.
   **Uzasadnienie:** `R28` stawiała dotąd wprost znak równości między finalizacją a przejściem w `CROSS_DOCKED` albo `IN_PUTAWAY`, a `R21` i `R42` powtarzały ten skrót. Po zmianie modelu transportu po stronie Inbound status `IN_PUTAWAY` nadawany jest przy zakończeniu przewozu, nie przy przekazaniu `TU` — czytana dosłownie, poprzednia treść oznaczała `TU` w statusie `IN_PUTAWAY` stojącą fizycznie w strefie cross-dock, sprzecznie z kanonem Inbound. Rozbieżność zgłoszona przez równoległą sesję Inbound; poprawiona w warstwie normatywnej, a nie zdaniem-wyjątkiem obok istniejących reguł, żeby plik nie opisywał dwóch mechanizmów naraz.
   **Odrzucone alternatywy:** poprawienie samej `R42` z pozostawieniem `R21` i `R28` bez zmian — odrzucone, dawałoby sprzeczność wewnątrz jednego pliku; przeniesienie opisu momentu nadania `IN_PUTAWAY` do prozy zamiast do reguł — odrzucone, proza streszcza reguły i nie jest warstwą normatywną; opisanie inboundowego zadania transportowego w tym procesie — odrzucone, przekracza granicę procesową i powielałoby kanon Inbound.
    **Data:** 2026-08-23.
    **Reguły:** `proces_2_outbound_crossdock.md` — `R21` (rewizja), `R28` (rewizja), `R42` (rewizja).

22. **DEC-L52 — definicja pełnego dopasowania 1:1 w cross-dockingu:**
   **Decyzja:** pełne dopasowanie 1:1 oznacza jednorodność przeznaczenia, nie zawartości: źródłowa Inbound `TU` może zawierać wiele `SKU`. Pełne dopasowanie zachodzi, gdy cała zadeklarowana (ASN) zawartość źródłowej Inbound `TU` pokrywa zapotrzebowanie `CustomerOrderLine` w `BACKORDERED` spełniające łącznie warunki wspólnego wydania: ten sam klient, ten sam adres dostawy, zgodny `priority` i identyczny `slaDeadline`. Przy `allowPartialShipment = true` zapotrzebowanie może pochodzić z kilku `CustomerOrder` tego samego klienta; przy `allowPartialShipment = false` — wyłącznie z jednego.
   **Uzasadnienie:** kryterium 1:1 było w korpusie wyrażone terminem „zasada jednorodności zawartości `TU`", którego korpus nigdzie nie definiuje. Jednorodność zawartości jest przy tym kryterium fałszywym — o wspólnym wydaniu decyduje przeznaczenie, nie skład `SKU`. Ustalenie kontekstowe: Inbound kwalifikuje `TU` do cross-dockingu wyłącznie po `SKU` i ilości, więc kryterium jednorodności jest w całości kryterium Outbound i nie wymaga żadnej zmiany po stronie Inbound.
   **Odrzucone alternatywy:** brak.
   **Data:** 2026-08-25.
   **Reguły:** `P2 R2`.

23. **DEC-L54 — `priority` i `slaDeadline` crossdockowego `OutboundOrder`:**
   **Decyzja:** crossdockowy `OutboundOrder` dziedziczy `priority` i `slaDeadline` z macierzystego `CustomerOrder`. Grupowanie linii w jeden crossdockowy `OutboundOrder` jest dopuszczalne wyłącznie przy zgodnym kliencie, zgodnym adresie dostawy, zgodnym `priority` i identycznym `slaDeadline`. Dla `allowPartialShipment = false` — wyłącznie linie jednego `CustomerOrder`.
   **Uzasadnienie:** reguła wskazująca źródło `priority`/`slaDeadline` obowiązywała oba kanały realizacji, a w korpusie ma odpowiednik wyłącznie dla kanału standardowego. Oba atrybuty są nośne w cross-dockingu: przydział `CrossDockPickTask` porządkuje po `slaDeadline` i `priority`, automatyczne zamknięcie docelowej Outbound `TU` odwołuje się do jej `slaDeadline`, a wpięcie Packing `TU` do `Shipment` wymaga identycznego `slaDeadline`. Bez dziedziczenia z tego samego `CustomerOrder` obie połówki zamówienia z `allowPartialShipment = false` mogłyby mieć różne wartości i nie trafiłyby do jednego `Shipment`.
   **Odrzucone alternatywy:** agregat wzorem stosowanym dla kanału standardowego, czyli przyjęcie wartości najpilniejszych spośród grupowanych zamówień — odrzucone, ponieważ cross-docking wymaga identycznego `slaDeadline` już na poziomie linii, więc agregat rozstrzygałby nieistniejący remis. Pozostawienie granicy nieokreśloną — odrzucone, ponieważ reguły zależne od tych atrybutów nie miałyby wskazanego źródła wartości.
   **Data:** 2026-08-25.
   **Reguły:** `P2 R43`.

24. **DEC-L55 — `sourceInboundTU` nie wraca jako atrybut Outbound `TU`:**
   **Decyzja:** atrybut `sourceInboundTU` nie powstaje na Outbound `TU`. Proweniencja pozostaje tam, gdzie jest dziś — na `CrossDockPickTask`. Archiwalny człon reguły wymagającej tej referencji jest świadomie niewprowadzany; pozycja zostaje zamknięta tą decyzją, nie odtworzeniem treści w korpusie.
   **Uzasadnienie:** kontrola kompletności GR per źródłowa Inbound `TU` nie potrzebuje tego atrybutu. `CrossDockPickTask` niesie jednocześnie `sourceInboundTU` i `grAcceptanceStatus`, a `DEC-L32` ustawia ten sam `grAcceptanceStatus` na wszystkich zadaniach o danym `sourceInboundTU` — systemowo, niezależnie od `Shipment`, także w topologii n:n. Agregacja zadań po `sourceInboundTU` daje pełną odpowiedź w jednym skoku, a `P2 R36` eksponuje to pole `Warehouse Supervisor`. Przejście przez `targetOutboundOrderLine` jest do tego celu zbędne. `DEC-L31` i `DEC-L32` są uzasadnieniem tej decyzji, nigdy dowodem wcześniejszej supersesji — żadna z nich nie rozstrzyga o atrybucie na Outbound `TU`.
   **Czego ta decyzja nie zamyka:** niedookreślone pozostaje wejście bramki `P2 R25`/`R33`. Wyznaczenie zbioru źródłowych Inbound `TU` danego `Shipment` wymaga przejścia `Shipment` → Outbound `TU` → zawartość → `OutboundOrderLine` → `CrossDockPickTask` → `sourceInboundTU`, a relacja zawartości Outbound `TU` z `OutboundOrderLine` nie istnieje w katalogu danych. Ten sam brak dotyczy `P1 R27`, `P1 R58` i `P1 R60`. Zagadnienie zostaje wyprowadzone do pozycji follow-up w `WMSAI_OUTBOUND/BACKLOG.md`.
   **Odrzucone alternatywy:** brak.
   **Data:** 2026-08-26.
   **Reguły:** brak reguł — decyzja o niewprowadzaniu zmiany.

25. **DEC-L58 — zawężenie `DEC-L38`: jawna korekta Supervisora nie jest objęta zakazem aktualizacji ilości:**
   **Decyzja:** zakaz z `DEC-L38`, w brzmieniu „nie istnieje mechanizm dogenerowania kolejnych `CrossDockPickTask` do już utworzonej `OutboundOrderLine` ani aktualizacji jej wymaganej ilości po utworzeniu", dotyczy automatycznego dogenerowania zadań i automatycznej aktualizacji ilości, a nie jawnej, nadzorowanej korekty wykonanej przez `Warehouse Supervisor` w gałęzi korekty `P2 R15`. Korekta ilości wymaganej crossdockowej `OutboundOrderLine`, wykonana razem z korektą `CustomerOrderLine.Quantity` przez `Warehouse Supervisor`, jest dopuszczalna. Pozostała treść `DEC-L38` obowiązuje bez zmian.
   **Uzasadnienie:** `DEC-L38` powstała, żeby zamknąć otwarty mechanizm dogenerowania zadań, dla którego nie było zdarzenia wyzwalającego ani kryterium zatrzymania. Jawna korekta Supervisora ma jedno i drugie: wyzwalaczem jest jego decyzja, a zakresem ta jedna linia. `P2 R15` już dziś przewiduje kontynuację tej samej linii do `PACKING_SEALED` po korekcie. Bez tego zawężenia korygowalność ilości wymaganej w gałęzi `P2 R15` byłaby sprzeczna z obowiązującym rejestrem.
   **Odrzucone alternatywy:** brak.
   **Data:** 2026-08-26.
   **Reguły:** `P2 R15`, `P2 R30`, `P1 R69`.

26. **DEC-L63 — rozłączność kanałów: `demandEligibleQty` odejmuje obie formy zabezpieczenia popytu:**
   **Decyzja:** `demandEligibleQty` dla `CustomerOrderLine` = `Quantity` minus `ATPReservation`, minus suma `requiredQty` wszystkich jej `OutboundOrderLine`, które nie są `CANCELLED`, niezależnie od kanału. Zastępuje człon `demandEligibleQty` z `DEC-L20`. `sourceEligibleQty` bez zmian.
   **Uzasadnienie:** `P1 R11` przenosi ilość z rezerwacji miękkiej do twardej, przez co poprzedni wzór — odejmujący wyłącznie `ATPReservation` — przestawał widzieć ilość już obiecaną. Przy `Quantity` 100, `ATPReservation` 0 i standardowej `OutboundOrderLine` o `requiredQty` 30 dawał 100 zamiast 70, czyli przyznawał trzydzieści sztuk drugi raz kanałem crossdockowym. Odjęcie obu form zamyka lukę bez wprowadzania nowego atrybutu. Przy okazji znika niezdefiniowana w korpusie fraza „brakująca ilość `CustomerOrderLine` w `BACKORDERED`" oraz osobny człon o przypisaniach innych `CrossDockPickTask`, wchłonięty przez sumę `requiredQty` — cross-dockowa `OutboundOrderLine` powstaje razem ze swoim zadaniem (`P2 R30`).
   **Odrzucone alternatywy:** odejmowanie sumy `reservedQty` alokacji w `SHORT`/`RESERVED`/`CONFIRMED` zamiast `requiredQty` — odrzucone, bo alokacja w `SHORT` trzyma mniej, niż linia obiecuje, więc cross-dock zabrałby ilość, którą kanał standardowy odzyska po uzupełnieniu zapasu; byłby to ten sam błąd w drugą stronę. Dostawienie członu o `Allocation` obok istniejącego `ATPReservation` — odrzucone, bo utrwalałoby dwie miary tego samego zabezpieczenia.
   **Data:** 2026-08-28.
   **Reguły:** `P2 R6`.

### 6.2a Shipment — kolejność `READY_FOR_DISPATCH` i granica anulowania

1. **DEC-L09 — `READY_FOR_DISPATCH` przed Carrier Selection:**
   **Decyzja:** `Shipment.READY_FOR_DISPATCH` przenosi się przed Carrier Selection (zamiast po `LABEL_GENERATED`) i oznacza zamknięcie grupowania `TU` w tym `Shipment`. Carrier Selection uruchamia się dopiero po osiągnięciu tego stanu.
   **Uzasadnienie:** R61 liczy `maxWeight`/`maxVolume` z Packing `TU` tego `Shipment` — dopóki zbiór `TU` nie jest zamknięty, wynik może być nieaktualny, jeśli kolejna `TU` dojdzie po uruchomieniu Carrier Selection.
   **Odrzucone alternatywy:** pozostawienie `READY_FOR_DISPATCH` po `LABEL_GENERATED` (dzisiejsza wersja) — odrzucone, bo nie gwarantuje kompletności zbioru `TU` przed liczeniem wagi/objętości; przemianowanie `POSTED` na `READY_FOR_DISPATCH` — rozważone i wycofane w dyskusji, `POSTED` zachowuje swoją nazwę i znaczenie.
   **Data:** 2026-08-11.
   **Reguły:** SP8, R61 (rozszerzenie), §4.9.
2. **DEC-L10 — konsolidacja `Shipment` wymaga identycznego `slaDeadline`, zamknięcie z timeoutem:**
   **Decyzja:** Konsolidacja wielu `OutboundOrder` w jednym `Shipment` (SP14/R20) wymaga identycznego `slaDeadline` wszystkich kontrybuujących `OutboundOrder`, bez tolerancji. `Shipment.READY_FOR_DISPATCH` osiągane jest też automatycznie po upłynięciu tego wspólnego `slaDeadline`, nawet jeśli nie wszystkie kontrybuujące `OutboundOrder` zdążyły się spakować — te, które nie zdążyły, tworzą nowy `Shipment`.
   **Uzasadnienie:** bez wspólnego `slaDeadline` i mechanizmu timeout `Shipment` mógłby czekać w nieskończoność na wolniej realizowany `OutboundOrder`, blokując wysyłkę tych, które są już gotowe.
   **Odrzucone alternatywy:** tolerancja `slaDeadline` jak w R1 — odrzucona, bo to etap zamykania okna wysyłki, nie planowania; czekanie bez limitu czasu — odrzucone jako ryzyko operacyjne.
   **Data:** 2026-08-11.
   **Reguły:** SP8, SP14 (rozszerzenie).
3. **DEC-L11 — granica anulowania `Shipment` po `POSTING_PENDING`:**
   **Decyzja:** `Shipment` można anulować wyłącznie przed wejściem w `POSTING_PENDING` (z `CREATED`/`READY_FOR_DISPATCH`/`CARRIER_SELECTED`/`OWN_TRANSPORT`/`CARRIER_PENDING`/`LABEL_GENERATED`) albo z `POSTING_ERROR`. Od `POSTING_PENDING` włącznie anulowanie przez WMS (w tym R65 gałąź `PACKED`) jest niemożliwe; dalsza obsługa to zwykły zwrot towaru (Return Receipt) w ramach przyszłego procesu obsługi po wydaniu, nie korekta dokumentu `Shipment` w WMS.
   **Uzasadnienie:** ERP jest zleceniodawcą (patrz uzasadnienie R54/krok 11a) — po zgłoszeniu do ERP korekta powinna iść analogicznym torem co inne zdarzenia po wydaniu, nie przez wewnętrzny mechanizm anulowania WMS.
   **Odrzucone alternatywy:** korekta dokumentu `Shipment` bezpośrednio w ERP z nowym kanałem integracyjnym do WMS — odrzucone na tym etapie jako przedwczesne; mechanizm zaprojektowany zostanie w ramach przyszłego Return Receipt.
   **Data:** 2026-08-11.
   **Reguły:** R55, R65 (zawężenie gałęzi `PACKED`).
4. **DEC-L12 — `OutboundOrder.PACKED` jako pełny agregat:**
   **Decyzja:** `OutboundOrder.PACKED` oznacza pełny agregat — wszystkie jego `OutboundOrderLine` i wszystkie ich `TU` są spakowane, spójnie z wzorcem SP3 (`PICKED`). `PACKED`→`READY_FOR_DISPATCH` jest przejściem natychmiastowym, bez dodatkowego warunku, i służy jako sygnał konsumowany przez `Shipment` (SP8).
   **Uzasadnienie:** spójność z istniejącym wzorcem agregacji `OutboundOrder` (SP2–SP4) — stan zbiorczy odzwierciedla pełne domknięcie, nie częściowy postęp.
   **Odrzucone alternatywy:** `PACKED` przy pierwszej gotowej `TU` (częściowy postęp) — odrzucone jako niespójne z SP3 i mylące dla konsumentów tego statusu.
   **Data:** 2026-08-11.
   **Reguły:** §4.3, SP8 (zależność).
5. **DEC-L22 — `POSTING_ERROR` z niezgodności zawartości, dwie ścieżki naprawy:**
   **Decyzja:** Gdy `Shipment POSTING_ERROR` wynika z niezgodności zawartości (ustrukturyzowany powód odrzucenia ERP, odróżnialny od awarii technicznej komunikacji), przyczyna leży po jednej z dwóch stron: (a) po stronie ERP (np. brak ceny towaru, rozjazd stanu ATP między WMS a ERP) — retry z niezmienioną treścią kończy się sukcesem po naprawie w ERP; (b) po stronie WMS (błędne dane `CustomerOrder`/`Shipment`) — korekta poprzedza retry, wprowadzana ręcznie przez `Warehouse Supervisor` albo przez OMS/ERP przez webservice (jak `R65`). W obu przypadkach ponowienie zgłoszenia jest osobną, ręczną decyzją `Warehouse Supervisor` (`POSTING_ERROR→POSTING_PENDING`, `R54`); fizyczna zawartość spakowanej `TU` i stan `OutboundOrderLine`/`PACKED` nie są ruszane w żadnym wariancie.
   **Uzasadnienie:** ERP nigdy nie widzi fizycznej zawartości `TU` — każde jego odrzucenie jest z definicji rozbieżnością na poziomie danych WMS↔ERP, nie stwierdzeniem błędu fizycznego pakowania; rozróżnienie strony winnej (ERP vs WMS) determinuje, czy retry wymaga wcześniejszej korekty danych, czy nie.
   **Odrzucone alternatywy:** modelowanie osobnej ścieżki dla „fizycznej niezgodności” wykrytej na etapie `POSTING_ERROR` — odrzucone, bo ta bramka strukturalnie nie może takiej niezgodności wykryć (wymagałoby fizycznego otwarcia zapieczętowanej `TU`, poza zakresem tego mechanizmu).
   **Data:** 2026-08-12.
   **Reguły:** `propozycja_procesow_outbound.md` — `R80`, §3.1 krok 11a, §4.9.

6. **DEC-L23 — bramka GR per Inbound `TU` dla `Shipment` cross-dockowego (ICR-05):**
   [HISTORYCZNE — doprecyzowane przez `R81`/`DEC-L25` (2026-08-14)] Zdanie w „Decyzja" poniżej wymieniające „identyfikator `CrossDockPickTask`" jako element wymaganej treści sygnału GR jest nieaktualne — `R81` koreluje sygnał wyłącznie po `sourceInboundTU`/`GR_SETTLEMENT_SOURCE`, bez identyfikatora zadania (patrz `R81`).
   [HISTORYCZNE — zawężone przez `DEC-L40` (2026-08-22)] Zdanie „`GR_REJECTED` dla którejkolwiek wymaganej Inbound `TU` przenosi `Shipment` do `POSTING_ERROR` jako jawne odrzucenie w istniejącej bramce; ponowienie pozostaje ręczną decyzją `Warehouse Supervisor`" w polu „Decyzja" poniżej jest nieaktualne — bramka jest ponownie ewaluowana per komunikat GR i `GR_REJECTED` nie powoduje samodzielnie `POSTING_ERROR`, patrz `DEC-L40`.
   **Decyzja:** bramka ERP `Shipment` cross-dockowego przestaje czekać na `POSTED` całego źródłowego ASN. `Shipment` w `POSTING_PENDING` wysyła `POST` dopiero po zarejestrowaniu `GR_ACCEPTED` dla każdej źródłowej Inbound `TU`, która przez `CrossDockPickTask` dostarczyła `confirmedQty` do dowolnej Outbound `TU` tego `Shipment`. W n:n nie wystarcza akceptacja GR dla jednego bezpośrednio powiązanego `CrossDockPickTask`: `Shipment` czeka na pełny zbiór źródłowych Inbound `TU` wnoszących do niego ilość. WMS przechowuje na `CrossDockPickTask` atrybut `grAcceptanceStatus` z wartościami `GR_PENDING`, `GR_ACCEPTED` i `GR_REJECTED`. Sygnał konsumowany przez Outbound musi wskazywać: identyfikator Inbound `TU`, identyfikator `CrossDockPickTask` oraz wynik akceptacja/odrzucenie. Niezgodność wskazanej Inbound `TU` z `sourceInboundTU` zadania jest odrzucana jako niespójna. `GR_REJECTED` dla którejkolwiek wymaganej Inbound `TU` przenosi `Shipment` do `POSTING_ERROR` jako jawne odrzucenie w istniejącej bramce; ponowienie pozostaje ręczną decyzją `Warehouse Supervisor`.

   **Uzasadnienie:** elementarna Inbound `TU` rozliczona przez cross-docking ma niezależny wynik GR w ERP, więc oczekiwanie na Putaway i GR pozostałych `TU` tego samego ASN nie odzwierciedla już stanu ilości wysyłanej przez `Shipment`. Warunek pełnego zbioru źródłowych `TU` zachowuje kontrolę księgową również wtedy, gdy rozsortowanie łączy wiele źródeł w jednej Outbound `TU` albo rozdziela jedno źródło na wiele Outbound `TU`.

   **Odrzucone alternatywy:** oczekiwanie na `POSTED` całego ASN — odrzucone, ponieważ niweluje przewagę czasową cross-dockingu; odblokowanie `Shipment` po `GR_ACCEPTED` tylko jednej Inbound `TU`/jednego `CrossDockPickTask` — odrzucone, ponieważ przy n:n pozwala wysłać `POST` przed akceptacją GR dla części fizycznej zawartości `Shipment`; odblokowanie po samym zakończeniu `CrossDockPickTask` — odrzucone, ponieważ nie potwierdza akceptacji GR przez ERP.

   **Data:** 2026-08-12.

   **Reguły:** `propozycja_procesow_outbound.md` — `R54`, `R81`, §4.13 (`CrossDockPickTask.grAcceptanceStatus`).

7. **DEC-L24 — GR dwuetapowe dla Inbound `TU` rozliczanej częściowo przez cross-docking:**
   **Decyzja:** Gdy Inbound `TU` rozliczana przez cross-docking pozostawia rezydualną ilość i przechodzi w `IN_PUTAWAY` (`R77`, bez zmian), rozliczenie GR tej `TU` odbywa się w dwóch niezależnych krokach: rozliczenie crossdockowe (ilość już potwierdzona przez zakończone `CrossDockPickTask`, w momencie przejścia z `IN_CROSS_DOCK`) i rozliczenie putawayowe (rezydualna ilość, w momencie zakończenia standardowego Putaway tej samej `TU`). Oba rozliczenia razem pokrywają całą zadeklarowaną (ASN) ilość `OK` tej `TU`, bez luki i bez nakładania. Bramka `Shipment` (`R54`) czeka wyłącznie na wynik rozliczenia crossdockowego.
   **Uzasadnienie:** wymuszanie pełnego fizycznego rozsortowania `TU` przed jakimkolwiek GR (rozważony wcześniej wariant z nową jednostką logistyczną dla resztki) rozwiązuje ten sam problem SLA kosztem dodatkowej pracy operatora na stacji crossdock i nowego obiektu w modelu. Rozbicie samego rozliczenia GR na dwa niezależne kroki dla tej samej Inbound `TU` osiąga ten sam efekt bez zmiany fizycznego procesu sortowania — `Warehouse Operator` pracuje dokładnie jak dziś. Dokument `PZ` już dziś agreguje wiele `TU` w jeden dokument dla `ASN` (Inbound); rozszerzenie o kolejną wersję rozliczenia tej samej `TU` jest tym samym mechanizmem agregacji, większą liczbą komunikatów, bez nowej logiki integracyjnej.
   **Odrzucone alternatywy:** pełne fizyczne rozsortowanie źródłowej `TU` przed każdym GR, z nową jednostką logistyczną (`InternalTU`) dla rezydualnej ilości — odrzucone jako cięższe zarówno operacyjnie (dodatkowa praca `Warehouse Operator` na stacji crossdock), jak i architektonicznie (nowy obiekt, nowa tożsamość `TU_ID` w Inbound) bez przewagi nad rozbiciem samego rozliczenia GR.
   **Data:** 2026-08-13.
   **Reguły:** `propozycja_procesow_outbound.md` — `R54` (doprecyzowanie), §3.2 krok 3/4.
8. **DEC-L25 — kontrakt sygnału GR ze wskazaniem rozliczenia:**
   **Decyzja:** sygnał `GR_ACCEPTED`/`GR_REJECTED` konsumowany przez Outbound musi jednoznacznie wskazywać, że dotyczy rozliczenia crossdockowego danej Inbound `TU`, nie jej ewentualnego późniejszego rozliczenia putawayowego. System WMS reaguje wyłącznie na sygnał dotyczący rozliczenia crossdockowego; sygnał dotyczący rozliczenia putawayowego tej samej `TU` nie jest powiązany z żadnym `CrossDockPickTask` i nie zmienia `grAcceptanceStatus`.
   **Uzasadnienie:** bez tego rozróżnienia System WMS nie mógłby odróżnić dwóch niezależnych wyników GR dla tej samej Inbound `TU` — ryzyko błędnego powiązania sygnału putawayowego z zadaniem, które już dawno zasiliło wysłany `Shipment`.
   **Odrzucone alternatywy:** poleganie na kolejności odbioru sygnałów (pierwszy odebrany = crossdockowy) — odrzucone jako niepewne przy braku gwarancji kolejności dostarczenia komunikatów.
   **Data:** 2026-08-13.
   **Reguły:** `propozycja_procesow_outbound.md` — `R81` (rozszerzenie; doprecyzowane 2026-08-14 po odpowiedzi Inbound `D48` — dyskryminator to `GR_SETTLEMENT_SOURCE`, nie numer wersji, bo wersje liczą się osobno per źródło rozliczenia).
9. **DEC-L26 — brak zmian w `R77`/§3.2 krok 3, brak nowej jednostki logistycznej:**
   **Decyzja:** rezydualna ilość pozostała po cross-dockingu nadal trafia na `TRANSIT`, a źródłowa Inbound `TU` nadal przechodzi w `IN_PUTAWAY` (`R77`, bez zmian), skąd przechodzi przez standardowy Putaway bez modyfikacji tego procesu. Nie wprowadza się nowej jednostki logistycznej dla rezydualnej ilości.
   **Uzasadnienie:** rozbicie rozliczenia GR (`DEC-L24`) już rozwiązuje problem SLA; utrzymanie tej samej `TU_ID` przez cały cykl upraszcza traceability względem wariantu z nową jednostką i nie wymaga zmiany pracy `Warehouse Operator` na stacji crossdock.
   **Odrzucone alternatywy:** patrz `DEC-L24`.
   **Data:** 2026-08-13.
   **Reguły:** brak nowej reguły — `R77` pozostaje bez zmian.

10. **DEC-L27 — pole `damagedQty` w kontrakcie crossdockowym z Inbound:**
   **Decyzja:** `CrossDockPickTask` zyskuje atrybut `damagedQty` — ilość SKU zgodnej z deklaracją TU/SKU, wykryta jako `DAMAGED` podczas pobrania (`R67`) i odłożona na `QC`, wyłączona z `confirmedQty`. Suma `damagedQty` wszystkich powiązanych `CrossDockPickTask` danej Inbound `TU` jest przekazywana Inbound w tym samym momencie i kanałem co dzisiejsze potwierdzenie rozliczenia crossdockowego (`R77`, §3.2 krok 3). Nie obejmuje `SKU` nieoczekiwanego (`R68`).
   **Uzasadnienie:** towar `DAMAGED` wykryty podczas cross-dockingu fizycznie opuszcza Inbound `TU` (trafia na `QC`), ale nie był dotąd widoczny w żadnym polu przekazywanym Inbound — formuła Inbound liczącą ilość rezydualną po cross-dockingu (`declaredQty − confirmedQty`) fałszywie liczyła tę ilość jako brakującą (`MISSING`) przy kontroli ilościowej Putaway. Zgłoszone jako prośba integracyjna z równoległej sesji Inbound (`D50`, `../MEMORY.md`, 2026-08-15), zweryfikowana bezpośrednio względem naszych plików przed przyjęciem.
   **Odrzucone alternatywy:** brak — czysto naprawcze rozszerzenie istniejącego kontraktu (analogicznego do `confirmedQty`/`grAcceptanceStatus`), nie nowy model.
   **Data:** 2026-08-15.
   **Reguły:** `propozycja_procesow_outbound.md` — nowa `R83`, §4.13, §3.2 krok 2/3, §6.8.

### 6.2b Outbound Crossdock — fulfillmentChannel, PICKING, wielokrotna Outbound TU

1. **DEC-L13 — `fulfillmentChannel` + SP19/SP20 + diagram nagłówka `OutboundOrder`:**
   **Decyzja:** `OutboundOrder` otrzymuje atrybut `fulfillmentChannel` (`STANDARD`/`CROSSDOCK`), ustawiany przy tworzeniu, niezmienny; wszystkie `OutboundOrderLine` jednego `OutboundOrder` mają ten sam `fulfillmentChannel` (SP19). Dla `CROSSDOCK` nagłówek pomija `ALLOCATION_IN_PROGRESS`/`ALLOCATED`/`PICKING_IN_PROGRESS`/`PICKED` i wchodzi wprost w `PACKING_IN_PROGRESS` po pierwszym `CrossDockPickTask.IN_PROGRESS` (SP20); wyjście do `PACKED` dzieli z flow standardowym. Nowa krawędź `PACKING_IN_PROGRESS`→`CANCELLED` dla przypadku, gdy wszystkie `OutboundOrderLine` cross-dockowe zostały `CANCELLED` przez R66/R67, żadna nie osiągnęła `PACKED` (rozszerzenie R58).
   **Uzasadnienie:** SP18 pokrywał wyłącznie brak `Allocation` na poziomie linii; header nie miał dotąd żadnej ścieżki dla crossdocku w §4.3. `PACKING_IN_PROGRESS` wybrano zamiast `PICKING_IN_PROGRESS`, bo rozsortowanie cross-dockowe jest formą pakowania do docelowej `TU` wysyłkowej, nie zbierania do `TU` dalej pakowanej. Krawędź do `CANCELLED` ograniczona wyłącznie do przypadku R66/R67 (już kontrolowanego, bez ryzyka lokalizacji towaru) — nie do ogólnego anulowania, które w tym stanie pozostaje zablokowane (DEC-L17).
   **Odrzucone alternatywy:** brak krawędzi `CANCELLED` z `PACKING_IN_PROGRESS` w ogóle — odrzucone, header zostawałby trwale zawieszony przy wyzerowaniu wszystkich linii; `PICKING_IN_PROGRESS` jako stan crossdockowy zamiast `PACKING_IN_PROGRESS` — odrzucone na rzecz trafniejszej semantyki pakowania.
   **Data:** 2026-08-11.
   **Reguły:** SP19, SP20, R58 (rozszerzenie), §4.3.
2. **DEC-L14 — `crossDockEligibleQty`:**
   **Decyzja:** Ilość kwalifikowalna do cross-dockingu dla `CustomerOrderLine` (`crossDockEligibleQty`) = min(ilość dostępna w Inbound `TU`, brakująca ilość `CustomerOrderLine` w `BACKORDERED`); wyliczana przy generowaniu `CrossDockPickTask` (§3.2 krok 1). Zapisana jako R76 z adnotacją „Invariant:", bez nowej sekcji dokumentu.
   **Uzasadnienie:** prosty niezmiennik matematyczny wynikający z już opisanego mechanizmu (R70, krok 1), nie nowa decyzja procesowa.
   **Odrzucone alternatywy:** osobna sekcja „Niezmienniki ilościowe" w dokumencie — odrzucona jako przedwczesna dla pojedynczego wzoru.
   **Data:** 2026-08-11.
   **Reguły:** R76 (nowa).
3. **DEC-L15 — częściowe rozliczenie `CrossDockPickTask`:**
   **Decyzja:** Gdy `CrossDockPickTask` kończy się z `confirmedQty < plannedQty` (brak/`DAMAGED` wykryte w trakcie pobrania albo przy zakończeniu, R66/R67): dla `allowPartialShipment=true` — automatycznie, rozsortowana część `OutboundOrderLine`→`PACKED`, brakująca `CustomerOrderLine`→`BACKORDERED`. Dla `allowPartialShipment=false` — eskalacja do `Warehouse Supervisor`, trzy wyniki analogiczne do `SHORT_PICKED` (§3.5): „czekamy" (`BACKORDERED`), „anulowanie" (rozsortowana część [HISTORYCZNE — zastąpione przez DEC-L21: nie wraca do Inbound `TU`/`IN_PUTAWAY`, tylko `PutBackTask`→`Inventory AVAILABLE`], cała linia `CANCELLED`), „trwała zmiana `allowPartialShipment` na `true`".
   **Uzasadnienie:** crossdock nie ma zapasu do realokacji (brak `Inventory`, DEC-L07), więc automatyczna realokacja z `SHORT_PICKED` nie ma odpowiednika — przenosi się wyłącznie część eskalacyjna. Wariant „anulowanie" reużywa istniejący mechanizm resztki n:n (R72) zamiast `PutBackTask`, bo nie ma lokalizacji źródłowej do odłożenia [HISTORYCZNE — zastąpione przez `DEC-L21`: odzysk `PutBackTask`→`Inventory AVAILABLE`, patrz pole „Decyzja" powyżej].
   **Odrzucone alternatywy:** automatyczne zachowanie rozsortowanej części jako `PACKED` niezależnie od `allowPartialShipment` — odrzucone, dla `allowPartialShipment=false` klient nie zgodził się na częściową realizację.
   **Data:** 2026-08-11.
   **Reguły:** R66/R67 (rozszerzenie o gałąź `allowPartialShipment`), §3.2 krok 2.
4. **DEC-L16 — wielokrotna Outbound `TU` per `CrossDockPickTask`:**
   **Decyzja:** Jeden `CrossDockPickTask` może użyć kilku Outbound `TU` po zapełnieniu poprzedniej — analogicznie do R14 dla `PickTask`/Picking `TU`. `Warehouse Operator` zamyka pełną `TU` (`PACKING_SEALED`) przez skan RF i kontynuuje to samo zadanie do nowej, otwartej Outbound `TU`. Normalny krok procesu (§3.2 krok 2), nie eskalacja Supervisora.
   **Uzasadnienie:** R14 już ustanawia dokładnie ten wzorzec dla standardowego flow — rozszerzenie zamiast nowej reguły jest spójniejsze. Aktor to `Warehouse Operator` (skan RF), nie `Warehouse Supervisor` — to fizyczne ograniczenie pojemności kontenera, nie wyjątek.
   **Odrzucone alternatywy:** ręczne zamknięcie `TU` przez `Warehouse Supervisor` z polami audytowymi (`closedBy`/`closedAt`/`closeReason`) i powrotem niezrealizowanej ilości jako outstanding demand — odrzucone jako nadmiarowa komplikacja.
   **Data:** 2026-08-11.
   **Reguły:** R14 (rozszerzenie), §3.2 krok 2, §4.13.
5. **DEC-L17 — status `PICKING` dla `OutboundOrderLine` cross-dockowej i blokada anulowania:**
   **Decyzja:** `OutboundOrderLine` cross-dockowa otrzymuje pośredni status `PICKING` między `CREATED` i `PACKED`: `CREATED`→`PICKING` przy `CrossDockPickTask`→`IN_PROGRESS`, `PICKING`→`PACKED` po rozsortowaniu. `CREATED`→`CANCELLED` zostaje wyłącznie dla `TU` pustej przed pobraniem; brak/`DAMAGED` w trakcie przechodzi `PICKING`→`CANCELLED` (R66/R67). Ogólne anulowanie (R65) niedozwolone w `PICKING` — System WMS odrzuca żądanie, do ponowienia po `PACKED`.
   **Uzasadnienie:** R73 ukrywał postęp fizyczny wewnątrz `CREATED` — R65 kluczowany po statusie `OutboundOrderLine` nie odróżniał „zadanie nieprzypisane" od „Packer w trakcie skanowania". W trakcie rozsortowania nie da się jednoznacznie ustalić, gdzie fizycznie znajduje się SKU — cofnięcie w toku jest ryzykowne, dlatego blokada zamiast automatycznego put-backu, analogicznie do ograniczenia już przyjętego dla `PACKED`.
   **Odrzucone alternatywy:** rozszerzenie tej samej blokady na standardowy flow (`PICKING`/`PICKED`) — rozważone i wycofane, dotyczy wyłącznie crossdocku; zgoda Supervisora + reużycie `IN_PUTAWAY` dla anulowania w trakcie `PICKING` (analogicznie do DEC-L15) — odrzucone na rzecz prostszej twardej blokady, bo to inny wyzwalcz (żądanie zewnętrzne, nie stwierdzony brak).
   **Data:** 2026-08-11.
   **Reguły:** R65 (nowa gałąź cross-dock), R73 (przeredagowanie), §3.5 punkt 5 (nowy), §4.4.
6. **DEC-L18 — kryterium `CROSS_DOCKED`/`IN_PUTAWAY` oparte na rezydualnej ilości:**
   **Decyzja:** Inbound `TU` przechodzi do `CROSS_DOCKED`, gdy cała zadeklarowana (ASN) zawartość `TU` została rozliczona przez cross-docking, niezależnie od topologii 1:1, 1:n, n:1 albo n:n. Jeżeli po zakończeniu wszystkich powiązanych `CrossDockPickTask` pozostaje rezydualna, nieprzypisana fizyczna ilość `SKU`, resztka trafia do `TRANSIT`, a Inbound `TU` przechodzi do `IN_PUTAWAY`.
   **Uzasadnienie:** liczba docelowych Outbound `TU` opisuje topologię sortowania, nie stan fizycznie nierozliczonej ilości źródłowej `TU`.
   **Odrzucone alternatywy:** kryterium oparte wyłącznie na topologii 1:1 albo n:n — odrzucone, ponieważ powoduje `IN_PUTAWAY` mimo braku rezydualnej ilości przy pełnym rozliczeniu 1:n, n:1 lub n:n.
   **Data:** 2026-08-11.
   **Reguły:** `R77`.
7. **DEC-L19 — zamek ilościowy na `CrossDockPickTask`:**
   [HISTORYCZNE — zawężone przez `DEC-L38` (2026-08-22)] Zdanie „Jedna `OutboundOrderLine` może agregować ilość z wielu `CrossDockPickTask` pochodzących z różnych Inbound `TU`" w polu „Decyzja" poniżej jest nieaktualne — generowanie jest jednorazowe i zamknięte, patrz `DEC-L38`. Zamek ilościowy na `CrossDockPickTask` pozostaje bez zmian.
   **Decyzja:** zamek ilościowy cross-dockingu znajduje się na `CrossDockPickTask`, nie na `OutboundOrderLine`. `plannedQty` rezerwuje ilość źródłowej Inbound `TU`/`SKU` do potwierdzenia, a ta sama fizyczna ilość nie może być równocześnie `planned` ani `confirmed` w innym aktywnym `CrossDockPickTask`. Jedna `OutboundOrderLine` może agregować ilość z wielu `CrossDockPickTask` pochodzących z różnych Inbound `TU`.
   **Uzasadnienie:** `OutboundOrderLine` agreguje popyt i potwierdzenia, ale bez blokady na źródłowej `TU`/`SKU` nie chroni przed podwójnym przypisaniem tej samej fizycznej ilości.
   **Odrzucone alternatywy:** aktywna `OutboundOrderLine` jako jedyny zamek ilościowy — odrzucone, ponieważ nie rozróżnia dwóch zadań konkurujących o tę samą ilość źródłową.
   **Data:** 2026-08-11.
   **Reguły:** `R71` (rewizja), `R78`, `R79`.
8. **DEC-L20 — rewizja `crossDockEligibleQty`:**
   **Decyzja:** `crossDockEligibleQty` dla `CustomerOrderLine` = min(`sourceEligibleQty`, `demandEligibleQty`). `sourceEligibleQty` pomniejsza zadeklarowaną (ASN) ilość źródłowej Inbound `TU`/`SKU` o już aktywnie zaplanowane i zakończone potwierdzone ilości; `demandEligibleQty` pomniejsza brakującą ilość `CustomerOrderLine` w `BACKORDERED` o ilości już przypisane przez `CrossDockPickTask` i o `ATPReservation` ze standardowego flow. Zastępuje wcześniejszy wzór z `DEC-L14`.
   **Uzasadnienie:** wzór musi uwzględniać już przypisane ilości po stronie źródła i popytu, aby nie dopuścić do nadmiarowego przypisania.
   **Odrzucone alternatywy:** min(pełna ilość dostępna w Inbound `TU`, pełny brak `CustomerOrderLine`) — odrzucone, bo pomija istniejące przypisania `CrossDockPickTask` i `ATPReservation`.
   **Data:** 2026-08-11.
   **Reguły:** `R76` (rewizja).
   **Nota (2026-08-28):** człon `demandEligibleQty` tej decyzji został zastąpiony przez `DEC-L63`; pozostała treść bez zmian. Wpis pozostaje zapisem historycznym i nie jest przepisywany.
9. **DEC-L21 — odzysk `confirmedQty` po utracie demand:**
   **Decyzja:** gdy potwierdzona ilość `confirmedQty` cross-dockowego `CrossDockPickTask` traci demand w ścieżce „czekamy" albo pełnego anulowania przy `allowPartialShipment = false`, System WMS tworzy `PutBackTask`; po jego `COMPLETED` ilość trafia na zwalidowaną lokalizację jako zwykły `Inventory AVAILABLE` (`ATP`). Ta ilość nie wraca do Inbound `TU` ani `IN_PUTAWAY`. Stanowi to wąski wyjątek od `DEC-L07`, ograniczony wyłącznie do ścieżki odzysku po utracie demand; happy path `DEC-L07` pozostaje bez zmian.
   **Uzasadnienie:** po rozsortowaniu i utracie popytu towar wymaga fizycznego, śledzalnego odzysku do standardowego zapasu; ponowne włączenie do Inbound `TU` nie odzwierciedla jego rzeczywistego położenia.
   **Odrzucone alternatywy:** pozostawienie ilości w Outbound `TU` bez zadania odzysku albo zwrot do Inbound `TU`/`IN_PUTAWAY` — odrzucone, ponieważ nie zapewniają fizycznego rozliczenia po utracie demand.
   **Data:** 2026-08-11.
   **Reguły:** `R65`, `R71`, `SP18`, §3.2 krok 2, §4.12.

10. **DEC-L28 — `sourceEligibleQty` uwzględnia `damagedQty`:**
   **Decyzja:** wzór `sourceEligibleQty` z `R76` odejmuje od zadeklarowanej (ASN) ilości Inbound `TU`/`SKU`, obok `plannedQty` aktywnych `CrossDockPickTask`, sumę `confirmedQty` i `damagedQty` zakończonych `CrossDockPickTask` — nie samo `confirmedQty` jak dotąd. Zastępuje wzór z `DEC-L20`.
   **Uzasadnienie:** ilość `DAMAGED` zakończonego `CrossDockPickTask` fizycznie opuszcza źródłową Inbound `TU` (trafia na `QC`, `damagedQty`, wprowadzone `DEC-L27`), ale dotychczasowy wzór `sourceEligibleQty` jej nie odejmował — System WMS mógłby błędnie liczyć tę ilość jako wciąż dostępną do zaplanowania nowego `CrossDockPickTask`, mimo że fizycznie już jej tam nie ma. Ten sam typ błędu co zgłoszony przez równoległą sesję Inbound (`D50`) dla ich własnej formuły, znaleziony przez Outbound przy weryfikacji spójności po wdrożeniu `DEC-L27`.
   **Odrzucone alternatywy:** brak — czysto korygujące doprecyzowanie istniejącego wzoru, nie nowy model.
   **Data:** 2026-08-15.
   **Reguły:** `propozycja_procesow_outbound.md` — `R76` (rewizja).

### 6.2c Podejmowanie zadań magazynowych — wycofanie puli operatorów (PickTask, CrossDockPickTask, PutBackTask)

1. **DEC-L41 — Wycofanie konceptu puli operatorów i modułu `WMSAI_ADM`:**
   **Decyzja:** koncept konfigurowalnej puli operatorów per typ zadania magazynowego (obiekt `WarehouseTaskTypeOperatorAssignment`, ranga priorytetu typów, przypisywanie operatorów przez `Warehouse Supervisor`) zostaje wycofany w całości. Moduł `WMSAI_ADM` wraz z `decyzje_adm.md` (`DEC-ADM-01`–`DEC-ADM-08`) i `wymagania_operator_pool.md` trafia do `../Archiwum/` i przestaje być źródłem. Wycofane zostają: `DEC-L08`, `P2 R38`, `FR-P2-20`, `TC-095`.
   **Uzasadnienie:** właściciel produktu ocenił koncept jako nieoperacyjny. Problem, który miał rozwiązywać — który typ zadania ma pierwszeństwo dla operatora kwalifikującego się do kilku — okazał się pozorny: znika, gdy typ pracy wybiera sam operator, wchodząc do właściwego modułu terminala RF. Mechanizm konfiguracji komplikował algorytm zamiast go upraszczać.
   **Odrzucone alternatywy:** utrzymanie puli w zawężonym zakresie; zachowanie samej reguły pierwszeństwa `CrossDockPickTask` nad `PickTask` w innej formie — bezprzedmiotowe, bo operator jest w danej chwili w jednym module.
   **Data:** 2026-08-23.

2. **DEC-L42 — Podejmowanie zadań przez wybór modułu pracy w RF:**
   **Decyzja:** `Warehouse Operator` wybiera typ pracy, wchodząc do modułu na terminalu RF (zbieranie / crossdocking / zwroty). W module zbierania wskazuje dodatkowo jedną strefę. System WMS przydziela mu kolejne zadanie danego typu zgodnie z obowiązującą kolejnością, wyłącznie gdy operator nie ma aktywnego zadania magazynowego żadnego typu. Zadanie powstaje w `CREATED` i czeka na operatora. Operator musi zakończyć zadanie, żeby opuścić moduł.
   **Uzasadnienie:** przydział bez konfiguracji i bez udziału `Warehouse Supervisor`; wybór typu pracy należy do operatora, który wie, gdzie fizycznie jest i co robi.
   **Odrzucone alternatywy:** wybór kilku stref naraz (system wysyłałby operatora przez cały magazyn); warunek „brak aktywnego zadania tego typu" zamiast „żadnego typu" (operator z dwoma otwartymi pojemnikami); zawieszanie zadania przy wyjściu z modułu — odłożone do wspólnego `BACKLOG.md`, Etap 2.
   **Data:** 2026-08-23.
   **Reguły:** `proces_1_standard_fulfillment.md` — `R54`, `R56`; `proces_2_outbound_crossdock.md` — `R39`; `proces_4_physical_putback.md` — `R9`.

3. **DEC-L43 — Picking `TU` niezależna od pojedynczego `PickTask`:**
   **Decyzja:** Picking `TU` może zbierać towar z kolejnych `PickTask` tego samego `OutboundOrder` w różnych strefach. Zakończenie `PickTask` nie zamyka `TU` — zamyka ją decyzja operatora albo osiągnięcie limitu (`PICK_FULL`). Wybór kontynuacji zamówienia w innej strefie przydziela wskazane zadanie niezależnie od kolejności priorytetowej i od strefy operatora, a strefa operatora staje się strefą tego zadania. Deklaracja `directPackDeclared` jest wiążąca dla całej `TU`, nie dla pojedynczego `PickTask`.
   **Uzasadnienie:** przy zbieraniu bezpośrednio do opakowania wysyłkowego zamykanie pojemnika na granicy strefy byłoby bezcelowe. Rozdzielenie cyklu życia `TU` od cyklu życia zadania usuwa sklejenie, które istniało tylko dlatego, że dotąd jedna `TU` odpowiadała jednemu zadaniu.
   **Data:** 2026-08-23.
   **Reguły:** `proces_1_standard_fulfillment.md` — `R13`, `R16`, `R55`.

4. **DEC-L44 — Brak wskazania miejsca pracy w module crossdockingu (decyzja negatywna):**
   **Decyzja:** moduł crossdockingu świadomie nie ma odpowiednika strefy. Operator wchodzi do modułu i dostaje kolejne zadanie, bez zawężenia do miejsca rozsortowania.
   **Uzasadnienie:** Outbound nie modeluje fizycznego umiejscowienia rozsortowania — `IN_CROSS_DOCK` należy do kanonu Inbound, a `TRANSIT` występuje w `proces_2_outbound_crossdock.md` wyłącznie jako miejsce przekazania resztki. Wprowadzenie zawężenia wymagałoby nowego bytu i sięgnięcia po dane Inbound (granica `D41`), żeby rozwiązać problem niezgłoszony przez nikogo.
   **Zakres nierozstrzygnięty:** magazyn z kilkoma miejscami rozsortowania wymaga osobnej, świadomej decyzji.
   **Data:** 2026-08-23.
   **Reguły:** `proces_2_outbound_crossdock.md` — `R39`.

5. **DEC-L45 — `PutBackTask` objęty tym samym modelem:**
   **Decyzja:** `PutBackTask` jest podejmowany przez wejście do własnego modułu zwrotów w RF, bez wskazania strefy, w kolejności zgłoszenia zadań.
   **Uzasadnienie:** trzeci typ zadania magazynowego w Outbound miał tę samą nienazwaną lukę; pozostawienie go bez mechanizmu tworzyłoby asymetrię zgłaszaną przez każdy kolejny audyt. Kolejność zgłoszenia, a nie priorytet, bo `PutBackTask` nie niesie `priority` ani `slaDeadline` (`model_stanow_outbound.md` §12), a pozycja, której dotyczy, jest już anulowana.
   **Odrzucone alternatywy:** włączenie `PutBackTask` do modułu zbierania — przywróciłoby problem „który typ zadania ma pierwszeństwo", zlikwidowany przez `DEC-L42`.
   **Data:** 2026-08-23.
   **Reguły:** `proces_4_physical_putback.md` — `R9`.

### 6.3 Picking `TU` i Packing `TU`

1. `PickContainer` i `PackUnit` są rolami Outbound `TU` w procesie.
2. Każde kompletowanie odbywa się do Picking `TU` (`PickContainer`).
3. Typ `TU` określa podstawową możliwość wykorzystania jednostki do wysyłki.
4. System dodatkowo ocenia konkretną jednostkę i sugeruje pozostawienie, przepakowanie albo konsolidację.
5. Ostateczną decyzję podejmuje `Warehouse Operator (Packer)`.
6. Odstępstwo od sugestii systemu wymaga zgody `Warehouse Supervisor`.
7. Progi masy i wykorzystania objętości są konfigurowane per typ `TU`. [ZAWĘŻONE — patrz `DEC-L50`] Próg objętościowy jest progiem bezwzględnym objętości zawartości (`TUSetup.minIssueVolume`), nie udziałem wykorzystania objętości.
8. Jeżeli Picking `TU` spełnia warunki wydania, ten sam fizyczny `TU` może pełnić rolę Packing `TU` i zachowuje `TU_NUMBER`.
9. Jeżeli Picking `TU` nie spełnia warunków, zawartość jest przepakowywana do jednego lub wielu Packing `TU` dopuszczonych do wydania.
10. Konsolidacja jest wiele-do-wielu:
    - jedno Packing `TU` może zawierać zawartość wielu Picking `TU`;
    - zawartość jednego Picking `TU` może zostać rozdzielona między wiele Packing `TU`;
    - każde przeniesienie jest śledzone na poziomie SKU i ilości.
11. Jedno Packing `TU` może zawierać pozycje z wielu `OutboundOrder`, jeśli źródłowe `CustomerOrder` mają tego samego klienta, adres dostawy, zgodny priorytet i termin dostawy.
12. Wszystkie pozycje w jednym Packing `TU` muszą należeć do jednego `Shipment`.
13. **DEC-G11 (L6):** wersja 1 nie wprowadza reguł zgodności/niekompatybilności towaru przy wspólnym pakowaniu (np. chemia vs. spożywcze, wymogi temperaturowe) — konsolidacja podlega wyłącznie warunkom z pkt 11 (klient/adres/priorytet/termin).
14. **DEC-G12 — Tryby przepakowania «repack all» / «repack by SKU»:** przy przepakowaniu/konsolidacji do Packing `TU` (pkt 9-10) dostępne są dwa tryby: „repack all" — bez kontroli ilościowej per SKU, ryzyko rozbieżności akceptowane; „repack by SKU" — Packer skanuje i podaje ilość dla każdego SKU, System WMS kontroluje zgodność z `pickedQty`. Tryb konfigurowalny per magazyn: administrator może wyłączyć jeden z trybów dla całego magazynu (wymuszając drugi) albo pozostawić wybór Packerowi dla każdej `TU`.
    **Uzasadnienie:** decyzja Darka 2026-08-07 — elastyczność operacyjna (szybkość „repack all" vs kontrola „repack by SKU") z możliwością narzucenia jednolitej polityki przez administratora tam, gdzie ryzyko rozbieżności jest niepożądane.
15. **DEC-G13 — SKU nieoczekiwane w źródłowym `TU` podczas pakowania:** na ścieżce „repack by SKU" (DEC-G12), gdy Packer napotka w źródłowym Picking `TU` towar nieoczekiwany względem zawartości zapisanej w WMS dla tej `TU` — nadwyżkową ilość SKU już oczekiwanego albo zupełnie inny, nieprzypisany kod (jednakowe traktowanie) — System WMS informuje Packera, że pozycja nie należy do tej `TU`; Packer odkłada ją na wyznaczone miejsce `QC` (Quality Control). Zdarzenie nie uruchamia mechanizmu `SHORT_PICKED` — jest rejestrowane jako niezgodność do wyjaśnienia w ramach przyszłych procesów Inventory Management (`WMSAI_OUTBOUND/BACKLOG.md` B9).
     **Uzasadnienie:** decyzja Darka 2026-08-07 — rozliczenie przyczyny (skąd towar w tej `TU`) wymaga dochodzenia, którego mechanizm nie jest dziś projektowany; fizyczne wydzielenie na `QC` zabezpiecza towar do czasu wyjaśnienia, bez wstrzymywania pakowania.

16. **DEC-L49 — `externalIssuable = false` jest blokadą absolutną:**
    **Decyzja:** `P1 R65` znosi wyłącznie dolne progi wydania — masy i objętości. Nie znosi warunku wydawalności typu nośnika. `TU` na typie z `externalIssuable = false` nie może zostać zapieczętowana ani wydana; jedyną drogą jest przepakowanie na typ wydawalny (`P1 R66`). Przy `directPackDeclared = true` i typie niewydawalnym obowiązuje istniejąca Ścieżka B `KROK 7` w `proces_1_standard_fulfillment.md`.
    **Uzasadnienie:** obowiązujące `P1 R65` pozwala obejść warunek wydawalności typu, którego zniesienie nie było przedmiotem tej reguły — wymuszenie ma znosić wyłącznie progi ilościowe. Bez tego rozgraniczenia istnieje ścieżka, którą towar opuszcza magazyn na nośniku niedopuszczonym do wydania.
    **Odrzucone alternatywy:** brak.
    **Data:** 2026-08-25.
    **Reguły:** `P1 R65`, `P1 R66`.

17. **DEC-L50 — `minIssueVolumeUtilization` zastąpione progiem bezwzględnym `minIssueVolume`:**
    **Decyzja:** atrybut `TUSetup.minIssueVolumeUtilization` zmienia nazwę na `TUSetup.minIssueVolume` i zmienia znaczenie: z udziału wykorzystania objętości na próg bezwzględny objętości zawartości, w pełnej analogii do `minIssueWeight`. Typ atrybutu w katalogu zmienia się z „liczba (udział)" na „liczba". `P1 R64` traci przez to zależność od `TUSetup.maxVolume`.
    **Uzasadnienie:** próg wyrażony jako udział wiązał ocenę wydawalności z `maxVolume`, czyli z kubaturą nośnika, która nie jest limitem egzekwowanym przy pakowaniu. Próg bezwzględny mierzy to samo, co `minIssueWeight` — osiągniętą zawartość — i nie wymaga drugiego atrybutu do wyliczenia.
    **Odrzucone alternatywy:** brak.
    **Data:** 2026-08-25.
    **Reguły:** `P1 R64`.

18. **DEC-L56 — ratyfikacja reguł `P1 R63`–`R66` (progi wydania):**
    **Decyzja:** cztery reguły `P1 R63`–`R66`, które weszły do bieżącej prozy `proces_1_standard_fulfillment.md` 2026-08-24 (wersje 1.6–1.8) bez wpisu w rejestrze, zostają ratyfikowane jednym wpisem zbiorczym, obejmującym je jako jedną zmianę semantyki progów wydania. Ratyfikacja dotyczy stanu tych reguł sprzed pakietów naprawczych audytu B16.
    **Uzasadnienie:** cztery reguły domykają jeden mechanizm, którego korpus po migracji nie miał, mimo że kilkanaście miejsc się do niego odwoływało. `R63` nadaje masie i objętości zawartości `TU` jedno policzalne źródło — sumę po `SKU` z kartoteki towarowej magazynu; bez tego „limit `TU`" był pojęciem bez definicji, a `TUSetup.maxWeight` mieszał dwa znaczenia: udźwig typu i faktyczne zapełnienie. `R64` nadaje treść „progom wydania", używanym dotąd kilkanaście razy bez definicji: typ musi być wydawalny na zewnątrz oraz osiągnięty jest co najmniej jeden dolny próg; rozdzielenie wydawalności typu od progu zapełnienia jest celowe — to dwa niezależne warunki, nie jeden. `R65` zapewnia, że próg zapełnienia nie zablokuje zamówienia fizycznie gotowego: operator może wymusić wydawalność poniżej progów, gdy `TU` jest ostatnia dla swojego `OutboundOrder` albo zbliża się `slaDeadline`, z zapisaniem powodu; wymuszenie znosi wyłącznie progi ilościowe. `R66` zamyka ścieżkę, przez którą przepakowanie na typ niewydawalny nie było niczym blokowane; jest wobec `R64` alternatywą, nie dodatkowym warunkiem — przepakowanie na właściwy typ zastępuje ocenę progów.
    **Odrzucone alternatywy:** progi wydania jako jeden warunek łączący wydawalność typu i zapełnienie — odrzucone: wymuszenie z `R65` znosiłoby wtedy także wydawalność typu, czyli otwierało wydanie na nośniku, którym wydawać nie wolno. Objętość zawartości jako drugi warunek blokady zapełnienia (`R15`), na równi z masą — odrzucone: blokada zapełnienia jest mierzalna masą, a o zapełnieniu objętościowym decyduje operator, zamykając `TU`. `TUSetup.maxVolume` jako egzekwowany limit — odrzucone: to kubatura nośnika, wejście doboru przewoźnika (`R30`), nie granica pakowania. Pozostawienie przepakowania bez kontroli typu — odrzucone: realna luka wypuszczająca towar na nośniku niewydawalnym.
    **Nota o dacie:** reguły weszły do prozy `proces_1_standard_fulfillment.md` 2026-08-24 (wersje 1.6–1.8) przed zapisem w rejestrze. Niniejszy wpis jest ratyfikacją z 2026-08-26, nie zapisem decyzji z 2026-08-24.
    **Nota o zakresie:** wpis ratyfikuje mechanizm. Próg objętościowy `R64` zostaje osobno zmieniony na próg bezwzględny `TUSetup.minIssueVolume` decyzją zapisaną w `DEC-L50`; ratyfikacja nie cytuje brzmienia `R64` sprzed tej zmiany jako obowiązującego.
    **Data:** 2026-08-26.
    **Reguły:** `P1 R63`–`P1 R66`.

**Reguły:** `propozycja_procesow_outbound.md` — pkt 11–12: `SP6`, `SP14`, `R20`.

## 7. `Shipment`, Carrier Selection i wydanie

### 7.1 Tworzenie `Shipment`

1. `Shipment` powstaje po przygotowaniu Packing `TU`.
2. `Shipment` grupuje jedno lub wiele gotowych Packing `TU`.
3. Jedno Packing `TU` należy dokładnie do jednego `Shipment`.
4. Powiązania `Shipment` z `CustomerOrder` i `OutboundOrder` wynikają z zawartości Packing `TU`.

**Reguły:** `propozycja_procesow_outbound.md` — `SP7`, `R21`.

### 7.2 Carrier Selection

1. Kolejność działań: Packing `TU` → `Shipment` → Carrier Selection → etykieta.
2. Wybór zewnętrznego przewoźnika wykorzystuje ostateczne parametry przesyłki, adres i termin.
3. Transport własny jest wskazany wcześniej i nie uczestniczy w Carrier Selection.
4. Brak wyniku reguł Carrier Selection wymaga decyzji `Warehouse Supervisor`.
5. Etykieta powstaje dopiero po zatwierdzeniu przewoźnika.
6. **DEC-J10:** GS1 AI `(00)` może zostać użyte na drukowanej etykiecie wyłącznie dla poprawnego `SSCC`, nigdy dla zwykłego `TU_NUMBER`.
7. **DEC-J11 — mechanizm reguł Carrier Selection (L3):** wybór przewoźnika zewnętrznego opiera się na dwóch skonfigurowanych kryteriach, nie na regułach zakodowanych na stałe:
   - **`Region`** — nowy obiekt konfiguracyjny reprezentujący obszar dostawy (dowolnie definiowany per magazyn: kod pocztowy, zakres, grupa województw itd.); niezależny od `Zone` (topologia wewnętrzna magazynu).
   - **`CarrierSetup`** — nowy obiekt konfiguracyjny łączący `Carrier`, `Region` oraz przedział wagi (`minWeight`–`maxWeight`) i przedział objętości (`minVolume`–`maxVolume`). Jeden `Carrier` może mieć wiele `CarrierSetup` (różne regiony i/lub warianty wagowo-objętościowe).
   - `Carrier` staje się formalnym obiektem master data (dziś w dokumencie występuje wyłącznie jako aktor „poza granicą" WMS, §3.1 P1) i dostaje nowy atrybut `priority` — unikalną wartość w słowniku przewoźników, używaną wyłącznie jako ostateczny tie-break (DEC-J12).
   **Reguły:** `propozycja_procesow_outbound.md` — `R61`.
8. **DEC-J12 — algorytm dopasowania i tie-break:** przy Carrier Selection System WMS liczy `maxWeight` i `maxVolume` jako wartości maksymalne spośród **wszystkich** Packing `TU` należących do danego `Shipment` — niezależnie, tzn. mogą pochodzić z różnych `TU` (np. najcięższa `TU` i osobno największa objętościowo `TU`). `CarrierSetup` pasuje, gdy: `Region` dostawy zgodny ORAZ `maxWeight` mieści się w `[minWeight, maxWeight]` tego `CarrierSetup` ORAZ `maxVolume` mieści się w `[minVolume, maxVolume]` tego `CarrierSetup`. Gdy dla tego samego `Region` pasuje więcej niż jeden `CarrierSetup` (zachodzące na siebie przedziały), rozstrzyga kolejno: (a) najwęższy przedział objętości (`maxVolume - minVolume`); (b) przy remisie — najwęższy przedział wagi (`maxWeight - minWeight`); (c) przy dalszym remisie — `Carrier.priority` (rozstrzyga zawsze jednoznacznie, bo wartość unikalna w słowniku `Carrier`).
   **Reguły:** `propozycja_procesow_outbound.md` — `SP17`, `R61`, `R62`.
9. **DEC-J13:** brak dopasowanego `CarrierSetup` dla danego `Shipment` → wybór przewoźnika ręczny przez `Warehouse Supervisor` (zgodne z pkt 4). `Warehouse Supervisor` może również zawsze zmienić wynik Carrier Selection — automatyczny albo ręczny — bez konieczności podania powodu (odróżnienie od trwałej zmiany `allowPartialShipment`, R6, która wymaga powodu).
   **Reguły:** `propozycja_procesow_outbound.md` — `R6`, `SP17`, `R63`.
10. Cały `Shipment` jedzie jednym `Carrier` — bez podziału pozycji jednego `Shipment` między wielu przewoźników (zgodne z D-7.1/SP7, `propozycja_procesow_outbound.md`).
    **Uzasadnienie (DEC-J11-J13):** decyzja Darka 2026-08-02 — start od prostego mechanizmu dwuparametrowego (region + waga/objętość), w pełni konfigurowalnego bez zmiany kodu, z jednoznacznym tie-breakiem (`Carrier.priority` gwarantuje rozstrzygnięcie) i pełną swobodą korekty przez Supervisora. Zamyka L3.
11. **DEC-J14 — L4 nie dotyczy w wersji 1:** etykieta jest dokumentem drukowanym przez System WMS na podstawie już posiadanych danych (dane Packing `TU`, opis wybranego `Carrier`, adres dostawy) — bez wywołania zewnętrznego API przewoźnika i bez kroku potwierdzenia/zgody po jego stronie. W konsekwencji: (a) nie istnieje techniczny tryb awarii generowania etykiety wymagający osobnej ścieżki wyjątku; (b) nie istnieje elektroniczne odrzucenie przesyłki przez przewoźnika. Realny odpowiednik: problem z załadunkiem, ujawniony **przed** `CarrierManifest.CLOSED` — `Warehouse Supervisor` ręcznie zmienia `Carrier` (DEC-J13), bez ponownego druku etykiety. Granica bez zmian: zmiana `Carrier` niemożliwa po `CarrierManifest.CLOSED` (DEC-K10). Zamyka L4.
    **Uzasadnienie:** decyzja Darka 2026-08-02 — wersja 1 nie zakłada integracji z systemem przewoźnika (ani wysyłki danych, ani odbioru potwierdzenia/zgody), więc oba pierwotne scenariusze L4 (błąd techniczny API, elektroniczne odrzucenie) nie mają w tej wersji punktu zaczepienia; jedyny realny wyjątek to decyzja operacyjna WS przy załadunku, już pokryta DEC-J13.

### 7.3 `CarrierManifest`

1. Ten sam obiekt `CarrierManifest` obsługuje przewoźnika zewnętrznego i transport własny.
2. Dla przewoźnika zewnętrznego manifest dotyczy jednego przewoźnika i konkretnego odbioru.
3. Dla transportu własnego manifest grupuje `Shipment` dla konkretnego kursu otrzymanego z procesu zewnętrznego.
4. Planowanie pojazdu i trasy pozostaje poza zakresem WMS.
5. Jeden `Shipment` może należeć tylko do jednego `CarrierManifest`.
6. Manifest zamyka `Dispatcher`.
7. **DEC-K08:** `CarrierManifest.CLOSED` oznacza zablokowanie składu manifestu i gotowość do fizycznego przekazania. Nie oznacza jeszcze przekazania przesyłek.
8. **DEC-K09:** dopiero fizyczne przekazanie powoduje `CarrierManifest.HANDED_OVER` oraz odpowiedni stan wydania powiązanych `Shipment` i `OutboundOrder`.
9. **DEC-K10:** anulowanie jest niedozwolone od chwili osiągnięcia `CarrierManifest.CLOSED`, mimo że fizyczne przekazanie może nastąpić później. Zamkniętego manifestu nie można ponownie otworzyć.
10. **DEC-L60:** `OutboundOrder` osiąga `COMPLETED` dopiero po potwierdzeniu (`CarrierManifest CONFIRMED`) wszystkich `Shipment` zawierających jego Outbound `TU`. Potwierdzenie pierwszego z kilku takich manifestów zostawia zlecenie w `DISPATCHED`; kolejność potwierdzania nie ma znaczenia. Rozliczenie `OutboundOrderLine PACKED → SHIPPED`, `Allocation CONFIRMED → CONSUMED` i `Inventory PICKED → SHIPPED` jest zawężone do linii wydanych danym manifestem. Powód: invariant żył w archiwalnym dokumencie analitycznym jako `SP4` i nie został przeniesiony przy migracji B16, mimo że `DEC-D18` nadal powołuje się na niego jako na obowiązującą przesłankę braku stanu „częściowo wydany" na `OutboundOrder`. Zlecenie może mieć Packing `TU` w kilku `Shipment`, gdy część z nich zostaje zapieczętowana po automatycznym zamknięciu grupowania przez `slaDeadline`.
    **Reguły:** `proces_1_standard_fulfillment.md` — `R70`, KROK 13; `model_stanow_outbound.md` — sekcja 3, wiersz `DISPATCHED→COMPLETED`.

11. **DEC-L62:** Potwierdzenie `CarrierManifest` rozlicza `Inventory` ilościowo — wyłącznie dla ilości znajdującej się w Outbound `TU` tego manifestu — i pomniejsza o nią `Allocation.reservedQty`. Stany terminalne `OutboundOrderLine SHIPPED` i `Allocation CONSUMED` osiągane są dopiero wtedy, gdy każda Outbound `TU` wnosząca ilość danej linii należy do `Shipment` z manifestem `CONFIRMED`. Powód: `P1 R27` dopuszcza wiele Outbound `TU` na jednej `OutboundOrderLine`, a `P1 R28` i `P1 R29` mogą rozdzielić je między różne `Shipment`; dotychczasowy zapis terminalizował całą linię i całą alokację przy pierwszym potwierdzonym manifeście, mimo fizycznie niewydanej reszty. Częściowe rozliczenie zapasu pozostaje dozwolone — blokada dotyczy wyłącznie przejść terminalnych. Nie wprowadzono nowych stanów; `P1 R70` na poziomie `OutboundOrder` pozostaje bez zmian. Reguła opiera się na relacji zawartości Outbound `TU` ↔ `OutboundOrderLine`, tak jak istniejące `P1 R27`, `P1 R58` i `P1 R60`; docelowy zapis tej relacji należy do `BACKLOG.md` B24 i obejmie wszystkie cztery reguły naraz.
    **Reguły:** `proces_1_standard_fulfillment.md` — `R72`, `R71`, KROK 13; `model_stanow_outbound.md` — sekcja 4 wiersz `PACKED→SHIPPED`, sekcja 5 wiersz `CONFIRMED→CONSUMED`.

**Reguły:** `propozycja_procesow_outbound.md` — pkt 5: `SP9`, `R29`; pkt 9: `SP10`.

## 8. Anulowanie i put-back

1. Zwolnienie rezerwacji i fizyczny put-back są oddzielnymi wariantami:
   - przed pobraniem system zwalnia `Allocation` bez pracy fizycznej;
   - po pobraniu powstaje zadanie fizycznego odłożenia towaru.
2. Fizyczny put-back tworzy osobne zadanie dla operatora.
3. Lokalizację docelową proponuje system albo wskazuje operator, gdy zna miejsce składowania. Lokalizacja musi przejść walidację WMS.
4. Anulowanie po zapakowaniu wymaga zgody `Warehouse Supervisor` i może obejmować unieważnienie etykiety, wycofanie Packing `TU` z `Shipment` oraz fizyczny put-back.
5. Faktyczną granicą anulowania jest zamknięcie `CarrierManifest`.
6. Po zamknięciu manifestu anulowanie nie jest możliwe, nawet jeśli fizyczne przekazanie jeszcze nie nastąpiło.
7. **DEC-L01 — `PutBackTask` bez limitu prób (L5, INFO #8):** pętla `LOCATION_VALIDATION↔IN_PROGRESS` (`propozycja_procesow_outbound.md` §4.12) nie ma limitu prób ani automatycznej eskalacji do `Warehouse Supervisor` — świadomie, bo ryzyko odrzucenia dotyczy praktycznie wyłącznie ścieżki ręcznej (operator sam wskazuje lokalizację, pkt 3); rekomendacja Systemu WMS pozostaje dostępna operatorowi przez cały czas trwania zadania jako wyjście awaryjne. Przy odrzuceniu wskazanej przez siebie lokalizacji operator albo szuka poprawnej dalej, albo przyjmuje rekomendację systemu — bez twardego wymuszenia po N próbach. Zamyka INFO #8 (kontrola B2).
   **Uzasadnienie:** decyzja Darka 2026-08-02 — głównym zadaniem systemu jest wskazać miejsce odłożenia z pominięciem ryzyka odrzucenia (rekomendacja systemowa); limit i eskalacja są zbędne, skoro operator ma zawsze dostępne bezpieczne wyjście.
8. **DEC-L02 — Ogólne anulowanie `CustomerOrder`/`CustomerOrderLine` niezwiązane z niedoborem:** żądanie anulowania przyjmowane z dwóch niezależnych kanałów — systemu zewnętrznego (OMS/ERP przez webservice) oraz ręcznie przez `Warehouse Supervisor` w WMS. Anulowanie całego `CustomerOrder` dozwolone wyłącznie, gdy każda jego `CustomerOrderLine` spełnia warunek anulowania linii; jeśli choć jedna nie spełnia, cały nagłówek pozostaje zablokowany. Warunek dla pojedynczej `CustomerOrderLine` zależy od statusu powiązanej `OutboundOrderLine`: brak `OutboundOrderLine` albo `CREATED`/`ALLOCATED`/`SHORT_ALLOCATED` — zwolnienie `Allocation` bez pracy fizycznej; `PICKING` w toku — anulowanie `PickTask` (istniejący mechanizm R59); `PICKED`/`SHORT_PICKED` — komunikat `RF` wstrzymujący pakowanie i `PutBackTask` dla pobranej ilości; `PACKED` — poza tym mechanizmem, wymaga osobnej decyzji `Warehouse Supervisor` (D-8.4/R32: anulowanie spakowanego `TU`, wycofanie Packing `TU` z `Shipment`) — dopiero po tym linia wraca do normalnego toru anulowania. Granica ogólna bez zmian — zamknięcie `CarrierManifest` (R30).
   **Uzasadnienie:** rozdzielenie samoobsługowego kanału anulowania (webservice/Supervisor, bez dodatkowej zgody) od cięższej ścieżki wymagającej decyzji Supervisora, gdy towar już spakowany — zapobiega niekontrolowanemu rozpakowywaniu bez świadomej decyzji.

**Reguły:** `propozycja_procesow_outbound.md` — pkt 5–6: `SP10`, `R30`.

## 9. Decyzje wycofane

Poniższych wcześniejszych założeń nie należy używać:

1. `OutboundOrder` jako bezpośrednie zadanie kompletacji dla operatora.
2. Tworzenie oddzielnego `OutboundOrder` dla każdej strefy magazynu.
3. Wymaganie wielu `OutboundOrder` dla `allowPartialShipment = false` ze względu na podział strefowy.
4. Możliwość anulowania do `HANDED_TO_CARRIER`. Ostateczną granicą jest zamknięcie `CarrierManifest`.
5. Traktowanie `TU_ID` i `SSCC` jako tego samego pojęcia.
6. Traktowanie `CarrierManifest` i `Load` jako dwóch obiektów w wersji 1.
7. **WD-06:** zapisywanie niepoprawnego SSCC albo dowolnego numeru zewnętrznego w polu `SSCC`.
8. **WD-08:** projektowanie `PickWave` jako elementu wersji 1 albo technicznego punktu rozszerzenia w wersji 1.
9. **WD-09:** odstępstwo Supervisora jako tymczasowe, nieodmieniające wartości allowPartialShipment, ograniczone do jednego Shipment. Zastąpione trwałą zmianą flagi (sekcja 4.1 pkt 4-5, 2026-07-18).

## 10. Otwarte kwestie

Poniższych kwestii nie wolno przedstawiać jako zatwierdzonych decyzji. Warianty i rekomendacje muszą być oznaczone jako propozycje:

1. ~~Pełny katalog stanów i przejść w `model_stanow_outbound.md`.~~ → **rozstrzygnięte:** `model_stanow_outbound.md` v1.0 (2026-08-18, `BACKLOG.md` B5).
2. ~~Zdarzenia biznesowe emitowane przy przejściach i aktorzy odpowiedzialni za poszczególne etapy.~~ → **rozstrzygnięte:** jw.

Brak innych otwartych kwestii (stan na 2026-08-18).

## 11. Historia zmian
- **2026-08-28 (zamknięcie B21 i B23):** dodano `DEC-L64` w sekcji 4.1 (bramka liczy pokrycie mieszane) i `DEC-L65` w sekcji 5.1 (dziedziczenie `directPackDeclared`). Zmienione `P1 R6`, `P1 R16`, `P1 R67`, proza `KROKU 2A` i `KROKU 6`, `FR-P1-03`, `FR-P1-41`; nowy `TC-135`. `proces_1_standard_fulfillment.md` → 1.20, `model_stanow_outbound.md` → 1.19. Liczba reguł lokalnych bez zmian (132).
- **2026-08-28 (zamknięcie B22):** dodano `DEC-L63` w sekcji 6.2 — `demandEligibleQty` odejmuje `ATPReservation` i sumę `requiredQty` niezanulowanych `OutboundOrderLine`, zamiast samego `ATPReservation`. Przy `DEC-L20` dopisano notę zawężającą, bez przepisywania jej treści. Zmienione `P2 R6`, `FR-P2-03`, `TC-036`; nowy `TC-134`; `proces_2_outbound_crossdock.md` → 1.13. Liczba reguł lokalnych bez zmian (132).
- **2026-08-28 (rozliczenie wieloetapowe pozycji):** dodano `DEC-L62` w sekcji 7.3 — `OutboundOrderLine SHIPPED` i `Allocation CONSUMED` dopiero po wydaniu wszystkich Outbound `TU` wnoszących ilość linii; `Inventory` rozliczane ilościowo przy każdym manifeście. Nowa `P1 R72`, uzupełniona `P1 R71`, `FR-P1-46`, `TC-132`/`TC-133`; `proces_1_standard_fulfillment.md` → 1.19, `model_stanow_outbound.md` → 1.17, macierz pokrycia 132 reguły lokalne. `P1 R70` bez zmian.
- **2026-08-28 (`BACKLOG.md` B20, część `Allocation`):** dodano `DEC-L61` w sekcji 4.2 — atrybut `reservedQty` i wkład sześciu stanów alokacji do ilości zajętej, wariant A. Nowa `P1 R71`, `FR-P1-45`, `TC-130`/`TC-131`; `proces_1_standard_fulfillment.md` → 1.18, `model_stanow_outbound.md` → 1.16, macierz pokrycia 131 reguł lokalnych. `P2 R6` bez zmian — B22 pozostaje otwarte.
- **2026-08-28 (strata migracyjna B16 — archiwalne `SP4`):** dodano `DEC-L60` w sekcji 7.3 — `OutboundOrder COMPLETED` dopiero po potwierdzeniu wszystkich jego `Shipment`. Nowa `P1 R70`, `FR-P1-44`, `TC-128`/`TC-129`; `proces_1_standard_fulfillment.md` → 1.16, `model_stanow_outbound.md` → 1.15, macierz pokrycia 130 reguł lokalnych.
- **2026-08-28 (pakiet WP-10 planu naprawy audytu B16):** uzupełniono pola **Reguły:** we wszystkich jedenastu wpisach `DEC-L49`–`DEC-L59` faktycznymi referencjami do korpusu po WP-01–WP-09; `DEC-L55` otrzymał jawne potwierdzenie decyzji o niewprowadzaniu zmiany. Treść wpisów jest zgodna z aktualnym korpusem. Nie utworzono żadnego wpisu `DEC-L`.
- **2026-08-26 (pakiet WP-00 planu naprawy audytu B16):** założono jedenaście wpisów `DEC-L49`–`DEC-L59` za decyzje właściciela z 2026-08-25 i 2026-08-26, stanowiące podstawę rejestrową zmian zachowania nanoszonych w kolejnych pakietach planu. Sekcja 4.1: `DEC-L53` (guard `allowPartialShipment = false` kluczowany po `CustomerOrder`) i `DEC-L57` (atrybut `requiredQty` i jego cykl życia). Sekcja 6.1: `DEC-L51` (typ zewnętrzny nośnika Outbound `TU`) i `DEC-L59` (wartość `EXTERNAL` atrybutu `TUSetup.processUsage`). Sekcja 6.2: `DEC-L52` (definicja pełnego dopasowania 1:1), `DEC-L54` (`priority`/`slaDeadline` crossdockowego `OutboundOrder`), `DEC-L55` (`sourceInboundTU` nie wraca na Outbound `TU`) i `DEC-L58` (zawężenie `DEC-L38`). Sekcja 6.3: `DEC-L49` (`externalIssuable = false` jako blokada absolutna), `DEC-L50` (`minIssueVolume` jako próg bezwzględny) i `DEC-L56` (ratyfikacja reguł `P1 R63`–`R66`). Pola **Reguły:** wszystkich jedenastu wpisów pozostają puste do pakietu WP-10, który uzupełni je faktycznymi numerami reguł nadanymi w pakietach WP-01–WP-09. Dopisano dwie noty zawężające bez przepisywania treści, których dotyczą: przy `DEC-L38` (wskazanie `DEC-L58`) i przy sekcji 6.3 pozycja 7 (wskazanie `DEC-L50`). `DEC-L29` bez zmian. Żaden plik procesu ani modelu nie został w tym pakiecie zmieniony.
- **2026-08-23 (domknięcie wymiany z Inbound):** dodano `DEC-L48` — rozdzielenie finalizacji źródłowej Inbound `TU` od nadania jej statusu `IN_PUTAWAY`, po zgłoszeniu rozbieżności przez sesję Inbound. Rewizja `R21`, `R28` i `R42` w `proces_2_outbound_crossdock.md` → 1.9. Bez nowych reguł, wymagań i scenariuszy.
- **2026-08-23 (najpóźniejszy):** dodano `DEC-L47` — domknięcie granicy Inbound→Outbound Crossdock po zmianach zgłoszonych przez sesję Inbound: kwalifikacja Inbound jest przesłanką transportu, wiążące dopasowanie powstaje przy `IN_CROSS_DOCK` (`R30`); opisana ścieżka zerowego dopasowania (`TU` finalizuje się natychmiast do `IN_PUTAWAY` z całą ilością rezydualną) oraz niezmiennik granicy (wyłącznie `TU` `ELEMENTARY`). `proces_2_outbound_crossdock.md` → 1.8, nowe `R41`, `R42`.
- **2026-08-23 (później):** dodano `DEC-L46` — domknięcie pozycji audytu V3-CD-02 (PutBack vs współdzielona docelowa Outbound `TU`): guard anulowania pustej `TU` uzupełniony o warunek braku aktywnych i zaplanowanych `CrossDockPickTask` wskazujących ją jako cel (usuwa dangling target reference), z jawnie zamierzoną asymetrią wobec `DEC-L39` — anulowanie nie czeka na `slaDeadline`; dodany niezmiennik ciągłości celu aktywnego zadania. Oznaczono `[HISTORYCZNE]` na `DEC-L31` ze wskazaniem na `DEC-L46`. Rezultaty naniesione w `proces_2_outbound_crossdock.md` 1.6 (`R34` rewizja, `R40` nowa), `model_stanow_outbound.md` 1.5, `wymagania_outbound.md` (`FR-P2-19` rewizja, nowe `FR-P2-22`), `scenariusze_testowe_outbound.md` (`TC-035` rozszerzone, nowe `TC-101`), `macierz_pokrycia_outbound.md` (nowy wiersz `P2 R40`, licznik P2 37 → 38, razem 110 → 111).
- **2026-08-23:** wycofano koncept puli operatorów w całości (`DEC-L41`) — `WMSAI_ADM` zarchiwizowany, `DEC-L08` oznaczone historyczne. Nowe `DEC-L42`–`DEC-L45`: zasady podejmowania zadań magazynowych przez wybór modułu pracy w RF, dla `PickTask`/`CrossDockPickTask`/`PutBackTask`. Realizacja `BACKLOG.md` B10 (zamknięte jako odrzucenie pierwotnego założenia, nie jego wykonanie).
- **2026-08-22 (później):** dodano `DEC-L38` (jednorazowe, zamknięte generowanie `CrossDockPickTask`/`OutboundOrderLine`), `DEC-L39` (automatyczne zamknięcie docelowej Outbound `TU` po `slaDeadline`) i `DEC-L40` (ponowna ewaluacja bramki GR per komunikat, `GR_REJECTED` nie powoduje `POSTING_ERROR`, widoczność `grAcceptanceStatus` dla Supervisora) — trzy decyzje właściciela zamykające audyt V3-CD. Oznaczono jako historyczne odpowiednie fragmenty `DEC-L19` (zawężone), `DEC-L23` (zawężone) i `DEC-L36` (zastąpione). Rezultaty naniesione w `proces_2_outbound_crossdock.md` 1.3, `model_stanow_outbound.md` 1.4, `wymagania_outbound.md`, `scenariusze_testowe_outbound.md`, `macierz_pokrycia_outbound.md` (partie 1–4 audytu V3-CD).
- **2026-08-22:** dodano `DEC-L36` (dwa wyzwalacze `PACKING_SEALED` docelowej Outbound `TU` w cross-dockingu) i `DEC-L37` (pierwszeństwo finalizacji źródłowej Inbound `TU` przed nowym zapotrzebowaniem) — decyzje właściciela podjęte przy naprawie luk propagacji po migracji B5/B16. Dodano `DEC-A14` (archiwizacja dokumentu analitycznego) i `DEC-A15` (brak osobnego pliku dla PROCESU 5) — utrwalenie ustaleń z 2026-08-22 w kanonicznym rejestrze decyzji zgodnie z `DEC-A10`. Rezultaty naniesione w `proces_1_standard_fulfillment.md` 1.2, `proces_2_outbound_crossdock.md` 1.2, `model_stanow_outbound.md` 1.3, `wymagania_outbound.md`, `scenariusze_testowe_outbound.md` i `macierz_pokrycia_outbound.md`.
- **2026-08-18 (jeszcze później):** dodano `DEC-L35` — alternatywna ścieżka kompletacji: Picker deklaruje zbieranie bezpośrednio do Outbound `TU` (`TU.directPackDeclared`), System WMS automatycznie zamyka rolę `PackUnit` i przenosi `OutboundOrderLine PICKED→PACKED`, gdy warunki wydania spełnione; w przeciwnym razie standardowy tor Packera jako zapasowy punkt weryfikacji. Zaktualizowano `propozycja_procesow_outbound.md` → 1.29 (`R86`, §3.1 krok 6a/7, §4.7, §4.4) i `model_stanow_outbound.md` → 1.2.
- **2026-08-18 (później):** zamknięto `BACKLOG.md` B15. Dodano `DEC-L33` (wyzwalacz `CustomerOrder SHIPPED→CLOSED` — wyliczane automatycznie, gdy wszystkie kontrybuujące `OutboundOrder` `COMPLETED`) i `DEC-L34` (usunięcie nieużywanego `OutboundOrder.ON_HOLD`, brak pokrycia w żadnym kroku procesu ani wcześniejszej decyzji). Zaktualizowano `propozycja_procesow_outbound.md` → 1.28 (`R85`, §3.1 krok 13a, §4.1, §4.3, §9 L-INV) i `model_stanow_outbound.md` → 1.1.
- **2026-08-18:** zamknięto §10 pkt 1–2 — utworzono `model_stanow_outbound.md` v1.0 (`BACKLOG.md` B5): katalog stanów, przejść, zdarzeń domenowych (`PascalCase` EN) i aktorów dla 17 obiektów §4 `propozycja_procesow_outbound.md` v1.27, format kolumn i sekcji ujednolicony ze standardem Inbound `model_stanow.md`. Przy tej pracy ujawniono i zgłoszono do `BACKLOG.md` (B15) dwie luki dokumentacyjne bez opisanego kroku procesu w §3: `CustomerOrder SHIPPED→CLOSED` i `OutboundOrder PICKING_IN_PROGRESS↔ON_HOLD`/wznowienie.
- **2026-08-13:** zamknięto `BACKLOG.md` B4. Dodano `DEC-G14`–`DEC-G19`: walidację `SSCC`, format i unikalność `TU_NUMBER`, brak późniejszej reklasyfikacji, obsługę konfliktu w trybie serwisowym oraz model `Sequence`/`TUSetup`. Zaktualizowano `propozycja_procesow_outbound.md` do wersji 1.21 (`R26`, `R27`, `R82`, §4.7, §4.8, §4.17, `K10`, `K11`).
- **2026-07-18:** przeniesiono kluczowe ustalenia z promptu kontrolnego do źródeł projektu; zatwierdzono rozdzielenie planowania od częściowej alokacji, semantykę `CarrierManifest.CLOSED`/`HANDED_OVER`, hybrydową obsługę `SHORT_PICKED`, konfigurację limitu realokacji magazyn/klient, zasady SSCC oraz zarządzanie wiedzą bez promptów jako źródeł.
- **2026-07-18:** zastąpiono jednorazowe, tymczasowe odstępstwo Supervisora (allowPartialShipment) trwałą zmianą flagi z podaniem powodu — patrz sekcja 4.1 pkt 4-5 i WD-09.
- **2026-07-18:** rozstrzygnięto L2 — kolejność wykonywania PickTask jako parametr magazynu (slaDeadline→priority domyślnie, albo priority→slaDeadline), remis rozstrzyga kolejność zgłoszenia PickTask. Patrz sekcja 4.3 pkt 7.
- **2026-07-18:** rozstrzygnięto B3a — mechanizm rezerwacji ATP na poziomie CustomerOrderLine (ATPReservation), dodano sekcję 4.3 pkt 8-14, zaktualizowano pkt 3.
- **2026-07-19:** domknięcie MEDIUM #3 (kontrola B2, kontynuacja): poprawiono DEC-D14 i DEC-D15 (jawny zakres `allowPartialShipment = false`, rozróżnienie statusu `CustomerOrderLine` `BACKORDERED`/`OPEN` po anulowaniu `SHORT_ALLOCATED`), DEC-F15 (to samo rozróżnienie dla `SHORT_PICKED`, korekta zwrotu `ATPReservation` o potwierdzoną ilość brakującą, wyjątek `PACKED` dla powrotu nagłówka do `ACCEPTED`) i §4.3 pkt 13 (źródło R47 — analogiczna korekta).
- **2026-08-02:** rozstrzygnięto L1/L7 — agregacja statusu pozycja→nagłówek. L1 (`OutboundOrder`): już pokryte SP2–SP4/R58 w `propozycja_procesow_outbound.md`, bez nowej decyzji. L7 (`CustomerOrder`): nagłówek = najmniej zaawansowana aktywna `CustomerOrderLine`, wyjątek `PARTIALLY_SHIPPED` przy choć jednej `SHIPPED`, wyjątek `BACKORDERED` wyłącznie gdy wszystkie aktywne pozycje `BACKORDERED`. Dodano DEC-D18 (nowa sekcja 4.4); rozszerzono DEC-D15 na oba warianty `allowPartialShipment` i dodano wyzwalacz (d) do DEC-D16 (§4.2 pkt 12/13/14); usunięto z §10 Otwarte kwestie.
- **2026-08-02 (później, L3):** rozstrzygnięto L3 — mechanizm Carrier Selection. Nowe obiekty konfiguracyjne `Region` i `CarrierSetup` (region dostawy + przedział wagi/objętości per `Carrier`); dopasowanie do `maxWeight`/`maxVolume` liczonych z Packing `TU` `Shipment`; tie-break: najwęższa objętość → najwęższa waga → `Carrier.priority` (nowy atrybut, unikalny w słowniku `Carrier`); brak dopasowania → ręczny wybór `Warehouse Supervisor`, zawsze może zmienić wynik bez podania powodu. Dodano DEC-J11–DEC-J13 (§7.2); usunięto L3 z §10 Otwarte kwestie.
- **2026-08-02 (jeszcze później, L4):** rozstrzygnięto L4 — nie dotyczy w wersji 1. Etykieta to wydruk System WMS bez zewnętrznego API i bez kroku zgody przewoźnika — brak trybu awarii, brak elektronicznego odrzucenia. Realny mechanizm: problem z załadunkiem przed `CarrierManifest.CLOSED` → ręczna zmiana `Carrier` przez `Warehouse Supervisor` (DEC-J13), bez ponownego druku etykiety; granica `CLOSED` bez zmian (DEC-K10). Dodano DEC-J14 (§7.2).
- **2026-08-02 (kolejno, L5):** rozstrzygnięto L5, część put-back — DEC-L01 (§8): `PutBackTask` bez limitu prób w pętli `LOCATION_VALIDATION↔IN_PROGRESS`, rekomendacja systemu zawsze dostępna jako wyjście; zamyka INFO #8 (kontrola B2). Część „kontrola zablokowanej lokalizacji" (`StockCheckTask`, DEC-F10) odłożona do przyszłej grupy procesów Inventory Management, zaparkowana w `../MEMORY.md`. Usunięto z §10 Otwarte kwestie punkty 5 (etykieta, rozstrzygnięte L4 — przeoczone przy poprzedniej edycji) i 6 (put-back).
- **2026-08-03 (L6):** rozstrzygnięto L6 — zgodność SKU/opakowań przy wspólnym pakowaniu. Wersja 1 nie wprowadza żadnych reguł zgodności/niekompatybilności towaru; konsolidacja podlega wyłącznie warunkom klient/adres/priorytet/termin już opisanym w §6.3 pkt 11. Dodano DEC-G11 (§6.3 pkt 13); usunięto punkt 4 z §10 Otwarte kwestie.
- **2026-08-07:** rozstrzygnięto ogólne wyzwalacze anulowania (BACKLOG B3), DEC-L02; zaktualizowano `propozycja_procesow_outbound.md` (wersja 1.10), `BACKLOG.md` i `handover_outbound_wms.md`.
- **2026-08-07:** rozstrzygnięto ostatni punkt B3 — brak/uszkodzenie/nadwyżka wykryte podczas pakowania; DEC-F18, DEC-F19, DEC-G12, DEC-G13; `propozycja_procesow_outbound.md` wersja 1.12; `BACKLOG.md`, `handover_outbound_wms.md`.
- **2026-08-08:** uzupełniono ślady decyzja→reguła (linie **Reguły:**) dla decyzji cytowanych dotąd w §5/§7 `propozycja_procesow_outbound.md` — warunek wstępny BACKLOG B8.
- **2026-08-11:** usunięto z §10 Otwarte kwestie dwa nieaktualne punkty — priority/slaDeadline (rozstrzygnięte 2026-07-18, §4.3 pkt 7) i pełny przebieg cross-dockingu paczkowego (rozstrzygnięte 2026-08-10, Outbound Crossdock §3.2/§6.2); przenumerowano listę.
- **2026-08-11:** poprawiono DEC-L04 (cytowanie R1/R4→SP14/R20, korekta redakcyjna). Dodano DEC-L09–DEC-L12: kolejność READY_FOR_DISPATCH przed Carrier Selection w Shipment, konsolidacja wymaga identycznego slaDeadline z zamknięciem po timeout, granica anulowania Shipment = POSTING_PENDING (dalej Return Receipt, BACKLOG B13), OutboundOrder.PACKED jako pełny agregat. propozycja_procesow_outbound.md → 1.15.
- **2026-08-11 (Grupa B):** dodano DEC-L13–DEC-L17 (§6.2b Outbound Crossdock): `fulfillmentChannel`/SP19/SP20 i diagram nagłówka `OutboundOrder` dla crossdocku; R76 (`crossDockEligibleQty`); gałąź `allowPartialShipment` dla braku/DAMAGED przy zakończeniu `CrossDockPickTask`; rozszerzenie R14 (wielokrotna Outbound `TU` per zadanie); status `PICKING` dla `OutboundOrderLine` cross-dockowej i blokada ogólnego anulowania do `PACKED`. `propozycja_procesow_outbound.md` → 1.16.
- **2026-08-11 (follow-up Crossdock):** dodano `DEC-L18`–`DEC-L21` (kryterium `CROSS_DOCKED`/`IN_PUTAWAY`, zamek ilościowy `CrossDockPickTask`, rewizja `crossDockEligibleQty`, mechanizm odzysku `confirmedQty`); uzupełniono `DEC-L07` o odesłanie do `DEC-L21`. `propozycja_procesow_outbound.md` → 1.17.
- **2026-08-12:** dodano DEC-L22 — mechanizm korekty `Shipment POSTING_ERROR` przy niezgodności zawartości (dwie ścieżki: przyczyna ERP-side vs WMS-side); zamyka `WMSAI_OUTBOUND/BACKLOG.md` B3c. `propozycja_procesow_outbound.md` → 1.19, `R80`.
- **2026-08-12:** dodano `DEC-L23` — ICR-05: bramka ERP `Shipment` cross-dockowego czeka na `GR_ACCEPTED` każdej źródłowej Inbound `TU` wnoszącej ilość do `Shipment`, zamiast na `POSTED` całego ASN; dodano `grAcceptanceStatus` i `R81`. `propozycja_procesow_outbound.md` → 1.20.
- 2026-08-13: dodano `DEC-L24`–`DEC-L26` — rozliczenie GR dla Outbound Crossdock rozbite na rozliczenie crossdockowe i putawayowe tej samej Inbound `TU`; `Shipment` czeka wyłącznie na rozliczenie crossdockowe. `propozycja_procesow_outbound.md` → 1.22 (`R54`, `R81` doprecyzowane; §3.2 krok 3/4, §4.13, §6.8, §9 L-CD zaktualizowane).
- 2026-08-14: uzupełniono DEC-L25 (Reguły) po odpowiedzi Inbound D48 na ICR-06–ICR-08 — dyskryminator rozliczenia crossdockowego to GR_SETTLEMENT_SOURCE, nie numer wersji. propozycja_procesow_outbound.md → 1.23 (R81 doprecyzowane).
- 2026-08-15: dodano `DEC-L27` — `damagedQty` w kontrakcie crossdockowym z Inbound; `propozycja_procesow_outbound.md` → 1.24 (`R83`, §4.13, §3.2 krok 2/3, §6.8).
- 2026-08-15: dodano `DEC-L28` — korekta `R76`: `sourceEligibleQty` odejmuje także `damagedQty` zakończonych `CrossDockPickTask`; `propozycja_procesow_outbound.md` → 1.25.
- 2026-08-15: dodano `DEC-L29`–`DEC-L30` — audyt spójności Outbound Crossdock (8 luk propagacji): doprecyzowano moment powstania Outbound `TU` w cross-dockingu (przy pierwszym odłożeniu `SKU`, nie przy `PACKING_SEALED`, `DEC-L29`); ujednolicono terminologię bazy ilościowej `R76`/`R77`/§3.2 z „fizycznie zidentyfikowana" na „zadeklarowana (ASN)", zgodnie z D42 (`DEC-L30`); oznaczono `[HISTORYCZNE]` na `DEC-L03`, `DEC-L06` i fragmencie `DEC-L15`. `propozycja_procesow_outbound.md` → 1.26.
- 2026-08-17: kontynuacja audytu spójności Outbound Crossdock — dodano `DEC-L31` (Outbound `TU` pusta po pełnym `PutBackTask` → `CANCELLED`, nowa `R84`) i `DEC-L32` (`grAcceptanceStatus` aktualizowany systemowo po `sourceInboundTU`, nie per `Shipment`, rewizja `R81`); poprawiono `R27` (referencja do źródłowej, nie „zakończonej", Inbound `TU`); poprawiono diagram stanów/tabelę przejść §4.7 (nowa krawędź cross-dockowa `CREATED`→`PACKING_SEALED`, doprecyzowanie `CREATED`→`CANCELLED`); poprawiono diagram sekwencji §6.8 (moment przydziału `TU_NUMBER`/`SSCC` przeniesiony z `PACKING_SEALED` na pierwsze odłożenie `SKU`, zgodnie z §3.2 krok 1); doprecyzowano zdanie w przeglądzie procesów (moment powstania Outbound `TU`); oznaczono `[HISTORYCZNE]` na `DEC-L23` (fragment o identyfikatorze zadania w sygnale GR) i fragmencie `DEC-L15` (Uzasadnienie). `propozycja_procesow_outbound.md` → 1.27.
