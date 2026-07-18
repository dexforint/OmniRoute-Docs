---
title: "Codex CLI — настройка с OmniRoute"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Codex CLI — настройка с OmniRoute

Полное руководство по использованию Codex CLI, направленного на OmniRoute как OpenAI-совместимый бэкенд.

---

## Готовый к вставке config.toml

Замените `<YOUR_HOST>` и `<YOUR_KEY>` своими значениями:

```toml
# ~/.codex/config.toml
model                          = "cx/gpt-5.5"
model_provider                 = "omniroute"
model_reasoning_effort         = "xhigh"
model_context_window           = 400000
model_auto_compact_token_limit = 350000
tool_output_token_limit        = 32768    # предел хранения истории на вызов инструмента

[model_providers.omniroute]
name                 = "OmniRoute"
base_url             = "http://<YOUR_HOST>:20128/v1"
env_key              = "OMNIROUTE_API_KEY"
requires_openai_auth = false
wire_api             = "responses"
```

```bash
# ~/.bashrc или ~/.zshrc — реальное значение ключа, никогда в config.toml
export OMNIROUTE_API_KEY="<YOUR_KEY>"
```

> **Распространённые варианты хоста**
>
> | Доступ         | URL                           |
> | -------------- | ----------------------------- |
> | Локальная сеть | `http://192.168.0.1:20128/v1` |
> | Tailscale      | `http://100.x.x.x:20128/v1`   |
> | Loopback       | `http://localhost:20128/v1`   |

---

## `wire_api = "responses"` — почему это работает для всех моделей

Codex CLI объявил устаревшим `wire_api = "chat"` (Chat Completions) в феврале 2026 и теперь **требует** `wire_api = "responses"` (OpenAI Responses API). Установка `wire_api = "chat"` вызывает немедленный сбой при запуске начиная с v0.138.

DeepSeek, GLM, Kimi и другие предоставляют только эндпоинт Chat Completions — не Responses API. Если направить Codex прямо на них, он упадёт.

**OmniRoute решает это прозрачно:**

```
Codex CLI
  → wire_api = "responses"
  → POST /v1/responses (OmniRoute)
    → Трансформер Responses ↔ Chat Completions OmniRoute
    → POST /chat/completions (DeepSeek / Mistral / GLM / Kimi / любой провайдер)
```

Вам никогда не нужен отдельный прокси-транслятор при использовании OmniRoute. **Все модели используют `wire_api = "responses"`** — об остальном позаботится OmniRoute.

> **`wire_api` имеет значение по умолчанию** — поле по умолчанию равно `"responses"` и может быть полностью опущено в `config.toml`. Задавайте его явно, только если документируете намерение.

---

## Контекстное окно и сжатие

### Поля конфигурации токенов

| Поле                             | Описание                                                                                                                                                                             |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `model_context_window`           | Общий бюджет токенов для активной модели. Устанавливайте равным заявленному лимиту модели.                                                                                           |
| `model_auto_compact_token_limit` | Порог, запускающий автоматическое сжатие истории. **Максимум: 90% от `model_context_window`** — значения выше 90% молча игнорируются.                                                |
| `tool_output_token_limit`        | Предел токенов, хранимых на один вывод вызова инструмента в истории. Не даёт одному большому ответу инструмента заполнить окно. **Это не макс. вывод** — это предел хранения истории. |
| `compact_prompt`                 | Встроенное переопределение системного промпта, используемого при сжатии (v0.138+).                                                                                                   |

> **Замечание о `model_max_output_tokens`**: это поле **не входит в схему конфигурации Codex CLI** (отсутствует в кодовой базе Codex на Rust). При установке оно молча игнорируется. Не полагайтесь на него — используйте `tool_output_token_limit`, чтобы контролировать, сколько вывода инструментов хранится в истории.

### Контекстные окна по моделям

