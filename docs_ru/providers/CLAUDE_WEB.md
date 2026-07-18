---
title: "Провайдеры — Claude Web"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Провайдеры — Claude Web

## claude-web

Провайдер на основе веб-cookie для **Claude AI** (`claude.ai`), использующий аутентификацию по сессионным cookie.

### Как это работает

1. Пользователь вставляет свои сессионные cookie с `claude.ai` в панель управления OmniRoute
2. `ClaudeWebExecutor` преобразует запросы формата OpenAI в формат Claude Web API
3. Запросы отправляются через **`tls-client-node`** с **TLS-отпечатком Chrome 124** для обхода Cloudflare Turnstile
4. Ответы стримятся обратно через SSE (`text/event-stream`)

### Необходимые cookie

| Cookie         | Назначение                     | Источник                               |
| -------------- | ------------------------------ | -------------------------------------- |
| `sessionKey`   | Основная аутентификация        | Сессия браузера `claude.ai`            |
| `routingHint`  | Маршрутизация Anthropic        | Сессия браузера `claude.ai`            |
| `cf_clearance` | Прохождение Cloudflare Turnstile | Автоматически устанавливается Cloudflare после проверки |
| `__cf_bm`      | Управление ботами Cloudflare   | Автоматически устанавливается Cloudflare |
| `_cfuvid`      | ID посетителя Cloudflare       | Автоматически устанавливается Cloudflare |

> **Примечание**: `cf_clearance` привязан к TLS-отпечатку браузера, который прошёл проверку Cloudflare Turnstile. Библиотека `tls-client-node` (через `claudeTlsClient.ts`) подделывает TLS-рукопожатие Chrome 124, чтобы токен clearance работал с сервера OmniRoute.

### Справочник API

**Эндпоинт**: `POST /api/organizations/{orgId}/chat_conversations/{convId}/completion`

**Обязательные заголовки**:

```
accept: text/event-stream
anthropic-client-platform: web_claude_ai
anthropic-device-id: <uuid>
content-type: application/json
Referer: https://claude.ai/chat/{convId}
```

**Тело запроса**:

```json
{
  "prompt": "user message",
  "model": "claude-sonnet-4-6",
  "timezone": "Asia/Jakarta",
  "locale": "en-US",
  "personalized_styles": [...],
  "tools": [...],
  "rendering_mode": "messages",
  "create_conversation_params": {
    "name": "",
    "model": "claude-sonnet-4-6",
    "is_temporary": false
  }
}
```

### Архитектура

```
Cookie пользователя (claude.ai)
    ↓
Панель управления OmniRoute
    ↓
ClaudeWebExecutor (open-sse/executors/claude-web.ts)
    ↓ Преобразование запроса (OpenAI → формат Claude Web)
    ↓
tlsFetchClaude() (open-sse/services/claudeTlsClient.ts)
    ↓ Подделка TLS-отпечатка Chrome 124
    ↓
tls-client-node (нативная привязка Go, koffi)
    ↓
API claude.ai
    ↓ SSE-поток
```

### Файлы

| Файл                                                  | Назначение                                           |
| ----------------------------------------------------- | ---------------------------------------------------- |
| `src/shared/constants/providers.ts`                   | Регистрация провайдера (WEB_COOKIE_PROVIDERS)        |
| `src/lib/providers/webCookieAuth.ts`                  | Утилиты для cookie (нормализация/извлечение сессионных cookie) |
| `open-sse/executors/claude-web.ts`                    | Реализация исполнителя                               |
| `open-sse/executors/index.ts`                         | Регистрация исполнителя                              |
| `open-sse/services/claudeTlsClient.ts`                | Подделка TLS-отпечатков через tls-client-node        |
| `open-sse/services/__tests__/claudeTlsClient.test.ts` | Тесты TLS-клиента                                    |
| `tests/unit/claude-web.test.ts`                       | Тесты исполнителя                                    |

### Тестирование

```bash
# Модульные тесты
node --import tsx/esm --test tests/unit/claude-web.test.ts

# Тесты TLS-клиента
npx vitest run open-sse/services/__tests__/claudeTlsClient.test.ts
```

### Настройка

1. Запустите OmniRoute: `omniroute`
2. Перейдите в Панель управления → Providers → Add Provider
3. Выберите категорию «Web Cookie»
4. Выберите «Claude Web»
5. Вставьте полный заголовок cookie из DevTools браузера на `claude.ai` (вкладка Network → Copy as fetch → заголовок Cookie)
