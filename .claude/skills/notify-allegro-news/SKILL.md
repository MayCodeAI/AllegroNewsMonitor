---
name: notify-allegro-news
description: Wysyła na Slacka powiadomienie o pojedynczym newsie z bloga Allegro Developers, który został uznany za istotny dla projektu. Używany przez agenta głównego po otrzymaniu wyniku analizy od subagenta. Kanał docelowy pobierany jest ze zmiennej środowiskowej `SLACK_CHANNEL_ID`.
allowed-tools: mcp__slack__slack_post_message
---

# notify-allegro-news

Wysyła jedno powiadomienie na Slacka o newsie wymagającym reakcji. Skill używa wyłącznie `mcp__slack__slack_post_message`.

Uwaga: MCP `@modelcontextprotocol/server-slack` nie wspiera parametru `blocks` — formatowanie realizowane jest przez Slack **mrkdwn** w polu `text` (`*bold*`, `_italic_`, `<url|label>`, `•` listy, `\n` nowe linie). Dzięki temu zachowujemy spójny format wiadomości bez zmiany serwera MCP.

## Wejście

Wywołujący (agent główny) musi dostarczyć następujące dane:

- `news_url` (wymagane, tekst) — URL wpisu na blogu Allegro.
- `news_title` (wymagane, tekst) — tytuł wpisu.
- `analysis_summary` (wymagane, tekst) — krótkie uzasadnienie dlaczego news wymaga reakcji.
- `affected_areas` (opcjonalne, lista tekstów) — obszary projektu dotknięte zmianą (spójne z sekcją *Zakres monitorowania* w `CLAUDE.md`).
- `news_published_at` (opcjonalne, data ISO 8601 lub `YYYY-MM-DD`) — data publikacji wpisu.

Skill **nie** przyjmuje `channel_id` jako parametru — zawsze używa `SLACK_CHANNEL_ID` ze środowiska.

## Procedura

1. **Zwaliduj wymagane pola.** Jeśli brakuje `news_url`, `news_title` lub `analysis_summary` — zakończ z błędem i nie wysyłaj wiadomości.
2. **Odczytaj `SLACK_CHANNEL_ID`** ze środowiska. Jeśli zmienna jest pusta — zakończ z błędem (brak konfiguracji, patrz `CLAUDE.md` sekcja *Sekrety*).
3. **Zbuduj `text`** zgodnie z szablonem z sekcji *Szablon wiadomości*. Pola opcjonalne, które nie zostały przekazane, pomiń razem z ich linijką (nie wstawiaj pustych "—").
4. **Escape mrkdwn-specjalnych znaków w polach dostarczonych przez wywołującego** (`news_title`, `analysis_summary`, `affected_areas`): zamień `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`. Reszty (`*`, `_`, `` ` ``) nie ruszaj — jeśli wystąpi w treści, niech Slack zrenderuje ją jak potrafi; to świadomy tradeoff zamiast agresywnego escape'owania.
5. **Wywołaj** `mcp__slack__slack_post_message` z `channel_id` = `$SLACK_CHANNEL_ID` i przygotowanym `text`.
6. **Przekaż `ts` i `channel`** ze zwróconego obiektu jako wynik — agent główny może je zalogować / użyć do wątku.

## Szablon wiadomości

```
:rotating_light: *Nowa aktualność Allegro wymaga reakcji*

*<{news_url}|{news_title}>*

*Dlaczego istotne:*
{analysis_summary}

*Obszary:* {affected_areas_joined}
*Opublikowano:* {news_published_at}
```

- `{affected_areas_joined}` — elementy połączone `, ` (np. `REST API, OAuth`). Jeśli brak — pomiń całą linijkę `*Obszary:* ...`.
- `{news_published_at}` — jeśli brak, pomiń całą linijkę `*Opublikowano:* ...`.
- Między sekcjami jedna pusta linia (`\n\n`) — Slack renderuje to jako oddzielne akapity.
- Emoji `:rotating_light:` Slack zrenderuje jako 🚨.

## Przykład

Dla wejścia:
- `news_url`: `https://developer.allegro.pl/news/new-offer-endpoint`
- `news_title`: `Zmiana w endpoint 'offers'`
- `analysis_summary`: `Zmieniono format pola price — wymaga aktualizacji parsera.`
- `affected_areas`: `["REST API", "Oferty"]`
- `news_published_at`: `2026-04-15`

Wiadomość (`text`):

```
:rotating_light: *Nowa aktualność Allegro wymaga reakcji*

*<https://developer.allegro.pl/news/new-offer-endpoint|Zmiana w endpoint 'offers'>*

*Dlaczego istotne:*
Zmieniono format pola price — wymaga aktualizacji parsera.

*Obszary:* REST API, Oferty
*Opublikowano:* 2026-04-15
```

## Uwagi

- Skill wysyła **jedną** wiadomość na wywołanie. Dla wielu newsów wywołaj go wielokrotnie.
- Skill nie czyta ani nie zapisuje bazy danych — decyzja "czy wysłać" należy do agenta głównego (na podstawie `requires_code_changes` zwróconego przez subagenta).
- Zgodnie z `CLAUDE.md` agent główny komunikuje się ze Slackiem **wyłącznie przez MCP** — nie buduj bezpośrednich wywołań Web API.
- Jeśli w przyszłości przesiądziemy się na MCP wspierający Slack Blocks, szablon wiadomości z tego skilla jest punktem wyjścia do zmapowania na strukturę blocks.
