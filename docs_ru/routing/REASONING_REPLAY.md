---
title: "Кэш повторной передачи рассуждений (Reasoning Replay)"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Кэш повторной передачи рассуждений (Reasoning Replay Cache)

> **Источник истины:** `src/lib/db/reasoningCache.ts`, `open-sse/services/reasoningCache.ts`
> **Последнее обновление:** 2026-06-28 — v3.8.40

OmniRoute захватывает `reasoning_content`, формируемый моделями с режимом «мышления», и прозрачно подставляет его обратно при многошаговых запросах, если апстрим-провайдер этого требует. Это устраняет ошибки HTTP 400, которые строгие провайдеры выдают, когда в истории диалога клиента отсутствует рассуждение предыдущего шага.

## Зачем это существует

Некоторые провайдеры с режимом «мышления» отклоняют следующий шаг диалога, если **предыдущее сообщение ассистента не содержит исходный `reasoning_content`**. Апстрим возвращает 400 с сообщениями вида:

```
Param Incorrect: The reasoning_content in the thinking mode must be passed back to the API.
```

Но типичные клиенты (Cursor, Cline, Roo Code, OpenAI SDK) вырезают `reasoning_content` из истории, которую отправляют обратно. OmniRoute восстанавливает его из серверного кэша, чтобы запрос, который видит апстрим, был согласованным. Issue #1628 добавил гибридное хранилище (память/SQLite), чтобы кэш переживал перезапуски процесса.

## Архитектура

```
Шаг N (ассистент генерирует ответ):
  → ответ содержит reasoning_content + tool_calls
  → cacheReasoningFromAssistantMessage() пишет (память + БД), ключ — каждый tool_call.id
  → ответ пересылается клиенту (который может сохранить рассуждение, а может и нет)

Шаг N+1 (клиент отправляет следующий запрос):
  → транслятор определяет: requiresReasoningReplay(provider, model) === true
  → для каждого сообщения ассистента с tool_calls и без reasoning_content:
      lookupReasoning(toolCalls[0].id) → память → БД
      найдено  → msg.reasoning_content = из кэша; recordReplay()
      не найдено → msg.reasoning_content = "" (устаревший фолбэк для старых DeepSeek)
  → апстрим видит согласованную историю → нет ошибки 400
```

Захват происходит в `open-sse/handlers/chatCore.ts` (два места, примерно строки 4093 и 4380). Повторная подстановка — в `open-sse/translator/index.ts`, после приведения схемы, но перед отправкой.

## Хранилище — гибрид «память + SQLite»

Горячий путь использует `Map` в памяти (LRU по времени создания), подкреплённый таблицей SQLite для восстановления после сбоев и отображения в панели управления.

| Слой   | Реализация                                     | Назначение                             |
| ------ | ---------------------------------------------- | -------------------------------------- |
| Память | `Map` в `open-sse/services/reasoningCache.ts`  | Быстрые обращения, вытеснение самых старых после 2000 |
| БД     | таблица `reasoning_cache` (`src/lib/db/`)      | Сохраняется между перезапусками, питает статистику |

Запись идёт в оба слоя. Чтение сначала проверяет память, затем БД (попадания в БД поднимаются обратно в память). Сбои БД не фатальны — кэш в памяти продолжает обслуживать горячий путь.

**Значения по умолчанию:**

- TTL: `2h` (`TTL_MS = 2 * 60 * 60 * 1000`)
- Максимум записей в памяти: `2000` (`MAX_MEMORY_ENTRIES`)
- Вытеснение: сначала самые старые по `createdAt`

## Схема базы данных

Миграция: `src/lib/db/migrations/033_create_reasoning_cache.sql`

```sql
CREATE TABLE IF NOT EXISTS reasoning_cache (
  tool_call_id   TEXT PRIMARY KEY,
  provider       TEXT NOT NULL,
  model          TEXT NOT NULL,
  reasoning      TEXT NOT NULL,
  char_count     INTEGER NOT NULL DEFAULT 0,
  created_at     TEXT NOT NULL DEFAULT (datetime('now')),
  expires_at     INTEGER NOT NULL
);
```

Индексы: `expires_at`, `provider`, `model`, `created_at`. `expires_at` хранится как Unix epoch в секундах; слой SELECT нормализует устаревшие текстовые значения через `EXPIRES_AT_EPOCH_SQL`.

## Определение провайдера / модели

Повторная передача включается, когда `requiresReasoningReplay(provider, model)` возвращает `true`. Функция проверяет два списка в `open-sse/services/reasoningCache.ts`.