| Модель                               | ID в OmniRoute                       | Контекстное окно       | `auto_compact` | `tool_output_limit` |
| ------------------------------------ | ------------------------------------ | ---------------------- | -------------- | ------------------- |
| GPT-5.5                              | `cx/gpt-5.5`                         | 400k надёжно (1M макс.) | 350,000      | 32,768              |
| Kimi K2.7 (thinking)                 | `kmc/kimi-k2.7`                      | 131,072                | 112,000        | 32,768              |
| Kimi K2.6                            | `kmc/kimi-k2.6`                      | 131,072                | 112,000        | 32,768              |
| GLM-5.2 / 5.2-max (thinking)         | `glm/glm-5.2`                        | 131,072                | 112,000        | 32,768              |
| MiMo V2.5 Pro (thinking)             | `opencode-go/mimo-v2.5-pro`          | 131,072                | 112,000        | 32,768              |
| Qwen 3.7 Plus (thinking)             | `opencode-go/qwen3.7-plus`           | 32,768                 | 28,000         | 16,384              |
| DeepSeek V4 Pro (OllamaCloud)        | `ollamacloud/deepseek-v4-pro`        | 131,072                | 112,000        | 32,768              |
| DeepSeek V4 Pro                      | `ds/deepseek-v4-pro`                 | 1,000,000              | 900,000        | 65,536              |
| MiMo V2.5                            | `opencode-go/mimo-v2.5`              | 131,072                | 112,000        | 32,768              |
| Gemma 4 31B (OllamaCloud)            | `ollamacloud/gemma4:31b`             | 32,768                 | 28,000         | 16,384              |
| Nemotron 3 Super (OllamaCloud)       | `ollamacloud/nemotron-3-super`       | 32,768                 | 28,000         | 16,384              |
| GPT-OSS 20B (OllamaCloud)            | `ollamacloud/gpt-oss:20b`            | 32,768                 | 28,000         | 16,384              |
| DeepSeek V4 Flash (OllamaCloud)      | `ollamacloud/deepseek-v4-flash`      | 65,536                 | 56,000         | 16,384              |
| Gemini 3 Flash Preview (OllamaCloud) | `ollamacloud/gemini-3-flash-preview` | 1,000,000              | 850,000        | 32,768              |
| GLM-5 Turbo                          | `glm/glm-5-turbo`                    | 131,072                | 112,000        | 16,384              |
| GLM-4.7 Flash                        | `glm/glm-4.7-flash`                  | 131,072                | 112,000        | 16,384              |
| Mistral Large Latest                 | `mistral/mistral-large-latest`       | 262,144                | 220,000        | 16,384              |

> **Формула сжатия:** `effective_window = model_context_window - min(tool_output_token_limit, 20000)`. Значения выше 20k не меняют порог срабатывания сжатия.

> **Эмпирическое правило:** устанавливайте `model_auto_compact_token_limit` в 85–88% от `model_context_window`. Никогда не превышайте 90% — молча игнорируется.

---

## Префикс модели: `cx/`

Все модели Codex в OmniRoute используют префикс `cx/`:

| Имя в Codex CLI         | Модель в OmniRoute   |
| ----------------------- | -------------------- |
| `cx/gpt-5.5`            | GPT-5.5 стандарт     |
| `cx/gpt-5.4`            | GPT-5.4 стандарт     |
| `cx/gpt-5.4-mini`       | GPT-5.4 mini         |
| `cx/gpt-5.1-codex-mini` | GPT-5.1 Codex mini   |

Другие провайдеры используют собственный префикс (`kmc/`, `glm/`, `ds/`, `ollamacloud/`, `opencode-go/`, `mistral/`) — префикс совпадает с алиасом провайдера OmniRoute.

---

## Reasoning Effort

Управляет тем, сколько модель «думает» перед ответом.

| Значение | Использовать для                                |
| -------- | ----------------------------------------------- |
| `none`   | Без рассуждений — прямой ответ                  |
| `low`    | Тривиальные задачи (переименование, форматирование) |
| `medium` | **Значение сервера по умолчанию**, если не указано |
| `high`   | Промежуточные задачи (рефакторинг, отладка)     |
| `xhigh`  | Архитектура, глубокий анализ, сложные задачи    |

```bash
# Переопределение на один вызов
codex -c model_reasoning_effort=low "rename variable x to count"
codex -c model_reasoning_effort=xhigh "design the auth module"
```

---

## Профили — именованные конфигурации для моделей/процессов

Профили позволяют переключать модель + контекстное окно одним флагом. Каждый профиль — это плоский
`~/.codex/<name>.config.toml`, накладываемый поверх базового `config.toml`.

