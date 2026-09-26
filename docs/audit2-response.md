# Rozstrzygnięcie audytu 2

Uzupełnienie po kolejnej recenzji: poniższa tabela dokumentuje rozstrzygnięcia audytu 2. Rekomendacja B27 dotycząca symlinków/generacji została następnie wycofana: aktualny PRD §17.2 i specyfikacja §11.1 przewidują zwykły katalog wynikowy, ograniczony backup i odzyskiwanie po przerwaniu, bez obietnicy atomowej zamiany katalogów. Kolejna korekta doprecyzowuje również błędy D2/zajętej blokady, wygląd sequence, fingerprint theme, dystrybucję przypiętego D2 i współdzielenie walidatorów. W tych sprawach obowiązują aktualne dokumenty, nie historyczne propozycje z tabeli.

Ocena uwag z [audyt2.md](../audyt2.md) względem dokumentów na commit `4469c74`. Poprawki dotyczą [PRD](prd.md), [specyfikacji technicznej](technical-spec.md), [planu implementacji](implementation-plan.md) i [README](../README.md). Oryginalny audyt pozostaje bez zmian.

Audyt wykrywa rzeczywiste luki kontraktów i zależności. Nie zgadzam się jednak ze wszystkimi ocenami krytyczności ani z automatycznym przyjmowaniem proponowanych rozszerzeń. Wspólny identyfikator wersji nie uniemożliwia walidacji, gdy typ dokumentu wynika z argumentu CLI; podobny graf flow i infrastruktury nie jest sam w sobie błędem. Problemem są brakujące reguły, nie konieczność nadania diagramom odmiennej topologii. Propozycje poniżej ograniczają nowe mechanizmy do zakresu v0.1.

## A. Spójność kontraktów

| Punkt | Ocena i rozstrzygnięcie |
|---|---|
| A1 | Trafne usprawnienie, zawyżona waga K. Kontekst CLI rozróżniał dokumenty, ale osobne `flowframe-model/v1` i `flowframe-view/v1` są czytelniejsze i ułatwiają samodzielną identyfikację. Zmieniono przykłady, manifest i listę schematów przed pierwszą implementacją. |
| A2 | Trafne. Jawne ID są wymagane dla wszystkich relacji i kroków. Nie wprowadzamy ukrytego wyprowadzania identyfikatorów. |
| A3 | Trafne. Końce relacji wskazują wyłącznie elementy; poprawiono przykład diagnostyki i doprecyzowano regułę. |
| A4 | Trafne. Grupy usunięto z Sequence IR v0.1; pozostają po MVP. |
| A5 | Trafne, ale prefiks nie wystarcza do wyznaczenia kodu. Ta sama rodzina FFT może opisywać literówkę i niebezpieczny SVG. Dodano kategorię błędu, tabelę mapowania, pierwszeństwo zaobserwowanych błędów i SIGINT 130. Walidacja nie kontynuuje niebezpiecznych operacji tylko po to, by zebrać więcej błędów. |
| A6 | Trafne. Próg AI i korpus są zamrażane w P7.4 po pilotażu, przed ewaluacją wydania. P0 nadal ustala budżety renderowania. |
| A7 | Trafne. `uv` i Hatchling są wybrane; P0 zapisuje uzasadnienie w ADR, nie otwiera decyzji ponownie. |
| A8 | Trafna niejasność własności. Framework kontroluje wzór, grot, grubość i etykiety semantyczne; theme kontroluje kolory. Dodano reguły dla interaction/directionality i legendę. Sam zakaz custom patternów nie zastępuje testów czytelności. |
| A9 | Trafne. Zakaz surowych stylów dotyczy Model/View. Jawnie zlecona globalna zmiana brandingu pozostaje dozwolona. |
| A10 | Trafne. Źródła są względne wobec project root, wyniki wobec katalogu manifestu; zasoby pakietu mają logiczne identyfikatory. |
| A11 | Trafne. Hash obejmuje surowe bajty theme, inwentarza licencji i zasobów, z jednoznacznym kodowaniem ścieżek/długości. Kanonikalizacja dotyczy osobnego hasha przetworzonego SVG. |
| A12 | Trafne. PRD odsyła do jednego układu pakietu; ujednolicono testy i projekt `examples/payments`. Usunięto projektowy config z proponowanego korzenia implementacji. |
| A13 | Częściowo trafne. Różna liczba warstw nie oznacza innego zachowania, lecz opcjonalne sprawdzanie bezpieczeństwa SVG było mylące. Ujednolicono granice etapów i obowiązkową weryfikację wyjścia. |
| A14 | Trafne. JSON Pointer + plik/wiersz/kolumna; brak lokalizacji dopuszczalny dla błędów bez pola źródłowego. |
| A15 | Trafne. Końcowy kompozytor musi dodać metadane dostępności, niezależnie od możliwości D2. |
| A16 | Trafne. Self-message jest wymagane i sprawdzane w P0; nie może zniknąć z zakresu pod hasłem „if supported”. |
| A17 | Trafne. Jeden algorytm selekcji architektury/flow, a sequence ma jawnie odrębny kontrakt scenariusza bez select/exclude. |
| A18 | Trafne. Przy zamkniętych schematach nowe pola i wartości enum zmieniają akceptowany język. Wymagają nowej wersji kontraktu; nie dokładamy równoległego `minFlowframeVersion`. |
| A19 | Trafne. Skrócono tytuł PRD i zastąpiono powielony plan tabelą Stage → P. |
| A20 | Trafne. Deterministyczny review jest jawnie w MVP, source-conformance korzysta z opcjonalnego adaptera P7, a visual jest po MVP. Usunięto obietnicę visual z kontraktu v0.1. |

