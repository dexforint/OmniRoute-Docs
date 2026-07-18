---
title: "📖 Руководство по установке — OmniRoute"
version: 3.8.40
lastUpdated: 2026-06-28
---

# 📖 Руководство по установке — OmniRoute

> Полный справочник по установке OmniRoute. Краткую версию см. в [Быстром старте в README](../README.md#-quick-start).

## Содержание

- [Способы установки](#install-methods)
- [Настройка CLI-инструментов](#cli-tool-configuration)
- [Настройка протоколов (MCP + A2A)](#protocol-setup-mcp--a2a)
- [Настройка таймаутов](#timeout-configuration)
- [Режим раздельных портов](#split-port-mode)
- [Void Linux (xbps-src)](#void-linux-xbps-src-template)
- [Удаление](#uninstalling)

---

## Способы установки

### npm (рекомендуется)

```bash
npm install -g omniroute
omniroute
```

Панель управления открывается на `http://localhost:20128`, а базовый URL API — `http://localhost:20128/v1`.

### pnpm

```bash
pnpm add -g omniroute@latest --allow-build=better-sqlite3 --allow-build=@swc/core
omniroute
```

> **Пользователям pnpm:** флаг `--allow-build` требуется, чтобы включить нативные скрипты сборки для `better-sqlite3` и `@swc/core`. Команда `pnpm approve-builds -g` не поддерживается для глобальных установок в pnpm v11.

### Arch Linux (AUR)

```bash
yay -S omniroute-bin
systemctl --user enable --now omniroute.service
```

[Пакет AUR](https://aur.archlinux.org/packages/omniroute-bin) устанавливает OmniRoute и предоставляет пользовательский сервис systemd.

### Из исходников

```bash
npm install
PORT=20128 DASHBOARD_PORT=20129 NEXT_PUBLIC_BASE_URL=http://localhost:20129 npm run dev
```

> **Примечание:** `npm install` автоматически генерирует `.env` из `.env.example` при первом запуске. Последующие установки не перезаписывают существующий `.env`, поэтому ваши настройки сохраняются. Чтобы пересоздать, удалите `.env` перед повторным запуском.

### Docker

Полную настройку Docker, включая профили Compose и Caddy HTTPS, см. в [Руководстве по Docker](./DOCKER_GUIDE.md).

### Десктопное приложение (Electron)

OmniRoute поставляет десктопную обёртку, построенную на Electron 41 + electron-builder 26.10. Доступные скрипты (корень воркспейса):

```bash
npm run electron:dev          # Запуск десктопа с горячей перезагрузкой
npm run electron:build        # Сборка для текущей ОС (автоопределение)
npm run electron:build:win    # Установщик Windows (NSIS + portable)
npm run electron:build:mac    # macOS (dmg + zip, arm64+x64)
npm run electron:build:linux  # Linux (AppImage + deb + rpm)
npm run electron:smoke:packaged  # Дымовой тест собранной сборки
```

Релизы десктопных установщиков прикреплены к GitHub Releases. Полное погружение в Electron (подписание, IPC-мост, дистрибутивы) см. в [`ELECTRON_GUIDE.md`](./ELECTRON_GUIDE.md) _(создан на более позднем этапе)_.

### Headless-сервер (CI/автоматизация)

Для установок без участия пользователя (Docker, Kubernetes, CI) используйте:

```bash
omniroute setup --non-interactive
omniroute providers test-batch
```

В сочетании с переменными окружения (`INITIAL_PASSWORD`, `OMNIROUTE_WS_BRIDGE_SECRET` и др.) это позволяет поднять полностью скриптуемый экземпляр OmniRoute.

### Опции CLI

| Команда                 | Описание                                                        |
| ----------------------- | --------------------------------------------------------------- |
| `omniroute`             | Запустить сервер (`PORT=20128`, API и панель на одном порту)    |
| `omniroute setup`       | Управляемый онбординг CLI: пароль и первый провайдер            |
| `omniroute doctor`      | Запустить локальные проверки здоровья без старта сервера        |
| `omniroute providers`   | Обнаружение, список, валидация и тестирование провайдеров из CLI |
| `omniroute config`      | Настройка CLI-инструментов — список, чтение, запись, валидация конфигов |
| `omniroute status`      | Офлайн-панель статуса — версия, БД, инструменты, конфиг         |
| `omniroute logs`        | Поток логов использования из API (поддерживает `--follow`)      |
| `omniroute update`      | Проверить или применить обновления OmniRoute                    |
| `omniroute provider`    | Управление подключениями провайдеров — добавить, список, удалить, тест, по умолчанию |
| `omniroute --port 3000` | Задать канонический/API-порт 3000                               |
| `omniroute --mcp`       | Запустить MCP-сервер (транспорт stdio)                          |
| `omniroute --no-open`   | Не открывать браузер автоматически                              |
| `omniroute --help`      | Показать справку                                                |

Headless-установку можно скриптовать флагами или переменными окружения:

```bash
omniroute setup --non-interactive --password "$OMNIROUTE_PASSWORD"
omniroute setup --non-interactive --add-provider --provider openai --api-key "$OPENAI_API_KEY"
omniroute setup --non-interactive --add-provider --provider openai --api-key "$OPENAI_API_KEY" --test-provider
```

Запуск локальной диагностики без открытия панели управления:

```bash
omniroute doctor
omniroute doctor --json
omniroute doctor --no-liveness
```

Управление провайдерами по SSH или из скриптов без открытия панели управления:

```bash
omniroute providers available
omniroute providers available --search openai
omniroute providers available --category api-key
omniroute providers list
omniroute providers test <id-or-name>
omniroute providers test-all
omniroute providers validate
```

---

## Настройка CLI-инструментов

### 1) Подключите провайдеров и создайте API-ключ

1. Откройте Панель → `Providers` и подключите хотя бы одного провайдера (OAuth или API-ключ).
2. Откройте Панель → `Endpoints` и создайте API-ключ.
3. (Опционально) Откройте Панель → `Combos` и задайте вашу цепочку переключений.

### 2) Направьте ваш инструмент для кодирования

```txt
Base URL: http://localhost:20128/v1
API Key:  [скопируйте со страницы Endpoint]
Model:    if/kimi-k2-thinking (или любой префикс провайдер/модель)
```

Если ваш редактор не может отправлять `Authorization: Bearer ...`, используйте токенизированную совместимую базу:

```txt
Base URL: http://localhost:20128/api/v1/vscode/YOUR_KEY/
Models URL: http://localhost:20128/api/v1/vscode/YOUR_KEY/models
Chat URL: http://localhost:20128/api/v1/vscode/YOUR_KEY/chat/completions
Ollama Tags URL: http://localhost:20128/api/v1/vscode/YOUR_KEY/api/tags
```

Работает с Claude Code, Codex CLI, Cursor, Cline, OpenClaw, OpenCode и OpenAI-совместимыми SDK.

#### Автонастройка через `setup-*`

Вместо ручной вставки base URL и ключа позвольте OmniRoute записать
собственный конфиг каждого инструмента из живого каталога моделей. Одна команда на инструмент:

```bash
omniroute setup-codex        # профили ~/.codex/<name>.config.toml
omniroute setup-claude       # ~/.claude/profiles/<name>/settings.json
omniroute setup-opencode     # ~/.config/opencode/opencode.json (openai-совместимый)
omniroute setup-cline        # Cline CLI + настройки расширения VS Code
omniroute setup-kilo         # Kilo Code
omniroute setup-continue     # ~/.continue/config.yaml (Continue / cn)
omniroute setup-cursor       # печатает шаги внутри приложения Cursor
omniroute setup-roo          # импорт Roo Code + указатель autoImport
omniroute setup-crush        # ~/.config/crush/crush.json
omniroute setup-goose        # ~/.config/goose/config.yaml
omniroute setup-qwen         # ~/.qwen/settings.json
omniroute setup-aider        # ~/.aider.conf.yml
```

Каждая принимает `--remote <url> --api-key <key>` для настройки локального инструмента против
**удалённого** OmniRoute, плюс `--dry-run` для предпросмотра. Лаунчеры
`omniroute launch` (Claude Code) и `omniroute launch-codex` (Codex) запускают CLI
с нужными переменными окружения, вообще не записывая конфиг.

Полную таблицу (что пишет каждая команда, все флаги, локально vs удалённо, соглашения
о `/v1` в base URL) см. в **[CLI-интеграции](./CLI-INTEGRATIONS.md)**.

Подробную настройку каждого инструмента (Claude Code, Codex CLI, Cursor, Cline, OpenClaw, Kilo Code, Copilot и других) см. в выделенном **[Руководстве по CLI-инструментам](../reference/CLI-TOOLS.md)**.

---

## Настройка протоколов (MCP + A2A)

### Настройка MCP (Model Context Protocol)

Запустите транспорт MCP в режиме stdio:

```bash
omniroute --mcp
```

Рекомендуемый процесс валидации:

```bash
# 1. Запустите MCP-сервер
omniroute --mcp

# 2. Из вашего MCP-клиента вызовите:
omniroute_get_health        # Должен вернуть здоровье системы
omniroute_list_combos       # Должен вернуть активные комбо

# 3. Или запустите полный E2E-набор:
npm run test:protocols:e2e
```

#### Настройка MCP-клиента

**Claude Code:**

```bash
claude mcp add-server omniroute --type http --url http://localhost:20128/api/mcp/stream
```

**Cursor / Cline:**

Добавьте в ваши настройки MCP:

```json
{
  "mcpServers": {
    "omniroute": {
      "command": "omniroute",
      "args": ["--mcp"],
      "env": {}
    }
  }
}
```

**Полная документация MCP:** [README MCP-сервера](../../open-sse/mcp-server/README.md) — 87 инструментов, конфиги IDE, клиенты Python/TS/Go.

### Настройка A2A (протокол Agent-to-Agent)

Проверьте Agent Card:

```bash
curl http://localhost:20128/.well-known/agent.json
```

Отправьте задачу:

```bash
curl -X POST http://localhost:20128/a2a \
  -H 'content-type: application/json' \
  -d '{"jsonrpc":"2.0","id":"quickstart","method":"message/send","params":{"skill":"quota-management","messages":[{"role":"user","content":"Give me a short quota summary."}]}}'
```

**Полная документация A2A:** [README A2A-сервера](../../src/lib/a2a/README.md) — JSON-RPC 2.0, навыки, стриминг, жизненный цикл задач.

---

## Настройка таймаутов

### Базовые таймауты

Для большинства развёртываний вам нужны только эти две переменные:

| Переменная               | По умолчанию                  | Назначение                                                                                                                                     |
| ------------------------ | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `REQUEST_TIMEOUT_MS`     | `600000`                      | Общий базовый уровень для таймаута начала ответа апстрима, скрытых таймаутов Undici, запросов отпечатка TLS и таймаутов запроса/прокси API-моста |
| `STREAM_IDLE_TIMEOUT_MS` | наследует `REQUEST_TIMEOUT_MS` | Максимальный промежуток между потоковыми чанками, после которого OmniRoute прерывает SSE-поток                                                |

Обратная совместимость сохранена: существующие `FETCH_TIMEOUT_MS`, `API_BRIDGE_PROXY_TIMEOUT_MS` и другие послойные переменные таймаутов по-прежнему работают и переопределяют общий базовый уровень.

### Заметки по конкретным провайдерам

Для Claude Code-совместимых апстримов (`anthropic-compatible-cc-*`) OmniRoute выводит исходящий заголовок `X-Stainless-Timeout` из разрешённого таймаута fetch, чтобы таймауты чтения на стороне провайдера оставались согласованными с вашей конфигурацией окружения.

Для сторонних Claude Code-совместимых обратных прокси OmniRoute сохраняет набор `anthropic-beta` по умолчанию консервативным и, когда `Client Cache Control` оставлен в `Auto`, пересылает только предоставленные клиентом маркеры `cache_control`. Включайте переключатель «Enable redact-thinking beta» на уровне подключения только когда апстрим специально требует потоки Claude с отредактированным thinking.

### Расширенные переопределения таймаутов

| Переменная                               | По умолчанию                               | Назначение                                                           |
| ---------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------- |
| `FETCH_TIMEOUT_MS`                       | наследует `REQUEST_TIMEOUT_MS`             | Таймаут начала ответа апстрима, действующий до получения заголовков ответа |
| `FETCH_HEADERS_TIMEOUT_MS`               | наследует `FETCH_TIMEOUT_MS`               | Лимит времени Undici на получение заголовков ответа апстрима         |
| `FETCH_BODY_TIMEOUT_MS`                  | наследует `FETCH_TIMEOUT_MS`               | Лимит времени Undici между чанками тела апстрима (`0` отключает)     |
| `FETCH_CONNECT_TIMEOUT_MS`               | `30000`                                    | Таймаут TCP-подключения Undici                                       |
| `FETCH_KEEPALIVE_TIMEOUT_MS`             | `4000`                                     | Таймаут простаивающего keep-alive сокета Undici                      |
| `TLS_CLIENT_TIMEOUT_MS`                  | наследует `FETCH_TIMEOUT_MS`               | Таймаут запросов отпечатка TLS через `wreq-js`                       |
| `API_BRIDGE_PROXY_TIMEOUT_MS`            | наследует `REQUEST_TIMEOUT_MS` или `600000` | Таймаут прокси-пересылки `/v1` с порта API на порт панели            |
| `API_BRIDGE_SERVER_REQUEST_TIMEOUT_MS`   | `max(API_BRIDGE_PROXY_TIMEOUT_MS, 300000)` | Таймаут входящего запроса на сервере API-моста                       |
| `API_BRIDGE_SERVER_HEADERS_TIMEOUT_MS`   | `60000`                                    | Таймаут входящих заголовков на сервере API-моста                     |
| `API_BRIDGE_SERVER_KEEPALIVE_TIMEOUT_MS` | `5000`                                     | Таймаут keep-alive на сервере API-моста                              |
| `API_BRIDGE_SERVER_SOCKET_TIMEOUT_MS`    | `0`                                        | Таймаут неактивности сокета на сервере API-моста (`0` отключает)     |

> **Примечание:** для потоковых запросов `FETCH_TIMEOUT_MS` покрывает только установление соединения / ожидание первого ответа апстрима. Как только поток активен, OmniRoute прервёт его только при реальной остановке (`STREAM_IDLE_TIMEOUT_MS`) или неактивности тела Undici (`FETCH_BODY_TIMEOUT_MS`).

### Совместимость с обратными прокси

Если вы запускаете OmniRoute за Nginx, Caddy, Cloudflare или другим обратным прокси, убедитесь, что таймауты прокси также больше ваших таймаутов потока/fetch OmniRoute.

---

## Режим раздельных портов

Запускайте API и панель управления на разных портах для продвинутых сценариев (обратный прокси, контейнерная сеть):

```bash
PORT=20128 DASHBOARD_PORT=20129 omniroute
# API:       http://localhost:20128/v1
# Dashboard: http://localhost:20129
```

---

## Шаблон для Void Linux (xbps-src)

Пользователи Void Linux могут собрать нативный пакет с помощью `xbps-src`. Сохраните этот блок как `srcpkgs/omniroute/template`:

```bash
# Файл шаблона для 'omniroute'
pkgname=omniroute
version=3.8.0
revision=1
hostmakedepends="nodejs python3 make"
depends="openssl"
short_desc="Universal AI gateway with smart routing for multiple LLM providers"
maintainer="zenobit <zenobit@disroot.org>"
license="MIT"
homepage="https://github.com/diegosouzapw/OmniRoute"
distfiles="https://github.com/diegosouzapw/OmniRoute/archive/refs/tags/v${version}.tar.gz"
# Перегенерируйте контрольную сумму для каждого релиза командой:
#   curl -L -o /tmp/omniroute.tar.gz "https://github.com/diegosouzapw/OmniRoute/archive/refs/tags/v${version}.tar.gz" && sha256sum /tmp/omniroute.tar.gz
checksum=PLACEHOLDER_REGENERATE_PER_RELEASE
system_accounts="_omniroute"
omniroute_homedir="/var/lib/omniroute"
export NODE_ENV=production
export npm_config_engine_strict=false
export npm_config_loglevel=error
export npm_config_fund=false
export npm_config_audit=false

do_build() {
	local _gyp_arch
	case "$XBPS_TARGET_MACHINE" in
		aarch64*) _gyp_arch=arm64 ;;
		armv7*|armv6*) _gyp_arch=arm ;;
		i686*) _gyp_arch=ia32 ;;
		*) _gyp_arch=x64 ;;
	esac

	NODE_ENV=development npm ci --ignore-scripts

	npm run build
	cp -r .next/static .next/standalone/.next/static
	[ -d public ] && cp -r public .next/standalone/public || true

	local _node_gyp=/usr/lib/node_modules/npm/node_modules/node-gyp/bin/node-gyp.js
	(cd node_modules/better-sqlite3 && node "$_node_gyp" rebuild --arch="$_gyp_arch")

	local _bs3_release=.next/standalone/node_modules/better-sqlite3/build/Release
	mkdir -p "$_bs3_release"
	cp node_modules/better-sqlite3/build/Release/better_sqlite3.node "$_bs3_release/"

	rm -rf .next/standalone/node_modules/@img

	for _mod in pino-abstract-transport split2 process-warning; do
		cp -r "node_modules/$_mod" .next/standalone/node_modules/
	done
}

do_check() {
	npm run test:unit
}

do_install() {
	vmkdir usr/lib/omniroute/.next
	vcopy .next/standalone/. usr/lib/omniroute/.next/standalone

	for _d in \
		.next/standalone/.next/server/app/dashboard \
		.next/standalone/.next/server/app/dashboard/settings \
		.next/standalone/.next/server/app/dashboard/providers; do
		touch "${DESTDIR}/usr/lib/omniroute/${_d}/.keep"
	done

	cat > "${WRKDIR}/omniroute" <<'EOF'
#!/bin/sh
export PORT="${PORT:-20128}"
export DATA_DIR="${DATA_DIR:-${XDG_DATA_HOME:-${HOME}/.local/share}/omniroute}"
export APP_LOG_TO_FILE="${APP_LOG_TO_FILE:-false}"
mkdir -p "${DATA_DIR}"
exec node /usr/lib/omniroute/.next/standalone/server.js "$@"
EOF
	vbin "${WRKDIR}/omniroute"
}

post_install() {
	vlicense LICENSE
}
```

---

## Удаление

| Команда                  | Действие                                                                                |
| ------------------------ | --------------------------------------------------------------------------------------- |
| `npm run uninstall`      | Удаляет системное приложение, но **сохраняет вашу БД и конфигурации** в `~/.omniroute`. |
| `npm run uninstall:full` | Удаляет приложение И безвозвратно **стирает все конфигурации, ключи и базы данных**.    |

> Подробные инструкции по удалению для всех способов см. в [UNINSTALL.md](./UNINSTALL.md).
