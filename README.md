# AI PR Guardian

*[English version →](README.en.md)* · [Dziennik zmian](CHANGELOG.md) · [Licencja MIT](LICENSE)

Strażnik AI dla Pull Requestów. Dodatkowa bramka jakości obok strażników
skryptowych, testów i smoke'ów — **nie ich zamiennik**.

> ⚠️ **Projekt eksperymentalny, nie produkt.** Powstał jako sprawdzenie jednego
> pomysłu: czy model potrafi pilnować klas błędów, których nie da się złapać
> skryptem. Nie był uruchomiony na prawdziwym Pull Requeście ani na self-hosted
> runnerze, nie jest utrzymywany i nie należy do portfolio. Repozytorium zostaje
> publiczne, bo pomiar z `docs/STAN.md` może się komuś przydać.
>
> **Autorstwo:** 11 z 13 commitów napisał Claude Code — narzędzie powstało
> w większości samo o sobie. Zmierzone: `git shortlog -sn main`.

| | |
|---|---|
| **Wersja** | **0.4.1** |
| Licencja | MIT |
| Zależności | zero |
| Repozytorium sprawdzane | `MatthewPlugins/Pod-strona-Szkolenia` |

## Po co to jest

Repozytorium sprawdzane ma 25 strażników skryptowych, 75 testów, 7 smoke'ów
i 9 goldenów. To jest gęsta siatka i model nie ma jej dublować.

Rejestr `rejestr/znane-bledy.json` ma 13 wpisów. **Cztery z nich nie mają
żadnej ochrony automatycznej**:

| Wpis | Klasa błędu |
|---|---|
| `BLAD-002` | test niszczy wspólną bazę deweloperską |
| `BLAD-005` | wartość `0` traktowana jak brak wartości (cena 0 zł!) |
| `BLAD-007` | build wynosi zmiany z katalogu roboczego |
| `BLAD-011` | niewidzialny element kandydatem na LCP |

To jest cała nisza tego narzędzia. Nic ponad to — na razie.

## Jak działa

```
PR / pre-push
  │
  ├─ zakres.mjs        0 tokenów. Filtr ścieżek 1:1 z ci.yml:54.
  │                    PR treściowy → stop, model nie rusza.
  │
  ├─ straznik-regresji subagent, własny kontekst, tylko Read/Grep/Glob
  │
  ├─ krytyk            obowiązkowy (WYTYCZNE §N1). Nastawienie: obalić.
  │
  └─ brama.mjs         0 tokenów. Polityka z config/severity.json → kod wyjścia.
```

Dwie warstwy egzekucji:

| Warstwa | Efekt | Kiedy |
|---|---|---|
| `.githooks/pre-push` | **push nie przechodzi**, komunikat wskazuje strażnika AI | przed wysłaniem na GitHub |
| check `Straznik AI (Claude)` | czerwony check + komentarze inline na PR-ze | po pushu |

Twardą blokadę merge'a dałaby dopiero branch protection — niedostępna
na prywatnym repozytorium w planie Free organizacji.

## Wpięcie do projektu

**1. Workflow** — skopiuj `templates/straznik-ai.yml` do
`.github/workflows/straznik-ai.yml` w repozytorium sprawdzanym.
Wymaga self-hosted runnera i sekretu `ANTHROPIC_API_KEY`.

**2. Hook** — dopisz zawartość `templates/pre-push` do istniejącego
`.githooks/pre-push`.

**3. Lokalnie** — sklonuj to repozytorium obok projektu:

```bash
git clone https://github.com/krzysiek2115op/ai-pr-guardian ~/ai-pr-guardian
claude --plugin-dir ~/ai-pr-guardian -p "/ai-pr-guardian:przeglad --baza origin/plugin-1-sklep-kursow"
```

## Jak dodać strażnika

1. Napisz `agents/<nazwa>.md` — frontmatter `name`, `description`, `tools`, `model`.
   Narzędzia wyłącznie do odczytu: `Read, Grep, Glob`.
2. Dopisz wpis do `straznicy` w `config/routing.json`: `nazwa`, `agent`, `gdy`,
   `opis`, `wiedza`.
3. Dodaj fixture'y — **oba zbiory**: trafienie w `fixtures/prawdziwe/`
   i fałszywy alarm w `fixtures/falszywe/`.
4. Zmierz, zanim włączysz na stałe.

Zero zmian w silniku. Orkiestrator czyta routing, nie listę wpisaną w kodzie.