> **Правило именования (Codex CLI v0.137+):** файл должен быть `~/.codex/<name>.config.toml` — **без префикса `profile-`**.
> CLI разрешает `-p kimi-k27` → `~/.codex/kimi-k27.config.toml`. Если файл не найден, молча применяется значение по умолчанию.

```bash
codex --profile kimi-k27 "analyze 10k lines of this codebase"
codex -p glm52 "architecture review"
codex --profile deepseek-flash "rename variable"   # быстро, дёшево
```

### Профили effort (одна модель, разные усилия)

```bash
codex -p low      # cx/gpt-5.5, effort=low
codex -p medium   # cx/gpt-5.5, effort=medium
codex -p high     # cx/gpt-5.5, effort=high
codex -p xhigh    # cx/gpt-5.5, effort=xhigh (по умолчанию)
codex -p chat     # cx/gpt-5.5, усилие не задано (значение сервера по умолчанию)
```

### Модели с мышлением (alto pensamento) — xhigh + подробная сводка

| Профиль      | Модель                      | Контекст | Использовать для             |
| ------------ | --------------------------- | -------- | ---------------------------- |
| `kimi-k27`   | `kmc/kimi-k2.7`             | 128k     | Лучшее качество мышления (Kimi) |
| `glm52`      | `glm/glm-5.2`               | 128k     | Мышление GLM                 |
| `glm52max`   | `glm/glm-5.2-max`           | 128k     | Мышление GLM max             |
| `mimo-pro`   | `opencode-go/mimo-v2.5-pro` | 128k     | Мышление MiMo                |
| `qwen37plus` | `opencode-go/qwen3.7-plus`  | 32k      | Мышление Qwen                |

### Хорошие модели (bons) — высокое усилие

| Профиль        | Модель                        | Контекст | Использовать для                  |
| -------------- | ----------------------------- | -------- | --------------------------------- |
| `kimi-k26`     | `kmc/kimi-k2.6`               | 128k     | Общего назначения (Kimi)          |
| `deepseek-pro` | `ollamacloud/deepseek-v4-pro` | 128k     | DeepSeek Pro через OllamaCloud    |
| `deepseek`     | `ds/deepseek-v4-pro`          | 1M       | DeepSeek Pro напрямую, огромный контекст |
| `mimo`         | `opencode-go/mimo-v2.5`       | 128k     | MiMo общего назначения            |

### Простые модели (simples) — без усилия рассуждений

| Профиль    | Модель                         | Контекст | Использовать для        |
| ---------- | ------------------------------ | -------- | ----------------------- |
| `gemma4`   | `ollamacloud/gemma4:31b`       | 32k      | Экономичная, способная  |
| `nemotron` | `ollamacloud/nemotron-3-super` | 32k      | NVIDIA Nemotron         |
| `gptoss`   | `ollamacloud/gpt-oss:20b`      | 32k      | GPT с открытым кодом    |

### Быстрые модели — низкое усилие

| Профиль          | Модель                               | Контекст | Использовать для          |
| ---------------- | ------------------------------------ | -------- | ------------------------- |
| `deepseek-flash` | `ollamacloud/deepseek-v4-flash`      | 64k      | Быстрые задачи            |
| `gemini-flash`   | `ollamacloud/gemini-3-flash-preview` | 1M       | Очень быстрая, огромный контекст |
| `glm5turbo`      | `glm/glm-5-turbo`                    | 128k     | GLM Turbo                 |
| `glm47flash`     | `glm/glm-4.7-flash`                  | 128k     | GLM Flash                 |
| `mistral`        | `mistral/mistral-large-latest`       | 256k     | Mistral Large             |

### Таблица быстрого выбора

| Задача                           | Рекомендуемый профиль                              |
| -------------------------------- | -------------------------------------------------- |
| Переименование, форматирование, шаблонный код | `--profile deepseek-flash` или `-p low`   |
| Объяснение, лёгкое ревью         | `-p chat` или `-p gemini-flash`                    |
| Отладка, умеренный рефакторинг   | `-p medium` или `-p kimi-k26`                      |
| Новая функция, сложные тесты     | `-p high` или `-p mimo`                            |
| Архитектура, глубокий анализ     | `-p kimi-k27` или `-p glm52` или `-p xhigh`        |
| Анализ кодовой базы (нужен контекст 1M) | `--profile deepseek` или `--profile gemini-flash` |
| Максимальное качество мышления   | `-p glm52max` или `-p mimo-pro`                    |
| Экономия средств                 | `-p gemma4` или `-p gptoss`                        |