**ID провайдеров (точное совпадение, без учёта регистра):**

- `deepseek`
- `opencode-go`
- `siliconflow`
- `nebius`
- `deepinfra`
- `sambanova`
- `fireworks`
- `together`
- `xiaomi-mimo`

**Регулярные выражения для моделей (без учёта регистра):**

- `/deepseek-r1/i`
- `/deepseek-reasoner/i`
- `/deepseek-chat/i`
- `/deepseek[-/]?v4[-.]flash/i` и `/deepseek[-/]?v4[-.]pro/i` (V4 Flash / Pro, необязательный суффикс `-free`)
- `/(deepseek|zen\/deepseek)-v4/i`
- `/kimi-k2/i`
- `/qwq/i`
- `/qwen.*think/i`
- `/glm.*think/i`
- `/^mimo[-.]?v\\d/i`

Чтобы добавить нового строгого провайдера/модель, дополните один из этих списков и напишите юнит-тест, проверяющий подстановку рассуждения. В описании PR следует указать точную строку ошибки 400 апстрима, вызвавшую изменение.

## REST API

Кэш предоставляет два эндпоинта в `src/app/api/cache/reasoning/route.ts`. Оба требуют аутентификации управления (`isAuthenticated` из `@/shared/utils/apiAuth`).

| Метод  | Эндпоинт                                                | Описание                                                  |
| ------ | --------------------------------------------------------- | -------------------------------------------------------- |
| GET    | `/api/cache/reasoning`                                    | Статистика + записи с пагинацией                          |
| GET    | `/api/cache/reasoning?provider=deepseek&model=...&limit=` | Фильтрованный список (`limit` ограничен диапазоном `[1, 200]`) |
| DELETE | `/api/cache/reasoning`                                    | Очистить всё (память + БД) и сбросить счётчики попаданий/промахов |
| DELETE | `/api/cache/reasoning?provider=deepseek`                  | Очистить записи только одного провайдера                  |
| DELETE | `/api/cache/reasoning?toolCallId=call_abc`                | Удалить одну запись                                       |

**Формат ответа GET:**

```json
{
  "stats": {
    "memoryEntries": 12,
    "dbEntries": 47,
    "totalEntries": 47,
    "totalChars": 138291,
    "hits": 84,
    "misses": 6,
    "replays": 81,
    "replayRate": "90.0%",
    "byProvider": { "deepseek": { "entries": 32, "chars": 98412 } },
    "byModel": { "deepseek-reasoner": { "entries": 32, "chars": 98412 } },
    "oldestEntry": "2026-05-13T10:00:00.000Z",
    "newestEntry": "2026-05-13T11:42:11.000Z"
  },
  "entries": [
    {
      "toolCallId": "call_abc",
      "provider": "deepseek",
      "model": "deepseek-reasoner",
      "reasoning": "...",
      "charCount": 3128,
      "createdAt": "...",
      "expiresAt": "..."
    }
  ]
}
```

## Эксплуатационные замечания

- **Очистка:** `cleanupReasoningCache()` удаляет истёкшие записи из памяти и выполняет `DELETE FROM reasoning_cache WHERE expires_at <= unixepoch('now')`. Воркеры проверки здоровья вызывают её периодически.
- **Восстановление после сбоя:** после перезапуска память пуста, но в БД остаются непросроченные записи. Первое обращение по данному `tool_call_id` идёт в БД; последующие — в память.
- **Нет рассуждения — нет кэша:** `cacheReasoningFromAssistantMessage` возвращает `0`, если в сообщении ассистента нет поля `reasoning_content` / `reasoning`, поэтому ответы без режима «мышления» ничего не стоят.
- **Нестрогие провайдеры:** когда `requiresReasoningReplay` равен `false` и целевой формат — OpenAI, транслятор **вырезает** поле `reasoning_content` из исходящих сообщений — OpenAI Chat Completions его не принимает.

## См. также

- [RESILIENCE_GUIDE.md](../architecture/RESILIENCE_GUIDE.md) — circuit breakers, кулдауны, блокировки моделей
- [TROUBLESHOOTING.md](../guides/TROUBLESHOOTING.md) — диагностика ошибок 400 апстрима
- Исходный код: `src/lib/db/reasoningCache.ts`, `open-sse/services/reasoningCache.ts`, `open-sse/translator/index.ts`
- Миграция: `src/lib/db/migrations/033_create_reasoning_cache.sql`
- API-маршрут: `src/app/api/cache/reasoning/route.ts`
- Исходный issue: #1628
