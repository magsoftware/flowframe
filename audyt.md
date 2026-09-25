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
