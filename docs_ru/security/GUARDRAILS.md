---
title: "Гардрейлы (Guardrails)"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Гардрейлы (Guardrails)

> **Источник истины:** `src/lib/guardrails/`
> **Последнее обновление:** 2026-06-28 — v3.8.40 (покрытие injection-guard + граница сканирования 16 КБ + red-team)

Гардрейлы обеспечивают соблюдение безопасности, политик и трансформаций содержимого на
границе между OmniRoute и апстрим-провайдерами. Каждый гардрейл может инспектировать (и
опционально отклонять, трансформировать или аннотировать) полезные нагрузки запросов
(`preCall`) и ответы апстрима (`postCall`).

Система **fail-open**: если гардрейл выбрасывает исключение при выполнении, реестр
записывает ошибку и продолжает со следующим гардрейлом вместо провала запроса.
Блокировка — это явное решение (`block: true`), никогда — случайность.

## Встроенные гардрейлы

Реестр автоматически загружает три гардрейла в порядке приоритета при импорте
(см. `registry.ts` → `registerDefaultGuardrails()`):

| Приоритет | Имя                | Стадия(и)      | Файл                 |
| --------- | ------------------ | -------------- | -------------------- |
| `5`       | `vision-bridge`    | `preCall`      | `visionBridge.ts`    |
| `10`      | `pii-masker`       | `pre` + `post` | `piiMasker.ts`       |
| `20`      | `prompt-injection` | `preCall`      | `promptInjection.ts` |

Меньшие номера приоритетов выполняются **первыми**.

### Vision Bridge (`visionBridge.ts`)

Перехватывает запросы с изображениями, направленные к **моделям без поддержки зрения**, и заменяет
части с изображениями текстовыми описаниями, произведёнными настраиваемой зрительной моделью,
до вызова апстрима. Это позволяет текстовым провайдерам прозрачно обрабатывать
мультимодальные полезные нагрузки.

Поток:

1. Пропустить, если целевая модель уже поддерживает зрение (если она не входит в
   список принудительного моста `isVisionBridgeForcedModel`).
2. Извлечь части с изображениями через `extractImageParts(messages)`. Пропустить, если их нет.
3. Загрузить конфигурацию времени выполнения из `getSettings()` (`visionBridgeEnabled`,
   `visionBridgeModel`, `visionBridgePrompt`, `visionBridgeTimeout`,
   `visionBridgeMaxImages`).
4. Ограничить изображения до `maxImages`, вызвать зрительную модель **параллельно**
   (`Promise.allSettled`) и вставить текстовые части `[Image N]: <description>`
   на их место — неудачные изображения становятся `[Image N]: (unavailable)`.
5. Вернуть `modifiedPayload` + мету (`imagesProcessed`, `processingTimeMs`,
   `visionModel`).

Значения по умолчанию находятся в `src/shared/constants/visionBridgeDefaults.ts`. Гардрейл
предоставляет опцию конструктора `deps`, чтобы тесты могли внедрять поддельные реализации
`getSettings` и `callVisionModel`.

### PII Masker (`piiMasker.ts`)

Выполняется на **обеих** стадиях.

- **`preCall`** клонирует полезную нагрузку, обходит массивы `system`, `messages` и
  `input` и применяет `processPII()` (из `@/shared/utils/inputSanitizer`) к
  строковым полям `content`/`text`. Когда `PII_REDACTION_ENABLED=true` **и**
  `INPUT_SANITIZER_MODE=redact`, обнаруженные PII вырезаются/редактируются в
  исходящей нагрузке. Иначе вызов записывает счётчики обнаружений без
  перезаписи содержимого.
- **`postCall`** глубоко клонирует ответ, запускает `sanitizePIIResponse()` плюс
  маскировщик формы Responses API (`maskResponsesOutput` — покрывает
  `output_text` и `output[].content[].text`). Если произошло редактирование,
  изменённый ответ заменяет оригинал.

Гардрейл никогда не блокирует; он только аннотирует (`meta.detections`,
`meta.redacted`) или перезаписывает.

### Prompt Injection (`promptInjection.ts`)

Обнаруживает состязательные структуры в пользовательском содержимом и применяет
настроенную политику. Поведение управляется переменными окружения и опциями
конструктора:

