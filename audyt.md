# Audyt FlowFrame PRD

- **Dokument:** `docs/prd.md`
- **Data:** 2026-09-25
- **Zakres:** spójność, kompletność, koncepcje i pomysły

---

## Ogólna ocena

PRD jest koncepcyjnie dojrzały i gotowy jako podstawa implementacji. Najmocniejsze elementy: rozdzielenie System Model / View / Projection / Family IR / D2 generator, ortogonalne właściwości relacji (`semantic` + `interaction`), reklasyfikacja `internet` i `trust`, deterministyczny build z manifestem i testami golden/snapshot, polityka silników layoutu oraz „source-conformance review".

Poniżej problemy i propozycje rozwiązań.

---

## 1. Niespójności wewnętrzne (`prd.md`)

### 1.1. `maxDepth` vs `relatedDepth`
- §10.3 używa `select: ... maxDepth: 3`.
- §10.5 opisuje: „`includeRelated` performs one hop unless `relatedDepth` is provided".
- **Problem:** dwa różne pola na to samo; nie wiadomo też, czy `maxDepth` dotyczy zagnieżdżenia, czy trawersowania relacji.
- **Propozycja:** ujednolicić do `relatedDepth` (liczba przeskoków dla `includeRelated`), a głębokość zawierania nazwać osobno `containmentDepth`. Zsynchronizować przykład z tekstem.

### 1.2. `system` jako klucz główny i jako `boundary.kind`
- §8.2 korzeń modelu to `system: {id, label}`.
- §9.2 w `boundary.kind` jest `system`.
- **Problem:** „system" znaczy jednocześnie cały system i rodzaj granicy/podsystemu.
- **Propozycja:** przemianować boundary kind `system` na `subsystem`; korzeń zostawić jako `system`.

### 1.3. Kolizja nazwy `direction`
- §9.3 relacja ma opcjonalne pole `direction`.
- §12.1 layout ma `direction: up|down|left|right`.
- **Problem:** ta sama nazwa na dwóch poziomach modelu.
- **Propozycja:** dla relacji użyć `orientation` lub `flowDirection`; `direction` zarezerwować dla layoutu.

### 1.4. Niezdefiniowane enumy `audience` i `detail`
- §10.1 mówi o „audience and detail level"; przykłady używają `audience: architect`, `detail: medium`, ale brak dozwolonych wartości i znaczeń.
- **Propozycja:** zdefiniować `detail: low | medium | high` (co każdy poziom filtruje) oraz listę `audience` (`architect`, `developer`, `security`, `business`, …).

### 1.5. `kind: message` w scenariuszach nie ma słownika
- §8.2 kroki scenariusza mają `kind: message`; §11.3 Sequence IR wspomina `self-messages`, `notes`, grupy `alt/loop/parallel`.
- **Problem:** brak enumeracji dozwolonych typów kroków.
- **Propozycja:** dodać listę step kinds (np. `message`, `self-message`, `note`) w §9 lub §11.3 i powiązać ze schematem.

### 1.6. Semantyka selekcji niepełna
- §10.3 używa `select.tags` + `includeRelated` + `maxDepth`; nie jest jasne, skąd startuje trawersowanie i jak te pola współgrają.
- **Propozycja:** opisać algorytm selekcji krok po kroku: (1) zbiór startowy z `tags`/`kinds`, (2) domknięcie relacyjne do `relatedDepth`, (3) dodanie przodków zawierania, (4) usunięcie wykluczonych.

---

## 2. Niespójności `README.md` ↔ `prd.md`

| Temat | README (nieaktualne) | PRD (aktualne) |
|---|---|---|
| Layout engine | „TALA by default, ELK as a fallback" | ELK baseline, TALA optional |
| Typy komponentów | `user, external, web, …, network, cluster, namespace, security` | `actor, external-system, web-application, …, secret-store, security-control`; `network/cluster/namespace` → boundary kinds |
| Typy relacji | płaska lista `sync, async, data, control, trust, internet, …` | `semantic` + `interaction`; `trust` i `internet` usunięte |
| Pipeline | „Diagram Intent → Architecture Model → D2" | System Model + View → projection → Family IR → D2 |
| Struktura repo | `lib/` z `layouts.d2, icons.d2`, `schema/` z `diagram-intent.schema.json` | `src/flowframe/`, `pyproject.toml`, `tests/`, `schema/{system-model,view}.schema.json`, `lib/boundaries.d2` |

**Propozycja:** zsynchronizować README z PRD (layout engine, słownik, pipeline, struktura).

