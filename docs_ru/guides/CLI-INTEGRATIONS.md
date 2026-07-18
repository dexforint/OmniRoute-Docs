---
title: "CLI-интеграции — направьте любой CLI для кодирования на OmniRoute"
version: 3.8.40
lastUpdated: 2026-06-28
---

# CLI-интеграции

OmniRoute поставляет семейство команд `setup-*`, которые настраивают CLI
для кодирования (Codex, Claude Code, OpenCode, Cline, …) на использование OmniRoute в качестве бэкенда — так
что инструмент обращается к **одному** эндпоинту, а OmniRoute маршрутизирует на нужного провайдера с
автоматическим переключением. Каждая команда читает **живой** каталог моделей из запущенного
OmniRoute (локального или удалённого) и записывает собственный конфиг инструмента на **вашей**
машине. API-ключ указывается через переменную окружения везде, где инструмент это поддерживает, поэтому
секрет никогда не записывается на диск (исключения отмечены ниже).

Есть также два лаунчера — `omniroute launch` (Claude Code) и
`omniroute launch-codex` (Codex) — которые запускают CLI с нужными переменными окружения,
вообще не записывая никаких конфигов.

Разовую базовую настройку вручную для двух самых богатых интеграций см. в углублённых
руководствах по инструментам:

- [Настройка Claude Code](./CLAUDE-CODE-CONFIGURATION.md)
- [Настройка Codex CLI](./CODEX-CLI-CONFIGURATION.md)
- [Удалённый режим](./REMOTE-MODE.md) — управление удалённым OmniRoute (VPS / Tailnet) с вашего ноутбука

---

## Сводная таблица

Каждая команда учитывает **активный контекст** (задаётся через `omniroute connect`, см.
[Удалённый режим](./REMOTE-MODE.md)) или явные флаги `--remote <url> --api-key <key>`.
«Локально или удалённо» ниже означает: без флагов цель — `http://localhost:20128`;
с `--remote` (или активным удалённым контекстом) команда запрашивает каталог с этого
сервера и записывает конфиг локально.

| Команда | Инструмент | Что записывает | Ключевые флаги | Локально/удалённо |
|---------|------------|----------------|----------------|-------------------|
| `omniroute setup-codex` | OpenAI Codex CLI | `~/.codex/<name>.config.toml` — профиль на каждую совместимую текстовую модель (`codex --profile <name>`) | `--remote` `--api-key` `--only` `--dry-run` `--port` `--codex-home` | Оба |
| `omniroute setup-claude` | Claude Code | `~/.claude/profiles/<name>/settings.json` — профиль на каждую подходящую модель (`CLAUDE_CONFIG_DIR`) | `--remote` `--api-key` `--only` `--dry-run` `--port` `--claude-home` | Оба |
| `omniroute setup-opencode` | OpenCode (openai-совместимый) | `~/.config/opencode/opencode.json` — провайдер `omniroute` со всеми моделями каталога (`opencode -m omniroute/<model>`) | `--remote` `--api-key` `--only` `--model` `--dry-run` `--port` | Оба |
| `omniroute setup-cline` | Cline | `~/.cline/data/{globalState,secrets}.json` (режим CLI) + печатает настройки расширения VS Code | `--remote` `--api-key` `--model` `--yes` `--dry-run` `--port` `--cline-dir` | Оба |
| `omniroute setup-kilo` | Kilo Code | `~/.local/share/kilo/auth.json` (CLI) + объединяет `kilocode.*` в `settings.json` VS Code, если он есть | `--remote` `--api-key` `--model` `--yes` `--dry-run` `--port` `--auth-path` `--vscode-settings` | Оба |
| `omniroute setup-continue` | Continue / CLI `cn` | `~/.continue/config.yaml` — модели `provider: openai`, ключ через `${{ secrets.OMNIROUTE_API_KEY }}` | `--remote` `--api-key` `--only` `--dry-run` `--port` `--config-path` | Оба |
| `omniroute setup-cursor` | Cursor | Ничего — печатает шаги внутри приложения (конфиг Cursor — непрозрачный SQLite) | `--remote` `--api-key` `--only` `--port` | Оба |
| `omniroute setup-roo` | Roo Code | `~/.omniroute/roo-settings.json` (документ для импорта) + задаёт `roo-cline.autoImportSettingsPath`, если существует `settings.json` VS Code | `--remote` `--api-key` `--model` `--yes` `--dry-run` `--port` `--import-path` `--vscode-settings` | Оба |
| `omniroute setup-crush` | Crush | `~/.config/crush/crush.json` — провайдер `openai-compat`, ключ через `$OMNIROUTE_API_KEY` | `--remote` `--api-key` `--only` `--dry-run` `--port` `--config-path` | Оба |
| `omniroute setup-goose` | Goose | `~/.config/goose/config.yaml` (`GOOSE_PROVIDER`/`OPENAI_HOST`/`GOOSE_MODEL`) + печатает рецепт переменных окружения | `--remote` `--api-key` `--model` `--yes` `--dry-run` `--port` `--config-path` | Оба |
| `omniroute setup-qwen` | Qwen Code | `~/.qwen/settings.json` — `modelProvider` openai, ключ через `envKey` (`OMNIROUTE_API_KEY`) | `--remote` `--api-key` `--model` `--yes` `--dry-run` `--port` `--config-path` | Оба |
| `omniroute setup-aider` | Aider | `~/.aider.conf.yml` (`openai-api-base` + `model: openai/<id>`) + печатает рецепт переменных окружения | `--remote` `--api-key` `--model` `--yes` `--dry-run` `--port` `--config-path` | Оба |
| `omniroute launch` | Claude Code | Ничего — запускает `claude` с подставленными `ANTHROPIC_BASE_URL`/`ANTHROPIC_AUTH_TOKEN` | `--remote` `--api-key` `--token` `--profile` `--port` | Оба |
| `omniroute launch-codex` | OpenAI Codex CLI | Ничего — запускает `codex` с провайдером `omniroute`, подставленным через флаги `-c` | `--remote` `--api-key` `--profile` (`-p`) `--port` | Оба |