| Настройка       | Переменная окружения                            | По умолчанию | Эффект                                  |
| --------------- | ----------------------------------------------- | ------------ | --------------------------------------- |
| Включён         | `INPUT_SANITIZER_ENABLED`                       | `true`       | Когда `false`, гардрейл замыкается накоротко. |
| Режим           | `INJECTION_GUARD_MODE` / `INPUT_SANITIZER_MODE` | `warn`       | `block`, `warn` или `log`.              |
| Порог блокировки | Опция `blockThreshold`                         | `high`       | Минимальная серьёзность для блокировки. |

**Приоритет режима** (`getMode`): `options.mode` вызывающего →
**переопределение флагом функции в БД** `INJECTION_GUARD_MODE` (Dashboard → Settings →
Feature Flags) → env `INJECTION_GUARD_MODE` → env `INPUT_SANITIZER_MODE` →
`warn`. Переопределение из панели, таким образом, побеждает переменные окружения,
так что UI Feature Flags управляет работающим гардом на лету (без перезапуска). Чтение
из БД отказобезопасно: при ошибке гард откатывается к поведению на основе env, а когда
переопределение не задано, поведение идентично разрешению только по env.

Источники обнаружения:

1. `sanitizeRequest()` из `@/shared/utils/inputSanitizer` (общий набор детекторов,
   используемый в других местах конвейера).
2. Встроенные `DEFAULT_GUARD_PATTERNS` (в настоящее время `system_override_inline` и
   `markdown_system_block`, оба серьёзности `high`).
3. Опциональные `customPatterns`, переданные через опции конструктора (строки, regex
   или записи `{ name, pattern, severity }`).

Когда `mode === "block"` **и** хотя бы одно обнаружение достигает порога
серьёзности, `preCall` возвращает `{ block: true, message: "Request rejected:
suspicious content detected" }`. В режимах `warn`/`log` гардрейл логирует, но
пропускает вызов. Общий помощник `evaluatePromptInjection()` также экспортируется
для вызывающих, которым нужно оценивать промпты без прохождения через реестр.

