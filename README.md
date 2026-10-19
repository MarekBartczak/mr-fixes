# mr-fixes

Skill dla Claude Code, który ogarnia poprawki do merge requesta: zbiera uwagi z wątków w GitLabie i z komentarzy w Jirze, analizuje je, proponuje poprawki, a po Twojej akceptacji poprawia i commituje lokalnie. Niczego nie wypycha i nigdzie nie odpisuje.

## TL;DR

1. Skopiuj katalog `mr-fixes/` do `~/.claude/skills/`.
2. Wpisz `/mr-fixes <link do MR-a albo ticketu>`.
3. Dostajesz listę uwag ze statusem i propozycją poprawki. Odpowiadasz np. „1, 3 tak, 2 nie, 4 inaczej: …".
4. Na końcu dostajesz raport i gotową komendę `git -C /bezwzględna/ścieżka push origin <gałąź>` do skopiowania.

## Problem, który rozwiązuje

Uwagi do MR-a przychodzą z dwóch miejsc: z wątków code review w GitLabie i z komentarzy QA albo PM-a w tickecie w Jirze. Część jest już naprawiona, część to dyskusja, część to Twoje własne notatki „do poprawy". Model zostawiony sam sobie:

- poprawia jedną wskazaną linię, choć ten sam błąd siedzi w pięciu miejscach, a recenzent skomentował tylko pierwsze;
- uznaje wątek „outdated" za naprawiony;
- poprawia, zanim zobaczysz propozycję;
- sam odpisuje „Naprawione", rozwiązuje wątki i wypycha gałąź.

Skill ustala stały przebieg z jednym punktem decyzji (Twoja akceptacja) i bez żadnych zapisów na zewnątrz.

## Przebieg

| Krok | Co się dzieje |
|---|---|
| Wejście | Link do MR-a albo ticketu; skill dociąga drugą stronę (z MR-a ticket po kluczu, z ticketu jego MR) |
| Worktree | Znajduje albo zakłada worktree gałęzi MR-a, sprawdza zgodność z origin |
| Target | Merge świeżej gałęzi docelowej MR-a (master, release — wg MR-a); konflikty rozwiązuje sam i opisuje w raporcie |
| Uwagi | GitLab: wszystkie nierozwiązane wątki i notatki ogólne, także Twoje. Jira: wszystkie komentarze po ostatnim „Zrobione" / „Naprawione" |
| Analiza | Miejsce w kodzie, czy już naprawione (commity + kod), czy uwaga trafna, **powtórzenia tego samego wzorca w diffie MR-a i w serwisie** |
| Propozycje | Lista ze statusami: do poprawy / już naprawione / nie zgadzam się / niejasne / bez akcji — i stop |
| Poprawki | Po akceptacji: jeden commit na uwagę; duże poprawki robi pojedynczy subagent w tym samym worktree |
| Bramki | Lint, typy, testy tylko zmienionych pakietów |
| Raport | Tabela uwag z commitami, wyniki bramek, komenda push z bezwzględną ścieżką |

Odpowiedzi na uwagi skill pisze tylko na prośbę — przez [jira-comment](https://github.com/MarekBartczak/jira-comment), w tym samym stylu dla Jiry i GitLaba.

## Wymagania

- `git` z obsługą `worktree`.
- Dostęp do GitLaba z Claude Code — MCP GitLaba albo `glab`.
- Dostęp do Jiry — MCP Atlassian (opcjonalnie; bez niego skill pracuje tylko na GitLabie).
- Skill [jira-comment](https://github.com/MarekBartczak/jira-comment) — tylko do pisania odpowiedzi.

## Instalacja

Skill to katalog `mr-fixes/` z jednym plikiem `SKILL.md`.

Linux / macOS:

```bash
git clone https://github.com/MarekBartczak/mr-fixes.git
mkdir -p ~/.claude/skills
cp -r mr-fixes/mr-fixes ~/.claude/skills/
```

Windows (PowerShell):

```powershell
git clone https://github.com/MarekBartczak/mr-fixes.git
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse mr-fixes\mr-fixes "$env:USERPROFILE\.claude\skills\"
```

Windows z WSL: Claude Code uruchomiony w WSL czyta `~/.claude/skills` z systemu plików WSL.

Tylko jeden projekt zamiast globalnie: to samo do `<repo>/.claude/skills/mr-fixes/`.

## Użycie

```
/mr-fixes https://gitlab.com/grupa/projekt/-/merge_requests/123
/mr-fixes https://firma.atlassian.net/browse/PROJ-456
```

Albo naturalnie: „ogarnij poprawki do tego MR-a", „QA coś zgłosił w PROJ-456".

## Dostosowanie

`SKILL.md` jest po polsku. Pod swój zespół zmień:

- **ścieżkę worktree** (krok 1) — domyślnie `<repo>/.claude/worktrees/<gałąź>`, chyba że instrukcje projektu mówią inaczej;
- **słowa-granice w Jirze** (krok 2) — domyślnie „Zrobione" i „Naprawione";
- **bramki** (krok 6) — skill bierze komendy z instrukcji projektu (`CLAUDE.md`, `AGENTS.md`); jeśli ich tam nie ma, dopisz je.

## Licencja

MIT — patrz `LICENSE`.
