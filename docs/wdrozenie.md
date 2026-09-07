# Wdrożenie — instrukcja krok po kroku

Stan na 2026-08-22, wersja pluginu 0.2.0. Wszystko po stronie właściciela;
agent nie ma uprawnień do żadnego z tych kroków.

## Kolejność

```
token  →  pomiar na fixture'ach  →  kalibracja agenta  →  runner  →  testowy PR
```

Od 0.2.0 pomiar precyzji **nie wymaga self-hosted runnera** — wystarczy
`claude` CLI z tokenem i `node scripts/zmierz.mjs`. Nie ma sensu stawiać
infrastruktury pod narzędzie, którego jeszcze nie zmierzyliśmy.

Krok 0 (gałąź domyślna) jest wykonany po stronie agenta — `main` wypchnięty,
zostaje przestawienie w ustawieniach repozytorium.

---

## KROK 0 — gałąź domyślna (30 sekund)

Repo `ai-pr-guardian` było puste, więc GitHub ustawił gałąź domyślną na
`claude/ai-pr-guardian-github-aptb2v`. Workflow pobiera plugin przez
`actions/checkout` bez `ref`, czyli z gałęzi domyślnej — powinna nazywać się
`main`.

`main` jest już wypchnięty. Zostaje:

    https://github.com/krzysiek2115op/ai-pr-guardian/settings/branches

→ *Default branch* → ikona przełączenia → wybierz `main` → **Update**.

Kontrola: strona główna repo pokazuje `main` jako wybraną gałąź.

---

## KROK 1 — sekret z kluczem (5 minut)

### Najpierw wybierz wariant

| | `ANTHROPIC_API_KEY` | `CLAUDE_CODE_OAUTH_TOKEN` |
|---|---|---|
| Skąd | platform.claude.com | `claude setup-token` lokalnie |
| Rozliczenie | osobne, za tokeny | z limitu Twojej subskrypcji |
| Wpływ na Twoją pracę | **żaden** | przeglądy PR-ów zjadają Twój limit |
| Koszt | ~$0,60–0,90 za PR kodu, czyli ~$36–54/mies. przy 2 PR-ach dziennie | 0 zł dodatkowo |

**DECYZJA WŁAŚCICIELA (2026-08-22): `CLAUDE_CODE_OAUTH_TOKEN`.**

Powód: zero ryzyka finansowego w etapach kalibracji, gdzie potrzeba
kilkunastu przebiegów. Realne zużycie zmierzymy przed decyzją o płaceniu.

Próg przełączenia na klucz API: gdy strażnik po raz drugi zablokuje pracę
w środku innego zadania. Wtedy ~$36–54 miesięcznie kupuje spokój, a decyzja
opiera się na pomiarze, nie na szacunku. Przełączenie to podmiana sekretu —
szablon obsługuje oba warianty i nie wymaga edycji pliku.

### Wariant A — klucz API

1. `https://platform.claude.com` → **API Keys** → **Create Key**
2. Skopiuj klucz (pokaże się raz)
3. `https://github.com/MatthewPlugins/Pod-strona-Szkolenia/settings/secrets/actions`
4. **New repository secret**
   - Name: `ANTHROPIC_API_KEY`
   - Secret: wklejony klucz
5. **Add secret**

Warto od razu ustawić limit wydatków: platform.claude.com → *Limits*.

### Wariant B — token subskrypcji

1. Lokalnie w terminalu: `claude setup-token`
2. Skopiuj wynik
3. Ten sam ekran sekretów, ale nazwa: `CLAUDE_CODE_OAUTH_TOKEN`

### Kontrola

Lista sekretów pokazuje jedną pozycję o właściwej nazwie.
Wartości nie zobaczysz nigdy więcej — to jest w porządku.

---

## KROK 2 — self-hosted runner (15 minut)

Maszyna: ta, na której stoi Postgres w podmanie. Wymagania: `git`, `curl`,
`tar`, Node 22+. Wszystko już masz — projekt tego używa.

**Ważne:** self-hosted runner nie ma preinstalowanego oprogramowania, które
ma `ubuntu-latest`. Nasz workflow potrzebuje tylko git i Node, więc jest bezpieczny.

### Rejestracja

1. `https://github.com/MatthewPlugins/Pod-strona-Szkolenia/settings/actions/runners`
2. **New self-hosted runner** → **Linux** → **x64**
3. GitHub wyświetli gotowe komendy z jednorazowym tokenem.
   **Wykonaj je dosłownie, w podanej kolejności** — token wygasa po godzinie,
   a wersja runnera w komendach jest aktualna. Nie przepisuj ich z pamięci
   ani z żadnej instrukcji, w tym z tej.

   Sekwencja wygląda mniej więcej tak (ale weź WERSJĘ ZE STRONY):

       mkdir actions-runner && cd actions-runner
       curl -o actions-runner-linux-x64-X.Y.Z.tar.gz -L <adres ze strony>
       tar xzf ./actions-runner-linux-x64-X.Y.Z.tar.gz
       ./config.sh --url https://github.com/MatthewPlugins/Pod-strona-Szkolenia --token <TOKEN ZE STRONY>

