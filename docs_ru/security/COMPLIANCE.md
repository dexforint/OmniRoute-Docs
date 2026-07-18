---
title: "Соответствие требованиям и аудит"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Соответствие требованиям и аудит

> **Источник истины:** `src/lib/compliance/`, `src/app/api/compliance/`
> **Последнее обновление:** 2026-06-28 — v3.8.40

OmniRoute записывает административные действия, события аутентификации, изменения
жизненного цикла учётных данных провайдеров и вызовы инструментов MCP в таблицы аудита
на базе SQLite. Эта страница описывает, что логируется, где хранится, как долго,
как API-ключи могут отказаться от логирования и как запрашивать данные.

Реализация находится в `src/lib/compliance/index.ts` (T-43 — «Комплаенс-контроль»)
и `src/lib/compliance/providerAudit.ts`. Записи аудита никогда не выбрасывают исключений:
при любой ошибке вызов молча проглатывается, чтобы логирование аудита не могло сломать
основной поток запроса.

## Что логируется

### Административные события аудита (`audit_log`)

Каждый вызов `logAuditEvent({ action, actor, target, details, ... })` создаёт
одну строку. Строки действий следуют паттерну `domain.verb` (или `domain.verb.outcome`).
Подтверждённые типы действий в дереве исходников:

| Действие                             | Источник                                |
| ------------------------------------ | --------------------------------------- |
| `auth.login.success`                 | `src/app/api/auth/login/route.ts`       |
| `auth.login.failed`                  | `src/app/api/auth/login/route.ts`       |
| `auth.login.locked`                  | `src/app/api/auth/login/route.ts`       |
| `auth.login.error`                   | `src/app/api/auth/login/route.ts`       |
| `auth.login.misconfigured`           | `src/app/api/auth/login/route.ts`       |
| `auth.login.setup_required`          | `src/app/api/auth/login/route.ts`       |
| `auth.logout.success`                | `src/app/api/auth/logout/route.ts`      |
| `provider.credentials.created`       | `src/app/api/providers/route.ts`        |
| `provider.credentials.updated`       | `src/app/api/providers/[id]/route.ts`   |
| `provider.credentials.revoked`       | `src/app/api/providers/[id]/route.ts`   |
| `provider.credentials.batch_revoked` | `src/app/api/providers/route.ts`        |
| `sync.token.created`                 | `src/app/api/sync/tokens/route.ts`      |
| `sync.token.revoked`                 | `src/app/api/sync/tokens/[id]/route.ts` |
| `compliance.cleanup`                 | `src/lib/compliance/index.ts`           |

Каждая запись содержит `action`, `actor` (по умолчанию `"system"`), `target`,
`details`/`metadata` (JSON), `ip_address`, `resource_type`, `status`,
`request_id` и `timestamp`. Чувствительные ключи (`apiKey`, `accessToken`,
`refreshToken`, `password`, всё совпадающее с `*token`/`*secret`/`*apikey` и т.д.)
рекурсивно редактируются в `"[redacted]"` до записи строки.

### Вызовы инструментов MCP (`mcp_tool_audit`)

Каждый вызов инструмента MCP пишет строку через
`open-sse/mcp-server/audit.ts`. Схема (из
`src/lib/db/migrations/002_mcp_a2a_tables.sql`):

| Колонка          | Примечания                          |
| ---------------- | ----------------------------------- |
| `id`             | автоинкремент                       |
| `tool_name`      | идентификатор инструмента MCP       |
| `input_hash`     | sha256 входа (полезная нагрузка не хранится) |
| `output_summary` | короткая, усечённая сводка          |
| `duration_ms`    | время по стене                      |
| `api_key_id`     | вызывающий (nullable)               |
| `success`        | `1` / `0`                           |
| `error_code`     | терминальный код ошибки при сбое    |
| `created_at`     | метка времени ISO                   |

### Логи запросов / использования

Это операционная телеметрия (не строго административный аудит), но она использует
тот же конвейер удержания:

- `usage_history` — сводка использования по запросам
- `call_logs` — полный лог каждого запроса (с ограничением по строкам, см. ниже)
- `proxy_logs` — лог трафика прокси (с ограничением по строкам)
- `request_detail_logs` — устаревший подробный лог запросов (всё ещё очищается, если присутствует)

