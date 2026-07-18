# Функция требований к контексту в комбо

## Обзор

Функция требований к контексту позволяет конфигурациям комбо фильтровать и сортировать цели по размеру их контекстного окна. Это полезно для сценариев, требующих больших контекстных окон, например:

- Обработка длинных документов (100k+ токенов)
- Анализ больших кодовых баз
- Обширные истории диалогов
- Ревью кода с участием множества файлов

## Настройка

### Схема

Добавьте `contextRequirements` в runtime-конфигурацию вашего комбо:

```json
{
  "contextRequirements": {
    "minContextWindow": 128000,
    "preferLargeContext": true,
    "contextFilterMode": "strict"
  }
}
```

### Поля

#### `minContextWindow` (необязательно)

- **Тип**: `number` (от 0 до 10 000 000)
- **По умолчанию**: `undefined` (без фильтрации)
- **Описание**: Отфильтровывает модели с контекстным окном ниже этого порога

**Примеры**:

- `32000` — отфильтровать модели с контекстом <32K
- `128000` — требовать контекст 128K+ (GPT-4 Turbo, Claude 3)
- `200000` — требовать контекст 200K+ (Claude 3 Opus)
- `1000000` — требовать контекст 1M+ (Gemini 1.5 Pro)

#### `preferLargeContext` (необязательно)

- **Тип**: `boolean`
- **По умолчанию**: `false`
- **Описание**: При `true` оставшиеся цели сортируются по размеру контекста (по убыванию). Модели с большим контекстом пробуются первыми.

#### `contextFilterMode` (необязательно)

- **Тип**: `"strict"` | `"lenient"`
- **По умолчанию**: `"lenient"`
- **Описание**: Как обрабатывать модели с неизвестным лимитом контекстного окна
  - `"strict"`: исключать модели с неизвестным лимитом контекста
  - `"lenient"`: включать модели с неизвестным лимитом контекста

## Поведение

### Конвейер фильтрации

Требования к контексту применяются после `filterTargetsByRequestCompatibility()`:

1. **Фильтрация по совместимости с запросом** — удаляет модели, несовместимые с запросом (инструменты, vision, структурированный вывод)
2. **Фильтрация по требованиям к контексту** — применяет `minContextWindow` и `contextFilterMode`
3. **Сортировка по контексту** — если `preferLargeContext` равен true, сортировка по размеру контекста по убыванию

### Логика режимов фильтрации

Когда задан `minContextWindow`:

**Мягкий режим (lenient)** (по умолчанию):

- ✅ Включает модели с контекстом >= minContextWindow
- ✅ Включает модели с неизвестным лимитом контекста
- ❌ Исключает модели с контекстом < minContextWindow

**Строгий режим (strict)**:

- ✅ Включает модели с контекстом >= minContextWindow
- ❌ Исключает модели с неизвестным лимитом контекста
- ❌ Исключает модели с контекстом < minContextWindow

### Логика сортировки

Когда `preferLargeContext` равен true:

- Модели сортируются по размеру контекстного окна (по убыванию)
- Модели с неизвестным контекстом попадают в конец
- При равенстве используется исходный порядок стратегии

## Сценарии использования

### Пример 1: обработка длинных документов

```json
{
  "name": "Document Analysis",
  "strategy": "fusion",
  "config": {
    "contextRequirements": {
      "minContextWindow": 128000,
      "preferLargeContext": true,
      "contextFilterMode": "strict"
    }
  }
}
```

Эта конфигурация:

- Требует контекстное окно 128K+
- Предпочитает модели с большим контекстом (Gemini 1.5 Pro > Claude 3 Opus > GPT-4 Turbo)
- Исключает модели с неизвестным лимитом контекста

### Пример 2: анализ большой кодовой базы

```json
{
  "name": "Code Review",
  "strategy": "auto",
  "config": {
    "contextRequirements": {
      "minContextWindow": 200000,
      "preferLargeContext": true,
      "contextFilterMode": "lenient"
    }
  }
}
```

Эта конфигурация:

