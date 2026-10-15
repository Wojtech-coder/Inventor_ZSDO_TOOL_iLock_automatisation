# ZSDO_TOOL

Offset geodezyjny: strefa oblaminowania króćców.

Narzędzie dla Autodesk Inventor 2026. Wyznacza na powierzchni zbiornika GRP strefę oblaminowania króćca, czyli pas o szerokości **X mierzonej po powierzchni zbiornika** od krawędzi wpalenia. Wynikiem jest **bryła** (powierzchnia pogrubiona o 1 mm na zewnątrz zbiornika), z której w Visual Components generowane są ścieżki szlifowania. Visual Components nie widzi samych powierzchni.

## Status

- [x] Etap 1: reguła iLogic (prototyp): [iLogic/GeodesicOffset.iLogicVb](iLogic/GeodesicOffset.iLogicVb)
- [x] Tryb automatyczny: wszystkie króćce, X z tabeli króćców rysunku DWG
- [ ] Etap 2: walidacja na modelach (płaszcz, dennica, przejście płaszcz–dennica, wzmocnienia). W toku: projekt 250461.
- [ ] Etap 3: eksport do Visual Components (STEP/IGES)
- [ ] Etap 4: Add-In (przycisk w ribbonie, okno parametrów)

## Użycie (iLogic)

Otwórz **złożenie** zbiornika i uruchom regułę zewnętrzną `GeodesicOffset.iLogicVb`. Reguła pyta o tryb.

### TAK: automatycznie, wszystkie króćce

1. Wskaż rysunek DWG z tabelą króćców. Okno otwiera się w folderze `3_Drawings` projektu.
2. Tabela jest czytana w tle przez AutoCAD bez okna (`accoreconsole.exe`, trwa kilka sekund). Dla każdej pozycji N1, N2… liczona jest szerokość **X = L1 + 6·S6** z kolumn „Overlaminate”.
3. Pozycja jest łączona z króćcem w złożeniu: najpierw po numerze N w nazwie wystąpienia (np. `250461-N2-LF TYPE D-DN 80…`). Jeśli numeru nie ma, łączenie idzie po iProperty Description (= kolumna Name) i DN. Pozycja pasująca do kilku wystąpień jest zgłaszana i pomijana.
4. Krawędź wpalenia jest szukana przy osi króćca (lokalna oś X wystąpienia).
5. Wszystkie strefy trafiają do podzłożenia `Oblaminowanie_strefy`, każda jako osobna część `N2_X132.4`.
6. Raport końcowy pokazuje status każdej pozycji, króćce z modelu bez pozycji w tabeli oraz ostrzeżenie, gdy rewizja rysunku różni się od rewizji modelu.

### NIE: ręcznie, jeden króciec

1. Podaj X w mm.
2. Wskaż **zewnętrzną** krawędź wpalenia (na skórze zbiornika, nie na wewnętrznej powierzchni).
3. Do złożenia zostanie dodana część `Oblaminowanie_X...`.

### Zawartość części wynikowej

- szkic 3D `Krawedz_wpalenia` i szkic 3D `Granica_oblaminowania` (offset o X),
- powierzchnia `…_powierzchnia` (loft przez pierścienie leżące na zbiorniku),
- bryła: pogrubienie powierzchni na zewnątrz zbiornika, więc dolna ściana bryły leży na zbiorniku.

Części i podzłożenie nie są zapisywane automatycznie. Inventor zapyta o nie przy zapisie złożenia. Poprzednie wyniki (wystąpienia `Oblaminowanie_*`) są ukrywane na czas liczenia. Przed ponownym uruchomieniem warto je usunąć.

## Jak to działa

1. **Pętla wpalenia:** do wskazanej krawędzi przylegają dwie ściany, a skórą zbiornika jest większa z nich razem ze ścianami połączonymi z nią stycznie. Pętla jest domykana po krawędziach między skórą a ścianą spoza skóry, więc działa także dla wpalenia leżącego na kilku ścianach.
2. **Promienie geodezyjne:** z punktów co 10 mm na krawędzi wychodzi promień prostopadły do krawędzi i styczny do powierzchni. Idzie krokami 10 mm po **zewnętrznej powłoce całego złożenia**:
   - start zawsze tuż nad ostatnim punktem powłoki, czyli na pewno w powietrzu,
   - promień `FindUsingRay` do przodu wykrywa wznoszącą się powierzchnię albo ścianę,
   - promień w dół z końca kroku daje kolejny punkt na powłoce.

   Dzięki temu wzmocnienia (ta sama bryła, osobna bryła albo osobna część) są traktowane jednakowo i offset wchodzi na ich spadki. Nie przeszkadza też nic, co wisi nad powierzchnią, np. pionowa szyjka króćca nad nachyloną dennicą.
3. **Przeszkody:** inne wpalenie, koniec zbiornika albo uskok stromszy niż 70°. Promień kończy się min. 20 mm przed przeszkodą, także gdy przeszkoda leży z boku albo tuż za granicą X. Przy ukośnej ściance wpalenia odsunięcie liczone jest od miejsca, w którym zaczyna ubywać materiału.
4. **Bryła:** pierścienie na odległościach 0, X/4, X/2, 3X/4 i X leżą na zbiorniku. Loft przez nie tworzy powierzchnię, a pogrubienie o 1 mm tworzy bryłę. Jeśli loft się nie uda, reguła ponawia go bez zbyt bliskich punktów, a potem przez 3 pierścienie. Gdy wszystkie próby zawiodą, część z pierścieniami zostaje jako `…_BLAD` do obejrzenia.

Parametry (krok, odsunięcie, kąt przeszkody, grubość bryły) są w procedurze `Configure` reguły.

## Narzędzia pomocnicze

- [iLogic/RaportModelu.iLogicVb](iLogic/RaportModelu.iLogicVb): zrzut drzewa komponentów, iProperties, położeń i osi oraz cech i szkiców złożenia do pliku tekstowego w folderze `dane/`. Folder jest wyłączony z repozytorium, bo zawiera dane klienta.

## Znane ograniczenia (prototyp)

- Loft bywa zawodny dla części króćców (projekt 250461: N4, N5). Przyczyna jest w trakcie diagnozy.
- Nakładające się strefy sąsiednich króćców nie są przycinane.
- Wpalenie wklęsłe o promieniu krzywizny mniejszym niż X da samoprzecięcia granicy. Nie jest to jeszcze wykrywane.
- Elementy przylegające do ścianki od wewnątrz zbiornika są liczone jako grubszy materiał.
