---
title: "Claude Code CLI — настройка с OmniRoute"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Claude Code CLI — настройка с OmniRoute

Направьте CLI **Claude Code** (`claude`) на OmniRoute — локальный или удалённый VPS —
с профилями для каждой модели, по аналогии с настройкой Codex.

---

## Быстрый старт

```bash
# Запустить Claude Code против локального OmniRoute (автоматически определяет активный контекст)
omniroute launch

# Против удалённого OmniRoute (после `omniroute connect <host>` это происходит автоматически)
omniroute launch --remote http://192.168.0.15:20128 --api-key oma_live_xxx

# Сгенерировать профили для каждой модели, затем запустить один
omniroute setup-claude            # записывает ~/.claude/profiles/<name>/settings.json
omniroute launch --profile glm52  # Claude Code, использующий glm/glm-5.2 через OmniRoute
```

---

## Как Claude Code подключается к шлюзу

Claude Code говорит по **Anthropic Messages API** и направляется на пользовательский
эндпоинт переменными окружения (флага `--base-url` у него нет):

| Переменная                                   | Назначение                                                                              |
| -------------------------------------------- | --------------------------------------------------------------------------------------- |
| `ANTHROPIC_BASE_URL`                         | Корневой URL шлюза (Claude Code дописывает `/v1/messages`). **Без суффикса `/v1`.**     |
| `ANTHROPIC_AUTH_TOKEN`                       | Отправляется как `Authorization: Bearer …` — используйте токен доступа / API-ключ OmniRoute |
| `ANTHROPIC_API_KEY`                          | Альтернатива: отправляется как `x-api-key`. Если заданы обе, побеждает `ANTHROPIC_AUTH_TOKEN` |
| `ANTHROPIC_MODEL`                            | Принудительно задать конкретную модель (переопределяет выбор по умолчанию в `/model`)   |
| `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` | `1` → нативный выбор `/model` перечисляет модели `claude*`/`anthropic*` из `/v1/models` |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS`              | Ограничение выходных токенов на ответ (напр. `65536`)                                   |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW`            | Порог токенов для автосжатия                                                            |

> Переменные окружения читаются **один раз при запуске** — перезапускайте Claude Code после их изменения.

`omniroute launch` задаёт все эти переменные за вас: он разрешает base URL + токен
из активного контекста (поэтому `omniroute connect <vps>`, а затем `omniroute launch`
просто работает), проверяет здоровье сервера и запускает `claude`.

---

## Профили (`CLAUDE_CONFIG_DIR`)

У Claude Code **нет нативных файлов профилей** (в отличие от `~/.codex/<name>.config.toml` у Codex).
Идиоматичный механизм — `CLAUDE_CONFIG_DIR`: отдельный каталог конфигурации на
профиль, каждый со своими `settings.json`, учётными данными, историей и кэшем.

`omniroute setup-claude` запрашивает живой каталог `/v1/models` и записывает один
профиль на модель в `~/.claude/profiles/<name>/settings.json`, используя
**те же имена, что и `setup-codex`** (`glm52`, `kimi-k27`, `deepseek-pro`, …):

```jsonc
// ~/.claude/profiles/glm52/settings.json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "glm/glm-5.2",
  "effortLevel": "xhigh",
  "env": {
    "ANTHROPIC_BASE_URL": "http://192.168.0.15:20128",
    "ANTHROPIC_MODEL": "glm/glm-5.2",
    "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY": "1",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "190000",
  },
}
```

> **Токен авторизации никогда не записывается в профиль.** Запускайте через
> `omniroute launch --profile <name>` (он подставляет `ANTHROPIC_AUTH_TOKEN` из
> активного контекста) или экспортируйте `ANTHROPIC_AUTH_TOKEN` сами и выполните
> `CLAUDE_CONFIG_DIR=~/.claude/profiles/<name> claude`.

**Автосинхронизация после обнаружения моделей (opt-in).** OmniRoute может автоматически пересоздавать
эти же файлы `~/.claude/profiles/<name>/settings.json` всякий раз, когда синхронизация моделей провайдера
меняет живой каталог — так новые/переименованные модели получают профили без повторного запуска
команды. Это **по умолчанию выключено**: включите из **панели CLI Code** («CLI profile
auto-sync» → Claude Code) или задайте `OMNIROUTE_AUTO_SYNC_CLAUDE_PROFILES=true` (также учитывается
`CLI_ALLOW_CONFIG_WRITES`, по умолчанию включён). Когда включено, записываются только файлы профилей; никогда
не меняются ваш активный/дефолтный конфиг Claude, авторизация или `~/.claude/settings.json`.

### Генерация и использование профилей

```bash
# Локальный OmniRoute
omniroute setup-claude

# Удалённый VPS (вшивает URL VPS в каждый профиль)
omniroute setup-claude --remote http://192.168.0.15:20128 --api-key oma_live_xxx

# Только некоторые провайдеры
omniroute setup-claude --only glm,kimi

# Предпросмотр без записи
omniroute setup-claude --dry-run

# Запуск профиля
omniroute launch --profile kimi-k27
```

---

## Уровни моделей (необязательно)

Claude Code маршрутизирует на уровни возможностей. Сопоставьте каждый с моделью OmniRoute через окружение /
настройки, если хотите разных провайдеров на уровень:

```bash
export ANTHROPIC_DEFAULT_OPUS_MODEL="glm/glm-5.2"
export ANTHROPIC_DEFAULT_SONNET_MODEL="kmc/kimi-k2.6"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="glm/glm-4.7-flash"
```

Иначе для всего используется единый `ANTHROPIC_MODEL` (что и задают профили).

---

## Удалённый режим

После выполнения `omniroute connect <host>` (см.
[Удалённый режим](./REMOTE-MODE.md)) `omniroute launch` и `omniroute setup-claude`
автоматически нацеливаются на этот удалённый сервер и используют его токен доступа с ограниченной областью — дополнительные
флаги не нужны. Переопределяйте на каждый вызов через `--remote` / `--api-key`.

---

## Устранение неполадок

**Claude Code игнорирует шлюз** — убедитесь, что в `ANTHROPIC_BASE_URL` **нет
`/v1`**, и перезапустите `claude` (окружение читается один раз при запуске). `omniroute launch`
делает это за вас.

**Выбор `/model` пуст / нет моделей шлюза** — нужен Claude Code
v2.1.129+ и `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1`. В выборе появляются только ID моделей `claude*` /
`anthropic*`; принудительно задайте любую другую модель через
`ANTHROPIC_MODEL=<id>` (именно это и делают профили).

**Ошибки авторизации** — в профиле нет токена. Используйте `omniroute launch --profile`
(подставляет его) или экспортируйте `ANTHROPIC_AUTH_TOKEN`.

**Профили не изолируются** — каждый профиль — это отдельный `CLAUDE_CONFIG_DIR`;
проверьте, что `echo $CLAUDE_CONFIG_DIR` внутри сессии указывает на
`~/.claude/profiles/<name>`.