4. `config.sh` zada cztery pytania. Enter przy każdym przyjmuje domyślne:
   - *runner group* → Enter
   - *name of runner* → Enter albo własna nazwa, np. `dom-podman`
   - *additional labels* → **Enter** (doda `self-hosted`, `Linux`, `X64`)
   - *work folder* → Enter

   Etykieta `self-hosted` jest tą, której używa nasz workflow. Jeśli dodasz
   własne, workflow nadal zadziała — `self-hosted` jest dokładany zawsze.

### Uruchomienie jako usługa

`./run.sh` działa tylko dopóki terminal jest otwarty. Dla stałej pracy:

    sudo ./svc.sh install
    sudo ./svc.sh start
    sudo ./svc.sh status

### Kontrola

Ekran *Settings → Actions → Runners* pokazuje runnera ze statusem
**Idle** i zieloną kropką.

### Trzy rzeczy, o których warto wiedzieć

1. **Nie włączaj tego runnera dla repozytoriów publicznych.** Przy publicznym
   repo każdy może wysłać PR-a, a workflow z forka wykonałby się na Twojej
   maszynie. Przy prywatnym repo z dwoma zaufanymi osobami — bezpiecznie.

2. **Nie przenoś na razie `ci.yml` na self-hosted.** Job `baza` używa
   kontenera usługowego `postgres:17-alpine`, a kontenery usługowe GitHub
   Actions oczekują Dockera. Na podmanie da się to skonfigurować, ale to
   osobna walka. Nasz `straznik-ai.yml` nie ma kontenerów usługowych, więc
   ruszy od razu. Migrację reszty CI potraktuj jako osobne zadanie po
   1 września.

3. **Runner wykonuje kod workflow z Twoimi uprawnieniami.** Dlatego nasz
   workflow ma `contents: read`, a żaden agent nie dostaje `Write`, `Edit`
   ani ogólnego `Bash`. Nie osłabiaj tego.

---

## KROK 3 — zmiana §N1 (10 minut, tekst gotowy do wklejenia)

Twoja dokumentacja mówi w **trzech** miejscach, że agenci AI nie są stałą
obsadą. Automat na każdym PR-cie kodu jest z tym sprzeczny. Poniżej gotowe
teksty — dopisują wyjątek, nie kasują reguły.

### 3a. `docs/WYTYCZNE.md` — dopisz zaraz pod §N1

    ### N1a — wyjątek dla strażnika AI (decyzja właściciela, 2026-08-22)

    §N1 pozostaje w mocy: żaden agent nie działa bez krytyka. Doprecyzowanie
    dotyczy wyłącznie częstotliwości uruchomień, nie zasady.

    Strażnik AI (`ai-pr-guardian`) działa automatycznie na Pull Requestach
    przechodzących filtr ścieżek kodu z `ci.yml:54` — ok. 2 PR-y dziennie wg
    pomiaru z 2026-08-22. PR-y treściowe i dokumentacyjne pozostają nietknięte.

    Powód: cztery klasy błędów z `rejestr/znane-bledy.json` (BLAD-002,
    BLAD-005, BLAD-007, BLAD-011) nie mają strażnika skryptowego i są
    nieuchwytne dla wzorca. Bramka uruchamiana ręcznie nie chroni przed
    błędem, o którym autor nie wie, że go popełnia.

    Ograniczenia bez zmian: krytyk obowiązkowy, strażnik bez Write/Edit/Bash,
    workflow z `contents: read`, znalezisko bez dowodu odrzucane.

### 3b. `CLAUDE.md` — linie 27-28

Obecnie:

    Agenci AI tylko jako bramki jakości (z KRYTYKIEM — nigdy sami);
    codzienna kontrola = SKRYPTY (strażnicy/testy/goldeny).

Zamień na:

    Agenci AI tylko jako bramki jakości (z KRYTYKIEM — nigdy sami);
    codzienna kontrola = SKRYPTY (strażnicy/testy/goldeny).
    Wyjątek: strażnik AI na PR-ach kodu — patrz WYTYCZNE §N1a.

### 3c. `DIAGRAM.md` — linia 180

Obecnie kończy się na: `tylko B7 (agenci to bramki, nie stała obsada)`.

Dopisz w tej samej linii albo tuż pod:

    + strażnik AI na PR-ach kodu od 0.32.0 (WYTYCZNE §N1a)

### 3d. `CHANGELOG.md` — nowy wpis na górze