- Требует контекстное окно 200K+
- Предпочитает модели с большим контекстом
- Включает модели с неизвестным лимитом (lenient)

### Пример 3: предпочтение большого контекста без жёстких требований

```json
{
  "name": "Flexible Chat",
  "strategy": "weighted",
  "config": {
    "contextRequirements": {
      "preferLargeContext": true
    }
  }
}
```

Эта конфигурация:

- Без минимального требования (подходят все модели)
- Сортирует по размеру контекста (сначала самый большой)
- Полезно, когда большой контекст предпочтителен, но не обязателен

## Ответ API

Когда требования к контексту фильтруют цели, логгер комбо выводит:

```
[COMBO] Context requirements: filtered 10 → 3 targets (minContextWindow: 128000, mode: strict)
[COMBO] Context requirements: kept models gemini-1.5-pro, claude-3-opus-20240229, gpt-4-turbo
[COMBO] Context requirements: sorted by context size (descending): gemini-1.5-pro(1000000), claude-3-opus-20240229(200000), gpt-4-turbo(128000)
```

## Детали реализации

### Backend-модуль

`open-sse/services/combo/contextRequirements.ts`:

- `applyContextRequirements()` — основная функция фильтрации
- `getTargetContextWindow()` — вспомогательная функция поиска контекста
- Использует `getModelContextLimit()` из `modelCapabilities.ts`

### Точка интеграции

`open-sse/services/combo.ts`, строка 1187:

```typescript
orderedTargets = filterTargetsByRequestCompatibility(orderedTargets, body, log);
orderedTargets = applyContextRequirements(orderedTargets, config.contextRequirements, log);
```

### Определение схемы

`src/shared/validation/schemas/combo.ts`:

```typescript
contextRequirements: z
  .object({
    minContextWindow: z.coerce.number().int().min(0).max(10_000_000).optional(),
    preferLargeContext: z.boolean().optional(),
    contextFilterMode: z.enum(["strict", "lenient"]).optional(),
  })
  .strict()
  .optional(),
```

## Тестирование

### Запуск тестов

```bash
# Модульные тесты (схема + логика)
npm test tests/unit/combo-context-requirements.test.ts

# Интеграционные тесты (сквозные)
npm test tests/unit/combo/context-requirements-integration.test.ts
```

### Покрытие тестами

- Валидация схемы: 6 тестов
- Логика фильтрации: 6 тестов
- Интеграция: 5 тестов
- **Итого**: 17/17 проходят ✅

## Устранение неполадок

### Все цели отфильтрованы

**Проблема**: все цели удалены, комбо возвращает «no compatible models»

**Решения**:

1. Понизьте порог `minContextWindow`
2. Переключитесь в режим `"lenient"`, чтобы включить модели с неизвестным контекстом
3. Уберите `minContextWindow` и используйте только `preferLargeContext`

### Модели с неизвестным контекстом исключены

**Проблема**: пользовательские/новые модели исключены, хотя у них большой контекст

**Решения**:

1. Переключитесь в режим `"lenient"` (по умолчанию)
2. Добавьте лимит контекста модели в `modelCapabilities.ts`
3. Уберите фильтрацию по контексту и полагайтесь на порядок стратегии

### Сортировка не применяется

**Проблема**: `preferLargeContext` не меняет порядок

**Проверьте**:

1. Убедитесь, что в конфигурации `preferLargeContext: true`
2. Проверьте, не имеют ли все цели неизвестный контекст (тогда все сортируются одинаково)
3. Убедитесь, что после фильтрации осталось несколько целей

## Связанные материалы

- [Стратегии маршрутизации Auto-Combo](./routing/AUTO-COMBO.md)
- [Руководство по отказоустойчивости](./architecture/RESILIENCE_GUIDE.md)

## История версий

- **v3.8.47**: первоначальная реализация
  - Добавлена конфигурация `contextRequirements`
  - Создан backend-модуль фильтрации
  - Полное тестовое покрытие (отдельного интерфейса в панели управления пока нет — настройка через JSON комбо)