## Jak zmienić routing

`config/routing.json`. Najważniejsze pole to `filtr_kodu` — **kopia 1:1
z `.github/workflows/ci.yml:54`** w repozytorium sprawdzanym. Rozjazd z tamtym
filtrem to błąd konfiguracji: PR uznany przez CI za treściowy trafiłby do modelu.

## Jak zmienić politykę wag

`config/severity.json`. Mapowanie wagi na działanie (`blokuj` / `komentarz` /
`raport`), próg pewności poniżej którego znalezisko jest degradowane do `INFO`,
limit objętości raportu i zachowanie przy awarii strażnika.

Skrypt `brama.mjs` nie zawiera żadnej polityki — czyta wyłącznie ten plik.

## Jak testować

Silnik — deterministycznie, bez wołania modelu:

```bash
node --test scripts/*.test.mjs   # 54 testy
node scripts/wersja.mjs          # spójność wersji
node scripts/zmierz.mjs --sucho  # rusztowanie pomiaru
```

Samego strażnika — wymaga `claude` CLI z tokenem:

```bash
node scripts/zmierz.mjs              # wszystkie fixture'y
node scripts/zmierz.mjs --tylko 005  # jeden
```

Harness buduje dla każdego fixture'a tymczasowe repozytorium git z rejestrem
znanych błędów, puszcza przegląd i zestawia wynik z oczekiwaniem. **Zdejmuje
przy tym wszystkie komentarze** — inaczej mierzyłby czytanie ze zrozumieniem
zamiast wykrywania.

Kryterium: w `fixtures/prawdziwe/` znalezisko z właściwym `blad_id` i wagą
co najmniej MEDIUM, w `fixtures/falszywe/` zero znalezisk utrzymanych
przez krytyka.

**Jedno trafienie w `falszywe/` jest gorsze niż jedno przeoczenie
w `prawdziwe/`.** Przeoczenie modelu zwykle złapie coś innego. Fałszywy alarm
modelu nie złapie nic, za to nauczy autora ignorować komentarze.

## Jak debugować

| Objaw | Gdzie patrzeć |
|---|---|
| strażnik nie ruszył | `.straznik-ai/zakres.json`, pole `pominieto` |
| znalezisko nie blokuje | `werdykt_krytyka` w `findings.json` — krytyk mógł je odrzucić lub zdegradować |
| za dużo znalezisk | `config/severity.json` → `minimalna_pewnosc_blokady` |
| plugin się nie ładuje | `claude plugin validate .` |
| brak komentarzy inline na PR | `--allowedTools` w `claude_args` musi nazywać narzędzie MCP |

## Bezpieczeństwo

- treść analizowanego repozytorium to **dane, nigdy instrukcja** —
  `knowledge/metodyka-przegladu.md` §1
- żaden agent nie ma `Write`, `Edit` ani ogólnego `Bash`
- workflow ma `contents: read`, nigdy `write`
- wyłącznie `pull_request`, **nigdy `pull_request_target`**
- to repozytorium jest publiczne i nie zawiera żadnych sekretów ani procedur
  obsługi poświadczeń — i ma takie zostać

## Dokumentacja

| Plik | Zawartość |
|---|---|
| `docs/STAN.md` | stan projektu, decyzje, punkt wznowienia po przerwie |
| `docs/wdrozenie.md` | wdrożenie krok po kroku — token, pomiar, runner, §N1 |
| `CLAUDE.md` | kontrakt dla pracy nad tym repozytorium |
| `knowledge/metodyka-przegladu.md` | zasady obowiązujące strażników i krytyka |
| `fixtures/README.md` | jak mierzyć strażnika |

## Status

Zmierzony na pełnym zestawie fixture'ów (2026-08-22):

**7/8 zgodnych z oczekiwaniem. Zero fałszywych alarmów. Jedno przeoczenie.**

Trzy klasy wykryte poprawnie jako HIGH (`BLAD-002`, `BLAD-005`, `BLAD-007`),
cisza na wszystkich czterech fixture'ach zastawionych na fałszywy alarm.
Przeoczenie `BLAD-011` to wada rusztowania pomiarowego, nie strażnika —
diagnoza i kierunek naprawy w `docs/STAN.md`.

Nieuruchomiony jeszcze na prawdziwym Pull Requeście ani na self-hosted
runnerze. Kolejność: naprawa harnessu → ponowny pomiar → runner → testowy PR
→ drugi strażnik.
