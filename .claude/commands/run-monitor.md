---
description: Uruchamia monitoring newsów Allegro — woła subagenta analizującego, a dla istotnych wpisów wysyła powiadomienie na Slacka.
allowed-tools: Agent, Skill
---

# /run-monitor

Jesteś agentem głównym monitoringu newsów Allegro Developers. Twoje zadanie to zebrać nowe aktualności od subagenta i powiadomić Slacka o tych istotnych. Bazy danych **nie** dotykasz bezpośrednio — to domena subagenta.

## Procedura

1. **Deleguj analizę** do subagenta `allegro-news-analyzer` przez narzędzie `Agent` (`subagent_type: "allegro-news-analyzer"`). Prompt: "Przeanalizuj wszystkie nowe newsy na pierwszej stronie https://developer.allegro.pl/news/ zgodnie z procedurą w swojej definicji i zwróć listę JSON-podobnych obiektów." Nie powielaj logiki subagenta — on wie co robi.

2. **Odbierz wynik** — lista obiektów z polami `news_url`, `news_title`, `news_published_at`, `summary`, `requires_code_changes`, `affected_areas`, `analysis_note`. Pusta lista = brak nowych wpisów.

3. **Dla każdego wpisu z `requires_code_changes = true`** wywołaj skill `notify-allegro-news`, przekazując:
   - `news_url`
   - `news_title`
   - `analysis_summary` — połącz `summary` + `analysis_note` w jeden spójny tekst (summary na początku, uzasadnienie na końcu).
   - `affected_areas` — z wyniku subagenta.
   - `news_published_at` — jeśli dostępne.

   Wpisy z `requires_code_changes = false` **pomijasz** — nie wysyłasz na Slacka.

4. **Raport końcowy** do użytkownika — jedno-dwa zdania:
   - ile newsów przeanalizowano (nowych), ile wymaga reakcji i zostało wysłanych na Slacka, ile pominięto jako nieistotne.
   - jeśli subagent zwrócił pustą listę — krótko "brak nowych newsów od ostatniego uruchomienia".

## Reguły

- Jeśli subagent lub skill zwróci błąd — przerwij, zgłoś błąd użytkownikowi. Nie retryuj automatycznie, nie maskuj.
- Nie wysyłaj tej samej wiadomości dwa razy. Subagent już zadbał o deduplikację w bazie; ty tylko iterujesz po tym, co zwrócił.
- Nie wymyślaj własnego formatu wiadomości Slack — zawsze przez skill `notify-allegro-news`, który trzyma spójny format.
- Nie dodawaj komentarzy na Slacku poza powiadomieniami o istotnych newsach (żadnego "uruchomiono monitoring o 14:00").
