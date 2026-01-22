# WiseLock

CLI do tworzenia i refaktoryzacji **pełnych, dużych zmian** w repozytorium — **wraz z testami**.

`wiselock` bierze Twoje zadanie (jedna linia), zbiera kontekst z repo, prosi model(e) o wygenerowanie **diffa/patcha**, opcjonalnie przeprowadza **review diffa**, a potem (jeśli chcesz) **aplikuje zmiany** i **uruchamia testy**.

```bash
wiselock "Add request validation to /users" \
  --model gpt-5.2 \
  --apply \
  --run-tests
```

---

## Dlaczego to ma sens

W realnych projektach najtrudniejsze nie jest „napisanie kilku linii”, tylko:

- zebranie odpowiedniego kontekstu (gdzie to zmienić, co jest powiązane),
- utrzymanie spójności w wielu plikach,
- zmiany bez łamania kontraktów i edge-case’ów,
- dopilnowanie testów.

`wiselock` porządkuje ten proces w powtarzalny pipeline.

---

## MVP (CLI)

MVP to jedno polecenie, które robi:

1. **zbiera kontekst repo**
2. **buduje prompt**
3. **woła API**
4. **zapisuje diff (patch)**
5. **git apply** (opcjonalnie)
6. **testy** (opcjonalnie)

Przykład:

```bash
wiselock "Add request validation to /users" \
  --model gpt-5.2 \
  --apply \
  --run-tests
```

### Tryb interaktywny (następny krok, opcjonalnie)

```bash
wiselock "Add validation to /users" --interactive
```

Wtedy narzędzie:

- generuje patch,
- pokazuje diff,
- pyta:

```text
Apply changes? [y/N]
```

Domyślnie w trybie nieinteraktywnym rekomendowane jest zachowanie „bezpieczne” (dry-run) i wymaganie jawnego `--apply`.

---

## Kluczowy detal: rozdzielenie „tworzenia” od „oceny”

To jedna z najważniejszych zasad w tym podejściu:

- **Prompt A (generator):** generuje diff/patch
- **Prompt B (reviewer):** krytycznie ocenia diff/patch

Korzyści:

- model nie „broni” własnych decyzji (mniej confirmation bias),
- dostajesz inny tryb myślenia (kreatywny vs sceptyczny),
- łatwiej ustawić automatyczne reguły: co zaakceptować, co zablokować.

### Review musi dostać: diff + oryginalne zadanie

Review bez zadania jest praktycznie bezsensowne („czy kod jest ładny?”).
Dopiero razem z zadaniem reviewer może sprawdzić:

- czy zmiana realizuje wymagania,
- czy nie robi za dużo,
- czy nie zmienia kontraktów/public API,
- czy nie psuje edge-case’ów,
- czy testy pokrywają krytyczne ścieżki.

---

## Format odpowiedzi z review (3 części)

Review prompt powinien zwracać stabilny, maszynowo-parsowalny format.

### 1) Verdict (enum, nie skala)

```text
VERDICT:
- APPROVE
- APPROVE_WITH_WARNINGS
- REJECT
```

Enum jest stabilniejszy niż liczby/oceny, łatwiej go użyć jako gate.

### 2) Risk assessment (konkret!)

Przykład:

```text
RISKS:
- [HIGH] Missing validation for optional fields
- [MEDIUM] No tests for error paths
- [LOW] Minor style inconsistency
```

### 3) Confidence score (opcjonalnie, pomocniczo)

```text
CONFIDENCE: 0.82
```

To nie musi być „blokada”, raczej sygnał diagnostyczny.

### Rekomendowany minimalny payload (przykład JSON)

```json
{
  "verdict": "APPROVE_WITH_WARNINGS",
  "risks": [
    {"level": "HIGH", "note": "Missing validation for optional fields"},
    {"level": "MEDIUM", "note": "No tests for error paths"},
    {"level": "LOW", "note": "Minor style inconsistency"}
  ],
  "confidence": 0.82,
  "notes": [
    "Consider adding tests for 400 response on invalid payload.",
    "Keep validation errors consistent with existing endpoints."
  ]
}
```

---

## Flow w CLI (praktycznie)

Domyślny pipeline:

```text
wiselock "Add validation to /users"
  ↓
[generate diff]
  ↓
[review diff]
  ↓
[decision logic]
  ↓
[apply?] → [run tests?]
```

Zasada: **użyj innego prompta / stylu do review niż do generowania**.

- Generator: kreatywny, skupiony na rozwiązaniu
- Reviewer: sceptyczny, defensywny, „paranoiczny”, jak security/staff engineer

---

## Wybór modeli i „custom per task”

Założenie: różne etapy mogą używać różnych modeli (i różnych dostawców).

Przykładowe flagi (docelowo):

```bash
wiselock "..." \
  --model gpt-5.2 \
  --review-model claude-opus \
  --apply
```

Oraz profile, np.:

- `--profile fast` – jeden model, minimalny kontekst, bez review
- `--profile safe` – generate + review, bez auto-apply
- `--profile paranoid` – generate + review + test plan + testy, ścisłe gate’y

---

## Dokładny plan (multi-model workflow)

Proponowany „pełny” przepływ:

1. **GPT-5.2** robi plan
2. **Claude** robi *peer review* planu i poprawia
3. **Claude** implementuje
4. **GPT-5.2** robi finalny review kodu + szuka ukrytych bugów

Dlaczego to działa:

- rozdzielasz role (planowanie vs implementacja vs audyt),
- peer review planu łapie nieporozumienia zanim powstanie kod,
- final review od innego modelu zwiększa szanse wykrycia subtelnych regresji.

