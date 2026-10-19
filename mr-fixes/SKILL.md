---
name: mr-fixes
description: Użyj, gdy user wkleja link do merge requesta w GitLabie albo ticketu w Jirze i chce ogarnąć uwagi z code review lub QA — „ogarnij poprawki", „są uwagi w MR-ze", „popraw po CR", „QA coś zgłosił w tickecie" — albo wpisuje /mr-fixes.
argument-hint: "<link do MR-a w GitLabie albo ticketu w Jirze>"
---

# Poprawki do MR-a z uwag w GitLabie i Jirze

## Zasada

Ty analizujesz i proponujesz, user decyduje. Przebieg kończy się lokalnymi commitami i komendą `git push` do skopiowania. Na zewnątrz nie wychodzi nic.

Przez cały przebieg:

- **Zero zapisów w GitLabie i Jirze:** komentarz, odpowiedź, draft note, resolve wątku, zmiana statusu, reakcja. Także gdy uwaga jest „oczywiście naprawiona".
- **Zero `git push`**, w żadnej odmianie.
- **Zero edycji kodu przed akceptacją propozycji** (krok 4).
- **Bez `git stash`, `git rebase`, `git reset --hard`, `git merge --abort`**, chyba że user wprost każe.

## Krok 0 — Wejście

Argument: link do MR-a albo ticketu. Brak → poproś o link i stop.

- **Link do MR-a:** z URL-a bierzesz tylko `<ścieżka projektu>/-/merge_requests/<iid>`, host pomijasz. Pobierz MR — gałąź źródłowa, **gałąź docelowa**, tytuł, opis, `web_url`, `user_notes_count`. Klucz ticketu: z gałęzi, gdy jej brak — z tytułu, gdy brak — z opisu. Klucze innych ticketów (np. MR-a bazowego) nie wchodzą w zakres.
- **Link do ticketu:** znajdź otwarte MR-y z kluczem w gałęzi, tytule albo w linkach ticketu. Jeden → bierzesz. Kilka → lista i pytanie. Zero → zapytaj o gałąź.
- Druga strona nie istnieje (MR bez ticketu) → pracujesz na jednej, odnotowujesz w raporcie.

Gałąź docelową bierzesz zawsze z MR-a, nigdy z założenia „master".

## Krok 1 — Worktree i świeży target

1. **Worktree gałęzi MR-a:** `git worktree list`, szukasz gałęzi (główny checkout też jest na liście).
   - Jest → używasz go.
   - Gałąź lokalna istnieje, worktree nie → `git worktree add <ścieżka> <gałąź>`.
   - Nie ma nic → `git fetch origin <gałąź>`, potem `git worktree add --track -b <gałąź> <ścieżka> origin/<gałąź>`.
   - Ścieżka wg instrukcji projektu, domyślnie `<repo>/.claude/worktrees/<gałąź>`.
   - Brudny working tree → pokaż `git status`, zapytaj, stop.
2. `WT=$(git -C <worktree> rev-parse --show-toplevel)` — ścieżka bezwzględna. Każda komenda git jako `git -C "$WT" …`.
3. **Zgodność z origin:** `git -C "$WT" fetch origin <gałąź> <target>`.
   - Lokalna za origin → `git -C "$WT" merge --ff-only origin/<gałąź>`.
   - Lokalna i origin rozjechane w obie strony → pokaż oba zakresy commitów, zapytaj, stop.
4. **Merge targetu:** `git -C "$WT" merge origin/<target>`. Merge, nie rebase — rebase wymusza force push.
   - **Konflikty rozwiązujesz sam**, zachowując intencję obu stron. Każdy zapisujesz do raportu: plik, co zostało z której strony, dlaczego.
   - Pytasz tylko, gdy obie strony zmieniły tę samą logikę i z kodu ani historii nie da się ustalić, która intencja wygrywa. Merge zostaje w toku do odpowiedzi.
   - Po rozwiązaniu commit merge'a z domyślnym komunikatem.

## Krok 2 — Zebranie uwag

Pobierasz **wszystko**, ze wszystkich stron paginacji. Duży wynik (dziesiątki wątków) parsujesz skryptem, nie czytasz w całości.

