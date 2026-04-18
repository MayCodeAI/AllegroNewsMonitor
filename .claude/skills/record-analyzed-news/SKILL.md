---
name: record-analyzed-news
description: Zapisuje w tabeli `public.analyzed_news` wynik analizy pojedynczego newsa z bloga Allegro Developers. Używany przez subagenta po zakończeniu analizy nowej aktualności. Deduplikacja po `news_url` (UNIQUE) — ewentualny konflikt sygnalizuje błąd logiki, nie jest maskowany.
allowed-tools: mcp__supabase__execute_sql
---

# record-analyzed-news

Wstawia jeden wiersz do tabeli `public.analyzed_news` z wynikiem analizy jednego newsa.
Skill zakłada, że tabela już istnieje — jeśli subagent nie ma pewności, powinien najpierw wywołać skill `init-news-history-table`.

## Wejście

Wywołujący (subagent) musi dostarczyć następujące dane analizy:

- `news_url` (wymagane, tekst) — URL wpisu; klucz deduplikacji (UNIQUE).
- `news_title` (wymagane, tekst) — tytuł wpisu.
- `news_published_at` (opcjonalne, ISO 8601 lub `NULL`) — data publikacji wpisu na blogu Allegro.
- `requires_code_changes` (wymagane, boolean) — wynik analizy.
- `analysis_summary` (opcjonalne, tekst lub `NULL`) — krótkie uzasadnienie decyzji.
- `affected_areas` (opcjonalne, lista tekstów lub `NULL`) — obszary projektu, których wpis dotyczy (spójne z sekcją *Zakres monitorowania* w `CLAUDE.md`).

## Procedura

1. **Zwaliduj wymagane pola.** Jeśli brakuje `news_url`, `news_title` lub `requires_code_changes` — zakończ z błędem i nie wykonuj INSERT.
2. **Zbuduj SQL INSERT** wg szablonu z sekcji *SQL insertu*, stosując reguły z sekcji *Cytowanie wartości*.
3. **Wykonaj** `mcp__supabase__execute_sql` z przygotowanym zapytaniem.
4. **Odczytaj zwrócony wiersz** (`RETURNING id, analyzed_at`) i przekaż go jako wynik do agenta głównego.
5. **Błąd UNIQUE na `news_url`** — nie retryuj i nie maskuj. Zgłoś go jako błąd logiki: subagent powinien był pominąć ten news już na etapie deduplikacji (patrz pkt 2 procedury subagenta w `CLAUDE.md`).

## SQL insertu

Szablon — podstaw wartości zgodnie z regułami cytowania:

```sql
INSERT INTO public.analyzed_news (
    news_url,
    news_title,
    news_published_at,
    requires_code_changes,
    analysis_summary,
    affected_areas
) VALUES (
    <news_url>,
    <news_title>,
    <news_published_at>,
    <requires_code_changes>,
    <analysis_summary>,
    <affected_areas>
)
RETURNING id, analyzed_at;
```

## Cytowanie wartości

Wartości tekstowe z bloga Allegro mogą zawierać apostrofy, cudzysłowy i znaki specjalne. Używaj PostgreSQL dollar-quoting, żeby nie escape'ować ręcznie:

- **Tekst (URL, tytuł, podsumowanie):** owiń w `$nws$...$nws$`. Jeśli w treści wystąpi dokładnie fragment `$nws$`, wybierz inny tag (np. `$nws2$`).
- **`NULL`:** wpisz literalnie `NULL` (bez cudzysłowów).
- **`news_published_at`:** jeśli znane — `$nws$2026-04-15T10:00:00Z$nws$::timestamptz`; jeśli nieznane — `NULL`.
- **`requires_code_changes`:** `true` lub `false` (bez cudzysłowów).
- **`affected_areas`:** tablica tekstów, np. `ARRAY[$nws$REST API$nws$, $nws$OAuth$nws$]::text[]`; jeśli brak obszarów — `NULL`.

### Przykład

Dla newsa:
- URL: `https://developer.allegro.pl/news/new-endpoint`
- Tytuł: `Zmiana w endpoint 'offers'`
- Publikacja: `2026-04-15T10:00:00Z`
- Wymaga zmian w kodzie: tak
- Uzasadnienie: `Zmieniono format odpowiedzi.`
- Obszary: `REST API`, `Oferty`

Zapytanie:

```sql
INSERT INTO public.analyzed_news (
    news_url, news_title, news_published_at,
    requires_code_changes, analysis_summary, affected_areas
) VALUES (
    $nws$https://developer.allegro.pl/news/new-endpoint$nws$,
    $nws$Zmiana w endpoint 'offers'$nws$,
    $nws$2026-04-15T10:00:00Z$nws$::timestamptz,
    true,
    $nws$Zmieniono format odpowiedzi.$nws$,
    ARRAY[$nws$REST API$nws$, $nws$Oferty$nws$]::text[]
)
RETURNING id, analyzed_at;
```

## Uwagi

- Skill używa wyłącznie `mcp__supabase__execute_sql` (DML). Operacje DDL należą do migracji / skilla `init-news-history-table`.
- Kolumny `id` i `analyzed_at` wypełnia baza (`gen_random_uuid()`, `now()`) — nie podawaj ich w INSERT.
- Skill wstawia **jeden** wiersz na wywołanie. Dla wielu newsów wywołaj go wielokrotnie — upraszcza to obsługę błędów pojedynczego wpisu.
- Zgodnie z `CLAUDE.md` subagent komunikuje się z bazą **wyłącznie przez MCP** — nie buduj bezpośrednich połączeń do Postgresa.