---

## Testy (przyszłość) – sugerowany workflow

Dalszy etap: osobny pipeline pod testy.

- **GPT-5.2 Pro** decyduje *co testować* (minimum sensowne), wskazuje ryzykowne miejsca.
- **Claude Opus 4.5** pisze konkretne testy (szybko, bez lania wody).

Takie rozdzielenie bywa bardzo skuteczne, bo „co testować” wymaga priorytetyzacji i myślenia ryzykiem, a samo pisanie testów to często ciężka, mechaniczna praca.

### Piramida testów (orientacyjnie)

```text
                ┌──────────────┐   ← bardzo mało (1–8%)
                │   E2E / UI   │
           ┌────┴──────────────┴────┐
           │  Kontraktowe / Komponentowe  │   ← 10–20%
      ┌────┴────────────────────────────┴────┐
      │          Integracyjne                 │
┌─────┴───────────────────────────────────────┴─────┐   ← najwięcej (60–85%)
│              Unit / komponentowe                  │
└────────────────────────────────────────────────────┘
```

W skrócie:

- większość wartości daje szybka warstwa unit/komponentowa,
- integracyjne pilnują kleju i realnych zależności,
- kontrakty chronią API i boundary,
- E2E/UI tylko tam, gdzie najbardziej boli regresja.

---

## Jak `wiselock` działa „pod spodem”

### 1) Zbieranie kontekstu repo

Cel: podać modelowi **tylko to, co potrzebne**, w ramach budżetu tokenów.

Typowe źródła kontekstu:

- struktura repo (drzewo katalogów),
- metadane: język, framework, package manager, test runner,
- pliki kluczowe (np. routery, kontrolery, schematy, modele),
- grep/ripgrep po endpointach/nazwach,
- zależności (importy, moduły powiązane),
- istniejące testy w okolicy zmiany,
- konwencje (lint/format, styleguide, error handling).

Docelowo: tryby kontekstu, np. `smart` (wycina tylko relevant), `full` (większy kontekst), `custom` (include/exclude globs).

### 2) Budowanie prompta

W praktyce: prompt to „pakiet” składający się z:

- opisu zadania (Twoje polecenie),
- kontekstu repo (wybrane fragmenty),
- instrukcji formatu wyjścia (patch unified diff),
- ograniczeń (nie dotykaj X, zachowaj kontrakt Y, styl Z),
- wymagań dot. testów (jeśli włączone).

### 3) Wołanie API

Warstwa „adapterów” per provider/model:

- OpenAI
- Anthropic
- (w przyszłości) inne

### 4) Zapis patcha

- patch zapisany do pliku (audyt, reprodukcja),
- metadane: model, hash kontekstu, czas, parametry.

### 5) Aplikacja zmian

- preferowane: `git apply` (czysto, odwracalnie)
- opcjonalnie: `git apply --check` przed właściwą aplikacją

### 6) Uruchomienie testów

- uruchamiane tylko na żądanie (`--run-tests`) lub w profilu,
- konfigurowalne komendą (`--test-command "..."`).

---

## Decyzje i gate’y

Przykładowa logika decyzji po review:

- `APPROVE` → można auto-apply (jeśli `--apply`/profil na to pozwala)
- `APPROVE_WITH_WARNINGS` → pokaż ostrzeżenia, preferuj tryb interaktywny
- `REJECT` → nie aplikuj; wyświetl powody i sugerowane poprawki

Dodatkowe gate’y (opcjonalnie):

- blokuj auto-apply przy `[HIGH]` risk,
- wymagaj testów przy zmianach w krytycznych katalogach (np. auth, payments),
- wymagaj czystego `git diff --check`.

---

## Konfiguracja

Docelowe źródła konfiguracji (od najwyższego priorytetu):

1. flagi CLI
2. `.wiselock.yml` (per repo)
3. zmienne środowiskowe (API keys, default provider)

Przykład (koncepcyjnie):

```yaml
profile: safe
models:
  generate: gpt-5.2
  review: claude-opus
context:
  mode: smart
  include:
    - "src/**"
    - "tests/**"
  exclude:
    - "dist/**"
    - "**/*.min.js"
tests:
  command: "npm test"
  on_apply: true
```

---

## Bezpieczeństwo i higiena

Zalecenia projektowe (guardrails):

- **domyślnie nie aplikuj zmian** bez `--apply` (albo bez potwierdzenia w `--interactive`),
- loguj dokładnie: patch, parametry, verdict review,
- limity na rozmiar kontekstu i liczbę plików,
- wyraźne rozróżnienie: „generator” nie uruchamia komend, tylko produkuje patch,
- test runner uruchamia tylko komendy z config/flag (brak „dowolnej egzekucji” z outputu modelu).

---

## Roadmap

- `--interactive` z wygodnym viewerem diffa
- profile i polityki gate’ów
- automatyczna pętla naprawcza:
  - testy fail → streszczenie błędów → poprawka patcha → ponowny review → retry
- lepsze „smart context” (ranking plików po relewancji)
- integracja z CI (komentarze do PR / check run)
- wsparcie dla monorepo i wielu runnerów testów

---

## Kontrybucje

Ten repozytorium ma być praktyczne: prosty core, a reszta jako moduły (provider adapters, UI, profiles).

Jeśli chcesz pomóc:

- dodaj adapter do kolejnego providera,
- dopracuj heurystyki kontekstu,
- zaproponuj format review, który lepiej się parsuje i jest bardziej stabilny,
- dodaj przykładowe profile dla popularnych stacków.

---

## Licencja

Do ustalenia (np. MIT/Apache-2.0).