**Separator w nagłówku to em dash U+2014, nie ASCII `-`.**
Skopiuj poniższy nagłówek dosłownie, nie przepisuj ręcznie —
`straznik-wersji` sprawdza to regexem.

    ## [0.32.0] — 2026-08-22

    ### Dodane

    - `.github/workflows/straznik-ai.yml` — strażnik AI jako dodatkowa bramka
      jakości na PR-ach kodu, wykonywany na self-hosted runnerze
    - `.githooks/pre-push` — blokada pusha przy znalezisku blokującym

    ### Zmienione

    - `docs/WYTYCZNE.md` — dodany §N1a
    - `CLAUDE.md`, `DIAGRAM.md` — odesłania do §N1a

    ### Decyzje właściciela

    - Strażnik AI działa automatycznie na PR-ach przechodzących filtr ścieżek
      kodu, nie jako bramka wywoływana ręcznie. Powód: cztery klasy błędów
      z rejestru bez ochrony skryptowej. Krytyk pozostaje obowiązkowy.
    - Wykonanie na self-hosted runnerze, nie na GitHub Actions — minuty planu
      Free wyczerpane 2026-08-18.
    - Zakres v1: jeden strażnik. Kolejni po pomiarze na fixture'ach.

    ### Dowody

    - (uzupełnić po pierwszym przebiegu na testowym PR-cie)

### 3e. `README.md` — tabela „Stan projektu"

    | **Wersja** | **0.32.0** |

`straznik-wersji` sprawdza dosłownie `**0.32.0**` w README i zestawia
z najwyższym nagłówkiem CHANGELOG. Rozjazd zatrzyma commit.

### Kontrola kroku 3

    node tools/straznicy/straznik-wersji.mjs
    node tools/straznicy/uruchom-wszystkie.mjs

Oba muszą przejść. `straznik-linkow` sprawdzi też, czy odesłania do §N1a
prowadzą do istniejącej kotwicy.

---

## Po trzech krokach — kontrola całości

Zanim otworzysz testowego PR-a, sprawdź łańcuch lokalnie:

    git clone https://github.com/krzysiek2115op/ai-pr-guardian ~/ai-pr-guardian
    cd ~/ai-pr-guardian
    node --test scripts/*.test.mjs        # oczekiwane: 20/20
    claude plugin validate .              # oczekiwane: Validation passed

Potem, w katalogu Pod-strona-Szkolenia, na gałęzi z jakąkolwiek zmianą kodu:

    node ~/ai-pr-guardian/scripts/zakres.mjs --baza origin/plugin-1-sklep-kursow

Powinno wypisać `"kod": true` i `"straznicy": ["straznik-regresji"]`.
Jeśli wypisze `"kod": false` z powodem — filtr ścieżek uznał zmianę za
treściową. To poprawne zachowanie dla PR-a bez kodu.

---

## Czego się spodziewać, gdy coś nie zadziała

| Objaw | Najbardziej prawdopodobna przyczyna |
|---|---|
| Job czeka w nieskończoność „Waiting for a runner" | runner offline albo etykieta nie pasuje do `runs-on: self-hosted` |
| `claude-code-action` pada na starcie | brak Node w PATH runnera-usługi; usługa ma inne środowisko niż Twoja powłoka |
| Brak komentarzy inline, ale job zielony | `--allowedTools` musi nazywać `mcp__github_inline_comment__create_inline_comment` — jest w szablonie, sprawdź czy nie zginęło przy kopiowaniu |
| `zakres.mjs` pada na `git diff` | gałąź bazowa niepobrana; workflow robi `git fetch`, lokalnie zrób sam |
| Strażnik nic nie znajduje na fixture'ach | to jest wynik do zmierzenia, nie awaria — patrz `fixtures/README.md` |

**Najbardziej prawdopodobny punkt awarii to `claude-code-action` na
self-hosted runnerze.** Nie testowałem go w tym środowisku i nie będę
udawał, że wiem, że zadziała. Jeśli padnie, mamy drogę awaryjną: wywołanie
`claude -p` bezpośrednio w kroku `run:`, bez akcji — kosztem komentarzy
inline, które trzeba by wtedy publikować osobnym krokiem.


---

## KROK 4 — pomiar na fixture'ach (dodany w 0.2.0)

Wymaga wyłącznie tokenu. **Nie wymaga runnera.**

    git clone https://github.com/krzysiek2115op/ai-pr-guardian ~/ai-pr-guardian
    cd ~/ai-pr-guardian
    node --test scripts/*.test.mjs      # kontrola: 54/54
    node scripts/zmierz.mjs --sucho     # kontrola: 8 fixture'ów
    node scripts/zmierz.mjs             # właściwy pomiar

Harness buduje dla każdego fixture'a tymczasowe repozytorium git z rejestrem
znanych błędów, kładzie fixture pod ścieżką z nagłówka `// SCIEZKA:`,
zdejmuje wszystkie komentarze i puszcza przegląd.

### Kryterium

| Zbiór | Oczekiwanie |
|---|---|
| `fixtures/prawdziwe/` (4) | znalezisko z właściwym `BLAD-xxx`, utrzymane przez krytyka, waga ≥ MEDIUM |
| `fixtures/falszywe/` (4) | zero znalezisk utrzymanych przez krytyka |

Skrypt liczy osobno **przeoczenia** i **fałszywe alarmy**. Fałszywy alarm
waży więcej — kod strażnika poprawiamy wtedy w pierwszej kolejności.

### Co zrobić z wynikiem

Wklej całe wyjście do rozmowy. Kalibracja polega na poprawianiu
`agents/straznik-regresji.md` i `agents/krytyk.md`, nie skryptów.