**Граница сканирования (v3.8.20):** детектор инспектирует только **первые 16 КБ**
объединённого текста промпта — `MAX_INJECTION_SCAN_BYTES = 16 * 1024` (16 384 байта) в
`src/shared/utils/inputSanitizer.ts`. И `detectInjection()`, и
`evaluatePromptInjection()` выполняют `slice(0, MAX_INJECTION_SCAN_BYTES)` перед запуском
цикла паттернов. Директивы инъекции располагаются ближе к началу ввода, так что это
ограничивает затраты CPU/GC на регулярные выражения для нагрузок в сотни килобайт без
ослабления обнаружения (см. #3932, #4041).

## Базовый контракт (`base.ts`)

```typescript
class BaseGuardrail {
  enabled: boolean;
  name: string;
  priority: number;

  constructor(name: string, options?: { enabled?: boolean; priority?: number });

  async preCall(payload: unknown, context: GuardrailContext): Promise<GuardrailResult | void>;

  async postCall(response: unknown, context: GuardrailContext): Promise<GuardrailResult | void>;
}

interface GuardrailResult<TValue = unknown> {
  block?: boolean; // true замыкает цепочку накоротко
  message?: string; // отображается при блокировке
  meta?: Record<string, unknown> | null;
  modifiedPayload?: TValue; // возвращается preCall для перезаписи запроса
  modifiedResponse?: TValue; // возвращается postCall для перезаписи ответа
}

interface GuardrailContext {
  apiKeyInfo?: Record<string, unknown> | null;
  disabledGuardrails?: string[] | null;
  endpoint?: string | null;
  headers?: Headers | Record<string, unknown> | null;
  log?: GuardrailLog | Console | null;
  method?: string | null;
  model?: string | null;
  provider?: string | null;
  sourceFormat?: string | null;
  stream?: boolean;
  targetFormat?: string | null;
}
```

Гардрейл сигнализирует «без изменений», возвращая `void`, `{}` или
`{ block: false }`. Возврат `modifiedPayload`/`modifiedResponse` заменяет
значение, протекающее по цепочке, для нижестоящих гардрейлов.

## Реестр (`registry.ts`)

Синглтон `guardrailRegistry` предоставляет:

- `register(guardrail)` — добавляет (или заменяет по нормализованному имени) гардрейл
  и пересортировывает по возрастанию `priority`.
- `clear()` / `list()` — административные помощники.
- `runPreCallHooks(payload, context)` — перебирает активные гардрейлы, прокидывает
  нагрузку через `modifiedPayload` и останавливается на первом `block: true`.
- `runPostCallHooks(response, context)` — тот же поток на стороне ответа.
- `resetGuardrailsForTests({ registerDefaults })` — очищает состояние и опционально
  перерегистрирует значения по умолчанию для чистой изоляции тестов.

Оба раннера возвращают `{ blocked, payload|response, results, guardrail?, message? }`,
где `results` — массив записей `GuardrailExecutionResult`, включающих
погардрейловые поля `blocked`, `skipped`, `modified`, `error` и `meta`,
полезные для трассировки.

### Отключение гардрейлов на запрос

`resolveDisabledGuardrails({ apiKeyInfo, body, headers })` агрегирует
дедуплицированный список имён гардрейлов, которые должны быть пропущены для текущего
запроса. Источники (все необязательные, все объединяются):

- `apiKeyInfo.disabledGuardrails`
- Тело запроса `disabledGuardrails` (верхний уровень)
- Тело запроса `metadata.disabledGuardrails`
- Заголовок `x-omniroute-disabled-guardrails` (или устаревший
  `x-disabled-guardrails`)

Значения могут быть массивами строк или строкой через запятую; имена
нормализуются к нижнему kebab-case (`pii_masker` → `pii-masker`). Результат
передаётся через `context.disabledGuardrails` в реестр, который пропускает
совпадающие гардрейлы (`skipped: true` в `results`).

## Порядок выполнения

Для каждого запроса, протекающего через `src/sse/handlers/chat.ts` и
`open-sse/handlers/chatCore.ts`:

1. `resolveDisabledGuardrails(...)` строит список пропуска из API-ключа, тела
   и заголовков.
2. `guardrailRegistry.runPreCallHooks(body, ctx)` запускает гардрейлы в порядке
   возрастания приоритета:
   - Отключённые гардрейлы записываются как `skipped`.
   - `preCall` каждого гардрейла может перезаписать нагрузку через `modifiedPayload`.
   - Первое `block: true` замыкает цепочку накоротко, и обработчик возвращает
     ответ отклонения гардрейла.
3. (Возможно, перезаписанная) нагрузка протекает в маршрутизацию комбо и отправку
   апстриму.
4. После сборки ответа `guardrailRegistry.runPostCallHooks(...)`
   запускает ту же цепочку на ответе. `block: true` здесь отбрасывает ответ
   апстрима.

Гардрейлы, выбрасывающие исключения, записываются с `error: <message>` и логируются через
`logger.warn`, но цепочка продолжается — fail-open по дизайну.

## Конфигурация

Переменные окружения, читаемые встроенными гардрейлами:

| Переменная                            | Используется                     | Эффект                                                                                           |
| ------------------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------ |
| `INPUT_SANITIZER_ENABLED`             | `prompt-injection`               | Установите `false`, чтобы полностью отключить обнаружение.                                       |
| `INPUT_SANITIZER_MODE`                | `prompt-injection`, `pii-masker` | Общий режим: `warn`, `block`, `log` или `redact`.                                                |
| `INJECTION_GUARD_MODE`                | `prompt-injection`               | Режим гарда инъекций; также флаг функции в БД, который **переопределяет** переменные окружения (БД > ENV). |
| `PII_REDACTION_ENABLED`               | `pii-masker`                     | Когда `true` + режим `redact`, PII запроса вырезаются.                                           |
| `PII_RESPONSE_SANITIZATION` / `_MODE` | `pii-masker` (нижестоящий)       | Управляет поведением маскировщика на стороне ответа.                                             |

Vision Bridge читает конфигурацию времени выполнения из хранилища настроек на базе БД
(`getSettings()`), а не из переменных окружения: `visionBridgeEnabled`, `visionBridgeModel`,
`visionBridgePrompt`, `visionBridgeTimeout`, `visionBridgeMaxImages`. Значения по умолчанию
находятся в `src/shared/constants/visionBridgeDefaults.ts`.

## Пользовательские гардрейлы

```typescript
import { BaseGuardrail, guardrailRegistry } from "@/lib/guardrails";

class BudgetGuardrail extends BaseGuardrail {
  constructor() {
    super("budget", { priority: 50 });
  }

  async preCall(payload, ctx) {
    if (ctx.apiKeyInfo?.budgetExceeded) {
      return { block: true, message: "Daily budget exceeded" };
    }
    return { block: false };
  }
}

guardrailRegistry.register(new BudgetGuardrail());
```

Шаги:

1. Создайте `src/lib/guardrails/myGuardrail.ts`, расширяющий `BaseGuardrail`.
2. Реализуйте `preCall` и/или `postCall`.
3. Либо зарегистрируйтесь при импорте (push из `registerDefaultGuardrails`), либо
   вызовите `guardrailRegistry.register(...)` во время выполнения — реестр заменяет
   любой предыдущий гардрейл с тем же нормализованным именем.
4. Добавьте тесты в `tests/unit/` (существующие примеры:
   `tests/unit/guardrails-registry.test.ts`,
   `tests/unit/prompt-injection-guard.test.ts`,
   `tests/unit/guardrails/visionBridge.test.ts`).

## Тестирование

Используйте `resetGuardrailsForTests()` между тестами, чтобы начинать с известного состояния.
Передайте `{ registerDefaults: false }`, чтобы начать с пустого реестра и
зарегистрировать только тестируемые гардрейлы. Гардрейл Vision Bridge принимает
внедрение зависимостей (`deps.getSettings`, `deps.callVisionModel`), так что тесты
могут прогонять полный поток без доступа к БД или сети.

## См. также

- `src/lib/guardrails/` — реализация
- `src/shared/utils/inputSanitizer.ts` — общий детектор, питающий
  prompt-injection и маскирование PII
- `src/shared/constants/visionBridgeDefaults.ts` — значения по умолчанию Vision Bridge и
  список моделей принудительного моста
- `docs/architecture/RESILIENCE_GUIDE.md` — ортогональный слой (circuit breaker, охлаждения)
- `docs/reference/ENVIRONMENT.md` — полный справочник переменных окружения

## Покрытие маршрутов injection-guard и red-team (Phase 8 · Block D)

Гард инъекций (`createInjectionGuard` / `withInjectionGuard`) покрывает все маршруты,
принимающие пользовательские промпты. Он уважает `INJECTION_GUARD_MODE` (по умолчанию `warn` = только лог;
`block` = возвращает HTTP 400 `SECURITY_001`).

| Тип             | Маршруты                                                                                                                                             | Режим по умолчанию |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| Текстовые (существующие) | `/v1/chat/completions`, `/v1/completions`, `/v1/relay/chat/completions`                                                                  | warn               |
| Генеративные    | `/v1/messages`, `/v1/responses`, `/v1/images/generations`, `/v1/images/edits`, `/v1/videos/generations`, `/v1/music/generations`, `/v1/audio/speech` | warn      |
| Данные          | `/v1/embeddings`, `/v1/rerank`, `/v1/search`, `/v1/moderations`                                                                                      | warn               |

Извлечение текста (`extractMessageContents`) покрывает `messages`/`input`/`prompt`/`query`+`documents`/`instructions`/`system`.

**Red-team (ночной, `nightly-llm-security.yml`):** promptfoo проверяет, что каждый маршрут блокирует
корпус OWASP-LLM в `INJECTION_GUARD_MODE=block`; garak запускает пробы (пропускается без секрета).
`moderations` включён для единообразия — операторы в режиме block могут освободить его через
`resolveDisabledGuardrails`.

Ночной рабочий процесс (`.github/workflows/nightly-llm-security.yml`, cron + ручной
запуск) имеет две задачи:

- **`promptfoo-guard` (блокирующая)** — запускает `promptfoo eval -c promptfooconfig.yaml`
  с `INJECTION_GUARD_MODE=block`. Каждый состязательный случай (например, «ignore all
  previous instructions…», джейлбрейки в стиле DAN) утверждает, что ответ несёт
  `error.code === "SECURITY_001"`, т.е. гард действительно отклонил запрос.
- **`garak` (рекомендательная)** — запускает garak `--probes promptinject,dan,leakreplay`
  против локального инстанса OmniRoute (`http://localhost:20128/v1`). Закрывается
  секретом провайдера (`PROMPTFOO_PROVIDER_KEY`); пропускается плавно и имеет суффикс
  `|| true`, так что отчитывается без провала CI.

Покрытие помощника гарда (`createInjectionGuard` / `withInjectionGuard`)
охватывает каждый маршрут `/v1` с промптами; текст промпта извлекается из
`messages`/`input`/`prompt`/`query`+`documents`/`instructions`/`system` через
`extractMessageContents()` в `src/shared/utils/inputSanitizer.ts`.