**GitLab** — dyskusje MR-a (zawierają też pojedyncze notatki ogólne), `per_page=100`, kolejne strony aż strona wróci niepełna:
- kontrola kompletności: liczba notatek niesystemowych = `user_notes_count` MR-a;
- dyskusja systemowa = wszystkie jej notatki mają `system: true` → pomijasz. Notatka systemowa wewnątrz wątku („changed this line…") to poszlaka zmiany, nie powód pominięcia;
- wątek rozwiązany = każda notatka z `resolvable: true` ma `resolved: true` → pomijasz, liczysz do linii „Zebrane";
- bierzesz nierozwiązane wątki (przy kodzie i ogólne) oraz notatki ogólne;
- **autor nie decyduje** — notatka usera to często jego własne „do poprawy".

**Jira** — wszystkie komentarze ticketu, chronologicznie. Pobranych mniej niż `total` → dociągasz resztę (`startAt`):
- **granica:** najnowszy komentarz dowolnego autora, którego cała treść to „Zrobione" albo „Naprawione" (dowolna wielkość liter, najwyżej z jednym zdaniem typu „wejdzie z najbliższym wdrożeniem"). Wszystko do granicy włącznie pomijasz;
- komentarz z ustaleniami albo specyfikacją, bez prośby o zmianę → BEZ AKCJI, jedna pozycja;
- odpowiedź częściowa („1–2 OK, 3 naprawione") **nie jest granicą** — wymienione punkty traktujesz jako zgłoszone jako naprawione i weryfikujesz w kodzie;
- komentarz odsyła do załącznika, którego nie widzisz → status NIEJASNE, pytasz usera, co na nim jest.

Jedna notatka z kilkoma prośbami → osobne pozycje.

## Krok 3 — Analiza każdej pozycji

1. **Miejsce:** `plik:linia` w aktualnym kodzie. Pozycja wątku pochodzi z czasu komentarza: `git -C "$WT" show <position.head_sha>:<position.new_path>` pokazuje dokładnie skomentowany kod, potem znajdujesz go po treści w aktualnym.
2. **Czy już obsłużona:** `git -C "$WT" log --since=<data uwagi> -- <plik>` plus odczyt aktualnego kodu. Wątek „outdated" albo odpowiedź „poprawione" w GitLabie to poszlaka — rozstrzyga kod.
3. **Trafność:** sprawdzasz uwagę technicznie, nie przyjmujesz w ciemno. Niezgoda → argument z kodu (plik:linia, test, kontrakt) zamiast poprawki.
4. **Powtórzenia** — przy każdej uwadze o wzorcu (idiom, styl, brak walidacji, nazwa, błędne wywołanie, brakujący przypadek testowy). Recenzent zwykle komentuje pierwsze wystąpienie, a ten sam błąd siedzi w N miejscach:
   - szukasz wzorca w całym diffie MR-a (`git -C "$WT" diff origin/<target>...HEAD`) i w całym pakiecie albo serwisie, którego dotyczy uwaga;
   - podajesz liczbę i pełną listę `plik:linia`, osobno: w diffie MR-a / poza diffem;
   - propozycja obejmuje wystąpienia z diffu; poza diffem to osobna opcja do decyzji usera;
   - uwaga ewidentnie jednostkowa → piszesz, czego szukałeś i że powtórzeń jest 0.

Każda pozycja ma dokładnie jeden status:

| Status | Kiedy |
|---|---|
| DO POPRAWY | uwaga trafna, kod jej nie spełnia |
| JUŻ NAPRAWIONE | kod spełnia uwagę — z commitem |
| NIE ZGADZAM SIĘ | uwaga błędna — z argumentem |
| NIEJASNE | brakuje informacji — z pytaniem do usera |
| BEZ AKCJI | to nie prośba: informacja, odpowiedziane pytanie, status |

## Krok 4 — Propozycje i stop

Dwie linie na start:

- **Target:** co zmergowane (liczba commitów), konflikty i jak rozwiązane.
- **Zebrane:** GitLab — N wątków do analizy, M rozwiązanych pominiętych; Jira — K komentarzy po granicy albo „brak granicy".

Potem pozycje, numerowane w kolejności źródeł: GitLab, potem Jira, chronologicznie.

DO POPRAWY, NIE ZGADZAM SIĘ i NIEJASNE w pełnym formacie; link do wątku to `<web_url>#note_<id pierwszej notatki>`:

```
### 3. GitLab · Autor · src/orders/sync.ts:42 · <link do wątku>
> cytat uwagi, najwyżej 2 linie
Status: DO POPRAWY
Miejsca (4): w diffie — sync.ts:42, sync.ts:88, mapper.ts:13 · poza diffem — legacy.ts:7
Propozycja: co i jak, 1–3 linie; snippet, gdy słowa są niejednoznaczne
Skala: mała
```

JUŻ NAPRAWIONE i BEZ AKCJI po jednej linii: numer, źródło, skrót uwagi, commit albo powód.

Na końcu numery pogrupowane wg statusu i pytania z pozycji NIEJASNE.

Zero pozycji DO POPRAWY i NIEJASNE → bez stopu, przechodzisz do kroku 6 (merge targetu też wymaga bramek), potem raport.

**Stop. Czekasz na odpowiedź usera** i czytasz ją dosłownie:
- „ok" albo „tak" bez numerów → wszystkie DO POPRAWY w wersji z propozycji;
- numery z „tak" / „nie" / „inaczej: …" → tylko wymienione, niewymienionych nie ruszasz;
- miejsca poza diffem poprawiasz tylko po wprost wyrażonej zgodzie.

## Krok 5 — Poprawki

- Kolejność wg numerów. **Jeden commit na pozycję**, ze wszystkimi jej miejscami. Komunikat wg konwencji repo (np. prefiks klucza ticketu).
- **Mała** (kilka plików, bez przebudowy testów) → robisz sam, w sesji.
- **Duża** (wiele plików albo spora zmiana testów) → jeden subagent naraz, w tym samym worktree. Prompt: `$WT`, cytat uwagi, zaakceptowana propozycja, pełna lista miejsc, reguły commitów usera, zakaz push i stash. Model średni do poprawek, najmniejszy do mechanicznych zmian nazw.
- Propozycja w trakcie przestaje pasować (kolejne miejsce, inny skutek, potrzebna zmiana kontraktu) → wstrzymujesz pozycję i wracasz do usera z nową propozycją.

## Krok 6 — Bramki

Raz, po wszystkich poprawkach — także gdy poprawek nie było, a merge targetu coś wniósł. Nic nie wniósł i poprawek brak → pomijasz. Lint/format, typy i testy, komendy wg instrukcji projektu. Tylko pakiety z plikami w `git -C "$WT" diff --name-only origin/<target>...HEAD`.

- Fail przez poprawkę → naprawiasz i dokładasz commit.
- Fail niezależny od MR-a (przyszedł z targetem) → odnotowujesz w raporcie, nie szukasz przyczyny.

## Krok 7 — Raport

```
## !<iid> <gałąź> → <target>
Target: zmergowany origin/<target> (12 commitów) · konflikty: 2 — a.ts (…), b.ts (…)

| # | Źródło | Uwaga | Status | Commit |
|---|---|---|---|---|

Bramki: lint ✅ · typy ✅ · testy <pakiet> ✅ 214/214
Do wypchnięcia: 5 commitów
```

Na samym końcu osobny blok z bezwzględną ścieżką worktree — działa z dowolnego katalogu:

```bash
git -C /bezwzględna/ścieżka/worktree push origin <gałąź>
```

`git -C "$WT" rev-list --count origin/<gałąź>..HEAD` = 0 → zamiast komendy jedna linia „Nic do wypchnięcia". Na tym przebieg się kończy.

## Odpowiedzi na uwagi

Tylko na prośbę usera („odpisz na 3", „daj odpowiedzi"). **REQUIRED SUB-SKILL:** jira-comment — dla komentarzy w Jirze i dla wątków GitLaba.

- Jira: obowiązuje gate jira-comment — wysyłka dopiero po jawnym „ok".
- GitLab: tekst do wklejenia, osobno per wątek. Publikacja i resolve wątku tylko na wprost wyrażone polecenie.

## Racjonalizacje

| Myśl | Rzeczywistość |
|---|---|
| „Uwaga wskazuje jedną linię, poprawię jedną linię" | Recenzent komentuje pierwsze wystąpienie. Szukasz wzorca i podajesz liczbę. |
| „Wątek jest outdated, czyli naprawione" | Outdated = linia się zmieniła. Rozstrzyga kod. |
| „Ktoś odpisał «poprawione» w GitLabie, pomijam" | Granica „Zrobione" działa tylko w Jirze. W GitLabie sprawdzasz kod. |
| „Notatka jest od usera, to nie uwaga do mnie" | User sam zostawia sobie „do poprawy". Bierzesz ją jak każdą inną. |
| „Poprawka oczywista, nie będę czekać" | Stop po propozycjach obowiązuje każdą pozycję. |
| „Odpiszę «Naprawione» i rozwiążę wątek, oszczędzę klikania" | Zero zapisów w GitLabie i Jirze bez polecenia. |
| „Wypchnę, user i tak to zrobi" | Push robi user. Ty pokazujesz komendę. |
| „Rebase da czystszą historię" | Rebase = force push. Merge. |
| „W konflikcie wezmę wersję z targetu" | Zachowujesz intencję obu stron i zapisujesz decyzję w raporcie. |

## Czerwone flagi — zatrzymaj się

- Edytujesz plik, a user jeszcze nie odpowiedział na propozycje.
- Propozycja mówi „i podobne miejsca" zamiast listy `plik:linia`.
- Wywołujesz narzędzie piszące do GitLaba albo Jiry.
- Komenda push ze ścieżką względną albo bez `-C`.
- W historii poleceń jest `git push`, `git rebase`, `git stash` albo `git reset --hard`.