---

## Автоматическая генерация профилей через `omniroute setup-codex`

Если вы запускаете OmniRoute на VPS, вы можете автоматически генерировать файлы профилей из живого каталога моделей:

```bash
# С VPS (использует локальный OmniRoute на порту 20128)
omniroute setup-codex

# С любой машины — направьте на ваш VPS
omniroute setup-codex --remote http://100.x.x.x:20128 --api-key sk-xxx

# Предпросмотр без записи файлов
omniroute setup-codex --remote http://100.x.x.x:20128 --dry-run

# Генерировать только профили GLM и Kimi
omniroute setup-codex --only glm,kimi

# Записать в пользовательский каталог
omniroute setup-codex --codex-home /path/to/.codex
```

Команда запрашивает `/v1/models`, использует настроенные профили для известных моделей, откатывается к метаданным каталога для других совместимых текстовых моделей и пишет `~/.codex/<name>.config.toml` для каждой. Идемпотентно — безопасно запускать повторно.

OmniRoute также может **автосинхронизировать** эти же файлы профилей после того, как успешное обнаружение/импорт моделей провайдера меняет живой каталог. Это **opt-in и по умолчанию выключено**: включите из **панели CLI Code** («CLI profile auto-sync» → Codex) или задайте `OMNIROUTE_AUTO_SYNC_CODEX_PROFILES=true` (также учитывается `CLI_ALLOW_CONFIG_WRITES`, по умолчанию включён). Когда включено, записываются только отдельные файлы профилей `~/.codex/*.config.toml`; никогда не меняются активный/дефолтный `~/.codex/config.toml`, настройки Codex-lb, авторизация или выбор провайдера.

---

## Запуск Codex через `omniroute launch-codex`

Проверяет здоровье вашего экземпляра OmniRoute перед запуском Codex:

```bash
# Запуск против локального OmniRoute (порт по умолчанию 20128)
omniroute launch-codex

# Запуск с конкретным профилем
omniroute launch-codex --profile kimi-k27

# Запуск против удалённого VPS
omniroute launch-codex --remote http://100.x.x.x:20128/v1 --api-key sk-xxx

# Передать дополнительные аргументы codex
omniroute launch-codex --profile glm52 -- --yolo "fix this bug"
```

---

## Новые возможности Codex CLI (v0.138–v0.141)

| Версия | Возможность                                                                                                                                                        |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| v0.138 | Передача в десктопное приложение (`/app`), персональные токены доступа v2, `--profile` как единственный селектор профиля (устаревшие внутрифайловые таблицы `[profiles]` вызывают сбой при запуске) |
| v0.139 | `web_search = "live"` — нативный веб-поиск из режима кода; `oneOf`/`allOf` в схемах инструментов MCP; диагностика окружения `codex doctor`                          |
| v0.140 | Просмотр токенов `/usage` внутри сессии; `/import` из сессий Claude Code; подкоманда `codex delete <SESSION_ID>`; авторизация Amazon Bedrock через объект `aws` в конфиге провайдера |
| v0.141 | Зашифрованный E2E Noise relay для удалённых исполнителей; исправление SQLite WAL; поддержка TLS P-521                                                                |

### Новые поля `config.toml` (после v0.137)

```toml
# Нативный веб-поиск (v0.139)
web_search = "live"   # "disabled" | "cached" | "live"

# Отдельный системный промпт разработчика (v0.138)
developer_instructions = "Always prefer functional style."

# Пользовательский промпт сжатия
compact_prompt = "Summarise the above as bullet points."

# Направить /review на более дешёвую модель
review_model = "glm/glm-5-turbo"

# Уровень обслуживания OpenAI
service_tier = "fast"   # "fast" | "flex"
```

### Новые поля `[model_providers.<id>]`

```toml
[model_providers.omniroute]
base_url             = "http://100.x.x.x:20128/v1"
env_key              = "OMNIROUTE_API_KEY"
requires_openai_auth = false

# Статические дополнительные заголовки на каждый запрос
[model_providers.omniroute.http_headers]
"X-Custom-Header" = "value"

# Заголовки, читаемые из переменных окружения
[model_providers.omniroute.env_http_headers]
"X-Trace-Id" = "TRACE_ID"

# Дополнительные query-параметры URL (полезно для Azure api-version)
[model_providers.omniroute.query_params]
"api-version" = "2024-12-01-preview"
```