---

## 3. Luki kompletności

1. **Brak definicji wartości `semantic`** (§9.3) — jest lista, brak jednolinijkowych znaczeń (`request` vs `event` vs `data-access` vs `data-flow`).
   - **Propozycja:** dodać tabelkę definicji.
2. **`manifest.json` bez jawnego kontraktu** — §16.1 wymienia pola, ale nie jako listę.
   - **Propozycja:** wydzielić „Manifest fields" (wersje, hashe, engine, flags, theme, timestamp).
3. **`web-application` vs `service`** oraz **`actor` vs `external-system`** — bez rozróżnienia.
   - **Propozycja:** jedna linijka definicji dla każdego `kind`.
4. **`detail`/`audience` oraz boundary kind `system`** — formalnie nie są w „Open decisions".
   - **Propozycja:** dodać do §25.

---

## 4. Koncepcje — mocne strony i pomysły

### Mocne strony (zostawić)
- Rozdzielenie System Model / View / Projection / Family IR / D2 generator — czyste i testowalne.
- `semantic` + `interaction` zamiast przeładowanego „relation type".
- Reklasyfikacja `internet` i `trust` (nie jako typy relacji).
- Deterministyczny build + manifest + golden/snapshot testy.
- Polityka silnika (brak cichego przełączania, `allow-engine-fallback`).
- „Source-conformance review" jako ochrona przed halucynacjami AI.
- Konkretne, mierzalne kryteria akceptacji (§22).

### Pomysły do rozważenia
- `detail: low|medium|high` jako preset projekcji (które pola `display` są domyślnie włączone).
- Sekcja „Glossary" z wszystkimi enumami w jednym miejscu.
- `View` z obsługą wielu `scenarioId` (dziś §10.4 ma jedno).

---

## Podsumowanie

Do poprawy przede wszystkim: **`maxDepth`/`relatedDepth`**, **boundary kind `system`**, **kolizja `direction`**, **brakujące enumy** oraz **synchronizacja README z PRD**.

---

## 5. Mechanizm personalizacji — krytyka i rekomendacje (dodatek, 2026-09-25)

Mechanizm `flowframe.yaml` + Theme/Brand Pack + dekoracje jest spójny i dobrze wpleciony w PRD oraz README. Uwagi do dopracowania (sugestie, bez ingerencji w dokumenty):

1. **Niespójna nazwa schematu konfiguracji.** Plik to `schema/project-config.schema.json` (§20), a `schemaVersion` to `flowframe-config/v1` (§13.2, §16.1). **Rekomendacja:** ujednolicić do jednej nazwy (np. `flowframe-config.schema.json` + `flowframe-config/v1`).

2. **Niejasna lokalizacja wbudowanych motywów.** §13.3: „Theme IDs resolve first from the project `themes/` directory and then from built-in themes", ale nie wiadomo, gdzie fizycznie są motywy wbudowane (repo czy dane pakietu) i względem czego liczony jest „project `themes/`". **Rekomendacja:** korzeń = katalog rozwiązanego `flowframe.yaml`; motywy wbudowane trzymać w danych pakietu, nie w `themes/` projektu.

3. **Sanitizacja SVG logo bez whitelisty.** §16.4 zabrania skryptów, referencji zewnętrznych i `foreignObject`, ale nie definiuje, co jest dozwolone — sanitizer może zepsuć logo oparte o gradienty/filtry/clipPath/wewnętrzne `url(#id)`. **Rekomendacja:** jawna whitelista cech SVG: dozwolone `linearGradient`, `radialGradient`, `clipPath`, `mask`, wewnętrzne `url(#id)`; zabronione `<script>`, `foreignObject`, handlery zdarzeń, zewnętrzne `url(http…)`, `@import`, `<style>` z referencjami zewnętrznymi.

