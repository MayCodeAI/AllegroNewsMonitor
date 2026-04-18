# Allegro News Monitor

## Opis projektu

Projekt jest Agentem, którego zadaniem jest monitorowanie aktualności publikowanych na blogu **Allegro Developers** (https://developer.allegro.pl/news/). Agent analizuje każdą nową aktualność pod kątem tego, czy wprowadzane przez Allegro zmiany mogą wymagać modyfikacji w kodzie projektu korzystającego z Allegro API.

Jeżeli Agent uzna, że dana aktualność wymaga reakcji (np. zmian w kodzie, aktualizacji integracji, dostosowania do nowych wymagań), wysyła powiadomienie na dedykowany kanał Slack.

## Architektura

### Agent główny

Agent główny jest uruchamiany cyklicznie (harmonogram zostanie ustalony później). Jego odpowiedzialności:

1. Delegowanie zadania pobrania i analizy aktualności do **subagenta**.
2. Obsługa wyniku pracy subagenta — w szczególności decyzja, które aktualności wymagają powiadomienia.
3. Wysyłanie powiadomień na dedykowany kanał **Slack** dla aktualności uznanych za istotne.
4. Komunikacja ze **Slackiem** odbywa się **wyłącznie poprzez MCP** (Model Context Protocol).

### Subagent

Subagent odpowiada wyłącznie za pobranie i analizę aktualności:

1. Pobiera listę ostatnich aktualności z bloga Allegro Developers.
2. Sprawdza w bazie danych, które aktualności zostały już wcześniej przeanalizowane — pomija je.
3. Dla każdej nowej aktualności wykonuje analizę pod kątem listy technologii i funkcji wykorzystywanych w projekcie (sekcja *Zakres monitorowania* poniżej).
4. Zapisuje w bazie danych informację o tym, że aktualność została przeczytana i przeanalizowana (wraz z wynikiem analizy).
5. Zwraca wynik analizy do agenta głównego. Subagent **nie komunikuje się ze Slackiem**.
6. Komunikacja z **bazą danych** odbywa się **wyłącznie poprzez MCP**.

### Zasady komunikacji

- **Agent główny** komunikuje się ze **Slackiem** — wyłącznie przez MCP.
- **Subagent** komunikuje się z **bazą danych** — wyłącznie przez MCP.
- Żadne bezpośrednie wywołania API bazy danych ani Slack API poza MCP nie są dozwolone.
- Subagent nie ma dostępu do Slacka; agent główny nie komunikuje się bezpośrednio z bazą danych.

## Zakres monitorowania

Poniższa lista opisuje funkcjonalności i zasoby Allegro API, z których korzysta projekt. Subagent powinien uznawać aktualność za istotną, jeżeli dotyczy ona któregokolwiek z poniższych obszarów.

1. **Pobieranie ofert, zamówień, billing entries** — wszystko co dotyczy listowania/odczytu ofert, zamówień (checkout-forms, orders) oraz wpisów rozliczeniowych (billing entries).
2. **Autoryzacja konta** — OAuth, tokeny, device flow, scope'y, logowanie aplikacyjne.
3. **Wiadomości i dyskusje** — messaging między sprzedającym a kupującym, dyskusje przy ofertach, threads.

## Sekrety

Do działania MCP wymagane są następujące zmienne środowiskowe (nie commitowane do repo — patrz `.env.example`):

- `SUPABASE_PROJECT_REF` — identyfikator projektu Supabase.
- `SUPABASE_ACCESS_TOKEN` — personal access token z dashboardu Supabase.
- `SLACK_BOT_TOKEN` — token bota Slack (`xoxb-...`) z zainstalowanej aplikacji Slack.
- `SLACK_TEAM_ID` — ID workspace'u Slack (`T...`).
- `SLACK_CHANNEL_ID` — ID kanału Slack, na który wysyłane są powiadomienia o istotnych newsach.

Ładowanie przed startem Claude Code:

```bash
set -a; source .env; set +a; claude
```