### Авторизация Amazon Bedrock (v0.140)

```toml
[model_providers.bedrock]
base_url = "https://bedrock-runtime.us-east-1.amazonaws.com"

[model_providers.bedrock.aws]
profile = "default"   # профиль ~/.aws/credentials
region  = "us-east-1"
```

---

## Несколько серверов

```toml
[model_providers.omniroute-main]
base_url = "http://192.168.0.1:20128/v1"
env_key  = "OMNIROUTE_API_KEY"

[model_providers.omniroute-tailscale]
base_url = "http://100.x.x.x:20128/v1"
env_key  = "OMNIROUTE_API_KEY"
```

---

## Claude Code — эквивалентная конфигурация

| Codex CLI (`config.toml`)         | Claude Code (переменная окружения)    | Эффект                  |
| --------------------------------- | ------------------------------------- | ----------------------- |
| `tool_output_token_limit = 32768` | _(не предоставлено напрямую)_         | Предел истории на инструмент |
| `model_context_window = 400000`   | _(определяется моделью)_              | Контекстное окно        |
| —                                 | `CLAUDE_CODE_MAX_OUTPUT_TOKENS=65536` | Макс. токенов на ответ  |

```bash
# ~/.bashrc — предел токенов Claude Code
export CLAUDE_CODE_MAX_OUTPUT_TOKENS=65536
```

---

## Краткий справочник — флаги CLI

| Флаг                  | Короткий | Эффект                                       |
| --------------------- | -------- | -------------------------------------------- |
| `--model <id>`        | `-m`     | Переопределяет `model` для этого вызова      |
| `--profile <name>`    | `-p`     | Загружает `~/.codex/<name>.config.toml`      |
| `--config key=value`  | `-c`     | Переопределяет любое поле config.toml (повторяемо) |
| `--enable <feature>`  | —        | Принудительно включает флаг функции          |
| `--disable <feature>` | —        | Принудительно выключает флаг функции         |
| `--search`            | —        | Включить живой веб-поиск для этого вызова    |

Новое в v0.140:

```bash
codex delete <SESSION_ID>          # удалить сессию
codex delete <SESSION_ID> --force  # без подтверждения
codex debug models --bundled       # вывести встроенный каталог моделей как JSON
```

Внутри интерактивной сессии:

| Команда   | Эффект                                      |
| --------- | ------------------------------------------- |
| `/model`  | Открывает выбор модели                      |
| `/usage`  | Показывает использование токенов этой сессии (v0.140) |
| `/app`    | Передаёт в десктопное приложение (v0.138)   |
| `/import` | Импортировать сессию Claude Code (v0.140)  |
| `/help`   | Выводит все slash-команды                   |

---

## Устранение неполадок

**`Error: wire_api = "chat" is no longer supported`**
Удалите `wire_api = "chat"` из вашего конфига. Задайте `wire_api = "responses"` или опустите поле (по умолчанию `"responses"` с v0.138).

**`Error: model not found`**
Убедитесь, что модель существует в OmniRoute с правильным префиксом. Используйте `omniroute models list` или откройте `/dashboard/providers/<provider>`.

**`Authentication error`**
Убедитесь, что `OMNIROUTE_API_KEY` экспортирован: `echo $OMNIROUTE_API_KEY`.

**`Connection refused`**
Убедитесь, что OmniRoute запущен и хост/порт в `base_url` правильный для вашей сети (локальная vs Tailscale vs VPS).

**Сессия падает около предела контекста**
Задайте `model_context_window` и `model_auto_compact_token_limit` явно. См. таблицу контекстных окон выше.

**Сжатие срабатывает слишком поздно**
Понизьте `model_auto_compact_token_limit` до 80–85% окна. Никогда не задавайте выше 90%.

**Профиль не загружается (`-p <name>` молча игнорируется)**
Убедитесь, что файл существует по пути `~/.codex/<name>.config.toml` (без префикса `profile-`). Выполните `ls ~/.codex/*.config.toml`.