4. **Mechanizm rezerwacji miejsca pod dekoracje nieokreślony.** D2/TALA/ELK nie rezerwują paddingu dla tytułu/stopki/loga dodawanych po layoucie; §13.6 wymaga „reserve sufficient padding", ale nie mówi jak. **Rekomendacja:** dekoracje renderować w warstwie otaczającej z marginesem canvas liczonym z tokenów (`decoration-gap`), a test „bez nakładania na treść" uwzględnić w spike Stage 0 (obecny wpis to tylko „verify title, footer and embedded local logo rendering").

5. **Stopka nie obsługuje tekstu statycznego bez `{source}`.** Gdy widok nie ma `presentation.source`, stopka jest pomijana w całości — nie da się wyświetlić stałego tekstu. **Rekomendacja:** dopuścić template bez placeholderów (tekst statyczny) lub jawnie udokumentować to ograniczenie.

6. **Escaping placeholderów.** §13.2 mówi o „escaped `{source}` placeholder", ale nie definiuje reguły escapowania literalnych `{`/`}`. **Rekomendacja:** zdefiniować regułę (np. `{{` → `{`) spójnie z walidacją „unknown placeholders are errors".

7. **Redundancja limitów logo.** W theme są per-asset `maxWidth`/`maxHeight` (§13.3), a §25/12 pyta o „Maximum logo dimensions and SVG asset size limits". **Rekomendacja:** rozdzielić per-asset bounds (theme) od globalnych limitów (walidacja/bezpieczeństwo) i doprecyzować oba.

8. **Determinizm sanitizera (opcjonalne).** Wynik sanitizacji logo zależy od wersji sanitizera. **Rekomendacja:** rejestrować wersję/algorytm sanitizera w manifeście dla pełnej reprodukowalności.

---

## 6. Audyt `technical-spec.md` i `implementation-plan.md` (2026-09-25)

Oba dokumenty są wysokiej jakości, dobrze rozwarstwione (spec: kontrakty/architektura/bezpieczeństwo; plan: fazy z bramkami wyjścia) i wzajemnie spójne. Przy okazji domykają wcześniejsze uwagi: pełna gramatyka stopki (`{{`/`}}`, stopka statyczna — spec §12.2), rozdzielenie limitów per-asset od globalnych (§13.1), `source`/`target` wymagane dla relacji nieskierowanych (§8.1), normatywny profil sanitizera jako osobny dokument (§13.2 + `rules/svg-logo-profile-v1.md`).

Główne problemy to **rozjazd z PRD** (który spec sam uznaje za autorytatywny) oraz **rozjazd kontraktu CLI**.

### Problemy i rekomendacje

1. **Nazwy schematów: PRD vs spec (konflikt autorytetu).** PRD §20: `project-config.schema.json`, `theme.schema.json`, `manifest.schema.json`; spec §6.1 i plan P1.3/P1.6: `flowframe-config.schema.json`, `flowframe-theme.schema.json`, `flowframe-manifest.schema.json`. Spec deklaruje „if the documents disagree, follow the PRD" — a więc to błąd do naprawy. **Rekomendacja:** ujednolicić w PRD §20 do nazw `flowframe-*`.

2. **Kontrakt CLI rozjechany między 4 dokumentami.** PRD §17 i README: pozycyjne `MODEL VIEW`, `--out`, `render diagram.d2 --layout elk`. Spec §16: flagi `--model/--view/--config`, `--input/--output`. **Rekomendacja:** wybrać jedną formę — sugeruję flagi (`--model`, `--view`, `--config`, `--output-dir`), bo są spójne z `--config` i rozszerzalne — i zaktualizować PRD §17 oraz README. Pozycyjne `MODEL VIEW` zostawić tylko, jeśli jest to świadomy wybór ergonomii.

3. **Gramatyka stopki: spec idzie dalej niż PRD.** PRD §13.2 mówi tylko o placeholderze `{source}`; spec §12.2 definiuje literal + escapowanie `{{`/`}}` + stopkę statyczną (plan P0.4/P4.5 to testuje). **Rekomendacja:** przenieść pełną gramatykę do PRD §13.2 (PRD jest źródłem prawdy), a w specu zostawić samą implementację.

4. **Lokalizacja wbudowanego motywu.** PRD §20 pokazuje `themes/flowframe-light/` w korzeniu repo; spec §5/§6.3 pakuje go jako `resources/themes/flowframe-light/`. **Rekomendacja:** ujednolicić; w PRD §20 rozdzielić „project `themes/`" (motywy użytkownika) od pakowanych zasobów wbudowanych.

5. **Brak kodu wyjścia dla błędu wewnętrznego.** PRD §17.1 ma kody 0–5, bez kodu awarii; spec §16 mapuje `FFX` → 1, czyli ten sam kod co błąd walidacji. Skrypty nie odróżnią „zły input" od „awaria narzędzia". **Rekomendacja:** dodać osobny kod (np. 6) w PRD i spec dla nieoczekiwanych błędów wewnętrznych.

6. **Determinizm tekstu vs. rendering u konsumenta (największe ryzyko techniczne).** Spec §12.3 wymaga deterministycznego pomiaru bez przeglądarki, ale tekst SVG renderuje konsument (font rendering), więc identyczny pomiar ≠ identyczny wygląd wszędzie. **Rekomendacja:** ADR Stage 0 powinien wprost rozstrzygnąć: (a) deterministyczny pomiar (fontTools) + tekst jako `<text>` (lepsza dostępność, wygląd zależny od konsumenta) **albo** (b) konwersja tekstu na ścieżki (pełna przenośność, gorsza a11y). Nie mieszać milcząco.

7. **`d2 validate` — do potwierdzenia w P0.** Spec §8.10/§11 i plan P0.2 zakładają tryb validate/check D2. **Rekomendacja:** potwierdzić w spike, czy D2 udostępnia `validate`/`check`; jeśli nie — polegać na kodzie wyjścia renderu i `d2 fmt --check`.

8. **Open decisions: spec §22 pomija PRD §25 #7 (polityka wyświetlania technologii/protokołów) i #10 (dark theme).** **Rekomendacja:** dopisać obie pozycje do spec §22 albo jawnie oddelegować (dark theme → P9; protocol policy → ADR Stage 0/1).

9. **Konflikt slotów (title + logo w tym samym top slocie) nie jest w PRD §13.2.** Spec §12.3 i plan P1.3/P4.6 traktują to jako błąd, ale PRD tego nie mówi. **Rekomendacja:** dodać regułę konfliktu slotów do PRD §13.2.

10. **`review` i `compare-layouts` w spec §16 to tylko „…".** **Rekomendacja:** dodać zarys sygnatur i kodów wyjścia albo oznaczyć jako post-MVP/P7 (obecnie `review` ma wymagania w PRD §18.2, a brakuje kontraktu CLI).

11. **Package layout: drobna rozbieżność.** PRD §20 `generators/`, `model.py`; spec §5 `generation/`, `domain/model.py`. **Rekomendacja:** spec §5 jako kanoniczny, PRD §20 uprościć lub zsynchronizować.

### Konkluzja

Plan jest wykonalny i dobrze sekwencjonowany (P0 spike → kontrakty → walidacja → pionowy slice → branding → pozostałe rodziny → hardening → AI → release). Do naprawy przed implementacją: **nazwy schematów, kontrakt CLI i uzupełnienie PRD o gramatykę stopki oraz konflikt slotów** — bo PRD jest autorytatywny, a dziś to spec jest od niego bogatszy.

### Rozstrzygnięcie po weryfikacji

1. **Nieaktualne.** Bieżący PRD §20 używa już `flowframe-config.schema.json`, `flowframe-theme.schema.json` i `flowframe-manifest.schema.json`.
2. **Zasadne.** Przyjęto nazwane opcje `--model`, `--view`, `--config`, `--input`, `--output` i `--output-dir` we wszystkich dokumentach.
3. **Nieaktualne.** Bieżący PRD §13.2 zawiera już tekst statyczny, `{source}`, `{{`/`}}` oraz pełne reguły błędów.
4. **Nieaktualne.** Bieżący PRD §20 umieszcza motyw wbudowany w `src/flowframe/resources/themes/flowframe-light/` i oddziela go od motywów projektu.
5. **Zasadne.** Dodano kod wyjścia 6 dla nieoczekiwanych błędów wewnętrznych (`FFX`).
6. **Zasadne i krytyczne.** Stage 0 musi wybrać dla całego SVG tekst z przypiętymi/osadzonymi fontami albo konwersję do ścieżek oraz udokumentować skutki dla przenośności i dostępności.
7. **Potwierdzone, nie jest błędem.** Lokalny D2 v0.9.0 i oficjalny manual udostępniają `d2 validate`. P0 nadal weryfikuje tę funkcję dla ostatecznie przypiętej wersji.
8. **Częściowo zasadne.** Politykę technologii/protokołów dodano do decyzji Stage 0. Dark theme nie jest decyzją otwartą: pozostaje jawnie post-MVP.
9. **Nieaktualne, ale poprawiono czytelność.** Reguła była już w PRD §13.6; §13.2 otrzymał bezpośrednie odesłanie i jawny przykład konfliktu `top-left`.
10. **Zasadne.** Dodano pełne sygnatury i semantykę `review` oraz `compare-layouts`; drugie polecenie jest jawnie post-MVP.
11. **Zasadne.** PRD pokazuje teraz ten sam układ pakietu (`domain/`, `generation/` itd.) i wskazuje specyfikację techniczną jako kanoniczną dla szczegółów.