## B. Luki i proponowane rozszerzenia

| Punkt | Ocena i rozstrzygnięcie |
|---|---|
| B1 | Luka jest trafna, ale nie blokowała prototypu z domyślnym ELK. Dodano `render.layoutEngine` i `build --layout`; v0.1 dopuszcza tylko ELK. Nie dodano bezużytecznego fallbacku przy jednym wspieranym silniku. |
| B2 | Trafny brak kontraktu, zbyt mocne stwierdzenie o identyczności rodzin. Dodano filtry relacji, payload i domyślne ustawienia flow. Nie zgadujemy ról biznesowych z samego stopnia węzła: store wynika z kind, a source/sink może być tylko opisową adnotacją projekcji. |
| B3 | Trafne. Sequence nie przyjmuje select/exclude ani boundaries; porządek uczestników wynika z pierwszego wystąpienia lub jawnej dokładnej permutacji. Dodano protocol/relationId, reguły zgodności i referencje wyłącznie do elementów. |
| B4 | Trafne. Dodano pełny katalog display, domyślne layouty, wymagane semantic/interaction oraz pełny domyślny config z rekurencyjnymi wartościami domyślnymi. |
| B5 | Trafne. Dodano normatywną macierz; event+synchronous ostrzega, ale nie jest odrzucane jako niemożliwe. Nie narzucamy asynchroniczności każdemu zdarzeniu. |
| B6 | Trafne, lecz `technologyId`/`icon` nie są konieczne dla generycznego MVP. Indeks mapuje kind na lokalny zasób/licencję/hash. Osadzanie ikon w D2 ma bramkę P0; P4.9 dostarcza implementację. Vendor mapping pozostaje po MVP. |
| B7 | Trafne. Pliki D2 pakietu są wewnętrznymi szablonami. Klasy powstają z rozwiązanego theme, a wynik nie ma importów. |
| B8 | Trafne. Compile nie uruchamia D2. Render daje wyłącznie body bez manifestu/dekoracji i sprawdza pełny lint oraz nagłówek; nagłówek nie jest zabezpieczeniem. Dodano opcjonalny jawny config i fingerprint theme, aby custom fonty nie zależały od ukrytego katalogu roboczego. Pełny build obejmuje kompozytor. |
| B9 | Trafna luka workflow, choć jedno polecenie mogło oznaczać skrypt przykładów. Wybrano użyteczniejszy jawny `views` registry + `build --all`, jeden model i katalog per View ID. Bez globów i bez obietnicy atomowości całego projektu. |
| B10 | Trafne. Uzupełniono typografię, typy/jednostki tokenów, padding, komplet mapowań, boundary mappings, zgodność ID/katalogu, licencje i zakaz przesłaniania wbudowanych ID. TTF potwierdza pomoc lokalnego D2; finalne zachowanie przypiętego wydania musi sprawdzić P0. |
| B11 | Trafne. Dodano description, status i purpose. Usunięto niezdefiniowane priority/emphasis; założenia AI trafiają do osobnego raportu dla człowieka, nie do nieznanych pól YAML. |
| B12 | Trafne. parentId może wskazywać system; jego brak oznacza obiekt poza jawną granicą systemu. Dodano systemBoundary i walidację zewnętrznych uczestników; poprawiono przykład modelu. |
| B13 | Trafne. Rodzicem jest boundary/system, nigdy element. Zagnieżdżanie rodzajów boundary nie udaje walidacji konkretnej chmury; obowiązuje brak cykli. |
| B14 | Częściowo trafne. Usunięto niemodelowalny „external network element” i poprawiono przykład Internetu. Nie rozszerzono automatycznie słownika o klienta, region i account; w MVP wystarczą zewnętrzny system, network, tagi i opis. Platform/operations mogą mieć nakładające się audytoria, bo są metadanymi bez wpływu na selekcję. |
| B15 | Trafne wymaganie gramatyki i przestrzeni nazw. Dodano regex, limit 64 i zasady tagów. Odrzucono zakaz słów D2 w semantycznym modelu: kolizje rozwiązuje prefiksowanie/escapowanie generatora. |
| B16 | Trafne. Wszystkie wartości są danymi tekstowymi, więc `a < b` jest legalne. Dodano escaping, NFC/code points, zakaz znaków kontrolnych i deterministyczne zawijanie zamiast heurystycznego wykrywania markup. |
| B17 | Trafne, ale sama suma slotów nie wystarcza dla tytułu wyśrodkowanego na canvasie. Przy bocznym logo trzeba zapewnić symetryczny odstęp wokół środka. Dodano odpowiedni wzór, wyrównanie pionowe, padding, legendę i jednoznaczną obsługę bounds D2. |
| B18 | Trafne. `generatedAt` jest domyślnie pomijane; SOURCE_DATE_EPOCH daje stabilny UTC. Niepotrzebne jest niestandardowe słowo schema „volatile”. Dodano provenance fontów/ikon, logiczne flagi i warunkowe assetProcessing. |
| B19 | Trafne. Wyszukiwanie FLOWFRAME_D2 → PATH jest jawne, a pin/version/checksum obowiązują zawsze. Nie dokładamy trybu reproducible ani drugiego przełącznika ścieżki. |
| B20 | Częściowo trafne. Dodano debug, format dla poleceń, walidację model-only/theme-only i 130. `--strict` jest użytecznym rozszerzeniem, lecz brak tej funkcji nie jest sprzecznością — odroczono ją. |
| B21 | Trafne. Podsumowanie jest w SVG desc, z opisami obiektów. Dodano legendę i zadania P4.9; nie wymaga to czwartego pliku wynikowego. |
| B22 | Trafne. Importy źródłowe są po MVP; lokalne deklarowane assety nie są importami YAML/D2. |
| B23 | Trafne. Discovery kończy się na korzeniu VCS; poza VCS sprawdza tylko katalog modelu. Dalszy config można podać jawnie. Dodatkowy znacznik `root: true` nie jest potrzebny. |
| B24 | Trafne, ale usuwanie wszystkich obcych przestrzeni byłoby zbyt szerokie. Dopuszczono tylko jawnie wymienione inertne metadane/komentarze edytorów, po ograniczonym parsowaniu i kontroli aktywnej zawartości. Pozostały fail-closed oraz bezstratna serializacja liczb; dodano fragment-only xlink. |
| B25 | Trafne ryzyko integracyjne, do sprawdzenia w P0. Profil wyjścia dopuszcza kontrolowane embedded fonty i ikony; profil wejściowego logo nie dopuszcza dowolnych data URLs. Markdown/foreignObject są zabronione. Hash fontu po subset/re-encoding nie musi być hashem oryginalnego TTF — rejestrujemy pochodzenie i oba etapy. |
| B26 | Trafne. Progi to 4.5:1 dla tekstu i 3:1 dla istotnych elementów nietekstowych, z właściwym wyjątkiem logo. Niespełnienie mierzalnych progów jest błędem; nie oznacza to automatycznej certyfikacji WCAG. |
| B27 | Trafna diagnoza, niewystarczająca recepta. Osobne replace + manifest-last wykrywa niekompletny zestaw, ale nie zachowuje atomowo poprzedniego. Wybrano blokadę pisarza, niezmienne generacje i atomową podmianę wskaźnika katalogu. Czytelnik musi przypiąć generację; P0 weryfikuje ograniczenia systemu plików i eksport wyników. |
| B28 | Trafne. Akceptujemy jeden początkowy BOM UTF-8 do parsowania; hash źródła nadal obejmuje oryginalne bajty. |
| B29 | Trafne. Pętla jest po MVP i wymaga zdefiniowanej metryki przed implementacją. |
| B30 | Trafne. Platforma adaptera to decyzja P7.1; adapter sieciowy jest poza gwarancją offline deterministycznego rdzenia. |