Заметки о флагах (проверено в исходниках команд):

- `--remote <url>` — запросить каталог с удалённого OmniRoute (переопределяет `--port`
  и активный контекст). `--api-key <key>` передаёт учётные данные для этого
  сервера (по умолчанию — переменная окружения `OMNIROUTE_API_KEY` или токен активного контекста).
- `--only <patterns>` — подстроки через запятую; оставить только те ID моделей, которые совпадают
  (напр. `--only glm,kimi`). Доступно в `setup-codex`, `setup-claude`,
  `setup-opencode`, `setup-continue`, `setup-cursor`, `setup-crush`.
- `--dry-run` — вывести ровно то, что было бы записано, не трогая
  файловую систему. Доступно в каждой команде `setup-*` **кроме** `setup-cursor`
  (которая никогда не пишет файл).
- `--model <id>` — обязателен (или выбирается интерактивно) для инструментов без
  автообнаружения моделей: Cline, Kilo, Roo, Goose, Qwen, Aider. Эти инструменты
  также принимают `--yes` для неинтерактивного запуска (что тогда требует `--model`).
  `setup-opencode` принимает `--model` для задания модели верхнего уровня по умолчанию.
- `--port <port>` — порт локального OmniRoute (по умолчанию `20128`, игнорируется при
  заданном `--remote`). Есть во всех `setup-*` и обоих лаунчерах.
- Два лаунчера (`launch`, `launch-codex`) принимают `--profile <name>` для выбора
  профиля, записанного `setup-claude` / `setup-codex`, плюс сквозные аргументы для
  нижележащего бинарника `claude` / `codex`.

> `setup-opencode` — это **облегчённая openai-совместимая** интеграция OpenCode.
> Есть также более богатая плагинная интеграция — `omniroute setup opencode` — которая
> устанавливает `@omniroute/opencode-plugin`. Это разные команды; таблица
> выше документирует `setup-opencode`.

---

## Локальное использование

С запущенным на `localhost:20128` OmniRoute просто выполните команду настройки для вашего
инструмента. Каталог запрашивается с локального сервера.

```bash
# Codex: записать профиль на каждую подходящую модель в ~/.codex/
omniroute setup-codex
codex --profile glm52            # использовать сгенерированный профиль

# Claude Code: записать профили на каждую модель, затем запустить один
omniroute setup-claude
omniroute launch --profile glm52

# OpenCode: записать openai-совместимого провайдера со всеми моделями каталога
omniroute setup-opencode
export OMNIROUTE_API_KEY=sk-...  # указывается через {env:OMNIROUTE_API_KEY}, никогда на диске
opencode -m omniroute/glm/glm-5.2 "..."

# Инструменты без автообнаружения требуют явной модели:
omniroute setup-aider --model glm/glm-5.2
omniroute setup-qwen  --model kmc/kimi-k2.7

# Предпросмотр без записи:
omniroute setup-continue --dry-run
```

Запуск вообще без записи конфигов (только подстановка окружения):

