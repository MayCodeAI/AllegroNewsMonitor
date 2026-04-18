---
name: init-news-history-table
description: Sprawdza, czy w Supabase istnieje tabela historii przeanalizowanych newsów Allegro (`public.analyzed_news`). Jeżeli nie istnieje — tworzy ją przy pomocy migracji. Uruchamiaj na początku pracy subagenta analizującego newsy, zanim zacznie czytać lub zapisywać historię.
allowed-tools: mcp__supabase__list_tables, mcp__supabase__apply_migration
---

# init-news-history-table

Inicjalizuje tabelę `public.analyzed_news` w Supabase. Tabela przechowuje historię newsów z bloga Allegro Developers przeanalizowanych przez subagenta (klucz deduplikacji + wynik analizy).

## Procedura

1. **Sprawdź, czy tabela istnieje.** Wywołaj `mcp__supabase__list_tables` dla schematu `public`.
2. **Jeżeli w zwróconej liście jest `analyzed_news`** — zakończ pracę. Nie wykonuj migracji. Zgłoś: „Tabela `public.analyzed_news` już istnieje, nic nie zmieniam."
3. **Jeżeli tabeli nie ma** — wywołaj `mcp__supabase__apply_migration` z parametrami:
   - `name`: `create_analyzed_news`
   - `query`: SQL z sekcji *SQL migracji* poniżej (skopiuj dokładnie).
4. Po wykonaniu migracji zgłoś: „Utworzono tabelę `public.analyzed_news` migracją `create_analyzed_news`."

## SQL migracji

```sql
CREATE TABLE IF NOT EXISTS public.analyzed_news (
    id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    news_url              text NOT NULL UNIQUE,
    news_title            text NOT NULL,
    news_published_at     timestamptz,
    analyzed_at           timestamptz NOT NULL DEFAULT now(),
    requires_code_changes boolean NOT NULL,
    analysis_summary      text,
    affected_areas        text[]
);

CREATE INDEX IF NOT EXISTS idx_analyzed_news_analyzed_at
    ON public.analyzed_news (analyzed_at DESC);
```

## Opis kolumn

- `id` — klucz główny (uuid, generowany automatycznie).
- `news_url` — URL wpisu na blogu Allegro; klucz deduplikacji (UNIQUE).
- `news_title` — tytuł wpisu, dla czytelności logów.
- `news_published_at` — data publikacji wpisu przez Allegro (może być `NULL`, jeśli niedostępna).
- `analyzed_at` — moment analizy przez subagenta (domyślnie `now()`).
- `requires_code_changes` — wynik analizy: `true` jeśli wpis wymaga zmian w kodzie projektu.
- `analysis_summary` — krótkie uzasadnienie decyzji subagenta.
- `affected_areas` — lista obszarów projektu, których wpis dotyczy (spójna z sekcją *Zakres monitorowania* w `CLAUDE.md`).

## Uwagi

- Skill jest idempotentny — bezpiecznie wywoływać go przy każdym starcie subagenta.
- Skill **nie** modyfikuje istniejącej tabeli. Jeśli schemat trzeba zmienić, stwórz nową migrację ręcznie.
- Skill korzysta wyłącznie z MCP Supabase — żadnych bezpośrednich wywołań API ani SQL przez inny kanał.
