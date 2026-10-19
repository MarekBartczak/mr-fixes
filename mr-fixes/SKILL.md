---
name: mr-fixes
description: Użyj, gdy user wkleja link do merge requesta w GitLabie albo ticketu w Jirze i chce ogarnąć uwagi z code review lub QA — „ogarnij poprawki", „są uwagi w MR-ze", „popraw po CR", „QA coś zgłosił w tickecie" — albo wpisuje /mr-fixes.
argument-hint: "<link do MR-a w GitLabie albo ticketu w Jirze>"
---

# Poprawki do MR-a z uwag w GitLabie i Jirze

Ty analizujesz i proponujesz, user decyduje. Przebieg kończy się lokalnymi commitami i komendą push do skopiowania.

**Zakazy na cały przebieg**, chyba że user wprost każe:
- jakikolwiek zapis w GitLabie lub Jirze: komentarz, odpowiedź, draft, resolve, status, reakcja — także przy uwadze „oczywiście naprawionej";
- `git push`, `git rebase` (wymusza force push), `git stash`, `git reset --hard`, `git merge --abort`;
- edycja kodu przed akceptacją propozycji (krok 5).

## 1. Wejście

Link do MR-a albo ticketu; brak → poproś o link.

- **MR:** z URL-a bierzesz tylko `<projekt>/-/merge_requests/<iid>`, host pomijasz. Pobierasz gałąź źródłową, **docelową** (zawsze z MR-a, nigdy „master" z założenia), `web_url`, `user_notes_count`. Klucz ticketu: z gałęzi → tytułu → opisu. Klucze innych ticketów (np. MR-a bazowego) są poza zakresem.
- **Ticket:** szukasz otwartych MR-ów z kluczem w gałęzi, tytule albo linkach ticketu. Kilka → pytasz; zero → pytasz o gałąź.
- Brak drugiej strony → pracujesz na jednej, odnotowujesz w raporcie.

## 2. Worktree i świeży target

1. Worktree gałęzi szukasz w `git worktree list`. Brak → zakładasz (`git worktree add`, z `--track -b <gałąź> … origin/<gałąź>`, gdy gałęzi nie ma lokalnie) w ścieżce z instrukcji projektu, domyślnie `<repo>/.claude/worktrees/<gałąź>`. Brudny working tree → pokazujesz `git status`, stop.
2. `WT=$(git -C <worktree> rev-parse --show-toplevel)` — każda komenda git jako `git -C "$WT" …`.
3. `fetch origin <gałąź> <target>`. Lokalna za origin → `merge --ff-only origin/<gałąź>`. Rozjazd w obie strony → pokazujesz oba zakresy, pytasz, stop.
4. `merge origin/<target>`. Konflikty rozwiązujesz sam, z intencją obu stron, każdy do raportu: plik, co zostało z której strony, dlaczego. Pytasz tylko, gdy obie strony zmieniły tę samą logikę i historia nie rozstrzyga — merge zostaje wtedy w toku.

## 3. Zebranie uwag

Wszystko, ze wszystkich stron paginacji; duży wynik parsujesz skryptem.

**GitLab** — dyskusje MR-a (zawierają też notatki ogólne), `per_page=100` aż strona wróci niepełna; notatek niesystemowych ma wyjść `user_notes_count`.
- Bierzesz nierozwiązane wątki i notatki ogólne, **także usera** — to często jego własne „do poprawy".
- Pomijasz wątki rozwiązane (każda notatka `resolvable` ma `resolved: true`) — tylko je liczysz — i dyskusje w całości systemowe. Notatka systemowa wewnątrz wątku to poszlaka zmiany, nie powód pominięcia.

**Jira** — wszystkie komentarze, chronologicznie; pobranych mniej niż `total` → dociągasz (`startAt`).
- **Granica:** najnowszy komentarz dowolnego autora, którego cała treść to „Zrobione" albo „Naprawione" (± jedno zdanie, np. „wejdzie z najbliższym wdrożeniem"). Wszystko do niej włącznie pomijasz. Granica działa tylko w Jirze.
- Częściowe „1–2 OK, 3 naprawione" to nie granica — te punkty sprawdzasz w kodzie.
- Ustalenia albo specyfikacja bez prośby o zmianę → jedna pozycja BEZ AKCJI.
- Odwołanie do załącznika, którego nie widzisz → NIEJASNE, pytasz usera, co na nim jest.

Kilka próśb w jednej notatce → osobne pozycje.

## 4. Analiza pozycji

- **Miejsce:** `git show <position.head_sha>:<position.new_path>` pokazuje skomentowany kod; w aktualnym znajdujesz go po treści.
- **Już obsłużona?** `git log --since=<data uwagi> -- <plik>` plus aktualny kod. Wątek „outdated" albo odpowiedź „poprawione" to poszlaka — rozstrzyga kod.
- **Trafna?** Nie przyjmujesz w ciemno. Niezgoda → argument z kodu (plik:linia, test, kontrakt) zamiast poprawki.
- **Powtórzenia** — przy każdej uwadze o wzorcu (idiom, styl, walidacja, nazwa, wywołanie, brakujący test). Recenzent komentuje pierwsze wystąpienie, a ten sam błąd siedzi w N miejscach. Szukasz w całym diffie MR-a (`diff origin/<target>...HEAD`) i w pakiecie, którego dotyczy uwaga. Podajesz liczbę i pełną listę `plik:linia`, osobno w diffie / poza diffem — nigdy „i podobne miejsca". Uwaga jednostkowa → piszesz, czego szukałeś, i że powtórzeń jest 0.

| Status | Kiedy |
|---|---|
| DO POPRAWY | uwaga trafna, kod jej nie spełnia |
| JUŻ NAPRAWIONE | kod spełnia — z commitem |
| NIE ZGADZAM SIĘ | uwaga błędna — z argumentem |
| NIEJASNE | brak informacji — z pytaniem do usera |
| BEZ AKCJI | nie prośba: informacja, odpowiedziane pytanie, status |

## 5. Propozycje i stop

```
Target: zmergowany origin/<target> (5 commitów) · konflikty: 0
Zebrane: GitLab — 4 wątki, 20 rozwiązanych pominiętych · Jira — 2 komentarze po granicy

### 1. GitLab · Autor · src/orders/sync.ts:42 · <web_url>#note_<id pierwszej notatki>
> cytat uwagi, najwyżej 2 linie
Status: DO POPRAWY
Miejsca (4): w diffie — sync.ts:42, sync.ts:88, mapper.ts:13 · poza diffem — legacy.ts:7
Propozycja: co i jak, 1–3 linie; snippet, gdy słowa są niejednoznaczne
Skala: mała

2. Jira · Autor · „skrót uwagi" — JUŻ NAPRAWIONE w a1b2c3d

DO POPRAWY: 1 · JUŻ NAPRAWIONE: 2 · pytania: —
```

Numeracja: GitLab, potem Jira, chronologicznie. Pełny blok dla DO POPRAWY, NIE ZGADZAM SIĘ, NIEJASNE; jedna linia dla JUŻ NAPRAWIONE i BEZ AKCJI.

Zero DO POPRAWY i NIEJASNE → bez stopu, krok 7. W przeciwnym razie **stop** i czytasz odpowiedź dosłownie:
- „ok" bez numerów → wszystkie DO POPRAWY jak w propozycji;
- numery z „tak" / „nie" / „inaczej: …" → tylko wymienione;
- miejsca poza diffem → tylko po wprost wyrażonej zgodzie.

## 6. Poprawki

- Wg numerów, **jeden commit na pozycję** ze wszystkimi jej miejscami; komunikat wg konwencji repo.
- Mała → robisz sam. Duża (wiele plików, spora zmiana testów) → jeden subagent naraz w `$WT`, w prompcie: uwaga, zaakceptowana propozycja, lista miejsc, reguły commitów usera, zakazy z góry. Model średni; najmniejszy do mechanicznych zmian nazw.
- Propozycja w trakcie przestaje pasować (kolejne miejsce, inny skutek, zmiana kontraktu) → wstrzymujesz pozycję i wracasz z nową propozycją.

## 7. Bramki i raport

Bramki raz — gdy były poprawki albo merge targetu coś wniósł: lint/format, typy i testy wg instrukcji projektu, tylko pakiety z `diff --name-only origin/<target>...HEAD`. Fail przez poprawkę → naprawiasz, dokładasz commit. Fail przyniesiony z targetem → do raportu, bez szukania przyczyny.

```
## !<iid> <gałąź> → <target>
Target: … · konflikty: … (jak rozwiązane)

| # | Źródło | Uwaga | Status | Commit |
|---|---|---|---|---|

Bramki: lint ✅ · typy ✅ · testy <pakiet> ✅ 214/214
Do wypchnięcia: 5 commitów
```

Na końcu osobny blok z **bezwzględną** ścieżką worktree:

```bash
git -C /bezwzględna/ścieżka/worktree push origin <gałąź>
```

`rev-list --count origin/<gałąź>..HEAD` = 0 → zamiast komendy „Nic do wypchnięcia". Koniec przebiegu.

## Odpowiedzi na uwagi

Tylko na prośbę usera. **REQUIRED SUB-SKILL:** jira-comment — dla Jiry i wątków GitLaba. Jira: wysyłka po „ok" wg jira-comment. GitLab: tekst do wklejenia per wątek; publikacja i resolve tylko na polecenie.

## Pokusy

| Myśl | Rzeczywistość |
|---|---|
| „Uwaga wskazuje jedną linię, poprawię jedną" | Recenzent komentuje pierwsze wystąpienie. Szukasz wzorca, podajesz liczbę. |
| „Outdated albo «poprawione» w wątku = naprawione" | Rozstrzyga kod. |
| „Poprawka oczywista, nie będę czekać" | Stop po propozycjach obowiązuje każdą pozycję. |
| „Odpiszę «Naprawione» i rozwiążę wątek" | Zero zapisów bez polecenia. |
| „Wypchnę, user i tak to zrobi" | Push robi user. Ty dajesz komendę. |
| „W konflikcie wezmę wersję z targetu" | Intencja obu stron, decyzja w raporcie. |
