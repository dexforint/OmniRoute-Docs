---
title: "Руководство по настройке AgentRouter"
version: 3.8.40
lastUpdated: 2026-06-28
---

# Руководство по настройке AgentRouter

[AgentRouter](https://agentrouter.org) — Anthropic-совместимый ретранслятор, перепродающий
Claude и другие модели, часто по ценам ниже прямого API Anthropic. Он
спроектирован как прямая замена `ANTHROPIC_BASE_URL` для официального клиента Claude Code,
поэтому принимает только трафик, соответствующий сетевому образу Claude Code (специфический
User-Agent, флаги `anthropic-beta`, заголовки Stainless SDK и т.д.).

## Быстрый старт — используйте встроенного провайдера `agentrouter` (рекомендуется)

Для большинства пользователей **никакой специальной настройки не требуется**. OmniRoute поставляется
со встроенным провайдером `agentrouter`, в который уже вшит полный сетевой образ Claude Code (см.
`open-sse/config/providerRegistry.ts` → `agentrouter`). Чтобы использовать его:

1. Откройте **Панель управления → Providers → Add Provider**.
2. Выберите **AgentRouter** из списка.
3. Вставьте ваш API-ключ `sk-...` и сохраните.

Это всё — ни переменных окружения, ни пользовательского типа провайдера. Встроенные модели
включают `claude-opus-4-6`, `claude-haiku-4-5-20251001`, `glm-5.1` и
`deepseek-v3.2`.

Остальная часть этого руководства описывает **продвинутый путь**: использование
типа провайдера `anthropic-compatible-cc-*`. Применяйте его, когда нужен больший контроль
над сетевым образом — например, при подключении к другим ретрансляторам в стиле
AgentRouter, которых пока нет в реестре встроенных провайдеров, или при переопределении
базового URL, пути чата или набора заголовков.

---

## Продвинутый вариант: подключение через тип провайдера Claude Code compatible

OmniRoute также поддерживает AgentRouter (и похожие ретрансляторы) через тип провайдера
**Claude Code compatible** (`anthropic-compatible-cc-*`), который говорит на Anthropic
Messages API с корректным сетевым образом. Обычный провайдер `openai-compatible-chat`,
указывающий на `https://agentrouter.org`, работать **не будет** — апстрим-WAF отклоняет
запросы, не похожие на Claude Code.

---

## Предварительные требования

- Аккаунт AgentRouter и API-ключ. Новые пользователи получают бесплатные кредиты по партнёрской
  ссылке в [README](../README.md) проекта.
- Запущенный OmniRoute с включённым флагом `ENABLE_CC_COMPATIBLE_PROVIDER`
  (см. ниже).

## 1. Включите тип провайдера CC-compatible

Тип провайдера Claude Code compatible закрыт флагом функции, потому что он
отправляет трафик, очень похожий на официальный клиент Claude Code. Включите его,
установив переменную окружения перед запуском OmniRoute:

```bash
ENABLE_CC_COMPATIBLE_PROVIDER=true
```

Пример с Docker:

```bash
docker run -d --name omniroute \
  --restart unless-stopped \
  -p 20128:20128 \
  -v omniroute-data:/app/data \
  -e ENABLE_CC_COMPATIBLE_PROVIDER=true \
  diegosouzapw/omniroute:latest
```

После перезапуска панель управления покажет опцию **Add Claude Code Compatible**
в дополнение к существующим сценариям OpenAI-compatible и Anthropic-compatible.

## 2. Создайте провайдера в панели управления

1. Откройте **Панель управления → Providers → Add Provider**.
2. Выберите **Add Claude Code Compatible** (видно только при установленном флаге выше).
3. Заполните поля:

| Поле      | Значение                                                       |
| --------- | -------------------------------------------------------------- |
| Name      | `AgentRouter` (или любая метка)                                 |
| Prefix    | `agentrouter` (дружелюбный псевдоним для логов и панели)        |
| Base URL  | `https://agentrouter.org`                                      |
| Chat path | `/v1/messages?beta=true` (по умолчанию — оставьте как есть)     |

> Канонический идентификатор модели по-прежнему использует полный ID узла провайдера
> (`anthropic-compatible-cc-{uuid}/{model}`). **Prefix** — это просто отображаемый
> псевдоним, разрешаемый `src/lib/usage/callLogs.ts` для более читаемых логов.

4. (Необязательно) Вставьте ваш API-ключ в поле **Validate** и нажмите **Check**, чтобы
   проверить связность перед сохранением.
5. Нажмите **Add**.

После создания откройте провайдера и добавьте **Connection** с вашим API-ключом
AgentRouter (`sk-...`). Статус `test_status` подключения должен стать `active`.

## 3. Используйте через комбо или напрямую

Ссылайтесь на модель, используя префикс вашего провайдера как пространство имён:

```bash
curl -X POST http://localhost:20128/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "agentrouter/claude-opus-4-6",
    "messages": [{"role": "user", "content": "hello"}],
    "max_tokens": 100
  }'
```

Канонический ID модели `anthropic-compatible-cc-{uuid}/claude-opus-4-6` тоже работает
и именно он отображается в базе данных и конфигурации комбо.

Или добавьте его в комбо для маршрутизации, переключения при отказе и управления квотами,
как любого другого провайдера.

---

## Детали сетевого образа

Для справки: мост cc-compatible отправляет следующее в каждом апстрим-запросе
(см. `open-sse/services/claudeCodeCompatible.ts`):

| Заголовок                                   | Значение                                                                                                |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `Authorization`                             | `Bearer <api-key>`                                                                                      |
| `User-Agent`                                | `claude-cli/2.1.207 (external, sdk-cli)`                                                                |
| `anthropic-version`                         | `2023-06-01`                                                                                            |
| `anthropic-beta`                            | `claude-code-20250219,interleaved-thinking-2025-05-14,effort-2025-11-24`                                |
| Переключатель redact-thinking beta на подключение | Добавляет `redact-thinking-2026-02-12` для апстримов, которым явно нужны потоки с редактированными рассуждениями |
| Переключатель summarized thinking на подключение | Добавляет `display: "summarized"` к запросам CC Compatible thinking, у которых не задан режим отображения |
| `anthropic-dangerous-direct-browser-access` | `true`                                                                                                  |
| `x-app`                                     | `cli`                                                                                                   |
| `X-Stainless-*`                             | Разные заголовки Stainless SDK (язык, версия пакета, ОС, архитектура и т.д.)                            |

Именно это позволяет запросам проходить апстрим-WAF / белый список клиентов.

---

## Устранение неполадок

**`{"error":{"message":"unauthorized client detected, ..."}}`** — ваш запрос не
совпал с сетевым образом Claude Code. Такое случается, когда провайдер настроен
как `openai-compatible-chat` вместо `anthropic-compatible-cc`, или когда флаг
`ENABLE_CC_COMPATIBLE_PROVIDER=true` не был установлен при запуске.

**`{"error":{"message":"无效的令牌","type":"new_api_error"}}` (HTTP 401)** —
«Недействительный токен». Сетевой образ корректен, но API-ключ отклонён. Сгенерируйте
новый ключ в панели AgentRouter и обновите подключение.

**`{"error":{"code":"content-blocked","type":"agent_router_api_error"}}`
(HTTP 400)** — хук модерации AgentRouter отклонил содержимое запроса или
тариф ключа не разрешает запрошенную модель. Попробуйте другой промпт или модель;
обратитесь в поддержку AgentRouter, если безобидный промпт блокируется постоянно.

**`[400]: content-blocked` только на конкретных моделях** — большинство тарифов
AgentRouter разрешают только подмножество моделей (например, `claude-opus-4-6`). Другие
ID моделей возвращают `unauthorized_client_error`, даже если ключ валиден. Проверьте,
какие модели покрывает ваш тариф, в панели AgentRouter.

**`Invalid JSON response from provider (reset after Ns)` в логах omniroute** —
апстрим вернул не-JSON тело (обычно HTML-страницу ошибки от WAF).
Обычно это означает, что запрос вообще не дошёл до бэкенда AgentRouter — проверьте,
что ID провайдера начинается с `anthropic-compatible-cc-` (обратите внимание на
завершающий дефис — см. `CLAUDE_CODE_COMPATIBLE_PREFIX` в
`open-sse/services/claudeCodeCompatible.ts`) и что флаг функции включён.

---

## См. также

- [`docs/providers/CLAUDE_WEB.md`](./CLAUDE_WEB.md) — заметки по интеграции провайдера Claude Web
- [`docs/reference/FREE_TIERS.md`](../reference/FREE_TIERS.md) — каталог провайдеров
  с бесплатными тарифами
- [`open-sse/services/claudeCodeCompatible.ts`](../../open-sse/services/claudeCodeCompatible.ts)
  — реализация сетевого образа