## Схема хранилища

`audit_log` создаётся лениво через `ensureAuditLogSchema()` при первом использовании:

```sql
CREATE TABLE IF NOT EXISTS audit_log (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp     TEXT NOT NULL DEFAULT (datetime('now')),
  action        TEXT NOT NULL,
  actor         TEXT NOT NULL DEFAULT 'system',
  target        TEXT,
  details       TEXT,
  ip_address    TEXT,
  resource_type TEXT,
  status        TEXT,
  request_id    TEXT,
  metadata      TEXT
);
```

Индексы создаются по `timestamp`, `action`, `actor`, `resource_type`,
`status` и `request_id`. Недостающие колонки в старых БД добавляются через
`ALTER TABLE` по требованию.

## Удержание и очистка

Соблюдаются два отдельных окна удержания:

| Переменная                  | По умолчанию | Применяется к                                                     |
| --------------------------- | ------------ | ----------------------------------------------------------------- |
| `APP_LOG_RETENTION_DAYS`    | `7`          | `audit_log`, `mcp_tool_audit`                                     |
| `CALL_LOG_RETENTION_DAYS`   | `7`          | `usage_history`, `call_logs`, `proxy_logs`, `request_detail_logs` |
| `CALL_LOGS_TABLE_MAX_ROWS`  | `100000`     | Обрезка по числу строк для `call_logs`                            |
| `PROXY_LOGS_TABLE_MAX_ROWS` | `100000`     | Обрезка по числу строк для `proxy_logs`                           |

`cleanupExpiredLogs()` выполняет проход удержания. Вызывается при запуске сервера
из `src/server-init.ts` и `src/instrumentation-node.ts`. Каждый запуск логирует
событие аудита `compliance.cleanup` с числом удалений по таблицам. Обрезка логов
прокси/вызовов пакетная (`BATCH_SIZE = 5000`), чтобы избежать длинных блокировок записи.

Ручная очистка истории запросов отделена от удержания. Страница Request Logs
вызывает `POST /api/settings/purge-request-history`, который удаляет `call_logs`,
устаревшие `request_detail_logs` и локальные артефакты запросов в
`${DATA_DIR}/call_logs/`.

Значения по умолчанию определены в `src/lib/logEnv.ts`
(`DEFAULT_APP_LOG_RETENTION_DAYS = 7`, `DEFAULT_CALL_LOG_RETENTION_DAYS = 7`).

## Отказ `noLog` (на API-ключ)

API-ключи можно пометить, чтобы их нижестоящий трафик вызовов не логировался.
Флаг живёт в таблице `api_keys` (`no_log INTEGER DEFAULT 0`) и зеркалируется
в набор в памяти для проверок на горячем пути.

```bash
# Создать no-log ключ (требуется управляющая аутентификация)
curl -X POST http://localhost:20128/api/keys \
  -H "Cookie: auth_token=..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Privacy key", "noLog": true}'
```

Помощники (`src/lib/compliance/index.ts`):

- `setNoLog(apiKeyId, true|false)` — переключить запись в памяти
- `isNoLog(apiKeyId)` — проверяется на пути запроса; откатывается на кэшированное
  (30 с) чтение из `api_keys.no_log`
- `NO_LOG_API_KEY_IDS` (env, через запятую) — предзагружается в набор в памяти
  при загрузке; полезно, когда нельзя переключить колонку напрямую

Административные события аудита (вход, изменения провайдеров, вызовы инструментов
MCP и т.д.) **не** затрагиваются флагом `noLog` — отключается только логирование
трафика по запросам.

## REST API

| Эндпоинт                    | Метод  | Описание                                    | Аутентификация |
| --------------------------- | ------ | ------------------------------------------- | -------------- |
| `/api/compliance/audit-log` | `GET`  | Пагинированные записи админ-аудита с фильтрами | management  |
| `/api/mcp/audit`            | `GET`  | Пагинированные записи аудита инструментов MCP | (open-sse)  |
| `/api/mcp/audit/stats`      | `GET`  | Агрегированная статистика аудита MCP          | (open-sse)  |

