---
title: "🐳 Руководство по Docker — OmniRoute"
version: 3.8.40
lastUpdated: 2026-06-28
---

# 🐳 Руководство по Docker — OmniRoute

> Полный справочник по развёртыванию в Docker. Для быстрого старта см. [раздел Docker в README](../README.md#-docker).

## Содержание

- [Быстрый запуск](#quick-run)
- [С файлом окружения](#with-environment-file)
- [Docker Compose](#docker-compose)
- [Доступные профили](#available-profiles)
- [Sidecar Redis](#redis-sidecar)
- [Продакшен-Compose](#production-compose)
- [Стадии Dockerfile](#dockerfile-stages)
- [Критичные переменные окружения](#critical-environment-variables)
- [Docker Compose с Caddy (HTTPS)](#docker-compose-with-caddy-https-auto-tls)
- [Cloudflare Quick Tunnel](#cloudflare-quick-tunnel)
- [Теги образов](#image-tags)
- [Важные замечания](#important-notes)

---

## Быстрый запуск

```bash
docker run -d \
  --name omniroute \
  --restart unless-stopped \
  --stop-timeout 40 \
  -p 20128:20128 \
  -v omniroute-data:/app/data \
  diegosouzapw/omniroute:latest
```

## С файлом окружения

```bash
# Сначала скопируйте и отредактируйте .env
cp .env.example .env

docker run -d \
  --name omniroute \
  --restart unless-stopped \
  --stop-timeout 40 \
  --env-file .env \
  -p 20128:20128 \
  -v omniroute-data:/app/data \
  diegosouzapw/omniroute:latest
```

## Docker Compose

```bash
# Базовый профиль (без CLI-инструментов)
docker compose --profile base up -d

# Профиль CLI (встроенные Claude Code, Codex, OpenClaw)
docker compose --profile cli up -d

# Профиль host (в первую очередь Linux; монтирует бинарники CLI хоста read-only)
docker compose --profile host up -d

# Комбинация CLI + sidecar CLIProxyAPI
docker compose --profile cli --profile cliproxyapi up -d
```

## Доступные профили

OmniRoute поставляется с четырьмя профилями Compose. Выберите тот, что соответствует вашей среде.

| Профиль          | Сервис           | Когда использовать                                                                                                                  | Команда                                      |
| ---------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `base` (по умолч.) | `omniroute-base` | Headless-сервер / минимальная среда выполнения, без встроенных CLI провайдеров                                                    | `docker compose --profile base up -d`        |
| `cli`            | `omniroute-cli`  | Агентные процессы, вызывающие `omniroute providers/setup/doctor` и встроенные CLI (Codex, Claude Code, Droid, OpenClaw)             | `docker compose --profile cli up -d`         |
| `host`           | `omniroute-host` | Linux-хосты, которым нужен доступ в стиле `network_mode` к CLI хоста через монтирование `~/.local/bin`, `~/.codex`, `~/.claude` и т.д. read-only | `docker compose --profile host up -d`        |
| `cliproxyapi`    | `cliproxyapi`    | Запуск sidecar [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) на порту `8317` для проксирования апстрим-CLI            | `docker compose --profile cliproxyapi up -d` |

> Несколько профилей можно комбинировать: `docker compose --profile cli --profile cliproxyapi up -d`.

## Sidecar Redis

OmniRoute использует Redis для распределённого ограничителя частоты и общего кэша. Сервис `redis` **всегда определён** в `docker-compose.yml` (у него нет ограничения по профилю) и запускается вместе с любым другим профилем.

| Деталь               | Значение                          |
| -------------------- | --------------------------------- |
| Образ                | `redis:7-alpine`                  |
| Имя контейнера       | `omniroute-redis`                 |
| Внутренний порт      | `6379`                            |
| Порт хоста (переопр.) | `REDIS_PORT` (по умолч. `6379`)  |
| Том                  | `omniroute-redis-data` → `/data`  |
| Проверка здоровья    | `redis-cli ping` (интервал 10 с)  |

Связанные переменные окружения:

- `REDIS_URL` — строка подключения, внедряемая в приложение (по умолчанию `redis://redis:6379`).
- `REDIS_PORT` — проброс порта Redis на стороне хоста.

**Отключать Redis не рекомендуется** (ограничитель частоты деградирует до резерва в памяти). Если необходимо, удалите/закомментируйте блок сервиса `redis:` в `docker-compose.yml` или масштабируйте его до нуля:

```bash
docker compose up -d --scale redis=0
```

## Продакшен-Compose

Для изолированного продакшен-снимка, работающего параллельно с dev, используйте `docker-compose.prod.yml`.

| Деталь                 | Значение                                                                           |
| ---------------------- | ---------------------------------------------------------------------------------- |
| Файл                   | `docker-compose.prod.yml`                                                          |
| Порт панели по умолч.  | `PROD_DASHBOARD_PORT=20130` (проброшен на внутренний `${DASHBOARD_PORT:-20128}`)   |
| Порт API по умолчанию  | `PROD_API_PORT=20131`                                                              |
| Образ                  | `omniroute:prod` (собран из цели `runner-cli`)                                     |
| Контейнер Redis        | `omniroute-redis-prod` (`redis:8.6.2`, выделенный том `redis-prod-data`)           |
| Том данных             | `omniroute-prod-data` (именованный, сохраняется между пересборками)                |
| Проверки здоровья      | `node healthcheck.mjs` + `redis-cli ping`, с `depends_on`, завязанным на здоровье Redis |

Как использовать:

```bash
# Собрать и запустить продакшен-стек
docker compose -f docker-compose.prod.yml up -d --build

# Следить за логами
docker compose -f docker-compose.prod.yml logs -f

# Остановить (тома сохраняются)
docker compose -f docker-compose.prod.yml down
```

Продакшен-стек работает параллельно с dev-compose (разные имена контейнеров, порты и тома), так что вы можете продолжать локальную разработку, пока продакшен работает.

## Стадии Dockerfile

Репозиторий поставляет многостадийный Dockerfile (`Dockerfile`). Доступны три стадии; выберите подходящую `target` для вашей задачи.

| Стадия        | Базовый образ              | Назначение                                                                                                                                                          |
| ------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `builder`     | `node:24.15.0-trixie-slim` | Устанавливает зависимости (`npm ci --legacy-peer-deps`) и выполняет `npm run build -- --webpack`                                                                    |
| `runner-base` | `node:24.15.0-trixie-slim` | Продакшен-среда выполнения с standalone-сборкой Next.js. **Без встроенных CLI провайдеров.**                                                                        |
| `runner-cli`  | `runner-base`              | Добавляет `git`, `docker.io`, `docker-compose` и глобальные CLI: `@openai/codex`, `@anthropic-ai/claude-code`, `droid`, `openclaw`. **Выбирайте для агентных задач.** |

Ручная сборка конкретной цели:

```bash
docker build --target runner-base -t omniroute:base .
docker build --target runner-cli  -t omniroute:cli  .
```

Значения по умолчанию, экспортируемые `runner-base`: `PORT=20128`, `HOSTNAME=0.0.0.0`, `NODE_OPTIONS=--max-old-space-size=512`, `DATA_DIR=/app/data`, `OMNIROUTE_MIGRATIONS_DIR=/app/migrations`.

Поведение памяти в Docker:

- `NODE_OPTIONS=--max-old-space-size=512` вшит в образ как резервное значение.
- Реальный серверный процесс запускается standalone-лаунчером, который читает `OMNIROUTE_MEMORY_MB` и дописывает `--max-old-space-size=<OMNIROUTE_MEMORY_MB>`.
- Node использует последнее повторное значение `--max-old-space-size`, поэтому установка `OMNIROUTE_MEMORY_MB` управляет эффективным лимитом кучи в Docker.
- Если `OMNIROUTE_MEMORY_MB` не задан, лаунчер использует `512`.

## Критичные переменные окружения

Помимо значений по умолчанию, задокументированных в [ENVIRONMENT.md](../reference/ENVIRONMENT.md), при запуске в Docker наиболее важны следующие переменные:

| Переменная                    | Назначение                                                                                          | По умолчанию             |
| ----------------------------- | --------------------------------------------------------------------------------------------------- | ------------------------ |
| `OMNIROUTE_WS_BRIDGE_SECRET`  | Общий секрет WebSocket-моста. **Обязателен в продакшене** — задайте стойкую случайную строку.       | не задан (нужно указать) |
| `REDIS_URL`                   | Строка подключения для бэкенда ограничителя частоты / кэша                                          | `redis://redis:6379`     |
| `REDIS_PORT`                  | Порт хоста для встроенного контейнера Redis                                                         | `6379`                   |
| `AUTO_UPDATE_HOST_REPO_DIR`   | Путь хоста, монтируемый в профиль `cli` как `/workspace/omniroute` для процессов самообновления     | `.` (текущий каталог)    |
| `OMNIROUTE_MEMORY_MB`         | Предел кучи Node для standalone-сервера в Docker; переопределяет резервное значение образа          | `512`                    |
| `DASHBOARD_PORT` / `API_PORT` | Переопределение открываемых портов панели (20128) и API (20129)                                     | `20128` / `20129`        |
| `PROD_DASHBOARD_PORT`         | Порт панели на стороне хоста для `docker-compose.prod.yml`                                          | `20130`                  |
| `CLIPROXYAPI_PORT`            | Порт хоста для sidecar `cliproxyapi`                                                                | `8317`                   |

## Docker Compose с Caddy (HTTPS Auto-TLS)

OmniRoute можно безопасно открыть наружу, используя автоматическую выдачу SSL от Caddy. Убедитесь, что DNS A-запись вашего домена указывает на IP вашего сервера.

```yaml
services:
  omniroute:
    image: diegosouzapw/omniroute:latest
    container_name: omniroute
    restart: unless-stopped
    volumes:
      - omniroute-data:/app/data
    environment:
      - PORT=20128
      # Обращённый к браузеру origin для OAuth callback, ссылок панели и генерируемых публичных URL.
      - NEXT_PUBLIC_BASE_URL=https://your-domain.com
      # Внутренний URL сервер-сервер для запланированных задач / самозапросов.
      - BASE_URL=http://omniroute:20128
      - AUTH_COOKIE_SECURE=true

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    command: caddy reverse-proxy --from https://your-domain.com --to http://omniroute:20128

volumes:
  omniroute-data:
```

Caddy выставляет стандартные заголовки пересылки для апстрим-контейнера. OmniRoute использует
`NEXT_PUBLIC_BASE_URL` как канонический публичный origin для OAuth callback и генерируемых публичных
ссылок; аутентифицированные записи панели используют запросы same-origin плюс CSRF-защиту, привязанную к сессии.
Включайте `OMNIROUTE_TRUST_PROXY` только для продвинутых развёртываний, где вы намеренно
хотите, чтобы OmniRoute выводил публичный origin из доверенных заголовков пересылки вместо явной
конфигурации.

## Cloudflare Quick Tunnel

Поддержка Docker-развёртываний в панели включает **Cloudflare Quick Tunnel** в один клик на `Dashboard → Endpoints`. Первое включение скачивает `cloudflared` только при необходимости, запускает временный туннель к вашему текущему эндпоинту `/v1` и показывает сгенерированный URL `https://*.trycloudflare.com/v1` прямо под вашим обычным публичным URL.

Панели туннелей эндпоинтов (Cloudflare, Tailscale, ngrok) можно показывать или скрывать в `Settings → Appearance` без изменения активного состояния туннелей.

### Замечания о туннелях

- URL Quick Tunnel временные и меняются после каждого перезапуска.
- Quick Tunnel не восстанавливаются автоматически после перезапуска OmniRoute или контейнера. Включайте их заново из панели управления по мере необходимости.
- Управляемая установка в настоящее время поддерживает Linux, macOS и Windows на `x64` / `arm64`.
- Управляемые Quick Tunnel по умолчанию используют транспорт HTTP/2, чтобы избежать шумных предупреждений о буфере QUIC UDP в ограниченных контейнерных средах. Задайте `CLOUDFLARED_PROTOCOL=quic` или `auto`, если хотите другой транспорт.
- Образы Docker содержат системные корневые CA и передают их управляемому `cloudflared`, что избавляет от сбоев доверия TLS при поднятии туннеля внутри контейнера.
- Задайте `CLOUDFLARED_BIN=/absolute/path/to/cloudflared`, если хотите, чтобы OmniRoute использовал существующий бинарник вместо скачивания.

## Теги образов

| Образ                    | Тег      | Размер | Описание                |
| ------------------------ | -------- | ------ | ----------------------- |
| `diegosouzapw/omniroute` | `latest` | ~250MB | Последний стабильный релиз |
| `diegosouzapw/omniroute` | `3.8.0`  | ~250MB | Текущая версия          |

Мультиплатформенный манифест: `linux/amd64` + `linux/arm64` нативно (Apple Silicon, AWS Graviton, Raspberry Pi). Docker выбирает соответствующую архитектуру автоматически; передайте `--platform linux/amd64`, если нужно принудительно использовать эмуляцию AMD64 на ARM-хостах.

## Важные замечания

- **Режим SQLite WAL:** `docker stop` должен завершиться корректно, чтобы OmniRoute успел сделать checkpoint последних изменений обратно в `storage.sqlite`. Прилагаемые Compose-файлы уже задают 40-секундный период остановки. Если вы запускаете образ напрямую, сохраните `--stop-timeout 40`.
- **`DISABLE_SQLITE_AUTO_BACKUP`:** установите в `true`, если резервные копии управляются извне.
- **Сохранность данных:** всегда монтируйте том к `/app/data`, чтобы сохранить базу данных, ключи и конфигурации между перезапусками контейнера.
- **Настройка портов:** переопределите переменную окружения `PORT`, чтобы изменить порт по умолчанию `20128`.

## См. также

- [Руководство по развёртыванию на ВМ](../ops/VM_DEPLOYMENT_GUIDE.md) — ВМ + nginx + Cloudflare
- [Руководство по развёртыванию на Fly.io](../ops/FLY_IO_DEPLOYMENT_GUIDE.md) — развёртывание на Fly.io
- [Конфигурация окружения](../reference/ENVIRONMENT.md) — полный справочник `.env`
