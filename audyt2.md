# Audyt 2 — dokumentacja FlowFrame

Zakres: [docs/prd.md](docs/prd.md), [docs/technical-spec.md](docs/technical-spec.md), [docs/implementation-plan.md](docs/implementation-plan.md) (stan na commit `4469c74`).

Waga: **K** – krytyczny (blokuje implementację), **W** – wysoki, **Ś** – średni, **N** – niski.

## Najważniejsze, od których warto zacząć

1. **K** Nie da się nigdzie wybrać silnika layoutu dla `build` (B1).
2. **K** Widok `integration-flow` niczym się nie różni od `infrastructure` (B2).
3. **K** System Model i View mają ten sam `schemaVersion` (A1).
4. **W** Alpha.1 (P3) zależy od rzeczy, które powstają dopiero w P4 (D1).
5. **W** Nie ma mapowania diagnostyk na kody wyjścia i jest sprzeczność 0/1 vs 5 (A5).
6. **W** Selekcja przechodzi przez elementy, które potem są wykluczane (C1).

---

## A. Sprzeczności (wewnątrz dokumentów i między nimi)

**A1 (K) Ten sam `schemaVersion: flowframe/v1` dla System Model i View.** Występuje w [prd.md:291](docs/prd.md#L291), [prd.md:536](docs/prd.md#L536) i w manifeście [prd.md:1064-1065](docs/prd.md#L1064-L1065). Tymczasem P1.2 wymaga „schemaVersion discriminators”, a wspólna wartość niczego nie rozróżnia.
→ Użyć `flowframe-model/v1` i `flowframe-view/v1`, spójnie z `flowframe-config/v1`, `-theme/v1` i `-manifest/v1`.

**A2 (W) ID kroków i relacji: SHOULD czy MUST?**
- [prd.md:371](docs/prd.md#L371) mówi SHOULD.
- Tabela w [prd.md:475](docs/prd.md#L475) wymaga `id` dla kroków.
- IR „MUST retain source model IDs”, a tech spec buduje `relation_by_id`.

→ Zmienić na MUST dla relacji i kroków. Znika wtedy potrzeba reguł wyprowadzania ID.

**A3 (W) Czy relacja może wskazywać boundary?** Przykład FFM102 mówi „Unknown element **or boundary** ID” ([prd.md:1043](docs/prd.md#L1043)). Selekcja działa jednak tylko na elementach ([prd.md:618](docs/prd.md#L618)).
→ W v0.1 końce relacji to tylko elementy. Poprawić przykład (przy okazji `model.yaml` → `system-model.yaml`).

**A4 (W) Sequence IR z „grouping constructs”.** Tak jest w [technical-spec.md:333](docs/technical-spec.md#L333), a PRD 9.4 i 11.3 mówią, że grupy są post-MVP i MUST NOT wystąpić.
→ Usunąć z tech spec.

**A5 (W) Kody wyjścia.**
- P2.7 mówi, że `validate` zwraca „0 lub 1” ([implementation-plan.md:455](docs/implementation-plan.md#L455)).
- PRD 15.2 każe jednak w walidacji sprawdzać sanitizer logo i nadpisania per widok, a to są naruszenia polityki lub bezpieczeństwa, czyli kod 5.
- Nigdzie nie ma mapowania rodzin diagnostyk (FFC/FFS/FFM/…) na kody wyjścia.
- Nie ma priorytetu, gdy w jednym uruchomieniu wystąpią np. błędy z kategorii 1 i 5.

→ Dodać tabelę „prefiks lub kategoria → exit code”. Zdefiniować, co jest „policy” (np. pola stylu w źródle, niebezpieczny asset, zdalny zasób). Ustalić pierwszeństwo, np. 6 > 5 > 4 > 3 > 1.

**A6 (Ś) Kiedy ustalić próg jakości AI?** Są cztery różne odpowiedzi:
- PRD §25 ([prd.md:1606](docs/prd.md#L1606)): „Stage 0 lub 1”,
- tech spec §22 pkt 9: Stage 0,
- PRD §22 ([prd.md:1540](docs/prd.md#L1540)): „po Stage 0”,
- plan P7.4: „z baseline P0/P7”.

Adapter AI powstaje dopiero w Stage 4.
→ Próg ustalać w P7, przed wydaniem. Usunąć go z list decyzji Stage 0.

**A7 (Ś) Menedżer pakietów.** PRD §25 pkt 2 ma go jako otwartą decyzję, tech spec §3 już wybrał `uv` + Hatchling, P0.6 każe decydować ponownie, a P1.1 zakłada Hatchling.
→ Uznać za zdecydowane (ADR w P0.1) i usunąć z list otwartych.

**A8 (Ś) Kto odpowiada za wygląd relacji?**
- §13.5 ([prd.md:893](docs/prd.md#L893)) mówi, że theme.
- §13.3 ([prd.md:844](docs/prd.md#L844)) mówi, że „relation semantics remain framework-owned”.
- Schemat theme ma tylko tokeny kolorów, bez wzoru linii, grotu i grubości.
- Jeśli custom theme steruje wzorem linii, nie da się zagwarantować czytelności w skali szarości.

→ Wzór linii i grot należą do frameworka. Theme dostaje kolory i ewentualnie tokeny grubości.

**A9 (Ś) AI a surowe kolory.** §4 ([prd.md:128](docs/prd.md#L128)) mówi bezwzględnie „AI MUST NOT define raw colors”, a §5 ([prd.md:164](docs/prd.md#L164)) pozwala na to przy wyraźnej prośbie o branding.
→ Zawęzić §4 do System Model i View.

**A10 (Ś) Ścieżki w manifeście są „relative to the build root”, ale to pojęcie nie jest zdefiniowane** ([prd.md:1092](docs/prd.md#L1092)). W przykładzie `flowframe.yaml` i `diagram.d2` są względne wobec różnych katalogów.
→ Źródła względne wobec project root, wyjścia względne wobec katalogu manifestu.

**A11 (Ś) Hash theme: surowe bajty czy forma kanoniczna?** [technical-spec.md:208](docs/technical-spec.md#L208) mówi o surowych bajtach, a [implementation-plan.md:619](docs/implementation-plan.md#L619) o „canonical theme metadata”.
→ Surowe bajty.

**A12 (Ś) Struktura repozytorium w PRD §20 nie zgadza się z planem.**
- Testy: `tests/schema|semantic|golden|snapshots` ([prd.md:1396](docs/prd.md#L1396)) vs `tests/unit|contract|integration|golden|security` (P1.1).
- Przykłady: `examples/infrastructure/…` vs `examples/payments/` (§26).
- Wyjście: `build/` vs `build/payments/infrastructure/`.
- `flowframe.yaml` w korzeniu repo ([prd.md:1341](docs/prd.md#L1341)) zostanie znaleziony przez wyszukiwanie w górę dla każdego przykładu, który nie ma własnego configu.

→ Ujednolicić. PRD powinno odsyłać do tech spec.

**A13 (N) Warstwy walidacji:** PRD §15.1 ma 11, tech spec §8 ma 13, z inną kolejnością i podziałem.

**A14 (N) Format lokalizacji diagnostyk.** PRD używa „YAML path” (`relations[2].target`), tech spec JSON Pointer z linią i kolumną. PRD wymaga też pliku i ścieżki YAML w *każdej* diagnostyce, co jest niemożliwe dla FFR, FFD i FFX.
→ Jeden format, np. `system-model.yaml:42:9 /relations/2/target`. Lokalizacja opcjonalna dla błędów renderera i wewnętrznych.

**A15 (N) Metadane dostępności:** PRD §13.7 mówi MUST, tech spec [§12.3](docs/technical-spec.md#L463) mówi SHOULD.

**A16 (N) Self-message:** P5.4 mówi „self-message if supported” ([implementation-plan.md:773](docs/implementation-plan.md#L773)), a PRD wymaga go normatywnie.
→ Zweryfikować w P0.3 i traktować jako obowiązkowy.

**A17 (N) Selekcja zależna od rodziny.** P5.1 ma „family-specific selection expansion”, a tech spec §9.1 „sequence-specific contract”. PRD definiuje jeden algorytm i żadnego kontraktu dla sekwencji (patrz B3).

**A18 (Ś) Wersjonowanie a zamknięte schematy.** Tech spec [§21](docs/technical-spec.md#L690) pozwala dodać pole opcjonalne bez zmiany wersji. Przy `additionalProperties: false` starszy FlowFrame odrzuci jednak nowszy plik.
→ Każde nowe pole oznacza nową wersję kontraktu albo wprowadzamy wersję minor lub `minFlowframeVersion`.

**A19 (N) Tytuł i zakres PRD.** PRD nazywa się „Product Requirements and Technical Specification”, a jego §21 duplikuje plan, co grozi rozjazdem.
→ Zmienić tytuł. §21 zastąpić tabelą Stage → P z linkiem do planu.

**A20 (Ś) Czy `review` należy do MVP?** Nie ma go w zakresie MVP (§3.1) ani w kryteriach akceptacji, a plan implementuje go w P7.5. Tryb `visual` koliduje z „advanced visual review” zapisanym jako post-MVP (§3.2).
→ Zdecydować jawnie. Propozycja: tryby syntax, semantic i policy w MVP; `visual` i `source-conformance` jako opcjonalne adaptery.

## B. Luki w kontraktach

**B1 (K) Brak wyboru silnika layoutu dla `build`.**
- `build` nie ma `--layout`.
- View nie ma pola silnika, a config nie ma sekcji renderera.
- Opcja `allow-engine-fallback` ([prd.md:198](docs/prd.md#L198)) nie jest nigdzie zdefiniowana.

→ Dodać `flowframe.yaml: render.layoutEngine: elk` jako domyślne, `--layout` w CLI jako nadpisanie i flagę `--allow-engine-fallback`. Wybrany silnik zapisać w manifeście.

**B2 (K) `integration-flow` niczym się nie różni od infrastructure.**
- Selekcja filtruje tylko elementy. Relacji nie da się filtrować po `semantic`, `interaction` ani tagach, nie ma też `exclude` dla relacji.
- Flow IR ma zawierać „events or data artifacts” ([prd.md:646](docs/prd.md#L646)), ale model nie ma pola na payload.
- Role producer/processor/store/consumer nie mają reguł wyprowadzania.

→ Dodać:
- `select.relations: {semantics, interactions, tags}` i `exclude.relations`,
- opcjonalne pole relacji `payload` (albo `dataObject`),
- regułę ról, np. `database/cache/storage` → store, reszta według stopnia wejścia i wyjścia,
- domyślne wartości `display` i `layout` dla flow.

**B3 (W) Widoki sequence są niedookreślone.**
- §10.1 mówi, że każdy widok deklaruje selekcję, a przykład sequence jej nie ma.
- Kolejność uczestników nie jest zdefiniowana.
- Nie wiadomo, czy wyświetlać boundaries ani czy działa `exclude`.
- Opcjonalne pola kroku nie są opisane (`protocol` jest w przykładzie, ale nie w tabeli).
- Nie jest powiedziane, że `from` i `to` wiadomości to tylko elementy (jest to powiedziane wyłącznie dla notatki).
- „relation-backed messages” (P5.3, tech spec §8.1 „referenced-relation sets”) nie mają pola w modelu.

→ Ustalić:
- sequence w v0.1 nie ma `select` ani `exclude`,
- uczestnicy są w kolejności pierwszego wystąpienia, z opcjonalnym jawnym `participants:`,
- pola kroku to `protocol` i opcjonalnie `relationId`,
- `from` i `to` wskazują tylko elementy.

**B4 (W) Brak wartości domyślnych.** Nie są zdefiniowane:
- `includeRelated`,
- `display.boundaries` (pominięte oznacza wszystkie czy żadne?),
- katalog pól `display` dla każdej rodziny,
- `layout.direction` i `profile`,
- czy `semantic` i `interaction` są wymagane,
- czy `theme` w configu jest wymagany (P1.3 jest tu niejednoznaczne),
- pełna zawartość wbudowanego `flowframe-default`: theme, pozycje, domyślny footer.

→ Dodać do PRD tabelę wartości domyślnych i pełną zawartość `flowframe-default`.

**B5 (W) Brak macierzy dozwolonych kombinacji właściwości relacji.** Wymagają jej PRD §15.2 ([prd.md:1015](docs/prd.md#L1015)) i P2.5.
→ Dodać macierz `semantic × interaction × directionality`. Przykłady: `dependency` wymaga `not-applicable`; `event` + `synchronous` daje ostrzeżenie; `replication` może być `bidirectional`.

**B6 (W) Ikony.**
- `technology` to wolny tekst i nie ma mapowania technologia → ikona.
- W planie nie ma pakietu roboczego dla ikon, choć PRD Stage 2 je obejmuje.
- Ikona w D2 to ścieżka do pliku, co koliduje z zakazem ścieżek absolutnych w D2 i z samodzielnym `render`.

→ Dodać indeks ikon (klucz → plik) w icon packu i opcjonalne pole `icon` albo `technologyId`. Ikony osadzać przy kompilacji. Dodać P4.x „Generic icon set”.

**B7 (W) Statyczne pliki D2 kontra custom theme.** `resources/d2/theme.d2` i pozostałe są statyczne, a theme projektu jest własny. Klas nie da się więc trzymać statycznie. Importy są zakazane przez lint i wprowadzałyby ścieżki.
→ `diagram.d2` jest samowystarczalny, a klasy są generowane z rozwiązanego theme. `resources/d2` to wewnętrzne szablony.

**B8 (W) `compile` + `render` ≠ `build`.**
- `render` nie ma `--config`, więc daje body bez dekoracji.
- Diagram w PRD §7 pomija kompozytor.
- Nie wiadomo, czy `compile` woła `d2 validate` (wtedy potrzebuje D2 i może zwrócić exit 3) ani czy `render` pisze manifest.
- `render` przyjmuje dowolne D2, co przeczy [prd.md:166](docs/prd.md#L166) („compatibility mode w przyszłości”).

→ Udokumentować `render` jako niskopoziomowy, tylko body, akceptujący wyłącznie D2 z nagłówkiem FlowFrame. Alternatywnie usunąć go z MVP.

**B9 (W) Brak budowania całego projektu.**
- Kryterium akceptacji 15 i P8.3 wymagają zbudowania wszystkich diagramów jednym poleceniem.
- CLI buduje tylko jeden widok.
- Stałe nazwy `diagram.*` kolidują przy wielu widokach.

→ Dodać `flowframe build --all` z wykrywaniem widoków (lista `views:` w `flowframe.yaml` albo glob) i domyślne wyjście `<out>/<view-id>/`.

**B10 (Ś) Luki w schemacie theme.**
- Brak typografii i fontów (przykład ich nie ma). D2 CLI przyjmuje, o ile wiem, tylko TTF (`--font-regular/bold/…`); do weryfikacji w P0.
- Token `network` istnieje, ale nie ma `mappings.boundaries`.
- Brak domyślnej tabeli mapowań dla 13 rodzajów elementów i 9 semantyk: dokąd trafiają `actor`, `gateway`, `cache`, `identity-provider`, `authentication`, `replication`?
- Tokeny mają różne typy (`decoration-gap` to liczba, reszta to kolory).
- Brak tokenu poziomego paddingu.
- Nie wiadomo, czy `id` musi być równe nazwie katalogu.
- Nie wiadomo, gdzie są metadane licencji: w `LICENSES.md` czy w `theme.yaml`.
- Nie wiadomo, czy theme projektu może przesłonić ID wbudowanego.

→ Uzupełnić schemat. Przesłanianie wbudowanych ID proponuję odrzucić.

**B11 (Ś) Pola, na które dokumenty się powołują, ale których nie ma w schematach:**
- `state` external/deprecated ([prd.md:717](docs/prd.md#L717)),
- emphasis i priority ([prd.md:718](docs/prd.md#L718), [prd.md:136](docs/prd.md#L136)),
- `description` (P1.4),
- intent widoku,
- znaczniki niepewności dla AI ([prd.md:1236](docs/prd.md#L1236), P7.2) przy zamkniętych schematach.

→ Dodać `status`, `description`, `purpose` w widoku oraz `assumptions[]` (albo plik sidecar).

**B12 (Ś) Granica systemu.**
- Nie wiadomo, czy `parentId` może wskazywać `system.id`.
- Nie wiadomo, czy elementy najwyższego poziomu są w systemie, czy poza nim.
- Widok nie ma opcji pokazania granicy systemu.
- Reguła „external systems outside” (§14) nie jest walidowana.

→ `parentId: <system.id>`, `display.systemBoundary`, walidacja, że `external-system` i `actor` nie są wewnątrz systemu.

**B13 (Ś) Reguły zawierania.** Nie wiadomo, czy element może być rodzicem elementu ani jakie są reguły zagnieżdżania boundaries (np. `network` w `namespace`).
→ `parentId` wskazuje tylko boundary lub system. Zagnieżdżanie swobodne, jedynie z wykrywaniem cykli.

**B14 (Ś) Braki w słowniku.**
- Nie da się zamodelować „Internet” z przykładu §19.2, bo „external network element” z §9.3 nie ma swojego kind.
- Brak aplikacji klienckiej lub mobilnej.
- Brak boundaries `region` i `account/subscription`, typowych dla infrastruktury.
- `platform` i `operations` w `audience` się pokrywają.

**B15 (Ś) Gramatyka ID.**
- Brak regexu i limitu długości.
- Zarezerwowane słowa D2 (`label`, `style`, `shape`, `icon`, `near`, `direction`, `classes`, `vars`, …) mogą być ID.
- Nie wiadomo, czy jest jedna przestrzeń nazw, czy „per namespace” (P2.5).
- Tagi nie mają gramatyki.

→ `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`, maksymalnie 64 znaki, jedna przestrzeń nazw. Writer D2 zawsze prefiksuje albo cytuje ID, a zarezerwowane słowa są odrzucane.

**B16 (Ś) Tekst prezentacji.**
- Wymóg „MUST NOT contain D2/HTML/SVG markup” jest nieegzekwowalny i da fałszywe alarmy (np. „a < b”).
- Nie jest zdefiniowana jednostka „characters”.
- Brak zawijania: tytuł 120 znaków czy źródło 500 znaków robią bardzo szeroki canvas.

→ Zawsze plain text i escaping. Odrzucać znaki kontrolne. Liczyć code pointy po NFC. Zawijanie deterministyczne z maksymalną szerokością.

**B17 (Ś) Algorytm dekoracji.**
- Rozszerzanie szerokości per dekoracja ([prd.md:911](docs/prd.md#L911)) nie uwzględnia tytułu `top-center` i logo `top-right` w jednym pasie, więc mogą na siebie nachodzić.
- Wyrównanie w pionie w pasie nie jest zdefiniowane.
- Padding poziomy nie jest zdefiniowany.
- `--pad` D2 wpływa na bounds body.

→ Szerokość = max(body, suma szerokości slotów górnych + odstępy).

**B18 (Ś) Manifest.**
- `generatedAt` powoduje diff przy każdym buildzie, jeśli artefakty są commitowane.
- Brak strefy czasowej.
- JSON Schema nie ma słowa kluczowego „volatile” (tech spec §15).
- Nie jest zapisywane pochodzenie ikon i fontów.
- `flags` mogą zawierać absolutne ścieżki fontów.
- `assetProcessing` jest wymagany już w P3, gdzie nie ma sanitizera.

→ UTC, obsługa `SOURCE_DATE_EPOCH` (albo `--no-timestamp`), własne słowo kluczowe `x-flowframe-volatile`, opcjonalny `assetProcessing`, relatywizacja flag.

**B19 (Ś) Odnajdywanie binarki D2.** Tech spec mówi „not silently from PATH in reproducible/CI mode”, ale ani ten tryb, ani mechanizm wyszukiwania nie są zdefiniowane.
→ `--d2-path` lub `FLOWFRAME_D2`, weryfikacja checksumy z listy wbudowanej, jawna flaga `--reproducible`.

**B20 (Ś) Brakujące opcje CLI:**
- `--debug` (na nim polega [technical-spec.md:611](docs/technical-spec.md#L611)),
- `--strict` (ostrzeżenia jako błędy),
- `--format json` dla `compile` i `build`,
- walidacja samego modelu albo samego theme,
- kod wyjścia po SIGINT (P6.5 mówi o „interrupted”, ale spoza zakresu 0–6).

**B21 (Ś) Tekstowe podsumowanie i legenda.** Wymagane podsumowanie tekstowe (§13.7) nie ma określonego miejsca (SVG `<desc>`? osobny plik?) ani pakietu w planie. Legendy nie ma w ogóle, a wzory linii bez legendy są niezrozumiałe.

**B22 (N) Importy.** Ograniczenia importów są w [prd.md:1120](docs/prd.md#L1120), ale mechanizmu importów nie ma. „D2 imports” w §6.1 jest wymienione jako zaleta, choć lint je zabrania.
→ Usunąć albo oznaczyć jako post-MVP.

**B23 (N) Wyszukiwanie configu w górę do korzenia systemu plików** może złapać przypadkowy `~/flowframe.yaml`.
→ Zatrzymywać się na korzeniu VCS albo na znaczniku `root: true`.

**B24 (N) Sanitizer w praktyce.** Logo z Illustratora czy Inkscape'a zawierają `<metadata>`, komentarze, przestrzenie `inkscape:`/`sodipodi:` i `xlink:href`. Zasada „reject, not strip” odrzuci większość takich plików.
→ Deterministycznie usuwać komentarze, metadata i przestrzenie edytorów w ramach kanonikalizacji. Wspierać `xlink:href`. Kanonikalizacja liczb musi być bezstratna.

**B25 (N) Weryfikator SVG a wyjście D2.** D2 osadza fonty jako `data:` w `<style>`, a etykiety markdown renderuje przez `foreignObject`.
→ Weryfikator dopuszcza `data:` z fontami D2. Generator nigdy nie emituje etykiet markdown. Potwierdzić w P0.3.

**B26 (N) Kontrast custom theme.** Nie wiadomo, czy niespełniony kontrast to błąd, czy ostrzeżenie. Brak progów.
→ Wpisać progi WCAG: 4.5:1 dla tekstu, 3:1 dla elementów nietekstowych.

**B27 (N) Publikacja transakcyjna.** Rename niepustego katalogu nie jest atomowy, a nic nie chroni przed współbieżnymi buildami.
→ `os.replace` dla każdego pliku, z manifestem na końcu jako znacznikiem commitu, plus plik blokady.

**B28 (N) Decyzja o BOM** ([technical-spec.md:217](docs/technical-spec.md#L217)) nie trafiła na żadną listę decyzji.
→ Zdecydować teraz, proponuję akceptować BOM i go usuwać.

**B29 (N) Pętla ulepszeń (§18.3).** Nie wiadomo, czy to MVP, a „measurable improvement” nie ma metryki.
→ Oznaczyć jako post-MVP.

**B30 (N) AI.** Wybór platformy adaptera AI nie jest na liście otwartych decyzji. Brak jawnego wyłączenia adaptera AI z wymogu pracy offline.

## C. Błędy logiczne w selekcji ([prd.md:613-620](docs/prd.md#L613-L620))

**C1 (W) Przechodzenie przez elementy, które potem są wykluczane.** Łańcuch A–B–C, `relatedDepth: 2`, B wykluczone: C zostaje dodane, choć jest odłączone.
→ Wykluczać przed przechodzeniem, tak żeby wykluczone węzły były barierą (rekomendowane). Alternatywnie ostrzeżenie.

**C2 (Ś) AND między `ids` a `tags` jest nieintuicyjny.** `ids: [x], tags: [runtime]` wybierze x tylko wtedy, gdy ma tag `runtime`.
→ Seed = `ids` ∪ (`tags` ∩ `kinds`).

**C3 (Ś) Niezdefiniowany seed.** Brak definicji dla `select`, które ma tylko `includeRelated`, oraz dla pustych tablic.
→ Brak wypełnionych kryteriów oznacza wszystkie elementy. Pusta tablica oznacza pole niewypełnione.

**C4 (Ś) Kierunek przechodzenia nie jest określony.**
→ Oba kierunki, niezależnie od `directionality`.

**C5 (N) Krok 6 („omit empty boundaries”) nigdy nie zadziała**, bo dodawani są tylko przodkowie wybranych elementów. Albo jest zbędny, albo `select.ids` ma móc wskazywać boundary (przydatne: „pokaż całą sieć X”).

**C6 (N) Pusta selekcja nie jest zdefiniowana.** P3.1 odsyła do „PRD semantics”, których nie ma.
→ Błąd FFV.

**C7 (N) Element z `select.ids` wycięty przez `exclude` znika po cichu.**
→ Ostrzeżenie.

**C8 (N) „Traversal limits” nie są zdefiniowane.** Brak górnego limitu `relatedDepth`.

**C9 (N) `detail: low` mówi o „major boundaries”**, co nie jest zdefiniowane i nie wiadomo, jak ma się do `display.boundaries`.

**C10 (N) Nie wiadomo, z którego wbudowanego theme brać zastępcze mapowania**, gdy wbudowanych theme będzie kilka (np. dark).
→ Domyślne mapowania jako osobny zasób frameworka. Niezmiennik: odwołują się tylko do tokenów wymaganych, więc zawsze da się je rozwiązać.

## D. Plan implementacji

**D1 (W) P3 (alpha.1) zależy od P4.**
- Przykład zawiera `flowframe.yaml`, a branding i dekoracje są dopiero w P4. Nie wiadomo, co P3 robi z logo, tytułem i stopką.
- Manifest w P3.7 potrzebuje hasha theme, implementowanego dopiero w P4.2.
- Schemat manifestu wymaga sekcji sanitizera.

→ W P3: config z samym `theme: flowframe-light` albo branding walidowany, ale nierenderowany, z diagnostyką info. Hash theme przenieść do P3. `assetProcessing` opcjonalny.

**D2 (Ś) P4.7 buduje fixture'y wszystkich trzech rodzin** ([implementation-plan.md:703](docs/implementation-plan.md#L703)), ale P5 nie jest jego zależnością.
→ Przenieść do wyjścia z P5, gdzie ta kontrola już jest.

**D3 (Ś) Kolejność w P0.** P0.3 i P0.4 testują „every supported documentation consumer”, a lista wspieranych konsumentów jest ustalana dopiero w P0.6.
→ Przenieść tę decyzję do P0.1.

**D4 (Ś) Duplikacja zakresu.**
- Tokeny i mapowania są w P2.5 i w P4.1.
- Rozwiązywanie theme jest w P2.2 i w P4.2.
- Manifest jest w P3.7 i w P4.8.

→ Wyraźnie rozdzielić: P2 to struktura i referencje, P4 to przetwarzanie.

**D5 (Ś) Brakujące pakiety robocze:**
- komenda `render`,
- ikony,
- `rules/*.md` poza profilem SVG,
- prompty `review-diagram`, `simplify-view` i `compare-with-source` oraz `skills/`,
- podsumowanie tekstowe i metadane dostępności,
- **semantyczna walidacja View**: family/subtype, istnienie `scenarioId`, referencje `select.ids`, reguły `relatedDepth`, zgodność layoutu z silnikiem (P2.5 obejmuje tylko model i config),
- lista reguł lintu D2,
- rejestr kodów diagnostyk (`docs/diagnostics.md`),
- `build --all`.

**D6 (N) Konflikt slotów** jest i w fixture'ach schematu (P1.3), i w semantyce (P2.5).
→ Tylko semantycznie, z kodem FFC. Diagnostyka jest wtedy lepsza.

**D7 (N) P1.4.** Pomija relacje `bidirectional`; powinno być „wszystkie relacje”. „Syntaktyczne” ograniczenie `participant` do kształtu ID elementu nic nie daje, bo ID elementu i boundary mają ten sam kształt. Usunąć.

**D8 (N) `validate` z P2 nie spełni PRD §15.2 aż do P4**, bo sanitizer powstaje w P4. Zapisać to jawnie.

**D9 (N) Macierz śledzenia (§16) jest niepełna.** Brakuje dostępności, kodów wyjścia, flow i sequence, review, ikon.
→ Lepiej zmapować 15 kryteriów akceptacji z PRD §22.

**D10 (N) „Exit criteria pass in CI”** nie pasuje do P0, gdzie bramkami są ADR-y i przegląd ludzki.

**D11 (N) P7 nie musi czekać na P6.** Wystarczą mu kontrakty z P5.

**D12 (N) Kryteria akceptacji PRD §22.**
- Brakuje kryterium bezpieczeństwa (korpus security „fails closed”), metadanych dostępności i `review`.
- Warunek w [prd.md:1650](docs/prd.md#L1650) nie jest bramką alpha.1, tylko P5. Doprecyzować.

## E. Drobne i redakcyjne

- „TALA … licensed for commercial use” ([prd.md:191](docs/prd.md#L191)) → „requires a commercial license”.
- Tabela statusu podaje TALA jako silnik opcjonalny, choć to post-MVP. Drzewo `icons/` pokazuje pakiety vendorów, które też są post-MVP. Oznaczyć jako przyszłe.
- „Decoration classes” ([prd.md:903](docs/prd.md#L903), [prd.md:1470](docs/prd.md#L1470)): dekoracje nie są w D2. Zamienić na „tokeny/style”.
- Tech spec §2 mówi „normalized inputs”, PRD „same inputs”. Ujednolicić.
- „`d2 validate` is present in D2 v0.9.0” ([technical-spec.md:388](docs/technical-spec.md#L388)) zapisuje konkretną wersję przed przypięciem jej w Stage 0. Przenieść do ADR.
- Hash binarki D2 jest „where practical” (tech spec §11), a weryfikacja checksumy to MUST (§3). Ujednolicić.
- Układ pakietów w tech spec §5 nie ma modułów `review/` ani adaptera AI.
- Nie jest zdefiniowane, jak renderować `encrypted`.