## C. Selekcja

| Punkt | Ocena i rozstrzygnięcie |
|---|---|
| C1 | Zasadna niejasność, choć filtr po przejściu też mógłby być świadomą semantyką. Wybrano wykluczenia jako bariery przed BFS; wymagany jest test A–B–C. |
| C2 | Nie przyjęto. AND między filtrami jest spójne i było już jawne. Proponowana suma uprzywilejowuje ids i zmienia znaczenie kombinacji bez jednoznacznej korzyści. Dodano testy zamiast nowego operatora. |
| C3 | Trafne. Brak wypełnionych kryteriów oznacza wszystkie elementy; puste listy są nieaktywne. |
| C4 | Trafne. Przechodzenie w obie strony, niezależnie od grotów, wyłącznie po kandydujących relacjach. |
| C5 | Trafna redundancja. Usunięto pusty krok; nie rozszerzono select.ids o boundaries. |
| C6 | Trafne. Pusty wynik jest błędem FFV. |
| C7 | Trafne. Jawnie wybrane i wykluczone ID daje ostrzeżenie; exclude nadal wygrywa. |
| C8 | Trafne. relatedDepth jest ograniczone do 1–10 i niedozwolone przy wyłączonej ekspansji. |
| C9 | Trafne. Usunięto „major boundaries”; granice wynikają z jednoznacznych display defaults, nie ukrytej ważności. |
| C10 | Trafne. Fallback mapowań to odrębny wersjonowany zasób frameworka używający wyłącznie wymaganych tokenów. |

