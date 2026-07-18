---
title: "Санитизация сообщений об ошибках"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Санитизация сообщений об ошибках

> **Источник истины:** `open-sse/utils/error.ts` — `sanitizeErrorMessage`, `buildErrorBody`, `createErrorResult`
> **Тесты:** `tests/unit/error-message-sanitization.test.ts`
> **Последнее обновление:** 2026-06-28 — v3.8.40
> **Аудитория:** любой инженер, работающий с ответами об ошибках (HTTP-маршруты, SSE-потоки, исполнители, обработчики MCP).
> **Статус:** **ОБЯЗАТЕЛЬНО** для каждого пути кода, возвращающего сообщение об ошибке клиенту.

## Зачем это существует

Правило CodeQL `js/stack-trace-exposure` (CWE-209) помечает любой путь кода, где сообщение об ошибке, происходящее из исключения времени выполнения, попадает в HTTP / SSE-ответ без санитизации. Трассировки стека и абсолютные пути файлов в продакшн-ответах дают атакующим:

- Внутреннюю структуру каталогов (`/srv/app/src/lib/...`) → разведку для дальнейших атак.
- Версии библиотек / фреймворков, выводимые из кадров стека → целевой подбор эксплоитов.
- Чувствительные значения времени выполнения, которые могут интерполироваться в ошибки (запросы к БД, значения конфигурации).

Помощник `sanitizeErrorMessage` в `open-sse/utils/error.ts` устраняет оба класса утечек:

1. Многострочные трассировки стека — остаётся только первая строка (само сообщение об ошибке).
2. Абсолютные пути (`/...*.{ts,js,tsx,jsx,mjs,cjs}[:line[:col]]` и `C:\...`) — заменяются на `<path>`.

## Обязательный паттерн

### 1. Построение ответа об ошибке (HTTP / API-маршруты)

Используйте `buildErrorBody()` — санитизация встроена:

```ts
import { buildErrorBody } from "@omniroute/open-sse/utils/error.ts";

export async function POST(req: Request) {
  try {
    // ... логика обработчика ...
  } catch (err) {
    return new Response(JSON.stringify(buildErrorBody(500, String(err))), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
}
```

Или, для удобных обёрток из того же модуля:

```ts
import {
  errorResponse, // готовый объект Response
  writeStreamError, // писатель SSE
  createErrorResult, // форма { success: false, status, response, ... }
  unavailableResponse, // добавляет Retry-After
  providerCircuitOpenResponse,
  modelCooldownResponse,
} from "@omniroute/open-sse/utils/error.ts";
```

Все они проходят через `buildErrorBody`, а значит — через `sanitizeErrorMessage`. **Вам никогда не нужно вызывать `sanitizeErrorMessage` вручную** при использовании этих помощников.

### 2. Пользовательские обёртки ошибок (редко)

Когда помощники выше использовать нельзя (например, форма ответа продиктована апстрим-протоколом вроде Connect-RPC), импортируйте `sanitizeErrorMessage` напрямую:

```ts
import { sanitizeErrorMessage } from "@omniroute/open-sse/utils/error.ts";

const body = JSON.stringify({
  error: {
    message: sanitizeErrorMessage(rawMessage),
    type: "invalid_request_error",
    code: "",
  },
});
```

Это единственный санкционированный способ собрать пользовательское тело ошибки. Эталонная реализация — `open-sse/executors/cursor.ts::buildErrorResponse`.

### 3. Логирование vs. ответ

`sanitizeErrorMessage` должен **только** оборачивать значение, пересекающее сетевую границу. Внутренние логи (`pino`, `console`) должны сохранять полное сообщение, включая стек, чтобы операторы могли отлаживать. Паттерн:

```ts
try {
  // ...
} catch (err) {
  log.error({ err }, "handler failed"); // полный err со стеком — внутренний лог
  return errorResponse(500, getErrorMessage(err)); // санитизировано — уходит клиенту
}
```

### 4. Запрещённые паттерны

❌ **Никогда** не помещайте сырой вывод исключения в тело Response:

```ts
// ПЛОХО: трассировка стека + пути файлов достигают клиента
return new Response(JSON.stringify({ error: { message: err.stack || err.message } }), {
  status: 500,
});
```

❌ **Никогда** не пишите собственный разделитель первой строки:

```ts
// ПЛОХО: забывает вырезать абсолютные пути, может разойтись с каноническим помощником
const safe = String(err).split("\n")[0];
```

❌ **Никогда** не санитизируйте в маршруте, забыв путь SSE. Всё, что пишет в поток, проходит через `writeStreamError` (или его нижележащий `buildErrorBody`).

❌ **Никогда** не включайте `process.cwd()`, `__filename`, `__dirname`, пути из переменных окружения в сообщения об ошибках — они обходят регулярное выражение путей и раскрывают топологию развёртывания.

## Покрытие в CI

`tests/unit/error-message-sanitization.test.ts` обеспечивает:

- Каждый маршрут под `/api/model-combo-mappings/*` возвращает санитизированные тела на 4xx/5xx.
- `sanitizeErrorMessage` вырезает многострочные трассировки стека.
- `sanitizeErrorMessage` заменяет абсолютные пути POSIX и Windows на `<path>`.
- `sanitizeErrorMessage` безопасно обрабатывает входы `null`/`undefined`/экземпляр `Error`.
- `buildErrorBody` никогда не раскрывает трассировки стека в поле `message`.

При добавлении нового маршрута или исполнителя копируйте паттерн проверок из этого файла. Контрольная точка покрытия (`npm run test:coverage`) требует ≥60% операторов/строк/функций/ветвей — пути ошибок должны быть покрыты.

## Связанные механизмы контроля

- Оповещения CodeQL `js/stack-trace-exposure` в `.github/security` всегда должны **либо** исправляться через эти помощники, **либо** отклоняться с комментарием, ссылающимся на этот документ.
- Конфигурация редактирования `pino` (`src/shared/utils/logRedaction.ts`) отдельно обрабатывает редактирование структурированных логов. Этот документ покрывает только поверхность сообщений в ответах.
- Список запрещённых апстрим-заголовков (`src/shared/constants/upstreamHeaders.ts`) покрывает утечки через заголовки — держите оба файла синхронизированными при добавлении нового вектора эксфильтрации.

## Пропускание апстрим-деталей

`buildErrorBody` принимает необязательный третий аргумент `upstreamDetails` (сырое
распарсенное тело от апстрим-провайдера). Когда он передан, он санитизируется
`sanitizeUpstreamDetails` перед включением в ответ как `upstream_details`.

Правила санитизации, применяемые к `upstreamDetails`:

1. Строковые листья: проходят через `sanitizeErrorMessage` (вырезает стеки + абсолютные пути).
2. Блок-лист ключей: ключи, совпадающие с `/stack|trace|path|file|cwd|dir|password|secret|token|key/i`,
   удаляются.
3. Ограничение глубины: вложенность глубже 4 уровней заменяется строкой `"[truncated]"`.
4. Массивы ограничены 32 элементами.

Только семь мест вызова `createErrorResult` для апстрим-ошибок в `chatCore.ts` передают
`upstreamErrorBody`. Внутренние ошибки OmniRoute (сбои парсинга SSE, пустое содержимое,
блокировки guardrail) не включают `upstream_details`.

НЕ передавайте сырые `err.stack`, `err.message` или любую строку из исключения времени
выполнения в `upstreamDetails`. Они по-прежнему должны проходить через `errorResponse` /
`buildErrorBody(code, msg)` без апстрим-тела.

## Известное ограничение CodeQL: пользовательские санитизеры не распознаются

Запрос CodeQL [`js/stack-trace-exposure`](https://codeql.github.com/codeql-query-help/javascript/js-stack-trace-exposure/) использует фиксированный белый список паттернов санитизаторов (например, встроенный `.split("\n")[0]`, `String#replace` со специфическими формами регулярных выражений, доступ к `.message` на `Error`). Он **не** распознаёт косвенность через пользовательский помощник вроде нашего `sanitizeErrorMessage()`.

Это означает, что места вызовов, которые заведомо санитизируют через этот модуль — например, `open-sse/utils/error.ts::errorResponse` и `open-sse/executors/cursor.ts::buildErrorResponse` — могут продолжать вызывать оповещение, хотя код функционально безопасен. Прецеденты отклонений: `#224`, `#231` (май 2026), оба помечены `false positive` с техническим обоснованием.

**Как обрабатывать новое срабатывание:**

1. Убедитесь, что место вызова действительно пропускает сообщение через `sanitizeErrorMessage` / `buildErrorBody` / одну из задокументированных выше обёрток (прочитайте цепочку вызовов целиком — не доверяйте комментарию).
2. Убедитесь, что `tests/unit/error-message-sanitization.test.ts` покрывает этот путь (или добавьте покрытие).
3. Отклоните оповещение через `gh api ... -X PATCH state=dismissed -f 'dismissed_reason=false positive'`, сославшись на этот документ.
4. **Не** «исправляйте» встраиванием `.split("\n")[0]` повсюду — помощник является единственным источником истины; дублирование паттерна ослабляет санитизатор (теряется очистка путей, ограничение длины, приведение типов) ради видимости удовлетворения сканера.

Принятие опциональных возможностей вроде [конфигурации пользовательских санитизаторов `@codeql/javascript-models`](https://codeql.github.com/docs/codeql-language-guides/customizing-library-models-for-javascript/) — долгосрочное решение; оно находится за рамками этого документа.

## Ссылки

- [CWE-209: раскрытие информации через сообщение об ошибке](https://cwe.mitre.org/data/definitions/209.html)
- [CodeQL `js/stack-trace-exposure`](https://codeql.github.com/codeql-query-help/javascript/js-stack-trace-exposure/)
- [OWASP: шпаргалка по обработке ошибок](https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html)
- Коммит, централизовавший помощник: `1a39c31f` — _fix(security): mask public upstream creds + centralize error sanitization_