Эндпоинта экспорта CSV сегодня нет — экспортируйте из панели или запрашивайте
базу SQLite напрямую.

### Запросы к `/api/compliance/audit-log`

Поддерживаемые параметры запроса (все необязательные, текстовые фильтры используют
сопоставление `LIKE %value%`):

- `action`, `actor`, `target`, `resourceType` (или `resource_type`),
  `status`, `requestId` (или `request_id`)
- `from` / `since`, `to` / `until` — метки времени ISO
- `limit` (по умолчанию `50`, мин `1`, макс `500`)
- `offset` (по умолчанию `0`, макс `10_000`)

Ответ — JSON-массив. Метаданные пагинации возвращаются в заголовках:
`x-total-count`, `x-page-limit`, `x-page-offset`.

```bash
curl "http://localhost:20128/api/compliance/audit-log?action=provider.credentials&from=2026-05-01" \
  -H "Cookie: auth_token=..."
```

## Панель управления

Панель предоставляет данные аудита по адресу **`/dashboard/audit`**
(`src/app/(dashboard)/dashboard/audit/page.tsx`). На странице две вкладки:

- **Compliance** (`ComplianceTab.tsx`) — события админ-аудита из
  `/api/compliance/audit-log`. Фильтры по типу события, серьёзности (info / warning
  / critical, выводится из action + status) и диапазону дат. Серьёзность
  вычисляется на клиенте из строк action/status.
- **MCP** (`McpAuditTab.tsx`) — аудит инструментов MCP из `/api/mcp/audit`, с
  фильтрами по имени инструмента и успеху/сбою.

Обе вкладки пагинируют с размерами страниц `50` (compliance) и `25` (MCP).

## Помощники учётных данных провайдеров

`src/lib/compliance/providerAudit.ts` предоставляет помощники формирования данных,
используемые маршрутами управления провайдерами при испускании событий учётных данных:

- `summarizeProviderConnectionForAudit(connection)` — вырезает `apiKey`,
  `accessToken`, `refreshToken`, `idToken` и
  `providerSpecificData.consoleApiKey` перед тем, как снимок подключения
  записывается в `details`.
- `getProviderAuditTarget(connection)` — составляет стабильную строку
  `"<provider>:<name|id>"` для поля `target`.
- `extractProviderWarnings(...payloads)` — сканирует ответы провайдеров на
  предупреждения политики/безопасности (`[sanitizer]`, `prompt injection detected`,
  `content has been filtered`, `safety filter`, `policy violation`) и
  выдаёт до 5 попаданий, каждое усечено до 400 символов.

## Лучшие практики

- Помечайте API-ключи, обрабатывающие PII (юридические, медицинские данные и т.д.), флагом `noLog: true`.
- Настройте `APP_LOG_RETENTION_DAYS` / `CALL_LOG_RETENTION_DAYS` под свою политику
  удержания. Значения по умолчанию в 7 дней — консервативные.
- Экспортируйте таблицу аудита за пределы платформы (`sqlite3 dump`) с той периодичностью,
  которую требует ваша программа комплаенса — встроенной архивации нет.
- Отслеживайте счётчики `auth.login.failed` и `auth.login.locked` для обнаружения
  брутфорса.
- При добавлении новых админ-эндпоинтов вызывайте `logAuditEvent({ ... })` со стабильной
  строкой действия `domain.verb.outcome` и передавайте контекст запроса через
  `getAuditRequestContext(request)`, чтобы IP и `requestId` захватывались
  автоматически.

## См. также

- [`docs/security/GUARDRAILS.md`](./GUARDRAILS.md) — маскирование PII, инъекции промптов
- [`docs/frameworks/MCP-SERVER.md`](../frameworks/MCP-SERVER.md) — каталог инструментов MCP и области
- [`docs/reference/ENVIRONMENT.md`](../reference/ENVIRONMENT.md) — полный справочник переменных окружения
- Исходный код: `src/lib/compliance/`, `src/app/api/compliance/`,
  `src/app/api/mcp/audit/`, `src/lib/logEnv.ts`