## D. Plan wykonawczy

| Punkt | Ocena i rozstrzygnięcie |
|---|---|
| D1 | Trafne. Alpha.1 ma jawnie ograniczony config; nieobsługiwany branding jest odrzucany, nie ignorowany. Hash wbudowanego theme powstaje w P3; sanitizer provenance tylko wtedy, gdy coś rzeczywiście przetworzono. |
| D2 | Trafne. P4 sprawdza wiele widoków infrastruktury; pełny test trzech rodzin jest bramką P5 wymagającą również P4. |
| D3 | Trafne. Lista konsumentów jest ustalana w P0.1 przed eksperymentami. |
| D4 | Trafne. P2 odpowiada za struktury/referencje, P3 za bazowy hash/manifest, P4 za przetwarzanie i rozszerzenie istniejącego kolektora. |
| D5 | Trafne braki śledzenia. Dodano render, ikony, rules, podsumowanie/legendę, walidację View, lint, rejestr diagnostyk i build-all. Prompty/skills przydzielono do P7 lub jawnie P9, zamiast uznać wszystkie za MVP. |
| D6 | Preferencja implementacyjna, nie niemożliwość schematu. Przyjęto walidację semantyczną konfliktu slotów z FFC, aby poprawić lokalizację i wyjaśnienie. |
| D7 | Trafne. Wymagane końce wszystkich relacji; typ referencji rozstrzyga semantyka, nie identyczny regex ID. |
| D8 | Trafne. P2 jawnie odrzuca żądania nieobsługiwanego jeszcze pełnego sprawdzania assetów; P4 domyka validate. |
| D9 | Trafne. Macierz obejmuje kryteria 1–18, w tym rodziny, dostępność, review, ikony, błędy i build-all. |
| D10 | Trafne. Bramka P0 obejmuje akceptację ADR i przegląd człowieka; nie jest deklarowana jako sam test CI. |
| D11 | Trafne. P7 może iść równolegle do P6 po kontraktach P5; P8 wymaga obu. |
| D12 | Trafne. Dodano bezpieczeństwo, metadane i review do akceptacji. Reuse flow/sequence jest warunkiem P5, nie alpha.1. |

## E. Uwagi redakcyjne i granice weryfikacji

Ujednolicono zakres TALA/vendor icons jako post-MVP, język licencji, tokeny dekoracji, deterministyczne wejścia, wymagany hash D2, układ modułów review/adaptera oraz tekstową prezentację encrypted. Usunięto obserwację konkretnej wersji D2 z normatywnego kontraktu na rzecz zadania ADR w P0.

Lokalne `d2 --help` dla v0.9.0 potwierdza `d2 validate`, opcje fontów TTF i `--pad`. To nie jest wybór wersji produkcyjnej ani dowód przenośności SVG, poprawności sanitizera czy embeddowania wszystkich zasobów. Te eksperymenty pozostają jawnymi bramkami P0. Próba weryfikacji przez wyszukiwarkę nie powiodła się z powodu błędu uwierzytelnienia narzędzia; nie przypisuję jej żadnych potwierdzonych ustaleń.

Zmiany są na poziomie dokumentacji. Nie stanowią implementacji ani wyniku testów nieistniejących jeszcze schematów/compilerów.