```bash
omniroute launch                 # Claude Code → локальный OmniRoute
omniroute launch-codex           # Codex CLI → локальный OmniRoute
omniroute launch-codex --profile glm52
```

---

## Удалённое использование

Направьте любую команду настройки на удалённый OmniRoute с `--remote` + `--api-key`. Каталог
запрашивается с удалённого сервера; конфиг записывается на вашей локальной машине.

```bash
# OpenCode против удалённого VPS, оставить только модели glm/kimi
omniroute setup-opencode --remote http://192.168.0.15:20128 --api-key oma_live_xxx \
  --only glm,kimi
opencode -m omniroute/glm/glm-5.2 "..."   # сначала export OMNIROUTE_API_KEY

# Профили Codex из удалённого каталога
omniroute setup-codex --remote http://192.168.0.15:20128 --api-key oma_live_xxx

# Запустить CLI напрямую против удалённого
omniroute launch       --remote http://192.168.0.15:20128 --api-key oma_live_xxx
omniroute launch-codex --remote http://192.168.0.15:20128 --api-key oma_live_xxx
```

Вместо передачи `--remote`/`--api-key` каждый раз войдите один раз и позвольте
**активному контексту** подставлять их автоматически:

```bash
omniroute connect 192.168.0.15        # выпускает токен с областью, сохраняет контекст
omniroute setup-codex                 # ← теперь использует удалённый каталог
omniroute setup-opencode              # ← то же самое
omniroute launch                      # ← Claude Code против удалённого
```

Контексты, области и управление токенами см. в [Удалённый режим](./REMOTE-MODE.md).

---

## Соглашения о Base URL (каким инструментам нужен `/v1`)

OmniRoute предоставляет поверхность OpenAI на `/v1`, поверхность Anthropic в корне
и нативную поверхность Gemini на `/v1beta`. Каждая интеграция подключена к той форме, которую
ожидает её инструмент (проверено в исходниках команд):

| Интеграция | Записываемый Base URL | `/v1`? |
|------------|-----------------------|--------|
| `setup-cline` (`openAiBaseUrl`) | корень | Нет — Cline дописывает `/v1/chat/completions` |
| `setup-goose` (`OPENAI_HOST`) | корень | Нет — Goose дописывает путь |
| `setup-aider` (`OPENAI_API_BASE`) | корень | Нет — LiteLLM дописывает `/v1/chat/completions` |
| `setup-kilo`, `setup-roo`, `setup-continue`, `setup-crush`, `setup-qwen`, `setup-cursor` | с `/v1` | Да |
| `setup-claude` (`ANTHROPIC_BASE_URL`), `launch` | корень | Нет — Claude Code дописывает `/v1/messages` |
| `setup-codex`, `launch-codex` (`model_providers.omniroute.base_url`) | с `/v1` | Да |

---

## Сохранение нативных зависимостей при обновлении: `--include=optional`

Когда вы обновляетесь через `omniroute update` (после подтверждения или с `--apply`),
OmniRoute выполняет установку со встроенным `--include=optional`:

```bash
npm install -g omniroute@latest --include=optional
```

Это **не** флаг, который вы передаёте в `omniroute update` — он всегда применяется
обновлятором. Он гарантирует, что `optionalDependencies` (`better-sqlite3`, `keytar`,
`tls-client`, стек SLM LLMLingua) переживут обновление, даже если в вашей конфигурации npm
задано `omit=optional`, что иначе молча уронило бы нативный драйвер SQLite
и биндинг системного хранилища ключей. Чтобы предварительно просмотреть точную команду без применения:

```bash
omniroute update --dry-run
# [DRY RUN] Would run: npm install -g omniroute@latest --include=optional
```

Прочие флаги `omniroute update` (проверено в исходниках): `--check` (код выхода 1, если
устарело), `--apply` (установить без запроса), `--changelog`, `--no-backup`,
`--yes`.

---

## См. также

- [Настройка Claude Code](./CLAUDE-CODE-CONFIGURATION.md) — более глубокое руководство по Claude Code
- [Настройка Codex CLI](./CODEX-CLI-CONFIGURATION.md) — разовая базовая настройка `[model_providers.omniroute]`
- [Удалённый режим](./REMOTE-MODE.md) — контексты, токены доступа с областью, управление удалённым сервером
- [Справочник CLI-инструментов](../reference/CLI-TOOLS.md) — полный каталог поддерживаемых инструментов + страницы панели
- [Руководство по установке](./SETUP_GUIDE.md) — способы установки и первоначальная настройка
