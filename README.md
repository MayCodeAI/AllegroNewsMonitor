# Allegro News Monitor

Opis projektu i architektura: patrz [`CLAUDE.md`](./CLAUDE.md).

## Konfiguracja Slack MCP — krok po kroku

### 1. Otwórz panel aplikacji Slack

Wejdź na **<https://api.slack.com/apps>**. Zaloguj się tym samym kontem Slack, które jest w workspace'ie, gdzie ma działać bot.

### 2. Utwórz nową aplikację

- Kliknij zielony przycisk **Create New App** (prawy górny róg).
- W oknie, które się otworzy, kliknij **From an app manifest** (nie „From scratch").
- Wybierz workspace z listy → **Next**.
- Pojawi się edytor manifestu. Przełącz zakładkę na **YAML** i **wklej** ten manifest w całości (zastąp wszystko, co tam jest domyślnie):

  ```yaml
  display_information:
    name: Allegro News Monitor
    description: Powiadomienia o zmianach w Allegro API wymagających reakcji w kodzie.
  features:
    bot_user:
      display_name: Allegro News Monitor
      always_online: true
  oauth_config:
    scopes:
      bot:
        - chat:write
        - channels:read
        - channels:history
        - users:read
  settings:
    org_deploy_enabled: false
    socket_mode_enabled: false
    token_rotation_enabled: false
  ```

- **Next** → **Create**. Aplikacja jest utworzona, ale jeszcze nie zainstalowana w workspace'ie.

### 3. Zainstaluj aplikację w workspace'ie

- Na lewym pasku (w panelu aplikacji) kliknij **Install App**.
- Kliknij przycisk **Install to Workspace**.
- Slack pokaże ekran zgody („Allegro News Monitor is requesting permission to access…") → kliknij **Allow**.

### 4. Skopiuj Bot User OAuth Token

- Po instalacji zostajesz przeniesiony na stronę **OAuth & Permissions** (albo kliknij tę pozycję w lewym pasku).
- U góry, w sekcji **OAuth Tokens for Your Workspace**, zobaczysz pole **Bot User OAuth Token** — zaczyna się od `xoxb-`.
- Kliknij **Copy**. To jest wartość, którą wpisujesz do `.env` jako `SLACK_BOT_TOKEN=xoxb-…`.

### 5. Znajdź Team ID

Najprościej przez przeglądarkę:

- Otwórz Slacka w przeglądarce: **<https://app.slack.com>** (nie w aplikacji desktopowej).
- Zaloguj się na właściwy workspace. Jak klikniesz w dowolny kanał, URL w pasku adresu wygląda tak:

  ```
  https://app.slack.com/client/T01AB2CD3EF/C09XY8ZAB12
  ```

- Fragment zaczynający się od `T` (do pierwszego `/`) to Team ID, w przykładzie `T01AB2CD3EF`. To `SLACK_TEAM_ID`.

### 6. Zaproś bota na kanał powiadomień

- Otwórz w Slacku kanał, na który mają przychodzić powiadomienia (albo utwórz nowy, np. `#allegro-news-monitor`).
- Wpisz w polu wiadomości i wyślij:

  ```
  /invite @Allegro News Monitor
  ```

- Slack potwierdzi: „added Allegro News Monitor to #…". Bez tego bot nie opublikuje wiadomości na tym kanale.

### 7. Zapisz ID kanału

Potrzebne będzie przy konfiguracji agenta głównego (w `CLAUDE.md` jest to osobna pozycja w checkliście):

- Kliknij **nazwę kanału na górze** okna Slacka.
- W oknie **About** przewiń na sam dół — jest tam **Channel ID** (zaczyna się od `C...`). Kliknij **Copy**.

Alternatywnie: w URL-u web-clienta to fragment po Team ID (w przykładzie powyżej `C09XY8ZAB12`).

### 8. Uzupełnij `.env` i zrestartuj Claude Code

W katalogu projektu:

```bash
cp .env.example .env    # tylko jeśli nie masz jeszcze .env
# edytuj .env — wklej SLACK_BOT_TOKEN i SLACK_TEAM_ID
set -a; source .env; set +a; claude
```

### 9. Sprawdź, że MCP wstał

W Claude Code wpisz **`/mcp`**. Na liście powinien być `slack` obok `supabase`, oba *connected*. Jeśli jest *failed* — pokaż output, żeby zdiagnozować przyczynę.
